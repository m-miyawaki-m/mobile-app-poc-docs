# mobile-app-poc-frontend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ionic + Vue + Capacitor のモバイルフロントエンドを scaffold し、mock サーバーに対してログイン + アイテムCRUD の2画面を実装、ブラウザで動作確認、初期コミット + GitHub push まで完了させる。

**Architecture:** 手動 scaffold（ionic CLI 非対話依存を避ける）。`ionic start` ではなく必要ファイルを直接作成。`sync-spec.mjs` で api-spec を取得 → redocly bundle → openapi-typescript で型生成のチェイン。CapacitorHttp 薄ラッパー1本 + Pinia 2 store + Vue Router 認証ガード。bridge 連携・Android プラットフォーム追加は対象外（README に手順のみ）。

**Tech Stack:** Ionic 8.x (Vue), Vue 3, TypeScript, Vite, Capacitor 6.x, Pinia, @capacitor/preferences, @capacitor/core (CapacitorHttp), openapi-typescript, @redocly/cli

---

## File Structure

Working directory: `C:\Oracle\3df002\mobile-app\mobile-app-poc-frontend\`

**Created by this plan:**

```
mobile-app-poc-frontend/
├─ package.json
├─ package-lock.json
├─ index.html
├─ vite.config.ts
├─ tsconfig.json
├─ tsconfig.node.json
├─ capacitor.config.ts
├─ ionic.config.json
├─ .env.development
├─ .env.example
├─ .gitignore
├─ .gitattributes
├─ .npmrc
├─ README.md
├─ scripts/
│  └─ sync-spec.mjs
├─ public/
│  └─ favicon.ico                  # 空でOK
└─ src/
   ├─ main.ts
   ├─ App.vue
   ├─ vite-env.d.ts
   ├─ theme/
   │  └─ variables.css
   ├─ router/
   │  └─ index.ts
   ├─ stores/
   │  ├─ auth.ts
   │  └─ items.ts
   ├─ api/
   │  └─ client.ts
   └─ views/
      ├─ LoginPage.vue
      └─ ItemsPage.vue
```

**取得物（gitignored、本プランで生成だが版管理外）:**

```
mobile-app-poc-frontend/
├─ spec/                           # api-spec git clone + bundled.yaml
├─ src/api/generated/openapi.d.ts  # openapi-typescript 出力
├─ node_modules/
└─ dist/
```

各ファイルの責務:
- `package.json`: 依存と scripts
- `vite.config.ts`: Vite + Vue plugin
- `capacitor.config.ts`: Capacitor アプリID/名前
- `src/api/client.ts`: HTTP通信の唯一の窓口（CapacitorHttp wrap）
- `src/stores/auth.ts`: JWT 状態 + 永続化
- `src/stores/items.ts`: アイテム一覧の状態 + CRUD アクション
- `src/views/*.vue`: 各ページのUI + ロジック
- `src/router/index.ts`: ルート定義 + 認証ガード
- `src/App.vue`: ルートコンポーネント (`<ion-app>` + `<ion-router-outlet>`)
- `src/main.ts`: アプリエントリ + Ionic CSS imports + Pinia/Router登録
- `scripts/sync-spec.mjs`: api-spec取得（mock流用）

## Testing Approach

PoCのため自動テストはスコープ外。検証 = `npm run build`（vue-tsc型チェック）通過 + `npm run dev` 起動 + mock 別ターミナルで起動 + ブラウザでログイン→items操作。Task 11 で実施。

**Critical**: Git init は Task 12 まで遅延。途中でコミットしない。

---

## Task 1: プロジェクト初期化（package.json + npm install）

**Files:**
- Create: `mobile-app-poc-frontend/` (directory)
- Create: `mobile-app-poc-frontend/package.json`

- [ ] **Step 1: ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend"
```

- [ ] **Step 2: package.json を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/package.json`

```json
{
  "name": "mobile-app-poc-frontend",
  "version": "0.1.0",
  "private": true,
  "description": "Ionic + Vue + Capacitor frontend for mobile-app-poc, consumes mobile-app-poc-mock",
  "apiSpecRepo": "https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git",
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
  },
  "dependencies": {
    "@capacitor/core": "^6.0.0",
    "@capacitor/preferences": "^6.0.0",
    "@ionic/vue": "^8.0.0",
    "@ionic/vue-router": "^8.0.0",
    "ionicons": "^7.0.0",
    "pinia": "^2.1.0",
    "vue": "^3.4.0",
    "vue-router": "^4.2.0"
  },
  "devDependencies": {
    "@capacitor/cli": "^6.0.0",
    "@redocly/cli": "^2.0.0",
    "@vitejs/plugin-vue": "^5.0.0",
    "openapi-typescript": "^7.0.0",
    "typescript": "^5.4.0",
    "vite": "^5.0.0",
    "vue-tsc": "^2.0.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

- [ ] **Step 3: 依存インストール**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npm install
```

Expected: `added N packages` 出力、`node_modules/` と `package-lock.json` 生成。

- [ ] **Step 4: 主要 CLI 確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npx vite --version && npx redocly --version && npx openapi-typescript --version
```

Expected: 3つのバージョン番号が表示される。

---

## Task 2: gitignore / gitattributes / npmrc / .env

**Files:**
- Create: `mobile-app-poc-frontend/.gitignore`
- Create: `mobile-app-poc-frontend/.gitattributes`
- Create: `mobile-app-poc-frontend/.npmrc`
- Create: `mobile-app-poc-frontend/.env.development`
- Create: `mobile-app-poc-frontend/.env.example`

- [ ] **Step 1: .gitignore**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/.gitignore`

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

- [ ] **Step 2: .gitattributes**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/.gitattributes`

```
*.yaml text eol=lf
*.yml  text eol=lf
*.json text eol=lf
*.ts   text eol=lf
*.vue  text eol=lf
*.html text eol=lf
*.css  text eol=lf
*.md   text eol=lf
*.mjs  text eol=lf
```

- [ ] **Step 3: .npmrc**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/.npmrc`

```
engine-strict=true
```

- [ ] **Step 4: .env.development**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/.env.development`

```
VITE_API_BASE=http://localhost:4010
VITE_LOGIN_USERNAME=demo-user
VITE_LOGIN_PASSWORD=demo-password
```

- [ ] **Step 5: .env.example**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/.env.example`

```
# API接続先 (mock=4010, backend=8080, 本番=社内エンドポイント)
VITE_API_BASE=http://localhost:4010

# ログインフォーム初期値（PoC便宜用、本番では削除）
VITE_LOGIN_USERNAME=demo-user
VITE_LOGIN_PASSWORD=demo-password
```

---

## Task 3: scripts/sync-spec.mjs + 型生成チェイン実行

**Files:**
- Create: `mobile-app-poc-frontend/scripts/sync-spec.mjs`

- [ ] **Step 1: scripts ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/scripts"
```

- [ ] **Step 2: sync-spec.mjs を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/scripts/sync-spec.mjs`

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
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && node --check scripts/sync-spec.mjs
```

Expected: 出力なし。

- [ ] **Step 4: spec取得 + bundle + 型生成（チェイン全体実行）**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npm run sync-spec
```

Expected 出力（要旨）:
- `Cloning https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git into spec/`
- `spec sync complete.`
- `> npm run bundle-spec` → `Created a bundle for spec/openapi.yaml at spec/openapi.bundled.yaml`
- `> npm run gen-types` → 型生成（`src/api/generated/openapi.d.ts` 作成）

- [ ] **Step 5: 生成物の存在確認**

```bash
ls "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/spec/openapi.yaml" "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/spec/openapi.bundled.yaml" "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/api/generated/openapi.d.ts"
```

Expected: 3ファイルすべて存在。

---

## Task 4: ビルド設定 + theme（vite/tsconfig/capacitor/index.html/vite-env/variables.css）

**Files:**
- Create: `mobile-app-poc-frontend/vite.config.ts`
- Create: `mobile-app-poc-frontend/tsconfig.json`
- Create: `mobile-app-poc-frontend/tsconfig.node.json`
- Create: `mobile-app-poc-frontend/capacitor.config.ts`
- Create: `mobile-app-poc-frontend/ionic.config.json`
- Create: `mobile-app-poc-frontend/index.html`
- Create: `mobile-app-poc-frontend/src/vite-env.d.ts`
- Create: `mobile-app-poc-frontend/src/theme/variables.css`
- Create: `mobile-app-poc-frontend/public/favicon.ico` (空ファイル)

- [ ] **Step 1: src/, src/theme/, public/ ディレクトリ作成**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && mkdir -p src/theme public
```

- [ ] **Step 2: vite.config.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/vite.config.ts`

```ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import path from 'node:path';

export default defineConfig({
  plugins: [vue()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 5173,
  },
});
```

- [ ] **Step 3: tsconfig.json**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "jsx": "preserve",
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "lib": ["ESNext", "DOM"],
    "skipLibCheck": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["vite/client"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "src/**/*.tsx", "src/**/*.vue"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

- [ ] **Step 4: tsconfig.node.json**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/tsconfig.node.json`

```json
{
  "compilerOptions": {
    "composite": true,
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowSyntheticDefaultImports": true
  },
  "include": ["vite.config.ts"]
}
```

- [ ] **Step 5: capacitor.config.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/capacitor.config.ts`

```ts
import type { CapacitorConfig } from '@capacitor/cli';

const config: CapacitorConfig = {
  appId: 'jp.example.mobileapppoc',
  appName: 'mobile-app-poc',
  webDir: 'dist',
};

export default config;
```

- [ ] **Step 6: ionic.config.json**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/ionic.config.json`

```json
{
  "name": "mobile-app-poc-frontend",
  "integrations": {
    "capacitor": {}
  },
  "type": "vue"
}
```

- [ ] **Step 7: index.html**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/index.html`

```html
<!DOCTYPE html>
<html lang="ja">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/x-icon" href="/favicon.ico" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <title>mobile-app-poc</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

- [ ] **Step 8: src/vite-env.d.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/vite-env.d.ts`

```ts
/// <reference types="vite/client" />

declare module '*.vue' {
  import type { DefineComponent } from 'vue';
  const component: DefineComponent<{}, {}, any>;
  export default component;
}

interface ImportMetaEnv {
  readonly VITE_API_BASE: string;
  readonly VITE_LOGIN_USERNAME: string;
  readonly VITE_LOGIN_PASSWORD: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

- [ ] **Step 9: src/theme/variables.css（最小）**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/theme/variables.css`

```css
/* Ionic CSS Variables - PoC最小版 */
:root {
  --ion-color-primary: #3880ff;
  --ion-color-primary-rgb: 56, 128, 255;
  --ion-color-primary-contrast: #ffffff;
  --ion-color-primary-contrast-rgb: 255, 255, 255;
  --ion-color-primary-shade: #3171e0;
  --ion-color-primary-tint: #4c8dff;
}
```

- [ ] **Step 10: public/favicon.ico（空ファイル）**

```bash
touch "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/public/favicon.ico"
```

---

## Task 5: src/api/client.ts

**Files:**
- Create: `mobile-app-poc-frontend/src/api/client.ts`

- [ ] **Step 1: src/api/ ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/api"
```

- [ ] **Step 2: client.ts を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/api/client.ts`

```ts
import { CapacitorHttp } from '@capacitor/core';
import { useAuthStore } from '@/stores/auth';

const BASE_URL = import.meta.env.VITE_API_BASE;

type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';

export class ApiError extends Error {
  constructor(
    public status: number,
    public body: unknown,
    message: string,
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export async function request<T = unknown>(
  method: HttpMethod,
  path: string,
  body?: unknown,
): Promise<T> {
  const auth = useAuthStore();
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
  };
  if (auth.token) {
    headers['Authorization'] = `Bearer ${auth.token}`;
  }

  const response = await CapacitorHttp.request({
    method,
    url: `${BASE_URL}${path}`,
    headers,
    data: body,
  });

  if (response.status >= 400) {
    throw new ApiError(
      response.status,
      response.data,
      `API ${method} ${path} failed with ${response.status}`,
    );
  }

  return response.data as T;
}
```

---

## Task 6: Pinia stores（auth + items）

**Files:**
- Create: `mobile-app-poc-frontend/src/stores/auth.ts`
- Create: `mobile-app-poc-frontend/src/stores/items.ts`

- [ ] **Step 1: src/stores/ ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/stores"
```

- [ ] **Step 2: stores/auth.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/stores/auth.ts`

```ts
import { defineStore } from 'pinia';
import { Preferences } from '@capacitor/preferences';
import { CapacitorHttp } from '@capacitor/core';
import type { paths } from '@/api/generated/openapi';

const STORAGE_KEY = 'mobile-app-poc.auth';
const BASE_URL = import.meta.env.VITE_API_BASE;

type LoginResponse =
  paths['/auth/login']['post']['responses']['200']['content']['application/json'];

interface AuthState {
  token: string | null;
  expiresAt: number | null;
}

export const useAuthStore = defineStore('auth', {
  state: (): AuthState => ({
    token: null,
    expiresAt: null,
  }),
  getters: {
    isAuthenticated(state): boolean {
      if (!state.token || !state.expiresAt) return false;
      return Date.now() < state.expiresAt;
    },
  },
  actions: {
    async login(username: string, password: string): Promise<void> {
      const response = await CapacitorHttp.request({
        method: 'POST',
        url: `${BASE_URL}/auth/login`,
        headers: { 'Content-Type': 'application/json' },
        data: { username, password },
      });

      if (response.status >= 400) {
        throw new Error(`ログイン失敗 (${response.status})`);
      }

      const data = response.data as LoginResponse;
      this.token = data.token;
      this.expiresAt = Date.now() + data.expiresIn * 1000;

      await Preferences.set({
        key: STORAGE_KEY,
        value: JSON.stringify({
          token: this.token,
          expiresAt: this.expiresAt,
        }),
      });
    },

    async logout(): Promise<void> {
      this.token = null;
      this.expiresAt = null;
      await Preferences.remove({ key: STORAGE_KEY });
    },

    async loadFromStorage(): Promise<void> {
      const { value } = await Preferences.get({ key: STORAGE_KEY });
      if (!value) return;
      try {
        const parsed = JSON.parse(value) as AuthState;
        if (parsed.expiresAt && Date.now() < parsed.expiresAt) {
          this.token = parsed.token;
          this.expiresAt = parsed.expiresAt;
        } else {
          await Preferences.remove({ key: STORAGE_KEY });
        }
      } catch {
        await Preferences.remove({ key: STORAGE_KEY });
      }
    },
  },
});
```

- [ ] **Step 3: stores/items.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/stores/items.ts`

```ts
import { defineStore } from 'pinia';
import { request } from '@/api/client';
import type { paths } from '@/api/generated/openapi';

type ItemsListResponse =
  paths['/items']['get']['responses']['200']['content']['application/json'];
type Item = ItemsListResponse[number];
type NewItem =
  paths['/items']['post']['requestBody']['content']['application/json'];

interface ItemsState {
  items: Item[];
  loading: boolean;
  error: string | null;
}

export const useItemsStore = defineStore('items', {
  state: (): ItemsState => ({
    items: [],
    loading: false,
    error: null,
  }),
  actions: {
    async fetchAll(): Promise<void> {
      this.loading = true;
      this.error = null;
      try {
        this.items = await request<ItemsListResponse>('GET', '/items');
      } catch (e) {
        this.error = e instanceof Error ? e.message : String(e);
      } finally {
        this.loading = false;
      }
    },

    async create(input: NewItem): Promise<void> {
      this.error = null;
      try {
        await request('POST', '/items', input);
        await this.fetchAll();
      } catch (e) {
        this.error = e instanceof Error ? e.message : String(e);
        throw e;
      }
    },

    async remove(id: string): Promise<void> {
      this.error = null;
      try {
        await request('DELETE', `/items/${id}`);
        await this.fetchAll();
      } catch (e) {
        this.error = e instanceof Error ? e.message : String(e);
        throw e;
      }
    },
  },
});
```

---

## Task 7: src/views/LoginPage.vue

**Files:**
- Create: `mobile-app-poc-frontend/src/views/LoginPage.vue`

- [ ] **Step 1: src/views/ ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/views"
```

- [ ] **Step 2: LoginPage.vue**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/views/LoginPage.vue`

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonItem,
  IonLabel,
  IonInput,
  IonButton,
  IonToast,
} from '@ionic/vue';
import { useAuthStore } from '@/stores/auth';

const router = useRouter();
const auth = useAuthStore();

const username = ref(import.meta.env.VITE_LOGIN_USERNAME ?? '');
const password = ref(import.meta.env.VITE_LOGIN_PASSWORD ?? '');
const errorMessage = ref<string | null>(null);
const showError = ref(false);
const loggingIn = ref(false);

async function onLogin() {
  errorMessage.value = null;
  loggingIn.value = true;
  try {
    await auth.login(username.value, password.value);
    await router.replace('/items');
  } catch (e) {
    errorMessage.value = e instanceof Error ? e.message : String(e);
    showError.value = true;
  } finally {
    loggingIn.value = false;
  }
}
</script>

<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>ログイン</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content class="ion-padding">
      <ion-item>
        <ion-label position="stacked">ユーザー名</ion-label>
        <ion-input v-model="username" autocomplete="username" />
      </ion-item>
      <ion-item>
        <ion-label position="stacked">パスワード</ion-label>
        <ion-input v-model="password" type="password" autocomplete="current-password" />
      </ion-item>
      <ion-button
        expand="block"
        class="ion-margin-top"
        :disabled="loggingIn || !username || !password"
        @click="onLogin"
      >
        {{ loggingIn ? '送信中...' : 'ログイン' }}
      </ion-button>
      <ion-toast
        :is-open="showError"
        :message="errorMessage ?? ''"
        :duration="3000"
        color="danger"
        @did-dismiss="showError = false"
      />
    </ion-content>
  </ion-page>
</template>
```

---

## Task 8: src/views/ItemsPage.vue

**Files:**
- Create: `mobile-app-poc-frontend/src/views/ItemsPage.vue`

- [ ] **Step 1: ItemsPage.vue**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/views/ItemsPage.vue`

```vue
<script setup lang="ts">
import { ref, onMounted, computed } from 'vue';
import { useRouter } from 'vue-router';
import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonButton,
  IonButtons,
  IonList,
  IonItem,
  IonLabel,
  IonItemSliding,
  IonItemOptions,
  IonItemOption,
  IonFab,
  IonFabButton,
  IonIcon,
  IonModal,
  IonInput,
  IonToast,
} from '@ionic/vue';
import { add } from 'ionicons/icons';
import { useAuthStore } from '@/stores/auth';
import { useItemsStore } from '@/stores/items';

const router = useRouter();
const auth = useAuthStore();
const store = useItemsStore();

const showError = ref(false);
const errorText = computed(() => store.error ?? '');

const isModalOpen = ref(false);
const newName = ref('');
const newQuantity = ref<number | null>(null);

onMounted(async () => {
  await store.fetchAll();
  if (store.error) showError.value = true;
});

function openCreate() {
  newName.value = '';
  newQuantity.value = null;
  isModalOpen.value = true;
}

async function submitCreate() {
  if (!newName.value || newQuantity.value === null) return;
  try {
    await store.create({ name: newName.value, quantity: newQuantity.value });
    isModalOpen.value = false;
  } catch {
    showError.value = true;
  }
}

async function deleteItem(id: string) {
  try {
    await store.remove(id);
  } catch {
    showError.value = true;
  }
}

async function onLogout() {
  await auth.logout();
  await router.replace('/login');
}
</script>

<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>アイテム</ion-title>
        <ion-buttons slot="end">
          <ion-button @click="onLogout">ログアウト</ion-button>
        </ion-buttons>
      </ion-toolbar>
    </ion-header>
    <ion-content>
      <ion-list v-if="store.items.length > 0">
        <ion-item-sliding v-for="item in store.items" :key="item.id">
          <ion-item>
            <ion-label>
              <h2>{{ item.name }}</h2>
              <p>quantity: {{ item.quantity }}</p>
            </ion-label>
          </ion-item>
          <ion-item-options side="end">
            <ion-item-option color="danger" @click="deleteItem(item.id)">削除</ion-item-option>
          </ion-item-options>
        </ion-item-sliding>
      </ion-list>
      <div v-else-if="!store.loading" class="ion-padding ion-text-center">
        アイテムがありません。
      </div>

      <ion-fab vertical="bottom" horizontal="end" slot="fixed">
        <ion-fab-button @click="openCreate">
          <ion-icon :icon="add" />
        </ion-fab-button>
      </ion-fab>

      <ion-modal :is-open="isModalOpen" @did-dismiss="isModalOpen = false">
        <ion-header>
          <ion-toolbar>
            <ion-title>新規アイテム</ion-title>
            <ion-buttons slot="end">
              <ion-button @click="isModalOpen = false">キャンセル</ion-button>
            </ion-buttons>
          </ion-toolbar>
        </ion-header>
        <ion-content class="ion-padding">
          <ion-item>
            <ion-label position="stacked">名前</ion-label>
            <ion-input v-model="newName" />
          </ion-item>
          <ion-item>
            <ion-label position="stacked">数量</ion-label>
            <ion-input v-model.number="newQuantity" type="number" min="0" />
          </ion-item>
          <ion-button
            expand="block"
            class="ion-margin-top"
            :disabled="!newName || newQuantity === null"
            @click="submitCreate"
          >
            保存
          </ion-button>
        </ion-content>
      </ion-modal>

      <ion-toast
        :is-open="showError"
        :message="errorText"
        :duration="3000"
        color="danger"
        @did-dismiss="showError = false"
      />
    </ion-content>
  </ion-page>
</template>
```

---

## Task 9: router + main + App

**Files:**
- Create: `mobile-app-poc-frontend/src/router/index.ts`
- Create: `mobile-app-poc-frontend/src/App.vue`
- Create: `mobile-app-poc-frontend/src/main.ts`

- [ ] **Step 1: src/router/ ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/router"
```

- [ ] **Step 2: router/index.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/router/index.ts`

```ts
import { createRouter, createWebHistory } from '@ionic/vue-router';
import type { RouteRecordRaw } from 'vue-router';
import { useAuthStore } from '@/stores/auth';

const routes: RouteRecordRaw[] = [
  {
    path: '/',
    redirect: '/items',
  },
  {
    path: '/login',
    component: () => import('@/views/LoginPage.vue'),
  },
  {
    path: '/items',
    component: () => import('@/views/ItemsPage.vue'),
    meta: { requiresAuth: true },
  },
];

const router = createRouter({
  history: createWebHistory(import.meta.env.BASE_URL),
  routes,
});

router.beforeEach((to) => {
  const auth = useAuthStore();
  if (to.meta.requiresAuth && !auth.isAuthenticated) {
    return '/login';
  }
  if (to.path === '/login' && auth.isAuthenticated) {
    return '/items';
  }
});

export default router;
```

- [ ] **Step 3: App.vue**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/App.vue`

```vue
<script setup lang="ts">
import { IonApp, IonRouterOutlet } from '@ionic/vue';
</script>

<template>
  <ion-app>
    <ion-router-outlet />
  </ion-app>
</template>
```

- [ ] **Step 4: main.ts**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/src/main.ts`

```ts
import { createApp } from 'vue';
import { createPinia } from 'pinia';
import { IonicVue } from '@ionic/vue';

import App from './App.vue';
import router from './router';
import { useAuthStore } from './stores/auth';

/* Core CSS required for Ionic components to work properly */
import '@ionic/vue/css/core.css';

/* Basic CSS for apps built with Ionic */
import '@ionic/vue/css/normalize.css';
import '@ionic/vue/css/structure.css';
import '@ionic/vue/css/typography.css';

/* Optional CSS utils */
import '@ionic/vue/css/padding.css';
import '@ionic/vue/css/float-elements.css';
import '@ionic/vue/css/text-alignment.css';
import '@ionic/vue/css/text-transformation.css';
import '@ionic/vue/css/flex-utils.css';
import '@ionic/vue/css/display.css';

/* Theme variables */
import './theme/variables.css';

const pinia = createPinia();
const app = createApp(App).use(IonicVue).use(pinia).use(router);

const auth = useAuthStore();
auth.loadFromStorage().finally(() => {
  router.isReady().then(() => {
    app.mount('#app');
  });
});
```

- [ ] **Step 5: 型チェック**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npx vue-tsc --noEmit
```

Expected: エラーなし（出力なしまたは「Found 0 errors.」）。型エラーが出た場合は該当ファイルを修正。

---

## Task 10: README.md

**Files:**
- Create: `mobile-app-poc-frontend/README.md`

- [ ] **Step 1: README.md を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend/README.md`

````markdown
# mobile-app-poc-frontend

Ionic + Vue + Capacitor によるモバイルアプリPoCのフロントエンド。
`mobile-app-poc-mock` の API に対してログイン + アイテムCRUD（一覧/作成/削除）を実装する。

## 位置づけ

- mobile-app-poc-api-spec
- mobile-app-poc-mock — 開発時の API 接続先
- **mobile-app-poc-frontend** ← 本リポジトリ
- mobile-app-poc-bridge — 別セッションで開発中、実装後に統合
- mobile-app-poc-backend — 別セッションで開発中、実装後に切替

## 前提

- Node.js `>=18` 必須（LTS 22.x 推奨）
- OS: Windows ネイティブ（PowerShell 前提）
- `git` CLI が PATH 上にあること（spec取得用）
- mock サーバーを別ターミナルで起動可能なこと
- 社内イントラネット内での運用のみ

## セットアップ

```powershell
npm install
npm run sync-spec
```

`npm run sync-spec` は以下を一括実行:
1. `./spec/` に api-spec を git clone（初回）または pull
2. `redocly bundle` で `spec/openapi.bundled.yaml` 生成
3. `openapi-typescript` で `src/api/generated/openapi.d.ts` 生成

## 開発起動

```powershell
npm run dev
```

→ Vite が `http://localhost:5173` で待ち受け。

別ターミナルで `mobile-app-poc-mock` を起動:

```powershell
cd ..\mobile-app-poc-mock
npm run mock
```

## 動作確認

ブラウザで `http://localhost:5173` を開く:

1. ログイン画面に遷移（`demo-user` / `demo-password` が初期値）
2. 「ログイン」 → JWT 取得 → アイテム一覧へ
3. サンプルアイテムA/B/C が表示される
4. 右下「+」で新規作成モーダル → name/quantity 入力 → 保存
5. アイテムを左スワイプ → 「削除」
6. 右上「ログアウト」 → ログイン画面に戻る

注: Prism モックは永続化しないため、新規作成・削除は次回の取得時には反映されません（PoC の想定通り）。

## 環境変数

`.env.development` で開発時の値を設定。`.env.example` を参考にしてください。

| 変数 | 用途 |
|---|---|
| `VITE_API_BASE` | API 接続先（mock=4010、backend=8080） |
| `VITE_LOGIN_USERNAME` | ログインフォーム初期値（PoC便宜） |
| `VITE_LOGIN_PASSWORD` | ログインフォーム初期値（PoC便宜） |

実機検証時は `.env.device` を作成し `VITE_API_BASE=http://<PC LAN IP>:4010` に。

## ディレクトリ構成

```
mobile-app-poc-frontend/
├─ scripts/sync-spec.mjs   # api-spec取得スクリプト
├─ spec/                   # api-spec取得物 + bundled.yaml（gitignored）
├─ src/
│  ├─ api/                 # CapacitorHttp ラッパー + 型生成物
│  ├─ stores/              # Pinia stores (auth, items)
│  ├─ views/               # 画面コンポーネント
│  ├─ router/              # Vue Router 設定
│  ├─ theme/variables.css  # Ionic CSS 変数
│  ├─ App.vue
│  └─ main.ts
├─ .env.development
└─ package.json
```

## ビルド

```powershell
npm run build
```

→ `vue-tsc` で型チェック後、`vite build` で `dist/` に出力。

## Android プラットフォーム追加（参考）

実機検証段階で対応:

```powershell
npm install @capacitor/android
npx cap add android
# .env.device 作成 (VITE_API_BASE=http://<PC LAN IP>:4010)
npm run build
npx cap sync android
npx cap open android        # Android Studio 起動
# または
npx cap run android --livereload  # 実機 Live Reload
```

ファイアウォールで mock の 4010 を許可、mock も `npm run mock:lan` で起動すること。

## backend 接続切替（実装後）

`.env.production` を作成し `VITE_API_BASE=http://<backend host>:8080` に変更。
`npm run build -- --mode production` で本番ビルド。

## bridge 連携追加（実装後）

`mobile-app-poc-bridge` 完成後、本リポジトリに追加手順を記載予定。
基本フロー: bridge を npm install (git URL or local file) → import → 呼び出し。

## トラブルシュート

| 症状 | 対処 |
|---|---|
| `npm install` で実行ポリシーエラー | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `npm run sync-spec` 失敗 | git CLI が PATH にあるか、リポジトリURLにアクセス可能か |
| `Cannot find module '@/api/generated/openapi'` | `npm run sync-spec` 未実行。型生成からやり直す |
| ログインで CORS エラー | mock 側で CORS 有効か（Prism は既定で `*` を返す） |
| ブラウザで CapacitorHttp 動作しない | Web では fetch にフォールバック。mock 側のCORS確認 |
| アイテム一覧が空のまま | mock が起動しているか、`VITE_API_BASE` が正しいか |
````

---

## Task 11: 動作確認（mock + frontend ブラウザ動作）

このタスクは新規ファイルを作らず、これまでの成果物が機能することを実機確認する。

- [ ] **Step 1: 型チェックとビルド成功を確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npm run build
```

Expected: 型エラーなし、`dist/` 配下にビルド成果物が生成される。

- [ ] **Step 2: mock を別ターミナル相当でバックグラウンド起動**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-mock" && npm run mock
```

`run_in_background: true` で起動。Expected ログ: `Prism is listening on http://127.0.0.1:4010`

- [ ] **Step 3: mock の動作確認（curl）**

```bash
curl -s -H "Authorization: Bearer x" http://localhost:4010/items | head -c 200
```

Expected: `サンプルアイテムA` を含む配列レスポンス。

- [ ] **Step 4: frontend を別バックグラウンドで起動**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && npm run dev
```

`run_in_background: true` で起動。Expected ログ: `VITE v5.x.x  ready in NNN ms` と `Local:   http://localhost:5173/`

- [ ] **Step 5: フロントエンドの応答確認**

```bash
curl -sf http://localhost:5173/ -o /dev/null && echo "frontend OK"
```

Expected: `frontend OK`

- [ ] **Step 6: ブラウザ動作確認の手順を控える（実ブラウザ確認は人手）**

報告時に以下を記載:
- frontend が起動した URL
- mock の状態
- 推奨: ブラウザで `http://localhost:5173` を開いて以下を手動確認
  - ログイン画面が表示
  - 「ログイン」で `/items` 遷移
  - サンプルアイテムA/B/C 表示
  - 「+」で新規追加 → 一覧反映
  - スワイプで削除（mockは永続化しないので消える挙動は確認まで）
  - 「ログアウト」で `/login` に戻る

- [ ] **Step 7: バックグラウンドプロセスを停止**

frontend と mock の background プロセスを終了する（execution環境による：TaskStop など）。

```bash
# 残留プロセス確認
netstat -ano -p tcp | grep -E ":(4010|5173).*LISTENING"
# 必要に応じて taskkill //F //PID <pid>
```

---

## Task 12: git init + ローカル user 設定 + 初期コミット

**Files:**
- Create: `mobile-app-poc-frontend/.git/` (git init による自動生成)

- [ ] **Step 1: git init + main ブランチ + ローカル user**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && git init && git symbolic-ref HEAD refs/heads/main && git config user.name "m-miyawaki" && git config user.email "miyawakifamilymyam@gmail.com"
```

- [ ] **Step 2: ステージ + 確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && git add . && git status --short
```

Expected: 以下のみ追加対象（`spec/`, `node_modules/`, `dist/`, `src/api/generated/` 等は除外）:
- `.gitattributes`, `.gitignore`, `.npmrc`
- `.env.development`, `.env.example`
- `README.md`
- `capacitor.config.ts`, `ionic.config.json`
- `index.html`
- `package-lock.json`, `package.json`
- `public/favicon.ico`
- `scripts/sync-spec.mjs`
- `src/App.vue`, `src/main.ts`, `src/vite-env.d.ts`
- `src/api/client.ts`
- `src/router/index.ts`
- `src/stores/auth.ts`, `src/stores/items.ts`
- `src/theme/variables.css`
- `src/views/ItemsPage.vue`, `src/views/LoginPage.vue`
- `tsconfig.json`, `tsconfig.node.json`
- `vite.config.ts`

- [ ] **Step 3: 初期コミット**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && git commit -m "$(cat <<'EOF'
chore: initial scaffold of mobile-app-poc-frontend

Ionic + Vue + Capacitor frontend with login + items CRUD against
the mock server. CapacitorHttp wrapper, Pinia stores, openapi-typescript
type generation via redocly bundle. Android platform setup deferred,
bridge integration deferred.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: コミット成功、ファイル数報告。

- [ ] **Step 4: 最終確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && git log --oneline && git status
```

Expected:
- `git log --oneline`: 1行の初期コミット
- `git status`: `nothing to commit, working tree clean`

---

## Task 13: GitHub repo 作成 + push（ユーザ承認後）

**重要**: このタスクは **コントローラがユーザに公開リポジトリ作成 + push の許可を取ってから** 実行する。

- [ ] **Step 1: ユーザ確認（コントローラが実施）**

ユーザに以下を確認:
- リポジトリ名: `mobile-app-poc-frontend`
- 公開設定: api-spec/mock と同じ public で良いか
- アカウント: `m-miyawaki-m`

- [ ] **Step 2: gh で公開リポジトリ作成 + push**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && gh repo create m-miyawaki-m/mobile-app-poc-frontend --public --source=. --remote=origin --push
```

Expected: URL表示、`branch 'main' set up to track 'origin/main'`、`* [new branch]      HEAD -> main`。

- [ ] **Step 3: 確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-frontend" && git remote -v && git log --oneline
```

Expected:
- `origin   https://github.com/m-miyawaki-m/mobile-app-poc-frontend.git (fetch/push)`
- 初期コミットが表示される

---

## 完了条件

- [ ] 上記Task 1〜13すべて完了
- [ ] `npm run sync-spec` で `spec/`, `spec/openapi.bundled.yaml`, `src/api/generated/openapi.d.ts` がそろう
- [ ] `npm run build` が型エラーなく通過
- [ ] `npm run dev` で Vite サーバー起動 (`http://localhost:5173`)
- [ ] mock 起動状態でブラウザから login → items 画面遷移、items 表示、新規作成、削除、ログアウトまで一通り動作
- [ ] `git log --oneline` に初期コミット存在
- [ ] GitHub に `m-miyawaki-m/mobile-app-poc-frontend` が public で存在し、push済み

---

## 注意事項

1. **Git 初期化は Task 12 まで遅延**。途中でコミットしない。
2. **`spec/` `src/api/generated/` `node_modules/` `dist/` を絶対にコミットしない**。`.gitignore` で除外、Step 2 の git status で混入チェック。
3. **絶対パスで操作**（Windowsバス環境での作業ディレクトリ混乱を避けるため）。
4. **バックグラウンド mock/dev サーバーの停止忘れ**に注意（Task 11 Step 7）。
5. **GitHub repo 作成は破壊性のある外部操作**。Task 13 はユーザ確認必須。
6. **`@capacitor/android` は初期deps から除外**。実機検証段階でユーザが追加。
7. **Vue ファイルの triple-backtick** に注意（テンプレート部分が含まれる）。実装時はファイル直接書き込みで対応。
