---
title: "フォルダ構成 - Unityプロジェクト"
---

![フォルダ構成 - Unity](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/folder-unity.png)

最終的に以下のようになりました。

- 見やすさのためにアルファベット順ではなく敢えて入れ替えたりしています。

## フォルダ構成（Unityプロジェクト）

```text
Assets/
    _Project/   # 原則、この中に成果物を入れる
        Scenes/ # シーンごとのコンポーネント
            FooScene/
                FooScene.scene
                FooSceneLifetimeScope.prefab
                Prefabs/    # シーン内で使用するPrefab
                    ...

        Scripts/             # C#スクリプト（レイヤーごとにasmdefを作成する）
            WordPuzzle/
                WordPuzzle.asmdef
                ...
            WordPuzzle.Domain/
                WordPuzzle.Domain.asmdef
                ...
            ...
        Scripts.Tests/       # テスト（Scripts の隣に並ぶように命名）
            WordPuzzle.Tests/
                WordPuzzle.Tests.asmdef
                ...
            WordPuzzle.Domain.Tests/
                WordPuzzle.Domain.Tests.asmdef
                ...
            ...
        Scripts.Editors/
            WordPuzzle.Core.Editor/             # 独自型をインスペクタ表示するためのEditorライブラリ
                WordPuzzle.Core.Editor.asmdef
                ...
            ...                                 # 必要に応じてレイヤーごとにEditorを定義する

        Settings/              # Settings（ScriptableObject）群
            WordPuzzle.Domain/
            WordPuzzle.Application/
            ...

        Database/           # スプレッドシートから自動生成したアセット群
            ...

        Art/                 # アート関連（テクスチャ、マテリアル等）
            ...

        UI/                  # UI関連（今回はUI Toolkitを使用）
            ...

        Input/                                  # 入力定義
            WordPuzzle.Input.asmdef             # asmdef を分けておく
            WordPuzzleInputActions.inputactions
            WordPuzzleInputActions.cs           # 自動生成される.csファイル

        Localization/        # ローカライゼーション関連
            LocalizationSettings.asset
            Japanese.asset
            ...

    # 外部アセットは `_Project` の外に置き、Addressablesで管理する。
    ThirdParty/
        sample.com/         # アセット名、配布サイト名、作者アカウント名などでフォルダ名をつける
            ...
        @ushibutatory/
            ...
```

## 各フォルダの補足

### _Assets/\_Project/Scenes_

シーンごとにフォルダを作成します。

シーン内で使用する Prefab を格納します。

別のシーンで使いたい Prefab もありますが、基本的に「最も主に使用するシーン」で管理するものとします。
つまり、一番使用するシーンで Prefab 編集されて困ることがある場合、それはもう別の Prefab にすべきでしょ、という判断となります。

よって、今回は「Common」のようなフォルダは作りませんでした。

エフェクトやダイアログなどは、`PersistentScene`（[別ページ](./presentation-persistent)） で生成・破棄を管理しているのでそちらのフォルダで Prefab 管理しました。

### _Assets/\_Project/Database_

今回の開発では、大量のデータをスプレッドシートで管理していて、それを一括で取り込むツールを別で作成して使用していました。

そういった「手動で編集しないSOアセット」については、「手動で編集するSOアセット」を格納する「_/\_Project/Settings_」フォルダに入れず、別フォルダで管理することとしました。

「_Assets/\_Project/Settings_」の中にサブフォルダを作ってそこに格納するのでも良かった気がします。多分どちらでもよい。

### _Assets/\_Project/Input_

`.asmdef` を分けています。

### _Assets/\_Project/Localization_

`.asmdef` を分けませんでしたが、分けてもよかったのではないかと今は思っています。

### _Assets/\_Project/Settings_

[別ページ](./infrastructure-settings)にまとめました。

### _Assets/\_Project/UI_

[別ページ](./presentation-ui)にまとめました。

### _Assets/ThirdParty_

インポートした外部アセットのうち、特にアート系アセットを入れています。
AssetStoreからインポートした場合はそのフォルダ名のままとし、それ以外の配布サイト等から取得した場合は、ドメインをフォルダ名にして管理するようにしました。

お借りしたアセット（成果物）と、自身の成果物を明確に区別するためです。
ただし、 Addressables 内では区別なくラベリングしています。

## ポイント

### 単数形か、複数形か

大まかに以下のような方針で名前をつけるようにしています。

- 複数形 ... 同じ性質のもの（ファイル）を集めたもの
  - C#ファイル（.cs）の集合 → Script**s**
  - フォントアセット（.ttf + .asset）の集合 → Font**s**
  - シーン（.unity）の集合 → Scene**s**
- 単数形 ... カテゴリやドメインで分類したもの
  - キャラクター関連 → Character
  - UI 関連 → UI

ざっくり言うと、

- 「～たち」と表せるものは複数形
- 「～関連」と表せるものは単数形

というイメージです。

あとは Unity 業界の慣例や公式などを調べつつ、あまりにも圧倒的多数派がいるようであればそっちに寄せたり寄せなかったりしています。

### `.asmdef` の分割

基本はアーキテクチャレイヤーに沿って分割しています。

`Assembly-CSharp.dll` は使用せず、すべての C# ファイルを必ず何かしらの `.asmdef` で管理するようにしました。

また、レイヤー間の参照関係を明示的に管理するために、自作の `.asmdef` は `autoReferenced: false` にしています。
（`Assembly-CSharp.dll`がなければ意味がない設定らしいですが……）

## 所感

こういうのはテンプレ化しておきたいですね。できるんですかね、できそうですよね。次開発の前に調べておきます。
