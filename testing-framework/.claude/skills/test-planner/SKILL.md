---
name: test-planner
description: 一般 Web システムの知識ベースで、機能・非機能テスト計画と Excel テストケースを策定する。「テスト計画作って」「テスト観点出して」「ケース起こして」「ISO 25010 で観点整理」「アクセシビリティチェック」「パフォーマンステスト計画」などで起動する。`testing-framework/templates/test-book-template-ja.xlsx` をベースに `projects/<project>/test-book.xlsx` を生成し、`テストケース` シートの A〜H 列を埋める。LIFF 固有観点は liff-reviewer に重ねを依頼、データパターンは test-data-designer に依頼する責務分離を守る。
allowed-tools: Read, Grep, Edit, Write, Bash, WebFetch
---

# Test Planner — テスト計画・ケース策定スキル

## 役割

機能仕様を読み解き、**一般 Web システムの観点でテスト計画とテストケースを策定**する。LIFF 固有観点は重ねるため liff-reviewer の仕事。データパターン詳細は test-data-designer の仕事。自分は「アプリケーションそのものの正しさ」を網羅する。

## 絶対遵守ルール

1. **Excel スキーマを必ず先に確認**: `@docs/excel-schema.md` を Read してから書き込む。列番号・列挙値を間違えない
2. **テンプレ起点**: `templates/test-book-template-ja.xlsx` を **コピー**して使う。原本を直接編集しない
3. **責務分離**: LIFF 固有観点は自分で書かない(liff-reviewer に明示的に引き継ぐ)
4. **イシュー起点**: 「全網羅」を最初から目指さない。リスクベースで P0〜P3 を振る
5. **行追加は openpyxl で**: 手動で Excel を Write しない。Python スクリプトで列指定追記する

## カバレッジモデル

### ISO/IEC 25010 ベースの観点リスト

| 特性 | サブ特性 | 主なテスト観点 |
|---|---|---|
| **機能適合性** | 機能完全性・正確性・適切性 | 仕様通りの動作、エッジケース、異常系 |
| **性能効率性** | 時間効率・資源効率・容量 | レスポンス時間、メモリ、同時接続 |
| **互換性** | 共存性・相互運用性 | ブラウザ・OS・LINE バージョン |
| **使用性** | 適切性認識・習得性・運用操作性・ユーザーエラー防止・UI 美的・アクセシビリティ | UI 一貫性、エラーメッセージ、WCAG 2.2 AA |
| **信頼性** | 成熟性・可用性・障害許容性・回復性 | 異常系、ネットワーク断、リトライ |
| **セキュリティ** | 機密性・インテグリティ・否認防止・責任追跡性・真正性 | 認証・認可、XSS、CSRF、token 管理 |
| **保守性** | (テストでは扱わない) | — |
| **移植性** | 適応性・設置性・置換性 | 他環境動作、データ移行 |

LIFF 固有のセキュリティ・互換性は **liff-reviewer** に引き継ぐが、一般 Web レベル(XSS・CSRF・パスワード・セッション)は test-planner の責務。

### リスクベース優先度

| 優先度 | 基準 | 対応 |
|---|---|---|
| **P0** | クリティカルパス。失敗で即サービス停止 / データ消失 | 必ず実施・自動化推奨 |
| **P1** | 主要機能。失敗で主要ユーザー体験が損なわれる | 必ず実施 |
| **P2** | 補助機能。失敗でも回避可能 | 時間あれば実施 |
| **P3** | エッジケース・希少パス | 余力で実施 |

(優先度は `コメント` 列に `[P0]` 等で書き込む。Excel に専用列が無いため)

### ケース粒度技法

設計するテスト技法を意識的に選ぶ:

- **同値分割**: 入力域を等価クラスに分け、各クラス代表 1 件
- **境界値分析**: クラスの境界 ±1
- **状態遷移**: 画面遷移・状態マシンを網羅
- **デシジョンテーブル**: 条件組合せ
- **原因結果グラフ / ペアワイズ**: 多パラメータ
- **ユースケース駆動**: 主シナリオ + 代替フロー + 例外フロー

## 標準ワークフロー

### Step 1: 案件理解

1. ユーザーから渡された仕様(URL/PDF/口頭)を Read / WebFetch
2. 主要画面・主要操作・主要 API を整理
3. **不明点リスト**を作る(仕様書から読めなかった事項) → ユーザーに確認

### Step 2: シナリオ抽出

```
シナリオ #1: <主機能Aの正常系>
シナリオ #2: <主機能Aの異常系>
シナリオ #3: <主機能Bの正常系>
...
シナリオ #N: <セキュリティ系一括>
```

### Step 3: 各シナリオのケース展開

シナリオごとに `ケース#` を `<シナリオ番号>.<連番>` 形式で振る:
- `1.1`, `1.2`, `1.3` ... → シナリオ 1 のケース群

ケース粒度技法を意識的に選んで展開。

### Step 4: Excel 書き込み

Python (openpyxl) で `projects/<project>/test-book.xlsx` を生成する。最初の起動なら `templates/test-book-template-ja.xlsx` をコピーしてから追記。

```python
import shutil, openpyxl
src = "testing-framework/templates/test-book-template-ja.xlsx"
dst = "testing-framework/projects/<project>/test-book.xlsx"
import os
os.makedirs(os.path.dirname(dst), exist_ok=True)
if not os.path.exists(dst):
    shutil.copy(src, dst)

wb = openpyxl.load_workbook(dst)
ws = wb["テストケース"]

# Row 1 メタデータ更新(初回のみ)
ws["A1"] = "プロジェクト名: <案件名>\n環境: <URL>"

# 追記開始行を探す(空の最初の行)
def next_row(ws, start=3):
    r = start
    while ws.cell(row=r, column=2).value:  # B列(ケース#)で判定
        r += 1
    return r

r = next_row(ws)
ws.cell(row=r, column=1, value=1)              # シナリオ#
ws.cell(row=r, column=2, value="1.1")          # ケース#
ws.cell(row=r, column=3, value="シナリオ名")    # シナリオ
ws.cell(row=r, column=4, value="前提条件")      # 前提条件
ws.cell(row=r, column=5, value="ケース内容")    # ケース内容
ws.cell(row=r, column=6, value="手順1\n手順2") # 手順
ws.cell(row=r, column=7, value="期待結果")      # 期待結果
ws.cell(row=r, column=8, value="参考URL")       # 参考資料
ws.cell(row=r, column=13, value="[P1]")        # コメント列に優先度
wb.save(dst)
```

**列番号の対応**(誤りやすい):
| 列 | 番号 | 名前 |
|---|---|---|
| A | 1 | シナリオ# |
| B | 2 | ケース# |
| C | 3 | シナリオ |
| D | 4 | 前提条件 |
| E | 5 | ケース内容 |
| F | 6 | 手順 |
| G | 7 | 期待結果 |
| H | 8 | 参考資料 |
| I | 9 | 担当者1(test-planner は触らない) |
| J | 10 | 担当者2(test-planner は触らない) |
| K | 11 | 結果1(test-planner は触らない) |
| L | 12 | 結果2(test-planner は触らない) |
| M | 13 | コメント |

### Step 5: 引継ぎ

- **liff-reviewer** へ: 「LIFF 固有観点を重ねて末尾に追記してください。シナリオ番号は <最終番号+1> から」と明示
- **test-data-designer** へ: 「以下のケースで境界値・異常値データが必要」とリスト渡し

## アクセシビリティ・パフォーマンス観点の組み込み

専任ロールを置かず test-planner に内包する方針(設計合意済み)。

### アクセシビリティ(WCAG 2.2 AA 準拠ベース)

最低限カバーする観点:
- キーボード操作可能性(Tab・Enter・Esc)
- フォーカス可視
- コントラスト比 4.5:1 以上(通常テキスト)、3:1(大テキスト)
- 画像 alt 属性
- フォームラベル関連付け
- スクリーンリーダー読み上げ(VoiceOver / TalkBack)
- 拡大表示(200%)で破綻しない

### パフォーマンス

- 初期ロード(LCP < 2.5s)
- 操作レスポンス(INP < 200ms)
- メモリリーク(長時間操作後)
- ネットワーク遅延時のフィードバック(loading 表示)

LIFF 固有のパフォーマンス(`liff.init` 後のレンダリング遅延等)は liff-reviewer 側。

## 出力フォーマット

### ユーザーへの中間報告

```markdown
## テスト計画作成完了

### 計画スコープ
- 対象: <機能リスト>
- 除外: <意図的に外した範囲とその理由>

### シナリオ・ケース統計
- シナリオ数: N
- ケース総数: M
  - P0: a / P1: b / P2: c / P3: d
- 観点別内訳: 機能 X件 / 性能 Y件 / 互換 Z件 / 使用性 W件 / 信頼性 V件 / セキュリティ U件

### 出力先
- `projects/<project>/test-book.xlsx` の `テストケース` シート

### 次の引継ぎ
- liff-reviewer に LIFF 観点重ねを依頼予定
- test-data-designer にデータ設計を依頼予定

### 確認事項
- <仕様の不明点>
```

## 関連ドキュメント

- `@docs/design.md` §2.2: test-planner 役割定義
- `@docs/excel-schema.md`: テストケースシートの列定義
