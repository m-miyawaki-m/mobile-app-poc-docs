# Ionic (Vue) + Capacitor 開発ワークフロー総覧

本ドキュメントは、`mobile-app-poc` プロジェクトで採用した **Ionic + Vue + Capacitor** 構成のアプリを「ゼロから設計・実装・実機起動する流れ」と、その開発環境として **VS Code を主・Android Studio をビルド専用に絞る構成** をまとめたもの。

## 想定読者

- Ionic Vue を初めて触る開発者
- Web/JS は分かるがネイティブ (Android/Kotlin) は経験が浅い人
- Android Studio 中心の重い IDE 運用を避けたい人

## 用語

| 用語 | 意味 |
|---|---|
| Ionic | Web 技術でネイティブ風 UI を作る UI フレームワーク |
| Capacitor | Web アプリを iOS/Android にラップするネイティブブリッジ |
| CapacitorHttp | Capacitor 標準の HTTP クライアント（CORS の影響を受けない） |
| CapacitorPlugin | ネイティブ機能を JS から呼び出すためのプラグイン仕組み |
| Custom Plugin | 自前で書く CapacitorPlugin（本PoCでは `mobile-app-poc-bridge` がこれ） |

---

# 第1部: 設計 → 実装 → 実機起動 までの一気通貫フロー

## 1. このスタックの全体像

```
            ┌─────────────────────┐
            │   Vue 3 + Pinia     │  ← UI/状態管理 (TypeScript)
            │   + Ionic 8 (UI)    │
            └──────────┬──────────┘
                       │ CapacitorHttp
                       ▼
            ┌─────────────────────┐
            │  HTTP API (mock or  │  ← Prism / Spring Boot
            │  backend)           │
            └─────────────────────┘

            ┌─────────────────────┐
            │   Capacitor Core    │  ← Web ↔ Native 橋渡し
            └──────────┬──────────┘
                       │ CapacitorPlugin
                       ▼
            ┌─────────────────────┐
            │ Custom Plugin (aar) │  ← Java/Kotlin native
            │ = bridge            │
            └─────────────────────┘
```

ブラウザでも実機でも同じ Vue コードが動く。ネイティブ機能はプラグインで隔離。

## 2. 検討フェーズで決めること

実装に入る前に決めておくと手戻りが減る項目:

| 領域 | 決めること | 例 |
|---|---|---|
| API 契約 | OpenAPI で型を共通化するか | `openapi.yaml` を SSOT に |
| 認証 | JWT / OAuth / 独自 | JWT + Bearer ヘッダ |
| 画面 | 何画面、ルーティングは？ | login + items（最小） |
| 状態管理 | Pinia / Vuex / 自前 | Pinia |
| HTTP クライアント | fetch / axios / CapacitorHttp | CapacitorHttp（CORS回避） |
| 永続化 | localStorage / Capacitor Preferences | Preferences（クロスプラットフォーム） |
| ネイティブ機能 | 何を使うか、自作プラグイン要否 | 業務デバイス連携 → 自作 |
| 配布 | Play Store / 社内配布 | 社内のみ |
| 対応 OS | Android のみ / 両対応 | Android |

## 3. 環境準備（共通）

| ツール | 用途 | 推奨バージョン |
|---|---|---|
| Node.js | フロント開発・スクリプト | LTS 22.x |
| Java JDK | Android ビルド | 17 (Capacitor 6 推奨) |
| Android SDK | Android ビルド・実機接続 | 最新 |
| git | ソース管理・spec取得 | 最新 |
| VS Code | エディタ | 最新 |
| Android Studio | エミュレータ + 必要時のビルド | 最新 |

Windows ネイティブ前提。WSL2/Docker は使わない方針。

## 4. プロジェクト scaffold

`ionic start` ではなく **手動で必要ファイルだけ作る** 方針が再現性が高い。
本PoCの構造:

```
mobile-app-poc-frontend/
├─ scripts/sync-spec.mjs        # api-spec取得
├─ spec/                         # 取得物 + bundled.yaml（gitignored）
├─ src/
│  ├─ api/                       # CapacitorHttp ラッパー + 型生成
│  ├─ stores/                    # Pinia
│  ├─ views/                     # 画面
│  ├─ router/                    # 認証ガード
│  ├─ theme/variables.css
│  ├─ App.vue
│  └─ main.ts
├─ public/
├─ .env.development              # VITE_API_BASE 等
├─ capacitor.config.ts           # appId, appName
├─ ionic.config.json
├─ vite.config.ts
├─ tsconfig.json
├─ package.json
└─ README.md
```

各ファイルの責務を1つに保つことで AI 補助・人間の理解の両方が楽になる。

## 5. API 連携層

Schema-First で型を openapi から自動生成:

```
api-spec (openapi.yaml)
   │
   ▼ npm run sync-spec  (frontend側)
spec/openapi.yaml
   │
   ▼ redocly bundle
spec/openapi.bundled.yaml      ← externalValue を全インライン化
   │
   ▼ openapi-typescript
src/api/generated/openapi.d.ts ← TS型
```

呼び出し:

```ts
import type { paths } from '@/api/generated/openapi';
type Items = paths['/items']['get']['responses']['200']['content']['application/json'];
```

`src/api/client.ts` で薄いラッパー1本（CapacitorHttp + 認証ヘッダ自動付与）。

**学び**: Prism CLI 5.x は `externalValue` を解決しないため、`@redocly/cli bundle` で `value:` にインライン化する一段が必須。

## 6. 状態管理 (Pinia)

```
stores/auth.ts:
  state:    token, expiresAt
  getters:  isAuthenticated
  actions:  login, logout, loadFromStorage  ← Capacitor Preferences

stores/items.ts:
  state:    items, loading, error
  actions:  fetchAll, create, remove
```

API 呼び出しは store の action に閉じ込め、view は state を読むだけ。

**学び**: 起動時に `auth.loadFromStorage()` を await してから mount しないと、初回レンダーで未認証扱いになる。

## 7. ルーティング + 認証ガード

```ts
router.beforeEach((to) => {
  const auth = useAuthStore();
  if (to.meta.requiresAuth && !auth.isAuthenticated) return '/login';
  if (to.path === '/login' && auth.isAuthenticated) return '/items';
});
```

## 8. ブラウザでの開発サイクル

```
ターミナル1: cd mock     && npm run mock      # Prism @ :4010
ターミナル2: cd frontend && npm run dev       # Vite @ :5173
ブラウザ: http://localhost:5173
```

または親ディレクトリの launcher (`npm run dev`) で両方一括起動。

ホットリロード:
- Vue/TS/CSS は即反映
- `openapi.yaml` 変更時は `npm run sync-spec` で型再生成 + Vite が変更検知

**学び**: ブラウザでは CapacitorHttp が `fetch` にフォールバックするため CORS の影響を受ける。Prism の既定 CORS で大体回避できる。

## 9. 実機ビルド準備（Android）

frontend に Capacitor Android プラットフォームを入れる:

```powershell
npm install @capacitor/android@^6
npx cap add android
```

`android/` ディレクトリが生成される。Capacitor 同梱の `.gitignore` がビルド成果物のみ除外する設定なので、`android/` 自体はコミット対象。

`capacitor.config.ts`:

```ts
{
  appId: 'jp.example.mobileapppoc',
  appName: 'mobile-app-poc',
  webDir: 'dist',
  // server: { cleartext: true }  // http (非HTTPS) を使う場合
}
```

**学び**: Capacitor 6 + Android で `http://` を使うと cleartext 制限に当たる場合があり、必要に応じ `server.cleartext: true` を追加。

## 10. 実機起動 3 パターン

| パターン | コマンド | 用途 |
|---|---|---|
| APK 直焼き | `npm run android:sync` → Android Studio で Run | 通常リリース手前のビルド検証 |
| Android Studio Run | `npm run android:open` → ▶ | UI 確認、デバッガ使用 |
| Live Reload | `npm run android:livereload` | コード変更を即実機反映（開発中） |

**LAN 接続要件（実機/エミュレータ共通）**:

```
PC ─── Wi-Fi ─── スマホ
  │                │
  ├─ mock @ 0.0.0.0:4010    ← npm run mock:lan で起動
  └─ Vite @ 0.0.0.0:5173    ← Live Reload 時のみ
```

`localhost` は実機にとって「実機自身」を指すため、必ず PC の LAN IP を使う。

`.env.device`:
```
VITE_API_BASE=http://192.168.1.10:4010
```

Windows ファイアウォール許可:
```powershell
New-NetFirewallRule -DisplayName "Prism mock 4010" -Direction Inbound -LocalPort 4010 -Protocol TCP -Action Allow
```

## 11. mock → backend 切替

backend 完成後に `.env.production` を作成:

```
VITE_API_BASE=http://<backend-host>:8080
```

ビルド: `npm run build -- --mode production`

`vite.config.ts` の env mode で `.env.production` が読み込まれる。

## 12. ネイティブ機能（bridge）追加

aar 取り込みは独立リポ `mobile-app-poc-bridge` で行う:

```
bridge (npm package)
  ├─ android/libs/xxx.aar       ← デバイス提供元提供
  ├─ android/src/.../Plugin.kt  ← @CapacitorPlugin
  └─ src/index.ts               ← TS インターフェース
```

frontend から install:
```
npm install ../mobile-app-poc-bridge      # ローカル参照
# または git URL
npm install git+ssh://internal-git/...
```

呼び出し:
```ts
import { DeviceBridge } from 'mobile-app-poc-bridge';
const result = await DeviceBridge.scan();
```

詳細は `mobile-app-poc-bridge` 側 README に委譲。

## 13. PoC で判明した落とし穴

| 課題 | 対処 |
|---|---|
| Prism が `externalValue` を解決しない | `@redocly/cli bundle` で `value:` にインライン化 |
| Prism が auth ヘッダ存在を検証する | 任意の `Bearer x` でも OK、JWT署名検証はしない |
| Windows で CRLF 混入 | `.gitattributes` で `* eol=lf` を強制 |
| Capacitor 6 + http (cleartext) 接続失敗 | `capacitor.config.ts` の `server.cleartext: true` |
| `cap sync` 忘れで古い JS が実機に残る | `android:sync` を必ず通す習慣 |
| Capacitor Preferences の async 起動順 | `loadFromStorage().finally(() => mount)` |
| ブラウザでの CapacitorHttp は fetch fallback | CORS 設定要確認 |
| `localhost` は実機では実機自身 | 必ず PC の LAN IP を使う |

---

# 第2部: VS Code を主・Android Studio をビルド専用にする開発環境

主対象は `mobile-app-poc-bridge` プラグインの **Kotlin/Java native コード編集**。frontend の `android/` 配下は変更頻度が低いため副次的。

## なぜ VS Code 主体にするか

| 観点 | Android Studio | VS Code |
|---|---|---|
| 起動時間 | 重い | 軽い |
| Kotlin LSP | フル機能 | 70-80% 程度（fwcd拡張） |
| フロント (Vue/TS) | 弱い | 強い |
| Git | UI 充実 | 同左 |
| AI 連携 | 限定的 | 各種拡張が豊富 |
| エミュレータ管理 | 標準で楽 | CLI のみ（多少手間） |
| デバッガ (native) | 強力 | 弱い（ブレークポイント可だが快適度は劣る） |
| ビルド | 統合 | gradle CLI |

**結論**: 普段の編集は VS Code、エミュレータ起動と難解なネイティブデバッグだけ Android Studio。

## VS Code 側の準備

### 必須拡張機能

| 拡張 | 役割 |
|---|---|
| `vue.volar`（Vue Official） | Vue 3 + TS 補完 |
| `fwcd.kotlin` | Kotlin 言語サポート（LSP） |
| `vscjava.vscode-java-pack`（Java Extension Pack） | Java 補完・実行 |
| `vscjava.vscode-gradle` | Gradle 統合 |
| `redhat.vscode-xml` | AndroidManifest.xml 等 |
| `dbaeumer.vscode-eslint` | TS lint |
| `editorconfig.editorconfig` | スタイル統一 |

インストール一括（ターミナルから）:
```powershell
code --install-extension Vue.volar
code --install-extension fwcd.kotlin
code --install-extension vscjava.vscode-java-pack
code --install-extension vscjava.vscode-gradle
code --install-extension redhat.vscode-xml
code --install-extension dbaeumer.vscode-eslint
code --install-extension EditorConfig.EditorConfig
```

### 環境変数

`システムのプロパティ → 環境変数` または PowerShell で:

```powershell
[Environment]::SetEnvironmentVariable("JAVA_HOME", "C:\Program Files\Java\jdk-17", "User")
[Environment]::SetEnvironmentVariable("ANDROID_HOME", "$env:LOCALAPPDATA\Android\Sdk", "User")
$env:Path += ";$env:JAVA_HOME\bin;$env:ANDROID_HOME\platform-tools;$env:ANDROID_HOME\emulator"
```

PowerShell 再起動後 `java -version` / `adb version` / `emulator -version` で確認。

### VS Code 設定 (`.vscode/settings.json`)

各リポにそれぞれ:

**bridge 用** (`mobile-app-poc-bridge/.vscode/settings.json`):
```json
{
  "java.configuration.runtimes": [
    { "name": "JavaSE-17", "path": "C:\\Program Files\\Java\\jdk-17", "default": true }
  ],
  "java.import.gradle.enabled": true,
  "java.import.gradle.wrapper.enabled": true,
  "kotlin.languageServer.enabled": true,
  "kotlin.compiler.jvm.target": "17",
  "files.eol": "\n",
  "[xml]": { "editor.defaultFormatter": "redhat.vscode-xml" }
}
```

**frontend 用** (`mobile-app-poc-frontend/.vscode/settings.json`):
```json
{
  "typescript.tsdk": "node_modules\\typescript\\lib",
  "vue.server.hybridMode": true,
  "files.eol": "\n",
  "editor.formatOnSave": false
}
```

### Multi-root Workspace (`.code-workspace`)

5リポを1ウィンドウで開く `mobile-app-poc.code-workspace`（親ディレクトリに置く）:

```json
{
  "folders": [
    { "path": "mobile-app-poc-api-spec",  "name": "1. api-spec" },
    { "path": "mobile-app-poc-mock",      "name": "2. mock" },
    { "path": "mobile-app-poc-backend",   "name": "3. backend" },
    { "path": "mobile-app-poc-bridge",    "name": "4. bridge" },
    { "path": "mobile-app-poc-frontend",  "name": "5. frontend" },
    { "path": "docs",                     "name": "docs" }
  ],
  "settings": {
    "files.exclude": {
      "**/node_modules": true,
      "**/dist": true,
      "**/build": true,
      "**/.gradle": true
    }
  },
  "extensions": {
    "recommendations": [
      "Vue.volar",
      "fwcd.kotlin",
      "vscjava.vscode-java-pack",
      "vscjava.vscode-gradle"
    ]
  }
}
```

開く: `File → Open Workspace from File... → mobile-app-poc.code-workspace`

## Android Studio 側の役割を限定

「閉じておきたいが、これだけは Studio で」という線引き:

| 用途 | Android Studio 必須？ |
|---|---|
| エミュレータ作成 (AVD Manager) | 推奨（CLI でも可だが面倒） |
| SDK Manager で API レベル追加 | 推奨 |
| ネイティブのブレークポイントデバッグ | 必須（VS Code は限界あり） |
| UI Inspector / Profiler | 必須 |
| 通常のビルド・実行 | **不要**（VS Code + CLI で完結） |
| Gradle 同期 | **不要**（VS Code 拡張が代替） |
| ファイル編集 | **不要** |

エミュレータ起動だけは Studio で1回作っておけば、以降は CLI から:

```powershell
emulator -list-avds
emulator -avd Pixel_7_API_34
```

## ビルドは CLI 経由

### bridge (Capacitor Custom Plugin)

```powershell
cd mobile-app-poc-bridge\android
.\gradlew assembleRelease    # → android/build/outputs/aar/*.aar 生成
.\gradlew clean
```

VS Code の Gradle 拡張を使えば左サイドバーから GUI クリックでも実行可。

### frontend (Capacitor app)

```powershell
cd mobile-app-poc-frontend
npm run android:sync         # build + cap sync
npm run android:run          # USB接続実機にインストール
# または
cd android
.\gradlew assembleDebug      # APK生成 (android/app/build/outputs/apk/debug/)
adb install -r app\build\outputs\apk\debug\app-debug.apk
```

## デバッグ

### Web フロント側（Vue）

実機 Chrome での remote debug:

1. PC で Chrome を開いて `chrome://inspect`
2. 実機を USB 接続（USB デバッグ ON）
3. アプリを起動
4. 「inspect」をクリック → Chrome DevTools が開く
5. Sources タブでブレークポイント、Console / Network も使える

### ネイティブ側 (Kotlin/Java)

VS Code ターミナルで Logcat:

```powershell
adb logcat *:E AppName:V        # エラー全部 + AppName のログ全レベル
adb logcat | Select-String "Capacitor|MyPlugin"   # Capacitor + 自作プラグインだけ
```

詳細なブレークポイントデバッグは Android Studio で `Run → Attach Debugger to Android Process`。

## VS Code Multi-root Workspace の運用例

実際の典型ワークフロー:

```
朝:
  - VS Code で .code-workspace を開く
  - 親ディレクトリのターミナル: npm run dev (mock + frontend 起動)
  - ブラウザ: http://localhost:5173 で動作確認

ネイティブ修正が必要になったら:
  - bridge ペインを開く、Kotlin 編集
  - VS Code ターミナル: cd android && .\gradlew assembleDebug
  - frontend で: npm install ../mobile-app-poc-bridge (再リンク)
  - npm run android:sync && npm run android:run

エミュレータが欲しい時だけ:
  - 別シェル: emulator -avd Pixel_7_API_34
  - そのまま VS Code で開発継続

native 側でハマったら:
  - Android Studio を開く（モジュール選択不要、bridge or frontend/android を直接開く）
  - Run → Attach Debugger
  - 解決後 Studio は閉じる
```

## Kotlin LSP の限界（fwcd.kotlin の現状）

得意:
- 基本的な補完、定義ジャンプ
- 型推論
- 簡単なリファクタリング

苦手:
- Android Gradle DSL の補完
- aar 内のクラスの補完（解決できない場合あり）
- Compose プレビュー（Studio 必須）
- Kapt / KSP 生成コード参照

→ 「ネイティブのコアロジック編集は VS Code、Compose UI と複雑なリファクタリングは Studio」が現実解。

PoC の bridge プラグインは比較的小さいネイティブコード（@CapacitorPlugin クラス + aar 呼び出し）なので、VS Code でほぼ完結する想定。

---

## まとめ

- **Ionic Vue + Capacitor** は Web 経験者にとって入りやすい構成。Vue/Pinia/Vite は WebSPA そのもの、Capacitor だけネイティブを意識する
- **Schema-First**（OpenAPI → 型生成）と **bundle 対処** はチェーン化して `npm run sync-spec` 一発で完結させる
- **ブラウザで開発、実機で検証**の二段運用が効率的
- **VS Code 主・Android Studio 限定使用**で起動コストと記憶コストを下げられる。ネイティブの込み入ったデバッグだけ Studio に頼る

各リポの README は本ドキュメントよりも具体的な手順を提供する。本書は「全体像と判断軸」を整理したもの。
