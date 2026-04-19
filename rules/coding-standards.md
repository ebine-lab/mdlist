---
title: "HTML/CSS/Performance Coding Standards"
description: "HTML・CSS・パフォーマンスのコーディング規約 / HTML, CSS, and performance coding standards"
version: "1.2.0"
status: "Stable"
last_updated: "2026-03-28T23:09+09:00"
lang: "ja"
---

# HTML/CSS/Performance Coding Standards

**HTML・CSS・パフォーマンスの実装標準** - Web開発における技術規約を定義


---

## 共通

- レガシーコード不可（1年以上前のバージョンは参考にしない）
- セキュアなコードを生成（詳細は [secure-code-requirements.md](secure-code-requirements.md) を参照）
- フレームワーク・ライブラリは最新版または最新LTSを使用
- 勝手に古いバージョンに固定・ダウングレードしない
- LTSがある場合は最新版と比較・提案し、最終決定はTEDが行う
- **外部依存関係は必ず最小限に抑える（追加時はTEDに事前確認必須）**
- モジュール化されたファイル構造（HTML、CSS、JavaScriptの分離）は必須
- サンプルコードのコメントに句読点を入れない
- 1行で済むコメントを複数行で書かない
- 正当な括弧での注釈は許容する

---

## 禁止事項

- jQueryは使用禁止（フロントエンド・サーバサイド共）
- ExcelでのVBA使用不可

---

## ハードコーディングの禁止

ソースコード・スタイルシート・マークアップのいずれにおいても、意味の不明な定数値・環境依存の値・将来変更される可能性のある値を直接記述することを禁止する。全ての値は命名された定数、CSS Custom Properties（デザイントークン）、設定ファイル、または環境変数を経由して参照しなければならない。

### CSS におけるハードコーディング禁止

#### 色値の直書き禁止

HEX・RGB・RGBA・HSL・OKLCH のリテラル値をプロパティ値に直接記述してはならない。必ず CSS Custom Properties（デザイントークン）経由で参照する。

```css
/* ✅ 必須 */
.card {
  background-color: var(--color-surface-default);
  color: var(--color-text-primary);
  border: 1px solid var(--color-border-default);
}

/* ❌ 禁止 */
.card {
  background-color: #ffffff;
  color: rgb(51, 51, 51);
  border: 1px solid oklch(0.8 0.02 250);
}
```

色値のリテラルが許容されるのは、デザイントークンの定義ファイル（`:root` や `[data-theme]` でプリミティブトークンを宣言する箇所）のみとする。

#### サイズ・間隔の直書き禁止

`px`・`em`・`rem` のリテラル値をレイアウト・余白・サイズのプロパティに直接記述してはならない。デザイントークンまたはプロジェクト定義の CSS Custom Properties を使用する。

```css
/* ✅ 必須 */
.container {
  padding: var(--spacing-4) var(--spacing-6);
  margin-bottom: var(--spacing-8);
  gap: var(--spacing-2);
  max-width: var(--content-max-width);
}

/* ❌ 禁止 */
.container {
  padding: 16px 24px;
  margin-bottom: 32px;
  gap: 8px;
  max-width: 1200px;
}
```

`font-size` は rem 単位かつデザイントークン経由を必須とする（本文書 CSS セクションの既定規約を参照）。

**例外**: `0`、`100%`、`1px`（ボーダー線幅として文脈上自明な場合）は許容する。ただし同一値が3箇所以上で使用される場合は Custom Property 化を必須とする。

#### z-index の直書き禁止

`z-index` に数値リテラルを直接記述してはならない。プロジェクト共通の z-index スケールを CSS Custom Properties で定義し参照する。

```css
/* ✅ 必須 */
:root {
  --z-index-dropdown: 100;
  --z-index-sticky: 200;
  --z-index-modal-backdrop: 300;
  --z-index-modal: 400;
  --z-index-toast: 500;
}
.modal-backdrop { z-index: var(--z-index-modal-backdrop); }
.modal { z-index: var(--z-index-modal); }
.toast { z-index: var(--z-index-toast); }

/* ❌ 禁止 */
.modal-backdrop { z-index: 999; }
.modal { z-index: 1000; }
.toast { z-index: 1100; }
```

#### ブレークポイントの直書き禁止

メディアクエリにピクセル値を直接記述してはならない。[css-breakpoints-guidelines.md](css-breakpoints-guidelines.md) で定義されたカスタムプロパティまたは定数を使用する。

```css
/* ✅ 必須: CSS Custom Media（postcss-custom-media 等で使用） */
@custom-media --viewport-md (width >= 768px);
@media (--viewport-md) { ... }

/* ✅ 必須: container query の閾値も定数化 */
@container sidebar (min-width: 400px) { ... }
/* → 閾値は設計仕様書で定義された値のみ使用し、任意の数値を書かない */

/* ❌ 禁止 */
@media (min-width: 768px) { ... }
@media (min-width: 1024px) { ... }
```

#### アニメーション・トランジション値の直書き禁止

`duration`・`timing-function`・`delay` にリテラル値を直接記述してはならない。

```css
/* ✅ 必須 */
.fade-in {
  transition: opacity var(--duration-normal) var(--easing-standard);
}

/* ❌ 禁止 */
.fade-in {
  transition: opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1);
}
```

#### 影・角丸の直書き禁止

`box-shadow`・`border-radius` にリテラル値を直接記述してはならない。

```css
/* ✅ 必須 */
.card {
  box-shadow: var(--shadow-md);
  border-radius: var(--radius-md);
}

/* ❌ 禁止 */
.card {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
}
```

#### フォントファミリーの直書き禁止

`font-family` にフォント名を直接記述してはならない。トークンを定義して参照する。

```css
/* ✅ 必須 */
body {
  font-family: var(--font-family-base);
}

/* ❌ 禁止 */
body {
  font-family: 'Inter', 'Noto Sans JP', sans-serif;
}
```

### HTML におけるハードコーディング禁止

#### `<table>` / `<tr>` / `<th>` / `<td>` 要素のハードコーディング禁止

テーブル関連要素はプレゼンテーション属性が多数残存しており、ハードコーディングの温床になりやすい。以下の属性を要素ごとに厳密に禁止する。

**`<table>` の禁止属性**

| 禁止属性 | 対処 |
|---|---|
| `border` | CSS: `border` プロパティ + デザイントークン |
| `cellpadding` | CSS: `th, td { padding: var(--spacing-*); }` |
| `cellspacing` | CSS: `border-spacing` + デザイントークン、または `border-collapse: collapse` |
| `width` / `height` | CSS: `width` / `max-width` + デザイントークン |
| `bgcolor` | CSS: `background-color` + デザイントークン |
| `align` | CSS: `margin-inline: auto` 等のレイアウト制御 |
| `rules` | CSS: `border` の個別制御 |
| `frame` | CSS: `border` の個別制御 |
| `summary` | `<caption>` 要素を使用（`summary` は HTML5 で廃止） |

**`<tr>` の禁止属性**

| 禁止属性 | 対処 |
|---|---|
| `bgcolor` | CSS: `background-color` + デザイントークン |
| `align` | CSS: `text-align` + デザイントークン |
| `valign` | CSS: `vertical-align` |
| `height` | CSS: `height` / `min-height` + デザイントークン |

ストライプ行の背景色は `:nth-child(even)` / `:nth-child(odd)` + デザイントークンで制御する。

**`<th>` の禁止属性**

| 禁止属性 | 対処 |
|---|---|
| `bgcolor` | CSS: `background-color` + デザイントークン |
| `align` | CSS: `text-align` |
| `valign` | CSS: `vertical-align` |
| `width` / `height` | CSS: `width` / `min-width` / `max-width` + デザイントークン |
| `nowrap` | CSS: `white-space: nowrap` |
| `abbr`（属性値のハードコード） | 正しい略語を `abbr` 属性に記述すること自体は許容するが、表示テキストと乖離しないよう管理する |

**`<td>` の禁止属性**

| 禁止属性 | 対処 |
|---|---|
| `bgcolor` | CSS: `background-color` + デザイントークン |
| `align` | CSS: `text-align` |
| `valign` | CSS: `vertical-align` |
| `width` / `height` | CSS: `width` / `min-width` / `max-width` + デザイントークン |
| `nowrap` | CSS: `white-space: nowrap` |

**`<th>` / `<td>` 共通: `colspan` / `rowspan` の注意**

`colspan` / `rowspan` は HTML 仕様上の構造属性であり、CSS では代替できないため使用を許容する。ただし、値が動的に変わる場合はリテラル値ではなく変数・定数で管理する。

```html
<!-- ✅ 許容: 構造属性としての使用 -->
<td colspan="3">合計</td>

<!-- ✅ 推奨: 動的な場合は定数化 -->
<!-- JS側: const SUMMARY_COLSPAN = columnCount; -->
```

**CSS でのテーブルスタイリングにおけるハードコーディング禁止**

テーブルの CSS 定義においても、リテラル値の直書きを禁止する。

```css
/* ✅ 必須: 全値をデザイントークン経由で参照 */
table {
  width: 100%;
  max-width: var(--table-max-width);
  border-collapse: collapse;
}
th {
  background-color: var(--color-surface-subtle);
  color: var(--color-text-primary);
  padding: var(--table-cell-padding-y) var(--table-cell-padding-x);
  font-weight: var(--font-weight-semibold);
  text-align: left;
  border-bottom: var(--table-header-border-width) solid var(--color-border-strong);
}
td {
  padding: var(--table-cell-padding-y) var(--table-cell-padding-x);
  border-bottom: var(--table-border-width) solid var(--color-border-default);
}
tr:nth-child(even) {
  background-color: var(--color-surface-striped);
}
tr:hover {
  background-color: var(--color-surface-hover);
}

/* ❌ 禁止: リテラル値の散在 */
table {
  width: 100%;
  max-width: 960px;
  border-collapse: collapse;
}
th {
  background-color: #f5f5f5;
  color: #333333;
  padding: 12px 16px;
  font-weight: 600;
  text-align: left;
  border-bottom: 2px solid #dee2e6;
}
td {
  padding: 8px 16px;
  border-bottom: 1px solid #e0e0e0;
}
tr:nth-child(even) {
  background-color: #fafafa;
}
tr:hover {
  background-color: #f0f0f0;
}
```

**テーブル用トークン定義例**

```css
:root {
  --table-max-width: 60rem;
  --table-cell-padding-x: var(--spacing-4);
  --table-cell-padding-y: var(--spacing-3);
  --table-border-width: 1px;
  --table-header-border-width: 2px;
  --color-surface-striped: var(--color-neutral-50);
  --color-surface-hover: var(--color-neutral-100);
}
```

**固定列幅の禁止**

個別の列に対してピクセル値で幅を指定してはならない。列幅の制御が必要な場合は比率指定または `min-width` / `max-width` をトークン経由で使用する。

```css
/* ✅ 許容: 比率による列幅制御 */
th:nth-child(1) { width: 20%; }
th:nth-child(2) { width: 50%; }
th:nth-child(3) { width: 30%; }

/* ✅ 許容: トークン経由の制約 */
th:nth-child(1) { min-width: var(--table-col-min-width-name); }

/* ❌ 禁止: 列幅のピクセル値ハードコード */
th:nth-child(1) { width: 200px; }
th:nth-child(2) { width: 400px; }
th:nth-child(3) { width: 120px; }
```

#### `style` 属性の禁止（既定規約の再確認）

`style` 属性によるインラインスタイルは厳密に禁止する（本文書 HTML セクション既定）。インラインスタイルは全てハードコードされた値であり、テーマ切替・ダークモード対応・一括変更が不可能になる。

#### 固定寸法属性の禁止

`<img>`・`<video>`・`<canvas>` を除き、HTML 属性による `width`・`height` の指定を禁止する（本文書 HTML セクション既定）。`<img>` 等においても CLS 防止目的のアスペクト比指定に限定し、見た目の制御は CSS で行う。

#### 色・スタイル関連の非推奨属性の禁止

`bgcolor`・`color`・`border`・`align`・`valign` 等のプレゼンテーション属性は禁止する（本文書 HTML セクション既定）。

#### data 属性の値に関する注意

`data-*` 属性にステート値やテーマ値を格納する場合、属性値として使用する文字列は JavaScript 側の定数と一致させ、文字列リテラルの散在を防ぐ。

```html
<!-- ✅ 必須: 定数を一元管理 -->
<!-- JS側:
  const THEMES = { LIGHT: 'light', DARK: 'dark' } as const;
  el.dataset.theme = THEMES.DARK;
  if (el.dataset.theme === THEMES.DARK)
-->

<!-- ❌ 禁止: 文字列リテラルが HTML と JS に散在 -->
<div data-theme="dark">
<!-- JS側: if (el.dataset.theme === 'dark') -->
```

### JavaScript / TypeScript におけるハードコーディング禁止

#### マジックナンバー・マジックストリング

コード中に意味を持たない数値・文字列リテラルを直接記述してはならない。

| 禁止例 | 対処 |
|---|---|
| `if (status === 3)` | 名前付き定数を定義: `const STATUS_APPROVED = 3` |
| `setTimeout(fn, 3000)` | 定数化: `const DEBOUNCE_DELAY_MS = 3000` |
| `if (items.length > 50)` | 定数化: `const MAX_DISPLAY_ITEMS = 50` |

**例外**: `0`, `1`, `-1`, `100`（パーセント）など、文脈上意味が自明な値は許容する。ただし同一値が2箇所以上で使用される場合は定数化を必須とする。

#### 環境依存値

API エンドポイント、ドメイン名、ポート番号、ファイルパス、外部サービスの識別子など環境によって変わる値の直書きを禁止する。

```javascript
// ✅ 必須
fetch(`${process.env.API_BASE_URL}/users`)

// ❌ 禁止
fetch('https://api.example.com/v2/users')
```

環境変数の管理には `.env` ファイル（`.env.local`, `.env.production` 等）を使用し、バージョン管理には `.env.example`（値なしのキー一覧）のみをコミットする。

#### テキスト・ラベルの直書き（i18n 対応プロジェクト）

多言語対応が想定されるプロジェクトでは、ユーザーに表示する文字列をソースコード内にリテラルとして記述してはならない。翻訳キー・リソースファイル経由で参照する。単一言語プロジェクトでもエラーメッセージは定数ファイルに集約することを推奨する。

#### 日時フォーマット・ロケール

日時のフォーマット文字列やロケール識別子を各所に散在させてはならない。定数として一元管理する。

### 定数定義の原則

- 定数名は `SCREAMING_SNAKE_CASE` で命名する（CSS Custom Properties は `--kebab-case`）
- 関連する定数は目的別のファイル（`constants/api.ts`, `constants/ui.ts` 等）に集約する
- 定数ファイルは1ファイルに全てを詰め込まず、責務ごとに分割する
- 定数に対するコメントで意味・単位・許容範囲を記述する

---

## HTML

参照：[HTML Living Standard](https://html.spec.whatwg.org/multipage/)

- HTML Living Standard準拠（HTML5以前の非推奨要素・属性・書式を厳密に排除）
- 全ての`<script>`タグは`<head>`内に記載（`<body>`内禁止）
- 必ず`defer`または`async`属性を使用
- JavaScriptは必ず外部`.js`ファイルを`<script src>`で読み込み
- CSSは`<style>`で記述しない、必ず外部`.css`ファイルを`<link>`で読み込み

### ID・class属性

**基本概念**

| 属性 | 一意性 | 用途 | CSSセレクタ |
|---|---|---|---|
| `id` | ページ内で一意（重複禁止） | ページ内リンク、JS操作対象、フォームラベル紐付け | `#id-name` |
| `class` | 複数要素で共有可能 | 再利用可能なスタイル定義 | `.class-name` |

**原則**

- `id`はJavaScriptでの要素取得、ページ内アンカー、`<label for="">`との紐付けに使用
- **class属性はCSSセレクタで代替可能な場合は必ず代替（class付与前にセレクタ優先順位表を確認）**
- セマンティックな要素構造を活用し、class付与を最小限に抑える

**CSSセレクタ優先順位**

参照：[CSS Selectors Level 4](https://www.w3.org/TR/selectors-4/)

classを付与する前に以下のセレクタで対応可能か検討すること。[Can I Use](https://caniuse.com/)を参照し、最新の書式を積極的に取り入れる。

| 優先度 | セレクタ種別 | 例 | CSS Level |
|---|---|---|---|
| 1 | 要素セレクタ | `article`, `section`, `nav`, `header`, `footer`, `main`, `aside` | 1 |
| 2 | 子孫セレクタ | `article p`, `nav ul li` | 1 |
| 3 | 子セレクタ | `ul > li`, `nav > ul` | 2 |
| 4 | 隣接兄弟セレクタ | `h2 + p` | 2 |
| 5 | 一般兄弟セレクタ | `h2 ~ p` | 3 |
| 6 | 属性セレクタ | `[type="text"]`, `[data-state="active"]` | 2/3 |
| 7 | 擬似クラス（基本） | `:first-child`, `:last-child`, `:hover`, `:focus` | 2 |
| 8 | 擬似クラス（構造） | `:nth-child()`, `:nth-of-type()`, `:only-child`, `:empty` | 3 |
| 9 | 擬似クラス（否定） | `:not()` | 3 |
| 10 | 擬似クラス（状態） | `:checked`, `:disabled`, `:enabled`, `:valid`, `:invalid` | 3 |
| 11 | 擬似クラス（Level 4） | `:is()`, `:where()`, `:has()`, `:focus-visible`, `:focus-within` | 4 |
| 12 | 擬似要素 | `::before`, `::after`, `::first-line`, `::first-letter`, `::marker`, `::placeholder` | 2/3 |

**CSS Level 4 セレクタ（積極採用）**

[Can I Use](https://caniuse.com/)でサポート状況を確認の上、以下を積極的に採用：

| セレクタ | 用途 | サポート状況確認 |
|---|---|---|
| `:is()` | セレクタリストのグループ化 | [Can I Use :is()](https://caniuse.com/css-matches-pseudo) |
| `:where()` | 詳細度0のグループ化 | [Can I Use :where()](https://caniuse.com/mdn-css_selectors_where) |
| `:has()` | 親要素・前方兄弟の条件指定 | [Can I Use :has()](https://caniuse.com/css-has) |
| `:focus-visible` | キーボードフォーカス時のみスタイル適用 | [Can I Use :focus-visible](https://caniuse.com/css-focus-visible) |
| `:focus-within` | 子孫にフォーカスがある場合 | [Can I Use :focus-within](https://caniuse.com/css-focus-within) |
| `:not()` (複数引数) | 複数条件の否定 | [Can I Use :not()](https://caniuse.com/css-not-sel-list) |

**classが必要なケース**

- 同一要素タイプで異なるスタイルが必要な場合
- コンポーネントベースの再利用可能なスタイル
- JavaScript操作で動的に付与・削除するステート管理

### セマンティックマークアップ

参照：[HTML Living Standard](https://html.spec.whatwg.org/)

**文書構造**
- `<header>`, `<main>`, `<footer>`, `<nav>`, `<aside>` で文書構造を明示
- `<article>`, `<section>` で内容を論理的に区分
- 見出しは `<h1>`, `<h2>`, `<h3>`, `<h4>`, `<h5>`, `<h6>` を階層順に使用（スキップ禁止）

**テキスト・コンテンツ**
- 段落は `<p>`
- リストは `<ul>`, `<ol>`, `<dl>` を内容に応じて選択
- 引用は `<blockquote>`, `<q>`
- 強調は `<strong>`（重要性）, `<em>`（強勢）
- コードは `<code>`, `<pre>`

**インタラクティブ要素**
- ボタンは `<button>`（`<div onclick>` 禁止）
- リンクは `<a href>`
- フォーム要素は `<form>`, `<input>`, `<label>`, `<fieldset>`, `<legend>`

**禁止・制限**

参照：[HTML Living Standard - Obsolete features](https://html.spec.whatwg.org/multipage/obsolete.html)

*要素*
- `<span>` **完全禁止（例外なし）**
  - インライン強調：`<strong>`, `<em>`, `<mark>` を使用
  - コード：`<code>`, `<kbd>`, `<samp>`, `<var>` を使用
  - 引用：`<q>`, `<cite>` を使用
  - 略語：`<abbr>` を使用
  - 日時：`<time datetime="">` を使用
  - データ値：`<data value="">` を使用
  - 削除・挿入：`<del>`, `<ins>` を使用
  - 定義：`<dfn>` を使用
  - ルビ：`<ruby>`, `<rt>`, `<rp>` を使用
  - 上付き・下付き：`<sup>`, `<sub>` を使用
  - アイコン：CSS `::before`/`::after` 疑似要素を使用
- `<div>` **最上位親要素のみ許可**
  - 許可される使用：`<div id="root">`（Reactマウントポイント）、アプリ全体の最上位構造コンテナ（`role="presentation"`必須）
  - **深さ3以上での使用は絶対禁止**
  - セクション分割：`<section>`, `<article>`, `<aside>` を使用
  - ヘッダー・フッター：`<header>`, `<footer>` を使用
  - ナビゲーション：`<nav>` を使用
  - 図表：`<figure>` + `<figcaption>` を使用
  - リスト：`<ul>`, `<ol>`, `<dl>` を使用
  - フォームグループ：`<fieldset>` + `<legend>` を使用
  - アドレス：`<address>` を使用
  - 引用ブロック：`<blockquote>` を使用
  - 詳細表示：`<details>` + `<summary>` を使用
  - ダイアログ：`<dialog>` を使用
- `<br>` の連続使用禁止（余白はCSSで制御）
- `<table>` はレイアウト目的で使用禁止（表形式データのみ）
- `<center>` 禁止 → CSSで中央揃え
- `<font>` 禁止 → CSSでフォント指定
- `<b>` 禁止 → `<strong>` を使用
- `<i>` 禁止 → `<em>` または `<cite>` を使用
- `<u>` 禁止 → CSSで下線
- `<s>`, `<strike>` 禁止 → `<del>` を使用
- `<marquee>` 禁止
- `<blink>` 禁止
- `<frame>`, `<frameset>`, `<noframes>` 禁止
- `<applet>` 禁止
- `<acronym>` 禁止 → `<abbr>` を使用
- `<big>` 禁止 → CSSでサイズ指定
- `<tt>` 禁止 → `<code>` を使用

*属性*
- `align` 属性禁止 → CSSで配置
- `bgcolor` 属性禁止 → CSSで背景色
- `border` 属性禁止（`<table>`含む） → CSSでボーダー
- `cellpadding`, `cellspacing` 禁止 → CSSで余白
- `width`, `height` 属性（`<img>`, `<video>`, `<canvas>`以外）禁止 → CSSでサイズ
- `valign` 属性禁止 → CSS `vertical-align`
- `nowrap` 属性禁止 → CSS `white-space`
- `hspace`, `vspace` 属性禁止 → CSSでマージン
- `onclick` 等のインラインイベントハンドラ → JSで `addEventListener`
- `style` 属性は厳密に使用を禁止 → 外部CSSファイルで定義

*パターン*
- スペーサーGIF禁止 → CSSで余白
- テーブルレイアウト禁止 → CSS Grid / Flexbox
- `<br>` での余白調整禁止 → CSSでマージン/パディング
- `&nbsp;` の連続使用禁止 → CSSで余白
- `<div onclick>` でのボタン実装禁止 → `<button>` を使用
- `<a href="#">` でのボタン実装禁止 → `<button>` を使用
- 空の `href` や `src` 禁止
- `target="_blank"` 単独使用禁止 → `rel="noopener noreferrer"` を併記

参考：
- [MDN - Deprecated and obsolete features](https://developer.mozilla.org/en-US/docs/Web/HTML/Element#obsolete_and_deprecated_elements)
- [HTML5 Differences from HTML4](https://www.w3.org/TR/html5-diff/)

### `<dialog>` 要素の実装規約

参照：
- [WHATWG HTML Living Standard: dialog](https://html.spec.whatwg.org/multipage/interactive-elements.html)
- [MDN: `<dialog>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/dialog)
- [MDN: HTMLDialogElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLDialogElement)
- [MDN: Invoker Commands API](https://developer.mozilla.org/en-US/docs/Web/API/Invoker_Commands_API)
- [Chrome Developers: Introducing command and commandfor](https://developer.chrome.com/blog/command-and-commandfor)

**基本方針**

ダイアログの実装には`<dialog>`要素を必須とする。`position: fixed` + `z-index`による独自実装は禁止。`<dialog>`はclose watcher、フォーカストラップ、`::backdrop`、Escキー対応、`aria-modal`の自動付与をネイティブで提供する。

**開く操作**

| 方法 | 用途 |
|---|---|
| `showModal()` | モーダルダイアログ（背景がinertになる） |
| `command="show-modal"` | Invoker Commandsによる宣言的なモーダル表示（JS不要） |
| `show()` | 非モーダル表示。Popover APIが同等機能を提供するため、新規実装では非推奨 |

`open`属性の直接操作（`dialog.open = true/false`）は禁止。MDNおよびHTML仕様が明示的に非推奨としている。`close`イベントが発火しない、`close()`/`requestClose()`が効かなくなる等の不整合が生じる。

**閉じる操作**

`close()`メソッドおよびInvoker Commandsの`command="close"`は使用禁止とする。

| 禁止 | 理由 |
|---|---|
| `dialog.close()` | `cancel`イベントを経由せず即座にダイアログを閉じるため、キャンセル可能な閉じ方の統一が崩れる。未保存データ保護等の一元的な制御が不可能になる |
| `command="close"` | `close()`と同等の動作。`cancel`イベントを経由しない |

代わりに以下を使用する。

| 許可 | 動作 |
|---|---|
| `dialog.requestClose()` | `cancel`イベントを発火し、`preventDefault()`で阻止可能。阻止されなければ`close`イベントが続く。close watcherと同じイベントフローを提供する |
| `command="request-close"` | Invoker Commandsによる宣言的な`requestClose()`呼び出し（JS不要） |

これにより、全てのクローズ導線（Escキー、Closeボタン、プログラム的クローズ）で`cancel`イベントによる一元的制御が保証される。

`<form method="dialog">`によるダイアログの閉じる動作はブラウザの仕様に基づく動作であり、本規約の適用範囲外とする。

**`cancel`イベントハンドラーの設計パターン**

未保存データ保護の標準パターン：

```javascript
dialog.addEventListener('cancel', (event) => {
  if (hasUnsavedChanges()) {
    event.preventDefault();
    showConfirmation();
  }
});
```

プログラム的自動クローズ（保存完了後等）で`cancel`ガードを通過させるパターン：

```javascript
dialog.addEventListener('cancel', (event) => {
  // 保存済みフラグがあればガードしない
  if (dialog.dataset.saved === 'true') return;

  if (hasUnsavedChanges()) {
    event.preventDefault();
    showConfirmation();
  }
});
```

`cancel`ハンドラーのガードを迂回するために`close()`を使うのは設計の怠慢である。ガードの条件分岐で対処すること。

**`closedby` 属性**

参照：[Can I Use: closedby](https://caniuse.com/mdn-html_elements_dialog_closedby)

| 値 | 閉じ方 |
|---|---|
| `any` | ライトディスミス（背景クリック）、プラットフォーム固有操作（Escキー等）、開発者実装 |
| `closerequest` | プラットフォーム固有操作（Escキー等）、開発者実装 |
| `none` | 開発者実装のみ |
| 未指定（Auto） | `showModal()`→ `closerequest`相当、`show()`→ `none`相当 |

2026年2月時点でSafari（デスクトップ・iOS共）が未対応であり、Baseline未達（グローバルカバー率約70%）。Safari対応が必要なプロジェクトではポリフィル（[dialog-closedby-polyfill](https://github.com/tak-dcxi/dialog-closedby-polyfill)）を使用するか、`closedby`に依存しない実装とする。

モバイルでの注意：`closedby="any"`または`closedby="closerequest"`を指定した場合、Androidの「戻る」ボタンやスワイプジェスチャーがclose requestとして機能する。独自実装のダイアログが混在するページでは、「戻る」がダイアログを閉じるかページ遷移するかが予測不能になるリスクがある。

**Invoker Commands API（`command` / `commandfor`）**

参照：[Can I Use: Invoker Commands](https://caniuse.com/wf-invoker-commands)

Baseline Newly available（2025年12月、Safari 26.2で全主要ブラウザ対応完了）。JavaScriptなしで宣言的にダイアログの開閉を制御できる。`aria-expanded`相当のアクセシビリティ関係やフォーカス管理をブラウザが自動処理する。

ダイアログで使用するコマンド：

| コマンド | 動作 | 使用可否 |
|---|---|---|
| `show-modal` | `showModal()`相当 | **許可** |
| `request-close` | `requestClose()`相当 | **許可** |
| `close` | `close()`相当 | **禁止**（閉じる操作の規約を参照） |

```html
<!-- 標準パターン -->
<button type="button" commandfor="my-dialog" command="show-modal">
  ダイアログを開く
</button>

<dialog id="my-dialog">
  <h2>ダイアログタイトル</h2>
  <p>内容</p>
  <button type="button" commandfor="my-dialog" command="request-close">
    閉じる
  </button>
</dialog>
```

---

## CSS

- CSS Level 3およびLevel 4の最新書式を使用
- [Can I Use](https://caniuse.com/) で主要ブラウザのサポート状況を確認
  - 対象ブラウザ：Safari / Chrome / Edge / Firefox / Android Chrome / iOS Safari / Android Firefox
  - 対象バージョン：各ブラウザの最新バージョンと1つ前のバージョンまで
  - それ以前のバージョンが未対応でも対応不要（フォールバック実装の要否はTEDに確認）
- 判断が必要な際はTEDに確認
- ベンダープレフィックスは標準プロパティより先に記載（フォールバックを先に書き、標準プロパティで上書き）
- スクリーン向けカラーは[OKLCH](https://oklch.com/)を基準にHEX（ショートハンド不可）とRGBAも併記
- 参考：[OKLCH in CSS: why we moved from RGB and HSL](https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl)
- font-sizeはrem単位（基準値16px=1rem）、偶数ピクセル相当値のみ許可
- ライトモード・ダークモード・高コントラスト設定の3モード対応を基本とする

### CSS Custom Properties（CSS Variables）

**基本方針**: CSS Custom Properties はハードコーディング禁止の実装手段であり、全てのリテラル値を一元管理するための必須技術として位置づける。デザイントークン・テーマ切替・ダークモード対応・レスポンシブ対応の基盤として全プロジェクトで採用する。

#### 基本構文

Custom Properties は `--` プレフィックスで宣言し、`var()` 関数で参照する。

```css
/* 宣言 */
:root {
  --color-primary: oklch(0.55 0.2 260);
  --spacing-4: 1rem;
  --font-size-base: 1rem;
}

/* 参照 */
.button {
  background-color: var(--color-primary);
  padding: var(--spacing-4);
  font-size: var(--font-size-base);
}
```

#### ショートハンドプロパティでの使用

`var()` はプロパティ値内の任意の位置で使用でき、ショートハンドプロパティの各パートに個別に適用できる。W3C 仕様で `var()` はトークン列として展開されると定義されており、値の途中への挿入も有効である。

```css
/* ✅ 有効: ショートハンドの各パートに個別適用 */
.element {
  padding: var(--spacing-y) var(--spacing-x);
  margin: var(--spacing-2) auto var(--spacing-4);
  border: var(--border-width) solid var(--color-border);
  border-bottom: var(--table-border-width) solid var(--color-border-default);
  box-shadow: var(--shadow-offset-x) var(--shadow-offset-y) var(--shadow-blur) var(--color-shadow);
  transition: opacity var(--duration-normal) var(--easing-standard);
  font: var(--font-weight-normal) var(--font-size-base)/var(--line-height-normal) var(--font-family-base);
  grid-template-columns: var(--sidebar-width) 1fr var(--sidebar-width);
}

/* ✅ 有効: calc() との組み合わせ */
.container {
  max-width: calc(var(--content-max-width) + var(--spacing-8) * 2);
  font-size: clamp(var(--font-size-sm), 2vw, var(--font-size-lg));
}
```

#### 命名規則

| 規則 | 説明 |
|---|---|
| ケース | `--kebab-case`（小文字ハイフン区切り） |
| 大文字小文字の区別 | Custom Properties は**ケースセンシティブ**。`--my-color` と `--My-color` は別のプロパティとして扱われる。混乱防止のため小文字のみを使用する |
| プレフィックス | プリミティブトークンはカテゴリ名: `--color-*`, `--spacing-*`, `--font-*` |
| セマンティクス | セマンティクストークンは用途名: `--color-text-primary`, `--color-surface-default` |
| コンポーネント固有 | コンポーネント名をプレフィックス: `--button-*`, `--table-*`, `--card-*` |

```css
/* ✅ 推奨: 階層的な命名 */
:root {
  /* Primitive tokens（定義ファイルのみ） */
  --color-blue-600: oklch(0.55 0.2 260);

  /* Semantic tokens（参照用） */
  --color-action-primary: var(--color-blue-600);
  --color-text-primary: var(--color-neutral-900);

  /* Component tokens */
  --button-padding-x: var(--spacing-4);
  --table-cell-padding-y: var(--spacing-3);
}

/* ❌ 禁止: 意味不明な命名 */
:root {
  --c1: #1a73e8;
  --s: 16px;
  --x: 8px;
}
```

#### トークン階層（プリミティブ → セマンティクス → コンポーネント）

| 階層 | 定義場所 | 参照可否 | 例 |
|---|---|---|---|
| プリミティブ | トークン定義ファイルの `:root` | セマンティクストークンからのみ参照可 | `--color-blue-600`, `--spacing-4` |
| セマンティクス | トークン定義ファイルの `:root` / `[data-theme]` | コンポーネント・ユーティリティから参照可 | `--color-action-primary`, `--color-text-primary` |
| コンポーネント | コンポーネントの CSS | コンポーネント内部でのみ使用 | `--button-padding-x`, `--table-border-width` |

コンポーネント層からプリミティブトークンを直接参照してはならない（本文書「ハードコーディングの禁止」セクションおよび [component-design-patterns.md](component-design-patterns.md) §10 を参照）。

#### スコープとカスケード

Custom Properties は通常の CSS プロパティと同様にカスケードに従い、子要素に継承される。

```css
/* グローバルスコープ: 全要素に継承 */
:root {
  --color-surface-default: oklch(1 0 0);
}

/* テーマスコープ: data属性でテーマ切替 */
[data-theme="dark"] {
  --color-surface-default: oklch(0.2 0 0);
}

/* コンポーネントスコープ: 特定要素以下でのみ有効 */
.card {
  --card-padding: var(--spacing-4);
  padding: var(--card-padding);
}
.card.compact {
  --card-padding: var(--spacing-2);
}

/* メディアクエリスコープ: レスポンシブ対応 */
:root {
  --content-max-width: 100%;
}
@media (min-width: 768px) {
  :root {
    --content-max-width: 48rem;
  }
}
```

**注意**: `var()` はメディアクエリやコンテナクエリの条件式内では使用できない。条件式の外（プロパティ値として）でのみ使用可能。

```css
/* ✅ 有効: メディアクエリ内のプロパティ値 */
@media (min-width: 768px) {
  .container {
    max-width: var(--content-max-width);
  }
}

/* ❌ 無効: メディアクエリの条件式内 */
@media (min-width: var(--breakpoint-md)) { ... }
```

#### ライトモード / ダークモードの切替

CSS Custom Properties はライトモード / ダークモードの切替基盤として機能する。`:root` にライトモードのトークンを定義し、条件に応じてダークモードのトークンで上書きする構成を採用する。

**切替方式の比較**

| 方式 | 仕組み | 用途 |
|---|---|---|
| `prefers-color-scheme` | OS / ブラウザのシステム設定に自動追従 | ユーザー手動切替が不要な場合 |
| `data-theme` 属性 | JavaScript で `<html>` に属性を付与 | ユーザーが手動でテーマを選択する場合 |
| `color-scheme` プロパティ | ブラウザ UA スタイルシート（フォーム・スクロールバー等）の配色を宣言 | 上記いずれかと併用し、UA デフォルトの配色を連動させる |
| `light-dark()` 関数 | 1行で両モードの色値を指定 | 個別プロパティ単位で簡潔に書き分ける場合 |

**方式1: `prefers-color-scheme` によるシステム設定追従**

OS のダークモード設定に自動で追従する。ユーザー操作なしでテーマが切り替わる。

```css
/* ライトモード（デフォルト） */
:root {
  color-scheme: light dark;  /* UA スタイルシートにも両モード対応を宣言 */

  /* 背景・テキスト */
  --color-surface-default: oklch(1 0 0);          /* 白 */
  --color-surface-subtle: oklch(0.97 0 0);         /* 薄いグレー */
  --color-text-primary: oklch(0.2 0 0);            /* ほぼ黒 */
  --color-text-secondary: oklch(0.4 0 0);          /* グレー */

  /* ボーダー */
  --color-border-default: oklch(0.85 0 0);
  --color-border-strong: oklch(0.7 0 0);

  /* アクション */
  --color-action-primary: oklch(0.55 0.2 260);
  --color-action-primary-hover: oklch(0.45 0.2 260);
  --color-action-primary-text: oklch(1 0 0);       /* ボタン上の白文字 */

  /* ステータス */
  --color-status-success: oklch(0.55 0.15 145);
  --color-status-warning: oklch(0.7 0.15 85);
  --color-status-error: oklch(0.55 0.2 25);

  /* 影 */
  --shadow-sm: 0 1px 2px oklch(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px oklch(0 0 0 / 0.07);
}

/* ダークモード: システム設定が dark の場合に上書き */
@media (prefers-color-scheme: dark) {
  :root {
    /* 背景・テキスト */
    --color-surface-default: oklch(0.15 0 0);        /* ほぼ黒 */
    --color-surface-subtle: oklch(0.2 0 0);           /* 濃いグレー */
    --color-text-primary: oklch(0.93 0 0);            /* ほぼ白 */
    --color-text-secondary: oklch(0.7 0 0);           /* 明るいグレー */

    /* ボーダー */
    --color-border-default: oklch(0.3 0 0);
    --color-border-strong: oklch(0.45 0 0);

    /* アクション（ダークモードでは明度を上げて視認性を確保） */
    --color-action-primary: oklch(0.7 0.15 260);
    --color-action-primary-hover: oklch(0.8 0.12 260);
    --color-action-primary-text: oklch(0.1 0 0);     /* ボタン上の暗い文字 */

    /* ステータス（ダークモードでは明度を上げる） */
    --color-status-success: oklch(0.7 0.15 145);
    --color-status-warning: oklch(0.8 0.12 85);
    --color-status-error: oklch(0.7 0.17 25);

    /* 影（ダークモードでは影を強くする） */
    --shadow-sm: 0 1px 2px oklch(0 0 0 / 0.3);
    --shadow-md: 0 4px 6px oklch(0 0 0 / 0.4);
  }
}
```

HTML 側には `<meta>` タグを追加し、ページ読み込み時のフラッシュ（FOUC）を防止する。

```html
<head>
  <meta name="color-scheme" content="light dark">
  <link rel="stylesheet" href="main.css">
</head>
```

**方式2: `data-theme` 属性によるユーザー手動切替**

ユーザーがUIスイッチで明示的にテーマを選択する場合に使用する。`localStorage` でユーザーの選択を永続化できる。

```css
/* ライトモード（デフォルト） */
:root {
  color-scheme: light;

  --color-surface-default: oklch(1 0 0);
  --color-surface-subtle: oklch(0.97 0 0);
  --color-text-primary: oklch(0.2 0 0);
  --color-text-secondary: oklch(0.4 0 0);
  --color-border-default: oklch(0.85 0 0);
  --color-action-primary: oklch(0.55 0.2 260);
  --shadow-md: 0 4px 6px oklch(0 0 0 / 0.07);
}

/* ダークモード: data-theme 属性で上書き */
[data-theme="dark"] {
  color-scheme: dark;

  --color-surface-default: oklch(0.15 0 0);
  --color-surface-subtle: oklch(0.2 0 0);
  --color-text-primary: oklch(0.93 0 0);
  --color-text-secondary: oklch(0.7 0 0);
  --color-border-default: oklch(0.3 0 0);
  --color-action-primary: oklch(0.7 0.15 260);
  --shadow-md: 0 4px 6px oklch(0 0 0 / 0.4);
}
```

```javascript
// テーマ切替の JavaScript 実装
const THEMES = { LIGHT: 'light', DARK: 'dark' } as const;
const STORAGE_KEY = 'theme-preference';

function getSystemPreference(): string {
  return window.matchMedia('(prefers-color-scheme: dark)').matches
    ? THEMES.DARK
    : THEMES.LIGHT;
}

function getStoredPreference(): string | null {
  return localStorage.getItem(STORAGE_KEY);
}

function setTheme(theme: string): void {
  document.documentElement.dataset.theme = theme;
  localStorage.setItem(STORAGE_KEY, theme);
}

// 初期化: 保存済み設定 → システム設定の優先順
setTheme(getStoredPreference() ?? getSystemPreference());

// システム設定の変更を監視（ユーザーが明示的に選択していない場合のみ追従）
window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (e) => {
  if (!getStoredPreference()) {
    setTheme(e.matches ? THEMES.DARK : THEMES.LIGHT);
  }
});
```

**方式3: 方式1 + 方式2 の併用（推奨）**

システム設定への自動追従をベースとし、ユーザーが明示的に上書きできる構成。

```css
/* 1. ライトモード（デフォルト） */
:root {
  color-scheme: light dark;

  --color-surface-default: oklch(1 0 0);
  --color-text-primary: oklch(0.2 0 0);
  --color-border-default: oklch(0.85 0 0);
  --color-action-primary: oklch(0.55 0.2 260);
}

/* 2. システム設定がダークの場合に自動追従 */
@media (prefers-color-scheme: dark) {
  :root {
    --color-surface-default: oklch(0.15 0 0);
    --color-text-primary: oklch(0.93 0 0);
    --color-border-default: oklch(0.3 0 0);
    --color-action-primary: oklch(0.7 0.15 260);
  }
}

/* 3. ユーザーが明示的にライトモードを選択した場合（システム設定を上書き） */
[data-theme="light"] {
  color-scheme: light;

  --color-surface-default: oklch(1 0 0);
  --color-text-primary: oklch(0.2 0 0);
  --color-border-default: oklch(0.85 0 0);
  --color-action-primary: oklch(0.55 0.2 260);
}

/* 4. ユーザーが明示的にダークモードを選択した場合（システム設定を上書き） */
[data-theme="dark"] {
  color-scheme: dark;

  --color-surface-default: oklch(0.15 0 0);
  --color-text-primary: oklch(0.93 0 0);
  --color-border-default: oklch(0.3 0 0);
  --color-action-primary: oklch(0.7 0.15 260);
}
```

**`@import` によるテーマファイル分離（推奨）**

テーマごとのトークン定義が増大した場合は、ファイルを分離する。

```
styles/
└── tokens/
    ├── colors.css              /* :root のライトモード色定義 */
    ├── colors-dark.css         /* @media (prefers-color-scheme: dark) の色定義 */
    ├── colors-theme-light.css  /* [data-theme="light"] の色定義 */
    └── colors-theme-dark.css   /* [data-theme="dark"] の色定義 */
```

```css
/* tokens.css */
@import url('tokens/colors.css');
@import url('tokens/colors-dark.css');
@import url('tokens/colors-theme-light.css');
@import url('tokens/colors-theme-dark.css');
@import url('tokens/spacing.css');
@import url('tokens/typography.css');
/* ... */
```

**`light-dark()` 関数による簡略記法**

`light-dark()` CSS 関数を使用すると、1行で両モードの色値を指定できる。`color-scheme` の宣言が前提。Baseline 2024（全主要ブラウザ対応済み）。

```css
:root {
  color-scheme: light dark;

  --color-surface-default: light-dark(oklch(1 0 0), oklch(0.15 0 0));
  --color-text-primary: light-dark(oklch(0.2 0 0), oklch(0.93 0 0));
  --color-border-default: light-dark(oklch(0.85 0 0), oklch(0.3 0 0));
  --color-action-primary: light-dark(oklch(0.55 0.2 260), oklch(0.7 0.15 260));
}
```

`light-dark()` は `<color>` 型のプロパティでのみ使用可能。`box-shadow` の offset や `border-width` 等の非色値には使用できない。非色値のモード分岐は `prefers-color-scheme` メディアクエリまたは `data-theme` 属性で対応する。

```css
/* ✅ 有効: 色値部分のみに使用 */
.card {
  /* shadow の色部分に light-dark() を適用し、offset/blur はトークン経由 */
  --shadow-color: light-dark(oklch(0 0 0 / 0.07), oklch(0 0 0 / 0.4));
  box-shadow: var(--shadow-offset-x) var(--shadow-offset-y) var(--shadow-blur) var(--shadow-color);
}

/* ❌ 無効: light-dark() は色値以外に使えない */
.card {
  box-shadow: 0 4px 6px light-dark(oklch(0 0 0 / 0.07), oklch(0 0 0 / 0.4));
  /* ↑ box-shadow 全体は <color> 型ではないため無効 */
}
```

**`color-scheme` プロパティの役割**

`color-scheme` は UA スタイルシート（ブラウザデフォルト）のフォームコントロール・スクロールバー・システムカラーの配色を宣言する。`prefers-color-scheme` や `data-theme` とは役割が異なり、併用が推奨される。

```css
/* color-scheme を宣言しないと、ダークモードでもフォームやスクロールバーが
   ライトモードのまま表示され、視覚的に不統一になる */

:root {
  color-scheme: light dark;  /* 両モードに対応することをブラウザに宣言 */
}

/* 特定要素をライトモード固定にする場合 */
.always-light-section {
  color-scheme: only light;
}
```

**ダークモード設計の注意事項**

| 観点 | 指針 |
|---|---|
| 明度の反転 | 単純な白黒反転ではなく、ダークモードでは明度を適切に調整する。純白（`oklch(1 0 0)`）はダーク背景上で眩しいため、`oklch(0.93 0 0)` 程度に抑える |
| 彩度の調整 | ダーク背景では高彩度色が視覚的に強くなるため、彩度を下げるか明度を上げて調整する |
| コントラスト比 | WCAG 2.2 AA 基準（通常テキスト 4.5:1、大テキスト 3:1）を両モードで満たすこと |
| 影 | ダーク背景では影の不透明度を高くしないと視認できない |
| 画像 | 明るい画像はダークモードで眩しくなる場合がある。CSS filter による減光（`brightness(0.8)`）を検討する |
| テスト | 両モードでの表示確認を必須とする。Chrome DevTools の Rendering パネルで `prefers-color-scheme` をエミュレート可能 |

参照：
- [MDN - prefers-color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-color-scheme)
- [MDN - color-scheme](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/color-scheme)
- [MDN - light-dark()](https://developer.mozilla.org/en-US/docs/Web/CSS/color_value/light-dark)
- [web.dev - color-scheme](https://web.dev/articles/color-scheme)
- [web.dev - prefers-color-scheme](https://web.dev/articles/prefers-color-scheme)
- [CSS-Tricks - color-scheme](https://css-tricks.com/almanac/properties/c/color-scheme/)

#### フォールバック値

`var()` の第2引数でフォールバック値を指定できる。Custom Property が未定義または無効な場合に適用される。

```css
/* フォールバック値の指定 */
.element {
  color: var(--color-text-primary, oklch(0.2 0 0));
  padding: var(--spacing-custom, var(--spacing-4));
}
```

フォールバック値は以下の場合に使用する。
- 外部から注入される Custom Property（サードパーティテーマ等）で未定義の可能性がある場合
- コンポーネントが単独で動作する必要がある場合

プロジェクト内で `:root` に定義済みの Custom Property にフォールバックを付ける必要はない。

#### ファイル分離と `@import` 管理

Custom Properties の定義が増加した場合、単一ファイルへの集約は可読性と保守性を損なう。カテゴリ（色・間隔・タイポグラフィ等）ごとに個別ファイルへ分離し、`@import` で統合することを推奨する。

**推奨ディレクトリ構成**

```
styles/
├── tokens/
│   ├── colors.css          /* 色トークン */
│   ├── spacing.css         /* 間隔トークン */
│   ├── typography.css      /* タイポグラフィトークン */
│   ├── shadows.css         /* 影トークン */
│   ├── radii.css           /* 角丸トークン */
│   ├── z-index.css         /* z-index スケール */
│   ├── motion.css          /* アニメーション duration / easing */
│   ├── breakpoints.css     /* ブレークポイント定義 */
│   └── components/
│       ├── table.css       /* テーブル固有トークン */
│       ├── button.css      /* ボタン固有トークン */
│       └── dialog.css      /* ダイアログ固有トークン */
├── tokens.css              /* トークン統合エントリポイント */
├── reset.css               /* CSS リセット */
└── main.css                /* アプリケーション全体のエントリポイント */
```

**トークン統合ファイル（`tokens.css`）**

```css
/* tokens.css — トークン定義の統合エントリポイント */
/* @import はファイル先頭に記述すること（他のルールより前） */

/* グローバルトークン */
@import url('tokens/colors.css');
@import url('tokens/spacing.css');
@import url('tokens/typography.css');
@import url('tokens/shadows.css');
@import url('tokens/radii.css');
@import url('tokens/z-index.css');
@import url('tokens/motion.css');
@import url('tokens/breakpoints.css');

/* コンポーネントトークン */
@import url('tokens/components/table.css');
@import url('tokens/components/button.css');
@import url('tokens/components/dialog.css');
```

**メインエントリポイント（`main.css`）**

```css
/* main.css — アプリケーション全体のエントリポイント */
@import url('tokens.css');
@import url('reset.css');

/* 以降にアプリケーション固有のスタイルを記述 */
```

**各トークンファイルの記述例**

```css
/* tokens/colors.css */
:root {
  /* Primitive */
  --color-blue-50: oklch(0.97 0.01 250);
  --color-blue-600: oklch(0.55 0.2 260);
  --color-blue-700: oklch(0.45 0.2 260);
  --color-neutral-50: oklch(0.98 0 0);
  --color-neutral-900: oklch(0.2 0 0);

  /* Semantic */
  --color-action-primary: var(--color-blue-600);
  --color-action-primary-hover: var(--color-blue-700);
  --color-text-primary: var(--color-neutral-900);
  --color-surface-default: oklch(1 0 0);
  --color-border-default: var(--color-neutral-200);
}

[data-theme="dark"] {
  --color-surface-default: oklch(0.15 0 0);
  --color-text-primary: var(--color-neutral-50);
}
```

```css
/* tokens/spacing.css */
:root {
  --spacing-1: 0.25rem;   /*  4px */
  --spacing-2: 0.5rem;    /*  8px */
  --spacing-3: 0.75rem;   /* 12px */
  --spacing-4: 1rem;      /* 16px */
  --spacing-6: 1.5rem;    /* 24px */
  --spacing-8: 2rem;      /* 32px */
}
```

```css
/* tokens/components/table.css */
:root {
  --table-max-width: 60rem;
  --table-cell-padding-x: var(--spacing-4);
  --table-cell-padding-y: var(--spacing-3);
  --table-border-width: 1px;
  --table-header-border-width: 2px;
  --color-surface-striped: var(--color-neutral-50);
  --color-surface-hover: var(--color-neutral-100);
}
```

**分離の判断基準**

| 状況 | 対応 |
|---|---|
| トークン定義が合計50件未満 | 単一ファイル（`tokens.css` に直接記述）でも許容 |
| トークン定義が合計50件以上 | カテゴリ別ファイルへの分離を必須とする |
| コンポーネント固有トークンが10件以上 | `tokens/components/` 配下に個別ファイルとして分離 |
| テーマ（ダークモード等）の定義が増大 | テーマ別ファイル（`tokens/theme-dark.css` 等）への分離を推奨 |

**`@import` の制約と注意事項**

| 制約 | 説明 |
|---|---|
| 記述位置 | `@import` はスタイルシートの先頭に記述しなければならない。`@charset` と `@layer` 以外のルールより前に置く。他のルールの後に書かれた `@import` はブラウザに無視される |
| ネスト禁止 | `@import` は `@media` や `@supports` 等の条件付きグループルール内に記述できない |
| 読み込み順 | `@import` の記述順がカスケード順序を決定する。同名の Custom Property が複数ファイルに存在する場合、後に `@import` されたファイルの値が優先される |
| パフォーマンス | `@import` はリクエストを逐次発行する（HTTP/1.1 環境）。本番環境ではビルドツール（PostCSS・Sass・Vite 等）でファイルを結合することを推奨する |
| メディアクエリ付き `@import` | `@import url('print.css') print;` の形式でメディア条件付きインポートが可能 |
| `@layer` 付き `@import` | `@import url('lib.css') layer(vendor);` の形式でカスケードレイヤーへのインポートが可能 |

**ビルドツールとの併用**

開発時は `@import` によるファイル分離で可読性を確保し、本番ビルドではツールによる結合で HTTP リクエスト数を削減する。

```javascript
// postcss.config.js（postcss-import プラグイン使用例）
module.exports = {
  plugins: [
    require('postcss-import'),  // @import をインライン展開
    require('postcss-custom-media'),  // @custom-media を展開
    require('autoprefixer'),
  ],
};
```

```javascript
// vite.config.js（Vite は CSS @import を標準でバンドル）
export default {
  css: {
    // Vite は @import を自動的にインライン展開する
    // 追加設定不要
  },
};
```

参照：
- [MDN - @import](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@import)
- [CSS Cascading and Inheritance Level 5 - @import](https://www.w3.org/TR/css-cascade-5/#at-import)
- [CSS-Tricks - @import](https://css-tricks.com/almanac/rules/i/import/)

#### `@property` によるカスタムプロパティの型定義

`@property` アットルールを使用すると、Custom Properties に型制約・初期値・継承制御を宣言的に定義できる。Baseline 2024（全主要ブラウザ対応済み）。

```css
/* 型安全な Custom Property の定義 */
@property --color-primary {
  syntax: "<color>";
  inherits: true;
  initial-value: oklch(0.55 0.2 260);
}

@property --spacing-base {
  syntax: "<length>";
  inherits: true;
  initial-value: 1rem;
}

@property --rotation {
  syntax: "<angle>";
  inherits: false;
  initial-value: 0deg;
}
```

| 記述子 | 必須 | 説明 |
|---|---|---|
| `syntax` | 必須 | 許容される値の型。`"<color>"`, `"<length>"`, `"<number>"`, `"<percentage>"`, `"<angle>"`, `"*"`（任意） |
| `inherits` | 必須 | `true`: 親要素から継承 / `false`: 継承しない |
| `initial-value` | `syntax` が `"*"` 以外の場合は必須 | 初期値。`syntax` の型に一致する値を指定 |

**`@property` の利点**:
- 無効な値が代入された場合、`initial-value` にフォールバックする（通常の Custom Properties は `unset` になる）
- アニメーション・トランジションの中間値補間が可能になる（通常の Custom Properties は離散的に変化する）
- 型制約による設計意図の明文化

```css
/* ✅ @property で定義すると色のトランジションが可能になる */
@property --bg-color {
  syntax: "<color>";
  inherits: false;
  initial-value: oklch(0.95 0 0);
}
.card {
  background-color: var(--bg-color);
  transition: --bg-color var(--duration-normal) var(--easing-standard);
}
.card:hover {
  --bg-color: oklch(0.9 0.02 250);
}
```

**`@property` の使用基準**:

| 状況 | 推奨 |
|---|---|
| トランジション・アニメーション対象の Custom Property | `@property` で型定義する |
| デザイントークンの初期値保証が必要 | `@property` で `initial-value` を宣言する |
| 継承を明示的に制御したい | `@property` で `inherits: false` を指定する |
| 単純な値の一元管理のみ | `:root` での通常宣言で十分 |

#### 禁止事項

| 禁止 | 理由 |
|---|---|
| 大文字を含む命名（`--myColor`、`--MyColor`） | ケースセンシティブによる混乱防止。`--kebab-case` に統一 |
| 1〜2文字の省略命名（`--c1`、`--s`） | 意味不明。セマンティックな命名を必須とする |
| `var()` のネスト5階層以上 | デバッグ困難。3階層以内を推奨 |
| JavaScript からの `style.setProperty` による大量の Custom Property 操作 | パフォーマンス劣化。クラス切替やテーマ属性の変更で対処する |

#### ブラウザサポート

| 機能 | サポート状況確認 |
|---|---|
| CSS Custom Properties (`--*` / `var()`) | [Can I Use - CSS Variables](https://caniuse.com/css-variables) |
| `@property` アットルール | [Can I Use - @property](https://caniuse.com/mdn-css_at-rules_property) |

#### 参照

- [CSS Custom Properties for Cascading Variables Module Level 1](https://www.w3.org/TR/css-variables-1/) - W3C Candidate Recommendation
- [CSS Custom Properties Level 2 Editor's Draft](https://drafts.csswg.org/css-variables-2/)
- [MDN - Using CSS custom properties](https://developer.mozilla.org/en-US/docs/Web/CSS/Using_CSS_custom_properties)
- [MDN - Custom properties (--*)](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/--*)
- [MDN - @property](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@property)
- [Can I Use - CSS Variables](https://caniuse.com/css-variables)
- [Can I Use - @property](https://caniuse.com/mdn-css_at-rules_property)
- [web.dev - Custom properties](https://web.dev/learn/css/custom-properties)
- [CSS-Tricks - A Complete Guide to Custom Properties](https://css-tricks.com/a-complete-guide-to-custom-properties/)

### CSSカスケードの原則

参照：
- [MDN - Introduction to the CSS cascade](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Cascade/Introduction)
- [web.dev - The cascade](https://web.dev/learn/css/the-cascade)

| 原則 | 説明 |
|---|---|
| 同一詳細度では後が勝つ | 同じセレクタ・同じ詳細度の場合、最後に宣言されたプロパティが適用される |
| ベンダープレフィックス→標準 | ブラウザは最後に理解できるルールを適用するため、標準プロパティを最後に記載 |
| フォールバック値の活用 | 同じプロパティを複数回宣言することでフォールバックを実現（未対応ブラウザは無視） |

```css
/* 正しい順序の例 */
.element {
  -webkit-backdrop-filter: blur(10px);  /* ベンダープレフィックス（先） */
  backdrop-filter: blur(10px);          /* 標準プロパティ（後） */
  
  font-size: 16px;                      /* フォールバック値（先） */
  font-size: clamp(1rem, 2vw, 1.5rem);  /* 標準値（後） */
}
```

### CSSリセット

**原則：ブラウザデフォルトスタイルをリセットし、一貫した表示基盤を構築する**

参照：
- [W3C CSS Specifications](https://www.w3.org/Style/CSS/)
- [CSS Current Work](https://www.w3.org/Style/CSS/current-work)
- [CSS Basic User Interface Module Level 3](https://www.w3.org/TR/css-ui-3/)
- [CSS Basic User Interface Module Level 4](https://www.w3.org/TR/css-ui-4/)
- [Josh W. Comeau - A Modern CSS Reset（2025/12/27のブログ記事）](https://www.joshwcomeau.com/css/custom-css-reset/)
- [Andy Bell - A (more) Modern CSS Reset](https://piccalil.li/blog/a-more-modern-css-reset/)
- [modern-normalize](https://github.com/sindresorhus/modern-normalize)
- [CSS Remedy](https://github.com/jensimmons/cssremedy)

**採用基準**
- Josh W. Comeau's Modern CSS Reset（2025/12/27のブログ記事）を業界標準として参照
- CSS Level 3およびLevel 4の最新機能を網羅
- 既存 GENERAL.md 規約（line-height: 1.5, font: inherit）との整合性確認済み

#### 基本のリセット項目

```css
/* 1. box-sizing: border-box（全要素） - CSS Level 3 */
*, *::before, *::after {
  box-sizing: border-box;
}

/* 2. margin: 0（dialogを除く全要素） */
*:not(dialog) {
  margin: 0;
}

/* 3. interpolate-size（CSS Level 4: height: auto等へのアニメーション対応） */
@media (prefers-reduced-motion: no-preference) {
  html {
    interpolate-size: allow-keywords;
  }
}

/* 4. line-height: 1.5（WCAG 2.2 AA準拠） */
body {
  line-height: 1.5;
  /* 5. -webkit-font-smoothing: antialiased（macOS/iOSテキスト表示改善） */
  -webkit-font-smoothing: antialiased;
}

/* 6. メディア要素: display: block; max-width: 100% */
img, picture, video, canvas, svg {
  display: block;
  max-width: 100%;
}

/* 7. フォーム要素: font: inherit（13.333px問題解決） */
input, button, textarea, select {
  font: inherit;
}

/* 8. overflow-wrap: break-word（テキストオーバーフロー防止） */
p, h1, h2, h3, h4, h5, h6 {
  overflow-wrap: break-word;
}

/* 9. text-wrap: pretty（CSS Level 4: 段落の改善された改行） */
p {
  text-wrap: pretty;
}

/* 10. text-wrap: balance（CSS Level 4: 見出しのバランス調整） */
h1, h2, h3, h4, h5, h6 {
  text-wrap: balance;
}

/* 11. #root, #__next（React/Next.js用スタッキングコンテキスト） */
#root, #__next {
  isolation: isolate;
}
```

**ブラウザサポート状況**

| 機能 | CSS Level | サポート状況確認 |
|---|---|---|
| `box-sizing` | 3 | [Can I Use box-sizing](https://caniuse.com/css3-boxsizing) |
| `interpolate-size` | 4 | [Can I Use interpolate-size](https://caniuse.com/mdn-css_properties_interpolate-size) |
| `text-wrap: pretty` | 4 | [Can I Use text-wrap pretty](https://caniuse.com/mdn-css_properties_text-wrap_pretty) |
| `text-wrap: balance` | 4 | [Can I Use text-wrap balance](https://caniuse.com/css-text-wrap-balance) |

**注意事項**
- 見出し（h1-h6）に対する`line-height: 1.5`は大きすぎる場合があるため、必要に応じて上書き
- `max-width: 100%`がレイアウトに支障をきたす場合は`max-width: revert`で解除
- プロジェクト固有の調整は必要に応じて追加

#### フォームリセット

参照：
- [CSS Basic User Interface Module Level 3](https://www.w3.org/TR/css-ui-3/)
- [CSS Basic User Interface Module Level 4](https://www.w3.org/TR/css-ui-4/)
- [MDN - appearance](https://developer.mozilla.org/en-US/docs/Web/CSS/appearance)

```css
/* フォーム要素の完全リセット（CSS UI Level 3/4準拠） */
input,
select,
textarea,
button {
  /* appearance（CSS UI Level 3/4） */
  -webkit-appearance: none;
  appearance: none;
  
  /* 基本リセット */
  border: none;
  border-radius: 0;
  background: none;
  font: inherit;
  color: inherit;
  margin: 0;
  padding: 0;
}
```

### フォーム要素スタイリング

**原則：OS由来のアピアランスを全てリセットし、CSSで独自に定義する**

参照：[MDN - appearance](https://developer.mozilla.org/en-US/docs/Web/CSS/appearance)

**リセット対象要素**

| 要素 | リセットプロパティ |
|---|---|
| `input[type="text"]`, `input[type="email"]`, `input[type="password"]`, `input[type="search"]`, `input[type="tel"]`, `input[type="url"]`, `input[type="number"]` | `appearance: none;` |
| `input[type="checkbox"]`, `input[type="radio"]` | `appearance: none;` |
| `input[type="range"]` | `appearance: none;` |
| `input[type="file"]` | `appearance: none;` （`::file-selector-button`も含む） |
| `input[type="date"]`, `input[type="time"]`, `input[type="datetime-local"]`, `input[type="month"]`, `input[type="week"]` | `appearance: none;` |
| `select` | `appearance: none;` |
| `textarea` | `appearance: none;` |
| `button`, `input[type="button"]`, `input[type="submit"]`, `input[type="reset"]` | `appearance: none;` |
| `progress` | `appearance: none;` |
| `meter` | `appearance: none;` |

**必須の独自定義項目**

- ボーダー・アウトライン
- 背景色（ライトモード・ダークモード・高コントラスト設定対応）
- フォーカス状態（`:focus`, `:focus-visible`）
- 無効状態（`:disabled`）
- 入力検証状態（`:valid`, `:invalid`, `:required`）
- プレースホルダー（`::placeholder`）
- チェックボックス・ラジオボタンのチェック状態（`:checked`）

**高コントラスト設定対応**
- チェック状態は`background-color`ではなく`border`で示す（`forced-colors`で上書きされるため）
- フォーカスリングは`outline`を使用（`box-shadow`は`none`になる）
- システムカラーキーワード（`Field`, `FieldText`, `ButtonFace`, `ButtonText`）を適切に使用

---

## パフォーマンス原則

- 不要なDOM再レンダリングを回避
- ページ全体の再レンダリングではなく、特定UIコンポーネントのみを更新するターゲット更新関数を採用
- 大規模アイテムリストのパフォーマンス最適化を考慮
