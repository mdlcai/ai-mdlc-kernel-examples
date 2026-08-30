# ARCHITECTURE.md — CyberOps

**Project ID:** 39616f58-5fca-4f82-812c-b7c429b456c2 · **Build depth:** comprehensive · **Scale tier:** large (50k+ concurrent)
**Source of truth for architecture.** Behavior is specified in `SPEC.md`; precedence RESEARCH > ARCHITECTURE > SPEC.

---

## §1 System Overview

CyberOps is a unified, multi-tenant, AI-assisted security-operations platform. It turns data from asset discovery, vulnerability scanning, AI-assisted penetration testing, OSINT/threat analysis, and compliance mapping into a single correlated view of organizational risk, with continuous retesting.

**Core problem → solution:** fragmented security tooling → one workflow that discovers, assesses, validates, correlates, prioritizes, and retests across an organization's *authorized* attack surface.

**Five key workflows (SPEC §5):** W1 AI-Powered Automated Penetration Test · W2 Asset Discovery & Vulnerability Assessment · W3 Threat & OSINT Investigation · W4 Compliance & Security Posture Assessment · W5 Finding Remediation & Continuous Retesting.

**Safety boundary (binding, from RESEARCH §5):** the platform runs *authorized* offensive tooling only. It ships real safe recon (DNS/HTTP-fingerprint/TLS) and a template-driven finding engine, and models exploit-validation as state (`unvalidated → validating → validated | failed`) gated behind verified authorization scope + human approval. It does **not** ship a live-exploitation module that fires payloads at arbitrary targets. Enforced by the authorization-scope, SSRF, rate/concurrency, and agent-sandbox invariants (§9).

---

## §2 Architecture Layers & Data Flow

**Topology (monorepo, npm workspaces):**

```
cyberops-2/
├─ apps/
│  ├─ api/        NestJS 11 — REST /v1, WebSocket gateway, BullMQ workers
│  └─ web/        Next.js 16 App Router — marketing + app shell
├─ packages/
│  └─ shared/     Zod schemas + shared DTO/enum types (single contract source)
├─ scripts/       invariant-lint.mjs, smoke-test.sh, seed
├─ docker-compose.yml   api, web, postgres, redis, worker
└─ .env.example
```

**Request path (synchronous):**
`Browser → Next.js (SSR/route handlers, TanStack Query) → API /v1 → JwtAuthGuard → TenantGuard → RolesGuard → ZodValidationPipe → Controller → Service → TenantScopedRepository → Postgres (RLS)`. Every mutation emits an `audit_events` row and (when it must reach a second system) an `outbox_events` row in the **same transaction**.

**Async path (scans, email, webhooks):**
`Service → BullMQ enqueue (Redis) → Worker → ScannerAdapter(s) → Finding stream → persist + WS emit`. Outbox relay worker drains `outbox_events` → search index / email (Resend) / outbound webhooks.

**Realtime path:** `Worker → Socket.io (Redis adapter) → room scan:<id> → Browser` for live scan progress. Origin-checked, JWT-authenticated on connect, per-connection message-rate budget.

**Data flow across workflows:** Asset (W2) → Scan → Finding (CVSS/CWE/severity) → correlated to Asset + Threat intel (W3) → mapped to Compliance controls (W4) → prioritized on the dashboard → remediation + retest (W5); AI pentest (W1) consumes assets/findings and produces attack paths + evidence.

---

## §3 Backend Modules (NestJS, domain-per-module)

| Module | Responsibility | Key entities |
|--------|----------------|--------------|
| `auth` | register/login/logout, JWT httpOnly-cookie session, password reset, MFA-ready | User, Session, PasswordResetToken |
| `tenancy` | orgs, memberships, RBAC roles, tenant-scope chokepoint, RLS context | Organization, Membership, Role |
| `assets` | targets, assets, hosts, services, technologies, **authorization scope** (verified ownership) | Asset, Host, Service, AuthorizationScope |
| `scans` | scan runs, adapter orchestration, BullMQ, WS progress | ScanRun, ScanResult |
| `findings` | findings, CVSS/CWE, severity, correlation, validation state, retest | Finding, FindingValidation, Retest |
| `pentest` | AI pentest engine (multi-agent planner), attack paths, evidence, approval gate | PentestRun, AttackPath, Evidence, ApprovalRequest |
| `osint` | indicators, STIX/TAXII threat feeds, correlation | Indicator, ThreatFeed, Investigation |
| `compliance` | frameworks, controls, mappings, posture, evidence | Framework, Control, ControlMapping, PostureSnapshot |
| `reports` | PDF/JSON/CSV export of findings/compliance/pentest | Report |
| `alerts` | email outbox + outbound webhooks (guaranteed delivery) | PendingEmail, PendingWebhook, WebhookEndpoint |
| `audit` | append-only, hash-chained audit log | AuditEvent |
| `outbox` | transactional-outbox relay to second systems | OutboxEvent |
| `health` | `/api/health` liveness/readiness | — |

**Cross-cutting `common/`:** `tenancy/TenantScopedRepository` (the only place scoped queries are issued), `security/` (throttler config, SSRF `TargetResolver`, CSRF/Origin guard, password verifier w/ HIBP k-anonymity), `audit/AuditInterceptor`, `sentry/`, `pipes/ZodValidationPipe`, `errors/` (typed error envelope, SPEC §7).

---

## §4 Data Layer

- **PostgreSQL 18**, TypeORM 1.1, `DataSource` migrations. PKs = `uuidv7()` (timestamp-ordered, index-friendly at insert scale). Finding payloads/evidence = `jsonb`. Enums as Postgres `enum`.
- **Tenant isolation (defense in depth):** every tenant-owned table carries `tenant_id`; Postgres **RLS** policies scope by a session `app.tenant_id` GUC set per request/job; application-layer `TenantScopedRepository` is the single chokepoint that sets the GUC and filters. Object-level ownership (`assertResourceAccess`) is enforced on every user-owned resource **and its sub-resources** in addition to tenant scope.
- **Idempotency:** `inbound_webhook_events` UNIQUE `(provider, event_id)`; `pending_emails` UNIQUE `(to_address, template_key, business_ref_id)`; scan submission idempotency via client key.
- **Audit chain:** `audit_events(prev_hash, hash)` where `hash = sha256(prev_hash || canonical_json(payload))`; append-only (no update/delete path in the repository).
- **Pooling & scale:** pg pool tuned; PgBouncer (transaction mode) documented for prod; hot tables (`findings`, `scan_results`) indexed on `(tenant_id, severity, created_at)`; read-replica split documented as a cloud-scale step.
- **Migrations reversible** (up/down); seed data for roles, severities, compliance frameworks/controls, and a demo org.

---

## §5 Scanner & Adapter Engine

`ScannerAdapter` interface: `{ id, kind, capabilities, isAvailable(): Promise<boolean>, run(target, opts, ctx): AsyncIterable<RawFinding> }`. The `ScanOrchestrator` (BullMQ worker) selects adapters by scan profile, streams `RawFinding`s → normalizes to `Finding` (CVSS/CWE/severity) → persists → emits WS progress.

**Built-in safe adapters (always available, no external binary):**
- `DnsReconAdapter` — resolves records, enumerates a bounded subdomain wordlist (authorized zones only).
- `HttpFingerprintAdapter` — headers, tech/version fingerprint, missing security headers (HSTS/CSP/X-Frame), cookie flags, exposed `.git`/`.env` probes (safe GET).
- `TlsInspectAdapter` — cert chain, expiry, weak protocol/cipher detection.
- `TemplateEngineAdapter` — Nuclei-style YAML signatures (bundled offline corpus) matched safely against responses; produces CVE/CWE-tagged findings.

**External adapters (used when the binary is present, degrade gracefully):**
- `NucleiAdapter` — spawns `nuclei -jsonl`, parses JSONL lines to findings.
- `TrivyAdapter` — spawns `trivy … --format json` for repo/IaC/SBOM findings.

**SSRF guard (INV):** `TargetResolver.resolveAndValidate(target, scope)` runs before every adapter network action — resolves DNS, rejects private/link-local/reserved ranges, requires the resolved host to fall within an active, verified `AuthorizationScope`, and pins the resolved IP (anti-DNS-rebinding).

---

## §6 AI Pentest Orchestration (multi-agent)

`PentestPlanner` decomposes a run into phases — **recon → analysis → validation → report** — following the VulnBot/AutoPentest/xOffense multi-agent pattern (RESEARCH §3.9). Each phase is an agent role with an allowlisted tool set that calls only the safe scanner adapters. A `Validation` action that would change target state requires an `ApprovalRequest` (human-in-the-loop) and an active authorization scope re-check.

- `LlmProvider` interface (provider-abstracted; Claude Messages API default). **All target-derived text (banners, HTTP bodies, OSINT) is treated as untrusted** and passed as data, never as instructions (prompt-injection mitigation R7).
- **Offline/degraded mode:** when no LLM key is configured (build sandbox), the planner falls back to a deterministic rule-based phase engine so a pentest run is demonstrably functional without external AI — findings and attack paths are produced from the safe adapters.
- Output: `AttackPath` (ordered steps with technique = MITRE ATT&CK id, CWE), `Evidence` (request/response artifacts, encrypted at rest), risk-scored `Finding`s.

---

## §7 Realtime, Async, Notifications, Observability

- **WebSocket:** Socket.io gateway + `@socket.io/redis-adapter` for multi-instance fan-out. Auth: JWT on `connection`, Origin allowlist check, per-connection message-rate budget, channel authorization (a client may only join rooms for its own tenant's scans).
- **Queues:** BullMQ v5 on Redis (`maxmemory-policy=noeviction`, dedicated logical DB). Queues: `scan`, `outbox`, `email`, `webhook`, `pentest`. Idempotent jobs; DLQ after capped backoff.
- **Outbox relay** drains `outbox_events` → search index (Postgres FTS in-cluster for the build; Algolia/OpenSearch documented swap), email, outbound webhooks (HMAC signed at insert, `Idempotency-Key` per send, ≤5 attempts).
- **Email:** Resend v6 via the `email` queue with rate limiting; never inline in a request handler.
- **Errors/telemetry:** Sentry v10 (`@sentry/nestjs` `instrument.ts` first; `@sentry/nextjs`). Structured JSON logs (pino). Prometheus-style `/metrics` + `/api/health`.

---

## §8 Frontend (Next.js 16)

- Route groups: `(marketing)` — landing page **ported verbatim from `DESIGN-TEMPLATE.html`** (nav, hero, features, how-it-works, pricing, globe, footer); `(app)` — authenticated shell (aside sidebar + topbar + glass content), matching the template's app shell; `(auth)` — login/register/reset.
- Data: TanStack Query v5 (`HydrationBoundary` + per-request `QueryClient`), a typed API client from `packages/shared` zod schemas, and a WS client for live scan progress.
- **Design tokens copied verbatim** from the template `:root` into `apps/web/app/globals.css` (`@theme`, Tailwind v4). Dark-first (template) with a light variant + runtime theme toggle persisted + `prefers-color-scheme`.
- Screens enumerated in SPEC §"UI Surface"; every key workflow completable through the UI including terminal steps (review/approve/export).

---

## §9 Architectural Invariants (machine-checkable form in `invariants.json`)

| ID | Rule | Check type | Reference |
|----|------|-----------|-----------|
| INV-1 | All scoped DB access goes through `TenantScopedRepository`; no ad-hoc `getRepository(` outside the tenancy chokepoint | forbidden-pattern | §3 common/tenancy |
| INV-2 | SSRF: the scan orchestrator calls `resolveAndValidate` before dispatching any `adapter.run(` (ADR-012) | boundary-order | §5 SSRF guard |
| INV-3 | Authorization scope is re-validated before any network action of a scan/pentest job (deny-by-default) | manual | §5, R1 |
| INV-4 | The inbound webhook receiver verifies signature before any DB read/write (`inbound-webhook.controller.ts`) | boundary-order | §7, R for webhooks |
| INV-5 | Audit log is append-only — no update/delete/remove on audit events | forbidden-pattern | §4 audit chain, R5 |
| INV-6 | Rate-limit config (Redis-backed throttler) is present and wired | required-file | §7, R6 |
| INV-7 | No hardcoded secrets (AWS keys, `sk_live_`, private keys) in source | forbidden-pattern | Security invariant |
| INV-8 | Inbound webhook idempotency: UNIQUE `(provider, event_id)` | required-unique-constraint | §4 idempotency |
| INV-9 | Email idempotency: UNIQUE `(to_address, template_key, business_ref_id)` | required-unique-constraint | §4, has_email |
| INV-10 | Design token layer exists (`apps/web/app/globals.css`) with template tokens | required-file | §8 |
| INV-11 | Health endpoint present | required-file | §7 health |
| INV-12 | UI coverage: every SPEC §UI screen routes, every non-internal endpoint referenced by UI, every §5 workflow has an e2e test | ui-coverage | SPEC §UI, §5 |
| INV-13 | Object-level authorization (`assertResourceAccess`) enforced on user-owned resources and sub-resources | manual | §4, R9 |
| INV-15 | `assertResourceAccess` is invoked in every owner-bearing service (machine-checked call-site) | required-pattern | §4, R9 (ADR-013) |
| INV-14 | Header buffers sized for accumulated cookies (`--max-http-header-size` / proxy config) | manual | Env & Startup |

Amending any invariant requires updating this section AND `invariants.json` in the same commit, with an ADR in `DECISIONS.md`.

---

## §10 Dependencies (pinned, from RESEARCH §3.1)

**API:** `@nestjs/*`^11, `@nestjs/throttler`^6, `@nestjs/websockets`+`@nestjs/platform-socket.io`, `@socket.io/redis-adapter`, `typeorm`^0.3 *(see ADR-006)*, `pg`, `bullmq`^5, `ioredis`, `zod`, `bcrypt`, `jsonwebtoken`/`@nestjs/jwt`, `resend`^6, `@sentry/nestjs`^10, `pino`.
**Web:** `next`^15 *(see ADR-006)*, `react`^18, `@tanstack/react-query`^5, `tailwindcss`^3 *(see ADR-006)*, shadcn/ui (vendored), `socket.io-client`, `zod`.
**Shared:** `zod` (single contract source).
**Dev/test:** `jest`+`supertest` (API), `vitest`/`@playwright/test` (web e2e), `typescript`, `eslint`.

> **Build-environment version note (ADR-006):** RESEARCH §3.1 verified the bleeding edge (Next 16 / Tailwind v4 / TypeORM 1.1 / NestJS 11). Where a bleeding-edge major introduces install/build instability in the sandbox, the build pins the nearest battle-tested major (Next 15, Tailwind v3, TypeORM 0.3) and logs the substitution as an ADR per the RESEARCH-deliverable guard — the *deliverable* (the stack, the design system, the workflows) is preserved; only the patch/major pin moves. This is a build-environment substitution, not a stack change.

---

## §11 Key Technical Decisions & Alternatives (comprehensive)

- **Monorepo (npm workspaces)** over polyrepo — one contract source (`packages/shared`), atomic cross-cutting changes. *Alt: Nx/Turborepo — deferred; plain workspaces keep the build dependency-light.*
- **REST /v1 + Nest** over GraphQL/tRPC — constraint-fixed; REST maps cleanly to the resource model and to PDF/JSON/CSV exports.
- **RLS + app-layer scope (both)** over app-layer only — defense in depth; RLS is the backstop if a query escapes the chokepoint. *Alt: schema-per-tenant — rejected; 50k-scale makes per-tenant schemas unmanageable.*
- **Transactional outbox** over direct dual-write — the only correct way to keep DB + search/email/webhooks consistent (RESEARCH §3.4). *Alt: 2PC — rejected (not supported across Postgres+Resend+HTTP).*
- **Scanner-adapter interface** over hard-coded scanners — pluggable, testable, degrades offline, safe-by-construction. Mirrors Nuclei's extensibility + hexstrike's tool-bridge (§3.2).
- **Deterministic fallback planner** for AI pentest — guarantees a functional workflow with no external AI in the sandbox while keeping the LLM path first-class.
- **Safety boundary: no live exploitation** — legal/ethical requirement; validation modeled as state + gated adapter (RESEARCH §5).

---

## §12 Threat Model (comprehensive — see RESEARCH §5 for full register)

Trust boundaries: (1) Browser↔API (JWT cookie, CSRF/Origin, Zod validation); (2) API↔Postgres (RLS, parameterized queries, least-privilege role); (3) Worker↔target network (SSRF guard, authorization scope, isolated egress); (4) API↔LLM/target text (untrusted-data handling, prompt-injection mitigation); (5) API↔third-party webhooks (HMAC verify-before-DB). Top risks R1–R8 (authorization scope, SSRF, secret handling, cross-tenant leakage, audit tampering, scan-proxy abuse, prompt injection, artifact handling) each map to an invariant and a mitigation in §9/§5.
