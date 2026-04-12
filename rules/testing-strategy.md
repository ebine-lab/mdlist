---
title: "Testing Strategy"
description: "テスト戦略 - 単体/統合/E2Eテスト規約・カバレッジ基準・モック方針 / Testing strategy - unit/integration/E2E conventions, coverage criteria, mocking policy"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-21T22:13+09:00"
lang: "ja"
---

# Testing Strategy

**説明** - フロントエンド開発におけるテスト設計、実装、運用、品質判定の基準を定義する。


## 目的

本規定は、変更に強く、回帰を早期に検出できるテスト運用を標準化することを目的とする。  
単体テスト、統合テスト、E2Eテストの責務を分離し、テストの実行速度、安定性、保守性を同時に満たす。

## 背景

フロントエンド実装では、UI変更、状態管理、API連携、ブラウザー依存の相互作用が同時に発生する。  
単一レベルのテストでは検出漏れが発生しやすいため、複数レベルを役割分担して運用する必要がある。  
本規定は Playwright、Cypress、Testing Library、Jest、Vitest の公式指針を基礎に策定する。

出典・参考情報: Playwright Best Practices、Cypress Best Practices、Testing Library Guiding Principles、Jest Configuration、Vitest Config Reference。

## 対象

- Webフロントエンド（TypeScript/JavaScript）
- UIコンポーネント
- 画面遷移とフォーム操作
- API連携を含むユーザーフロー
- CI上で実行される自動テスト

## 非対象

- バックエンドAPI仕様そのものの妥当性検証
- 負荷試験、耐久試験の詳細設計
- セキュリティ侵入試験の詳細手順

## 本文

### サービスレベル目標と判定原則

本規定における SLA（Service Level Agreement - サービス品質保証）は、提供者が利用者に対して合意したサービス品質水準を指す。  
本規定のテスト判定は、機能正当性に加えて SLA の達成可能性を評価対象とする。  
ここでいうサービスレベル目標は、少なくとも次を含む。

1. 可用性目標。
2. 主要機能の応答時間目標。
3. 障害検知と復旧時間目標。
4. セキュリティインシデント回避の運用品質目標。

テスト設計、CI品質ゲート、例外運用、リリース判定はすべて SLA に整合していることを必須とする。

#### SLI/SLO閾値（必須）

1. 可用性 SLI は、対象期間内の成功リクエスト数を全対象リクエスト数で除した値とする。
2. 可用性 SLO は30日ローリングで 99.9%以上とする。
3. エラーバジェットは `100% - SLO` とし、可用性SLO 99.9% の場合は 0.1% を月次上限とする。
4. エラーバジェット消費率が100%に到達した場合は、新規機能リリースを停止し、信頼性回復を優先する。
5. フロントエンド性能 SLI は LCP、INP、CLS とし、集計は28日ローリングの75パーセンタイルで判定する。
6. フロントエンド性能 SLO は、LCP 2.5秒以下、INP 200ミリ秒以下、CLS 0.1以下とする。
7. 障害対応の運用目標は、MTTD 15分以内、MTTR 60分以内とする。
8. 上記 SLO のいずれかが未達の場合、未達要因が解消されるまでリリース判定を保留する。

出典・参考情報: SRE Book - Service Level Objectives、SRE Workbook - Embracing Risk、SRE Book - Monitoring Distributed Systems、ISO/IEC 20000-1:2018、Atlassian - SLA（Service Level Agreement）、Web Vitals。

### テスト設計原則

1. 利用者視点の検証を優先する。  
   実装の内部構造より、画面上で観測可能な挙動を基準にする。
2. テストを相互に独立させる。  
   前テストの副作用に依存するシナリオを禁止する。
3. 固定待機時間に依存しない。  
   明示的理由のない `sleep` 相当の待機を禁止し、イベントや状態変化を条件に待機する。
4. 外部要因を制御する。  
   第三者サービスや不安定ネットワークへの依存は、目的に応じてモックまたはスタブ化する。
5. SLA への影響を観測できるテストを含める。  
   応答時間、可用性、復旧可能性に影響する変更は、対応する検証を必須化する。

出典・参考情報: Testing Library Guiding Principles、Playwright Best Practices、Cypress Best Practices。

### テスト価値基準（4観点）

本規定のテストは、次の4観点を同時に満たすことを品質要件とする。

1. 退行保護: 変更後の破壊を早期に検出できること。
2. リファクタリング耐性: 内部実装の変更で不必要に壊れないこと。
3. 迅速なフィードバック: 開発ループ内で実行可能な時間で結果が返ること。
4. 保守性: 失敗時に原因特定と修正が容易であること。

出典・参考情報: JavaScript Testing Best Practices、JavaScriptのテストに関するベストプラクティス（Zenn）、フロントエンドテストにおける知見の宝庫（Zenn）。

### 開発フェーズ別テスト投資配分

テスト配分は固定比率で運用せず、プロダクトのフェーズに合わせて調整する。

| フェーズ | 単体 | 統合 | E2E | 補足 |
| --- | --- | --- | --- | --- |
| 立ち上げ期 | 高 | 中 | 低 | 主要導線の smoke のみ E2E 化し、仕様変動の大きい領域は過剰自動化しない。 |
| 拡張期 | 中 | 高 | 中 | 機能追加の主戦場を統合テストに置き、利用価値単位で検証する。 |
| 安定運用期 | 中 | 中 | 高 | 主要導線 E2E とビジュアル回帰で退行検出力を高める。 |

#### フェーズ移行判定基準

1. フェーズ判定は四半期ごとに実施し、判定根拠を記録する。
2. 立ち上げ期から拡張期への移行条件は、主要導線の smoke E2E が全件自動化され、2スプリント連続で重大障害の流出がないこととする。
3. 拡張期から安定運用期への移行条件は、required checks の失敗率が直近30日で 5% 未満、かつ不安定テスト件数の30日移動平均が直近3か月連続で前月比減少していることとする。
4. 判定条件を満たさない場合は、次期も同一フェーズを継続し、過不足のあるテストレベルを補強する。

出典・参考情報: FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）、cypressで始めるE2Eテスト（Qiita）。

### テストレベルと責務

#### 単体テスト

- 対象: 純粋関数、フォーマッター、バリデーター、カスタムフックの分岐ロジック。
- 目的: ロジック単位で仕様逸脱を最短で検出する。
- 原則: 外部I/Oを隔離し、入力と出力を明示して検証する。

#### 統合テスト

- 対象: コンポーネントと状態管理、画面とAPI境界、フォーム送信フロー。
- 目的: 複数モジュール結合時の破綻を検出する。
- 原則: 依存の境界だけを置換し、業務ロジック本体は実装実体で検証する。

#### E2Eテスト

- 対象: 認証、主要導線、決済前後、設定変更、回帰リスクの高い操作列。
- 目的: 実ブラウザー上で本番に近い利用シナリオを検証する。
- 原則: 最重要導線に絞って運用し、過剰なケース追加で保守不能化させない。

#### コンポーネント回帰テスト（Storybook）

- 対象: UIコンポーネントの状態差分、対話フロー、表示退行。
- 目的: E2Eに入れる前段で、視覚差分とUI対話の破壊を早期検出する。
- 原則: Story を仕様単位として維持し、API依存は `MSW` で境界モック化する。

出典・参考情報: Playwright Best Practices、Cypress Best Practices、Testing Library Guiding Principles、Interaction tests | Storybook docs、Mocking network requests | Storybook docs、Mock Service Worker - API mocking library for browser and Node.js、FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）。

### 変更種別と必須テスト

| 変更種別 | 単体 | 統合 | E2E |
| --- | --- | --- | --- |
| ユーティリティ関数変更 | 必須 | 条件付き | 不要 |
| UIコンポーネント表示変更 | 必須 | 必須 | 条件付き |
| フォーム仕様変更 | 必須 | 必須 | 必須 |
| 画面遷移変更 | 条件付き | 必須 | 必須 |
| APIレスポンス処理変更 | 必須 | 必須 | 条件付き |
| 認証/課金導線変更 | 必須 | 必須 | 必須 |

#### 条件付き判定基準

次の3条件をすべて満たす場合のみ、`条件付き` のテストを省略可能とする。

1. 変更が表示文言、スタイル、定数の軽微修正に限定され、状態遷移や分岐条件を変更していない。
2. 対象変更が既存の必須テスト範囲で間接的に検証されることを確認できる。
3. Pull Request本文に省略理由と影響範囲を明記している。

上記のいずれかを満たさない場合は、`条件付き` を `必須` として扱う。

出典・参考情報: Testing Library Guiding Principles、Playwright Best Practices、Cypress Best Practices。

### テストファイル命名と配置

#### Unit/Integration（Jest/Vitest）

- 新規作成するテストファイル名は `*.test.ts`、`*.test.tsx` に統一する。
- `*.spec.ts`、`*.spec.tsx` は既存資産の保守時のみ許可し、新規追加を禁止する。
- 配置は実装ファイル隣接を標準とする。
- `__tests__/` 配下集約は既存構成を維持する場合のみ例外として許可する。

#### E2E（Playwright/Cypress）

- Playwright は `tests/e2e/**/*.e2e.spec.ts` を標準とし、`testDir` と `testMatch` を明示設定する。
- Cypress は `cypress/e2e/**/*.cy.ts` を標準とする。
- Cypress 自体は `*.cy.{js,jsx,ts,tsx}` を許容するが、本規定では運用統一のため `*.cy.ts` を採用する。
- 採用理由は、`tsc --noEmit` による型検査をテストコードまで一貫適用し、レビュー基準と CI 失敗条件を単純化するためである。
- 新規追加は `*.cy.ts` を必須とし、既存の JavaScript 拡張子は保守時に段階移行する。

出典・参考情報: Playwright Test Configuration、Cypress Writing and Organizing Tests、Jest Configuration、Vitest Config Reference。

### テストケース命名とクエリ規約

1. テスト名は `対象_条件_期待結果` の3要素で記述する。
2. `describe` の階層は原則2段以内とし、責務が曖昧な入れ子を禁止する。
3. DOM 取得は `role`、`label`、`text` を優先し、`data-testid` は最終手段とする。
4. `class`、`id`、DOM構造に依存したセレクターを主要判定に使わない。
5. `data-testid` を使う場合は `機能-要素-意図` の命名規約で固定し、見た目変更で崩れない識別子とする。

出典・参考情報: JavaScript Testing Best Practices、JavaScriptのテストに関するベストプラクティス（Zenn）、About Queries | Testing Library、ByTestId | Testing Library、Cypress Best Practices。

### カバレッジ基準

#### 基準値設定の明示

1. カバレッジ値には普遍的な理想値がなく、業務影響と保守性を踏まえて決定する（Google Testing Blog）。
2. 新規変更に対する 80%以上の基準は、SonarQube の推奨品質ゲートでも採用されている（SonarQube Quality Gates）。
3. 本規定の全体基準 `statements/functions/lines 85%` は、Google の一般目安（60/75/90）のうち「75（commendable）」と「90（exemplary）」の間を狙う組織目標値として設定する。
4. `branches 75%` は同一般目安の下限を採用した最低基準とし、重要領域では `branches 85%` へ引き上げる。
5. 上記数値は外部規格の固定値ではなく、SLA とリスク許容度に基づく運用基準である。

#### 全体基準

- statements: 85%以上
- branches: 75%以上
- functions: 85%以上
- lines: 85%以上

#### 重要領域基準

- 対象: 認証、課金、権限、データ削除、監査ログ関連。
- lines: 90%以上
- branches: 85%以上

#### 運用ルール

1. Pull Request では全体基準未達を失敗扱いとする。
2. 重要領域変更時は重要領域基準も同時に満たす。
3. カバレッジ値は最低基準であり、品質保証そのものではない。境界値と異常系の明示テストを必須とする。
4. 重要領域変更の判定条件と強制方法は「CI最終ゲート（required status checks）＞重要領域判定（カバレッジ強化条件）」を正本とし、該当時は `ci/unit-integration` 内で重要領域基準（lines 90%、branches 85%）を追加強制する。

出典・参考情報: Google Testing Blog - Code Coverage Best Practices、Quality gates | SonarQube Server 10.8、Jest configuration (`coverageThreshold`)、Vitest coverage config、Using conditions to control job execution、Running variations of jobs in a workflow。

### モック・スタブ使用方針

1. 単体テストでは外部I/Oをモック化し、ロジック本体の観測可能な結果を検証する。
2. 統合テストではネットワーク境界と時刻依存のみを主対象に置換する。
3. E2Eでは、第三者要因を除くための最小限のスタブのみ許可する。
4. テスト対象そのもののメソッドを過剰モック化し、実装の挙動検証を失わせることを禁止する。

#### モック対象選定基準

1. モック対象は「自チーム管理外で不安定な依存」に限定する。
2. 同一リポジトリ内のモジュールやフックは原則実体で検証し、内部結合をモックで隠蔽しない。
3. API 境界のモックは `MSW` を標準とし、Storybook と統合テストで定義を共通化する。
4. モック追加時は「実環境で再現不能な失敗を防ぐ目的」があることを PR に明記する。

出典・参考情報: Testing Library Guiding Principles、Playwright Best Practices、Cypress Best Practices、Mock Service Worker - API mocking library for browser and Node.js、Mocking network requests | Storybook docs、JavaScriptのテストに関するベストプラクティス（Zenn）。

### 待機、再試行、タイムアウト

1. 固定待機の常用を禁止する。
2. Playwright は Web First Assertions と自動待機を前提に記述する。
3. Testing Library は `findBy` / `waitFor` を使用し、非同期更新を状態条件で待機する。
4. CI の自動再試行は E2E に限定し、上限を1回とする。
5. 自動再試行の対象は、ネットワークタイムアウト、外部依存の一時不安定、ブラウザー実行環境由来の失敗に限定する。
6. 仕様不一致、アサーション不一致、セレクター不一致は再試行対象にしない。
7. 再試行で成功したテストは合格ではなく不安定テストとして記録し、恒久修正対象とする。

出典・参考情報: Playwright Best Practices、Playwright Test Configuration、Cypress Best Practices、Cypress Test Retries、Testing Library Async Methods。

### アクセシビリティ検証

1. 自動検証と手動検証を併用する。
2. 自動検証は `axe-core` 系ツールを入口として導入する。
3. 主要画面は次を手動で確認する。
   - キーボード操作のみで完了可能か。
   - フォーカス遷移が視認可能か。
   - `role` とアクセシブルネームが妥当か。

出典・参考情報: Cypress Accessibility Testing Guide、GitHub - dequelabs/axe-core、Web Content Accessibility Guidelines (WCAG) 2.2、Accessible Rich Internet Applications (WAI-ARIA) 1.2。

### 不安定テスト管理

#### 定義

同一コミット・同一環境で再実行時に結果が変動するテストを不安定テストと定義する。

#### 対応手順

1. 検出時に「原因仮説」「再現条件」「暫定対策」「影響範囲」を記録する。
2. 記録先は `tests/flaky/flaky-tests.yaml` とし、チケット番号を必須とする。
3. 一時隔離する場合の期限は最大7日とし、期限超過を許可しない。
4. 隔離対象は Pull Request テンプレートに明記し、レビュー承認なしで隔離しない。
5. 復帰条件は「連続30回のCI成功」または「夜間全量実行で連続5日成功」のいずれかとする。

出典・参考情報: Playwright Isolation、Cypress Test Isolation、Cypress Test Retries、Playwright Test Configuration。

### CI品質ゲート

1. PR作成時は `ci/lint`、`ci/typecheck`、`ci/unit-integration` を必須とし、全ジョブ成功をマージ条件とする。
2. PR作成時はカバレッジ閾値（statements/functions/lines 85%以上、branches 75%以上）未達を失敗扱いとする。
3. `main` マージ判定で参照する required check 名、失敗条件、必須証跡、no-op許可条件は「CI最終ゲート（required status checks）」を正本とする。
4. `ci/e2e-critical` の対象テストは主系統に応じて、Playwright では `tests/e2e/critical/**/*.e2e.spec.ts`、Cypress では `cypress/e2e/critical/**/*.cy.ts` を用いる。
5. クロスブラウザー方針は `Chromium系 + Safari(WebKit) + Firefox` とし、`main` マージ前の required 判定は Chromium系 + Safari(WebKit) を対象、Firefox は夜間または週次回帰で補完する。
6. `main` マージ前は E2E の再試行成功のみを合格扱いにしない。
7. 夜間定期実行では E2E 全量を実行し、失敗時はログ、スクリーンショット、トレースを必ず保存する。
8. 夜間定期実行で失敗が出た場合は、翌営業日までにチケット化し、担当者と期限を設定する。
9. SLA に関わる監視指標に回帰がある場合、機能テストが合格でもリリース判定を保留する。
10. `CI品質ゲート` と `CI最終ゲート` が矛盾する場合は `CI最終ゲート` を優先し、同一PRで同期修正する。

出典・参考情報: Quality gates | SonarQube Server 10.8、Jest configuration (`coverageThreshold`)、Vitest coverage config、Playwright Test Configuration、Browsers | Playwright、Cypress Writing and Organizing Tests、Cross Browser Testing: Cypress Guide、Managing protected branches、Troubleshooting required status checks。

### 品質検査ツール一覧

#### 静的検査ツール

1. `ESLint`
2. `typescript-eslint`
3. `Stylelint`
4. `Prettier --check`
5. `TypeScript（tsc --noEmit）`

#### テスト実行ツール

1. `Vitest`（単体・統合）
2. `Jest`（単体・統合）
3. `Testing Library`（UI統合）
4. `MSW`（API 境界モック）
5. `Storybook Interaction Tests`（UI対話検証）
6. `Chromatic`（ビジュアルリグレッション）
7. `Playwright`（E2E）
8. `Cypress`（E2E）
9. `axe-core`（アクセシビリティ自動検査）
10. `Lighthouse CI`（性能ゲート）
11. `web-vitals`（実利用性能監視）

#### セキュリティ検査ツール

1. `CodeQL（JavaScript/TypeScript）`
2. `actions/dependency-review-action`
3. `Dependabot alerts`
4. `npm audit`
5. `GitHub Secret Scanning Push Protection`
6. `OWASP ZAP Baseline`

出典・参考情報: ESLint、typescript-eslint Getting Started、Stylelint Getting started、Prettier Options、TypeScript TSConfig `noEmit`、Vitest Config Reference、Jest Configuration、Testing Library Guiding Principles、Mock Service Worker - API mocking library for browser and Node.js、Interaction tests | Storybook docs、Introduction to TurboSnap • Chromatic docs、Playwright Best Practices、Cypress Best Practices、Cypress Accessibility Testing Guide、GitHub - dequelabs/axe-core、Lighthouse CI Getting Started、Web Vitals、Queries for CodeQL analysis、CodeQL `js/xss` query help、Dependency Review Action、About Dependabot alerts、npm audit、About push protection、ZAP Baseline Scan Action。

### 品質検査ツール運用基準

1. PR では静的検査ツールを全件実行し、1件でも失敗した場合はマージ不可とする。
2. PR では単体・統合テストを必須とし、`Vitest` または `Jest` のいずれか一系統でカバレッジ閾値を強制する。
3. `main` マージ前は `Playwright` または `Cypress` による重要導線 E2E を必須とし、再試行成功のみは合格扱いにしない。
4. クロスサイトスクリプティングを含むクライアントサイド脆弱性は `CodeQL` の JavaScript/TypeScript セキュリティクエリで検出し、重大度しきい値を超えた場合はリリース不可とする。
5. PR時の `ci/dependency` は `actions/dependency-review-action` と `npm audit` を必須とし、High 以上を未解消のまま許容しない。
6. `Dependabot alerts` は継続監視として運用し、未解消の High/Critical が残る場合はリリース不可とする（PR required check には直接含めない）。
7. 秘密情報漏えい対策として `GitHub Secret Scanning Push Protection` を有効化し、検出された push を遮断する。
8. 公開前ステージでは `OWASP ZAP Baseline` を実行し、警告以上はチケット化して是正期限を設定する。
9. `axe-core` を CI で継続実行し、重大違反未解消を `ci/accessibility` 失敗とする。
10. 主要画面のキーボード操作、フォーカス遷移、`role` とアクセシブルネームの手動確認は PR レビューの必須チェックリストとする。
11. `Lighthouse CI` の性能アサーションと `web-vitals` の LCP、INP、CLS 監視を SLA 判定に連携する。
12. `Storybook Interaction Tests` と `Chromatic` は required check として常時有効化し、UI変更判定と no-op 許可条件は「CI最終ゲート（required status checks）＞UI変更判定（no-op許可条件）」に従う。
13. `Chromatic` は差分対象を変更Storyに限定する設定を標準とし、全量再計測は定期ジョブで補完する。

出典・参考情報: OWASP Cross Site Scripting Prevention Cheat Sheet、Web Vitals、Queries for CodeQL analysis、CodeQL `js/xss` query help、Dependency Review Action、About Dependabot alerts、npm audit、About push protection、ZAP Baseline Scan Action、Lighthouse CI Getting Started、Cypress Accessibility Testing Guide、GitHub - dequelabs/axe-core、Web Content Accessibility Guidelines (WCAG) 2.2、Accessible Rich Internet Applications (WAI-ARIA) 1.2、Interaction tests | Storybook docs、Introduction to TurboSnap • Chromatic docs、FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）。

### 例外運用

#### 緊急例外の定義

本規定における「リリース阻害を回避する緊急例外」とは、次の条件をすべて満たす場合を指す。

1. 期限が固定された外部要件が存在し、予定時刻を超過すると事業上または契約上の重大な不利益が発生する。
2. 通常手順での修正完了が、期限内に現実的に不可能であることを、担当者とレビュアーの双方が確認している。
3. 例外適用範囲が限定され、代替検証手段によって受容可能なリスク水準に抑えられる。
4. 例外解消の期限と担当者が明示され、追跡可能なチケットが作成されている。

本例外は品質基準、SLA 判定、required checks の免除を意味しない。  
免除が必要になる場合は例外適用ではなく、リリース日改定を選択する。

次の状況は緊急例外として扱わない。

1. 見積もり不足や着手遅延など、計画不備のみを理由とする遅延。
2. 成果物品質より納期優先を選ぶだけの判断。
3. 恒久対応の計画がない一時回避。

#### リリース日改定の原則

品質が本規定の基準を満たさない場合は、例外適用で出荷を優先せず、リリース日を改定する。  
品質判定は機能品質と SLA 達成可能性を同時に満たすことを条件とする。

次のいずれかに該当する場合、リリース日改定を必須とする。

1. 必須テストが未実施または失敗のままである。
2. カバレッジ基準未達の状態で、代替検証で不足を補完できない。
3. 重大不具合の解消見込みがリリース時点までに立たない。
4. セキュリティインシデントにつながる脆弱性または誤実装の懸念が残る。
5. サービスまたはアプリケーションの持続運用性を損なう障害リスクが受容水準を超える。
6. リリース後に SLA を満たせない蓋然性が高く、事前対策で解消できない。

#### サービスレベル適合性判定

本規定で扱うサービスレベル適合性は、次の観点で判定する。

1. 可用性目標を満たせる運用設計になっているか。
2. 主要操作の応答時間がサービス目標の範囲内か。
3. 障害発生時の復旧時間目標を達成できる監視と復旧手順があるか。
4. 既知リスクが SLA 違反に直結しない水準まで低減されているか。

サービスレベル適合性が未判定、または未達の場合はリリース不可とする。

#### リリース日改定時のアナウンス準備

リリース日改定を決定した場合は、実施前に次の情報を準備し、関係者へ通知する。

1. 改定理由（どの基準が未達か）。
2. 影響範囲（機能、利用者、関連スケジュール）。
3. 新しいリリース予定日。
4. 代替対応（暫定運用、回避策、問い合わせ窓口）。
5. 次回判定日と責任者。
6. セキュリティ影響評価（機密性、完全性、可用性への影響）。
7. 運用継続性への影響評価（停止可能性、性能劣化、復旧難易度）。
8. SLA 影響評価（想定違反指標、影響時間、利用者影響）。
9. SLA 回復計画（是正策、回復目標時刻、監視強化内容）。

上記の準備が完了するまで、改定の確定連絡を出さない。

リリース阻害を回避する緊急例外を承認する場合は、例外記録に次の3項目を必須記載とする。

1. 期限: 例外の失効日時。
2. 代替検証手段: 欠落した検証を補完する具体手順と判定基準。
3. 修正担当者: 恒久対応の責任者。

出典・参考情報: SRE Book - Service Level Objectives、SRE Book - Monitoring Distributed Systems、ISO/IEC 20000-1:2018、Atlassian - SLA（Service Level Agreement）、OWASP Cross Site Scripting Prevention Cheat Sheet。

期限・代替・担当のいずれかが欠ける場合、例外承認を無効とする。

## 実装例

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      thresholds: {
        statements: 85,
        branches: 75,
        functions: 85,
        lines: 85,
      },
    },
  },
});
```

```ts
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  testMatch: [
    '**/__tests__/**/*.[jt]s?(x)',
    '**/?(*.)+(spec|test).[jt]s?(x)',
  ],
  coverageThreshold: {
    global: {
      statements: 85,
      branches: 75,
      functions: 85,
      lines: 85,
    },
  },
};

export default config;
```

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  testMatch: /.*\.e2e\.spec\.ts/,
  retries: process.env.CI ? 1 : 0,
  use: {
    trace: 'on-first-retry',
  },
});
```

```ts
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    specPattern: 'cypress/e2e/**/*.cy.ts',
  },
  retries: {
    runMode: 1,
    openMode: 0,
  },
});
```

## 工程のオートメーション化（AI + CIオーケストレーション）

### 目的

本章は、超少人数体制でも、AIを活用してテスト、品質、セキュリティチェックを継続遂行するための実装可能な運用基準を定義する。  
対象は、コミット前、PRレビュー、AIによる修正実行、CI最終判定、ブランチ保護、夜間定期実行、リリース判定までとする。

### 全体アーキテクチャ

| 工程 | AIレイヤー | CIレイヤー | 人間レイヤー |
| --- | --- | --- | --- |
| コミット前 | CodeRabbit IDE/CLI、Copilot review（任意） | ローカル検証 | 開発者 |
| PRレビュー | CodeRabbit、Copilot code review | PR checks 実行 | 著者、レビュアー |
| 修正実装 | OpenAI Codex、Copilot coding agent | 修正ブランチ上で再検証 | 著者 |
| 最終判定 | 補助情報のみ | required status checks + branch protection | 承認者 |

出典・参考情報: CodeRabbit Pull Request Reviews、Using GitHub Copilot code review、About GitHub Copilot coding agent、Introducing Codex。

### AIツール別運用規定

#### CodeRabbit（一次レビュー）

1. PR作成時は自動レビュー、push時は増分レビューを必須とする。
2. 手動再実行は `@coderabbitai review`、全体再評価は `@coderabbitai full review` を使用する。
3. コミット前の差分確認は `coderabbit --plain --type uncommitted` を標準とする。
4. Pre-Merge Checks は初期を `warning`、運用安定後に `error` へ段階移行する。

出典・参考情報: CodeRabbit Pull Request Reviews、CodeRabbit Manage code reviews、CodeRabbit Code review commands、CodeRabbit Built-in Pre-Merge Checks、CodeRabbit Code Reviews in IDE and CLI、CodeRabbit Command-Line Review Tool。

#### GitHub Copilot code review（補助レビュー）

1. 補助的な観点追加として併用を許可する。
2. Copilot review は `Comment` のみで、承認要件やマージ阻止には直接使えない前提で運用する。

出典・参考情報: Using GitHub Copilot code review。

#### GitHub Copilot coding agent（修正実行）

1. Issue または PR コメントから `@copilot` で修正作業を委譲できる。
2. エージェントはGitHub Actionsベースの隔離環境で実装、テスト、リンター実行を行う。
3. エージェント成果物はPRとして提出し、レビュー経由で取り込む。

出典・参考情報: About GitHub Copilot coding agent。

#### OpenAI Codex（修正実行）

1. Codex はタスクごとに分離環境でコード編集、テスト、リンター、型検査を実行する。
2. 修正結果は差分と実行ログを必須提出とし、PRに添付して追跡可能にする。
3. Codex の結果も required status checks 全件成功を満たすまでマージしない。

出典・参考情報: Introducing Codex。

### テスト自動化の導入優先順位

1. 最初に自動化する対象は「高頻度」「高リスク」「判定が機械化しやすい」導線とする。
2. 仕様変動が大きい機能は、初期段階で全ケースをE2E化せず、smoke + 統合テスト + 手動探索で運用する。
3. 自動化対象は四半期ごとに見直し、実行コストと欠陥検出率が見合わないケースは縮退または再設計する。
4. 手動検証は廃止せず、新機能探索とUX評価に集中させる。

出典・参考情報: cypressで始めるE2Eテスト（Qiita）、FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）。

### CI最終ゲート（required status checks）

#### 必須チェック識別子（固定）

| check名 | 主な実行内容 | 失敗条件 | 必須証跡 |
| --- | --- | --- | --- |
| `ci/lint` | ESLint、Stylelint、Prettier check | いずれか非ゼロ終了 | 実行ログ |
| `ci/typecheck` | `tsc --noEmit` | 型エラー1件以上 | 実行ログ |
| `ci/unit-integration` | Jest または Vitest + coverage | テスト失敗、閾値未達 | カバレッジレポート |
| `ci/e2e-critical` | 主要導線E2E | 失敗、再試行後も不安定 | 動画、スクリーンショット、トレース |
| `ci/security-sast` | CodeQL | 重大度しきい値超過 | SARIF |
| `ci/dependency` | dependency-review、npm audit | High 以上未解消 | 監査ログ |
| `ci/performance` | Lighthouse CI（PRラボ計測） | ラボ予算超過 | Lighthouse結果 |
| `ci/accessibility` | axe-core 自動検査 | 重大違反未解消 | a11y結果 |
| `ci/storybook-interaction` | Storybook Interaction Tests | テスト失敗、未実行 | 実行ログ |
| `ci/visual-regression` | Chromatic | 未承認差分、検査失敗 | 差分レポートURL |

#### 運用ルール

1. PRマージ可否は上記 required checks の合否のみで機械判定する。
2. required check 名はワークフロー間で重複させない。
3. required check の送信元は必要に応じて期待する GitHub App を固定する。
4. E2E基盤はリポジトリ単位で `Playwright` または `Cypress` の主系統を固定し、クロスブラウザー戦略全体は `Chromium系 + Safari(WebKit) + Firefox` とする。
5. `ci/storybook-interaction` と `ci/visual-regression` は UI変更がない場合も必ず終了ステータスを返し、`Pending` のまま残さない。
6. `ci/e2e-critical` の required 判定は Chromium系 + Safari(WebKit) を対象とし、Firefox は夜間または週次の回帰ジョブで補完する。
7. `ci/performance` は PR required check として Lighthouse ラボ閾値で判定する。

#### 重要領域判定（カバレッジ強化条件）

1. 重要領域は認証、課金、権限、データ削除、監査ログ関連とする。
2. 判定対象パスと判定ロジックは生成元（スクリプト/workflow）を正本として管理し、required check ジョブ内へのハードコードを禁止する。
3. 重要領域差分を含むPRでは `ci/unit-integration` で lines 90%以上、branches 85%以上を強制する。
4. 判定不能（差分取得失敗、rename判定失敗、パス上限超過など）の場合は fail-safe として重要領域基準を強制する。

#### UI変更判定（no-op許可条件）

1. `ci/storybook-interaction` と `ci/visual-regression` が no-op を返せるのは、PR差分が判定対象パスに1件も一致しない場合のみとする。
2. 判定対象パスは生成元（スクリプト/workflow）を正本として管理し、required check ジョブ内へのハードコードを禁止する。
3. 生成元（スクリプト/workflow）で管理する初期判定対象は `src/**/*.{ts,tsx,js,jsx,css,scss,sass,less,styl}`、`app/**/*.{ts,tsx,js,jsx,css,scss,sass,less,styl}`、`packages/**/src/**/*.{ts,tsx,js,jsx,css,scss,sass,less,styl}`、`**/*.stories.{ts,tsx,js,jsx,mdx}`、`.storybook/**`、`storybook/**`、`public/**/*.{html,css,svg,png,jpg,jpeg,webp,avif,gif}` とする。
4. ディレクトリ構成を変更するPRでは、同一PRで生成元（スクリプト/workflow）の判定対象も更新し、取りこぼしを許可しない。
5. 差分取得失敗、rename判定失敗、パス上限超過などで判定不能な場合は fail-safe として実検査を実行する。
6. no-op 実行時も job は `success` で終了し、判定対象ファイル一覧と判定結果をログまたはアーティファクトで保存する。
7. 判定ロジックを変更する場合は `CI最終ゲート` と `GitHub Actions実装規約` を同一PRで更新する。

#### SLA/SLOリリース判定（required check外）

1. web-vitals による SLO 達成判定は `release/slo-readiness` ジョブで実施し、PR required check には含めない。
2. 判定指標は LCP、INP、CLS とし、集計は28日ローリングの75パーセンタイルを用いる。
3. いずれかのSLOが未達の場合は、required checks が全件成功でもリリース停止とする。
4. `release/slo-readiness` の結果はリリース判定記録に必ず残す。

出典・参考情報: Managing protected branches、Troubleshooting required status checks、Browsers | Playwright、Cross Browser Testing: Cypress Guide、Interaction tests | Storybook docs、Introduction to TurboSnap • Chromatic docs、Using conditions to control job execution、Skipping workflow runs、Web Vitals。

### ブランチ保護ルール（必須設定）

1. `Require a pull request before merging` を有効化する。
2. `Require status checks to pass before merging` を有効化する。
3. `Require branches to be up to date before merging` を有効化する。
4. `Require conversation resolution before merging` を有効化する。
5. `Do not allow bypassing the above settings` を有効化する。
6. 承認数は体制に応じて設定する。1名体制は0、2名以上は1以上を必須とする。

出典・参考情報: Managing protected branches。

### GitHub Actions実装規約

1. required check 対象ワークフローに `paths` / `paths-ignore` を設定しない。
2. 実行可否分岐はワークフロー単位ではなく job / step の条件式で制御する。
3. required check が `Pending` で残る構成を禁止する。
4. matrix 実行時は required check と非required check を分離し、判定対象を明確にする。
5. 同一PRの古い実行は `concurrency` で中断し、最新結果のみを評価する。
6. UI変更判定の対象パスは生成元（スクリプト/workflow）を正本とし、required check ワークフロー内のハードコードを禁止する。

出典・参考情報: Using conditions to control job execution、Skipping workflow runs、Running variations of jobs in a workflow、Troubleshooting required status checks。

### CI実行最適化（時間とノイズ制御）

1. E2E は `shard` または `parallel` を利用して実行時間を短縮し、完了時間の上限目標を定める。
2. クロスブラウザー実行は `Chromium系 + Safari(WebKit) + Firefox` を対象とし、critical path は Chromium系 + Safari(WebKit) を優先する。
3. 失敗証跡のアップロードは `failure()` 条件で実行し、成功時の不要アーティファクト保存を禁止する。
4. チャット通知は失敗時のみ送信し、同一原因の重複通知を抑制する。
5. UI差分検証は変更Story優先で実行し、全量実行は夜間または定期ジョブに分離する。

出典・参考情報: Sharding | Playwright、Parallelization | Cypress Documentation、Cross Browser Testing: Cypress Guide、Store and share data with workflow artifacts、Evaluate expressions in workflows and actions、Introduction to TurboSnap • Chromatic docs、cypressで始めるE2Eテスト（Qiita）、FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）。

### 失敗時SOP（超少人数向け）

| 区分 | 例 | 初動 | 期限 |
| --- | --- | --- | --- |
| Blocker | `ci/security-sast`、`ci/dependency` 重大失敗 | マージ停止、即時チケット化、担当者固定 | 当日 |
| Critical | `ci/e2e-critical`、`ci/performance`、`ci/accessibility`、`ci/storybook-interaction`、`ci/visual-regression`、`release/slo-readiness` 失敗、`Dependabot alerts` の High/Critical 未解消 | マージ停止またはリリース停止、当日中に再実行と原因分類、翌営業日までに恒久対応チケット起票 | 翌営業日 |
| Major | `ci/lint`、`ci/typecheck`、`ci/unit-integration` 失敗 | 修正コミット、再実行 | 当日 |
| Minor | required check 以外のうち、リリース可否に直接影響しないジョブ（夜間回帰、通知系、分析補助ジョブ）の失敗 | 翌営業日までにチケット化し、影響評価付きで改善計画を登録 | 1週間以内 |

出典・参考情報: Troubleshooting required status checks、CodeRabbit Built-in Pre-Merge Checks、About Dependabot alerts。

### 超少人数体制の運用基準

1. 1名体制でもAIレビュー結果とrequired checks全件成功を満たすまでマージしない。
2. 2名体制以上では、実装担当と承認担当を分離し、自己承認を禁止する。
3. 緊急例外時でも、required checks のスキップによるマージを行わず、期限付き代替検証を必須とする。

出典・参考情報: Managing protected branches、CodeRabbit Pull Request Reviews、CodeRabbit Built-in Pre-Merge Checks、Troubleshooting required status checks。

## 出典・参考情報
### 情報・規格

- [Google Testing Blog - Code Coverage Best Practices](https://testing.googleblog.com/2020/08/code-coverage-best-practices.html) - カバレッジ値に理想の固定値はなく、業務影響に応じて設定すべきという原則と、60/75/90 の一般目安。
- [Quality gates | SonarQube Server 10.8](https://docs.sonarsource.com/sonarqube-server/10.8/instance-administration/analysis-functions/quality-gates) - 推奨品質ゲートにおける新規コードカバレッジ 80% 条件。
- [SRE Book - Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) - SLA/SLO/SLI の関係と運用設計。
- [SRE Workbook - Embracing Risk](https://sre.google/workbook/embracing-risk/) - エラーバジェット運用とリリース判断の原則。
- [SRE Book - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/) - サービスレベル監視と指標設計。
- [ISO/IEC 20000-1:2018](https://www.iso.org/standard/70636.html) - サービスマネジメントにおける合意水準管理の国際規格。
- [Atlassian - SLA（Service Level Agreement）](https://www.atlassian.com/itsm/service-request-management/slas) - SLA の定義と実務運用の整理。
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) - XSS 防御原則。
- [Web Vitals](https://web.dev/articles/vitals) - LCP、INP、CLS と75パーセンタイル判定基準。
- [Web Content Accessibility Guidelines (WCAG) 2.2](https://www.w3.org/TR/WCAG22/) - アクセシビリティ達成基準。
- [Accessible Rich Internet Applications (WAI-ARIA) 1.2](https://www.w3.org/TR/wai-aria-1.2/) - ARIA ロールと属性仕様。

### ツール公式ドキュメント

- [Playwright Best Practices](https://playwright.dev/docs/best-practices) - E2E設計原則と推奨パターン。
- [Browsers | Playwright](https://playwright.dev/docs/browsers) - Chromium、WebKit、Firefox の実行基盤。
- [Sharding | Playwright](https://playwright.dev/docs/test-sharding) - E2E並列分割実行。
- [Playwright Isolation](https://playwright.dev/docs/browser-contexts) - テスト分離と再現性の設計。
- [Playwright Test Configuration](https://playwright.dev/docs/test-configuration) - `testDir`、`testMatch`、`retries`、`trace` の設定。
- [Cypress Best Practices](https://docs.cypress.io/app/core-concepts/best-practices) - 待機・状態管理・実装方針。
- [Parallelization | Cypress Documentation](https://docs.cypress.io/cloud/features/smart-orchestration/parallelization) - Cypress並列実行。
- [Cross Browser Testing: Cypress Guide](https://docs.cypress.io/guides/guides/cross-browser-testing) - クロスブラウザー実行戦略。
- [Cypress Test Isolation](https://docs.cypress.io/app/core-concepts/test-isolation) - テスト独立性の原則。
- [Cypress Writing and Organizing Tests](https://docs.cypress.io/app/core-concepts/writing-and-organizing-tests) - テスト命名と配置規約。
- [Cypress Test Retries](https://docs.cypress.io/app/guides/test-retries) - 再試行ポリシーの運用。
- [Cypress Accessibility Testing Guide](https://docs.cypress.io/app/guides/accessibility-testing) - アクセシビリティ検証の導入方針。
- [GitHub - dequelabs/axe-core](https://github.com/dequelabs/axe-core) - ルールエンジンと実装仕様。
- [Testing Library Guiding Principles](https://testing-library.com/docs/guiding-principles) - 利用者視点でのテスト原則。
- [About Queries | Testing Library](https://testing-library.com/docs/queries/about) - クエリ優先順位。
- [ByTestId | Testing Library](https://testing-library.com/docs/queries/bytestid) - `data-testid` 利用指針。
- [Testing Library Async Methods](https://testing-library.com/docs/dom-testing-library/api-async) - 非同期テストの待機手法。
- [JavaScript Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices) - JavaScriptテスト設計の実践パターン。
- [Mock Service Worker - API mocking library for browser and Node.js](https://mswjs.io/) - API境界モックの標準実装。
- [Mocking network requests | Storybook docs](https://storybook.js.org/docs/writing-stories/mocking-data-and-modules/mocking-network-requests) - Storybookでのネットワークモック。
- [Interaction tests | Storybook docs](https://storybook.js.org/docs/writing-tests/interaction-testing) - Story単位のUI対話検証。
- [Introduction to TurboSnap • Chromatic docs](https://docs.chromatic.com/docs/turbosnap/) - 差分対象を絞るVRT最適化。
- [Jest Configuration](https://jestjs.io/docs/configuration) - `testMatch` と `coverageThreshold` の設定。
- [Vitest Config Reference](https://vitest.dev/config/) - `include` とカバレッジ閾値の設定。
- [ESLint](https://eslint.org/) - JavaScript/TypeScript の静的解析。
- [typescript-eslint Getting Started](https://typescript-eslint.io/getting-started/) - TypeScript 向け ESLint 設定。
- [Stylelint Getting started](https://stylelint.io/user-guide/get-started) - CSS 系スタイル検査。
- [Prettier Options](https://prettier.io/docs/options) - 整形規約の標準化。
- [TypeScript TSConfig `noEmit`](https://www.typescriptlang.org/tsconfig/noEmit.html) - 生成物を出力しない型検査。
- [Vitest coverage config](https://vitest.dev/config/coverage.html) - `coverage.thresholds` の基準設定。
- [Jest configuration (`coverageThreshold`)](https://jestjs.io/docs/configuration#coveragethreshold-object) - カバレッジ閾値失敗条件。
- [Queries for CodeQL analysis](https://docs.github.com/en/code-security/reference/code-scanning/codeql/codeql-queries) - `default` / `security-extended` クエリスイート。
- [CodeQL `js/xss` query help](https://codeql.github.com/codeql-query-help/javascript/js-xss/) - DOM ベース XSS の検出。
- [Dependency Review Action](https://github.com/actions/dependency-review-action) - PR での依存関係脆弱性検査。
- [About Dependabot alerts](https://docs.github.com/en/code-security/dependabot/dependabot-alerts/about-dependabot-alerts) - 既知脆弱性の継続検出。
- [npm audit](https://docs.npmjs.com/cli/v10/commands/npm-audit/) - 依存関係の脆弱性監査。
- [About push protection](https://docs.github.com/code-security/secret-scanning/protecting-pushes-with-secret-scanning) - 秘密情報の push 時遮断。
- [ZAP Baseline Scan Action](https://github.com/zaproxy/action-baseline) - 自動化可能なベースライン動的診断。
- [Lighthouse CI Getting Started](https://googlechrome.github.io/lighthouse-ci/docs/getting-started.html) - CI での性能アサーション。
- [CodeRabbit Pull Request Reviews](https://docs.coderabbit.ai/overview/pull-request-review) - PR自動レビューと増分レビュー。
- [CodeRabbit Manage code reviews](https://docs.coderabbit.ai/guides/commands/) - `@coderabbitai` によるレビュー制御。
- [CodeRabbit Code review commands](https://docs.coderabbit.ai/reference/review-commands/) - `review`、`full review` を含むコマンド定義。
- [CodeRabbit Built-in Pre-Merge Checks](https://docs.coderabbit.ai/pr-reviews/pre-merge-checks) - Pre-Merge Checks の `off`、`warning`、`error` 運用。
- [CodeRabbit Code Reviews in IDE and CLI](https://docs.coderabbit.ai/overview/ide-cli-review) - IDEとCLIでのレビュー運用。
- [CodeRabbit Command-Line Review Tool](https://docs.coderabbit.ai/cli) - コミット前CLIレビューの実行方法。
- [Using GitHub Copilot code review](https://docs.github.com/en/copilot/how-tos/agents/copilot-code-review/using-copilot-code-review) - Copilot review の挙動と制約。
- [About GitHub Copilot coding agent](https://docs.github.com/copilot/concepts/coding-agent/coding-agent) - コーディングエージェントの実行環境と制約。
- [Introducing Codex](https://openai.com/index/introducing-codex/) - Codex の隔離実行環境と検証実行能力。
- [Managing protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches) - 保護ブランチとルール管理。
- [Troubleshooting required status checks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/troubleshooting-required-status-checks) - required check 運用時の注意点。
- [Using conditions to control job execution](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/control-jobs-with-conditions) - job条件制御時のステータス挙動。
- [Evaluate expressions in workflows and actions](https://docs.github.com/en/actions/reference/evaluate-expressions-in-workflows-and-actions) - `failure()` など条件式評価。
- [Skipping workflow runs](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/skipping-workflow-runs) - スキップ時に required check が Pending になる条件。
- [Running variations of jobs in a workflow](https://docs.github.com/en/actions/using-jobs/using-a-matrix-for-your-jobs) - matrix による並列実行設計。
- [Store and share data with workflow artifacts](https://docs.github.com/en/actions/how-tos/writing-workflows/choosing-what-your-workflow-does/storing-and-sharing-data-from-a-workflow) - 失敗証跡の保存設計。

### 実践記事・事例

- [JavaScriptのテストに関するベストプラクティス（Zenn）](https://zenn.dev/y_hsgw/articles/07c5f3748afada) - 品質の高いテスト条件、モック境界、命名規約の整理。
- [FAANSにおけるフロントエンドテスト戦略策定の取り組み（ZOZO Tech Blog）](https://techblog.zozo.com/entry/2024-faans-web-test) - フェーズ別テスト戦略と運用指標の実践。
- [フロントエンドテストにおける知見の宝庫（Zenn）](https://zenn.dev/hrbrain/articles/18d4dbc2ac8e4f) - JavaScript testing best practices の要点整理。
- [cypressで始めるE2Eテスト（Qiita）](https://qiita.com/cycle/items/ebbd16935f46d60b0053) - テスト自動化導入順序とCI運用の実践。
