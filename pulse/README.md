# Pulse

**Real-time infrastructure monitoring — know the instant something breaks.**

[▶ Live preview](https://mdlcai.github.io/ai-mdlc-kernel-examples/pulse/index.html) · [System architecture](https://mdlcai.github.io/ai-mdlc-kernel-examples/pulse/architecture.html) · [Build with MDLC →](https://mdlc.ai)

![Pulse](preview.png)

> One of eight reference apps built end-to-end with the **[MDLC](https://mdlc.ai)** methodology — from a `RESEARCH.md` blueprint, through architecture and build, to a passing set of quality gates. Nothing here was hand-tuned after generation.

## What it does

Real-time monitoring for servers, networks, and services: multi-protocol health checks, a **flap-resistant alert engine**, and guaranteed-delivery notifications. Sub-minute detection with zero alert fatigue — live topology, incident timelines, and per-check history.

## Built from a blueprint

Every file below was generated in sequence. Read them in order to see the methodology work:

| Stage | Artifact | What it is |
|-------|----------|------------|
| 1 · Research | [`RESEARCH.md`](RESEARCH.md) | Product vision, users, threat model, GO/NO-GO |
| 2 · Architecture | [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`architecture.html`](https://mdlcai.github.io/ai-mdlc-kernel-examples/pulse/architecture.html) | System design, data flow, layer-by-layer |
| 3 · Contract | [`SPEC.md`](SPEC.md) · [`DECISIONS.md`](DECISIONS.md) | API surface + the ADRs behind every choice |
| 4 · Assurance | [`COMPLIANCE.md`](COMPLIANCE.md) · [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | OWASP mapping + security review |
| 5 · Build report | [`REPORT.md`](REPORT.md) | Every gate that ran, with evidence |

## The gates it passed

Straight from [`REPORT.md`](REPORT.md):

- **21 / 21** end-to-end smoke flows PASS
- **11 / 11** machine-checked invariants
- Clean `typecheck` · `lint` · `build` · invariant-lint — all exit 0
- **Reviewer Gate: PASS**

## Stack

`React SPA` · `Express on Node` · `Postgres` · `REST` · `Docker Compose`
Domain signals: `has_webhooks` · `has_websocket` · `has_geo`

---

*This folder ships the standalone preview + the build's evidence pack. The runnable application source lives in the build, not here.* **[mdlc.ai](https://mdlc.ai)**
