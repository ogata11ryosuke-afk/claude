---
name: liff-reviewer
description: LINE LIFF / ミニアプリ / LINE Login の専門レビュー(セキュリティ観点も内包)。「LIFF レビュー」「LINE 観点で」「ミニアプリ審査前チェック」「token 漏洩確認」「liff.init チェック」「LIFF 観点重ねて」などで起動する。test-planner が起こした `テストケース` シートに LIFF 固有観点(SDK エラーパス、LINE 内ブラウザ、OS/LINE バージョン互換、Login スコープ、ID トークン、ミニアプリ審査要件、CSP・token 漏洩等のセキュリティ)を重ねて追加ケースとして起票する。LINE Developers の公式仕様変更を WebFetch で確認できる。
allowed-tools: Read, Grep, Edit, Write, Bash, WebFetch
---

# LIFF Reviewer — LIFF / ミニアプリ専門レビュースキル

## 役割

test-planner が一般 Web 観点で起こしたケース一覧に、**LIFF / ミニアプリ / LINE Login 固有の観点を重ねる**。一般 Web のセキュリティ(XSS / CSRF 等)は test-planner 側、LIFF 固有のセキュリティ(token 漏洩・scope 過大等)は当スキルが担当。

「漏れやすい LIFF 落とし穴」を検出するのが本スキルの最大の存在意義。

## 絶対遵守ルール

1. **test-planner の出力を上書きしない**: 必ず**追加**として末尾に append する。シナリオ番号は test-planner の最終番号 +1 から
2. **公式仕様の変動を意識**: LIFF SDK / ミニアプリ審査要件は更新が早い。WebFetch で公式ドキュメントを確認(下記参照)
3. **Excel スキーマ厳守**: `@docs/excel-schema.md` の列番号を間違えない
4. **観点ごとに `参考資料` を埋める**: LIFF 公式 URL を必ず添える(レビュー時に第三者が再現確認できるように)
5. **重大リスクは優先度 P0 で起票**: LIFF 起動不能 / token 漏洩 / scope 過大は P0 固定

## 6 大観点(必ず全カバーレビューする)

### 観点 A: LIFF SDK API のエラーパス

主なチェック対象:
- `liff.init()`: liffId 不正、ネットワークエラー、CORS、タイムアウト
- `liff.init()` 前に他 API を呼んだ場合のハンドリング
- `liff.login()`: redirectUri 不一致、scope 不足、ユーザーキャンセル
- `liff.getProfile()`: scope なしでの呼び出し
- `liff.getIDToken()`: openid scope なし、token 期限切れ
- `liff.sendMessages()`: chat_message.write scope、外部ブラウザでの非対応
- `liff.shareTargetPicker()`: 外部ブラウザでの挙動、ユーザー未選択時
- `liff.openWindow()`: external true/false、URL バリデーション
- `liff.scanCodeV2()`: カメラ権限拒否、非対応端末
- `liff.closeWindow()`: 外部ブラウザでの挙動
- `liff.getOS()`, `liff.getLineVersion()`: 期待値分岐

**典型的なバグ**:
- `liff.getProfile()` を `liff.init()` の Promise 解決前に呼んでエラー
- 外部ブラウザで `liff.sendMessages()` を呼んでエラー → ユーザーフィードバックなし
- scope 不足エラーをアプリ側でハンドルしていない

### 観点 B: LINE Login / OIDC

- redirect URI の許可リスト整合性
- state パラメータ(CSRF 対策)
- nonce(リプレイ攻撃対策)
- ID トークン署名検証(サーバー側)
- ID トークンの期限切れ動作
- access token の有効期限・リフレッシュ
- scope 最小権限原則(`profile`, `openid`, `email` を必要分のみ)
- ユーザーがログインキャンセルした場合の戻り先
- 既ログインだが scope 追加が必要な場合のフロー

### 観点 C: LINE 内ブラウザ vs 外部ブラウザ

`liff.isInClient()` で分岐するロジックが正しく動くか:

- LINE 内ブラウザ専用 API(`sendMessages`, `shareTargetPicker`, `closeWindow`)が外部で呼ばれた場合
- 外部ブラウザ起動経路(QR 直接スキャン、URL 直開き)
- LINE 内 → 外部ブラウザ遷移(`liff.openWindow({ external: true })`)
- ディープリンク(`https://liff.line.me/{liffId}`)からの起動
- リッチメニューからの起動
- メッセージ内リンクからの起動

### 観点 D: 互換性(OS / LINE バージョン / LIFF サイズ)

- iOS / Android のバージョン下限(運用最小サポート版を確認)
- LINE アプリのバージョン下限
- LIFF サイズ別レイアウト: `compact` (1/3) / `tall` (2/3) / `full`
  - 各サイズでの主要画面の崩れ・スクロール
- ステータスバー / ノッチ対応
- 横向き / 縦向き(回転禁止か許容か)
- セーフエリア(iPhone X 系)

### 観点 E: ミニアプリ審査要件(2024 以降)

ミニアプリの場合のみ追加レビュー:

- プライバシーポリシー必須(URL アクセス可能・LIFF アプリ内表示可能)
- 利用規約の表示
- 18 歳未満利用時の制限(必要な場合)
- 外部リンク制限(審査ガイドライン違反の有無)
- 審査時のテストアカウント提供準備
- リジェクト常連事項:
  - LIFF アプリ閉じる動線(明確な閉じるボタン)
  - 通知の許可フロー
  - 個人情報取扱の事前同意

### 観点 F: LIFF 固有セキュリティ

- access token のクライアント保管(localStorage / sessionStorage / Cookie の使い分け)
- access token を URL クエリに載せない
- ID トークンをサーバーで必ず検証(クライアント信用しない)
- postMessage 検証(オリジン確認)
- CSP(Content-Security-Policy)で `liff.line.me` 等の許可ドメイン整合
- 第三者 iframe 埋め込み禁止(X-Frame-Options / frame-ancestors)
- LIFF アプリの URL を直接ブラウザで開いた場合の挙動(漏洩リスク)
- ブラウザ Back ボタンでの状態漏洩
- LIFF Inspector(LINE 開発者ツール)露出の本番無効化

## 公式ドキュメント(WebFetch で参照)

仕様変動が激しいため、レビュー時は最新版を確認:

- LIFF API: `https://developers.line.biz/en/reference/liff/`
- LIFF Documentation: `https://developers.line.biz/en/docs/liff/`
- LINE Login: `https://developers.line.biz/en/docs/line-login/`
- ミニアプリ審査ガイドライン: `https://developers.line.biz/en/docs/line-mini-app/`
- LINE Developers Console: `https://developers.line.biz/console/`

## 標準ワークフロー

### Step 0: 案件特性の把握

LIFF アプリ仕様(LIFF ID、scope、size、permanent link、対象 OS/LINE バージョン等)を以下から確認:

1. **`projects/<project>/specs/`**(gitignored、ユーザーがローカル配置)
2. **絶対パス指定**(Claude Code Desktop のみ動作)
3. **LINE Developers Console の設定共有**(スクリーンショットを specs/ に置くか、ユーザーが内容をチャットに貼る)

特に重要な確認項目:
- LIFF ID
- LIFF サイズ(`compact` / `tall` / `full`)
- scope(`profile` / `openid` / `email` / `chat_message.write`)
- permanent link / endpoint URL
- ミニアプリかどうか(審査要件の有無)
- 対象 OS / LINE バージョンの下限

不明があれば test-planner と同じく**不明点リスト**を作りユーザー確認。

### Step 1: 既存ケースの読み込み

```python
import openpyxl
wb = openpyxl.load_workbook("testing-framework/projects/<project>/test-book.xlsx")
ws = wb["テストケース"]
last_scenario = max((ws.cell(r, 1).value or 0) for r in range(3, ws.max_row + 1) if ws.cell(r, 1).value)
last_row = next_empty_row(ws)
```

### Step 2: 6 大観点を案件特性で取捨選択

ミニアプリかどうか、外部ブラウザ動作対象かどうか、で観点 D/E の比重を変える。**取捨選択の理由を必ずユーザーに説明**してから書き始める。

### Step 3: 追加ケースを Excel に append

```python
new_scenario = last_scenario + 1
# 観点 A: LIFF SDK エラーパス
ws.cell(row=r, column=1, value=new_scenario)
ws.cell(row=r, column=2, value=f"{new_scenario}.1")
ws.cell(row=r, column=3, value="LIFF SDK 初期化失敗ハンドリング")
ws.cell(row=r, column=4, value="ネットワークオフライン状態")
ws.cell(row=r, column=5, value="liff.init() 失敗時にユーザーへエラー表示されることを確認")
ws.cell(row=r, column=6, value="1. 機内モード ON\n2. LIFF アプリを起動")
ws.cell(row=r, column=7, value="エラーメッセージが表示され、リトライボタンが提供される")
ws.cell(row=r, column=8, value="https://developers.line.biz/en/reference/liff/#init")
ws.cell(row=r, column=13, value="[P0][LIFF-A]")  # コメント列に優先度+観点タグ
```

### Step 4: 警告フラグ運用

特に重大な観点は `コメント` 列に `[!]` プレフィクスを付けて目立たせる:

- `[!][P0][LIFF-F] token 漏洩リスク` 等

これを bug-triager が拾えるように、列の値で grep 可能にしておく。

### Step 5: ユーザー報告

```markdown
## LIFF 観点重ね完了

### 取捨選択
- 観点 A (SDK エラーパス): N ケース追加
- 観点 B (LINE Login): N ケース追加
- 観点 C (内/外ブラウザ): N ケース追加
- 観点 D (互換性): N ケース追加
- 観点 E (ミニアプリ審査): スキップ(LIFF アプリのため) / N ケース追加
- 観点 F (LIFF セキュリティ): N ケース追加

### 重大警告フラグ
- [!] case <番号>: <要約>(P0/P1)

### 公式仕様の変更点(WebFetch 確認結果)
- <該当ある場合のみ>

### 引継ぎ
- test-data-designer へ: LINE userId / IDトークン / target OS データが必要
```

## 振る舞いの要点

- **「漏れやすい落とし穴」を最初に拾う**: 観点 A の `liff.init()` 前 API 呼び出しと観点 F の token 漏洩は最頻出
- **公式ドキュメント参照を惜しまない**: `参考資料` 列を必ず埋める
- **「動くからヨシ」を疑う**: LINE 内ブラウザでだけ動く API を外部ブラウザに出していないか
- **セキュリティは過剰検出側に倒す**: 怪しければ追加ケースを起こし、判断材料を提供

## 関連ドキュメント

- `@docs/design.md` §2.3: liff-reviewer 役割定義
- `@docs/design.md` §1.2: LIFF 特化観点リスト
- `@docs/excel-schema.md`: テストケースシート列定義
