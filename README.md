# pdf-trust-skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Skill](https://img.shields.io/badge/Claude-Skill-D97757?logo=anthropic&logoColor=white)](https://github.com/shuji-bonji/pdf-trust-skill)

🌐 [日本語版 (README.ja.md)](./README.ja.md)

A **Claude Skill** that orchestrates the [PDF family](https://github.com/shuji-bonji#-pdf-family) MCP servers to audit the **trustworthiness of a PDF** — is it authentic, has it been tampered with, can it be relied upon? It runs cryptographic signature verification, tamper detection, PAdES/PDF-A checks and (optionally) Japanese-law cross-referencing through domain-specific profiles, and returns a **Trust Report** with an explicit recommendation.

> **Scope note**: this skill judges **authenticity, not truth**. It verifies that a document is intact and cryptographically genuine; whether its *content* is correct or legally effective is left to the user (and, where applicable, to qualified professionals).

## What it provides

This repository is **not an MCP server — it is a Skill**: a Markdown playbook that tells Claude *how to combine* the PDF family MCPs when a user asks "can I trust this PDF?".

```mermaid
graph TB
  subgraph skill["Skill layer (this repository)"]
    direction TB
    S1["Domain profiles<br/>(contract / financial / legal / medical / government)"]
    S2["Verification orchestration<br/>(which tool, in which order)"]
    S3["Result interpretation<br/>(valid ≠ trusted, indeterminate triage)"]
    S4["Trust Report template<br/>+ unified recommendation vocabulary"]
  end

  subgraph mcp["MCP layer (separate repositories)"]
    direction TB
    M1["@shuji-bonji/pdf-verify-mcp<br/>(required)"]
    M2["@shuji-bonji/pdf-reader-mcp<br/>(optional)"]
    M3["@shuji-bonji/pdf-spec-mcp<br/>(optional)"]
    M4["houki family MCPs<br/>(optional, Japanese law)"]
  end

  skill -->|orchestrate| mcp

  classDef skill fill:#fff3cd,stroke:#ffc107,color:#333
  classDef mcp fill:#cce5ff,stroke:#0066cc,color:#333
  class S1,S2,S3,S4 skill
  class M1,M2,M3,M4 mcp
```

### Why a Skill and not an MCP server?

The PDF family follows a simple rule: **deterministic computation (cryptography, parsing) belongs in MCP servers; procedures, judgment and knowledge belong in Skills**. Trust auditing is pure orchestration — profiles, ordering, interpretation — so shipping it as a server would only add a process to maintain. `pdf-verify-mcp` does the actual cryptographic work.

## What an audit looks like

1. **Profile selection** — infer the document domain (contract, invoice, medical record, …) and load the matching profile from `references/`
2. **Base verification** (all files) — `verify_signatures` + `verify_integrity`
3. **Interpretation** — apply the built-in interpretation table (e.g. *valid with trust `not_evaluated` means cryptographic integrity only, signer identity unverified*)
4. **Profile checks** — e.g. signing-time cross-check for contracts, PDF/A + PAdES LTV for long-term storage, PDF/UA for medical/government
5. **Legal grounding** (optional) — fetch the actual statute text via houki MCPs, with source URLs
6. **Trust Report** — findings table with per-check tool provenance, warnings, and one of four recommendations:

| Recommendation | Meaning |
|---|---|
| `trust_and_use` | valid + trusted chain + no revocation + all required profile checks passed |
| `use_with_caution` | cryptographically valid but identity unevaluated / revocation unknown |
| `human_review_required` | indeterminate results, DocMDP violations, failed required checks |
| `reject` | invalid — digest mismatch, failed signature, or confirmed revocation |

## Domain profiles

| Profile | Typical documents | Emphasis |
|---|---|---|
| [contract](references/contract.md) | Contracts, NDAs, purchase orders | Signer identity, signing-time consistency, PAdES B-T+ |
| [financial](references/financial.md) | Invoices, financial statements, tax filings | Long-term verifiability (LTV, PDF/A), Japanese e-bookkeeping law |
| [legal](references/legal.md) | Litigation materials, legal documents | Evidential strength, full revision history |
| [medical](references/medical.md) | Referral letters, test reports | Most conservative judgments, PDF/UA |
| [government](references/government.md) | Administrative documents, public notices | PDF/A archival, PDF/UA, deep cert chains (GPKI) |
| general | Everything else | Base verification only |

## Prerequisite MCPs

| MCP | Required | Role |
|---|---|---|
| [@shuji-bonji/pdf-verify-mcp](https://www.npmjs.com/package/@shuji-bonji/pdf-verify-mcp) | **Yes** | Signature verification, tamper detection, PAdES level, PDF/A validation |
| [@shuji-bonji/pdf-reader-mcp](https://www.npmjs.com/package/@shuji-bonji/pdf-reader-mcp) | No | Signature field structure, PDF/UA tag validation, metadata |
| [@shuji-bonji/pdf-spec-mcp](https://www.npmjs.com/package/@shuji-bonji/pdf-spec-mcp) | No | ISO 32000 citations for deviations |
| [houki-egov-mcp](https://github.com/shuji-bonji/houki-egov-mcp) / [houki-nta-mcp](https://github.com/shuji-bonji/houki-nta-mcp) etc. | No | Japanese statute / tax-ruling grounding |

Without the optional MCPs the skill degrades gracefully: skipped checks are reported as *not performed (tool not connected)* rather than silently dropped.

```jsonc
// claude_desktop_config.json — minimal setup
{
  "mcpServers": {
    "pdf-verify-mcp": {
      "command": "npx",
      "args": ["-y", "@shuji-bonji/pdf-verify-mcp"]
    }
  }
}
```

## Installation

### A. Manual clone (Claude Code)

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/shuji-bonji/pdf-trust-skill pdf-trust
```

### B. Cowork (Claude Desktop)

Package the skill directory as a `.skill` (zip) file and add it via **Settings → Capabilities → Skills**, or ask Claude in a Cowork session to install it from this repository.

### C. Marketplace (planned)

Registration in [shuji-bonji/claude-plugins](https://github.com/shuji-bonji/claude-plugins) is planned but **not yet available**. Once registered:

```bash
/plugin marketplace add shuji-bonji/claude-plugins
/plugin install pdf-trust@shuji-bonji
```

## Directory layout

```
pdf-trust-skill/
├── SKILL.md                 # Main playbook loaded by Claude
├── references/              # Domain profiles (loaded on demand)
│   ├── contract.md
│   ├── financial.md
│   ├── legal.md
│   ├── medical.md
│   └── government.md
├── README.md                # This file
├── README.ja.md             # Japanese version
└── LICENSE                  # MIT
```

> The skill instructions (`SKILL.md`, `references/`) are currently written in **Japanese**. Claude follows them regardless of the conversation language, and the audited PDFs can be in any language — but the built-in legal grounding targets **Japanese law** (Electronic Signature Act, e-bookkeeping preservation rules).

## Design notes

- Verdict interpretation is grounded in the actual observed behavior of `pdf-verify-mcp` (e.g. a self-signed leaf passed as a trust anchor yields *untrusted: not a CA certificate*; re-encrypting a signed PDF legitimately breaks its signature).
- The skill never lets Claude infer validity from structure alone — every authenticity claim must cite an MCP verification result.
- Statute citations must come from houki MCPs (primary sources with URLs), never from model memory.

## Evaluation

The initial release was benchmarked against a no-skill baseline on three scenarios (contract acceptance audit, DocMDP tamper detection, long-term storage audit): **15/15 assertions passed with the skill vs 11/15 without**. The gap came from unified recommendation vocabulary, domain procedures, and primary-source legal citations.

## License

MIT. This skill produces **technical findings, not legal advice**. Final decisions on legal effectiveness, evidential weight, or regulatory compliance belong to the user and, where required, qualified professionals.

## Related projects

| Project | Role |
|---|---|
| [pdf-verify-mcp](https://github.com/shuji-bonji/pdf-verify-mcp) | Authenticity layer — cryptographic verification (required) |
| [pdf-reader-mcp](https://github.com/shuji-bonji/pdf-reader-mcp) | Structure layer — PDF internals inspection |
| [pdf-spec-mcp](https://github.com/shuji-bonji/pdf-spec-mcp) | Canon layer — ISO 32000 specification access |
| [houki-research-skill](https://github.com/shuji-bonji/houki-research-skill) | Sibling orchestration skill for Japanese law research |
| [claude-plugins](https://github.com/shuji-bonji/claude-plugins) | Personal marketplace (future distribution channel) |
