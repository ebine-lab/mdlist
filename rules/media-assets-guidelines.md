---
title: "Media Assets Guidelines"
description: "メディアファイルの技術標準と実装パターン - フォーマット選定・命名規則・圧縮最適化 / Media assets standards - format selection, naming conventions, compression optimization"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-11T00:00+09:00"
lang: "ja"
---

# Media Assets Guidelines

**メディアアセット（画像・動画・音声・フォント）のフォーマット選定・命名規則・ディレクトリ構成・最適化基準** - Web配信・アプリ開発・ローカル制作物を対象としたメディアファイルの技術標準を定義


## 目的

プロジェクトで使用するメディアアセットについて、フォーマット選定基準・ファイル命名規則・ディレクトリ構成・圧縮と最適化の基準を統一し、Web配信・アプリ開発・ローカル制作物の各用途で品質とパフォーマンスを両立する。

## 対象

画像（ラスター・ベクター）、動画、音声、フォント（Web配信用・ネイティブアプリ用）、ファビコン、アプリアイコン、ドキュメント形式（PDF・Markdown）。

## 非対象

- CSSデザイントークン（[design-system-tokens-spec.md](design-system-tokens-spec.md) 管轄）
- CSSリセットにおけるメディア要素の表示制御（[coding-standards.md](coding-standards.md) 管轄）
- SEO向けOGP画像・メタ画像指定（[seo-requirements.md](seo-requirements.md) 管轄）
- Core Web Vitalsの数値基準（将来のperformance-optimization.md管轄）
- アプリストア審査用スクリーンショット・プラットフォーム固有UIガイドライン（各プラットフォームの公式ガイドラインに従う）

## 用語

| 用語 | 定義 |
|---|---|
| ロスレス圧縮 | 元データを完全に復元できる圧縮方式。PNG・FLAC・WOFF2等が該当。 |
| ロッシー圧縮 | 人間の知覚に影響が少ないデータを削除して圧縮する方式。AVIF・WebP・Opus等が該当。 |
| フォールバック | 第一候補のフォーマットに非対応の環境向けに提供する代替フォーマット。 |
| サブセット化 | フォントファイルから使用する文字（グリフ）のみを抽出し、ファイルサイズを削減する処理。 |

---

## 画像フォーマット

### 採用フォーマット

| フォーマット | 種別 | 主な用途 | 透過 | アニメーション |
|---|---|---|---|---|
| AVIF（.avif） | ロッシー/ロスレス | 写真・グラデーション画像の第一候補。HDR・広色域対応。 | 対応 | 対応 |
| WebP（.webp） | ロッシー/ロスレス | AVIFのフォールバック。アニメーション用途にも適する。 | 対応 | 対応 |
| PNG（.png） | ロスレス | スクリーンショット・図表・透過が必要なUI要素。APNGによるアニメーションを含む。 | 対応 | 対応（APNG） |
| SVG（.svg） | ベクター | アイコン・ロゴ・図形・インフォグラフィック。アニメーション用スプライトを含む。 | 対応 | 対応（SMIL/CSS） |

### 制作・保存用

| フォーマット | 種別 | 主な用途 | 備考 |
|---|---|---|---|
| TIFF（.tif/.tiff） | ロスレス（非圧縮/LZW等） | 印刷・DTP入稿、写真編集マスター保存、スキャン原本保存。 | 16bit/チャネル・CMYK・レイヤー保持等に対応し、制作段階での品質保持に最適。ファイルサイズが大きくブラウザの標準サポートがないため、Web配信前にAVIF・WebP・PNGへの変換が必須。 |

### 非採用フォーマット

| フォーマット | 分類 | 理由 |
|---|---|---|
| GIF（.gif） | 禁止 | アニメーションGIFを含め使用禁止。256色制限、圧縮効率が極めて低い。WebPまたはAPNGで代替する。 |
| JPEG（.jpg/.jpeg） | 非推奨 | 受け入れは可能だが新規作成では非推奨。AVIFまたはWebPで代替する。外部提供素材等でやむを得ない場合のみ許容。 |
| JPEG 2000（.jp2） | 禁止 | Safari以外のブラウザが未対応。AVIF・WebPが上位互換。 |
| HEIC（.heic） | 条件付き採用 | ブラウザ互換性スコア13%。Safari 17.6以降のみ対応し、Chrome・Firefox・Edgeは未対応。ライセンス料が複雑で高額。**iOS/iPadOS/macOS向けのネイティブアプリでは基本的に採用可能。** Web配信ではAVIFまたはWebPへの変換が必須。iPhone/iPad撮影素材等の受け入れは可能。 |

### 将来候補（経過観察）

| フォーマット | 状況（2026年2月時点） |
|---|---|
| JPEG XL（.jxl） | Safari 17以降で対応済み。Chromiumは2026年1月にRustベースデコーダ（jxl-rs）をマージしたが、`chrome://flags`で手動有効化が必要でデフォルト無効。Firefoxは Nightly のみ。ロスレスJPEG再圧縮・プログレッシブ復号・HDR対応等の優位性があり、Chromiumデフォルト有効化後に採用を再検討する。 |

### 画像配信パターン

`<picture>`要素によるフォーマットフォールバックを標準とする。ブラウザは先頭から順に対応フォーマットを選択する。

```html
<picture>
  <source srcset="image.avif" type="image/avif">
  <source srcset="image.webp" type="image/webp">
  <img src="image.png" alt="YOUR_IMAGE_ALT_TEXT" loading="lazy" decoding="async">
</picture>
```

写真系画像のフォールバック順序はAVIF→WebP→JPEG（またはPNG）とする。ファーストビュー内のLCP候補画像には`loading="lazy"`を付与せず、`fetchpriority="high"`を指定する。

レスポンシブ画像の`srcset`と`sizes`の詳細は [css-breakpoints-guidelines.md](css-breakpoints-guidelines.md) のブレークポイント定義と連携して設計する。

### アスペクト比の調整

ターゲットの表示枠と画像のアスペクト比が異なる場合、以下のいずれかの方法で対応する。

| 方法 | 概要 | 適用場面 |
|---|---|---|
| CSS `object-fit` | `contain`（余白あり全体表示）または`cover`（トリミングで全面表示）で表示を制御する。画像データ自体は変更しない。 | サムネイル一覧・カード型UIなど、統一比率の表示枠に多様な比率の画像を収める場合。 |
| 背景埋め（パディング生成） | 不足領域を白・黒・単色、またはAI生成（Generative Fill等）で補完し、ターゲット比率の画像として書き出す。 | OGP画像・バナー・サムネイルなど、配信先が固定比率を要求し`object-fit`による制御ができない場合。 |
| トリミング（クロップ） | 被写体の重要領域を維持しつつ不要部分を切り取る。 | 被写体が明確で周辺に余白がある場合。 |

- CSS `object-fit`による制御を第一選択とし、画像データの加工は配信先の制約で必要な場合に限定する。
- 背景埋めを行う場合、AI生成による補完は被写体の改変・捏造にあたらない範囲（背景・余白の延長）に限定する。生成結果は目視で確認し、不自然なアーティファクトがないことを検証する。
- トリミングは`object-position`（CSS）またはアートディレクション（`<picture>`＋`media`属性）で動的に制御できる場合、画像データの事前クロップより優先する。

### SVG運用規則

- インラインSVGはCSS・JavaScriptによるスタイル制御が必要な場合に限定する。
- 装飾目的のSVGは外部ファイルとして`<img>`で読み込む。
- アイコンスプライトは`<symbol>`と`<use>`パターンを使用する。
- SVGファイルにはSVGO等のツールで最適化を施す。不要なメタデータ・エディタ情報・コメントを除去する。
- `viewBox`属性を必ず指定し、`width`/`height`属性による固定サイズ指定は避ける。

### インライン埋め込み（Data URI）

画像をBase64またはURLエンコードしてCSS・HTMLに直接埋め込むことで、HTTPリクエストを削減できる。

#### 埋め込み方式

| 方式 | 構文例 | 特性 |
|---|---|---|
| Base64（ラスター画像向け） | `url("data:image/png;base64,iVBOR...")` | あらゆる画像形式に対応。元データより約33%サイズが増加する。 |
| URLエンコード（SVG向け） | `url("data:image/svg+xml;charset=UTF-8,%3Csvg...")` | SVGのテキスト構造を活かしBase64より小さい。可読性も維持される。`#`→`%23`、`"`→`%22`等のエスケープが必要。`xmlns='http://www.w3.org/2000/svg'`を必ず含めること。 |

#### 使用基準

- **元ファイルサイズが1KB以下の画像に限定する。** Base64エンコードにより約33%増加するため、大きな画像ではCSS/HTMLのファイルサイズが肥大化する。サーバー側のGzip/Brotli圧縮が有効な場合、増加分は約8〜9%に軽減される。
- 1ピクセルパターン・小型装飾アイコン・CSSのみで完結させたい背景パターン等に適する。
- Data URIは埋め込み先のCSS/HTMLと一体でキャッシュされ、画像単体での個別キャッシュができない。更新頻度の高い画像や複数ページで共有する画像には外部ファイルを使用する。
- HTTP/2以降の環境では多重化により小ファイルのリクエストオーバーヘッドが大幅に軽減されるため、Data URIの利点は限定的になる。外部ファイル＋適切なキャッシュ設定を優先し、Data URIはリクエスト削減が明確に有効な場合にのみ採用する。

#### コード例

```css
/* Base64: 1pxの透過PNG */
.spacer {
  background-image: url("data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII=");
}

/* URLエンコード: 小型SVGアイコン */
.icon-chevron {
  background-image: url("data:image/svg+xml;charset=UTF-8,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24'%3E%3Cpath d='M9 18l6-6-6-6' fill='none' stroke='%23333' stroke-width='2'/%3E%3C/svg%3E");
}
```

---

## 動画フォーマット

### Web配信用

| フォーマット | コーデック | 主な用途 |
|---|---|---|
| MP4（.mp4） | H.264（AVC） | 汎用動画。最も広いブラウザ互換性。 |
| MP4（.mp4） | H.265（HEVC） | 高圧縮動画。Safari・iOS/iPadOS中心の環境向け。ブラウザ互換性が限定的なため`<source>`でH.264フォールバックを提供すること。 |
| WebM（.webm） | VP9 / AV1 | ロイヤリティフリーの高効率コーデック。Chrome・Firefox・Edge対応。 |

### 制作・編集用（中間コーデック）

映像制作ワークフローにおいて、撮影素材の編集・カラーグレーディング・合成作業に使用する中間（mezzanine/intermediate）コーデック。いずれもイントラフレーム圧縮により編集時のシーク・スクラブ性能に優れ、再エンコードを繰り返しても画質劣化が少ない。最終的なWeb配信時にはH.264・H.265・AV1等へ変換する。

| コーデック | コンテナ | 主なバリアント | 用途・特性 |
|---|---|---|---|
| Apple ProRes | MOV / MXF | ProRes 422 Proxy：オフライン編集。低ビットレート（約45Mbps @1080/29.97p）。 | Apple公式の中間コーデック。Final Cut Proとの親和性が最も高い。全バリアント10bit対応、4444系は12bit対応。iPhone 13 Pro以降でカメラ直接収録にも対応。エンコードはmacOS環境が前提（Windows環境ではFFmpeg等で限定的に対応）。 |
| | | ProRes 422 LT：軽量編集。ProRes 422の約70%のデータレート。 | |
| | | ProRes 422：標準編集。バランスの取れた品質とファイルサイズ（約145Mbps @1080/60i）。 | |
| | | ProRes 422 HQ：高品質編集・仕上げ。アルファチャネル不要時の最高品質（約220Mbps @1080/60i）。 | |
| | | ProRes 4444：合成・VFX。4:4:4カラー＋ロスレスアルファチャネル対応（約330Mbps @1080/29.97p）。 | |
| | | ProRes 4444 XQ：HDRグレーディング。ProResファミリー最高品質（約500Mbps @1080/29.97p）。 | |
| | | ProRes RAW：RAWセンサーデータを効率的に保持。デベイヤーをポスプロに委ねる。 | |
| Avid DNxHD / DNxHR | MXF / MOV | DNxHD：1080p以下の解像度に対応。8bit/10bit。 | Avid Media Composer標準の中間コーデック。Windows・macOSの両環境でエンコード・デコードが可能。クロスプラットフォームのワークフローに適する。 |
| | | DNxHR LB / SQ / HQ / HQX / 444：2K・4K・8K解像度に対応。HQXは12bit、444はアルファチャネル対応。 | |
| GoPro CineForm | MOV / AVI | Low〜Filmscan 1：品質レベルを段階選択。10bit 4:2:2、RGB 12bitにも対応。 | オープンソース（SDK公開済み）のウェーブレット圧縮コーデック。4K以上の高解像度でも効率的に動作し、360°映像にも対応。Windows環境での安定性に定評がある。 |

参照：[Apple ProRes（Apple公式）](https://support.apple.com/en-us/102207)、[Apple ProRes White Paper](https://www.apple.com/final-cut-pro/docs/Apple_ProRes.pdf)、[Frame.io - Intermediate Codec Comparison](https://blog.frame.io/2017/02/13/compare-50-intermediate-codecs/)

#### 中間コーデック選定の目安

- **macOS中心のワークフロー（Final Cut Pro）**：Apple ProResを第一候補とする。
- **クロスプラットフォーム（Premiere Pro / DaVinci Resolve）**：DNxHD/DNxHRまたはCineFormを選択する。ProResはデコードは可能だがWindows環境でのエンコードに制約がある。
- **VFX・合成作業でアルファチャネルが必要**：ProRes 4444、DNxHR 444、またはCineForm RGBを選択する。
- **オフライン編集（プロキシ）**：ProRes 422 Proxy、DNxHR LB等の低ビットレートバリアントを使用し、仕上げ時にオリジナルにリリンクする。

### 非採用フォーマット

FLV、WMV、AVI（CineFormコンテナとしての用途を除く）、MOV（Web配信用途）、3GP等の旧形式はWeb配信に使用しない。

### アスペクト比と黒帯処理

16:9を基準フレームとし、異なるアスペクト比の映像を収める際は以下の処理を適用する。

| 処理名 | 発生条件 | 表示結果 | 代表的なソース |
|---|---|---|---|
| レターボックス（letterbox） | フレームより横長の映像を収める場合 | 上下に黒帯が生じる | シネマスコープ（2.39:1）、21:9（2.33:1）等の映画素材 |
| ピラーボックス（pillarbox） | フレームより縦長の映像を収める場合 | 左右に黒帯が生じる | SD放送素材（4:3）、スマートフォン縦撮り（9:16）等 |

- 黒帯は映像データとして焼き込む（ハードマット）のではなく、CSSまたはプレイヤー設定で表示制御する。焼き込みはトリミングやリフレームの余地を失い、異なるアスペクト比のデバイスで二重黒帯（ウィンドウボックス）が発生する原因になる。
- `<video>`要素のデフォルト（`object-fit: contain`相当）はレターボックス/ピラーボックスを自動適用する。映像をフレーム全体に拡大して黒帯を排除する場合は`object-fit: cover`を使用するが、上下または左右が切り取られる点に留意する。
- 映画素材等で意図的にレターボックスを焼き込んだマスター（ハードマット済み）を受け入れる場合は、そのまま使用し追加の黒帯処理を行わない。

### `<video>`要素の属性

| 属性 | 値 | 用途 | 備考 |
|---|---|---|---|
| `controls` | （ブール属性） | ブラウザ標準の再生コントロールを表示する。 | アクセシビリティのために原則必須。カスタムコントロールを実装する場合のみ省略し、JavaScriptで`controls`を無効化する（スクリプト無効環境で標準コントロールが残るようにする）。 |
| `preload` | `none` / `metadata` / `auto` | 動画データの事前読み込みの範囲を指定する。 | `metadata`を標準とする。`autoplay`指定時は`preload`より`autoplay`が優先される。`none`はユーザーが再生しない可能性が高い場合に使用する。 |
| `playsinline` | （ブール属性） | モバイルSafari等でインライン再生を有効化する。 | 指定しないとiOS Safariが全画面再生に切り替わる場合がある。 |
| `poster` | 画像URL | 再生開始前に表示するサムネイル画像を指定する。 | 未指定の場合、最初のフレームが利用可能になるまで何も表示されない。空文字列（`poster=""`）は無効。指定する場合は必ず有効なURLを設定する。 |
| `width` / `height` | ピクセル値（整数） | 動画の表示領域サイズを指定する。 | CLS（Cumulative Layout Shift）防止のため指定を推奨。ブラウザがアスペクト比を算出してレイアウトシフトを回避する。パーセント値・`auto`は不可。レスポンシブ対応はCSSで行う。 |
| `crossorigin` | `anonymous` / `use-credentials` | CDN・サブドメインからの動画取得にCORSを使用する。 | `<canvas>`で動画フレームを描画する場合に必須（未設定だとcanvasがtaintedになる）。CDN配信時は`anonymous`を指定する。`<track>`でクロスオリジンのVTTファイルを読み込む場合にも必要。 |
| `controlslist` | `nodownload` / `nofullscreen` / `noremoteplayback`（スペース区切り） | ブラウザ標準コントロールの一部を非表示にする。 | Chromium系ブラウザの実験的属性。HTML標準には未採用のため、HTMLバリデータでは警告が出る。Safari・Firefoxでは無視される。 |
| `disablepictureinpicture` | （ブール属性） | Picture-in-Pictureモードを無効化する。 | Chromium系ブラウザで有効。PiPボタンの非表示とPiP APIの無効化を行う。 |
| `autoplay` | （ブール属性） | 動画を自動再生する。 | ブラウザのAutoplay Policyにより、`muted`を併用しないとブロックされる。背景動画・ループ動画に限定して使用する。 |
| `muted` | （ブール属性） | 動画の音声を無効化する。 | `autoplay`と併用する。ユーザーが明示的に操作するコンテンツ動画には使用しない。 |
| `loop` | （ブール属性） | 動画を繰り返し再生する。 | 背景動画・ループアニメーションに使用する。 |
| `src` | 動画URL | 埋め込む動画のURLを直接指定する。 | `<source>`子要素による指定が推奨。単一フォーマットのみで十分な場合に使用する。`<source>`と併用した場合、`src`は無視される。 |

### `<source>`要素の属性

`<source>`要素は`<video>`の子要素として配置し、複数のフォーマット・品質を提供する。ブラウザは上から順に評価し、最初に再生可能なソースを選択する。

| 属性 | 説明 |
|---|---|
| `src` | メディアファイルのURL。必須。 |
| `type` | MIMEタイプ。コーデックパラメータを含めることで、ブラウザがファイルをダウンロードせずに再生可否を判定できる（例：`video/webm; codecs="vp09.00.10.08"`、`video/mp4; codecs="avc1.42E01E, mp4a.40.2"`）。省略時はブラウザがサーバーに問い合わせる。 |
| `media` | メディアクエリ。ビューポートサイズに応じたソース選択を可能にする（例：`(max-width: 599px)`）。Safari・Firefoxで対応。Chromium系は`media`属性を無視し、最初の再生可能なソースを選択する。 |

#### `<source>`の`media`属性によるレスポンシブ動画

```html
<video controls preload="metadata" playsinline width="1280" height="720">
  <source src="video-small.mp4" type="video/mp4" media="(max-width: 599px)">
  <source src="video-large.mp4" type="video/mp4">
</video>
```

- `media`属性は2014年にHTML仕様から一度削除されたが、2024年以降Safari・Firefoxで再度サポートされている。Chromium系ブラウザは未対応のため、`media`属性を無視して最初の再生可能なソースを選択する。
- Chromium系を含む全ブラウザで動画サイズを最適化する場合は、HLS/DASH等のアダプティブストリーミングまたはJavaScriptによるソース切り替えを使用する。
- `media`属性による切り替えはページ読み込み時のみ評価され、ビューポートのリサイズでは再評価されない。

### メディアフラグメントURI

`<video>`（および`<audio>`）のソースURLにフラグメント`#t=`を付加することで、再生範囲を指定できる。W3C Media Fragments URI 1.0仕様に基づく。

| 書式 | 動作 |
|---|---|
| `#t=10` | 10秒地点から末尾まで再生する。 |
| `#t=10,20` | 10秒地点から20秒地点まで再生する。 |
| `#t=,20` | 先頭から20秒地点まで再生する。 |
| `#t=01:30,02:00` | 1分30秒から2分00秒まで再生する。 |

```html
<!-- 15秒〜20秒の区間のみ再生する -->
<video controls preload="metadata" playsinline>
  <source src="video.mp4#t=15,20" type="video/mp4">
</video>
```

- 時刻の書式はNPT（Normal Play Time）を使用する。秒数（`10`、`10.5`）またはHH:MM:SS形式（`01:30:00`）。分・秒は2桁で記述する（`1:20`は無効、`01:20`が正しい）。
- Firefox・WebKit/Safari・Chromium系ブラウザでサポートされている。
- フラグメント指定はブラウザ側でのシーク処理であり、サーバーから部分的なデータのみをダウンロードするものではない。

### 動画配信パターン

#### 標準パターン

```html
<video
  controls
  preload="metadata"
  playsinline
  poster="video-thumbnail.avif"
  width="1280"
  height="720">
  <source src="video.webm" type="video/webm; codecs=av01.0.08M.08">
  <source src="video.mp4" type="video/mp4">
  <track kind="captions" src="captions-ja.vtt" srclang="ja" label="日本語" default>
  <track kind="captions" src="captions-en.vtt" srclang="en" label="English">
  <p>お使いのブラウザは動画の再生に対応していません。<a href="video.mp4" download>動画をダウンロード</a>してご覧ください。</p>
</video>
```

- `poster`にはAVIF・WebP・PNG等の静止画を指定する。本ガイドラインの画像フォーマット規則に従う。
- `width`/`height`はソース動画のアスペクト比と一致させる。レスポンシブ対応は`max-width: 100%; height: auto;`等のCSSで行う。
- フォールバックテキスト（`<video>`の子要素としてのテキスト・リンク）は動画要素非対応のブラウザでのみ表示される。ダウンロードリンクを含めることを推奨する。

#### レスポンシブ動画CSS

```css
video {
  max-width: 100%;
  height: auto;
}
```

- `width`/`height`属性でアスペクト比を宣言し、CSSで`max-width: 100%; height: auto;`を適用することで、コンテナ幅に追従しつつCLS（レイアウトシフト）を防止する。
- `object-fit`は`<video>`要素にも適用可能。`contain`（デフォルト相当）でレターボックス/ピラーボックスを自動適用、`cover`でフレーム全体に拡大（上下または左右がクロップされる）。

#### CDN配信パターン

```html
<video
  controls
  preload="metadata"
  playsinline
  crossorigin="anonymous"
  poster="https://cdn.example.com/videos/video-thumbnail.avif"
  width="1280"
  height="720">
  <source src="https://cdn.example.com/videos/video.webm" type="video/webm; codecs=av01.0.08M.08">
  <source src="https://cdn.example.com/videos/video.mp4" type="video/mp4">
  <track kind="captions" src="https://cdn.example.com/videos/captions-ja.vtt" srclang="ja" label="日本語" default>
</video>
```

CDNまたはサブドメインから動画・VTTファイルを配信する場合、`crossorigin="anonymous"`を指定する。配信サーバー側で`Access-Control-Allow-Origin`ヘッダーの設定が必要。

#### 背景動画パターン

```html
<video
  autoplay
  muted
  loop
  playsinline
  preload="auto"
  poster="hero-bg-fallback.avif"
  width="1920"
  height="1080"
  aria-hidden="true">
  <source src="hero-bg.webm" type="video/webm; codecs=av01.0.08M.08">
  <source src="hero-bg.mp4" type="video/mp4">
</video>
```

- 装飾目的の背景動画には`controls`を付与せず、`autoplay muted loop playsinline`を指定する。
- `aria-hidden="true"`を指定し、スクリーンリーダーが装飾動画を読み上げないようにする。
- `prefers-reduced-motion: reduce`が有効な環境では動画を停止し、`poster`画像または静止画で代替する。
- 背景動画は音声トラックを除去してファイルサイズを削減する。

#### `prefers-reduced-motion`による動画停止

CSS方式：

```css
@media (prefers-reduced-motion: reduce) {
  video[autoplay] {
    display: none;
  }
  /* poster画像をbackground-imageで表示するフォールバック要素を用意する */
  .hero-bg-fallback {
    display: block;
  }
}
```

JavaScript方式：

```javascript
const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
const video = document.querySelector('video[autoplay]');

function handleReducedMotion(e) {
  if (e.matches) {
    video.pause();
  } else {
    video.play();
  }
}

mediaQuery.addEventListener('change', handleReducedMotion);
handleReducedMotion(mediaQuery);
```

- CSS方式は動画要素自体を非表示にし、別のフォールバック要素で`poster`相当の静止画を表示する。
- JavaScript方式は動画要素を保持したまま`pause()`/`play()`で制御する。ユーザーが設定を変更した場合にも即座に反応する。

### `<track>`要素（字幕・キャプション）

`<track>`要素は`<video>`（および`<audio>`）の子要素として配置し、WebVTT形式（`.vtt`）のテキストトラックを関連付ける。`<source>`要素の後に配置する。

#### `kind`属性の種別

| 値 | 用途 | 対象ユーザー |
|---|---|---|
| `captions` | クローズドキャプション。話者の識別・効果音・音楽の説明を含む音声の完全な文字起こし。 | 聴覚障害者、音声を再生できない環境のユーザー。 |
| `subtitles` | 字幕。主に会話の翻訳。効果音の記述は通常含まない。 | 動画の言語を理解しないユーザー。 |
| `descriptions` | 音声解説。視覚的な情報の説明を音声で提供する。 | 視覚障害者。 |
| `chapters` | チャプターマーカー。動画内のナビゲーションポイント。 | 全ユーザー。 |
| `metadata` | スクリプトで使用するメタデータ。ユーザーには表示されない。 | アプリケーション（検索インデックス・インタラクティブ機能等）。 |

#### `<track>`属性

| 属性 | 説明 |
|---|---|
| `src` | VTTファイルのURL。クロスオリジンの場合は`<video>`に`crossorigin`属性が必要。 |
| `kind` | テキストトラックの種別。上記テーブル参照。 |
| `srclang` | トラックの言語。BCP 47言語タグ（`ja`、`en`等）。`kind="subtitles"`の場合は必須。 |
| `label` | ユーザーに表示されるトラック名（例：`日本語`、`English`）。 |
| `default` | デフォルトで有効化するトラックに指定する。同一`kind`で1つのみ。 |

#### WCAG準拠の注記

- **WCAG 1.2.2（キャプション - 収録済み）**：収録済みの音声コンテンツにはキャプション（`kind="captions"`）を提供する。
- **WCAG 1.2.5（音声解説 - 収録済み）**：収録済みの映像コンテンツには音声解説（`kind="descriptions"`）を提供する（AA）。
- 同一の`<video>`要素に同じ`kind`・`srclang`・`label`の組み合わせを持つ`<track>`を複数配置できない。

#### WebVTTファイルの書式

```
WEBVTT

1
00:00:01.000 --> 00:00:04.000
こんにちは、本日のプレゼンテーションを始めます。

2
00:00:05.000 --> 00:00:08.000
[拍手]

3
00:00:09.000 --> 00:00:12.500
まず、プロジェクトの概要をご説明します。
```

- ファイル先頭に`WEBVTT`ヘッダーを記述する。BOM（U+FEFF）は任意。
- キューIDは省略可能だが、編集・デバッグの容易さのため付与を推奨する。
- キャプション（`kind="captions"`）では話者の識別（例：`<v 田中>`）や効果音（例：`[拍手]`）を含める。

#### `::cue`擬似要素によるキャプションスタイリング

CSS `::cue`擬似要素でWebVTTキューの表示スタイルを制御できる。

```css
/* 全キューの基本スタイル */
video::cue {
  color: #fff;
  background-color: rgba(0, 0, 0, 0.75);
  font-size: 1rem;
  font-family: sans-serif;
}

/* VTTファイル内の<b>タグに対応 */
video::cue(b) {
  color: #ffcc00;
}
```

`::cue`で使用可能なCSSプロパティ：

| カテゴリ | プロパティ |
|---|---|
| 色 | `color`、`opacity` |
| 背景 | `background`およびその個別プロパティ（`background-color`、`background-image`等）。各キューに個別適用される。 |
| フォント | `font`およびその個別プロパティ（`font-size`、`font-family`、`font-weight`、`font-style`等） |
| テキスト装飾 | `text-decoration`およびその個別プロパティ、`text-shadow` |
| 輪郭 | `outline`およびその個別プロパティ |
| その他 | `white-space`、`line-height` |

- `margin`、`padding`、`border`、`display`、`position`等のレイアウトプロパティは使用できない。
- VTTファイル内の`<b>`、`<i>`、`<u>`、`<c.classname>`等のタグに対して`::cue(b)`、`::cue(i)`、`::cue(u)`、`::cue(.classname)`でセレクタを指定できる。
- `::cue-region`擬似要素は仕様に存在するが、現時点でブラウザサポートはない。

---

## 音声フォーマット

### 制作・保存用（ロスレス）

| フォーマット | 用途 | 備考 |
|---|---|---|
| WAV（.wav） | マスター音源の保存・編集素材 | 非圧縮。ファイルサイズが大きいためWeb配信には不適。 |
| FLAC（.flac） | アーカイブ・高音質保存 | ロスレス圧縮。.ogg/.oga等のOggコンテナ派生も含む。Web配信前に変換が必要。 |
| AIFF（.aif/.aiff） | macOS環境での編集素材 | 非推奨。WAVで代替可能。既存素材の受け入れのみ許容。 |

### Web配信用（ロッシー/圧縮）

| フォーマット | 優先度 | ブラウザ対応 | 用途 |
|---|---|---|---|
| Opus（.opus） | 第一候補 | 互換性スコア92%。Chrome・Firefox・Edge全対応。Safari 18.5以降で完全対応。 | 音声配信の最優先フォーマット。同ビットレートでAAC・MP3より高品質。WebRTCの必須コーデック。 |
| AAC（.m4a/.aac） | 第二候補 | 全主要ブラウザ対応。 | Apple環境との互換性が高い。Opus非対応環境のフォールバック。 |
| MP3（.mp3） | フォールバック | 全ブラウザ対応。 | 音質・圧縮率ではOpus・AACに劣るが互換性は最高。レガシー環境向け最終フォールバック。 |

### 音声配信パターン

```html
<audio controls preload="metadata">
  <source src="audio.opus" type="audio/opus">
  <source src="audio.m4a" type="audio/mp4">
  <source src="audio.mp3" type="audio/mpeg">
</audio>
```

---

## フォントフォーマット

### 採用フォーマット

| フォーマット | 用途 | 備考 |
|---|---|---|
| WOFF2（.woff2） | Web配信の標準 | ブラウザ対応率97%以上。Brotli圧縮によりWOFFより約30%小さい。Web配信では原則WOFF2のみを使用する。 |

参照：[web.dev - Best practices for fonts](https://web.dev/articles/font-best-practices)、[Can I Use WOFF2](https://caniuse.com/woff2)

### 変換前フォーマット（ソースファイル）

| フォーマット | 用途 |
|---|---|
| TTF（.ttf） | WOFF2への変換元。iOS/iPadOS/Android等のネイティブアプリでカスタムフォントを使用する場合に有効。 |
| OTF（.otf） | WOFF2への変換元。デスクトップアプリケーション・DTP用途。 |

TTF/OTFはWOFF2への変換ソースとしてローカルで保持するが、**Webプロジェクトのアセットディレクトリには配置しない。** ネイティブアプリプロジェクトでカスタムフォントが必要な場合のみ、当該プロジェクトのアセットに配置する。

### 廃止フォーマット

WOFF 1.0（.woff）、EOT（.eot）、SVGフォント（.svg）は使用しない。

### フォント運用規則

- `@font-face`ではWOFF2のみを指定する。
- サブセット化を実施し、使用するUnicode範囲のみを含める。日本語フォントはサブセット化の効果が特に大きい。
- `font-display: swap`を指定し、フォント読み込み中にシステムフォントで代替表示する。
- `<link rel="preconnect">`で外部フォントサーバーへの事前接続を行う。セルフホスティングの場合は`<link rel="preload">`でLCP対象テキストのフォントを先行読み込みする。
- フォントファイルにGZIP・Brotli等のサーバー側圧縮を適用しない（WOFF2は圧縮済み）。
- 可変フォント（Variable Fonts）は複数ウェイトが必要な場合にファイル数削減の手段として検討する。

```css
@font-face {
  font-family: "YOUR_FONT_NAME";
  src: url("/fonts/YOUR_FONT_NAME.woff2") format("woff2");
  font-weight: 400;
  font-style: normal;
  font-display: swap;
  unicode-range: U+0000-00FF, U+2000-206F; /* Latin基本+一般句読点 */
}
```

---

## 命名規則

### ファイル名

形式：`kebab-case`で統一する。使用可能な文字は半角英小文字（`a-z`）、数字（`0-9`）、ハイフン（`-`）のみ。

```
{カテゴリ}-{説明}-{バリアント}.{拡張子}
```

| 要素 | 説明 | 例 |
|---|---|---|
| カテゴリ | アセット種別の接頭辞 | `icon-`、`hero-`、`bg-`、`logo-`、`thumb-`、`avatar-` |
| 説明 | 内容を識別する固有名称 | `navigation-menu`、`product-detail`、`team-member` |
| バリアント | サイズ・状態・解像度・テーマ等の区分（必要時） | `-24px`、`-640w`、`-2x`、`-dark`、`-hover` |

### 文字長

ファイル名（拡張子を除く）は**25文字程度を推奨**する。簡潔さと可読性のバランスを優先する。ファイルシステム上の上限は255文字とする。

### バリアント・サフィックス

| 用途 | サフィックス例 | 対象アセット | 備考 |
|---|---|---|---|
| アイコンサイズ | `-16px`、`-20px`、`-24px`、`-32px`、`-48px` | アイコン・ロゴ等の固定サイズアセット | 4pxグリッド準拠の実ピクセル値。主要デザインシステム（Material Design・IBM Carbon等）の標準的なサイズ体系に基づく。 |
| レスポンシブ幅 | `-320w`、`-640w`、`-960w`、`-1280w`、`-1920w` | 写真・ヒーロー・コンテンツ画像 | HTML `srcset`の`w`ディスクリプタと直接対応。ファイル名から`srcset`属性値への変換が自明になる。 |
| 解像度 | `-2x`、`-3x` | 全アセット共通 | Web配信用。`@2x`/`@3x`はiOS/iPadOS/macOSアプリ開発等のローカルアセットに限定し、Web配信では使用禁止。 |
| 状態 | `-hover`、`-active`、`-disabled` | インタラクション状態を持つアセット | |
| テーマ | `-dark`、`-light` | テーマ切替対象アセット | |
| 方向 | `-left`、`-right`、`-up`、`-down` | 向きの区別が必要なアセット | |

**`-sm`/`-md`/`-lg`/`-xl`等の相対的なサイズ表記は使用しない。** すべてのサイズバリアントは具体的な数値（`-{N}px`または`-{N}w`）で表記する。

### 命名例

```
icon-arrow-right.svg
icon-arrow-right-24px.svg
icon-close-16px.svg
hero-landing-main.avif
hero-landing-main-640w.avif
hero-landing-main-1280w.avif
hero-landing-main-2x.avif
bg-gradient-dark.webp
logo-company-full.svg
logo-company-mark-48px.svg
ogp-default.png
ogp-about.png
thumb-article-preview-320w.avif
avatar-user-default.avif
font-noto-sans-jp-regular.woff2
```

### 禁止事項

- スペース、全角文字、大文字、アンダースコア（`_`）をファイル名に使用しない。
- **連番の使用を厳密に禁止する。** `-01`/`-02`、`-a`/`-b`、`-001`等の連番サフィックスは一切使用しない。すべてのファイルに内容を識別可能な固有名称を付与する。名称で区別できない場合はディレクトリ構成またはアセット自体の設計を見直す。
- フォーマット名をファイル名に含めない（`photo-webp.webp`は不可）。
- `@`、`#`、`$`、`%`、`&`、`+`、`=`等の特殊文字を使用しない。

---

## ディレクトリ構成

```
assets/
├── images/                 # サイト内表示用の画像（OGP・ファビコン等の外部配信用は含めない）
│   ├── icons/          # SVGアイコン・アイコンスプライト
│   ├── hero/           # ヒーロー画像・メインビジュアル
│   ├── backgrounds/    # 背景画像
│   ├── logos/          # ロゴ・ブランドマーク
│   ├── thumbnails/     # サムネイル画像
│   └── content/        # 記事・コンテンツ内画像
├── videos/
├── audio/
├── fonts/              # WOFF2ファイルを直置き
├── favicon/            # ファビコン・アプリアイコン
├── ogp/                # OGP（Open Graph Protocol）画像・SNSシェア用画像
└── docs/               # PDF・Markdown等のドキュメント
```

プロジェクト規模に応じてサブディレクトリを追加・省略する。ディレクトリ名はkebab-caseで統一する。

---

## アセット配信ドメイン

大規模サイトや高トラフィック環境では、静的アセットを専用サブドメインまたはCDNドメインから配信することでパフォーマンスと運用の最適化を図れる。

### サブドメイン命名パターン

| パターン | 用途 | 例 |
|---|---|---|
| 統合型 | 全静的アセットを単一サブドメインで配信 | `static.example.com`、`cdn.example.com`、`assets.example.com` |
| アセット種別型 | アセット種別ごとにサブドメインを分離 | `images.example.com`、`fonts.example.com`、`media.example.com` |

統合型を基本とし、キャッシュポリシーの分離や運用上の理由がある場合にのみアセット種別型を検討する。

### 利点

- **クッキーレス配信。** メインドメイン（`www.example.com`）に設定されたCookieが静的アセットのリクエストに付与されなくなり、リクエストヘッダーのサイズが削減される。Cookieが2KB程度のサイトでは、ページあたり数十〜数百リクエスト分のヘッダー削減となる。
- **CDNエッジ配信。** サブドメインのCNAMEをCDNプロバイダに向けることで、ユーザーに地理的に近いエッジサーバーからアセットを配信できる。
- **独立したキャッシュポリシー。** アセット専用ドメインに長期キャッシュ（`Cache-Control: public, max-age=31536000, immutable`）を設定しつつ、メインドメインのHTMLには短いキャッシュを維持できる。
- **オリジンサーバーの負荷軽減。** 静的アセットのリクエストがCDNで処理されるため、オリジンサーバーはHTMLの生成・API処理に集中できる。

### 不利点・注意事項

- **追加のDNSルックアップ。** 新しいドメインごとにDNS解決が発生する。`<link rel="dns-prefetch">`または`<link rel="preconnect">`で事前解決を行い、初回アクセスの遅延を軽減する。
- **追加のTLSハンドシェイク。** メインドメインとは別のTLS接続が必要になる。特にモバイル回線では接続確立のレイテンシが大きい。ワイルドカード証明書（`*.example.com`）を使用すれば、HTTP/2のConnection Coalescingによりブラウザが接続を統合し、ハンドシェイクのオーバーヘッドを回避できる場合がある。
- **CORSの設定が必須。** サブドメインからフォント（WOFF2等）を配信する場合、ブラウザのCORSポリシーによりブロックされる。アセット配信サーバーでフォントファイルに`Access-Control-Allow-Origin`ヘッダーを設定すること。
- **HTTP/2環境ではドメインシャーディングが逆効果。** HTTP/2の多重化により単一ドメインで並列ダウンロードが可能なため、並列化目的で複数サブドメインに分散する手法（ドメインシャーディング）は不要であり、ストリーム優先度の最適化を妨げる。アセット配信サブドメインは**1つに統合**し、CDNのエッジ配信やクッキーレス化の利点を活用する構成とする。

### CORS設定例

```
# Apache - フォントファイルにCORSヘッダーを付与
<FilesMatch "\.(woff|woff2)$">
  Header set Access-Control-Allow-Origin "https://www.example.com"
</FilesMatch>
```

```
# Nginx - フォントファイルにCORSヘッダーを付与
location ~* \.(woff|woff2)$ {
  add_header Access-Control-Allow-Origin "https://www.example.com";
}
```

`Access-Control-Allow-Origin`にはワイルドカード（`*`）ではなくメインドメインを明示的に指定することを推奨する。

### 事前接続の指定

```html
<!-- アセット配信ドメインへの事前接続（DNS+TCP+TLS） -->
<link rel="preconnect" href="https://static.example.com" crossorigin>

<!-- DNSのみの事前解決（preconnectが使えない場合のフォールバック） -->
<link rel="dns-prefetch" href="https://static.example.com">
```

`crossorigin`属性はフォント等のCORSリクエストに必要な接続を事前確立するために指定する。

---

## 圧縮・最適化基準

### 画像

| フォーマット | 推奨品質設定 | 目安ファイルサイズ |
|---|---|---|
| AVIF（ロッシー） | 品質値：CQ 28〜36（ツール依存） | ヒーロー画像200KB以下、サムネイル50KB以下 |
| WebP（ロッシー） | 品質値：80〜85 | ヒーロー画像300KB以下、サムネイル80KB以下 |
| PNG | pngquant等でロスレス最適化 | アイコン・UI要素は10KB以下を目標 |
| SVG | SVGO等で最適化 | 不要メタデータ除去後5KB以下を目標 |

品質設定はあくまで出発点であり、対象画像ごとに視覚品質を確認して調整する。品質と圧縮率の検証には [Squoosh](https://squoosh.app/) を使用する。

### 動画

- 背景動画・ループ動画は音声トラックを除去する。
- 長尺動画はストリーミング配信（HLS/DASH）を検討する。

#### Web配信用ビットレート目安（H.264基準）

| 解像度 | 標準フレームレート（24/25/30fps） | 高フレームレート（48/50/60fps） |
|---|---|---|
| 4K（2160p） | 35–45 Mbps | 53–68 Mbps |
| 2K（1440p） | 16 Mbps | 24 Mbps |
| 1080p | 8 Mbps | 12 Mbps |
| 720p | 5 Mbps | 7.5 Mbps |

- 上記はYouTube公式推奨アップロード設定（H.264 AVC、SDR、VBR）に準拠する。
- Webでの一般的な配信上限は1080pとする。4K配信は帯域・再生環境を考慮し必要時のみ採用する。
- HDR配信は同解像度のSDR比で約25–30%高いビットレートが必要となる（例：1080p/30fps SDR 8Mbps → HDR 10Mbps）。

#### コーデック別ビットレート補正

H.265・AV1使用時は、上記H.264基準値より低いビットレートで同等画質を実現できる。

| コーデック | 対H.264ビットレート削減率 | 備考 |
|---|---|---|
| H.265（HEVC） | 約50% | 例：1080p/30fps H.265 @ 4Mbps ≒ H.264 @ 8Mbps |
| AV1 | 約50–65%（高解像度ほど効率的） | ロイヤリティフリー。エンコード時間はH.265の5–10倍。 |

- `<source>`要素でコーデック別ソースを提供し、ブラウザが最適なものを選択できるようにする。

#### ローカルファイル（編集素材・アーカイブ）

- 収録時はカメラ・ツールの最高品質設定を使用する。
- 目安：1080pで50 Mbps以上、4Kで100 Mbps以上。
- 制作・編集用の中間コーデック（ProRes・DNxHD/HR・CineForm）の仕様は「フォーマット選定基準」セクションを参照。

### 音声

- **最低ビットレート：128kbps Stereo。** 音声のみのファイル・映像内の音声トラックの双方に適用する。**192kbps以上を推奨。**
- **ローカルファイルの場合は256kbps以上またはロスレス（FLAC・WAV）を推奨する。**
- フォーマット別ビットレート目安（Web配信用）：Opusは128〜192kbps、AACは128〜256kbps、MP3は192〜320kbps。
- 音声の種類（音楽・ナレーション・効果音）に応じてビットレートを調整する。ただし最低ビットレート（128kbps Stereo）を下回ってはならない。

### フォント

- サブセット化で使用文字のみを含める。
- 日本語フォントは漢字のUnicodeブロック単位で分割し、`unicode-range`で必要なブロックのみを読み込む（Google Fontsの分割方式を参考にする）。

---

## ファビコン・アプリアイコン

| ファイル | フォーマット | サイズ | 用途 |
|---|---|---|---|
| favicon.ico | ICO | 32×32（16×16を内包） | レガシーブラウザ向け |
| favicon.svg | SVG | 可変 | モダンブラウザ向け。ダークモード対応可能。 |
| apple-touch-icon.png | PNG | 180×180 | iOS/iPadOS Safari |
| icon-192.png | PNG | 192×192 | PWA（manifest.json） |
| icon-512.png | PNG | 512×512 | PWA（manifest.json） |

```html
<link rel="icon" href="/favicon.ico" sizes="32x32">
<link rel="icon" href="/favicon.svg" type="image/svg+xml">
<link rel="apple-touch-icon" href="/apple-touch-icon.png">
```

参照：[How to Favicon in 2024（Evil Martians）](https://evilmartians.com/chronicles/how-to-favicon-in-2021-six-files-that-fit-most-needs)

---

## ドキュメント形式

プロジェクトで配布・共有・管理するドキュメントファイルの標準フォーマット。

### 採用フォーマット

| フォーマット | 主な用途 | 備考 |
|---|---|---|
| PDF（.pdf） | 最終成果物の配布・印刷用入稿・契約書等の書面。レイアウトが環境に依存せず固定される。 | フォーム入力可能なPDFはアクセシビリティ（タグ付きPDF）を考慮する。Web埋め込みは`<object>`または`<iframe>`で提供し、ダウンロードリンクを併記する。ファイルサイズが大きい場合は画像の圧縮やフォントのサブセット埋め込みで最適化する。 |
| Markdown（.md） | プロジェクト内ドキュメント・README・技術仕様書・変更履歴（CHANGELOG）。 | プレーンテキストベースでバージョン管理（Git diff）に適する。記法は [CommonMark](https://commonmark.org/) 準拠を基本とし、GitHub Flavored Markdown（GFM）拡張（テーブル・タスクリスト・脚注）を許容する。 |

### 運用規則

- PDFは最終出力（読み取り専用）として使用し、編集が必要なドキュメントのソースはMarkdownまたは他のテキスト形式で管理する。
- Markdownファイル内の画像参照は相対パスとし、本ガイドラインの画像フォーマット規則に従う。
- Markdownのファイル名はkebab-caseで統一する（例：`project-setup-guide.md`）。

---

## 構成の根拠

| 判断事項 | 根拠 |
|---|---|
| AVIFを写真系第一候補に採用 | 2024年1月に全主要ブラウザ対応完了。ブラウザ互換性スコア92%。WebPより圧縮率が高い。 |
| GIF禁止 | 256色制限、圧縮効率が低い。WebP・APNGが完全な上位互換。 |
| HEIC条件付き採用 | ブラウザ互換性スコア13%。Safari以外未対応。ライセンス問題。iOS/iPadOS/macOSネイティブアプリでは基本的に採用可能だが、Web配信では変換必須。 |
| JPEG XLを経過観察 | Chromiumがデフォルト有効化していない（2026年2月時点でフラグ必須）。 |
| WOFF2のみ採用 | web.dev公式推奨。ブラウザ対応率97%以上。WOFF比30%高圧縮。 |
| Opusを音声第一候補 | ロイヤリティフリー。ブラウザ互換性スコア92%。同ビットレートでAAC・MP3より高品質。 |
| ProRes・DNxHD/HR・CineFormを制作用に採用 | 業界標準の中間コーデック3種を網羅。ProResはmacOS/Final Cut Pro環境の標準。DNxHD/HRはクロスプラットフォーム対応。CineFormはオープンソースで高解像度に強い。いずれもイントラフレーム圧縮による高い編集効率と再エンコード耐性を持つ。 |
| TIFFを制作・保存用に採用 | 16bit/チャネル・CMYK・レイヤー保持等、制作段階で必要な品質情報を完全に保持できる。ブラウザ標準サポートがないためWeb配信には使用しない。 |
| PDF・Markdownをドキュメント形式に採用 | PDFは環境非依存のレイアウト固定に適し最終成果物の配布に標準的。MarkdownはGit管理との親和性が高くプロジェクト内ドキュメントに最適。 |
| kebab-case・カテゴリ接頭辞型を採用 | kebab-caseはNext.js・Biome等の主要ツールが推奨するWebアセットのデファクト標準。URLとの自然な整合性を持ち、OS間のcase-sensitivity問題を回避する。カテゴリ接頭辞型はファイル単体で用途が判別可能。 |
| 連番サフィックスを厳密に禁止 | 連番は内容の識別性を持たず、アセットの増減時に管理が破綻する。すべてのファイルに固有名称を付与することでアセットの目的と内容を保証する。 |
| Web解像度サフィックスに`-2x`を採用 | `@2x`はApple開発エコシステムの慣習だがURLエンコードが必要になる場合がある。`-2x`はkebab-caseとの一貫性を維持しWeb配信に安全。 |
| サイズサフィックスに`-{N}px`・`-{N}w`を採用 | `-sm`/`-lg`等の相対表記はプロジェクト間で定義が不統一になる。`-{N}px`はMaterial Design・IBM Carbon等の主要デザインシステムの4pxグリッド体系と整合する。`-{N}w`はHTML `srcset`の`w`ディスクリプタと直接対応し、ファイル名からマークアップへの変換が自明になる。 |
| 映像ビットレートにYouTube公式SDR値を採用 | H.264基準の業界標準値として最も広く参照される。H.265・AV1のコーデック補正係数はITU-T仕様およびAOMedia学術比較（MDPI 2024）に基づく。 |
| 音声最低128kbps Stereo | 128kbps未満では音楽・効果音でアーティファクトが知覚される。YouTube公式もStereo 384kbps・Mono 128kbpsを推奨。192kbps以上で知覚的に透明な品質を確保。 |
| Data URIの埋め込み上限を元ファイル1KB以下に設定 | Base64エンコードで約33%増加するため、1KBの画像は約1.33KBのCSS/HTML増加となる。RFC 2397およびWikipediaの推奨（サーバー圧縮非対応時は1KB以下）に準拠。HTTP/2多重化環境ではリクエストオーバーヘッドが軽減されるため、大きな画像の埋め込みは費用対効果が低い。 |
| アセット配信サブドメインは1つに統合 | HTTP/2多重化により並列化目的のドメインシャーディングは不要かつ逆効果（Cloudflare・MDN・imgix等が明示的にアンチパターンと指摘）。CDNサブドメイン1つに統合し、クッキーレス配信・エッジ配信・独立キャッシュポリシーの利点を維持しつつ、DNS/TLSオーバーヘッドを最小化する。ワイルドカード証明書＋同一IPでHTTP/2 Connection Coalescingを有効化すればハンドシェイクも統合される。 |
| `<video>`にwidth/height属性を推奨 | CLS（Cumulative Layout Shift）防止のため、ブラウザがアスペクト比を事前算出しレイアウトシフトを回避できるようにする。web.dev・MDNが推奨するプラクティス。 |
| `<track>`によるWebVTTキャプションを標準化 | WCAG 1.2.2（キャプション - 収録済み）準拠。W3C WAI Technique H95で`<track kind="captions">`の使用が明示的に規定されている。WebVTTは全主要ブラウザ対応のテキストトラック形式。 |
| 背景動画に`aria-hidden="true"`を指定 | 装飾目的の動画をスクリーンリーダーが読み上げることを防止する。WAI-ARIAプラクティスに準拠。`prefers-reduced-motion`対応と併せてアクセシビリティを確保する。 |
| `<source>`の`media`属性を条件付き採用 | 2014年にHTML仕様から削除されたが、2024年以降Safari・Firefoxで再サポート。Chromium系は未対応のため、フォールバック設計が必須。全ブラウザ対応にはHLS/DASH等のアダプティブストリーミングが必要。Scott Jehl・Filament Groupが再導入を推進し、W3C仕様に再統合された。 |
| メディアフラグメントURI（`#t=`）を記載 | W3C Media Fragments URI 1.0勧告（2012年）に基づく標準的な時間範囲指定。Firefox・WebKit/Safari・Chromium系で実装済み。ブラウザ側シーク処理のため追加サーバー対応不要。 |
| `::cue`擬似要素によるキャプションスタイリングを記載 | MDN公式リファレンスおよびCSS仕様に基づく。全主要ブラウザで`::cue`をサポート。使用可能CSSプロパティはcolor・background・font・text-decoration・outline等に限定される（CSS仕様による制約）。 |

## 出典・参考情報
- [Can I Use](https://caniuse.com/) - ブラウザ対応状況の確認に使用。
- [Squoosh](https://squoosh.app/) - 画像の品質・圧縮率比較に使用。Google提供。
- [SVGO](https://svgo.dev/) - SVG最適化ツール。
- [web.dev - Best practices for fonts](https://web.dev/articles/font-best-practices) - Webフォント最適化の公式推奨事項。
- [web.dev - Use modern image formats](https://web.dev/articles/choose-the-right-image-format) - 画像フォーマット選定の公式ガイド。
- [MDN - Web audio codec guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Audio_codecs) - Web音声コーデックの互換性と特性。
- [MDN - Web video codec guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Video_codecs) - Web動画コーデックの互換性と特性。
- [Apple ProRes（Apple公式）](https://support.apple.com/en-us/102207) - ProResファミリーの公式仕様。
- [Apple ProRes White Paper](https://www.apple.com/final-cut-pro/docs/Apple_ProRes.pdf) - ProResの技術詳細（Apple公式ホワイトペーパー）。
- [Frame.io - Compare 50 Intermediate Codecs](https://blog.frame.io/2017/02/13/compare-50-intermediate-codecs/) - 中間コーデックのデータレート比較表。
- [How to Favicon in 2024（Evil Martians）](https://evilmartians.com/chronicles/how-to-favicon-in-2021-six-files-that-fit-most-needs) - ファビコン構成の実践ガイド。
- [CommonMark Spec](https://commonmark.org/) - Markdown記法の標準仕様。
- [GitHub Flavored Markdown Spec](https://github.github.com/gfm/) - GFM拡張仕様（テーブル・タスクリスト・脚注等）。
- [Biome - useFilenamingConvention](https://biomejs.dev/linter/rules/use-filenaming-convention/) - ファイル命名規則のLintルール。kebab-case等のケース規則を自動検証。
- [Material Icons Guide（Google）](https://developers.google.com/fonts/docs/material_icons) - アイコンサイズ体系（18/24/36/48px）の公式ガイド。
- [IBM Carbon - UI Icons Usage](https://carbondesignsystem.com/elements/icons/usage/) - アイコンサイズ体系（16/20/24/32px）とテキスト対応の公式ガイド。
- [MDN - Responsive images](https://developer.mozilla.org/en-US/docs/Web/HTML/Guides/Responsive_images) - `srcset`・`w`ディスクリプタ・`sizes`属性の公式リファレンス。
- [Cloud Four - Responsive Images 101](https://cloudfour.com/thinks/responsive-images-101-part-4-srcset-width-descriptors/) - `srcset`幅ディスクリプタの実践解説。
- [YouTube - Recommended upload encoding settings](https://support.google.com/youtube/answer/1722171?hl=en) - 映像・音声ビットレートの公式推奨値。
- [MDPI Electronics - Performance Comparison of VVC, AV1, HEVC, and AVC (2024)](https://www.mdpi.com/2079-9292/13/5/953) - コーデック間圧縮効率の学術比較。
- [MDN - data: URLs](https://developer.mozilla.org/en-US/docs/Web/URI/Reference/Schemes/data) - Data URIスキームの公式リファレンス。
- [MDN - Domain sharding](https://developer.mozilla.org/en-US/docs/Glossary/Domain_sharding) - ドメインシャーディングの定義とHTTP/2での非推奨化。
- [Cloudflare - HTTP/2 For Web Developers](https://blog.cloudflare.com/http-2-for-web-developers/) - HTTP/2環境でのドメインシャーディング回避・多重化の公式解説。
- [daniel.haxx.se - HTTP/2 connection coalescing](https://daniel.haxx.se/blog/2016/08/18/http2-connection-coalescing/) - HTTP/2 Connection Coalescingの技術詳細。ワイルドカード証明書＋同一IPによる接続統合。
- [MDN - `<video>`: The Video Embed element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/video) - `<video>`要素の全属性リファレンス。
- [MDN - `<track>`: The Embed Text Track element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/track) - `<track>`要素の属性・kind種別・WebVTT連携の公式リファレンス。
- [W3C WAI - H95: Using the track element to provide captions](https://www.w3.org/WAI/WCAG22/Techniques/html/H95) - WCAG 1.2.2準拠のキャプション実装技法。`<track kind="captions">`の使用例。
- [MDN - Accessible multimedia](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Accessibility/Multimedia) - 動画・音声のアクセシビリティ実装ガイド。WebVTT・字幕・キャプションの実践解説。
- [MDN - `<source>`: The Media or Image Source element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/source) - `<source>`要素の属性（`src`・`type`・`media`）の公式リファレンス。
- [W3C - Media Fragments URI 1.0](https://www.w3.org/TR/media-frags/) - メディアフラグメントURI（`#t=`時間指定等）のW3C勧告仕様。
- [MDN - `::cue`](https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/::cue) - WebVTTキューのCSS擬似要素スタイリング。使用可能プロパティ一覧。
- [Scott Jehl - How to Use Responsive HTML Video](https://scottjehl.com/posts/using-responsive-video/) - `<source media="">`属性によるレスポンシブ動画配信の実践解説。
