---
title: "Design System Tokens Spec (Color / Typography / Control / Spacing)"
description: "デザイントークン仕様 - Color・Typography・Control・Spacingトークン定義 / Design system tokens - color, typography, control, and spacing definitions"
version: "1.0.0"
status: "Stable"
last_updated: "2026-01-25T20:25+09:00"
lang: "ja"
---

# Design System Tokens Spec (Color / Typography / Control / Spacing)

**デザイントークンの命名規則・値・構造の定義（カラー / タイポグラフィ / コントロール / スペーシング）** - UIコンポーネント実装時はこの仕様に準拠すること


---

**スコープと規約**
- 言語/記法：英語、小文字、**ハイフン区切り**、ASCII（`[a-z0-9-]`のみ）
- レイヤー構造：**Primitives → Semantics → Component**（コンポーネントからプリミティブを直接参照しない、セマンティクス経由のみ）
- テーマ：ライト/ダークは**同一トークンキー**で**モードベース**の値切り替え（例：Figma Variables）
- アクセシビリティ：**WCAG 2.2 AA**準拠、本文テキスト ≥ 4.5:1、UIアウトライン ≥ 3:1
- 交換フォーマット：**DTCG**（Design Tokens Community Group）でCSS/iOS/Android等へ連携

---

## 1. 命名原則

- **グローバル接頭辞なし**（`color-`、`font-`等は省略）
- **セマンティックファースト**：例 `text-primary`、`background-surface`、`interactive-background-hover`
- **状態はサフィックス**：`-hover`、`-active`、`-disabled`、`-selected`、`-focus`、`-visited`
- **強度/レベル**：`subtle`、`muted`、`strong`、`inverse`、`raised-1..5`
- **デバイス軸**：必要な場合のみ `-mobile` / `-desktop` を付加
- **数値**：ステップ/値のみに使用：`gray-100..900`、`text-size-16`、`space-24`、`line-height-150`

---

## 2. カラー

### 2.1 プリミティブ

- **グレー（フルスケール）**：`gray-100 … gray-900`（明 → 暗）
- **非グレー（アンカーのみ）**：例 `blue-300|400|500|600|700`（セマンティクスの要求がある場合のみ中間ステップを追加）
- **カラーフォーマット**：OKLCHを主体とし、HEX / RGBAへのフォールバック必須。`@media (color-gamut: p3)` で広色域ディスプレイ向けに `color(display-p3 …)` を条件適用

### 2.2 セマンティクス（例：フルワード、ハイフンケース）

- **テキスト / アイコン**
  `text-primary`、`text-secondary`、`text-tertiary`、`text-muted`、`text-inverse`、
  `text-link`、`text-link-hover`、`text-link-active`、`text-link-visited`、
  `icon-primary`、`icon-secondary`、`icon-muted`、`icon-inverse`
- **背景 / サーフェス**
  `background-canvas`、`background-surface`、`background-subtle`、`background-inverse`、`background-raised-1`、
  `background-selected`、`background-highlight`
- **ボーダー / フォーカス / ディバイダー**
  `border-default`、`border-strong`、`border-subtle`、`border-invalid`、`border-valid`、`border-selected`、
  `focus-ring`、`divider-default`
- **インタラクティブ（汎用）**
  `interactive-label`、`interactive-label-hover`、`interactive-label-active`、
  `interactive-background`、`interactive-background-hover`、`interactive-background-active`、
  `interactive-border`、`interactive-border-hover`
- **ステータス（セマンティック）**
  `status-success-text|background|border`、`status-warning-*`、`status-error-*`、`status-info-*`
  ソフトサーフェス用バリアント：`status-success-background-subtle` 等
- **On-* / オーバーレイ / 選択**
  `on-surface-text`、`on-canvas-text`、`on-brand-text`、
  `overlay-scrim`、`selection-background`、`selection-text`

> **ライト/ダーク**：同一セマンティックキー、モードごとに参照プリミティブを切り替え（例：`text-primary` = ライト→`gray-900` / ダーク→`gray-100`）

### 2.3 コントラスト管理

- `text×background`、`border×background`、`interactive×background` の組み合わせでAA基準（テキスト ≥ 4.5:1、非テキスト ≥ 3:1）を満たす**ペアリングマトリクス**（許可リスト）を維持
- マトリクス外の組み合わせは使用不可

---

## 3. タイポグラフィ

### 3.1 プリミティブ（厳密値）

- **テキストサイズ（rem単位、基準倰16px=1rem）**：`text-size-10|12|14|16|18|20|24|28|32|36|40|48|56|64|72|96`

| px | rem | 用途 |
|---|---|---|
| 10px | 0.625rem | 法的通知のみ |
| 12px | 0.75rem | キャプション、補足 |
| 14px | 0.875rem | 小テキスト、ラベル |
| 16px | 1rem | 本文（ベース） |
| 18px | 1.125rem | 本文（大） |
| 20px | 1.25rem | リード文 |
| 24px | 1.5rem | 見出し（小） |
| 28px | 1.75rem | 見出し |
| 32px | 2rem | 見出し（大） |
| 36px | 2.25rem | ディスプレイ（小） |
| 40px | 2.5rem | ディスプレイ |
| 48px | 3rem | ディスプレイ（大） |
| 56px | 3.5rem | ヒーロー（小） |
| 64px | 4rem | ヒーロー |
| 72px | 4.5rem | ヒーロー（大） |
| 96px | 6rem | 特大見出し |

- **行高さ（単位なし数値）**：`line-height-110|120|130|140|150|160|170|180`

参照：[MDN - line-height](https://developer.mozilla.org/en-US/docs/Web/CSS/line-height)

**単位なし数値を採用する理由**：`%`や`em`で指定すると計算済みの固定値が子要素に継承され、意図しない表示崩れが発生する。単位なし数値は各要素のfont-sizeに対する乗数として再計算されるため、継承問題を回避できる。

| 用途 | 値 |
|---|---|
| 本文 | 1.40 / 1.50 / 1.60 / 1.70 / 1.80 |
| 見出し | 1.10 / 1.20 / 1.30 / 1.40 |
| ディスプレイ | 1.10 / 1.20 / 1.30 |

注：見出し・ディスプレイのデフォルト1.50は大きすぎるため、1.10〜1.40を推奨。複数行表示時にアセンダー/ディセンダーの重なりがないか要確認。

- **レタースペース（em単位）**：`letter-spacing-tight|normal|loose`

参照：
- [WCAG 1.4.12 Text Spacing](https://www.w3.org/WAI/WCAG22/Understanding/text-spacing.html)
- [Tailwind CSS Letter Spacing](https://tailwindcss.com/docs/letter-spacing)

**CJK言語の例外**：WCAG 1.4.12のletter-spacing基準（0.12em以上）は欧文向け。日本語・中国語・韓国語（CJK）は表意文字を使用するため、WCAGで明示的に適用対象外とされている。CJKには欧文より控えめな値を適用する。

| 用途 | 日本語 | 欧文 |
|---|---|---|
| 本文 | 0em / 0.02em / 0.05em | 0.05em / 0.08em / 0.12em |
| 見出し | 0em / 0.01em / 0.02em | 0.02em / 0.05em / 0.08em |
| ディスプレイ | 0em / 0.005em / 0.01em | 0em / 0.02em / 0.05em |
| キャプション・ラベル | 0.02em / 0.03em / 0.05em | 0.05em / 0.08em / 0.1em |

- **ワードスペース（em単位）**

| 用途 | 日本語 | 欧文 |
|---|---|---|
| 全般 | 適用外（単語間スペース不使用） | 0.16em以上（WCAG準拠） |

- **フォントウェイト**：`text-weight-100|200|300|400|500|600|700|800|900`（Thin* / Extra Light / Light / Regular / Medium / Semi Bold / Bold / Extra Bold / Black）*100 (Thin) は定義に含めるが使用しない
- **フォントファミリー**：`text-family-base`、`text-family-code`

### 3.2 セマンティクス（役割 × デバイス）

- サイズ（例）：
  `text-size-display-1-mobile|desktop`、`text-size-headline-1-mobile|desktop`、`text-size-title-1-mobile|desktop`、
  `text-size-body-1-mobile|desktop`、`text-size-body-2-mobile|desktop`、
  `text-size-label-1-mobile|desktop`、`text-size-caption-mobile|desktop`
- 行高さ / レタースペース（例）：
  `line-height-body-1-mobile|desktop`、`line-height-headline-1-mobile|desktop`、
  `letter-spacing-label-1-mobile|desktop`

**推奨マッピング（編集可）：**
- `text-size-body-1-mobile → 1rem` / `…-desktop → 1.125rem`
- `text-size-body-2-mobile → 0.875rem` / `…-desktop → 1rem`
- `text-size-title-1-mobile → 1.25rem` / `…-desktop → 1.5rem`
- `text-size-headline-1-mobile → 2rem` / `…-desktop → 2.25rem`
- `text-size-display-1-mobile → 2.5rem` / `…-desktop → 4rem`
- `line-height-body-1-mobile → 1.50` / `…-desktop → 1.60`
- 見出し：1.10–1.40、本文：1.50–1.60、ラベル/キャプション：1.40–1.50

### 3.3 実装

- **デバイス固有キー**：デフォルトのセマンティックエイリアスは `-mobile` を参照、メディア/コンテナクエリで `-desktop` に切り替え
- **フルイドサイジング**：適切な場合は `clamp()` で連続スケーリング、エッジケースのみデバイスキーと併用

---

## 4. コントロール（高さ / パディング / 角丸）

### 4.1 サイズスケールラベル

`xxl`、`xl`、`l`、`standard`、`small`、`xsmall`、`xxsmall`

### 4.2 汎用コントロールトークン（セマンティクス）

- **高さ**：`control-height-xxl|xl|l|standard|small|xsmall|xxsmall[-mobile|-desktop]`
- **パディング**：`control-padding-inline-*`、`control-padding-block-*`（i18n対応のため `inline/block` を使用、left/right/top/bottom は不可）
- **角丸**：`control-radius-xxl|xl|l|standard|small|xsmall|xxsmall`
- **ラベルサイズ**：`control-label-size-*-mobile|desktop`

**スターターマッピング（編集可）：**
- 高さ：`xxl=64`、`xl=56`、`l=48`、`standard=44`、`small=40`、`xsmall=36`、`xxsmall=32`
- ボタンパディング基準：`standard` でインライン `16`、ブロック `10`、サイズラベルに応じてスケール

### 4.3 コンポーネントスコープ（セマンティクスのみ）

- **ボタン**：`button-primary-height-*`、`button-primary-padding-inline-*`、`button-primary-padding-block-*`、`button-primary-label-size-*-mobile|desktop`
- **フィールド**：`field-background`、`field-text`、`field-placeholder`、`field-border`、`field-border-focus`、`field-border-invalid`

---

## 5. スペーシング

### 5.1 プリミティブ

`space-2|4|6|8|12|16|24|32|48|64|72|96|128|160`（px）

### 5.2 セマンティックエイリアス（任意）

- `space-stack-section → {space-64}`
- `space-stack-block → {space-24}`
- `space-inline-component → {space-16}`
- `space-grid-gap → {space-8}`

チーム間の曖昧さを減らすため、数値とセマンティック両方を公開する。

---

## 6. Figma Variables構造

- **コレクション**：
  `primitives-colors`、`primitives-typography`、`primitives-spacing`、`primitives-control`
  `semantics-colors`、`semantics-typography`、`semantics-control`、`semantics-spacing`
- **モード**：
  `light` / `dark`（カラー用）、オプションで `mobile` / `desktop`（プレビュー用）
  モード間で**同一セマンティック変数名**を使用、参照プリミティブ値のみ変更

---

## 7. エクスポートとプラットフォーム規約

- **CSS Variables**：トークン名に直接 `--` を付加（例：`--text-primary`）
- **iOS**：ハイフンケースをlowerCamelCaseに変換（例：`textSizeBody1Mobile`）
- **Android**：snake_caseに変換（例：`text_size_body_1_mobile`）
- **JS/TS定数**：SCREAMING_SNAKE_CASE（例：`TEXT_SIZE_BODY_1_MOBILE`）

**広色域CSS例**
```css
@media (color-gamut: p3){
  :root{ --blue-600: color(display-p3 0.24 0.40 0.92); }
}
```

---

## 8. ガバナンス

- **デマンドファースト成長**：セマンティックの要求が定義された**後にのみ**非グレーステップや新サイズステップを追加
- **コントラスト許可リスト**：維持されたペアリングマトリクスでAAを強制、不許可の組み合わせをブロック
- **プリミティブ直接参照禁止**：コンポーネントはセマンティクスのみを参照（lint/デザインレビューで検証）
- **非推奨化**：古いセマンティックキーは移行パスとともに非推奨としてマーク、バージョンとリリースノート必須
- **i18n**：`inline`/`block` を使用（left/right/top/bottom は不可）、非ネイティブへの明確性のため略語を避ける

---

## 9. DTCG / CSSスニペット（例示）

**DTCG（抜粋）**
```json
{
  "gray-100": {"$type":"color","$value":"oklch(0.97 0 0)"},
  "gray-100-hex": {"$type":"color","$value":"#F5F6F7"},
  "gray-900": {"$type":"color","$value":"oklch(0.20 0.02 260)"},
  "gray-900-hex": {"$type":"color","$value":"#171A21"},
  "blue-600": {"$type":"color","$value":"oklch(0.63 0.12 250)"},
  "blue-600-hex": {"$type":"color","$value":"#3D66EB"},
  "text-primary": {"$type":"color","$value":"{gray-900}"},
  "background-canvas": {"$type":"color","$value":"{gray-100}"},
  "interactive-background": {"$type":"color","$value":"{blue-600}"},

  "text-size-16": {"$type":"dimension","$value":"1rem"},
  "text-size-18": {"$type":"dimension","$value":"1.125rem"},
  "text-size-body-1-mobile": {"$type":"dimension","$value":"{text-size-16}"},
  "text-size-body-1-desktop":{"$type":"dimension","$value":"{text-size-18}"},
  "line-height-150": {"$type":"number","$value":1.50},
  "line-height-body-1-mobile":{"$type":"number","$value":"{line-height-150}"},

  "space-16":{"$type":"dimension","$value":"16px"},
  "control-height-standard":{"$type":"dimension","$value":"44px"}
}
```

**CSS（抜粋：ライト/ダーク + デバイス切り替え）**
```css
:root{
  /* OKLCH primary with HEX fallback */
  --gray-100: oklch(0.97 0 0); /* #F5F6F7 */
  --gray-900: oklch(0.20 0.02 260); /* #171A21 */
  --blue-600: oklch(0.63 0.12 250); /* #3D66EB */
  
  --text-primary: var(--gray-900);
  --background-canvas: var(--gray-100);
  --interactive-background: var(--blue-600);

  --text-size-body-1-mobile: 1rem;
  --text-size-body-1-desktop: 1.125rem;
  --text-size-body-1: var(--text-size-body-1-mobile);

  --line-height-150: 1.50;
  --line-height-body-1: var(--line-height-150);

  --space-16: 16px;
  --control-height-standard: 44px;
}
@media (prefers-color-scheme: dark){
  :root{
    --text-primary: var(--gray-100);
    --background-canvas: var(--gray-900);
  }
}
@media (min-width: 768px){
  :root{ --text-size-body-1: var(--text-size-body-1-desktop); }
}
```
