---
title: "はじめに"
---

趣味で作った Unity ゲーム「3D迷路」の、設計・実装まわりの記録ノートです。

なお、この本は下記シリーズの一部です。シリーズの趣旨なども下記リンク先に記載しています。

- [作ったゲームと設計パターンの試行錯誤まとめ](https://zenn.dev/ushibutatory/articles/442ac51a81bc00)

よろしくお願いいたします。

## どんなゲーム

3D の迷路です。

![イメージ](https://raw.githubusercontent.com/ushibutatory/u1w-dungeon/refs/heads/main/docs/thumbnail.png?token=GHSAT0AAAAAAEIZUP5OHDLXU2GBYZGSW3IC2VKFTUA)

- 開発した時期
  - 2020 年頃
- 実行環境
  - unityroom
    - <https://unityroom.com/games/ushibutatory-maze>

## 使用ライブラリなど

※アーキテクチャに関するもののみ記載

- UniTask
  - 非同期処理
- UniRx
  - Rx ライブラリ
- Extenject（Zenject）
  - DI コンテナ
- YamlDotNet for Unity
  - ダンジョンのマップデータを yaml で管理していたので、その読み込み用。
