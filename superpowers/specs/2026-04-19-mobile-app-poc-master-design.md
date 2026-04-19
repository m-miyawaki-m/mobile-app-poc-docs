---
name: mobile-app-poc マスター設計（全体分解）
description: 5リポジトリ構成のモバイルアプリPoCを構築するための、全体方針と構築順序を定義するマスター設計書
date: 2026-04-19
status: draft
---

# mobile-app-poc マスター設計（全体分解）

## 目的

業務用デバイス連携を伴うモバイルアプリ開発の、開発環境・技術スタック・アーキテクチャを検証する PoC を、5つの独立リポジトリ構成で構築する。

本書は **各リポジトリのサブ設計書への親ドキュメント** として、全体方針・配置・構築順序・依存関係を定義する。各リポジトリの詳細設計は個別のspecに切り出す。

## 位置づけ

- docs配下6ファイル（`1. mobile-app-poc-全体統括.md` 〜 `6. mobile-app-poc-backend.md`）が原典
- 本書は原典を踏まえた上で「実際に構築する」ための判断をまとめたもの
- 個々のリポジトリ固有の技術判断はサブ設計書に委譲

## リポジトリ構成

| # | リポジトリ名 | 役割 | 主要技術 |
|---|---|---|---|
| 1 | mobile-app-poc-api-spec | API契約（SSOT） | OpenAPI 3.0.3 |
| 2 | mobile-app-poc-mock | モックAPIサーバー | Prism |
| 3 | mobile-app-poc-backend | バックエンドAPI | Spring Boot 3.x + Java 21 |
| 4 | mobile-app-poc-bridge | aar連携プラグイン | Capacitor Custom Plugin |
| 5 | mobile-app-poc-frontend | モバイルフロントエンド | Ionic + Vue + Capacitor |

## 配置

```
C:\Oracle\3df002\mobile-app\
├─ docs\                          （既存・原典）
│  └─ superpowers\specs\          （本書と各サブ設計書）
├─ mobile-app-poc-api-spec\
├─ mobile-app-poc-mock\
├─ mobile-app-poc-backend\
├─ mobile-app-poc-bridge\
└─ mobile-app-poc-frontend\
```

docs原典には `C:\work\mobile-app-poc\` とあるが、既に `C:\Oracle\3df002\mobile-app\` にdocsが配置済みのため、ここを親ディレクトリとして採用。

## 構築順序と依存関係

```
api-spec ──┬─> mock ───────────┐
           ├─> backend ────────┤
           │                    ├─> frontend（最終結合先）
           └─> （型生成）───────┤
                                │
bridge ─────────────────────────┘
```

**構築順序は依存順に厳守**:

1. **api-spec** — `openapi.yaml` がmock/backend/frontendすべての入力。最初に固める
2. **mock** — Prismでapi-specを起動可能にする。frontendの動作確認先
3. **backend** — Schema-First生成。api-specの第二の消費者
4. **bridge** — Capacitorプラグイン単体。frontendに組み込まれるが単独開発可能
5. **frontend** — 全てを統合する最終工程。mock/bridgeをimportし、将来backendへ切替

各段階で動作確認可能な状態を保ち、後工程の手戻りを最小化する。

## 初期APIスコープ（api-spec）

PoCの主目的は「開発環境・技術スタックの検証」であり、ビジネスロジックではない。初期APIは最小に抑え、**5リポ間のつなぎ目の検証** に集中する。

- `POST /auth/login` — JWT発行
- `GET /items` — 一覧取得
- `POST /items` — 新規作成
- `GET /items/{id}` — 単体取得
- `PUT /items/{id}` — 更新
- `DELETE /items/{id}` — 削除
- 共通エラースキーマ（400 / 401 / 404 / 500）

ドメインは汎用的な「アイテム」を採用（ユーザー/商品/在庫などの抽象）。検証が一周してから拡張検討。

## Git運用

- 各リポジトリは**独立したGitリポジトリ**として構築
- scaffold完了時点で `git init` + 初期コミット（アシスタントが実施）
- GitHub リモートへの push は各リポジトリごとにユーザが個別実施（アシスタントは関与しない）
- 親ディレクトリ `C:\Oracle\3df002\mobile-app\` 自体はGit管理しない
- 各リポジトリには `.gitignore` と `.gitattributes`（改行コード `eol=lf`）を初期コミットに含める

## 各リポジトリのサイクル

各リポジトリは以下のサイクルを**個別に**一周する:

```
brainstorming → spec（詳細設計書） → plan（実装計画） → implementation → 動作確認 → git init + 初期コミット
```

サブ設計書の配置: `docs/superpowers/specs/YYYY-MM-DD-<repo-name>-design.md`

## 対象外

- 外部公開（Tailscale Funnel / ngrok / クラウドデプロイ）
- iOS対応（bridge はAndroidのみ、frontendはAndroid実機動作を優先）
- 本番レベルのセキュリティ要件（社内イントラ前提）
- パブリックnpm registryへの公開
- WebLogicへの実機デプロイ検証（backend調査観点には含むが、PoCスコープ外）

## 次のステップ

本書承認後、**mobile-app-poc-api-spec のbrainstorming** を開始する。
