---
title: "「タワーディフェンス」(4) フォルダ構成 - C#"
---

最終的に以下のようになりました。

（見やすさのためにアルファベット順ではなく敢えて入れ替えていたりします）

## フォルダ構成（C# - VisualStudioプロジェクトの構成）

```text
# 全体統制
MyGame/
  # - DIコンテナやライフサイクルの設定
  # - 全体の初期化、DontDestroyOnLoadへの配置
  LifetimeScopes/
    RootLifetimeScope.cs   # ルート
    Scenes/
      BootstrapSceneLifetimeScope.cs
      TitleSceneLifetimeScope.cs
      ...
```
```text
# アプリケーション層
MyGame.Application/
  # - ドメインのまとまりごとに名前空間を分ける
  # 例）
  Shop/
    ShopSession.cs                 # ショップ機能のセッション
    ShopSessionState.cs            # その状態
    Events/
      # アプリケーションイベント
      PlayerPurchasedShopItem.cs   # プレイヤーがアイテムを購入した
      ...
    UseCases/
      # ユースケース
      ListPurchasableShopItem.cs   # 購入可能なアイテム一覧を取得する
      PurchaseShopItem.cs          # アイテムを購入する
      ...
  Battle/
    ...
  System/
    ...
```
```text
# コア層
MyGame.Core/
  # - 全レイヤー共通のValueObjectや定数、列挙型を定義する
  # 例）
  Enemy/
    EnemyId.cs    # etc.
  # - 標準の Vector2, Vector3, float などに意味を付けた型定義もCoreで行う
  Time/
    Seconds.cs    # 秒
  Space/
    Angle.cs      # 角度
    Speed.cs      # 速度
    ...
```
```text
# ドメイン層
MyGame.Domain/
  # - ドメインのまとまりごとに定義する
  # - 簡略のためFunction以外は直下に配置するようにした
  # 例）
  Battle/
    Enemy.cs             # エネミーのエンティティクラス
    IEnemyFactory.cs     # Factoryインタフェース
    Funcions/
      CalculateDamage.cs               # ダメージ計算
      CalculateEnemySpawnPosition.cs   # スポーン位置計算
```
```text
# プレゼンテーション層
MyGame.Presentation/
  # MonoBehaviour系
  Components/
    Audio/
      MusicPlayer.cs
      SoundPlayer.cs
    Battle/
      Direction/
        CameraController.cs  # カメラ本体
        CameraMovable.cs     # カメラ移動機能
        CameraShakable.cs    # カメラ振動機能
        ...
      Player/
        PlayerController.cs        # プレイヤー本体
        PlayerHurtBox.cs           # プレイヤー当たり判定
        PlayerHurtBoxCollider.cs   # プレイヤー当たり判定用のコライダー
        PlayerSpawner.cs           # プレイヤーの出現装置
        ...
    
  # コンポーネント間通信リクエスト（PresentationEvent）
  # 上記のComponents/ではなく別の名前空間に分ける。
  # （他コンポーネントはRequestだけを参照できればよいので）
  Requests/
    Audio/
      PlayAudioRequest.cs
      PauseAudioRequest.cs
      UpdateVolumeRequest.cs
      ...
    Scene/
      LoadSceneRequest.cs
      ...

  # InputSystem
  Inputs/
    InputMode.cs     # 入力モード列挙型
    InputActions.cs  # （自動生成）
    Modes/
      GeneralUIMode.cs    # モードごとに入力の検知方法を定義する
      PlayInGameMode.cs
      PauseInGameMode.cs
      ...

  # シーン制御
  # （詳細は別ページに記載）
  Scenes/
    SceneNavigator.cs   # シーンを読み込んだり遷移履歴を保持したりするクラス
    ...

  # UI Toolkit関連
  # （詳細は別ページに記載）
  UIs/
    Shop/
      ShopUIView.cs      # ショップUIの制御本体
      Sections/
        ShopMenuUIViewSection.cs     # ショップメニューセクション
        ShopCatalogUIViewSection.cs  # 商品一覧セクション
    ...

```
```text
# インフラストラクチャ層
MyGame.Infrastructure/
  # アセット管理
  AssetLists/
    IAudioAssetList.cs    # インタフェース
    AudioAssetList.cs     # 実装
    ...

  # Factory実装
  Factories/
    EnemyFactory.cs
    ...
  
  # Repository実装
  Repositories/
    PlayerProfileRepository.cs
    ...
  
  # 外部ファイル保存
  ExternalSave/
    IExternalSaveDataStore.cs   # 外部ファイルI/Oインタフェース
    SaveDataStores/
      PlainJsonSaveDataStore.cs  # プレーンJson形式のファイルI/O実装
      ...                        # 暗号化など追加していく
                                 # （外部アセットを使う場合はちょっと変わるかもしれない）
    SaveData.cs                  # セーブデータ本体
    Sections/                    # 一定のまとまりごとに分ける
      UserProfile.cs     # ユーザプロファイル（ゲーム進捗、アンロック状態など）
      Player.cs          # プレイヤー（Lvなど）
      Shop.cs            # ショップ（在庫状態など）
      ...

  # 設定（ScriptableObject）
  # （詳細は別ページに記述）
  Settings/
    General/
      GeneralSettingsContainer.cs                # 全体設定まとめ
      AudioSettings.cs                           # オーディオ設定
      ...
    Battle/
      BattleSettingsContainer.cs                 # バトル関連設定まとめ
      BattleDirectionCameraMovableSettings.cs    # カメラ移動設定
      BattleDirectionCameraShakableSettings.cs   # カメラ振動設定
      ...
    Enemy/
      EnemySettingsContainer.cs    # エネミー設定まとめ
      EnemyDefinition.cs           # エネミー定義
      EnemySpawnSettings.cs        # エネミー出現設定
      ...
```

```text
# エディタ拡張
MyGame.Editor/
  # 自作クラスのインスペクター表示等
  # （割愛）
```

## 所感

つい付けたくなる汎用的なサフィックス（Controller、Managerなど）をあちこちで付けないように苦労しました。
