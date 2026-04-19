---
name: mobile-app-poc-api-spec 詳細設計
description: OpenAPI 3.0.3契約リポジトリの設計書。YAMLは骨格のみ保持し、スキーマ・example・再利用レスポンスをすべてJSONに外出しする方針
date: 2026-04-20
status: draft
parent: 2026-04-19-mobile-app-poc-master-design.md
---

# mobile-app-poc-api-spec 詳細設計

## 目的

モバイルアプリとバックエンドの間の **API 契約** を OpenAPI 3.0.3 形式の `openapi.yaml` を Single Source of Truth として管理する。mock / frontend / backend の3リポジトリがこの契約から型・Controller雛形を生成する。

## 設計原則（最重要）

**openapi.yaml は初回構築後ほぼ編集不要な骨格として維持し、日常的な変更はすべて外部JSONファイルに閉じ込める。**

YAMLの手編集は記号位置ずれ・インデント誤り・list/map混同などの事故が起きやすい。契約の**データ値**（スキーマ定義、example、エラー文言、enum、型）を1箇所でも変更するたびにYAMLを触る運用は不採用。

適用ルール:
- すべてのスキーマ定義は `schemas/*.json`、`$ref` で参照
- すべてのレスポンス例は `examples/*.json`、`externalValue` で参照
- 再利用する共通レスポンス（400/401/404/500）は `responses/*.json`、`$ref` で参照
- 共通パラメータは `parameters/*.json`、`$ref` で参照
- YAML内には具体データ値（文字列・数値・enum値）を記述しない

新規エンドポイント追加時のみYAML編集（`paths` 追記）が発生する。データ形状変更は完全にJSON内で完結する。

## 位置づけ

```
api-spec ──┬─> mock       （openapi.yaml を Prism で起動）
           ├─> frontend   （openapi.yaml → TypeScript型自動生成）
           └─> backend    （openapi.yaml → Controller雛形自動生成）
```

本リポジトリは4リポジトリの上流に位置し、破壊的変更は全体への影響が大きい。lint通過を必須運用とする。

## ディレクトリ構成

```
mobile-app-poc-api-spec/
├─ openapi.yaml                    # 骨格のみ（paths構造 + $ref/externalValue）
├─ schemas/                        # データ型定義
│  ├─ Item.json
│  ├─ NewItem.json
│  ├─ UpdateItem.json
│  ├─ LoginRequest.json
│  ├─ LoginResponse.json
│  └─ Error.json
├─ responses/                      # 再利用レスポンス定義
│  ├─ BadRequest.json              # 400
│  ├─ Unauthorized.json            # 401
│  ├─ NotFound.json                # 404
│  └─ InternalServerError.json     # 500
├─ parameters/                     # 共通パラメータ
│  └─ ItemIdPath.json              # /{id} のpath param
├─ examples/                       # レスポンス例
│  ├─ items-list.json              # GET /items 正常（複数件）
│  ├─ items-empty.json             # GET /items 正常（空）
│  ├─ item-single.json             # GET/PUT /items/{id} 正常
│  ├─ item-created.json            # POST /items 201
│  ├─ login-success.json           # POST /auth/login 200
│  └─ errors/
│     ├─ validation-400.json
│     ├─ unauthorized-401.json
│     ├─ not-found-404.json
│     └─ internal-500.json
├─ .spectral.yaml                  # lint設定
├─ package.json
├─ .gitignore
├─ .gitattributes                  # *.yaml *.json text eol=lf
└─ README.md
```

## エンドポイント一覧

| Method | Path | 認証 | 成功レスポンス | 主なエラー |
|---|---|---|---|---|
| POST | `/auth/login` | 不要 | 200 + LoginResponse | 401 |
| GET | `/items` | 必要 | 200 + Item[] | 401 |
| POST | `/items` | 必要 | 201 + Item | 400, 401 |
| GET | `/items/{id}` | 必要 | 200 + Item | 401, 404 |
| PUT | `/items/{id}` | 必要 | 200 + Item | 400, 401, 404 |
| DELETE | `/items/{id}` | 必要 | 204（No Content） | 401, 404 |

全エンドポイントで500発生の可能性あり（共通responseで参照）。

認証方式: `bearerAuth` セキュリティスキーム（`Authorization: Bearer <JWT>`）、`components.securitySchemes` に1つだけ定義し、認証が必要なエンドポイントで `security: [{ bearerAuth: [] }]` を指定。

## スキーマ詳細

### Item
必須フィールド: `id`, `name`, `quantity`, `createdAt`, `updatedAt`

| フィールド | 型 | 制約 |
|---|---|---|
| id | string | format: uuid、サーバー採番 |
| name | string | minLength: 1, maxLength: 100 |
| quantity | integer | minimum: 0 |
| createdAt | string | format: date-time、サーバー採番 |
| updatedAt | string | format: date-time、サーバー採番 |

### NewItem（POST入力）
必須: `name`, `quantity`（id/timestamps はサーバー生成）

### UpdateItem（PUT入力）
必須: `name`, `quantity`（PUTは全置換想定、PATCHは初期スコープ外）

### LoginRequest
必須: `username` (string, minLength:1), `password` (string, minLength:1)

### LoginResponse
必須: `token` (string, 発行済みJWT), `expiresIn` (integer, 秒数)

### Error（全エラーレスポンス共通）
必須: `code`, `message`, `timestamp`

| フィールド | 型 | 説明 |
|---|---|---|
| code | string | マシン可読コード（例: `ITEM_NOT_FOUND`, `UNAUTHORIZED`, `VALIDATION_ERROR`, `INTERNAL_ERROR`） |
| message | string | ユーザー向けメッセージ（日本語） |
| timestamp | string | format: date-time |

## Spectral lint

`.spectral.yaml`:
```yaml
extends:
  - "spectral:oas"
```

PoC初期は公式推奨ルールのみ。プロジェクト進行で違反が過剰または不足と判明したらルール追加検討。

`npm run lint` で `openapi.yaml` に対して実行、違反時は非ゼロ終了（CI想定）。

## package.json

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

- `private: true`（npmへの誤公開防止）
- スクリプトは最小限。検証が進んでから bundle / docs-preview を追加検討
- Redocly / Swagger UI は**初期スコープ外**

## サンプルデータ方針

- Item名: `サンプルアイテムA` / `サンプルアイテムB` / `サンプルアイテムC`
- quantity: `10`, `5`, `0`（境界含む）
- UUID: `11111111-1111-1111-1111-111111111111` 系の**例であることが一目でわかる固定値**
- タイムスタンプ: `2026-04-20T10:00:00Z` 系で固定
- username/password: `demo-user` / `demo-password`（認証例）

## .gitignore

```
node_modules/
captures/
*.log
.DS_Store
```

## .gitattributes

```
*.yaml text eol=lf
*.json text eol=lf
*.md   text eol=lf
```

Windows固有のCRLF混入防止。

## Node.js / 実行要件

- 必須: Node.js `>=18`
- 推奨: LTS 22.x（READMEに記載）
- `engines.node: ">=18"` を package.json に設定

## README構成

以下の章立てを最小構成として記載:

1. 概要（本リポジトリの役割、SSOTとしての位置づけ）
2. 前提（Node.js 18+、推奨22.x、Windows前提）
3. セットアップ（`npm install`）
4. lint実行（`npm run lint`、PowerShell前提）
5. ディレクトリ構成の説明
6. 他リポジトリとの関係（mock / frontend / backend への供給元）
7. 破壊的変更時の注意

## 対象外（初期スコープから除外）

- Redocly / Swagger UI によるドキュメントプレビュー
- OpenAPI bundle（`$ref` 解決後の単一ファイル生成）— mock側で必要になれば対応
- PATCH エンドポイント（PUTのみ）
- ページネーション、ソート、フィルタ
- リフレッシュトークン
- CIワークフロー（手動lint運用でPoCは十分）
- 非英語圏向けの多言語エラーメッセージ

## Git運用

- scaffold完了時点でリポジトリ内部で `git init` + 初期コミット実施（アシスタント）
- 初期コミットには全ファイル含める（`.gitignore` / `.gitattributes` も含む）
- GitHubリモート作成・push は**ユーザが個別実施**（アシスタントは関与しない）

## 実装後の検証

- [ ] `npm install` 成功
- [ ] `npm run lint` がエラーなしで通過
- [ ] `openapi.yaml` が有効なOpenAPI 3.0.3ドキュメントとして解釈できる（Spectralが構文違反を拾う）
- [ ] すべての `$ref` / `externalValue` が解決可能
- [ ] examples JSON が対応schemaにマッチする値を持つ

## 次のステップ

本書承認後、**writing-plans スキルで実装計画を作成**し、段階的に scaffold → 各JSON作成 → lint通過確認 → git init を行う。
