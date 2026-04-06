---
title: "「タワーディフェンス」(2) アーキテクチャ"
---

## 全体像

![アーキテクチャ全体]()

## 詳細

### Domain 層

#### Entity

当プロジェクトでは、「構成」と「状態」を持つモデル、としました。

- 「構成」は変更不可のフィールドです。
  - 例）ID、ローカライズ用の名前キー（或いは名前そのもの）、など
- 「状態」は変更可能なフィールドです。
  - 例）最大HP、現在HPなど
  - 「最大HPは変更されない」などは仕様に依ります。

具体的な実装は `init` or `private set` で表現しています。

```cs
public class Enemy
{
    // 構成は生成時にセットされた後、変更不可
    public required EnemyId Id { get; init; }
    public required Guid InstanceId { get; init; }
    public required NameKey NameKey { get; init; }

    // 状態は変更可能（ただし変更のためのメソッドを用意して整合性を担保する）
    public required HealthPoint MaxHp { get; private set; }
    public required HealthPoint CurrentHp { get; private set; }

    // 例）ダメージ量は別途 DomainFunction で計算する。
    public void TakeDamage(HealthPoint damage)
    {
        // 負数にならないように調整する、など
    }
}
```

#### ValueObject

Core層で定義しました。

#### DomainFunction

「定義」や「状態」を持たない、単発の処理です。
また、これ自体は「設定」を持たず、パラメータで挙動を指示します。

- クリーンアーキテクチャでは `DomainService` と呼ばれていますが、当プロジェクトでは `Service` ではなく `Function` と呼ぶことにしました。
  - 「サービス」だと副作用や状態を含みそうなイメージがあるため。
  - 「関数」という名前にしておけば、「Input に対して処理を行い Output を返すだけ」というイメージがしやすいと考えています。
  - あくまで、私個人の好みです。

#### Factory（インタフェース）

Entityの生成インタフェースです。
実装は Infra 層で定義します（設定を参照するため）。

```cs
public interface EnemyFactory
{
    Enemy Create(EnemyId id);
}
```

#### Repository（インタフェース）

Entityを永続化するインタフェースです。
実装は Infra 層で定義します（外部I/O実装のため）。

```cs
public interface PlayerRepository
{
    // 取得できなかった場合は null を返す（フォールバックは呼び出し元で実装する）
    // （カスタム例外をthrowする方針でもよさそう）
    UniTask<Player?> GetAsync();
    UniTask SaveAsync(Player player);
}
```

---

### Application 層

#### Session

特定の場面のライフサイクルと連動し、状態を保持します。ここでいう「場面」とは、タイトルやショップ、バトルなど、まとまった機能を指します（「シーン」ではありません）。
この場面ごとの状態を当プロジェクトでは「Session（セッション）」と呼ぶことにしました。

- 例）`ShopSession`
  - 店舗の在庫状態、現在の所持金などを保持します。
    - 対象場面で必要なデータをまとめて管理するだけであり、他場面で所持金が必要になったからといってそちらからこの `ShopSession` を参照することはしません。
  - ショップに入ったタイミングで生成、初期化します。
  - ショップを抜けたタイミングで破棄します。
  - 必要なタイミングで永続データの参照や更新を行います。

ショップ機能がシーン遷移なのかUI切り替えなのか、という意識はアプリケーション層では持ちません。
すべての更新はユースケースから操作されます（プレゼンテーション層から操作はできない）。

#### SessionState

セッションが保持する具体的なデータ（状態）です。
エンティティのインスタンスなどを保持します。またそれらの更新をR3で公開します。
プレゼンテーション層はこれらを購読し、画面に反映します。

操作はユースケースから行います。

#### SessionManager

セッションを生成・破棄します。

- 各ユースケースは、この SessionManager から現在の Session を取得します。
- プレゼンテーション層は、この SessionManager から現在の SessionState を取得します。

#### UseCase

ユースケースを定義します。
1クラス1ユースケースで定義していきます。
例えば「アイテム一覧を取得する」「アイテムを購入する」「アイテムを装備する」など。

- ドメイン操作を組み合わせて一連の業務ロジックを表現します。
  - 検証 → ドメイン操作 → 結果の通知 をまとめます。
  - ドメイン操作した結果（更新後のEntityインスタンス等）は、SceneSession経由でSceneSessionStateに保存します。
- 1つのユースケース内でデータの整合性を担保するため、「トランザクション」と表現してもいいかもしれません。
- ユースケース自体は状態を持ちません。必要に応じて前述のセッションやリポジトリを参照・更新します。

処理の一元性を保つためにプレゼンテーション層から実行するドメイン操作は常にユースケースを経由するようにし、それ以外の方法でデータの参照や更新は行えないようにしています。

#### ApplicationEvent

アプリケーション層から通知するイベント。
購読はプレゼンテーション層でのみ行います。

```cs
// セッション
public class SampleSession
{
    [Inject] private readonly IPublisher<DataUpdated> _dataUpdated = default!;

    public void Update()
    {
        // 何らかの状態を更新する

        // 変更を通知する
        _dataUpdated.Publish(new DataUpdated
        {
            ...
        });
    }
}

// ユースケース
public class UpdateSampleData
{
    public void Execute()
    {
        var session = _sessionManager.Current;
        session.Update();

        // ユースケースは更新の通知を行わない。
        // 処理結果を戻り値で返すことは可能。
    }
}
```

当プロジェクトでは、セッション内からのみ通知するものとしました。
- セッション状態は複数のユースケースから操作されるため（通知漏れを防ぐため）。

--- 

### Presentation 層

#### Component

MonoBehaviour を継承したコンポーネント。

#### UIView

UI Toolkitの表示を行うクラスです。
MonoBehaviour は継承しません。UIToolkit関連については後述です。

#### PresentationEvent

コンポーネント間の通信用イベントです。

- コンポーネントが他のMonoBehaviourコンポーネントに出す指示を定義します。
- コンポーネントの変化を通知することはしません（Application層の更新を購読します）。

すべて `～Request` というサフィックスをつけます。

- `PlayAudioRequest`, `PauseAudioRequest`などのオーディオ操作
- `PlaySoundRequest`などの効果音操作
- `LoadSceneRequest`などのシーン操作

---

### Core 層

#### ValueObject

「値」のみを持つドメインモデルです。
Entityと異なり「状態」を持ちません。イミュータブルです。

基本的に単純な構造体（struct）や列挙型として実装することがほとんどです。
定数や型変換、各種演算子もあわせて定義します。

```cs
public readonly record struct HealthPoint
{
    private int Value { get; init;}

    private HealthPoint(int value)
    {
        Value = value;
    }
    public static HealthPoint Of(int value) => new(value);

    // 定数
    public static HealthPoint Zero => Of(0);

    // 型変換
    public static implicit operator int(HealthPoint point) => point.Value;
    public static implicit operator HealthPoint(int value) => new(value);

    // 各種演算（四則演算、比較演算など）
    public static HealthPoint operator +(HealthPoint a, HealthPoint b)
    ...
}
```

- ドメイン層に定義するとプレゼンテーション層で使えません（対応するDTOを定義して値を入れ替えて渡す）。ただ、ValueObject自体がシンプルな型であり、かつDTOとして全く同じ構造を持つクラスを定義することが煩わしいと感じたため、全レイヤーから参照できるよう Core 層に配置しました。

---

### Infrastructure 層

#### Factory（実装）

Entityの生成処理です。

- 生成時のパラメータは引数、または ScriptableObject で指定します。

#### Repository（実装）

Entityを永続化します。

- キャッシュを持たず、常に外部ファイルを参照・更新します。
- アクセス頻度が高い場合はキャッシュ機能を設けたくなりますが、リポジトリ内部ではなくリポジトリを使用する側（Sessionなど）で制御するようにしました。
  - どのデータにアクセスするのか、を明確にするため。

#### AssetList

アセットを取得するためのクラスです。例えばBGM群の場合は「AudioAssetList」などとします。

- アセットの管理方法（Addressables、ScriptableObject等）を隠蔽します。
- キャッシュ数や期間などは ScriptableObject で設定化します。
  - Repositoryと異なり、こちらは AssetList 内でキャッシュ制御します（使用場面ごとのチューニングは不要と判断）。

#### Setting

ScriptableObject群です。（→[別ページ](./4-rhythm-5)）
定義した Setting はDIコンテナで各クラスに Inject します。

## 所感

「状態をどこが持つのか」という設計にいつも悩みます（今回はセッションという単位で解決しました）。
