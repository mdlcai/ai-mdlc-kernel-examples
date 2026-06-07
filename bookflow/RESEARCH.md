# RESEARCH.md — BookFlow

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Administration"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_dual_write"]
```

## Product Vision
**Problem:** Organizations often manage reservations through spreadsheets, email, and disconnected calendar systems, leading to double bookings, approval bottlenecks, poor visibility, and administrative overhead.

**Who it affects:** Employees requesting bookings across departments—need frictionless reservation submission without navigating multiple systems or approval chains; pain points include unclear approval status, rejected requests with no guidance, and inability to find available resources quickly.

Managers and Approvers reviewing requests within their domain—need clear visibility into pending approvals, context about requesters and resource impact, and ability to enforce org-specific rules (budget limits, advance notice, team quotas); bottlenecked by manual email chains and lack of audit trails.

Operations Teams managing resource allocation and schedules—need real-time visibility into utilization rates, conflict detection, and capacity planning; pain points include manual conflict resolution, inability to enforce resource dependencies (e.g., room + AV setup), and reactive rather than predictive scheduling.

Facilities Teams overseeing physical spaces and shared assets—need to prevent double bookings, enforce maintenance windows, manage room-specific rules (capacity, equipment included), and track usage patterns; currently rely on spreadsheets or fragmented calendar systems.

IT Teams managing equipment and asset reservations—need to track availability across distributed inventory, enforce checkout/return workflows, flag high-demand items, and integrate with asset management systems; pain points include lost equipment, unclear ownership during loans, and no automated reminders.

Service Providers delivering scheduled services or appointments—need flexible scheduling that accommodates client availability, staff capacity, and service-specific requirements; need to communicate confirmations and changes without manual outreach.

Leadership requiring operational intelligence—need dashboards showing resource utilization trends, approval cycle times, demand forecasting, and cost allocation; currently lack data to optimize resource investment or identify bottlenecks.

**Why existing solutions fall short:** Many booking platforms are designed for a single use case and lack workflow flexibility. Organizations frequently require approvals, custom business rules, resource dependencies, and reporting that traditional scheduling tools cannot easily provide without costly customization.

**Solution:** BookFlow is a configurable booking platform that streamlines appointment and resource management through customizable workflows. It enables organizations to automate booking requests, approvals, and scheduling across appointments, facilities, equipment, and services from a single portal. The key differentiator is workflow customization—allowing each organization to enforce their specific approval processes and business rules without custom development.


## Users & Outcomes
**Key Workflows:**
Employee submits booking request for a resource (room, equipment, service) through a single portal, specifying dates, times, and requirements. System validates real-time availability, checks business rules (budget limits, advance notice, team quotas, resource dependencies), and routes to appropriate approvers based on org-defined rules. Approver reviews request with full context (requester history, resource impact, conflicting bookings) and approves, rejects with guidance, or requests changes within SLA. Employee receives status updates and guidance on rejections. Outcome: 80% of requests approved without rejection cycles; approval SLA met 95% of time; employees find available resources in <2 minutes.

Manager enforces org-specific approval policies (budget thresholds, advance booking windows, team allocation caps) without code changes. System applies rules consistently across all requests, flags policy violations, and provides audit trail of decisions. Manager reviews dashboards showing pending approvals, approval cycle times, and policy exceptions. Outcome: approval bottlenecks reduced by 60%; policy compliance auditable; manual email chains eliminated.

Operations team monitors real-time resource utilization, capacity constraints, and booking conflicts across all resources. System detects double bookings, enforces resource dependencies (room + AV setup), and surfaces high-demand items. Team resolves conflicts through system-guided reassignment or capacity alerts. Outcome: zero double bookings; resource utilization visibility in real-time; conflict resolution time reduced by 75%.

Facilities team prevents double bookings on physical spaces, enforces maintenance windows and room-specific rules (capacity, included equipment), and tracks usage patterns for planning. System blocks conflicting bookings and generates utilization reports. Outcome: 100% booking accuracy; maintenance windows protected; usage data informs space optimization.

IT team tracks equipment availability across distributed inventory, enforces checkout/return workflows, and flags high-demand items. System sends automated reminders for returns, integrates with asset management, and surfaces equipment ownership during loans. Outcome: equipment loss reduced by 90%; checkout/return compliance >95%; high-demand items identified for procurement.

Service provider schedules appointments accommodating client availability, staff capacity, and service requirements. System auto-confirms bookings and sends notifications to clients and staff. Outcome: confirmation turnaround <1 hour; no-show rate reduced by 40%; manual outreach eliminated.

**Success Metrics:**
Booking Completion Rate – 92% of submitted requests result in confirmed bookings within 30 days; <8% rejected or abandoned.

Time to Confirm Booking – Median 4 hours from submission to confirmation; 95th percentile ≤24 hours; service appointments ≤1 hour.

Resource Utilization – 65-75% of available resource slots booked monthly; peak-hour utilization ≥80%; identify items <40% utilization for reallocation.

Scheduling Conflict Reduction – Zero double bookings post-launch; <0.5% of bookings flagged for dependency violations (e.g., room without required AV); 100% conflict detection accuracy.

Approval Turnaround Time – Median 2 hours for standard requests; 95% completed within 8 hours; SLA breach <5% monthly; budget-threshold requests ≤4 hours.

User Adoption Rate – 85% of eligible employees submit ≥1 booking monthly via platform; 70% transition from email/spreadsheet within 90 days; <15% concurrent manual process usage by month 6.

Self-Service Usage – 88% of bookings completed without admin intervention; <12% requiring manual reassignment or conflict resolution.

User Satisfaction – Net Promoter Score ≥50; ease-of-use rating ≥4.2/5; approval clarity rating ≥4.0/5; rejection guidance helpfulness ≥4.1/5.

Operational Efficiency – 60% reduction in admin hours spent on booking management; email approval chains eliminated for 90% of requests; manual conflict resolution time ≤15 minutes per incident.

System Availability – 99.5% uptime monthly; page load <2 seconds; search results <1 second; zero data loss incidents; recovery time objective ≤1 hour.

**Non-Goals:**
Financial management, billing, or payment processing.
Human Resources management or employee record keeping.
Customer Relationship Management (CRM) functionality.
Project or task management beyond reservation workflows.
Inventory management beyond tracking reservable resources.
Infrastructure monitoring or IT operations management.
Asset lifecycle management, procurement, or maintenance tracking.
Video conferencing, messaging, or collaboration platform replacement.
Advanced workforce scheduling or timekeeping.
Enterprise Resource Planning (ERP) functionality.


## Build Constraints

```yaml
# Infrastructure & Ops
hosting_environment: "cloud platform with autoscaling"
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "metrics + alerting"
backup_strategy: "automated daily with point-in-time recovery"
container_strategy: "orchestrated / multi-instance"
error_reporting: "centralized error tracking"

# Data & Storage
database_preference: "PostgreSQL"
data_retention_policy: "audit logs retained for 7 years; booking records retained indefinitely"
pii_handling: ["employee names and contact info", "approval decision audit trail"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"

# Security & Compliance
security_baseline: ["SOC2"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables + vault"

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

# Performance & Quality
performance_requirements: ["page load < 2 seconds", "search results < 1 second", "approval SLA ≤ 8 hours", "99.5% uptime"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "small — under 1k concurrent"
multi_tenant: true
target_platforms: ["web"]
report_formats: ["PDF", "CSV"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: admin

This product's design archetype is **admin**. Its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components from `DESIGN.md` Part II §`admin` are binding. The Universal Excellence floor (`DESIGN.md` Part I) applies on top. **A Design Template was uploaded** (`DESIGN-TEMPLATE.html`); its `:root` tokens are copied verbatim and are canonical (see §Design Template + DECISIONS.md).

### Brand Voice
Utilitarian, dense, fast. Maximum useful information per screen; zero ornament; guarded destructive actions.

### Art Direction
Maximum useful information per screen. Dense, sharp, neutral, zero ornament; clear data tables and guarded destructive actions. Fast and flat, no decorative motion.

### Tokens (from uploaded Design Template — canonical)
- Palette: bg `#f4f4f3`, surface `#ffffff`, surface-2 `#ebebe9`, fg `#111111`, muted `#5c5c5c`, border `#dcdcd9`, accent `#000000`, accent-soft `#e6e6e3`, success `#1c7c4a`, warning `#9a6400`, error `#b3261e`.
- Type: display `IBM Plex Sans`, body `Geist`, mono `Geist Mono`.
- Spacing: 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64 px. Radius: sm 4 / md 6 / lg 10. Shadows: sm/md/lg minimal.
- Template is light-only; a dark palette is **derived alongside** (template tokens never replaced) per DESIGN.md "derive missing tokens" rule — logged in DECISIONS.md.

### Accessibility
- WCAG 2.2 AA minimum (config requested AAA; AA is the enforced floor, AAA targeted where contrast allows). Lighthouse target 98+. Breakpoints 640 / 768 / 1024 / 1280px.

## Design Template

An HTML design template was uploaded with this project and fetched at build time via `get_design_template`. Saved verbatim to project root as `DESIGN-TEMPLATE.html`. Its `:root` custom properties are copied verbatim as the canonical token set (see DECISIONS.md ADR-0002). The landing/home marketing surface scaffold (sticky nav, two-column hero with simulated app preview, bento feature grid, dark CTA band, footer) is ported into the Next.js app.

---

## 3. Source Categories

### 3.1 Official / Vendor Documentation

| Source | URL | Covers | Current stable (mid-2026) |
|---|---|---|---|
| Next.js | https://nextjs.org/docs | App Router, caching, Server Components, routing | **16.2.x** (Turbopack default; Cache Components stable) |
| NestJS | https://docs.nestjs.com | Modules, DI, guards, interceptors, SSE, WebSockets | **11.1.x** (v12 ESM in Q3-2026 — stay on 11 for stability) |
| TypeORM | https://typeorm.io | Entities, migrations, transactions, locking | **0.3.x** (latest 0.3 line; pin exact for migration stability) |
| PostgreSQL | https://www.postgresql.org/docs/current/ | SQL, range types, exclusion constraints, PITR | **16/17** (16 used in container; range + btree_gist) |
| Tailwind CSS | https://tailwindcss.com/docs | Utility CSS | **3.4.x** (stable JS-config; shadcn-compatible) |
| shadcn/ui | https://ui.shadcn.com/docs | Component registry, CLI | current CLI |
| TanStack Query | https://tanstack.com/query/latest | Server-state cache, mutations, invalidation | **5.x** |
| Resend | https://resend.com/docs | Email send API, webhooks (Svix-signed) | API v1 / Node SDK current (`resend` npm) |
| @nestjs/throttler | https://docs.nestjs.com/security/rate-limiting | Per-route/global rate limiting | **6.x** (Nest 11 compatible) |
| Docker Compose | https://docs.docker.com/reference/compose-file/ | Multi-container orchestration, healthchecks, secrets | **Compose Spec v2 plugin** — no top-level `version:` key |

**Version-specific gotchas**
- **Next.js (15/16)**: App Router `fetch` is uncached by default — explicitly opt into caching; availability endpoints use `no-store`/short `revalidate` and rely on SSE for live state.
- **NestJS 12 (upcoming)**: CJS→ESM migration. For a SOC2 build targeting stability, **pin v11** and defer v12.
- **TypeORM + ESM**: keep TypeORM and NestJS in the same module system (CJS) to avoid dual-package hazards; `reflect-metadata` import order matters.
- **btree_gist** must be `CREATE EXTENSION` before an exclusion constraint can mix `=` (scalar) with `&&` (range) operators.
- **Tailwind**: pin to a stable line that shadcn/ui CLI targets; don't mix v3 JS-config with v4 CSS-config patterns.

### 3.2 GitHub Repositories

| Repo | URL | ~Stars | Recency | License | Reusable / learnable |
|---|---|---|---|---|---|
| henriqueweiand/nestjs-typeorm-multi-tenancy | https://github.com/henriqueweiand/nestjs-typeorm-multi-tenancy | hundreds | active 2024-25 | MIT | **Row/CLS-based tenancy** using `nestjs-cls` — closest to our `org_id` row-level scoping. |
| fullstackhouse/nestjs-outbox | https://github.com/fullstackhouse/nestjs-outbox | low-mid | active | MIT | **Transactional outbox** — atomic business row + outbox row, separate publisher. Template for email/webhook dual-write. |
| Nestixis/nestjs-inbox-outbox | https://github.com/Nestixis/nestjs-inbox-outbox | low-mid | active | MIT/Apache | Inbox **and** outbox; inbox covers `verify-then-read + idempotency` webhook ingestion. |
| calcom/cal.com | https://github.com/calcom/cal.com | ~41k | very active | AGPLv3 (core) | Production scheduling: availability math, conflicts, multi-tenant teams. Reference architecture (AGPL — learn, don't copy). |

### 3.3 Video / Tutorials

| Title / Channel | URL | Teaches |
|---|---|---|
| "NestJS Tutorial: Build a REST API in 13 Steps [2026]" | https://tech-insider.org/nestjs-tutorial-rest-api-13-steps-2026/ | Nest 11 REST + TypeORM wiring, DTO validation, versioned controllers. |
| Tailwind CSS Full Course 2026 (YouTube) | https://www.youtube.com/watch?v=6biMWgD6_JY | Tailwind config + shadcn/ui integration. |
| "A Guide to using Server-Sent Events (SSE) with NestJS" (Medium) | https://medium.com/@odenigbo67/a-guide-to-using-server-sent-events-sse-with-nestjs-5e28de80617a | Nest `@Sse()` pattern for live availability/approval push. |

### 3.4 Articles / Blog Posts

| Title | URL | Relevance |
|---|---|---|
| Schema-based multitenancy in NestJS with TypeORM | https://www.scalzotto.nl/posts/nestjs-typeorm-schema-multitenancy/ | Postgres isolation strategies (schema vs row) + NestJS wiring trade-offs. |
| "PostgreSQL's GiST Exclusion Constraint: The DB-Level Answer to Double Bookings" | https://amitavroy.com/articles/postgresql-gist-exclusion-constraintthe-database-evel-answer-to-double-bookings | DB-level no-double-booking via `EXCLUDE USING GIST`; why app checks are insufficient. |
| "Solving the Dual Write Problem with NestJS: Inbox and Outbox Patterns" | https://axotion.medium.com/solving-the-dual-write-problem-with-nestjs-implementing-inbox-and-outbox-patterns-3b20a8bd49a1 | Concrete NestJS outbox/inbox for email + webhook dual-write. |

### 3.5 Standards / RFCs

| Standard | URL | Relevance |
|---|---|---|
| RFC 7807/9457 (Problem Details for HTTP APIs) | https://www.rfc-editor.org/rfc/rfc7807 | `application/problem+json` error envelope for the `/v1/` API. |
| RFC 5545 (iCalendar) | https://www.rfc-editor.org/rfc/rfc5545 | Time/recurrence semantics; `.ics` export of bookings. |
| NIST SP 800-63B | https://pages.nist.gov/800-63-3/sp800-63b.html | Password/auth (length over complexity, no forced rotation, rate-limit before compare). |
| WCAG 2.2 | https://www.w3.org/TR/WCAG22/ | Accessibility target for the shadcn/ui frontend. |
| OWASP ASVS / Top 10 | https://owasp.org/www-project-top-ten/ | Security baseline; maps to the threat model in §5. |
| SOC 2 Trust Services Criteria | https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services | Security/Availability/Confidentiality criteria driving audit logging, backups, access control. |
| RFC 6750 (Bearer Token Usage) | https://www.rfc-editor.org/rfc/rfc6750 | `Authorization: Bearer` token handling. |
| RFC 5789 (PATCH) | https://www.rfc-editor.org/rfc/rfc5789 | Partial-update semantics for resource edits. |

### 3.6 Competing Products

| Product | URL | Positioning | Gap BookFlow fills |
|---|---|---|---|
| Skedda | https://www.skedda.com | Workplace/space booking; has request-first approvals | Approvals not deeply **configurable per-tenant** or multi-step/multi-approver. |
| Robin | https://robinpowered.com | Enterprise room/desk booking at scale | Heavy; limited tenant-configurable approval logic; not API-first. |
| OfficeRnD | https://www.officernd.com | Coworking + hybrid workspace mgmt | Coworking-centric; less general for equipment/services. |
| Cal.com | https://cal.com | Open-source scheduling infra, API-first | Person-availability scheduling, not resource + configurable multi-step approvals. |
| Calendly | https://calendly.com | SaaS meeting scheduling | No multi-tenant resource inventory, no approval chains. |
| Envoy | https://envoy.com | Workplace/visitor + desk/room | Visitor-first; approvals not the core configurable primitive. |

**Differentiator**: tenant-configurable, multi-step approval workflows over heterogeneous resources (rooms/equipment/services) with DB-guaranteed conflict prevention, audit trails, and SSO-ready multi-tenancy.

### 3.7 Community Threads

| Thread | URL | Topic |
|---|---|---|
| "Exclusion constraints in PostgreSQL and a tricky problem" (Cybertec) | https://www.cybertec-postgresql.com/en/exclusion-constraints-in-postgresql-and-a-tricky-problem/ | Gotchas with `EXCLUDE USING GIST` on `tstzrange` (NULLs, partial `WHERE`, deferrable). |
| nestjs/typeorm Issue #58 — "Multi tenancy with Nest and TypeORM and Postgres?" | https://github.com/nestjs/typeorm/issues/58 | Request-scoped connections vs row-level `org_id` scoping. |
| "Exploring PostgreSQL's EXCLUDE Operator" (dev.to/rozhnev) | https://dev.to/rozhnev/exploring-postgresqls-exclude-operator-advanced-data-constraints-2k60 | Overlap-prevention with `WHERE status != 'cancelled'` partial constraints. |

### 3.8 APIs / Integrations

- **Resend** — Send: `https://resend.com/docs/api-reference/emails/send-email`. Webhooks: `https://resend.com/docs/dashboard/webhooks/verify-webhooks-requests`. Signed via **Svix** — headers `svix-id`, `svix-timestamp`, `svix-signature`; verify HMAC-SHA256 over `${svix_id}.${svix_timestamp}.${body}` with the base64 secret (after `whsec_`). **Constant-time compare** + timestamp tolerance for replay protection. Read the **raw** body.
- **Postgres btree_gist / range types** — `https://www.postgresql.org/docs/current/btree-gist.html`, `https://www.postgresql.org/docs/current/rangetypes.html`. `CREATE EXTENSION btree_gist;` then exclusion constraint mixing `=` and `&&`.
- **Calendar (iCal / Google Calendar)** — RFC 5545 for `.ics` export; Google Calendar API later. iCal export is low-effort interop.
- **SSO / SAML** ("SSO ready") — SAML 2.0 / OIDC. Node options: `@node-saml/passport-saml` or hosted broker (WorkOS / BoxyHQ Jackson). Design auth to accept external IdP assertions per-tenant even if deferred.

### 3.9 Patterns

- **(a) Postgres EXCLUSION constraint (tstzrange + GIST)** — `CREATE EXTENSION btree_gist; ... EXCLUDE USING GIST (resource_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE (status <> 'cancelled' AND status <> 'rejected')`. *Use for*: structurally guaranteed no-double-booking across all code paths. App-level checks are UX-only.
- **(b) Transactional outbox** — In one DB transaction, write the business row + an `outbox` row; a separate worker polls (`FOR UPDATE SKIP LOCKED`) and delivers to Resend / webhook consumers. *Use for*: email (guaranteed delivery) and outbound webhooks (`has_dual_write`). Pair with retry/backoff → DLQ.
- **(c) Row-level multi-tenant scoping** — Every tenant-owned table carries `org_id`; a scoped repository wrapper injects `org_id` into every query. *Use for*: tenant isolation without per-tenant schemas (right fit at <1k concurrent).
- **(d) Optimistic locking / `SELECT ... FOR UPDATE`** — TypeORM `@VersionColumn` on edits; pessimistic `FOR UPDATE` ordered by stable key (id). *Use for*: approval state transitions and booking mutations.
- **(e) Webhook verify → idempotency → process** — Verify Svix signature on raw body → check `svix-id` against a processed-events table (insert-or-skip) → process in a transaction. *Use for*: Resend inbound webhooks (`has_webhooks`).
- **(f) RBAC** — Roles (admin / manager / approver / ops / member) + permission checks via NestJS guards, scoped within tenant. *Use for*: user→approver privilege boundary and admin-only workflow configuration.

---

## 4. Stack Candidates

Stack is fixed by build constraints; this table records resolved versions and rationale.

| Layer | Chosen | One-line rationale |
|---|---|---|
| Frontend framework | Next.js (App Router) | SSR/RSC + mature routing; pair with TanStack Query for server-state. |
| UI components | shadcn/ui | Owned, accessible components; design-token theming. |
| Styling | Tailwind CSS | Token-driven utility CSS; integrates with shadcn. |
| Client data | TanStack Query | Cache/invalidate/mutations; complements SSE live updates. |
| Backend | NestJS 11 | DI, guards, interceptors, native SSE/WS; pin v11, defer v12 ESM. |
| API style | REST, URL-path `/v1/` | Explicit versioning; RFC 7807 errors, RFC 5789 PATCH, RFC 6750 bearer. |
| ORM | TypeORM 0.3.x | Migrations, version/pessimistic locking. |
| Database | PostgreSQL 16 | Range types + `btree_gist` exclusion constraints = core booking guarantee; PITR. |
| Realtime | NestJS SSE | One-way live availability/approval push; WS only if bidirectional needed. |
| Email | Resend + outbox | Guaranteed delivery via transactional outbox worker; Svix-signed webhooks. |
| Rate limiting | @nestjs/throttler | Per-route/global limits; applied before auth hash compare. |
| Container | Docker Compose (local) + cloud autoscaler | Multi-instance, healthchecks; prod on autoscaling container platform. |

**Architecture tier note**: scale is small (**<1k concurrent**) → **no Redis cluster or read replicas required** for v1; single primary Postgres + daily backups + PITR. The **SSE realtime channel** and a **standalone outbox worker** are still mandatory (correctness/availability, not scaling). If horizontal API scaling is enabled later, the outbox poller uses `FOR UPDATE SKIP LOCKED` (already designed in) and SSE fan-out needs a lightweight pub/sub — first thing to add if concurrency grows.

---

## 5. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Double-booking via race condition | Med | High | Postgres `EXCLUDE USING GIST` exclusion constraint — DB rejects overlaps atomically; app handles violation as 409. |
| Cross-tenant data leak (missing `org_id` filter) | Med | Critical | Mandatory scoped-query wrapper; default-deny; tenant-isolation tests in CI; not-found (not forbidden) for cross-tenant reads. |
| Dual-write inconsistency (email/webhook vs tx) | Med | High | Transactional outbox — business row + outbox row in one tx; idempotent worker; retry/DLQ. |
| Duplicate/forged inbound webhooks | Med | Med | Verify Svix signature on raw body + timestamp tolerance + idempotency table keyed on `svix-id`; rate-limit ingress. |
| NestJS v12 ESM migration disruption | Low-Med | Med | Pin v11.x; v12 is post-launch migration, not a dependency. |
| Auth account enumeration / brute force | Med | High | Uniform responses; throttler **before** hash compare; NIST 800-63B; audit logging. |
| Backup/recovery untested (SOC2) | Low | Critical | Daily backups + PITR; restore drills documented. |
| Stale availability via Next caching | Med | Med | Explicit cache opt-out on availability; SSE for live state. |

### Threat-Model Assessment (STRIDE by trust boundary)

| Trust boundary | Top threat(s) | Control |
|---|---|---|
| **Anonymous → Authenticated** | Spoofing (credential stuffing), Info-disclosure (enumeration), DoS (brute force) | Anti-enumeration uniform responses; rate limiting **before** hash compare; bearer tokens; NIST 800-63B; audit logging. |
| **Tenant → Tenant** | Info-disclosure / Tampering (IDOR, cross-org) | Row-level `org_id` scoping wrapper on every query; default-deny; isolation tests; `E.NOT_FOUND` for cross-tenant. |
| **User → Approver (privilege)** | Elevation of privilege (self-approving, editing others') | RBAC guards; approval transitions server-enforced + version-locked; full audit trail. |
| **Webhook ingress (Resend → us)** | Spoofing (forged events), Replay, Tampering | Svix HMAC-SHA256 verify on raw body; timestamp tolerance; idempotency table on `svix-id`; constant-time compare; ingress rate limit. |
| **Email egress (us → Resend)** | Repudiation / data loss; Info-disclosure | Transactional outbox (atomic), idempotent worker, retry/DLQ; recipient bound to tenant-scoped data; audit log of sends. |
| **Booking write path** | Tampering (double-booking, overlap) | DB `EXCLUDE USING GIST` constraint; `FOR UPDATE` ordered by id for multi-row locks. |
| **Cross-cutting** | Repudiation | Structured JSON audit logging (who/what/when/tenant) on all state changes; append-only audit table. |

---

## 6. Research Gaps

1. **Resend webhook signature exactness** — Svix HMAC-SHA256 over `id.timestamp.body` confirmed; re-confirm header set / `resend.webhooks.verify()` signature against the live SDK at implementation time.
2. **SSE vs WebSocket** — SSE sufficient for ≤1k concurrent (one-way push, auto-reconnect). Commit to SSE; WS upgrade path documented if presence/collab added.
3. **SAML/SSO provider** — "SSO ready" is a v1 design constraint; provider unresolved (passport-saml vs BoxyHQ vs WorkOS). Per-tenant IdP config model deferred; auth designed to accommodate.
4. **Multi-tenant isolation depth** — Row-level `org_id` chosen for tier; RLS or schema-per-tenant may be required later for high-sensitivity tenants.

---

## 7. Summary & GO/NO-GO

The fixed stack is mature, current, and well-matched to a configurable multi-tenant booking platform: Next.js + shadcn/ui + Tailwind + TanStack Query on the frontend; NestJS 11 + TypeORM + PostgreSQL on the backend. The two hardest domain problems have first-class, proven solutions — guaranteed no-double-booking via Postgres `EXCLUDE USING GIST`, and reliable email/webhook dual-write via the transactional outbox — both with reference implementations on GitHub. Multi-tenancy (row-level `org_id`), SOC2 controls (audit logging, rate limiting, backups/PITR), and SSO-readiness are standard documented patterns at the target scale (<1k concurrent; SSE + outbox worker still required). Open items are configuration choices (SSE-vs-WS, SAML provider, RLS depth), none blocking.

**Go/No-Go: GO**
