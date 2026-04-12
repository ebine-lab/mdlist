---
title: "UI/UX Guidelines"
description: "UI/UX・アクセシビリティ・アイコン・カラーモード・デザインシステム規約 / UI/UX, accessibility, icons, color modes, design system standards"
version: "1.0.0"
status: "Stable"
last_updated: "2026-01-27T21:00+09:00"
lang: "ja"
---

# UI/UX Guidelines

**UI/UX・アクセシビリティ・アイコン・カラーモード・デザインシステム規約** - ユーザー体験とビジュアルデザインの標準を定義


---

## 基本原則

- ユーザビリティを厳守
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/) AA準拠を目標
- [WAI-ARIA 1.2](https://www.w3.org/TR/wai-aria-1.2/) に従いARIA属性を適切に使用
- [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) でコントラスト比を検証
- アクセシビリティリンターの指摘を重視
- ライトモード・ダークモード・高コントラスト設定対応

---

## カラーモード対応

**必須対応モード**
- ライトモード（`prefers-color-scheme: light`）
- ダークモード（`prefers-color-scheme: dark`）
- 高コントラスト設定（`prefers-contrast: more` / `forced-colors: active`）

**OS別設定名称**

| OS | 設定名 | 設定場所 |
|---|---|---|
| Windows 10以前 | ハイコントラストモード | 設定 → 簡単操作 → ハイコントラスト |
| Windows 11 | コントラストテーマ | 設定 → アクセシビリティ → コントラストテーマ |
| macOS | コントラストを上げる | システム設定 → アクセシビリティ → ディスプレイ → コントラストを上げる |
| iOS | コントラストを上げる | 設定 → アクセシビリティ → 画面表示とテキストサイズ → コントラストを上げる |
| Android | 高コントラストテキスト | 設定 → ユーザー補助 → 高コントラストテキスト |

**CSSメディアクエリ**

| メディアクエリ | 用途 | 動作 |
|---|---|---|
| `prefers-contrast: more` | コントラスト増加の要求を検出 | デザイナー定義のスタイルで対応（macOS/iOS向け） |
| `prefers-contrast: less` | コントラスト減少の要求を検出 | デザイナー定義のスタイルで対応 |
| `forced-colors: active` | 強制カラーモードを検出 | ブラウザがカラーを強制上書き（Windows向け） |

**`prefers-contrast` と `forced-colors` の違い**
- `prefers-contrast`: ユーザーのコントラスト設定を検出し、開発者がカスタムスタイルを提供
- `forced-colors`: ブラウザがユーザー指定のカラーパレットを強制適用（開発者のスタイルを上書き）

参照：
- [MDN - prefers-contrast](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-contrast)
- [MDN - forced-colors](https://developer.mozilla.org/en-US/docs/Web/CSS/@media/forced-colors)

**CSSシステムカラーキーワード**

強制カラーモードで使用するシステムカラー（CSS Color Module Level 4）：

| キーワード | 用途 |
|---|---|
| `Canvas` | ドキュメント背景 |
| `CanvasText` | ドキュメントテキスト |
| `ButtonFace` | ボタン背景 |
| `ButtonText` | ボタンテキスト |
| `ButtonBorder` | ボタンボーダー |
| `Field` | 入力フィールド背景 |
| `FieldText` | 入力フィールドテキスト |
| `Highlight` | 選択範囲背景 |
| `HighlightText` | 選択範囲テキスト |
| `LinkText` | リンクテキスト |
| `VisitedText` | 訪問済みリンク |
| `ActiveText` | アクティブリンク |
| `GrayText` | 無効状態テキスト |
| `AccentColor` | アクセントカラー |
| `AccentColorText` | アクセントカラー上のテキスト |
| `Mark` | ハイライト背景 |
| `MarkText` | ハイライトテキスト |

参照：[MDN - system-color](https://developer.mozilla.org/en-US/docs/Web/CSS/system-color)

**強制カラーモードで上書きされるプロパティ**
- `color`
- `background-color`
- `border-color`
- `outline-color`
- `column-rule-color`
- `text-decoration-color`
- `-webkit-tap-highlight-color`
- `fill` / `stroke`（SVG）
- `box-shadow` → `none`
- `text-shadow` → `none`

**実装ベストプラクティス**

1. **セマンティックHTMLを使用**：ブラウザが要素の役割に応じてシステムカラーを自動適用

2. **`box-shadow`の代わりに`border`を使用**：`box-shadow`は強制カラーモードで`none`になる
   ```css
   /* 推奨 */
   .button {
     border: 2px solid currentColor;
   }
   
   /* または透明なoutlineを追加 */
   .button:focus {
     outline: 3px solid transparent; /* 強制カラーモードで可視化 */
     box-shadow: 0 0 0 3px blue; /* 通常モード用 */
   }
   ```

3. **システムカラーを適切なペアで使用**：コントラストが保証されるペアを使用
   - `Canvas` と `CanvasText`
   - `ButtonFace` と `ButtonText`
   - `Field` と `FieldText`
   - `Highlight` と `HighlightText`

4. **`forced-color-adjust: none`は慎重に使用**：ユーザーの選択を尊重し、例外的な場合のみ

5. **カスタムチェックボックス・ラジオボタン**：`background-color`ではなく`border`でチェック状態を示す

参照：
- [Microsoft Edge Blog - Styling for Windows high contrast](https://blogs.windows.com/msedgedev/2020/09/17/styling-for-windows-high-contrast-with-new-standards-for-forced-colors/)
- [Smashing Magazine - Windows High Contrast Mode](https://www.smashingmagazine.com/2022/03/windows-high-contrast-colors-mode-css-custom-properties/)
- [A11Y Project - Operating System and Browser Accessibility Display Modes](https://www.a11yproject.com/posts/operating-system-and-browser-accessibility-display-modes/)

---

## レスポンスタイム

| 時間 | ユーザー知覚 | 必要な対応 |
|---|---|---|
| 0.1秒未満 | 即時反応（直接操作感） | フィードバック不要 |
| 0.1〜1秒 | 遅延を感知するが流れは維持 | ボタン状態変化など軽微なフィードバック |
| 1〜10秒 | 待機中と認識、制御感低下 | ローディングインジケーター必須 |
| 10秒以上 | 注意力の限界、離脱リスク | 進捗バー＋キャンセル手段＋残り時間表示 |

参照：[Nielsen Norman Group - Response Times: The 3 Important Limits](https://www.nngroup.com/articles/response-times-3-important-limits/)

---

## フィードバック

- ユーザーアクションに対し状態変化（hover / active / focus / disabled）を明示
- 成功・エラー・警告を視覚的に区別（色＋アイコン＋テキスト、色のみに依存しない）
- 非同期処理には必ずローディング表示（determinate / indeterminate を使い分け）
- 完了時に確認メッセージを提示
- 触覚フィードバック（Haptics）でアクションを強化（モバイル）

参照：
- [Apple HIG - Feedback](https://developer.apple.com/design/human-interface-guidelines/feedback)
- [Material Design 3 - Progress Indicators](https://m3.material.io/components/progress-indicators/guidelines)
- [Material Design 3 - States](https://m3.material.io/foundations/interaction/states/overview)

---

## エラーメッセージ

**表示（WCAG 3.3.1 Error Identification）**
- エラー発生箇所の直近に配置（インライン表示優先）
- 視覚的強調（赤色テキスト＋ボーダーハイライト＋アイコン）
- **テキストでの説明必須**（色・アイコンのみは不可）
- フォームではサマリーをアクションボタン上部にも表示
- `aria-invalid`、`aria-describedby`、`role="alert"` で支援技術に通知

**内容（WCAG 3.3.3 Error Suggestion）**
- 何が問題かを具体的に説明
- どう修正すればよいかを指示
- 正しい入力例・形式を提示
- ユーザーを責めない表現を使用
- システムコード・専門用語を避ける

**タイミング**
- インライン検証を優先（入力完了時点で即座にフィードバック）
- 探索的操作（フォーカス移動のみ等）をエラーとして表示しない

参照：
- [WCAG 2.2 - 3.3.1 Error Identification](https://www.w3.org/WAI/WCAG22/Understanding/error-identification.html)
- [WCAG 2.2 - 3.3.3 Error Suggestion](https://www.w3.org/WAI/WCAG22/Understanding/error-suggestion.html)
- [Nielsen Norman Group - Error Message Guidelines](https://www.nngroup.com/articles/error-message-guidelines/)
- [Material Design - Errors](https://m1.material.io/patterns/errors.html)

---

## エラー防止（WCAG 3.3.4）

- 破壊的アクション（削除、送信、決済等）には確認ダイアログ
- 入力制約でエラーを事前に防止（文字数制限、形式マスク）
- 利用不可オプションは非活性化（グレーアウト）
- オートコンプリート・サジェストで入力支援
- 法的・金融取引では確認・修正・取消の機会を提供

参照：[WCAG 2.2 - 3.3.4 Error Prevention (Legal, Financial, Data)](https://www.w3.org/WAI/WCAG22/Understanding/error-prevention-legal-financial-data.html)

---

## タッチターゲット

| プラットフォーム | 最小サイズ | 出典 |
|---|---|---|
| iOS | 44×44 pt | Apple HIG |
| Android | 48×48 dp | Material Design |
| Web（WCAG AA） | 24×24 CSS px | WCAG 2.2 SC 2.5.8 |
| Web（WCAG AAA） | 44×44 CSS px | WCAG 2.2 SC 2.5.5 |

- インタラクティブ要素間に十分な間隔（最小24px）を確保
- 視覚サイズとタッチ領域は異なる場合がある（タッチ領域を拡張可）

参照：
- [Apple HIG - Accessibility](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [Material Design - Touch Targets](https://m3.material.io/foundations/accessible-design/accessibility-basics)
- [WCAG 2.2 - 2.5.8 Target Size (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html)

---

## 一貫性

- 同一機能には同一の用語・アイコン・操作を使用
- プラットフォームガイドライン準拠（iOS HIG / Material Design）
- デザインシステムのコンポーネントを再利用
- 業界標準のUIパターンを尊重（戻るボタンの位置、タブバーの配置等）

参照：
- [Apple HIG - Designing for iOS](https://developer.apple.com/design/human-interface-guidelines/designing-for-ios)
- [Material Design 3 - Design Foundations](https://m3.material.io/foundations)
- Nielsen Heuristic #4: Consistency and Standards

---

## 認知負荷軽減

- 記憶より認識（オプションを可視化、記憶に頼らせない）
- 情報の段階的開示（詳細は「もっと見る」で展開）
- 入力済み情報の再入力を求めない（WCAG 3.3.7 Redundant Entry）
- 明確なラベルと説明を提供（WCAG 3.3.2 Labels or Instructions）

参照：
- [WCAG 2.2 - 3.3.2 Labels or Instructions](https://www.w3.org/WAI/WCAG22/Understanding/labels-or-instructions.html)
- [WCAG 2.2 - 3.3.7 Redundant Entry](https://www.w3.org/WAI/WCAG22/Understanding/redundant-entry.html)
- Nielsen Heuristic #6: Recognition Rather Than Recall

---

## ヘルプ・支援（WCAG 3.3.5）

- コンテキストヘルプ（ツールチップ、インラインヒント）を提供
- 複雑なタスクにはステップバイステップのガイダンス
- ヘルプドキュメントはユーザーワークフローに沿って構成

参照：
- [WCAG 2.2 - 3.3.5 Help](https://www.w3.org/WAI/WCAG22/Understanding/help.html)
- Nielsen Heuristic #10: Help and Documentation

---

## その他

- アスキーアートでのワイヤー作成は不要、必要なUI要素のみ簡潔に記述
- 生成ツール等のフロントエンドは最低限のレイアウトと装飾に留める
- 一度作成した生成用フロントエンドのコードを再利用

---

## アイコン

- SVGまたはアイコンフォントを基本とする
- [Material Symbols](https://fonts.google.com/icons)を採用、CSS内の埋め込みSVGデータURIとして実装（CDNホストのリソースや絵文字より優先）
- 参照：[Material Symbols Guide](https://developers.google.com/fonts/docs/material_symbols)
- ライトモード・ダークモード・高コントラスト設定対応で可視性を確保
- 固定色アイコンの不可視化を防ぐため、動的クラスによる文脈依存スタイリングを適用
- SVGアイコンは`currentColor`を使用し、強制カラーモードでシステムカラーを継承

---

## デザインシステム規約

### デザイントークン

詳細仕様は [design-system-tokens-spec.md](design-system-tokens-spec.md) を参照。

**基本原則**
- レイヤー構造：Primitives → Semantics → Component
- コンポーネントからプリミティブを直接参照しない（セマンティクス経由のみ）
- ライト/ダークモードは同一トークンキーで値のみ切り替え
- 交換フォーマット：DTCG（Design Tokens Community Group）

### 命名規則

- Tailwind基準で作成
- 接頭辞なしで章ベースの階層構造
- 意味論的命名を徹底
- ハイフン区切り、小文字ASCII（`[a-z0-9-]`）
- 状態はサフィックス：`-hover`, `-active`, `-disabled`, `-selected`, `-focus`, `-visited`

### カラー

参照（必須）：
- [WCAG 2.2 - Contrast](https://www.w3.org/WAI/WCAG22/Understanding/contrast-minimum.html)
- [WCAG 2.2 - Non-text Contrast](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html)

- OKLCHを主体とし、HEX / RGBAへのフォールバック必須
- ライトモード・ダークモード・高コントラスト設定対応
- コントラスト比を包括的に文書化
  - テキスト：WCAG AA（4.5:1）/ AAA（7:1）準拠
  - 大きなテキスト：WCAG AA（3:1）/ AAA（4.5:1）準拠
  - UIコンポーネント・グラフィック：3:1以上
- カラースケール表示には中立的なグレー背景を使用
- ペアリングマトリクス（許可リスト）でコントラスト比を管理

**高コントラスト設定対応**
- `prefers-contrast: more` 用のカラーバリエーションを定義
- `forced-colors: active` 時はCSSシステムカラーキーワードを使用
- セマンティックカラートークンを通じてモード切替を実装

**カラーコンテキスト階層**

カラートークンの適用文脈を階層で定義し、背景色と前景色の組み合わせを明確にする。

| 階層 | 説明 | 背景トークン例 | 前景トークン例 |
|---|---|---|---|
| ベース（Base） | 最下層の背景、アプリの基盤 | `background-canvas` | `text-primary` |
| キャンバス上（On-Canvas） | Canvas上の要素 | - | `on-canvas-text` |
| 表面（Surface） | カード、シート等の浮いた要素 | `background-surface` | `on-surface-text` |
| ブランド上（On-Brand） | ブランドカラー背景上の要素 | `interactive-background` | `on-brand-text` |

参照：[Material Design 3 - Color System](https://m3.material.io/styles/color/system/overview)

### タイポグラフィ

参照：[WCAG 2.2 - Text](https://www.w3.org/WAI/WCAG22/quickref/#text-alternatives)

**フォントウェイト**（100は定義に含めるが使用しない）
- 100 (Thin) / 200 (Extra Light) / 300 (Light) / 400 (Regular) / 500 (Medium) / 600 (Semi Bold) / 700 (Bold) / 800 (Extra Bold) / 900 (Black)

**テキストサイズ**（rem単位、基準値16px=1rem、偶数ピクセル相当値のみ）

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

**行高さ**（line-height、単位なし数値）

参照：[MDN - line-height](https://developer.mozilla.org/en-US/docs/Web/CSS/line-height)

**単位なし数値を採用する理由**：`%`や`em`で指定すると計算済みの固定値が子要素に継承され、意図しない表示崩れが発生する。単位なし数値は各要素のfont-sizeに対する乗数として再計算されるため、継承問題を回避できる。

| 用途 | 値 |
|---|---|
| 本文 | 1.40 / 1.50 / 1.60 / 1.70 / 1.80 |
| 見出し | 1.10 / 1.20 / 1.30 / 1.40 |
| ディスプレイ | 1.10 / 1.20 / 1.30 |

注：見出し・ディスプレイのデフォルト1.50は大きすぎるため、1.10〜1.40を推奨。複数行表示時にアセンダー/ディセンダーの重なりがないか要確認。

**レタースペース**（letter-spacing、em単位）

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

**ワードスペース**（word-spacing、仕様サイズはem単位）

| 用途 | 日本語 | 欧文 |
|---|---|---|
| 全般 | 適用外（単語間スペース不使用） | 0.16em以上（WCAG準拠） |

### スペーシング

プリミティブ値（px）：2 / 4 / 6 / 8 / 12 / 16 / 24 / 32 / 48 / 64 / 72 / 96 / 128 / 160

### コントロール

#### タッチターゲットサイズ基準

参照：
- [WCAG 2.5.5 Target Size (Enhanced)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-enhanced.html)
- [WCAG 2.5.8 Target Size (Minimum)](https://www.w3.org/WAI/WCAG22/Understanding/target-size-minimum.html)

**アクセシビリティ基準**

| 基準 | サイズ | レベル | 備考 |
|---|---|---|---|
| WCAG 2.5.8 | 24×24 CSS px | AA（必須） | 最小要件、または24pxの間隔確保 |
| WCAG 2.5.5 | 44×44 CSS px | AAA（推奨） | 拡張要件 |

**プラットフォーム別ガイドライン**

| プラットフォーム | 最小サイズ | 単位 | 出典 |
|---|---|---|---|
| Apple iOS | 44×44 | pt | Human Interface Guidelines |
| Apple macOS | 可変（controlSize） | pt | Human Interface Guidelines |
| Apple visionOS | 60×60 | pt | Human Interface Guidelines |
| Android / Material Design | 48×48 | dp | Material Design Guidelines |
| Microsoft Fluent | 44×44 | px | Fluent Design System |
| Web（推奨） | 44×44 | CSS px | WCAG AAA準拠 |

#### サイズスケール

**スケールラベル**
- xxl / xl / l / standard / small / xsmall / xxsmall

**高さ基準値（コントロール）**

| スケール | 高さ | タッチターゲット | 用途 |
|---|---|---|---|
| xxl | 64px | 64px | 特大ボタン、ヒーローCTA |
| xl | 56px | 56px | 大ボタン、プライマリCTA |
| l | 48px | 48px | 大きめのコントロール |
| standard | 44px | 44px | 標準コントロール（WCAG AAA準拠） |
| small | 40px | 44px | 小コントロール（視覚40px、タッチ44px） |
| xsmall | 36px | 44px | 極小コントロール（視覚36px、タッチ44px） |
| xxsmall | 32px | 44px | 最小コントロール（視覚32px、タッチ44px） |

注：視覚的サイズがタッチターゲットより小さい場合、paddingでタッチ領域を拡張

#### コンポーネント別サイズ規定

**ボタン**

| フレームワーク | Small | Medium | Large | 出典 |
|---|---|---|---|---|
| MUI | 30.75px | 36.5px | 42.25px | MUI Button API |
| Material Design | 36dp | 36dp | 48dp | Material Design 1/2 |
| Material Design 3 | 40dp | 40dp | 56dp | Material Design 3 |

**テキストフィールド**

| フレームワーク | Small | Standard | 出典 |
|---|---|---|---|
| MUI | 40px | 56px | MUI TextField |
| Material Design 3 | 40dp | 56dp | Material Design 3 |

**macOS コントロールサイズ**

参照：[Apple NSControl.ControlSize](https://developer.apple.com/documentation/appkit/nscontrol/controlsize-swift.enum)

| サイズ | 用途 |
|---|---|
| mini | 高密度UI、システム設定 |
| small | コンパクトなツールバー |
| regular | 標準（デフォルト） |
| large | 強調されたコントロール |

#### 実装ガイドライン

**タッチターゲット拡張**

視覚的に小さいコントロールでもタッチターゲットを確保：

```css
/* 視覚的に32pxかつタッチターゲット44px */
.button-xxsmall {
  height: 32px;
  min-height: 32px;
  padding: 6px 12px; /* タッチ領域拡張 */
  position: relative;
}

.button-xxsmall::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  min-width: 44px;
  min-height: 44px;
}
```

**レスポンシブ対応**

```css
/* タッチデバイス向け */
@media (pointer: coarse) {
  .control {
    min-height: 48px; /* Android推奨 */
  }
}

/* 精密ポインター向け */
@media (pointer: fine) {
  .control {
    min-height: 32px; /* デスクトップ向け */
  }
}
```

**間隔規定**
- 隣接コントロール間：最小8px（推奨12px）
- タッチターゲットが重複しないよう配置

### アイコンサイズ

**UIアイコン**（デフォルト24px）
- 8px / 12px（定義のみ、使用しない） / 16px / 20px / 24px / 32px / 40px / 48px / 56px / 64px

**ディスプレイアイコン**（デフォルト256px）
- 72px / 96px / 128px / 192px / 256px / 384px / 512px / 768px / 1024px

### 参照デザインシステム

- [MUI](https://mui.com/)
- [Material Design 3](https://m3.material.io/)
- [Apple Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/)
- [Carbon Design System](https://carbondesignsystem.com/)
- [Tailwind CSS](https://tailwindcss.com/docs)
- [Atlassian Design System](https://atlassian.design/)
- [Adobe Spectrum](https://spectrum.adobe.com/)
- [Open Design Systems](https://www.designsystems.com/open-design-systems/) - デザインシステム一覧

### カラー参考リソース

- [OKLCH Color Picker](https://oklch.com/)
- [OKLCH in CSS](https://evilmartians.com/chronicles/oklch-in-css-why-quit-rgb-hsl)
- [CSS Color Level 4](https://www.w3.org/TR/css-color-4/)
