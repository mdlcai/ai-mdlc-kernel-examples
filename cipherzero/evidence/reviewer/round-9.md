Commit reviewed: c0f949c
Previous round reviewed: artifacts/reviewer/round-8.md (80fc26b)

# Reviewer round 9: narrow settle check (Review Round, round cap)

Note on scope: the requested range `03fae4a~1..c0f949c` is empty because `03fae4a~1` is `c0f949c`. The intended
diff, `git show c0f949c` (c0f949c~1..c0f949c), was reviewed. FRONTEND-AUDIT*.json and GOVERNANCE-DECISIONS.jsonl
were excluded.

Mode: read-only. Only git, grep and sed were used. No vitest, build or docker was run, because a Lighthouse
baseline was running on the machine. Test verdicts below come from reading the code.

## Inputs
- `git show c0f949c` (18 files).
- artifacts/reviewer/round-8.md: N1 (:51, :69) and the run-lock note (:63, :75).
- artifacts/security/pass-6.md "## Round 2 (80fc26b)": the SEC-30 residual MEDIUM with its fix list 1–5, plus
  SEC-37, SEC-38 and SEC-39.
- DECISIONS.md ADR-095 (:1467–1500) and the ADR-094 upgrade note.
- Context read: packages/core/src/quantum/ingest.ts, apps/api/src/services/quantum.ts (plan validation),
  apps/worker/src/runner.ts, node_modules/bullmq 6.3.15 (worker.js, job.js, types/processor.d.ts).

## 1. SEC-30 residual: run budget, partial ingest, rotation, lock: PASS
- **Budget checks.**
  - `RUN_BUDGET_MS = 20 * 60_000` is at apps/worker/src/jobs/external-discovery.ts:181.
  - The deadline `runAt + RUN_BUDGET_MS` is set at :213.
  - The check before each domain is at :215–219 and counts the domain as skipped.
  - The check before each probe is at :275–278. It runs after the INV-56 recheck await (:273), so a probe never
    starts past the deadline. Probes already in flight finish.
- **Null results are handled downstream.**
  - The status update uses `results[i]?.status === status` (:299). Un-probed hosts keep their status and
    `lastProbeAt`.
  - Scope and findings use `r?.status !== 'reachable'` (:310–311). So `scope` lists only `tls://host:port` for
    reachable ports of probed reachable hosts, and nulls add no findings.
  - `pool` (:53–) fills every slot, because a cut lane returns null at once. A thrown error is still rethrown.
  - Nothing else reads `results`.
- **A partial report cannot wrongly mark an asset gone.**
  - Plan validation (apps/api/src/services/quantum.ts:852–867; ARCHITECTURE §5.8, ":362 not seen while its locator
    was in scope") judges an asset by the newest same-domain report.
  - An asset whose host was not probed is not seen, and its locator is not in that report's scope. It is therefore
    reported as blocking `not_in_scope` (:866), not validated. A partial report fails closed.
  - `ingestFindings` adds only finding locators to the scope (ingest.ts:154).
- **Rotation.**
  - Hosts are ordered `[{ lastProbeAt: { sort: 'asc', nulls: 'first' } }, { hostname: 'asc' }]` (:257).
  - Domains are ordered `[{ lastScanAt: … nulls: 'first' }, { id: 'asc' }]` (:203).
  - Both columns are nullable (packages/core/prisma/schema.prisma:456, :480). Postgres sorts NULLs last on ASC, so
    the explicit `nulls: 'first'` is required, and it is present.
- **Run lock.**
  - RUN_LOCK_MS is 2 h (:184). It is taken per domain, just before `scanDomain`.
  - The worst-case hold is about 25–27 min: discovery (one CT call within the 120 s client timeout plus a 30 s
    transaction), probes until the deadline, the in-flight tail (within 120 s per probe), the status updates, and
    the ingest transaction (60 s timeout).
  - This closes round-8's run-lock note (:63, :75) and makes SEC-35 effectively unreachable.
- **Test** (apps/worker/test/external-discovery.test.ts:390).
  - Each probe advances a `fixedClock` by budget/4, so 2 lanes start 4 probes, and the test asserts at most 6.
  - It also asserts `processed: 1, skipped: 1`, an ingested report for the first domain only, the count of
    never-probed hosts, and that run 2 starts with `late-ext.org` and probes the never-probed hosts first.
  - It would fail on the old code in three ways:
    - The import of `RUN_BUDGET_MS` would fail.
    - All 12 hosts would be probed.
    - Both domains would be processed.
  - It would also fail if either budget check were removed. Without the domain check, the second domain would get
    a report, so the count of 0 fails.

## 2. `limitSlots` (apps/worker/src/scheduler.ts:50–86): PASS
- BullMQ is 6.3.15 (packages/core/package.json:44; node_modules/bullmq).
  - `Processor` is `(job, token?, signal?)` (types/processor.d.ts:6).
  - `callProcessJob` passes the token as the 2nd argument (worker.js:190–191).
  - The wrapper's signature `(job, token)` receives it and forwards it to `job.moveToDelayed(now() + SLOT_RETRY_MS,
    token)` (:76).
- `Job.moveToDelayed` calls the backend with `skipAttempt: true` (job.js:783–792), so no attempt is spent.
- `handleFailed` returns early for `DelayedError` (worker.js:643–650). It does not call `moveToFailed` or emit
  `failed`, so `deadLetterIfExhausted` is not triggered. `maxStartedAttempts` is not configured, so
  `attemptsStarted` growth is harmless.
- **The in-flight count cannot leak or go negative.**
  - The get, check and set run synchronously with no await between them (:73–79), so concurrent jobs cannot race.
  - The delayed path never increments.
  - The `finally` decrement (:83) pairs one-to-one with the increment.
- Names without a limit go straight to `run(job)` (:72).
- **Tests.** apps/worker/test/slot-limit.test.ts:12 checks:
  - the third concurrent job is delayed to `now + SLOT_RETRY_MS` and rejects with `DelayedError`;
  - `domain-verify` is not held back;
  - a freed slot admits the next job.

  The test at :38 checks the slot is released when a run throws. Without the `finally`, the third call would
  reject with `DelayedError` instead of `boom`. Both tests fail on the old code, where `limitSlots` does not exist.

## 3. SEC-37 / SEC-38 / SEC-39: PASS
- **SEC-37.** `server.on('error', …)` logs and calls `process.exit(1)` (apps/prober/src/main.ts:45–48). It is
  registered before `server.listen` (:49), so EADDRINUSE and EACCES now end the process and the container
  restarts. This closes round-8 N1.
- **SEC-38.** The guard is at deploy/pki/bootstrap.mjs:29–35. `inside(out, dir) || inside(dir, out)` refuses:
  - equal paths (relative is `''`);
  - a child of `--out` (relative does not start with `..`);
  - a parent of `--out` (relative is `pki`).

  A different Windows drive gives an absolute relative path, which counts as no overlap. The guard runs while the
  arguments are parsed, at the top level (:23–37), before any `run(openssl …)` or file write. The test at
  apps/prober/test/pki-mount.test.ts:88–105 covers all three cases and asserts no `ca.key` was written.
- **SEC-39.** `PROBER_TIMEOUT_MS: int(1000, 15000, 10000)` is at apps/prober/src/env.ts:41. The SPEC.md:562 row
  reads "1000–15000 (ADR-095)", and ADR-095 (:1488) says 1–15 s. These agree.

## 4. Docs match the code: PASS (two wording/count corrections, non-blocking)
- ADR-095 (DECISIONS.md:1467–1500) matches the code in four places:
  - the budget is checked before each probe and before each domain;
  - un-probed hosts keep their status;
  - hosts and domains are ordered longest-ago first;
  - there are 2 slots, a 30 s delay and `DelayedError`.
- SPEC.md:324 is the rate line. ARCHITECTURE.md:455–458 is §5.10 Discovery step 2. CHANGELOG.md:5 is 0.5.2.
  VERSION.md:3 and :7 read 0.5.2. The SECURITY-AUDIT.md Pass 6 table (:121–134) has the correct per-ID
  severities and statuses.
- invariants.json:15: INV-2 `accepted` lost only the `landing`/`lighthouse` carry. This is a tightening. INV-2
  staleness was ignored as instructed.
- Corrections (non-blocking):
  - **SECURITY-AUDIT.md:13** says "INFO 6 (SEC-38, 39 fixed; SEC-34, 35, 36 noted)". The list has five IDs, and
    pass-6.md:414 says INFO 5. It should read **INFO 5**.
  - **"One job" budget wording.** The budget is per tenant within a job, not per job: `runAt` is taken per
    `runExternalDiscoveryForOrg` call (:198, :213). The cron sweep is one BullMQ job with no `tenantId`, which
    iterates every tenant (apps/worker/src/runner.ts:44–49, :62). So one sweep job can hold one of the two
    external-discovery slots for up to 20 minutes per slow tenant. Verify, grant expiry and rotation are still
    protected by the slot limit, so SEC-30 stays closed. Three places should say "per tenant per run":
    ADR-095:1479, ARCHITECTURE.md:456 and the code comment at external-discovery.ts:178. The ADR-095 "Known gaps"
    should also say that a slow tenant delays later tenants in the daily sweep by up to 20 minutes each.

## Non-blocking notes
- **N1. An empty report when the budget runs out right after discovery.**
  - Where: external-discovery.ts:215, :275, :333.
  - When it happens: a domain that passes the domain check just before the deadline runs discovery, then has
    every probe cut.
  - Effect 1: it still ingests a report with empty scope and findings.
  - Effect 2: that report sets `lastScanAt`, so the domain goes to the back of the rotation with nothing probed.
  - Effect 3: it becomes the newest report for that domain, so plan validation blocks its assets as
    `not_in_scope`. This is fail-closed.
  - Suggested fix: when `cut === hosts.length && hosts.length > 0`, return false before recording, or skip the
    `lastScanAt` bump.
- **N2. Liveness on very slow large domains.** Plan validation uses the newest same-domain report. On a domain
  that always runs out of budget, each report covers a different slice. A plan whose assets span slices then stays
  blocked as `not_in_scope`. This fails closed, and benign hosts finish well inside 20 minutes.
- **N3. Test gap on the token.** The slot-limit test's fake ignores `token`. A regression that dropped the token
  would pass the test, but fail in BullMQ with a missing lock. Consider asserting that the token is forwarded.
- **N4. Prober error logging.**
  - main.ts:46 and :59 still log only `err.message`. Logging the stack was item 2 of the SEC-37 fix and was not
    done.
  - net.Server also emits `error` on accept failures such as EMFILE. Those now restart the prober, which is
    acceptable because it fails closed.
- **N5. The 15 s timeout cap is near the edge.** At T = 15 s, a request takes about 5T plus DNS. That is 105 s,
  and up to about 120 s with pass-6's 3T DNS estimate, against the worker's 120 s idle timeout. The cap matches
  the SEC-39 recommendation, but deriving the worker timeout from T would remove the edge.
- **N6. The slot limit is per worker process.** ADR-095 "Known gaps" already records this.

Verdict: PASS
