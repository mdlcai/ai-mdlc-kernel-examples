# RESEARCH.md — ServiceHub

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Productivity"

## Domain Signals

```yaml
domain_signals: ["has_image_uploads", "has_webhooks", "has_websocket"]
```

## Product Vision
**Problem:** IT requests are often submitted through email, chat, phone calls, or spreadsheets, resulting in lost requests, inconsistent prioritization, limited visibility, and inefficient support processes. IT teams need a structured system to manage incidents, service requests, and escalations at scale.

**Who it affects:** Employees across the organization (end-users submitting IT support requests via email, chat, phone, or ad-hoc channels) who experience friction when reporting issues, lack visibility into ticket status, and receive inconsistent response times due to fragmented request routing.

Help Desk and Service Desk agents who manually triage incoming requests from multiple channels, spend excessive time searching for ticket history across systems, struggle with unclear prioritization rules, and lack real-time workload visibility to balance incoming volume fairly across team members.

Infrastructure and Operations teams who inherit tickets without sufficient context, cannot easily identify recurring infrastructure issues or patterns that indicate systemic problems, and lack automation to route tickets to the right specialist, resulting in repeated handoffs and extended resolution times.

Security teams who need rapid incident escalation pathways for security-related requests, require audit trails and compliance documentation for all security ticket activity, and currently lack integration with their existing security tools and alert systems.

IT Managers and Leadership who cannot access consolidated reporting on team performance, SLA compliance, and recurring issue categories, struggle to justify staffing decisions or identify training needs, and lack data-driven insights to optimize resource allocation and reduce repeat incidents.

**Why existing solutions fall short:** Many IT ticketing platforms are overly complex, expensive, and require significant customization before delivering value. Users often struggle with cluttered interfaces, confusing workflows, and slow ticket submission processes, leading to poor adoption and increased support overhead.

For IT teams, existing solutions frequently include unnecessary features, fragmented workflows, and cumbersome administration. Reporting, automation, and SLA management can be difficult to configure, while licensing costs often scale aggressively as organizations grow.

This platform focuses on simplicity, speed, and operational efficiency by providing streamlined ticket management, intuitive workflows, automated routing, and actionable reporting without the complexity of traditional enterprise ITSM solutions.

**Solution:** A centralized IT service management platform that consolidates employee requests, incident reports, and support interactions into a single interface, eliminating fragmented ticketing and reducing time-to-resolution. ServiceHub streamlines service delivery by giving IT teams real-time visibility into workload distribution, performance metrics, and recurring issue patterns, enabling proactive problem-solving and faster incident response. The platform's core differentiator is its unified request-to-resolution workflow that simultaneously improves employee experience through transparent ticket tracking and status updates while equipping IT operations with the data and automation tools needed to optimize resource allocation and reduce repeat incidents.


## Users & Outcomes
**Key Workflows:**
**Employee submits an IT support request through the self-service portal and receives immediate confirmation with transparent tracking.** An employee experiencing a software license issue accesses ServiceHub's web portal, selects "Software & Licensing" from the categorized request menu, describes the problem in a structured form with optional attachment capability, and submits within 90 seconds. The system instantly generates a ticket number, displays expected resolution timeframe based on historical SLA data for that request type, and sends a confirmation email with a direct link to real-time ticket status. Outcome: Employee reduces submission friction from multi-channel uncertainty to single-step clarity, eliminating lost requests and enabling self-service visibility into ticket progression without requiring agent follow-up.

**Help Desk agent receives automatically routed and prioritized tickets with full context, reducing triage time and handoff cycles.** A Help Desk agent logs into ServiceHub and views an incoming queue pre-sorted by priority (critical security incidents flagged first, then standard requests ordered by submission time and SLA urgency). Each ticket displays the employee's request history, related open tickets, and system-detected category tags (e.g., "password reset," "hardware provisioning"). The agent resolves 40% of tickets directly from the portal using templated responses and knowledge base articles, or routes remaining tickets to Infrastructure or Security teams with a single click, automatically notifying the assigned specialist with full ticket context. Outcome: Agent reduces average triage time from 8 minutes to 2 minutes per ticket, eliminates manual context-gathering across fragmented systems, and decreases repeat handoffs by 60% through pre-populated specialist routing.

**Infrastructure Operations team identifies and resolves recurring systemic issues using automated pattern detection and consolidated reporting.** An Infrastructure Operations lead accesses ServiceHub's analytics dashboard and discovers that 34% of tickets submitted in the past two weeks relate to VPN connectivity failures, concentrated between 9–11 AM. The system automatically flags this pattern, suggests root-cause categories (e.g., authentication service degradation, network capacity), and generates a pre-filtered ticket cohort for investigation. The team uses this data to schedule a maintenance window, implement a temporary workaround, and document the resolution in a knowledge base article that ServiceHub automatically links to future similar requests. Outcome: Team reduces repeat incident volume by 70% for that issue category within one sprint, shifts from reactive firefighting.
- W1 — A requester submits an IT request, comments on it, attaches
  a file, and tracks status to resolution, seeing only their own
  tickets — never another employee's.
  - W2 — An agent picks up a ticket, adds an internal note (not
  visible to the requester), and resolves it.
  - W3 — A requester attempting to open another employee's ticket or
  its attachments is denied (404).
  - W4 — A manager reviews the team queue and SLA breaches across the
  org.

**Success Metrics:**
Reduced mean time to resolution (MTTR) from current baseline of 18 hours to 6 hours or less for standard requests and 2 hours or less for critical incidents within 90 days of platform adoption, measured across all ticket categories and tracked weekly via ServiceHub's analytics dashboard.

Improved SLA compliance to 95% or higher for all ticket priority tiers (critical incidents resolved within 2 hours, high-priority requests within 8 hours, standard requests within 24 hours, low-priority requests within 72 hours) measured as percentage of tickets meeting their assigned SLA threshold, reported daily by ticket category and assigned team.

Increased first-contact resolution rate from current baseline of 28% to 55% or higher within 120 days, defined as tickets closed by initial Help Desk agent without escalation or reassignment, tracked per agent and aggregated by request category to identify knowledge gaps.

Reduced ticket backlog from current average of 340 open tickets to 120 or fewer open tickets at any given time, with no ticket remaining unassigned for more than 30 minutes during business hours, measured daily at 9 AM and reported weekly by priority level and assignment status.

Higher end-user satisfaction scores with a target of 4.2 out of 5.0 or higher on post-resolution surveys (minimum 60% survey response rate), measured across all request categories, with sub-scores tracked for submission ease (target 4.5+), status transparency (target 4.3+), and resolution quality (target 4.1+), reported monthly with drill-down by department and request type to identify systemic friction points.

**Non-Goals:**
Remote desktop management
Infrastructure monitoring
Endpoint management and patching
Network performance monitoring
Security event monitoring (SIEM)


## Build Constraints

```yaml
# Infrastructure & Ops
hosting_environment: "cloud platform with autoscaling"
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "metrics + alerting"
backup_strategy: "daily automated backups with point-in-time recovery"
container_strategy: "orchestrated / multi-instance"
error_reporting: "centralized error tracking with alerting"

# Data & Storage
database_preference: "PostgreSQL"
data_retention_policy: "retain all ticket data indefinitely for audit and historical analysis"
pii_handling: ["employee names and contact info", "ticket content and attachments"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"

# Security & Compliance
security_baseline: ["SOC2", "OWASP Top 10"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables with encryption"

# Frontend
frontend_framework: "Next.js"
ui_component_library: "shadcn/ui"
css_approach: "Tailwind CSS"
state_management: "TanStack Query"

# Backend
backend_framework: "NestJS"
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "TypeORM"
realtime_needed: true
background_jobs: "BullMQ"

# Performance & Quality
performance_requirements: ["ticket submission < 2s", "dashboard load < 1.5s", "search results < 500ms", "MTTR target 6 hours for standard requests", "SLA compliance 95%"]
testing_strategy: "TDD"
logging_format: "structured JSON"

# Scope & Platform
scale: "medium — 1k-50k concurrent"
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

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

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

### Layout
- Pattern: Sidebar + Content
- Max width: 1440px, sidebar: 224px
- Spacing: Comfortable (12/16/24/32px)
- Breakpoints: 640 / 768 / 1024 / 1280px

### Component Style
- Variant: Rounded
- Border radius: 8px (sm: 4px, lg: 12px, xl: 16px)
- Shadows: Subtle — `0 1px 3px rgba(0,0,0,0.1)`
- Theme: Light + Dark with runtime toggle following prefers-color-scheme

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px

---

# §3 — Source Categories

> Stage 0 research performed 2026-06-08 (build_depth: comprehensive). All versions/stats fetched and confirmed live (npm / GitHub API / vendor docs). No URLs or versions invented. Stack is fixed by the customer Build Constraints; research verifies current versions and patterns rather than re-selecting the stack.

## §3.1 Official / Vendor Documentation (versions confirmed 2026-06-08)

| Tech | Doc URL | Confirmed stable | Key-API relevance |
|------|---------|------------------|-------------------|
| NestJS | https://docs.nestjs.com/ | 11.1.x (`@nestjs/core` 11.1.26) | `CanActivate` Guards + `@UseGuards()` are the RBAC enforcement point; custom `RolesGuard` + `Reflector` for tenant-scoped role checks. |
| Next.js (App Router) | https://nextjs.org/docs/app | 16.2.x | App Router default; Server Components + Route Handlers; Turbopack default for dev/build. |
| TypeORM | https://typeorm.io/ · changelog https://orkhan.gitbook.io/typeorm/changelog | 1.0.0 (2026-05-19) | `DataSource` + migration files; `migration:generate`/`migration:run`; never `synchronize:true` in prod. |
| BullMQ | https://docs.bullmq.io/ | 5.78.x | `Queue` (producer) + `Worker` (consumer) over Redis; repeatable/delayed jobs for SLA timers + email retries. |
| @nestjs/bullmq | https://docs.nestjs.com/techniques/queues | 11.0.4 | `BullModule.registerQueue()` + `@Processor`/`@WorkerHost`. |
| PostgreSQL | https://www.postgresql.org/docs/ | 18.x (18.4 current minor; 19 in beta, do not adopt) | Row-Level Security `ALTER TABLE … ENABLE ROW LEVEL SECURITY` + `CREATE POLICY … USING (tenant_id = current_setting('app.tenant_id'))` — hard isolation layer. |
| Resend Node SDK | https://resend.com/docs/send-with-nodejs · https://resend.com/docs/api-reference/emails/send-email | 6.12.x | `resend.emails.send({ from, to, subject, html })` → `{ data, error }`; supports idempotency keys. `POST https://api.resend.com/emails`, `Authorization: Bearer re_…`. |
| TanStack Query v5 | https://tanstack.com/query/latest | 5.101.x | `useQuery({ queryKey, queryFn })`, `useMutation` + `queryClient.invalidateQueries` for cache sync after writes. |
| shadcn/ui | https://ui.shadcn.com/docs | CLI v4 (Mar 2026), copy-in | Component primitives copied into repo (not a dependency); must be restyled to ServiceHub tokens (anti-slop: no stock theme). |
| Tailwind CSS | https://tailwindcss.com/docs | 4.3 (2026-05-08) | CSS-first `@theme`; logical properties; faster incremental builds. |
| class-validator | https://github.com/typestack/class-validator | 0.15.1 | Decorator DTO validation; global `ValidationPipe({ whitelist: true, transform: true })`. |
| @nestjs/passport | https://docs.nestjs.com/recipes/passport | 11.0.5 | `AuthGuard('jwt')` + `PassportStrategy`. |
| @nestjs/jwt | https://docs.nestjs.com/security/authentication | 11.0.2 | `JwtService.signAsync`/`verifyAsync`; refresh-token rotation. |
| @nestjs/websockets + socket.io | https://docs.nestjs.com/websockets/gateways | 11.1.x / socket.io 4.8.3 | `@WebSocketGateway()` + `@SubscribeMessage()`; Redis adapter to scale across pods. |

Version caveats: `@nestjs/passport` (11.0.5), `@nestjs/jwt` (11.0.2), `@nestjs/bullmq` (11.0.4), `socket.io` (4.8.3) are stable, last-published several months ago (normal for mature packages). Target PostgreSQL 18 GA. shadcn starters on this exact stack are small/dated — layout references only.

## §3.2 GitHub Reference Implementations (stats via GitHub API, 2026-06-08)

| Repo | Stars | Last active | License | Relevance |
|------|-------|-------------|---------|-----------|
| chatwoot/chatwoot | ~30,033 | 2026-06-09 | Custom (MIT-based) | Production helpdesk — conversation/ticket modeling, inbox UX, multi-agent assignment. |
| zammad/zammad | ~5,665 | 2026-06-08 | AGPL-3.0 | Mature ITSM — SLA, escalation, ticket-state machine, RBAC roles. |
| freescout-help-desk/freescout | ~4,320 | 2026-06-07 | AGPL-3.0 | Lightweight helpdesk — mailbox/email-to-ticket threading. |
| Peppermint-Lab/peppermint | ~3,131 | 2025-09-21 | Custom (AGPL-flavored) | TS self-hosted helpdesk (Next.js+Node) — closest stack analog; ticket schema + markdown editor. |
| ejazahm3d/fullstack-turborepo-starter | ~586 | 2023-08 (stale) | MIT | NestJS + Next.js + Tailwind + Docker monorepo layout reference. |
| stelescuraul/rls | ~72 | 2026-01-13 | ISC | RLS package for TypeORM + NestJS — directly relevant to Postgres-RLS multi-tenancy (maintained fork of Avallone-io/rls). |
| taskforcesh/bullmq | ~8,973 | 2026-06-08 | MIT | Canonical BullMQ source + queue/worker example patterns. |
| resend/resend-node | ~914 | 2026-06-05 | MIT | Official Resend Node SDK source + examples. |

## §3.3 Video / Tutorials
- NestJS official courses (https://courses.nestjs.com/) — Guards, interceptors, queues.
- TanStack Query v5 official examples (https://tanstack.com/query/latest/docs/framework/react/examples/simple) — mutation + invalidation patterns used for ticket list cache.

## §3.4 Articles / Engineering Blogs
**Multi-tenant PostgreSQL isolation**
- AWS DB Blog — Multi-tenant data isolation with PostgreSQL RLS: https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/ — canonical RLS + `current_setting()` SaaS Factory pattern.
- AWS samples (working RLS code): https://github.com/aws-samples/aws-saas-factory-postgresql-rls
- PlanetScale — Approaches to tenancy in Postgres: https://planetscale.com/blog/approaches-to-tenancy-in-postgres — trade-off table shared-schema/RLS/schema-per-tenant.
- The Nile — Multi-tenant SaaS using Postgres RLS: https://www.thenile.dev/blog/multi-tenant-rls — secure-by-default argument.
- Rico Fritzsche — Mastering PostgreSQL RLS: https://ricofritzsche.me/mastering-postgresql-row-level-security-rls-for-rock-solid-multi-tenancy/ — `SET LOCAL` + transaction scoping (TypeORM pool-leak relevance).
- Logto — Multi-tenancy with PostgreSQL: https://blog.logto.io/implement-multi-tenancy — shared-schema + tenant_id walkthrough.

**IDOR / object-level authorization (same-tenant cross-user)**
- OWASP IDOR: https://owasp.org/www-community/attacks/insecure_direct_object_reference
- OWASP IDOR Prevention Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html
- OWASP WSTG Testing for IDOR: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References
- API1 Broken Object Level Authorization (BOLA): https://apisecurity.io/encyclopedia/content/owasp/api1-broken-object-level-authorization.htm

**SLA timer / breach computation**
- AppMaster — SLA timers & escalations: https://appmaster.io/blog/sla-timers-escalations-workflow-apps — single idempotent SLA evaluator computing `due_at` + OK/AtRisk/Breached; `pause_started_at`/`paused_total_seconds`.
- Hiver — SLAs in ticketing software: https://hiverhq.com/blog/slas-in-ticketing-software — business-hours pausing.
- Giva — SLA breach root-cause / Watermelon Effect: https://www.givainc.com/blog/sla-breach-root-cause-analysis/

**ITSM ticket lifecycle / state machine**
- ManageEngine request life cycle: https://www.manageengine.com/products/service-desk/automation/onpremises-request-life-cycle.html
- State Machine Pattern: https://wendelladriel.com/blog/welcome-to-the-state-machine-pattern · https://sourcemaking.com/design_patterns/state

## §3.5 Standards / RFCs
- OWASP Top 10:2021: https://owasp.org/Top10/2021/ · A01 Broken Access Control: https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/ (lists IDOR; CWE-639, CWE-566). 2025 is RC; 2021 is the cited stable.
- OWASP ASVS 5.0.0 (May 2025): https://owasp.org/www-project-application-security-verification-standard/ · https://asvs.dev/v5.0.0/Preface/ — verification checklist for security gate.
- NIST SP 800-63B (memorized secrets, Rev.4): https://github.com/usnistgov/800-63-3/blob/nist-pages/sp800-63b/appA_memorized.md — min 8 / accept 64+ / no composition / no forced rotation / SHALL screen breached. HIBP k-anonymity: https://haveibeenpwned.com/API/v3#PwnedPasswords
- RFC 9457 Problem Details for HTTP APIs: https://www.rfc-editor.org/rfc/rfc9457.html — obsoletes RFC 7807; `application/problem+json`. **ServiceHub error contract format.**
- SOC 2 Trust Services Criteria (AICPA 2017 w/ 2022 PoF): https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022 — scope: Security + Confidentiality + Availability.
- ITIL 4 Incident & Service Request Management: https://itsm.tools/34-itil-4-management-practices/ · https://www.givainc.com/resources/itil/service-request-management/ — drives ticket "type" distinction (incident vs service request) + SLA models.

## §3.6 Competing Products
- **ServiceNow** — enterprise ITSM leader, deepest workflow/CMDB. Weakness: highest complexity + cost, months-long implementation. ServiceHub wins on time-to-value.
- **Jira Service Management** — best mid-market all-rounder, fast implementation. Weakness: Atlassian admin overhead, ecosystem lock-in. ServiceHub wins on lighter footprint.
- **Zendesk** — excellent UX, omnichannel, support-first. Weakness: shallow core ITSM (limited change mgmt, no CMDB depth). ServiceHub matches UX, ITSM-native.
- **Freshservice** — simple fast-to-deploy mid-market ITSM. Weakness: limited customization, basic reporting on low tiers. ServiceHub differentiates on multi-tenant isolation + audit rigor.
Comparison source: https://kanini.com/blog/itsm-software-comparison-2025-servicenow-vs-jira-vs-freshservice-vs-zendesk-vs-ivanti-vs-solarwinds/

## §3.7 Community Threads
- Marius Margowski — Real-World Multi-Tenancy in NestJS: https://mariusmargowski.com/article/the-real-world-guide-to-multi-tenancy-in-nestjs — request-scoped DI vs durable providers.
- PAS7 — NestJS Request Context: AsyncLocalStorage (2026): https://pas7.com.ua/blog/en/nestjs-request-context-als-2026 — recommends `nestjs-cls` (ALS) over REQUEST-scope. **Chosen tenant-context approach.**
- NestJS Injection scopes: https://docs.nestjs.com/fundamentals/injection-scopes
- TypeORM #5857 RLS pool connections leak tenant context: https://github.com/typeorm/typeorm/issues/5857 — #1 implementation pitfall (mitigate w/ `SET LOCAL` in a transaction).
- TypeORM #9464 RLS not first-class: https://github.com/typeorm/typeorm/issues/9464
- BullMQ idempotent jobs: https://docs.bullmq.io/patterns/idempotent-jobs · deduplication: https://docs.bullmq.io/guide/jobs/deduplication — at-least-once; make consumers idempotent.

## §3.8 APIs / Integrations
- **Resend send email**: `POST https://api.resend.com/emails`, `Authorization: Bearer re_…`; SDK `resend.emails.send({ from, to, subject, html })` → `{ data, error }`; batch `/emails/batch`; idempotency keys supported. Docs: https://resend.com/docs/api-reference/emails/send-email
- **Resend inbound webhook signature (Svix / Standard Webhooks)**: headers `svix-id`, `svix-timestamp`, `svix-signature`; HMAC-SHA256 over `${svix_id}.${svix_timestamp}.${rawBody}` keyed by base64-decoded secret after `whsec_`. Verify against **raw body**, constant-time compare, reject stale timestamps (replay). Docs: https://resend.com/docs/dashboard/webhooks/verify-webhooks-requests · https://docs.svix.com/receiving/verifying-payloads/how · https://www.standardwebhooks.com/

## §3.9 Patterns
- **Transactional Outbox** (guaranteed email/webhook delivery): https://microservices.io/patterns/data/transactional-outbox.html · https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html — write business row + outbox row in one tx; relay worker publishes; at-least-once. **Used for Resend emails + outbound webhooks (`notification_urgency: guaranteed delivery`).**
- **Outbox/Inbox delivery guarantees**: https://event-driven.io/en/outbox_inbox_patterns_and_delivery_guarantees_explained/ — consumers must be idempotent (inbox dedup).
- **Idempotent webhook + optimistic locking (NestJS/TypeORM)**: https://dev.to/aniefon_umanah_ac5f21311c/building-reliable-stripe-subscriptions-in-nestjs-webhook-idempotency-and-optimistic-locking-3o91 — `webhook_events` dedup table + `@VersionColumn()`.
- **Optimistic locking (version column)**: https://event-driven.io/en/idempotent_command_handling/ — compare-and-increment; reject stale writes (HTTP 409 + RFC 9457).

---

# §4 — Stack Candidates / Decisions

Stack is **fixed by customer Build Constraints** (hard constraints). Research confirms each is current, well-supported, and fit-for-purpose; no substitutions warranted.

| Concern | Chosen (constraint) | Confirmed version | Alternatives considered & why rejected |
|---------|---------------------|-------------------|----------------------------------------|
| Frontend framework | Next.js (App Router) | 16.2.x | Remix/Vite-SPA — constraint fixes Next.js; App Router gives SSR + route handlers for BFF. |
| UI components | shadcn/ui + Tailwind | CLI v4 / TW 4.3 | MUI/Chakra — constraint fixes shadcn; copy-in model lets us fully restyle to tokens (avoids stock-theme anti-slop). |
| Client state/data | TanStack Query v5 | 5.101.x | Redux/SWR — constraint fixes TanStack; ideal for server-cache + mutation invalidation. |
| Backend framework | NestJS (REST, `/v1/`) | 11.1.x | Express/Fastify-bare, Hono — constraint fixes Nest; DI + Guards give clean RBAC/tenant chokepoints. |
| ORM | TypeORM | 1.0.0 | Prisma/Drizzle — constraint fixes TypeORM; v1.0 stable; supports migrations + RLS via `SET LOCAL`. |
| Database | PostgreSQL | 18.x | MySQL/Mongo — constraint fixes Postgres; RLS is the decisive multi-tenant feature. |
| Jobs/queues | BullMQ (+ Redis) | 5.78.x | Agenda/pg-boss — constraint fixes BullMQ; mature, repeatable jobs for SLA timers. |
| Email | Resend | SDK 6.12.x | SES/Postmark — constraint fixes Resend; clean SDK + Svix webhooks. |
| Realtime | @nestjs/websockets + socket.io | 4.8.3 | SSE/raw WS — socket.io gives rooms + Redis adapter for multi-pod fan-out. |
| Tenant context | nestjs-cls (AsyncLocalStorage) | current | REQUEST-scoped DI — rejected (rebuilds DI tree per request; ALS recommended by §3.7). |
| Auth | JWT (access+refresh) via @nestjs/passport+jwt | 11.x | Session/cookie-only — JWT fits stateless autoscaling; refresh-token rotation; httpOnly cookie storage. |

# §5 — Risk Register & Threat Model (comprehensive)

## Risk register
| ID | Risk | Likelihood | Impact | Mitigation |
|----|------|-----------|--------|------------|
| R1 | TypeORM connection-pool leaks tenant context across requests | Med | Critical | `SET LOCAL app.tenant_id` inside a per-request transaction; never set on a shared connection (TypeORM #5857). App-layer tenant guard as defense-in-depth. |
| R2 | Same-tenant cross-user IDOR (W3) | High | High | `assertResourceAccess(row, principal)` chokepoint on every ticket/sub-resource/attachment route; return 404 (not 403) to non-owners. |
| R3 | Attachment upload abuse (malware, oversized, content-type spoof) | Med | High | Magic-byte sniff + extension allowlist, randomized object keys, size + per-user quota, store outside webroot, AV-scan-before-serve hook. |
| R4 | Auth account enumeration / brute force | Med | Med | Uniform login/reset/signup responses + timing; per-(ip,email) rate-limit before hash compare; HIBP breach screening (NIST 800-63B). |
| R5 | Guaranteed-delivery email/webhook lost on crash (dual-write) | Med | High | Transactional outbox (`pending_emails`/`pending_webhooks`); drain worker w/ capped backoff → DLQ; UNIQUE dedup keys. |
| R6 | Inbound webhook spoofing/replay | Med | High | HMAC-SHA256 verify over raw body BEFORE any DB read; reject stale timestamp; idempotent inbox dedup by event id. |
| R7 | Concurrent ticket edit clobbering | Med | Med | `@VersionColumn` optimistic lock; 409 + RFC 9457 on mismatch. |
| R8 | SLA/audit-log tampering | Low | High | Append-only audit log w/ server timestamps; SLA computed only by server-side evaluator; no client-settable due_at/paused. |
| R9 | WebSocket cross-tenant/channel leak | Med | High | Authenticate handshake (JWT); authorize room join per tenant+ticket; re-check authz on join. |
| R10 | Secrets exposure | Low | Critical | Env vars w/ encryption at rest (constraint); `.env.example` placeholders only; no secrets in repo/logs; Gitleaks in audit. |

## Threat model (STRIDE-flavored, top boundaries)
1. **Tenant isolation breach** → RLS `SET LOCAL` per-tx + mandatory `tenant_id` + app guard. (R1)
2. **Same-tenant cross-user IDOR (tickets/attachments/comments/timeline)** → object-level authz chokepoint; 404 to non-owners. (R2)
3. **Attachment upload abuse / malware** → magic-byte validation, allowlist, randomized keys, quota, scan-before-serve. (R3)
4. **SSRF via upload/URL fields & outbound webhooks** → block internal/link-local ranges, allowlist webhook destinations, no user-controlled fetch URLs. (R3/R6)
5. **Auth / account enumeration** → uniform responses+timing, rate-limit before hash, HIBP screening. (R4)
6. **SLA / audit tampering** → append-only audit, server-authoritative SLA evaluator. (R8)
7. **WebSocket channel authz** → authenticate handshake, authorize per-room, re-check on emit/join. (R9)
8. **Webhook spoofing / replay** → HMAC over raw body before DB read, timestamp tolerance, idempotent dedup. (R6)
9. **At-least-once duplicate side effects** → idempotency keys + dedup table; BullMQ custom job IDs; outbox. (R5)
10. **Concurrent edit clobbering** → optimistic locking, 409 + problem+json. (R7)

# §6 — Research Gaps
- **G1 (resolved-by-design):** AV scanning for attachments — no managed AV in constraints; design provides a `scan-before-serve` hook + quarantine state; pluggable ClamAV/cloud AV documented as a deploy-time integration (DECISIONS ADR). Local build uses magic-byte validation + quarantine flag; AV worker is a stub interface.
- **G2:** Object storage target unspecified (S3/GCS) — abstract behind a `StorageService` interface; local build uses local-disk driver outside webroot; cloud uses S3-compatible driver via env config.
- **G3:** Business-hours SLA calendar — comprehensive SLA pausing uses 24/7 clock by default; business-hours calendar is a documented extension point (single evaluator already isolates the calculation).
- **G4:** Multi-tenant onboarding/billing flows are out of scope (Non-Goal-adjacent); tenants are provisioned by seed/admin for this build.

# §7 — Summary & Go/No-Go
**Summary:** ServiceHub is a multi-tenant ITSM/ticketing platform on a fully-specified, current, well-supported stack (NestJS 11 / Next.js 16 / PostgreSQL 18 / TypeORM 1.0 / BullMQ 5 / Resend 6). The decisive architecture forks — tenant isolation (RLS + `SET LOCAL` per-tx), object-level authz (chokepoint, 404-to-non-owner), guaranteed delivery (transactional outbox), and SLA computation (single server-side evaluator) — are all backed by canonical, verified sources. Threat model identifies 10 boundaries, each with a concrete mitigation that maps to an architectural invariant. `scale: medium (1k–50k)` mandates connection pooling + Redis cache + real BullMQ queue (all present). No blocking gaps; G1–G4 are bounded extension points with safe local defaults.

**Go/No-Go: GO**
