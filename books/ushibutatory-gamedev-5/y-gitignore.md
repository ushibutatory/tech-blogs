---
title: "余談 - .gitignoreの追記"
---

:::message
アーキテクチャの話ではなく、開発時のやらかしの話です。
:::

gitignore.io などで生成すると、嬉しくない除外ルールが含まれることがあります。

以下のような対応を手動で行いました。

> gitignoreは上から下に判定される（後ろに書かれたルールが優先される）ので、追記位置に注意。

## asmdef.metaをgit管理対象とする

### Before

```.gitignore
.meta
```

metaファイルがgit管理されず、環境ごとにmetaファイルが生成されます。

そのため開発環境ごとにGUIDが変わってしまい、Missing Script や asmdef の参照切れが多発します。

一人で開発しているので気づきませんでしたが、PCを買い替えたことにより発覚しました。
（チーム開発なら常識なのでしょうね……）

### After

```.gitignore
# あればコメントアウト、或いは削除
#.meta

# 以下を追記
!/[Aa]ssets/**/*.meta
```

Missing Script をひとつひとつ探して張り直していく作業は地獄でした。

## csc.rspをgit管理対象とする

### Before

```.gitignore
*.rsp
```

C#バージョンを指定するために `csc.rsp` を記述していましたが、git管理から漏れていました。

### After

```.gitignore
!csc.rsp
```

## 所感

特に `.meta` を欠落させた件が本当に最悪でした。
