---
title: "「タワーディフェンス」(6) シーン遷移のあれこれ"
---

実現したいことは以下の2点でした。

- パラメータを受け渡しする
- 「（一つ前のシーンに）戻る」ができる

きっとほぼ全てのゲーム開発者が各々作っているであろう車輪の再開発です。

## パラメータ付きシーン遷移

![シーン遷移](https://storage.googleapis.com/zenn-user-upload/662acda4ae3b-20260304.png)

### コンポーネント

- `SceneNavigator`
  - `MonoBehaviour` を継承します。
  - `DontDestroyOnLoad` に配置します。
  - シーン遷移を実行します。
  - シーン遷移後、`SceneLoaded`イベントを発行します。
  - 「SceneManager」や「SceneController」、「SceneSystem」といった名前にすると責務過多を起こしそうだったので、Navigator としました。
- `SceneContext`
  - `MonoBehaviour` を継承します。
    - シーンごとに定義します。（`TitleSceneContext`など）
  - ヒエラルキーに配置します。
  - `SceneLoaded`イベントを購読します。
  - `SceneContextReady`イベントを発行します。
  - 自身のシーンの現在の状態を表します。
    - 特に、パラメータを保持し、プロパティとして他コンポーネントに公開します。
    - イミュータブルとします。
      - `SceneContext`の値を更新したくなった時は、以下のことを検討します。
        - 「この値はインフラ層に永続化すべきドメインモデルではないのか？」
        - 「状態を持つゲームオブジェクトとして配置すべきコンポーネントではないのか？」

### Pure C#

- `ISceneParameter`
  - パラメータインタフェースです。
- `SceneParameter`
  - シーンごとに定義します。（`TitleSceneParameter`など）
  - シーン間の受け渡しパラメータを定義します。
  - `LoadSceneRequest` の発行時に生成して一緒に送ります。

### イベント

コンポーネント間で疎に通信するためのイベントです（`MessagePipe` を使用しました）。

- `LoadSceneRequest`
  - 各コンポーネントから`SceneNavigator`に対して発行されます。
  - シーン読み込みを要求します。
- `SceneLoaded`
  - `SceneNavigator`から各`SceneContext`に対して発行されます。
  - シーン読み込みが完了したことを通知します。
- `SceneContextReady`
  - シーン内の`SceneContext`から各コンポーネントに対して発行されます。
  - 各コンポーネントが購読し、必要な初期化処理を行います。
    - 順番保証が不要な独立したコンポーネントはこのイベントを待たず`Awake()`や`Start()`などで初期化を行います。

## 遷移したシーンの履歴管理

「直前のシーンに戻る」のような挙動が必要な場合、ユーザ操作によって遷移先のシーンが異なります。
そこで、シーンの遷移順を管理するためにスタックによる管理と、スタック操作種別を実装しました。

どれぐらい必要になるかは別として「そういう仕組みをやってみよう」でやってみました。

```csharp
namespace Elemecho.Presentation.Constants
{
    /// <summary>
    /// シーン遷移種別
    /// </summary>
    public enum SceneTransitionType
    {
        /// <summary>
        /// スタックをクリアして遷移します。
        /// </summary>
        Load,

        /// <summary>
        /// 現在のシーンをスタックに保存してから遷移します。
        /// </summary>
        Push,

        /// <summary>
        /// スタックに保存されている直前のシーンに遷移します。
        /// 前のシーンがない場合は何もしません。
        /// </summary>
        Pop,

        /// <summary>
        /// 現在のシーンを置き換えます。
        /// スタック状態は変更しません（置換前のシーンは保存しません）。
        /// </summary>
        Replace,

        /// <summary>
        /// スタックをクリアして遷移します。
        /// （Loadと挙動は同じです。実装意図の区別のために使います。）
        /// </summary>
        Reset
    }
}
```

`LoadSceneRequest`発行時に、どのようにシーンを読み込むかを指定します。

```csharp
namespace Elemecho.Presentation.Events.System
{
    public record LoadSceneRequest : PresentationEvent
    {
        public SceneId SceneId { get; init; }
        public ISceneParameter Parameter { get; init; }
        public SceneTransitionType SceneTransitionType { get; init; } = SceneTransitionType.Load; // デフォルト動作はLoadとする
        ...
    }
}
```

それぞれの遷移イメージは以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/f4532140a72d-20260304.png)

なお、 `LoadSceneMode` は常に `Single` としています。
`Additive` が必要な場面では代わりに Prefab で代替しています。

`SceneNavigator`がスタックする項目は、「シーン ID」「シーン遷移時のパラメータ」のみです。
以下のような構造体（`record struct`）としました。

```csharp
namespace Elemecho.Presentation.SceneManagement
{
    public class SceneNavigator : SystemComponent<SceneLoader>
    {
        ...
        private SceneNavigation _currentScene = default!;
        private Stack<SceneNavigation> _sceneStack = new();

        private readonly record struct SceneNavigation(SceneId SceneId, ISceneParameter Parameter);
        ...
    }
}
```

`SceneNavigator`は「どのような順番、どのようなパラメータでシーンが呼ばれたか」のみを管理します（`Navi`と名付けているのはそういう理由です）。
シーンがどのような状態であるかは `SceneNavigator` ではなく、ドメインモデルとインフラ層（永続化処理）で管理します。発射された弾など、永続化しなかったものにシーン遷移した際に破棄されます。

つまり、シーン遷移時に記録しておきたいものはドメインモデルとして定義し、シーン遷移するまでにインフラ層に永続化しておく必要があります。

そのため、`Pop`により直前シーンに戻った場合、シーン起動時のパラメータは同じですが、実際の状態は起動時にインフラ層から読み込まれるため、前回表示時の状態を復元するのではなく最新の状態が表示されることになります。発射された弾など永続化しなかったオブジェクトも復元されません。

このあたりの管理方針は個人の好みで別れるところかと思います。

## ゲーム起動時の処理の流れ

タイトルシーンとは別に、ゲーム起動用のシーン（`BootstrapScene`）を用意してみました。

下記コンポーネントを `BootstrapScene` のヒエラルキーに配置しておきます。

```csharp
namespace Elemecho.Presentation.SceneManagement.Bootstrap
{
    public class BootstrapSceneContext : SceneContext<BootstrapSceneContext>
    {
        [Inject] private readonly IPublisher<LoadSceneRequest> _loadSceneRequest = default!;

        protected override void Start()
        {
            base.Start();

            UniTask.Void(async () =>
            {
                // 待機
                await UniTask.Delay((int)(_delaySeconds * 1000));

                // 最初のシーンをロード
                _presentation.LoadSceneRequest.Publish(LoadSceneRequest.Load(SceneId.Title));
            });
        }
    }
}
```

Bootstrap シーンはゲーム起動時に 1 度だけ読み込まれ、上記処理によって自動でタイトルシーンに遷移した後は戻ってきません。

開発中、シーンごとに動作確認する際に、必ず事前実行しておかなければならない処理をこの Bootstrapシーンに記述するようにしました。
こうすることで、毎回「タイトル→...→テストしたいシーン」といった遷移を挟まずに動作確認できるようになりました。

## 所感

「Now Loading...」を作らなかったので、シーン遷移時にゲームが停止する感じになります。
次は作りたい。
