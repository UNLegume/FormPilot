# フォーム入力・送信ステップ（1社あたり）

企業ループ内で各社ごとに実行するステップ。SKILL.md のフェーズ2 から参照される。

---

## ステップ 5: フォームURLに遷移

フォームURLに遷移する前に `error-rules.json` を確認する：

1. `error-rules.json` の `rules` から、現在の企業名に一致するルールを検索する
2. 一致するルールが存在し `action` が `"skip"` の場合:
   スキップしてエラーリストに「自動スキップ: {reason}」として記録し、次の企業に進む
3. 一致するルールがない場合: `browser_navigate` でC列のURLに遷移する

## ステップ 6: フォーム検出

1. `browser_run_code` でフォームフィールドの存在を軽量に確認する：
   ```javascript
   document.querySelectorAll('input,textarea,select').length
   ```
2. フィールド数が1以上の場合: `browser_snapshot` でページ構造を取得し、そのままステップ7へ進む
3. フィールドが0の場合: フォーム探索ロジック（後述）を実行する

### フォーム探索ロジック

URLがフォームページでない場合（ステップ6でフォームが見つからない場合）に実行する。

1. `browser_run_code` で現在のページから関連リンクを軽量に抽出する：
   ```javascript
   const keywords = /お問い合わせ|Contact|パートナー|協業|提携|SES|採用|ビジネス/i;
   JSON.stringify(Array.from(document.querySelectorAll('a')).filter(a => keywords.test(a.textContent)).map(a => ({ text: a.textContent.trim(), href: a.href })).slice(0, 5));
   ```
2. 最も関連性が高いリンクを選択し、`browser_click` で遷移する
3. 遷移先で `browser_run_code` でフォームフィールドの有無を確認する（`document.querySelectorAll('input,textarea,select').length`）
4. フィールドが1以上であれば `browser_snapshot` を取得してステップ7へ進む
5. 1階層目でも見つからない場合: 同様のキーワードで2階層目まで探索する（`browser_run_code` でリンク抽出→クリック→run_code で確認）
6. 2階層探索してもフォームが見つからない場合: スキップしてエラーリストに「フォーム未検出」として記録する

## ステップ 7: フィールド照合

意味的にマッチングする。詳細は `references/config-schema.md` のマッチングロジック参照。

## ステップ 8: 未知フィールド対応

各フィールドについて以下を判断する：

**マッチ済みフィールド:** `commonData` の対応値を自動使用する

**未知フィールド（`commonData` にも意味的に近いキーがない場合）:**
1. `field-log.json` の `entries` を検索し、同じまたは類似した `fieldLabel` のエントリを探す
2. ログに記録あり → そのエントリの `value` を使用し、`count` を +1、`companies` に当該企業名を追加する
3. ログに記録なし → `AskUserQuestion` ツールでユーザーに質問する：

```
[未知フィールド] 「{フィールドラベル}」への入力値を教えてください。
（このフォームでは選択肢: {選択肢リスト} があります）
```

ユーザーの回答を `field-log.json` に新規エントリとして追加する（`count: 1`）。

## ステップ 9: フォーム入力

`browser_fill_form` を優先使用し、複数のテキストフィールドを1回のAPI呼び出しでまとめて入力する。
`browser_fill_form` で対応できないフィールドは個別に処理する：
- チェックボックス・ラジオボタン: `browser_click`
- セレクトボックス: `browser_select_option`

バッチ処理の効率を最大化するため、個々の入力にユーザー確認は挟まない。

## ステップ 10: Write ツールで送信前の入力内容を JSON 保存

**⚠ `browser_take_screenshot` は使用しない。** 必ず Write ツールで JSON ファイルを保存すること。

フォーム入力完了後、送信前に入力データをJSONファイルとして **Write ツール** で保存する。
保存先: `company/{企業名}/before.json`（プロジェクトルート相対、企業名の `/` 等は `_` に置換）
内容: フォームに入力した全フィールドのデータ（フィールド名と値のペア）

```json
{
  "company": "企業名",
  "url": "フォームURL",
  "submittedAt": "2026-03-04T00:00:00.000Z",
  "fields": {
    "お名前": "山田 太郎",
    "メールアドレス": "example@example.com",
    "お問い合わせ内容": "..."
  }
}
```

`config.options.confirmBeforeSubmit` が `true` の場合、ここで `AskUserQuestion` ツールを使い「{企業名} のフォームを送信しますか？」とユーザー確認を求める。`false` または未設定の場合はそのままステップ11へ進む。

## ステップ 11: 送信・確認画面対応・結果確認

日本のフォームは「確認画面→送信」の2段階が一般的なため、完了まで最大3回のクリックを試みる：

1. 送信ボタン（「送信」「Submit」「確認する」「次へ」等）を `browser_click` でクリック
2. `browser_run_code` でページ状態を軽量に確認する：
   ```javascript
   JSON.stringify({ url: location.href, title: document.title, hasThanks: /送信完了|ありがとう|complete|thanks/i.test(document.body.innerText), hasConfirm: /確認|confirm/i.test(document.title) })
   ```
3. **確認画面の場合**（`hasConfirm: true`）: 「送信する」「上記内容で送信」「確定する」等のボタンを `browser_click` でクリックし、再度 `browser_run_code` で確認（最大3回）
4. **完了画面の場合**（`hasThanks: true`、またはURL が `/thanks`、`/complete` 等に遷移）:
   - `config.options.screenshotAfterSubmit` が `true` なら `browser_take_screenshot` で撮影（保存先: `company/{企業名}/after.png`）
   - 成功として記録
5. 成功と判断できない場合はステップ 12（リトライ＆エラー処理）に進む

## ステップ 12: リトライ＆エラー処理

### リトライルール（最大3回）

エラーが発生した場合、同じ企業に対して最大3回までリトライする。

1. 1回目の試行でエラー → ページをリロードして2回目を試行
2. 2回目もエラー → フォームURLに再遷移して3回目を試行
3. 3回目もエラー → その企業をパス（スキップ）する

リトライ中に成功した場合は、成功企業リストに追加する。

### 3回失敗した場合

- エラー内容（企業名・エラー種別・詳細・試行回数）をメモリ内のエラーリストに追加する
- `error-log.json` に失敗を記録する（同じ `company` + `errorType` の既存エントリがあれば `count` +1・`sessions` に追記、なければ新規追加）
- スプレッドシートのB列は更新しない
- 次の企業の処理に進む（ステップ 5 に戻る）

`config.options.skipOnError: false` の場合:
- 3回リトライしても失敗した時点でバッチ処理を停止する

**注意:** ステップ13〜19（スプレッドシート更新・ログ保存・レポート表示）は `skipOnError` の値に関わらず必ず実行する。「バッチ処理を停止」とは新しい企業の処理を開始しないことを意味し、後処理はスキップしない。

### エラー分類

エラーは **リトライ対象** と **即スキップ** に分類される。リトライ対象のエラーはステップ 12 のリトライルール（最大3回）に従い、即スキップのエラーはリトライせず次の企業に進む。

#### CAPTCHA対応

`browser_run_code` でCAPTCHA要素（`iframe[src*="recaptcha"]`, `.h-captcha`, `#hcaptcha` 等）の有無を軽量確認し、検出した場合は以下の対応を行う：

| CAPTCHAの種類 | 対応 |
|---|---|
| reCAPTCHA v2 | チェックボックスをクリック→通過すれば続行、画像チャレンジが出たら即スキップ |
| reCAPTCHA v3 | そのまま送信→CAPTCHAエラー時はステップ12でリトライ |
| hCaptcha/その他 | 即スキップ |

#### タイムアウト

`browser_navigate` が想定時間内に完了しない場合:
→ ステップ 12（リトライ＆エラー処理）に進む。3回リトライしても失敗した場合は「タイムアウト」としてエラーリストに記録する

#### ログイン要求

`browser_snapshot` の結果にログインフォーム（`type="password"` フィールドや「ログイン」「サインイン」ボタン）が含まれる場合:
→ スキップしてエラーリストに「ログイン要求」として記録する

#### 送信エラー

送信後のページに「エラー」「入力内容をご確認」「必須項目」「送信に失敗」等のメッセージが含まれる場合、またはフォームが再表示されている場合:
→ ステップ 12（リトライ＆エラー処理）に進む。3回リトライしても失敗した場合は「送信エラー: {エラーメッセージ内容}」としてエラーリストに記録する

#### フォーム未検出

→ フォーム探索ロジック（ステップ6の後述セクション）に従う
