# Google Sheets MCP セットアップガイド

本リポジトリで使用する `mcp-google-sheets` の初回セットアップ手順。
**所要時間の目安: 15〜25分**(GCP画面を初めて触る場合)。

---

## 完了後にできるようになること

- `log-keeper` スキルが Google Sheets に練習記録を追記できる
- `performance-analyst` スキルが履歴を読み取って週次/月次レビューを生成できる
- iPhone/iPad/PC の Claude Code Web(claude.ai/code)で同じシートを共用できる

---

## Step 0: 前提

- Google アカウントを持っていること(無料で十分)
- Google Sheets を1つ新規作成できること
- クレジットカード登録は **不要**(Sheets API は無料枠内)

---

## Step 1: GCP プロジェクトを作成(5分)

1. https://console.cloud.google.com/ にアクセス(Googleログイン)
2. 画面上部の「プロジェクトを選択」プルダウン → 「新しいプロジェクト」
3. 名前: 例 `masters-swim-log`(自由)
4. 「作成」ボタン → 数秒待って作成完了
5. 作成したプロジェクトを選択状態にする(上部プルダウンで確認)

---

## Step 2: 必要な API を有効化(3分)

プロジェクトを選択した状態で:

1. 左メニュー「APIとサービス」→「ライブラリ」
2. 検索ボックスに `Google Sheets API` と入力 → 結果をクリック → **「有効にする」**
3. もう一度ライブラリへ戻り、`Google Drive API` を検索 → **「有効にする」**

※ Drive API はシート一覧取得などに必要。

---

## Step 3: サービスアカウントを作成(5分)

1. 左メニュー「IAMと管理」→「サービスアカウント」
2. 画面上部「サービスアカウントを作成」
3. 以下を入力:
   - **サービスアカウント名**: `masters-swim-mcp`(自由)
   - **サービスアカウントID**: 自動生成でOK
   - **説明**: `MCP server for swim log sheets`(任意)
4. 「作成して続行」
5. **「ロールを選択」はスキップしてOK**(個別のシートを共有するだけで権限が渡るため、プロジェクト全体のロール付与は不要)
6. 「完了」

---

## Step 4: サービスアカウントのメールアドレスをコピー(1分)

1. サービスアカウント一覧で、作成したアカウントの行を確認
2. **メールアドレス形式**(例: `masters-swim-mcp@masters-swim-log.iam.gserviceaccount.com`)をコピー
3. **このメールは後で Google Sheets の共有先として使う**ので、控えておく

---

## Step 5: JSON 鍵をダウンロード(2分)

1. サービスアカウント一覧で、作成したアカウント名をクリック
2. 上部タブ「キー」をクリック
3. 「鍵を追加」→「新しい鍵を作成」
4. キーのタイプ: **JSON** を選択 → 「作成」
5. 自動ダウンロードが始まる(ファイル名例: `masters-swim-log-abc123.json`)
6. **このファイルは超重要な秘密情報**。以下に注意:
   - GitHub には絶対にコミットしない(`.gitignore` で保護済み)
   - ファイル名は好きに変えてOK。例: `service-account.json` にリネーム
   - 失くしたら再作成が必要

---

## Step 6: Sheets 側で共有設定(2分)

1. Google Sheets で新規スプレッドシートを作成
   - 名前例: `Masters Swim Log - Swimmer`
2. 右上「共有」ボタン
3. **Step 4 でコピーしたサービスアカウントのメールアドレス**を入力
4. 権限: **編集者** に設定
5. 「通知しない」にチェック → 「共有」

※ これで、そのサービスアカウントからこのシートを読み書きできるようになる。

---

## Step 7: ローカルに鍵を配置(2分)

ダウンロードした JSON 鍵を、Claude Code から見える場所に配置:

### 推奨パス(macOS / Linux)

```
~/.secrets/masters-swim-service-account.json
```

```bash
mkdir -p ~/.secrets
mv ~/Downloads/masters-swim-log-*.json ~/.secrets/masters-swim-service-account.json
chmod 600 ~/.secrets/masters-swim-service-account.json   # 自分だけ読める権限
```

### Windows

```
C:\Users\<username>\.secrets\masters-swim-service-account.json
```

---

## Step 8: `.claude/settings.local.json` に絶対パスを記入(1分)

このファイルは `.gitignore` 済みなのでコミットされない。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "env": {
    "SERVICE_ACCOUNT_PATH": "/home/<username>/.secrets/masters-swim-service-account.json",
    "DRIVE_FOLDER_ID": ""
  }
}
```

- **macOS**: `/Users/<username>/.secrets/masters-swim-service-account.json`
- **Linux**: `/home/<username>/.secrets/masters-swim-service-account.json`
- **Windows**: `C:\\Users\\<username>\\.secrets\\masters-swim-service-account.json`(バックスラッシュをエスケープ)

`DRIVE_FOLDER_ID` は空のままで OK(特定フォルダに絞りたい場合のみ Drive で該当フォルダのURL末尾IDを入れる)。

---

## Step 9: Claude Code を再起動して動作確認

1. Claude Code を完全に再起動(セッションも新規で)
2. 起動時に `.mcp.json` の `gsheets` サーバーが承認済みとして認識される
3. 初回のみ「Approve MCP server `gsheets`?」と聞かれる可能性あり → **Allow**
4. 以下を試す:

```
「log-keeper を使って、Masters Swim Log シートの season-plan タブに
今日の日付でフェーズAと書き込んでみて」
```

成功すれば、Sheets に行が追加される。

---

## トラブルシューティング

### エラー: `uvx: command not found`

```bash
# uv(uvxを含む)をインストール
curl -LsSf https://astral.sh/uv/install.sh | sh
# PATH反映のため新しいシェルで
```

### エラー: `Permission denied` / `403 Forbidden`

- Step 6 で共有した対象のメールアドレスが、Step 4 でコピーしたサービスアカウントのメールと完全一致しているか確認
- シート側の共有権限が「編集者」になっているか確認

### エラー: `FileNotFoundError: SERVICE_ACCOUNT_PATH`

- `.claude/settings.local.json` のパスが絶対パスになっているか
- そのパスに実際にファイルが存在するか(`ls -la` で確認)

### エラー: `Google Sheets API has not been used in project ...`

- Step 2 でAPIを有効化していない。再度ライブラリから有効化

### iPhone/iPad の Claude Code Web で使いたい

- Web環境ではローカルファイルパスを参照できない。代わりに以下のいずれか:
  - **(推奨)** Claude Code Web の Environment Variables 設定画面で `SERVICE_ACCOUNT_PATH` または同等のシークレットを登録
  - もしくは `GOOGLE_APPLICATION_CREDENTIALS_JSON` として JSON 本体を直接env varに格納(使用するMCPサーバ実装が対応していれば)
- まず PC で動作確認してから Web 環境に移行することを推奨

---

## セキュリティチェックリスト

- [ ] サービスアカウントJSONをGitリポにコミットしていない(`git status` で確認)
- [ ] `.gitignore` に credential 系パターンが入っている
- [ ] 鍵ファイルの権限が `600`(自分のみ読取可)
- [ ] 共有したSheetに不要な他ユーザーが追加されていない
- [ ] サービスアカウントにプロジェクトレベルのロールを付与していない(必要最小権限の原則)

---

## 最後に

このガイドで詰まったら、どのステップで何が起きたかを教えてください。Claude が該当箇所を個別にトラブルシュートします。
