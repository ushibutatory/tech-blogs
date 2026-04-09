---
title: "「タワーディフェンス」(3) フォルダ構成 - Unity"
---

最終的に以下のようになりました。

（見やすさのためにアルファベット順ではなく敢えて入れ替えていたりします）

## フォルダ構成（Unityプロジェクト）

```text
Assets/
  _Project/              # 原則、この中に成果物を入れる
    Scenes/              # シーン（.unity）
      ...

    Prefabs/
      System/            # システム系コンポーネント（オーディオなど）
      Common/            # 汎用プレハブ
        Character/
        Environment/
        UI/
        ...
      Scene/             # シーンごとの固有プレハブ
        Title/
          Environment/
          UI/
          ...
        Home/
          Environment/
          UI/
          ...

    Scripts/             # C#（レイヤーごとにasmdefを作成する）
      MyGame/
        MyGame.asmdef
        ...
      MyGame.Domain/
        MyGame.Domain.asmdef
        ...
    Scripts.Tests/       # テスト（Scripts の隣に並ぶように命名）
      MyGame.Tests/
        MyGame.Tests.asmdef
        ...
      MyGame.Domain.Tests/
        MyGame.Domain.Tests.asmdef
        ...

    Settings              # Settings（ScriptableObject）群
      ...

    Art/                 # アート関連
      Environment/       # 環境（床、壁、建造物など）
        Materials/       # マテリアル
        Models/          # 3Dモデル
        Textures/        # テクスチャ
      Character/         # キャラクタ
      Item/              # アイテム
      ...

    UI/                  # UI Toolkit関連
      ...

    Input/               # 入力管理（.inputactions）
      ...

    Localization/        # ローカライゼーション関連
      StringTables/
        ...

  # 外部アセット
  # → `_Project` の外に置く
  Plugins/
  Packages/

  # 外部アセットのうち特にアート系アセット
  # → `_Project` の外に置き、Addressablesで管理する。
  ThirdParty/
    アセット名、配布サイト名 など
```

## 意識したこと

### 単数形か、複数形か

大まかに以下のような方針で名前をつけるようにしました。

- **複数形** ... 同じ性質のもの（ファイル）を集めたもの
  - C#ファイル（`.cs`）の集合 → `Scripts/`
  - フォントアセット（`.ttf` + `.asset`）の集合 → `Fonts/`
  - シーン（`.unity`）の集合 → `Scenes/`
- **単数形** ... カテゴリやドメインで分類したもの
  - キャラクター関連 → `Character/`
  - UI 関連 → `UI/`

ざっくり言うと、「～たち」と表せるものは複数形、「～関連」と表せるものは単数形、というイメージです。

あとは Unity 業界の慣例や公式などを調べつつ、あまりにも圧倒的多数派がいるようであればそっちに寄せたり寄せなかったりしています。

### `.asmdef` の管理

アーキテクチャレイヤーごとに分けています。

`Assembly-CSharp.dll` は使用せず、すべての C# ファイルが必ず何かしらの `.asmdef` で管理されるようにしました。

また、レイヤー間の参照を明示的に管理するために、`_Project` 内にあるすべての `.asmdef` は `autoReferenced: false` にしています。
（`Assembly-CSharp.dll`がなければ意味がない設定らしいですが）

### ThirdPartyフォルダ

インポートした外部アセットのうち、特にアート系アセットを入れています。
AssetStoreからインポートした場合はそのフォルダ名のままとし、それ以外の配布サイト等から取得した場合は、ドメインをフォルダ名にして管理するようにしました。

![ThirdPartyフォルダ](フォルダ構成とフォルダ名_ThirdParty.png)

## 色々な悩み

- 「`UI` は UIComponent の集合なんだから `UIs` または `UIComponents` ではないのか？」
  - 今回は「キャラクター・環境・UI という分類」としました。
  - 解釈なので、どっちでもいいと思います。
- 「`Art`？　`Arts`？」
  - 中に入っているファイルが 3D モデルだったりマテリアルだったり、色んな種類があるため「複数形の命名ルール」に当てはまらない、としました。
  - でも、`Arts` でもいいと思います。
- 「`Prefabs/Scene` は `Prefabs/Scenes` ではないのか？」
  - シーン（`.unity`）を複数集めたフォルダではなく、「システム系・汎用・シーン固有 という分類」なので、`Prefabs/Scene` としました。
  - ちなみにシーン（`.unity`）を集めたフォルダ名は「`Scenes`」としています。
- 「`ThirdParty/` を分ける意味？」
  - お借りしたアセット（成果物）と、自身の成果物を明確に区別するためです。
  - ただし、 `Addressables` 内では区別なくラベリングしています。

## 所感

途中でフォルダ構成を変えると参照が切れて大変だったので、できるだけ最初に固めたほうがいいなと思いました。
