---
title: "Performance Optimization Guidelines"
description: "パフォーマンス最適化 - Core Web Vitals・画像フォント最適化・バンドルサイズ・レンダリング / Performance optimization - Core Web Vitals, image/font optimization, bundle size, rendering"
version: "1.1.0"
status: "Stable"
last_updated: "2026-03-05T00:00+09:00"
lang: "ja"
---

# Performance Optimization Guidelines

**Web パフォーマンス最適化標準** - Core Web Vitals を基準とした実装レベルの最適化規約


---

## Core Web Vitals

Google が定義するユーザー体験の中核指標。検索ランキングに直接影響し、75パーセンタイル値で評価する（75%のページビューが Good 閾値を満たす必要がある）。

### 指標と閾値

| 指標 | Good | Needs Improvement | Poor | 測定対象 |
|---|---|---|---|---|
| **LCP**（Largest Contentful Paint） | ≤ 2.5秒 | 2.5秒〜4秒 | > 4秒 | 最大コンテンツ要素の描画時間 |
| **INP**（Interaction to Next Paint） | ≤ 200ms | 200ms〜500ms | > 500ms | ページ全体のインタラクション応答性 |
| **CLS**（Cumulative Layout Shift） | ≤ 0.1 | 0.1〜0.25 | > 0.25 | 予期しないレイアウトシフトの累積スコア |

INP は 2024年3月に FID（First Input Delay）を置き換えた指標。ページライフサイクル全体のインタラクション応答性を測定する。

### SEO・ビジネスへの影響

全3指標達成で検索可視性が 8〜15% 向上する（Google Search Central 実績値）。Poor から Good への改善でコンバージョン率 12〜15% 向上、直帰率 18% 減少の事例が報告されている。競合クエリではランキングウェイトとして 25〜30% を占める。

出典・参考情報: Google Search Central、web.dev Core Web Vitals

---

## LCP（Largest Contentful Paint）最適化

### 対象要素の特定

LCP 候補になりうる要素は次のとおり。

- `<img>` 要素
- `<image>` 要素内の SVG
- `background-image` が設定された要素
- テキストを含むブロックレベル要素（`<p>`、`<h1>` 等）

ヒーロー画像（Above the fold のメイン画像）が LCP 候補になる場合が最も多い。

### `fetchpriority="high"`（必須）

LCP 候補画像への適用を必須とする。ブラウザのリソース優先度ヒューリスティックを上書きし、即座に高優先度でフェッチさせる。Etsy の実績では 4% の LCP 改善、サイトによっては 20〜30% 改善が報告されている。

```html
<!-- LCP 候補画像への適用例 -->
<img
  src="hero.avif"
  alt="ヒーロー画像"
  width="1200"
  height="630"
  fetchpriority="high"
>
```

**禁止事項**：LCP 画像に `loading="lazy"` を適用しない。必ず遅延が発生し LCP が悪化する。

### Preload による先読み

HTML パーサーが画像を発見する前に、ブラウザへ取得を指示する。`fetchpriority` と組み合わせて使用する。

```html
<!-- 静的画像の preload -->
<link
  rel="preload"
  href="hero.avif"
  as="image"
  fetchpriority="high"
>

<!-- レスポンシブ画像の preload（imagesrcset / imagesizes 必須） -->
<link
  rel="preload"
  href="hero-800.avif"
  as="image"
  imagesrcset="hero-400.avif 400w, hero-800.avif 800w, hero-1200.avif 1200w"
  imagesizes="(max-width: 600px) 100vw, 50vw"
  fetchpriority="high"
>
```

CDN からクロスオリジンで配信する場合は `crossorigin="anonymous"` を追加する。

### 画像フォーマットの優先順位

1. **AVIF**：WebP より高圧縮率。第一候補。
2. **WebP**：AVIF 非対応環境へのフォールバック。
3. **JPEG / PNG**：最終フォールバック。

`<picture>` 要素でフォールバックを提供する。

```html
<picture>
  <source type="image/avif" srcset="hero.avif">
  <source type="image/webp" srcset="hero.webp">
  <img
    src="hero.jpg"
    alt="ヒーロー画像"
    width="1200"
    height="630"
    fetchpriority="high"
  >
</picture>
```

### TTFB（Time to First Byte）との関係

LCP はサーバーの応答速度に依存する。TTFB が高い場合、2.5秒 LCP の達成は困難または不可能になる。CDN 利用、サーバーサイドキャッシュ、リダイレクト削減で TTFB を削減する。

### JavaScript 管理コンテンツの回避

LCP 要素を JavaScript で生成・挿入すると Preload Scanner が検出できず、必ず遅延が発生する。LCP 候補要素は HTML に静的に記述する。

出典・参考情報: web.dev LCP、Google Developers fetchpriority

---

## INP（Interaction to Next Paint）最適化

### INP の構成要素

INP は次の3フェーズの合計で決まる。

1. **Input Delay**：ユーザー操作からイベントハンドラーが実行されるまでの待機時間。メインスレッドが長タスクで占有されている間に操作が発生すると増加する。
2. **Processing Time**：イベントハンドラーの実行時間。
3. **Presentation Delay**：ハンドラー完了後、視覚的な更新がレンダリングされるまでの時間。

長タスク（50ms を超えるメインスレッド占有）が INP 悪化の主因。

### `scheduler.yield()` による長タスク分割

Chrome の標準 API。`setTimeout()` と異なり、継続タスクをキューの先頭に配置するため、ユーザー入力に対する応答性を維持しながら処理を分割できる。

```javascript
// 大規模データ処理でメインスレッドを占有しない実装例
async function processLargeDataset(data) {
  const results = [];
  for (let i = 0; i < data.length; i++) {
    results.push(expensiveOperation(data[i]));
    // 10件ごとにメインスレッドを解放
    if (i % 10 === 0) {
      await scheduler.yield();
    }
  }
  return results;
}
```

クロスブラウザ対応が必要な場合はポリフィルを使用する。

```javascript
// scheduler.yield() ポリフィル
const yieldToMain = () => {
  if ('scheduler' in window && 'yield' in scheduler) {
    return scheduler.yield();
  }
  return new Promise(resolve => setTimeout(resolve, 0));
};
```

### Web Workers による重計算の分離

画像処理、データ変換、暗号化などの CPU 負荷の高い処理はメインスレッドから Web Worker に移行する。

```javascript
// メインスレッド
const worker = new Worker('heavy-compute.js');
worker.postMessage({ data: largeDataset });
worker.onmessage = (event) => {
  updateUI(event.data.result);
};

// heavy-compute.js（Worker 内）
self.onmessage = (event) => {
  const result = performHeavyComputation(event.data.data);
  self.postMessage({ result });
};
```

### イベントハンドラーの最適化

```javascript
// 禁止パターン：ハンドラー内で重い処理を同期実行
button.addEventListener('click', () => {
  const result = heavyComputation(); // メインスレッドをブロック
  updateDOM(result);
});

// 推奨パターン：最小限の処理のみ実行し、重い処理は分離
button.addEventListener('click', async () => {
  showLoadingState();
  await scheduler.yield();
  const result = await processAsync();
  updateDOM(result);
});
```

### サードパーティスクリプトの遅延読み込み

チャット Widget、広告タグ、アナリティクスは初期化を遅延させる。Next.js の Script コンポーネントを使用する場合は `strategy="afterInteractive"` または `strategy="lazyOnload"` を指定する。

```html
<!-- vanilla HTML での遅延読み込み -->
<script>
  window.addEventListener('load', () => {
    const script = document.createElement('script');
    script.src = 'https://third-party.example.com/widget.js';
    document.body.appendChild(script);
  });
</script>
```

出典・参考情報: web.dev INP、Chrome Developers Scheduler API、MDN Web Workers

---

## CLS（Cumulative Layout Shift）最適化

### 画像・動画への `width` / `height` 属性指定（最重要）

CLS 問題の 60% 以上を解決する最も効果的な対策。`width` と `height` 属性を明示することで、ブラウザが CSS 読み込み前にアスペクト比を計算し、スペースを確保できる。

```html
<!-- 必須：width / height 属性を明示する -->
<img src="photo.jpg" alt="写真" width="800" height="600">
<video src="video.mp4" width="1280" height="720"></video>

<!-- レスポンシブ対応は CSS で実施 -->
```

```css
/* レスポンシブ対応：CSS で max-width を指定 */
img,
video {
  max-width: 100%;
  height: auto;
}
```

### CSS `aspect-ratio` による動的コンテンツの領域確保

動的にサイズが決まる要素（埋め込みコンテンツ、API から取得する画像等）には `aspect-ratio` で領域を事前確保する。

```css
/* 動画・iframe のアスペクト比確保 */
.video-wrapper {
  aspect-ratio: 16 / 9;
  width: 100%;
}

/* 正方形サムネイル */
.thumbnail {
  aspect-ratio: 1;
  width: 100%;
  object-fit: cover;
}
```

### フォント置き換えによる CLS の防止

`font-display: swap` 単体では、カスタムフォント到着時のフォールバックとの置き換えで CLS が発生する。`size-adjust`、`ascent-override`、`descent-override` でフォールバックフォントのメトリクスをカスタムフォントに合わせることで CLS を最大 70% 削減できる。

```css
/* フォールバックフォントのメトリクス調整 */
@font-face {
  font-family: 'FallbackFont';
  src: local('Arial');
  ascent-override: 90%;
  descent-override: 22%;
  line-gap-override: 0%;
  size-adjust: 107%;
}

/* カスタムフォント定義 */
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2');
  font-display: swap;
}

body {
  font-family: 'CustomFont', 'FallbackFont', sans-serif;
}
```

### 広告・動的コンテンツの領域確保

広告スロットや API から取得するコンテンツには、最小サイズのプレースホルダーを設ける。

```css
/* 広告スロットのプレースホルダー */
.ad-slot {
  min-height: 250px;
  width: 300px;
  contain: layout; /* レイアウト境界を作成し、CLS の影響範囲を限定 */
}
```

### アニメーションによる CLS の回避

レイアウトを変化させるプロパティ（`width`、`height`、`top`、`left`、`margin`、`padding`）のアニメーションは CLS を引き起こす。`transform` と `opacity` のみを使用する。

```css
/* 禁止パターン：レイアウトに影響するプロパティのアニメーション */
.element {
  transition: width 0.3s ease; /* CLS 発生 */
}

/* 推奨パターン：transform のみ使用 */
.element {
  transition: transform 0.3s ease;
}
.element:hover {
  transform: scale(1.05); /* レイアウト再計算なし */
}
```

出典・参考情報: web.dev CLS、MDN aspect-ratio、web.dev font-display

---

## フォント最適化

### WOFF2 のみ使用

WOFF2 は Brotli 圧縮を使用し、WOFF より 30% 小さい。主要ブラウザの 97% が対応している。WOFF、EOT、SVG フォント、TTF の直接配信は行わない。

```css
@font-face {
  font-family: 'CustomFont';
  src: url('font.woff2') format('woff2'); /* WOFF2 のみ記載 */
  font-weight: 400;
  font-style: normal;
  font-display: swap;
}
```

### フォントの Preload

重要なフォント（本文、見出し用）を 1〜2 個に限定して preload する。過剰な preload は他のリソースのフェッチを妨げる。`crossorigin` 属性は CORS 要求のため必須。

```html
<!-- フォントの preload（同一オリジン配信） -->
<link
  rel="preload"
  href="/fonts/custom-regular.woff2"
  as="font"
  type="font/woff2"
  crossorigin
>
```

外部フォントサービス（Google Fonts 等）を使用する場合は `preconnect` で DNS・TCP・TLS 接続を事前確立する。

```html
<!-- Google Fonts の preconnect -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
```

### `font-display` 戦略の選択基準

| 値 | 動作 | 適用場面 |
|---|---|---|
| `swap` | 即座にフォールバック表示 → カスタムフォント到着時に置換 | 本文テキスト（CLS 対策として `size-adjust` 併用必須） |
| `optional` | 100ms 以内にダウンロード完了しない場合はフォールバックを使用 | CLS を完全に回避したい場合（初回訪問でカスタムフォントが表示されない可能性がある） |
| `fallback` | 短いブロック期間後にフォールバック表示。スワップ期間は 3秒 | バランス重視の場合 |
| `block` | フォント読み込みまで不可視テキスト（FOIT） | アイコンフォント等、フォールバックでは機能しない場合のみ |

### フォントのサブセット化

日本語フォントはファイルサイズが 10MB 以上になることがある。使用文字のみを抽出するサブセット化を実施する。

```css
/* unicode-range による分割読み込み */
@font-face {
  font-family: 'JapaneseFont';
  src: url('japanese-hiragana.woff2') format('woff2');
  unicode-range: U+3040-309F; /* ひらがな */
}

@font-face {
  font-family: 'JapaneseFont';
  src: url('japanese-katakana.woff2') format('woff2');
  unicode-range: U+30A0-30FF; /* カタカナ */
}

@font-face {
  font-family: 'JapaneseFont';
  src: url('japanese-kanji.woff2') format('woff2');
  unicode-range: U+4E00-9FFF; /* CJK 統合漢字 */
}
```

### Critical CSS 内でのフォント宣言

外部スタイルシートの `@font-face` 宣言は、スタイルシートのダウンロード完了まで待機が発生する。Critical CSS をインライン化する際は `@font-face` も含める。

```html
<head>
  <style>
    /* Critical CSS 内に @font-face を含める */
    @font-face {
      font-family: 'CustomFont';
      src: url('/fonts/custom.woff2') format('woff2');
      font-display: swap;
    }
    /* Above-the-fold の Critical CSS */
    body { font-family: 'CustomFont', sans-serif; }
    h1 { font-size: 2rem; }
  </style>
  <link rel="stylesheet" href="main.css" media="print" onload="this.media='all'">
</head>
```

出典・参考情報: web.dev font-best-practices、MDN font-display、Google Fonts optimization

---

## JavaScript バンドルサイズ最適化

### Code Splitting（動的 import）

ルートごとにバンドルを分割し、初期読み込みサイズを削減する。

```javascript
// React での動的 import（React.lazy）
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

大型コンポーネント（チャートライブラリ、リッチテキストエディター等）も動的 import で分割する。

```javascript
// コンポーネントレベルの分割（インタラクション時に読み込む）
const loadChart = async () => {
  const { Chart } = await import('./Chart');
  return Chart;
};
```

### Tree Shaking

Dead code を除去するために ES6 モジュール（`import` / `export`）を使用する。CommonJS（`require` / `module.exports`）では Tree Shaking が機能しない。

```javascript
// 禁止パターン：ライブラリ全体を import
import _ from 'lodash'; // lodash 全体（約 70KB gzip）
const debounced = _.debounce(fn, 300);

// 推奨パターン：名前付き import で必要な関数のみ取得
import { debounce } from 'lodash-es'; // 必要な関数のみ（数KB）
const debounced = debounce(fn, 300);
```

`package.json` に `"sideEffects": false` を宣言することで Webpack の Tree Shaking が高速化する。

```json
{
  "name": "my-library",
  "sideEffects": false
}
```

### ベンダーバンドルの分離

React、Vue 等のフレームワークと、Lodash 等のユーティリティライブラリを独自バンドルに分離する。これらは更新頻度が低いため、長期キャッシュが効く。

```javascript
// Webpack での splitChunks 設定例
module.exports = {
  optimization: {
    splitChunks: {
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        react: {
          test: /[\\/]node_modules[\\/](react|react-dom)[\\/]/,
          name: 'react',
          chunks: 'all',
        },
      },
    },
  },
};
```

### バンドルサイズ制限

Webpack のパフォーマンス警告を設定し、肥大化を検知する。

```javascript
module.exports = {
  performance: {
    maxAssetSize: 244 * 1024,       // 244 KiB
    maxEntrypointSize: 244 * 1024,  // 244 KiB
    hints: 'warning',               // 超過時に警告
  },
};
```

### 軽量代替ライブラリへの移行

| 重いライブラリ | 軽量代替 | 削減効果の目安 |
|---|---|---|
| `moment.js`（~67KB gzip） | `date-fns`（必要関数のみ）/ `dayjs`（~2KB gzip） | 60KB 以上削減 |
| `lodash`（~70KB gzip） | `lodash-es`（Tree Shaking 対応）/ ネイティブ JS | 大幅削減 |
| `jQuery`（~30KB gzip） | ネイティブ DOM API（[coding-standards.md](coding-standards.md) では全面禁止） | 30KB 削減 |

### 分析ツール

- **Webpack Bundle Analyzer**：バンドル内の各モジュールのサイズを視覚化。
- **Lighthouse**：Chrome DevTools の Lighthouse タブで unused JavaScript を検出。
- **DebugBear**：RUM（Real User Monitoring）でスクリプトごとの実際の遅延を測定。

出典・参考情報: web.dev code-splitting、Webpack Documentation、web.dev tree-shaking

---

## Critical Rendering Path 最適化

### CRP（Critical Rendering Path）の構成

HTML 解析 → DOM 構築 → CSS 解析 → CSSOM 構築 → Render Tree 構築 → Layout → Paint → Composite

CSS と `defer`/`async` なしの JavaScript はレンダリングブロックリソースとなり、CRP を遅延させる。

### CSS の最適化

**Critical CSS のインライン化**：Above-the-fold（初期表示領域）に必要な CSS を `<head>` 内の `<style>` タグにインライン化する。残りの CSS は非同期で読み込む。

```html
<head>
  <style>
    /* Critical CSS：初期表示に必要な最小限のスタイル */
    body { margin: 0; font-family: sans-serif; }
    header { background: #fff; height: 60px; }
    h1 { font-size: 2rem; }
  </style>
  <!-- 残りの CSS を非同期読み込み -->
  <link
    rel="stylesheet"
    href="main.css"
    media="print"
    onload="this.media='all'"
  >
  <noscript><link rel="stylesheet" href="main.css"></noscript>
</head>
```

**Media Queries による条件付き非ブロック化**：印刷用 CSS や特定画面幅専用の CSS には `media` 属性を付与し、レンダリングブロックを回避する。

```html
<link rel="stylesheet" href="print.css" media="print">
<link rel="stylesheet" href="tablet.css" media="(min-width: 768px)">
```

### JavaScript の最適化

CLAUDE.md 規約により、全 `<script>` タグは `<head>` 内に記載し、`defer` または `async` を必須とする。

| 属性 | 実行タイミング | 実行順序 | 適用場面 |
|---|---|---|---|
| `defer` | DOM 構築完了後（DOMContentLoaded 前） | 記述順を保持 | DOM を参照するスクリプト（推奨） |
| `async` | ダウンロード完了次第 | 保証なし | 独立したスクリプト（アナリティクス等） |
| なし（`<head>` 内） | 即座に実行（解析ブロック） | — | 絶対禁止 |

```html
<head>
  <!-- DOM を参照するメインスクリプト -->
  <script defer src="app.js"></script>
  <!-- 独立したサードパーティスクリプト -->
  <script async src="https://analytics.example.com/track.js"></script>
</head>
```

### Preload Scanner との連携

Preload Scanner はブラウザの二次 HTML パーサー。メインスレッドがレンダリングブロック中でも、先行して HTML を解析し重要リソースを発見・取得する。ただし、HTML に静的に記述されたリソースのみ検出可能。JavaScript で動的生成されたリソース（`document.createElement('img')` 等）は検出できない。

**重要リソースは HTML 内に静的に記述し、Preload Scanner に確実に検出させる。**

### リソースヒントの使い分け

```html
<!-- preload：現ページで確実に使うリソースを優先取得 -->
<link rel="preload" href="critical-font.woff2" as="font" crossorigin>

<!-- preconnect：外部ドメインへの接続を事前確立（DNS + TCP + TLS） -->
<link rel="preconnect" href="https://api.example.com">

<!-- dns-prefetch：DNS 解決のみ事前実施（preconnect の軽量版） -->
<link rel="dns-prefetch" href="https://cdn.example.com">

<!-- prefetch：次ページで使うリソースを低優先度でキャッシュ -->
<link rel="prefetch" href="next-page.js">
```

`preload` は使用されないと Console に警告が出る。実際に使用するリソースのみ preload する。

### Compression（圧縮）

サーバー側で Brotli または Gzip 圧縮を必ず有効にする。テキストリソース（HTML、CSS、JavaScript、JSON、SVG）に適用する。

| 圧縮方式 | 圧縮率 | ブラウザサポート | 推奨 |
|---|---|---|---|
| Brotli | GZIP より 15〜25% 優れる | 主要ブラウザ全対応 | 第一候補 |
| Gzip | 標準的 | 全ブラウザ | Brotli 非対応時のフォールバック |

出典・参考情報: MDN Critical Rendering Path、web.dev render-blocking-resources、MDN rel=preload

---

## 画像・メディア最適化

フォーマット選定基準と詳細な実装規約は [media-assets-guidelines.md](media-assets-guidelines.md) を参照。本章は CWV（Core Web Vitals）に直結する最適化のみ記載する。

### Lazy Loading の適用基準

Above-the-fold 外の画像に `loading="lazy"` を適用する。LCP 候補画像への適用は禁止。

```html
<!-- Above-the-fold の画像：lazy 禁止、fetchpriority 必須 -->
<img src="hero.avif" alt="..." width="1200" height="630" fetchpriority="high">

<!-- Above-the-fold 外の画像：lazy 推奨 -->
<img src="thumbnail.avif" alt="..." width="300" height="200" loading="lazy">
```

### 画像サイズの最適化

ビューポートに表示されるサイズより大きな画像を配信しない。`srcset` と `sizes` でデバイスに適したサイズを提供する。

```html
<img
  src="image-800.avif"
  srcset="
    image-400.avif  400w,
    image-800.avif  800w,
    image-1200.avif 1200w
  "
  sizes="(max-width: 600px) 100vw, (max-width: 1200px) 50vw, 800px"
  alt="説明"
  width="800"
  height="600"
  loading="lazy"
>
```

出典・参考情報: web.dev images、MDN lazy loading

### SVG の配信方式とパフォーマンス

#### インライン `<svg>`

HTTP リクエストが 0 になり FCP/LCP に直結するため、Above-the-fold のロゴや重要アイコンに有効。

**アンチパターン（逆効果）**
- 複数ページで共有するアイコンをインライン化する → ページごとに SVG が送信されブラウザキャッシュが効かなくなる。外部ファイルまたはスプライトに移行する。
- Below-the-fold のアイコンをインライン化する → HTML サイズが増加するのみで FCP/LCP への寄与がない。

#### SVG スプライト（外部 `.svg` + `<use>`）

全アイコンを 1 リクエストで取得しブラウザキャッシュが効くため、繰り返し使うアイコンセットに最適。

**アンチパターン（逆効果）**
- スプライトファイルを過剰に肥大化させる → 使用しないアイコンも毎回ダウンロードされる。実際に使用するアイコンのみを含む。
- LCP 候補になりうる Above-the-fold のロゴをスプライトで参照する → スプライトファイルのダウンロード完了まで描画が遅延する。インライン `<svg>` に切り替える。

#### `<img src="icon.svg">`

Below-the-fold の単体アイコンには `loading="lazy"` を適用できる。装飾目的の SVG に適した方式。

**アンチパターン（逆効果）**
- `loading="lazy"` を付けずに多数の `<img src="*.svg">` を並べる → HTTP/2 環境でもリクエスト数が増加し他のリソース取得を圧迫する。スプライトへ統合する。
- CSS や JavaScript でスタイル・状態制御が必要なアイコンに使用する → `<img>` 内の SVG は外部 CSS・JS から操作できない。インライン `<svg>` に切り替える。

#### CSS Data URI（URLエンコード）

リクエストゼロで CSS と一体配信できるが、[media-assets-guidelines.md](media-assets-guidelines.md) の規定により 1KB 以下の装飾アイコンのみ使用可。

**アンチパターン（逆効果）**
- 1KB を超える SVG を Data URI に変換する → Base64 エンコードでサイズが約 33% 増加し、Brotli 圧縮の恩恵が得られにくくなる。
- 複数コンポーネントで共有するアイコンを Data URI にする → CSS ファイルに埋め込まれるためアイコン単体のキャッシュ更新が不可能になる。
- Base64 エンコードで埋め込む → URLエンコード方式より大きくなり可読性も失われる。SVG は必ず URLエンコードで埋め込む。

HTTP/2 環境ではリクエストオーバーヘッドがほぼゼロになるため、Data URI とインライン化の優位性は Above-the-fold の Critical Path に限定される。インライン SVG・スプライトの実装詳細は [media-assets-guidelines.md](media-assets-guidelines.md) を参照。

---

## 測定・監視

### ツール一覧

| ツール | 種別 | 用途 |
|---|---|---|
| [Google Search Console](https://search.google.com/search-console) | Field（実ユーザーデータ） | CrUX 28日間データ、75パーセンタイル評価、SEO への影響確認 |
| [PageSpeed Insights](https://pagespeed.web.dev/) | Lab + Field | Lighthouse スコアと CrUX データの統合確認 |
| Chrome DevTools Performance | Lab | フレームごとの詳細分析、Long Task の特定 |
| Chrome DevTools Lighthouse | Lab | Core Web Vitals、unused JavaScript 検出 |
| [WebPageTest](https://www.webpagetest.org/) | Lab | TTFB、完全読み込み時間、Waterfall 詳細 |
| [DebugBear](https://www.debugbear.com/) | Lab + Field | INP の構成要素分析、スクリプトごとの遅延可視化 |

### Lighthouse CI による回帰防止

CI/CD パイプラインに Lighthouse CI を組み込み、デプロイごとにパフォーマンス指標を検証する。

```yaml
# .lighthouserc.js
module.exports = {
  ci: {
    assert: {
      assertions: {
        'categories:performance': ['warn', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
  },
};
```

### Long Animation Frames API

INP 悪化の原因となるスクリプトを特定する。

```javascript
// LoAF（Long Animation Frames）の観測
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.duration > 50) {
      console.warn('Long animation frame detected:', {
        duration: entry.duration,
        scripts: entry.scripts.map(s => ({
          src: s.sourceURL,
          duration: s.duration,
        })),
      });
    }
  }
});
observer.observe({ type: 'long-animation-frame', buffered: true });
```

出典・参考情報: web.dev measure、Chrome Developers Long Animation Frames API

---

## 実装優先順位

パフォーマンス改善の効果とコストを考慮した推奨実装順序。

1. **LCP 画像に `fetchpriority="high"` 適用**（即効性高・実装容易）
2. **全画像・動画に `width` / `height` 属性追加**（CLS 対策・実装容易）
3. **`<script>` への `defer` 適用**（レンダリングブロック解消・実装容易）
4. **フォント preload**（1〜2 フォントのみ・実装容易）
5. **AVIF / WebP への画像フォーマット変換**（LCP・PageWeight 改善）
6. **ルートベース Code Splitting 導入**（初期 JS サイズ削減）
7. **`scheduler.yield()` による長タスク分割**（INP 改善）
8. **Critical CSS 抽出・インライン化**（LCP・FCP 改善・実装複雑）
9. **フォント `size-adjust` によるメトリクス調整**（CLS 削減・高度）
10. **Lighthouse CI のパイプライン組み込み**（回帰防止）

---

## 出典・参考情報

### 情報・規格

- [web.dev Core Web Vitals](https://web.dev/articles/vitals) - LCP/INP/CLS の定義と測定基準
- [web.dev LCP](https://web.dev/articles/lcp) - Largest Contentful Paint 最適化ガイド
- [web.dev INP](https://web.dev/articles/inp) - Interaction to Next Paint 最適化ガイド
- [web.dev CLS](https://web.dev/articles/cls) - Cumulative Layout Shift 最適化ガイド
- [web.dev font-best-practices](https://web.dev/articles/font-best-practices) - フォント最適化
- [web.dev code-splitting](https://web.dev/articles/code-splitting-suspense) - Code Splitting
- [web.dev render-blocking-resources](https://web.dev/articles/render-blocking-resources) - レンダリングブロックリソース
- [MDN Critical Rendering Path](https://developer.mozilla.org/en-US/docs/Web/Performance/Critical_rendering_path) - CRP の概念
- [MDN rel=preload](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/rel/preload) - Preload の仕様
- [MDN font-display](https://developer.mozilla.org/en-US/docs/Web/CSS/@font-face/font-display) - font-display の値と動作
- [Google Search Central - Core Web Vitals](https://developers.google.com/search/docs/appearance/core-web-vitals) - SEO への影響
- [Chrome Developers fetchpriority](https://developer.chrome.com/docs/web-platform/fetchpriority) - fetchpriority の詳細

### ツール公式ドキュメント

- [PageSpeed Insights](https://pagespeed.web.dev/) - パフォーマンス計測
- [WebPageTest](https://www.webpagetest.org/) - 詳細なパフォーマンス分析
- [DebugBear](https://www.debugbear.com/) - RUM + Lab テスト
- [Webpack Documentation](https://webpack.js.org/guides/code-splitting/) - Code Splitting 設定
- [Lighthouse CI](https://github.com/GoogleChrome/lighthouse-ci) - CI 組み込み用ツール
