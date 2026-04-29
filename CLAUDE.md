# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

This is a **Chinese-language Web3 technical knowledge base** — a pure Markdown docs repository (no build, no tests, no package manager). All work consists of authoring, revising, and verifying `.md` articles that follow a strict shared structure.

Read these four files before meaningful edits; they are the contract:

- `CONTRIBUTING.md` — writing style rules (frontmatter, terminology, citations, code blocks, prohibited language)
- `TEMPLATE.md` — the 10-section article skeleton every topic file must follow
- `SOURCES.md` — the Tier 1–5 citation allowlist; every technical claim needs Tier 1 + Tier 2/3 cross-reference
- `PROGRESS.md` — single source of truth for each article's status; must be updated when status changes

`README.md` is the navigation index; keep its links in sync when adding/renaming files.

## Content architecture

Top-level directories are numbered and represent the knowledge taxonomy. Do not renumber or reorganize without explicit instruction:

- `00-overview/` — landscape, history, glossary
- `01-infrastructure/` — public chains, consensus, ledger models, cross-chain, oracle, layer2, DA
- `02-wallet/` — cryptography, HD wallets, custody models, AA, hardware
- `03-smart-contract/` — EVM, Solana, Move, other VMs
- `04-dapp/` — token standards, DeFi, NFT, stablecoin, RWA, social/DAO, gaming
- `05-security/` — audit methodology, vuln classes, tools
- `06-third-party/` — RPC, indexing, analytics, KYT, audit firms, explorers, dev infra
- `07-privacy/` — ZKP, FHE, MPC, mixers, TEE
- `08-security-incidents/` — event post-mortems organized by **attack vector** subdirectory (`bridge/`, `cex-custodian/`, `defi-protocol/`, `oracle-mev/`, `supply-chain-phish/`, `wallet-key/`, `governance-admin/`, `misc/`), filenames prefixed with year: `YYYY-<project>.md`

Many subdirectories have an `_index.md` acting as overview + comparison table. Update the `_index.md` when adding a sibling article.

## Hard rules for every article

**Frontmatter is required** and schema is fixed (see `CONTRIBUTING.md` §二):
```yaml
---
title: <中文标题（English Term）>
module: <path>
priority: P0 | P1 | P2
status: TODO | DRAFT | REVIEWED | DONE
word_count_target: 5000
last_verified: YYYY-MM-DD
primary_sources: [...]
---
```

Status flow is `TODO → DRAFT → REVIEWED → DONE`. When you change status, also update the corresponding line in `PROGRESS.md`.

**Article body** must follow the 10-section structure in `TEMPLATE.md`. Core-principle section (§2) has a ≥1500-word floor with 4–6 subsections; architecture section (§3) has a ≥1200-word floor with 3–5 subsections. Each P0/P1 article aims at `word_count_target` (default 5000).

**Chinese + English term convention**: first use `中文（English Term, ABBR）`, thereafter the abbreviation. Protocol names, product names, EIP/BIP numbers, opcodes, file/function names stay in original form. Glossary consistency table is in `00-overview/glossary.md` and `CONTRIBUTING.md` §三.

**Citations**:
- Every technical assertion needs ≥1 Tier 1 source (official spec/whitepaper/official GitHub) plus a Tier 2/3 cross-reference.
- Code references use `repo/path/file.ext:line` and the frontmatter must pin a commit hash or release tag along with the code repository url.
- Numeric data (TVL, TPS, market cap, validator counts) must carry a date: `（截至 2026-04-29，来源 DefiLlama、rwa.xyz、coindesk、coinmarketcap）`.
- Tier 5 (Medium/Twitter/Wikipedia) is auxiliary only — never sole source.
- Security-incident articles have an extra source allowlist in `SOURCES.md` (SlowMist Hacked, PeckShield, BlockSec, rekt.news, DeFiHackLabs, FBI/OFAC, victim post-mortems). Each incident needs ≥1 Tier 1 (official post-mortem / FBI / OFAC / on-chain evidence from SlowMist or PeckShield) + 1 independent cross-source.

**Today's date** for `last_verified` and data-freshness callouts: **2026-04-29** (per user config).

## Figures and code blocks

- Prefer **Mermaid** over images for diagrams so they stay text-reviewable. Each core-principle section should include ≥1 Mermaid state/sequence diagram plus an ASCII structure sketch.
- Image assets go in `<module>/<topic>/assets/` named `fig-01-xxx.png` with source/date captions.
- Code blocks must declare language; keep snippets ≤80 lines with Chinese comments; replace secrets with `<YOUR_KEY>`.

## Prohibited

- No fabricated numbers — if no authoritative source exists, use "据 XX 估计…" instead of a specific figure.
- No marketing language ("革命性"/"颠覆"/"史诗级") and no verbatim copy of official docs — always restructure and translate.
- No Medium/Twitter threads as the only source.
- Financial/security judgments must stay neutral and state "本内容不构成投资建议".

## Common workflows

- **Adding a new topic**: create the file from `TEMPLATE.md`, add a link in the relevant `_index.md` and in `README.md`, register the row in `PROGRESS.md` with initial status.
- **Promoting DRAFT → REVIEWED**: verify every external link returns 200, every code reference resolves at the pinned commit/tag, and cross-source claims hold; bump `last_verified`.
- **Deep-dive follow-ups** (see recent `docs/*-deep-dive` branches): land as a separate PR on a `docs/<topic>-deep-dive` branch; do not fold into unrelated edits.
- **Commit messages** follow `docs(<area>): <summary>` (see `git log`). Group docs changes with matching area tags (`docs(morpho)`, `docs(stargate)`, etc.).

## What this repo does not have

No build, lint, test, or CI commands. Do not introduce a toolchain (package.json, Makefile, pre-commit) unless the user explicitly asks. Verification is manual: link checks, source re-reads, commit-pinned code spot-checks.
