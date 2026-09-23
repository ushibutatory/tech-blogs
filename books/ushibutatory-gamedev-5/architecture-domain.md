---
title: "アーキテクチャ詳細 - Domain層"
---

## Domain層

- すること
  - ビジネスロジックを定義する。
- しないこと
  - 外部リソースへのアクセスをしてはならない。
  - Core層を除く外部ライブラリに依存してはならない。

### Entity

「構成」、「状態」、「状態の操作」を持つモデル、としました。

- 例）山札、手札、カードの実体、など

1. 構成
   - 更新不可❌️のフィールドです。
   - 例）ID、カード種別など
1. 状態
   - 更新可能⭕️なフィールドです。
   - 例）現在のスコア、山札や手札の状態など
   - ただし外部から直接更新することはできません（参照は可）。
1. 状態の操作
   - 状態を更新するためのメソッドです。

Entityのインスタンス（生成、保持、破棄）は、Application層で管理します。

::: details 例

```csharp
namespace WordPuzzle.Domain.InGameComponents
{
    /// <summary>
    /// 山札
    /// </summary>
    public class CardDeck<TCardDefinition, TCard>
        where TCardDefinition : ICardDefinition
        where TCard : ICard
    {
        private readonly IReadOnlyList<TCardDefinition> _definitions;
        private readonly IRandom _random;

        private readonly Func<TCardDefinition, TCard> _factory;
        private Queue<TCardDefinition> _remainDefinitions;

        public CardDeck(IEnumerable<TCardDefinition> definitions, IRandom random, Func<TCardDefinition, TCard> factory)
        {
            _definitions = definitions.ToList();
            _random = random;
            _factory = factory;
            _remainDefinitions = _Shuffle(definitions, random);
        }

        /// <summary>
        /// 山札の総カード枚数を取得します。
        /// </summary>
        public int TotalCardCount => _definitions.Count;

        /// <summary>
        /// 山札に残っているカードの枚数を取得します。
        /// </summary>
        public int RemainCount => _remainDefinitions.Count;

        /// <summary>
        /// 山札からカードを1枚引きます。
        /// </summary>
        /// <remarks>
        /// - 山札が空の場合は、再シャッフルして山札を再構築してからカードを引きます。
        /// </remarks>
        public TCard DrawOne()
        {
            if (_remainDefinitions.Count == 0)
            {
                Reset();
            }

            return _factory(_remainDefinitions.Dequeue());
        }

        /// <summary>
        /// 山札をリセットします。
        /// </summary>
        public void Reset()
        {
            _remainDefinitions = _Shuffle(_definitions, _random);
        }

        /// <summary>
        /// カードをシャッフルして並べます。
        /// </summary>
        private Queue<TCardDefinition> _Shuffle(IEnumerable<TCardDefinition> definitions, IRandom random)
            => new(definitions.OrderBy(_ => random.Value()));
    }
}
```

:::

### DomainFunction

単発の処理です。入力に対して結果を返します。

- 例）点数計算、文章成立の判定、など

挙動に関する設定なども自身では保持しません。常にパラメータで挙動を指示します。
また、同じ入力であれば同じ出力が返ることを想定しています。
乱数が必要な場合、内部で乱数を管理するのではなく、乱数ジェネレータを入力で渡します。

アプリケーション層のテストでスタブ差し替えするために、インタフェースと実装を分けるようにしています。

::: details 例1

```csharp
namespace WordPuzzle.Domain.InGameMechanics.Grammar.SentenceSemantic
{
    /// <summary>
    /// 文章の意味検証インタフェース
    /// </summary>
    public interface ISentenceSemanticValidator
    {
        abstract record Error : DomainError;
        abstract record Result
        {
            public sealed record Success : Result;
            public sealed record Failure : Result
            {
                public required IReadOnlyList<Error> Errors { get; init; }
            }
        }

        /// <summary>
        /// 手札を意味検証します。
        /// </summary>
        Result Validate(IHand hand);
    }
}

namespace WordPuzzle.Domain.InGameMechanics.Grammar.SentenceSemantic
{
    /// <summary>
    /// 文章の意味検証
    /// </summary>
    public class SentenceSemanticValidator : ISentenceSemanticValidator
    {
        public abstract record Error : ISentenceSemanticValidator.Error
        {
            public sealed record ParticleNotAccepted(Bunsetsu Bunsetsu, IPredicateCard Predicate) : Error;
            ...
        }

        // サブルール群
        private readonly IReadOnlyDictionary<ParticleRole, ISentenceSemanticValidateRule> _rules;

        // NOTE: 実際の挙動はVContainerによるインジェクション
        public SentenceSemanticValidator(IEnumerable<ISentenceSemanticValidateRule> rules)
        {
            _rules = rules.ToDictionary(r => r.ParticleRole);
        }

        /// <summary>
        /// 手札を意味検証します。
        /// </summary>
        public ISentenceSemanticValidator.Result Validate(IHand hand)
        {
            var errors = new List<ISentenceSemanticValidator.Error>();

            ... // 割愛

            return errors.Count == 0
                ? new ISentenceSemanticValidator.Result.Success()
                : new ISentenceSemanticValidator.Result.Failure { Errors = errors };
        }
    }
}
```

:::

クリーンアーキテクチャでは _DomainService_ と呼ばれることが多いですが、私は _Service_ ではなく _Function_ と呼ぶことにしています。

- 「サービス」だと副作用や状態管理を含みそうなイメージがあるため。
- 「関数」という意味の名前にしておけば、「Input に対して処理を行い Output を返すだけ」というイメージがしやすいと考えています。
- あくまで、私個人の好みです。

### Repositoryインタフェース

Entityの永続化に関するインタフェースです。

- 例）
  - カードリポジトリ
  - タグリポジトリ

アプリケーション層から利用されます。

具体的な処理はインフラ層で実装します。

::: details 例

```csharp
namespace WordPuzzle.Domain.InGameComponents.Repositories
{
    public interface INounCardDefinitionsRepository
    {
        readonly record struct FilterOption
        {
            public bool? IsEnabled { get; init; }
            public Difficulty? Difficulty { get; init; }

            public static FilterOption All => new();
            public static FilterOption EnabledOnly => new() { IsEnabled = true };
        }

        INounCardDefinition Get(CardKey cardKey);

        IReadOnlyList<INounCardDefinition> GetDefinition(FilterOption filterOption);
    }
}
```

:::

### DomainSettingsインタフェース

ドメイン層の挙動を指定する設定群です。

- 例）
  - 点数計算の倍率、出現するカードリスト、等

インタフェースで定義しておき、実装はInfrastructure層（ScriptableObject）で行います。
ドメイン層ではDIライブラリ（今回はVContainer）が使用できないので、インジェクションはコンストラクタで行います。

::: details 例

```csharp
namespace WordPuzzle.Domain.InGameMechanics.Score.Rules.BasicScore
{
    public interface IBasicScoreSettings : IScoreSettings
    {
        ScorePoint ScorePoint { get; }
    }
}

namespace WordPuzzle.Domain.InGameMechanics.Score.Rules.BasicScore
{
    public class BasicScoreRule : ScoreRule<IBasicScoreSettings>
    {
        public override ScoreType ScoreType => ScoreType.Basic;

        public BasicScoreRule(IBasicScoreSettings settings) : base(settings) { }

        public override bool IsApplicable(IScoreContext context) => _settings.Enabled;

        public override ScoreItem Calculate(IScoreContext context) => new ScoreItem(ScoreType.Basic)
        {
            Point = _settings.ScorePoint,
        };
    }
}
```

:::

### その他のプロバイダインタフェース

時計（`System.DateTime`）や乱数（`Random`）を直接ロジック上から参照しないために、インタフェースを定義しておきます。

少し長くなった＆当ページの本筋から逸れるので、別の記事に分けました。
→[乱数は直接参照しない|zenn](https://zenn.dev/ushibutatory/articles/4624baa67a9acc)

## 所感

当初、「PureC#で書くべし」という発想に対する思い込みから、DIもしてはならないと勘違いしていました。
実際は「DIライブラリに依存してはならない」であって、DIの構造自体は何も問題ないことに途中で気づき、そこから設計の柔軟さが少し上がったような気がします。
