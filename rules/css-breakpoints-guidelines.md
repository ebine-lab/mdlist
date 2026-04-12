---
title: "CSS Breakpoints Guidelines"
description: "CSSブレークポイント規定 - レスポンシブデザインの閾値定義 / CSS breakpoint definitions for responsive design"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-06T01:25+09:00"
lang: "ja"
---

# CSS Breakpoints Guidelines

**説明** - CSSブレークポイントの規定、画面幅区分と旧機種向け狭幅対応の定義を含む


## 目的
本規定はWeb UIの画面幅区分を統一し、実装とデザインの不一致を防ぐことを目的とする。
ブレークポイントはモバイルファースト設計を基本概念とする。

## 画面幅区分
| 区分 | 最小幅 | 最大幅 | 用途 |
| --- | --- | --- | --- |
| Mobile | 0px | 767px | 基本レイアウト |
| Tablet | 768px | 1024px | 列数と余白の拡張 |
| Desktop | 1025px | なし | 情報密度を最大化 |

## 構成の根拠
対象調査国は `en-US`、`en-GB`、`en-CA`、`de-DE`、`fr-FR`、`ja-JP`、`zh-TW` とする。
対象国の統計ではMobile解像度として390x844や414x896が上位に含まれ、Desktop解像度では1920x1080が上位に含まれるため、MobileとDesktopの分布差が大きい。
一般的なフレームワークでは768px前後を中間帯の境界として採用し、1024px前後を上位帯の境界として採用する例があるため、Tabletを768pxから1024px、Desktopを1025pxからとする。

## 適用方針
Mobileを基底とし、`min-width`で上書きする。区分間に空白を作らない。

```css
/* base: Mobile */
@media (min-width: 768px) {
  /* Tablet */
}

@media (min-width: 1025px) {
  /* Desktop */
}
```

## Tailwind向けの書式
TailwindではMobileを基底とし、TabletとDesktopを`screens`で定義する。

```js
// tailwind.config.js
module.exports = {
  theme: {
    screens: {
      tablet: "768px",
      desktop: "1025px"
    }
  }
}
```

## 旧機種向け狭幅対応
狭幅例外は最大幅360pxとする。狭幅例外は専用の層または専用のスタイルファイルに隔離する。

```css
@layer legacy {
  @media (max-width: 360px) {
    /* 旧機種向けの最小限の差分 */
  }
}
```

## 旧機種向け狭幅対応のTailwind向けの書式
Tailwindでは`legacy-narrow`を`screens`に追加し、HTMLには数値を直書きしない。

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      screens: {
        "legacy-narrow": { max: "360px" }
      }
    }
  }
}
```

```css
/* legacy-narrow.css */
.legacy-narrow-compact {
  @apply legacy-narrow:px-3 legacy-narrow:text-sm;
}
```

## 廃止方針
廃止目標時期は2029年ごろとする。対象時期以降は`legacy`層または`legacy-narrow.css`を削除する。

## 出典・参考情報
- [StatCounter Screen Resolution Stats United States Of America](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile-tablet/united-states-of-america) - 米国のデスクトップ、モバイル、タブレットの画面解像度分布。
- [StatCounter Screen Resolution Stats United Kingdom](https://gs.statcounter.com/screen-resolution-stats/mobile-tablet/united-kingdom) - 英国のモバイルとタブレットの画面解像度分布。
- [StatCounter Screen Resolution Stats Canada](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile-tablet/canada/2018) - カナダのデスクトップ、モバイル、タブレットの画面解像度分布。
- [StatCounter Screen Resolution Stats Germany](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile-tablet/germany) - ドイツのデスクトップ、モバイル、タブレットの画面解像度分布。
- [StatCounter Screen Resolution Stats France](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile/france) - フランスのデスクトップとモバイルの画面解像度分布。
- [StatCounter Screen Resolution Stats Japan](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile-tablet/japan) - 日本のデスクトップ、モバイル、タブレットの画面解像度分布。
- [StatCounter Screen Resolution Stats Taiwan](https://gs.statcounter.com/screen-resolution-stats/desktop-mobile-tablet/taiwan) - 台湾のデスクトップ、モバイル、タブレットの画面解像度分布。
- [Tailwind CSS Screens](https://v3.tailwindcss.com/docs/screens) - Tailwindの`screens`定義。
- [Bootstrap Breakpoints](https://getbootstrap.com/docs/5.3/layout/breakpoints/) - 代表的フレームワークの区切り幅。
- [MDN Using Media Queries](https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Media_queries/Using) - メディアクエリの基本概念。
