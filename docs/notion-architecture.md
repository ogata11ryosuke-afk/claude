# Notion Architecture — Masters Swim Log

Windows の Claude Code Desktop(Notion MCP 接続済)で構築する Notion ワークスペースの完全仕様。

**前提**: 実行側の Claude は `notion-search` / `notion-fetch` / `notion-create-database` / `notion-create-pages` / `notion-create-view` 等の Notion MCP ツールを使用できる。

---

## 全体構造

```
📘 Masters Swim Log (親ページ・新規作成)
 ├─ 🧍 Swimmer Profile        ← サブページ(静的)
 ├─ 🏊 Training DB             ← メインハブ(1行=1セッション)
 ├─ 📊 Reviews DB              ← AI生成の振り返り(週次/月次/フェーズ判断)
 ├─ 🩹 Injury Log DB           ← 痛み・違和感の状態変化
 └─ 🧠 Memo Summary            ← AI生成・週次上書き更新
```

### 設計原則

1. **Training DB が中心** — 予定も実績も同一行、`状態` プロパティで区別(行を増やさない)
2. **メモは Training DB のページ本文** — 別DBを作らず自由記述エリアに集約
3. **タグは Multi-select で横断フィルタ** — 後で「肩痛言及の週」等を即抽出
4. **Reviews DB は Training DB に Relation** — 引用元セッションへのリンク保持
5. **Memo Summary は上書き型ページ** — 履歴が必要なら Reviews DB に格納する

---

## Database 1: Training DB

### プロパティ

| 名称 | Type | 詳細 / 選択肢 |
|---|---|---|
| `Name` | Title | 自動: `{日付} {曜日} {種別} {テーマ}` をデフォルト表示(ユーザが変更しなくてOK) |
| `日付` | Date | 必須 |
| `曜日` | Formula | `formatDate(prop("日付"), "ddd")` で月/火/…を表示 |
| `週開始日` | Formula | `dateSubtract(prop("日付"), day(prop("日付")) == 0 ? 6 : day(prop("日付")) - 1, "days")` で月曜日を算出 |
| `種別` | Select | `プール` 🏊 / `陸トレ` 💪 / `モビリティ` 🧘 / `休息` 😴 |
| `テーマ` | Select | `技術` / `有酸素` / `スピード` / `統合` / `回復` / `導入` / `その他` |
| `所要(分)` | Number | 予定時間(format: Number) |
| `実所要(分)` | Number | 実施時間(空欄=未記入) |
| `状態` | Select | `📋 予定`(default) / `✅ 実施` / `⏭ スキップ` / `🚫 中止` |
| `詳細メニュー` | Rich text | 改行含む実行手順(Sheets の 詳細メニュー列を移行) |
| `主要キュー/注意` | Rich text | キューと禁忌の要約 |
| `RPE` | Number | 0-10 整数 |
| `肩NRS前` | Number | 0-10 整数 |
| `肩NRS後` | Number | 0-10 整数 |
| `膝NRS` | Number | 0-10 整数 |
| `タグ` | Multi-select | (下記オプション一覧) |
| `関連レビュー` | Relation → Reviews DB | (双方向) |
| `作成日時` | Created time | 自動 |
| `更新日時` | Last edited time | 自動 |

### タグ(Multi-select)オプション

- **痛み/違和感系**: `肩痛` / `膝違和感` / `腰違和感` / `首張り`
- **メンタル系**: `モチベ↑` / `モチベ↓` / `集中○` / `集中△`
- **パフォーマンス系**: `タイム○` / `タイム△` / `ストローク数改善` / `心拍高`
- **技術要素**: `フォーム気づき` / `ナローキック` / `ヘッドダウン` / `プル軌道` / `プルアウト` / `undulation` / `呼吸` / `タイミング`
- **警告系**: `旧癖出現` / `1980s再浮上` / `幅広キック危険`
- **学習系**: `動画学習` / `Peaty参照` / `北島参照` / `Marchand参照` / `渡辺一平参照` / `学び`
- **その他**: `新発見` / `疲労感強` / `好調` / `要フォロー`

### ビュー(6種)

| アイコン/名前 | タイプ | 条件 |
|---|---|---|
| 📅 **カレンダー** | Calendar | `日付` |
| 🗓 **今週** | Table | `週開始日` = 今週月曜(Formula で自動判定) |
| 📍 **今日** | Table | `日付` = today |
| ⚠️ **未実施(遅れ)** | Table | `状態` = `📋 予定` AND `日付` < today |
| 🏊 **プール抜粋** | Table | `種別` = `プール` |
| 🎯 **テーマ別** | Board | Group by `テーマ` |

### 表示順(Tableビュー共通)

`日付` 降順。列順: `日付` / `曜日` / `種別` / `テーマ` / `所要(分)` / `状態` / `詳細メニュー`(概要のみ) / `RPE` / `肩NRS前/後` / `タグ`

---

## Database 2: Reviews DB

### プロパティ

| 名称 | Type | 詳細 |
|---|---|---|
| `タイトル` | Title | 例: `2026-W17 週次レビュー(技術週)` |
| `タイプ` | Select | `週次` / `月次` / `TT分析` / `故障傾向` / `フェーズ移行判断` |
| `期間開始` | Date | |
| `期間終了` | Date | |
| `今のイシュー` | Rich text | 1文(例:「ナローキック感覚が25m持続するか」) |
| `空(数値)` | Rich text | セッション件数、NRS推移など |
| `雨(解釈)` | Rich text | 2-3文で「何が起きているか」 |
| `傘(提案)` | Rich text | 次期アクション |
| `関連セッション` | Relation → Training DB | (複数) |
| `作成日時` | Created time | 自動 |

### ビュー

- 🗓 **時系列**: Table sort `期間開始` desc
- 🎯 **タイプ別**: Board group by `タイプ`

---

## Database 3: Injury Log DB

### プロパティ

| 名称 | Type | 詳細 |
|---|---|---|
| `タイトル` | Title | 自動: `{日付} {部位} {判定}` |
| `日付` | Date | |
| `部位` | Multi-select | `右肩` / `左肩` / `両肩` / `右膝` / `左膝` / `両膝` / `腰` / `首` / `その他` |
| `NRS` | Number | 0-10 |
| `増悪動作` | Rich text | |
| `判定` | Select | `🟢 Green` / `🟡 Yellow` / `🟠 Orange` / `🔴 Red` |
| `対応` | Rich text | |
| `次回評価日` | Date | |
| `解消日` | Date | 空 = オープン、値入り = 解消済 |
| `エスカレーション` | Checkbox | 受診/専門家相談を要する場合 |
| `メモ` | Rich text | 追加コンテキスト |

### ビュー

- 📍 **Open** (`解消日` empty) ← default
- 📅 **全件時系列** (sort `日付` desc)
- 🎯 **部位別** (Board group by `部位`)

---

## Page 4: Memo Summary(ページ・DBではない)

ページ本文を週1回 memo-curator が上書き。構造テンプレ:

```md
# Memo Summary — 2026-W17 (2026-04-20 〜 04-26)

期間: [期間]
データソース: Training DB 該当週の `詳細メニュー` + `タグ` + ページ本文
生成日: [date]

## 🏊 フォーム気づき
- [抽出した3-5項目、Trainingページへのバックリンク]

## 💪 身体の体感(肩・膝・筋疲労)
- [NRS推移要約、違和感の表現]

## 💭 モチベ・メンタル
- [モチベ↑/↓、集中度]

## ⏱ タイム/進捗
- [ストローク数、持続感、TT結果]

## 📚 学び・参考
- [動画学習の発見、参考選手の観察点]

## ⚠️ 旧癖出現
- [1980s系の再浮上兆候があれば要注意]

## 🔁 反復パターン(3回以上言及)
- [同テーマの繰り返し言及を抽出]

## 🎯 次週への申し送り
- [training-planner / performance-analyst 向けの入力]
```

---

## Page 5: Swimmer Profile(サブページ)

`swimmer-profile.md` の要点を Notion 上に複製:

- 基本情報(51歳・165-170cm・60-70kg・主戦場スプリント平)
- 過去の競技歴(3-17歳全国レベル・50m平PB 29.8)
- 現在のフェーズ: **Phase A**
- 戦略的意思(モダンフォームで神経系上書き)
- 制約(週1回25mプール・自重のみ・計測機器持込不可)
- 現症(両肩痛🟡・五十肩既往)
- 参照選手(Peaty/Marchand/北島/渡辺一平/Martinenghi)

プロファイル更新時はこのページを真実源として再同期する。

---

## 初期データ(移行元: Google Sheets)

### 移行元シートURL

https://docs.google.com/spreadsheets/d/1cEP2_xigk6T_co866SACkp2ER4l3lKtqp4IjRS6bUv8/edit

### Training DB に移行する 7 行(weekly-menu タブ)

**全7行の完全な内容は `.claude/data/initial-training-rows.json` に格納**(このドキュメントと同時作成)。Windows 側 Claude はそのJSONを読んで `notion-create-pages` を回す。

概要:
- 2026-04-20 月 モビリティ 25分 導入
- 2026-04-21 火 陸トレ 30分 技術(導入)
- 2026-04-22 水 モビリティ 15分 技術(予習)
- **2026-04-23 木 ★プール日★ プール 75分 技術週**(938字の詳細メニュー)
- 2026-04-24 金 モビリティ 15分 回復
- 2026-04-25 土 陸トレ 35分 技術(発展)
- 2026-04-26 日 休息 0分 回復

全行とも `状態 = 📋 予定` で投入。実施後に各行を更新する運用。

### Injury Log DB に移行する 1 行

- 日付: 2026-04-20
- 部位: `両肩`
- NRS: 3
- 増悪動作: ストリームライン
- 判定: `🟡 Yellow`
- 対応: R4毎日+ストリームライン短時間化
- 次回評価日: 2026-05-18
- 解消日: (空)

---

## 構築手順(Windows Claude への指示)

1. **親ページ作成**: Workspace に `Masters Swim Log` ページを新規作成
2. **Swimmer Profile サブページ**: `swimmer-profile.md` の内容を本文として作成
3. **Training DB 作成**: 上記プロパティ19個 + タグMulti-select の全オプション + ビュー6種
4. **Reviews DB 作成**: プロパティ10個 + ビュー2種
5. **Injury Log DB 作成**: プロパティ11個 + ビュー3種
6. **Memo Summary ページ作成**: テンプレ構造で初期化
7. **Training DB と Reviews DB の Relation 設定**: 双方向
8. **初期データ投入**:
   - Training DB: `.claude/data/initial-training-rows.json` の7行
   - Injury Log DB: 上記1行
9. **動作確認**: 今日行(4/20)を実施状態に変えて再取得、タグ追加してフィルタ動作
10. **完了報告**: 親ページのURL・各DBのID・構造サマリを出力

---

## スキル側の接続

構築完了後、以下スキルが Notion を利用:

- `log-keeper`: Training DB への追記/更新、Injury Log DB への追記、Memo Summary の更新
- `performance-analyst`: Training DB 読取、Reviews DB への生成結果書込(log-keeper経由)
- `memo-curator`: Training DBページ本文+タグから Memo Summary を生成(週次)

各スキルの実装は `.claude/skills/` 内の該当 SKILL.md を参照。
