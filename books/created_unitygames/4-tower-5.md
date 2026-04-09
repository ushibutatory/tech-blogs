---
title: "「タワーディフェンス」(5) UI Toolkit関連の処理"
---

## 構成

![]()

### UIViewSection

表示・操作するひとまとまりです。

- UXMLに対応します。
  - 複数の `VisualElement` を配置し、UIのまとまりを作成します。
  - 「ヘッダー」「メニュー」「一覧」「詳細」など。
  - UIイベントの購読や描画更新などを行います。
- `MonoBehaviour` を継承しない。

### UIView

いわゆる「画面」です。

- UXMLに対応します。
  - 複数の `UIViewSection` をまとめ、ひとつの画面をデザインする。
  - 「ショップ画面」「ステータス画面」など。
- `MonoBehaviour` を継承しない。

### UINavigator

UIViewの状態を制御します。

- 複数の `UIView` をまとめて、表示・非表示を制御します。
- `MonoBehaviour` を継承する。

### SceneContext

シーン内で起きたイベントに応じてUINavigatorを制御します。

- 例）
  - ショップの前にプレイヤーが到着した → 遷移確認UIViewを表示する。
  - OKが押された → 遷移確認UIViewを非表示にし、ショップメニューUIViewを表示する。

`SceneContext` という名前は、そのシーンの最初から最後までを一貫して監視する理由から命名しました。
（UIView表示切替の他、他シーンへの遷移なども行います）

## 詳細

### Unity

```text
Asset/
  _Project/
    UI/
      Uxml/
        ShopUIView.uxml
      Uxml.Sections/
        ShopHeaderUIViewSection.uxml
        ShopMenuUIViewSection.uxml
        ...
```

### C#

```csharp
namespace MyGame.Presentation.UIs.Shop
{
    // UIViewSection
    // - UIElementに対する表示や更新を行う
    // - ボタン等のイベントを検知して公開する
    public class ShopMenuUIViewSection : UIViewSection<ShopMenuUIViewSection>
    {
        private readonly Subject<Unit> _onBackButtonClicked = new();
        public Observable<Unit> OnBackButtonClicked => _onBackButtonClicked;

        // セクションに配置されるUIElement
        [QueryKey("back-button")] private Button _backButton = default!;
        ...

        public override void Dispose()
        {
            _onBackButtonClicked?.Dispose();
            ...

            base.Dispose();
        }

        protected override void _SetSubscribes()
        {
            // UIElementのイベントを外部に通知する
            _backButton.OnClickAsObservable().Subscribe(_ =>
            {
                _onBackButtonClicked.OnNext(Unit.Default);
            }).AddTo(_disposables);
            ...
        }
    }

    // UIView
    // - セクションを複数組みあわせて画面を構成する
    // - セクションで起きたイベントを外部に公開する
    // - セクション同士を連携させる
    public class ShopUIView : UIView<ShopUIView>
    {
        // セクションを注入する
        [Inject] private readonly ShopHeaderUIViewSection _headerSection = default!;
        [Inject] private readonly ShopMenuUIViewSection _menuSection = default!;
        ...

        // 各イベントを公開する
        // （UIViewはユースケースを実行しない）
        public Observable<ShopItemId> OnPurchaseButtonClicked => _purchaseSection.OnPurchaseButtonClicked;
        public Observable<Unit> OnBackButtonClicked => _menuSection.OnBackButtonClicked;

        protected override void _Initialize(VisualElement root)
        {
            // 各セクションをバインドする
            _headerSection.Bind(root.Q<VisualElement>("section-header"));
            _menuSection.Bind(root.Q<VisualElement>("section-menu"));
            ...
        }

        protected override void _SetSubscribes()
        {
            // セクションで起こったイベントを別セクションに反映する
            // （アプリケーション層を経由しない単純な処理）
            _catalogSection.OnItemSelected.Subscribe(shopItem =>
            {
                // 例）一覧セクションで選択されたアイテムを詳細セクションに表示する
                _purchaseSection.ViewDetail(shopItem);
            }).AddTo(_disposables);
            ...
        }
    }
}
```

UIViewが画面ひとつを表すので、画面切り替えを担うクラスを定義します。

```csharp
    public class HomeSceneUINavigator : SceneUINavigator<HomeSceneUINavigator>
    {
        // UXMLをアサイン
        [SerializeField] private UIDocument _homeUIDocument = default!;
        [SerializeField] private UIDocument _shopUIDocument = default!;
        ...

        // UIViewを注入
        [Inject] private readonly HomeUIView _homeUIView = default!;
        [Inject] private readonly ShopUIView _shopUIView = default!;
        ...

        // イベントを公開
        public Observable<Unit> OnOpenShopButtonClicked => _homeUIView.OnShopButtonClicked;
        public Observable<Unit> OnCloseShopButtonClicked => _shopUIView.OnBackButtonClicked;
        ...

        public void Initialize()
        {
            // UXMLとUIViewを関連付ける
            _homeUIView.Bind(_homeUIDocument.rootVisualElement);
            _shopUIView.Bind(_shopUIDocument.rootVisualElement);
            ...

            // 初期状態ではすべて非表示とする
            HideAll();
        }

        public void HideAll()
        {
            Hide_Home();
            Hide_Shop();
            ...
        }

        // UIViewの表示切替
        public void Show_Home() => _homeUIView.Show();
        public void Hide_Home() => _homeUIView.Hide();

        public void Show_Shop() => _shopUIView.Show();
        public void Hide_Shop() => _shopUIView.Hide();

        ...
```

UIの切り替えは、SceneContextクラスで定義します。
SceneContextクラスは各シーンに1つずつ配置し、シーンの初期化、イベントや操作に応じてシーン遷移やUI切り替え、BGM変更、入力モードの変更などを行う独自クラスです。

```csharp
namespace MyGmae.Presentation.SceneManagement.Home
{
    public class HomeSceneContext : SceneContext<HomeSceneContext>
    {
        protected override SceneId SceneId => SceneId.Home;

        ...

        protected override void _Initialize(ISceneParameter.NoParameter parameter)
        {
            // 入力無効化
            _presentation.InputActions.DisableAll();

            // BGM開始
            _presentation.PlayMusicRequest.Publish(new PlayMusicRequest
            {
                MusicId = MusicId.Home,
            });

            // UI初期化
            _presentation.UINavigator.Initialize();

            // ホームセッションを開始
            _application.StartHomeSession.ExecuteAsync().Forget();
        }

        protected override void _SetupSubscribes()
        {
            ...

            _presentation.UINavigator.OnOpenShopButtonClicked.Subscribe(async _ =>
            {
                await UniTask.WhenAll(
                    UniTask.Create(async () =>
                    {
                        // カメラ移動
                        await _presentation.Camera.Movable.MoveToShopPositionAsync();
                    }),
                    UniTask.Create(async () =>
                    {
                        // 入力モード設定
                        _presentation.InputActions.SetInputMode(InputMode.Common_UI);

                        // UI非表示
                        _presentation.UINavigator.Hide_Home();
                    }));

                // UI表示
                _presentation.UINavigator.Show_Shop();
            }).AddTo(this);

            _presentation.UINavigator.OnCloseShopButtonClicked.Subscribe(async _ =>
            {
                await UniTask.WhenAll(
                    UniTask.Create(async () =>
                    {
                        // カメラ移動
                        await _presentation.Camera.Movable.MoveToHomePositionAsync();
                    }),
                    UniTask.Create(async () =>
                    {
                        // 入力モード設定
                        _presentation.InputActions.SetInputMode(InputMode.Common_UI);

                        // UI非表示
                        _presentation.UINavigator.Hide_Shop();
                    }));

                // UI表示
                _presentation.UINavigator.Show_Home();
            }).AddTo(this);

            _presentation.OnEnteringBattle.Subscribe(_ =>
            {
                // ホームセッション終了
                _application.EndHomeSession.Execute();

                // シーン遷移：バトルへ
                _presentation.LoadSceneRequest.Publish(LoadSceneRequest.Load(SceneId.Battle, new BattleSceneParameter
                {
                    ...
                }));
            }).AddTo(this);

            ...
```

## USS

以下のような構成にしました。

```text
# デバッグ用のスタイル
__debug.uss       # 本来透明な領域に色をつけて確認しやすくするなど

# 汎用スタイル
_components.uss   # ボタンやテキスト等の共通スタイル
_layouts.uss      # ヘッダ、フッタ、各種エリアの幅など
_variables.uss    # 変数定義（フォントサイズ等）

# 個別スタイル
HomeUIView.uss
ShopUIView.uss
...
```

## 所感

初めて触ったので非常に苦戦しました。
特に MonoBehaviour でないものを画面上で動作させる、という感覚が掴めませんでした（今も怪しい）。

個人的には「UI」という言葉がすごく曖昧な概念に感じます。
かといって「HUD」とも違うようなUIは、いつも「画面」という言い方になってしまいます。
