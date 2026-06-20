# RESEARCH.md — Sentinel

build_depth: comprehensive
review_gates: auto
force_research: false
domain: "Security"

## Domain Signals

```yaml
domain_signals: ["has_webhooks", "has_geo", "has_dual_write"]
```

## Product Vision
**Problem:** Modern software teams ship fast, but their security tooling is fragmented and developer-hostile. To cover their code-to-cloud attack surface, a team has to stitch together half a dozen separate tools — one for static code analysis (SAST), another for dependency/CVE scanning (SCA), another for live web testing (DAST), plus port and infrastructure scanners. Each has its own setup, dashboard, output format, and pricing.

This creates three concrete problems:

- **Tool sprawl.** No single place to see "what's wrong across all of my code and infrastructure." Security posture is spread across disconnected products.
- **Alert fatigue.** Each scanner reports independently, so the same underlying issue surfaces multiple times, low-risk findings drown out critical ones, and there's no reachability or context to tell a developer what actually matters.
- **Developer friction.** Most security tools are built for security teams, not the developers who have to fix the issues — so findings arrive late, without context, and without clear remediation.

The result: small and mid-size teams are effectively under-secured not because tools don't exist, but because integrating and operating them is too costly and noisy.

**What we're building:** one platform that runs all five scan types (ports, DAST, SAST, SCA, and more) against a team's repos, domains, and infrastructure, then normalizes, deduplicates, and prioritizes the findings into a single developer-friendly view with clear fixes.

**Who it affects:** Primary — developers and engineering teams at small-to-mid-size companies. Buyer/champion — the CTO, VP Eng, or lead engineer at companies without a CISO. Security engineers and AppSec teams where they exist. Compliance and GRC owners pursuing SOC 2 / ISO 27001. Indirectly: the business/leadership, end users, partners and downstream consumers. Not for (initially): large enterprises with mature security orgs and existing Veracode/Checkmarx contracts.

**Why existing solutions fall short:** Point tools (Semgrep, Snyk, ZAP, nmap) each solve one slice; teams must buy/configure/operate 5–6 and reconcile output manually. Legacy enterprise suites (Veracode, Checkmarx, Black Duck) bundle scan types but are expensive, slow, complex, and per-seat. Open-source DIY stacks are free but offload orchestration, sandboxing, normalization, dedup, and dashboards onto the team. Recurring gaps: no unification, noise-not-signal, built-for-security-not-developers, pricing that scales against you. The wedge: broad coverage + aggressive noise reduction + developer-first experience at a price that works for small-to-mid teams.

**Solution:** A unified AppSec platform (Aikido clone). Users connect a target — a Git repo, a domain, or an IP — and get back a deduped, prioritized list of security findings across five scan types, in one dashboard.

Core scans (wrap existing OSS engines, don't write scanners):
- Port scan → nmap / masscan (target: IP/host)
- DAST (live web vuln scan) → OWASP ZAP + Nuclei (target: URL)
- SAST (static code) → Semgrep (target: repo)
- SCA (dependency CVEs + licenses) → Trivy + OSV-Scanner (target: repo/lockfiles)
- (secrets via gitleaks, IaC via checkov — easy add-ons later)

Architecture (the part that's actually yours):
- API + web app — auth, projects, targets, dashboard (Next.js + Postgres).
- Job queue — enqueue scans (Redis + worker pool).
- Sandboxed runners — each scanner runs in an isolated Docker container with the target mounted; network-restricted for SAST/SCA, network-allowed for port/DAST.
- Normalizer — map every engine's output into one Finding schema: {id, type, severity, title, location, cve, remediation, status}.
- Dedup + triage engine — collapse duplicate findings across scanners, assign severity, (later) reachability analysis. This is the real moat.
- Dashboard — findings list, filters, per-project rollups, fix guidance, status tracking (open/ignored/fixed).

Data model (minimum): User → Org → Project → Target → Scan → Finding

MVP cut (build in this order):
1. Auth + create project + add target
2. One scanner end-to-end (SCA/Trivy is easiest) → store findings → show list
3. Add the other four scanners behind the same job/finding interface
4. Dedup + severity scoring
5. Polish dashboard + remediation text

Key constraints:
- Run scanners as untrusted code → strong container isolation, resource/time limits.
- DAST/port scans hit live systems → require proof of target ownership before scanning (legal must-have).
- Findings normalization is where most engineering effort goes, not the scanners themselves.

## Users & Outcomes
**Key Workflows:**
1. **Onboarding & connect a target** — Sign up → create an org/project → connect a target: link a Git repo (OAuth to GitHub/GitLab), add a domain/URL, or add an IP range. For live targets (DAST, port scan), verify ownership before any scan runs.
2. **Run a scan** — Pick scan types (or run all) → platform enqueues jobs → each scanner runs in its own sandboxed container → results stream back as they finish. Triggers: manual ("Scan now"), on every push/PR (CI integration), or scheduled (nightly).
3. **Triage findings** — Land on the dashboard → see one deduped, severity-sorted list across all scanners → filter by type/severity/project → open a finding to see location, the CVE/rule, why it matters, and the suggested fix.
4. **Remediate & track status** — For each finding: view fix guidance, mark as fixed, ignored (with reason), or snoozed → re-scan confirms resolution → status updates automatically.
5. **CI/CD gate (developer-in-the-loop)** — On each PR, scans run automatically → results post back as a PR check/comment → optionally block merge if new critical findings appear.
6. **Monitor & report** — Dashboard rollups (open criticals, trend over time, per-project posture) → scheduled scans keep coverage continuous → export reports for compliance (SOC 2 / ISO evidence) and leadership.
7. **Manage org & access** — Invite teammates, set roles, group targets by project/team, configure integrations (Git, CI, Slack/email alerts).

The spine: connect → scan (sandboxed) → normalize → dedup/prioritize → fix → track → repeat.

**Success Metrics:** North Star — time-to-fixed-vulnerability. Activation: signup → first completed scan under 10 min. Core value — noise reduction: dedup rate, signal ratio, false-positive rate, % findings actioned. Remediation: MTTR by severity, fix rate, reduction in open criticals 30/60/90. Coverage: scans per project per week, % projects with CI/scheduled scanning. Engagement: weekly active teams, retention/churn. Business: free→paid, CAC/LTV. If only two numbers: dedup/signal ratio and time-to-remediate.

**Non-Goals:** Scanner R&D (we orchestrate OSS engines). Enterprise security suite at launch (no on-prem/air-gapped, no deep RBAC/SSO complexity at launch). Automatic fixing/remediation-as-a-service. SIEM / runtime security. CSPM at launch. Pentest replacement. Offensive/unauthorized scanning (no scanning targets a user can't prove they own). Building our own CI/CD or code host. Custom rule authoring (v1).

## Build Constraints

```yaml
# Infrastructure & Ops
hosting_environment: "cloud-managed (AWS/GCP/Azure)"
protocol_support: "HTTPS only"
ci_cd_required: true
monitoring: "metrics + alerting"
backup_strategy: "automated daily snapshots"
container_strategy: "orchestrated / multi-instance"
error_reporting: "Sentry"

# Data & Storage
database_preference: "PostgreSQL"
data_retention_policy: "findings retained for 90 days; audit logs for 1 year"
pii_handling: ["user email", "git credentials (encrypted)", "target URLs/IPs"]

# Notifications & Messaging
email_service: "Resend"
notification_urgency: "guaranteed delivery"
webhook_targets: ["GitHub", "GitLab"]

# Security & Compliance
security_baseline: ["OWASP Top 10", "SOC2"]
rate_limiting: true
audit_logging: true
secrets_management: "AWS Secrets Manager"

# Frontend
frontend_framework: "Next.js"
css_approach: "Tailwind CSS"
state_management: "TanStack Query"

# Backend
backend_framework: "Express"
api_style: "REST"
api_versioning: "URL path (/v1/)"
orm_preference: "Prisma"

# Performance & Quality
performance_requirements: ["API response < 200ms", "dashboard load < 2s", "scan job queue latency < 5s"]
testing_strategy: "test-after"
logging_format: "structured JSON"

# Scope & Platform
scale: "small — under 1k concurrent"
multi_tenant: true
target_platforms: ["web"]
alert_channels: ["email", "Slack"]
report_formats: ["PDF", "JSON"]

# Pipeline Attribution
mdlc_attribution: "structural"

# Scope Boundaries
hard_dependencies: ["Docker", "Redis", "nmap", "Semgrep", "Trivy", "OWASP ZAP", "Nuclei", "gitleaks", "checkov"]
```

## Design Language

### Archetype
Archetype: saas

### Brand Voice
Professional, confident, efficient. Composed information hierarchy; tables and forms done well; full dark mode.

### Art Direction
The visual system anchors on a composed, information-dense dashboard aesthetic built for security engineers and under-resourced dev teams who need to parse findings fast and act on them with confidence—think the clarity of Linear's issue view or Vercel's deployment dashboard, not the decorative excess of consumer SaaS. The palette is anchored in cool, neutral surfaces paired with a restrained, purposeful accent in #0c7ed4—a confident, cool blue deployed sparingly but decisively on actionable elements. Typography pairs a modern slightly-condensed sans (Space Grotesk / Inter) with a monospace (JetBrains Mono) for code, CVE IDs, and file paths. Strict 8px grid; dense-but-breathing findings tables with subtle row dividers; 24px gutters on modals/detail panels. Motion functional and minimal (200ms ease-out on state changes, 2px hover shift on rows, restrained loading pulse on in-flight scans). Accessibility is the floor: WCAG AA+, 3px accent focus states, color never the sole severity indicator (always paired with icon + text), full keyboard operability. Rejects the default "AI SaaS" template — no purple gradients, no rounded corners beyond ~6px, no decorative illustrations.

### Color System — Light Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #0c7ed4 | Buttons, links, active states |
| Secondary | #6316ca | Accents, badges, highlights |
| Accent | #d36f23 | Callouts, hover states |
| Background | #fafafa | Page background |
| Surface | #f4f5f5 | Cards, elevated containers |
| Text | #161a1d | Headings, body text |
| Text Secondary | #5c6770 | Captions, muted text |
| Success | #1fad53 | Success states, confirmations |
| Warning | #ec9c13 | Warnings, pending states |
| Error | #df2020 | Errors, destructive actions |

### Color System — Dark Mode
| Role | Hex | Usage |
|------|-----|-------|
| Primary | #0c7ed4 | Buttons, links, active states |
| Secondary | #7f30e8 | Accents, badges, highlights |
| Accent | #e28f50 | Callouts, hover states |
| Background | #090a0c | Page background |
| Surface | #121517 | Cards, elevated containers |
| Text | #eaebec | Headings, body text |
| Text Secondary | #838e95 | Captions, muted text |
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
- Theme: Light + Dark — runtime theme toggle following `prefers-color-scheme`, persists explicit choice

### CSS Custom Properties
```css
/* Light mode */
:root {
  --color-primary: #0c7ed4;
  --color-secondary: #6316ca;
  --color-accent: #d36f23;
  --color-background: #fafafa;
  --color-surface: #f4f5f5;
  --color-text: #161a1d;
  --color-text-secondary: #5c6770;
  --color-success: #1fad53;
  --color-warning: #ec9c13;
  --color-error: #df2020;
  --font-heading: 'Space Grotesk', system-ui, sans-serif;
  --font-body: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
  --font-size-base: 14px;
  --radius: 6px;
}

/* Dark mode */
.dark, [data-theme="dark"] {
  --color-primary: #0c7ed4;
  --color-secondary: #7f30e8;
  --color-accent: #e28f50;
  --color-background: #090a0c;
  --color-surface: #121517;
  --color-text: #eaebec;
  --color-text-secondary: #838e95;
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

No design template was uploaded for this project. Fall back to the archetype defaults from `DESIGN.md` Part II plus the tokens in `get_project_config`.

---

# Stage 0 Research (build_depth: comprehensive) — verified June 2026

> Research performed at build time because §3 was absent from the supplied blueprint and `force_research: false` (skip-when-populated did not apply since nothing was populated). All versions / flags / URLs were confirmed against vendor or GitHub sources; unverifiable items are flagged inline.

## 3. Source Categories

### 3.1 Official / Vendor Documentation (verified command/flag/version per entry)

**Scanners** — each entry lists the verified machine-readable-output invocation.

- **nmap 7.99** — [Nmap Reference Guide](https://nmap.org/book/man.html) · [Output options](https://nmap.org/book/man-output.html). No native JSON/SARIF. Use XML: `nmap -oX scan.xml <target>` (or `-oX -` to stdout), convert XML→JSON downstream. **Correction:** `-oJ` does NOT exist for nmap (that flag belongs to masscan).
- **OWASP ZAP 2.17.0** — [ZAP Docs](https://www.zaproxy.org/docs/) · [Docker / baseline scan](https://www.zaproxy.org/docs/docker/baseline-scan/). JSON report via `-J`: `zap-baseline.py -t https://example.com -J report.json`. No built-in SARIF (use a converter/Action). Canonical image: `ghcr.io/zaproxy/zaproxy:stable` (old `owasp/zap2docker-*` names are deprecated).
- **Nuclei 3.9.0** — [Nuclei running docs](https://docs.projectdiscovery.io/tools/nuclei/running). JSONL via `-j`/`-jsonl`: `nuclei -u https://example.com -jsonl`. File export: `-je`/`-jle`. Image: `projectdiscovery/nuclei` (also `ghcr.io/projectdiscovery/nuclei`).
- **Semgrep 1.167.x** — [Semgrep CLI reference](https://docs.semgrep.dev/cli-reference). JSON `--json`; SARIF `--sarif`: `semgrep scan --config auto --sarif -o results.sarif .`. Image: `semgrep/semgrep`.
- **Trivy 0.71.2** — [Trivy docs](https://trivy.dev/latest/docs/) · [reporting formats](https://trivy.dev/latest/docs/configuration/reporting/). `trivy image --format json -o results.json <img>` and `--format sarif` (OASIS 2.1.0-compliant). Image: `aquasec/trivy` (mirror `ghcr.io/aquasecurity/trivy`).
- **OSV-Scanner 2.4.0** — [OSV-Scanner docs](https://google.github.io/osv-scanner/) · [output](https://google.github.io/osv-scanner/output/). `osv-scanner scan --format json ./path` and `--format sarif`. Image: `ghcr.io/google/osv-scanner`.
- **gitleaks 8.30.1** — [gitleaks.org](https://gitleaks.org/) + [repo README](https://github.com/gitleaks/gitleaks). `gitleaks detect --report-format json --report-path results.json` (or `--report-format sarif`); 8.x also exposes `gitleaks git`/`gitleaks dir`. Images: `zricethezav/gitleaks`, `ghcr.io/gitleaks/gitleaks`.
- **checkov 3.3.x** — [checkov.io](https://www.checkov.io/). `checkov -d . -o sarif` (writes `results.sarif`) and `-o json`. Image: `bridgecrew/checkov`.
- **masscan 1.3.2** — [README](https://github.com/robertdavidgraham/masscan) · [man page](https://github.com/robertdavidgraham/masscan/blob/master/doc/masscan.8.markdown). `masscan -p80 10.0.0.0/8 -oJ output.json` (`-oJ` = JSON). **Note:** latest tagged release is **1.3.2 from 2021** — effectively maintenance mode; **no official Docker image**.

**App stack** (current stable + one authoritative doc each):

- [Next.js Docs](https://nextjs.org/docs) — **16.x** (Turbopack default, React Compiler stable; min Node 20+).
- [Express 5.x API](https://expressjs.com/en/5x/api.html) — **5.x** now npm `latest`.
- [Prisma ORM Docs](https://www.prisma.io/docs/orm) — **7.x** (Rust-free TypeScript query engine).
- [PostgreSQL 18 Docs](https://www.postgresql.org/docs/18/) — **18.x**.
- [Redis Docs](https://redis.io/docs/latest/) — **Redis 8** OSS GA.
- [BullMQ Docs](https://docs.bullmq.io/) — **5.x**.
- [TanStack Query v5](https://tanstack.com/query/latest/docs/framework/react/overview) — **v5** (requires React 18+).
- [Tailwind CSS Docs](https://tailwindcss.com/docs) — **v4.x** (CSS-first `@theme`, breaking vs v3).
- [Resend Node SDK](https://resend.com/docs/send-with-nodejs) — **`resend` 6.x**.
- [Sentry for Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/) — **`@sentry/nextjs` 10.x**.
- [AWS SDK v3 — Secrets Manager](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/Package/-aws-sdk-client-secrets-manager/) — `@aws-sdk/client-secrets-manager` v3.

> Patch-level digits on fast-moving npm packages are "latest at search time," not pinned — treat **major.minor** as reliable and pin exact patches in the lockfile at build time.

### 3.2 GitHub Repositories (stars · last-active · license · use)

| Repo | Stars (approx) | Last active | License | What we use it for |
|---|---|---|---|---|
| [nmap/nmap](https://github.com/nmap/nmap) | ~13.1k | 7.99, Mar 2026 | Nmap Public Source License (custom; **no clean SPDX**) | Host/port discovery (XML→JSON) |
| [zaproxy/zaproxy](https://github.com/zaproxy/zaproxy) | ~15.3k | 2.17.0, Dec 2025 | Apache-2.0 | DAST baseline/full scans |
| [projectdiscovery/nuclei](https://github.com/projectdiscovery/nuclei) | ~29.3k | 3.9.0, Jun 2026 | MIT | Template-based vuln/DAST checks |
| [semgrep/semgrep](https://github.com/semgrep/semgrep) | ~15.6k | 1.167.0, Jun 2026 | LGPL-2.1 | SAST |
| [aquasecurity/trivy](https://github.com/aquasecurity/trivy) | ~36.5k | 0.71.2, Jun 2026 | Apache-2.0 | SCA / container / IaC / SBOM |
| [google/osv-scanner](https://github.com/google/osv-scanner) | ~10.5k | 2.4.0, Jun 2026 | Apache-2.0 | SCA via OSV.dev |
| [gitleaks/gitleaks](https://github.com/gitleaks/gitleaks) | ~27.8k | 8.30.1, Mar 2026 | MIT | Secrets detection |
| [bridgecrewio/checkov](https://github.com/bridgecrewio/checkov) | ~8.8k | 3.3.x, Jun 2026 | Apache-2.0 | IaC misconfiguration |
| [robertdavidgraham/masscan](https://github.com/robertdavidgraham/masscan) | ~25.8k | 1.3.2, **2021** | AGPL-3.0 | Fast port sweep |

> **License flags for the clone:** masscan is **AGPL-3.0** (copyleft — keep as an isolated, separately-invoked container, never link into our codebase); nmap's **NPSL is GPL-incompatible** with redistribution constraints — invoke as an external binary only. All others (MIT/Apache/LGPL) are integration-safe.

### 3.3 Video / Tutorials

- [OWASP ZAP — Automation Framework / Docker docs walkthroughs](https://www.zaproxy.org/docs/docker/) (vendor, headless CI patterns).
- [ProjectDiscovery — Nuclei docs & channel](https://docs.projectdiscovery.io/tools/nuclei/running) (templating + CI usage).
- [Trivy CI/CD integration docs](https://trivy.dev/latest/docs/) (GitHub Actions, SARIF upload to code scanning). *(Vendor docs are the authoritative tutorial surface; third-party video quality varies — UNVERIFIED for specific creators.)*

### 3.4 Articles / Blog Posts

- [Aikido — Open Source page](https://www.aikido.dev/open-source) (their engine strategy: Opengrep, Betterleaks, Zen, Safe Chain — clearest articulation of the "wrap OSS scanners" model).
- [OWASP SSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) (informs target-URL validation directly).
- [Docker — Seccomp security profiles](https://docs.docker.com/engine/security/seccomp/) (sandbox hardening rationale).

### 3.5 Standards / RFCs

- **SARIF 2.1.0** (OASIS Standard, Errata 01 Sep 2023) — [OASIS standard page](https://www.oasis-open.org/standard/sarif-v2-1-0/) · [Errata 01 text](https://docs.oasis-open.org/sarif/sarif/v2.1.0/errata01/os/sarif-v2.1.0-errata01-os.html) · [JSON schema](https://github.com/oasis-tcs/sarif-spec/blob/main/sarif-2.1/schema/sarif-schema-2.1.0.json). Our canonical normalization target.
- **CVE Record Format 5.x (latest 5.2.0)** — [cve-schema repo](https://github.com/CVEProject/cve-schema) · [docs](https://cveproject.github.io/cve-schema/).
- **CVSS 4.0** (Nov 2023) — [CVSS v4.0](https://www.first.org/cvss/v4.0/) · [spec](https://www.first.org/cvss/specification-document). Store vector + numeric score; support v3.1 fallback from scanner output.
- **OSV schema 1.7.x** — [ossf/osv-schema](https://github.com/ossf/osv-schema) · [rendered spec](https://ossf.github.io/osv-schema/).
- **OWASP Top 10** — [2021](https://owasp.org/Top10/) and **[2025 — already released](https://owasp.org/Top10/2025/)** (supply-chain debuts at #3, misconfiguration at #2). **Correction:** 2025 edition is live — tag findings against 2025, keep 2021 mapping for back-compat.
- **SOC 2** — [AICPA Trust Services Criteria](https://www.aicpa-cima.com/resources/download/get-description-criteria-for-your-organizations-soc-2-r-report) (Security/Availability/Confidentiality/Processing Integrity/Privacy).
- **Problem Details** — RFC 7807 is **obsoleted by [RFC 9457](https://www.rfc-editor.org/info/rfc9457/)** (Jul 2023); `application/problem+json` wire format unchanged. Use 9457 for our API error contract.

### 3.6 Competing Products

- **[Aikido Security](https://www.aikido.dev) — primary clone target.** Dev-first all-in-one AppSec (SAST/DAST/SCA/secrets/IaC/container/CSPM + autofix PRs); positioned on **consolidation + noise reduction**. Officially orchestrates Opengrep (Semgrep fork), Betterleaks, Zen, Safe Chain; community-reported Trivy/Syft/Checkov additions are **UNVERIFIED** officially.
- **[Snyk](https://snyk.io)** — AI-native enterprise developer-security platform (Code/Open Source/Container/IaC/API DAST); premium tier.
- **[Semgrep AppSec Platform](https://semgrep.dev)** — code-focused (SAST/SCA/secrets), high-accuracy/low-FP; no DAST/container/IaC/CSPM. The engine Aikido's Opengrep forked.
- **[Arnica](https://www.arnica.io)** — ASPM, "pipelineless" scanning, developer-native (Slack/GitHub).
- **[Orca Security](https://orca.security)** — agentless CNAPP; cloud/CSPM center of gravity.

> **Clone takeaway:** Aikido's moat is not novel scanning — it's orchestration of OSS engines behind one UI + dedup/triage + autofix. Our differentiation must live in normalization quality, cross-scanner dedup, and sandbox safety.

### 3.7 Community Threads

- [Nuclei GitHub Issues](https://github.com/projectdiscovery/nuclei/issues) — JSONL schema stability, headless rate-limiting.
- [Trivy GitHub Issues](https://github.com/aquasecurity/trivy/issues) — SARIF field mapping, DB caching in ephemeral containers.
- [gVisor GitHub Issues](https://github.com/google/gvisor/issues) — compatibility gaps running scanners under `runsc`. *(Specific high-signal threads not pinned — UNVERIFIED individual links.)*

### 3.8 APIs / Integrations

- **GitHub** — prefer a **GitHub App** over OAuth App (fine-grained perms, short-lived installation tokens, centralized webhooks): [App vs OAuth](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps) · [App webhooks](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/using-webhooks-with-github-apps) · [Events & payloads](https://docs.github.com/en/webhooks/webhook-events-and-payloads). Events: `push`, `pull_request`, `installation`, `installation_repositories`, `repository`. Verify `X-Hub-Signature-256`.
- **GitLab** — [OAuth 2.0 provider](https://docs.gitlab.com/integration/oauth_provider) · [Project webhooks](https://docs.gitlab.com/api/project_webhooks/) (verify `X-Gitlab-Token`).
- **Resend** — [Send Email API](https://resend.com/docs/api-reference/emails/send-email): `POST /emails`, Bearer `re_…`, `resend.emails.send({from,to,subject,html}) → {data,error}`.
- **Sentry** — [Next.js SDK](https://docs.sentry.io/platforms/javascript/guides/nextjs/) for the web app; [Node SDK](https://docs.sentry.io/platforms/javascript/guides/node/) for the Express worker.
- **AWS Secrets Manager** — [GetSecretValue](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html) · [SecretsManagerClient (SDK v3)](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/client/secrets-manager/).
- **Slack** — [Incoming webhooks](https://docs.slack.dev/messaging/sending-messages-using-incoming-webhooks/) for finding/notification delivery.

### 3.9 Patterns

- **Scanner orchestration** — one ephemeral container per scanner-job; CLI invoked with the verified machine-readable flag (§3.1); stdout/file captured by a worker. Reference: [Aikido engine strategy](https://www.aikido.dev/open-source).
- **Output normalization to a unified Finding schema** — adopt **[SARIF 2.1.0](https://www.oasis-open.org/standard/sarif-v2-1-0/)** as the internal canonical shape; non-SARIF emitters (Nuclei JSONL, gitleaks JSON, nmap XML, masscan JSON) get per-scanner adapters mapping into `Finding { ruleId, severity (CVSS), location, fingerprint, scanner, raw }` with CVE/CVSS 4.0/OSV enrichment.
- **Cross-scanner dedup** — stable per-finding fingerprint: `hash(normalizedRuleId + filePath + lineRange)` for SAST/IaC; `hash(cve + package + version)` for SCA; `hash(host + port + service)` for network. SARIF `partialFingerprints` is the precedent.
- **Container sandboxing** — [docker run security flags](https://docs.docker.com/reference/cli/docker/container/run/): `--network none --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges --pids-limit 256 --memory 512m --cpus 1 --rm`; optional [gVisor `--runtime=runsc`](https://gvisor.dev/docs/user_guide/quick_start/docker/) for untrusted code; external `timeout` wrapper (no native run-timeout flag).
- **Job-queue fan-out** — [BullMQ](https://docs.bullmq.io/) on Redis: a scan request fans out to N per-scanner jobs; a parent/aggregation job normalizes + dedups when all children settle (BullMQ flows).

## 4. Stack Candidates

| Layer | Chosen (locked by constraints) | Alternative | Rationale |
|---|---|---|---|
| Web framework | **Next.js 16 (App Router)** | Remix | App Router + RSC for the dashboard; largest ecosystem, locked. |
| API/worker runtime | **Express 5** | Fastify | Mature, minimal; Express 5 now default. |
| ORM | **Prisma 7** | Drizzle | Type-safe migrations + multi-tenant modeling; Prisma 7 dropped the Rust engine. |
| Primary DB | **PostgreSQL 18** | CockroachDB | Relational findings + RLS for tenant isolation; JSONB for raw scanner output. |
| Queue/cache | **Redis 8 + BullMQ 5** | RabbitMQ / SQS | BullMQ flows model parent-child scan fan-out cleanly. |
| Server state (client) | **TanStack Query v5** | SWR | Caching/invalidation for findings dashboard; locked. |
| Styling | **Tailwind CSS v4** | CSS Modules | Utility-first; v4 CSS-first config (note v3→v4 breaking). |
| Email | **Resend 6** | Postmark | Simple API + React Email; locked. |
| Error reporting | **Sentry 10** | OTel + Grafana | First-class Next.js + Node SDKs; locked. |
| Secrets | **AWS Secrets Manager** | HashiCorp Vault | Encrypted git tokens/scanner creds via GetSecretValue; locked. |
| Sandbox runtime | **Docker + gVisor (runsc)** | Firecracker microVMs | gVisor is a drop-in OCI runtime for untrusted scanner code. |
| Scan SAST/SCA/secrets/IaC | **Semgrep, Trivy, OSV-Scanner, gitleaks, checkov** | Bandit/Grype/TruffleHog | Permissive licenses, SARIF/JSON native, active. |
| Scan DAST/network | **ZAP, Nuclei, nmap, masscan** | Nikto | Cover DAST + port/host; note nmap NPSL & masscan AGPL isolation. |

## 5. Risk Register & Threat Model

### Risk Register

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Container escape from untrusted scanner code/target payloads | Med | Critical | gVisor `runsc`, `--cap-drop ALL`, `no-new-privileges`, read-only root, seccomp default, non-root UID, ephemeral `--rm`. |
| R2 | SSRF via user-supplied target URLs (cloud metadata `169.254.169.254`, RFC1918) | High | High | DNS-resolution-time IP validation + allowlist, egress filtering, `--network none` where the scan doesn't need egress. |
| R3 | Scanning targets the user doesn't own (DAST/port scan abuse → liability) | High | High | Proof-of-ownership (DNS TXT / file token / verified repo OAuth) before any live DAST/port scan; rate caps; audit log. |
| R4 | Multi-tenant data leakage (one tenant sees another's findings) | Med | Critical | Postgres RLS + org/tenant id on every row, scoped queries, per-tenant queue namespacing, no shared scratch volumes. |
| R5 | Git credential / secret theft | Med | Critical | AWS Secrets Manager, short-lived GitHub App installation tokens (not PATs), encrypt at rest, never log tokens. |
| R6 | Resource exhaustion / fork bomb / runaway scan | High | Med | `--pids-limit`, `--memory`, `--cpus`, external `timeout`, per-tenant concurrency quotas in BullMQ. |
| R7 | License contamination (masscan AGPL-3.0, nmap NPSL) | Med | High | Run both ONLY as isolated external-process containers; never link/bundle; document SBOM separation. |
| R8 | Normalization/dedup errors (false merges or dupes erode trust) | Med | Med | Versioned per-scanner adapters, golden-output fixtures, conservative fingerprinting, manual-override UI. |
| R9 | Supply-chain compromise of a scanner image | Low | Critical | Pin image digests, verify signatures, scan our own images with Trivy. |
| R10 | Stale vuln DB → missed CVEs | Med | Med | Refresh Trivy/OSV/Nuclei-templates DBs on schedule; record DB version per scan. |

### Threat Model (STRIDE per trust boundary)

- **TB1 — Untrusted scanner engine ↔ host kernel.** *Tampering:* malicious target output exploits a scanner parser → contained by gVisor syscall interception + read-only FS. *Elevation:* container escape → `--cap-drop ALL`, `no-new-privileges`, non-root, seccomp. *DoS:* resource bomb → cgroup limits + timeout. *Repudiation:* per-job audit record with image digest + DB version.
- **TB2 — User-supplied target URL ↔ our egress (SSRF).** *Spoofing:* DNS rebinding to internal IP → validate at resolution time, re-resolve, pin. *Information disclosure:* hitting metadata/internal services → egress allowlist + RFC1918/link-local denylist at the network layer.
- **TB3 — Proof-of-ownership for live DAST/port scans.** *Spoofing:* user claims a target they don't own → require a verified ownership token before active scanning. *Repudiation:* immutable audit trail of who scanned what, when, with what authorization artifact.
- **TB4 — Multi-tenant isolation.** *Information disclosure/Tampering:* cross-tenant read/write → Postgres RLS + mandatory org scoping, separate queue namespaces, no shared container volumes.
- **TB5 — Encrypted git credentials.** *Information disclosure:* token leak → AWS Secrets Manager, short-lived installation tokens, secrets scrubbed from logs/Sentry breadcrumbs, least-privilege IAM.
- **TB6 — Public API surface.** *Tampering/Repudiation:* forged webhooks → verify signatures (`X-Hub-Signature-256`, `X-Gitlab-Token`); RFC 9457 problem+json errors without leaking internals.

## 6. Research Gaps

- **gVisor scanner compatibility (spike).** Raw-socket tools (nmap/masscan) may not run correctly under `runsc`; likely need `runc` + tighter net/cap policy instead. **UNVERIFIED** which scanners break.
- **ZAP/nmap SARIF gap.** ZAP (JSON only) and nmap (XML only) need custom adapters; field-fidelity unproven — prototype normalizers against golden fixtures.
- **masscan maintenance risk.** Last release 2021, AGPL — decide inclusion vs. leaning on nmap.
- **Proof-of-ownership UX/flow** needs a concrete design spike (token types, re-verification cadence, org-level vs repo-level).
- **Dedup fingerprinting** across heterogeneous scanners needs empirical tuning — no cross-scanner standard beyond SARIF intra-tool fingerprints.
- **Egress firewall implementation** (iptables vs VPC/proxy) is environment-dependent — needs an infra decision.

## 7. Summary & Go/No-Go

The technology surface is real, current, and viable. All nine scanners exist and are actively maintained (except masscan, frozen at 1.3.2/2021) and emit machine-readable output via verified flags — with two caveats baked into the architecture: nmap has **no JSON** (XML→convert) and ZAP has **no built-in SARIF** (JSON + adapter). SARIF 2.1.0 is the canonical normalization target, with CVSS 4.0, CVE 5.x, and OSV 1.7.x for enrichment. The locked app stack (Next.js 16, Express 5, Prisma 7, PostgreSQL 18, Redis 8/BullMQ 5, Tailwind v4, TanStack v5, Resend, Sentry, AWS Secrets Manager) is current and well-documented; watch-items are the Tailwind v3→v4 and TanStack v4→v5 breaking boundaries and Prisma 7's engine change.

The hard problems are not "can we run scanners" but safety and trust: sandboxing untrusted scanner code (gVisor + full cap-drop + cgroup + ephemeral containers), SSRF defense on user-supplied targets, proof-of-ownership before live scans (a legal/abuse necessity), and multi-tenant isolation (Postgres RLS + scoped queues + secrets in AWS Secrets Manager). These are solved patterns with authoritative references; several need spike work, but none block proceeding.

Competitively, Aikido proves the model; our differentiation must be normalization/dedup quality and sandbox rigor since the scanning tech is commodity OSS. License hygiene (isolate AGPL masscan and NPSL nmap as external-process containers) is a real but manageable constraint.

**Go/No-Go: GO** — every mandatory research section is verified against vendor sources, the stack is current, and all major risks have known mitigations. Open items are scoped Stage-1 spikes, not blockers.
