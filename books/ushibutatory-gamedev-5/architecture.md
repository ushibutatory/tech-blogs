---
title: "アーキテクチャ"
---

## 概要

レイヤードアーキテクチャを意識して作成しました。

![レイヤー間の参照](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/architecture-layers.png)

### 構成

| レイヤー       | 概要                         | 参照先                                  |
| -------------- | ---------------------------- | --------------------------------------- |
| Core           | 汎用インタフェース、列挙型   | なし                                    |
| Domain         | データモデル、ドメインルール | Core                                    |
| Application    | ユースケース、状態管理       | Core, Domain                            |
| Presentation   | ゲームオブジェクト、UI       | Core, Application                       |
| Infrastructure | 外部リソース                 | Core, Domain, Application, Presentation |

![アーキテクチャイメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/architecture.png)

※図の中ではCoreは省略

### asmdef配置

`Assembly-CSharp` を使わないようにしています。

![スクショ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/folder-scripts.png)

```text
Assets/_Project/
    Scripts/
        # 各レイヤーのスクリプト
        WordPuzzle/
            WordPuzzle.asmdef
        WordPuzzle.Application/
            WordPuzzle.Application.asmdef
        WordPuzzle.Core/
            WordPuzzle.Core.asmdef
        WordPuzzle.Domain/
            WordPuzzle.Domain.asmdef
        WordPuzzle.Infrastructure/
            WordPuzzle.Infrastructure.asmdef
        WordPuzzle.Presentation/
            WordPuzzle.Presentation.asmdef
    Scripts.Editor/
        # エディタスクリプト
        WordPuzzle.Domain.Editor/
            WordPuzzle.Domain.Editor.asmdef
        ...
    Scripts.Tests/
        # 各レイヤー及びエディターのテストスクリプト
        WordPuzzle.Application.Tests/
            WordPuzzle.Application.Tests.asmdef
        WordPuzzle.Core.Tests/
            WordPuzzle.Core.Tests.asmdef
        ...
    ...
    Input/
        # 例外的に、InputActionsのasmdefはレイヤー構造とは独立させている。
        # レイヤードアーキテクチャの外にある、入力ライブラリという位置づけ。
        WordPuzzle.Input.asmdef
        WordPuzzleInputActions.inputactions
```

## レイヤー詳細

長くなったので分割しました。

- [Core層](./architecture-core)
- [Domain層](./architecture-domain)
- [Application層](./architecture-application)
- [Presentation層](./architecture-presentation)
- [Infrastructure層](./architecture-infrastructure)
- [各レイヤーのテスト](./architecture-testing)

## 所感

ここはどうすればいいのかな、みたいな悩みはたくさんあり、少しずつ対応しながら構築していました。
もう少し経験を積めばもっと柔軟かつ堅牢な構成が作れるようになるでしょうが、まだまだ勉強不足だなと感じます。
