# DECISIONS

Architecture Decision Records for QAI // QUANTUM SDK (project ce0ff2f1). review_gates: auto.

## ADR-01 — Design template outranks the config palette (dark-first, single theme)
- Context: Build config says "Light only" with a light palette; the bound design template (DESIGN-TEMPLATE.html) is dark-first (`--color-bg #05070D`, cyan accent `#00D9FF`).
- Decision: Template tokens are binding (Step 2b). Ship one fixed dark theme copied from the template `:root`; no theme switcher. Fonts (Manrope / Inter / JetBrains Mono) are self-hosted via next/font instead of the template's Google Fonts link.
- Alternatives: honour "Light only" (rejected: contradicts binding template); offer both (rejected: doubles state surface, template has no light tokens).
- Consequences: config's "Light only" is recorded as Conflict→resolved in REPORT.md Build Input Reconciliation.

## ADR-02 — Reverse proxy header buffers
- Context: Service floor requires explicit header-buffer sizing to avoid 431/502 on large cookies.
- Decision: Caddy is the TLS terminator; its default header limits (Go net/http, 1 MiB max header bytes) are generous, so no override is needed. Recorded here per service floor.

## ADR-03 — Kernel scripts pinned verbatim
- Decision: scripts/invariant-lint.mjs (v1.7), scripts/govern.mjs (v1.10), scripts/governance-policy.json (policy 1.5), scripts/web-baseline.mjs (v1.8) copied byte-for-byte from the MDLC gateway; never edited. usage-report.mjs pinned at Stage 4.
- Evidence: `node scripts/govern.mjs --self-test` → PASS; `node scripts/invariant-lint.mjs --self-test` → OK. web-baseline self-test needs @playwright/test, run once the web app installs it.

## Pre-Build Review (Step 3b)
Eight findings applied to RESEARCH.md §"Pre-Build Review". Each fix is carried by the ADR cited below.

## ADR-04 — Defaults for blank fields
- prompt_mode blank → "direct". confidence_level blank with review_gates auto → self-verify and proceed. Stage 1 gate outcome: auto (row `"auto"`/any).
- deployment blank → Docker Compose local (`deploy: local`) per container_strategy. backend_language → Python (FastAPI hard constraint). auth_model blank → first-party email+password sessions (Argon2id) + tenant API keys, RBAC owner/admin/member/viewer per tenant.
- Python: images use python:3.14-slim (research pin 3.14.10); local dev host has 3.13.7, so `requires-python >=3.12` and CI matrix 3.13/3.14.
- Queue: custom Postgres `jobs` table with `FOR UPDATE SKIP LOCKED` claim (zero framework, transactional enqueue) + Valkey for cache and rate limiting. Alternatives: Procrastinate (rejected: framework coupling for a small surface), Redis Streams (rejected: second durability source).
- Cache/rate limit store: Valkey 9.1.3 (BSD-3) over Redis 8 (licensing, R9). Rate limiting via `limits` 5.8.0 async (slowapi pins redis<4, R8).

## ADR-05 — Enterprise integration (W7) scope
API keys (hashed, prefix lookup, tenant-scoped, revocable) on `/api/v1` with the same RBAC and audit trail as sessions. Agent connectors/SDK clients beyond the REST API deferred.

## ADR-06 — Compiler and backends
Minimal compile pass in V1 (validation, decomposition to the native gate set, adjacent self-inverse cancellation, depth/count report). Tensor-network (quimb) and hardware backends deferred; `Backend` protocol keeps the seam.

## ADR-07 — Windows reproducibility
SDK `qai` is pure Python over NumPy/SciPy wheels; CI runs SDK tests on windows-latest + ubuntu-latest. Platform services are Linux containers run via Docker Desktop on Windows.

## ADR-08 — Performance thresholds
Fidelity |<ψ_qai|ψ_ref>|² ≥ 1 − 1e-10 (up to global phase) for every differential case ≤ 12 qubits; 20-qubit statevector supported; QAOA report includes approximation ratio vs exact (n ≤ 12) and greedy baseline; API read p95 < 300 ms, write p95 < 500 ms at test load; job submission returns 202 in < 200 ms p95.

## ADR-09 — Scale tier on Compose
Large tier realised as: Caddy edge (rate limit, compression, immutable caching of /_next/static as CDN stand-in), web ×2, api ×2, worker ×2 (bounded concurrency, backpressure 429 when queue depth > threshold, DLQ status after 3 attempts), Postgres primary + streaming replica with read/write split, Valkey cache-aside. Multi-host/multi-AZ is a residual for a remote deploy.

## ADR-10 — Retention and email
Retention sweep deletes job payloads/results older than 90 days (configurable `RETENTION_DAYS`); never touches `audit_events` or `evidence_records`. No email provider: owners issue invite tokens and password-reset tokens shown once in the UI.

## Stage 1 gate
- Review gate row matched: `review_gates: "auto"` → self-verify completeness and proceed.
- Build Input Reconciliation: 20 rows (18 non-blank fields + design template conflict + domain), 0 unresolved conflicts.
- Invariant proof (`node scripts/invariant-lint.mjs --prove`) closing line: `33 total / 32 machine (32 proven, 0 unproven) / 1 manual (skipped -- Reviewer Gate item 8)`
- Lint runner walks `.venv` and `.mdlc`; Python virtualenvs live outside the repo, and forbidden-pattern entries exempt `.mdlc/**` and `**/*.md`.

## ADR-11 — Scope expansion (comprehensive)
- scope expansion (comprehensive): operations surface (job queue + dead-letter requeue) → F-10 `/app/jobs`.
- scope expansion (comprehensive): audit trail screen → F-9 `/app/settings/audit`.
- scope expansion (comprehensive): data export (evidence, audit; JSON/CSV) → F-16.
- scope expansion (comprehensive): overview analytics dashboard → F-16 `/app`.
- scope expansion (comprehensive): API key management → F-9 `/app/settings/api-keys` (also W7).
- scope expansion (comprehensive): scheduler run visibility → F-13 `/app/benchmarks`.
None of these is a Non-Goal (QPU, tensor network, >20 qubits, QASM import, email, SSO, billing, agent connectors).

## Stage 2 gate
- `review_gates: "auto"` → self-verified: W1–W7 each have numbered steps with a named terminal step completed in a §6 screen; Non-Goals unspecced; §1–§10 present; §7 RFC 7807 envelope; §8 env vars.

## Stage 3 Multi-Agent Plan Gate
- `review_gates: "auto"` row → no halt; plan summarised, then dispatched automatically.
- 8 waves, 16 features. Waves 1–4, 6 and 8 run in the orchestrator (FOUNDATIONAL, CRITICAL or single-feature). Wave 5 runs 4 parallel subagents (A: F-5→F-7, B: F-6, C: F-8, D: F-9). Wave 7 runs 5 parallel subagents (F-11 to F-15).
- Disjointness comes from auto-discovered routers, pre-assigned migration revisions 0001–0008, per-feature model/repository modules and per-area page trees.
- Estimate 20 h; `t_max` = 2026-11-03T01:18:51Z (2.5×). Governance kernel: govern.mjs v1.10 + governance-policy.json 1.5, pinned.

## ADR-12 — Dependency reconciliation (F-3)
- Added `pytest-asyncio==1.4.1` (test extra, services/api). ARCHITECTURE §10 omits it, but async FastAPI/asyncpg tests need an event-loop runner. It is test-only, so no runtime surface change. Alternative: anyio's pytest plugin, rejected because the suite uses session-scoped asyncpg fixtures that `asyncio_default_fixture_loop_scope` handles directly.
- Added `@axe-core/playwright` and `lighthouse` (devDependencies, apps/web). The pinned web-baseline kernel needs them, and they are never shipped in the image.

## ADR-13 — Web shell conventions (F-3)
- **Auth gating.** `proxy.ts` sends anonymous `/app/*` requests a 307 to `/login?next=…`, judged only by whether the session cookie is present. The app layout still verifies the session against `/api/v1/auth/me`. A non-401 failure there renders a "Session service unavailable" banner instead of crashing the shell.
- **Headers.** The CSP nonce comes from `proxy.ts` and is read via `headers()` in the root layout, which makes every route dynamic (a deliberate cost of nonce-based CSP). `X-Frame-Options: DENY` is set, and `Permissions-Policy` adds `payment=()`. HSTS is emitted only when `HSTS_ENABLED=true` (QUICKSTART "Protocol & TLS"). `upgrade-insecure-requests` is omitted, since Caddy already redirects http to https with a 308.
- **Startup.** Env vars are validated at server start in `instrumentation.ts`, so a missing `PUBLIC_ORIGIN` exits non-zero. The validation is skipped during `next build`, so images build without secrets.
- **Dev only.** The `/api` rewrite to the API exists only under `next dev`. In compose, Caddy routes `/api/*`.
- **Tooling.** ESLint 9 is called directly, because `next lint` was removed in Next 16. Prettier keeps the `#00D9FF` brand token under `prettier-ignore`, because its lower-casing would break INV-20.
- **Assets.** There is no JSON-LD, which would need `dangerouslySetInnerHTML`. The OG and apple icons draw the brand diamond from boxes rather than the ◈ glyph, which would force a build-time font fetch.
- **Image.** The web image builds from the repository root, using the workspace lockfile and `npm ci`, so it is reproducible. The standalone server runs at `apps/web/server.js`. Caddy health-checks the web upstream on `/`.

## ADR-14 — Auth, sessions and rate-limit details (F-4)

- **Job cost cap.** The job-cost rate-limit weight is `min(max(1, 2^(n-10)), 200)` for an n-qubit job, so a 20-qubit job (weight 200) still fits a full principal budget and stays submittable. The SPEC curve without the cap would make 20 qubits unsubmittable.
- **Limiter storage.** The `limits` async Redis storage runs on coredis, which takes `stream_timeout`/`connect_timeout` (0.5 s each), not `socket_timeout`. When the store is down, login, signup and reset fail closed with 503 UNAVAILABLE, and authenticated reads keep working.
- **Session resolution.** Expired or orphaned sessions are deleted, and the 401 is raised after the transaction block so the delete commits.
- **API-key contract for F-9.** `deps.py` imports `qai_api.repositories.access.find_api_key` and `touch_api_key` lazily. F-9 provides them, and the api_keys RLS policy must allow lookup by prefix before the tenant is known (system scope), in the same way session resolution does.
- **Web.** `lib/session.ts` is renamed to `lib/session.tsx` to match the manifest path, and `Me` now mirrors `/auth/me` (slug, per-tenant role, isReviewer, principal). The auth screens share `app/login/_auth/` (AuthShell, auth.css, useAuthForm). `next` is accepted only as a same-site relative path. The e2e `account` fixture is worker-scoped because signup is limited to 5/h per IP, and it clears the `LIMITS:*` keys first through `E2E_LIMITER_RESET`.
- **Line endings.** `.gitattributes` pins LF (`* text=auto eol=lf`). A clean Windows checkout with `core.autocrlf=true` otherwise gets CRLF and fails the Prettier check, which the F-4 clean-worktree checkpoint caught.
- **Checkpoint fixes.** At startup the API calls `ratelimit.warm()`, which connects to Valkey and loads the Lua scripts; otherwise the first request on each worker took 6 to 7 s. The auth forms use `method="post"`, so a submit before hydration can never put credentials in a URL. They also set `data-ready` once hydrated (`useSyncExternalStore`), and e2e waits for it. Login costs about 1 s of Argon2id CPU (argon2-cffi RFC 9106 defaults, run in `asyncio.to_thread`), so the auth e2e assertions allow 15 s under parallel workers.

## ADR-15 — SDK algorithm bounds (F-5)
- MaxCut edge weights must be finite, 0 < w ≤ 1e6 (SPEC §3 leaves w unbounded) so ⟨C⟩/C_max ∈ [0, 1].
- An explicit Grover `iterations` is capped at 1000 (the optimum at n=12, M=1 is 50).
- Grover and QAOA ⟨C⟩ run on the statevector directly (phase oracle and diffuser; diagonal cost phase and per-qubit RX). `grover_circuit` and `qaoa_circuit` build the equivalent allow-listed gate-level circuits and are cross-checked to fidelity ≥ 1−1e-10; the gate-level Grover raises once it would exceed MAX_OPS.
- COBYLA uses rhobeg=0.5 from PCG64(seed) initial angles; reported angles are the best evaluated point. `approximationRatio` = ⟨C⟩ / exact optimum (brute force, n ≤ 16).
- `shots` and `seed` are keyword-only and required in `grover`.

## ADR-16 — Qiskit reference handling and evidence hashing (F-6)
- qiskit 2.6.1 and qiskit-aer 0.18.2 stay in the optional `[reference]` extra; CI installs `packages/qai-sdk[test,reference]`. `qai/validation/reference.py` is the only SDK module importing them, lazily inside functions (INV-17), including the Aer runs used by benchmarks. Without the extra, `compare`/`aer_statevector` raise `ReferenceUnavailable`, `run_checks(reference=True)` reports `skipped` (never a pass), and benchmark rows carry `referenceMs`/`fidelity` = null. Reference tests skip cleanly without the extra.
- Evidence hash canonicalisation v1: sha256 over ASCII canonical JSON of `{v:1, kind, inputs, outputs, env}`; string keys sorted, no whitespace, `ensure_ascii`, floats via `repr()`, `-0.0`→`0.0`, NaN/inf rejected, complex→`{im, re}`, numpy → Python scalars, unsupported types rejected; depth ≤ 64, output ≤ 32 MiB. env = {python, numpy, scipy, qai, platform}, so a hash is reproducible per environment (R2). `verify()` recomputes.
- The benchmark function lives in `qai/validation/bench.py` so `qai.validation.benchmark` is the function; `qai/__init__.py` exports `algorithms`, `evolution`, `validation` as submodules.

## ADR-17 — Evolution protocol (F-7)
- Candidates are data only: {p, maxiter, initialParams}; any other field (j, score, vPass, ...) raises `CircuitError(path="baseline")` (S14). The engine runs QAOA itself for the baseline and every candidate.
- `seeds` as a count derives distinct seeds from SeedSequence(rng_seed).spawn(2)[0]; candidate generation uses spawn(2)[1].
- Perturbation: p+{−1,0,+1} clamped 1–5; maxiter×{1,1.5} clamped 10–500; angles + N(0, 0.15) from the parent's angles (or a Δt=0.75 linear ramp), interpolated to the new p, rounded to 6 dp. With explicit initialParams each seed adds N(0, 0.05) from PCG64(seed) so J measures robustness across seeds.
- V(S): every seed's run passes normalisation, probability-sum, counts-sum and finite-history checks, ratio ∈ [0, 1], and the first seed re-runs identically. J(S) = mean(1 − ratio). Accept iff V ∧ J ≤ J(Ψn) − ε (inclusive) → `Decision.ACCEPTED`, else `Decision.REJECTED`.
- Bounds: iterations 1–20, ε 0–1, seeds 5–30. `max_seconds` aborts with `EvolutionTimeout`; F-14 must pass `max_seconds` below JOB_TIMEOUT_SEC (worst case at caps is tens of minutes with SciPy COBYLA).

## ADR-18 — Projects and circuit builder (F-8)
- Circuits in an archived project are read only: POST circuits, PUT and DELETE return 409 INVALID_STATE; GET, compile and qasm still work.
- A malformed UUID path id returns 404 NOT_FOUND, the same as a foreign id (S6).
- `projects.updated_at` is added beyond the SPEC model, for list display.
- Project counts for jobs, evidence and releases, and the HAS_RELEASES delete guard, take effect once those tables exist (checked with `to_regclass`); until then they are 0 or no-op.
- Compile runs inline in the API (pure SDK pass, no simulation) and is not audited. Circuit lists return opCount, not ops. PUT is compare-and-set on `version` (409 VERSION_CONFLICT); unique-name races map to 409 NAME_TAKEN via a savepoint.
- Builder: compile/export/Run save pending edits first; gates move via the inspector's target/control selects (no drag); the canvas is a single-tab-stop `role="grid"` with aria-activedescendant. Large circuits render as DOM cells (virtualisation deferred). Run depends on F-10 `POST /api/v1/jobs`.
- `getShellSession` is memoised with React `cache()` so layout and page share one /auth/me call per request.

## ADR-19 — Access model: members, invites, API keys, audit (F-9)
- Pre-tenant lookups of invites and API keys use SELECT-only RLS policies keyed on transaction-local GUCs (`app.invite_token_hash`, `app.api_key_prefix`), set for one statement and cleared; not `app.system`.
- An API key's effective role is min(key.role, the creator's current role in that tenant); rejected if the creator is gone or disabled; removing a member revokes the keys they created. Keys are capped at admin and cannot manage keys, members, invites or audit.
- Admin-issued password resets are refused (409) for users who belong to more than one workspace, except for yourself, because a reset affects the account everywhere. A new reset invalidates older unused tokens; TTL 24 h.
- Only an owner may grant owner, or change, remove or reset an owner. The last owner cannot be demoted or removed (409 LAST_OWNER, owner rows locked FOR UPDATE first).
- A role change ends the target's sessions in that tenant; a change to your own role rotates your session.
- Invites may be email-bound (anonymous mismatch 422, signed-in mismatch 403), are single use via a conditional UPDATE, expire after 7 d; a refused accept does not consume the invite. Additive response fields: invite list `status/createdBy/acceptedAt/revokedAt`, accept returns `{tenant, role}`.
- Audit export accepts the list filters, is capped at 50,000 rows with `X-Export-Truncated`, `Cache-Control: no-store`, and is itself audited. Residual (accepted, low): export is a GET with a SameSite=Lax cookie; a cross-site top-level link could trigger a download to the victim's own machine, audited.
- vitest now also collects `app/**/*.test.ts(x)`.
- Wave 5 checkpoint fixes: Caddy probes the web tier on `/robots.txt` (an SSR probe marked every replica down under load → 503) with `health_timeout 5s`, `health_fails 3`, and `lb_try_duration 5s` / `lb_try_interval 250ms` on both upstreams so a slow or restarting replica is retried, not surfaced. Local e2e defaults to 2 workers (`E2E_WORKERS`; CI keeps cpus()) and Argon2-heavy specs get a 90 s test timeout, because the stack shares a 4-CPU Docker VM. Follow-ups for later features: worker/scheduler crash-loop until F-10/F-13 ship their modules; `/auth/me` is ~0.3–0.5 s idle (examine at the performance gate); tini should run with `-s` or as PID 1.

## ADR-20 — Job execution model (F-10)
- Each job attempt runs in a fresh child process (`python -m qai_api.worker.child`, asyncio subprocess, JSON on stdin/stdout), not a ProcessPoolExecutor as ARCHITECTURE §5 sketched. Reasons: a hard kill on timeout without poisoning a pool, per-job POSIX RLIMIT_AS/CPU, an env allowlist with no credentials, crash or OOM isolation, and no pickle (INV-10). Cost: about 0.3 to 0.6 s of interpreter and numpy startup per job.
- Retries: INVALID_INPUT and INVALID_OUTPUT fail at once; TIMEOUT, CRASHED, BAD_OUTPUT, SPAWN_FAILED and runner errors requeue with run_after = now + 5 s·2^attempts; after JOB_MAX_ATTEMPTS (3) the job is dead. Admin requeue (session only) moves failed or dead back to queued with attempts reset.
- Leases of LEASE_SEC (60) are renewed every lease/3; the watchdog returns expired leases to queued, or to dead when attempts are spent; a worker that lost its lease cannot complete (guarded by worker_id). Claims use FOR UPDATE SKIP LOCKED.
- Evidence is written in the same transaction as succeeded, sha256 over canonical JSON, append-only by trigger, and re-verified on read (`verified`).
- Memory: compose sets JOB_MEMORY_MB=768 per child with a 2g worker limit, so WORKER_CONCURRENCY=2 children plus the parent fit and an oversized job fails as CRASHED instead of the cgroup OOM-killing the worker. On Windows dev only the timeout applies.
- `/api/health` counts queued/running jobs in a read-only, transaction-local `app.system` scope that returns aggregates only (previously the RLS-scoped count read 0).
- Deferred: backend selector beyond the statevector (UI shows "(Planned)"); evidence detail screen is F-12; full algorithm panels are F-11 (`jobs/[id]/algorithm-slot.tsx`).
- F-10 checkpoint fixes: `import qai` no longer pulls in `scipy.optimize` (QAOA imports it at call time; evidence reads the scipy version from package metadata), which took ~7 s per job child on the contended host. `.table-wrap` is now `position: relative`, so an absolutely positioned `.sr-only` header inside a horizontally scrolling table is clipped instead of widening the page on phones (it pushed a 412 px layout to 628 px and misplaced touch targets). The API healthcheck in compose allows 15 s and a 60 s start period because interpreter start alone took ~4 s on the shared Docker VM. Local e2e now defaults to 1 worker; W1/W7 job waits allow 150 s.

## ADR-21 — Scheduler owns T1–T3 with per-window idempotent claims (F-13)
- A separate process (`python -m qai_api.scheduler`) ticks every SCHEDULER_INTERVAL_SEC. T1 locks due schedules with FOR UPDATE SKIP LOCKED and claims `(schedule_id, window_start)` with INSERT … ON CONFLICT DO NOTHING against UNIQUE `uq_scheduled_runs_schedule_window` (INV-9); missed windows collapse into one catch-up run; a window is skipped while the previous run is still queued or running; QUEUE_FULL rolls back the claim so the next tick retries.
- T2 (lease watchdog) runs every tick and T3 (retention: run_results and jobs older than RETENTION_DAYS in batches of 500; evidence and audit kept) hourly. They moved out of the worker's `maintain()` (ADR-20), so lease recovery is not coupled to the process whose leases expire; the claim counts the attempt, T2 requeues with backoff or marks dead.
- Every trigger is `run_tX(now, tenant_id=None)` with injected UTC `now`. Narrowest scope per step: system_session only lists tenant ids with due work; per-tenant work uses tenant_session; the global `scheduler_runs` log uses user_session. Idle T1/T2 ticks write no row; T3 and manual triggers always do.
- SIGTERM finishes the current batch, then exits; `/tmp/scheduler-alive` is touched each tick for the compose healthcheck. The manual trigger is owner-only, 404 unless SCHEDULER_TRIGGER_ENABLED, hidden from OpenAPI, and audited (`scheduler.trigger`) after the run. Schedules are admin-managed, 25 per tenant. GET /benchmarks/summary is not cached (W4.5 needs fresh results).

## ADR-22 — Evolution persistence and server-side verification (F-14)
- Evolution runs execute as F-10 jobs (`kind: evolution`, internal: POST /jobs still refuses it; only POST /api/v1/evolution/runs submits). The API stores `evolution_runs` (config, status, baseline, checkpoints, best_checkpoint, summary) and append-only `evolution_decisions` (UPDATE/DELETE revoked from qai_app; runs cannot be deleted; UNIQUE(run_id, iteration); RLS WITH CHECK).
- S14: scores and decisions are computed only in the worker child with the SDK. Request models are extra=forbid with strict caps (nodes 2–12, edges ≤ 66, iterations 1–20, seeds 5–30, p 1–5, maxiter 10–500, epsilon 0–1); shots fixed at 256; rngSeed optional (server-chosen when absent); maxSeconds is server-derived.
- Before persisting, `repositories.evolution.verify_outcome` re-checks the whole outcome against SDK `decide()` and the checkpoint chain (count, order, epsilon, baselines, finiteness). A mismatch marks the run failed/INTEGRITY with zero decision rows. A DB CHECK enforces the gate (`decision='rejected' OR (v_pass AND j_value <= j_baseline - epsilon)`); with epsilon 0 an equal J is accepted, as in ADR-17.
- Budget from the job timeout: hard = t − min(10, 0.1t), soft = hard − min(20, 0.2·hard) (110 s / 90 s at 120 s); an over-budget run fails as non-retryable INVALID_INPUT.
- Persistence happens in `services.jobs.complete`, in the same transaction as the job result (integration change), so T3 retention cannot remove run_results before the run is first read; the idempotent lazy `sync_run` on read stays as the fallback and mirrors failed/dead/cancelled job states.

## ADR-23 — Release governance (F-15)
- Release candidates live in `release_candidates` (migration 0008; RLS WITH CHECK, DELETE revoked from qai_app) and change status only through `services/governance.py` (INV-27..30): pending → ready | blocked (evaluate) → approved → released.
- Evaluate: ready when, for every required kind (1–6 of the evidence kinds), the project's latest evidence of that kind exists, passed and its sha256 re-verifies; otherwise blocked with `missing[]` and `failing[]`. Approve and release re-verify; a failure blocks the release (audited) and returns 409 NOT_READY / NOT_APPROVED. New failing or unverified evidence of a required kind moves open ready/approved releases back to blocked (system actor), which clears the approval.
- Separation of duties is enforced three ways: the service (403 SELF_APPROVAL, including a reviewer-author and a release created through the author's API key, whose author is the key's creator), a DB CHECK `approved_by <> created_by`, and session-only approve/release (API keys 403). Approve needs member + reviewer flag; release needs admin. Invalid transitions return 409 INVALID_STATE; every transition is audited in the same transaction.
- Known limitation: evidence `created_at` and audit `at` use the transaction-start `now()`, so "latest" and audit order are by transaction start; overlapping transactions can interleave. Acceptable for one evidence write per job transaction; a sequence tiebreak is the follow-up if this ever matters.
- Integration: `repositories.evidence.latest_by_kind` uses `postgresql.distinct_on` (the `.distinct(col)` form is deprecated in SQLAlchemy 2.1).

## ADR-24 — Job soft budgets are measured from the child's spawn (wave 7 checkpoint)
- The worker's hard wall clock (`maxSeconds` / `JOB_TIMEOUT_SEC`) counts from the child's spawn, but the evolution runner started the SDK budget only after interpreter start and imports. On the shared 4-CPU Docker VM (other projects' containers load it; wall ≈ 3× CPU time) the soft deadline landed after the hard kill: the run died as a retryable `TIMEOUT` twice before the third attempt failed cleanly (W6 e2e).
- `worker/child.py` now records its start at import and passes `_elapsed` (outside `params`, like `_fault`; the API cannot set it) to the runner, and `evolution_runner._budget` subtracts it (floored at 1 s). A run that cannot finish ends with the non-retryable budget error before the hard kill.
- The W6 spec uses a smaller baseline (`maxiter` 20) so it stays inside the budget on a loaded host; product limits are unchanged.

## ADR-25 — Overview cache keying and invalidation (F-16)
- `GET /api/v1/overview` is cache-aside in Valkey: `qai:overview:v1:{tenant_id}:{gen}` with a 30 s TTL, where `gen` is a per-tenant integer at `qai:overview:gen:{tenant_id}` (no TTL). The key depends only on the active tenant: every viewer+ role sees the same body, and a tenant switch reads the other tenant's key. Queue depth is the tenant's own queued jobs, never the global queue.
- Invalidation INCRs `gen` instead of deleting the body, after commit, on cancel, success, terminal failure and lease-expiry death (`services/jobs.py`). A request that read the database before a job finished and writes its body afterwards writes it under the old generation, so a completion is never hidden for the rest of the TTL. Job submission does not invalidate (SPEC asks for completion; a new queued job shows within 30 s).
- Every Valkey error is swallowed and logged (`overview_cache_unavailable`, op and error class only): reads fall back to the database, never a 500; the TTL bounds staleness. Rejected: an in-process cache (not shared across API replicas) and plain DEL (stale-write race).

## ADR-26 — Python lint and format with ruff (final checkpoint)
- One root `ruff.toml` covers `services/api` and `packages/qai-sdk` (`npm run lint:py`, CI step "Lint and format check (Python)"). `ruff==0.17.3` is pinned in both test extras. Target `py312`, matching `requires-python >=3.12`: a `py314` target made the formatter drop the parentheses on multi-type `except` clauses, which is a SyntaxError below 3.14. Three such sites were restored and `compileall` is part of the check.
- Rules: E, F, W, I, B, UP, ASYNC, S. Ignored: E501 (the formatter owns line length, 120), B008 (FastAPI `Depends()`/`Query()` defaults). In tests: S101/S105/S106/S108 (asserts and fixture secrets), S603 (subprocess calls with fixed argv), ASYNC110 (polling loops in scheduler tests). Two inline `noqa`s carry a reason: `run_child(timeout=)` is the child wall clock enforced with `wait_for`, and `Decision(str, Enum)` keeps the member str/repr in the record format.
- Code changes from the findings: `zip(..., strict=True)` in QAOA, `RuntimeError` instead of `assert` in `db.engine.primary()`, the heartbeat file touch moved to `asyncio.to_thread`. Generated migrations are excluded. Rejected: flake8+black+isort (three tools for one job).

## ADR-27 — App page roots use one shrinkable grid column; W6 e2e pins its run cost (final checkpoint)
- Every app page root (`.ov-page`, `.jobs-page`/`.job-page`, `.opt-page`/`.evo-page`, `.algo-page`, `.bench-page`, `.gov-page`, `.sim-page`, `.val-page`, `.skeleton-page`) is `display: grid` with `grid-template-columns: minmax(0, 1fr)`. Without a template the implicit `auto` column grows to the widest descendant's min-content width: on the evolution run page the chart's data table widened the layout viewport to 935 px on a 412 px phone, so the page scrolled sideways and taps landed on the wrong element. A wide table now scrolls inside its own `.table-wrap`. W6 asserts `scrollWidth <= clientWidth` as the regression guard.
- W6 e2e pins `rngSeed: 6` (the form leaves the seed to the server) via a request rewrite and uses the maxiter floor (10): 1 accepted and 3 rejected decisions, about 1.3 s of CPU. Measured on the shared 4-CPU VM (52 containers, other projects about two thirds of the load): the same run took 15 s to more than 96 s wall, and an unpinned random seed varied the CPU cost about 2.5x. The product's bounded budget correctly failed the slow runs with INVALID_INPUT; the product timeouts are unchanged. Rejected: raising JOB_TIMEOUT_SEC to absorb another project's load.


## ADR-28 — Health probe timeout is a failure bound of 2 s (Verification Gate)

- **Status:** accepted (Verification Gate)
- **Context:** `/api/health` wrapped each dependency probe in a 0.4 s `wait_for`. The primary probe issues four statements (heartbeat max, `to_regclass`, `set_config`, the queue counts). On the shared 4-CPU Docker host a single replica ping measured ~220 ms, so the probe timed out and the endpoint returned 503 `db: TimeoutError` while Postgres was healthy. Compose and Caddy healthchecks use `/api/health/live`, so containers stayed healthy and the false 503 only showed on the full endpoint.
- **Decision:** `CHECK_TIMEOUT = 2.0`. It bounds how long health waits on a dead dependency. The SPEC p95 < 50 ms target for a healthy stack is unchanged and is measured by the load test.
- **Alternatives:** issue fewer statements (the counts need the RLS system scope, so the statements stay); cache the snapshot (stale health hides outages). Rejected for now.
- **Consequences:** a dead primary yields a 503 in ≤ 2 s instead of ≤ 0.4 s.

## ADR-29 — Local TLS trust for the web baseline; HSTS stays off on localhost (Verification Gate)

- **Status:** accepted; the browser trust step needs the human (see REPORT "Verification Gate").
- **Context:** the pinned web-baseline v1.8 talks to `https://localhost:8443` with Node `fetch` and stock Playwright Chromium contexts (no `ignoreHTTPSErrors`, no TLS hooks). Caddy serves `tls internal`, so both reject the certificate. Node is fixed without touching the system: `scripts/baseline-prep.mjs` exports the Caddy root and sets `NODE_EXTRA_CA_CERTS`. Chromium on Windows reads only the OS trust store, which the kernel says to populate (`caddy trust` / mounted root) for the enforced-trust check. Changing a host's root store is the machine owner's decision, so the build does not do it.
- **HSTS:** v1.8 flags "no Strict-Transport-Security" on any https `baseUrl`. Reference security-headers §1 says HSTS is never emitted on a `tls internal`/`localhost` origin: HSTS ignores the port, so once the local CA is trusted it would pin every other `https://localhost:<port>` app. `HSTS_ENABLED` stays false locally; this finding is carried as an informational residual, not fixed.
- **Also fixed:** the anonymous `/app` 307 now carries `default-src 'none'` CSP (it had none), and the invite screen is declared as `/invite/[token]` with an example route (tokenized and `noindex`, so it is not a sitemap entry).

## ADR-30 — Review Round 1 remediation batch

- **Status:** accepted (Review Round 1; artifacts/reviewer/round-1.md, artifacts/security/pass-1.md, artifacts/design/score-1.md).
- **Watchdog (R1-1, R1-2):** the duplicate `services.jobs.recover_expired_leases` (system-scope, never called in production) and its repository query are deleted; T2 lives only in `scheduler/triggers.py` (per-tenant). T2 and admin requeue now invalidate the `/app` overview cache after commit, like cancel/complete/fail.
- **Compile (Security F-1):** `POST /circuits/{id}/compile` runs SDK validation + compile in `asyncio.to_thread`, and a per-principal `compile:principal` limit of 30/min applies (general scope: fail-open to the in-process window). Threads still share the GIL, so the limit — not the thread — is the fairness bound; a process pool is not warranted at the 2000-op cap.
- **Password reset scope (Security F-2):** the issue-time "sole workspace" rule is re-checked at claim time without a schema change: an admin-issued token (issuer ≠ user) is honoured only while the user's single membership is a workspace where the issuer is still admin/owner (owner targets need an owner issuer); otherwise the generic `TOKEN_INVALID`. Pending resets are also voided on member removal and on signed-in invite acceptance.
- **SSR client address (Security F-3):** server components forward Caddy's `X-Forwarded-For` with the cookie (`lib/api-server.ts` `forwarded()`), so SSR `/auth/me` calls hit the caller's per-IP bucket, not the web container's. uvicorn keeps `--forwarded-allow-ips "*"` (F-10, info): the API is reachable only on the internal `frontend`/`backend` networks (caddy, web and our own services), Caddy overwrites client-supplied `X-Forwarded-For`, and pinning container addresses would need static IPAM. Residual, revisit for any deployment that exposes the API port.
- **Logs (F-7):** value-level redaction (DSN credentials, bearer tokens, `qai_…` keys, emails) applies to every string, including rendered tracebacks. **postcss (F-8):** bumped to 8.5.32 (clears the only high dev advisory). Deferred lows: F-4 (email verification; SPEC Gap 12, no mail provider), F-5 (child shares UID/network: env scrubbed, no secrets mounted; per-UID isolation needs a privileged worker), F-6 (RLS bypass GUC: no SQL injection found, all SQL parameterised), F-9 (hashed Python lock: pins are exact; hashes at Stage 4 if the toolchain allows).
- **Design:** the six drifted shadow/glow tokens are copied verbatim from DESIGN-TEMPLATE.html; h1–h4 are weight 800 and the hero/section/CTA headings uppercase as in the template; below 420 px the header drops the `// QUANTUM` suffix so the menu toggle stays on-screen; field and invite-row alignment and member-email truncation fixed. The page h1 stays in page content, not the top bar (SPEC §6 wording): one h1 per page in the main landmark is the accessibility-correct structure and the top bar carries the workspace switcher.

## ADR-31 — Review Round 2 remediation batch (Security pass 2)

- **Status:** accepted (artifacts/reviewer/round-2.md PASS; artifacts/security/pass-2.md PASS-ADVISORY, 2 medium).
- **CPU-bound request work (P2-1, supersedes the ADR-30 compile note):** the pass-2 measurement showed `to_thread` does not protect the loop (GIL contention: 1 ms sleeps stalled to 1.2 s p99 with 8 concurrent compiles). Compile and QASM export now run through `services/cpu.run_bounded`: one CPU slot per API process, and a caller that finds it busy gets `503 UNAVAILABLE` + `Retry-After: 1` instead of queueing (fail fast; the loop keeps a GIL slot at every switch interval). A tenant-wide `compile:tenant` window (120/min) sits above the per-principal 30/min, and a tenant may hold at most 25 active API keys (`409 INVALID_STATE`), so minting keys no longer multiplies the budget. A process pool was rejected: pickling circuits per call and a pool per replica cost more than the bounded slot at the 2000-op cap.
- **Quadratic validation (P2-2):** the SDK tracks measured qubits incrementally (`Circuit._measured` cache keyed on the ops list; `from_dict` threads one set), so validating a 2000-op circuit drops from ~150 ms to ~20 ms; this covers create, PUT, compile and QASM. QASM export (viewer, API keys) now also takes the compile budget.
- **Log redaction (P2-4):** the DSN pattern accepts an empty username and passwords containing `@`.
- **Accepted, tracked in REPORT.md:** P2-3 (audit/evidence export serialisation on the loop; admin session, already rate-limited by the export rule), P2-5 (public `/api/health` shows aggregate queue counts, no tenant data; SPEC allows it unauthenticated), P2-6 (reset/invite race, not practically exploitable). F-5's Valkey password is deferred to the deployment hardening list (local stack only; P2-4 now redacts it). F-11 (CI actions pinned by tag, no audit or secret-scan job) is accepted for v1: dependabot-style SHA pinning and a secret-scan job are listed in REPORT.md residuals with F-4 and F-9.

## ADR-32 — Review Round 3 remediation batch (Security pass 3, Design score 2)

- **Status:** accepted (artifacts/reviewer/round-3.md PASS; artifacts/security/pass-3.md FAIL: 1 high, 1 medium; artifacts/design/score-2.md 14 ✓ / 9 ⚠ / 1 ✗).
- **Log redaction is linear (P3-1, high):** every quantifier in the value patterns is bounded (`_EMAIL` local part ≤ 64, domain ≤ 253, TLD ≤ 24; `_DSN_CREDS` user ≤ 128, password ≤ 512), and `_scrub_str` truncates any string to 16 KiB before scrubbing. The access line logs the matched route template or `<unmatched>`, never the raw, attacker-chosen path. The unhandled-error lines truncate the path to 200 characters. Test: a 16 KB adversarial path, `://:`×4000, and 100 KB inputs each redact in under 50 ms.
- **Fair CPU slots (P3-2, medium; supersedes ADR-31's single slot):** `MAX_CONCURRENT = 2` per process, with at most one job in flight per tenant, so one tenant cannot hold every slot. Callers check `cpu.available(tenant)` before charging any window, so a 503 costs no budget. `compile:tenant` is cost-weighted by op count (`1 + ops // 250`; a 2000-op compile costs 9 of the 120/min), and the principal window is charged first (P3-5), so one throttled key cannot drain its tenant. A slot is released when the worker thread finishes (done-callback, awaited through `asyncio.shield`), not when the caller does (P3-3).
- **Key cap is race-free (P3-4):** `create_key` takes a transaction-scoped advisory lock per tenant (`pg_advisory_xact_lock(hashtext('api_keys:<tenant>'))`) before counting.
- **Doc corrections (P3-6, P3-7):** create and PUT still validate on the event loop, about 80 ms at the op cap. That is accepted: it runs once per write and the general principal limit bounds it. P2-3 is rate-limited by the general per-principal and per-IP windows only; no export-specific limit exists.
- **Design (score 2):** the populated Overview no longer overflows at 375 (`.ov-grid` has `minmax(0, 1fr)`). The landing evidence, governance and tile headings are uppercase as in the template. The settings submit button aligns with the inputs. Audit filters align to the top. The API-key creator email truncates. The validation summary column has a minimum width. Evidence labels are humanised (`numQubits` becomes "Num qubits"). Accepted as judgement calls: the circuit-builder toolbar wrap, eight media queries at 960/1100/30rem/419 rather than the WEB.md §1 set, "Open project" link text where the job payload carries no project name, the Projects table's horizontal scroll at 375 without a scroll cue, and the evolution failure text, which comes from the API.
- **Info (P3-8, P3-9):** the SDK `_measured` cache assumes ops are appended, not mutated in place; the API never mutates in place. DSN over-redaction is accepted. The `scheme://password@host` form is not redacted because it cannot be told apart from `user@host`.

## ADR-33: Release version agreement invariant (Stage 4)

- **Status:** Accepted, 2026-11-01.
- **Context:** The release version is stored in eleven places: the root and web manifests, the lockfile, both pyprojects, the SDK `__version__`, the API `app_version` default, the web `APP_VERSION` default, `.env.example` and the compose defaults. The kernel requires a `version-agreement` invariant for every multi-package build (root plus the `apps/web` workspace). invariants.json had none.
- **Decision:**
  - Add INV-34 (`version-agreement`, root `package.json`), with a pattern for each non-manifest location.
  - The release commit moves every location to the version in VERSION.md and nothing else.
  - The evidence-fixture `"qai": "0.1.0"` strings in tests are recorded environment data, not version defaults, so they stay.
- **Consequences:** A stale version default fails `lint:invariants` instead of shipping a mismatched `/api/health` version or image tag.

## ADR-34: Hybrid photonic pipeline — Classical AI → Photonics → Quantum Operations → Validation (Re-Entry R1)

- **Status:** Accepted, 2026-11-01.
- **Context:** The user asked for the enhancement "Classical AI > Photonics > Quantum Operations > Validation". The platform had no classical-ML stage and no photonic model. Its validation covered only gate-model circuits. RESEARCH.md "Re-Entry R1" records the physics, the sources and the prototype measurements.
- **Decision:**
  - **One workflow, one job.** W8 runs as job kind `pipeline` with evidence kind `pipeline`. There are no new tables or routes; it reuses the job queue, the isolated worker child, the evidence hashing and the release gate.
  - **Classical AI** is a NumPy 1→H→1 tanh MLP (seeded Adam, angle-space MSE). PyTorch and JAX were rejected because they add about 1 GB of wheels for a 50-parameter model and weaken cross-platform determinism. Marginal-space loss was rejected as non-identifiable (4 of 15 prototype seeds wrapped).
  - **Photonics** is an ideal discrete-variable linear-optics simulator: a Fock basis, permanents via Ryser, dual-rail encoding, ≤ 8 modes and ≤ 4 photons.
    - It applies single-qubit gates only. Deterministic linear optics cannot realise a CNOT (KLM), so a multi-qubit gate is a field error rather than an approximation.
    - Noise (loss, distinguishability, detector efficiency) is out of scope for V1.
  - **Validation** cross-checks the two physical models (TVD ≤ 1e-10), the interferometer's unitarity, post-selection and normalisation, and the surrogate's accuracy against the target marginals.
    - The default tolerance is 0.03, set from the worst 0.025 over 40 prototype seeds at hidden 16 and 1,000 epochs.
    - An undertrained model fails. The job still succeeds and stores fail evidence, so a release can be blocked on it.
  - Migration 0009 extends the three kind CHECKs (jobs, evidence, release required_kinds 1–7). Its downgrade refuses while `pipeline` rows exist, because evidence is append-only (trigger from 0005).
  - CPU is bounded by epochs ≤ 5,000, hidden ≤ 64, ≤ 4 photons and the existing job timeout and RLIMIT_CPU. The job-cost limit counts len(targets) qubits.
  - No new invariant is needed. INV-10 (schema caps), INV-11 (text-only rendering) and INV-12 (SQL only in repositories) already cover the new code.
- **Consequences:**
  - Three features ship in wave 9: F-17 SDK, F-18 API/worker and F-19 web.
  - SPEC §1/§2/§3/§5/§6/§9/§10 and ARCHITECTURE §2.2/§2.7 are updated first. COMPLIANCE is re-checked at the gate.
  - The Security Audit Gate and Drift re-run before ship, and the version moves to 0.4.0.

## ADR-35: Review Round 4 remediation batch (RB-5)

- **Status:** accepted (artifacts/reviewer/round-4.md PASS; artifacts/security/pass-5.md PASS-ADVISORY).
- **Huge integers are range errors (P5-1 / N-R4-1, low):** `PipelineParams` compares each target and the tolerance against its bounds without `math.isfinite`, so a 400-digit JSON integer is a 422 `OUT_OF_RANGE`, never an `OverflowError` 500. NaN and infinities fail the same comparison. The SDK (`qai.pipeline` targets, `qai.photonics` element parameters) uses the same comparison. Tests: 400-digit ints in targets and tolerance at the API and the SDK.
- **Wrong types are `INVALID` (P5-3, info):** strings, booleans, null and lists in `targets[i]` or `tolerance` return the SPEC §7 field code `INVALID`. Numbers outside the range stay `OUT_OF_RANGE`.
- **Every evidence kind exports (N-R4-2, low):** `GET /api/v1/export/evidence?kind=pipeline` is accepted.
- **Web kind lists include `pipeline` (N-R4-3, low):** the job kind type, job kind labels and filter, and the validation and governance evidence kinds and labels. `Record<Kind, string>` keeps the label maps complete at type-check time.
- **Postselection check value (N-R4-4, info):** the check stores `|1 − P(valid dual-rail)|`, as SPEC §5 F-17 states. The stage still reports `photonics.postselection` as the probability. Evidence created after this change hashes the new value. Older records keep their stored canonical text and still verify.
- **Docs (N-R4-5, info):** SPEC F-17 names `transition_amplitudes(interferometer, occupation)`, which matches the code. The REPORT Stage 1 tally is labelled as the Stage 1 figure, and the current tally is recorded.
- **Accepted residuals:** P5-2 is accepted. The largest pipeline job costs 1 job-rate unit for about 8.4 s of worker time, and there is no per-tenant queue fairness; the 120 s job timeout, the `jobs:tenant` rate limit and the 50-queued-per-tenant cap bound it, as they already do for a cost-1 QAOA job. P5-4 is also accepted. The SDK's limits are looser than the API caps (`MAX_HIDDEN` 256 and `MAX_EPOCHS` 20000 against 64 and 5000). The SDK is a local library, and the API schema is the network boundary.

## ADR-36: Wordmark is `I AM // QAI`

- **Status:** superseded by ADR-39 (user direction, 2026-11-01).
- **Decision:** the visible brand changes from `Ψ ◈ QAI // QUANTUM` to `Ψ ◈ I AM // QAI` in the web wordmark, the page titles and metadata, the manifest, the Open Graph image, the footer signature and the design template's wordmark. The compact mark (app sidebar), and the site-header mark below 420 px, read `Ψ ◈ QAI`.
- **Not changed:** the `qai` package, the API title, repository docs and historical review artifacts keep the product name QAI // QUANTUM SDK. Design tokens are unchanged, so ADR-01 and ADR-30 still hold.

## ADR-37: Observatory redesign (Re-Entry R2, F-20)

- **Status:** accepted (user direction, 2026-11-01: "redesign the site to look like" a supplied reference page).
- **Context:** ADR-01 and ADR-30 bind the web tokens to DESIGN-TEMPLATE.html. The user supplied a new visual reference, the "Observatory" page: a deep-space background with a star-grid field, glassy gradient cards, Space Grotesk headings, mono eyebrows, a cyan and magenta glow, and grouped top-tab navigation.
- **Decision:** the template's `:root` values change to the Observatory values, and `globals.css` copies them verbatim, so ADR-01's source rule still holds. Space Grotesk replaces Manrope (still `next/font`, so the nonce CSP needs no font CDN). The radii are 4/8/14. On desktop the app shell moves from a 224 px sidebar to grouped top tabs with a pill sub-nav; below 1024 px the existing accessible drawer stays. The landing page is restyled with a cinematic hero, phenomena tiles, the equation band and entry cards.
- **Kept:** the accent stays `#00D9FF`. It is the brand cyan that INV-20 checks, and it sits close to the reference's `#5fe3ff`. Violet stays for the rotation gates and the optimization meaning (SPEC §6). Status colour still comes with an icon and a text label.
- **Not adopted:** the reference's raster images (`assets/*.webp`) were not supplied, so the art is CSS/SVG gradients. The reference's demo dataset is fictional and is not copied; the app renders only real tenant data. The reference's Google Fonts `<link>` is replaced by `next/font`.

## ADR-38: Review round 5 follow-ups (RB-6)

- **Status:** accepted (artifacts/reviewer/round-5.md PASS; 1 low, 4 info).
- **N-R5-2 (low):** at ≥ 1024px `.page-title` is 40px, the SPEC §6 h1 step, instead of an off-scale 48px.
- **N-R5-3 (info):** DESIGN-TEMPLATE.html now loads Space Grotesk instead of Manrope, which matches its display font (ADR-37).
- **N-R5-5 (info):** the unused `--sidebar-w` token is removed from globals.css. It is not a template token.
- **Accepted:** N-R5-1 (F-20 also touched its own manifest path and the design report, which are both build records). N-R5-4 (the chained comparison in a test is correct as written). The design-4 tab order, where the top-bar tabs come before Sign out and then the sub-nav, follows DOM order and is accepted for V1.

## ADR-39: Wordmark is `Ψ ◈ QAI // QUANTUM INTELLIGENCE` (supersedes ADR-36)

- **Status:** accepted (user direction, 2026-11-01: "Rebrand demo site from I AM // QAI to Ψ ◈ QAI // QUANTUM INTELLIGENCE").
- **Decision:** the visible brand changes from `Ψ ◈ I AM // QAI` to `Ψ ◈ QAI // QUANTUM INTELLIGENCE` in the web wordmark, page titles and metadata, the manifest, the Open Graph image, the error page, the footer signature and the design template. In the wordmark, `QAI` is the display-weight lead; `QUANTUM INTELLIGENCE` is a mono, tracked, uppercase tagline (`fg-2`). The link's accessible name is its visible text, "QAI Quantum Intelligence" (or "QAI" where the tagline is hidden), so screen readers say words rather than spell capitals (ADR-45; it was an `aria-label`, "QAI Quantum Intelligence home").
- **Responsive:** the tagline and slash hide below 1280 px wherever the wordmark appears (site header, app top bar, auth and 404 cards); measured at 768/1024, the full mark overlapped the nav, leaving `Ψ ◈ QAI`. The compact mark (drawer) is `Ψ ◈ QAI`.
- **Not changed:** the `qai` package, the API title, repository docs and historical review artifacts keep QAI // QUANTUM SDK. Tokens are unchanged (ADR-37 holds).

## ADR-40: Landing hero uses raster artwork

- **Status:** accepted (user direction, 2026-11-01: "incorporate this hero image into the home page", source `Quantum Intelligence Beyond Classical.png`).
- **Decision:** the hero's right column shows the artwork from the supplied mockup: a Bloch sphere over a `Ψ ◈ QAI` processor, with the Circuit, Simulate, Optimize and Validate step cards. It replaces the Bell-pair `HeroConsole` panel and the CSS orbit art (ADR-37). The image is cropped to the art alone and stored at `apps/web/assets/hero-quantum.webp` (1101 × 704, about 115 KB) and statically imported, so Next emits it as a hashed `/_next/static/media` file that Caddy already serves with immutable caching (no Dockerfile change; the web image does not ship `public/`). The mockup's nav, headline, buttons and feature strip are already real HTML, so cropping them out avoids duplicate, unreadable text. A small baked-in fragment of the eyebrow text was painted out.
- **Delivery:** the image is static, same-origin (CSP `img-src 'self' data:` unchanged) and pre-encoded. It renders through the `Artwork` primitive (`components/ui`) as a plain `<img>` with explicit dimensions (no layout shift) and eager, high-priority loading. `next/image` was tried first and rejected: it injects an inline `style` that the nonce CSP blocks. Its edges are masked so they fade into the page background. The alt text describes the scene.
- **Not changed:** the hero copy, CTAs, meta strip and design tokens (ADR-37 and ADR-39 hold). The design template keeps its console mock; this is a layout choice for the live landing, not a token change.

## ADR-41: Re-Entry R3 — AI Simulation with a physics-informed circuit surrogate

- **Status:** accepted (user direction, 2026-11-02: "Let's add AI powered simulation https://simutechgroup.com/ai-simulation/").
- **Context:** the reference offers simulation surrogates in the Ansys SimAI style: sample a design space, run the full solver, train a model on the results, explore with the model, and confirm with the solver. QAI already has an exact solver and an evidence chain, so the natural "design" is a saved circuit and its rotation angles (RESEARCH R3).
- **Decision:**
  - Add SDK module `qai.surrogate` (F-21), job and evidence kind `surrogate` with migration 0010 (F-22), and the `/app/surrogate` screen with job result panels (F-23). The workflow is W9.
  - The surrogate is a **Fourier ridge regression** on the tensor basis {1, cos θ_j, sin θ_j}^⊗d. Every allow-listed parametrised gate makes each output probability a degree-1 trigonometric polynomial in each of its angles, so this basis contains the true response. With more than 3^d training samples the fit is exact to floating point at the full span ±π; at narrower spans the basis is near-collinear and the ridge term leaves a hold-out TVD near 1e-6 to 2e-5 (ADR-42, ADR-43); below 3^d samples it is underdetermined, and the hold-out check fails it honestly.
  - d ≤ 6 (729 features), n ≤ 10 qubits, samples ≤ 4,096, candidates ≤ 65,536. Each run fits a fresh model in memory. Nothing is persisted but the result and evidence.
  - Everything runs locally in the worker child. There is no external model, AI service or network call, so tenant circuits never leave the stack.
- **Alternatives rejected:**
  - A **neural network** (MLP, the R1 `AngleSurrogate` style). It is accurate for d ≤ 3, but at d = 6 over ±π its held-out TVD was 0.16–0.57 with up to 2,048 samples. That is no better than predicting the mean, and it needs tuning per circuit (RESEARCH R3).
  - **Gaussian processes.** O(N³) in samples, with kernels that need tuning, and no exactness guarantee.
  - **Calling an external AI simulation service.** It would send tenant data off the stack and contradicts the no-network SDK rule.
  - **Tying shared parameters** (one γ across QAOA edges). Deferred; each occurrence is its own angle in V1.
- **Consequences:**
  - The four W9 checks give a falsifiable claim: an evidence record passes only if the surrogate matched the exact solver on held-out and on chosen designs.
  - `requiredKinds` grows to 8 kinds. The "AI Simulation" link joins the Execution group of the navigation, after Pipeline.
  - Releases 0.5.0.

## ADR-42: Review round 10 follow-ups (RB-9)

- **Status:** accepted (Reviewer Gate round 10, `artifacts/reviewer/round-10.md`, 2026-11-02).
- **Context:** round 10 passed F-21..F-23 with 3 minor findings. M-R10-1: "exact to floating point" holds only at span ±π; narrower spans measured hold-out TVD 1.7e-6 to 9e-6 with `determined: true`. M-R10-2: an empty or over-long `params.parameters` returned pydantic's TOO_SHORT / TOO_LONG instead of the SPEC §3 NO_PARAMETERS / TOO_MANY_PARAMETERS. M-R10-3: a server `circuitId` error did not land on the circuit select.
- **Decision:**
  - The exactness claim (ADR-41, SPEC, SDK docstring, form note) is qualified to the full span; the form advises a tolerance of at least 1e-4 at narrower spans. The SPEC acceptance test stays at ±π, where it holds.
  - SurrogateParams checks the op list length itself and returns NO_PARAMETERS / TOO_MANY_PARAMETERS; the target is checked without a regex pattern, so its message no longer exposes one (N-R10-1). DUPLICATE (a §7 field code already used by grover `marked`) is listed in SPEC §3 for a repeated index (N-R10-2).
  - The circuit select carries the `circuitId` error and `aria-describedby` (M-R10-3); the underdetermined warning is a persistent live region (N-R10-7), and a unit test renders it (N-R10-6).
- **Consequences:** no wire-format change other than the two error codes, which now match SPEC. N-R10-3 (synchronous circuit parse before the rate limit, ~10 ms), N-R10-4 (one rate-limit unit per surrogate job, 2.2× timeout headroom) and N-R10-5 (ranking on the unclipped prediction) are accepted as notes.

## ADR-43: Review round 11 follow-ups

- **Status:** accepted (Reviewer Gate round 11, `artifacts/reviewer/round-11.md`, 2026-11-02).
- **Context:** round 11 passed C-19, RB-9 and RB-10 with 2 minor findings and 3 notes. M-R11-1: ADR-42 said the SPEC was qualified, but the SPEC §5 F-21 hypothesis criterion still promised TVD ≤ 1e-6 at any span (measured 1.3e-6 to 1.6e-6 at d = 2, span ±0.05 / ±0.01). M-R11-2: the 0.5.0 CHANGELOG omitted RB-7, RB-8 and RB-9 (an API-visible error-code change) and repeated the unqualified exactness claim.
- **Decision:**
  - The SPEC criterion is scoped to the default span π, with ≤ 1e-4 at narrower spans; the narrow-span band in ADR-41 and the SDK docstring reads 1e-6 to 2e-5 (N-R11-3 measured 1.4e-5 to 1.7e-5 at d = 4 and 6). The form keeps advising a tolerance of at least 1e-4.
  - The CHANGELOG 0.5.0 section lists RB-7, RB-8 and RB-9, and the VERSION.md row names the RB-9 code change.
  - Accepted: N-R11-1 (the parameters and target checks run after field validation and report the first failure; the list is bounded by the 256 KiB API body limit and is checked at ≤ 6 before use), N-R11-2 (the SDK-message-to-code mapping for no-rotation / duplicate indices stays; each mapped code, NO_PARAMETERS, NOT_PARAMETRISED and DUPLICATE, is asserted end to end in `test_jobs_surrogate.py`, so a reworded SDK message fails the suite).
- **Consequences:** documentation only; no behaviour change.

## ADR-44: Security pass 6 (AI Simulation)

- **Status:** accepted (Security pass 6, `artifacts/security/pass-6.md`, 2026-11-02; PASS-ADVISORY, 2 low, 2 info).
- **Context:** pass 6 audited F-21..F-23 and RB-9. P6-1 (low): RB-9 dropped the schema bound on `params.parameters`, so a 256 KiB list of strings produced 65,486 field errors (6.8 MB, ~450 ms on the event loop), and unknown `params` keys already amplified the same way for every kind. P6-2 (low): the heaviest surrogate job (10 qubits, 2,000 ccx, 6 angles, maximal samples and candidates) takes ~65 s single-threaded and costs 1 job unit, widening P5-2. P6-3 (info): the SDK `_num` raises OverflowError on a huge integer span/tolerance when called directly. P6-4 (info): SDK messages pass through in lower case, and an out-of-range op index reads NOT_PARAMETRISED.
- **Decision:**
  - P6-1 fixed: a `mode="before"` validator rejects a `parameters` list longer than 6 with one TOO_MANY_PARAMETERS error before its entries are validated, and job-params 422s list at most 50 field errors while the message counts all of them (SPEC §7). Tested in `test_jobs_surrogate.py`.
  - P6-2 accepted as a residual tracked with P5-2: the worst case stays inside JOB_TIMEOUT_SEC 120, RLIMIT_CPU and the memory limit, and the per-tenant queue cap of 50 bounds the backlog. Work-weighted job cost, non-retryable TIMEOUT for deterministic kinds and per-tenant fairness in `claim` are the follow-up. RESEARCH R3 now records the ~65 s ccx measurement.
  - P6-3 and P6-4 accepted (info): the API bounds span and tolerance before the SDK sees them, and the messages leak nothing internal.
- **Consequences:** an oversized params body costs one bounded response; a validation response is never larger than 50 entries.

## ADR-45: Web-baseline fixes (RB-13)

- **Status:** accepted (Ship gate, 2026-11-02; the full `web:baseline` reported 47 failing cells after 0.5.0).
- **Context:** 23 cells were axe `label-content-name-mismatch` on the wordmark: its `aria-label` ("QAI Quantum Intelligence home") did not start with the visible text, which breaks voice control (WCAG 2.5.3). 12 console cells and 3 `0 h1` cells came from auditing the detail screens with the zero UUID, so each rendered its designed not-found state and logged the 404 fetch. 3 cells could not load `/app/algorithms`, because Git Bash had rewritten the route in `web-baseline.config.json` to `C:/Program Files/Git/app/algorithms`. Governance at 375 px shifted 0.108 when the create form replaced its 3-line skeleton. invite-accept had no canonical. The landing hero sent the 1101 px file to phones.
- **Decision:**
  - The wordmark drops its `aria-label`; the accessible name is the visible text (ADR-39 amended).
  - `scripts/baseline-prep.mjs` seeds the baseline account with a project, a succeeded simulate job and its evidence, an evolution run and a release candidate, and writes `artifacts/web-baseline.config.json` (gitignored) with the detail screens' `exampleRoute` pointed at them. `npm run web:baseline` reads that config, so detail screens are measured with real records. The committed config keeps the zero-UUID routes as the fallback and the algorithms route is `/app/algorithms` again.
  - `EmptyState` accepts `headingLevel={1}`; release-detail's not-found state uses it, so that screen has an h1 in every state.
  - The governance create panel's loading state reserves the form's measured height (809/404/341 px at 375/768/1280).
  - The second full run (17 cells) then measured the same skeleton-to-form shift on simulator, validation and optimization. Their loading states reserve the loaded block's measured height too. Evidence detail's canonical JSON now scrolls on the focusable `CodeBlock` region rather than on its inner `<pre>` (axe `scrollable-region-focusable`).
  - invite-accept emits a self-canonical. The page is noindex and no-referrer, and the token is already in the URL its holder opened.
  - `Artwork` takes a `small` copy and `sizes`; the hero offers a 720 px webp (49 KB, against 115 KB) through `srcset`.
  - The evolution screen's title is "Evolution Engine" (user direction, 2026-11-02: keep the product name; the eyebrow stays Optimization).
- **Not changed:** HSTS stays off on the `tls internal` localhost origin (SPEC S1, ADR-29), so the `headers` site cell remains an accepted residual.

## ADR-46: Re-Entry R4 — Orb Lab (ORB-0 energy validation) and the Evolution Engine demo

- **Status:** accepted (user direction, 2026-11-03: the QAI // ORB LAB infographic, "Ingest image and update program accordingly", "Should the demo include more info about qai evolution engine", then "start R4").
- **Context:** the infographic asks for a validation platform for an "orb energy theory": field confinement, resonance, energy retention, information alignment (Rényi D_α), QAI optimisation and direct extraction, through Model → Simulate → Optimize → Validate → Compare → Report with pass / fail / inconclusive. It names no device and no theory beyond its equations (RESEARCH R4).
- **Decision:**
  - The experiment is **ORB-0**: an ideal spherical cavity with finitely conducting walls, one input and one extraction port. It is the simplest system that exercises every equation, and it has closed forms, so every stage can be checked rather than trusted.
  - Add SDK module `qai.orb` (F-24), job and evidence kind `orb` with migration 0011 (F-25), and the `/app/orb` screen with job result panels (F-26). The workflow is W10.
  - The energy ledger ΔE = E_initial + E_supplied − E_extracted − E_lost − E_final is integrated with the mode amplitude and cross-checked against a closed form. Any physics check that fails makes the run `fail`; an optimiser disagreement makes it `inconclusive`, never `pass`. Evidence stores only pass or fail, so inconclusive is stored as fail with the outcome kept in the record.
  - The QAI optimiser is Dürr–Høyer Grover maximum finding with a budget of ⌈8√N⌉ queries by default (≥ 98.5 % success measured). It is compared on oracle queries, not wall-clock time, and the report says when it used no fewer queries than N.
  - The image's min_h D_α(P‖Q_h) s.t. C̃(h) ≤ B is read as the Grover search's alignment with the optimum within the budget. Ψ_{n+1} = G(V(Q(L(Ψ_n, E_n)))) is shown as the Dürr–Høyer rounds, the same propose / evaluate / validate / accept loop as the Evolution Engine (ADR-17).
  - The landing page gains an Evolution Engine section with the loop and its limits (QAOA MaxCut, ≤ 12 nodes, simulator only, 1–20 iterations), and the Optimization screen gains "Load example".
- **Alternatives rejected:**
  - **Meep / FDTD.** No native Windows build, a heavy dependency, and no closed form to check against for the sphere's Q. It may come later as an optional backend for other geometries.
  - **Plasma or "free energy" models.** Not physically grounded; the badges in the image require falsifiable, conservative physics. Extraction is bounded by the supplied energy, and a check enforces it.
  - **Qiskit for the Grover stage.** The SDK's state-vector Grover is already validated, and a check ties the model to its gate-level circuit.
- **Consequences:** `requiredKinds` grows to 9 kinds. "Orb Lab" joins the Execution group of the navigation after AI Simulation. A result is about 90 KB, inside the existing result limits. Releases 0.6.0.

## ADR-47: Orb Lab reports where the optimum was found (RB-14)

**Context.** Review Round 12 (M-R12-1) found that Dürr–Høyer stops only on its budget, so `qai.queries` always equalled the budget and `compare.queryRatio` was always budget / 2^qubits. The "≤ 128 queries" acceptance check and the W10.3 assertion could never fail. The round also found a wrong τ0 hint (M-R12-2) and a stage order that differed between SPEC and ARCHITECTURE §2.9 / the panel labels (M-R12-3).

**Decision.** `durr_hoyer` also returns `foundAt`, the query count at which the final best was first measured (1 when the initial guess was already the best). `optimize.qai.foundAt` and the evidence outputs carry it; `queryRatio` becomes foundAt / 2^qubits; `queries` stays as the budget spent. The pass reason reads "at query N of a B-query budget". The Optimize panel shows both, Compare's hint says "Queries to find the optimum / classical evaluations", and W10.3 asserts foundAt ≤ budget and queryRatio = foundAt / 256. The τ0 hint is Q0/ω, matching the SDK. Stage order is SPEC's: 1 Model, 2 Simulate, 3 Optimize, 4 Validate, 5 Compare, 6 Report; ARCHITECTURE §2.9 and the panel eyebrows now agree.

**Consequences.** Older orb results lack `foundAt`; the panel then shows only the budget spent. No migration: results are JSON.

**Amendment (RB-16, Review Round 14 M-R14-1).** `foundAt` is where the QAI search reached *its own* best, which can miss the classical optimum on a small budget. The Optimize cell therefore says "optimum at query N" only when the QAI best equals the classical best and "best at query N" otherwise (RB-15), and Compare's query-ratio hint reads "Queries to reach the QAI best / classical evaluations" (RB-15's commit message claimed this hint change; RB-16 makes it).

## ADR-48: Re-Entry R5 — Formula Lab: user formulas as falsifiable experiments (safe math grammar, no eval)

- **Status:** accepted (user direction, 2026-11-07: "how do I test my formulas with this app", then "I would like to do this with the website", choosing all four formula kinds, a job with evidence and math notation).
- **Context:** the SDK can already check a formula offline (a Python script against `qai.orb.ledger` or `durr_hoyer`), but the website only runs the built-in experiments. The user wants to type their own formula on the site: an objective to optimise, a physics formula checked against the Orb Lab ledger, an evolution map Ψₙ₊₁ = f(Ψₙ, Eₙ), and a quantum state or circuit. Each run should be a job with a result page and a hashed evidence record. Running user text as code would break INV-10 and the worker's trust model.
- **Decision:**
  - Add `qai.expr` (F-27): a closed grammar (numbers, declared variables, + − * / ^, parentheses, 26 whitelisted NumPy functions, pi, e, and i for amplitudes only), parsed by recursive descent into tuples and evaluated by a fixed dispatch table. There is no `eval`, `exec`, `compile`, attribute access, indexing, strings or implicit multiplication. The limits are 2,000 characters, 12 definitions, 512 terms and depth 32, and depth is checked before recursing.
  - Add `qai.formula` (F-28) with four modes. Each one compares the user's formula with something independent: classical exhaustive search (objective), the numerical energy ledger (physics), the map's own fixed-point and stability conditions (map), and the exact distribution under seeded sampling with a stated statistical bound (state). The outcome rules are those of ADR-46: a physics check failure is `fail`, a comparison check failure is `inconclusive`, and only `pass` is stored as evidence status pass.
  - In physics mode a formula that disagrees with the ledger is `fail` (it has been falsified), not `inconclusive`.
  - Add job and evidence kind `formula` with migration 0012 (F-29), and the `/app/formula` screen with result panels (F-30). The workflow is W11.
  - The web app mirrors the parser in TypeScript (`lib/expr.ts`) for live errors and the objective preview. Shared JSON vectors keep the two parsers in step, and the server's parse is authoritative.
  - The expression is parsed at submit for an early 422. That is O(n) on at most 2,000 characters, so it runs inline and not through `cpu.run_bounded` (ADR-30), which is for compile and QASM.
- **Alternatives rejected:**
  - **Python `eval` with a restricted namespace, `ast` whitelisting, numexpr or SymPy `sympify`.** `eval` and `sympify` run code (`sympify` calls `eval`), `ast` whitelisting still hands untrusted text to Python's parser and is easy to get wrong, and numexpr is a new native dependency. A 300-line grammar is auditable and is the same in Python and TypeScript.
  - **LaTeX input.** It is ambiguous to parse and harder to type. Math notation as plain text was the user's choice.
  - **Vector-valued Ψ for maps.** The map is scalar in R5, which covers the Evolution formula's shape. Vector state can come later through the same grammar with indexed variables.
- **Consequences:** `requiredKinds` grows to 10 kinds. "Formula Lab" joins the Execution group of the navigation after Orb Lab. A result is up to about 90 KB (a 1,000-step trajectory or a 1,024-point landscape). Releases 0.7.0.

## ADR-49: Re-Entry R6 — Evidence protocol for the evolution engine (six falsifiable tests)

- **Status:** accepted (user direction, 2026-11-07: a six-test table — correctness, improvement, baselines, repeatability, generalization, governance — then "go R6").
- **Context:**
  - The evolution engine (§2.6, F-7/F-14) decides each candidate on one problem and one fixed seed set. Its record shows that the gate fired; it does not show that the adopted configuration is actually better.
  - A skeptical reader would ask six things: is the optimum right; does the improvement survive fresh seeds; does it beat random search and a classical optimizer at the same cost; does it repeat; does it hold on instances it was not tuned on; and does the gate refuse bad input.
  - The existing record answers none of them with a statistic.
- **Decision:**
  - Add `gate` and `evaluate_set` to `qai.evolution` (F-31). The gate checks evidence completeness (one record per instance × seed, each with its ratio and checks) before V and J. `decide` is unchanged and is reused for the J rule, so the protocol and the evolution service share one acceptance rule.
  - Add `qai.protocol` (F-32), which runs `runs` (default 30) independently seeded replicates. Each replicate tunes on train, selects on validation (adopting only when the paired bootstrap CI excludes 0), locks a hash of the choice, and evaluates the test set exactly once.
  - The six tests and their pass rules are those of SPEC F-32.
  - Baselines spend the same B = candidates + 1 configuration evaluations (same instances, seeds and shots):
    - random search over the config space;
    - COBYLA over the initial angles.
    Greedy MaxCut is a reference, not a competitor, because it never touches the budget.
  - Outcome rules follow ADR-46:
    - an integrity failure is `fail`: a wrong optimum, unequal budget, nondeterminism, a broken lock, or an accepted injection;
    - a statistic that does not clear its bar is `inconclusive`: the improvement CI includes 0, or runs < 30;
    - six passes are `pass`.
  - Add job and evidence kind `protocol` with migration 0013 (F-33), and the `/app/protocol` screen with result panels (F-34). The workflow is W12.
  - **Compute.** A protocol is long-running by nature: 30 replicates of real QAOA optimisation took 163 s at the defaults on the prototype. It gets its own wall clock, `PROTOCOL_TIMEOUT_SEC` (default 900, 60–3600), instead of raising `JOB_TIMEOUT_SEC` for every kind. A tenant may hold one queued or running protocol job (429 QUEUE_FULL, an existing §7 code), so one tenant cannot hold both worker replicas.
- **Alternatives rejected:**
  - **Fan out each replicate as its own job and aggregate.** It needs a parent/child job relation, a join step and partial-failure rules: a new state machine for a 3-minute computation. One child process with a deadline is simpler, and is deterministic end to end.
  - **Raise `JOB_TIMEOUT_SEC` globally.** That would let every kind run 15 minutes, and weaken S10's bound for jobs that need 5 s.
  - **Shrink the protocol until it fits 120 s.** At 10 replicates, repeatability would not meet the user's 30.
  - **A t-test instead of the bootstrap.** The differences are bounded and skewed (J is in [0, 1], and many runs adopt nothing). The percentile bootstrap makes no normality assumption, and it is seeded, so it is reproducible.
  - **Writing protocol decisions to `evolution_decisions`.** Those rows record a user's evolution run. A protocol's thousands of internal decisions belong in its own hashed evidence.
- **Consequences:**
  - `requiredKinds` grows to 11 kinds.
  - "Evidence Protocol" joins the Execution group of the navigation after Formula Lab.
  - A result is up to about 120 KB at 30 runs.
  - The protocol reports honestly when the engine loses. On the prototype, greedy MaxCut beats the 8-node QAOA engine (J 0.05 against 0.16), and the panels show it.
  - Releases 0.8.0.

## ADR-50: Governance injections need a strict margin; the protocol slot holds for requeue (RB-21)
- **Status:** Accepted (Review Round 20: N-R20-1, N-R20-2 and N-R19-3).
- **Context:**
  - The governance test's "worse" injection is Ψ₀ scored on the train set.
  - When no run adopts a checkpoint, the checkpoint is Ψ₀ itself, so the two J values are equal. A checkpoint adopted at ε = 0 can also tie Ψ₀.
  - The gate rule `J(S) ≤ J(Ψn) − ε` accepts an equal J at ε = 0. Governance then reported fail, and the whole protocol failed, even though the gate behaved exactly as specified. Seed 2 reproduces it (nodes 6, runs 2, candidates 1, baseline p 2 / maxiter 60, ε 0).
  - RB-20 made requeue honour the one-protocol-per-tenant slot (M-R19-1), but the endpoint table did not list the 429.
- **Decision:**
  - The governance injections run the real `gate` with ε_g = max(ε, 1e-9). A tie is "not better" and is rejected with the gate's own `is not <=` reason. With ε > 0, ε_g is ε, so nothing else changes.
  - The evolution search's own gate keeps the configured ε, 0 included.
  - SPEC §4 lists 429 QUEUE_FULL with `Retry-After: 60` on requeue of a protocol job.
  - W12.1 states the order the e2e uses: the first protocol is posted through the API, then the UI submit meets it.
  - RB-22 (Round 21, N-R21-1/N-R21-2): the SPEC §5 result schema lists `epsilonGate` on each governance injection, and §3 and F-33 (rule and AC) name the requeue path for the one-protocol-per-tenant 429.
- **Alternatives rejected:**
  - **A different "worse" candidate (for example, zeroed angles).** It is usually worse but not provably so, and the test would become flaky on some graphs.
  - **A minimum ε of 0.001 for the protocol.** It would change what the user's ε means for the search too, not only for the injections.
- **Consequences:**
  - At ε = 0 the governance test can still only fail on a real gate defect.
  - The result records both `epsilon` and `epsilonGate`.

## ADR-51: Formula Lab reads Ψ notation (Re-Entry R7)
- **Status:** Accepted (user direction, 2026-11-08: "Line 1, column 1: unexpected character 'Ψ'", then "Yes, make it accept Ψ").
- **Context:**
  - The landing page and SPEC write the map as Ψₙ₊₁ = f(Ψₙ, Eₙ, n). A user who pasted that notation into Formula Lab got a lexer error at column 1, because names were ASCII only and the result line could not carry a left-hand side.
- **Decision:**
  - `Ψ` and `ψ` lex as the name `psi`. A name may end in the subscript `ₙ`, which is dropped, so `Ψₙ` is `psi` and `Eₙ` is `E`. No other non-ASCII character is accepted, and other subscripts stay errors, so `x₁` and `x₂` cannot silently collapse into one name.
  - `parse` takes `result_labels`. When the last line begins with one of them followed by `=`, the label is skipped. Map mode declares `Ψₙ₊₁` and `ψₙ₊₁` for the expression and `Eₙ` and `E` for the energy; the other modes declare none.
  - The grammar stays closed (INV-10): this is a lexer alias and a fixed prefix match, not new evaluation. Error columns still count characters of the source, and `canonical()` renders ASCII without the label, so stored formulas and evidence are unchanged. The web mirror (F-30) does the same and the shared vectors cover it.
- **Alternatives:**
  - **Normalise the text before parsing (replace Ψ with psi).** It shifts every later error column.
  - **Accept any Unicode identifier.** It widens the grammar for no user need and opens confusable names.
- **Consequences:**
  - Formulas copied from the Evolution Engine notation run in Map mode as written. Every formula that parsed before parses to the same canonical form.

## ADR-52: The map's result line may use arrow notation (Re-Entry R7)
- **Status:** Accepted (user direction, 2026-11-08: "Line 1, column 4: unexpected character '→'", then "Yes, make it accept the arrow").
- **Context:**
  - After ADR-51, Map mode read `Ψₙ₊₁ = f(Ψₙ, Eₙ, n)`, but maps are just as often written as `Ψₙ → f(Ψₙ, Eₙ, n)` or `Ψ ↦ f(Ψ)`. The user's arrow form failed in the lexer.
- **Decision:**
  - A result label is a left-hand side together with its separator. Map mode accepts `Ψₙ₊₁ =` and `ψₙ₊₁ =` (the next state equals …), and `Ψₙ →`, `ψₙ →`, `Ψ →`, `ψ →` and `psi →` (the current state maps to …). `↦` and the ASCII `->` are read as `→`. The energy keeps `Eₙ =` and `E =`.
  - The separator must match the left-hand side, so `Ψₙ₊₁ → …` and `Ψₙ = …` stay errors: one reads as a step of the next state, the other as redefining the current one.
  - Arrows are only a prefix on the last line. They are not operators, and anywhere else they stay an unexpected character. The grammar stays closed (INV-10) and `canonical()` is unchanged.
- **Alternatives:**
  - **Accept any separator after any label.** `Ψₙ = cos(Ψₙ)` looks like an equation in Ψₙ, not a map, so it would hide a likely mistake.
  - **Make `→` an operator.** It has no meaning inside an expression.
- **Consequences:**
  - The labels passed to `parse` change shape (`"Ψₙ₊₁ ="` instead of `"Ψₙ₊₁"`). Only the map uses them, and the SDK, the web mirror and the shared vectors change together.

## ADR-53: Formulas accept square brackets: the index `[n]` and grouping (Re-Entry R7)
- **Status:** Accepted (user direction, 2026-11-08: "Line 1, column 2: unexpected character '['", then "make it accept the brackets").
- **Context:**
  - Maps are often written with index brackets, `Ψ[n+1] = f(Ψ[n], E[n], n)`, and some sources group with `[ ]`. Both failed in the lexer at the first `[`.
- **Decision:**
  - A name may end in `[n]`, with spaces allowed inside, and it is dropped like the subscript `ₙ` (ADR-51): `Ψ[n]` is `psi`, `E[n]` is `E`. It is notation, not indexing: any other index (`x[0]`, `Ψ[n-1]`) is an error, "expected an operator before '['".
  - `[ ]` groups exactly like `( )`, and a `[` must be closed by `]` ("expected ']'"). Function calls still use `( )`.
  - Result labels are compared after normalising both the source's left-hand side and the declared labels: `[n+1]` reads as `ₙ₊₁`, and a trailing `[n]` or `ₙ` is dropped. So `Ψ[n+1] =`, `psi[n+1] =`, `Ψ[n] →`, `psiₙ →` and `E[n] =` are accepted. Map mode adds `psiₙ₊₁ =`, the ASCII next state. The separator must still match the left-hand side (ADR-52), so `Ψ[n] = …` is rejected.
  - The grammar stays closed (INV-10). `canonical()` is unchanged: it renders ASCII names and round brackets.
- **Alternatives:**
  - **Accept any index and ignore it.** `x[0]` would silently read as `x`, which hides a real mistake.
  - **Real indexing.** The language has no arrays, and ARCHITECTURE keeps "no indexing".
- **Consequences:**
  - The one shared vector about brackets, `x[0]`, now reports "expected an operator before '['" at the same column 2, instead of "unexpected character '['". Every formula that parsed before parses to the same canonical form. The SDK, the web mirror and the shared vectors change together.

## ADR-54: Validation Lab phase A: release decisions, the versioned active Ψ and the audit hash chain (Re-Entry R8)
- **Status:** Accepted (user direction, 2026-11-08: "injest this MD file and make enhancements accordinly" with the Validation Lab V2.0 build context, then "go" on the proposed phase A, the role mapping and the size limits).
- **Context:**
  - The context file asks for a Max-Cut validation lab whose governed state Ψ is adopted only through independent validation and an authorised decision (`APPROVE`, `REJECT` or `HOLD`), promoted atomically with a rollback pointer, and recorded in a hash-verifiable event history (its §8, §13, §14 and acceptance tests 5–10 and 13).
  - The repo already covers much of it: exact Max-Cut scoring and matched baselines (F-32), paired bootstrap CIs, disjoint train / validation / test with a test-once lock, the adoption gate's rejection of worse, invalid and missing candidates (ADR-49), per-record sha256 evidence (ADR-16), and reviewer approval that blocks the author (F-15).
  - Missing were: reject and hold decisions with a rationale, a versioned active state with promotion and rollback, and a chain over the event history.
- **Decision:**
  - Phase A extends the release candidates (F-15) instead of adding a parallel experiment-decision model. A release may carry a Ψ snapshot (an evolution run's best checkpoint with its sha256). Hold and reject are reviewer decisions with a required reason; rejected is terminal, and held resumes by evaluating.
  - Releasing promotes: it appends an `engine_switches` row in the same transaction. The active Ψ is the latest switch; rollback appends a `rolled_back` switch to an earlier released version with a reason. The list is append-only, so the rollback pointer is the history itself.
  - "Duplicate promotion requests are idempotent" is met as "exactly once, no second effect": a repeated release request returns 409 INVALID_STATE as it does today and writes nothing. Changing it to 200 would break the published contract (W5).
  - The audit table gets a per-tenant hash chain set by a database trigger, so every insert path is chained (requests, the worker, the scheduler), and an admin verify endpoint recomputes it.
  - Roles map onto SPEC §4 S7 with no new role: observer = viewer, researcher = member, reviewer = member + reviewer capability, admin = admin or owner. The context's "validator" is the server: the worker recomputes objectives and the governance service re-verifies evidence; letting a person attest validation would contradict the context's own "never allow a proposer to self-attest" rule. Rollback needs an admin who also holds the reviewer capability.
- **Conflicts with the context file, resolved:**
  - **Sizes.** The context's 15- and 20-vertex graphs exceed what this simulator runs: QAOA ≤ 12 nodes, the exact solver ≤ 16, the protocol 6–10. The limits stay (ARCHITECTURE §5, ADR-49), and reports state the sizes used.
  - **Qiskit.** It stays the independent reference adapter only (INV-17), not the QAOA engine.
  - **Proprietary engine.** There is none in the repo: the evolution engine is the open `qai` SDK. Its checkpoint is the Ψ.
  - **Routes.** The context's `/experiments`, `/runs` and `/versions` map onto the existing `/jobs`, `/evolution/runs` and `/releases`, plus `/projects/{id}/engine`. No duplicate route families.
  - **Hardware.** Every result is noiseless state-vector simulation. Labelling that on every report is deferred to a later phase with the report bundle.
  - **Wording.** ADR-48 and ADR-49 called evidence "signed". It is hashed (sha256), not signed; both are corrected here, and the chain is described as "hash chain verified", never "tamper-proof".
- **Alternatives:**
  - **A new experiment / decision state machine** (`DRAFT → … → PROMOTED / ROLLED_BACK`). It would duplicate jobs, evidence and releases, which already hold those states, and split governance across two writers.
  - **A mutable pointer column on projects.** It would lose the history that rollback needs, and an UPDATE cannot be made append-only.
  - **Hashing in the application.** Paths that insert audit rows outside `services.audit` would escape the chain; the trigger cannot be skipped by the app role.
  - **Signing or external anchoring.** It needs key management or an external immutable store; out of scope for a local stack. The limit is stated instead.
- **Consequences:**
  - New statuses `held` and `rejected` (INV-35, INV-36); migration 0014 adds them, the Ψ columns, the decision columns and `engine_switches` (INV-37); migration 0015 adds the chain (INV-38). 0014's downgrade refuses while its data exists; 0015's downgrade drops the derived chain (review round 29, N-R29-2).
  - Every transaction takes its row locks before its first audit insert, because the chain's advisory lock is held until commit (review round 29, N-R29-1; ARCHITECTURE §2.12).
  - Audit inserts in one tenant are serialised from the insert to commit. Requests in different tenants do not contend.
  - Deferred to later phases, each with its own SPEC change: the artifact bundle and Markdown report with the code commit, more baselines and the -L/-Q/-V ablations, a standalone bitstring scorer, multiple-comparison correction and effect sizes, the simulation label on reports, and a VALIDATION LAB navigation group.

## ADR-55: Validation Lab phase B: scorer, more baselines and ablations, corrected statistics, the execution label, the report bundle and the VALIDATION LAB group (Re-Entry R9)
- **Status:** Accepted (user direction, 2026-11-11: "go phase B" for the six items ADR-54 deferred, then `extend 24h`).
- **Context:** the Validation Lab context file (§12, §14, acceptance tests 1–3, 11, 14 and 15) asks for an independent bitstring scorer, seeded weighted instances, stronger classical baselines and the -L/-Q/-V/-G ablations, effect sizes with a multiple-comparison adjustment, results labelled noiseless, noisy or hardware, an artifact bundle with a Markdown report and a manifest, the code commit in every record, and a VALIDATION LAB menu. ADR-54 deferred all six.
- **Decision:**
  - **Scorer and instances (F-42).** `qai.algorithms.maxcut` adds `score_cut(graph, bitstring)`. It uses the SDK's bitstring order (highest node leftmost, `int(bitstring, 2)` is the assignment) and rejects a wrong length or a character other than 0 and 1 with a `bitstring` path. It also adds `weighted_graph(numNodes, edgeProbability, seed, maxWeight)`, a connected graph with integer weights from a seed, and `instance_id(graph)`, the sha256 of the canonical graph. The protocol's correctness test rescores the best sampled bitstring of every test record with `score_cut` and requires it to equal the reported value.
  - **Classical baselines (F-43).** Random bitstring search, local search with restarts (steepest 1-flip ascent from random starts) and simulated annealing run on the same frozen test instances and seeds. Each one spends exactly `classicalEvaluations` objective evaluations per instance and seed (new param, 8–1024, default 64). That budget is declared separately from the QAOA configuration budget B, because a cut evaluation and a QAOA configuration evaluation are not equivalent (context §12). The classical metric is the best-found gap 1 − C/C*. It is compared, paired by run, against the evolved method's best-sampled gap, never against J.
  - **Ablations (F-43).** These are offline arms inside the protocol. None of them touches `gate`, the release flow or production approval (ADR-49):
    - **-L (`noLearner`):** random configurations replace `propose`, through the same gate and the same validation selection. It has its own B train evaluations.
    - **-V (`noValidation`):** the last accepted train checkpoint, without the validation selection.
    - **-G (`ungated`):** the last configuration the engine proposed, adopted without the gate or validation. Offline only.
    - **-Q:** there is no quantum-free QAI loop to ablate into. The classical arms stand in for it, and the result says so.
    - -V and -G reuse the evolved arm's B train evaluations. All QAOA arms are evaluated in the run's single test evaluation, so the test-once count stays 1.
  - **Statistics (F-43).**
    - Every comparison with the evolved method (random, COBYLA, -L, -V, -G and the three classical arms) reports the paired difference CI, a two-sided sign-flip permutation p (2,000 seeded flips), a Holm-adjusted p over the whole family, and the paired effect size d_z = mean / sd (null when sd is 0).
    - A verdict is better or worse only when the CI excludes 0 and the adjusted p is < 0.05. Otherwise it is indistinguishable.
    - Every new draw comes from its own SeedSequence stream, so the existing arms' numbers are unchanged.
  - **Execution label (F-43, F-45).** The protocol result and its evidence outputs carry `execution {mode: simulation, noise: noiseless, backend: qai.statevector, hardware: false}`. Every job result page and the report state "Noiseless state-vector simulation". No noisy simulator or hardware backend exists (SPEC §1 non-goal), so no other label is ever emitted.
  - **Code commit (F-44).** `current_env()` adds `commit`: `GIT_COMMIT` when it is 7 to 40 lowercase hex characters, otherwise `unknown`. The worker passes `GIT_COMMIT` to the child. Existing evidence still verifies against its stored env.
  - **Report bundle (F-44).** `GET /api/v1/evidence/{id}/report` (session, member+, audited `export.evidence_report`):
    - `format=md` returns `report.md`.
    - `format=json` returns the bundle: `config.json`, `outputs.json`, `environment.json` and `report.md`, plus `baselines.json`, `stats.json`, `validation.json` and `decision.json` for protocol evidence with a job result. A `manifest` lists each file's sha256 and size, the evidence sha256, and whether the record verifies.
    - The report states the execution label, the sizes used and the verification result. Its text is deterministic: the generation time lives only in the manifest, outside every hashed file.
  - **Navigation (F-45).** The Verification group becomes Validation Lab: Evidence Protocol, Validation, Benchmarks, Optimization and Governance. Pages are regrouped, not added.
- **Conflicts with the context file, resolved:**
  - **Sizes.** The 15- and 20-vertex instances stay out (ADR-54): the protocol is 6–10 nodes, and the report states the sizes.
  - **Weights.** The protocol keeps unweighted G(n, p) instances, so its numbers stay comparable with earlier evidence. Weighted instances are available through the SDK scorer and registry.
  - **QAOA depth 1–4 sweep and equal-wall-clock comparisons.** Not added. Random search already spans p 1–3, and wall-clock on a shared host is not repeatable. The report lists both as not measured.
  - **Signing.** The manifest is hashed, not signed. The report says "hash chain verified" and never "tamper-proof" (ADR-54).
- **Alternatives:**
  - **A zip bundle.** It would need a new dependency on the browser side. A JSON bundle with a manifest is just as verifiable.
  - **A Bonferroni adjustment.** It is uniformly less powerful than Holm, and both control the family-wise error rate.
  - **Comparing classical best-found cuts with QAOA J.** That would mix an expected value with a maximum. They are compared on the best-found gap only.
- **Consequences:**
  - The protocol takes about 30 % longer: the -L arm adds B train evaluations, and the extra arms add test evaluations. The defaults stay within `PROTOCOL_TIMEOUT_SEC` 900.
  - Protocol evidence produced after this change has a new `outputs` shape (execution, more methods, corrected statistics), so its sha256 differs from that of an earlier record with the same params.
  - New evidence env has a `commit` key.

## ADR-56: Landing and app Lighthouse performance accepted as host-bound residuals (Re-Entry R9)

- **Context.** Three `--full` web baselines on 2026-11-11 (at 4a40598 twice, and at 9b87fda after RB-34) pass every cell except mobile Lighthouse performance:

  | Screen | Performance (median of 3) | Mobile LCP | Gate |
  |---|---|---|---|
  | landing | 63, 63, 70 | 3.6, 3.6, 3.4 s | ≥ 80, ≤ 2.5 s |
  | app | 70, 67, 67 | 2.7, 2.8, 2.7 s | ≥ 80, ≤ 2.5 s |

  Time to first byte is about 30 ms. LCP equals FCP, and almost all of it is render delay from roughly 4 s of main-thread work under DevTools 4x CPU throttling, which multiplies host contention. An isolated earlier measure scored 83. RB-34 cut landing Style & Layout from 1.9 s to 1.2 s.
- **Decision** (the user, 2026-11-11, "Accept via ADR"). INV-2 accepts `landing/lighthouse` and `app/lighthouse` as host-bound residuals, through `invariants.json` `accepted` entries that cite this ADR. Accessibility, best practices and SEO stay gated. The acceptance is reviewed on the next quiet-host run or in CI.
- **Rejected alternatives.**
  - Removing the header backdrop-filter, a visible design change, for an uncertain gain.
  - Lowering the threshold in the pinned runner, which must never be edited.
