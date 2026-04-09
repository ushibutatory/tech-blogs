---
title: "「星座づくり」(0) 概要"
---

## なにこれ

ランダムに配置される星をつないで、星座を描くゲームです。
描いた星座の形をもとに、AI に星座名と物語を考えてもらいます。

AI はコスト重視で`Groq API`を採用しました。

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
