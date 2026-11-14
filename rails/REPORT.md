# REPORT.md — RΛILS build report

Project `cd32b70e-34d4-468e-8d26-6afe7aab6492` · build_depth comprehensive · review_gates auto

## Build Input Reconciliation

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| `build_depth` | comprehensive | Applied | SPEC.md exhaustive §5, threat model RESEARCH §5, SECURITY-AUDIT.md, load test |
| `review_gates` | auto | Applied | No human halts except the allowed ones; outcomes logged in DECISIONS.md |
| `force_research` | false | Applied | §3 was empty, so research ran once (Stage 0) |
| `domain` | Finance | Applied | PCI posture, ledger, money invariants |
| `has_webhooks` | signal | Applied | SPEC §5 F-11; INV-3, INV-6 |
| `has_geo` | signal | Applied | SPEC §5 F-08 risk signals (coarse lat/lng NUMERIC, bounds CHECK) |
| `has_scheduled_work` | signal | Applied | ARCHITECTURE §6 T1–T6; INV-4, INV-40 |
| `has_webhook_send` | signal | Applied | SPEC §5 F-12; INV-5, INV-26–28 |
| `has_payments` | signal | Applied | ARCHITECTURE §4–§5; INV-1, INV-2, INV-14–25 |
| `has_websocket` | signal | Applied | ARCHITECTURE §7 (Action Cable); INV-33 |
| `has_email` | signal | Applied | SPEC §5 F-03; INV-29 |
| `has_image_uploads` | signal | Applied | SPEC §5 F-16 (merchant logo pipeline) |
| `has_dual_write` | signal | Applied | Outboxes `pending_webhooks` / `pending_emails`; Solid Queue in the same DB (ARCHITECTURE §10) |
| `protocol_support` | HTTPS only | Applied | `compose.yaml` service `edge` (Caddy `tls internal`, publishes `${HTTPS_PORT:-9443}:443`, `${HTTP_PORT:-9080}:80` → redirect); `edge/Caddyfile` |
| `monitoring` | basic health checks | Applied | `GET /api/health` (INV-7); compose healthchecks |
| `container_strategy` | Docker Compose | Applied | `compose.yaml`, `api/Dockerfile`, `web/Dockerfile` |
| `database_preference` | PostgreSQL | Applied | ARCHITECTURE §4 (Postgres 18 primary + replica) |
| `pii_handling` | encrypted at rest + in transit | Applied | Active Record Encryption (INV-34); TLS at the edge; volume encryption delegated (COMPLIANCE ⚠) |
| `security_baseline` | ["PCI-DSS"] | Applied | SPEC §4 PCI rows; COMPLIANCE.md; ADR-005 |
| `rate_limiting` | true | Applied | `api/config/initializers/rack_attack.rb` (INV-32) |
| `audit_logging` | true | Applied | Hash-chained `audit_events` (ARCHITECTURE §3) |
| `frontend_framework` | Next.js | Applied | `web/` Next.js 16 |
| `state_management` | React Context | Applied | `web/context/*` |
| `backend_framework` | Rails | Applied | `api/` Rails 8.1 API |
| `typescript_backend` | false | Applied | Ruby backend |
| `realtime_needed` | true | Applied | Action Cable `MerchantEventsChannel` |
| `performance_requirements` | availability ≥99.99%, p95 <250ms, p95 <50ms | Applied (latency) / Deferred (availability) | SPEC §9 + load test; availability delegated to multi-AZ prod topology (ADR-016) |
| `logging_format` | structured JSON | Applied | `api/config/initializers/logging.rb`; web logs JSON |
| `scale` | large — 50k+ concurrent → tier `large` (+ has_payments row locks, has_websocket Redis cable) | Applied | ARCHITECTURE §2 scale table |
| `multi_tenant` | true | Applied | merchant_id scoping, INV-10, cross-tenant smoke |
| `target_platforms` | ["web"] | Applied | Responsive web; no native apps |
| `hard_dependencies` | ["Stripe"] | Applied | `StripeAdapter` (contract mode until keys are provisioned, ADR-008) |
| `Archetype` | saas | Applied | DECISIONS ADR-001; SPEC §6 Design System |
| Design Template | rails_1.html | Applied | DECISIONS "Design Template tokens"; landing port |

Reconciled fields: 34 non-blank → 34 rows. 0 unresolved Conflict.

## Assumptions (blank fields resolved by the agent)

- auth_model → email/password console sessions plus hashed API keys.
- email_service → SMTP (Mailpit locally).
- background_jobs → Solid Queue.
- api_style → REST.
- api_versioning → `/api/v1`.
- orm → Active Record.
- testing → RSpec, Vitest, Playwright.
- ci_cd → GitHub Actions workflow committed.
- backup → `pg_dump` script documented.
- error_reporting → structured logs only.
- data_retention → 30d for idempotency keys and webhook payloads; ledger/audit indefinite.
- i18n → no (en).
- offline → no.
- mdlc_attribution → structural.
- theme → dark default + light toggle.

## Features

- F-01 Platform foundation — 4f81f5c — compose stack, conventions, audit trigger, env schema; checkpointed solo (FOUNDATIONAL).
- F-03 Email outbox — ed0519f — outbox + retry/DLQ job, mailer templates.
- F-02 Identity & sessions — 3ee62b0 — signup/login/reset/sessions; W11 e2e live; review focus GD-0003.
- F-04 Team & roles — aafd492 — invitations, roles, last-owner guard; W8 e2e live; 93 api specs at its checkpoint; review focus GD-0007.
- F-05 API keys & modes — 1752af1 — 22 affected passing (13 added); W1 (full) + W7.1 e2e live; GD-0009 queued (HUMAN_APPROVAL).
- F-06 Customers & payment methods — 9caf8aa — 17 passing (17 added) + 3 log-redaction; W4 e2e live; GD-0012 queued (HUMAN_APPROVAL, payments).
- F-16 Merchant settings & logo — 8295465 — 14 passing (14 added); GD-0013 review focus.
- F-15 Audit & governance evidence — e344cce — 14 passing (11 added); W9 e2e live; GD-0014 review focus.
- F-07 Processor adapters — 38 passing (29 added); full api 178/178; W3.1–W3.2 + W4 e2e live; Stripe [contract] (keys_not_configured); GD queue per hook.
- F-08 Payment intents — 42 passing (40 api + 2 web unit added); full api 218/218; W2 + W3 (full) e2e live, 8/8; INV-2, INV-14..21 pass; ADR-034.
- F-09 Ledger & reporting reads — 2805305 — 18 passing (17 api + 1 web unit added); full api 235/235 (one F-08 expectation updated for populated `ledger_journal_ids`); W5 e2e live + W2 re-run, 2/2; INV-30, INV-31 pass; ADR-035.
- F-10 Refunds & disputes — 88f5189 — 30 passing (26 api + 4 web unit added); full api 261/261; W6 e2e live (3/3 repeat) + full e2e 10/10; INV-22..25 pass; ADR-036.
- F-11 Inbound webhooks & event log — 19 passing added (16 api + 3 web unit); full api 277/277; web unit 40/40; full e2e 10/10 live (W4 toast-race fix, 3/3 repeat); INV-3 and INV-6 pass; ADR-037.
- F-12 Outbound merchant webhooks — 19 passing added (16 api + 3 web unit) and the W7 e2e; full api 293/293; web unit 43/43; full e2e 11/11 live (W7 3/3 repeat); INV-5, INV-10, INV-26..28 pass; ADR-038.
- F-13 Realtime console feed — 13 passing added (8 api channel + 5 web unit) and the W2.3 live-feed e2e; full api 301/301; web unit 48/48; full e2e 11/11 live; SEC-13 and has_websocket ✓; ADR-039 (wave 7 sequential).
- F-14 Dashboard & analytics — 15 passing added (8 api: 5 service + 3 request; 7 web unit) and the W5.3/W5.4 e2e; full api 309/309; web unit 55/55; full e2e 12/12 live; SEC-17 ✓; INV-37 no longer lists /reports; ADR-040.
- F-17 Scheduled operations — 16 passing added (14 job effect specs T3–T6 + 2 ops request); full api 325/325; full e2e 12/12 live (regression); live scheduler loads all 6 recurring tasks; INV-4 now passes; has_scheduled_work ✓; ADR-041 (T6 re-screen from an encrypted pending SHA-1).
- F-18 Platform operator console — 4 api + 4 web unit + 1 e2e added (W10.1–W10.4); full api 329/329, web unit 59/59, e2e 13/13 live; INV-37 now passes (lint 38 / 1, only INV-35 left for the Verification Gate); SEC-19 ✓; ADR-042 (non-operator 404, no PII, audited suspension on the merchant chain).
- F-19 Public site & docs — 13a83f3 — 11 web unit passing (11 added); live: /, /docs, robots, sitemap, manifest, icon, 404; GD-0015 review focus.

## Interface Contract Validation

- **Wave 3 (F-02, F-04, F-05), after F-05:**
  - Full suite: api 106/106 (3 seeds), web unit 19/19, e2e 4/4 live (W1, W7.1, W8, W11). Typecheck, lint, rubocop and prettier are clean.
  - Endpoint coverage: every wave-3 SPEC §3 endpoint is routed. FIXED: an unspecified `PUT /v1/members/:id` alias was removed.
  - Front-end/back-end contract: the team, invite, api-keys, sign-up/in/reset and sessions screens call the literal SPEC paths.
  - Input surface: FIXED: the api-keys screen now sends the user-selected `mode` (the toggle) on list and create.
  - Auth wiring: every S route is `session_only`, and only `invitations#show/accept` are public. `api_keys` refuses K auth (403).
  - Schema alignment: the api_keys, invitations, sessions and memberships models match their migrations.
  - Env completeness: FIXED: `.env.example` gained `DB_STATEMENT_TIMEOUT_MS` plus the test-only `BASE_URL`/`RAISE_ERRORS`.
  - Test isolation: FIXED: an order-dependent route leak in `api_key_auth_spec.rb` (it now uses `draw`/`reload_routes!`).
  - Screens measured: none (the baseline is measured at the Verification Gate).
- **Wave 4 (F-06, F-15, F-16, F-19 — 4 parallel sub-agents, ADR-027), after F-19:**
  - Suite: 149/149 api (seed 9444), 30/30 web unit, 6/6 e2e live (W1, W4, W7.1, W8, W9, W11); rubocop, eslint, tsc, prettier clean. Invariant lint 16 PASS / 23 FAIL, all owned by unbuilt features (no regression).
  - Endpoint coverage: all 15 wave-4 SPEC §3 rows routed and specced (7 customers/payment_methods, 4 merchant/logo, 4 audit/governance); no extra routes (PATCH-only merchant, no PUT alias).
  - FE/BE contract: the screens call literal SPEC paths; INV-37 reports no gaps for customers, governance or settings (only `/settings/processors`, owned by F-07).
  - Auth wiring: merchant, logo, governance, audit and customer redaction are `session_only`; other customer and payment-method routes accept S|K per SPEC §3.
  - Schema alignment: the customers, payment_methods, merchant_assets and evidence_exports models match their migrations (CHECKs, composite tenant FK, immutability trigger).
  - Input surface: PAN guard on customer and payment-method writes; logo magic-byte, size and pixel caps; audit filters validated.
  - Env completeness: no new variables (the logo limit honours `RATE_LIMIT_SCALE`; the card fingerprint reuses `BLIND_INDEX_KEY`).
  - Security read: FIXED: request-log paths leaked query values such as `?email=` (SEC-14), now redacted (ADR-028). FIXED: added an evidence-export throttle of 10/min per session (ADR-030).
  - Design read: the landing page and /docs render under the nonce CSP; JSON-LD carries the nonce; one h1 per page; 360px card layout for tables.
  - Test infra: FIXED: six journeys now exceed the 5/h signup limit in one run, so the local rate-limit reset runs before each signup (`e2e/support/rate-limits.ts`). Remote targets keep the real limit.
- **Wave 5 (F-07 → F-08 → F-09, orchestrator, CRITICAL), after F-09:**
  - Suite: 235/235 api (seed 43627, after one F-08 expectation was updated for populated `ledger_journal_ids`), 33/33 web unit, 9/9 e2e live (W1, W2, W3, W4, W5, W7.1, W8, W9, W11). Rubocop (191 files), eslint, tsc and prettier are clean.
  - Invariant lint: 26 PASS / 13 FAIL (was 16 / 23 at wave 4). Every remaining FAIL is owned by an unbuilt feature (INV-4 F-13; INV-35 Verification Gate; INV-37 refunds, disputes, reports, webhooks, events, sandbox, ops screens and W6/W10). No regression.
  - Endpoint coverage: all 11 wave-5 SPEC §3 rows are routed and specced (2 processor_accounts, 6 payment_intents, 3 ledger), with no extra routes.
  - FE/BE contract: `/settings/processors`, `/payments`, `/payments/new`, `/payments/[id]`, `/ledger` and `/ledger/entries/[id]` call the literal SPEC paths. FIXED: the payment detail's journal links pointed at `/ledger?journal=`; they now open `/ledger/entries/[id]` (SPEC §6).
  - Auth wiring: processor settings and ledger reads are `session_only` (ledger is `:finance`). Payment intents accept S|K, and API keys must send `Idempotency-Key`.
  - Schema alignment: the processor_accounts, payment_intents, payment_attempts, idempotency_keys, risk_signals and ledger models match their migrations (CHECKs, composite tenant FKs, immutability and deferred balance triggers).
  - Input surface: money amounts are integer minor units validated at the boundary (INV-14). The PAN guard covers intent writes. Ledger filters (kind, currency, account `la_` id, dates, mode) are validated with 422.
  - Env completeness: `STRIPE_SECRET_KEY`/`STRIPE_API_VERSION` are documented in `.env.example` (F-07). F-08 and F-09 add no variables.
  - Security read: money POSTs are throttled at 25/s per key (SEC-7, now specced). The raw token is encrypted and never serialised. Ledger rows cannot be updated or deleted, even by the app role (trigger), and merchant FKs on the ledger do not cascade. Note: a ledger accounts read may insert zero-balance default-currency accounts for merchants created before F-09 (idempotent, ADR-035).
  - Design read: one h1 per screen; tables carry captions and `data-label` cells for the 360px card layout; the balance check uses text, not colour alone ("Balanced · Σ debits = Σ credits"); filtered and unfiltered empty states are distinct.
  - Test infra: the full e2e run hit intermittent connection resets and timeouts on the host port, and those requests never reached the API. The cause is Docker Desktop's port forwarding, shared with other local stacks under heavy CPU load. Recreating the edge didn't fix it. With `--retries=2`, the run passed 9/9 with W11 flaky once (its first navigation). This is not an app defect; CI runs inside its own runner.
- **Wave 6 (F-10 → F-11 → F-12, orchestrator, CRITICAL), after F-12:**
  - Suite: 293/293 api, 43/43 web unit, 11/11 e2e live (W1–W9, W11; no retries needed). Rubocop, eslint, tsc and prettier are clean.
  - Invariant lint: 36 PASS / 3 FAIL (was 26 / 13 at wave 5). The remaining FAILs are owned by later work: INV-4 (F-17 scheduled runs), INV-35 (Verification Gate baseline), INV-37 (`/reports` F-14, `/ops` F-18, W10). No regression.
  - Endpoint coverage: all 19 wave-6 SPEC §3 rows are routed and specced (8 refunds/disputes/sandbox, 3 inbound webhook/events, 8 webhook endpoints/deliveries). FIXED: an unspecified `GET /v1/webhook_endpoints/:id` was removed. `/developers/webhooks/[id]` now reads the list, as SPEC §6 lists for that screen; a mode holds at most 16 endpoints, so one page is complete.
  - FE/BE contract: `/refunds`, `/disputes`, `/disputes/[id]`, `/developers/sandbox`, `/developers/events`, `/developers/webhooks` and `/developers/webhooks/[id]` call the literal SPEC paths. The web's `WEBHOOK_EVENT_TYPES` match `WebhookEndpoint::EVENT_TYPES` (from `Payments::Hooks::EVENTS` plus refund/dispute types). Delivery statuses match the `pending_webhooks` CHECK and have STATUS_TONE entries.
  - Plug-in hooks: `Payments::Hooks` now resolves `Ledger::PostPayment` (F-09) and `OutboundWebhooks::Enqueue` (F-12). `Realtime::Broadcast` is still a no-op until F-13.
  - Auth wiring: `POST /v1/webhooks/stripe` is the only new unauthenticated route (signature first, INV-6). Events accept S|K for any role. Endpoints and deliveries are `session_only` for owner/admin/developer. Refund writes are `:refund`, and the sandbox is test-mode only.
  - Schema alignment: the refunds, disputes, webhook_events, webhook_endpoints and pending_webhooks models match their migrations (UNIQUE (source, event_id), UNIQUE idempotency_key, status CHECKs, delivered_at CHECK, cascading endpoint FK).
  - Input surface: endpoint URLs are checked syntactically on save and by address before every attempt (SEC-12). Event-type, status and endpoint filters are validated with 422. Stripe payloads are redacted before storage.
  - Env completeness: `STRIPE_WEBHOOK_SECRET` is in `.env.example` and the env schema. F-12 adds no variables (secrets are per endpoint and encrypted with the existing Active Record encryption keys).
  - Security read: the outbound path is guarded twice (save and attempt), with the IP pinned against rebinding, no redirects, 5 s timeouts, a 64 KB response cap and DNS bounded to 2 s. FIXED during F-12: the unscoped delivery retry update (INV-10), and transport errors referenced ahead of the guard (INV-5).
  - Design read: both webhook screens cover loading, empty, error and success. The secret is shown once in a panel with Copy/Done. Destructive actions use Dialog confirms. Tables have captions and `data-label` cells, and statuses use text plus tone.
  - Test infra: FIXED: the W7 e2e passed alone but failed in the suite, because re-selecting the same status filter never refetched. The deliveries table gained a Refresh control and the spec polls with it (3/3 repeat, then 11/11 suite with no retries).
- **Wave 7 (F-13 → F-14 → F-17, sequential per ADR-039), after F-17:**
  - Suite: 325/325 api, 55/55 web unit, 12/12 e2e live (W1–W9, W11 plus W2.3 live feed and W5.3/W5.4 reports). Rubocop (262 files), eslint, tsc and prettier are clean.
  - Invariant lint: 37 PASS / 2 FAIL (was 36 / 3 at wave 6). INV-4 now passes. The remaining FAILs are owned by later work: INV-35 (Verification Gate baseline) and INV-37 (`/ops` F-18, W10). INV-40 (manual) has an effect spec for every one of T1–T6 in `api/spec/jobs/`. No regression.
  - Endpoint coverage: all 6 wave-7 SPEC §3 rows are routed and specced: `GET /cable`, the three `/v1/reports/*` rows, and `GET`/`POST /v1/ops/scheduled_runs`.
  - FE/BE contract: the dashboard and `/reports` call the literal report paths with `from/to/currency/granularity`, and the CSV download uses the same window. The `/ops` scheduled-runs view is F-18, and `ScheduledRun#as_json` (`job_name, run_key, merchant, trigger, status, counts, error, claimed_at, finished_at`) is the shape it will read.
  - Plug-in hooks: `Realtime::Broadcast` (F-13) is now live and also bumps the report cache version (F-14). T4 expiry goes through `StateMachine` → `Payments::Hooks.changed!`, so an expired intent is audited, gets a `payment_intent.canceled` outbox row (F-12), is broadcast (F-13) and busts reports (F-14), the same path a client cancel takes.
  - Auth wiring: the cable needs the session cookie plus an allow-listed origin and re-checks membership. Reports `view` is all roles and the CSV is `finance`. `/v1/ops/scheduled_runs` is operator-session only, and the trigger is also gated by `OPS_TRIGGER_ENABLED` and refused in production (twice: the env schema at boot and the controller).
  - Schema alignment: `ScheduledRun::JOBS` keys equal the `job_name` CHECK, and `STATUSES` equal the status CHECK. The `cancellation_reason` CHECK adds the system-only `expired`, while `PaymentIntent::CANCELLATION_REASONS` (the API input) is unchanged, which a spec asserts. The users T6 columns carry a CHECK that ties the hash to `breach_check_pending`.
  - Input surface: report windows, currencies and granularities are validated with 422 and capped at 366 / 7 days. The CSV is capped at 50k rows and formula-escaped. `job_name` and `merchant_id` on the ops endpoints are allow-listed or looked up, with 422 otherwise.
  - Env completeness: F-17 adds no variables. `OPS_TRIGGER_ENABLED` (default `false`, forbidden `true` in production) and `HIBP_*` were already in `.env.example` and the env schema.
  - Live: the rebuilt worker's scheduler loads all six recurring tasks (`deliver_pending_webhooks`, `deliver_pending_emails`, `ledger_integrity`, `expire_stale_intents`, `retention_sweep`, `rescreen_passwords`).
  - Test infra: a prettier `--check` failure on `volume-chart.test.tsx` (line endings only; the committed content was unchanged) was re-formatted.

## Pre-Delivery Completeness

- All 19 features are built and committed (F-01 to F-19). Rework items RB-1 to RB-24 are committed, and RB-10 and C-5 were blocked by governance and reverted.
- Every SPEC §3 endpoint is routed and specced, and every SPEC §6 screen is built (INV-37 passes).
- `.env.example` and the env schema list every variable the code reads.

## Verification Gate

| Command | Result | Exit |
|---|---|---|
| `npm run test:api` (d4c6ec6, the last api change) | 341 examples, 0 failures | 0 |
| `docker run … rubocop` | 267 files inspected, no offenses | 0 |
| `npm run test:web` (RB-24) | 17 files, 64/64 passed | 0 |
| `npm --prefix web run lint` / `typecheck` | clean (max-warnings 0) | 0 / 0 |
| `npm --prefix web run test:e2e` (RB-24, Desktop Chrome + Pixel 7) | 26 passed | 0 |
| `npm run web:baseline -- --full` (c84c320, quiet host) | 1 failure (HSTS, accepted under ADR-048), 1 residual | 1 |
| `npm run lint:invariants` (c84c320) | 39/39 machine checks pass | 0 |

Invariant lint summary line, verbatim:

```
40 total / 39 machine (39 pass, 0 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)
```

`.claude/settings.json` was not edited by the build.

## Coverage

API line coverage is 96.33% in total (3382 / 3511 lines), and 97.14% across `app/services`, where payments, ledger, refunds, disputes and webhooks live.

## Web Delivery Baseline

From `FRONTEND-AUDIT.summary.json` (gitSha c84c320, build 55758e51f211, production, 2026-10-21T19:19Z): 30 screens × 375/768/1280 × both palettes, 150 cells, 0 failing.

| Page | Performance (runs) | Accessibility | Best practices | SEO | LCP | CLS |
|---|---|---|---|---|---|---|
| landing | 93 (92 / 95 / 93) | 100 | 100 | 100 | 1781 ms | 0.002 |
| dashboard | 93 (93 / 96 / 93) | 100 | 100 | 63 (noindex by contract) | 1555 ms | 0 |

- Bundles: landing 139 KB and an app screen 150 KB, against a 400 KB budget.
- The only failure is the missing HSTS header on the localhost edge, accepted under ADR-048.
- Runs on a contended host scored landing 83–84. ADR-054 records the investigation and the quiet-host resolution.

## Reviewer Gate

Rounds 1–3 failed and were fixed (RB-1 to RB-5). Rounds 4–8 passed; round 8 reviewed RB-23 with three advisories (A1–A3, ADR-055). Reviewer M2 is carried under ADR-051.

## Security Audit

Pass 3 verdict: **PASS**, with no CRITICAL, HIGH or MEDIUM finding open. One LOW is deferred (SA-4, the 72-byte bcrypt input) and two LOWs are residuals (N3 webhook sweep fairness, N4 idempotent-resume edges). Gitleaks' six findings are the same known false positives (template and docs example keys, contract-test fixtures).

## COMPLIANCE

Posture B: 62 ✓ / 7 ⚠ / 0 ✗ (ADR-053). Every ⚠ is a recorded residual: HSTS on localhost (four rows), SA-4, N3 and N4. The web-delivery rows now cite the quiet-host baseline on c84c320.

## Design Quality Gate

Four independent re-scores moved the 30 screens from 5 ✓ / 21 ⚠ / 4 ✗ to 27 ✓ / 3 ⚠ / 0 ✗ (`artifacts/design/score-4.md`). The three ⚠ are artifacts of the pinned capture runner (docs@1280, ledger@375, governance@375). ADR-055 carries them with the reviewer advisories. DESIGN-NOTES.md records the outcomes and the performance lessons.

## Load test

`artifacts/loadtest/RESULTS.md`: the sustained profile **FAILS** its §9 latency targets on this single-node local host (p95 of seconds against 50/250 ms targets, because VUs queue on the api threads). The error rate was 0%, and the replay mix produced **0** duplicate charges. The service-time step (4 VUs) is the comparable number. Capacity work is a residual.

## Residuals carried

- HSTS absent on the localhost edge (ADR-048). The local certificate is not trusted by the host store, which was not modified.
- SA-4, N3 and N4 (LOW). Reviewer M2 (ADR-051) and r8 advisories A1–A3 (ADR-055).
- Load test sustained FAIL (latency only; 0% errors, 0 duplicate charges).
- IP-country risk signal is a header-derived caveat on localhost. Stripe runs in contract mode without live keys.
- Dashboard SEO 63 (noindex by contract). Design artifact-class ⚠ ×3. Bulk sign-out is a spec note.
- Landing performance margin: 93 on a quiet host, 83–84 on a contended one (ADR-054).
- One flaky e2e: the drawer Escape step needs a retry on a loaded host.

## Reconciliation

The Build Input Reconciliation rows still hold at ship. The HTTPS-only transport is applied by the Caddy `edge` service in `compose.yaml` (publishes `${HTTPS_PORT:-9443}:443` and redirects `:80`), and every check below went through it. The drift gate (`npm run lint:invariants` on HEAD after RB-25) reports 39/39 machine checks passing, exit 0. RB-25 changed documentation and audit outputs only.

## Stage 4

| Step | Command | Result | Exit |
|---|---|---|---|
| Rebuild from HEAD | `GIT_COMMIT=$(git rev-parse --short HEAD) docker compose up -d --build api worker` | api and worker healthy; web kept at the measured c84c320 build (same source) | 0 |
| Drift gate | `npm run lint:invariants` | 39/39 machine checks pass | 0 |
| Governance | `node scripts/govern.mjs queue --check` | 0 awaiting human approval | 0 |
| Smoke | `sh smoke-test.sh` | 53/53 passed (`smoke-test.log`) | 0 |

## Startup verification

`docker compose ps`: api, db, db-replica, edge, mail, redis, web and worker are all `healthy`. `GET https://localhost:9443/api/health` returned:

```json
{"status":"ok","version":"1.0.1","commit":"0a8587b","checks":{"db":"ok","db_replica":"ok","redis":"ok","queue":"ok"}}
```

## Smoke Test Results

`sh smoke-test.sh` against `https://localhost:9443`: **53/53 passed**, exit 0. It covers sign-up and sign-in, payments with `Idempotency-Key`, refunds, the ledger, the inbound Stripe webhook contract, password reset without enumeration, the worker delivering mail to Mailpit, 17 KB request headers, customer redaction and logout. The full log is `smoke-test.log`.

## Deploy Target Reachability

**Outcome C** (`deploy: local`): no remote target is configured, so the local Compose stack is the shipped artifact. Its published ports (9443, 9080, 9025) are owned by this stack.

| Check | Command | Result | Exit |
|---|---|---|---|
| Edge root | `curl -sk -o /dev/null -w "%{http_code}" https://localhost:9443/` | 200 | 0 |
| Health | `curl -sk https://localhost:9443/api/health` | `status: ok`, commit 0a8587b | 0 |

The edge serves a `tls internal` certificate, so the probes accept it for localhost only. A browser needs the one-time CA trust in QUICKSTART.md. The host trust store was not modified by the build.

## Governance

75 decision record(s) over 49 change(s), policy 1.4 (`GOVERNANCE-DECISIONS.jsonl`).

| Final outcome | Changes |
|---|---|
| AUTO_APPROVE | 35 |
| REVIEW | 12 |
| REWORK | 0 |
| HUMAN_APPROVAL | 0 |
| BLOCK | 2 |

**Human decisions:** F-01 approved (GD-0056); F-05 approved (GD-0057); F-06 approved (GD-0058); F-07 approved (GD-0059); F-08 approved (GD-0060); F-09 approved (GD-0061); F-10 approved (GD-0062); F-11 approved (GD-0063); F-12 approved (GD-0064); F-13 approved (GD-0065); F-19 approved (GD-0066); RB-1 approved (GD-0067); RB-17 approved (GD-0068); RB-2 approved (GD-0069); RB-20 approved (GD-0070); RB-3 approved (GD-0071); RB-4 approved (GD-0072); RB-5 approved (GD-0073); RB-9 approved (GD-0074).
**Escalated to a human:** F-01 (minimum for project dependencies+infrastructure (planned in F-01)); F-05 (minimum for project authentication); F-05 (HUMAN_APPROVAL from GD-0009 stands (minimum for project api+application (planned in F-05))); F-05 (HUMAN_APPROVAL from GD-0010 stands (minimum for module build-record+documentation+tooling)); F-06 (minimum for project payments (planned in F-06)); F-19 (minimum for module security-boundary); F-07 (minimum for project payments); F-08 (minimum for project payments); F-09 (minimum for project payments); F-09 (HUMAN_APPROVAL from GD-0019 stands (minimum for file tests (planned in F-09))); F-09 (HUMAN_APPROVAL from GD-0020 stands (minimum for file build-record+documentation)); F-10 (minimum for project payments); F-11 (minimum for project payments); F-12 (minimum for project payments); F-12 (minimum for project payments); F-13 (minimum for project payments); RB-1 (minimum for project dependencies+infrastructure+unknown); RB-2 (minimum for project authentication+dependencies+infrastructure+payments); RB-3 (minimum for project architecture+authentication+infrastructure+payments); RB-4 (minimum for project authentication); RB-5 (minimum for service authentication); RB-9 (minimum for service payments); RB-17 (minimum for service authentication); RB-20 (minimum for service payments).
**Blocked:** RB-10 (policy: kernel application+tooling); C-5 (policy: kernel application).

## Usage

Measured from the Claude Code session transcripts that built this project (main session + subagents), by `scripts/usage-report.mjs`. Tokens are what the model processed; on a subscription plan this is what draws the usage limits, and on the API it is what is billed.

| Metric | Value |
|---|---|
| Sessions / subagents | 1 / 21 |
| Assistant turns | 7026 (5397 main) |
| Cache-read tokens | 774,804,912 (main 624.9M, subagents 149.9M) |
| Cache-write tokens | 22,876,311 |
| Output tokens | 5,618,320 |
| Uncached input tokens | 14,190 |
| Context per turn (main) | avg 119K, peak 168K, 0 turns over 400K |
| Compactions | 42 |
| Active time | 20 h (15 pauses totalling 73.3 h; span 2026-10-17T22:15:47.503Z to 2026-10-21T19:33:51.842Z) |
| Models | claude-opus-5-5 x7025, <synthetic> x1 |

Per hour (UTC): cache-read / output / turns

| Hour | Cache-read | Output | Turns |
|---|---|---|---|
| 2026-10-17T22Z | 29.3M | 604K | 260 |
| 2026-10-17T23Z | 38.8M | 750K | 320 |
| 2026-10-18T00Z | 58.5M | 432K | 580 |
| 2026-10-18T01Z | 44.0M | 464K | 380 |
| 2026-10-18T02Z | 17.5M | 187K | 164 |
| 2026-10-18T03Z | 44.9M | 437K | 385 |
| 2026-10-18T04Z | 34.2M | 408K | 304 |
| 2026-10-18T05Z | 47.0M | 420K | 403 |
| 2026-10-18T06Z | 69.2M | 310K | 587 |
| 2026-10-18T07Z | 63.6M | 221K | 572 |
| 2026-10-18T08Z | 74.8M | 272K | 704 |
| 2026-10-18T09Z | 27.6M | 105K | 236 |
| 2026-10-18T12Z | 988K | 7K | 11 |
| 2026-10-18T13Z | 5.2M | 27K | 44 |
| 2026-10-18T14Z | 1.8M | 11K | 13 |
| 2026-10-18T21Z | 17.0M | 76K | 158 |
| 2026-10-18T22Z | 694K | 4K | 7 |
| 2026-10-18T23Z | 12.2M | 67K | 100 |
| 2026-10-19T00Z | 6.8M | 42K | 56 |
| 2026-10-19T18Z | 338K | 3K | 6 |
| 2026-10-20T00Z | 2.0M | 12K | 24 |
| 2026-10-20T01Z | 1.7M | 11K | 17 |
| 2026-10-20T10Z | 11.1M | 20K | 102 |
| 2026-10-20T11Z | 12.7M | 64K | 120 |
| 2026-10-20T13Z | 2.6M | 8K | 22 |
| 2026-10-20T14Z | 1.1M | 5K | 8 |
| 2026-10-20T15Z | 6.7M | 53K | 58 |
| 2026-10-20T16Z | 6.0M | 41K | 52 |
| 2026-10-20T18Z | 17.2M | 100K | 152 |
| 2026-10-20T19Z | 62.3M | 159K | 619 |
| 2026-10-20T20Z | 29.0M | 139K | 295 |
| 2026-10-20T21Z | 12.8M | 82K | 136 |
| 2026-10-21T00Z | 4.3M | 14K | 30 |
| 2026-10-21T01Z | 3.6M | 19K | 25 |
| 2026-10-21T10Z | 654K | 4K | 10 |
| 2026-10-21T11Z | 467K | 2K | 6 |
| 2026-10-21T19Z | 6.0M | 37K | 60 |
