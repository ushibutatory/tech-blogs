---
title: "アーキテクチャ詳細 - 各レイヤーのテスト"
---

## テストライブラリの作成

それぞれのレイヤーに対して、テストライブラリを作成しています。

![レイヤーとの関係](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/architecture-testing.png)

スクリプト（`.asmdef`）はレイヤーテストごとに分けて作成します。

![フォルダ構成](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/architecture-testing-unity.png)

例えばドメイン層（`Domain`）に対するテストライブラリ（`Domain.Tests`）は以下のような構成になります。

![サンプル](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/architecture-testing-sample.png)

- `_Helpers`
  - テストをする上で頻出の共通処理やクラスの組み立て等があれば適宜作成します。
  - 例）手札の構築（引数で指定したカードの手札を生成して返す）等
- `_Stubs`
  - ドメイン層にインタフェースが定義されており、Infrastructure層で実装する類の処理をスタブとして定義しています。
    - 例）乱数
  - 基本的に、テスト対象クラスの名前空間を踏襲したフォルダに作成します。
- テストコード
  - これらも、基本的にテスト対象クラスの名前空間を踏襲したフォルダに作成します。

## テストの例

::: details コード

```csharp
using NUnit.Framework;
using System.Collections.Generic;
using System.Linq;
using WordPuzzle.Core.Card;
using WordPuzzle.Core.Tests._Stubs.Card;
using WordPuzzle.Domain.InGameComponents;
using WordPuzzle.Domain.InGameComponents.Cards;
using WordPuzzle.Domain.Tests._Stubs;

namespace WordPuzzle.Domain.Tests.InGameComponents
{
    public class CardDeckTest
    {
        /// <summary>
        /// 指定した枚数のカード定義を適当に作成します。
        /// </summary>
        private static IReadOnlyList<StubNounCardDefinition> _CreateCards(int count)
            => Enumerable.Range(1, count)
                .Select(i => new StubNounCardDefinition { Word_Kana = $"Card{i}" })
                .ToList();

        // --- DrawOne ---

        [Test]
        public void カードを引く()
        {
            var cards = _CreateCards(3);
            var deck = new CardDeck<INounCardDefinition, INounCard>(cards, new StubRandom(), s => new NounCard(s));

            var card = deck.DrawOne();

            Assert.That(cards, Contains.Item(card.Definition));
            Assert.That(deck.RemainCount, Is.EqualTo(cards.Count - 1));
        }

        [Test]
        public void 全カードが過不足なく出る()
        {
            var pool = _CreateCards(5);
            var deck = new CardDeck<INounCardDefinition, NounCard>(pool, new StubRandom(), s => new NounCard(s));

            var drawn = Enumerable.Range(0, pool.Count)
                .Select(_ => deck.DrawOne())
                .ToList();

            Assert.That(drawn.Select(c => c.Definition), Is.EquivalentTo(pool));
            Assert.That(deck.RemainCount, Is.Zero);
        }

        [Test]
        public void 山札が尽きると再シャッフルされ引き続けられる()
        {
            var pool = _CreateCards(3);
            var deck = new CardDeck<INounCardDefinition, NounCard>(pool, new StubRandom(), s => new NounCard(s));

            // 一旦全部引く
            for (var i = 0; i < pool.Count; i++)
            {
                deck.DrawOne();
                Assert.That(deck.RemainCount, Is.EqualTo(pool.Count - i - 1));
            }

            // 更に引く
            var card = deck.DrawOne();
            Assert.That(pool, Contains.Item(card.Definition));
        }

        [Test]
        public void 同じRandomなら同じ順序でカードが引ける()
        {
            var pool = _CreateCards(5);
            var deck1 = new CardDeck<INounCardDefinition, NounCard>(pool, new StubRandom(0.5f), s => new NounCard(s));
            var deck2 = new CardDeck<INounCardDefinition, NounCard>(pool, new StubRandom(0.5f), s => new NounCard(s));

            for (var i = 0; i < pool.Count; i++)
            {
                Assert.That(deck1.DrawOne().Definition, Is.SameAs(deck2.DrawOne().Definition));
            }
        }

        [Test]
        public void 山札をリセットできる()
        {
            var pool = _CreateCards(5);
            var deck = new CardDeck<INounCardDefinition, NounCard>(pool, new StubRandom(), s => new NounCard(s));

            // 数枚引いてから
            deck.DrawOne();
            Assert.That(deck.RemainCount, Is.EqualTo(pool.Count - 1));
            deck.DrawOne();
            Assert.That(deck.RemainCount, Is.EqualTo(pool.Count - 2));

            // リセット
            deck.Reset();

            Assert.That(deck.RemainCount, Is.EqualTo(pool.Count));
        }
    }
}
```

:::

## 所感

スタブをテストライブラリに入れています。
最初は `Domain.Tests`と`Domain.Stubs`みたいに分けて定義しようと思いましたが、複雑さの割に得られるメリットが少ないと感じたので一緒にしてしまいました。

とはいえ、実は 「`Application.Tests`の中で`Domain.Tests`のスタブクラスが使いたい」というような場面はそれなりに多かったので、分けても良かったのではと思います。次開発では分けてみようと思います。
