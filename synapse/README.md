# SYNAPSE

**Unified network diagnostics + lightweight monitoring, as one deployable app.**

Engineers responsible for internet-facing infrastructure juggle a dozen
fragmented utilities when something breaks — separate tools for DNS, TLS, HTTP
timing, and reachability, each with its own interface and blind spots. SYNAPSE
puts them in one place: run a diagnostic once, then promote any check into a
monitor that runs on a schedule and alerts on change. One tool to move from
reactive troubleshooting to proactive monitoring.

> **This is a reference build.** It is a self-contained demonstration of the
> diagnostics + monitoring core as a single governed build. It intentionally
> excludes the full production platform's multi-service edge architecture,
> payment billing, and privileged scanning. See `RESEARCH.md §Non-Goals`.

## What's in it

- **Four diagnostic tools (public, no signup):** DNS Lookup, SSL Certificate
  Checker, HTTP Header Check (with phase timing), and DNS Propagation.
- **Monitoring:** a signed-in user turns any check into a monitor with a target,
  interval, and optional expected value; a scheduled runner records results and
  computes status (up/down/warning).
- **Alerting:** per-monitor email/webhook channels, dispatched exactly once per
  status transition (no repeat spam while still down).
- **Accounts:** email/password auth; every user sees only their own monitors,
  checks, and alerts.

## Stack

- **Framework:** Next.js 15 (App Router) + React 19 + TypeScript (strict)
- **Styling:** Tailwind CSS — dark, technical, instrument-panel aesthetic
- **Data:** PostgreSQL via `pg` (users, monitors, checks, alert channels, alerts,
  sessions)
- **Auth:** email/password with scrypt hashing + opaque server-side sessions
  (HttpOnly cookie; only the token hash stored)
- **Scheduling:** a `POST /api/cron` endpoint (shared-secret auth) invoked by an
  external scheduler
- **Deploy target:** a single Node app (Vercel or any Node host) — no
  multi-worker split

## Prerequisites

- Node.js 20+
- A PostgreSQL database (a hosted provider, or local Postgres for dev)

## Setup

### 1. Install

```bash
npm install
```

### 2. Environment

Copy the example env and fill it in (never commit secrets):

```bash
cp .env.example .env.local
```

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string. |
| `CRON_SECRET` | Long random string the external scheduler must present to `POST /api/cron`. The endpoint is **not** publicly triggerable. |
| `PGCA` *(optional)* | Trusted CA bundle for verifying the DB server's TLS cert. Most hosted Postgres providers publish one. If unset, the system trust store is used. |
| `PGSSL=disable` *(dev only)* | Plaintext DB connection for a **local** Postgres without TLS. Never set in production. |
| `ALERT_EMAIL_FROM`, `SMTP_URL` *(optional)* | Outbound email transport for alert channels. |

> **TLS note:** outside `PGSSL=disable`, the app verifies the database server
> certificate (`rejectUnauthorized: true`). Supply a CA via `PGCA` if your
> provider isn't publicly trusted. This is a security requirement, not a
> convenience toggle — see `SECURITY-AUDIT.md`.

### 3. Database

Apply the schema (creates `users`, `monitors`, `checks`, `alert_channels`,
`alerts`, `sessions`, and their indexes):

```bash
psql "$DATABASE_URL" -f db/schema.sql
```

### 4. Run locally

```bash
npm run dev        # http://localhost:3000
```

Type-check before pushing:

```bash
npm run typecheck  # tsc --noEmit (strict) — expected: exit 0
```

Production build:

```bash
npm run build && npm run start
```

### Trigger a scheduled run

Point an external scheduler at the cron endpoint (or invoke it manually with the
shared secret):

```bash
curl -X POST http://localhost:3000/api/cron \
  -H "Authorization: Bearer $CRON_SECRET"
```

It selects due monitors, runs each check, records a result, updates status, and
on a transition opens/closes an alert and dispatches the configured channels.

## Project layout

```
app/
  tools/            four public diagnostic tools (dns, ssl, http, propagation)
  monitors/         monitor list, detail, and create flows
  (auth)/           login / signup
  api/
    diagnostics/    public diagnostic handlers + shared engine (core.ts)
    auth/           signup / login / logout
    monitors/       owner-scoped monitor + channel + check + alert CRUD
    cron/           scheduled runner (route + run-check + dispatch)
components/         header, UI primitives, alert components
lib/
  db.ts             Postgres pool + ownership-scoped query helpers
  auth.ts           scrypt hashing + session management
  ssrf.ts           outbound-request safety gate (SSRF guard + address pinning)
  types.ts          shared row/DTO shapes
db/schema.sql       the full schema
```

## Security

The build passed a three-lens security review — ownership isolation (IDOR),
SSRF / outbound-request safety, and secrets & credential confidentiality. The
full findings and remediations are in **[`SECURITY-AUDIT.md`](./SECURITY-AUDIT.md)**.
Highlights: all monitor/check/alert access is scoped to the session user;
outbound requests are SSRF-guarded and pinned to the validated IP (closing the
DNS-rebinding window); the cron endpoint uses a constant-time shared-secret check;
and the DB connection authenticates the server certificate.

---

## Built with MDLC

This is a reference build produced with **MDLC** — the Managed Development
Lifecycle. It was generated from [`RESEARCH.md`](./RESEARCH.md) through the MDLC
kernel: the blueprint drove the architecture, feature slices, and quality gates,
and every deliverable was checked against it. The receipts for this governed
build live alongside this file:

- **[`RESEARCH.md`](./RESEARCH.md)** — the blueprint the build was generated from
- **[`BUILD-REPORT.md`](./BUILD-REPORT.md)** — what was built, the architecture,
  the feature slices, the quality gates, and the final typecheck status
- **[`SECURITY-AUDIT.md`](./SECURITY-AUDIT.md)** — the three-lens security gate,
  every finding, its remediation, and the final verdict

Learn more at **[mdlc.ai](https://mdlc.ai)**.
