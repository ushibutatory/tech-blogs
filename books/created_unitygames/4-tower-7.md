---
title: "「タワーディフェンス」(7) ScriptableObject"
---

ScriptableObjectを本格的に使うのは初めてだったので苦労しました。

## 例）ショップアイテムの管理

### 設定本体

`.asset`ファイルとして作成するための定義です。

```csharp
namespace MyGame.Infrastructure.Settings.Shop
{
    [CreateAssetMenu(
        fileName = nameof(ShopItemDefinition),
        menuName = MENU + "Shop/" + nameof(ShopItemDefinition)
    )]
    public class ShopItemDefinition : ScriptableSettings, IShopItem
    {
        // プリミティブ型で設定する
        [SerializeField] private int _id;
        [SerializeField] private ShopItemCategory _category;
        [SerializeField] private string _name;
        ...

        // ValueObject等に変換して使用する
        public ShopItemId Id => ShopItemId.Of(_id);
        public ShopItemCategory Category => _category;
        public string Name => _name;
        ...
    }
}

namespace MyGame.Infrastructure.Settings.Shop
{
    [CreateAssetMenu(
        fileName = nameof(ShopCatalogSettings),
        menuName = MENU + "Shop/" + nameof(ShopCatalogSettings)
    )]
    public class ShopCatalogSettings : ScriptableSettings
    {
        [SerializeField] private List<ShopItemDefinition> _catalog = new();
        ...

        // 読み取り専用にして公開する
        public IReadOnlyList<IShopItem> List() => _catalog;

        // 作業用のボタンなど適宜作成する
        [Button(Name = "Sort（IDの昇順）")]
        private void _SortById()
        {
            _catalog.Sort((a, b) => a.Id.CompareTo(b.Id));
        }
    }
}

// ちなみに、各SOは直接 ScriptableObject を継承せずに、カスタムクラスを1つ挟むようにしています。

namespace MyGame.Infrastructure.Settings.Abstractions
{
    // 今回は Odin Inspector の SerializedScriptableObject を継承しています。
    public abstract class ScriptableSettings : SerializedScriptableObject
    {
        ...
    }
}
```

### 設定のコンテナ

各SOをまとめてDIコンテナで管理するためのクラスです。

```csharp
namespace MyGame.Infrastructure.Settings.Shop
{
    [CreateAssetMenu(
        fileName = nameof(ShopSettingsContainer),
        menuName = MENU + nameof(ShopSettingsContainer)
    )]
    public class ShopSettingsContainer : ScriptableSettings
    {
        [SerializeField] private ShopCatalogSettings _shopCatalogSettings = default!;
        [SerializeField] private ShopCameraSettings _shopCameraSettings = default!;
        ...

        public ShopCatalogSettings ShopCatalogSettings => _shopCatalogSettings;
        public ShopCameraSettings ShopCameraSettings => _shopCameraSettings;
        ...
    }
}
```

### DIコンテナへ登録

定義したクラスからSOアセットを作成し、DIコンテナに登録します。
設定は使用するスコープ単位で登録しました。

```csharp
// 使用したDIコンテナはVContainer
namespace MyGame.LifetimeScopes
{
    public class HomeSceneLifetimeScope : SceneLifetimeScope
    {
        [SerializeField] private ShopSettingsContainer _shopSettingsContainer;
        ...

        protected override void _Configure(IContainerBuilder builder)
        {
            // 設定をそれぞれ個別に登録する
            builder.RegisterInstance(_shopSettingsContainer.ShopCatalogSettings);
            builder.RegisterInstance(_shopSettingsContainer.ShopCameraSettings);
            ...
```

### 設定値を参照する

#### 例）アプリケーション層で参照したい場合

```csharp
namespace MyGame.Application.Home.Shop
{
    // ユースケース：購入可能なアイテムの一覧を取得する
    public class ListPurchasableItems : UseCase<ListPurchasableItems>
    {
        [Inject] private readonly ShopCatalogSettings _shopCatalogSettings = default!;
        ...

        public IReadOnlyCollection<IShopItem> Execute()
        {
            // ショップに設定されている全アイテム一覧を取得
            var allItems = _shopCatalogSettings.List()
                ?? Array.Empty<IShopItem>();
            ...
```

## 所感

インスペクタ上でも直接ValueObject型で編集できればもっと良かったのですが、うまく出来なかったので後回しにしてしまいました。
