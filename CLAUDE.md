# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## リポジトリの位置づけ

[chezmoi](https://www.chezmoi.io/) で管理されるユーザー個人のドットファイル群。リポジトリ自体はソースであり、`chezmoi apply` でホームディレクトリに展開される。`master` ブランチが production。

## chezmoi 命名規約（このリポジトリの読み方）

ファイル名そのものが chezmoi の指示子になっているため、リネーム・新規追加時は規約に従う必要がある。

- `dot_xxx` → `~/.xxx` に展開（例: `dot_zshrc.tmpl` → `~/.zshrc`）
- `dot_xxx.tmpl` → Go テンプレートとして処理してから展開
- `dot_config/...` → `~/.config/...`
- `run_once_before_*.sh.tmpl` → `chezmoi apply` 時に **一度だけ・他のファイル展開より前に** 実行されるスクリプト（`*.sh.tmpl` はテンプレート処理後にスクリプトとして実行）

## OS 分岐の構造

`*.tmpl` ファイル内で `.chezmoi.os` による分岐が組まれており、同一ファイルから darwin / linux / windows それぞれの最終出力を生成する。OS 固有の差分を入れる場合は新規ファイルではなく既存 `.tmpl` のブロックを編集する。

```go-template
{{ if eq .chezmoi.os "darwin" -}}
...
{{ else if eq .chezmoi.os "linux" -}}
...
{{ else if eq .chezmoi.os "windows" -}}
...
{{ end -}}
```

現在 Linux ブロックは **WSL2 前提** で、ホスト Windows 側の 1Password SSH Agent / `win32yank.exe` に依存している（`alias ssh=ssh.exe`、tmux の `copy-pipe-and-cancel "cat | win32yank.exe -i"` など）。Linux 編集時はこの前提に注意。

Windows 向けには PowerShell プロファイルを **`dot_config/powershell/profile.ps1`** で共通管理し、Windows PowerShell 5.1 と PowerShell 7+ の両方の `$PROFILE` から source する設計（詳細は下節）。OS ガードはルートの `.chezmoiignore`（テンプレート）で `.config/powershell` を非 Windows では除外することで実現している。

## よく使うコマンド

ドットファイルそのもののテストはなく、検証は chezmoi のドライラン／差分で行う。

```bash
chezmoi diff                      # 適用したらホームに何が変わるかを確認
chezmoi apply -v                  # 実際に展開
chezmoi execute-template < file   # 単一テンプレートのレンダリング結果を確認
chezmoi cd                        # このリポジトリに移動（chezmoi 経由運用時）
```

## ファイル間の依存関係

- `dot_zshrc.tmpl` → `dot_zsh/functions/` 配下を `fpath` に追加し autoload する。新しい関数を増やす場合は `autoload -Uz <name>` の追記が必要。
- `dot_config/nvim/init.vim` → `dot_config/nvim/dein.toml` と `deinlazy.toml` を読み込み、各プラグインの設定は `dot_config/nvim/plugins/*.rc.vim` に分離されている（dein の `hook_add` から `source` される）。
- `dot_gitconfig` は `~/.gitconfig.local` を `[include]` しているため、機微な設定（業務用メールなど）はリポジトリに入れず local 側に置く。同様に `dot_zshrc.tmpl` 末尾も `~/.zshrc.local` を source する。
- **PowerShell プロファイル**：本体は `dot_config/powershell/profile.ps1`（→ `~/.config/powershell/profile.ps1`）に集約。実 `$PROFILE`（Windows PowerShell 5.1 と PowerShell 7+ の 2 箇所）には `. "$HOME\.config\powershell\profile.ps1"` の 1 行ブートストラップだけが入る。このブートストラップ生成は `run_onchange_before_install-powershell-profile-windows.ps1.tmpl` が担当：
    - `$PROFILE` が存在しなければ作成、存在すれば触らない（source 行が無ければ警告のみ）
    - 5.1 用パスは常に対象、7+ 用パスは `Get-Command pwsh` で動的検出
    - `Documents` フォルダーは OneDrive / KnownFolder で別ドライブにリダイレクトされ得るため、必ず `[Environment]::GetFolderPath('MyDocuments')` で動的解決する（`~/Documents` をハードコードしない）
    - `run_onchange_*` 名なので、テンプレ展開時の `pwsh` 検出結果（コメント先頭行に埋め込み）が変わると再実行される — つまり 7+ を後からインストールすれば次の `chezmoi apply` で 7+ 用ブートストラップも自動で生成される
- エディション固有のコードを書きたいときは本体内で `$PSVersionTable.PSVersion.Major` で分岐する（5.1 と 7+ で profile を分けない方針）。
