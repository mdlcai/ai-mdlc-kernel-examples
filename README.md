<div align="center">

# MDLC — Example Builds

### Eleven real applications, each built end-to-end by the MDLC methodology.

**[🌐 Browse the live gallery →](https://mdlcai.github.io/ai-mdlc-kernel-examples/)**

[mdlc.ai](https://mdlc.ai) · From a one-page blueprint to a gate-passing build — no hand-tuning after generation.

</div>

---

## What you're looking at

**MDLC** (the Markdown Development Life Cycle) turns a single `RESEARCH.md` blueprint into a working, reviewed application — driving it through architecture, build, and a battery of automated quality gates. These eleven examples are the proof. Each one was generated, not hand-written, and each ships the **complete paper trail**: the blueprint that started it, the architecture that fell out of it, the spec and decisions it was built to, and the build report showing every gate it passed.

Open any folder and read it top to bottom — `RESEARCH.md` → `ARCHITECTURE.md` → `SPEC.md` → `DECISIONS.md` → `REPORT.md` — and you can follow the process end to end.

## The gallery

| | App | What it is | Gates passed |
|---|-----|-----------|--------------|
| [<img src="tradewind/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/tradewind/index.html) | **[Tradewind](tradewind/)** | Services marketplace with milestone escrow — fund, release on approval, payouts, disputes, and verified Stripe webhooks, every movement an immutable double-entry ledger entry that always balances. | 29 tests · 17/17 smoke · 12 invariants (double-entry Σ=0) · 0 crit/high |
| [<img src="patchlab/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/patchlab/index.html) | **[PatchLAB](patchlab/)** | AI sound design for FL Studio producers — a plain-language prompt becomes WAV samples, synth presets, and packs via a deterministic offline DSP engine. | 33 tests · 10/10 smoke · 12/12 invariants · 0 crit/high |
| [<img src="sentinel/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/sentinel/index.html) | **[Sentinel](sentinel/)** | Unified AppSec scanning — runs OSS scanners (SAST, SCA, DAST, secrets, IaC) in sandboxed containers, then dedupes and prioritizes into one dashboard. | 21 tests · 23/23 smoke · 11/11 invariants · 0 crit/high |
| [<img src="mcpsentinel/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/mcpsentinel/index.html) | **[MCP Sentinel](mcpsentinel/)** | Audits the security posture of MCP servers — tool poisoning, scope, leaked secrets, rug pulls, missing auth. | 37 tests · 17/17 smoke · 10/10 invariants |
| [<img src="pulse/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/pulse/index.html) | **[Pulse](pulse/)** | Real-time infrastructure monitoring with a flap-resistant alert engine and sub-minute detection. | 21/21 smoke · 11/11 invariants |
| [<img src="recon/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/recon/index.html) | **[Recon](recon/)** | Continuous attack-surface monitoring — alerts the moment your internet exposure changes. | 29 tests · 8/8 smoke · 15/15 invariants |
| [<img src="smartcoder/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/smartcoder/index.html) | **[SmartCoder](smartcoder/)** | Medical-coding workflow & QA training on synthetic encounters — no real PHI. | 14 tests · 16/16 smoke · Reviewer PASS |
| [<img src="servicehub/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/servicehub/index.html) | **[ServiceHub](servicehub/)** | Multi-tenant IT service management — requests, incidents, approvals, SLA tracking. | 27/27 smoke (incl. same-tenant IDOR) · 15/15 invariants · 0 crit/high |
| [<img src="workgrid/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/workgrid/index.html) | **[WorkGrid](workgrid/)** | One workspace for tickets, tasks, and projects — self-routing work, live dashboards. | 15/15 smoke · 13/13 invariants |
| [<img src="ember/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/ember/index.html) | **[Ember](ember/)** | A specialty coffee roaster's storefront — catalog, tasting notes, cart, Stripe checkout + subscriptions. | 39 tests · 23 smoke · 12 invariants |
| [<img src="bookflow/preview.png" width="220">](https://mdlcai.github.io/ai-mdlc-kernel-examples/bookflow/index.html) | **[BookFlow](bookflow/)** | Booking & reservation platform — approval workflows, DB-guaranteed conflict prevention (no double-booking), audit trail. | 10 tests · 17/17 smoke · 9/9 invariants |

> Previews are self-contained, multi-screen HTML — design tokens and screens lifted from each deployed build. Use the screen switcher in the top-right of any preview to walk the app.

## What's in each folder

Every example ships the same evidence pack:

| File | Stage | What it proves |
|------|-------|----------------|
| `RESEARCH.md` | Research | The blueprint — vision, users, threat model, GO/NO-GO |
| `ARCHITECTURE.md` + `architecture.html` | Architecture | System design, data flow, layer by layer |
| `SPEC.md` | Contract | The API surface and UI the build was held to |
| `DECISIONS.md` | Contract | Every ADR — the *why* behind each choice |
| `COMPLIANCE.md` | Assurance | OWASP / controls mapping |
| `SECURITY-AUDIT.md` | Assurance | Independent security review *(security-domain builds)* |
| `REPORT.md` | Build report | **Every gate that ran, with evidence** — start here if you want proof |
| `index.html` | Output | Standalone, multi-screen working preview |

The runnable application source isn't checked in — these folders are the **showcase and the receipts**, not a deployable repo.

## Why this matters

Most "AI built my app" demos show you the happy-path screenshot. These show you the gates: input reconciliation (every blueprint field provably honored), interface-contract validation, a verification gate (typecheck/lint/tests/build all green), machine-checked security invariants, an independent reviewer pass, and an end-to-end functional smoke test. The screenshots are nice. The [`REPORT.md`](mcpsentinel/REPORT.md) files are the point.

---

<div align="center">

**Want one of these for your idea?** → **[mdlc.ai](https://mdlc.ai)**

</div>
