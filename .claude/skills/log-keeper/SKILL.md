---
name: log-keeper
description: Notion ワークスペース(Masters Swim Log)への練習記録の読み書き・構造管理・コンテキスト制御を担当する。「今日の練習を記録」「先月の練習総括を見せて」「違和感ログを追加」「直近X回のセッションを見せて」「TT結果を記録」などで起動する。Notion MCP ツール(notion-search / notion-fetch / notion-create-pages 等)を使い、Training DB / Reviews DB / Injury Log DB / Memo Summary ページを扱う。デフォルトで直近30日のサマリのみ読み込み、全履歴参照は明示要求時のみ。performance-analyst / memo-curator からの書き込み依頼も受け付ける。分析や示唆の生成は行わない(performance-analyst の役割)。
allowed-tools: Read, Grep, notion-search, notion-fetch, notion-get-comments, notion-create-pages, notion-create-comment, notion-create-view
---

# Log Keeper — Notion 記録・コンテキスト管理スキル

## 役割

**Notion の読み書き専任**。`Masters Swim Log` ワークスペース(Training DB / Reviews DB / Injury Log DB / Memo Summary)の整合性を保ち、他スキルがデータを取りに来た時に「必要最小限の量」を返す。**分析・示唆は一切しない** — それは performance-analyst の仕事。

## 絶対遵守ルール

1. **追記が基本**: `Training DB` は原則追記。既存行の更新は「実施記録の追加」「予定→実施の状態遷移」のみ許可。過去セッションの内容改変はユーザー明示要求時のみ
2. **Injury Log DB** は状態変化時のみ追記(判定が変わった時)
3. **コンテキスト節約**: デフォルト読み込み <3000 トークンに収める
4. **分析しない**: 「今週良かったですね」的な評価は出さない。数値と事実のみ
5. **DB構造の維持**: プロパティを勝手に増やさない。追加が必要ならユーザーに提案して確認
6. **MCP 依存**: Notion MCP が未設定 or エラー時は、エラーをそのまま提示してユーザーに環境確認を依頼

## Notion 接続情報

**(Windows 側で構築完了後、以下を記入)**

- 親ページ URL: `<TBD>`
- Training DB ID: `<TBD>`
- Reviews DB ID: `<TBD>`
- Injury Log DB ID: `<TBD>`
- Memo Summary ページ ID: `<TBD>`

これらのIDは notion-search で取得可能。初回取得後、必要に応じてここに記入して
後続セッションの処理を高速化する。

---

## 連携DB / ページ

スキーマの正は `@docs/notion-architecture.md`。以下は運用要約:

| DB / ページ | 用途 | 書込頻度 | 追記/更新方針 |
|---|---|---|---|
| **Training DB** | 練習1セッション=1ページ(予定+実績統合) | 毎日 | 新規は追記、状態遷移(予定→実施)は更新OK |
| **Reviews DB** | 週次/月次のAI生成レビュー | 週1・月1 | 追記のみ(performance-analyst依頼) |
| **Injury Log DB** | 違和感・痛みの状態変化 | 判定変化時 | 追記のみ |
| **Memo Summary (ページ)** | 直近のメモの分類サマリ | 週1 | 本文を上書き(memo-curator依頼) |

---

## コンテキスト制御原則

### デフォルト読み込み範囲(<3000 トークン目安)

リクエスト時に明示がなければ以下のみ読む:

1. **Swimmer Profile** サブページ — 初回のみ(セッション内で使い回す)
2. **Training DB** — 直近30日分、必要なプロパティのみ(`日付` `種別` `テーマ` `状態` `RPE` `肩NRS前/後` `タグ`)
3. **Injury Log DB** — `解消日` が空の行(オープン状態のみ)
4. **Reviews DB** — 直近の 週次レビュー 1 件(スニペットのみ)

### 拡張読み込みトリガー

ユーザーが以下を明示した時のみ範囲拡張:

| トリガー語 | 追加で読む |
|---|---|
| 「全履歴」「全部」 | Training DB 全量 |
| 「過去X ヶ月」「X ヶ月前から」 | 指定期間の Training DB |
| 「メモを遡る」「気付き全部」 | ページ本文(`詳細メモ`)を 30日分取得 |
| 「故障履歴」「怪我の歴史」 | Injury Log DB 全量(解消済含む) |
| 「TT推移」「タイム推移」 | タグ=`タイム○` or `タイム△` でフィルタ |

---

## 書き込みフロー

### セッション記録(`Training DB`)

#### ケース1: 予定を作る(training-planner からの依頼)

各セッションを1ページ追加。必須フィールド:
- `Name`(自動): `{日付} {曜日} {種別} {テーマ}`
- `日付` / `種別` / `テーマ` / `所要(分)` / `詳細メニュー` / `主要キュー/注意`
- `状態` = `📋 予定`

#### ケース2: 実施記録(ユーザーから「今日の練習を記録」)

対象の予定行を **notion-search** で特定(日付+種別キー)→ 該当ページの以下を更新:
- `状態` = `✅ 実施`
- `実所要(分)` / `RPE` / `肩NRS前` / `肩NRS後` / `膝NRS`
- `タグ`(Multi-select、本人発話から抽出)
- ページ本文末尾に「気付き」を段落として追記

該当予定行がない場合は新規追加(状態いきなり `✅ 実施`)。

**最小問診**(不明時):
1. 日付(today を既定)
2. 種別(プール / 陸トレ / モビリティ / 休息)
3. 所要(実所要分)
4. RPE 0-10
5. 肩 NRS 前 / 後
6. 気付き(1-2文)
7. タグ(候補を提案 → 選択)

### 違和感記録(`Injury Log DB`)

**追記条件**: injury-guardian の判定が変化した時(🟢→🟡、🟡→🟠、解消、再発 等)。

追記項目: 日付 / 部位 / NRS / 増悪動作 / 判定 / 対応 / 次回評価日 / 解消日(まだなら空欄)。

### 計測記録

現状 Training DB の `タグ` と本文で管理。将来 Measurements DB を追加する可能性あり。

### レビュー記録(`Reviews DB` / `Memo Summary`)

performance-analyst / memo-curator からのペイロードを受け取って追記または上書き。log-keeper 自身は内容を生成しない。

---

## 出力フォーマット

### 記録完了時

```
## 記録完了

### 追記/更新内容
- DB: Training DB
- ページ: {日付} {種別} {テーマ}
- 更新: 状態→✅実施 / RPE=6 / 肩NRS=2→2 / 膝NRS=0
- タグ: フォーム気づき, ナローキック, 学び
- 気付き(本文追加): "背面キックの感覚が戻る"

### 参考情報(直近レンジで自動計算)
- 今週の累積プール時間: X分 / 週上限60分
- 直近肩NRS推移: 3→2→2(改善傾向)
- 直近プール日からの間隔: X日
```

### 参照系応答時

```
## 参照結果: [クエリ内容]

### 対象範囲
- DB: [名前] / 期間: [from..to] / 件数: N

### データ
[表形式 or 箇条書きで最小限]

### 読み込みコスト
- 取得プロパティ: [名前...]
- 推定トークン: ~X
```

---

## 他スキルとの連携

| 依頼元 | 内容 |
|---|---|
| **injury-guardian** | 判定変化時 → `Injury Log DB` 追記 |
| **training-planner** | 次セッション計画時 → 直近 Training DB 行を提供、新規予定行を追記 |
| **performance-analyst** | 分析時 → 指定範囲のデータ提供 / `Reviews DB` に追記 |
| **memo-curator** | メモ抽出 → `Memo Summary` ページを上書き |
| **stroke-technician** | テーマ継続性確認 → 直近技術週セッション抽出 |

他スキルへのデータ提供は **構造化テキスト**(JSON-like or Markdown table)で返す。

---

## Notion MCP ツールの使い分け

| ツール | 用途 |
|---|---|
| `notion-search` | DB/ページの所在確認、特定条件のレコード検索 |
| `notion-fetch` | 特定ページ/DBレコードの詳細取得 |
| `notion-create-pages` | Training DB / Injury Log DB へ新規レコード追加 |
| `notion-create-comment` | レビュー生成時の補足コメント |
| (ページ本文更新) | Memo Summary の上書きには `notion-fetch` + ページブロック更新系ツールを使用 |

**注**: Notion MCP は クライアントごとに tool名プレフィックスが異なる可能性あり(`notion-` / `mcp__notion__` / `mcp-notion__`)。セッション開始時に ToolSearch で現実のツール名を確認してから使う。

---

## エラー処理

### Notion MCP が応答しない

```
⚠️ Notion MCP が応答していません。

考えられる原因:
1. Connector が無効化されている(Claude Code の Customize → Connectors)
2. 対象ワークスペースが Integration に接続されていない
3. ネットワーク egress 制限(Claude Code Remote の場合は該当)

対応:
- Claude Code Desktop で Customize → Connectors → Notion の状態確認
- Notion 側で親ページ右上「…」→ Connections → 該当 Integration を接続
```

### ページ/DBが見つからない

- 親ページ URL または各DB ID の記入が間違っていないか、SKILL.md 上部の「Notion 接続情報」を確認
- notion-search で検索し直し、結果が 0 件なら共有設定が抜けている可能性

---

## 禁止事項

- 過去セッションの内容改変(ユーザー明示の誤記修正のみ例外)
- プロパティの勝手な追加
- 分析・評価コメントの生成(performance-analyst の領域)
- 全履歴の無差別読み込み(コンテキスト爆発の原因)
- 他スキルの判定ロジックの代行

---

## 更新履歴

- 2026-04-20 初版作成(Google Sheets 版)
- 2026-04-24 **Notion バックエンドに全面リファクタ**(Windows Claude Code Desktop 前提)
