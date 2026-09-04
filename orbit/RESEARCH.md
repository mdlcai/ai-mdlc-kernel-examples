# RESEARCH.md — MDLC Orbit

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Productivity"

## Product Vision
**Problem:** IT teams often manage tickets, projects, assets, changes, approvals, and documentation across disconnected systems, spreadsheets, and email. This creates fragmented workflows, limited visibility, inconsistent processes, and difficulty tracking ownership, priorities, SLAs, and outcomes.

Organizations need a centralized IT operations platform that connects every request, project, asset, change, and workflow while providing real-time visibility, automation, accountability, and a complete audit trail.

The goal is to replace fragmented IT processes with a single, intelligent system for managing the entire lifecycle of IT work.

**Who it affects:** IT Teams — Spend time managing fragmented tools, manual processes, escalations, and administrative work instead of resolving issues and delivering projects.

Employees & End Users — Experience slow request fulfillment, limited visibility, inconsistent communication, and unclear ticket status.

IT Leadership — Lacks a unified view of workload, performance, SLA compliance, project progress, and operational risk.

Security & Compliance Teams — Need reliable audit trails, approvals, access controls, asset visibility, and evidence of operational processes.

Business Leaders — Are affected when IT delays impact productivity, projects, costs, or business operations.

The Organization — Ultimately bears the cost of inefficient IT processes, duplicated work, unresolved issues, and limited operational visibility.

**Why existing solutions fall short:** Existing ITSM platforms are often complex, expensive, heavily customized, and difficult to use. Many require extensive administration and training, while separating tickets, projects, assets, changes, and knowledge into disconnected experiences.

Smaller solutions are easier to deploy but often lack the enterprise workflows, automation, governance, integrations, and reporting needed by larger IT organizations.

The result is a tradeoff between power and usability. Teams need a platform that provides enterprise-grade capabilities without the complexity, rigidity, and overhead of traditional ITSM systems.

**Solution:** A centralized IT command center where employees submit requests, IT teams manage work, projects are coordinated, changes are governed, assets are tracked, and leadership gets real-time visibility into IT operations.

What you're building

IT Command Center

🎫 Ticketing — incidents, requests, access issues, problems
🚀 Projects — projects, milestones, tasks, dependencies, deadlines
🔄 Change Management — requests, approvals, risk, implementation, rollback
🛠️ Asset Management — computers, servers, software, network devices, users
📚 Knowledge Base — solutions, documentation, runbooks
👥 Teams & Users — ownership, assignments, roles, permissions
⚡ Automation — routing, escalation, SLA timers, notifications, recurring tasks
✅ Approvals — access, purchases, changes, projects
📊 Analytics — workload, SLA performance, MTTR, backlog, project health
🔍 Audit Trail — every status change, assignment, approval, and action
The core idea

Instead of having tickets in one system, projects in another, assets somewhere else, and approvals buried in email, everything is connected.

For example:

Employee requests laptop → Ticket created → Manager approves → Asset assigned → IT fulfills request → Ticket closes → Asset record updated → Complete history retained.

That makes it more than a help desk. It's an IT Operations Management platform.


## Users & Outcomes

> Rewritten at Stage 3b (Pre-Build Review, see DECISIONS.md ADR-013): the dashboard version repeated persona pain points under "Key Workflows" and had no measurable thresholds or Non-Goals. Personas are retained above under "Who it affects".

**Key Workflows:**
- **W1 — Org onboarding.** As an IT admin, I register my organization and first admin account, create teams, and add users with roles (admin / manager / agent / requester), so my IT team can start working in Orbit within 10 minutes.
- **W2 — Self-service request.** As an employee (requester), I search the knowledge base, then submit an incident or request from the portal and follow its status and comments through resolution, so I always know where my request stands.
- **W3 — Agent queue & SLA.** As an agent, I triage my team's queue, assign, respond, resolve and close tickets against visible SLA timers, and I am escalated to when a ticket breaches its SLA, so we resolve within SLA.
- **W4 — Connected fulfillment.** As a requester, I request a laptop; my manager approves; an agent assigns an asset and fulfills the request; the ticket closes and the asset record shows the assignment with complete history retained.
- **W5 — Change management.** As an agent, I submit a change request with risk and rollback plan; a manager approves it; the change is scheduled, implemented, and completed or rolled back, with every step audited.
- **W6 — Project delivery.** As an IT lead, I create a project with milestones and tasks, assign owners, set dependencies and due dates, and see project health on the dashboard.
- **W7 — Knowledge publishing.** As an agent, I author and publish a KB article that requesters find via search before they file a ticket (self-service deflection).
- **W8 — Automation.** As an admin, I configure a rule that auto-routes tickets by category and escalates on SLA breach, and I verify it fired from the automation run log and the audit trail.
- **W9 — Leadership visibility.** As IT leadership, I open the analytics dashboard to see workload, SLA performance, MTTR, backlog and project health, and I review the audit trail for any object.

**Success Metrics (measurable thresholds):**
| Metric | Threshold | Measured by |
|---|---|---|
| Faster resolution / MTTR | MTTR and time-to-first-response computed per team and priority; visible on `/analytics` | Analytics endpoint + e2e W9 |
| SLA performance | % resolved within SLA per policy visible in real time; breach detected ≤ 30 s after due time | SLA engine tick (15 s) + integration test |
| Automation | Routing, escalation, notification and recurring-task rules fire without manual steps; each run logged | Automation engine tests + W8 |
| User experience | Submitting a request takes < 60 s of user time (≤ 5 required fields); ticket screens LCP ≤ 2.5 s | Web Delivery Baseline (FRONTEND-AUDIT.json) |
| Visibility (real-time) | Queue, SLA and notification changes reach open browsers ≤ 2 s (SSE) | SSE integration test |
| Accountability | Every ticket, task, change and project has an owner or team; unassigned work is surfaced on the dashboard | Dashboard KPI + schema constraints |
| First-contact resolution | Reassignment count tracked per ticket and reported | Analytics |
| Self-service | KB search returns ranked results < 300 ms p95; portal search precedes ticket creation | Load test + W7 |
| Project delivery | On-time % and health (on track / at risk / late) per project | Analytics + W6 |
| Operational efficiency | API p95 < 300 ms reads / < 500 ms writes at 50 concurrent users on the local stack | autocannon load test (REPORT.md) |
| Governance | 100 % of state changes, assignments, approvals and role changes produce an immutable audit event | Audit tests + INV-5 |
| Adoption | WCAG 2.2 AA on every screen; Lighthouse ≥ 95 accessibility / best-practices / SEO, ≥ 85 performance | Web Delivery Baseline |
| Availability | Health endpoint ≤ 500 ms; graceful restart with zero dropped in-flight requests | Functional Smoke Test |

**Non-Goals (v1):** SSO / OAuth / SAML; file attachments and image uploads; outbound Slack / Teams / PagerDuty webhooks; SMS; native mobile apps; billing or payments; internationalization; offline mode; CMDB auto-discovery agents; inbound email-to-ticket parsing; multi-currency purchasing; custom scripting or plugins; multi-org membership per user.

**Integrations (v1):** SMTP (Nodemailer) for outbound notification email; HaveIBeenPwned Pwned Passwords range API for breached-password screening. No other external services.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
monitoring: "basic health checks"
container_strategy: "Docker Compose"

# Data & Storage
database_preference: "PostgreSQL"

# Security & Compliance
rate_limiting: true
audit_logging: true

# Frontend
frontend_framework: "Next.js"

# Backend
backend_framework: "Express"
realtime_needed: true

# Scope & Platform
scale: "small — under 1k concurrent"
multi_tenant: true
target_platforms: ["web"]
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Confident. Clear. Intelligent. Operational.

The platform should feel like a modern command center for IT, not another cumbersome enterprise system.

Clear — Use simple language and eliminate unnecessary ITSM jargon.
Confident — Communicate with authority without being corporate or overly formal.
Intelligent — Highlight automation, insights, context, and proactive recommendations.
Action-oriented — Focus on what needs attention, what is happening, and what comes next.
Professional — Enterprise-ready, trustworthy, and appropriate for IT leadership.
Modern — Fast, clean, focused, and technology-forward.
Human — Make complex IT operations approachable for both IT professionals and employees.

Voice principle:

Less administration. More action. Complete visibility.

Avoid: Buzzwords, overly technical language, excessive corporate terminology, unnecessary complexity, and language that makes routine IT work feel bureaucratic.

### Art Direction
Modern IT Command Center — clean, intelligent, dense, and operational.

The interface should feel like a blend of enterprise ITSM, modern SaaS, and an operations dashboard. Prioritize information density without creating visual clutter.

Clean & Minimal — Strong hierarchy, generous spacing, restrained visual noise.
Data-First — Status, priority, SLA, ownership, and health should be immediately visible.
Dark/Light Ready — Support a polished dark mode for IT operations and a clean light mode for everyday users.
Command-Center Layout — Persistent navigation, contextual side panels, dashboards, queues, and workspace views.
Status-Driven UI — Use consistent visual indicators for critical, warning, active, blocked, pending, and completed states.
Card-Based Operations — Use cards for KPIs, active incidents, projects, approvals, and work queues.
Progressive Disclosure — Keep the primary interface simple while exposing deeper technical details when needed.
Fast Interaction — Favor inline editing, quick actions, keyboard shortcuts, bulk actions, and command-style workflows.
Connected Objects — Tickets, users, assets, projects, changes, and services should link naturally to one another.
Consistent Components — Standardize tables, filters, badges, timelines, forms, modals, activity feeds, and status indicators.
Visual Personality

Precision over decoration.
Information over ornamentation.
Action over administration.

The product should look like a serious enterprise operations platform built for modern IT teams, not a traditional help-desk application.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #73bad9 | Buttons, links, active states |
| Secondary | #9478d4 | Accents, badges, highlights |
| Accent | #d8aa95 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #161b1d | Headings, body text |
| Text Secondary | #5c6a70 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #73bad9 | Buttons, links, active states |
| Secondary | #7551c7 | Accents, badges, highlights |
| Accent | #c8876a | Callouts, hover states |
| Background | #090b0c | Page background |
| Surface | #121517 | Cards, elevated containers |
| Text | #eaebec | Headings, body text |
| Text Secondary | #839095 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

### Typography
- Heading: Manrope (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 14px, scale ratio: 1.2
- Scale: 9.7 / 11.7 / 14 / 16.8 / 20.2 / 24.2 / 29px

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Rounded
- Border radius: 8px (sm: 4px, lg: 12px, xl: 16px)
- Shadows: Subtle — `0 1px 3px rgba(0,0,0,0.1)`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#73bad9',
    secondary: '#9478d4',
    accent: '#d8aa95',
    background: '#fafafa',
    surface: '#f4f5f5',
    foreground: '#161b1d',
    muted: '#5c6a70',
    success: '#1fad53',
    warning: '#ec9c13',
    destructive: '#df2020',
  },
  fontFamily: {
    heading: ['Manrope', 'system-ui', 'sans-serif'],
    body: ['Inter', 'system-ui', 'sans-serif'],
    mono: ['JetBrains Mono', 'monospace'],
  },
  borderRadius: {
    DEFAULT: '8px',
    sm: '4px',
    lg: '12px',
    xl: '16px',
  },
}
```

### CSS Custom Properties
```css
/* Light mode */
:root {
  --color-primary: #73bad9;
  --color-secondary: #9478d4;
  --color-accent: #d8aa95;
  --color-background: #fafafa;
  --color-surface: #f4f5f5;
  --color-text: #161b1d;
  --color-text-secondary: #5c6a70;
  --color-success: #1fad53;
  --color-warning: #ec9c13;
  --color-error: #df2020;
  --font-heading: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 8px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #73bad9;
  --color-secondary: #7551c7;
  --color-accent: #c8876a;
  --color-background: #090b0c;
  --color-surface: #121517;
  --color-text: #eaebec;
  --color-text-secondary: #839095;
  --color-success: #33cc6b;
  --color-warning: #e2a336;
  --color-error: #d74242;
}
```

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

No design template was uploaded for this project. Fall back to the archetype
defaults from `DESIGN.md` Part II plus the tokens in `get_project_config`.

---

# Research Sections (filled by Stage 0 on 2026-09-02, build_depth: comprehensive)

> Every URL below was returned by a web search or fetched page during Stage 0 research. Package versions were confirmed against the npm registry (`npm view <pkg> version`) on 2026-09-02. Cells marked "unverified" could not be confirmed against a primary source.

## 3. Source Categories

### 3.1 — Official Documentation & Vendor Sources

| Source | URL | What It Covers | Notes |
|--------|-----|----------------|-------|
| Next.js 16 upgrade guide | https://nextjs.org/docs/app/guides/upgrading/version-16 | `middleware.ts` → `proxy.ts` rename (Node runtime only, no edge); sync `params`/`searchParams`/`cookies()`/`headers()`/`draftMode()` fully removed (async only); Turbopack default for dev+build; `next lint` removed; `revalidateTag` needs 2nd arg; parallel routes require `default.js`; `serverRuntimeConfig`/`publicRuntimeConfig` removed; Node 20.9+ / TS 5.1+ | Page header `version: 16.3.4`, `lastUpdated: 2026-08-25`; verified 2026-09-02 |
| Next.js `output` config | https://nextjs.org/docs/app/api-reference/config/next-config-js/output | `output: 'standalone'` → `.next/standalone/server.js`; `public/` and `.next/static` must be copied manually; `outputFileTracingRoot` for monorepos; `PORT`/`HOSTNAME` env | `version: 16.3.4`, `lastUpdated: 2025-10-08`; verified 2026-09-02 |
| Next.js Font module | https://nextjs.org/docs/app/api-reference/components/font | `next/font/google`, build-time download / self-hosting, `variable` for CSS vars, Tailwind v4 `@theme inline` integration example | `version: 16.3.4`, `lastUpdated: 2025-08-06`; verified 2026-09-02 |
| Next.js CSP guide | https://nextjs.org/docs/app/guides/content-security-policy | Nonce generated in `proxy.ts`, `'strict-dynamic'`, `x-nonce` request header, nonce pages are dynamically rendered | Updated 2026-03-20 (v16.3.4); verified 2026-09-02 |
| Express 5 migration guide | https://expressjs.com/en/guide/migrating-5.html | Rejected promises auto-forwarded to error handlers; path-to-regexp v8 syntax (`/*splat`, `{.:ext}` optional, no regex chars, arrays for alternates); `req.query` read-only getter; `req.body` `undefined` when unparsed; `res.status()` ints 100–999; removed `res.send(body,status)`, `res.redirect('back')`, `req.param()`; Node 18+ | Matches npm express 5.2.1; verified 2026-09-02 |
| Zod 4 changelog | https://zod.dev/v4/changelog | `message`/`invalid_type_error`/`required_error`/`errorMap` → unified `error`; `z.string().email()` → `z.email()`; `.strict()`/`.passthrough()` → `z.strictObject()`/`z.looseObject()`; `.default()` must match output type (`.prefault()` for old behavior); `z.record()` requires 2 args; `z.coerce.*` input is `unknown` | npm zod 4.5.4; verified 2026-09-02 |
| Drizzle ORM — PostgreSQL get started | https://orm.drizzle.team/docs/get-started-postgresql | `drizzle-orm/node-postgres` driver init from connection string or existing `pg` `Pool` (`drizzle({ client: pool })`); `drizzle.config.ts` | npm drizzle-orm 0.45.2 (docs snippet shows `@rc`); verified 2026-09-02 |
| Drizzle Kit overview | https://orm.drizzle.team/docs/kit-overview | `drizzle-kit generate` (SQL files) / `migrate` (apply) / `push`; `./drizzle` out dir; `__drizzle_migrations` tracking table; `dialect` + `schema` required | npm drizzle-kit 0.31.10; verified 2026-09-02 |
| Drizzle RLS | https://orm.drizzle.team/docs/rls | `pgPolicy`, `.enableRLS()`, `pgRole` declared in schema so policies migrate with tables | verified 2026-09-02 |
| node-postgres Pool API | https://node-postgres.com/apis/pool | `max` (default 10), `idleTimeoutMillis` (10 s), `connectionTimeoutMillis`; `pool.connect()`/`client.release()` for transactions; never `pool.query()` for transactions; attach `pool.on('error')` | npm pg 8.23.0; verified 2026-09-02 |
| Pino redaction | https://github.com/pinojs/pino/blob/main/docs/redaction.md | `redact: { paths, censor, remove }`; dot / bracket / wildcard paths; ~2 % overhead for explicit paths; paths must never come from user input | npm pino 10.3.1 (MIT); verified 2026-09-02 |
| express-rate-limit configuration + changelog | https://express-rate-limit.mintlify.app/reference/configuration · https://express-rate-limit.mintlify.app/reference/changelog | `keyGenerator`, `ipKeyGenerator(ip, ipv6Subnet)`, `ipv6Subnet` (default /56), `keyGeneratorIpFallback` validation, `store`, `standardHeaders: 'draft-8'`, `limit` replaces `max` | Changelog top entry v8.7.0; npm 8.7.0; verified 2026-09-02 |
| Helmet | http://helmet.js.org/ | 13 default headers (CSP, HSTS, COOP/CORP, X-Frame-Options…); `contentSecurityPolicy: { useDefaults, directives }`; directive values may be `(req,res)=>` functions for nonces | npm helmet 8.3.0, MIT, engines ≥18; verified 2026-09-02 |
| node-argon2 README | https://github.com/ranisalt/node-argon2 | Prebuilt binaries (prebuildify/N-API) for Linux glibc x64/arm64, Alpine musl x64/arm64, macOS, Windows x64; fallback `--build-from-source` needs a C++ toolchain | README tested on Node ≥22; npm 0.45.1 engines ≥16.17; verified 2026-09-02 |
| PostgreSQL 17 — Row Security Policies | https://www.postgresql.org/docs/17/ddl-rowsecurity.html | `ENABLE`/`FORCE ROW LEVEL SECURITY`, `CREATE POLICY … USING / WITH CHECK`, permissive vs restrictive, superuser/`BYPASSRLS`/owner bypass, `row_security` GUC | PostgreSQL 17.11 docs; verified 2026-09-02 |
| PostgreSQL 17 — Explicit Locking / Advisory Locks | https://www.postgresql.org/docs/17/explicit-locking.html | `pg_advisory_lock`/`pg_try_advisory_lock` (session) vs `pg_advisory_xact_lock`; re-entrancy; bounded by `max_locks_per_transaction` | PostgreSQL 17.11; verified 2026-09-02 |
| PostgreSQL 17 — SELECT locking clause | https://www.postgresql.org/docs/17/sql-select.html | `FOR UPDATE … SKIP LOCKED` explicitly for queue-like tables; not allowed with GROUP BY/DISTINCT/aggregates | PostgreSQL 17.11; verified 2026-09-02 |
| Caddy — Automatic HTTPS | https://caddyserver.com/docs/automatic-https | Auto cert issuance + HTTP→HTTPS redirect; local CA for localhost/internal names (root under `pki/authorities/local`); `http://` prefix disables | Latest Caddy v2.11.4 (2026-06-03, GitHub releases); verified 2026-09-02 |
| Caddy — `tls` directive | https://caddyserver.com/docs/caddyfile/directives/tls | `tls internal`, `tls <cert> <key>`, `issuer acme|zerossl|internal`, `on_demand` | verified 2026-09-02 |
| Playwright — Installation + Release notes | https://playwright.dev/docs/intro · https://playwright.dev/docs/release-notes | `npx playwright install --with-deps`; Node 22.x/24.x/26.x required; Windows 11+, Ubuntu 22.04/24.04 | Release notes top = 1.62 (Chromium 151); npm @playwright/test 1.62.1; verified 2026-09-02 |
| Tailwind CSS — Theme variables | https://tailwindcss.com/docs/theme | CSS-first config: `@import "tailwindcss"; @theme { --color-*, --font-*, --text-*, --spacing-*, --breakpoint-*, --radius-*, --shadow-* }`; `@theme inline` for var references; `--color-*: initial` to reset defaults | Page labeled v4.3; npm tailwindcss 4.3.3; verified 2026-09-02 |
| MDN — Using server-sent events | https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events | `EventSource`, `text/event-stream`, `data:`/`event:`/`id:`/`retry:`/comment keep-alive, auto-reconnect, HTTP/1.1 6-connections-per-origin limit (HTTP/2 ~100 streams) | MDN last modified 2025-05-15; verified 2026-09-02 |
| Docker Compose — Services reference | https://docs.docker.com/reference/compose-file/services/ | `depends_on` long syntax: `condition: service_started | service_healthy | service_completed_successfully`; `healthcheck: test/interval/timeout/retries/start_period/start_interval` | Compose Specification; verified 2026-09-02 |
| TypeScript 7.0 announcement | https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/ | Native Go port shipped as `typescript`; **no programmatic Compiler API in 7.0** (`@typescript/typescript6` for tools like typescript-eslint); hard errors for `baseUrl`, `moduleResolution node`, `target es5`; new defaults `strict: true`, `module: esnext` | Post dated 2026-07-08; npm typescript 7.0.2; verified 2026-09-02 |
| ESLint — Migrate to v10 + configuration files | https://eslint.org/docs/latest/use/migrate-to-10.0.0 · https://eslint.org/docs/latest/use/configure/configuration-files | eslintrc removed; Node ≥20.19 / ≥22.13 / 24+; config resolved from each linted file's directory; `defineConfig()`; new recommended rules | npm eslint 10.9.1; verified 2026-09-02 |

**Key Takeaways:**
- **Express 5 route syntax is a hard break:** wildcards must be named (`/*splat`), optional segments use braces, regex characters in string paths are gone. `req.query` is a read-only getter, `req.body` is `undefined` until a parser runs, and rejected promises in async handlers reach `next(err)` automatically — no `express-async-errors`.
- **Next.js 16:** rename `middleware.ts` → `proxy.ts` (export `proxy()`), Node runtime only. `params`, `searchParams`, `cookies()`, `headers()` are Promise-only. `next lint` is gone (run ESLint directly). Standalone output requires copying `public/` and `.next/static` into `.next/standalone/` in the Dockerfile and `outputFileTracingRoot` in a workspace.
- **Zod 4:** `{ error: '...' }` replaces `message`/`required_error`; `z.email()`, `z.uuid()` are top-level; `z.strictObject()` / `z.looseObject()`; `z.record(key, value)` needs both args.
- **express-rate-limit v8:** IPv6 masked to /56 by default; custom `keyGenerator` falling back to `req.ip` must wrap it in `ipKeyGenerator(req.ip)`; use `limit`, `standardHeaders: 'draft-8'`.
- **TypeScript 7.0.2 ships no Compiler API**, so typescript-eslint 8.69 (peer `typescript >=4.8.4 <6.1.0`) cannot run on it → pin TypeScript 6.0.3 (ADR-005).
- **ESLint 10:** flat config only; resolved per linted file's directory — the monorepo root `eslint.config.mjs` uses `defineConfig()` with per-package overrides.
- **Tailwind v4:** no `tailwind.config.js`; tokens live in CSS `@theme`; runtime CSS variables (theme toggle) go through `@theme inline`.
- **argon2 prebuilt binaries** cover win32-x64 and Alpine musl, so `npm ci` works on the Windows dev box and in `node:22-alpine` without a compiler; never substitute bcryptjs.
- **PostgreSQL RLS:** table owners bypass RLS unless `FORCE ROW LEVEL SECURITY`; the app must connect as a non-owner, `NOBYPASSRLS` role; `SET LOCAL` per transaction; `FOR UPDATE SKIP LOCKED` for the SLA sweep; `pg_try_advisory_xact_lock` for the scheduler lease.
- **SSE limits:** HTTP/1.1 caps ~6 connections per origin, so terminate TLS with HTTP/2 at Caddy and use one `EventSource` per tab.

### 3.2 — GitHub & Open Source

| Repository | URL | Stars | Last Active | Relevance |
|------------|-----|-------|-------------|-----------|
| GLPI (glpi-project/glpi) | https://github.com/glpi-project/glpi | ~6.3k | Sep 2026 | Full ITIL service desk + CMDB/asset; GPL-3.0 — reference for entity (tenant) model, ticket lifecycle, SLA/OLA, change/problem workflows. Reference-only. |
| Zammad (zammad/zammad) | https://github.com/zammad/zammad | ~5.9k | Sep 2026 | Rails/Vue helpdesk with triggers/automation, SLA timers, roles/permissions; AGPL-3.0. Reference-only for automation rule design. |
| osTicket (osTicket/osTicket) | https://github.com/osTicket/osTicket | ~3.9k | Jun 2026 | Classic PHP ticket system; GPL-2.0. Reference for help topics / canned responses; less active. |
| FreeScout (freescout-help-desk/freescout) | https://github.com/freescout-help-desk/freescout | ~4.5k | Sep 2026 | Laravel shared-mailbox helpdesk; AGPL-3.0. Reference for conversation UX. Reference-only. |
| Peppermint (Peppermint-Lab/peppermint) | https://github.com/Peppermint-Lab/peppermint | ~3.2k | Archived Jul 2026 | Closest stack analogue (Next.js + Node + PostgreSQL) for a ticketing app; license "Other" (unverified). Archived — do not depend on. |
| Snipe-IT (grokability/snipe-it) | https://github.com/grokability/snipe-it | ~14.9k | Sep 2026 | Laravel IT asset & license management; AGPL-3.0. Reference for asset schema (statuses, check-in/out, custom fields, audit history). Reference-only. |
| iTop (Combodo/iTop) | https://github.com/Combodo/iTop | ~1.2k | Sep 2026 | ITIL-oriented CMDB + change mgmt (CI relationships, impact analysis); AGPL-3.0. Reference-only. |
| Drizzle ORM (drizzle-team/drizzle-orm) | https://github.com/drizzle-team/drizzle-orm | ~35.7k | Sep 2026 | Chosen ORM; Apache-2.0 (dependency-safe). |
| Express (expressjs/express) | https://github.com/expressjs/express | ~69.4k | Sep 2026 | Chosen API framework; MIT. Express 5 line. |
| Pino (pinojs/pino) | https://github.com/pinojs/pino | ~18.2k | Sep 2026 | Chosen structured logger with redaction; MIT. |
| express-rate-limit | https://github.com/express-rate-limit/express-rate-limit | ~3.3k | Aug 2026 | Chosen rate limiter; MIT. v8.7.0. |
| node-argon2 (ranisalt/node-argon2) | https://github.com/ranisalt/node-argon2 | ~2.2k | Aug 2026 | Chosen password hashing; MIT. Prebuilt binaries for win32/linux/alpine. |

**Evaluation Checklist:**
- [x] License compatible with our use case? — every runtime dependency is MIT / Apache-2.0 / MIT-0; all ITSM prior art is GPL/AGPL → reference-only.
- [x] Actively maintained (commits in last 6 months)? — all chosen libraries pushed within 3 days of 2026-09-02.
- [x] Good documentation / examples? — yes for all chosen libraries (see §3.1).
- [x] Community size & issue response time? — express/drizzle/pino large; express-rate-limit and argon2 smaller but actively released.
- [x] Use as dependency vs. reference only? — libraries: dependency; ITSM projects: reference only.

**Key Takeaways:**
- The license split is clean: everything shipped at runtime is permissive; every ITSM prior-art project is copyleft (AGPL triggers on network use), so no code is copied or vendored.
- Peppermint is the only Node/Next.js analogue and it is archived with an unclear license — skim only.
- Snipe-IT and iTop are the best asset/CMDB references (status lifecycle, check-in/out, CI relationships); GLPI and Zammad are the best references for ticket lifecycle, SLA timers, triggers and multi-entity scoping.
- Nothing is directly reusable (PHP/Ruby); the value is validating feature scope and avoiding known UX pitfalls.

### 3.3 — Video & Tutorial Sources (YouTube, Courses, Talks)

| Title | URL | Creator | Length | Why It's Useful |
|-------|-----|---------|--------|-----------------|
| Everything you need to know about Postgres Row Level Security — POSETTE 2024 | https://www.youtube.com/watch?v=vZT1Qx2xUCo | Microsoft Developer (Paul Copplestone, Supabase) | n/a | RLS policies, roles and pitfalls — the mechanism Orbit uses for shared-schema `org_id` isolation. |
| Divide and Conquer: Multi-tenancy in Postgres — Citus Con 2023 | https://www.youtube.com/watch?v=jTNeooqyTnc | Microsoft Developer | n/a | Compares db-per-tenant, schema-per-tenant and shared-table models; justifies shared schema + RLS at small scale. |
| Next.js + PostgreSQL + Drizzle ORM — Full Stack Project | https://www.youtube.com/watch?v=tiSm8ZjFQP0 | Dave Gray | n/a | Drizzle schema, migrations and typed queries inside an App Router project. |
| What's New in Express 5? | https://www.youtube.com/watch?v=TsUpLjA5H5E | Code With Ujjwal | n/a | Express 5 changes: automatic promise rejection → `next(err)`, dropped APIs. |
| Express.js 5 is here | https://www.youtube.com/watch?v=-MMjFX5UfN4 | Academind | n/a | Migration overview (path-to-regexp v8 syntax, `req.query` getter, async handlers). |
| Incident Management Fundamentals for Beginners | https://www.youtube.com/watch?v=cbox8-9Nh7Q | CodeLucky | n/a | Incident lifecycle in ITIL terms; source for states, priority matrix, MTTR definitions. |
| Server-Sent Events vs WebSockets — System Design | https://www.youtube.com/watch?v=X_DdIXrmWOo | System Design School | n/a | When one-way push is sufficient — validates SSE for Orbit's live queues. |
| WebSockets vs Polling vs Server Sent Events | https://www.youtube.com/watch?v=WS352jTTkPU | Piyush Garg | n/a | Reconnection behavior and proxy friendliness before implementing SSE behind Caddy. |

**Key Takeaways:**
- Shared tables with `org_id` + RLS is the standard small-scale multi-tenant choice; both Postgres talks stress forcing RLS on table owners and testing under a non-superuser role.
- Express 5's headline change is automatic async error forwarding; the route-syntax change is the one to plan for.
- Drizzle + App Router tutorials converge on schema file → `drizzle-kit generate` → committed SQL → typed queries.
- SSE is the right default for server→client notifications; WebSockets only pay off with high-frequency client pushes.
- ITIL vocabulary for analytics: time-to-acknowledge, time-to-resolve (MTTR), SLA breach %.

### 3.4 — Articles, Blogs & Written Tutorials

| Title | URL | Author/Site | Date | Relevance |
|-------|-----|-------------|------|-----------|
| Multi-tenant data isolation with PostgreSQL Row Level Security | https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security | Michael Beardsley / AWS Database Blog | 2020-05-18 | Canonical pattern: `tenant_id` column, policy on `current_setting('app.current_tenant')`, non-owner app role. |
| Row Level Security for Tenants in Postgres | https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres | Craig Kerstiens / Crunchy Data | 2024-04-03 | Why per-request DB users don't scale; session variables (`SET LOCAL`) as tenant context. |
| Shipping multi-tenant SaaS using Postgres Row-Level Security | https://www.thenile.dev/blog/multi-tenant-rls | Miki Pokryvailo / Nile | 2022-07-26 | Production write-up of RLS with session-variable context; HN thread in §3.7 covers pooling gotchas. |
| RLS sounds great until it isn't | https://planetscale.com/blog/rls-sounds-great-until-it-isnt | Josh Brown / PlanetScale | 2026-04-30 | Counterpoint: per-row policy cost, owner bypass, policy drift, `SET LOCAL` burden — a mitigation checklist. |
| Row-Level Security in Supabase: Multi-Tenant SaaS from Day One | https://dev.to/issuecapture/row-level-security-in-supabase-multi-tenant-saas-from-day-one-4lon | Benji Darby / DEV | unverified year | Policy-writing style; "default deny once RLS is enabled". |
| Understanding SLA Time Calculations | https://support.liveagent.com/804145-Understanding-SLA-Time-Calculations | LiveAgent Support | 2025-02-11 | Business-hours extension rules for due dates and which statuses pause the clock. |
| Automatic Audit Logging with PostgreSQL Triggers | https://emmer.dev/blog/automatic-audit-logging-with-postgresql-triggers/ | Christian Emmer | 2024-12-12 | Trigger-driven audit tables with operation/timestamp/actor metadata. |
| Tamper-evident audit trails in PostgreSQL with hash chaining | https://appmaster.io/blog/tamper-evident-audit-trails-postgresql | AppMaster blog | 2025-08-29 | Append-only rows storing the previous row's hash — a verifiable "immutable" claim. |
| How to Build a Server-Sent Events Endpoint with Express.js | https://oneuptime.com/blog/post/2026-02-17-server-sent-events-express-cloud-run-real-time/view | Nawaz Dhandala / OneUptime | 2026-02-17 | Express SSE with `id:` fields, 30 s heartbeat comments, `Last-Event-ID` replay. |
| Server Sent Events | https://javascript.info/server-sent-events | Ilya Kantor / javascript.info | n/a | Client side: `id`, `retry`, `Last-Event-ID` on reconnect, `readyState`. |
| How to set a Content Security Policy for your Next.js application | https://nextjs.org/docs/app/guides/content-security-policy | Vercel / Next.js docs | 2026-03-20 | Nonce in `proxy.ts`, `'strict-dynamic'`, `x-nonce` header, dynamic rendering requirement. |
| Implementing Optimistic Locking in PostgreSQL | https://reintech.io/blog/implementing-optimistic-locking-postgresql | Reintech | 2024-07-03 | Integer `version` column checked in `UPDATE … WHERE version = $n`; 0 rows = conflict (409). |
| Designing robust and predictable APIs with idempotency | https://stripe.com/blog/idempotency | Brandur Leach / Stripe | 2017-02-22 | `Idempotency-Key` pattern: store first response, replay on retry, expire keys after ~24 h. |
| Implementing Stripe-like Idempotency Keys in Postgres | https://brandur.org/idempotency-keys | Brandur Leach | 2017-10-27 | Postgres schema and atomic phases for idempotency keys. |
| How to design an RBAC model for multi-tenant SaaS | https://workos.com/blog/how-to-design-multi-tenant-rbac-saas | Maria Paktiti / WorkOS | 2025-11-28 | Global vs tenant-scoped vs hybrid role patterns — blueprint for teams/roles/permissions. |

**Key Takeaways:**
- One shared schema, `org_id` on every tenant-owned table, `ENABLE` + `FORCE ROW LEVEL SECURITY`, non-owner role, `SET LOCAL` inside each transaction so pooled connections cannot leak context.
- Keep RLS policies in migration files under source control and test them under the app role; use cheap `current_setting()` comparisons and keep app-layer authorization as defense in depth.
- SLA timers must be computed against a per-policy business calendar and paused on defined statuses.
- Audit trail: append-only table written by the service layer storing old/new JSONB, actor, request id; revoke UPDATE/DELETE.
- SSE: `id:` on every event, comment heartbeat every 20–30 s, replay from `Last-Event-ID`.
- Next.js 16 CSP nonces live in `proxy.ts` and force dynamic rendering; combine with version-column optimistic locking.

### 3.5 — Standards, RFCs & Specifications

| Standard | Reference | URL | Applicability |
|----------|-----------|-----|---------------|
| ITIL 4 Management Practices (PeopleCert/Axelos) | Incident, problem, service request, change enablement, service level management, IT asset management, knowledge management practice guides | https://www.peoplecert.org/ITIL4-practices | Vocabulary and process shape for tickets (incident/request/problem), change enablement with approval, assets, KB and SLA modules |
| RFC 9457 Problem Details for HTTP APIs (2023, obsoletes 7807) | IETF RFC 9457 | https://www.rfc-editor.org/rfc/rfc9457.html | Single JSON error envelope: `type`, `title`, `status`, `detail`, `instance` + extension members; `application/problem+json` |
| RFC 6265bis — Cookies (Internet-Draft, Aug 2026 revision) | draft-ietf-httpbis-rfc6265bis | https://httpwg.org/http-extensions/draft-ietf-httpbis-rfc6265bis.html | `SameSite`, `Secure`, `HttpOnly`, `__Host-` prefix for the session cookie |
| OWASP ASVS 5.0 (May 2025) | Application Security Verification Standard v5.0.0 | https://github.com/OWASP/ASVS | Verification checklist for auth, sessions, access control, validation, error handling, logging; target Level 2 |
| OWASP Top 10:2021 | A01–A10 | https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/ | Historical floor; A01/A03/A07/A09 map to Orbit controls |
| OWASP Top 10:2025 (published Jan 2026) | A01 Broken Access Control … A03 Software Supply Chain Failures … A10 Mishandling of Exceptional Conditions | https://owasp.org/Top10/2025/ | Current floor; A03 (lockfiles, pinned images) and A10 (fail-closed error handling, RFC 9457 responses) are new obligations |
| NIST SP 800-63B-4 (Final, July 2025) | §3.1.1 password verifiers | https://csrc.nist.gov/pubs/sp/800/63/b/4/final | ≥ 15 chars when the password is single-factor (≥ 8 inside MFA); allow ≥ 64 chars, all printing ASCII + Unicode; SHALL NOT impose composition rules; SHALL screen against blocklists/breached corpora; SHALL NOT force periodic rotation; rate-limit guesses |
| OWASP Password Storage Cheat Sheet | Argon2id parameter table | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html | Argon2id m=19456 KiB, t=2, p=1 (or 46 MiB / t=1) |
| WCAG 2.2 (W3C Rec, Oct 2023; ISO/IEC 40500:2025) | 2.4.11 Focus Not Obscured, 2.5.8 Target Size (Min), 3.3.7 Redundant Entry, 3.3.8 Accessible Authentication | https://www.w3.org/TR/WCAG22/ | Level AA target for console and portal; 3.3.8 forbids cognitive tests at login |
| WHATWG HTML Living Standard §9.2 Server-sent events | `EventSource`, `text/event-stream`, `id:`/`retry:`/`Last-Event-ID` | https://html.spec.whatwg.org/multipage/server-sent-events.html | Contract for live ticket/queue updates and reconnect replay |
| ISO/IEC 20000-1:2018 (+ Amd 1:2024) | Service management system requirements, clauses 4–10 | https://www.iso.org/standard/70636.html | Evidence records Orbit produces (incident/request, change, configuration, SLA reporting, audit trail) |
| Content Security Policy Level 3 (W3C WD) | nonces, `strict-dynamic`, `frame-ancestors` | https://www.w3.org/TR/CSP3/ | Nonce-based CSP from `proxy.ts`; `connect-src 'self'` covers SSE |
| RFC 9110 HTTP Semantics (2022) | §8.8 ETag, §13.1 conditional requests, 412 | https://httpwg.org/specs/rfc9110.html | Optimistic concurrency: `If-Match` = version → 409/412 with RFC 9457 body |
| IETF Idempotency-Key header (draft-07, Oct 2025) | `Idempotency-Key` | https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header | Reference for safe POST retries (deferred in v1 — ADR-020) |

**Key Takeaways:**
- Password policy inherits NIST SP 800-63B-4 verbatim: **minimum 15 characters** (Orbit has no MFA in v1), maximum 128, Unicode and spaces allowed, no composition rules, no periodic rotation, breached-password screening on set/change, throttled attempts.
- Hash with Argon2id at OWASP parameters (m=19456 KiB, t=2, p=1), storing parameters with the hash so cost can be raised later.
- Every API error is an RFC 9457 `application/problem+json` document with `type`, `title`, `status`, `detail`, `instance` plus `code`, `requestId`, `errors[]`.
- Session cookie per RFC 6265bis: `HttpOnly; Secure; SameSite=Strict; Path=/; __Host-` prefix; CSRF defense via SameSite + Origin check.
- The security floor is OWASP Top 10:2025 verified against ASVS 5.0 Level 2 controls.
- Accessibility target WCAG 2.2 AA; realtime follows the WHATWG SSE spec.

### 3.6 — Competing / Adjacent Products

| Product | URL | Pricing | Strengths | Weaknesses | Our Differentiation |
|---------|-----|---------|-----------|------------|---------------------|
| ServiceNow ITSM | https://www.servicenow.com/products/itsm/pricing.html | Quote-based (third-party estimates $70–$200/fulfiller/mo — unverified) | Full ITIL suite, CMDB, workflow engine, ecosystem | Cost, implementation complexity, minimum seats | Lightweight self-hostable command center; days-not-months setup |
| Jira Service Management | https://www.atlassian.com/software/jira/service-management/pricing | Free (3 agents); Standard $20; Premium $48 /agent/mo annual; Enterprise custom | Dev/ops integration, SLA calendars/pause, automation, marketplace | Asset/CMDB gated to Premium; add-on costs; Jira data model | Native assets + change + projects in one schema at entry tier |
| Freshservice | https://www.freshworks.com/freshservice/pricing/ | Starter $19, Growth $49, Pro $99 /agent/mo annual; AI add-on +$29 | Polished UX, ITIL-aligned, asset discovery | Add-on stacking; MSP multi-tenancy is a separate product | Multi-tenancy core; automation included, not metered |
| Zendesk Suite | https://www.zendesk.com/pricing/ | Team $55, Professional $115 /agent/mo annual | Omnichannel support, KB, reporting | Customer-service oriented: no change/CAB, no CMDB | IT-operations-first with one audit trail |
| ManageEngine ServiceDesk Plus | https://www.manageengine.com/products/service-desk/pricing.html | From $13 / $27 / $67 per technician/mo (cloud tiers unverified) | ITIL-ready, strong assets, on-prem option | Dated UI, edition sprawl | Modern UX with live SSE updates; single edition |
| SolarWinds Service Desk | https://www.solarwinds.com/service-desk/pricing | Essentials $39, Advanced $99, Premier $124 /technician/mo | ITAM integration, unlimited requesters | Change/CMDB in higher tiers; SaaS-only | Self-host from one Compose stack; change + assets at base |
| HaloITSM | https://usehalo.com/haloitsm/pricing | £66/agent/mo annual, single tier | All ITIL modules, concurrent licensing | Price for small teams; heavy configuration | Same no-tiers philosophy; explicit state machines instead of deep config |
| Zammad | https://zammad.com/en/pricing/table | Hosted €7 / €16 / €25 per agent/mo; self-host free | Open source, cheap, good ticketing | No CMDB, no change/CAB; SLA & KB withheld from Starter | Full ITSM scope with immutable audit |
| GLPI Network | https://www.glpi-project.org/en/pricing/ | Cloud €19/agent/mo; self-host support €100–€4500/mo | Mature CMDB/inventory, plugins | PHP legacy UI, plugin-driven | TypeScript/API-first with RFC 9457, ETags, SSE |
| Linear (UX reference) | https://linear.app/pricing | Free; Basic $10; Business $16 /user/mo | Keyboard-first, fast, opinionated | Not ITSM: no SLA, CMDB, portal | Borrow Linear's speed for the agent console with ITSM semantics |
| Snipe-IT | https://snipeitapp.com/pricing | Self-host free; hosting from $39.99/mo | Solid asset tracking, check-in/out, API | Asset-only | Assets linked to tickets/changes with audit |

**Key Takeaways:**
- The market splits into enterprise suites that gate CMDB/change behind higher tiers and cheap helpdesks that lack change/CAB and CMDB — Orbit targets the gap: full ITSM scope at helpdesk price.
- Almost every vendor stacks per-agent AI add-ons and overages; Orbit keeps rules/automation/SLA timers in the base.
- True multi-tenancy (RLS-enforced) is absent or a separate MSP SKU for most competitors; it is Orbit's core data model.
- `docker compose up` + automatic HTTPS matches open-source deployability while offering enterprise process scope.
- The UX bar is set by Linear (speed, keyboard, live updates); no ITSM incumbent meets it.

### 3.7 — Community & Forums

| Thread/Post | URL | Platform | Key Insight |
|-------------|-----|----------|-------------|
| Goodbye asyncHandler: Native Async Support in Express 5 | https://dev.to/mahmud007/goodbye-asynchandler-native-async-support-in-express-5-2o9p | DEV | Express 5 forwards rejections to the 4-arg error middleware; event-emitter errors still are not covered. |
| Drizzle: Updated Migration Process · Discussion #2624 | https://github.com/drizzle-team/drizzle-orm/discussions/2624 | GitHub Discussions | Strict migrations, `status` column in `__drizzle_migrations`, explicit `BEGIN/COMMIT`, locking to stop concurrent migration runs. |
| Drizzle: applying migrations needs drizzle-orm and a config file · Issue #2868 | https://github.com/drizzle-team/drizzle-orm/issues/2868 | GitHub Issues | Ship a small `migrate.ts` using `drizzle-orm/migrator` in the image rather than the CLI. |
| Next.js standalone in a PNPM workspace with Docker · Discussion #48604 | https://github.com/vercel/next.js/discussions/48604 | GitHub Discussions | Also copy `public/` and `.next/static/`; set `outputFileTracingRoot` in monorepos. |
| Best way to use Next.js with Docker · Discussion #16995 | https://github.com/vercel/next.js/discussions/16995 | GitHub Discussions | Multi-stage build, `npm ci`, non-root user, exec-form `CMD` for SIGTERM. |
| Caddy doesn't flush response buffer for SSE · Issue #4247 | https://github.com/caddyserver/caddy/issues/4247 | GitHub Issues | Write an initial comment (`:\n\n`) immediately; set `flush_interval -1` on the SSE route. |
| Server-Sent Events buffering with reverse_proxy | https://caddy.community/t/server-sent-events-buffering-with-reverse-proxy/11722 | Caddy Community | Caddy auto-streams `text/event-stream`; avoid `encode` compression on the SSE path. |
| Shipping Multi-Tenant SaaS Using Postgres RLS (HN) | https://news.ycombinator.com/item?id=32241820 | Hacker News | Session-variable context + pooling can authorize with the previous request's tenant unless `SET LOCAL` per transaction; views need `security_invoker`. |
| current_setting initialization · Discussion #516 | https://github.com/porsager/postgres/discussions/516 | GitHub Discussions | Per-transaction `SET LOCAL` in a wrapper rather than pool-wide state. |
| SET LOCAL doesn't become undefined after commit | https://www.postgresql.org/message-id/CAB_pDVVa84w7hXhzvyuMTb8f5kKV3bee_p9QTZZ58Rg7zYM7sw%40mail.gmail.com | pgsql-general | After commit the setting reads back as `''` — policies must use `NULLIF(current_setting('app.org_id', true), '')` so pooled connections fail closed. |
| BusinessDays/WorkHours-based SLA calculation | https://community.atlassian.com/forums/Jira-questions/BusinessDays-WorkHours-based-SLA-calculation/qaq-p/2018182 | Atlassian Community | Business-hour due dates need calendars with holidays. |
| SQL SLA calculation excluding weekends and holidays | https://www.sqlservercentral.com/forums/topic/sql-sla-calculation-excluding-weekends-and-holidays | SQLServerCentral | Count only in-window hours per day using a calendar table. |
| Reader comments: "You'll still need ServiceNow" | https://forums.theregister.com/forum/all/2022/05/16/bill_mcdermott_interview/ | The Register forums | "10-minute jobs" become "half a day filling out forms" (page fetch timed out — unverified). |
| Top ITSM Tools According to Reddit in 2026 | https://itsmtools.com/best/top-itsm-tools-according-to-reddit-in-2026/ | r/sysadmin & r/ITManagers roundup | ServiceNow only with dedicated admins; JSM setup "steep" for non-Atlassian shops — the gap Orbit targets. |

**Key Takeaways:**
- Direct Stack Overflow / Reddit URLs could not be fetched (crawler blocked); GitHub, HN, Caddy Community, Atlassian Community and pgsql-general threads were confirmed.
- Keep one 4-arg error middleware plus `process.on('unhandledRejection')`.
- Run Drizzle migrations from `migrate.ts` (drizzle-orm migrator) in the container, relying on its lock.
- Next.js standalone images need `.next/static` and `public` copied in, a non-root user and exec-form `CMD`.
- SSE through Caddy: no compression on the events route, `flush_interval -1`, initial comment + heartbeats.
- RLS + pooling: wrap every request in a transaction with `SET LOCAL app.org_id`, treat `''` as no tenant, keep app-level checks as a second layer.
- ITSM users' loudest complaints are process bloat and admin-heavy configuration; Orbit's differentiator is opinionated defaults that work without a platform admin.

### 3.8 — APIs & Integrations

| API/Service | URL | Auth Method | Rate Limits | Docs Quality | SDK Available? |
|-------------|-----|-------------|-------------|--------------|----------------|
| SMTP via Nodemailer (SES SMTP, Postmark SMTP, Mailpit for dev) | https://nodemailer.com/ | SMTP `auth: { user, pass }` over STARTTLS (587) or TLS (465) | Provider-dependent | Good — concise API, MIT-0 | `nodemailer` 9.x (npm) |
| Resend | https://resend.com/docs/api-reference/introduction | `Authorization: Bearer re_…`; `User-Agent` required | 10 req/s per team, 429 on excess | Good | `resend` 6.x (MIT) |
| Postmark | https://postmarkapp.com/developer/api/overview | `X-Postmark-Server-Token`; `POSTMARK_API_TEST` for no-send testing | 429 on excess; batch ≤ 500 msgs | Very good | `postmark` 5.x (MIT) |
| Amazon SES | https://docs.aws.amazon.com/ses/latest/dg/quotas.html | SigV4 (API) or SMTP credentials | Sandbox 200 emails/24 h, 1/s | Good | `@aws-sdk/client-sesv2`; Nodemailer SES transport |
| Slack Incoming Webhooks (non-goal v1) | https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks | Secret in URL | 1 msg/s per webhook; 429 + `Retry-After` | Good | `@slack/webhook` 8.x or `fetch` |
| Microsoft Teams Incoming Webhooks / Workflows (non-goal v1) | https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook | Secret in workflow URL; Office 365 Connectors retiring | > 4 req/s throttles; 28 KB payload | Good (updated 2026-08-03) | No SDK — Adaptive Card POST |
| Have I Been Pwned — Pwned Passwords range API | https://haveibeenpwned.com/API/v3 | None; k-anonymity (first 5 SHA-1 hex chars); `Add-Padding: true` | No rate limit | Very good | One `fetch` to `https://api.pwnedpasswords.com/range/{prefix}` |
| Fonts: Fontsource variable packages (Manrope, Inter, JetBrains Mono) | https://fontsource.org/docs/getting-started/introduction | None (npm packages, build-time) | None | Good | `@fontsource-variable/manrope` 5.3.0, `/inter` 5.3.0, `/jetbrains-mono` 5.3.0 (registry-verified) |

**Key Takeaways:**
- One SMTP adapter (Nodemailer) is provider-neutral: Mailpit locally, SES/Postmark/Resend SMTP in production, configured by `SMTP_URL`.
- Chat webhooks are plain HTTPS POSTs with secret URLs; they are a v1 non-goal but the outbox design accommodates them later.
- Pwned Passwords is free, unauthenticated and privacy-preserving — run inline on password set/change with a short timeout and fail-open policy.
- Fonts need no external API at runtime; Fontsource packages self-host, keeping CSP `font-src 'self'`.
- All third-party calls are outbound from the API server behind the outbox (`FOR UPDATE SKIP LOCKED`), isolating provider limits from user latency.

### 3.9 — Architecture & Design Patterns

| Pattern | Source | Why It Applies |
|---------|--------|----------------|
| Shared-schema multi-tenancy: `org_id` on every row + Postgres RLS | https://www.postgresql.org/docs/current/ddl-rowsecurity.html · https://orm.drizzle.team/docs/rls | One schema is cheapest to operate/migrate at < 1k users; RLS is defense in depth so a missing `WHERE org_id` cannot leak data (OWASP A01). |
| Append-only audit log (actor, entity, action, before/after JSONB, request id) | https://supabase.com/blog/audit · https://appmaster.io/blog/tamper-evident-audit-trails-postgresql | Immutable audit requirement; one generic table covers all entities; REVOKE UPDATE/DELETE keeps it immutable. |
| Explicit state-machine transition table for ticket/change lifecycles | https://stately.ai/docs/state-machines-and-statecharts | Deterministic, auditable transitions drive SLA pause/resume and UI affordances. |
| SLA clock with pause conditions + business-hours calendar | https://support.atlassian.com/jira-service-management-cloud/docs/set-up-sla-calendars/ · https://www.servicenow.com/community/itsm-blog/understanding-my-slas-part-iii-pause-times/ba-p/2266894 | MTTR/SLA % need elapsed business time; "waiting on requester" pauses the clock. |
| Transactional outbox for notifications | https://microservices.io/patterns/data/transactional-outbox.html | Avoids dual-write between Postgres and email; an event exists iff the change committed. |
| SSE fan-out with per-tenant channels | https://html.spec.whatwg.org/multipage/server-sent-events.html · https://dev.to/uaslimcreate/postgresql-listennotify-for-real-time-multi-tenant-events-ditching-polling-and-websocket-54pk | Web-only, < 1k concurrent: SSE over HTTP/2 through Caddy is simpler than WebSockets; per-org channels prevent cross-tenant leakage. |
| Optimistic concurrency via `version` column / `If-Match` | https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html · https://httpwg.org/specs/rfc9110.html | Multiple agents edit the same ticket; version stamp prevents lost updates. |
| Keyset (cursor) pagination | https://use-the-index-luke.com/no-offset | Long, append-heavy, live lists; OFFSET skips/duplicates under inserts. |
| RBAC role→permission matrix | https://csrc.nist.gov/projects/role-based-access-control/role-engineering-and-rbac-standards | Check permissions (e.g. `change:approve`) not role names in code. |
| Idempotency-key table | https://datatracker.ietf.org/doc/html/draft-ietf-httpapi-idempotency-key-header · https://docs.stripe.com/api/idempotent_requests | Safe retries for POSTs (deferred in v1, ADR-020). |
| Rule engine as data (`rules(trigger, conditions JSONB, actions JSONB)`) | https://martinfowler.com/bliki/RulesEngine.html | Routing/escalation/notification are per-tenant config; keep the rule language small. |
| Soft delete for audit integrity | https://brandur.org/soft-deletion · https://brandur.org/fragments/deleted-record-insert | Entities referenced by audit rows must remain resolvable. |
| Same-origin API behind one reverse proxy vs BFF vs CORS | https://samnewman.io/patterns/architectural/bff/ · https://nextjs.org/docs/app/guides/backend-for-frontend · https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS | Same origin keeps the cookie `HttpOnly`/`SameSite=Strict` without CORS credentials; Caddy routes `/api` and `/` to one origin. |
| Explicit approval records consumed as state-machine guards | https://www.peoplecert.org/ITIL4-practices | Approvals need auditable decisions with expiry; modeled as their own entity. |

**Key Takeaways:**
- Tenancy and audit are enforced in the database (RLS + append-only JSONB audit), not only in application code.
- Lifecycle logic is data: transition table, SLA calendar, RBAC matrix, rule rows — every evaluation lands in the audit log.
- Reliability primitives are HTTP-native: `version` ↔ `If-Match`, RFC 9457 envelope, keyset cursors.
- Realtime is Postgres-centric: outbox → in-process SSE hub → per-org channels with `Last-Event-ID` replay — no Redis at < 1k concurrent.
- One origin behind Caddy keeps cookies first-party and avoids CORS entirely.

## 4. Technology Stack Candidates

| Layer | Option A | Option B | Recommendation | Rationale |
|-------|----------|----------|----------------|-----------|
| Language | TypeScript 6.0.3 | TypeScript 7.0.2 (native) | **A** | 7.0 has no Compiler API; typescript-eslint peer range `<6.1.0` (ADR-005) |
| Frontend | Next.js 16.3.4 App Router + Tailwind v4 | Next.js 15 | **A** | Hard constraint Next.js; 16 is current; `proxy.ts` CSP pattern documented |
| Backend | Express 5.2.1 + zod 4 + pino | Fastify | **A** | Hard constraint Express; v5 async errors; helmet/rate-limit ecosystem |
| ORM | Drizzle 0.45 + drizzle-kit SQL migrations | Prisma | **A** | SQL-first, small image, RLS-friendly `SET LOCAL`, runner parses uniqueness |
| Database | PostgreSQL 17 | PostgreSQL 16 | **A** | Hard constraint; 17 current major with RLS/SKIP LOCKED/tsvector |
| Realtime | SSE (native) | WebSockets (ws) | **A** | One-directional push suffices; rides the session cookie; no upgrade CSRF surface |
| Auth | Server-side sessions + Argon2id | JWT | **A** | Instant revocation; WEB.md forbids localStorage tokens |
| Hosting / infra | Docker Compose + Caddy (`tls internal` → ACME) | nginx + certbot | **A** | Hard constraint Compose; Caddy automatic HTTPS satisfies "HTTPS only" |
| Jobs | In-process scheduler + advisory lock | BullMQ + Redis | **A** | `small` tier forbids Redis/queues |
| Email | Nodemailer SMTP | Resend SDK | **A** | Provider-neutral; SDK adapters can be added later |
| Search | Postgres tsvector | Meilisearch | **A** | No extra service at `small` |
| Tests | vitest + supertest + Playwright | jest | **A** | Vitest 4 native ESM/TS; Playwright is the baseline runner's dependency anyway |
| CI/CD | GitHub Actions (verification workflow only) | — | **A** | `ci_cd_required` blank → verification workflow always emitted, no deploy job |

## 5. Risk & Unknowns Register

| ID | Unknown / Risk | Severity | Mitigation | Status |
|----|----------------|----------|------------|--------|
| R1 | Express 5 route syntax changes (path-to-regexp 8) break wildcard/optional routes | med | Use only literal and `:param` segments; no regex routes | resolved (design) |
| R2 | Next 16 `proxy.ts` CSP nonce must be set on request **and** response or hydration dies under `strict-dynamic` | high | Copy the reference pattern; Web Delivery Baseline asserts zero CSP violations | open until baseline passes |
| R3 | RLS context leaking across pooled connections | high | `SET LOCAL` inside `withOrgScope` transactions; policy uses `NULLIF(current_setting(...), '')`; integration test asserts reset | open until tested |
| R4 | Local TLS trust for audit/e2e browsers (`tls internal`) | med | Export Caddy root CA; `NODE_EXTRA_CA_CERTS` for Node; Chromium trust via policy or Linux Playwright image with CA installed | open (Stage 3 ADR) |
| R5 | argon2 native binary on Windows dev host / Alpine image | low | Prebuilt binaries verified for win32-x64 and linux-musl; never substitute bcryptjs | resolved |
| R6 | SSE buffering through Next rewrites (dev) or Caddy | med | Dev mode polls; Caddy `flush_interval -1`, no `encode` on `/api/v1/events/*`, initial comment | resolved (design) |
| R7 | TypeScript 7 tooling gap | low | Pin 6.0.3; revisit | resolved |
| R8 | Scheduler and HTTP share one event loop | low | Bounded sweeps (`LIMIT 50`), yield between steps; load test verifies p95 | open until load test |
| R9 | HIBP unreachable during registration | low | 3 s timeout, fail-open with a warning log; `HIBP_ENABLED=false` for air-gapped deploys | resolved (design) |
| R10 | Host ports 80/443/5432 already taken on the build machine | low | Caddy on 8443/8080, Postgres on 5433 (ADR-008) | resolved |
| R11 | Rule engine loops (rule A triggers rule B triggers A) | med | Recursion depth cap 3, per-run trace in `automation_runs` | resolved (design) |
| R12 | SLA math across DST boundaries | med | Pure `slaClock.ts` with timezone-aware tests including DST | open until tested |

**Threat-model assessment (comprehensive depth):**

| Asset | Attacker | Vector | Control |
|---|---|---|---|
| Tenant data (tickets, assets, users) | Authenticated user of another org | IDOR via guessed UUIDs, list endpoints without filters | RLS forced + explicit `org_id`; cross-org → 404 |
| Tenant data | Same-org requester | Reading others' tickets, private notes, audit trail via sub-resource routes | `load<Entity>` chokepoint on every `/:id…` route (INV-14); `serialize()` field gating |
| Credentials | Internet | Credential stuffing, enumeration, timing | Argon2id; `(ip,email)` limiter before verify (INV-8); uniform 401; dummy verify on unknown email; HIBP screening |
| Sessions | XSS / network | Token theft | HttpOnly/Secure/Strict cookie, hashed tokens, CSP Profile N, HSTS |
| Integrity of approvals/changes | Malicious agent | Self-approval, replayed decisions, concurrent edits | approver ≠ requester; `status='pending'` guard; `version` checks; audit events |
| Audit trail | Admin covering tracks | UPDATE/DELETE on audit rows | RLS grants INSERT/SELECT only; INV-5 in code |
| Availability | Internet | Login floods, SSE exhaustion, oversized bodies/headers | Limiters; 5 streams/user; 256 kB body; 32 kB header buffer |
| Secrets | Repo readers | Committed `.env` | `.gitignore`; INV-16; env schema without production defaults |
| Egress | Attacker-controlled URL | SSRF | No user-controlled URLs are fetched (webhooks are a non-goal); SMTP/HIBP hosts fixed by env |

## 6. Research Gaps

- [x] Gap 1: Chromium trust for Caddy's internal CA on Windows for the pinned baseline runner — resolved at Stage 3 by test (policy key first, Linux Playwright container as fallback); not a blocker for architecture.
- [x] Gap 2: Nodemailer current major (9.x per research; exact version pinned at install time from the registry).
- [x] Gap 3: Stack Overflow / Reddit primary threads were unreachable to the crawler; equivalent GitHub/HN/community threads were substituted.
- [ ] Gap 4: Whether `typescript-eslint` adds TS 7 support before ship — tracked as debt (ADR-005), not blocking.

**Blocker?** No — every open gap has a bounded resolution path inside Stage 3.

## 7. Summary & Recommendation

The constraints (Next.js, Express, PostgreSQL, Docker Compose, HTTPS only, multi-tenant, small scale, realtime) resolve to a well-trodden stack whose current versions were verified against vendor docs and the npm registry: Next.js 16 with a `proxy.ts`-minted CSP nonce, Express 5 with zod 4 validation and RFC 9457 errors, Drizzle over PostgreSQL 17 with forced row-level security and `SET LOCAL` tenant context, Server-Sent Events for push, and Caddy for automatic HTTPS. The open-source ITSM landscape confirms the feature scope (tickets, changes, assets, KB, SLA, approvals, automation, audit) while its copyleft licenses make it reference-only. Competitive research shows the market gap Orbit targets: full ITSM scope with modern UX and no admin-heavy configuration. The two notable version findings — TypeScript 7 lacking a Compiler API and NIST 800-63B-4's 15-character single-factor minimum — are absorbed as ADRs. Remaining risks are implementation-level (CSP nonce plumbing, RLS pooling, local TLS trust) and each has a test that proves it closed.

**Go / No-Go Decision:** `GO`

**Next Step:** → [ARCHITECTURE.md](./ARCHITECTURE.md)
