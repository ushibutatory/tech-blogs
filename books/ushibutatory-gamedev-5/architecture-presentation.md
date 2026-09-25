---
title: "アーキテクチャ詳細 - Presentation層"
---

## Presentation層

### Component

MonoBehaviour を継承したコンポーネントです。
ひとつのゲームオブジェクトに対して、複数のコンポーネントをアタッチすることがよくあります。

よくあるパターンが「本体コンポーネント」＋「機能コンポーネント」という構成です。

- 命名ルール
  - 本体コンポーネント : `-Component`
  - 機能コンポーネント : `-able`、またはUnityコンポーネント名（`Collider`等）

- 例）
  - Cardオブジェクト
    - CardComponent（カードオブジェクトを制御する本体となるコンポーネント）
    - CardMovable（カードを移動させる機能コンポーネント）
    - CardHighlightable（カードをハイライトさせる機能コンポーネント）
    - CardTappable（カードをタップ可能にする機能コンポーネント）
    - CardCollider（カードの当たり判定コンポーネント）

::: details 例

本体コンポーネント

```csharp
namespace WordPuzzle.Presentation.Scenes.InGame.Components
{
    [RequireComponent(typeof(CardCollider))]
    [RequireComponent(typeof(CardMovable))]
    [RequireComponent(typeof(CardHighlightable))]
    public abstract class CardComponent : MonoBehaviour
    {
        public ICard? Card { get; private set; } = default!;

        // 機能コンポーネント
        public CardCollider? Collider { get; private set; }
        public CardMovable? Movable { get; private set; }
        public CardHighlightable? Highlightable { get; private set; }

        public void Initialize(ICard card)
        {
            Card = card;
        }

        private void Awake()
        {
            Collider = GetComponent<CardCollider>();
            Movable = GetComponent<CardMovable>();
            Highlightable = GetComponent<CardHighlightable>();
        }
    }
}
```

機能コンポーネント。
どこにどう移動させるかの詳細はできるだけ中で記述せず、引数や設定で制御できるようにしておくのが私好みです。

```csharp
namespace WordPuzzle.Presentation.Scenes.InGame.Components
{
    /// <summary>
    /// カードを移動可能とする
    /// </summary>
    public class CardMovable : MonoBehaviour
    {
        private Tween? _tween;

        private void OnDestroy()
        {
            _tween?.Kill();
        }

        public void MoveTo(WorldPosition position)
        {
            _tween?.Kill();
            transform.position = new Vector3(position.X, position.Y, transform.position.z);
        }

        public async UniTask MoveToAsync(WorldPosition position, Seconds duration, Ease ease)
        {
            _tween?.Kill();

            _tween = transform
                .DOMove(position, duration)
                .SetEase(ease);

            await _tween.ToUniTask(cancellationToken: destroyCancellationToken);
        }
    }
}
```

他のクラスから操作する場合は以下のようになります。

```csharp
if (cardComponent.Movable != null)
{
    // 機能コンポーネントを使って移動させる
    await cardComponent.Movable.MoveToAsync(
        position: ...,
        duration: ...,
        ease: ...);
}
```

:::

### UI

UIの表示を行うクラスです。MonoBehaviour は継承しません。

今回は UI Toolkit を使用しました。

[別ページ](./presentation-ui)にまとめました。

### PresentationEvent

コンポーネント間の通信用イベントです。
コンポーネントが他のコンポーネントに出す指示です。

すべて `-Request` というサフィックスをつけます。

- 例）
  - 効果音再生コンポーネントに対して、効果音再生をリクエストする。
  - エフェクト再生コンポーネントに対して、エフェクト再生をリクエストする。

自身のコンポーネントが保持しないコンポーネントに対して、何らかの操作を行いたい場合に使用します。なので「手札コンポーネントが手札内のカードコンポーネントを操作する」という場合には当イベントは使用しません（直接操作します）。

::: details 例

```csharp
using WordPuzzle.Presentation.Abstractions;

namespace WordPuzzle.Presentation.Audio.Requests
{
    /// <summary>
    /// プレゼンテーションイベント：効果音を再生してほしい
    /// </summary>
    public readonly record struct PlaySoundRequest(SoundId SoundId) : IPresentationEvent;
}

namespace WordPuzzle.Presentation.Effect.Requests
{
    /// <summary>
    /// プレゼンテーションイベント：エフェクトを再生してほしい
    /// </summary>
    public readonly record struct PlayEffectRequest(EffectId EffectId, WorldPosition Position) : IPresentationEvent
    {
        public Transform? FollowTarget { get; init; } = null!;
    }
}
```

:::

### Listener

アプリケーション層のSessionEventを購読するコンポーネントです。

セッション状態の変更通知をコンポーネントが直接購読してもいいのですが、あれもこれも制御しなければならない、それらの処理順序保証をしなければならない、という場合に使用しました。

::: details コード

```csharp
namespace WordPuzzle.Presentation.Scenes.InGame.Listeners
{
    public class HandUpdatedListener : EventListener<HandUpdatedListener>
    {
        [Inject] private readonly ApplicationDependencies _application = default!;
        public class ApplicationDependencies
        {
            [Inject] public readonly ISubscriber<HandUpdated> HandUpdated = default!;
        }

        [Inject] private readonly PresentationDependencies _presentation = default!;
        public class PresentationDependencies
        {
            [Inject] public readonly PlayerBoardComponent PlayerBoard = default!;
            [Inject] public readonly LayoutConductor LayoutConductor = default!;
            [Inject] public readonly CardPool CardPool = default!;

            [Inject] public readonly IInGameCardAnimationSettings AnimationSettings = default!;
        }

        protected override void _SetupSubscribes()
        {
            base._SetupSubscribes();

            _application.HandUpdated.Subscribe(async e =>
            {
                // 手札を再描画
                _presentation.PlayerBoard.Rebuild(e.Hand);

                switch (e.Reason)
                {
                    case HandUpdated.ReasonType.CardAdded:
                    case HandUpdated.ReasonType.CardRemoved:
                        // カードを移動
                        _MoveCardsToHandAsync(e.Hand).Forget();
                        break;

                    case HandUpdated.ReasonType.CardTrashed:
                        // カードを破棄
                        foreach (var cardId in e.CardIds)
                            _presentation.CardPool.Despawn(cardId);
                        break;
                }

                // 全体レイアウトの調整
                await _presentation.LayoutConductor.ConductAsync();
            }).AddTo(_disposables);
        }

        ...
    }
}
```

:::

### PresentationSettingsインタフェース

プレゼンテーション層の挙動を指定する設定群です。

DomainSettingsインタフェースと考え方は同じです。

実装はInfrastructure層でScriptableObjectとして実装します。

::: details 例

```csharp
namespace WordPuzzle.Presentation.Scenes.InGame.Settings
{
    public interface IInGameCardAnimationSettings
    {
        /// <summary>
        /// カード移動（フィールド配置）アニメーションの時間（秒）
        /// </summary>
        Seconds MoveToFieldDuration { get; }

        /// <summary>
        /// カード移動（フィールド配置）アニメーションのEase
        /// </summary>
        Ease MoveToFieldEase { get; }

        /// <summary>
        /// カード移動（手札配置）アニメーションの時間（秒）
        /// </summary>
        Seconds MoveToHandDuration { get; }

        /// <summary>
        /// カード移動（手札配置）アニメーションのEase
        /// </summary>
        Ease MoveToHandEase { get; }

        /// <summary>
        /// カード移動（破棄）アニメーションの時間（秒）
        /// </summary>
        Seconds MoveToDiscardDuration { get; }

        /// <summary>
        /// カード移動（破棄）アニメーションのEase
        /// </summary>
        Ease MoveToDiscardEase { get; }
    }
}
```

:::

## 所感

今までの開発では InputActions による入力をPresentation層の中に入れていましたが、今回は別ライブラリとしました。

同じように、Localizationも別ライブラリにしたほうが良かったかもしれません（UIテキストだけでなく、アプリケーション層で生成するエラーメッセージをローカライズしたいことがあったので、プレゼンテーション層で扱うと少し面倒くさい）。ただ、「アプリケーション層の（システム管理者向けの）エラーメッセージと、プレゼンテーション層の（ユーザ向け）エラーメッセージは別ものとして扱うべきである」という思いもあるので、どういう設計にするのがいいかはもう少し考えたいところです。

実は同じ理由で、Audio/Soundも別ライブラリにしたほうがいいんじゃないかぁと思ったりしています。次開発では分割して試してみたいと思います。
