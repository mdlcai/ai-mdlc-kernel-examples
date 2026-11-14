# Security Audit — Pass 1

Project: QAI // QUANTUM SDK (ce0ff2f1-2252-49ea-8d2e-5a25f2e08102)
Auditor: fresh-context Security auditor, pass 1
Date: 2026-11-01
Verdict: **PASS WITH ADVISORY**. There are no critical or high findings. Two medium findings should be fixed before release. The verdict is degraded to advisory because no secret-detection tool and no TypeScript SAST tool is available (see Method).

## Scope

- `services/api/qai_api`: routes, deps, middleware, services, repositories, db (scope, engine, rls), the worker (loop and child), config, logging, errors, and the migrations' RLS and grants.
- `packages/qai-sdk/qai`: compile and validation, insofar as they are reachable from the API.
- `apps/web`: proxy.ts (CSP and headers), lib/api.ts, lib/api-server.ts, lib/session.tsx, the credential forms, and the open-redirect guard.
- `infra/caddy/Caddyfile`, `compose.yaml`, `services/api/Dockerfile`, `apps/web/Dockerfile`, `.env.example`, `.gitignore` and `.github/workflows/ci.yml`.
- References: ARCHITECTURE.md §4, §5, §7, §7.1 and §9 (INV-1..33); SPEC.md §3 and §4 (S1–S14); BUILD-full.md pass-1 categories.
- Out of scope, per the brief: running containers, e2e and the web baseline. HSTS being off on the localhost `tls internal` deploy is deliberate (ADR-29) and is not a finding.

## Method

| Category | Tool | Result |
|---|---|---|
| Dependency vulns (npm) | `npm audit --omit=dev`, `npm audit --json` | 0 production advisories. 18 dev advisories triaged (F-8, F-12). |
| Dependency vulns (Python) | `pip-audit 2.10.0 --no-deps` on the pinned direct deps of services/api and qai-sdk | No known vulnerabilities. Transitive deps are not locked (F-9). |
| SAST (Python) | `ruff 0.15.0 --select S` (bandit rules) on qai_api and qai | All checks passed. One `noqa: S608` was reviewed by hand and is safe (fixed allow-list). |
| SAST (TypeScript) | none | **ADVISORY ONLY — no external tooling available for SAST (TypeScript/web). Agent self-review is not a substitute for automated scanning.** Manual review done. Recommend semgrep (`p/typescript`, `p/react`, `p/nextjs`) or eslint-plugin-security in CI. |
| Secret detection | none (gitleaks, trufflehog and semgrep are not installed) | **ADVISORY ONLY — no external tooling available for secret detection. Agent self-review is not a substitute for automated scanning.** I ran a regex grep for AWS keys, private keys, GitHub and Slack tokens, `sk-` keys and literal passwords, excluding node_modules, .next and .env. It found nothing. `.env` is git-ignored, and `.env.example` holds only `change-me-*` placeholders. Recommend gitleaks in CI and as a pre-commit hook. |
| Config / infra | manual review of compose, Caddyfile, Dockerfiles and CI | See findings and the no-finding list. |
| Env var audit | `.env.example` compared with `config.py` fields and compose interpolation | Consistent. The required secrets fail fast through `${VAR:?}` in compose, and config validation exits 1 on a bad value. |
| Web headers / CSP / forms / cookies / SSRF | manual review of proxy.ts, middleware.py, sessions.py, the forms and lib/api.ts | See below. |
| Authz / tenant isolation | manual trace of every route through deps, scope, repository and RLS | See below. |
| License compliance | not flagged | Skipped, as the kernel allows. |

I verified every finding by reading the code at the cited lines. Each finding carries a confidence level. I confirmed M-1's cost empirically: I timed `qai.compile` locally on circuits at the maximum allowed size, using the scratchpad script `compile_time.py`.

## Findings

| id | severity | file:line | description | exploit scenario | fix | confidence |
|---|---|---|---|---|---|---|
| F-1 | **medium** | services/api/qai_api/routes/circuits.py:128-135 (`sdk_compile(...)` at :134) | `POST /circuits/{id}/compile` needs only the Viewer role and accepts API keys. It runs `qai.compile` (ccx decomposition into 15 ops, then a fixed-point peephole pass) **synchronously on the asyncio event loop**. A circuit at the cap (2000 ccx ops) takes about 0.4–0.7 s of pure CPU per call (measured locally; the output is about 28k ops). No compile-specific rate limit exists: only the per-principal limit (600/min) and the per-IP limit (300/min, fail-open) apply. The same pattern, at lower cost, applies to `to_sdk` validation on circuit create and PUT (routes/projects.py:338). | A viewer, or a viewer API key in any tenant, stores one 2000-ccx circuit and then calls compile in a loop from a few IPs. Every call blocks that API worker's event loop for about 0.5 s, so two replicas saturate at a few requests per second. All tenants see stalled or timed-out requests, and health checks may fail. One low-privilege account can deny service platform-wide (cross-tenant availability impact). | Move compile off the loop with `await asyncio.to_thread(...)`, or better, a bounded `ProcessPoolExecutor` with a timeout. Add a per-principal compile limit, e.g. 30/min, fail-closed. Optionally cap the decomposed op count or reject a compile whose output would exceed N ops. | high |
| F-2 | **medium** | services/api/qai_api/routes/members.py:213-224 (issue-time check); services/api/qai_api/routes/auth.py:285-313 (claim at :291, no membership re-check) | Admin-issued password resets are guarded against cross-workspace takeover **only when the token is issued**. The issuer must be the only workspace the target belongs to, and otherwise gets a 409. The token lives 24 h (members.py:35). At claim time `password_reset` sets the password without re-checking the user's memberships, and `password_resets` records no issuing tenant. Accepting an invite does not invalidate pending resets: `invalidate_password_resets` is called only from members.py:224. Removing the member does not invalidate them either. | An admin of T1 issues a reset for a member whose only workspace is T1 and keeps the token. Within 24 h the member accepts an invite to T2, or is removed from T1 and later joins T2. The T1 admin redeems the token, sets the victim's password and signs in as the victim. That gives full access to T2 at the victim's role. This crosses the tenant boundary the issue-time 409 exists to protect. | Store `tenant_id` (the issuer's workspace) on `password_resets`. At claim time, in the same transaction, reject the token if the user has any membership outside that tenant or no longer belongs to it, with the same generic 400. Also call `invalidate_password_resets(user_id)` on invite acceptance and on member removal. | medium-high |
| F-3 | **medium** | apps/web/lib/session.tsx:29-40; apps/web/lib/api-server.ts:19-22; services/api/qai_api/deps.py:64-65, :143; compose api command (`--proxy-headers --forwarded-allow-ips "*"`) | Server-side rendering calls the API at `API_INTERNAL_URL` and forwards only the cookie, **not the client IP**. So every SSR `/auth/me` call, one per `/app` page render for all users, shares a single per-IP bucket keyed on the web container's address (300/min). `enforce_ip` runs before authentication (deps.py:143). proxy.ts checks only that the session cookie *exists*. | An unauthenticated attacker sends more than 300 requests/min to `https://host/app` with a junk `__Host-qai_session` cookie. Each request makes the web server call `/auth/me`, using up the shared bucket. For the rest of the window, every real user's app shell gets a 429, and `getShellSession` renders the "unavailable" degraded shell. This is a cheap, unauthenticated, platform-wide denial of service on the web UI, and it can also occur organically above about 300 page loads/min. | Forward the real client IP from SSR: pass the incoming `X-Forwarded-For` (set by Caddy) to the API. Then restrict uvicorn `--forwarded-allow-ips` to the caddy and web addresses only, not `*`. Alternatively, exempt the web container from the per-IP rule and rely on the per-principal limit, and add per-IP limiting at the edge (Caddy or proxy.ts) for `/app`. | high |
| F-4 | low | services/api/qai_api/routes/auth.py (signup, invite accept); no verification flow exists (SPEC Gap 12) | Email addresses are never verified. Anyone can create an account for an email they do not control, through signup or an invite with no email binding. Email-bound invites then trust that unverified address. | An attacker registers `victim@corp.com` first. The victim cannot sign up (409). An admin who later invites `victim@corp.com` with email binding hands the attacker-controlled account access to the workspace. | Add email verification before an account can accept email-bound invites, or document this as an accepted risk and let admins see an "unverified" flag. | high (the behaviour is by design; the impact depends on the deploy) |
| F-5 | low | services/api/qai_api/worker/loop.py (child spawn); compose.yaml (worker on the `backend` network) | The job child process runs with the same UID (10001) as the worker parent and on the same network namespace. Its environment is scrubbed, but it could read `/proc/<ppid>/environ`, which holds `DATABASE_URL` with the app password, and it can reach db and Valkey (no AUTH) on the backend network. | This applies only if an attacker achieves code execution inside the child, for example through a future numpy/scipy parser bug. They could then read the DB credentials and talk to Postgres as `qai_app`, which can set `app.system` and bypass RLS, or tamper with Valkey. I found no such path today: only server-built JSON (a validated circuit and bounded params) reaches the child. This is defence in depth. | Run the child under a separate UID with no access to the parent's /proc. Alternatively, `unshare -n` or seccomp the child (no sockets). Set `PR_SET_DUMPABLE 0` in the parent. Add `requirepass` or ACLs to Valkey. | medium |
| F-6 | low | services/api/qai_api/db/rls.py (policy predicate `... OR current_setting('app.system', true) = 'on'`) | The RLS bypass is a GUC that the NOBYPASSRLS app role can set itself. Tenant isolation therefore holds only as long as no SQL injection exists. I found none: all queries are bound parameters, and the single f-string SQL at repositories/projects.py:133 uses a fixed allow-list. | If any future injection appears, `SET app.system='on'` defeats RLS for every table. | Optional hardening: give the worker and scheduler a separate DB role for system scope, gate the bypass on `current_user` rather than a GUC, or use a SECURITY DEFINER helper. | high (the design weakness is confirmed; no exploit exists today) |
| F-7 | low | services/api/qai_api/logging.py:63-64 (`format_exc_info` runs before `redact`) | Exception tracebacks are rendered to a string before redaction. Key-based redaction does not apply inside the `exception` string; only the email regex runs on it. A driver error that includes a DSN or a token value in its message would be logged in clear. | A DB connection error at runtime logs a message that contains the connection string, including the password, to stdout or log aggregation. | Add a value-pattern scrub (e.g. `postgres(ql)?://[^@]+@`, `qai_[A-Za-z0-9]{8}_\S+`, `Bearer \S+`) applied to the rendered `exception` field after `format_exc_info`. | medium |
| F-8 | low | apps/web/package.json:40 (`postcss` 8.5.6, dev) | Direct dev dependency postcss ≤8.5.22 carries GHSA-6g55-p6wh-862q (high, arbitrary file read via `sourceMappingURL`), GHSA-fxqj-rqcc-2cmp and GHSA-qx2v-qp2m-jg93. postcss is used only at build time, through `@tailwindcss/postcss`, on repo-owned CSS. The Next standalone runtime image does not run it. | Exploitation needs attacker-controlled CSS fed to the build, for example from a malicious PR. CI runs on `pull_request` with `contents: read` and no secrets in that job, so the impact is limited to the CI runner. | Bump to `postcss@8.5.32` (non-major; npm reports a fix is available). | high |
| F-9 | low | services/api/Dockerfile:12; services/api/pyproject.toml | Direct Python deps are pinned (`==`), but there is no lock file or hashes. Transitive deps (starlette, anyio, h11, etc.) resolve at image build time, so a malicious or vulnerable transitive release lands silently, and pip-audit cannot fully cover the tree. | A compromised or vulnerable transitive package version is published. The next image build picks it up with no review. | Generate a hashed lock (`uv pip compile --generate-hashes` or `pip-compile`), install it with `--require-hashes`, and run `pip-audit -r` on it in CI. | high |
| F-10 | info | compose.yaml (api `--forwarded-allow-ips "*"`) | uvicorn trusts `X-Forwarded-For` from any peer. Caddy overwrites the header from untrusted clients, so external spoofing is not possible today. Anything on the frontend network (web) could still set an arbitrary client IP. | A compromised web container rotates spoofed IPs to evade the per-IP login and signup limits. | Restrict the list to the caddy and web addresses, or a fixed subnet (see the F-3 fix). | high |
| F-11 | info | .github/workflows/ci.yml:34,35,57,58,73,79 | Actions are pinned by tag (`@v5`, `@v6`), not by commit SHA. CI has no dependency-audit or secret-scan step. | A retagged upstream action could run in CI; the risk is limited by `contents: read`. | Pin actions to SHAs and add `npm audit --omit=dev`, `pip-audit` and gitleaks steps. | high |
| F-12 | info | root/apps devDependencies (eslint, typescript-eslint, eslint-plugin-jsx-a11y, @next/eslint-plugin-next, lighthouse → @sentry/node → glob/minimatch) | 17 moderate dev advisories, all ReDoS or DoS in minimatch, glob cancellation, and the eslint chain. They are reachable only from lint and Lighthouse tooling on trusted glob patterns and are absent from the production images (`npm audit --omit=dev` = 0). | None realistic; the tooling processes only repo-controlled patterns. | Update when non-breaking fixes ship; no action required for release. | high |

Counts: critical 0, high 0, medium 3, low 6, info 3.

## Areas checked with no finding

- **Authn and sessions.** Session and CSRF tokens use `token_urlsafe(32)` and are stored as SHA-256 hashes. The timeouts are 12 h idle and 7 d absolute. Logging in drops the presented session (fixation guard). A password change or reset revokes all sessions; a role change rotates or revokes them. Login is constant-time via a dummy Argon2 hash and returns a generic 401.
- **Cookies.** `__Host-qai_session` is HttpOnly. Both it and `__Host-qai_csrf` are Secure, SameSite=Lax and Path=/.
- **CSRF.** The double-submit token is bound to the session row (hash, `compare_digest`). Origin must equal PUBLIC_ORIGIN on every unsafe cookie-authenticated method. Signed-in invite acceptance requires CSRF.
- **Tenant isolation (app level).** Every route resolves its scope from the principal, never from client input. Every repository select, update and delete filters on `tenant_id`; I grepped the full repository layer. Child-entity creates verify the parent inside the tenant: circuits→project, jobs→project and circuit plus the circuit-project match, schedules→project, evolution→project, releases→project. Evidence checks `ev.tenant_id == row.tenant_id and project_id` match. Cross-tenant ids return 404. Job completion and failure run in `tenant_session(job.tenant_id)`.
- **Tenant isolation (DB level).** RLS is enabled and forced on all tenant tables (migrations 0001–0008). The app role is NOBYPASSRLS. The GUCs are transaction-local (`set_config(..., true)`). The invite and api_key lookup policies are SELECT-only and keyed on hash and prefix GUCs. UPDATE and DELETE are revoked on audit_events and evidence_records (tamper resistance).
- **Authorization per role.** require_role ranks are viewer < member < admin < owner. Session-only gates (`require_session*`) protect approve, release, project delete and schedules. `require_user_admin` blocks API keys from managing members, invites and keys. The last-owner lock holds under row locks. Self-approval is blocked in governance.py:378 and by the DB CHECK, and an API key's author is its creator (no approval by proxy). The scheduler force trigger is owner-only, session-only, and returns 404 unless `SCHEDULER_TRIGGER_ENABLED`.
- **API keys.** The format is `qai_<prefix>_<secret>`, and only the hash is stored. The comparison is constant time. The key's role is capped at both the creator's current membership role and admin. A key returns 401 if its creator left or was disabled, and removing a member revokes the keys they created.
- **Rate limits.** The auth endpoints (login per IP+email and per IP, signup, reset, invite accept) fail closed. Job submits are cost-weighted per tenant. Gaps are listed only under F-1 and F-3.
- **Input validation and size caps.**
  - Caddy `max_size` is 1 MB, and the API BodyLimitMiddleware is 256 KiB, checked on both declared and streamed length.
  - Pydantic models are strict with `extra=forbid`.
  - Circuit caps: 20 qubits, 2000 ops, a gate allow-list, finite params with |θ| ≤ 1e6.
  - All job and evolution params are bounded: iterations ≤ 20, seeds 5–30, maxiter ≤ 500, p ≤ 5. Evolution scores are server-computed (S14).
  - Query strings are length-capped, and cursors are bounded at 200 characters.
  - The LIKE search is escaped.
- **Worker sandbox (except F-5).**
  - The child gets JSON over stdin and stdout, with no pickle.
  - Resource limits: RLIMIT_AS = JOB_MEMORY_MB, RLIMIT_CPU = timeout+1, RLIMIT_CORE = 0, a wall-clock kill, input capped at 8 MiB and output at 64 MiB.
  - The environment is minimal and has no DB or Valkey credentials.
  - The fault-injection hook `_fault` works only with `QAI_WORKER_FAULTS=1`.
  - The worker container is read-only with tmpfs /tmp, `no-new-privileges`, `cap_drop: ALL` and a 2 GB memory cap.
- **Secrets handling.**
  - The required secrets fail fast (`${VAR:?}`).
  - config.py validates the URL shapes and exits on error.
  - `.env` is git-ignored.
  - Only Caddy publishes ports, and the backend network is internal.
  - The containers run as non-root (uid 10001; node for web) under tini.
- **Headers and CSP.**
  - Web (proxy.ts): a fresh nonce on every request and `strict-dynamic`, with no `unsafe-inline`. `unsafe-eval` is allowed only in development. The policy also sets `object-src 'none'`, `frame-ancestors 'none'`, `base-uri 'self'` and `form-action 'self'`. Redirects get the most restrictive CSP.
  - Headers on both web and API: nosniff, Referrer-Policy strict-origin-when-cross-origin, Permissions-Policy, X-Frame-Options DENY and COOP.
  - API only: `default-src 'none'` and `Cache-Control: no-store`. The docs CSP is relaxed only for `/api/v1/docs`, and the docs are disabled in production.
  - Caddy strips the `Server` header.
  - HSTS is gated by an environment variable (ADR-29).
- **Credential forms.** Login, signup, reset-password and invite-accept all declare `method="post"`, so no password can end up in the URL if JS fails.
- **Open redirect.** `safeNext` and `loginPath` reject `//` and `/\`, and allow only relative targets (unit-tested).
- **SSRF and outbound URLs.** The API and worker make no outbound HTTP requests. The only server-side fetch in the web app targets a fixed `API_INTERNAL_URL` from the environment, and request paths are app-constructed. No user-controlled URL is fetched anywhere.
- **Injection.**
  - All SQL uses SQLAlchemy with bound parameters; the one f-string table name comes from a fixed allow-list (projects.py:119,133).
  - Ruff S reports nothing.
  - No `dangerouslySetInnerHTML` in apps/web.
  - CSV export neutralises formula triggers (`= + - @ \t \r`).
  - Content-Disposition filenames are sanitised to `[A-Za-z0-9-_.]`, and the QASM download filename is sanitised too.
  - X-Request-ID must match `^[A-Za-z0-9-]{8,64}$`.
- **Logging of secrets (except F-7).**
  - Keys matching `password|token|secret|authorization|cookie|key|email|csrf` are redacted recursively, and email addresses are scrubbed from values.
  - The access log records the route template rather than the raw path, so invite and reset tokens do not leak into logs.
  - The uvicorn access log is disabled.
- **Error handling.** Unhandled exceptions return a generic problem+json with a request id. Stack traces are logged, never returned.
- **Password reset token.** The claim is single-use and atomic (a single UPDATE on `used_at IS NULL AND expires_at > now`), stored as a hash, valid 24 h, and issuing a new one supersedes the old. The cross-tenant issue is covered in F-2.
- **Scheduler.** Double execution is prevented by a unique claim on the run window. A schedule's params go through the same `validate_params` as interactive jobs. The limit is 25 schedules per tenant.
- **Production npm dependencies.** `npm audit --omit=dev` reports 0. **Python direct dependencies:** pip-audit is clean.
- **CI.** It runs on `pull_request` (not `pull_request_target`) with `permissions: contents: read` and no secrets exposed to PR code.
