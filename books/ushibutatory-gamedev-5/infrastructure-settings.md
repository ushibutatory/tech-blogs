---
title: "Infrastructure - 設定の構成"
---

たくさんの _ScriptableObject_ から設定アセットを作成しますが、それらをどのような構成で管理するかについて整理しました。

## C#スクリプト内の構成

![イメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/infrastructure-settings-csharp.png)

Infrastructure層に名前空間を定義し、それぞれ以下のようにSOを作成していきます。

- Settings
  - `ProductSettings`
    - 本番用の設定をまとめています。
  - `DebugSettings`
    - デバッグ用の設定を入れています。
  - SO基底クラス
    - 後述
- Settings.Domain
- Settings.Application
- Settings.Presentation
- Settings.Infrastructure
  - 各レイヤーのSettingsインタフェースの実装クラス群です。

### `ProductSettings`

各レイヤーの設定をまとめるためのクラスです。

```csharp
namespace WordPuzzle.Infrastructure.Settings
{
    public interface IProductSettings
    {
        IDomainSettings Domain { get; }
        IApplicationSettings Application { get; }
        IPresentationSettings Presentation { get; }
        IInfrastructureSettings Infrastructure { get; }
    }
}
```

::: details SO

同じくInfrastructure層で実装を定義します。

```csharp
namespace WordPuzzle.Infrastructure.Settings
{
    [CreateAssetMenu(
        fileName = nameof(ProductSettings),
        menuName = MENU + nameof(ProductSettings),
        order = 0
    )]
    public class ProductSettings : ScriptableSettings, IProductSettings
    {
        [Header("Layer Settings")]
        [SerializeField] private DomainSettings _domain = default!;
        [SerializeField] private ApplicationSettings _application = default!;
        [SerializeField] private PresentationSettings _presentation = default!;
        [SerializeField] private InfrastructureSettings _infrastructure = default!;

        public IDomainSettings Domain => _domain;
        public IApplicationSettings Application => _application;
        public IPresentationSettings Presentation => _presentation;
        public IInfrastructureSettings Infrastructure => _infrastructure;
    }
}
```

:::

`RootLifetimeScope`にアタッチできるようにしておき、各シーンの _LifetimeScope_ はここから必要な設定を個別に取得するようにしています。

::: details コード

`RootLifetimeScope` で `ProductSettings` をアタッチする。

```csharp
namespace WordPuzzle.LifetimeScopes
{
    public class RootLifetimeScope : LifetimeScope
    {
        [Header("Settings")]
        [SerializeField] private ProductSettings _productSettings = default!;
        [SerializeField] private DebugSettings _debugSettings = default!;
        ...

        protected override void Configure(IContainerBuilder builder)
        {
            base.Configure(builder);

            // Diagnostics
            _Configure_Diagnostics(builder);

            // Logger
            _Configure_Logger(builder);

            // MessagePipe
            var messagePipeOptions = builder.RegisterMessagePipe(options =>
            {
                options.InstanceLifetime = InstanceLifetime.Singleton;
                options.DefaultAsyncPublishStrategy = AsyncPublishStrategy.Parallel;
            });

            // Settings
            _Configure_Settings(builder);
        }

        private void _Configure_Settings(IContainerBuilder builder)
        {
            builder.RegisterInstance<IProductSettings>(_productSettings);
            builder.RegisterInstance<IDebugSettings>(_debugSettings);
        }
    }
}
```

シーンの _LifetimeScope_ はそこから必要な設定を取得してインジェクションする。

```csharp
// これは基底クラス
namespace WordPuzzle.LifetimeScopes.Scenes
{
    public abstract class SceneLifetimeScope : LifetimeScope
    {
        protected override void Configure(IContainerBuilder builder)
        {
            base.Configure(builder);

            // 親スコープからMessagePipeOptionsを取得
            var messagePipeOptions = Parent.Container.Resolve<MessagePipeOptions>();

            // 親スコープからプロダクト設定を取得
            var settings = Parent.Container.Resolve<IProductSettings>();

            // 個別設定
            _Configure(builder, settings, messagePipeOptions);
        }

        protected abstract void _Configure(IContainerBuilder builder, IProductSettings settings, MessagePipeOptions messagePipeOptions);
    }
}

// これはシーンのLifetimeScopeクラス
namespace WordPuzzle.LifetimeScopes.Scenes
{
    public class InGameSceneLifetimeScope : SceneLifetimeScope
    {
        protected override void _Configure(IContainerBuilder builder, IProductSettings settings, MessagePipeOptions messagePipeOptions)
        {
            // ------ Domain ------
            ...

            // ------ Application ------
            ...

            // ------ Presentation ------
            ...
            // Presentation - Settings
            builder.RegisterInstance(settings.Presentation.CameraSettings);
            builder.RegisterInstance(settings.Presentation.CardAnimationSettings);
            ...
        }
    }
}
```

:::

### `DebugSettings`

入れ物は作ったけど、今回は活用できませんでした。
以下のような設定をできるようにする想定で作成していました。

- デバッグログの出力On/Off
- Coliderなどを可視化
- 乱数の固定

### SO基底クラス

設定クラスは `UnityEngine.ScriptableObject` を直接継承するのではなく、ひとつ抽象クラスを挟むようにしています。

私の場合、[Odin Inspector](https://odininspector.com/) を使うことが多いので、以下のように定義しています。

```csharp
namespace WordPuzzle.Infrastructure.Settings
{
    /// <summary>
    /// ScriptableObject設定基底クラス
    /// </summary>
    /// <remarks>
    /// <br/>- Odin Inspector の SerializedScriptableObject を継承しています。
    /// </remarks>
    public abstract class ScriptableSettings : Sirenix.OdinInspector.SerializedScriptableObject
    {
        ...
    }
}
```

::: details 使用例

```csharp
namespace WordPuzzle.Infrastructure.Settings.Infrastructure.Report
{
    [CreateAssetMenu(
        fileName = nameof(SampleSettings),
        menuName = ... + nameof(ValidationReportSettings),
    )]
    public class SomeSettings : ScriptableSettings, ISomeSettings
    {
        ...
    }
}
```

:::

## Unityプロジェクト内の配置

![イメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/infrastructure-settings-unity.png)

探しやすくするため、基本的にはC#側の構成と揃えたフォルダ構成でアセットを作成します。

## 所感

設定まわりは開発中何度も編集するところなので、どうすれば認知負荷が下がるかを考えてこういう構成にしてみました。

- 呼び出し側
  - 設定化したい内容をインタフェースで定義する
  - インジェクションして値を参照する
- 設定の作成
  - SOクラスを作成する
  - レイヤーのSettingsクラスに追加する
  - シーンのLifetimeScopeでインジェクションする

……こういうものなんですかね。
