# DECISIONS.md — STΛRDANCΞR

Architecture Decision Records. Each ADR records a decision that **changed** something relative to the inputs (RESEARCH.md, kernel defaults) or resolved a blank / conflicting field.

Precedence used throughout: outer build prompt > kernel (BUILD.md / DESIGN.md / WEB.md); design template > RESEARCH.md Design Language > archetype defaults; RESEARCH Non-Goals bound scope.

---

## ADR-001 — Stage 0 research run (§3 absent) → GO
- **Context:** RESEARCH.md was wizard-generated with no §3 Source Categories; `force_research: false`.
- **Decision:** Ran Appendix A research at `build_depth: comprehensive`; appended §3.1–§3.9, §4 stack, §5 risk register + STRIDE threat model, §6 gaps, §7 GO.
- **Consequence:** Stage 1 reads the enriched document. Commit `625c081`.

## ADR-002 — Pre-build review (Step 3b): 8 findings applied
- **Decision:** PBR-1…PBR-8 (RESEARCH.md §"Pre-Build Review") applied as edits to RESEARCH.md: dark-only template theme, domain corrected to "Security & Operations Intelligence", measurable thresholds for freshness (p95 ≤ 60 s), availability (99.9 %), RTO ≤ 1 h / RPO ≤ 5 min, generic ingest API for named producers, concrete uses for `has_payments` / `has_geo`, simulated-data labelling, in-platform cryptographic trust.

## ADR-003 — Design tokens: DESIGN-TEMPLATE.html `:root` copied verbatim; dark-only
- **Context:** RESEARCH Design Language lists a light blue palette (`#2665ed`) and "Theme: Light only"; RESEARCH prose demands a "coral-accented dark interface"; the uploaded template is dark.
- **Decision:** The template's `:root` block is the canonical token set, transcribed exactly into `web/src/styles/tokens.css`:
  ```css
  :root{
    --bg:#06070a; --bg2:#0b0d12; --panel:rgba(14,17,23,.78);
    --line:rgba(255,255,255,.07); --line2:rgba(255,255,255,.13);
    --text:#ebe7df; --muted:#8b909a; --dim:#575c66;
    --amber:#ffb547; --ember:#ff7a3d; --cyan:#5fd4ff; --green:#5fe3a1; --red:#ff5d6c; --violet:#a78bfa; --white:#f4f1ea;
    --mono:ui-monospace,"SF Mono","JetBrains Mono",Menlo,Consolas,"Liberation Mono",monospace;
    --sans:"Inter","Segoe UI",system-ui,-apple-system,Roboto,"Helvetica Neue",Arial,sans-serif;
    --r:14px;
  }
  ```
  Tokens the template did not declare are **derived alongside** (never replacing): `--display` (Manrope, ADR-004), spacing scale `--s1..--s8` on a 4 px unit, type scale steps (base 14 × 1.2, RESEARCH), `--r-sm:6px`, `--r-xs:2px` (RESEARCH "sharp" radius for dense controls), motion `--t-fast:140ms --t-med:220ms --t-slow:320ms`, focus ring (amber, template `:focus-visible`). Semantic mapping: primary action = amber→ember gradient (template `.btn.primary`); relationships/telemetry = cyan; AI / Mirror Brain = violet; healthy = green; critical = red; warning = amber.
- **Theme:** single dark theme (template). RESEARCH light palette superseded (PBR-1). Ported surfaces: topbar, hero with `#field` canvas constellation + sigil, gradient `h1 .name`, chips, segmented control, stats band, panels.
- *Evolved (Stage 4 reconciliation):* `--dim` is `#7c818c` (WCAG AA, ADR-050); every other `:root` value is verbatim.

## ADR-004 — Typography: Manrope display + Inter body + JetBrains Mono data
- **Context:** Template `--sans` is Inter only — DESIGN.md §8.1 anti-slop risk (default face with no display face).
- **Decision:** Keep `--sans` verbatim for body/UI; add Manrope (RESEARCH typography "Heading: Manrope 600/700") as the deliberate display face for headings, metrics and the wordmark; JetBrains Mono (already in the template `--mono` stack) for identifiers, hashes, counters. All three self-hosted via `next/font` (≤ 3 families, WEB.md §4). *Evolved (Stage 4 reconciliation):* self-hosted with `@fontsource-variable/*` 5.3.0 imported in the root layout instead of `next/font` — same files, same-origin under the strict CSP (`font-src 'self'`), no build-time Google fetch.

## ADR-005 — Archetype `saas`, dense operational shell
- Archetype from RESEARCH (`saas`). Persistent left sidebar (224 px, RESEARCH layout) with the 11 destinations from the reference image; below `md` the sidebar becomes a focus-trapped drawer. Landing surface (`/`) ports the template hero (marketing-grade composition inside the saas product).

## ADR-006 — Database: Neon in production, `postgres:17` container in the local stack
- **Context:** `database_preference: Neon` (hard constraint); no Neon credentials available to the build; the deploy is a local stack.
- **Decision:** Prisma targets PostgreSQL; env accepts Neon pooled `DATABASE_URL` (`-pooler` host, `pgbouncer=true`), `DIRECT_URL` for migrations, optional `DATABASE_REPLICA_URL` for the read split. The local compose stack runs `postgres:17-alpine` as a wire-compatible stand-in. QUICKSTART documents the Neon switch. Disposition in reconciliation: **Applied** (Neon-compatible config) with local stand-in.

## ADR-007 — `auth_model` blank → email/password with server-side sessions; API keys for machines
- **Decision:** Human users: email + password (argon2id, NIST 800-63B verifier: ≥ 12 chars, ≤ 128, breached/common-password screen, no composition rules), opaque 256-bit session token in an `HttpOnly; Secure; SameSite=Lax` cookie, SHA-256 hash stored server-side, 12 h idle / 7 d absolute expiry, rotation on login, double-submit CSRF token. Machines (telemetry sources, AI agents): org-scoped API keys `sd_<kind>_<prefix>_<secret>`, SHA-256 hashed at rest, shown once, revocable. SSO/OIDC deferred (not in RESEARCH).
- *Evolved (Stage 4 reconciliation):* session and reset tokens are stored as HMAC-SHA256 keyed by `SESSION_SECRET` (ADR-038); API keys are `sd_<in|ag>_<secret>` with the display prefix taken from the secret (ADR-044).

## ADR-008 — Deploy: local compose stack (Outcome C) behind Caddy `tls internal`; ports 8280/8643
- **Context:** `deployment` blank; no cloud credentials; `protocol_support: HTTPS only`; host ports 80/443/3000/5432/3300 are owned by foreign processes (probed 2026-10-12).
- **Decision:** `compose.yaml` runs caddy (TLS edge, `:80`→`:443` redirect, internal CA), web (Next.js), api ×2 replicas, worker, postgres, redis, prometheus, grafana. Host ports via env: `HTTP_PORT=8280`, `HTTPS_PORT=8643`, `PROMETHEUS_PORT=9390` and `GRAFANA_PORT=3390` (both bound to 127.0.0.1). Deploy URL `https://localhost:8643`.

## ADR-009 — Realtime: Server-Sent Events + Redis pub/sub (not Socket.IO)
- **Context:** `realtime_needed: true`, flows are server→client (feed, health, proposals); 50k+ concurrent target across horizontally scaled API replicas.
- **Decision:** `GET /api/v1/stream` (SSE, cookie-auth, org-scoped channel) fed by Redis pub/sub; no sticky sessions, works through Caddy HTTP/2, CSP `connect-src 'self'`. Clients treat events as invalidation hints and refetch via RTK Query (events are never the source of truth). Heartbeat 25 s; `Last-Event-ID` ignored (refetch on reconnect). Alternative rejected: Socket.IO (sticky-session handshake, larger client bundle).

## ADR-010 — Mirror Brain Ψ: deterministic, explainable inference (no LLM)
- **Context:** No LLM provider named; RESEARCH "AI Independence: core system remains operational without an LLM"; Non-Goals forbid treating hypotheses as facts.
- **Decision:** Predictions are generated by documented rules over graph statistics (impact propagation, exposure × vulnerability × criticality, staleness/drift trends, change velocity). Each prediction stores `assumptions[]`, `evidence[]` refs, `confidence` (0–100) and an `uncertainty` statement, and is visually distinct from observed facts (violet, "PREDICTION" label). A provider interface is left for a future LLM adapter; none ships.
- *Evolved (Stage 4 reconciliation):* no LLM adapter seam ships; a future adapter would wrap `generatePredictions` in `server/src/engine/mirror-brain.ts`.

## ADR-011 — Billing without Stripe keys: simulated billing mode
- **Context:** `has_payments`; no Stripe credentials.
- **Decision:** `BILLING_MODE=stripe|simulated` (default `simulated` locally). In `stripe` mode Checkout Sessions/Refunds call Stripe with an `Idempotency-Key`. In `simulated` mode the server creates a local checkout session and the hosted-checkout step is replaced by an in-app "Simulated checkout" confirm page that causes the server to emit a **signed** webhook (`stripe.webhooks.generateTestHeaderString` with `STRIPE_WEBHOOK_SECRET`) to its own webhook route — the same verify → claim → process path real Stripe uses. Simulated routes return 404 when `BILLING_MODE=stripe`. Live keys are a QUICKSTART provisioning step (Pipeline Complete step 2).

## ADR-012 — No email provider: one-time links displayed to admins
- **Context:** No `has_email` signal / provider.
- **Decision:** Member invitations and admin-issued password-reset links are single-use, hashed, 72 h / 1 h expiry tokens whose URL is shown once to the issuing admin to deliver out-of-band. No self-service "forgot password" email (would require a provider). Sign-up anti-enumeration still holds (identical response shape/timing).

## ADR-013 — `has_geo`: entity site coordinates as DECIMAL(9,6); self-rendered map; no provider
- **Decision:** `entities.lat DECIMAL(9,6)`, `lng DECIMAL(9,6)` with DB CHECK (−90..90, −180..180) and Zod bounds; both-or-neither CHECK. The constellation "Geography" view projects coordinates with an equirectangular SVG projection drawn by the app — no tile/map provider, so there is no billable proxy to abuse (has_geo checklist item satisfied by elimination). Coordinates describe sites/regions (datacenters, offices, cloud regions), not people; still treated as sensitive topology (org-scoped, redacted from logs). Using DECIMAL means the money `forbidden-pattern` needs no lat/lng exemption.

## ADR-014 — Payments model: tenant subscription plans in BIGINT minor units
- **Decision:** Plans `explorer` (free), `constellation` (USD 49 000 minor units / month), `sovereign` (USD 249 000 / month) in a server-side catalogue; amounts `BIGINT` + `currency CHAR(3)`; webhook is the source of truth for subscription and payment state; `stripe_events.event_id` UNIQUE; checkout requests UNIQUE `(org_id, idempotency_key)`; refunds (owner-only, partial or full, ≤ remaining) and dispute lifecycle (`disputed` → `dispute_won|dispute_lost`) driven by webhooks; no PAN ever touches the app.

## ADR-015 — `security_baseline` blank → OWASP Top 10 + 800-63B verifier floor
- No named regime (SOC2/HIPAA/PCI) → OWASP Top 10 floor per kernel; adopted NIST 800-63B password verifier (cheap, password auth). PCI scope avoided by Stripe-hosted checkout.

## ADR-016 — `scale: large` → large tier architecture; load-test residual
- **Decision:** Stateless API (×2 in compose, horizontally scalable), Prisma pooled connections + read-replica routing for read-heavy endpoints (`DATABASE_REPLICA_URL`, falls back to primary), Redis cache-aside for hot reads (overview summary, graph snapshot) with explicit TTL + invalidation on write, Caddy edge rate limiting is not built in → edge-enforced limit implemented at the first app hop (Redis token bucket keyed by IP) plus cost-weighted limits on traversal endpoints; BullMQ workers with concurrency caps, backpressure (queue depth gate on ingest → 429) and DLQ (`failed` set retained + `dlq` queue). CDN: immutable hashed static assets with long `Cache-Control` (a CDN fronts them in prod).
- **Residual:** sustained 50k concurrent cannot be generated or served on one 8-core host; the load test (BUILD Load Test Methodology) runs at the highest stable concurrency and reports the §9 verdict honestly (may be red).

## ADR-017 — Blank Build Constraint / key-decision fields → defaults
| Field | Default | Rationale |
|---|---|---|
| `deployment` | local Docker Compose (ADR-008) | no cloud target supplied |
| `backend_language` | TypeScript on Node 22 | forced by Next.js + Express constraints |
| `container_strategy` | Docker (multi-stage, non-root, tini) | large-tier horizontal scale |
| `ci_cd_required` | GitHub Actions `ci.yml` | Service Floor |
| `pii_handling` | emails, names, IPs, user agents = PII → logger redaction, org scoping, no export of other users' PII to viewers | conservative |
| `data_migration` | greenfield | new product |
| `offline_capable` | false | operational console |
| `i18n` | English only, `lang="en"` | not requested |
| `notifications` | in-app feed + SSE only | no email/SMS provider (ADR-012) |
| `confidence_level` | n/a (`review_gates: auto` matches first) | — |
| `prompt_mode` | `direct` | blank |

## ADR-018 — Governed actions execute as recorded state transitions (no external connectors)
- **Context:** Non-Goals: "Execute unapproved infrastructure changes", "Build every possible integration before the core System Graph works".
- **Decision:** Approved proposals are executed by the in-platform executor, which applies the action's effect to the System Graph (e.g. `isolate_endpoint` → entity `containment: isolated`; `disable_credential` → identity `status: disabled`) and records the outcome in the evidence ledger, labelled "Recorded in STARDANCER — no external connector configured". Real connector adapters are deferred.
- *Evolved (Stage 4 reconciliation):* `disable_credential` records `containment: 'disabled'` on the entity plus evidence; the UI does not show a separate "no external connector" label.

## ADR-019 — Integrations: one authenticated generic ingest API
- `POST /api/v1/ingest` accepts batches of typed observations (entity / relationship / event / vulnerability / health / change) with `sourceKind` naming the producer (threat_watch, cyberops, synapse_pulse, alpha_zero, edr, siem, cloud, identity, cmdb, manual, simulated). Dedicated connectors deferred (PBR-5).

## ADR-020 — Header-buffer sizing
- Caddy: generous header limits by default — no action required (recorded per kernel). Node: `--max-http-header-size=32768` in the api, worker and web start commands and Dockerfiles (≥ 16 KB cookie smoke flow).
- *Evolved (Stage 4 reconciliation):* `--max-http-header-size=32768` is on the api and web Docker CMDs only (the worker serves no HTTP beyond metrics); Caddy pins `max_header_size 32KB`.

## ADR-021 — Monorepo: `shared` + `server` (Express API + workers) + `web` (Next.js UI)
- **Decision:** npm workspaces. Next.js renders UI only (no business logic in route handlers; only `robots`, `sitemap`, `manifest`, `opengraph-image`, `icon` metadata routes). Caddy routes `/api/*` → api, everything else → web, so the browser sees one origin (cookies, CSP `connect-src 'self'`). Alternative rejected: Next.js route handlers as the API (violates `backend_framework: Express`).

## ADR-022 — Versions pinned against registry (2026-10-12)
- Prisma 7.10.0 (8.0 is RC → rejected), TypeScript 6.0.3 (7.0 is outside typescript-eslint 8.71 peer range `<6.1.0`), Next 16.4.0, React 19.3.0, Express 5.3.0, BullMQ 6.3.12, ioredis 6.0.0, RTK 2.13.0, Zod 4.6.6, pino 10.4.0, prom-client 15.1.3, stripe 23.0.0, argon2 0.45.1, helmet 8.3.0, rate-limiter-flexible 11.2.1, Vitest 5.0.3, Playwright 1.64.0, autocannon 8.0.0.

## ADR-023 — HSTS gated to trusted-cert origins
- Per `reference/security-headers.md` §1, HSTS is **not** sent on the self-signed local deploy (it would pin a browser to an untrusted cert); `HSTS_ENABLED=true` turns it on for a trusted public origin (documented in QUICKSTART "Protocol & TLS"). HTTP is redirect-only at the edge.
- *Superseded for local deploys by ADR-065.*
- *Evolved (Stage 4 reconciliation):* the web container now receives `HSTS_ENABLED` too (it was passed only to api/worker, so pages never sent HSTS even when enabled) — fixed in compose.

## ADR-024 — Simulated environment data is always labelled
- PBR-7: `POST /api/v1/demo/environment` generates a realistic environment through the real ingest pipeline with `sourceKind: simulated`; every entity/event carries `simulated: true` and every surface shows a SIMULATED badge; never presented as live telemetry (Non-Goal "Display fabricated telemetry as live").

## ADR-025 — Dashboard read models
- Overview counters, system-health scores and top risks are computed by the scheduled `health-rollup` job into `health_snapshots` (CQRS-lite), cached in Redis; the overview never aggregates the full graph per request (large-tier read path).

## ADR-026 — Invariants (Stage 1 gate)
- `invariants.json`: 57 entries — 56 machine (forbidden-pattern, boundary-order, unique constraints, status writers, service-floor files, governance, web-baseline, ui-coverage) + 1 manual (scheduled-trigger effects; cap 3).
- Runner pinned: `scripts/invariant-lint.mjs` v1.5 (self-test OK).
- `node scripts/invariant-lint.mjs --prove` closing line:
  `57 total / 56 machine (56 proven, 0 unproven) / 1 manual (skipped -- Reviewer Gate item 8)`
- Plain run at Stage 1 (no source yet): `57 total / 56 machine (8 pass, 48 fail, 41 zero-files) / 1 manual (skipped -- Reviewer Gate item 8)`. These are expected failures; they must reach 0 fail by the Pre-Delivery gate.
- One prover iteration: INV-19 was first written as a quote-class pattern that the prover cannot synthesize, so it was simplified to `\bbcrypt`.

## ADR-027 — Stage 1 gate outcome
- `review_gates: auto` → architecture gate auto-approved; continue to Stage 2.

## ADR-028 — Team & invitations admin surface (F-04) — scope expansion (comprehensive)
## ADR-029 — Ingest observation log and source health (F-05) — scope expansion (comprehensive)
## ADR-030 — Evidence NDJSON export for auditors (F-15) — scope expansion (comprehensive)
## ADR-031 — Billing refunds, disputes and cancel-at-period-end (F-17) — scope expansion (comprehensive)
## ADR-032 — Operations health correlation screen (F-18) — scope expansion (comprehensive)
## ADR-033 — Profile & password management (F-03) — scope expansion (comprehensive)

## ADR-034 — Workflow numbering
- W1–W15 = RESEARCH "Key Workflows" 1–15; W16 = Users & Outcomes Workflow A; W17 = Workflow C. Workflow B is covered by W5+W2 and Workflow D by W9+W10+W15 (no duplicate ids).

## ADR-035 — Stage 2 gate outcome
- `review_gates: auto`. Self-verification passed: every Key Workflow has a §5 entry with a terminal step and a §6 screen; Non-Goals are not specced (governance boundaries are policy-only, there is no LLM chatbot, and simulated data is badged); the scope expansions are logged above.
- ui-coverage parse: `targets: 33 screens, 17 workflows, 89 endpoints` (the six machine/probe endpoints are marked internal).

## ADR-036 — Multi-Agent Plan Gate (auto) and file-disjointness strategy
- `review_gates: auto`, so the plan is not halted on; it auto-dispatches. BUILD-PLAN.md has 8 waves and 18 manifests. FOUNDATIONAL/CRITICAL features (F-01, F-03, F-15, F-14, F-17) run solo in the orchestrator. Waves 4, 6 and 7 dispatch parallel subagents.
- Disjointness changes from SPEC/ARCH defaults:
  - Route and job modules are auto-discovered from `server/src/routes/*.ts` and `server/src/jobs/*.ts`, with no shared index edits.
  - Shared zod contracts are per-area subpath modules.
  - RTK Query uses `injectEndpoints` per feature.
  - F-01 writes the full SPEC §2 schema as the base migration.
- `t_max` = plan start + 2.5 × a 16 h estimate (comprehensive).

## ADR-037 — Generated Prisma client lives in `server/build/prisma`
- The `prisma-client` generator emits TypeScript. Under `server/src/generated` the pinned invariant-lint (which skips only `node_modules`, `dist`, `build`, … and ignores `.gitignore`) matched the generated client against INV-01/02/03, giving 140+ false positives.
- The output moved to `server/build/prisma` (gitignored, dockerignored, excluded from lint). It is imported only by `server/src/db/client.ts`, and `tsc` emits it to `dist/build/prisma`. Invariant scopes are unchanged.

## ADR-038 — Session and reset token hashing; edge auth redirect (F-03)
- SPEC SC-14 says "SHA-256 stored". Session and reset tokens are stored as HMAC-SHA256 keyed with `SESSION_SECRET` (a pepper), so a database read alone cannot be used to confirm guessed tokens. The tokens are 256-bit random, so this is strictly stronger.
- Unauthenticated requests to app routes get a 307 from `web/src/proxy.ts` to `/login?next=…` on `PUBLIC_ORIGIN`. Without this, `(app)/loading.tsx` began streaming before the layout's `redirect()` ran, so the response was a 200 with a meta-refresh. The layout still re-checks the session (a cookie is presence-only at the edge).
- The Caddy access log filters `/reset/<token>` and `/invite(s)/<token>` out of `request.uri` (SC-15).
- A successful login clears the per-account login counter; the per-IP budget is unchanged.

## ADR-039 — Evidence verification anchoring (F-15)
- The hourly `ledger-verify` is incremental. It starts from the latest `verified` checkpoint after re-checking that checkpoint's HMAC and recomputing its record's hash, then walks only newer records and signs a checkpoint at the head. A user-run Verify (`POST /v1/evidence/verify` with no `fromSeq`) walks from genesis, so edits to older records are caught on demand. `fromSeq` runs are partial and write no checkpoint.
- A break upserts a `broken` checkpoint at that seq and raises one critical `integrity` event plus a feed item. A later run that finds the same break does not raise another.
- The hashed content includes `org_id`, `seq` and `created_at`, and `created_at` is set by the service rather than by the DB default. A record cannot be moved between orgs or re-sequenced without breaking the chain.
- The HMAC message is `stardancer-ledger:v1:<org>:<seq>:<hash>`.


## ADR-040 — Proposal risk is derived, not supplied (F-14)
- SPEC §3 has a `risk` column on proposals, but neither proposal endpoint accepts risk in its body. The server derives it: the action's floor (isolate/restore/mitigate low; disable_credential, revoke_relationship and model_update medium) raised to the target's `criticality`, and to at least `high` when the target is a crown jewel.
- `reversible` is a property of the action type. Isolate, restore, disable and model_update have a defined inverse; revoke_relationship and mitigate_vulnerability do not, so they are never auto-approved.
- Validation still fails a `low` proposal whose blast radius reaches a crown jewel (SPEC §5), so a low-criticality target next to a crown jewel cannot be auto-approved.
- Pure functions live in `server/src/engine/proposal-machine.ts`. Approver checks reuse F-13's `engine/policy.ts`.

## ADR-041 — Proposal execution and agent denials (F-14)
- An agent proposal outside its capabilities or above its `max_risk` is still stored as `policy_rejected` with evidence, so the attempt is auditable. The response is 403 `CAPABILITY_DENIED` naming the proposal id (W9.5 "denied"). Every other policy outcome, including no enabled policy, returns 201 with the resulting status.
- The stored `idempotency_key` is sha256(proposer type, proposer id, client key). Two actors in one org can never collide, and the column stays within 64 chars. The Redis idempotency middleware replays first; the DB unique constraint backs it under races and returns the winning proposal.
- Target types are checked during validation. Isolate applies to host, container, endpoint, cloud resource and AI agent. Disable applies to identity and AI agent. Restore applies to the union of both. A missing, foreign or retired target fails validation in exactly the same way, so it reveals nothing about other orgs.
- Execution happens in a single transaction: lock, `assertExecutable`, approved→executing, effect, then succeeded. If the gate fails or an effect's precondition fails, the proposal moves executing→failed with the reason in `result`. A crash leaves it `approved`, and the 1-minute `proposal-sweep` re-drives it. The `executor` BullMQ queue is the normal path.
- `assertExecutable` re-reads the pinned policy (it must be enabled, the same action and the same hash) and re-checks the approval. An auto-approval must still be `autoApprovable` under the current policy. A human approval must come from a current member who passes `checkApprover` against the current role. Any recorded rejection refuses execution.

## ADR-042 — Landing content follows the product, not the template copy (F-02)
- The landing stat tiles show product facts (ledger, approvals, ingest limits) rather than invented customer metrics, which would be placeholders (kernel: no fake data on marketing surfaces).
- The design template's loop is condensed to five stages (ingest → graph → predict → govern → prove) so it reads on a 375 px screen. Plan prices follow ADR-014 ($0 / $490 / $2,490) with no quota claims. The wordmark is rendered inline as text so it inherits the theme tokens.
- *Evolved (Stage 4 reconciliation):* the landing loop uses the five Observe → Understand → Predict → Validate → Act stages, and the stat tiles show four product facts (0 LLM calls, ≤ 60 s freshness, 26 controls, 1 hash chain).

## ADR-043 — Member and invite API details (F-04)
- `MemberDto` carries `id` (the membership id, used in member URLs) and `userId`. A duplicate pending invite returns 400 with field error `email` / code `DUPLICATE` so the form can show it inline, instead of a bare 409.
- Admin reset links: 403 for yourself (use the profile screen), 409 when the user also belongs to another org; issuing a new link expires earlier ones. Accept takes `{name?, password}`. Invite and reset-link responses are `Cache-Control: no-store` and are not Idempotency-Key replayed, because a replay would re-reveal a secret. Revoking a revoked invite is 204; revoking an accepted one is 409 `INVALID_STATE`.
- Known limit: login opens the oldest membership; there is no org switcher (single-org use is the v1 target).

## ADR-044 — Ingest semantics (F-05)
- Key prefixes are `sd_in_` (ingest) and `sd_ag_` (agent) from the shared `server/src/lib/apikey.ts`, not `sd_ingest_` / `sd_agent_`. The `simulated` source kind is not creatable through the API; F-06 uses `ensureSource`.
- Invalid items are not stored; they come back in `rejected[]` by index. Ingest sets criticality, crown jewel, owner and tags only on entity creation and vulnerability status only on first sighting, so analyst edits are not overwritten. A paused source gets 409 `INVALID_STATE`. Backpressure counts queued batch jobs. `ingest.processed` evidence is one record per batch, written after the batch, so the per-org ledger lock is taken once rather than per item.

## ADR-045 — Agent and policy API details (F-13)
- An action type without a policy is listed as `configured: false, enabled: false`, so every proposal for it is rejected. An identical PUT is a no-op (no new version); a concurrent PUT on the same version gets 409.
- Agent keys: the rate limit is per agent; issuing a key to a suspended agent is 409; at most 5 active keys per agent; key issue is limited to 20/h per user and is not Idempotency-Key replayed (it would re-reveal the key). Revoke uses the shared `DELETE /v1/api-keys/:id`. The agent context response carries a few extra read-only fields (agent name, capabilities) for client convenience.

## ADR-046 — Dependency-ready features start early (Stage 3 schedule)
- F-11 (needs F-05), F-16 (needs F-05, F-15) and F-17 (needs F-03, F-15) have every dependency committed, and their files_write sets are disjoint from wave 6. They are dispatched alongside wave 6 instead of waiting for waves 7 and 8. Each agent gets its own test database and Redis db.
- What stays the same: F-17 still gets its own CRITICAL integration checkpoint before commit. F-08, F-10 and F-18 still wait for F-06 and F-09. Each feature is still committed separately and gets its own banner.

## ADR-047 — Simulated environment and constellation details (F-06, F-07)
- The demo environment is seeded by orgId, so it is reproducible, and is sent to `acceptBatch` as one internal job (the SC-9 limit of 500 applies to the HTTP envelope only) so entities land before what references them. An existing or pending environment gives 409. Removal deletes only `simulated` rows and pending observations, never evidence; health snapshots and non-simulated rows that reference simulated entities are kept. There is no status endpoint yet, so settings shows only what a removal deleted.
- The constellation is drawn on a canvas (30 fps at 2,000 nodes) with a node listbox and table as the accessible equivalent. Position, environment and health come from the entities table; topology comes from graph-store. Graph, neighbors, impact and attack paths share one per-user `traversal` limiter (30 points/min, weighted by depth). PATCH also writes an expected `changes` row so edits appear on the timeline.

## ADR-048 — Intelligence, security analysis and prediction details (F-09, F-11, F-12)
- Drift: `from`/`to` also filter changes; the stale list includes overdue entities the sweep has not marked yet; lists are capped at 100 with full counts in the summary; a mass outage writes 10 feed items plus one rollup. Only entities are marked stale.
- Security: `entryId` is an added attack-path filter (W13.3). `truncated` means a frontier, work or time budget was hit, not the requested depth. Traversal is charged before lookup so probing is limited.
- Predictions: a verdict increments the rule's confirmed/refuted counters (accuracy needs them); the weight changes only through an approved `model_update` proposal. Fingerprint = signal + horizon-length bucket; refreshes never move `horizonAt`. System weight proposals are made only when an enabled `model_update` policy exists.

## ADR-049 — Compliance and billing details (F-16, F-17)
- Compliance: third-party services and load balancers are recognised by tags/attributes (no entity type exists); a control with no data fails as "no data" and is excluded from coverage, as are n/a controls; a pending evaluation returns 202 `{queued:false}`; evaluations older than 90 days are pruned; an org's first run fails GV.OC-03 by design.
- Billing: an idempotent replay returns 200 with the original body; checkout is 409 while an active subscription exists (cancel first); failed payments are keyed `<pi|invoice>#<eventId>`; `payment_intent.payment_failed`, `refund.updated` and `refund.failed` are also handled; events for an unknown org are acknowledged and ignored; in simulated mode cancellation completes on `customer.subscription.deleted`.

## ADR-050 — Cross-cutting fixes found while integrating wave 6
- Prisma P2024/P2028 (pool exhausted, transaction could not start) map to 503 `UNAVAILABLE` with `Retry-After: 2` instead of 500, and the default transaction `maxWait` is 5 s. An e2e signup under load had returned 500.
- `loadGraph` filters edges by org and state alone when the node set is not truncated; binding two 5,000-id IN lists dominated a 3.5 s load on a 5,200-node graph (§9 target 800 ms p95). Re-measured at the load test.
- Ingest keeps an entity's `healthScore` at its latest sample when samples in a batch arrive out of order. `Tabs` takes an `idBase` so `aria-controls` points at a real panel (it previously referenced a missing id). Table headers use `--muted`, inline auth links are underlined, and `--dim` moves from the template's #575c66 (3:1) to #7c818c (≈4.8:1) so dim text meets WCAG AA (axe colour-contrast and link-in-text-block; accessibility outranks the template value).
- e2e resolves `localhost` to IPv4: on this Windows host WSL's relay also bound `[::1]` on 8280/8643 and stopped answering, while Docker's IPv4 listener was healthy. The relay is not ours and is left running.
- The live-stream status is decided in an effect, not during render: the server has no `EventSource` and rendered "Offline" where the client rendered "Reconnecting". That hydration mismatch (React #418) made React re-render the whole app and discard text already typed into forms. On phones the topbar's account label is hidden and the org name truncates; a 466 px topbar had widened every page past 412 px, offsetting the visual viewport. e2e runs with reduced motion, which the template honours.

## ADR-051 — Overview, investigations and operations details (F-08, F-10, F-18)
- Overview: `health_snapshots.counters` also stores flow, mini-map summary, investigations and the latest prediction so `GET /v1/overview` is one query (cached 10 s); an empty tenant or a snapshot older than 3 min is computed live. Snapshots are derived metrics and are not written to the evidence ledger; the rollup deletes snapshots older than 7 days in the same transaction. "Critical risks" = entities scored ≥ 70; "active incidents" = open or in-progress investigations. Stream opens are limited to 30/min per session.
- Investigations: foreign or missing records referenced in a body are 400 field errors (the investigation is owned; the reference is input). A closed investigation refuses notes and items (409 `INVALID_TRANSITION`). Reconstruction spans `windowHours` around the first and last item plus 1-hop neighbours. Root cause, reconstruction and correlations charge the shared `traversal` limiter.
- Operations: degraded = health < 70 or ≥ 15 below the window's peak; onset is where health crosses the midpoint between peak and latest. A healthy entity's changes are ranked against now. Responses add `anchor`, `window`, `reasons`, `hops` and `truncated` to explain and bound the ranking.
- *Evolved (Stage 4 reconciliation):* stream opens are limited to 120/min per session (ADR-056).

## ADR-052 — Fixes found at the final integration checkpoint (CP-8)
- `/intelligence` failed on the server: the page called `isIntelTab`, exported from a client module, during render. The tab keys move to `features/intelligence/tabs.ts` (not a client module). The F-09 e2e spec had not run since wave 6; it now runs at every checkpoint.
- Root-cause evidence links are 24 px tall so wrapped links meet WCAG 2.2 target size on phones. The axe helper now names the failing nodes.
- INV-56 matches `test('W<n> …')`: the W6 and W9 specs are renamed, and W1 (source → key → ingest → Constellation) and W11 (verify → export) get their own specs. The W7 spec asserts order rather than an exact row count, since ingest also records "first observed" changes.
- `@vitest/coverage-v8` is added; `server` runs `vitest run --coverage` with a 70% statement threshold over lib/services/engine. CI runs `npm run format`.
- Migrations are forward-only (Prisma has no down migrations); the schema is one greenfield `init` migration. Rollback is restore-from-backup, taken before `migrate deploy` (QUICKSTART §4).

## ADR-053 — Web baseline against the compose stack
- **Fixture:** SPEC §6 example routes are fixed ids (`…0101`, `…0301`, …). `e2e/baseline-fixture.mjs` (local stack only) signs up a demo owner and generates the medium demo through the API. It creates an investigation, an agent, a prediction run, a proposal and a checkout, then re-keys one row per table to its example id: a copy under the fixed id, references repointed, the original deleted, all in one transaction. Credentials go to the gitignored `artifacts/baseline-account.env`. `controls` is the global catalogue, so its re-key applies to every tenant of that local database.
- **Build identity:** the stack serves the build made inside the web image, not the host's `web/.next`. The fixture copies the image's `BUILD_ID` (mtime kept) to `web/.next-served/BUILD_ID` (`config.buildIdFile`), so `buildId` and `buildFreshness` judge the served build.
- **TLS:** the runner launches stock Chromium and Node `fetch` with certificate checks on. On this host the Caddy local root must be trusted (QUICKSTART, Protocol & TLS), and Node gets `NODE_EXTRA_CA_CERTS` pointing at the exported root. TLS is never disabled for the runner.

## ADR-054 — Hero constellation sphere (human direction)
- **Context:** the human supplied a reference image of a network held in a glass sphere and asked for that interface. It shows a dense red core and a cyan/violet shell of linked nodes.
- **Decision:** `ConstellationSphere` is a canvas drawn on top of the template's ambient `#field`, on the hero's right side. Its 3D layout is deterministic: a Fibonacci shell of 34 nodes, each linked to its 3 nearest; an octahedral red core, fully linked and tied to the shell. It turns slowly and leans toward the pointer. Its colours are template tokens only (`--red`, `--cyan`, `--violet`, `--white`).
- **Constraints kept:** decorative (`aria-hidden`, `pointer-events: none`). One static frame under reduced motion. Paused when off screen or the tab is hidden. Below 1100px it sits dimmed behind the hero copy, which stays on top.

## ADR-055 — "I AM" overview merge (human direction)
- **Context:** the human uploaded a standalone "Stardancer — I AM" HTML design and asked to merge it. It has a glass System Graph orb with selectable regions and a Ψ core, a selection card, a five-stage loop pipeline, stat chips, a sample "ask" answer, fictional agents, Kernel and CIPHERZER0 workspaces, its own palette, and Google Fonts.
- **Decision:** merge it into `/overview`, the post-login home. The design maps onto data the overview already has. `SystemGraph` is a CSS-3D glass orb: each region is an entity type with its live count, and the Ψ core is the whole system. Both are `aria-pressed` buttons that drive a selection panel (entities, share, rank, link to the constellation). `LoopPipeline` makes the loop counters selectable stages, each with a description and a link. The page gets the "01 · I AM — Every enterprise is a constellation." head.
- **Not merged:** the parts that would be fiction. These are the canned LLM answer (the product makes no LLM calls), the invented agents and workspaces, the "inferred links" toggle (no inferred-edge data), the embedded image, its palette (mapped to template tokens, INV-17/18), and Google Fonts (CSP; self-hosted fonts stay).
- **Constraints kept:** the AppShell h1 "Overview" and the W16 bindings (counters group, labelled graph image, "The loop" region). The 3D decoration has `pointer-events: none`; without it, the rotated rings won hit-testing over the region buttons, and W16 now clicks a region in a real browser. Motion stops under reduced motion.

## ADR-056 — Web-baseline remediation (0.1.1)
- **Zod JIT off** (`web/src/instrumentation-client.ts`, which Next runs before app code): Zod 4 probes `new Function("")` to decide whether it may JIT, which the strict CSP (no `'unsafe-eval'`) reports as a violation on every screen. `jitless` keeps validation identical and the CSP unchanged.
- **Edge on loopback** (`EDGE_BIND`, compose): see QUICKSTART Troubleshooting (WSL `[::1]` relay). HSTS stays off locally (ADR-023), so the runner's `headers` cell is a contract residual.
- Accessibility/stability: no `autoFocus` on auth forms (it skipped the skip link), 44px zoom and menu targets, valid `outline: var(--focus)`, org settings render their layout while loading (no CLS), stream opens 120/min/session. Token routes are dynamic in the baseline config (never in the sitemap); the fixture seeds the SPEC example invite/reset tokens.

## ADR-057 — Review Round 1 fix batch (0.1.5)
- **Out-of-order relationships wait, not fail:** a relationship whose endpoints are unknown stays `received` ("Waiting for entity …"); any batch that creates entities retries the org's waiting relationships, and the drift sweep rejects ones still waiting after 24 h. Senders batch entities and edges independently, so rejecting on first sight lost real edges (Reviewer item 6). Rows are now applied in one transaction each.
- **Relationship staleness has a writer:** the drift sweep marks relationships unseen within the org's window `stale` (evidence `relationships.stale`); re-observation restores `active`. The health engine's stale-edge penalty is no longer dead code.
- **Streams follow the session:** revocation (logout, password change/reset, member removal) is announced on Redis `sessions:revoked`; each replica re-checks its streams then and on every ping, and ends any whose session, membership or user is gone. One user holds at most 6 streams per replica; a new one closes their oldest (security S-4).
- **Mirror Brain exploit_path follows attacker reach** (the attack-path engine's forward walk), not failure propagation; per-rule caps apply after sorting by confidence; expiry runs before dedup so an expired forecast can be re-raised in the same run; root cause never ranks or walks through retired entities or edges.

## ADR-058 — Review Round 2 remediation
- **Entity list cost is bounded** (security pass 2, N-1): risk is computed, so `GET /v1/entities` still ranks in Node, but every 5 000-row chunk it reads is charged to a per-org budget (240 chunks/min, 429 with Retry-After), and one request reads at most 40 chunks (200 000 rows). In an org above that size, matches past the cap are not ranked and the response does not flag it (settle check R3-N1). A persisted risk column with an index is the way to remove the cap, recorded for the next release.
- **The drift sweep retries waiting relationships before expiring them**: a retry can miss a row while two batches run at once, so each sweep catches what was missed.
- **Design residuals**: Trust function bars had zero height (a global `flex: 1` inside a column flex); "N of M satisfied" tallies of a failing control no longer show a green ✓; on phones a panel's meta line always takes its own row, so a web-font swap cannot shift the page (Members CLS); one filled primary in billing; prediction rows show the rule kind.

## ADR-059 — Lighthouse at the definitive baseline
- `lighthouse` was never installed, so every earlier baseline ran `--no-lighthouse`. The first `--full` run exited on the missing dependency. It is now a root devDependency (13.5; `npm audit --omit=dev` still 0).
- Run 13: accessibility, best-practices and SEO 100 (overview SEO is noindex by contract). LCP 2.1 s / 1.7 s, CLS ≤ 0.003. Performance landing 74, overview 69, against the 80 floor. The cost is Total Blocking Time (~1.5 s).
- Measured, then fixed what the evidence supported. Under 4× CPU throttling, removing every CSS animation and canvas left TBT unchanged, so decoration is not the cause. The long tasks are hydration script evaluation and first layout. On overview a further ~0.8 s layout follows the client-side `/v1/overview` fetch.
  - The Redux/RTK Query provider moved from the root layout to `(app)`, `(auth)` and `invite`. The landing and legal pages no longer ship RTK or immer (78 KB).
  - `instrumentation-client` imports only zod core's `config`. `shared` is marked `sideEffects: false`.
  - A hidden (phone) landing sphere never starts its loop.
- Same-build TBT ranged 0.8–2.0 s between runs on this host (4 CPUs shared with ~30 other projects' containers). The remaining gap is logged as a residual. Next steps, recorded for the next release: server-render the overview data (removes the fetch-then-render waterfall) and lazy-hydrate below-the-fold landing islands.
- **Compliance posture at the wide close**: A ✓ 23 / B ⚠ 3 / C ✗ 0 (COMPLIANCE.md). The ⚠ rows are SC-14 (HMAC token storage, ADR-038), HSTS off on `localhost` (ADR-023) and Lighthouse performance (above).

## ADR-060 — Design Quality Gate closed under auto with residual posture
Round 3 (fresh-context, commit 572f7b9): **A ✓ 13 / B ⚠ 18 / C ✗ 2** over 33 screens. The two ✗ are landing and
overview, both from the Lighthouse performance floor (72 and 67 vs 80); they are **not accepted** — the floor is not
shown to be unsatisfiable — and stay reported residuals with the ADR-059 next steps. The 18 ⚠ are listed in
DESIGN-NOTES.md. Under `review_gates: auto` the build proceeds to Stage 4.

## ADR-061 — `content-visibility: auto` on below-the-fold landing and overview sections
- **Context:** INV-55 fails on landing/overview Lighthouse performance (72/67 < 80). Traces show the long tasks are
  first layout of the whole DOM (~1 s on landing before FCP, ~0.5–0.8 s on overview after the data fetch), not script.
- **Decision:** landing `.block`/`.principle` sections and the overview's lower panel grid get `content-visibility: auto`
  with `contain-intrinsic-size: auto <h>`, so the browser skips their style/layout/paint until they near the viewport.
  Anchors, find-in-page and the accessibility tree are unaffected; offscreen size changes do not count as CLS.
- **Consequence:** first layout covers only the hero/KPI area. Re-measured by the web baseline.

## ADR-062 — Load-test limiter scale and build-time provenance
- **Context:** the Stage 4 load test must measure the service, not the limiter (BUILD Load Test Methodology), and every
  tier must report the build identity of the shipped commit. Compose passed `APP_VERSION`/`GIT_COMMIT` at run time,
  overriding the image values, so `/api/health` reported `unknown`.
- **Decision:** `RATE_LIMIT_SCALE` (integer, default 1) multiplies every `createLimiter` budget. `env()` refuses any
  value ≠ 1 unless `PUBLIC_ORIGIN` is a loopback origin, so it cannot be enabled on a served deployment. The load
  procedure sets it, runs, restores 1 and proves the restore by firing past login-ip (QUICKSTART "Load test").
  `APP_VERSION`/`GIT_COMMIT` are build args only (Dockerfile `ARG` → `ENV` + OCI labels); compose no longer overrides
  them at run time, and `docker compose up --build` takes `GIT_COMMIT=$(git rev-parse --short HEAD)`.
- **Consequence:** health and the image `org.opencontainers.image.revision` label agree with the shipped commit; the
  web tier is identified by its label and the Next `BUILD_ID`, Caddy by its stock image version.


## ADR-063 — Decorative motion starts after load + idle
- **Context:** baseline run 16 left INV-55 failing (Lighthouse performance: landing 72, overview 69). Attribution: on
  the landing page the 21 KB canvas chunk (sphere + field) uses ~5 s of main thread, because every frame builds a
  radial gradient per node. On the overview, page-level style/layout/paint (~6.5 s) comes from orb animations that
  are not composited (SVG `stroke-dashoffset`, an animated `box-shadow`).
- **Decision:** `afterLoadIdle()` (`web/src/lib/idle.ts`: window `load` → `requestIdleCallback`, timeout 2.5 s) gates
  all decorative motion. The landing canvases draw one static frame at once and start their loop afterwards. The
  orb's CSS animations hold `animation-play-state: paused` until the stage gets `data-live`. The sphere now blits
  cached sprites: one halo per tone, plus a backdrop for the glass and core glow repainted on resize. The core pulse
  cross-fades two fixed glows with opacity, which is composited.
  `prefers-reduced-motion` behaviour is unchanged.
- **Consequence:** the first paint is identical and motion begins about 1–2 s later. Hydration and first layout no
  longer compete with frame work. Re-measured by the web baseline.

## ADR-064 — Database pool resilience and drain order
- **Context:** the Stage 4 smoke run restarted the worker. The database-outage step restarts Postgres, which
  terminates every backend; node-postgres emits `error` on a checked-out client that has no listener, and the
  process died. During the load-test backlog the worker also failed ingest batches with "Unable to start a
  transaction in the given time": its queues allow 20 concurrent jobs (ingest 8, ARCH §4) against a pool of 10.
  On SIGTERM the API closed its pools while in-flight requests were still running.
- **Decision:** `server/src/db/pool.ts` builds the `pg.Pool` that Prisma's adapter uses. It attaches an `error`
  listener to the pool and to every client at connect, logs the error and lets the query reject; the pool discards
  the dead client. Pool size is 20 per process (3 × 20 = 60 < Postgres `max_connections` 100). On shutdown the API
  waits for `server.close()` to finish in-flight requests before closing queues, pools and Redis, inside a 9 s force
  timer under compose's stop budget. An empty `BILLING_MODE` falls back to `simulated` like the other optional
  variables.
- **Consequence:** a database restart costs a failed query, never a process. Ingest batches no longer starve for
  connections under backlog, and an in-flight request completes across a rolling stop.

## ADR-065 — HSTS on the local deploy (supersedes ADR-023 locally)
- **Context:** ADR-023 kept HSTS off with `tls internal` so a browser would not pin an untrusted certificate. The
  frontend audit (INV-55) fails every HTTPS page without the header. The human chose to turn it on (2026-10-13),
  and the Caddy local CA is trusted in the OS store (QUICKSTART "Protocol & TLS").
- **Decision:** `.env.example` sets `HSTS_ENABLED=true`. Browsers key HSTS by host, not port, so on a loopback
  `PUBLIC_ORIGIN` both tiers send `max-age=86400` without `includeSubDomains`; a public host gets one year with
  `includeSubDomains` (`hstsPolicy` in the API, `hstsValue` in the web proxy).
- **Consequence:** for up to a day after a visit, the browser upgrades every `http://localhost:<port>` to HTTPS,
  including other local dev servers. QUICKSTART documents the opt-out (`HSTS_ENABLED=false`) and how to clear the
  browser entry.

## ADR-066 — Server-rendered first overview snapshot
- **Context:** baseline run 19 (host freed to 15 containers) still left INV-55 failing: overview performance 46,
  mobile LCP 6.6 s. The Lighthouse LCP breakdown gives TTFB about 0.1 s and an element render delay of more than 6 s.
  The LCP element (a populated overview, or the empty-state copy for an empty tenant) renders only after
  hydration, followed by a client `GET /v1/overview`. Under 4× CPU throttling that fetch starts about 6 s into the
  load.
- **Decision:** `web/src/app/(app)/overview/page.tsx` reads the first snapshot on the server through
  `serverGet()` (`web/src/lib/server-api.ts`). This is the same cookie-forwarding internal read that the authed
  layout already uses for the session gate. The page passes the snapshot to `<Overview initial>`, which renders it
  until RTK Query resolves. Any server-read failure returns null, and the page falls back to the existing skeleton
  and error state. `<When>` and the prediction horizon `<time>` use `suppressHydrationWarning`, because locale,
  time zone and "now" differ between the server and the browser. ARCH §2 records the exception.
- **Consequence:** the overview content is in the first HTML paint, so LCP no longer waits for hydration. Each
  overview render does one extra internal API read (about 0.07–0.9 s server time), and the read is tenant-scoped by
  the forwarded session exactly like the client call. Live refreshes are unchanged (SSE hints → RTK refetch).

## ADR-067 — Mission Control overview and the "// LIGHT" motif
- **Context:** the human asked (2026-10-14) to enhance the demo with the "Mission Control" reference image and the
  "I AM // LIGHT" motif (two overlapping wireframe spheres, warm and cool). SPEC §6 already names the overview
  panels; the change is presentation only. INV-55 fails on performance, so nothing may add script weight.
- **Decision:** the overview becomes a Mission Control grid. Counters sit in the head of one "System constellation"
  panel that also holds the orb and its selection (span 8). The intelligence feed is a compact column beside it
  (span 4, newest 8, filters and live toggle kept, "View all in Intelligence" replaces "Show more"). The loop follows.
  Health, top risks, investigations (now with status pills) and prediction & impact form a four-panel row. Prediction
  leads with its confidence as the probability and lists impact from real fields (affected, crown jewels, origin, time
  to `horizonAt`); there is no trend series, so no sparkline is drawn. Hubs get one stroked icon per entity type. The
  orb sits over a warm and a cool sphere, and the landing gains a "// Light" section with the same pair and the eight
  domains from the image. All of it is static CSS and inline SVG.
- **Consequence:** no API, contract or behaviour change; no new client script. The empty-tenant overview keeps the full
  feed. Landing section numbers are unchanged (the new section is unnumbered).

## ADR-068 — Speed pass: self-hosted Latin fonts, no zod on the landing, no below-the-fold skipping on the landing
- **Context:** the human approved C-27 and chose a speed pass (2026-10-14) so that INV-55 (mobile Lighthouse
  performance ≥ 80) can pass. In baseline run 21 the landing scored 72 and the overview 71; LCP was fine and Total Blocking Time was the
  cost. Traces showed: six webfont files (the Greek subsets were pulled in by Ψ/Λ/Ξ), each swap re-laying out the
  whole page; next/font's size-adjusted `local('Arial')` fallback faces making the first layout cost 0.5–1 s on
  Windows; the landing's `content-visibility: auto` sections all being laid out at once after first paint (one
  400+ ms task); the root error boundary importing `isProblem` from `@stardancer/shared/common`, which builds zod
  schemas at load, so every page (landing included) shipped ~100 KB of zod; sidebar links prefetching every route;
  `fmtDateTime` building a new `Intl.DateTimeFormat` per call.
- **Decision:** fonts come from `next/font/local` with Latin-only variable files copied from `@fontsource-variable`
  (SIL OFL, licences in `web/src/fonts/`), preloaded, `display: optional`, no fallback size adjustment; glyphs outside
  Latin fall back to the system font. The landing drops `content-visibility` (supersedes ADR-061 there; the overview
  keeps it). Client code imports only types from zod-bearing shared modules where a value is not needed: `isProblem`
  lives in `web/src/lib/problem.ts`, and `HEALTH_AREAS`, `HEALTH_AREA_LABEL` and `LOOP_STAGES` move to the zod-free
  `shared/src/enums.ts` (still re-exported from `overview.ts`). Sidebar links use `prefetch={false}`. Date formatting
  reuses one formatter.
- **Consequence:** no API, contract or behaviour change. On a first visit over a slow link a face that misses the short
  `optional` block period is skipped for that page view and the system font shows. That is a visible deviation from the
  design template's typography on cold loads (DESIGN-NOTES). Navigating to a section from the sidebar fetches it on
  click instead of ahead of time. The `@fontsource-variable` packages are no longer imported.
- **Addendum (after baseline run 22: landing 82 pass, overview 77):** the overview still loaded zod through the demo
  form. `DEMO_SIZE_LABEL` moves to `enums.ts` (re-exported from `demo.ts`), and the form imports `demoGenerateBody`
  when it is submitted. Validation is unchanged: the same schema still runs client-side before the request. Overview
  links (feed rows, panels, orb) also use `prefetch={false}`, because the feed alone put a dozen links in the first
  viewport and decoding their prefetched payloads competed with hydration.

## ADR-069 — Observation intake as one jsonb-parameter insert
- **Context:** the v0.1.7 follow-up profiled the demo POST (11–15 s for `large`, answering 202 "queued"). Generating
  and validating the 7.7k observations costs 0.3–2 s of CPU. Prisma's `createManyAndReturn` for the same rows took
  14–21 s inside the API container, against ~3 s for the same rows sent as one `jsonb_to_recordset` parameter (both
  inside a rolled-back transaction, on the same contended host). Every `POST /v1/ingest` batch goes through the same
  `acceptBatch` insert.
- **Decision:** `acceptBatch` (`server/src/services/ingest.ts`) inserts accepted observations with one parameterised
  `INSERT … SELECT FROM jsonb_to_recordset($rows::jsonb) ON CONFLICT DO NOTHING RETURNING id` through tagged
  `$queryRaw`. The org and source are bound parameters, so the statement stays tenant-scoped, and INV-03 (no unsafe raw
  SQL) still holds. Conflict semantics are unchanged: an existing (org, source, idempotency key) is skipped, not
  returned, and counted as a duplicate.
- **Consequence:** no API, contract or behaviour change; intake time drops by roughly the Prisma bind overhead. The
  demo POST stays synchronous through intake (its response reports accepted counts); graph processing remains in the
  worker.

## ADR-070 — Server-side reads carry the caller's address
- **Context:** reviewer round 4 (R4-N1, medium). `serverGet()` (the authed layout's session gate and the overview's
  first snapshot, ADR-066) called api-1 from the web container without the browser's address. The API's global
  limiter keys on `req.ip`, so every user's server-side reads shared the web container's one 600/min bucket. A 429
  there failed the session gate, and the layout sent signed-in users to /login.
- **Decision:** `serverGet()` forwards the rightmost `X-Forwarded-For` entry it received as `X-Forwarded-For`. Caddy
  is the only published port and overwrites that header with the client address. The API trusts exactly one proxy
  hop (`trust proxy 1`), so it keys the request on the browser's address, the same as the browser's own calls through
  Caddy. A request without the header (local dev) behaves as before. Same review: the web proxy now accepts the API
  env schema's truthy values for `HSTS_ENABLED` (`true` | `1`, R4-N3). The demo form submits even if its lazily
  loaded schema chunk fails, and the API's field errors still map onto the control (R4-N4). Topbar sign-out
  swallows a network failure instead of leaking an unhandled rejection; it still clears the session and leaves.
- **Consequence:** one user's burst can no longer exhaust the limit for everyone's server-rendered pages, and each
  browser's budget now counts its server-side reads (two per overview load). The web container cannot set an
  arbitrary address, because the API is not reachable from outside the compose network.
