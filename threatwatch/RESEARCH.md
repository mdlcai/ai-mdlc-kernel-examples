# RESEARCH.md — ThreatWatch

build_depth: standard
review_gates: auto
force_research: false
domain: "Cybersecurity"

## Domain Signals

```yaml
domain_signals: ["has_scheduled_work", "has_webhooks", "has_payments", "has_dual_write", "has_webhook_send", "has_email", "has_websocket", "has_geo"]
```

## Product Vision
**Problem:** Enterprise security teams receive enormous quantities of security telemetry from disconnected systems:

Firewalls
EDR/XDR
SIEM platforms
Vulnerability scanners
IDS/IPS
Web application firewalls
Cloud platforms
Identity systems
DNS systems
Network sensors
Authentication systems
Server and application logs
Threat intelligence feeds

The data is often fragmented across products, consoles, formats, and vendors.

Security analysts must manually correlate:

source → event → asset → IP → country → attack technique → severity → affected system → response

Threat Watch provides a unified security intelligence layer across those sources.

**Who it affects:** Primary Users

Security Operations Analysts

Deal with large volumes of security alerts and events.
Spend significant time switching between security platforms.
Need to quickly identify real threats and investigate them.

Security Engineers

Manage security infrastructure, integrations, detection rules, and telemetry.
Need visibility across multiple security technologies.

Security Managers / Directors

Need a unified view of organizational threat activity.
Need to understand risk, trends, critical incidents, and security operations performance.

Incident Responders

Need to rapidly connect alerts, threat sources, assets, and historical events during an investigation.
Secondary Users

IT and Network Operations

Need visibility into suspicious network and infrastructure activity.

Cloud and Infrastructure Teams

Need to understand threats affecting cloud workloads, servers, applications, and services.

Compliance and Risk Teams

Need evidence of monitoring, detection, investigation, and security controls.

Executives

Need a simple view of the organization's current threat landscape without navigating technical security platforms.
Ultimately Affected

Enterprise organizations are affected when fragmented security data causes threats to be missed, investigations to take longer, or security teams to spend excessive time correlating information manually.

Threat Watch gives all of these users a shared security picture, with the level of detail appropriate to their role.

**Why existing solutions fall short:** Security data is fragmented across SIEM, EDR/XDR, firewalls, cloud platforms, identity systems, vulnerability scanners, and network tools.
Too much telemetry, too little context. Analysts receive enormous event volumes but must still determine which events are related and meaningful.
Alert fatigue makes it difficult to distinguish genuine threats from noise and false positives.
Investigation is manual. Analysts often have to pivot between multiple consoles to connect a source, target, asset, alert, and historical activity.
Tools are optimized around their own ecosystems. Each security product has its own data model, detection logic, interface, and terminology.
Threat intelligence is disconnected from operational telemetry. Knowing that an IP or domain is malicious is less useful unless you can immediately determine whether your environment interacted with it.
Existing dashboards often show events rather than relationships. They tell you what happened, but don't always make it obvious how individual events connect into an attack.
Integration creates additional complexity. More security products can mean more connectors, data pipelines, licensing, administration, and consoles.
Scaling security operations is expensive. Increasing telemetry often increases storage, processing, licensing, and analyst workload simultaneously.
The analyst remains the correlation engine. Even when the data exists, humans frequently have to reconstruct the larger picture manually.
The Gap Threat Watch Addresses

Existing solutions collect and detect. Threat Watch focuses on connecting the data.

Threat Watch is designed to turn:

Telemetry → Context → Correlation → Detection → Investigation

into a single security operating picture.

The goal is not to replace the enterprise's existing security stack. The goal is to make the existing stack more useful by giving security teams a unified view of what is happening, what is connected, and what matters.

**Solution:** What we're building first

The core product is the dashboard shown in your image:

Live threat overview
Network/attack map
Real-time alert feed
Threat-source intelligence
Targeted assets
Geographic attack activity
Detection rules and activity
Event/ingestion health
Search and investigation
Asset and threat-source profiles

Then underneath that is the actual security platform:

Enterprise Security Logs
        ↓
   Ingestion Layer
        ↓
  Event Normalization
        ↓
    Enrichment
        ↓
   Correlation Engine
        ↓
   Detection Engine
        ↓
      Alerts
        ↓
 Investigation / Threat Intelligence
        ↓
    Threat Watch Dashboard

And MDLC is governing the entire build: architecture, security, authorization, ingestion integrity, detection behavior, testing, reviewer gates, and production readiness.

So this isn't just a dashboard mockup. We're building the underlying security telemetry and threat-analysis platform that powers the dashboard.


## Users & Outcomes
**Key Workflows:**
1. Security Telemetry Ingestion

Enterprise Source → Threat Watch

Connect security data source
Authenticate and validate connection
Receive security events
Validate incoming data
Normalize into the Threat Watch event model
Enrich events with context
Store events
Report ingestion health and errors
2. Threat Detection

Event → Detection → Alert

Receive normalized event
Evaluate detection rules
Correlate with related activity
Enrich with threat intelligence
Determine severity and confidence
Generate alert when criteria are met
Notify appropriate users
3. Real-Time Monitoring

Events → Security Operations Dashboard

Display current threat activity
Update alert counts
Update threat-source activity
Update targeted assets
Update geographic activity
Update detection activity
Display ingestion health
Allow analysts to filter the entire view
4. Alert Investigation

Alert → Investigation

Analyst opens alert
Review detection and evidence
Identify threat source
Identify targeted asset
Review related events
Review threat intelligence
Examine timeline
Identify related alerts
Assign or escalate investigation
Resolve or close alert
5. Threat-Source Investigation

IP / Domain / Indicator → Threat Profile

Search indicator
Retrieve threat intelligence
Identify country/ASN/organization
Find historical observations
Identify targeted assets
Identify triggered detections
Identify related alerts
Pivot into associated events
6. Asset Investigation

Asset → Security Profile

Search or select asset
View asset identity and metadata
Review recent activity
Identify threat sources
Review alerts
Review detection history
Review exposure
Review historical activity
7. Threat Correlation

Individual Events → Connected Activity

Identify common source
Identify common target
Identify shared indicators
Identify temporal relationships
Identify matching detection rules
Group related events
Create a connected threat picture
8. Detection Management

Security Rule → Detection

Create detection rule
Define conditions
Define severity
Define required evidence
Test rule
Enable/disable rule
Monitor rule performance
Measure alerts and false positives
Version detection changes
9. Integration Management

Security Product → Threat Watch

Add integration
Configure credentials
Validate connection
Select telemetry
Begin ingestion
Monitor event flow
Detect ingestion failures
Retry/recover failed ingestion
Disable or remove integration
10. Investigation Search

Analyst Question → Evidence

Analyst searches for:

IP
Domain
Hostname
User
Asset
Alert
Event
Hash
URL
Detection rule

Search results should allow immediate pivots into related entities.

11. Notification Workflow

Detection → Notification

Detection generates alert
Evaluate notification policy
Determine severity/recipient
Send notification
Record delivery status
Escalate when required
12. Reporting

Security Activity → Security Report

Aggregate events and alerts
Identify trends
Identify top threat sources
Identify targeted assets
Identify critical activity
Generate report
Deliver report to authorized recipients
Core Threat Watch Workflow

The most important workflow should be:

INGEST → NORMALIZE → ENRICH → CORRELATE → DETECT → ALERT → INVESTIGATE → RESPOND

And the core analyst experience should be:

Dashboard → Alert → Source → Target → Related Events → Threat Intelligence → Timeline → Resolution

That should be the backbone of the MDLC FLOW.md rather than treating the dashboard as the product itself.

**Success Metrics:**
Security Detection
≥99.9% ingestion availability
<30 seconds from event ingestion to detection for real-time sources
100% of generated alerts traceable to supporting events and detection logic
0 confirmed cross-tenant data exposures
Track and continuously reduce false-positive rates
Detect and correlate related events into meaningful attack activity
Analyst Efficiency
Reduce investigation time compared with manually pivoting between security products
Analyst can move from alert → source → target → related events without leaving Threat Watch
<10 seconds to load a typical investigation
Track MTTA and MTTR
Reduce the number of security consoles required for routine investigations
Visibility
Real-time visibility into:
Threat sources
Targeted assets
Active alerts
Detection activity
Geographic activity
Event volume
100% of supported integrations expose ingestion-health status
Every supported indicator provides historical activity and related-event context
Platform Performance
Sustain the defined target events-per-second workload
<5 seconds for typical dashboard queries
Gracefully handle duplicate, malformed, delayed, and out-of-order events
No silent event loss
Horizontal scaling without architectural redesign
Product Adoption
Organizations successfully onboard their first security integration
Time from account creation to first ingested event
Number of active integrations per organization
Daily/weekly active security users
Alerts investigated per analyst
Organizations returning to Threat Watch for daily monitoring
Business Outcome

The ultimate success metric is:

Threat Watch reduces the time and effort required for a security team to understand, investigate, and act on threats across its existing security infrastructure.

Or, more simply:

More security visibility → less manual correlation → faster investigation → faster response.


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
state_management: "React Context"

# Backend
backend_framework: "Express"
orm_preference: "Prisma"
realtime_needed: true
background_jobs: "Temporal"

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
Threat Watch speaks with the authority of a security operations platform: direct, technical, confident, and focused on action.

Core characteristics
Authoritative — sound like the system that knows what is happening.
Technical — use precise security terminology, not watered-down marketing language.
Direct — short sentences. Clear statements. No fluff.
Calm — communicate serious threats without unnecessary alarmism.
Operational — focus on what happened, why it matters, and what to investigate.
Intelligent — emphasize correlation, context, relationships, and signal over raw volume.
Modern — sophisticated and cyber-native without sounding gimmicky.
Trustworthy — never exaggerate detections or imply certainty where there is only suspicion.
Voice formula

Observe. Correlate. Detect. Investigate.

Threat Watch should make complex security activity feel visible, connected, and actionable.

Language to favor

Use:

Detect
Correlate
Investigate
Observe
Identify
Analyze
Enrich
Trace
Monitor
Evidence
Activity
Threat
Signal
Context
Timeline
Source
Target
Indicator
Detection

Avoid:

Revolutionary
Game-changing
Next-generation
Unprecedented
Military-grade
AI-powered everything
Stop hackers
100% secure
Guaranteed protection
Example voice

Weak:

Our powerful AI platform helps security teams stay ahead of today's rapidly evolving cyber threats.

Threat Watch:

See the attack before it becomes an incident.

Weak:

Threat Watch provides comprehensive visibility across your entire security environment.

Threat Watch:

One view of what is attacking, what is being targeted, and what changed.

Brand personality

If Threat Watch were a person:

The senior SOC analyst who already connected the dots.

Not loud. Not theatrical. Precise, observant, technically credible, and always focused on the evidence.

### Art Direction
Threat Watch is a cyber-operations command center, not a conventional SaaS dashboard.

Visual Identity
Dark-first — near-black/void backgrounds with subtle depth.
High information density — prioritize actionable information over whitespace.
Technical precision — grids, coordinates, timestamps, event IDs, telemetry, and structured data reinforce the operational feel.
Real-time energy — restrained motion, live feeds, pulsing nodes, connection paths, and state changes.
Signal-driven color — color communicates severity and state, never decoration.
Layered intelligence — events become relationships, relationships become detections, detections become investigations.
Minimal chrome — thin borders, compact controls, restrained shadows, almost no unnecessary UI decoration.
Color System
Purpose	Direction
Background	Void black / deep navy
Primary interface	Dark blue-black
Network / telemetry	Cyan
Informational	Blue
Low severity	Green
Medium severity	Yellow
High severity	Orange
Critical	Red / hot pink
Disabled / inactive	Muted gray
Typography

Primary: Technical monospace / modern grotesk combination.

Monospace for telemetry, IPs, timestamps, IDs, logs, metrics, and technical values.
Clean sans-serif for navigation, descriptions, and user-facing actions.
Strong hierarchy with compact headings.
Uppercase labels for operational states and categories.
Interface Principles

1. See first.
The dashboard immediately communicates what is happening.

2. Follow the signal.
Every alert, source, target, and indicator should lead naturally to deeper context.

3. Relationships over isolated events.
Visualize connections between attacker → infrastructure → target → detection → evidence.

4. Evidence over assumptions.
Show why something was detected.

5. Density without chaos.
The interface can be information-rich, but hierarchy must remain obvious.

6. Motion has meaning.
Animations represent live activity, state changes, or relationships. Never animate simply for spectacle.

Signature Visual Motifs
Global threat map
Connected-node network graphs
Live event streams
Pulsing threat indicators
Attack paths
Timeline visualization
Severity signals
Technical grids
Telemetry counters
Detection/rule activity
Source → target relationships
Overall Feel

Dark. Precise. Live. Intelligent. Operational.

The visual language should feel like mission control for enterprise security: information-dense enough for an analyst, polished enough for an executive, and unmistakably built for cybersecurity.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #d0754e | Buttons, links, active states |
| Secondary | #a6ca55 | Accents, badges, highlights |
| Accent | #70afcb | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f5f5f4 | Cards, elevated containers |
| Text | #1d1816 | Headings, body text |
| Text Secondary | #70625c | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #d0754e | Buttons, links, active states |
| Secondary | #a4c850 | Accents, badges, highlights |
| Accent | #6aacc8 | Callouts, hover states |
| Background | #0c0a09 | Page background |
| Surface | #171312 | Cards, elevated containers |
| Text | #eceaea | Headings, body text |
| Text Secondary | #958983 | Captions, muted text |
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
- Variant: Rounded
- Border radius: 8px (sm: 4px, lg: 12px, xl: 16px)
- Shadows: Subtle — `0 1px 3px rgba(0,0,0,0.1)`
- Theme: Light + Dark — ship both palettes with a runtime theme toggle that follows the user's system preference (`prefers-color-scheme`) and persists their explicit choice

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#d0754e',
    secondary: '#a6ca55',
    accent: '#70afcb',
    background: '#fafafa',
    surface: '#f5f5f4',
    foreground: '#1d1816',
    muted: '#70625c',
    success: '#1fad53',
    warning: '#ec9c13',
    destructive: '#df2020',
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
  --color-primary: #d0754e;
  --color-secondary: #a6ca55;
  --color-accent: #70afcb;
  --color-background: #fafafa;
  --color-surface: #f5f5f4;
  --color-text: #1d1816;
  --color-text-secondary: #70625c;
  --color-success: #1fad53;
  --color-warning: #ec9c13;
  --color-error: #df2020;
  --font-heading: 'Space Grotesk', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 8px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #d0754e;
  --color-secondary: #a4c850;
  --color-accent: #6aacc8;
  --color-background: #0c0a09;
  --color-surface: #171312;
  --color-text: #eceaea;
  --color-text-secondary: #958983;
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

### If the template carries no `:root` block
- Some Claude Design exports style every element inline instead of declaring
  CSS custom properties. The template is still binding: derive the token set
  FROM the markup — background/surface/text/accent colours from the inline
  `style` attributes (ranked by frequency and role), the `font-family` stack
  from its `@font-face` + inline declarations, and radius/spacing/shadow from
  the values the template actually uses. Write that derived set as the
  project `:root` block, use it exactly as you would a copied one, and record
  it in `DECISIONS.md` with a note that it was derived rather than copied.
- `{{placeholder}}` tokens, `<sc-for>` loops and `<script type="text/x-dc"
  data-props="…">` tags are Claude Design component scaffolding. Treat each
  `data-props` default as the chosen value, treat `<sc-for>` bodies as the
  item template for a data-bound list, and never render a placeholder
  literally.
- If the file begins with a loader shell (`<noscript>This page requires
  JavaScript</noscript>` plus `<script type="__bundler/template">`), the real
  page is the JSON string inside that script and the assets are the base64
  entries in `__bundler/manifest`; decode those before reading tokens.

---

# Research Sections (Stage 0 — researched 2026-10-04, build_depth: standard)

## 3. Source Categories

### 3.1 — Official Documentation & Vendor Sources

| Source | URL | What It Covers | Notes |
|--------|-----|----------------|-------|
| Next.js 16 release | https://nextjs.org/blog/next-16 | App Router, React 19.2, `proxy.ts` replaces `middleware.ts` | npm `next@16.3.8` (Active LTS, verified with `npm view` 2026-10-04) |
| Next.js Proxy docs | https://nextjs.org/docs/app/getting-started/proxy | Request interception (CSP nonce, auth redirect) | Proxy runs on the Node runtime in v16; `middleware.ts` is deprecated but still works |
| Next.js July 2026 security release | https://nextjs.org/blog/july-2026-security-release | 13 advisories incl. proxy bypass, SSRF, cache poisoning | Pin a patched 16.3.x or later; never rely on proxy alone for authz |
| Express 5.1 LTS announcement | https://expressjs.com/en/blog/2025-03-31-v5-1-latest-release/ | Express 5 is the npm default; LTS timeline; async error propagation | npm `express@5.2.1` |
| Prisma ORM 7 announcement | https://www.prisma.io/blog/announcing-prisma-orm-7-0-0 | Rust-free client, driver adapters (`@prisma/adapter-pg`), `prisma.config.ts` | `prisma@latest` = 8.0.0-rc.19 (RC) → pin **7.10.0** (stable `prev`) |
| Prisma upgrade to v7 | https://www.prisma.io/docs/guides/upgrade-prisma-orm/v7 | Required generator `output`, adapter wiring, ESM | |
| Prisma PostgreSQL quickstart | https://www.prisma.io/docs/prisma-orm/quickstart/postgresql | Schema, migrate dev/deploy | |
| Prisma release status | https://www.prisma.io/docs/orm/release-status | GA vs preview features | Avoid preview features |
| Temporal TS SDK — Schedules | https://docs.temporal.io/develop/typescript/workflows/schedules | Create/trigger/pause schedules via ScheduleClient | `@temporalio/*@1.24.0` |
| Temporal TS SDK — Run a Worker | https://docs.temporal.io/develop/typescript/workers/run-worker-process | `Worker.create({connection, taskQueue, workflowsPath, activities})` | The SDK bundles workflow code with its own webpack |
| Temporal TS SDK — Activity execution | https://docs.temporal.io/develop/typescript/activities/execution | Timeouts, retry policies | |
| Temporal self-hosted deployment | https://docs.temporal.io/self-hosted-guide/deployment | Server images, persistence | `temporalio/auto-setup` is deprecated → use `temporalio/server` or the CLI dev server |
| Temporal TS API reference | https://typescript.temporal.io/api/namespaces/workflow | Workflow APIs (sleep, proxyActivities, signals) | |
| Caddy docs | https://caddyserver.com/docs/ | `tls internal`, `reverse_proxy` (proxies WebSocket upgrades natively) | Local CA; no header-size tuning needed |
| OWASP WebSocket Security Cheat Sheet | https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html | Origin allow-list on handshake, auth, message limits | The ws project recommends authenticating in the HTTP `upgrade` handler |
| OCSF schema browser | https://schema.ocsf.io/ | Categories, event classes, attribute dictionary | Basis of the normalized event model |
| OCSF Detection Finding (1.4.0) | https://schema.ocsf.io/1.4.0/classes/detection_finding | Alert/finding fields (severity_id, confidence, evidences) | The Alert model mirrors it |
| Microsoft Sentinel entity mapping | https://learn.microsoft.com/en-us/azure/sentinel/map-data-fields-to-entities | Entities (Account, Host, IP, URL, FileHash…) and alert grouping by shared entity | Source of the correlation-by-shared-entity pattern |
| Microsoft Sentinel entity types | https://learn.microsoft.com/en-us/azure/sentinel/entities-reference | Entity identifiers | Indicator types for search |
| DB-IP IP-to-City Lite | https://db-ip.com/db/download/ip-to-city-lite | Free monthly geo DB, CC BY 4.0, CSV/MMDB | Attribution required; MMDB is 121 MB (Oct 2026) |
| ThreatFox API | https://threatfox.abuse.ch/api/ | IOC lookup; a free Auth-Key is now required | IOCs expire after 6 months |
| Sigma rules specification | https://sigmahq.io/sigma-specification/specification/sigma-rules-specification.html | Rule structure: logsource, detection selections/condition, level | Inspiration for the rule-condition model |

**Key Takeaways:**
- Next.js 16 renames middleware to `proxy.ts`, and the CSP nonce (Profile N) is set there. Authorization stays in the Express API, because proxy-bypass advisories exist.
- Prisma 7 requires a driver adapter and an explicit generator output. Prisma 8 is still an RC, so pin 7.10.0.
- `temporalio/auto-setup` is deprecated and archived. At small scale, the supported single-container option is the Temporal CLI dev server (`temporalio/temporal server start-dev --db-filename …`) with a persisted volume. Production would move to `temporalio/server` + Postgres, with admin-tools schema jobs.
- Caddy proxies WebSocket upgrades with no extra config, and `tls internal` gives HTTPS-only local deploys.
- TypeScript `latest` is 7.0.2 (native compiler), but the typescript-eslint and Next toolchains are validated against 5.x, so pin **typescript@5.9.3**.

### 3.2 — GitHub & Open Source

| Repository | URL | Stars | Last Active | Relevance |
|------------|-----|-------|-------------|-----------|
| wazuh/wazuh | https://github.com/wazuh/wazuh | ~15.9k | active 2026 (GPLv2) | Reference architecture: agent → manager → indexer → dashboard. Reference only (GPL) |
| SigmaHQ/sigma-specification | https://github.com/SigmaHQ/sigma-specification | — | active 2026 | Rule schema. Reference only |
| elastic/detection-rules | https://github.com/elastic/detection-rules | — | active 2026 (Elastic License v2) | Rule metadata (risk score, severity, MITRE mapping, false positives). Reference only |
| ocsf/ocsf-schema | https://github.com/ocsf/ocsf-schema | — | active 2026 (Apache-2.0) | Normalized event model. Reference for field names |
| mitre-attack/attack-stix-data | https://github.com/mitre-attack/attack-stix-data | — | ATT&CK v19.1, 12 May 2026 | Technique IDs for detection tagging (seed a small subset) |
| temporalio/samples-typescript | https://github.com/temporalio/samples-typescript | — | active 2026 (MIT) | Schedule, worker and activity patterns |
| temporalio/samples-server (compose) | https://github.com/temporalio/samples-server/tree/main/compose | — | active 2026 | Supported compose files (postgres, dev) since the docker-compose repo was archived in Jan 2026 |
| temporalio/docker-builds | https://github.com/temporalio/docker-builds | — | archived Sep 2026 | Confirms the auto-setup deprecation |
| sapics/ip-location-db | https://github.com/sapics/ip-location-db | — | active 2026 | DB-IP lite redistributions (CSV/MMDB) |
| standard-webhooks/standard-webhooks | https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md | — | active (MIT) | HMAC signature scheme for inbound and outbound webhooks |

**Evaluation Checklist:**
- [x] License compatible? Wazuh (GPL) and Elastic (ELv2) are used as reference only. Apache, MIT and CC-BY sources are usable.
- [x] Actively maintained? Yes, except temporalio/docker-builds, which is archived and cited only as deprecation evidence.
- [x] Good documentation / examples? Yes.
- [x] Community size healthy? Yes.
- [x] Dependency vs reference? The dependencies are npm packages: next, express, prisma, @temporalio/*, ws, zod, pino, helmet, express-rate-limit, argon2. Every repo above is reference only.

**Key Takeaways:**
- Correlation-first products group alerts by shared entities (IP, host, user, hash) within a time window. ThreatWatch adopts the same pattern.
- A Sigma-like rule model fits: field selections with operators, a condition, a level and MITRE tags, all expressible as JSON in Postgres.
- Geo enrichment uses an offline DB-IP-derived table, which avoids both a paid map/geo API and a per-request external call.

### 3.3 — Video & Tutorial Sources

| Title | URL | Creator | Length | Why It's Useful |
|-------|-----|---------|--------|-----------------|
| Never Fear Complex Tasks Again — Temporal Workflow Engine for Developers | https://www.youtube.com/watch?v=HyP-ZvnPvIo | YouTube (community) | ~1h | Workflow/activity fundamentals |
| Run your first Temporal application (TypeScript) | https://learn.temporal.io/getting_started/typescript/first_program_in_typescript/ | Temporal | course | Worker/task-queue wiring, failure recovery |
| Temporal TypeScript tutorials index | https://learn.temporal.io/tutorials/typescript/ | Temporal | courses | Recurring/scheduled job tutorials |
| Introduction to Detection Engineering with Sigma | https://isaacdunham.github.io/posts/intro-detection-engineering-sigma/ | Isaac Dunham | tutorial | Selections, filters, conditions |

**Key Takeaways:**
- Temporal workflow code must be deterministic: all I/O (DB, HTTP, email) goes in activities, and time comes from the workflow clock. This matches the has_scheduled_work "injected clock" rule.

### 3.4 — Articles, Blogs & Written Tutorials

| Title | URL | Author/Site | Date | Relevance |
|-------|-----|-------------|------|-----------|
| What the 2025 SANS Detection & Response Survey Reveals | https://www.stamus-networks.com/blog/what-the-2025-sans-detection-response-survey-reveals-false-positives-alert-fatigue-are-worsening | Stamus Networks | 2025 | 73% name false positives as their top detection challenge, so the build tracks false positives per rule |
| Alert Fatigue in SOCs: Research Challenges and Opportunities | https://dl.acm.org/doi/10.1145/3723158 | ACM Computing Surveys | 2025 | Academic framing of alert fatigue and correlation |
| Can Risk-Based Alerting Mitigate Cybersecurity Alert Fatigue? | https://arxiv.org/pdf/2609.02465 | arXiv | Sep 2026 | Severity × confidence scoring |
| Measuring the Detection Efficacy of Security Logging Standards | https://arxiv.org/pdf/2605.05531 | arXiv | May 2026 | Effect of OCSF normalization on detection |
| How to Run Temporal in Docker | https://oneuptime.com/blog/post/2026-02-08-how-to-run-temporal-in-docker-for-workflow-orchestration/view | OneUptime | Feb 2026 | Compose patterns after the auto-setup deprecation |
| Self-Host Temporal in 2026 | https://automationatlas.io/guides/tutorial-temporal-self-hosted-deploy-2026/ | Automation Atlas | 2026 | temporalio/server + postgres compose |
| Next.js proxy.ts explained | https://dev.to/parsajiravand/nextjs-proxyts-explained-with-cheat-sheet-j4i | DEV | 2026 | Notes on migrating middleware → proxy |
| Webhook Security Best Practices: HMAC, Replay Attacks | https://dev.to/instawebhook/webhook-security-best-practices-hmac-replay-attacks-encryption-2de1 | DEV | 2026 | Timestamp tolerance, constant-time compare |

**Key Takeaways:**
- False-positive rate per rule is a first-class metric (SANS 2025; Omdia reports ~46% of alerts are false positives). Alerts get a `false_positive` resolution, and each rule shows its FP rate.
- Risk-based alerting supports scoring by severity × confidence instead of raw count.

### 3.5 — Standards, RFCs & Specifications

| Standard | Reference | URL | Applicability |
|----------|-----------|-----|---------------|
| OCSF 1.4 | Detection Finding / Network Activity / Authentication classes | https://schema.ocsf.io/ | Normalized event fields (class, severity_id, src/dst endpoint, actor) |
| Sigma | Sigma Rules Specification | https://github.com/SigmaHQ/sigma-specification/blob/main/specification/sigma-rules-specification.md | Rule logic vocabulary (selection, condition, level) |
| MITRE ATT&CK | Enterprise v19.1 | https://github.com/mitre-attack/attack-stix-data | Technique tags on rules and alerts |
| Standard Webhooks | spec v1 | https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md | Inbound ingest and outbound notification signatures (`webhook-id`, `webhook-timestamp`, `webhook-signature: v1,<base64>`; 300 s tolerance) |
| RFC 9457 (obsoletes 7807) | Problem Details for HTTP APIs | https://www.rfc-editor.org/rfc/rfc9457 | Error envelope |
| RFC 6455 | WebSocket protocol | https://www.rfc-editor.org/rfc/rfc6455 | Origin header semantics on the handshake |
| OWASP Top 10 | 2021 | https://owasp.org/www-project-top-ten/ | Security floor (security_baseline names no regime) |
| OWASP WSTG — Testing WebSockets | WSTG latest | https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/11-Client-side_Testing/10-Testing_WebSockets | WebSocket test cases (CSWSH) |
| WCAG 2.2 AA | W3C | https://www.w3.org/TR/WCAG22/ | UI accessibility floor |

**Key Takeaways:**
- `security_baseline` is blank, so OWASP Top 10 is the floor and no regulatory regime checklist applies. Password policy still follows NIST 800-63B length rules as best practice (ADR).
- One HMAC scheme (Standard Webhooks) covers both inbound ingestion and outbound notification webhooks.

### 3.6 — Competing / Adjacent Products

| Product | URL | Pricing | Strengths | Weaknesses | Our Differentiation |
|---------|-----|---------|-----------|------------|---------------------|
| Microsoft Sentinel | https://learn.microsoft.com/en-us/azure/sentinel/ | per-GB ingest | Entity mapping, incident grouping, huge connector set | Azure-centric, KQL learning curve, cost scales with telemetry | Vendor-neutral and relationship-first: source → target → detection pivots on one screen |
| Splunk Enterprise Security | https://www.splunk.com/en_us/products/enterprise-security.html | per-GB / workload | Mature correlation searches, risk-based alerting | Expensive, heavy to operate | A lightweight overlay on the existing stack, not a replacement SIEM |
| Elastic Security | https://www.elastic.co/security | tiered / self-host | Open rules repo, strong search | Operational burden of running a cluster | An opinionated SOC picture (map, live feed, profiles) out of the box |
| Wazuh | https://github.com/wazuh/wazuh | free (GPL) | Agent-based XDR + SIEM, FIM, compliance | Event-centric dashboard; limited cross-vendor correlation | Correlation groups and threat-source/asset profiles as primary objects |

**Key Takeaways:**
- The gap is correlation and how relationships are presented, not collection. ThreatWatch ingests normalized events from the tools a team already runs and makes the source → target → detection chain navigable.

### 3.7 — Community & Forums

| Thread/Post | URL | Platform | Key Insight |
|-------------|-----|----------|-------------|
| Websockets over https | https://caddy.community/t/websockets-over-https/16871 | Caddy Community | Caddy v2 proxies WebSocket upgrades automatically; `wss://` works behind `tls internal` |
| 1.23.1 images don't work on linux/arm64 | https://github.com/temporalio/docker-builds/issues/194 | GitHub issues | Temporal image tags vary by platform: pin a tag and verify it on the host |
| Update WebSocket ws authentication guidance (PR #2500) | https://github.com/OWASP/CheatSheetSeries/pull/2500 | GitHub (OWASP) | Authenticate in the HTTP `upgrade` handler, not in `verifyClient` |
| Unstable express@5.2.1 dependency (vercel/workflow #3448) | https://github.com/vercel/workflow/issues/3448 | GitHub issues | Pin exact versions and commit the lockfile |

**Key Takeaways:**
- The WebSocket is authenticated in the `upgrade` handler, in order: Origin allow-list, then session cookie, then subscription to the tenant-scoped channel.

### 3.8 — APIs & Integrations

| API/Service | URL | Auth Method | Rate Limits | Docs Quality | SDK Available? |
|-------------|-----|-------------|-------------|--------------|----------------|
| ThreatWatch Ingest API (own, inbound) | — (SPEC §3) | per-integration HMAC (Standard Webhooks) + integration id | per-integration limiter | n/a | curl / any |
| Temporal frontend (gRPC :7233) | https://docs.temporal.io/develop/typescript/workers/run-worker-process | none in-cluster (private network) | n/a | good | `@temporalio/client`, `@temporalio/worker` |
| ThreatFox IOC API (optional threat-intel sync) | https://threatfox.abuse.ch/api/ | Auth-Key header (free) | fair use | good | HTTP JSON |
| DB-IP City Lite (offline) | https://db-ip.com/db/download/ip-to-city-lite | none (CC BY 4.0) | n/a (local data) | good | CSV import |
| SMTP (email notifications) | https://nodemailer.com/ | SMTP auth | provider | good | nodemailer; Mailpit in local compose |
| Customer webhook endpoints (outbound) | Standard Webhooks spec | HMAC signature computed by us | 5 attempts / ~6 h backoff | n/a | fetch + SSRF guard |

**Key Takeaways:**
- No paid third-party API is a hard dependency. ThreatFox needs a free key, so threat intel is an org-managed indicator list with an optional feed sync. With `THREATFOX_AUTH_KEY` blank, sync shows an explicit "disabled" state, never a fake success.
- Geo uses a seeded offline IP-range table, so there is no map-provider proxy and no billing-abuse vector. The has_geo map-proxy rate-limit item is N/A (ADR).

### 3.9 — Architecture & Design Patterns

| Pattern | Source | Why It Applies |
|---------|--------|----------------|
| Pipeline: ingest → normalize → enrich → correlate → detect → alert | OCSF https://schema.ocsf.io/ | Each stage is a pure function over the event; detection runs as a Temporal activity |
| Transactional outbox | BUILD.md has_dual_write / has_email / has_webhook_send; https://docs.temporal.io/develop/typescript/workflows/schedules | The event row and the outbox row are written in one transaction; Temporal drain workflows deliver |
| Entity-based correlation | https://learn.microsoft.com/en-us/azure/sentinel/map-data-fields-to-entities | Alerts sharing a source IP, asset or indicator within a window are grouped into a correlation group |
| Sigma-style rule conditions | https://sigmahq.io/sigma-specification/specification/sigma-rules-specification.html | A JSON rule: `match` clauses (field, op, value) plus an optional threshold (count over window, group-by) |
| Signed inbound webhooks with replay window | https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md | Verify the signature before any DB read; a webhook-id idempotency table (UNIQUE, ≥72 h retention) |
| Durable schedules | https://docs.temporal.io/develop/typescript/workflows/schedules | Retention sweep, ingestion-health check, outbox drains and report generation run as schedules, with a manual trigger endpoint for smoke |
| Tenant scope wrapper | MDLC service floor (reference/service-floor.md) | Every query goes through `scoped(orgId)`; cross-tenant access returns 404 |
| WebSocket fan-out via Postgres LISTEN/NOTIFY | https://www.postgresql.org/docs/current/sql-notify.html | The worker is a separate process and must push to the API's WebSocket hub; NOTIFY avoids Redis (small tier has none) |

**Key Takeaways:**
- The worker (Temporal activities) and the API run as separate containers. Real-time updates cross between them via Postgres `LISTEN/NOTIFY`, so the small tier needs no Redis.

---

## 4. Technology Stack Candidates

| Layer | Option A | Option B | Recommendation | Rationale |
|-------|----------|----------|----------------|-----------|
| Language | TypeScript 5.9.3 | TypeScript 7.0 (native) | **TS 5.9.3** | Toolchain (typescript-eslint, Next) is validated on 5.x |
| Framework (web) | Next.js 16.3 (App Router) | Vite + React | **Next.js 16.3.8** | Hard constraint; React Context for state |
| Framework (API) | Express 5.2 | Fastify | **Express 5.2.1** | Hard constraint; async error propagation |
| ORM | Prisma 7.10 | Prisma 8 RC | **Prisma 7.10.0** + `@prisma/adapter-pg` | Stable line; v8 is an RC |
| Database | PostgreSQL 17 | — | **PostgreSQL 17** | Hard constraint; also hosts the LISTEN/NOTIFY fan-out |
| Background jobs | Temporal CLI dev server (single container, SQLite file on a volume) | temporalio/server + Postgres + admin-tools | **Temporal CLI dev server** at the small tier | Temporal is a hard constraint; this is the supported single-container option, with the upgrade path documented |
| Realtime | `ws` 8.22 | Socket.IO | **ws** | Raw WebSocket with an Origin check per the has_websocket checklist; smaller surface |
| Hosting/Infra | Docker Compose + Caddy (`tls internal`) | Kubernetes | **Compose + Caddy** | HTTPS-only constraint; :80 → :443 redirect |
| Auth | DB sessions, `__Host-` cookie, argon2id | JWT | **DB sessions + argon2id** | Revocable, no token in JS; CSRF via Origin check |
| Validation / logs | zod 4.6 / pino 10.4 + pino-http | — | as stated | Service floor |
| Security middleware | helmet 8.3 + express-rate-limit 8.7 (in-process memory store) | Postgres-backed store | **memory store** | The small tier runs a single API instance, so an in-process store is exact. A shared store becomes necessary only when the API scales out (documented upgrade path) |
| Email | nodemailer + Mailpit (local) | SES / Resend | **nodemailer SMTP** | No provider specified; outbox pattern |
| Testing | Vitest + Testing Library; Playwright 1.63 + axe | Jest | **Vitest + Playwright** | WEB.md mandate |
| CI/CD | GitHub Actions | — | **.github/workflows/ci.yml** | Service floor |

---

## 5. Risk & Unknowns Register

| ID | Unknown / Risk | Severity | Mitigation | Status |
|----|----------------|----------|------------|--------|
| R1 | Scope: 12 workflows and 10 dashboard modules, against a standard build of 6–10 features at ≤1000 LOC each | high | Merge the workflows into ≤10 vertical-slice features. W12 report delivery becomes in-app report snapshots plus an email of the summary. Log every deferral | open → Stage 1 |
| R2 | `has_payments` is declared, but no workflow charges money | med | Reconcile as Deferred (ADR). Still emit the money invariant (no FLOAT in migrations) so a later billing feature inherits it | open → Stage 1 |
| R3 | The Temporal dev server is not HA and persists to SQLite | med | Acceptable at the small tier (single instance), volume-persisted; production upgrade path documented in QUICKSTART | resolved (ADR) |
| R4 | Cross-process realtime (worker → API WebSocket hub) without Redis | med | Postgres LISTEN/NOTIFY channel `tw_events` carrying `{orgId, kind, id}`; clients refetch scoped data | resolved |
| R5 | Inbound ingest abuse (flood, replay, oversized payloads) | high | HMAC verified before any DB access, 300 s window, UNIQUE webhook-id, 256 KB body cap, per-integration rate limit, batches ≤500 events | open → SPEC §4 |
| R6 | SSRF via customer-supplied outbound webhook URLs | high | `assertSafeUrl` before every fetch: https only, DNS-resolved private ranges blocked, redirect:'error', 5 s timeout; boundary-order invariant | open → SPEC §4 |
| R7 | Cross-tenant leakage (success metric: 0 exposures) | critical | Scope wrapper, 404 on foreign ids, tenant-isolation tests per resource, WebSocket channel = org | open → SPEC §4 |
| R8 | Detection latency target <30 s | med | Ingest starts the per-batch detection workflow immediately (no polling); an effect test asserts the alert exists after ingest | open |
| R9 | Geo DB size (121 MB MMDB) bloats the image | low | Ship a compact seeded IP-range → country/lat/lng table (DB-IP-derived, CC BY 4.0 attribution); full import is a documented script | resolved (ADR) |
| R10 | Toolchain churn (TS 7, Prisma 8 RC, Next 16 security releases) | med | Pin exact versions, commit the lockfile, run `npm audit` in CI | resolved |
| R11 | Windows host / Docker Desktop port conflicts | low | Probe ports first; env-overridable `${HTTPS_PORT:-8443}`; fail fast | open → env artifacts |
| R12 | Success metrics "≥99.9% ingestion availability" and "horizontal scaling without redesign" cannot be demonstrated by a single-instance small-tier compose deploy | med | Design for it (stateless API, durable Temporal workflows, idempotent ingest, no in-process event state other than the rate-limit store) but report these as **Deferred (not measured)** in the Build Input Reconciliation, never Applied | open → REPORT |
| R13 | "No silent event loss" plus duplicate, malformed, delayed and out-of-order events | high | Ingest returns per-event accept/reject counts. Malformed events go to an `ingest_errors` table shown on integration health. Dedupe uses UNIQUE `(integration_id, source_event_id)`. Event time and receive time are stored separately, and windows use event time | open → SPEC §5 |

---

## 6. Research Gaps

- [ ] Gap 1: Verify the exact supported `temporalio/temporal` image tag on the host at compose time (pin it; never `latest`).
- [ ] Gap 2: RESEARCH.md does not quantify the target events-per-second. Assume the small tier: 50 EPS sustained per org, bursts of 500-event batches (ADR).
- [ ] Gap 3: Data retention is blank. Assume 90-day raw-event retention, with alerts kept indefinitely (ADR), enforced by the retention schedule.
- [ ] Gap 4: No email provider is named. Use SMTP, with Mailpit locally; real SMTP credentials are a `.env` placeholder.
- [ ] Gap 5: Non-Goals are blank. Stage 1 derives them as ADRs: no endpoint agent, no SOAR auto-response, no billing UI, no SSO in v1.

- [ ] Gap 6: `security_baseline` is blank, so no regime checklist applies. Passwords still use NIST 800-63B length rules (min 12, max 128, no composition rules) and a bundled top-breached-password list for offline screening (ADR). This is not a compliance claim.

**Derived Non-Goals (Step 3b, logged as ADR-005):** no endpoint/agent software (ThreatWatch receives events pushed by existing tools); no automated response/SOAR actions; no billing or payment UI in v1 (`has_payments` deferred); no SSO/SAML in v1; no PDF export (reports are in-app snapshots plus an emailed summary); no mobile app; no third-party map-tile provider (the threat map renders an SVG world projection locally).

**Blocker?** No. Each gap has a safe default that is logged in DECISIONS.md and can be revisited.

---

## 7. Summary & Recommendation

ThreatWatch can be built on the mandated stack (Next.js 16, Express 5, Prisma 7, PostgreSQL 17, Temporal, `ws`, Caddy HTTPS) with no paid dependencies. Research confirmed the gap that competitors leave open: they group alerts by shared entities but present events, not relationships. It also found standards to anchor the model: OCSF for normalized events, a Sigma-style rule vocabulary, ATT&CK tags, and Standard Webhooks for signed ingest and notifications. The main risk is scope (R1): twelve workflows must fit into ≤10 vertical-slice features, with explicit deferrals. Two infrastructure facts differ from older guidance, and the stack choice absorbs both: `temporalio/auto-setup` is deprecated (the small tier uses the CLI dev server), and Next.js 16 replaced middleware with `proxy.ts`.

**Go / No-Go Decision:** `GO`

**Next Step:** → [ARCHITECTURE.md](./ARCHITECTURE.md)
