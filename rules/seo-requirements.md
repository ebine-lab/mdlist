---
title: "SEO Requirements"
description: "SEO技術要件 - メタタグ・構造化データ・OGP・sitemap・robots.txt / SEO technical requirements - meta tags, structured data, OGP, sitemap, robots.txt"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-06T22:00+09:00"
lang: "ja"
---

# SEO Requirements

**検索エンジン最適化の技術要件** - メタタグ、構造化データ、ソーシャルメディア連携、クロール制御の実装標準を定義する。


## 目的

検索エンジンとソーシャルプラットフォームに対して、ページの内容・構造・関係性を正確に伝達するための技術標準を定める。検索結果での表示品質の向上、ソーシャルメディアでの共有時の表示制御、クローラーへの適切な指示を通じて、サイトの発見性と到達性を確保する。

## 対象

- HTMLページの`<head>`セクション内メタデータ
- Schema.org構造化データ（JSON-LD）
- Open Graph / X Cards ソーシャルメタデータ
- Apple Smart App Banners
- sitemap.xml / robots.txt によるクロール制御

## 非対象

- コンテンツSEO（キーワード戦略、文章構成）
- リンクビルディング
- サーバサイドの速度最適化（performance-optimization.md で扱う予定）
- CMS固有のプラグイン設定

---

## コード例の表記規則

コード例内で実装時に実際の値へ置き換える箇所にはプレースホルダーを使用する。必要に応じて凡例表を添える。

| プレースホルダー | 説明 |
| --- | --- |
| `YOUR_PAGE_TITLE` | ページ固有のタイトル |
| `YOUR_PAGE_DESCRIPTION` | ページ内容の要約文 |
| `YOUR_SOCIAL_DESCRIPTION` | ソーシャル共有向けの説明文 |
| `YOUR_ARTICLE_TITLE` | 記事のタイトル |
| `YOUR_SITE_NAME` | サイト名称 |
| `YOUR_AUTHOR_NAME` | 著者名 |
| `YOUR_X_ACCOUNT` | サイトのX（旧Twitter）アカウント名 |
| `YOUR_AUTHOR_X_ACCOUNT` | 著者のXアカウント名 |
| `YOUR_IMAGE_ALT_TEXT` | 画像の代替テキスト |
| `YOUR_APP_ID` | App StoreのアプリID |
| `YOUR_AFFILIATE_ID` | iTunesアフィリエイトID |
| `YOUR_CAMPAIGN_NAME` | キャンペーン名 |
| `example.com` | サイトのドメイン（[RFC 2606](https://www.rfc-editor.org/rfc/rfc2606)予約ドメイン） |

---

## メタタグ

参照：[Meta Tags and Attributes that Google Supports](https://developers.google.com/search/docs/crawling-indexing/special-tags)、[How to Write Meta Descriptions](https://developers.google.com/search/docs/appearance/snippet)

### 必須メタタグ

すべてのHTMLページの`<head>`内に以下を含める。

#### charset

```html
<meta charset="UTF-8">
```

- `<head>`内の最初の要素として配置する。
- UTF-8以外のエンコーディングは使用しない。

#### title

```html
<title>YOUR_PAGE_TITLE</title>
```

| 項目 | 基準 |
| --- | --- |
| 文字数 | 全角30文字以内（半角60文字以内） |
| 一意性 | ページごとに固有のタイトルを設定する |
| キーワード | 主要キーワードを先頭付近に配置する |
| ブランド名 | 必要に応じて末尾に「 \| サイト名」を付加する |
| 禁止事項 | キーワードの羅列、全ページ同一タイトル、曖昧な汎用タイトル |

Googleはtitleタグを書き換える場合がある（推定60%以上）。それでもtitleタグは検索順位に影響するランキング要因であるため、適切な設定が必要である。

#### description

```html
<meta name="description" content="YOUR_PAGE_DESCRIPTION">
```

| 項目 | 基準 |
| --- | --- |
| 文字数 | 全角70から80文字（半角140から160文字） |
| 一意性 | ページごとに固有の説明を設定する |
| 内容 | ページの内容を正確かつ具体的に要約する |
| 禁止事項 | キーワードの羅列、全ページ同一の説明文、ページ内容と無関係な記述 |

descriptionはランキング要因ではないが、検索結果のスニペットとして表示される可能性があり、クリック率に影響する。Googleは独自にスニペットを生成する場合もあるが、descriptionの内容が適切であれば採用される確率が高い。

#### robots

```html
<meta name="robots" content="index, follow">
```

| 値 | 効果 |
| --- | --- |
| `index` | ページをインデックスに含める（既定値） |
| `noindex` | ページをインデックスから除外する |
| `follow` | ページ内のリンクをたどる（既定値） |
| `nofollow` | ページ内のリンクをたどらない |
| `noarchive` | キャッシュを表示しない |
| `nosnippet` | スニペットを表示しない |
| `max-snippet:[数値]` | スニペットの最大文字数を指定する |
| `max-image-preview:[値]` | 画像プレビューの最大サイズ（`none`、`standard`、`large`） |
| `max-video-preview:[秒数]` | 動画プレビューの最大秒数 |

既定動作は`index, follow`であるため、インデックス対象のページでは省略可能である。`noindex`を指定する場合、robots.txtでクロールをブロックしていないことを確認する（robots.txtでブロックするとメタタグ自体が読み取られない）。

Googlebot固有の指示が必要な場合は`name="googlebot"`を使用する。Google News向けには`name="googlebot-news"`を使用する。

#### canonical

```html
<link rel="canonical" href="https://example.com/page/">
```

| 項目 | 基準 |
| --- | --- |
| URL形式 | 絶対パスを使用する（相対パスは非推奨） |
| 配置 | `<head>`セクション内 |
| 対象 | 重複コンテンツが存在する全ページ |
| 自己参照 | 正規ページ自体にも自己参照canonicalを設定する |
| hreflang | 同一言語内のcanonicalを指定する（異なる言語版をcanonicalに指定しない） |

参照：[How to Specify a Canonical](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls)

### Googleが無視するメタタグ

以下のメタタグはGoogleの検索ランキングに影響しない。使用しないこと。

- `<meta name="keywords">` — Googleは無視する。参照：[Google does not use the keywords meta tag](https://developers.google.com/search/blog/2009/09/google-does-not-use-keywords-meta-tag)
- `<meta name="author">` — Google検索では使用されない。
- `<meta name="revised">` — Google検索では使用されない。

### X-Robots-Tag

PDF、画像、動画などHTML以外のリソースには、HTTPレスポンスヘッダーで`X-Robots-Tag`を使用してインデックス制御を行う。

```
X-Robots-Tag: noindex
```

参照：[Robots Meta Tags Specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag)

### Viewport

参照：[Viewport meta tag - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag)

viewportメタタグはSEOではなくレスポンシブデザインのための設定だが、Googleのモバイルフレンドリー評価に直接影響するため本文書で定義する。

#### 標準設定

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
```

この1行をすべてのHTMLページに含める。

#### プロパティ一覧

| プロパティ | 値 | 説明 |
| --- | --- | --- |
| `width` | `device-width` または正の整数（1から10000） | ビューポートの幅。`device-width`を推奨する。固定値（`width=320`等）は使用しない。 |
| `height` | `device-height` または正の整数 | ビューポートの高さ。通常は指定不要。 |
| `initial-scale` | `0.1`から`10`の数値 | 初期ズーム倍率。`1`を指定する。 |
| `minimum-scale` | `0.1`から`10`の数値 | 最小ズーム倍率。既定値`0.1`。 |
| `maximum-scale` | `0.1`から`10`の数値 | 最大ズーム倍率。既定値`10`。 |
| `user-scalable` | `yes` または `no` | ユーザーによるズーム操作の許可。 |
| `interactive-widget` | `resizes-visual`、`resizes-content`、`overlays-content` | 仮想キーボード表示時のビューポート挙動。 |

#### アクセシビリティ要件（厳守）

| 禁止設定 | 理由 |
| --- | --- |
| `user-scalable=no` | WCAG 2.2 達成基準 1.4.4（テキストのサイズ変更）に違反する。弱視のユーザーがコンテンツを拡大できなくなる。 |
| `maximum-scale=1` | 実質的にズームを無効化するため同様に違反する。 |

WCAG 2.2では最低2倍のズームを要求し、5倍のズームを推奨している。`maximum-scale`を指定する場合は`3`以上とする。

参照：[WCAG 2.2 Success Criterion 1.4.4](https://www.w3.org/TR/WCAG22/#resize-text)

#### interactive-widget

仮想キーボード表示時のビューポート動作を制御する。

| 値 | 動作 |
| --- | --- |
| `resizes-visual` | ビジュアルビューポートのみリサイズする（既定値） |
| `resizes-content` | レイアウトビューポートとビジュアルビューポートの両方をリサイズする |
| `overlays-content` | いずれのビューポートもリサイズしない |

フォーム入力が多いページでは`resizes-content`を検討する。

```html
<meta name="viewport" content="width=device-width, initial-scale=1, interactive-widget=resizes-content">
```

### Apple Smart App Banners

参照：[Promoting Apps with Smart App Banners](https://developer.apple.com/documentation/webkit/promoting-apps-with-smart-app-banners)

iOS Safariでウェブページ上部にApp Storeへの誘導バナーを表示する機能。関連するiOSアプリが存在するサイトで使用する。

#### 構文

```html
<meta name="apple-itunes-app" content="app-id=YOUR_APP_ID, app-argument=https://example.com/page/">
```

`YOUR_APP_ID`はApp Storeの実際のアプリIDに置き換える。

#### パラメーター

| パラメーター | 必須 | 説明 |
| --- | --- | --- |
| `app-id` | 必須 | App StoreのアプリID（数値）。App Store Connectまたは App StoreのURLから取得する（`https://apps.apple.com/app/idYOUR_APP_ID` の数値部分）。 |
| `affiliate-data` | 任意 | iTunesアフィリエイトの文字列。キャンペーントラッキングにも使用可能（例：`at=YOUR_AFFILIATE_ID&ct=YOUR_CAMPAIGN_NAME`）。 |
| `app-argument` | 任意 | アプリ起動時に渡すURL。ウェブページとアプリ内コンテンツの対応付けに使用する。 |

#### 動作仕様

| 条件 | 動作 |
| --- | --- |
| アプリ未インストール | 「表示」ボタンでApp Storeに遷移する |
| アプリインストール済み | 「開く」ボタンでアプリを起動する（`app-argument`が渡される） |
| ダウンロード中 | 進捗バーを表示する |
| ユーザーが閉じた場合 | 同一サイトで再表示されない（ブラウザのデータ消去まで） |
| 非対応デバイス | バナーを表示しない |

#### 制約事項

- iOS Safari限定（Chrome等の他ブラウザでは表示されない）
- Universal Linksが有効かつアプリがインストール済みの場合、Safariは自動的に小型のバナーを表示する（メタタグとは別の仕組み）
- `app-argument`内のカンマはURLエンコード（`%2C`）する
- ディープリンク（アプリ内の特定画面への遷移）はアプリがインストール済みの場合のみ機能する

---

## 構造化データ

参照：[Intro to Structured Data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)、[General Structured Data Guidelines](https://developers.google.com/search/docs/appearance/structured-data/sd-policies)

### 形式

JSON-LDを使用する。Googleが推奨する形式であり、HTMLと分離して管理できるため保守性が高い。MicrodataやRDFaは使用しない。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "YOUR_ARTICLE_TITLE",
  "author": {
    "@type": "Person",
    "name": "YOUR_AUTHOR_NAME"
  },
  "datePublished": "2026-02-06",
  "dateModified": "2026-02-06"
}
</script>
```

### 実装規約

| 項目 | 基準 |
| --- | --- |
| 配置 | `<head>`内または`<body>`内（1エンティティにつき1箇所にまとめる） |
| 内容の一致 | 構造化データはページ上に表示されている内容と一致させる。ページに存在しない情報を構造化データに含めない。 |
| 必須プロパティ | 各スキーマタイプのrequiredプロパティをすべて含める |
| 推奨プロパティ | recommendedプロパティも可能な限り含める（リッチリザルト適格性が向上する） |
| `@id`の一貫性 | 同一エンティティには同じ`@id`値を使用し、ページ間で一貫性を保つ |
| `sameAs` | 同一エンティティの外部参照URL（公式サイト、SNSプロフィール等）を指定する |
| 重複排除 | 複数のCMSプラグインによる重複スキーマ出力を防ぐ |

### 配置方針

| スキーマタイプ | 配置対象 |
| --- | --- |
| `Organization` | トップページのみ |
| `WebSite` | トップページのみ |
| `BreadcrumbList` | 全ページ（トップページ除く） |
| `Article` / `BlogPosting` | 記事ページ |
| `Product` | 商品ページ |
| `FAQPage` | FAQページ |
| `LocalBusiness` | 店舗・拠点ページ |
| `Event` | イベントページ |
| `VideoObject` | 動画を含むページ |

すべてのページにすべてのスキーマタイプを配置する必要はない。ページの内容に合致するスキーマタイプのみを使用する。

### 検証

実装後は必ず以下のツールで検証する。

- [Rich Results Test](https://search.google.com/test/rich-results) - Googleリッチリザルト適格性の確認
- [Schema Markup Validator](https://validator.schema.org/) - Schema.org構文の検証

Google Search Consoleの「拡張」レポートでエラーを継続的に監視する。テンプレート変更後は再検証を行う。

---

## OGP（Open Graph Protocol）

参照：[The Open Graph protocol](https://ogp.me/)

ソーシャルメディア（Facebook、LinkedIn等）でURLが共有された際のプレビュー表示を制御する。

### 必須プロパティ

```html
<meta property="og:title" content="YOUR_PAGE_TITLE">
<meta property="og:type" content="website">
<meta property="og:url" content="https://example.com/page/">
<meta property="og:image" content="https://example.com/images/og-image.jpg">
```

| プロパティ | 説明 | 要件 |
| --- | --- | --- |
| `og:title` | 共有時に表示されるタイトル | 全角30文字以内を推奨。`<title>`と同じでもよいが、ソーシャル向けに調整可能。 |
| `og:type` | コンテンツタイプ | トップページは`website`、記事ページは`article`を指定する。 |
| `og:url` | 正規URL | canonicalと同一のURLを指定する。 |
| `og:image` | プレビュー画像のURL | 絶対URLで指定する。 |

### 推奨プロパティ

```html
<meta property="og:description" content="YOUR_PAGE_DESCRIPTION">
<meta property="og:site_name" content="YOUR_SITE_NAME">
<meta property="og:locale" content="ja_JP">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="YOUR_IMAGE_ALT_TEXT">
```

| プロパティ | 説明 |
| --- | --- |
| `og:description` | 共有時の説明文。全角100文字以内を推奨。 |
| `og:site_name` | サイトの名称。 |
| `og:locale` | 言語と地域。日本語は`ja_JP`。 |
| `og:image:width` / `og:image:height` | 画像の幅と高さをピクセルで指定する。プラットフォーム側のレンダリングを最適化する。 |
| `og:image:alt` | `og:image`を指定する場合は必ず設定する。 |

多言語対応時は`og:locale:alternate`で代替言語を指定する。

```html
<meta property="og:locale:alternate" content="en_US">
```

### OGP画像仕様

| 項目 | 基準 |
| --- | --- |
| 推奨サイズ | 1200 x 630 px（アスペクト比 1.91:1） |
| 最小サイズ | 600 x 315 px |
| 形式 | JPG、PNG、WebP |
| ファイルサイズ | 5 MB以下 |
| URL | 絶対URL（HTTPS推奨） |

---

## X（Twitter）Cards

参照：[Getting started with Cards](https://developer.x.com/en/docs/x-for-websites/cards/guides/getting-started)、[Cards Markup Tag Reference](https://developer.x.com/en/docs/x-for-websites/cards/overview/markup)

### OGPフォールバック

X（旧Twitter）のカードプロセッサは、まずTwitter固有のメタタグを確認し、存在しない場合はOGPタグにフォールバックする。OGPタグを適切に設定していれば、X固有のメタタグは`twitter:card`のみで十分である。

```html
<meta name="twitter:card" content="summary_large_image">
```

### カード種別

| カード種別 | 値 | 用途 |
| --- | --- | --- |
| Summary | `summary` | 小さなサムネイル付きカード |
| Summary with Large Image | `summary_large_image` | 大きな画像付きカード（推奨） |

`summary_large_image`はクリック率が高いため、特別な理由がない限りこちらを使用する。

### 任意の追加タグ

OGPタグが設定済みの場合、以下はOGPからフォールバックされるため省略可能である。ただし、X向けに個別に制御したい場合は明示的に指定する。

```html
<meta name="twitter:site" content="@YOUR_X_ACCOUNT">
<meta name="twitter:creator" content="@YOUR_AUTHOR_X_ACCOUNT">
<meta name="twitter:title" content="YOUR_PAGE_TITLE">
<meta name="twitter:description" content="YOUR_PAGE_DESCRIPTION">
<meta name="twitter:image" content="https://example.com/images/card.jpg">
<meta name="twitter:image:alt" content="YOUR_IMAGE_ALT_TEXT">
```

| タグ | OGPフォールバック | 説明 |
| --- | --- | --- |
| `twitter:title` | `og:title` | カードのタイトル |
| `twitter:description` | `og:description` | カードの説明文 |
| `twitter:image` | `og:image` | カードの画像 |
| `twitter:image:alt` | `og:image:alt` | 画像の代替テキスト |
| `twitter:site` | なし | サイトのXアカウント |
| `twitter:creator` | なし | コンテンツ作成者のXアカウント（`summary_large_image`時のみ有効） |

### X Cards画像仕様

| カード種別 | 推奨サイズ | 最小サイズ | 最大サイズ | アスペクト比 |
| --- | --- | --- | --- | --- |
| `summary` | 240 x 240 px | 144 x 144 px | 4096 x 4096 px | 1:1 |
| `summary_large_image` | 1200 x 628 px | 300 x 157 px | 4096 x 4096 px | 1.91:1（2:1も可） |

対応形式はJPG、PNG、WebP、GIF（静止画として表示）。ファイルサイズは5 MB以下。

OGP画像仕様（1200 x 630 px）と`summary_large_image`の推奨サイズはほぼ同一であるため、1枚の画像を共用できる。

---

## sitemap.xml

参照：[Build and Submit a Sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap)

### 形式

XML形式を使用する。文字コードはUTF-8。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/</loc>
    <lastmod>2026-02-06</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://example.com/about/</loc>
    <lastmod>2026-01-15</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

### 制約

| 項目 | 上限 |
| --- | --- |
| 1ファイルあたりのURL数 | 50,000件 |
| 1ファイルのサイズ | 50 MB（非圧縮） |
| 超過時の対応 | サイトマップインデックスファイルで複数ファイルに分割する |

### 実装規約

| 項目 | 基準 |
| --- | --- |
| URL形式 | 絶対URL |
| 含めるページ | canonicalかつインデックス対象の200応答ページのみ |
| 除外するページ | noindexページ、リダイレクトページ、404ページ、重複ページ、パラメーター付きURL |
| `lastmod` | 実際の最終更新日を正確に記載する（虚偽の日付を設定しない） |
| robots.txtとの整合性 | sitemapに記載するURLがrobots.txtでブロックされていないことを確認する |
| 配置場所 | サイトのルートディレクトリ（`https://example.com/sitemap.xml`） |
| 提出先 | Google Search ConsoleおよびBing Webmaster Toolsに提出する |
| robots.txtへの記載 | robots.txt内に`Sitemap:`ディレクティブで場所を明示する |
| 更新頻度 | コンテンツの追加・削除時に更新する。CMSによる自動生成を推奨する。 |

### サイトマップインデックス

50,000件を超える場合、またはコンテンツタイプ別に管理する場合に使用する。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://example.com/sitemap-pages.xml</loc>
    <lastmod>2026-02-06</lastmod>
  </sitemap>
  <sitemap>
    <loc>https://example.com/sitemap-posts.xml</loc>
    <lastmod>2026-02-05</lastmod>
  </sitemap>
</sitemapindex>
```

---

## robots.txt

参照：[Create and Submit a robots.txt File](https://developers.google.com/crawling/docs/robots-txt/create-robots-txt)、[How Google Interprets the robots.txt Specification](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt)

### 配置

サイトのルートディレクトリに配置する（`https://example.com/robots.txt`）。プレーンテキスト、ASCIIエンコーディング。

### 構文

```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /private/

Sitemap: https://example.com/sitemap.xml
```

### ディレクティブ

| ディレクティブ | 説明 |
| --- | --- |
| `User-agent` | 対象のクローラーを指定する。`*`は全クローラー。 |
| `Disallow` | クロールを禁止するパスを指定する。 |
| `Allow` | `Disallow`を上書きしてクロールを許可するパスを指定する。 |
| `Sitemap` | サイトマップの絶対URLを指定する。`User-agent`に紐付かず全クローラーに適用される。 |

### 実装規約

| 項目 | 基準 |
| --- | --- |
| パス指定 | 大文字小文字を区別する |
| 競合時の挙動 | Googleは最も具体的な（パスが長い）ルールを適用する。同じ長さの場合は許可を優先する。 |
| ブロック対象 | 管理画面、内部検索結果、ステージング環境、重複コンテンツ生成パス |
| ブロック禁止対象 | CSS、JavaScript、画像（レンダリングに必要なリソースをブロックしない） |
| 開発環境からの移行 | 本番公開時に`Disallow: /`が残っていないことを必ず確認する |
| `noindex`との関係 | robots.txtでブロックしたページのメタタグは読み取られない。インデックス除外には`noindex`メタタグを使用し、robots.txtではブロックしない。 |
| テスト | 変更前にGoogle Search Consoleのrobots.txtテスターで検証する |

### AIクローラー

AI学習目的のクローラーをブロックする場合の例。

```
User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

User-agent: Google-Extended
Disallow: /

User-agent: CCBot
Disallow: /
```

ブロック対象のクローラーはサイトのポリシーに応じて決定する。Googlebotのブロックは検索結果からの除外を意味するため、`Google-Extended`（AI学習用）と区別する。

---

## 実装例

すべてのメタタグを統合した`<head>`テンプレート。

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <!-- 文字コード -->
  <meta charset="UTF-8">

  <!-- Viewport -->
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <!-- SEO基本 -->
  <title>YOUR_PAGE_TITLE | YOUR_SITE_NAME</title>
  <meta name="description" content="YOUR_PAGE_DESCRIPTION">
  <meta name="robots" content="index, follow">
  <link rel="canonical" href="https://example.com/page/">

  <!-- OGP -->
  <meta property="og:title" content="YOUR_PAGE_TITLE">
  <meta property="og:description" content="YOUR_SOCIAL_DESCRIPTION">
  <meta property="og:type" content="article">
  <meta property="og:url" content="https://example.com/page/">
  <meta property="og:image" content="https://example.com/images/og-image.jpg">
  <meta property="og:image:width" content="1200">
  <meta property="og:image:height" content="630">
  <meta property="og:image:alt" content="YOUR_IMAGE_ALT_TEXT">
  <meta property="og:site_name" content="YOUR_SITE_NAME">
  <meta property="og:locale" content="ja_JP">

  <!-- X Cards -->
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@YOUR_X_ACCOUNT">

  <!-- Apple Smart App Banner（該当アプリがある場合） -->
  <meta name="apple-itunes-app" content="app-id=YOUR_APP_ID, app-argument=https://example.com/page/">

  <!-- 構造化データ -->
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "Article",
    "headline": "YOUR_PAGE_TITLE",
    "description": "YOUR_PAGE_DESCRIPTION",
    "author": {
      "@type": "Person",
      "name": "YOUR_AUTHOR_NAME"
    },
    "publisher": {
      "@type": "Organization",
      "name": "YOUR_SITE_NAME",
      "logo": {
        "@type": "ImageObject",
        "url": "https://example.com/images/logo.png"
      }
    },
    "datePublished": "2026-02-06",
    "dateModified": "2026-02-06",
    "image": "https://example.com/images/og-image.jpg",
    "mainEntityOfPage": {
      "@type": "WebPage",
      "@id": "https://example.com/page/"
    }
  }
  </script>
</head>
```

---

## 検証ツール

| ツール | 用途 | URL |
| --- | --- | --- |
| Google Rich Results Test | 構造化データのリッチリザルト適格性検証 | https://search.google.com/test/rich-results |
| Schema Markup Validator | Schema.org構文検証 | https://validator.schema.org/ |
| Google Search Console | インデックス状況、エラー監視、robots.txtテスト | https://search.google.com/search-console |
| Facebook Sharing Debugger | OGPタグのプレビューとキャッシュクリア | https://developers.facebook.com/tools/debug/ |
| X Card Validator | X Cardsのプレビュー | https://cards-dev.twitter.com/validator |
| metatags.io | OGP・X Cards統合プレビュー | https://metatags.io/ |
| URL Inspection Tool | Googleによるページの認識状態確認 | Google Search Console内 |

---

## 出典・参考情報
### Google公式ドキュメント

- [Meta Tags and Attributes that Google Supports](https://developers.google.com/search/docs/crawling-indexing/special-tags) - Googleがサポートするメタタグの一覧
- [How to Write Meta Descriptions](https://developers.google.com/search/docs/appearance/snippet) - メタディスクリプションとスニペットの仕様
- [Robots Meta Tags Specifications](https://developers.google.com/search/docs/crawling-indexing/robots-meta-tag) - robotsメタタグとX-Robots-Tagの仕様
- [How to Specify a Canonical](https://developers.google.com/search/docs/crawling-indexing/consolidate-duplicate-urls) - canonical指定の方法
- [Google does not use the keywords meta tag](https://developers.google.com/search/blog/2009/09/google-does-not-use-keywords-meta-tag) - keywordsメタタグの非使用
- [Intro to Structured Data](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data) - 構造化データの概要
- [General Structured Data Guidelines](https://developers.google.com/search/docs/appearance/structured-data/sd-policies) - 構造化データのポリシーとガイドライン
- [Build and Submit a Sitemap](https://developers.google.com/search/docs/crawling-indexing/sitemaps/build-sitemap) - サイトマップの作成と送信
- [Create and Submit a robots.txt File](https://developers.google.com/crawling/docs/robots-txt/create-robots-txt) - robots.txtの作成
- [How Google Interprets the robots.txt Specification](https://developers.google.com/search/docs/crawling-indexing/robots/robots_txt) - Googleのrobots.txt解釈仕様

### ソーシャルメディア

- [The Open Graph protocol](https://ogp.me/) - OGP公式仕様
- [Getting started with Cards](https://developer.x.com/en/docs/x-for-websites/cards/guides/getting-started) - X Cards概要とOGPフォールバック仕様
- [Cards Markup Tag Reference](https://developer.x.com/en/docs/x-for-websites/cards/overview/markup) - X Cardsマークアップリファレンス
- [Summary Card with Large Image](https://developer.x.com/en/docs/x-for-websites/cards/overview/summary-card-with-large-image) - 大画像付きサマリーカードの仕様

### Apple

- [Promoting Apps with Smart App Banners](https://developer.apple.com/documentation/webkit/promoting-apps-with-smart-app-banners) - Smart App Banners公式ドキュメント

### Web標準

- [Viewport meta tag - MDN](https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag) - viewportメタタグの仕様
- [WCAG 2.2 Success Criterion 1.4.4](https://www.w3.org/TR/WCAG22/#resize-text) - テキストのサイズ変更に関するアクセシビリティ要件
- [Schema.org](https://schema.org/) - 構造化データのボキャブラリ定義
- [Sitemaps.org](https://www.sitemaps.org/) - サイトマッププロトコルの仕様
