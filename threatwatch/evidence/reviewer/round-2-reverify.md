# Reviewer Round 2 re-verify: ThreatWatch (after revert-and-refix)

- **Reviewed commit:** c230639 ("RB-2: Review Round 2 revert-and-refix + Security pass-2 LOWs"). This is the round-2 re-verify after revert-and-refix (BUILD.md Review Round rule), diff-scoped as `git diff a9ee654 c230639`, plus the code at HEAD (HEAD = c230639).
- **Live stack:** `/api/health` and web `/healthz` both report `version 0.3.2, commit c230639`. All 7 compose services are healthy, including api and worker, which were rebuilt and restarted from c230639.
- **Mechanism:** an independent reviewer in a fresh context, read-only on source. Each R2-n verdict was re-derived from the diff and from HEAD, not from the RB-2 commit message or ADR-029.
- **Not reviewed:** these working-tree changes are not part of c230639:
  - BUILD-STATE.md, DECISIONS.md (ADR-029) and GOVERNANCE-DECISIONS.jsonl, which are modified
  - `.claude/`, which is untracked

## Re-run by reviewer

- `npx vitest run` over auth, cursors, assets, the notifications integration suites, unit `pinned-dispatch` and unit `config/env`: 6 files, 72/72 tests pass.
- Web component tests (`npm -w web test`): 6 files, 25/25 pass. `npm run typecheck` is clean.
- `npm run lint:invariants`: 43 invariants in total, 42 machine-checked, 41 of those pass.
  - INV-2 does not pass yet because `FRONTEND-AUDIT.json` is absent. That is unchanged from round 2 and is expected until the wide-close `--full` run.
  - The one manual invariant, INV-11, keeps its round-2 verdict. RB-2 did not touch the jobs or their tests.
- The web-baseline run in progress on c230639 is `FRONTEND-AUDIT.partial.summary.json`, gitSha c2306393, written 02:24.
  - It has measured 50 cells over 10 screens so far. None has a screen-cell issue and there are 0 regressions.
  - The only open item is site `buildFreshness`. That check compares the local `web/.next/BUILD_ID` with the source, not with the served build, and the served build is c230639.

## Round-2 findings re-verified

| R2 | Status | Evidence at c230639 |
|---|---|---|
| R2-1 Idempotency-Key lost on client disconnect (the round-2 regression) | RESOLVED | `server/src/api/authz/idempotency.ts:38-59`. The `res.once('close')` release is gone; this was the smaller re-fix that round 2 prescribed. `settled` is now set only by the `res.json` and `res.end` wrappers. So the handler's 2xx is always finalized, even after the client has gone away, and only a non-2xx releases the key. A placeholder from a request that died outright is still reclaimed after `IDEMPOTENCY_IN_FLIGHT_STALE_MS` (store.ts:11,32). The new test is auth.test.ts:409-421 ("keeps the key when the client disconnects mid-handler…"). It aborts the first POST at 50 ms against a 200 ms `/abandoned` handler, waits, retries with the same key, and asserts `201`, `Idempotent-Replayed: true` and `calls === 1`. That test discriminates: on a9ee654 the close-release would have deleted the placeholder and skipped finalize, so the retry would run the handler a second time (calls 2). The test passes at HEAD. The existing concurrent-duplicate test (one 201, one 409 `CONFLICT`) and the flaky-retry test also still pass. |
| R2-2 Web-baseline target-size and correlations overflow at 375 | RESOLVED | `web/src/components/AppShell.tsx:299-308`. `mb-6` under the Wordmark is restored. At `lg` the skip link moves to `left-[calc(var(--sidebar-width)+0.75rem)]`, so the focused skip link no longer overlaps the 44 px Wordmark target, which was the axe `target-size` overlap. Nav rows stay `min-h-8` (32 px), above the 24 px minimum. For the overflow, `CorrelationsView.tsx:155,175-179` adds `min-w-0` to the card and the Span cell, and splits the time range into two `whitespace-nowrap` spans that can wrap at the arrow. Evidence: the c230639 partial baseline shows 0 screen-cell issues and 0 regressions, and the orchestrator's narrow re-measure run (`web-baseline` with the re-check flag) over dashboard, alerts, correlations, correlation-detail and asset-detail was also clean. The definitive `--full` run (INV-2, R1-23) is still due at the wide close. |
| R2-3 Five lists ignored a malformed cursor | RESOLVED | `ingest/batches.ts:75,90` and `notify/log.ts:52` now use `pageCursor`. `assets/service.ts:126` throws `invalidCursor()` on the risk sort, :130 uses `pageCursor` for last_seen, and :336 covers asset activity. A grep shows no remaining list service calls a lenient decoder: all 14 cursor sites go through `pageCursor` or a strict decoder that throws. `cursors.test.ts` now covers `/notifications/log`, both asset sorts, and integration batches, integration errors and asset activity (with fixture ids). For each it checks 3 crafted cursors (400 with `errors[{field:cursor, code:INVALID_CURSOR}]`) and one well-formed cursor (200). All pass. |
| R2-4 Production pinned dispatch untested | RESOLVED | `server/test/unit/notify/pinned-dispatch.test.ts` drives `dispatchDelivery` without `fetchImpl`, so it exercises `pinnedPost` at dispatch.ts:50-64. `assertSafeUrl` is mocked to approve `rebind.test`, which does not resolve, at 127.0.0.1. The request reaches the local receiver only through the pinned `lookup`. The test asserts the Host header is kept, and that the body and signature arrive verbatim. It also asserts that a 302 is not followed: exactly one hit, `{ok:false,statusCode:302}`. The suggested 503 and timeout cases were not added. Both go through the same status mapping (dispatch.ts:93-94), and the network-error classifier in dispatch.ts:41-47 is already unit-tested, so this is not raised again. |
| R2-5 New RB-1 UI controls without tests | ACCEPTED-NOTE | Unchanged in RB-2. ADR-029 defers it, and it carries to the final re-score: the alerts from/to and asset filters, the endpoint enable switch, the `?version` highlight, and the settings access-denied states. The server side of each is covered. |
| R2-6 Doc drift | OPEN (NOTE, partial) | Fixed: ARCHITECTURE.md:30 (pinned IP), ARCHITECTURE.md:137 (dlq on the 6th attempt), SPEC S4 (600/min/IP ingest, ADR-028), the SPEC WS-OUT-5 pinned-POST wording, and the drain.ts:12 citation (now WS-OUT-5). Still drifting: see R2R-2. |
| R2-7 A finalize write error blocks the key for 5 min | ACCEPTED-NOTE | Unchanged (idempotency.ts:39,47,56 still only `warn`). The window is bounded by `IDEMPOTENCY_IN_FLIGHT_STALE_MS`, and the round-2 disposition was that it is acceptable as-is. It carries as a residual. |

## Regression scan of the RB-2 diff

- **`env.ts` superRefine (S2-2):**
  - **What it does:** it rejects `NODE_ENV=production` combined with `SMTP_REQUIRE_TLS=false` unless the port is 465 or the host is `mailpit`, `localhost`, `127.0.0.1` or `::1`.
  - **Shipped compose is safe:** compose.yaml:6,20,26 sets `NODE_ENV: production`, `SMTP_HOST` defaulting to `mailpit` and `SMTP_REQUIRE_TLS` defaulting to `false`, which passes the guard. .env.example uses `mailpit` with `development`. The live api and worker both started at c230639 through `loadEnvOrExit` (api/main.ts:11, worker/main.ts:12) and are healthy.
  - **External relay:** an operator who points `SMTP_HOST` at a real relay but leaves the compose default `false` is now refused at startup that names `SMTP_REQUIRE_TLS`. That is the intended closed-by-default behaviour, and the .env.example comment documents it.
  - **Tests:** unit env.test.ts:38-44 covers the guard for mailpit, an external relay on 587, the default true, port 465 and development.
- **`Breakable` (web/src/components/ui.tsx:137-150):**
  - **What it does:** it splits on a zero-width lookbehind `(?<=[.\-/@])`, so it adds `<wbr>` break points without changing the text content, and it uses keyed spans.
  - **Where it is used:** `PageHeader`'s `title` is typed `string` (ui.tsx:111), and the AlertsView asset cells already guard against null with `a.asset ? … : '—'`.
  - **Effect on the accessible name and tests:** the `<wbr>` elements and split spans leave the text and the accessible name unchanged. The web component tests pass and typecheck is clean.
- **assets.test.ts:142 (`?cursor=garbage` now expects 400, not 200):**
  - This aligns the test with the spec; it does not weaken it.
  - SPEC §7 (SPEC.md:619, amended by ADR-027) says a malformed list cursor is the 400 validation problem with `errors[{field:cursor, code:INVALID_CURSOR}]`.
  - The old 200 encoded the very silent-restart behaviour that R2-3 flagged. The assertion is now stricter, and cursors.test.ts asserts the exact problem body.
- **Other web changes:**
  - In AssetDetail and CorrelationDetail, `text-sev-high` becomes `font-medium text-text` for the negative event outcome. That still uses design tokens and has no colour-only meaning.
  - The CorrelationDetail pivot heading uses token sizes.
  - Nothing hard-coded was introduced.

## New findings

**R2R-1 · NOTE · The S2-2 production SMTP guard is not yet in SPEC, and ADR-029 is not committed**
- **Where:**
  - SPEC.md:654: the `SMTP_REQUIRE_TLS` row says it is read by `worker` only and says nothing about the startup refusal.
  - The guard is in server/src/config/env.ts:175-184, which both api/main.ts:11 and worker/main.ts:12 load.
  - ADR-029, which records the revert-and-refix and S2-2, is only in the working-tree DECISIONS.md. It is not in c230639.
- **What's wrong:** a new production startup refusal is behaviour, and the CLAUDE.md guardrail requires it in SPEC with an ADR. The code and tests are correct.
- **Fix:**
  - Amend SPEC.md:654 to read: "`true` by default; `false` only for the bundled Mailpit or a loopback relay. With `NODE_ENV=production` the api and worker refuse to start if it is `false` for any other host (except port 465). ADR-029". Change the reader column to `api, worker`.
  - Commit DECISIONS.md (ADR-029) with the next RB/F commit.

**R2R-2 · NOTE · Residual doc drift left from R2-6**
- **Where and what's wrong:**
  - SPEC.md:289 (WS-OUT-5) still says the drain "selects `pending` rows". The claim query at server/src/services/deliveries/drain.ts:53 also picks up rows in the retry-backoff status, as long as the endpoint is enabled.
  - ARCHITECTURE.md:253 (§7 SSRF) still lists `redirect:'error'`. The production path is the pinned `http.request`, which never follows redirects (dispatch.ts:50-64).
  - ARCHITECTURE.md:247 (§7 rate limits) omits the 600/min/IP ingest limiter (rate-limit.ts:32, ADR-028).
- **Fix:** documentation only, no behaviour change.
  - In WS-OUT-5, change "selects `pending` rows" to name both claimed statuses (pending and the retry-backoff status).
  - In ARCHITECTURE.md:253, replace `redirect:'error'` with "redirects not followed (a 3xx is an unsuccessful attempt)".
  - In ARCHITECTURE.md:247, add "+ 600/min/IP" to the ingest rate limit.
  - Cite ADR-029.

## Counts

| Item | Count | Detail |
|---|---|---|
| Round-2 items | 7 | 4 RESOLVED (R2-1, R2-2, R2-3, R2-4), 2 ACCEPTED-NOTE (R2-5, R2-7), 1 OPEN at NOTE severity (R2-6, continued as R2R-2) |
| New items | 2 | R2R-1 and R2R-2, both NOTE |
| Blocking severity | 0 | No blocking or WARN items remain open |
| Reviewer Gate item 6 (cold read of primary flows) | passes | Idempotent POST retry after a client timeout now replays (R2-1 resolved). Items 1-5 and 7-9 keep their round-2 verdicts. INV-2 still waits on the wide-close `--full` baseline. |

Verdict: PASS
