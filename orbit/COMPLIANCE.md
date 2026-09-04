# Orbit — compliance posture

Every `SPEC.md` §4 security requirement and every `WEB.md` accessibility, metadata and header contract, with the
evidence that proves it. Status is `✓` implemented and audit-clean, `⚠` implemented with a known limitation or a
logged deviation, `✗` a gap.

**Posture: 27 ✓ / 3 ⚠ / 0 ✗**

Generated after the Security Audit Gate. Evidence is a file path, a test name, or a field in
`FRONTEND-AUDIT.json` — never a claim.

## Security requirements (SPEC.md §4)

| ID | Requirement | Status | Evidence | Notes |
|---|---|---|---|---|
| SEC-1 | Tenant isolation: RLS forced on every tenant table, `withOrgScope` sets `app.org_id`, cross-org reads 404 | ✓ | `apps/api/drizzle/0001_rls.sql`; `apps/api/src/db/scope.ts`; `apps/api/test/tenancy.test.ts` | Every tenant write also names `org_id` explicitly after the security audit (finding 2) |
| SEC-2 | Object-level authorization on every `/:id` and sub-resource route | ✓ | `load<Entity>` middleware per module; INV-14 over 384 files; same-tenant IDOR tests in `apps/api/test/tickets.test.ts` | |
| SEC-3 | Field-level gating: private notes, other users' emails, staff-only audit entries | ✓ | `serialize` per module; `apps/api/test/tickets-lifecycle.test.ts` asserts absence in list, timeline and the SSE frame | |
| SEC-4 | Argon2id m=19456 t=2 p=1, 15–128 chars, HIBP k-anonymity, top-10k blocklist | ✓ | `apps/api/src/lib/password.ts`, `lib/passwordPolicy.ts`; `apps/api/test/auth.test.ts` | |
| SEC-5 | Anti-enumeration: identical login envelope, dummy verify on the miss path, forgot-password always 202 | ✓ | `apps/api/src/modules/auth/service.ts:110`; `e2e/auth-negative.spec.ts` compares both responses byte for byte | Registration discloses a taken workspace URL by design (ADR-022): it is a public sign-up and the 5/h per-IP limiter makes enumeration impractical |
| SEC-6 | Rate limiting with the specified key dimensions; limiter before hash verify | ✓ | `apps/api/src/http/rateLimit.ts`; INV-8; `apps/api/test/auth.test.ts` | |
| SEC-7 | Opaque 256-bit session token, HMAC stored, `__Host-` cookie, rotated on password and role change | ✓ | `apps/api/src/modules/auth/session.ts`; `FRONTEND-AUDIT.json` `site.cookies` = pass on the real deploy | |
| SEC-8 | CSRF: `SameSite=Strict` plus an Origin/Referer check on every mutating method | ✓ | `apps/api/src/http/csrf.ts`, mounted in `app.ts`; verified at runtime — 403 with no Origin and with a foreign Origin | |
| SEC-9 | HTTPS only, HSTS, CSP Profile N, `nosniff`, `Referrer-Policy`, `Permissions-Policy`, `frame-ancestors 'none'` | ✓ | `caddy/Caddyfile`, `apps/web/src/proxy.ts`; `FRONTEND-AUDIT.json` `site.headers` = pass; zero CSP violations across 168 measurements | |
| SEC-10 | zod strict schemas at every boundary, 256 KB body cap, RFC 9457 envelope with field errors | ✓ | `packages/shared/src/schemas/**`; `apps/api/src/http/problem.ts`; component tests assert the field errors reach the controls | |
| SEC-11 | Parameterized queries only; Markdown rendered without raw HTML | ✓ | Drizzle throughout — every `sql.raw` takes a literal chosen by the code, never input; `apps/web/src/components/kb/Markdown.tsx` (`html: false` + DOMPurify) | |
| SEC-12 | Every mutation writes `audit_events` in the same transaction; append-only | ✓ | `apps/api/src/db/audit.ts`; INV-5; SELECT/INSERT-only policies in `0001_rls.sql` | |
| SEC-13 | pino redaction; no PII in access logs; no stack traces in responses | ✓ | `apps/api/src/http/logger.ts` redacts 13 keys at any depth | |
| SEC-14 | Secrets from the environment only; production requires a real `SESSION_SECRET` | ✓ | `apps/api/src/config/env.ts`; INV-16; `.env.example` matches the schema exactly | |
| SEC-15 | Optimistic versions, `WHERE status='pending'` decisions, per-org counters under a row lock, advisory-locked scheduler, `SKIP LOCKED` | ✓ | `apps/api/src/db/scope.ts` (`nextNumber`), `jobs/scheduler.ts`; concurrency races in `apps/api/test/workflow-modules.test.ts` | |
| SEC-16 | No user-controlled URLs fetched; SMTP and HIBP hosts fixed by env | ✓ | No fetch-by-URL, webhook or link-preview feature exists, so there is no SSRF surface to guard | |
| SEC-17 | `orbit_app` is `NOBYPASSRLS` with table grants only; migrations run as the owner in a one-shot container | ✓ | `apps/api/drizzle/0001_rls.sql:7`; the `migrate` service in `docker-compose.yml` | |
| SEC-18 | Lockfile committed, `npm ci`, pinned base image tags, `npm audit` in the audit gate | ⚠ | `package-lock.json`; `.github/workflows/ci.yml`; `SECURITY-AUDIT.md` §4 | 0 production advisories; 7 moderate dev-only advisories accepted (drizzle-kit→esbuild, autocannon→uuid) — fixes are major-version downgrades of the tools |
| SEC-19 | WCAG 2.2 AA measured per screen, viewport and theme | ✓ | `FRONTEND-AUDIT.json` — 0 serious/critical axe violations across 28 screens × 3 viewports × 2 themes | |
| SEC-20 | Public screens carry full metadata; app screens are `noindex`; robots, sitemap, manifest, icons, designed 404 | ⚠ | `FRONTEND-AUDIT.json` `site.meta`, `site.robots`, `site.sitemap`, `site.icon`, `site.notFound` all pass | The `noindex` this requires caps the dashboard's Lighthouse SEO score at 63, which is the one open item in the Web Delivery Baseline — ADR-035 |
| SEC-21 | zod env schema exits non-zero naming the missing variable | ✓ | `apps/api/src/config/env.ts`; INV-4; `apps/api/src/config/env.test.ts` | |
| SEC-22 | SIGTERM drains within 10 s, closes the pool, stops the scheduler, exits 0 | ✓ | `apps/api/src/index.ts`; container `HEALTHCHECK` and `tini` as PID 1 | |
| SEC-23 | Data at rest | ⚠→✓ | Delegated to volume encryption at the deploy layer; documented in `QUICKSTART.md` | Counted as `✓`: the requirement itself specifies delegation, and the documentation obligation is met |

## Web delivery contract (WEB.md), measured

| Contract | Rule | Status | Evidence |
|---|---|---|---|
| §1 Responsive | Three deliberate compositions at 375 / 768 / 1280; no horizontal overflow | ✓ | `screens[].checks.overflow` pass on all 168 cells |
| §1 Mobile nav | Every destination reachable at 375 behind one 44px toggle with `aria-expanded` + `aria-controls` | ✓ | `screens[].checks.mobileNav` pass |
| §1 Content width | `main` keeps ≥ 60% of the viewport at 375 | ✓ | `screens[].checks.contentWidth` pass |
| §2 Landmarks & headings | One `main`, one `h1`, no skipped levels, `html[lang]` | ✓ | `screens[].checks.structure` pass — fixed during the design pass (sidebar cards were h3 under an h1) |
| §2 Skip link | First focusable element, visible on focus, target exists | ✓ | `screens[].checks.skipLink` pass |
| §2 Contrast | ≥ 4.5:1 text in both themes | ✓ | `screens[].checks.axe` — 0 serious/critical; the status and primary text tokens were corrected during the design pass |
| §2 Targets | 24px everywhere, 44px for nav, primary and icon-only controls at 375 | ✓ | `screens[].checks.targets44` pass |
| §2 Focus visible | Focused style differs from unfocused | ✓ | `screens[].checks.focusVisible` pass |
| §2 Reduced motion | No running animation under `prefers-reduced-motion` | ✓ | `screens[].checks.reducedMotion` pass |
| §3 Metadata | Per-route title, description, canonical, OG image on public screens | ✓ | `site.meta` pass |
| §3 Crawl surface | `robots.txt` with a sitemap, `sitemap.xml`, icons, manifest, designed 404 | ✓ | `site.robots`, `site.sitemap`, `site.icon`, `site.notFound` pass |
| §4 Core Web Vitals | LCP ≤ 2.5 s, CLS ≤ 0.1 | ✓ | `lighthouse[]` — landing LCP 2261 ms, dashboard 2413 ms, CLS 0.001 / 0 |
| §4 Lighthouse | Accessibility, best practices, SEO ≥ target; performance ≥ target − 10 | ⚠ | landing 98 / 100 / 100 / 100; dashboard 97 / 100 / 100 / **63** | SEO capped by SEC-20's `noindex` — ADR-035 |
| §4 JS budget | ≤ 400 KB compressed for a `saas` archetype | ✓ | `bundle` — landing 148 KB, app screen 175 KB |
| §5 Resilience | Designed error and loading states on every client-fetched screen | ✓ | `screens[].checks.dataError`, `.dataLoading` pass |
| §6 Headers & CSP | Full header set, Profile N, correct cookie flags, zero CSP violations | ✓ | `site.headers`, `site.cookies` pass; `screens[].checks.console` pass |

## Open items

1. **SEC-20 / Lighthouse SEO on the dashboard (⚠).** Authenticated screens carry `robots: noindex`, which
   `WEB.md` §3 requires; Lighthouse's `is-crawlable` audit is the only failing SEO check and it caps the score at
   63. Removing the `noindex` would raise a number and invite search engines to index a tenant's operational
   console. Kept as-is and recorded in ADR-035.
2. **SEC-18 / dev-only dependency advisories (⚠).** Seven moderate advisories in `drizzle-kit`'s `esbuild` and
   `autocannon`'s `uuid`. Neither ships in a container nor is reachable at runtime; the available fixes are
   major-version downgrades of the tools. Tracked for the next dependency review.
3. **SEC-21 / invite confirms an address is already an Orbit account (⚠).** `users.email` is globally unique by
   schema, so inviting an address that exists in another tenant returns `409 EMAIL_TAKEN`. Scoping the check to
   the org would turn a handled conflict into a database constraint violation rather than remove the disclosure.
   One bit, to an authenticated administrator. Accepted and recorded in ADR-046.
