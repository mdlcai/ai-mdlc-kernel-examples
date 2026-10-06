# Security Audit — Pass 3 (diff-scoped re-scan after the round-2 fix batch, read-only)

Audited commit: c230639 (RB-2), compared with pass 2 at a9ee654.

Date: 2026-10-05 · Depth: standard · review_gates: auto · Scope: checking the S2-1 … S2-3 dispositions in code and running their tests, reviewing `git diff a9ee654 c230639 -- server web compose.yaml .env.example` (20 files) for new issues, re-running the pass-2 scanner set against the tree and the **running** images rebuilt at c230639, and five live probes against https://localhost:9443 (compose project `threatwatch-mdlc`). No containers were rebuilt or restarted, and no accounts were registered.

Raw scanner output is in `artifacts/security/raw/pass-3/` (gitignored). Exit codes are in `exits.txt` (trivy image) and `exits-local.txt` (everything else). Image provenance is in `image-provenance.txt`.

Image provenance:
- `threatwatch-mdlc-api` and `-worker` carry label revision `c230639`, version 0.3.2, built 07:14:56Z.
- `threatwatch-mdlc-web` carries revision `c230639`, version 0.3.2, built 07:18:49Z.
- `threatwatch-mdlc-migrate` was built 07:14:58Z from the same `base` stage. It reads `unknown`/`0.0.0`, as in S2-I4.
- `/api/health` reports `{"version":"0.3.2","commit":"c230639"}` with db, temporal and listener all `ok`.
- RB-2 did not change `server/Dockerfile`, `web/Dockerfile`, `Caddyfile`, `compose.yaml` or `.github/` (`git diff --stat` is empty).

## Tool table

| Tool | Version | Command | Exit | Findings raw → after triage |
|---|---|---|---|---|
| vitest (targeted) | vitest 5.0.3 | `npx vitest run test/unit/notify/pinned-dispatch.test.ts test/unit/config/env.test.ts test/integration/auth/auth.test.ts` | 0 | 3 files, 29 tests passed |
| npm audit (all) | npm 10.9.3 / node 22.19.0 | `npm audit --json` | 0 | 0 → 0 |
| npm audit (prod) | npm 10.9.3 | `npm audit --omit=dev --json` | 0 | 0 → 0 |
| gitleaks (since pass 2) | zricethezav/gitleaks:v8.28.0 | `gitleaks git /src --redact --log-opts="a9ee654..c230639" --report-format json` | 1 | 1 → 0 (test fixture, see Dropped) |
| trivy config | aquasec/trivy:0.67.2 | `trivy config --timeout 30m --skip-dirs '**/node_modules,web/.next,.claude,.git,artifacts,server/src/generated' .` | 0 | 0 misconfigurations (server/Dockerfile, web/Dockerfile) |
| trivy fs | aquasec/trivy:0.67.2 | `trivy fs --timeout 30m --scanners vuln,secret --skip-dirs <same> --skip-files .env .` | 0 | 0 vulns (package-lock.json), 0 secrets |
| trivy image api (full) | aquasec/trivy:0.67.2 | `trivy image --scanners vuln,secret --format json threatwatch-mdlc-api:latest` | 0 | C4 H54 M109 L80 U1, **0 with a fixed version**, node-pkg 0, secrets 0 (identical to pass 2) |
| trivy image api (gate) | aquasec/trivy:0.67.2 | `trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 threatwatch-mdlc-api:latest` | **0** | 0 |
| trivy image worker (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | C4 H54 M109 L80 U1, all unfixed / 0 |
| trivy image migrate (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | C4 H54 M109 L80 U1, all unfixed / 0 |
| trivy image web (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | **0 vulnerabilities** (Alpine 3.24.2), secrets 0 |
| semgrep (changed files) | semgrep/semgrep:1.140.0 | `semgrep scan --config p/default --config p/typescript --config p/nodejs --config p/secrets --config p/dockerfile --config p/github-actions --metrics=off --json <20 files changed a9ee654..c230639>` (216 rules run) | 0 | 0 → 0, 0 parse errors |
| Manual review + live probes | — | S2-n disposition check, RB-2 diff review, headers/auth probes | — | INFO only |

Scanner notes:
- **trivy config/fs:** the first runs (`trivy-config.log`, `trivy-fs.log`, exit 1) hit trivy's default 5-minute timeout. The cause was a slow Windows bind mount (`helm scan error … context deadline exceeded` and `semaphore acquire: context deadline exceeded`), not a finding. I re-ran both with `--timeout 30m`, a read-only mount and glob skip-dirs (`trivy-config-retry.log`, `trivy-fs-retry.log`, exit 0). The retries detected the same targets as pass 2.
- **trivy image DB:** trivy 0.67.2 notes that 0.75.0 is available. The vulnerability DB was current, the same as pass-2 I9.

## Pass-2 finding verification

| ID | Pass-2 sev | Status | Evidence |
|---|---|---|---|
| S2-1 | LOW | **FIXED** | See S2-1 detail below. |
| S2-2 | LOW | **FIXED** (fail-closed variant of the suggested fix) | See S2-2 detail below. |
| S2-3 | LOW | **FIXED** | See S2-3 detail below. |
| S2-I2 | INFO | **FIXED** | `assets/service.ts` (last_seen list + activity), `ingest/batches.ts` (batches + errors) and `notify/log.ts` now use `pageCursor`. The risk-sorted asset list throws `invalidCursor()` on a bad cursor. `test/integration/cursors.test.ts` now covers `/notifications/log`, both asset sorts, integration batches/errors and asset activity. The pre-existing `assets.test.ts` expectation was corrected from 200 to 400 to match SPEC (the test was wrong, not weakened). |
| S1-8, S1-9, S1-12 | LOW | ACCEPTED (ADR-028), unchanged | Code and compose paths are untouched by RB-2. |
| S2-I1, S2-I3, S2-I4, S2-I5 | INFO | Unchanged | Carried. |

**S2-1 — FIXED.** Evidence:
- New `server/test/unit/notify/pinned-dispatch.test.ts` calls the production `dispatchDelivery` path with **no `fetchImpl`**. It mocks `assertSafeUrl` to approve the unresolvable host `rebind.test` at the local receiver's address `127.0.0.1`.
- The request only reaches the receiver if the socket lookup is pinned, and the test asserts that it does (`{ ok: true, statusCode: 204 }`). It also asserts that the `Host` header keeps `rebind.test:<port>` and that body and signature are delivered.
- A second case asserts that a 302 is a failed attempt and that exactly one request was made, so redirects are not followed.
- Dropping `lookup:` from `dispatch.ts:55`, or moving to `fetch` without a pinned dispatcher, now fails the test with a DNS error. Passed in this run.

**S2-2 — FIXED.** Evidence:
- `server/src/config/env.ts` adds a `superRefine` that rejects `SMTP_REQUIRE_TLS=false` when all of the following hold: `NODE_ENV=production`, `SMTP_PORT !== 465`, and `SMTP_HOST` (lowercased) is not one of `mailpit`, `localhost`, `127.0.0.1` or `::1`.
- The API (`api/main.ts:11`) and the worker, which sends the mail (`worker/main.ts:12`), both call `loadEnvOrExit()`, so a misconfigured deployment refuses to boot.
- The compose default (`SMTP_HOST=mailpit`, `SMTP_REQUIRE_TLS=false`) still boots. Pointing `SMTP_HOST` at a real relay without flipping the flag now fails closed instead of silently allowing opportunistic STARTTLS.
- The `.env.example:36` comment documents the rule.
- `test/unit/config/env.test.ts` covers five cases: mailpit allowed, external relay refused, external relay with the default allowed, 465 allowed, development allowed. Passed.

**S2-3 — FIXED.** Evidence:
- The `res.once('close', … releaseKey)` handler is removed from `server/src/api/authz/idempotency.ts`. After a disconnect, the handler's `res.json`/`res.end` wrappers still run, so a 2xx is stored with `finalizeKey` and a non-2xx releases the key.
- A process that dies leaves a placeholder. `claimKey` reclaims it after `IDEMPOTENCY_IN_FLIGHT_STALE_MS` (5 min) with a conditional delete.
- A retry while the handler is still running gets 409 `CONFLICT` (in flight), not a second execution.
- New test `auth.test.ts` "keeps the key when the client disconnects mid-handler…" aborts at 50 ms, waits, then retries. It asserts 201, `Idempotent-Replayed: true` and `calls === 1`. Passed.

## RB-2 diff review (a9ee654..c230639, server + web + compose.yaml + .env.example)

- **Idempotency (`idempotency.ts`).** Only the close-release was removed. The claim, replay, conflict and in-flight logic and the `guard → idempotent` mount order are unchanged. Two residual edges, neither a security issue:
  - A handler that hangs and never responds keeps the key `in_flight` (409) for up to 5 minutes, which is the intended behaviour.
  - A handler that takes more than 5 minutes could be taken over. The request timeouts make that unreachable, as noted in pass 2.
- **Env refinement (`env.ts`).** The check is strict and allow-list based.
  - Hosts not on the allow-list are refused, which is the safe direction. That includes other loopback literals (`127.0.0.2`, `[::1]`) and a trailing-dot `localhost.`.
  - The `mailpit` exemption is a name the operator controls. In production it could in principle point at a real relay through custom DNS, but that is a deliberate operator choice and not an attacker path (INFO S3-I1).
  - The `envSchema.superRefine` change keeps `z.infer` and typed consumers intact.
- **Cursor changes.** All new cursor values still flow into tagged `Prisma.sql` or Prisma filters with `::uuid`/`::timestamptz` casts. There is no `$queryRawUnsafe`/`Prisma.raw`. The `UUID_RE`/`isUuid` guards remain on top of `pageCursor`'s own validation. Bad cursors now give 400 instead of being silently ignored.
- **Web (`ui.tsx` `Breakable`, AlertsView, AssetDetail, Correlations, AppShell).**
  - `Breakable` splits a string with a fixed lookbehind regex `/(?<=[.\-/@])/`, which is linear with no backtracking, so there is no ReDoS. It renders the parts as React text nodes plus `<wbr />`, so there is no new HTML sink.
  - `PageHeader.title` is typed `string`.
  - The remaining web hunks are class-name and token changes only: the `text-sev-high` to `text-text` swap, `min-w-0`, and the skip-link offset.
  - The only `href` touched is the static `#main` skip link.
- **Test-only and version files.** `cursors.test.ts`, `assets.test.ts`, `env.test.ts`, `auth.test.ts`, the new `pinned-dispatch.test.ts`, the package versions 0.3.1 to 0.3.2, and a comment fix in `deliveries/drain.ts` (WS-OUT-6 to WS-OUT-5). Nothing ships to production from these beyond the version string.

No new CRITICAL, HIGH, MEDIUM or LOW findings.

## Findings

No new findings at LOW or above.

### INFO
- S3-I1: the `mailpit` and loopback exemption in the S2-2 refinement trusts the `SMTP_HOST` name. That is acceptable because only the operator sets it. If a production deploy ever runs a container named `mailpit` as a real relay, set `SMTP_REQUIRE_TLS=true` explicitly.
- S3-I2: trivy 0.67.2 is eight minor versions behind (0.75.0 is available). The DB is current, so results are valid. Bump it deliberately in CI along with S2-I5's pinned action.
- Carried: S2-I1 (I1, I3, I4, I5–I9, S1-7), S2-I3 (BusyBox wget in web), S2-I4 (migrate image has no provenance labels), S2-I5 (keep the trivy-action SHA pin). S2-I2 is **closed** (see the verification table).

## Dropped false positives

| Source | Item | Reason |
|---|---|---|
| gitleaks git | `server/test/integration/auth/auth.test.ts:415` `generic-api-key` | The Idempotency-Key literal `gone-12345678` in the new S2-3 test. Same class as pass-2's `race-…`/`flaky-…` fixtures, and not a secret. |
| trivy image | zlib1g CVE-2023-45853 (CRITICAL, api/worker/migrate) | minizip only. Debian `zlib1g` is not affected, the same as passes 1 and 2. |
| trivy image | api/worker/migrate CRITICAL/HIGH with no fixed version (perl-base, util-linux family, ncurses, libssl3/openssl deb, libsystemd0/libudev1, gzip, libacl1) | There is no upstream fix, and the counts are identical to pass 2 (C4 H54, 0 fixable). They are covered by the ADR-028 S1-3 residual acceptance and excluded from the CI gate by `--ignore-unfixed`. |

## Live probe results (all as expected)

Log: `artifacts/security/raw/pass-3/live-probes.log`. Five requests in total, with no registration and no writes.

- `GET /login` → 200.
  - CSP: nonce + `strict-dynamic`, `object-src 'none'`, `form-action 'self'`, `frame-ancestors 'none'`, `upgrade-insecure-requests`.
  - Other headers: COOP same-origin, Permissions-Policy, Referrer-Policy, nosniff, X-Frame-Options DENY, HSTS max-age=0 (ADR-025).
- `GET /api/health` → 200, with helmet CSP `default-src 'none'`, CORP/COOP same-origin, nosniff and Referrer-Policy. It reports commit `c230639`.
- Unauthenticated `GET /api/v1/alerts` → 401.
- `POST /api/v1/auth/login` with `Origin: https://evil.example` → 403 (CSRF same-origin, S3).
- `http://localhost:8088/` → 301 to `https://localhost:9443/`.

I did not re-check the in-container hardening (non-root, root-owned app tree, npm removed) in this pass. The write probe (`touch`) inside the containers was not permitted in this session. The Dockerfiles are byte-identical to a9ee654, where pass 2 verified S1-4 and S1-10 on the running images.

## Verdict

**PASS — the gate is met at pass 3. All three pass-2 LOWs are fixed and verified, and the re-scan is clean.**

| Severity | Open (pass 3) | IDs |
|---|---|---|
| Critical | 0 | — |
| High | 0 | — (0 fixable C/H on all four images; the unfixed residual is accepted in ADR-028) |
| Medium | 0 | — |
| Low | 0 new + 3 accepted | S2-1, S2-2, S2-3 are FIXED. Accepted (ADR-028): S1-8, S1-9, S1-12 |
| Info | 2 new + carried | S3-I1, S3-I2. Carried: S2-I1, S2-I3, S2-I4, S2-I5 (S2-I2 closed) |

Pass-2 → pass-3 delta: 3 LOWs fixed (S2-1, S2-2, S2-3) and 1 INFO closed (S2-I2). There are no new findings at any severity above INFO, and the scanner counts match pass 2 exactly.

| Gate criterion | Result |
|---|---|
| Zero CRITICAL residuals | MET. 0 fixable. The unfixed OS CRITICALs (perl-base ×3, plus the zlib1g false positive) are accepted with justification (ADR-028) and excluded by `--ignore-unfixed`. |
| Zero HIGH residuals | MET. 0 fixable on api/worker/migrate/web. The unfixed HIGHs are accepted (ADR-028). |
| Every MEDIUM dispositioned | MET. There are no open MEDIUMs. S1-1, S1-2 and S1-4 were FIXED in pass 2 and the code is unchanged. |
| Full test suite passing | PARTIAL (read-only audit). The targeted suites for every S2-n fix pass: 3 files, 29 tests, exit 0. The full suite is the build's responsibility (`npm test` / CI parity for RB-2). |
| Pass 3 re-scan clean | MET. The trivy gate view is 0 on all four images, npm audit is 0/0, gitleaks has 0 after triage, semgrep is 0, and trivy config and fs are 0. |
| `SECURITY-AUDIT.md` | N/A at `standard` depth. |

Open items: none blocking. The only optional follow-ups are INFO (S3-I1 and S3-I2, plus the carried S2-I3, S2-I4 and S2-I5).
