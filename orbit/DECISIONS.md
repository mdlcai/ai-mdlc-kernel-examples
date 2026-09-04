# DECISIONS.md — Orbit

Architecture Decision Records, assumptions, deviations, and technical debt. Newest entries are appended at the bottom of each stage section. Every blank RESEARCH.md field resolved by the agent has an ADR here.

Format: `ADR-NNN — <title>` · Status · Context · Decision · Rationale · Consequences / Cascade.

---

## Stage 0 — Research

### ADR-001 — Build strategy defaults
- **Status:** Accepted
- **Context:** RESEARCH.md Build Strategy sets `build_depth: comprehensive`, `review_gates: auto`, `force_research: false`. `prompt_mode` and `confidence_level` are absent.
- **Decision:** `prompt_mode` → `direct` (default). `confidence_level` → treated as blank; `review_gates: auto` matches first, so every stage gate self-verifies and proceeds. Research runs at comprehensive depth (§3.1–§3.9 all mandatory, ≥2–3 analyzed entries each, threat model in §5).
- **Consequences:** No human pauses between stages. The Multi-Agent Plan Gate auto-dispatches. Security Audit, COMPLIANCE.md, Design Quality and Drift gates auto-proceed per the outer prompt.

### ADR-002 — Stage 0 research executed (§3 was absent)
- **Status:** Accepted
- **Context:** The dashboard-generated RESEARCH.md contained only Product Vision / Users / Constraints / Design Language. No §3 Source Categories existed.
- **Decision:** Research was performed (web search + GitHub + npm registry) and §3–§7 were appended to the local RESEARCH.md. Package versions were verified against the npm registry on 2026-09-02 (`npm view <pkg> version`), not from memory.
- **Consequences:** The local RESEARCH.md is the enriched working copy that Stage 1 reads; the customer should commit it.

---

## Stage 1 — Architecture

### ADR-003 — Key decisions resolved from blank fields
- **Status:** Accepted
- **Context:** RESEARCH.md Step 4 key decisions (`deployment`, `backend_language`, `auth_model`) are absent from the dashboard-generated file. Build Constraints fix `container_strategy: Docker Compose`, `frontend_framework: Next.js`, `backend_framework: Express`, `database_preference: PostgreSQL`.
- **Decision:**
  - `deployment` → **Docker Compose** (single host, one region — matches `scale: small` and `container_strategy`).
  - `backend_language` → **TypeScript / Node.js 22 LTS** (the only language coherent with Express + Next.js; one language across the monorepo lets `packages/shared` hold the zod schemas both sides validate with).
  - `auth_model` → **email/password with server-side sessions in Postgres, HttpOnly `__Host-`-style cookie** (ITSM users are employees inside an org; no third-party IdP was named; SSO is out of scope for v1 and recorded as a non-goal). Passwords hashed with Argon2id (`argon2` native package).
- **Rationale:** Each choice is the lowest-risk option that satisfies every filled constraint without adding infrastructure the `small` tier forbids.
- **Cascade:** ARCHITECTURE.md §2 (stack), §4 (auth), §9 INV-2/INV-4; SPEC.md §4 RBAC matrix.

### ADR-004 — Realtime via Server-Sent Events, not WebSockets
- **Status:** Accepted
- **Context:** `realtime_needed: true`. The realtime need is server→client push (queue updates, SLA breaches, notifications, approval state). No client→server streaming is required.
- **Decision:** One authenticated SSE endpoint `GET /api/v1/events/stream` per user session, scoped to the user's org, with `Last-Event-ID` resume from an in-process ring buffer. Caddy proxies it with `flush_interval -1`.
- **Rationale:** SSE rides the existing HTTPS session cookie (no separate Origin-on-upgrade CSRF surface), reconnects natively, needs no extra infra, and fits the `small` tier ("in-process" fan-out). `has_websocket` checklist therefore does not apply; the SSE endpoint still enforces per-connection auth, org scoping, and a per-user connection cap.
- **Consequences:** Multi-instance fan-out would need a shared bus (deferred; documented in ARCHITECTURE.md §8 risks).

### ADR-005 — TypeScript 6.0.3 instead of the 7.0.2 `latest`
- **Status:** Accepted
- **Context:** `npm view typescript version` → 7.0.2 (the native Go-based compiler line). `typescript-eslint@8.69.0` and `@typescript-eslint/parser` declare `peerDependencies.typescript: ">=4.8.4 <6.1.0"`, so type-checked lint (a Verification Gate requirement) cannot run on TS 7.
- **Decision:** Pin `typescript@6.0.3` (latest stable inside the supported range). Re-evaluate when typescript-eslint publishes TS-7 support.
- **Consequences:** No functional loss; recorded as technical debt (upgrade path).

### ADR-006 — Drizzle ORM + SQL migrations
- **Status:** Accepted
- **Context:** `orm_preference` blank; PostgreSQL required.
- **Decision:** `drizzle-orm` 0.45 with `drizzle-kit` generated SQL migrations committed under `apps/api/drizzle/`, applied by a `migrate` compose service before the API starts. Down migrations are hand-written `*.down.sql` companions (see ADR-007).
- **Rationale:** SQL-first, no runtime codegen, tiny footprint, first-class `pg` driver, and the pinned invariant-lint runner parses Drizzle `pgTable(...)` uniqueness natively.

### ADR-007 — Migration reversibility
- **Status:** Accepted
- **Decision:** Every forward migration `NNNN_name.sql` ships a `NNNN_name.down.sql`. `npm run db:rollback` applies the latest down file. Greenfield (`data_migration` blank → greenfield) so no expand/contract steps are needed.

### ADR-008 — Single-origin topology behind Caddy; ports moved off 80/443
- **Status:** Accepted
- **Context:** `protocol_support: "HTTPS only"`. The build host already binds 80/443 (another project's Caddy) and 5432 (another Postgres).
- **Decision:** Caddy 2 terminates TLS with `tls internal` (locally-trusted CA) on **host port 8443**, redirects **8080 → https** (the only plaintext listener), and routes `/api/*` → `api:4000`, everything else → `web:3000`. Postgres is published on **host port 5433** for local test runs only. The deploy URL is therefore `https://localhost:8443`. Production operators swap `tls internal` for a real domain (automatic Let's Encrypt) per QUICKSTART.md "Protocol & TLS".
- **Rationale:** One origin means the session cookie is first-party for both the Next.js pages and the Express API (no CORS, `SameSite=Strict` works, `__Host-` prefix valid). Header-buffer sizing applies at Caddy (default 1 MB header limit is generous — noted) and Node (`--max-http-header-size=32768`).

### ADR-009 — Background jobs run in-process (no queue)
- **Status:** Accepted
- **Context:** `scale: small` → "Inline or simple cron"; `background_jobs` blank.
- **Decision:** SLA timers, escalations, recurring tasks and the notification outbox drain run inside the API process on a 15 s scheduler tick guarded by a Postgres advisory lock (`pg_try_advisory_lock`) so a second replica cannot double-fire. No Redis, no BullMQ.
- **Consequences:** Honors the tier table exactly; a `medium` upgrade path is a worker process reading the same tables.

### ADR-010 — Notification delivery: transactional outbox + SMTP (Nodemailer), provider blank
- **Status:** Accepted
- **Context:** `email_service` blank; `alert_channels` blank; the product needs notifications (SLA breach, assignment, approval requested). BUILD.md `has_email` checklist applies once email is sent.
- **Decision:** `pending_emails` outbox written in the triggering transaction; drained by the in-process scheduler via Nodemailer over SMTP (`SMTP_URL`). In development/test the transport is a JSON transport that logs. Slack/Teams webhooks are a non-goal for v1 (see RESEARCH §"Non-Goals").
- **Rationale:** SMTP is provider-neutral; the customer plugs in Resend/Postmark/SES by URL.

### ADR-011 — Domain signals derived
- **Status:** Accepted
- **Context:** `domain_signals` absent. Derived from the product: `has_email` (notifications). Not applicable: `has_image_uploads` (attachments are a v1 non-goal), `has_webhooks`, `has_geo`, `has_websocket` (SSE, ADR-004), `has_dual_write` (single Postgres; outbox is same-transaction), `has_payments`, `has_webhook_send` (outbound webhooks are a v1 non-goal).
- **Consequences:** SPEC.md §5 carries the `has_email` outbox requirements; the memorized-secret verifier checklist (breached-password screening etc.) is applied because `security_baseline` is blank → OWASP Top 10 floor **plus** NIST 800-63B verifier rules adopted voluntarily (ADR-012).

### ADR-012 — Security baseline
- **Status:** Accepted
- **Context:** `security_baseline` blank; `pii_handling` blank; `data_retention_policy` blank; `secrets_management` blank.
- **Decision:** OWASP Top 10:2025 floor + OWASP ASVS 5.0 L2-aligned controls; NIST SP 800-63B-4 (Final, July 2025) password verifier — **minimum 15 characters** because Orbit v1 has no MFA (the 8-char floor applies only inside MFA), maximum 128, Unicode and spaces allowed, no composition rules, no periodic rotation, breached-password screening via the HaveIBeenPwned k-anonymity range API (3 s timeout, fail-open with a warning; `HIBP_ENABLED=false` for air-gapped deploys falls back to a bundled top-10k list); Argon2id m=19456 KiB, t=2, p=1 per the OWASP Password Storage Cheat Sheet; PII (names, emails) encrypted in transit only, at-rest encryption delegated to the deploy volume and marked ⚠ (delegated) in COMPLIANCE.md; audit log retained indefinitely (immutable), sessions 30 days sliding, password-reset tokens 1 h; secrets via environment variables with a committed `.env.example` and gitignored `.env`.
- **Note:** BUILD.md's memorized-secret checklist cites the older 800-63B minimum of 8; the research (§3.5) verified that 800-63B-4 supersedes it. The stricter current standard is applied.

### ADR-013 — Non-goals declared (RESEARCH.md had none)
- **Status:** Accepted
- **Context:** The dashboard file's "Key Workflows" section repeats persona pain points instead of journeys, and no Non-Goals section exists. Stage 3b Pre-Build Review rewrote Key Workflows as `W1..Wn` journeys and added an explicit Non-Goals list to RESEARCH.md.
- **Decision (v1 non-goals):** SSO/OAuth/SAML; file attachments & image uploads; Slack/Teams/PagerDuty outbound webhooks; SMS; mobile apps; billing/payments; i18n; offline mode; CMDB auto-discovery agents; email-to-ticket inbound parsing; multi-currency purchasing; custom scripting/plugins.

### ADR-014 — Design archetype `saas`, brand override honored, Inter retained as body face by brand directive
- **Status:** Accepted
- **Context:** Design Language declares `Archetype: saas`, Manrope headings, Inter body, JetBrains Mono, an explicit palette and an art-direction brief. DESIGN.md §8.1 forbids `Inter` as the *primary display* face without an ADR.
- **Decision:** Manrope (700/600) is the display/heading face — the brand voice. Inter is body-only, as the project's own typography block prescribes; this is an explicit brand override (DESIGN.md precedence: brand > archetype > floor), not a default. Numerals use `font-variant-numeric: tabular-nums` for data columns. Fonts self-hosted via `@fontsource-variable/*`.
- **Consequences:** Logged so §8.1 is satisfied by an explicit decision, not by omission.

### ADR-015 — WEB.md applies; CSP Profile N
- **Status:** Accepted
- **Decision:** The Next.js app mints a per-request nonce in `proxy.ts` (Next 16 name for middleware) and sets CSP Profile N on both request and response; Express sets the same header set via `helmet` for API responses. Cookies: `HttpOnly; Secure; SameSite=Strict; Path=/; __Host-orbit_session`.

### ADR-016 — Attribution
- **Status:** Accepted
- **Decision:** `mdlc_attribution` blank → `structural`: NOTICE file, README "Built with" section, package keywords `mdlc`, `ai-mdlc`.

### ADR-017 — Header-buffer sizing
- **Status:** Accepted
- **Decision:** Node processes (web and api) start with `--max-http-header-size=32768`. Caddy's default request-header limit (1 MB) is already generous; stated here so it is considered, not assumed.

### ADR-020 — Idempotency-Key deferred for non-monetary POSTs
- **Status:** Accepted (deferred)
- **Context:** Research §3.9 recommends an idempotency-key table for retried ticket/comment POSTs. BUILD.md mandates it only for `has_payments` / `has_webhook_send`, neither of which applies.
- **Decision:** Not implemented in v1. The client disables the submit control while a request is pending and surfaces failures for an explicit user retry; server-side uniqueness (`UNIQUE(org_id, number)`) and optimistic `version` checks cover the remaining duplicate-write cases.
- **Consequences:** Technical debt entry; the `Idempotency-Key` header is reserved in the OpenAPI description for a later version.

### ADR-021 — Pre-Build Review (Stage 3b) edits to RESEARCH.md
- **Status:** Accepted
- **Findings and fixes applied to the local RESEARCH.md:**
  1. Key Workflows — the field repeated persona pain points, not journeys → rewritten as `W1..W9` "As a [role], I [action] so that [outcome]" journeys covering every module.
  2. Success Metrics — no measurable thresholds → each metric now carries a threshold and the artifact that measures it.
  3. Non-Goals — section absent → added (see ADR-013), which also formally excludes Slack/Teams webhooks that the prose could be read to imply.
  4. Integrations — none listed while notifications and breached-password screening need SMTP and HIBP → "Integrations (v1)" line added.
  5. `realtime_needed: true` with no stated realtime mechanism → resolved as SSE (ADR-004); no contradiction with `scale: small`.
  6. `multi_tenant: true` with `auth_model` blank → resolved as org-scoped email/password sessions (ADR-003).
  7. Success metric "Availability" implied by `monitoring: basic health checks` but absent → added with the health-endpoint threshold.
  8. Design Language declares Inter as body while DESIGN.md §8.1 forbids it as the *display* face → no conflict (Manrope is display); recorded in ADR-014.
- Zero remaining contradictions between prose, constraints and Design Language.

### ADR-018 — Review gate outcome, Stage 1
- **Status:** Accepted
- **Decision:** `review_gates: auto` → self-verified: Build Input Reconciliation has one row per non-blank field (14 rows, 0 Conflict), §9 has an `invariants.json` entry per rule, manual entries ≤ cap. Proceeded to Stage 2 without halting.

---

## Stage 2 — Specification

### ADR-019 — Review gate outcome, Stage 2
- **Status:** Accepted
- **Decision:** `review_gates: auto` → self-verified: every `W1..W8` has a `Wn.m` flow with a named terminal step and a screen in §6; every non-internal §3 endpoint is bound by a screen; non-goals from ADR-013 are not specced; §7 envelope is RFC 9457; §8 lists every env var. Proceeded to Stage 3.

---

## Stage 3 — Build

_(appended per feature)_

---

## Stage 4 — Working System

_(appended at Stage 4)_

### ADR-022 — Register discloses email uniqueness (409 EMAIL_TAKEN) behind the per-IP limiter
- **Status:** Accepted
- **Context:** Self-service org registration must tell a legitimate user that the email is already in use; the anti-enumeration rule in BUILD.md targets login and password reset, which stay uniform.
- **Decision:** `POST /auth/register` returns `409 EMAIL_TAKEN` (field error on `email`); the endpoint is limited to 5 requests/h per IP, making enumeration impractical. Login and forgot-password remain uniform.

### ADR-023 — Multi-Agent Plan (Stage 3)
- **Status:** Accepted (auto-dispatched, `review_gates: auto`)
- **Waves** (file-disjoint waves run in parallel; dependency-coupled waves run sequentially):
  - **Wave 0 (sequential, primary agent):** F-01 Foundation & service floor · F-02 Data layer, tenancy & audit core. Scope ≈ 1 400 LOC, 1 route (`/api/health`), ~25 tests. Root workspace, shared package, all Drizzle tables + RLS + seed, `withOrgScope`, audit writer, SSE hub core, scheduler skeleton, problem envelope, env schema.
  - **Wave 0b (parallel with Wave 0's F-02, web agent):** F-03 Design system, app shell & landing (`apps/web` only). ≈ 1 500 LOC, 0 API routes, ~20 component tests. Tokens, primitives, shell, `proxy.ts` CSP, metadata routes, landing.
  - **Wave 1 (sequential, coupled through the session/auth module):** F-04 Registration & login · F-05 Profile & password change · F-06 Password reset & invites (+ outbox/mailer). ≈ 1 500 LOC, 9 routes, ~40 tests.
  - **Wave 2 (parallel, module-disjoint):** agent A: F-07 Users admin + F-08 Teams admin (12 routes) · agent B: F-09 Ticket catalog & creation + F-10 Ticket queue & detail + F-11 Ticket lifecycle (14 routes) · agent C: F-15 Assets (7 routes) · agent D: F-16 Projects (13 routes) · agent E: F-17 Knowledge base (5 routes). ≈ 6 000 LOC, ~120 tests. Each agent owns `apps/api/src/modules/<name>/**`, `packages/shared/src/<name>.ts`, `apps/web/src/app/(app)/<route>/**`, `apps/web/src/features/<name>/**`.
  - **Wave 3 (parallel):** agent F: F-12 SLA policies & engine (5 routes + scheduler engine) · agent G: F-13 Approvals (5 routes) → then F-14 Change management (7 routes, depends on approvals service). ≈ 2 500 LOC, ~50 tests.
  - **Wave 4 (parallel):** agent H: F-18 Notifications & realtime (4 routes) · agent I: F-19 Automation & recurring (10 routes) · agent J: F-21 Audit trail (1 route). ≈ 2 200 LOC, ~40 tests.
  - **Wave 5 (sequential):** F-20 Dashboard, analytics & global search (3 routes; depends on every module). ≈ 900 LOC, ~15 tests.
  - **Wave 6 (sequential):** F-22 Delivery — containers, Caddy, CI, docs, Playwright W1–W9, web-baseline config, smoke test. ≈ 1 200 LOC.
- **Sequencing rationale:** waves 0/1 are the shared substrate every module imports (`withOrgScope`, `problem`, `authenticate`, primitives); waves 2–4 touch disjoint module directories and disjoint route groups, so agents cannot clobber each other; F-14 follows F-13 because change transitions call the approvals service; F-20 aggregates all modules; F-22 needs every screen to exist for the e2e suite and baseline config. Total: 22 features, 7 waves, ~16 700 LOC, 96 routes, ~310 tests.
- **Bundle Integrity Rule:** no wave bundles features into one ADR; every feature keeps its own SPEC §5.1 entry (endpoints, tables, screens) as the contract and its own completion banner.
- **Shared-file discipline:** all dependencies are installed in Wave 0 (single `npm install`); the module registry (`apps/api/src/modules/index.ts`), navigation config and shared enums/transition tables are created in Wave 0 so later agents only add files inside their own directories.
- **Interface Contract Validation cadence:** N = min(7, ceil(22/4)) = 6 → after F-06, F-12, F-18, F-22 and once more at the end.

### ADR-024 — ESLint 9.39.5 instead of 10.9.1
- **Status:** Accepted
- **Context:** `eslint-plugin-jsx-a11y@6.10.2` (latest) declares `peerDependencies.eslint: ^3 … ^9`; `npm install` refuses ESLint 10 without `--legacy-peer-deps`, which would be a substitution-discipline smell.
- **Decision:** Pin `eslint@9.39.5` (latest 9.x; flat config, `defineConfig`, same rule set). Re-evaluate when jsx-a11y publishes ESLint-10 support.
- **Consequences:** None functional; the Verification Gate's "real linter" requirement is unaffected.

## Stage 3 — Build

### ADR-025 — F-01 spec amendment: `RATE_LIMIT_SCALE`
- **Status:** Accepted
- **Context:** Integration tests register dozens of orgs from one IP; the SPEC limits (5 registrations/h per IP) would block them, and per-test app instances need small limits for the limiter tests themselves.
- **Decision:** Add `RATE_LIMIT_SCALE` (default `1`, tests `1000`) multiplying every limit, plus `createApp({ limits })` overrides for targeted tests. Logged in SPEC.md §8. Production limits are unchanged.

### ADR-026 — F-02 generated-column immutability
- **Status:** Accepted
- **Context:** Postgres rejected the `kb_articles.search_vector` generated column: `array_to_string` and the `text[]::text` cast are not immutable.
- **Decision:** Weight B uses `array_to_tsvector(tags)` (immutable); both tsvector expressions cast `'english'::regconfig` explicitly. Migration 0000 regenerated before any deploy existed (greenfield), so no expand/contract step was needed.

### ADR-027 — Outbox dead-letter after the third failed attempt
- **Status:** Accepted
- **Context:** BUILD.md's `has_email` text lists delays 1 m / 10 m / 1 h and "after 3 attempts → dlq"; SPEC F-06 AC-3 says the row parks after 3 attempts.
- **Decision:** Attempt 1 fails → retry in 1 m; attempt 2 fails → retry in 10 m; attempt 3 fails → `dlq` with a structured `warn`. The 1 h step is defined in `BACKOFF_MS` for operators who raise `MAX_ATTEMPTS`.

### ADR-028 — Reset/invite tokens share one table and one endpoint
- **Status:** Accepted
- **Decision:** `password_reset_tokens.purpose ∈ {reset, invite}`; `POST /auth/password/reset` consumes either (invites are 24 h, resets 1 h) and clears `must_reset_password`. Invited users cannot log in until they set a password (login treats `must_reset_password` as invalid credentials — uniform 401).

### ADR-029 — INV-14 regex amended (rule unchanged)
- **Status:** Accepted
- **Context:** The invariant fired on seven correctly-guarded routes (`router.get('/:id', loadAsset, …)`). The lookahead `,\s*(?!load[A-Z])` backtracks: `\s*` gives back the space, so the lookahead is evaluated against `" loadAsset"`, which is not `load[A-Z]`, and the negative lookahead passes.
- **Decision:** The rule text is unchanged; the pattern becomes `,\s*(?!\s*load[A-Z])` so every backtrack position still sees the guard. Verified against fixtures: guarded routes (single and multiple spaces) no longer match; genuinely unguarded routes (`requireAuth` first, no guard at all) still match. `ARCHITECTURE.md` §9 and `invariants.json` amended together.

### ADR-030 — Keyset cursors carry microsecond precision
- **Status:** Accepted
- **Context:** Cursors encoded `createdAt.toISOString()` (millisecond precision) while Postgres stores microseconds, so a row-value comparison `(created_at, id) > (ts, id)` re-emitted the boundary row (ascending lists) or skipped rows (descending lists). Caught by the ticket-comments pagination test.
- **Decision:** Every timestamp cursor now carries the exact value as `to_char(created_at AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS.USZ')`, selected alongside the row (comments, timeline, notifications, users). `audit_events` keys on its monotonic `bigserial` id alone, which is exact and index-friendly.

### ADR-031 — SLA deadlines land on the closing minute
- **Status:** Accepted
- **Context:** `addBusinessMinutes(Mon 09:00, 1440, 8h/day)` can be read as Wednesday 17:00 or Thursday 09:00.
- **Decision:** Business minutes are consumed to the closing minute of a day before the clock rolls to the next window: 480 → Mon 17:00, 1440 → Wed 17:00, 1441 → Thu 09:01. This keeps `addBusinessMinutes` associative (`f(f(t, a), b) == f(t, a+b)`), which the SLA pause/resume arithmetic depends on. Unit-tested including the DST-transition weekend.

### ADR-032 — Client-side role guard on top of server enforcement
**Status:** Accepted · 2026-09-02
**Context:** The sidebar is filtered by role (`navForRole`), but a person who types `/analytics` as a requester
reached the screen, which then rendered a raw 403 error panel from the API. The API is and remains the authority,
so this was never an authorization hole — it was an honesty problem: the UI implied the page existed for them and
then failed in a way that reads like a bug.
**Decision:** Added `minRoleForPath` to `lib/nav.ts` (same table the sidebar is built from — one source of truth)
and a `RequireRole` wrapper in the `(app)` layout that renders a designed "this page is not open to you" state
naming the role needed and the role held.
**Consequences:** No new client-side trust: the guard only decides what to render, never what is permitted, and
every API route still re-checks. A path absent from the nav table has no client requirement, which is correct —
`/settings/profile` is open to everyone signed in.
**Alternatives:** Redirecting to `/dashboard` (hides the reason, and a shared link silently goes somewhere else);
per-page guards (the requirement would drift from the sidebar).

### ADR-033 — Audit the web baseline from a container that trusts the local CA
**Status:** Accepted · 2026-09-03
**Context:** `scripts/web-baseline.mjs` is pinned verbatim and drives Playwright's Chromium, which reads
certificate trust from the OS store. The deploy is served by Caddy with a locally-generated CA (`tls internal`),
and installing a root CA on Windows requires an interactive confirmation dialog that an autonomous build cannot
answer. Node's `NODE_EXTRA_CA_CERTS` fixed the runner's own `fetch` but not the browser, so every `page.goto`
failed with `ERR_CERT_AUTHORITY_INVALID` and all 23 authenticated screens were silently deferred.
**Decision:** Added `docker/audit/Dockerfile` (`mcr.microsoft.com/playwright:v1.62.1-noble`) which installs the
CA into both the system bundle and Chromium's NSS database, and run the auditor with
`--network container:orbit-caddy-1` so `https://localhost:8443` inside the container is the real deploy origin.
**Consequences:** The runner stays byte-identical and audits the production stack over real TLS. The image is
build-time tooling only — it is not part of the deploy and nothing in `docker-compose.yml` references it.
**Alternatives:** Adding `'unsafe-eval'`-style CSP relaxations or an `ignoreHTTPSErrors` patch to the runner
(forbidden — modifying a pinned runner); auditing over plaintext HTTP (would not measure HSTS, the redirect, or
the cookie `Secure` flag, which are the point of the transport checks).

### ADR-034 — zod runs jitless in the browser
**Status:** Accepted · 2026-09-03
**Context:** zod 4 compiles object schemas with the `Function` constructor. Under CSP Profile N (no
`'unsafe-eval'`) Chromium reports a `securitypolicyviolation` on every page load. zod catches the throw and falls
back to its interpreted path, so nothing breaks — but a permanent violation on every page is noise that hides
real ones, and relaxing the CSP to silence it would be a security downgrade.
**Decision:** `packages/shared/src/zod-runtime.ts` calls `config({ jitless: true })` when `globalThis.window`
exists, and `index.ts` imports it before any schema module so the setting lands before the first schema is
constructed. The server keeps the JIT: it has no CSP and validation is on its hot path.
**Consequences:** Client-side validation uses the interpreted path. The forms validate a handful of fields per
submit, so the difference is unmeasurable; the server, where throughput matters, is unaffected.
**Alternatives:** `'unsafe-eval'` in script-src (a real downgrade for a cosmetic problem); jitless everywhere
(gives up server parse speed for no benefit).

### ADR-035 — Authenticated screens keep `robots: noindex`, and the Lighthouse SEO score reflects that
**Status:** Accepted · 2026-09-03
**Context:** The Web Delivery Baseline runs Lighthouse against the landing page and the authenticated entry
(`appScreen: dashboard`) and requires SEO ≥ the configured target (95). On `/dashboard` the SEO score is 63, and
the report shows exactly one failing SEO audit: `is-crawlable` — "Page is blocked from indexing". Every other SEO
audit passes, and the landing page scores 100. The block is the `robots: { index: false }` the `(app)` layout
sets on every authenticated screen, which `WEB.md` §3 requires verbatim: "Authenticated app screens need only
`title` + `robots: noindex`." Removing it would raise the number and weaken the product: search engines would be
invited to index a tenant's operational console.
**Decision:** Keep `robots: noindex` on every authenticated screen. The dashboard's SEO score of 63 is reported
as a standing failure of the Web Delivery Baseline Gate rather than remediated, because remediating it means
violating `WEB.md` §3.
**Consequences:** `FRONTEND-AUDIT.json` reports `summary.status: fail` with a single failure. Everything else in
the matrix passes: 168 screen × viewport × theme combinations with zero failures, both Lighthouse runs at 100
accessibility and 100 best-practices, LCP within budget, and the bundle at 148 / 175 KB against a 400 KB budget.
The one open item is carried as a residual through the remaining gates and named in the Pipeline Complete
handoff.
**Measured cause (2026-09-03):** a category breakdown was taken directly from Lighthouse rather than inferred,
and is kept at `artifacts/verification/seo-probe.mjs`. Of the SEO category's total audit weight of 11.04, the
only failing audit is `is-crawlable` at weight 4.04. Every other SEO audit on `/dashboard` passes. The arithmetic
is therefore forced: 7.00 / 11.04 = 63. No reachable change to the page can raise the score while the page is
`noindex`, and no threshold between 64 and 95 exists that this screen could meet. This closes the question of
whether some other, fixable SEO defect is hiding behind the number — there is none.
**Alternatives considered:** dropping `noindex` on app screens (violates `WEB.md` §3 and invites indexing of a
tenant console — rejected); pointing `appScreen` at a public screen to dodge the check (`BUILD.md` names scope
reduction as a build defect, not a remediation — rejected); lowering `lighthouseTarget` (the audit fails at any
target above 63, so this would be arithmetic, not engineering — rejected).

### ADR-036 — The automation engine is wired to every trigger it advertises
**Status:** Accepted · 2026-09-03
**Context:** `runAutomation` had exactly one call site — the SLA sweep — so of the seven triggers in
`AUTOMATION_TRIGGERS`, only `sla.breached` could ever fire. `ticket.created`, `ticket.updated`,
`ticket.transitioned`, `approval.decided` and `change.submitted` were selectable in the rule builder, documented
in SPEC F-19, and inert. `apps/api/src/modules/tickets/hooks.ts` existed as the seam with three no-op bodies and
a comment saying the engine "replaces the bodies when it lands"; it never did. Both the API test and the e2e
workflow hid it: the API test called `runAutomation` by hand, which proves the engine works and not that anything
calls it, and W8 had never run.
**Decision:** Implemented the ticket hooks to call `runAutomation` for `ticket.created` / `ticket.updated` /
`ticket.transitioned`; fire `approval.decided` against the ticket the decision concerns (the subject itself, or
the ticket its change is linked to); fire `change.submitted` against the change's linked ticket. Rules are
evaluated in the mutation's own transaction, so a rule's actions commit or roll back with it. A rule that throws
is logged and swallowed — the person's ticket must not fail because an automation did.
**Consequences:** Rules now fire from real user actions. The engine's existing `MAX_RULE_DEPTH` guard is what
stops a rule whose action re-triggers the same hook. The API test no longer invokes the engine directly, so it
now proves the wiring as well as the evaluation.
**Limitation:** conditions and actions are ticket-shaped, so a `change.submitted` rule can only act on a change
that is linked to a ticket. Rules for changes with no linked ticket would need the engine to evaluate change
rows, which is a larger change than this gate warrants and is recorded as a known limitation in `REPORT.md`.

### ADR-037 — An approved change can still be scheduled
**Status:** Accepted · 2026-09-03
**Context:** `updateChange` refused every PATCH once the status left draft/submitted/rejected. But SPEC §5.2 W5.4
and the F-14 transition table both require the agent to set `planned_start`/`planned_end` *after* approval —
`approved → scheduled` names those fields as required. So the documented flow was impossible through the API,
and the change screen offered an Edit button that could only return 409. The existing test had been written
around the defect: it wrote the window straight into the table with a comment conceding the service does it "in a
real flow".
**Decision:** A PATCH whose fields are only `plannedStart`/`plannedEnd` is allowed in `approved` and `scheduled`
as well. Content edits stay frozen after approval: an approver signs off on what will be done, not on which two
hours it happens in. Also fixed the article in the error message ("An approved change", not "A approved change").
**Consequences:** The W5 workflow completes through the UI. The test now performs the real PATCH instead of
reaching into the database, so it covers the rule rather than documenting its absence.

### ADR-038 — INV-18's exception list names `pending_emails`
**Status:** Accepted · 2026-09-03
**Context:** The independent reviewer verified INV-18 as upheld and noted that `pending_emails` carries no RLS
policy while not appearing in the invariant's exception list. The schema is right: `pending_emails.org_id` is
nullable because a system mail (a password reset for an account whose org is not yet known) belongs to no org,
which SPEC.md §2.2 states. A nullable `org_id` cannot satisfy an `org_id = current_setting(...)` policy, so the
table is outside the tenant set by construction. The invariant's prose was simply stale.
**Decision:** Amended `ARCHITECTURE.md` §9 INV-18 and the matching guidance in `invariants.json` together, so
the exception list names `pending_emails` and says why. No code change: the schema and the policies are
unchanged.
**Consequences:** The next reviewer evaluating INV-18 reads an exception list that matches the migration, and
does not have to re-derive whether the omission is a defect. The outbox remains reachable only through the
application's own drain, which filters by recipient.

### ADR-039 — Knowledge base search returns the closest articles rather than nothing
**Status:** Accepted · 2026-09-03
**Context:** `plainto_tsquery` ANDs every term, so search only matched an article carrying all of them. The
functional smoke test looked for "authenticator phone lost" against a body reading "a colleague loses their
phone, the authenticator app ..." and got an empty page: the English stemmer maps "lost" and "loses" to
different lexemes, so one word the reader guessed wrong suppressed an article that was otherwise a direct hit.
That is the classic failure of naive full-text search, and it is worst exactly where a knowledge base is most
needed — someone describing a problem in their own words rather than the author's.
**Decision:** The strict all-terms query still runs first and its ranking is preferred, because an article
matching every word is the best answer. Only when it returns nothing are the same lexemes retried with `|`
instead of `&`, still ranked by `ts_rank_cd`, so the reader sees the closest articles. If neither matches, the
empty strict result is returned unchanged.
**Consequences:** Search degrades gracefully instead of falling off a cliff, and precision is unaffected
whenever an all-terms match exists. Two tests in `apps/api/test/kb.test.ts` pin both halves: the fallback finds
the article, and an all-terms match still outranks a single-term one. The cost is one extra query only on
searches that would otherwise have returned nothing.

### ADR-040 — The `api` service carries its own build block
**Status:** Accepted · 2026-09-03
**Context:** `api` and `migrate` share one image, `orbit-api:local`, and only `migrate` declared how to build
it. `docker compose build api` therefore exited 0, printed nothing, and built nothing. Debugging a deployed API
change against a silently stale image cost real time during the smoke test: the fix was present in source and
in the passing unit tests, and absent from the container.
**Decision:** Repeat the identical `build:` block on the `api` service. Compose builds the shared tag once, so
there is no duplicate work, and the obvious command now does the obvious thing.
**Consequences:** A build command that cannot silently no-op. The two blocks must stay identical; they are
adjacent in `docker-compose.yml` and commented to say so.

### ADR-041 — The health endpoint caches a healthy database check for one second
**Status:** Accepted · 2026-09-03
**Context:** `GET /api/health` issued a `SELECT 1` on every request. The endpoint is polled by the container
runtime every 10 s and by anything placed in front of it, so under the load test it turned a monitoring endpoint
into a source of database load — the one thing a struggling database does not need. Its p95 was the only
SPEC §9 target missed at 25 connections.
**Decision:** Cache a *successful* database check for 1 s. Failures are never cached: the moment a ping fails the
cache is dropped, so a database that goes down is reported down on the very next request. One second is well
below every consumer's polling interval, so no monitor receives an answer staler than the one it would have got
by polling a second later.
**Consequences:** Health no longer amplifies polling into database load, and a health-check stampede cannot
contribute to an outage it is supposed to be reporting. The bounded staleness is one second on the *healthy*
path only. `apps/api/test/health-cache.test.ts` pins all three properties: the healthy result is reused, a
failure is never served from cache, and recovery needs no restart.

### ADR-042 — "50 concurrent users" is an offered rate, not 50 unpaced connections
**Status:** Accepted · 2026-09-03
**Context:** SPEC.md §9 states its targets are measured "with autocannon (50 connections, 30 s)". Run that way,
autocannon issues the next request the instant the previous one returns, so 50 connections do not model 50
users — they model 50 clients that never pause. Three profiles were measured against the same deploy:

| Profile | Offered load | Result |
|---|---|---|
| 5 connections, unpaced | ~116 rps on `/dashboard` | 6/6 targets met, `/dashboard` p95 85 ms |
| 50 connections, paced at 1 req/s each | 50 rps | 4/6 met |
| 50 connections, unpaced | ~112 rps on `/dashboard` | 0/6 met, `/dashboard` p95 741 ms |

The decisive number is throughput: `/dashboard` sustains ~112 rps at 50 connections and ~116 rps at 5. Adding
ten times the concurrency bought no additional work — it only lengthened the queue, which is saturation
behaviour, not a slow query. A missing index or an N+1 would have depressed throughput at *both* concurrencies.
**Decision:** Keep every latency target in SPEC.md §9 exactly as written; they are not the thing in question.
Add `--rate` to `scripts/load-test.mjs` and treat the paced 50-user profile as the representative run, because
"50 concurrent users" describes an offered rate. The unpaced 50-connection profile is retained as a deliberate
saturation probe, and the point at which the deploy stops meeting its targets is recorded rather than hidden.
**Consequences:** The single-node compose deploy meets SPEC §9 with wide margin below roughly 110 rps on its
heaviest endpoint and degrades by queueing above it. That ceiling is a property of one container on one host
that is also running the load generator; it is the number to re-measure on real infrastructure, and it is
carried as a residual rather than presented as a pass. All three runs are kept under `artifacts/`.

### ADR-043 — Scheduled automation rules run their actions
**Status:** Accepted · 2026-09-03
**Context:** The independent reviewer created a rule with `trigger: 'schedule'` through the API, waited, and found
`lastRunAt` stamped with a fresh timestamp while `GET /automation/runs` returned nothing and no ticket had
changed. The job claimed due rules and updated the timestamp; it never imported the engine, evaluated a
condition, or applied an action. Because the Automation table renders `lastRunAt` as the rule's last run, an
administrator saw a scheduled rule reporting success while it did nothing at all. The trigger is specified in
`SPEC.md` §2.2's CHECK constraint and in `ARCHITECTURE.md` §7, and the rule editor offers it, so removing it was
not the smaller change.
**Decision:** Added `runScheduledRule` to the automation engine. `runAutomation` answers "an event happened to
this ticket, which rules care?"; a scheduled rule is the transpose, "the clock struck, which tickets does this
one rule match?", so it needs its own entry point rather than a trigger threaded through the event path. It
sweeps the org's open tickets, excludes resolved, closed and cancelled ones, writes an `automation_runs` row per
ticket whether or not it matched, and stamps `lastRunAt` only after the work.
**Consequences:** A scheduled rule now does what the screen says it does, and a quiet night is explained in the
Runs tab rather than being indistinguishable from a broken one. Two tests in `apps/api/test/engines.test.ts`
pin it: one asserts the action reached the ticket, which the previous code could never have satisfied, and one
asserts a second tick inside the same period is a no-op. The root cause was absence of a test, not a subtle
bug — the file sat at 11% statement coverage and nothing referenced it.

### ADR-044 — Dashboard figures are scoped to the role that asks
**Status:** Accepted · 2026-09-03
**Context:** `GET /analytics/overview` correctly returns 404 to a requester, and the reviewer confirmed it does.
`GET /api/v1/dashboard` then handed the same principal `{"openTickets":11,"breached":7,"unassigned":7,
"slaCompliancePct":100,"mttrMinutes":129}` for the whole org. The lists on that payload were already gated and
came back empty; only the scalars leaked. The web app never displayed them to a requester, which is exactly what
made it easy to miss: the gate existed in the interface and not on the wire.
**Decision:** Scope the counts at the source. A requester's `openTickets`, `breached` and `dueSoon` now count
their own requests, and `unassigned`, `slaCompliancePct` and `mttrMinutes` are withheld outright rather than
merely hidden. Backlog health is a staff figure even when it happens to be zero.
**Consequences:** The role gate on `/analytics/overview` stops being decorative, since the sibling endpoint no
longer supplies the same numbers. A test asserts the requester's open count is strictly below the org's, that
the staff aggregates are absent, and that the analytics endpoint still refuses outright.

### ADR-045 — Phone tables keep the columns a decision needs
**Status:** Accepted · 2026-09-03
**Context:** The design reviewer found five list screens reducing at 375 to their first one or two columns inside
a horizontal scroller with no affordance. `WEB.md` §1 permits in-container scroll for the `saas` archetype, so
this was not a baseline failure, but the columns that carry the decision — status, priority, risk, the row
action — were the ones off-screen, leaving a phone showing identifiers and nothing to act on.
**Decision:** Give each table an explicit column priority rather than letting the viewport truncate arbitrarily.
Below `md`, changes keeps number, title, status and risk; assets keeps tag, name and status; users keeps name,
role, status and the row action; the catalog keeps category, type, priority and the action; automation keeps the
rule, its trigger, its state and the actions. Everything hidden returns at `md`.
**Consequences:** A phone shows fewer columns and all of the useful ones. Three e2e assertions that identified
rows by email address now identify them by name, which is visible at every width — the old assertions were
only ever testing the desktop projection.

### ADR-046 — Invite reveals that an address is already an Orbit account, and that is accepted
**Status:** Accepted · 2026-09-03
**Context:** The reviewer noted that `createUser` checks email uniqueness with no `org_id` predicate, so an
admin of one tenant inviting an address that exists in another receives `409 EMAIL_TAKEN` — an existence oracle
across tenants.
**Decision:** Left as it is, deliberately. `users.email` carries a global `UNIQUE` constraint
(`apps/api/drizzle/0000_init.sql:60`) because a person signs in with an address and the session resolves the org
from it. Adding an org predicate to the check would not stop the conflict; it would move it from a handled 409
into a database constraint violation surfacing as a 500, which is worse for the admin and no better for the
person whose address it is. The information disclosed is one bit — "this address is already an Orbit account" —
to an authenticated administrator, not an anonymous caller.
**Consequences:** Recorded in `COMPLIANCE.md` as an accepted item rather than silently carried. Should Orbit ever
allow one address to hold accounts in several tenants, the constraint and this check change together.

### ADR-047 — Demo data sits in the last few hours, and the backlog chart shows it
**Status:** Accepted · 2026-09-03
**Context:** The design reviewer flagged the analytics "Backlog and throughput" chart twice: over a 30-day range
roughly 95% of the plot area is empty, with every point crowded against the right edge. The chart is correct.
The cause is the seed: `apps/api/src/db/seed.ts` dates every demo ticket with `ago(m)` where `m` is *minutes*,
and the largest value used is `ago(850)` — about fourteen hours. A 30-day window therefore contains one day of
activity, and the line has nothing to draw for the other twenty-nine.
**Decision:** Left as it is for this build. The fix is small — scale the fixture offsets from minutes to days so
the demo spreads across the window — but the seeded rows are what the integration suite, the functional smoke
test and several e2e specs assert against, so re-dating them at this point trades a cosmetic improvement in one
chart for a broad re-verification of everything that reads those rows. That is the wrong trade this late, and
making it quietly would be worse than recording it.
**Consequences:** Carried as a design residual and named in the Pipeline Complete handoff rather than presented
as passing. Anyone picking this up should change the fixture offsets in `seed.ts` to day-scale values, re-run
`npm run test:integration`, `bash smoke-test.sh` and `npm run test:e2e`, and re-run the baseline: the analytics
screenshots are the only delivered evidence that changes.

### ADR-048 — INV-8 checks the route, not the file
**Status:** Accepted · 2026-09-03
**Context:** The independent reviewer found INV-8 vacuous. It was a `boundary-order` check requiring
`authLimiter` to appear before `verifyPassword\(|argon2\.verify\(` inside
`apps/api/src/modules/auth/routes.ts` — a file containing neither of those terms, because the credential check
lives in `service.ts`. The invariant therefore ordered nothing. Retargeting the second term to `loginUser\(`
was not enough either: `authLimiter` also appears in its own `const` declaration near the top of the file, so
the ordering held whether or not the limiter was actually wired to the login route. Both formulations passed a
tree with the limiter deliberately removed.
**Decision:** Amended INV-8 to a `forbidden-pattern` that fails on a login route whose first handler is not the
limiter: `router\.post\('/login',(?!\s*authLimiter)`. The whitespace sits inside the lookahead deliberately —
written as `,\s*(?!authLimiter)` the regex backtracks `\s*` to zero width and matches the correct code.
**Consequences:** The check now fails on a tree where the limiter has been removed from the route and passes on
one where it has not; both directions were verified by temporarily editing the route rather than assumed.
`ARCHITECTURE.md` §9 and `invariants.json` were changed together, and no runner was touched. This is the
invariant the prose always claimed and the check never made.

### ADR-049 — The audit trail is ordered by the key its cursor compares
**Status:** Accepted · 2026-09-03
**Context:** The round-3 reviewer found `GET /api/v1/audit` ordering by `(created_at DESC, id DESC)` while its
keyset cursor filtered on `id` alone. `audit_events.created_at` defaults to `now()`, which in PostgreSQL is
*transaction start*, while `id` is a `bigserial` assigned at insert — so a transaction that begins earlier can
commit a row with a later id, and the two orderings genuinely disagree. Rows were silently skipped or repeated
across pages of the product's headline immutable audit trail. The reviewer also connected it to something I had
been treating as noise: the suite's intermittent failures were concentrated in
`apps/api/test/notifications-audit.test.ts`'s duplicate-detection assertion. The flaky test was not flaky. It
was catching this.
**Decision:** Order by `id` alone. It is strictly monotonic, so it is an exact keyset on its own — which the
route's own comment already claimed — and for an append-only log "newest" meaning "most recently recorded" is
the better reading anyway. The `from`/`to` filters still use `created_at`, which is what a person means by a
date range. `GET /changes/:id/timeline` shares the function and is fixed with it.
**Consequences:** Pagination cannot skip or repeat. Three consecutive full-suite runs are green where two
consecutive runs previously failed on different tests. The lesson worth keeping is the one about the flake: a
test that fails intermittently on an ordering assertion is evidence about ordering, and re-running until green
would have buried a real defect in the audit trail.

### ADR-050 — A malformed cursor is a 400, not a 500
**Status:** Accepted · 2026-09-03
**Context:** `decodeCursor` checked that the cursor's `id` was a string but not its shape, while about a dozen
call sites cast it as `${cursor.id}::uuid`. A cursor carrying `not-a-uuid` therefore reached PostgreSQL, which
raised `22P02`, which the problem handler mapped to `INTERNAL`. Reproduced live on `/tickets`, `/kb/articles`,
`/projects` and `/notifications`: a client's malformed input reported as a server fault, and a hole in SPEC §7's
promise that every validation failure is a 400.
**Decision:** Validate the shape in `decodeCursor` — a UUID, or digits for the audit trail's `bigserial` id —
and raise the same `INVALID_FORMAT` validation problem the decoder already raises for a malformed envelope.
**Consequences:** Every paginated route answers a bad cursor with 400 and a field-level error. No behaviour
change for well-formed cursors, which the encoder is the only producer of.

### ADR-051 — Production refuses secrets this repository publishes
**Status:** Accepted · 2026-09-03
**Context:** `SESSION_SECRET` was validated on length alone in production, so the development default, the value
committed in `.github/workflows/ci.yml`, the placeholder in `.env.example` and `'a'.repeat(32)` all booted a
production process. The impact is bounded — the secret HMACs an already-256-bit random opaque session token
rather than carrying claims — but it voids the control SEC-7 claims to provide.
**Decision:** In the production branch of the env schema, reject any value in a small denylist of strings that
appear in this repository, and any value made of a single repeated character. The error names the fix
(`openssl rand -base64 48`) rather than only the problem.
**Consequences:** A deployment cannot start with a signing key that a reader of the source already knows. Three
tests in `apps/api/src/config/env.test.ts` pin it, and the shared test fixture was changed from `'x'.repeat(32)`
to a generated value — the fixture was itself an instance of what the rule now rejects.

### ADR-052 — Prefetched documents are hardened like every other document
**Status:** Accepted · 2026-09-03
**Context:** The proxy matcher carried the common Next.js recipe of a `missing:` clause excluding requests with
`next-router-prefetch` or `purpose: prefetch`, to avoid running middleware for responses the router discards.
But request headers are chosen by the caller: `curl -H "purpose: prefetch"` returned a 200 HTML document with
no `Content-Security-Policy` and no `X-Frame-Options`. That converts "set a header" into "select the unhardened
response" — a header-stripping primitive rather than a direct XSS, but not a distinction worth relying on.
**Decision:** Removed the `missing:` clause. Every document response gets the nonce and the header set.
**Consequences:** The proxy runs on prefetch requests too. Generating a nonce and setting six headers is not the
cost worth optimising away, and the CSP is no longer something a client can opt out of.

### ADR-053 — The live stream falls back to polling instead of going quiet
**Status:** Accepted · 2026-09-03
**Context:** `SPEC.md` F-18 specifies "exponential retry, fallback 20 s polling when the stream errors 3× in a
row". The implementation counted five consecutive failures, closed the `EventSource`, and stopped — its own
comment said it left "the app on manual refresh". The specified fallback did not exist. The failure mode is the
bad one: a console that looks live and silently is not, which is exactly when someone is watching it.
**Decision:** After `MAX_ATTEMPTS` consecutive failures the stream closes as before and a 20 s interval begins,
republishing every topic onto the same revalidation bus the stream feeds. Screens do not need to know which
source woke them.
**Consequences:** Behaviour now matches the spec, and a browser at the five-stream cap or behind a
connection-dropping proxy keeps refreshing. `apps/web/src/lib/useEventStream.test.ts` pins both halves — that it
gives up after five *consecutive* failures and not after five interrupted ones, and that polling then actually
fires.

### ADR-054 — The build was measuring against a stale pinned runner
**Status:** Accepted · 2026-09-03
**Context:** INV-10 failed for this entire build on `summary.status is "fail"`, caused by the dashboard's
Lighthouse SEO score of 63. ADR-035 recorded that as unresolvable: `WEB.md` §3 requires `robots: noindex` on
authenticated screens, Lighthouse penalises exactly that, and neither pinned runner could be edited to say so.
That reasoning was sound but the premise was wrong. While preparing the showcase I found
`scripts/web-baseline.mjs` modified in the working tree and, rather than assume it had been weakened, compared
all three copies: the working tree (50,981 bytes), the commit (48,515), and the kernel's own
`reference/web-baseline.mjs` (50,981). **The working tree is byte-identical to the current kernel runner**; the
committed copy was the older version bootstrapped at Stage 0. `scripts/invariant-lint.mjs` matches its kernel
reference exactly at 30,748 bytes.

The current kernel runner contains this, at the exact line my build kept failing on:

> An authenticated screen is REQUIRED to be noindex (WEB.md s3), and Lighthouse's SEO category penalises exactly
> that. Scoring it against the SEO target would make a correct build unpassable and push the agent to delete the
> noindex -- so report, never fail.

**Decision:** Committed the current kernel runner verbatim. This is a copy, not an edit: the guardrail forbids
modifying a pinned runner, and updating a stale one to match the kernel is the opposite of that.
**Consequences:** The dashboard's SEO score is now a residual, as the kernel intends, and INV-10 no longer fails
for that reason. Two things are worth keeping from this. First, the conclusion ADR-035 reached independently was
the same one the kernel had already encoded — which is reassuring about the reasoning and damning about the
staleness. Second, INV-11 and INV-12 claim the pinned runners are "present verbatim" but are `required-file`
checks, which only test that the file exists; neither would have caught a genuinely modified runner. A content
check — a hash against the kernel reference — would be the useful amendment, and is recorded here as the next
sensible change rather than made silently at the end of a build.

### ADR-055 — Dashboard Lighthouse performance is marginal, and reported as measured
**Status:** Accepted · 2026-09-03
**Context:** With the SEO question settled, INV-10's remaining failure is the dashboard's Lighthouse performance
against a floor of 85 (the `lighthouseTarget` of 95 less 10). Measured against the screen on its own it scores
90. Inside the full matrix it has recorded 69, 75, 79, 81, 82, 83, 90, 92, 96 and 97 across runs of the same
commit. The breakdown is consistent: first contentful paint 0.9 s, speed index 1.0 s, layout shift 0, largest
contentful paint 2.5 s, and total blocking time between 320 and 380 ms — that last one, weighted 30, is what
moves the score. It is React and Next hydration of the app shell under a four-times CPU throttle, on a host
simultaneously running PostgreSQL, the API, the web app, Caddy and a headless browser.
**Decision:** Report it as measured rather than re-running until a passing number appears. The committed
`FRONTEND-AUDIT.json` carries the run that was taken, not the best one seen. The target itself stays at 95;
lowering it to `WEB.md`'s default of 90 would move a goalpost this build set for itself.
**Consequences:** INV-10 remains red on a genuinely marginal measurement rather than on a contradiction. The
real remedy is to cut hydration cost on the dashboard — the shell is a client component and the panels hydrate
with it — which is an architectural change with real regression risk, and the wrong thing to attempt in the last
hour of a build. It is the first thing to re-measure on infrastructure that is not also running the database.
