---
name: form-submit
description: >
  SES企業パートナー候補のお問い合わせフォームにPlaywright MCPで定型データを
  自動入力・一括送信する。Google Spreadsheetの企業リストから未送信分を自動取得し、
  フォーム検出→入力→送信→スプレッドシート更新を自動ループする。
  ユーザーが「フォーム送信」「お問い合わせ送信」「パートナーフォーム」「営業フォーム」
  「一括送信」「form submit」「フォーム自動入力」と言ったとき、またはSES企業への
  問い合わせやパートナー提携の営業活動に言及したときに必ずこのスキルを使用すること。
  /form-submit で直接呼び出し可能。
argument-hint: "[--company 企業名] [--max N]"
---

# フォーム自動送信スキル

SES企業のお問い合わせフォームを自動巡回し、定型データを入力・送信するスキル。
Playwright MCP のブラウザ操作を使ってフォームの検出・入力・送信・スプレッドシート更新を自動ループする。

---

## 呼び出し方

```
/form-submit                            → 未送信企業を最大 config.options.maxCompanies 件まで連続処理
/form-submit --company サイバーフリークス  → 特定企業のみ処理
/form-submit --max 30                   → 最大30社まで処理（config.options.maxCompanies を上書き）
/form-submit --max 10 --company 企業名   → 組み合わせも可能
```

引数: $ARGUMENTS
- `--company {企業名}` が指定された場合、その企業のみを処理対象とする
- `--max {N}` が指定された場合、処理する最大企業数を N に設定する（config.options.maxCompanies より優先）
- 引数なしの場合、未送信企業を先頭から順に config.options.maxCompanies 件まで処理する

---

## 設定ファイル

### `~/.claude/form-submit/config.json`

実行前に必ず Read ツールで読み込む。ネスト構造の詳細は `references/config-schema.md` を参照。

主要キー: `spreadsheet.url`, `spreadsheet.columns`, `spreadsheet.sheet`, `options.*`, `commonData`

### `~/.claude/form-submit/field-log.json`

未知フィールドへの対応を永続記録。実行前に読み込み、ループ終了後に保存。存在しなければ `{ "entries": [] }` として扱う。

### `docs/error-log.json`

失敗履歴の蓄積ファイル。セッションごとに追記・マージ。実行前に読み込み、ループ終了後に保存。存在しなければ `{ "entries": [] }` として扱う。

### `docs/error-rules.json`

`error-log.json` で3回以上発生したエラーパターンの自動スキップルール。フォームURL遷移前に参照。存在しなければ `{ "rules": [] }` として扱う。

---

## 全体フロー

3フェーズで構成される。各フェーズの詳細は参照先ファイルを Read ツールで読み込んで実行する。

| フェーズ | ステップ | 概要 | 参照先 |
|---------|---------|------|--------|
| **1. バッチ初期化** | 1-4 | 設定読み込み→SS遷移→企業リスト取得→ループ開始 | `docs/batch-lifecycle.md`「バッチ初期化」 |
| **2. フォーム処理** | 5-12 | URL遷移→検出→入力→送信→リトライ（各社ループ） | `docs/form-steps.md` |
| **3. 後処理** | 13-19 | SS一括更新→ログ保存→昇格提案→レポート | `docs/batch-lifecycle.md`「ループ終了後」 |

### フェーズ 1: バッチ初期化（Step 1-4）

`docs/batch-lifecycle.md` の「バッチ初期化」セクションを Read ツールで読み込み、Step 1〜4 を実行する。

### フェーズ 2: フォーム処理ループ（Step 5-12）

`docs/form-steps.md` を Read ツールで読み込み、各企業について Step 5〜12 を実行する。

### フェーズ 3: 後処理（Step 13-19）

`docs/batch-lifecycle.md` の「ループ終了後」セクションを Read ツールで読み込み、Step 13〜19 を実行する。

---

## 前提条件（reCAPTCHA 対策）

- Playwright MCP は既存の Chrome プロファイル（Google ログイン済み）を使用する設定になっている
- **スキル実行前に Google Chrome を閉じること**（プロファイルロック競合を回避するため）
- Chrome を閉じ忘れた場合はエラーになるので、ユーザーに閉じるよう案内する

## 人間的な操作パターン

bot 検出を回避するため、フォーム入力開始前に `browser_run_code` で 1〜2秒の待機を1回だけ実施する。
入力自体は `browser_fill_form` でまとめて実行し、フィールド間の個別遅延は不要。
送信ボタンクリック前にも 1〜2秒の待機を実施する。

---

## 注意事項

- スプレッドシートの操作は Google Sheets API ではなく、Playwright MCP 経由のブラウザ操作で行う
- 未知フィールドへの質問には `AskUserQuestion` ツールを使用する
- 処理中は成功企業リストをメモリに保持し、ループ終了後にスプレッドシートへ一括反映する（毎回戻ると非効率）
- 独立したツール呼び出しは1ターンでまとめて実行する（例: navigate後のsnapshot取得など依存関係があるものは除く）

---

## 参考リソース

- [設定ファイル スキーマリファレンス](references/config-schema.md) — config.json / field-log.json / error-log.json / error-rules.json の詳細なフィールド定義
- [設定ファイル サンプル・設定例](references/config-schema.example.md) — サンプルデータと最小構成・フル構成の設定例
