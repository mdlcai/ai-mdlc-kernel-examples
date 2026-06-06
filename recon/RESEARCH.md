# RESEARCH.md — Recon

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Security"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_geo", "has_websocket", "has_dual_write"]
```

## Product Vision
**Problem:** Organizations often lack visibility into what systems, ports, and services are exposed to the internet. Firewall changes, cloud deployments, vendor modifications, and misconfigurations can introduce new attack paths without security teams being aware. Recon continuously monitors external assets and provides immediate notification when the attack surface changes.

**Who it affects:** Security Operations Centers (SOC) — incident responders and threat hunters who need real-time visibility into attack surface changes to correlate with threat intelligence and prioritize incident response. Pain: manual asset tracking across cloud and on-prem, delayed detection of unauthorized exposures, difficulty proving coverage during incidents.

Cybersecurity Teams — security architects and risk officers who must maintain compliance posture and demonstrate control over external assets. Pain: fragmented visibility across multiple scanners, inability to track exposure trends over time, audit readiness gaps when asked to prove asset inventory accuracy.

Network Engineering Teams — infrastructure owners responsible for firewall rules, DNS, and IP allocation who need to understand downstream security impact of their changes. Pain: unaware when their deployments create unintended exposures, reactive rather than proactive feedback on misconfigurations, no automated notification of drift from intended state.

Infrastructure Operations — cloud and on-prem ops teams managing deployments and service changes who need to catch configuration errors before they become security incidents. Pain: no visibility into what's actually internet-facing versus what should be, manual reconciliation of intended vs. actual exposure, slow feedback loops on security implications of infrastructure changes.

Compliance and Risk Management — compliance officers and auditors who must document and prove control over external assets for regulatory requirements. Pain: point-in-time scan reports don't show continuous monitoring, difficulty demonstrating timely detection and remediation of exposures, audit trails lack historical context.

Managed Security Service Providers (MSSPs) — security teams managing multiple customer environments who need scalable, continuous monitoring across client portfolios. Pain: manual asset discovery per client, inability to offer continuous monitoring without significant overhead, difficulty differentiating service value when using only point-in-time scanners.

**Why existing solutions fall short:** Many vulnerability scanners provide point-in-time assessments but lack continuous monitoring and change detection. Security teams are often forced to manually compare scan results, manage multiple tools, and investigate exposure changes without historical context. Recon focuses on continuous visibility, asset tracking, and actionable change intelligence.

**Solution:** Recon continuously monitors your external attack surface to discover exposed assets, misconfigurations, and security risks before attackers do. It maintains a real-time inventory of internet-facing infrastructure and alerts your team to changes that matter—giving you visibility and control over what's actually exposed to the internet.


## Users & Outcomes
**Key Workflows:**
Asset Discovery Journey
A Security Operations Center analyst registers public IP ranges and domains into Recon, triggering automatic host enumeration and service discovery across the registered scope. The analyst measures success by confirming all known external assets appear in the inventory within 24 hours and that previously unknown exposed services are identified.

Continuous Scanning Journey
A network engineering team configures recurring scan schedules against monitored assets. Recon executes scans on the defined cadence, cataloging open ports, running services, protocols, and certificate details. The team validates effectiveness by comparing scan frequency against their change detection SLA—ensuring no exposure window exceeds their acceptable risk threshold.

Change Detection & Response Journey
A SOC analyst reviews Recon's baseline comparison report and identifies a newly opened port on a production asset. The analyst investigates the change, determines if it's authorized, and measures response time from detection to remediation decision. Recon's historical context eliminates manual log searching and enables faster root cause analysis.

Alert Routing Journey
A compliance officer configures alert delivery to their team's Slack channel and webhook endpoint for downstream ticketing. When Recon detects a service change, the alert routes automatically with asset context and severity. The officer measures success by alert delivery latency and reduction in missed exposure changes.

Audit & Trend Reporting Journey
A risk management lead generates a quarterly report showing asset inventory changes, exposure trends, and remediation velocity. Recon provides audit-ready historical records proving continuous monitoring and enabling compliance documentation. The lead measures value by reducing audit preparation time and demonstrating control effectiveness to stakeholders.

**Success Metrics:**
Asset Discovery Journey: All registered public IP ranges and domains enumerated and cataloged within 24 hours; 100% of known external assets appear in inventory; previously unknown exposed services identified and logged with first-detection timestamp.

Continuous Scanning Journey: Scans execute within ±5 minutes of scheduled time; scan completion time ≤30 minutes per 256-IP subnet; zero missed scan cycles in maintenance window; port/service/certificate data captured with <2-minute staleness.

Change Detection & Response Journey: Exposure changes detected and alerted within 15 minutes of occurrence; baseline comparison includes delta timestamp, asset context, and prior state; analyst investigation time reduced by ≥60% versus manual log review through historical context availability.

Alert Routing Journey: Alert delivery to configured endpoints (Slack, webhook, email) within 2 minutes of detection; 99.95% delivery success rate; each alert includes asset fingerprint, change type, severity classification, and remediation guidance.

Audit & Trend Reporting Journey: Quarterly reports generated in <5 minutes; historical records span ≥12 months with immutable change logs; audit-ready exports include discovery date, scan frequency, detection latency, and remediation timeline per asset; compliance documentation preparation time reduced by ≥70%.

Platform Metrics: Monitor 1,000+ concurrent external assets; <1% false-positive change alerts (verified against authorized change logs); 99.9% uptime SLA with <15-minute incident detection and <1-hour MTTR; support <50ms query latency for asset inventory searches.

**Non-Goals:**
Non-Goals:

Endpoint Protection — Recon monitors external attack surface only; host-level agent deployment, malware detection, and endpoint response are out of scope.

Internal Network Monitoring — Recon focuses on internet-facing assets; internal network segmentation, east-west traffic analysis, and internal vulnerability scanning are excluded.

SIEM Replacement — Recon provides change alerts and asset context; log aggregation, correlation across security tools, and incident timeline reconstruction belong in dedicated SIEM platforms.

EDR/XDR Functionality — Recon detects exposure changes; endpoint detection, behavioral analysis, and cross-tool threat hunting are handled by EDR/XDR solutions.

Penetration Testing Automation — Recon discovers and monitors assets; automated exploitation, payload delivery, and post-compromise assessment are excluded to maintain clear separation from offensive security tooling.

Log Management Platform — Recon generates audit trails for compliance; centralized log storage, parsing, retention policies, and log-based alerting are outside scope.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "comprehensive"
backup_strategy: "continuous"
container_strategy: "Docker"
error_reporting: "Sentry"

# Data & Storage
database_preference: "PostgreSQL"
data_retention_policy: "12 months minimum"
pii_handling: ["IP addresses", "domain names", "certificate details"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"
webhook_targets: ["Slack", "custom endpoints"]

# Security & Compliance
security_baseline: ["SOC2"]
rate_limiting: true
audit_logging: true
secrets_management: "HashiCorp Vault"

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
background_jobs: "Bull"

# Performance & Quality
performance_requirements: ["Asset inventory search <50ms", "Change detection alert <15 minutes", "Scan completion <30 minutes per /24 subnet", "Alert delivery <2 minutes"]
testing_strategy: "TDD"
logging_format: "structured JSON"

# Scope & Platform
scale: "large — 50k+"
multi_tenant: true
target_platforms: ["web"]
alert_channels: ["Slack", "email", "webhook"]
report_formats: ["PDF", "JSON", "CSV"]

# Pipeline Attribution
mdlc_attribution: "full"
```

## Design Language

### Archetype
Archetype: saas

### Brand Voice
Recon is vigilant, precise, and proactive. It delivers clear security intelligence without noise, enabling security teams to understand exactly what is exposed to the internet and what has changed since the last scan. The platform emphasizes visibility, accountability, and continuous attack surface awareness.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #5e60d2 | Buttons, links, active states |
| Secondary | #8a8da8 | Accents, badges, highlights |
| Accent | #6c63f1 | Callouts, hover states |
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
| Primary | #7d70ff | Buttons, links, active states |
| Secondary | #b4b5c1 | Accents, badges, highlights |
| Accent | #8182f8 | Callouts, hover states |
| Background | #08090a | Page background |
| Surface | #131318 | Cards, elevated containers |
| Text | #f7f8f8 | Headings, body text |
| Text Secondary | #8a8f98 | Captions, muted text |
| Success | #34d399 | Success states, confirmations |
| Warning | #fbbf24 | Warnings, pending states |
| Error | #f87171 | Errors, destructive actions |

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
- Theme: Light only — single fixed palette, no theme switcher

### Accessibility
- WCAG AA compliance
- Lighthouse target: 95+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations


---

# Stage 0 Research (§3–§7)

# Recon — Stage 0 Research Appendix

Continuous external attack-surface monitoring SaaS for SOC / security teams.
Research date: 2026-06-04. build_depth: **comprehensive**. All versions/APIs verified against vendor docs (see citation URLs); star counts confirmed via GitHub repo pages on the research date.

---

## 3. Source Categories

### 3.1 Official / Vendor Documentation

Versions confirmed against the vendor's own reference on 2026-06-04.

| Component | Verified current version | Notes / signature confirmed | Source |
|---|---|---|---|
| **NestJS** | 11.1.x (v12 in roadmap, targeting Q3 2026 — ESM, Vitest, oxlint) | Stay on **v11 LTS** for this build; v12 is a breaking ESM/toolchain migration, avoid until GA. REST URL versioning via `app.enableVersioning({ type: VersioningType.URI })` → `/v1/`. | https://docs.nestjs.com/ , https://github.com/nestjs/nest/releases , https://www.infoq.com/news/2026/04/nestjs-12-roadmap-esm/ |
| **NestJS WebSockets** | `@nestjs/websockets` + `@nestjs/platform-socket.io` (v11 line) | Gateway via `@WebSocketGateway`, `@SubscribeMessage`; v12 adds disconnect-reason param. | https://docs.nestjs.com/websockets/gateways |
| **NestJS Throttler (rate limit)** | `@nestjs/throttler` | In-memory by default — **must** swap to Redis storage for multi-instance. `SlidingWindowCounter` algorithm recommended over default `FixedWindow`. | https://docs.nestjs.com/security/rate-limiting |
| **TypeORM** | **1.0.0** (released 2026-05-19; first major since 0.3 in 2021) | Compiled to ES2023, min Node 20. First stable major — validate migration from any 0.3 snippets; API largely compatible. | https://typeorm.io/ , https://github.com/typeorm/typeorm/releases |
| **BullMQ** | **5.78.x** | Use BullMQ, **not** legacy `bull` (maintenance-only). Repeatable jobs API deprecated since 5.16 in favor of **Job Schedulers** (`queue.upsertJobScheduler`). Separate `Queue`/`Worker`/`QueueEvents`. | https://docs.bullmq.io/ , https://docs.bullmq.io/guide/jobs/repeatable |
| **Next.js** | **16.2.6 LTS** (May 2026); v15 LTS EOS 2026-10-21 | Use 16.x LTS. App Router, stable Adapter API. | https://nextjs.org/blog/next-16 , https://endoflife.date/nextjs |
| **TanStack Query** | **v5** (requires React 18+) | `useQuery` single-object signature; `useSuspenseQuery`, `useMutationState` new. ~20% smaller than v4. | https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5 |
| **Tailwind CSS** | **v4** | `@theme` directive, OKLCH colors. `tailwindcss-animate` deprecated → `tw-animate-css`. | https://tailwindcss.com/ , https://ui.shadcn.com/docs/tailwind-v4 |
| **shadcn/ui** | Current (Tailwind v4 + React 19 supported) | `data-slot` attributes on primitives; CLI v4 initializes Tailwind v4. | https://ui.shadcn.com/docs/tailwind-v4 , https://ui.shadcn.com/docs/changelog |
| **PostgreSQL** | **17.10** (2026-05-14) | SQL/JSON `JSON_TABLE()`, improved VACUUM, failover for logical replication. Use 17.x. | https://www.postgresql.org/docs/17/release.html |
| **Redis** | **8.0** tri-licensed incl. **AGPLv3** (OSI-approved, May 2025) | Redis 8 OK for self-host under AGPLv3, but **Valkey 9.1** (BSD-3, Linux Foundation) is the cleaner license for a SaaS that may embed it. Redis 7.2 was last BSD before the fork. | https://redis.io/blog/agplv3/ , https://github.com/valkey-io/valkey/releases |
| **Sentry (NestJS)** | `@sentry/nestjs` **10.53.x** | Requires `instrument.ts` imported first; register `SentryGlobalFilter`. `@SentryCron()` wraps `@Cron` for check-ins. | https://docs.sentry.io/platforms/javascript/guides/nestjs/ , https://docs.nestjs.com/recipes/sentry |
| **HashiCorp Vault** | KV **v2** secrets engine | Path includes `/data/` (`secret/data/<path>`); configurable version retention (default 10); soft-delete + undelete. | https://developer.hashicorp.com/vault/api-docs/secret/kv/kv-v2 |
| **Resend** | `resend` Node SDK | `resend.emails.send({ from, to, subject, html | text })`; domain must be verified; React Email supported. | https://resend.com/docs/send-with-nodejs |
| **Slack incoming webhooks / Block Kit** | Current | POST JSON to webhook URL; supports Block Kit layout blocks (`section`, `header`, `divider`). | https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/ |
| **nmap** | 7.9x line | `-sV` version detection (`--version-intensity 0-9`, default 7); CPE output; `-oX` XML for parsing. | https://nmap.org/book/man-version-detection.html , https://nmap.org/book/man.html |
| **masscan** | 1.3.2 (last tagged 2021; AGPL-3.0) | `--rate` (default 100 pps, up to 25M w/ PF_RING); SYN async. Use for fast discovery, hand off to nmap for `-sV`. | https://github.com/robertdavidgraham/masscan/blob/master/doc/masscan.8.markdown |
| **crt.sh (CT search)** | Live service (Sectigo) | `https://crt.sh/?q=%.example.com&output=json`; indexes CT logs per RFC 6962. No SLA — treat as best-effort enrichment. | https://crt.sh/ |
| **OpenSSL / TLS cert inspection** | Node `tls` core module | `tls.connect()` + `socket.getPeerCertificate(true)` for cert chain, SAN, validity, fingerprint. | https://nodejs.org/api/tls.html |

### 3.2 GitHub Repositories

Stars verified on repo pages 2026-06-04.

| Repo | Stars | License | Activity / version | Use in Recon |
|---|---|---|---|---|
| [typeorm/typeorm](https://github.com/typeorm/typeorm) | ~36.5k | MIT | v1.0.0 (2026-05-19) | ORM (chosen). |
| [taskforcesh/bullmq](https://github.com/taskforcesh/bullmq) | ~6k+ | MIT | 5.78.x, very active | Job queue / scheduled scans. |
| [projectdiscovery/naabu](https://github.com/projectdiscovery/naabu) | ~6k | MIT | **v2.6.1** (2026-05-05), active | Fast Go SYN/CONNECT port scanner — strong candidate for discovery layer. |
| [projectdiscovery/httpx](https://github.com/projectdiscovery/httpx) | ~10k | MIT | active | HTTP probing / tech + TLS metadata at scale. |
| [robertdavidgraham/masscan](https://github.com/robertdavidgraham/masscan) | ~25.8k | **AGPL-3.0** | last tag 1.3.2 (2021); slow | Mass discovery — note AGPL: shell out as a separate process, do NOT link/embed. |
| [nmap/nmap](https://github.com/nmap/nmap) | ~10k+ | NPSL (custom, GPL-derived) | active | Deep `-sV`/NSE; license is restrictive — invoke as external binary only. |
| [maxmind/GeoIP2-node](https://github.com/maxmind/GeoIP2-node) | — | Apache-2.0 (API) | active | GeoIP enrichment (`has_geo`). DB under separate MaxMind EULA. |
| [resend/resend-node](https://github.com/resend/resend-node) | — | MIT | active | Email SDK. |
| [getsentry/sentry-javascript](https://github.com/getsentry/sentry-javascript) | ~8k+ | MIT | `@sentry/nestjs` 10.53.x | Error reporting. |

> Scanner-library strategy: prefer **naabu (MIT)** for the in-app discovery layer (clean license, Go binary, JSON output). Use **nmap** and **masscan** as optional external binaries behind a worker boundary — both carry copyleft/custom licenses that make embedding unwise. Node wrappers like `node-nmap`/`libnmap` exist but are thin and low-maintenance; prefer spawning the binary and parsing `-oX` XML directly.

### 3.3 Video / Tutorials

- BullMQ scheduled/repeatable jobs walkthrough — Better Stack: https://betterstack.com/community/guides/scaling-nodejs/bullmq-scheduled-tasks/
- NestJS REST API build (2026) — covers URI versioning, modules, guards: https://docs.nestjs.com/ (official "First steps" + recipes are the authoritative tutorial track).
- ProjectDiscovery Naabu overview docs (usage + flags): https://docs.projectdiscovery.io/tools/naabu/overview
- Nmap service/version detection chapter (canonical reference): https://nmap.org/book/vscan.html

### 3.4 Articles / Blog Posts

- Multi-tenant Postgres RLS, end-to-end (Nile): https://www.thenile.dev/blog/multi-tenant-rls
- AWS — multi-tenant data isolation with Postgres RLS: https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/
- Transactional outbox for reliable webhooks (Convoy): https://www.getconvoy.io/blog/webhooks-with-transactional-outbox
- microservices.io — Transactional Outbox pattern: https://microservices.io/patterns/data/transactional-outbox.html
- Censys "Ten Years Later" / Censys vs Shodan coverage & freshness data: https://censys.com/blog/evaluating-censys-performance/
- Monitoring certificate issuance via CT logs (germancoding): https://blog.germancoding.com/2020/03/31/monitoring-certificate-issuance-with-the-power-of-certificate-transparency/

### 3.5 Standards / RFCs

| Standard | Relevance to Recon | Source |
|---|---|---|
| **RFC 8446** — TLS 1.3 | TLS cert/handshake inspection; report protocol versions, flag legacy. | https://www.rfc-editor.org/rfc/rfc8446 |
| **RFC 6962** — Certificate Transparency (obsoleted by **RFC 9162**) | CT-log-driven subdomain/cert discovery via crt.sh; cite 9162 for current. | https://www.rfc-editor.org/rfc/rfc6962.html , https://www.rfc-editor.org/rfc/rfc9162 |
| **RFC 9110** — HTTP Semantics | REST API semantics, status codes, conditional requests. | https://www.rfc-editor.org/rfc/rfc9110 |
| **RFC 9457** — Problem Details for HTTP APIs (obsoletes **RFC 7807**) | Standard `application/problem+json` error envelope for `/v1/`. Use 9457. | https://www.rfc-editor.org/rfc/rfc9457 |
| **NIST SP 800-63B** — Digital Identity (auth) | Password/MFA/session guidance for SOC users. | https://pages.nist.gov/800-63-3/sp800-63b.html |
| **SOC 2 Trust Services Criteria** (AICPA 2017, 2022 PoF) | Security (CC1–CC9, required) + Availability + Confidentiality baseline. | https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022 |
| **OWASP ASVS 5.0.0** | Verification checklist incl. SSRF allow-list requirement. | https://owasp.org/www-project-application-security-verification-standard/ |
| **OWASP SSRF Prevention Cheat Sheet** | Core control for a scanner: allow-list targets, reject internal ranges. | https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html |

### 3.6 Competing Products

| Product | What it does well | Gaps Recon can exploit |
|---|---|---|
| **Shodan** | Best developer experience + libraries; **Shodan Monitor** watches registered IP ranges and alerts on new exposed services. | Scans only ~1,237 ports; ~68% of listed services live; slower detection (≈76h avg new-service detection). Recon offers **owned, authorized active scanning** + change-detection baselines rather than passive index lookups. |
| **Censys** | Scans all 65,535 ports, ~92% live services, strong TLS/cert analytics, fast detection (≈12h avg). | Heavy/enterprise; passive internet-wide index, not a tenant-scoped scheduler with custom alert routing. Recon's edge = scoped, scheduled, baseline-diff-driven alerting to Slack/webhook/email with audit trails. |
| **Microsoft Defender EASM** | Continuous discovery/mapping; per-asset/day billing; deep Azure/M365 integration. | Azure-centric; opaque per-asset pricing scales painfully; less flexible alert routing/export. Recon competes on predictable multi-tenant SaaS pricing + flexible exports (PDF/JSON/CSV) + open webhooks. |
| **Intruder.io** | 140k+ checks; clean UX; continuous monitoring. | Continuous ASM gated to top tiers; **misses assets not tied to your cloud providers**, and subsidiary/acquired-company assets. Recon's IP-range + domain + CT-log discovery widens coverage. |
| **Detectify** | Full EASM, research-built tests, 0-day coverage. | Vuln-test heavy, less focused on raw port/service/cert change-detection baselines and SOC alert-routing ergonomics. |

Sources: https://info.censys.com/censys-v-shodan-organic , https://www.intruder.io/blog/attack-surface-management-tools , https://blog.detectify.com/industry-insights/product-comparison-detectify-vs-intruder/ , https://learn.microsoft.com/en-us/azure/external-attack-surface-management/

### 3.7 Community Threads

- BullMQ repeatable→Job Scheduler migration discussion (docs changelog): https://docs.bullmq.io/changelog
- shadcn/ui Tailwind v4 upgrade discussion #2996: https://github.com/shadcn-ui/ui/discussions/2996
- naabu issues (real-world scan reliability / false-negative tuning): https://github.com/projectdiscovery/naabu/issues
- Redis vs Valkey license fork — practical implications: https://dev.to/synsun/redis-vs-valkey-in-2026-what-the-license-fork-actually-changed-1kni

### 3.8 APIs / Integrations

| API | Endpoint / signature | Source |
|---|---|---|
| **Slack incoming webhook** | `POST https://hooks.slack.com/services/...` JSON `{ blocks: [...] }` | https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/ |
| **Resend** | `POST https://api.resend.com/emails` / SDK `resend.emails.send(...)` | https://resend.com/docs/send-with-nodejs |
| **Sentry** | DSN ingest + `@sentry/nestjs` SDK; release/monitor (cron) APIs | https://docs.sentry.io/platforms/javascript/guides/nestjs/ |
| **Vault KV v2** | `GET/POST {addr}/v1/secret/data/<path>`, `X-Vault-Token` header | https://developer.hashicorp.com/vault/api-docs/secret/kv/kv-v2 |
| **crt.sh CT** | `GET https://crt.sh/?q=%25.<domain>&output=json` | https://crt.sh/ |
| **SSLMate CT Search** (alt, SLA'd) | `https://api.certspotter.com/v1/issuances` | https://sslmate.com/ct_search_api/ |
| **Censys (enrichment, optional)** | Platform API `https://api.platform.censys.io/v3/`; legacy `https://search.censys.io/api/` | https://search.censys.io/api , https://docs.censys.com/ |
| **Shodan (enrichment, optional)** | `https://api.shodan.io/` REST | https://developer.shodan.io/ |
| **MaxMind GeoIP2** (`has_geo`) | Web service or local mmdb reader; API Apache-2.0, DB under EULA | https://dev.maxmind.com/geoip/geolite2-free-geolocation-data/ |

### 3.9 Patterns

- **Multi-tenant isolation (Postgres/TypeORM):** Shared-DB + **Row-Level Security** with a `tenant_id` column and `current_setting('app.tenant_id')` policy is the right default for 50k+ scale (centralized enforcement, single migration set). Set the GUC per request/connection (transaction-scoped `SET LOCAL`) from tenant context; never trust app-layer `WHERE` alone. Schema-per-tenant rejected (migration/operational blow-up at scale). Sources: https://www.thenile.dev/blog/multi-tenant-rls , https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/
- **Outbox pattern (`has_dual_write`):** Write business change + outbox row in one ACID transaction; a relay (BullMQ poller or Postgres `LISTEN`/`NOTIFY`, or CDC/Debezium at higher scale) publishes webhooks/email. **At-least-once** delivery → consumers/webhooks must be idempotent (dedupe on event id). Covers Slack/webhook/email fan-out without lost-or-double dual writes. Sources: https://microservices.io/patterns/data/transactional-outbox.html , https://www.getconvoy.io/blog/webhooks-with-transactional-outbox
- **Baseline change-detection:** Persist normalized scan snapshots (sorted open ports / services / cert fingerprints + validity) per asset per run; diff current vs last accepted baseline; emit typed change events (`PORT_OPENED`, `SERVICE_CHANGED`, `CERT_ROTATED`, `CERT_EXPIRING`). Store hashes for cheap equality, full snapshot for trend reporting and 12-month retention.
- **Scheduled recurring scans:** BullMQ **Job Schedulers** (`upsertJobScheduler`, cron pattern) — idempotent per tenant+target; guard against overlapping runs with a per-target lock (Redis SETNX) so a slow scan never stacks.
- **WebSocket fan-out (`has_websocket`):** NestJS gateway + Redis adapter (`@socket.io/redis-adapter`) so realtime scan/alert events broadcast across all API instances; room-per-tenant for isolation.
- **Rate limiting:** `@nestjs/throttler` with **Redis storage** + `SlidingWindowCounter`; per-tenant and per-IP buckets; separate stricter bucket on scan-trigger endpoints.
- **Audit logging:** Append-only `audit_log` table (tenant_id, actor, action, target, before/after, ts), structured JSON logs (pino), immutable retention — maps to SOC 2 CC7/CC8.

---

## 4. Stack Candidates

| Layer | Chosen | Alternatives considered | Verdict / trade-off |
|---|---|---|---|
| API framework | **NestJS 11** (REST, URI `/v1/`) | Fastify-standalone; Express+tsoa | NestJS gives DI, guards, Throttler, WS, OpenAPI out of the box — sound. Pin v11; defer v12 ESM migration. **OK.** |
| ORM | **TypeORM 1.0** | Prisma; Drizzle | TypeORM 1.0 just GA'd — robust RLS-compatible raw query support and mature migrations. Prisma has weaker per-connection GUC story for RLS. **OK, with a note to validate 1.0 migration paths.** |
| Job queue | **BullMQ** (Redis) | Bull (legacy); pg-boss | BullMQ is the maintained successor; Job Schedulers fit recurring scans. **OK** — don't use legacy `bull`. |
| DB | **PostgreSQL 17** | — | Best-in-class RLS, JSONB for scan snapshots, `JSON_TABLE`. **OK.** |
| Cache/queue store | **Redis 8 (AGPLv3)** | **Valkey 9.1 (BSD-3)**; KeyDB | Functionally equivalent; **recommend Valkey** for license cleanliness in a SaaS, or managed ElastiCache. Flag, not blocker. |
| Frontend | **Next.js 16 LTS + shadcn/ui + Tailwind v4 + TanStack Query v5** | Remix; plain Vite SPA | Modern dominant stack; SSR + server actions help dashboards. **OK.** |
| Secrets | **Vault KV v2** | AWS Secrets Manager; Doppler | Vault fits SOC2 + self-host; rotation/versioning built in. **OK.** |
| Email | **Resend** | Postmark; SES | Resend DX excellent; verify deliverability/SLA for alert-critical email; SES as fallback. **OK.** |
| Error reporting | **Sentry** | Datadog; OpenTelemetry+Grafana | Sentry NestJS SDK first-class incl. cron check-ins. **OK.** |
| Discovery/scan engine | **naabu (MIT) in-app** + **nmap/masscan as external binaries** | pure-Node TCP scanner; ZMap | naabu = clean license + speed; nmap for deep `-sV`. **OK** — keep AGPL/custom-licensed binaries behind a process boundary. |

Overall: the customer's stack choices are **sound and current**. Only adjustments: pin NestJS v11 (not v12), and prefer Valkey over Redis 8 for licensing.

---

## 5. Risk Register + Threat Model

### Risk table

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **SSRF / scanning unauthorized or internal targets** (core abuse risk) | High | Critical | Mandatory **authorization-to-scan** proof (DNS TXT / file token / ACME-style domain control validation) before any scan; **deny-list RFC1918/loopback/link-local/metadata 169.254.169.254**; allow-list resolved IPs; re-resolve at scan time to prevent DNS rebinding. |
| Scan engine used as a DDoS/abuse amplifier | Medium | High | Global + per-tenant rate/pps caps; egress from isolated scanner subnet; legal ToS + per-target consent record; abuse monitoring. |
| Multi-tenant data leakage | Medium | Critical | Postgres RLS + per-request tenant GUC; tenant-scoped WS rooms; tests asserting cross-tenant 404; deny-by-default queries. |
| Webhook SSRF / internal pivot via alert URLs | High | High | Validate user webhook URLs against same deny-list; resolve+pin IP; block internal ranges; sign payloads (HMAC); no redirects. |
| Secret exposure (scan creds, API keys) | Medium | High | Vault KV v2, no secrets in DB/logs; short-lived tokens; log redaction; rotation. |
| At-least-once outbox → duplicate alerts | High | Low | Idempotency keys on events; dedupe window; webhook receivers idempotent. |
| BullMQ overlapping/slow scans stacking | Medium | Medium | Per-target Redis lock; job concurrency caps; timeouts + dead-letter. |
| Scan-result data sensitivity (it's a map of victim exposure) | Medium | High | Encrypt at rest; strict RBAC; 12-month retention then purge; export audit trail. |
| crt.sh / external API downtime | Medium | Medium | Fallback CT source (SSLMate Cert Spotter); cache; degrade gracefully. |
| Dependency/licensing (masscan AGPL, nmap NPSL) | Medium | Medium | Invoke as separate binaries (no linking); document license posture; prefer naabu MIT in-app. |
| Redis 8 AGPLv3 obligations in SaaS | Low | Medium | Use Valkey (BSD-3) or managed Redis to sidestep. |
| Next.js / NestJS CVEs | Medium | Medium | Pin LTS, Dependabot, Sentry alerts, monthly patch cadence. |

### Threat model (STRIDE) — security product specifics

**Trust boundaries:** (1) Internet ↔ API edge; (2) API ↔ scan workers (the workers hold the most dangerous capability — arbitrary outbound network access); (3) workers ↔ external scan targets; (4) app ↔ Postgres (RLS boundary between tenants); (5) app ↔ Vault; (6) app ↔ third-party sinks (Slack/Resend/webhooks). The **scan worker is the crown-jewel boundary**: it is, by design, an authorized SSRF engine, so it must be the most locked-down and network-segmented component.

- **Spoofing:** Strong auth (NIST 800-63B; MFA for SOC roles), signed webhooks (HMAC), Vault-issued service identities. Verify target ownership before scanning (anti-spoofed-authorization).
- **Tampering:** RLS + parameterized queries; append-only audit log; integrity hashes on scan snapshots; TLS everywhere.
- **Repudiation:** Immutable audit logging of who triggered which scan against which target, plus authorization-proof records (defends "we never authorized that scan").
- **Information disclosure:** This is the #1 sensitivity — results are a blueprint of a customer's exposure. Encrypt at rest, per-tenant RLS, least-privilege RBAC, redact secrets in logs, scoped exports. Multi-tenant leakage = critical.
- **Denial of service:** Rate limits, pps caps, scanner subnet isolation, job locks/timeouts; protect both Recon itself and (ethically/legally) third-party targets from being hammered.
- **Elevation of privilege:** Scanner must NOT reach internal infra — egress-filtered subnet, metadata endpoint blocked (IMDSv2/blocked 169.254.169.254), deny-list internal CIDRs, no SSRF pivot from webhook/scan target input. RLS prevents tenant→tenant escalation.

**Primary product-specific danger:** an attack-surface scanner is a dual-use weapon. The combination of (a) authorization-to-scan enforcement, (b) target allow-list / internal-range deny-list, (c) webhook URL SSRF validation, and (d) an isolated, egress-filtered scanner network is the non-negotiable control set. All four must ship in v1.

---

## 6. Research Gaps

1. **TypeORM 1.0 + RLS at scale** — confirm `SET LOCAL app.tenant_id` works cleanly with TypeORM 1.0's connection/transaction handling and the pool doesn't leak GUC between tenants.
2. **Authorization-to-scan UX** — decide on validation method(s): DNS TXT vs HTTP file token vs ACME-style; how to handle IP ranges (RIR/WHOIS ownership? signed legal attestation?).
3. **Scanner deployment topology** — naabu/nmap/masscan in worker containers vs a dedicated scanner fleet; egress IP allocation, PF_RING need, and how to prove scan-source IP to customers.
4. **CT log freshness/SLA** — crt.sh has no SLA; validate SSLMate Cert Spotter as primary vs fallback and its quota/pricing.
5. **Redis vs Valkey decision** — lock licensing call (recommend Valkey/managed) before infra build.
6. **Email deliverability for critical alerts** — Resend SLA + bounce handling; SES fallback path.
7. **12-month retention storage sizing** — snapshot volume at 50k+ assets; partitioning/compression strategy for trend reporting and PDF/CSV exports.
8. **Rate-of-scan legal/abuse policy** — ToS, per-target consent records, abuse-response runbook.
9. **WebSocket scale** — confirm Redis adapter fan-out and room-per-tenant under 50k+ concurrent dashboards.
10. **PDF export pipeline** — choose engine (e.g., Playwright/Chromium render vs server-side lib) for audit reports.

---

## 7. Summary + GO / NO-GO

The chosen stack is **current, well-supported, and appropriate** for a multi-tenant, 50k+-asset external attack-surface monitoring SaaS. Every major dependency was verified against vendor docs on 2026-06-04: NestJS 11 (pin, avoid v12 ESM churn), TypeORM 1.0, BullMQ 5.78 with Job Schedulers, Next.js 16 LTS, Tailwind v4 + shadcn/ui, TanStack Query v5, PostgreSQL 17, Vault KV v2, Sentry `@sentry/nestjs` 10.53, Resend, Slack Block Kit. Proven patterns exist for every domain signal: RLS for multi-tenant isolation, the outbox pattern for `has_dual_write`/webhooks/email, BullMQ Job Schedulers for recurring scans, snapshot-diff for baseline change detection, Redis-adapter WS fan-out for `has_websocket`, and MaxMind for `has_geo`. naabu (MIT) gives a clean in-app scan engine, with nmap/masscan available as external binaries behind a worker boundary.

The defining risk is inherent to the product class: an attack-surface scanner is dual-use and must enforce **authorization-to-scan**, **internal-range/SSRF deny-listing** (including webhook URLs), **scanner network egress isolation**, and **strict multi-tenant + scan-result data protection**. These are well-understood, ship-in-v1 controls, not unknowns. Two minor adjustments: pin NestJS v11 and prefer Valkey (BSD-3) over Redis 8 (AGPLv3). Open questions in §6 are build-time validations, not architecture blockers.

**Go/No-Go: GO** — stack verified, current, and de-risked; competitive gap (scoped, scheduled, baseline-diff alerting with open routing/exports) is real; the dominant threat-model risks have standard, known mitigations that must be enforced as hard invariants in Stage 1.

