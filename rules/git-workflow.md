---
title: "Git Workflow Guidelines"
description: "Gitワークフロー - コミット規約・ブランチ戦略・PR/MRテンプレート・マージ戦略 / Git workflow - commit conventions, branching strategy, PR/MR templates, merge strategy"
version: "1.1.0"
status: "Stable"
last_updated: "2026-02-21T14:24+09:00"
lang: "ja"
---

# Git Workflow Guidelines

**説明** - プロダクトタイプ・チーム規模・フェーズに応じたブランチ戦略の選択・運用・切り替えに関するワークフロー規約を定義する。


---

## 目的

リポジトリ運用の一貫性を確保し、個人・チーム・プロジェクトの規模やフェーズに応じた最適なブランチ戦略・コミット規約・リリース管理を定義する。

---

## 背景

同一プロダクトであってもフェーズや人数規模によって最適な戦略が異なる。また、戦略の途中切り替えを想定した運用手順が必要である。

---

## 対象

- 個人開発・チーム開発を問わず本規約を適用するすべてのリポジトリ
- Webアプリ・モバイルアプリ・ライブラリ・ドキュメントサイト等すべてのプロダクトタイプ

---

## 非対象

- 社外OSS・外部リポジトリへのコントリビューション（当該リポジトリの規約を優先する）

---

## 目次

1. [セットアップ：プロジェクト初期判断](#1-セットアッププロジェクト初期判断)
2. [ブランチ戦略](#2-ブランチ戦略)
3. [ブランチ戦略の切り替え](#3-ブランチ戦略の切り替え)
4. [バージョニング](#4-バージョニング)
5. [コミット規約](#5-コミット規約)
6. [PRレビュー基準とマージ方式](#6-prレビュー基準とマージ方式)
7. [タグ・リリースノート](#7-タグリリースノート)
8. [CHANGELOG](#8-changelog)
9. [補助設定ファイル](#9-補助設定ファイル)
10. [ユースケース別リファレンス](#10-ユースケース別リファレンス)
11. [Rebase / Force-push 規律](#11-rebase--force-push-規律)
12. [Rollback / Incident Runbook](#12-rollback--incident-runbook)
13. [リリース判定ゲート](#13-リリース判定ゲート)
14. [機密情報漏えい時のGit対応](#14-機密情報漏えい時のgit対応)

---

## 1. セットアップ：プロジェクト初期判断

新規プロジェクト開始時、または戦略見直し時に以下の2つを確定する。

### 1.1 プロダクトタイプの選択

| # | タイプ | 具体例 | 推奨戦略 |
|---|--------|--------|----------|
| A | Webアプリ / SaaS（継続的デリバリー） | Webサービス、API、ダッシュボード | GitHub Flow / TBD |
| B | モバイルアプリ（App Store審査あり） | iOS / Android ネイティブアプリ | Gitflow / 変形GitHub Flow |
| C | ライブラリ / パッケージ（明示バージョニング） | npmパッケージ、SDK、OSSライブラリ | Gitflow |
| D | 静的サイト / ドキュメント | ランディングページ、ブログ、ドキュメントサイト | GitHub Flow |
| E | 大規模プロダクト / 複数バージョン並行 | エンタープライズ、マルチリリース製品 | Gitflow / GitLab Flow |
| F | 高速プロトタイプ / 実験 | PoC、ハッカソン | Trunk-Based Development |

### 1.2 チーム規模の確認

| 規模 | 人数 | 特性 | 考慮事項 |
|------|------|------|----------|
| Solo | 1名 | 個人開発、意思決定が速い | シンプルなフローを優先 |
| Small | 2名から5名 | スタートアップ、小チーム | PR必須、レビュー1名以上 |
| Medium | 6名から20名 | 成長期チーム | CODEOWNERS設定、明確な役割分担 |
| Large | 21名以上 | 大規模組織 | 自動化必須、厳格なブランチ保護 |

> **決定記録**：決定したタイプと規模をリポジトリの `README.md` または `CONTRIBUTING.md` に明記する。

---

## 2. ブランチ戦略

### この章の用語

- ブランチ戦略: リポジトリ内でブランチを作成・統合・削除するルールの体系。
- `Trunk / main`: 常にデプロイ可能な状態を維持するデフォルトブランチ。
- 短命ブランチ: 作業完了後にマージ・削除される一時的なブランチ。
- Feature Flag: マージ済みコードの機能有効・無効を実行時に切り替える仕組み。

### 出典情報

- [Atlassian: Comparing Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [Runway: Mobile Branching Strategy](https://www.runway.team/blog/choosing-the-right-branching-strategy-for-mobile-development)

### 2.1 GitHub Flow（推奨：タイプA / D、SoloからMedium）

**概要：** `main` ブランチ + 短命なフィーチャーブランチ。最もシンプルで継続的デリバリーに最適。

```
main ──●──●──────────────────●── (常にデプロイ可能)
         \                  /
          feature/add-auth ●──●
```

**ブランチ種別と命名（必須）：**

```
feature/[issue-id]-short-description
fix/[issue-id]-short-description
hotfix/[issue-id]-short-description
docs/short-description
design/[issue-id]-short-description
asset/[issue-id]-short-description
chore/short-description
refactor/short-description
```

| パターン | 寿命 | 適用条件 |
|----------|------|----------|
| `main` | 永続 | 常にデプロイ可能な状態を維持するデフォルトブランチ |
| `feature/[issue-id]-*` | 短命（1日から3日を推奨） | 新機能追加 |
| `fix/[issue-id]-*` | 短命（1日から3日を推奨） | 既存不具合修正 |
| `hotfix/[issue-id]-*` | 短命（即日） | 本番障害・セキュリティ事故の緊急対応 |
| `docs/*` | 短命 | ドキュメント修正のみ |
| `design/[issue-id]-*` | 短命 | デザイン仕様変更（Figma連携） |
| `asset/[issue-id]-*` | 短命 | 画像・動画・素材の更新 |
| `chore/*` | 短命 | 雑務系変更（設定・依存更新など） |
| `refactor/*` | 短命 | 挙動変更を伴わないリファクタリング |

> 全作業ブランチは `main` から分岐し、PR経由で `main` に統合する。

**基本フロー：**

```zsh
# 1. mainから作業ブランチを作成
git checkout main
git pull origin main
git checkout -b feature/123-add-user-auth

# 2. 作業・コミット
git add .
git commit -m "feat(auth): add JWT authentication"

# 3. プッシュ・PR作成
git push origin feature/123-add-user-auth
# GitHub/GitLabでPRを作成

# 4. レビュー後にmainへマージし、即時デプロイ

# 5. ブランチ削除
git branch -d feature/123-add-user-auth
git push origin --delete feature/123-add-user-auth
```

---

### 2.2 Gitflow（推奨：タイプB / C / E、SmallからLarge）

**概要：** `main` + `develop` + フィーチャー / リリース / ホットフィックスブランチ。バージョン管理が明示的で、モバイルアプリやライブラリに適する。

```
main    ──●─────────────────────────●── (v1.0.0)   (v1.1.0)
           \                       /|\             /
hotfix/     \                     / | \           /
release/     \            ●──●──●  |  \         /
              \          /         |   \       /
develop   ──●──●──●──●──●──────────●────●──●──●──●──
                    \        /
feature/             ●──●──●
```

**ブランチ構成：**

| ブランチ | 寿命 | 説明 |
|----------|------|------|
| `main` | 永続 | リリース済みのコードのみ。タグで版管理 |
| `develop` | 永続 | 次リリースへの統合ブランチ |
| `feature/*` | 短命 | `develop` から分岐、`develop` へマージ |
| `release/*` | 短命 | `develop` から分岐。バグ修正のみ可 |
| `hotfix/*` | 短命 | `main` タグから分岐。`main` と `develop` 両方へマージ |
| `bugfix/*` | 短命 | `release/*` 内の修正用（拡張）|

**基本フロー：**

```zsh
# フィーチャー開発
git checkout develop
git pull origin develop
git checkout -b feature/456-push-notifications

# フィーチャー完了後にdevelopへマージ
git checkout develop
git merge --no-ff feature/456-push-notifications
git push origin develop
git branch -d feature/456-push-notifications

# リリース準備
git checkout -b release/1.2.0 develop
# バグ修正のみ。バージョン番号更新
git commit -m "chore(release): bump version to 1.2.0"

# リリース確定
git checkout main
git merge --no-ff release/1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"
git checkout develop
git merge --no-ff release/1.2.0
git branch -d release/1.2.0
git push origin main develop --tags

# ホットフィックス
git checkout -b hotfix/1.2.1 v1.2.0
git commit -m "fix(crash): prevent null pointer on startup"
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1"
git checkout develop
git merge --no-ff hotfix/1.2.1
git branch -d hotfix/1.2.1
git push origin main develop --tags
```

---

### 2.3 Trunk-Based Development（推奨：タイプF / 高速CIチーム、MediumからLarge）

**概要：** 全変更を `main`（trunk）に短期間で統合。CI/CDと機能フラグを前提とする。

```
main ──●──●──●──●──●──●──●──●──  (常に green)
          |     |
         短命   短命（< 1日）
```

**ブランチ構成：**

| ブランチ | 寿命 | 説明 |
|----------|------|------|
| `main` | 永続 | 常にビルド可能・テスト通過状態 |
| `feature/*` | 極短命（< 1日推奨） | PRを通じて即mainへ統合 |

**機能フラグ（Feature Flags）の使用：**

```typescript
// 未完成機能はフラグで隠蔽してmainにマージ可能にする
if (featureFlags.isEnabled('new-payment-flow')) {
  return <NewPaymentFlow />;
}
return <LegacyPaymentFlow />;
```

---

### 2.4 GitLab Flow（参考：タイプA / E、環境別デプロイ）

**概要：** GitHub Flowに環境ブランチを追加したハイブリッド。`production` や `staging` ブランチで環境を管理する。

```
mainからstagingを経由してproduction
```

---

## 3. ブランチ戦略の切り替え

同一プロダクトでも、フェーズやチーム規模の変化に応じて戦略を切り替えることがある。

### 3.1 切り替えトリガー

| 変化の内容 | 切り替え方向 | 理由 |
|-----------|-------------|------|
| チームが2名から10名に増加 | GitHub FlowからGitflow | 協調・安定性の必要性が増す |
| Webアプリがモバイルアプリを追加 | GitHub FlowからGitflow | App Store審査サイクルへの対応 |
| ライブラリのリリースサイクルが規則化 | TBDからGitflow | 明示バージョン管理が必要 |
| 大規模チームがCI/CDを整備完了 | GitflowからTBD | 統合頻度を上げて品質を高める |
| スタートアップの初期から成長期へ移行 | TBDからGitHub Flow | PR文化・コードレビューの導入 |

### 3.2 切り替え手順

**GitflowからGitHub Flowへの移行手順（必須）：**

1. `develop` を凍結し、新規PRのマージ先として使用停止を宣言する。
2. 未マージPRが `0` 件であることを確認する。
3. `develop` の最新を `main` へ統合し、CI成功を確認する。
4. 保全タグを作成してリモートへ送信する。
5. 既定ブランチと保護ルールを `main` に更新する。
6. `develop` は削除せず、最低14日間は読み取り専用で保持する。
7. 保持期間中にロールバック要件が発生しないことを確認後、削除を任意で実施する。

```zsh
# 2. 未マージPR/MRの確認
gh pr list --base develop --state open
glab mr list --target-branch develop --state opened
```

```zsh
# 3. develop を main へ統合
git checkout main
git pull origin main
git merge --no-ff develop
git push origin main

# 4. 保全タグを作成（日時はISO 8601を推奨）
git tag -a develop-archive-2026-02-21T01-53-00+09-00 -m "Archive before strategy switch"
git push origin develop-archive-2026-02-21T01-53-00+09-00

# 7. 削除する場合のみ実行（任意）
git push origin --delete develop
```

> **切り替え記録：** 切り替え日時（ISO 8601）・理由・担当者を `CONTRIBUTING.md` または運用ログに残す。

---

## 4. バージョニング

### この章の用語

- SemVer: `MAJOR.MINOR.PATCH` で互換性を表現するバージョニング仕様。
- Build番号: リリース番号とは独立した、ビルド単位の識別番号。

### 出典情報

- [Semantic Versioning 2.0.0](https://semver.org/)
- [GitVersion Docs](https://gitversion.net/docs/reference/version-increments)
- [GitHub Actions: Variables](https://docs.github.com/en/actions/learn-github-actions/variables)
- [GitLab CI/CD predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html)

### 4.1 Semantic Versioning（SemVer 2.0.0）

```
MAJOR.MINOR.PATCH[-prerelease][+build]
```

| 要素 | 増加条件 | 例 |
|------|----------|-----|
| MAJOR | 後方互換性のない変更（Breaking Change） | 1.0.0 から 2.0.0 |
| MINOR | 後方互換性のある機能追加 | 1.0.0 から 1.1.0 |
| PATCH | 後方互換性のあるバグ修正 | 1.0.0 から 1.0.1 |

**プレリリース識別子：**

```
1.0.0-alpha.1      # 社内テスト
1.0.0-beta.1       # 外部ベータテスター
1.0.0-rc.1         # リリース候補
1.0.0              # 安定リリース
```

**ルール：**

- バージョン `0.y.z` は初期開発フェーズ（APIは安定していない）
- `1.0.0` でパブリックAPIを定義する
- 一度リリースしたバージョンのコードは変更禁止

### 4.2 Build番号

Buildはリリースごとではなく、**ビルドごと**に増加する内部識別子である。SemVerとは独立して管理する。

**形式：**

```
MAJOR.MINOR.PATCH+BUILD
例: 1.2.3+456
```

**Build番号ポリシー（必須）：**

```zsh
# iOS/Android: CI起点の単調増加整数のみ許可
if [ -n "$GITHUB_RUN_NUMBER" ]; then
  BUILD_NUMBER="$GITHUB_RUN_NUMBER"
elif [ -n "$CI_PIPELINE_IID" ]; then
  BUILD_NUMBER="$CI_PIPELINE_IID"
else
  echo "CI起点の整数Build番号が取得できないため中止"
  exit 1
fi
echo "BUILD_NUMBER=${BUILD_NUMBER}"
```

```zsh
# Web/内部配布: SemVer build metadata（+build）の付与を許可
SHA=$(git rev-parse --short HEAD)
echo "1.2.3-rc.1+${GITHUB_RUN_NUMBER}.${SHA}"
```

**プラットフォーム別Build番号管理：**

| プラットフォーム | 設定箇所 | 許可される形式 | 禁止事項 |
|-----------------|----------|----------------|----------|
| iOS | `CFBundleVersion` | `456` のような単調増加整数 | `+build`、タイムスタンプ、Gitコミット数の直接利用 |
| Android | `versionCode` | `456` のような単調増加整数 | `+build`、タイムスタンプ、Gitコミット数の直接利用 |
| Web / Node.js | `package.json` の `version` | `1.2.3+456` などのSemVer build metadata | なし（ただしSemVer違反は不可） |
| GitHub Actions | `GITHUB_RUN_NUMBER` | 実行ごとの連番 | なし |
| GitLab CI | `CI_PIPELINE_IID` | プロジェクト内連番 | なし |

**iOS/Android の例（Gitflow運用時）：**

```zsh
# iOSでのバージョン・ビルド番号設定例
VERSION_NAME="1.2.3"      # SemVer（CFBundleShortVersionString）
BUILD_NUMBER="456"         # 常に増加（CFBundleVersion）

# PATCH番号とBuild番号は別物
# 1.2.3 のビルドが複数あり得る：1.2.3+454, 1.2.3+455, 1.2.3+456
```

### 4.3 Conventional Commitsとの連携

**標準連携（必須）：**

```
feat: MINOR バージョンアップ
fix: PATCH バージョンアップ
feat! / BREAKING CHANGE: MAJOR バージョンアップ
```

**拡張連携（任意）：**

`perf`、`revert`、`refactor` などをバージョン計算へ反映する場合は、各リポジトリのリリース設定ファイル（`semantic-release` 設定等）で明示する。未定義の場合はバージョン影響なしとして扱う。

---

## 5. コミット規約

### 出典情報

- [Conventional Commits v1.0.0](https://www.conventionalcommits.org/ja/v1.0.0/)

### 5.1 Conventional Commits 1.0.0

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### 5.2 タイプ一覧

| タイプ | SemVer影響 | 用途 |
|--------|------------|------|
| `feat` | MINOR | 新機能 |
| `fix` | PATCH | バグ修正 |
| `feat!` / `BREAKING CHANGE` | MAJOR | 後方互換性のない変更 |
| `docs` | なし | ドキュメントのみの変更 |
| `style` | なし | コードの意味に影響しない変更（空白・フォーマット等） |
| `refactor` | なし | バグ修正でも機能追加でもないコード変更 |
| `test` | なし | テストの追加・修正 |
| `build` | なし | ビルドシステム・依存関係の変更 |
| `ci` | なし | CI設定ファイルの変更 |
| `chore` | なし | その他の変更（ソースもテストも変更しない） |
| `perf` | 設定依存 | パフォーマンス改善 |
| `revert` | 設定依存 | 以前のコミットの取り消し |

### 5.3 書式ルール

```
# 件名（description）
- 命令形・現在形で記述（"add" であり "added" や "adds" ではない）
- 先頭を大文字にしない
- 末尾にピリオドを付けない
- 72文字以内に収める

# 本文（body）
- 件名と本文の間に空行を挿入
- 1行72文字以内
- 「何を」ではなく「なぜ」を説明する

# フッター（footer）
- 課題参照：Closes #123, Refs #456
- 共同著者：Co-authored-by: Name <email@example.com>
- 破壊的変更：BREAKING CHANGE: <description>
```

### 5.4 コミットメッセージの例

```zsh
# シンプルな修正
git commit -m "fix(auth): prevent token expiry on page refresh"

# スコープあり・本文あり
git commit -m "feat(payment): add Apple Pay support

Integrate Apple Pay using PassKit framework.
Previously only credit card payments were supported.

Closes #789"

# 破壊的変更
git commit -m "feat(api)!: remove deprecated /v1/users endpoint

BREAKING CHANGE: /v1/users has been removed. Use /v2/users instead.
Migration guide: https://docs.example.com/v2-migration"

# 複数の課題参照
git commit -m "fix(ui): correct button alignment in dark mode

Refs #101, Closes #102"
```

---

## 6. PRレビュー基準とマージ方式

### この章の用語

- PR: コードレビューを経て対象ブランチへのマージを要求する仕組み。
- CODEOWNERS: 特定のファイルやディレクトリの責任者を定義するファイル。

### 出典情報

- [Google Engineering Practices: Code Review](https://google.github.io/eng-practices/review/)
- [Microsoft Engineering Playbook](https://microsoft.github.io/code-with-engineering-playbook/code-reviews/pull-requests/)
- [GitHub Docs: About pull request merges](https://docs.github.com/articles/about-pull-request-merges)
- [Atlassian: Merging vs. Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)

### 6.1 PRの基本原則

| 項目 | 推奨値 | 理由 |
|------|--------|------|
| PR規模 | 200行から400行以内 | レビュー精度と速度のバランス |
| 対象 | 単一の目的に絞る | 追跡・差し戻しが容易 |
| レビュー待ち最大時間 | 24時間以内 | 開発の継続性 |

### 6.2 PRテンプレート（`.github/PULL_REQUEST_TEMPLATE.md`）

```markdown
## 概要

<!-- 変更の目的と背景を簡潔に説明する -->

## 変更内容

<!-- 具体的な変更点を列挙する -->

## テスト方法

<!-- レビュアーが動作確認する手順を記載する -->

## 関連Issue

Closes #

## チェックリスト

- [ ] 自己レビュー済み
- [ ] テスト追加済み（または不要な理由を記載）
- [ ] ドキュメント更新済み（または不要な理由を記載）
- [ ] CI/CDパスした状態でPRを作成した
- [ ] Breaking Changeがある場合は `!` を付けたコミットでマークした
```

### 6.3 レビュー基準

**機能・ロジック**

- 要件・仕様どおりに動作するか
- エッジケース・エラーハンドリングが適切か
- パフォーマンスへの悪影響がないか

**コード品質**

- コーディング規約（`coding-standards.md`）に準拠しているか
- 重複コードがないか（DRY原則）
- 命名が明確で意図が伝わるか

**セキュリティ**

- XSS / SQLインジェクション等の脆弱性がないか
- 機密情報（APIキー・パスワード）がコードに含まれていないか
- 依存関係の脆弱性がないか

**テスト**

- 新機能・バグ修正にテストが追加されているか
- 既存テストが通過するか（CI確認）

**デザイン変更**

- デザイン意図（課題、狙い、比較差分）がPR本文に記載されているか
- Figma参照URLまたは比較画像（Before/After）が添付されているか
- トークン（色、タイポ、余白）の逸脱がないか
- 画像アセットが適切な形式とサイズで管理されているか（必要時LFS）

**ドキュメント変更**

- 対象読者と想定ユースケースが明示されているか
- 手順が再現可能で、前提条件と失敗時の対処が記載されているか
- 用語の定義と表記ゆれが統一されているか
- 関連リンクが有効で、章構成と目次の整合が取れているか

**レビューコメントの書式（Conventional Comments推奨）：**

```
# 種別: コメント本文
nit: typo in variable name `recieve` to `receive`
suggestion: extracting this logic to a helper function would improve readability
issue: this will throw if `user` is null; add null check
question: why is this set to 30 seconds specifically?
```

### 6.4 マージ方式の選択

| 方式 | 概要 | 推奨シナリオ |
|------|------|-------------|
| **Merge Commit** | PRの全コミットを保持し、マージコミットを作成 | 完全な履歴の保存が必要な場合 |
| **Squash and Merge** | PRの全コミットを1つに圧縮してマージ | 作業中の "WIP" コミットを整理したい場合 |
| **Rebase and Merge** | PRのコミットをmainの先頭に再適用（マージコミットなし） | 線形履歴を維持したい場合 |

**戦略別の必須方針：**

- GitHub Flow / TBD: `Squash and Merge` または `Rebase and Merge` のどちらかをリポジトリで固定し、混在させない。
- Gitflow（`feature/*` から `develop` への統合）: `Merge Commit`（`--no-ff`）を必須とする。
- Gitflow（`release/*` / `hotfix/*` から `main` への統合）: `Merge Commit`（`--no-ff`）を必須とする。
- Gitflow運用時に `Allow merge commits: false` を設定してはならない。

### 6.5 チーム規模別のレビュー要件

| 規模 | 必要承認数 | CODEOWNERS | 自動チェック |
|------|-----------|------------|-------------|
| Solo | 0（任意） | 不要 | CI必須 |
| Small | 1以上 | 任意 | CI必須 + CodeRabbit推奨 |
| Medium | 1名以上（通常は2名） | 推奨 | CI + Lint必須 + CodeRabbit準必須（未導入時は管理者承認理由を記録） |
| Large | 2以上 + CODEOWNERS | 必須 | CI + Lint + セキュリティスキャン + CodeRabbit必須 |

### 6.6 GitHub / GitLab 保護設定マトリクス（必須）

GitHubはRulesets（Repository rules）を優先し、旧Branch protection rulesは移行期間または互換運用時のみ使用する。GitLabはBranch rulesを基準に統制する。

| 項目 | GitHub | GitLab |
|------|--------|--------|
| ルール定義方式 | Rulesets（Repository rules）を優先 | Branch rulesを優先 |
| 直push禁止 | Branch protectionで `Restrict who can push` を有効化 | Protected branchesで `Allowed to push: No one` またはMaintainer限定 |
| PR/MR必須 | `Require a pull request before merging` | `Merge request approvals` を必須化 |
| 承認数 | `Required approvals` をチーム規模に応じて設定 | `Approvals required` を同等数で設定 |
| 自己承認・直近コミット者承認の抑止 | `Require approval of the most recent reviewable push` を有効化 | `Prevent approval by merge request author` と `Prevent approvals by users who add commits` を有効化 |
| オーナーレビュー | `Require review from CODEOWNERS` | `Code Owners approval` |
| CI成功必須 | `Require status checks to pass before merging` | `Pipelines must succeed` |
| デプロイ成功必須（本番環境がある場合） | `Require deployments to succeed before merging` | 保護環境の承認とデプロイジョブ成功を必須化 |
| 最新取り込み必須 | `Require branches to be up to date` | `Merge checks` でtarget branchとの整合を必須化 |
| 会話解決必須 | `Require conversation resolution` | `All threads must be resolved` |
| 署名コミット（高統制で必須） | `Require signed commits` | Push rulesの `Reject unsigned commits` |
| リリースタグ保護 | `Protected tags` で `v*` の作成・削除を制限 | `Protected tags` で `v*` の作成を制限し、削除権限を最小化 |
| バイパス禁止 | `Do not allow bypassing...` | 例外ロールを最小化し監査ログを保持 |
| AIレビュー必須 | Required checks にCodeRabbitのレビュー状態を含める | CodeRabbitレビュー結果が反映されるまでマージしない |

### 6.7 CodeRabbit AIレビュー運用（必須）

- CodeRabbitはPR/MR作成時の自動レビューを有効化する。
- 導入強度はチーム規模で段階化する（Small: 推奨、Medium: 準必須、Large: 必須）。
- AIレビューは人間レビューの代替ではない。`6.5` の承認要件は維持する。
- CodeRabbitの未解決コメントが残っている場合はマージしない。
- 追加コミット後は増分レビューを待つ。必要に応じて手動で再レビューを要求する。

```text
@coderabbitai review       # 増分レビュー
@coderabbitai full review  # 全体再レビュー
```

---

## 7. タグ・リリースノート

### この章の決定事項

- リリースタグは必ずAnnotated Tagを使用する。
- タグ名とリリースノートの日時はISO 8601準拠で記録する。
- `v*` のリリースタグは保護し、作成・削除権限を限定する。

### 7.1 タグ規約

**常にAnnotated Tagを使用する：**

```zsh
# リリースタグの作成
git tag -a v1.2.3 -m "Release version 1.2.3"

# 特定コミットへのタグ付け
git tag -a v1.2.3 9fceb02 -m "Release version 1.2.3"

# タグをリモートへプッシュ
git push origin v1.2.3

# 全タグをプッシュ
git push origin --tags

# タグの詳細確認
git show v1.2.3

# タグ一覧
git tag -l "v*" --sort=-version:refname
```

**タグ命名規則：**

```
v{MAJOR}.{MINOR}.{PATCH}
v{MAJOR}.{MINOR}.{PATCH}-{prerelease}
v{MAJOR}.{MINOR}.{PATCH}-{prerelease}+{build}

例:
v1.2.3
v1.2.3-rc.1
v2.0.0-beta.2+20260221T140000Z
```

**保護タグ設定（必須）：**

- GitHub: `Protected tags` で `v*` を対象に設定し、作成と削除をリリース管理者のみに限定する。
- GitLab: `Protected tags` で `v*` を対象に設定し、作成をMaintainer以上に限定する。削除権限は最小化する。

### 7.2 リリースノートの書式

```markdown
## v1.2.3 (2026-02-21T14:00:00+09:00)

### 新機能

- Apple Pay 決済に対応 (#789)
- ダッシュボードにリアルタイム通知を追加 (#801)

### バグ修正

- ダークモードでのボタン配置を修正 (#834)
- トークン期限切れ後のページリロード問題を解決 (#821)

### パフォーマンス改善

- 画像読み込みの遅延最適化 (#815)

### 廃止予定

- `/v1/users` エンドポイントは v2.0.0 で削除予定。`/v2/users` への移行を推奨。

### 破壊的変更

なし

### 変更の全リスト

[v1.1.0...v1.2.3](https://github.com/org/repo/compare/v1.1.0...v1.2.3)
```

---

## 8. CHANGELOG

### 8.1 Keep a Changelog 形式（keepachangelog.com）

**ファイル：** リポジトリルートに `CHANGELOG.md` を配置する。

```markdown
# Changelog

このファイルは [Keep a Changelog](https://keepachangelog.com/ja/1.1.0/) 形式に従い、
[Semantic Versioning](https://semver.org/lang/ja/) を採用している。

## [Unreleased]

### 追加

### 変更

### 廃止予定

### 削除

### 修正

### セキュリティ

---

## [1.2.3] - 2026-02-21

### 追加

- Apple Pay 決済に対応 ([#789](https://github.com/org/repo/pull/789))

### 修正

- ダークモードでのボタン配置を修正 ([#834](https://github.com/org/repo/pull/834))

---

## [1.2.2] - 2026-01-15

...

[Unreleased]: https://github.com/org/repo/compare/v1.2.3...HEAD
[1.2.3]: https://github.com/org/repo/compare/v1.2.2...v1.2.3
[1.2.2]: https://github.com/org/repo/compare/v1.2.1...v1.2.2
```

### 8.2 自動生成ツール

| ツール | 特性 | 適用場面 |
|--------|------|----------|
| `semantic-release` | 完全自動。Conventional CommitsからバージョンとCHANGELOGを生成 | CI/CDに組み込む場合 |
| `conventional-changelog` | CLIで実行。各種フォーマットに対応 | 手動または半自動 |
| `commit-and-tag-version` | `npm version` の代替。タグ・CHANGELOG・バージョンを一括更新 | Node.js プロジェクト |

```zsh
# conventional-changelogの使用例
npx conventional-changelog -p angular -i CHANGELOG.md -s

# commit-and-tag-versionの使用例
npx commit-and-tag-version
```

---

## 9. 補助設定ファイル

### この章のコード例記法

この章の設定例で実値を置換する箇所は、接頭辞 `YOUR_` のプレースホルダーを使用する。

| プレースホルダー | 説明 |
| --- | --- |
| `YOUR_ORG` | GitHub Organization名 |
| `YOUR_CORE_TEAM` | コアチームのGitHubチーム名 |
| `YOUR_SECURITY_TEAM` | セキュリティチームのGitHubチーム名 |
| `YOUR_DEVOPS_TEAM` | DevOpsチームのGitHubチーム名 |

ドメイン名の例には RFC 2606 予約ドメイン `example.com` を使用する。

### 9.1 `.gitignore`

```gitignore
# ==============================
# 環境・シークレット
# ==============================
.env
.env.local
.env.*.local
*.pem
*.key
*.p12
*.mobileprovision

# ==============================
# Node.js
# ==============================
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnp
.pnp.js

# ==============================
# ビルド成果物
# ==============================
dist/
build/
out/
.next/
.nuxt/
.vite/
coverage/
*.min.js
*.min.css

# ==============================
# OS・エディタ
# ==============================
.DS_Store
.DS_Store?
._*
Thumbs.db
*.swp
*.swo
*~
.idea/
.vscode/
!.vscode/extensions.json
!.vscode/settings.json

# ==============================
# iOS / Xcode
# ==============================
*.xcworkspace/xcuserdata/
*.xcodeproj/xcuserdata/
DerivedData/
*.ipa
Pods/

# ==============================
# Android
# ==============================
*.apk
*.aab
*.ap_
local.properties
.gradle/
build/
captures/

# ==============================
# ログ・一時ファイル
# ==============================
*.log
*.tmp
*.temp
*.bak
```

### 9.2 `.gitattributes`

```gitattributes
# 改行コードの統一
* text=auto
*.md text eol=lf
*.sh text eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf

# バイナリファイル
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.webp binary
*.bmp binary
*.tif binary
*.tiff binary
*.avif binary
*.heic binary
*.heif binary
*.raw binary
*.ttf binary
*.otf binary
*.woff binary
*.woff2 binary
*.pdf binary
*.doc binary
*.docx binary
*.xls binary
*.xlsx binary
*.ppt binary
*.pptx binary
*.mp3 binary
*.wav binary
*.aac binary
*.m4a binary
*.flac binary
*.ogg binary
*.mp4 binary
*.mov binary
*.webm binary
*.mkv binary
*.avi binary
*.wmv binary
*.m4v binary
*.zip binary
*.tar binary
*.gz binary
*.tgz binary
*.bz2 binary
*.xz binary
*.7z binary
*.rar binary
*.jar binary
*.war binary
*.class binary
*.wasm binary
*.exe binary
*.dll binary
*.so binary
*.dylib binary
*.bin binary
*.db binary
*.sqlite binary
*.sqlite3 binary

# Git LFS（大容量ファイル）
*.psd filter=lfs diff=lfs merge=lfs -text
*.sketch filter=lfs diff=lfs merge=lfs -text
*.fig filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text

# Diff設定
*.md diff=markdown
```

### 9.3 pre-commit フック設定（`.pre-commit-config.yaml`）

```yaml
# pre-commit インストール: pip install pre-commit && pre-commit install
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace         # 末尾空白の除去
      - id: end-of-file-fixer           # ファイル末尾に改行を付与
      - id: check-merge-conflict        # マージコンフリクトマーカーの検出
      - id: check-added-large-files     # 大容量ファイルのコミット防止（デフォルト500KB）
        args: ['--maxkb=1024']
      - id: detect-private-key          # 秘密鍵の誤コミット防止
      - id: check-json                  # JSONの構文チェック
      - id: check-yaml                  # YAMLの構文チェック
      - id: no-commit-to-branch         # 保護ブランチへの直接コミット防止
        args: ['--branch', 'main', '--branch', 'develop']

  # Node.js / TypeScript（プロジェクトに応じて調整）
  - repo: local
    hooks:
      - id: commitlint
        name: Commitlint
        entry: npx --no -- commitlint --edit
        language: system
        stages: [commit-msg]
        pass_filenames: true
      - id: eslint
        name: ESLint
        entry: npx eslint --fix
        language: node
        files: \.(js|ts|jsx|tsx)$
        pass_filenames: true
      - id: prettier
        name: Prettier
        entry: npx prettier --write
        language: node
        files: \.(js|ts|jsx|tsx|json|css|md)$
        pass_filenames: true
```

### 9.4 commit-msg 検証（commitlint推奨）

**必須方針：**

- commit messageの検証は `commitlint` を標準とする。
- 手書きシェルフックは、暫定対応またはNode導入不可の環境でのみ許可する。
- いずれの方式でも `1行目（subject）のみ` を正規表現で検証する。

**`commitlint` 設定例（`.commitlintrc.cjs`）：**

```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'header-max-length': [2, 'always', 72],
    'subject-empty': [2, 'never'],
    'subject-full-stop': [2, 'never', '.'],
  },
};
```

```zsh
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

**手書きフック例（参考）：**

```zsh
#!/bin/sh
commit_msg_file="$1"
first_line="$(sed -n '1p' "$commit_msg_file")"

# マージコミットは除外
if echo "$first_line" | grep -qE '^(Merge|Revert)\b'; then
  exit 0
fi

# type(scope)!: description （subject最大72文字）
pattern='^(feat|fix|docs|style|refactor|test|build|ci|chore|perf|revert)(\([a-z0-9/_-]+\))?(!)?: [^ ].{0,71}$'

if ! echo "$first_line" | grep -qE "$pattern"; then
  echo "Conventional Commits形式エラー: <type>[scope]: <description>"
  exit 1
fi
```

### 9.5 CI/CD設定（戦略別）

**`.github/workflows/ci.yml`（GitHub Flow / TBD）：**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  merge_group:
    branches: [main] # Merge Queueを利用する場合
```

**`.github/workflows/ci.yml`（Gitflow）：**

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  merge_group:
    branches: [main, develop] # Merge Queueを利用する場合
```

> `jobs` セクションは同一でよい。違いはトリガーブランチと、Merge Queue利用時の `merge_group` 有無のみ。

**`.gitlab-ci.yml`（GitLab、戦略別の対象ブランチ）：**

```yaml
workflow:
  rules:
    # GitHub Flow / TBD
    - if: '$CI_COMMIT_BRANCH == "main"'
    # Gitflow
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

**`.github/workflows/release.yml`（タグプッシュでリリース）:**

```yaml
name: Release

on:
  push:
    tags:
      - 'v*.*.*'

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Get version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          tag_name: ${{ github.ref_name }}
          name: "Release v${{ steps.version.outputs.VERSION }}"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 9.6 CODEOWNERS（`CODEOWNERS`）

```
# リポジトリルートまたは .github/CODEOWNERS に配置

# デフォルトオーナー（全ファイル）
*                   @YOUR_ORG/YOUR_CORE_TEAM

# 認証関連
/src/auth/          @YOUR_ORG/YOUR_SECURITY_TEAM @YOUR_ORG/YOUR_CORE_TEAM

# インフラ・CI設定
/.github/           @YOUR_ORG/YOUR_DEVOPS_TEAM
/infra/             @YOUR_ORG/YOUR_DEVOPS_TEAM

# ドキュメント
/docs/              @YOUR_ORG/YOUR_CORE_TEAM

# パッケージ管理
package.json        @YOUR_ORG/YOUR_CORE_TEAM
package-lock.json   @YOUR_ORG/YOUR_CORE_TEAM
```

### 9.7 ブランチ保護設定プロファイル（GitHub / GitLab）

GitHubはRulesets（Repository rules）で実装することを優先し、以下は設定値の基準として扱う。

**GitHub Flow / TBD プロファイル：**

```yaml
# Settings > General > Pull Requests
Allow merge commits: false
Allow squash merging: true
Allow rebase merging: true
Automatically delete head branches: true

# Settings > Branches > Branch protection rules (main)
Require a pull request before merging: true
Required approvals: 1  # Soloは任意で0可
Require review from CODEOWNERS: true  # Medium以上
Require approval of the most recent reviewable push: true
  Require status checks to pass before merging: true
  Required checks:
    - lint-and-test
    - <coderabbit_check_name>  # 実際のチェック名に合わせて設定
Require deployments to succeed before merging: true  # 本番環境がある場合
Require signed commits: true  # Largeまたは高統制チーム
Require branches to be up to date: true
Require conversation resolution: true
Do not allow bypassing the above settings: true
Restrict who can push to matching branches: true
```

**Gitflow プロファイル：**

```yaml
# Settings > General > Pull Requests
Allow merge commits: true
Allow squash merging: false
Allow rebase merging: false
Automatically delete head branches: true

# Branch protection rules (main, develop)
Require a pull request before merging: true
Required approvals: 1-2
Require approval of the most recent reviewable push: true
Require status checks to pass before merging: true
  Required checks:
    - lint-and-test
    - <coderabbit_check_name>  # 実際のチェック名に合わせて設定
Require deployments to succeed before merging: true  # 本番環境がある場合
Require signed commits: true  # Largeまたは高統制チーム
Require branches to be up to date: true
Require conversation resolution: true
```

**GitLab 設定対応（同等要件）：**

| 要件 | GitLab設定の例 |
|------|----------------|
| マージ要求の強制 | `Merge request approvals` を有効化 |
| 承認数 | `Approvals required` をチーム規模に合わせる |
| 直push禁止 | `Protected branches` で `Allowed to push` を制限 |
| CI成功必須 | `Pipelines must succeed` を有効化 |
| 自己承認防止 | `Prevent approval by merge request author` を有効化 |
| コミット追加者承認防止 | `Prevent approvals by users who add commits` を有効化 |
| 会話解決 | `All threads must be resolved` を有効化 |
| 署名コミット | Push rules の `Reject unsigned commits` を有効化 |
| タグ保護 | `Protected tags` で `v*` の作成・削除権限を制限 |
| 履歴方針（Flow/TBD） | `Merge method: Fast-forward merge` または `Semi-linear history` |
| 履歴方針（Gitflow） | `Merge method: Merge commit` |

### 9.8 Issueテンプレート（Bug Report / Feature Request）

**運用方針（必須）：**

- 不具合報告は `bug_report` テンプレートを必須とする。
- 機能追加要望は `feature_request` テンプレートを必須とする。
- 再現手順または受け入れ条件が未記入のIssueは受付しない。

**GitHub（`.github/ISSUE_TEMPLATE/bug_report.md`）：**

```markdown
---
name: Bug report
about: 不具合を報告する
title: "[Bug] "
labels: ["bug", "triage"]
assignees: []
---

## 概要

<!-- 何が起きているかを一文で記載 -->

## 発生環境

- App/Service:
- Version/Commit:
- OS:
- Browser/Runtime:

## 再現手順

1.
2.
3.

## 期待される動作

<!-- 本来どうなるべきか -->

## 実際の動作

<!-- 実際に何が起きたか -->

## 影響範囲

- [ ] 本番影響あり
- [ ] 回避手段なし
- [ ] データ欠損の可能性あり

## 追加情報

<!-- ログ スクリーンショット 関連Issue/PR -->
```

**GitHub（`.github/ISSUE_TEMPLATE/feature_request.md`）：**

```markdown
---
name: Feature request
about: 機能追加を提案する
title: "[Feature] "
labels: ["enhancement", "triage"]
assignees: []
---

## 背景 / 課題

<!-- 現状の課題や困りごと -->

## 提案内容

<!-- 追加・変更したい機能 -->

## 受け入れ条件（Acceptance Criteria）

- [ ]
- [ ]
- [ ]

## 代替案

<!-- 検討した他案があれば記載 -->

## 影響範囲

- [ ] API変更あり
- [ ] DB変更あり
- [ ] UI変更あり
- [ ] ドキュメント更新が必要

## 補足

<!-- モック 参考リンク 関連Issue/PR -->
```

**GitHub（`.github/ISSUE_TEMPLATE/config.yml`）：**

```yaml
blank_issues_enabled: false
contact_links:
  - name: Security Report
    url: https://example.com/security
    about: 機密性の高い報告は公開Issueではなく専用窓口で受け付ける
```

> `Issue Forms` を使う場合も、同じディレクトリで `config.yml` を管理する。

**GitLab（`.gitlab/issue_templates/Bug Report.md`）：**

```markdown
## 概要

## 発生環境
- App/Service:
- Version/Commit:
- OS:
- Browser/Runtime:

## 再現手順
1.
2.
3.

## 期待される動作

## 実際の動作

## 影響範囲
- [ ] 本番影響あり
- [ ] 回避手段なし
- [ ] データ欠損の可能性あり

## 追加情報
```

**GitLab（`.gitlab/issue_templates/Feature Request.md`）：**

```markdown
## 背景 / 課題

## 提案内容

## 受け入れ条件（Acceptance Criteria）
- [ ]
- [ ]
- [ ]

## 代替案

## 影響範囲
- [ ] API変更あり
- [ ] DB変更あり
- [ ] UI変更あり
- [ ] ドキュメント更新が必要

## 補足
```

### 9.9 CodeRabbit組み込み設定（AIコードレビュー）

**GitHub導入手順（必須）：**

1. CodeRabbit GitHub Appをインストールし、対象リポジトリへアクセスを許可する。
2. PRを1件作成して、CodeRabbitレビューが自動投稿されることを確認する。
3. `9.7` の保護設定でCodeRabbitチェックを必須にする。

**GitLab導入手順（必須）：**

1. CodeRabbitで対象リポジトリをInstallする。
2. GitLabトークン（PATまたはGroup Access Token）を設定する。
3. Webhook `https://coderabbit.ai/gitlabHandler` が設定されることを確認する。

**`.coderabbit.yaml`（推奨最小構成）：**

```yaml
# yaml-language-server: $schema=https://coderabbit.ai/integrations/schema.v2.json
language: "ja-JP"
early_access: false
reviews:
  profile: "chill"
  request_changes_workflow: false
  high_level_summary: true
  review_status: true
  auto_review:
    enabled: true
    drafts: false
    auto_incremental_review: true
    base_branches:
      - main
chat:
  auto_reply: true
```

**`.coderabbit.yaml` 設定項目解説（推奨値の理由）：**

| 項目 | 目的 | 推奨値 | 公式参照 |
|------|------|--------|----------|
| `language` | レビューコメントの言語を統一する | `"ja-JP"` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `early_access` | 先行機能の利用有無を制御する | `false`（安定優先） | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.profile` | レビューの厳しさ・粒度を調整する | `"chill"`（運用開始時の過検知抑制） | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.request_changes_workflow` | Request changes主体運用にするかを制御する | `false`（人間レビュー主導） | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.high_level_summary` | PR全体の要約を出力する | `true` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.review_status` | レビュー状態をステータスへ反映する | `true` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.auto_review.enabled` | PR作成時の自動レビューを有効化する | `true` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.auto_review.drafts` | Draft PRも自動レビュー対象にするかを制御する | `false`（ノイズ抑制） | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.auto_review.auto_incremental_review` | 追コミット時に増分レビューを実行する | `true` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `reviews.auto_review.base_branches` | 自動レビュー対象の基準ブランチを固定する | 戦略別に `main` または `main, develop` | [YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) |
| `chat.auto_reply` | `@coderabbitai` メンションへの自動応答を有効化する | `true` | [Review Commands](https://docs.coderabbit.ai/guides/commands/) |

**戦略別`base_branches`：**

- GitHub Flow / TBD: `["main"]`
- Gitflow: `["main", "develop"]`

---

## 10. ユースケース別リファレンス

### 10.1 モバイルアプリ（iOS / Android）

**推奨：Gitflow**

モバイルアプリはApp Store / Google Playの審査サイクルがあるため、Gitflowが最適。

```
バージョン管理:
  versionName（ユーザー向け）: SemVer準拠 (1.2.3)
  versionCode / CFBundleVersion（Store向け）: 常に増加する整数

リリースサイクル:
  developからrelease/x.x.xを作成し、審査提出後に承認されたらmainへマージし、タグを作成
```

**iOS 固有：**

```zsh
# Info.plistの更新
/usr/libexec/PlistBuddy -c "Set :CFBundleShortVersionString 1.2.3" Info.plist
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion 456" Info.plist
```

**Android 固有：**

```groovy
// app/build.gradle
android {
    defaultConfig {
        versionCode 456         // ビルド番号（常に増加）
        versionName "1.2.3"     // SemVer
    }
}
```

**ホットフィックスの扱い（App Store審査後）：**

```zsh
# 審査済みタグからホットフィックスブランチを作成
git checkout -b hotfix/1.2.4 v1.2.3
# 修正・テスト後に再審査提出
git tag -a v1.2.4 -m "Hotfix 1.2.4: fix crash on startup"
```

---

### 10.2 npmパッケージ / ライブラリ

**推奨：Gitflow**

```zsh
# バージョンアップ手順
npm version patch   # 1.2.3 から 1.2.4
npm version minor   # 1.2.3 から 1.3.0
npm version major   # 1.2.3 から 2.0.0

# publish
npm publish
git push origin main --tags
```

**`package.json` でのビルド自動化：**

```json
{
  "scripts": {
    "version": "conventional-changelog -p angular -i CHANGELOG.md -s && git add CHANGELOG.md",
    "postversion": "git push && git push --tags"
  }
}
```

---

### 10.3 モノレポ（Monorepo）

**推奨：TBD または GitHub Flow + パッケージ別バージョニング**

```
monorepo/
├── packages/
│   ├── core/          # @myorg/core
│   ├── ui/            # @myorg/ui
│   └── cli/           # @myorg/cli
└── apps/
    ├── web/
    └── mobile/
```

**バージョニングツール：**

- `changesets`：パッケージ単位でバージョンと変更履歴を管理（推奨）
- `nx`：タスクランナー + バージョニング自動化

---

### 10.4 チームが拡大した際のスケーリング

**SoloからSmall（2名以上5名以下へ移行時）：**

```
変更:
  - PRとコードレビューを導入
  - CIを必須化（GitHub Actions等）
  - mainへの直接プッシュを禁止
  - CONTRIBUTING.mdを整備
```

**SmallからMedium（6名以上20名以下へ移行時）：**

```
変更:
  - CODEOWNERSを設定
  - ブランチ保護ルールを強化（承認数2以上）
  - CodeRabbit導入を準必須化（未導入時は例外理由を記録）
  - Linter / Formatterの統一（pre-commitフック導入）
  - PRテンプレートを整備
  - 戦略の見直し（GitHub FlowからGitflowへの移行が必要か検討）
```

**MediumからLarge（21名以上へ移行時）：**

```
変更:
  - main / develop の保護ルールを分離し、例外権限を最小化
  - 必須レビューを「2名以上 + CODEOWNERS承認」に固定
  - 必須チェックにAIレビュー（CodeRabbit）とセキュリティスキャンを厳密必須化
  - リリース承認フローを明文化（担当、期限、ロールバック責任者）
  - 週次で運用指標を監視（レビュー滞留時間、差し戻し率、障害件数）
```

### 10.4.1 ユーザー管理（追加 / 変更 / 削除）の基本概念

**運用原則（必須）：**

- 最小権限: 必要最小限の権限のみ付与する。
- ロールベース: 個別ユーザーではなく役割（Role）単位で権限を管理する。
- 職務分離: 権限申請者と承認者を分離する。
- 追跡可能性: 追加・変更・削除はすべてチケットと監査ログを残す。
- 人間ユーザーとサービスユーザー（Bot/App）を分離して管理する。

**JML（Joiner / Mover / Leaver）運用：**

| 区分 | トリガー | 必須アクション | 期限（SLA） |
|------|----------|----------------|-------------|
| Joiner（追加） | 新規参加 | 所属チームへ追加、初期ロール付与、2FA必須化 | 1営業日以内 |
| Mover（変更） | 役割変更/異動 | 旧ロール剥奪、新ロール付与、CODEOWNERS更新確認 | 当日中 |
| Leaver（削除） | 退職/離任 | 組織・グループから削除、トークン失効、担当引き継ぎ | 即時（遅くとも24時間以内） |

**AIコードレビュー用ユーザー（Bot/App）管理：**

| 区分 | 対象 | 必須アクション |
|------|------|----------------|
| 追加 | CodeRabbit等のAIレビューApp導入 | 対象リポジトリを限定してInstallし、必要最小権限のみ付与 |
| 変更 | 対象リポジトリ追加/削除、権限調整 | 変更チケットを作成し、管理者承認後に実施。`6.7` と `9.9` の設定整合を確認 |
| 削除 | AIレビュー停止、契約終了、インシデント対応 | App連携を解除し、Webhook/トークンを失効。保護設定の必須チェックを更新 |

**Bot/App運用ルール（必須）：**

- Personal Access Tokenを恒久運用しない。可能な限りGitHub App/GitLab App連携を使う。
- Appのオーナー、用途、対象リポジトリ、更新日時を台帳で管理する。
- 四半期ごとにBot/App棚卸しを実施し、未使用連携を削除する。

**ロール設計の基準（GitHub / GitLab）：**

| レベル | GitHub例 | GitLab例 | 用途 |
|--------|----------|----------|------|
| 閲覧 | Read（閲覧のみ） / Triage（課題整理） | Guest（閲覧中心） / Reporter（課題整理） | 閲覧・課題整理 |
| 開発 | Write | Developer | ブランチ作成、PR/MR作成 |
| 保守 | Maintain | Maintainer | 保護設定、リリース運用 |
| 管理 | Admin | Owner | 請求・統制・監査設定 |

**権限変更フロー（必須）：**

1. 権限申請をIssueまたはチケットで起票する（理由、期間、対象リポジトリを明記）。
2. 直属責任者とリポジトリ管理者の承認を取得する。
3. 権限付与/変更/削除を実施し、実施ログ（日時、実施者、差分）を記録する。
4. 月次または四半期ごとにアクセス棚卸しを実施し、不要権限を削除する。

---

### 10.5 緊急対応（Hotfix）フロー

**GitHub Flow の場合：**

```zsh
git checkout main
git pull origin main
git checkout -b hotfix/critical-security-fix
# 修正
git commit -m "fix(security): patch SQL injection vulnerability"
# 即PRを作成・承認・マージ・デプロイ
```

**Gitflow の場合：**

```zsh
git checkout -b hotfix/2.0.1 v2.0.0
# 修正
git commit -m "fix(auth): prevent unauthorized token reuse"
git checkout main
git merge --no-ff hotfix/2.0.1
git tag -a v2.0.1 -m "Hotfix 2.0.1: critical auth fix"
git checkout develop
git merge --no-ff hotfix/2.0.1
git push origin main develop v2.0.1
git branch -d hotfix/2.0.1
```

---

### 10.6 デザイン成果物フロー（UI / グラフィック）

**推奨：GitHub Flow + `design/*` / `asset/*` ブランチ**

```text
Figma更新
  - designチケット起票（目的・変更範囲・対象画面）
  - design/* ブランチ作成
  - アセット出力（必要時LFS管理）
  - PR作成（Figmaリンク・比較画像添付）
  - デザインレビュー + 実装レビュー
  - mainへマージ
```

**ブランチ命名（必須）：**

```text
design/[issue-id]-short-description
asset/[issue-id]-short-description
```

**PR必須添付（必須）：**

- FigmaファイルURL（対象ノードを含む）
- Before/After比較画像
- 影響範囲（画面、コンポーネント、トークン）
- アセット一覧（追加/更新/削除）

**アセット管理ルール（必須）：**

- 画像・動画・デザインソースの大容量ファイルはGit LFSを使用する。
- 同名上書きではなく、PR本文に差分理由を記載する。
- エクスポート設定（倍率、背景、圧縮方式）をテンプレート化して再利用する。

### 10.7 ドキュメントフロー（仕様 / 運用 / ナレッジ）

**推奨：GitHub Flow + `docs/*` ブランチ**

```text
課題起票
  - docs/* ブランチ作成
  - 章構成を先に確定
  - 本文作成（手順、前提、例外）
  - 自己レビュー（リンク、目次、表記統一）
  - PR作成
  - レビュー承認後にmainへマージ
```

**ブランチ命名（必須）：**

```text
docs/[issue-id]-short-description
```

**PRチェック項目（必須）：**

- 対象読者、目的、更新理由
- 変更章一覧（追加・更新・削除）
- 破壊的変更の有無（手順変更、非推奨化）
- 関連Issue/関連仕様へのリンク
- 文書フォーマットがMarkdown（`.md`）で統一されていること
- 改行コードがLFであること（`9.2 .gitattributes` に準拠）

**文書リリース運用（必須）：**

- 手順変更を含む場合、`CHANGELOG.md` の `Unreleased` に要点を追記する。
- 参照先が外部ドキュメントの場合、公開日・更新日を確認して古い情報を避ける。
- 重大な運用変更は、マージ後に通知（Slack/Teams等）を実施する。

---

## 11. Rebase / Force-push 規律

履歴改変は監査性を損なうため、対象ブランチを限定して運用する。

### 11.1 許可・禁止ルール（必須）

| ブランチ | Rebase | Force-push | 条件 |
|----------|--------|------------|------|
| `main` | 禁止 | 禁止 | 例外なし |
| `develop` | 禁止 | 禁止 | 例外なし（Gitflow） |
| `release/*` | 禁止 | 禁止 | 例外なし |
| `hotfix/*` | 禁止 | 禁止 | 例外なし |
| `feature/*` / `fix/*` / `docs/*` / `chore/*` / `refactor/*` | 許可 | 条件付き許可 | PR作成前、またはPRレビュー担当者の承認後のみ |

### 11.2 例外承認フロー（必須）

1. 実行者は「対象ブランチ」「対象コミット範囲」「理由」をPRまたはチケットへ記録する。
2. リポジトリ管理者またはテックリード1名以上の承認を取得する。
3. 実行後に新しいHEAD SHAを通知し、再レビューを依頼する。

### 11.3 履歴改変後の通知義務（必須）

```text
[History Rewritten]
Branch: feature/123-add-user-auth
Old HEAD: abc1234
New HEAD: def5678
Reason: squash before final review
Action Required: re-fetch and reset local branch
```

---

## 12. Rollback / Incident Runbook

障害時は「復旧優先」で対応し、原因分析は復旧後に実施する。

### 12.1 タグ単位ロールバック（最優先）

```zsh
# 現行タグ: v1.4.2 / 復旧先: v1.4.1 の例
git checkout main
git pull origin main
git checkout -b rollback/v1.4.1
git revert --no-edit v1.4.1..v1.4.2
git push origin rollback/v1.4.1
# PRを作成し、緊急承認後にmainへマージ
```

### 12.2 緊急revert（単一変更）

```zsh
git checkout -b revert/incident-20260221
git revert <bad_commit_sha>
git push origin revert/incident-20260221
```

### 12.3 再リリース条件（必須）

- 失敗原因の再現手順が文書化されている。
- 回避策ではなく恒久修正のテストがCIで成功している。
- CHANGELOGとリリースノートに障害番号と対処内容を記載している。
- セキュリティ影響がある場合は失効・再発行が完了している。

### 12.4 記録テンプレート（必須）

```markdown
## Incident Record
- 発生日時 (ISO 8601):
- 影響範囲:
- 検知方法:
- 一次対応:
- ロールバック有無:
- 恒久対応:
- 再発防止策:
- 担当者:
```

---

## 13. リリース判定ゲート

タグ作成前に、以下の全項目を満たさなければならない。

| 項目 | 判定基準 | 必須 |
|------|----------|------|
| CI | 対象ブランチの必須ジョブが全て成功 | Yes |
| テスト | 変更範囲に対応する自動テストが追加済み、または不要理由をPRに記載 | Yes |
| 変更履歴 | `CHANGELOG.md` の `Unreleased` が対象版へ反映済み | Yes |
| バージョン | `SemVer` / Build番号が規約に一致 | Yes |
| デプロイ | 本番環境がある場合は必要環境へのデプロイが成功 | Yes |
| セキュリティ | 重大脆弱性（Critical/High）が未解決でない | Yes |
| 承認 | 必要承認数とCODEOWNERS承認を満たす | Yes |

**実行例（GitHub）：**

```zsh
# 必須チェックがすべてsuccessであることを確認
gh pr checks <PR_NUMBER>
```

**実行例（GitLab）：**

```zsh
# MRのパイプライン成功と承認状況を確認
glab mr view <MR_IID>
```

---

## 14. 機密情報漏えい時のGit対応

### 14.1 初動（15分以内）

1. 漏えいしたトークン、鍵、パスワードを直ちに失効する。
2. 該当ブランチのマージを停止し、影響範囲を特定する。
3. 事故チケットを作成し、時刻・対象・担当者を記録する。

### 14.2 履歴対処（必要時）

```zsh
# 例: 誤ってコミットされた .env を履歴から削除
git filter-repo --path .env --invert-paths
git push --force --all
git push --force --tags
```

> 履歴改変前に、全開発者へ作業停止と再同期手順を通知すること。

### 14.3 再発防止（必須）

- Secret scanning（GitHub Advanced Security / GitLab Secret Detection）を有効化する。
- GitHubはSecret scanningのPush protectionを有効化し、検知時は例外解除を管理者承認制にする。
- `detect-private-key`、`gitleaks` 等をCIに追加する。
- 機密情報を `.env` とシークレットストアへ分離し、リポジトリには保存しない。
- 事故の事後レビューを実施し、チェックリストへ反映する。

---

## 出典・参考情報
- [Conventional Commits v1.0.0](https://www.conventionalcommits.org/ja/v1.0.0/) - コミットメッセージの構造化規約
- [Semantic Versioning 2.0.0](https://semver.org/) - バージョニング仕様
- [Keep a Changelog 1.1.0](https://keepachangelog.com/ja/1.1.0/) - CHANGELOG形式
- [Atlassian: Comparing Git Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows) - ブランチ戦略の比較
- [Trunk Based Development](https://trunkbaseddevelopment.com/) - TBDの公式ガイド
- [GitHub Docs: About pull request merges](https://docs.github.com/articles/about-pull-request-merges) - マージ方式の公式解説
- [Atlassian: Merging vs. Rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing) - マージとリベースの比較
- [Google Engineering Practices: Code Review](https://google.github.io/eng-practices/review/) - コードレビューガイド
- [pre-commit](https://pre-commit.com/) - Gitフックフレームワーク
- [Runway: Mobile Branching Strategy](https://www.runway.team/blog/choosing-the-right-branching-strategy-for-mobile-development) - モバイルアプリのブランチ戦略
- [Infinum Mobile Handbook: Git](https://infinum.com/handbook/devproc/git/git-for-mobile-platforms) - モバイル開発のGit運用
- [GitVersion Docs](https://gitversion.net/docs/reference/version-increments) - Build番号の自動導出
- [GitHub Actions: Variables](https://docs.github.com/en/actions/learn-github-actions/variables) - `GITHUB_RUN_NUMBER` 等のCI変数
- [GitLab CI/CD predefined variables](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html) - `CI_PIPELINE_IID` 等のCI変数
- [commitlint](https://commitlint.js.org/) - Conventional Commits検証ツール
- [GitHub Docs: About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-repository-settings/managing-rulesets/about-rulesets) - Rulesetsの基本
- [GitHub Docs: Available rules for rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-repository-settings/managing-rulesets/available-rules) - Rulesetsで利用可能な制約
- [GitHub Docs: About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) - ブランチ保護設定
- [GitHub Docs: Managing a merge queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) - Merge Queue運用
- [GitHub Docs: Events that trigger workflows](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#merge_group) - `merge_group` トリガー
- [GitHub Docs: Configuring protected tags](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/configuring-protected-tags) - タグ保護設定
- [GitLab Docs: Branch rules](https://docs.gitlab.com/user/project/repository/branches/branch_rules/) - Branch rules
- [GitLab Docs: Protected branches](https://docs.gitlab.com/user/project/repository/branches/protected/) - ブランチ保護設定
- [GitLab Docs: Merge request approvals](https://docs.gitlab.com/user/project/merge_requests/approvals/) - MR承認ルール
- [GitLab Docs: Protected tags](https://docs.gitlab.com/user/project/protected_tags/) - タグ保護設定
- [GitHub Docs: Configuring issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository) - Issueテンプレート管理
- [GitHub Docs: Syntax for issue forms](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms) - Issue Forms仕様
- [GitHub Docs: About push protection](https://docs.github.com/en/code-security/secret-scanning/introduction/about-push-protection) - シークレットのPush Protection
- [Git LFS](https://git-lfs.com/) - 大容量バイナリアセット管理
- [Figma Dev Mode](https://help.figma.com/hc/en-us/articles/1500004415582-Guide-to-Dev-Mode) - デザインから実装への受け渡し
- [git-filter-repo](https://github.com/newren/git-filter-repo) - 履歴から機密情報を除去するツール
- [Gitleaks](https://github.com/gitleaks/gitleaks) - シークレット検知ツール
- [CodeRabbit: GitHub Integration](https://docs.coderabbit.ai/platforms/github-com/) - GitHub導入手順
- [CodeRabbit: GitLab Integration](https://docs.coderabbit.ai/platforms/gitlab-com) - GitLab導入手順
- [CodeRabbit: YAML Configuration](https://docs.coderabbit.ai/getting-started/yaml-configuration) - `.coderabbit.yaml` 設定
- [CodeRabbit: Review Commands](https://docs.coderabbit.ai/guides/commands/) - `@coderabbitai` コマンド
