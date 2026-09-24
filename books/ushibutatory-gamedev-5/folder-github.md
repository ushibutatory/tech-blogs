---
title: "フォルダ構成 - GitHubリポジトリ"
---

以下のようになりました。

- 見やすさのためにアルファベット順ではなく敢えて入れ替えたりしています。

## フォルダ構成（GitHubリポジトリ）

```text
.gitignore                         # 全体向けのgitignore（Windows等）
.github/
    workflows/                    # GitHub Actions
docs/
    .gitignore                    # ドキュメント向けのgitignore（VSCode等）
    ...                           # 設計資料（markdown, drawio等）
src/
    WordPuzzle/                   # Unityプロジェクト
        .gitignore                # Unity向けのgitignore（Unity, VisualStudio, VSCode等）
        ...
    Tools/                        # 自作した周辺ツール群
        notify-to-discord/
            .gitignore            # ツールを作成した言語ごとに記述（node.js等）
            ...
arts/
    Affinity/
        .gitignore                # Affinity向けのgitignore
        ...                       # 作業ファイル及び成果物
builds/
    .gitignore
    Android/
        ...                       # 実際の成果物はignoreしておく
```

各フォルダのルートには `README.md` を置いておき、フォルダの説明や命名ルール、保管に関する注意事項（.gitignoreの除外ルールの説明）などを記載しておきます。

## 所感

`.gitignore` を階層ごとに定義する方法が気に入っています。
