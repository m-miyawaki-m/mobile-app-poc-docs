---
name: mobile-app-poc-backend 詳細設計
description: api-spec から openapi.yaml を取得し Schema-First で Spring Boot 3 / Java 21 / MyBatis / H2 / JWT / Swagger UI のバックエンド API を構築するリポジトリ
date: 2026-04-20
status: draft
parent: 2026-04-19-mobile-app-poc-master-design.md
---

# mobile-app-poc-backend 詳細設計

## 目的

`mobile-app-poc-api-spec` の `openapi.yaml` を入力として、Spring Boot 3.x + Java 21 によるバックエンド API を **Schema-First** で構築し、次の配線が Windows ネイティブで通ることを検証する。

- openapi-generator による Controller interface / DTO 自動生成
- MyBatis + H2 による DB 往復（`items` CRUD）
- Spring Security + JWT による認証フィルタ
- springdoc-openapi による Swagger UI
- `.\gradlew bootRun` 一発起動

本 PoC はビジネスロジックではなく **つなぎ目の検証** が主目的。ユーザー管理や本番運用機能は対象外。

## 位置づけ

```
mobile-app-poc-api-spec ──> mobile-app-poc-mock   （先行リポジトリ）
                         └─> mobile-app-poc-backend（本リポジトリ）──> （将来）frontend
```

backend は api-spec の**消費者**。frontend は当面 mock に接続し、将来 backend へ切替。api-spec 契約の破壊的変更は `implements XxxApi` のコンパイルエラーで検知する。

## 設計原則

- **api-spec を SSOT として尊重**: `src/main/resources/openapi/` は gitignore、backend 側で契約コピーを版管理しない
- **追加依存を最小化**: Spring Boot / MyBatis / H2 / jjwt / springdoc-openapi のみ
- **Windows ネイティブで完結**: WSL2 / Docker / グローバルツールは使わない
- **社内 LAN 内前提**: HTTPS 化、CI、外部公開は対象外
- **既存 SES 資産との親和性**: MyBatis + Oracle 志向、レイヤ別パッケージング、`.\gradlew` ベースの操作
- **Java 21 の記法を積極活用**: `record`、`var`、switch 式、pattern matching
- **Spring Boot 3 流儀**: `jakarta.*` パッケージ、`SecurityFilterChain` Bean ベースの Security 設定

## ディレクトリ構成

```
mobile-app-poc-backend/
├─ gradle/wrapper/
├─ gradlew / gradlew.bat
├─ settings.gradle
├─ build.gradle
├─ .gitignore
├─ .gitattributes
├─ README.md
├─ src/
│  ├─ main/
│  │  ├─ java/com/example/app/
│  │  │  ├─ Application.java
│  │  │  ├─ config/
│  │  │  │  ├─ CorsConfig.java
│  │  │  │  ├─ SecurityConfig.java
│  │  │  │  ├─ MyBatisConfig.java
│  │  │  │  └─ OpenApiConfig.java
│  │  │  ├─ controller/
│  │  │  │  ├─ AuthController.java        (implements AuthApi)
│  │  │  │  └─ ItemsController.java       (implements ItemsApi)
│  │  │  ├─ service/
│  │  │  │  ├─ AuthService.java
│  │  │  │  └─ ItemService.java
│  │  │  ├─ mapper/
│  │  │  │  └─ ItemMapper.java            (MyBatis interface)
│  │  │  ├─ domain/
│  │  │  │  └─ ItemEntity.java            (record)
│  │  │  ├─ security/
│  │  │  │  ├─ JwtAuthenticationFilter.java
│  │  │  │  └─ JwtTokenProvider.java
│  │  │  └─ exception/
│  │  │     ├─ GlobalExceptionHandler.java
│  │  │     └─ ItemNotFoundException.java
│  │  └─ resources/
│  │     ├─ application.yml
│  │     ├─ mapper/
│  │     │  └─ ItemMapper.xml
│  │     ├─ db/
│  │     │  ├─ schema.sql
│  │     │  └─ data.sql
│  │     └─ openapi/                      (gitignored, syncSpec が取得)
│  │        ├─ openapi.yaml
│  │        ├─ schemas/
│  │        ├─ responses/
│  │        ├─ parameters/
│  │        └─ examples/
│  └─ test/
│     └─ java/com/example/app/
│        └─ ApplicationSmokeTest.java
└─ build/generated/                       (openapi-generator 出力・gitignored)
   └─ src/main/java/...
      ├─ api/                             AuthApi.java, ItemsApi.java
      └─ model/                           Item.java, NewItem.java, ErrorResponse.java 等
```

`src/main/resources/openapi/` と `build/generated/` は gitignore。api-spec SSOT 尊重 & 毎ビルド再生成方針。

## Gradle / 依存

### プラグイン
```groovy
plugins {
  id 'org.springframework.boot' version '3.3.5'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'org.openapi.generator' version '7.8.0'
  id 'java'
}

group   = 'com.example'
version = '0.1.0'
java { toolchain.languageVersion = JavaLanguageVersion.of(21) }
```

### 依存
```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.boot:spring-boot-starter-validation'
  implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'
  implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
  runtimeOnly    'io.jsonwebtoken:jjwt-impl:0.12.6'
  runtimeOnly    'io.jsonwebtoken:jjwt-jackson:0.12.6'
  runtimeOnly    'com.h2database:h2'
  // 将来の Oracle 接続用（PoC では未使用、コメントアウト）:
  // runtimeOnly 'com.oracle.database.jdbc:ojdbc11'

  // openapi-generator 出力に必要:
  implementation 'io.swagger.core.v3:swagger-annotations:2.2.22'
  implementation 'jakarta.validation:jakarta.validation-api'
  implementation 'jakarta.annotation:jakarta.annotation-api'

  testImplementation 'org.springframework.boot:spring-boot-starter-test'
  testImplementation 'org.springframework.security:spring-security-test'
}
```

### openapi-generator 設定
```groovy
openApiGenerate {
  generatorName = 'spring'
  library       = 'spring-boot'
  inputSpec     = "$projectDir/src/main/resources/openapi/openapi.yaml"
  outputDir     = "$buildDir/generated"
  apiPackage    = 'com.example.app.generated.api'
  modelPackage  = 'com.example.app.generated.model'
  configOptions = [
    interfaceOnly        : 'true',
    useSpringBoot3       : 'true',
    useJakartaEe         : 'true',
    skipDefaultInterface : 'true',
    useTags              : 'true',
    dateLibrary          : 'java8',
    openApiNullable      : 'false'
  ]
}
sourceSets.main.java.srcDir "$buildDir/generated/src/main/java"
compileJava.dependsOn tasks.named('openApiGenerate')
```

### syncSpec タスク
```groovy
def specDir  = file("$projectDir/src/main/resources/openapi")
def specRepo = 'https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git'

task syncSpec {
  doLast {
    if (new File(specDir, '.git').exists()) {
      exec { commandLine 'git', '-C', specDir.absolutePath, 'pull', '--ff-only' }
    } else {
      specDir.parentFile.mkdirs()
      if (specDir.exists()) specDir.deleteDir()
      exec { commandLine 'git', 'clone', '--depth', '1', specRepo, specDir.absolutePath }
    }
  }
}
tasks.named('openApiGenerate').configure { dependsOn tasks.named('syncSpec') }
```

### 主要コマンド
```powershell
.\gradlew syncSpec            # api-spec を取得/更新
.\gradlew openApiGenerate     # Controller/DTO 生成（syncSpec 自動実行）
.\gradlew bootRun             # 起動（sync → generate → compile → run）
.\gradlew build               # ビルド
.\gradlew test                # テスト
```

### バージョン選定理由
- `Spring Boot` 3.3.5: LTS 的安定版の最新パッチ、Java 21 と MyBatis / openapi-generator との組合せ実績が豊富
- `openapi-generator` 7.8.0: Spring Boot 3.x / Jakarta EE サポート成熟
- `mybatis-spring-boot-starter` 3.0.3: Spring Boot 3.x 系列対応
- `springdoc-openapi` 2.6.0: Spring Boot 3.3 系列対応
- `jjwt` 0.12.x: API 安定化後の最新系列

## Schema-First 運用フロー

```
.\gradlew bootRun
  ↓
syncSpec           （git clone/pull で src/main/resources/openapi/ 更新）
  ↓
openApiGenerate    （openapi.yaml → build/generated/ に Api interface と DTO 生成）
  ↓
compileJava        （生成物 + 手書きコードをまとめてコンパイル）
  ↓
bootRun            （Spring Boot 起動）
```

**api-spec 更新時の流れ**:
1. api-spec リポジトリ側で `openapi.yaml` 更新 → `main` に merge
2. backend 側で `.\gradlew bootRun`
3. 破壊的変更があれば `implements AuthApi/ItemsApi` のコンパイルエラーで検知
4. エラーを手修正して再起動

**GitHub 到達不可環境**:
- `src/main/resources/openapi/` に api-spec の中身を手動配置（`.git` は不要）
- `syncSpec` を除外: `.\gradlew openApiGenerate -x syncSpec`
- mock と同じく「`spec/openapi.yaml` が存在すれば動く」状態を維持

**生成コードは触らない**: `build/generated/` は毎ビルド再生成。手修正・コピーは禁止。変更点はすべて `implements XxxApi` 側で吸収する。

## アーキテクチャ（レイヤ別パッケージング）

レイヤ別の理由:
- docs/6（原典）と整合
- 既存 SES 案件（Struts2 + MyBatis）からの移行者が迷わない
- openapi-generator の「1 interface = 1 Controller」と相性が良い

### 各パッケージの責務

| パッケージ | 役割 | 依存方向 |
|---|---|---|
| `controller/` | 生成された `XxxApi` を `implements`。DTO ↔ ドメインの変換、Service への委譲のみ | → service |
| `service/` | ビジネスロジック。例外 throw、トランザクション境界 | → mapper |
| `mapper/` | MyBatis interface。SQL は XML 側 | — |
| `domain/` | エンティティ（record）、値オブジェクト | — |
| `security/` | JWT フィルタ、トークン生成/検証 | — |
| `exception/` | 独自例外、`@RestControllerAdvice` | → generated.model.ErrorResponse |
| `config/` | Security / CORS / OpenAPI / MyBatis の `@Configuration` Bean | — |

### Controller 実装例

```java
@RestController
@RequiredArgsConstructor   // または手書きコンストラクタ
public class ItemsController implements ItemsApi {
  private final ItemService itemService;

  @Override
  public ResponseEntity<List<Item>> listItems() {
    var items = itemService.findAll().stream().map(this::toDto).toList();
    return ResponseEntity.ok(items);
  }

  private Item toDto(ItemEntity e) { /* ... */ }
}
```

- 生成 DTO: `com.example.app.generated.model.Item` 等
- ドメイン: `com.example.app.domain.ItemEntity`（record）
- 変換: Controller 内で手書き。PoC 規模ではマッパーライブラリ不要

## JWT 認証設計

### ユーザーストア
`application.yml` にハードコード:
```yaml
app:
  demo-user:
    username: demo-user
    password: demo-password
  jwt:
    secret: "poc-demo-secret-please-change-for-production-at-least-32-bytes"
    expiration-minutes: 60
```

### フロー
1. `POST /auth/login` → `AuthService` が username/password 検証（yml の値と一致するか）
2. 一致時: `JwtTokenProvider.create(username)` で HS256 JWT 発行、`LoginResponse { token, expiresIn }` 返却（`expiresIn` は有効期限までの秒数）
3. 以降のリクエストは `Authorization: Bearer <token>` 必須
4. `JwtAuthenticationFilter` (`OncePerRequestFilter`) が以下を実行:
   - ヘッダ抽出 → `JwtTokenProvider.verify()` → `Authentication` を `SecurityContextHolder` にセット
   - 検証失敗時は `SecurityContext` 空のまま次フィルタへ（`AuthenticationEntryPoint` が 401 を返す）

### Token 仕様
- アルゴリズム: HS256（jjwt）
- Claims: `sub = username`, `iat`, `exp` のみ（roles や権限は PoC 対象外）
- 有効期限: 60 分（yml 設定）
- リフレッシュトークン: **対象外**

### SecurityFilterChain
```java
http
  .csrf(AbstractHttpConfigurer::disable)
  .cors(Customizer.withDefaults())
  .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
  .authorizeHttpRequests(a -> a
    .requestMatchers("/auth/**", "/swagger-ui/**", "/swagger-ui.html",
                     "/v3/api-docs/**", "/h2-console/**").permitAll()
    .anyRequest().authenticated())
  .headers(h -> h.frameOptions(f -> f.sameOrigin()))   // H2 Console 用
  .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
```

## MyBatis / H2 / DDL

### DataSource
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:app;DB_CLOSE_DELAY=-1;MODE=Oracle
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  h2:
    console:
      enabled: true
      path: /h2-console
  sql:
    init:
      mode: always
      schema-locations: classpath:db/schema.sql
      data-locations:   classpath:db/data.sql
mybatis:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true
```

`MODE=Oracle` を付与し、Oracle 方言に近い振る舞いで動作させる（将来移行の摩擦低減）。

### schema.sql
```sql
DROP TABLE IF EXISTS items;
CREATE TABLE items (
  id         VARCHAR(36)   PRIMARY KEY,
  name       VARCHAR(100)  NOT NULL,
  quantity   INT           NOT NULL,
  created_at TIMESTAMP     NOT NULL,
  updated_at TIMESTAMP     NOT NULL
);
```

### data.sql
```sql
INSERT INTO items (id, name, quantity, created_at, updated_at) VALUES
  ('11111111-1111-1111-1111-111111111111', 'Sample Widget A', 10, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
  ('22222222-2222-2222-2222-222222222222', 'Sample Widget B',  3, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

### ItemEntity（record）
```java
public record ItemEntity(
  String id,
  String name,
  Integer quantity,
  OffsetDateTime createdAt,
  OffsetDateTime updatedAt) { }
```

### ItemMapper（interface）
```java
@Mapper
public interface ItemMapper {
  List<ItemEntity> findAll();
  ItemEntity findById(String id);
  int insert(ItemEntity entity);
  int update(ItemEntity entity);
  int deleteById(String id);
}
```

### ItemMapper.xml（抜粋）
```xml
<mapper namespace="com.example.app.mapper.ItemMapper">
  <select id="findAll" resultType="com.example.app.domain.ItemEntity">
    SELECT id, name, quantity, created_at, updated_at FROM items ORDER BY created_at
  </select>
  <!-- findById / insert / update / deleteById も同様 -->
</mapper>
```

### ID 採番
`ItemService.create()` 内で `UUID.randomUUID().toString()`。`createdAt` / `updatedAt` は `OffsetDateTime.now()`。

## Swagger UI / springdoc-openapi

### 配置
- UI: `http://localhost:8080/swagger-ui.html`
- JSON: `http://localhost:8080/v3/api-docs`
- Security 除外は上記フィルタ設定参照

### OpenApiConfig
```java
@Configuration
public class OpenApiConfig {
  @Bean
  public OpenAPI apiInfo() {
    return new OpenAPI()
      .info(new Info().title("mobile-app-poc backend").version("0.1.0"))
      .components(new Components().addSecuritySchemes("bearerAuth",
        new SecurityScheme().type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT")))
      .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
  }
}
```

Swagger UI 上で「Authorize」ボタンから JWT を貼り付け、保護エンドポイントを試せる。

### openapi.yaml との関係
- openapi-generator は Controller 雛形生成に使う
- Swagger UI に表示される契約は **Spring Boot ランタイムが実装から生成したもの**（springdoc）
- 原則として生成元の openapi.yaml と乖離しないが、乖離時は openapi.yaml を正とする

## エラーハンドリング

### 独自例外
```java
public class ItemNotFoundException extends RuntimeException { /* id を保持 */ }
```
Service 層でのみ throw。他の例外は Spring 標準／Security 標準。

### GlobalExceptionHandler（@RestControllerAdvice）

| 例外 | HTTP | code | message |
|---|---|---|---|
| `ItemNotFoundException` | 404 | `ITEM_NOT_FOUND` | `アイテムが見つかりません` |
| `MethodArgumentNotValidException` / `ConstraintViolationException` | 400 | `BAD_REQUEST` | 最初のバリデーション違反フィールド名+メッセージ |
| `AuthenticationException` / `BadCredentialsException` | 401 | `UNAUTHORIZED` | `認証に失敗しました` |
| `Exception`（フォールバック） | 500 | `INTERNAL_ERROR` | `サーバー内部エラー` |

ログインエンドポイントの資格情報違反は `AuthService` が `BadCredentialsException` を throw。JWT フィルタ側の検証失敗は `AuthenticationEntryPoint` 経由で 401。

### ErrorResponse
生成された `com.example.app.generated.model.ErrorResponse` を利用（`code` / `message` / `timestamp`）。独自クラス追加は不要。

## CORS

### CorsConfig
```java
@Configuration
public class CorsConfig {
  @Bean
  public CorsConfigurationSource corsConfigurationSource() {
    var cfg = new CorsConfiguration();
    cfg.setAllowedOrigins(List.of(
      "http://localhost:8100",    // ionic serve
      "capacitor://localhost",    // iOS
      "http://localhost"          // Android
    ));
    cfg.setAllowedMethods(List.of("GET","POST","PUT","DELETE","OPTIONS"));
    cfg.setAllowedHeaders(List.of("*"));
    cfg.setExposedHeaders(List.of("Authorization"));
    cfg.setAllowCredentials(true);
    cfg.setMaxAge(3600L);
    var src = new UrlBasedCorsConfigurationSource();
    src.registerCorsConfiguration("/**", cfg);
    return src;
  }
}
```

SecurityConfig 側で `.cors(Customizer.withDefaults())` を入れることで上記 Bean が適用される。

## application.yml（全体）

```yaml
server:
  port: 8080

spring:
  application:
    name: mobile-app-poc-backend
  datasource:
    url: jdbc:h2:mem:app;DB_CLOSE_DELAY=-1;MODE=Oracle
    driver-class-name: org.h2.Driver
    username: sa
    password: ""
  h2:
    console:
      enabled: true
      path: /h2-console
  sql:
    init:
      mode: always
      schema-locations: classpath:db/schema.sql
      data-locations:   classpath:db/data.sql

mybatis:
  mapper-locations: classpath:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl   # PoC 用、SQL を標準出力

springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs

logging:
  level:
    com.example.app: DEBUG
    org.springframework.security: DEBUG

app:
  demo-user:
    username: demo-user
    password: demo-password
  jwt:
    secret: "poc-demo-secret-please-change-for-production-at-least-32-bytes"
    expiration-minutes: 60

# --- Oracle 接続参考（コメントアウト、PoC では未使用） ---
# spring:
#   datasource:
#     url: jdbc:oracle:thin:@//host:1521/service
#     driver-class-name: oracle.jdbc.OracleDriver
#     username: appuser
#     password: xxxx
```

## テスト

### スコープ
- コンテキスト起動確認 1 件
- `/auth/login` → トークン取得 → `GET /items` が 200 を返す 1 件

### ApplicationSmokeTest
```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
class ApplicationSmokeTest {
  @Autowired TestRestTemplate rest;

  @Test void contextLoads() { /* 起動するだけ */ }

  @Test void loginAndListItems() {
    // 1) login
    var loginRes = rest.postForEntity("/auth/login",
        Map.of("username","demo-user","password","demo-password"), Map.class);
    assertThat(loginRes.getStatusCode().is2xxSuccessful()).isTrue();
    var token = (String) loginRes.getBody().get("token");

    // 2) GET /items with token
    var headers = new HttpHeaders();
    headers.setBearerAuth(token);
    var itemsRes = rest.exchange("/items", HttpMethod.GET,
        new HttpEntity<>(headers), List.class);
    assertThat(itemsRes.getStatusCode().is2xxSuccessful()).isTrue();
    assertThat(itemsRes.getBody()).hasSizeGreaterThanOrEqualTo(2);
  }
}
```

### 対象外
- `@MybatisTest` / `@WebMvcTest` のスライス検証
- セキュリティフィルタ単体の WireMock 検証
- 契約テスト（schemathesis 等）

## 動作確認（実装後の検証手順）

以下がすべて成功することを確認:

1. `git clone <backend repo> && cd mobile-app-poc-backend`
2. `.\gradlew build` → BUILD SUCCESSFUL
3. `.\gradlew bootRun` で `http://localhost:8080` 起動
4. ブラウザで `http://localhost:8080/swagger-ui.html` → エンドポイント一覧表示
5. ブラウザで `http://localhost:8080/h2-console` → JDBC URL `jdbc:h2:mem:app` で接続、`items` テーブル2件表示
6. `curl -X POST http://localhost:8080/auth/login -H "Content-Type: application/json" -d '{\"username\":\"demo-user\",\"password\":\"demo-password\"}'` → `{ "token": "...", "expiresIn": 3600 }`
7. `curl -H "Authorization: Bearer <TOKEN>" http://localhost:8080/items` → `data.sql` の2件が返る
8. `curl http://localhost:8080/items`（トークンなし）→ 401 Unauthorized + `{"code":"UNAUTHORIZED", ...}`
9. `curl -H "Authorization: Bearer <TOKEN>" http://localhost:8080/items/00000000-0000-0000-0000-000000000000` → 404 + `{"code":"ITEM_NOT_FOUND", ...}`
10. Swagger UI から Authorize → 保護エンドポイントが Try it out で叩ける

## 対象外（初期スコープから除外）

- Oracle 実接続（application.yml にコメントアウト雛形のみ）
- WebLogic への war デプロイ（`bootJar` のみ、`bootWar` なし）
- ユーザーテーブル（demo-user を yml にハードコード）
- リフレッシュトークン / `/auth/refresh`
- `@MybatisTest` / `@WebMvcTest` のスライス試験
- Actuator / Prometheus / メトリクス
- ログ集約、構造化ログ、ログローテーション（Spring Boot デフォルト Logback）
- HTTPS 化、証明書管理
- CI / GitHub Actions
- Docker 化
- パフォーマンステスト、負荷試験
- 外部公開（Tailscale Funnel / ngrok / クラウドデプロイ）
- 多言語メッセージ（エラーメッセージは日本語固定）
- DB マイグレーションツール（Flyway / Liquibase） — `schema.sql` / `data.sql` のみ
- `prism proxy` 的な外部API中継
- 契約テスト（schemathesis 等）

## .gitignore

```
build/
.gradle/
src/main/resources/openapi/
*.log
.idea/
.vscode/
*.iml
HELP.md
```

## .gitattributes

```
*.java   text eol=lf
*.xml    text eol=lf
*.yaml   text eol=lf
*.yml    text eol=lf
*.json   text eol=lf
*.md     text eol=lf
*.sql    text eol=lf
*.groovy text eol=lf
gradlew  text eol=lf
gradlew.bat text eol=crlf
```

## Git 運用

- scaffold 完了時に本リポジトリ内部で `git init` + 初期コミット（アシスタント実施）
- ローカル `git config user.name` / `user.email` 設定（本リポジトリのみ）
- GitHub リモート作成・push は api-spec / mock と同手順（`gh repo create ... --public --source=. --remote=origin --push`）
- `src/main/resources/openapi/` / `build/` / `.gradle/` / `node_modules`（本リポジトリでは使わないが念のため）は gitignore
- master ブランチではなく `main` を使用

## README 構成

1. 概要（Spring Boot 3 + Java 21、Schema-First、SSOT は api-spec 別リポ）
2. 前提（JDK 21、Gradle Wrapper 同梱、git コマンド、Windows ネイティブ、社内LAN）
3. セットアップ
   - JDK 21 確認: `java -version`
   - 方法A: そのまま `.\gradlew bootRun`（自動で syncSpec → generate → run）
   - 方法B: GitHub 到達不可時は `src/main/resources/openapi/` に api-spec を手動配置 → `.\gradlew openApiGenerate -x syncSpec` → `.\gradlew bootRun`
4. 主要コマンド（上記 `.\gradlew` 系）
5. 動作確認 curl 例（Swagger UI / H2 Console / login / items）
6. 起動後 URL 一覧
   - `http://localhost:8080/swagger-ui.html`
   - `http://localhost:8080/h2-console`
   - `http://localhost:8080/v3/api-docs`
7. トラブルシュート
   - PowerShell 実行ポリシー
   - ポート 8080 競合 / Hyper-V 予約範囲確認
   - `.\gradlew --refresh-dependencies`
   - `src/main/resources/openapi/` が空なら `syncSpec` 再実行
8. 他リポジトリとの関係（api-spec 消費者、mock と並列）
9. 今後の検討（WebLogic war 化、Oracle 実接続、リフレッシュトークン等）

## 次のステップ

本書承認後、**writing-plans スキルで実装計画作成**。想定段階:

1. Gradle プロジェクト初期化（wrapper、`build.gradle`、`settings.gradle`、`.gitignore`、`.gitattributes`）
2. `syncSpec` タスク + `openApiGenerate` 設定
3. `.\gradlew openApiGenerate` 動作確認（`build/generated/` に Api interface 出現）
4. パッケージスケルトン（`Application.java`、`config/*`、`controller/*`、`service/*`、`mapper/*`、`domain/*`、`security/*`、`exception/*`）
5. DDL / 初期データ / ItemMapper.xml
6. JWT 認証 + SecurityConfig + CorsConfig
7. Controller 実装（`implements AuthApi / ItemsApi`）
8. Swagger UI / OpenApiConfig
9. ApplicationSmokeTest
10. `.\gradlew bootRun` → 上記「動作確認」10項目の実機検証
11. git init + 初期コミット
12. GitHub リポジトリ作成 + push（ユーザ承認後）
