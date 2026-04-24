# Notion 構築 — Windows 引き継ぎプロンプト

**使い方**: Windows の Claude Code Desktop(Notion MCP 接続済)で新規セッションを開き、対象リポジトリ `ogata11ryosuke-afk/claude` の ブランチ `claude/masters-swimmer-skills-team-tNpab` を開いたら、以下の「引き継ぎプロンプト」全文をそのまま貼り付けて送信してください。

---

## 引き継ぎプロンプト(コピペ用)

```
あなたはこのリポジトリの Head Coach(@CLAUDE.md を読み込む)として動作している。

タスク: Masters Swim Log の Notion ワークスペースを新規構築する。

### 手順

1. `@docs/notion-architecture.md` と `@swimmer-profile.md` を読んで全体像を把握する。

2. Notion MCP ツール(notion-search / notion-create-database / notion-create-pages
   等)を使って以下を順に実行:

   a. Workspace に親ページ「Masters Swim Log」を新規作成
   b. サブページ「Swimmer Profile」を作成し、@swimmer-profile.md の要点を本文に貼る
   c. 親ページ直下に「Training DB」を作成(プロパティは
      @docs/notion-architecture.md の Database 1 仕様どおり、ビュー6種含む)
   d. 同様に「Reviews DB」(Database 2)、「Injury Log DB」(Database 3) を作成
   e. 「Memo Summary」ページを作成し、同 docs の「Page 4」のテンプレを本文にする
   f. Training DB と Reviews DB の間に Relation プロパティを設定(双方向)

3. 初期データ投入:
   a. Training DB に `.claude/data/initial-training-rows.json` の7行を投入
      (status = 📋 予定、詳細メニュー/キュー/注意も含む)
   b. Injury Log DB に 1 行投入:
      - 日付: 2026-04-20 / 部位: 両肩 / NRS: 3
      - 増悪動作: ストリームライン / 判定: 🟡 Yellow
      - 対応: R4毎日+ストリームライン短時間化
      - 次回評価日: 2026-05-18 / 解消日: (空)

4. 動作確認:
   a. notion-search で「Masters Swim Log」を検索し親ページIDが取れることを確認
   b. Training DB の「今週」ビューで4/20-4/26の7行が表示されることを確認
   c. 4/20行の「状態」を「✅ 実施」に変更して再取得(読み書き往復テスト)

5. 完了報告:
   a. 親ページURL
   b. Training DB / Reviews DB / Injury Log DB それぞれのID(将来スキルから参照)
   c. 各DBのレコード件数

6. スキル側の接続更新:
   リポジトリ内の `.claude/skills/log-keeper/SKILL.md`
   `.claude/skills/performance-analyst/SKILL.md`
   `.claude/skills/memo-curator/SKILL.md` の `## Notion 接続情報` セクションに、
   取得した親ページURL・各DB IDを記入してコミット。

### 制約

- 旧パターンドリル/幅広キック/長ストリームラインを暗に推奨しない
- 肩痛・五十肩既往・膝の Breaststroker's knee 予防は @CLAUDE.md の絶対ルールに従う
- 25mプール前提(LCM想定のセットは出さない)
- Notion MCP の書き込み系ツールは初回承認ダイアログが出る可能性あり — 承認して続行

### 制約2(重要)

- 既存 Google Sheet (ID: 1cEP2_xigk6T_co866SACkp2ER4l3lKtqp4IjRS6bUv8) は
  アーカイブとして残す。Notion 移行後は基本アクセスしない。
- サンドボックス側に残っている Google サービスアカウント JSON と
  NOTION_TOKEN は構築完了後にローテーション予定。
- 作業完了後に git commit + push する。ブランチは
  claude/masters-swimmer-skills-team-tNpab。

### 開始前の質問(任意)

架構が不明な点があれば、実行前に1-2問までまとめて質問してOK。
それ以上の不明点は自分で @docs/notion-architecture.md を再読することで解決できるはず。

開始してください。
```

---

## 引き継ぎ後の流れ(参考)

Windows Claude が構築完了したら:

1. そのセッションで取得した親ページ URL を スマホ/PCのブラウザで開いて目視確認
2. iPhone/iPad の Notion アプリでも同じページにアクセスできることを確認
3. 木曜 4/23 のプール前後に Notion アプリで直接記録開始
4. 日次の気付きは Training DB 該当行のページ本文(メモ)に自由記述
5. 週末に「週次レビューして」→ performance-analyst が動く(Windows セッションで)
6. 月1回「メモサマリ作って」→ memo-curator が Memo Summary を更新

---

## 既知の制約

- このサンドボックス(Claude Code Remote)から Notion API への直接アクセスは
  egress 制限で不可。Notion 関連は必ず Windows セッションで実行する。
- iPhone/iPad から Notion を使うのは通常どおり可能(Notion 公式アプリ)。
  Claude に依頼したい場合も Windows 側セッションに一度集約する。

---

## トラブルシュート

**Q: Windows 側で「notion-* ツールが見当たらない」**
- A: Claude Code Desktop の Customize → Connectors で Notion が有効か確認

**Q: 初回の書込で「Not authorized」が出る**
- A: Notion 側でそのページ/ワークスペースが Integration に共有されていないため。
  Notion の親ページ右上「…」→ Connections → 該当 Integration を接続

**Q: Training DB のビューが意図通りにならない**
- A: @docs/notion-architecture.md の「Database 1 → ビュー(6種)」を再確認。
  Formula プロパティの式を個別に指定する必要あり
