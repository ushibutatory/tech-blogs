---
title: "「タワーディフェンス」(6) UI Toolkit関連の処理"
---

## 構成

### UIViewSection

表示・操作するひとまとまりです。

- 複数の `VisualElement` を制御し、UIイベントの購読や描画更新などを行う。
- `MonoBehaviour` を継承しない。

### UIView

いわゆる「画面」です。

- 複数の `UIViewSection` をまとめ、ひとつの画面をデザインする。
- `MonoBehaviour` を継承しない。

### UINavigator

UIViewの状態を制御します。

- 複数の `UIView` をまとめて、表示・非表示を制御します。
- `MonoBehaviour` を継承する。

### SceneContext

シーン内で起きたイベントに応じてUINavigatorを制御します。

- 例）
  - ショップの前にプレイヤーが到着した → 遷移確認UIViewを表示する。
  - OKが押された → 遷移確認UIViewを非表示にし、ショップメニューUIViewを表示する。

`SceneContext` という名前は、そのシーンの最初から最後までを一貫して監視する理由から命名しました。
（UIView表示切替の他、他シーンへの遷移なども行います）

### Unity

```
```

### C#

```csharp
```

## 所感

初めて触ったので非常に苦戦しました。
特に MonoBehaviourでないものを画面上で動作させる、という感覚が掴めませんでした（今も怪しい）。
