---
title: "Documentation Standards"
description: "ドキュメント標準 - README・CONTRIBUTING・CHANGELOG・API文書の構成規約 / Documentation standards - README, CONTRIBUTING, CHANGELOG, API docs"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-27T00:45+09:00"
lang: "ja"
---

# Documentation Standards

**説明** - リポジトリドキュメントの標準形式を定義する。README 構成テンプレート、CHANGELOG 運用方針、OpenAPI/Swagger 仕様書の記述ルールを対象とする。


---

## 目的

ドキュメントの一貫性を確保し、読み手の認知コストを下げることを目的とする。生成物と手動記述の両方に適用される共通基準を定義する。

## 背景

README・CHANGELOG・API 仕様書はプロジェクトの外向き契約として機能する。形式が統一されていない場合、オンボーディングコストの増大・API クライアント SDK の破壊的変更・ドキュメントの陳腐化といった問題が発生する。本規約はそれぞれの標準形式を定め、自動化ツールとの連携を前提とした運用基準を提供する。

## 対象

- 全リポジトリの `README.md`
- 全リポジトリの `CHANGELOG.md`
- REST API を持つプロジェクトの OpenAPI 仕様書

## 非対象

- Storybook・VitePress・Docusaurus 等の専用ドキュメントサイト構成
- コードコメント規約（[javascript-typescript-standards.md](javascript-typescript-standards.md) を参照）
- API エンドポイント設計原則（バックエンド向け、本規約の対象外）

---

## README

### 配置と命名

- リポジトリルートに `README.md` を必ず配置する。
- ファイル名は `README.md` と大文字で命名する（GitHub / GitLab の自動表示条件に対応するため）。
- サブパッケージを持つモノレポでは、各パッケージルートにサブ `README.md` を配置する。

出典・参考情報: The Good Docs Project - README Template

### 必須セクションと推奨順序

以下の順序を標準とする。

1. バッジ（任意）
2. プロジェクト名・ロゴ（任意）
3. 概要（必須）
4. 前提条件（必須）
5. インストール（必須）
6. 使い方（必須）
7. 環境変数（必須・該当時）
8. ディレクトリ構成（任意）
9. テスト実行（必須・自動テストがある場合）
10. デプロイ（任意）
11. 貢献方法（任意）
12. ライセンス（必須）
13. 謝辞（任意）

出典・参考情報: The Good Docs Project - README Template、othneildrew/Best-README-Template

### 記述規則

- 概要セクションは使用技術より先に「何をするか」を書く。技術スタックは前提条件以降で記述する。
- 「10分以内にプロジェクトを起動できるか」を自己テストとして用いる。インストールから起動までの手順を一通り実行して確認する。
- バッジは自動更新されるもののみ使用する。更新されないバッジはリポジトリの信頼性を損なう。
- 環境変数は `.env.example` とセットで管理し、シークレット値のサンプルはダミー値を記載する。

出典・参考情報: freecodecamp - How to Structure Your README File

### テンプレート

```markdown
<!-- バッジ（任意）-->
![Build](https://github.com/ORG/REPO/actions/workflows/ci.yml/badge.svg)

# プロジェクト名

<!-- 1〜2文でプロジェクトの目的を説明する。技術スタックより先に「何をするか」を書く -->
一行で説明できる、このプロジェクトが解決する課題と提供する価値。

## 前提条件

- Node.js: 20 以上
- npm: 10 以上

## インストール

\`\`\`zsh
git clone https://github.com/ORG/REPO.git
cd REPO
npm ci
cp .env.example .env
\`\`\`

## 使い方

\`\`\`zsh
npm run dev
\`\`\`

## 環境変数

| 変数名 | 説明 | 必須 | 例 |
|--------|------|------|-----|
| `DATABASE_URL` | データベース接続文字列 | Yes | `postgresql://...` |
| `API_KEY` | 外部サービス API キー | Yes | — |
| `PORT` | サーバーポート番号 | No | `3000` |

`.env.example` を参照すること。シークレット値は `.env` に記載し、リポジトリへコミットしない。

## テスト実行

\`\`\`zsh
npm test
npm run test:e2e
\`\`\`

## 貢献方法

[CONTRIBUTING.md](CONTRIBUTING.md) を参照すること。

## ライセンス

[MIT](LICENSE)
```

### ライセンス

採用可能なライセンスの種類・選定基準・記載規則を定義する。

**種類と特徴**

| ライセンス | 特徴 | 採用ケース |
|-----------|------|-----------|
| MIT | 再配布・商用利用・改変すべて可。制約が最も少ない | OSS・社内ライブラリ・汎用ツール |
| Apache 2.0 | MIT に特許条項を追加。特許リスク対策 | 企業が関与する OSS |
| GPL v3 | 派生物も同ライセンス強制（コピーレフト） | 完全 OSS・派生物の公開を義務付けたい場合 |
| Proprietary | 全権利を保有。再配布・改変を禁止 | 商用プロダクト・クローズドソース |

**選定基準**

- 社内専用プロジェクト → Proprietary
- 外部公開 OSS → MIT または Apache 2.0（特許リスクがある場合は Apache 2.0）
- 派生物の公開を義務付けたい場合 → GPL v3

**記載規則**

- `LICENSE` ファイル（拡張子なし）をリポジトリルートに配置する。
- README の `## ライセンス` セクションにライセンス名とファイルへのリンクを記載する。
- 著作権表記は `Copyright (c) YEAR OWNER` の形式で `LICENSE` ファイル先頭に記載する。
- 複数ライセンスが混在する場合はファイル先頭に SPDX 識別子（`SPDX-License-Identifier: MIT`）を記載する。

出典・参考情報: choosealicense.com、SPDX License List

---

## CONTRIBUTING.md

貢献方法を定義するファイル。リポジトリルートに `CONTRIBUTING.md` として配置する。

**必須セクション**

1. 前提条件（開発環境のセットアップ手順）
2. ブランチ戦略（[git-workflow.md](git-workflow.md) §2 を参照）
3. コミットメッセージ規約（[git-workflow.md](git-workflow.md) §3 を参照）
4. PR 提出手順（レビュー依頼・チェックリスト）
5. コーディング規約へのリンク（coding-standards.md 等）
6. 行動規範（Code of Conduct）へのリンクまたは記載

**テンプレート**

```markdown
# 貢献ガイド

## 開発環境セットアップ

\`\`\`zsh
git clone https://github.com/ORG/REPO.git
cd REPO
npm ci
cp .env.example .env
\`\`\`

## ブランチ戦略

[git-workflow.md](git-workflow.md) §2 を参照すること。

## コミットメッセージ規約

[git-workflow.md](git-workflow.md) §3（Conventional Commits）に従うこと。

## PR 提出手順

1. `main` から feature ブランチを切る。
2. 変更を実装しテストを通す。
3. PR テンプレートに従って説明を記載する。
4. レビュワーをアサインする。

## コーディング規約

- [javascript-typescript-standards.md](javascript-typescript-standards.md)
- [component-design-patterns.md](component-design-patterns.md)

## 行動規範

すべての貢献者は [Contributor Covenant](https://www.contributor-covenant.org/ja/version/2/1/code_of_conduct/) に従うこと。
```

出典・参考情報: GitHub - Contributing to a Project、Contributor Covenant v2.1

---

## CHANGELOG

形式・セクション定義・自動生成ツール（semantic-release / conventional-changelog / commit-and-tag-version）は [git-workflow.md](git-workflow.md) §8 を正本とする。本セクションでは運用タイミングと禁止事項を補足する。

### 運用タイミング

1. リリースタグを作成する前に `[Unreleased]` セクションを対象バージョンに移動する。
2. `main` へのマージ後は、コミットログから自動生成ツールで追記することを推奨する。
3. ホットフィックスは通常リリースと同じ形式でエントリを追加する。

### 禁止事項

- コミットメッセージをそのままコピーしない。ユーザー向けに意味のある変更として言語化する。
- 内部リファクタリングのみの変更は `### 変更` ではなく省略するか `### 内部`（任意セクション）に分類する。

---

## OpenAPI / Swagger 仕様書

### バージョン選定

| バージョン | 状態 | 採用推奨度 |
|-----------|------|-----------|
| OAS 3.2.0（2025年9月） | 最新。ストリーミング・タグナビゲーション強化 | ツールサポートが安定後に採用を検討 |
| OAS 3.1.1（現行安定） | JSON Schema 2020-12 準拠。Redoc・Swagger UI 対応済み | **新規プロジェクトで採用推奨** |
| OAS 3.0.3 | 広く使われているが JSON Schema と乖離あり | 既存資産の維持のみ |

出典・参考情報: OpenAPI Specification v3.1.1、OpenAPI Specification v3.2.0

### ファイル形式と配置

- 形式: YAML を標準とする（JSON も可。プロジェクト内で統一する）。
- 配置: `docs/openapi.yaml` または `openapi/openapi.yaml`。
- CI 検証: Spectral によるリントを必須とする。

### 必須フィールド構成

```yaml
openapi: 3.1.1
info:
  title: Product API           # 50文字以内
  version: 1.0.0               # SemVer 準拠
  description: |
    API の目的と対象読者を記載する。
  contact:
    name: API Support
    url: https://example.com/support
    email: api@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://api-staging.example.com/v1
    description: Staging

tags:
  - name: users
    description: ユーザー管理に関する操作
  - name: auth
    description: 認証・認可に関する操作

paths:
  /users/{id}:
    get:
      tags: [users]
      operationId: getUserById   # 一意で安定した識別子（リリース後に変更しない）
      summary: ID でユーザーを取得
      description: 指定した ID のユーザー情報を返す。
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
              examples:
                example-1:
                  value:
                    id: "550e8400-e29b-41d4-a716-446655440000"
                    name: "山田太郎"
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '422':
          $ref: '#/components/responses/UnprocessableEntity'
        '500':
          $ref: '#/components/responses/InternalServerError'

components:
  schemas:
    User:
      type: object
      required: [id, name]
      properties:
        id:
          type: string
          format: uuid
          example: "550e8400-e29b-41d4-a716-446655440000"
        name:
          type: string
          example: "山田太郎"
    ErrorResponse:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
          example: "NOT_FOUND"
        message:
          type: string
          example: "指定したリソースが見つかりません。"

  responses:
    NotFound:
      description: リソースが見つからない
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    Unauthorized:
      description: 認証が必要
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    UnprocessableEntity:
      description: バリデーションエラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
    InternalServerError:
      description: サーバー内部エラー
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

出典・参考情報: OpenAPI Specification v3.1.1

### ベストプラクティス（5原則）

1. **Design-First を採用する**  
   コードを書く前に仕様書を確定し、仕様書をコントラクトとして扱う。コードから生成した仕様書は実態の記録であり、設計の合意点にはなりにくい。

2. **`$ref` でスキーマを一元管理する**  
   インラインスキーマを禁止し、すべてのスキーマ・パラメーター・レスポンスを `components` に定義して `$ref` で参照する。スキーマの変更を1箇所で済ませるために必須。

3. **全エンドポイントに examples を記載する**  
   `example` または `examples` が空のエンドポイントをリリースしない。テストツールがサンプルリクエストを自動生成する際に使われる。スキーマに準拠しない値は Spectral で検出される。

4. **エラーレスポンスを全エンドポイントに定義する**  
   最低限 `401`・`404`・`422`（バリデーションエラー）・`500` を `components/responses` に定義して参照する。エラーコード設計は [http-error-code-reference.md](http-error-code-reference.md) を参照する。

5. **Spectral でリントする**  
   PR 時に Spectral でリントを必須とする。`openapi` ルールセットを基礎とし、必要に応じてカスタムルールを追加する。

出典・参考情報: learn.openapis.org - Best Practices、apimatic.io - 14 Best Practices to Write OpenAPI

### ドキュメント生成ツール

| ツール | 用途 |
|--------|------|
| Swagger UI | インタラクティブな API ドキュメント |
| Redoc | 読みやすい静的ドキュメント |
| Redocly CLI | バンドル・リント・プレビュー |

### 禁止事項

- `operationId` をリリース後に変更しない。クライアント SDK の破壊的変更になる。
- インラインスキーマを `paths` 内に直接定義しない。`components/schemas` に分離する。
- `servers` を空または未定義のままにしない。ツールが `localhost` にフォールバックする。
- スキーマに準拠しない `example` 値を記載しない。

---

## 実装例

```zsh
# Spectral インストール
npm install --save-dev @stoplight/spectral-cli

# リント実行
npx spectral lint openapi/openapi.yaml --ruleset .spectral.yaml
```

```yaml
# .spectral.yaml（最小構成）
extends: ["spectral:oas"]
rules:
  operation-operationId: error
  operation-description: warn
  operation-tag-defined: error
```

---

## 出典・参考情報

### 情報・規格

- [The Good Docs Project - README Template](https://www.thegooddocsproject.dev/template/readme) - README セクション構成の根拠
- [Keep a Changelog 1.1.0](https://keepachangelog.com/ja/1.1.0/) - CHANGELOG 形式（[git-workflow.md](git-workflow.md) §8 を正本とする）
- [OpenAPI Specification v3.1.1](https://spec.openapis.org/oas/v3.1.1.html) - 現行安定版公式仕様
- [OpenAPI Specification v3.2.0](https://spec.openapis.org/oas/v3.2.0.html) - 2025年9月リリース。ストリーミング・タグナビゲーション追加
- [choosealicense.com](https://choosealicense.com/) - ライセンス種類と選定基準の根拠
- [SPDX License List](https://spdx.org/licenses/) - SPDX 識別子の正式定義

### ツール公式ドキュメント

- [Spectral](https://docs.stoplight.io/docs/spectral/674b27b261c3c-overview) - OpenAPI リント標準ツール
- [Swagger UI](https://swagger.io/tools/swagger-ui/) - インタラクティブドキュメント生成
- [Redoc](https://redocly.com/docs/redoc/) - 静的ドキュメント生成
- [Redocly CLI](https://redocly.com/docs/cli/) - バンドル・リント・プレビュー
- [othneildrew/Best-README-Template](https://github.com/othneildrew/Best-README-Template) - README 構成の参考実装

### ベストプラクティス

- [learn.openapis.org - Best Practices](https://learn.openapis.org/best-practices.html) - Design-First・タグ設計
- [apimatic.io - 14 Best Practices to Write OpenAPI](https://www.apimatic.io/blog/2022/11/14-best-practices-to-write-openapi-for-better-api-consumption) - インラインスキーマ禁止・examples 必須の根拠
- [freecodecamp - How to Structure Your README File](https://www.freecodecamp.org/news/how-to-structure-your-readme-file/) - 「10分以内に起動できるか」テストの根拠
- [GitHub - Contributing to a Project](https://docs.github.com/ja/get-started/exploring-projects-on-github/contributing-to-a-project) - CONTRIBUTING.md 構成の根拠
- [Contributor Covenant v2.1](https://www.contributor-covenant.org/ja/version/2/1/code_of_conduct/) - 行動規範標準テンプレート
