---
title: "Accessibility Implementation Guide"
description: "WCAG 2.2 AA準拠アクセシビリティ実装 - WAI-ARIA属性・キーボード操作・スクリーンリーダー・フォーカス管理 / WCAG 2.2 AA accessibility - WAI-ARIA attributes, keyboard navigation, screen readers, focus management"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-15T22:23+09:00"
lang: "ja"
---

# Accessibility Implementation Guide

**WAI-ARIA アクセシビリティ実装ガイドライン** - WCAG 2.2 AA/AAA準拠を目標としたアクセシビリティを実装レベルで担保するためのWAI-ARIA属性の正しい選定と適用


---

## 目的

ウェブコンテンツおよびウェブアプリケーションのアクセシビリティを実装レベルで担保するための仕様書。WAI-ARIA属性の正しい選定と適用、キーボード操作、フォーカス管理、支援技術（AT）対応、動的コンテンツ処理の実装規約を定義する。

## 背景

- WCAG 2.2（2023-10-05 W3C Recommendation）で9つの達成基準が新規追加された
- WAI-ARIAの誤用は「ARIA なしよりも悪い」結果を招く（WebAIM Million 2025調査：ARIA使用ページは未使用ページより平均11.2件多いエラーを検出。2024調査では34.2%多い検出率）
- 設計原則・コントラスト比・タッチターゲット・カラーモードは ui-ux-guidelines.md で定義済みであり、本書は実装面の仕様を補完する

## 対象

- HTML / CSS / JavaScript によるフロントエンド実装
- SPA（Single Page Application）を含む動的ウェブコンテンツ
- React / Vue 等のコンポーネントベースフレームワーク

## 非対象

- 設計原則・コントラスト比・タッチターゲット・カラーモード対応（→ ui-ux-guidelines.md）
- モーダルダイアログの閉じるアフォーダンス分類（→ modal-dialog-close-affordance.md）
- ネイティブモバイルアプリ（iOS / Android）固有のAPI
- PDF・オフィス文書のアクセシビリティ

## 用語

| 用語 | 定義 |
| --- | --- |
| AT（Assistive Technology） | スクリーンリーダー、拡大ソフト、音声入力、スイッチデバイス等、障害のあるユーザーを支援する技術の総称 |
| アクセシビリティツリー | ブラウザがDOMから構築し、ATに公開するUIの意味的表現。ロール・名前・状態・値で構成される |
| アクセシブルネーム | ATがユーザーに伝える要素の名前。Accessible Name and Description Computation 1.2 のアルゴリズムで算出される |
| ランドマーク | ページ構造の主要セクションを識別する領域。ATによるページ内ナビゲーションの基盤 |
| ライブリージョン | コンテンツが動的に更新される領域。ATが更新をリアルタイムに通知する |
| ロービングタブインデックス | 複合ウィジェット内で矢印キーによるフォーカス移動を実現するパターン。Tab/Shift+Tabではウィジェット単位でフォーカスし、内部要素間は矢印キーで移動する |
| フォーカストラップ | モーダルダイアログ等でTabキーのフォーカス移動をダイアログ内に制限するパターン |

---

## 1. ARIAの基本原則

### 1.1 ARIAの5つのルール

WAI-ARIAは「HTML の持っている意味や構造を拡張できる仕組み」であり、パッチ当て（絆創膏）として機能する。不用意な使用は、視覚的に見えている内容とスクリーンリーダーの読み上げ内容にズレを生じさせる。

| ルール | 内容 | 根拠 |
| --- | --- | --- |
| 第1ルール | ネイティブHTML要素・属性で実現できるなら、ARIAを使用しない | Using ARIA - W3C |
| 第2ルール | ネイティブセマンティクスを変更しない（例：`<h2 role="tab">` は不可） | Using ARIA - W3C |
| 第3ルール | すべてのインタラクティブARIAコントロールはキーボードで操作可能にする | Using ARIA - W3C |
| 第4ルール | フォーカス可能な要素に `role="presentation"` や `aria-hidden="true"` を使用しない | Using ARIA - W3C |
| 第5ルール | すべてのインタラクティブ要素にアクセシブルネームを付与する | Using ARIA - W3C |

### 1.2 ARIAが提供するもの

ARIAは**セマンティクスのみ**を提供する。動作（キーボード操作、フォーカス管理等）は実装者がJavaScriptで担保する。

```html
<!-- 悪い例：role だけでは動作しない -->
<div role="button">送信</div>

<!-- 良い例：ネイティブ要素を使う -->
<button type="submit">送信</button>

<!-- やむを得ずdivを使う場合：動作を自前で実装する -->
<div role="button" tabindex="0">送信</div>
```

`<div role="button">` を使用する場合、以下すべてを自前で実装する必要がある：

```javascript
const customButton = document.querySelector('[role="button"]');

// Enter キーとスペースキーで動作
customButton.addEventListener('keydown', (event) => {
  if (event.key === 'Enter' || event.key === ' ') {
    event.preventDefault();
    customButton.click();
  }
});

// フォーカス可能にする（tabindex="0" をHTMLに記述済み）
// クリックイベントを処理する
customButton.addEventListener('click', () => {
  // 送信処理
});
```

### 1.3 HTML要素の暗黙のロール

ネイティブHTML要素には暗黙のARIAロールが設定されている。明示的な `role` 属性での重複指定は不要。

| HTML要素 | 暗黙のロール | 上書き禁止のロール |
| --- | --- | --- |
| `<button>` | `button` | `link`, `menuitem`, `menuitemcheckbox`, `menuitemradio`, `option`, `radio`, `switch`, `tab` 以外禁止 |
| `<a href="...">` | `link` | `button`, `checkbox`, `menuitem`, `menuitemcheckbox`, `menuitemradio`, `option`, `radio`, `switch`, `tab`, `treeitem` 以外禁止 |
| `<input type="text">` | `textbox` | `combobox`, `searchbox`, `spinbutton` 以外禁止 |
| `<input type="checkbox">` | `checkbox` | `menuitemcheckbox`, `option`, `switch` 以外禁止 |
| `<input type="radio">` | `radio` | `menuitemradio` 以外禁止 |
| `<select>` | `combobox` / `listbox` | 条件付き |
| `<header>`（`<article>` 等の子でない場合） | `banner` | 上書き禁止 |
| `<nav>` | `navigation` | 上書き禁止 |
| `<main>` | `main` | 上書き禁止 |
| `<footer>`（`<article>` 等の子でない場合） | `contentinfo` | 上書き禁止 |
| `<aside>` | `complementary` | 上書き禁止 |
| `<section>`（アクセシブルネームあり） | `region` | — |
| `<form>`（アクセシブルネームあり） | `form` | — |
| `<table>` | `table` | — |
| `<img alt="テキスト">` | `img` | — |
| `<img alt="">` | `presentation` | — |

```html
<!-- 不要：暗黙のロールと同一 -->
<nav role="navigation">...</nav>
<main role="main">...</main>

<!-- 正しい：ネイティブ要素のみで十分 -->
<nav>...</nav>
<main>...</main>
```

---

## 2. アクセシブルネーム

### 2.1 アクセシブルネームの算出優先順序

Accessible Name and Description Computation 1.2 で定義されたアルゴリズムに基づく。上位が下位に優先する。

| 優先順位 | 方法 | 例 |
| --- | --- | --- |
| 1 | `aria-labelledby` | `aria-labelledby="YOUR_TITLE_ID"` |
| 2 | `aria-label` | `aria-label="閉じる"` |
| 3 | ネイティブラベル | `<label for="YOUR_INPUT_ID">`, `alt`, `<caption>`, `<legend>`, `<figcaption>` |
| 4 | `title` 属性 | `title="補足テキスト"` |
| 5 | `placeholder` 属性 | `placeholder="検索..."` （ラベルとして不十分、使用禁止） |

**重要**：`aria-labelledby` と `aria-label` を同一要素に併記した場合、`aria-labelledby` が優先され `aria-label` は無視される（MDN、WAI-ARIA仕様で明記）。

ただし、`aria-labelledby` で自分自身のIDを参照する場合は例外的に `aria-label` の値が使われる。これは Accessible Name Computation のアルゴリズムにおいて、再帰時に `aria-labelledby` のステップをスキップし `aria-label` にフォールバックするためである（React Ariaが採用しているテクニック）。

```html
<!-- aria-labelledby が優先し aria-label は無視される -->
<input
  type="text"
  aria-label="これは無視される"
  aria-labelledby="YOUR_VISIBLE_LABEL_ID"
/>

<!-- 例外: 自分自身を参照する場合 aria-label + 他要素のラベルが結合される -->
<div
  role="spinbutton"
  id="YOUR_SPIN_ID"
  aria-label="年, "
  aria-labelledby="YOUR_SPIN_ID YOUR_DATE_LABEL_ID"
>
  2026
</div>
<span id="YOUR_DATE_LABEL_ID">日付</span>
<!-- アクセシブルネーム = "年,  日付" -->
```

### 2.2 ネイティブHTML によるアクセシブルネーム付与（最優先）

ARIAを使用する前に、以下のネイティブ手段を最優先で検討する。

| HTML要素 | アクセシブルネームの付与方法 |
| --- | --- |
| `<button>` | 要素内テキスト |
| `<a href="...">` | 要素内テキスト |
| `<input>` / `<select>` / `<textarea>` | `<label>` 要素 |
| `<img>` | `alt` 属性 |
| `<table>` | `<caption>` 要素 |
| `<figure>` | `<figcaption>` 要素 |
| `<fieldset>` | `<legend>` 要素 |
| `<iframe>` | `title` 属性 |

```html
<!-- 良い例：ネイティブ手段 -->
<label for="YOUR_EMAIL_ID">メールアドレス</label>
<input type="email" id="YOUR_EMAIL_ID" />

<!-- 良い例：ボタンのテキスト -->
<button type="submit">送信する</button>

<!-- 良い例：リンクテキストで遷移先が明確 -->
<p>詳しくは<a href="https://example.com/docs">公式ドキュメント</a>をご覧ください。</p>

<!-- 悪い例：リンクテキストが不明確 -->
<p>詳しくは<a href="https://example.com/docs">こちら</a>をご覧ください。</p>
```

### 2.3 aria-labelledby（DOMに可視テキストが存在する場合）

ラベルとして使用するテキストがDOM上に表示されている場合、`aria-label` ではなく `aria-labelledby` を使用する。

**`aria-labelledby` の利点**：
- 晴眼者とスクリーンリーダー利用者に同一の情報を提供できる
- ラベルテキスト変更時に `aria-label` の同期忘れが発生しない
- 複数のIDをスペース区切りで指定し、ラベルを結合できる

```html
<!-- ダイアログのラベル付け -->
<dialog aria-labelledby="YOUR_DIALOG_TITLE_ID">
  <h2 id="YOUR_DIALOG_TITLE_ID">ログイン</h2>
  <form>
    <label for="YOUR_DIALOG_EMAIL_ID">メールアドレス</label>
    <input type="email" id="YOUR_DIALOG_EMAIL_ID" />
    <label for="YOUR_DIALOG_PASSWORD_ID">パスワード</label>
    <input type="password" id="YOUR_DIALOG_PASSWORD_ID" />
    <button type="submit">ログイン</button>
  </form>
</dialog>

<!-- 複数の nav がある場合のラベル付け -->
<nav aria-labelledby="YOUR_NAV_MAIN_HEADING_ID">
  <h2 id="YOUR_NAV_MAIN_HEADING_ID">メインメニュー</h2>
  <!-- ナビゲーション項目 -->
</nav>
<nav aria-labelledby="YOUR_NAV_FOOTER_HEADING_ID">
  <h2 id="YOUR_NAV_FOOTER_HEADING_ID">フッターメニュー</h2>
  <!-- ナビゲーション項目 -->
</nav>
```

### 2.4 aria-label（DOMに可視テキストが存在しない場合のみ）

`aria-label` は「表示領域の制約等でどうしてもテキストを配置できない場合のみ」使用する最終手段。

**`aria-label` の制約（使用前に認識必須）**：
- スクリーンリーダー利用者にのみ情報が伝わり、晴眼者には見えない
- ブラウザの検索機能（Ctrl+F）で検索できない
- テキスト選択・コピーができない
- `aria-label` はブラウザ、翻訳エンジン、表示条件によって翻訳可否や精度に差が出る。翻訳、検索、可視性の観点で信頼性が不安定なため、可能な限り可視テキストと `aria-labelledby`（または `visually-hidden` テキスト）を優先する

**適正な使用例**：

```html
<!-- アイコンのみのボタン（テキスト配置が不可能な場合） -->
<button aria-label="閉じる">
  <svg aria-hidden="true" focusable="false" width="24" height="24">
    <path d="M19 6.41L17.59 5 12 10.59 6.41 5 5 6.41 10.59 12 5 17.59 6.41 19 12 13.41 17.59 19 19 17.59 13.41 12z" />
  </svg>
</button>

<!-- 検索フォーム（ラベルテキストを配置する余地がない場合） -->
<form role="search">
  <input type="search" aria-label="サイト内検索" />
  <button type="submit" aria-label="検索">
    <svg aria-hidden="true"><!-- 虫眼鏡アイコン --></svg>
  </button>
</form>
```

**不適正な使用例**：

```html
<!-- 悪い例：可視テキストがあるのに aria-label で上書き -->
<a href="https://example.com/docs" aria-label="React 公式ドキュメント">こちら</a>
<!-- → リンクテキスト自体を「React 公式ドキュメント」に変更すべき -->

<!-- 悪い例：label 要素で代替できるのに aria-label を使用 -->
<input type="email" aria-label="メールアドレス" />
<!-- → <label for="...">メールアドレス</label> を使用すべき -->

<!-- 悪い例：補足説明に aria-label を使用 -->
<button aria-label="送信（この操作は取り消せません）">送信</button>
<!-- → aria-describedby を使用すべき -->
```

### 2.5 aria-label を使用してよい条件（判定フロー）

以下の順で判定し、すべて「いいえ」の場合のみ `aria-label` を使用する。

1. ネイティブHTML（`<label>`, `alt`, `<caption>`, `<legend>`, テキストコンテンツ）で代替できるか → **はい** → ネイティブHTMLを使用
2. DOM上に参照可能な可視テキストが存在するか → **はい** → `aria-labelledby` を使用
3. 補足説明であり、名前ではなく説明が必要か → **はい** → `aria-describedby` を使用
4. すべて「いいえ」 → `aria-label` を使用（対話型要素・ウィジェット・ランドマーク・画像・iframe のみ）

### 2.6 aria-label が機能しない要素

以下のロールの要素では `aria-label` は読み上げられない、または予期しない動作をする。使用禁止。

`code`, `caption`, `deletion`, `emphasis`, `generic`, `insertion`, `mark`, `paragraph`, `presentation` / `none`, `strong`, `subscript`, `superscript`, `suggestion`, `term`, `time`

```html
<!-- 禁止：非対話型要素に aria-label -->
<span aria-label="重要な注意事項">注意</span>
<p aria-label="段落の説明">テキスト</p>
<div aria-label="汎用コンテナの説明">内容</div>
```

### 2.7 aria-hidden

`aria-hidden="true"` を設定した要素は、その子要素を含めてATの読み上げ対象から除外される。

**使用する場合**：
- 装飾目的のアイコン・画像
- テキストとセットの冗長なアイコン
- 視覚的に表示されるが、ATには不要なコンテンツ

**使用禁止**：
- フォーカス可能な要素（ARIAの第4ルールに違反）
- `aria-label` と `aria-hidden` の同時設定（`aria-hidden` が優先され `aria-label` は無効化）
- ページ全体の主要コンテンツ

```html
<!-- 良い例：装飾アイコンを読み上げから除外 -->
<a href="https://example.com/help">
  <svg aria-hidden="true"><!-- ヘルプアイコン --></svg>
  ヘルプ
</a>

<!-- 良い例：テキストとセットのアイコン -->
<button>
  <i class="material-icons" aria-hidden="true">favorite</i>
  お気に入りに追加
</button>

<!-- 禁止：フォーカス可能な要素を隠す -->
<button aria-hidden="true">非表示ボタン</button>
```

### 2.8 aria-describedby と aria-description

補足説明・ヒント・エラーメッセージ等、名前ではなく追加情報を関連付ける場合に使用する。

```html
<!-- パスワード要件の説明 -->
<label for="YOUR_PASSWORD_ID">パスワード</label>
<input
  type="password"
  id="YOUR_PASSWORD_ID"
  aria-describedby="YOUR_PASSWORD_HINT_ID"
/>
<p id="YOUR_PASSWORD_HINT_ID">8文字以上、英数字と記号を含めてください。</p>

<!-- エラーメッセージの関連付け -->
<label for="YOUR_EMAIL_FIELD_ID">メールアドレス</label>
<input
  type="email"
  id="YOUR_EMAIL_FIELD_ID"
  aria-invalid="true"
  aria-describedby="YOUR_EMAIL_ERROR_ID"
/>
<p id="YOUR_EMAIL_ERROR_ID" role="alert">有効なメールアドレスを入力してください。</p>
```

---

## 3. ランドマーク

### 3.1 ページ構造テンプレート

```html
<body>
  <header> <!-- banner ランドマーク -->
    <nav aria-labelledby="YOUR_GLOBAL_NAV_HEADING_ID">
      <h2 id="YOUR_GLOBAL_NAV_HEADING_ID" class="visually-hidden">グローバルナビゲーション</h2>
      <!-- ナビゲーション -->
    </nav>
  </header>

  <nav aria-labelledby="YOUR_BREADCRUMB_HEADING_ID">
    <h2 id="YOUR_BREADCRUMB_HEADING_ID" class="visually-hidden">パンくずリスト</h2>
    <ol>
      <li><a href="/">ホーム</a></li>
      <li><a href="/products">製品一覧</a></li>
      <li aria-current="page">製品詳細</li>
    </ol>
  </nav>

  <main> <!-- main ランドマーク（ページに1つのみ） -->
    <h1>ページタイトル</h1>
    <article>
      <!-- メインコンテンツ -->
    </article>
  </main>

  <aside aria-labelledby="YOUR_SIDEBAR_HEADING_ID"> <!-- complementary ランドマーク -->
    <h2 id="YOUR_SIDEBAR_HEADING_ID">関連情報</h2>
    <!-- サイドバー -->
  </aside>

  <footer> <!-- contentinfo ランドマーク -->
    <nav aria-labelledby="YOUR_FOOTER_NAV_HEADING_ID">
      <h2 id="YOUR_FOOTER_NAV_HEADING_ID" class="visually-hidden">フッターナビゲーション</h2>
      <!-- フッターリンク -->
    </nav>
  </footer>
</body>
```

### 3.2 ランドマーク規則

| 規則 | 詳細 |
| --- | --- |
| `<main>` は1つのみ | ページに複数の `<main>` を配置しない |
| 同一ロールが複数存在する場合はラベル必須 | `<nav>` が2つ以上ある場合、`aria-labelledby` または `aria-label` で区別する |
| 同一種別ランドマークの不要な入れ子を禁止 | `<main>` の中に `<main>` を入れない。`<header>` または `<footer>` 内の `<nav>` など、意味的に妥当な入れ子は許可する |
| 見出し要素と併用する | 各ランドマーク内に適切な見出しレベルの `<h2>`〜`<h6>` を配置する |

---

## 4. ARIA状態属性

### 4.1 主要な状態属性一覧

| 属性 | 値 | 用途 | 使用パターン |
| --- | --- | --- | --- |
| `aria-expanded` | `true` / `false` | 開閉可能な要素の展開状態 | アコーディオン、ドロップダウン、ディスクロージャー |
| `aria-checked` | `true` / `false` / `mixed` | チェック状態 | カスタムチェックボックス、スイッチ |
| `aria-pressed` | `true` / `false` / `mixed` | トグルボタンの押下状態 | ツールバーのトグルボタン |
| `aria-selected` | `true` / `false` | 選択状態 | タブ、リストボックスのオプション |
| `aria-current` | `page` / `step` / `location` / `date` / `time` / `true` | 現在位置 | ナビゲーション、パンくず、ステッパー |
| `aria-invalid` | `true` / `false` / `grammar` / `spelling` | 入力値の妥当性 | フォームバリデーション |
| `aria-disabled` | `true` / `false` | 操作不可状態 | 条件未達のボタン |
| `aria-hidden` | `true` / `false` | AT非表示 | 装飾要素 |
| `aria-busy` | `true` / `false` | 更新中状態 | 非同期コンテンツ読み込み中 |
| `aria-modal` | `true` / `false` | モーダル状態 | `<dialog>` 、カスタムモーダル |

### 4.2 状態属性のJavaScript制御

```javascript
// アコーディオンの展開/折りたたみ
function toggleAccordion(trigger) {
  const panel = document.getElementById(trigger.getAttribute('aria-controls'));
  const isExpanded = trigger.getAttribute('aria-expanded') === 'true';

  trigger.setAttribute('aria-expanded', String(!isExpanded));
  panel.hidden = isExpanded;
}

// カスタムチェックボックスのトグル
function toggleCheckbox(checkbox) {
  const isChecked = checkbox.getAttribute('aria-checked') === 'true';
  checkbox.setAttribute('aria-checked', String(!isChecked));
}
```

### 4.3 aria-disabled と HTML disabled の使い分け

| 項目 | `aria-disabled="true"` | HTML `disabled` |
| --- | --- | --- |
| フォーカス可能 | はい（Tab で到達可能） | いいえ（Tab でスキップされる） |
| フォーム送信 | 送信される | 送信されない |
| スタイル | 開発者が CSS で制御 | ブラウザデフォルトのグレーアウト |
| イベント発火 | する（JavaScript で抑制が必要） | しない |
| 推奨用途 | AT利用者にも存在を認識させたい場合 | 完全に操作不可にする場合 |

```html
<!-- aria-disabled：存在は認識させるが操作は抑制 -->
<button aria-disabled="true" class="btn-disabled">次へ（入力完了後に有効化）</button>

<!-- HTML disabled：完全に無効化 -->
<button disabled>次へ</button>
```

```javascript
// aria-disabled の場合はクリックとキーイベントを抑制する
document.querySelector('[aria-disabled="true"]').addEventListener('click', (event) => {
  if (event.currentTarget.getAttribute('aria-disabled') === 'true') {
    event.preventDefault();
    event.stopPropagation();
  }
});
```

---

## 5. キーボードナビゲーション

### 5.1 tabindex の規則

| tabindex 値 | 動作 | 使用指針 |
| --- | --- | --- |
| 未指定 | ネイティブ要素のデフォルト動作 | `<button>`, `<a href>`, `<input>` 等は自動的にフォーカス可能 |
| `0` | DOMの順序でタブ順序に追加 | カスタムインタラクティブ要素に使用 |
| `-1` | タブ順序から除外、JavaScriptの `.focus()` で到達可能 | ロービングタブインデックス、プログラムによるフォーカス制御に使用 |
| 正の整数 | **使用禁止** | タブ順序を破壊し、保守性とアクセシビリティを損なう |

### 5.2 カスタムボタンのキーボード対応

`<button>` を使用できない場合の最低限の実装：

```html
<div
  role="button"
  tabindex="0"
  class="custom-button"
>
  カスタムボタン
</div>
```

```javascript
document.querySelector('[role="button"]').addEventListener('keydown', (event) => {
  // Enter または Space でアクティベート
  if (event.key === 'Enter' || event.key === ' ') {
    event.preventDefault(); // Space のスクロール防止
    event.currentTarget.click();
  }
});
```

### 5.3 タブパネルの実装（ロービングタブインデックス）

```html
<div role="tablist" aria-label="YOUR_TAB_GROUP_LABEL">
  <button
    role="tab"
    id="YOUR_TAB_1_ID"
    aria-selected="true"
    aria-controls="YOUR_PANEL_1_ID"
    tabindex="0"
  >
    タブ1
  </button>
  <button
    role="tab"
    id="YOUR_TAB_2_ID"
    aria-selected="false"
    aria-controls="YOUR_PANEL_2_ID"
    tabindex="-1"
  >
    タブ2
  </button>
  <button
    role="tab"
    id="YOUR_TAB_3_ID"
    aria-selected="false"
    aria-controls="YOUR_PANEL_3_ID"
    tabindex="-1"
  >
    タブ3
  </button>
</div>

<div
  role="tabpanel"
  id="YOUR_PANEL_1_ID"
  aria-labelledby="YOUR_TAB_1_ID"
  tabindex="0"
>
  タブ1の内容
</div>
<div
  role="tabpanel"
  id="YOUR_PANEL_2_ID"
  aria-labelledby="YOUR_TAB_2_ID"
  tabindex="0"
  hidden
>
  タブ2の内容
</div>
<div
  role="tabpanel"
  id="YOUR_PANEL_3_ID"
  aria-labelledby="YOUR_TAB_3_ID"
  tabindex="0"
  hidden
>
  タブ3の内容
</div>
```

```typescript
function setupTabs(): void {
  const tablist = document.querySelector<HTMLElement>('[role="tablist"]');
  if (tablist == null) {
    return;
  }

  const tabs = Array.from(tablist.querySelectorAll<HTMLElement>('[role="tab"]'));
  if (tabs.length === 0) {
    return;
  }

  tablist.addEventListener('keydown', (event: KeyboardEvent) => {
    const target = event.target;
    if (!(target instanceof HTMLElement)) {
      return;
    }

    const currentTab = target.closest<HTMLElement>('[role="tab"]');
    if (currentTab == null) {
      return;
    }

    const currentIndex = tabs.indexOf(currentTab);
    if (currentIndex === -1) {
      return;
    }

    let nextIndex: number | null = null;

    switch (event.key) {
      case 'ArrowRight':
        nextIndex = (currentIndex + 1) % tabs.length;
        break;
      case 'ArrowLeft':
        nextIndex = (currentIndex - 1 + tabs.length) % tabs.length;
        break;
      case 'Home':
        nextIndex = 0;
        break;
      case 'End':
        nextIndex = tabs.length - 1;
        break;
      default:
        return;
    }

    event.preventDefault();
    const nextTab = tabs[nextIndex];
    if (nextTab == null) {
      return;
    }

    activateTab(tabs, nextTab);
  });

  tabs.forEach((tab) => {
    tab.addEventListener('click', () => {
      activateTab(tabs, tab);
    });
  });
}

function activateTab(tabs: HTMLElement[], nextTab: HTMLElement): void {
  tabs.forEach((tab) => {
    tab.setAttribute('aria-selected', 'false');
    tab.setAttribute('tabindex', '-1');

    const panelId = tab.getAttribute('aria-controls');
    if (panelId == null) {
      return;
    }

    const panel = document.getElementById(panelId);
    if (panel == null) {
      return;
    }

    panel.hidden = true;
  });

  nextTab.setAttribute('aria-selected', 'true');
  nextTab.setAttribute('tabindex', '0');
  nextTab.focus();

  const nextPanelId = nextTab.getAttribute('aria-controls');
  if (nextPanelId == null) {
    return;
  }

  const nextPanel = document.getElementById(nextPanelId);
  if (nextPanel == null) {
    return;
  }

  nextPanel.hidden = false;
}

setupTabs();
```

**タブのキーボード操作仕様**：

| キー | 動作 |
| --- | --- |
| Tab | タブリストにフォーカスを移動（選択中のタブに到達） |
| → / ← | 次 / 前のタブに移動（末尾↔先頭でループ） |
| Home | 最初のタブに移動 |
| End | 最後のタブに移動 |
| Space / Enter | タブをアクティベート（自動アクティベーション時は不要） |

### 5.4 ロービングタブインデックスの適用対象

| ウィジェット | フォーカス移動キー | 補足 |
| --- | --- | --- |
| タブリスト（`tablist`） | ← → | 水平配置の場合。垂直の場合は ↑ ↓ |
| メニュー（`menu` / `menubar`） | ↑ ↓ / ← → | menubar は ← →、menu は ↑ ↓ |
| ラジオグループ（`radiogroup`） | ↑ ↓ / ← → | 方向は配置による |
| ツリービュー（`tree`） | ↑ ↓ | → で展開、← で折りたたみ |
| ツールバー（`toolbar`） | ← → | 水平配置の場合 |

---

## 6. フォーカス管理

### 6.1 フォーカスインジケーター

WCAG 2.2 達成基準 2.4.11（Focus Not Obscured）および 2.4.7（Focus Visible）に対応する。

```css
/* フォーカスインジケーター（:focus-visible を使用） */
:focus-visible {
  outline: 2px solid var(--color-focus, #1a73e8);
  outline-offset: 2px;
}

/* マウスクリック時はアウトラインを非表示（:focus-visible が対応） */
:focus:not(:focus-visible) {
  outline: none;
}

/* 高コントラストモード対応 */
@media (forced-colors: active) {
  :focus-visible {
    outline: 2px solid Highlight;
    outline-offset: 2px;
  }
}
```

**フォーカスインジケーターを `outline: none` で消してはならない**。カスタムスタイルを適用する場合でも、`:focus-visible` に対して視認可能なインジケーターを必ず提供する。

### 6.2 フォーカス移動のシナリオ

| シナリオ | フォーカス移動先 |
| --- | --- |
| ダイアログを開く | ダイアログ内の最初のフォーカス可能要素、またはダイアログ自体 |
| ダイアログを閉じる | ダイアログを開いたトリガー要素 |
| 要素を削除する | 削除された要素の前後の要素、またはリストの先頭 |
| 動的コンテンツの挿入 | 挿入されたコンテンツの先頭（必要な場合のみ） |
| バリデーションエラー | 最初のエラーが発生したフォーム要素 |
| ページ内リンク | リンク先の見出しまたはコンテナ |

### 6.3 フォーカストラップ（モーダルダイアログ）

ネイティブ `<dialog>` 要素を使用すれば、ブラウザがフォーカストラップを自動的に処理する。

```html
<dialog id="YOUR_MODAL_ID" aria-labelledby="YOUR_MODAL_TITLE_ID">
  <h2 id="YOUR_MODAL_TITLE_ID">確認</h2>
  <p>この操作を実行しますか？</p>
  <div class="dialog-actions">
    <button id="YOUR_MODAL_CANCEL_ID">キャンセル</button>
    <button id="YOUR_MODAL_CONFIRM_ID">実行</button>
  </div>
</dialog>
```

```javascript
const dialog = document.getElementById('YOUR_MODAL_ID');
const openButton = document.getElementById('YOUR_OPEN_BUTTON_ID');
const cancelButton = document.getElementById('YOUR_MODAL_CANCEL_ID');

// 開く（.showModal() でモーダルとして表示、フォーカストラップが自動適用）
openButton.addEventListener('click', () => {
  dialog.showModal();
});

// 閉じる
cancelButton.addEventListener('click', () => {
  dialog.close();
});

// Escape キーで閉じる（<dialog> はデフォルトで対応）
// ブラウザによってはフォーカスが自動復帰しないため明示的に制御する
dialog.addEventListener('close', () => {
  openButton.focus();
});
```

カスタムモーダルでフォーカストラップを実装する場合（`<dialog>` を使用できない環境）：

```javascript
function trapFocus(container) {
  const focusableElements = container.querySelectorAll(
    'a[href], button:not([disabled]), input:not([disabled]), select:not([disabled]), textarea:not([disabled]), [tabindex]:not([tabindex="-1"])'
  );
  const firstFocusable = focusableElements[0];
  const lastFocusable = focusableElements[focusableElements.length - 1];

  container.addEventListener('keydown', (event) => {
    if (event.key !== 'Tab') return;

    if (event.shiftKey) {
      // Shift + Tab：最初の要素から逆移動したら最後の要素へ
      if (document.activeElement === firstFocusable) {
        event.preventDefault();
        lastFocusable.focus();
      }
    } else {
      // Tab：最後の要素から移動したら最初の要素へ
      if (document.activeElement === lastFocusable) {
        event.preventDefault();
        firstFocusable.focus();
      }
    }
  });

  // 開いた時点で最初のフォーカス可能要素にフォーカス
  firstFocusable.focus();
}
```

### 6.4 フォーカス不可視の防止（WCAG 2.2 達成基準 2.4.11）

フォーカスが当たっている要素が、スティッキーヘッダーやフッター、モードレスダイアログ等に隠されてはならない。

```css
/* スティッキーヘッダーがある場合のスクロールマージン */
:target,
[tabindex="-1"]:focus {
  scroll-margin-top: 80px; /* ヘッダーの高さ + 余白 */
}

/* スキップリンク先や見出しへのフォーカス移動時 */
main :focus {
  scroll-margin-top: 80px;
  scroll-margin-bottom: 60px; /* スティッキーフッターの高さ + 余白 */
}
```

---

## 7. ライブリージョン

### 7.1 aria-live とロールの対応

| 方法 | 通知レベル | 用途 |
| --- | --- | --- |
| `aria-live="polite"` | 低（現在の読み上げ完了後に通知） | ステータス更新、検索結果件数、保存完了 |
| `aria-live="assertive"` | 高（即時割り込み通知） | エラーアラート、セッション期限切れ警告 |
| `role="status"` | `aria-live="polite"` 相当 | 状態メッセージ |
| `role="alert"` | `aria-live="assertive"` 相当 | 警告・エラーメッセージ |
| `role="log"` | `aria-live="polite"` 相当 | チャットログ、イベントログ |
| `role="timer"` | ライブリージョンではない | カウントダウン（頻繁な更新はATに負荷） |

### 7.2 ライブリージョンの実装規則

**必須：先にDOMに空の要素を配置し、後からコンテンツを挿入する**。ページ読み込み時に `aria-live` と同時にコンテンツを配置しても通知されない。

```html
<!-- 先にDOMに配置（空の状態） -->
<div id="YOUR_STATUS_REGION_ID" role="status" aria-live="polite"></div>
<div id="YOUR_ERROR_REGION_ID" role="alert" aria-live="assertive"></div>
```

```javascript
// コンテンツを後から挿入→ATが通知
function showStatus(message) {
  const region = document.getElementById('YOUR_STATUS_REGION_ID');
  region.textContent = message;
}

function showError(message) {
  const region = document.getElementById('YOUR_ERROR_REGION_ID');
  region.textContent = message;
}

// 使用例
showStatus('3件の検索結果が見つかりました。');
showError('セッションが切れました。再ログインしてください。');
```

### 7.3 ライブリージョンの注意事項

- `aria-live="assertive"` は緊急時のみ使用する。乱用するとATの読み上げが中断され続けてユーザー体験を損なう
- 頻繁な更新（1秒未満の間隔）はATに過大な負荷をかける。スロットリング（最短500ms間隔等）を実装する
- `aria-atomic="true"` を設定すると、リージョン内のコンテンツが部分的に変更されても全体が読み上げられる
- `aria-busy="true"` を設定中は更新通知が抑制される。非同期読み込み中に使用する

```javascript
// 非同期コンテンツ読み込み中のaria-busy制御
async function loadContent(container) {
  container.setAttribute('aria-busy', 'true');

  try {
    const data = await fetch('/api/data');
    const html = await data.text();
    container.innerHTML = html;
  } finally {
    container.setAttribute('aria-busy', 'false');
  }
}
```

---

## 8. フォームのアクセシビリティ

### 8.1 ラベルの関連付け

すべてのフォーム要素に明示的なラベルを関連付ける。`placeholder` はラベルの代替にならない。

```html
<!-- 明示的ラベル（推奨） -->
<label for="YOUR_NAME_ID">氏名</label>
<input type="text" id="YOUR_NAME_ID" autocomplete="name" />

<!-- グループ化（ラジオボタン、チェックボックス群） -->
<fieldset>
  <legend>連絡方法</legend>
  <label>
    <input type="radio" name="contact" value="email" />
    メール
  </label>
  <label>
    <input type="radio" name="contact" value="phone" />
    電話
  </label>
</fieldset>
```

### 8.2 エラー表示の実装手順

1. エラーが発生したフィールドに `aria-invalid="true"` を設定
2. エラーメッセージ要素を `aria-describedby` で関連付け
3. エラーメッセージ要素に `role="alert"` を設定（即時通知が必要な場合）
4. エラー解消時に `aria-invalid` を削除し、エラーメッセージを空にする

```html
<label for="YOUR_FORM_EMAIL_ID">メールアドレス <span aria-hidden="true">*</span></label>
<input
  type="email"
  id="YOUR_FORM_EMAIL_ID"
  required
  aria-required="true"
  aria-invalid="true"
  aria-describedby="YOUR_FORM_EMAIL_ERROR_ID YOUR_FORM_EMAIL_HINT_ID"
/>
<p id="YOUR_FORM_EMAIL_ERROR_ID" role="alert">有効なメールアドレスを入力してください。</p>
<p id="YOUR_FORM_EMAIL_HINT_ID">例：user@example.com</p>
```

```javascript
function validateEmail(input) {
  const errorElement = document.getElementById('YOUR_FORM_EMAIL_ERROR_ID');

  if (!input.validity.valid) {
    input.setAttribute('aria-invalid', 'true');
    errorElement.textContent = '有効なメールアドレスを入力してください。';
  } else {
    input.removeAttribute('aria-invalid');
    errorElement.textContent = '';
  }
}
```

### 8.3 必須フィールド

ネイティブフォーム要素では HTML の `required` 属性を使用し、`aria-required` は原則不要とする。`aria-required="true"` はネイティブ `required` を使えないカスタムコントロールに限定して使用する。視覚的な必須マーク（`*`）には `aria-hidden="true"` を付与し、二重読み上げを防止する。

```html
<label for="YOUR_REQUIRED_FIELD_ID">会社名 <span aria-hidden="true">*</span></label>
<input type="text" id="YOUR_REQUIRED_FIELD_ID" required />
```

### 8.4 autocomplete 属性

WCAG 2.2 達成基準 1.3.5（入力目的の特定）に対応する。個人情報を入力するフィールドには適切な `autocomplete` 値を設定する。

| autocomplete 値 | 用途 |
| --- | --- |
| `name` | 氏名 |
| `email` | メールアドレス |
| `tel` | 電話番号 |
| `postal-code` | 郵便番号 |
| `address-line1` | 住所1行目 |
| `cc-number` | クレジットカード番号 |
| `current-password` | 現在のパスワード |
| `new-password` | 新しいパスワード |
| `one-time-code` | ワンタイムパスワード |

---

## 9. 画像とメディア

### 9.1 alt 属性の判定

| 画像の種類 | alt の指定 | 例 |
| --- | --- | --- |
| 情報を伝える画像 | 画像が伝える情報を簡潔に記述 | `alt="2025年売上グラフ：前年比120%"` |
| 装飾画像 | 空の alt | `alt=""` |
| 機能的画像（ボタン・リンク内） | 機能・遷移先を記述 | `alt="検索"`, `alt="ホームに戻る"` |
| テキスト画像 | 画像内のテキストを記述 | `alt="特別セール 50%OFF"` |
| 複雑な画像（グラフ・図表） | 概要を alt に、詳細を本文またはリンク先に記述 | `alt="年間売上推移（詳細は下記表を参照）"` |
| グループ画像 | 代表の1枚に説明、他は空 | 先頭に `alt="チーム写真"`, 他は `alt=""` |

### 9.2 SVGのアクセシビリティ

```html
<!-- インライン SVG -->
<svg role="img" aria-labelledby="YOUR_SVG_TITLE_ID YOUR_SVG_DESC_ID" xmlns="http://www.w3.org/2000/svg">
  <title id="YOUR_SVG_TITLE_ID">棒グラフ</title>
  <desc id="YOUR_SVG_DESC_ID">2025年の四半期別売上を示す棒グラフ</desc>
  <!-- SVG コンテンツ -->
</svg>

<!-- 装飾 SVG -->
<svg aria-hidden="true" focusable="false">
  <!-- SVG コンテンツ -->
</svg>
```

### 9.3 動画・音声のアクセシビリティ

```html
<video controls>
  <source src="YOUR_VIDEO_SRC" type="video/mp4" />
  <track kind="captions" src="YOUR_CAPTIONS_SRC" srclang="ja" label="日本語字幕" default />
  <track kind="descriptions" src="YOUR_DESCRIPTIONS_SRC" srclang="ja" label="音声解説" />
</video>
```

---

## 10. テーブルのアクセシビリティ

### 10.1 データテーブル

```html
<table>
  <caption>2025年度 四半期別売上</caption>
  <thead>
    <tr>
      <th scope="col">四半期</th>
      <th scope="col">売上（万円）</th>
      <th scope="col">前年比</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">Q1</th>
      <td>1,200</td>
      <td>105%</td>
    </tr>
    <tr>
      <th scope="row">Q2</th>
      <td>1,500</td>
      <td>112%</td>
    </tr>
  </tbody>
</table>
```

### 10.2 テーブルの規則

- レイアウト目的で `<table>` を使用しない（CSS Grid / Flexbox を使用する）
- やむを得ずレイアウトテーブルを使用する場合は `role="presentation"` を設定する
- 列ヘッダーには `scope="col"`、行ヘッダーには `scope="row"` を指定する
- 複雑なヘッダー構造には `headers` 属性でセルとヘッダーを明示的に関連付ける
- ソート可能な列には `aria-sort` 属性を使用する（`ascending` / `descending` / `none` / `other`）

---

## 11. SPA（Single Page Application）のアクセシビリティ

### 11.1 ルート変更時のフォーカス管理

SPAでのページ遷移はブラウザのデフォルトのフォーカス・読み上げ動作が発生しない。明示的な対応が必要。

**方法1：ページ見出しにフォーカスを移動する**

```javascript
function onRouteChange() {
  const heading = document.querySelector('main h1');
  if (heading) {
    heading.setAttribute('tabindex', '-1');
    heading.focus();
    // フォーカスが外れた後にtabindexを削除（DOM汚染防止）
    heading.addEventListener('blur', () => {
      heading.removeAttribute('tabindex');
    }, { once: true });
  }
}
```

**方法2：ライブリージョンでページ遷移を通知する**

```html
<div id="YOUR_ROUTE_ANNOUNCER_ID" role="status" aria-live="polite" class="visually-hidden"></div>
```

```javascript
function onRouteChange(pageTitle) {
  const announcer = document.getElementById('YOUR_ROUTE_ANNOUNCER_ID');
  announcer.textContent = `${pageTitle}に移動しました。`;
}
```

### 11.2 非同期コンテンツ読み込み

```html
<section aria-busy="true" aria-label="検索結果">
  <p>読み込み中...</p>
</section>
```

```javascript
async function loadSearchResults(query) {
  const container = document.querySelector('[aria-label="検索結果"]');
  container.setAttribute('aria-busy', 'true');

  try {
    const results = await fetch(`/api/search?q=${encodeURIComponent(query)}`);
    const data = await results.json();
    container.innerHTML = renderResults(data);

    // 件数を通知
    const status = document.getElementById('YOUR_STATUS_REGION_ID');
    status.textContent = `${data.length}件の検索結果が見つかりました。`;
  } finally {
    container.setAttribute('aria-busy', 'false');
  }
}
```

### 11.3 無限スクロール

無限スクロールはキーボード・AT利用者のアクセシビリティを大きく損なう。以下を必須とする。

- 自動読み込みではなく明示的な「もっと読み込む」ボタンを設置
- 読み込み後、フォーカスを新しいコンテンツの先頭に移動
- 件数やページ情報をライブリージョンで通知
- フッターへのスキップリンクを提供

```html
<ul id="YOUR_RESULTS_LIST_ID" aria-label="記事一覧">
  <!-- 記事リスト -->
</ul>
<button id="YOUR_LOAD_MORE_ID">さらに10件を読み込む</button>
<div id="YOUR_LOAD_STATUS_ID" role="status" aria-live="polite"></div>
```

---

## 12. 動的アニメーションと `prefers-reduced-motion`

### 12.1 プログレッシブ・エンハンスメント（デフォルトでアニメーションなし）

```css
/* デフォルト：アニメーションなし */
.animated-element {
  transition: none;
}

/* アニメーション許可時のみ動きを追加 */
@media (prefers-reduced-motion: no-preference) {
  .animated-element {
    transition: transform 0.3s ease, opacity 0.3s ease;
  }
}
```

### 12.2 抑制対象と維持対象

| 抑制する | 維持する |
| --- | --- |
| パララックスエフェクト | 必須のローディングインジケーター |
| 自動再生アニメーション | ユーザーが明示的に開始した動作 |
| 大きなページ遷移トランジション | 意味のある状態変化（チェック→完了等） |
| スクロール連動アニメーション | `opacity` の穏やかな変化 |
| 装飾的なパーティクル・背景動画 | — |

---

## 13. スキップナビゲーション

WCAG 2.2 達成基準 2.4.1（ブロックスキップ）に対応する。

```html
<body>
  <a href="#YOUR_MAIN_CONTENT_ID" class="skip-link">メインコンテンツへスキップ</a>
  <header>...</header>
  <main id="YOUR_MAIN_CONTENT_ID" tabindex="-1">
    <h1>ページタイトル</h1>
    ...
  </main>
</body>
```

```css
.skip-link {
  position: absolute;
  top: -100%;
  left: 0;
  z-index: 10000;
  padding: 8px 16px;
  background: var(--color-surface, #ffffff);
  color: var(--color-on-surface, #1a1a1a);
  text-decoration: underline;
}

.skip-link:focus {
  top: 0;
}
```

---

## 14. 視覚的に非表示・ATには公開（visually-hidden）

CSS による視覚的非表示クラス。`display: none` や `visibility: hidden` はATからも隠れるため使用しない。

```css
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  clip-path: inset(50%);
  white-space: nowrap;
  border: 0;
}
```

---

## 15. WCAG 2.2 新規達成基準の実装

### 15.1 達成基準 2.4.11 隠されないフォーカス（AA）

フォーカスが当たっている要素が、スティッキーヘッダー・フッター・モードレスダイアログ等に覆い隠されてはならない（部分的な視認でも可）。

対応：フォーカス要素に `scroll-margin-top` / `scroll-margin-bottom` を設定（§6.4 参照）。

### 15.2 達成基準 2.5.7 ドラッグ動作（AA）

ドラッグ操作が前提の機能はすべて、ドラッグせずにシングルポインタで完遂できる代替手段を提供する。

```html
<!-- ドラッグ対応リスト + 代替操作ボタン -->
<ul id="YOUR_SORTABLE_LIST_ID">
  <li draggable="true" id="YOUR_ITEM_1_ID">
    項目1
    <button aria-label="項目1を上へ移動" class="move-up">↑</button>
    <button aria-label="項目1を下へ移動" class="move-down">↓</button>
  </li>
  <li draggable="true" id="YOUR_ITEM_2_ID">
    項目2
    <button aria-label="項目2を上へ移動" class="move-up">↑</button>
    <button aria-label="項目2を下へ移動" class="move-down">↓</button>
  </li>
</ul>
```

### 15.3 達成基準 2.5.8 ターゲットサイズ（AA）

タッチ/クリック可能領域は最小24×24 CSSピクセル。または隣接ターゲットとの間隔が24px以上。

詳細は ui-ux-guidelines.md のタッチターゲットサイズ規定を参照。

### 15.4 達成基準 3.2.6 一貫したヘルプ（A）

ヘルプ機能（FAQ、お問い合わせ、チャットサポート等）への導線は、同一サイト内の全ページで同じ相対位置に配置する。

### 15.5 達成基準 3.3.7 冗長な入力（A）

同一セッション・同一プロセスで以前入力された情報の再入力を求める場合、自動入力するか選択可能にする。ブラウザの `autocomplete` ではこの達成基準を満たさない。サーバーサイドまたはクライアントサイドでの情報保持が必要。

### 15.6 達成基準 3.3.8 アクセシブルな認証（AA）

認知機能テスト（パスワードの記憶、パズルの解読等）に依存しない認証方法を提供する。

**禁止**：
- パスワード入力欄の貼り付け（paste）無効化
- コピー＆ペーストの禁止
- CAPTCHAのみの認証

**必須**：
- `autocomplete="current-password"` の設定（パスワードマネージャーの動作を妨げない）
- WebAuthn / パスキー等の代替認証手段の提供

```html
<!-- autocomplete を設定しパスワードマネージャーを許可 -->
<label for="YOUR_LOGIN_PASSWORD_ID">パスワード</label>
<input
  type="password"
  id="YOUR_LOGIN_PASSWORD_ID"
  autocomplete="current-password"
/>
<!-- paste を無効化しない（JavaScript で preventDefault しない） -->
```

### 15.7 達成基準 2.4.12 隠されないフォーカス（強化）（AAA）

AAレベルの2.4.11では部分的な視認でも可だが、AAAではフォーカスインジケーターの**全体**が他の著者作成コンテンツに覆われてはならない。設定可能なインターフェースの例外もない。

対応：
- スティッキーヘッダー・フッターの高さ分を `scroll-margin-top` / `scroll-margin-bottom` で確保（§6.4 参照）
- トースト・スナックバー等のオーバーレイをフォーカス要素と重ならない位置に配置する
- `z-index` の管理でフォーカスインジケーターが常に最前面になるようにする

### 15.8 達成基準 2.4.13 フォーカスの外観（AAA）

キーボードフォーカスインジケーターが表示されている場合、以下の両方を満たす。

| 要件 | 内容 |
| --- | --- |
| サイズ | フォーカスインジケーターの領域が、コンポーネント外周の2 CSSピクセル幅の境界線以上の面積を持つ |
| コントラスト | フォーカス時と非フォーカス時の同一ピクセル間で3:1以上のコントラスト比 |

例外：フォーカスインジケーターがユーザーエージェント決定かつ著者未修正の場合は対象外。

```css
/* 推奨実装：2.4.13 準拠のフォーカスインジケーター */
:focus-visible {
  outline: 3px solid #0066cc;  /* 3:1以上のコントラストを確保 */
  outline-offset: 2px;
}

/* ダークモード対応 */
@media (prefers-color-scheme: dark) {
  :focus-visible {
    outline-color: #66b3ff;  /* 暗背景でも3:1以上を確保 */
  }
}
```

### 15.9 達成基準 3.3.9 アクセシブルな認証（強化）（AAA）

AAレベルの3.3.8を強化し、オブジェクト認識テスト（画像選択CAPTCHA等）および個人コンテンツの認識も例外としない。許容される例外は以下の2つのみ：

1. **代替手段**：認知機能テストに依存しない別の認証方法が利用可能
2. **メカニズム**：パスワードマネージャーの自動入力やコピー・ペーストが許可されている

3.3.8（AA）との差異：3.3.8ではオブジェクト認識と個人コンテンツの2つが例外として許容されるが、3.3.9ではそれらも例外ではなくなる。

---

## 16. フレームワーク固有のアクセシビリティ考慮事項

React / Vue 等のコンポーネントベースフレームワークでは、本書の規約に加えて以下の点を考慮する。コンポーネント設計の詳細は component-design-patterns.md を参照。

### 16.1 IDの一意性確保

コンポーネントが複数回レンダリングされる場合、ハードコードされたIDは重複する。フレームワークが提供するID生成機能を使用する。

```jsx
// React: useId() による一意なID生成
import { useId } from 'react';

function PasswordField() {
  const id = useId();
  const hintId = `${id}-hint`;
  const inputId = `${id}-input`;

  return (
    <>
      <label htmlFor={inputId}>パスワード</label>
      <input
        type="password"
        id={inputId}
        aria-describedby={hintId}
      />
      <p id={hintId}>8文字以上で入力してください。</p>
    </>
  );
}
```

### 16.2 クライアントサイドルーティングとフォーカス管理

SPAのルート切り替え時には、ページ遷移が発生しないため、以下を実装する：

- `document.title` をルートごとに更新する
- ルート切り替え後に `<main>` または `<h1>` にフォーカスを移動する
- ライブリージョンでページ切り替えを通知する

詳細は§11（SPAアクセシビリティ）を参照。

### 16.3 静的解析ツール

| フレームワーク | ツール | 用途 |
| --- | --- | --- |
| React | `eslint-plugin-jsx-a11y` | JSXのARIA属性・ロール・ラベルの静的検証 |
| Vue | `eslint-plugin-vuejs-accessibility` | Vueテンプレートのアクセシビリティ検証 |
| Svelte | `eslint-plugin-svelte` + a11yルール | Svelteコンポーネントのa11y検証 |
| 共通 | `axe-core` / `@axe-core/react` | ランタイムのアクセシビリティ違反検出 |

### 16.4 アクセシビリティライブラリ

| ライブラリ | 対応フレームワーク | 特徴 |
| --- | --- | --- |
| [React Aria](https://react-spectrum.adobe.com/react-aria/) | React | Adobe製。ARIA APG準拠のフック群。フォーカス管理・キーボード操作・国際化対応 |
| [Radix UI](https://www.radix-ui.com/) | React | ヘッドレスUIライブラリ。ARIAパターン内蔵 |
| [Headless UI](https://headlessui.com/) | React / Vue | Tailwind Labs製。モーダル・リストボックス・メニュー等のアクセシブルコンポーネント |

フレームワーク固有のコンポーネント設計パターン、Props設計原則、状態管理におけるアクセシビリティ考慮事項は component-design-patterns.md で定義する。

---

## 17. テスト

### 17.1 手動テストチェックリスト

| テスト項目 | 確認内容 |
| --- | --- |
| キーボード操作 | Tab / Shift+Tab で全インタラクティブ要素に到達可能。Enter / Space で操作可能 |
| フォーカス順序 | 視覚的な順序と一致。逆戻りや飛びがない |
| フォーカスインジケーター | すべてのフォーカス状態で視認可能なアウトラインが表示される |
| スクリーンリーダー | 見出し・ランドマーク・フォームラベル・ボタン名が正しく読み上げられる |
| 拡大表示 | 200% ズームでコンテンツが切れない、横スクロールが発生しない |
| テキスト間隔 | 行の高さ1.5倍、段落間隔2倍、文字間隔0.12倍、語間隔0.16倍に変更してもコンテンツが消えない |
| カラーモード | ライト / ダーク / 高コントラストで情報が欠落しない |
| 動的コンテンツ | ライブリージョンが正しく通知される。aria-busy が適切に制御される |

### 17.2 自動テストツール

| ツール | 種別 | 用途 |
| --- | --- | --- |
| axe-core / axe DevTools | ブラウザ拡張 + ライブラリ | 自動検出可能なWCAG違反の網羅的チェック |
| Lighthouse | ブラウザ内蔵 | アクセシビリティスコアとWCAG違反レポート |
| eslint-plugin-jsx-a11y | ESLintプラグイン | React JSX のアクセシビリティ静的解析 |
| Pa11y | CLI / CI統合 | 継続的インテグレーションでの自動テスト |
| Markuplint | HTML Linter | HTML構文・ARIAの適合性チェック |

### 17.3 スクリーンリーダーでの確認項目

| 確認項目 | 期待動作 |
| --- | --- |
| ページタイトル | ページ遷移時に `<title>` が読み上げられる |
| 見出し構造 | 見出しナビゲーション（H キー）で論理的な階層が確認できる |
| ランドマーク | ランドマークナビゲーションで main / nav / banner / contentinfo が検出される |
| フォームラベル | 各入力フィールドにフォーカスした際にラベルが読み上げられる |
| ボタン名 | ボタンにフォーカスした際に操作内容が読み上げられる |
| 状態変化 | `aria-expanded`, `aria-checked` 等の変更が通知される |
| エラーメッセージ | バリデーションエラー時に `role="alert"` が即時通知される |
| ライブリージョン | 動的更新が `aria-live` に従って通知される |

---

## 出典・参考情報
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/) - W3C Recommendation, 2023-10-05。ウェブコンテンツアクセシビリティガイドラインの最新版
- [WAI-ARIA 1.2](https://www.w3.org/TR/wai-aria-1.2/) - W3C Recommendation, 2023-06-06。ARIAロール・状態・プロパティの仕様
- [WAI-ARIA 1.3 Editor's Draft](https://w3c.github.io/aria/) - W3C。策定中の次世代仕様。新規ロール（`suggestion`, `comment`, `mark`）、新規属性（`aria-description`, `aria-braillelabel`, `aria-brailleroledescription`）を導入
- [ARIA Authoring Practices Guide (APG)](https://www.w3.org/WAI/ARIA/apg/) - W3C。ARIAウィジェットの設計パターンとキーボード操作仕様
- [ARIA in HTML](https://www.w3.org/TR/html-aria/) - W3C。HTML要素に許可されるARIAロール・属性の定義
- [Accessible Name and Description Computation 1.2](https://www.w3.org/TR/accname-1.2/) - W3C。アクセシブルネームの算出アルゴリズム
- [Using ARIA](https://www.w3.org/TR/using-aria/) - W3C Note。ARIAの5つのルールと実践的ガイダンス
- [MDN - ARIA](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA) - Mozilla。ARIAロール・属性のリファレンス
- [MDN - aria-label](https://developer.mozilla.org/ja/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label) - Mozilla。aria-label属性の仕様と注意事項
- [aria-label を使いすぎない](https://azukiazusa.dev/blog/do-not-use-aria-label-too-much/) - azukiazusa, 2023-06-25。aria-label の制約と代替手段
- [aria-labelとaria-labelledbyを併用する場合とは](https://zenn.dev/uhyo/articles/aria-label-and-labelledby) - uhyo, 2024-09-17。Accessible Name Computation仕様に基づく併用の仕組み
- [aria-labelの適切な利用について調べてみた](https://qiita.com/gilly/items/704680d57aba1381e9d0) - gilly, 2023-12-22。aria-label使用前の選択肢の優先順位
- [フォームのinput要素には表示可能なラベルが必要です](https://dequeuniversity.com/rules/axe/4.3/label-title-only?lang=ja) - Deque University。axe-coreのlabel-title-onlyルール解説
- [aria-labelとaria-hiddenとは](https://help.studio.design/ja/articles/5752183-aria-label-%E3%81%A8-aria-hidden-%E3%81%A8%E3%81%AF) - Studio Help。aria-label/aria-hiddenのユースケース
- [WCAG 2.2の新しい達成基準をざっくりと理解する](https://accessible-usable.net/2023/12/entry_231219.html) - Accessible & Usable, 2023-12-19。WCAG 2.2新規達成基準の日本語解説
- [勧告されたWCAG 2.2において新たに追加された達成基準の解説](https://burnworks.com/news/article/432/) - バーンワークス, 2024-09-24。WCAG 2.2達成基準の解説とフォーカスインジケータの重要性
- [WebAIM Million](https://webaim.org/projects/million/) - WebAIM。100万サイト調査によるARIA使用とエラー率の統計
- [WAI Images Tutorial](https://www.w3.org/WAI/tutorials/images/) - W3C WAI。画像の代替テキスト判定チュートリアル
