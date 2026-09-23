---
title: "アーキテクチャ詳細 - Core層"
---

## Core層

- すること
  - 全レイヤーに共通の型を定義する。
- しないこと
  - ビジネスロジックを定義してはならない。
  - 外部ライブラリに依存してはならない。

### ValueObject

意味的にはドメインに属しますが、システム全体で同じ意味を持たせたい「値型」は、Core 層に定義しました。

- 例）カードID、セッションID、など

状態を持たず、イミュータブルです。

レコード構造体（record struct）や列挙型として実装することがほとんどです。
定数や型変換、各種演算子もあわせて定義します。

::: details 例

```csharp
namespace WordPuzzle.Core.Card
{
    /// <summary>
    /// カードID
    /// </summary>
    /// <remarks>
    /// <br/>- カードIDは、配られたカードを一意に識別するための値です。
    /// <br/>- 同じ「猫」という名詞カードでも、配られたカードごとに異なるカードIDを持ちます。
    /// </remarks>
    public readonly record struct CardId
    {
        private readonly Guid _id;

        // NOTE: 都度 ToString() するのはコストが高いので、事前に計算して保持しておく
        public readonly string Value;
        public readonly string ShortValue;

        private CardId(Guid id)
        {
            _id = id;
            Value = id.ToString();
            ShortValue = id.ToString("N")[..8];
        }

        public static CardId None { get; } = new(Guid.Empty);
        public static CardId New() => new(Guid.NewGuid());

        public static CardId Of(string id)
        {
            if (string.IsNullOrEmpty(id))
            {
                return None;
            }

            if (id.Replace("-", "").Length < 32)
            {
                // 足りない場合はできるだけ再現する
                return new(Guid.Parse(id.Replace("-", "").PadRight(32, '0')));
            }

            return new(Guid.Parse(id));
        }

        public override string ToString() => ShortValue;
        public string ToFullString() => Value;
    }
}
```

:::

### 汎用の定数や列挙型

意味的にはドメインに属しますが、システム全体で同じ意味を持たせたい「列挙型」は、Core 層に定義しました。

- 例）難易度、学年別漢字配当のキー、など

:::details 例

```csharp
namespace WordPuzzle.Core.Card
{
    /// <summary>
    /// 学年別漢字配当
    /// </summary>
    public enum KanjiGrade
    {
        /// <summary>
        /// 漢字なし（かな、カナのみの言葉）
        /// </summary>
        None = 0,

        /// <summary>
        /// 小学校1年
        /// </summary>
        Grade1 = 1,
        ...
    }
}
```

:::

### DTOインタフェース

意味的にはドメインに属しますが、システム全体で同じように扱いたい「プロパティの集合」は、Core 層にインタフェースを定義しました。

- 例）タグやカードが公開する情報

実装は、Domain層でEntityとして実装したり、Infrastructure層でScriptableObjectとして実装したり、様々です。

レイヤー横断時にEntityをDTOに変換する手間を省くために作ったものです。

::: details Domain層で実装するインタフェースの例

```csharp
namespace WordPuzzle.Core.Tag
{
    /// <summary>
    /// タグのインタフェース
    /// </summary>
    public interface ITag : IEquatable<ITag>
    {
        string Value { get; }
    }
}

namespace WordPuzzle.Core.Card
{
    /// <summary>
    /// カードのインタフェース
    /// </summary>
    /// <remarks>
    /// <br/>- すべてのカードに共通するインタフェースです。
    /// <br/>- 品詞ごとに異なるプロパティはそれぞれの品詞カードインタフェースに定義すること。
    /// </remarks>
    public interface ICard
    {
        CardId Id { get; }
        CardType CardType { get; }

        CardKey CardKey { get; }

        TCard As<TCard>() where TCard : ICard;
    }
}
```

:::

::: details Infrastructure層でSOで実装するインタフェースの例

```csharp
namespace WordPuzzle.Core.Card
{
    /// <summary>
    /// 名詞カードの定義
    /// </summary>
    public interface INounCardDefinition : ICardDefinition
    {
        /// <summary>
        /// 単語：漢字表記
        /// </summary>
        string Word_Kanji { get; }

        /// <summary>
        /// 単語：かな表記
        /// </summary>
        string Word_Kana { get; }

        /// <summary>
        /// 単語：カタカナ表記
        /// </summary>
        string Word_Katakana { get; }

        /// <summary>
        /// 備考
        /// </summary>
        string Remarks { get; }

        /// <summary>
        /// 難易度
        /// </summary>
        Difficulty Difficulty { get; }

        /// <summary>
        /// 学年別漢字配当
        /// </summary>
        KanjiGrade KanjiGrade { get; }

        /// <summary>
        /// タグ：Is
        /// </summary>
        IReadOnlyList<IIsTag> IsTags { get; }

        /// <summary>
        /// タグ：Can
        /// </summary>
        IReadOnlyList<ICanTag> CanTags { get; }

        /// <summary>
        /// タグ：CanBe
        /// </summary>
        IReadOnlyList<ICanBeTag> CanBeTags { get; }
    }

}
```

:::

## 所感

便利ですが、便利すぎるのも考えもので、ここにもの（特にロジック）を置きすぎないように注意しました。
型定義にしても、後から変更が発生する影響範囲が大きいので、便利故に要注意なレイヤーだと思います。
