---
title: "はじめに"
---

趣味で作った Unity ゲーム「ことばパズル」の、設計・実装まわりの記録ノートです。

:::message
完璧さなどはなく、自分の振り返りと反省、次への改善のための記事です。
そのため、内容にはツッコミどころが多々あると思います。
:::

なお、この本は下記シリーズの一部です。シリーズの趣旨なども下記リンク先に記載しています。

- [作ったゲームと設計パターンの試行錯誤まとめ](https://zenn.dev/ushibutatory/articles/442ac51a81bc00)

よろしくお願いいたします。

## どんなゲーム

カードを選んで、意味が通る文章を作るゲームです。

※開発中の画面

![イメージ](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/thumbnail.png)

「名詞 + 助詞 + 動詞」「名詞 + 助詞 + 形容詞」といった係り受けの練習のために作りました。

## 使用ライブラリ、環境など

- [UniTask](https://github.com/Cysharp/UniTask)
  - 非同期ライブラリ
- [VContainer](https://github.com/hadashiA/VContainer)
  - DIコンテナライブラリ
- [R3](https://github.com/Cysharp/R3) / R3.Unity
  - Rxライブラリ
- [MessagePipe](https://github.com/Cysharp/MessagePipe) / MessagePipe.VContainer
  - Pub/Subライブラリ

（設計に大きく関わらないアセット等は割愛）

## 所感

設計自体はそれなりに構築できた気がしますが、如何せん、日本語という題材が難しすぎました。
国語のルールをシステムに落とし込みきれず、判定の不自然さを解決できなかったためお蔵入りとしました。
