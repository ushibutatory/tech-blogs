---
title: "アーキテクチャ"
---

## 概要

### 構成

![レイヤー間の参照]()

![アーキテクチャ]()

| レイヤー       | 概要                           | 参照先                                  |
| -------------- | ------------------------------ | --------------------------------------- |
| Core           | 汎用インタフェース、列挙型     | なし                                    |
| Domain         | ビジネスロジック               | Core                                    |
| Application    | ユースケース、進行管理         | Core, Domain                            |
| Presentation   | UI、MonoBehaviour              | Core, Application                       |
| Infrastructure | ScriptableObjectや外部IOの実装 | Core, Domain, Application, Presentation |

プレゼンテーション層の設定をSOで行うために、インフラ層もプレゼンテーション層を参照している。

### asmdef配置

`Assembly-Csharp` を使わないようにしています。

![スクショ](/folder-scripts.png)

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
        WordPuzzle.CardImport/
            WordPuzzle.CardImport.asmdef
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

ここはどうすればよかったのかな、みたいな細かい反省がたくさんあります。
もう少し経験を積めばもっと柔軟かつ堅牢な構成が作れるようになるでしょうが、まだまだ勉強不足だなと感じます。
