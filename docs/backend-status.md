# Backend Status Matrix — 真実源

「**どの環境から / どのバックエンドへ書けるか**」のマトリクス。
**ここを参照せずに「書けない」と発言してはならない**(CLAUDE.md 絶対ルール 11)。

最終更新: 2026-05-11

---

## マトリクス

| バックエンド | Claude Code Remote(サンドボックス・このセッション) | Windows Claude Code Desktop |
|---|---|---|
| **Google Sheets**(`docs.google.com`) | ✅ **書込可**(Python + service account / gsheets MCP) | ✅ 書込可 |
| **Google Drive**(`googleapis.com`) | ✅ 書込可 | ✅ 書込可 |
| **Notion**(`api.notion.com`) | ❌ **egress 遮断**(`Host not in allowlist`) | ✅ Notion MCP / Notion Connector |
| **GitHub**(`api.github.com` / git push) | ✅ 書込可(MCP + git push) | ✅ 書込可 |
| **ローカルファイル**(`/home/user/claude/`) | ✅ 書込可 | ✅ 書込可 |

---

## サンドボックス用 認証情報の場所

- **Google service account JSON**: `/home/user/.secrets/masters-swim-service-account.json`
- **環境変数**: `SERVICE_ACCOUNT_PATH` / `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE`(`.claude/settings.local.json`)
- **Notion トークン**: サンドボックスからは使えないため格納しない方針(Windows Connector 側で管理)

## 利用するべき書込手段(優先度順)

### このセッション(Claude Code Remote)からの書込

1. **Google Sheets**(現行・実証済)
   - 対象: `1cEP2_xigk6T_co866SACkp2ER4l3lKtqp4IjRS6bUv8`(Masters Swim Log)
   - 主要タブ: `weekly-menu` / `sessions` / `season-plan` / `injury-log` / `measurements` / `weekly-review` / `monthly-review`
   - 手段: Python(google-api-python-client)or gsheets MCP

2. **git リポ**(プラン草案・スキル更新・履歴メモ)
   - 真実源として残す価値があるもの: `weeks/YYYY-Www-plan.md` / `docs/*.md` / `.claude/skills/**/SKILL.md`
   - 手段: Write + git commit + git push

3. **ローカル一時ファイル**(中継・キャッシュ)
   - JSON データ等を一時保存して他処理に渡す

### Windows セッションでの書込

1. **Notion**(主用途)
   - Training DB / Reviews DB / Injury Log DB / Memo Summary
   - 手段: Notion Connector(`notion-search` / `notion-create-pages` 等)

2. それ以外は Claude Code Remote と同じ

---

## 「書けない」と言う前のチェックリスト

CLAUDE.md 絶対ルール 11 の再掲・運用版:

```
[ ] 対象バックエンドへの egress / MCP を 実際にテストした?
    - Sheets: gsheets MCP の list_spreadsheets 1回 / または curl
    - Notion: curl https://api.notion.com
    - GitHub: git ls-remote / mcp__github__*
[ ] 直接の対象が書けない場合、代替バックエンドが書けるか確認した?
    - 例: Notion 書けない → Sheets 書ける? → 書ける!
[ ] ユーザーに作業要求(コピペ等)する前に、3つの代替手段を検討した?
[ ] 単に「最近やっていない」ことを「できない」と混同していないか?
```

**いずれも No / 不明** の場合、「できない」発言は禁止。

---

## 状態変化の追跡

| 日付 | 変化 |
|---|---|
| 2026-04-24 | サンドボックスから `api.notion.com` が 403 (`Host not in allowlist`) であることを確認 |
| 2026-04-24 | gsheets MCP が一時切断、Python 直接ルートに変更で継続 |
| 2026-05-11 | gsheets MCP 再接続。Sheets 書込ルートは Python / MCP 両方利用可能 |

新たな状態変化があれば即時このファイルに追記する。

---

## 違反事例(再発防止のための記録)

### 2026-05-11 case

- **状況**: 今週(5/11〜5/17)のプラン生成依頼を受けた
- **すべきこと**: Sheets `weekly-menu` に 7行追記して完了報告
- **したこと**: チャット出力のみで完了とした。ユーザーから「Notion 書込忘れてない?」と指摘を受け、初動で「Notion はサンドボックスから書けない」とだけ返答。Sheets が書込可能であることを完全に失念
- **原因**: Notion 移行を「Sheets を捨てた」と誤解。データ永続化義務を本ファイル化していなかったため毎回その場判断で対応
- **対策(本ファイル)**: 絶対ルール 10・11 追加、本マトリクス作成、training-planner ワークフローに persist ステップ必須化

---

## 更新履歴

- 2026-05-11 初版作成(2026-05-11 case を受けて)
