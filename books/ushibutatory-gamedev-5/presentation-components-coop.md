---
title: "プレゼンテーション層のコンポーネント間のやりとり"
---

プレゼンテーション層のコンポーネント（ボタンやカード）が別のコンポーネントを操作したり描画を変更したりしたい場合は、以下のような方法になります。

### ユースケースを実行しない場合

最もシンプルな操作です。

```mermaid
sequenceDiagram

actor User
box Presentation
    participant A as Component A
    participant B as Component B
    participant C as Component C
end

User->>A: 操作
A->>A: 自身の描画を更新する
A->>B: 別コンポーネントの描画を更新する
A->>C: 別コンポーネントの描画を更新する
```

プレゼンテーションイベントを使ってリクエストする場合は以下のようになります。

```mermaid
sequenceDiagram

actor User
box Presentation
    participant A as Component A
    participant B as SoundPlayer
    participant C as EffectPlayer
    participant Event as PresentationEventBus
end

User->>A: 操作
A->>A: 描画を更新する
A->>Event: 効果音再生をリクエストする
Event-->>B: 購読
B->>B: 効果音を再生する
A->>Event: エフェクト再生をリクエストする
Event-->>C: 購読
C->>C: エフェクトを再生する
```

### ユースケースを実行する場合

いくつかのパターンで実装できます。

自身が管理する別コンポーネントを直接操作する場合は以下のようになります。

```mermaid
sequenceDiagram

actor User
box Presentation
    participant A as Component A
    participant B as Component B
    participant C as Component C
end
box Application
    participant UseCase
end

User->>A: 操作
A->>UseCase: ユースケースを実行する
UseCase->>A: 実行結果を返す
A->>A: 自身の描画を更新する
A->>B: 別コンポーネントの描画を更新する
A->>C: 別コンポーネントの描画を更新する
```

他コンポーネントへのリクエストを送る場合は以下のようになります。

```mermaid
sequenceDiagram

actor User
box Presentation
    participant A as Component A
    participant B as SoundPlayer
    participant C as EffectPlayer
    participant Event as PresentationEventBus
end
box Application
    participant UseCase
end

User->>A: 操作
A->>UseCase: ユースケースを実行する
UseCase->>A: 実行結果を返す
A->>A: 描画を更新する
A->>Event: 効果音再生をリクエストする
Event-->>B: 購読
B->>B: 効果音を再生する
A->>Event: エフェクト再生をリクエストする
Event-->>C: 購読
C->>C: エフェクトを再生する
```

各コンポーネントがセッション状態の更新を購読する場合は以下のようになります。

```mermaid
sequenceDiagram

actor User
box Presentation
    participant A as Component A
    participant B as Component B
    participant C as Component C
end
box Application
    participant UseCase
    participant Session as Session
    participant Event as SessionEventBus
end

User->>A: 操作
A->>UseCase: ユースケースを実行する
UseCase->>Session: セッション状態を更新する
Session->>Event: 更新を通知する
Event-->>A: 購読
A->>A: 自身の描画を更新する
Event-->>B: 購読
B->>B: 自身の描画を更新する
Event-->>C: 購読
C->>C: 自身の描画を更新する
```

例）

- ユースケース「手札を提出する」を実行した後、手札コンポーネント内にカードコンポーネントを移動させる、スコアUIを更新する
