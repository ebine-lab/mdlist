---
title: "デザイングリッド前提と実数プリセット（Figma用）"
description: "デザイングリッド規定 - Figma用グリッドプリセットと実装仕様 / Design grid specification - Figma grid presets and implementation"
version: "1.0.0"
status: "Stable"
last_updated: "2026-03-05T14:23+09:00"
lang: "ja"
---

## A. 提供指示ブロック（必ず含む）

1) レイアウト技術の前提  
CSS Subgrid が2026-03-15に Baseline Widely Available 予定。主要ブラウザは2024時点で実質100%近く対応済みで、フォールバック不要で採用可。(web-platform-dx.github.io)  
Container Queries は2025 State of CSS で認知86%・実使用7%。普及はこれからだが、主要ブラウザで実用域に到達。コンポーネント単位のブレークポイント設計が現実解に。(blog.logrocket.com)

2) グリッド列数・幅の傾向  
デスクトップ: 12列が依然デファクト。ただし「破りグリッド（非対称・重なり）」を12列ベースで実装し、視覚的リズムを付けるケースが増加。(techbuild.me)  
ワイド画面(≥1440px)では16列や24列を導入し、カード群やダッシュボードで細分化する事例が増えている（特にSaaS管理画面）。  
モバイル: 4–6列のシンプルなフラクショナルグリッド（repeat(auto-fit,minmax(…))）が主流。固定ピクセル列指定はほぼ撤退。

3) 単位とガター/マージン  
幅指定はfr＋minmax()が標準。パーセントは丸め差異が残るため大型デザインシステムでは敬遠され、frかclamp()での流体幅が推奨。(wired.com)  
ガターは8pt系スケール（8/12/16/24px）が継続。ダーク/ライト・高コントラスト両対応を考慮し、clamp(12px,1vw,24px)の流体指定が増加。  
コンテナクエリ＋clamp()でガター・余白をコンポーネントごとに最適化する流れが強まっている。

4) ブレークポイント設計  
ビューポート基準だけでなく「親カード幅」で分岐する Container Queries に移行。従来の768/1024/1280px固定分岐は「ベースライン」扱いになり、局所最適のサブブレークポイントを各コンポーネントが持つ設計が推奨。(codercops.com)  
Subgrid により、ネストしたカード内でも外側グリッドの列線に揃えやすくなり、複数行カードを揃える「編集グリッド」「記事カード」などで活用が進む。(frontendtools.tech)

5) “固定px vs パーセント vs 流体” の整理  
固定px: ヒーロー帯の最大幅（例: 1200–1440px）やタイポライン長の上限設定など「上限値」用途に限定。  
パーセント: レガシー理由以外で新規採用は減少。代替はfr。  
流体（fr / minmax / clamp）: 2026年のデフォルト。ギャップやカード幅にclamp()、列幅にminmax(16rem,1fr)を組み合わせるパターンが主流。

6) 近未来（〜2028）の見通し  
Subgridが「標準実装」として浸透し、デザインシステムのレイアウト原子に組み込まれる（カード内の見出し・本文・CTAの縦ライン揃えがノーコード化）。  
Container Queriesの実利用率が30–40%まで上昇し、ページ全体のブレークポイント定義が最小化される。  
非対称・重なり表現（Broken Grid）がブランド差別化要素として定着する一方、アクセシビリティを担保するための「論理的読み順」との両立が評価指標になる。(techbuild.me)

実装リコメンド（すぐ使えるスニペット）
```
.page {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: clamp(12px, 1vw, 24px);
}

.card {
  grid-column: span 4;
  display: grid;
  grid-template-rows: subgrid; /* 2026以降は素直に使う */
  grid-row: 1 / span 3;        /* 高さ揃え */
}

@container (min-width: 480px) { .card { grid-column: span 3; } }
@container (min-width: 768px) { .card { grid-column: span 2; } }

@media (max-width: 767px) {
  .page { grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); }
}
```

導入ロードマップ（提案）  
- 新規ページは即Subgrid + frベースで設計。レガシーサポート不要ならフォールバックを外す。  
- 既存12列レイアウトは、グリッド定義をトークン化（列幅=fr、ガター=clamp）し、Container Query対応コンポーネントへ段階移行。  
- デザインシステムに「コンテナブレークポイント表」を追加（例: xs 0–359px / sm 360–599 / md 600–959 / lg 960–1279 / xl 1280–1599 / 2xl 1600+）、ただしコンポーネントは必要分のみ参照。  
- Broken Grid採用時は読み順（DOM順）を崩さず、視覚効果はgrid-row/columnとtransformで実装し、フォーカス順が破綻しないかをWCAGチェック。  

## B. 画面タイプ別プリセット（Figmaに即設定可能）
| 画面タイプ | 幅レンジ(px) | 列数 | コンテナ最大幅 | 左右マージン | ガター | 用途 |
| --- | --- | --- | --- | --- | --- | --- |
| Mobile S | 320–399 | 4 | 312 | 16 | 12 | 小型端末 |
| Mobile M | 400–479 | 4 | 360 | 16 | 12 | 一般スマホ |
| Mobile L | 480–599 | 6 | 432 | 16 | 12 | 大画面・横持ち |
| Tablet | 600–959 | 6 | 880 | 24 | 16 | タブレット |
| Tablet Wide | 960–1199 | 8 | 1040 | 32 | 16 | 11–13インチ |
| Desktop Base | 1200–1439 | 12 | 1200 | 32 | 20 | 汎用Web/LP |
| Desktop Wide | 1440–1679 | 12 | 1320 | 48 | 24 | ワイドLP |
| Desktop XL | 1680–1919 | 16 | 1440 | 64 | 24 | 高密度UI |
| Dashboard Pro | 1920–2399 | 16–24 | 1580 | 80 | 24–28 | 管理画面/可視化 |
| Cinema / Ultra | 2400+ | 24 | 1680 | 96 | 28–32 | 4K/ウルトラワイド |

補足:  
- 長文はナロー容器720–920px（45–75字/行）を別スタイルで保持。  
- ガター上限はデスクトップ24px、ワイドでも32px程度で抑制。  
- サイドバー併用例：左240px＋残り12列（ガター20px）。  
- ダッシュボードはカード幅を4/6/8/12列スパンの倍数で運用すると再配置が容易。  

## C. ブレークポイント設計と単位運用
- ビューポート固定（例 480/768/1024+）はベースライン。親幅起点のサブブレークポイントを併用。  
- 固定pxは上限値用途に限定。パーセントは原則不採用、fr/clampで流体化。  

## D. Figma設定フロー（簡略）
1. Frame幅に応じてプリセットを選択。  
2. Layout Gridに列数・ガター・マージンを入力（センター）。  
3. コンテナ幅スタイル（ナロー720–920 / スタンダード1200/1320 / ワイド1440/1580）を登録。  
4. スペーシング変数は8pt系（4/8/12/16/24/32/48/64）。  
5. ダッシュボード用に「16列・ガター24・マージン64」など専用グリッドを別スタイルで保持。  

## 出典・参考情報
- web-platform-dx.github.io（Subgrid Baseline）
- blog.logrocket.com（Container Queries動向）
- techbuild.me（Broken Grid潮流）
- wired.com（%指定の丸め問題）
- codercops.com（コンテナ幅起点設計）
- frontendtools.tech（Subgrid活用）
- MetLife / Wikimedia / Festa / GitLab各DS公開値（本文反映済み）
