# government プロファイル — 行政文書・公共告知

長期保存性（PDF/A）とアクセシビリティ（PDF/UA）、公文書としての真正性を監査する。
将来的に MoA（civil-signed-bulletin）検証と結節する足場でもある。

## 必須チェック

1. `verify_signatures` — 官職証明書・GPKI 系はチェーンが深いことがある。中間証明書
   不足で untrusted になったら、`check_revocation: "online"`（AIA によるチェーン補完が
   効く）をユーザーに提案
2. `verify_integrity` — verify v0.10.0+ なら SKILL.md「Phase 2.5」でオブジェクト単位まで落とす
3. **長期保存チェック（常時実施）**:
   - `validate_conformance` — PDF/A 適合（公文書の保存形式）
   - `detect_pades_level` — B-LTA が理想。未達なら推奨アクションに
4. **PDF/UA** — pdf-verify-mcp `validate_conformance`（`flavour: "pdfua-1"`。行政文書は
   アクセシビリティ義務の観点で必須寄り）。※ reader の `validate_tagged` は非推奨（verify へ移管済み）。
   未接続なら未実施と明記

## 判定の上書き

> **注**: この上書き条件は pdf-verify-mcp v0.7.0 の `evaluate_policy` ルールエンジンにコード化済み。通常は engine の verdict に反映されて返るため、ここを LLM が適用するのは evaluate_policy が使えないフォールバック時のみ。

- 閾値は 5 プロファイル中最も緩い（住民向け告知等、署名なし文書も現実に多い）。
  署名なしは reject にせず、「真正性の技術的裏付けなし — 入手経路（公式サイト等）で
  補強すること」を推奨アクションに書く

## 法令根拠（houki-egov）

- 公文書等の管理に関する法律（保存義務）
- 事案に応じ国民保護法・災害対策基本法（公共告知の場合）
- デジタル手続法（行政手続のオンライン化根拠）
