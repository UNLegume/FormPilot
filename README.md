# FormPilot

Spreadsheet の企業リストからお問い合わせフォームへ自動入力・送信する Claude Code スキル。

---

## 特徴

| カテゴリ | 機能 |
|---|---|
| **入力** | LLM による意味的フィールドマッチング（表記ゆれ自動対応）、確認画面の自動処理 |
| **学習** | 未知フィールドを質問→記録→次回自動流用、3回以上出現で config 昇格提案 |
| **耐障害** | 最大3回リトライ、reCAPTCHA v2/v3 対応、3回失敗で自動スキップルール化 |
| **運用** | 最大150社バッチ処理（中断再開対応）、進捗記録で中断再開、フォーム種別に応じたスクリーンショット |

---

## 必要環境

| 必要なもの | 説明 |
|---|---|
| Claude Code | スキルの実行環境 |
| Playwright MCP サーバー | ブラウザ操作に使用（`mcp__playwright__*` ツール群） |
| Google Spreadsheet | 企業名・送信済みフラグ・フォームURLを管理するリスト |

---

## セットアップ

1. リポジトリをクローンし、スキルのシンボリックリンクを作成:

   ```bash
   git clone https://github.com/finn-inc/FormPilot.git
   ln -s "$(pwd)/FormPilot" ~/.claude/skills/form-submit
   ```

2. Playwright MCP が `~/.claude.json` または `~/.claude/settings.json` に設定されていることを確認

3. `/form-submit` を実行 — config.json が未設定の場合、対話形式で自動生成される

スキーマの詳細は [`references/schemas.md`](references/schemas.md)、設定例は [`references/config-examples.md`](references/config-examples.md) を参照。

### Chrome を閉じてからスキルを実行

Playwright MCP は既存の Chrome プロファイルを再利用して reCAPTCHA の信頼スコアを確保している。Chrome と Playwright が同じプロファイルを同時使用できないため、**スキル実行前に Google Chrome を完全に閉じること**。

---

## 使い方

```
/form-submit                       未送信企業を最大150社まで連続処理（デフォルト）
/form-submit --max 10              最大10社まで処理（config の maxCompanies を上書き）
/form-submit --company <企業名>     指定した企業のみ処理
```

---

## ディレクトリ構成

```
FormPilot/
├── SKILL.md                  # スキル定義（全17ステップ）
├── docs/
│   ├── progress.json         # 前回の続きから再開するための進捗記録
│   ├── error-log.json        # セッション横断の失敗履歴
│   └── error-rules.json      # 3回以上失敗した企業の自動スキップルール
└── references/
    ├── schemas.md            # 全 JSON スキーマ定義
    └── config-examples.md    # サンプルデータ・設定例
```

実行時に使用する設定ファイルは `~/.claude/form-submit/` に配置する（リポジトリ外）。

```
~/.claude/form-submit/
├── config.json               # スプレッドシートURL・commonData・オプション
└── field-log.json            # 未知フィールドの回答ログ（自動作成）

company/                        # 企業別の送信データ（プロジェクトルート、git管理外）
└── YYYY-MM-DD/
    └── {企業名}/
        ├── before.json           # 送信前の入力内容（JSON）
        └── after.png             # 送信後のスクリーンショット
```

---

## エラーハンドリング

エラーは **リトライ対象** と **即スキップ** に分類される。

| 分類 | エラー種別 | 動作 |
|---|---|---|
| リトライ対象 | タイムアウト | 最大3回リトライ（リロード→再遷移→スキップ） |
| リトライ対象 | 送信エラー（必須項目・入力不備等） | 最大3回リトライ |
| リトライ対象 | reCAPTCHA v3（非表示型） | 送信後にエラーが返された場合、最大3回リトライ |
| 即スキップ | reCAPTCHA v2 画像チャレンジ | ユーザーに手動解決を依頼、不可ならスキップ |
| 即スキップ | hCaptcha / その他CAPTCHA | リトライせずスキップ |
| 即スキップ | ログイン要求 | リトライせずスキップ |
| 即スキップ | フォーム未検出（2階層探索後） | リトライせずスキップ |

3回以上失敗した企業は `docs/error-log.json` に蓄積され、セッション終了時に `docs/error-rules.json` への昇格をユーザーに提案する。次回セッション以降、該当企業はフォーム遷移前に自動スキップされる。

---

## 設定ファイル一覧

| ファイル | 場所 | 内容 |
|---|---|---|
| `config.json` | `~/.claude/form-submit/` | スプレッドシートURL・commonData・オプション |
| `field-log.json` | `~/.claude/form-submit/` | 未知フィールドの回答ログ（自動作成） |
| `progress.json` | `docs/` | 前回の最終処理行・各社結果 |
| `error-log.json` | `docs/` | セッション横断の失敗履歴 |
| `error-rules.json` | `docs/` | 自動スキップルール |

---

## ライセンス

MIT
