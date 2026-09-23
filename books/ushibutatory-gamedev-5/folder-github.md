---
title: "フォルダ構成 - GitHubリポジトリ"
---

以下のようになりました。

- 見やすさのためにアルファベット順ではなく敢えて入れ替えたりしています。
- 記事用に`MyGame`という仮プロジェクト名に変換しています。

## フォルダ構成（GitHubリポジトリ）

```text
.gitignore                  # 全体向けのgitignore（Windows等）
.github/
  workflows/                # GitHub Actions
docs/
  .gitignore                # ドキュメント向けのgitignore（VSCode等）
  ...                       # 設計資料（markdown, drawio等）
src/
  WordPuzzle/
    .gitignore              # Unity向けのgitignore（Unity, VisualStudio, VSCode等）
    ...                     # Unityプロジェクト
  Tools/
    ...                     # 自作した周辺ツール群
arts/
  Affinity/
    .gitignore              # Affinity向けのgitignore
    ...                     # 作業ファイル及び成果物
```

## 所感

`.gitignore` を階層ごとに定義する方法が気に入っています。
