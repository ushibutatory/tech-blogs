---
title: "データ構造"
---

## ダンジョンデータを YAML で管理する

1 マス分のデータを以下のように定義しました。

```yaml
Grids:
  - Grid:
      X: (int)
      Y: (int)
    Wall:
      North: (boolean)
      East: (boolean)
      West: (boolean)
      South: (boolean)
    Door:
      North: (boolean)
      East: (boolean)
      West: (boolean)
      South: (boolean)
    Message:
      North: (string)
      East: (string)
      West: (string)
      South: (string)
  - Grid: ...
```

「1 行につき 1 項目」を担保しようとして YAML を採用したら、こうなりました。
もう想像つくと思いますが、10\*10 のマス目を表現しようとすると 1,800 行になります。

本腰を入れて開発するなら、このマップデータをビジュアル的に編集し YAML ファイルを出力する補助ツールと作るとよいと思います。
今回はマップが 1 種類固定だったので、「手書きのコスト」と「ツール開発のコスト」を天秤にかけて手書きを選択しました。

## 所感

やはり手書きは面倒くさくて辛かったです。
