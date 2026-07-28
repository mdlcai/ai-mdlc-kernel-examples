# RESEARCH.md — SYNAPSE

build_depth: standard

---

## Product Vision

**Problem:** Engineers responsible for internet-facing infrastructure juggle a dozen
fragmented utilities when something breaks. Separate tools for DNS, TLS, HTTP timing, and
reachability, each with its own interface and blind spots, slow down root-cause analysis and
leave gaps in visibility.

**Solution:** SYNAPSE is a single web app that unifies on-demand network diagnostics with
lightweight continuous monitoring. An engineer runs a diagnostic once (DNS, SSL, HTTP,
reachability), and can promote any check into a monitor that runs on a schedule and alerts on
change. One place to move from reactive troubleshooting to proactive monitoring.

**Who it is for:** Network engineers, SREs, and IT administrators managing a handful to a few
dozen internet-facing endpoints.

**What this build is (and is not):** This is a self-contained reference build demonstrating the
diagnostics + monitoring core as one deployable app. It intentionally excludes the full
production platform's multi-service edge architecture, payment billing, and privileged scanning
(see Non-Goals). Those are out of scope for a single governed build.

## Core Features

### 1. Diagnostic tools (on-demand, no signup)
- **DNS Lookup** — resolve A, AAAA, CNAME, MX, TXT, NS for a domain via a public DoH resolver.
- **SSL Certificate Checker** — read a host's live certificate: issuer, validity dates, days to
  expiry, SANs, trust.
- **HTTP Header Check** — status code, redirect chain, response headers, and phase timing
  (DNS, connect, TLS, TTFB, total).
- **DNS Propagation** — query a record from several public resolvers and show agreement/drift.
Each tool has its own page with a shared result layout and a "monitor this" call to action.

### 2. Monitoring
- A signed-in user can create **monitors** from any diagnostic type (uptime/HTTP, SSL expiry,
  DNS record).
- Each monitor stores a target, a check type, an interval, and an optional expected value.
- A scheduled job runs due monitors, records the result, and computes status (up/down/warning).
- Monitor detail shows recent check history and current status.

### 3. Alerting
- Per-monitor alert configuration: email and/or webhook.
- On a status change (up->down, or SSL entering a warning window), an alert is dispatched once
  per transition (no repeat spam while still down).

### 4. Accounts
- Email/password auth. A user sees only their own monitors and alerts.

## Tech Stack

- **Framework:** Next.js (App Router) + React + TypeScript (strict).
- **Styling:** Tailwind CSS. Dark, technical, instrument-panel aesthetic. Monospace for data.
- **Data:** Postgres (via a hosted provider) for users, monitors, checks, alerts.
- **Auth:** email/password with sessions; row-level ownership enforced on every query.
- **Scheduling:** a cron endpoint that runs due checks (invoked by an external scheduler).
- **Deploy target:** a single app (Vercel or a Node host). No multi-worker split for this build.

## Data Model

- **users**: id, email, password_hash, created_at.
- **monitors**: id, user_id (fk), name, target, check_type (http|ssl|dns), interval_seconds,
  expected_value (nullable), status (up|down|warning|unknown), last_checked_at, created_at.
- **checks**: id, monitor_id (fk), ran_at, ok (bool), latency_ms (nullable), detail (jsonb).
- **alert_channels**: id, monitor_id (fk), kind (email|webhook), destination.
- **alerts**: id, monitor_id (fk), opened_at, closed_at (nullable), reason.

Ownership: every monitors/checks/alerts row traces to a user_id; queries filter by the session
user. A user can never read or mutate another user's rows.

## Key Flows

1. **Anonymous diagnostic:** visitor opens /tools/ssl-checker, enters a host, sees the live
   certificate result, and a prompt to sign up to monitor it.
2. **Create a monitor:** signed-in user picks a check type + target + interval, saves it.
3. **Scheduled run:** cron hits /api/cron; the app selects due monitors, runs each check,
   writes a checks row, updates monitor status, and on a status transition opens/closes an alert
   and dispatches configured channels.
4. **Review:** user opens a monitor to see status + recent check history.

## Security Requirements

- All monitor/check/alert access is scoped to the session user (ownership check on every query).
- The cron endpoint is authenticated with a shared secret; it is not publicly triggerable.
- Outbound diagnostic requests are guarded against SSRF: reject requests to private/loopback/
  link-local IP ranges and non-http(s) schemes.
- Passwords hashed with a modern KDF. No secrets in client code or the repo.
- Webhook alert destinations are validated (https only, no internal addresses).

## Non-Goals (explicitly out of scope for this build)

- Payment/billing and paid tiers.
- The production multi-service edge architecture (separate probe worker, Durable-Object monitor
  fleet, queue consumer).
- Privileged/aggressive scanning (nmap, port scans, packet capture) and the isolated probe host.
- Multi-region distributed probing.
- Team seats, status pages, and the full 30-tool catalog (four representative tools only).

## Success Criteria

- All four diagnostic tools return correct live results for a known-good and a known-bad host.
- A user can create a monitor, a scheduled run records a check and updates status, and a
  simulated down transition opens an alert and dispatches a webhook exactly once.
- Ownership isolation holds: user B cannot read or modify user A's monitors via any endpoint.
- SSRF guard rejects a monitor/diagnostic targeting a private IP.
- Typecheck passes; the app builds and runs.
