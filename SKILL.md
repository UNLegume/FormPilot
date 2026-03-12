---
name: form-submit
description: >
  Google Spreadsheetの企業リストからお問い合わせフォームにPlaywright MCPで
  定型データを自動入力・一括送信する。フォーム検出→入力→送信→スプレッドシート
  更新を自動ループする。
  ユーザーが「フォーム送信」「お問い合わせ送信」「営業フォーム」
  「一括送信」「form submit」「フォーム自動入力」「問い合わせ自動化」
  「フォーム一括」「お問い合わせ自動」「フォームを送って」「企業にフォームを送る」
  「自動で問い合わせ」と言ったときに必ずこのスキルを使用すること。
  /form-submit で直接呼び出し可能。
argument-hint: "[--company 企業名] [--max N]"
allowed-tools: >
  Read, Write, Glob, Grep,
  Bash(ls *), Bash(mkdir *),
  mcp__playwright__browser_navigate,
  mcp__playwright__browser_click,
  mcp__playwright__browser_type,
  mcp__playwright__browser_take_screenshot,
  mcp__playwright__browser_run_code,
  mcp__playwright__browser_snapshot,
  mcp__playwright__browser_select_option,
  mcp__playwright__browser_fill_form,
  mcp__playwright__browser_press_key
---

# フォーム自動送信スキル

お問い合わせフォームを Playwright MCP で自動巡回し、定型データを入力・送信するスキル。

---

## 呼び出し方

```
/form-submit                            → 未送信企業を最大 maxCompanies 件まで連続処理
/form-submit --company サイバーフリークス  → 特定企業のみ処理
/form-submit --max 10                   → 最大10社まで処理
/form-submit --max 10 --company 企業名   → 組み合わせも可能
```

引数: $ARGUMENTS
- `--company {企業名}` — その企業のみを処理対象
- `--max {N}` — 処理する最大企業数（config.options.maxCompanies より優先）
- 引数なし — 未送信企業を先頭から順に maxCompanies 件まで処理

---

## 設定ファイル

実行前に `~/.claude/form-submit/config.json` を Read で読み込む。
スキーマ詳細は `${CLAUDE_SKILL_DIR}/references/schemas.md` を参照。設定例は `${CLAUDE_SKILL_DIR}/references/config-examples.md` を参照。

その他のファイル:
- `~/.claude/form-submit/field-log.json` — 未知フィールドの過去回答。実行前に読み込み、ループ終了後に保存。なければ `{ "entries": [] }` として扱う
- `${CLAUDE_SKILL_DIR}/docs/error-log.json` — 失敗履歴の蓄積。セッションごとに追記・マージ。なければ `{ "entries": [] }` として扱う
- `${CLAUDE_SKILL_DIR}/docs/error-rules.json` — 自動スキップルール。フォーム URL 遷移前に参照。なければ `{ "rules": [] }` として扱う

---

## 前提条件

- Playwright MCP は既存の Chrome プロファイル（Google ログイン済み）を使用する
- スキル実行前に Google Chrome を閉じること（プロファイルロック競合を回避）
- Chrome を閉じ忘れた場合はエラーになるので、ユーザーに閉じるよう案内する

---

## フェーズ 1: バッチ初期化（Step 1–4）

### Step 1: 設定・ログ読み込み

`$ARGUMENTS` を解析する（`--max {N}` → maxCompanies 上書き、`--company {名}` → 対象限定）。

Read で以下を読み込む：

1. `~/.claude/form-submit/config.json` — spreadsheet, options, commonData を取得
2. `~/.claude/form-submit/field-log.json` — 未知フィールドの過去回答
3. `${CLAUDE_SKILL_DIR}/docs/progress.json` — 前回の最終処理行
4. `${CLAUDE_SKILL_DIR}/docs/error-log.json` — 過去の失敗履歴
5. `${CLAUDE_SKILL_DIR}/docs/error-rules.json` — 自動スキップルール
6. Playwright MCP 設定ファイルを Read し `--user-data-dir` の有無を確認。未設定なら「reCAPTCHA 通過率が下がりますが続行しますか？」と確認

**progress.json が存在する場合:**
ユーザーに選択肢を提示する：
- 続きから処理する（前回の lastRow + 1 から）
- 開始行を指定する

**progress.json が存在しない場合:**
何行目から開始するかユーザーに直接聞く。

### Step 2: スプレッドシート遷移と Google ログイン確認

1. `browser_navigate` で `config.spreadsheet.url` に遷移
2. `browser_run_code` でログイン状態を確認（スプレッドシート表示 or ログインページ）
3. ログイン済み → Step 3 へ
4. ログイン画面 → ユーザーにブラウザでのログインを依頼し、完了後に再確認

### Step 3: スプレッドシートから企業リスト取得

1. `browser_run_code` で企業名・送信済みフラグ・URL を JS で抽出する（`browser_snapshot` は DOM が大きすぎるため使わない）
2. B列が未チェック（空欄または FALSE）の行を抽出し企業リストを作成
3. `--company` 指定時は該当企業のみ対象

### Step 4: 企業ループ開始

未送信企業を先頭から順に処理。進捗を `[3/30] 株式会社XXX を処理中...` の形式で都度表示する。

成功企業名はメモリ内リストに保持し、ループ終了後にスプレッドシートへ一括反映する。1社ごとにスプレッドシートへ戻ると往復コストが高いため、後処理 Step 13 でまとめて更新する。

---

## フェーズ 2: フォーム処理ループ（Step 5–12）

各企業について以下を実行する。

### Step 5: フォーム URL に遷移

1. `error-rules.json` で現在の企業名に一致する `action: "skip"` ルールがあればスキップし、エラーリストに「自動スキップ: {reason}」として記録
2. ルールなし → `browser_navigate` でフォーム URL に遷移

### Step 6: フォーム検出

1. `browser_run_code` でフォームフィールド数（`input,textarea,select`）を確認
2. フィールド 1 以上 → `browser_snapshot` でページ構造を取得し Step 7 へ
3. フィールド 0 → フォーム探索ロジック:
   - `browser_run_code` で `config.options.formSearchKeywords` のキーワード（未設定時デフォルト: 「お問い合わせ|Contact」）を含むリンクを抽出
   - 最も関連性が高いリンクを `browser_click` で遷移し、フォームフィールドの有無を確認
   - 1 階層目で見つからなければ 2 階層目まで探索
   - 2 階層探索しても見つからない → スキップし「フォーム未検出」として記録

### Step 7: フィールド照合

commonData のキーとフォームのフィールドラベルを**意味的にマッチング**する。

**動作の原則:**

- **意味的対応の認識**: 表記が異なっていても同じ意味を指すラベルは同一として扱う（例: config の `"会社名"` → フォームの「御社名」「貴社名」「Company Name」「企業名」）
- **セレクトボックス**: commonData の値に意味的に最も近い選択肢を選ぶ（例: `"資料請求"` → 「資料のご請求」）
- **テキストエリア**: 改行を含む長文もそのまま入力。`\n` は実際の改行として扱う
- **マッチしない場合**: field-log.json に記録して Step 8 で対応

**主なマッチング例:**

| config のキー | マッチするフォームラベルの例 |
|---|---|
| `"会社名"` | 「御社名」「貴社名」「Company」「企業名」等 |
| `"氏名"` | 「お名前」「Name」「ご担当者名」等 |
| `"メールアドレス"` | 「E-mail」「email」「メール」等 |
| `"電話番号"` | 「TEL」「お電話番号」「Phone」等 |

> スキーマ詳細: `${CLAUDE_SKILL_DIR}/references/schemas.md`

### Step 8: 未知フィールド対応

- **マッチ済み:** commonData の対応値を使用
- **未知フィールド:**
  1. `field-log.json` で同じ/類似の `fieldLabel` を検索
  2. 記録あり → その `value` を使用し `count` +1、`companies` に企業名追加
  3. 記録なし → `AskUserQuestion` でユーザーに質問し、回答を field-log.json に追加（`count: 1`）

### Step 9: フォーム入力

- `browser_fill_form` で複数テキストフィールドをまとめて入力
- チェックボックス・ラジオ → `browser_click`
- セレクトボックス → `browser_select_option`
- 入力開始前と送信ボタンクリック前に `browser_run_code` で 1–2 秒待機（bot 検出回避）

### Step 10: 送信前データ保存

送信前の入力内容を JSON で保存する。スクリーンショットより JSON の方が後から解析しやすく、ストレージも節約できるため `browser_take_screenshot` は使用しない。Write で `company/{YYYY-MM-DD}/{企業名}/before.json` に保存（`/` 等は `_` に置換）。内容: `{ "company", "url", "submittedAt", "fields": { フィールド名: 値, ... } }`

> スキーマ詳細: `${CLAUDE_SKILL_DIR}/references/schemas.md`

`config.options.confirmBeforeSubmit` が `true` の場合、`AskUserQuestion` で「{企業名} のフォームを送信しますか？」と確認。

### Step 11: 送信・確認画面対応・結果確認

日本のフォームは「確認画面→送信」の 2 段階が一般的。完了まで最大 3 回のクリックを試みる：

1. 送信ボタン（「送信」「Submit」「確認する」「次へ」等）を `browser_click`
2. `browser_run_code` でページ状態を確認（完了: `送信完了|ありがとう|complete|thanks`、確認画面: `確認|confirm`）
3. **確認画面** → 「送信する」「上記内容で送信」等をクリックし再確認（最大 3 回）
4. **完了画面** → `screenshotAfterSubmit` が `true` なら以下のルールで `browser_take_screenshot`（保存先: `{YYYY-MM-DD}/{企業名}/after.png`）→ 成功として記録
   - **通常フォーム（完了ページに遷移する場合）**: 完了メッセージが見える位置までスクロールしてからスクリーンショット
   - **iframe 内フォーム（Microsoft Forms 等）**: iframe 要素を `ref` 指定してスクリーンショット（親ページではなく iframe 内の完了メッセージを撮る）
   - **サイレントリセット型（Wix フォーム等、送信後にフォームがリセットされる場合）**: 送信成功はネットワークレスポンス（status 200）で確認。after.png は撮らない
5. 完了と判断できない → Step 12 へ

### Step 12: リトライ＆エラー処理

**リトライルール（最大 3 回）:**

1. 1 回目エラー → ページリロードして 2 回目
2. 2 回目エラー → フォーム URL に再遷移して 3 回目
3. 3 回目エラー → スキップ（`skipOnError: false` の場合は新規企業の処理を停止。ただし後処理 Step 13–17 は必ず実行する）

**エラー分類早見表:**

| エラー種別 | 対応 |
|---|---|
| タイムアウト | リトライ（最大 3 回） |
| 送信エラー（必須項目・入力不備等） | リトライ（最大 3 回） |
| reCAPTCHA v3（送信後エラー） | リトライ（最大 3 回） |
| reCAPTCHA v2 チェックボックス | クリック → 画像チャレンジなら `AskUserQuestion` で手動解決依頼 |
| 画像認証 CAPTCHA（テキスト読み取り型） | `browser_take_screenshot` で読み取り試行 → 失敗なら `AskUserQuestion` で手動入力依頼 |
| hCaptcha / その他 CAPTCHA | 即スキップ |
| ログイン要求 | 即スキップ |
| フォーム未検出（2 階層探索後） | 即スキップ |

3 回失敗した場合: エラーリストに追加し `${CLAUDE_SKILL_DIR}/docs/error-log.json` に記録（既存エントリは `count` +1）、次の企業へ（Step 5 に戻る）。

スキップまたはエラーとなった企業は Write で `company/{YYYY-MM-DD}/{企業名}/error.json` に保存する。

> スキーマ詳細: `${CLAUDE_SKILL_DIR}/references/schemas.md`

---

## フェーズ 3: 後処理（Step 13–17）

### Step 13: スプレッドシート一括更新

1. `browser_navigate` で spreadsheet URL に遷移
2. メモリ内の成功企業リストに基づき、A 列を検索して各企業の行を特定
3. B 列のチェックボックスを `browser_click` で ON にする
4. `browser_run_code` で更新結果を確認

### Step 14: ログファイル保存

以下の 2 ファイルを Write で保存する：

1. `~/.claude/form-submit/field-log.json` — 未知フィールドの学習データ
2. `${CLAUDE_SKILL_DIR}/docs/error-log.json` — エラー履歴（セッション追記）

### Step 15: 昇格提案

**エラールール昇格:** `error-log.json` で `count >= 3` かつ未ルール化のエントリを抽出し、自動スキップルールへの追加をユーザーに提案する。承認されたら `${CLAUDE_SKILL_DIR}/docs/error-rules.json` に Write で保存。

**config 昇格:** `field-log.json` で `count >= 3` のエントリを抽出し、`config.json` の commonData への追加をユーザーに提案する。承認されたら `~/.claude/form-submit/config.json` に Write で保存。

### Step 16: 進捗記録

Write で `${CLAUDE_SKILL_DIR}/docs/progress.json` に lastRow, lastCompany, updatedAt を記録。

### Step 17: 最終レポート表示

処理範囲・成功数（リトライ後成功含む）・失敗数・スキップ数を集計表示。成功/失敗の詳細一覧、昇格候補の処理結果、進捗保存先を表示。

---

## 参考リソース

- [`${CLAUDE_SKILL_DIR}/references/schemas.md`](${CLAUDE_SKILL_DIR}/references/schemas.md) — 全 JSON スキーマ定義
- [`${CLAUDE_SKILL_DIR}/references/config-examples.md`](${CLAUDE_SKILL_DIR}/references/config-examples.md) — 設定サンプル・業種別カスタマイズ例
