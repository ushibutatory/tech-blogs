---
title: "「タワーディフェンス」(2) フォルダ構成 - GitHubリポジトリ"
---

最終的に以下のようになりました。

（見やすさのためにアルファベット順ではなく敢えて入れ替えていたりします）

## フォルダ構成（GitHubリポジトリ）

```text
.gitignore                  # 全体向けのgitignore（Windows等）
.github/
  copilot-instructions.md   # カスタムインストラクションのルートファイル
  instructions/             # カスタムインストラクションのルール群
  workflows/                # GitHub Actions
docs/
  .gitignore                # ドキュメント向けのgitignore（VSCode等）
  ...                       # 設計資料（markdown, drawio等）
src/
  MyGame/
    .gitignore              # Unity向けのgitignore（Unity, VisualStudio, VSCode等）
    ...                     # Unityプロジェクト
  Tools/
    ...                     # 自作した周辺ツール群
arts/
  Blender/
    .gitignore              # Blender向けのgitignore
    ...                     # 作業ファイル及び成果物
  StudioOne/
    .gitignore              # StudioOne向けのgitignore
    ...                     # 作業ファイル及び成果物
```

---

## 所感

`.gitignore` を階層ごとに定義する方法が気に入っています。
