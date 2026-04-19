---
name: mobile-app-poc-mock 詳細設計
description: api-spec から openapi.yaml を取得して Prism モックサーバーを起動するリポジトリ。Prism操作ガイドも併設
date: 2026-04-20
status: draft
parent: 2026-04-19-mobile-app-poc-master-design.md
---

# mobile-app-poc-mock 詳細設計

## 目的

`mobile-app-poc-api-spec` の `openapi.yaml` を入力として **Prism (Stoplight)** によるモック API サーバーを立て、frontend / bridge / backend が完成していなくても Web/モバイル開発を進められる状態を提供する。

## 設計原則

- **api-spec を SSOT として尊重する**: mock リポジトリは契約の改変を行わない。`./spec/` 配下は取得物としてgitignoreし、版管理しない
- **追加依存を最小化**: Prism CLI 1つだけ、faker等は初期スコープ外
- **Windows ネイティブで完結**: WSL2 / Docker / グローバル install 不使用
- **公開範囲は社内LAN内まで**: 外部公開（Funnel/ngrok/CDN）は対象外
- **ローカル単独起動と LAN共有起動の両方に対応**

## 位置づけ

```
mobile-app-poc-api-spec ──> mobile-app-poc-mock ──> frontend（API接続先）
                                                  └─> （その他検証用クライアント）
```

mock は api-spec の**消費者**であり、frontend / bridge の**供給者**。

## ディレクトリ構成

```
mobile-app-poc-mock/
├─ spec/                          # api-spec から取得（gitignored）
│  ├─ openapi.yaml
│  ├─ schemas/
│  ├─ responses/
│  ├─ parameters/
│  └─ examples/
├─ scripts/
│  └─ sync-spec.mjs               # api-spec を取得（git clone or pull）
├─ docs/
│  └─ prism-guide.md              # Prism概要・操作ガイド
├─ captures/                      # 取得データ用（gitignored、空ディレクトリ）
├─ package.json
├─ .gitignore
├─ .gitattributes
├─ .npmrc
└─ README.md
```

## spec/ の取得方針

mock 単体では `openapi.yaml` を保持しない。`./spec/` に api-spec を取得して使う。取得方法は2系統:

### 方法A: 自動同期（デフォルト・推奨）

`npm run sync-spec` で `https://github.com/m-miyawaki-m/mobile-app-poc-api-spec` の `main` ブランチを `./spec/` に取得する。

- 初回: `git clone --depth 1 <repo> spec`
- 2回目以降: `git -C spec pull --ff-only`
- 失敗時は exit code 非ゼロで終了

`spec/` 全体は `.gitignore` で除外。mock リポジトリ側は api-spec のスナップショットを版管理しない。

### 方法B: 手動コピー（GitHub到達不可な環境用）

GitHubに到達できない環境では、別手段で取得した `openapi.yaml` + `schemas/` + `responses/` + `parameters/` + `examples/` を `./spec/` 配下に直接配置する。npm script は実行しない。

mock リポジトリ側は「`spec/openapi.yaml` が存在すれば動く」状態を保つ。中身がgit cloneで来ようが手動コピーで来ようが関係ない。

## npm scripts

```json
{
  "scripts": {
    "sync-spec": "node scripts/sync-spec.mjs",
    "mock": "prism mock spec/openapi.yaml --port 4010",
    "mock:lan": "prism mock spec/openapi.yaml --port 4010 --host 0.0.0.0"
  }
}
```

| スクリプト | 用途 | バインド |
|---|---|---|
| `sync-spec` | api-spec取得・更新 | — |
| `mock` | ローカル開発（自PC内のみ） | `127.0.0.1:4010` |
| `mock:lan` | LAN共有（他PC/実機からアクセス） | `0.0.0.0:4010` |

`mock:dynamic`（faker動的モック）は **初期スコープ外**。PoCのつなぎ目検証は静的examplesで十分。後続フェーズで必要になれば追加検討。

## sync-spec.mjs

Node.js 標準モジュールのみで実装、追加依存なし:

- `process.env.API_SPEC_REPO` または `package.json` の `apiSpecRepo` フィールドからリポジトリURL取得
- 既定: `https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git`
- `spec/.git` の存在チェック → 無ければclone、あればpull
- `child_process.spawnSync('git', ...)` でgit呼び出し
- 各操作の終了コードを確認、失敗時は標準エラーにメッセージ出して `process.exit(1)`

## 依存

```json
{
  "devDependencies": {
    "@stoplight/prism-cli": "^5.0.0",
    "@redocly/cli": "^2.0.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

追加依存（faker, dotenv, etc.）はなし。`@redocly/cli` は次の「externalValue 制約と bundle 対処」で必要となるため追加。

## externalValue 制約と bundle 対処（実装中の発見）

**問題**: Prism CLI 5.x は OpenAPI の `externalValue`（外部JSONファイル参照）を解決しない。api-spec は examples を `examples/*.json` に分離管理する設計のため、Prism にそのまま渡すと examples が無視され、schema から自動生成された汎用データが返ってしまう。

**対処**: `@redocly/cli` の `bundle` コマンドで全 `$ref` と `externalValue` をインライン解決した単一ファイル `spec/openapi.bundled.yaml` を生成し、Prism にはこの bundled YAML を渡す。

**npm scripts に bundle ステップを追加**:

```json
{
  "scripts": {
    "sync-spec": "node scripts/sync-spec.mjs",
    "postsync-spec": "npm run bundle-spec",
    "bundle-spec": "redocly bundle spec/openapi.yaml -o spec/openapi.bundled.yaml",
    "mock": "prism mock spec/openapi.bundled.yaml --port 4010",
    "mock:lan": "prism mock spec/openapi.bundled.yaml --port 4010 --host 0.0.0.0"
  }
}
```

- `sync-spec` 実行時に `postsync-spec` フックで bundle が自動実行
- 手動コピー（GitHub到達不可な環境）の場合は `npm run bundle-spec` を別途実行
- bundled YAML は `spec/` 配下にあるため `.gitignore` 対象（既存）。再生成可能なビルド成果物として扱う

**含意**: 他の OpenAPI ツール（型生成・codegen 等）も `externalValue` の扱いはバラバラなため、frontend/backend で型生成する際も bundle 後のファイルを入力にする方針が安全。api-spec 側の設計（JSON分離）は変えず、消費者側で bundle するのが正解。

## .gitignore

```
node_modules/
spec/
captures/
*.log
.DS_Store
```

`spec/` を除外する点に注意（api-spec が SSOT）。

## .gitattributes

```
*.yaml text eol=lf
*.yml  text eol=lf
*.json text eol=lf
*.md   text eol=lf
*.mjs  text eol=lf
```

## .npmrc

```
engine-strict=true
```

`engines.node` を強制（Node.js 18 未満で `npm install` 拒否）。

## captures/

PoC初期は空ディレクトリ。`.gitkeep` を1つ置く。Prismの `--captures` オプションで使う想定（future use）。

## README構成

最小限の起動手順とトラブルシュート:

1. 概要（Prism モック、SSOTはapi-spec別リポ）
2. 前提（Node.js 18+ / LTS22.x推奨、Windowsネイティブ、社内LAN）
3. セットアップ
   - `npm install`
   - 方法A: `npm run sync-spec`（GitHub到達可）
   - 方法B: 手動で `./spec/` に api-spec の中身を配置（GitHub到達不可）
4. 起動
   - ローカル: `npm run mock`
   - LAN共有: `npm run mock:lan`（ファイアウォール許可必要）
5. 動作確認: `curl http://localhost:4010/items` 等
6. トラブルシュート
   - PowerShell実行ポリシー: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`
   - ポート競合: `netsh interface ipv4 show excludedportrange`
   - ファイアウォール許可（LAN共有時のみ）
   - 社内プロキシ: `npm config set proxy ...`
7. 詳細操作ガイド: `docs/prism-guide.md` への参照
8. 他リポジトリとの関係（api-specの消費者、frontend/bridgeの供給者）

## Prism操作ガイド（docs/prism-guide.md）

mockリポジトリ内に Prism の概要と操作詳細をまとめた専用ドキュメントを配置する。新人開発者が「mockサーバって何ができるの？」と聞かれて即答できる粒度。以下を含める:

### 章立て

1. **Prismとは何か** — Stoplight 提供のOSS、OpenAPI契約からモックサーバーを生成するCLI
2. **本リポジトリでの使い方** — 静的モック（examples返却）に特化していること
3. **基本操作**
   - サーバー起動・停止
   - デフォルトレスポンスの仕組み（最初に定義されたexample or 200番台）
   - リクエスト送信（curl, ブラウザ, Postman）
4. **example切替**
   - `Prefer: example=<key>` ヘッダで複数example切替
   - 例: `curl -H "Prefer: example=empty" http://localhost:4010/items`
   - 本リポジトリで定義済みexampleキー一覧
5. **ステータスコード強制**
   - `Prefer: code=404` で任意の定義済みステータスを返却
   - 例: 認証失敗を試したい場合 `-H "Prefer: code=401"`
6. **リクエストバリデーション**
   - Prismはデフォルトでリクエストボディ・パラメータを openapi.yaml に照合
   - 違反時は `422 Unprocessable Entity` で詳細を返す
   - 緩和したい場合: `--errors` オフ等のオプション
7. **動的モック（参考・初期スコープ外）**
   - `--dynamic` フラグで faker による自動データ生成
   - 利点と注意点
8. **プロキシモード（参考）**
   - `prism proxy` で実APIへの中継 + 検証
   - 本PoCでは未使用
9. **CORS** — Prismは標準でゆるいCORS設定を返す（Capacitor/Webから直接叩ける）
10. **ログとデバッグ**
    - 起動ログの読み方
    - リクエスト/レスポンスログ
    - `DEBUG=prism:*` 環境変数で詳細ログ
11. **本リポジトリでのよくある操作シナリオ**
    - 「frontendから繋がらない」→ ポート、ファイアウォール、`mock:lan`使用確認
    - 「想定と違うレスポンス」→ `Prefer: example=...` 確認、example一覧確認
    - 「openapi.yaml更新後に変化なし」→ Prismは起動時に読み込むため再起動必要
    - 「authが必要なエンドポイントが認証なしで200を返す」→ Prismは認証検証しない仕様
12. **公式ドキュメントへのリンク**
    - https://github.com/stoplightio/prism
    - https://docs.stoplight.io/docs/prism

### 想定読者

- 本リポジトリを初めて触る開発者（Prism未経験）
- frontend担当者で「とりあえずモックを叩く」レベルの理解で十分な人
- バックエンド担当者で「Prismは認証検証しない」「動的モックは別」を理解しておきたい人

### 制約

- Prismの**全機能を網羅しない**。本PoCで実際に使う/参照する操作に絞る
- バージョンアップで挙動が変わる箇所は公式ドキュメントへの参照に留める

## 起動確認（実装後の検証）

- `npm install` 成功
- `npm run sync-spec` で `spec/openapi.yaml` 出現
- `npm run mock` で `http://localhost:4010` 起動、ログにエンドポイント一覧表示
- `curl http://localhost:4010/items` → `examples/items-list.json` の内容が返る
- `curl -H "Prefer: example=empty" http://localhost:4010/items` → 空配列 `[]` が返る
- `curl -X POST http://localhost:4010/auth/login -H "Content-Type: application/json" -d '{"username":"demo-user","password":"demo-password"}'` → `examples/login-success.json` 相当が返る

## 対象外（初期スコープから除外）

- 動的モック（faker、`mock:dynamic`）
- nssm / Windowsサービス化（READMEに参考リンクのみ）
- 認証検証（Prismは標準でJWT検証しない仕様、許容）
- HTTPS化
- `prism proxy` モード
- `prism captures` 使用（ディレクトリだけ用意、機能利用は別フェーズ）
- 外部公開（Tailscale Funnel / ngrok / クラウドデプロイ）
- CIワークフロー
- 多言語ドキュメント

## Git運用

- scaffold完了時にリポジトリ内部で `git init` + 初期コミット（アシスタント実施）
- ローカル `git config user.name` / `user.email` 設定（本リポジトリのみ）
- GitHub リモート作成・push は api-spec と同じ手順（`gh repo create ... --public --source=. --remote=origin --push`）
- `spec/` `node_modules/` `captures/*`（`.gitkeep`除く）はgitignoreで除外

## 次のステップ

本書承認後、**writing-plans スキルで実装計画作成**。段階は以下を想定:

1. プロジェクト初期化（package.json, .gitignore, .gitattributes, .npmrc）
2. sync-spec.mjs スクリプト作成
3. README.md
4. docs/prism-guide.md
5. captures/.gitkeep
6. `npm install` → `npm run sync-spec` → `npm run mock` で動作確認
7. git init + 初期コミット
8. GitHubリポジトリ作成 + push（ユーザ承認後）
