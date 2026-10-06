# Security Audit — Pass 1 (initial scan, read-only)

Audited commit: d700300

Date: 2026-10-05 · Depth: standard · review_gates: auto · Scope: `server/`, `web/`, `scripts/`, Dockerfiles, `compose.yaml`, `Caddyfile`, `.github/workflows`, plus live read-only and auth-negative probes against https://localhost:9443 (compose project `threatwatch-mdlc`).

Raw scanner output is in `artifacts/security/raw/`.

## Tool table

| Tool | Version | Command | Exit | Findings raw → after triage |
|---|---|---|---|---|
| npm audit (all) | npm 10.9.3 / node 22.19.0 | `npm audit --json` | 0 | 0 → 0 |
| npm audit (prod) | npm 10.9.3 | `npm audit --omit=dev --json` | 0 | 0 → 0 |
| gitleaks (history) | zricethezav/gitleaks:v8.28.0 | `gitleaks git /src --redact --report-format json` | 1 | 4 → 0 (test fixtures) |
| gitleaks (working tree) | zricethezav/gitleaks:v8.28.0 | `gitleaks dir /src --redact --report-format json` | 1 | 17 → 0 (gitignored `.env`/build output/test fixtures) |
| trivy config (1st run) | aquasec/trivy:0.67.2 | `trivy config /src` | 1 | timed out walking node_modules (helm analyzer), no result |
| trivy config (rerun) | aquasec/trivy:0.67.2 | `trivy config --timeout 20m --skip-dirs node_modules,web/.next,artifacts /src` | 0 | 0 failures (server/Dockerfile 27 passed, web/Dockerfile 27 passed) → 0 |
| trivy fs (1st run) | aquasec/trivy:0.67.2 | `trivy fs --scanners vuln,secret --skip-files /src/.env /src` | 1 | timed out (context deadline on `.claude/mdlc-cache.json`), no result |
| trivy fs (rerun) | aquasec/trivy:0.67.2 | `trivy fs --timeout 20m --scanners vuln,secret --skip-dirs node_modules,.next,.claude,artifacts,.git --skip-files /src/.env /src` | 0 | 0 vulns (package-lock.json), 0 secrets → 0 |
| trivy image api | aquasec/trivy:0.67.2 | `trivy image --scanners vuln,secret threatwatch-mdlc-api:latest` | 0 | C9 H108 M163 L92 U4, 0 secrets → S1-3, S1-4 |
| trivy image worker | aquasec/trivy:0.67.2 | `trivy image --scanners vuln,secret threatwatch-mdlc-worker:latest` | 0 | C9 H108 M163 L92 U4, 0 secrets → S1-3, S1-4 (same base as api) |
| trivy image web | aquasec/trivy:0.67.2 | `trivy image --scanners vuln,secret threatwatch-mdlc-web:latest` | 0 | C5 H58 M49 L55, 0 secrets → S1-3, S1-4 |
| semgrep | semgrep/semgrep:1.140.0 | `semgrep scan --config p/default --config p/typescript --config p/nodejs --config p/secrets --metrics=off --json` (273 rules, 310 files) | 0 | 29 → 2 (S1-6, S1-13); 2 partial-parse errors, see note |
| license summary | lockfile walk (node) | `package-lock.json` non-dev packages → `raw/license-prod.txt` | 0 | 0 issues |
| Manual review + live probes | — | SAST review of auth, authz, tenant scoping, ingest, SSRF, outboxes, WS, CSP/headers, Docker/compose/Caddy, env | — | S1-1, S1-2, S1-5, S1-7..S1-12, INFO items |

Semgrep note: `web/src/app/(app)/alerts/[id]/AlertDetail.tsx:325` and `web/src/app/(app)/rules/_parts.tsx:402` failed to parse completely (the literal `&CK` in "ATT&CK" JSX text), so those two files were only partly analysed. I reviewed both by hand. They have no `dangerouslySetInnerHTML`, no URL sinks and no `eval`.

## Category coverage

| Category | Tooling | Result |
|---|---|---|
| Dependency vulnerabilities | npm audit, trivy fs, trivy image | App dependencies clean. Container base layers flagged (S1-3, S1-4). |
| SAST | semgrep (4 registry packs) + manual review | S1-1, S1-6, S1-5, S1-7, S1-8, S1-12 |
| Secret detection | gitleaks (git + dir), trivy fs/image secret scanners, semgrep p/secrets | No real secret committed. `.env` is gitignored (`.gitignore:7`) and untracked. `.env.example` holds only placeholders. |
| Config / infrastructure | trivy config (Dockerfiles) + manual review of compose/Caddy/CI | S1-9, S1-10, S1-13. Only Caddy publishes ports (9443/8088). Temporal UI and Mailpit UI are on loopback. Postgres/api/web are not published. |
| Environment variable audit | manual (`server/src/config/env.ts` vs `.env.example`) | Every key in the zod env schema is in `.env.example`. Secrets are validated for length. No defaults for secrets in production. |
| License compliance | RESEARCH.md:765 flags it (Wazuh GPL / Elastic ELv2 as reference only); lockfile walk | Production dependencies are MIT/Apache/ISC/BSD, plus LGPL-3.0 (sharp libvips, dynamically loaded optional Next dependency), EPL-2.0 (elkjs, Prisma Studio, dev tooling path), CC-BY-4.0 (caniuse-lite data) and Unlicense (unionfs). Nothing copyleft is linked into the app. Rules are authored in-repo, not vendored from Wazuh/Elastic. |
| Web headers / CSP / forms | live probes + review of `web/src/proxy.ts`, `next.config.ts`, helmet | Nonce + `strict-dynamic` CSP, `frame-ancestors 'none'`, `object-src 'none'`, nosniff, Referrer-Policy, COOP, Permissions-Policy. HSTS max-age=0 is local by design (ADR-025). Credential forms lack `method="post"` (S1-2). |

## Findings

Severity is after triage. Each finding gives the clause violated, the exploit scenario and the fix.

### S1-1 — MEDIUM — SSRF guard not pinned to the validated IP (DNS rebinding TOCTOU)
- **File:** `server/src/services/deliveries/dispatch.ts:5-7, 51, 61`. `server/src/lib/ssrf.ts:57-61` already returns `address` "so the caller can pin it", but the caller ignores it.
- **Clause:** ARCHITECTURE §7 webhook egress ("re-validates and pins the resolved IP for the request"), SPEC S6 / §9 INV-10 (no egress to private/loopback/link-local/metadata ranges).
- **Scenario:**
  1. An admin with `policies:write` (or a compromised admin account in any tenant) creates a webhook endpoint on a low-TTL rebinding domain.
  2. `assertSafeUrl` resolves it to a public IP and passes.
  3. `fetch(url)` resolves again, gets `10.x`/`127.0.0.1`/`169.254.169.254` and POSTs the signed alert body to an internal HTTPS service.
  4. `lastError` (`Request failed: ECONNREFUSED`/`CERT_*`/timeout) is shown in the notification log. That turns it into a blind SSRF and an internal port-scan oracle that crosses the tenant boundary into platform infrastructure.
- **Fix:**
  - Connect to `safe.address` explicitly. Either use `https.request({ host: safe.address, servername: url.hostname, headers: { host: url.host }, ... })`, or use an undici `Agent({ connect: { lookup: (h, o, cb) => cb(null, safe.address, safe.family) } })` passed as `dispatcher`.
  - Keep `redirect: 'error'` and the per-dispatch re-check.
  - Replace the stored error text with a coarse class (`connect_failed`, `tls_failed`, `timeout`, `http_<status>`).
  - Add an integration test where the resolver returns a public IP for the check and a private IP for the connect. Assert that no connection goes to the private IP.

### S1-2 — MEDIUM — Credential forms have no `method="post"`
- **File:** `web/src/app/(auth)/login/LoginForm.tsx:40`, `web/src/app/(auth)/register/RegisterForm.tsx:47`, `web/src/app/(app)/settings/account/AccountView.tsx:93`.
- **Clause:** BUILD.md Security Audit Gate, web delivery check ("credential forms without `method=\"post\"`"). SPEC S9 (logging hygiene: passwords never logged). ARCH §7 Headers: `form-action 'self'` does not stop a same-origin GET.
- **Scenario:**
  1. The forms are `<form onSubmit=… noValidate>` and the inputs carry `name` attributes (`components/ui.tsx:37` spreads props).
  2. Hydration can fail or be slow: a JS error, a blocked chunk, CSP breakage, a user submitting before hydration, or a JS-disabled client.
  3. The browser then falls back to a native GET submit. Email and password (and current/new password on the account page) end up in the URL, browser history, Caddy's JSON access log (`uri`) and any proxy logs.
  4. A live probe of `GET /login?email=…&password=…` returned 200, so nothing rejects such URLs.
- **Fix:**
  - Add `method="post"` (and `action` set to the current path) to all three forms. `e.preventDefault()` keeps the SPA flow.
  - Optionally, make `web/src/proxy.ts` redirect any request whose query carries `password`/`currentPassword`/`newPassword` to the bare path.
  - Add a component-test assertion that `form.method === 'post'`.

### S1-3 — HIGH — Runtime base images are stale and unpatched (fixable CRITICAL/HIGH OS CVEs)
- **File:** `server/Dockerfile:14-15` (`FROM node:22.19-bookworm-slim`, Debian 12.12, no `apt-get upgrade`), `web/Dockerfile:15-16` (`FROM node:22.19-alpine`, Alpine 3.22.1, no `apk upgrade`).
- **Clause:** BUILD.md Security Audit Gate, dependency vulnerabilities (zero unresolved CRITICAL/HIGH). ARCH §7 OWASP floor (A06 vulnerable components). ARCH §7 Dependencies covers only `npm audit`, not image layers.
- **Evidence:**
  - api/worker have fixes available for libgnutls30 3.7.9-2+deb12u5 → deb12u7 (CRITICAL CVE-2026-33845, CVE-2026-42010; HIGH CVE-2026-33846, CVE-2026-3833, CVE-2026-42009), libpam* → deb12u2 (CVE-2025-6020), libpcre2-8-0 → deb12u1/u2 (4 HIGH), libcap2 → deb12u3 (CVE-2026-4878) and gpgv → deb12u2 (CVE-2025-68973). In total 5 CRITICAL and 51 HIGH have fixed versions.
  - web has fixes available for libssl3/libcrypto3 3.5.1-r0 → 3.5.6-r0+ (CRITICAL CVE-2026-31789; HIGH CVE-2025-15467, CVE-2025-69421, CVE-2026-28387..28390, CVE-2026-45447, CVE-2026-14456), musl → 1.2.5-r12 (CVE-2026-40200) and zlib → 1.3.2-r0 (CVE-2026-22184).
- **Reachability:** limited, which is why this is HIGH rather than CRITICAL.
  - Node's TLS uses its statically bundled OpenSSL, not the system libssl. gnutls is pulled in only by `wget`, which is used solely for loopback HTTP healthchecks.
  - Any code execution or library-parsing path in the container still inherits these libraries, and the images are what ship.
- **Fix:**
  1. Bump both bases to the current Node 22 LTS patch and pin by digest. Add `apt-get upgrade -y` (server) and `apk upgrade --no-cache` (web) in the runtime stage.
  2. Drop `wget` from `server/Dockerfile:15`. Write healthchecks as `node -e "fetch('http://127.0.0.1:4000/api/health').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"` (same for worker :4001 and web :3000). That removes gnutls/wget and their CVEs entirely.
  3. Add a CI step: `trivy image --severity CRITICAL,HIGH --ignore-unfixed --exit-code 1` on the built images.
- **Residual no-fix items (disposition: accept with justification, re-scan each release):**
  - perl-base 5.36 CRITICAL CVE-2026-13221/42496/8376: perl is never invoked by the app, which has no `child_process`.
  - util-linux family HIGHs: mount tooling, not used.
  - ncurses CVE-2025-69720.
  - libssl3 (deb) CVE-2026-84782: Node does not link it.
  - wget CVEs disappear with step 2.
  - Alternatively, move api/worker to `node:22-alpine` or a distroless Node base to shed most of them.

### S1-4 — MEDIUM — Unused npm CLI shipped in runtime images (bundled `tar`, `pacote`, `sigstore`, `minimatch`… CVEs)
- **File:** `server/Dockerfile:14` (base for `api`/`worker`), `web/Dockerfile:15`. Path in the image: `/usr/local/lib/node_modules/npm/**`.
- **Clause:** BUILD.md Security Audit Gate, dependency vulnerabilities. ARCH §7 OWASP floor (A06, A05 least functionality).
- **Evidence:** trivy flags npm-bundled `tar` 6.2.1/7.4.3 (raw CRITICAL CVE-2026-59873 plus 8 HIGH), `brace-expansion`, `glob` CVE-2025-64756, `minimatch`, `picomatch`, `ip-address`, `pacote`, `sigstore` and `http-cache-semantics`. That is 3 raw CRITICAL and 39 raw HIGH per image. The app's own dependency tree is clean (npm audit 0).
- **Triage:** reduced to MEDIUM. `api`/`worker`/`web` run `node …` directly and never execute npm. It is only useful to an attacker who already has code execution in the container. The `migrate` stage does use `npx prisma`.
- **Fix:**
  - In the `api`, `worker` and web `runtime` stages: `RUN rm -rf /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/corepack /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack /opt/yarn*`.
  - Keep npm only in `migrate`, or change `migrate` to `node node_modules/prisma/build/index.js migrate deploy`.

### S1-5 — LOW — Ingest has no per-IP ceiling before the per-integration limiter
- **File:** `server/src/services/ingest/ingest.ts:36` and `server/src/api/middleware/rate-limit.ts` (`globalLimiter` skips `/api/v1/ingest`).
- **Clause:** SPEC S4 (rate limits) and ARCH §7 Rate limits. The per-integration limit holds, but nothing bounds unauthenticated volume per source.
- **Scenario:**
  - Random integration UUIDs each get a fresh 120/min bucket, so one IP can drive unlimited unauthenticated requests. Each one costs a DB lookup plus an HMAC (decoy path).
  - Knowing a real integration id lets an attacker exhaust its quota and drop legitimate events.
- **Fix:** add a per-IP limiter (for example 600/min) in front of the ingest route, before the integration lookup. Keep the per-integration bucket keyed after signature verification for quota.

### S1-6 — LOW — AES-GCM decrypt does not enforce a 16-byte auth tag
- **File:** `server/src/lib/crypto.ts:25-27` (semgrep `gcm-no-tag-length`, ERROR).
- **Clause:** SPEC S8 (secrets at rest, authenticated encryption).
- **Scenario:**
  - `setAuthTag` accepts truncated tags (down to 4 bytes) when `authTagLength` is unset.
  - An attacker with write access to `integrations.secret_enc`/`webhook_endpoints.secret_enc` and a decrypt oracle could forge with about 2^32 attempts, not 2^128.
  - It requires DB write access, hence LOW.
- **Fix:** `createDecipheriv('aes-256-gcm', key, iv, { authTagLength: 16 })`, and reject when `Buffer.from(tagB,'base64url').length !== 16` or the IV is not 12 bytes.

### S1-7 — INFO (reclassified from LOW) — Registration reveals existing emails (409 `EMAIL_TAKEN`)
- **File:** `server/src/services/auth/service.ts:53`.
- **Clause:** SPEC error contract (SPEC.md:623, `EMAIL_TAKEN`). Behaviour matches SPEC.
- **Scenario:** account enumeration, throttled to 5/h/IP by the register limiter.
- **Fix:** none required. `EMAIL_TAKEN` is part of the SPEC error contract (SPEC.md:623). It is spec-sanctioned and throttled. Revisit only if open registration becomes multi-tenant self-service at scale.

### S1-8 — LOW — Login timing differs for existing and unknown users
- **File:** `server/src/services/auth/service.ts:94-107`. Argon2 runs on both paths, but the failure audit row is written only for existing users.
- **Clause:** ARCH §7 Passwords ("Login failures return a generic message"). Timing is a side channel to the same information.
- **Fix:** write the audit/event row (or do equivalent work) on both paths, or record it asynchronously after responding.

### S1-9 — LOW — Temporal container runs as root; Temporal UI and Mailpit UI are unauthenticated (loopback)
- **File:** `compose.yaml` (`temporal` service `user: '0:0'`, UI `127.0.0.1:8233`, Mailpit `127.0.0.1:8026`).
- **Clause:** ARCH §7 OWASP floor (A05 security misconfiguration, least privilege).
- **Scenario:**
  - Any local process or user on the host can read workflow history or emails. Mailpit holds temporary-password emails when `EMAIL_TEMP_PASSWORDS=true`.
  - A container escape from Temporal starts as root.
- **Fix:**
  - Run Temporal as its image's default non-root user, with a named volume chowned for the sqlite file.
  - Document both UIs as dev-only, and leave them out of any production compose profile.

### S1-10 — LOW — Application files in runtime images are writable by the runtime user
- **File:** `server/Dockerfile:22-27`, `web/Dockerfile:24-26` (`COPY --chown=node:node`, then `USER node`).
- **Clause:** ARCH §7 OWASP floor (A05/A08 integrity of deployed code).
- **Scenario:** an RCE as `node` can persist by modifying `dist/`/`node_modules` within the container's lifetime.
- **Fix:** copy as root (drop `--chown`) and keep `USER node`. Optionally set `read_only: true` with a `tmpfs: /tmp` in compose for api/worker/web.

### S1-11 — LOW — SMTP STARTTLS is opportunistic
- **File:** `server/src/services/email/transport.ts:18` (`secure: port===465`, no `requireTLS`).
- **Clause:** ARCH §7 OWASP floor (A02 cryptographic failures: credentials in transit).
- **Scenario:** with a production relay on 587, an on-path attacker strips STARTTLS and reads alert and temporary-password emails.
- **Fix:** `requireTLS: env.NODE_ENV === 'production' && env.SMTP_PORT !== 465` (or an `SMTP_REQUIRE_TLS` env flag defaulting to true in production). Mailpit dev stays plaintext.

### S1-12 — LOW — Temporary password stored in plaintext in the email outbox until drained
- **File:** `server/src/services/team/service.ts` (`createUser`/reset → `pending_emails.payload`).
- **Clause:** SPEC S8 (secrets at rest).
- **Scenario:**
  - A DB read (backup, replica, SQL access) during the drain window, or after a stuck or failed send, exposes valid temporary passwords.
  - Impact is bounded by forced change at first login.
- **Fix:** encrypt the payload field with `encryptSecret` (AAD = email id), or keep the scrub-on-send and also scrub on terminal failure and on expiry. Verify both paths have a test.

### S1-13 — LOW — CI uses mutable action tags
- **File:** `.github/workflows/ci.yml:28,29,70` (`actions/checkout@v5`, `actions/setup-node@v5`). Semgrep `github-actions-mutable-action-tag`.
- **Clause:** ARCH §7 OWASP floor (A08 software and data integrity, CI supply chain).
- **Fix:** pin to full commit SHAs, with the tag in a comment, and let Dependabot/Renovate bump them.

### INFO
- I1:
  - `/api/health` is unauthenticated and reports dependency status, uptime and version.
  - `/api/openapi.json` is public.
  - Acceptable per SPEC. Consider trimming version and dependency detail for unauthenticated callers in production.
- I2: a session with `mustChangePassword` can still open the WebSocket. Payloads are only `{kind,id}`, and every fetch is blocked, so nothing leaks.
- I3: the IPv4-compatible `::a.b.c.d` (`::/96`) form is not in the SSRF blocklist. It is deprecated and not routable on Linux. Add `::/96` for completeness. IPv4-mapped `::ffff:` is handled correctly (verified).
- I4: idempotency keys are scoped per org, not per user (`api/authz/idempotency.ts`). A colliding key from a teammate replays their response within the org. Scope by `(orgId, userId, key)`.
- I5: API responses have no Permissions-Policy. They are JSON only, and the web responses carry one.
- I6: local HSTS is `max-age=0` by design (ADR-025). Production must set a real policy at its trusted origin.
- I7: the Next.js server-actions encryption key and preview keys are generated at build time and baked into the web image (standard Next behaviour). Set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` from a secret for multi-replica or prod builds.
- I8: rate limits are held in memory, single instance, as documented. Use a shared store before scaling out.
- I9: the pinned trivy 0.67.2 is behind the current release (0.75.0). Results use the current vulnerability DB (downloaded 2026-10-05), so coverage is current.

## Dropped false positives

| Source | Item | Reason |
|---|---|---|
| gitleaks git | `server/test/helpers/auth.ts:9` | Test fixture password for an ephemeral test DB, not a credential. |
| gitleaks git | `server/test/integration/auth/auth.test.ts:305`, `:318` | Test fixture password and idempotency key literals. |
| gitleaks git | `web/e2e/helpers.ts:23` | E2E fixture password for seeded local users. |
| gitleaks dir | `.env:8,15,16` | The real secrets live where they belong. The file is gitignored (`.gitignore:7`) and untracked. Not copied here. |
| gitleaks dir | `artifacts/e2e/owner.json` | Gitignored local e2e credential cache, not in git. |
| gitleaks dir | `web/.next/**` preview/encryption keys (several) | Gitignored build output, regenerated per build (see I7). |
| gitleaks dir | the same test fixtures as above | Same reason. |
| semgrep | `detect-non-literal-regexp` ×24 (`scripts/govern.mjs`, `scripts/invariant-lint.mjs`, `scripts/web-baseline.mjs`, `web/e2e/*.spec.ts`) | Dev and CI tooling over repo-controlled input (globs, route ids, test names). No untrusted input reaches the regexes and nothing ships in a runtime image. |
| semgrep | `unsafe-formatstring` `scripts/send-sample-events.mjs:37` | Dev CLI that logs its own constants. No attacker-controlled format string. |
| trivy image | zlib1g CVE-2023-45853 (CRITICAL, api/worker) | Affects minizip only, which Debian's `zlib1g` does not ship. Debian marks it not-affected. |
| trivy config | first run exit 1 | Timeout walking `node_modules` (helm analyzer). The rerun with `--skip-dirs` passed with 0 failures. |
| trivy fs | first run exit 1 | Timeout on `.claude/mdlc-cache.json`. Rerun with exclusions (see table). |
| live probe | `localhost:4000` / `localhost:3000` answered | Unrelated host processes (PIDs 19104/5636, not Docker). `docker ps` confirms threatwatch api/web publish no ports. |

## Live probe results (all as expected)
- Unauthenticated `/api/v1/alerts` → 401. A forged session cookie → 401.
- Login with no `Origin` or a foreign `Origin` → 403 `CSRF_ORIGIN`. A `text/plain` POST → 415.
- Bad credentials → 401 with a generic message. RateLimit headers present (300/min global, 10/min IP, 5/min email).
- Ingest with a bad signature → 401 `INGEST_UNAUTHORIZED`. Malformed JSON → 400 `VALIDATION_FAILED`. Unknown route → 404 problem+json.
- WebSocket upgrade with a foreign Origin → 403. `http://` → 301 to https.
- `/` and `/api/health` carry CSP (nonce, strict-dynamic), nosniff, Referrer-Policy, COOP, X-Frame-Options DENY, Permissions-Policy (web) and HSTS max-age=0 (ADR-025).

## Verdict

**FAIL — gate not met at pass 1.**

| Severity | Count | IDs |
|---|---|---|
| Critical | 0 | — |
| High | 1 | S1-3 |
| Medium | 3 | S1-1, S1-2, S1-4 |
| Low | 8 | S1-5, S1-6, S1-8 … S1-13 |
| Info | 10 | S1-7, I1 … I9 |

Gate criteria: zero unresolved CRITICAL/HIGH, and every MEDIUM dispositioned. The gate is blocked by S1-3 (HIGH) and three open MEDIUMs. All four have concrete code or Dockerfile fixes and can be auto-remediated in pass 2:
- pin the egress IP and genericise the error (S1-1)
- add `method="post"` (S1-2)
- refresh and upgrade the bases, drop wget, add a CI trivy gate (S1-3)
- strip npm from the runtime stages (S1-4)

The no-fix residual OS CVEs under S1-3 need an explicit accept-with-justification disposition. The LOWs are recommended. S1-6 and S1-11 are one-line fixes and worth taking in the same pass.

Tooling coverage: every category had automated tooling (npm audit, gitleaks, trivy config/fs/image, semgrep, lockfile license walk). No ADVISORY-ONLY degradation applies, apart from the semgrep partial-parse gap on 2 files, which was covered by manual review.
