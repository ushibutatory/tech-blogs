---
title: "アーキテクチャ詳細 - Application層"
---

## Application層

- すること
  - ドメインロジックの組み合わせにより、アプリケーションの仕様（ユースケース）を定義する。
- しないこと
  - シーンやUI、演出など、Unity領域の制御をしてはならない。

### UseCase

ユースケースを定義します。基本は「_主語（S）が 目的語（O）を 動詞（V）する_」です。

- 例）
  - プレイヤーがカードを選択する
  - プレイヤーが手札を提出する
  - システムが山札をシャッフルする
  - システムが山札から新規カードをディールする

「タイトル画面に戻る」はユースケース（の命名）として適切ではありません。そういう場合は「プレイヤーが現在のゲームを終了する」というユースケースとして定義します（そのユースケース完了を待って、プレゼンテーション層でシーン遷移をします）。

ユースケースでは、ドメインロジックの呼び出しやセッション（状態）の操作を組み合わせて一連のトランザクションを表現します。
ユースケース自体は状態を持たない単発の関数です。必要に応じてセッションやリポジトリを参照・更新します。

アプリケーション層のテストでスタブ差し替えするために、インタフェースと実装を分けるようにしています。

::: details 例

```csharp
namespace WordPuzzle.Application.InGame.UseCases
{
    /// <summary>
    /// ユースケース：カードを手札に追加する
    /// </summary>
    public interface ITryAddCardToHand : IUseCase
    {
        enum ErrorCode
        {
            Unknown,
            InvalidCardPlacement,
        }

        record Error(ErrorCode ErrorCode) : ApplicationError
        {
            public required ICard ErrorCard { get; init; }
        }

        record Request
        {
            public required ICard Card { get; init; }
            public required FieldSlotId FieldSlotId { get; init; }
        }

        abstract record Response
        {
            public sealed record Success : Response
            {
                public required IHand Hand { get; init; }
            }

            public sealed record Failure : Response
            {
                public required Request Request { get; init; }
                public required IReadOnlyList<Error> Errors { get; init; }
            }
        }

        Response Execute(Request request);
    }
}

namespace WordPuzzle.Application.InGame.UseCases.Implements
{
    public class TryAddCardToHand : UseCase<TryAddCardToHand>, ITryAddCardToHand
    {
        [Inject] private readonly ApplicationDependencies _application = default!;
        public class ApplicationDependencies
        {
            [Inject] public readonly IInGameSessionManager InGameSession = default!;
        }

        [Inject] private readonly DomainDependencies _domain = default!;
        public class DomainDependencies
        {
            [Inject] public readonly IConnectionStructureValidator ConnectionStructureValidator = default!;
            [Inject] public readonly ISentenceStructureValidator SentenceStructureValidator = default!;
        }

        public ITryAddCardToHand.Response Execute(ITryAddCardToHand.Request request)
        {
            try
            {
                _logger?.LogDebug($"{nameof(Execute)} ...");

                _logger?.LogDebug("現在のセッション状態を取得します ...");
                var state = _application.InGameSession.Current;
                if (!state.IsInitialized)
                    throw new InvalidSessionException(InvalidSessionException.ReasonType.SessionNotInitialized);

                var card = request.Card;

                _logger?.LogDebug($"カードを手札に追加できるか検証します ... カードID: {card.Id}");
                var result = _domain.ConnectionStructureValidator.Validate(state.Hand, card);

                switch (result)
                {
                    case IConnectionStructureValidator.Result.Success:
                        _logger?.LogDebug("カードを手札に追加できます。");

                        _logger?.LogDebug("カードを手札に追加します ...");
                        state.HandState.AddCardToHand(card);

                        _logger?.LogDebug("手札が揃ったかどうかをチェックします ...");
                        state.HandState.SetReadyToSubmit(_domain.SentenceStructureValidator.IsSuccess(state.Hand));

                        _logger?.LogDebug("カードをフィールドから取り除きます ...");
                        state.FieldState.RemoveCard(request.FieldSlotId);

                        _logger?.LogDebug("カードを手札に追加しました。");
                        return new ITryAddCardToHand.Response.Success
                        {
                            Hand = state.Hand
                        };

                    case IConnectionStructureValidator.Result.Failure failure:
                        _logger?.LogDebug($"手札を提出できません。エラー件数: {failure.Errors.Count}");
                        foreach (var error in failure.Errors)
                            _logger?.LogDebug($"  - {error}");

                        return new ITryAddCardToHand.Response.Failure
                        {
                            Request = request,
                            Errors = failure.Errors.Select(e => _ConvertError(e, card)).ToList()
                        };

                    default:
                        throw new InvalidOperationException("Unexpected result type");
                }
            }
            catch (DomainException ex)
            {
                throw new ApplicationException("カードを手札に追加できませんでした。", ex);
            }
        }
    }
}
```

検証結果を`switch`で分岐させていますが、Errorの場合にearly returnしてもいいと思います。どっちがいいんだろう。

:::

### Service

ユースケースのような固有の文脈を持たない、汎用的な機能を提供するクラスです。

- 例）
  - プレイヤーのプロファイルデータを参照・更新する。

プレゼンテーション層からは利用されません。
アプリケーション層の複数のユースケースから呼び出される、サブユースケース、みたいなイメージです。

あまり作りすぎるとユースケースとの境目で曖昧になり管理が難しくなりそうだと思ったので、原則としてはまず固有のユースケースを作成し、その中で「前処理・本処理・後処理」のような一連の流れが何度も登場した場合に _Service_ として切り出すようにしました。

::: details 例

```csharp
namespace WordPuzzle.Application.Services
{
    /// <summary>
    /// アプリケーションサービス：プレイヤーの統計情報管理
    /// </summary>
    /// <remarks>
    /// <br/>- <see cref="IPlayerStats"/> ではなく <see cref="PlayerStats"/> クラスを扱います。
    /// <br/>- また、各操作はアプリケーション層内からのみ実行されるため、 internal とします。
    /// </remarks>
    public interface IPlayerStatsService
    {
        internal UniTask<DomainPlayerStats> LoadAsync(PlayerProfileId playerProfileId);
        internal UniTask SaveAsync(PlayerProfileId playerProfileId, DomainPlayerStats stats);
    }
}

namespace WordPuzzle.Application.Services.Implements
{
    public class PlayerStatsService : IPlayerStatsService
    {
        [Inject] private readonly ILogger<PlayerStatsService> _logger = default!;
        [Inject] private readonly IPlayerStatsRepository _repository = default!;

        public async UniTask<DomainPlayerStats> LoadAsync(PlayerProfileId playerProfileId)
        {
            _logger?.LogDebug($"{nameof(LoadAsync)} ...");

            var stats = await _repository.LoadAsync(playerProfileId).AsUniTask();

            _logger?.LogDebug($"{nameof(LoadAsync)} ... Done.");

            return stats;
        }

        public async UniTask SaveAsync(PlayerProfileId playerProfileId, DomainPlayerStats playerStats)
        {
            _logger?.LogDebug($"{nameof(SaveAsync)} ...");

            await _repository.SaveAsync(playerProfileId, playerStats).AsUniTask();

            _logger?.LogDebug($"{nameof(SaveAsync)} ... Done.");
        }
    }
}
```

:::

### Session

特定のライフサイクルと連動し、状態（具体的には _Entity_ のインスタンス等）を保持します。

この状態と状態管理の仕組みを「_Session_（セッション）」と呼ぶことにしています。

「タイトルシーンに入ってから抜けるまで」、「インゲームが始まってから終わるまで」など、まとまった期間の状態を管理します。ただし、必ずしもシーンと1対1で対応するわけではありません。「サブメニューを開いてから閉じるまで」なども独立したセッションにできます（します、ではない）。

ユースケースから更新され、更新内容を `R3` や _SessionEvent_（後述） でプレゼンテーション層に通知します。

::: details 例

```csharp
namespace WordPuzzle.Application.InGame
{
    public interface IInGameSessionManager : ISessionManager
    {
        IInGameSessionState Current { get; }

        internal void Initialize(InGameSessionContext context);
    }
}

namespace WordPuzzle.Application.InGame
{
    public interface IInGameSessionState : ISessionState
    {
        IField Field { get; }
        IHand Hand { get; }

        ReadOnlyReactiveProperty<IScoreResult> CurrentScore { get; }

        ReadOnlyReactiveProperty<bool> EnableReport { get; }

        internal StartupOptionState StartupOption { get; }
        internal FieldState FieldState { get; }
        internal HandState HandState { get; }
        internal ScoreState ScoreState { get; }
    }
}
```

:::

長くなったので別ページとしました。
→[セッション](./application-session)

### SessionEvent

アプリケーション層から通知するイベントです。「セッション状態が変更された」というイベントを発行します。
購読はプレゼンテーション層で行います。

::: details 例

```csharp
namespace WordPuzzle.Application.InGame.Events
{
    /// <summary>
    /// セッションイベント：手札が更新された
    /// </summary>
    public readonly record struct HandUpdated : ISessionEvent
    {
        public enum ReasonType
        {
            Initialized,
            CardAdded,
            CardRemoved,
            CardTrashed,
        }

        public required ReasonType Reason { get; init; }
        public required IHand Hand { get; init; }
        public required IReadOnlyCollection<CardId> CardIds { get; init; }
    }
}
```

:::

### ApplicationSettingsインタフェース

アプリケーション層の挙動を指定する設定群です。

DomainSettingsインタフェースと考え方は同じです。

## 所感

ユースケースを定義する際の命名が「IVerbObjectインタフェース」「VerbObjectクラス」というような動詞始まりなのが、未だに少し慣れません。
