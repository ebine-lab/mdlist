---
title: "Database Design Guidelines"
description: "データベース設計・セキュリティ・運用の共通規約 / Database design, security, and operations standards"
version: "1.0.0"
status: "Stable"
last_updated: "2026-03-20T02:27+09:00"
lang: "ja"
---

# Database Design Guidelines

**説明** - データベースの設計、セキュリティ、運用に関する共通規約。RDB（SQL）を基本とし、NoSQL（MongoDB）についても言及する。


---

## 対応データベース

| 分類 | DB | 用途 |
|:---|:---|:---|
| RDB | PostgreSQL | 本番用メインDB |
| RDB | MySQL / MariaDB | 軽量案件・既存システム |
| RDB | SQLite | 開発・テスト・組み込み・モバイル |
| NoSQL | MongoDB | ドキュメント指向・柔軟スキーマ |

---

## 1. DB選定基準

プロジェクト要件に応じて以下の観点で選定する。

- RDB を選択する場合：トランザクションの厳密性が要求される、テーブル間の参照整合性が複雑、既存のRDB資産がある
- MongoDB を選択する場合：スキーマが頻繁に変化する、階層的・非定型のデータ構造が主体、読み取り負荷が高くスケールアウトが前提
- PostgreSQL vs MySQL / MariaDB：PostgreSQL は JSONB・GiST / GIN インデックス・RLS 等の拡張機能が必要な場合に選択する。MySQL / MariaDB は軽量案件やホスティング環境との互換性を優先する場合に選択する
- SQLite の適用範囲：開発・テスト環境、モバイルアプリのローカルDB、組み込みシステムに限定する。同時書き込みが必要な本番環境には使用しない
- ORM / クエリビルダーの選定は§17を参照

---

## 2. スキーマ設計原則

### 2-1. RDB（SQL）

正規化は第三正規形（3NF）を基準とする。3NF を超える正規化はパフォーマンス要件と照合して判断する。意図的な非正規化は読み取り負荷の高いシステムやレポーティング用途に限定し、非正規化の根拠をスキーマ文書に記録する。

エンティティの識別とリレーションシップのモデリング（1:1 / 1:N / N:N）は、ER図を作成して視覚化した上で設計する。N:N リレーションシップは中間テーブルで実装する。

### 2-2. NoSQL（MongoDB）

ドキュメント設計では埋め込み（embedding）と参照（referencing）の選択基準を明確にする。頻繁に同時取得するデータは埋め込み、独立して更新・参照されるデータは参照で設計する。埋め込みドキュメントの上限は16MBのBSONサイズ制限を考慮する。

スキーマバリデーション（`$jsonSchema`）を適用し、ドキュメント構造の一貫性を維持する。

---

## 3. 命名規則

### 3-1. 基本原則

- ケース規約：全DB共通で **snake_case** を使用する。camelCase / PascalCase / kebab-case をDB名・テーブル名・カラム名に使用しない
- 言語：英語のみ使用する。ローマ字表記の日本語は禁止
- 略語：一般に認知された略語のみ許可（`id`, `url`, `qty`, `amt` 等）。プロジェクト固有の略語は略語一覧を定義した上で使用する
- 予約語の回避：SQL / MongoDB の予約語をオブジェクト名に使用しない（`user`, `order`, `group`, `select`, `index` 等）。衝突する場合はサフィックスで回避する（`user_account`, `purchase_order`）

### 3-2. RDB — テーブル名

- **複数形**を使用する（`users`, `orders`, `products`）
- 中間テーブルは関連する2テーブル名をアルファベット順に結合する（`products_tags`）
- テーブル名にプレフィックスを付けない（`tbl_users` は禁止）

### 3-3. RDB — カラム名

- 主キー：`id`（サロゲートキー）
- 外部キー：参照先テーブルの単数形 + `_id`（`user_id`, `product_id`）
- 真偽値：`is_` / `has_` / `can_` プレフィックス（`is_active`, `has_verified`, `can_edit`）
- 日時：`_at` サフィックス（`created_at`, `updated_at`, `deleted_at`, `published_at`）
- 日付のみ：`_on` サフィックス（`born_on`, `expired_on`）
- カウント：`_count` サフィックス（`comment_count`, `retry_count`）
- 金額：`_amount` / `_price` サフィックス + DECIMAL型との組み合わせ

### 3-4. RDB — 共通カラム（原則）

| カラム名 | 型 | 説明 |
|:---|:---|:---|
| `id` | PostgreSQL: BIGSERIAL / UUID、MySQL / MariaDB: BIGINT UNSIGNED AUTO_INCREMENT / UUID、SQLite: INTEGER PRIMARY KEY / TEXT(UUID) | 主キー |
| `created_at` | PostgreSQL: TIMESTAMP WITH TIME ZONE、MySQL / MariaDB: TIMESTAMP(6) または DATETIME(6)（UTC）、SQLite: TEXT（ISO 8601 UTC） | レコード作成日時 |
| `updated_at` | PostgreSQL: TIMESTAMP WITH TIME ZONE、MySQL / MariaDB: TIMESTAMP(6) または DATETIME(6)（UTC）、SQLite: TEXT（ISO 8601 UTC） | レコード更新日時 |

論理削除を採用するテーブルには `deleted_at`（PostgreSQL: TIMESTAMP WITH TIME ZONE、MySQL / MariaDB: TIMESTAMP(6) または DATETIME(6)、SQLite: TEXT（ISO 8601 UTC）、NULL許容）を追加する。
例外として、N:Nの中間テーブルで複合主キーを採用する場合は `id` を省略可とする。

### 3-5. RDB — 制約名・インデックス名

| 種別 | 命名パターン | 例 |
|:---|:---|:---|
| 主キー | `pk_{table}` | `pk_users` |
| 外部キー | `fk_{table}_{column}` | `fk_orders_user_id` |
| ユニーク制約 | `uq_{table}_{column}` | `uq_users_email` |
| チェック制約 | `ck_{table}_{column}` | `ck_products_price` |
| インデックス | `idx_{table}_{column}` | `idx_users_email` |
| 複合インデックス | `idx_{table}_{col1}_{col2}` | `idx_orders_user_id_created_at` |

### 3-6. MongoDB — コレクション名・フィールド名

- コレクション名：**複数形** + snake_case（`users`, `purchase_orders`）
- フィールド名：snake_case（`user_id`, `created_at`）。MongoDBの慣例であるcamelCaseは本規約では採用しない（RDBとの一貫性を優先）
- `_id` フィールドはMongoDBの既定動作に従う（ObjectId自動生成）。カスタムIDを使用する場合は§6の設計方針に従う
- 埋め込みドキュメント名：単数形（1件の場合）/ 複数形（配列の場合）

---

## 4. データ型選定

### 4-1. RDB

- 数値型は対象DBの型体系に合わせて最小の型を選択する（PostgreSQL: SMALLINT / INTEGER / BIGINT、MySQL / MariaDB: TINYINT / SMALLINT / INT / BIGINT）
- 金銭データには DECIMAL / NUMERIC を必須とする。FLOAT / REAL は丸め誤差があるため金銭データに使用しない
- VARCHAR の長さは要件に基づいて適正値を設定する。`VARCHAR(255)` の機械的使用を禁止する
- タイムスタンプはUTC保存を原則とし、型はDBごとの対応型を使用する（PostgreSQL: TIMESTAMP WITH TIME ZONE、MySQL / MariaDB: TIMESTAMP(6) または DATETIME(6)、SQLite: TEXT / INTEGER）
- BOOLEAN は真偽値に使用する。TINYINT(1) による代用は MySQL 互換が必要な場合に限定する
- JSON / JSONB は構造化データの格納に使用する。PostgreSQL では JSONB を優先する（インデックス作成・検索性能）
- DB間の型差異に注意する。PostgreSQL の `SERIAL` / `BIGSERIAL` は MySQL の `AUTO_INCREMENT` に対応する。DB間のデータ型マッピングの詳細は§18-4を参照

### 4-2. MongoDB

- ObjectId をドキュメント識別子の既定とする
- 金銭データには Decimal128 を使用する
- 日時データには Date 型を使用する（文字列での日時保存を禁止）

---

## 5. 制約設計

### 5-1. RDB

- NOT NULL を原則とする。NULL 許容は要件上の根拠がある場合のみ認め、根拠をスキーマ文書に記録する
- UNIQUE 制約はビジネス上一意であるべきカラムに設定する（メールアドレス、ユーザー名等）
- CHECK 制約で値域制限・フォーマット検証を実装する（例：`ck_products_price CHECK (price >= 0)`）
- DEFAULT 値は業務ロジックに基づいて設定する。`created_at` には `CURRENT_TIMESTAMP` を設定する
- FOREIGN KEY 制約を必ず定義し、参照整合性をDB層で強制する。カスケードポリシー（ON DELETE / ON UPDATE）は§6で規定する

### 5-2. MongoDB

- Schema Validation（`$jsonSchema`）で必須フィールドとデータ型を定義する
- `required` フィールドの設定方針はRDBの NOT NULL 方針に準ずる

---

## 6. 主キー・外部キー設計

### 6-1. RDB

サロゲートキー（auto-increment の整数 または UUID）を主キーとして使用する。自然キー（メールアドレス、SKU等）は主キーに使用しない。理由は、自然キーはビジネスロジックの変更で値が変わる可能性があり、外部キーで参照している全テーブルへの伝播が発生するため。

UUIDは分散システムやマルチテナント環境で採用する。単一サーバー構成ではauto-increment整数を優先する（結合性能の優位性）。UUIDを採用する場合はUUIDv7（RFC 9562）を推奨する。UUIDv7は先頭にタイムスタンプを含むため時系列順にソート可能であり、B-Treeインデックスのフラグメンテーションを抑制する。UUIDv4（完全ランダム）はインデックスの局所性が低く、大量INSERTのパフォーマンスに影響するため、新規設計では避ける。

複合主キーは中間テーブル以外では原則使用しない。

外部キーのカスケードポリシー：

| ポリシー | 使用場面 |
|:---|:---|
| `ON DELETE RESTRICT` | 既定。参照先が存在する限り削除を禁止する |
| `ON DELETE CASCADE` | 親子関係が強い場合（親削除時に子も不要になる場合）のみ使用する |
| `ON DELETE SET NULL` | 参照先が削除されても行自体は残す必要がある場合に使用する |
| `ON UPDATE CASCADE` | 主キーの値が変更される場合（サロゲートキー使用時は原則不要） |

### 6-2. MongoDB

`_id` フィールドはObjectIdの自動生成を既定とする。外部コレクションへの参照は手動参照（参照先の `_id` を値として保持）を使用する。DBRefは使用しない（ドライバー依存が高いため）。

---

## 7. インデックス設計方針

### 7-1. 共通原則

- WHERE / JOIN / ORDER BY で頻出するカラムにインデックスを作成する
- カーディナリティ（一意値の多さ）が高いカラムを優先する
- 複合インデックスのカラム順序は選択性の高いカラムを先頭に配置する
- カバリングインデックス（クエリに必要な全カラムをインデックスに含める）を活用する
- 過剰なインデックスは書き込み性能を劣化させるため、不要なインデックスは作成しない
- EXPLAIN / ANALYZE による実行計画の確認を義務化する

### 7-2. PostgreSQL固有

- JSONB の検索には GIN インデックスを使用する
- 地理データには GiST インデックスを使用する
- 部分インデックス（`WHERE` 条件付き）を活用して不要な行をインデックスから除外する

### 7-3. MySQL / MariaDB固有

- 全文検索には FULLTEXT インデックスを使用する
- InnoDBのクラスタードインデックス特性（主キーの物理的順序がデータ配置に影響する）を考慮する

### 7-4. MongoDB固有

- 単一フィールド / 複合 / マルチキー / テキスト / 地理空間インデックスを要件に応じて使い分ける
- `explain()` で実行計画を確認する

出典・参考情報: PostgreSQL Documentation、MySQL Reference Manual、MongoDB Documentation

---

## 8. セキュリティ

> 汎用的なセキュリティ要件は [secure-code-requirements.md](../Documents/ClaudeCode/AI-instructions/secure-code-requirements.md) を参照。本セクションはDB固有のセキュリティ対策を規定する。

### 8-1. SQLインジェクション対策（RDB）

- プリペアドステートメント / パラメータバインドを必須とする（`secure-code-requirements.md` §1 と連動）
- ORM使用時も生SQL実行にはパラメータバインドを必須とする
- ストアドプロシージャ内の動的SQL構築は `EXECUTE ... USING` を基本とする。識別子を動的に扱う場合は `format('%I', ...)` または `quote_ident()` を使用し、値の文字列連結を禁止する（PostgreSQL）
- 入力値のホワイトリスト検証と組み合わせる

### 8-2. NoSQLインジェクション対策（MongoDB）

- オペレーターインジェクション防止（`$gt`, `$ne`, `$where` 等の意図しない注入を防ぐ）
- ユーザー入力はスキーマで定義された期待型（文字列 / 数値 / 真偽値 / ObjectId 等）を検証してからクエリに使用する
- `$where` / `mapReduce` の使用を制限する

### 8-3. クロスサイトスクリプティング（XSS）対策

- DBから取得したデータの出力時エスケープを必須とする（`secure-code-requirements.md` §5 と連動）
- 保存時のサニタイズではなく出力時エスケープを基本とする
- JSON / JSONB カラムに格納されたユーザー入力も出力時エスケープの対象とする

### 8-4. 通信暗号化（SSL/TLS）

全DB接続にSSL/TLSを必須化する。

| DB | 設定 |
|:---|:---|
| PostgreSQL | `sslmode=verify-full` |
| MySQL | `--ssl-mode=VERIFY_IDENTITY`（最低でも `VERIFY_CA`） |
| MariaDB | `--ssl-ca=/path/to/ca.pem` を指定し、`--ssl-verify-server-cert` を有効化する（バージョン差分は公式ドキュメントで確認） |
| MongoDB | `tls=true` / `tlsCAFile` |
| SQLite | ローカル接続のため通信暗号化は不要。ファイルレベル暗号化が必要な場合はSQLCipherを検討する |

### 8-5. パスワードのハッシュ化（必須仕様）

パスワードは復号の必要がないため、暗号化ではなくハッシュ化で保護する。平文保存は厳禁（`secure-code-requirements.md` §2 と連動）。

**使用禁止アルゴリズム：** SHA-1 / MD5。コリジョン攻撃への脆弱性が実証されており、計算速度が高速なためブルートフォース攻撃やレインボーテーブル攻撃に対する耐性が不十分。

**必須アルゴリズム（優先順）：**

| アルゴリズム | 最低ラウンド数 / コスト | 用途 |
|:---|:---|:---|
| Argon2id | メモリー: 64MB以上、反復: 3以上 | 新規実装の第一選択。メモリーハード関数によりGPU攻撃にも耐性あり |
| bcrypt | コストファクター: 12以上 | Argon2id が使用できない環境での代替 |
| scrypt | N: 2^15以上、r: 8、p: 1 | bcrypt の代替（メモリーハード関数） |

**必須要件：**
- ソルトは暗号論的擬似乱数生成器（CSPRNG）で生成し、ユーザーごとに一意とする。ソルト長は16バイト以上
- ハッシュ結果にソルトを含めて保存する（bcrypt / Argon2id は自動的に含む）
- パスワード検証は定時間比較（timing-safe comparison）で実行し、タイミング攻撃を防止する

出典・参考情報: OWASP Password Storage Cheat Sheet

### 8-6. 個人情報・機密データの保存時暗号化（Encryption at Rest）

パスワード以外の個人情報や機密データは、復号が必要なため暗号化（可逆的な変換）で保護する。

**データ分類と暗号化要件：**

| 分類 | 対象データの例 | 暗号化要件 |
|:---|:---|:---|
| **認証情報** | パスワード | §8-5のハッシュ化を適用（暗号化ではない） |
| **機密度：高** | クレジットカード番号、銀行口座情報、マイナンバー、パスポート番号 | アプリケーションレベル暗号化必須（AES-256-GCM）。DBレベル暗号化との二重保護を推奨 |
| **機密度：中** | メールアドレス、電話番号、住所、生年月日、氏名（実名） | アプリケーションレベル暗号化を推奨。最低限DBレベル暗号化（TDE / pgcrypto）を適用 |
| **機密度：低** | 表示名（ニックネーム）、ユーザー設定、公開プロフィール | 暗号化任意。DBレベル暗号化（TDE）で十分 |

**暗号化方式：**

| 方式 | 実装層 | 適用DB | 特徴 |
|:---|:---|:---|:---|
| アプリケーションレベル暗号化 | アプリケーションコード | 全DB共通 | 暗号化・復号はアプリ側で実行。DBには暗号文のみ保存。DB管理者にも平文が見えない |
| pgcrypto | DB拡張 | PostgreSQL | DB内で暗号化・復号を実行。SQL上で透過的に扱える |
| TDE（透過的データ暗号化） | DBエンジン | MySQL / MariaDB（InnoDB）、MongoDB（WiredTiger。Atlas または MongoDB Enterprise のみ） | ストレージレベルの暗号化。アプリ側の変更不要だが、DB接続があれば平文が見える |
| SQLCipher | DBレベル | SQLite | SQLiteファイル全体を暗号化 |

**暗号化鍵の管理（必須）：**
- 暗号化鍵はDBと同一サーバーに保存しない（鍵とデータの物理的分離）
- 鍵管理にはシークレット管理機構（AWS KMS / Google Cloud KMS / HashiCorp Vault / 環境変数）を使用する
- 鍵のローテーション手順を策定し、定期的に実行する
- アプリケーションコード内への鍵のハードコードは厳禁（`secure-code-requirements.md` §3 と連動）

### 8-7. アクセス制御

最小権限の原則に基づきロールを設計する（`secure-code-requirements.md` §9 と連動）。

- PostgreSQL：`GRANT` / `REVOKE` でテーブル・カラム単位の権限を設定する
- MySQL / MariaDB：ユーザー権限体系に基づき設定する
- MongoDB：ビルトインロール（`read`, `readWrite`, `dbAdmin` 等）を使用し、必要に応じてカスタムロールを定義する
- アプリケーション接続ユーザーにはDDL権限を付与しない。DMLも必要なテーブルのみに限定する
- マルチテナント設計では行レベル隔離を必須とする。PostgreSQL は RLS（`CREATE POLICY`）を使用し、MySQL / MariaDB / MongoDB ではDB権限設計とアプリケーション層のテナントフィルタリングで同等の隔離を実装する

### 8-8. 接続情報管理

- 接続文字列のハードコード禁止（環境変数 / シークレット管理機構の使用必須）
- `.env` ファイルは `.gitignore` に登録必須
- 本番環境の接続情報は定期的にローテーションする

### 8-9. その他のセキュリティリスク対応

- ブルートフォース攻撃対策（レートリミット、アカウントロック）
- DB管理ポートの外部公開禁止（ファイアウォール / ネットワーク分離）
- DBログへの機密情報出力禁止（ALTER USER のパスワード平文問題等）
- 不要なDB機能の無効化（PostgreSQL の `dblink`、MySQL の `LOAD DATA LOCAL INFILE` 等）
- 定期的な脆弱性スキャンとパッチ適用

出典・参考情報: OWASP Database Security Cheat Sheet、OWASP Transport Layer Security Cheat Sheet

---

## 9. 論理削除（Soft Delete）

アプリケーションからのレコード削除は論理削除を必須とし、物理削除（`DELETE FROM`）を禁止する。物理削除はデータの不可逆的な喪失、参照整合性の破壊、監査証跡の消失を招くため、通常の運用で使用しない。

法令（GDPR第17条等）により物理削除が義務付けられる場合に限り、「論理削除 → 保持期間経過後に定期パージジョブで物理削除」の二段階運用で対応する。

**実装パターン：** `deleted_at` タイムスタンプ方式を推奨する。`is_deleted` フラグ方式は削除日時が追跡できないため推奨しない。

**クエリフィルタリングの自動化：** ORM のグローバルフィルター、DBビュー、または RLS を使用し、アプリケーションコードで `WHERE deleted_at IS NULL` を毎回記述する必要がない設計とする。

**物理削除へのアーカイブ：** 論理削除後の保持期間をプロジェクト要件・法令要件に基づき決定し、保持期間経過後に定期パージジョブで物理削除する。

**MongoDB固有：** `deleted_at` フィールドをドキュメントに追加し、RDBと同じタイムスタンプ方式で論理削除する。クエリフィルタリングの自動化にはAggregation Pipelineの `$match` ステージまたはMongoose等のORMミドルウェアを使用する。法令による物理削除が必要な場合はTTLインデックス（`expireAfterSeconds`）を `deleted_at` に設定し、保持期間経過後の自動物理削除を検討する。

**論理削除とUNIQUE制約の衝突対処：** 論理削除済みレコードがUNIQUE制約を占有し、同一値での新規登録を阻害する問題に対処する。PostgreSQLは部分インデックス（`CREATE UNIQUE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL`）で対応する。MySQL / MariaDBは部分インデックスを持たないため、仮想カラム（`GENERATED ALWAYS AS (IF(deleted_at IS NULL, 1, NULL)) VIRTUAL`）を追加し、複合UNIQUE制約（`UNIQUE(email, not_archived)`）で対応する。MongoDBは `deleted_at` の運用方式に合わせて部分インデックスを定義する。削除時のみ `deleted_at` を付与する運用では `partialFilterExpression: { deleted_at: { $exists: false } }`、常時 `deleted_at: null` を保持する運用では `partialFilterExpression: { deleted_at: null }` を使用する。

---

## 10. 監査証跡（Audit Trail）

変更履歴の記録対象はプロジェクトの監査要件に基づき決定する。PII を含むテーブル、金銭に関わるテーブルは原則として監査対象とする。

**記録すべきメタデータ：** who（誰が）、what（何を変更したか）、when（いつ）、where（どのテーブル・カラム）、before（変更前の値）、after（変更後の値）。

**実装パターンの選定：** トリガーベース（DB層での自動記録）、アプリケーションレイヤー（ORMフック / ミドルウェア）、シャドウテーブル / ヒストリーテーブル、イベントソーシングから、パフォーマンス要件と監査要件に応じて選定する。

**運用規定：**
- 監査テーブルは書き込み専用とし、UPDATE / DELETE を禁止する
- 監査データはパーティショニング・アーカイブで管理し、本番テーブルへの性能影響を抑制する
- 改ざん防止が必要な場合はハッシュチェーンを検討する

**MongoDB固有：** MongoDBにはRDBのトリガーに相当する機能がないため、変更検知にはChange Streams（MongoDB 3.6以降）またはアプリケーションレイヤー（Mongooseのpre/postフック等）を使用する。Change Streamsはレプリカセット環境で利用可能であり、oplogに基づくリアルタイム変更通知を提供する。

---

## 11. マイグレーション管理

### 11-1. 基本原則

Schema as Code の原則に基づき、全てのスキーマ変更をマイグレーションファイルで管理する。

### 11-2. ツール選定

| ツール | 対応DB | 特徴 |
|:---|:---|:---|
| Prisma Migrate | PostgreSQL / MySQL / SQLite（MongoDB非対応） | 宣言的スキーマ定義 + SQLマイグレーション生成。MongoDBは `prisma db push` を使用 |
| Flyway | PostgreSQL / MySQL / MariaDB 他 | SQLファイルベースのバージョン管理 |
| Liquibase | マルチDB | XML / SQL / YAML 形式に対応 |
| Atlas | マルチDB | スキーマ駆動型、ORM統合、自動差分計算 |
| Alembic | PostgreSQL / MySQL（Python / SQLAlchemy） | Python プロジェクト向け |

### 11-3. マイグレーションファイル規約

- 命名規約：`V{version}__{description}.sql`（Flyway形式）またはツール既定の命名に従う
- 1マイグレーション = 1原子的変更の原則。無関係な変更を1ファイルにまとめない
- マイグレーションファイルはアプリケーションコードと同一リポジトリで管理する

### 11-4. ロールバック戦略

前方修正（forward fix）を基本とする。ロールバックスクリプトの作成は推奨するが、データ構造の変更後にロールバックするとデータ損失のリスクがあるため、問題発生時は新しいマイグレーションで修正する。

### 11-5. 本番適用フロー

マイグレーションの本番適用は§11-10のスキーマ変更作業フロー（手順3〜8）に従う。ORM自動生成のマイグレーションSQLは本番適用前に必ず人間がレビューする（§11-8と連動）。

### 11-6. 破壊的変更の安全な実行

カラム削除（`DROP COLUMN`）、データ型変更（`ALTER COLUMN ... TYPE`）、カラム名変更（`RENAME COLUMN`）等の破壊的変更は、直接実行を禁止する。段階的に実施する：新カラム追加 → データ移行 → アプリケーションコードの切り替え → 旧カラムの非推奨化 → 十分な猶予期間の後に削除。カラム名変更の場合は、ORM スキーマ定義、既存クエリ、インデックス名、制約名の連動更新を切り替え時に確認する。

### 11-7. CI/CDパイプライン統合

マイグレーションの適用はCI/CDパイプラインに組み込む。手動でのSQL実行は禁止する。

### 11-8. マイグレーション生成後のレビュー義務

ORM が自動生成したマイグレーションSQLは、本番適用前に必ずレビューする。意図しない破壊的変更が含まれていないか確認する。

### 11-9. 既存マイグレーションの改変禁止

適用済みのマイグレーションファイルは改変しない。修正が必要な場合は新しいマイグレーションファイルを作成する。

### 11-10. スキーマ変更の作業・運用規定

スキーマ変更（テーブル追加、カラム追加、カラム変更、テーブル変更）は不可逆性の高い操作であり、データ被害に直結する。全てのスキーマ変更は以下のフローに従う。

**作業フロー（必須手順）：**

1. 変更要件を文書化する（追加するテーブル/カラムの名前、型、制約、DEFAULT値、目的）
2. 既存スキーマへの影響範囲を調査する（参照整合性、インデックス、ORM定義、アプリケーションコード）
3. マイグレーションファイルを作成する（§11の命名規約・1原子的変更の原則に従う）
4. ステージング環境で適用し、動作検証する
5. レビュー承認を得る
6. 本番適用前にバックアップを取得する（§15のバックアップフローに従う）
7. 本番環境に適用する
8. 適用後の整合性を検証する（レコード数、制約の有効性、アプリケーション動作）
9. ER図・テーブル定義書を更新する（§12と連動）

**カラム追加の規定：**

- 新規カラムには `NOT NULL` を原則とする。`NOT NULL` で追加する場合は `DEFAULT` 値の指定が必須（既存レコードへの影響を回避するため）
- NULL許容で追加する場合は、NULL許容の根拠をマイグレーションファイルのコメントに記載する
- §3の命名規則に従う。命名規則に合致しないカラム名は追加を禁止する
- 機密データを含むカラムを追加する場合は、§8-5 / §8-6 の暗号化要件を同時に適用する

**列削除（`DROP COLUMN`）の規定：**

- 列削除は原則として即時実行しない。先にカラムを非推奨（deprecated）として扱い、アプリケーションコードからの参照を除去する
- 列削除の判断は、§9の論理削除原則（不可逆削除を最終手段とする考え方）に準じ、不可逆操作を最小化する
- 実施手順は「非推奨化 → 参照除去完了の確認 → 互換期間の確保 → バックアップ取得 → 専用マイグレーションで物理削除」とする
- 実行前に依存オブジェクト（ビュー、トリガー、関数、インデックス、制約）の影響調査を必須とし、依存を解消してから削除する
- 実行後はER図・テーブル定義書・変更履歴を更新する

**テーブル追加の規定：**

- §3-4 の共通カラム方針に従う（`created_at`, `updated_at` は必須。N:N中間テーブルで複合主キーを採用する場合は `id` を省略可）
- 論理削除対象のテーブルには `deleted_at` カラムを含める（§9と連動）
- 外部キー制約を設計時に定義し、参照先テーブルとの整合性を確認する
- 新規テーブルに必要なインデックスを同一マイグレーションで作成する

**禁止事項：**

- 本番環境への直接SQL実行によるスキーマ変更（マイグレーションツール経由を必須とする）
- マイグレーションファイルを経由しないカラム/テーブルの追加
- 既存カラムの物理削除（`DROP COLUMN`）を、§9の論理削除原則および上記「列削除（`DROP COLUMN`）の規定」を経ずに直接実行すること
- 既存カラムのデータ型変更（`ALTER COLUMN ... TYPE`）を直接実行すること。データ型変更が必要な場合は、新カラム追加 → データ移行 → 旧カラム非推奨化の手順で段階的に実施する
- 複数の無関係なスキーマ変更を1つのマイグレーションにまとめること
- 既存カラムのカラム名変更（`RENAME COLUMN`）を直接実行すること。カラム名変更が必要な場合は§11-6の段階的手順に従う

**MongoDB固有：**

- フィールド追加時は Schema Validation ルール（§5）を同時に更新する
- コレクション追加時は必要なインデックスを同時に作成する
- フィールドのリネームは `$rename` オペレーターで実行し、アプリケーションコードとの同期を確認する

---

## 12. ER図・スキーマ文書化

- ER図は全プロジェクトで作成を義務付ける
- 記法は Crow's Foot 記法を標準とする
- 作成ツール：dbdiagram.io / Mermaid / pgModeler からプロジェクトに応じて選定する
- MongoDB のスキーマ構造はMongoDB Compassのスキーマ分析機能で可視化する
- マイグレーション作成時にER図を連動更新する
- テーブル定義書（カラム名、型、制約、説明、関連テーブル）をスキーマ文書として維持する

---

## 13. 高速化・軽量化（パフォーマンス）

> 汎用的なWebパフォーマンス最適化は [performance-optimization.md](../Documents/ClaudeCode/AI-instructions/performance-optimization.md) を参照。本セクションはDB固有のパフォーマンス対策を規定する。

### 13-1. クエリ最適化

- EXPLAIN / ANALYZE による実行計画の確認を義務化する
- N+1問題を検出・防止する（ORM使用時の `include` / `eager loading` を適切に使用する）
- 不要な `SELECT *` を禁止し、必要カラムのみを指定する
- サブクエリ vs JOIN の使い分けはEXPLAIN結果に基づいて判断する
- 大量データの一括操作にはバッチ INSERT / UPSERT を使用する

### 13-2. 接続管理

接続プーリングを必須化する。

| DB | ツール |
|:---|:---|
| PostgreSQL | PgBouncer / Prisma Connection Pool |
| MySQL | ProxySQL / MySQL Connection Pooling |
| MongoDB | ドライバーの接続プール設定（`maxPoolSize`） |

SQLiteは接続プーリングではなく、単一接続またはWALモードでの読み取り並行化で対応する。接続数はアプリケーションフレームワーク側で制御する。

アイドル接続のタイムアウトと最大接続数をプロジェクト要件に基づき設定する。

### 13-3. キャッシュ戦略

- 頻繁に読み取られるが更新頻度の低いデータにはキャッシュ（Redis / Memcached）を適用する
- PostgreSQL のマテリアライズドビューを集計・レポーティング用途に活用する
- キャッシュ無効化はTTLまたはイベント駆動で設計する

### 13-4. テーブル設計によるパフォーマンス対策

- テーブルパーティショニング（Range / List / Hash）は大量データのテーブルに適用を検討する
- 古いデータのアーカイブテーブルへの分離
- PostgreSQL では VACUUM / ANALYZE を定期実行し、テーブル膨張（bloat）を防止する

### 13-5. MongoDB固有

- 埋め込み vs 参照は読み取り/書き込みのパターンに基づいて性能トレードオフを判断する
- Aggregation Pipeline は `$match` / `$project` をパイプライン先頭に配置して処理対象を早期に絞り込む
- シャーディングはデータ量とスループット要件に基づいて検討する

---

## 14. トランザクション管理

### 14-1. RDB

- ACID特性を遵守する
- トランザクション分離レベルは `READ COMMITTED` を既定とする。より強い分離が必要な場合のみ `REPEATABLE READ` / `SERIALIZABLE` を使用する
- デッドロック防止：リソースのロック順序を統一する。長時間トランザクションを避ける
- トランザクションスコープは必要最小限に保つ

### 14-2. MongoDB

- マルチドキュメントトランザクション（MongoDB 4.0以降）は、複数コレクションへの原子的更新が必要な場合に使用する。単一ドキュメント操作は既定で原子的であるため不要
- 楽観的同時実行制御（バージョンフィールドによる更新検知）をデータ競合が発生する箇所に適用する

### 14-3. SQLite

- SQLiteは単一ライター制限を持つ。同一時点で書き込みできるのは1プロセスのみ
- WALモード（`PRAGMA journal_mode=WAL`）を有効化すると、読み取りと書き込みの同時実行が可能になる。本番利用時はWALモードを推奨する
- 同時書き込みが競合した場合は `SQLITE_BUSY` エラーが返る。`busy_timeout`（`PRAGMA busy_timeout=5000` 等）を設定し、リトライを自動化する
- トランザクション分離レベルの概念はRDBと異なり、SQLiteはシリアライザブルに近い動作をする

出典・参考情報: PostgreSQL Documentation、MySQL Reference Manual、MongoDB Documentation、SQLite Documentation

---

## 15. バックアップ・リカバリ

### 15-1. バックアップ戦略の種別

- フルバックアップ：DB全体のスナップショット。リカバリの起点として必須
- 差分バックアップ：前回フルバックアップ以降の変更分
- インクリメンタルバックアップ：前回バックアップ（フルまたはインクリメンタル）以降の変更分
- 継続的ログアーカイブ：トランザクションログを連続的に保存し、PITRの基盤とする

### 15-2. バックアップフロー（必須手順）

1. フルバックアップを定期取得する（頻度はプロジェクト要件に基づき決定）
2. トランザクションログ（WAL / binlog / oplog）の継続アーカイブを有効化する
3. バックアップ完了後、整合性を検証する（`pg_verifybackup` 等）
4. バックアップデータを本番DBとは異なる場所（別サーバー / 別リージョン / オフサイト）に保管する
5. バックアップデータを暗号化する（保存時暗号化必須）
6. リストア手順を定期的にテストする（リストア訓練の実施義務）

### 15-3. DB別バックアップツールとログ機構

| DB | フルバックアップ | トランザクションログ | PITR基盤 |
|:---|:---|:---|:---|
| PostgreSQL | `pg_basebackup` / pgBackRest / Barman | WAL（Write-Ahead Log） | WALアーカイブ + ベースバックアップ |
| MySQL | `mysqldump` / MySQL Enterprise Backup / Percona XtraBackup | バイナリログ（binlog） | binlog + フルバックアップ |
| MariaDB | `mariadb-dump` / `mariadb-backup` | バイナリログ（binlog） | binlog + フルバックアップ |
| MongoDB | `mongodump` / Percona Backup for MongoDB / Atlas自動バックアップ | oplog（Operation Log） | oplogスライス + スナップショット |
| SQLite | ファイルコピー / `.backup` コマンド / Litestream | なし（単一ファイルDB） | Litestreamによる WAL ストリーミング、またはファイルコピーの世代管理 |

### 15-4. Point-in-Time Recovery（PITR）— 安全なリワインド方法

PITRは「フルバックアップ + トランザクションログの再生」により、DBを任意の時点の状態に復元する手法。誤操作からの復旧に不可欠。

**共通フロー：**

1. 直前のフルバックアップをリストアする
2. トランザクションログを目標時刻まで順次再生（リプレイ）する
3. 目標時刻到達後、リストア結果を検証する
4. 検証完了後に通常運用に切り替える

**PostgreSQL の PITR 手順：**

1. `archive_mode = on` / `wal_level = replica` を設定し、WALアーカイブを有効化する
2. `pg_basebackup` でベースバックアップを定期取得する
3. 復旧時：ベースバックアップをリストアし、`recovery_target_time` に目標時刻を指定してWALを再生する
4. 目標時刻到達後の遷移は `recovery_target_action` に従う。`pause` の場合は `SELECT pg_wal_replay_resume()` で再開し、`promote` の場合は自動で通常運用へ移行する
5. 補助ツール：pgBackRest（差分・インクリメンタル・パラレルリストア対応）、Barman（リモートバックアップ管理）
6. 高速リワインド手段：遅延レプリカ（`recovery_min_apply_delay` 設定）を用意しておくと、数時間前の状態へ即座に切り替え可能

**MySQL / MariaDB の PITR 手順：**

1. バイナリログを有効化する（MySQL は 8.0 以降で既定有効だが、初期化時や `--skip-log-bin` 指定時は無効になり得るため `log_bin` を確認する。MariaDB は設定値を明示して有効化する）
2. `mysqldump` または Percona XtraBackup でフルバックアップを取得する
3. 復旧時：フルバックアップをリストアし、`mysqlbinlog` でバイナリログを目標時刻 / イベント位置まで再生する
4. `--start-position` / `--stop-position` を使用する（`--start-datetime` / `--stop-datetime` はイベント欠落リスクがあるため非推奨）

**MongoDB の PITR 手順：**

1. レプリカセットの oplog サイズを十分に確保する
2. `mongodump` または Percona Backup for MongoDB でスナップショットを取得する
3. 復旧時：スナップショットをリストアし、oplog を目標時刻まで再生する（`--oplogReplay` / `--oplogLimit`）
4. Atlas 使用時は管理画面から目標時刻を指定して自動復元する

**SQLite の復旧方法：**

- 単一ファイルDBのため、トランザクションログベースのPITRは標準機能としては存在しない
- Litestream を使用したWALストリーミングによる準リアルタイムバックアップ、または定期ファイルコピーの世代管理で対応する
- 復旧はバックアップファイルの差し替え

### 15-5. バックアップ保持・ローテーション

- 保持期間はプロジェクト要件・法令要件に基づき決定する
- ローテーションポリシーの例：日次7世代 + 週次4世代 + 月次12世代
- 古いバックアップの自動削除はスクリプト化し、手動削除を禁止する

### 15-6. リストア訓練（必須）

- リストア手順は文書化し、定期的に実行する（四半期に1回以上を推奨）
- PITRの目標時刻指定による復元を含めた訓練とする
- 訓練結果（所要時間・成否・課題）を記録する

出典・参考情報: PostgreSQL: Continuous Archiving and PITR、MySQL: Point-in-Time Recovery Using Binary Log、pgBackRest Documentation、Percona Backup for MongoDB、Litestream Documentation

---

## 16. 監視・運用

- スロークエリの検出・通知設定（PostgreSQL: `log_min_duration_statement`、MySQL: `slow_query_log`）
- 接続数・クエリ実行数・レプリケーション遅延の監視
- MongoDB固有：`db.currentOp()` によるアクティブ操作の監視、Database Profiler（`db.setProfilingLevel()`）によるスロークエリ検出、Atlas使用時はPerformance Advisorによるインデックス推奨の活用
- ディスク使用量・テーブルサイズの監視
- 定期的なインデックスメンテナンス（不要インデックスの検出・削除）
- 統計情報の更新スケジュール（PostgreSQL: `ANALYZE`、MySQL: `ANALYZE TABLE`）
- SQLite固有：`PRAGMA integrity_check` による定期的なDB整合性検証、`PRAGMA optimize` による統計情報更新、DBファイルサイズの監視（`VACUUM` によるファイルサイズ最適化）

**環境分離（Dev / Staging / Production）：**

- DB環境はDev（開発）、Staging（検証）、Production（本番）の3環境を原則とする
- Staging環境のDB構成（バージョン、拡張機能、設定パラメーター）はProductionと同一に維持する
- シードデータ（開発・テスト用の初期データ）はマイグレーション完了後にシードスクリプトで投入する。シードスクリプトはスキーマ定義と分離し、データ投入のみを行う
- 本番データをDev / Staging環境に複製する場合は、個人情報のマスキング（匿名化）を必須とする（§8-6のデータ分類に基づき、機密度：高・中のデータを対象とする）
- 開発者のProduction環境への直接接続を禁止する。全ての変更はCI/CDパイプライン経由で適用する（§11-7と連動）

出典・参考情報: PostgreSQL Documentation、MySQL Reference Manual、MongoDB Documentation、SQLite Documentation

---

## 17. ORM / クエリビルダー使用規約

- ORM使用時の生SQL実行にはパラメータバインドを必須とする（§8-1 と連動）
- N+1問題の防止を実装パターンとして徹底する（`include` / `eager loading` / `select` の適切な使用）
- ORM が自動生成するマイグレーションSQLは本番適用前にレビューを必須とする（§11-8 と連動）
- ORM固有の制約・注意点はプロジェクト開始時に確認し、文書化する
- 主要な選択肢：Prisma（PostgreSQL / MySQL / SQLite / MongoDB対応）、Drizzle（TypeScript型安全）、Sequelize（Node.js）、Mongoose（MongoDB専用）からプロジェクト要件に応じて選定する

---

## 18. データベース移行（DB方式の変更）

DBエンジンの変更（PostgreSQL → MySQL、RDB → MongoDB 等）はスキーマ変更よりも影響範囲が広く、データ損失・ダウンタイム・アプリケーション障害のリスクが高い。以下のフローに従い段階的に実施する。

### 18-1. 移行判断の前提条件

DB方式の変更は以下のいずれかに該当する場合にのみ検討する。現行DBのチューニングやアーキテクチャー変更で解決できる場合は移行しない。

- 現行DBがプロジェクトの技術要件（スケーラビリティー、データモデル、クエリパターン）を構造的に満たせない
- ベンダーロックインの解消やコスト最適化が経営判断として確定している
- 現行DBのサポート終了（EOL）が確定している

### 18-2. 移行計画の策定（必須手順）

1. 移行元と移行先のDB間でデータ型・制約・機能の差異を調査し、互換性マトリクスを作成する
2. 移行対象のスキーマ・データ量・依存するアプリケーションコードの範囲を特定する
3. 移行方式を選定する（18-3参照）
4. ロールバック計画を策定する（移行失敗時に元のDBへ復帰する手順）
5. 許容ダウンタイムとデータ整合性の要件を定義する
6. 移行スケジュールを策定し、関係者の承認を得る

### 18-3. 移行方式の選定

| 方式 | 概要 | 適用場面 |
|:---|:---|:---|
| オフライン移行 | サービスを停止し、データをエクスポート → 変換 → インポートする | 許容ダウンタイムがある小中規模システム |
| オンライン移行（段階的） | 新旧DBを並行稼働させ、書き込みを二重化した後に切り替える | ダウンタイムが許容されない本番システム |
| ストリーミング移行 | CDC（Change Data Capture）ツール（Debezium、AWS DMS、pglogical等）で変更を継続的に同期し、切り替えタイミングで新DBに移行する | 大規模データ・高可用性が求められるシステム |

### 18-4. データ変換の規定

DB移行時のデータ変換はデータ損失の最大リスク要因となる。移行前に以下のマッピング表を基礎として、プロジェクト固有の互換性マトリクスを作成する。

#### 18-4-1. RDB間の主要データ型マッピング

| 用途 | PostgreSQL | MySQL / MariaDB | SQLite | 移行時の注意 |
|:---|:---|:---|:---|:---|
| 小整数 | `SMALLINT` | `TINYINT` / `SMALLINT` | `INTEGER`（INTEGER affinity） | PostgreSQLに `TINYINT` は存在しない。MySQL `TINYINT` → PostgreSQL `SMALLINT` に変換する。MySQL `UNSIGNED` 属性はPostgreSQLに存在しないため、CHECK制約で代替する |
| 整数 | `INTEGER` | `INT` | `INTEGER` | MySQL `INT UNSIGNED`（0〜4,294,967,295）→ PostgreSQL `BIGINT` に変換する（`INTEGER` では範囲不足） |
| 大整数 | `BIGINT` | `BIGINT` | `INTEGER`（8バイト整数） | MySQL `BIGINT UNSIGNED` → PostgreSQLに直接対応する型がない。`NUMERIC(20,0)` で代替する |
| 自動採番 | `SERIAL` / `BIGSERIAL`（PostgreSQL 10以降は `GENERATED ALWAYS AS IDENTITY` 推奨） | `AUTO_INCREMENT` | `INTEGER PRIMARY KEY`（rowid自動採番） | `SERIAL` はシーケンスオブジェクトを生成する。MySQL → PostgreSQL 移行時はシーケンスの初期値を現在の最大値以上に設定する |
| 固定小数点（金銭） | `NUMERIC` / `DECIMAL` | `DECIMAL` / `NUMERIC` | `NUMERIC`（NUMERIC affinity、ただし内部はTEXTまたはREALで保存） | 精度（precision）と位取り（scale）の指定を移行先でも一致させる。SQLiteのNUMERIC affinityは厳密な固定小数点ではないため、金銭データの精度は移行後に検証が必要 |
| 浮動小数点 | `REAL`（4バイト）/ `DOUBLE PRECISION`（8バイト） | `FLOAT`（4バイト）/ `DOUBLE`（8バイト） | `REAL`（8バイトIEEE浮動小数点） | PostgreSQL `REAL` = MySQL `FLOAT`（4バイト）。SQLite `REAL` は常に8バイト |
| 固定長文字列 | `CHAR(n)` | `CHAR(n)` | `TEXT`（TEXT affinity） | SQLiteは固定長を保証しない。RDB間ではパディング動作の差異を確認する |
| 可変長文字列 | `VARCHAR(n)` / `TEXT` | `VARCHAR(n)` / `TEXT` / `MEDIUMTEXT` / `LONGTEXT` | `TEXT` | PostgreSQL `TEXT` = MySQL `TEXT` だが、MySQL `TEXT` は65,535バイト上限。PostgreSQLの `TEXT` に上限はない。MySQL `MEDIUMTEXT` / `LONGTEXT` → PostgreSQL `TEXT` に変換する |
| 真偽値 | `BOOLEAN`（`TRUE` / `FALSE`） | `TINYINT(1)` / `BOOLEAN`（`TINYINT(1)` の別名） | `INTEGER`（0 / 1） | MySQL `BOOLEAN` は内部的に `TINYINT(1)` で保存される。移行時に値の変換（0/1 ↔ true/false）を確認する |
| 日付 | `DATE` | `DATE` | `TEXT`（ISO 8601文字列）/ `INTEGER`（Unixタイムスタンプ） | MySQL `DATE` は `0000-00-00` を許容するがPostgreSQLは許容しない。移行時に `0000-00-00` を `NULL` に変換する。SQLiteは日付専用型がないため、保存形式（TEXT / INTEGER）を統一する |
| 日時（タイムゾーン付き） | `TIMESTAMP WITH TIME ZONE` | `TIMESTAMP`（UTC変換で保存、取得時にセッションTZで変換） | `TEXT` / `INTEGER` | PostgreSQLは入力値をUTCに正規化して内部保存し、出力時にセッションTZへ再変換する（元のタイムゾーン文字列は保持しない）。MySQLの `TIMESTAMP` もUTC変換されるため、両者の動作差異をテストで確認する |
| 日時（タイムゾーンなし） | `TIMESTAMP WITHOUT TIME ZONE` | `DATETIME` | `TEXT` / `INTEGER` | MySQL `DATETIME` はタイムゾーン変換をしない。PostgreSQL `TIMESTAMP WITHOUT TIME ZONE` と同等 |
| JSON | `JSON` / `JSONB` | `JSON` | `TEXT` | PostgreSQL `JSONB` はバイナリー格納でインデックス作成・検索が可能。MySQL `JSON` はバイナリー格納だがインデックス対応が限定的。移行時にJSON構造の妥当性を再検証する |
| バイナリー | `BYTEA` | `BLOB` / `MEDIUMBLOB` / `LONGBLOB` | `BLOB` | 型名と最大サイズが異なる。MySQL `LONGBLOB`（最大4GB）→ PostgreSQL `BYTEA`（最大1GB）はサイズ制限に注意 |
| UUID | `UUID`（ネイティブ型） | `CHAR(36)` / `BINARY(16)` | `TEXT` / `BLOB` | PostgreSQLはUUIDネイティブ型を持つ。MySQL / SQLiteは文字列またはバイナリーで保存する。移行時にフォーマット（ハイフン有無、大文字小文字）を統一する |
| 配列 | `ARRAY`（例：`INTEGER[]`） | 非対応 | 非対応 | PostgreSQL固有。MySQL / SQLiteへの移行時はJSON配列または正規化（子テーブル）に変換する |
| 列挙型 | `ENUM`（CREATE TYPEで定義） | `ENUM` | 非対応 | PostgreSQLとMySQLの `ENUM` は実装が異なる（PostgreSQLは独立した型、MySQLはカラム定義に内包）。移行時はCHECK制約またはルックアップテーブルへの変換を検討する |

#### 18-4-2. RDB → MongoDB 変換の指針

| RDB型 | MongoDB型 | 注意 |
|:---|:---|:---|
| `INTEGER` / `BIGINT` | `Int32` / `Int64` | 範囲を確認して適切な型を選択する |
| `DECIMAL` / `NUMERIC` | `Decimal128` | 金銭データは `Decimal128` 必須。`Double` は丸め誤差があるため使用しない |
| `VARCHAR` / `TEXT` | `String` | 長さ制限はSchema Validationで実装する |
| `BOOLEAN` | `Boolean` | ネイティブ対応 |
| `TIMESTAMP` | `Date` | MongoDBの `Date` はミリ秒精度のUTCタイムスタンプ。マイクロ秒精度が必要な場合は文字列保存を検討する |
| `BYTEA` / `BLOB` | `BinData` | GridFSは16MB超のファイルに使用する |
| `UUID` | `UUID`（BSON Binary subtype 4）/ `String` | ドライバーの対応状況を確認する |
| `JSON` / `JSONB` | ドキュメント構造に直接展開 | JSONカラムの内容はMongoDBのネイティブドキュメント構造に変換する |

#### 18-4-3. MongoDB → RDB 変換の指針

| MongoDB型 | PostgreSQL | MySQL | 注意 |
|:---|:---|:---|:---|
| `ObjectId` | `CHAR(24)` / `BYTEA` | `CHAR(24)` / `BINARY(12)` | 文字列保存（24文字の16進数）またはバイナリー保存を選択する。PostgreSQLで12バイト固定を担保する場合は `CHECK (octet_length(object_id_bin)=12)` を併用する。新規サロゲートキーの付与も検討する |
| `Decimal128` | `NUMERIC` / `DECIMAL` | `DECIMAL` | 精度と位取りを明示的に指定する |
| `Date` | `TIMESTAMP WITH TIME ZONE` | `DATETIME` / `TIMESTAMP` | MongoDBの `Date` はUTCミリ秒。タイムゾーン取り扱い方針を統一する |
| 埋め込みドキュメント | 正規化（子テーブル）/ `JSONB` | 正規化（子テーブル）/ `JSON` | §2-1に基づき3NFを基準として正規化する。頻繁に同時取得するデータは `JSONB` / `JSON` カラムでの保持も検討する |
| 配列フィールド | 正規化（中間テーブル）/ `ARRAY`（PostgreSQL）/ `JSON` | 正規化（中間テーブル）/ `JSON` | 要素数が少なく固定的ならJSON、可変・大量なら正規化を選択する |

#### 18-4-4. 変換時の共通注意事項

- 文字コード・照合順序（collation）の差異を確認し、データの文字化けやソート順序の変化を防止する
- 日時データのタイムゾーン取り扱いの差異を確認する（UTC保存の統一を推奨）
- 自動採番（シーケンス / AUTO_INCREMENT / ObjectId）の移行方針を決定する。移行先のシーケンス初期値は、移行データの最大値を超える値に設定する
- NULL / デフォルト値の動作差異を確認する（MySQL `0000-00-00` 問題、SQLiteの動的型付け等）
- マッピング表はプロジェクト固有のカラムを追記して互換性マトリクスとして完成させ、移行計画書に含める

出典・参考情報: PostgreSQL Data Types、MySQL Data Types、SQLite Datatypes、MongoDB BSON Types

### 18-5. 移行実行フロー

1. 移行元DBのフルバックアップを取得する（§15のバックアップフローに従う）
2. ステージング環境で移行を実行し、データ整合性を検証する
3. 検証項目：レコード数の一致、制約の有効性、外部キー参照の整合性、アプリケーション動作テスト
4. 本番環境で移行を実行する
5. 移行後の整合性検証を実施する（ステージングと同一の検証項目）
6. 問題がなければ旧DBを読み取り専用にし、移行完了後の監視期間（最低1週間を推奨）を設ける
7. 監視期間終了後、旧DBのアーカイブと停止を実施する

### 18-6. アプリケーションコードの対応

- ORM / クエリビルダーの変更（ドライバー差し替え、スキーマ定義の更新）
- DB固有のSQL構文・関数の書き換え（PostgreSQL固有関数、MySQL固有関数等）
- 接続設定（接続文字列、SSL/TLS設定、接続プール設定）の更新
- トランザクション分離レベルの差異への対応
- テストスイートの全件実行による動作検証

### 18-7. 禁止事項

- 移行計画・互換性マトリクスなしでの本番移行
- ロールバック計画なしでの移行実行
- ステージング環境での検証を省略した本番移行
- 移行期間中の旧DBスキーマ変更（移行対象が変動するため）

---

## 出典・参考情報

### 情報・規格

- [OWASP Database Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html) - DB セキュリティの包括的なガイドライン
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) - SQLインジェクション防止
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) - パスワードハッシュ化の推奨仕様
- [OWASP Transport Layer Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html) - TLS構成の推奨事項

### DB公式ドキュメント

- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/) - PostgreSQL公式
- [PostgreSQL: Data Types](https://www.postgresql.org/docs/current/datatype.html) - PostgreSQLデータ型リファレンス
- [PostgreSQL: Continuous Archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html) - PostgreSQL PITRガイド
- [MySQL Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/) - MySQL公式
- [MySQL: Data Types](https://dev.mysql.com/doc/refman/8.4/en/data-types.html) - MySQLデータ型リファレンス
- [MySQL: PostgreSQL Type Mapping](https://dev.mysql.com/doc/workbench/en/wb-migration-database-postgresql-typemapping.html) - MySQL Workbench公式の型マッピング表
- [MySQL: Point-in-Time Recovery Using Binary Log](https://dev.mysql.com/doc/refman/8.4/en/point-in-time-recovery-binlog.html) - MySQL PITRガイド
- [MongoDB Documentation](https://www.mongodb.com/docs/) - MongoDB公式
- [MongoDB: BSON Types](https://www.mongodb.com/docs/manual/reference/bson-types/) - MongoDBデータ型リファレンス
- [SQLite Documentation](https://www.sqlite.org/docs.html) - SQLite公式
- [SQLite: Datatypes](https://www.sqlite.org/datatype3.html) - SQLiteデータ型・型親和性リファレンス

### ツール公式ドキュメント

- [Prisma Documentation](https://www.prisma.io/docs/) - Prisma ORM / Migrate
- [Flyway Documentation](https://documentation.red-gate.com/fd) - Flyway マイグレーションツール
- [Liquibase Documentation](https://docs.liquibase.com/) - Liquibase マイグレーションツール
- [Atlas Documentation](https://atlasgo.io/docs) - Atlas スキーマ管理
- [pgBackRest Documentation](https://pgbackrest.org/) - PostgreSQLバックアップツール
- [Percona Backup for MongoDB](https://docs.percona.com/percona-backup-mongodb/) - MongoDB バックアップツール
- [Litestream Documentation](https://litestream.io/) - SQLite WALストリーミング

### 関連ファイル

- [secure-code-requirements.md](../Documents/ClaudeCode/AI-instructions/secure-code-requirements.md) - セキュリティ要件（§8との連動元）
- [performance-optimization.md](../Documents/ClaudeCode/AI-instructions/performance-optimization.md) - パフォーマンス最適化（§13との棲み分け元）
