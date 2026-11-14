Commit audited: 03a93c8

# Security Audit — pass 1 (read-only)

- **Date:** 2026-10-18
- **Scope:** `api/` (Rails 8.1.4), `web/` (Next 16.4.2), `edge/Caddyfile`, `compose.yaml`, `.env.example`.
- **Checked against:**
  - SPEC.md §4: SEC-1..SEC-20, the §4.2 RBAC matrix and the §4.3 checklists.
  - ARCHITECTURE.md §9 (INV-1..INV-40).
- **Note on the tree:** the working tree's HEAD (ff7d921) differs from 03a93c8 only in `GOVERNANCE-DECISIONS.jsonl`, so every file reviewed here is at 03a93c8.
- **Rules followed:**
  - No project file was edited.
  - No test suites or compose builds were run, and no running container was touched.
  - No database was read.
  - Every scanner ran in a throwaway container. The probes that execute code ran with `--network none` and `--cpus=1` or `--cpus=2`.

## 1. Scan results by category

### 1.1 Dependencies

| Tool | Command | Exit | Result |
|---|---|---|---|
| npm audit (web) | `cd web && npm audit --omit=dev` | 0 | found 0 vulnerabilities |
| npm audit (root) | `npm audit --omit=dev` | 0 | found 0 vulnerabilities |
| bundler-audit | `docker run --rm -v "$PWD/api:/rails" rails-api-dev bundle exec bundle-audit check --update` | 0 | ruby-advisory-db updated (1266 advisories, 2026-10-17). "No vulnerabilities found" |

### 1.2 SAST

**Brakeman 8.1.0**

- **Command:** `docker run --rm -v "$PWD/api:/rails" rails-api-dev bundle exec brakeman --no-pager -q --no-exit-on-warn`
- **Exit code:** 0
- **Coverage:** 35 controllers and 29 models.
- **Result:** 0 errors and **0 security warnings**.

**Semgrep 1.95.0**

- **Command:** `docker run --rm --cpus=2 -v "$PWD:/src" semgrep/semgrep:1.95.0 semgrep scan --config p/default --metrics=off /src/web /src/api/app`
- **Exit code:** 2. Under the 2-CPU cap, Semgrep ran out of memory parsing `web/lib/payments.test.ts`, so 5 files were only partly analysed. The rest of the scan finished.
- **Coverage:** 297 rules on 298 files.
- **Result:** 7 findings, all triaged as false positives:

| Rule | Location | Triage |
|---|---|---|
| check-unsafe-reflection | `api/app/controllers/api/v1/ops/scheduled_runs_controller.rb:28` | **FP.** `job` is the value from the fixed `ScheduledRun::JOBS[...]` allow-list (line 20). An unknown name gets a 422 before `constantize` runs. The endpoint is operator-only, non-production and gated on `OPS_TRIGGER_ENABLED`. |
| avoid-raw | `api/app/models/password_reset.rb:12` | **FP.** `raw` is a local variable holding the raw token, not the `raw()` view helper. |
| weak-hashes-sha1 | `api/app/services/passwords/breach.rb:13` | **FP.** The HIBP k-anonymity protocol requires SHA-1 (SEC-8). It is never used as a password hash. |
| var-in-href ×2 | `transactional_mailer/invitation.html.erb:3`, `password_reset.html.erb:3` | **FP.** The URLs are built on the server as `"#{APP_ORIGIN}/invite/#{raw}"` and `"#{APP_ORIGIN}/reset-password/#{raw}"` (`team/invite.rb:23`, `passwords/reset.rb:14`). No user input reaches the scheme. |
| detect-insecure-websocket | `web/context/realtime.test.tsx:98` | **FP.** A test assertion. |
| detect-non-literal-regexp | `web/e2e/w7-developer.spec.ts:40` | **FP.** E2E test code. |

### 1.3 Secrets

**gitleaks v8.21.2**

- **Command:** `docker run --rm --cpus=2 -v "$PWD:/repo" zricethezav/gitleaks:v8.21.2 detect --source /repo --no-banner --redact -v`
- **Exit code:** 1 (leaks reported).
- **Coverage:** 68 commits scanned.
- **Result:** 6 findings, all false positives:
  - **`api/spec/services/processors/stripe_adapter_spec.rb:40,51`** (stripe-access-token). These are made-up `sk_test_51Cont…` contract-test fixtures in a spec. They are not live keys.
  - **`web/app/(marketing)/docs/page.tsx:78`** (generic-api-key). An example `Idempotency-Key` UUID in the public docs.
  - **`DESIGN-TEMPLATE.html:1175-1177`** (generic-api-key ×3). Example idempotency keys in code samples.

No real credentials, `.env` file or private keys are tracked.

### 1.4 Configuration and infrastructure

I reviewed these by hand.

**`edge/Caddyfile`**
- Uses `tls internal`. The `http://` site sends a 308 to `https://localhost:{HTTPS_PORT}` (SEC-1).
- Offers only h1/h2, has the admin API off and caps headers at 32KB.
- Strips `Server` and `X-Powered-By`.
- Overwrites `X-Request-Id` with its own UUID.
- Does not trust any upstream `X-Forwarded-For`, because Caddy's default `trusted_proxies` is empty. Rack-attack therefore sees the real client IP.

**`compose.yaml`**
- Only the edge (9443 and 9080) and Mailpit (bound to 127.0.0.1) are published. Postgres, Redis, api and web are reachable only on the internal network.
- Secrets use the `${VAR:?}` form, so compose refuses to start without them.

**Origin checks and CSRF (SEC-3)**
- Every unsafe request on a session needs an `Origin` in `EnvSchema.allowed_origins` plus `secure_compare` of `X-CSRF-Token`, which is HMAC(secret_key_base, "csrf:"+digest).
- Unauthenticated auth POSTs go through `verify_origin!`.
- Action Cable sets `allowed_request_origins` with `allow_same_origin_as_host = false`.

**Cookies (SEC-2)**
- `__Host-rails_session` is encrypted, `httponly`, `secure`, `same_site: :lax` and `path: "/"`.
- Sessions time out after 30 minutes idle and 12 hours absolute.
- The session is rotated on login and revoked on logout and on password reset.

**CSP (SEC-18)**
- `web/proxy.ts` implements Profile N: a per-request nonce plus `'strict-dynamic'`, `object-src 'none'`, `frame-ancestors 'none'`, `form-action 'self'`, and `'unsafe-eval'` only when not in production.
- The API sends `default-src 'none'; frame-ancestors 'none'; base-uri 'none'`, plus nosniff, XFO DENY, Referrer-Policy, Permissions-Policy, COOP and CORP.

**Rack-attack (SEC-7)**
- Uses a Redis store.
- `RATE_LIMIT_SCALE` is forced to 1 in production, and `EnvSchema::PROD_FORBIDDEN_TRUE` refuses `ALLOW_RATE_LIMIT_SCALE`, `ALLOW_HTTP_WEBHOOKS` and `OPS_TRIGGER_ENABLED` in production.
- **Two of the throttles can be bypassed (SA-1).**

**TLS**
- `production.rb` sets `assume_ssl = true` and `force_ssl = false`, because the edge owns TLS.
- Outbound webhooks use `VERIFY_PEER` while pinning the IP.

### 1.5 Environment variables

I compared every `ENV[...]` / `ENV.fetch` reference in `api/{app,config,lib,bin}` and every `process.env.*` reference in `web/` (excluding node_modules) against `.env.example`.

**Documented:** every runtime variable is in `.env.example` or is set by compose.yaml or an image. That covers:
- secrets: `SECRET_KEY_BASE`, the `AR_ENCRYPTION_*` keys, `BLIND_INDEX_KEY`, `STRIPE_*`, `OPERATOR_*`;
- database and cache URLs: `DATABASE_URL`, `DATABASE_REPLICA_URL`, `REDIS_URL`;
- origins: `APP_ORIGIN`, `ALLOWED_ORIGINS`;
- service and runtime settings: `HIBP_*`, `SMTP_*`, `LOG_LEVEL`, `RAILS_MAX_THREADS`, `WEB_CONCURRENCY`, `DB_STATEMENT_TIMEOUT_MS`, `RATE_LIMIT_SCALE`;
- switches: `ALLOW_*`, `OPS_TRIGGER_ENABLED`, `TRUSTED_CERT`;
- build and web: `APP_VERSION`, `GIT_COMMIT`, `API_INTERNAL_URL`;
- set by the images: `PORT`, `PIDFILE`, `NODE_ENV`, `BUNDLE_GEMFILE`.

**Missing from `.env.example`:** only tooling and test variables:
- `LOADTEST_INTENTS` and `LOADTEST_PASSWORD` (`api/lib/loadtest/seed.rb`);
- `SKIP_ENV_SCHEMA`, which is ignored when `RAILS_ENV=production` (`api/lib/env_schema.rb:62`);
- the web e2e variables `E2E_OPERATOR_EMAIL`, `E2E_OPERATOR_PASSWORD`, `E2E_RESET_LIMITS`, `MAILPIT_URL` and `NODE_TLS_REJECT_UNAUTHORIZED`.

None of these is read by the running stack.

**Defined but never read:** `SSRF_ALLOW_HOSTS` (`.env.example:64`) is not referenced anywhere in `api/`. It is a dead variable, not a vulnerability.

**Hardcoded secret values:** none found.
- The only literal fallbacks for secret-class variables are the `rails:rails@localhost` URLs in the development and test sections of `config/database.yml:15-27`.
- Production values come from compose with `:?`.
- `EnvSchema` refuses to boot without the required keys.

### 1.6 Web delivery headers and CSP

**Source:** `artifacts/web-baseline/FRONTEND-AUDIT.nolh.summary.json` (the summary only). It was built at 2026-10-18T06:57Z with buildMode `production` against baseUrl `https://localhost:9443`.

**Baseline results:**
- **`site.headers`: fail.** The detail is "landing: no Strict-Transport-Security | dashboard: no Strict-Transport-Security".
- **`site.cookies`: pass.**
- **Header-related groups:** no other group failed. The only other failures were layout checks (targets44 on `docs` at 375px and overflow on `webhook-endpoint` at 375px), which are not security issues.

**HSTS check against the code:**
- HSTS is set only in `web/proxy.ts:32-34`, under `if (process.env.TRUSTED_CERT === 'true')`.
- Neither the Caddyfile nor the API's `security_headers.rb` sends HSTS.
- compose passes `TRUSTED_CERT=${TRUSTED_CERT:-false}`, and `.env.example:79` sets `false`.

So the missing HSTS on localhost with `tls internal` is the **intended behaviour required by SEC-1**, not a defect. The `headers` check will pass once a deployment sets `TRUSTED_CERT=true`.

## 2. Findings

**Counts:** CRITICAL 0 · HIGH 1 · MEDIUM 2 · LOW 3

| ID | Sev | Clause | Exploit path | Location | Remediation |
|---|---|---|---|---|---|
| SA-1 | HIGH | SEC-7 (and SEC-9, PCI-8.3.4 in effect) | Every auth throttle matches the exact string `req.path == "/api/v1/auth/login"` (likewise `/signup`, `/password_resets` and `/governance/evidence_exports`). The Rails router treats `/api/v1/auth/login/` and `/api/v1/auth/login.json` as the same route. I confirmed this with `Rails.application.routes.recognize_path`: both resolve to `api/v1/auth/sessions#create`, and `/signup/` and `/password_resets/` resolve to their controllers. Caddy passes these paths through unchanged. Requests to those paths skip `login/ip` (20/min), `login/email` (5/min), `signup/ip` (5/h), `reset/email` (3/h) and the evidence-export limit, leaving only `api/ip` at 300 per second. Separately, `login/email` and `reset/email` get their key from `RateLimitKeys.json_field`, which returns `nil` unless the body is `application/json`. Rails also parses `application/x-www-form-urlencoded`, and the controllers read `params[:email]`, so a form-encoded body against the canonical path also escapes both per-email limits. A script that sets `Origin: https://localhost:9443` (which `verify_origin!` accepts) can therefore make about 300 requests per second per IP. That allows credential stuffing (lockout costs 10 guesses per account every 15 minutes, across unlimited accounts), password spraying, mass signups, and flooding a victim's inbox with reset emails, bypassing the 3/h per-email cap. | `api/config/initializers/rack_attack.rb:16, 41, 44, 48, 51, 68` | Match on a normalised path, for example `p = req.path.chomp("/").delete_suffix(".json")`, or on a regex `%r{\A/api/v1/auth/login(?:\.json)?/?\z}`. Alternatively, reject a trailing slash or format suffix at the edge or in a middleware placed before Rack::Attack. Get the email for throttling from `req.params["email"]` (which covers JSON and form bodies), or have the auth endpoints return 415 for any non-JSON body. Add request specs for `/login/`, `/login.json` and a form-encoded body. |
| SA-2 | MEDIUM | SEC-12 ("5s timeout"); INV on tenant isolation of the delivery worker | `post!` sets `open_timeout`, `read_timeout`, `write_timeout` and `ssl_timeout` to 5 seconds each but has no overall deadline. `read_timeout` restarts on every socket read, and the body is read in chunks up to 64 KB. A merchant (any developer, or anyone holding an API key) can register an https endpoint that sends status-line or header bytes, or body bytes, every 4.9 seconds, which holds one delivery for hours (64 KB at that rate is far more than 24 hours). `DeliverPendingWebhooksJob#deliver` calls the dispatcher **one row after another** for up to `SWEEP_BATCH` (100) rows **across every merchant**, so one hostile endpoint, especially with many pending rows, stalls webhook delivery for all tenants. After `CLAIM_TTL` (10 minutes) the stalled rows also become reclaimable, so they are re-posted in parallel and use up more worker threads. | `api/app/services/outbound_webhooks/dispatcher.rb:58-74`; `api/app/jobs/deliver_pending_webhooks_job.rb:19-23` | Enforce a deadline per attempt from a monotonic clock: check `Process.clock_gettime(Process::CLOCK_MONOTONIC)` inside the `read_body` block and around `request`, and fail at 5 seconds total, or wrap the attempt in a bounded `Timeout.timeout(5)`. Have the sweep deliver per merchant or per endpoint, for example by fanning out jobs per endpoint with a concurrency limit, so one endpoint cannot block other tenants. Keep `CLAIM_TTL` well above the maximum attempt time. |
| SA-3 | MEDIUM | §4.3 web checklist / SEC-2 phishing resistance (no direct SEC clause; OWASP open redirect) | After sign-in the form calls `router.replace(safeNext(params.get('next')))`. `safeNext` accepts any value that starts with `/` and does not start with `//` or `/\`. A tab or newline after the first slash passes that check, but the WHATWG URL parser strips it. I confirmed `/sign-in?next=/%09/evil.example/x` gives `next = "/\t/evil.example/x"`, `safeNext` returns it, and `new URL(next, 'https://localhost:9443/sign-in')` is `https://evil.example/x`. The Next router treats a different origin as external (`isExternalURL`) and does a hard navigation. An attacker emails a genuine RΛILS sign-in link. The victim signs in on the real origin and lands on an attacker page (for example "session expired, re-enter password"), which harvests credentials or passes itself off as the console. | `web/app/(auth)/password-input.tsx:49-51`; `web/app/(auth)/sign-in/sign-in-form.tsx:27` | Resolve the URL and check its origin: `const u = new URL(next, location.origin); return u.origin === location.origin ? u.pathname + u.search + u.hash : '/dashboard'`. Optionally also reject any control character or backslash (`/[\u0000-\u001f\\]/`). Add unit cases for `/\t/x`, `/\n/x`, `/%09/x` and `/\\x`. |
| SA-4 | LOW | SEC-8 (15–128 chars, bcrypt) | The policy measures password length in **characters** up to 128. `has_secure_password validations: false` turns off Rails' 72-byte check, and bcrypt 3.1.22 silently truncates at 72 bytes. I confirmed that a hash of `"a"*72 + "X"*50` accepts `"a"*72 + "Y"*10`. Anything after byte 72 (as few as 18 four-byte characters) adds no strength. Users who rely on long passphrases with a shared prefix get much less entropy than they expect. | `api/app/models/user.rb:5`; `api/app/services/passwords/policy.rb:17` | Reject `password.bytesize > 72` in `Policy.check!` with a clear message. Alternatively, pre-hash, for example `bcrypt(Base64(HMAC-SHA256(pepper, pw)))`, with a migration path. |
| SA-5 | LOW | SEC-12 (block private ranges) | `BLOCKED` covers `::ffff:0:0/96` and `64:ff9b::/96` but not the other IPv6 transition ranges that embed an IPv4 address: `2002::/16` (6to4), `2001::/32` (Teredo), `64:ff9b:1::/48` (local NAT64) and the deprecated `fec0::/10` (site-local). An attacker who controls DNS can make the endpoint resolve to an AAAA record such as `2002:a9fe:a9fe::`. If the worker host has a 6to4 or NAT64 route, that reaches 169.254.169.254 or RFC 1918 hosts. The compose network has no IPv6, which limits exploitability today. | `api/app/services/security/ssrf_guard.rb:13-17` | Add `2002::/16`, `2001::/32`, `64:ff9b:1::/48` and `fec0::/10`. Better still, for 6to4, Teredo and NAT64 addresses, extract the embedded IPv4 address and check it against the v4 list. |
| SA-6 | LOW | SEC-9 (no enumeration) | Unknown emails pay for a dummy bcrypt compare, but a **locked** account returns before any compare. The attacker locks a candidate account by sending 10 wrong passwords. After that, its 401 returns about 250 ms faster (cost 12) than the 401 for an email that does not exist. The response timing reveals that the account exists, even though the error text is the same. | `api/app/services/passwords/login.rb:20` | Run `BCrypt::Password.new(DUMMY_DIGEST).is_password?(password)` (or `user.authenticate`, discarding the result) before raising on the locked branch. |

**Reviewed with no finding:**
- authN: API keys are stored as SHA-256 digests, are mode-bound and take the developer role. Session lookup goes through the `active` scope.
- authZ: the §4.2 MATRIX matches SPEC. Ops endpoints return 404 to non-operators. The `/me` endpoints are scoped to `current_user`.
- Tenant isolation: `scoped`/`find_scoped!` look up by public id, and every resource controller uses them. Channels reauthorise every 60 seconds.
- Idempotency: the key is UNIQUE by merchant, mode and key, with a fingerprint of method, path and canonical body.
- Stripe: `construct_event` runs with a 300-second tolerance before any processing. Dedupe uses `insert_all unique_by`, and livemode is checked against the mode.
- Outbound webhook signatures: HMAC `t=,v1=`.
- PAN guard.
- AR encryption: `support_unencrypted_data = false`, with a blind index for email.
- Audit chain: an advisory lock per merchant, a hash chain, readonly records plus a database trigger, and data filtered through `filter_parameters`.
- CSV formula injection: `= + - @ \t \r` are prefixed and the cell is quoted.
- Logo upload: magic-byte sniffing, HEIC rejected, metadata stripped and dimensions capped.
- Mass assignment: merchant update permits 5 fields, and the other controllers use explicit permits.
- SQL injection: Brakeman is clean and no interpolated SQL was found.
- Log redaction: tokens in paths are redacted, query values are filtered, and the filter list covers passw, email, secret, token, _key, pan, cvv, number, authorization and evidence.
- Password reset is single-use and revokes sessions. HIBP is called with `Add-Padding`, and fail-open is the decision recorded for SEC-8.
- `dangerouslySetInnerHTML` appears only in `components/seo/json-ld.tsx`, where `<` is escaped and the nonce is applied.

## 3. Narrative

**Trust boundaries.**
- There is one public entry point, Caddy at :9443. It terminates TLS and splits traffic by path: `/api/*` and `/cable` go to Puma, and everything else goes to Next.
- Internal services (Postgres primary and replica, Redis, Solid Queue and Mailpit SMTP) are not published.
- Data crosses trust boundaries in four places:
  1. **browser → edge → API**, carrying the session cookie, CSRF HMAC and `Rails-Mode` header;
  2. **merchant servers → API**, using Bearer `sk_<mode>_` keys;
  3. **Stripe → API**, as signed webhooks;
  4. **API workers → arbitrary merchant URLs**, for outbound webhooks.
- The inbound boundaries are well defended. Origin plus CSRF HMAC protects sessions, keys are stored only as digests, and Stripe signatures are verified before any parsing side effects.
- The weakest boundary is the fourth one. The SSRF guard resolves the name, pins the IP and verifies the certificate, which is good. But the HTTP exchange that follows has no overall time limit, and one sweep serves every tenant (SA-2), so a hostile tenant controls how long a shared worker stays busy.

**Data-flow risks.**
- Money and ledger writes run in a transaction that also writes the outbox row, and they follow the lock order (audit chain, then ledger accounts). I saw no path where an event is emitted without its change being committed.
- PII (customer email and name, user email) is encrypted at rest. Lookups use a deterministic blind index, list views are masked, and logs and audit data are filtered.
- Exports neutralise formulas.
- The rest of the data-flow risk is around availability and abuse rather than confidentiality: SA-1 removes the per-account and per-IP brakes on the auth endpoints, and SA-2 allows a cross-tenant delivery stall.

**AuthN and authZ edge cases.**
- Mode comes from a header, but every query is scoped by (`current_merchant`, `current_mode`), and API keys are bound to a mode. Switching the header therefore only changes which of the caller's own datasets they see.
- Suspension is enforced everywhere except the `Auth::`, `Ops::` and `me/active_merchants` paths. These are deliberate exemptions so a user can log out or switch merchants.
- API keys bypass role checks on refunds, as recorded in an ADR. In effect a key holds the developer role.
- Lockout is per account, with generic error text. However, the rate-limit bypass in SA-1 turns lockout into the only control against spraying many accounts. The timing difference in SA-6 lets an attacker confirm that the accounts they locked exist.
- The open redirect after login (SA-3) does not break authentication. It makes phishing that follows a real sign-in much more convincing.
- bcrypt truncation (SA-4) quietly reduces the strength that SEC-8's 128-character limit implies.
