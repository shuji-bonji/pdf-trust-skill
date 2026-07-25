---
name: pdf-trust
description: PDF の信頼性監査オーケストレーション。受け取った PDF（契約書・請求書・診療文書・申告書・行政文書など）が「本物か・信用してよいか・改ざんされていないか」を、pdf-verify-mcp を軸に pdf-reader-mcp / pdf-spec-mcp / houki 系 MCP を編成して監査し、推奨判定付きの Trust Report を返す。4 値判定は pdf-verify-mcp の evaluate_policy（決定論的ルールエンジン）が下し、Skill は発火ルールの解説・推奨アクション・法令根拠を担う。ユーザーが「この PDF は信用できる？」「署名を確認して」「受入監査して」「改ざんされてない？」「電子署名の有効性」「長期保存（電帳法・PDF/A・LTV）できるか」「取引先から届いた PDF のチェック」などに言及したら、単発のツール呼び出しで済ませず必ずこの Skill を使う。複数 PDF の一括監査にも使う。
---

# pdf-trust — ドメイン別 PDF 信頼性監査

PDF family の trust 層を担う Skill。自前の検証ロジックは持たず、
検証は必ず MCP ツールの結果を根拠にする。**構造解析からの推測で真正性を代替しない**こと —
「本物か」に答えられるのは暗号学的検証だけであり、それは pdf-verify-mcp の仕事。

中核原則（family 共通）:

1. 内容の真偽は判定しない — 判定するのは真正性（原本性・完全性）のみ
2. 検証結果は技術的事実として返し、解釈・最終判断は利用者に委ねる
3. 判定の根拠（どのツールの何の結果か）を必ず明示する
4. **ジャッジはコード、ナラティブは LLM** — 4 値判定は `evaluate_policy`
   （pdf-verify-mcp v0.7.0+ の決定論的ルールエンジン）が下す。この Skill（LLM）の
   仕事は firedRules の解説・推奨アクションの文章化・法令根拠の引用であり、
   判定の上書きではない

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-verify-mcp（**v0.7.0+ 推奨**） | **必須** | `evaluate_policy` による 4 値判定・署名検証・改ざん検知・PAdES レベル・PDF/A 検証。v0.7.0 未満は evaluate_policy が無くフォールバック手動判定に縮退 |
| pdf-reader-mcp | 任意 | 署名フィールド構造・PDF/UA タグ検証・メタデータ |
| pdf-spec-mcp | 任意 | 逸脱時の ISO 32000 根拠引用 |
| houki-egov / houki-nta / tax-law / labor-law | 任意 | 法令根拠（プロファイルが指定） |

pdf-verify-mcp が未接続なら監査は成立しない。その旨を伝え、`npx @shuji-bonji/pdf-verify-mcp`
の接続を案内して停止する。任意 MCP が無い場合は縮退動作し、レポートの該当項目に
「未実施（ツール未接続）」と明記する — 黙って項目を落とすと「チェック済みで問題なし」と誤読される。

## 手順

### Phase 0 — 目的とプロファイルの特定

ファイルパス（絶対パス）と利用目的を確認し、プロファイルを選ぶ。目的が不明瞭でも
文書種別から推定できることが多い。推定した場合はレポートに推定根拠を書く。

| プロファイル | 想定文書 | 参照 |
|---|---|---|
| contract | 契約書・NDA・発注書 | [references/contract.md](references/contract.md) |
| financial | 請求書・決算書・申告書・領収書 | [references/financial.md](references/financial.md) |
| legal | 訴訟資料・法務文書全般 | [references/legal.md](references/legal.md) |
| medical | 診療情報提供書・検査報告書 | [references/medical.md](references/medical.md) |
| government | 行政文書・公共告知 | [references/government.md](references/government.md) |
| general | 上記以外 | プロファイル追加チェックなし |

**該当プロファイルの references ファイルをこの時点で読む**（必須チェック・閾値・法令照合先が
書いてある）。あわせて次も確認する:

- trust_anchors（信頼する CA 証明書）を持っているか → あれば Phase 1 で渡す
- 長期保存が目的に含まれるか（「保存」「アーカイブ」「電帳法」「10年」等）→ Phase 3 の長期保存チェックを追加
- 暗号化 PDF でパスワードを知っているか → `password` パラメータで渡す

### Phase 1 — 基礎検証と判定（全ファイル一括）

全対象ファイルに対して `pdf-verify-mcp: evaluate_policy` を実行する（v0.7.0+）。
`profile` に Phase 0 で選んだプロファイルを渡し、trust_anchors・password があれば渡す。
`check_revocation` はプロファイル指定に従う（既定 `embedded`。`online` は外部への
HTTP アクセスを伴うため、プロファイルが要求する場合もユーザーに一言断ってから）。

**最終判定（recommendation）は evaluate_policy の `verdict` をそのまま使う。**
このツールは verify_signatures / verify_integrity / detect_pades_level（長期保存
プロファイルでは validate_conformance も）を内部で実行し、固定ルール表で決定論的に
4 値判定する。同じファイル・同じプロファイルなら常に同じ判定になる。
**LLM（この Skill を実行しているあなた）が verdict を上書きすることは禁止** —
firedRules / advisories を「なぜこの判定になったか」の解説材料として使うこと。
文書の本文内容（契約金額・重要度など）を判定材料にしてはならない。

接続先の pdf-verify-mcp が古く evaluate_policy が無い場合のみ、旧手順
（verify_signatures + verify_integrity を個別実行し、後述のフォールバック判定表で
判定）に縮退し、レポートに「判定: 手動判定（evaluate_policy 未使用）」と明記する。

複数ファイルの一括監査では、まず evaluate_policy だけを全件に回して問題のある
ファイル（reject / human_review_required）を特定し、Phase 2 以降の深掘りは
問題のあるファイルに絞る（全件深掘りは時間と文脈の無駄遣い）。

### Phase 2 — 結果の解釈

判定は Phase 1 の evaluate_policy が済ませている。この Phase の仕事は
**firedRules の各ルールを人間に説明できるようにする**こと。深掘りが必要なときは
`verify_signatures` / `verify_integrity` を個別に呼んで詳細（cms.error・notes・
certificatePath 等）を取得する。解釈の背景知識として次の表を使う:

| 観測（firedRules / facts） | 意味 | 解説・推奨アクションに書くこと |
|---|---|---|
| POL-CAUTION-TRUST-NOT-EVALUATED | 暗号学的完全性のみ確認。**署名者の身元は未評価** | 必ずその旨を明記。trust_anchors の提供を促す |
| （何も発火せず trust_and_use） | 完全性 + 署名者身元 + 失効とも確認 | 最良の状態 |
| POL-CAUTION-TRUST-UNTRUSTED | 署名は有効だがチェーンがアンカーに到達しない | 中間 CA 不足か、アンカー相違。trust.detail を確認 |
| POL-REJECT-INVALID / POL-REJECT-REVOKED | ダイジェスト不一致・署名検証失敗・失効 | 原因を verify_signatures の notes で特定して解説 |
| POL-REVIEW-INDETERMINATE | 未対応形式 or 検証未完了 | 下記の切り分けへ |
| POL-REVIEW-UNSIGNED-REQUIRED / POL-CAUTION-UNSIGNED | 真正性の技術的裏付けなし | 入手経路など他の補強手段を提案 |
| POL-CAUTION-REVOCATION-UNKNOWN | 失効情報が確認できなかった | 「失効していない」とは言えない。online 再試行を検討 |

indeterminate の切り分け: cms の error / notes を読む → SubFilter 未対応
（adbe.pkcs7.sha1 等）か、CMS パース失敗か、暗号化で復号できないか。pdf-reader-mcp が
あれば `inspect_signatures` で構造側から確認する。

知っておくべき実挙動（実測に基づく）:

- 自己署名のリーフ証明書そのものを trust_anchors に渡すと、チェーンエンジンは
  「not a CA certificate」で untrusted にする（直接信頼リーフは非対応）。CA 証明書を渡すこと
- untrusted のメッセージは原因を区別できる: 「not a CA certificate」= 渡したアンカーが
  CA でない、「No valid certificate paths found」= アンカーが署名者チェーンと無関係。
  後者は正しい CA 証明書の入手を促す
- indeterminate + 「CMS payload is not valid BER/DER」+ Embedded certificates: 0 は、
  改ざんではなく署名生成ツール側の不備（証明書の埋め込み漏れ等）のサイン。
  改ざんと断定せず「検証不能な署名」として human_review_required に送る
- 署名済み PDF を後から暗号化・再保存すると署名は壊れる（INVALID は改ざんとは限らない —
  再保存の痕跡かもしれない。verify_integrity の増分更新情報と突き合わせる）
- 増分更新は合法（DSS 追加・連署など）。「署名後に変更あり」= 改ざんではなく、
  DocMDP 違反や digest 不一致と組み合わせて判断する
- linearized PDF はリビジョン数が +1 に見えることがある（ヒューリスティックの限界）

### Phase 3 — プロファイル別チェック

references/<profile>.md の必須チェックのうち、evaluate_policy が扱わないもの
（署名時刻の突き合わせ、PDF/UA 検証、増分更新履歴の詳述など）を実行する。
PAdES レベルと PDF/A 適合は evaluate_policy が facts / advisories に出している
（financial / government プロファイルでは PDF/A 検証込み）。深掘りが必要なら:

1. `pdf-verify-mcp: detect_pades_level` — 構造が B-LT / B-LTA に一致しなければ、証明書失効後に
   検証不能になるリスクを警告（LTV データの実在検証込みなので「宣言だけの B-LT」も検出される）。
   **返り値の `normativeBasis` は常に `"T3"`** — レベルは観測であって適合判定ではないので、
   レポートには「構造が一致する」と書く（上記「報告するときの言い方」）
2. `pdf-verify-mcp: validate_conformance` — PDF/A 適合。engine は auto のまま
   （veraPDF があれば権威的結果、なければ内蔵サブセット。どちらだったかをレポートに書く —
   native の「違反なし」は認証ではない）

### Phase 4 — 法令根拠（プロファイル指定時）

プロファイルが法令照合を指定する場合、houki 系 MCP で根拠条文を取得する。
houki 系 MCP のサーバ指示（原文引用・出典 URL 明記）に従うこと。取得できなければ
「法令根拠: 未取得」と明記（勝手に条文番号を記憶から書かない）。

### 報告するときの言い方 — 規範の 3 層（T1 / T2 / T3）

**同じレポートの中で、項目ごとに言える強さが違う。** 正典は `specs/09-family-scope.md §2`。
verify の出力は `normativeBasis` フィールドと注記でこれを示してくるので、**それに従って語彙を選ぶ**。

| 層 | 対象 | 書いてよい | 書いてはいけない |
|---|---|---|---|
| **T1** | ISO 32000（署名構造・増分更新）・ISO 14289（PDF/UA） | 条文を引用して断定できる | — |
| **T2** | **PDF/A（ISO 19005）** | 「**veraPDF が COMPLIANT と判定**」 | 「ISO 19005-3 準拠」 |
| **T3** | **PAdES（ETSI EN 319 142）** | 「**構造が B-LT に一致する**」「LTV データが署名者証明書を覆っている」 | 「**PAdES B-LT に適合**」 |

**T3 が特殊なのは、判定者すら居ないこと。** T2 には veraPDF という第三者検証器があるので
「誰が判定したか」を名指しできるが、**PAdES には規範も検証器も無く、構造を観測しているのは
family 自身**である。だから「veraPDF はこう言った」のような逃げ方ができない —
**「これは観測であって適合判定ではない」と述べるしかない**。

`detect_pades_level` は各レポートに `normativeBasis: "T3"` を返し、markdown では
レベルより**前**に注記を置く。**その注記をレポートへ運ぶこと**（数字だけ抜き出さない）。

> **なぜここまで言うか**: 受入監査のレポートは、相手に「この文書は PAdES B-LT 準拠です」と
> 伝えるために使われる。**適合の刻印は、押した者が責任を負う。** ETSI の原文を読んでいないのに
> 適合と書けば、それは実測していないことを実測したかのように述べたことになる。
> `pdf-writer-mcp` が「宣言は書けるが適合は作れない」と言うのと、向きが逆の同じ話。

### Phase 5 — Trust Report 生成

以下のテンプレートで報告する。複数ファイルの場合は冒頭にサマリ表を置き、
問題のあるファイルのみ個票を付ける。

```markdown
# PDF 信頼性監査レポート

- 対象: <ファイル名（複数なら件数）> / プロファイル: <profile>（<選定根拠>）
- 実施日時・使用ツール: <MCP 名とバージョン情報が得られれば>

## 判定: <evaluate_policy の verdict をそのまま>

<1〜3 行の要約。発火ルールがなぜ発火したかの平易な説明>

## 根拠

| 検査 | 結果 | 根拠ツール | 規範根拠 |
|---|---|---|---|
| 総合判定（発火ルール: <POL-... の列挙>） | ... | evaluate_policy | — |
| 署名の暗号学的有効性 | ... | evaluate_policy (facts) / verify_signatures | — |
| 署名者の身元（信頼チェーン） | ... | verify_signatures (trust) | — |
| 失効確認 | ... | verify_signatures (revocation) | — |
| 署名後の変更 | ... | verify_integrity | T1（ISO 32000 §14.4 / §12.8） |
| PAdES レベル | **構造が <B-T 等> に一致** | detect_pades_level | **T3（観測。ETSI 原文は照合していない）** |
| PDF/A 適合 | **veraPDF が <COMPLIANT 等> と判定** | validate_conformance | **T2（判定者は veraPDF）** |
| <プロファイル別項目> | ... | ... | ... |

## 警告・制限事項

- <trust_anchors 未指定なら必ず: 「valid は暗号学的完全性のみの主張であり署名者の身元は未評価」>
- <PAdES に言及したなら必ず: 「PAdES レベルは構造からの観測であり、ETSI EN 319 142 の原文照合ではない」>
- <PDF/A に言及したなら必ず: 「判定者は veraPDF。ISO 19005 の条文は確認していない」>
- <未実施項目（ツール未接続・取得失敗）>

## 推奨アクション

- <trust_anchors の入手、online 失効確認、LTV 化、人手レビュー対象箇所など>
```

### recommendation の判定

**判定は evaluate_policy の verdict をそのまま使う。LLM による上書き禁止。**
判定表・プロファイル上書き（medical の格上げ等）はすべて verify 側の
ルールエンジン（`services/policy-engine.ts`）にコード化されており、この Skill 側で
再現・変更しない。firedRules に無い理由で判定を変えたくなったら、それは
ルールエンジンの改善要望として pdf-verify-mcp に issue を立てる。

**フォールバック判定表**（evaluate_policy が無い旧バージョンに接続した場合のみ）:

| 判定 | 条件 |
|---|---|
| trust_and_use | valid + trusted + 失効なし（good / 失効情報が embedded で確認）+ プロファイル必須チェック全通過 |
| use_with_caution | valid だが trust: not_evaluated / untrusted、または revocation: unknown、または任意チェックに軽微な指摘 |
| human_review_required | indeterminate、DocMDP 違反疑い、プロファイル必須チェックの不合格、署名なし（署名必須プロファイル） |
| reject | invalid（digest 不一致・署名検証失敗・失効確認済み） |

フォールバック時もプロファイルの references の上書き条件（例: medical は
use_with_caution を human_review_required に格上げ）を優先し、レポートに
「判定: 手動判定（evaluate_policy 未使用）」と明記する。

## やらないこと

- 内容の真偽・法的有効性の最終判断（技術的事実と法令根拠の提示まで）
- **evaluate_policy の verdict の上書き**（本文内容・重要度・文脈を判定材料にしない）
- 構造解析（inspect_signatures 等）だけを根拠にした「有効/無効」の判断
- 記憶からの条文引用（必ず houki 系 MCP で原文を取得）
