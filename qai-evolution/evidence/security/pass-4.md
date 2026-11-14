# Security Audit — Pass 4

Project: QAI // QUANTUM SDK (ce0ff2f1-2252-49ea-8d2e-5a25f2e08102)
Auditor: fresh-context Security auditor, pass 4
Date: 2026-11-01
Tree audited: HEAD `c52bf7b` (remediation commit `2d2ba1a`, RB-4, ADR-32)
Verdict: **PASS-ADVISORY.** P3-1 to P3-5 are fixed at the root, and I found no regressions.

- **P3-1 (high) is closed.** The log redactor now costs at most about 38 ms on any single string, against 42 s before. The only attacker-chosen access-line field that reaches it unauthenticated is `method`, which h11 limits to tchar, and that case costs about 9 ms.
- **CPU slots (P3-2, P3-3).** They are per tenant, they are leak-free under exceptions and cancellation, and they stay bounded at two threads. A tenant's compile CPU budget dropped from about 70 to about 8 CPU-s/min.
- **Key cap (P3-4).** The key-cap lock sits in the same transaction as the count and the insert.
- **Rate windows (P3-5).** The windows are in the right order.
- **Compile and QASM 404 path.** A cross-tenant id still returns 404 and costs no compile budget.

The new findings are two lows and two infos:

- A small window between the slot check and its acquisition, where the budget is charged and the caller still gets 503.
- A remaining cross-tenant contention residual, which now needs about 15–30 workspaces.
- Redaction gaps caused by the bounded quantifiers and the truncation.
- Test-coverage notes.

The standing advisory from passes 1–3 still applies: there is no tooling for secret detection or TypeScript SAST.

## Scope and method

- I read `git show 2d2ba1a -- services`, ADR-32, the pass-3 report, and the current `logging.py`, `middleware.py`, `errors.py`, `services/cpu.py`, `services/ratelimit.py`, `routes/circuits.py`, `routes/api_keys.py`, `repositories/access.py` and `db/scope.py`.
- I probed in-process only. As instructed, I ran no pytest against the shared test database and did not touch any container or the live stack. The scripts are in [local scratchpad path redacted]:
  - `p4_redos.py`: 22 adversarial inputs through `redact()`, plus per-pattern timings.
  - `p4_cpu.py`: `run_bounded` under exceptions, cancelled callers and 40 contending tenants with 12 cancel-and-retry cycles. It also measures loop latency with 1 and 2 cap compiles, and the race between `available()` and `run_bounded`.
  - `p4_cost.py`: compile CPU per budget unit at each `compile_cost` step.
  - `p4_routes.py`: the real `compile_circuit` and `circuit_qasm` handlers, with a fake repository and session that mimic the tenant filter and the `limits` MemoryStorage window. They check the 404 path, the 503 path and the order of the rate windows.
- I made no source edits and no commits.

| Category | Tool | Result |
|---|---|---|
| ReDoS timing | `p4_redos.py` | Worst case is **37.5 ms** for a 16 KiB string (`://`×5461, `_DSN_CREDS`). `_EMAIL` worst is 26 ms. A 1 MB `://:` input takes 29 ms because it is truncated to 16 KiB first. `/api/v1/`+`a`×16000 takes 9.3 ms (was 1022 ms). `://:`×4000 takes 27 ms (was 509 ms). |
| CPU slot behaviour | `p4_cpu.py` | Exceptions propagate and free the slot. A cancelled caller keeps its slot until the thread ends. A cancelled caller whose fn raises frees the slot and reports no "exception never retrieved". 40 tenants with 12 cancel-and-retry cycles gave **at most 2 threads**, and `_inflight` was empty afterwards. Loop 1 ms-sleep latency is p50 31 ms / p99 72 ms with 1 compile, and p50 32 ms / p99 78 ms with 2 compiles, so there is no loop regression. |
| Fairness | `p4_cost.py` | Per tenant, compile CPU is at most **8.1 CPU-s/min** across the cluster at any op count. A 749-ccx circuit, at cost 3, is the worst case. It was about 70 CPU-s/min in pass 3. |
| Route ordering | `p4_routes.py` | 50 foreign ids and 1 malformed id gave 404 ×51 with both windows untouched (120/30 remaining). Busy gave 503 with the window untouched. An own 2000-op circuit gave 200 with a tenant charge of 9. 100 calls from one principal gave 30 ok and 70 × 429, leaving the tenant with **90** (30 consumed, not 100). |
| SAST (Python) | `ruff check --select S,ASYNC,RUF006` on qai_api and qai | All checks passed. |
| Dependency vulns | not re-run | `2d2ba1a` changes no dependency manifest. The pass-2 results still apply. |
| SAST (TS) / secrets | none | **ADVISORY ONLY — no external tooling available for SAST (TypeScript/web) or secret detection. Agent self-review is not a substitute for automated scanning.** |

## Findings (new in pass 4)

| id | severity | file:line | description | exploit scenario | fix | confidence |
|---|---|---|---|---|---|---|
| P4-1 | low | services/api/qai_api/routes/circuits.py:133-143, 149-154; services/cpu.py:28-35 | **The slot is checked, then acquired later; a request that loses the slot in between pays and still gets 503.** `available(tenant)` runs before `_load` (a DB round-trip) and `enforce_compile` (two Valkey round-trips). `run_bounded` re-checks synchronously, so the bound itself holds (max 2 threads, measured). But a request can pass the first check, lose the slot during those awaits, and get 503 after both windows were charged. With two attacker tenants churning the slots, `p4_cpu.py` measured 2 of 201 victim requests charged and then refused, and 103 refused for free. ADR-32's "a 503 costs no budget" is therefore true only when there is no contention between the two checks. Concurrent requests from one tenant also pay for each other's 503. | Self-harm, or slight extra harm to a victim tenant while an attacker keeps the slots busy. Small: the window is a few ms. | Reserve synchronously before the first await (`token = cpu.reserve(tenant)` or 503, and release it in `finally` if the job never starts). Alternatively, refund the windows on a `run_bounded` 503. | high (measured) |
| P4-2 | low | services/api/qai_api/services/cpu.py:20, 28-29; services/ratelimit.py:30, 120-128 | **A cross-tenant compile contention residual remains (the P3-2 residual).** Slots stay shared in-process, two per replica, so enough workspaces can still occupy them. Per tenant, compile is at most about 8 CPU-s/min (`p4_cost.py`). Two compiles on one replica share the GIL, so the cluster has about 120 CPU-s/min. Roughly 15–30 workspaces looping at budget would keep most slots busy, and every other tenant would then see frequent 503s on compile and QASM. Each workspace also needs 4+ keys to reach the tenant budget, because the principal window is 30/min. Signup allows 5 workspaces per hour per IP. That is a large improvement on pass 3, where two workspaces were enough. The rest of the API is unaffected. | Several hours of signups from one IP, or a handful of IPs, then a sustained compile loop. The result is degradation of one feature, not an outage. | Accept for v1 and record the residual in REPORT.md. Longer term, move compile and QASM onto the worker queue under cost-weighted job limits, or add a global per-replica CPU-seconds window. | medium (computed from measured costs) |
| P4-3 | info | services/api/qai_api/logging.py:21, 29-31 | **Redaction gaps from bounding and truncation.** (a) A DSN with a password over 512 characters or a user over 128 is no longer redacted. `redis://:`+`p`×600+`@valkey` came out unredacted, though the old `\S*@` caught it. (b) `_scrub_str` truncates **before** scrubbing, so a DSN that straddles the 16 KiB cut keeps a partial secret: `…redis://:supersecre...[truncated 16 chars]`. (c) A long traceback loses its tail, which holds the exception line. That only affects debugging. No current config uses such long secrets, and exception text rarely carries a DSN. | None practical. An attacker would have to control the 16 KiB in front of a secret in the same string. | (b) On truncation, also drop the trailing partial token, i.e. cut back to the last whitespace, or redact the tail if it contains `://`. (a) Optionally add an unbounded but anchored fallback for `://[^\s@]{129,}@`-style userinfo after the cap. (c) Optionally keep the last 2 KiB of tracebacks instead of the first 16 KiB only. | high (measured) |
| P4-4 | info | services/api/tests/test_logging.py:30-39; services/api/tests/test_api_keys.py | **Gaps in the tests.** (a) The new ReDoS test asserts under 50 ms per payload. On this machine the `://`×40000 case already takes about 36 ms, so a slower or loaded CI runner may fail it intermittently. (b) RB-4 adds no concurrency test for P3-4. Pass 3 recommended one built on `gather`, and no such test exists. The fix is correct by inspection, but nothing guards it against regression. | None. | (a) Allow about 250 ms, which still catches the old quadratic patterns at around 500 ms or more, or use a ratio against a linear baseline. (b) Add a `gather` test with about 40 creates that asserts at most 25 are active. | high |

Counts (new in pass 4): critical 0, high 0, medium 0, low 2, info 2.

## Pass-3 findings status

| id | pass-3 severity | status | verification notes |
|---|---|---|---|
| P3-1 | high | **fixed at the root** | Both value patterns are bounded: `_EMAIL` `{1,64}@{1,253}\.{2,24}` and `_DSN_CREDS` `{0,128}:\S{0,512}@`. The cost per start position is now at most about 640 steps, and `_MAX_SCRUB = 16 KiB` caps each string before any pattern runs. `_API_KEY` and `_BEARER` are linear: the `[A-Za-z0-9]+` run in `_API_KEY` ends at the non-word character that a new `\bqai_` start needs. The worst case I could construct is 37.5 ms per string. The access line now logs only `route.path` or `<unmatched>` (middleware.py:136). The two `unhandled_error` lines truncate the path to 200 characters (middleware.py:115, errors.py:154). The only remaining attacker-chosen access field is `method`. h11 restricts it to tchar, which excludes `:`, `/` and `@`, so it cannot trigger `_DSN_CREDS` or the `@` path of `_EMAIL`, and it costs about 9 ms at 16 KB. No other log call site logs request-derived strings: I grepped all `log.*` calls in qai_api. New residual: P4-3 (info). |
| P3-2 | medium | **fixed at the root → residual P4-2 (low), P4-1 (low)** | `MAX_CONCURRENT = 2` with at most one job in flight per tenant, so one tenant can never hold every slot. Check and set in `run_bounded` are synchronous with no await between them, so the bound is exact: max 2 threads with 40 contending tenants. `compile:tenant` is cost-weighted (`1 + ops // 250`), which brings a tenant's ceiling down from about 70 to about 8 CPU-s/min. The windows are charged only after `available()` and after `_load`. The pre-check and acquisition race is P4-1. Two concurrent compiles do not degrade loop latency further: p99 78 ms against 72 ms. |
| P3-3 | low | **fixed** | `ensure_future(to_thread(fn))` with a done-callback `pop`, awaited through `asyncio.shield`. A cancelled caller leaves the inner task running, and the slot stays held until the thread finishes (measured). An exception in fn, including an `ApiError` 422 from `to_sdk`, frees the slot and reaches the caller. When the caller is cancelled and fn later raises, `shield`'s inner callback marks the exception retrieved, so no loop-handler log appears (measured). The task is strongly referenced through the executor future chain (RUF006 clean). Only loop shutdown cancels the inner task, which does not matter here. |
| P3-4 | low | **fixed (by inspection; not executed against the DB per instructions)** | `tenant_session` opens one transaction (`session.begin()` → `SET LOCAL` GUCs → yield → commit). `lock_tenant_api_keys` runs `pg_advisory_xact_lock(hashtext('api_keys:<tid>'))` as the first statement in that transaction. The count follows, then the insert (in a SAVEPOINT, which does not release an xact lock), the audit row and the commit. Under READ COMMITTED, the count after the lock gets a fresh snapshot that includes the previous holder's committed insert. The lock is released at commit or rollback. `create_key` is the only caller of `insert_api_key`. A 32-bit `hashtext` collision between tenants only serialises the two, so nothing leaks. There is no other advisory-lock user in the API, so no key-space clash. There is no regression test (P4-4). |
| P3-5 | low | **fixed** | `enforce_compile` hits `compile:principal` first, then `compile:tenant` with `cost`. `limits` `hit` records only when allowed, so a throttled principal consumes nothing from the tenant: 100 calls left the tenant at 90 of 120 (measured). The opposite case is a principal token spent when the tenant is exhausted. That only affects the caller's own budget. |
| P3-6 | low | accepted (ADR-32) — reasonable | The figure of about 80 ms per write at the cap is now recorded correctly. It is bounded by the general principal and IP windows. |
| P3-7 | low | **fixed (docs)** | REPORT.md now says "bounded only by the general per-principal and per-IP windows". ADR-32 corrects ADR-31's "export rule" wording. ADR-31 still contains the old text, but it is superseded. |
| P3-8 | info | accepted (ADR-32) — reasonable | Not reachable from the API. |
| P3-9 | info | accepted (ADR-32) — reasonable | Over-redaction errs on the safe side. Bounding introduces the under-redaction edges in P4-3. |

Earlier findings (P2-3, P2-5, P2-6, F-4, F-5, F-6, F-9, F-10, F-11, F-12): no change since pass 3. RB-4 does not touch them.

## Areas re-checked with no new finding

- **Compile and QASM 404 and authorisation path.** The `Viewer` dependency authenticates before the handler body runs. `available()` is the first statement, then `_load` (`tenant_session` + `get_circuit` filtered on `Circuit.tenant_id == principal.tenant_id`, plus RLS) returns `None` → 404, and only then is `enforce_compile` called. A foreign or malformed id returns 404 and charges neither window (measured ×51 per route). When the caller's own tenant is busy, a foreign id returns 503 rather than 404. That decision depends only on the caller's own slot state, never on the target id, so it is not an existence oracle. Loading before charging means a tenant whose budget is used up can still trigger `_load`. That costs the same as `GET /circuits/{id}`, which the general principal window already bounds.
- **`compile_cost` input.** It uses `len(circuit.ops)` from the stored row, never from client input. The op cap (≤ 2000) is enforced on write, so the cost is at most 9 and never exceeds the 120 window.
- **The `_inflight` dict.** It is mutated only on the loop thread: the check and set in `run_bounded`, and the done-callback, which runs on the loop. Keys are `str(principal.tenant_id)` from the resolved principal, so a client cannot occupy another tenant's slot or free it.
- **503 envelope.** `busy_error()` is unchanged: §7 `UNAVAILABLE` plus `Retry-After: 1`, and it discloses nothing beyond "busy".
- **Test helpers.** `test_circuits.py` writes `cpu._inflight` directly and clears it in `finally`. The fairness test uses synthetic tenant names and waits until the slot drains. Neither leaks state into later tests.
