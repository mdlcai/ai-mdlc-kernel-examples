# RESEARCH.md — STΛRSHΞILD

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "CYBERSECURITY"

## Domain Signals

```yaml
domain_signals: ["has_geo", "has_scheduled_work"]
```

## Product Vision
**Problem:** Distributed security operations teams must investigate signals across fragmented endpoint, identity, network, application, cloud, and vulnerability tools. Context switching, inconsistent evidence quality, noisy alerts, poor asset linkage, and delayed decisions raise response time and risk. Ungoverned AI automation adds risks of fabricated conclusions, overbroad containment, policy bypass, and unverified remediation. Organizations need a defensible, measurable path from signal to verified response.

**Who it affects:** - **SOC analysts and threat hunters:** fewer context switches, evidence-linked case triage, and explainable prioritization.
- **Incident commanders and response approvers:** scoped action proposals, blast-radius previews, dual-control approvals where required, and rollback visibility.
- **Security engineers and architects:** reusable connectors, detection rules, policy-as-code, asset/identity relationships, and safe integration boundaries.
- **CISOs, directors, and executive stakeholders:** operational risk, exposure trends, service health, and decision-ready reporting in COMMAND.
- **Auditors, privacy, and compliance teams:** immutable/tamper-evident records, access history, decision provenance, retention and export controls.
- **IT, cloud, network, and branch operations:** targeted remediation with maintenance windows and business-critical asset safeguards.
- **Managed-service or multi-tenant operators:** strict tenant isolation, delegated roles, quotas, and scoped investigations.

**Why existing solutions fall short:** Many incumbent SIEM/XDR/SOAR and exposure products are powerful within their own ecosystems but may require costly integration, duplicated telemetry, custom playbook maintenance, or multiple consoles to assemble a cross-domain investigation. Automated recommendations are not always accompanied by independently replayable evidence and explicit approval/rollback gates. Some products provide partial coverage for legacy and highly distributed environments. These are market-level design hypotheses, not claims that every competing product lacks these features. STΛRSHΞILD differentiates through an interoperable control plane, independently enforced validation, evidence-first workflows, and measurable closed-loop optimization; these advantages must be demonstrated against baselines rather than assumed.

**Solution:** STΛRSHΞILD is an enterprise-grade, vendor-neutral autonomous cyber defense control plane, branded **I AM // STΛRSHΞILD** under **INTELLIGENCE IN MOTION™** and governed by **MDLC.ai**. It unifies security telemetry, identity and endpoint detections, cloud and network exposure, attack-path intelligence, AI-assisted investigations, and policy-bounded response. The **QAI Evolution Engine** proposes and optimizes detection, investigation, and mitigation candidates; an independent validation plane and zero-trust policy gate control adoption and execution. An append-only, tamper-evident evidence ledger preserves inputs, reasoning references, approvals, actions, and independently observed outcomes.

Deliver two coordinated portal experiences with shared APIs, data, identity, and governance: **COMMAND**, the executive/analyst overview (security KPIs, active cases, QAI recommendations, governance status, evidence summaries), and **TACTICAL**, the SOC operations center (global threat visualization, threat feed, attack-path graph, response timeline, posture analytics). The holographic globe and all illustrative incident counts in design references are visual concepts, not real telemetry. Every production metric must be sourced, time-stamped, permission-filtered, and distinguish real data from demo/synthetic data.


## Users & Outcomes
**Key Workflows:**
1. **Observe and normalize:** collect from EDR, IAM, SIEM, network, cloud, and vulnerability systems; validate source authenticity, schema, tenant, and event time.
2. **Correlate and investigate:** enrich alerts with assets, identities, threat intelligence, and related events; generate evidence-linked cases and attack-path hypotheses.
3. **Propose and optimize:** QAI ranks hypotheses and bounded defensive options with confidence, costs, and constraints; no claim of quantum speedup without benchmarks.
4. **Independently validate:** separate validator checks evidence provenance, policy, confidence calibration, blast radius, change windows, reversibility, and approval requirements.
5. **Authorize and respond:** read-only by default; low-risk automation only after explicit policy enablement; high-impact containment requires authorized human approval and just-in-time scoped credentials.
6. **Verify and recover:** independently measure post-action telemetry, detect side effects, rollback or escalate on failure, and record final disposition.
7. **Learn and govern:** evaluate false positives, missed detections, drift, safety, latency, and cost; accept model/rule changes only through MDLC reviewer gates and versioned releases.
8. **Operate the portal:** COMMAND summarizes validated posture and cases; TACTICAL exposes event timelines, threat geography where location is grounded, graph exploration, and response controls; both enforce RBAC/ABAC and tenant filtering.

**Success Metrics:**
The following are **proposed pilot targets**, subject to baseline measurement and risk acceptance, not measured results:
- **Evidence completeness:** ≥99% of response decisions include source references, validator result, policy version, approver/executor identity, and outcome.
- **Unauthorized high-impact actions:** zero in tests and production; every attempted bypass is blocked and logged.
- **Validation coverage:** 100% of response actions pass independent policy checks before execution.
- **Tenant isolation:** zero cross-tenant data exposures in automated and adversarial tests.
- **Investigation efficiency:** ≥25% reduction in median analyst triage time versus a comparable baseline on a defined pilot dataset.
- **Detection quality:** no statistically material regression (defined: recall drop ≤ 2 percentage points and false-positive-rate rise ≤ 2 pp, 95% CI) against the locked baseline _(Step 3b fix #3)_; report confidence intervals and test-set composition.
- **Response verification:** ≥99% of executed actions have independent post-action verification or a documented escalation.
- **Reliability:** ≥99.9% monthly availability target for the control plane after production hardening, excluding documented maintenance.
- **Recovery:** 100% of supported reversible response playbooks have tested rollback paths; document exceptions.
- **Governance:** zero bypassed mandatory MDLC reviewer gates; every production policy/model release has reproducible evaluation evidence.

**Non-Goals:**
- Replace all existing SIEM, EDR, firewall, IAM, or vulnerability management products in V1.
- Grant an LLM unrestricted privileges or allow it to self-approve response actions.
- Automatically isolate business-critical systems, disable privileged identities, or change production network policy without explicit authorization and guardrails.
- Perform offensive intrusion, exploit development, persistence, or destructive security testing.
- Promise complete threat prevention, perfect attribution, zero false positives, or fully autonomous incident closure.
- Claim proven quantum advantage or real-time predictive certainty without independent experimental evidence.
- Use illustrative portal globe locations, alert counts, or scores as factual operational data.
- Build a general-purpose ITSM, CMDB, or observability replacement in the first release.
- Ship vendor-specific connectors (CrowdStrike, Okta, Splunk, AWS, etc.) in V1 — V1 provides a signed generic ingest API plus a clearly-badged synthetic connector _(Step 3b fix #4)_.


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
monitoring: "Prometheus + Grafana"

# Data & Storage
database_preference: "PostgreSQL"

# Security & Compliance
rate_limiting: true
audit_logging: true

# Frontend
frontend_framework: "Next.js"
state_management: "Redux"

# Backend
backend_framework: "Express"
api_style: "REST"
orm_preference: "Prisma"
background_jobs: "BullMQ"

# Performance & Quality
performance_requirements: ["sustained load test at 50k+ concurrent users"]
logging_format: "structured JSON"

# Scope & Platform
scale: "large — 50k+ concurrent"
multi_tenant: true
target_platforms: ["web"]

# Step 3b fixes #5–#6 (resolved defaults, see DECISIONS.md)
auth_model: "email/password + server-side sessions (SSO deferred)"
jit_credentials: "platform-issued short-lived signed action tokens (≤15 min); external vault deferred"
deployment: "Docker Compose behind Caddy TLS edge"
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
Composed and dense without clutter. Calm neutral surfaces, a disciplined grid, tables and forms done well, restrained accent use, subtle depth. Functional motion, not decoration.

This art-direction brief is a binding directive — honor its palette feel, imagery, type personality, layout mood, and motion over the archetype defaults (per `DESIGN.md` precedence). An uploaded Design Template still outranks it on concrete tokens.

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
- Heading: Rajdhani 600/700 (display labels) + Orbitron 700 (wordmark) per Design Template — _(Step 3b fix #2: Outfit superseded by template typography)_
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
- Variant: Sharp
- Border radius: 2px (sm: 0px, lg: 6px, xl: 10px)
- Shadows: None — `none`
- Theme: auto — dark default (Design Template), derived light palette, runtime toggle persisted per user  _(Step 3b fix #1: was "Light only", contradicted Brand Voice "full dark mode" and the dark template)_

### Tailwind Config
```typescript
// tailwind.config.ts — theme.extend
{
  colors: {
    primary: '#5e60d2',
    secondary: '#8a8da8',
    accent: '#6c63f1',
    background: '#fafafa',
    surface: '#ffffff',
    foreground: '#08090a',
    muted: '#6c6f7a',
    success: '#10b981',
    warning: '#f59e0b',
    destructive: '#ef4444',
  },
  fontFamily: {
    heading: ['Outfit', 'system-ui', 'sans-serif'],
    body: ['Inter', 'system-ui', 'sans-serif'],
    mono: ['JetBrains Mono', 'monospace'],
  },
  borderRadius: {
    DEFAULT: '2px',
    sm: '0px',
    lg: '6px',
    xl: '10px',
  },
}
```

### CSS Custom Properties
```css
/* Light mode */
:root {
  --color-primary: #5e60d2;
  --color-secondary: #8a8da8;
  --color-accent: #6c63f1;
  --color-background: #fafafa;
  --color-surface: #ffffff;
  --color-text: #08090a;
  --color-text-secondary: #6c6f7a;
  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
  --font-heading: 'Outfit', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 2px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #7d70ff;
  --color-secondary: #b4b5c1;
  --color-accent: #8182f8;
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

# Research Sections (Stage 0 — researched 2026-11-07, build_depth: comprehensive)

## 3. Source Categories

### 3.1 — Official Documentation & Vendor Sources

| Source | URL | What It Covers | Notes |
|--------|-----|----------------|-------|
| Next.js — Content Security Policy guide | https://nextjs.org/docs/15/app/guides/content-security-policy | Per-request nonce in middleware, `'strict-dynamic'`, `x-nonce` header handoff | A nonce forces dynamic rendering for routes that use it. This drives WEB.md CSP Profile N. |
| Redux Toolkit — Setup with Next.js | https://redux-toolkit.js.org/usage/nextjs | App Router `makeStore()` factory per request, client-only Provider, typed hooks | Server Components must not touch the store |
| BullMQ — Job Schedulers | https://docs.bullmq.io/guide/job-schedulers/ | `upsertJobScheduler(id, {pattern or every}, template)` replaces legacy repeatable jobs | Upsert is idempotent per scheduler id, so it is safe to call on every boot |
| Grafana — Node.js exporter / prom-client | https://grafana.com/oss/prometheus/exporters/nodejs-exporter/ | `collectDefaultMetrics()`, `/metrics` with `register.contentType` | Counter and Histogram for RED metrics |
| Grafana k6 — constant-arrival-rate executor | https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/ | Open-model load: `rate`, `timeUnit`, `preAllocatedVUs`, `maxVUs` | Open model separates generator saturation from SUT slowness |
| Grafana k6 — dropped iterations | https://grafana.com/docs/k6/v2.3.x/using-k6/scenarios/concepts/dropped-iterations/ | Interpreting `dropped_iterations` | Used in the load-test verdict |
| NIST SP 800-61 Rev. 3 (final, Apr 2025) | https://csrc.nist.gov/pubs/sp/800/61/r3/final | IR recommendations as a CSF 2.0 Community Profile; supersedes Rev. 2 | Incident lifecycle maps to the CSF functions |
| NIST news on SP 800-61r3 | https://nist.gov/news-events/news/2025/04/nist-revises-sp-800-61-incident-response-recommendations-and-considerations | Summary of changes | — |
| Pwned Passwords k-anonymity (Troy Hunt) | https://www.troyhunt.com/understanding-have-i-been-pwneds-use-of-sha-1-and-k-anonymity/ | `GET https://api.pwnedpasswords.com/range/{5-hex-prefix}`, `Add-Padding: true` | Range endpoint needs no key. When unreachable, fail open and log a warning. |
| Cloudflare — leaked passwords with k-anonymity | https://blog.cloudflare.com/validating-leaked-passwords-with-k-anonymity/ | Privacy rationale for range queries | — |

**Key Takeaways:**
- The CSP nonce is minted in `middleware.ts` per request. App pages are dynamic anyway (authenticated, tenant data), so the static-render cost is acceptable.
- BullMQ `upsertJobScheduler` makes schedule definition idempotent at boot. The DB-level claim (`UPDATE … WHERE claimed_at IS NULL`) remains the effect-idempotency guard (has_scheduled_work item 4).
- k6 open-model executors plus `dropped_iterations` give the saturation-vs-slowness split that the Load Test Methodology requires.

### 3.2 — GitHub & Open Source

| Repository | URL | Stars | Last Active | Relevance |
|------------|-----|-------|-------------|-----------|
| express-rate-limit/rate-limit-redis | https://github.com/express-rate-limit/rate-limit-redis | ~200 (MIT, TS) | active 2025–26 | Redis store for express-rate-limit, shared across API replicas. Dependency. |
| siimon/prom-client | https://github.com/siimon/prom-client | ~3.5k (Apache-2.0) | active | Prometheus client for Node. Dependency. |
| taskforcesh/bullmq | https://github.com/taskforcesh/bullmq | ~7k (MIT) | active | Queue/scheduler. Dependency. |
| mitre/cti | https://github.com/mitre/cti | ~2k (ATT&CK terms) | active | ATT&CK in STIX 2.1. Technique IDs on findings and attack-path edges. Reference; we seed a subset of IDs. |
| TheHive-Project/TheHive | https://github.com/TheHive-Project/TheHive | ~3.5k (AGPL) | StrangeBee-maintained | Prior art for the case and observable model. Reference only (AGPL). |
| StrangeBeeCorp (org) | https://github.com/StrangeBeeCorp | — | active | Docker profiles for TheHive/Cortex. Reference. |
| luisaveiro/docker-reverse-proxy | https://www.github.com/luisaveiro/docker-reverse-proxy | small | 2024 | Caddy `tls internal` compose pattern. Reference. |

**Evaluation Checklist:**
- [x] License compatible: runtime dependencies are permissive (MIT, Apache-2.0, ISC, BSD). The one exception is sharp's optional prebuilt libvips binaries (`@img/sharp-libvips-*`, LGPL-3.0-or-later), which are dynamically linked and unmodified, so they are compatible (SEC-12). AGPL projects are reference only.
- [x] Actively maintained: bullmq, prom-client and rate-limit-redis all have commits within the last 6 months.
- [x] Good documentation: official docs exist for every dependency.
- [x] Community size: large for bullmq/prom-client; small but org-backed for rate-limit-redis.
- [x] Dependency vs reference is marked per row.

**Key Takeaways:**
- Use the current `express-rate-limit` org repo. Older forks (`mifi`, `knoxcard2`) have outdated APIs.
- Open-source SOC tooling (TheHive/Cortex, Wazuh) validates the case, observable and evidence model, but none provides governed AI proposals with independent validation. That gap is the differentiator.

### 3.3 — Video & Tutorial Sources

| Title | URL | Creator | Length | Why It's Useful |
|-------|-----|---------|--------|-----------------|
| Bull-Ish Your Queues With BullMQ (Node Congress 2025) | https://gitnation.com/contents/bull-ish-your-queues-with-bullmq | Luca Del Pupo (NearForm) | ~25 min | Production BullMQ patterns: retries, concurrency, schedulers |
| Vercel screwed up (Next.js CVE-2025-29927 breakdown) | https://rosetta.to/u/t3dotgg/vercel-screwed-up-breaking-down-the-next-js-cve | Theo (t3.gg) | ~20 min | Middleware must not be the only auth layer |
| Managing Authentication in Next.js | https://gitnation.com/contents/managing-authentication-in-nextjs | GitNation (Node Congress) | workshop | Session/cookie patterns across server and client |

**Key Takeaways:**
- Defense in depth: Next middleware redirects for UX only. The Express API re-verifies session, tenant and role on every request.

### 3.4 — Articles, Blogs & Written Tutorials

| Title | URL | Author/Site | Date | Relevance |
|-------|-----|-------------|------|-----------|
| Next.js App Router authentication guide 2026 | https://workos.com/blog/nextjs-app-router-authentication-guide-2026 | WorkOS | 2026-02 | Multi-layer verification |
| Best way to handle auth in App Router | https://clerk.com/blog/best-auth-nextjs-app-router.md | Clerk | 2025 | Route handlers are public endpoints |
| Next.js CSP nonce investigation | https://zenn.dev/hcompany/articles/20250822_nextjs_csp | hcompany (zenn) | 2025-08 | Confirms that a nonce requires dynamic rendering |
| Multi-tenancy with Prisma and PostgreSQL RLS | https://stratum-docs.pages.dev/guides/prisma-rls/ | Stratum | 2025 | `set_config` in a transaction, FORCE RLS, non-owner role |
| Prisma row-level security | https://atlasgo.io/guides/orms/prisma/row-level-security | Atlas | 2025 | RLS policy definitions alongside Prisma |
| What is alert fatigue | https://panther.com/blog/what-is-alert-fatigue | Panther | 2025 | Prioritise by asset criticality, not volume |
| How to reduce alert fatigue | https://www.kaspersky.com/resource-center/b2b/how-to-reduce-alert-fatigue-in-siem-and-soc-environments | Kaspersky | 2025 | Detection tuning and triage workflow |
| What's new in ASVS 5.0 | https://softwaremill.com/whats-new-in-asvs-5-0/ | SoftwareMill | 2025 | Requirement mapping for COMPLIANCE.md |
| Top SIEM tools 2026 | https://guptadeepak.com/top-5-siem-tools-of-2026-microsoft-sentinel-vs-splunk-vs-the-rest/ | Deepak Gupta | 2026 | Competitive landscape |

**Key Takeaways:**
- Prisma has no first-class RLS. Primary isolation is an app-layer tenant scope: every repository query takes `orgId`, enforced by a lint invariant. Postgres RLS as a second wall is evaluated in the ARCHITECTURE alternatives (gap G3).
- Asset-criticality-weighted prioritisation is the analyst-validated fix for alert fatigue, and it feeds the QAI ranking formula.

### 3.5 — Standards, RFCs & Specifications

| Standard | Reference | URL | Applicability |
|----------|-----------|-----|---------------|
| STIX 2.1 | OASIS CTI TC | https://docs.oasis-open.org/cti/stix/v2.1/cs02/stix-v2.1-cs02.pdf | Indicator vocabulary (STIX-compatible `type` and `pattern` fields; no full bundles in v1) |
| MITRE ATT&CK | mitre/cti | https://github.com/mitre/cti | Technique IDs on findings and attack-path edges |
| NIST SP 800-61r3 / CSF 2.0 | NIST | https://csrc.nist.gov/pubs/sp/800/61/r3/final | Incident state machine and response timeline phases |
| OWASP ASVS 5.0.0 | OWASP | https://github.com/OWASP/ASVS | Verification checklist (baseline: OWASP Top 10 plus an ASVS L2 subset) |
| NIST SP 800-63B | NIST | https://pages.nist.gov/800-63-4/sp800-63b.html | Password rules: minimum length, breach check, no composition rules, no forced rotation |
| RFC 6797 HSTS | IETF | https://www.rfc-editor.org/rfc/rfc6797 | HTTPS-only constraint; HSTS on hosts other than localhost |
| RFC 9457 Problem Details | IETF | https://www.rfc-editor.org/rfc/rfc9457 | REST error envelope |
| WCAG 2.2 AA | W3C | https://www.w3.org/TR/WCAG22/ | UI floor |

**Key Takeaways:**
- 800-61r3 drops the linear Rev. 2 lifecycle for the CSF functions. Our incident states (`open → triaged → contained → eradicated → recovered → closed`) each map to a CSF function in metadata.

### 3.6 — Competing / Adjacent Products

| Product | URL | Pricing | Strengths | Weaknesses | Our Differentiation |
|---------|-----|---------|-----------|------------|---------------------|
| Microsoft Sentinel | https://www.gartner.com/reviews/product/microsoft-sentinel/alternatives | Ingest-based, high at list price | Cloud-native, M365/Azure depth | Ecosystem gravity, cost at volume | Vendor-neutral; governed proposals with an independent validator |
| Splunk Enterprise Security | https://www.peerspot.com/products/comparisons/microsoft-sentinel_vs_splunk | Ingest/workload | Mature SPL search and correlation | Manual rule upkeep, heavy ops | Evidence-linked cases with replayable provenance |
| CrowdStrike Falcon Next-Gen SIEM | https://www.g2.com/compare/crowdstrike-falcon-endpoint-protection-platform-vs-splunk-enterprise-security-vs-microsoft-sentinel | Quote / data volume | 500+ integrations, EDR-native | Newer SIEM; best inside the Falcon estate | Dual-control approval and rollback as first-class concepts |
| TheHive + Cortex (OSS) | https://thehive-project.org | Free / commercial | Case management, analyzers | No governed AI; AGPL | Policy-bounded, validator-gated automation |
| Wazuh (OSS) | https://faresbltagy.gitbook.io/footprintinglabs/build-home-lab-soc-automation/set-up-wazuh-and-thehive-for-threat-detection-and-case-management | Free | Agent plus SIEM | Not a response control plane | We ingest from tools like it rather than replace them |

**Key Takeaways:**
- No SIEM is the consensus best; the choice follows the incumbent stack. STARSHIELD is a control plane that sits above existing tools, and its value is governance: validator, dual control, evidence ledger and rollback.

### 3.7 — Community & Forums

| Thread/Post | URL | Platform | Key Insight |
|-------------|-----|----------|-------------|
| SOC alert volume discussion | https://reddit.sentinel-team.org/posts/1rcpv5v/snapshots/2026-02-28T00%3A40%3A02.980228Z | Reddit (mirror) | Volume is a symptom of tuning; tag alerts by asset criticality |
| 23k triaged / 11 true positives | https://reddit.sentinel-team.org/posts/1q1ed3o/snapshots/2026-01-02T20%3A20%3A12.826626Z | Reddit (mirror) | Upstream prioritisation beats alert-chasing |
| Reducing alert fatigue for SOC analysts | https://www.peerspot.com/articles/reducing-alert-fatigue-for-soc-analysts | PeerSpot | Batch similar alerts; be conservative with auto-close |
| k6 constant throughput | https://community.grafana.com/t/can-k6-support-send-requests-at-constant-throughput/99714 | Grafana Community | Arrival-rate executors give constant RPS |
| Prisma RLS extension caveats | https://www.answeroverflow.com/m/1468652383277023252 | Prisma Discord (mirror) | The official RLS example is not production-ready |

**Key Takeaways:**
- Severity-tabbed feed grouping (already in the template), asset criticality in scoring, and dedup/correlation before case creation.

### 3.8 — APIs & Integrations

| API/Service | URL | Auth Method | Rate Limits | Docs Quality | SDK Available? |
|-------------|-----|-------------|-------------|--------------|----------------|
| HIBP Pwned Passwords range | https://www.troyhunt.com/understanding-have-i-been-pwneds-use-of-sha-1-and-k-anonymity/ | none | generous, CDN-cached | good | no (plain fetch) |
| Connector ingest (inbound, own API) | `/v1/ingest/events` | per-connector HMAC-SHA256 plus key id | per-connector token bucket | ours | n/a |
| Prometheus scrape | https://grafana.com/oss/prometheus/exporters/nodejs-exporter/ | internal network only | n/a | good | prom-client |
| Map/geo tiles | none (self-hosted globe asset; markers projected locally in SVG) | n/a | no third-party map API | n/a | n/a |

**Key Takeaways:**
- There is no external map provider. has_geo's map-proxy rate-limit item is satisfied by having no proxy plus a rate-limited geo API (ADR).
- Real vendor connectors are out of v1 scope. v1 ships a signed generic ingest API and a synthetic connector, badged SYNTHETIC.

### 3.9 — Architecture & Design Patterns

| Pattern | Source | Why It Applies |
|---------|--------|----------------|
| Per-request CSP nonce (Profile N) | https://nextjs.org/docs/15/app/guides/content-security-policy | WEB.md CSP requirement |
| Store-per-request Redux | https://redux-toolkit.js.org/usage/nextjs | Redux is mandated; avoids cross-request leakage |
| Idempotent scheduler plus DB claim | https://docs.bullmq.io/guide/job-schedulers/ | has_scheduled_work items 3, 4, 7 |
| App-layer tenant scoping, optional RLS | https://stratum-docs.pages.dev/guides/prisma-rls/ | Hard isolation for multi_tenant |
| Hash-chained append-only evidence ledger | NIST 800-61r3 evidence handling; ASVS V16 logging | Tamper-evident audit for the auditor persona |
| Propose → Validate → Approve → Execute → Verify | RESEARCH workflows 3–6 | Core governance loop; the validator cannot write proposals |
| Open-model load test | https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate/ | Methodology for the 50k+ target |
| TLS edge proxy (Caddy `tls internal`) | https://dev.to/hugo__df/docker-compose-local-https-with-nginx-or-caddy-5emk | HTTPS-only constraint on local deploy |

**Key Takeaways:**
- The governance loop is the product. Each state transition becomes an evidence-ledger entry with `prev_hash`, so tampering is detectable through a verify endpoint.

## 4. Technology Stack Candidates

| Layer | Option A | Option B | Recommendation | Rationale |
|-------|----------|----------|----------------|-----------|
| Language | TypeScript (Node 22 LTS) | Go | TypeScript | Implied by Express, Prisma and Next; one language end to end |
| Framework (web) | Next.js 15 App Router | Remix | Next.js 15 | Hard constraint |
| Framework (API) | Express 5 | Fastify | Express 5 | Hard constraint; native async error handling |
| State | Redux Toolkit + RTK Query | Zustand | Redux Toolkit | Hard constraint |
| Database | PostgreSQL 16 | — | PostgreSQL 16 | Hard constraint |
| ORM | Prisma 6 | Drizzle | Prisma | Hard constraint |
| Jobs | BullMQ 5 on Redis 7 | pg-boss | BullMQ | Hard constraint |
| Hosting/Infra | Docker Compose plus Caddy TLS edge | Kubernetes | Compose | Single host; k8s deferred |
| Auth | Email/password with server sessions (opaque token, httpOnly Secure cookie) | OIDC SSO | Email/password + sessions; SSO deferred | No IdP named; sessions are revocable |
| Observability | prom-client + Prometheus + Grafana; pino JSON logs | OpenTelemetry | prom-client + pino | Hard constraints |
| Load test | k6 | Artillery | k6 | Arrival-rate executors and dropped-iteration signal |
| CI/CD | GitHub Actions | — | GitHub Actions | Default |

## 5. Risk & Unknowns Register

| ID | Unknown / Risk | Severity | Mitigation | Status |
|----|----------------|----------|------------|--------|
| R1 | 50k+ concurrent can't be proven on one dev host | high | Open-model k6 against the deployed stack at the highest sustainable rate, with per-replica capacity extrapolated with stated headroom. Residual recorded with the exact gap. | open (residual) |
| R2 | Cross-tenant leak through a missed `orgId` filter | high | Repository layer requires `orgId`. A forbidden-pattern invariant bans Prisma model calls outside repositories. Adversarial cross-tenant tests. | mitigated in build |
| R3 | Fabricated AI/QAI conclusions or self-approval | high | Evidence refs on proposals, an independent validator, proposer ≠ approver, no LLM in v1 (deterministic labelled scoring) | resolved by design |
| R4 | High-impact action without dual control | high | Policy marks the action `high`, which requires 2 distinct approvers. The state machine rejects otherwise. | resolved by design |
| R5 | Evidence ledger tampering | high | SHA-256 hash chain, unique(orgId, seq), trigger blocks UPDATE/DELETE, verify endpoint | resolved by design |
| R6 | Scheduled jobs double-fire | med | `UPDATE … WHERE claimed_at IS NULL RETURNING` claim, plus a run-twice test | resolved by design |
| R7 | Illustrative geo/KPIs mistaken for real data | med | `synthetic` flag per row; UI badge | resolved by design |
| R8 | Precise coordinates are PII (has_geo) | med | Stored at 2-decimal precision (~1 km); bounds CHECK and Zod | resolved by design |
| R9 | CSP nonce vs template inline styles | low | Template inline styles ported to CSS modules/tokens | build |
| R10 | Next.js middleware bypass class (CVE-2025-29927) | med | Patched Next pinned; API re-verifies auth; Caddy strips `x-middleware-subrequest` | resolved by design |

### Threat Model (STRIDE)

Trust boundaries:
- Browser ↔ Caddy (TLS) ↔ Next web ↔ Express API ↔ Postgres/Redis
- Connectors ↔ ingest API
- Workers ↔ DB
- Prometheus ↔ `/metrics` (internal network only)

| Threat | Asset / Boundary | Attack | Control |
|--------|------------------|--------|---------|
| Spoofing | Session | Credential stuffing, session theft | Rate limit before hashing, Argon2id, HIBP check, httpOnly Secure SameSite=Lax cookie, rotation on login, CSRF token on mutations |
| Spoofing | Ingest | Forged connector events | HMAC-SHA256 over timestamp and body, 5-minute skew window, per-connector secret, replay cache |
| Tampering | Evidence ledger | Edit or delete history | Hash chain, DB trigger, verify endpoint |
| Tampering | Proposals | Edit after validation | Validation pins the content hash; any edit resets to `draft` |
| Repudiation | Approvals/actions | "I didn't approve that" | Audit log with actor, session, IP, user agent and request id; ledger entries |
| Info disclosure | Tenant data | IDOR / cross-tenant reads | orgId scoping, 404 on foreign ids, adversarial tests |
| Info disclosure | Errors/logs | Stack traces or secrets in logs | RFC 9457 problems without stack traces; pino redaction |
| DoS | API | Floods, expensive queries | Redis-backed rate limits per IP/user/org, page size ≤100, 256 KB body limit, per-connector bucket |
| Elevation | Roles | Analyst approves their own high-impact action | RBAC, proposer ≠ approver, 2 approvers for high impact, JIT scoped credential ≤15 min |
| Elevation | Test trigger endpoint | Abuse in production | Mounted only when not production AND `ENABLE_TEST_TRIGGERS=1`; admin session required |

## 6. Research Gaps

- [x] G1: Exact package versions are verified at install (`npm view`) and recorded in VERSION.md.
- [x] G2: The HIBP range endpoint needs no key. If it is unreachable, fail open and increment a warning metric (ADR).
- [ ] G3: Full Postgres RLS vs app-layer scoping, decided in the ARCHITECTURE alternatives (non-blocking).
- [ ] G4: Real vendor connector schemas (CrowdStrike, Okta, AWS) deferred to v2; generic signed ingest in v1 (non-blocking).
- [ ] G5: Proving 50k concurrent needs multi-host infra, so it is residual R1 (non-blocking, reported honestly).
- [ ] G6: There is no benchmark for the QAI "quantum" claim. v1 uses deterministic, explainable scoring labelled as such.
- [ ] G7: Grafana dashboard provisioning JSON will be authored during the build (non-blocking).

**Blocker?** No. Every open gap has a non-blocking mitigation or a recorded residual.

## 7. Summary & Recommendation

STARSHIELD is buildable on the mandated stack: Next.js 15 + Redux Toolkit, Express 5 REST, Prisma/PostgreSQL, BullMQ/Redis and Prometheus/Grafana. Each risky piece follows a documented vendor pattern:
- CSP nonces in middleware.
- Store-per-request Redux.
- Idempotent BullMQ schedulers plus a DB claim.
- Open-model k6 load tests.

The differentiator over the SIEM incumbents (Sentinel, Splunk ES, Falcon NG-SIEM) and open-source tools (TheHive, Wazuh) is the governance loop: evidence-linked proposals, an independent validator, dual-control approval, a hash-chained evidence ledger and verified rollback. Build effort concentrates there.

Known limitations, all recorded:
- R1: 50k concurrent can't be fully proven on one host (residual).
- Real vendor connectors are deferred.
- No AI/quantum performance claims are made.

**Go / No-Go Decision:** `GO`

**Next Step:** → [ARCHITECTURE.md](./ARCHITECTURE.md)
