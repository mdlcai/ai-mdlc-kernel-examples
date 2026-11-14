# Security Audit

This is the Security Audit Gate for RΛILS 1.0.1. It runs in three passes against SPEC.md §4 (SEC-1..SEC-20, the
§4.2 RBAC matrix and the §4.3 checklists) and ARCHITECTURE.md §9 (INV-1..INV-40).

| Pass | Commit | What it did | Where |
|---|---|---|---|
| 1. Initial scan (read-only subagent, parallel with the Reviewer) | 03a93c8 | Every scan category plus a manual adversarial review | `artifacts/security/pass-1.md` |
| 2. Auto-remediate (fix batches RB-3, RB-4, RB-5) | 4aa49b4, 9335abe, RB-5 | Fixed every HIGH and MEDIUM finding and two of the three LOW findings, each with a test. Reviewer round 2 found gaps in the RB-3 fixes (N1, N2, then N5 on re-verify), fixed in RB-4 and RB-5 | this file, §2 |
| 3. Re-scan | d426eab | Same scan as pass 1, against the remediated tree | `artifacts/security/pass-3.md` |

**Verdict:** PASS. After remediation and re-scan, CRITICAL 0 · HIGH 0 · MEDIUM 0 are open. One LOW is deferred with its reason (SA-4), and two LOWs are residuals (Reviewer round 2, N3 and N4).

## 1. Scan categories (pass 1)

| Category | Tool | Exit | Result |
|---|---|---|---|
| Dependencies (web, root) | `npm audit --omit=dev` | 0 / 0 | 0 vulnerabilities |
| Dependencies (api) | `bundle-audit check --update` (advisory DB 2026-10-17, 1266 advisories) | 0 | No vulnerabilities found |
| SAST (Rails) | Brakeman 8.1.0, 35 controllers / 29 models | 0 | 0 warnings |
| SAST (web + api) | Semgrep 1.95.0 `p/default`, 297 rules / 298 files | 2 (5 test files only partly parsed under a 2-CPU memory cap) | 7 findings, all false positives: an allow-listed `constantize`, the local variable `raw`, HIBP's protocol SHA-1, two server-built mail links, and two in test code |
| Secrets | gitleaks 8.21.2, 68 commits | 1 | 6 false positives: Stripe contract-test fixtures and example idempotency keys in docs. No `.env`, key or credential file is tracked |
| Configuration and infrastructure | Manual: Caddyfile, compose.yaml, CSRF/Origin, cookies, CSP, rack-attack, TLS | n/a | One finding (SA-1) |
| Environment variables | Every `ENV[...]` and `process.env.*` compared with `.env.example` | n/a | No hard-coded secrets. The only gaps are tooling and test variables. The dead `SSRF_ALLOW_HOSTS` was removed (ADR-050) |
| Web delivery headers | Web Delivery Baseline `site.headers` / `site.cookies` | n/a | Cookies pass. HSTS is absent on localhost with `tls internal`, which SEC-1 requires (ADR-048); it is sent once `TRUSTED_CERT=true` |

## 2. Findings and remediation

Pass 1 found **CRITICAL 0 · HIGH 1 · MEDIUM 2 · LOW 3**. The fix batch remediated 5 of them; one LOW was deferred.

| ID | Sev | Clause | Finding | Remediation (pass 2) | Test |
|---|---|---|---|---|---|
| SA-1 | HIGH | SEC-7, SEC-9 | The auth throttles compared the raw path, so `/login/`, `/login.json` and `//login` (which the router sends to the same action) skipped the per-IP and per-email limits. The per-email key also read JSON bodies only, so a form-encoded body escaped it. Together these allowed credential stuffing, spraying and reset-mail flooding at about 300 requests per second per IP. | Every throttle compares `RateLimitKeys.endpoint`, which is the path normalised the way the router matches it. `param_field` reads the field from Rails' own parsed params, the same ones the controller reads: query string, form body and JSON body under any content type Rails decodes as JSON (RB-4/RB-5, Reviewer round 2 N2 and N5: `?email=` with an empty body, and a `text/x-json` body, skipped the per-email limits). | `spec/requests/auth/sessions_spec.rb`: `/login/`, `/login.json` and three form-encoded attempts with a mixed-case email count toward one limit, and the next correct login gets `429 RATE_LIMITED`; `?email=` on login and on `password_resets` and a `text/x-json` body count toward the same limits (`sessions_spec.rb`, `password_resets_spec.rb`) |
| SA-2 | MEDIUM | SEC-12 | The webhook timeouts were per socket operation, with no overall deadline. A hostile endpoint that trickled bytes could hold one delivery for hours, and one sweep serves every tenant. | `Dispatcher` wraps each attempt in a 5 s `Timeout.timeout`, and the attempt counts as a retryable failure. The sweep defers an endpoint's remaining rows once it is slow (≥ 2 s), so one endpoint costs a sweep at most one deadline. | `spec/services/outbound_webhooks/dispatcher_spec.rb` (deadline gives `retrying` with the reason) and `spec/jobs/deliver_pending_webhooks_job_spec.rb` (the slow endpoint's second row is deferred, and another tenant's row is still delivered) |
| SA-3 | MEDIUM | §4.3 web, SEC-2 | `safeNext` accepted `/\t/evil.example`, which the URL parser turns into another origin, so a genuine sign-in link could redirect to an attacker page after login. | Control characters and backslashes are refused. The value is resolved against a placeholder origin and kept only if it stays on it and its path does not start with `//` (RB-4, Reviewer round 2 N1: dot segments such as `/..//evil.example` collapsed into a protocol-relative path). | `web/app/(auth)/password-input.test.ts`: tab, newline, CRLF, decoded `%09` and `..\` variants fall back to `/dashboard`, and `/..//`, `/.//`, `/%2e%2e//` and `/x/..//` variants fall back too; encoded tabs in a query still work |
| SA-4 | LOW | SEC-8 | bcrypt ignores bytes after 72, so a 128-character passphrase is only as strong as its first 72 bytes. | **Deferred.** Rejecting more than 72 bytes would break SEC-8 (15–128 characters) and NIST's 64-character floor for non-ASCII passwords. The proper fix is a versioned pre-hash (HMAC-SHA256, then bcrypt) with a rehash-on-login migration. That changes a security primitive's input and needs its own ADR and migration, so it is scheduled after 1.0.1. 72 bytes is still at least 72 ASCII characters (about 18 four-byte characters), far above the 15-character floor. | n/a |
| SA-5 | LOW | SEC-12 | The SSRF block list missed IPv6 transition ranges that embed an IPv4 address: 6to4 `2002::/16`, Teredo `2001::/32`, local NAT64 `64:ff9b:1::/48` and site-local `fec0::/10`. | Added to `SsrfGuard::BLOCKED`. | `spec/services/outbound_webhooks/ssrf_guard_spec.rb`: a literal from each range is refused |
| SA-6 | LOW | SEC-9 | A locked account answered its 401 without a bcrypt compare, about 250 ms faster than an unknown email, so its timing confirmed that the account exists. | The locked branch pays for the same dummy compare as an unknown email. | `spec/services/passwords/login_spec.rb`: a locked account still runs the dummy-digest compare |

There were no CRITICAL findings, so there is nothing for mandatory post-build review under the CRITICAL rule.
No security primitive was swapped.

## 3. Re-scan (pass 3)

Pass 3 re-ran every pass-1 category against d426eab (RB-5, the tree after RB-3, RB-4 and RB-5). The full record is in `artifacts/security/pass-3.md`.

- **Dependencies:** npm audit (web and root) found 0 vulnerabilities, and bundle-audit found none.
- **Brakeman:** 0 warnings.
- **Semgrep:** 8 false positives. The seven from pass 1 are back, plus an Arel `lt` predicate in `concerns/pagination.rb:15` (the value is bound, and the file is unchanged since F-01). Pass 1 likely missed it because some files only partly parsed under that run's memory cap.
- **gitleaks:** the same 6 false positives as pass 1.
- **No new finding.**

Reviewer round 2 found three gaps in the RB-3 fixes themselves. Each was fixed with a test and passed the Reviewer's re-verify:

- N1, HIGH: `safeNext` collapsed `/..//host` into a protocol-relative path. Fixed in RB-4 (9335abe).
- N2, MEDIUM: `?email=` skipped the per-email throttles. Fixed in RB-4.
- N5, MEDIUM: a `text/x-json` body skipped them too. Fixed in RB-5 (d426eab).

**Residual, LOW (Reviewer N3).** The webhook sweep picks its 100 oldest due rows before it defers slow endpoints. A single endpoint with 100 or more due retries can therefore occupy one sweep, delaying other tenants' retries by about a minute per sweep. Fresh deliveries go out per row, and the deadline still bounds each attempt. The fix (a per-endpoint cap in the batch query) is scheduled after 1.0.1.

**Residual, LOW (Reviewer N4).** Two idempotent-resume edges remain. First, the 60 s stale-claim window is shorter than the Stripe client's read timeout, so a retry during a very slow first call can run alongside it; the processor idempotency keys and the settle CAS still prevent a second charge or refund. Second, a failover resume re-runs the routing chain from the first processor: if processor A errored before reaching Stripe, B succeeded and the local commit then failed, a retry can succeed on A and charge twice. This window predates RB-3 (stale-claim resume at 03a93c8). The fix (persist attempts before the settle step, or resume from the recorded processor) is scheduled after 1.0.1; Stripe is in contract mode in this deployment (ADR-008), so live failover across two real accounts is not reachable here.

## 4. Trust boundaries

There is one public entry point: Caddy on :9443. It terminates TLS, redirects `http://` with a 308, strips `Server`,
and overwrites `X-Request-Id`. Caddy trusts no upstream `X-Forwarded-For`, so rate limits see the real client IP.
Postgres (primary and replica), Redis, Solid Queue and Mailpit are not published.

Data crosses into the system at four boundaries:

1. **Browser → edge → API.**
   - The session cookie is `__Host-`, `secure`, `httponly` and `SameSite=Lax`. It expires after 30 minutes idle or 12 hours, and it is rotated on login and revoked on logout and on password reset.
   - Every unsafe verb also needs an allow-listed `Origin` and an HMAC CSRF token.
   - The `Rails-Mode` header only chooses between the caller's own test and live data.
2. **Merchant servers → API.**
   - `sk_<mode>_` keys are stored as SHA-256 digests, bound to one mode, and act with the developer role.
   - Money POSTs need an `Idempotency-Key`, which is unique per merchant, mode and key and fingerprinted by method, path and canonical body.
3. **Stripe → API.** `construct_event` verifies the signature, with a 300 s tolerance, before anything is written. Events are deduplicated with `insert_all unique_by`, and `livemode` must match the mode.
4. **API workers → merchant URLs** (outbound webhooks). This is the weakest boundary, because a tenant chooses the destination.
   - The guard requires https.
   - It refuses credentials and internal names, resolves the name, and refuses the request if any address is private, metadata or transition-embedded (SA-5).
   - It pins the connection to the checked IP with `VERIFY_PEER`, and follows no redirects.
   - Since RB-3, a tenant also cannot hold a shared worker beyond one 5 s deadline per sweep (SA-2).

## 5. Data-flow risks

- **Money and ledger.**
  - Each change commits in one transaction with its outbox row (INV: outbox in the same transaction).
  - Locks are taken in ADR-045 order: the audit chain, then the ledger accounts.
  - Journals are immutable and balanced by deferred database triggers. The hourly T3 job verifies them and flags any drift.
- **Idempotent retries.**
  - Reviewer round 1 found that a 5xx after the processor call deleted the idempotency claim, so a retry could charge or refund twice.
  - RB-3 releases a claim without deleting it once a resource is attached. The retry then resumes the same payment or refund under the same processor idempotency key.
  - Request specs cover both the refund 503 path and the payment-create crash after authorization.
- **Personal data.**
  - Customer and user emails and names are encrypted at rest with Active Record encryption (`support_unencrypted_data = false`), and looked up through a blind index.
  - Lists mask them, and logs and audit data pass through `filter_parameters`.
  - The PAN guard refuses any Luhn-valid card number in a card-like parameter, plus the description and metadata, before anything is stored, and never echoes it (SEC-10).
- **Exports.** CSV cells that a spreadsheet would run as a formula (`= + - @`, tab and CR) are prefixed and quoted, and every export is audited.
- **What remains is availability and abuse:**
  - The rate limits are per IP, per email, per key and per session, in a shared Redis.
  - The 300 requests per second per-IP ceiling is the backstop.
  - The load test (REPORT.md, ADR-047) measures these limits under load.

## 6. Authentication and authorization edge cases

- **Tenant isolation.**
  - Every resource lookup goes through `scoped` or `find_scoped!` on (`current_merchant`, mode) by public id.
  - Another tenant's resource answers 404, including a refund or sandbox dispute naming a foreign payment (SEC-5, ADR-049).
  - Action Cable re-checks membership every 60 s.
- **Roles.**
  - The §4.2 matrix matches SPEC.
  - `/v1/ops/*` answers 404 to anyone who is not an operator.
  - `/me` is scoped to the current user.
  - API keys refund with the developer role, as recorded in an ADR.
- **Suspension.** It is enforced on every path except sign-out, merchant switching and ops. These are deliberate exemptions so that a user of a suspended merchant can still leave.
- **Lockout and enumeration.**
  - Ten failures lock an account for 15 minutes, with one generic error text.
  - Unknown emails and, since RB-3, locked accounts pay the same bcrypt cost (SA-6).
  - With SA-1 fixed, the per-email and per-IP throttles apply whatever form the path or body takes.
- **Redirects after sign-in.** `next=` stays on the console origin (SA-3).
- **Passwords.**
  - 15–128 characters, with no composition rules.
  - A HIBP k-anonymity screen that fails open and flags the account for the T6 re-screen (ADR-006).
  - bcrypt cost 12.
  - The 72-byte bcrypt input limit is the one open residual (SA-4, deferred above).
