---
title: "フォルダ構成 - C#"
---

最終的に以下のようになりました。

![フォルダ構成 - C#](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/folder-scripts.png)

- 見やすさのためにアルファベット順ではなく敢えて入れ替えたりしています。

## フォルダ構成（VisualStudioプロジェクト）

### 全体の統制

`Assembly-CSharp.dll` を使わず、明示的に `.asmdef` を作成してスクリプトをまとめています。

- DIコンテナやライフサイクルの設定
- 初期化処理
- 永続シーンの追加ロード

名前空間は大まかに **用途ごと** に分けます。

::: details フォルダ構成

```text
WordPuzzle/
    Diagnostics/
        GlobalExceptionMonitor.cs      # 全体例外監視
        ...
    LifetimeScopes/
        RootLifetimeScope.cs          # ルート
        Scenes/
            BootstrapSceneLifetimeScope.cs
            HomeSceneLifetimeScope.cs
            ...
```

:::

### アーキテクチャレイヤー：Core層

- 全レイヤー共通の ValueObject や定数、列挙型
- 標準の Vector2, Vector3, float などに意味を付けた型定義

名前空間は大まかに **意味ごと** に分けます。

::: details フォルダ構成

```text
WordPuzzle.Core/
    Card/
        CardId.cs      # readonly record struct
        CardType.cs    # enum
    Time/
        Seconds.cs    # 秒
    Space/
        Angle.cs      # 角度
        Speed.cs      # 速度
        ...
```

:::

### アーキテクチャレイヤー：Domain層

- ドメインルールの表現

名前空間は大まかに **ドメイン領域ごと** に分けます。

アプリケーション層のテストのために、インタフェースと実装を分けるようにしています。

::: details フォルダ構成

```text
WordPuzzle.Domain/
    InGameComponents/                    # ゲームで使用する道具（カードや手札、デッキなど）
        Cards/
            NounCard.cs                  # 名詞カード（entity）
            VerbCard.cs                  # 動詞カード（entity）
            ...
            Repositories/
                INounCardDefinitionsRepository.cs   # 名詞カードの定義情報のリポジトリインタフェース
                ...
        Tags/
            ...
        ...
    InGameMechanics/                     # ゲームで採用するルール（判定、採点など）
        Grammar.ConnectionStructure/     # 接続ルール（助詞の使い方）
            IConnectionStructureValidator.cs    # ルールを検索して検証するインタフェース
            ConnectionStructureValidator.cs     # その実装
            Rules/                            # 適用するルール群
                NounStructureRule.cs          # 名詞に対する助詞のルール
                VerbStructureRule.cs          # 動詞に対する助詞のルール
                ...
        Grammar.SentenceSemantic/        # 意味ルール（タグによる検証）
            ...
        Grammar.SentenceStructure/       # 構成ルール（品詞の組み合わせによる文章の成立ルール）
            ...
        Score/                          # 採点ルール
            IScoreCalculator.cs          # 採点インタフェース
            ScoreCalculator.cs           # その実装
            Rules/
                BasicScoreRule.cs
                ...
            ScoreResult.cs               # 採点結果（インタフェースはCore層にある）
        ...
    PlayerStats/                        # 統計
        Repositories/
            IPlayerStatsRepository.cs
            ...
        PlayerStats.cs
        ...
    ...
```

:::

### アーキテクチャレイヤー：Application層

- ユースケースの表現（ドメインルールの組み合わせ）
- セッション（状態管理）とその更新の通知
- どのユースケースにも属さないサービス

名前空間は大まかに **ドメイン領域ごと** に分けます。

プレゼンテーション層のテストのために、インタフェースと実装を分けるようにしています。

::: details フォルダ構成

```text
WordPuzzle.Application/
    InGame/
        IInGameSessionManager.cs        # セッション管理
        InGameSessionManager.cs         # その実装
        IInGameSessionState.cs          # セッション状態の公開インタフェース
        InGameSessionState.cs           # その実装
        Events/                        # 通知イベント
            HandUpdated.cs              # 手札が更新された
            FieldUpdated.cs             # フィールドが更新された
            ...
        UseCases/                      # ユースケースの公開インタフェース
            IStartInGame.cs            # インゲームを開始する
            ITryAddCardToHand.cs       # カードを手札に追加する
            ISupplyCard.cs             # カードをフィールドに配る
            ...
        UseCases.Implements/           # ユースケースの実装
            ...
    PlayerStats/
        IPlayerStatsSessionManager.cs
        PlayerStatsSessionManager.cs
        ...
        Events/
            ...
        UseCases/
            ...
        UseCases.Implements/
            ...
    Services/
        IPlayerProfileService.cs       # プレイヤープロファイルの参照・更新サービス
        ...
        Implements/
            PlayerProfileService.cs    # その実装
            ...
```

:::

### アーキテクチャレイヤー：Presentation層

- ゲームオブジェクトやUIの描画、制御、演出
- ユーザからの操作検知
- 必要なタイミングでのユースケースの実行

名前空間は大きく2つの方針で分けます。

- シーン固有のもの → **シーンごと** に名前空間を分ける
  - 例）Audio、Effect、Localization、...
- 全体に関わるもの → **用途ごと** に名前空間を分ける

::: details フォルダ構成

```text
WordPuzzle.Presentation/
    Audio/                              # オーディオ関連
        MusicPlayer.cs                  # BGMプレイヤー（MonoBehaviour）
        SoundPlayer.cs                  # SEプレイヤー（MonoBehaviour）
        Requests/                       # 操作リクエスト
            PlayMusicRequest.cs
            ...
        Settings/                       # 設定インタフェース（実装はInfrastructure層のSO）
            IMusicSettings.cs
            ISoundSettings.cs
            ...
    Effect/                             # エフェクト関連
        ...                             # 略（Audioと似ている）
    ...
    Scenes/                              # シーンごと
        Bootstrap/
        Home/
        Persistent/
        InGame/
            InGameSceneDirector.cs         # シーンの統括（非MonoBehaviour。VContainer.Unity.IStartable）
            InGameSceneParameter.cs        # シーンの起動パラメータ（record）
            Camera/                       # カメラ関連
                InGameCameraComponent.cs   # カメラの本体コンポーネント（MonoBehaviour）
                InGameCameraMovable.cs     # カメラの移動用機能コンポーネント（MonoBehaviour）
                ...
                Settings/
                    IInGameCameraSettings.cs # カメラの設定インタフェース（実装はInfrastructure層のSO）
                    ...
            Components/                   # その他の雑多なコンポーネント群
                CardComponent.cs          # カード本体
                CardMovable.cs            # カードの移動用機能コンポーネント
                CardCollider.cs           # カードの当たり判定コンポーネント
                CardPool.cs               # カードの管理プール
                CardSpawner.cs            # カード生成と初期化
                DeckComponent.cs          # 山札の本体
                ...
                Settings/
                    IInGameCardAnimationSettings.cs # カードのアニメーション設定インタフェース
                    ...
            Input/                        # 入力
                InGameInputHandler.cs     # InputActionsを扱いやすいイベントに変換する
            Listeners/                    # アプリケーションイベントの購読
                HandUpdatedListener.cs    # 手札が更新されたことを検知して他のコンポーネントを操作する
                ...
            UI/                           # UI
                InGameUINavigator.cs      # UIの管理
                InGameUIView.cs           # 1つのUI Documentの表示を制御する
                Sections/
                    InGameHUDSection.cs  # UI部品
                    InGameMenuSection.cs
                    ...
        ...
```

:::

Componentsが肥大化しがちなので、必要に応じて管理しやすい単位に名前空間を分けてもいいと思います。

### アーキテクチャレイヤー：Infrastructure層

- 抽象化された各レイヤーのインタフェースの具体的な実装
  - _ScriptableObject_ によるアセット化
  - 外部I/O

名前空間は大まかに **対象レイヤーの用途ごと** に分けます。

::: details フォルダ構成

```text
WordPuzzle.Infrastructure/
    Adapters.Domain/                                   # ドメイン層のアダプタ
        ProductDateTimeProvider.cs                     # プロダクト用日時プロバイダ
        ProductRandomFactory.cs                        # プロダクト用乱数ファクトリ
        SeededRandom.cs                                # 乱数の実装
        PlayerProfile/
            LocalPlayerProfileRepository.cs            # プレイヤープロファイルリポジトリ（ローカルI/O）
            FirebasePlayerProfileRepository.cs         # 上記のFirebase版
            ...
    Settings.Domain/                                   # ドメイン層の設定実装
        IDomainSettings.cs                             # ドメイン層設定のまとめインタフェース
        DomainSettings.cs                              # その実装（ScriptableObject）
        Card/
            NounCardDefinition.cs                      # 名詞カード1枚分の定義（ScriptableObject）
            NounCardDefinitionsRepository.cs           # 名詞カードリポジトリ（ScriptableObject）
            ...
        Tag/
            CanTag.cs
            CanTagRepository.cs
            CanBeTag.cs
            CanBeTagRepository.cs
            IsTag.cs
            IsTagRepository.cs
            ...
        Score/
            ...
    Settings.Application/
        ...
```

:::

### Scripts.Editor

レイヤーごとに作成しています。

- インスペクタ用の EditorGUI
- ちょっとした開発補助ツール

名前空間は、対象レイヤーの名前空間を基本的に踏襲します。

::: details フォルダ構成

```text
WordPuzzle.Presentation.Editor/
    InGame/
        Components/
            NounCardComponentEditor.cs  # 名詞カードコンポーネントのEditorGUI
            VerbCardComponentEditor.cs  # 動詞カードコンポーネントのEditorGUI
            ...
WordPuzzle.Infrastructure.Editor/
    Tools.CardImport/                   # カードデータインポートツール
        CardImporterWindows.cs          # GUI
        CardImporter.cs                 # 処理本体
        Parser/
            NounCardParser.cs           # 名詞カードパーサー（スプレッドシート→アセット化）
            ...
        Externals.Google/
            SpreadSheetsClient.cs       # スプレッドシート操作クライアント
            ...
    Tools.GoogleAuth/                   # Googleリソース操作用の認証ツール
        ...
```

:::

### テストライブラリ

レイヤーごとに作成しています。

- 正常系
- 異常系
- スタブ

名前空間は、対象レイヤーの名前空間を基本的に踏襲します。

::: details フォルダ構成

```text
WordPuzzle.Domain.Tests/
    _Stubs/
        StubRandom.cs   # 乱数スタブ
        ...
    InGameComponents/
        CardDeckTest.cs # 山札のテスト
        HandTest.cs     # 手札のテスト
        ...
    InGameMechanics/
        Grammar/
            SemanticValidatorTest.cs    # 意味検証ルールのテスト
            StructureValidatorTest.cs   # 構造検証ルールのテスト
            ...
```

:::

## 所感

やっと慣れてきたように感じますが、まだまだ「あれはどこにあるんだっけ」と思う瞬間があるので、もっと精査していければと思います。
