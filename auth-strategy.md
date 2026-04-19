# 認証戦略 - JWT 詳細と代替案の検討

本ドキュメントは `mobile-app-poc` の認証方式について、JWT を中心に詳細と代替方式の比較をまとめたもの。
PoC 段階の現状方針 + 本番化に向けた選択肢を整理する。

## 現状サマリ

| 項目 | 現状 |
|---|---|
| 採用方式 | JWT (Bearer) |
| 発行 | mock (Prism) が固定の例トークンを返す（本物の署名なし） |
| 検証 | mock は署名検証しない（`Authorization: Bearer` ヘッダ存在のみチェック） |
| 保存場所 | Capacitor Preferences（Web=localStorage、Android=SharedPreferences） |
| 失効処理 | フロント側で `expiresAt` 判定のみ。サーバ失効なし |
| リフレッシュ | 未実装 |
| 想定本番 | Spring Boot 側で jjwt 等を使い HS256 / RS256 で本物発行・検証 |

PoC は「つなぎ目検証」が主目的のため、認証ロジックの厳密化は backend 実装フェーズに先送りしている。

---

## 第1部: JWT (JSON Web Token) の詳細

### 構造

```
xxxxx.yyyyy.zzzzz
└─┬─┘ └─┬─┘ └─┬─┘
header payload signature
```

各部分は Base64URL でエンコードされた JSON。

| 部分 | 内容 |
|---|---|
| Header | アルゴリズム種別 (`alg`) + トークン種別 (`typ`) |
| Payload | claims（ユーザID・有効期限・発行者など） |
| Signature | header+payload を秘密鍵で署名したもの（改ざん検知） |

### 例（mock が返すサンプル）

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJkZW1vLXVzZXIifQ.SAMPLE_SIGNATURE
```

デコード:
- header: `{"alg":"HS256"}`
- payload: `{"sub":"demo-user"}`
- signature: ダミー文字列 (PoC では検証しない)

### 主要な claims

| claim | 意味 | PoC で使う？ |
|---|---|---|
| `sub` | Subject (ユーザ ID) | ◎ |
| `iat` | Issued At (発行時刻、UNIX秒) | ○ |
| `exp` | Expiration (有効期限、UNIX秒) | ◎（フロントの判定にも使用） |
| `nbf` | Not Before (有効開始時刻) | △ |
| `iss` | Issuer (発行者) | △ |
| `aud` | Audience (受信者) | △ |
| `jti` | JWT ID（失効管理用） | ブラックリストする場合 |

本PoC の `LoginResponse` では `expiresIn`（秒数）を別途返しているが、本来は payload の `exp` で同じ情報を持てる。冗長だがフロントで Date 計算しやすい意図で残している。

### 署名アルゴリズム選択

| | HS256 (HMAC) | RS256 / ES256 (公開鍵) |
|---|---|---|
| 鍵運用 | 共通鍵 1つ | 秘密鍵 + 公開鍵ペア |
| 署名 | サーバーのみ | サーバーのみ |
| 検証 | 鍵を持つすべて（リスク） | 公開鍵を配るだけ |
| 適用シナリオ | 単一サービス内で完結 | 複数サービス・マイクロサービス |
| 推奨鍵長 | 256bit以上 | RSA 2048 / ECDSA P-256 |
| 性能 | 速い | 署名・検証ともやや重い |
| Spring Boot 実装 | jjwt + 共通鍵 | jjwt + KeyStore |

PoC スコープ: HS256 で十分。本番想定で複数システムから検証するなら RS256 推奨。

### JWT 運用の落とし穴

| 課題 | 対策 |
|---|---|
| トークン失効ができない（自己完結） | 短い有効期限 + リフレッシュトークン or サーバ側ブラックリスト (`jti`) |
| サイズが大きい（リクエスト毎に送る） | claims を最小限に。1〜2KB が現実的上限 |
| ブラウザ XSS で盗まれる | `httpOnly` Cookie 推奨。localStorage は危険 |
| アルゴリズム `none` 攻撃 | サーバー側で受け入れ alg を allowlist |
| 秘密鍵漏洩 | KMS / Secrets Manager で隔離、ローテーション |
| 時計ズレで `exp` 判定誤り | サーバ・クライアント NTP 同期、`leeway` 設定 |
| ペイロードは Base64 だけで暗号化されない | 秘密情報を入れない（JWE は別仕様） |

### 本PoC frontend の auth store 概要

```ts
state:    token: string | null, expiresAt: number | null
getters:  isAuthenticated  ← exp 判定はフロント側
actions:
  login(username, password)  → POST /auth/login → token保存
  logout()                    → state クリア + Preferences 削除
  loadFromStorage()           → 起動時に Preferences から復元
```

起動時 `loadFromStorage()` を await してから mount しないと、初回レンダーで未認証扱いになる点に注意。

### 本PoC の保存場所選定理由

| 候補 | 採用？ | 理由 |
|---|---|---|
| `localStorage` | △ | XSS で盗難されやすい |
| `sessionStorage` | × | タブを閉じると消える、PoC 用途に不向き |
| `Cookie (httpOnly)` | × | Capacitor Native では Cookie 取り扱いが煩雑、Cap環境制約大 |
| `Capacitor Preferences` | ◎ | クロスプラットフォーム抽象化、Web=localStorage、Android=SharedPreferences |
| `Capacitor SecureStorage`（任意） | ○ | 本番化で検討。Android Keystore で暗号化保存 |

PoC は Capacitor Preferences で割り切り、本番化で SecureStorage または OS keychain への移行を検討。

---

## 第2部: JWT 以外の認証方式の検討

### 全体比較表

| 方式 | 概要 | 長所 | 短所 | 本PoC適性 |
|---|---|---|---|---|
| **JWT (現状)** | 自己完結トークン | サーバ状態不要、マイクロサービス向き | 失効困難、payload 肥大化 | ◎ シンプル |
| **Session Cookie** | サーバ側 Session ID + Cookie | 失効容易、HTTPOnly で XSS 強耐性 | サーバ状態必要、CORS 設定面倒 | △ Cap 経由で Cookie が扱いにくい |
| **OAuth 2.0 / OIDC** | 外部 IdP に委譲 (Azure AD, Google等) | SSO、堅牢、規格統一 | 構築コスト高、外部依存 | △ 社内のみ運用には過剰 |
| **API Key** | 固定キー（ヘッダ送付） | 実装最小 | 失効困難、ユーザ識別不可 | × ユーザ別認証に不適 |
| **mTLS (相互TLS)** | クライアント証明書で認証 | 高セキュリティ、改ざん検知 | 証明書配布管理が大変 | × 業務デバイス単独配布なら検討余地 |
| **PASETO** | JWT の改良版（alg混乱なし） | より安全、シンプル | エコシステム小、Spring 連携弱 | △ JWT で十分 |
| **Opaque Token + Introspection** | ランダム文字列 + 都度サーバ問合せ | 失効容易、payload非露出 | 毎回 DB 問合せ、性能劣化 | △ 失効重視なら有力 |
| **Capacitor Biometric** | 指紋/顔で本端末ログイン保持 | UX良好、JWT を機内に閉じ込められる | プラグイン依存、Android API 23+ | ○ JWT と組合せで強化 |

### 各方式の詳細

#### Session Cookie

伝統的な方式。サーバ側で Session を保持し、Cookie で Session ID を送受信。

- 長所: ログアウト即時効力、`httpOnly`+`SameSite=Strict` で XSS/CSRF に強い
- 短所:
  - Capacitor Web では動作するが Native (Android WebView) では Cookie の扱いに `CookieManager` 設定が必要
  - サーバ水平スケール時に Session 共有ストア (Redis等) が必要
  - CORS の `Access-Control-Allow-Credentials: true` 設定が必要

PoC は CapacitorHttp (= Native fetch) 経由のため Cookie 扱いが面倒で見送り。

#### OAuth 2.0 / OIDC

外部 IdP（Azure AD / Google / Keycloak / 社内 SSO）に認証委譲。

- 長所: SSO、ユーザ管理・パスワードポリシーを IdP に集約、規格統一
- 短所:
  - クライアント (Capacitor) 側の実装に `@capacitor-community/generic-oauth2` 等が必要
  - リダイレクトベース認可の場合、Custom URL Scheme か Universal Link 設定が必要
  - 構築・運用コストが高い
- 適用: 既に社内 SSO 基盤があるか、複数アプリで認証統合したいケース

PoC スコープ外。本番化で社内 IdP 統合する場合は本格検討。

#### API Key

固定キー文字列を `X-API-Key` ヘッダ等で送るだけ。

- 長所: 実装最小、テスト用に便利
- 短所: ユーザ識別不可、失効はキー再発行のみ
- 適用: サービス間認証 (B2B API)、PoC でも内部用途なら可

ユーザー認証には不適。

#### mTLS (相互 TLS)

クライアント証明書でサーバが認証。

- 長所: ネットワーク層で認証完了、改ざん耐性高
- 短所: 端末ごとの証明書配布・管理が大変、失効には CRL/OCSP 仕組み要
- 適用: 業務デバイスへの証明書焼き込みが可能で、極めて堅牢な認証が必要なケース

本PoC のような汎用デバイスには重い。

#### PASETO

JWT の代替仕様。`alg: none` 攻撃や暗号化アルゴリズム混在を排除し、より安全に設計。

- 長所: JWT の既知脆弱性を回避
- 短所: エコシステム未成熟、Spring Boot 連携ライブラリが少ない
- 適用: セキュリティ要件が極めて厳しい新規システム

PoC スコープでは JWT の堅実な運用で十分。

#### Opaque Token + Introspection

JWT の代わりにランダム文字列を発行、検証時にサーバへ問合せ (RFC 7662)。

- 長所: 即時失効可、payload を URL に露出しない
- 短所: リクエスト毎にサーバ問合せが必要、性能要件次第でボトルネック
- 適用: 失効性重視 + サーバ性能に余裕

OAuth 2.0 の Resource Server で採用例多い。

#### Capacitor Biometric

指紋認証や顔認証で「端末ロック解除 = ログイン状態解錠」を実現。`@capacitor-community/biometric-auth` 等のプラグイン使用。

- 長所: UX 向上、JWT を端末内で安全に保持できる（再ログイン不要）
- 短所: プラグイン追加、Android API 23+ 限定、サーバとの認証フロー別途必要
- 適用: ログイン頻度を下げたい業務アプリ

JWT と組合せて使う「上に被せる」レイヤとして有効。

---

## 推奨パターン

### Pattern A（現状維持・PoC 推奨）

**JWT (HS256) + Capacitor Preferences**

- 社内イントラ、PoC 目的、単一バックエンド
- backend 実装時に Spring Security + jjwt で本物化
- 1時間程度の有効期限 + 再ログインで運用
- リフレッシュトークン未導入、ユーザは期限後に再ログイン

→ 現状の frontend 実装（auth store）はこの方針を前提に動作する。

### Pattern B（堅牢化）

**JWT (RS256) + Refresh Token + Biometric Lock**

- アクセストークン: 短命 (15分)
- リフレッシュトークン: 長命 (30日)、Capacitor Preferences に分離保存
- 起動時に指紋認証で Preferences 解錠（端末紛失対策）
- 本格運用に移行する際の選択肢
- backend が Refresh Token のローテーション/失効を管理

### Pattern C（既存社内基盤がある場合）

**SAML / OIDC で社内 IdP 連携**

- 既に Active Directory / Azure AD / 社内 SSO がある場合のみ意味あり
- Capacitor + OIDC は `@capacitor-community/generic-oauth2` 等で実装可
- 本PoCの調査範囲外だが、本番化で要検討

---

## 判断軸（次のステップを決めるための質問）

| 質問 | 回答による方向 |
|---|---|
| ユーザ管理は誰が？ | 自前 backend → JWT、社内 SSO → OIDC |
| 端末紛失時の即失効必要？ | Yes → Opaque Token / 短命JWT+Refresh、No → JWT 短命のみ |
| 業務デバイス自体に認証情報を保持？ | Yes → mTLS or Biometric Lock |
| 複数バックエンドサービスへの認証統一？ | Yes → JWT (RS256) or OIDC |
| 完全オフライン動作必要？ | JWT のみ（Opaque は不可） |
| ログイン頻度を下げたい？ | Refresh Token + Biometric |

---

## 本PoC で次に決めるべきこと

backend (`mobile-app-poc-backend`) の Spring Boot 実装段階で:

1. **アルゴリズム**: HS256（PoC 完結なら）or RS256（マイクロサービス想定なら）
2. **有効期限**: 1時間（短）/ 24時間（長）/ どちらか
3. **リフレッシュトークン**: 入れる/入れない
4. **失効方針**: ブラックリスト不要 / `jti` ベースで持つ
5. **JWT ライブラリ**: `io.jsonwebtoken:jjwt` (推奨) / `nimbus-jose-jwt`
6. **鍵保管**: 開発時 = `application.yml`、本番 = Vault / KMS

これらは backend のスペックで詳細化する。

## 参考資料

- RFC 7519 (JWT): https://datatracker.ietf.org/doc/html/rfc7519
- RFC 7662 (Token Introspection): https://datatracker.ietf.org/doc/html/rfc7662
- jjwt (JWT Java library): https://github.com/jwtk/jjwt
- Capacitor Preferences: https://capacitorjs.com/docs/apis/preferences
- @capacitor-community/biometric-auth: https://github.com/capacitor-community/biometric-auth
- OWASP JWT Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html
