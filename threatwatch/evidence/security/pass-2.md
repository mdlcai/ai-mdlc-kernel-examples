# Security Audit — Pass 2 (re-scan after remediation, read-only)

Audited commit: a9ee654 (RB-1), compared with pass 1 at d700300.

Date: 2026-10-05 · Depth: standard · review_gates: auto · Scope: verifying every S1-n fix in code, re-running the pass-1 scanners against the tree and the **running** images rebuilt at a9ee654, reviewing `git diff d700300 a9ee654 -- server web` for new issues, and a small set of live probes against https://localhost:9443 (compose project `threatwatch-mdlc`).

Raw scanner output is in `artifacts/security/raw/pass-2/` (gitignored). Exit codes are in `exits.txt` (trivy image) and `exits-local.txt` (everything else).

Working-tree note: during this audit the working tree also held **uncommitted** RB-2 edits on top of a9ee654 (14 files under `server/` and `web/`). This report audits the committed a9ee654 and the images built from it. I read those edits and found no security regressions. They remove the `res.once('close')` key release from `server/src/api/authz/idempotency.ts`, which addresses S2-3. They move assets, ingest batches and notify log to `pageCursor`/`invalidCursor`, which addresses S2-I2. They also add a text-only `<Breakable>` wbr splitter in `ui.tsx`. Pass 3 must confirm these against the committed tree and rebuilt images.

Image provenance: `threatwatch-mdlc-api`/`-worker` (label revision `a9ee654`, version 0.3.1, built 06:09Z), `threatwatch-mdlc-web` (revision `a9ee654`, built 06:31Z), and `threatwatch-mdlc-migrate` (built 06:11Z from the same `base` stage; it has no provenance label because compose passes no build args to it). The api/worker/web containers were healthy when probed.

## Tool table

| Tool | Version | Command | Exit | Findings raw → after triage |
|---|---|---|---|---|
| npm audit (all) | npm 10.9.3 / node 22.19.0 | `npm audit --json` | 0 | 0 → 0 |
| npm audit (prod) | npm 10.9.3 | `npm audit --omit=dev --json` | 0 | 0 → 0 |
| gitleaks (history since pass 1) | zricethezav/gitleaks:v8.28.0 | `gitleaks git /src --redact --log-opts="d700300..a9ee654" --report-format json` | 1 | 3 → 0 (fixtures, see Dropped) |
| trivy config | aquasec/trivy:0.67.2 | `trivy config --skip-dirs node_modules,…,.claude,.git /src` | 0 | 0 misconfigurations (server/Dockerfile, web/Dockerfile) |
| trivy fs | aquasec/trivy:0.67.2 | `trivy fs --scanners vuln,secret --skip-dirs node_modules,web/.next,.claude,artifacts,.git --skip-files /src/.env /src` | 0 | 0 vulns (package-lock.json), 0 secrets → 0 |
| trivy image api (full) | aquasec/trivy:0.67.2 | `trivy image --scanners vuln,secret threatwatch-mdlc-api:latest` | 0 | C4 H54 M109 L80 U1 (all **unfixed**), node-pkg 0, secrets 0 (pass 1: C9 H108 M163 L92 U4) |
| trivy image api (gate) | aquasec/trivy:0.67.2 | `trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1 threatwatch-mdlc-api:latest` | **0** | 0 |
| trivy image worker (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | C4 H54 M109 L80 U1, all unfixed / 0 (same base as api) |
| trivy image migrate (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | C4 H54 M109 L80 U1, all unfixed / 0 (one-shot job, same base) |
| trivy image web (full / gate) | aquasec/trivy:0.67.2 | same as above | 0 / **0** | **0 vulnerabilities** (Alpine 3.24.2, 19 OS packages, 24 node packages), secrets 0 (pass 1: C5 H58 M49 L55) |
| semgrep (changed files) | semgrep/semgrep:1.140.0 | `semgrep scan --config p/default --config p/typescript --config p/nodejs --config p/secrets --config p/dockerfile --config p/github-actions --metrics=off --json <109 files changed d700300..a9ee654>` (255 rules) | 0 | 8 → 0 (see Dropped). 4 partial-parse errors, see note |
| GitHub API | gh | `gh api repos/<action>/git/refs/tags` for each pinned SHA | 0 | All 3 SHAs exist and match their commented tags |
| Manual review + live probes | — | S1-n fix verification, RB-1 diff review (idempotency, PATCH endpoint, cursors, web components), headers, WS, ingest limiter | — | S2-1 … S2-3, INFO |

Semgrep note: `alerts/[id]/AlertDetail.tsx`, `rules/_parts.tsx` (the `ATT&CK` JSX text, as in pass 1) and `integrations/[id]/IntegrationDetail.tsx` parsed only partly. I reviewed the RB-1 hunks of all three by hand, along with a grep of the whole web diff for `dangerouslySetInnerHTML`, `innerHTML`, `href={`/`src={`, `window.open`, `eval`, `new Function` and `location.*`. The only `dangerouslySetInnerHTML` is still the nonce'd static theme script (`web/src/app/layout.tsx:38`, INV-18). Every new `<Link href>` is a fixed `/…/` prefix plus a server UUID or an `encodeURIComponent` value, so none can carry a `javascript:` URL.

## Pass-1 fix verification

| ID | Pass-1 sev | Status | Evidence |
|---|---|---|---|
| S1-1 | MEDIUM | **FIXED** | `server/src/services/deliveries/dispatch.ts:34-39` `pinnedLookup` always answers with the `assertSafeUrl` address (both `all` and single forms). `:50-64` `pinnedPost` uses `http(s).request(safe.url, { lookup: pinnedLookup(safe.address, safe.family), signal })`. TLS SNI and certificate checks still use the URL hostname. Redirects are not followed, because `node:http` never follows them and any 3xx is a failed attempt (`:94-95`). Errors come back only as `timeout`/`tls_failed`/`connect_failed` (`:42-48`, `:96-99`). The re-validation runs on every dispatch (`:67-72`). The unit test covers `pinnedLookup` and `failureClass` (`server/test/unit/notify/signing.test.ts:85-101`). See S2-1 for the end-to-end test gap. |
| S1-2 | MEDIUM | **FIXED** | `method="post"` is on `LoginForm.tsx:40`, `RegisterForm.tsx:47` and `AccountView.tsx:93`. These are the only three `type="password"` forms in `web/src`. Live: `GET /login?email=…&password=x` → 200, a harmless page render (the SPA never builds that URL now). A no-JS native `POST /login` → 200, with the body discarded by Next and not logged by Caddy. |
| S1-3 | HIGH | **FIXED** (fixable part). Residual unfixed CVEs remain ACCEPTED per ADR-028. | `server/Dockerfile:17-18` uses `node:22-bookworm-slim` plus `apt-get upgrade -y` and no wget. `web/Dockerfile:17-18` uses `node:22-alpine` plus `apk upgrade`. The running images are Debian 12.15 / Alpine 3.24.2 / node v22.23.3 (pass 1: 12.12 / 3.22.1 / 22.19). Trivy gate view (`--ignore-unfixed` C/H) is **0 on all four images**. In the full view every remaining api/worker/migrate CRITICAL/HIGH has **no fixed version**: perl-base ×3 C + 5 H, util-linux family, ncurses/libtinfo, Debian libssl3/openssl CVE-2026-84782, libsystemd0/libudev1 CVE-2026-16742, gzip CVE-2026-41992, libacl1 CVE-2026-54369, and zlib1g CVE-2023-45853 (a minizip false positive as in pass 1). Healthchecks use node `http` (`server/Dockerfile:39-40,44-45`, `web/Dockerfile:36-37`). CI gates the images (`.github/workflows/ci.yml` `images` job, trivy-action `--severity CRITICAL,HIGH --ignore-unfixed --exit-code 1` ×3). |
| S1-4 | MEDIUM | **FIXED** | npm/npx/corepack/yarn are removed in the runtime stages (`server/Dockerfile:18`, `web/Dockerfile:18`). In the running containers, `which npm npx corepack` finds nothing and `/usr/local/lib/node_modules` is empty. Trivy node-pkg findings went from 3 C + 39 H per image to 0. Migrate runs `node ../node_modules/prisma/build/index.js migrate deploy` (`server/Dockerfile:52`). |
| S1-5 | LOW | **FIXED** | `server/src/api/middleware/rate-limit.ts:31` `ingestIpLimiter` (600/min/IP), mounted at `server/src/api/app.ts:39` before same-origin, body parsing and routing, so it runs before any integration lookup. `trust proxy` is 1 (Caddy). Live: three bad-signature posts to three random integration ids show the shared IP bucket going down (`600-in-1min; r=599 → 598 → 597`), while each per-integration bucket is fresh (`120-in-1min; r=119`). It was not hammered. |
| S1-6 | LOW | **FIXED** | `server/src/lib/crypto.ts:27-29` rejects any IV other than 12 bytes or tag other than 16 bytes, and uses `createDecipheriv(..., { authTagLength: 16 })`. Semgrep `gcm-no-tag-length` no longer fires on the file. |
| S1-7 | INFO | Unchanged (spec-sanctioned) | — |
| S1-8 | LOW | ACCEPTED (ADR-028) | — |
| S1-9 | LOW | ACCEPTED (ADR-028) | Temporal and Mailpit UIs are still loopback-only (`127.0.0.1:8233`, `127.0.0.1:8026`). |
| S1-10 | LOW | **FIXED** | Running as `uid=1000(node)`: `/app/server/dist`, `/app/node_modules` and `/app/web/server.js` are root-owned, and `touch` on them gives `Permission denied`. Only `/app/web/.next/cache` is owned by node. Only the migrate stage chowns the prisma engine dirs (`server/Dockerfile:49-51`). |
| S1-11 | LOW | **FIXED in code**, with a deploy-default caveat (S2-2) | `server/src/services/email/transport.ts:20-21` sets `requireTLS: port !== 465 && SMTP_REQUIRE_TLS`, and `server/src/config/env.ts:38` defaults to `true`. `compose.yaml:26` and `.env.example:37` default it to `false`. |
| S1-12 | LOW | ACCEPTED (ADR-028) | RB-1 also queues the temporary password on **reset** (`server/src/services/team/service.ts:291-309`). It uses the same `temp_password` template key, so the sent/dlq scrub in `services/email/drain.ts:53,72,88` covers it. The posture is unchanged. |
| S1-13 | LOW | **FIXED** | `actions/checkout@fbc6f399… # v5.1.0`, `actions/setup-node@a0853c24… # v5.0.0`, `aquasecurity/trivy-action@57a97c7e… # 0.35.0`. The GitHub API confirms each SHA is the commented tag. Top-level `permissions: contents: read`. |

## RB-1 diff review (d700300..a9ee654, server + web)

- **Idempotency store** (`server/src/services/idempotency/store.ts`, `server/src/api/authz/idempotency.ts`, migration 0002). The placeholder is claimed with `createMany … skipDuplicates` (`INSERT … ON CONFLICT DO NOTHING`) under `UNIQUE(org_id, key)`, so of two concurrent first requests exactly one runs and the other gets 409 `CONFLICT`. Taking over a stale or expired row deletes conditionally on `(id, createdAt)`, so two concurrent takers cannot both win. Finalize and release are conditional on `statusCode IS NULL`. `idempotent()` is always mounted **after** `guard(...)` (`rules.ts:80`, `indicators.ts:72,77`, `integrations.ts:166,187`, `notifications.ts:184,231`, alerts/reports `once`), so a replay never skips the permission check. No secret-returning create endpoint (integration create, delivery-endpoint create, team create/reset) uses `idempotent()`, so the org-wide key scope (I4) cannot replay a one-time secret to a teammate. One correctness edge remains: S2-3.
- **PATCH `/api/v1/delivery-endpoints/:id`** (`notifications.ts:209-213`, `endpoints.ts:146-163`). It requires `policies:write`, the `id` is validated as a UUID, and the body is strict `{enabled: boolean}`. It uses `updateMany({ where: { id, orgId } })` and returns 404 when the count is 0, and the read-back is org-scoped. A foreign id gives 404 and the action is audited, which matches the tenant convention. The drain's new `EXISTS (… e.enabled)` clause sits inside the parameterized `$queryRaw`.
- **Org-scoped writes.** The `deleteMany`/`updateMany` conversions (endpoints, indicators, policies, rules `updateMany` now adds `orgId`, team `resetPassword`) all add `orgId` and map count 0 to 404. They tighten tenant scoping and do not widen it.
- **Cursor parsing** (`server/src/lib/pagination.ts:19-44`, `alerts/queries.ts:45,68`). Cursor ids must match a UUID regex before they reach `${id}::uuid` (events, rules) or a Prisma uuid filter, which closes the tampered-cursor 500. All raw SQL uses tagged `$queryRaw`/`Prisma.sql`. There is no `$queryRawUnsafe` or `Prisma.raw` outside the generated client. Three lists still use `decodeCursor` and quietly fall back to page 1 on a bad cursor instead of returning 400 (INFO S2-I2). That is not a security issue.
- **Register validation** (`routes/auth.ts:30-41`). It only adds password-policy messages to the 400. Nothing is echoed back except field codes.
- **WebSocket** (`ws/hub.ts:151-152`). It now refuses `mustChangePassword` sessions with 428, which closes pass-1 I2.
- **Worker `/healthz`** now reports `version`/`commit`. It is unpublished (internal network only), the same as I1.
- **Web.** The new `DataTable`, `FilterBar`, `Stat` and `AssetFilter` components and the reworked views render API data as React text nodes only. `AssetFilter` sends the URL `?assetId=` through `encodeURIComponent` to a same-origin GET, and the server validates `assetId` as `z.uuid()` (`routes/alerts.ts:41`) and scopes it by org. There are no new sinks.

## Findings

### S2-1 — LOW — S1-1 has no end-to-end regression test for the pinned connect
- **File:** `server/test/unit/notify/signing.test.ts:85-101`. It tests `pinnedLookup` and `failureClass` in isolation. The rebinding test at `server/test/unit/lib/ssrf.test.ts:32` covers only the check, not the connect. `dispatchDelivery` tests go through `fetchImpl` (`dispatch.ts:88-91`), which bypasses `pinnedPost`.
- **Clause:** ARCH §7 webhook egress ("pins the resolved IP"), INV-10. Pass-1 S1-1 asked for a "check-public / connect-private" test.
- **Scenario:** a later refactor of `pinnedPost` could drop `lookup:` (for example, a move to `fetch`/undici without a dispatcher) and every test would still pass. The DNS-rebinding TOCTOU would come back silently.
- **Fix:** add a test that calls `dispatchDelivery` **without** `fetchImpl` and with `allowHttp: true`. Use a resolver that returns a public-looking address (for example `203.0.114.1`, which is outside the blocklist) and spy on `http.request` (`vi.spyOn(http, 'request')`). Assert that `options.lookup` is set and that calling it for the URL host gives back `203.0.114.1`. Alternatively, point the test at a local receiver on `127.0.0.1` with a resolver that says "public" and assert the call fails with `connect_failed` rather than reaching the receiver.

### S2-2 — LOW — The shipped compose and `.env.example` default `SMTP_REQUIRE_TLS=false`
- **File:** `compose.yaml:26` (`SMTP_REQUIRE_TLS: ${SMTP_REQUIRE_TLS:-false}`), `.env.example:37` (`SMTP_REQUIRE_TLS=false`). The code default `server/src/config/env.ts:38` is `true`.
- **Clause:** ARCH §7 OWASP floor (A02). S1-11 fix intent in ADR-028: "only the bundled Mailpit sets it to false".
- **Scenario:** an operator follows QUICKSTART, copies `.env.example`, and points `SMTP_HOST`/`SMTP_PORT=587` at a real relay without noticing the extra flag. STARTTLS is then opportunistic again, so an on-path attacker can strip it and read alert and temporary-password mail. This is the same exposure as S1-11, now as an operator footgun instead of a code default.
- **Fix:**
  - Default it to true in compose (`${SMTP_REQUIRE_TLS:-true}`) and set `SMTP_REQUIRE_TLS: 'false'` only on the bundled Mailpit path. One way is to make it conditional on the host being `mailpit` in `env.ts`: `requireTLS` false only when `SMTP_HOST === 'mailpit'`.
  - Or make `env.ts` refuse to start when `NODE_ENV=production`, `SMTP_REQUIRE_TLS=false` and `SMTP_HOST` is not `mailpit`/loopback.
  - Either way, put a comment on `.env.example:37` saying it must be true for any real relay.

### S2-3 — LOW — Idempotency key released on client disconnect while the handler is still running
- **File:** `server/src/api/authz/idempotency.ts:58-62` (`res.once('close', …)` → `releaseKey`), together with `:41-47`. A later `res.json` sees `settled === true` and skips `finalizeKey`.
- **Clause:** SPEC §3 Idempotency-Key semantics (exactly one execution per key). Correctness rather than a confidentiality or authorization issue.
- **Scenario:**
  1. A client sends `POST /rules` with key K and drops the connection, or a proxy times out, before the handler finishes.
  2. The `close` handler deletes the placeholder.
  3. The handler then commits the rule, and nothing is stored for K.
  4. The client's retry with K claims a fresh placeholder and creates a second rule, along with a second audit row or notification for other endpoints.
  The 5-minute stale takeover (`store.ts:12,36`) has the same effect for a handler that runs longer than 5 minutes, which is unlikely given the request timeouts.
- **Fix:** in the `close` handler, do not release while the handler may still be running. Leave the placeholder so the wrapped `res.json`/`res.end` still finalizes or releases it. That means not setting `settled` there, because the wrappers run even when the socket is gone. Rely on `IDEMPOTENCY_IN_FLIGHT_STALE_MS` to recover a placeholder only when the process died. Add a test that aborts the request mid-handler and asserts a retry replays (or gets 409 while the handler is still in flight) instead of running twice.

### INFO
- S2-I1: carried-over INFO items I1 (public health/openapi; worker `/healthz` adds version/commit, unpublished), I3 (`::/96` still not in `server/src/lib/ssrf.ts` V6 list), I4 (idempotency scoped per org rather than per user; safe today because no secret-returning endpoint is idempotent and `guard` runs first), I5, I6 (HSTS max-age=0 local, ADR-025), I7, I8, I9 (trivy 0.67.2 with the current DB). I2 is **closed** by the WS 428 change.
- S2-I2: `server/src/services/assets/service.ts:129,335`, `server/src/services/ingest/batches.ts:75,90` and `server/src/services/notify/log.ts:52` still call `decodeCursor` and quietly ignore an invalid cursor, while other lists return 400 `INVALID_CURSOR`. UUID validation in `decodeCursor` prevents any 500. Switch them to `pageCursor` for consistency (reviewer R1 scope).
- S2-I3: `/usr/bin/wget` in the web image is the BusyBox applet from the Alpine base (no CVEs, unused). Removing it is optional.
- S2-I4: the migrate image has no `APP_VERSION`/`GIT_COMMIT` build args in `compose.yaml`, so its labels read `0.0.0`/`unknown`. This is a provenance gap only.
- S2-I5: the trivy-action SHA is pinned and matches tag `0.35.0` today. Keep it pinned and bump it deliberately, because tags of that action have been force-pushed before.

## Dropped false positives

| Source | Item | Reason |
|---|---|---|
| gitleaks git | `scripts/smoke-test.mjs:13` `PASSWORD = '…'` | The password for throwaway orgs that the smoke test registers itself on a local stack. It is not a credential for any real account. |
| gitleaks git | `server/test/integration/auth/auth.test.ts:388`, `:392` `key: '…'` | Idempotency-Key literals (`race-12345678`, `flaky-12345678`) in tests, not secrets. |
| semgrep | `detect-non-literal-regexp` ×8 (`web/e2e/w03…w07*.spec.ts`) | Playwright specs building regexes from their own fixture names. No untrusted input, and nothing ships. |
| trivy image | zlib1g CVE-2023-45853 (CRITICAL, api/worker/migrate) | minizip only. Debian `zlib1g` does not ship it (not-affected), same as pass 1. |
| trivy image | api/worker/migrate CRITICAL/HIGH with no fixed version (perl-base, util-linux family, ncurses, libssl3/openssl deb, libsystemd0/libudev1, gzip, libacl1) | No upstream fix, and not reachable from the app: no `child_process`, and Node links its own OpenSSL. They are covered by the ADR-028 S1-3 residual acceptance and excluded from the CI gate by `--ignore-unfixed`. Re-scan each release. New since pass 1: libsystemd0/libudev1 CVE-2026-16742, gzip CVE-2026-41992, libacl1 CVE-2026-54369 (all unfixed). They fall under the same acceptance rationale. |

## Live probe results (all as expected)
Log: `artifacts/security/raw/pass-2/live-probes.log`. About 15 requests in total.
- `GET /login?email=…&password=x` → 200, a plain page render (harmless; see S1-2). A native form `POST /login` → 200.
- `/login` carries a nonce + `strict-dynamic` CSP with `object-src 'none'`, `form-action 'self'` and `frame-ancestors 'none'`, plus COOP same-origin, Permissions-Policy, Referrer-Policy, nosniff, X-Frame-Options DENY and HSTS max-age=0 (ADR-025).
- `/api/health` carries helmet CSP `default-src 'none'`, CORP/COOP same-origin, nosniff and Referrer-Policy.
- Unauthenticated `/api/v1/alerts` → 401. Login with a foreign `Origin` → 403.
- Ingest with a bad signature → 401, and both `600-in-1min` (per IP, shared) and `120-in-1min` (per integration) RateLimit headers are present (S1-5).
- WS `/ws`: foreign Origin → 403, no Origin → 403, same-origin with no session → 401.
- `http://localhost:8088/` → 301 to `https://localhost:9443/`.

## Verdict

**PASS — gate met at pass 2.**

| Severity | Open (pass 2) | IDs |
|---|---|---|
| Critical | 0 | — |
| High | 0 | — (S1-3 fixed: 0 fixable C/H on all images; unfixed residual accepted in ADR-028) |
| Medium | 0 | — (S1-1, S1-2, S1-4 fixed and verified) |
| Low | 3 new + 3 accepted | New: S2-1, S2-2, S2-3. Accepted (ADR-028): S1-8, S1-9, S1-12 |
| Info | 5 new + carried | S2-I1 … S2-I5 (carries I1, I3–I9, S1-7) |

Pass-1 → pass-2 delta: 1 HIGH and 3 MEDIUM fixed. Of the 8 LOWs, 5 were fixed (S1-5, S1-6, S1-10, S1-11, S1-13) and 3 accepted. There are no new CRITICAL, HIGH or MEDIUM findings.

| Gate criterion | Result |
|---|---|
| Zero CRITICAL residuals | MET. 0 fixable. The unfixed OS CRITICALs (perl-base, plus the zlib1g false positive) are accepted with justification (ADR-028) and excluded by `--ignore-unfixed`. |
| Zero HIGH residuals | MET. 0 fixable on api/worker/migrate/web. The unfixed HIGHs are accepted (ADR-028). |
| Every MEDIUM dispositioned | MET. S1-1, S1-2 and S1-4 are FIXED and verified in code and in the running images. |
| Full test suite passing | Not re-run by this read-only audit. It is the build's responsibility (RB-1 commit claims CI parity). |
| Re-scan introduces no new CRITICAL/HIGH | MET. The trivy gate view is 0 on all images, npm audit is 0, gitleaks/semgrep have 0 after triage, and trivy config is 0. |
| `SECURITY-AUDIT.md` | N/A at `standard` depth. |

Recommended follow-ups (LOW, deferrable with a `Deferred — <reason>` note): S2-2 is a one-line compose and env change worth taking now. S2-3 and S2-1 can go in the next RB.
