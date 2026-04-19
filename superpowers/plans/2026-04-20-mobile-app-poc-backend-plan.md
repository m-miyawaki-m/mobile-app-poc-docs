# mobile-app-poc-backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `mobile-app-poc-backend` リポジトリを scaffold し、Schema-First で Spring Boot 3.3 + Java 21 + MyBatis + H2 + JWT + Swagger UI のバックエンド API を構築、`.\gradlew bootRun` で `/auth/login` と `/items` CRUD が動作する状態まで到達させ、初期コミット + GitHub push まで完了させる。

**Architecture:** レイヤ別パッケージング (`controller / service / mapper / domain / security / exception / config`)。Gradle `syncSpec` タスクが `mobile-app-poc-api-spec` を `src/main/resources/openapi/` に git clone/pull し、`openApiGenerate` が `build/generated/` に `AuthApi` / `ItemsApi` / DTO を出力、手書きの `@RestController` 実装クラスがそれを `implements` する。認証は JWT (HS256, jjwt)、DB は H2 インメモリ + `schema.sql` / `data.sql` で初期化、MyBatis で `items` を CRUD。

**Tech Stack:** Java 21, Spring Boot 3.3.5, Gradle 8.x (Wrapper同梱), openapi-generator 7.8.0, mybatis-spring-boot-starter 3.0.3, H2 (runtime), jjwt 0.12.6, springdoc-openapi 2.6.0, JUnit 5 + Spring Boot Test

**Spec reference:** `docs/superpowers/specs/2026-04-20-mobile-app-poc-backend-design.md`

---

## File Structure

Working directory: `C:\Oracle\3df002\mobile-app\mobile-app-poc-backend\`

**Version-controlled files (this plan creates):**

```
mobile-app-poc-backend/
├─ gradle/wrapper/
│  ├─ gradle-wrapper.jar
│  └─ gradle-wrapper.properties
├─ gradlew
├─ gradlew.bat
├─ settings.gradle
├─ build.gradle
├─ .gitignore
├─ .gitattributes
├─ README.md
└─ src/
   ├─ main/
   │  ├─ java/com/example/app/
   │  │  ├─ Application.java
   │  │  ├─ config/
   │  │  │  ├─ CorsConfig.java
   │  │  │  ├─ SecurityConfig.java
   │  │  │  ├─ MyBatisConfig.java       (@MapperScan のみ)
   │  │  │  └─ OpenApiConfig.java
   │  │  ├─ controller/
   │  │  │  ├─ AuthController.java
   │  │  │  └─ ItemsController.java
   │  │  ├─ service/
   │  │  │  ├─ AuthService.java
   │  │  │  └─ ItemService.java
   │  │  ├─ mapper/
   │  │  │  └─ ItemMapper.java
   │  │  ├─ domain/
   │  │  │  └─ ItemEntity.java
   │  │  ├─ security/
   │  │  │  ├─ JwtAuthenticationFilter.java
   │  │  │  └─ JwtTokenProvider.java
   │  │  └─ exception/
   │  │     ├─ GlobalExceptionHandler.java
   │  │     └─ ItemNotFoundException.java
   │  └─ resources/
   │     ├─ application.yml
   │     ├─ mapper/
   │     │  └─ ItemMapper.xml
   │     └─ db/
   │        ├─ schema.sql
   │        └─ data.sql
   └─ test/
      └─ java/com/example/app/
         └─ ApplicationSmokeTest.java
```

**Generated/取得物 (gitignored, not checked in):**

```
mobile-app-poc-backend/
├─ build/                          # Gradle output
│  └─ generated/                   # openapi-generator 出力
├─ .gradle/                        # Gradle local cache
└─ src/main/resources/openapi/     # syncSpec 取得物
   ├─ openapi.yaml
   ├─ schemas/
   ├─ responses/
   ├─ parameters/
   └─ examples/
```

### 各ファイルの責務

- `build.gradle`: 依存、Java toolchain、openapi-generator 設定、`syncSpec` タスク
- `syncSpec` タスク: `https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git` を `src/main/resources/openapi/` に clone/pull
- `Application.java`: `@SpringBootApplication` エントリポイント
- `config/SecurityConfig.java`: `SecurityFilterChain` Bean、JWT フィルタの登録、認証除外パス定義
- `config/CorsConfig.java`: `CorsConfigurationSource` Bean（Capacitor 3オリジン許可）
- `config/MyBatisConfig.java`: `@MapperScan("com.example.app.mapper")`
- `config/OpenApiConfig.java`: springdoc に JWT security scheme を登録
- `security/JwtTokenProvider.java`: HS256 トークンの生成・検証
- `security/JwtAuthenticationFilter.java`: `OncePerRequestFilter` で `Authorization` ヘッダから JWT 取得 → `SecurityContext` セット
- `exception/ItemNotFoundException.java`: 独自 RuntimeException
- `exception/GlobalExceptionHandler.java`: `@RestControllerAdvice` で例外 → `ErrorResponse` マッピング
- `domain/ItemEntity.java`: DB 行を表す record
- `mapper/ItemMapper.java` + `mapper/ItemMapper.xml`: MyBatis CRUD
- `service/AuthService.java`: yml の demo-user と照合、JWT 発行
- `service/ItemService.java`: `ItemMapper` 呼出し、UUID 採番、`ItemNotFoundException` throw
- `controller/AuthController.java`: 生成 `AuthApi` を `implements`、`AuthService` に委譲
- `controller/ItemsController.java`: 生成 `ItemsApi` を `implements`、DTO ↔ Entity 変換、`ItemService` に委譲
- `test/ApplicationSmokeTest.java`: コンテキスト起動 + `/auth/login` → `/items` 疎通

## Testing Approach

**スモークテスト 2件のみ**（spec 準拠）:
1. `@SpringBootTest` でコンテキスト起動確認
2. `TestRestTemplate` で `/auth/login` → トークン取得 → `Authorization: Bearer` ヘッダで `GET /items` が 200

`@MybatisTest` や `@WebMvcTest` のスライス試験、契約テストは対象外。

実機動作確認（curl 10項目）は Task 17 で手動実施。

**Critical**:
- このプロジェクトに `git` リポジトリは Task 18 まで存在しない。途中で `git add` / `git commit` しないこと。
- `src/main/resources/openapi/` は Task 4 の `syncSpec` で取得物として出現する。手動でファイルを置かない。
- `build/generated/` 配下は openapi-generator が再生成する。手書き禁止。

## PowerShell / bash 互換

このプランは Windows ネイティブ前提だが、Claude Code は bash で動作。`.\gradlew` コマンドは bash でも `./gradlew` で実行可能（gradlew は bash スクリプト、gradlew.bat は cmd 用）。以下では bash 風に書く。

---

## Task 1: リポジトリディレクトリ作成と git 設定ファイル

**Files:**
- Create: `mobile-app-poc-backend/` (directory)
- Create: `mobile-app-poc-backend/.gitignore`
- Create: `mobile-app-poc-backend/.gitattributes`

- [ ] **Step 1: ディレクトリ作成**

```bash
mkdir -p "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
```

- [ ] **Step 2: `.gitignore` を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-backend/.gitignore`

```
# Build
build/
.gradle/

# Generated/fetched spec (api-spec is SSOT)
src/main/resources/openapi/

# IDE
.idea/
.vscode/
*.iml
*.ipr
*.iws

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Spring Boot
HELP.md
out/
```

- [ ] **Step 3: `.gitattributes` を作成**

Path: `C:/Oracle/3df002/mobile-app/mobile-app-poc-backend/.gitattributes`

```
*.java        text eol=lf
*.xml         text eol=lf
*.yaml        text eol=lf
*.yml         text eol=lf
*.json        text eol=lf
*.md          text eol=lf
*.sql         text eol=lf
*.groovy      text eol=lf
*.properties  text eol=lf
gradlew       text eol=lf
gradlew.bat   text eol=crlf
*.jar         binary
```

- [ ] **Step 4: 確認**

```bash
ls -la "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
```
Expected: `.gitignore` と `.gitattributes` が存在。

No commit yet (git init is Task 18).

---

## Task 2: Gradle Wrapper 配置

**Files:**
- Create: `gradle/wrapper/gradle-wrapper.jar`
- Create: `gradle/wrapper/gradle-wrapper.properties`
- Create: `gradlew`
- Create: `gradlew.bat`

Gradle Wrapper は手書きでなく `gradle` コマンドで生成する。ローカルに `gradle` コマンドがない場合は Gradle 8.10 を使うための `gradle-wrapper.properties` を手書き → `gradle-wrapper.jar` を既存リポジトリから流用、も可。ここでは正攻法として `gradle init` を使う。

- [ ] **Step 1: Gradle コマンドが使えるか確認**

```bash
gradle --version 2>&1 || echo "gradle not found, fall back to manual wrapper"
```

**Case A: `gradle` コマンドが使える場合**

- [ ] **Step 2A: `gradle wrapper` で Wrapper を生成**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
gradle wrapper --gradle-version 8.10 --distribution-type bin
```

Expected: `gradlew`, `gradlew.bat`, `gradle/wrapper/gradle-wrapper.jar`, `gradle/wrapper/gradle-wrapper.properties` が生成される。

**Case B: `gradle` コマンドがない場合**

- [ ] **Step 2B-1: `gradle-wrapper.properties` 手書き**

Path: `gradle/wrapper/gradle-wrapper.properties`

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.10-bin.zip
networkTimeout=10000
validateDistributionUrl=true
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

- [ ] **Step 2B-2: `gradle-wrapper.jar` をダウンロード**

```bash
curl -L -o gradle/wrapper/gradle-wrapper.jar \
  "https://raw.githubusercontent.com/gradle/gradle/v8.10.0/gradle/wrapper/gradle-wrapper.jar"
```

- [ ] **Step 2B-3: `gradlew` 手書き**

`gradlew` スクリプトは https://raw.githubusercontent.com/gradle/gradle/v8.10.0/gradlew から取得:

```bash
curl -L -o gradlew "https://raw.githubusercontent.com/gradle/gradle/v8.10.0/gradlew"
curl -L -o gradlew.bat "https://raw.githubusercontent.com/gradle/gradle/v8.10.0/gradlew.bat"
chmod +x gradlew
```

- [ ] **Step 3: Wrapper 動作確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
./gradlew --version
```

Expected: `Gradle 8.10` と表示。Java 21 で動作することも確認。

---

## Task 3: settings.gradle と build.gradle（最小スケルトン）

**Files:**
- Create: `settings.gradle`
- Create: `build.gradle`

- [ ] **Step 1: `settings.gradle` を作成**

Path: `settings.gradle`

```groovy
rootProject.name = 'mobile-app-poc-backend'
```

- [ ] **Step 2: `build.gradle` を作成（最小スケルトン、openapi-generator 前）**

Path: `build.gradle`

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.5'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'java'
}

group = 'com.example'
version = '0.1.0'

java {
  toolchain {
    languageVersion = JavaLanguageVersion.of(21)
  }
}

repositories {
  mavenCentral()
}

dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

tasks.named('test') {
  useJUnitPlatform()
}
```

- [ ] **Step 3: Application.java を作成**

Path: `src/main/java/com/example/app/Application.java`

```java
package com.example.app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
  }
}
```

- [ ] **Step 4: 最小起動確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
./gradlew build
```

Expected: `BUILD SUCCESSFUL`。テストなしで jar ビルド成功。

```bash
./gradlew bootRun
```

別ターミナル（または Ctrl+C 待ち受け中）で:

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/
```

Expected: `401` または `404`（Security 未設定 = 全許可なら 404、デフォルトで認証あり = 401）。サーバーが反応すれば OK。Ctrl+C で停止。

---

## Task 4: syncSpec タスク追加と実行確認

**Files:**
- Modify: `build.gradle`

- [ ] **Step 1: `build.gradle` に `syncSpec` タスクを追加**

`build.gradle` 末尾に追記:

```groovy
// ------ Schema-First: api-spec 同期 ------
def specDir  = file("$projectDir/src/main/resources/openapi")
def specRepo = 'https://github.com/m-miyawaki-m/mobile-app-poc-api-spec.git'

task syncSpec {
  group = 'openapi'
  description = 'api-spec リポジトリを取得または更新する'
  doLast {
    if (new File(specDir, '.git').exists()) {
      println "Pulling api-spec into ${specDir}"
      exec {
        commandLine 'git', '-C', specDir.absolutePath, 'pull', '--ff-only'
      }
    } else {
      println "Cloning api-spec into ${specDir}"
      if (specDir.exists()) specDir.deleteDir()
      specDir.parentFile.mkdirs()
      exec {
        commandLine 'git', 'clone', '--depth', '1', specRepo, specDir.absolutePath
      }
    }
  }
}
```

- [ ] **Step 2: syncSpec 実行確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
./gradlew syncSpec
```

Expected: `BUILD SUCCESSFUL`、`src/main/resources/openapi/` に `openapi.yaml`, `schemas/`, `responses/`, `parameters/`, `examples/` が出現。

```bash
ls src/main/resources/openapi/
```

Expected: `openapi.yaml` を含む api-spec の内容。

- [ ] **Step 3: 再実行で pull になることを確認**

```bash
./gradlew syncSpec
```

Expected: `Pulling api-spec into ...` のログ、エラーなく完了。

---

## Task 5: openapi-generator 設定と生成確認

**Files:**
- Modify: `build.gradle`

- [ ] **Step 1: `build.gradle` の `plugins` ブロックに openapi-generator を追加**

```groovy
plugins {
  id 'org.springframework.boot' version '3.3.5'
  id 'io.spring.dependency-management' version '1.1.6'
  id 'org.openapi.generator' version '7.8.0'
  id 'java'
}
```

- [ ] **Step 2: 依存を拡充**

`dependencies` ブロックを以下に置換:

```groovy
dependencies {
  // Spring Boot
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.boot:spring-boot-starter-validation'

  // MyBatis + H2
  implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
  runtimeOnly    'com.h2database:h2'
  // 将来の Oracle 接続（PoC では未使用）:
  // runtimeOnly 'com.oracle.database.jdbc:ojdbc11'

  // springdoc-openapi (Swagger UI)
  implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0'

  // JWT
  implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
  runtimeOnly    'io.jsonwebtoken:jjwt-impl:0.12.6'
  runtimeOnly    'io.jsonwebtoken:jjwt-jackson:0.12.6'

  // openapi-generator 出力の依存
  implementation 'io.swagger.core.v3:swagger-annotations:2.2.22'
  implementation 'jakarta.validation:jakarta.validation-api'
  implementation 'jakarta.annotation:jakarta.annotation-api'

  // Test
  testImplementation 'org.springframework.boot:spring-boot-starter-test'
  testImplementation 'org.springframework.security:spring-security-test'
}
```

- [ ] **Step 3: `openApiGenerate` 設定を `build.gradle` 末尾に追記**

```groovy
// ------ openapi-generator 設定 ------
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
tasks.named('openApiGenerate').configure { dependsOn tasks.named('syncSpec') }
tasks.named('compileJava').configure { dependsOn tasks.named('openApiGenerate') }
```

- [ ] **Step 4: 生成確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
./gradlew openApiGenerate
```

Expected: `BUILD SUCCESSFUL`。

```bash
ls build/generated/src/main/java/com/example/app/generated/api/
ls build/generated/src/main/java/com/example/app/generated/model/
```

Expected:
- `api/` に `AuthApi.java`, `ItemsApi.java`
- `model/` に `Item.java`, `NewItem.java`, `UpdateItem.java`, `LoginRequest.java`, `LoginResponse.java`, `Error.java`

- [ ] **Step 5: ビルドまで通ることを確認**

```bash
./gradlew build -x test
```

Expected: `BUILD SUCCESSFUL`（コンパイルが通る、テストは後回し）。

---

## Task 6: application.yml

**Files:**
- Create: `src/main/resources/application.yml`

- [ ] **Step 1: `application.yml` を作成**

Path: `src/main/resources/application.yml`

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
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl

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

- [ ] **Step 2: 設定 + DB未定義でも起動できることを軽く確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。`bootRun` は schema.sql / data.sql がないと失敗するので Task 7 まで保留。

---

## Task 7: H2 スキーマと初期データ

**Files:**
- Create: `src/main/resources/db/schema.sql`
- Create: `src/main/resources/db/data.sql`

- [ ] **Step 1: `schema.sql` を作成**

Path: `src/main/resources/db/schema.sql`

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

- [ ] **Step 2: `data.sql` を作成**

Path: `src/main/resources/db/data.sql`

```sql
INSERT INTO items (id, name, quantity, created_at, updated_at) VALUES
  ('11111111-1111-1111-1111-111111111111', 'Sample Widget A', 10, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP),
  ('22222222-2222-2222-2222-222222222222', 'Sample Widget B',  3, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);
```

- [ ] **Step 3: bootRun で起動確認（Security / Controller 未実装でもコンテキストは起動する想定）**

```bash
./gradlew bootRun
```

別ターミナル:
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/h2-console
```

Expected: `200` または `302`（ログイン画面）。H2 Console に到達できれば OK。

Security は spring-boot-starter-security 依存で自動 enable されるため、デフォルトで Basic 認証要求になる。ログには起動時のパスワードが出力される。H2 Console へのアクセスは 302 リダイレクトで ログインページ。

Ctrl+C で停止。

---

## Task 8: ItemEntity + ItemMapper + ItemMapper.xml

**Files:**
- Create: `src/main/java/com/example/app/domain/ItemEntity.java`
- Create: `src/main/java/com/example/app/mapper/ItemMapper.java`
- Create: `src/main/java/com/example/app/config/MyBatisConfig.java`
- Create: `src/main/resources/mapper/ItemMapper.xml`

- [ ] **Step 1: `ItemEntity.java` を作成（record）**

Path: `src/main/java/com/example/app/domain/ItemEntity.java`

```java
package com.example.app.domain;

import java.time.OffsetDateTime;

public record ItemEntity(
    String id,
    String name,
    Integer quantity,
    OffsetDateTime createdAt,
    OffsetDateTime updatedAt
) { }
```

- [ ] **Step 2: `ItemMapper.java` を作成**

Path: `src/main/java/com/example/app/mapper/ItemMapper.java`

```java
package com.example.app.mapper;

import com.example.app.domain.ItemEntity;
import org.apache.ibatis.annotations.Mapper;

import java.util.List;

@Mapper
public interface ItemMapper {
  List<ItemEntity> findAll();
  ItemEntity findById(String id);
  int insert(ItemEntity entity);
  int update(ItemEntity entity);
  int deleteById(String id);
}
```

- [ ] **Step 3: `MyBatisConfig.java` を作成**

Path: `src/main/java/com/example/app/config/MyBatisConfig.java`

```java
package com.example.app.config;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@MapperScan("com.example.app.mapper")
public class MyBatisConfig {
}
```

- [ ] **Step 4: `ItemMapper.xml` を作成**

Path: `src/main/resources/mapper/ItemMapper.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.app.mapper.ItemMapper">

  <resultMap id="itemResult" type="com.example.app.domain.ItemEntity">
    <constructor>
      <idArg  column="id"         javaType="java.lang.String"/>
      <arg    column="name"       javaType="java.lang.String"/>
      <arg    column="quantity"   javaType="java.lang.Integer"/>
      <arg    column="created_at" javaType="java.time.OffsetDateTime"/>
      <arg    column="updated_at" javaType="java.time.OffsetDateTime"/>
    </constructor>
  </resultMap>

  <select id="findAll" resultMap="itemResult">
    SELECT id, name, quantity, created_at, updated_at
    FROM items
    ORDER BY created_at
  </select>

  <select id="findById" parameterType="string" resultMap="itemResult">
    SELECT id, name, quantity, created_at, updated_at
    FROM items
    WHERE id = #{id}
  </select>

  <insert id="insert" parameterType="com.example.app.domain.ItemEntity">
    INSERT INTO items (id, name, quantity, created_at, updated_at)
    VALUES (#{id}, #{name}, #{quantity}, #{createdAt}, #{updatedAt})
  </insert>

  <update id="update" parameterType="com.example.app.domain.ItemEntity">
    UPDATE items
       SET name = #{name},
           quantity = #{quantity},
           updated_at = #{updatedAt}
     WHERE id = #{id}
  </update>

  <delete id="deleteById" parameterType="string">
    DELETE FROM items WHERE id = #{id}
  </delete>

</mapper>
```

- [ ] **Step 5: コンパイル確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 9: ItemNotFoundException + ItemService

**Files:**
- Create: `src/main/java/com/example/app/exception/ItemNotFoundException.java`
- Create: `src/main/java/com/example/app/service/ItemService.java`

- [ ] **Step 1: `ItemNotFoundException.java` を作成**

Path: `src/main/java/com/example/app/exception/ItemNotFoundException.java`

```java
package com.example.app.exception;

public class ItemNotFoundException extends RuntimeException {
  private final String itemId;

  public ItemNotFoundException(String itemId) {
    super("Item not found: " + itemId);
    this.itemId = itemId;
  }

  public String getItemId() {
    return itemId;
  }
}
```

- [ ] **Step 2: `ItemService.java` を作成**

Path: `src/main/java/com/example/app/service/ItemService.java`

```java
package com.example.app.service;

import com.example.app.domain.ItemEntity;
import com.example.app.exception.ItemNotFoundException;
import com.example.app.mapper.ItemMapper;
import org.springframework.stereotype.Service;

import java.time.OffsetDateTime;
import java.util.List;
import java.util.UUID;

@Service
public class ItemService {
  private final ItemMapper itemMapper;

  public ItemService(ItemMapper itemMapper) {
    this.itemMapper = itemMapper;
  }

  public List<ItemEntity> findAll() {
    return itemMapper.findAll();
  }

  public ItemEntity findById(String id) {
    var entity = itemMapper.findById(id);
    if (entity == null) {
      throw new ItemNotFoundException(id);
    }
    return entity;
  }

  public ItemEntity create(String name, Integer quantity) {
    var now = OffsetDateTime.now();
    var entity = new ItemEntity(UUID.randomUUID().toString(), name, quantity, now, now);
    itemMapper.insert(entity);
    return entity;
  }

  public ItemEntity update(String id, String name, Integer quantity) {
    var existing = findById(id);  // throws ItemNotFoundException if missing
    var updated = new ItemEntity(
        existing.id(), name, quantity,
        existing.createdAt(), OffsetDateTime.now());
    itemMapper.update(updated);
    return updated;
  }

  public void delete(String id) {
    findById(id);  // throws ItemNotFoundException if missing
    itemMapper.deleteById(id);
  }
}
```

- [ ] **Step 3: コンパイル確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 10: GlobalExceptionHandler

**Files:**
- Create: `src/main/java/com/example/app/exception/GlobalExceptionHandler.java`

openapi-generator が出力する `com.example.app.generated.model.Error` クラス（OpenAPI `Error` スキーマに由来）を使う。フィールド名は `code` / `message` / `timestamp`。

- [ ] **Step 1: 生成された Error クラスのシグネチャを確認**

```bash
cat build/generated/src/main/java/com/example/app/generated/model/Error.java | head -50
```

Expected: コンストラクタまたはセッターで `code`, `message`, `timestamp` を設定できることを確認。

- [ ] **Step 2: `GlobalExceptionHandler.java` を作成**

Path: `src/main/java/com/example/app/exception/GlobalExceptionHandler.java`

```java
package com.example.app.exception;

import com.example.app.generated.model.Error;
import jakarta.validation.ConstraintViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.core.AuthenticationException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.OffsetDateTime;

@RestControllerAdvice
public class GlobalExceptionHandler {

  @ExceptionHandler(ItemNotFoundException.class)
  public ResponseEntity<Error> handleNotFound(ItemNotFoundException ex) {
    return build(HttpStatus.NOT_FOUND, "ITEM_NOT_FOUND", "アイテムが見つかりません");
  }

  @ExceptionHandler(MethodArgumentNotValidException.class)
  public ResponseEntity<Error> handleValidation(MethodArgumentNotValidException ex) {
    var msg = ex.getBindingResult().getFieldErrors().stream()
        .findFirst()
        .map(f -> f.getField() + ": " + f.getDefaultMessage())
        .orElse("リクエストが不正です");
    return build(HttpStatus.BAD_REQUEST, "BAD_REQUEST", msg);
  }

  @ExceptionHandler(ConstraintViolationException.class)
  public ResponseEntity<Error> handleConstraint(ConstraintViolationException ex) {
    return build(HttpStatus.BAD_REQUEST, "BAD_REQUEST", ex.getMessage());
  }

  @ExceptionHandler({BadCredentialsException.class, AuthenticationException.class})
  public ResponseEntity<Error> handleAuth(AuthenticationException ex) {
    return build(HttpStatus.UNAUTHORIZED, "UNAUTHORIZED", "認証に失敗しました");
  }

  @ExceptionHandler(Exception.class)
  public ResponseEntity<Error> handleOther(Exception ex) {
    return build(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", "サーバー内部エラー");
  }

  private ResponseEntity<Error> build(HttpStatus status, String code, String message) {
    var err = new Error();
    err.setCode(code);
    err.setMessage(message);
    err.setTimestamp(OffsetDateTime.now());
    return ResponseEntity.status(status).body(err);
  }
}
```

NOTE: `Error` クラスの setter 名が `setCode` / `setMessage` / `setTimestamp` であることを Step 1 で確認済み前提。openapi-generator 7.8 デフォルトで JavaBean 形式の setter を出力する。コンストラクタで渡す形式の場合は `new Error(code, message, OffsetDateTime.now())` に変更。

- [ ] **Step 3: コンパイル確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 11: JwtTokenProvider

**Files:**
- Create: `src/main/java/com/example/app/security/JwtTokenProvider.java`

- [ ] **Step 1: `JwtTokenProvider.java` を作成**

Path: `src/main/java/com/example/app/security/JwtTokenProvider.java`

```java
package com.example.app.security;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Date;

@Component
public class JwtTokenProvider {

  private final SecretKey key;
  private final long expirationMinutes;

  public JwtTokenProvider(
      @Value("${app.jwt.secret}") String secret,
      @Value("${app.jwt.expiration-minutes}") long expirationMinutes) {
    this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    this.expirationMinutes = expirationMinutes;
  }

  public String create(String username) {
    var now = Instant.now();
    var exp = now.plusSeconds(expirationMinutes * 60);
    return Jwts.builder()
        .subject(username)
        .issuedAt(Date.from(now))
        .expiration(Date.from(exp))
        .signWith(key, Jwts.SIG.HS256)
        .compact();
  }

  public String verifyAndGetSubject(String token) {
    return Jwts.parser()
        .verifyWith(key)
        .build()
        .parseSignedClaims(token)
        .getPayload()
        .getSubject();
  }

  public long getExpirationSeconds() {
    return expirationMinutes * 60;
  }
}
```

- [ ] **Step 2: コンパイル確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 12: JwtAuthenticationFilter

**Files:**
- Create: `src/main/java/com/example/app/security/JwtAuthenticationFilter.java`

- [ ] **Step 1: `JwtAuthenticationFilter.java` を作成**

Path: `src/main/java/com/example/app/security/JwtAuthenticationFilter.java`

```java
package com.example.app.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.List;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

  private static final String BEARER_PREFIX = "Bearer ";
  private final JwtTokenProvider tokenProvider;

  public JwtAuthenticationFilter(JwtTokenProvider tokenProvider) {
    this.tokenProvider = tokenProvider;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request,
                                  HttpServletResponse response,
                                  FilterChain chain) throws ServletException, IOException {
    var header = request.getHeader("Authorization");
    if (header != null && header.startsWith(BEARER_PREFIX)) {
      var token = header.substring(BEARER_PREFIX.length());
      try {
        var username = tokenProvider.verifyAndGetSubject(token);
        var auth = new UsernamePasswordAuthenticationToken(username, null, List.of());
        SecurityContextHolder.getContext().setAuthentication(auth);
      } catch (Exception e) {
        // 検証失敗: SecurityContext は空のまま次フィルタへ（AuthenticationEntryPoint が 401）
        SecurityContextHolder.clearContext();
      }
    }
    chain.doFilter(request, response);
  }
}
```

- [ ] **Step 2: コンパイル確認**

```bash
./gradlew compileJava
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 13: CorsConfig + SecurityConfig + OpenApiConfig

**Files:**
- Create: `src/main/java/com/example/app/config/CorsConfig.java`
- Create: `src/main/java/com/example/app/config/SecurityConfig.java`
- Create: `src/main/java/com/example/app/config/OpenApiConfig.java`

- [ ] **Step 1: `CorsConfig.java` を作成**

Path: `src/main/java/com/example/app/config/CorsConfig.java`

```java
package com.example.app.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

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
    cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
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

- [ ] **Step 2: `SecurityConfig.java` を作成**

Path: `src/main/java/com/example/app/config/SecurityConfig.java`

```java
package com.example.app.config;

import com.example.app.security.JwtAuthenticationFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

  private final JwtAuthenticationFilter jwtFilter;

  public SecurityConfig(JwtAuthenticationFilter jwtFilter) {
    this.jwtFilter = jwtFilter;
  }

  @Bean
  public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .cors(Customizer.withDefaults())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(a -> a
            .requestMatchers(
                "/auth/**",
                "/swagger-ui/**",
                "/swagger-ui.html",
                "/v3/api-docs/**",
                "/h2-console/**"
            ).permitAll()
            .anyRequest().authenticated())
        .headers(h -> h.frameOptions(f -> f.sameOrigin()))  // H2 Console 用
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
  }
}
```

- [ ] **Step 3: `OpenApiConfig.java` を作成**

Path: `src/main/java/com/example/app/config/OpenApiConfig.java`

```java
package com.example.app.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

  @Bean
  public OpenAPI apiInfo() {
    return new OpenAPI()
        .info(new Info().title("mobile-app-poc backend").version("0.1.0"))
        .components(new Components().addSecuritySchemes("bearerAuth",
            new SecurityScheme()
                .type(SecurityScheme.Type.HTTP)
                .scheme("bearer")
                .bearerFormat("JWT")))
        .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
  }
}
```

- [ ] **Step 4: ビルド確認**

```bash
./gradlew build -x test
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 14: AuthService + AuthController

**Files:**
- Create: `src/main/java/com/example/app/service/AuthService.java`
- Create: `src/main/java/com/example/app/controller/AuthController.java`

- [ ] **Step 1: 生成された `AuthApi` のシグネチャを確認**

```bash
cat build/generated/src/main/java/com/example/app/generated/api/AuthApi.java | grep -A 20 "login"
```

Expected: `ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest loginRequest)` のようなシグネチャ。

- [ ] **Step 2: 生成された `LoginRequest` / `LoginResponse` のフィールドとコンストラクタ/setter を確認**

```bash
cat build/generated/src/main/java/com/example/app/generated/model/LoginRequest.java | head -40
cat build/generated/src/main/java/com/example/app/generated/model/LoginResponse.java | head -40
```

Expected: `LoginRequest` に `getUsername()`, `getPassword()`。`LoginResponse` に `setToken(String)`, `setExpiresIn(Integer)` またはコンストラクタ引数。

- [ ] **Step 3: `AuthService.java` を作成**

Path: `src/main/java/com/example/app/service/AuthService.java`

```java
package com.example.app.service;

import com.example.app.security.JwtTokenProvider;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.stereotype.Service;

@Service
public class AuthService {
  private final String configuredUsername;
  private final String configuredPassword;
  private final JwtTokenProvider tokenProvider;

  public AuthService(
      @Value("${app.demo-user.username}") String configuredUsername,
      @Value("${app.demo-user.password}") String configuredPassword,
      JwtTokenProvider tokenProvider) {
    this.configuredUsername = configuredUsername;
    this.configuredPassword = configuredPassword;
    this.tokenProvider = tokenProvider;
  }

  public record TokenResult(String token, long expiresInSeconds) { }

  public TokenResult authenticate(String username, String password) {
    if (!configuredUsername.equals(username) || !configuredPassword.equals(password)) {
      throw new BadCredentialsException("invalid credentials");
    }
    var token = tokenProvider.create(username);
    return new TokenResult(token, tokenProvider.getExpirationSeconds());
  }
}
```

- [ ] **Step 4: `AuthController.java` を作成**

Path: `src/main/java/com/example/app/controller/AuthController.java`

```java
package com.example.app.controller;

import com.example.app.generated.api.AuthApi;
import com.example.app.generated.model.LoginRequest;
import com.example.app.generated.model.LoginResponse;
import com.example.app.service.AuthService;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class AuthController implements AuthApi {

  private final AuthService authService;

  public AuthController(AuthService authService) {
    this.authService = authService;
  }

  @Override
  public ResponseEntity<LoginResponse> login(LoginRequest request) {
    var result = authService.authenticate(request.getUsername(), request.getPassword());
    var body = new LoginResponse();
    body.setToken(result.token());
    body.setExpiresIn((int) result.expiresInSeconds());
    return ResponseEntity.ok(body);
  }
}
```

NOTE: `LoginResponse` の setter 名が異なる場合（例: コンストラクタ必須）は Step 2 の確認結果に従って調整。

- [ ] **Step 5: ビルド確認**

```bash
./gradlew build -x test
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 15: ItemsController

**Files:**
- Create: `src/main/java/com/example/app/controller/ItemsController.java`

- [ ] **Step 1: 生成された `ItemsApi` と `Item` / `NewItem` / `UpdateItem` を確認**

```bash
cat build/generated/src/main/java/com/example/app/generated/api/ItemsApi.java | grep -E "(List|Item |UUID|String) " | head -20
cat build/generated/src/main/java/com/example/app/generated/model/Item.java | head -60
```

Expected:
- `ItemsApi` の各メソッド: `listItems()`, `createItem(NewItem)`, `getItem(UUID id)`, `updateItem(UUID id, UpdateItem)`, `deleteItem(UUID id)`
- パラメータ `id` の型は `java.util.UUID`（`format: uuid` から生成される標準挙動）
- `Item` は `id` が `UUID`、`createdAt`/`updatedAt` が `OffsetDateTime`、`name` `String`、`quantity` `Integer`

- [ ] **Step 2: `ItemsController.java` を作成**

Path: `src/main/java/com/example/app/controller/ItemsController.java`

```java
package com.example.app.controller;

import com.example.app.domain.ItemEntity;
import com.example.app.generated.api.ItemsApi;
import com.example.app.generated.model.Item;
import com.example.app.generated.model.NewItem;
import com.example.app.generated.model.UpdateItem;
import com.example.app.service.ItemService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.UUID;

@RestController
public class ItemsController implements ItemsApi {

  private final ItemService itemService;

  public ItemsController(ItemService itemService) {
    this.itemService = itemService;
  }

  @Override
  public ResponseEntity<List<Item>> listItems() {
    var items = itemService.findAll().stream().map(this::toDto).toList();
    return ResponseEntity.ok(items);
  }

  @Override
  public ResponseEntity<Item> createItem(NewItem newItem) {
    var created = itemService.create(newItem.getName(), newItem.getQuantity());
    return ResponseEntity.status(HttpStatus.CREATED).body(toDto(created));
  }

  @Override
  public ResponseEntity<Item> getItem(UUID id) {
    var entity = itemService.findById(id.toString());
    return ResponseEntity.ok(toDto(entity));
  }

  @Override
  public ResponseEntity<Item> updateItem(UUID id, UpdateItem update) {
    var updated = itemService.update(id.toString(), update.getName(), update.getQuantity());
    return ResponseEntity.ok(toDto(updated));
  }

  @Override
  public ResponseEntity<Void> deleteItem(UUID id) {
    itemService.delete(id.toString());
    return ResponseEntity.noContent().build();
  }

  private Item toDto(ItemEntity e) {
    var dto = new Item();
    dto.setId(UUID.fromString(e.id()));
    dto.setName(e.name());
    dto.setQuantity(e.quantity());
    dto.setCreatedAt(e.createdAt());
    dto.setUpdatedAt(e.updatedAt());
    return dto;
  }
}
```

NOTE: `Item` / `NewItem` / `UpdateItem` の setter / getter は Step 1 の確認結果に合わせて調整。openapi-generator 7.8 デフォルトで JavaBean 形式の setter を出力する。

- [ ] **Step 3: ビルド確認**

```bash
./gradlew build -x test
```

Expected: `BUILD SUCCESSFUL`。

---

## Task 16: ApplicationSmokeTest

**Files:**
- Create: `src/test/java/com/example/app/ApplicationSmokeTest.java`

- [ ] **Step 1: `ApplicationSmokeTest.java` を作成**

Path: `src/test/java/com/example/app/ApplicationSmokeTest.java`

```java
package com.example.app;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.context.SpringBootTest.WebEnvironment.RANDOM_PORT;

@SpringBootTest(webEnvironment = RANDOM_PORT)
class ApplicationSmokeTest {

  @Autowired
  TestRestTemplate rest;

  @Test
  void contextLoads() {
    // Spring Context が起動するだけで通過
  }

  @Test
  @SuppressWarnings({"unchecked", "rawtypes"})
  void loginAndListItems() {
    // 1) login
    var body = Map.of("username", "demo-user", "password", "demo-password");
    ResponseEntity<Map> loginRes = rest.postForEntity("/auth/login", body, Map.class);
    assertThat(loginRes.getStatusCode().is2xxSuccessful()).isTrue();
    var token = (String) loginRes.getBody().get("token");
    assertThat(token).isNotBlank();

    // 2) GET /items with token
    var headers = new HttpHeaders();
    headers.setBearerAuth(token);
    ResponseEntity<List> itemsRes = rest.exchange("/items", HttpMethod.GET,
        new HttpEntity<>(headers), List.class);
    assertThat(itemsRes.getStatusCode().is2xxSuccessful()).isTrue();
    assertThat(itemsRes.getBody()).hasSizeGreaterThanOrEqualTo(2);
  }
}
```

- [ ] **Step 2: テスト実行**

```bash
./gradlew test
```

Expected: `BUILD SUCCESSFUL`、`ApplicationSmokeTest` 2 件 PASS。

テストレポートは `build/reports/tests/test/index.html`。失敗時はそこを確認。

---

## Task 17: 実機動作確認（手動 curl 検証）

**No files created. Manual verification using bootRun.**

- [ ] **Step 1: `bootRun` 起動**

```bash
./gradlew bootRun
```

別ターミナルで以下の 10 項目を実行:

- [ ] **Step 2: Swagger UI 起動確認**

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/swagger-ui.html
```

Expected: `200` または `302`（リダイレクトされる場合あり）。ブラウザで `http://localhost:8080/swagger-ui.html` を開き、`/auth/login` と `/items` 系エンドポイントが一覧表示されること。

- [ ] **Step 3: H2 Console 起動確認**

ブラウザで `http://localhost:8080/h2-console` を開き、JDBC URL を `jdbc:h2:mem:app` に変更して Connect。`items` テーブルに2行（Sample Widget A / B）が表示されること。

- [ ] **Step 4: `/auth/login` 成功**

```bash
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-user","password":"demo-password"}'
```

Expected: `{"token":"eyJhbGci...","expiresIn":3600}`。

token を環境変数に保存:
```bash
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"demo-user","password":"demo-password"}' | \
  sed -E 's/.*"token":"([^"]+)".*/\1/')
echo "TOKEN=$TOKEN"
```

- [ ] **Step 5: `/auth/login` 失敗 → 401**

```bash
curl -s -w "\nHTTP %{http_code}\n" -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"wrong","password":"wrong"}'
```

Expected: `HTTP 401` + `{"code":"UNAUTHORIZED","message":"認証に失敗しました","timestamp":"..."}`。

- [ ] **Step 6: 認証なしで `/items` → 401**

```bash
curl -s -w "\nHTTP %{http_code}\n" http://localhost:8080/items
```

Expected: `HTTP 401`。

- [ ] **Step 7: 認証付きで `/items` 一覧取得**

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/items
```

Expected: 2件の items JSON 配列（Sample Widget A / B）。

- [ ] **Step 8: `/items` POST で作成**

```bash
curl -s -X POST http://localhost:8080/items \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"New Item","quantity":5}'
```

Expected: HTTP 201 + `{"id":"...","name":"New Item","quantity":5,"createdAt":"...","updatedAt":"..."}`。

作成された id をメモ:
```bash
NEW_ID=$(curl -s -X POST http://localhost:8080/items \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"To Delete","quantity":1}' | \
  sed -E 's/.*"id":"([^"]+)".*/\1/')
echo "NEW_ID=$NEW_ID"
```

- [ ] **Step 9: `/items/{id}` 単体取得**

```bash
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/items/$NEW_ID
```

Expected: HTTP 200 + 該当アイテム JSON。

- [ ] **Step 10: `/items/{id}` PUT で更新**

```bash
curl -s -X PUT http://localhost:8080/items/$NEW_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated","quantity":99}'
```

Expected: HTTP 200 + 更新後 JSON（`quantity` が 99）。

- [ ] **Step 11: `/items/{id}` DELETE → GET で 404**

```bash
curl -s -w "\nHTTP %{http_code}\n" -X DELETE http://localhost:8080/items/$NEW_ID \
  -H "Authorization: Bearer $TOKEN"
curl -s -w "\nHTTP %{http_code}\n" -H "Authorization: Bearer $TOKEN" http://localhost:8080/items/$NEW_ID
```

Expected:
- DELETE: `HTTP 204`（ボディなし）
- GET: `HTTP 404` + `{"code":"ITEM_NOT_FOUND",...}`

- [ ] **Step 12: Swagger UI からの操作**

ブラウザで `http://localhost:8080/swagger-ui.html` を開き:
- 「Authorize」ボタンから `Bearer <TOKEN>` を入力
- `/items` GET の「Try it out」→ Execute → 200 OK で items 一覧

- [ ] **Step 13: bootRun 停止**

Ctrl+C で停止。

すべて通ったら次へ。

---

## Task 18: README.md

**Files:**
- Create: `README.md`

- [ ] **Step 1: `README.md` を作成**

Path: `README.md`

```markdown
# mobile-app-poc-backend

モバイルアプリ PoC のバックエンド API。Spring Boot 3.3 + Java 21 で Schema-First 実装。

## 技術スタック

- Java 21 / Gradle 8.10 (Wrapper同梱)
- Spring Boot 3.3.5 / Spring Security
- MyBatis 3.0 + H2 (インメモリ、本番想定は Oracle)
- JWT (jjwt 0.12)
- springdoc-openapi 2.6 (Swagger UI)
- openapi-generator 7.8 (Schema-First)

## 前提

- JDK 21
- Windows ネイティブ (WSL2/Docker は不要)
- 社内LAN (公開範囲はイントラネット内のみ)
- `git` コマンド (syncSpec タスクから呼び出し)

## セットアップ

### 方法A: GitHub 到達可 (標準)

```powershell
.\gradlew bootRun
```

自動で以下が走る:
1. `syncSpec`: `mobile-app-poc-api-spec` を `src/main/resources/openapi/` に clone/pull
2. `openApiGenerate`: `openapi.yaml` から Controller interface / DTO を `build/generated/` に生成
3. `compileJava`: 生成物 + 手書きコードをコンパイル
4. `bootRun`: http://localhost:8080 で起動

### 方法B: GitHub 到達不可

1. `src/main/resources/openapi/` に api-spec の中身を手動配置 (`openapi.yaml`, `schemas/`, `responses/`, `parameters/`, `examples/`)
2. syncSpec を除外して実行: `.\gradlew openApiGenerate -x syncSpec` → `.\gradlew bootRun`

## 主要コマンド

| コマンド | 用途 |
|---|---|
| `.\gradlew syncSpec` | api-spec 取得/更新 |
| `.\gradlew openApiGenerate` | Controller/DTO 生成 (syncSpec 自動実行) |
| `.\gradlew bootRun` | 起動 (sync → generate → compile → run) |
| `.\gradlew build` | ビルド + テスト |
| `.\gradlew test` | テストのみ |

## 起動後の URL

- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI JSON: http://localhost:8080/v3/api-docs
- H2 Console: http://localhost:8080/h2-console (JDBC URL: `jdbc:h2:mem:app`)

## 動作確認

```powershell
# 1) ログイン → トークン取得
$TOKEN = (Invoke-RestMethod -Method Post -Uri http://localhost:8080/auth/login `
  -ContentType "application/json" `
  -Body '{"username":"demo-user","password":"demo-password"}').token

# 2) items 一覧
Invoke-RestMethod -Uri http://localhost:8080/items -Headers @{Authorization="Bearer $TOKEN"}
```

## トラブルシュート

- **`gradlew` の実行が拒否される**: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`
- **ポート 8080 競合**: `netsh interface ipv4 show excludedportrange protocol=tcp` で Hyper-V 予約範囲を確認。`application.yml` の `server.port` を変更
- **依存解決が遅い/失敗**: `.\gradlew --refresh-dependencies` で再解決
- **`src/main/resources/openapi/` が空**: `.\gradlew syncSpec` を再実行
- **社内プロキシ配下**: `~/.gradle/gradle.properties` に `systemProp.https.proxyHost=...` を設定

## 他リポジトリとの関係

- **消費先**: [mobile-app-poc-api-spec](https://github.com/m-miyawaki-m/mobile-app-poc-api-spec) (`openapi.yaml` の SSOT)
- **並列**: [mobile-app-poc-mock](https://github.com/m-miyawaki-m/mobile-app-poc-mock) (Prism モック。frontend は当面こちらを参照)
- **供給先**: mobile-app-poc-frontend (将来的に mock から接続先を切替)

## 今後の検討

- WebLogic への war デプロイ (現在は `bootJar` のみ、`SpringBootServletInitializer` 未継承)
- Oracle への実接続 (`application.yml` にコメントアウト雛形あり)
- リフレッシュトークン / ユーザーテーブル
- Actuator / メトリクス
- Flyway / Liquibase によるマイグレーション管理
```

---

## Task 19: git init と初期コミット

**Files:**
- Create: `.git/` (git init)

- [ ] **Step 1: git 設定確認**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
git config user.name  2>/dev/null || echo "not set"
git config user.email 2>/dev/null || echo "not set"
```

- [ ] **Step 2: `git init`**

```bash
git init -b main
```

Expected: `Initialized empty Git repository in .../mobile-app-poc-backend/.git/`

- [ ] **Step 3: ローカルリポジトリの user 設定（グローバル未設定の場合）**

```bash
# 必要に応じて（グローバルに user.name/email があればスキップ可）
git config user.name  "m-miyawaki-m"
git config user.email "miyawakifamilymyam@gmail.com"
```

- [ ] **Step 4: `git status` で追加対象を確認**

```bash
git status
```

Expected: `.gitignore`, `.gitattributes`, `README.md`, `build.gradle`, `settings.gradle`, `gradle/`, `gradlew`, `gradlew.bat`, `src/` が untracked。`build/`, `.gradle/`, `src/main/resources/openapi/` は表示されない（gitignore 済み）。

- [ ] **Step 5: 意図しないファイルが untracked に含まれていないか念押し確認**

```bash
git status --short | grep -E "(build/|\.gradle/|openapi/)"
```

Expected: 空（何も出力されない）。もし出力されたら gitignore を見直す。

- [ ] **Step 6: 全ファイル add + 初期コミット**

```bash
git add .
git commit -m "chore: initial commit - mobile-app-poc-backend scaffold

Spring Boot 3.3 + Java 21 + MyBatis + H2 + JWT + Swagger UI.
Schema-First via openapi-generator 7.8 with syncSpec task
that pulls api-spec from GitHub on every build."
```

Expected: `[main (root-commit) ...] chore: initial commit...` と出力。

- [ ] **Step 7: `git log` で確認**

```bash
git log --oneline
```

Expected: 1件のコミットが表示される。

---

## Task 20: GitHub リポジトリ作成と push (ユーザ承認後)

**Files:** None (GitHub operation)

**⚠️ このタスクは実行前にユーザーに確認すること。** api-spec / mock と同じく `gh repo create` で実施。

- [ ] **Step 1: `gh` コマンドで認証確認**

```bash
gh auth status
```

Expected: `m-miyawaki-m` で認証済み。未認証なら `gh auth login` を案内して一旦停止。

- [ ] **Step 2: ユーザー承認を確認**

実装エージェントはここで「GitHub リポジトリ `mobile-app-poc-backend` を作成して push してよいか」をユーザーに確認すること。承認を得てから Step 3 へ。

- [ ] **Step 3: リモート作成 + push**

```bash
cd "C:/Oracle/3df002/mobile-app/mobile-app-poc-backend"
gh repo create mobile-app-poc-backend \
  --public \
  --source=. \
  --remote=origin \
  --description "モバイルアプリ PoC バックエンド。Spring Boot 3 + Java 21 + MyBatis + H2 + JWT、Schema-First for mobile-app-poc-api-spec." \
  --push
```

Expected: `https://github.com/m-miyawaki-m/mobile-app-poc-backend` が作成され、初期コミットが push される。

- [ ] **Step 4: リモート確認**

```bash
git remote -v
git log --oneline origin/main
```

Expected:
- `origin https://github.com/m-miyawaki-m/mobile-app-poc-backend.git (fetch/push)`
- origin/main に初期コミットが見える

- [ ] **Step 5: ブラウザでリポジトリを開く（任意）**

```bash
gh repo view --web
```

---

## 全タスク完了後の状態

- `mobile-app-poc-backend/` が GitHub に公開リポジトリとして存在
- `./gradlew bootRun` で起動 → Swagger UI / H2 Console / `/auth/login` / `/items` CRUD すべて動作
- `ApplicationSmokeTest` 2 件すべて PASS
- api-spec の更新は `./gradlew bootRun` 一発で反映される
