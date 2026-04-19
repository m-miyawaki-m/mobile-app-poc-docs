# mobile-app-poc-api-spec Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** OpenAPI 3.0.3 契約リポジトリ `mobile-app-poc-api-spec` をscaffoldし、Spectral lintが通る状態で初期コミットまで完了させる。

**Architecture:** YAML骨格 + JSON外出しの2層構造。`openapi.yaml` は paths構造と`$ref`/`externalValue`のみ保持し、スキーマ・再利用レスポンス・パラメータ・exampleは全てJSONに切り出す。初回構築後はYAML編集がほぼ不要になる。

**Tech Stack:** OpenAPI 3.0.3, Node.js 18+, `@stoplight/spectral-cli` 6.x, PowerShell/bash (Windows native)

---

## File Structure

Working directory: `C:\Oracle\3df002\mobile-app\mobile-app-poc-api-spec\`

**Created by this plan:**

```
mobile-app-poc-api-spec/
├─ openapi.yaml                    # 骨格（paths + $ref/externalValue）
├─ package.json                    # npm + Spectral依存
├─ .spectral.yaml                  # lint設定
├─ .gitignore
├─ .gitattributes
├─ README.md
├─ schemas/
│  ├─ Item.json
│  ├─ NewItem.json
│  ├─ UpdateItem.json
│  ├─ LoginRequest.json
│  ├─ LoginResponse.json
│  └─ Error.json
├─ responses/
│  ├─ BadRequest.json
│  ├─ Unauthorized.json
│  ├─ NotFound.json
│  └─ InternalServerError.json
├─ parameters/
│  └─ ItemIdPath.json
└─ examples/
   ├─ items-list.json
   ├─ items-empty.json
   ├─ item-single.json
   ├─ item-created.json
   ├─ login-success.json
   └─ errors/
      ├─ validation-400.json
      ├─ unauthorized-401.json
      ├─ not-found-404.json
      └─ internal-500.json
```

Each file has one clear responsibility: schema definitions are pure data shape, responses are HTTP response wrappers, parameters are path param definitions, examples are concrete instance data.

## Testing Approach

Since this is a contract spec (not executable code), "tests" = `npm run lint` (Spectral) passing with zero errors. Every task that adds a new file verifies by running lint. Additionally, we will run a manual ref-resolution check at the end.

**Critical**: This project has no `git` repository until the final task. **Do not attempt `git add`/`git commit` mid-way**. All files accumulate, and a single initial commit happens at the end (Task 10).

---

## Task 1: プロジェクトディレクトリと package.json の作成

**Files:**
- Create: `mobile-app-poc-api-spec/package.json`
- Create: `mobile-app-poc-api-spec/` (directory)

- [ ] **Step 1: ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec"
```

- [ ] **Step 2: package.json を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/package.json`

```json
{
  "name": "mobile-app-poc-api-spec",
  "version": "0.1.0",
  "private": true,
  "description": "OpenAPI 3.0.3 API contract (SSOT) for mobile-app-poc",
  "scripts": {
    "lint": "spectral lint openapi.yaml"
  },
  "devDependencies": {
    "@stoplight/spectral-cli": "^6.11.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

- [ ] **Step 3: 依存インストール**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && npm install
```

Expected: `added N packages` 出力、`node_modules/` と `package-lock.json` が生成される。

- [ ] **Step 4: Spectral CLI が動くことを確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && npx spectral --version
```

Expected: バージョン番号（例: `6.11.1`）が表示される。

---

## Task 2: .gitignore / .gitattributes の作成

**Files:**
- Create: `mobile-app-poc-api-spec/.gitignore`
- Create: `mobile-app-poc-api-spec/.gitattributes`

- [ ] **Step 1: .gitignore を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/.gitignore`

```
node_modules/
captures/
*.log
.DS_Store
```

- [ ] **Step 2: .gitattributes を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/.gitattributes`

```
*.yaml text eol=lf
*.yml  text eol=lf
*.json text eol=lf
*.md   text eol=lf
```

---

## Task 3: スキーマJSONファイルの作成

**Files:**
- Create: `mobile-app-poc-api-spec/schemas/Item.json`
- Create: `mobile-app-poc-api-spec/schemas/NewItem.json`
- Create: `mobile-app-poc-api-spec/schemas/UpdateItem.json`
- Create: `mobile-app-poc-api-spec/schemas/LoginRequest.json`
- Create: `mobile-app-poc-api-spec/schemas/LoginResponse.json`
- Create: `mobile-app-poc-api-spec/schemas/Error.json`

- [ ] **Step 1: schemas ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/schemas"
```

- [ ] **Step 2: Item.json を作成**

Path: `.../schemas/Item.json`

```json
{
  "type": "object",
  "required": ["id", "name", "quantity", "createdAt", "updatedAt"],
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "サーバー採番のUUID"
    },
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "quantity": {
      "type": "integer",
      "minimum": 0
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    },
    "updatedAt": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

- [ ] **Step 3: NewItem.json を作成（POST入力）**

Path: `.../schemas/NewItem.json`

```json
{
  "type": "object",
  "required": ["name", "quantity"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "quantity": {
      "type": "integer",
      "minimum": 0
    }
  }
}
```

- [ ] **Step 4: UpdateItem.json を作成（PUT入力、全置換）**

Path: `.../schemas/UpdateItem.json`

```json
{
  "type": "object",
  "required": ["name", "quantity"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "quantity": {
      "type": "integer",
      "minimum": 0
    }
  }
}
```

- [ ] **Step 5: LoginRequest.json を作成**

Path: `.../schemas/LoginRequest.json`

```json
{
  "type": "object",
  "required": ["username", "password"],
  "properties": {
    "username": {
      "type": "string",
      "minLength": 1
    },
    "password": {
      "type": "string",
      "minLength": 1
    }
  }
}
```

- [ ] **Step 6: LoginResponse.json を作成**

Path: `.../schemas/LoginResponse.json`

```json
{
  "type": "object",
  "required": ["token", "expiresIn"],
  "properties": {
    "token": {
      "type": "string",
      "description": "JWT access token"
    },
    "expiresIn": {
      "type": "integer",
      "description": "有効期限（秒）"
    }
  }
}
```

- [ ] **Step 7: Error.json を作成**

Path: `.../schemas/Error.json`

```json
{
  "type": "object",
  "required": ["code", "message", "timestamp"],
  "properties": {
    "code": {
      "type": "string",
      "description": "マシン可読なエラーコード",
      "example": "ITEM_NOT_FOUND"
    },
    "message": {
      "type": "string",
      "description": "ユーザー向けメッセージ（日本語）"
    },
    "timestamp": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

---

## Task 4: 再利用レスポンス定義の作成

**Files:**
- Create: `mobile-app-poc-api-spec/responses/BadRequest.json`
- Create: `mobile-app-poc-api-spec/responses/Unauthorized.json`
- Create: `mobile-app-poc-api-spec/responses/NotFound.json`
- Create: `mobile-app-poc-api-spec/responses/InternalServerError.json`

これらは OpenAPI の `components.responses` に入るレスポンスオブジェクト。`$ref` でパス側から参照する。

- [ ] **Step 1: responses ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/responses"
```

- [ ] **Step 2: BadRequest.json を作成**

Path: `.../responses/BadRequest.json`

```json
{
  "description": "リクエスト内容が不正（バリデーションエラー等）",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "../schemas/Error.json"
      },
      "examples": {
        "validation": {
          "externalValue": "../examples/errors/validation-400.json"
        }
      }
    }
  }
}
```

- [ ] **Step 3: Unauthorized.json を作成**

Path: `.../responses/Unauthorized.json`

```json
{
  "description": "認証が必要、または認証情報が不正",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "../schemas/Error.json"
      },
      "examples": {
        "unauthorized": {
          "externalValue": "../examples/errors/unauthorized-401.json"
        }
      }
    }
  }
}
```

- [ ] **Step 4: NotFound.json を作成**

Path: `.../responses/NotFound.json`

```json
{
  "description": "指定リソースが存在しない",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "../schemas/Error.json"
      },
      "examples": {
        "notFound": {
          "externalValue": "../examples/errors/not-found-404.json"
        }
      }
    }
  }
}
```

- [ ] **Step 5: InternalServerError.json を作成**

Path: `.../responses/InternalServerError.json`

```json
{
  "description": "サーバー内部エラー",
  "content": {
    "application/json": {
      "schema": {
        "$ref": "../schemas/Error.json"
      },
      "examples": {
        "internal": {
          "externalValue": "../examples/errors/internal-500.json"
        }
      }
    }
  }
}
```

---

## Task 5: 共通パラメータ定義の作成

**Files:**
- Create: `mobile-app-poc-api-spec/parameters/ItemIdPath.json`

- [ ] **Step 1: parameters ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/parameters"
```

- [ ] **Step 2: ItemIdPath.json を作成**

Path: `.../parameters/ItemIdPath.json`

```json
{
  "name": "id",
  "in": "path",
  "required": true,
  "description": "アイテムID（UUID）",
  "schema": {
    "type": "string",
    "format": "uuid"
  }
}
```

---

## Task 6: example JSON の作成

**Files:**
- Create: `mobile-app-poc-api-spec/examples/items-list.json`
- Create: `mobile-app-poc-api-spec/examples/items-empty.json`
- Create: `mobile-app-poc-api-spec/examples/item-single.json`
- Create: `mobile-app-poc-api-spec/examples/item-created.json`
- Create: `mobile-app-poc-api-spec/examples/login-success.json`
- Create: `mobile-app-poc-api-spec/examples/errors/validation-400.json`
- Create: `mobile-app-poc-api-spec/examples/errors/unauthorized-401.json`
- Create: `mobile-app-poc-api-spec/examples/errors/not-found-404.json`
- Create: `mobile-app-poc-api-spec/examples/errors/internal-500.json`

- [ ] **Step 1: examples ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/examples/errors"
```

- [ ] **Step 2: items-list.json を作成（正常・複数件）**

Path: `.../examples/items-list.json`

```json
[
  {
    "id": "11111111-1111-1111-1111-111111111111",
    "name": "サンプルアイテムA",
    "quantity": 10,
    "createdAt": "2026-04-20T10:00:00Z",
    "updatedAt": "2026-04-20T10:00:00Z"
  },
  {
    "id": "22222222-2222-2222-2222-222222222222",
    "name": "サンプルアイテムB",
    "quantity": 5,
    "createdAt": "2026-04-20T10:05:00Z",
    "updatedAt": "2026-04-20T10:05:00Z"
  },
  {
    "id": "33333333-3333-3333-3333-333333333333",
    "name": "サンプルアイテムC",
    "quantity": 0,
    "createdAt": "2026-04-20T10:10:00Z",
    "updatedAt": "2026-04-20T10:10:00Z"
  }
]
```

- [ ] **Step 3: items-empty.json を作成（正常・空配列）**

Path: `.../examples/items-empty.json`

```json
[]
```

- [ ] **Step 4: item-single.json を作成（単体取得・更新の正常例）**

Path: `.../examples/item-single.json`

```json
{
  "id": "11111111-1111-1111-1111-111111111111",
  "name": "サンプルアイテムA",
  "quantity": 10,
  "createdAt": "2026-04-20T10:00:00Z",
  "updatedAt": "2026-04-20T10:00:00Z"
}
```

- [ ] **Step 5: item-created.json を作成（POST 201 の例）**

Path: `.../examples/item-created.json`

```json
{
  "id": "44444444-4444-4444-4444-444444444444",
  "name": "新規アイテム",
  "quantity": 3,
  "createdAt": "2026-04-20T11:00:00Z",
  "updatedAt": "2026-04-20T11:00:00Z"
}
```

- [ ] **Step 6: login-success.json を作成**

Path: `.../examples/login-success.json`

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJkZW1vLXVzZXIifQ.SAMPLE_SIGNATURE",
  "expiresIn": 3600
}
```

- [ ] **Step 7: errors/validation-400.json を作成**

Path: `.../examples/errors/validation-400.json`

```json
{
  "code": "VALIDATION_ERROR",
  "message": "name は必須項目です",
  "timestamp": "2026-04-20T10:15:30Z"
}
```

- [ ] **Step 8: errors/unauthorized-401.json を作成**

Path: `.../examples/errors/unauthorized-401.json`

```json
{
  "code": "UNAUTHORIZED",
  "message": "認証が必要です",
  "timestamp": "2026-04-20T10:15:30Z"
}
```

- [ ] **Step 9: errors/not-found-404.json を作成**

Path: `.../examples/errors/not-found-404.json`

```json
{
  "code": "ITEM_NOT_FOUND",
  "message": "指定されたアイテムが見つかりません",
  "timestamp": "2026-04-20T10:15:30Z"
}
```

- [ ] **Step 10: errors/internal-500.json を作成**

Path: `.../examples/errors/internal-500.json`

```json
{
  "code": "INTERNAL_ERROR",
  "message": "サーバー内部エラーが発生しました",
  "timestamp": "2026-04-20T10:15:30Z"
}
```

---

## Task 7: openapi.yaml の作成

**Files:**
- Create: `mobile-app-poc-api-spec/openapi.yaml`

- [ ] **Step 1: openapi.yaml を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/openapi.yaml`

```yaml
openapi: 3.0.3
info:
  title: mobile-app-poc API
  version: 0.1.0
  description: |
    モバイルアプリPoC用のAPI契約（SSOT）。
    データ定義はすべて schemas/, responses/, parameters/, examples/ 配下のJSONに外出ししている。

servers:
  - url: http://localhost:4010
    description: Prism モックサーバー（開発用）
  - url: http://localhost:8080
    description: Spring Boot バックエンド（開発用）

security:
  - bearerAuth: []

paths:
  /auth/login:
    post:
      operationId: login
      tags: [auth]
      summary: ログインしJWTを発行
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: './schemas/LoginRequest.json'
      responses:
        '200':
          description: 認証成功
          content:
            application/json:
              schema:
                $ref: './schemas/LoginResponse.json'
              examples:
                success:
                  externalValue: './examples/login-success.json'
        '400':
          $ref: './responses/BadRequest.json'
        '401':
          $ref: './responses/Unauthorized.json'
        '500':
          $ref: './responses/InternalServerError.json'

  /items:
    get:
      operationId: listItems
      tags: [items]
      summary: アイテム一覧取得
      responses:
        '200':
          description: 一覧取得成功
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: './schemas/Item.json'
              examples:
                list:
                  externalValue: './examples/items-list.json'
                empty:
                  externalValue: './examples/items-empty.json'
        '401':
          $ref: './responses/Unauthorized.json'
        '500':
          $ref: './responses/InternalServerError.json'
    post:
      operationId: createItem
      tags: [items]
      summary: アイテム新規作成
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: './schemas/NewItem.json'
      responses:
        '201':
          description: 作成成功
          content:
            application/json:
              schema:
                $ref: './schemas/Item.json'
              examples:
                created:
                  externalValue: './examples/item-created.json'
        '400':
          $ref: './responses/BadRequest.json'
        '401':
          $ref: './responses/Unauthorized.json'
        '500':
          $ref: './responses/InternalServerError.json'

  /items/{id}:
    parameters:
      - $ref: './parameters/ItemIdPath.json'
    get:
      operationId: getItem
      tags: [items]
      summary: アイテム単体取得
      responses:
        '200':
          description: 取得成功
          content:
            application/json:
              schema:
                $ref: './schemas/Item.json'
              examples:
                single:
                  externalValue: './examples/item-single.json'
        '401':
          $ref: './responses/Unauthorized.json'
        '404':
          $ref: './responses/NotFound.json'
        '500':
          $ref: './responses/InternalServerError.json'
    put:
      operationId: updateItem
      tags: [items]
      summary: アイテム更新（全置換）
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: './schemas/UpdateItem.json'
      responses:
        '200':
          description: 更新成功
          content:
            application/json:
              schema:
                $ref: './schemas/Item.json'
              examples:
                updated:
                  externalValue: './examples/item-single.json'
        '400':
          $ref: './responses/BadRequest.json'
        '401':
          $ref: './responses/Unauthorized.json'
        '404':
          $ref: './responses/NotFound.json'
        '500':
          $ref: './responses/InternalServerError.json'
    delete:
      operationId: deleteItem
      tags: [items]
      summary: アイテム削除
      responses:
        '204':
          description: 削除成功（ボディなし）
        '401':
          $ref: './responses/Unauthorized.json'
        '404':
          $ref: './responses/NotFound.json'
        '500':
          $ref: './responses/InternalServerError.json'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

---

## Task 8: .spectral.yaml の作成と lint 実行

**Files:**
- Create: `mobile-app-poc-api-spec/.spectral.yaml`

- [ ] **Step 1: .spectral.yaml を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/.spectral.yaml`

```yaml
extends:
  - "spectral:oas"
```

- [ ] **Step 2: lint を実行して違反ゼロを確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && npm run lint
```

Expected: 違反ゼロで正常終了（終了コード0）。出力例:
```
> spectral lint openapi.yaml
No results with a severity of 'error' found!
```

**違反が出た場合の対処**:
- `$ref` のパスが間違っている → ファイル位置と相対パスを確認
- `externalValue` のJSONが不正 → 対応するexampleファイルのJSON構文を確認
- schema名の誤参照 → `schemas/` 配下のファイル名とYAML内の参照を突合
- 該当行を修正 → 再度 `npm run lint` で違反ゼロを確認

- [ ] **Step 3: 手動で $ref 解決を確認**

全ての `$ref` と `externalValue` に対応するファイルが存在することを確認:

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && ls schemas/ responses/ parameters/ examples/ examples/errors/
```

Expected: 以下のファイルがすべて存在:
- `schemas/`: Item.json, NewItem.json, UpdateItem.json, LoginRequest.json, LoginResponse.json, Error.json
- `responses/`: BadRequest.json, Unauthorized.json, NotFound.json, InternalServerError.json
- `parameters/`: ItemIdPath.json
- `examples/`: items-list.json, items-empty.json, item-single.json, item-created.json, login-success.json
- `examples/errors/`: validation-400.json, unauthorized-401.json, not-found-404.json, internal-500.json

---

## Task 9: README.md の作成

**Files:**
- Create: `mobile-app-poc-api-spec/README.md`

- [ ] **Step 1: README.md を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec/README.md`

````markdown
# mobile-app-poc-api-spec

モバイルアプリPoCの **API契約（Single Source of Truth）** を管理するリポジトリ。
OpenAPI 3.0.3 形式の `openapi.yaml` を中心に、スキーマ定義・レスポンス例・再利用レスポンスをJSONに分離管理する。

## 位置づけ

モバイルアプリPoCの5リポジトリのうち、他4リポジトリに対して契約を供給する上流リポジトリ:

- **mobile-app-poc-api-spec** ← 本リポジトリ
- mobile-app-poc-mock — 本リポジトリの `openapi.yaml` を Prism で起動
- mobile-app-poc-frontend — 本リポジトリから TypeScript型を自動生成
- mobile-app-poc-bridge
- mobile-app-poc-backend — 本リポジトリから Spring Boot Controller雛形を自動生成

## 前提

- Node.js `>=18` 必須（LTS 22.x 推奨）
- OS: Windows ネイティブ（PowerShell 前提）
- 社内イントラネット内での運用のみ

## セットアップ

```powershell
npm install
```

## lint 実行

```powershell
npm run lint
```

`openapi.yaml` を Spectral でチェック。違反があると非ゼロ終了。

## ディレクトリ構成

```
mobile-app-poc-api-spec/
├─ openapi.yaml          # 骨格（paths + $ref/externalValue）
├─ schemas/              # データ型定義（$ref 参照元）
├─ responses/            # 再利用レスポンス定義（$ref 参照元）
├─ parameters/           # 共通パラメータ（$ref 参照元）
├─ examples/             # レスポンス例（externalValue 参照元）
│  └─ errors/            # エラー例
├─ .spectral.yaml        # lint設定（extends: spectral:oas）
└─ package.json
```

## 設計原則

**`openapi.yaml` は骨格のみ保持し、日常的な変更は全てJSONで完結させる。**

| 変更タイプ | YAML編集 | JSON編集 |
|---|---|---|
| スキーマフィールド追加・変更 | なし | `schemas/*.json` |
| エラーコード追加 | なし | `schemas/Error.json`（enum） |
| example追加・差替 | なし | `examples/*.json` |
| エラー文言変更 | なし | `examples/errors/*.json` |
| 新規エンドポイント追加 | あり | schemas/responses/examples 追加 |

YAMLの手編集は事故が起きやすいため、初回構築後は最小化する方針。

## エンドポイント一覧

| Method | Path | 認証 |
|---|---|---|
| POST | `/auth/login` | 不要 |
| GET | `/items` | 必要 |
| POST | `/items` | 必要 |
| GET | `/items/{id}` | 必要 |
| PUT | `/items/{id}` | 必要 |
| DELETE | `/items/{id}` | 必要 |

認証: `Authorization: Bearer <JWT>` ヘッダ。

## 破壊的変更時の注意

`openapi.yaml` への破壊的変更（既存エンドポイント削除、必須フィールド追加、型変更等）は、mock/frontend/backend 全てに影響する。変更時は:

1. 変更内容を本リポジトリの lint に通す
2. 影響を受ける下流リポジトリ（mock/frontend/backend）に告知
3. 各リポジトリで型再生成・動作確認
````

---

## Task 10: git init と初期コミット

**Files:**
- Create: `mobile-app-poc-api-spec/.git/` (git init によって自動生成)

- [ ] **Step 1: git init**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git init
```

Expected: `Initialized empty Git repository in ...` の出力。

- [ ] **Step 2: ブランチ名を main に設定**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git symbolic-ref HEAD refs/heads/main
```

Expected: 出力なし（成功時は silent）。

- [ ] **Step 3: 全ファイルをステージ**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git add .
```

- [ ] **Step 4: ステージ内容を確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git status
```

Expected: 以下のファイルが新規追加としてリストアップされる:
- `.gitattributes`
- `.gitignore`
- `.spectral.yaml`
- `README.md`
- `examples/errors/internal-500.json`
- `examples/errors/not-found-404.json`
- `examples/errors/unauthorized-401.json`
- `examples/errors/validation-400.json`
- `examples/item-created.json`
- `examples/item-single.json`
- `examples/items-empty.json`
- `examples/items-list.json`
- `examples/login-success.json`
- `openapi.yaml`
- `package-lock.json`
- `package.json`
- `parameters/ItemIdPath.json`
- `responses/BadRequest.json`
- `responses/InternalServerError.json`
- `responses/NotFound.json`
- `responses/Unauthorized.json`
- `schemas/Error.json`
- `schemas/Item.json`
- `schemas/LoginRequest.json`
- `schemas/LoginResponse.json`
- `schemas/NewItem.json`
- `schemas/UpdateItem.json`

`node_modules/` は `.gitignore` により除外されていること。

- [ ] **Step 5: 初期コミット**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git commit -m "$(cat <<'EOF'
chore: initial scaffold of mobile-app-poc-api-spec

OpenAPI 3.0.3 contract repository with YAML skeleton + externalized
JSON schemas/responses/parameters/examples. Spectral lint passes.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: コミット成功、ハッシュとファイル数が表示される。

- [ ] **Step 6: 最終確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-api-spec" && git log --oneline && git status
```

Expected:
- `git log --oneline`: 1行の初期コミットが表示
- `git status`: `nothing to commit, working tree clean`

---

## 完了条件

- [ ] 上記Task 1〜10すべて完了
- [ ] `npm run lint` 実行時に違反ゼロ
- [ ] `git log --oneline` に1件のコミットが存在
- [ ] `git status` が clean
- [ ] 全26ファイル（`.git`/`node_modules`/`package-lock.json` 除く）が存在
- [ ] ユーザへの報告: 「scaffold完了、GitHubリモート作成とpushは手動で実施してください」

---

## 注意事項

1. **Git 初期化は Task 10 まで遅延する**。途中でコミットしない。
2. **全ファイルパスは絶対パス**で指定する（Windowsバス環境での作業ディレクトリ混乱を避けるため）。
3. **JSON ファイルは末尾改行 LF**（`.gitattributes` で強制されるが、書き出し時も配慮）。
4. **`openapi.yaml` の `$ref` は相対パス**（`./schemas/Item.json`）、`responses/*.json` 内は親ディレクトリからの相対パス（`../schemas/Error.json`）で記述する。
5. **Spectral lint でエラーが出た場合**、ファイル位置とパスの整合性を最優先で疑う。
