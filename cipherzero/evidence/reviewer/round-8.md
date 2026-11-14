# Reviewer Gate — round 8 (re-entry R-3, Review Round 2, diff-scoped)
Commit reviewed: 80fc26b
Previous round reviewed: artifacts/reviewer/round-7.md (393294e)
Mechanism: Claude Code Agent tool, general-purpose subagent (fresh context), read-only except this file
Verdict: PASS

Inputs:
- artifacts/reviewer/round-7.md (the previous round; its FAIL item 5 is re-derived below)
- `git diff 393294e 80fc26b` (21 files; artifacts/ used only as context), and the working tree at 80fc26b for the files the diff names
- DECISIONS.md ADR-094 (scope context, as the brief directs), plus the changed parts of SPEC.md §10 (env table), ARCHITECTURE.md §6.5, CHANGELOG.md and VERSION.md
- .mdlc/BUILD.md, sections "Reviewer Gate" and "Review Round"
- Commands run (read-only):
  - `npx vitest run apps/prober/test/pki-mount.test.ts apps/prober/test/routes.test.ts apps/scanner/test apps/worker/test/domain-verify.test.ts apps/worker/test/external-discovery.test.ts`: 9 files and 105 tests passed. With the verbose reporter, the openssl-gated export test and the drip and budget tests all ran; none was skipped.
  - `npm run lint:invariants`: `60 total / 59 machine (58 pass, 1 fail, 0 zero-files) / 1 manual`. The only FAIL is INV-2, which is stale because the source changed since baseline ea0336b. The brief says to ignore it, since the definitive baseline runs at the wide close.

## Round-7 FAIL item (verbatim item text) re-derived
| # | Item | Verdict | Rationale (file:line) |
|---|---|---|---|
| 5 | Security invariants are upheld: no hardcoded secrets, OWASP basics handled, input validation at trust boundaries, secure defaults | PASS | Each runtime mTLS service now mounts only its own volume, read-only: trust-core compose.yaml:126 `pki-trust-core:/pki:ro`, api :156, worker :182, prober :213. The full `pki` volume, which holds ca.key and every leaf key, is mounted only by pki-init (compose.yaml:35). pki-init writes the exports with `--export <svc>:/x/<svc>` (compose.yaml:20-34). The export copies only `ca.crt`, `<svc>.crt` and `<svc>.key`, deletes anything else in the target directory, and sets the key to 0600 (deploy/pki/bootstrap.mjs:119-130). The tools stage creates `/x/<svc>` owned by node (Dockerfile:32-35), so a fresh named volume copies up that ownership and pki-init (node) can write it. apps/prober/test/pki-mount.test.ts checks this: each service has exactly `[pki-<svc>:/pki:ro]` (:32-34); pki-init has exactly the five mounts and passes each export arg; `pki:/` appears exactly once (:36-42); each `pki-<svc>:` appears three times (:44-49); and a real bootstrap run with a planted stale `ca.key` ends with only the three files in the export directory (:60-85). The test runs and passes. |

### Upgrade path and export pruning (brief check 1)
- **Upgrading an existing deployment.** The old `pki` volume keeps its material and ownership. pki-init still mounts it at `/pki` (compose.yaml:35), and its idempotent issuance keeps the unexpired CA and leaves (bootstrap.mjs:56-82).
- **New volumes.** The new `pki-*` volumes are empty on first mount. They take the node ownership of the image's `/x/<svc>` directory (Dockerfile:33-34). The export then copies the existing, unchanged material into them on every run of pki-init (bootstrap.mjs:119-130). Every service still `depends_on` pki-init completing (e.g. compose.yaml:128 and :221), so no service starts against an empty volume.
- **Pruning.** It is limited to the export target directory. No `--export` directory overlaps `/pki` (`/x/*` and `/pki` are siblings), so the prune cannot reach the CA's directory in this configuration.
- **Non-blocking hardening.**
  - bootstrap.mjs does not refuse an `--export` directory equal to or containing `--out`. For example, `--export prober:/pki` would delete `ca.key` and the other leaves. Only the compose file calls the script, so this is a footgun, not a defect.
  - The pki-mount test parses only flow-style `volumes: [...]`. A switch to block or long syntax makes the per-service equality check fail closed. However, a long-syntax `source: pki` mount added to another service would not be counted by the `pki:/` tally.
- **Operator note (non-blocking).** Earlier deployments ran the prober with read access to ca.key. ADR-094 and QUICKSTART do not suggest rotating the CA on upgrade, for example `docker volume rm` of `pki` and of each `pki-*` volume. That rotation is the only remedy if a pre-0.5.1 prober was ever compromised.

## Checklist (BUILD.md Reviewer Gate), diff-scoped
| # | Item | Verdict | Rationale (file:line) |
|---|---|---|---|
| 1 | RESEARCH requirements implemented, not stubbed | PASS | The diff removes no R-3 capability. The prober now really "holds no secrets worth taking": its volume has only ca.crt, prober.crt and prober.key (bootstrap.mjs:121, compose.yaml:213). |
| 2 | SPEC features have behavior tests | PASS | Each behaviour this batch adds has a test through its production path. Export and mounts: pki-mount.test.ts:32-85. The notAfter that does not parse: apps/scanner/test/tls.test.ts:91-95. The absolute handshake deadline against a dripping peer: tls.test.ts:97-116. The per-request port budget through `POST /probe-tls`: apps/prober/test/routes.test.ts:71-93. 503 BUSY through runDomainVerifyForOrg, leaving PROBER_DOWN_KEY unset: apps/worker/test/domain-verify.test.ts:164-176. Gaps (non-blocking): ProberBusyError has no test through external-discovery, and the uncaughtException handler has no test. |
| 3 | Architecture matches ARCHITECTURE.md | PASS | ARCH §6.5 (ARCHITECTURE.md:522) now describes the `pki` and `pki-<service>` split exactly as compose.yaml:15-36 and :291-295 implement it. The prober still has no DB or Redis access and sits only on probe-ctl and egress (compose.yaml:214). |
| 4 | Scope discipline | PASS | The new surface is `--export`, PROBE_BUDGET_PORTS and ProberBusyError. Each answers a round-7 or SEC finding, and no product surface was added. The web changes are a heading switched to `sr-only` (domains-panel.tsx:108, quantum-client.tsx:591; `.sr-only` is defined at globals.css:213) and codeblock wrapping (globals.css:663-664). Both are design-score fixes, and the accessible names are unchanged. |
| 5 | Security | PASS | See the re-derivation above. The SEC-29 and SEC-30 code also shrinks the prober's attack surface (see item 6 and the focus table). |
| 6 | Cold read: no obvious bugs | PASS | probeTls (apps/scanner/src/tls.ts:63-117): see "SEC-29/30 correctness" below. One non-blocking note there, N1 (a startup listen error is now swallowed), was introduced by this batch. |
| 7 | Runnable as documented | PASS | The compose topology stays self-contained: the new volumes are declared at compose.yaml:291-295 and the mount points exist in the image (Dockerfile:33-34). The existing QUICKSTART troubleshooting for a root-owned `pki` volume (QUICKSTART.md:97-98) is unaffected. Compose was not run, per the brief. |
| 8 | Manual invariants (one verdict each) | PASS | INV-11 "Every scheduled trigger has an effect test": PASS. The diff adds and changes no schedule (packages/core/src/queue.ts is untouched). The domain-verify cron effect test still passes (domain-verify.test.ts, 105/105 in the run above), and external-discovery's cron and startup effect tests are unchanged and pass. |
| 9 | Nothing declared is unreachable | PASS | Every new declaration has a writer and a test through it. `ProberBusyError`: written by `proberError` at prober-client.ts:81-85, which the three endpoints call (:177, :189, :207), tested at domain-verify.test.ts:164. `PROBE_BUDGET_PORTS` and the error "request time budget exhausted": written by routes.ts:155-160, tested at routes.test.ts:71. `--export`: called by compose.yaml:20-34, tested at pki-mount.test.ts:60. `notAfterOf`: called by tls.ts:108 and certs.ts:108, tested at tls.test.ts:91. |

Counts: 9 PASS / 0 FAIL / 0 UNCERTAIN.

## SEC-29/30 correctness (brief check 2)
| Area | Verdict | Rationale (file:line) |
|---|---|---|
| probeTls deadline, settle and try/catch | PASS | The deadline timer (tls.ts:77-78) is cleared on every exit: in `fail` (:81) and in `settle` (:111). If `settle` throws before :111, the catch at :94-96 calls `fail`, which clears it. The `settled` flag (:79, :83-84, :112) means an `error` after resolve (a late reset or the post-`end` idle `timeout`) destroys the socket but never rejects again. `socket.on('error', fail)` (:88) keeps an error listener for the socket's whole life, so a second error cannot become an uncaught `error` event. `fail` and `settle` are `const`s declared after their use sites, but they run only from async callbacks, so there is no temporal-dead-zone access. The `once('timeout')` listener is not removed after resolve. That is harmless, because it only destroys an already-ended socket, and it predates this batch. There is no double resolve: once the socket is destroyed, `secureConnect` cannot follow `fail`, and a second settlement would be a no-op anyway. |
| notAfterOf (tls.ts:58-61) and certs.ts:108 | PASS | Date.parse plus an isFinite check: a date that does not parse yields `{}` and no longer throws a RangeError from `toISOString`. `notAfter` is optional in the finding schema (the existing scanner tests validate reports against `reportSchema`). The cert-file scanner shares the same helper. |
| Port budget (apps/prober/src/routes.ts:127-160) | PASS | The budget is checked before each port. The last port started inside the budget can still run its full timeout, so the true bound is just under (PROBE_BUDGET_PORTS + 1) × timeout. The test asserts that bound (routes.test.ts:92). This is well inside the server's requestTimeout (main.ts:41, 8T + 4T + 10 s). Non-blocking: CHANGELOG.md says "at most four handshake timeouts" and ADR-094 says "at most PROBE_BUDGET_PORTS (4) handshake timeouts", but the code allows up to ~5. Ports skipped after the budget are reported unreachable, so a host with more than 4 filtered ports before an open one is under-reported. The `error` string makes this visible. |
| Process handlers (apps/prober/src/main.ts:51-56) | PASS (note N1) | They log and continue, as ADR-094 decides. **N1 (non-blocking, introduced by this batch).** `https.Server` has no `error` listener (server.ts:28-56). A `listen` failure such as EADDRINUSE or EACCES at main.ts:43 therefore becomes an uncaughtException. It is now only logged, and the process keeps running with nothing bound. The compose prober has no healthcheck and uses `restart: unless-stopped` (compose.yaml:200-222), so it stays up but dead instead of crash-restarting; the worker treats it as PROBER_UNAVAILABLE. Fix: add `server.on('error', (e) => { log.error(...); process.exit(1); })` before `listen`, or register the handlers inside the `listen` callback. Also, the comment at :51 says a stray throw "must fail that request". In practice that request hangs until `requestTimeout`, because nothing answers it. |
| Worker ProberBusyError and proberError (apps/worker/src/prober-client.ts:56-85) | PASS | ProberBusyError is not a subclass of ProberUnavailableError. The `instanceof ProberUnavailableError` checks that set PROBER_DOWN_KEY (jobs/domain-verify.ts:51, jobs/external-discovery.ts:98 and :264) therefore skip it. It is not a ProberLookupError either, so it is rethrown (domain-verify.ts:56; external-discovery.ts:102 and :268), and the job fails and retries as documented. In external-discovery, a busy answer mid-pool aborts that domain's whole run until the retry. That is the same outcome as before, minus the global down signal. Only `503` + `BUSY` maps to busy, and `500 BUSY` and `503 INTERNAL_ERROR` stay unavailable (tested at domain-verify.test.ts:165-167). |
| Leaked timers and listeners in the tests | PASS | tls.test.ts:113-115 clears the drip intervals and closes the server. routes.test.ts:89-90 closes the app and the silent listeners. |

## Docs, CHANGELOG and VERSION (brief check 3)
| Area | Verdict | Rationale |
|---|---|---|
| VERSION.md / CHANGELOG.md 0.5.1 | PASS | VERSION.md:3 says `current: 0.5.1`, and its history at :7-8 lists 0.5.1 and 0.5.0. The CHANGELOG.md:5-13 entries match the code: --export and the per-service volumes, notAfterOf, the absolute deadline, the process handlers, PROBE_BUDGET_PORTS, and BUSY without PROBER_DOWN_KEY. A wording nit ("at most four", see the port-budget row) is non-blocking. |
| SPEC / ARCH | PASS | SPEC.md:533 now says MTLS_* comes "from the service's own pki-<service> volume (ADR-094)". ARCHITECTURE.md:522 describes the split. Minor drift, older than this diff: ARCH:522 still lists the leaf certificates as "trust-core, api and worker" without the prober, although ARCH:468 and the bootstrap default (bootstrap.mjs:19) include it. Neither SPEC nor ARCH mentions the per-request probe budget; it lives only in ADR-094 and the code. Both are non-blocking. |
| package.json versions and compose APP_VERSION | Carried | Every package.json is still 0.1.0, and compose `APP_VERSION` defaults to 0.1.4. This is the known open item "version label" in BUILD-STATE and predates this diff. |

## Round-7 non-blocking note (brief check 4)
**Still open (non-blocking).** RUN_LOCK_MS is still 2 h (apps/worker/src/jobs/external-discovery.ts:178). The new budget lowers one host's worst case from about 8 × T to about 5 × T (routes.ts:155-160). Even so, 1,000 hosts × ~50 s across 2 lanes (with the default T = 10 s) is about 7 h, which is still longer than the lock. Overlap stays bounded by the idempotent `ext:<id>:<window>` scanId and the guarded ingest. Suggested follow-up: refresh the lock's TTL during the pool (e.g. every RECHECK_EVERY hosts), or set RUN_LOCK_MS from the domain's host count × ports × timeout.

## Findings
None blocking.

Non-blocking:
- N1 (prober main.ts, introduced by the batch): a startup `listen` error is swallowed by the new uncaughtException handler. Add a `server.on('error')` that exits, or register the handlers after `listening`.
- bootstrap `--export` has no guard against a target directory that overlaps `--out`.
- The pki-mount test parses flow-style YAML only.
- CHANGELOG and ADR-094 say "at most four", but the true bound is just under 5 × T.
- No suggestion to rotate the CA on upgrade.
- ARCH:522 omits the prober from the leaf list.
- The round-7 run-lock note remains open.
