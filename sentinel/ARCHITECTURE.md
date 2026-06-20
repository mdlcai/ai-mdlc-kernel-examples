# ARCHITECTURE.md — Sentinel

> Unified AppSec scanning platform. Orchestrates OSS security scanners in sandboxed containers, normalizes their output into one Finding model, dedups + prioritizes, and surfaces a developer-first dashboard. Source of truth for architecture (precedence: RESEARCH > ARCHITECTURE > SPEC).
>
> Build tier: **small** (`scale: "small — under 1k concurrent"`). Build to tier — neither over- nor under-build.

## 1. System Architecture (layers, modules, data flow)

Sentinel is a multi-tenant monorepo with four runtime processes plus an isolated scanner-execution plane:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Browser (Next.js 16 App Router · Tailwind v4 · TanStack Query v5)   │
│  Dashboard · Findings · Targets · Scans · Settings · Auth            │
└───────────────┬─────────────────────────────────────────────────────┘
                │ HTTPS (cookie session)  REST /v1/*
┌───────────────▼─────────────────────────────────────────────────────┐
│  API service  (Express 5 · Prisma 7)                                 │
│  • Auth (session, RBAC)        • Projects/Targets/Scans CRUD         │
│  • Findings query + triage     • Ownership verification              │
│  • Webhook ingress (GitHub/GitLab)  • Rate limiting · Audit log      │
└───────┬───────────────────────────────────────┬─────────────────────┘
        │ enqueue (BullMQ)                       │ read/write
┌───────▼───────────────┐               ┌────────▼───────────────────┐
│  Redis 8 (BullMQ)     │               │  PostgreSQL 18 (RLS)        │
│  scan queue + flows   │               │  orgs/users/projects/...    │
└───────┬───────────────┘               └────────▲───────────────────┘
        │ consume                                 │ persist findings
┌───────▼──────────────────────────────────────────────────────────────┐
│  Worker service  (Node · BullMQ worker)                               │
│  Orchestrator → fans a Scan into per-scanner child jobs               │
│  For each child: launch sandboxed scanner container, capture output   │
│  Aggregator (parent job): Normalize → Enrich → Dedup → persist        │
└───────┬──────────────────────────────────────────────────────────────┘
        │ docker run (hardened)
┌───────▼──────────────────────────────────────────────────────────────┐
│  Scanner execution plane (untrusted)                                  │
│  Ephemeral containers: trivy · osv-scanner · semgrep · gitleaks ·     │
│  checkov · zap · nuclei · nmap · masscan                              │
│  --network none|restricted --read-only --cap-drop ALL ... (gVisor)    │
└───────────────────────────────────────────────────────────────────────┘
```

**Modules (monorepo packages):**

| Package | Responsibility |
|---|---|
| `apps/web` | Next.js dashboard (UI surface, auth pages, TanStack Query data layer). |
| `apps/api` | Express REST API (`/v1`), auth, RBAC, CRUD, triage, webhook ingress, rate limiting, audit. |
| `apps/worker` | BullMQ worker: scan orchestration, sandboxed scanner execution, normalize/enrich/dedup. |
| `packages/db` | Prisma schema, migrations, generated client, RLS policies, scoped-query wrappers. |
| `packages/core` | Shared domain: Finding model, severity/CVSS, error contract (RFC 9457), validation (zod), audit. |
| `packages/scanners` | Per-scanner adapters (invocation spec + output→Finding normalizers) and the sandbox runner. |
| `packages/config` | Env loading/validation, secrets resolution (env → AWS Secrets Manager), design tokens. |

**Data flow (the spine):** `connect target → verify ownership (live targets) → enqueue Scan → fan-out per-scanner child jobs → sandboxed execution → normalize to Finding → enrich (CVE/CVSS/OSV) → dedup by fingerprint → persist → dashboard triage → status tracking → re-scan`.

## 2. Agent & Tool Orchestration Design

- **Scan request** creates a `Scan` row (`status=queued`) and enqueues a **parent flow job** in BullMQ. Each selected scanner becomes a **child job** under the flow (BullMQ parent-child flows).
- Each child job runs the **SandboxRunner**: resolves the scanner adapter, builds a hardened `docker run` invocation, mounts the target read-only, captures stdout/file, enforces an external `timeout`, and returns the raw scanner output blob.
- When all children settle, the **parent (aggregator) job** runs the pipeline: `Normalize (adapter per scanner) → Enrich → Dedup → persist Findings → update Scan status (completed|failed|partial)`.
- **Scanner network policy** (per adapter): SAST/SCA/secrets/IaC (Semgrep, Trivy, OSV, gitleaks, checkov) run `--network none` over a read-only mounted repo checkout; DAST/network (ZAP, Nuclei, nmap, masscan) run with restricted egress and **only after ownership verification**.
- **Adapter contract** (`packages/scanners`): `{ id, displayName, targetKinds, image, license, network: 'none'|'restricted', argv(target, workdir), parse(raw): Finding[] }`. Adding a scanner = adding one adapter; the orchestrator is scanner-agnostic.

## 3. Design Patterns

- **Adapter pattern** — one normalizer per scanner; the canonical internal shape is SARIF-2.1.0-inspired `Finding`.
- **Strategy pattern** — sandbox network policy + resource profile selected per scanner.
- **Pipeline** — Normalize → Enrich → Dedup → Persist, each a pure, independently testable stage.
- **Repository / scoped-query wrapper** — all tenant-scoped reads/writes go through `withOrgScope(orgId, fn)`; direct Prisma access to scoped tables outside the wrapper is forbidden (INV-1).
- **Chokepoint authorization** — `assertResourceAccess(row, principal)` is the single object-level authz gate (INV-2).
- **Idempotency at the boundary** — webhook handlers verify signature → check idempotency (unique `delivery_id`) → process (INV-7).
- **Outbox/audit** — every privileged mutation writes an `audit_log` row (immutable).

## 4. Dependencies (libraries, APIs, versions)

Pinned at **major.minor** here; exact patches locked in the lockfile (RESEARCH §3.1).

| Dependency | Version | Role |
|---|---|---|
| next | 16.x | Web app (App Router) |
| react / react-dom | 19.x | UI |
| express | 5.x | API + worker HTTP |
| @prisma/client / prisma | 7.x | ORM + migrations |
| pg (PostgreSQL) | 18 | Database |
| bullmq | 5.x | Job queue / flows |
| ioredis | 5.x | Redis client |
| @tanstack/react-query | 5.x | Client server-state |
| tailwindcss | 4.x | Styling |
| zod | 3.x | Validation / env parsing |
| argon2 | 0.40+ | Password hashing (memorized-secret verifier) |
| resend | 6.x | Transactional email |
| @sentry/nextjs · @sentry/node | 10.x | Error reporting |
| @aws-sdk/client-secrets-manager | 3.x | Secrets resolution |
| pino | 9.x | Structured JSON logging |
| vitest · supertest · @playwright/test | latest | Tests (unit/integration/e2e) |

**External APIs:** GitHub App (OAuth + webhooks, `X-Hub-Signature-256`), GitLab (OAuth + webhooks, `X-Gitlab-Token`), Resend, Sentry, AWS Secrets Manager, Slack incoming webhooks.

**Scanner engines (invoked as external containers, never linked):** trivy, osv-scanner, semgrep, gitleaks, checkov, zap, nuclei, nmap (NPSL — isolated), masscan (AGPL-3.0 — isolated).

## 5. Key Technical Decisions & Alternatives

| Decision | Chosen | Alternative considered | Why |
|---|---|---|---|
| Tenant isolation | App-level org scoping (`withOrgScope`) **+** Postgres RLS as defense-in-depth | RLS only | App scoping is testable + portable; RLS catches mistakes. Both, per ADR-001. |
| Scanner sandbox | Docker + hardened flags, gVisor when available | Firecracker microVMs | gVisor is a drop-in OCI runtime; Firecracker heavier ops for a small tier. ADR-002. |
| Canonical finding shape | SARIF-2.1.0-inspired internal `Finding` | Invent bespoke schema | Reuses an open standard; most scanners already emit SARIF. ADR-003. |
| Dedup | Deterministic per-class fingerprint hash | ML/heuristic merge | Predictable, testable, explainable; fits the "noise reduction we can defend" goal. ADR-004. |
| Password hashing | argon2id | bcrypt | NIST-aligned memorized-secret verifier; no silent substitution. ADR-005. |
| Local-dev scanners | A real **demo scanner** (`trivy`-class SCA against committed fixtures) runs end-to-end without Docker; full engine set behind a Docker-present feature flag | Require Docker for any scan | Lets the system do real work + smoke-test on a machine without the Docker scanner images, while keeping the production path. ADR-006. |
| API error contract | RFC 9457 `application/problem+json` with field-level `errors[]` | Ad-hoc `{error}` | Standard, lets the UI bind field errors to controls. ADR-007. |

## 6. Implementation Notes & Risks

- **Demo vs. production scan plane.** The default-on **demo SCA scanner** parses real dependency manifests and matches them against a committed vulnerability fixture set (OSV-shaped), producing genuine Findings through the full normalize→dedup→persist pipeline — so the platform does real work locally. Each production engine (Trivy/Semgrep/etc.) is a sibling adapter gated by `SCANNERS_DOCKER_ENABLED`; the orchestrator, normalizer, dedup, persistence, and UI are identical for both. This keeps the build runnable and smoke-testable while the architecture is honestly the production one.
- **SSRF.** Target URLs validated at DNS-resolution time against an RFC1918 / link-local / metadata denylist before any live scan (R2/TB2).
- **Ownership.** Live targets (domain/IP) require a verified ownership token (DNS TXT or file token) before a DAST/port scan can be enqueued (R3/TB3).
- **Secrets.** Git tokens stored via AWS Secrets Manager reference; in local dev a sealed `.env` fallback is used. Tokens never logged; Sentry `beforeSend` scrubs (R5/TB5).
- **Risk register & full STRIDE threat model:** RESEARCH.md §5 (carried as the architectural threat model for comprehensive depth).
- **Open spikes (RESEARCH §6):** gVisor compatibility for raw-socket scanners; ZAP/nmap adapter fidelity; egress-firewall implementation. None block the small-tier build.

## 7. Scale → Tier Realization (tier: small)

| Subsystem | Small-tier choice |
|---|---|
| DB & pooling | Single Postgres 18, Prisma pool (≤10 conns). |
| Caching | Redis 8 (also the queue); short-TTL dashboard rollups. |
| Async/jobs | BullMQ single worker process, per-tenant concurrency caps. |
| Compute & deploy | Docker Compose (local stack); single web + api + worker instance. |
| Rate limiting | Redis fixed-window per `(ip, route)` and `(ip, email)` on auth. |
| Perf verification | Smoke test asserts API < 200ms p50 on health/list paths. |

Domain signals `has_webhooks` (GitHub/GitLab ingress with idempotency) and `has_dual_write` (Finding persistence + audit_log written atomically) bump the webhook + persistence rows to include explicit idempotency/transaction handling above the bare small-tier default. `has_geo` is noted but not exercised at this tier (logged ADR-008).

## 8. Environment & Deployment

- **Protocol:** HTTPS only (`protocol_support`). Production terminates TLS at a reverse proxy (Caddy/nginx) with HTTP→HTTPS redirect + HSTS; local dev uses mkcert (documented in QUICKSTART.md "Protocol & TLS").
- **Containers:** orchestrated/multi-instance in production; `docker-compose.yml` for the local stack (web, api, worker, postgres, redis, proxy).
- **Config:** all config via env (`.env.example` committed, `.env` gitignored); secrets via AWS Secrets Manager in cloud.
- **Observability:** structured JSON logs (pino), Sentry error reporting, `/api/health` healthcheck, metrics counters for queue latency/scan duration.
- **Backups:** automated daily Postgres snapshots (documented; infra-delegated).

## 9. Architectural Invariants

Every rule below has a machine-checkable or `manual` entry in `invariants.json` (id = cross-reference).

| ID | Rule | File / section | Check type |
|---|---|---|---|
| INV-1 | Tenant-scoped DB access only through `withOrgScope`; no direct Prisma client calls to scoped tables outside the wrapper. | `packages/db/src/index.js`; enforced repo-wide | forbidden-pattern |
| INV-2 | Object-level authorization is centralized in `assertResourceAccess`; routes reading/mutating owned resources call it. | `packages/core/src/authz.js` | required-file |
| INV-3 | Passwords hashed with argon2 only; no bcrypt/bcryptjs/plaintext substitution. | `apps/api/src/auth` | forbidden-pattern |
| INV-4 | Every scoped table carries an `orgId`; the `findings` dedup uniqueness is enforced by a DB unique constraint, not app logic. | `packages/db` migration | required-unique-constraint |
| INV-5 | Secrets are never hardcoded; only read via the config/secrets module. No literal API-key patterns in source. | repo-wide outside `packages/config`, `.env.example` | forbidden-pattern |
| INV-6 | Scanner engines run only through the SandboxRunner with hardened flags; no ad-hoc `docker run`/`child_process` exec of scanners outside it. | `packages/scanners/src/runner.ts` | forbidden-pattern |
| INV-7 | Webhook handlers verify signature before any DB read/write and enforce idempotency on delivery id. | `apps/api/src/webhooks` | boundary-order |
| INV-8 | API errors use the RFC 9457 problem+json helper; no raw `res.status(...).json({error})` ad-hoc shapes for validation failures. | `packages/core/src/errors.js` | required-file |
| INV-9 | The design-token system is the single source of color/type/spacing/radius; no hardcoded hex colors in component source outside the token definition. | `apps/web` token file | forbidden-pattern |
| INV-10 | The invariant-lint runner exists and is wired as `lint:invariants`. | `scripts/invariant-lint.mjs` | required-file |
| INV-11 | SSRF denylist is applied to user-supplied target hosts before any live scan is enqueued. | `packages/core/src/ssrf.js` | required-file |
| INV-12 | Ownership verification gates live (domain/IP) scans — a live Target cannot be scanned while `ownershipVerified=false`. | `apps/api` scan-create path | manual |
| INV-13 | Every SPEC §"UI Surface" screen route resolves and every Key Workflow is completable through the UI. | `apps/web` routes vs SPEC | manual |

**Conventions:** every §9 invariant has a JSON entry; amend §9 + `invariants.json` together with an ADR; never disable an invariant to pass a gate.

## Trade-offs (summary)

- **Demo scanner default-on** trades full-fleet coverage out-of-the-box for a system that runs and proves the end-to-end pipeline anywhere; production engines are one env flag + Docker images away, behind the identical orchestrator.
- **App-scoping + RLS** doubles the isolation cost for defense-in-depth — justified by the multi-tenant `pii_handling` constraint.
- **Deterministic dedup** trades maximal noise reduction for explainability/testability — aligned with the product's "trust the short list" thesis.
