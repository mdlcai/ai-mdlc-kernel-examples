# Security Audit — pass 1
Commit audited: 59bbee66f24280418cfbf0f10865bed24b59956c
Tools:
- Dependency vulnerabilities: `npm audit --omit=dev` (npm, lockfile v3) — exit 0, 0 vulnerabilities; `npm audit` (incl. dev) — 1 low (esbuild GHSA-g7r4-m6w7-qqqr, dev server only); Trivy 0.74.0 `fs --scanners vuln,misconfig` on a `git archive` snapshot of HEAD — exit 0, package-lock.json 0 vulnerabilities.
- SAST: Semgrep 1.177.0 (`--config auto`, HEAD snapshot) — exit 0, 57 results, 2 parse errors (non-source files); JSON at `artifacts/security/semgrep-pass1.json`.
- Secrets: Gitleaks 8.30.1 — git history (44 commits) exit 1, 3 hits; HEAD tree exit 1, 3 hits; all are test fixtures (triaged below).
- Config / infra: Trivy 0.74.0 misconfig (Dockerfile: 0 misconfigurations; compose not covered) + manual review of compose.yaml, Dockerfile, .dockerignore, deploy/caddy/Caddyfile, .github/workflows/ci.yml.
- Env var audit: manual (packages/core/src/env.ts vs .env.example vs compose.yaml).
- License: not flagged by RESEARCH.md — skipped.
- Web headers / CSP / cookies / forms: manual review (apps/web/src/proxy.ts, security-headers.ts, apps/api/src/app.ts) plus `artifacts/web-baseline/FRONTEND-AUDIT.nolh.summary.json` (site.headers pass, site.cookies pass).

Summary: CRITICAL 0 · HIGH 0 · MEDIUM 1 · LOW 6

## Scan categories

### Dependency vulnerabilities
Tool: npm audit + Trivy 0.74.0. Result: 0 production vulnerabilities. One dev-only low (esbuild <=0.28.0 dev-server arbitrary file read on Windows, GHSA-g7r4-m6w7-qqqr), reached via the vitest/tsx toolchain; it does not ship in runtime images (`npm ci --omit=dev` in Dockerfile runtime stage). Triage: no finding; upgrade at the next toolchain bump.

### SAST
Tool: Semgrep 1.177.0. Result: 57 results, all triaged as false positives or out-of-scope tooling:
- `node_password` / `node_secret` in web form components (settings-client.tsx:111-112, AcceptInviteForm.tsx:62, LoginForm.tsx:29, RegisterForm.tsx:32) — state variable names, not literals.
- `node_secret` apps/api/src/services/billing.ts:47 — HMAC verification with the configured secret, constant-time `timingSafeEqual`; correct.
- `node_sha1` packages/core/src/auth/password.ts:37 — SHA-1 used only for HIBP k-anonymity breached-password lookup (protocol requirement), not for password storage (argon2id).
- `regex_dos` apps/api/src/app.ts:51 (`ID_RE = /^[A-Za-z0-9._-]{8,128}$/`) and lib/validate.ts:47 (UUID regex) — anchored, linear, bounded.
- `helmet_header_*` INFO on app.ts:45/82 — helmet is configured; informational.
- Remaining hits are in e2e/, load/k6.js, scripts/*.mjs (CI/dev tooling: non-literal RegExp over repo-controlled input, Math.random in load-test data, test credentials read from env) and `.mdlc/BUILD.md` (documentation) — not runtime code.
- `github-actions-mutable-action-tag` ci.yml:41/45/113 — actions pinned to major tags (`@v4`) rather than SHAs; first-party GitHub actions, `permissions: contents: read`; noted under LOW SEC-7.

### Secrets
Tool: Gitleaks 8.30.1. Result: history 3 hits / tree 3 hits, all `generic-api-key` on test fixtures: apps/api/test/team.test.ts (tree L170, L298; history commit fd2cdc85) and apps/api/test/auth.test.ts:61 (a deliberately breached sample password used to assert `BREACHED_PASSWORD`; commit 8ae38a68). Values not reproduced. Triage: false positives. `.env` and `.pki/` are git-ignored (.gitignore:9, :11), `git log --all -- .env .pki` is empty, and both are excluded from the Docker build context (.dockerignore). A working-directory scan's additional hits are confined to git-ignored `.env`, `.pki/` and `node_modules` — not findings.

### Config / infra
Tool: Trivy misconfig (Dockerfile clean) + manual. Result: postgres/redis/prometheus/grafana bind only to 127.0.0.1; api (4000), trust-core (50051), worker and web publish no host ports; only Caddy publishes 80/443, with :80 a 308 redirect to HTTPS, 1 MB body cap, `-Server`/`-X-Powered-By` stripped, `/metrics` 404 at the edge. Caddy's reverse_proxy has no `trusted_proxies`, so client-supplied X-Forwarded-For is replaced by the real peer address — API `trust proxy 'loopback, uniquelocal'` therefore cannot be spoofed from outside. Runtime images run as `USER node`, with HEALTHCHECK and tini. CI: `permissions: contents: read`, no `pull_request_target`, no secrets in workflow. Grafana sign-up disabled, admin password required. Triage: SEC-7 (LOW).

### Env var audit
Manual. Every SPEC §8 secret is `${VAR:?set VAR}`-required in compose, with no defaults. `.env.example` holds placeholders only. The 32-byte base64 keys (PII_*, TRUST_ROOT_KEY) reject the placeholder, and BILLING_WEBHOOK_SECRET's placeholder fails `min(16)`. However, the SESSION_SECRET placeholder passes the `min(32)` length check — see SEC-6. TRUST_ROOT_KEY goes only to trust-core and is deleted from `process.env` after parsing (apps/trust-core/src/main.ts:90).

### License
Not flagged by RESEARCH.md — skipped per BUILD.md.

### Web headers / CSP / cookies / credential forms / static assets
Manual + web-baseline summary (headers pass, cookies pass). CSP Profile N: per-request nonce + `strict-dynamic`, `object-src 'none'`, `frame-ancestors 'none'`. The only `dangerouslySetInnerHTML` (layout.tsx:53) is a nonce'd constant theme bootstrap. HSTS is gated on HSTS_ENABLED (proxy.ts:81, INV-45). API: helmet, Permissions-Policy, `Cache-Control: no-store`, x-powered-by disabled, etag off. Cookies: `__Host-` prefix, Secure, HttpOnly session, SameSite=Strict. All credential forms (login, register, invite accept, settings password, team reset/invite) use `method="post"`. `safeNext` blocks `//` and `/\` open redirects. No finding.

## Findings

### SEC-1 — MEDIUM — Broken access control (agent credential scope bypass)
- Clause violated: ARCHITECTURE §3 ("AI agents … authenticate with a short-lived grant token `cza_…`, TTL ≤ 15 min", scoped by action/key/purpose); SPEC W2.3 ("grant scope for agents"); SPEC §4 least privilege.
- Location: apps/api/src/services/workloads.ts:252-270 (`issueCredential` has no `kind` check); apps/api/src/middleware/authenticate.ts:79-92 (a `czw_` token yields `type: 'workload'` even when `workload.kind === 'agent'`; the computed `kind` field is never consulted anywhere); apps/api/src/services/vault.ts:100-106 (`grantScopeViolation` returns null for any non-`agent` type). The default policy (packages/core/src/policy/index.ts:175-179) allows `workload` principals every `key.encrypt|decrypt|sign|verify`.
- Exploit path: An owner/admin/security_officer registers a workload of kind `agent` (or an attacker gains that role briefly) and calls `POST /v1/workloads/:id/credentials` on it. The API mints a long-lived `czw_` bearer that never expires, is not MFA-gated per grant, and is not bound to actions, key IDs or purpose. The AI agent, or anyone who exfiltrates its environment or prompt context, can then encrypt or decrypt against every key in the tenant that the default `allow-workload-crypto` rule covers. The 15-minute grant TTL, the key allow-list and the purpose binding that are the stated containment for agents are all bypassed. Only object-level purpose matching on vault decrypt remains, and the caller chooses that purpose. Impact is limited to one tenant and needs a privileged insider or a mistake to mint the credential, hence MEDIUM.
- Remediation: In `issueCredential`, after `findWorkload`, add `if (w.kind === 'agent') throw problems.conflict('Agents authenticate with short-lived grants; issue a grant instead.');`. In `bearerPrincipal`, reject `czw_` when `cred.workload.kind === 'agent'` (`throw invalidCredential()`), so credentials minted before the fix stop working. Add a migration or one-off job that revokes existing `workload_credentials` rows whose workload is kind `agent`, plus an integration test asserting 409 on issue and 401 on use.

### SEC-2 — LOW — Account / membership enumeration
- Clause violated: SPEC §7 (uniform errors so identifiers cannot be probed); OWASP ASVS V2.2/V3.
- Location: apps/api/src/services/auth.ts:196 and :229 (register → 409 "account … already exists"); apps/api/src/services/auth.ts:263-271 (login against a locked account → 429 with a lockout-specific retry-after, distinct from the generic 401); apps/api/src/services/team.ts:353 (invite accept → 409 if the email exists in any tenant).
- Exploit path: An unauthenticated attacker posts candidate emails to `/v1/auth/register` and reads 409 vs 201/400 to learn which emails hold accounts on the platform, then targets them for credential stuffing or phishing. Per-IP rate limits slow this but do not prevent it.
- Remediation: Return the same 202 "check your email / account created if eligible" response for register whether or not the email exists, and notify the existing owner out-of-band. On login, return the generic 401 `INVALID_CREDENTIALS` for locked accounts (keep the lockout server-side). On invite accept, return a generic 409 that does not depend on the email.

### SEC-3 — LOW — MFA enrollment without re-authentication
- Clause violated: SPEC §4 (step-up / session hardening); ASVS V2.8 (re-auth before authenticator changes).
- Location: apps/api/src/services/auth.ts:346-358 (`mfaSetup`) and :360 (`mfaEnable`); routes/auth.ts:61.
- Exploit path: An attacker with a stolen session cookie for a user who has not enrolled TOTP calls `/v1/auth/mfa/setup` and `/enable`, binding their own authenticator. They now hold `mfaFresh` (needed for approvals, grants and decrypt step-up), and the legitimate user is locked out of MFA-gated actions until an admin intervenes.
- Remediation: Require `currentPassword` in the `mfaSetup` (or `mfaEnable`) body and verify it with `verifyPassword` behind the login limiter before storing `totpSecretEnc`. Audit `auth.mfa_enable` with the session id.

### SEC-4 — LOW — Vault purpose not bound into the AEAD associated data
- Clause violated: ARCHITECTURE §5 (purpose binding of protected objects); defense in depth for INV on context binding.
- Location: apps/api/src/services/vault.ts:28 (`objectAad` binds tenantId, object id, keyId and classification, but not `purpose`); the purpose check at vault.ts:334 relies on the DB column alone.
- Exploit path: Anyone with write access to the `vault_objects` table (DB credential leak, SQL access from a compromised replica consumer) can change an object's `purpose` to one an agent grant or policy rule allows. Decryption still succeeds because the AAD does not cover purpose, so the purpose control silently drops to a database-integrity control.
- Remediation: Include purpose in the AAD (`cz:obj|${tenantId}|${id}|${keyId}|${classification}|${purpose}`) for new objects with an AAD version byte, and re-wrap or re-seal existing objects in the rewrap job.

### SEC-5 — LOW — System-scope dead-letter entries retryable from any tenant
- Clause violated: ARCHITECTURE §9 tenant-isolation invariant (tenant-scoped operator actions); SPEC §7 ops routes (owner/admin of *their* tenant).
- Location: apps/api/src/services/ops.ts:96-100 (`dlqVisible` returns `'system'` for jobs without `tenantId`) and :194-210 (`retryDlq` re-enqueues them). Billing-webhook jobs are enqueued without tenantId (apps/api/src/services/billing.ts:96).
- Exploit path: An owner/admin of any tenant lists the DLQ, sees global entries (failure reasons are redacted), and retries them. That re-runs platform-wide jobs, such as another tenant's failed billing-webhook processing or global retention, on demand and lets one tenant influence cross-tenant processing timing. Handlers are idempotent, so the impact is limited.
- Remediation: In `listDlq`/`retryDlq`, return 404 for `'system'` scope unless the caller is a platform operator (no such role exists today, so hide them). Alternatively, stamp billing-webhook jobs with the resolved tenantId when known.

### SEC-6 — LOW — Placeholder SESSION_SECRET accepted by env validation
- Clause violated: SPEC §8 ("placeholders only" in .env.example; secrets generated by `npm run dev:secrets`); SPEC §4 fail-closed configuration.
- Location: packages/core/src/env.ts:44 (`SESSION_SECRET: z.string().min(32)`); .env.example:20 (placeholder longer than 32 characters).
- Exploit path: An operator who copies `.env.example` to `.env` without running `dev:secrets` starts a stack whose SESSION_SECRET is the public placeholder. The other keys fail validation, so in practice this needs the operator to have set those by hand. The CSRF-token HMAC (packages/core/src/auth/tokens.ts:17) is then keyed with a public value. The token is still derived from the secret session token, so exploitation also needs session-token disclosure. Low impact, but the system does not fail closed.
- Remediation: Add `.refine(v => !/change[_-]?me/i.test(v) && Buffer.from(v, 'base64').length >= 32, 'SESSION_SECRET must be a generated random value')` and apply the same placeholder denylist to POSTGRES_PASSWORD, REPLICATION_PASSWORD and GRAFANA_ADMIN_PASSWORD in the dev:secrets check / compose preflight.

### SEC-7 — LOW — CI actions pinned to mutable tags
- Clause violated: SPEC §4 supply-chain hygiene (pinned dependencies).
- Location: .github/workflows/ci.yml:41, :45, :113, :129, :147 (`actions/checkout@v4`, `actions/setup-node@v4`, `actions/upload-artifact@v4`).
- Exploit path: If an upstream action tag is moved or compromised, CI runs attacker code with the repository's read token and build environment. The blast radius is small (`contents: read`, no secrets, no `pull_request_target`).
- Remediation: Pin each action to a full commit SHA with a `# v4.x.y` comment and let Dependabot (github-actions ecosystem) bump them.

## Reviewed and clean
- Authentication middleware: deny-by-default with explicit PUBLIC/BEARER allowlists; case and trailing-slash variants fail closed (apps/api/src/middleware/authenticate.ts).
- CSRF: double-submit token plus stored hash on cookie sessions; bearer requests exempt (apps/api/src/middleware/csrf.ts).
- Password storage: argon2id m=19456, t=2, p=1; length 15..128; breached-password check (packages/core/src/auth/password.ts).
- Login: limiter runs before `verifyPassword` (auth.ts:254 → :274, INV-14); lockout 20 per 15 min; dummy verify for unknown users.
- TOTP: ±1 step window, Redis replay key (NX, 120 s), per-user TOTP limiter.
- Sessions: stored hashed; rotated on login, MFA and password change; idle 30 min / absolute 12 h; admin password reset ends all of the member's sessions (team.ts:221) and does not clear TOTP, so takeover still needs the victim's second factor.
- Invites: `czi_` 32 random bytes, SHA-256 stored, 7-day TTL, atomic one-time claim, token redacted from the access log, `no-referrer`, uniform 404.
- Idempotency: keys scoped to (tenant_id, key) with a request-hash mismatch check.
- Vault: 256 KiB strict base64 limit, tenant-scoped `findFirst` → 404, audit written before plaintext release, `no-store` on decrypt.
- Grants: issued only to kind=agent workloads, TTL ≤ 900 s, MFA and role required, PDP evaluated per (action, key) (workloads.ts:341).
- Approvals: requester ≠ decider checked in the service (approvals.ts:96) and enforced by a DB CHECK (`approval_requests_sod_check`, migration.sql:643); decider roles per kind (lib/executors.ts:13-18); step-up MFA; atomic pending→approved claim with expiry; only the original approver may re-run a failed executor.
- Policy bundles: draft → pending_approval uses optimistic concurrency (policies.ts:333); hybrid Ed25519 + ML-DSA-65 signature over tenant|version|digest (anti-replay across tenants); trust-core verifies before activation and on every cache miss, failing closed (service.ts:172-186, 536-551); `defaultEffect` is schema-locked to `deny`.
- Trust-core transport: mTLS with mandatory client certificates (`createSsl(..., true)`, server.ts:67), SPIFFE SAN allow-list checked per RPC (server.ts:29-34), internal errors mapped to a generic "internal error", 8 MB receive cap, root key scrubbed from env and held only in the provider closure.
- Trust-core crypto: AES-256-GCM with random 96-bit nonces and 128-bit tags (packages/core/src/crypto/aead.ts); per-version KEK invocation counter with hard refusal past the limit (service.ts:99-110); DEK AAD binds tenant|key|version|aad; key versions wrapped with AAD `cz:kv|tenant|key|version`; destroy erases wrapped material (crypto-shred); DEKs and secret keys zeroed after use.
- System keys: `__tenant_signing__` cannot be disabled or destroyed (keys.ts:299, :348, :385); names starting with `__` are reserved (routes/keys.ts:29); trust-core `Sign` is called only with server-generated data (policy digest, audit checkpoint, quantum probe from `randomBytes(32)`), so there is no signing oracle over bundle messages.
- Billing: Stripe-Signature HMAC-SHA256 verified before any DB access with constant-time compare and timestamp tolerance (routes/billing-webhook.ts:14, INV-15); replayed event id is a no-op; checkout/cancel/refund are owner-only; `simulate` is refused outside `BILLING_MODE=simulated` (billing.ts:314); money columns have non-negative CHECKs.
- Audit log: append-only via `BEFORE UPDATE OR DELETE` trigger plus REVOKE (migration.sql:650-661).
- Logging: pino redaction covers every SPEC §4 key at top level and `*.` depth, including request headers, set-cookie, stripe-signature and TRUST_ROOT_KEY (packages/core/src/logger.ts).
- Injection sinks: no `$queryRawUnsafe`/`$executeRawUnsafe`, `child_process`, `eval` or `new Function` in app code; raw SQL uses Prisma tagged templates; `Math.random` appears only in load/e2e tooling.
- Ops and role guards: ops routes owner/admin only; key/policy/quantum/apex writers owner/admin/security_officer; evidence export owner/admin/auditor — all match SPEC §7.
- Invite SSR page: no cookies forwarded to the API and server-side GETs only. The default `API_INTERNAL_URL` fallback differs from SPEC, but compose sets it explicitly (informational).
- Edge and network exposure: only Caddy is published; data stores bind to loopback; XFF cannot be spoofed through Caddy (see Config / infra).
- Secrets hygiene: `.env` and `.pki/` are git-ignored, never committed and excluded from the Docker context; `.env.example` contains placeholders only.
