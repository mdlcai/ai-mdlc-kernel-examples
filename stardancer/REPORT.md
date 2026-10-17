# REPORT.md — STΛRDANCΞR

Project `aede2d71-e1fb-4ef5-9067-ed24f2bd9c5a` · build_depth comprehensive · review_gates auto · scale tier **large**

## Build Input Reconciliation

Every non-blank RESEARCH.md field with its disposition; where each is applied is in ARCHITECTURE.md and DECISIONS.md (ADR numbers there).

| # | Field | Disposition |
|---|---|---|
| 1 | domain_signals.has_geo | Applied |
| 2 | domain_signals.has_scheduled_work | Applied |
| 3 | domain_signals.has_payments | Applied |
| 4 | protocol_support | Applied |
| 5 | monitoring | Applied |
| 6 | database_preference | Applied (local stand-in) |
| 7 | rate_limiting | Applied |
| 8 | audit_logging | Applied |
| 9 | frontend_framework | Applied |
| 10 | state_management | Applied |
| 11 | backend_framework | Applied |
| 12 | api_style | Applied |
| 13 | orm_preference | Applied |
| 14 | realtime_needed | Applied |
| 15 | background_jobs | Applied |
| 16 | performance_requirements | Applied (residual) |
| 17 | logging_format | Applied |
| 18 | scale | Applied |
| 19 | multi_tenant | Applied |
| 20 | target_platforms | Applied |
| 21 | design.archetype | Applied |
| 22 | design.palette (light) | Conflict → template wins |
| 23 | design.theme | Applied |
| 24 | design.typography | Applied |
| 25 | design.type scale | Applied |
| 26 | design.layout | Applied |
| 27 | design.breakpoints | Applied |
| 28 | design.radius | Conflict → template wins |
| 29 | design.shadows | Applied |
| 30 | design template | Applied |

**Reconciled fields: 30** (27 Applied, 2 Conflict resolved by precedence, 0 Deferred, 1 Applied-with-residual counted in Applied).

Signal bumps: `has_payments` + `scale: large` → `risk: high` on billing and ingest features; `multi_tenant` → tenant-isolation invariant on every service; `has_scheduled_work` → job-claim uniqueness.

## Assumptions summary
- No cloud, Neon, Stripe or email credentials are available → local compose deploy, Postgres stand-in, simulated billing mode, admin-delivered links (ADR-006/008/011/012).
- No LLM provider → deterministic explainable Mirror Brain (ADR-010).
- Named ecosystem producers integrate through one generic ingest API (ADR-019).

## Features
18 features in build order (F-01 … F-18), 263 tests at CP-8; per-feature counts in BUILD-STATE history and `artifacts/reviewer/`.

## Interface Contract Validation
- CP-1…CP-8: tsc/lint/prettier ✓ at each checkpoint, shared zod DTOs, schema diff 0, only specified routes; CP-8 263/263, server statements 86.7 %, invariants 55/56, e2e W1, W2, W4–W7, W9, W11, W12, W16 ✓ desktop + mobile.

## Pre-Delivery Completeness Gate
- PASS. Endpoints: 89 in SPEC, all routed and bound in web source (INV-56); no TODO/stub. Migrations: one `init`, forward-only with a backup rollback (ADR-052). Seeds: control catalogue (`migrate`), demo via `seed-demo.mjs` / Overview.
- Validation shared zod on both sides; CRUD and auth flows complete; 33/33 §6 screens with empty/loading/error states and in `web-baseline.config.json`.
- `.env.example` covers every variable read (17); QUICKSTART, README (commands + Built with), NOTICE, BUILD-PLAN (18 manifests), ci.yml (now with prettier), `.editorconfig`, eslint/prettier configs, Dockerfile COPY sets, `invariants.json` + `lint:invariants` present.

## Verification Gate
- PASS. `npm ci` 0 (run #1 EPERM from an orphaned `next dev`, stopped by the human) · build · compose up · typecheck · eslint · prettier 0 · `npm test` 270/270, server statements 86.66%.
- web:baseline: Caddy CA trust + fixture (ADR-053); edge bound to loopback (WSL `[::1]` stall); run #3's 213 failing cells in 15 root causes fixed in 0.1.1–0.1.3 (ADR-056) → 0. HSTS on since ADR-065.
- e2e #1 48/52 → 52/52 after root-cause fixes; W1/W16 wait on queue state, not wall-clock (0.1.4). ci.yml runs stack → baseline → e2e → invariant lint.

## Reviewer Gate
- R1 (b13b14f) FAIL: items 1/5/6/9 + F-12. R2 (d6b3b07) PASS, none surviving. R3 settle (2c1c18e, focus C-11) PASS; note R3-N1 (scan cap 200k rows, ADR-058). R4 (662174e, v0.1.7) PASS; findings fixed in ADR-070. `artifacts/reviewer/`.

## Security Audit Gate
- Pass 1 findings: 10 (0C/1H/3M/6L) · Auto-remediated: 7 · Pass 2 residual: 0C/0H/0M/7L, deferred with reasons. See SECURITY-AUDIT.md.

## Design Quality Gate
- Posture: A ✓ 13 / B ⚠ 18 / C ✗ 2 over 33 screens (round 3, fresh context, 572f7b9; rounds 1→2→3: 5/23/5 → 14/14/5 → 13/18/2).
- ✓ landing and overview: Lighthouse perf ≥ 80 since the ADR-068 speed pass (Stage 4 run 23: 83/81; run 22: 82/77). Earlier: (round 3: 72/67; Stage 4 run 21: 72/71 after ADR-067, run 20: 70/70 after ADR-066, run 19: 70/46, run 18: 69/47, run 17: 75/68) on a contended host (ADR-060). Template conformant (`--dim` ADR-003, Manrope ADR-004). DESIGN-NOTES.md.


## Stage 4 — Ship

### v0.1.7 follow-up (`da6bc53`)

**Verdict: PASS — shipped.** C-44 and C-45 approved by the human (GD-0108, GD-0109).
- **Reviewer round 4** (`artifacts/reviewer/round-4.md`, 662174e against 2c1c18e): PASS. The five pending reviews
  (C-20/24/28/31/38) were resolved with the reviewer evidence, so 0 reviews are pending. Its findings are fixed in ADR-070:
  - `serverGet` forwards the client's X-Forwarded-For, so server-rendered reads no longer share one per-IP bucket;
  - SystemOrb no longer prefetches;
  - the proxy accepts `HSTS_ENABLED=1` as well as `true`, the same values as the API;
  - the demo form's zod import moved inside the try.
- **Demo generation** (ADR-069): the cause was Prisma `createManyAndReturn` in `acceptBatch` (14–21 s for 7.7k rows).
  It is now one `jsonb_to_recordset` parameter insert, and the demo POST takes 3.2 s (was 11–15 s).
  Graph reads on a calm host for 2,000 entities: cold 120–170 ms, cached 106–265 ms (target 400), impact depth 6 and
  attack paths 0.2–0.5 s (target 800). The load suite's red rows are 4 shared Docker CPUs saturating.
- **Tests:** 569 pass (server 217, shared 7, web 345).
  - Web statement coverage went from 65.8% to 92.6% (branches 85.9%, functions 91.8%, lines 93.6%).
  - Writing the new tests surfaced two bugs, both fixed: a missing catch in the Topbar sign-out, and the evidence "Clear" button not resetting its inputs.
- **Smoke** at `da6bc53`: 52/52 (`smoke-test.log`); `/api/health` reports `0.1.7`/`da6bc53`.
- **web:baseline** (`--full`): PASS, 0 failures, 1 residual (overview SEO, noindex by contract). Landing 91
  (LCP 1.4 s, CLS 0) and overview 81 (LCP 1.3 s). One earlier full run had a single 375 px overflow cell on
  investigation-detail that did not reproduce; the `--failing` re-run and two later full runs pass.
- **Invariant lint:** `57 total / 56 machine (56 pass, 0 fail, 0 zero-files) / 1 manual`.
- **Releasing:** versions 0.1.7, including the `@stardancer/shared` workspace pins (the cc28150 trap).
  - Repeated smoke and audit runs within an hour exhaust the `signup-ip` (5/h) and `reset-ip` (10/15 min) limiters.
  - Their 429s are the limiter working as designed, not a regression; let the windows expire between full runs.

### v0.1.6

**Verdict: PASS — shipped.** Build `ef37e7f` (v0.1.6; Mission Control ADR-067, speed pass ADR-068).
web:baseline run 23 passes (landing 83, overview 81). The stale INV-55 `site/headers` acceptance (ADR-023, obsolete
since HSTS ADR-065) was removed on the human's "approve all" (e7dfc80), and C-36 was approved (GD-0079). Tagged `v0.1.6` at `662174e`.

- **Startup:** sequential build (QUICKSTART "compose build tag race") → `docker compose up -d --wait` exit 0;
  `/api/health` `{"status":"ok","version":"0.1.6","commit":"ef37e7f"}`; OCI revision labels on api/worker/web agree.
- **Smoke** (`./smoke-test.sh` → `smoke-test.log`, exit 0, **52/52**): API 33/33, UI field errors, missing env,
  Postgres outage 503 → 200, 16.5 KB cookie, worker never restarts, SIGTERM with a request in flight (200, exit 0
  in 6 s), optional variables absent/empty, provenance, OS-trust curl and Chromium (`ignoreHTTPSErrors:false`),
  site-only audit. Runs 1–2 found a worker crash on a Postgres restart, pool starvation, drain order, empty optional
  variables and an unheadered sign-in redirect; fixed in `db76d35`/`0d8fbea` (ADR-064).
- **HSTS** on (human decision, ADR-065): 1-day `max-age` on loopback, one year on a public host.
- **Tests:** server 217/217, web 90/90; typecheck, eslint, prettier clean.
- **Invariant lint:** `57 total / 56 machine (56 pass, 0 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`
- **web:baseline run 23** (`--full`, gitSha `ef37e7f`, buildFreshness current): **PASS, 0 failures**, 1 residual
  (overview SEO not scored: noindex by contract). landing 83 (LCP 1.6 s, CLS 0); overview 81 (LCP 1.6 s, CLS 0.001).
  ADR-068 speed pass: Latin-only next/font (preloaded, `optional`, no `local()` fallback faces, whose first layout cost 0.5–1 s),
  no `content-visibility` on the landing, zod (≈100 KB) off the landing and overview, sidebar and overview links
  without prefetch, one cached date formatter. Run 22 (`2ce0b43`, before the overview addendum): 82/77.
- **web:baseline run 21** (`--full`, gitSha `d44beba`, buildFreshness current): FAIL, 2 failures, performance score
  only. landing 72; overview 71 (LCP 1.9–2.0 s). The Mission Control layout and the static // LIGHT spheres (ADR-067)
  cost nothing measurable. Run 20 (`e7e75f9`): landing 70 (LCP 2.0 s); overview 70 (LCP 2.0 s, CLS 0.045). Every LCP, CLS, a11y, best-practices and SEO gate passes.
  Run 19 (after the human freed the host to 15 containers) measured 70/46. Its overview LCP of 6.6 s was a real
  defect: the content rendered only after hydration plus a client fetch. ADR-066 fixes it by server-rendering the
  first snapshot. The remaining gap is TBT: about 1.5–2.2 s of hydration script evaluation and font-swap relayouts
  under 4× mobile CPU throttling on a shared 8-CPU host. ADR-063 (motion after load + idle) also shipped.

**Deploy Target Reachability — Outcome C (local).** `docker compose up -d --wait` → 0; `curl https://localhost:8643/api/health`
(OS trust store) → 200; Chromium with TLS verification → 200. URL: `https://localhost:8643`.

`── Drift scan complete (0 drifts) ── Auto-proceeding to deploy.`

### Load (`artifacts/load/load-report.json`, `limit-proof.json`)
SPEC §9 parameters; 2,000-entity dataset; `RATE_LIMIT_SCALE` raised for the run, then restored to 1 and proven
(attempt 21 on login-ip → 429). autocannon p97.5 vs the p95 target. Measured at `db9eba7`, before ADR-064's pool of 20.

| Target (p95) | c=10 p97.5 | c=50 p97.5 | Verdict |
|---|---|---|---|
| health ≤ 50 ms | 250 | 1,601 | red |
| overview ≤ 150 | 825 | 4,160 | red |
| lists ≤ 200 (events/feed/entities/proposals/evidence) | 349–3,388 | 1,560–14,176 | red |
| detail / neighbors ≤ 200 | 932 / 784 | 2,564 / 2,567 | red |
| graph ≤ 400 ms, ≤ 400 KB gzip | 4,159; 160.9 KB | 20,102 | red (size green) |
| impact / attack-paths ≤ 800 | 14,578 / 6,826 | 19,797 / 19,950 | red |
| ingest ≤ 300 | 11,631 | 20,168, 11.1 % errors | red |
| login ≤ 500 | 666 | — | red |
| freshness ≤ 60 s | 1 s | — | green |
| SSE ≤ 5 s / proposal ≤ 2 s | not measured by the load script | — | unmeasured |

Zero non-429 errors below saturation. Every latency target is red on this host: the single-host ceiling (ADR-016)
on a CPU-starved machine. QUICKSTART "Load test" documents the limiter reset.

## Claude Code permissions
- `.claude/settings.json` was not written by the build (kernel rule); the proposed `permissions.allow` additions are in QUICKSTART.md "Claude Code permissions".

## Governance

109 decision record(s) over 65 change(s), policy 1.3 (`GOVERNANCE-DECISIONS.jsonl`).

| Final outcome | Changes |
|---|---|
| AUTO_APPROVE | 65 |
| REVIEW | 0 |
| REWORK | 0 |
| HUMAN_APPROVAL | 0 |
| BLOCK | 0 |

**Human decisions:** F-01 approved (GD-0021); F-03 approved (GD-0022); F-06 approved (GD-0023); F-15 approved (GD-0024); F-17 approved (GD-0025); F-18 approved (GD-0026); C-1 approved (GD-0035); C-2 approved (GD-0036); C-10 approved (GD-0046); C-13 approved (GD-0047); C-14 approved (GD-0048); C-16 approved (GD-0049); C-9 approved (GD-0050); C-18 approved (GD-0053); C-19 approved (GD-0054); C-21 approved (GD-0057); C-22 approved (GD-0060); C-23 approved (GD-0061); C-27 approved (GD-0072); C-36 approved (GD-0079); C-44 approved (GD-0108); C-45 approved (GD-0109).
**Escalated to a human:** F-01 (minimum for project dependencies+infrastructure (planned in F-01)); F-01 (HUMAN_APPROVAL from GD-0001 stands (minimum for module application)); F-03 (minimum for project authentication+infrastructure+security-boundary); F-15 (minimum for project authentication); F-06 (minimum for project authentication+security-boundary); F-17 (minimum for project payments (planned in F-17)); F-18 (minimum for project dependencies+infrastructure); C-1 (minimum for project infrastructure); C-2 (minimum for module security-boundary); C-9 (minimum for project authentication+dependencies+infrastructure); C-10 (minimum for project authentication+dependencies); C-13 (minimum for module payments); C-14 (minimum for project dependencies); C-16 (minimum for project infrastructure+security-boundary); C-18 (minimum for project dependencies); C-19 (minimum for project dependencies); C-21 (minimum for project dependencies); C-22 (minimum for project architecture); C-23 (minimum for project architecture+security-boundary); C-27 (minimum for project architecture); C-36 (minimum for project unknown); C-44 (minimum for project architecture); C-45 (minimum for project dependencies+infrastructure).
**Blocked:** none.

## Usage

Measured from the Claude Code session transcripts that built this project (main session + subagents), by `scripts/usage-report.mjs`. Tokens are what the model processed; on a subscription plan this is what draws the usage limits, and on the API it is what is billed.

| Metric | Value |
|---|---|
| Sessions / subagents | 1 / 37 |
| Assistant turns | 9163 (5306 main) |
| Cache-read tokens | 917,603,639 (main 577.8M, subagents 339.8M) |
| Cache-write tokens | 28,005,273 |
| Output tokens | 5,249,160 |
| Uncached input tokens | 18,526 |
| Context per turn (main) | avg 112K, peak 167K, 0 turns over 400K |
| Compactions | 34 |
| Active time | 24.2 h (22 pauses totalling 55.6 h; span 2026-10-12T02:52:27.923Z to 2026-10-15T10:44:40.394Z) |
| Models | claude-opus-5-5 x9163 |

Per hour (UTC): cache-read / output / turns

| Hour | Cache-read | Output | Turns |
|---|---|---|---|
| 2026-10-12T02Z | 40K | 2K | 2 |
| 2026-10-12T03Z | 141K | 1K | 3 |
| 2026-10-12T04Z | 1.4M | 10K | 27 |
| 2026-10-12T07Z | 6.5M | 33K | 68 |
| 2026-10-12T08Z | 50.6M | 1.1M | 489 |
| 2026-10-12T09Z | 43.1M | 412K | 376 |
| 2026-10-12T10Z | 86.2M | 520K | 824 |
| 2026-10-12T11Z | 109.2M | 425K | 1147 |
| 2026-10-12T12Z | 42.7M | 199K | 437 |
| 2026-10-12T13Z | 49.4M | 188K | 459 |
| 2026-10-12T14Z | 7.4M | 53K | 74 |
| 2026-10-12T18Z | 2.5M | 13K | 34 |
| 2026-10-12T19Z | 24.6M | 123K | 218 |
| 2026-10-12T20Z | 5.0M | 47K | 48 |
| 2026-10-12T21Z | 9.8M | 76K | 101 |
| 2026-10-12T23Z | 26.9M | 137K | 243 |
| 2026-10-13T00Z | 11.8M | 46K | 91 |
| 2026-10-13T01Z | 16.0M | 73K | 149 |
| 2026-10-13T02Z | 109.1M | 242K | 1194 |
| 2026-10-13T03Z | 35.5M | 105K | 385 |
| 2026-10-13T04Z | 28.0M | 77K | 293 |
| 2026-10-13T05Z | 12.0M | 89K | 125 |
| 2026-10-13T06Z | 55.8M | 282K | 598 |
| 2026-10-13T07Z | 7.0M | 74K | 57 |
| 2026-10-13T09Z | 501K | 6K | 9 |
| 2026-10-13T10Z | 2.3M | 13K | 28 |
| 2026-10-13T11Z | 9.7M | 55K | 86 |
| 2026-10-13T12Z | 9.3M | 46K | 82 |
| 2026-10-13T17Z | 33K | 1K | 2 |
| 2026-10-13T19Z | 587K | 7K | 10 |
| 2026-10-13T20Z | 839K | 7K | 10 |
| 2026-10-13T22Z | 8.2M | 40K | 77 |
| 2026-10-13T23Z | 14.8M | 70K | 111 |
| 2026-10-14T00Z | 21.5M | 161K | 209 |
| 2026-10-14T01Z | 12.0M | 76K | 99 |
| 2026-10-14T02Z | 16.1M | 104K | 159 |
| 2026-10-14T03Z | 5.2M | 16K | 35 |
| 2026-10-14T11Z | 1.4M | 8K | 17 |
| 2026-10-14T17Z | 614K | 5K | 12 |
| 2026-10-14T19Z | 525K | 2K | 10 |
| 2026-10-14T20Z | 1.4M | 5K | 20 |
| 2026-10-14T21Z | 1.4M | 11K | 16 |
| 2026-10-15T00Z | 206K | 522 | 3 |
| 2026-10-15T01Z | 4.5M | 14K | 39 |
| 2026-10-15T08Z | 47.4M | 225K | 536 |
| 2026-10-15T09Z | 11.6M | 32K | 81 |
| 2026-10-15T10Z | 6.8M | 28K | 70 |
