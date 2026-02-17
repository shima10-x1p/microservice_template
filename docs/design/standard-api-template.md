---
doc_type: feature_design
feature_id: F-001
title: マイクロサービス標準APIテンプレート
status: draft
owners:
  - motoki
reviewers: []
created: 2026-02-18
updated: 2026-02-18
related:
  issues: []
  pull_requests: []
---

# 1. 概要

## 1.1 目的

個人開発マイクロサービスで繰り返し発生するAPI設計・プロジェクト構成の意思決定を排除し、拡張容易・後方互換を重視した標準APIテンプレートを提供する。

## 1.2 背景

マイクロサービスを新規に立ち上げるたびに、レスポンス形式、エラーハンドリング、ページネーション、ディレクトリ構成などをゼロから決めている。統一された雛形があれば、サービス間の一貫性が保たれ、開発立ち上げが高速化される。

## 1.3 スコープ

### 対象

* 統一レスポンスエンベロープ（成功 / エラー）
* ページネーション（オフセットベース: `limit` + `offset`）
* APIバージョニング（URLパス方式: `/api/v1/...`）
* ヘルスチェック / メタ情報エンドポイント
* フィルタリング / ソート / 検索のクエリ規約
* バリデーションエラーの標準形式
* CRUD操作サンプル（POST / GET / PUT / DELETE）の `items` リソース
* リポジトリパターン（インターフェース + インメモリ実装）
* Dockerfile / docker-compose.yml
* テストフレームワーク（pytest + httpx）
* ログ / 設定管理（pydantic-settings）
* パッケージ管理: uv

### 対象外

* 認証・認可（各サービスで個別に追加）
* DB接続 / マイグレーション（SQLAlchemy, Alembic等）
* CI/CD（GitHub Actions等）
* フロントエンド / UI

## 1.4 成果物

| 成果物 | 説明 |
| --- | --- |
| `pyproject.toml` | uv によるプロジェクト定義・依存管理 |
| `app/` | FastAPI アプリケーション本体 |
| `app/schemas/` | 共通レスポンス / エラー / ページネーションスキーマ |
| `app/api/v1/` | v1 ルーター群（items サンプル含む） |
| `app/repositories/` | リポジトリインターフェース + インメモリ実装 |
| `Dockerfile` / `docker-compose.yml` | コンテナ化 |
| `tests/` | pytest テスト群 |
| `README.md` | テンプレートの使い方 |

# 2. 用語・定義

| 用語 | 説明 | 備考 |
| --- | --- | --- |
| エンベロープ | すべてのAPIレスポンスを包む統一JSON構造 | `success`, `data`, `meta`, `error` フィールドで構成 |
| リソース | CRUDの対象となるドメインエンティティ | テンプレートでは `Item` を使用 |
| リポジトリ | データアクセスの抽象レイヤー | Protocol でインターフェースを定義し、DI で差し替え可能にする |

# 3. 前提・制約

## 3.1 動作環境・依存

* 実行環境: Python 3.13+
* フレームワーク: FastAPI (最新安定版)
* バリデーション: Pydantic v2
* 設定管理: pydantic-settings
* パッケージ管理: uv
* テスト: pytest + httpx (FastAPI TestClient)
* コンテナ: Docker

## 3.2 互換性・移行

* 互換性ポリシー: URLパス方式バージョニング (`/api/v1/`) により、破壊的変更は新バージョン (`/api/v2/`) として追加する。既存バージョンのエンドポイントは削除しない。
* 移行方針: テンプレートリポジトリの新規作成のため、既存データの移行は不要。

## 3.3 制約

* 認証は含めないため、テンプレート単体では認証なしで全エンドポイントにアクセス可能。
* DB接続を含めないため、サンプルのデータ永続化はインメモリ（プロセス再起動で消失）。

# 4. ユースケース

## 4.1 利用者と権限

| ロール | できること | できないこと |
| --- | --- | --- |
| テンプレート利用者（開発者） | テンプレートをクローンし、リソース追加・リポジトリ実装差し替え・認証追加などのカスタマイズ | — |
| APIクライアント | 全エンドポイントへのリクエスト（認証なし） | — |

## 4.2 ユースケース一覧

| UC ID | 概要 | トリガー | 終了条件 |
| --- | --- | --- | --- |
| UC-001 | アイテム作成 | POST `/api/v1/items` | 201 + 作成されたアイテムが返る |
| UC-002 | アイテム一覧取得 | GET `/api/v1/items` | 200 + アイテム配列 + ページネーション情報 |
| UC-003 | アイテム単体取得 | GET `/api/v1/items/{item_id}` | 200 + 単一アイテム |
| UC-004 | アイテム更新 | PUT `/api/v1/items/{item_id}` | 200 + 更新後アイテム |
| UC-005 | アイテム削除 | DELETE `/api/v1/items/{item_id}` | 200 + `data: null` |
| UC-006 | ヘルスチェック | GET `/health` | 200 + ステータス |
| UC-007 | メタ情報取得 | GET `/api/v1/meta` | 200 + バージョン情報 |

## 4.3 ユースケース詳細

### UC-001 アイテム作成

#### 事前条件

* なし

#### 基本フロー

1. クライアントが `POST /api/v1/items` にリクエストボディを送信する。
2. サーバーがリクエストボディをバリデーションする。
3. サーバーが UUID v4 を採番し、`created_at` / `updated_at` を現在時刻で設定する。
4. リポジトリにアイテムを保存する。
5. `201 Created` + 作成されたアイテムをエンベロープで返す。

#### 代替フロー

* なし

#### 例外フロー

* リクエストボディが不正 → `422 Unprocessable Entity` + バリデーションエラーをエンベロープで返す。

### UC-002 アイテム一覧取得

#### 事前条件

* なし

#### 基本フロー

1. クライアントが `GET /api/v1/items` にリクエストする（任意でクエリパラメータ付与）。
2. サーバーがクエリパラメータ（`limit`, `offset`, `sort`, フィルタ）をバリデーションする。
3. リポジトリからフィルタ・ソート条件に合致するアイテムを取得する。
4. `200 OK` + アイテム配列と `meta`（`total`, `limit`, `offset`）をエンベロープで返す。

#### 代替フロー

* クエリパラメータなしの場合、デフォルト値（`limit=10`, `offset=0`, ソートなし、フィルタなし）で返す。
* 該当アイテムが0件の場合、`data: []`, `meta.total: 0` で返す。
* `offset` がレコード総数以上の場合、`data: []` で返す（エラーにはしない）。

#### 例外フロー

* `limit` が 1 未満または 100 超 → `422` + バリデーションエラー。
* `offset` が 0 未満 → `422` + バリデーションエラー。
* `sort` の形式が不正 → `422` + バリデーションエラー。

### UC-003 アイテム単体取得

#### 事前条件

* なし

#### 基本フロー

1. クライアントが `GET /api/v1/items/{item_id}` にリクエストする。
2. サーバーが `item_id` を UUID としてバリデーションする。
3. リポジトリからアイテムを取得する。
4. `200 OK` + アイテムをエンベロープで返す。

#### 例外フロー

* `item_id` が UUID 形式でない → `422` + バリデーションエラー。
* アイテムが存在しない → `404 Not Found` + エラーエンベロープ。

### UC-004 アイテム更新

#### 事前条件

* 対象アイテムが存在する。

#### 基本フロー

1. クライアントが `PUT /api/v1/items/{item_id}` にリクエストボディを送信する。
2. サーバーが `item_id` と リクエストボディをバリデーションする。
3. リポジトリからアイテムを取得し、存在を確認する。
4. アイテムのフィールドを更新し、`updated_at` を現在時刻に設定する。
5. `200 OK` + 更新後アイテムをエンベロープで返す。

#### 例外フロー

* `item_id` が UUID 形式でない → `422` + バリデーションエラー。
* リクエストボディが不正 → `422` + バリデーションエラー。
* アイテムが存在しない → `404 Not Found` + エラーエンベロープ。

### UC-005 アイテム削除

#### 事前条件

* 対象アイテムが存在する。

#### 基本フロー

1. クライアントが `DELETE /api/v1/items/{item_id}` にリクエストする。
2. サーバーが `item_id` を UUID としてバリデーションする。
3. リポジトリからアイテムを取得し、存在を確認する。
4. リポジトリからアイテムを削除する。
5. `200 OK` + `{success: true, data: null}` を返す。

#### 例外フロー

* `item_id` が UUID 形式でない → `422` + バリデーションエラー。
* アイテムが存在しない → `404 Not Found` + エラーエンベロープ。

### UC-006 ヘルスチェック

#### 基本フロー

1. クライアントが `GET /health` にリクエストする。
2. `200 OK` + `{"status": "ok"}` を返す。

### UC-007 メタ情報取得

#### 基本フロー

1. クライアントが `GET /api/v1/meta` にリクエストする。
2. `200 OK` + APIバージョン情報をエンベロープで返す。

# 5. 外部仕様

## 5.1 API仕様

### エンドポイント一覧

| API ID | Method | Path | 認証 | 目的 |
| --- | --- | --- | --- | --- |
| API-001 | POST | `/api/v1/items` | none | アイテム作成 |
| API-002 | GET | `/api/v1/items` | none | アイテム一覧取得 |
| API-003 | GET | `/api/v1/items/{item_id}` | none | アイテム単体取得 |
| API-004 | PUT | `/api/v1/items/{item_id}` | none | アイテム更新 |
| API-005 | DELETE | `/api/v1/items/{item_id}` | none | アイテム削除 |
| API-006 | GET | `/health` | none | ヘルスチェック |
| API-007 | GET | `/api/v1/meta` | none | メタ情報取得 |

### 共通レスポンスエンベロープ

#### 成功レスポンス

```json
{
  "success": true,
  "data": "<リソース or リソース配列 or null>",
  "meta": "<メタ情報オブジェクト or null>"
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `success` | `bool` | yes | 常に `true` |
| `data` | `T \| list[T] \| null` | yes | レスポンスのペイロード。削除時は `null` |
| `meta` | `object \| null` | yes | ページネーション等のメタ情報。不要な場合は `null` |

#### エラーレスポンス

```json
{
  "success": false,
  "error": {
    "code": "NOT_FOUND",
    "message": "Item not found",
    "details": []
  }
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `success` | `bool` | yes | 常に `false` |
| `error.code` | `string` | yes | 機械可読なエラーコード（大文字スネークケース） |
| `error.message` | `string` | yes | 人間可読なエラーメッセージ |
| `error.details` | `list[object]` | yes | フィールド単位のエラー詳細。該当なしなら空配列 |

#### バリデーションエラーの details

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {"field": "name", "message": "Field required"},
      {"field": "price", "message": "Input should be a valid number"}
    ]
  }
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `details[].field` | `string` | yes | エラーが発生したフィールド名。ネストの場合はドット区切り（例: `address.zip`） |
| `details[].message` | `string` | yes | そのフィールドのエラーメッセージ |

#### ページネーション meta

```json
{
  "total": 42,
  "limit": 10,
  "offset": 0
}
```

| フィールド | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| `meta.total` | `int` | yes | フィルタ条件に合致する全件数 |
| `meta.limit` | `int` | yes | 今回の取得上限 |
| `meta.offset` | `int` | yes | 今回のオフセット |

### エラーコード一覧

| エラーコード | HTTPステータス | 発生条件 |
| --- | --- | --- |
| `VALIDATION_ERROR` | 422 | リクエストボディ / クエリパラメータのバリデーション失敗 |
| `NOT_FOUND` | 404 | 指定されたリソースが存在しない |
| `INTERNAL_ERROR` | 500 | サーバー内部エラー（詳細は隠蔽） |

### API詳細

#### API-001 アイテム作成

##### リクエスト

* Method: `POST`
* Path: `/api/v1/items`
* Content-Type: `application/json`

| 名前 | 場所 | 型 | 必須 | 制約 | 例 |
| --- | --- | --- | --- | --- | --- |
| `name` | body | `string` | yes | 1文字以上、255文字以下 | `"Widget"` |
| `description` | body | `string` | no | 1000文字以下。省略時は `null` | `"A useful widget"` |
| `price` | body | `number` | yes | 0以上の数値 | `19.99` |

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 201 | 作成成功 | 成功エンベロープ（`data` に作成されたアイテム） |
| 422 | バリデーションエラー | エラーエンベロープ（`VALIDATION_ERROR`） |

##### 例

* Request

```http
POST /api/v1/items HTTP/1.1
Content-Type: application/json

{
  "name": "Widget",
  "description": "A useful widget",
  "price": 19.99
}
```

* Response (201)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Widget",
    "description": "A useful widget",
    "price": 19.99,
    "created_at": "2026-02-18T10:00:00Z",
    "updated_at": "2026-02-18T10:00:00Z"
  },
  "meta": null
}
```

#### API-002 アイテム一覧取得

##### リクエスト

* Method: `GET`
* Path: `/api/v1/items`

| 名前 | 場所 | 型 | 必須 | 制約 | 例 |
| --- | --- | --- | --- | --- | --- |
| `limit` | query | `int` | no | 1〜100。デフォルト: `10` | `20` |
| `offset` | query | `int` | no | 0以上。デフォルト: `0` | `0` |
| `sort` | query | `string` | no | `field:direction` のカンマ区切り。`direction` は `asc` または `desc`。ソート可能フィールド: `name`, `price`, `created_at`, `updated_at`。デフォルト: ソートなし（挿入順） | `name:asc,price:desc` |
| `name` | query | `string` | no | 部分一致フィルタ（大文字小文字を区別しない） | `Widget` |

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 取得成功（0件含む） | 成功エンベロープ（`data` にアイテム配列、`meta` にページネーション） |
| 422 | クエリパラメータ不正 | エラーエンベロープ（`VALIDATION_ERROR`） |

##### ソートのバリデーションルール

* `sort` パラメータの形式: `field:direction` のカンマ区切り
* `field` はソート可能フィールド（`name`, `price`, `created_at`, `updated_at`）のみ許可
* `direction` は `asc` または `desc` のみ許可
* 不正フィールドまたは不正direction → `422` + `VALIDATION_ERROR`（`details` にどのフィールドが不正かを含む）

##### 例

* Request

```http
GET /api/v1/items?limit=10&offset=0&sort=name:asc&name=Widget HTTP/1.1
```

* Response (200)

```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Widget",
      "description": "A useful widget",
      "price": 19.99,
      "created_at": "2026-02-18T10:00:00Z",
      "updated_at": "2026-02-18T10:00:00Z"
    }
  ],
  "meta": {
    "total": 1,
    "limit": 10,
    "offset": 0
  }
}
```

#### API-003 アイテム単体取得

##### リクエスト

* Method: `GET`
* Path: `/api/v1/items/{item_id}`

| 名前 | 場所 | 型 | 必須 | 制約 | 例 |
| --- | --- | --- | --- | --- | --- |
| `item_id` | path | `uuid` | yes | UUID v4 形式 | `550e8400-e29b-41d4-a716-446655440000` |

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 取得成功 | 成功エンベロープ（`data` にアイテム） |
| 404 | アイテムが存在しない | エラーエンベロープ（`NOT_FOUND`） |
| 422 | `item_id` が UUID 形式でない | エラーエンベロープ（`VALIDATION_ERROR`） |

##### 例

* Request

```http
GET /api/v1/items/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
```

* Response (200)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Widget",
    "description": "A useful widget",
    "price": 19.99,
    "created_at": "2026-02-18T10:00:00Z",
    "updated_at": "2026-02-18T10:00:00Z"
  },
  "meta": null
}
```

#### API-004 アイテム更新

##### リクエスト

* Method: `PUT`
* Path: `/api/v1/items/{item_id}`
* Content-Type: `application/json`

| 名前 | 場所 | 型 | 必須 | 制約 | 例 |
| --- | --- | --- | --- | --- | --- |
| `item_id` | path | `uuid` | yes | UUID v4 形式 | `550e8400-e29b-41d4-a716-446655440000` |
| `name` | body | `string` | yes | 1文字以上、255文字以下 | `"Updated Widget"` |
| `description` | body | `string` | no | 1000文字以下。省略時は `null` | `"An updated widget"` |
| `price` | body | `number` | yes | 0以上の数値 | `29.99` |

PUT はリソースの完全置換とする。部分更新（PATCH）はスコープ外。

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 更新成功 | 成功エンベロープ（`data` に更新後アイテム） |
| 404 | アイテムが存在しない | エラーエンベロープ（`NOT_FOUND`） |
| 422 | バリデーションエラー | エラーエンベロープ（`VALIDATION_ERROR`） |

##### 例

* Request

```http
PUT /api/v1/items/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
Content-Type: application/json

{
  "name": "Updated Widget",
  "description": "An updated widget",
  "price": 29.99
}
```

* Response (200)

```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Updated Widget",
    "description": "An updated widget",
    "price": 29.99,
    "created_at": "2026-02-18T10:00:00Z",
    "updated_at": "2026-02-18T10:30:00Z"
  },
  "meta": null
}
```

#### API-005 アイテム削除

##### リクエスト

* Method: `DELETE`
* Path: `/api/v1/items/{item_id}`

| 名前 | 場所 | 型 | 必須 | 制約 | 例 |
| --- | --- | --- | --- | --- | --- |
| `item_id` | path | `uuid` | yes | UUID v4 形式 | `550e8400-e29b-41d4-a716-446655440000` |

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 削除成功 | `{"success": true, "data": null, "meta": null}` |
| 404 | アイテムが存在しない | エラーエンベロープ（`NOT_FOUND`） |
| 422 | `item_id` が UUID 形式でない | エラーエンベロープ（`VALIDATION_ERROR`） |

##### 例

* Request

```http
DELETE /api/v1/items/550e8400-e29b-41d4-a716-446655440000 HTTP/1.1
```

* Response (200)

```json
{
  "success": true,
  "data": null,
  "meta": null
}
```

#### API-006 ヘルスチェック

##### リクエスト

* Method: `GET`
* Path: `/health`

パラメータなし。

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 正常 | `{"status": "ok"}` |

ヘルスチェックはエンベロープを使用しない。軽量・シンプルであることを優先する。

##### 例

* Request

```http
GET /health HTTP/1.1
```

* Response (200)

```json
{
  "status": "ok"
}
```

#### API-007 メタ情報取得

##### リクエスト

* Method: `GET`
* Path: `/api/v1/meta`

パラメータなし。

##### レスポンス

| code | 条件 | ボディ |
| --- | --- | --- |
| 200 | 正常 | 成功エンベロープ（`data` にメタ情報） |

##### 例

* Request

```http
GET /api/v1/meta HTTP/1.1
```

* Response (200)

```json
{
  "success": true,
  "data": {
    "api_version": "v1",
    "app_name": "microservice-template",
    "app_version": "0.1.0"
  },
  "meta": null
}
```

# 6. データ設計

## 6.1 エンティティ一覧

| Entity | 用途 | 永続化 | 備考 |
| --- | --- | --- | --- |
| `Item` | サンプルCRUDリソース | インメモリ（dict） | テンプレートのサンプル。DB実装は各サービスで差し替え |

## 6.2 スキーマ

### Item

| 項目 | 型 | NULL | 一意 | 制約 | 説明 | 例 |
| --- | --- | --- | --- | --- | --- | --- |
| `id` | `uuid` | no | yes | UUID v4。サーバー側で採番 | アイテムの一意識別子 | `550e8400-...` |
| `name` | `string` | no | no | 1〜255文字 | アイテム名 | `"Widget"` |
| `description` | `string` | yes | no | 0〜1000文字 | アイテムの説明 | `"A useful widget"` |
| `price` | `float` | no | no | 0以上 | 価格 | `19.99` |
| `created_at` | `datetime` | no | no | ISO 8601 UTC。サーバー側で設定 | 作成日時 | `2026-02-18T10:00:00Z` |
| `updated_at` | `datetime` | no | no | ISO 8601 UTC。作成時・更新時にサーバー側で設定 | 更新日時 | `2026-02-18T10:00:00Z` |

## 6.3 既存データへの影響

* 新規テンプレートのため、既存データへの影響なし。

## 6.4 マイグレーション

* 該当なし（インメモリ実装のため）。

# 7. 振る舞い仕様

## 7.1 ビジネスルール

| Rule ID | 条件 | 処理 | 出力/副作用 | 優先順位 | 備考 |
| --- | --- | --- | --- | --- | --- |
| R-001 | POST リクエスト受信 | UUID v4 を採番し、`created_at` / `updated_at` を現在時刻 (UTC) で設定してリポジトリに保存 | 201 + 作成されたアイテム | 1 | |
| R-002 | GET 一覧リクエスト受信 | フィルタ・ソートを適用後、`offset` / `limit` でスライスし、全件数をカウント | 200 + アイテム配列 + メタ情報 | 1 | |
| R-003 | GET 単体リクエストで ID が存在 | リポジトリからアイテムを取得 | 200 + アイテム | 1 | |
| R-004 | GET 単体リクエストで ID が存在しない | 処理中断 | 404 + NOT_FOUND | 1 | |
| R-005 | PUT リクエストで ID が存在 | リクエストボディで全フィールドを上書き、`updated_at` を現在時刻に更新、`created_at` は変更しない | 200 + 更新後アイテム | 1 | |
| R-006 | PUT リクエストで ID が存在しない | 処理中断 | 404 + NOT_FOUND | 1 | PUT はupsertしない |
| R-007 | DELETE リクエストで ID が存在 | リポジトリからアイテムを削除 | 200 + `data: null` | 1 | |
| R-008 | DELETE リクエストで ID が存在しない | 処理中断 | 404 + NOT_FOUND | 1 | |
| R-009 | リクエストバリデーション失敗 | 処理中断 | 422 + VALIDATION_ERROR + details | 0 (最優先) | FastAPIの RequestValidationError をキャッチしてエンベロープに変換 |
| R-010 | 未ハンドルの例外 | ログ出力して処理中断 | 500 + INTERNAL_ERROR（詳細は隠蔽） | 0 (最優先) | スタックトレースはログのみ |

## 7.2 フィルタリング仕様

| フィルタパラメータ | 対象フィールド | マッチ方式 | 例 |
| --- | --- | --- | --- |
| `name` | `Item.name` | 部分一致（大文字小文字を区別しない） | `?name=wid` → `name` に `wid` を含むアイテム |

テンプレートではサンプルとして `name` のフィルタのみ提供する。各サービスでリソース固有のフィルタを追加する想定。

## 7.3 ソート仕様

* クエリパラメータ: `sort`
* 形式: `field:direction` のカンマ区切り
* `field`: ソート可能フィールド名（`name`, `price`, `created_at`, `updated_at`）
* `direction`: `asc`（昇順）または `desc`（降順）
* 複数フィールド指定時は左から優先順にソートする
* 省略時: ソートなし（挿入順）
* 例: `sort=price:asc,name:desc` → `price` 昇順でソートし、同値なら `name` 降順

## 7.4 並行性・整合性

* インメモリ実装のため、マルチプロセス間のデータ共有は保証しない。
* 単一プロセス + asyncio で動作する前提。uvicorn の worker 1 で動かす想定。
* DB実装に差し替えた場合の並行性はDB層の責務とし、テンプレートでは扱わない。

# 8. エラー処理

## 8.1 エラー一覧

| Error code | 発生条件 | 利用者への表示 | ログ | 再試行 | 備考 |
| --- | --- | --- | --- | --- | --- |
| `VALIDATION_ERROR` | リクエストのバリデーション失敗 | `"Request validation failed"` + `details` にフィールド単位のエラー | WARNING + リクエスト内容 | no | FastAPI の `RequestValidationError` をキャッチして変換 |
| `NOT_FOUND` | リソースが存在しない | `"Item not found"` または `"Resource not found"` | INFO + リソースID | no | |
| `INTERNAL_ERROR` | 未ハンドルの例外 | `"Internal server error"` | ERROR + スタックトレース | no | 詳細はクライアントに返さない |

## 8.2 例外と復旧

* **FastAPI の `RequestValidationError`**: グローバル例外ハンドラでキャッチし、エラーエンベロープに変換する。Pydantic のバリデーションエラーから `field` と `message` を抽出して `details` に格納。
* **アプリケーション例外 `ItemNotFoundError`**: `NOT_FOUND` 用のカスタム例外。グローバル例外ハンドラでキャッチして 404 エラーエンベロープを返す。
* **未ハンドル例外**: グローバル例外ハンドラ（`Exception` 型をキャッチ）で 500 エラーエンベロープを返す。スタックトレースはログに出力し、クライアントには返さない。
* **404 Not Found（FastAPI のルーティング不一致）**: FastAPI デフォルトの 404 をオーバーライドし、エラーエンベロープ形式で返す。

# 9. 非機能要件

## 9.1 性能

* テンプレートとしての性能目標は設けない。
* インメモリ実装のため、DB接続のボトルネックはない。

## 9.2 セキュリティ

* 認証: なし（テンプレートのスコープ外。各サービスで追加する）。
* 入力検証: Pydantic v2 によるリクエストボディバリデーション + クエリパラメータバリデーション。
* 500エラー時に内部情報（スタックトレース等）をクライアントに返さない。

## 9.3 可観測性

* ログ
  * Python 標準 `logging` を使用する。
  * デフォルトのログレベルは設定（`pydantic-settings`）で制御する。
  * フォーマット: JSON 構造化ログ（`timestamp`, `level`, `message`, `extra`）。
  * リクエスト/レスポンスのアクセスログは uvicorn に委譲する。

## 9.4 運用

* 構成値（環境変数）

| キー | 型 | 必須 | デフォルト | 説明 |
| --- | --- | --- | --- | --- |
| `APP_NAME` | `string` | no | `"microservice-template"` | アプリケーション名 |
| `APP_VERSION` | `string` | no | `"0.1.0"` | アプリケーションバージョン |
| `API_V1_PREFIX` | `string` | no | `"/api/v1"` | v1 API のパスプレフィックス |
| `DEFAULT_LIMIT` | `int` | no | `10` | ページネーションの limit デフォルト値 |
| `MAX_LIMIT` | `int` | no | `100` | ページネーションの limit 最大値 |
| `LOG_LEVEL` | `string` | no | `"INFO"` | ログレベル |
| `HOST` | `string` | no | `"0.0.0.0"` | バインドするホスト |
| `PORT` | `int` | no | `8000` | バインドするポート |

# 10. テスト設計

## 10.1 テスト方針

* テスト種類: 結合テスト（FastAPI TestClient を使ったHTTPレベルのテスト）
* モック方針: リポジトリ層をインメモリ実装のテスト用インスタンスとして使用する（外部依存がないため、モックは不要）。テストごとにリポジトリの状態をリセットする。

## 10.2 テストケース一覧

| TC ID | 対象 | 観点 | 前提 | 入力 | 期待結果 | 優先度 |
| --- | --- | --- | --- | --- | --- | --- |
| TC-001 | API-001 | normal | なし | 有効なリクエストボディ | 201 + 作成アイテム + UUID付与 + タイムスタンプ付与 | high |
| TC-002 | API-001 | error | なし | `name` 欠落 | 422 + VALIDATION_ERROR + details に `name` のエラー | high |
| TC-003 | API-001 | error | なし | `price` が負値 | 422 + VALIDATION_ERROR + details に `price` のエラー | high |
| TC-004 | API-001 | boundary | なし | `name` が空文字 | 422 + VALIDATION_ERROR | med |
| TC-005 | API-001 | boundary | なし | `name` が255文字ちょうど | 201 + 正常作成 | med |
| TC-006 | API-001 | boundary | なし | `name` が256文字 | 422 + VALIDATION_ERROR | med |
| TC-007 | API-001 | boundary | なし | `description` 省略 | 201 + `description: null` | med |
| TC-008 | API-002 | normal | アイテム3件存在 | `limit=10&offset=0` | 200 + 3件 + `meta.total=3` | high |
| TC-009 | API-002 | normal | アイテム0件 | パラメータなし | 200 + `data=[]` + `meta.total=0` | high |
| TC-010 | API-002 | normal | アイテム15件 | `limit=10&offset=0` | 200 + 10件 + `meta.total=15` | high |
| TC-011 | API-002 | normal | アイテム15件 | `limit=10&offset=10` | 200 + 5件 + `meta.total=15` | high |
| TC-012 | API-002 | boundary | アイテム5件 | `offset=100` | 200 + `data=[]` + `meta.total=5` | med |
| TC-013 | API-002 | error | なし | `limit=0` | 422 + VALIDATION_ERROR | high |
| TC-014 | API-002 | error | なし | `limit=101` | 422 + VALIDATION_ERROR | med |
| TC-015 | API-002 | error | なし | `offset=-1` | 422 + VALIDATION_ERROR | med |
| TC-016 | API-002 | normal | アイテム3件 (A, B, C) | `sort=name:asc` | 200 + name昇順で並んだ配列 | high |
| TC-017 | API-002 | normal | アイテム3件 | `sort=price:desc,name:asc` | 200 + price降順→name昇順でソート | med |
| TC-018 | API-002 | error | なし | `sort=invalid_field:asc` | 422 + VALIDATION_ERROR | med |
| TC-019 | API-002 | error | なし | `sort=name:invalid` | 422 + VALIDATION_ERROR | med |
| TC-020 | API-002 | normal | アイテム3件 (Widget, Gadget, Widget Pro) | `name=Widget` | 200 + Widget, Widget Pro の2件 | high |
| TC-021 | API-002 | normal | フィルタ + ソート + ページネーション組み合わせ | `name=Widget&sort=price:asc&limit=1&offset=0` | 200 + 条件に合致した結果 | med |
| TC-022 | API-003 | normal | アイテム存在 | 有効な UUID | 200 + アイテム | high |
| TC-023 | API-003 | error | アイテム不存在 | 存在しない UUID | 404 + NOT_FOUND | high |
| TC-024 | API-003 | error | なし | 不正な UUID 形式 | 422 + VALIDATION_ERROR | med |
| TC-025 | API-004 | normal | アイテム存在 | 有効なリクエストボディ | 200 + 更新後アイテム + `updated_at` が更新済み + `created_at` は不変 | high |
| TC-026 | API-004 | error | アイテム不存在 | 有効なリクエストボディ | 404 + NOT_FOUND | high |
| TC-027 | API-004 | error | なし | 不正なリクエストボディ | 422 + VALIDATION_ERROR | high |
| TC-028 | API-005 | normal | アイテム存在 | 有効な UUID | 200 + `data: null` | high |
| TC-029 | API-005 | error | アイテム不存在 | 存在しない UUID | 404 + NOT_FOUND | high |
| TC-030 | API-005 | normal | アイテム存在 → 削除済み | 同じ UUID を再度 DELETE | 404 + NOT_FOUND | med |
| TC-031 | API-006 | normal | なし | なし | 200 + `{"status": "ok"}` | high |
| TC-032 | API-007 | normal | なし | なし | 200 + バージョン情報エンベロープ | high |
| TC-033 | 全API | normal | なし | 存在しないパス | 404 + NOT_FOUND エラーエンベロープ | med |
| TC-034 | 全API | error | 内部エラーを発生させる | なし | 500 + INTERNAL_ERROR（詳細なし） | med |

## 10.3 境界値・データパターン

* `name`: 空文字（不可）、1文字（最小）、255文字（最大）、256文字（超過）
* `description`: `null`（省略）、空文字（許可）、1000文字（最大）、1001文字（超過）
* `price`: `0`（最小許可値）、`0.01`（正の最小値）、負値（不可）
* `limit`: `0`（不可）、`1`（最小）、`100`（最大）、`101`（超過）
* `offset`: `-1`（不可）、`0`（最小）、大きな値（0件返却）
* `item_id`: 有効な UUID、不正な形式（例: `"abc"`）、存在しない UUID

# 11. 影響範囲

## 11.1 変更点サマリ

新規テンプレートリポジトリの構築であり、既存機能への影響はない。

| 分類 | 対象 | 変更内容 | 互換性 |
| --- | --- | --- | --- |
| プロジェクト | `pyproject.toml` | 新規作成 | — |
| アプリ | `app/` 以下全体 | 新規作成 | — |
| テスト | `tests/` 以下全体 | 新規作成 | — |
| コンテナ | `Dockerfile`, `docker-compose.yml` | 新規作成 | — |
| ドキュメント | `README.md` | 新規作成 | — |

## 11.2 影響を受ける機能

* なし（新規テンプレート）

## 11.3 デグレ防止

* 新規テンプレートのため、デグレリスクなし。テストスイートを初期から含めることで、今後の変更に対するデグレ防止基盤を構築する。

# 12. 実装指示 (AIエージェント向け)

## 12.1 変更するファイルと責務

| ファイル | 責務 | 変更内容 |
| --- | --- | --- |
| `pyproject.toml` | プロジェクト定義・依存管理 | uv プロジェクト設定、FastAPI / pydantic-settings / uvicorn / pytest / httpx の依存を定義 |
| `app/__init__.py` | パッケージ初期化 | 空ファイル |
| `app/main.py` | FastAPI アプリ初期化 | アプリ生成、ルーター登録、グローバル例外ハンドラ（`RequestValidationError` → 422エンベロープ、`AppError` → 対応エンベロープ、`Exception` → 500エンベロープ）、ヘルスチェックエンドポイント |
| `app/config.py` | 設定管理 | `pydantic-settings` の `BaseSettings` で環境変数を読み込む。`APP_NAME`, `APP_VERSION`, `API_V1_PREFIX`, `DEFAULT_LIMIT`, `MAX_LIMIT`, `LOG_LEVEL`, `HOST`, `PORT` |
| `app/schemas/__init__.py` | パッケージ初期化 | 空ファイル |
| `app/schemas/envelope.py` | 共通エンベロープ定義 | `SuccessResponse[T]`, `ErrorResponse`, `ErrorDetail`, `PaginationMeta` の Pydantic モデル |
| `app/schemas/item.py` | Item リソースのスキーマ | `ItemCreate`（入力）、`ItemUpdate`（入力）、`ItemResponse`（出力）の Pydantic モデル |
| `app/models/__init__.py` | パッケージ初期化 | 空ファイル |
| `app/models/item.py` | Item ドメインモデル | `Item` dataclass（`id`, `name`, `description`, `price`, `created_at`, `updated_at`） |
| `app/repositories/__init__.py` | パッケージ初期化 | 空ファイル |
| `app/repositories/base.py` | リポジトリインターフェース | `ItemRepository` Protocol（`create`, `get_by_id`, `get_list`, `update`, `delete`, `count`） |
| `app/repositories/memory.py` | インメモリリポジトリ実装 | dict ベースの `InMemoryItemRepository` |
| `app/api/__init__.py` | パッケージ初期化 | 空ファイル |
| `app/api/v1/__init__.py` | v1 ルーター集約 | v1 ルーターを束ねる `APIRouter` |
| `app/api/v1/items.py` | items リソースルーター | CRUD 5エンドポイント |
| `app/api/v1/meta.py` | メタ情報ルーター | `/meta` エンドポイント |
| `app/exceptions.py` | カスタム例外 | `AppError`（基底）、`NotFoundError` |
| `app/dependencies.py` | DI定義 | リポジトリのインスタンスを提供する `get_item_repository` |
| `Dockerfile` | コンテナイメージ | Python 3.13-slim ベース、uv install、uvicorn 起動 |
| `docker-compose.yml` | ローカル開発用 | アプリサービス定義、ポートマッピング、環境変数 |
| `tests/__init__.py` | パッケージ初期化 | 空ファイル |
| `tests/conftest.py` | テスト共通設定 | TestClient fixture（テストごとにリポジトリリセット） |
| `tests/api/__init__.py` | パッケージ初期化 | 空ファイル |
| `tests/api/v1/__init__.py` | パッケージ初期化 | 空ファイル |
| `tests/api/v1/test_items.py` | items CRUD テスト | TC-001〜TC-030 の結合テスト |
| `tests/test_health.py` | ヘルスチェックテスト | TC-031 |
| `tests/api/v1/test_meta.py` | メタ情報テスト | TC-032 |
| `README.md` | テンプレートの使い方 | セットアップ手順、API仕様概要、カスタマイズ方法 |

## 12.2 手順の境界

* このPRで行うこと: 上記ファイルすべての新規作成
* このPRでは行わないこと: 認証・認可、DB接続、CI/CD、PATCH対応

## 12.3 ビルド・テスト・検証

* ローカルで実行するコマンド

```sh
# 依存インストール
uv sync

# テスト実行
uv run pytest -v

# ローカル起動
uv run uvicorn app.main:app --reload

# Docker ビルド・起動
docker compose up --build
```

* 手動確認手順

| 手順 | 操作 | 期待結果 |
| --- | --- | --- |
| 1 | `GET /health` | `200` + `{"status": "ok"}` |
| 2 | `POST /api/v1/items` に `{"name": "Test", "price": 10}` | `201` + 作成されたアイテム（UUID, タイムスタンプ付き） |
| 3 | `GET /api/v1/items` | `200` + 1件のアイテム + `meta.total=1` |
| 4 | `GET /api/v1/items/{id}` (手順2のID) | `200` + 該当アイテム |
| 5 | `PUT /api/v1/items/{id}` に `{"name": "Updated", "price": 20}` | `200` + 更新後アイテム |
| 6 | `DELETE /api/v1/items/{id}` | `200` + `{"success": true, "data": null, "meta": null}` |
| 7 | `GET /api/v1/items/{id}` (削除済み) | `404` + NOT_FOUND エラーエンベロープ |
| 8 | `POST /api/v1/items` に `{}` (空ボディ) | `422` + VALIDATION_ERROR + details |
| 9 | `GET /api/v1/meta` | `200` + バージョン情報 |
| 10 | `GET /nonexistent` | `404` + エラーエンベロープ |

# 13. 受け入れ条件

| AC ID | Given | When | Then |
| --- | --- | --- | --- |
| AC-001 | テンプレートをクローン済み | `uv sync` → `uv run uvicorn app.main:app` を実行 | サーバーが起動し、`/health` が `200` を返す |
| AC-002 | サーバー起動済み | `POST /api/v1/items` に有効なボディを送信 | `201` + 作成されたアイテム（UUID, created_at, updated_at 付き）がエンベロープで返る |
| AC-003 | アイテムが複数件存在 | `GET /api/v1/items?limit=10&offset=0` を送信 | `200` + アイテム配列 + `meta` に `total`, `limit`, `offset` が含まれる |
| AC-004 | アイテムが存在 | `GET /api/v1/items/{id}` を送信 | `200` + 該当アイテムがエンベロープで返る |
| AC-005 | アイテムが存在 | `PUT /api/v1/items/{id}` に有効なボディを送信 | `200` + 更新後アイテムがエンベロープで返る。`updated_at` が更新され、`created_at` は不変 |
| AC-006 | アイテムが存在 | `DELETE /api/v1/items/{id}` を送信 | `200` + `{"success": true, "data": null, "meta": null}` |
| AC-007 | なし | 不正なリクエストボディでPOSTを送信 | `422` + `VALIDATION_ERROR` + `details` にフィールド単位のエラー |
| AC-008 | アイテム不存在 | `GET /api/v1/items/{id}` を送信 | `404` + `NOT_FOUND` エラーエンベロープ |
| AC-009 | アイテムが複数件存在 | `GET /api/v1/items?sort=name:asc` を送信 | `200` + `name` 昇順でソートされた配列 |
| AC-010 | アイテムが複数件存在 | `GET /api/v1/items?name=Widget` を送信 | `200` + `name` に `Widget` を含むアイテムのみ。`meta.total` はフィルタ後の件数 |
| AC-011 | なし | `GET /api/v1/meta` を送信 | `200` + `api_version`, `app_name`, `app_version` を含むエンベロープ |
| AC-012 | なし | `GET /health` を送信 | `200` + `{"status": "ok"}`（エンベロープなし） |
| AC-013 | なし | `GET /api/v1/items?limit=0` を送信 | `422` + `VALIDATION_ERROR` |
| AC-014 | なし | Dockerfile でビルド・起動 | コンテナ内でサーバーが起動し、全エンドポイントが動作する |
| AC-015 | なし | `uv run pytest` を実行 | 全テストがパスする |

# 14. 未決事項

| 項目 | 決めること | 期限 | 担当 | 影響 |
| --- | --- | --- | --- | --- |
| — | — | — | — | — |

すべての未確定事項はIntake時のQ&Aで解消済み。

# 15. 変更履歴

| 日付 | 版 | 変更者 | 変更内容 |
| --- | --- | --- | --- |
| 2026-02-18 | v0.1 | motoki | 初版作成 |
