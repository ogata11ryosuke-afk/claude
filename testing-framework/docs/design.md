# テストフレームワーク設計書

**対象**: LINE LIFF / LINE ミニアプリ(複数アプリで使い回す**汎用**フレームワーク)
**バージョン**: v0.3(設計フェーズ・Excel スキーマ確定)
**作成日**: 2026-04-28
**更新**:
- 2026-04-28 v0.2: 4 主要未決事項解消(汎用化、Excel 一元化、GitHub Actions、運用言語=日本語)
- 2026-04-28 v0.3: Excel テンプレ取り込み済み、スキーマ確定(`docs/excel-schema.md`)、日本語版テンプレ `test-book-template-ja.xlsx` 生成

---

## 1. 設計方針

### 1.1 基本原則

1. **単一責務**: 1 ロール 1 目的。役割の重複を避け、引継ぎ点を明確にする
2. **Skill / Subagent の使い分け**:
   - **Skill**: 助言・計画策定・レビュー(読み取り中心、メインコンテキストで動作)
   - **Subagent**: 実行系・並列性が必要・独立コンテキストが欲しい役割
3. **両モード対応**: アーキテクト主導パイプライン と 個別ロール直接起動 のどちらでも動作
4. **Excel 一元管理**: テストケースもバグレポートも **同一 Excel** に記録(ユーザー提供テンプレート `Test cases & Bug report.xlsx` 準拠)。設計議論や軽量メモのみ Markdown
5. **LIFF 特化 × 汎用フレームワーク**: 複数の LIFF/ミニアプリ案件で使い回す前提で、プロジェクト単位にディレクトリを切る
6. **運用言語は日本語**: テンプレート原本は英語だが、実運用時の記載・出力は日本語に変換して使う

### 1.2 実行環境とファイル運用

**前提**: 日常運用は **Claude Code Desktop(Windows)**。Web/Remote(Linux サンドボックス)はローカルファイルが見えないため、実プロジェクト適用には不向き。

#### git 管理 / 非管理 の境界

| 区分 | 例 | 理由 |
|---|---|---|
| git 管理 | スキル / 設計 / Excel テンプレ(空)/ 案件 test-book.xlsx / 自動テストコード / 補助メモ | 履歴・複数環境配布・チーム共有のため |
| **gitignore**(`projects/<name>/specs/`)| 仕様書 (Word/PDF) / ワイヤーフレーム / Figma エクスポート / 画面キャプチャ | 機密性・容量・更新頻度の観点で git に上げない方が運用が楽 |

ユーザーは仕様書を以下のいずれかで Claude に渡す:

1. **コピー**: `projects/<name>/specs/` 配下に置く(gitignore 対象)
2. **絶対パス参照**: OneDrive/Desktop の絶対パスを Claude に直接指定

スキル側はどちらにも対応する(Read tool に絶対パス・相対パスのどちらを渡しても良い)。

### 1.3 「LIFF 特化」が要求する独自テスト観点

設計の前提として、LIFF/ミニアプリで漏れやすい観点を最初から組み込む:

- LIFF SDK 初期化失敗・タイムアウトハンドリング
- `liff.init` 前後の API コール順序(`liff.getProfile()` を init 前に呼ぶ等)
- LINE 内ブラウザ vs 外部ブラウザ(`liff.isInClient()`)の挙動差
- iOS / Android / LINE バージョン(最低サポート版)別の互換性
- LINE Login スコープ(`profile`, `openid`, `email`)の最小権限原則
- ID トークン検証・access token のクライアント保管リスク
- `liff.sendMessages()` / `liff.shareTargetPicker()` の権限とエラーパス
- ミニアプリの審査要件(プライバシーポリシー必須、外部リンク制限等)
- LIFF サイズ(`compact` / `tall` / `full`)別のレイアウト崩れ
- ディープリンク(`https://liff.line.me/{liffId}` / カスタム URI スキーム)からの起動経路

これらは `liff-reviewer` の責務として定義し、`test-planner` が機能観点でテスト計画を立てた後に必ず重ねる。

---

## 2. ロール詳細仕様

### 2.1 system-architect(システムアーキテクト)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Skill**(`.claude/skills/system-architect/SKILL.md`) |
| 起動条件 | 「テスト全体を計画して」「テスト統括」「全部レビューして」「テストフレームワーク使って」等 |
| 主責務 | 全体オーケストレーション、各ロール成果物の整合性レビュー、最終承認 |
| 入力 | ユーザー要望(機能仕様 URL/PDF、LIFF アプリ概要、テスト目的) |
| 出力 | テスト戦略文書、ロール起動順、最終成果物統合レポート |
| 他ロールとの関係 | **全ロールを呼び出す**。test-planner → liff-reviewer → test-data-designer → automation-engineer/tester → bug-triager の順を基本とし、状況により並列化 |
| 使用ツール | Read, Grep, Bash(限定的), Agent(Subagent 起動用) |

**振る舞いの要点**:
- ユーザー要望を分解し、「今回は何を検証するのが価値最大か」を最初に提示(イシュー起点)
- 各ロールに渡すブリーフを明示(その役が何を返すべきか)
- ロール間の出力差分・抜け漏れを検出(例: test-planner は OS 互換性を見たが LIFF サイズ違いを見ていない、等)
- テスト実施後の Go/No-Go 判断を支援

### 2.2 test-planner(テスト計画者)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Skill**(`.claude/skills/test-planner/SKILL.md`) |
| 起動条件 | 「テスト計画」「テストケース作って」「観点出して」等 |
| 主責務 | 一般 Web システムの知識ベースで、機能/非機能テスト計画とケースを起こす |
| 入力 | 機能仕様、画面遷移、API 仕様 |
| 出力 | テスト戦略(範囲・優先度)、テストケース一覧(Excel 出力対象) |
| 包含観点 | 機能、UI/UX、アクセシビリティ(WCAG 2.2 AA 準拠ベース)、パフォーマンス(Lighthouse / Web Vitals)、互換性、回帰 |
| 連携 | liff-reviewer に LIFF 観点重ねを依頼、test-data-designer にデータパターン依頼 |
| 使用ツール | Read, Grep, WebFetch(仕様参照用) |

**カバレッジモデル**:
- 機能観点: ISO/IEC 25010 ベースに整理(機能適合性・性能効率性・互換性・使用性・信頼性・セキュリティ・保守性・移植性)
- リスクベース: 影響度 × 発生頻度で優先度 P0〜P3
- 同値分割・境界値・状態遷移・デシジョンテーブル・原因結果グラフ をケース粒度に応じて使い分け

### 2.3 liff-reviewer(LIFF / ミニアプリ専門レビュアー)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Skill**(`.claude/skills/liff-reviewer/SKILL.md`) |
| 起動条件 | 「LIFF レビュー」「LINE 観点で」「ミニアプリ審査前チェック」等 |
| 主責務 | test-planner が立てた計画/ケースに **LIFF/ミニアプリ固有レイヤ** を重ねる。LIFF 関連セキュリティもここに内包 |
| 入力 | test-planner のケース一覧、LIFF アプリの設定(LIFF ID、permanent link、スコープ) |
| 出力 | LIFF 固有観点での追加ケース、リスク警告、審査要件チェックリスト |
| カバー領域 | (1) LIFF SDK 全 API のエラーパス、(2) LINE Login / OIDC、(3) LINE 内ブラウザ環境、(4) ミニアプリ審査要件、(5) LIFF 固有セキュリティ(token 漏洩、postMessage、scope 過大、CSP) |
| 使用ツール | Read, Grep, WebFetch(LIFF 公式ドキュメント参照) |

**専門知識の柱**:
- LIFF SDK バージョン互換性
- LINE Developers コンソールの設定整合性(エンドポイント URL、scope、size)
- ミニアプリ審査ガイドライン(2024 以降)
- 国別/地域別の LINE 機能差(JP / TW / TH)

### 2.4 test-data-designer(テストデータ設計者)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Skill**(`.claude/skills/test-data-designer/SKILL.md`) |
| 起動条件 | 「テストデータ作って」「境界値出して」「ダミーユーザー」等、または test-planner からの委託 |
| 主責務 | 各テストケースが要求するデータパターンを設計 |
| 入力 | テストケース一覧(test-planner 出力) |
| 出力 | データ仕様(境界値・異常値表)、ダミーユーザー定義、LIFF 固有テストデータ |
| LIFF 固有データ | LINE userId(U + 32 桁 hex)、ID トークン(JWT 形式)、displayName(絵文字・サロゲートペア・RTL 文字)、target OS パターン、LIFF browser サイズ |
| 使用ツール | Read, Grep |

**設計時の観点**:
- 同値クラス分割
- 境界値(off-by-one、最大長、空)
- 多言語(JP/EN/TH/中国語繁体・簡体・絵文字・RTL)
- 個人情報のダミー化原則(本物の userId を使わない)

### 2.5 automation-engineer(自動化エンジニア)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Subagent**(`.claude/agents/automation-engineer.md`) |
| 起動条件 | system-architect または tester から呼び出し。または「自動テスト書いて」等 |
| 主責務 | クリティカルパスの E2E / コンポーネントテストを Playwright/Vitest 等で実装 |
| 入力 | テストケース ID + 期待結果(test-planner 出力)、LIFF モックの方針 |
| 出力 | テストコード、CI ワークフロー(GitHub Actions 想定)、実行レポート形式 |
| LIFF 固有 | `@line/liff-mock` または `liff` オブジェクトのスタブで LINE 内環境を再現 |
| 使用ツール | Read, Edit, Write, Bash(npm, npx playwright 等), Grep |
| Subagent 化理由 | (1) 大量のコード生成・実行で親コンテキストを汚さない、(2) テストランナーを並列実行できる、(3) 失敗時のリトライを独立タスクで処理 |

### 2.6 tester(テスター)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Subagent**(`.claude/agents/tester.md`) |
| 起動条件 | system-architect から呼び出し。または「テスト実施して」「ケース流して」 |
| 主責務 | テストケースを実行し、結果を Excel に記録。失敗時は bug-triager に引き継ぎ |
| 動作モード | (a) 自動: automation-engineer の生成コードを実行、(b) 手動: ケースごとに手順を提示しユーザーに実行依頼 → 報告を整形記録、(c) ハイブリッド: ケースごとに手段選択 |
| 入力 | テストケース一覧、自動テストコード、対象 URL/環境 |
| 出力 | 実行結果(Pass/Fail/Block/Skip)、エビデンス(スクリーンショット、ログ、HTTP レスポンス)、失敗ケースのバグレポート下書き |
| 使用ツール | Read, Bash(テストランナー実行), Edit(結果ファイル更新), Grep |
| Subagent 化理由 | 実行ログが膨大になりがちで親コンテキストを保護したい。複数ケースを並列実行する余地もある |

### 2.7 bug-triager(バグトリアージ / 障害管理者)

| 項目 | 内容 |
|---|---|
| 実装形態 | **Skill**(`.claude/skills/bug-triager/SKILL.md`) |
| 起動条件 | tester からの失敗報告を受けて自動起動、または「不具合まとめて」「issue 化して」 |
| 主責務 | 不具合の重大度・優先度判定、再現手順の整形、GitHub issue 起票案、回帰防止チェック |
| 入力 | tester の失敗ケース、エビデンス |
| 出力 | **同一 Excel(Bug report シート)** へのバグエントリ追記。タイトル・概要・再現手順・期待/実際・環境・優先度・関連ケース ID を埋める |
| 優先度モデル | Severity(S1〜S4)× Priority(P0〜P3)。LIFF 起動不能や PII 漏洩は即 S1/P0 |
| 使用ツール | Read, Edit(Excel 操作ライブラリ経由), Grep |
| GitHub issue 連携 | **当面なし**(Phase 2 時点では Excel 一元管理。将来要望が出たら追加検討) |

---

## 3. ワークフロー

### 3.1 デフォルト: アーキテクト主導パイプライン

```
ユーザー要望
   ↓
[system-architect]
   ├─ ① 計画フェーズ
   │   ├→ test-planner(ケース起票)
   │   └→ liff-reviewer(LIFF 観点重ね)
   ├─ ② データ整備フェーズ
   │   └→ test-data-designer(ケースに紐付くデータ生成)
   ├─ ③ 実装フェーズ(必要時)
   │   └→ automation-engineer [Subagent](自動テストコード)
   ├─ ④ 実施フェーズ
   │   └→ tester [Subagent](実行・結果記録)
   ├─ ⑤ 障害管理フェーズ
   │   └→ bug-triager(失敗ケース → バグレポート)
   └─ ⑥ 統合レビュー
       └→ ユーザー報告(Go/No-Go 判断材料込み)
```

**フェーズ間ゲート**: 各フェーズ終了時に system-architect が成果物の整合性を点検し、次フェーズへ進めるか判断する(過去フェーズへの戻しもあり)。

### 3.2 個別起動モード

ユーザーが特定ロールに直接依頼するパターンも全ロール対応。例:
- 「ケースだけ作って」→ test-planner 単独
- 「この既存ケースに LIFF 観点足して」→ liff-reviewer 単独
- 「この自動テストだけ流して」→ tester 単独(automation-engineer の生成物前提)

**注意**: 個別起動時は前後ロールの引継ぎが省略されるため、データ整合性はユーザー責任。

### 3.3 ロール間データ受け渡し形式

**Excel 1 ファイル一元管理**:`templates/test-book-template-ja.xlsx` をプロジェクトごとにコピーし、`テストケース` / `バグレポート` の 2 シートで全工程を完結させる。スキーマ詳細は `@docs/excel-schema.md`。

| 受け渡し | 形式 | 場所 | 列(該当シート) |
|---|---|---|---|
| 計画 → ケース | Excel `テストケース` シート | `testing-framework/projects/<project>/test-book.xlsx` | A〜H 列(`シナリオ#` 〜 `参考資料`)|
| ケース → データ | Markdown / YAML(補助) | `testing-framework/projects/<project>/data.md` | — |
| ケース → 自動コード | TypeScript/JavaScript | `testing-framework/projects/<project>/automation/` | — |
| 実施 → 結果 | 同 Excel の `テストケース` シートに追記 | (同上) | I〜M 列(`担当者1` 〜 `コメント`)|
| 失敗 → バグ | 同 Excel `バグレポート` シートに追記 | (同上) | A〜H 列全列 |
| 相互参照 | テストケース `コメント` 列に Bug ID を書き込み | (同上) | `テストケース.コメント` ←→ `バグレポート.バグID` |

**プロジェクト単位の隔離**: 汎用フレームワークのため `projects/<project-name>/` 配下に各案件を閉じ込める。横断比較が必要な集計は将来検討。

---

## 4. ディレクトリ構造(設計時点・実装で確定)

```
testing-framework/
├─ CLAUDE.md                        # サブプロジェクト指針
├─ docs/
│  ├─ design.md                     # 本ファイル
│  ├─ liff-checklist.md             # (後続) LIFF レビュー観点詳細
│  ├─ excel-schema.md               # (後続) ユーザー提供テンプレートの列定義(英→日訳含む)
│  └─ workflow-examples.md          # (後続) ワークフロー実例
├─ .claude/
│  ├─ skills/
│  │  ├─ system-architect/SKILL.md
│  │  ├─ test-planner/SKILL.md
│  │  ├─ liff-reviewer/SKILL.md
│  │  ├─ test-data-designer/SKILL.md
│  │  └─ bug-triager/SKILL.md
│  └─ agents/
│     ├─ automation-engineer.md
│     └─ tester.md
├─ templates/
│  ├─ test-book-template.xlsx       # ★ ユーザー提供原本(英語)
│  └─ test-book-template-ja.xlsx    # 日本語化版(運用時に使用)
├─ projects/                        # プロジェクト別作業領域
│  └─ <project-name>/
│     ├─ test-book.xlsx             # ケース + 結果 + バグを一元管理(git 管理)
│     ├─ data.md                    # テストデータ補助メモ(git 管理)
│     ├─ automation/                # 自動テストコード Playwright/Vitest(git 管理)
│     └─ specs/                     # 仕様書・ワイヤーフレーム ★ gitignored
└─ .github/
   └─ workflows/                    # GitHub Actions(自動テスト CI)
```

**注**: `.claude/skills/` と `.claude/agents/` を testing-framework サブディレクトリに置くか、リポジトリルートの `.claude/` に併設するかは実装段階で再確認(下記「残課題」参照)。

---

## 5. 実装フェーズ計画

```
Phase 1: 設計確定                    ← 現在
   - 本ドキュメントレビュー
   - Excel テンプレート受領
   - 残課題の解消
       ↓
Phase 2: コアロール 3 つ実装
   - system-architect(SKILL)
   - test-planner(SKILL)
   - liff-reviewer(SKILL)
   → 「ケースを起こす」までが完結する最小構成
       ↓
Phase 3: データ・実施ロール実装
   - test-data-designer(SKILL)
   - tester(Subagent)
   - bug-triager(SKILL)
   → 手動テスト実行が完結
       ↓
Phase 4: 自動化レイヤ
   - automation-engineer(Subagent)
   - CI 連携
       ↓
Phase 5: 実プロジェクト適用
   - 本番 LIFF アプリで run、設計を改善
```

**60% 完成度で回転**: Phase 2 完了時点で実プロジェクトに当てて、出てきた課題で Phase 3 以降を補正する。最初から完璧を目指さない。

---

## 6. 残課題・確認事項

### 6.1 ユーザー回答済み(2026-04-28)

- [x] **Excel テンプレート**: `Test cases & Bug report.xlsx`(英語原本)を提供。**運用時は日本語化**して使用 → `templates/test-book-template-ja.xlsx` を作成する
  - サンドボックス環境からは Windows ローカルパスを直接読めないため、ファイル中身の取り込みは別途必要(下記 6.2 参照)
- [x] **対象 LIFF アプリ**: **汎用フレームワーク**(複数アプリで使い回す)。`projects/<project-name>/` 配下にプロジェクトを隔離
- [x] **GitHub issue 連携**: **当面なし**。バグレポートは Excel の `Bug report` シートに記載
- [x] **CI 環境**: **GitHub Actions** 採用。`.github/workflows/` 配下にワークフロー定義

### 6.2 実装着手前に必要なアクション

- [x] **Excel テンプレートの取り込み**: GitHub Web UI 経由で `templates/Test cases & Bug report.xlsx`(原本・英語、約 15 MB)をコミット済み
- [x] **英→日 列対訳の確定**: `docs/excel-schema.md` に確定済み
- [x] **日本語版テンプレ生成**: `templates/test-book-template-ja.xlsx`(空テンプレ、データ検証付き、約 12 KB)生成済み

### 6.3 設計内で要決定(実装段階で再確認)

- [ ] **`.claude/` 配置場所**: testing-framework 配下に閉じるか、ルート `.claude/` に併設するか
  - サブディレクトリ案: 水泳プロジェクトと完全分離、cd して起動するイメージ
  - ルート併設案: 一つの Claude セッションから両ドメインに行き来できるが、自動起動条件で混線リスク
- [ ] **Excel 操作の実装手段**: Python(`openpyxl`)/Node(`exceljs`) などライブラリ選定。tester / test-planner / bug-triager で Excel 読み書きが必要
- [ ] **LIFF モック戦略**: `@line/liff-mock` 利用か独自モックか。automation-engineer の前提に影響

### 6.3 将来検討事項

- [ ] テストケースの版管理(機能改修時のケース更新フロー)
- [ ] 過去実行結果の集計・トレンド分析(performance-analyst 相当のレイヤを将来追加するか)
- [ ] 多言語(JP / EN)テストケースの併用フォーマット

---

## 7. 設計上のトレードオフ

| 論点 | 選んだ案 | 採用しなかった案 | 理由 |
|---|---|---|---|
| Skill vs Subagent | 役割で使い分け | 全ロール Skill / 全 Subagent | 計画系は親コンテキスト共有が有利、実行系は分離が有利 |
| セキュリティ専任ロール | liff-reviewer に内包 | 独立 security-reviewer | LIFF/LINE 固有のセキュリティが大半で、専門家を分けると重複が発生 |
| アクセシビリティ・パフォーマンス | test-planner に内包 | 独立ロール | 一般 Web テスト計画と密結合のため。深掘り必要時は将来分離 |
| 成果物形式 | Excel 中心 + Markdown 補助 | Markdown のみ / JSON のみ | ユーザー指定。レビュー性と機械可読性のバランスは Excel の cell 単位で確保 |
| アーキテクト主導 vs 個別起動 | 両対応 | パイプライン強制 | 反復作業時は個別起動の方が早い。ユーザー裁量で切替可能 |

---

## 8. このドキュメントの位置付け

本ドキュメントは **設計フェーズの成果物**。v0.3 時点で実装の前提はすべて整っている:

1. ✅ ユーザーがロール構成・ワークフローに合意
2. ✅ Excel テンプレ取り込み・スキーマ確定・日本語版生成
3. ✅ 4 つの方針確定済み(汎用化、Excel 一元、GitHub issue なし、GitHub Actions)

**Phase 2 着手中**: コアロール 3 つ(system-architect / test-planner / liff-reviewer)を実装。
