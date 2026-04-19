---
name: mobile-app-poc-frontend 詳細設計
description: Ionic + Vue + Capacitor によるモバイルフロントエンド。mockサーバーに対してログイン + アイテムCRUDを実装、Android実機検証は後追い
date: 2026-04-20
status: draft
parent: 2026-04-19-mobile-app-poc-master-design.md
---

# mobile-app-poc-frontend 詳細設計

## 目的

モバイルアプリのフロントエンド。Ionic + Vue + Capacitor で構築し、`mobile-app-poc-mock` の API に対して **ログイン + アイテムCRUD** を実装する。Android実機動作は後追い。bridge連携は別セッション完成後に統合する。

## 設計原則

- **mock を SSOT 上流として尊重**: api-spec → mock → frontend のチェーン
- **型は openapi.yaml から自動生成**（手書きしない、ドリフト防止）
- **bundle 経由で型生成**: Prism と同様に `externalValue` を含む形式は redocly bundle 後の YAML を入力にする
- **API 接続先は環境変数で切替**: `VITE_API_BASE`（mock / backend / 本番）
- **CapacitorHttp 採用**: CORS 影響を受けず Web/Android 両対応
- **YAGNI**: 初期は最小2画面（Login + Items）、PUT個別更新・bridge連携は後追い

## 位置づけ

```
api-spec ─> mock ─> frontend ←─ (将来) backend
                ↑
                bridge（別セッション完成後に統合）
```

frontend は mock の消費者であり、最終的に bridge を組み込んでネイティブ機能を呼び出す。

## ディレクトリ構成

```
mobile-app-poc-frontend/
├─ spec/                                # api-spec取得物 + bundled.yaml（gitignored）
│  ├─ openapi.yaml
│  └─ openapi.bundled.yaml              # redocly bundle 出力
├─ scripts/
│  └─ sync-spec.mjs                     # mockリポと同等
├─ src/
│  ├─ api/
│  │  ├─ generated/openapi.d.ts         # openapi-typescript出力（gitignored）
│  │  └─ client.ts                      # CapacitorHttp ラッパー
│  ├─ stores/
│  │  ├─ auth.ts                        # JWT管理
│  │  └─ items.ts                       # アイテム一覧管理
│  ├─ views/
│  │  ├─ LoginPage.vue
│  │  └─ ItemsPage.vue
│  ├─ router/index.ts
│  ├─ App.vue
│  ├─ main.ts
│  └─ vite-env.d.ts
├─ public/
├─ .env.development                     # VITE_API_BASE=http://localhost:4010 等
├─ .env.example                         # 環境変数テンプレート
├─ capacitor.config.ts
├─ ionic.config.json
├─ vite.config.ts
├─ tsconfig.json
├─ index.html
├─ package.json
├─ .gitignore
├─ .gitattributes
├─ .npmrc
└─ README.md
```

## ベースの作り方

`ionic start` 標準テンプレート（`blank --type=vue --capacitor`）から派生。Ionic CLI が生成する scaffold をベースに、不要部分削除 + 本設計の各要素を追加する。

`ionic start` 自動生成物のうち維持するもの:
- `package.json`、`vite.config.ts`、`tsconfig.json`、`ionic.config.json`、`capacitor.config.ts`
- `index.html`、`public/`、`src/main.ts`、`src/App.vue` の構造（中身は書き換え）

削除・改変するもの:
- 既定の `Tabs/Tab1/Tab2/Tab3` view → 削除して LoginPage / ItemsPage に置き換え
- `theme/` 配下の余計なCSSは PoC スコープ外として最小化

## 技術スタック

| 領域 | 採用 |
|---|---|
| フレームワーク | Ionic 8.x (Vue) |
| UI | Vue 3 (Composition API, `<script setup>`) |
| 言語 | TypeScript |
| ビルド | Vite |
| ネイティブ | Capacitor 6.x |
| 状態管理 | Pinia |
| ルーティング | @ionic/vue-router |
| HTTP | CapacitorHttp（@capacitor/core） |
| 永続化 | @capacitor/preferences |
| 型生成 | openapi-typescript |
| spec bundling | @redocly/cli |

## API クライアント方針

`src/api/client.ts` に薄いラッパー1本:

```ts
async function request<T>(method, path, body?): Promise<T>
```

責務:
- baseURL = `import.meta.env.VITE_API_BASE`
- `Authorization: Bearer <token>` を auth store から取得して自動付与
- `Content-Type: application/json` 既定
- `CapacitorHttp.request({...})` 呼び出し
- HTTP ステータスが 4xx/5xx なら throw（呼び元で catch）
- レスポンスボディを `T` として返却

型は `openapi-typescript` 生成の型を直接参照:

```ts
import type { paths } from './generated/openapi';

type ItemsListResponse =
  paths['/items']['get']['responses']['200']['content']['application/json'];
```

呼び出し例:

```ts
const items = await request<ItemsListResponse>('GET', '/items');
```

ヘルパは作らない（YAGNI）。型サポートはあるので生 `request` で十分。

## openapi 取り込みフロー

mock リポと同じ `sync-spec.mjs` パターンを踏襲し、加えて型生成も自動チェイン:

```
npm run sync-spec
  → spec/ にapi-spec git clone (or pull)
  → postsync-spec フック発火
    → npm run bundle-spec
      → redocly bundle spec/openapi.yaml -o spec/openapi.bundled.yaml
      → postbundle-spec フック発火
        → npm run gen-types
          → openapi-typescript spec/openapi.bundled.yaml -o src/api/generated/openapi.d.ts
```

1コマンド (`npm run sync-spec`) で「取得 → bundle → 型生成」が完了する。手動コピー時は `npm run bundle-spec` を直接実行（postbundle-spec で型生成も連動）。

`spec/` と `src/api/generated/` は両方とも `.gitignore` 対象（再生成可能なビルド成果物）。

## 状態管理

### `stores/auth.ts`

```
state:  token: string | null, expiresAt: number | null
getters: isAuthenticated
actions:
  - login(username, password)        → POST /auth/login → 保存
  - logout()                          → 状態クリア + Preferences削除
  - loadFromStorage()                 → 起動時にPreferencesから復元
```

永続化: `@capacitor/preferences`（Web=localStorage、Android=SharedPreferences）

`expiresIn` は秒数で返るため `Date.now() + expiresIn*1000` を `expiresAt` に保存。
有効期限切れチェックは getter `isAuthenticated` 内で実施。

### `stores/items.ts`

```
state:   items: Item[], loading: boolean, error: string | null
actions:
  - fetchAll()       → GET /items → state.items更新
  - create(input)    → POST /items → fetchAll()
  - remove(id)       → DELETE /items/{id} → fetchAll()
```

`Item` 型は openapi-typescript 生成からインポート。

PUT個別更新は初期スコープ外。

## ルーティング

| Path | View | 認証ガード |
|---|---|---|
| `/login` | LoginPage | 不要 |
| `/items` | ItemsPage | 必要（未認証は `/login` リダイレクト） |
| `/` | リダイレクト | 認証済 → `/items`、未認証 → `/login` |

`router.beforeEach` で auth store の `isAuthenticated` 判定。

## UI 概要

### LoginPage

- `IonHeader` + `IonTitle` 「ログイン」
- `IonContent`:
  - `IonInput` × 2（username / password、`.env.development` の値で初期化）
  - `IonButton` 「ログイン」
- 失敗時 `IonToast` でエラーメッセージ
- 成功時 `router.replace('/items')`

### ItemsPage

- `IonHeader` + `IonTitle` 「アイテム」 + 右上に「ログアウト」ボタン
- `IonContent`:
  - `IonList` で `items` を表示
    - 各 `IonItem`: `name` + `quantity`
    - `IonItemSliding` で右スワイプ削除
  - 取得失敗時 `IonToast`
  - 空配列時はメッセージ表示
- `IonFab`（右下「+」）→ `IonModal` で新規作成フォーム
  - フォーム: `IonInput`（name） + `IonInput type="number"`（quantity）
  - 「保存」「キャンセル」ボタン

## .env

### `.env.development`（git追跡対象）

```
VITE_API_BASE=http://localhost:4010
VITE_LOGIN_USERNAME=demo-user
VITE_LOGIN_PASSWORD=demo-password
```

### `.env.example`（git追跡対象、テンプレート）

```
# API接続先 (mock / backend / 本番)
VITE_API_BASE=http://localhost:4010

# ログインフォーム初期値（PoC便宜用、本番では削除）
VITE_LOGIN_USERNAME=demo-user
VITE_LOGIN_PASSWORD=demo-password
```

### `.env.device`（git追跡対象外）

実機検証時に作成。`VITE_API_BASE=http://<PC LAN IP>:4010` 等。READMEに作成手順を記載するのみ。

### `.env.production`（git追跡対象外）

backend完成後に作成。同上。

## package.json scripts

```json
{
  "scripts": {
    "dev": "vite",
    "serve": "ionic serve",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "sync-spec": "node scripts/sync-spec.mjs",
    "postsync-spec": "npm run bundle-spec",
    "bundle-spec": "redocly bundle spec/openapi.yaml -o spec/openapi.bundled.yaml",
    "postbundle-spec": "npm run gen-types",
    "gen-types": "openapi-typescript spec/openapi.bundled.yaml -o src/api/generated/openapi.d.ts"
  }
}
```

## 主要依存

```
dependencies:
  @ionic/vue, @ionic/vue-router
  vue, vue-router, pinia
  @capacitor/core, @capacitor/preferences
  ionicons

devDependencies:
  @capacitor/cli
  typescript, vite, @vitejs/plugin-vue, vue-tsc
  openapi-typescript
  @redocly/cli
```

iOS は対象外のため `@capacitor/ios` は含めない。Android は後追いのため初期 deps からは除外（追加時に `@capacitor/android` を入れる）。

## .gitignore

```
node_modules/
dist/
spec/
src/api/generated/
.env.local
.env.*.local
.env.device
.env.production
android/
ios/
*.log
.DS_Store
```

## .gitattributes

```
*.yaml text eol=lf
*.yml  text eol=lf
*.json text eol=lf
*.ts   text eol=lf
*.vue  text eol=lf
*.md   text eol=lf
*.mjs  text eol=lf
```

## .npmrc

```
engine-strict=true
```

## README構成

1. 概要（Ionic + Vue + Capacitor、mock消費者）
2. 前提（Node.js 18+、Windows、mock別ターミナル起動）
3. セットアップ
   - `npm install`
   - `npm run sync-spec`（取得 + bundle + 型生成 自動）
4. 開発起動
   - `npm run dev`（Vite、`http://localhost:5173`）
   - `npm run serve`（Ionic CLI、`http://localhost:8100`）
5. mock 起動方法（別リポ参照）
6. 動作確認シナリオ（demo-user/demo-passwordログイン → items表示 → 追加・削除）
7. 環境変数説明（`.env.example` 参照）
8. Android プラットフォーム追加手順（参考）
   - `npm install @capacitor/android`
   - `npx cap add android`
   - `.env.device` 作成（PC LAN IP）
   - `npx cap sync android`
   - `npx cap open android` または `npx cap run android --livereload`
9. backend 接続切替手順（VITE_API_BASE 変更）
10. bridge 連携追加手順（実装後に詳細を追記）
11. トラブルシュート

## 起動確認（実装後の検証）

- [ ] `npm install` 成功
- [ ] `npm run sync-spec` で `spec/openapi.yaml`、`spec/openapi.bundled.yaml`、`src/api/generated/openapi.d.ts` がそろう
- [ ] `npm run dev` で開発サーバー起動、ブラウザで `http://localhost:5173` 表示
- [ ] mock を別ターミナルで起動し、`/login` で `demo-user/demo-password` 入力 → `/items` 遷移
- [ ] サンプルアイテムA/B/C が表示される
- [ ] 「+」でモーダル開き、新規作成 → 一覧に反映（Prismは永続化しないので再fetchで消える、想定通り）
- [ ] スワイプ削除 → DELETE API 呼ばれる（同上）
- [ ] ログアウトボタンで `/login` に戻る、再アクセスで token 復元しないこと確認
- [ ] `npm run build` が型エラーなく完了

## 対象外（初期スコープから除外）

- bridge 連携（他セッション完成後に追加）
- 実 backend 接続検証（mock で代替、将来切替）
- Android プラットフォーム実機ビルド（README に手順のみ記載）
- iOS プラットフォーム
- ユニットテスト（PoCスコープ外、ブラウザ手動検証）
- E2Eテスト
- 国際化、ダークモード、CSS デザインカスタマイズ
- PUT 個別更新画面（ItemDetail）
- ページネーション、ソート、フィルタ
- リフレッシュトークン

## Git 運用

- scaffold完了時にリポジトリ内部で `git init` + 初期コミット（アシスタント実施）
- ローカル `git config user.name` / `user.email` 設定（本リポジトリのみ）
- GitHub リモート作成・push は api-spec/mock と同手順（`gh repo create m-miyawaki-m/mobile-app-poc-frontend --public --source=. --remote=origin --push`、ユーザ承認後）
- `spec/`, `src/api/generated/`, `node_modules/`, `dist/`, `.env.device`, `.env.production`, `android/`, `ios/` はgitignoreで除外

## 想定リスク・調査観点

- **Ionic CLI Vue scaffold の生成物との整合**: `ionic start` の生成構造に本設計を上書きする際の競合
- **CapacitorHttp + 開発時CORS**: ブラウザの場合 CapacitorHttp は通常の fetch にフォールバック、CORS 影響を受ける可能性 → mock 側のCORS設定で対応
- **vue-tsc とopenapi-typescript出力の整合**: 巨大なpaths型生成での型チェック性能
- **Pinia + Capacitor Preferences の起動順**: Preferences は非同期、起動時 `loadFromStorage()` のタイミング設計が必要

## 次のステップ

本書承認後、**writing-plans スキルで実装計画作成**。段階の想定:

1. `ionic start` で scaffold 生成（または手動セットアップ）
2. 不要ファイル削除 + 構造調整
3. `sync-spec.mjs`, package.json scripts追加 + 依存インストール
4. `openapi.bundled.yaml` 生成 + 型生成
5. `src/api/client.ts` 実装
6. `stores/auth.ts`, `stores/items.ts` 実装
7. `LoginPage.vue`, `ItemsPage.vue` 実装
8. `router/index.ts` 設定
9. `App.vue`, `main.ts` 調整
10. `.env.development`, `.env.example` 作成
11. README, .gitignore, .gitattributes, .npmrc
12. 動作確認（mock 起動 + frontend 起動 + ブラウザ操作）
13. git init + 初期コミット
14. GitHub push（ユーザ承認後）
