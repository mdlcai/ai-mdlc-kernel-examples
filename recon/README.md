# Recon

**Continuous attack-surface monitoring — see your exposure change in real time.**

[▶ Live preview](https://mdlcai.github.io/ai-mdlc-kernel-examples/recon/index.html) · [System architecture](https://mdlcai.github.io/ai-mdlc-kernel-examples/recon/architecture.html) · [Build with MDLC →](https://mdlc.ai)

![Recon](preview.png)

> One of eight reference apps built end-to-end with the **[MDLC](https://mdlc.ai)** methodology — from a `RESEARCH.md` blueprint, through architecture and build, to a passing set of quality gates. Nothing here was hand-tuned after generation.

## What it does

Recon continuously watches the systems, ports, and services an organization exposes to the internet and **alerts the moment the attack surface changes** — a new exposure, a drifted firewall rule, a service that shouldn't be there. Firewall changes, cloud deploys, and vendor modifications stop being blind spots.

## Built from a blueprint

Every file below was generated in sequence. Read them in order to see the methodology work:

| Stage | Artifact | What it is |
|-------|----------|------------|
| 1 · Research | [`RESEARCH.md`](RESEARCH.md) | Product vision, users, threat model, GO/NO-GO |
| 2 · Architecture | [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`architecture.html`](https://mdlcai.github.io/ai-mdlc-kernel-examples/recon/architecture.html) | System design, data flow, layer-by-layer |
| 3 · Contract | [`SPEC.md`](SPEC.md) · [`DECISIONS.md`](DECISIONS.md) | API surface + the ADRs behind every choice |
| 4 · Assurance | [`COMPLIANCE.md`](COMPLIANCE.md) · [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | OWASP mapping + security review |
| 5 · Build report | [`REPORT.md`](REPORT.md) | Every gate that ran, with evidence |

## The gates it passed

Straight from [`REPORT.md`](REPORT.md):

- **29** tests green
- **8 / 8** functional smoke flows PASS
- **15 / 15** machine-checked invariants
- Clean sequential verification run — `typecheck` · `build` · invariant-lint all exit 0

## Stack

`Next.js` · `NestJS` · `Postgres` · `REST` · `Docker Compose`
Domain signals: `has_webhooks` · `has_geo` · `has_websocket` · `has_dual_write`

---

*This folder ships the standalone preview + the build's evidence pack. The runnable application source lives in the build, not here.* **[mdlc.ai](https://mdlc.ai)**
