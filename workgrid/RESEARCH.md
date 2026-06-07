# RESEARCH.md — WorkGrid

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Team work"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_websocket", "has_dual_write"]
```

## Product Vision
**Problem:** Organizations often rely on disconnected tools, spreadsheets, email, and chat to manage work, resulting in poor visibility, missed deadlines, duplicated effort, and inconsistent processes. Teams need a single platform that organizes work, standardizes workflows, and provides real-time insight into execution.

**Who it affects:** IT and Service Desk teams managing incident response, change requests, and SLA-driven ticket queues with fragmented tools causing delayed escalations and lost context across handoffs. Project Managers coordinating cross-functional work across multiple concurrent initiatives, struggling with scattered status updates, resource conflicts, and inability to enforce consistent planning standards. Engineering and Development teams balancing sprint commitments with unplanned work, lacking visibility into dependencies and blocked tasks across repositories and tracking systems. Operations teams executing repetitive processes and runbooks across infrastructure, applications, and services with manual handoffs creating bottlenecks and audit gaps. Business stakeholders and leadership needing real-time portfolio visibility, budget tracking, and outcome metrics without direct access to underlying project data, forcing reliance on manual reporting and outdated dashboards.

**Why existing solutions fall short:** Many work management platforms are either too complex, too rigid, or too expensive. They require extensive customization, overwhelm users with features, and make it difficult to adapt workflows to different teams. As organizations grow, maintaining consistency and visibility across projects becomes increasingly challenging.

**Solution:** WorkGrid is a unified work management platform that centralizes planning, tracking, and execution across projects, tickets, tasks, and workflows. It eliminates tool fragmentation by giving teams one workspace to manage priorities, automate processes, and maintain end-to-end visibility from intake through completion.


## Users & Outcomes
**Key Workflows:**
**IT Service Desk Manager** creates a new incident ticket from email, auto-routes to on-call engineer based on severity and skills, and escalates unresolved P1s after 30 minutes—reducing mean time to response from 45 to 15 minutes.

**Project Manager** establishes a project with linked tasks, assigns dependencies across teams, and views critical path with resource conflicts highlighted—enabling on-time delivery within planned scope.

**Engineering Lead** configures a sprint workflow that surfaces blocked tasks when dependencies fail, auto-notifies dependent teams, and prevents sprint commitment overload—reducing sprint scope creep by 40%.

**Operations Engineer** executes a runbook as a templated task sequence with approval gates at each step, auto-escalates if manual steps exceed SLA, and logs audit trail—ensuring compliance and eliminating handoff delays.

**Executive** accesses portfolio dashboard showing project health, budget burn, and outcome metrics updated in real-time from underlying work data—eliminating weekly status meetings and enabling data-driven decisions.

**Service Desk Agent** receives auto-assigned ticket with full context from previous interactions, completes resolution steps in configured workflow, and closes with customer notification—reducing resolution time and repeat tickets.

**Product Manager** prioritizes backlog by impact and effort, moves items through intake-to-planning workflow with stakeholder approval, and tracks velocity—ensuring roadmap alignment and predictable delivery.

**Team Lead** monitors workload distribution across team members, identifies overallocation before sprint starts, and rebalances assignments—preventing burnout and maintaining sustainable pace.

**Success Metrics:**
Achieve 90% on-time project delivery within planned scope and budget. Reduce task and ticket backlog by 60% within 6 months through automated routing and prioritization. Increase team productivity by 35% measured by tasks completed per person per week. Achieve 75% workflow automation adoption across configured processes within 3 months. Provide real-time visibility with 95% of active work items updated within 24 hours and portfolio dashboards refreshing every 15 minutes. Reduce mean time to incident response from 45 to 15 minutes. Decrease sprint scope creep by 40%. Lower ticket resolution time by 50%. Eliminate weekly status meetings through automated executive reporting. Reduce repeat ticket rate by 45%. Achieve 99% audit trail completeness for compliance-critical workflows.

**Non-Goals:**
**Non-Goals:**

Infrastructure monitoring — WorkGrid tracks work about infrastructure but does not collect metrics, alerts, or dashboards from monitoring systems; teams integrate monitoring tools via API for context only.

Endpoint management — WorkGrid does not provision, patch, or manage devices; it organizes work tickets related to endpoint issues but delegates execution to dedicated MDM platforms.

Security event monitoring — WorkGrid does not ingest, analyze, or alert on security events; it routes security incidents as tickets once detected by SIEM or security tools.

Source code management — WorkGrid does not host repositories, manage branches, or track commits; it links to external repos and surfaces PR/build status as work context without replacing Git workflows.

Financial or ERP management — WorkGrid does not manage general ledger, procurement, or accounting; it tracks budget allocation and burn for projects but requires export to financial systems for reconciliation and reporting.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "application performance monitoring"
backup_strategy: "daily automated backups with point-in-time recovery"
container_strategy: "Docker with Kubernetes orchestration"
error_reporting: "centralized error tracking with alerting"

# Data & Storage
database_preference: "PostgreSQL"
pii_handling: ["user identities", "email addresses", "task/ticket content"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"
webhook_targets: ["GitHub", "Slack"]

# Security & Compliance
security_baseline: ["SOC2", "OWASP Top 10"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables with encryption"

# Frontend
frontend_framework: "Next.js"
ui_component_library: "shadcn/ui"
css_approach: "Tailwind CSS"
state_management: "TanStack Query with Zustand"

# Backend
backend_framework: "NestJS"
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "TypeORM"
realtime_needed: true
background_jobs: "Bull with Redis"

# Performance & Quality
performance_requirements: ["API response < 200ms", "dashboard load < 2s", "real-time updates < 1s"]
testing_strategy: "TDD"
logging_format: "structured JSON"

# Scope & Platform
scale: "large — 50k+"
multi_tenant: true
target_platforms: ["web"]
alert_channels: ["email", "in-app notification"]
report_formats: ["PDF", "CSV", "JSON"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Clear, organized, and execution-focused. WorkGrid empowers teams to move work forward with structure, visibility, and accountability while keeping workflows simple and intuitive.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #a92fe1 | Accents, badges, highlights |
| Accent | #dcb04f | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #16181d | Headings, body text |
| Text Secondary | #5c6270 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #ac36e2 | Accents, badges, highlights |
| Accent | #ddb255 | Callouts, hover states |
| Background | #090a0c | Page background |
| Surface | #121317 | Cards, elevated containers |
| Text | #eaeaec | Headings, body text |
| Text Secondary | #838995 | Captions, muted text |
| Success | #33cc6b | Success states, confirmations |
| Warning | #e2a336 | Warnings, pending states |
| Error | #d74242 | Errors, destructive actions |

### Typography
- Heading: Space Grotesk (600/700 weight)
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
- Variant: Minimal
- Border radius: 6px (sm: 2px, lg: 10px, xl: 14px)
- Shadows: None — `none`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

An HTML design template was uploaded with this project. Fetched at build time
via `get_design_template` and saved to the project root as `DESIGN-TEMPLATE.html`
(filename: "WorkGrid (standalone).html", sha256
2f57eaa66f6da6767860a2b7e685cee0de62be87fee6c5df36789e8c5373e849). Its `:root`
tokens are the canonical design-token set and override the Design Language tokens
above wherever they conflict (see DECISIONS.md ADR-0002 for the copied block).


## 3. Source Categories

### 3.1 Official / Vendor Documentation

- **Next.js 16 Release / Upgrade Guide** — https://nextjs.org/blog/next-16 — Confirms Next.js 16.x is the current major (latest stable 16.2.7 LTS, June 2026); Turbopack is the default bundler for `next dev`/`next build`, React Compiler support and `cacheLife`/`cacheTag` are stable. App Router is the supported paradigm.
- **NestJS Docs — Versioning** — https://docs.nestjs.com/techniques/versioning — Native URI/URL-path versioning via `app.enableVersioning({ type: VersioningType.URI })`, yielding `/v1/` route prefixes. Latest core is `@nestjs/core` 11.1.x on Express v5.
- **TypeORM 1.0 Changelog** — https://orkhan.gitbook.io/typeorm/changelog — TypeORM v1.0.0 (2026-05-19), first major in ~5 years; ES2023, minimum Node.js 20. `DataSource`, repositories, migrations stable.
- **PostgreSQL Release News** — https://www.postgresql.org/about/news/ — PostgreSQL 18.4 (2026-05-14) current stable minor on the 18 line. Authoritative for RLS, GUC `current_setting()`, logical replication for PITR.
- **Resend SDK / Docs** — https://resend.com/docs/sdks — Official Node SDK (`resend`, latest 6.12.x); `Authorization: Bearer re_...` API-key auth, idempotency keys, batch send, inbound/webhook event signing via Svix.
- **GitHub — Validating webhook deliveries** — https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries — `X-Hub-Signature-256: sha256=<hmac>` (HMAC-SHA256 hex over raw UTF-8 body), constant-time compare.
- **Slack — Verifying requests from Slack** — https://docs.slack.dev/authentication/verifying-requests-from-slack/ — Signing base `v0:{timestamp}:{body}`, `X-Slack-Signature: v0=<hmac>` (HMAC-SHA256), `X-Slack-Request-Timestamp` replay window ±5 min.

### 3.2 GitHub Repositories

- **vercel/next.js** — https://github.com/vercel/next.js — ~135k★, active daily, MIT. App Router data fetching, route handlers, middleware for HTTPS-only edge auth.
- **nestjs/nest** — https://github.com/nestjs/nest — ~73k★, active daily, MIT. v11 line, Express 5 default; source of truth for guards/interceptors/gateways.
- **typeorm/typeorm** — https://github.com/typeorm/typeorm — ~36k★, active weekly, MIT. v1.0.0 (ES2023/Node 20); migrations, query builder, transactions, optimistic locking via `@VersionColumn`.
- **taskforcesh/bullmq** — https://github.com/taskforcesh/bullmq — ~7k★, active daily, MIT. BullMQ 5.78.x; Bull is maintenance-only, BullMQ is the actively-developed successor — SLA escalation jobs, delayed/repeatable jobs, flows.
- **pmndrs/zustand** — https://github.com/pmndrs/zustand — ~52k★, active weekly, MIT. v5.0.14; minimal client-state store complementing TanStack Query.

### 3.3 Video / Tutorials

- **"Tailwind CSS v4 Full Course 2026"** — https://www.youtube.com/watch?v=6biMWgD6_JY — v4 CSS-first config (`@theme`, no `tailwind.config.js` required), relevant for Tailwind v4.x + shadcn/ui.
- **"Socket.IO: The Complete Guide to Building Real-Time Web Applications (2026 Edition)"** — https://dev.to/abanoubkerols/socketio-the-complete-guide-to-building-real-time-web-applications-2026-edition-c7h — Rooms/namespaces and the Redis adapter for horizontal fan-out, applicable to multi-pod real-time.

### 3.4 Articles / Blog Posts

- **AWS — Multi-tenant data isolation with PostgreSQL Row Level Security** — https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/ — Pooled-connection + session-variable (`SET app.current_tenant_id`) RLS pattern; production-grade approach over per-tenant DB users.
- **Crunchy Data — Row Level Security for Tenants in Postgres** — https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres — RLS policy authoring, `FORCE ROW LEVEL SECURITY`, verifying the tenant predicate via `EXPLAIN`.

### 3.5 Standards / RFCs

- **RFC 9457 — Problem Details for HTTP APIs** — https://www.rfc-editor.org/info/rfc9457/ — Obsoletes RFC 7807; `application/problem+json` error envelope (`type`/`title`/`status`/`detail`/`instance`). Standard for our REST error contract.
- **OWASP Top 10 / ASVS** — https://owasp.org/www-project-top-ten/ — Mandated baseline; injection (parameterized TypeORM), broken access control (RLS + RBAC), SSRF (outbound webhook URL allowlists).
- **NIST SP 800-63B — Digital Identity / Authentication** — https://pages.nist.gov/800-63-3/sp800-63b.html — Password/verifier storage (Argon2id), session lifetime, MFA; feeds the auth boundary.
- **GitHub webhook signature spec (HMAC-SHA256)** — https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries — `X-Hub-Signature-256`.
- **Slack request signing spec (HMAC-SHA256, v0)** — https://docs.slack.dev/authentication/verifying-requests-from-slack/ — `X-Slack-Signature` + timestamp replay protection.
- **WCAG 2.2 (AA)** — https://www.w3.org/TR/WCAG22/ — Accessibility target; shadcn/ui (Radix primitives) supports compliance.

### 3.6 Competing Products

- **Jira / Jira Service Management** — Learn: configurable workflows with transition conditions/validators/post-functions, JQL filtering, SLA goals with calendars/pause conditions. Caution: configuration complexity — keep workflow config opinionated.
- **ServiceNow** — Learn: enterprise ITSM/incident with SLA definitions + escalation chains, approval gates, strong audit trail. Reference for SLA escalation scheduling and approval-gate state machine.
- **Linear** — Learn: best-in-class latency, optimistic UI with real-time sync; informs optimistic-concurrency + WebSocket fan-out UX target. Keyboard-first, minimal-surface ethos for shadcn/ui screens.
- **Asana / Monday.com** — Learn: flexible project/task views (board/list/timeline), custom fields, executive dashboard widgets — model for dashboard aggregation queries.

### 3.7 Community Threads

- **GitHub Community — "Unable to validate X-Hub-Signature-256 from webhook" #24646** — https://github.com/orgs/community/discussions/24646 — Signature must be computed over the raw, unparsed body; JSON re-serialization breaks HMAC. Drives raw-body middleware design.
- **NestJS docs — Rate Limiting** — https://docs.nestjs.com/security/rate-limiting — `@nestjs/throttler` (v6.x) with multiple named throttlers and Redis storage for distributed limits across pods.

### 3.8 APIs / Integrations

- **Resend API** — https://resend.com/docs/sdks — Auth `Authorization: Bearer re_<key>`. Idempotency keys (safe worker retries) and batch endpoints. Inbound/event webhooks signed via Svix (`svix-id`/`svix-timestamp`/`svix-signature`) — verify, do not trust.
- **GitHub Webhooks + REST API** — https://docs.github.com/en/webhooks — Inbound: verify `X-Hub-Signature-256` (HMAC-SHA256, `sha256=` prefix). Outbound REST: token/App auth, paginated, rate-limited (respect `X-RateLimit-Remaining`/`Retry-After`).
- **Slack Webhooks / Events API** — https://docs.slack.dev/authentication/verifying-requests-from-slack/ — Inbound: verify `X-Slack-Signature` (`v0=`, base `v0:ts:body`) + ±5-min window; respond to `url_verification`. Outbound: Incoming Webhooks (per-channel URL secret) or `chat.postMessage` with bot token.

### 3.9 Patterns

- **Multi-tenant isolation via PostgreSQL RLS + session GUC** — Every tenant table carries `tenant_id`; `ENABLE`/`FORCE ROW LEVEL SECURITY` with policy `tenant_id = current_setting('app.current_tenant_id')::uuid`; app sets the GUC per-request on a pooled connection. DB enforces isolation even if an app query forgets the filter.
- **Transactional Outbox (avoid dual-write)** — Persist domain change + outbox row in one DB transaction; a BullMQ relay publishes to Slack/GitHub/Resend and marks sent. Eliminates "DB committed but webhook lost / webhook sent but DB rolled back."
- **SLA escalation scheduling** — BullMQ delayed + repeatable jobs keyed to the ticket; re-evaluate on state change (cancel/replace job) rather than busy-polling. Idempotent reconciliation sweep recomputes breaches from `due_at`.
- **Optimistic concurrency** — TypeORM `@VersionColumn`; updates carry the expected version; conflicting writes get 409 (problem+json). Pairs with optimistic-UI.
- **RBAC** — Roles/permissions enforced in NestJS guards layered on top of RLS tenant scoping (authorization above isolation); permissions checked per-resource, audit-logged.
- **Real-time fan-out** — Socket.IO with the Redis adapter so events broadcast across pods to tenant-scoped rooms (`room = tenant:{id}`); never broadcast globally.

## 4. Stack Candidates

| Layer | Chosen | Verified Version (mid-2026) | Justification / Alternative |
|---|---|---|---|
| Frontend framework | Next.js (App Router) | 16.2.x LTS | Turbopack-default, RSC, stable caching; HTTPS/SSR. Alt: Remix — less batteries-included. |
| UI components | shadcn/ui + Radix | CLI-distributed | Copy-in, a11y-first (WCAG 2.2), full control. Alt: MUI — heavier. |
| CSS | Tailwind CSS | 3.4.x (stable, pinned) | Mature, shadcn-compatible config. (v4 evaluated; pin v3.4 for shadcn/ui ecosystem stability.) |
| Server state | TanStack Query | 5.x | Caching/invalidation/optimistic updates. Alt: SWR. |
| Client state | Zustand | 5.0.x | Minimal UI state. Alt: Redux Toolkit. |
| Backend framework | NestJS | 11.x | DI, guards/interceptors, native URI versioning (`/v1/`). Alt: Fastify-standalone. |
| API style | REST + URL-path versioning | `VersioningType.URI` | Constraint; native NestJS support. |
| ORM | TypeORM | 0.3.x (pinned, Node 20+) | `@VersionColumn` optimistic locking, migrations, RLS-compatible session GUC. Alt: Prisma — weaker raw RLS control. |
| Real-time | @nestjs/websockets + socket.io | socket.io 4.8.x | Gateways + Redis adapter for fan-out. Alt: raw `ws`. |
| Background jobs | @nestjs/bullmq + BullMQ | BullMQ 5.x | Actively maintained; delayed/repeatable jobs for SLA. |
| Redis client | ioredis | 5.x (`maxRetriesPerRequest: null`) | Cluster support, BullMQ-recommended. |
| Database | PostgreSQL | 16/18 | RLS multi-tenancy, logical replication for PITR, JSONB custom fields. |
| Email | Resend SDK | 6.x | Bearer-key auth, idempotency, batch; Svix-signed inbound. |
| Auth hashing | argon2 | 0.4x (Argon2id) | NIST 800-63B memory-hard. Alt: bcrypt. |
| Validation | class-validator | 0.14/0.15 | DTO validation via NestJS `ValidationPipe`. |
| Security headers | helmet | 8.x | CSP/HSTS for HTTPS-only baseline. |
| Rate limiting | @nestjs/throttler | 6.x | Named throttlers + Redis storage for distributed limits. |

> Version note: exact patch versions are pinned in the package manifests at build time (lockfiles committed). TypeORM/Tailwind are pinned to the stable lines proven with the NestJS/shadcn ecosystems rather than newest-major to reduce integration risk; this is logged as ADR-0003.

## 5. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Multi-tenant data leakage | Medium | Critical | RLS `FORCE` on all tenant tables + per-request `tenant_id` GUC; never trust app-layer filtering alone; automated cross-tenant access tests in CI. |
| Webhook spoofing (inbound) | Medium | High | Verify HMAC over raw body (GitHub `X-Hub-Signature-256`, Slack `v0=` + timestamp window, Resend/Svix); constant-time compare; reject on mismatch/stale timestamp. |
| SLA scheduler reliability / clock drift | Medium | High | BullMQ delayed/repeatable jobs + idempotent reconciliation sweep recomputing breaches from `due_at`; Redis AOF; alert on worker lag. |
| Dual-write inconsistency (DB vs webhook/email) | Medium | High | Transactional outbox: domain change + outbox row atomically; relay worker delivers with idempotency keys + retries/DLQ. |
| Real-time scaling (multi-pod fan-out) | Medium | Medium | Socket.IO Redis adapter; tenant-scoped rooms; sticky sessions / connection-aware LB; backpressure limits. |
| PII exposure | Medium | High | Encrypt at rest, TLS in transit (HTTPS only), field minimization, audit-logged access, redact PII from logs/APM. |
| Rate-limit bypass / DoS | Medium | High | `@nestjs/throttler` with Redis store, per-tenant + per-IP limits, ingress limits, honor `Retry-After` outbound. |
| Secret management | Medium | High | Env vars via K8s Secrets / external manager, no secrets in repo/images, rotation, per-integration signing secrets. |
| Dependency CVEs | High | Medium | Pin + lockfile, automated SCA in CI, scheduled patch cadence, SBOM. |

### Threat Model

**Trust boundaries:** (1) Browser ↔ Next.js/API; (2) tenant ↔ tenant within shared DB/Redis; (3) external SaaS (GitHub/Slack/Resend) ↔ inbound webhook receiver; (4) WebSocket client ↔ real-time gateway; (5) app ↔ DB/Redis/secrets.

- **Auth boundary (STRIDE).** Spoofing: short-lived signed JWT/session, Argon2id storage, MFA-ready. Tampering: signature-verified tokens, HTTPS only. Repudiation: audit log every auth/privileged action. Info disclosure: generic auth errors, no user enumeration, helmet HSTS/CSP. DoS: throttler on login/reset. Elevation: RBAC guards above RLS; deny-by-default.
- **Tenant boundary (STRIDE).** Spoofing tenant context: `tenant_id` derived from authenticated session only — never client-supplied. Tampering: RLS `FORCE` so buggy queries can't cross tenants; GUC set server-side per request. Info disclosure: RLS backstop; cross-tenant CI tests; pooled connections reset GUC. Elevation: role checks scoped to tenant.
- **Inbound-webhook boundary (STRIDE).** Spoofing: mandatory HMAC verification over raw body. Tampering: signature covers body. Replay: Slack timestamp ±5-min + delivery-id dedup. DoS: size/rate limits, fast pre-parse reject. Info disclosure: generic 401/403.
- **Real-time channel (STRIDE).** Spoofing: authenticate WS handshake, bind socket to tenant room. Tampering/Info disclosure: emit only to `tenant:{id}` rooms; authorize each subscription. DoS: per-connection message caps, max connections per tenant. Elevation: re-check RBAC on subscribe/emit.

## 6. Research Gaps

1. **RLS + connection pooling at scale** — Validate per-request `SET app.current_tenant_id` GUC is reliably reset on pooled connections (PgBouncer transaction vs session mode) under 50k-user load; benchmark RLS planner overhead with `EXPLAIN ANALYZE`.
2. **Socket.IO horizontal scaling ceiling** — Confirm standard Redis adapter vs `@socket.io/redis-streams-adapter` for our pod count; quantify fan-out latency before committing to sticky-session LB config.
3. **@nestjs/bullmq freshness** — Wrapper lags BullMQ core; verify peer-dependency compatibility at build time and pin accordingly.
4. **Resend deliverability + inbound signing** — Confirm current Svix signature header set/rotation for inbound webhooks; validate DKIM setup against 50k-user notification volume.

## 7. Summary + GO / NO-GO

The constrained stack is fully realizable on current, verified-stable releases: Next.js 16.2 LTS, NestJS 11 (native `/v1/` URI versioning), TypeORM on Node 20+, PostgreSQL 18 (RLS for tenant isolation), BullMQ for SLA/outbox jobs, Socket.IO 4.8 with the Redis adapter for fan-out, and Resend for email. All security primitives are first-class and standards-aligned (Argon2id per NIST 800-63B, helmet, @nestjs/throttler, HMAC webhook verification per GitHub/Slack/Svix, RFC 9457 error envelope). The principal engineering risks — cross-tenant leakage, dual-write consistency, webhook spoofing — all have well-established, documented mitigations (RLS `FORCE`, transactional outbox, raw-body HMAC). Open gaps are operational tuning, not feasibility blockers.

`Go/No-Go: GO` — every layer is confirmed on a current stable version with documented best-practice patterns for the SOC2/OWASP and multi-tenant requirements; remaining items are build-time validations, not unknowns.
