# mobile-app-poc-mock Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prismモックサーバーを起動するリポジトリ `mobile-app-poc-mock` をscaffoldし、`api-spec` を `./spec/` に取得して `npm run mock` でAPI応答する状態まで構築、初期コミット + GitHub push まで完了させる。

**Architecture:** mock リポジトリ自体は openapi.yaml を保持しない。`scripts/sync-spec.mjs` が `mobile-app-poc-api-spec` GitHub repo を `./spec/` に git clone/pull する。Prism CLI が `spec/openapi.yaml` を読んでポート4010でモックAPIを提供する。Prism操作ガイドを `docs/prism-guide.md` に併設。

**Tech Stack:** Node.js 18+, `@stoplight/prism-cli` 5.x, `git` CLI（`scripts/sync-spec.mjs` から呼び出し）, PowerShell/bash (Windows native)

---

## File Structure

Working directory: `C:\Oracle\3df002\mobile-app\mobile-app-poc-mock\`

**Created by this plan:**

```
mobile-app-poc-mock/
├─ package.json                   # npm + Prism依存 + apiSpecRepo
├─ .gitignore                     # node_modules/, spec/, captures/* (.gitkeep除く)
├─ .gitattributes                 # *.yaml *.json *.md *.mjs eol=lf
├─ .npmrc                         # engine-strict=true
├─ README.md                      # セットアップ・起動・トラブルシュート
├─ scripts/
│  └─ sync-spec.mjs               # api-spec取得スクリプト（標準モジュールのみ）
├─ docs/
│  └─ prism-guide.md              # Prism概要・操作ガイド
└─ captures/
   └─ .gitkeep                    # 空ディレクトリ保持用
```

**取得物（gitignored、本プランで生成だが版管理外）:**

```
mobile-app-poc-mock/
├─ spec/                          # npm run sync-spec の取得物
│  ├─ openapi.yaml
│  ├─ schemas/
│  ├─ responses/
│  ├─ parameters/
│  └─ examples/
└─ node_modules/                  # npm install の取得物
```

各ファイルの責務:
- `package.json`: 依存と起動スクリプトの宣言
- `scripts/sync-spec.mjs`: api-spec の clone/pull の唯一のロジック
- `README.md`: 利用者向け（操作手順・トラブルシュート）
- `docs/prism-guide.md`: Prism機能と操作の詳細ガイド
- gitignore/gitattributes/npmrc: 環境正規化

## Testing Approach

mockリポジトリは実行可能ソフトウェア。テスト = 「`npm run sync-spec` → `npm run mock` → `curl http://localhost:4010/items` で期待レスポンスが返る」。Task 7 で実施。

途中タスクでは `node --check scripts/sync-spec.mjs` で構文チェックのみ行う（Prismを起動するTaskは1つに集約）。

**Critical**: このプロジェクトに `git` リポジトリは Task 8 まで存在しない。途中で `git add`/`git commit` しないこと。すべてのファイルが揃ってから一括コミット。

---

## Task 1: ディレクトリ作成と package.json

**Files:**
- Create: `mobile-app-poc-mock/` (directory)
- Create: `mobile-app-poc-mock/package.json`

- [ ] **Step 1: ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock"
```

- [ ] **Step 2: package.json を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/package.json`

```json
{
  "name": "mobile-app-poc-mock",
  "version": "0.1.0",
  "private": true,
  "description": "Prism mock server for mobile-app-poc. Sources OpenAPI spec from mobile-app-poc-api-spec repository.",
  "apiSpecRepo": "https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git",
  "scripts": {
    "sync-spec": "node scripts/sync-spec.mjs",
    "mock": "prism mock spec/openapi.yaml --port 4010",
    "mock:lan": "prism mock spec/openapi.yaml --port 4010 --host 0.0.0.0"
  },
  "devDependencies": {
    "@stoplight/prism-cli": "^5.0.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

- [ ] **Step 3: 依存インストール**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && npm install
```

Expected: `added N packages` 出力、`node_modules/` と `package-lock.json` が生成される。

- [ ] **Step 4: Prism CLI が動くことを確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && npx prism --version
```

Expected: バージョン番号（例: `5.x.y`）が表示される。

---

## Task 2: gitignore / gitattributes / npmrc

**Files:**
- Create: `mobile-app-poc-mock/.gitignore`
- Create: `mobile-app-poc-mock/.gitattributes`
- Create: `mobile-app-poc-mock/.npmrc`

- [ ] **Step 1: .gitignore を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/.gitignore`

```
node_modules/
spec/
captures/*
!captures/.gitkeep
*.log
.DS_Store
```

`captures/*` で中身を全除外しつつ `.gitkeep` だけ例外で追跡対象にする。

- [ ] **Step 2: .gitattributes を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/.gitattributes`

```
*.yaml text eol=lf
*.yml  text eol=lf
*.json text eol=lf
*.md   text eol=lf
*.mjs  text eol=lf
```

- [ ] **Step 3: .npmrc を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/.npmrc`

```
engine-strict=true
```

---

## Task 3: scripts/sync-spec.mjs の作成

**Files:**
- Create: `mobile-app-poc-mock/scripts/sync-spec.mjs`

- [ ] **Step 1: scripts ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/scripts"
```

- [ ] **Step 2: sync-spec.mjs を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/scripts/sync-spec.mjs`

```javascript
import { spawnSync } from 'node:child_process';
import { existsSync, readFileSync } from 'node:fs';
import { resolve } from 'node:path';

const SPEC_DIR = resolve('spec');
const GIT_DIR = resolve(SPEC_DIR, '.git');
const DEFAULT_REPO = 'https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git';

function readPackageRepo() {
  try {
    const pkg = JSON.parse(readFileSync('package.json', 'utf8'));
    return pkg.apiSpecRepo ?? null;
  } catch {
    return null;
  }
}

const repoUrl = process.env.API_SPEC_REPO ?? readPackageRepo() ?? DEFAULT_REPO;

function runGit(args, cwd) {
  const where = cwd ? ` (in ${cwd})` : '';
  console.log(`> git ${args.join(' ')}${where}`);
  const result = spawnSync('git', args, { cwd, stdio: 'inherit' });
  if (result.error) {
    console.error(`Failed to spawn git: ${result.error.message}`);
    process.exit(1);
  }
  if (result.status !== 0) {
    console.error(`git ${args.join(' ')} exited with code ${result.status}`);
    process.exit(result.status ?? 1);
  }
}

if (existsSync(GIT_DIR)) {
  console.log(`spec/ exists. Updating from ${repoUrl}...`);
  runGit(['pull', '--ff-only'], SPEC_DIR);
} else {
  console.log(`Cloning ${repoUrl} into spec/...`);
  runGit(['clone', '--depth', '1', repoUrl, SPEC_DIR]);
}

console.log('spec sync complete.');
```

- [ ] **Step 3: 構文チェック**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && node --check scripts/sync-spec.mjs
```

Expected: 出力なし（エラーがあれば構文エラーが表示される）。

---

## Task 4: captures/.gitkeep

**Files:**
- Create: `mobile-app-poc-mock/captures/.gitkeep`

- [ ] **Step 1: captures ディレクトリと .gitkeep を作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/captures" && touch "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/captures/.gitkeep"
```

Expected: `captures/.gitkeep` が空ファイルとして存在。

---

## Task 5: README.md

**Files:**
- Create: `mobile-app-poc-mock/README.md`

- [ ] **Step 1: README.md を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/README.md`

````markdown
# mobile-app-poc-mock

`mobile-app-poc-api-spec` の `openapi.yaml` を入力に **Prism (Stoplight)** でモックAPIサーバーを立てるリポジトリ。
バックエンド未実装でもフロント開発・実機検証を進められる状態を提供する。

## 位置づけ

- mobile-app-poc-api-spec — このリポジトリの `./spec/` に取得元
- **mobile-app-poc-mock** ← 本リポジトリ
- mobile-app-poc-frontend — このモックを開発時API接続先に使用
- mobile-app-poc-bridge
- mobile-app-poc-backend — 将来本番APIに切替

## 前提

- Node.js `>=18` 必須（LTS 22.x 推奨）
- OS: Windows ネイティブ（PowerShell 前提）
- `git` CLI が PATH 上にあること（spec取得用）
- 社内イントラネット内での運用のみ

## セットアップ

### 共通

```powershell
npm install
```

### A: 自動同期（GitHub到達可な環境）

```powershell
npm run sync-spec
```

`./spec/` 配下に `mobile-app-poc-api-spec` の `main` を取得（初回clone、2回目以降pull）。

リポジトリURLを変更したい場合は環境変数で上書き:

```powershell
$env:API_SPEC_REPO="https://example.com/your-fork.git"; npm run sync-spec
```

### B: 手動コピー（GitHub到達不可な環境）

別手段で取得した `mobile-app-poc-api-spec` の中身を `./spec/` 配下に直接配置する。
`./spec/openapi.yaml`, `./spec/schemas/`, `./spec/responses/`, `./spec/parameters/`, `./spec/examples/` がそろっていれば動作する。

## 起動

### ローカル（自PC内のみ）

```powershell
npm run mock
```

→ `http://localhost:4010` で待ち受け。

### LAN共有（他PC・実機からアクセス）

```powershell
npm run mock:lan
```

→ `http://0.0.0.0:4010` で待ち受け。Windowsファイアウォールで4010ポートの受信許可が必要。

## 動作確認

```powershell
# 一覧取得（正常）
curl http://localhost:4010/items

# 一覧取得（空配列のexampleに切替）
curl -H "Prefer: example=empty" http://localhost:4010/items

# ログイン
curl -X POST http://localhost:4010/auth/login `
  -H "Content-Type: application/json" `
  -d '{"username":"demo-user","password":"demo-password"}'
```

## トラブルシュート

| 症状 | 対処 |
|---|---|
| `npm install` で実行ポリシーエラー | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| ポート4010が使用中 | `netsh interface ipv4 show excludedportrange` で予約範囲確認、または `--port 別番号` |
| LAN内他PCから繋がらない | `mock:lan` 使用？ファイアウォールで4010受信許可？ |
| 社内プロキシで `npm install` 失敗 | `npm config set proxy http://...`, `npm config set https-proxy http://...` |
| `npm run sync-spec` 失敗 | git CLI が PATH にあるか、リポジトリURLにアクセス可能か |
| openapi.yaml 更新後にレスポンスが変わらない | Prism は起動時に読み込む。`Ctrl+C` で停止して再起動 |

## ディレクトリ構成

```
mobile-app-poc-mock/
├─ scripts/sync-spec.mjs   # api-spec取得スクリプト
├─ docs/prism-guide.md     # Prism概要・操作詳細
├─ captures/               # 取得データ用（gitignored）
├─ spec/                   # api-spec取得物（gitignored）
└─ package.json
```

## Prism詳細操作

`docs/prism-guide.md` を参照（example切替、ステータスコード強制、リクエスト検証、ログ確認等）。

## サービス化（参考）

常時稼働が必要な場合 `nssm` で Windows サービス化可能。本PoCの初期スコープ外。
````

---

## Task 6: docs/prism-guide.md

**Files:**
- Create: `mobile-app-poc-mock/docs/prism-guide.md`

- [ ] **Step 1: docs ディレクトリを作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/docs"
```

- [ ] **Step 2: prism-guide.md を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-mock/docs/prism-guide.md`

````markdown
# Prism 操作ガイド

本ドキュメントは Prism (Stoplight) の概要と、本リポジトリでよく使う操作をまとめたもの。
Prism公式のすべてを網羅するものではない。詳細は[公式ドキュメント](https://docs.stoplight.io/docs/prism)を参照。

## Prism とは

Stoplight が提供する OSS の HTTP モックサーバー。OpenAPI 3.x 契約から **コード生成なし** でモックAPIを起動できる。

主な特徴:
- 静的モック: openapi.yaml の `examples` をそのまま返す
- 動的モック: schema から faker で擬似データ生成
- リクエスト検証: 受信リクエストを openapi.yaml に照合、違反時は 422 を返す
- 複数example切替: HTTP ヘッダ `Prefer` で example 名を指定して切替可能
- ステータスコード強制: `Prefer: code=...` で任意の定義済みコードを返却
- proxy モード: 実APIへの中継 + 検証も可能

## 本リポジトリでの使い方

**静的モックに特化**。`examples/*.json` をそのまま返す前提で構築されている。動的モック（faker）は初期スコープ外。

起動コマンド:
- `npm run mock` — `http://localhost:4010`、自PC内のみ
- `npm run mock:lan` — `http://0.0.0.0:4010`、LAN内他PCからアクセス可

## 基本操作

### サーバー起動・停止

```powershell
npm run mock
```

起動ログでエンドポイント一覧が表示される:
```
[CLI] ✔  success   Prism is listening on http://127.0.0.1:4010
[CLI] ●  note      GET    http://127.0.0.1:4010/items
[CLI] ●  note      POST   http://127.0.0.1:4010/items
...
```

停止: `Ctrl+C`

### デフォルトレスポンス

各エンドポイントに対して、Prism は以下の優先順位でレスポンスを選ぶ:
1. **2xx 範囲** で定義された最初の response
2. その response の **最初に定義された example**

例: `GET /items` には `200` と `examples.list` / `examples.empty` がある。デフォルトでは `list` (最初に定義) が返る。

### リクエスト送信例

```powershell
# GET（Bearer認証ヘッダなしでも返る — Prismは認証検証しない）
curl http://localhost:4010/items

# POST（リクエストボディは openapi の NewItem schema に検証される）
curl -X POST http://localhost:4010/items `
  -H "Content-Type: application/json" `
  -d '{"name":"newItem","quantity":5}'

# 不正なボディを送ると 422 が返る
curl -X POST http://localhost:4010/items `
  -H "Content-Type: application/json" `
  -d '{"name":""}'
```

## example 切替

`Prefer: example=<key>` ヘッダで複数example間を切り替え。

```powershell
# 一覧 → 空配列
curl -H "Prefer: example=empty" http://localhost:4010/items
```

本リポジトリで定義済みの example キー（`spec/openapi.yaml` 参照）:

| エンドポイント | キー | 期待レスポンス |
|---|---|---|
| GET /items | `list` | サンプルアイテムA/B/C |
| GET /items | `empty` | `[]` |
| GET /items/{id} | `single` | サンプルアイテムA |
| POST /items | `created` | 新規アイテム |
| PUT /items/{id} | `updated` | item-single |
| POST /auth/login | `success` | demo-user用JWT |

エラー response 内の examples は通常 `Prefer: code=...` 経由で取り出す（次節）。

## ステータスコード強制

`Prefer: code=<status>` で任意の定義済みステータスコードを返却。

```powershell
# 401 を強制
curl -H "Prefer: code=401" http://localhost:4010/items

# 404 を強制
curl -H "Prefer: code=404" http://localhost:4010/items/00000000-0000-0000-0000-000000000000

# 500 を強制
curl -H "Prefer: code=500" http://localhost:4010/items
```

`Prefer: code=401, example=unauthorized` のように組み合わせ可能。

## リクエスト検証

Prism はデフォルトで以下を openapi.yaml に照合:
- パスパラメータの型・format（UUID format 違反は 422）
- クエリパラメータの required / 型
- ヘッダの required
- リクエストボディの schema

違反時は `422 Unprocessable Entity` で詳細メッセージを返す。
バリデーションをゆるめたい場合は `--errors` オプションを参照（本PoCでは未使用）。

## 動的モック（参考・初期スコープ外）

`prism mock --dynamic spec/openapi.yaml` で起動すると、example が定義されていなくても schema から faker でデータ生成。

利点: example を毎回書く必要がない、ランダムテストデータを得られる
注意: テスト結果が再現しない、example で意図したコーナーケースが出ない

本リポジトリでは静的モックを基本とし、動的モックは別フェーズで検討。

## proxy モード（参考）

`prism proxy spec/openapi.yaml http://real-backend:8080` で実APIへ中継。実レスポンスも openapi に検証可能。
本PoCでは未使用。

## CORS

Prism はデフォルトで `Access-Control-Allow-Origin: *` 等のゆるいCORSヘッダを返す。
Capacitor / Web フロントから直接呼び出して CORS で詰まることはほぼない（Capacitor 側で `CapacitorHttp` を使えば CORS 自体が無関係）。

## ログとデバッグ

通常起動でリクエスト/レスポンスログが標準出力に流れる。

詳細ログが欲しい場合（環境変数）:
```powershell
$env:DEBUG="prism:*"; npm run mock
```

## よくある操作シナリオ

| 困りごと | 確認ポイント |
|---|---|
| frontendから繋がらない | ポート、ファイアウォール、`mock:lan`使用、`VITE_API_BASE` の値 |
| 想定と違うレスポンス | `Prefer: example=...` で意図したexampleを指定したか、`spec/openapi.yaml` の example 定義 |
| openapi.yaml更新後に変化なし | Prismは起動時に読み込む。`Ctrl+C` → `npm run mock` で再起動 |
| 認証ありエンドポイントが認証なしで200を返す | Prism は JWT 検証しない仕様。認証ロジックの検証は backend 側で行う |
| 422 が返ってリクエストが通らない | リクエスト形状を openapi.yaml の `requestBody.schema` と照合 |

## 公式ドキュメント

- リポジトリ: https://github.com/stoplightio/prism
- ドキュメント: https://docs.stoplight.io/docs/prism
- HTTP request: https://docs.stoplight.io/docs/prism/83dbbd75532cf-http-mocking
````

---

## Task 7: 動作確認（sync-spec → mock 起動 → curl）

このタスクは新規ファイルを作らず、これまでの成果物が機能することを実機確認する。

- [ ] **Step 1: spec を取得**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && npm run sync-spec
```

Expected:
- 出力に `Cloning https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git into spec/...` 表示
- `spec/openapi.yaml`, `spec/schemas/`, `spec/responses/`, `spec/parameters/`, `spec/examples/` が出現

- [ ] **Step 2: spec が揃ったか確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && ls spec/openapi.yaml spec/schemas spec/responses spec/parameters spec/examples
```

Expected: 5項目すべて存在。

- [ ] **Step 3: Prism を バックグラウンドで起動**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && npm run mock
```

このコマンドは `run_in_background: true` で起動し、後段で curl 検証する。
Expected ログ（起動から数秒以内）: `Prism is listening on http://127.0.0.1:4010`

- [ ] **Step 4: curl で正常レスポンス確認**

```bash
curl -s http://localhost:4010/items
```

Expected: `examples/items-list.json` 相当の配列（サンプルアイテムA/B/C を含む）。

- [ ] **Step 5: curl で example 切替確認**

```bash
curl -s -H "Prefer: example=empty" http://localhost:4010/items
```

Expected: `[]` （空配列）

- [ ] **Step 6: curl でステータスコード強制確認**

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Prefer: code=401" http://localhost:4010/items
```

Expected: `401`

- [ ] **Step 7: バックグラウンド Prism を停止**

起動した npm run mock のバックグラウンドジョブを終了する（実行手段はexecution環境による）。

```bash
# Bash の場合
kill %1 2>/dev/null || true
```

- [ ] **Step 8: 動作確認結果をまとめる**

確認できた項目:
- [x] sync-spec で spec/ 取得
- [x] mock 起動でポート4010待ち受け
- [x] GET /items 正常レスポンス
- [x] Prefer: example=empty で空配列に切替
- [x] Prefer: code=401 で401レスポンス強制

---

## Task 8: git init + ローカル user 設定 + 初期コミット

**Files:**
- Create: `mobile-app-poc-mock/.git/` (git init による自動生成)

- [ ] **Step 1: git init**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git init
```

- [ ] **Step 2: ブランチ名を main に設定**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git symbolic-ref HEAD refs/heads/main
```

- [ ] **Step 3: ローカル git user 設定（このリポジトリのみ）**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git config user.name "m-miyawaki" && git config user.email "miyawakifamilymyam@gmail.com"
```

- [ ] **Step 4: ステージ前確認（spec/ と node_modules/ が除外されていること）**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git add . && git status --short
```

Expected: 以下のみ追加対象として表示（`spec/*`, `node_modules/*`, `captures/*` (.gitkeep除く) は出てこない）:
- `.gitattributes`
- `.gitignore`
- `.npmrc`
- `README.md`
- `captures/.gitkeep`
- `docs/prism-guide.md`
- `package-lock.json`
- `package.json`
- `scripts/sync-spec.mjs`

- [ ] **Step 5: 初期コミット**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git commit -m "$(cat <<'EOF'
chore: initial scaffold of mobile-app-poc-mock

Prism mock server with sync-spec script that pulls openapi.yaml
from mobile-app-poc-api-spec. Includes Prism operation guide.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: コミット成功、9ファイル追加。

- [ ] **Step 6: 最終確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git log --oneline && git status
```

Expected:
- `git log --oneline`: 1行の初期コミットが表示
- `git status`: `nothing to commit, working tree clean`

---

## Task 9: GitHub repo 作成 + push（ユーザ承認後）

**重要**: このタスクは **ユーザの明示確認後** に実行する。コントローラがユーザに「公開リポジトリ作成 + push して良いか」を確認し、承諾を得てから着手する。

- [ ] **Step 1: ユーザ確認（コントローラが実施）**

ユーザに以下を確認:
- リポジトリ名: `mobile-app-poc-mock`
- 公開設定: api-spec と同じ public で良いか
- アカウント: `m-miyawaki-m`

- [ ] **Step 2: gh で公開リポジトリ作成 + push**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && gh repo create m-miyawaki-m/mobile-app-poc-mock --public --source=. --remote=origin --push
```

Expected: リポジトリURL表示、`branch 'main' set up to track 'origin/main'`、`* [new branch]      HEAD -> main`。

- [ ] **Step 3: 確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && git remote -v && git log --oneline
```

Expected:
- `origin   https://github.com/m-miyawaki-m/mobile-app-poc-mock.git (fetch/push)`
- 初期コミットが表示される

---

## 完了条件

- [ ] 上記Task 1〜9すべて完了
- [ ] `npm run sync-spec` で `spec/` が取得できる
- [ ] `npm run mock` で Prism が起動し、`/items` 等が応答する
- [ ] `Prefer` ヘッダによる example 切替・コード強制が動作する
- [ ] `git log --oneline` に初期コミット存在
- [ ] GitHub に `m-miyawaki-m/mobile-app-poc-mock` が public で存在し、push済み
- [ ] ユーザへの報告: 「scaffold完了、リポジトリURL: https://github.com/m-miyawaki-m/mobile-app-poc-mock」

---

## 注意事項

1. **Git 初期化は Task 8 まで遅延する**。途中でコミットしない。
2. **`spec/` を絶対にコミットしない**。`.gitignore` で除外、Step 4 の git status で混入チェック。
3. **node_modules も同様**。
4. **絶対パスで操作**（Windowsバス環境での作業ディレクトリ混乱を避けるため）。
5. **Prism バックグラウンド起動の停止忘れ**に注意（Task 7 Step 7）。
6. **GitHub repo 作成は破壊性のある外部操作**。Task 9 はユーザ確認必須。
