---
title: "はじめに"
---

趣味で作った Unity ゲーム「星座づくり」の、設計・実装まわりの記録ノートです。

:::message
完璧さなどはなく、自分の振り返りと反省、次への改善のための記事です。
そのため、内容にはツッコミどころが多々あると思います。
:::

なお、この本は下記シリーズの一部です。シリーズの趣旨なども下記リンク先に記載しています。

- [作ったゲームと設計パターンの試行錯誤まとめ](https://zenn.dev/ushibutatory/articles/442ac51a81bc00)

よろしくお願いいたします。

## どんなゲーム

![イメージ](https://raw.githubusercontent.com/ushibutatory/game-stella_generator-pages/refs/heads/main/images/screenshot2.png)

ランダムに配置される星をつないで、星座を描くゲームです。
描いた星座の形をもとに、AI に星座名と物語を考えてもらいます。

AI は `Groq API` を採用しました。

- 動作確認
  - unityroom
    - <https://unityroom.com/games/seiza-zukuri>
  - GitHub
    - <https://github.com/ushibutatory/game-stella_generator-pages>

## 使用ライブラリ、環境など

※アーキテクチャに関するもののみ記載

### Unity

- `UniTask`
  - 非同期処理のため
- `MessagePipe`
  - Pub/Sub パターン実装のため
  - 主に以下の用途で使用しています。
    - レイヤー間の通信、特に「メソッドの戻り値（＝実行結果）」に該当する処理。
    - プレゼンテーション層における、オブジェクト間の通信。
- `VContainer`
  - DI コンテナ
- `R3`
  - Rx ライブラリ
  - 主に UI コンポーネントのイベント（ボタンクリック等）のハンドリングに使用。

### API

- `AWS Lambda` + `API Gateway`
  - `Groq API` キー及びプロンプトを秘匿するため
- Node.js
- `Groq SDK`
  - `Groq API` を実行するため
  - なぜ Groq を選んだかというと、無料枠がそれなりにあったからです。
