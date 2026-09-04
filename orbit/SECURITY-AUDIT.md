# Orbit — security audit

Adversarial second-pass review against `SPEC.md` §4 (SEC-1..SEC-23) and `ARCHITECTURE.md` §9, run at
`build_depth: comprehensive` after Stage 3. Three passes: scan, auto-remediate, re-scan. Everything below was
checked against the running production stack at `https://localhost:8443`, not only by reading source.

**Result:** no CRITICAL or HIGH findings. Four MEDIUM/LOW findings, all remediated. Two dev-only dependency
advisories accepted with rationale.

---

## 1. Trust boundaries

Orbit has five places where data crosses from something less trusted to something more.

**The browser to the API.** Every request arrives at Caddy over TLS and is proxied to Express. The boundary is
enforced in one middleware chain (`apps/api/src/app.ts`), in a fixed order: request id, pino logging, helmet,
a 256 KB body limit, cookie parsing, the Origin/Referer CSRF guard, rate limiting, session authentication. Body,
query and route params are parsed by zod strict objects, so an unknown key is a 400 rather than a silently
ignored field — this is what closes mass assignment. Verified at runtime: a mutating request with no `Origin`
header and one with a foreign `Origin` are both refused 403; the same request with the correct origin proceeds
to validation.

**One tenant to another.** The hard boundary. Every query that touches tenant data runs inside `withOrgScope`,
which opens a transaction and sets `app.org_id`; PostgreSQL row-level security is `ENABLE`d **and `FORCE`**d on
all 18 tenant tables with a policy of `org_id = NULLIF(current_setting('app.org_id', true), '')::uuid`. The
`NULLIF` is what makes it fail closed: if the setting is missing or was reset, the comparison is against NULL and
no row matches, rather than every row matching. The application role is created `NOBYPASSRLS NOSUPERUSER`, so
even a query that forgets its predicate cannot cross the line. Background jobs use `forEachOrg`, which runs the
same scoped wrapper once per org, so a scheduler sweep is never a hole in the model. Following this audit, every
update and delete on a tenant table also names `org_id` in its `WHERE`, so a mistake has to defeat two
independent barriers.

**One user to another inside the same tenant.** RLS does not help here, and this is where multi-tenant products
usually leak. Every `/:id` route and every sub-resource route (`/:id/comments`, `/:id/timeline`,
`/:id/approvals`) opens with a `load<Entity>` middleware that applies org scope *and* row ownership before the
handler runs; a requester asking for someone else's ticket gets 404, not 403, so existence never leaks. INV-14
enforces the pattern mechanically across 384 files, so a newly added sub-resource that forgets the guard fails
the lint rather than shipping. Staff-only fields — private comments, the audit trail, other people's email
addresses — are filtered in the query and in the SSE fan-out, not hidden in the UI.

**The system to the outside world.** Two egress paths: SMTP, and the HIBP range API for password screening.
Both are configuration, not user-supplied URLs; there is no fetch-by-URL, webhook-send or link-preview feature,
so there is no SSRF surface and no chokepoint to enforce. HIBP is queried by k-anonymity — the first five
characters of the SHA-1 hash — so a candidate password never leaves the process.

**Time and the scheduler.** The in-process scheduler holds a PostgreSQL advisory lock, and claims work with
`FOR UPDATE SKIP LOCKED`, so two API replicas cannot double-send an email or double-breach an SLA.

## 2. Data-flow risks

**Credentials.** Argon2id at m=19456, t=2, p=1. A login for an address that does not exist still performs a
dummy verify, so the response time does not distinguish the two cases; the rate limiter fires *before* the hash
comparison, which is the ordering that matters at network scale (a limiter after the hash is a CPU exhaustion
vector). Verified at runtime: an unknown address and a wrong password return byte-identical envelopes apart from
the request id.

**Sessions.** Opaque 256-bit tokens, stored as HMAC-SHA256 so the database never holds anything replayable, in a
`__Host-orbit_session` cookie with `HttpOnly; Secure; SameSite=Strict; Path=/`. No token in `localStorage`.
Rotated on password change and on role change, with every other session revoked. Measured by the web baseline's
cookie check on the real deploy.

**Reset and invite tokens.** Single-use, hashed at rest, one-hour TTL. A spent or invented token returns the
same message as an expired one, and the reset revokes every session for that user.

**Stored content.** Knowledge-base bodies are author-supplied Markdown. The API stores them verbatim and never
renders HTML; the client renders with markdown-it configured `html: false` — so raw HTML cannot survive the
parser — and then passes the result through DOMPurify. Two independent barriers, and the CSP is the third.

**The audit trail.** Append-only by construction: the table carries SELECT and INSERT policies only, and no code
path issues an update or delete against it. Before/after snapshots never contain `passwordHash`.

**Personal data in logs.** pino redacts thirteen keys (authorization, cookie, password, token, secret and
relatives) at every depth, so a request body in a log line cannot carry a credential.

## 3. Authentication and authorization edge cases

These are the cases that do not follow from the happy path and were checked individually.

- **A deactivated user with a live session.** Authentication reloads the user on every request; a deactivated
  account is rejected on its next request rather than surviving until its cookie expires.
- **Role change mid-session.** Sessions are revoked on role change, so a demoted admin cannot keep acting with
  the old role from an open tab.
- **Self-modification.** An admin cannot remove their own admin role or deactivate themselves; the server
  enforces it (`SELF_MODIFICATION`) and the UI disables the controls, so the last admin cannot be locked out.
- **Approving your own request.** Refused with `SELF_APPROVAL`; the approver must differ from the requester.
- **Deciding an approval twice.** The decision updates `WHERE status = 'pending'` and returns the row, so a
  concurrent second decision affects no rows and answers `ALREADY_DECIDED` rather than overwriting.
- **A role that may not use a route.** Answers 404, not 403, so an agent cannot map which admin routes exist.
- **Optimistic concurrency.** A versioned mutation without `If-Match` is 428; with a stale one, 409. Two
  concurrent edits cannot silently overwrite each other.
- **SSE.** The stream is behind authentication, clients are keyed by org, per-user events are addressed, and the
  sixth concurrent stream for one user is refused 429 — so an open browser cannot be used to exhaust
  file descriptors.
- **Automation.** Rules run with a depth cap; a rule whose action re-triggers its own trigger stops at three
  levels and the stop is recorded. Actions run in the triggering transaction, so a failed rule cannot half-apply.

## 4. Findings and disposition

| # | Severity | Finding | Disposition |
|---|---|---|---|
| 1 | MEDIUM | `scripts/load-test.mjs` set `NODE_TLS_REJECT_UNAUTHORIZED='0'` and passed `rejectUnauthorized: false` to autocannon, disabling certificate verification for the whole process to reach the local deploy. Guarded to localhost, but it is precisely the shortcut that outlives the script it was added for. | **FIXED.** The script now requires `NODE_EXTRA_CA_CERTS` and prints the two commands that produce it; autocannon is handed the CA. Certificate verification is never disabled. |
| 2 | MEDIUM | ~20 updates and deletes on RLS-forced tenant tables filtered by id alone, leaving forced RLS as the only barrier where `ARCHITECTURE.md` §4.1 asks for two. | **FIXED.** Every tenant write now names `org_id` in its `WHERE`. Verified: zero remain. |
| 3 | LOW | `loadNotification` was the only entity loader without a principal guard, so an anonymous `PATCH /api/v1/notifications/<uuid>` dereferenced an absent principal and answered 500 where the contract says 401. Not an authorization hole — an unauthenticated request could not reach data — but a 500 on an anonymous request is an information-free failure and a monitoring false positive. | **FIXED.** The guard matches its siblings. |
| 4 | LOW | `csrfOriginGuard` was exported but never mounted (the app builds its own guard from the configured origin). A second, differently configured guard sitting in the module is a trap for whoever mounts it next. | **FIXED.** Removed; the runtime guard is unchanged and verified. |
| 5 | LOW | `.env.example` omitted `PORT` and the load-test variables, so the documented environment was incomplete against the boot schema. | **FIXED.** The example now matches the schema exactly. |

**Accepted, with rationale:** `npm audit` reports zero advisories in the production dependency tree and seven
moderate advisories in dev-only tooling — `esbuild` under `drizzle-kit` (migration authoring) and `uuid` under
`autocannon` (the load-test tool). Neither is shipped in a container or reachable at runtime; both fixes are
major-version downgrades of the tools themselves, which would be a larger regression than the advisories.
Tracked in `REPORT.md` for the next dependency review.

## 5. Scan coverage and tooling

| Category | Tool | Result |
|---|---|---|
| Dependency vulnerabilities | `npm audit` (production and full trees) | 0 in production; 7 moderate dev-only, accepted above |
| Secret detection | pattern sweep for key material and credential assignments across source, docs, compose and CI | none; the only non-production default is the dev session secret, guarded so production cannot boot without a real one |
| SAST-style review | pattern sweep plus manual review of every `sql.raw`, `dangerouslySetInnerHTML` and dynamic execution site | `sql.raw` is only ever passed a literal chosen by the code, never user input; the two `dangerouslySetInnerHTML` sites are the nonce'd theme bootstrap and the sanitized Markdown renderer |
| Configuration, TLS and headers | measured on the running deploy by `scripts/web-baseline.mjs` (`site.headers`, `site.cookies`) plus direct inspection | HTTPS only with an HTTP→HTTPS redirect, HSTS `max-age=63072000; includeSubDomains; preload`, CSP Profile N (nonce + `strict-dynamic`, `object-src 'none'`, `frame-ancestors 'none'`), `nosniff`, `Referrer-Policy`, `Permissions-Policy`; zero `securitypolicyviolation` events across 168 screen/viewport/theme measurements |
| Environment audit | schema-versus-example diff | no gaps after finding 5 |
| License compliance | not flagged in `RESEARCH.md` | not run |

**Tooling note — ADVISORY.** No external SAST or secret-scanning tool (Semgrep, Gitleaks, Trivy) is installed in
this environment, so those two categories were covered by pattern sweeps and manual review rather than by a
scanner. Agent self-review is not a substitute for automated scanning. Recommended for the next run: add
Semgrep and Gitleaks to `.github/workflows/ci.yml`.

## 6. Residual posture

No CRITICAL or HIGH findings, before or after remediation. Every MEDIUM is fixed. The standing items a reader
should know about:

1. **Dev-only dependency advisories** (above) — accepted, tracked.
2. **No automated SAST or secret scanner** in CI — advisory, recommended above.
3. **`change.submitted` automation rules only reach a change that is linked to a ticket**, because rule
   conditions and actions are ticket-shaped. Not a security issue; recorded so nobody assumes wider coverage.
