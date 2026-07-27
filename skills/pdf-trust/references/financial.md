# financial プロファイル — 請求書・決算書・申告書・領収書

電子帳簿保存法（電帳法）の保存要件と税務調査への耐性の観点で監査する。
このドメインは「今有効か」より「保存期間（7〜10年）の終わりまで検証可能か」が重要。

## 必須チェック

1. `verify_signatures` — 請求書等は署名なしも多い（その場合は「真正性の技術的裏付けなし。
   電帳法の真実性確保は別手段（タイムスタンプ・訂正削除防止システム等）に依存」と明記）。
   署名があるのに invalid なら reject
2. `verify_integrity` — 受領後の改変痕跡の確認（verify v0.10.0+ なら SKILL.md「Phase 2.5」で
   オブジェクト単位まで落とす。金額欄のウィジェットが署名後に書き換わっていないかは特に見る）
3. **長期保存チェック（このプロファイルでは常時実施）**:
   - `detect_pades_level` — B-LT / B-LTA でなければ LTV 化を推奨アクションに
   - `validate_conformance` — PDF/A 適合（保存形式として）

## 推奨チェック

- レガシー署名への注意: 古い請求書（AWS 等）は MD5 / adbe.pkcs7.detached のことがある。
  verify は weak digest フラグを notes に出すので、それを「完全性保証は限定的」として転記する

## 法令根拠（houki-nta / tax-law）

- houki-nta `nta_search_qa` — 「電子取引 保存」で電帳法一問一答（保存要件・タイムスタンプ要件）
- houki-nta `nta_search_tax_answer` — 個別論点（スキャナ保存、検索要件など）
- 税法上の位置づけが必要なら tax-law で法令原文を取得

## 判定の上書き

> **注**: この上書き条件は pdf-verify-mcp v0.7.0 の `evaluate_policy` ルールエンジンにコード化済み。通常は engine の verdict に反映されて返るため、ここを LLM が適用するのは evaluate_policy が使えないフォールバック時のみ。

- 署名なし + 電帳法目的 → 判定は use_with_caution を上限にし、「真実性確保措置の確認」を
  推奨アクションに必ず入れる
