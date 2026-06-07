# BookFlow

**Bookings and approvals, on your rules — reservations for rooms, equipment, and services.**

[▶ Live preview](https://mdlcai.github.io/ai-mdlc-kernel-examples/bookflow/index.html) · [System architecture](https://mdlcai.github.io/ai-mdlc-kernel-examples/bookflow/architecture.html) · [Build with MDLC →](https://mdlc.ai)

![BookFlow](preview.png)

> One of eight reference apps built end-to-end with the **[MDLC](https://mdlc.ai)** methodology — from a `RESEARCH.md` blueprint, through architecture and build, to a passing set of quality gates. Nothing here was hand-tuned after generation.

## What it does

A configurable booking-and-reservation platform: one portal for employees to request rooms, equipment, facilities, and services, with **customizable approval workflows**, **database-guaranteed conflict prevention** (no double-booking), and a full audit trail. Replaces the spreadsheet/email/calendar sprawl that causes double bookings, approval bottlenecks, and zero visibility.

## Built from a blueprint

Every file below was generated in sequence. Read them in order to see the methodology work:

| Stage | Artifact | What it is |
|-------|----------|------------|
| 1 · Research | [`RESEARCH.md`](RESEARCH.md) | Product vision, users, threat model, GO/NO-GO |
| 2 · Architecture | [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`architecture.html`](https://mdlcai.github.io/ai-mdlc-kernel-examples/bookflow/architecture.html) | System design, data flow, layer-by-layer |
| 3 · Contract | [`SPEC.md`](SPEC.md) · [`DECISIONS.md`](DECISIONS.md) | API surface + the ADRs behind every choice |
| 4 · Assurance | [`COMPLIANCE.md`](COMPLIANCE.md) · [`SECURITY-AUDIT.md`](SECURITY-AUDIT.md) | SOC2-aligned controls mapping + security review |
| 5 · Build report | [`REPORT.md`](REPORT.md) | Every gate that ran, with evidence |

## The gates it passed

Straight from [`REPORT.md`](REPORT.md):

- **10 / 10** unit + integration tests green
- **17 / 17** end-to-end smoke flows PASS — including **W5 booking conflict → 409** (no double-booking) and tenant isolation
- **9 / 9** machine-checked invariants
- Reliable notifications via an **outbox** (dual-write) — 18 emails delivered, 0 pending

## Stack

`Next.js` · `NestJS` · `Postgres` · `REST` · `Docker Compose`
Domain signals: `has_webhooks` · `has_dual_write`

---

*This folder ships the standalone preview + the build's evidence pack. The runnable application source lives in the build, not here.* **[mdlc.ai](https://mdlc.ai)**
