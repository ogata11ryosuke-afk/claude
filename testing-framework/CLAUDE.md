# CLAUDE.md — Testing Framework (LINE LIFF / ミニアプリ)

## このサブプロジェクトの位置付け

このディレクトリ配下は、ルート(`/home/user/claude/CLAUDE.md` のマスターズ水泳支援チーム)とは**完全に独立**した文脈で動作する。LINE LIFF / ミニアプリのテスト計画策定からテスト実施までを 7 ロール構成で支援するフレームワーク。

**重要**: このサブディレクトリ配下で作業する Claude セッションは、ルートの水泳コーチ文脈を引き継がない。`@docs/design.md` を真実源として扱う。

## スコープ

- **対象システム**: LINE LIFF アプリ / LINE ミニアプリ(**汎用フレームワーク**: 複数案件で使い回す)
- **テスト範囲**: 機能テスト + LIFF 固有レビュー + セキュリティ(LIFF 観点) + 自動化 E2E + バグ管理
- **テスト実施形態**: ハイブリッド(クリティカルパスは自動化、探索的・UI 確認は手動)
- **成果物フォーマット**: **Excel 一元管理**(`Test cases & Bug report.xlsx` ベース、運用時は日本語化)+ Markdown 補助
- **CI**: GitHub Actions(`.github/workflows/`)
- **バグ管理**: 当面は Excel 内 `Bug report` シートに集約(GitHub issue 連携は将来検討)

## ロール構成(7 ロール)

| # | ロール | 実装形態 | 役割概要 |
|---|---|---|---|
| 1 | **system-architect** | Skill | 統括・全体管理・最終レビュー・他ロールのオーケストレーション |
| 2 | **test-planner** | Skill | テスト計画/テストケース策定(機能・アクセシビリティ・パフォーマンス含む) |
| 3 | **liff-reviewer** | Skill | LIFF SDK / ミニアプリ仕様 / LINE Login の専門レビュー(セキュリティ観点も内包) |
| 4 | **test-data-designer** | Skill | 境界値・異常値・LINE 固有テストデータ(userId/IDトークン/OS パターン)設計 |
| 5 | **automation-engineer** | Subagent | Playwright/Vitest 等の自動テストコード生成・CI 連携 |
| 6 | **tester** | Subagent | テストケース実行(自動・手動)・結果記録 |
| 7 | **bug-triager** | Skill | 不具合記録・優先度付け・再現手順テンプレ・GitHub issue 連携 |

詳細は `@docs/design.md` 参照。

## オーケストレーション方針

- **デフォルト**: system-architect 主導パイプライン(計画 → レビュー → 実装の 2 パス)
- **柔軟性**: ユーザーは個別ロールを直接起動することも可能

## 実行環境(重要)

このフレームワークの **日常運用は Claude Code Desktop(Windows)前提**。Claude Code Web/Remote(Linux サンドボックス)では仕様書・ワイヤーフレーム等のローカルファイルが読めないため、実プロジェクトでの利用には Desktop が必要。

| 用途 | 推奨環境 | 理由 |
|---|---|---|
| 日常のテスト計画・ケース策定・レビュー | **Claude Code Desktop** | 仕様書を絶対パスで読む必要がある |
| スキル本体の改修・設計議論 | どちらでも可 | git 管理の `.md` 編集のみ |
| Notion 連携(将来必要なら) | Desktop のみ | egress 制限のため |

### ファイルの置き場所ルール

| ファイル種別 | 置き場所 | git 管理 | 例 |
|---|---|---|---|
| スキル本体 | `testing-framework/.claude/` | ✅ あり | SKILL.md など |
| 設計ドキュメント | `testing-framework/docs/` | ✅ あり | design.md, excel-schema.md |
| Excel テンプレート(空) | `testing-framework/templates/` | ✅ あり | test-book-template-ja.xlsx |
| 案件のテストブック Excel | `testing-framework/projects/<name>/test-book.xlsx` | ✅ あり | 履歴を残したい |
| **仕様書・ワイヤーフレーム・画面キャプチャ** | `testing-framework/projects/<name>/specs/` | ❌ **gitignored** | Word, PDF, Figma エクスポート |
| 自動テストコード | `testing-framework/projects/<name>/automation/` | ✅ あり | Playwright/Vitest |
| 補助メモ | `testing-framework/projects/<name>/data.md` 等 | ✅ あり | テストデータ仕様 |

### 仕様書の渡し方(2 パターン)

**パターン 1: project 配下にローカルコピー**
```
C:\<repo>\testing-framework\projects\<案件名>\specs\
  ├─ requirements.docx
  ├─ wireframe.pdf
  └─ screen-flow.png
```
`specs/` は `.gitignore` で無視されるため push されない。Claude にはこの相対パスで参照させる。

**パターン 2: OneDrive/Desktop の絶対パスを直接指定**
ユーザーが Claude に対し:
> 「`C:\Users\rogata_w\OneDrive\...\spec.pdf` を読んで計画を作って」

と絶対パスを渡せば、Desktop の Claude は直接読める。コピー不要。Phase 2 のスキル群もこの形式に対応している。

## 残課題

設計フェーズの未確定項目は `@docs/design.md` の「残課題・確認事項」セクション参照。実装着手前にユーザー確認が必要。

## 更新履歴

- 2026-04-28 初版設計(`docs/design.md` v0.1)
- 2026-04-28 4 主要未決事項解消(汎用化・Excel 一元・GitHub issue なし・GitHub Actions)→ `docs/design.md` v0.2
