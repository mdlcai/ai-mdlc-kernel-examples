# SECURITY-AUDIT.md — Ember (comprehensive depth)

Formal adversarial second pass vs SPEC §4 (SEC-1…SEC-13) + ARCHITECTURE §9 invariants.
3-pass procedure: **initial scan → auto-remediate → re-scan.** Date: 2026-06-07. Build 0.1.0 → 0.1.1.

## Scan categories covered
Dependency vulnerability (`npm audit`), SAST (manual review of auth, webhook, stock, query-scoping,
CSRF, error mapper), secret detection (INV-7 grep — clean), configuration/infra (CSP, HSTS, cookie
flags, TLS), env-var audit (`.env.example` complete vs SPEC §8), license compliance (Apache-2.0).

## Pass 1 — Initial scan (findings)
| # | Severity | Clause | Finding | Exploit path |
|---|----------|--------|---------|--------------|
| F1 | **MEDIUM** | SEC-7/12, R6, INV-8 | `loginRateLimit`/`uploadRateLimit` custom keyGenerator used `req.ip` without `ipKeyGenerator` → IPv6 normalization missing (`ERR_ERL_KEY_GEN_IPV6`). | An IPv6 client rotates the low bits of its /64 to get a fresh key per request, bypassing the per-IP brute-force limit on `/auth/login`. |
| F2 | **MEDIUM** | BUILD header-buffer; availability | Web (Next.js) tier ran with Node's default 16 KB header limit → a ≥16 KB `Cookie:` returned **431**, while the API tier (started with `--max-http-header-size=32768`) returned 200. | Accumulated/oversized cookies make the storefront unreachable (431) for affected users; inconsistent with the resilience requirement. |
| F3 | **LOW** | robustness | Redis client `enableOfflineQueue:false` caused `RedisStore.init` to throw a startup race before the socket was ready (noisy error; risk of first-request limiter miss). | Narrow window at boot where a limiter store isn't ready; self-heals but logs errors. |
| F4 | **LOW** | SEC-10 / A03 supply chain | `npm audit` (prod): 2 moderate advisories on **postcss** pulled transitively by **next** (CSS-stringify XSS, build-time only). Fix requires a breaking `next` downgrade. | Build-time CSS stringification only; not reachable in Ember's runtime SSR output. |

No CRITICAL or HIGH findings. SAST review confirmed: webhook signature-verify-before-process (INV-6),
stock atomic decrement + `CHECK(stock>=0)` (racing test green), per-user query scoping → 404 on
cross-tenant (tested), uniform auth envelope (smoke-verified), CSRF double-submit + Origin check,
RFC 9457 errors leak no internals, no card-data identifiers (INV-1), no secret literals (INV-7).

## Pass 2 — Auto-remediate
- **F1 FIXED:** `apps/api/src/middleware/rateLimit.ts` — import + apply `ipKeyGenerator(req.ip)` in the
  `(ip,email)` login key and the upload key. Startup `ERR_ERL_KEY_GEN_IPV6` validation error gone.
- **F2 FIXED:** web started with `NODE_OPTIONS=--max-http-header-size=32768`; added the same to the
  `web` service env in `compose.yaml`. Large-cookie on `/` now returns 200.
- **F3 FIXED:** `apps/api/src/redis.ts` — `enableOfflineQueue:true` so boot-time store commands queue
  until connected. Clean startup, no errors.
- **F4 DEFERRED — LOW:** build-time-only postcss advisory via next; auto-fix is a breaking next
  downgrade. Tracked; revisit on next minor upgrade. Not runtime-exploitable.
- VERSION patch bump 0.1.0 → 0.1.1.

## Pass 3 — Re-scan (residual)
| # | Status |
|---|--------|
| F1 | RESOLVED — no IPv6 key-gen warning; limiter normalizes IPv6. |
| F2 | RESOLVED — large-cookie (16 KB) on `/` and `/api/health` → 200 (smoke-verified). |
| F3 | RESOLVED — clean api startup, no store-init error. |
| F4 | RESIDUAL **LOW** — deferred with tracking (build-time postcss/next). |

**Residual: 0 CRITICAL · 0 HIGH · 0 MEDIUM · 1 LOW (deferred).** Full test suite green (39 passing).
**Gate passes** (zero CRITICAL/HIGH residual; every MEDIUM dispositioned FIXED; LOW deferred w/ note).
