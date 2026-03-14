# CLAUDE.md - GWS CLI Template

GWS CLI (`@googleworkspace/cli`) を使った Google Workspace 自動化テンプレート

## 概要

このリポジトリは GWS CLI のセットアップ・スクリプト・セキュリティルールを束ねる「ランチャー」。
GCP インフラ（API 有効化、IAM 等）の管理は別リポジトリ `gcp-ops-guide` で Terraform により行う。

## ディレクトリ構成

```
scripts/                        - ユースケース & ヘルパースクリプト
  ├── uc1-slides-template.sh    - Slides テンプレート量産
  ├── uc2-gmail-to-drive.sh     - Gmail 添付 → Drive 保存
  ├── uc3-issues-gmail-to-sheets.sh - Issues + Gmail → Sheets
  ├── helper-xlsx-to-sheets.sh  - Excel → Sheets 変換
  ├── helper-xlsx-update.sh     - Sheets → xlsx 書き戻し
  └── helper-sheets-rw.sh       - Sheets 汎用 読み書き
setup.sh                        - セットアップスクリプト
Taskfile.yml                    - タスクランナー
claude-code-settings.json       - AI エージェント deny 設定
AGENTS.md                       - GWS CLI コマンドリファレンス（全API）
```

## よく使うコマンド

```bash
# タスクランナー経由（task コマンド）
task check            # 前提条件チェック
task setup            # GWS CLI インストール & API 有効化
task auth             # OAuth 認証
task auth-status      # 認証ステータス確認
task uc1 / uc2 / uc3  # ユースケース実行
task all              # 全ユースケース順次実行

# ヘルパー
task xlsx-to-sheets -- <FILE_ID>             # xlsx → Sheets 変換
task xlsx-update -- <SHEETS_ID> <XLSX_ID>    # Sheets → xlsx 書き戻し
task sheets-read -- <SHEET_ID> "A1:Z100"     # Sheets 読み取り
task sheets-write -- <SHEET_ID> "A1" '[…]'   # Sheets 書き込み
task sheets-append -- <SHEET_ID> "A1" '[…]'  # Sheets 追記
task sheets-list -- <SHEET_ID>               # シート一覧

# 直接実行
source .env && gws auth login
gws auth status 2>/dev/null
```

## 安全ルール（claude-code-settings.json で deny 済み）

- **削除禁止**: delete / trash / remove（Drive, Calendar, Admin 含む全サービス）
- **メール送信禁止**: gmail send / drafts create / drafts send
- **カレンダー予定作成禁止**: calendar events insert
- **認証情報保護**: .env 読み取り、credentials / client_secret ファイル読み取り、auth export すべて禁止
- データの **読み取り・更新** は許可

## GWS CLI 必須パターン

```bash
# stderr フィルタリング — jq にパイプする前に必ず付ける
gws drive files list --params '...' --format json 2>/dev/null | jq '.'

# 共有ドライブ — 全 Drive API に必須
--params '{"supportsAllDrives": true}'
```

## 関連リポジトリ

| リポジトリ | 用途 | 関係 |
|-----------|------|------|
| `gcp-ops-guide` | GCP インフラ管理（Terraform） | API 有効化・IAM はこちらで管理 |

GCP 側のセットアップ（API 有効化、サービスアカウント作成等）が必要な場合は `gcp-ops-guide` リポジトリを参照。

---

## Claude Code 向け: GCP/GWS 既知のハマりポイント

以下は `gcp-ops-guide` リポジトリから抜粋した、GWS CLI 作業時に遭遇しうる問題と対処法。

### 1. gcloud CLI の PATH 問題

brew でインストールした gcloud は `/opt/homebrew/bin/gcloud` にあるが、
シェル環境によっては見つからない。

```bash
# 確実に実行するにはフルパスを使う
/opt/homebrew/bin/gcloud --version
```

### 2. OAuth 同意画面は CLI から作成できない（個人アカウント）

個人 Gmail アカウント（組織なし）のプロジェクトでは、
OAuth 同意画面を gcloud CLI や API から作成できない。

**対処:** GCP Console（Web UI）で手動作成。
- 同意画面: `https://console.cloud.google.com/apis/credentials/consent?project=<PROJECT_ID>`
- クライアント ID: `https://console.cloud.google.com/apis/credentials?project=<PROJECT_ID>`

### 3. OAuth 同意画面のテストユーザー追加

GCP Console の OAuth 同意画面設定で、テストユーザーの追加は最初のページには表示されない。

**手順:** SAVE AND CONTINUE で進む → 3ページ目「Test users」で ADD USERS

### 4. GWS CLI の「Access blocked: Authorization Error」

原因は2つ:
1. **OAuth 同意画面が未設定** → Console で設定する
2. **テストユーザーに自分のメールが未追加** → 上記 Step 3 で追加

「このアプリは Google で確認されていません」警告が出る場合:
→ **Advanced（詳細）** → **Go to \<app-name\> (unsafe)** をクリック

### 5. GWS CLI の client_id 不一致

GWS CLI は認証時の Client ID を `~/.config/gws/client_secret.json` に保存する。
`.env` の Client ID を変更しても、この内部ファイルは自動更新されない。

**対処:** Client ID を変更した場合は `~/.config/gws/client_secret.json` も手動で更新。

```bash
# 不一致の確認
gws auth status  # client_id と config_client_id を比較
```

### 6. GCP API 有効化で PERMISSION_DENIED

プロジェクト作成直後は API 有効化に数秒〜数分のラグがある。

**対処:** 少し待ってからリトライ。または複数 API をまとめて1コマンドで有効化。
