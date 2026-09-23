---
title: "Presentation - 永続コンポーネント置き場"
---

コンポーネントについて、シーンを横断して永続させたい時があります。

これまで、そういうものはよく `DontDestroyOnLoad` に配置していたのですが、今回、明示的に専用シーンを作成して管理するようにしてみました。

## 永続コンポーネント用のシーン「PersistentScene」

![Hierarchy](https://raw.githubusercontent.com/ushibutatory/tech-images/refs/heads/main/books/ushibutatory-gamedev-5/persistent-hierarchy.png)

全シーン共通の

## 所感

この永続専用シーンを作成したことで、 `DontDestroyOnLoad` がとてもクリーンになりました。
