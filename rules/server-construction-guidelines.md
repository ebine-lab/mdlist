---
title: "Server Construction Guidelines"
description: "サーバーアプリケーション・クラウド/VPS構築（AWS / Google Cloud / Azure / OCI）・コンテナ技術・セキュリティ・パフォーマンス最適化・デプロイ・監視の共通規約 / Server application, cloud/VPS infrastructure (AWS, Google Cloud, Azure, OCI), container technology, security, performance optimization, deployment, and monitoring standards"
version: "1.0.0"
status: "Stable"
last_updated: "2026-03-22T05:18+09:00"
lang: "ja"
---

# Server Construction Guidelines

**説明** - サーバーアプリケーション（Apache / nginx）の設定、クラウド・VPS構築（AWS / Google Cloud / Azure / Oracle Cloud Infrastructure / 国内VPS）、コンテナ技術（Docker / Kubernetes）、セキュリティ強化、パフォーマンス最適化、デプロイパイプライン、監視に関する共通規約。


---

## 対象ソフトウェア・プラットフォーム一覧

| 分類 | 対象 | 用途 |
|:---|:---|:---|
| Webサーバー | nginx | リバースプロキシ・静的配信・ロードバランサー |
| Webサーバー | Apache httpd | Webサーバー・.htaccess依存環境 |
| クラウド | AWS | EC2 / VPC / ALB / CloudFront / Route 53 / IAM / ACM |
| クラウド | Google Cloud | Compute Engine / Cloud Load Balancing / Cloud CDN / Cloud DNS |
| クラウド | Azure | VM / Application Gateway / Front Door / Azure DNS / Key Vault |
| クラウド | Oracle Cloud Infrastructure (OCI) | Compute / VCN / Load Balancer / WAF / Cloud Guard / Vault |
| CDN / エッジ | Cloudflare | DNS / CDN / WAF / DDoS防御 / SSL / エッジコンピューティング |
| VPS | さくらVPS / ConoHa VPS | 国内VPS（自前サーバー管理） |
| コンテナ | Docker / Docker Compose | コンテナビルド・ローカル〜本番オーケストレーション |
| コンテナ | Kubernetes | コンテナオーケストレーション（マネージド: EKS / GKE / AKS） |
| IaC | Terraform | マルチクラウドインフラストラクチャー管理 |
| 構成管理 | Ansible | サーバー構成自動化 |
| 監視 | Prometheus + Grafana | メトリクス収集・可視化 |
| APM | OpenTelemetry | 分散トレーシング・計装 |

---

## 非対象

- アプリケーションコード自体の設計・実装（→ [coding-standards.md](/Users/TED/Documents/ClaudeCode/AI-instructions/coding-standards.md)、[component-design-patterns.md](/Users/TED/Documents/ClaudeCode/AI-instructions/component-design-patterns.md)）
- データベースサーバーの設計・運用（→ [database-design-guidelines.md](/Users/TED/Documents/ClaudeCode/AI-instructions/database-design-guidelines.md)）
- Cloudflare Workers / Pages のアプリケーションロジック実装詳細（ルーティング・キャッシュ制御設定は本書で扱う）
- ドメイン取得・移管手続き（→ [domain-placement-matrix.md](/Users/TED/Documents/ClaudeCode/AI-instructions/domain-placement-matrix.md)）
- OAuth / 認証プロバイダーの実装詳細（→ 将来の oauth-login-providers.md）
- 外部API連携の実装詳細（→ 将来の external-api-integration.md）
- モバイルアプリ固有のサーバー設定
- メールサーバー構築（Postfix / Dovecot 等）

---

## 1. Webサーバー選定基準

### 1-1. nginx vs Apache の判断基準

| 観点 | nginx | Apache httpd |
|:---|:---|:---|
| アーキテクチャー | イベント駆動・非同期 | プロセス / スレッドベース（Event MPM推奨） |
| 静的ファイル配信 | 高速（非同期I/O） | 標準的 |
| リバースプロキシ | 標準機能で高性能 | mod_proxy で対応 |
| .htaccess | 非対応 | 対応（パフォーマンスコスト有り） |
| HTTP/3（QUIC） | 1.25.0以降で利用可能（実験的） | 非対応（現行2.4系） |
| 動的モジュール | ビルド時または動的ロード | 実行時ロード（apxs） |
| 設定リロード | `nginx -s reload`（ゼロダウンタイム） | `graceful`（若干の遅延あり） |

nginx をリバースプロキシ・静的配信・ロードバランサーとして使用し、.htaccess 依存のレガシー環境のみ Apache を選択する。HTTP/3 が必要な場合は nginx を前段に配置する。

### 1-2. バージョンポリシー

**nginx** はデュアルブランチモデルを採用する。stable ブランチ（偶数番号）は重大なバグ修正中心、mainline ブランチ（奇数番号）は新機能を含む。nginx 公式チームは **mainline を本番環境に推奨** している。変更凍結要件がある場合のみ stable を選択する。

**Apache httpd** は単一アクティブブランチ **2.4.x** を維持する。正式なLTS指定はないため、2.4系の最新リリースへの継続追従を**必須**とする。運用対象は常に最新の2.4.xリリースとする。

固定バージョン番号は陳腐化しやすいため、運用対象の固定値は保持しない。互換性や設定差分の説明に必要な最小バージョンのみ記載する。適用前は、導入済みバージョン確認と公式最新リリース確認を分けて実施する。

```bash
# 導入済みバージョン確認（ローカル環境）
nginx -v
apachectl -v
```

最新リリースは公式情報で確認する。

- nginx: `https://nginx.org/en/CHANGES`
- Apache httpd: `https://downloads.apache.org/httpd/`

出典・参考情報: nginx公式、Apache httpd公式

---

## 2. Webサーバー共通設定

### 2-1. バーチャルホスト

推奨ディレクトリ構造：

```
/var/www/
├── example.com/
│   ├── public/       # DocumentRoot / root
│   └── logs/         # Per-site logs (optional)
```

**nginx**（`/etc/nginx/conf.d/example.com.conf`）：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    root /var/www/example.com/public;
    index index.html;

    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log warn;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }
}
```

**Apache**（`/etc/apache2/sites-available/example.com.conf`）：

```apache
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example.com/public
    ErrorLog ${APACHE_LOG_DIR}/example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/example.com-access.log combined

    <Directory /var/www/example.com/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

**【構成差分】** nginx は `conf.d/` 配下のファイルが自動で読み込まれる（`include /etc/nginx/conf.d/*.conf;` による）。Apache は `a2ensite example.com.conf` で有効化し `systemctl reload apache2` で反映する。nginx は `.htaccess` に相当する機能を持たないため、全設定をサーバー設定ファイルに集約する。

### 2-2. リバースプロキシとロードバランシング

**nginx** は5種のロードバランシング手法を提供する：`round-robin`（既定）、`least_conn`、`ip_hash`、`hash $key`、`random two least_conn`。

```nginx
upstream backend_app {
    least_conn;
    server 127.0.0.1:3000 weight=3;
    server 127.0.0.1:3001 weight=2;
    server 127.0.0.1:3002 backup;
    keepalive 32;
}

server {
    listen 443 ssl;
    server_name app.example.com;

    location / {
        proxy_pass http://backend_app;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Connection "";
        proxy_read_timeout 90s;
        proxy_connect_timeout 5s;
    }
}
```

**Apache** は mod_proxy_balancer で4種を提供する：`byrequests`、`bytraffic`、`bybusyness`、`heartbeat`。

```apache
<Proxy balancer://backend_cluster>
    BalancerMember http://127.0.0.1:3000 loadfactor=3
    BalancerMember http://127.0.0.1:3001 loadfactor=2
    BalancerMember http://127.0.0.1:3002 status=+H
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass        / balancer://backend_cluster/
ProxyPassReverse / balancer://backend_cluster/
ProxyPreserveHost On
```

**【構成差分】** nginx の `keepalive 32` は upstream への持続接続数を制御し、`proxy_set_header Connection ""` と組み合わせて使用する。Apache は mod_proxy で `keepalive=On` と `connectiontimeout` で制御する。nginx はアクティブヘルスチェックが商用版（NGINX Plus）のみ、OSS版ではパッシブチェック（`max_fails`、`fail_timeout`）で対応する。Apache は mod_proxy_hcheck でアクティブヘルスチェックを標準提供する。

出典・参考情報: nginx公式ドキュメント、Apache mod_proxy公式ドキュメント

### 2-3. SSL/TLS設定

Mozilla Intermediate プロファイルを基準とする。TLS 1.2を最低バージョン、TLS 1.3を推奨とし、HSTS、OCSP Stapling を有効化する。

**TLS 1.3 暗号スイート**（TLSスタックが自動設定）：

```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
```

**TLS 1.2 暗号スイート（Intermediate）：**

```
ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
```

**nginx SSL設定：**

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
resolver 1.1.1.1 1.0.0.1 valid=300s;

add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

**Apache SSL設定：**

```apache
SSLEngine on
SSLProtocol -all +TLSv1.2 +TLSv1.3
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
SSLHonorCipherOrder off

SSLUseStapling On
SSLStaplingCache shmcb:/var/run/apache2/ssl_stapling(128000)

Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
```

**【構成差分】** nginx は `ssl_stapling` ディレクティブで個別のserver ブロックごとにOCSP Staplingを設定する。Apache は `SSLStaplingCache` をグローバルに定義し、各VirtualHostで `SSLUseStapling On` を設定する。セッションキャッシュは nginx が `shared:SSL:10m` で共有メモリーを使用し、Apache は `shmcb` でサイクリックバッファーを使用する。HSTS preload の最低要件は `max-age>=31536000`（1年以上）であり、`63072000`（2年）は推奨設定である。

出典・参考情報: Mozilla Server Side TLS、Mozilla SSL Configuration Generator

### 2-4. HTTP/2・HTTP/3（QUIC）

nginx 1.25.1以降、HTTP/2は `listen` パラメーターではなく独立ディレクティブで有効化する。HTTP/3（QUIC）は nginx 1.25.0以降で利用可能だが実験的機能であるため、採用時は公式互換情報の確認と十分な負荷試験を前提とする。TLS 1.3とUDPポート443の開放は必須。

```nginx
server {
    listen 443 ssl;
    http2 on;

    # HTTP/3 (QUIC over UDP)
    listen 443 quic reuseport;
    http3 on;
    # 0-RTT はリプレイ対策が必要なため既定では無効
    ssl_early_data off;
    quic_retry on;

    add_header Alt-Svc 'h3=":443"; ma=86400' always;
}
```

**Apache** は mod_http2 でHTTP/2に対応する。HTTP/3は現行2.4系では非対応。

```apache
Protocols h2 http/1.1
# HTTP/2 Server Push は主要ブラウザーで実質廃止
H2Push off
H2ModernTLSOnly on
```

**【構成差分】** Apache でHTTP/3が必要な場合は、nginx または Caddy を前段に配置し、Apache をバックエンドとして使用する構成を採用する。nginx の `reuseport` はHTTP/3 の `listen` ディレクティブに付与し、複数worker間でUDPソケットを共有する。Apache の `H2Push` はHTTP/2 Server Pushだが、主要ブラウザーが2022〜2024年にServer Push対応を廃止したため、`103 Early Hints` と `<link rel="preload">` で代替する。

出典・参考情報: nginx HTTP/3公式ドキュメント、Apache mod_http2公式ドキュメント

### 2-5. セキュリティヘッダー

```nginx
# nginx — server {} ブロック内に配置
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), camera=(), microphone=()" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'self'; object-src 'none'; base-uri 'self';" always;
```

```apache
# Apache — VirtualHost または .conf 内
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Permissions-Policy "geolocation=(), camera=(), microphone=()"
Header always set Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'self'; object-src 'none'; base-uri 'self';"
```

`X-XSS-Protection` は廃止済み。主要ブラウザーがXSS Auditorを削除しており、Content-Security-Policy で代替する。CSPの値はプロジェクトの要件に応じてカスタマイズする。

出典・参考情報: MDN HTTP Headers、OWASP Secure Headers Project

### 2-6. レートリミットとDDoS緩和

**nginx：**

```nginx
# http {} ブロック — ゾーン定義
limit_req_zone $binary_remote_addr zone=general:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=1r/s;
limit_conn_zone $binary_remote_addr zone=perip:10m;
limit_req_status 429;

# location ブロック — 適用
location / {
    limit_req zone=general burst=20 nodelay;
    limit_conn perip 20;
}
location /login {
    limit_req zone=login burst=5 nodelay;
}
```

10mゾーンで追跡可能なIPエントリー数は保存状態とキー長で変動し、固定値ではない。`$binary_remote_addr` は可変長の文字列表現ではなく、IPv4を4バイト、IPv6を16バイトのバイナリーで格納するため効率が良い。

**Apache** は mod_ratelimit（帯域制限）と mod_evasive（DDoS緩和）で対応する。

```apache
# mod_ratelimit — 帯域制限
<Location /downloads>
    SetOutputFilter RATE_LIMIT
    SetEnv rate-limit 512
</Location>
```

**【構成差分】** nginx はIPアドレス単位のリクエストレート制限をネイティブに提供する。Apache は mod_evasive でDDoSパターン検知を行うが、nginx の `limit_req` のような細かいバースト制御はない。高度なレートリミットが必要な場合はCloudflare Rate Limiting（§3-5-5）またはAWS WAF Rate-based Rules（§4-6-3）をWebサーバーの前段に配置する。

#### 2-6-1. SYN Flood対策（nginx / Apache 共通）

SYN FloodはTCP 3-way handshakeの初期段階（SYN受信）を枯渇させる攻撃であり、アプリケーション層のレートリミットだけでは防ぎ切れない。防御は **カーネル（SYNキュー）・L4/L7前段・Webサーバー設定** の3層で実施する。

**OS（Linux）最小基準：**

```ini
# /etc/sysctl.d/70-syn-flood.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 65535
net.core.somaxconn = 65535
net.ipv4.tcp_synack_retries = 3
```

`sysctl --system` で反映し、`ss -n state syn-recv` でSYN_RECVの滞留を監視する。値は本書 §6-2 の性能チューニング値と整合させる。

**nginx / Apache の受け口整合：**

- nginx: `listen 443 ssl backlog=65535;` でacceptキューをOS上限と整合
- Apache: `ListenBacklog 65535` を設定し、Event MPM運用を前提にする
- いずれも `keepalive_timeout` / `KeepAliveTimeout` を短くし、攻撃時の接続占有時間を抑制する

**重要：** SYN Floodの主戦場はL3/L4である。単体サーバー設定のみで完結させず、クラウドLB/CDN/WAF側の吸収機構を必ず併用する（§3各クラウド節参照）。

出典・参考情報: Linux Kernel Documentation (ip-sysctl)、nginx Core Module公式、Apache Core/MPM公式、AWS Shield Standard公式、Cloudflare DDoS Protection公式

### 2-7. パフォーマンスチューニング

**nginx：**

```nginx
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 8192;
    multi_accept on;
    use epoll;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 15;
    keepalive_requests 100;
    client_body_buffer_size 16k;
    client_max_body_size 16m;
    proxy_buffer_size 4k;
    proxy_buffers 8 16k;
}
```

最大同時接続数 = `worker_processes` × `worker_connections`。4コア × 8,192 = **32,768同時接続**。

**Apache（Event MPM推奨）：**

```apache
<IfModule mpm_event_module>
    StartServers            3
    MinSpareThreads         75
    MaxSpareThreads         250
    ThreadsPerChild         25
    MaxRequestWorkers       400
    MaxConnectionsPerChild  10000
</IfModule>
KeepAlive On
MaxKeepAliveRequests 500
KeepAliveTimeout 3
```

最大同時接続数 = `MaxRequestWorkers`（Apacheはこの値で直接制御）。Prefork MPMはプロセス単位のためメモリー消費が大きく、Event MPMへの移行を強く推奨する。

**【構成差分】** nginx の `worker_processes auto` はCPUコア数に自動設定される。Apache の `MaxRequestWorkers` はメモリー容量から逆算して設定する（1スレッドあたりの使用メモリー × ThreadsPerChild × StartServers が物理メモリーを超えないこと）。nginx の `keepalive_timeout` はクライアント接続の維持時間、upstream 向けには `keepalive` ディレクティブで別途制御する。Apache の `KeepAliveTimeout` はクライアント接続のみを対象とする。

出典・参考情報: nginx Core Module公式、Apache MPM公式

### 2-8. 圧縮（Gzip / Brotli）

Brotli は同等速度でGzipより15〜25%小さいファイルを生成する。動的コンテンツにはBrotliレベル4〜5、静的プリコンプレッションにはレベル11を使用し、Gzipをフォールバックとする。

**nginx：**

```nginx
# Gzip
gzip on;
gzip_vary on;
gzip_comp_level 6;
gzip_min_length 256;
gzip_static on;
gzip_types text/plain text/css text/javascript application/javascript
           application/json application/xml image/svg+xml font/woff2;

# Brotli (ngx_brotli module)
brotli on;
brotli_comp_level 4;
brotli_static on;
brotli_types text/plain text/css text/javascript application/javascript
             application/json application/xml image/svg+xml;
```

**Apache：**

```apache
# mod_brotli (Apache 2.4.26+)
<IfModule mod_brotli.c>
    AddOutputFilterByType BROTLI_COMPRESS text/html text/plain text/css
    AddOutputFilterByType BROTLI_COMPRESS text/javascript application/javascript
    AddOutputFilterByType BROTLI_COMPRESS application/json application/xml image/svg+xml
    BrotliCompressionQuality 4
</IfModule>

# mod_deflate (Gzip fallback)
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/css
    AddOutputFilterByType DEFLATE text/javascript application/javascript
    AddOutputFilterByType DEFLATE application/json application/xml image/svg+xml
    DeflateCompressionLevel 6
</IfModule>
```

既に圧縮済みのフォーマット（JPEG、PNG、WebP、AVIF、MP4、WOFF2、ZIP、GZ）は圧縮対象から除外する。

**【構成差分】** nginx の `gzip_static on` / `brotli_static on` はビルド時にプリコンプレッションしたファイル（`.gz` / `.br`）を直接配信する。Apache は mod_brotli が動的圧縮のみを提供し、プリコンプレッションには mod_rewrite でのルール設定が必要。nginx の ngx_brotli モジュールはサードパーティーモジュールのため、ディストリビューションパッケージに含まれない場合は手動ビルドが必要。

出典・参考情報: nginx ngx_http_gzip_module公式、Apache mod_brotli公式

### 2-9. ログ設定とローテーション

JSON構造化ログにより、Fluentd・Elasticsearch 等のログ集約ツールでの解析を容易にする。

**nginx：**

```nginx
log_format json_log escape=json '{'
    '"time":"$time_iso8601",'
    '"remote_addr":"$remote_addr",'
    '"request_method":"$request_method",'
    '"request_uri":"$request_uri",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"upstream_response_time":"$upstream_response_time",'
    '"http_user_agent":"$http_user_agent",'
    '"request_id":"$request_id"'
'}';

access_log /var/log/nginx/access.json json_log buffer=32k flush=5s;
error_log /var/log/nginx/error.log warn;
```

**Apache：**

```apache
LogFormat "{\"time\":\"%{%Y-%m-%dT%H:%M:%S%z}t\",\"remote_addr\":\"%a\",\"request_method\":\"%m\",\"request_uri\":\"%U%q\",\"status\":%>s,\"body_bytes_sent\":%B,\"request_time\":%D,\"http_user_agent\":\"%{User-Agent}i\"}" json_log
CustomLog /var/log/apache2/access.json json_log
ErrorLog /var/log/apache2/error.log
LogLevel warn
```

**logrotate設定**（`/etc/logrotate.d/nginx`）：

```
/var/log/nginx/*.log /var/log/nginx/*.json {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}
```

**【構成差分】** nginx は `USR1` シグナルでログファイルを再オープンする（ゼロダウンタイム）。Apache は `graceful` で再起動する。nginx では `copytruncate` ではなく `USR1` を使用する（高負荷時のログ欠損を防止）。Apache の `%D` はリクエスト処理時間をマイクロ秒単位で出力する（nginx の `$request_time` は秒単位、小数点以下3桁）。

出典・参考情報: nginx log_format公式、Apache mod_log_config公式

---

## 3. クラウド・VPSインフラストラクチャー

### 3-1. AWS

#### 3-1-1. EC2インスタンス選定

Graviton（ARM）プロセッサーはx86同等インスタンスに対して最大40%の価格性能比改善を提供する。Linuxワークロードでは **Graviton優先** を原則とし、ソフトウェアがARM非対応の場合のみx86を選択する。

| 種別 | ファミリー | 用途 |
|:---|:---|:---|
| バースタブル | T4g（ARM）/ T3（x86） | 開発・テスト、小規模Webサーバー、平均CPU使用率40%未満 |
| 汎用 | M7g / M8g（ARM）/ M7i（x86） | アプリケーションサーバー、中規模DB |
| コンピューティング最適化 | C7g / C8g（ARM）/ C7i（x86） | バッチ処理、動画エンコード、ML推論 |
| メモリー最適化 | R7g / R8g（ARM） | インメモリーDB、リアルタイム分析 |

Graviton4（M8g / C8g / R8g）が現行推奨世代（2026-03-22確認時点）：Graviton3比で30%性能向上、DDR5メモリー、最大50Gbpsネットワーク。Graviton5（M9g）はプレビュー段階（2026-03-22確認時点）。

#### 3-1-2. VPC設計

3層マルチAZ構成を標準とする。

```
VPC: 10.0.0.0/16 (65,536 IP)

Availability Zone A:
  Public Subnet:  10.0.1.0/24   — ALB, NAT Gateway, Bastion
  Private Subnet: 10.0.10.0/24  — Application servers
  Data Subnet:    10.0.20.0/24  — RDS, ElastiCache

Availability Zone B:
  Public Subnet:  10.0.2.0/24
  Private Subnet: 10.0.11.0/24
  Data Subnet:    10.0.21.0/24
```

AWSはサブネットごとに5つのIPを予約するため、/24で**251個**が使用可能。高可用性のためNAT Gatewayは各AZに1つ配置する。

#### 3-1-3. セキュリティグループとNACL

**セキュリティグループ**はステートフル・インスタンスレベル・許可ルールのみで、全ルールを集合的に評価する。**NACL**はステートレス・サブネットレベル・拒否ルールをサポートし、番号順に評価する。

設計方針：セキュリティグループを主要な制御機構とし、NACLは多層防御（既知の不正CIDRブロック等）として追加する。

#### 3-1-4. ロードバランサー

| 判断基準 | ALB（L7） | NLB（L4） |
|:---|:---|:---|
| プロトコル | HTTP / HTTPS | TCP / UDP / TLS |
| ルーティング | パスベース / ホストベース / ヘッダーベース | ポートベース |
| 静的IP | 非対応 | 対応 |
| WAF統合 | 対応 | 非対応 |
| WebSocket | 対応 | 対応 |
| レイテンシー | 標準 | 超低レイテンシー |
| PrivateLink | 非対応 | 対応 |

HTTP/HTTPSワークロードにはALBを選択する。静的IP・非HTTPプロトコル・極低レイテンシー・PrivateLinkが必要な場合はNLBを選択する。ヘルスチェック間隔は ALB: 15〜300秒、NLB: 10〜30秒。接続ドレインのデフォルトは300秒。

#### 3-1-5. Route 53

ホストゾーン作成後、レコードセットを設計する。Route 53固有の**エイリアスレコード**はAWSリソース（ALB / CloudFront / S3）をゾーンApexで直接指定可能（CNAMEの制約を回避）。ヘルスチェックとフェイルオーバールーティングを組み合わせることで、障害時に自動でスタンバイ環境へ切り替える。

#### 3-1-6. CloudFront

ディストリビューションの設定要素は、オリジン（S3 / ALB / カスタムオリジン）、ビヘイビア（パスパターン別のキャッシュ・転送設定）、エラーページ。

OAC（Origin Access Control）によりS3バケットへの直接アクセスを禁止し、CloudFront経由のみに制限する。OAI（Origin Access Identity）はレガシーであり、新規構成ではOACを使用する。

Lambda@Edge はビューアーリクエスト / オリジンリクエスト / オリジンレスポンス / ビューアーレスポンスの4トリガーポイントで実行される。CloudFront Functions は軽量処理（ヘッダー操作・URLリライト・リダイレクト）向けで、Lambda@Edgeより低レイテンシー・低コスト。

料金クラス（Price Class）はエッジロケーションの範囲を制限してコストを抑える：Price Class 100（北米・欧州のみ）が最安、Price Class All（全リージョン）が最高品質。

#### 3-1-7. IAMベストプラクティス

最小権限の原則を徹底する。ルートアカウントは初期設定後に使用せず、MFAを必須化する。IAMユーザーの代わりにIAMロール（EC2インスタンスプロファイル / ECSタスクロール）を使用する。アクセスキーの発行は最小限にし、定期的にローテーションする。AWS Organizations + SCP（Service Control Policy）でアカウント境界を設定する。

#### 3-1-8. ACM（AWS Certificate Manager）

パブリック証明書は無料で自動更新される。ALB / CloudFront / API Gateway と統合可能。CloudFrontで使用する証明書は **us-east-1 リージョン** に作成する必要がある（リージョン制約）。DNS検証を推奨（メール検証より自動化が容易）。

#### 3-1-9. SYN Flood対策（AWS）

AWSでは、SYN Floodは **Shield Standard（L3/L4）+ WAF（L7）+ ALB/NLB前段化** で対処する。EC2を直接公開せず、Route 53 / CloudFront / ALB-NLBをアプリケーション境界に固定する。

1. 境界保護
- Shield Standardの自動保護対象（Route 53、CloudFront、Global Accelerator、ALB/NLB経由構成）を優先する
- オリジンEC2はprivate subnet配置を原則とし、Security Groupで直接到達を禁止する

2. L7抑制
- AWS WAFのRate-based ruleで突発的な高頻度IPを遮断/Challengeする
- `/login` `/api/auth` など高負荷エンドポイントはしきい値を個別管理する

3. 影響局所化
- Auto Scalingでバックエンドを水平拡張し、単一AZ飽和を回避する
- VPC Flow LogsでTCP SYN増加（`tcp-flags=2`）を可視化し、NACLで既知悪性CIDRを短期遮断する

出典・参考情報: AWS EC2公式ドキュメント、AWS VPC公式ドキュメント、AWS ELB公式ドキュメント、AWS CloudFront公式ドキュメント、AWS IAMベストプラクティス、AWS ACM公式ドキュメント、AWS Shield Standard公式、AWS WAF Rate-based Rules公式、AWS VPC Flow Logs公式、AWS Auto Scaling公式

### 3-2. Google Cloud

#### 3-2-1. Compute Engineインスタンス選定

| ファミリー | シリーズ | 用途 |
|:---|:---|:---|
| 汎用（コスト効率） | E2 | 開発・テスト、小規模Web |
| 汎用（バランス） | N2 / N2D（AMD） | 標準ワークロード |
| コンピューティング最適化 | C3 / C4 | ゲームサーバー、HPC |
| メモリー最適化 | M3 | 大規模インメモリーDB |
| ARM | T2A（Ampere Altra） | ARM対応ワークロード（コスト優位） |

T2A（ARM）はx86 E2と比較してコスト効率が高い。ARM対応のワークロードでは優先的に検討する。

#### 3-2-2. VPCネットワーク設計

Google CloudのVPCは**グローバルリソース**であり、サブネットはリージョナル。AWSと異なり、VPC自体がリージョンをまたぐ。

共有VPC（Shared VPC）はホストプロジェクトでネットワークを一元管理し、サービスプロジェクトがサブネットを使用する構成。マルチプロジェクト環境で推奨。

VPC ファイアウォールルールはインスタンスの**ネットワークタグ**または**サービスアカウント**に基づいて適用される。

#### 3-2-3. ファイアウォールルール

Google Cloudのファイアウォールルールはグローバルに定義され、**優先度**（0〜65535、低い数値が高優先度）で評価順序を制御する。

設計方針：優先度1000番台を標準ルール、100番台を緊急ブロック、65534を暗黙の拒否として予約する。ネットワークタグで論理グループ（`web-server`、`app-server`、`db-server`）を定義し、タグ単位でルールを適用する。

#### 3-2-4. Cloud Load Balancing

Google Cloudのロードバランサーは**グローバル**（外部HTTP(S)、外部TCP/UDPプロキシー）と**リージョナル**（内部TCP/UDP、内部HTTP(S)）に分類される。

外部HTTP(S)ロードバランサーはGoogle Front End（GFE）で処理され、自動的にDDoS防御（Cloud Armor Basic）を提供する。サーバーレスNEG（Network Endpoint Group）により、Cloud Run / App Engine / Cloud Functionsをバックエンドに直接指定可能。

#### 3-2-5. Cloud DNS

マネージドDNSゾーンの作成後、レコードセットを設定する。DNSSEC有効化はコンソールまたはCLI（`gcloud dns managed-zones update --dnssec-state on`）で実行する。

ルーティングポリシー：加重ラウンドロビン（WRR）、地理位置情報（GEO）、フェイルオーバーを設定可能。

#### 3-2-6. Cloud CDN

| キャッシュモード | 動作 |
|:---|:---|
| CACHE_ALL_STATIC | 静的コンテンツ（画像・CSS・JS等）を自動キャッシュ |
| USE_ORIGIN_HEADERS | Cache-Control ヘッダーに従う |
| FORCE_CACHE_ALL | 全レスポンスを強制キャッシュ（TTL指定必須） |

キャッシュキーのカスタマイズ（ヘッダー / Cookie / クエリパラメーター）により、同一URLでも異なるレスポンスをキャッシュ可能。署名付きURL / 署名付きCookieで認証済みユーザーのみにコンテンツを配信する。

Cloud Armorとの統合により、CDNエッジでWAFルールを適用可能。

#### 3-2-7. IAMとサービスアカウント

Workload Identity Federation により、外部IDプロバイダー（GitHub Actions / AWS IAM等）からサービスアカウントキーなしでGoogle Cloudリソースにアクセス可能。サービスアカウントキーの発行は最小限にし、Workload Identity を優先する。

カスタムロールで最小権限を実装する。プリミティブロール（Owner / Editor / Viewer）は本番環境で使用しない。

#### 3-2-8. Certificate Manager

マネージド証明書はDNS認証で自動発行・自動更新される。ロードバランサーへの証明書マップ（Certificate Map）経由で関連付ける。ワイルドカード証明書はDNS認証のみ対応。

#### 3-2-9. SYN Flood対策（Google Cloud）

Google Cloudでは、外部公開を **Cloud Load Balancing + Cloud Armor** に集約し、SYN/Volumetric攻撃をエッジ側で吸収する。バックエンドVMはpublic IPを持たせない構成を標準とする。

1. 前段防御
- 外部HTTP(S) LB + Cloud Armorセキュリティポリシーを適用する
- ルール優先度を設計し、国・IP・パス単位で攻撃トラフィックを段階遮断する

2. 直接到達遮断
- VPCファイアウォールで、バックエンドへの到達元をLB/ヘルスチェック経路のみに限定する
- 緊急時は高優先度denyルールを投入できるよう、優先度帯を事前予約する

3. 監視/運用
- Cloud Monitoring + VPC Flow Logsで新規接続急増を検知し、Cloud Armorルールへ即時反映する
- 大規模L3/L4リスクが高い公開構成ではCloud Armor上位機能（Enterprise/Advanced保護）を評価する

出典・参考情報: Google Cloud Compute Engine公式、Google Cloud VPC公式、Google Cloud Load Balancing公式、Google Cloud CDN公式、Google Cloud IAM公式、Google Cloud Certificate Manager公式、Google Cloud Armor公式、VPC Flow Logs公式、Cloud Monitoring公式

### 3-3. Azure

#### 3-3-1. VM選定

| シリーズ | 用途 | 備考 |
|:---|:---|:---|
| Bv2（バースタブル） | 開発・テスト、軽量Web | B-series v1は2028年11月廃止予定 |
| Dv5 / Dasv5 | 汎用ワークロード | AMD EPYC |
| Dpsv5（ARM） | ARM対応ワークロード | Azure Cobaltプロセッサー |
| Ev5 / Easv5 | メモリー最適化 | 大規模DB、キャッシュ |
| Fsv2 / Falsv6 | コンピューティング最適化 | バッチ処理、CI/CD |

B-series v1（Bs、Bms）は2028年11月廃止。Bv2への移行を計画する。Dpsv5（Cobalt ARM）はLinuxワークロードでコスト効率が高い。

#### 3-3-2. VNet設計

Azure VNetはリージョナルリソース。サブネット、NSG（Network Security Group）、サービスエンドポイント / Private Endpointで構成する。

サービスエンドポイントはAzureバックボーン経由でPaaS（Storage / SQL Database等）にアクセスする。Private EndpointはVNet内にプライベートIPを割り当て、PaaSリソースをVNet内部に公開する。新規構成ではPrivate Endpointを推奨する。

#### 3-3-3. NSGとASG

NSG（Network Security Group）はサブネットまたはNIC単位で適用するステートフルファイアウォール。ルールは優先度（100〜4096、低い数値が高優先度）で評価される。

ASG（Application Security Group）はVM群を論理グループ化し、NSGルールのソース / デスティネーションとして使用する。IPアドレスの直接指定を回避し、アプリケーション構成に基づくルール設計を可能にする。

#### 3-3-4. Application Gateway

Application Gateway v2はL7ロードバランサーで、WAFポリシー統合、パスベースルーティング、自動スケーリング、セッションアフィニティーを提供する。

WAF_v2 SKUでAzure WAFが統合される。Application GatewayとFront Doorの選定基準：単一リージョンの場合はApplication Gateway、グローバル分散の場合はFront Doorを選択する。

#### 3-3-5. Azure DNS

ゾーン管理はAzureポータルまたはCLI（`az network dns zone create`）で実行する。**エイリアスレコード**はAzureリソース（Public IP / Traffic Manager / Front Door）をゾーンApexで指定可能。

Private DNS Zoneにより、VNet内部の名前解決をカスタマイズする。VNetリンクで対象VNetを関連付ける。

#### 3-3-6. Azure CDN / Front Door

Azure Front Door（推奨）はグローバルL7ロードバランサー + CDN + WAFを統合する。Azure CDN Standardはシンプルなキャッシュ配信向け。

Front Doorのルールエンジンは条件（リクエストヘッダー / URL / クエリ文字列 / 地理情報）に基づくURL書き換え、リダイレクト、ヘッダー操作を提供する。プライベートリンクオリジンにより、バックエンドをパブリックインターネットに公開せずにFront Door経由のみでアクセス可能にする。

キャッシュパージは単一URL、ワイルドカード、全パージの3種で実行可能。

#### 3-3-7. Azure IAMとマネージドID

Azure RBAC（Role-Based Access Control）で最小権限を実装する。マネージドID（システム割り当て / ユーザー割り当て）により、資格情報なしでAzureリソースにアクセスする。Entra ID（旧Azure AD）との統合で、条件付きアクセスポリシーを適用する。

#### 3-3-8. Key Vault

証明書管理（自動更新対応）、シークレット管理（接続文字列・APIキー）、暗号化キー管理を一元化する。アクセスポリシーまたはRBACでアクセス制御を実装する。Application Gateway / Front DoorとKey Vault証明書の連携で、SSL証明書のデプロイを自動化する。

#### 3-3-9. SYN Flood対策（Azure）

Azureでは、SYN Flood対策を **Azure DDoS Protection（L3/L4）+ WAF（L7）+ VNet分離** で実装する。公開IPを持つリソースを最小化し、Front Door / Application Gateway経由を強制する。

1. L3/L4防御
- 公開エンドポイントを含むVNetにAzure DDoS Protectionを適用する
- DDoS保護メトリクス/アラートをAzure Monitorに統合する

2. L7防御
- Front Door WAFまたはApplication Gateway WAF_v2でHTTP Floodを遮断する
- レート制御・Bot対策・Geo/IP制御を組み合わせる

3. バックエンド保護
- NSG/ASGで到達元をLB/WAF経路に限定し、直接アクセスを禁止する
- VMSSオートスケールで攻撃時の可用性低下を抑制する

出典・参考情報: Azure VM公式ドキュメント、Azure VNet公式ドキュメント、Azure Application Gateway公式ドキュメント、Azure Front Door公式ドキュメント、Azure DNS公式ドキュメント、Azure Key Vault公式ドキュメント、Azure DDoS Protection公式、Azure WAF公式、Azure Monitor公式、Azure NSG公式

### 3-4. 国内VPS（さくらVPS / ConoHa VPS）

#### 3-4-1. プラン選定基準

| 項目 | さくらVPS | ConoHa VPS |
|:---|:---|:---|
| 最小プラン | 512MB / ¥671〜/月 | 512MB / ¥751〜/月 |
| 課金体系 | 月額固定（3か月最低契約） | 時間課金（月額上限あり） |
| スケール変更 | プラン変更（要再起動） | リアルタイムスケールアップ/ダウン |
| リージョン | 石狩（最安）/ 大阪 / 東京 | 東京 / 大阪 |
| 無料期間 | 2週間お試し | なし |
| 長期割引 | なし | まとめトク（最大69%割引） |

短期・変動需要にはConoHa VPS（時間課金）、安定稼働にはさくらVPS（月額固定）を選択する。価格・契約条件は改定されるため、表の数値は2026-03-22確認時点として扱う。

#### 3-4-2. 初期セットアップ手順

```bash
# OS更新・タイムゾーン設定
apt update && apt upgrade -y
timedatectl set-timezone Asia/Tokyo

# 非rootユーザー作成
adduser deploy && usermod -aG sudo deploy

# SSH鍵配置
su - deploy
mkdir -p ~/.ssh && chmod 700 ~/.ssh
# Paste public key to ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# sshd強化 (§4-1参照)
# Firewall設定 (§4-2参照)
```

#### 3-4-3. ファイアウォール設定

さくらVPSはコントロールパネルの「パケットフィルター」機能を提供する（ポート単位のIN/OUTフィルター）。ConoHa VPSはコントロールパネルの「セキュリティグループ」で同等の制御が可能。

いずれもOS内部のファイアウォール（ufw / nftables）と併用する。コントロールパネルのフィルターは第一層、OS内部のファイアウォールは第二層として多層防御を構成する。

#### 3-4-4. ネットワーク設定

グローバルIPは各プランに1つ付与される。追加IPはオプション。ローカルネットワーク（VPS間プライベート通信）はさくらVPS・ConoHa VPSともに提供する。IPv6はさくらVPSが対応、ConoHa VPSも対応。

#### 3-4-5. 【構成差分】クラウドとVPSの責任境界

| 項目 | クラウド（AWS / GCP / Azure） | VPS |
|:---|:---|:---|
| OS管理 | AMI / イメージ選択、パッチは自己責任 | 自己責任（apt upgrade） |
| ファイアウォール | マネージド（SG / FWルール / NSG） | 自前（ufw / nftables） |
| ロードバランサー | マネージド（ALB / Cloud LB / App GW） | 自前（nginx / HAProxy） |
| SSL証明書 | マネージド（ACM / Certificate Manager / Key Vault） | 自前（Certbot） |
| バックアップ | マネージド（スナップショット / PITR） | 自前（スクリプト + cron） |
| 監視 | マネージド（CloudWatch / Cloud Monitoring / Monitor） | 自前（Prometheus / Netdata） |
| スケーリング | オートスケーリング対応 | 手動プラン変更 |

VPSではクラウドのマネージドサービスに相当する機能を全て自前で構築・運用する必要がある。

#### 3-4-6. SYN Flood対策（国内VPS）

VPSではL3/L4の吸収を自前で担うため、**上位フィルター（提供事業者）+ OSファイアウォール + カーネルチューニング** を同時適用する。Webサーバー設定のみでの防御を禁止する。

1. 事業者側フィルター
- さくらVPSパケットフィルター / ConoHaセキュリティグループで80/443以外を閉塞する
- 攻撃時はCIDR単位で一時遮断し、OS側ルール反映より先に入口を絞る

2. OS側レート制限（例: iptables）

```bash
iptables -A INPUT -p tcp --syn --dport 80 \
  -m hashlimit --hashlimit-name syn80 \
  --hashlimit-above 50/second --hashlimit-burst 100 \
  --hashlimit-mode srcip --hashlimit-srcmask 32 -j DROP

iptables -A INPUT -p tcp --syn --dport 443 \
  -m hashlimit --hashlimit-name syn443 \
  --hashlimit-above 50/second --hashlimit-burst 100 \
  --hashlimit-mode srcip --hashlimit-srcmask 32 -j DROP

ip6tables -A INPUT -p tcp --syn --dport 80 \
  -m hashlimit --hashlimit-name syn80v6 \
  --hashlimit-above 50/second --hashlimit-burst 100 \
  --hashlimit-mode srcip --hashlimit-srcmask 64 -j DROP

ip6tables -A INPUT -p tcp --syn --dport 443 \
  -m hashlimit --hashlimit-name syn443v6 \
  --hashlimit-above 50/second --hashlimit-burst 100 \
  --hashlimit-mode srcip --hashlimit-srcmask 64 -j DROP
```

3. カーネル
- `tcp_syncookies`、`tcp_max_syn_backlog`、`somaxconn` を §2-6-1 / §6-2 と整合させる
- 監視でSYN_RECV急増を検知したら、閾値超過IPを即時ブロックリストへ反映する

出典・参考情報: さくらVPS仕様一覧、ConoHa VPS料金・スペック、Linux Kernel Documentation (ip-sysctl)、netfilter/iptables公式

### 3-5. Cloudflare

#### 3-5-1. プラン選定と機能比較

| 機能 | Free | Pro | Business | Enterprise |
|:---|:---|:---|:---|:---|
| 月額 | $0 | $20 | $200 | 要見積 |
| WAFマネージドルール | 基本 | 拡張 | 拡張 | 全機能 |
| カスタムWAFルール | 5 | 20 | 100 | 無制限 |
| Rate Limiting | 1ルール | 10ルール | 25ルール | 無制限 |
| Bot Management | Bot Fight Mode | Super Bot Fight Mode | Super Bot Fight Mode | Bot Management |
| Image Optimization | なし | Polish | Polish | Polish + 高度な最適化 |
| Argo Smart Routing | 有料アドオン | 有料アドオン | 有料アドオン | 含む |
| カスタム証明書 | なし | 対応 | 対応 | 対応 |
| アップロード上限 | 100MB | 100MB | 200MB | 500MB |
| サポート | コミュニティー | メール | メール + チャット | 専任 + 電話 |

上記プラン料金・ルール数・アップロード上限は2026-03-22確認時点。

#### 3-5-2. DNS設定

Cloudflare DNSへの移行はドメインレジストラーでネームサーバーをCloudflare指定のNSに変更して実行する。

**プロキシモード（オレンジ雲）vs DNSオンリー（グレー雲）：** プロキシモードではCloudflareのCDN・WAF・DDoS防御が有効になり、オリジンIPが秘匿される。DNSオンリーはDNS解決のみで、トラフィックはCloudflareを経由しない。

CNAME Flatteningにより、Cloudflare管理ゾーンではゾーンApex相当でもCNAMEライクな運用が可能。これはCloudflare固有仕様であり、汎用DNSサーバーでは同一挙動を前提にしない。DNSSEC有効化はダッシュボードのDNS設定から実行可能。

#### 3-5-3. SSL/TLS設定

**暗号化モード選定：**

| モード | オリジンへの接続 | 証明書検証 | 推奨度 |
|:---|:---|:---|:---|
| Off | HTTP | なし | 禁止 |
| Flexible | HTTP | なし | 非推奨（オリジン〜Cloudflare間が平文） |
| Full | HTTPS | 検証なし（自己署名可） | 最低限 |
| Full (Strict) | HTTPS | 有効な証明書を検証 | **必須** |

**Full (Strict)** を必須とする。オリジン証明書はCloudflare Origin CA（最大15年有効）を使用するか、Let's Encryptで取得する。

Authenticated Origin Pulls（mTLS）により、Cloudflareからのリクエストのみをオリジンが受け付ける。nginx設定例：

```nginx
ssl_client_certificate /etc/ssl/cloudflare/origin-pull-ca.pem;
ssl_verify_client on;
```

最小TLSバージョンは1.2に設定する。Automatic HTTPS Rewritesと Always Use HTTPS を有効化する。

#### 3-5-4. キャッシュ設定

**キャッシュレベル：** No Query String（クエリ文字列を無視）/ Ignore Query String（クエリ文字列を除外してキャッシュ）/ Standard（クエリ文字列ごとに個別キャッシュ、推奨）。

**Cache Rules** でパス・ホスト・クエリ文字列等の条件に基づくキャッシュ制御を設定する。Page Rulesのキャッシュ設定はレガシーであり、Cache Rulesへの移行を推奨する。

キャッシュパージは全パージ、URL指定、タグ指定（Enterprise）、プレフィックス指定（Enterprise）の4種。

**Tiered Cache** はオリジンへのリクエストを上位エッジ経由に集約し、オリジン負荷を削減する。**Cache Reserve**（Enterprise）は永続エッジキャッシュとして、エビクション（キャッシュ追い出し）を防止する。

**Always Online** はオリジンダウン時にCloudflareのキャッシュ（またはInternet Archiveのスナップショット）からステールコンテンツを配信する。

**【構成差分】キャッシュ制御モデルの違い：** Cloudflareはデフォルトで静的ファイルをキャッシュし、HTMLはキャッシュしない。CloudFrontはビヘイビア設定でキャッシュポリシーを明示的に割り当てる。Cloud CDNはキャッシュモード（CACHE_ALL_STATIC / USE_ORIGIN_HEADERS / FORCE_CACHE_ALL）で動作を選択する。Front Doorはルールエンジンでキャッシュ動作をカスタマイズする。

#### 3-5-5. セキュリティ機能

**WAF：** §4-6-4 でルール設計とチューニング方針を詳述する。本節では設定手順を記載する。

マネージドルールセットの有効化はダッシュボードのセキュリティ → WAFから実行する。Cloudflare Managed RulesetとOWASP Core Rulesetの2つを有効化する。各ルールのアクションはDefault / Block / Managed Challenge / JS Challenge / Logから選択する。

**DDoS防御：** L3/L4はCloudflareが自動的に緩和する（全プランで有効）。L7はHTTP DDoS Attack Protectionで自動検知・緩和される。感度とアクションはルール単位でカスタマイズ可能。

**Rate Limiting Rules：** リクエストカウント（期間内のリクエスト数）に基づく制限。カウント方式はIP / IPとURLの組み合わせ / カスタム式から選択する。レスポンスアクションはBlock / Challenge / JS Challenge / Logから設定する。

**Bot Fight Mode**（Free）/ **Super Bot Fight Mode**（Pro以上）はボットトラフィックを自動検知する。Enterprise の Bot Management は機械学習ベースの高度なボット分類を提供する。

**IP Access Rules** でIP / IPレンジ / 国 / ASNに基づくアクセス制御を設定する。

セキュリティレベル設定（Off / Essentially Off / Low / Medium / High / I'm Under Attack!）は、Cloudflareの脅威スコアに基づくChallengeの閾値を制御する。DDoS攻撃時は「I'm Under Attack!」モードを有効化する。

#### 3-5-6. パフォーマンス機能

Auto Minify は一部旧機能が非推奨化されているため、利用中の場合はCloudflare公式の移行手順で状態確認し、必要に応じて無効化する。代替として以下を使用する。

**Cloudflare Fonts：** Google Fontsをエッジから配信し、サードパーティーリクエストを削減。**Early Hints：** `103 Early Hints` レスポンスでブラウザーにリソースのプリロードを指示。**Rocket Loader：** JavaScriptの非同期読み込みを自動化（互換性問題が報告される場合あり、テスト必須）。

**HTTP/2・HTTP/3（QUIC）：** ダッシュボードから有効化。全プランで対応。

**Argo Smart Routing**（有料アドオン）はCloudflareの専用ネットワーク経由で最適経路を選択し、レイテンシーを削減する。

**Polish：** 画像の自動最適化。ロスレス / ロッシー圧縮を選択可能。WebP / AVIF自動変換に対応。

**Tiered Cache：** Free プランでも有効化可能。上位エッジ（スマートティアリング）でオリジンリクエストを集約する。

#### 3-5-7. Page Rules / Configuration Rules / Transform Rules

**Page Rules**（レガシー）はルール数が制限されており（Free: 3、Pro: 20、Business: 50、2026-03-22確認時点）、新しいルールエンジンへの移行を推奨する。

**Configuration Rules：** SSL / キャッシュ / セキュリティレベル等をリクエスト条件に基づいて個別適用する。Page Rulesの後継。

**Transform Rules：** URLリライト（パス変更 / クエリ文字列変更）とHTTPリクエスト / レスポンスヘッダーの変更を実行する。

**Redirect Rules：** バルクリダイレクト（大量のURL単位リダイレクト）を効率的に管理する。

#### 3-5-8. Workers / Pages（概要）

Cloudflare Workers はエッジでJavaScript / TypeScript / Wasm を実行するサーバーレスプラットフォーム。A/Bテスト、認証チェック、リダイレクト、ヘッダー操作等をオリジンへのリクエスト前に処理可能。

Workers Routes で対象パスパターンを指定し、適用範囲を制御する。本ドキュメントではルーティング・キャッシュ制御の設定方法までを対象とし、アプリケーションロジックの実装詳細は対象外とする。

#### 3-5-9. 【構成差分】Cloudflare固有の注意事項

**クライアントIP取得：** Cloudflareプロキシモードではクライアントの実IPが`CF-Connecting-IP` ヘッダーに格納される。nginx で取得する場合：

```nginx
# Cloudflare公開IPレンジを外部ファイルで管理
include /etc/nginx/cloudflare-realip.conf;
real_ip_header CF-Connecting-IP;
real_ip_recursive on;
```

**オリジンIP秘匿：** オリジンサーバーのファイアウォールでCloudflare IPレンジからの接続のみを許可する。Cloudflare IPリスト（`https://www.cloudflare.com/ips-v4` / `ips-v6`）は定期更新し、設定ファイル生成を自動化する。

**アップロードサイズ制限：** Free / Pro: 100MB、Business: 200MB、Enterprise: 500MB（2026-03-22確認時点）。この制限を超えるファイルアップロードはCloudflareプロキシをバイパスする別ドメイン（グレー雲）で処理する。

**WebSocket：** 全プランで対応。ダッシュボードから有効化する。

#### 3-5-10. SYN Flood対策（Cloudflare前段）

Cloudflareプロキシ（オレンジ雲）を前段化すると、L3/L4の大半はCloudflare側で吸収される。SYN Flood対策の主目的は **オリジン直叩き防止** と **L7過負荷の抑制** である。

1. エッジ吸収
- HTTP/S公開は必ずCloudflareプロキシ経由に統一する
- HTTP DDoS ProtectionとWAF Managed Rulesを同時有効化する

2. 追加制御
- Rate Limiting Rulesで `/login` `/api/*` などを個別制御する
- 攻撃時はSecurity Level引き上げや "I'm Under Attack!" を短期適用する

3. オリジン保護
- オリジンFWでCloudflare公開IPレンジのみ許可する
- Authenticated Origin Pulls（mTLS）を有効化し、Cloudflare経由以外を拒否する
- 直叩き防止のため、オリジン専用ホスト名を外部公開しない

出典・参考情報: Cloudflare Developer Docs、Cloudflare SSL/TLS公式、Cloudflare Cache公式、Cloudflare WAF公式、Cloudflare Workers公式、Cloudflare DDoS Protection公式、Cloudflare Rate Limiting公式、Cloudflare IP Ranges公式

### 3-6. DNS設定（プロバイダー共通）

#### 3-6-1. レコード種別と用途

| レコード | 用途 | 例 |
|:---|:---|:---|
| A / AAAA | ドメイン → IPv4 / IPv6アドレス | `example.com. 3600 IN A 93.184.216.34` |
| CNAME | エイリアス（ゾーンApex不可） | `www IN CNAME example.com.` |
| MX | メールルーティング（数値が小さいほど高優先） | `@ IN MX 10 mail.example.com.` |
| TXT / SPF | メール送信元認可 | `"v=spf1 mx include:_spf.google.com ~all"` |
| TXT / DMARC | メール認証ポリシー | `_dmarc IN TXT "v=DMARC1; p=quarantine; rua=mailto:..."` |
| TXT / DKIM | メール署名検証 | `selector._domainkey IN TXT "v=DKIM1; k=rsa; p=..."` |
| CAA | 証明書発行認可局の制限 | `@ IN CAA 0 issue "letsencrypt.org"` |
| SRV | サービスディスカバリー | `_sip._tcp IN SRV 10 60 5060 sip.example.com.` |

#### 3-6-2. TTL設計方針

通常時は安定レコードに **3600〜14400秒**（1〜4時間）を設定する。移行・変更予定の24〜48時間前にTTLを **300〜600秒** に引き下げ、完了後に元に戻す。

#### 3-6-3. DNSSEC有効化

全ての公開ドメインでDNSSECの有効化を推奨する。Route 53、Cloudflare、Google Cloud DNSはいずれもマネージドDNSSECを提供する。Azure DNSはDNSSEC署名に対応（2024年GA）。

#### 3-6-4. メール認証レコード（SPF / DKIM / DMARC）

メール配信を行うドメインでは3種のレコードを全て設定する。SPFは送信元IPの認可、DKIMはメール署名の検証、DMARCは認証失敗時のポリシー（none / quarantine / reject）を定義する。

出典・参考情報: RFC 1035、RFC 7208 (SPF)、RFC 6376 (DKIM)、RFC 7489 (DMARC)、RFC 8659 (CAA)

### 3-7. 【プロバイダー間構成差分まとめ】

#### ネットワーク分離モデル

| 項目 | AWS | Google Cloud | Azure | OCI | Cloudflare |
|:---|:---|:---|:---|:---|:---|
| ネットワーク範囲 | リージョナルVPC | グローバルVPC | リージョナルVNet | リージョナルVCN | グローバルエッジ |
| FWルール適用 | SG（インスタンス単位） | タグ / SA（グローバル） | NSG + ASG（サブネット / NIC） | NSG + Security List（VNIC / サブネット） | WAF（エッジ） |
| 拒否ルール | NACLのみ | 優先度ベース | NSG優先度ベース | 明示denyなし（許可ルール + 暗黙deny） | WAFカスタムルール |

#### ロードバランサー設計思想

AWS ALBはリージョナル、Google Cloud外部HTTP(S) LBはグローバル、Azure Application Gatewayはリージョナル（Front Doorでグローバル化）、OCI Load Balancerはリージョナル。グローバルLBが必要な場合はGoogle Cloud またはCloudflare / Front Doorを選択し、OCIはリージョン内配信を前提に設計する。

#### 証明書管理

| プロバイダー | サービス | 自動更新 | ワイルドカード | 料金 |
|:---|:---|:---|:---|:---|
| AWS | ACM | 対応 | 対応 | 無料 |
| Google Cloud | Certificate Manager | 対応 | DNS認証で対応 | 無料 |
| Azure | Key Vault + App GW連携 | 対応 | 対応 | Key Vault料金 |
| OCI | Certificates + Load Balancer連携 | 対応（運用方式に依存） | 対応 | サービス従量 |
| Cloudflare | Universal / Advanced | 対応 | Advanced以上 | Free〜 |
| 自前 | Let's Encrypt + Certbot | Certbot自動 | DNS認証で対応 | 無料 |

#### CDN / エッジ機能比較

| 機能 | CloudFront | Cloud CDN | Front Door | Cloudflare |
|:---|:---|:---|:---|:---|
| WAF統合 | AWS WAF | Cloud Armor | Azure WAF | Cloudflare WAF |
| DDoS防御 | Shield Standard（無料） | Cloud Armor Basic | DDoS Protection Basic | 全プラン自動 |
| エッジコンピューティング | Lambda@Edge / CF Functions | なし（Cloud Run NEG） | Rules Engine | Workers |
| HTTP/3 | 対応 | 対応 | 対応 | 対応 |
| 料金モデル | 転送量 + リクエスト数 | 転送量 + リクエスト数 | 転送量 + リクエスト数 | プラン定額 + 超過分 |

OCIはリージョナル配信を基本とし、グローバル配信はCloudflareなどのCDN前段化を組み合わせる。

### 3-8. Oracle Cloud Infrastructure（OCI）

#### 3-8-1. 標準アーキテクチャ（設定）

OCIでは、リージョン / Availability Domain（AD）/ Fault Domain（FD）を前提に可用性を設計する。AD数はリージョンごとに異なるため、**複数ADがあるリージョンではAD分散、単一ADリージョンではFD分散**を必須とする。

ネットワークはVCNを境界として分離し、公開面はWAF + Load Balancerのみとする。Computeはprivate subnetに配置し、運用アクセスはBastion経由に限定する。

| レイヤー | OCIサービス | 構成要件 |
|:---|:---|:---|
| Edge | WAF + Public Load Balancer | 直接Compute公開を禁止、TLS終端はLBまたはWAFで統制 |
| App | Compute / Instance Pool | private subnet配置、NSGで通信を最小化 |
| Data | Object Storage / DB / Block Volume | public endpoint無効、鍵管理はVaultで統一 |
| Ops | Bastion | 管理アクセスは時間制限セッション + 接続元CIDR制限 |
| Hybrid | DRG + FastConnect / IPSec VPN | オンプレ接続は専用線優先、冗長経路を設計 |

構成例（CIDR設計）:

```text
VCN: 10.40.0.0/16
  public-edge-subnet   10.40.1.0/24   (WAF/LB)
  private-app-subnet-a 10.40.10.0/24  (App: AD/FD分散)
  private-app-subnet-b 10.40.11.0/24  (App: AD/FD分散)
  private-data-subnet  10.40.20.0/24  (DB/Storage access)
  private-ops-subnet   10.40.30.0/24  (Bastion)
```

#### 3-8-2. ネットワーク制御（NSG / Security List）

OCIの仮想ファイアウォールはSecurity List（サブネット単位）とNSG（VNIC単位）を併用できる。**本番ではNSGを主制御、Security Listは最小共通ルールのみ**とし、許可範囲の過大化を防ぐ。

- Security List: サブネット全体に効くため、広すぎる許可を作りやすい
- NSG: ワークロード単位で絞り込めるため、マイクロセグメンテーション向き
- OSファイアウォール（nftables/iptables/Windows Firewall）も同時に成立させる

#### 3-8-3. セキュリティ設計（IAM / Cloud Guard / Security Zones / Vault / WAF）

OCIのセキュリティは「権限最小化 + 設定逸脱の継続検出 + 鍵管理統制」を同時に実施する。単一機能への依存を禁止する。

1) IAM / Compartment分離  
テナンシ直下にリソースを直置きせず、`network` / `security` / `app` / `data` / `ops` のコンパートメントを分離する。  
ポリシーはグループ単位で定義し、Administrators常用を禁止する。

```policy
Allow group net-admins to manage virtual-network-family in compartment prod-network
Allow group sec-admins to manage waf-family in compartment prod-security
Allow group app-ops to manage instance-family in compartment prod-app
Allow dynamic-group app-instances to use keys in compartment prod-security
Allow group obs-admins to read metrics in compartment prod-app
```

2) Security Zones  
高機密コンパートメントはSecurity Zoneに関連付け、ポリシー違反操作を作成時点で拒否する。

3) Cloud Guard  
Detector Recipe / Responder Recipeを有効化し、設定逸脱・危険操作を継続監視する。重大イベントは自動是正または即時通知のどちらかを事前に定義する。

4) Vault（KMS）  
暗号化はVaultの顧客管理鍵（CMK）を標準化し、鍵ローテーション運用を必須とする。

5) WAF（日本以外遮断の実装例）  
Access Rulesで`Country/Region is not JP`を`BLOCK`に設定し、日本国外を遮断する。API指定時は2文字国コード（`JP`）を使う。  
また、運用者アクセスは`IP Address in Address List`で許可管理し、国条件と重ねて誤許可を防止する。  
注記: WAFではIP whitelistがAccess Rulesより先に評価されるため、運用許可IPは必ず事前登録する。

#### 3-8-4. 高速化（性能チューニング）

OCIの高速化は、Compute・LB/NLB・Block Volume・接続方式・監視の5層で同時に調整する。単体最適化のみを禁止する。

1) Compute  
ワークロードに合わせてShapeを選定し、継続的にCPU/メモリ/ネットワーク使用率で見直す。  
負荷変動がある場合はInstance Pool + Autoscalingを使用し、**Metric-based** と **Schedule-based** を使い分ける。

2) Load Balancer / Network Load Balancer  

| 要件 | 選択 | 設定観点 |
|:---|:---|:---|
| HTTP/HTTPS、L7制御、TLS終端 | Load Balancer | backend set、health check、session persistence、cipher管理 |
| TCP/UDP中心、低遅延、長時間接続 | Network Load Balancer | L3/L4パススルー、接続維持、backend健全性監視 |

3) Block Volume  
ボリューム性能レベル（Lower Cost / Balanced / Higher Performance / Ultra High Performance）をI/O特性に合わせる。既定値固定を禁止し、遅延・IOPS実測で段階調整する。

4) ネットワーク経路  
オンプレ連携で遅延・帯域が支配要因の場合、FastConnectを採用する。private peering / public peering を用途で分離し、単一回線運用を禁止する（冗長経路前提）。

5) 可観測性（Observability）  
MonitoringでメトリクスとAlarm閾値を定義し、LoggingでAudit / Service / Custom logsを集約する。  
VCN Flow LogsとLBログを常時有効化し、性能劣化の原因（ネットワーク/アプリ/バックエンド）を切り分け可能にする。

#### 3-8-5. 導入時の最低チェック（OCI）

| 項目 | 合格条件 |
|:---|:---|
| 公開面統制 | 公開エンドポイントはWAF/LBのみ（Compute直接公開なし） |
| 権限分離 | Compartment分離 + グループ単位ポリシー適用 |
| 管理アクセス | Bastionの時間制限セッション + CIDR制限 |
| 地理制限 | WAFで`Country/Region is not JP` BLOCK実装 |
| 鍵管理 | Vault CMK運用 + ローテーション手順定義 |
| 監視 | Monitoring Alarm + Logging集約 + Flow Logs有効 |
| 性能 | Autoscaling・LBヘルスチェック・Volume性能レベル調整済み |

#### 3-8-6. SYN Flood対策（OCI）

OCIではL3/L4 DDoS緩和が基盤側で提供されるため、公開面はLoad Balancer/WAFへ集約し、オリジンをprivate subnetに閉じる設計を徹底する。

1. L3/L4
- 公開接点をLB/WAFへ固定し、Compute直公開を禁止する
- VCN/NSGで到達経路を最小化する

2. L7
- OCI WAFでAccess Rules（IP/Country/Path）とレート制御を適用する
- HTTP急増時のアラームをMonitoringで定義し、段階的にBlockルールへ切替する

3. オペレーション
- Bastion経由運用を維持し、攻撃時の緊急遮断手順（Address List更新、WAF rule優先度変更）を事前定義する

出典・参考情報: OCI Compute公式、OCI VCN公式、OCI NSG公式、OCI IAM Policies公式、OCI Bastion公式、OCI WAF公式、OCI Layer 7 DDoS Mitigation公式、OCI FastConnect公式、OCI Autoscaling公式、OCI Block Volume Performance公式、OCI Monitoring公式、OCI Logging公式


---

## 4. セキュリティ

> 汎用的なセキュリティ要件は [secure-code-requirements.md](/Users/TED/Documents/ClaudeCode/AI-instructions/secure-code-requirements.md) を参照。本セクションはサーバーインフラストラクチャー固有のセキュリティ対策を規定する。

### 4-1. SSH強化

#### SSHキーアルゴリズム選定

**Ed25519を必須とする。** ECDSA・DSAは新規生成を禁止し、RSAはレガシー互換時の例外（RSA-4096）を除いて新規生成を禁止する。

| アルゴリズム | ステータス | 鍵長 | セキュリティ等価強度 | 問題点 |
|:---|:---|:---|:---|:---|
| **Ed25519（EdDSA）** | **必須** | 256bit固定 | 128bit（RSA-3072相当） | なし |
| RSA | **非推奨（レガシー互換の例外のみ）** | 4096bit（例外） | 約140bit | 鍵サイズ肥大、低速、量子耐性なし |
| ECDSA（NIST曲線） | **禁止** | 256 / 384 / 521bit | 128 / 192 / 256bit | NIST曲線の信頼性問題、実装脆弱性 |
| DSA | **禁止** | 1024bit固定 | 80bit | OpenSSH 7.0で既定無効、セキュリティ不十分 |

#### Ed25519が必須である理由

Ed25519はDaniel J. Bernsteinが設計したCurve25519上のEdDSA署名方式であり、以下の特性によりSSH運用標準として必須とする。

**セキュリティ上の優位性：**

- **決定論的ノンス生成：** ECDSAが署名時にランダムノンスを要求するのに対し、Ed25519は秘密鍵とメッセージのハッシュからノンスを決定論的に生成する。これにより、乱数生成器の品質不良による秘密鍵漏洩のリスクを排除する（PlayStation 3のECDSA鍵漏洩事件はノンス再利用が原因）
- **サイドチャネル攻撃耐性：** 定時間演算（constant-time operation）で設計されており、タイミング攻撃・キャッシュ攻撃に対する耐性が高い
- **曲線の透明性：** Curve25519は設計パラメーターの選定根拠が完全に公開されており、「nothing-up-my-sleeve」の原則を満たす。NIST曲線のような不透明なシード値の問題がない

**性能上の優位性：**

- 256bit固定鍵でRSA-3072と同等のセキュリティ強度を提供する
- 署名生成・検証の速度がRSA・ECDSAより高速
- 鍵サイズが小さく（公開鍵32バイト）、ストレージ・転送オーバーヘッドが最小

**互換性：** OpenSSH 6.5（2014年）以降で対応。2026年現在、実運用で非対応環境に遭遇することはまれ。

出典・参考情報: RFC 8032、OpenSSH公式ドキュメント、SafeCurves (Bernstein & Lange)

#### RSAが非推奨である理由

RSA（Rivest-Shamir-Adleman）は1977年に設計されたアルゴリズムであり、以下の問題を抱える。

- **鍵サイズの肥大化：** Ed25519と同等のセキュリティ（128bit）を得るにはRSA-3072が必要。将来的な安全性（128bit超）にはRSA-4096以上が必要となり、鍵サイズと演算コストが非線形に増大する
- **低速な演算：** 署名生成・検証ともにEd25519より大幅に遅い。特にリソース制約のある環境（IoT・組み込み）で問題となる
- **量子コンピューティング脆弱性：** Shorのアルゴリズムにより、十分な規模の量子コンピューターでRSAの素因数分解問題が多項式時間で解ける。Ed25519も楕円曲線離散対数問題への量子攻撃の対象だが、RSAは鍵長に対する量子攻撃の効率がより高い
- **実装の複雑さ：** パディング方式（PKCS#1 v1.5 / OAEP / PSS）の選択ミスによる脆弱性が歴史的に多数報告されている

RSA-4096はEd25519非対応のレガシーシステムとの接続にのみ使用を許可する。その場合も `~/.ssh/config` でホスト別にEd25519とRSAを使い分ける。

#### ECDSAが禁止である理由

本ガイドラインでは、鍵アルゴリズムをEd25519へ統一する運用方針によりECDSAを禁止する。主な理由は次のとおり。

- ECDSA署名は高品質なnonce管理に依存し、運用不備時に秘密鍵漏洩リスクを生じる
- アルゴリズム混在はサーバー/クライアント双方の設定面を増やし、監査と障害切り分けを複雑化する
- OpenSSH運用ではEd25519を標準化することで鍵配布・検証手順を単純化できる

出典・参考情報: NIST FIPS 186-5、RFC 8032、OpenSSH公式ドキュメント

#### DSAが禁止である理由


- OpenSSH 7.0（2015年）でDSA鍵のサポートがデフォルトで無効化された
- 鍵長が1024bitに制限されており（FIPS 186-2）、セキュリティ強度が80bitと現代の基準を満たさない
- 2023年以降、主要Linuxディストリビューションの多くがDSA鍵の生成・認証を完全に無効化している

#### Ed25519鍵の生成

```bash
ssh-keygen -t ed25519 -C "user@hostname"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server_ip
```

レガシーシステム向けRSAフォールバック鍵（必要な場合のみ）：

```bash
ssh-keygen -t rsa -b 4096 -C "legacy-fallback@hostname" -f ~/.ssh/id_rsa_legacy
```

`~/.ssh/config` でホスト別に使い分ける：

```
# 標準（Ed25519）
Host modern-server
    HostName server.example.com
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

# レガシー互換（RSA-4096、やむを得ない場合のみ）
Host legacy-server
    HostName old.example.com
    IdentityFile ~/.ssh/id_rsa_legacy
    IdentitiesOnly yes
```

#### sshd_config必須設定

```
Port 2222
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
MaxAuthTries 3
LoginGraceTime 30
MaxSessions 3
AllowUsers deployer admin
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
AllowTcpForwarding no

# ホスト鍵をEd25519のみに限定
HostKey /etc/ssh/ssh_host_ed25519_key

# 暗号アルゴリズム（運用ポリシー、2026-03-22確認時点）
KexAlgorithms sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
HostKeyAlgorithms ssh-ed25519
PubkeyAcceptedAlgorithms ssh-ed25519
```

`HostKey` をEd25519のみに限定し、`HostKeyAlgorithms` と `PubkeyAcceptedAlgorithms` でEd25519以外の鍵タイプを拒否する。レガシークライアントからの接続が必要な場合は `PubkeyAcceptedAlgorithms` に `rsa-sha2-512,rsa-sha2-256` を追加する（`ssh-rsa`（SHA-1ベース）はOpenSSH 8.8で既定無効化されており、追加しない）。

設定変更後の検証：`sshd -t` 実行後、`systemctl reload sshd || systemctl reload ssh` で反映する。

#### fail2ban設定

```ini
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2222
maxretry = 3
bantime = 86400
findtime = 600
```

出典・参考情報: OpenSSH公式ドキュメント、RFC 8032、NIST FIPS 186-5、SafeCurves (Bernstein & Lange)

### 4-2. ファイアウォール

**nftables**（iptablesの後継、新規環境で推奨）：

```
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        ct state invalid drop
        iif lo accept
        icmp type echo-request limit rate 5/second accept
        icmpv6 type { nd-neighbor-solicit, nd-neighbor-advert, nd-router-solicit, nd-router-advert } accept
        tcp dport 2222 ct state new limit rate 3/minute accept
        tcp dport { 80, 443 } accept
        counter log prefix "nft-drop: " drop
    }
    chain forward { type filter hook forward priority 0; policy drop; }
    chain output { type filter hook output priority 0; policy accept; }
}
```

**ufw**（シンプルな設定向け）：

```bash
ufw default deny incoming && ufw default allow outgoing
ufw allow 2222/tcp && ufw allow 80/tcp && ufw allow 443/tcp
ufw limit 2222/tcp
ufw enable
```

**【構成差分】** VPSではOS内部のファイアウォール（nftables / ufw）を自前で管理する。クラウドではマネージドファイアウォール（AWS SG / GCPファイアウォールルール / Azure NSG）が主要な制御機構となり、OS内部のファイアウォールは第二層として併用する。nftablesはiptablesより構造化されたルール記述が可能で、パフォーマンスも改善されている。

出典・参考情報: nftables wiki、ufw公式ドキュメント

### 4-3. OS強化

**自動セキュリティ更新**（`/etc/apt/apt.conf.d/50unattended-upgrades`）：

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "false";
```

**カーネルハードニング**（`/etc/sysctl.d/99-hardening.conf`）：

```ini
net.ipv4.ip_forward = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
kernel.randomize_va_space = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
fs.suid_dumpable = 0
```

不要なサービスの確認と無効化：`systemctl list-unit-files --state=enabled` で有効なサービスを確認し、不要なもの（`cups`、`avahi-daemon` 等）を無効化する。

出典・参考情報: CIS Benchmarks、Ubuntu Security Guide

### 4-4. ファイル権限

```bash
# Webファイル: deployer所有、www-dataが読み取り
chown -R deployer:www-data /var/www/html
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;

# 機密ファイル
chmod 640 /var/www/html/.env
chmod 600 /etc/ssl/private/*.key
```

**【構成差分】** コンテナ環境ではDockerfile内で `USER` ディレクティブにより非rootユーザーで実行し、`--read-only` フラグでファイルシステムを読み取り専用にする（§5-3参照）。ホストのファイル権限とコンテナ内の uid/gid マッピングに注意する。

### 4-5. SSL証明書自動化

**Let's Encrypt + Certbot：**

```bash
# snap経由でインストール（推奨）
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot

# nginx用証明書取得
certbot --nginx -d example.com -d www.example.com

# ワイルドカード証明書（DNS認証、Cloudflareプラグイン例）
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /etc/letsencrypt/cloudflare.ini \
  -d example.com -d "*.example.com"

# 自動更新テスト
certbot renew --dry-run
```

証明書の有効期間は90日。60日時点で自動更新される（systemdタイマーが1日2回実行）。レート制限：登録ドメインあたり週50証明書。

**【構成差分】** クラウドマネージド証明書（ACM / Certificate Manager / Key Vault）はロードバランサーと直接統合され、自動更新が完全マネージド。Certbotはオリジンサーバーに直接証明書を配置する場合に使用する。Cloudflare利用時はエッジ証明書（Universal / Advanced）がCloudflare側で自動管理され、オリジンにはCloudflare Origin CA証明書を配置する。

出典・参考情報: Let's Encrypt公式、Certbot公式ドキュメント

### 4-6. WAF（Web Application Firewall）

#### 4-6-1. WAF選定基準

| 判断基準 | ModSecurity（自前） | AWS WAF | Cloudflare WAF | Azure WAF | OCI WAF |
|:---|:---|:---|:---|:---|:---|
| 運用負荷 | 高（ルール管理・更新自前） | 中（マネージドルール + カスタム） | 低（ダッシュボード設定） | 中（ポリシー管理） | 中（ポリシー/ルール管理） |
| カスタマイズ性 | 最高（SecRule構文） | 高（条件・ラベル） | 中（Wirefilter構文） | 中（マッチ条件） | 中（Access/Protection Rules） |
| コスト | サーバーリソースのみ | Web ACL + ルール数 + リクエスト数 | プラン定額内 | App GW / Front Door料金 | 従量課金 |
| レイテンシー | オリジンで処理 | エッジ（CloudFront）/ リージョン（ALB） | エッジ | リージョン / グローバル | エッジ / リージョン |
| マネージドルール品質 | OWASP CRS（コミュニティー） | AWS + サードパーティー | Cloudflare独自 + OWASP | Microsoft DRS + OWASP | Oracle管理ルール + カスタム |

多層防御として、CDN/エッジWAF（Cloudflare / AWS WAF / OCI WAF）とオリジンWAF（ModSecurity）を併用するパターンを推奨する。

#### 4-6-2. ModSecurity v3 + OWASP CRS

ModSecurity v3は libmodsecurity ライブラリーとWebサーバーコネクターで構成される。

**段階的導入手順：**

1. DetectionOnly モードで導入し、ログを収集する
2. 偽陽性を分析し、除外ルールを作成する
3. Blocking モード（`SecRuleEngine On`）に切り替える

```
# /etc/nginx/modsec/main.conf
Include /etc/nginx/modsec/modsecurity.conf
Include /etc/nginx/modsec/coreruleset-4/crs-setup.conf
Include /etc/nginx/modsec/coreruleset-4/rules/*.conf
```

**Paranoia Level（PL）選定：** PL1（既定、低偽陽性）→ PL2（追加ルール、偽陽性増加）→ PL3 / PL4（最大検出、偽陽性も最大）。本番環境ではPL1から開始し、偽陽性を管理しながら段階的に引き上げる。

**ルール除外（偽陽性対応）：**

```
# 特定ルールIDの除外
SecRuleRemoveById 920350

# 特定パスでのルール除外
SecRule REQUEST_URI "@beginsWith /api/upload" \
    "id:1001,phase:1,pass,nolog,\
     ctl:ruleRemoveById=920350"
```

**カスタムルール作成：**

```
SecRule REQUEST_HEADERS:User-Agent "@rx (bot|crawler|spider)" \
    "id:2001,phase:1,deny,status:403,\
     log,msg:'Blocked known bot pattern'"
```

**【構成差分】** nginx ではModSecurity v3コネクター（`ngx_http_modsecurity_module`）をビルドして使用する。Apache では `mod_security2` を `a2enmod` で有効化する。nginx版はリクエストボディのバッファリング方式が異なるため、大容量アップロード時のパフォーマンス影響をテストする。

出典・参考情報: ModSecurity公式、OWASP Core Rule Set公式

#### 4-6-3. AWS WAF

**Web ACL設計：** ルールグループは優先度順に評価される（数値が小さいほど先に評価）。Web ACLあたりのWCU（Web ACL Capacity Units）固定上限は5,000。1,500を超える利用は追加課金対象。

**推奨マネージドルールグループ：**

| ルールグループ | 目的 |
|:---|:---|
| AWSManagedRulesCommonRuleSet | OWASP Top 10対策 |
| AWSManagedRulesSQLiRuleSet | SQLインジェクション |
| AWSManagedRulesKnownBadInputsRuleSet | Log4j / SSRF等の既知攻撃 |
| AWSManagedRulesAmazonIpReputationList | 悪質IP |
| AWSManagedRulesAnonymousIpList | VPN / Tor / プロキシー |
| AWSManagedRulesBotControlRuleSet | ボット検出（Common / Targeted） |

**ルールアクション：** Allow / Block / Count（ログのみ、テスト用）/ CAPTCHA / Challenge。新しいルール追加時は必ず Count モードで検証してから Block に切り替える。

**Rate-based ルール：** IP単位、カスタムキー（ヘッダー値等）で5分間のリクエスト数に基づく制限。最小閾値は100リクエスト/5分。

**ログ設定：** S3 / CloudWatch Logs / Kinesis Firehose の3つの送信先。ログ分析にはAthena（S3の場合）またはCloudWatch Logs Insights を使用する。

**【構成差分】** ALB関連付けではクライアントIPが `X-Forwarded-For` ヘッダーから取得される。CloudFront関連付けではCloudFrontが自動的にクライアントIPを処理する。CloudFrontでは地理的制限がCloudFront機能として提供されるが、ALBではAWS WAFの地理一致条件で実装する。

出典・参考情報: AWS WAF公式ドキュメント、AWS WAFマネージドルール一覧

#### 4-6-4. Cloudflare WAF

**マネージドルールセット：** Cloudflare Managed Ruleset（Cloudflare独自ルール）とOWASP Core Ruleset の2つを有効化する。各ルールのアクションと感度レベル（Sensitivity）をダッシュボードから調整する。

**カスタムルール**（Wirefilter構文）：

地理ブロック例：

```
http.request.uri.path starts_with "/admin" and not ip.geoip.country in {"JP"}
→ Action: Block
```

URIパターン検査例：

```
(http.request.uri.path contains "/.env" or http.request.uri.path contains "/wp-login.php")
→ Action: Block
```

**Rate Limiting Rules：** リクエストカウント条件（例：同一IPから60秒間に100リクエスト超）に対してBlock / Challenge / JS Challenge を適用する。

**Firewall Rules → WAF Custom Rules 移行：** 旧Firewall Rules機能は2024年に廃止された。既存ルールはWAF Custom Rulesに移行する。

**Bot検出プラン別差分：**

| 機能 | Free | Pro | Business | Enterprise |
|:---|:---|:---|:---|:---|
| Bot Fight Mode | 基本 | — | — | — |
| Super Bot Fight Mode | — | 対応 | 対応 | — |
| Bot Management（ML） | — | — | — | 対応 |
| JavaScript検出 | なし | 対応 | 対応 | 対応 |
| API保護 | なし | なし | なし | 対応 |

出典・参考情報: Cloudflare WAF公式ドキュメント、Cloudflare Rate Limiting公式

#### 4-6-5. Azure WAF

**WAFポリシー：** Application Gateway WAF（v2 SKU）とFront Door WAFの2種。ポリシーモードはDetection（ログのみ）とPrevention（ブロック）。

**マネージドルールセット：** DRS 2.1（Default Rule Set）/ OWASP 3.2 / Bot Managerルール。各ルールの有効/無効・アクション変更が可能。

**カスタムルール：** マッチ条件（IPアドレス / 地理 / リクエストヘッダー / URI / ボディサイズ）と優先度（1〜100）で構成。マネージドルールより先にカスタムルールが評価される。

**除外設定：** リクエストヘッダー / Cookie / クエリパラメーター / ボディの特定フィールドをルール評価から除外する（偽陽性対応）。

**ログ分析：** Application Gateway診断ログまたはFront Doorアクセスログを Log Analytics ワークスペースに送信し、KQL（Kusto Query Language）で分析する。

**【構成差分】** Application Gateway WAFはリージョナルでVNet内部のアプリケーションを保護する。Front Door WAFはグローバルエッジで動作する。Front Door WAFはレート制限機能が限定的（カスタムルールで実装）。

出典・参考情報: Azure WAF公式ドキュメント、Azure Front Door WAF公式

#### 4-6-6. WAF運用共通規定

**段階的導入フロー：**

1. 監視モード（DetectionOnly / Count / Detection）で全ルールを有効化する
2. 1〜2週間のログを収集し、偽陽性パターンを特定する
3. 偽陽性対応（ルール除外 / 条件付きバイパス / 感度調整）を実施する
4. ブロックモード（On / Block / Prevention）に切り替える
5. 切り替え後も継続的にログを監視し、新たな偽陽性に対応する

**偽陽性対応手順：**

1. ログから該当ルールIDとリクエスト内容を特定する
2. 正当なリクエストかどうかを判定する
3. ルール除外（特定パス / メソッド限定）、条件付きバイパス、ルール感度調整のいずれかで対応する
4. 変更適用後の再検証を実施する

**偽陰性検出：** 定期的なペネトレーションテスト、OWASP ZAP / Nuclei によるWAFルール検証を実施する。

**ルール更新管理：** マネージドルールの自動更新ポリシーを確認する。AWS WAFはマネージドルールのバージョニング（静的バージョン / デフォルトバージョン）で制御する。Cloudflare / Azureはプロバイダーが自動更新する。更新による偽陽性増加をCountモードで事前検証する。

**インシデント対応フロー：**

1. WAFアラート検知（異常なブロック数増加 / 特定ルールの頻繁なトリガー）
2. 攻撃パターンの分析（ログ / イベント）
3. 緊急対応（IPブロック / Rate Limiting強化 / 「I'm Under Attack!」モード有効化）
4. 根本対応（アプリケーション脆弱性修正）
5. 事後レビュー（ルール改善 / 対応手順更新）

**WAFバイパス対策：** オリジンIPの秘匿は必須。Cloudflare利用時はAuthenticated Origin Pulls（§3-5-3）を有効化する。AWS利用時はALBのセキュリティグループでCloudFrontのマネージドプレフィックスリスト（`com.amazonaws.global.cloudfront.origin-facing`）からの接続のみを許可する。

**パフォーマンス影響の監視：** WAFルール評価によるレイテンシー増加をモニタリングする。ModSecurityはオリジンCPUに影響するため、負荷テスト時にWAF有効状態で検証する。

**WAFテスト手法：** OWASP ZAP（DAST）/ Nuclei でWAFルールの有効性を検証する。テストは必ずステージング環境で実施する。

#### 4-6-7. 【WAFプラットフォーム横断比較テーブル】

| 比較軸 | ModSecurity + CRS | AWS WAF | Cloudflare WAF | Azure WAF | OCI WAF |
|:---|:---|:---|:---|:---|:---|
| マネージドルール | OWASP CRS（コミュニティー） | AWS + Marketplace | Cloudflare独自 + OWASP | Microsoft DRS + OWASP | Oracle管理ルール |
| カスタムルール構文 | SecRule（正規表現ベース） | JSON条件式 | Wirefilter | マッチ条件 | 条件式（Access/Protection Rules） |
| Rate Limiting | 別途実装（nginx limit_req等） | Rate-based ルール | Rate Limiting Rules | カスタムルール | WAFルールで実装 |
| Bot検出 | なし（別途導入） | Bot Control（有料） | Bot Fight Mode〜Bot Management | Bot Managerルール | カスタム実装 |
| ログ分析 | auditlog + 自前解析 | Athena / CloudWatch Insights | Security Events | Log Analytics + KQL | Logging + Monitoring |
| 導入難易度 | 高（ビルド・設定） | 中（コンソール / IaC） | 低（ダッシュボード） | 中（ポリシー設定） | 中（コンソール / API） |
| コストモデル | サーバーリソース | 従量課金 | プラン内定額 | App GW / Front Door従量 | 従量課金 |
| レイテンシー影響 | オリジン処理（数ms） | エッジ / リージョン（1ms未満） | エッジ（1ms未満） | リージョン / グローバル | エッジ / リージョン |

出典・参考情報: OWASP ModSecurity CRS公式、AWS WAF Managed Rules、Cloudflare WAF Managed Rules、Azure WAF Managed Rules、OCI WAF公式

#### 4-6-8. 地理ベースアクセス制御（日本国内のみ許可の例）

管理画面や社内向けAPIでは、要件に応じて国別制限を適用する。実装はオリジンではなくエッジ/WAF層を優先する。

**nginx（GeoIP2、JP以外を403）：**

`ngx_http_geoip2_module` を有効化し、MaxMind DBを読み込む。`X-Forwarded-For` 配下では先に `real_ip_header` / `set_real_ip_from` を設定し、GeoIP判定の `source` には復元済みのクライアントIP（`$remote_addr`）を使う。

```nginx
geoip2 /usr/share/GeoIP/GeoLite2-Country.mmdb {
    auto_reload 5m;
    $geoip2_country_code source=$remote_addr country iso_code;
}

map $geoip2_country_code $is_allowed_country {
    default 0;
    JP 1;
}

server {
    listen 443 ssl;
    server_name app.example.com;

    location /admin/ {
        if ($is_allowed_country = 0) { return 403; }
        proxy_pass http://backend_app;
    }
}
```

**Apache（mod_maxminddb、JPのみ許可）：**

```apache
MaxMindDBEnable On
MaxMindDBFile COUNTRY_DB /usr/share/GeoIP/GeoLite2-Country.mmdb
MaxMindDBEnv MM_COUNTRY_CODE COUNTRY_DB/country/iso_code

<Location "/admin">
    Require expr %{ENV:MM_COUNTRY_CODE} == "JP"
</Location>
```

**Cloudflare WAF Custom Rules（JP以外を遮断）：**

```
not ip.geoip.country in {"JP"}
→ Action: Block
```

管理画面のみ制限する場合：

```
http.request.uri.path starts_with "/admin" and not ip.geoip.country in {"JP"}
→ Action: Block
```

**AWS WAF（GeoMatchでJP以外を明示Block）：**

1. 優先度の高いルールで `NotStatement(GeoMatchStatement: ["JP"])` を `BLOCK`
2. Web ACL の `Default action` は `ALLOW` にする
3. マネージドルールはそのまま併用し、JPトラフィックに対しても検査を継続する

`GeoMatchStatement: ["JP"]` を先頭 `ALLOW` にする構成は、後続ルール評価を短絡させるため採用しない。

**OCI WAF Access Rules（JP以外を遮断）：**

```
Country/Region is not JP
→ Action: Block
```

APIで指定する場合は2文字国コード（`JP`）を使用する。管理画面のみ制限する場合は、URL条件（例: `/admin`）とCountry条件を組み合わせる。

CloudFront配下ではエッジで判定される。ALB直付けでも同様にGeoMatchを適用可能。

非JPを明示Blockするルール例（Web ACL JSONイメージ）：

```json
{
  "Name": "block-non-jp",
  "Priority": 10,
  "Statement": {
    "NotStatement": {
      "Statement": {
        "GeoMatchStatement": { "CountryCodes": ["JP"] }
      }
    }
  },
  "Action": { "Block": {} }
}
```

**AWS CloudFront（静的配信の簡易制限）：**

CloudFrontの `Geo restriction` を `Whitelist`、`JP` を指定すると、配信自体を日本向けに限定できる。

**Azure WAF（Application Gateway / Front Door）：**

カスタムルールで `RemoteAddr` の地理一致を `Japan` に設定し、それ以外を `Block` にする。まず `Detection` で誤検知を確認してから `Prevention` に切り替える。

**運用上の注意：**

- GeoIP判定は完全ではない。VPN・モバイル回線・海外ローミングで誤判定が発生する
- 初期導入は `Challenge` / `Count` / `Detection` でログを確認してから `Block` に移行する
- 障害対応用に緊急バイパス手段（固定許可IP、緊急トークン、一時的なルール無効化手順）を事前定義する

### 4-7. 侵入検知

**AIDE**（Advanced Intrusion Detection Environment）はファイル整合性監視ツール。

```bash
apt install aide aide-common
aideinit
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check
```

日次cronで `aide --check` を実行し、変更を検知する。Wazuh（OSSEC後継）はログ分析・侵入検知・ファイル整合性監視を統合的に提供するが、導入・運用の複雑さが増す。小規模環境ではAIDE、大規模環境ではWazuhを検討する。

### 4-8. 脆弱性スキャンとパッチ管理

定期スキャン方針：月次でOS脆弱性スキャン（`apt list --upgradable` + CVE照合）を実施する。Webアプリケーション脆弱性スキャン（OWASP ZAP / Nuclei）は四半期に1回以上。

CVE監視：NVD（National Vulnerability Database）/ JVN（Japan Vulnerability Notes）の通知を購読する。使用ソフトウェアのセキュリティアドバイザリーメーリングリストに登録する。

パッチ適用フロー：CVE公開 → 影響評価 → ステージング環境でパッチ適用・検証 → 本番環境適用。Critical / Highの脆弱性は72時間以内に対応する。

出典・参考情報: AIDE公式、Wazuh公式、NVD、JVN

---

## 5. コンテナ技術

### 5-1. Docker

#### 5-1-1. Dockerfile設計原則

マルチステージビルドでビルド環境と実行環境を分離する。レイヤーキャッシュを最大活用するため、依存マニフェストを先にコピーしてインストールし、ソースコードを後からコピーする。

#### 5-1-2. ベースイメージ選定

| ベースイメージ | サイズ | 用途 |
|:---|:---|:---|
| Alpine | 約5MB | 軽量、多くの言語ランタイムに対応 |
| distroless（Google） | 約2MB | 最小、シェルなし、デバッグ困難 |
| Debian slim | 約80MB | 互換性重視、glibc |
| Ubuntu | 約75MB | 開発環境との一致 |

タグは具体的なバージョンを指定する（`node:22.11-alpine3.20` のように少なくとも `major.minor`、可能な限り `major.minor.patch` を明示）。`:latest` は禁止。

#### 5-1-3. 言語別Dockerfile例

**Node.js：**

```dockerfile
FROM node:22.11-alpine3.20@sha256:<actual_digest> AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22.11-alpine3.20@sha256:<actual_digest> AS production
WORKDIR /app
RUN addgroup -g 1001 appgroup && adduser -u 1001 -G appgroup -D appuser
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
RUN npm ci --omit=dev && npm cache clean --force
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

**Go（distroless）：**

```dockerfile
FROM golang:1.23.4-alpine3.20@sha256:<actual_digest> AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server

FROM gcr.io/distroless/static-debian12:nonroot@sha256:<actual_digest>
# 本番運用ではダイジェスト固定を必須とする: gcr.io/distroless/static-debian12:nonroot@sha256:<actual_digest>
COPY --from=builder /app/server /server
USER nonroot:nonroot
ENTRYPOINT ["/server"]
```

#### 5-1-4. ビルドキャッシュ戦略

BuildKitのキャッシュエクスポート（`--cache-to type=gha`）でCI/CDパイプラインのビルド時間を短縮する。`.dockerignore` で `.git` / `node_modules` / テストファイル等を除外し、ビルドコンテキストを最小化する。

出典・参考情報: Docker Dockerfile Best Practices公式、Google distroless

### 5-2. Docker Compose

#### 5-2-1. サービス定義

Docker Compose v2（`docker compose`）は Compose Specification に準拠し、`version:` キーは廃止。Composeはv2系を標準とする。

```yaml
name: myapp
services:
  web:
    build: { context: ., target: production }
    ports: ["8080:3000"]
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://appuser:${DB_PASSWORD}@db:5432/myapp
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_healthy }
    restart: unless-stopped
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }
      replicas: 2
    networks: [frontend, backend]

  db:
    image: postgres:17.4-alpine3.20@sha256:<actual_digest>
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 10s
    networks: [backend]

  redis:
    image: redis:7.4.2-alpine3.20@sha256:<actual_digest>
    command: >
      redis-server --appendonly yes
      --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
    networks: [backend]

volumes: { pgdata: {} }
networks:
  frontend: { driver: bridge }
  backend: { driver: bridge, internal: true }
```

#### 5-2-2. ネットワーク設計

`internal: true` フラグでバックエンドネットワークの外部アクセスを遮断する。フロントエンド（公開）とバックエンド（内部）の2ネットワーク構成を標準とする。

#### 5-2-3. ボリュームと永続化

名前付きボリューム（`volumes: { pgdata: {} }`）を使用し、バインドマウント（ホストパス直接指定）は開発環境のみに限定する。本番環境でのバインドマウントはポータビリティーとセキュリティの観点から避ける。

#### 5-2-4. 環境変数管理

`.env` ファイルで環境変数を管理し、`.gitignore` に登録する。機密情報はDocker Secrets（Swarmモード）または外部シークレット管理（AWS Secrets Manager / HashiCorp Vault）で管理する。

#### 5-2-5. リソース制限

`deploy.resources.limits` でCPU・メモリーの上限を設定する。`pids_limit` でPID数を制限し、フォーク爆弾を防止する。

#### 5-2-6. 【構成差分】開発環境 vs 本番環境

`docker-compose.override.yml`（開発用自動読み込み）と `docker-compose.prod.yml`（本番用明示指定）で設定を分離する。

| 項目 | 開発環境 | 本番環境 |
|:---|:---|:---|
| ビルドターゲット | `development`（devDependencies含む） | `production` |
| ボリューム | バインドマウント（ホットリロード用） | 名前付きボリューム |
| ポート公開 | デバッグポート含む | 最小限 |
| ログ | stdout（verbose） | JSON構造化ログ |
| リソース制限 | なし | 必須 |
| replicas | 1 | 2以上 |

`docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d` で本番設定を適用する。Compose profiles（`profiles: [debug]`）で開発専用サービスを条件付きで起動する方式も有効。

注記：`deploy` セクションはCompose Specification上のオプションであり、実行環境の実装に依存する。`replicas` が反映されない環境では `docker compose up --scale web=2 -d` のように `--scale` または `services.<name>.scale` を使用して明示する。

出典・参考情報: Docker Compose公式ドキュメント、Compose Specification

### 5-3. コンテナセキュリティ

#### 5-3-1. 非rootユーザー実行

Dockerfile内で専用ユーザーを作成し、`USER` ディレクティブで切り替える。`docker run` 時の `--user` フラグでも指定可能。

#### 5-3-2. read-onlyファイルシステム / tmpfs

```bash
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --memory=512m --cpus=1.0 --pids-limit=100 \
  --cap-drop ALL --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --user 10001:10001 \
  myregistry.io/myapp:1.2.3@sha256:<actual_digest>
```

#### 5-3-3. capability制御

`--cap-drop ALL` で全capabilityを削除し、必要なもののみ `--cap-add` で追加する。Webサーバーが80/443をバインドする場合は `NET_BIND_SERVICE` が必要（1024未満のポート使用時。1024以上のポートを使用してリバースプロキシで転送する方式ではcap不要）。

#### 5-3-4. イメージスキャン

CI/CDパイプラインでイメージスキャンを実行する。`trivy image --severity HIGH,CRITICAL myregistry.io/myapp:1.2.3@sha256:<actual_digest>`。HIGH / CRITICALの脆弱性が検出された場合はビルドを失敗させる。

#### 5-3-5. イメージ署名

Cosign（Sigstoreプロジェクト）でイメージ署名を行い、サプライチェーンの改ざん検知と真正性検証を可能にする。

出典・参考情報: Docker Security Best Practices公式、Trivy公式、Sigstore公式

### 5-4. Kubernetes

#### 5-4-1. コアリソース

Kubernetes **v1.35** は2026-03-22確認時点の最新安定版。N-2サポートポリシーにより、通常は最新3マイナー（例: 1.35/1.34/1.33）がサポート対象。リリースサイクルは約15週間。

主要リソース：Pod（最小デプロイ単位）、Service（ネットワーク抽象化）、Deployment（宣言的Pod管理）、Ingress（L7ルーティング）、ConfigMap（設定データ）、Secret（機密データ）。

#### 5-4-2. Deployment戦略

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: myapp
          image: myregistry.io/myapp:1.2.3@sha256:<actual_digest>
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          resources:
            requests: { cpu: 200m, memory: 256Mi }
            limits: { cpu: "1", memory: 512Mi }
          livenessProbe:
            httpGet: { path: /healthz, port: 8080 }
            periodSeconds: 15
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            periodSeconds: 10
```

`maxSurge: 1, maxUnavailable: 0` により、常にフル容量を維持しながらローリングアップデートを実行する。

#### 5-4-3. リソース設定

| ワークロード | CPU Request | CPU Limit | Memory Request | Memory Limit |
|:---|:---|:---|:---|:---|
| 軽量API | 100m | 500m | 128Mi | 256Mi |
| 標準Webアプリケーション | 250m | 1000m | 256Mi | 512Mi |
| 高負荷処理 | 500m | 2000m | 512Mi | 1Gi |

メモリーLimit は Request と同値に設定する（OOMKill回避）。CPU Limitは議論があり、Requestのみ設定してLimitを省略するアプローチも有効。

#### 5-4-4. Probe設計

**livenessProbe：** アプリケーションがデッドロック状態に陥った場合にPodを再起動する。`/healthz` エンドポイントで基本的な応答能力を確認する。

**readinessProbe：** トラフィックを受け入れる準備ができているかを確認する。`/ready` エンドポイントで依存サービス（DB / キャッシュ）への接続状態を確認する。

**startupProbe：** 起動が遅いアプリケーション向け。startupProbeが成功するまでliveness / readinessProbeは実行されない。

#### 5-4-5. 【構成差分】マネージドK8s間のIngress・LB連携

| 項目 | EKS（AWS） | GKE（Google Cloud） | AKS（Azure） |
|:---|:---|:---|:---|
| Ingress Controller | AWS Load Balancer Controller | GKE Ingress（既定） | AGIC / nginx-ingress |
| 外部LB | ALB（Ingressアノテーション経由） | Google Cloud LB（自動） | Application Gateway / Azure LB |
| 内部LB | NLB（内部向けアノテーション） | 内部HTTP(S) LB | 内部Azure LB |
| SSL終端 | ALB + ACM | Google-managed cert | App GW + Key Vault |

出典・参考情報: Kubernetes公式ドキュメント、EKS公式、GKE公式、AKS公式

### 5-5. Pod Security Admission

PodSecurityPolicy（PSP）は廃止済み。Pod Security Admission がNamespace単位のラベルで制御する。

```bash
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted
```

| プロファイル | 制約レベル | 用途 |
|:---|:---|:---|
| privileged | 制約なし | システムコンポーネント |
| baseline | 基本的な制約 | 汎用ワークロード |
| restricted | 最大制約 | **本番環境必須** |

本番Namespaceには `restricted` を適用する。

### 5-6. コンテナレジストリ運用

| レジストリ | プロバイダー | 特徴 |
|:---|:---|:---|
| ECR | AWS | IAM統合、ライフサイクルポリシー |
| Artifact Registry | Google Cloud | Container Registryは2025-03-18以降書き込み不可。Artifact Registry（gcr.io hosted含む）へ移行 |
| ACR | Azure | Entra ID統合、Tasks（CI/CDビルド） |
| Docker Hub | Docker | 公開レジストリ、レート制限あり |

**運用ルール：** イミュータブルタグを有効化する。未タグイメージは30日後に自動削除する。直近25世代の本番イメージを保持する。本番環境ではイメージダイジェスト（`image: myapp@sha256:<actual_digest>`）を使用する。CI/CDにPush権限、ランタイムにPull-only権限を付与する。

Docker Hubのレート制限は未認証で6時間あたり100プル、認証済み無料で200プル（2026-03-22確認時点）。プルスルーキャッシュ（ECR / Harbor）の導入を推奨する。

**【構成差分】** ECRはAWSアカウント・リージョン単位。Artifact Registryはプロジェクト・リージョン単位。ACRはリソースグループ単位。クロスリージョンレプリケーションはECR / ACR（Premium SKU）が対応。

出典・参考情報: ECR公式ドキュメント、Artifact Registry公式、ACR公式、Docker Hub公式

---

## 6. パフォーマンス最適化

> 汎用的なWebパフォーマンス最適化は [performance-optimization.md](/Users/TED/Documents/ClaudeCode/AI-instructions/performance-optimization.md) を参照。本セクションはサーバーインフラストラクチャー固有のパフォーマンス対策を規定する。

### 6-1. キャッシュ戦略

#### CDNキャッシュヘッダー設計

| コンテンツ種別 | Cache-Control | TTL |
|:---|:---|:---|
| バージョン付き静的ファイル（ハッシュ付きJS/CSS） | `public, max-age=31536000, immutable` | 1年 |
| バージョンなし画像 | `public, max-age=604800, stale-while-revalidate=86400` | 7日 |
| HTMLページ | `public, max-age=300, stale-while-revalidate=60` | 5分 |
| 公開APIレスポンス | `public, max-age=60, s-maxage=300` | ブラウザー1分、CDN 5分 |
| 認証済みAPIレスポンス | `private, no-cache, must-revalidate` | キャッシュなし |
| 機密データ | `no-store` | キャッシュ禁止 |

#### Varnish Cache

Varnish Cache 7.xはHTTPアクセラレーターの標準。VCL設定でキャッシュ動作を制御する。grace mode（`beresp.grace = 1h`）によりオリジンダウン時にステールコンテンツを最大1時間配信可能。

#### Redis HTTPキャッシュ

`maxmemory-policy allkeys-lru`、`aof-use-rdb-preamble yes`、`appendfsync everysec` を推奨設定とする。

**【構成差分】** CloudFrontはキャッシュポリシーで明示的にTTL・転送ヘッダーを制御する。Cloud CDNはキャッシュモードで動作を選択する。Cloudflareはデフォルトで静的ファイルをキャッシュし、Cache RulesでカスタムTTLを設定する。Front DoorはルールエンジンでCache-Controlヘッダーを操作する。Tiered Cache + Cache Reserve（Cloudflare）の組み合わせはオリジンリクエストを最小化し、キャッシュヒット率を最大化する。

出典・参考情報: Varnish Cache公式、Redis公式

### 6-2. カーネルパラメーターチューニング

```ini
# /etc/sysctl.d/99-performance.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_slow_start_after_idle = 0
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
vm.swappiness = 10
fs.file-max = 2097152
```

**TCP BBR** は `net.ipv4.tcp_congestion_control = bbr` で有効化する。高レイテンシー環境でスループットが改善される。

**ファイルディスクリプター上限**（`/etc/security/limits.conf`）：

```
www-data    soft    nofile    65535
www-data    hard    nofile    65535
```

systemd管理サービスでは `LimitNOFILE=65535` をサービスオーバーライドファイルに追加する。

### 6-3. 接続管理

**TCP keepalive：** `net.ipv4.tcp_keepalive_time = 60`、`net.ipv4.tcp_keepalive_intvl = 10`、`net.ipv4.tcp_keepalive_probes = 6` でアイドル接続の検知を高速化する。

**TCP Fast Open：** `net.ipv4.tcp_fastopen = 3`（サーバー + クライアント双方で有効化）。初回接続後のハンドシェイクを1RTT削減する。

**【構成差分】** コンテナ環境ではホストOSのカーネルパラメーターが共有される。Kubernetes Podから個別にsysctlを設定する場合は `securityContext.sysctls` で「安全な」sysctlのみ許可される（`net.ipv4.ip_local_port_range` 等）。`net.core.somaxconn` はPodレベルで変更可能だが、ノードレベルの設定が優先される場合がある。VM環境ではインスタンスごとに独立して設定可能。

出典・参考情報: Linux Kernel Documentation、TCP BBR (Google Research)

---

## 7. デプロイとインフラストラクチャー・アズ・コード

### 7-1. CI/CDパイプライン

**GitHub Actions: Dockerイメージビルド・プッシュ：**

```yaml
name: Build and Deploy
on:
  push:
    branches: [main]
    tags: ['v*']
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=semver,pattern={{version}}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

マルチ環境デプロイはブランチ / タグのトリガー条件で分離する（`main` → staging、`v*` タグ → production）。

### 7-2. デプロイ戦略

**Blue-Green：** 同一構成の2環境（Blue / Green）を維持し、ロードバランサーで一括切り替え。即時ロールバック可能。リソースコストは2倍。

**Canary：** トラフィックの一部（5〜10%）を新バージョンに振り分け、メトリクスを監視しながら段階的に拡大。nginx の weighted upstream で実装可能。

**Rolling Update：** Kubernetes Deployment の `maxSurge: 1, maxUnavailable: 0` でゼロダウンタイム達成を目標化できる。追加リソースは1Pod分のみ。

**ゼロダウンタイムチェックリスト：** ヘルスチェック設定済み、SIGTERM のgraceful shutdown実装済み、後方互換DBマイグレーション、ロードバランサーの接続ドレイン設定、クライアントのリトライロジック、ロールバック計画のテスト済み。

**【構成差分】** Blue-Greenは独立した環境が必要（AWS: 2つのTarget Group + ALBリスナールール切り替え）。Canaryはトラフィック分割機能が必要（nginx weighted upstream / Kubernetes Ingress アノテーション / Istio VirtualService）。Rolling UpdateはKubernetes Deployment の標準機能。

### 7-3. Terraform

**バージョン・プロバイダー管理：**

```hcl
terraform {
  required_version = ">= 1.14.0"
  required_providers {
    aws = { source = "hashicorp/aws"; version = "~> 6.0" }
  }
}
```

**【構成差分】ステート管理バックエンド：**

| プロバイダー | バックエンド | ロック機構 |
|:---|:---|:---|
| AWS | S3 | S3ロックファイル（`use_lockfile`） |
| Google Cloud | GCS | GCS ネイティブロック |
| Azure | Azure Storage | Azure Blob リース |
| OCI | OCI Object Storage | オブジェクトロックファイル（If-None-Match） |

モジュール設計方針：環境（dev / staging / production）ごとにディレクトリを分離し、共通リソースをモジュール化する。`terraform plan` の結果を必ずレビューしてから `apply` を実行する。

### 7-4. Ansible

べき等性（同じPlaybookを何度実行しても同じ結果になる）の原則を遵守する。

```bash
# Ad-hocコマンド
ansible all -m ping
ansible webservers -m apt -a "name=nginx state=present" -b
```

Ansible Vault で機密情報を暗号化する：`ansible-vault encrypt secrets.yml`。インベントリは静的（INI / YAML）または動的（クラウドプロバイダー連携）で管理する。

出典・参考情報: GitHub Actions公式、Terraform公式ドキュメント、Ansible公式ドキュメント

---

## 8. 用途別構成例

### 8-1. Webアプリケーション配信

リバースプロキシ（nginx）+ アプリケーションサーバー（Node.js / Gunicorn）構成。

```nginx
# map は http {} コンテキストに配置
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream nodejs_cluster {
    least_conn;
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    keepalive 64;
}

server {
    listen 443 ssl;
    http2 on;
    server_name app.example.com;

    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://nodejs_cluster;
    }
}
```

Python / Gunicorn の場合はUnixソケット接続を推奨（同一マシンでTCPオーバーヘッドを回避）。ワーカー数の計算式：(2 × CPUコア数) + 1。

**systemdサービス定義：**

```ini
[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/app
ExecStart=/usr/bin/node /var/www/app/server.js
Restart=on-failure
RestartSec=10
NoNewPrivileges=true
ProtectSystem=strict
```

**【構成差分】** Unixソケット接続はTCP接続より低レイテンシーだが、同一マシン限定。マルチサーバー構成ではTCP接続が必須。WebSocket対応は `Upgrade` / `Connection` ヘッダーの転送が必要。

### 8-2. API専用サーバー

```nginx
# map / limit_req_zone は http {} コンテキストに配置
map $http_origin $cors_origin {
    default "";
    "~^https://(app|admin)\.example\.com$" $http_origin;
}

# 未許可Origin付きCORSリクエストは拒否
map $http_origin $cors_reject {
    default 1;
    "" 0;
    "~^https://(app|admin)\.example\.com$" 0;
}

limit_req_zone $binary_remote_addr zone=api_general:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=api_auth:10m rate=5r/m;

server {
    listen 443 ssl;
    http2 on;
    server_name api.example.com;

    client_max_body_size 1m;
    proxy_connect_timeout 10s;
    proxy_read_timeout 30s;
    default_type application/json;

    if ($cors_reject) { return 403; }

    location /v1/ {
        if ($http_origin != "") {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Vary "Origin" always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
            add_header Access-Control-Max-Age 86400 always;
        }

        if ($request_method = OPTIONS) {
            add_header Content-Type text/plain always;
            add_header Content-Length 0 always;
            return 204;
        }

        limit_req zone=api_general burst=20 nodelay;
        proxy_pass http://127.0.0.1:3000/;
    }
    location /v1/auth/ {
        if ($http_origin != "") {
            add_header Access-Control-Allow-Origin $cors_origin always;
            add_header Vary "Origin" always;
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
            add_header Access-Control-Max-Age 86400 always;
        }

        if ($request_method = OPTIONS) {
            add_header Content-Type text/plain always;
            add_header Content-Length 0 always;
            return 204;
        }

        limit_req zone=api_auth burst=5 nodelay;
        proxy_pass http://127.0.0.1:3000/auth/;
    }

    error_page 429 @rate_limited;
    location @rate_limited {
        add_header Content-Type application/json always;
        add_header Retry-After 60 always;
        return 429 '{"error":"Rate limit exceeded","retry_after":60}';
    }
}
```

### 8-3. 静的ファイル配信

`sendfile on`、`tcp_nopush on`、`tcp_nodelay on`、`open_file_cache` を有効化する。フィンガープリント付きアセットには1年のimmutableキャッシュ、HTMLには5分キャッシュを設定する。

ビルド時にプリコンプレッション（`brotli -q 11 -k` / `gzip -9 -k`）を実行し、`brotli_static on` / `gzip_static on` で配信する。

### 8-4. メディア配信

```nginx
location /videos/ {
    mp4;
    mp4_buffer_size 1m;
    mp4_max_buffer_size 5m;
    limit_rate_after 10m;
    limit_rate 1m;
}

location /downloads/ {
    # http {} で limit_conn_zone $binary_remote_addr zone=perip:10m; を定義しておく
    limit_conn perip 3;
    limit_rate_after 5m;
    limit_rate 512k;
}
```

`limit_rate_after` で動画インデックスの高速取得を許可し、その後帯域を制限する。大容量ファイルダウンロードでは `limit_conn` でIP単位の同時接続数を制限する。

**【構成差分】** 大規模メディア配信ではCDN経由配信を推奨する（CloudFront / Cloud CDN / Cloudflare）。自前配信はCDNコストを回避できるがオリジン負荷が高い。判断基準：同時視聴者数100未満は自前配信で対応可能、100以上はCDNを検討する。

### 8-5. バッチ処理専用

systemd timerをcronの代替として使用する（ログ・リソース制御が優れている）。

```ini
# /etc/systemd/system/batch-job.service
[Service]
Type=oneshot
User=batchuser
ExecStart=/opt/batch/process.py
Nice=19
IOSchedulingClass=idle
CPUWeight=50
MemoryMax=2G
OOMScoreAdjust=500
```

```ini
# /etc/systemd/system/batch-job.timer
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=300
```

`Nice=19`（最低優先度）、`IOSchedulingClass=idle`（I/Oアイドル時のみ）、`OOMScoreAdjust=500`（OOMKiller優先対象）で、バッチ処理がWebサーバーのパフォーマンスに影響しないよう隔離する。

---

## 9. 監視とアラート

### 9-1. リソース監視

Prometheus + Grafana + Node Exporter を標準構成とする。Node Exporter を全サーバーにインストールし、Prometheusで5秒間隔にスクレイプする。

**prometheus.yml：**

```yaml
global:
  scrape_interval: 5s
scrape_configs:
  - job_name: 'node_exporter'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
```

**メトリクス × 閾値テーブル：**

| メトリクス | 正常 | 警告 | 危険 |
|:---|:---|:---|:---|
| CPU使用率 | 70%未満 | 70〜90% | 90%超 |
| メモリー使用率 | 70%未満 | 70〜85% | 85%超 |
| ディスク使用率 | 70%未満 | 70〜90%未満 | 90%以上 |
| ディスクI/O Wait | 10%未満 | 10〜25% | 25%超 |
| Load Average | CPUコア数未満 | 1〜2倍 | 2倍超 |
| Swap使用率 | 10%未満 | 10〜50% | 50%超 |

Grafanaダッシュボード **#1860**（Node Exporter Full）を推奨。

### 9-2. アラート設計

**Alertmanagerルール例：**

```yaml
groups:
  - name: infrastructure
    rules:
      - alert: HighCPUUsage
        expr: >
          100 - (avg by(instance)
          (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels: { severity: warning }

      - alert: DiskSpaceLow
        expr: >
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 10m
        labels: { severity: critical }

      - alert: HighErrorRate
        expr: >
          (rate(http_requests_total[5m]) > 0)
          and (
            rate(http_requests_total{status=~"5.."}[5m])
            / rate(http_requests_total[5m]) > 0.05
          )
        for: 5m
        labels: { severity: critical }

      - alert: SSLCertExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels: { severity: warning }
```

**ルーティング：** Critical → PagerDuty（15分以内対応）、Warning → Slack（1時間以内対応）、Info → Email（翌営業日対応）。
`SSLCertExpiringSoon` は Blackbox Exporter 等で `probe_ssl_earliest_cert_expiry` を収集している前提。

**抑制ルール（inhibition）：** 同一インスタンスでCriticalとWarningが同時に発報された場合、Warningを抑制する。`for` パラメーターで一過性スパイクを除外する（CPU: 5分、ディスク: 10分）。

### 9-3. APM

**RED手法：** Rate（リクエスト/秒）、Errors（目標: 0.1%未満）、Duration（P50 < 200ms、P95 < 500ms、P99 < 1s）。

**OpenTelemetry計装：**

```javascript
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: 'http://jaeger:4318/v1/traces'
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

分散トレーシングのバックエンドとしてJaeger（OSS）を使用する。

**【構成差分】** クラウドマネージドAPM（CloudWatch Application Insights / Cloud Monitoring / Azure Monitor Application Insights）は導入が容易だが、ベンダーロックインとなる。マルチクラウド環境ではOpenTelemetry + Jaeger（OSS）を推奨する。小規模環境での簡易監視にNetdataを使用する場合は、配布手順を検証し、パッケージ署名またはチェックサム検証を実施して導入する。

出典・参考情報: Prometheus公式ドキュメント、Grafana公式、OpenTelemetry公式、Jaeger公式

---

## 出典・参考情報

### Webサーバー公式ドキュメント

- [nginx Documentation](https://nginx.org/en/docs/) - nginx公式ドキュメント
- [nginx CHANGES](https://nginx.org/en/CHANGES) - nginx変更履歴
- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/current/) - Apache httpd公式ドキュメント
- [Apache Module Index](https://httpd.apache.org/docs/current/mod/) - Apacheモジュール一覧

### クラウドプロバイダー公式ドキュメント

- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/) - EC2公式
- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/) - VPC公式
- [AWS ELB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/) - ELB公式
- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/) - CloudFront公式
- [AWS WAF Documentation](https://docs.aws.amazon.com/waf/) - WAF公式
- [AWS Shield Standard Overview](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-standard-summary.html) - L3/L4 DDoS防御
- [AWS WAF Rate-based Rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html) - レートベース遮断
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) - IAMベストプラクティス
- [AWS ACM Documentation](https://docs.aws.amazon.com/acm/) - ACM公式
- [Google Cloud Compute Engine](https://cloud.google.com/compute/docs) - Compute Engine公式
- [Google Cloud VPC](https://cloud.google.com/vpc/docs) - VPC公式
- [Google Cloud Load Balancing](https://cloud.google.com/load-balancing/docs) - Cloud Load Balancing公式
- [Google Cloud CDN](https://cloud.google.com/cdn/docs) - Cloud CDN公式
- [Google Cloud Armor Overview](https://docs.cloud.google.com/armor/docs/cloud-armor-overview) - DDoS/WAF保護
- [Google Cloud IAM](https://cloud.google.com/iam/docs) - IAM公式
- [Google Cloud Certificate Manager](https://cloud.google.com/certificate-manager/docs) - Certificate Manager公式
- [Azure Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/) - VM公式
- [Azure VNet](https://learn.microsoft.com/en-us/azure/virtual-network/) - VNet公式
- [Azure Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/) - Application Gateway公式
- [Azure Front Door](https://learn.microsoft.com/en-us/azure/frontdoor/) - Front Door公式
- [Azure WAF](https://learn.microsoft.com/en-us/azure/web-application-firewall/) - WAF公式
- [Azure DDoS Protection Overview](https://learn.microsoft.com/en-us/azure/ddos-protection/ddos-protection-overview) - L3/L4 DDoS保護
- [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/) - Key Vault公式
- [OCI Physical Architecture Concepts](https://docs.oracle.com/en-us/iaas/Content/GSG/Concepts/concepts-physical.htm) - Region / AD / FDの基礎
- [OCI Compute](https://docs.oracle.com/en-us/iaas/Content/Compute/home.htm) - Compute公式
- [OCI VCNs and Subnets](https://docs.oracle.com/en-us/iaas/Content/Network/Tasks/VCNs.htm) - VCN/サブネット
- [OCI Network Security Groups](https://docs.oracle.com/iaas/Content/Network/Concepts/networksecuritygroups.htm) - NSG公式
- [OCI Getting Started with Policies](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/policygetstarted.htm) - IAMポリシー
- [OCI Managing Compartments](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingcompartments.htm) - コンパートメント設計
- [OCI Bastion Overview](https://docs.oracle.com/en-us/iaas/Content/Bastion/Concepts/bastionoverview.htm) - Bastion
- [OCI Vault](https://docs.oracle.com/en-us/iaas/Content/KeyManagement/home.htm) - 鍵管理
- [OCI Cloud Guard](https://docs.oracle.com/en-us/iaas/Content/cloud-guard/home.htm) - セキュリティ姿勢監視
- [OCI Security Zones](https://docs.oracle.com/en-us/iaas/Content/security-zone/home.htm) - 予防的ポリシー制御
- [OCI Web Application Firewall](https://docs.oracle.com/en-us/iaas/Content/WAF/home.htm) - WAF公式
- [OCI Layer 7 DDoS Mitigation](https://docs.oracle.com/en-us/iaas/Content/WAF/Concepts/ddos.htm) - L7 DDoS対策
- [OCI WAF Access Rules](https://docs.oracle.com/en-us/iaas/Content/WAF/Tasks/access-rules.htm) - 国/地域・IPルール
- [OCI Load Balancer](https://docs.oracle.com/en-us/iaas/Content/Balance/) - L7 LB公式
- [OCI Network Load Balancer](https://docs.oracle.com/en-us/iaas/Content/NetworkLoadBalancer/home.htm) - L3/L4 NLB公式
- [OCI Autoscaling](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/autoscalinginstancepools.htm) - Instance Pool自動スケール
- [OCI Block Volume Performance](https://docs.oracle.com/en-us/iaas/Content/Block/Concepts/blockvolumeperformance.htm) - ストレージ性能
- [OCI FastConnect Overview](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/fastconnectoverview.htm) - 専用線接続
- [OCI Monitoring Overview](https://docs.oracle.com/en-us/iaas/Content/Monitoring/Concepts/monitoringoverview.htm) - メトリクス/アラーム
- [OCI Logging Overview](https://docs.oracle.com/en-us/iaas/Content/Logging/Concepts/loggingoverview.htm) - ログ集約

### CDN / エッジプラットフォーム

- [Cloudflare Developer Docs](https://developers.cloudflare.com/) - Cloudflare公式ドキュメント
- [Cloudflare DDoS Protection](https://developers.cloudflare.com/ddos-protection/) - DDoS防御仕様
- [Cloudflare SSL/TLS](https://developers.cloudflare.com/ssl/) - Cloudflare SSL/TLS設定
- [Cloudflare Cache](https://developers.cloudflare.com/cache/) - Cloudflareキャッシュ設定
- [Cloudflare WAF](https://developers.cloudflare.com/waf/) - Cloudflare WAF設定
- [Cloudflare Workers](https://developers.cloudflare.com/workers/) - Cloudflare Workers公式
- [Cloudflare IP Ranges](https://www.cloudflare.com/ips/) - Cloudflare IPアドレス範囲

### セキュリティ情報・規格

- [Mozilla Server Side TLS](https://wiki.mozilla.org/Security/Server_Side_TLS) - TLS設定推奨事項
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) - SSL設定ジェネレーター
- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) - セキュリティヘッダー
- [OWASP ModSecurity Core Rule Set](https://coreruleset.org/) - OWASP CRS公式
- [ModSecurity Documentation](https://github.com/owasp-modsecurity/ModSecurity/wiki) - ModSecurity公式
- [OpenSSH Manual (sshd_config)](https://man.openbsd.org/sshd_config) - SSHサーバー設定リファレンス
- [SafeCurves](https://safecurves.cr.yp.to/) - 楕円曲線の安全性評価（Bernstein & Lange）
- [RFC 8032](https://www.rfc-editor.org/rfc/rfc8032) - EdDSA（Ed25519）仕様
- [NIST FIPS 186-5](https://csrc.nist.gov/pubs/fips/186-5/final) - Digital Signature Standard
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) - OSセキュリティベンチマーク
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/) - Let's Encrypt公式
- [Certbot Documentation](https://certbot.eff.org/docs/) - Certbot公式
- [NVD](https://nvd.nist.gov/) - National Vulnerability Database
- [JVN](https://jvn.jp/) - Japan Vulnerability Notes

### コンテナ技術公式ドキュメント

- [Docker Documentation](https://docs.docker.com/) - Docker公式
- [Dockerfile Best Practices](https://docs.docker.com/build/building/best-practices/) - Dockerfileベストプラクティス
- [Docker Compose Specification](https://docs.docker.com/compose/compose-file/) - Compose仕様
- [Kubernetes Documentation](https://kubernetes.io/docs/) - Kubernetes公式
- [Kubernetes Releases](https://kubernetes.io/releases/) - Kubernetesリリース情報
- [Trivy](https://trivy.dev/) - コンテナイメージスキャン
- [Sigstore / Cosign](https://docs.sigstore.dev/) - イメージ署名

### IaC・デプロイツール

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs) - Terraform公式
- [Ansible Documentation](https://docs.ansible.com/) - Ansible公式
- [GitHub Actions Documentation](https://docs.github.com/en/actions) - GitHub Actions公式

### 監視ツール

- [Prometheus Documentation](https://prometheus.io/docs/) - Prometheus公式
- [Grafana Documentation](https://grafana.com/docs/) - Grafana公式
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/) - OpenTelemetry公式
- [Jaeger Documentation](https://www.jaegertracing.io/docs/) - Jaeger公式

### VPSプロバイダー

- [さくらVPS 料金・仕様一覧](https://vps.sakura.ad.jp/specification/) - さくらVPS公式
- [ConoHa VPS 料金・スペック](https://vps.conoha.jp/pricing/) - ConoHa VPS公式

### 関連ファイル

- [secure-code-requirements.md](/Users/TED/Documents/ClaudeCode/AI-instructions/secure-code-requirements.md) - セキュリティ要件（§4との連動元）
- [performance-optimization.md](/Users/TED/Documents/ClaudeCode/AI-instructions/performance-optimization.md) - パフォーマンス最適化（§6との棲み分け元）
- [database-design-guidelines.md](/Users/TED/Documents/ClaudeCode/AI-instructions/database-design-guidelines.md) - データベース設計（§非対象で明示）
- [coding-standards.md](/Users/TED/Documents/ClaudeCode/AI-instructions/coding-standards.md) - コーディング規約
- [domain-placement-matrix.md](/Users/TED/Documents/ClaudeCode/AI-instructions/domain-placement-matrix.md) - ドメイン配置マトリクス
