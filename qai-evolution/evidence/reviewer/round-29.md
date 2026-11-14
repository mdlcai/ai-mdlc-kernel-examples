Commits reviewed: 596822b (F-40, audit hash chain with a verify endpoint)
Previous round reviewed: round 28 (F-39 PASS)

# Reviewer Gate: Review Round 29 (F-40)

- **Mechanism:** a fresh-context `invariant-check` subagent read `git show 596822b` and checked it against these sources:
  - SPEC §2 audit_events row (SPEC.md:72), the §3 row GET /api/v1/audit/verify (SPEC.md:145), §4 S7 / S8 (SPEC.md:296), §5 F-40 and its ACs (SPEC.md:836-845), §7 and the §8 test map (SPEC.md:1149);
  - ADR-54 (DECISIONS.md:509-537), ARCHITECTURE §2.12 (ARCHITECTURE.md:167-174), §9 INV-38 and invariants.json.
- **Invariant lint:** `node scripts/invariant-lint.mjs` printed `38 total / 37 machine (35 pass, 2 fail, 0 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`.
  - The 2 failures are the expected carries:
    - INV-2 (`artifacts` web-baseline report): stale, 8 unaccepted failures, covers 2 of 30 screens.
    - INV-4 (ui-coverage): `/app/governance/engine` has no page and W13 has no Playwright spec (F-41).
  - INV-38 now passes (`services/api/migrations/versions/0015_audit_chain.py:113`).
- **Python lint:** `npm run lint:py` reported "All checks passed!" and "168 files already formatted".
- **Tests:** with `TEST_DB_NAME=qai_test_r29` against qai-test-pg / qai-test-valkey:
  - `pytest tests/test_audit_chain.py`: 4 passed.
  - `pytest tests/test_migrations.py -k 0015`: 1 passed.
- **Deadlock probe:** a scratch probe against the r29 database reproduced a lock-order deadlock. One transaction took a row lock and then inserted an audit event. The other inserted an audit event and then requested the same row lock (FOR NO KEY UPDATE). Postgres returned `DeadlockDetectedError: ... waits for ExclusiveLock on advisory lock ...` (see N-R29-1).
  - The probe used a tenant it created itself and deleted afterwards.
  - Script: [local scratchpad path redacted].
- No repo file was edited, because the reviewer has no write tools.

## Verdict table

| Invariant / SPEC ref | File:line | Verdict | Notes |
|---|---|---|---|
| INV-38 trigger exists | 0015_audit_chain.py:113-115 | OK | `CREATE TRIGGER audit_events_chain BEFORE INSERT ... FOR EACH ROW`; the lint passes. |
| SQL only in repositories | repositories/audit.py:53-95; routes/audit.py:109-117 | OK | The verify SQL (`_SUMMARY`, `_FIRST_BREAK`) is in `repositories/audit.py`. The route calls `audit_repo.verify_chain` only. Other raw SQL is in the migration and tests, where it is allowed. |
| tenant_session + RLS | routes/audit.py:115 | OK | Uses `tenant_session(tenant, user, read=True)`. Both queries also filter `tenant_id = :t` explicitly. `audit_chain_hash` keeps the default PUBLIC EXECUTE, so qai_app can call it. |
| Trigger reads the last seq under RLS | 0015:47-67; 0001 (RLS enabled, not FORCEd) | OK | SECURITY DEFINER runs as the table owner. RLS is enabled but not forced, so the owner bypasses it, and the `ORDER BY seq DESC LIMIT 1` lookup sees the whole tenant chain. This covers `user_session` inserts with an empty `app.tenant_id` (login failure, `api_key.used` after `rescope`) and `system_session` inserts. `search_path = public, pg_temp` (pg_temp last), and qai_app has USAGE but not CREATE on public, so a search-path hijack is not possible. Revoking EXECUTE from PUBLIC on the trigger function does not affect firing. |
| Per-tenant advisory lock and concurrency | 0015:56-62; db/engine.py (default READ COMMITTED) | OK | `pg_advisory_xact_lock(hashtextextended('audit_events_chain:'‖tenant, 0))` is held until commit. Under READ COMMITTED, the SELECT after the lock takes a fresh snapshot, so it sees the previous committed insert. The lock is re-entrant within one transaction. UNIQUE(tenant_id, seq) is the backstop. A 64-bit hash collision between tenants would only cause contention. See N-R29-1 for lock ordering. |
| Hash determinism | 0015:30-45 | OK | `jsonb_build_array(...)::text` has deterministic output: jsonb keys are normalised and numerics are exact. NULL becomes JSON `null`, which differs from the string `"null"`. On PG 18, `extract(epoch FROM timestamptz)` is numeric and timezone-independent, so `*1e6 ::bigint` is exact to the microsecond. CHAR(64) to text casts strip trailing blanks, but the CHECK forces 64 hex characters, so there is never padding. The trigger (`NEW.prev_hash` implicit cast), the backfill (`prev` text) and verify (`prev_hash::text`) all feed the same text. See N-R29-4 for the IMMUTABLE label. |
| Backfill order and lock | 0015:69-91, 94-101 | OK | Rows are numbered per tenant in `ORDER BY tenant_id, at, id`, with the counter and `prev` reset when the tenant changes. The trigger is created only after the backfill and is INSERT-only. `transaction_per_migration=True`, and ADD COLUMN already holds ACCESS EXCLUSIVE, so the explicit SHARE ROW EXCLUSIVE lock is redundant but harmless. The NOT NULL, CHECK and UNIQUE constraints are added after the backfill. |
| Downgrade completeness | 0015:118-127 | OK (see N-R29-2) | Drops the trigger, the trigger function, the 3 constraints, the 3 columns and `audit_chain_hash`. The test confirms the columns and function are gone and that re-upgrade works. |
| ADR-54 consequence "Both downgrades refuse while their data exists" | DECISIONS.md:534 vs 0015:118 | DRIFT (low) | The 0015 downgrade drops the chain without checking for data. The docstring calls it derived data. See N-R29-2. |
| ORM mirrors 0015 | models/core.py:65-75 | OK | BigInteger and CHAR(64) with `FetchedValue`, so RETURNING picks up the values the trigger sets. The CHECK texts and `uq_audit_events_tenant_seq` match the migration. SPEC.md:72 matches. |
| Existing audit inserts (user, api_key, scheduler, system) | services/audit.py:25-50; deps.py:125; governance.py:829 | OK | Every insert path goes through `repositories/audit.insert`, which is an ORM INSERT, and the trigger chains all of them, including the evidence hook (`system`) and `api_key.used` in `user_session`. App-supplied values are overwritten (tested). |
| verify_chain break order: gap, then link, then hash | repositories/audit.py:66-84 | OK | The `CASE` tests, in this order: `seq <> lag(seq)+1` (starting from 0, so the first event must be seq 1), then `prev_hash <> COALESCE(lag(hash), zero)`, then the recomputed hash. The first non-null reason in seq order is returned. Deleting the first event gives `gap` at seq 2. Editing `hash` itself gives `hash` at that seq. A swap of seqs gives `link`. An empty chain gives `events 0, head null, brokenAt null, verified true`. See N-R29-3 for tail truncation. |
| §3 verify response shape | routes/audit.py:116; repositories/audit.py:91-95 | OK | Returns `{verified, events, head{seq,hash}\|null, brokenAt{seq,id,reason}\|null, checkedAt}` exactly. `checkedAt` is ISO with a Z suffix (tested). |
| RBAC S7: verify is S, admin | routes/audit.py:110 (`UserAdmin` = `require_user_admin`) | OK | A viewer or member (even with the reviewer capability) gets 403. An admin-role API key gets 403 FORBIDDEN. Anonymous gets 401. An admin gets 200 (tested). |
| Read-only and writes no audit event | routes/audit.py:115-116 | OK | Both queries are SELECTs, there is no `audit.record`, and the session uses `read=True`. The test checks that the event count is unchanged. |
| SPEC §7 error codes | routes/audit.py | OK | Only FORBIDDEN and UNAUTHENTICATED come from the existing dependencies. No new codes. |
| No pickle / eval / exec; run_bounded | all files | OK | None of the three is present. Verify runs in the database, not as CPU-bound Python work. |
| Design tokens / pinned scripts | n/a | OK | The commit changes no web files, no `scripts/*.mjs` and no `governance-policy.json`. |
| S8 wording: "hash chain verified", never "tamper-proof" or "signed" | 0015 docstring; routes/audit.py:112-113 | OK | Only "hashed, not signed" and "does not stop the database owner" appear. No "tamper-proof" or "signed" claim. The test project names "Tamper {i}" are test data. |
| ARCH §2.12 lock serialisation | governance.py:567-578 vs 675; routes/projects.py:268-275 | DRIFT (medium) | A new lock-order inversion produces a deadlock and a 500. See N-R29-1. |
| AC1: concurrent consecutive seqs; independent tenant chains | test_audit_chain.py:30-50 | OK | 6 concurrent `gather` creates give seqs 1..n with valid links. A second tenant starts at seq 1 with zero `prev_hash`. |
| AC2: owner edit gives hash, delete gives gap, prev_hash change gives link | test_audit_chain.py:53-85 | OK | Each case asserts the exact `brokenAt {seq, id, reason}`. Reordering is also detected. |
| AC3: viewer, member and API key get 403; verify writes no event | test_audit_chain.py:105-123 | OK | All covered, plus 401 for anonymous and 200 for an admin. |
| 0015 migration test | test_migrations.py:345-394 | OK (gaps low) | Covers backfill in `at` order, the chained insert after the backfill, the trigger's presence, a clean downgrade and re-upgrade. See N-R29-5. |
| §8 test map | SPEC.md:1149 | OK | test_audit_chain.py and test_migrations.py cover the listed items. |

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| F-40 | 596822b | **PASS** (0 blockers, 0 major, 1 medium, 4 low) | The trigger is correct: SECURITY DEFINER, a fixed search_path, a per-tenant xact advisory lock with a fresh READ COMMITTED snapshot, and UNIQUE as the backstop. The hash is deterministic (jsonb text, exact epoch microseconds, consistent CHAR to text casts, NULL as JSON null). The backfill is per tenant in (at, id) order. The downgrade drops everything it added. Verify returns the first break in gap, link, hash order and matches the §3 shape exactly. It is session admin only, read-only, and its SQL is in the repository. All three ACs are tested and pass. The wording is compliant. N-R29-1 is a must-fix follow-up, but it only aborts one transaction cleanly, with no integrity or security effect. |

## Findings

- **N-R29-1 (medium, concurrency; introduced by F-40):** the audit advisory lock creates a lock-order inversion.
  - **The lock order:**
    - The trigger takes the tenant advisory lock at the first audit insert and holds it until commit.
    - `governance.release` writes the `release.released` audit (governance.py:567) and then calls `_switch`. `_switch` locks the project row FOR NO KEY UPDATE (governance.py:675).
    - Two other paths take the project row lock first and audit second:
      - PATCH /projects/{id} runs `update_project`, an UPDATE that takes a NO KEY UPDATE row lock (routes/projects.py:268), and then audits (line 275);
      - rollback runs `_switch`, which locks the project and then audits (governance.py:675, 701).
    - The two orders form a cycle.
  - **The result:** Postgres detects the deadlock after `deadlock_timeout` and aborts one transaction, which surfaces as a 500. That status is outside the §3 / §7 contract.
    - A scratch probe reproduced the cycle on the r29 database. It used a row lock followed by an audit insert, against an audit insert followed by FOR NO KEY UPDATE: `DeadlockDetectedError ... waits for ExclusiveLock on advisory lock`.
    - Before F-40 the same interleaving only waited.
  - **Same pattern elsewhere:** `jobs.complete` writes several evidence rows. Each evidence hook locks release rows (`open_for_kind` FOR UPDATE) after an earlier hook in the same transaction may already have audited. So a concurrent approve, hold or evaluate on a second release of the project can deadlock against the worker.
  - **Fix:**
    - In `release()`, write the `release.released` audit after `_switch`, or lock the project before the first audit.
    - Add the rule "the audit insert is the last lock a transaction takes; take every row lock before the first `audit.record`" to ARCHITECTURE §2.12.
    - Optionally map SQLSTATE 40P01 to a retry.
  - **Severity:** not major. The aborted transaction rolls back atomically, so no half promotion and no lost audit event are possible. Fix it before F-41 adds the UI that makes release and rollback concurrent.
- **N-R29-2 (low, ADR drift):** ADR-54 Consequences (DECISIONS.md:534) says "Both downgrades refuse while their data exists". The 0015 downgrade drops the chain without checking (0015:118-127), and its docstring calls the chain derived data.
  - A downgrade followed by an upgrade re-derives the chain from the current, possibly edited, rows in (at, id) order. That silently re-verifies earlier edits.
  - This needs owner rights, which the stated limit already concedes.
  - Recommend either refusing the downgrade while `audit_events` has rows, or correcting ADR-54 to "0015's downgrade drops the derived chain".
- **N-R29-3 (low, limit wording):** deleting the newest event(s) cannot be detected.
  - After the deletion, the trigger's next insert reuses the freed seq and links to the new head, so verify passes. `head{seq,hash}` makes this detectable only by comparison with a value recorded outside the database.
  - SPEC S8 says the chain "detects an edited, removed or reordered event". Recommend adding "except removal of the newest events, unless the head is recorded outside the database" to the limits in SPEC §5 F-40 and ARCHITECTURE §2.12.
  - Related: `created_at` is not hashed, which matches the SPEC field list. `at` is the displayed time and is hashed.
- **N-R29-4 (low, labelling):** `audit_chain_hash` is declared IMMUTABLE but calls STABLE functions (`jsonb_build_array`, `extract` on timestamptz).
  - The value does not actually depend on the TimeZone or other GUCs, and the function is not used in an index. So the label is harmless today, but STABLE would be accurate.
  - Separately, verify recomputes every hash of the tenant chain in one unbounded statement. That is acceptable for an admin-only endpoint. Consider a statement timeout or a windowed scan when tenants grow large.
- **N-R29-5 (low, coverage):**
  - The 0015 backfill test uses one tenant, so the per-tenant reset of `n` and `prev` in the backfill loop is untested. A second tenant whose events interleave in time would cover it.
  - The concurrency test runs `gather` through the in-process client. It is not shown that two audit inserts actually wait on the advisory lock. A two-connection test as the owner would pin it.
  - The empty-chain response and the `brokenAt` of a reorder are not asserted.
  - No test covers N-R29-1.
- **Carried:**
  - INV-2 (stale web baseline) and INV-4 (the F-41 screen and W13 not built) are expected until F-41.
  - N-R28-3 (`delete_project` lock) is now also part of the N-R29-1 lock-order cycle.
  - N-R28-4 is still open.

**Verdict: PASS**

## Recorded by the build session

The invariant-check reviewer has no write tools, so the build session saved its hand-back report above verbatim. The verdict is the reviewer's. N-R29-1 is fixed in a follow-up before the F-41 UI ships. N-R29-2..5 are carried.
