# Excel テンプレートスキーマ

`templates/Test cases & Bug report.xlsx`(英語原本)を解析した結果のスキーマ定義と、運用時の日本語訳。

**原本サイズ**: 約 15 MB(過去案件の埋め込み画像含む)
**シート数**: 2(`Test Cases`, `Bug Report`)
**用途**: 過去案件 [PRD]Hilton Intutive Survey の実テンプレ。LIFF 案件の汎用テンプレに転用する

---

## 1. シート: `Test Cases` → 日本語: `テストケース`

### レイアウト

- **Row 1**: プロジェクト/環境メタデータ(自由記述)
  - 原本: `Project Name: [PRD]Hilton Intutive Survey\nEnvironment: https...`
  - 運用時: 案件ごとに上書き
- **Row 2**: ヘッダー行
- **Row 3 以降**: テストケース本体

### 列定義(A〜M を使用、N〜T は空)

| 列 | 英語ヘッダー | 日本語ヘッダー | 型/値 | 必須 | 役割 |
|---|---|---|---|---|---|
| A | `#` | `シナリオ#` | 整数(同一シナリオ内ではブランク継承) | 任意 | シナリオの大番号。同一シナリオの 2 件目以降は空欄 |
| B | `CASE#` | `ケース#` | 文字列 `M.N` 形式 | **必須** | テストケース ID(例: `1.1`, `2.3`) |
| C | `SCENARIO` | `シナリオ` | 文字列 | **必須**(初出のみ) | シナリオ名。同一 `#` 内では初出のみ記載 |
| D | `PRE-REQUISITES` | `前提条件` | 文字列(複数行可) | 任意 | 実行前の状態・設定 |
| E | `CASE DESCRIPTION` | `ケース内容` | 文字列 | **必須** | 検証目的(`Verify that ...`) |
| F | `REPRODUCTION STEPS` | `手順` | 文字列(複数行可) | **必須** | 操作手順(箇条書き推奨) |
| G | `EXPECTED RESULTS` | `期待結果` | 文字列 | **必須** | 期待動作 |
| H | `REFERENCE` | `参考資料` | URL/文字列 | 任意 | 仕様書リンク・図番号 |
| I | `TESTER 1` | `担当者1` | 人物名(`@username` 形式) | **必須** | 1 人目のアサイン |
| J | `TESTER 2` | `担当者2` | 人物名(`@username` 形式) | 任意 | 2 人目(クロスチェック用) |
| K | `RESULT TESTER 1` | `結果1` | 列挙(下記) | 実施後**必須** | 担当者 1 の実行結果 |
| L | `RESULT TESTER 2` | `結果2` | 列挙(下記) | 担当者 2 がいる場合**必須** | 担当者 2 の実行結果 |
| M | `COMMENTS` | `コメント` | 文字列(複数行可) | 任意 | 備考・関連バグ ID |

### 結果(`RESULT TESTER N`)の列挙値

| 値 | 意味 |
|---|---|
| `Pass` | 期待通り動作 |
| `Fail` | 不具合検出(→ Bug Report に記録、`COMMENTS` に Bug ID を相互参照) |
| `SKIP` | 意図的にスキップ(対象外/環境制約等。理由を `COMMENTS` に記載) |
| `Not tested` | 未実施 |

---

## 2. シート: `Bug Report` → 日本語: `バグレポート`

### レイアウト

- **Row 1**: ヘッダー行
- **Row 2 以降**: 不具合本体

### 列定義(A〜H を使用、I〜T は空)

| 列 | 英語ヘッダー | 日本語ヘッダー | 型/値 | 必須 | 役割 |
|---|---|---|---|---|---|
| A | `Bug ID` | `バグID` | 整数(連番) | **必須** | 通し番号 |
| B | `Issue` | `内容` | 文字列(複数行可) | **必須** | 事象の要約 + 期待 vs 実際 |
| C | `Attachment #1` | `添付1` | セル内画像 / リンク | 任意 | スクリーンショット |
| D | `Attachment #2` | `添付2` | セル内画像 / リンク | 任意 | 補助資料 |
| E | `Comments` | `コメント` | 文字列(複数行可) | 任意 | やり取り・修正対応の経緯(`@担当者`)|
| F | `Status` | `ステータス` | 列挙(下記) | **必須** | 対応状況 |
| G | `Priority` | `優先度` | 列挙(下記) | **必須** | 重要度 |
| H | `Issue Type` | `種別` | 列挙(下記、複数選択は `,` 区切り) | **必須** | 不具合カテゴリ |

### 列挙値

#### Status(ステータス)

| 値 | 意味 |
|---|---|
| `Reported` | 報告済(未対応) |
| `Test` | 修正完了・再テスト待ち |
| `Done` | 完了(再テスト Pass) |
| `Won't Fix` | 対応見送り(理由を `Comments` に記載) |

#### Priority(優先度)

| 値 | 意味 |
|---|---|
| `High` | 高(機能不全・主要画面崩れ・データ不整合) |
| `Medium` | 中(部分的な UI 崩れ・軽微な使い勝手) |
| `Low` | 低(オプション)。原本での実例は無いが運用時に追加可 |

#### Issue Type(種別)

| 値 | 意味 |
|---|---|
| `Functionality` | 機能不具合(動作不正、エラー、データ) |
| `UI / UX` | 表示崩れ・UX 課題 |
| `UI / UX, Functionality` | 両方該当(複数選択時はカンマ区切り) |

LIFF 案件で追加を推奨する種別(本テンプレ拡張案):
- `LIFF SDK`(SDK 初期化、`liff.*` API 関連)
- `LINE Login / OIDC`(認可・ID トークン関連)
- `Compatibility`(LINE バージョン・OS 互換性)
- `Performance`(初期ロード・操作レスポンス)
- `Security`(token 漏洩、XSS、CSRF など)
- `Accessibility`(WCAG 違反、a11y)

---

## 3. 運用ルール(原本観察から導出)

1. **シナリオグルーピング**: `#` 列に大番号、`CASE#` 列に `大.小` 形式の ID。同一シナリオの 2 件目以降は `#` と `SCENARIO` を空欄にして視覚的にグループ化
2. **担当者表記**: `@FirstName LastName` 形式(Notion / Slack のメンション風)
3. **結果記録のタイミング**: `RESULT` 列はテスト実施前は空欄。実施直後に Pass/Fail/SKIP/Not tested を入力
4. **Fail 時の連動**: `RESULT` を `Fail` にしたら、Bug Report シートにバグを起票し、Test Cases の `COMMENTS` 列に Bug ID を書いてリンク
5. **Bug Report の Comments 文化**: 修正担当者と報告者の対話ログを Comments 列に積む(`@Daniel 04/02` のように日付付きメンション)

---

## 4. ロール別の Excel 操作責務

| ロール | 読む | 書く |
|---|---|---|
| system-architect | 両シート全て | なし(レビューのみ) |
| test-planner | Test Cases | Test Cases(A〜H + COMMENTS) |
| liff-reviewer | Test Cases | Test Cases に追記(LIFF 観点ケース) + COMMENTS に LIFF 警告 |
| test-data-designer | Test Cases | なし(別ファイルにデータ仕様) |
| automation-engineer | Test Cases | なし(コード生成は別ファイル) |
| tester | Test Cases | Test Cases の I〜M 列(担当者・結果・コメント) |
| bug-triager | Bug Report + Test Cases | Bug Report 全列 + Test Cases の COMMENTS(Bug ID 相互参照) |

---

## 5. 日本語版テンプレート(`test-book-template-ja.xlsx`)の生成仕様

原本から以下を変換した日本語版を `templates/test-book-template-ja.xlsx` として配置する:

1. **ヘッダー行を日本語化**(上記の対訳表に従う)
2. **過去案件のサンプル行を全削除**(空テンプレートにする)
3. **Row 1 のメタデータを汎用テンプレに置換**:
   - 原本: `Project Name: [PRD]Hilton Intutive Survey\nEnvironment: https://...`
   - 運用版: `プロジェクト名: <案件名を記入>\n環境: <検証環境URLを記入>`
4. **データ検証(Data Validation)を追加**:
   - `結果1` / `結果2` 列: `Pass / Fail / SKIP / Not tested` のドロップダウン
   - バグレポート `ステータス` 列: `Reported / Test / Done / Won't Fix`
   - バグレポート `優先度` 列: `High / Medium / Low`
   - バグレポート `種別` 列: 上記拡張案を含む
5. **列幅・スタイル**: 原本の見た目を踏襲(ヘッダー太字、罫線、結果列の色分け)。完全再現は次フェーズに先送り可

---

## 6. 既知の課題

- **原本ファイルサイズが 15 MB**: 過去案件の添付画像が大量に埋め込まれている。汎用テンプレ化時に画像は全削除して数十 KB に収める
- **行数**: 原本は最大 200+ 行確保されている。汎用テンプレでは 100 行程度に縮小して再利用しやすくする
- **シート間リンク**: 原本では Test Cases の COMMENTS と Bug Report の Bug ID が手動相互参照。将来は数式または条件付き書式で自動化検討
