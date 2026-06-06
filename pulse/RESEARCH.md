# RESEARCH.md — Pulse

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Infrastructure monitoring"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_websocket", "has_geo"]
```

## Product Vision
**Problem:** IT and operations teams struggle to quickly identify outages, performance degradation, and infrastructure issues across distributed environments. Existing monitoring platforms are often expensive, overly complex, or require significant expertise to manage.

**Who it affects:** Network Operations Center (NOC) staff managing 24/7 monitoring shifts across multi-site infrastructure, needing rapid incident triage to minimize MTTR and reduce false-positive alert storms that cause burnout. Infrastructure Engineers responsible for capacity planning and performance optimization who need historical trend analysis and predictive insights without manual data aggregation across fragmented tools. System Administrators in mid-market organizations handling both infrastructure and application monitoring with limited staff, requiring minimal configuration overhead and self-service troubleshooting to reduce escalations. Managed Service Providers delivering monitoring as part of managed services, needing per-customer isolation, white-label capabilities, and cost-efficient scaling to maintain margins as customer counts grow. Enterprise IT Operations teams coordinating across multiple business units and geographies, struggling with alert correlation across siloed monitoring systems and needing unified visibility to coordinate incident response across teams.

**Why existing solutions fall short:** Most infrastructure monitoring platforms have evolved into large, complex ecosystems that require significant time, expertise, and cost to deploy and maintain. Teams often struggle with alert fatigue, complicated configuration, fragmented dashboards, and excessive feature sets that obscure critical operational insights. Many solutions become expensive as device counts grow, making enterprise-scale monitoring difficult to justify. As environments expand across data centers, cloud providers, and remote locations, organizations need a simpler platform that delivers fast issue detection, clear visibility, and actionable insights without the overhead and complexity of traditional monitoring tools.

**Solution:** Pulse is a scalable infrastructure monitoring platform delivering real-time visibility into server, network, application, and service health. It prioritizes rapid anomaly detection and alerting with minimal operational overhead, enabling teams to troubleshoot faster while maintaining simplicity in deployment and scaling.


## Users & Outcomes
**Key Workflows:**
**Infrastructure Discovery**
NOC staff automatically discover and inventory servers, network devices, applications, and services across distributed environments within 15 minutes of deployment, reducing manual asset tracking overhead by 80% and eliminating blind spots in monitoring coverage.

Infrastructure Engineers maintain an accurate, real-time asset inventory that auto-updates when infrastructure changes, enabling capacity planning decisions based on current infrastructure state rather than outdated spreadsheets.

**Health & Availability Monitoring**
NOC operators continuously monitor CPU, memory, disk, uptime, network interfaces, and service availability through ICMP, TCP, HTTPS, DNS, and custom health checks, detecting infrastructure degradation within 30 seconds of occurrence to minimize customer impact.

System Administrators configure health checks for their infrastructure with zero custom scripting, reducing setup time from hours to minutes and enabling self-service monitoring without engineering expertise.

**Alerting & Escalation**
NOC staff receive correlated, context-rich alerts through email, SMS, Teams, Slack, or webhooks that distinguish critical outages from noise, reducing alert fatigue by 70% and enabling faster incident triage during 24/7 shifts.

Managed Service Providers route customer-specific alerts to appropriate teams with per-customer isolation, maintaining SLA compliance while scaling to hundreds of customers without proportional operational overhead.

**Incident Investigation**
Infrastructure Engineers access real-time metrics, historical trends, dependency mapping, and diagnostic data in a single interface, identifying root causes and affected systems in under 5 minutes rather than correlating data across fragmented tools.

Enterprise IT Operations teams visualize cross-system dependencies and trace incidents across multiple business units and geographies, coordinating response across siloed teams without manual escalation delays.

**Operational Visibility**
NOC staff view unified infrastructure health through dashboards, geographic maps, and topology views, replacing fragmented monitoring systems with a single-pane-of-glass that reduces mean time to resolution by 40%.

System Administrators troubleshoot infrastructure issues independently using clear operational visibility, reducing escalations to engineering teams by 60% and improving incident response during off-hours.

**Success Metrics:**
Sub-minute issue detection
99.9%+ monitoring platform availability
Less than 5-minute mean time to detect (MTTD)
Reduced mean time to resolve (MTTR)
Support for 50,000+ monitored devices
High alert accuracy with minimal false positives

**Non-Goals:**
Full SIEM functionality
Endpoint management or patching
Configuration management
Asset lifecycle management
IT service desk replacement


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "comprehensive"
container_strategy: "Docker"

# Data & Storage
database_preference: "PostgreSQL"

# Notifications & Messaging
notification_urgency: "guaranteed delivery"
webhook_targets: ["Slack", "Teams", "custom webhooks"]
sms_service: "Twilio"

# Security & Compliance
security_baseline: ["SOC2", "OWASP Top 10"]
rate_limiting: true
audit_logging: true
secrets_management: "environment variables"

# Backend
api_style: "REST"
api_versioning: "URL path (/v1/)"
realtime_needed: true
background_jobs: "job queue"

# Performance & Quality
performance_requirements: ["sub-minute issue detection", "MTTD < 5 minutes", "API response < 200ms", "99.9% platform availability"]
logging_format: "structured JSON"

# Scope & Platform
scale: "large — 50k+"
multi_tenant: true
target_platforms: ["web"]
alert_channels: ["email", "SMS", "Slack", "Teams", "webhooks"]
report_formats: ["JSON", "CSV"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; the uploaded Design Template's `:root` tokens override them per the precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Clear, reliable, operational, and actionable. Present infrastructure health in a way that allows operators to identify, understand, and resolve issues within seconds. Focus on simplicity, speed, and operational confidence.

### Theme
Light only — single fixed palette, no theme switcher (per project config).

### Typography
- Heading/Display: Manrope (600/700/800 weight)
- Body: Inter (400/500/600 weight)
- Mono: JetBrains Mono (code, labels, telemetry)

### Layout
- Pattern: Topnav + Content (app shell with sidebar for authenticated area)
- Max width: 1280px (marketing 1200px container)

### Design Template
An HTML design template was uploaded with this project and fetched at build time via `get_design_template`. Its `:root` tokens are copied verbatim into the token layer (see DECISIONS.md ADR-0002). Raw template saved at project root as `DESIGN-TEMPLATE.html`.

---

## 3. Source Categories

### 3.1 Official / Vendor Documentation

| Component | Verified current version (Jun 2026) | Official docs URL | Key command / API signature confirmed |
|---|---|---|---|
| Node.js | 24.x Active LTS (22 now Maintenance LTS) | https://nodejs.org/en/about/previous-releases | Target Node 24 LTS for security window |
| TypeScript | 5.9.x (6.0 also GA) | https://www.typescriptlang.org/docs/ | `tsc --build` |
| Express | 5.x production-recommended (4.21.x maintained) | https://expressjs.com/ | `app.use('/v1', router)` path versioning |
| PostgreSQL | 18.x (16/17 supported) | https://www.postgresql.org/docs/ | PG16+ acceptable; 18 current |
| Drizzle ORM | drizzle-orm + drizzle-kit (active) | https://orm.drizzle.team/docs/kit-overview | `drizzle-kit generate` then `drizzle-kit migrate`; `__drizzle_migrations` |
| BullMQ | 5.x | https://docs.bullmq.io/ | `new Queue(name,{connection})`; `new Worker(name,proc,{connection})`; repeatable/cron jobs |
| Redis / Valkey | Redis 8 (AGPLv3) / Valkey 8 (BSD drop-in) | https://redis.io/docs/ | ioredis client; Valkey preferred for permissive license |
| Socket.IO | 4.8.x | https://socket.io/docs/v4/ | `new Server(httpServer,{cors})`; `io.to(room).emit(evt,payload)` per-tenant rooms |
| React | 19.x | https://react.dev/ | function components + hooks |
| Vite | 6+ | https://vite.dev/ | `@vitejs/plugin-react` |
| Tailwind CSS | 3.4.x / 4.x | https://tailwindcss.com/docs | utility-first, token theme |
| TanStack Query | 5.x | https://tanstack.com/query/latest | `useQuery({queryKey,queryFn})` |
| React Router | 6/7.x | https://reactrouter.com/ | SPA routing |
| Recharts | 2/3.x | https://recharts.org/ | `<LineChart><Line dataKey=.../></LineChart>` |
| argon2 (node) | 0.4x (Node >=22) | https://github.com/ranisalt/node-argon2 | `argon2.hash(pw)` (Argon2id), `argon2.verify(hash,pw)` |
| Twilio Node SDK | 5/6.x | https://www.twilio.com/docs/libraries/reference/twilio-node/ | `client.messages.create({body,from,to})`; `twilio.validateRequest(token,sig,url,params)` |
| nodemailer | 6/7.x | https://nodemailer.com/ | `createTransport({...}).sendMail({...})` |
| Zod | 3/4.x | https://zod.dev/ | `z.object({...}).safeParse(data)` |
| Pino | 9/10.x | https://getpino.io/ | `pino({level}).info({...fields}, msg)` structured JSON |
| Vitest | 1-4.x | https://vitest.dev/ | `vitest run` |
| Playwright | active | https://playwright.dev/ | `npx playwright test` |
| Docker Compose | Compose V2 (`docker compose`) | https://docs.docker.com/compose/ | `docker compose up -d`; no top-level `version:` key |

**Deep analysis.**
1. **Drizzle ORM (vs Prisma).** Two-step SQL-transparent migrations (`drizzle-kit generate` emits reviewable `.sql`, `migrate` applies, tracked in `__drizzle_migrations`) — every schema change is an auditable artifact (SOC2). No engine binary, so small worker image and fast cold start. Multi-tenant scoping is done in app code via a `scopedDb(orgId)` wrapper injecting `org_id`; layer Postgres RLS underneath as defense-in-depth.
2. **BullMQ scheduler + delivery.** Repeatable/cron jobs key each monitor's checks; `attempts` + exponential backoff + dead-letter for guaranteed-delivery notifications. At 50k scale add per-job jitter to avoid thundering herd. Redis/Valkey is the critical scaling dependency, so it must be HA.
3. **Twilio inbound verification.** `validateRequest(authToken, signature, url, params)` is HMAC-SHA1 over full URL + sorted params; preserve blank params; reconstruct the exact external URL behind proxies (SOC2 + OWASP A08).

### 3.2 GitHub Repositories

| Repo | Stars (approx) | Last active | License | Relevance |
|---|---|---|---|---|
| louislam/uptime-kuma | ~87,600 | Active 2026 | MIT | Closest UX analog; realtime push model |
| TwiN/gatus | ~11,100 | Active 2026 | Apache-2.0 | Config-as-code health-check semantics |
| statping-ng/statping-ng | community fork | Maintained | GPL-3.0 | Status page + notifier-plugin model (learning only) |
| expressjs/express | core | Active | MIT | API framework |
| drizzle-team/drizzle-orm | active | Active 2026 | Apache-2.0 | ORM |
| taskforcesh/bullmq | active | recent | MIT | Job queue |
| socketio/socket.io | active | Active | MIT | Realtime |
| colinhacks/zod | active | Active | MIT | Validation |
| pinojs/pino | active | Active | MIT | Logging |
| ranisalt/node-argon2 | active | Active | MIT | Password hashing |

**Reference architecture lessons.** (1) **Uptime Kuma (MIT)** — keep the Socket.IO realtime UX but decouple the checker into a scalable BullMQ worker pool (Kuma's single-process model is its scaling ceiling). (2) **Gatus (Apache-2.0)** — its declarative condition model (`[STATUS]==200`, `[RESPONSE_TIME]<300`) is a great template for Pulse's custom assertions; statelessness makes checkers cheap to scale. (3) **Statping-ng (GPL-3.0)** — its one-interface-many-channels notifier abstraction is a clean fan-out model (reference only; GPL).

### 3.3 Video / Tutorials
- "Socket.IO: The Complete Guide to Building Real-Time Web Applications (2026)" — DEV. Per-tenant room patterns. https://dev.to/abanoubkerols/socketio-the-complete-guide-to-building-real-time-web-applications-2026-edition-c7h
- "Background Job Processing in Node.js: BullMQ, Queues, and Worker Patterns (2026)" — DEV. Repeatable jobs, retry/backoff. https://dev.to/young_gao/background-job-processing-in-nodejs-bullmq-queues-and-worker-patterns-31d4
- "Tailwind CSS v4 Full Course 2026" — YouTube. CSS-first config migration. https://www.youtube.com/watch?v=6biMWgD6_JY

### 3.4 Articles / Blog Posts
- "Production Logging in Node.js with Pino — A Complete Guide" — Medium 2026. Structured JSON + secret redaction. https://medium.com/@nannurimanoj26/production-logging-in-node-js-with-pino-js-a-complete-guide-b69f5576603b
- "A Complete Guide to Pino Logging in Node.js" — Better Stack. Child loggers / correlation IDs. https://betterstack.com/community/guides/logging/how-to-install-setup-and-use-pino-to-log-node-js-applications/
- "Redis vs Valkey in 2026: What the License Fork Actually Changed" — DEV 2026. Backing-store decision. https://dev.to/synsun/redis-vs-valkey-in-2026-what-the-license-fork-actually-changed-1kni

### 3.5 Standards / RFCs

| Standard | Scope | URL |
|---|---|---|
| RFC 792 — ICMP | Ping/echo checks | https://www.rfc-editor.org/rfc/rfc792 |
| RFC 9293 — TCP | TCP port checks | https://www.rfc-editor.org/rfc/rfc9293 |
| RFC 1035 — DNS | DNS checks | https://www.rfc-editor.org/rfc/rfc1035 |
| RFC 9110 — HTTP Semantics | HTTPS checks, REST status codes | https://www.rfc-editor.org/rfc/rfc9110 |
| RFC 9457 — Problem Details (obsoletes 7807) | `application/problem+json` errors | https://www.rfc-editor.org/rfc/rfc9457.html |
| OWASP ASVS 5.0 | Security verification baseline | https://owasp.org/www-project-application-security-verification-standard/ |
| OWASP Top 10 (2021) | Threat-model mapping | https://owasp.org/Top10/ |
| SOC 2 Trust Services Criteria | Compliance baseline | https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2 |
| WCAG 2.2 AA | Dashboard accessibility | https://www.w3.org/TR/WCAG22/ |
| RFC 2104 — HMAC | Outbound webhook signing | https://www.rfc-editor.org/rfc/rfc2104 |

> Correction: RFC 7807 is obsoleted by **RFC 9457** (backward-compatible). Pulse targets 9457.

### 3.6 Competing Products
- **Datadog** — full-stack APM/infra; powerful, expensive, complex. *Pulse:* focused uptime/health at a fraction of cost/setup.
- **Prometheus + Grafana** — pull-based OSS standard; steep ops burden. *Pulse:* turnkey SaaS with built-in multi-channel alerting.
- **Uptime Kuma** — beloved self-hosted single-node; no multi-tenancy. *Pulse:* multi-tenant, 50k scale, SOC2.
- **PagerDuty** — incident/on-call orchestration, not a checker. *Pulse:* integrated checks to incidents (can also feed PagerDuty).
- **Better Stack (Uptime)** — closest commercial analog. *Pulse:* competes on simplicity + price + topology/geo.
- **Zabbix / Nagios** — mature/legacy, agent-heavy, dated UX. *Pulse:* modern web-first UX, fast onboarding.

**Wedge:** simplicity + speed-to-value + transparent cost; multi-protocol checks, real-time dashboards, guaranteed-delivery alerting without Datadog pricing or Prometheus ops overhead.

### 3.7 Community Threads
- Office 365 connector retirement (Teams webhooks) — Microsoft Q&A. https://learn.microsoft.com/en-us/answers/questions/5854971/will-teams-incoming-webhooks-stop-delivering-alert
- Twilio signature validation in practice — chatwoot#13619. https://github.com/chatwoot/chatwoot/issues/13619
- Self-hosted uptime comparison (Kuma vs Gatus vs Statping). https://selfhostedguides.com/uptime-monitoring-self-hosted/

### 3.8 APIs / Integrations
- **Twilio Messages API.** Basic auth (SID+Token) over HTTPS; `POST .../Accounts/{SID}/Messages.json`; SDK `client.messages.create({body,from,to})`. Inbound callback verify: HMAC-SHA1, header `X-Twilio-Signature`, `validateRequest(...)`. Preserve blank params; exact external URL behind proxies.
- **Slack Incoming Webhooks.** `POST` JSON to per-channel URL; `{text}` or Block Kit `{blocks}`. The secret *is* the URL, so store encrypted.
- **Microsoft Teams.** O365 connector "Incoming Webhook" retired May 2026, so use **Power Automate Workflows** webhook URL (supports MessageCard). Flow-bot default identity.
- **Generic outbound webhook (RFC 2104).** `X-Pulse-Signature: sha256=HMAC-SHA256(secret, timestamp + "." + body)` + `X-Pulse-Timestamp`; reject stale timestamps (replay); constant-time compare.

### 3.9 Patterns / Best Practices
- **Distributed scheduler.** Each monitor = BullMQ repeatable job keyed by id; interval change re-upserts; per-job jitter; stateless autoscaled workers; Redis coordinates.
- **Alert state machine.** `OK to DEGRADED to DOWN` + recovery with flap detection (N-consecutive / failure-ratio hysteresis); persist transition timestamps — primary alert-fatigue control.
- **Notification fan-out + guaranteed delivery.** One job per channel; `attempts`+backoff+DLQ; idempotency keys (no double-send); per-channel circuit breakers.
- **Dedup / correlation.** Coalesce by (monitor,incident); cooldown suppression; topology-aware suppression of child failures under a parent incident.
- **Multi-tenant isolation.** `scopedDb(orgId)` wrapper + Postgres RLS; indexed `org_id` on every tenant table; isolation integration tests; deny-by-default.
- **Time-series retention/downsampling.** Raw to rollups (1m to 5m to 1h), tiered retention; TimescaleDB hypertables + continuous aggregates + compression for 50k write volume (partition-ready on vanilla Postgres).
- **WebSocket auth + rooms.** Authenticate handshake (session/JWT); `socket.join('org:'+orgId)`; emit only to that room; Redis adapter to scale across instances.
- **Per-tenant rate limiting.** Redis token-bucket keyed by org (+IP); separate auth/read/write tiers; `429` + `Retry-After` (RFC 9110).

---

## 4. Stack Candidates

| Subsystem | Chosen | Alternatives | Rationale |
|---|---|---|---|
| Runtime | Node.js 24 LTS | Node 22, Bun | 24 = Active LTS security window |
| Language | TypeScript 5.x | TS 6.0 | Mature, ecosystem-stable |
| API framework | Express 5 | Fastify, Hono | Production-recommended; simple `/v1` path versioning; huge ecosystem |
| ORM | Drizzle ORM | Prisma, Kysely | SQL-transparent migrations (SOC2), no engine binary |
| Database | PostgreSQL 16+ (+ TimescaleDB optional) | ClickHouse for metrics | Constraint = Postgres; partition-ready metrics table |
| Queue | BullMQ + Redis/Valkey | RabbitMQ, pg-boss | Repeatable jobs + retries/DLQ = scheduler & guaranteed delivery |
| Realtime | Socket.IO (+ Redis adapter) | raw ws, SSE | Rooms-per-tenant + horizontal scaling + reconnection |
| Frontend | React 19 + Vite + TS | — | Current, supported |
| Styling | Tailwind CSS | — | Token-driven; design-template tokens |
| Server-state | TanStack Query 5 | SWR | Object-form API, caching |
| Routing | React Router | — | SPA routing |
| Charts | Recharts | visx, ECharts | Good dashboard default |
| Auth/hashing | argon2 (Argon2id) | bcrypt | OWASP-preferred |
| Validation | Zod | Valibot | Shared schemas API+frontend |
| Logging | Pino | Winston | Fast structured JSON, redaction |
| SMS | Twilio Node | Vonage, SNS | Mature, signed callbacks |
| Email | nodemailer | Resend, SES | Transport-agnostic |
| Testing | Vitest + Playwright | Jest + Cypress | Vite-native unit + E2E |
| Container | Docker + Compose V2 | Podman | Constraint = Docker |
| CI | GitHub Actions | GitLab CI | Native to GitHub |

---

## 5. Risk Register + Threat Model

### (a) Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| False-positive alert storms (flapping) | High | High | Flap detection (N-consecutive/failure-ratio hysteresis), dedup by (monitor,incident), cooldown, topology suppression |
| Check-scheduling drift at 50k | Medium | High | BullMQ repeatable jobs + jitter; queue-depth metrics; autoscale workers; backlog alerts |
| Notification delivery failure | Medium | High | Retries + backoff + DLQ; idempotency keys; per-channel circuit breakers; delivery audit log |
| Multi-tenant data leakage | Low | Critical | `scopedDb(orgId)` + Postgres RLS; indexed `org_id`; isolation tests; deny-by-default |
| WebSocket scaling | Medium | Medium | Socket.IO Redis adapter; rooms-per-tenant; connection caps; token handshake |
| Postgres write load from metrics | High | High | Partition-ready metrics + rollups/downsampling (TimescaleDB optional); batch inserts |
| Secret leakage | Low | Critical | Env-var secrets; Pino redaction; never log tokens/URLs; encrypt channel secrets at rest |
| Rate-limit bypass | Medium | Medium | Redis token-bucket keyed by tenant+IP; gateway-level; `429`+`Retry-After`; per-route tiers |
| Redis/Valkey SPOF | Medium | High | HA (Sentinel/Cluster/managed); health-monitor the monitor |
| Inbound webhook spoofing | Medium | High | `validateRequest` (Twilio); HMAC + replay window (generic); reject unsigned/stale |

### (b) Threat Model (STRIDE) — Trust Boundaries

**B1. Anonymous Internet to API.** Spoofing/Elevation: credential stuffing to Argon2id, lockout/rate-limit, optional MFA (A07). Tampering/Disclosure: injection/mass-assignment to Zod validation, parameterized queries, HTTPS-only (A03). DoS: floods to per-tenant+IP rate limiting, body-size limits, timeouts (A05). SOC2: TLS, auth audit log.

**B2. Authenticated tenant to other tenants' data.** IDOR/Elevation to `scopedDb(orgId)` + Postgres RLS, object-level authz, deny-by-default (A01). Cross-tenant writes to same scoping; isolation integration tests. SOC2: access control CC6, tenant-scoped audit log.

**B3. Inbound webhooks to handler.** Spoofing/Tampering: forged callbacks to `validateRequest` (HMAC-SHA1) for Twilio; HMAC-SHA256 + timestamp/replay window generic; constant-time compare (A08). Repudiation to log every inbound webhook + validation result.

**B4. Outbound notification targets.** Disclosure/SSRF via user-supplied custom-webhook URLs to egress allow-list / block internal IP ranges, require https scheme, sign payloads (A10). Tampering/MITM to TLS-only, verify cert. SOC2: encrypt channel secrets at rest, delivery audit trail.

**B5. Worker to DB.** Elevation/Tampering: over-privileged creds, injection from check results to least-privilege worker DB role, parameterized writes, validate/normalize external responses before storing (A03/A05). Disclosure: verbose logs to Pino redaction. SOC2: TLS to Postgres, change-managed migrations as audit artifacts.

**Cross-cutting:** structured JSON audit logging with correlation IDs (Repudiation), env-var secrets + redaction (A02), CI dependency scanning (A06), HTTPS-only + HSTS, RFC 9457 problem+json for non-leaky errors.

---

## 6. Research Gaps
1. **Agent-based vs agentless OS metrics.** ICMP/TCP/HTTPS/DNS are agentless; OS-level CPU/mem/disk implies an agent or SNMP/SSH. Decision: Pulse ingests OS metrics via an authenticated push endpoint (`/v1/ingest`) so a lightweight agent OR agentless collector can post — keeps MVP agentless-first. **Resolved in architecture.**
2. **Teams webhook migration.** O365 connectors disabled May 2026, so target Power Automate Workflows webhook (accepts MessageCard JSON). **Resolved: Teams channel posts MessageCard payload.**
3. **Time-series storage at 50k+.** TimescaleDB is the Postgres-native answer; if unavailable, partitioned tables + rollup job. **Decision: metrics table designed to work on vanilla Postgres (partition-ready), Timescale optional.**
4. **Redis vs Valkey licensing.** Both run BullMQ. **Decision: Valkey-compatible; default `redis` image for local dev (interchangeable).**

---

## 7. Summary + GO/NO-GO

**Feasibility.** Every component of the stack is real, current, actively maintained, and well documented as of June 2026. The architecture — Express REST API with `/v1` path versioning, PostgreSQL + Drizzle, BullMQ/Redis for both the health-check scheduler and guaranteed-delivery notifications, Socket.IO for per-tenant realtime, and a React/Vite/Tailwind/TanStack frontend — is a proven, conventional pattern. Reference projects (Uptime Kuma, Gatus, Statping-ng) prove the checker/notifier model; Pulse's differentiation (multi-tenancy, scale, SOC2, incident management, topology/geo) is additive engineering, not novel research.

**Version posture.** Targets are modernized to current stable majors where they install cleanly: Node 24 LTS, Express 5, React 19, Vite, Tailwind, current Drizzle/BullMQ/Socket.IO/Zod/Pino. Where a current major carries migration friction inside a time-boxed autonomous build, the immediately-prior stable major is an acceptable, documented fallback (logged as an ADR). PostgreSQL 16+ satisfies the constraint. Standards target RFC 9457 (problem+json) and OWASP ASVS 5.0 / Top 10 2021.

**Principal risks & handling.** The two real engineering challenges are (1) scheduling 50k+ checks without drift/thundering-herd — BullMQ repeatable jobs with jitter + stateless autoscaled workers — and (2) metrics write volume — partition-ready metrics table with rollup/downsampling (TimescaleDB optional). Redis/Valkey is the critical-path dependency and must be HA in production. Alert fatigue is handled by a flap-detecting state machine with dedup/correlation. Multi-tenant isolation gets defense-in-depth (`scopedDb` wrapper + Postgres RLS). The one time-sensitive integration item — Microsoft Teams O365 connector retirement — is handled by targeting Power Automate Workflows webhooks from day one.

`Go/No-Go: GO` — the stack and architecture are mature and fully verified; proceed, applying current-major version updates and integrating Microsoft Teams via Power Automate Workflows rather than the retired O365 connectors.
