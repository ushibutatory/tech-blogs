---
title: "Presentation - 永続シーン"
---

コンポーネントについて、シーンを横断して永続させたい時があります。

- 例）
  - 全シーンで参照・更新するプレイヤープロファイル
  - シーン遷移用の共通コンポーネント
  - シーン横断で途切れないように音楽再生するコンポーネント、等

これまで、そういったものは `DontDestroyOnLoad` に配置していたのですが、今回、明示的に永続専用シーンを作成して管理するようにしてみました。

## 永続コンポーネント用のシーン「PersistentScene」

![Hierarchy](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/persistent-hierarchy.png)

永続シーンで管理するコンポーネントのうち、MonoBehaviourを継承しているものたちをシーン内に配置していきます。

## PersistentScene用のLifetimeScope

永続シーンで管理するコンポーネントをシーン用の _LifetimeScope_ に登録します。

::: details コード

```csharp
namespace WordPuzzle.LifetimeScopes.Scenes
{
    public class PersistentSceneLifetimeScope : SceneLifetimeScope
    {
        protected override void _Configure(IContainerBuilder builder, IProductSettings settings, MessagePipeOptions messagePipeOptions)
        {
            // ------ Domain ------
            // Domain - PlayeyProfile
            builder.Register<IPlayerProfileRepository, LocalPlayerProfileRepository>(Lifetime.Singleton);
            ...

            // ------ Application ------
            // Application - Session
            builder.Register<IPlayerProfileSessionManager, PlayerProfileSessionManager>(Lifetime.Singleton);
            builder.Register<PlayerProfileSessionState>(Lifetime.Singleton);
            ...

            // ------ Infrastructure ------
            // PlayerProfile - Settings
            builder.RegisterInstance<IPlayerProfileSettings>(settings.Infrastructure.PlayerProfileSettings);
            ...

            // ------ Presentation ------
            // Presentation - SceneNavigator
            builder.Register<SceneNavigator>(Lifetime.Singleton);
            ...

            // Presentation - Localization
            builder.RegisterInstance<ILocalizationService>(settings.Presentation.LocalizationService);
            ...

            // Presentation - Audio
            builder.Register<IAudioController, AudioController>(Lifetime.Scoped);
            ...

            // Presentation - Audio - Music
            builder.RegisterComponentInHierarchy<MusicPlayer>();
            ...

            // Presentation - Audio - Music - Events
            builder.RegisterMessageBroker<PlayMusicRequest>(messagePipeOptions);
            ...

            // Presentation - Audio - Music - Settings
            builder.RegisterInstance<IMusicSettings>(settings.Presentation.MusicSettings);
            ...
        }
    }
}
```

:::

## 永続シーンのロード

`RootLifetimeScope` の `Start()` で読み込みます。

- `LoadSceneMode.Additive` でロードします。
- `RootLifetimeScope` の子シーンとしてロードします。
  - `RootLifetimeScope`が破棄されたタイミングで一緒に破棄するため。

::: details コード

```csharp
namespace WordPuzzle.LifetimeScopes
{
    public class RootLifetimeScope : LifetimeScope
    {
        ...

        private async void Start()
        {
            // PersistentSceneがまだロードされていなければロード
            const string persistentSceneName = "PersistentScene";
            if (!SceneManager.GetSceneByName(persistentSceneName).isLoaded)
            {
                // RootScopeの子としてロード（重要）
                using (LifetimeScope.EnqueueParent(LifetimeScope.Find<RootLifetimeScope>()))
                {
                    await SceneManager.LoadSceneAsync(persistentSceneName, LoadSceneMode.Additive);
                }
            }
        }

        ...
    }
}
```

:::

## 所感

この永続シーンを作成したことで、今までごちゃごちゃしていた `DontDestroyOnLoad` がクリーンになり、永続コンポーネントを管理しやすくなりました。
