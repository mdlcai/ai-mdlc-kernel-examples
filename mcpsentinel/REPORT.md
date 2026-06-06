# REPORT.md — MCP Sentinel build report

## Pre-build assumptions (one-liner)
Blank/derived fields resolved by ADR: runtime=Node LTS (ADR-001), ORM=Drizzle
(ADR-002), engine=deterministic (ADR-003), sessions=cookie (ADR-007),
hashing=argon2id+HIBP (ADR-008). `domain_signals: has_webhooks` Deferred (ADR-010,
no inbound webhook surface in v1).

## Build Input Reconciliation
Every non-blank input field is honored provably. 16 non-blank fields → 16 rows.
Disposition ∈ {Applied, Deferred, Conflict}.

| # | Field | Value | Disposition | Evidence / rationale |
|---|-------|-------|-------------|----------------------|
| 1 | protocol_support | HTTPS only | Applied | ARCHITECTURE §12 (TLS at proxy, HTTP→HTTPS redirect, HSTS); QUICKSTART §Protocol & TLS. Target-scheme nuance in ADR-004. |
| 2 | database_preference | PostgreSQL | Applied | ARCHITECTURE §2/§7 (Drizzle→Postgres 16); docker-compose `db` service. |
| 3 | security_baseline | OWASP Top 10 | Applied | ARCHITECTURE §10 + COMPLIANCE.md maps controls; A10 SSRF = INV-1/INV-2; OWASP MCP Top 10 mapping in RESEARCH §5. |
| 4 | rate_limiting | true | Applied | ARCHITECTURE §5 middleware/rateLimit; INV-12; per-IP + per-user + cost-based scan limit. |
| 5 | audit_logging | true | Applied | ARCHITECTURE §7 `audit_log` table + §12 structured logs. |
| 6 | secrets_management | environment variables | Applied | ARCHITECTURE §12; `.env.example` enumerates every var; no secrets in code. |
| 7 | frontend_framework | React | Applied | ARCHITECTURE §6 (React 18 + Vite + TS). |
| 8 | backend_framework | Hono | Applied | ARCHITECTURE §2/§5 (Hono on Node). |
| 9 | api_style | REST | Applied | ARCHITECTURE §2; SPEC §API surface = REST/JSON, problem+json errors. |
| 10 | performance_requirements | first scan < 10s | Applied | ARCHITECTURE §4 caps (10s total deadline); SPEC perf targets; smoke load test. |
| 11 | testing_strategy | test-after | Applied | ADR-011; tests written after impl within each feature slice. |
| 12 | logging_format | structured JSON | Applied | ARCHITECTURE §5 lib/logger (pino JSON + redaction). |
| 13 | scale | small — under 1k users | Applied | ADR-007 (cookie sessions), single-node docker-compose; no premature scale-out. |
| 14 | target_platforms | web | Applied | ARCHITECTURE §6 (responsive web SPA). |
| 15 | mdlc_attribution | structural | Applied | Stage 4: NOTICE file + README "Built with" + package keywords (no runtime branding). |
| 16 | domain_signals | has_webhooks | Deferred | ADR-010 — no inbound webhook surface in v1; webhook risk-knowledge applied to rug-pull/`tools.listChanged` detection instead. |

Conflicts: **none unresolved.** Non-blank field count (16) = row count (16). ✓

## Interface Contract Validation (cadence N = max(1, min(7, ceil(8/4))) = 2; final run after F-07)
- **API endpoint coverage:** all 16 SPEC §5 endpoints implemented (auth ×4, scans
  ×6 incl. rescan/share/export, endpoints ×3, share ×1, health ×1, export ×1). ✓
- **Frontend↔backend contract:** shared zod schemas/DTOs in `@sentinel/shared`
  consumed on both sides; URL/method/payload/response shapes match. ✓
- **UI reverse coverage:** every non-internal endpoint reachable from a screen;
  all 13 SPEC §UI Surface screens routed; W1–W5 completable through UI to terminal
  step (`ui-coverage` invariant INV-11 PASS). ✓
- **Auth middleware wiring:** `loadUser`+`requireAuth` on /api/scans + /api/endpoints;
  CSRF on every mutating verb. ✓
- **Database schema alignment:** Drizzle schema ↔ migrations ↔ DTOs consistent;
  enums mirrored in `@sentinel/shared`. ✓
- **Env var completeness:** every `process.env` ref has a `.env.example` row. ✓

## Cascade Impacts
None. ADR-012/013/014/015 are equivalence-preserving substitutions (same
guarantees, no RESEARCH deliverable downgraded) — no cascade to unbuilt features.

## Pre-Delivery Completeness Gate
- Route & handler completeness ✓ · Data layer (migrations + reversible down + no
  seed needed; first-run is register) ✓ · Validation (zod front+back, field-level
  errors) ✓ · CRUD completeness (scan create/list/detail; endpoint delete) ✓ ·
  Auth flow e2e (register/login/logout/session, protected routes reject 401) ✓ ·
  UI Surface completeness (no stub/JSON-in-`<pre>`; empty/loading/error/success
  designed; workflows terminal) ✓ · Environment & startup completeness ✓.

## Verification Gate (exit-code discipline — all exit 0)
- Clean install (`npm install`) ✓
- Typecheck (`tsc --noEmit` × shared/api/web) ✓
- Lint (`eslint . --max-warnings 0`) ✓ (0 errors, 0 warnings)
- Unit + integration tests (`vitest run`) ✓ — **37/37 passing**
- Production build (`npm run build` × 3) ✓
- Invariant lint (`node scripts/invariant-lint.mjs`) ✓ — **10/10 machine-checks
  PASS, 4 manual** (INV-4/9/12/13 audited below)

## Reviewer Gate
Independent fresh-context review (general-purpose subagent, fed only RESEARCH/
ARCHITECTURE/SPEC + source). **Verdict: PASS** — no correctness-class (Items 1–6)
failures; Item 7 (Runnable) PASS (typecheck + 37 tests green, confirmed by reviewer).
- Item 1 Genuine implementation: PASS — detectors are generic (entropy >3.5
  bits/char, phrase lists, hidden-Unicode, real hash-diff), not fixture-keyed.
- Items 2–6: PASS with code-level evidence (API conformance, data model, enforced
  SSRF/token/tenant/CSRF/rate-limit/no-LLM invariants, UI coverage, designed states).
- 3 non-blocking advisories dispositioned in DECISIONS ADR-016.

## Smoke Test Results
Functional Smoke Test against the running deploy (`http://localhost:8787`, docker
local stack) — **17/17 flows PASS** (`smoke-test.log`). Covers every Key Workflow
end-to-end: W1 scan(bad)→grade F/8 findings + scan(good)→grade A/0 (no false
pos/neg), W2 evidence+redaction, W3 rescan, W4 history, W5 share public-read; plus
auth bootstrap, unauth→401, tenant isolation→404, SSRF-blocked (metadata +
RFC1918), large-header (16 KB cookie)→200, invalid-url→422. UI visually verified
via headless browser (landing + authenticated report screen render correctly).

**Root causes found & fixed during ship (re-entered Stage 3):**
1. Hono `onError` mounting boundary — sub-router AppErrors were swallowed as 500;
   fixed by registering `onErrorHandler` on every Hono instance.
2. undici pinned-dispatcher `connect.lookup` must honor `{ all: true }` (array
   callback) — hostname targets failed `UNREACHABLE`; added regression test.
3. Design/CSP: `@import tokens.css` was after `@tailwind` (dropped → washed-out
   UI); CSP `font-src`/`script-src` blocked Vite-inlined fonts + theme script.
   Fixed import order + CSP (`data:` fonts, inline-script hash).

## Startup verification (Stage 4)
Built artifacts; copied `apps/web/dist` → `apps/api/public`; ran
`node --max-http-header-size=32768 apps/api/dist/server.js` against a fresh DB:
- Process started, migrations applied, no errors.
- `GET /api/health` → 200 `{"status":"ok","db":"up"}`.
- `GET /` → 200 text/html (SPA `index.html`); `GET /app/scans/x` → 200 (SPA
  fallback); `GET /assets/*.js` → 200 text/javascript.
- Fixed a cwd-relative static-serving bug found here (server.ts derives the public
  dir relative to `process.cwd()` so it works under Docker `/app`, npm workspace,
  and repo root).

## Deploy Target Reachability
- **Outcome C — local-stack deploy.** No cloud target is configured for this build;
  the deploy target is the bundled `docker-compose.yml` (db + app). Docker daemon
  verified UP (`docker ps` exit 0). The only external is HaveIBeenPwned (outbound
  HTTPS, no key/secret — `HIBP_ENABLED` toggle). Postgres ships in the stack. No
  placeholder `.env` externals block the local deploy. → ship to `docker compose`
  and smoke-test against `http://localhost:8787`.

## Invariant lint result
- Machine-checkable: **10 evaluated / 10 passing / 0 failing** (INV-1,2,3,5,6,7,8,10,11,14).
- Manual (prose-audited):
  - **INV-4** — `lib/logger.ts` pino `redact` covers authToken/token/password/secret/
    authorization/cookie; no `logger.*` call interpolates a raw token. ✓
  - **INV-9** — every detector in `scanner/rules/*` uses `matcher.ts` (substring/
    indexOf/charscan), caps input at 32 KB; no backtracking RegExp on MCP strings;
    a ReDoS-fixture test asserts <200 ms on a 50 K adversarial string. ✓
  - **INV-12** — `rateLimit` wired on /api/auth/* (key=ip) and /api/scans (key=user)
    + concurrency cap; auth limiter runs before the scrypt compare. ✓
  - **INV-13** — `csrf` guards every POST/PUT/PATCH/DELETE (Origin match + double-
    submit token). ✓

## Security baseline summary
OWASP Top 10 — controls tracked in COMPLIANCE.md; A10 SSRF mitigated by INV-1/INV-2
(SafeConnector). Must reconcile with COMPLIANCE.md (no ⚠/❌ rolled up into "Applied").
