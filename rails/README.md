# RΛILS

**Programmable financial infrastructure. One API for payments, subscriptions, refunds and payouts, with a double-entry ledger that records every movement of money.**

[▶ Interactive demo](https://mdlc.ai/rails-demo.html) · [Build dossier](https://mdlcai.github.io/ai-mdlc-kernel-examples/rails/index.html) · [Build with MDLC →](https://mdlc.ai)

![RΛILS](preview.webp)

> One of fourteen reference apps built end-to-end with the **[MDLC](https://mdlc.ai)** methodology, from a `RESEARCH.md` blueprint, through architecture and build, to a passing set of quality gates.

## What it does

A multi-tenant payment orchestration control plane: one API (`/api/v1`), processor adapters (Stripe and a sandbox), a deterministic payment state machine, exactly-once idempotency, signed webhooks, a realtime console, and an append-only double-entry ledger that a database trigger protects from updates and deletes.

The interactive demo is **read-only** and runs entirely in your browser on simulated data. It does not connect to a running RΛILS, and nothing you do in it is saved. The numbers below come from the real build.

## Built from a blueprint

| Stage | Artifact | What it is |
|-------|----------|------------|
| 1 · Research | [`RESEARCH.md`](RESEARCH.md) | Product vision, workflows, threat model, GO decision |
| 2 · Architecture | [`ARCHITECTURE.md`](ARCHITECTURE.md) | System design and invariants, each tied to its proof |
| 3 · Contract | [`SPEC.md`](SPEC.md) · [`DECISIONS.md`](DECISIONS.md) | Screens, endpoints, and the ADRs behind them |
| 4 · Assurance | [`COMPLIANCE.md`](COMPLIANCE.md) · [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | Control mapping and the security audit with its re-scan |
| 5 · Governance | [`GOVERNANCE-DECISIONS.jsonl`](GOVERNANCE-DECISIONS.jsonl) | Append-only decision records, including the human approvals |
| 6 · Build report | [`REPORT.md`](REPORT.md) | Every gate, every run, through the v1.0.1 ship |
| Evidence | [`evidence/`](evidence/) | Reviewer rounds 1 to 8 and the security passes |

## The gates it passed

| Gate | Result |
|------|--------|
| Tests | API 341 / 341 (96.3% coverage), web unit 64 / 64, end-to-end 26 / 26 on desktop and mobile |
| Smoke | 53 / 53 |
| Invariants | 39 / 39 machine-checked, 1 manual check left to the reviewer |
| Reviewer | Rounds 1 to 3 FAIL, fixed (including a refund retry that could refund twice); rounds 4 to 8 PASS |
| Security audit | Pass 1: 1 high, 2 medium, 3 low. After the re-scan: 0 critical, 0 high, 0 medium; 3 low deferred or residual with reasons |
| Frontend audit | 150 cells, only failure HSTS on localhost (ADR-048); landing Lighthouse performance 93, accessibility 100 |
| Compliance | 62 implemented, 7 with a recorded residual, 0 gaps |

## Honest posture

- Live Stripe payments were not verified; Stripe runs in contract mode until keys are provisioned (ADR-008).
- The sustained load test misses its latency targets on a single local host, with 0% errors and 0 duplicate charges.
- HSTS is absent on the localhost edge (ADR-048).

The [build dossier](https://mdlcai.github.io/ai-mdlc-kernel-examples/rails/index.html) has the full gate history and every open item.

## Stack

Rails 8.1 API and a Next.js 16 console on PostgreSQL 18 (primary and replica), Redis, Solid Queue and Action Cable, behind Caddy in Docker Compose. Playwright end-to-end on Desktop Chrome and Pixel 7.
