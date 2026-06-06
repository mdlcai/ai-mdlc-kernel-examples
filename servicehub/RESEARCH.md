# RESEARCH.md — ServiceHub

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Information technology ticketing"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_image_uploads"]
```

## Product Vision
**Problem:** IT requests are often submitted through email, chat, phone calls, or spreadsheets, resulting in lost requests, inconsistent prioritization, limited visibility, and inefficient support processes. IT teams need a structured system to manage incidents, service requests, and escalations at scale.

**Who it affects:** Primary Users:

**End-User Employees** — Staff across departments submitting IT requests (hardware issues, software access, account problems, connectivity). Pain points: multiple submission channels causing lost requests, unclear status visibility, slow resolution times, no self-service options for common issues.

**Help Desk Technicians** — First-line support handling incoming tickets, triaging severity, and routing to specialists. Pain points: manual ticket creation from emails/calls, context switching between systems, unclear escalation paths, inability to track SLA compliance, repetitive work on similar issues.

**Service Desk Managers** — Overseeing help desk operations, staffing, and SLA adherence. Pain points: limited visibility into team workload distribution, difficulty identifying bottlenecks, manual reporting on resolution times and ticket trends, no data-driven insights for resource planning.

**Infrastructure & Operations Teams** — Managing servers, networks, storage, and system deployments. Pain points: tickets arriving through ad-hoc channels, unclear dependencies between requests, difficulty prioritizing maintenance vs. incident response, lack of integration with monitoring tools.

**Security Teams** — Handling access requests, compliance incidents, and security-related tickets. Pain points: audit trail gaps, slow approval workflows, difficulty tracking policy violations, manual coordination with other teams on sensitive issues.

**IT Leadership & Directors** — Responsible for service delivery metrics, budget allocation, and vendor management. Pain points: opaque operational metrics, inability to demonstrate ROI, slow decision-making due to fragmented data, difficulty forecasting staffing needs based on demand patterns.

**Why existing solutions fall short:** Many IT ticketing platforms are overly complex, expensive, and require significant customization before delivering value. Users often struggle with cluttered interfaces, confusing workflows, and slow ticket submission processes, leading to poor adoption and increased support overhead.

For IT teams, existing solutions frequently include unnecessary features, fragmented workflows, and cumbersome administration. Reporting, automation, and SLA management can be difficult to configure, while licensing costs often scale aggressively as organizations grow.

This platform focuses on simplicity, speed, and operational efficiency by providing streamlined ticket management, intuitive workflows, automated routing, and actionable reporting without the complexity of traditional enterprise ITSM solutions.

**Solution:** ServiceHub is a centralized IT service management platform that unifies employee support requests, incident reporting, and ticket tracking in one interface. It accelerates resolution through streamlined workflows and gives IT teams actionable visibility into workload patterns and recurring issues—reducing response times while improving service delivery efficiency.


## Users & Outcomes
**Key Workflows:**
**End-User Employee** submits an IT request through a self-service portal with pre-populated context (department, device, issue category), receives immediate confirmation with ticket number and expected resolution timeframe, reducing submission time from 10+ minutes across multiple channels to under 2 minutes and eliminating lost requests.

**Help Desk Technician** receives automatically routed tickets matched to skill level and current workload, views complete request history and related incidents in a single interface, reducing context-switching time and enabling first-contact resolution for 40%+ of routine issues.

**Help Desk Technician** escalates unresolved tickets by dragging to a priority queue with automated notification to specialists and manager, with SLA countdown visible, ensuring critical issues reach appropriate teams within defined timeframes instead of languishing in email threads.

**Service Desk Manager** accesses a real-time dashboard showing ticket volume by category, resolution times by technician, SLA breach rate, and queue depth, enabling data-driven staffing decisions and identification of bottleneck issue types within minutes rather than manual weekly reporting.

**Infrastructure & Operations Team** receives tickets tagged with dependencies and system impact, views related incidents and maintenance windows in context, and prioritizes requests against monitoring alerts, reducing coordination overhead and preventing conflicting changes.

**Security Team** processes access requests through a structured workflow with built-in approval chains, maintains complete audit trail of who approved what and when, and flags policy-violation patterns automatically, enabling compliance reporting and faster incident response.

**IT Leadership** views executive dashboard with ticket volume trends, cost-per-resolution by category, team utilization rates, and SLA performance metrics, translating operational data into business impact and enabling forecasting for budget and headcount planning.

**Success Metrics:**
Reduce mean time to resolution (MTTR) from current 18-24 hours to <8 hours for routine requests and <4 hours for critical incidents within 6 months of deployment. Track by ticket category and technician to identify optimization opportunities.

Achieve 95%+ SLA compliance rate (tickets resolved within defined timeframes by severity: critical <2h, high <8h, medium <24h, low <72h) measured weekly, with automated alerts at 80% threshold to prevent breaches.

Increase first-contact resolution rate from current 25-30% to 45%+ within 3 months by enabling technicians to resolve common issues without escalation, measured as percentage of tickets closed by initial assignee without reassignment.

Reduce ticket backlog from current 200-300 open tickets to <100 within 90 days, with daily backlog tracking by priority level and aging ticket alerts for items exceeding 48 hours in queue.

Improve end-user satisfaction score from current 3.2/5.0 to 4.3/5.0 or higher within 6 months, measured through post-resolution surveys with minimum 40% response rate, tracking satisfaction by issue category and technician to identify training needs.

**Non-Goals:**
**Non-Goals:**

Remote desktop management — ServiceHub routes tickets to technicians; remote access tools integrate via API but aren't built-in, keeping the platform lightweight and avoiding licensing conflicts with existing RMM solutions.

Infrastructure monitoring — Real-time system health dashboards and alerting belong in dedicated monitoring platforms; ServiceHub consumes monitoring data to contextualize tickets, not generate it.

Endpoint management and patching — Device inventory, software deployment, and patch orchestration require specialized tools; ServiceHub tracks endpoint-related tickets and links to external MDM/patch systems for visibility without managing endpoints directly.

Network performance monitoring — Packet analysis, bandwidth tracking, and QoS management stay in network monitoring tools; ServiceHub receives network alerts and routes them as tickets without duplicating network-specific instrumentation.

Security event monitoring (SIEM) — Log aggregation, threat detection, and forensic analysis require dedicated security platforms; ServiceHub processes security-related tickets and maintains audit trails for compliance, not raw security event ingestion or analysis.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "application performance monitoring"
backup_strategy: "daily automated backups with point-in-time recovery"
container_strategy: "Docker with container orchestration"
error_reporting: "centralized error tracking"

# Data & Storage
database_preference: "PostgreSQL"
pii_handling: ["employee names", "email addresses", "department information", "device identifiers"]

# Notifications & Messaging
notification_urgency: "guaranteed delivery"
webhook_targets: ["monitoring systems", "external ticketing integrations"]

# Security & Compliance
security_baseline: ["SOC2", "OWASP Top 10"]
rate_limiting: true
audit_logging: true
secrets_management: "environment-based secrets"

# Frontend
frontend_framework: "React"

# Backend
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "TypeORM"
realtime_needed: true
background_jobs: "job queue for ticket routing and notifications"

# Performance & Quality
performance_requirements: ["API response time < 200ms", "dashboard load time < 2s", "ticket submission < 1s"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "medium — 1k-50k"
target_platforms: ["web"]
alert_channels: ["email", "in-app notifications"]
report_formats: ["PDF", "CSV"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, efficient, and user-focused. Emphasize simplicity, accountability, transparency, and rapid resolution of IT issues.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #5e6ed2 | Buttons, links, active states |
| Secondary | #8a90a8 | Accents, badges, highlights |
| Accent | #636bf1 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #ffffff | Cards, elevated containers |
| Text | #08090a | Headings, body text |
| Text Secondary | #6c6f7a | Captions, muted text |
| Success | #10b981 | Success states, confirmations |
| Warning | #f59e0b | Warnings, pending states |
| Error | #ef4444 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #7074ff | Buttons, links, active states |
| Secondary | #b4b6c1 | Accents, badges, highlights |
| Accent | #8190f8 | Callouts, hover states |
| Background | #08090a | Page background |
| Surface | #131318 | Cards, elevated containers |
| Text | #f7f8f8 | Headings, body text |
| Text Secondary | #8a8f98 | Captions, muted text |
| Success | #34d399 | Success states, confirmations |
| Warning | #fbbf24 | Warnings, pending states |
| Error | #f87171 | Errors, destructive actions |

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
- Variant: Rounded
- Border radius: 8px (sm: 4px, lg: 12px, xl: 16px)
- Shadows: Subtle — `0 1px 3px rgba(0,0,0,0.1)`
- Theme: Light only — single fixed palette, no theme switcher

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#5e6ed2',
    secondary: '#8a90a8',
    accent: '#636bf1',
    background: '#fafafa',
    surface: '#ffffff',
    foreground: '#08090a',
    muted: '#6c6f7a',
    success: '#10b981',
    warning: '#f59e0b',
    destructive: '#ef4444',
  },
  fontFamily: {
    heading: ['Space Grotesk', 'system-ui', 'sans-serif'],
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
  --color-primary: #5e6ed2;
  --color-secondary: #8a90a8;
  --color-accent: #636bf1;
  --color-background: #fafafa;
  --color-surface: #ffffff;
  --color-text: #08090a;
  --color-text-secondary: #6c6f7a;
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --font-heading: 'Space Grotesk', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 8px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #7074ff;
  --color-secondary: #b4b6c1;
  --color-accent: #8190f8;
  --color-background: #08090a;
  --color-surface: #131318;
  --color-text: #f7f8f8;
  --color-text-secondary: #8a8f98;
  --color-success: #34d399;
  --color-warning: #fbbf24;
  --color-error: #f87171;
}
```

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

An HTML design template was uploaded with this project (built by the customer
in Claude Design / artifacts). Fetch it at build time via the
`get_design_template` MCP tool — that is the source of truth, not this section.

### How to use it
- **Copy the template's `:root` design tokens verbatim FIRST.** The template
  declares its palette, typography, spacing scale, radius, and shadow as CSS
  custom properties on `:root` (`--color-*`, `--font-*`, `--space-*`,
  `--radius-*`, `--shadow-*`). Transcribe those values EXACTLY into the
  project's token layer (CSS variables / Tailwind theme / equivalent) — do not
  re-interpret, round, average, or "improve" them. This copied set is the
  canonical design-token system for the whole build.
- If the build needs a token the template did not declare (e.g. a missing
  state color or extra elevation), derive it to sit alongside the copied
  palette — never replace a value the template set.
- Use the markup as the visual scaffold for the landing/home surface — port
  its sections, hero, nav, and components into the project's framework rather
  than reinventing the layout.
- The copied tokens are binding for the ENTIRE build, OVERRIDING the values in
  `get_project_config` / the Design Language section below wherever they
  conflict. Express archetype-driven screens (anything not in the template)
  through these same tokens so the whole app stays visually coherent.
- Save the raw template to the project root as `DESIGN-TEMPLATE.html`, and
  record in `DECISIONS.md` the exact `:root` block you copied.

---

# Research (Stage 0) — added by build pipeline

> build_depth: comprehensive → all source categories, ≥2–3 analyzed entries
> each, plus a threat-model assessment in §5. Versions verified against vendor
> docs / npm during research. Exact installed versions are pinned at build time
> (npm resolves the real stable release); the figures below record what research
> surfaced as current.

## 3. Source Categories

### 3.1 Official / Vendor Documentation

| Library | What it's for | Version (research) | License | Docs |
|---|---|---|---|---|
| **React** | Frontend UI library | 19.x | MIT | https://react.dev |
| **Vite** | Frontend build/dev tooling | 7.x/8.x | MIT | https://vite.dev |
| **Node.js** | Server runtime (Active LTS) | 22/24 LTS | MIT-style | https://nodejs.org/en/about/previous-releases |
| **TypeScript** | Typed superset of JS | 5.x (stable) | Apache-2.0 | https://www.typescriptlang.org/docs/ |
| **Express** | HTTP framework / routing (v5 stable) | 5.x | MIT | https://expressjs.com/en/guide/migrating-5.html |
| **TypeORM** | ORM (entities, migrations, repositories) | 0.3.x | MIT | https://typeorm.io |
| **PostgreSQL** | Relational DB (RLS, JSONB, PITR) | 16/17/18 | PostgreSQL License | https://www.postgresql.org/support/versioning/ |
| **Socket.IO** | Realtime (rooms/namespaces/reconnect) + Redis adapter | 4.8.x | MIT | https://socket.io/docs/v4/ |
| **BullMQ** | Redis-backed job queue (routing, notifications, delayed SLA jobs) | 5.x | MIT | https://docs.bullmq.io |
| **ioredis** | Redis client for BullMQ / rate-limit / adapter | 5.x | MIT | https://github.com/redis/ioredis |
| **Zod** | Runtime schema validation + TS inference | 3.x/4.x | MIT | https://zod.dev |
| **argon2 / bcrypt** | Password hashing — Argon2id (OWASP default) | latest | MIT | https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html |
| **jsonwebtoken** | JWT signing/verification (access + refresh) | 9.x | MIT | https://github.com/auth0/node-jsonwebtoken |
| **express-rate-limit + rate-limit-redis** | Distributed rate limiting | latest | MIT | https://www.npmjs.com/package/express-rate-limit |
| **Helmet** | Security HTTP headers (CSP, HSTS, …) | 8.x | MIT | https://helmetjs.github.io |
| **pino** | Fast structured JSON logging | 9.x/10.x | MIT | https://github.com/pinojs/pino |
| **@sentry/node** | Centralized error tracking | 8.x+ | MIT | https://docs.sentry.io/platforms/javascript/guides/node/ |
| **@opentelemetry/sdk-node** | APM — tracing/metrics (vendor-neutral) | latest | Apache-2.0 | https://opentelemetry.io/docs/languages/js/ |
| **Multer + sharp** | Multipart upload + image transcode/EXIF strip | 2.x / 0.34.x | MIT / Apache-2.0 | https://sharp.pixelplumbing.com |
| **PDFKit** | Programmatic PDF report export | 0.17.x | MIT | https://pdfkit.org |
| **fast-csv** | Stream-based CSV export | 5.x | MIT | https://c2fo.github.io/fast-csv/ |
| **Playwright** | End-to-end browser testing | 1.x | Apache-2.0 | https://playwright.dev |
| **Vitest** | Unit/integration testing | 1.x–4.x | MIT | https://vitest.dev |

### 3.2 GitHub Repositories
| Repo | ~Stars | Active | License | Relevance |
|---|---|---|---|---|
| [Chatwoot](https://github.com/chatwoot/chatwoot) | ~29k | Active | MIT/BSL (verify) | Omni-channel support platform; agents, SLAs, realtime reference |
| [Zammad](https://github.com/zammad/zammad) | ~5k+ | Active | AGPLv3 | Helpdesk: ticket states, escalation, SLA, audit trails (ref only) |
| [FreeScout](https://github.com/freescout-help-desk/freescout) | ~4k+ | Active | AGPLv3 | Lightweight shared-inbox helpdesk; simple data model |
| [osTicket](https://github.com/osTicket/osTicket) | ~3k+ | Maintained | GPL | Classic ticketing; SLA plans, queues/routing (ref only) |
| [BullMQ](https://github.com/taskforcesh/bullmq) | ~6k+ | Very active | MIT | Job-queue reference impl we depend on |

> AGPL/GPL repos are architectural reference only — code not vendored into this proprietary SaaS.

### 3.3 Video / Tutorials
- BullMQ queues & worker/retry/backoff patterns — https://dev.to/young_gao/background-job-processing-in-nodejs-bullmq-queues-and-worker-patterns-31d4
- Socket.IO official "Performance tuning" (scaling realtime + Redis adapter) — https://socket.io/docs/v4/performance-tuning/
- Production logging in Node with Pino — https://github.com/pinojs/pino/blob/main/docs/help.md

### 3.4 Articles / Blog Posts
- Argon2id vs bcrypt parameter guidance — https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- Problem Details (RFC 9457): doing API errors well — https://swagger.io/blog/problem-details-rfc9457-doing-api-errors-well/
- Transactional Outbox pattern — https://microservices.io/patterns/data/transactional-outbox.html

### 3.5 Standards / RFCs
| Standard | Relevance | URL |
|---|---|---|
| **ITIL 4** | Incident & Service Request lifecycle, priority/impact/urgency, escalation | https://www.axelos.com/certifications/itil-service-management |
| **SOC 2 — 2017 TSC (rev. 2022)** | Compliance baseline (Security/Availability/Confidentiality/PI/Privacy) | https://www.aicpa-cima.com |
| **OWASP Top 10:2021** | Primary web-app security risk baseline | https://owasp.org/Top10/2021/ |
| **OWASP ASVS** | Security verification checklist | https://owasp.org/www-project-application-security-verification-standard/ |
| **NIST SP 800-63B** | Password/auth rules (length-not-composition, blocklist, no forced rotation) | https://pages.nist.gov/800-63-3/sp800-63b.html |
| **RFC 9457** (obsoletes 7807) | `application/problem+json` error responses | https://www.rfc-editor.org/rfc/rfc9457.html |
| **RFC 6238 (TOTP)** | MFA via authenticator apps | https://www.rfc-editor.org/rfc/rfc6238 |
| Webhook HMAC signing (Stripe/Svix de-facto) | Outbound signing + inbound verify w/ timestamp + constant-time compare | https://www.svix.com |

### 3.6 Competing Products
| Product | Strength | ServiceHub difference |
|---|---|---|
| ServiceNow | Deep enterprise ITSM workflow engine | Simplicity/speed, low setup cost for mid-market |
| Zendesk | Polished support UX, app marketplace | Transparent model, API/webhook-first, self-host-friendly |
| Jira Service Management | Tight dev-tool integration | Purpose-built ticketing without Jira config overhead |
| Freshservice | Good mid-market ITSM | Leaner surface, faster realtime updates |
| Zammad (OSS) | Free, full-featured, self-hosted | Modern TS/React stack + SaaS operability (PITR, APM) |

### 3.7 Community Threads
- TypeORM migrations in production (avoid `synchronize:true`, schema drift) — https://github.com/typeorm/typeorm/issues
- BullMQ reliability / stalled jobs (lock duration, idempotent workers, graceful shutdown) — https://docs.bullmq.io/guide/jobs/stalled
- SLA timer / business-hours implementation (pause on status, calendar deadlines) — https://stackoverflow.com/questions/tagged/typeorm

### 3.8 APIs / Integrations
| Integration | Notes | Docs |
|---|---|---|
| Outbound webhook delivery | HMAC-SHA256 sign, timestamp, exp backoff, DLQ; via outbox + BullMQ | https://www.svix.com |
| Email — SMTP (Nodemailer) | Provider-agnostic transport; default/dev via MailHog | https://nodemailer.com/ |
| Email — SendGrid/Postmark/SES | Transactional + signed event webhooks (bounce/complaint) | https://www.twilio.com/docs/sendgrid |
| Inbound monitoring webhooks | Ingest alerts → auto-create tickets; verify shared secret before DB write | https://prometheus.io/docs/alerting/latest/configuration/#webhook_config |
| Slack / Teams notifications | Incoming Webhooks for ticket alerts | https://api.slack.com/messaging/webhooks |

### 3.9 Patterns
| Pattern | One-liner | Reference |
|---|---|---|
| Multi-tenant isolation (org scoping) | Every row carries `org_id`; mandatory query scope + Postgres RLS as defense-in-depth | https://www.postgresql.org/docs/current/ddl-rowsecurity.html |
| SLA timer / escalation engine | Persist absolute deadlines; delayed BullMQ jobs re-evaluated on status change; business-hours aware | https://docs.bullmq.io/guide/jobs/delayed |
| Transactional Outbox | Domain change + outbox row in one tx; relay publishes webhooks/emails (at-least-once) | https://microservices.io/patterns/data/transactional-outbox.html |
| Idempotent webhook receiver | Verify HMAC + timestamp BEFORE any DB work; dedupe by event id (unique constraint) | https://www.svix.com |
| Job retry/backoff + DLQ | Exponential backoff, capped attempts, failed → DLQ | https://docs.bullmq.io/guide/retrying-failing-jobs |
| Append-only audit log | Immutable insert-only audit table (no UPDATE/DELETE) — SOC2 CC7 | https://owasp.org/Top10/2021/A09_2021-Security_Logging_and_Monitoring_Failures/ |
| RBAC | Roles→permissions enforced server-side per route + per org; least privilege | https://owasp.org/Top10/2021/A01_2021-Broken_Access_Control/ |

## 4. Stack Candidates

| Layer | Chosen | Rationale | Alternative → why rejected |
|---|---|---|---|
| Frontend | React + Vite + TypeScript | Mature, fast builds, SPA fits authed dashboard | Next.js → unneeded SSR complexity |
| Runtime | Node.js LTS | LTS stability + security updates | Node Current → not LTS |
| HTTP framework | Express 5 | Stable, ubiquitous middleware (Helmet, rate-limit, Multer) | Fastify (smaller mw ecosystem); NestJS (heavier) |
| ORM | TypeORM | **Mandated**; decorators + migrations | Prisma → constraint specifies TypeORM |
| Database | PostgreSQL | RLS, JSONB, PITR, robust multi-tenant | MySQL → weaker RLS/JSONB |
| Realtime | Socket.IO (+Redis adapter) | Rooms/reconnection for per-org/per-ticket channels | raw ws → rebuild rooms/reconnect |
| Queue | BullMQ + Redis | Reliable queue: routing, notifications, delayed (SLA) jobs | pg-boss (no Redis) → fewer flow features |
| Validation | Zod | TS-first runtime validation at boundary | class-validator → pairs w/ NestJS |
| Auth | jsonwebtoken + argon2 (Argon2id) | JWT access/refresh; Argon2id per OWASP/NIST | bcrypt → acceptable fallback |
| Logging / Errors / APM | pino / Sentry / OpenTelemetry | Structured JSON + centralized errors + vendor-neutral tracing | winston (slower); proprietary APM (lock-in) |
| Uploads / Export | Multer + sharp / PDFKit + fast-csv | EXIF strip + transcode; PDF + streaming CSV | Puppeteer PDF → heavier |
| Testing | Vitest + Supertest + Playwright | Unit/integration + e2e | Jest → fine; Vitest Vite-native |

## 5. Risk Register + Threat Model

### Risk Register
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| SLA-timer inaccuracy under load | Medium | High | Persist absolute deadlines (not in-memory timers); BullMQ delayed jobs; re-evaluate on status change; idempotent escalation worker; business-hours calendar service |
| Webhook delivery not guaranteed | Medium | High | Transactional outbox + BullMQ relay; exp backoff; capped retries → DLQ; signed payloads; delivery status visible |
| Multi-tenant data leakage | Medium | Critical | Mandatory `org_id` scoping in base repo; Postgres RLS defense-in-depth; per-endpoint authz tests; deny-by-default RBAC |
| File-upload abuse (malware/EXIF/SVG XSS/bombs) | Medium | High | Multer size/type limits; sharp re-encode (strips EXIF); block/sanitize SVG; store outside webroot; `Content-Disposition`; decode caps |
| Notification delivery not guaranteed | Medium | Medium | Outbox + queue retries; provider bounce webhooks; in-app fallback; idempotency keys |
| TypeORM migration drift | Medium | High | `synchronize:false` everywhere; review generated migrations; CI schema==entities check; PITR backup before migrate |
| Realtime scaling | Medium | Medium | Redis adapter + sticky sessions; per-org/per-ticket rooms; backpressure; load test |
| Secret leakage | Low | Critical | Env-based secrets; rotate JWT + webhook signing secrets; least-privilege DB creds |
| Rate-limit bypass | Medium | Medium | Distributed rate-limit keyed per org/user/IP; stricter on auth + export |

### Threat Model (STRIDE per trust boundary → control → OWASP 2021)
**Browser ↔ API** — Spoofing: short-lived JWT + refresh rotation, optional TOTP MFA [A07]; Tampering/IDOR: server-side authz + org scoping + RLS [A01]; Info-disclosure: deny-by-default RBAC, field allow-lists [A01]; DoS: rate-limit + HTTPS-only [A05]; all input via Zod [A03].
**API ↔ DB** — Injection: TypeORM parameterized queries [A03]; Elevation: least-privilege DB role + RLS [A01]; Repudiation: append-only audit log [A09]; Disclosure: env secrets + TLS to DB [A02/A05].
**API ↔ Queue (Redis/BullMQ)** — Tampering: Redis auth + TLS + network isolation [A05]; DoS: producer rate limits, bounded concurrency [A05]; Lost jobs: idempotent workers, retries/backoff, DLQ [A09].
**Inbound webhooks** — Spoofing: HMAC verify on raw body **before** DB write [A08]; Replay: timestamp tolerance ≤5 min + event-id dedupe (unique constraint) [A08]; DoS: fast 401 + rate limit + IP allow-list [A05].
**Outbound webhooks** — Receiver verification: sign HMAC-SHA256 [A08]; SSRF: block private/internal IP ranges, DNS-rebinding protection, egress allow-list [A10]; Reliability: outbox + retries + DLQ [A09].
**File uploads** — Malware: sharp re-encode, type/size limits, strip EXIF [A08]; Stored XSS (SVG/HTML): block/sanitize SVG, isolated origin + CSP + `Content-Disposition` [A03]; DoS (bombs): dimension/size caps, decode timeouts [A05].
Cross-cutting: Helmet HSTS/CSP [A05]; dependency scanning/SBOM [A06]; Sentry + pino + audit trail [A09].

## 6. Research Gaps
1. **SLA business-hours calendar rules** — working hours/holidays, per-org timezone, pause/resume semantics (e.g. pause on "Pending Customer"). *Resolution:* default 24×7 SLA clock with configurable business-hours calendar per org; pause on `pending` statuses (logged as ADR in Stage 1).
2. **Email provider** — SendGrid vs Postmark vs SES not selected. *Resolution:* provider-agnostic SMTP (Nodemailer) with MailHog in the dev/local stack; provider swappable via env.
3. **Inbound monitoring webhook formats** — which monitoring systems and signing schemes. *Resolution:* generic signed-webhook receiver (HMAC shared secret) with a normalized alert→ticket mapper.
4. **SOC2 data-retention & residency** — retention window for tickets/audit logs/backups. *Resolution:* default 1-year ticket / 7-year audit retention, configurable; PITR window 7 days (logged as ADR).
5. **MFA scope & SSO** — TOTP mandatory? SSO (SAML/OIDC) in scope? *Resolution:* optional TOTP MFA per user in v1; SSO is a documented Non-Goal for v1.

## 7. Summary + GO / NO-GO

The proposed stack is mature, current, and exceptionally well-documented. Every load-bearing component (React, Vite, Node LTS, TypeScript, Express 5, TypeORM, PostgreSQL, Socket.IO, BullMQ, Zod, Helmet, pino, Sentry, OpenTelemetry, Multer, sharp, PDFKit, fast-csv, Playwright, Vitest) is permissively licensed (MIT/Apache-2.0/PostgreSQL) and actively maintained. Exact versions are pinned at build time from npm.

Security and compliance posture is achievable: OWASP Top 10:2021, ASVS, SOC 2 TSC, NIST SP 800-63B (Argon2id hashing, length-based password policy, no forced rotation), RFC 9457 error envelopes, and RFC 6238 TOTP all map cleanly onto the chosen libraries. The hard parts — multi-tenant isolation (org scoping + RLS), guaranteed delivery (transactional outbox + BullMQ), SLA escalation (persisted deadlines + delayed jobs), idempotent signed webhooks, and append-only audit logging — are established, documented patterns with reference implementations.

The open questions in §6 are product/configuration decisions, not feasibility risks; each has a sensible default recorded for Stage 1. None invalidate the stack.

`Go/No-Go: GO` — every component is verified-current, permissively licensed, actively maintained, and the SOC2 + OWASP baseline is fully satisfiable; remaining gaps are product decisions with recorded defaults, not blockers.

