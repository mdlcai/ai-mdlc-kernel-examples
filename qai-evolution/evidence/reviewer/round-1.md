Commit reviewed: c019a8ea84f285a79a1d0fbb714fc2c84f2df44f

# Reviewer Gate: Review Round 1

- **Mechanism:** a fresh-context Task-tool subagent. Static review only. No e2e, docker or web baseline was run, and no containers were touched.
- **Inputs:** RESEARCH.md, SPEC.md, ARCHITECTURE.md, DECISIONS.md, BUILD-PLAN.md, invariants.json, GOVERNANCE-DECISIONS.jsonl (focus: GD-0001/0002/0008/0010/0012/0015/0016/0017/0018/0019/0020/0022), and the source under packages/qai-sdk, services/api and apps/web.
- **Taken as established, not re-run:**
  - full e2e 40/40;
  - API tests 272/272 (91% coverage);
  - SDK tests 195;
  - web vitest 109;
  - typecheck, lint and ruff clean;
  - invariant lint 31/32. INV-2, the web baseline, is pending a host CA-trust step and is not counted as a finding.

## Checklist

| # | Item | Verdict | Rationale and evidence |
|---|------|---------|------------------------|
| 1 | Every RESEARCH requirement is implemented and traceable, with no stubs | PASS | The core features are all real implementations: the simulator (qai/simulator.py), compile/peephole (qai/compile.py), validation with an optional Qiskit reference (qai/validation/checks.py:152), the algorithms (qai/algorithms/qaoa.py:1,44-46 COBYLA with exact and greedy baselines), the evolution gate (qai/evolution.py decide()), the job queue (services/jobs.py), the scheduler (scheduler/triggers.py), governance (services/governance.py) and the exports (routes/audit.py:108, routes/export_evidence.py:57). |
| 2 | Every SPEC feature has code and behavioural tests | PASS | All §3 endpoints exist as routes, including /admin/scheduler/{trigger}/run (routes/schedules.py:267), /scheduler/runs (:248) and the two export routes. F-13 has DB-effect tests (tests/test_scheduler.py:69,99,229,247,294). The suites listed above are green. |
| 3 | Implementation matches ARCHITECTURE: no shadow modules, no framework swap | PASS | The stack matches: FastAPI, async SQLAlchemy and Alembic, Next.js and Caddy. The worker and scheduler sit in `qai_api/worker/` and `qai_api/scheduler/`, not in `qai_api/jobs/worker.py` and `jobs/scheduler.py` as ARCHITECTURE.md:56-57 says. That layout is sanctioned by BUILD-PLAN.md:224,278 (`worker/**`, `scheduler/**`), so this is stale documentation (LOW note), not a shadow module. |
| 4 | Scope discipline: no invented features | PASS | The only addition beyond the SPEC body is the optional `rngSeed` on POST /evolution/runs (routes/evolution.py). It is a reproducibility knob and does not let a client supply scores (S14 holds, because the worker recomputes V and J; see repositories/evolution.py verify_outcome). LOW note. |
| 5 | Security invariants upheld | PASS | CSRF uses both an Origin check and a double-submit token on unsafe session requests (deps.py:69-74,101-107). API keys use a constant-time hash compare, revoked keys get 401, and key role is capped at admin (deps.py:117-139). Admin-only management refuses API keys (deps.py:176-181). The last-owner guard returns 409 (routes/members.py:121,173). Self-approval is blocked and approve is member+, session-only and reviewer-only (services/governance.py approve(), routes/releases.py). Audit export is admin+ and itself audited (routes/audit.py:134-143). The overview cache key is tenant-scoped (services/overview_cache.py:31-36). |
| 6 | Cold read: no obvious bugs, dead code or broken wiring | **FAIL** | (a) `services/jobs.py:430` `recover_expired_leases()` is a second, divergent T2 implementation that production never calls; only tests/test_worker.py:274,282 use it. Its module docstring (services/jobs.py:16) wrongly names it as the T2 transition. (b) The real T2 (scheduler/triggers.py:222-244) moves jobs to `dead` without invalidating the overview cache. The dead copy does invalidate, and `requeue()` (services/jobs.py:290) does not. See F-R1-1 and F-R1-2. |
| 7 | Runnable (README/QUICKSTART) | PASS | README.md:27-30 gives `docker compose up -d --build` and https://localhost:8443, and links QUICKSTART.md for TLS, ports and troubleshooting. The test commands are at README.md:50. |
| 8 | Every manual invariant upheld | PASS | INV-8 (scheduler T1–T3 effect tests) is the only manual invariant. Verdict below. |
| 9 | Nothing declared is unreachable | PASS | Every job state has a production writer: queued (submit, fail-retry, T2), running (claim), succeeded (complete), failed (fail non-retryable), dead (fail exhausted, T2), cancelled (cancel). Every release state has a writer in services/governance.py: pending (create), ready/blocked (`_apply`, on_evidence_written :465), approved (approve), released (release). Evolution accepted/rejected is written in qai/evolution.py decide() and persisted by repositories/evolution.py record_outcome. Evidence `skipped` comes from qai/validation/checks.py:152. Each of these is exercised by API or e2e tests. |

## Manual invariants

| Invariant | Verdict | Evidence |
|-----------|---------|----------|
| INV-8: scheduler T1–T3 have effect tests | PASS | The tests assert real DB effects rather than mocks. `test_t1_enqueues_benchmark_job` (tests/test_scheduler.py:69) checks that the job row is enqueued and next_run_at advances. `test_t1_idempotent_same_window` (:99) checks the UNIQUE claim. `test_t2_requeues_expired_lease` (:229) and `test_t2_dead_after_max_attempts` (:247) check the status, error code and backoff. `test_t3_deletes_old_results_keeps_evidence_and_audit` (:294) checks retention. Per-tenant, UTC, coalescing, no-overlap, QUEUE_FULL rollback and SIGTERM cases are also covered. |

## Governance focus items

| GD | Feature | Verdict | Rationale |
|----|---------|---------|-----------|
| GD-0001 | F-1 SDK circuit model | PASS | qai/circuit.py enforces the gate allow-list, arity, finite params, qubit range, distinct qubits and terminal measures. from_dict reports field paths. Limits are 20 qubits and 2000 ops. |
| GD-0002 | F-2 simulator | PASS | qai/simulator.py does tensordot little-endian application, seeded PCG64 multinomial sampling, and keys counts by measured qubits. |
| GD-0008 | F-5 algorithms | PASS | qai/algorithms/qaoa.py:1,17,44-77 runs seeded COBYLA and reports approximationRatio against both the exact optimum and greedy_maxcut (baselines.py). grover.py is present. |
| GD-0010 | F-7 compile/peephole | PASS | qai/compile.py decomposes swap, ccx (Clifford+T) and u, then cancels inverse pairs and merges rotations. `_blocked_after` is redundant but harmless (no finding). |
| GD-0012 | F-9 members, invites, keys, audit | PASS | Member, invite and key routes use `UserAdmin` (routes/api_keys.py:68,79,122; routes/invites.py:110,121,165). The last-owner 409 is at routes/members.py:121,173, and the "no role above your own" check is at :117. Removing a member revokes the keys they created and audits each one (:180-200). The audit list and export filter by actorKind (routes/audit.py:27,84,113). |
| GD-0015 | F-13 placeholder migrations 0006–0008 | PASS | 0006_schedules.py, 0007_evolution.py and 0008_releases.py now contain real `upgrade()` bodies (:37/:41/:42) and `downgrade()` (:80/:110/:77). The FKs are SET NULL as designed (scheduled_runs.job_id and evolution_runs.job_id). |
| GD-0016 | F-11 algorithm result panels | PASS | apps/web/components/results/algorithm-panels.tsx:245-289 renders approximationRatio, the ratio against greedy and the greedy baseline row. There is a colocated test (algorithm-panels.test.tsx). |
| GD-0017 | F-12 evidence and reference | PASS | The validation evidence `passed` value comes from the report status (worker/runners.py:234-245). Reference evidence is emitted only when the reference is available (:247-256). Otherwise the check is `skipped: reference extra not installed` (checks.py:152) and does not count as a pass (checks.py:61). The evidence export is member+, session-only and audited (routes/export_evidence.py:5,35). |
| GD-0018 | F-13 scheduler | PASS | T1, T2 and T3 behave as specified (scheduler/triggers.py). Note F-R1-2: the T2 dead path skips the overview invalidation. |
| GD-0019 | F-14 evolution API | PASS | Request models are extra="forbid", budgets stay below JOB_TIMEOUT, and the server re-verifies outcomes (repositories/evolution.py verify_outcome). Observation: GET list and detail call `sync_run`, which writes under FOR UPDATE. These are side-effecting GETs (design note, no finding). |
| GD-0020 | F-15 governance/releases | PASS | Release status has a single writer, evidence is re-verified by hash at evaluate, approve and release, failing evidence blocks ready or approved releases, self-approval is refused, and release requires approved (409 NOT_APPROVED). |
| GD-0022 | F-16 overview | PASS | The 30 s cache is keyed by tenant and generation, Valkey failures fall back to the DB, and queueDepth is the tenant's own count (repositories/overview.py). The invalidation gaps are in F-R1-2. |

## Findings

### F-R1-1: duplicate, divergent T2 watchdog path (dead production code) [FAIL, item 6]
- **Severity:** LOW
- **Where:** services/api/qai_api/services/jobs.py:430-455 and the docstring at services/jobs.py:16. Its only callers are services/api/tests/test_worker.py:274,282.
- **What's wrong:** `recover_expired_leases()` reimplements the lease-expiry transition, but production never calls it. The production T2 is `scheduler/triggers.py:222` `_t2_tenant`. The two copies diverge:
  - The dead copy writes error `"The worker stopped renewing its lease"`. Production writes the SPEC string `"lease expired"` (triggers.py:56).
  - The dead copy runs under `system_session()` across all tenants. Production runs per tenant under `tenant_session`.
  - The dead copy invalidates the overview cache. Production does not.
  - The docstring calls the dead copy "the lease-expiry transition for the F-13 T2 watchdog", which is false.
- **Failure scenario:** `test_expired_leases_are_recovered` gives false coverage, because it passes while testing code production never runs. A maintainer who follows the docstring could fix a T2 bug in the dead copy, see green tests, and ship with the real watchdog still broken. A later caller of the dead copy would also put a non-SPEC error message in front of users.
- **Suggested fix:** Delete `recover_expired_leases()`, fix the docstring at jobs.py:16, and point test_worker.py:268-282 at `scheduler.triggers.run_t2(now)`, or drop that test since test_scheduler.py:229,247 already cover T2.

### F-R1-2: overview cache not invalidated on T2 dead or on requeue [FAIL, item 6]
- **Severity:** LOW
- **Where:**
  - services/api/qai_api/scheduler/triggers.py:239-241: the T2 dead branch has no `overview_cache.invalidate`.
  - services/api/qai_api/services/jobs.py:290: `requeue()` has no invalidation, unlike `cancel` and `fail` (jobs.py:424-425).
- **What's wrong:** routes/overview.py:5 and services/overview_cache.py say the cache generation is bumped when a job reaches a terminal state. The watchdog's dead transition is terminal but does not bump it. Requeue moves a dead or failed job back into the queued counts and does not bump it either.
- **Failure scenario:** A worker crashes on a job's third attempt and T2 marks the job dead. For up to 30 s, the /app overview shows the job as running and the dead count unchanged. An admin requeues a dead job, and for up to 30 s the dashboard still counts it as dead with queueDepth unchanged. The e2e flows that read the dashboard right after these actions would flake. The staleness is bounded by the TTL, hence LOW.
- **Suggested fix:** Collect the tenant ids whose jobs went dead in `_t2_tenant`, and call `await overview_cache.invalidate(tenant_id)` after the session commits. Add the same call after commit in `requeue()`. Add one assertion each in test_scheduler.py and test_jobs.py that `gen_key` was incremented.

### Non-blocking notes (no verdict impact)
- **N-1 (doc):** ARCHITECTURE.md:56-57 gives the worker and scheduler paths as `qai_api/jobs/worker.py` and `qai_api/jobs/scheduler.py`. The real paths are `qai_api/worker/` and `qai_api/scheduler/`, as BUILD-PLAN.md:224,278 says. ARCHITECTURE §2.3 also mentions `POST /projects/{id}/runs`, where SPEC and the code use `POST /api/v1/jobs`.
- **N-2 (doc):** ARCHITECTURE.md:94 says "fixed 30-seed protocol. Held-out seeds are stored server-side". SPEC.md:128,269 lets the client pick 5–30 seeds and derives them from `rng_seed`, and the client may supply `rngSeed`. Align the architecture text with SPEC. S14 is still upheld because scores are recomputed server-side.
- **N-3 (design):** GET /evolution/runs and GET /evolution/runs/{id} call `repo.sync_run`, which locks the row and can write (for example, it can mark a run EXPIRED). This works, but a viewer GET takes row locks and writes. Consider moving the sync into the worker's completion path and T3, which ADR-22 already partly does.

## Overall verdict

**FAIL.** Item 6 is FAIL with two LOW findings, F-R1-1 and F-R1-2. Every other item, INV-8 and all twelve GD focus items are PASS.
