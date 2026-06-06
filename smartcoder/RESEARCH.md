# RESEARCH.md — SmarCoders

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Medical coding"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_dual_write"]
```

## Product Vision
**Problem:** Medical coding teams need a simple way to practice, manage, and audit chart coding workflows without using real patient data. Existing tools are often tied to production EHR or billing systems, making them too complex, risky, or expensive for demo, training, and workflow validation. This application solves that by using synthetic medical encounters in a coding queue where coders assign codes and supervisors audit accuracy, feedback, and compliance.

**Who it affects:** **Medical Coders** reviewing 20-40 charts daily who struggle with documentation gaps, ambiguous diagnoses, and code selection confidence—need faster chart navigation, real-time coding guidance, and clear audit feedback to reduce rework and improve first-pass accuracy.

**Coding Supervisors and Auditors** managing 5-15 coders who face manual review bottlenecks, inconsistent feedback delivery, and difficulty tracking which coders need targeted coaching—need streamlined audit workflows, standardized feedback templates, and visibility into error patterns by coder and code type.

**Coding Managers** accountable for team productivity and quality metrics who lack integrated dashboards showing queue depth, coder throughput, audit completion rates, and accuracy trends—need real-time performance visibility, benchmarking against targets, and drill-down capability to identify bottlenecks or training gaps.

**Healthcare Organizations** (hospitals, clinics, billing services) conducting coder onboarding or validating new coding processes who cannot use production data or live EHR access for training—need a safe sandbox with realistic encounters, configurable difficulty levels, and audit trails demonstrating competency before production deployment.

**Medical Coding Education Programs** teaching ICD-10, CPT, and HCPCS who need diverse, clinically accurate patient scenarios with built-in audit and feedback mechanisms—need scenario libraries, automated grading, and performance analytics to assess student mastery and identify curriculum gaps.

**Compliance and Revenue Cycle Teams** ensuring coding accuracy supports quality reporting and reimbursement claims who need audit evidence, error root-cause analysis, and compliance documentation—need audit trails, compliance reporting, and integration points for downstream billing validation.

**Why existing solutions fall short:** Most medical coding platforms are designed for production healthcare environments and are tightly integrated with EHR, billing, and revenue cycle systems. These solutions are often expensive, complex to configure, and require access to protected health information (PHI), making them unsuitable for training, demonstrations, workflow testing, or process improvement initiatives. Many platforms also focus primarily on coding productivity while providing limited visibility into coder performance, audit workflows, feedback loops, and quality metrics. Organizations looking to train coders, validate coding processes, or demonstrate coding operations often lack a simple, standalone solution that combines realistic chart coding, supervisor auditing, performance tracking, and reporting using safe synthetic patient data.

**Solution:** SmarCoders is a medical coding workflow platform that enables coders to practice and code realistic synthetic patient encounters, then supervisors to validate accuracy and provide feedback—all without real patient data or production system access. The platform automates end-to-end ICD-10 coding and QA workflows while tracking productivity, accuracy, and audit metrics in a single system.
Load demo data into the system for testing.  It can be found here https://synthea.mitre.org/downloads.


## Users & Outcomes
**Key Workflows:**
### 1. Chart Intake & Queue Management

Medical Coding Manager imports 500 synthetic patient encounters from Synthea into the platform. System automatically distributes charts across 8 coders by specialty (cardiology, orthopedics, internal medicine) and workload balance. Manager views real-time queue dashboard showing 127 unassigned, 89 in-progress, 34 pending audit, and 250 completed charts. Outcome: 100% chart assignment within 2 hours; queue visibility prevents bottlenecks and enables workload rebalancing.

### 2. Medical Chart Coding

Medical Coder opens assigned chart, reviews synthetic patient history, labs, imaging, and encounter note. Coder assigns 3–7 ICD-10-CM diagnosis codes and 1–3 procedure codes, documents clinical rationale for each code selection, and flags documentation gaps (e.g., "severity of pneumonia not specified"). Coder submits chart for audit. Outcome: Chart coded within 8 minutes; rationale recorded; first-pass audit pass rate improves from 72% to 85% within 30 days.

### 3. Coding Validation & Quality Checks

System validates submitted codes against ICD-10 format rules, checks for required secondary diagnoses (e.g., comorbidities for DRG accuracy), verifies code sequencing (principal diagnosis first), and flags unsupported code combinations. System alerts coder to 2 formatting errors and 1 missing comorbidity before audit. Outcome: 94% of charts pass automated validation; rework cycles reduced by 40%; coder confidence in code selection increases.

### 4. Supervisor Audit & Feedback

Coding Supervisor reviews 15 coded charts daily against source documentation. Supervisor approves 12 charts, modifies 2 codes (incorrect sequencing), and returns 1 chart to coder with structured feedback: "Principal diagnosis correct; add secondary code for hypertension per documentation on page 2." Supervisor records audit score (87% accuracy), coaching note, and assigns targeted training module. Outcome: Coder receives feedback within 4 hours; audit findings logged for trend analysis; supervisor completes 15 audits in 90 minutes (vs. 180 minutes manual review).

**Success Metrics:**
**Coding Accuracy** – Percentage of submitted charts with zero code modifications required during audit; target ≥90% by month 3; tracked per coder, specialty, and code type (diagnosis vs. procedure).

**Audit Pass Rate** – Percentage of charts approved without rework on first supervisor review; target ≥85% within 30 days; measured daily with 7-day rolling average to identify trend shifts.

**Coder Productivity** – Charts coded per 8-hour shift; baseline 6–8 charts/shift; target 8–10 charts/shift by week 4; tracked individually and by specialty cohort to detect workload imbalance.

**Coding Turnaround Time** – Median time from chart assignment to coder submission; target ≤10 minutes per chart; 95th percentile ≤15 minutes; flagged if exceeds 20 minutes to identify bottlenecks.

**Audit Turnaround Time** – Median time from coding submission to supervisor approval or return; target ≤4 hours; 95th percentile ≤6 hours; SLA breach logged if exceeds 8 hours.

**Queue Health** – Real-time counts: unassigned charts (target <5% of daily intake), in-progress (target 60–80% of active coder count), pending audit (target <10% of daily volume), overdue (target 0; alert if any chart exceeds 24-hour SLA).

**Error Rate** – Frequency per 100 charts by category: diagnosis sequencing (target <3 errors/100), missing secondary codes (target <2/100), procedure code format (target <1/100), documentation gaps flagged (target >80% identified by coder); tracked weekly by coder and specialty.

**Rework Rate** – Percentage of charts returned to coders for correction after audit; target ≤15% by week 2, ≤10% by week 6; measured per coder to identify coaching needs.

**User Adoption** – Percentage of assigned coders and auditors logging in ≥4 days/week; target ≥95% by week 2; daily active user count tracked to monitor engagement drop-off.

**Training


## Build Constraints

```yaml
# Infrastructure & Ops
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "Datadog"
backup_strategy: "daily automated snapshots"
container_strategy: "Docker"
error_reporting: "Sentry"

# Data & Storage
database_preference: "PostgreSQL"
pii_handling: ["synthetic patient data", "audit trails for PHI-equivalent handling"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"

# Security & Compliance
security_baseline: ["HIPAA"]
rate_limiting: true
audit_logging: true
secrets_management: "AWS Secrets Manager"

# Frontend
frontend_framework: "Next.js"
ui_component_library: "shadcn/ui"
state_management: "TanStack Query"

# Backend
backend_framework: "NestJS"
api_style: "REST"
realtime_needed: true
background_jobs: "Bull"

# Performance & Quality
performance_requirements: ["chart load < 2s", "audit submission < 500ms", "queue dashboard refresh < 1s"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "medium — 1k-50k"
target_platforms: ["web"]
alert_channels: ["in-app", "email"]
report_formats: ["PDF", "CSV"]
```

## Design Language

### Archetype
Archetype: saas

This product's design archetype is **saas**. Read `DESIGN.md` Part II §`saas` (fetched alongside BUILD.md from the MDLC kernel) and treat its Layout Doctrine, Density, Type System, Color & Atmosphere, Motion Budget, and Signature Components as binding requirements, and its Good-vs-Avoid list as the acceptance rubric. The token tables below are the resolved starting palette; an explicit brand override outranks them per the `DESIGN.md` precedence list. The Universal Excellence floor (`DESIGN.md` Part I) applies on top regardless of archetype.

### Brand Voice
The platform should feel like a modern coding operations system built for healthcare professionals. Communication should be clear, concise, and compliance-focused, avoiding unnecessary technical jargon while maintaining coding accuracy and professionalism.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #2563eb | Buttons, links, active states |
| Secondary | #7c3aed | Accents, badges, highlights |
| Accent | #06b6d4 | Callouts, hover states |
| Background | #ffffff | Page background |
| Surface | #f8fafc | Cards, elevated containers |
| Text | #0f172a | Headings, body text |
| Text Secondary | #64748b | Captions, muted text |
| Success | #16a34a | Success states, confirmations |
| Warning | #d97706 | Warnings, pending states |
| Error | #dc2626 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #3b82f6 | Buttons, links, active states |
| Secondary | #8b5cf6 | Accents, badges, highlights |
| Accent | #22d3ee | Callouts, hover states |
| Background | #0f172a | Page background |
| Surface | #1e293b | Cards, elevated containers |
| Text | #f1f5f9 | Headings, body text |
| Text Secondary | #94a3b8 | Captions, muted text |
| Success | #22c55e | Success states, confirmations |
| Warning | #f59e0b | Warnings, pending states |
| Error | #ef4444 | Errors, destructive actions |

### Typography
- Heading: Manrope (600/700 weight)
- Body: Inter (400/500 weight)
- Mono: JetBrains Mono (code, pre, kbd)
- Base size: 16px, scale ratio: 1.25
- Scale: 10.2 / 12.8 / 16 / 20 / 25 / 31.3 / 39.1px

### Layout
- Pattern: Topnav + Content
- Max width: 1280px
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
    primary: '#2563eb',
    secondary: '#7c3aed',
    accent: '#06b6d4',
    background: '#ffffff',
    surface: '#f8fafc',
    foreground: '#0f172a',
    muted: '#64748b',
    success: '#16a34a',
    warning: '#d97706',
    destructive: '#dc2626',
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
  --color-primary: #2563eb;
  --color-secondary: #7c3aed;
  --color-accent: #06b6d4;
  --color-background: #ffffff;
  --color-surface: #f8fafc;
  --color-text: #0f172a;
  --color-text-secondary: #64748b;
  --color-success: #16a34a;
  --color-warning: #d97706;
  --color-error: #dc2626;
  --font-heading: 'Manrope', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 16px;
  --radius: 8px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #3b82f6;
  --color-secondary: #8b5cf6;
  --color-accent: #22d3ee;
  --color-background: #0f172a;
  --color-surface: #1e293b;
  --color-text: #f1f5f9;
  --color-text-secondary: #94a3b8;
  --color-success: #22c55e;
  --color-warning: #f59e0b;
  --color-error: #ef4444;
}
```

### Accessibility
- WCAG AA compliance
- Lighthouse target: 90+
- Responsive breakpoints: 640 / 768 / 1024 / 1280px
- Reduced motion: Standard animations

## Design Template

An HTML design template was uploaded with this project (built by the customer
in Claude Design / artifacts). Fetch it at build time via the
`get_design_template` MCP tool — that is the source of truth, not this section.
A copy is saved at the project root as `DESIGN-TEMPLATE.html`.

---

## 3. Source Categories

### 3.1 Official / Vendor Documentation

| # | Vendor / Spec | Official Doc URL | Current Stable (verified Jun 2026) | Verified fact (command / config / API) |
|---|---|---|---|---|
| 1 | **Synthea** | https://github.com/synthetichealth/synthea · https://synthea.mitre.org/downloads | `master-branch-latest` (built 2026-05-28); Apache-2.0 | Generate cohort: `./run_synthea -p 1000 Massachusetts`. Seeded city: `./run_synthea -s 987 Washington Seattle`. CSV export is **off by default** — set `exporter.csv.export = true` in `src/main/resources/synthea.properties` (FHIR R4 is default under `output/fhir/`). |
| 2 | **ICD-10-CM (CMS/CDC)** | https://www.cms.gov/medicare/coding-billing/icd-10-codes · https://www.cdc.gov/nchs/icd/icd-10-cm/files.html | FY2026 set, eff. **2025-10-01 → 2026-09-30** | FY2026 added **487 new** codes, 38 revisions, 28 deletions. Guidelines PDF: https://www.cms.gov/files/document/fy-2026-icd-10-cm-coding-guidelines.pdf. ICD-10-CM is public domain. |
| 3 | **CPT / HCPCS** | CPT (AMA): https://www.ama-assn.org/practice-management/cpt · HCPCS (CMS): https://www.cms.gov/medicare/coding-billing/healthcare-common-procedure-system | CPT 2026 & HCPCS eff. **2026-01-01** | CPT = 5-digit AMA-maintained, **license-restricted** (cannot be redistributed freely). HCPCS Level II (CMS) covers supplies/DME. Design must treat CPT descriptors as licensed content. |
| 4 | **Next.js** | https://nextjs.org/docs/app | 15.x stable | App Router v15: `fetch`, GET Route Handlers, client navigations **no longer cached by default**. Upgrade codemod: `npx @next/codemod@canary upgrade latest`. |
| 5 | **NestJS** | https://docs.nestjs.com/ | 11.1.x | v11 makes **SWC the default compiler**. Queues guide: https://docs.nestjs.com/techniques/queues |
| 6 | **shadcn/ui** | https://ui.shadcn.com/docs | CLI **v4** (Mar 2026) | Config in `components.json`; add via `npx shadcn@latest add button`. |
| 7 | **TanStack Query** | https://tanstack.com/query/latest | v5.x | v5 **removed `onError`/`onSuccess`** from `useQuery`; single-object signature `useQuery({ queryKey, queryFn, enabled, select })`; status `pending \| error \| success`. |
| 8 | **BullMQ** (`@nestjs/bullmq`) | https://docs.bullmq.io/guide/nestjs · https://docs.nestjs.com/techniques/queues | `@nestjs/bullmq` 11.0.x | Processor: extend `WorkerHost`, implement `async process(job)`; events via `@OnWorkerEvent('completed')`. Requires Redis. (Bull is maintenance-only — use BullMQ.) |
| 9 | **PostgreSQL** | https://www.postgresql.org/docs/17/ | **17.x** | v17 adds `JSON_TABLE()` projecting JSON into relational rows; useful for FHIR payloads. |
| 10 | **Resend** | https://resend.com/docs/send-with-nodejs | current SDK | `const resend = new Resend(apiKey); await resend.emails.send({ from, to, subject, html })`. Delivery/bounce webhooks (verify signature). |
| 11 | **Datadog APM** (`dd-trace`) | https://docs.datadoghq.com/tracing/trace_collection/dd_libraries/nodejs/ | `dd-trace` **5.x** (Node ≥18) | Must be the **first import**: tracer file runs `require('dd-trace').init()` before app code. |
| 12 | **Sentry** | https://docs.sentry.io/platforms/javascript/guides/nestjs/ · https://docs.sentry.io/platforms/javascript/guides/nextjs/ | `@sentry/nestjs` & `@sentry/nextjs` **10.x** | NestJS: register `SentryGlobalFilter` + `@SentryExceptionCaptured()`; init before module imports. |
| 13 | **AWS Secrets Manager** | https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html | v2 API | `GetSecretValue` returns `AWSCURRENT` when no `VersionId`/`VersionStage`. Needs `secretsmanager:GetSecretValue` (+ `kms:Decrypt` for CMK). |

Synthea download confirmed: **https://synthea.mitre.org/downloads** hosts pre-generated sample datasets (FHIR/CSV/C-CDA). To generate: clone `synthetichealth/synthea`, run `./run_synthea -p <N> <State> [City]`. Default export is HL7 **FHIR R4**; CSV opt-in via `synthea.properties`.

### 3.2 GitHub Repositories

| Repo | URL | Stars (approx) | Last active | License |
|---|---|---|---|---|
| `synthetichealth/synthea` | https://github.com/synthetichealth/synthea | ~3.2k | May 2026 | Apache-2.0 |
| `nestjs/nest` | https://github.com/nestjs/nest | ~70k | Active (11.x) | MIT |
| `shadcn-ui/ui` | https://github.com/shadcn-ui/ui | ~85k+ | Active (CLI v4) | MIT |
| `TanStack/query` | https://github.com/TanStack/query | ~45k | Active (v5) | MIT |
| `taskforcesh/bullmq` | https://github.com/taskforcesh/bullmq | ~7k+ | Active | MIT |
| `vercel/next.js` | https://github.com/vercel/next.js | ~135k | Active (15.x/16) | MIT |
| `getsentry/sentry-javascript` | https://github.com/getsentry/sentry-javascript | ~8k | Active (10.x) | MIT |

All MIT/Apache-2.0 — clean for commercial use.

### 3.3 Videos / Tutorials

1. **NestJS official Queues guide** — `WorkerHost` processor + BullMQ + Redis: https://docs.nestjs.com/techniques/queues
2. **"Level Up Your NestJS App with BullMQ Queues, DLQs & Bull Board"** (DEV): https://dev.to/ronak_navadia/level-up-your-nestjs-app-with-bullmq-queues-dlqs-bull-board-5hnn
3. **"Worker Queues in NestJS: Scaling with BullMQ and Redis"** (Medium): https://medium.com/@bhagyarana80/worker-queues-in-nestjs-scaling-with-bullmq-and-redis-without-breaking-your-api-903fdcff43df

### 3.4 Articles / Blog Posts

1. **"Solving the Dual-Write Problem with the Transactional Outbox Pattern"** (Medium) — maps to `has_dual_write`: https://yashodharanawaka.medium.com/solving-the-dual-write-problem-with-the-transactional-outbox-pattern-e74a79fed0ef
2. **AWS Prescriptive Guidance — Transactional outbox pattern** (Postgres-applicable): https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html
3. **"2026 ICD-10-CM Coding Updates"** (Oncology Practice Management): https://www.oncpracticemanagement.com/issues/2025/september-2025-vol-15-no-9/2026-icd-10-cm-coding-updates-what-you-need-to-know

### 3.5 Standards / RFCs

- **ICD-10-CM FY2026 Official Guidelines** (CMS/CDC, eff. 2025-10-01): https://www.cms.gov/files/document/fy-2026-icd-10-cm-coding-guidelines.pdf
- **CPT (AMA)** — HIPAA-designated procedure code set (license-restricted); **HCPCS Level II** (CMS), HIPAA-adopted under 45 CFR 162.1002.
- **HIPAA Security Rule — 45 CFR Part 164, Subparts A & C**: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164. **2024-12-27 NPRM** proposes mandatory MFA + encryption at rest/in transit (no more "addressable"): https://www.federalregister.gov/documents/2025/01/06/2024-30983/hipaa-security-rule-to-strengthen-the-cybersecurity-of-electronic-protected-health-information
- **HL7 FHIR R4 (v4.0.1)** — Synthea default export: https://hl7.org/fhir/R4/
- **HTTP rate-limit signaling** — IETF draft `draft-ietf-httpapi-ratelimit-headers` (`RateLimit`/`RateLimit-Policy`): https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/. `429 Too Many Requests` = **RFC 6585**; `Retry-After` per RFC 9110.
- **Webhook signing** — de-facto HMAC-SHA256 over raw payload in a signature header (Stripe/Resend/GitHub style).

### 3.6 Competing Products

1. **Solventum 360 Encompass** (ex-3M; includes **3M CodeFinder**) — enterprise CAC + CDI + auditing with NLU code suggestion for production coding: https://www.solventum.com/en-us/home/health-information-technology/solutions/360-encompass-cac/
2. **Optum Encoder Pro / Expert** — knowledge-based encoder with compliance edits, CAC + CDI for HIM departments.
3. **TruCode (TruBridge), Nuance Clintegrity, AAPC training, Find-A-Code/SuperCoder** — encoder/lookup and coder-education tools.

**Gap SmarCoders fills:** incumbents are *production* encoders for live revenue-cycle coding on **real PHI**. None is a purpose-built **training + QA workflow sandbox on synthetic data** with built-in audit trails, coder-assessment queues, and reviewer feedback loops. SmarCoders targets onboarding/competency-training and QA scoring — complements rather than competes head-on.

### 3.7 Community Threads

- **microservices.io — "Pattern: Transactional outbox"** (Chris Richardson): https://microservices.io/patterns/data/transactional-outbox.html
- **NLM Clinical Tables change log** (ICD-10-CM API versioning): https://clinicaltables.nlm.nih.gov/CHANGELOG_for_users.html

### 3.8 APIs / Integrations

- **Synthea output** — HL7 **FHIR R4** transaction bundles (default `output/fhir/*.json`); optional **CSV** (`patients.csv`, `conditions.csv`, `procedures.csv`, …); also C-CDA, bulk-FHIR ndjson.
- **NLM Clinical Tables — ICD-10-CM search API** (free, no key). Base: **`https://clinicaltables.nlm.nih.gov/api/icd10cm/v3/search`**. Example: `…/search?sf=code,name&terms=tuberc&df=code,name`. Response is a 4-element array `[totalCount, codesArray, extraDataHash|null, displayRows]`. Docs: https://clinicaltables.nlm.nih.gov/apidoc/icd10cm/v3/doc.html
- **Resend** — `resend.emails.send({...})` + delivery webhooks (verify signature).
- **Datadog** — `dd-trace` (init before app), log/trace correlation.
- **Sentry** — `@sentry/nestjs` (`SentryGlobalFilter`) + `@sentry/nextjs`.
- **AWS Secrets Manager** — `GetSecretValue` at runtime; `RotateSecret` + Lambda for rotation.

### 3.9 Patterns

- **Outbox pattern (`has_dual_write`)** — write business row + event row in **one Postgres transaction**; a relay (polling worker) publishes from the outbox. **At-least-once**, so consumers must be idempotent.
- **Webhook verify-first + idempotency (`has_webhooks`)** — verify HMAC-SHA256 over the **raw** body *before* parsing; reject on mismatch; dedupe on a stored delivery ID in a `processed_events` table; reject stale timestamps.
- **Queue / worker (BullMQ)** — NestJS `WorkerHost` processors on Redis; retries/backoff, dead-letter queue, `@OnWorkerEvent` for status; Bull-Board for ops.
- **RBAC** — role + tenant claims on the principal; NestJS guards enforce role *and* tenant on every chart-access route (default-deny).
- **Audit logging** — append-only audit table for every PHI-equivalent access/mutation (who/what/when/why); hash-chained for tamper-evidence.
- **Real-time queue dashboard** — **SSE** for one-way job-status streaming; WebSocket only if bidirectional control is needed.

---

## 4. Stack Candidates (fixed — confirmed)

| Layer | Choice | Rationale | Pin |
|---|---|---|---|
| Runtime | **Node.js LTS** | dd-trace & NestJS 11 require ≥18; LTS for security support | **Node 22 LTS** |
| Frontend | **Next.js (App Router)** | SSR/RSC + mature ecosystem; pairs with shadcn | **15.x** |
| UI | **shadcn/ui** | Owned, copy-in accessible components | CLI **v4** |
| Data fetching | **TanStack Query v5** | Cache/invalidation for queue & chart views | **5.x** |
| Backend | **NestJS (REST)** | DI/guards → clean RBAC + audit interceptors | **11.x** |
| Jobs | **BullMQ** (`@nestjs/bullmq`) | DLQ + events for dashboard | **11.x** + Redis |
| DB | **PostgreSQL** | Transactions enable outbox; JSON for FHIR | **17.x** |
| Email | **Resend** | Clean DX, React templates, delivery webhooks | current SDK |
| Monitoring | **Datadog** (`dd-trace`) | APM + log/trace correlation | **5.x** |
| Errors | **Sentry** | NestJS + Next.js first-class SDKs | **10.x** |
| Secrets | **AWS Secrets Manager** | Managed rotation; KMS-backed | v2 API |
| Packaging | **Docker**; **HTTPS only**; rate limiting; audit logging | HIPAA baseline | — |

No blockers; Node 22 satisfies every SDK floor.

---

## 5. Risk Register + Threat Model (comprehensive)

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| CPT descriptors are AMA-licensed; redistribution breaches license | Med | High (legal) | Treat CPT text as licensed; do NOT seed it into the repo/dataset; gate behind license or use code-only references. ICD-10-CM (public domain) + HCPCS (CMS) are fine. |
| Dual-write inconsistency (DB committed, event lost) | Med | High | **Transactional outbox** + idempotent consumers; monitor relay lag in Datadog. |
| Webhook spoofing / replay | Med | High | Verify HMAC on raw body before parse; dedupe by delivery ID; reject stale timestamps. |
| Audit-log tampering / repudiation | Low | High | Append-only table, restricted write role, hash-chaining; alert on gaps. |
| Broken access control (chart leakage across tenant/role) | Med | High | Default-deny guards enforcing tenant **and** role on every route; isolation integration tests. |
| Auth user-enumeration / brute force | Med | Med | Uniform error messages, generic "invalid credentials," rate limiting before hash compare + lockout, audit failed logins. |
| Secret leakage (API keys, DB creds) | Low | High | AWS Secrets Manager + KMS; no secrets in images; rotation. |
| Synthetic data mistaken-for real PHI | Low | High | Hard policy + UI banner: data is Synthea-synthetic; block any real-PHI ingestion path. |
| ICD-10/CPT version drift breaks scoring | Med | Med | Pin code-set version per scenario; surface effective dates; annual refresh (Oct 1 / Jan 1). |
| Redis outage stalls queue/workers | Low | Med | Managed Redis HA; BullMQ retries/backoff + DLQ; dashboard alerting. |

### STRIDE-style boundary list

- **Auth boundary** — *Spoofing/Repudiation*: MFA-ready, uniform anti-enumeration responses (shape + timing), rate limit before hash compare + lockout, audit every auth event.
- **Chart access (tenant/role isolation)** — *Elevation of Privilege / Info Disclosure*: default-deny guards checking tenant + role on every chart route; row scoping; isolation tests.
- **Audit log** — *Tampering/Repudiation*: append-only, restricted writer role, hash-chain, gap-detection alerting.
- **Webhook ingress** — *Spoofing/Tampering/Replay*: verify-first HMAC on raw body, idempotency store, timestamp/replay rejection.
- **Dual-write / outbox** — *Tampering/DoS (inconsistency)*: single-transaction outbox, at-least-once + idempotent consumers, relay-lag monitoring.
- **Secrets / encryption-at-rest** — *Info Disclosure*: Secrets Manager + KMS, TLS-only transit, encrypted Postgres volume, least-privilege IAM.

---

## 6. Research Gaps

1. **CPT licensing** — exact terms/cost for using CPT descriptors in a training product are unresolved. ICD-10-CM/HCPCS are free; CPT may require an AMA license or code-only display. Needs legal confirmation before any CPT content ships. **Build decision:** ship ICD-10-CM coding only in v1; CPT/procedure codes referenced by code (no licensed descriptors seeded).
2. **HIPAA NPRM finalization** — the 2024 Security Rule NPRM (mandatory MFA + encryption) is not yet final as of June 2026; build to its stricter posture as forward-compatible baseline.
3. **SSE vs WebSocket for queue dashboard** — SSE chosen for one-way job-status streaming; revisit only if interactive push controls are required.
4. **Code-set refresh cadence** — operational process for annual ICD-10-CM (Oct 1) updates and scenario version pinning to be formalized post-v1.

---

## 7. Summary + GO / NO-GO

SmarCoders is a synthetic-data (Synthea FHIR R4 / CSV) medical-coding training and QA platform on a fully mainstream, currently-versioned stack: Next.js 15 + shadcn/ui v4 + TanStack Query v5; NestJS 11 + BullMQ on Redis; PostgreSQL 17; Resend, Datadog, Sentry, AWS Secrets Manager; Docker, HTTPS, rate limiting, audit logging. Every vendor doc, version, and key API signature was verified against primary sources. The two domain signals are covered by established patterns — **transactional outbox** (`has_dual_write`) and **verify-first HMAC + idempotency** (`has_webhooks`). Because data is synthetic, true PHI risk is low; we nonetheless treat audit-trail integrity, access control, encryption-at-rest, anti-enumeration, and rate-limiting as hard HIPAA boundaries. The only non-engineering risk is **CPT licensing** (AMA-restricted) — a content/legal matter resolved for v1 by shipping ICD-10-CM coding only. No technical blockers.

**Verdict: GO.** Carry-forward action items: (1) resolve CPT licensing before shipping CPT descriptors; (2) pin the code-set fiscal year per training scenario.
