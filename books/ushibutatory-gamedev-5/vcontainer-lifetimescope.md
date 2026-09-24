---
title: "VContainer - LifetimeScope"
---

VContainerのLifetimeScopeをどういう風に設定したかという話。

## 定義する場所

レイヤーライブラリとは別の、ゲーム全体の設定を構成するためのライブラリを定義し、そこで _LifetimeScope_ やインジェクションの定義をしました。

![asmdefの場所](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-asmdef.png)
![asmdefの詳細](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-asmdef-details.png)

## RootLifetimeScope

全体の構成を記述します。

- Logging
- MessagePipeの基本設定
- 設定（ScriptableObject）のインジェクション

::: details コード

```csharp
using MessagePipe;
using Microsoft.Extensions.Logging;
using System.Threading;
using UnityEngine;
using UnityEngine.SceneManagement;
using VContainer;
using VContainer.Unity;
using WordPuzzle.Diagnostics;
using WordPuzzle.Infrastructure.Settings;
using ZLogger;
using ZLogger.Unity;

namespace WordPuzzle.LifetimeScopes
{
    public class RootLifetimeScope : LifetimeScope
    {
        [Header("Settings")]
        [SerializeField] private ProductSettings _productSettings = default!;
        [SerializeField] private DebugSettings _debugSettings = default!;

        private readonly CancellationTokenSource _cancellationTokenSource = new();

        protected override void Awake()
        {
            base.Awake();

            var logger = Container.Resolve<ILogger<RootLifetimeScope>>();
            logger.LogInformation("__________/_________/________/_______/______/_____/____/___/__/_/");
            logger.LogInformation(nameof(Awake));
        }

        private async void Start()
        {
            // PersistentSceneがまだロードされていなければロード
            if (!SceneManager.GetSceneByName("PersistentScene").isLoaded)
            {
                // RootScopeの子としてロード（重要）
                using (LifetimeScope.EnqueueParent(LifetimeScope.Find<RootLifetimeScope>()))
                {
                    await SceneManager.LoadSceneAsync("PersistentScene", LoadSceneMode.Additive);
                }
            }
        }

        protected override void OnDestroy()
        {
            var logger = Container.Resolve<ILogger<RootLifetimeScope>>();
            logger.LogInformation(nameof(OnDestroy));

            _cancellationTokenSource?.Cancel();
            _cancellationTokenSource?.Dispose();

            base.OnDestroy();
        }

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

        private void _Configure_Diagnostics(IContainerBuilder builder)
        {
            // Global exception monitor
            builder.RegisterEntryPoint<GlobalExceptionMonitor>(Lifetime.Singleton);
        }

        private void _Configure_Logger(IContainerBuilder builder)
        {
            builder.RegisterInstance(LoggerFactory.Create(logging =>
            {
#if DEBUG
                // ログレベル
                logging.SetMinimumLevel(LogLevel.Debug);

                // コンソール出力
                logging.AddZLoggerUnityDebug(options => { options.IncludeScopes = true; });

                // ファイル出力
                var date = System.DateTime.UtcNow;
                var filePath = System.IO.Path.Combine(
                    UnityEngine.Application.persistentDataPath,
                    $"logs/log_{date:yyyy-MM-dd}.log");

                logging.AddZLoggerFile(filePath, options =>
                {
                    options.UsePlainTextFormatter(formatter =>
                    {
                        formatter.SetPrefixFormatter(
                            $"{0:yyyy-MM-dd HH:mm:ss.fff zzz} | {1:short} | {2} | ",
                            (in MessageTemplate template, in LogInfo info) =>
                            {
                                template.Format(info.Timestamp, info.LogLevel, info.Category);
                            });
                    });
                });

#else
                // ログレベル
                logging.SetMinimumLevel(LogLevel.Warning);

                // コンソール出力
                logging.AddZLoggerUnityDebug(options => { options.IncludeScopes = false; });
#endif
            }));
            builder.Register(typeof(ILogger<>), typeof(Logger<>), Lifetime.Singleton);
        }

        private void _Configure_Settings(IContainerBuilder builder)
        {
            builder.RegisterInstance<IProductSettings>(_productSettings);
            builder.RegisterInstance<DebugSettings>(_debugSettings);
        }
    }
}
```

:::

Prefabを作成し、上記の _RootLifetimeScope_ をアタッチします。

![RootLifetimeScopeのPrefab](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-rootlifetimescope-prefab.png)

![RootLifetimeScopeのPrefabのInspector](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-rootlifetimescope-prefab-details.png)

上記の Prefab を、`VContainterSettings`の`RootLifetimeScope`に指定して完了です。

## シーンごとのLifetimeScope

シーンごとに定義します。

![シーンごとのLifetimeScope](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-scenelifetimescope.png)

::: details コード

```csharp
namespace WordPuzzle.LifetimeScopes.Scenes
{
    public class InGameSceneLifetimeScope : SceneLifetimeScope
    {
        protected override void _Configure(IContainerBuilder builder, IProductSettings settings, MessagePipeOptions messagePipeOptions)
        {
            // ------ Domain ------
            // Domain
            builder.Register<IRandomFactory, ProductRandomFactory>(Lifetime.Scoped);

            // Domain - InGameComponents - Repositories
            builder.RegisterInstance(settings.Domain.FieldConfigurationRepository);
            builder.RegisterInstance(settings.Domain.NounCardRepository);
            ...

            // Domain - InGameMechanics - Grammar
            builder.Register<IConnectionStructureValidator, ConnectionStructureValidator>(Lifetime.Scoped);
            builder.Register<IConnectionStructureValidateRule, NounStructureRule>(Lifetime.Scoped);
            builder.Register<IConnectionStructureValidateRule, VerbStructureRule>(Lifetime.Scoped);
            ...

            builder.Register<ISentenceStructureValidator, SentenceStructureValidator>(Lifetime.Scoped);

            builder.Register<ISentenceSemanticValidator, SentenceSemanticValidator>(Lifetime.Scoped);
            builder.Register<ISentenceSemanticValidateRule, SubjectSemanticRule>(Lifetime.Scoped);
            ...

            // Domain - InGameMechanics - Grammar - Repositories
            builder.RegisterInstance(settings.Domain.ParticleUsageRepository);
            builder.RegisterInstance(settings.Domain.CanTagRepository);
            ...

            // Domain - InGameMechanics - Score
            builder.Register<IScoreCalculator, ScoreCalculator>(Lifetime.Scoped);
            builder.Register<IScoreRule, BasicScoreRule>(Lifetime.Scoped);
            ...

            // Domain - InGameMechanics - Score - Repositories
            builder.RegisterInstance(settings.Domain.ScoreSettingsRepository.Get<IBasicScoreSettings>(ScoreType.Basic));
            builder.RegisterInstance(settings.Domain.ScoreSettingsRepository.Get<IDifficultyBonusSettings>(ScoreType.Difficulty));

            // ------ Application ------
            // Application - Session
            builder.Register<IInGameSessionManager, InGameSessionManager>(Lifetime.Scoped);
            builder.Register<InGameSessionState>(Lifetime.Transient);
            builder.Register<StartupOptionState>(Lifetime.Transient);
            builder.Register<FieldState>(Lifetime.Transient);
            ...

            // Application - UseCases
            builder.Register<IStartInGame, StartInGame>(Lifetime.Scoped);
            builder.Register<StartInGame.ApplicationDependencies>(Lifetime.Transient);
            builder.Register<StartInGame.DomainDependencies>(Lifetime.Transient);

            builder.Register<ISupplyCard, SupplyCard>(Lifetime.Scoped);
            builder.Register<SupplyCard.ApplicationDependencies>(Lifetime.Transient);

            builder.Register<ITryAddCardToHand, TryAddCardToHand>(Lifetime.Scoped);
            builder.Register<TryAddCardToHand.ApplicationDependencies>(Lifetime.Transient);
            builder.Register<TryAddCardToHand.DomainDependencies>(Lifetime.Transient);
            ...

            // Application - Events
            builder.RegisterMessageBroker<InGameStarted>(messagePipeOptions);
            builder.RegisterMessageBroker<FieldUpdated>(messagePipeOptions);
            ...

            // ------ Presentation ------
            // Presentation - Scene
            builder.RegisterEntryPoint<InGameSceneDirector>(Lifetime.Scoped);
            builder.Register<InGameSceneDirector.PresentationDependencies>(Lifetime.Transient);
            builder.Register<InGameSceneDirector.ApplicationDependencies>(Lifetime.Transient);

            // Presentation - Camera
            builder.RegisterComponentInHierarchy<InGameCameraComponent>();
            builder.RegisterComponentInHierarchy<InGameCameraMovable>();

            // Presentation - Input
            builder.Register<InGameInputHandler>(Lifetime.Scoped);

            // Presentation - Components
            builder.RegisterEntryPoint<InputListener>(Lifetime.Scoped);

            // Presentation - Components - InHierarchy
            builder.RegisterComponentInHierarchy<CardSpawner>();
            builder.RegisterComponentInHierarchy<CardPool>();

            builder.RegisterComponentInHierarchy<FieldBoardComponent>();
            builder.RegisterComponentInHierarchy<FieldBoardLayout>();
            builder.RegisterComponentInHierarchy<FieldSlotContainer>();

            builder.RegisterComponentInHierarchy<PlayerBoardComponent>();
            builder.RegisterComponentInHierarchy<PlayerBoardLayout>();
            ...

            // Presentation - Components - OnNewGameObject
            builder.RegisterEntryPoint<CardTappedListener>(Lifetime.Scoped);
            builder.Register<CardTappedListener.ApplicationDependencies>(Lifetime.Transient);
            builder.Register<CardTappedListener.PresentationDependencies>(Lifetime.Transient);

            // Presentation - Events
            builder.RegisterMessageBroker<BackgroundTapped>(messagePipeOptions);
            ...

            // Presentation - Listeners
            builder.RegisterEntryPoint<InGameStartedListener>(Lifetime.Scoped);
            builder.Register<InGameStartedListener.ApplicationDependencies>(Lifetime.Transient);
            ...

            // Presentation - UI
            builder.RegisterComponentInHierarchy<InGameUINavigator>();

            builder.Register<InGameUIView>(Lifetime.Scoped);
            builder.Register<InGameMenuSection>(Lifetime.Scoped);
            ...

            // Presentation - Settings
            builder.RegisterInstance(settings.Presentation.BoardLayoutSettings);
            builder.RegisterInstance(settings.Presentation.CameraSettings);
            ...
        }
    }
}
```

:::

これも _Prefab_ を作成し、上記スクリプトをアタッチします。

![シーンごとのLifetimeScopeのPrefab](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-scenelifetimescope-prefab-details.png)

作成した _Prefab_ を、シーンのヒエラルキーに配置して完了です。

![シーンごとのLifetimeScopeをHierarchyに配置](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/vcontainer-scenelifetimescope-hierarchy.png)

## 所感

やっと少しだけ扱いに慣れてきました。

まだ `Awake()` や `Start()` を十分に使いこなせていない感じがするので、引き続き勉強していきたいです。
