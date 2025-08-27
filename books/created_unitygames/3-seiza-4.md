---
title: "「星座づくり」(4) privateメンバ多すぎ問題への対処"
---

MonoBehaviour を継承したコンポーネントは、private メンバが多くなりがちです。

## 元のコード

```csharp
namespace MyGame.Presentation
{
    public class SampleComponent : MonoBehaviour
    {
        // Inspectorで設定する項目
        [Header("...")]
        [SerializeField] private TMP_Text _text;
        [SerializeField] private Image _backgroundImage;
        [SerializeField] private Button _button;

        [Header("...")]
        [SerializeField, Range(0f, 1f)] private float _someValue = 0.5f;

        // Application層のユースケース
        [Inject] private readonly ISampleUpdate _sampleUpdate = default!;

        // Application層からのイベント購読
        [Inject] private readonly ISubscriber<SampleUpdateCompleted> _sampleUpdateCompleted = default!;
        [Inject] private readonly ISubscriber<SampleUpdateFailed> _sampleUpdateFailed = default!;

        // Presentation層のイベント
        [Inject] private readonly IPublisher<SomeRequest> _someRequestPublisher = default!;

        // ローカル管理する状態
        private int _currentValue;

        ...

        private void Awake()
        {
            // イベントの購読
            _sampleUpdateCompleted?.Subscribe(e =>
            {
                ...
            }).AddTo(this);
            _sampleUpdateFailed?.Subscribe(e =>
            {
                ...
            }).AddTo(this);

            // UIイベント
            _button.OnClickAsObservable()
                .Subscribe(...)
                .AddTo(this);

            // 状態の初期化
            _currentValue = ...;
        }

        private void Method()
        {
            // Inspector項目へのアクセス
            _text.text = "...";

            // アプリケーション層へのアクセス
            _sampleUpdate?.Execute(...);
            // 或いは await _sampleUpdate.ExecuteAsync(...);

            // プレゼンテーションイベントの発行
            _someRequestPublisher?.Publish(...);

            // 状態の変更
            _currentValue = ...;
        }
    }
}
```

## よくある対応

一番よく見かける対応は、装飾された目立つコメント行を間に挟むことだと思います。

```csharp
        // ======Inspectorで設定する項目======

        [Header("...")]
        [SerializeField] private TMP_Text _text;
        [SerializeField] private Image _backgroundImage;
        [SerializeField] private Button _button;

        [Header("...")]
        [SerializeField, Range(0f, 1f)] private float _someValue = 0.5f;

        // ======ユースケース======

        [Inject] private readonly ISampleUpdate _sampleUpdate = default!;
        ...
```

或いは、（最近使っている人をあまり見かけませんが）`#region` もありかと思います。

```csharp
#region Inspectorで設定する項目
        [Header("...")]
        [SerializeField] private TMP_Text _text;
        [SerializeField] private Image _backgroundImage;
        [SerializeField] private Button _button;

        [Header("...")]
        [SerializeField, Range(0f, 1f)] private float _someValue = 0.5f;
#endregion

#region Application層のユースケース
        [Inject] private readonly ISampleUpdate _sampleUpdate = default!;
#endregion

...
```

## 今回試したこと

私は、コメントではなく構造で解決できないかと考えて、以下のような構成を試してみました。

```csharp
namespace MyGame.Presentation
{
    public class SampleComponent : MonoBehaviour
    {
        // Inspectorで設定する項目
        [Serializable] // ←Inspector対応のため
        public class Assignments
        {
            [Header("...")]
            [SerializeField] public TMP_Text Text;
            [SerializeField] public Image BackgroundImage;
            [SerializeField] public Button Button;

            [Header("...")]
            [SerializeField, Range(0f, 1f)] public float SomeValue = 0.5f;
        }
        [SerializeField] private Assignments _assignments = new();

        // Application層の依存関係
        public class ApplicationDependencies
        {
            // ユースケース
            [Inject] public readonly ISampleUpdate SampleUpdate = default!;

            // アプリケーションイベント
            [Inject] public readonly ISubscriber<SampleUpdateCompleted> SampleUpdateCompleted = default!;
            [Inject] public readonly ISubscriber<SampleUpdateFailed> SampleUpdateFailed = default!;
        }
        [Inject] private readonly ApplicationDependencies _application = default!;

        // Presentation層の依存関係
        public class PresentationDependencies
        {
            [Inject] public readonly IPublisher<SomeRequest> SomeRequestPublisher = default!;
        }
        [Inject] private readonly PresentationDependencies _presentation = default!;

        // ローカル管理する状態
        private class Status
        {
            public int CurrentValue;
        }
        private Status _status;

        ...

        private void Awake()
        {
            // イベントの購読
            _application?.SampleUpdateCompleted?.Subscribe(e =>
            {
                ...
            }).AddTo(this);
            _application?.SampleUpdateFailed?.Subscribe(e =>
            {
                ...
            }).AddTo(this);

            // UIイベント
            _assignments.Button.OnClickAsObservable()
                .Subscribe(...)
                .AddTo(this);

            // 状態の初期化
            _status = new Status
            {
                CurrentValue = ...;
            };
        }

        private void Method()
        {
            // Inspector項目へのアクセス
            _assignments.Text.text = "...";

            // アプリケーション層へのアクセス
            _application?.SampleUpdate?.Execute(...);
            // 或いは await _application.SampleUpdate.ExecuteAsync(...);

            // プレゼンテーションイベントの発行
            _presentation?.SomeRequestPublisher?.Publish(...);

            // 状態の変更
            _status.CurrentValue = ...;
        }
    }
}
```

構造化することにより必然的にプレフィックス付与と同等の記述となるため、何をしているのか分かりやすくなったと思います。

## 所感

言うまでもなく複雑さは増しますし、何でもかんでもこれでやっていくのは過剰だと思います。
private メンバが多くてややこしいと感じたら、まずはコンポーネント分割を検討したほうがよい（責務過多を疑ったほうがよい）と思います。

## 余談

本文では割愛していますが、実際は Awake 開始時に各フィールドが Injection されているかどうかを検証しています。
