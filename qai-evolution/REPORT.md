# REPORT — QAI // QUANTUM SDK

## Build Input Reconciliation

Non-blank fields: 19 (incl. template). Reconciliation rows: 20 (19 + domain). Unresolved conflicts: 0.

| Field | Value | Disposition | Evidence / rationale |
|-------|-------|-------------|----------------------|
| `build_depth` | comprehensive | Applied | ARCHITECTURE.md §7.1 threat model, §8 alternatives; 12–20 features in BUILD-PLAN.md |
| `review_gates` | auto | Applied | DECISIONS.md ADR-04 (self-verify, proceed) |
| `force_research` | false | Applied | Research ran because §3 was absent (Stage 0 gate), not because of force_research |
| `domain` | QUANTUM PHYSICS | Applied | ARCHITECTURE.md §2.2 SDK modules |
| `domain_signals` | ["has_scheduled_work"] | Applied | ARCHITECTURE.md §6 (T1–T3, effect tests, idempotent claim); invariants INV-8, INV-9 |
| `protocol_support` | HTTPS only | Applied | `compose.yaml` service `edge` (Caddy 2.11.7, `tls internal`, publishes ${HTTPS_PORT}:443, :80→:443 redirect); `infra/caddy/Caddyfile`; INV-19 |
| `monitoring` | basic health checks | Applied | `services/api/qai_api/routes/health.py` (`/api/health` db/cache/queue/worker); compose healthchecks; INV-5 |
| `container_strategy` | Docker Compose | Applied | `compose.yaml` (Stage 3 deliverable); ARCHITECTURE.md §2.1 |
| `database_preference` | PostgreSQL | Applied | ARCHITECTURE.md §4 (PostgreSQL 18.6 primary + streaming replica) |
| `rate_limiting` | true | Applied | ARCHITECTURE.md §7 (edge + `limits` on Valkey, cost-based job submit); INV-15 |
| `audit_logging` | true | Applied | ARCHITECTURE.md §4 `audit_events` append-only; INV-14 |
| `frontend_framework` | Next.js | Applied | ARCHITECTURE.md §2.2 `apps/web` (Next.js 16.4.3) |
| `state_management` | React Context | Applied | ARCHITECTURE.md §8 (React Context + server components) |
| `backend_framework` | FastAPI | Applied | ARCHITECTURE.md §2.2 `services/api` (FastAPI 0.162.0) |
| `orm_preference` | SQLAlchemy | Applied | ARCHITECTURE.md §4 (SQLAlchemy 2.1 async + Alembic); INV-12 |
| `typescript_backend` | false | Applied | Backend is Python; TypeScript only in `apps/web` |
| `scale` | large — 50k+ concurrent | Applied | Tier **large**: replica + read/write split (§4), Valkey cache + static immutable caching at edge (§5), workers ×2 with backpressure + DLQ (§5), web/api ×2 stateless (§2.1), edge rate limit (§7). Signal bumps: none. Multi-host is a residual (ADR-09) |
| `multi_tenant` | true | Applied | ARCHITECTURE.md §4 tenant_id + RLS; INV-13; §7.1 IDOR tests |
| `target_platforms` | ["web"] | Applied | `apps/web`; SDK installable as a library (ADR-07) |
| Design template / `theme.mode` | template dark vs config light | Applied (conflict resolved) | DECISIONS.md ADR-01: template is binding, single dark theme; INV-20 |

Blank key decisions (`deployment`, `backend_language`, `auth_model`), plus `prompt_mode` and `confidence_level`, are dispositioned in DECISIONS.md ADR-04.

## Stage 1 — Invariant proof

At Stage 1: `33 total / 32 machine (32 proven, 0 unproven) / 1 manual (skipped -- Reviewer Gate item 8)`.

Current (Re-Entry R1, RB-5): `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual`. INV-34 was added at Stage 4 (ADR-33). INV-2 (web-baseline) fails because the report cannot be produced until `artifacts/caddy-root.crt` is trusted on the host (PENDING HUMAN).

## Features
- F-1 SDK circuit core: 19 tests (19 added), including a hypothesis compile-equivalence check over 200 circuits against an independent dense reference.
- Checkpoint F-1 (FOUNDATIONAL, solo): full suite 19/19.
- F-2 SDK simulator: 10 tests added (29 in the suite), with 60 hypothesis circuits on up to 5 qubits checked against the independent dense reference.
- Checkpoint F-2 (wave 2 end): full suite 29/29.
- F-3 platform foundation: 18 API tests (foundation config, errors, health, headers, migrations: append-only audit, RLS WITH CHECK, downgrade/upgrade) plus 12 web unit tests. API and web images build. Web-baseline self-test passes. compose.yaml validates.
- F-4 auth, tenants, RBAC: 43 API tests (passwords, auth flows, CSRF/Origin, RBAC matrix, rate limits incl. store-down, tenant isolation) and the login/signup/reset screens with e2e fixtures and an auth spec. Web typecheck, lint, unit tests and build pass. INV-15 and INV-16 now pass.
- F-5 SDK algorithms: Grover (phase oracle + diffuser; n=3, M=1 success 0.945 ≥ 0.94) and QAOA-MaxCut with COBYLA (4-ring p=2 ratio ≈ 0.99–1.0 ≥ 0.75 across seeds 0–4), with exact and greedy baselines; 70 tests, SDK suite green.
- F-6 validation: 50-circuit Qiskit differential suite (≤ 12 qubits) min fidelity 0.9999999999999973 (≥ 1 − 1e-10); 10q/d20/1024 shots 0.021 s (< 1 s); 20q/d10 12.8 s, 16 MiB (< 30 s, < 64 MiB); 69 tests; SDK suite 167 passed, 1 opt-in skip.
- F-7 SDK evolution engine: seeded config perturbation, V/J computed by the engine over a fixed seed protocol, gate J ≤ J(Ψn) − ε with checkpoints; V-false never accepted, ε=1 accepts nothing, deterministic history; 28 tests, SDK suite 195 passed / 1 opt-in skip. INV-31/32 pass.
- F-8 projects and circuit builder: migration 0003 (projects, circuits, RLS WITH CHECK), 11 endpoints, 3 screens with a keyboard-operable canvas; 30 API tests (73 in the suite at the time), 39 web unit tests; INV-4 clear for F-8, INV-12 pass.
- F-9 members, invites, API keys, audit: API-key role capped at the creator's current role, GUC-scoped lookup RLS, admin resets limited to single-workspace users (fixed a cross-workspace takeover path), formula-safe CSV export; 36 API tests (109 in the suite), 7 web unit tests; five settings/invite screens with designed states.
- Checkpoint wave 5 (F-5..F-9): SDK 195/195 (+1 opt-in skip), API 109/109, web 58/58, web build OK, e2e 24/24 desktop+mobile twice against compose (new wave5 spec: W1 project→circuit→H+CX→compile depth 2, W7.1 API key reveal + Bearer). Fixes: Caddy health probe/retry, local e2e workers.
- F-10 Job queue, worker, simulator: Postgres queue (SKIP LOCKED, leases, retries, dead letter, watchdog), sandboxed child-process worker, append-only sha256 evidence, /app/simulator, /app/jobs, /app/jobs/[id]; API 154/154, web 67/67.
- Checkpoint wave 6 (F-10): SDK 195/195 (+1 skip), API 154/154, web 67/67, build OK, e2e 28/28 desktop+mobile (W1 builder Run→job→counts+hash verified, W7 API key job + audit + revoke). Fixes: lazy scipy import in the SDK, table-wrap containing block (mobile overflow), API healthcheck timing, worker memory 768 MB/child in a 2g container.
- F-11 Algorithms: /app/algorithms Grover and QAOA MaxCut tabs submit jobs; the job page compares QAOA against exact and greedy baselines and Grover against sin² theory, every chart with a table alternative; 2 API tests (258 in the suite with the wave-7 trees), 12 web unit tests, W3 e2e spec.
- F-12 Validation and evidence: /app/validation runs validate jobs, lists evidence with filters and an audited CSV/JSON export (50k cap, X-Export-Truncated, formula-safe); /app/validation/[id] shows Hash verified or mismatch (server re-verifies), check-by-check results and the exact canonical JSON; 14 API tests, W2 e2e spec. Additive F-10 helpers: `repositories.evidence.list_for_export`, `services.evidence.canonical_text`, `canonical` on GET /evidence/{id}.
- F-13 Benchmarks and scheduled work: /app/benchmarks (run, qai vs reference chart and table, schedules CRUD, scheduler runs), scheduler process with T1 benchmarks, T2 lease watchdog and T3 retention, idempotent per-window claims (INV-9); 26 API and 6 web tests. Integration fix: the SIGTERM tick test waits for the third tick instead of a fixed 150 ms sleep (flaked under load).
- F-14 Optimization and evolution: POST/GET /api/v1/evolution/runs, evolution worker runner, migration 0007 (append-only decisions, gate CHECK, RLS), server-side re-verification of every outcome against SDK decide() (S14), /app/optimization and /app/optimization/[id] with polling, J chart and decision log; 44 API and 6 web tests. Integration change: decisions persist in `services.jobs.complete` (same transaction as the result) instead of only lazily on read.
- F-15 MDLC governance: release candidates gated on verified evidence (latest per required kind passed and hash re-verified at evaluate, approve and release), reviewer-not-author approval (service, DB CHECK, session-only), admin release, revert to blocked on failing evidence, full audit trail; /app/governance and /app/governance/[id]; 18 API and 5 web tests, W5 e2e spec. Integration: `latest_by_kind` uses postgresql.distinct_on (removes 29 deprecation warnings).
- Wave 7 checkpoint (F-11..F-15): API 258/258, SDK 195/195 (+1 skip), web typecheck/lint 0, vitest 104/104, production build 0, compose up --build all healthy (scheduler healthy, tini double-init removed), e2e 34/36 (1 worker): W6 desktop+mobile failed on evolution TIMEOUT retries; root cause the soft budget not counting child start-up under host contention (ADR-24), fixed and re-verified in the final checkpoint; invariant-lint: INV-2 (baseline report, produced by the gate) and INV-4 (W4/overview, F-16) open. Added README.md, NOTICE, root .editorconfig, MDLC keywords and root test:e2e / lint:invariants / web:baseline / lint:py scripts.
- F-16 Overview dashboard and export: GET /api/v1/overview (viewer+, session or API key; projects, circuits, jobs by status, 10 recent jobs, evidence pass rate, release readiness, tenant queue depth) with per-tenant generation-stamped Valkey cache-aside (30 s TTL, invalidated after commit on terminal job transitions, DB fallback when Valkey is down; ADR-25); /app overview (KPI tiles, recent jobs, release summary, quick actions; loading/empty/error/success); W4 e2e (validate, evidence, sha256 detail, benchmark, CSV export, overview); auth spec restores the overview assertion; scripts/load-test.mjs. Tests: test_overview.py 14, jobs/worker/scheduler 63, vitest 109.
- Final checkpoint (Stage 3): ruff config + format across API and SDK (ADR-26), Playwright HTML report moved outside apps/web (INV-11 false positive on generated JS). API 272/272 (77% stmt), SDK 195 (+1 skip, 97% stmt), web 109/109, typecheck, lint and build pass, invariant-lint 31/32 (INV-2 pending the Web Baseline Gate).
- Final checkpoint e2e: mobile W6 exposed a real layout bug (implicit auto grid column let a wide chart table widen the 412 px viewport to 935 px; fixed on all nine app page roots, ADR-27, guard added to W6). W6 run cost pinned (rngSeed 6, maxiter 10) after measuring 15-96 s wall for identical runs on the saturated shared VM.

## Pre-Delivery Completeness Gate

Run 2026-11-01 on the F-16 final-checkpoint tree (build_depth: comprehensive). Verdict: **PASS** (one fix made during the gate, below).

| Area | Result | Evidence |
|---|---|---|
| Route and handler completeness | PASS | All 55 SPEC endpoints have a handler, checked against the app's OpenAPI document; the only extra route is `GET /api/health`. Status codes are 200/201/204 and problem+json 400/401/403/404/409/422/429/500 through `ApiError`, asserted in the API suite. |
| No stubs | PASS | No `TODO`/`FIXME`/not-implemented hits in source. The only `NotImplementedError` mentions are signal-handler suppressions in the worker and scheduler entry points (needed for Windows event loops). |
| Migrations | PASS | 0001–0008: every one has a non-empty `downgrade()`. |
| Supabase Data API grants | N/A | Direct Postgres over SQLAlchemy with the `qai_app` role; grants and RLS are in the migrations. |
| Seed and reference data | PASS | Roles, statuses and job kinds are enums and CHECK constraints in the migrations, not tables; there is no reference table to seed. |
| Validation | PASS | Pydantic strict models on every input endpoint, with field-level problem+json `errors[]`; client validation mirrors it (`_lib/*.ts` validators with vitest coverage). Raw DB errors never reach the client (the ApiError mapper and the 500 handler). |
| CRUD | PASS | Projects, circuits, schedules, API keys, invites, members and releases implement every SPEC operation (the handler cross-check above). |
| Auth flows | PASS | Signup, login, logout, switch-tenant, password reset (self-serve and admin) and API keys; covered by e2e auth.spec and wave5 W7.1. Unauthenticated requests return 401 problem+json (e2e and the API suite). |
| UI surface | PASS | INV-4 ui-coverage passes: every SPEC §6 screen, endpoint and workflow is reachable in the UI and e2e-tested. Empty, loading, error and success states are asserted per screen in vitest and e2e. |
| Environment | PASS | `.env.example` covers every variable compose reads; QUICKSTART.md is present; dependencies are pinned in pyproject and package-lock. |
| Delivery artifacts | PASS | BUILD-PLAN.md, README (commands and "Built with"), ci.yml, .editorconfig, ruff.toml plus eslint.config.mjs plus .prettierrc.json, multi-stage non-root Dockerfiles with HEALTHCHECK (compose build green), invariants.json and the pinned runner (`lint:invariants`), the pinned web-baseline plus config, and ARCHITECTURE §7.1 threat model and §8 alternatives. |
| Comprehensive extras | PASS (fixed) | `test:integration` added (the API suite against real Postgres and Valkey) plus `test:sdk`. Coverage was under-measured: SQLAlchemy's asyncio layer resumes code in greenlets, so `[tool.coverage.run] concurrency = ["greenlet", "thread"]` was added. API 77% became 91% (services 96%, services+repositories 95%) and SDK 97%, so the 80% bar is met, and CI now enforces floors of 85% (API) and 90% (SDK). |

## Verification Gate

Run 2026-11-01 on the stack rebuilt at HEAD (`/api/health` 200, commit identity matches). Verdict: **BLOCKED on one item, the web baseline** (needs local CA trust in the browser store); everything else passes.

| Step | Result | Evidence (artifacts/logs) |
|---|---|---|
| Clean install | PASS | `npm ci` 0 (VG-npm-ci); prod `npm audit` 0 vulnerabilities; 18 dev-only advisories go to the Security Audit |
| Build | PASS | `npm run build` 0; `docker compose up -d --build --wait` all healthy |
| Typecheck / strict | PASS | `tsc` 0; `"strict": true` |
| Lint and format | PASS | eslint + prettier (web) 0; `lint:py` (ruff check + format) 0; root `prettier --check .` 0 with a scoped `.prettierignore` (pinned scripts, stage docs and hook records excluded) |
| Unit and integration tests | PASS | web vitest 109/109; SDK 195 (+1 skip), 97%; API (`test:integration`, real Postgres + Valkey) 272/272, 91% (services 96%) |
| Full e2e (the pipeline's one run) | PASS | 40/40, 1 worker, desktop + mobile, 15.5 min (VG-e2e) |
| Invariant lint | 31/32 | INV-2 (web baseline report) is the only failure |
| CI parity | PASS | ci.yml now: ruff, invariant lint, SDK + API with coverage floors, web typecheck/lint/test/build, then stack up, health, e2e and web baseline (via `web:baseline:prep`) |
| Web baseline `--no-lighthouse` | **BLOCKED** | Node fetch fixed (`NODE_EXTRA_CA_CERTS`). Chromium: `ERR_CERT_AUTHORITY_INVALID`, so no screen is measured until the Caddy root CA is trusted in the Windows user store (ADR-29). |

Fixes made during the gate: the `/api/health` probe timeout went from 0.4 s to 2 s (ADR-28; a healthy DB had been reported down under host contention). `services/api/qai_api/middleware.py` was committed with CR-only line endings (git treated it as non-text, so its diffs were unreadable) and is now LF; 22 other files were CR-only in the working copy only. The `/app` redirect now carries a CSP. The invite screen's route is declared as a pattern. The Playwright HTML report moved to `artifacts/` (INV-11 false positive). QUICKSTART documents the per-OS one-time trust step and the correct HSTS values.

## Review Rounds

| Round | Reviewer | Security | Design | Remediation |
|---|---|---|---|---|
| 1 | FAIL (2 low) | PASS-ADVISORY (3 medium, 6 low, 3 info) | FAIL (2 ✗) | RB-1, ADR-30 |
| 2 | PASS | PASS-ADVISORY (2 medium, 2 low, 2 info) | 14 ✓ / 9 ⚠ / 1 ✗ (score-2) | RB-2, ADR-31 |
| 3 | PASS (10 focus IDs) | FAIL (1 high, 1 medium, 5 low, 2 info) | ✗ and mechanical ⚠ fixed in RB-4 | RB-4, ADR-32 |
| 4 (verify) | – | PASS-ADVISORY (0 medium, 2 low, 2 info) | – | residuals P4-1..P4-4 |
| R1-4 (W8 pipeline) | PASS (C-1, C-2, F-17, F-18, F-19; 3 low, 2 info) | PASS-ADVISORY pass 5 (1 low, 3 info) | – | RB-5, ADR-35 |
| R1-5 (wordmark + Observatory) | PASS (RB-5, C-7, C-8, F-20; 1 low, 4 info) | – (UI and docs only; no new endpoint or dependency) | score-4: 3/5 → 4/5 | F-20 fix 061d159, RB-6, ADR-38 |
| R1-6 (release 0.4.0) | PASS (RB-6, C-9, C-10; notes only) | – (no new endpoint or dependency) | – | C-11 |
| R1-7 (rebrand + 0.4.1) | PASS (C-12, C-13; 1 minor, 4 notes) | – (copy, styles, versions only) | header check 320–1440 px, no overflow | RB-7 05d558b |
| R1-8 (hero artwork) | PASS (C-15; 1 minor, 5 notes) | – (static asset, styles only) | hero 375–1440 px: image 200 image/webp, LCP = hero image at ≥ 768 px, 0 console errors, no overflow | RB-8 2c772ef |
| R1-9 (RB-8) | PASS (RB-8; 0 blocking, 2 notes accepted) | – | 404 card top-anchored (C-16): wordmark at the 24 px auth inset, 375/1440 px | – |
| R3-10 (W9 AI Simulation) | PASS (F-21, F-22, F-23; 3 minor, 7 notes) | – (see pass 6) | – | RB-9 1d4aeff, ADR-42 |
| R3-11 (release 0.5.0) | PASS (C-19, RB-9, RB-10; 2 minor, 3 notes) | PASS-ADVISORY pass 6 (2 low, 2 info) | – | RB-11 b2f4c0e, ADR-43; RB-12 400cbef, ADR-44 |
| R4/R5 12–18 (Orb Lab, Formula Lab, 0.6.0, 0.7.0) | PASS (C-21, RB-14..RB-17); round 17 FAIL on RB-18 (1 blocker), round 18 PASS with RB-19 | – | – | RB-14..RB-19, ADR-47 (`round-12.md`..`round-18.md`) |
| R6-19 (W12 evidence protocol) | F-33 FAIL (M-R19-1, requeue bypassed the one-protocol slot); F-31, F-32, F-34 PASS | – | – | RB-20 1d076b8 |
| R6-20 (RB-20) | PASS with minors (2 minor, 2 notes) | – | – | RB-21 ccf7d96, ADR-50 |
| R6-21 (RB-21) | PASS with minors (1 minor, 3 notes) | – | – | RB-22 ade705e |
| R6-22 (release 0.8.0) | PASS (C-27, RB-22, RB-23; 1 minor, 2 notes) | – (no new endpoint or dependency since F-33) | – | RB-24 98d1271 |

### Accepted residuals (tracked)

- **F-4** email verification at signup: SPEC Gap 12, no mail provider in v1.
- **F-5** job child shares the worker UID/network; Valkey has no password on the local stack (deployment hardening).
- **F-6** RLS bypass GUC: all SQL parameterised; no injection path found.
- **F-9** hashed Python lockfile: pins are exact; hashes when the toolchain allows.
- **F-10** uvicorn `--forwarded-allow-ips "*"`: API reachable only on internal networks behind Caddy.
- **F-11** CI actions pinned by tag; no secret-scan job (no scanner available in the build environment).
- **P2-3** audit/evidence export serialises on the event loop (admin session; bounded only by the general per-principal and per-IP windows, ADR-32 P3-7).
- **P2-5** `/api/health` is public with aggregate queue counts (SPEC allows unauthenticated health).
- **P2-6** narrow reset/invite race, not practically exploitable.
- **P4-1** compile slot is checked before, and taken after, the circuit load and rate charge: a request that loses the race pays its budget and gets 503 (2/201 under adversarial churn).
- **P4-2** roughly 15-30 workspaces at budget can still keep compile slots busy; only compile/QASM degrade (signup is 5/hour/IP). Long term: move compile onto the worker queue.
- **P4-3** bounded DSN pattern misses passwords over 512 / users over 128 chars; a DSN straddling the 16 KiB cut can keep part of its secret; long tracebacks lose their tail.
- **P4-4** the 50 ms redaction speed test is close to the measured 36 ms; no concurrency test for the key-cap lock (verified by code read).
- **P5-2** largest pipeline job is about 8.4 s of worker time for 1 job-rate unit; bounded by the 120 s timeout, `jobs:tenant` and the 50-queued cap (ADR-35).
- **P6-2** heaviest surrogate job (10 qubits, 2,000 ccx, 6 angles, maximal samples/candidates) is about 65 s single-threaded for 1 job-rate unit; inside the 120 s timeout, RLIMIT_CPU and the 50-queued cap. Follow-up with P5-2: work-weighted cost, non-retryable TIMEOUT for deterministic kinds, per-tenant fairness in `claim` (ADR-44).
- **P6-3 / P6-4** (info) SDK OverflowError on a huge integer span when called directly (the API bounds it first); SDK messages pass through in lower case.
- **N-R11-1 / N-R11-2** surrogate `parameters`/`target` errors report the first failure only; the code for a no-rotation / duplicate index is mapped from the SDK message, asserted end to end (ADR-43).
- **P5-4** SDK limits are looser than the API caps (the SDK is a local library; the API schema is the boundary).
- **Design-4 minors:** top-bar tab order (DOM order), tall Evidence card on job detail, Theory-card paragraph spacing, no scroll cue on the 375 jobs table, Overview Refresh alignment.
- **Security advisory:** no secret scanner or TypeScript SAST tool was available; ruff security rules, pip-audit and npm audit ran clean.

## Reviewer Gate

**PASS** (round R6-22, `artifacts/reviewer/round-22.md`): C-27 (release 0.8.0), RB-22 and RB-23 reviewed, 0 blocker / 0 major. RB-24 fixed N-R22-1 (SPEC: requeue of a failed or dead protocol job) and the CHANGELOG wording notes N-R22-2/3. `govern.mjs resolve-review` cleared C-27's REVIEW. Rounds R6-19..R6-21 took W12 (F-31..F-34) from a requeue-slot FAIL (M-R19-1) to PASS through RB-20..RB-22 (ADR-50). Earlier: **PASS** (round R3-11, `artifacts/reviewer/round-11.md`): C-19 (release 0.5.0), RB-9 and RB-10 reviewed, 0 blocker / 0 major; M-R11-1 (SPEC F-21 criterion scoped to span π) and M-R11-2 (CHANGELOG RB-7/8/9) fixed in RB-11 (ADR-43). `govern.mjs resolve-review` turned C-19's REVIEW into AUTO_APPROVE. Round R3-10 (`round-10.md`) passed F-21..F-23 (AI Simulation); RB-9 fixed M-R10-1..3 (ADR-42). Earlier: round R1-9 ( `artifacts/reviewer/round-9.md`): RB-8 (round 8 follow-ups: dead console CSS removed, Artwork primitive trimmed, plain-language alt text, ADR-40 CSP quote) reviewed, with 0 blocking findings; N-R9-1/2 accepted. Round R1-8 (`round-8.md`) passed C-15 (landing hero artwork, ADR-40); its N-R8-6 (no web-baseline LCP/CLS figure for the hero) stays open behind INV-2. Round R1-7 (`round-7.md`): C-12 (wordmark `Ψ ◈ QAI // QUANTUM INTELLIGENCE`, ADR-39) and C-13 (release 0.4.1) reviewed, with 0 blocking findings. `govern.mjs resolve-review` turned C-13's REVIEW into AUTO_APPROVE. RB-7 fixed the minor finding N-R7-1 (the full wordmark on the auth and 404 cards at 320 px) and note N-R7-2 (ADR-36 now marked superseded). N-R7-3..5 are accepted: the template tagline styling is not a token, the 1280 px cutoff is intentional (ADR-39), and the v0.4.1 tag is cut at completion. Earlier rounds: R1-6 (`round-6.md`) passed RB-6, C-9 and C-10. R1-5 passed RB-5, C-7, C-8 and F-20. R1-4 passed C-1, C-2 and F-17..F-19.

## Security Audit Gate

**PASS-ADVISORY** (pass 6, `artifacts/security/pass-6.md`): critical 0, high 0, medium 0, low 2, info 2, over W9 AI Simulation (F-21..F-23, RB-9). Tenancy (cross-tenant circuit 404, viewer 403), 22/23 adversarial inputs, memory (169 MiB peak), evidence determinism and hashing, the migration 0010 downgrade and text-only rendering all held. RB-12 fixed P6-1 (an oversized `parameters` list now costs one error; job-params 422s list at most 50 errors, SPEC §7); P6-2..P6-4 accepted (ADR-44). Earlier: **PASS-ADVISORY** (pass 5, `artifacts/security/pass-5.md`): critical 0, high 0, medium 0, low 1, info 3, over the W8 hybrid photonic pipeline. RB-5 fixed P5-1 (huge integers escaped as a 500) and P5-3 (wrong types now return `INVALID`). P5-2 and P5-4 are accepted residuals (above). C-7, F-20 and RB-6 change only copy, styles and docs: no new endpoint, input or dependency.

## Design Quality Gate

**PASS** (score-4, `artifacts/design/score-4.md`): the Observatory redesign (ADR-37) at 375 / 768 / 1280 across the landing, overview, jobs, job detail, pipeline, simulator, settings and the mobile drawer. It scored **3/5** first. The major finding was that the full wordmark showed in the mobile top bar, because `.brand` overrode `.topbar-brand`. That was fixed in 061d159, and the re-check scored **4/5**.
- Passed: no horizontal overflow; no console errors; visible focus on every stop; group tabs and sub-nav are keyboard-operable with correct `aria-current`; the drawer traps and returns focus; sticky bars never cover content; muted text contrast is about 6.6:1.
- Earlier rounds: score-3 (20 ✓ / 4 ⚠ / 0 ✗) on the previous design.
- Remaining minors are accepted (listed above).
- Rebrand (C-12, RB-7): site header, app top bar, login, sign-up and 404 measured at 320 / 375 / 768 / 1024 / 1280 / 1440 px. There is no horizontal overflow at any width. Below 1280 px the mark is `Ψ ◈ QAI` (76 px wide at 320 px on the auth cards), and from 1280 px the full tagline shows.

## Stage 4 — Delivery evidence

Release **0.8.0** (C-27 bfb538e; deployed with RB-24 at 4cc928f), 2026-11-08. Re-Entry R6 adds W12, the Evolution evidence protocol (F-31..F-34, ADR-49, ADR-50).

- Invariant lint, verbatim: `34 total / 33 machine (32 pass, 1 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`. INV-34 agrees at 0.8.0; the only failure is INV-2 (web baseline: HSTS off on localhost by contract, ADR-29; Lighthouse performance host-bound, ADR-45).
- Suites: SDK **519 passed, 9 skipped**, 96% (`artifacts/logs/sdk-0.8.0.log`); API **336 passed**, 90% (`api-0.8.0.log`, run from a venv outside the repo: an in-repo `.venv` is scanned by INV-10/INV-12); web vitest **326 passed** (`web-0.8.0.log`), lint 0; e2e **50 passed** desktop + mobile at d17614e, W12 included (`e2e-0.8.0.log`).
- Stack deployed at 0.8.0 (`deploy-0.8.0.log`, all services healthy). `/api/health` returns `{"status":"ok","version":"0.8.0",...}` with db, cache, replica, worker and queue ok.
- Functional smoke at 0.8.0: **29/29 PASS** (`smoke-0.8.0.log`).
- Not a public deploy: the stack is `deploy: local` (`tls internal`). A remote host needs an ACME site block, `PUBLIC_ORIGIN`, `HSTS_ENABLED=true`, real secrets, a Valkey password (F-5) and database backups; email delivery is a V1 non-goal.

Previous release **0.5.0** (C-19 e9113ce; deployed at 400cbef with RB-10..RB-12; tag v0.5.0), refreshed 2026-11-02. Re-Entry R3 adds W9 AI Simulation (F-21..F-23).

- Invariant lint (`npm run lint:invariants`), verbatim: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`. INV-34 (version agreement at 0.5.0) passes; the only failure is still INV-2 (Web Delivery Baseline, local CA trust).
- Suites at 400cbef: SDK **302 passed, 1 skipped**; API **295 passed**; web typecheck 0, vitest **154 passed**; e2e **44 passed** desktop + mobile (`artifacts/logs/e2e-0.5.0.log`, 2 workers; W9 AI Simulation included).
- Stack deployed at 0.5.0 (`artifacts/logs/deploy-0.5.0.log`, all services healthy). `/api/health` returns `{"status":"ok","version":"0.5.0","commit":"400cbef",...}` with db, cache, replica, worker and queue ok.
- Functional smoke at 0.5.0: **26/26 PASS** (`artifacts/logs/smoke-0.5.0.log`), including the `surrogate` AI Simulation job.

Previous release **0.4.1** (C-13 e558094, tag v0.4.1), refreshed 2026-11-01:

- Invariant lint (`npm run lint:invariants`), verbatim: `34 total / 33 machine (32 pass, 1 fail, 1 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`. The only failure is INV-2, the Web Delivery Baseline report, which needs the one-time local CA trust (ADR-29, QUICKSTART). The zero-files invariant is INV-2 itself. INV-34 (version agreement, 10 files) passes at 0.4.1.
- SDK suite: **269 passed, 1 skipped** (project venv). Web: typecheck, lint, prettier and vitest pass, and the production build succeeds. The API suite was last run in full at F-18 (286 passed). Since then only copy, styles and version defaults have changed in the API.
- Stack deployed at 0.4.1 (`docker compose up -d --build --wait`, all services healthy, `artifacts/logs/deploy-0.4.1.log`). `/api/health` returns `{"status":"ok","version":"0.4.1","commit":"e558094"}`, and db, cache, replica, worker and queue all report ok. The web container was rebuilt again with RB-7 (CSS only).
- Note for Windows hosts: Git Bash curl (Schannel) rejects the Caddy local CA with error 60 (`revocation status is unknown`), because the local CA publishes no revocation list. `curl --ssl-no-revoke --cacert artifacts/caddy-root.crt …` still verifies the chain.
- Startup verification: running the API with `VALKEY_URL` unset exits 1 with `Invalid environment (VALKEY_URL)` (`artifacts/logs/startup-missing-env-2.log`).
- `/api/health` latency (measured at RB-4, unchanged code path): p50 219 ms, p95 618 ms. **The p95 misses the 500 ms target** on this Docker Desktop host (`artifacts/logs/health-timing-idle.log`).
- Functional smoke (`./smoke-test.sh`) at 0.4.1: **25/25 PASS** (`artifacts/logs/smoke-0.4.1.log`). This includes GHZ-3 simulation, Grover, and the end-to-end photonic pipeline job.

## Governance

166 decision record(s) over 85 change(s), policy 1.5 (`GOVERNANCE-DECISIONS.jsonl`).

| Final outcome | Changes |
|---|---|
| AUTO_APPROVE | 85 |
| REVIEW | 0 |
| REWORK | 0 |
| HUMAN_APPROVAL | 0 |
| BLOCK | 0 |

**Human decisions:** C-3 approved (GD-0046); F-10 approved (GD-0047); F-15 approved (GD-0048); F-16 approved (GD-0049); F-17 approved (GD-0050); F-3 approved (GD-0051); F-4 approved (GD-0052); F-6 approved (GD-0053); F-8 approved (GD-0054); F-9 approved (GD-0055); RB-1 approved (GD-0056); RB-2 approved (GD-0057); RB-3 approved (GD-0058); RB-4 approved (GD-0059); RB-5 approved (GD-0066); C-11 approved (GD-0078); C-12 approved (GD-0079); C-8 approved (GD-0080); C-18 approved (GD-0105); F-21 approved (GD-0106); RB-11 approved (GD-0107); RB-12 approved (GD-0108); RB-9 approved (GD-0109); RB-13 approved (GD-0111); C-21 approved (GD-0123); RB-14 approved (GD-0124); RB-16 approved (GD-0125); C-24 approved (GD-0142); C-26 approved (GD-0147); F-33 approved (GD-0152); RB-21 approved (GD-0161); RB-22 approved (GD-0162); RB-24 approved (GD-0166).
**Escalated to a human:** F-3 (minimum for project authentication+dependencies+infrastructure (planned in F-3)); F-4 (minimum for module authentication); F-4 (HUMAN_APPROVAL from GD-0004 stands (minimum for project api+application+authentication+data-access+security-boundary (planned in F-4))); F-4 (HUMAN_APPROVAL from GD-0005 stands (minimum for module build-record+documentation+tooling)); F-4 (HUMAN_APPROVAL from GD-0006 stands (minimum for project api+application+security-boundary (planned in F-4))); F-6 (minimum for project infrastructure); F-8 (minimum for project authentication); F-9 (minimum for project infrastructure); F-10 (minimum for project infrastructure); F-15 (minimum for project dependencies+infrastructure); F-16 (minimum for project authentication+authorization+infrastructure+security-boundary); F-16 (minimum for project dependencies+infrastructure); F-16 (minimum for project dependencies+infrastructure+security-boundary); RB-1 (minimum for project authentication+dependencies+security-boundary); RB-2 (minimum for project security-boundary); RB-3 (minimum for project architecture); RB-4 (minimum for project security-boundary); C-3 (minimum for project architecture); F-17 (minimum for project architecture); RB-5 (minimum for project architecture); C-8 (minimum for project architecture); C-11 (minimum for project architecture); C-12 (minimum for project architecture); C-18 (minimum for project architecture); F-21 (minimum for project architecture); RB-9 (minimum for project architecture); RB-11 (minimum for project architecture); RB-12 (minimum for project architecture); RB-13 (minimum for project dependencies); C-21 (minimum for project architecture); RB-14 (minimum for project architecture); RB-16 (minimum for project architecture); C-24 (minimum for project architecture); C-26 (minimum for project architecture); F-33 (minimum for project infrastructure); RB-21 (minimum for project architecture); RB-22 (minimum for project architecture); RB-24 (minimum for project architecture).
**Blocked:** none.
**Ledger head:** `6a98be8179726d8b7bc73fc7be82cfad74bc3b9a197e77d344a39c92e6691e9f` after 166 record(s). Keep it outside this repository; `node scripts/govern.mjs verify 6a98be8179726d8b` proves a later ledger still contains this one.

## Usage

Measured from the Claude Code session transcripts that built this project (main session + subagents), by `scripts/usage-report.mjs`. Tokens are what the model processed; on a subscription plan this is what draws the usage limits, and on the API it is what is billed.

| Metric | Value |
|---|---|
| Sessions / subagents | 1 / 24 |
| Assistant turns | 4760 (2087 main) |
| Cache-read tokens | 499,303,505 (main 239.4M, subagents 259.9M) |
| Cache-write tokens | 16,344,218 |
| Output tokens | 5,390,529 |
| Uncached input tokens | 9,682 |
| Context per turn (main) | avg 117K, peak 167K, 0 turns over 400K |
| Compactions | 16 |
| Active time | 13.9 h (1 pauses totalling 0.4 h; span 2026-10-31T22:50:30.792Z to 2026-11-01T13:10:24.640Z) |
| Models | claude-opus-5-5 x4757, claude-haiku-5-5 x3 |

Per hour (UTC): cache-read / output / turns

| Hour | Cache-read | Output | Turns |
|---|---|---|---|
| 2026-10-31T22Z | 12.6M | 140K | 139 |
| 2026-10-31T23Z | 46.6M | 1.0M | 415 |
| 2026-11-01T00Z | 32.8M | 390K | 286 |
| 2026-11-01T01Z | 58.5M | 1.0M | 564 |
| 2026-11-01T02Z | 37.1M | 363K | 321 |
| 2026-11-01T03Z | 16.4M | 111K | 173 |
| 2026-11-01T04Z | 67.5M | 777K | 672 |
| 2026-11-01T05Z | 60.0M | 471K | 567 |
| 2026-11-01T06Z | 16.7M | 90K | 136 |
| 2026-11-01T07Z | 10.0M | 68K | 102 |
| 2026-11-01T08Z | 13.5M | 65K | 113 |
| 2026-11-01T09Z | 13.3M | 62K | 119 |
| 2026-11-01T10Z | 47.7M | 310K | 514 |
| 2026-11-01T11Z | 43.6M | 303K | 426 |
| 2026-11-01T12Z | 22.6M | 180K | 205 |
| 2026-11-01T13Z | 494K | 8K | 8 |
