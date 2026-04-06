---
description: "ファイル操作・エージェント作業時のルール"
applyTo: "**"
---

# エージェント操作ルール

## ファイル編集前の git 確認

ファイルの編集・リネーム・削除を実施する前に、必ず以下を確認すること。

```powershell
git log --oneline -5
git status
```

以下のいずれかに該当する場合は、作業前にユーザーへ「先にコミットしておいたほうがよいのでは？」と確認を促すこと。

- 直近コミットから 1 日以上経過している
- 未コミットの変更（`git status` に表示）が多い
- git 管理外（untracked）のファイルを複数編集する

## PowerShell でのファイル操作

Windows の PowerShell は日本語環境においてデフォルトのエンコーディングが Shift-JIS になるため、日本語を含むファイルを扱う際は必ず以下を遵守すること。

- ファイルを読み書きする前に、先頭バイトを確認してエンコーディングを特定すること
  - `EF BB BF` → BOM 付き UTF-8
  - BOM なし → UTF-8 または Shift-JIS（内容で判断）
- `Get-Content` / `Set-Content` には必ず `-Encoding UTF8` を明示すること
- より確実な方法として `[System.IO.File]::ReadAllText` / `WriteAllText` に `[System.Text.Encoding]::UTF8` を明示して使用すること

```powershell
# 推奨パターン
$utf8 = [System.Text.Encoding]::UTF8
$content = [System.IO.File]::ReadAllText($path, $utf8)
[System.IO.File]::WriteAllText($path, $content, $utf8)
```

エンコーディングを誤ると多重変換で復元不能な文字化けになるため、**事前確認を省略しないこと**。
