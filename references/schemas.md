# form-submit JSON スキーマ リファレンス

全 JSON スキーマの定義を集約したリファレンス。各スキーマの構造と各フィールドの説明を記載する。

---

## config.json

パス: `~/.claude/form-submit/config.json`

```json
{
  "spreadsheet": {
    "url": "(Google SpreadsheetのURL)",
    "columns": {
      "companyName": "A",
      "submitted": "B",
      "formUrl": "C"
    },
    "sheet": "シート1"
  },
  "commonData": {
    "会社名": "...",
    "会社フリガナ": "...",
    "氏名": "...",
    "姓": "...",
    "名": "...",
    "フリガナ（ひらがな）": "...",
    "フリガナ（カタカナ）": "...",
    "メールアドレス": "...",
    "担当者メールアドレス": "...",
    "電話番号": "...",
    "部署名": "...",
    "役職": "...",
    "お問い合わせ内容": "...",
    "お問い合わせ種別": "..."
  },
  "options": {
    "confirmBeforeSubmit": false,
    "screenshotAfterSubmit": true,
    "skipOnError": true,
    "maxCompanies": 30,
    "formSearchKeywords": ["お問い合わせ", "Contact"]
  }
}
```

### spreadsheet フィールド

| フィールド | 型 | 説明 |
|---|---|---|
| `url` | string | 対象の Google Spreadsheet の URL。企業リストを管理するシート。 |
| `columns` | object | 各列の役割を定義するオブジェクト。 |
| `sheet` | string | 対象シート名。デフォルト: `"シート1"` |

#### columns の詳細

| フィールド | 型 | デフォルト | 説明 |
|---|---|---|---|
| `companyName` | string | `"A"` | 企業名の列 |
| `submitted` | string | `"B"` | 送信済みフラグの列。チェックボックス形式で管理する。 |
| `formUrl` | string | `"C"` | お問い合わせフォームの URL の列 |

### commonData フィールド

キーはフォームのフィールドラベルとの意味的マッチングに使用される。任意のキー・値を追加可能。field-log.json のエントリで `count >= 3` になったフィールドは config への昇格候補となる。

### options フィールド

| フィールド | 型 | デフォルト | 説明 |
|---|---|---|---|
| `confirmBeforeSubmit` | boolean | `false` | 送信前に毎回確認するか |
| `screenshotAfterSubmit` | boolean | `true` | 送信後にスクリーンショットを撮影するか |
| `skipOnError` | boolean | `true` | エラー発生時にスキップして次の企業に進むか |
| `maxCompanies` | number | `150` | 1 回の実行で処理する最大企業数 |
| `formSearchKeywords` | string[] | `["お問い合わせ", "Contact"]` | フォーム探索時にリンクを検索するキーワード |

---

## field-log.json

パス: `~/.claude/form-submit/field-log.json`

フォーム送信時に検出されたフィールド情報を蓄積するログファイル。

### 構造

```json
{
  "entries": [
    {
      "fieldLabel": "(フォーム上のフィールドラベル)",
      "category": "(意味カテゴリ: department, position, url 等)",
      "value": "(ユーザーが入力した値)",
      "count": 1,
      "companies": ["(出現した企業名)"]
    }
  ]
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `fieldLabel` | string | フォーム上で検出されたフィールドのラベルテキスト |
| `category` | string | LLM が判断した意味カテゴリ。同じ意味のフィールドをグループ化するために使用する（例: `department`, `position`, `url`）。 |
| `value` | string | そのフィールドに入力する値 |
| `count` | number | 出現回数。**3 以上で config への昇格候補**となる。 |
| `companies` | string[] | 出現した企業名のリスト |

### category フィールドのガイドライン

| category 値 | 対象フィールドの例 |
|---|---|
| `department` | 部署名・所属部署・担当部署 |
| `position` | 役職・職位・肩書き |
| `inquiry_type` | お問い合わせ種別・お問い合わせカテゴリ・ご用件 |
| `company_url` | 会社URL・企業サイト・ホームページURL |
| `employee_count` | 従業員数・社員数・会社規模 |
| `how_found` | 弊社を知ったきっかけ・本サービスを知ったルート・参照元 |
| `other` | 上記いずれにも該当しない雑多なフィールド |

---

## error-log.json

パス: `${CLAUDE_SKILL_DIR}/docs/error-log.json`

フォーム送信時に発生したエラーの履歴を蓄積するログファイル。セッションごとに上書きせず追記・マージする。

### 構造

```json
{
  "entries": [
    {
      "errorType": "(エラー種別)",
      "company": "(企業名)",
      "formUrl": "(フォームURL)",
      "count": 1,
      "lastOccurrence": "2026-03-04T14:57:00+09:00",
      "sessions": ["2026-03-04"]
    }
  ]
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `errorType` | string | エラーの種別。「CAPTCHA検出」「タイムアウト」「送信エラー」「フォーム未検出」「ログイン要求」等 |
| `company` | string | エラーが発生した企業名 |
| `formUrl` | string | エラーが発生したフォームの URL |
| `count` | number | 同じ `company` + `errorType` の組み合わせでの発生回数。**3 以上で error-rules への昇格候補**となる |
| `lastOccurrence` | string | 最後にエラーが発生した日時（ISO 8601 形式） |
| `sessions` | string[] | エラーが発生したセッションの日付リスト |

### マージルール

- 同じ `company` + `errorType` の組み合わせで既存エントリがあれば `count` を +1、`sessions` に現在の日付を追記、`lastOccurrence` を更新する
- 新規の組み合わせなら新規エントリを追加する
- ファイルが存在しない場合は `{ "entries": [] }` として扱う

---

## error-rules.json

パス: `${CLAUDE_SKILL_DIR}/docs/error-rules.json`

`error-log.json` で `count >= 3` になったエラーパターンから昇格した自動スキップルール。スキル実行時にフォーム URL 遷移前に参照し、該当企業・エラーパターンを自動スキップする。

### 構造

```json
{
  "rules": [
    {
      "errorType": "(エラー種別)",
      "company": "(企業名)",
      "formUrl": "(フォームURL)",
      "action": "skip",
      "reason": "(昇格理由)",
      "createdAt": "2026-03-04T15:00:00+09:00"
    }
  ]
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `errorType` | string | エラーの種別。error-log.json の `errorType` と同じ値 |
| `company` | string | 対象企業名 |
| `formUrl` | string | 対象フォームの URL |
| `action` | string | ルールのアクション。現在は `"skip"`（自動スキップ）固定。将来的に拡張可能 |
| `reason` | string | ルール昇格の理由（例: 「CAPTCHA検出が3回以上発生」） |
| `createdAt` | string | ルールが作成された日時（ISO 8601 形式） |

### ルール照合ロジック

1. スキル実行時、フォーム URL 遷移前に `error-rules.json` の `rules` を走査する
2. 現在の企業名と一致する `company` のルールが存在し、`action` が `"skip"` の場合、その企業をスキップする
3. スキップした場合、エラーリストに「自動スキップ: {reason}」として記録する

---

## before.json

パス: `company/{YYYY-MM-DD}/{企業名}/before.json`

フォーム送信直前の入力内容を保存するファイル。

### 構造

```json
{
  "company": "(企業名)",
  "url": "(フォームURL)",
  "submittedAt": "2026-03-04T14:57:00+09:00",
  "fields": {
    "会社名": "株式会社サンプル",
    "氏名": "山田 太郎",
    "メールアドレス": "info@example.com"
  }
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `company` | string | 送信先企業名 |
| `url` | string | フォームの URL |
| `submittedAt` | string | 送信日時（ISO 8601 形式） |
| `fields` | object | フォームに入力した全フィールドのキー・値ペア |

---

## error.json

パス: `company/{YYYY-MM-DD}/{企業名}/error.json`

フォーム送信失敗時にエラー情報を保存するファイル。

### 構造

```json
{
  "company": "(企業名)",
  "url": "(フォームURL)",
  "status": "skip",
  "error": "(詳細理由)",
  "errorCategory": "(エラー種別)",
  "retriable": false,
  "occurredAt": "2026-03-04T15:00:00+09:00"
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `company` | string | 対象企業名 |
| `url` | string | フォームの URL |
| `status` | string | `"skip"` または `"error"` |
| `error` | string | エラーの詳細理由 |
| `errorCategory` | string | エラーの種別（「CAPTCHA検出」「タイムアウト」「送信エラー」「フォーム未検出」「ログイン要求」等） |
| `retriable` | boolean | リトライ可能かどうか |
| `occurredAt` | string | エラー発生日時（ISO 8601 形式） |

---

## progress.json

パス: `${CLAUDE_SKILL_DIR}/docs/progress.json`

セッション内の処理進捗を記録するファイル。

### 構造

```json
{
  "lastRow": 1049,
  "lastCompany": "(最後に処理した企業名)",
  "updatedAt": "2026-03-06T21:00:00+09:00",
  "results": [
    {
      "row": 1043,
      "company": "(企業名)",
      "status": "success",
      "retries": 0
    },
    {
      "row": 1044,
      "company": "(企業名)",
      "status": "skip",
      "reason": "(スキップ理由)"
    }
  ]
}
```

### 各フィールドの説明

| フィールド | 型 | 説明 |
|---|---|---|
| `lastRow` | number | 最後に処理した行番号 |
| `lastCompany` | string | 最後に処理した企業名 |
| `updatedAt` | string | 更新日時（ISO 8601 形式） |
| `results` | array | セッション内の各企業の処理結果 |

### results の各要素

| フィールド | 型 | 説明 |
|---|---|---|
| `row` | number | 行番号 |
| `company` | string | 企業名 |
| `status` | string | `"success"` または `"skip"` |
| `retries` | number | リトライ回数（success 時） |
| `reason` | string | スキップ理由（skip 時） |
