---
name: pdf-trust
description: PDF の信頼性監査オーケストレーション。受け取った PDF（契約書・請求書・診療文書・申告書・行政文書など）が「本物か・信用してよいか・改ざんされていないか」を、pdf-verify-mcp を軸に pdf-reader-mcp / pdf-spec-mcp / houki 系 MCP を編成して判定し、推奨判定付きの Trust Report を返す。ユーザーが「この PDF は信用できる？」「署名を確認して」「受入監査して」「改ざんされてない？」「電子署名の有効性」「長期保存（電帳法・PDF/A・LTV）できるか」「取引先から届いた PDF のチェック」などに言及したら、単発のツール呼び出しで済ませず必ずこの Skill を使う。複数 PDF の一括監査にも使う。
---

# pdf-trust — ドメイン別 PDF 信頼性監査

PDF family の trust 層（specs/04-pdf-trust-mcp.md の Skill 実装）。自前の検証ロジックは持たず、
検証は必ず MCP ツールの結果を根拠にする。**構造解析からの推測で真正性を代替しない**こと —
「本物か」に答えられるのは暗号学的検証だけであり、それは pdf-verify-mcp の仕事。

中核原則（family 共通）:

1. 内容の真偽は判定しない — 判定するのは真正性（原本性・完全性）のみ
2. 検証結果は技術的事実として返し、解釈・最終判断は利用者に委ねる
3. 判定の根拠（どのツールの何の結果か）を必ず明示する

## 前提 MCP

| MCP | 必須/任意 | 役割 |
|---|---|---|
| pdf-verify-mcp | **必須** | 署名検証・改ざん検知・PAdES レベル・PDF/A 検証 |
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

### Phase 1 — 基礎検証（全ファイル一括）

全対象ファイルに対して実行する:

1. `pdf-verify-mcp: verify_signatures` — trust_anchors があれば渡す。`check_revocation` は
   プロファイル指定に従う（既定 `embedded`。`online` は外部への HTTP アクセスを伴うため、
   プロファイルが要求する場合もユーザーに一言断ってから）
2. `pdf-verify-mcp: verify_integrity` — 増分更新・署名後変更・DocMDP 違反

複数ファイルの一括監査では、まずこの 2 つだけを全件に回して問題のあるファイルを特定し、
Phase 2 以降の深掘りは問題のあるファイルに絞る（全件深掘りは時間と文脈の無駄遣い）。

### Phase 2 — 結果の解釈

解釈を誤ると監査全体が誤導になるため、次の表に従う:

| 観測 | 意味 | 対応 |
|---|---|---|
| verdict: valid + trust: not_evaluated | 暗号学的完全性のみ確認。**署名者の身元は未評価** | レポートに必ずその旨を明記。trust_anchors の提供を促す |
| verdict: valid + trust: trusted | 完全性 + 署名者身元とも確認 | 最良の状態 |
| verdict: valid + trust: untrusted | 署名は有効だがチェーンがアンカーに到達しない | 中間 CA 不足か、アンカー相違。detail を確認 |
| verdict: invalid | ダイジェスト不一致・署名検証失敗・**失効** | 原則 reject。notes で原因を特定 |
| verdict: indeterminate | 未対応形式 or 検証未完了 | 下記の切り分けへ |
| 署名なし | 真正性の技術的裏付けなし | プロファイルが署名必須なら不合格 |
| revocation: unknown | 失効情報が確認できなかった | 「失効していない」とは言えない。online 再試行を検討 |

indeterminate の切り分け: cms の error / notes を読む → SubFilter 未対応
（adbe.pkcs7.sha1 等）か、CMS パース失敗か、暗号化で復号できないか。pdf-reader-mcp が
あれば `inspect_signatures` で構造側から確認する。

知っておくべき実挙動（実測に基づく）:

- 自己署名のリーフ証明書そのものを trust_anchors に渡すと、チェーンエンジンは
  「not a CA certificate」で untrusted にする（直接信頼リーフは非対応）。CA 証明書を渡すこと
- 署名済み PDF を後から暗号化・再保存すると署名は壊れる（INVALID は改ざんとは限らない —
  再保存の痕跡かもしれない。verify_integrity の増分更新情報と突き合わせる）
- 増分更新は合法（DSS 追加・連署など）。「署名後に変更あり」= 改ざんではなく、
  DocMDP 違反や digest 不一致と組み合わせて判断する
- linearized PDF はリビジョン数が +1 に見えることがある（ヒューリスティックの限界）

### Phase 3 — プロファイル別チェック

references/<profile>.md の必須チェックを実行する。長期保存が目的なら追加で:

1. `pdf-verify-mcp: detect_pades_level` — B-LT / B-LTA でなければ、証明書失効後に
   検証不能になるリスクを警告（LTV データの実在検証込みなので「宣言だけの B-LT」も検出される）
2. `pdf-verify-mcp: validate_conformance` — PDF/A 適合。engine は auto のまま
   （veraPDF があれば権威的結果、なければ内蔵サブセット。どちらだったかをレポートに書く —
   native の「違反なし」は認証ではない）

### Phase 4 — 法令根拠（プロファイル指定時）

プロファイルが法令照合を指定する場合、houki 系 MCP で根拠条文を取得する。
houki 系 MCP のサーバ指示（原文引用・出典 URL 明記）に従うこと。取得できなければ
「法令根拠: 未取得」と明記（勝手に条文番号を記憶から書かない）。

### Phase 5 — Trust Report 生成

以下のテンプレートで報告する。複数ファイルの場合は冒頭にサマリ表を置き、
問題のあるファイルのみ個票を付ける。

```markdown
# PDF 信頼性監査レポート

- 対象: <ファイル名（複数なら件数）> / プロファイル: <profile>（<選定根拠>）
- 実施日時・使用ツール: <MCP 名とバージョン情報が得られれば>

## 判定: <recommendation>

<1〜3 行の要約>

## 根拠

| 検査 | 結果 | 根拠ツール |
|---|---|---|
| 署名の暗号学的有効性 | ... | verify_signatures |
| 署名者の身元（信頼チェーン） | ... | verify_signatures (trust) |
| 失効確認 | ... | verify_signatures (revocation) |
| 署名後の変更 | ... | verify_integrity |
| <プロファイル別項目> | ... | ... |

## 警告・制限事項

- <trust_anchors 未指定なら必ず: 「valid は暗号学的完全性のみの主張であり署名者の身元は未評価」>
- <未実施項目（ツール未接続・取得失敗）>

## 推奨アクション

- <trust_anchors の入手、online 失効確認、LTV 化、人手レビュー対象箇所など>
```

### recommendation の判定

| 判定 | 条件 |
|---|---|
| trust_and_use | valid + trusted + 失効なし（good / 失効情報が embedded で確認）+ プロファイル必須チェック全通過 |
| use_with_caution | valid だが trust: not_evaluated / untrusted、または revocation: unknown、または任意チェックに軽微な指摘 |
| human_review_required | indeterminate、DocMDP 違反疑い、プロファイル必須チェックの不合格、署名なし（署名必須プロファイル） |
| reject | invalid（digest 不一致・署名検証失敗・失効確認済み） |

プロファイルの references に、この既定をドメイン事情で上書きする条件が書いてある場合は
そちらを優先する（例: medical は use_with_caution を human_review_required に格上げ）。

## やらないこと

- 内容の真偽・法的有効性の最終判断（技術的事実と法令根拠の提示まで）
- 構造解析（inspect_signatures 等）だけを根拠にした「有効/無効」の判断
- 記憶からの条文引用（必ず houki 系 MCP で原文を取得）
