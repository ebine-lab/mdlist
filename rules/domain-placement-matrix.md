---
title: "Domain Placement Matrix"
description: "ドメイン配置意思決定マトリクス / Domain placement decision matrix"
version: "1.0.0"
status: "Stable"
last_updated: "2026-01-28T21:30+09:00"
lang: "ja"
---

# Domain Placement Matrix

**ドメイン配置意思決定マトリクス** - 新サービスのURL配置方式を比較し、採用方針を定義


---

## 目的
- 新サービスの URL 配置方式を比較可能な形で提示する
- 新規ドメイン取得は原則として推奨しない
- 廃止時の管理負債としてドメイン終活リスクを比較に含める

## 比較対象
- A：既存ドメイン配下ディレクトリ `example.com/service`
- B：既存ドメイン配下サブドメイン `service.example.com`
- C：新規ドメイン（親ドメイン配下ではなく、別に登録するドメイン）例：`service.com`

## マトリクス

### 前提条件
本マトリクスにおいてA（ディレクトリ方式）が有利と評価されるのは、母体となるドメインが以下の条件を満たす場合に限る。

- 市場において十分な認知度を確立している
- ブランド価値が確立され、信頼性の基盤として機能している
- 既存顧客基盤およびステークホルダーに広く浸透している

母体ドメインの認知度が低い、または新規事業として独立したブランディングが求められる場合は、この評価は適用されない、または逆転する場合もある。

### 評価の凡例
- 5：最も有利/負担最小
- 4：有利/負担小
- 3：中立/負担中
- 2：やや不利/負担やや大
- 1：不利/負担最大

| 評価軸 | 判断基準 | A ディレクトリ | B サブドメイン | C 新規ドメイン |
|---|---|---|---|---|
| 母体事業との連携性 | 既存サービスとの一体感/既存導線・顧客基盤との接続 | 5 | 4 | 1 |
| サービス認知の容易性 | URLで公式性・所属が伝わる/共有時に誤認が起きにくい | 5 | 4 | 1 |
| URLの簡潔さ | 短く意味が推測できる/入力・共有で迷いにくい | 3 | 4 | 5 |
| 周知・正規性説明コスト | 社内外への説明・FAQ・注意喚起の必要量 | 5 | 4 | 1 |
| 運用負担（DNS/証明書/監視/権限） | 運用の増分 | 5 | 3※ | 1 |
| 終活負債（廃止時） | 後処理の重さ/残存設定による悪用リスク | 4 | 3※ | 1 |

※ B（サブドメイン）を「3」とする理由（運用負担/終活負債 共通）
- DNS レコード追加と、その棚卸し・削除・継続監視が増える
- TLS 対象（ホスト名）が増えるため、証明書の発行・更新・期限監視が増える
- 監視対象（死活、DNS 変更、証明書期限、設定変更）が増える
- DNS 変更/証明書更新/監視対応の責務分担と権限運用が増える

## 採用方針
- A または B を採用対象とする
- C は原則として採用しない

## URL命名条件
対象はディレクトリ名・サブドメイン名・独立したドメイン名とする。

- 短く、口頭伝達で誤認しない
- 長い単語や語句、スペルチェックがないと間違えるような単語を使用しない
- 意味が推測できる語を使う
- 読み間違い・打ち間違いを誘発しない
- 区切りはハイフン（`-`）を基本とし、アンダースコア（`_`）は避ける
- 省略語（abbreviation）・頭字語（acronym）は以下の基準で判断する
  - 広く認知された省略語は使用可（例：`api`、`id`、`faq`、`pdf`）
  - 専門的または独自の省略語は避け、完全な語を優先（例：`fn` → `first-name`）
  - 対象者が即座に理解できない省略語は使用しない

## 付録：ドメインの防御的取得と誘導

### 将来展開に備えた先行取得

サービスとして独立していなくてもドメインを取得し、正規URLへ誘導する。

- **意図**：将来の展開余地の確保、誤認誘導の受け皿、外部露出増加時の移行コスト低減
- **実装**：A/CNAMEレコードで到達先を制御し、HTTPリダイレクト（301/308）で正規URLへ誘導

### 誤入力・類似ドメインの取得

間違えられそうなドメインを取得し、正規URLへリダイレクトする。

- **対象例**：ハイフン有無、語順違い、タイプミス、見間違いが起きやすい文字列
- **目的**：誤入力・誤認を正規URLへ吸収し、フィッシング等の悪用余地を減らす

### TLD違いの網羅取得

同名でトップレベルドメイン違いのドメイン（例：`service.jp` / `service.net` / `service.org` 等）が取得可能な場合、全て取得して保持する。

- **目的**：第三者取得による偽装サイト化を抑止し、正規URLへの誘導統制を維持
- **留意**：取得数が増えるほど更新・棚卸し・失効防止の運用負債が増えるため、管理責任者と更新手順を最初に固定する

## 出典・参考情報
### ドメイン終活
- [ドメイン名の終活について - JPAAWG 7th -](https://speakerdeck.com/mikit/domeinming-nozhong-huo-nituite-jpaawg-7th)
- [ドメインの終活ロードマップを考えてみる](https://zenn.dev/banboobloom/articles/2025030600001)
- [使わないドメインが悪用される..考えておくべきドメインの終活問題](https://grphca.jp/1314/)

### URL構造・命名規則
- [Google 検索における URL 構造のベスト プラクティス](https://developers.google.com/search/docs/crawling-indexing/url-structure?hl=ja)
- [Google developer documentation style guide - Abbreviations](https://developers.google.com/style/abbreviations)
- [Google AIP-190: Naming conventions](https://google.aip.dev/190)
- [REST API URI Naming Conventions and Best Practices](https://restfulapi.net/resource-naming/)
- [URL最適化](https://blog.asobou.co.jp/web/url-optimisation)
- [わかりやすくシンプルなURLにする重要性とオススメの決め方](https://kumaweb-d.com/blog/seo-url/)

### 運用リスク
- [WSTG Test for Subdomain Takeover](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/02-Configuration_and_Deployment_Management_Testing/10-Test_for_Subdomain_Takeover)
- [終わったWebサイトのDNS設定、そのままになっていませんか?（Dangling Records）](https://jprs.jp/tech/security/2025-01-21-danglingrecords.pdf)
- [MITRE ATT&CK: Acquire Infrastructure: Domains](https://attack.mitre.org/techniques/T1583/001/)
- [ドメイン名ハイジャックによるドメイン名の乗っ取りに注意](https://www.cybertrust.co.jp/blog/certificate-authority/client-authentication/domain-hijacking.html)
