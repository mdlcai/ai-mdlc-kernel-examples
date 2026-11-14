# REPORT.md — STΛRSHΞILD

Project e5214c3a-f03e-4631-8cff-5ab54215fa87 · build_depth comprehensive · review_gates auto

## Build Input Reconciliation

Rows whose evidence is a future deliverable are **provisional (P)**. They are re-verified at the Pre-Delivery Completeness Gate and flip to ⚠ if the deliverable is missing.

| Field | Value | Disposition | Evidence |
|---|---|---|---|
| protocol_support | HTTPS only | Applied (P) | compose `caddy` service :443 `tls internal`, :80 → 301 https (`deploy/Caddyfile`); INV-19; QUICKSTART "Protocol & TLS" |
| monitoring | Prometheus + Grafana | Applied (P) | compose `prometheus` :9090, `grafana` :3001; `deploy/prometheus.yml`; api/worker `/metrics` |
| database_preference | PostgreSQL | Applied (P) | compose `postgres` (16) + `pgbouncer`; `apps/api/prisma/schema.prisma` |
| rate_limiting | true | Applied (P) | ARCHITECTURE §8 four layers; express-rate-limit + rate-limit-redis; INV-11 |
| audit_logging | true | Applied (P) | AuditLog model + EvidenceEntry ledger; INV-3/4/5 |
| frontend_framework | Next.js | Applied (P) | `apps/web` Next.js 15 App Router |
| state_management | Redux | Applied (P) | Redux Toolkit + RTK Query, `apps/web/src/store/**`; INV-14 |
| backend_framework | Express | Applied (P) | `apps/api` Express 5 |
| api_style | REST | Applied (P) | `/api/v1/*`, `openapi.json` |
| orm_preference | Prisma | Applied (P) | Prisma 6; INV-1 |
| background_jobs | BullMQ | Applied (P) | `apps/api/src/worker.ts`, `upsertJobScheduler`; INV-12 |
| performance_requirements | sustained load test at 50k+ concurrent users | Partial ⚠ (P) | k6 constant-arrival-rate in `load/`; single-host measurement + headroom extrapolation; a full 50k proof on one host is residual R1 (ADR-017) |
| logging_format | structured JSON | Applied (P) | pino JSON + requestId; INV-16 |
| scale | large — 50k+ concurrent | Applied (P) with residual R1 | ARCHITECTURE §12 (replica read client, Redis cache, stateless replicas, DLQ, edge + cost limits) |
| multi_tenant | true | Applied (P) | orgId on every tenant row; INV-M2; tenant-isolation tests |
| target_platforms | web | Applied (P) | responsive web; FRONTEND-AUDIT.json at 375/768/1280 |
| domain_signals | has_geo, has_scheduled_work | Applied (P) | SPEC §5 checklists; INV-20 (geo bounds); INV-12, INV-M1 (jobs) |
| auth_model (Step 3b) | email/password + server sessions | Applied (P) | ADR-006; INV-11/47/48 |
| jit_credentials (Step 3b) | signed action tokens ≤ 15 min | Applied (P) | ARCHITECTURE §5; INV-9 |
| deployment (Step 3b) | Docker Compose behind Caddy | Applied (P) | `compose.yaml`, `Dockerfile` |
| build_depth | comprehensive | Applied | scope expansion ADRs (Stage 2); Lighthouse in web baseline |
| review_gates | auto | Applied | no approval halts; BUILD-STATE.md |
| domain | CYBERSECURITY | Applied | ARCHITECTURE threat model §10 |
| Design Template | Claude Design export | Applied (P) | DECISIONS.md tokens; ADR-001..003; Template Conformance at the Design gate |
| Accessibility | WCAG AA, Lighthouse 95+, breakpoints 640/768/1024/1280, standard motion | Applied (P) | will cite `FRONTEND-AUDIT.json` (Web Delivery Baseline) |
| security_baseline (blank → default) | OWASP Top 10 + ASVS 5.0 L2 subset + 800-63B verifier | Applied (P) | ADR-007; COMPLIANCE.md (Stage 3) |

## Assumptions (Stage 3 pre-build)

- Stack resolved per ADR-004..018: Node 22, Express 5, Prisma 6, Next.js 15, Redux Toolkit/RTK Query, BullMQ 5, Postgres 16 + pgbouncer, Redis 7, Caddy tls internal.
- Auth: email/password, server sessions in Postgres, CSRF token; NIST 800-63B verifier (HIBP fail-open ADR-023); recovery via admin reset (ADR-027).
- Response actions run against a SIMULATED adapter (ADR-022); synthetic telemetry badged SYNTHETIC; no external map provider.
- Identity folded into Asset (ADR-025); QAI candidates not persisted (ADR-026).
- Scale: large tier proven by single-host k6 measurement + extrapolation (residual R1).
- Plan: 19 features in 11 waves (BUILD-PLAN.md); estimate 18 h; t_max 2026-11-09T03:15:00Z.

## Invariant lint

Stage 1 `--prove`: `50 total / 48 machine (48 proven, 0 unproven) / 2 manual (skipped -- Reviewer Gate item 8)`

Wide close (on 33e4742): `50 total / 48 machine (48 pass, 0 fail, 0 zero-files) / 2 manual (skipped -- Reviewer Gate item 8)`. INV-25 is carried by one `accepted` entry (command/lighthouse, ADR-059). The manual invariants were prose-audited for the drift gate: INV-M1 UPHELD and INV-M2 UPHELD (see Drift Detection Gate).

## Web Baseline

Definitive run `npm run web:baseline -- --full` on 33e4742 (rebuild 8): 20 screens × 5 cells = 100 cells, 0 failing cells, site checks all pass, and 1 failure:

| Screen | Lighthouse runs (perf / bp) | Median | Status |
|---|---|---|---|
| landing | 91/100, 0/0, 95/100 | pass | ✓ |
| command | 93/100, 0/0, 80/100 | perf 80 < 85 | accepted (ADR-059) |

The 0/0 runs are NO_NAVSTART runtime errors caused by COOP (ADR-059), and they count as 0 in the median. The command clean runs average 86.5, but one clean run scored 80, so command performance is marginal. It is a v0.2 residual (ADR-061). The residual "SEO 58 not scored" is by contract: authenticated screens are noindex. An earlier run (#3) failed `buildFreshness`, because the 0.1.2 pin bump postdated BUILD_ID. Fixed by rebuilding and restarting.

**Merge waves 1–2 (2026-11-10, ADR-070):** the definitive `--full` run on the wave 2 build covered 24 screens (120 cells): 0 failing cells and all site checks passing. Landing then returned NO_NAVSTART on all three runs (0/0 ×3) under the pinned channel `chromium`. A clean headless run, after the feature CSS moved to `app/(app)/layout.tsx`, scored landing performance 87 / 86 / 88 and best practices 100. landing/lighthouse is carried next to command per ADR-070; it is a residual, not a pass.

## Verification Gate

Run against production builds (`next start` :3400, API :4000, worker :4001) on Postgres :55432 and Redis :56479.

| Check | Command | Result |
|---|---|---|
| Format | `npm run format:check` | exit 0 |
| Lint (type-checked, jsx-a11y, react-hooks, max-warnings 0) | `npm run lint` | exit 0 |
| Typecheck | `npm run typecheck` | exit 0 |
| Build | `npm run build` | exit 0 |
| Unit + integration + coverage | `npm run test:coverage` | exit 0: 57 files, 253/253 tests; statements 96.29 %, branches 85.26 %, functions 92.03 %, lines 96.29 % (threshold 80 %) |
| E2E (desktop + mobile projects) | `E2E_WORKERS=2 npx playwright test` | exit 0: 82/82 passed (3.0 m) |
| Web baseline (failing set) | `npm run web:baseline -- --failing` | exit 0: 275 → 100 → 2 → 0 failing cells over five runs (artifacts/logs/W12-baseline-*.log) |
| Invariants | `node scripts/invariant-lint.mjs` | 47/48 machine pass. INV-25 is pending only because the root FRONTEND-AUDIT.json is written by the `--full` run at the Review Round wide close. INV-M1 and INV-M2 are manual (Reviewer). |

The root causes and fixes are recorded in ADR-058. The partial report (`--failing`) skips the site-level, bundle and Lighthouse checks. Those are measured by the definitive `--full` run.

## Reviewer Gate

**PASS after 3 rounds.** Mechanism: a Claude Code Task tool subagent (general-purpose) with fresh context in each round. Rounds 2 and later are diff-scoped.

| Round | Commit reviewed | Verdict | File |
|---|---|---|---|
| 1 | 631d000 | FAIL (validator confidence calibration, export scope, and others; fixed in ADR-059) | artifacts/reviewer/round-1.md |
| 2 | 3697f5b | FAIL (items 2 and F-12, the smoke sla-escalation flow; item 4; item 9; fixed in ADR-060) | artifacts/reviewer/round-2.md, round-2.diff |
| 3 (settle check) | 54327f3 | **PASS**. Item 9 was UNCERTAIN, then re-judged PASS after the W8 scoped-export e2e (e2e/w8.spec.ts:64-80). The SPEC minors are amended. | artifacts/reviewer/round-3.md, round-3.diff |

`govern.mjs resolve-review --evidence reviewer=artifacts/reviewer/round-3.md` resolved 26 REVIEW items. No REVIEW items remain pending.

## Security Audit Gate

**PASS WITH ADVISORY** (SECURITY-AUDIT.md). The audit ran in 3 passes: initial scan, auto-remediation, and re-scan.

| Pass 1 findings | Auto-remediated | Pass 2 residual (C/H/M) | LOW accepted | INFO |
|---|---|---|---|---|
| 19 (0 C / 0 H / 3 M / 10 L / 6 I). The re-scan added 5 more, for 24 catalogued. | 10 (SEC-1, 2, 3, 7, 9, 12, 20, 21, 22, 24), including all 4 MEDIUMs | **0 / 0 / 0** | 7 (SEC-4, 5, 6, 8, 10, 11, 13), plus the advisory residual of no edge rate limit | 7 |

## COMPLIANCE.md

See COMPLIANCE.md. Posture: **21 ✓ / 7 ⚠ / 0 ✗**.
- The ⚠ items are S-2 (SEC-5), S-4 (SEC-11), T-1 (SEC-6), D-1 (no edge limit), production hygiene (SEC-4, 8, 10, 13) and NIST-1 (HIBP fail-open, ADR-007).
- Every web-delivery row cites the definitive FRONTEND-AUDIT.summary.json.
- The posture is logged in ADR-061.

## Design Quality Gate

**Conforms. 13 ✓ / 7 ⚠ / 0 ✗ over 20 screens** (final re-score, artifacts/design/score-3.md). Score history: 2/5/13, then 8/10/2, then 13/7/0.
- The residuals R1–R7 are recorded as accepted polish concerns in DESIGN-NOTES.md (ADR-060). None of them is an anti-slop item or a floor failure.
- Template Conformance: the token diff is 38/38.
- After score 3, the only frontend change was the source-list sort order (newest first), which changes no layout. The definitive `--full` run refreshed every screenshot.

## Pre-Delivery Completeness Gate

All items pass.
- **Routes:** every SPEC §3 route has a handler. The contract check reports 53/55, and the remaining 2 are intended exclusions (internal test trigger and metrics). There are no TODO, FIXME or not-implemented stubs.
- **Data layer:** there is one Prisma migration plus a reversible down script, `apps/api/prisma/down/20261107000000_init.down.sql`. Its round trip was verified on a scratch database: 26 tables up, then 1 down, then 26 back up (ADR-060). Seed data covers the demo org, roles and users.
- **Validation:** inputs are validated with zod at the API and in the shared form schemas, and errors are returned as RFC 7807 problems.
- **CRUD, auth and UI:** the RBAC matrix, sessions, CSRF and admin reset are covered by integration tests. All 20 SPEC screens are routable with loading, empty, error and success states. Every Key Workflow is covered by e2e: 82/82.
- **Env and ops:** the following are present:
  - `.env.example`, matching the env schema
  - Dockerfile, compose.yaml and Caddyfile
  - CI, .editorconfig, ESLint and Prettier
  - invariants.json and web-baseline.config.json
  - QUICKSTART.md, with a Protocol & TLS section
  - README "Built with", BUILD-PLAN.md, and the ARCHITECTURE threat model and alternatives

## Stage 4 — Deploy (local)

**Deployed:**
- Docker Compose behind Caddy `tls internal`, at **https://localhost:8543**.
- The api, worker and web images are built from **574eb33**, with `org.opencontainers.image.revision` labels and `/api/health` reporting the commit.
- Release 0.1.2.

**Startup and configuration:**
- Health takes 308 ms on a cold start and 72 ms warm.
- With `SESSION_SECRET` unset, the API exits 78, and the message names the variable.
- The global limiter returns 429 at request 601 of 600/min.
- The QUICKSTART curls (login, overview, signed ingest) are verified against the deploy.

**Functional smoke:** `bash smoke-test.sh` returns **SMOKE PASS** (artifacts/logs/smoke-final.log, smoke-test.log). The checks:
- trusted-origin TLS with `--cacert` (no `-k`)
- health
- auth bootstrap
- source CRUD
- signed ingest with worker job, forged-signature 401 and replay idempotency
- cross-tenant isolation
- RBAC viewer
- ledger export
- SLA escalation by the scheduled worker
- 16 KB cookie
- worker liveness
- no stranded jobs
- provenance for all four tiers plus the edge
- graceful shutdown: the in-flight request is answered and the API exits 0 in 3 s
- fail-fast

**Smoke fixes (574eb33, C-29, Reviewer Round 5 PASS):**
- **Graceful shutdown was a race.** The old `/api/health` probe either finished before SIGTERM or hit the closed listener. The smoke check now holds a throttled request body in flight.
- **That exposed a real drain defect.** Caddy's keep-alive sockets kept `server.close()` pending until the 10 s force-close, so shutdown took 11 s. The drain now reaps idle sockets every 100 ms.
- **The health probe now retries.** It makes up to 3 readiness probes, and still requires `ok` within 500 ms.

**Load (SPEC §9).** k6 1.3.0 against one API process on the shared 4-vCPU Docker VM. Details are in ADR-062, ADR-063, ADR-064 and ADR-065, and the logs are in artifacts/logs/load-*.log.

| Run | Offered | Achieved | Failed | Finding |
|---|---|---|---|---|
| 1 | reads + ingest | 179.6 rps | 1.78 % (500s) | asset upsert locks inside the batch transaction → ADR-062 |
| 2 | reads + ingest | 67.7 rps | 20.2 % | post-enqueue UPDATE blocked behind the worker's row lock → ADR-063 |
| 3 | reads + ingest | 124 rps | 7 % | lock contention (C-26 notes fixed) |
| 4 | SPEC mix 200 rps + ingest | 38.7 rps | 49.95 % | overview cache-expiry stampede drained the pool (5,770 P2024/60 s) → ADR-064 single flight |
| knee 60 | SPEC reads 60 rps, 60 s | 60 rps | 0.00 % | p95 0.56–0.84 s per route |
| knee 200 | SPEC reads 200 rps, 60 s | 128–137 rps | 0.00–0.05 % | event-loop bound |
| final | SPEC mix 200 rps + ingest 1×50/s, 10 min | 93.8 rps | 16.27 % | client-side timeouts only; server 0 P2024, no app 5xx; success p95 5.58 s / p99 11.97 s |

**SPEC §9 targets are NOT met on this host:**
- overview p95 < 150 ms
- list and geo p95 < 250 ms
- ingest p95 < 300 ms
- error rate < 0.5 % at 200 rps

There is no headroom to 5,000 RPS. ADR-065 records this as residual **R1**, a single-host ceiling: one Node process on a contended VM, with event-loop lag up to 7 s at 25–40 % CPU. The path to the target is API replicas behind Caddy on dedicated hosts, re-measured with `scripts/load/run.sh`.

**Web baseline:**
- **Full run** (`npm run web:baseline -- --full`, local production servers): 1 failure, the `command` Lighthouse cell, which is carried and accepted under ADR-059.
- **INV-25** is fresh again, and `npm run lint:invariants` reports **48/48 machine invariants PASS**.
- **Site-only run against the live deploy** (Playwright 1.65 container, https://localhost:8543 with the Caddy CA trusted; artifacts/web-baseline/FRONTEND-AUDIT.site.json, auxiliary evidence): all checks pass except two expected cells:
  - **HSTS is absent on localhost by design (INV-19).** Caddy sends it only for a real domain, so local browsers don't pin localhost.
  - **Sign-in and register meta wasn't collected**, because the site-only pass doesn't load those screens. They pass in the full run.

**Reviewer:**
- **Round 4 (49b688a) PASS:** C-26 and C-27, minor findings only.
- **Round 5 (574eb33) PASS:** C-29, minor findings only.
- `resolve-review` resolved C-26, C-27 and C-29.

**Governance:** the 10 HUMAN_APPROVAL items were answered by the human ("approve all", GD-0082–GD-0091); `govern queue --check` now exits 0.

**Deploy-readiness:**
- **Ready for local and single-tenant evaluation.** Every functional, security, isolation and ops smoke check passes on the deployed images, with commit provenance verified on every tier.
- **Durable under failure.** Graceful drain and fail-fast configuration are proven. No ingest batch is stranded, and replay is idempotent. The evidence ledger is written in the business transaction.
- **Not ready for SPEC §9 production scale on one host (R1).** Plan horizontal API replicas and dedicated hosts before production load.
- **Open before production:**
  - the ADR-061 residuals and WEB-4
  - a real domain in `CADDY_SITE`, which brings ACME certificates and HSTS

## Usage

Measured from the Claude Code session transcripts that built this project (main session + subagents), by `scripts/usage-report.mjs`. Tokens are what the model processed; on a subscription plan this is what draws the usage limits, and on the API it is what is billed.

| Metric | Value |
|---|---|
| Sessions / subagents | 2 / 29 |
| Assistant turns | 7218 (4681 main) |
| Cache-read tokens | 808,933,395 (main 566.3M, subagents 242.7M) |
| Cache-write tokens | 21,512,917 |
| Output tokens | 6,114,281 |
| Uncached input tokens | 15,210 |
| Context per turn (main) | avg 123K, peak 167K, 0 turns over 400K |
| Compactions | 34 |
| Active time | 15.7 h (1 pauses totalling 0.7 h; span 2026-11-07T05:33:25.292Z to 2026-11-07T21:53:31.489Z) |
| Models | claude-opus-5-5 x7218 |

Per hour (UTC): cache-read / output / turns

| Hour | Cache-read | Output | Turns |
|---|---|---|---|
| 2026-11-07T05Z | 69K | 256 | 2 |
| 2026-11-07T06Z | 34.7M | 643K | 317 |
| 2026-11-07T07Z | 62.0M | 748K | 523 |
| 2026-11-07T08Z | 92.6M | 761K | 868 |
| 2026-11-07T09Z | 63.6M | 477K | 548 |
| 2026-11-07T10Z | 29.9M | 230K | 263 |
| 2026-11-07T11Z | 86.0M | 782K | 799 |
| 2026-11-07T12Z | 41.4M | 226K | 347 |
| 2026-11-07T13Z | 133.8M | 810K | 1237 |
| 2026-11-07T14Z | 23.1M | 97K | 175 |
| 2026-11-07T15Z | 24.7M | 129K | 206 |
| 2026-11-07T16Z | 95.9M | 499K | 854 |
| 2026-11-07T17Z | 56.9M | 305K | 532 |
| 2026-11-07T18Z | 20.8M | 119K | 163 |
| 2026-11-07T19Z | 14.9M | 92K | 120 |
| 2026-11-07T20Z | 16.0M | 111K | 143 |
| 2026-11-07T21Z | 12.5M | 85K | 121 |

## Governance

91 decision record(s) over 49 change(s), policy 1.5 (`GOVERNANCE-DECISIONS.jsonl`).

| Final outcome | Changes |
|---|---|
| AUTO_APPROVE | 49 |
| REVIEW | 0 |
| REWORK | 0 |
| HUMAN_APPROVAL | 0 |
| BLOCK | 0 |

**Human decisions:** C-17 approved (GD-0082); C-19 approved (GD-0083); C-21 approved (GD-0084); C-22 approved (GD-0085); C-23 approved (GD-0086); C-25 approved (GD-0087); C-5 approved (GD-0088); F-1 approved (GD-0089); F-2 approved (GD-0090); F-4 approved (GD-0091).
**Escalated to a human:** F-1 (minimum for project dependencies+infrastructure); F-2 (minimum for project authentication+security-boundary); F-4 (minimum for project security-boundary); F-4 (HUMAN_APPROVAL from GD-0006 stands (minimum for project api)); C-5 (minimum for file unknown); C-17 (minimum for project infrastructure); C-19 (minimum for project authentication+dependencies+infrastructure); C-21 (minimum for project architecture+authentication+dependencies+infrastructure+security-boundary); C-22 (minimum for project architecture+authentication+dependencies+infrastructure); C-23 (minimum for project architecture); C-25 (minimum for project architecture).
**Blocked:** none.
**Ledger head:** `e0dc01a59e120ee146c6c9e00874d9c8b3ad479df45b079d2710339b46b79f7f` after 91 record(s). Keep it outside this repository; `node scripts/govern.mjs verify e0dc01a59e120ee1` proves a later ledger still contains this one.
