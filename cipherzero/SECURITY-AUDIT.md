# SECURITY-AUDIT

Security Audit Gate for project c16fd425 (CipherZer0) at `build_depth: comprehensive`. Three passes ran; their raw reports are in `artifacts/security/`.

| Pass | Commit | Report | Result |
|---|---|---|---|
| 1 | 59bbee6 | artifacts/security/pass-1.md (+ semgrep-pass1.json) | CRITICAL 0 · HIGH 0 · MEDIUM 1 · LOW 6 |
| 2 | 32b23e1 | artifacts/security/pass-2.md (+ semgrep-pass2.json) | SEC-1/3/6 fixed; new SEC-8 (M), SEC-9 (M), SEC-10 (L) |
| 3 | af52231 | artifacts/security/pass-3.md (+ semgrep-pass3.json) | SEC-8/9/10 fixed; new SEC-11 (L) → fixed in the wide-close batch |
| 4 | wide-close diff on e8666d6 | artifacts/security/pass-4.md | SEC-11 verified; new SEC-12 (L) → fixed; new SEC-13 (L) → deferred |
| R-1 | re-entry delta a35c750..9d03cc6 (F-18, F-19, F-20) | this file, "Pass 4 (re-entry R-1 delta)" below | CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 2 (SEC-14 fixed, SEC-15 deferred) · INFO 2 |
| R-2 | re-entry delta b0672a0..HEAD (F-21–F-24 + ops fixes) | this file, "Pass 5 (re-entry R-2 delta)" below | CRITICAL 0 · HIGH 1 dependency (body-parser, fixed) · MEDIUM 1 (SEC-18, fixed) · LOW 6 (SEC-19, 21, 22, 24 fixed; SEC-20, 23 deferred) · INFO 3 (SEC-25, 26 fixed; SEC-27 noted) |
| R-3 | re-entry delta 6ec8983..393294e (F-25–F-28), fix batches 80fc26b (ADR-094) and 0.5.2 (ADR-095) | this file, "Pass 6 (re-entry R-3 delta)" below; artifacts/security/pass-6.md (rounds 1 and 2, + semgrep-pass6.json) | CRITICAL 0 · HIGH 0 · MEDIUM 3 (SEC-28, 29, 30 fixed) · LOW 4 (SEC-37 fixed; SEC-31, 32, 33 deferred) · INFO 5 (SEC-38, 39 fixed; SEC-34, 35, 36 noted) |

**Verdict: PASS WITH ADVISORY.** Residual: CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 11 deferred (SEC-2, SEC-4, SEC-5, SEC-7, SEC-13, SEC-15, SEC-20, SEC-23, SEC-31, SEC-32, SEC-33) plus the SEC-6 compose residual. The re-entry R-1 delta (F-18, F-19, F-20) passes with no CRITICAL, HIGH or MEDIUM findings. The re-entry R-2 delta (F-21–F-24) had one MEDIUM (SEC-18) and one HIGH dependency advisory, both fixed and regression-tested before commit. The re-entry R-3 delta (F-25–F-28) had three MEDIUM findings in the prober (SEC-28, SEC-29, SEC-30), fixed in 80fc26b (ADR-094). Round 2 confirmed SEC-28 and SEC-29 and found SEC-30 only bounded per request; the whole-run budget and worker slot limit closed it in 0.5.2 (ADR-095), with SEC-37, SEC-38 and SEC-39 from the same round.

Banner values: Pass 1 findings 7 (M1 · L6) · Auto-remediated 8 (SEC-1, SEC-3, SEC-6, SEC-8, SEC-9, SEC-10, SEC-11, SEC-12) · Pass 2 residual CRITICAL 0 · HIGH 0 · MEDIUM 0 · LOW 5 (+ SEC-6 compose residual).

## Trust boundaries

1. **Internet → edge (Caddy).** Only Caddy publishes host ports (80 → 308 to 443, 443). It caps bodies at 1 MB, strips `Server`/`X-Powered-By`, and returns 404 for `/metrics`. Caddy has no `trusted_proxies`, so a client-supplied `X-Forwarded-For` is replaced by the real peer. The API trusts only `loopback, uniquelocal` proxies, so client IPs (and the per-IP rate-limit buckets keyed on them) cannot be spoofed from outside.
2. **Edge → web (Next.js) and API (Express).** The web tier holds no secrets beyond the session cookie it forwards. Server components read the session and the dashboard summary through `serverGet` (apps/web/src/lib/server-api.ts), which forwards only the caller's own cookie to the internal API, uses `cache: 'no-store'`, and fails to `null`. React `cache()` scopes the session read to one request, and the zustand store's server snapshot is its initial state, so nothing is shared across requests (ADR-071).
3. **API → trust-core (gRPC).** This is the cryptographic boundary. mTLS requires client certificates, every RPC checks a SPIFFE SAN allow-list, internal errors map to a generic message, and receive size is capped at 8 MB. `TRUST_ROOT_KEY` reaches only trust-core and is deleted from `process.env` after parsing. The API never holds a KEK or an unwrapped DEK beyond the operation that needs it.
4. **API/worker → Postgres and Redis.** Both bind only to 127.0.0.1. Every query is tenant-scoped (`tenantId` in the `where`), and cross-tenant reads return 404. The audit log is append-only through a `BEFORE UPDATE OR DELETE` trigger plus REVOKE.
5. **Human principals vs machine principals.** Humans use cookie sessions (`__Host-`, Secure, HttpOnly, SameSite=Strict, CSRF double-submit). Service workloads use `czw_` bearers. AI agents use only short-lived `cza_` grants (TTL ≤ 900 s, action/key/purpose scoped). Since SEC-1, an agent-kind workload can neither be issued nor use a `czw_` credential.
6. **Billing provider → webhook.** The Stripe-Signature HMAC is verified with a constant-time compare and timestamp tolerance before any DB access (INV-15). Replayed event IDs are no-ops.
7. **CI → repository.** The workflow token is `contents: read`, there is no `pull_request_target`, and there are no secrets. Actions are pinned to mutable tags (SEC-7, deferred).

## Data-flow risks

- **Plaintext through the vault.** Plaintext enters on protect and leaves on decrypt. It is size-limited (256 KiB strict base64), audited before release, sent with `Cache-Control: no-store`, and zeroed after use. Each object's AAD binds tenant, object id, key id and classification. Purpose is checked from the DB column only (SEC-4, deferred), so purpose binding degrades to a database-integrity control for anyone with DB write access.
- **Key migration (quantum, W5).** Migration re-seals data under a successor key and disables the source. SEC-9 showed that this silently stopped key-scoped policy rules and grants from applying. The `keyScopeBlocker` (apps/api/src/services/quantum.ts) now refuses to move objects off a key that any active rule or live grant names unless the successor is also named, and the plan cannot validate until then. Pass 3 tried six bypasses (empty `keyIds`, pending bundles, a pre-created successor, the retry path, signing keys, TOCTOU) and none worked. One informational gap remains: a key-A rule pending at migration time and activated afterwards does not cover A-pq. That activation needs two-person approval, and the approver sees the key IDs.
- **Policy bundles.** Bundles are signed with hybrid Ed25519 + ML-DSA-65 over tenant|version|digest, which prevents replay across tenants. Trust-core verifies the signature before activation and on every cache miss, failing closed. `defaultEffect` is schema-locked to `deny`.
- **Logs.** Pino redaction covers every SPEC §4 key at top level and at `*.` depth: request headers, set-cookie, stripe-signature and `TRUST_ROOT_KEY`. Invite tokens are redacted from the access log. SEC-8 and SEC-11 closed the two places where a credential could reach a URL (and so a log) before hydration.
- **Dead-letter queue.** System-scope jobs (with no tenantId) are visible to and retryable by any tenant owner or admin. Failure reasons are hidden and the handlers are idempotent, so the impact is timing only (SEC-5, deferred).

## Authentication and authorization edge cases

- **Deny by default.** The authenticate middleware uses explicit PUBLIC and BEARER allowlists. Case and trailing-slash variants fail closed.
- **Login.** The limiter runs before `verifyPassword` (INV-14). Accounts lock after 20 failures in 15 minutes. Unknown users get a dummy argon2id verify, so timing does not reveal them. Locked accounts get a distinct 429, and register and invite-accept return a 409 for existing emails, so account existence can be enumerated (SEC-2, deferred: fixing it needs a SPEC §7 contract change).
- **Re-authentication.** MFA setup and password change require the current password (SEC-3). They are throttled per IP+email and per user, at 10 per 15 minutes (SEC-10), so guesses spread across many IPs are capped per account.
- **Step-up.** Approvals, grants and decrypt require `mfaFresh`. The TOTP window is ±1 step with a Redis NX replay key (120 s) and a per-user limiter. The step-up form now posts (SEC-11).
- **Separation of duties.** The approval requester can never be the decider; the service checks this and the `approval_requests_sod_check` DB CHECK enforces it. Decider roles are fixed per approval kind. A pending request moves to approved through an atomic claim with expiry, and only the original approver may re-run a failed executor.
- **Sessions.** Session tokens are stored hashed and rotated on login, MFA and password change. Sessions expire after 30 minutes idle or 12 hours absolute. An admin password reset ends all of the member's sessions but keeps their TOTP, so a takeover still needs the second factor.
- **System keys.** `__tenant_signing__` cannot be disabled or destroyed, and names starting with `__` are reserved. Trust-core signs only server-generated data, so it is not a signing oracle.
- **Agents.** Grants are issued only to agent-kind workloads, require MFA and an authorized role, and are checked against the PDP per (action, key). Agent use outside the declared purpose is denied and recorded, and APEX surfaces it as a finding. Remediation still requires human approval (W6).

## Findings ledger

| ID | Severity | Title | Status |
|---|---|---|---|
| SEC-1 | MEDIUM | Agent workloads could mint long-lived `czw_` credentials | FIXED (pass 2): 409 on issue, 401 on use |
| SEC-2 | LOW | Account and membership enumeration through register, login and invite | DEFERRED |
| SEC-3 | LOW | MFA enrollment without re-authentication | FIXED (pass 2) |
| SEC-4 | LOW | Vault purpose not bound into the AEAD AAD | DEFERRED |
| SEC-5 | LOW | System-scope DLQ entries retryable from any tenant | DEFERRED |
| SEC-6 | LOW | Placeholder SESSION_SECRET accepted | FIXED (pass 2); compose `change-me` residual deferred |
| SEC-7 | LOW | CI actions pinned to mutable tags | DEFERRED |
| SEC-8 | MEDIUM | MFA re-auth password form without `method="post"` | FIXED (pass 3) |
| SEC-9 | MEDIUM | Quantum migration stopped key-scoped policy rules and grants from applying | FIXED (pass 3), ADR-070 |
| SEC-10 | LOW | Re-auth throttled per IP only | FIXED (pass 3) |
| SEC-11 | LOW | TOTP step-up form without `method="post"` | FIXED (wide close, ADR-071) |
| SEC-12 | LOW | Soft navigation at sign-in/sign-out kept the previous user's server-read pages in the router cache (Back showed them) | FIXED (wide close, ADR-071): full page load at every auth boundary |
| SEC-13 | LOW | Server-side session/summary reads share the web container's per-IP rate-limit bucket | DEFERRED |

### Pass 4 (re-entry R-1 delta: F-18, F-19, F-20)

Scope: `git diff --ignore-cr-at-eol a35c750..9d03cc6`, excluding FRONTEND-AUDIT*.json and GOVERNANCE-DECISIONS.jsonl. Read-only review; no code was changed.

| ID | Severity | File:line | Title | Status |
|---|---|---|---|---|
| SEC-14 | LOW | apps/api/src/middleware/authenticate.ts:106 | `czw_` principal kind is mapped `observer ? 'observer' : 'service'`, so any workload kind added later would fail open to the service routes (vault). `agent` is refused explicitly at :95, so nothing is exploitable today | FIXED (authenticate.ts: only `service`/`observer` kinds hold a czw_ bearer; any other kind gets 401) |
| SEC-15 | LOW | apps/api/src/routes/observe.ts:25 | Each snapshot call runs `probeDependencies` (DB, replica, Redis, trust-core gRPC health and worker heartbeat, 800 ms each) | DEFERRED |
| SEC-16 | INFO | apps/api/src/services/dashboard.ts:178, :188, :209-214 | The summary now gives every session role finding `detail`, key names/purposes, the active-session count and the newest 6 audit events (opaque `user:<id>` actors and `<type>:<id>` targets, no `data`). Every session role can already read all of these through `/apex/findings`, `/keys` and `/evidence/events`, so there is no escalation. Tenant-scoped, replica reads, bounded `take` | NOTED |
| SEC-17 | INFO | demo/cipherzer0-demo.html:13, :2715 | Static demo (ADR-079): two inline scripts, no external script, style or font loads (fonts and icon are data URIs), no network calls, no CSP meta (it is a file, not served by the app). Data is generated for a fictional `*.example` tenant with no tokens or keys. The hash router never reflects the URL fragment: unknown ids fall to a constant 404, and free text goes through `esc()`. Builder escapes `</script` | NOTED |

Checked with no finding:
- **Observer authz (F-20).** Bearer kinds are disjoint (authenticate.ts BEARER table). An observer reaches only `GET /v1/observe/{snapshot,events}`, and service and agent bearers and sessions get 403 (`observer()` in lib/http.ts runs before validation). Grants are still refused to non-agent workloads (workloads.ts:346), and the kind cannot be changed after creation. The tests cover cross-tenant isolation, vault denial and replica-only reads (apps/api/test/observe.test.ts).
- **Observe data exposure.** Snapshot and events carry ids, names, statuses, the key purpose and profile, the audit hash and prevHash, and the opaque actor and target. They never carry key material, ciphertext, audit `data` or emails. System keys appear by name only.
- **Input and DoS.** `after` is a ≤19-digit decimal within int64. `limit` is an integer from 1 to 100, and the query object is strict. Snapshot lists are capped at 500 (+1 to detect truncation, reported in `truncated`). The tenant rate-limit bucket applies to observer bearers. Dashboard `range` is an enum (`30d` = 30 day buckets).
- **Headers.** Observe responses set `Cache-Control: private, no-store`. F-18 changed no CSP, header or cookie code (proxy.ts untouched). The new web links are internal routes built from server ids, and nothing uses `dangerouslySetInnerHTML`.
- **Dependencies.** No package-lock or workspace manifest changed. The root package.json adds only the `demo:build` script.

Deferred, with reasons:
- **SEC-14.** It is not reachable (there are only three kinds, `agent` is refused before the mapping, and kinds are a zod enum at creation). Fix: an exhaustive switch that throws `invalidCredential()` for an unknown kind.
- **SEC-15.** Each probe is bounded at 800 ms, the tenant bucket applies, and the unauthenticated `/api/health` already runs the same probes, so the observer endpoint adds no new amplification. Fix: memoize the probe for about 5 s.

### Pass 5 (re-entry R-2 delta: F-21, F-22, F-23, F-24 and the ops fixes)

Scope: `git diff --ignore-cr-at-eol b0672a0..HEAD` plus the fix batch, excluding FRONTEND-AUDIT*.json and GOVERNANCE-DECISIONS.jsonl (114 files, about 5.7k lines). A read-only reviewer pass ran first, then fixes, each with a regression test that fails on the old code where a failure can be shown.

| ID | Severity | File:line | Title | Status |
|---|---|---|---|---|
| SEC-18 | MEDIUM | apps/api/src/lib/mode.ts | The `PRODUCT_MODE=readiness` gate matched paths case-sensitively, but Express routes case-insensitively, so `/api/v1/VAULT/objects` or `/api/v1/Keys/:id/ROTATE` reached the vault and key routers in readiness mode | FIXED: the path is lower-cased before matching; apps/api/test/modes.test.ts "case variants" (400 → 404 on the old code) |
| SEC-19 | LOW | apps/api/src/lib/secret-guard.ts, apps/scanner/src/guard.ts | The `long_base64` rule (513+ chars) could never fire because every ingest field is capped at 512. A bare SEC1/PKCS#8 DER key in base64, a PEM body without its BEGIN line, a percent-encoded PEM marker and PGP or PuTTY private keys all passed | FIXED: a base64/base64url run of 96+ characters with ≥ 5.0 bits/char Shannon entropy is refused (random base64 of that length never scores below about 5.1; long paths and hostnames score near 4); PGP `PRIVATE KEY BLOCK` and `PuTTY-User-Key-File-` markers; percent-encoded strings are also checked decoded, one escape run at a time so a malformed `%ZZ` cannot block decoding. Both copies share test/fixtures/secret-material.json (16 reject, 12 accept cases). Residual: a raw 64-hex key is indistinguishable from a SHA-256 fingerprint |
| SEC-20 | LOW | apps/api/src/services/quantum-ingest.ts | A scanner-declared `scope` lets a report validate a migration plan for a locator it never re-observed | DEFERRED |
| SEC-21 | LOW | apps/scanner/src/deps.ts | Package names were looked up in a plain object, so `constructor`, `toString` or `__proto__` hit `Object.prototype` | FIXED: `Object.hasOwn`; apps/scanner/test/deps.test.ts |
| SEC-22 | LOW | apps/scanner/src/certs.ts | The lazy `BEGIN…END CERTIFICATE` regex was quadratic on a planted file of BEGIN lines with no END (27.7 s on a 1 MB file) | FIXED: linear indexOf scan capped at 1000 blocks per file; apps/scanner/test/certs.test.ts (< 1 s) |
| SEC-23 | LOW | apps/scanner/src/certs.ts | A combined certificate+key file whose name does not look like a key (`server.pem`) is read into scanner memory. Only `CERTIFICATE` blocks are parsed and only their metadata leaves the function | DEFERRED |
| SEC-24 | LOW | apps/scanner/src/upload.ts | `*.localhost` names counted as loopback for the plain-HTTP exception, although a resolver may not map them to 127.0.0.1 | FIXED: only `localhost`, `127.0.0.0/8` and `::1`; apps/scanner/test/payload.test.ts |
| SEC-25 | INFO | apps/api/src/routes/quantum-ingest.ts | The scanner-principal check ran inside the handler, after validation, the secret guard and the idempotency claim, so a session learnt 400/422 verdicts before the 403 | FIXED: `scannerOnly` runs first; apps/api/test/quantum-ingest.test.ts |
| SEC-26 | INFO | apps/api/src/routes/quantum-ingest.ts, apps/scanner/src/schema.ts | Reported strings could carry C0/C1 control characters and bidi overrides (terminal escape or console spoofing in `--dry-run` output and lists) | FIXED: refused with 400 (tab, CR, LF stay so a multi-line PEM still reaches the secret guard and gets 422); apps/api/test/quantum-ingest.test.ts |
| SEC-27 | INFO | apps/api/src/services/quantum-ingest.ts | The set of known locators is read before the per-tenant assets lock, so two concurrent reports can both see an asset as new. The plan limit is enforced under the lock, so the effect is a count off by the race, never a limit bypass | NOTED |

Checked with no finding:
- **Scanner bearer (F-23).** `czw_` scanner tokens map to their own principal kind. A scanner reaches only `POST /v1/quantum/ingest` and gets 403 on assets, vault, observe and keys. Sessions, service and observer bearers get 403 on ingest (apps/api/test/quantum-ingest.test.ts).
- **Entitlements (F-22).** Limits are read inside the write transaction and enforced server-side (403 `PLAN_LIMIT`). A downgrade keeps existing data and refuses new writes over the limit. Money stays integer minor units (INV-47).
- **CBOM and evidence.** The CBOM export is tenant-scoped, needs a session, carries algorithms and locators only, and is audited as `quantum.cbom.exported`.
- **Raw SQL (ops fixes).** The job_runs index probes are `$queryRaw` tagged templates with bound parameters, no string interpolation.
- **Scanner CLI.** Key-named files are never opened. Symlinks are not followed. File size, depth and file-count caps apply. The token is read from `CZ_SCANNER_TOKEN` only, never argv, and is not printed. Uploads require https except to loopback. The report is validated and secret-guarded before printing or sending (exit 3).
- **Web (F-21–F-23).** No `dangerouslySetInnerHTML`. Scanned locators render as text. No CSP, header or cookie change.

Deferred, with reasons:
- **SEC-20.** Whoever holds a scanner credential can already report any findings, so a report is the tenant's own attestation, not independent evidence. Declared scope does not widen that. The plan validation evidence names the scan id and scanner, and COMPLIANCE QR-PLAN-EVIDENCE records the trust limit. Fix: require each scoped locator to be re-observed in the same report.
- **SEC-23.** The private key is never parsed, stored or sent, and the file is on the customer's own host, read by their own process. Skipping such files would hide real certificates. Fix: refuse any file whose text holds a `PRIVATE KEY` marker and list it as skipped.

### Pass 6 (re-entry R-3 delta: F-25, F-26, F-27, F-28)

Scope: `git diff 6ec8983..393294e` (the isolated prober, monitored domains, external discovery, the Domains console), then round 2 on the fix batch `393294e..80fc26b`. Tools: `npm audit --omit=dev` 0 vulnerabilities (8 dev-only in the full audit); Semgrep 1.177.0, 9 results, all classes triaged in earlier passes (26 rule timeouts, those files read by hand); Gitleaks 8.30.1, nothing in 29 commits; invariant lint 59/59 machine checks. Trivy 0.74.0 aborted on a Docker Desktop bind-mount limit ("readdirent: cannot allocate memory"); compose and Dockerfile misconfiguration were reviewed by hand instead.

| ID | Severity | File:line | Title | Status |
|---|---|---|---|---|
| SEC-28 | MEDIUM | compose.yaml (prober), deploy/pki/bootstrap.mjs | The internet-facing prober mounted the whole `pki` volume, owned by the uid it runs as, so code execution in it could read `ca.key` and every service's key and mint any SPIFFE identity | FIXED (80fc26b, ADR-094): `bootstrap.mjs --export <svc>:<dir>` copies only ca.crt and the service's own certificate and key; each service mounts only its `pki-<svc>` volume read-only; the CA key stays in `pki`, mounted by pki-init alone. apps/prober/test/pki-mount.test.ts |
| SEC-29 | MEDIUM | apps/scanner/src/tls.ts | A peer certificate with a notAfter that does not parse threw RangeError inside the `secureConnect` listener; with no process handler, one hostile endpoint could crash the prober shared by every tenant | FIXED: `notAfterOf` drops a date that does not parse (also for local files in certs.ts); the listener body runs in try/catch that rejects only that probe; the socket keeps an `error` listener for its whole life; the prober logs uncaught exceptions and unhandled rejections and keeps serving. apps/scanner/test/tls.test.ts "hostile peers" |
| SEC-30 | MEDIUM | apps/scanner/src/tls.ts, apps/prober/src/routes.ts, apps/worker/src/prober-client.ts, apps/worker/src/jobs/external-discovery.ts, apps/worker/src/scheduler.ts | The handshake timeout was an idle timeout, so a slow-drip peer could hold the prober's 8 slots; the resulting 503 BUSY set the global PROBER_DOWN_KEY and failed verify and scan for every tenant. Round 2: nothing bounded a whole run, so 1,000 slow hosts per domain kept one run going about 7 h and four runs held every worker slot | FIXED: an absolute handshake deadline; one request starts no handshake after `PROBE_BUDGET_PORTS` (4) timeouts; 503 BUSY maps to `ProberBusyError`, which retries without setting PROBER_DOWN_KEY (80fc26b). Then a 20-minute run budget with longest-ago-first hosts and domains, and at most 2 external-discovery jobs per worker, delayed without spending an attempt (0.5.2, ADR-095). apps/scanner/test/tls.test.ts (drip), apps/prober/test/routes.test.ts (budget), apps/worker/test/domain-verify.test.ts (busy), apps/worker/test/external-discovery.test.ts (run budget), apps/worker/test/slot-limit.test.ts |
| SEC-31 | LOW | apps/worker/src/jobs/external-discovery.ts | Prober output reaches ingest with only a shape check; locators are not bound to the probed host and port | DEFERRED |
| SEC-32 | LOW | apps/worker/src/jobs/domain-verify.ts | A SERVFAIL or timeout on the challenge name never counts as a miss (ADR-091), so a domain whose DNS fails persistently never lapses | DEFERRED (COMPLIANCE EXT-VERIFIED-ONLY ⚠) |
| SEC-33 | LOW | apps/prober/src/ct.ts | A CT response is buffered up to 10 MB, which several concurrent searches could push past the prober's 256 MB memory limit | DEFERRED |
| SEC-34 | INFO | apps/prober/src/ct.ts | The CT fetch re-resolves the CT host after the egress guard checked it | NOTED |
| SEC-35 | INFO | apps/worker/src/jobs/external-discovery.ts | The run lock is deleted unconditionally, so a run that outlived its lock can release a newer run's lock | NOTED |
| SEC-37 | LOW | apps/prober/src/main.ts | (Round 2, introduced by 80fc26b; reviewer round 8 N1.) The log-and-continue `uncaughtException` handler also swallowed a `listen` failure, leaving a prober that listens nowhere and is never restarted | FIXED (0.5.2): `server.on('error')` logs and exits |
| SEC-38 | INFO | deploy/pki/bootstrap.mjs | (Round 2.) `--export` prunes its directory, so one that overlaps `--out` would delete the CA key | FIXED (0.5.2): refused before anything is issued; apps/prober/test/pki-mount.test.ts |
| SEC-39 | INFO | apps/prober/src/env.ts | (Round 2.) `PROBER_TIMEOUT_MS` up to 60 s let one request outlast the worker's fixed 120 s prober timeout and set PROBER_DOWN_KEY | FIXED (0.5.2): capped at 15 s |
| SEC-36 | INFO | packages/core/src/quantum/domain-name.ts | A domain longer than 231 characters cannot carry the challenge label, so it can never verify | NOTED |

Deferred, with reasons:
- **SEC-31.** The prober is ours, reached over mTLS with a pinned SPIFFE id, and every finding still passes the ingest schema and the secret guard (INV-57). A compromised prober could forge findings for the tenant's own domains only. Fix: drop findings whose locator is not `tls://<probed host>:<probed port>`.
- **SEC-32.** Counting resolver failures as misses would lapse healthy domains during a resolver outage, which ADR-091 rejected. Fix: lapse after N days of consecutive lookup failures, separately from misses.
- **SEC-33.** The prober's memory limit contains the effect to a prober restart; no data or tenant boundary is crossed. Fix: stream the CT body and stop at a name count.

### Deferred, with reasons
- **SEC-2.** The fix (a uniform 202 on register, a generic 401 when locked) changes the SPEC §7 API contract. Per-IP limits apply, and only account existence leaks, never credentials.
- **SEC-4.** The fix needs an AAD version byte and a re-seal of every existing object, an architectural change that needs its own ADR. Exploiting it requires DB write access.
- **SEC-5.** Retries only re-run idempotent, claim-guarded jobs early, and no data crosses tenants. Hiding these entries properly needs a platform-operator role that SPEC does not define.
- **SEC-7.** All actions are first-party, the token is `contents: read`, and there are no secrets and no `pull_request_target`. Pin actions to SHAs and images to digests, with Dependabot for github-actions.
- **SEC-13.** Draining the shared bucket only makes server reads fail, and the console falls back to the client fetch, which is limited per real client IP. First paint is slower; there is no auth bypass and no data exposure. The fix is a trusted internal channel (mTLS or a shared-secret header that the API maps to its own bucket), an API contract change for a later ADR.
- **SEC-6 compose residual.** The Postgres, replica and Grafana `change-me` defaults bind only to loopback, and `npm run dev:secrets` generates real values. Add the placeholder denylist to the compose preflight later.

## Tool results (pass 3, af52231)
- **Re-entry R-2: `npm audit --omit=dev`** first reported body-parser 2.3.0 (HIGH) and fast-uri 3.1.8 (moderate), both transitive. `npm update body-parser fast-uri` moved them to 2.4.0 and 3.1.9 inside the existing ranges, and the audit now reports `found 0 vulnerabilities`.
- **Re-entry R-2: Gitleaks v8.30.1.** `git --log-opts=b0672a0..HEAD` reports no leaks (exit 0). A `dir` scan of the uncommitted fix batch reports 5 `private-key` hits, all in test/fixtures/secret-material.json: the 3 deliberate F-23 reject fixtures already present at HEAD, plus the new PGP and PuTTY marker cases. These are test inputs the guard must refuse, triaged as false positives (add them to `.gitleaksignore` with the CI advisory below). A whole-directory scan was abandoned because it walks untracked `node_modules`, `.next` and the local `.pki` (never committed).
- **Re-entry R-2: Semgrep 1.177.0** (`p/default p/typescript p/nodejsscan`, the 81 TypeScript and script files changed since b0672a0). Exit 0, 15 results, triaged as follows. The helmet INFO and `node_password`/`node_secret` results are test fixture names, the same classes as pass 3. The `regex_dos` results (app.ts:53, scanner tls.ts:55, core quantum/risk.ts:42–43) are fixed anchored patterns with no nested quantifiers. `bypass-tls-verification` (scanner tls.ts:52) is deliberate: the probe records the handshake metadata of internal and self-signed endpoints and sends nothing, and uploads go through a separate certificate-verified HTTPS client. `contains-bidirectional-characters` (quantum-ingest.test.ts) was real: the SEC-26 test held literal bidi and control bytes. It is fixed with `String.fromCodePoint`, and a repo grep finds no literal bidi or control characters in apps, packages, scripts or test.
- **Re-entry R-2: Trivy 0.74.0** (`fs --scanners vuln,misconfig --severity HIGH,CRITICAL` over the tracked manifests, the updated package-lock.json, the Dockerfile and the Caddyfile). Exit 0: package-lock.json 0, Dockerfile 0.
- **Re-entry R-1 (9d03cc6): `npm audit --omit=dev --audit-level=high`** reports `found 0 vulnerabilities` (exit 0). The R-1 delta changes no lockfile or dependency.
- **Secrets: Gitleaks v8.30.1.** Full history (47 commits) and the snapshot `dir` scan each exit 1 with 3 hits. All are known `generic-api-key` test fixtures (apps/api/test/team.test.ts ×2, apps/api/test/auth.test.ts:61), triaged as false positives in pass 1. The settle range (2 commits) is clean.
- **SAST: Semgrep 1.177.0** (`p/default p/typescript p/nodejsscan`). Exit 0, 38 results, none new; all are triaged (variable names, the HIBP SHA-1, anchored regexes, tooling scripts, SEC-7 tags). There were 2 partial parses and 12 rule timeouts, all on files unchanged in the diff; the diff hunks were reviewed by hand.
- **Dependencies.** `npm audit --omit=dev` reports 0 vulnerabilities. The full audit reports 1 dev-only low (esbuild GHSA-g7r4-m6w7-qqqr), which does not ship in runtime images.
- **Dependencies and config: Trivy 0.74.0** (`fs --scanners vuln,misconfig,secret --severity HIGH,CRITICAL`). Exit 0: package-lock.json 0, Dockerfile 0.
- **License.** Not flagged by RESEARCH.md, so skipped.
- **Web headers, CSP and cookies.** These were reviewed by hand and checked by the definitive `--full` web baseline (site.headers and site.cookies pass). CSP: per-request nonce plus `strict-dynamic`, `object-src 'none'`, `frame-ancestors 'none'`. The wide-close batch changed no header or CSP code.

## Advisories
- There are no dedicated regression tests for the SEC-10 per-user re-auth bucket or the SEC-9 grant branch.
- **CI security job.** The repo has no `.gitleaksignore` and no semgrep excludes, so the CI `security` job (`gitleaks --exit-code 1`, `semgrep --error`) will fail from its first run on the 3 known fixtures and the triaged results. Add the fixture fingerprints to `.gitleaksignore`, and `nosemgrep` annotations or rule excludes, so the job gates only on new findings.
- **Pending policy activated after migration.** A rule that names key A, is still pending when migration runs, and is activated afterwards will not cover the successor (informational). The policy editor could warn when a rule names a disabled key.
