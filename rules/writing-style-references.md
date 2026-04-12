---
title: "Writing Style References"
description: "日本語・英語ライティングリファレンス一覧 / Japanese and English writing style references"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-21T14:02+09:00"
lang: "ja"
---

# Writing Style References

AIプロンプトに転用可能な日本語・英語の文章スタイルガイド・リファレンス集。
紙の書籍・電子書籍（直接読み込み不可）は除外。

---

## 日本語リファレンス

### スタイルガイド

| 名称 | URL | 形式 | 特徴 |
|------|-----|------|------|
| **JTF日本語標準スタイルガイド（翻訳用）第3.0版** | https://www.jtf.jp/pdf/jtf_style_guide.pdf | PDF | 翻訳向け日本語表記ガイドライン、12の基本ルール、CC BY 4.0 |
| **外来語（カタカナ）表記ガイドライン第3版** | http://www.jtca.org/standardization/ | Web | カタカナ長音・複合語の表記基準 |
| **Microsoft 日本語スタイルガイド** | https://www.microsoft.com/ja-jp/language/styleguides | PDF | Microsoft製品の日本語翻訳基準 |
| **WordPress 翻訳ガイドライン** | https://ja.wordpress.org/team/handbook/translation/ | Web | OSS翻訳向け、Sunスタイルガイドベース |
| **「やさしい日本語」の作り方** | http://human.cc.hirosaki-u.ac.jp/kokugo/EJ9tsukurikata2.htm | Web | 平易な日本語の作成ルール |

### 内閣告示・公的基準（文化庁）

| 名称 | URL | 概要 |
|------|-----|------|
| **常用漢字表（平成22年内閣告示第2号）** | https://www.bunka.go.jp/kokugo_nihongo/sisaku/joho/joho/kijun/naikaku/kanji/ | 2,136字の漢字使用目安 |
| **現代仮名遣い（昭和61年内閣告示第1号）** | https://www.bunka.go.jp/kokugo_nihongo/sisaku/joho/joho/kijun/naikaku/gendaikana/ | 仮名遣いのよりどころ |
| **送り仮名の付け方（昭和48年内閣告示第2号）** | https://www.bunka.go.jp/kokugo_nihongo/sisaku/joho/joho/kijun/naikaku/okurikana/ | 送りがなの基準 |
| **外来語の表記（平成3年内閣告示第2号）** | https://www.bunka.go.jp/kokugo_nihongo/sisaku/joho/joho/kijun/naikaku/gairai/ | 外来語カタカナ表記基準 |
| **ローマ字のつづり方（昭和29年内閣告示第1号）** | https://www.bunka.go.jp/kokugo_nihongo/sisaku/joho/joho/kijun/naikaku/roma/ | ローマ字表記基準 |
| **公用文作成の考え方（令和4年）** | https://www.bunka.go.jp/seisaku/bunkashingikai/kokugo/hokoku/93650001_01.html | 公用文の書き方指針 |

### JIS規格

| 名称 | URL | 概要 |
|------|-----|------|
| **JIS Z 8301:2019 規格票の様式及び作成方法** | http://kikakurui.com/z8/Z8301-2019-01.html | 規格文書の作成方法 |

### 文章校正ツール（ルール参照用）

| 名称 | URL | 概要 |
|------|-----|------|
| **textlint** | https://github.com/textlint/textlint | 日本語文章校正ツール（ルール定義参照可能） |
| **textlint-rule-preset-JTF-style** | https://github.com/textlint-ja/textlint-rule-preset-JTF-style | JTFスタイルガイドのtextlintルール実装 |
| **RedPen** | https://redpen.cc/docs/latest/index.html | 文書校正ツール（Validator一覧が参考になる） |
| **prh（proofread-helper）** | https://github.com/prh/prh | 校正ルール辞書形式 |

---

## 英語リファレンス

### テクニカルライティング

| 名称 | URL | 形式 | 特徴 |
|------|-----|------|------|
| **Google Developer Documentation Style Guide** | https://developers.google.com/style | Web | テクニカルドキュメント向け、構造化されておりAI転用しやすい |
| **Microsoft Writing Style Guide** | https://learn.microsoft.com/en-us/style-guide/ | Web | 幅広い文書タイプ対応、ボイス＆トーン定義が明確 |
| **Red Hat Style Guide** | https://stylepedia.net/ | Web | オープンソース、翻訳・グローバル対応充実 |
| **GitLab Documentation Style Guide** | https://docs.gitlab.com/ee/development/documentation/styleguide/ | Web | API・技術ドキュメント向け |
| **DigitalOcean Technical Writing Guidelines** | https://www.digitalocean.com/community/tutorials/digitalocean-s-technical-writing-guidelines | Web | ステップバイステップ文書に強い |

### コンテンツライティング

| 名称 | URL | 形式 | 特徴 |
|------|-----|------|------|
| **Mailchimp Content Style Guide** | https://styleguide.mailchimp.com/ | Web | マーケティング文書向け、読みやすさ重視 |
| **GOV.UK Style Guide** | https://www.gov.uk/guidance/style-guide | Web | 英国政府の平易な英語ガイド |

### Web標準・仕様書

| 名称 | URL | 概要 |
|------|-----|------|
| **WHATWG HTML Living Standard** | https://html.spec.whatwg.org/ | HTML仕様 |
| **W3C CSS Specifications** | https://www.w3.org/Style/CSS/ | CSS仕様 |
| **MDN Web Docs** | https://developer.mozilla.org/ | Web技術リファレンス |
| **WCAG 2.2** | https://www.w3.org/TR/WCAG22/ | アクセシビリティガイドライン |
| **WAI-ARIA 1.2** | https://www.w3.org/TR/wai-aria-1.2/ | アクセシビリティAPI仕様 |

---

## 推奨優先順位

### 日本語（AIプロンプト転用向け）

1. **JTF日本語標準スタイルガイド** - PDFで全文読み込み可能、CC BY 4.0
2. **textlint-rule-preset-JTF-style** - ルール定義がJSON/JS形式で参照しやすい
3. **文化庁 内閣告示** - 公的基準として信頼性高い

### 英語（AIプロンプト転用向け）

1. **Google Developer Documentation Style Guide** - 構造化されておりAI転用しやすい
2. **Microsoft Writing Style Guide** - ボイス＆トーン定義が明確
3. **Mailchimp Content Style Guide** - マーケティング文書向け

---

## 関連ツール

| 名称 | URL | 用途 |
|------|-----|------|
| **textlint** | https://textlint.github.io/ | 日本語文章校正 |
| **RedPen** | https://redpen.cc/ | 文書校正 |
| **Vale** | https://vale.sh/ | 英語文章校正（スタイルガイド対応） |
| **write-good** | https://github.com/btford/write-good | 英語文章改善提案 |

---

## 出典・参考情報
- [日本語文章のスタイルガイドのまとめ - Qiita](https://qiita.com/azu/items/623e5f50ccac2d4a8ac8) - by @azu

---

**注記**: 紙の書籍・電子書籍（直接AI読み込み不可）は除外。
- 『日本語スタイルガイド（第3版/第4版）』TC協会編著
- 『用字用語 新表記辞典（新訂四版）』第一法規
- 『新しい国語表記ハンドブック』三省堂
