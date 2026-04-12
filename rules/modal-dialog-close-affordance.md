---
title: "Modal Dialog Close Affordance"
description: "モーダルダイアログとWallの分類・設計根拠・閉じるアフォーダンス検証プロトコル / Modal dialog and wall classification, design rationale, close affordance verification"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-08T12:00+09:00"
lang: "ja"
---

# Modal Dialog Close Affordance

**UIガイドライン** - モーダルダイアログとWall（インタースティシャル）の分類・設計根拠・検証プロトコル


---

## 概要

対象：Web UI/UX（モーダル / ダイアログ / インタースティシャル）。本資料は「右上/左上」を同一カテゴリ（コーナーにCloseがある）として扱う。

モーダルダイアログには、ユーザーが退出できる視認可能な閉じる要素（× 閉じるボタン/Close/Cancel等）を含めることがW3C WAI-ARIA APGにより強く推奨されている。閉じられない、あるいは閉じても利用継続できないUIは、モーダルダイアログではなくインタースティシャル・ウォール（Interstitial Wall、以降「Wall」）として別カテゴリで論じられる。

### 本資料で言えること（一次資料に明記）

- モーダルダイアログには視認可能な閉じる要素を含めることが強く推奨される（WAI-ARIA APG）。
- Escキーによるクローズとフォーカス制御はアクセシビリティ要件である（WCAG Technique H102、MDN）。
- 閉じられないオーバーレイは検索品質・規制（同意）の文脈で別カテゴリとして扱われる（Google Search Central、EDPB）。

### 本資料で言えないこと（現時点の公開根拠では証明不可）

- 「Web全体で閉じられるモーダルが多数派」という出現率の断定。大規模計測または第三者統計が必要（付録A参照）。

---

## 用語と分類

### Dismissible Modal（閉じられるモーダルダイアログ）

次のいずれかを満たし、ユーザーが任意に退出できるもの。

- 視認できるクローズ要素（× 閉じるボタン/Close/Cancel/閉じる等）がある
- Escキーで閉じられる
- 背景クリック（backdrop click）で閉じられる

### Hard Gate / Wall（閉じられない・利用継続できないUI）

ログイン、登録、購読、同意などを満たさないと先に進めないUI。Closeボタンがない、またはCloseしても閲覧・利用ができない。典型例はログインウォール、ペイウォール、Cookie wallである。

---

## 規範的根拠

### WAI-ARIA APG：閉じる要素の強い推奨

W3C WAI-ARIA APGはモーダルダイアログを次のように定義している。

> A dialog is a window overlaid on either the primary window or another dialog window. Windows under a modal dialog are inert. That is, users cannot interact with content outside an active dialog window.
> — W3C WAI-ARIA APG, Dialog (Modal) Pattern

モーダルの標準構成要素は、コンテンツ領域、アクション（主要動作ボタン）、退出手段（× 閉じるボタン/Cancel/Escキー/背景クリック等）の3つである。

タブ順序について、APGは次を明記している。

> It is strongly recommended that the tab sequence of all dialogs include a visible element with role button that closes the dialog, such as a close icon or cancel button.
> — W3C WAI-ARIA APG, Dialog (Modal) Pattern

**技術的含意**
- 閉じる導線がないとフォーカストラップからの脱出不能・タスク完了不能を引き起こす
- closeがある方がフォーカス復帰・状態遷移の設計が容易

参照：https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/

### WCAG 2.2 Technique H102：Escキーとフォーカス制御

WCAG 2.2達成基準2.4.3（フォーカス順序）およびTechnique H102（`<dialog>`要素によるモーダル実装）では、Escキーによるクローズとフォーカスの移動・復帰を要件として挙げている。

**技術的含意**
- モーダル開放時にフォーカスがモーダル内に閉じ込められるため、視認できる閉じるボタンがなければスクリーンリーダー利用者が操作不能に陥る
- 「閉じる」は視覚的要素ではなくフォーカス順序・操作可能性に直結する要件

参照：
- https://www.w3.org/WAI/WCAG22/Techniques/html/H102
- https://waic.jp/translations/WCAG22/Techniques/html/H102

### MDN Web Docs：`<dialog>`要素

MDNではダイアログをEscキー、内部のCloseボタン、`close()`メソッドで閉じるものとして記述している。仕様・実装ガイド・ブラウザAPIのいずれも「モーダルは閉じられるもの」を前提としている。

参照：https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/dialog

### Escキーによるクローズ（必須）

モーダルダイアログはEscキーで閉じられなければならない。

- HTML `<dialog>` 要素はブラウザがEscキーによるクローズをデフォルトで処理する。`cancel`イベントで `preventDefault()` を呼ばない限り、追加実装は不要。
- `<dialog>` 以外でモーダルを実装する場合、`keydown`イベントでEscキーを検出し、クローズ処理とフォーカス復帰を実装する。
- Escキーによるクローズ後のフォーカスは、モーダルを開いたトリガー要素に復帰させる（WCAG 2.4.3）。
- 未保存データがある場合は、Escキーで即座に閉じるのではなく、破棄確認ダイアログを挟んでよい。ただしEscキー自体を無効化してはならない。

---

## Wallの定義根拠

### Google Search Central：intrusive interstitials and dialogs

Google Search Centralは、コンテンツ閲覧を妨げるオーバーレイUIを「intrusive interstitials and dialogs」として定義し、検索品質の評価要素として扱っている。対象はユーザー起点のモーダルダイアログではなく、サービス側が強制するアクセス制御である。

参照：https://developers.google.com/search/docs/appearance/avoid-intrusive-interstitials

### EDPB Guidelines 05/2020：Cookie wallと同意の自由意思

EDPB（GDPR同意ガイドライン 05/2020）は、サービス利用をCookie同意に条件付けるCookie wallについて、自由意思による同意の要件を満たさない場合があると明記している（Conditionality, paragraphs 39–41付近）。

**含意**
- 閉じられないオーバーレイはUX論点にとどまらず、同意の有効性（法規・規制）に直結する
- 通常のモーダルUIではなくアクセス制御（Wall）として別建てで評価すべきである

参照：https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_202005_consent_en.pdf

---

## 混同リスク

モーダルとWallを区別せず「閉じるボタン不要」と判断した場合のリスク。

- **アクセシビリティ**: 閉じるボタン省略 → フォーカストラップからの脱出不能 → WCAG 2.4.3抵触。Wallであってもアクセシビリティ担保は別途必要。
- **検索品質**: モーダル意図の実装がクローラーにWall（intrusive interstitial）と認識される可能性。
- **法規（GDPR等）**: 閉じる手段のないCookie同意UIは自由意思要件を満たさないとEDPBが指摘。

設計判断の前提として「このUIはユーザーが閉じられるのか、条件を満たさないと先に進めないのか」を明確にする。

---

## 判定フロー

開始：UIが画面上にオーバーレイ表示される

**分岐1**：ユーザーは操作を中断してこのUIを閉じることができるか？（× 閉じるボタン、Cancel、Escキー、背景クリック等）
- → Yes → **Modal Dialog**
- → No → 分岐2へ

**分岐2**：ユーザーの操作（ログイン・登録・同意・課金等）が完了しないと先に進めないか？
- → Yes → **Wall**（interstitial / paywall / login wall等）
- → No → **設計不備**（閉じる導線の実装漏れの可能性を検証）

### 設計不備の判定基準

以下のいずれかに該当する場合、閉じる導線の実装漏れとして是正する。

- 閉じる手段（× 閉じるボタン/Cancel/Escキー/背景クリック）がいずれも実装されていない
- `aria-modal="true"` または `role="dialog"` が付与されているがフォーカス復帰先が未定義
- Escキーイベントが未処理（`keydown`リスナーなし、かつ`<dialog>`要素未使用）
- フォーカストラップが実装されているが脱出手段がない

---

## 付録A：実態調査プロトコル

出現率の統計主張には大規模計測が必要である。以下に再現可能な計測設計を記載する。

### 母集団（サンプルフレーム）

- 例A：Tranco上位Nサイト（N=1,000 / 10,000）
- 例B：特定業界（EC/SaaS/メディア等）から層化抽出N
- 例C：自社競合・類似サービスの全集合（N=50〜200）

### 計測環境

Headless Browser（Playwright / Puppeteer）を使用し、JS実行後のDOMを観測する。静的HTML取得ではモーダルの後出し描画を取りこぼす。

### モーダル検出ルール

ページロード後（例：10秒）に以下で候補要素を抽出する。

- `[role="dialog"]` または `[aria-modal="true"]` を持つ要素
- `<dialog open>` 要素
- `position: fixed|absolute` かつviewportを大きく覆う要素で、z-indexが高く背景スクロール抑制を伴うもの

### Dismissible判定ルール

候補モーダルに対し以下を順に試行し、状態変化を判定する。

1. Close要素検出（DOM）：`aria-label`にclose/閉じる/キャンセル/cancel/×相当語彙、クラス名にclose/dismiss等
2. Escキーの送出で閉じるか
3. Backdrop clickで閉じるか

いずれか成功 → `dismissible = true`。すべて失敗かつ主要コンテンツ操作不能 → `hard_gate_like = true`（属性併記）。

### 監査

ランダム抽出5〜10%を人手レビューし精度推定。Inter-rater reliabilityも取得推奨。

### 出力指標

- Dismissible modal出現率
- Hard gate / wall出現率
- その他（モーダル無し / 非モーダル）
- 95% CI（信頼区間）
- 誤判定推定（監査補正）

---

## 付録B：データセットスキーマ

| カラム | 例 | 説明 |
|---|---|---|
| url | https://example.com | 対象URL |
| country / locale | JP | 検証ロケール |
| timestamp | 2026-01-26T10:00+09:00 | 計測時刻 |
| modal_detected | true/false | モーダル候補検出 |
| modal_type | dialog / interstitial / unknown | 概要分類 |
| dismissible | true/false/unknown | 閉じられるか |
| close_mechanisms | button,esc-key,backdrop | 成功した閉じ方 |
| close_button_corner | corner/none/unknown | コーナー配置 |
| hard_gate_like | true/false | 閉じられない壁か |
| gate_kind | login / signup / paywall / cookie / age / other | 壁の種別 |
| evidence_screenshot_before | path | スクリーンショット（前） |
| evidence_screenshot_after | path | スクリーンショット（後） |
| evidence_dom_snippet | string | DOM抜粋 |
| notes | text | 例外・失敗理由 |

---

## 付録C：関連研究

大規模計測の方法論（ブラックボックス計測、サイト横断自動検証）として以下が参考になる。いずれも「閉じるボタンの有無」を直接統計化したものではないが、手法は転用可能。

- [Keeping out the Masses: Understanding the Popularity and Implications of Paywalls on the Web（WWW 2020）](https://www.peteresnyder.com/static/papers/paywalls-www-2020.pdf)

- [A Large-Scale Measurement of Website Login Policies（USENIX Security 2023）](https://www.usenix.org/system/files/usenixsecurity23-al-roomi.pdf)

---

## 出典・参考情報
### Webアクセシビリティ・標準仕様

- [W3C WAI-ARIA APG - Dialog (Modal) Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/)

- [W3C WAI-ARIA APG - Modal Dialog Example](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/examples/dialog/)

- [WCAG 2.2 Technique H102 - Creating modal dialogs with the HTML dialog element](https://www.w3.org/WAI/WCAG22/Techniques/html/H102)

- [WCAG 2.2 Technique H102（日本語訳 - WAIC）](https://waic.jp/translations/WCAG22/Techniques/html/H102)

- [MDN Web Docs - `<dialog>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/dialog)

### Interstitial Wall

- [Google Search Central - Avoid intrusive interstitials and dialogs](https://developers.google.com/search/docs/appearance/avoid-intrusive-interstitials)

- [EDPB Guidelines 05/2020 on consent under GDPR](https://www.edpb.europa.eu/sites/default/files/files/file1/edpb_guidelines_202005_consent_en.pdf)

### 大規模計測研究（参考）

- [Keeping out the Masses: Understanding the Popularity and Implications of Paywalls on the Web（WWW 2020）](https://www.peteresnyder.com/static/papers/paywalls-www-2020.pdf)

- [A Large-Scale Measurement of Website Login Policies（USENIX Security 2023）](https://www.usenix.org/system/files/usenixsecurity23-al-roomi.pdf)
