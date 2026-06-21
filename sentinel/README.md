# Sentinel

**Unify your security findings into one short, trustworthy list.**

[▶ Live preview](https://mdlcai.github.io/ai-mdlc-kernel-examples/sentinel/index.html) · [System architecture](https://mdlcai.github.io/ai-mdlc-kernel-examples/sentinel/architecture.html) · [Build with MDLC →](https://mdlc.ai)

![Sentinel](preview.png)

> One of eleven reference apps built end-to-end with the **[MDLC](https://mdlc.ai)** methodology — from a `RESEARCH.md` blueprint, through architecture and build, to a passing set of quality gates. Nothing here was hand-tuned after generation.

## What it does

Sentinel is a unified AppSec scanning platform. Point it at a repo, domain, or IP and it orchestrates best-of-breed open-source scanners (SAST, SCA, DAST, secrets, IaC) in **sandboxed containers**, then normalizes, deduplicates, and prioritizes every result into a single developer-first dashboard — turning a week of tool-wrangling into one connected view. Built for engineering teams without a dedicated security org.

## Built from a blueprint

Every file below was generated in sequence. Read them in order to see the methodology work:

| Stage | Artifact | What it is |
|-------|----------|------------|
| 1 · Research | [`RESEARCH.md`](RESEARCH.md) | Product vision, users, threat model, GO/NO-GO |
| 2 · Architecture | [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`architecture.html`](https://mdlcai.github.io/ai-mdlc-kernel-examples/sentinel/architecture.html) | System design, scan pipeline, layer-by-layer |
| 3 · Contract | [`SPEC.md`](SPEC.md) · [`DECISIONS.md`](DECISIONS.md) | API surface + the ADRs behind every choice |
| 4 · Assurance | [`COMPLIANCE.md`](COMPLIANCE.md) · [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | OWASP mapping + security review |
| 5 · Build report | [`REPORT.md`](REPORT.md) · [`SMOKE-TEST.md`](SMOKE-TEST.md) | Every gate that ran + the functional smoke matrix |

## The gates it passed

Straight from [`REPORT.md`](REPORT.md):

- **21** tests green
- **23 / 23** functional smoke flows PASS (+ a headless-browser UI walkthrough)
- **11 / 11** machine-checked invariants (+ 2 manual verified)
- **Security audit: PASS** — 0 critical / 0 high (real SSRF/CIDR blocking, hardened container sandbox, multi-tenant scoping)
- Clean sequential verification run — `typecheck` · `build` · invariant-lint all exit 0

## Stack

`Next.js` · `Express` · `PostgreSQL` · `Redis / BullMQ` · `Docker Compose`
Domain signals: `has_webhooks` · `has_geo` · `has_dual_write`

---

*This folder ships the standalone preview + the build's evidence pack. The runnable application source lives in the build, not here.* **[mdlc.ai](https://mdlc.ai)**
