# バッチライフサイクル（初期化・後処理）

バッチ処理の前後に実行するステップ。SKILL.md のフェーズ1・フェーズ3 から参照される。

---

## バッチ初期化

### ステップ 1: 設定・ログ読み込み

まず `$ARGUMENTS` を解析する：
- `--max {N}` が含まれていれば、その値を `maxCompanies` として記憶する（config より優先）
- `--company {企業名}` が含まれていれば、その企業のみを処理対象とする

Read ツールで以下を読み込む：
1. `~/.claude/form-submit/config.json` → `spreadsheet.url`、`options`、`commonData` を取得。`--max` 引数があれば `options.maxCompanies` を上書きする
2. `~/.claude/form-submit/field-log.json` → 未知フィールドの過去回答を取得
3. `docs/progress.json`（正規パス） → 前回の最終処理行を取得。※ `~/.claude/form-submit/progress.json` は旧形式のため使用しないこと
4. `docs/error-log.json` → 過去の失敗履歴を取得
5. `docs/error-rules.json` → 自動スキップルールを取得

6. Playwright MCP のプロファイル設定を確認する:
   - Playwright MCP の設定ファイル（`~/.claude/plugins/marketplaces/claude-plugins-official/external_plugins/playwright/.mcp.json`）を Read ツールで読み込む
   - `--user-data-dir` 引数が設定されているか確認する
   - 設定されていない場合: ユーザーに「Chromeプロファイルが未設定です。reCAPTCHA通過率が大幅に下がりますが続行しますか？」と確認する
   - 設定されている場合: プロファイルパスが存在するか確認し、問題なければ続行する

**progress.json が存在する場合:**
ユーザーに選択肢を提示する：
```
前回は行{lastRow}（{lastCompany}）まで処理済みです。
[1] 続きから処理する（行{lastRow + 1}から）
[2] 開始行を指定する
```

**progress.json が存在しない場合:**
ユーザーに開始行を直接聞く：
```
progress.json が見つかりません。何行目から開始しますか？
```

### ステップ 2: スプレッドシート遷移とGoogleログイン確認

1. `browser_navigate` で `config.spreadsheet.url` に直接遷移する
2. `browser_run_code` でログイン状態を確認する：
   ```javascript
   JSON.stringify({ url: location.href, isSpreadsheet: location.href.includes('docs.google.com/spreadsheets'), isLoginPage: location.href.includes('accounts.google.com') || document.title.includes('ログイン') || document.title.includes('Sign in') })
   ```
3. ログイン済みでスプレッドシートが表示されている → ステップ3のデータ取得へ進む
4. アカウント選択画面またはログイン画面にリダイレクトされた → ユーザーにブラウザでのログイン・アカウント選択を依頼し、完了後 `browser_run_code` で再確認

### ステップ 3: スプレッドシートから企業リスト取得

1. スプレッドシートが表示済みの状態から続ける（ステップ2で既に遷移済み）
2. `browser_run_code` でJavaScriptを実行し、スプレッドシートから必要なデータのみ取得する（`browser_snapshot` の全DOMは数千〜数万トークンになるため、JS経由で必要データのみ抽出する）：
   ```javascript
   // スプレッドシートのテーブルから企業名・送信済み・URLを抽出
   const rows = document.querySelectorAll('table tbody tr');
   const data = Array.from(rows).map(row => {
     const cells = row.querySelectorAll('td');
     return { company: cells[0]?.textContent?.trim(), submitted: cells[1]?.textContent?.trim(), url: cells[2]?.textContent?.trim() };
   }).filter(r => r.company);
   JSON.stringify(data);
   ```
   ※ 実際のスプレッドシートのDOM構造に応じてセレクタを調整する
3. B列が未チェック（空欄またはFALSE）の行を抽出し、企業リストを作成する
4. `--company` オプション指定時は該当企業名の行のみを対象とする

### ステップ 4: 企業ループ開始

未送信企業を先頭から順番に処理する。バッチ処理の効率を最大化するため、個々のフォーム入力・送信にはユーザー確認を挟まない（未知フィールドへの質問のみが中断ポイント）。

進捗は都度表示してユーザーが状況を把握できるようにする：

```
[3/150] 株式会社XXX を処理中...
```

成功した企業名はメモリ内リストに追記し、ループ終了後にスプレッドシートをまとめて更新する（1社ごとにスプレッドシートへ戻るとブラウザ往復コストが高いため）。

---

## ループ終了後

### ステップ 13: スプレッドシートを一括更新

全社処理完了後にまとめてB列を更新する：

1. `browser_navigate` で `config.spreadsheet.url` に遷移する
2. `browser_run_code` でJavaScriptを実行し、成功企業のチェックボックスを一括操作する：
   - メモリ内の成功企業リストに基づき、A列を検索して各企業の行を特定
   - B列のチェックボックスをクリックしてONにする
   ※ Google Spreadsheetの場合、チェックボックスのクリックはDOM操作では反映されないことがあるため、`browser_click` でのクリックにフォールバックする
3. `browser_run_code` で更新結果を確認する

### ステップ 14: フィールドログ保存

処理中に更新した `field-log.json` の内容を Write ツールで保存する。
パス: `~/.claude/form-submit/field-log.json`

### ステップ 15: エラーログ保存

処理中に更新した `error-log.json` を `docs/error-log.json` に Write ツールで保存する。

### ステップ 16: エラールール昇格提案

`error-log.json` の `entries` の中で `count >= 3` のエントリを抽出し、
かつ `error-rules.json` にまだルール化されていないものを以下の形式で提案する：

```
以下のエラーが3回以上発生しています。自動スキップルールに追加しますか？

- イプソス株式会社: CAPTCHA検出（3回発生）
- 株式会社XXX: タイムアウト（4回発生）

[Y] すべて追加  [n] スキップ  または追加する企業を番号で指定してください
```

ユーザーが承認した場合は `error-rules.json` に該当ルールを追加して `docs/error-rules.json` に Write ツールで保存する。

### ステップ 17: config昇格提案

`field-log.json` の `entries` の中で `count >= 3` のエントリを抽出し、以下の形式でユーザーに提案する：

```
以下のフィールドが3回以上出現しています。config.json の commonData に追加しますか？

- 「部署名」→ 値: 営業部（5回出現）
- 「ご担当者名カナ」→ 値: ヤマダ タロウ（3回出現）

[Y] すべて追加  [n] スキップ  または追加するフィールドを番号で指定してください
```

ユーザーが承認した場合は `~/.claude/form-submit/config.json` の `commonData` に該当キーバリューを追加して Write ツールで保存する。

### ステップ 18: 進捗記録

セッション終了時、スプレッドシートの最後に処理した行番号と各企業の結果を `docs/progress.json` に記録する。
次回セッション開始時にこのファイルを読み込み、前回の続きから処理を再開できるようにする。

lastRow, lastCompany, updatedAt を記録する。

### ステップ 19: 最終レポート表示

処理範囲・成功数（リトライ後成功含む）・失敗数・スキップ数を集計表示する。
成功/失敗の詳細（行番号・企業名・結果）を一覧表示する。
config昇格候補・エラールール昇格候補の処理結果も表示する。
進捗保存先（progress.json）を表示する。
