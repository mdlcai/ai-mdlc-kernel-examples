# RESEARCH.md — Cyberops

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Security"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_websocket", "has_dual_write"]
```

## Product Vision
**Problem:** Security teams, penetration testers, and organizations responsible for protecting digital infrastructure often rely on multiple disconnected tools for asset discovery, vulnerability scanning, penetration testing, OSINT, threat analysis, and compliance. This creates fragmented data, manual workflows, inconsistent risk prioritization, and significant time spent moving between tools and validating findings.

The problem is becoming more significant as attack surfaces grow, vulnerabilities are discovered faster, and organizations need continuous security validation rather than periodic assessments. Existing tools often specialize in a single area and can generate large volumes of findings without effectively correlating them into actionable attack paths and business risk.

The platform aims to address the need for a unified, AI-assisted CyberOps workflow that can continuously discover, assess, validate, correlate, and prioritize security risks across an organization's authorized attack surface.

**Who it affects:** Security Operations Centers (SOC) and security analysts — Need a unified view of assets, vulnerabilities, threats, and attack paths without manually correlating multiple security tools.
Penetration testers and red teams — Need automated reconnaissance, AI-assisted testing, exploit validation, evidence collection, and repeatable assessments.
Vulnerability management teams — Need to prioritize exploitable vulnerabilities based on actual risk rather than large volumes of isolated scanner findings.
Security engineers and DevSecOps teams — Need continuous scanning and testing across applications, infrastructure, cloud environments, and code.
Compliance and GRC teams — Need security findings mapped to controls, evidence collected automatically, and remediation tracked for audits.
Security leadership / CISOs — Need an executive-level view of organizational exposure, validated risk, remediation progress, and compliance posture.

The primary users are security analysts and penetration testers, with security engineers, GRC teams, and security leadership consuming the resulting findings, workflows, and reports.

**Why existing solutions fall short:** Current security platforms are often fragmented across separate tools for vulnerability management, penetration testing, OSINT, threat intelligence, asset discovery, and compliance. Organizations may need to operate multiple products, manually correlate their results, and switch between dashboards to understand the full security picture.

Many existing scanners also produce large volumes of findings without determining whether vulnerabilities are actually exploitable or how multiple weaknesses could be chained into a realistic attack path. AI-powered security tools are emerging, but they are often focused on a specific area, such as autonomous penetration testing, rather than providing a unified CyberOps workflow.

The gap is a platform that combines AI-powered penetration testing, continuous scanning, vulnerability management, OSINT, threat analysis, and compliance while automatically correlating findings into validated attack paths, prioritized risk, actionable remediation, and continuous retesting from a single interface.

**Solution:** A unified CyberOps portal that turns security data from scanning, vulnerability assessment, threat intelligence, OSINT, and compliance into a single actionable view of organizational risk.
Build these tools into the application:
1. Shannon
https://github.com/KeygraphHQ/shannon?
This is the one I'd study for the AI pentesting workflow. It isn't simply an LLM wrapper—it attempts to reason through attack paths and validate vulnerabilities with actual exploits.

2. Strix
https://github.com/usestrix/strix?
Study this for the AI-agent architecture and developer-facing workflow. It combines AI agents with real security tooling and emphasizes validated findings rather than just scanner output.

3. PentAGI
https://github.com/vxcontrol/pentagi?
Study this for multi-agent orchestration. Its architecture includes sandboxing, autonomous planning, persistent memory, and a collection of established pentesting tools.

4. CAI
https://github.com/aliasrobotics/CAI?
This is especially interesting if you want to build your own AI security-agent framework rather than hard-code one pentesting workflow. CAI provides agents, tools, model integrations, and security-oriented orchestration.

5. Nuclei
https://github.com/projectdiscovery/nuclei?
This should probably be one of your core scanning engines. Its template-based approach makes it particularly attractive for building a scalable scanner into your platform.

6. Trivy
https://github.com/aquasecurity/trivy?
Use this as inspiration for the broader vulnerability-management layer—especially containers, Kubernetes, IaC, SBOMs and cloud.

7. HexStrike AI
https://github.com/0x4m4/hexstrike-ai?
This one is particularly relevant to your overall vision because it demonstrates the idea of an AI agent controlling a large collection of existing security tools through an orchestration layer. It advertises 150+ integrated security tools and MCP-based agent integration.


## Users & Outcomes
**Key Workflows:**
AI-Powered Automated Penetration Test
User selects an authorized target and defines testing scope.
AI performs reconnaissance and attack-surface discovery.
AI orchestrates approved security testing tools.
Vulnerabilities are validated for exploitability where safely permitted.
Attack paths and evidence are generated.
Findings are risk-scored and presented in the dashboard.
User can review, export, remediate, and retest findings.
Asset Discovery & Vulnerability Assessment
User adds an authorized domain, IP range, application, cloud environment, or repository.
Platform discovers assets, ports, services, technologies, and dependencies.
Vulnerability scanners identify known vulnerabilities and misconfigurations.
Findings are correlated with asset criticality and threat intelligence.
Platform prioritizes vulnerabilities by risk and exploitability.
Threat & OSINT Investigation
Analyst submits an indicator, domain, IP, hostname, vulnerability, or threat actor.
Platform gathers authorized public intelligence and threat data.
AI correlates indicators, assets, vulnerabilities, and related threats.
Analyst receives an investigation summary with supporting evidence and risk context.
Compliance & Security Posture Assessment
User selects an applicable compliance framework.
Platform maps discovered vulnerabilities, configurations, and security controls to requirements.
Missing controls and evidence gaps are identified.
AI generates prioritized remediation recommendations.
Compliance status and evidence are presented through the dashboard and reports.
Finding Remediation & Continuous Retesting
User selects a vulnerability or security finding.
Platform provides remediation guidance and tracks its status.
After remediation, the user initiates or schedules a retest.
Scanners or the AI pentesting engine revalidate the finding.
The platform records the result, updates the risk score, and maintains an audit history.

**Success Metrics:**
AI Pentest Completion: At least 95% of authorized test runs complete successfully without unrecoverable errors.
Vulnerability Detection: At least 90% of known seeded vulnerabilities in the test environment are detected and reported.
Finding Validation: At least 80% of detected high/critical vulnerabilities receive an exploitability or validation assessment.
Asset Discovery: At least 95% of in-scope test assets are successfully discovered and inventoried.
Finding Correlation: 100% of findings are associated with an asset, severity/risk score, source, and timestamp.
Workflow Completion: All 5 critical workflows must function end-to-end during the Stage 4 smoke test.
Dashboard Performance: Core dashboard pages should load in under 3 seconds under the defined test workload.
Remediation Verification: At least 95% of remediated seeded vulnerabilities are correctly identified as resolved during retesting.
Auditability: 100% of security actions and AI-initiated testing activities must produce an auditable event containing the user/agent, target scope, timestamp, action, and outcome.
Authorization Safety: 100% of automated pentesting actions must remain within explicitly authorized target scope during testing.


## Build Constraints

```yaml
# Infrastructure & Ops
hosting_environment: "cloud-native (AWS/GCP/Azure)"
protocol_support: "HTTPS only"
monitoring: "metrics + alerting"
backup_strategy: "automated daily with point-in-time recovery"
container_strategy: "orchestrated / multi-instance"
error_reporting: "Sentry"

# Data & Storage
database_preference: "PostgreSQL"
data_retention_policy: "audit logs retained for 2 years; findings retained per compliance framework"
pii_handling: ["IP addresses", "domain names", "vulnerability details", "audit logs"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"

# Security & Compliance
security_baseline: ["OWASP Top 10", "SOC2"]
rate_limiting: true
audit_logging: true
secrets_management: "AWS Secrets Manager"

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
performance_requirements: ["dashboard load < 3s", "API response < 200ms", "sustained load at 50k+ concurrent"]
testing_strategy: "TDD"
logging_format: "structured JSON"

# Scope & Platform
scale: "large — 50k+ concurrent"
target_platforms: ["web"]
alert_channels: ["email", "webhook"]
report_formats: ["PDF", "JSON", "CSV"]

# Pipeline Attribution
mdlc_attribution: "structural"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
The visual system anchors on a composed, information-dense interface built for sustained analytical work—calm neutral surfaces (charcoal and off-white in light mode, deep navy and near-black in dark) paired with a disciplined 8px grid that enforces clarity without rigidity, where tables, forms, and data views are treated as primary craft surfaces rather than afterthoughts, each cell and input field precisely kerned and aligned to reduce cognitive load during threat assessment and remediation tracking. The accent color—a confident, slightly desaturated lime-green (#4add2c)—functions as a validation and action signal: it marks exploitable vulnerabilities, confirmed findings, successful agent runs, and remediation completion, appearing sparingly but unmistakably in buttons, status indicators, and risk-priority callouts so that analysts can scan a dense dashboard and immediately locate what demands attention without visual noise. Typography is anchored in a geometric sans-serif (Roboto Mono for data, Inter for UI prose) that conveys precision and technical authority while remaining legible at small sizes in tables and logs; hierarchy is enforced through weight and scale rather than color, with a restrained three-tier system (body, label, caption) that respects the analyst's need to absorb large volumes of structured information without distraction. Layout density is intentional: cards and panels are compact, whitespace is functional rather than decorative, and the grid accommodates both wide data tables and narrow sidebar controls without awkward gaps or forced breathing room—this is a tool for professionals who value information density over visual spaciousness. Motion is purposeful and minimal: state transitions (vulnerability status changes, agent progress, filter application) use subtle 200–300ms easing to confirm action without drawing attention away from content; loading states employ restrained progress indicators rather than animated illustrations, and micro-interactions (hover states, focus rings) are precise and high-contrast to support accessibility without introducing visual clutter. The accessibility floor is WCAG AA across all interactive elements, with a minimum 4.5:1 contrast ratio on all text, a 3:1 ratio on the accent green against both light and dark backgrounds, keyboard navigation fully supported with visible focus indicators, and a color-blind-safe palette where the lime-green accent is never the sole differentiator—status is always reinforced by icon, text label, or position in the hierarchy.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #4add2c | Buttons, links, active states |
| Secondary | #35d4b9 | Accents, badges, highlights |
| Accent | #bb53d1 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f4 | Cards, elevated containers |
| Text | #171d16 | Headings, body text |
| Text Secondary | #5f705c | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #4add2c | Buttons, links, active states |
| Secondary | #42d7be | Accents, badges, highlights |
| Accent | #c05ed4 | Callouts, hover states |
| Background | #090c09 | Page background |
| Surface | #131712 | Cards, elevated containers |
| Text | #eaecea | Headings, body text |
| Text Secondary | #869583 | Captions, muted text |
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
    primary: '#4add2c',
    secondary: '#35d4b9',
    accent: '#bb53d1',
    background: '#fafafa',
    surface: '#f4f5f4',
    foreground: '#171d16',
    muted: '#5f705c',
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
  --color-primary: #4add2c;
  --color-secondary: #35d4b9;
  --color-accent: #bb53d1;
  --color-background: #fafafa;
  --color-surface: #f4f5f4;
  --color-text: #171d16;
  --color-text-secondary: #5f705c;
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
  --color-primary: #4add2c;
  --color-secondary: #42d7be;
  --color-accent: #c05ed4;
  --color-background: #090c09;
  --color-surface: #131712;
  --color-text: #eaecea;
  --color-text-secondary: #869583;
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

---

# Research (Stage 0 — added 2026-08-15, build_depth: comprehensive)

> Research performed per RESEARCH.md Appendix A. Sources verified live against vendor references / GitHub / official specs on 2026-08-15. `force_research: false` and §3 was previously absent, so this is the first research pass.

## §3 Source Categories

### §3.1 Official / Vendor Documentation

All versions verified against the vendor's own docs / npm registry on 2026-08-15.

| Tool | Docs URL | Version (current stable) | Critical integration note |
|------|----------|--------------------------|---------------------------|
| NestJS | https://docs.nestjs.com | v11 (`@nestjs/core` 11.2.x) | Nest 11 = Express 5 semantics (named wildcard routes `*splat`, not bare `*`), Node 20+. Nest 12 (ESM-first) imminent — pin `^11`. |
| @nestjs/websockets | https://docs.nestjs.com/websockets/gateways | 11.2.x | Needs `@nestjs/platform-socket.io`. At 50k+ concurrent, install `@socket.io/redis-adapter` via a custom `IoAdapter` — default in-memory adapter is single-node only. |
| @nestjs/throttler | https://docs.nestjs.com/security/rate-limiting | v6 (6.5.x) | v5+ config is a named-throttler array `ThrottlerModule.forRoot([{ name, ttl, limit }])` with **ttl in milliseconds** (was seconds in v4 — silent 1000× bug). Use a Redis-backed `ThrottlerStorage` for multi-instance correctness. |
| TypeORM | https://typeorm.io | v1 (1.1.x) | Hit 1.0 in Jun 2026; ES2023, Node 20+. `DataSource` API drives migrations (`typeorm migration:generate -d data-source.ts`). Native `uuid`/`jsonb`/`enum`; enum renames generate destructive migrations — review generated SQL. |
| PostgreSQL | https://www.postgresql.org/docs/18/ | 18 (18.6) | Native `uuidv7()` — timestamp-ordered, index-friendly under heavy insert; prefer over `gen_random_uuid()` for scan/finding PKs at scale. |
| Next.js | https://nextjs.org/docs | v16 (16.3.x, App Router) | Request APIs are async: `params`, `searchParams`, `cookies()`, `headers()` return Promises — await them. Route handlers uncached by default. Turbopack is the default bundler. |
| shadcn/ui | https://ui.shadcn.com/docs/cli | CLI v4 | Not an npm dep — components vendored via `npx shadcn init` + `npx shadcn add <c>` (`components.json`). You own the code; updates manual. |
| Tailwind CSS | https://tailwindcss.com/docs | v4 | CSS-first: no `tailwind.config.js`; `@import "tailwindcss"` + `@theme { --color-…: … }` tokens in global CSS; PostCSS plugin is `@tailwindcss/postcss`. Older v3 snippets/`@tailwind base;` won't work. |
| TanStack Query | https://tanstack.com/query/latest | v5 (5.101.x) | Single object signature `useQuery({ queryKey, queryFn })`; `onSuccess`/`onError` removed from `useQuery`. App Router: `HydrationBoundary` + per-request `QueryClient`. |
| BullMQ | https://docs.bullmq.io | v6 (6.1.x) new; `^5` battle-tested | Redis must run `maxmemory-policy=noeviction` or jobs silently evict. One Redis connection per Worker/Queue; set `concurrency` per worker. |
| Resend | https://resend.com/docs/send-with-nodejs | Node SDK v6 (6.20.x) | Returns `{ data, error }` — does NOT throw; check `error` or emails fail silently. Default 2 req/s — use `resend.batch.send()` + a queue with rate limiter, never inline in request handlers. |
| Sentry | https://docs.sentry.io/platforms/javascript/guides/nestjs/ , /guides/nextjs/ | v10 (10.65+/10.70+) | OTel-based. NestJS: `Sentry.init()` in `instrument.ts` imported before all modules (top of `main.ts`), `SentryModule.forRoot()` + `SentryGlobalFilter` before custom filters. Next.js: `instrumentation-client.ts` + `onRequestError`. |
| Nuclei (ProjectDiscovery) | https://docs.projectdiscovery.io/tools/nuclei | v3 (3.8.x) | CLI-first: `nuclei -u <t> -jsonl -o results.jsonl`. Official Go SDK (`.../nuclei/v3/lib`) — no Node SDK, so spawn CLI in a BullMQ worker and parse JSONL for realtime progress. YAML templates, `-update-templates`. Pin ≥3.8.0. |
| Trivy | https://trivy.dev/latest/docs/ | v0.73.x | `trivy image|fs|repo <t> --format json`. Exit 0 even with vulns unless `--exit-code 1 --severity HIGH,CRITICAL`. First run downloads ~600MB DB — pre-warm/cache `~/.cache/trivy`. Pin exact verified versions (supply-chain incidents early 2026). |

**Cross-cutting version pins:** NestJS `^11` · TypeORM `^1.1` · PG `18` · Next.js `^16` · Tailwind `^4` · TanStack Query `^5` · BullMQ `^5` · Resend `^6` · Sentry `^10` · throttler `^6`. Several majors sit on a transition boundary (Nest 12 imminent, TanStack Query 6 beta, BullMQ 6 one month old) — pin explicitly, do not float `latest`. The 50k-concurrent target makes three notes load-bearing: **Socket.io Redis adapter, Redis-backed throttler storage, and `noeviction` Redis policy** (BullMQ on a dedicated logical DB, separate from the Socket.io adapter).

### §3.2 GitHub Repositories (reference implementations to study)

| Project | URL | Stars | Last active | License | Language |
|---|---|---|---|---|---|
| KeygraphHQ/shannon | https://github.com/KeygraphHQ/shannon | ~46.8k | 2026-08-14 | AGPL-3.0 | TypeScript |
| usestrix/strix | https://github.com/usestrix/strix | ~52.7k | 2026-08-14 | Apache-2.0 | Python |
| vxcontrol/pentagi | https://github.com/vxcontrol/pentagi | ~21.8k | 2026-08-06 | MIT | Go |
| aliasrobotics/cai | https://github.com/aliasrobotics/cai | ~9.7k | 2026-07-14 | NOASSERTION | Python |
| projectdiscovery/nuclei | https://github.com/projectdiscovery/nuclei | ~30.5k | 2026-08-15 | MIT | Go |
| aquasecurity/trivy | https://github.com/aquasecurity/trivy | ~37.4k | 2026-08-14 | Apache-2.0 | Go |
| 0x4m4/hexstrike-ai | https://github.com/0x4m4/hexstrike-ai | ~11.0k | 2026-08-03 | MIT | Python |

- **shannon** — White-box AI pentester: analyzes source to plan attack surface, dispatches vuln agents that attempt real PoC exploits, LLM-driven attack-path reasoning recon→analysis→exploitation. Study the source-informed attack-path graph. **AGPL-3.0 is a copyleft risk for a commercial platform** — do not link/derive; study only.
- **strix** — Python multi-agent orchestration; specialized agents collaborate in isolated Docker sandboxes with real offensive tooling. Reference for **agent-tool contracts and sandbox-per-target isolation**.
- **pentagi** — Go autonomous multi-agent (researcher/developer/executor) in sandboxed segmented-network Docker; **semantic memory via PostgreSQL + vector embeddings** for cross-engagement knowledge. Strong orchestration+persistence model.
- **CAI** — Python framework/SDK of agent primitives + tool integration + security benchmarks. Study as a framework layer; **non-standard license needs legal review**.
- **nuclei** — YAML-DSL templated scanner across apps/APIs/network/DNS/cloud. **Canonical model for a pluggable, community-extensible scan-signature engine** — our vuln-scanning module.
- **trivy** — Single-binary vuln/misconfig/secret/SBOM scanner. **Reference for the compliance/SBOM/IaC pillar.**
- **hexstrike-ai** — MCP server exposing 150+ tools to LLM agents. **The MCP-bridge pattern** — wrapping many offensive tools behind an agent-callable interface — is our AI-orchestration integration approach.

### §3.3 Video / Tutorials
`build_depth: comprehensive` — applicable but low-priority for this stack (all vendors have first-party written docs). Key hands-on references: NestJS official "First steps" + WebSockets guide (docs.nestjs.com), ProjectDiscovery Nuclei "Running Nuclei" (docs.projectdiscovery.io/tools/nuclei/running), and the shadcn/ui Next.js install walkthrough (ui.shadcn.com/docs/installation/next). Video tutorials deprioritized in favor of versioned written docs per §3.1.

### §3.4 Articles & Patterns
- **Transactional outbox** (dual-write DB→search/email): microservices.io/patterns/data/transactional-outbox.html · https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html · https://developer.confluent.io/courses/microservices/the-transactional-outbox-pattern/ — write domain row + outbox row in one ACID tx; relay/CDC publishes to indexer & email queue.
- **WebSocket auth & scaling**: OWASP WebSocket Security Cheat Sheet (Origin/CSWSH, authenticate-before-accept, per-conn size/rate limits, ping/pong) https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html · https://websocket.org/guides/security/ · Ably "Scaling Pub/Sub with WebSockets and Redis".
- **Idempotent webhook receiver**: Stripe webhook best practices https://docs.stripe.com/webhooks#best-practices — verify HMAC on raw body → INSERT event_id with UNIQUE → enqueue async → return 200 fast.

### §3.5 Standards / RFCs

| Standard | URL | Relevance |
|---|---|---|
| OWASP Top 10:2025 (Nov 2025; A01 Broken Access Control #1, new A03 Software Supply Chain, A10 Mishandling Exceptional Conditions) | https://owasp.org/Top10/2025/ | Risk categories to defend against AND map findings to. |
| OWASP ASVS 5.0 (May 2025; ~350 reqs, 17 chapters, 3 levels) | https://owasp.org/www-project-application-security-verification-standard/ | Verification checklist for our own auth/session/API/crypto controls. |
| AICPA SOC 2 — 2017 TSC (Revised Points of Focus 2022) | https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria-with-revised-points-of-focus-2022 | Criteria the compliance module maps to; our own audit target. |
| NIST SP 800-63B-4 (final 31 Jul 2025) | https://csrc.nist.gov/pubs/sp/800/63/b/4/final | Password verifier rules: min length, no forced composition/rotation, breached-password screening, phishing-resistant MFA. |
| CVSS v4.0 (FIRST.org, since Nov 2023) | https://www.first.org/cvss/v4.0/ | Canonical severity scoring for every finding. |
| MITRE ATT&CK v19 (28 Apr 2026) | https://attack.mitre.org/ | Technique taxonomy for pentest actions + threat analysis. |
| MITRE CWE (v4.x) | https://cwe.mitre.org/ | Weakness taxonomy for classifying findings, cross-ref CVEs. |
| STIX 2.1 / TAXII 2.1 (OASIS, 2021) | https://docs.oasis-open.org/cti/stix/v2.1/os/stix-v2.1-os.html | Interchange + transport for OSINT/threat-feed ingest & indicator export. |

### §3.6 Competing Products
- **Tenable One / Nessus** — exposure management on Nessus heritage; deepest vuln-signature coverage, but scan-and-report (identifies, doesn't autonomously exploit/chain).
- **Qualys VMDR / TruRisk** — cloud-native VM detection & response; strong prioritization/compliance, weak on autonomous offensive validation.
- **Rapid7 InsightVM (+Metasploit)** — VM + mature exploit library, but exploitation is analyst-driven, not AI-orchestrated attack-path reasoning.
- **Horizon3.ai NodeZero** — autonomous continuous pentesting that actually exploits & chains weaknesses (137% YoY ARR 1H2025). Closest analog to our AI-pentest pillar; we differentiate by bundling scanning + OSINT + asset discovery + compliance around the same autonomy.
- **Pentera** — automated security validation, fast setup/ROI, safe attack emulation; validation-focused, not a full multi-domain suite.
- **XM Cyber** — continuous exposure management, graph attack-path modeling toward critical assets (simulation), vs a platform that both models and executes.

### §3.7 Community Threads
Deprioritized per stack maturity; primary community references: ProjectDiscovery Discord + nuclei-templates repo issues (template authoring), NestJS Discord (WebSocket + throttler scaling), r/netsec + r/AskNetsec (authorization-scope / legal norms for automated pentesting). Consensus load-bearing point echoed across threads: **scope authorization + SSRF egress control are the non-negotiable safety gates** for any product that runs scans on user-supplied targets — captured in §5.

### §3.8 APIs / Integrations
- **Scan engines (subprocess, CLI→JSONL/JSON):** Nuclei v3 (`-jsonl`), Trivy v0.73 (`--format json`). Wrapped behind a `ScannerAdapter` interface; spawned in BullMQ workers; degrade gracefully to built-in safe adapters when the binary is absent.
- **Email:** Resend v6 (`{data,error}`, batch send via queue).
- **Errors/telemetry:** Sentry v10 (@sentry/nestjs, @sentry/nextjs).
- **Secrets:** AWS Secrets Manager (prod); env-var + `.env` (local/dev), documented in ADR.
- **Threat intel:** STIX 2.1 / TAXII 2.1 for feed ingest; NVD/CVE + EPSS + KEV for enrichment (adapter-based, offline-capable seed corpus for sandbox).
- **AI orchestration:** Anthropic Claude (Messages API / MCP) as the reasoning layer for the AI-pentest planner (provider-abstracted).

### §3.9 Patterns (architectural)
- **Transactional outbox** for every dual-write (DB→search index, DB→email, DB→outbound webhook) — the `has_dual_write`/`has_email`/`has_webhooks` domain signals all resolve here.
- **Scanner-adapter + queue-worker** — pluggable adapters behind one interface, run in BullMQ workers, JSONL streamed to WebSocket for realtime progress.
- **Multi-agent phase decomposition** (VulnBot / AutoPentest / xOffense, arXiv 2501.13411 / 2505.10321 / 2509.13021): specialized recon/scan/exploit agents + a validation/summarizer agent, structured tool-call outputs, human-in-the-loop before state-changing/exploit actions.
- **Append-only hash-chained audit log** for SOC2 non-repudiation.
- **Deny-by-default authorization scope** (verified target ownership) checked at job submit AND re-checked before every network action.
- **SSRF-safe target resolution** — resolve-then-validate against allowlist, block RFC-1918/link-local/reserved, pin resolved IP (anti-DNS-rebinding), isolated egress.

## §4 Stack Candidates & Decision

The RESEARCH Build Constraints pin the stack as hard constraints; §3.1 verified each is current and viable. Resolved stack (alternatives considered in parentheses):

- **Frontend:** Next.js 16 App Router + shadcn/ui (vendored) + Tailwind v4 + TanStack Query v5. *(Alt: Remix/Vite SPA — rejected; constraints fix Next.js and SSR/streaming aids the dense dashboard.)*
- **Backend:** NestJS 11 REST, URL-path versioning `/v1/`, TypeORM 1.1. *(Alt: Fastify-standalone / tRPC — rejected; constraints fix NestJS + REST; Nest DI suits the module-per-domain design.)*
- **DB:** PostgreSQL 18 (`uuidv7()` PKs, `jsonb` for finding payloads, RLS for tenant isolation). *(Alt: Mongo — rejected; relational findings/assets/controls + RLS demand Postgres.)*
- **Async/cache/realtime:** Redis + BullMQ v5 workers; Socket.io + Redis adapter for WebSocket fan-out. *(Alt: SQS/SNS — rejected for local-runnable parity; documented as a cloud swap.)*
- **Scanners:** Nuclei v3 + Trivy adapters (subprocess), plus built-in safe recon adapters (DNS/HTTP-fingerprint/TLS) that always run without external binaries.
- **AI:** Claude Messages API behind a provider-abstracted planner; agent tools are an allowlisted, sandboxed action set.
- **Email/telemetry/secrets:** Resend v6 · Sentry v10 · AWS Secrets Manager (prod) / env (dev).

## §5 Risk Register & Threat Model (comprehensive)

**Threat model — a platform that itself runs authorized offensive tooling (highest-priority risks):**

| # | Risk | Likelihood/Impact | Mitigation direction |
|---|------|-------------------|----------------------|
| R1 | **Out-of-scope / unauthorized scanning** (CFAA/legal liability) | Med / Critical | Signed per-engagement authorization scope (verified domain/CIDR ownership); checked server-side at job submit AND re-validated before every network action; deny-by-default. **INV-authz-scope.** |
| R2 | **SSRF via user-supplied targets** (169.254.169.254, RFC-1918, localhost) | High / Critical | Resolve-then-validate vs allowlist; block private/link-local/reserved; pin resolved IP (anti-rebinding); isolated egress. **INV-ssrf-guard.** |
| R3 | **Scan credential/secret handling** | Med / High | KMS/vault envelope encryption; inject only into ephemeral workers; never log; scope/rotate per engagement. |
| R4 | **Cross-tenant leakage of findings/targets/reports** | Med / Critical | Tenant isolation at data layer (RLS + `tenant_id` on every row); tenant context carried through every job/webhook/ws channel; scoped-write wrapper. **INV-tenant-scope.** |
| R5 | **Audit-log tampering** (breaks SOC2 non-repudiation) | Low / High | Append-only, hash-chained tamper-evident log written via outbox in the same tx as the action; immutable sink. **INV-audit-append-only.** |
| R6 | **Scanning engine abused as attack proxy / DDoS amplifier** | Med / High | Per-tenant rate/concurrency budgets, packet-rate throttling, target reputation checks, outbound-volume anomaly detection, human approval gate for high-impact modules. |
| R7 | **Prompt injection / unsafe autonomy in LLM agents** (malicious banners/OSINT hijack tool calls) | High / High | Treat all target-derived text as untrusted; sandbox tools with an allowlisted action set; scope re-check + confirmation before state-changing/exploit actions; validation agent + human-in-the-loop. |
| R8 | **Insecure handling of findings/exploit artifacts** | Med / High | Encrypt at rest; minimize/redact captured data; short retention + secure deletion; signed access-controlled report export/webhook delivery. |

**Standard product risks:**
- R9 Broken access control (OWASP A01) — object-level authz on every user-owned resource + sub-resource, not just tenant scope.
- R10 Software supply chain (OWASP A03:2025) — pin exact scanner-binary versions/checksums (Trivy incidents early 2026); lockfiles; `npm audit`/Trivy in CI.
- R11 Bleeding-edge version churn — Nest 12 / TanStack Query 6 / BullMQ 6 on the boundary; pin `^`-ranges, no `latest`.
- R12 Auth verifier gaps — NIST 800-63B-4: breached-password screening (HIBP k-anonymity), length≥8 (≥12 for PCI), no forced composition/rotation, rate-limit before hash compare.
- R13 Money/PCI — N/A (no payments in scope for this build; pricing page is marketing only).

**Safety boundary (build-level decision):** This build implements the full discover→assess→validate→correlate→prioritize→retest workflow and a real, extensible scanner-adapter engine with **safe built-in recon** (DNS resolution, HTTP fingerprinting, TLS inspection) against authorized targets, and a **template-driven finding engine** (Nuclei-style signatures). It models exploit-validation *state* (`unvalidated → validating → validated/failed`) but does **not** ship a live-exploitation module that fires payloads at arbitrary internet targets — that is gated behind explicit authorization scope + human approval and, in this sandbox, runs as a simulated validation adapter. This is the ethically and legally correct boundary (authorized security testing only; no mass/destructive capability) and is enforced by R1/R2/R6/R7 mitigations.

## §6 Research Gaps
- **Live exploitation adapters** intentionally out of scope for the initial build (safety boundary above); the adapter interface is designed so an authorized, sandboxed exploit runner can be added later behind the approval gate.
- **Real Nuclei/Trivy binaries** may be absent in the build sandbox — adapters degrade to built-in safe recon + a bundled offline template/CVE seed corpus so the engine is demonstrably functional without network-fetching a 600MB DB.
- **AWS-managed externals** (Secrets Manager, RDS, ElastiCache, SES/Resend prod keys) are unprovisioned in the build environment — the build targets a local docker-compose stack (Postgres + Redis) with a documented cloud-swap path.
- **STIX/TAXII live feeds** require credentials; ingest is built against the spec with a seed feed for the sandbox.

## §7 Summary & Go/No-Go

The problem (fragmented security tooling → a unified, AI-assisted CyberOps workflow) is well-formed, the five key workflows are concrete and testable, and the constrained stack is current and viable (every version verified §3.1). Reference implementations (§3.2) validate the architecture (multi-agent orchestration + pluggable scanner adapters + outbox persistence). The dominant risks are safety/authorization risks (R1–R7), all of which have clear, buildable mitigations expressed as architectural invariants. The only scoped-out capability is live exploitation, which is the correct safety boundary and does not block any of the five workflows (validation is modeled as state + a gated adapter).

**Go/No-Go: GO** — build proceeds to Stage 1. Sources: 14 vendor/official · 7 GitHub · 8 standards/RFCs + 6 competitors + pattern/threat references.
