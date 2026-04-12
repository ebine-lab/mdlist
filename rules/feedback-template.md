---
title: "Feedback Document Format"
description: "バグ・エラーリポート作成用テンプレート / Bug and error report template"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-21T14:14+09:00"
lang: "ja"
---

# Feedback Document Format

**フィードバック作成時の共通フォーマット** - フィードバック文作成時においてこの要件を遵守すること

## 形式

- Markdownで作成・管理
- メール送信時はテキスト変換してコピーできる状態で提示（見出し構造は維持、リストも維持）
- **下記のFeedbackフォーマットは全て英文で作成**
- TEDの確認用に日本語訳もCanvasに表示（ファイルには含めない）

## 構造

### タイトルステータス（使用可能）

- FEATURE REQUEST
- BUG REPORT
- CRITICAL
- CRITICAL BUG REPORT
- FUTURE REQUEST
- UI IMPROVEMENT
- SECURITY ISSUE
- HOTFIX REQUEST
- INCIDENT REPORT
- DOCS IMPROVEMENT
- ACCESSIBILITY IMPROVEMENT
- PERFORMANCE IMPROVEMENT

```markdown
# [STATUS]: [件名]

**URGENT PRIORITY** - [緊急性の理由を1文で]

## SUMMARY

[詳細だがわかりやすく。必要な長さで記述]

## THE CORE PROBLEM（最重要）

[現状の問題と非効率なワークフロー]
[ユーザーが強いられている状況]
[なぜ不便で困っているのかを具体的かつ詳細に]

※フィードバックを出す動機そのもの。ここが弱いと提案全体が弱くなる

## CONCRETE EXAMPLE

[実際の会話例やシナリオ]

## THE COST

[ユーザー視点のコスト]
- 時間の浪費
- 生産性への影響
- 作業フローの中断
- 信頼性の低下

## PROPOSED SOLUTION

[解決策の概要]
[具体的な機能仕様]
[ユーザー体験の例]

## WHY EXISTING SOLUTIONS DO NOT WORK

[現在の代替手段とその限界]
[なぜ製品側での対応が必要か]

## REFERENCE: EXISTING SOLUTIONS

[事実に基づく調査結果]
[前例がない場合は省略可、または「前例がないからこそ先行すべき」と記述]

## EXPECTED OUTCOME

[導入前後の比較]

## WHY THIS MATTERS NOW

[他者がやっていないからこそやるべき理由]
[先行者利益の可能性]
[時間的緊急性]

## CLOSING

[実装優先度を上げるべき理由]
```

## 文章作成規約（厳守）

### 冗長性の厳格な排除

**絶対禁止：**
- 同じ構造・同じ論調・同じ論点を繰り返して字数を稼ぐ行為
- 一度成功したパターンを機械的に複製して長文化する行為
- 本質的に新しい情報がないのに、表現を変えて同じ内容を繰り返す行為
- セクション間で論点が重複する構成
- ユーザーの個人情報（名前や個人情報やディレクトリパスのユーザー名など）が記載されていないか精査し置き換える
- 回りくどい言い回しや無駄な修飾語を使用しない
- 冗長な文章を削除し、より簡潔で明確な表現を使用する
- チャットのような会話形式で文章を作成しない
- 明確な問題点を具体的に示し、解決策を提案する

**必須原則：**
- 各セクションは独立した新しい論点のみを扱う
- 既に述べた論点は一切繰り返さない
- 長さではなく正確な情報と密度を優先
- 同じ事例・比喩・論証を複数回使用しない
- 異なる問題は別の独立したフィードバックとして作成

**理由：**
冗長なフィードバックはフィードバック先に真剣に読まれない。適度に簡潔だが正確で密度の高い文章が最も効果的。

---

## 送信先

| 方法 | 用途 |
|------|------|
| feedback@anthropic.com | 詳細な機能要望・バグレポート（feedback+noreply@anthropic.com は自動返信のアドレスの為送信に使用しない） |
| thumbs down button | 簡易フィードバック |
| GitHub Issues | https://github.com/anthropics/claude-code/issues（Claude Code関連） |
