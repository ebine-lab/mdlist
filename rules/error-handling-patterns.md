---
title: "Error Handling Patterns"
description: "フロントエンドのエラー分類・検知・表示・回復・記録の標準パターン / Frontend error handling - classification, detection, display, recovery, logging"
version: "1.4.0"
status: "Stable"
last_updated: "2026-02-25T01:18+09:00"
lang: "ja"
---

# Error Handling Patterns

**説明** - フロントエンド実装を中心に、エラーの分類、検知、表示、回復、記録の標準パターンを定義する。


## 目的

利用者にとって理解可能で、運用者にとって追跡可能なエラー処理を標準化する。  
本規定は「失敗しても安全に回復できる」状態を全画面で実現することを目的とする。

## 背景

HTTP ステータスだけでは、利用者向け案内や機械処理に必要な情報が不足しやすい。  
また、フロントエンドでは通信失敗、入力誤り、描画例外、非同期失敗が混在する。  
そのため、表示文言、再試行、監視記録、API 契約を一体で設計する必要がある。

## 対象

- フロントエンドの通信、入力、描画、状態更新に関するエラー処理。
- 利用者向けエラーメッセージと復旧導線。
- フロントエンドから送信する監視、診断ログ。
- API エラーを受け取るクライアント実装。

## 非対象

- バックエンド内部例外の実装詳細。
- SIEM や監視基盤の製品選定。
- 侵入検知、脆弱性診断手順の詳細。

## 作成開始時点の方針

本ドキュメントの初版作成は次の範囲から開始する。

1. API エラー契約、HTTP ステータス別の画面挙動、再試行規定を先に固定する。
2. 次に表示文言、アクセシビリティ、診断ログの仕様を固定する。
3. 最後にテスト観点と禁止事項を固定し、運用監視へ接続する。

## 確定済み項目

- `code` の業務コード体系と命名規則は API エラー契約とログ出力規定で固定。
- `requestId` と `traceId` の優先利用ルールは通信リクエスト必須メタデータで固定。
- 429 応答時の待機秒数算出ルールはリトライ規定で固定。
- `operationIdRetentionMs`、`deadLetterRetentionMs`、`retentionSweepIntervalMs` の見直し条件は送信データ保持規定で固定。

## 関連規定

- HTTP エラーコードの定義、原因、対処の詳細は [HTTP Error Code Reference](http-error-code-reference.md) を参照する。

## 適合判定スコープ

1. 本文の規定番号を適合判定の正規仕様とする。
2. 実装例は規定との対応確認用サンプルとし、本文規定に矛盾してはならない。
3. テスト規定は本文規定の検証条件とし、本文規定と実装例の双方を満たすことを完了条件とする。

## 要件トレーサビリティマップ

| 要件ID | 対象規定 | 実装例 | テスト規定 |
| --- | --- | --- | --- |
| `EH-COMM-WEB-001` | HTTP クライアント実装規定、タイムアウト規定、リトライ規定 | `api-client.ts`（`requestJson` `requestMutation`） | 単体テスト、統合テスト、E2E テスト |
| `EH-COMM-APP-001` | HTTP クライアント実装規定、タイムアウト規定、リトライ規定、プラットフォームマッピング | `プラットフォームマッピング（Web/APP）` | 統合テスト、E2E テスト |
| `EH-QUEUE-001` | 通信過多と順番待ち規定、バックプレッシャー規定 | `request-queue.ts`、`websocket-backpressure.ts` | 単体テスト、統合テスト、E2E テスト |
| `EH-OUTBOX-001` | 送信データ保持規定、二重送信防止規定 | `delivery-state.ts`、`operation-id.ts` | 単体テスト、統合テスト、E2E テスト |
| `EH-RECV-001` | 二重受信防止規定、ローカルアプリ再接続規定 | `delivery-state.ts`、`local-reconnect.ts` | 単体テスト、統合テスト、E2E テスト |
| `EH-OBS-001` | ログ出力規定、監視と運用規定 | `logging.ts` | 単体テスト、統合テスト |

## 本文

### 基本原則

1. 利用者には「何が起きたか」「どう直すか」を短く提示する。  
2. 開発者には再現に必要な文脈を構造化ログで残す。  
3. 一時障害は再試行可能、恒久障害は代替導線を提示する。  
4. セキュリティ上不要な内部情報を表示しない。  
5. エラーは握り潰さず、必ず分類して処理する。

### エラー分類

| 区分 | 例 | 主な対処 |
| --- | --- | --- |
| 入力エラー | 必須未入力、形式不正 | 該当項目を明示し修正候補を提示 |
| 認証エラー | セッション期限切れ | 再認証導線へ遷移 |
| 権限エラー | 操作権限不足 | 実行不可理由と問い合わせ導線を提示 |
| 通信エラー | ネットワーク断、タイムアウト | 再試行とオフライン案内 |
| サーバーエラー | 500 系 | 一時失敗として再試行案内 |
| 競合エラー | 更新競合 | 最新データ再取得と再入力案内 |
| 描画例外 | UI コンポーネント例外 | 画面単位フォールバックと例外境界で隔離表示 |

### API エラー契約

API エラー形式は RFC 9457 に準拠した `application/problem+json` を標準とする。  
必須項目は次のとおりとする。

- `type`
- `title`
- `status`
- `detail`
- `instance`

運用拡張項目を追加してよい。

- `code`（業務コード）
- `traceId`（追跡識別子）
- `errors`（フィールド単位詳細）

出典・参考情報: RFC 9457: Problem Details for HTTP APIs。

### HTTP エラーコード適用方針

本規定は「フロントエンドがどう振る舞うか」を定義し、  
HTTP ステータスコード自体の定義、原因、対処の詳細は `HTTP Error Code Reference` に委譲する。

| 分類 | フロントエンド既定動作 |
| --- | --- |
| 入力・業務不整合（400/422） | 入力エラーとして扱い、該当項目への誘導と修正候補を提示する。 |
| 認証・認可（401/403） | 再認証導線または権限不足表示へ遷移し、操作継続を停止する。 |
| 不存在・競合（404/409） | 戻る導線または再読込導線を提示し、状態を再同期する。 |
| レート制限（429） | 待機時間を明示し、サーバー指示がある場合はそれを優先する。 |
| サーバー障害（5xx） | 一時障害として扱い、手動再試行導線と問い合わせ導線を提示する。 |

出典・参考情報: RFC 9110: HTTP Semantics、HTTP response status codes | MDN Web Docs、429 Too Many Requests | MDN Web Docs。

### データサーバー通信エラー詳細分類

アプリとデータサーバー間の通信失敗は、次の区分で扱う。

| 種別 | 代表的な検知条件 | 画面既定動作 | 自動再試行 |
| --- | --- | --- | --- |
| ネットワーク断 | `fetch` が `TypeError` で失敗 | 通信不能を明示し、手動再試行導線を表示 | 冪等操作または同一 `operationId` 更新系のみ可 |
| タイムアウト | `AbortSignal.timeout()` で中断 | 「時間超過」と再試行導線を表示 | 冪等操作または同一 `operationId` 更新系のみ可 |
| レート制限 | HTTP 429 | 待機秒数を表示し、次回再試行予定を示す | `Retry-After` に従う |
| 一時サーバー障害 | HTTP 502/503/504 | 一時障害を表示し、自動再試行中を明示 | 冪等操作または同一 `operationId` 更新系のみ可 |
| 応答形式不正 | JSON 解析失敗、想定外 `content-type` | 障害扱いで問い合わせ導線へ誘導 | 原則不可 |
| 利用者キャンセル | 画面遷移、明示的キャンセル | エラー表示を出さず操作中断を確定 | 不可 |

### 通信リクエスト必須メタデータ

データサーバー通信の追跡性を担保するため、次の情報をすべて保持する。

- `requestId` または `traceId`（サーバーから返却された識別子）
- `clientRequestId`（クライアント側で生成する識別子）
- `endpoint`（パス。クエリ値は必要最小限）
- `method`
- `durationMs`
- `attempt` と `maxRetries`

### HTTP クライアント実装規定（Web/APP）

HTTP クライアントはプラットフォーム差異に依存せず、`status` とエラー種別で判定する。  
Web と APP の代表実装は次を使用する。

| プロファイル | 代表 API |
| --- | --- |
| `Web` | `fetch()` |
| `APP` | iOS `URLSession`、Android `OkHttp` または同等 API |

1. 通信失敗と HTTP エラーを同一扱いにしない。  
2. HTTP エラー判定は `status` で行い、`problem+json` を優先解析する。  
3. 利用者表示文言は API 文言をそのまま表示しない。  
4. `204` と `205` は正常完了として扱い、JSON 本文を必須にしない。
5. 更新系は `requestMutation` などの専用関数に分離し、戻り値契約を `void` で固定する。

出典・参考情報: Window: fetch() method、Using the Fetch API、Response: ok property、URLSession。

### タイムアウト規定（データサーバー通信）

1. タイムアウト制御はプラットフォーム標準 API を使用する。  
   - `Web` は `AbortSignal.timeout()` を使用する。  
   - `APP` は iOS と Android の標準タイムアウト設定とキャンセル API を使用する。  
2. 既定タイムアウトは 8 秒とし、画面特性に応じて個別設定する。  
3. 読み取り系の上限は 15 秒、更新系の上限は 30 秒を超えない。  
4. 画面遷移時や利用者キャンセル時は未完了リクエストを中断する。  
5. タイムアウトと利用者キャンセルは別エラーとして分類する。
6. 利用者キャンセルは「障害」ではなく「中断」として扱い、自動再試行しない。  
7. 利用者キャンセルの対象は明示キャンセル操作、画面遷移やアンマウント、後続操作のための旧要求中断とする。  
8. 利用者キャンセル時は `failed` や `dead-letter` へ遷移させず、`in-flight` 解放とローディング解除のみ実施する。
9. 更新系のタイムアウト時は `operationId` で状態照会を先行し、`acked` なら成功確定とする。  
10. 状態照会結果が `not_found` の場合のみ同一 `operationId` で再送可とする。  
11. 状態照会不能時は `unknown` として扱い、自動再送を停止し、手動再試行または問い合わせへ遷移する。  
12. 再送が必要な場合も新規 `operationId` の発行を禁止し、同一 `operationId` を使用する。

出典・参考情報: AbortSignal: timeout() static method、Cancellation and timeouts、URLSessionConfiguration.timeoutIntervalForRequest、URLSessionTask.cancel()。

### リトライ規定（再試行）

1. 自動再試行は原則 `GET`、`HEAD`、`PUT`、`DELETE`、`OPTIONS` など冪等操作に限定する。  
2. 更新系は同一 `operationId` を付与した場合のみ自動再試行を許可する。  
3. 更新系タイムアウトは状態照会結果が `not_found` の場合のみ自動再送する。  
4. 再試行対象は通信断、タイムアウト、408、429、502、503、504 とする。  
5. 401、403、404、409、422 は自動再試行しない。  
6. 429、503 で `Retry-After` がある場合、次の優先順で待機秒数を算出する。
   - 秒数形式はその値を使用する。
   - 日時形式は `現在時刻との差分` を使用する。
   - 不正値は指数バックオフへフォールバックする。
7. 待機は指数バックオフとジッターを使用する。既定値は `base=300ms`、`maxRetries=2` とする。  
8. 3 回目の失敗で自動再試行を停止し、手動再試行導線へ切り替える。  
9. 自動再試行中は画面に「再接続中」を表示し、利用者操作による手動再試行導線を常に提供する。

出典・参考情報: RFC 9110: HTTP Semantics、429 Too Many Requests | MDN Web Docs。

### 通信過多と順番待ち規定（過負荷制御）

1. クライアントはエンドポイント単位で同時実行数上限を持つ。  
2. 送信待ちキューは無制限にしない。  
3. キュー満杯時は即時失敗させ、利用者へ「順番待ち超過」を表示する。  
4. キュー優先度は `interactive` と `background` を分離し、`interactive` を先に処理する。  
5. 期限超過した待ちリクエストは送信せず破棄し、再操作を促す。  
6. 同一読取要求は `requestKey` で重複統合し、同一結果を複数画面で共有する。  
7. 429、503、接続拒否、接続リセットは過負荷シグナルとして扱う。  
8. 過負荷シグナル連続時は短時間の送信停止窓を設け、復帰後に段階的に再開する。
9. 実効処理率は `μ_effective = min(μ_client, μ_server, μ_backbone)` で評価し、基幹回線を含む最小値で制御する。  
10. `maxQueue` は固定値で決め打ちせず、劣化時プロファイルの `λ_peak`、`μ_effective`、許容待機時間 `T` から `maxQueue >= (λ_peak - μ_effective) * T` を基準に決定する。  
11. `μ_backbone` は帯域、RTT、ジッター、パケット損失、再送率の劣化で低下する前提で扱う。  
12. 回線劣化で `μ_backbone` が閾値未満になった場合、`background` を先に間引き、`interactive` のみ段階送信へ切り替える。
13. 算出結果が 0 以下となる場合は `maxQueue=1` を下限として適用する。
14. HTTP 429/503 を受信した場合は `onOverloadSignal` で `request-queue.ts` の `recordOverloadSignal` を呼び出し、送信停止窓と段階復帰を開始する。

環境別既定プロファイルは次を使用する。

| プロファイル | `maxInFlight` | `maxQueue` | `maxQueueWaitMs` |
| --- | --- | --- | --- |
| `development` | 2 | 20 | 5000 |
| `staging` | 4 | 60 | 8000 |
| `production` | 6 | 100 | 10000 |

`production` を既定プロファイルとする。

### 非同期受領規定（202 Accepted）

1. 202 は「受領済みで未完了」を意味し、完了成功として扱わない。  
2. 202 応答本文は処理状態確認に必要な情報を含む。最小要素は `statusUrl` または同等の監視情報とする。  
3. 202 の状態確認は固定間隔ポーリングではなく、指数バックオフで実行する。  
4. 状態確認の上限時間を超えた場合は失敗へ遷移し、利用者に再実行または問い合わせ導線を提示する。

出典・参考情報: RFC 9110: HTTP Semantics。

### バックプレッシャー規定（WebSocket とストリーム）

1. WebSocket API はバックプレッシャーを提供しない前提で設計する。  
2. 送信バッファは `WebSocket.bufferedAmount` を監視し、しきい値超過時は送信を一時停止する。  
3. `highWatermark` 超過中は低優先度メッセージ送信を停止し、`lowWatermark` 以下で再開する。  
4. `send()` 実行時にバッファ溢れで接続が閉じられた場合、過負荷断として再接続へ遷移する。  
5. 実行環境が Streams API に対応する場合、内部キューは backpressure を尊重する。  
6. `WebSocketStream` は互換性制約を確認した上でのみ採用し、非対応環境の代替経路を必須化する。
7. `sendWithBackpressure` が `reconnect_required` を返した場合は再接続フローを即時開始する。

出典・参考情報: WebSocket API、WebSocket: bufferedAmount property、WebSocket: send() method、Streams Standard。

### プラットフォームマッピング（Web/APP）

| 観点 | `Web` | `APP（iOS/Android）` |
| --- | --- | --- |
| HTTP 通信 | `fetch()` | `URLSession` / `OkHttp` または同等 API |
| タイムアウト | `AbortSignal.timeout()` | リクエストタイムアウト設定 + キャンセル API |
| 利用者キャンセル | `AbortController.abort()` | iOS `Task.cancel` 相当 / Android `Job.cancel` 相当 |
| オンライン判定 | `navigator.onLine` + 疎通確認 | OS 接続監視 API + 疎通確認 |
| 背景再送 | Service Worker + Background Sync | iOS `BGTaskScheduler` / Android `WorkManager` |
| 離脱時診断送信 | `sendBeacon` / `fetch keepalive` | 背景転送 API または永続キュー + 背景ワーカー |
| 過負荷シグナル入力 | HTTP 429/503 | HTTP 429/503 + 接続拒否/接続リセット |
| 描画例外隔離 | React `ErrorBoundary` | 画面単位フォールバック + グローバル例外捕捉 |

### オフラインと復旧規定

1. オフライン検知時は即時に通信を打ち切り、オフライン状態を画面表示する。  
2. 復旧検知時は自動再取得を一度だけ実行し、連続失敗時は手動再試行へ切り替える。  
3. 入力中データは破棄せず、復旧後の再送信に備えて保持する。  
4. 送信系操作は二重送信防止のため、操作単位の一意識別子を持たせる。
5. `Web` は `navigator.onLine` の値だけで接続可否を断定せず、疎通確認要求で復旧判定する。  
6. `APP` は OS の接続監視 API だけで接続可否を断定せず、疎通確認要求で復旧判定する。  
7. 背景再送は `Web` では Service Worker + Background Sync、`APP` では OS 背景タスク機構を優先する。
8. クライアント側の不安定（高遅延・高損失）を検知した場合は `maxInFlight=1` に一時的に絞り、指数バックオフと手動再試行導線を表示する。復旧後は通常プロファイルへ段階的に戻す。  
9. 無線切替による不安定判定の参考閾値（実装時に調整可）: `RTT p95 > 500ms` または `packetLossRate > 2%` が 30 秒継続したら不安定とみなし 8 を適用する。  
10. モバイル背景制約で送信できない場合は Snackbar 等で「送信保留中」を即時表示し、再送ボタン（ユーザー明示再送）を提示する。OSの復帰通知後に再送するまで保留を維持する。  

出典・参考情報: Window: online event、Background Synchronization API、NWPathMonitor、ConnectivityManager、BGTaskScheduler、WorkManager。

### 送信データ保持規定（通信不能時）

1. 送信系操作は送信前に永続キューへ書き込み、保存成功後に初回送信する。  
2. キュー要素は少なくとも次を保持する。  
   - `operationId`（操作単位の一意識別子）  
   - `idempotencyKey`（再送時も不変）  
   - `payloadDigest`（内容同一性検証）  
   - `createdAt`、`attempt`、`state`  
3. 状態は `queued`、`sending`、`acked`、`failed`、`dead-letter` で管理する。  
4. アプリ再起動後は `queued` と `sending` を再評価し、未確定送信のみ再開する。  
5. `acked` は即時削除せず、`operationIdRetentionMs` で設定した期間を重複防止窓として保持してから削除する。  
6. `dead-letter` は `deadLetterRetentionMs` 経過後に削除対象とする。  
7. 保持と破棄の制御は次の設定項目で管理する。  
   - `operationIdRetentionMs`  
   - `deadLetterRetentionMs`  
   - `retentionSweepIntervalMs`  
8. 設定値は環境ごとに変更可能とし、0 以下の値を禁止する。  
9. 保存容量上限を超える場合は古い `dead-letter` から削除し、未送信データを優先保持する。
10. `operationIdRetentionMs` は `operationIdPoolExhaustedCount > 0` または重複送信再発時に延長を見直す。  
11. `deadLetterRetentionMs` は調査要求遅延の p95 が保持期間の 80% を超えた場合に延長を見直す。  
12. `retentionSweepIntervalMs` は掃除処理時間 p95 が 200ms 超で延長し、削除遅延 p95 が `2 * retentionSweepIntervalMs` 超で短縮を見直す。

環境別既定プロファイルは次を使用する。

| プロファイル | `operationIdRetentionMs` | `deadLetterRetentionMs` | `retentionSweepIntervalMs` |
| --- | --- | --- | --- |
| `development` | 600000（10分） | 86400000（1日） | 15000（15秒） |
| `staging` | 1800000（30分） | 259200000（3日） | 30000（30秒） |
| `production` | 3600000（1時間） | 604800000（7日） | 60000（60秒） |

`production` を既定プロファイルとする。

### ローカルアプリ再接続規定

1. 接続状態は `disconnected`、`connecting`、`connected`、`degraded` で管理する。  
2. 断線時の再接続は指数バックオフとジッターを使用し、待機上限を 30 秒とする。  
3. 再接続ハンドシェイクで次を必ず交換する。  
   - `clientInstanceId`  
   - `sessionId`  
   - `lastAckedOperationId`  
   - `lastAppliedEventSeq`  
4. 再接続成功後は `lastAckedOperationId` 以降の未確定送信だけを再送する。  
5. `lastAppliedEventSeq` に欠番がある場合、差分同期ではなく全体再同期を優先する。  
6. 再接続不能が継続する場合はローカル機能を縮退運転に切り替え、利用者へ状態を表示する。
7. ハンドシェイクで `peerMaxInFlight` と `peerQueueDepth` を受け取り、送信速度を動的調整する。

### 二重送信防止規定

1. 更新系リクエストには必ず `operationId` を付与し、再試行時も同じ値を使用する。  
2. HTTP 送信では `Idempotency-Key` として `operationId` を送る。  
3. 同一 `operationId` の並行送信をクライアントで禁止し、送信中フラグで抑止する。  
4. タイムアウトで結果不明となった送信は、状態照会可能なら照会後に再送可否を決める。  
5. 状態照会不能または `unknown` の場合は自動再送を禁止し、同一 `operationId` の手動再試行のみ許可する。  
6. サーバー応答の `acked` を受けるまで操作完了と見なさない。
7. 送信が完了した内容はエラー時を除き全入力欄・進捗表示をクリアする（完了後の重複送信防止）。  
8. `operationId` は再利用可能なスロット方式で払い出し、識別子空間の枯渇を回避する。  
9. 再利用は `operationIdRetentionMs` 経過後かつ `queued` `sending` `failed` の未確定状態がない場合のみ許可する。  
10. 再利用候補がない場合は `operation_id_pool_exhausted` で即時失敗し、待機後の再試行を促す。  
11. `operationId` は `clientInstanceId` と再利用スロット識別子を含む構成を必須とする。
12. `slotCount` は環境別プロファイルで固定し、`production` を既定値として使用する。
13. `slotCount` の処理能力は `C_slot = slotCount / τ_ret` で評価し、`τ_ret = operationIdRetentionMs / 1000`（秒）で算出する。  
14. 観測窓を `N` サンプル（間隔 `Δt`）としたとき、`λ_peak` の連続超過率を `R_cont = (1 / (N - 1)) * Σ_{i=2..N} I(λ_peak_i > C_slot ∧ λ_peak_{i-1} > C_slot)` で定義する。  
15. `R_cont >= 0.1` が 2 観測窓連続、または `operation_id_pool_exhausted` が 1 回でも発生した場合、`slotCount` 見直しを必須とする。  
16. 見直し後の目標値は `slotCount_target = ceil(λ_peak_p99 * τ_ret * S)` で算出し、`S` は安全率として既定 `1.2` を使用する。
17. 観測パラメータは `N=60`、`Δt=5` 秒で固定し、観測窓は 5 分とする。  
18. `λ_peak_i` はサンプル `i` の受理更新件数を `Δt` で除算した値（件/秒）とする。  
19. `λ_peak_p99` は直近 24 時間の `λ_peak_i` を母集団とした p99 で算出する。
20. 利用者キャンセル時は `operationId` スロットを `cancelReuseCooldownMs` の短時間クールダウン後に再利用可能とする。  
21. `cancelReuseCooldownMs` の経過前は同一スロット再割当を禁止し、連続キャンセルによる過剰再投入を抑制する。  
22. 連続キャンセルの負荷試験で `operation_id_pool_exhausted` と再送遅延の監視値を検証し、必要時は `cancelReuseCooldownMs` を見直す。

| プロファイル | `slotCount` | `cancelReuseCooldownMs` |
| --- | --- | --- |
| `development` | 4096 | 3000 |
| `staging` | 16384 | 5000 |
| `production` | 65536 | 10000 |

`production` を既定プロファイルとする。

### 二重受信防止規定

1. サーバーまたはローカルアプリから受信するイベントは `eventId` と `eventSeq` を必須とする。  
2. クライアントは適用済み `eventId` を保持し、重複受信時は再適用せず破棄する。  
3. `eventSeq` が `lastAppliedEventSeq` 以下なら `duplicate` として破棄する。  
4. `eventSeq = lastAppliedEventSeq + 1` の場合のみ `apply` として適用する。  
5. `eventSeq > lastAppliedEventSeq + 1` または順序逆転を検知した場合は `resync_required` として後続イベント適用を停止し再同期を要求する。  
6. 状態更新処理は同一入力で同一結果となる冪等実装を必須とする。  
7. 受信確認 `ack` はローカル永続化と状態反映の完了後に送信する。  
8. 送信ボタンの進捗UXを次のステップで統一し重複押下を防止する。  
   1) 送信前: 表記は「送信」  
   2) 押下直後: バリデーションとエラーチェックを実行し、失敗時はステップ1に戻す。成功時のみボタン内でスピナー＋「送信中」を表示  
   3) 完了時: ボタン内で「送信完了」を表示  
   4) 必要に応じて disabled を併用する  
   5) 送信失敗時は元の入力を保持したままステップ1に戻す。成功時は全入力項目と進捗表示をクリアする（必ず全フィールドをクリア）  

### 送受信状態遷移モデル

| 対象 | 現在状態 | 条件 | 遷移先 | 備考 |
| --- | --- | --- | --- | --- |
| 送信 | `queued` | 送信開始 | `sending` | `operationId` は同一値を保持 |
| 送信 | `sending` | 応答 `acked` | `acked` | 保持期間経過後に削除 |
| 送信 | `sending` | 一時失敗 | `failed` | 再送時も同一 `operationId` |
| 送信 | `failed` | 再送投入 | `queued` | `operationId` 変更禁止 |
| 送信 | `failed` | 試行上限到達 | `dead-letter` | 再送停止 |
| 受信 | `lastAppliedEventSeq` | `eventSeq <= lastAppliedEventSeq` | `duplicate` | 破棄 |
| 受信 | `lastAppliedEventSeq` | `eventSeq = lastAppliedEventSeq + 1` | `apply` | 適用 |
| 受信 | `lastAppliedEventSeq` | `eventSeq > lastAppliedEventSeq + 1` または順序逆転 | `resync_required` | 後続適用停止 |

### 画面表示規定

1. エラー文言は原因と対処を一文ずつ提示する。  
2. 「不明なエラーが発生しました」のみで終了しない。  
3. 主要導線には次の操作を一つ提示する。
   - 再試行
   - 前画面に戻る
   - 問い合わせ
4. 利用者に責任を転嫁する文言を禁止する。
5. 利用者キャンセル時は障害トーストを表示せず、必要な場合のみ中断通知を簡潔に表示する。  
6. 監視集計では利用者キャンセルをエラー率に含めず、`canceled` 系指標として分離集計する。

### フォームエラー規定

1. エラー項目を特定し、項目名と理由をテキストで示す。  
2. 修正候補が分かる場合は必ず提案を併記する。  
3. 送信失敗時はエラー要約をフォーム先頭に表示する領域を必ず確保する（表示位置の詳細はUIデザインに従う）。  
4. フォーカスを最初のエラー項目へ移動する。  
5. バリデーションは送信前チェックを必須とし、逐次検証は必要最小限（例: フォーム離脱や送信直前）に限定する。  

### アクセシビリティ規定

1. エラー状態を色のみに依存して伝えない。  
2. WCAG 3.3.1 に従い、何が誤りかをテキストで示す。  
3. WCAG 3.3.3 に従い、修正提案が可能なら提示する。  
4. 動的エラーメッセージは `role="alert"` または `aria-live` で通知する。

出典・参考情報: Understanding SC 3.3.1: Error Identification、Understanding SC 3.3.3: Error Suggestion、Technique ARIA19。

### 描画例外規定

1. 画面全体を一括で落とさず、画面単位フォールバックで影響範囲を限定する。  
2. `Web` は Error Boundary でフォールバック表示とログ送信を実装する。  
3. `APP` は画面単位フォールバックとグローバル例外捕捉の併用を必須とする。  
4. 非同期例外とイベント処理例外は UI 例外境界の捕捉対象外として別処理を設計する。

出典・参考情報: Component | React。

### ログ出力規定

フロントエンドログは JSON 形式で共通スキーマを次の2段で運用する。

#### エッセンシャルログ

- `timestamp`
- `level`
- `errorType`
- `code`
- `route`
- `endpoint`
- `method`
- `durationMs`
- `attempt`
- `maxRetries`
- `requestId` または `traceId`
- `clientRequestId`
- `operationId`（更新系は必須 読取系は省略可）
- `release`

#### フルログ

- `queueDepth`
- `queueWaitMs`
- `inFlight`
- `droppedReason`
- `backpressureState`
- `highWatermark`
- `lowWatermark`
- `eventId`
- `eventSeq`
- `connectionState`
- `sessionId`
- `lastAppliedEventSeq`
- `userAction`

ログのPII除外方針: 次の個人情報は記録禁止（ハッシュも不可）。`name`、`email`、`phone`、`address`、`birthdate`、クレジットカード・口座番号、認証トークン、パスワード、ワンタイムコード、GPS 生の座標、顔画像・指紋等の生体情報。ユーザー入力欄をログする場合は必ずマスキングまたは未記録とする。  

### 診断送信規定

1. `Web` のページ離脱直前の診断データ送信は `navigator.sendBeacon()` を優先する。  
2. `Web` で応答内容が必要な通信は `fetch(..., { keepalive: true })` を使用する。  
3. `APP` では即時送信に失敗しても喪失しないよう、永続キューへ保存して背景タスクで再送する。

出典・参考情報: Navigator: sendBeacon() method、Using the Fetch API、BGTaskScheduler、WorkManager。

### セキュリティ規定

1. 利用者向け画面にスタックトレースを表示しない。  
2. 内部ホスト名、SQL、トークン断片を露出しない。  
3. `problem+json` の `detail` は攻撃補助情報を含めない。  
4. ログには個人情報を平文で保存しない。

出典・参考情報: OWASP Error Handling Cheat Sheet、OWASP Logging Cheat Sheet。

### 監視と運用規定

1. エラー率を画面種別と操作種別で監視する。  
2. 突発増加時はリリース、外部依存、地域障害の順で切り分ける。  
3. 上位エラーコードは週次で削減計画を更新する。
4. 次の通信過負荷指標を必須監視とする。
   - `queueDepth` の p95
   - `queueWaitMs` の p95
   - `overloadResponses`（429/503）
   - `connectionDropRate`
   - `duplicateSuppressedCount`
   - `operationIdPoolExhaustedCount`
   - `lambdaPeakContinuousExceedRate`
   - `backboneRttMs` の p95
   - `backboneJitterMs` の p95
   - `packetLossRate`
   - `retransmitRate`
5. `queueWaitMs p95 > 3000` または `connectionDropRate > 1%` または `packetLossRate > 0.5%` が 5 分継続した場合、負荷抑制モードへ移行する。  
6. 負荷抑制モードでは background 系送信を間引き、interactive 系を優先する。  
7. `backboneRttMs` と `packetLossRate` が復帰閾値を連続 5 分満たすまで通常モードへ戻さない。  
8. 監視値は平常時だけでなく劣化時プロファイルで基準化し、`maxQueue` と `maxInFlight` の見直しに反映する。
9. `lambdaPeakContinuousExceedRate = (1 / (N - 1)) * Σ_{i=2..N} I(λ_peak_i > C_slot ∧ λ_peak_{i-1} > C_slot)` を算出し、`>= 0.1` が 2 観測窓連続または `operationIdPoolExhaustedCount > 0` で `slotCount` を見直す。  
10. 観測パラメータは `N=60`、`Δt=5` 秒、`λ_peak_i = acceptedMutations_i / Δt` で固定する。

### テスト規定

#### 単体テスト

- `ok` 判定分岐。
- タイムアウト分岐。
- `problem+json` 解析失敗時のフォールバック。
- 再試行対象ステータスと非対象ステータスの分岐。
- `Retry-After` の秒数形式、日時形式、不正値の分岐。
- 試行ごとに独立したタイムアウト signal が生成されること。
- `204` と `205` を正常完了として扱うこと。
- 更新系タイムアウト時に状態照会結果 `acked` `not_found` `unknown` で分岐すること。
- 同一 `operationId` 再送時に新規送信扱いにならないこと。
- 更新系の自動再試行が同一 `operationId` 条件でのみ実行されること。
- `operationId` の再利用条件を満たさない間は同一スロットが再割当されないこと。
- 利用者キャンセル時に `cancelReuseCooldownMs` 経過前は同一スロットが再割当されないこと。
- 利用者キャンセル時に `cancelReuseCooldownMs` 経過後は同一スロットが再割当されること。
- 利用者キャンセル時に `inflightOperationIds` が解放され `queued` に戻ること。
- `operation_id_pool_exhausted` 発生時に再試行待機導線へ遷移すること。
- `slotCount` 範囲外の `operationId` がスロット状態を更新しないこと。
- `development` `staging` `production` の各プロファイルで `slotCount` が想定値に固定されること。
- `lambdaPeakContinuousExceedRate` が閾値を超過した場合に `slotCount` 見直し判定が発火すること。
- `requestId` または `traceId` の少なくとも一方が必須であること。
- `requestKey` 重複統合が `read` のみで実行され `mutation` では統合されないこと。
- `computeRecommendedMaxQueue` の返却値が常に 1 以上であること。
- `eventSeq <= lastAppliedEventSeq` は `duplicate` 判定になること。
- `eventSeq = lastAppliedEventSeq + 1` は `apply` 判定になること。
- `eventSeq > lastAppliedEventSeq + 1` は `resync_required` 判定になること。
- 同一 `eventId` 再受信時に状態を再適用しないこと。
- `maxInFlight` 超過時に待ちキューへ遷移すること。
- `maxQueue` 超過時に即時失敗し `droppedReason` が記録されること。
- `development` `staging` `production` の各プロファイルで `maxInFlight` `maxQueue` `maxQueueWaitMs` が想定値に固定されること。
- `bufferedAmount` が `highWatermark` を超えたとき送信停止すること。
- `highWatermark` 超過中に `low` 優先度のみ停止し `high` は継続送信できること。
- `sendWithBackpressure` が `reconnect_required` を返す条件で再接続フローへ遷移すること。
- `μ_backbone` 低下を与えたとき `maxQueue` 判定と負荷抑制判定が発火すること。
- `operationIdRetentionMs` 経過後に `acked` が削除対象になること。
- `deadLetterRetentionMs` 経過後に `dead-letter` が削除対象になること。
- `retentionSweepIntervalMs` 条件を満たした時だけ掃除処理が実行されること。
- `development` `staging` `production` の各プロファイルで保持設定値が想定どおりに選択されること。

#### 統合テスト

- 401、403、409、422、429、500 の画面挙動。
- フォームエラー時のフォーカス移動。
- `Web` は Error Boundary のフォールバック表示で継続動作すること。
- `APP` は画面単位フォールバックとグローバル例外捕捉で継続動作すること。
- `Web` と `APP` で同一要件IDの成功条件と失敗条件が一致すること。
- オフライン検知から復旧時の自動再取得挙動。
- 自動再試行中の表示文言と手動再試行導線の整合性。
- アプリ再起動後に永続キューから未確定送信のみ再開すること。
- 再接続ハンドシェイク後に `lastAckedOperationId` 以前を再送しないこと。
- `operationId` スロット再利用時に重複防止窓を超えたIDのみ再利用されること。
- 受信イベントの欠番検知時に再同期へ遷移すること。
- 202 受領後に完了確認フローへ遷移し、即時成功扱いしないこと。
- 過負荷シグナル連続時に送信停止窓へ移行し、段階復帰すること。
- WebSocket 送信で接続閉塞を検知した場合に再接続へ遷移すること。
- 回線劣化プロファイルで `packetLossRate` 超過時に background 間引きへ遷移すること。

#### E2E テスト

- 通信断時の再試行導線。
- 送信失敗時の入力保持。
- 離脱時の診断送信。
- 429 応答で `Retry-After` 指定がある場合の待機挙動。
- 502/503/504 の一時障害時に上限回数で停止する挙動。
- 送信直後にアプリを再起動しても二重送信せずに送信再開できること。
- `operationId` プール枯渇時に `operation_id_pool_exhausted` が表示され待機後に再試行できること。
- ローカルアプリ再接続後に重複イベントで画面状態が二重更新されないこと。
- 通信過多状態で `interactive` が `background` より先に処理されること。
- キュー待ち時間上限超過時に古い待ち要求が破棄されること。
- 基幹回線遅延と損失を注入した条件で `maxQueue` 超過時の失敗導線と復帰導線が維持されること。

### 禁止事項

- 例外を `console.error` のみで終了すること。
- 画面文言とログ文言を無関係に運用すること。
- HTTP status を見ずに成功扱いすること。
- API 失敗の詳細を利用者へ無加工表示すること。
- タイムアウト後に新規 `operationId` で同一操作を再送すること。
- `eventId` と `eventSeq` を持たない受信データを適用すること。
- 同時実行数とキュー上限を未設定のまま運用すること。
- 過負荷時に固定間隔で再送し続けること。
- 202 応答を処理完了として確定表示すること。

## 実装例

この章のコードは `Web` プロファイルの準拠実装例を中心に、再接続とキュー制御は `Web/APP` 共通ロジック例を含む。`APP` は本文の `プラットフォームマッピング（Web/APP）` で同一要件を実装する。

```ts
// api-client.ts
// RFC 9457準拠のエラー詳細
export type ProblemDetails = {
  type?: string;
  title?: string;
  status?: number;
  detail?: string;
  instance?: string;
  code?: string;
  traceId?: string;
  errors?: Array<{ field: string; message: string }>;
};

// API呼び出し失敗を表すアプリケーション例外
export class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public problem?: ProblemDetails,
    public retryable = false
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

type RequestJsonOptions = RequestInit & {
  timeoutMs?: number;
  maxRetries?: number;
  retryBaseMs?: number;
  onOverloadSignal?: (signal: { kind: 'http'; status: 429 | 503 }) => void;
};

type MutationMethod = 'POST' | 'PUT' | 'PATCH' | 'DELETE';

type RequestMutationOptions = Omit<RequestJsonOptions, 'method'> & {
  method?: MutationMethod;
  operationId: string;
  resolveOperationStatus: (operationId: string) => Promise<OperationStatus>;
  onUserCanceled?: (operationId: string) => void;
};

type OperationStatus = 'acked' | 'not_found' | 'unknown';

const RETRYABLE_STATUS = new Set([408, 429, 502, 503, 504]);
const IDEMPOTENT_METHOD = new Set(['GET', 'HEAD', 'PUT', 'DELETE', 'OPTIONS']);
const MUTATION_METHOD = new Set(['POST', 'PUT', 'PATCH', 'DELETE']);

function isIdempotentMethod(method: string): boolean {
  return IDEMPOTENT_METHOD.has(method.toUpperCase());
}

function isMutationMethod(method: string): boolean {
  return MUTATION_METHOD.has(method.toUpperCase());
}

function parseRetryAfterMs(value: string | null): number | undefined {
  if (!value) {
    return undefined;
  }

  const seconds = Number(value);
  if (Number.isFinite(seconds) && seconds >= 0) {
    return seconds * 1000;
  }

  const dateMs = Date.parse(value);
  if (!Number.isNaN(dateMs)) {
    return Math.max(0, dateMs - Date.now());
  }

  return undefined;
}

function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function calcBackoffMs(attempt: number, baseMs: number): number {
  const jitter = Math.floor(Math.random() * baseMs);
  return baseMs * 2 ** attempt + jitter;
}

function buildRequestSignal(timeoutMs: number, externalSignal?: AbortSignal): AbortSignal {
  const timeoutSignal = AbortSignal.timeout(timeoutMs);
  if (!externalSignal) {
    return timeoutSignal;
  }

  return AbortSignal.any([externalSignal, timeoutSignal]);
}

async function parseProblemDetails(response: Response): Promise<ProblemDetails | undefined> {
  const contentType = response.headers.get('content-type') ?? '';
  if (!contentType.includes('application/problem+json')) {
    return undefined;
  }

  try {
    return (await response.json()) as ProblemDetails;
  } catch {
    return undefined;
  }
}

async function parseSuccessJson<T>(response: Response): Promise<T> {
  if (response.status === 204 || response.status === 205) {
    return undefined as T;
  }

  const responseContentType = response.headers.get('content-type') ?? '';
  if (!responseContentType.includes('application/json')) {
    throw new ApiError('Unexpected response format', response.status, undefined, false);
  }

  return (await response.json()) as T;
}

// 読み取り系を中心にJSON APIを呼び出す
export async function requestJson<T>(
  input: RequestInfo,
  options: RequestJsonOptions = {}
): Promise<T> {
  const {
    timeoutMs = 8000,
    maxRetries = 2,
    retryBaseMs = 300,
    method = 'GET',
    onOverloadSignal,
    ...init
  } = options;

  const methodText = method.toUpperCase();
  const retryableMethod = isIdempotentMethod(methodText);

  for (let attempt = 0; ; attempt += 1) {
    const combinedSignal = buildRequestSignal(timeoutMs, init.signal);
    try {
      const response = await fetch(input, {
        ...init,
        method: methodText,
        signal: combinedSignal,
        headers: {
          'Accept': 'application/json, application/problem+json',
          ...(init.headers ?? {}),
        },
      });

      if (!response.ok) {
        if (response.status === 429 || response.status === 503) {
          onOverloadSignal?.({ kind: 'http', status: response.status as 429 | 503 });
        }
        const problem = await parseProblemDetails(response);
        const retryable = retryableMethod && RETRYABLE_STATUS.has(response.status);
        if (retryable && attempt < maxRetries) {
          const retryAfterMs = parseRetryAfterMs(response.headers.get('retry-after'));
          await sleep(retryAfterMs ?? calcBackoffMs(attempt, retryBaseMs));
          continue;
        }

        throw new ApiError(
          problem?.title ?? 'Request failed',
          response.status,
          problem,
          retryable
        );
      }

      return await parseSuccessJson<T>(response);
    } catch (error: unknown) {
      const isTimeout = error instanceof DOMException && error.name === 'TimeoutError';
      const isUserCanceled = error instanceof DOMException && error.name === 'AbortError';
      const isNetworkError = error instanceof TypeError;
      const retryable = retryableMethod && (isTimeout || isNetworkError);

      if (isUserCanceled) {
        throw new ApiError('Request canceled by user', 499, undefined, false);
      }

      if (retryable && attempt < maxRetries) {
        await sleep(calcBackoffMs(attempt, retryBaseMs));
        continue;
      }

      if (isTimeout) {
        throw new ApiError('Request timed out', 408, undefined, retryable);
      }

      if (isNetworkError) {
        throw new ApiError('Network request failed', 0, undefined, retryable);
      }

      throw error;
    }
  }
}

// 更新系をoperationId前提で呼び出す
export async function requestMutation(
  input: RequestInfo,
  options: RequestMutationOptions
): Promise<void> {
  const {
    timeoutMs = 8000,
    maxRetries = 2,
    retryBaseMs = 300,
    method = 'POST',
    operationId,
    resolveOperationStatus,
    onUserCanceled,
    onOverloadSignal,
    ...init
  } = options;

  const methodText = method.toUpperCase();
  if (!isMutationMethod(methodText)) {
    throw new ApiError('Mutation method is required', 400, undefined, false);
  }

  for (let attempt = 0; ; attempt += 1) {
    const combinedSignal = buildRequestSignal(timeoutMs, init.signal);
    try {
      const response = await fetch(input, {
        ...init,
        method: methodText,
        signal: combinedSignal,
        headers: {
          'Accept': 'application/json, application/problem+json',
          'Idempotency-Key': operationId,
          'X-Operation-Id': operationId,
          ...(init.headers ?? {}),
        },
      });

      if (!response.ok) {
        if (response.status === 429 || response.status === 503) {
          onOverloadSignal?.({ kind: 'http', status: response.status as 429 | 503 });
        }
        const problem = await parseProblemDetails(response);
        const retryable = RETRYABLE_STATUS.has(response.status);
        if (retryable && attempt < maxRetries) {
          const retryAfterMs = parseRetryAfterMs(response.headers.get('retry-after'));
          await sleep(retryAfterMs ?? calcBackoffMs(attempt, retryBaseMs));
          continue;
        }

        throw new ApiError(
          problem?.title ?? 'Request failed',
          response.status,
          problem,
          retryable
        );
      }

      return;
    } catch (error: unknown) {
      const isTimeout = error instanceof DOMException && error.name === 'TimeoutError';
      const isUserCanceled = error instanceof DOMException && error.name === 'AbortError';
      const isNetworkError = error instanceof TypeError;

      if (isUserCanceled) {
        onUserCanceled?.(operationId);
        throw new ApiError('Request canceled by user', 499, undefined, false);
      }

      if (isTimeout) {
        let status: OperationStatus = 'unknown';
        try {
          status = await resolveOperationStatus(operationId);
        } catch {
          status = 'unknown';
        }

        if (status === 'acked') {
          return;
        }

        if (status === 'not_found' && attempt < maxRetries) {
          await sleep(calcBackoffMs(attempt, retryBaseMs));
          continue;
        }

        if (status === 'not_found') {
          throw new ApiError('Request timed out and not completed', 408, undefined, true);
        }

        throw new ApiError('Request timed out and operation status unresolved', 504, undefined, false);
      }

      if (isNetworkError && attempt < maxRetries) {
        await sleep(calcBackoffMs(attempt, retryBaseMs));
        continue;
      }

      if (isNetworkError) {
        throw new ApiError('Network request failed', 0, undefined, true);
      }

      throw error;
    }
  }
}
```

```ts
// logging.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

type ConnectionState = 'disconnected' | 'connecting' | 'connected' | 'degraded';
type BackpressureState = 'normal' | 'high' | 'paused';

type TraceContext =
  | { requestId: string; traceId?: string }
  | { requestId?: string; traceId: string };

type FrontendLog = {
  // エッセンシャルログ
  timestamp: string;
  level: LogLevel;
  errorType: string;
  code: string;
  route: string;
  endpoint: string;
  method: string;
  durationMs: number;
  attempt: number;
  maxRetries: number;
  clientRequestId: string;
  operationId?: string;
  release: string;

  // フルログ - 必要なければ全部または一部を削除
  queueDepth?: number;
  queueWaitMs?: number;
  inFlight?: number;
  droppedReason?: string;
  backpressureState?: BackpressureState;
  highWatermark?: number;
  lowWatermark?: number;
  eventId?: string;
  eventSeq?: number;
  connectionState?: ConnectionState;
  sessionId?: string;
  lastAppliedEventSeq?: number;
  userAction?: string;
} & TraceContext;

type BuildFrontendLogInput = {
  level: LogLevel;
  errorType: string;
  code: string;
  route: string;
  endpoint: string;
  method: string;
  durationMs: number;
  attempt: number;
  maxRetries: number;
  clientRequestId: string;
  operationId?: string;
  release: string;
  queueDepth?: number;
  queueWaitMs?: number;
  inFlight?: number;
  droppedReason?: string;
  backpressureState?: BackpressureState;
  highWatermark?: number;
  lowWatermark?: number;
  eventId?: string;
  eventSeq?: number;
  connectionState?: ConnectionState;
  sessionId?: string;
  lastAppliedEventSeq?: number;
  userAction?: string;
} & TraceContext;

export function buildFrontendLog(input: BuildFrontendLogInput): FrontendLog {
  const essentialLog: FrontendLog = {
    timestamp: new Date().toISOString(),
    level: input.level,
    errorType: input.errorType,
    code: input.code,
    route: input.route,
    endpoint: input.endpoint,
    method: input.method,
    durationMs: input.durationMs,
    attempt: input.attempt,
    maxRetries: input.maxRetries,
    requestId: input.requestId,
    traceId: input.traceId,
    clientRequestId: input.clientRequestId,
    operationId: input.operationId,
    release: input.release,
  };

  const fullLog: FrontendLog = {
    ...essentialLog,
    queueDepth: input.queueDepth,
    queueWaitMs: input.queueWaitMs,
    inFlight: input.inFlight,
    droppedReason: input.droppedReason,
    backpressureState: input.backpressureState,
    highWatermark: input.highWatermark,
    lowWatermark: input.lowWatermark,
    eventId: input.eventId,
    eventSeq: input.eventSeq,
    connectionState: input.connectionState,
    sessionId: input.sessionId,
    lastAppliedEventSeq: input.lastAppliedEventSeq,
    userAction: input.userAction,
  };

  return fullLog;
}
```

```ts
// delivery-state.ts
type OutboxState = 'queued' | 'sending' | 'acked' | 'failed' | 'dead-letter';

type OutboxRecord = {
  operationId: string;
  idempotencyKey: string;
  payloadDigest: string;
  createdAt: string;
  attempt: number;
  state: OutboxState;
  ackedAt?: string;
  deadLetterAt?: string;
};

type OutboxRetentionConfig = {
  operationIdRetentionMs: number;
  deadLetterRetentionMs: number;
  retentionSweepIntervalMs: number;
};

type RetentionProfile = 'development' | 'staging' | 'production';

const retentionProfiles: Record<RetentionProfile, OutboxRetentionConfig> = {
  development: {
    operationIdRetentionMs: 10 * 60 * 1000,
    deadLetterRetentionMs: 24 * 60 * 60 * 1000,
    retentionSweepIntervalMs: 15 * 1000,
  },
  staging: {
    operationIdRetentionMs: 30 * 60 * 1000,
    deadLetterRetentionMs: 3 * 24 * 60 * 60 * 1000,
    retentionSweepIntervalMs: 30 * 1000,
  },
  production: {
    operationIdRetentionMs: 60 * 60 * 1000,
    deadLetterRetentionMs: 7 * 24 * 60 * 60 * 1000,
    retentionSweepIntervalMs: 60 * 1000,
  },
};

const defaultRetentionProfile: RetentionProfile = 'production';
const defaultRetentionConfig = retentionProfiles[defaultRetentionProfile];

export function getRetentionConfig(profile: RetentionProfile): OutboxRetentionConfig {
  return retentionProfiles[profile];
}

const inflightOperationIds = new Set<string>();
const appliedEventIds = new Set<string>();

// 同一operationIdの並行送信を防止し送信開始状態へ遷移する
export function beginSend(record: OutboxRecord): boolean {
  if (inflightOperationIds.has(record.operationId)) {
    return false;
  }

  inflightOperationIds.add(record.operationId);
  record.state = 'sending';
  return true;
}

// 送信完了後にinflight管理を解放する
export function completeSend(record: OutboxRecord, nowIso = new Date().toISOString()): void {
  inflightOperationIds.delete(record.operationId);
  record.state = 'acked';
  record.ackedAt = nowIso;
}

// 送信失敗時に失敗状態またはdead-letterへ遷移する
export function failSend(
  record: OutboxRecord,
  maxAttempts = 3,
  nowIso = new Date().toISOString()
): OutboxState {
  inflightOperationIds.delete(record.operationId);
  record.attempt += 1;
  if (record.attempt >= maxAttempts) {
    record.state = 'dead-letter';
    record.deadLetterAt = nowIso;
    return record.state;
  }

  record.state = 'failed';
  return record.state;
}

// 利用者キャンセル時にinflightを解放しqueuedへ戻す
export function cancelSend(record: OutboxRecord): void {
  inflightOperationIds.delete(record.operationId);
  record.state = 'queued';
}

function parseIsoMs(value: string): number {
  return Date.parse(value);
}

// operationId保持期間とdead-letter保持期間に基づき削除対象を判定する
export function shouldPurgeRecord(
  record: OutboxRecord,
  nowMs = Date.now(),
  config: OutboxRetentionConfig = defaultRetentionConfig
): boolean {
  if (record.state === 'acked') {
    const baseMs = parseIsoMs(record.ackedAt ?? record.createdAt);
    return nowMs - baseMs >= config.operationIdRetentionMs;
  }

  if (record.state === 'dead-letter') {
    const baseMs = parseIsoMs(record.deadLetterAt ?? record.createdAt);
    return nowMs - baseMs >= config.deadLetterRetentionMs;
  }

  return false;
}

// 掃除実行タイミングを設定値で判定する
export function shouldRunSweep(
  lastSweepMs: number,
  nowMs = Date.now(),
  config: OutboxRetentionConfig = defaultRetentionConfig
): boolean {
  return nowMs - lastSweepMs >= config.retentionSweepIntervalMs;
}

// 設定した保持期間を超えたレコードを除外する
export function sweepOutbox(
  records: OutboxRecord[],
  nowMs = Date.now(),
  config: OutboxRetentionConfig = defaultRetentionConfig
): OutboxRecord[] {
  return records.filter((record) => !shouldPurgeRecord(record, nowMs, config));
}

type EventApplyDecision = 'duplicate' | 'apply' | 'resync_required';

// 重複受信を抑止し適用種別を判定する
export function decideEventAction(
  eventId: string,
  eventSeq: number,
  lastAppliedEventSeq: number
): EventApplyDecision {
  if (appliedEventIds.has(eventId)) {
    return 'duplicate';
  }

  if (eventSeq <= lastAppliedEventSeq) {
    return 'duplicate';
  }

  const expectedSeq = lastAppliedEventSeq + 1;
  if (eventSeq === expectedSeq) {
    return 'apply';
  }

  return 'resync_required';
}

// 状態反映完了後に適用済みeventIdを記録する
export function markEventApplied(eventId: string): void {
  appliedEventIds.add(eventId);
}
```

```ts
// local-reconnect.ts
type LocalReconnectState = 'disconnected' | 'connecting' | 'connected' | 'degraded';

type ReconnectHandshakeRequest = {
  clientInstanceId: string;
  sessionId: string;
  lastAckedOperationId?: string;
  lastAppliedEventSeq: number;
};

type ReconnectHandshakeResponse = {
  peerMaxInFlight: number;
  peerQueueDepth: number;
  hasEventGap: boolean;
};

type ReconnectContext = {
  state: LocalReconnectState;
  clientInstanceId: string;
  sessionId: string;
  lastAckedOperationId?: string;
  lastAppliedEventSeq: number;
};

type ReconnectHooks = {
  onApplyPeerLimits: (peerMaxInFlight: number, peerQueueDepth: number) => void;
  onRequireFullResync: () => Promise<void>;
  onResendPending: (lastAckedOperationId?: string) => Promise<void>;
};

// 再接続ハンドシェイクを実行し未確定送信の再送対象を決定する
export async function reconnectLocalApp(
  context: ReconnectContext,
  connect: (request: ReconnectHandshakeRequest) => Promise<ReconnectHandshakeResponse>,
  hooks: ReconnectHooks
): Promise<LocalReconnectState> {
  context.state = 'connecting';
  const response = await connect({
    clientInstanceId: context.clientInstanceId,
    sessionId: context.sessionId,
    lastAckedOperationId: context.lastAckedOperationId,
    lastAppliedEventSeq: context.lastAppliedEventSeq,
  });

  hooks.onApplyPeerLimits(response.peerMaxInFlight, response.peerQueueDepth);

  if (response.hasEventGap) {
    await hooks.onRequireFullResync();
    context.state = 'degraded';
    return context.state;
  }

  await hooks.onResendPending(context.lastAckedOperationId);
  context.state = 'connected';
  return context.state;
}
```

```ts
// operation-id.ts
type OperationIdPoolConfig = {
  clientInstanceId: string;
  slotCount: number;
  operationIdRetentionMs: number;
  cancelReuseCooldownMs: number;
};

type OperationIdPoolProfile = 'development' | 'staging' | 'production';

const slotCountByProfile: Record<OperationIdPoolProfile, number> = {
  development: 4096,
  staging: 16384,
  production: 65536,
};

const cancelReuseCooldownMsByProfile: Record<OperationIdPoolProfile, number> = {
  development: 3000,
  staging: 5000,
  production: 10000,
};

const defaultOperationIdPoolProfile: OperationIdPoolProfile = 'production';

export function createOperationIdPoolConfig(
  clientInstanceId: string,
  operationIdRetentionMs: number,
  profile: OperationIdPoolProfile = defaultOperationIdPoolProfile
): OperationIdPoolConfig {
  return {
    clientInstanceId,
    slotCount: slotCountByProfile[profile],
    operationIdRetentionMs,
    cancelReuseCooldownMs: cancelReuseCooldownMsByProfile[profile],
  };
}

type OperationIdPoolState = {
  nextSlot: number;
  inUseSlots: Set<number>;
  retiredAtBySlot: Map<number, number>;
  cancelLockedUntilBySlot: Map<number, number>;
  pendingStateBySlot: Map<number, PendingSlotState>;
};

type PendingSlotState = 'queued' | 'sending' | 'failed';

type AcquireOperationIdResult =
  | { ok: true; operationId: string; slot: number }
  | { ok: false; code: 'operation_id_pool_exhausted'; retryAfterMs: number };

export function createOperationIdPoolState(): OperationIdPoolState {
  return {
    nextSlot: 0,
    inUseSlots: new Set<number>(),
    retiredAtBySlot: new Map<number, number>(),
    cancelLockedUntilBySlot: new Map<number, number>(),
    pendingStateBySlot: new Map<number, PendingSlotState>(),
  };
}

function encodeSlot(slot: number): string {
  return slot.toString(36).padStart(4, '0');
}

function decodeSlot(operationId: string, slotCount?: number): number | undefined {
  const parts = operationId.split(':');
  if (parts.length < 2) {
    return undefined;
  }

  const slot = Number.parseInt(parts[1], 36);
  if (!Number.isFinite(slot)) {
    return undefined;
  }

  if (slot < 0) {
    return undefined;
  }

  if (slotCount !== undefined && slot >= slotCount) {
    return undefined;
  }

  return slot;
}

function composeOperationId(clientInstanceId: string, slot: number): string {
  return `${clientInstanceId}:${encodeSlot(slot)}`;
}

function canReuseSlot(
  slot: number,
  nowMs: number,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig
): boolean {
  if (state.inUseSlots.has(slot)) {
    return false;
  }

  if (state.pendingStateBySlot.has(slot)) {
    return false;
  }

  const cancelLockedUntil = state.cancelLockedUntilBySlot.get(slot);
  if (cancelLockedUntil !== undefined) {
    if (nowMs < cancelLockedUntil) {
      return false;
    }

    state.cancelLockedUntilBySlot.delete(slot);
  }

  const retiredAt = state.retiredAtBySlot.get(slot);
  if (retiredAt === undefined) {
    return true;
  }

  return nowMs - retiredAt >= config.operationIdRetentionMs;
}

function estimateSlotRetryWaitMs(
  slot: number,
  nowMs: number,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig
): number {
  if (state.inUseSlots.has(slot) || state.pendingStateBySlot.has(slot)) {
    return config.operationIdRetentionMs;
  }

  const retiredAt = state.retiredAtBySlot.get(slot);
  const retiredWaitMs =
    retiredAt === undefined
      ? 0
      : Math.max(0, config.operationIdRetentionMs - (nowMs - retiredAt));

  const cancelLockedUntil = state.cancelLockedUntilBySlot.get(slot);
  const cancelWaitMs =
    cancelLockedUntil === undefined ? 0 : Math.max(0, cancelLockedUntil - nowMs);

  return Math.max(retiredWaitMs, cancelWaitMs);
}

function estimateRetryAfterMs(
  nowMs: number,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig
): number {
  let minWaitMs = config.operationIdRetentionMs;
  for (let slot = 0; slot < config.slotCount; slot += 1) {
    if (canReuseSlot(slot, nowMs, state, config)) {
      return 0;
    }

    const waitMs = estimateSlotRetryWaitMs(slot, nowMs, state, config);
    if (waitMs < minWaitMs) {
      minWaitMs = waitMs;
    }
  }

  return minWaitMs;
}

// operationIdを再利用可能スロットから取得する
export function acquireOperationId(
  state: OperationIdPoolState,
  config: OperationIdPoolConfig,
  nowMs = Date.now()
): AcquireOperationIdResult {
  for (let offset = 0; offset < config.slotCount; offset += 1) {
    const slot = (state.nextSlot + offset) % config.slotCount;
    if (!canReuseSlot(slot, nowMs, state, config)) {
      continue;
    }

    state.inUseSlots.add(slot);
    state.pendingStateBySlot.set(slot, 'queued');
    state.nextSlot = (slot + 1) % config.slotCount;
    return {
      ok: true,
      operationId: composeOperationId(config.clientInstanceId, slot),
      slot,
    };
  }

  return {
    ok: false,
    code: 'operation_id_pool_exhausted',
    retryAfterMs: estimateRetryAfterMs(nowMs, state, config),
  };
}

// outboxの未確定状態をスロットへ同期する
export function setOperationPendingState(
  operationId: string,
  pendingState: PendingSlotState,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig
): void {
  const slot = decodeSlot(operationId, config.slotCount);
  if (slot === undefined) {
    return;
  }

  state.pendingStateBySlot.set(slot, pendingState);
}

// outboxの未確定状態をスロットから解除する
export function clearOperationPendingState(
  operationId: string,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig
): void {
  const slot = decodeSlot(operationId, config.slotCount);
  if (slot === undefined) {
    return;
  }

  state.pendingStateBySlot.delete(slot);
}

// acked確定時に再利用待機状態へ遷移する
export function markOperationAcked(
  operationId: string,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig,
  nowMs = Date.now()
): void {
  const slot = decodeSlot(operationId, config.slotCount);
  if (slot === undefined) {
    return;
  }

  state.inUseSlots.delete(slot);
  state.pendingStateBySlot.delete(slot);
  state.retiredAtBySlot.set(slot, nowMs);
}

// dead-letter確定時に再利用待機状態へ遷移する
export function markOperationDeadLetter(
  operationId: string,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig,
  nowMs = Date.now()
): void {
  const slot = decodeSlot(operationId, config.slotCount);
  if (slot === undefined) {
    return;
  }

  state.inUseSlots.delete(slot);
  state.pendingStateBySlot.delete(slot);
  state.retiredAtBySlot.set(slot, nowMs);
}

// 利用者キャンセル時に短時間クールダウンでスロット再利用を遅延する
export function releaseOperationInUse(
  operationId: string,
  state: OperationIdPoolState,
  config: OperationIdPoolConfig,
  nowMs = Date.now()
): void {
  const slot = decodeSlot(operationId, config.slotCount);
  if (slot === undefined) {
    return;
  }

  state.inUseSlots.delete(slot);
  state.pendingStateBySlot.delete(slot);
  state.cancelLockedUntilBySlot.set(slot, nowMs + config.cancelReuseCooldownMs);
}

// 保持期間切れとクールダウン解除済みスロットの記録を掃除する
export function sweepRetiredOperationSlots(
  state: OperationIdPoolState,
  config: OperationIdPoolConfig,
  nowMs = Date.now()
): void {
  for (const [slot, retiredAt] of state.retiredAtBySlot.entries()) {
    if (nowMs - retiredAt < config.operationIdRetentionMs) {
      continue;
    }

    state.retiredAtBySlot.delete(slot);
  }

  for (const [slot, cancelLockedUntil] of state.cancelLockedUntilBySlot.entries()) {
    if (nowMs < cancelLockedUntil) {
      continue;
    }

    state.cancelLockedUntilBySlot.delete(slot);
  }
}
```

```ts
// request-queue.ts
type QueuePriority = 'interactive' | 'background';

type QueueTask<T> = {
  requestKey: string;
  requestKind: 'read' | 'mutation';
  priority: QueuePriority;
  enqueuedAt: number;
  run: () => Promise<T>;
};

type QueueDropReason = 'queue_full' | 'queue_wait_timeout';

type QueueDropEvent = {
  requestKey: string;
  reason: QueueDropReason;
  queueDepth: number;
  queueWaitMs: number;
  message: string;
};

class QueueDropError extends Error {
  constructor(
    public reason: QueueDropReason,
    public queueDepth: number,
    public queueWaitMs: number,
    message: string
  ) {
    super(message);
    this.name = 'QueueDropError';
  }
}

type InternalQueueTask = {
  requestKey: string;
  requestKind: 'read' | 'mutation';
  priority: QueuePriority;
  enqueuedAt: number;
  run: () => Promise<unknown>;
  resolve: (value: unknown) => void;
  reject: (error: unknown) => void;
};

type QueueHooks = {
  onDrop?: (event: QueueDropEvent) => void;
};

type QueueConfig = {
  maxInFlight: number;
  maxQueue: number;
  maxQueueWaitMs: number;
};

type QueueProfile = 'development' | 'staging' | 'production';

const queueConfigByProfile: Record<QueueProfile, QueueConfig> = {
  development: {
    maxInFlight: 2,
    maxQueue: 20,
    maxQueueWaitMs: 5000,
  },
  staging: {
    maxInFlight: 4,
    maxQueue: 60,
    maxQueueWaitMs: 8000,
  },
  production: {
    maxInFlight: 6,
    maxQueue: 100,
    maxQueueWaitMs: 10000,
  },
};

type OverloadSignal =
  | { kind: 'http'; status: 429 | 503 }
  | { kind: 'network'; code: 'connection_refused' | 'connection_reset' };

const defaultQueueProfile: QueueProfile = 'production';
let queueConfig: QueueConfig = queueConfigByProfile[defaultQueueProfile];
let inFlight = 0;
const interactiveQueue: InternalQueueTask[] = [];
const backgroundQueue: InternalQueueTask[] = [];
const pendingByRequestKey = new Map<string, Promise<unknown>>();
const queueHooks: QueueHooks = {};
let sendPauseUntilMs = 0;
let dispatchRampFactor = 1;

// λ_peak μ_effective Tから推奨maxQueueを算出する
export function computeRecommendedMaxQueue(
  lambdaPeak: number,
  muEffective: number,
  waitMs: number
): number {
  const waitSeconds = waitMs / 1000;
  return Math.max(1, Math.ceil((lambdaPeak - muEffective) * waitSeconds));
}

// キュープロファイルを選択する
export function setQueueProfile(profile: QueueProfile): void {
  queueConfig = queueConfigByProfile[profile];
}

// キュー設定を直接上書きする
export function setQueueConfig(config: QueueConfig): void {
  if (config.maxInFlight <= 0 || config.maxQueue <= 0 || config.maxQueueWaitMs <= 0) {
    throw new Error('Queue config must be greater than zero');
  }

  queueConfig = config;
}

function emitDrop(event: QueueDropEvent): void {
  queueHooks.onDrop?.(event);
}

function dequeue(): InternalQueueTask | undefined {
  return interactiveQueue.shift() ?? backgroundQueue.shift();
}

// droppedReason記録用のフックを設定する
export function setQueueHooks(hooks: QueueHooks): void {
  queueHooks.onDrop = hooks.onDrop;
}

// 過負荷シグナルを受けて送信停止窓と復帰係数を更新する
export function recordOverloadSignal(
  signal: OverloadSignal,
  nowMs = Date.now(),
  pauseWindowMs = 3000
): void {
  const windowMs = signal.kind === 'network' ? Math.max(pauseWindowMs, 5000) : pauseWindowMs;
  sendPauseUntilMs = Math.max(sendPauseUntilMs, nowMs + windowMs);
  dispatchRampFactor = 0.5;
}

// 停止窓解除後に段階的に通常送信へ戻す
export function recordOverloadRecovery(nowMs = Date.now(), step = 0.25): void {
  if (nowMs < sendPauseUntilMs) {
    return;
  }

  dispatchRampFactor = Math.min(1, dispatchRampFactor + step);
}

function effectiveMaxInFlight(): number {
  return Math.max(1, Math.floor(queueConfig.maxInFlight * dispatchRampFactor));
}

function canStartDispatch(nowMs = Date.now()): boolean {
  if (nowMs < sendPauseUntilMs) {
    return false;
  }

  return inFlight < effectiveMaxInFlight();
}

// 優先度キューへタスクを投入し読取要求のみ重複requestKeyを統合する
export function enqueueTask<T>(task: QueueTask<T>): Promise<T> {
  if (task.requestKind === 'read') {
    const existing = pendingByRequestKey.get(task.requestKey);
    if (existing) {
      return existing as Promise<T>;
    }
  }

  const queueDepth = interactiveQueue.length + backgroundQueue.length;
  if (queueDepth >= queueConfig.maxQueue) {
    const message = 'Queue is full, please retry';
    const drop: QueueDropEvent = {
      requestKey: task.requestKey,
      reason: 'queue_full',
      queueDepth,
      queueWaitMs: 0,
      message,
    };
    emitDrop(drop);
    return Promise.reject(
      new QueueDropError(drop.reason, drop.queueDepth, drop.queueWaitMs, drop.message)
    );
  }

  let resolveTask: (value: T) => void = () => {};
  let rejectTask: (error: unknown) => void = () => {};
  const result = new Promise<T>((resolve, reject) => {
    resolveTask = resolve;
    rejectTask = reject;
  });
  if (task.requestKind === 'read') {
    pendingByRequestKey.set(task.requestKey, result as Promise<unknown>);
  }

  const queuedTask: InternalQueueTask = {
    requestKey: task.requestKey,
    requestKind: task.requestKind,
    priority: task.priority,
    enqueuedAt: task.enqueuedAt,
    run: task.run as () => Promise<unknown>,
    resolve: (value: unknown) => resolveTask(value as T),
    reject: rejectTask,
  };

  if (task.priority === 'interactive') {
    interactiveQueue.push(queuedTask);
  } else {
    backgroundQueue.push(queuedTask);
  }

  void pumpQueue();
  return result;
}

// 同時実行数と待機時間上限を守ってキューを処理する
export async function pumpQueue(): Promise<void> {
  recordOverloadRecovery();
  while (canStartDispatch()) {
    const task = dequeue();
    if (!task) {
      return;
    }

    const queueWaitMs = Date.now() - task.enqueuedAt;
    if (queueWaitMs > queueConfig.maxQueueWaitMs) {
      const queueDepth = interactiveQueue.length + backgroundQueue.length;
      const message = 'Queue wait exceeded, please retry';
      const drop: QueueDropEvent = {
        requestKey: task.requestKey,
        reason: 'queue_wait_timeout',
        queueDepth,
        queueWaitMs,
        message,
      };
      emitDrop(drop);
      task.reject(
        new QueueDropError(drop.reason, drop.queueDepth, drop.queueWaitMs, drop.message)
      );
      if (task.requestKind === 'read') {
        pendingByRequestKey.delete(task.requestKey);
      }
      continue;
    }

    inFlight += 1;
    void task.run()
      .then((value) => {
        task.resolve(value);
      })
      .catch((error) => {
        task.reject(error);
      })
      .finally(() => {
        if (task.requestKind === 'read') {
          pendingByRequestKey.delete(task.requestKey);
        }
        inFlight -= 1;
        void pumpQueue();
      });
  }
}
```

```ts
// websocket-backpressure.ts
type BackpressureController = {
  highWatermark: number;
  lowWatermark: number;
  paused: boolean;
};

type SendPriority = 'high' | 'low';
type SendResult = 'sent' | 'deferred' | 'reconnect_required' | 'not_open';

export function createBackpressureController(
  highWatermark = 1_000_000,
  lowWatermark = 500_000
): BackpressureController {
  return {
    highWatermark,
    lowWatermark,
    paused: false,
  };
}

// 送信可否をバッファ量で判定する
export function canSend(
  ws: WebSocket,
  controller: BackpressureController,
  priority: SendPriority = 'high'
): boolean {
  if (ws.readyState !== WebSocket.OPEN) {
    return false;
  }

  if (ws.bufferedAmount >= controller.highWatermark) {
    controller.paused = true;
  }

  if (controller.paused && ws.bufferedAmount <= controller.lowWatermark) {
    controller.paused = false;
  }

  if (controller.paused && priority === 'low') {
    return false;
  }

  return true;
}

// バックプレッシャーを考慮してメッセージ送信し再接続判定も返す
export function sendWithBackpressure(
  ws: WebSocket,
  payload: string,
  controller: BackpressureController,
  priority: SendPriority = 'high'
): SendResult {
  if (ws.readyState !== WebSocket.OPEN) {
    return 'not_open';
  }

  if (!canSend(ws, controller, priority)) {
    return 'deferred';
  }

  try {
    ws.send(payload);
  } catch {
    return 'reconnect_required';
  }

  if (ws.readyState === WebSocket.CLOSING || ws.readyState === WebSocket.CLOSED) {
    return 'reconnect_required';
  }

  return 'sent';
}
```

```tsx
// error-boundary.tsx
import * as React from 'react';
import { buildFrontendLog } from './logging';

type State = { hasError: boolean };

// 描画例外を隔離しフォールバック表示へ切り替える
export class ErrorBoundary extends React.Component<React.PropsWithChildren, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  componentDidCatch(error: unknown, info: React.ErrorInfo): void {
    const frontendLog = buildFrontendLog({
      level: 'error',
      errorType: 'render_exception',
      code: 'ui_render_error',
      route: location.pathname,
      endpoint: '/frontend-errors',
      method: 'POST',
      durationMs: 0,
      attempt: 0,
      maxRetries: 0,
      traceId: crypto.randomUUID(),
      clientRequestId: crypto.randomUUID(),
      release: 'web-1.0.0',
      userAction: 'render',
    });

    navigator.sendBeacon('/frontend-errors', JSON.stringify({
      ...frontendLog,
      error: String(error),
      stack: info.componentStack,
    }));
  }

  render(): React.ReactNode {
    if (this.state.hasError) {
      return <p>問題が発生しました。しばらくしてから再試行してください。</p>;
    }

    return this.props.children;
  }
}
```

```ts
// overload-wiring.ts
import { requestJson, requestMutation } from './api-client';
import { recordOverloadSignal } from './request-queue';

// HTTP 429/503 を検知したらキュー制御へ伝播するラッパー例
export async function fetchWithOverload<T>(input: RequestInfo, init?: RequestInit): Promise<T> {
  return requestJson<T>(input, {
    ...init,
    onOverloadSignal: signal => recordOverloadSignal(signal),
  });
}

// WebSocket送信結果に応じて再接続を開始する例
import { createBackpressureController, sendWithBackpressure } from './websocket-backpressure';

export async function sendWs(ws: WebSocket, payload: string): Promise<void> {
  const controller = createBackpressureController();
  const result = sendWithBackpressure(ws, payload, controller, 'low');
  if (result === 'reconnect_required') {
    ws.close();
    // 再接続開始（実装は環境依存）
  }
}
```

## 出典・参考情報
### 情報・規格

- [RFC 9457: Problem Details for HTTP APIs](https://www.ietf.org/rfc/rfc9457.html) - `application/problem+json` の標準仕様。
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html) - HTTPステータスの意味と動作定義。
- [RFC 6585: Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585.html) - 428、429 など追加ステータス定義。
- [Window: fetch() method](https://developer.mozilla.org/en-US/docs/Web/API/Window/fetch) - `fetch()` が HTTP エラーで reject しない仕様。
- [Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) - `response.ok` 判定を含む実装パターン。
- [Response: ok property](https://developer.mozilla.org/en-US/docs/Web/API/Response/ok) - 200 台判定の仕様。
- [HTTP response status codes | MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) - HTTPステータスコードの実装参照。
- [429 Too Many Requests | MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429) - レート制限時のレスポンス解釈。
- [AbortSignal: timeout() static method](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static) - タイムアウト中断の標準 API。
- [Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html) - Android 実装でのタイムアウトとキャンセル制御。
- [URLSessionConfiguration.timeoutIntervalForRequest](https://developer.apple.com/documentation/foundation/urlsessionconfiguration/timeoutintervalforrequest) - iOS のリクエストタイムアウト設定。
- [URLSessionTask.cancel()](https://developer.apple.com/documentation/foundation/urlsessiontask/1411590-cancel) - iOS のリクエスト中断 API。
- [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API) - WebSocket API の前提と制約。
- [WebSocket: bufferedAmount property](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/bufferedAmount) - 送信バッファ量の監視方法。
- [WebSocket: send() method](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket/send) - 送信バッファ溢れ時の動作。
- [Streams Standard](https://streams.spec.whatwg.org/) - backpressure の基礎仕様。
- [Window: online event](https://developer.mozilla.org/en-US/docs/Web/API/Window/online_event) - online イベントの意味と制限。
- [Background Synchronization API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API) - 背景再送の仕様と制約。
- [URLSession](https://developer.apple.com/documentation/foundation/urlsession) - iOS の標準 HTTP クライアント。
- [NWPathMonitor](https://developer.apple.com/documentation/network/nwpathmonitor) - iOS のネットワーク状態監視。
- [BGTaskScheduler](https://developer.apple.com/documentation/backgroundtasks/bgtaskscheduler) - iOS の背景タスク実行。
- [ConnectivityManager](https://developer.android.com/reference/android/net/ConnectivityManager) - Android のネットワーク状態監視。
- [WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager) - Android の背景再送実行。
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) - 永続キュー実装向けブラウザ保存 API。
- [Storage quotas and eviction criteria](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria) - 保存容量と削除条件。
- [Navigator: sendBeacon() method](https://developer.mozilla.org/docs/Web/API/Navigator/sendBeacon) - 離脱時診断送信の推奨手法。
- [Component | React](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) - Error Boundary の公式仕様。
- [Understanding SC 3.3.1: Error Identification](https://www.w3.org/WAI/WCAG22/Understanding/error-identification) - エラー識別の要件。
- [Understanding SC 3.3.3: Error Suggestion](https://www.w3.org/WAI/WCAG22/Understanding/error-suggestion.html) - 修正提案提示の要件。
- [Technique ARIA19](https://www.w3.org/WAI/WCAG22/Techniques/aria/ARIA19) - `role="alert"` と live region の実装技法。
- [OWASP Error Handling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html) - エラー情報露出を抑える実装原則。
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) - セキュアなログ記録要件。
