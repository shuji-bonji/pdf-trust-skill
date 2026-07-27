# medical プロファイル — 診療情報提供書・検査報告書

患者安全に直結するため、5 プロファイル中で最も保守的に判定する。
改ざん検知とアクセシビリティ（PDF/UA）を重視する。

## 必須チェック

1. **署名必須** — 未署名の診療文書は human_review_required
2. `verify_signatures` — trust_anchors（医療機関・HPKI 系 CA）推奨
3. `verify_integrity` — 署名後変更は原則すべて注記（検査値の書き換えは一文字でも重大）。
   **verify v0.10.0+ なら `objectChangesAfterLastSignature` でオブジェクト番号・型・役割まで、
   reader v0.10.0+ の `locate_objects` を併用すればページと矩形まで書く**（SKILL.md「Phase 2.5」）。
   ただし **`basis: page-content-stream` の矩形はページ全体**であって「書き換わった検査値の位置」ではない —
   本文の一文字を特定できたかのように書かない
4. **PDF/UA** — pdf-verify-mcp `validate_conformance`（`flavour: "pdfua-1"`）で検証する。
   verify の `identify_conformance` は宣言の識別まで。
   ※ reader の `validate_tagged` は非推奨（verify へ移管済み）。verify 未接続なら
   「PDF/UA 検証: 未実施」と明記

## 判定の上書き（最重要）

> **注**: この上書き条件は pdf-verify-mcp v0.7.0 の `evaluate_policy` ルールエンジンにコード化済み。通常は engine の verdict に反映されて返るため、ここを LLM が適用するのは evaluate_policy が使えないフォールバック時のみ。

- **use_with_caution は human_review_required に格上げする** — 「注意して使う」という
  中間状態を患者情報で許容しない。疑わしきは人手レビューへ
- trust: not_evaluated の場合も human_review_required（身元未評価の診療文書を
  「概ね問題なし」と報告しない）

## 法令根拠

- 必要に応じ houki-egov で医師法（文書作成義務）・個人情報保護法関連を参照
- 本プロファイルの法令照合は補助的（技術検証が主）。医療情報システムの安全管理
  ガイドライン等の告示類は houki 系で取得できないことがある — その場合は取得不能と明記
