# ARCHITECTURE.md — STΛRSHΞILD

Project e5214c3a-f03e-4631-8cff-5ab54215fa87 · Stage 1 · version 0.1.0 · build_depth comprehensive · scale tier **large**

> I AM // STΛRSHΞILD — INTELLIGENCE IN MOTION™ — governed by MDLC.ai.
> This is a vendor-neutral cyber-defense **control plane**. It offers two portal experiences, COMMAND and TACTICAL, over one API, one data model and one governance loop.

---

## 1. System overview

```
             browser (COMMAND / TACTICAL portals)
                         │  https://  (HTTP :80 → 301 → :443)
                ┌────────▼────────┐
                │  Caddy 2 edge   │  tls internal (local) / ACME (prod) · HSTS (non-localhost)
                │                 │  strips x-middleware-subrequest · security headers · lb
                └───┬─────────┬───┘
         /api/*     │         │   everything else
          ┌─────────▼──┐   ┌──▼────────────┐
          │ api ×N     │   │ web ×N        │  Next.js 15 App Router (standalone)
          │ Express 5  │   │ CSP nonce MW  │  Redux Toolkit + RTK Query (10 s polling)
          └──┬───┬───┬─┘   └───────────────┘
             │   │   │ enqueue (BullMQ)
             │   │   └──────────────┐
     ┌───────▼┐ ┌▼───────┐   ┌──────▼─────┐
     │pgbouncer│ │Redis 7 │◄──┤ worker ×N  │  BullMQ 5 consumers + job schedulers
     └───┬────┘ └────────┘   └──────┬─────┘
     ┌───▼──────────┐                │
     │ PostgreSQL 16│◄───────────────┘   (read client → replica when DATABASE_READ_URL set)
     └──────────────┘
     Prometheus ── scrapes api:/metrics, worker:/metrics ── Grafana dashboards
```

Process roles:

| Process | Role | Scaling | Stateful? |
|---|---|---|---|
| `caddy` | TLS edge, reverse proxy, header policy, round-robin LB across api/web replicas | 1 (prod: L4 LB in front) | certs volume only |
| `web` | Next.js SSR shell and client app | horizontal | no |
| `api` | REST `/api/v1/*`, auth, RBAC, governance policy decisions, ingest acceptance | horizontal (`--scale api=N`) | no; sessions and limits live in Redis/PG |
| `worker` | BullMQ consumers (ingest normalize, correlate, validate, verify) and scheduled jobs | horizontal; claims via DB | no |
| `migrate` | one-shot `prisma migrate deploy` + seed | once per deploy | — |
| `postgres` | system of record; evidence ledger | primary (+ read replica in prod) | yes |
| `pgbouncer` | transaction pooling for N api/worker replicas | 1 | no |
| `redis` | sessions cache, rate-limit counters, BullMQ, replay cache, API cache | 1 (prod: managed/HA) | ephemeral + AOF |
| `prometheus` / `grafana` | metrics and dashboards (`monitoring: Prometheus + Grafana`) | 1 | yes (volumes) |

## 2. Layers and repository layout

The repo is an npm workspaces monorepo. There is one release version and a `version-agreement` invariant (INV-6).

```
package.json                 root, workspaces [apps/*, packages/*], version = release
packages/shared/             @starshield/shared — Zod schemas, enums, state-machine tables, RFC7807 types,
                             QAI scoring (pure), geo truncation (pure). No I/O.
apps/api/                    @starshield/api — Express 5 + Prisma 6
  prisma/schema.prisma       data model; prisma/migrations/** forward migrations (+ down.sql per migration)
  src/config/env.ts          Zod env schema, fail-fast (Service Floor §2)
  src/app.ts                 express app factory (middleware order is fixed, §4)
  src/server.ts              API entry: listen + graceful shutdown
  src/worker.ts              worker entry: BullMQ workers + upsertJobScheduler registrations
  src/routes/*.ts            HTTP layer: parse (Zod) → authorize → call service → shape response
  src/services/*.ts          business logic; the ONLY layer that touches Prisma (INV-1)
  src/jobs/*.ts              scheduled/queued job bodies; call services, take an injected clock
  src/lib/*.ts               logger, errors (problem+json), crypto, metrics, rate limiters, request context
  src/db/client.ts           Prisma clients (write + read), pgbouncer-aware
apps/web/                    @starshield/web — Next.js 15 App Router
  src/middleware.ts          per-request CSP nonce, strict-dynamic; auth redirect
  src/app/**                 routes (COMMAND, TACTICAL, cases, governance, ledger, admin, …)
  src/store/**               Redux Toolkit store factory (per request) + RTK Query api slice (only fetch site, INV-14)
  src/components/**          ported template primitives (shell, panels, KPI, gauge, radar, graph, timeline)
deploy/Caddyfile             TLS edge; deploy/prometheus.yml; deploy/grafana/**
compose.yaml                 full stack; Dockerfile (multi-stage, per target)
load/                        k6 scripts and results
scripts/                     pinned kernel runners (verbatim) + smoke-test.sh, backup.sh
```

Each layer may only call the layer below it:

| Layer | May call | May not call |
|---|---|---|
| web components | RTK Query hooks / store | `fetch` directly (INV-14), server internals |
| routes | services, lib | Prisma (INV-1) |
| jobs | services, lib, injected clock | Prisma (INV-1), `Date.now()` for due-time decisions |
| services | db client, lib, shared | express `req`/`res` |
| shared | nothing with I/O | — |

## 3. Domain model (summary; SPEC.md §2 is authoritative)

The model is multi-tenant. Every tenant row carries `orgId`. A user joins an org through a `Membership` that holds a role.

| Entity | Purpose | Key constraints |
|---|---|---|
| Org, User, Membership(role) | tenancy + identity | unique(orgId,userId) |
| Session | opaque server session (hashed token) | token hash unique; idle 30 min / absolute 12 h |
| Source | ingest source with HMAC secret (stored encrypted, shown once) | unique(orgId, name) |
| IngestNonce | replay cache backing store | unique(sourceId, nonce) |
| Event | normalized telemetry (CSF-mapped category, severity, geo lat/lng truncated to 2 dp, `synthetic` flag) | CHECK lat ∈ [-90,90], lng ∈ [-180,180]; index (orgId, observedAt) |
| Asset (kind host/identity/service/network/cloud) | inventory linked to events, with criticality; identities are assets of kind `identity` (Stage 2) | unique(orgId, externalId) |
| IngestBatch, Notification | async ingest receipt (queued → processed / failed); in-app notifications (unique org, kind, subject) | Stage 2 additions |
| Alert | correlated detection; status `new → linked | suppressed` | |
| Incident (case) | `open → triaged → contained → eradicated → recovered → closed`; each state maps to a NIST CSF 2.0 function; SLA due timestamps | |
| AttackPath | hypothesis graph nodes and edges per incident | |
| Proposal | QAI-ranked response option with an explainable score breakdown; lifecycle §5 | |
| Validation | independent validator verdict + check list (policy version, blast radius, window, reversibility) | one per proposal revision |
| Approval | approver decision | **unique(proposalId, approverId)** — the same person cannot count twice |
| ActionToken | JIT signed single-use token (≤ 15 min, one action, one target) | unique jti; `usedAt` |
| ActionExecution | dispatched action + post-action verification result, rollback ref | |
| Policy | versioned policy-as-code (JSON rules) | unique(orgId, version) |
| EvidenceEntry | append-only hash-chained ledger | **unique(orgId, seq)**; trigger blocks UPDATE/DELETE |
| AuditLog | access/admin audit trail (`audit_logging: true`) | |
| JobRun | per-job, per-window run record + outcome counts | **unique(jobKey, windowStart)** = idempotent claim |
| PostureSnapshot | per-org KPI snapshot (posture radar, threat level) | unique(orgId, takenAt) |

## 4. Request pipeline (api), in a fixed order

1. `requestContext`: accepts an inbound `x-request-id` or mints a UUID, echoes it, and creates a child pino logger.
2. `pino-http` access log. It records the route pattern, not the raw path. Redaction covers authorization, cookie, set-cookie, password, token, secret, email and apiKey.
3. `helmet`-equivalent API headers and `Cache-Control: no-store` on authenticated responses.
4. Body parse with a 1 MB limit, or 5 MB on ingest. The raw body is retained for HMAC.
5. Global edge rate limit: express-rate-limit + rate-limit-redis, keyed by IP.
6. `session`: resolves an opaque cookie to a session through a Redis cache, falling back to Postgres. It sets `req.actor` = {userId, orgId, role}.
7. `csrf`: a double-submit token on every non-GET request authenticated by cookie. Ingest is exempt because it is HMAC-authenticated with no cookie.
8. Route-level: a Zod parse, then `authorize(permission)`, then a per-route cost-based limiter where defined, then the service call.
9. `errorHandler`: RFC 7807 `application/problem+json` with `code`, `requestId` and `errors[]`. It never leaks stacks. A foreign-tenant resource returns **404**.

## 5. Governance loop (the core pattern)

```
 Alert/Incident ──► QAI propose ──► Validate (independent) ──► Approve ──► Execute (JIT token) ──► Verify ──► Close
                    proposed         validated | validation_failed  approved | rejected  executed | execution_failed  verified | rolled_back
                                                                     └─ expired (token/approval window lapsed; scheduled job)
```

- **Propose.** The QAI engine (`packages/shared/src/qai.ts`, pure and deterministic, ADR-015) scores candidate actions from the playbook catalog: score = w₁·severity + w₂·assetCriticality + w₃·confidence − w₄·blastRadius − w₅·irreversibility. Every component and weight is stored and shown in the UI. Read-only actions (enrich, tag, notify) are `impact: low`; containment actions (isolate host, disable identity, block IP) are `impact: high`.
- **Validate.** This is a separate service module (`services/validator.ts`) that runs in the **worker** process from a queue, never in the request that created the proposal. It checks:
  - Evidence provenance: every referenced event exists, belongs to the org, and is hash-anchored in the ledger.
  - Policy: the current policy version allows the action type for the target's criticality.
  - Blast radius ≤ the policy max.
  - The change window is open.
  - Reversibility: a rollback is defined for high impact.
  - Approval requirement: 1 approver for low impact, 2 for high impact.

  The verdict is persisted with the policy version.
- **Approve.** `proposer ≠ approver` is enforced in the service (`assertNotProposer`, INV-10) and at the DB level by unique(proposalId, approverId). High impact requires 2 distinct approvers, each holding the `approver` permission. A rejection is terminal.
- **Execute.** Executing mints an ActionToken: an HMAC-SHA256 JWT-like compact token with claims {jti, orgId, proposalId, action, target, exp ≤ 15 min}. The executor verifies the token (`verifyActionToken`), marks it `usedAt` atomically (`UPDATE … WHERE used_at IS NULL`), and only then dispatches (INV-9). v1 dispatches to a **simulated connector adapter** that records the intended change and synthetic effect, badged SIMULATED. Vendor connectors are out of scope by Non-Goal.
- **Verify / rollback.** The `post-action-verification` job runs after `verifyAfter`. It independently re-reads telemetry for the target, checks the expected signal (e.g. no further events from the isolated host), and records `verified`. Otherwise it records `rolled_back`, executing the rollback through the same token path, and escalates the incident.
- **Ledger.** Every transition appends an EvidenceEntry with `{seq, prevHash, payloadHash, hash = SHA-256(prevHash ∥ canonicalJSON(payload))}` inside the **same transaction** as the business write (INV-3/INV-12). `GET /api/v1/ledger/verify` recomputes the chain.

## 6. Ingest (signed generic API)

- `POST /api/v1/ingest/events` takes `X-Starshield-Source`, `X-Starshield-Timestamp`, `X-Starshield-Nonce` and `X-Starshield-Signature: v1=<hex HMAC-SHA256(secret, ts + "." + nonce + "." + rawBody)>`.
- The request is rejected (401) on clock skew > 5 min, a bad signature (constant-time compare), or a replay (Redis `SET NX` 10 min, backed by IngestNonce unique). Verification happens **before** any DB read of business data (INV-8). Only the source lookup by id is allowed first.
- An accepted batch of ≤ 500 events is enqueued (`ingest` queue) and returns 202 with a batch id. The worker normalizes events (Zod), truncates geo, maps CSF categories, upserts assets (identities are assets of kind `identity`; discovery is set-based and runs before the batch transaction, ADR-062), and correlates them into alerts. A batch still `queued` after 10 min with no job receipt is failed by the watchdog (Service Floor §1).
- **Synthetic connector.** The `synthetic-feed` scheduled job (enabled by `SYNTHETIC_FEED=1`) emits a realistic event stream per demo org with `synthetic: true`. Every synthetic datum renders with a `SYNTHETIC` badge (Non-Goal: illustrative data never presented as factual).

## 7. Scheduled work (has_scheduled_work)

Every job takes `now: Date` (injected clock) and exposes `runXForOrg(orgId, now)` next to the all-tenant sweep. The schedule is evaluated in **UTC**.

Claiming is done in two ways:
- Per window, `INSERT INTO job_runs (job_key, window_start) … ON CONFLICT DO NOTHING`, enforced by unique(jobKey, windowStart).
- Per item, a conditional `UPDATE … WHERE claimed_at IS NULL`.

Each run records `processed / skipped / failed` in JobRun, and the API exposes them in Admin → Jobs and `/api/health` `checks.worker`.

| Job key | Schedule (BullMQ `upsertJobScheduler`) | Business effect (asserted by its effect test) |
|---|---|---|
| `sla-escalation` | every 60 s | incidents past `slaDueAt` in `open/triaged` get `escalatedAt` + a ledger entry + a notification row, exactly once |
| `proposal-expiry` | every 60 s | `validated/approved` proposals past `expiresAt` → `expired`; unused tokens revoked |
| `post-action-verification` | every 60 s | `executed` actions past `verifyAfter` → `verified` or `rolled_back` (+ incident escalation) |
| `retention-sweep` | daily 02:00 UTC | events/alerts older than the org retention (default 400 d) deleted; ledger untouched |
| `ledger-integrity` | hourly | chain verified per org; a broken chain sets `org.ledgerStatus = 'broken'` + critical alert |
| `posture-snapshot` | every 15 min | a PostureSnapshot row per org (KPIs, radar, threat level) |
| `ingest-watchdog` | every 60 s | ingest batches stuck `queued` > 10 min → `failed` with reason |
| `synthetic-feed` | every 30 s (if enabled) | synthetic events inserted for demo orgs, flagged `synthetic` |
| `outbound-drain` | every 30 s | due `pending/failed` outbound messages pass the outbound validator, then send (email) or POST (signed webhook); outcome, attempts and checks recorded; terminal outcomes ledgered, each message delivered at most once (F-23) |
| `alert-escalation` | every 60 s | `new` real alerts older than a policy's `escalate_after_min` get `escalation_level + 1`, `escalated_at`, `alert.escalated` messages, an in-app notification and a ledger entry, exactly once per level (F-23) |
| `report-generate` | hourly | enabled report schedules whose `next_run_at` is due get one report for the period ending at their latest slot (daily: 24 h, weekly: 7 d), a `report.created` ledger entry and one `report.ready` email per recipient who is still an active member; `next_run_at` advances by compare-and-set, so a slot is generated at most once (F-26) |

- **Startup.** On boot, the worker runs a bounded catch-up: windows missed within the last 24 h run once, and older ones are skipped and logged.
- **Shutdown.** `worker.close()` lets the in-flight job finish within 10 s, then requeues it.
- **Test trigger.** `POST /api/v1/admin/jobs/:key/run` is mounted only when `NODE_ENV !== 'production' && ENABLE_TEST_TRIGGERS === '1'` and requires an admin session (INV-15). The smoke test uses it to assert effects.

## 8. Cross-cutting concerns

- **AuthN.** Email/password with Argon2id (m=19 MiB, t=2, p=1, the OWASP baseline) and an NIST 800-63B verifier:
  - Minimum length 12, maximum 128, no composition rules.
  - HIBP range API with `Add-Padding: true`, failing open with a logged warning.
  - The limiter runs before the hash compare (INV-11).
  - Session rotation on login.
  - Anti-enumeration: uniform responses and timing.
- **AuthZ.** RBAC roles are `admin`, `responder`, `approver`, `analyst`, `auditor` and `viewer`. Each maps to a permission set in `packages/shared/src/rbac.ts`. ABAC is the tenant scope: every service function takes `actor` and filters `where: { orgId: actor.orgId }` (manual INV-M2, plus adversarial tenant tests).
- **Rate limiting** (`rate_limiting: true`) has four layers:
  - Edge (Caddy request size and timeouts).
  - Global per-IP limit: 600/min.
  - Per-route cost limits. Login is 5/15 min per (ip, email) and 20/15 min per IP. Ingest is 120 batches/min per source. Export is 5/hour per user. Mutation routes are 120/min per user.
  - BullMQ queue concurrency caps.

  All limits are backed by Redis so they hold across replicas.
- **Audit logging** (`audit_logging: true`) has two parts. The AuditLog records every auth event, admin change, export and permission denial. The EvidenceEntry ledger records every governance transition. Both are viewable and exportable (CSV/JSON) by `auditor`.
- **Observability** (`monitoring: Prometheus + Grafana`, `logging_format: structured JSON`):
  - pino JSON logs with requestId.
  - prom-client: HTTP histograms by route, queue depth, job outcome counters and ledger append latency.
  - Grafana dashboard JSON provisioned from `deploy/grafana/`.
- **Caching** (large tier):
  - Redis read-through cache for COMMAND KPIs (posture snapshot, 15 s TTL).
  - Next.js static assets are immutable and content-hashed, so a CDN can sit in front of Caddy.
  - `Cache-Control: no-store` on API data.
- **Read scaling.** `db.read` uses `DATABASE_READ_URL` when set (replica), else the primary. It is used only for list/analytics reads that tolerate lag.
- **Security headers** (web, via middleware + Caddy):
  - CSP `default-src 'self'; script-src 'self' 'nonce-…' 'strict-dynamic'; style-src 'self' 'nonce-…'; img-src 'self' data:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'`.
  - X-Content-Type-Options, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy and COOP same-origin.
  - HSTS max-age=31536000 on non-localhost.
- **Theming.** Theme is auto with a dark default (ADR-003). The choice is persisted in a cookie and applied server-side (no flash). Tokens come from DECISIONS.md.

## 9. Architectural Invariants

Machine form: `invariants.json`. Each id is the cross-reference. The runner is the pinned `scripts/invariant-lint.mjs` (v1.7).

| ID | Rule | Check |
|---|---|---|
| INV-1 | Prisma model access only in `apps/api/src/services/**` and `apps/api/src/db/**` (routes/jobs call services); the pattern lists every Prisma model, including `indicator` and `ipNetwork` (ADR-068) and `rule` and `ruleVersion` (ADR-069) and `notifyPolicy`, `deliveryEndpoint`, `outboundMessage` (ADR-071) and `vulnerability`, `vulnFinding`, `reportSchedule`, `report` (ADR-074) and `alertCopilot` (ADR-075) | forbidden-pattern |
| INV-2 | No raw-unsafe SQL anywhere (`$queryRawUnsafe` / `$executeRawUnsafe`) | forbidden-pattern |
| INV-3 | EvidenceEntry writes only via `services/ledger.ts` (append-only through one door) | forbidden-pattern |
| INV-4 | Ledger trigger blocks UPDATE/DELETE on `evidence_entries` | required-pattern (migrations) |
| INV-5 | Ledger sequence unique per org | required-unique-constraint evidence_entries(org_id, seq) |
| INV-6 | One release version across workspaces, lockfile and compose `APP_VERSION` | version-agreement |
| INV-7 | Same approver cannot approve a proposal twice | required-unique-constraint approvals(proposal_id, approver_id) |
| INV-8 | Ingest signature verified before events are accepted | boundary-order in routes/ingest.ts |
| INV-9 | Action token verified before any dispatch | boundary-order in services/execution.ts |
| INV-10 | proposer≠approver asserted before an approval row is written | boundary-order in services/approvals.ts |
| INV-11 | Login limiter before password verification | boundary-order in routes/auth.ts |
| INV-12 | Job claim idempotent per window | required-unique-constraint job_runs(job_key, window_start) |
| INV-13 | Ingest replay protected | required-unique-constraint ingest_nonces(source_id, nonce) |
| INV-14 | Web client network calls only through the RTK Query slice / server-fetch helper | forbidden-pattern `fetch(` in apps/web/src |
| INV-15 | Test-trigger router mounted only behind the `ENABLE_TEST_TRIGGERS` guard | boundary-order in app.ts |
| INV-16 | No `console.log` in api source (pino only) | forbidden-pattern |
| INV-17 | `dangerouslySetInnerHTML` only in the JSON-LD component | forbidden-pattern |
| INV-18 | Caddy strips `x-middleware-subrequest` (CVE-2025-29927) | required-pattern deploy/Caddyfile |
| INV-19 | Caddy redirects HTTP→HTTPS and sets HSTS | required-pattern deploy/Caddyfile |
| INV-20 | Geo bounds CHECK constraint in migrations | required-pattern |
| INV-21 | Health route exists | required-file apps/api/src/routes/health.ts |
| INV-22 | `.env.example` exists | required-file |
| INV-23 | Env schema module exists | required-file apps/api/src/config/env.ts |
| INV-24 | Web baseline runner pinned | required-file scripts/web-baseline.mjs |
| INV-25 | Web Delivery Baseline report passes and is fresh (accepted: command/lighthouse ADR-059, landing/lighthouse ADR-070 -- NO_NAVSTART-zeroed runs only; every other check still gates) | web-baseline |
| INV-26 | Every SPEC §6 screen / endpoint / control / §5 workflow is covered by UI + e2e | ui-coverage |
| INV-27 | Governance ledger audit | governance |
| INV-28..33 | Incident statuses `open, triaged, contained, eradicated, recovered, closed` are each written by a production writer | required-pattern ×6 |
| INV-34..43 | Proposal statuses `proposed, validated, validation_failed, approved, rejected, executed, execution_failed, verified, rolled_back, expired` are each written | required-pattern ×10 |
| INV-44..46 | Alert statuses `new, linked, suppressed` are each written | required-pattern ×3 |
| INV-47 | Passwords hashed with Argon2id (argon2 lib used in the password module) | required-pattern apps/api/src/lib/password.ts |
| INV-48 | HIBP check uses only the range API prefix (k-anonymity) | required-pattern `api.pwnedpasswords.com/range/` |
| INV-49 | Outbound email and webhook egress (nodemailer, https / `request(` clients) only in `apps/api/src/lib/outbound/**`, which `services/outbound.ts` calls only after the outbound validator passed (ADR-071) | forbidden-pattern |
| INV-50 | Model egress (vendor SDK imports, STARGAZER `/v1/route/structured`, raw `/v1/messages` or `/v1/chat/completions` paths) only in `apps/api/src/lib/copilot/**`, which `services/copilot.ts` calls only after the copilot egress validator passed (ADR-075) | forbidden-pattern |
| INV-51 | The live WebSocket hub (`lib/live/hub.ts`) checks the upgrade `Origin` against `PUBLIC_ORIGIN` before it resolves the session cookie, so a cross-site page cannot open a socket with the user's cookie (ADR-076) | boundary-order |
| INV-M1 | (manual) Each of the 11 declared scheduled triggers has an effect test | manual |
| INV-M2 | (manual) Every tenant-data service query is scoped by `actor.orgId`; foreign ids → 404 | manual |

There are 2 manual entries out of 53 (≤ 3 and ≤ 20%).

## 10. Threat model (STRIDE; extends RESEARCH §5)

| Threat | Vector | Mitigation | Evidence |
|---|---|---|---|
| **S**poofing — forged telemetry | attacker posts events to ingest | per-source HMAC over ts+nonce+body, 5 min skew, replay cache | INV-8, INV-13, ingest tests |
| Spoofing — session theft | XSS / cookie theft | httpOnly Secure SameSite=Lax cookie, CSP nonce + strict-dynamic, session rotation | INV-17, headers in FRONTEND-AUDIT |
| Spoofing — middleware bypass | CVE-2025-29927 `x-middleware-subrequest` | Caddy strips the header; Next ≥ 15.2.3 | INV-18 |
| **T**ampering — evidence edits | DB admin or app bug rewrites history | hash chain + DB trigger blocking UPDATE/DELETE + verify endpoint + hourly integrity job | INV-3/4/5, ledger tests |
| Tampering — CSRF | cross-site mutation | double-submit CSRF token + SameSite | csrf tests |
| **R**epudiation | approver denies approving | approval rows bound to session user and ledger entry with actor id | INV-7, ledger |
| **I**nfo disclosure — cross-tenant | IDOR via ids | every query scoped by `orgId`; foreign → 404; adversarial tests | INV-M2 |
| Info disclosure — precise location | geo PII | 2-dp truncation (~1.1 km), bounds CHECK | INV-20 |
| Info disclosure — logs | secrets in logs | pino redaction list; no console.log | INV-16 |
| **D**oS — floods | login / ingest / export abuse | Redis-backed per-dimension limiters, body limits, queue concurrency, k6-measured headroom | rate-limit tests, load report |
| **E**levation — self-approval | proposer approves own action | assertNotProposer + unique approver + 2-person rule for high impact | INV-10, INV-7 |
| Elevation — token reuse | replaying an action token | single-use jti, `used_at` conditional update, ≤ 15 min exp | INV-9, execution tests |
| Elevation — LLM/AI overreach | automation self-approves | QAI is deterministic and propose-only; it has no execute permission; execute requires a human approval path | Non-Goals, RBAC tests |
| Elevation — test trigger in prod | job trigger endpoint exposed | mounted only non-prod + flag + admin | INV-15 |

## 11. Key decisions and alternatives considered

| Decision | Chosen | Alternatives considered | Why |
|---|---|---|---|
| API framework | Express 5 (RESEARCH constraint) | Fastify, NestJS | constraint; Express 5 has native async error propagation |
| Frontend state | Redux Toolkit + RTK Query (constraint) | Zustand + TanStack Query | constraint; RTK Query gives cache, polling and invalidation tags in one place |
| Realtime | 10 s polling | WebSocket/SSE | no `realtime_needed`; polling is cacheable, simpler to scale and avoids has_websocket surface (ADR-018) |
| Queue | BullMQ 5 on Redis (constraint) | pg-boss, Temporal | constraint; `upsertJobScheduler` gives idempotent repeatable schedules |
| Idempotent jobs | DB claim table + conditional updates | BullMQ jobId dedupe only | Redis dedupe is lost on flush; the DB claim gives proof by unique constraint |
| Ledger integrity | SHA-256 hash chain + trigger in PG | external transparency log, QLDB-like store | self-contained, verifiable offline, no extra infra; external anchoring is a v2 option |
| Action credentials | platform-signed JIT tokens | HashiCorp Vault dynamic secrets | v1 has no real vendor connectors; Vault deferred (Step 3b fix #6) |
| Sessions | opaque tokens in PG + Redis cache | stateless JWT | instant revocation, rotation, admin "sign out everywhere" |
| TLS | Caddy `tls internal` locally, ACME in prod | nginx + mkcert | automatic certs, HTTP→HTTPS redirect, simple header config |
| Map | self-hosted globe image + local projection | Mapbox/Google | no billing-abuse proxy, no third-party requests under strict CSP (ADR-016) |
| QAI | deterministic weighted explainable model | LLM ranking | explainability, reproducible evaluations; honours the no-unproven-claims Non-Goal (ADR-015) |
| Pooling | pgbouncer transaction mode | Prisma Accelerate | self-hosted; `pgbouncer=true` param for Prisma |

## 12. Scale tier — large (50k+ concurrent)

| Concern | Mechanism |
|---|---|
| Read replicas | `db.read` client on `DATABASE_READ_URL` (compose provides a primary; replica is a prod overlay, documented) |
| Cache + CDN | Redis KPI cache; immutable hashed static assets, `Cache-Control: public, max-age=31536000, immutable` |
| Autoscaled workers + DLQ | stateless worker replicas; BullMQ `attempts: 5`, exponential backoff, failed jobs retained → `dlq` queue + metric |
| Horizontal stateless deploy | api/web/worker hold no local state; Caddy `lb_policy round_robin` over `api:4000` replicas via DNS |
| Edge rate limit + cost controls | per-IP global + per-route cost limiters in Redis; ingest batch cap 500; export quota |
| Sustained load + headroom | k6 `constant-arrival-rate` against the read-heavy COMMAND poll mix. The target is 50k concurrent users polling every 10 s, which is ≈ 5,000 req/s. The single-host test measures achievable RPS at p95 < 500 ms and extrapolates replicas; the gap to a full 50k proof is residual R1 (ADR-017) |

## 13. Risks (top, see RESEARCH §5)

| Risk | Detail | Mitigation |
|---|---|---|
| R1 | A single dev host cannot prove 50k concurrent users | Measured RPS + headroom is reported, and the gap is carried as a residual |
| R2 | Ledger trigger vs Prisma migrations | The trigger lives in raw SQL in the migration and is tested by an UPDATE attempt |
| R3 | CSP strict-dynamic with Next.js | Nonce middleware + `dynamic` rendering; measured by the web baseline |
| R4 | Clock skew on HMAC | 5 min window; the server time is returned in the 401 problem detail |
| R5 | Polling load | 10 s interval, Redis cache, `If-None-Match` ETags on KPI endpoints |
