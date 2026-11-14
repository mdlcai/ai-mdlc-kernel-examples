Commits reviewed: 538f0f5d826515fbb0d747b5943d41fad1a6ca63 (RB-15), a5201bcd2040481bcc60132bde5b15608c70d35d (C-23, release 0.6.0); also reviewed: artifacts/security/pass-7.md (uncommitted) and the 0.6.0 delivery evidence (e2e, web baseline, smoke)
Previous round reviewed: e74317a (round 13, RB-14; F-25 and F-26 re-checked as amended)

# Reviewer Gate: Review Round 14 (RB-15 round 13 follow-ups, C-23 release 0.6.0, security pass 7, delivery evidence)

- **Mechanism:** a fresh-context subagent reviewed `git show 538f0f5` against artifacts/reviewer/round-13.md (N-R13-1, N-R13-3), SPEC.md §3 `orb` (the note on `queries`, `foundAt` and `queryRatio`), §5 F-26, W10.3 and DECISIONS.md ADR-47. It reviewed `git show a5201bc` against INV-34, `git log --oneline -15` and `git log v0.5.0..HEAD`. It read security pass 7 and checked the delivery claims against the run logs in the session scratchpad (`orb/e2e2.log`, `orb/e2e3.log`, `orb/baseline-full.log`, `orb/baseline-full2.log`, `orb/baseline-failing.log`, `orb/site-only.log`, `orb/site-only2.log`, `orb/smoke2.log`, `orb/deploy3.log`, `orb/deploy4.log`) and the retained `artifacts/playwright-report/index.html`. It ran these:
  - Web checks: `npx vitest run components/results/orb-panels.test.tsx` gave **1 file, 10 tests passed** (9 before RB-15, plus 1). `npx eslint` on orb-panels.tsx and orb-panels.test.tsx exits 0. `npx tsc --noEmit` exits 0.
  - `npm run lint:invariants`: 34 total, 33 machine-checked (31 pass, **2 fail**), 1 manual. INV-34 passes: "0.6.0: 1 workspace(s), 1 internal package(s), the lockfile, 7 declared default file(s) agree". INV-1 fails only on c473132 (F-24, M-R12-0). INV-2 fails: "11 unaccepted failure(s): app/load, projects/load, audit-log/load; report covers 20 screens, SPEC UI Surface lists 27".
  - `node scripts/govern.mjs queue` (read-only): awaiting human approval C-21 (GD-0112) and RB-14 (GD-0116); pending review C-23. GOVERNANCE-DECISIONS.jsonl has GD-0119 RB-15 AUTO_APPROVE and GD-0120 C-23 REVIEW.
  - A P7-2 reproduction with `python -I`: `run_orb(seed=1, radius=10**400)` and `tolerance=10**400` raise OverflowError; `radius=nan` raises CircuitError.
  - Read-only HTTPS requests to the running stack with curl: `/api/health`, the headers of `/`, `/login` and `/app`, the 11 `/_next/static` assets referenced by the landing HTML, `/_next/static/chunks/0umf92h1sb77c.js`, `/favicon.ico`, `/icon.svg` and an unknown route.

  It did not run Playwright, the web baseline or the smoke test (the host is contended). It did not touch docker, did not read FRONTEND-AUDIT.json, ran govern.mjs only as `queue`, edited only this file and staged nothing.

## Verdict table

| ID | Commit | Verdict | Notes |
|---|---|---|---|
| RB-15 | 538f0f5 Round 13 follow-ups | **PASS** (1 minor, 2 notes) | The Optimize cell now says "optimum" only when the QAI best equals the classical best, and "best" otherwise. That matches SPEC §3's definition of `foundAt`. The commit message says the Compare hint changed, but it did not. N-R13-1's Compare half is still open (M-R14-1). |
| C-23 | a5201bc Release 0.6.0 | **PASS** (0 findings) | INV-34 passes across 10 files. No stale 0.5.0 default is left in any manifest, config or compose file. CHANGELOG 0.6.0 lists exactly the non-governance commits in v0.5.0..a5201bc. VERSION.md has the row. GD-0120 (REVIEW) is pending. |
| Security pass 7 | artifacts/security/pass-7.md | **AGREE** (1 note) | P7-1 to P7-3 need no fix before release. P7-4's INV-2 figures are out of date, and its header triage is answered below (N-R14-1). |
| Delivery evidence | e2e, web baseline, smoke | **JUSTIFIED, with qualifications** (3 notes) | Each claim matches its logs. The e2e and baseline summaries leave out details that the record should keep (N-R14-2, N-R14-3, N-R14-4). |

Carried forward (human action): M-R12-0 (INV-1, F-24 c473132 has no governance decision), and the queue: C-21 (GD-0112) and RB-14 (GD-0116) HUMAN_APPROVAL, C-23 (GD-0120) REVIEW. INV-2 still fails (N-R8-6, N-R14-3).

## RB-15 (538f0f5): finding by finding

| Check | Result | Evidence |
|---|---|---|
| N-R13-1: Optimize cell wording | PASS | orb-panels.tsx:641-643 renders `optimum` only when `o.qai.best !== undefined && o.qai.best === o.classical.best`, and `best` otherwise. Both values come from `num()`, so they are finite numbers or undefined. An undefined value on either side gives "best", which is the safe default. SPEC.md:192-193 defines `foundAt` as "the query at which the final best was first measured", so "best at query N" is exact in every case and "optimum" only on agreement. For seed 7 (best 107 = classical 107) the cell still reads "optimum at query 61", as SPEC W10.3 (SPEC.md:630) and F-24 AC (SPEC.md:533) describe. |
| N-R13-1: Compare hint | **PARTIAL (M-R14-1)** | The RB-15 commit message says the hint "reads 'Queries to reach the QAI best / classical evaluations'". The diff does not touch it. orb-panels.tsx:750 still reads `hint="Queries to find the optimum / classical evaluations"`. That is ADR-47's literal text (DECISIONS.md:389), so the code agrees with the ADR, but round 13's case (seed 7, budget 4: QAI best 57, classical best 107, queryRatio 0.016) still shows 0.016 under "Queries to find the optimum". CHANGELOG.md's RB-15 line describes only the cell, which is accurate. |
| Docs first | NOTE (N-R14-5) | RB-15 changes visible wording without touching SPEC W10.3 or ADR-47. The change refines a label within ADR-47's definitions, and no behaviour or contract changes. |
| New test: missed optimum | PASS | orb-panels.test.tsx:259-276 overrides `qai.best` to 57 (classical stays 107) and asserts that the cell contains "best at query 61" and does not contain "optimum". The fixture keeps `report.outcome` pass and every check passing, which is not a result the SDK could produce, but this test checks only the label. |
| New test: older result without foundAt | PASS (N-R14-5) | It drops `foundAt` and asserts no "at query" text and "128 of 128 queries spent", which is ADR-47's fallback in its consequences (DECISIONS.md:391). It does not assert that `data-found-at` is absent. React omits an undefined attribute, so the behaviour is correct but untested. |
| N-R13-3 | PARTIAL (N-R14-5) | The panel gap is closed. test_jobs_orb.py:137 still asserts only `queries <= 128` and does not check `foundAt`. RB-15 did not touch it. The commit message claims only the panel part. |
| Tests, lint, types | PASS | vitest 10/10, eslint 0, tsc 0. |
| W10 and smoke | PASS | w10-orb.spec.ts reads only `data-found-at` (line 95), not the cell text, so the wording change cannot break it. W10 passed on desktop (31.1 s) and mobile (19.1 s) in e2e run 1 (e2e2.log:19, :42). |
| Governance | PASS | GD-0119 RB-15 AUTO_APPROVE (file scope). |

## C-23 (a5201bc): finding by finding

| Check | Result | Evidence |
|---|---|---|
| Version agreement | PASS | INV-34 passes. The diff bumps package.json, apps/web/package.json, package-lock.json (3 entries), packages/qai-sdk/pyproject.toml and `qai/__init__.py`, services/api/pyproject.toml and `config.py`, apps/web/lib/env.ts, .env.example and compose.yaml (4 defaults). A repo-wide grep for `0.5.0`, excluding node_modules, .next and artifacts, finds only history: CHANGELOG, VERSION, REPORT.md, DECISIONS.md, BUILD-STATE and gitignored `.hypothesis` caches. The running stack's `/api/health` reports `"version":"0.6.0","commit":"68c3881"`. |
| CHANGELOG scope | PASS | `git log v0.5.0..a5201bc`, without governance records, is RB-13, C-21, F-24, C-22, F-25, F-26, RB-14 and RB-15. The 0.6.0 section lists exactly these. v0.5.0 is f541497 (C-20), so leaving out C-20 is correct. `git log --oneline -15` reaches back to c473132 and agrees. |
| CHANGELOG content | PASS | F-24's "8 physics and 2 comparison checks" matches the 10/10 smoke. F-25's "honours an explicit budget under 1 s" matches pass 7's `_budget` reading. RB-14's "`foundAt`, the query where the search first reached its best; `queryRatio` = foundAt / 2^qubits; τ0 hint Q0/ω; stage order 2 Simulate, 3 Optimize" matches round 13. RB-15's line claims only the cell change, which is accurate. |
| VERSION.md | PASS | The row `0.6.0 · 2026-11-04 · Re-Entry R4 · … RB-14 foundAt (ADR-47); RB-15` matches the CHANGELOG date. 2026-11-04 is the UTC date of a 21:10 −06:00 commit. No tag is claimed yet, which is correct because v0.6.0 does not exist. |
| Commit message | PASS | "Bumps every manifest, the lockfile and the version defaults to 0.6.0 (INV-34), adds the VERSION row and the CHANGELOG 0.6.0 section for F-24..F-26, RB-13..RB-15 and C-21..C-22" is accurate. |
| Governance | PASS (pending) | GD-0120 C-23 REVIEW (project scope, api+application). `govern.mjs queue` shows "pending review: C-23". |

## Security pass 7: agreement

| Finding | Agree no fix before release? | Reasoning |
|---|---|---|
| P7-1 (info), the heaviest orb job is the smallest grid, at cost 1 | **Yes** | 15 s CPU and 236 KB is 12 % of the 120 s timeout and RLIMIT_CPU, and it matches ARCHITECTURE §2.9's "~15 s and ~260 KB at the limits", which round 12 measured independently (15.3 s, 261,344 B). It adds only marginally to the accepted P5-2 fairness residual. Clamping `budget` to `8·2^qubits` would need a SPEC §3 change first, so it belongs in a later round. |
| P7-2 (info), SDK OverflowError on a huge int | **Yes** | Reproduced: `radius=10**400` and `tolerance=10**400` raise OverflowError, not CircuitError. The API cannot reach it, because `_bounded` returns OUT_OF_RANGE first and the runner passes `float()` of validated values. It is a direct-SDK error-type defect, the same as the open P6-3. Fixing both together with a shared `_num` helper is the right follow-up. |
| P7-3 (info), the two-phase 422 | **Yes** | It is a usability gap. The response stays within the SPEC §7 contract, leaks nothing, and the error count matches the list. |
| P7-4 (info), INV-1 and INV-2 fail | **Yes, but its figures are out of date** (N-R14-1) | Pass 7 quotes the pre-R4 report (3 failures, gitSha d7df5cb). The current report is the second 0.6.0 full run, with 11 failures. Pass 7 asked for `site/headers` to be triaged before release. This round did that: every security header except HSTS is present (N-R14-1), so there is no header regression. INV-1 needs the human governance action M-R12-0. |

## Delivery evidence

| Claim | Result | Evidence |
|---|---|---|
| Full e2e: run 1 had 10 failures, all ECONNRESET, ERR_TIMED_OUT or fixture timeouts | JUSTIFIED (N-R14-2) | e2e2.log:332-344 reads "10 failed, 6 did not run, 30 passed (9.8m)". The causes are 1 `page.goto: net::ERR_TIMED_OUT` (smoke landing), 3 `read ECONNRESET` (W5 desktop and mobile on `POST /api/v1/invites/accept`, and mobile auth on `GET /api/v1/auth/me`), and the rest `Fixture "account" timeout of 30000ms exceeded during setup` (fixtures.ts:79). None is an assertion failure. W10 passed on both projects. |
| A serial re-run of the failures passed 14 of 16 | JUSTIFIED | e2e3.log reads "Running 16 tests using 1 worker … 2 failed, 14 passed (2.9m)". The 16 are the 10 failures plus the 6 that did not run. The 2 failures are W5 desktop and W5 mobile, both `read ECONNRESET` on `POST /api/v1/invites/accept` (w5-release.spec.ts:110). |
| W5 then passed alone on both projects | PARTLY EVIDENCED (N-R14-2) | artifacts/playwright-report/index.html (03:35:36Z) records 1 test: `mobile w5-release.spec.ts … expected`, 18.5 s, and test-results/.last-run.json reads `passed`. No artifact of the desktop solo pass survives, because the report was overwritten. |
| Web baseline: the failures are page-load timeouts and unreachable site probes | MOSTLY JUSTIFIED (N-R14-3) | Run 2 (baseline-full2.log) has 6 `load` cells, all `page.goto: net::ERR_TIMED_OUT` or `Timeout 45000ms`, plus site `meta`, `icon` (`/favicon.ico 0`, the kernel's fallback when landing meta is not loaded, web-baseline.mjs:498), `notFound` (`returned 0`) and `headers` (every header missing on both pages, which is what status 0 looks like). Run 1 also had two failures that were not load-related: `buildFreshness`, where the local apps/web/.next/BUILD_ID predated a5201bc (fixed by the 21:51 local rebuild, build2.log), and `evidence-detail@375 cwv` LCP 4244 ms, which passed in the `--failing` re-run. Two `--site-only` runs also reported `/_next/static/chunks/0umf92h1sb77c.js -> unreachable`. |
| Caddy never logged those requests; curl and node fetch succeed | PARTLY VERIFIED | Caddy logs were not read (no docker access). This round's curl got **200** for all 11 landing assets, including `0umf92h1sb77c.js` (13,822 B), `/icon.svg` 200, an unknown route 404 (designed), and `/favicon.ico` 404 (the page declares `/icon.svg`, so the 404 is not the failing cell). |
| Steady-state residuals equal 0.5.0's: HSTS off (ADR-29/45) and Lighthouse performance below 80 (landing 65, app 69-70) | JUSTIFIED as an inference, not a clean measurement (N-R14-3) | Headers: `/` and `/login` carry the nonce CSP, nosniff, Referrer-Policy, Permissions-Policy, X-Frame-Options and COOP. `/app` returns 307 with the restrictive CSP. None has HSTS, as ADR-29 and ADR-45 require. Lighthouse: run 2 has landing 65 and app 69, run 1 has app 70 (landing was 0, unreachable). No 0.6.0 full run is clean apart from these residuals, and the baseline covers 20 of 27 SPEC screens, without `/app/orb` (N-R8-6). |
| Smoke 27/27 | JUSTIFIED (N-R14-4) | smoke2.log reads "smoke: 27/27 passed", including "Orb Lab ORB-0 end to end … 10/10 checks, ledger residual 1.2e-13, optimum at QAI query 61/128, evidence verified". It ran against the deploy3 build (qai_api-0.6.0 wheel, `qai-api:0.6.0`), but health then reported `commit 664deff` because the GIT_COMMIT in .env was stale. |

## Findings

- **M-R14-1 (minor, RB-15): the Compare hint did not change, though the commit message says it did, and N-R13-1's Compare half is still open.**
  - **Evidence:** the RB-15 commit message says "the Compare hint reads 'Queries to reach the QAI best / classical evaluations'". orb-panels.tsx:750 still reads "Queries to find the optimum / classical evaluations", and `git show 538f0f5` changes only lines 641-643 of that file. When QAI misses the optimum (round 13's seed 7, budget 4), the Compare KPI still shows 0.016 under a hint that claims the optimum was found. The outcome badge reads inconclusive, so the page as a whole does not mislead, but the KPI does.
  - **Fix:** amend ADR-47's hint text first (docs first), then change the hint to "Queries to reach the QAI best / classical evaluations", or blank or qualify the KPI when `qai.best !== classical.best`. Add an assertion on `[data-query-ratio]`'s hint to the missed-optimum test. Do not rewrite the pushed commit message. Record the correction in the follow-up commit.
- **N-R14-1 (note, security pass 7): P7-4 quotes a web-baseline state that has since been replaced.** Pass 7 cites 3 failures (`site/headers`, `landing/lighthouse`, `app/lighthouse`, gitSha d7df5cb). INV-2 now reports 11 failures from the second 0.6.0 full run. The header triage pass 7 asked for has been done (see the delivery table): the full header set except HSTS is served on `/`, `/login` and `/app`, so this is not a header regression. P7-1, P7-2 and P7-3 stand as info, and this reviewer agrees that none of them blocks release.
- **N-R14-2 (note, e2e evidence): the W5 failure repeats at one call, and its desktop solo pass left no record.**
  - W5 failed 5 times out of 5 across the three loaded runs (the earlier e2e.log mobile, and both projects in e2e2.log and e2e3.log). Every time, it was `read ECONNRESET` on `POST /api/v1/invites/accept`, the first request on a fresh `playwright.request.newContext`.
  - Host load is the likely cause. Other ECONNRESETs hit `/api/v1/projects`, `/api/v1/evidence/{id}` and `/api/v1/auth/me`. `git diff v0.5.0..HEAD` does not touch w5-release.spec.ts, fixtures.ts, the invites routes or services, or the Caddyfile. W5 passed at 0.5.0 (44/44) and passed alone on mobile.
  - Only the mobile solo pass is on disk. The 0.6.0 e2e, smoke, baseline and deploy logs exist only in the session scratchpad. At 0.5.0 they were copied to artifacts/logs.
  - Recommendation: run W5 desktop alone once more on a quiet host. Then copy the 0.6.0 logs to artifacts/logs/ (e2e-0.6.0, smoke-0.6.0, baseline-0.6.0) and cite them in REPORT.md.
- **N-R14-3 (note, web-baseline evidence): the summary leaves out three facts, and no full 0.6.0 baseline run is clean apart from the accepted residuals.**
  - The summary does not mention that run 1's `buildFreshness` was a real local-artifact staleness, fixed by a local rebuild. The served build hash `3cfac19bd943` is the same in both runs, so the served image did not change.
  - It does not mention that two `--site-only` runs reported a referenced chunk unreachable. This round fetched that chunk with a 200.
  - It does not mention that the `headers` cell's text says every header is missing. That is a status-0 artefact, not the HSTS residual.
  - The "residuals = HSTS + Lighthouse" conclusion rests on direct header checks and the Lighthouse numbers. It is reasonable, but no run measured it. INV-2's current report (11 failures, 20 of 27 screens, with no `/app/orb`, which is N-R8-6) cannot be accepted as it stands.
  - The record should say so plainly. Either a quiet-host `--full` run, or an explicit human acceptance of INV-2 for 0.6.0, should come before the ship banner. Neither is a code defect.
- **N-R14-4 (note, smoke): smoke 27/27 ran against the 0.6.0 images, but before the commit label was corrected.** The health check in smoke2.log shows `commit 664deff`. deploy4 then recreated api, worker and scheduler with GIT_COMMIT=68c3881 (same images, env-only change), and health now reports 0.6.0 / 68c3881. The smoke was not re-run after deploy4. The risk is low, but REPORT.md should quote the smoke's health line as it was, or re-run `/smoke` once.
- **N-R14-5 (note, RB-15 / F-25): small gaps in docs and tests.**
  - The "optimum/best" wording is not recorded in ADR-47 or W10.3. Fold it into the M-R14-1 ADR amendment.
  - The legacy-result test does not assert that `data-found-at` is absent.
  - The missed-optimum fixture keeps a pass outcome, which no real result could have.
  - test_jobs_orb.py:137 still does not assert `foundAt` (the API half of N-R13-3).
- Carried forward: N-R8-6 / INV-2 (now N-R14-3). N-R13-2 (queryRatio is a hindsight figure) is unchanged and disclosed by SPEC §3. N-R13-4 is resolved: W10 ran green on both projects in e2e run 1.

Counts: 0 blocker, 0 major, 1 minor, 5 notes (plus M-R12-0 carried forward for a human).

## Carried forward (human action)

- **M-R12-0:** F-24 (c473132) has no governance decision. `npm run lint:invariants` reports `FAIL INV-1 c473132 "F-24: SDK Orb Lab (qai.orb, ORB-0)" has no governance decision`. Only a human can record it. This reviewer ran govern.mjs only as `queue`.
- **Queue:** C-21 (GD-0112) and RB-14 (GD-0116) await human approval. C-23 (GD-0120) is pending review, and this round is its review evidence.
- **INV-2:** fails with 11 unaccepted failures. It needs a clean quiet-host run or an explicit human acceptance for 0.6.0 (N-R14-3).

## Overall

- RB-15 fixes the Optimize cell correctly and in line with SPEC §3. Its new test covers the missed-optimum and older-result cases, and vitest, eslint and tsc pass. Its commit message overstates the change: the Compare hint is unchanged (M-R14-1, minor, worth one small docs-first follow-up).
- C-23 is a clean release bump. INV-34 passes, and the CHANGELOG and VERSION entries cover exactly v0.5.0..HEAD and describe it accurately.
- This reviewer agrees with security pass 7 that P7-1 to P7-4 need no fix before release. P7-4's data is out of date, and its header question is answered.
- The delivery characterisations hold up against the logs. The record should add the W5 pattern and missing desktop artifact, the extra baseline failures (buildFreshness, LCP, the site-only chunk), the stale smoke commit label, and that the residual conclusion is inferred.

Verdict: PASS (RB-15, C-23). No FAIL lines. The release still waits on the human governance actions (M-R12-0, the GD-0112, GD-0116 and GD-0120 queue) and on a decision about INV-2.
