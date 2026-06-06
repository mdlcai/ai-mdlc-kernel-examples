# SPEC.md — MCP Sentinel

> Behavioral contract (Stage 2). Source of truth for **behavior**. Build depth:
> **comprehensive** — every state transition, error path, race/idempotency
> guarantee, input-validation boundary, and trust transition is specified.
> Complete enough that another agent could recreate the system from this alone.
> Precedence: RESEARCH > ARCHITECTURE > SPEC.

## 1. Feature inventory (vertical slices)

| ID | Feature | Surface | Key Workflow |
|----|---------|---------|--------------|
| F-00 | Project scaffold, config, DB, health, design tokens | infra | — |
| F-01 | Auth: register / login / logout / session | UI + API | (foundation for all) |
| F-02 | SafeConnector (SSRF-guarded egress) | backend | (foundation for scan) |
| F-03 | McpClient enumeration (Streamable HTTP + fallback) | backend | W1 |
| F-04 | Rule engine + detectors + grader | backend | W1 |
| F-05 | Scan orchestration + report persistence API | API | W1 |
| F-06 | Dashboard + new-scan + scan-in-progress UI | UI | W1 |
| F-07 | Report view + findings list UI | UI | W1, W2 |
| F-08 | Finding drill-down + raw JSON-RPC viewer | UI | W2 |
| F-09 | Endpoint history (timeline + grade trend) | UI + API | W4 |
| F-10 | Re-scan + diff/compare | UI + API | W3 |
| F-11 | Report export / shareable link | UI + API | W5 |

## 2. Key Workflows (from RESEARCH §Users) — each has exactly one e2e test (`W<n>`)

- **W1 — Scan a server.** User pastes an MCP endpoint URL (+ optional token),
  clicks Scan → Sentinel connects, enumerates, runs checks, renders a graded
  report (posture grade + severity-ranked findings). Terminal artifact: the
  rendered report screen with a grade and ≥0 findings.
- **W2 — Drill into a finding.** Expand a finding → see evidence (offending
  description / over-broad permission / echoed secret), the rule that flagged it,
  and remediation; raw JSON-RPC one click away. Terminal: evidence panel + raw
  envelope visible.
- **W3 — Re-scan and compare.** Re-run a scan on a prior endpoint → diff vs last
  result, highlighting new / resolved / changed findings + mutated tool
  definitions. Terminal: diff view rendered.
- **W4 — Review history.** Open an endpoint's history → grade-over-time + scan
  timeline. Terminal: history screen with ≥1 scan and a trend.
- **W5 — Self-audit / share.** Scan own endpoint and export or copy a shareable
  report link. Terminal: export downloaded / share link created and resolvable.

## 3. Data model (PostgreSQL DDL — authoritative)

```sql
-- users
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email         TEXT NOT NULL,
  password_hash TEXT NOT NULL,          -- argon2id; never the password
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX users_email_unique ON users (lower(email));  -- INV-6

-- sessions (server-side; token stored hashed)
CREATE TABLE sessions (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash TEXT NOT NULL,            -- sha256 of the opaque cookie token
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX sessions_token_unique ON sessions (token_hash);
CREATE INDEX sessions_user_idx ON sessions (user_id);

-- endpoints (one per normalized URL per user)
CREATE TABLE endpoints (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  url            TEXT NOT NULL,        -- as entered (no token)
  url_normalized TEXT NOT NULL,        -- scheme+host+port+path, lowercased host
  label          TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX endpoints_user_url_unique ON endpoints (user_id, url_normalized);

-- scans
CREATE TYPE scan_status AS ENUM ('queued','running','completed','failed');
CREATE TYPE grade AS ENUM ('A','B','C','D','F');
CREATE TABLE scans (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id  UUID NOT NULL REFERENCES endpoints(id) ON DELETE CASCADE,
  user_id      UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status       scan_status NOT NULL DEFAULT 'queued',
  grade        grade,
  rule_version TEXT NOT NULL DEFAULT '1.0.0',
  error_code   TEXT,                   -- machine code on failure
  error_detail TEXT,                   -- human, redacted
  started_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  finished_at  TIMESTAMPTZ,
  duration_ms  INTEGER
);
CREATE INDEX scans_endpoint_idx ON scans (endpoint_id, started_at DESC);
CREATE INDEX scans_user_idx ON scans (user_id);

-- findings
CREATE TYPE severity AS ENUM ('critical','high','medium','low','info');
CREATE TABLE findings (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id           UUID NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
  rule_id           TEXT NOT NULL,     -- e.g. 'tool-poisoning'
  severity          severity NOT NULL,
  title             TEXT NOT NULL,
  explanation       TEXT NOT NULL,
  remediation       TEXT NOT NULL,
  evidence_redacted TEXT,              -- offending snippet, secrets masked
  location          TEXT,              -- e.g. 'tools[3].description'
  framework_ref     TEXT,              -- e.g. 'OWASP MCP03'
  fingerprint       TEXT NOT NULL      -- stable id for diffing: hash(rule_id+location)
);
CREATE INDEX findings_scan_idx ON findings (scan_id);

-- inventory snapshot (for rug-pull diff)
CREATE TYPE item_kind AS ENUM ('tool','resource','resource_template','prompt');
CREATE TABLE inventory_items (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id             UUID NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
  kind                item_kind NOT NULL,
  name                TEXT NOT NULL,
  definition_hash     TEXT NOT NULL,   -- sha256 of canonical definition
  definition_redacted JSONB NOT NULL   -- canonicalized, secrets masked
);
CREATE INDEX inventory_scan_idx ON inventory_items (scan_id);

-- scan diffs
CREATE TABLE scan_diffs (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id      UUID NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
  prev_scan_id UUID REFERENCES scans(id) ON DELETE SET NULL,
  added        JSONB NOT NULL DEFAULT '[]',   -- finding fingerprints new this scan
  resolved     JSONB NOT NULL DEFAULT '[]',   -- present before, gone now
  changed      JSONB NOT NULL DEFAULT '[]'    -- inventory items whose hash changed
);
CREATE UNIQUE INDEX scan_diffs_scan_unique ON scan_diffs (scan_id);

-- shareable report links
CREATE TABLE share_links (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scan_id    UUID NOT NULL REFERENCES scans(id) ON DELETE CASCADE,
  user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token      TEXT NOT NULL,            -- opaque, URL-safe
  expires_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX share_links_token_unique ON share_links (token);

-- audit log
CREATE TABLE audit_log (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    UUID REFERENCES users(id) ON DELETE SET NULL,
  action     TEXT NOT NULL,           -- 'auth.login','scan.create','share.create',...
  target     TEXT,
  ip         INET,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX audit_user_idx ON audit_log (user_id, created_at DESC);
```

**No table stores a user MCP auth token** (INV-3). Inventory + findings store
only redacted content (INV-3/T6).

## 4. Security requirements (mapped to COMPLIANCE.md IDs)

- **SEC-1 (SSRF, A10):** every user-host connection validated at resolved-IP layer
  + IP-pinned; deny loopback/private/link-local/ULA/metadata/IPv4-mapped; reject
  private-IP redirects; max 3 hops. Egress only via SafeConnector. (INV-1/INV-2)
- **SEC-2 (Bounded consumption, LLM10):** connect timeout 5s; total scan deadline
  10s (configurable); response byte ceiling 5 MB; JSON depth ≤ 64; SSE read
  deadline; per-user scan concurrency ≤ 3.
- **SEC-3 (Token handling, MCP01/LLM02):** MCP auth token request-scoped only,
  never persisted, never logged (redacted), never attached across cross-host
  redirect, redacted from evidence/reports.
- **SEC-4 (ReDoS, LLM10):** detection matching via `re2` on length-capped input
  (≤ 32 KB per field). (INV-9)
- **SEC-5 (AuthN/Z):** argon2id; sessions httpOnly+Secure+SameSite=Lax cookies;
  protected routes reject unauthenticated (401); per-user data isolation via
  `withUserScope` (INV-5). Anti-enumeration: register with existing email returns
  the same shape as new; login failure returns one envelope regardless of cause;
  rate-limit fires before hash compare.
- **SEC-6 (Rate limiting):** per-IP global + per-user; auth endpoints `(ip,email)`
  keyed; scan endpoint cost-limited (≤ 10/min/user, ≤ 3 concurrent). (INV-12)
- **SEC-7 (CSRF/Origin):** all state-mutating verbs require valid `Origin`/`Referer`
  match + double-submit CSRF token. (INV-13)
- **SEC-8 (Breached-password screening, NIST 800-63B):** HIBP k-anonymity range
  check on register/password-set; reject min length < 8; accept up to 64; no
  composition rules; no forced rotation.
- **SEC-9 (Secret-at-rest / XSS):** detected secrets masked before persistence;
  all MCP-origin strings escaped on render, never `dangerouslySetInnerHTML` (INV-8);
  CSP header set.
- **SEC-10 (Audit logging):** security-relevant actions written to `audit_log`
  with user, action, target, IP, timestamp (no secrets).

## 5. API surface (REST/JSON; errors are RFC 9457 `application/problem+json`)

All `/api` responses on error use:
```json
{ "type":"https://sentinel/errors/<code>", "title":"<human>", "status":<int>,
  "detail":"<human, redacted>", "code":"<MACHINE_CODE>", "errors": { "<field>":"<msg>" } }
```
`errors` is present only for 422 validation failures (field-level).

| Method | Path | Auth | Body / params | Success | Errors |
|--------|------|------|---------------|---------|--------|
| GET | `/api/health` | none | — | `200 {status,db,version,uptime_s}` | 503 if db down |
| POST | `/api/auth/register` | none | `{email,password}` | `201 {user:{id,email}}` + session cookie | 422 invalid; 409→returned as 201-shape (anti-enum) actually `200`-neutral; 429 |
| POST | `/api/auth/login` | none | `{email,password}` | `200 {user}` + cookie | 401 `AUTH_FAILED` (uniform); 429 |
| POST | `/api/auth/logout` | session | — | `204` | 401 |
| GET | `/api/auth/me` | session | — | `200 {user}` | 401 |
| POST | `/api/scans` | session | `{url, authToken?}` | `201 {scan:{id,status}}` | 422 bad url; 400 `SSRF_BLOCKED`; 429 |
| GET | `/api/scans` | session | `?endpointId&limit&cursor` | `200 {scans:[...], nextCursor}` | 401 |
| GET | `/api/scans/:id` | session | — | `200 {scan, findings, inventory, diff}` | 401; 404 |
| POST | `/api/scans/:id/rescan` | session | — | `201 {scan}` (new scan on same endpoint) | 401; 404; 429 |
| GET | `/api/endpoints` | session | — | `200 {endpoints:[{...,latestGrade,scanCount}]}` | 401 |
| GET | `/api/endpoints/:id` | session | — | `200 {endpoint, scans:[grade,started_at]}` (history+trend) | 401; 404 |
| DELETE | `/api/endpoints/:id` | session | — | `204` (cascades scans) | 401; 404 |
| POST | `/api/scans/:id/share` | session | `{expiresInDays?}` | `201 {url, token, expires_at}` | 401; 404 |
| GET | `/api/share/:token` | none | — | `200 {scan, findings, inventory}` (read-only, redacted) | 404; 410 expired |
| GET | `/api/scans/:id/export` | session | `?format=json` | `200` file download (redacted report) | 401; 404 |

**Scan lifecycle (async-ish):** `POST /api/scans` creates the scan `queued`,
then runs it synchronously within the request when under the deadline (scans are
≤10s by SPEC); status transitions `queued→running→completed|failed`. The client
GETs `/api/scans/:id` to render. (A worker queue is unnecessary at this scale;
the deadline bounds request time — ADR in DECISIONS if revisited.)

## 6. State transitions & error paths (comprehensive)

**Scan state machine:**
```
queued ──(SafeConnector validates)──► running ──(enumerate+rules ok)──► completed
   │                                      │
   │ url invalid / SSRF blocked           │ connect timeout / total deadline /
   ▼                                      │ transport error / non-MCP server /
 failed(SSRF_BLOCKED|INVALID_URL)         ▼ byte-cap exceeded
                                       failed(<code>)
```
Failure `error_code` values: `INVALID_URL`, `SSRF_BLOCKED`, `UNREACHABLE`,
`TIMEOUT`, `NOT_MCP` (handshake failed / not JSON-RPC), `PROTOCOL_ERROR`,
`RESPONSE_TOO_LARGE`, `TOO_MANY_REDIRECTS`, `INTERNAL`. Each maps to a
human, redacted `error_detail` and renders as a designed error state (not a
stack trace). A `failed` scan still produces a report row so history shows the
failure.

**Empty-but-valid result:** a server that enumerates successfully but exposes
zero tools/resources/prompts → `completed`, grade `A`, findings empty → the
report renders an explanatory empty state ("0 items exposed — nothing to flag",
not a blank panel). Distinguished from a failed scan.

**Idempotency / races:**
- Endpoint find-or-create uses `INSERT ... ON CONFLICT (user_id, url_normalized)
  DO NOTHING RETURNING` then select — no duplicate endpoints under concurrent
  scans (DB-level, not app check).
- Concurrent scans of the same endpoint each get a distinct `scans` row; the diff
  for scan S compares against the most recent *completed* scan with
  `started_at < S.started_at` (deterministic ordering by `started_at, id`).
- Share-link tokens are unique (DB constraint); collisions retry.
- Session token rotation on login; logout deletes the session row.

**Input validation boundaries (zod):**
- `url`: must parse as URL, scheme ∈ {http, https}, host present, no userinfo
  (`user:pass@`) component, length ≤ 2048. Port optional. (Resolved-IP validation
  happens in SafeConnector, not at the zod layer.)
- `email`: RFC-ish, ≤ 254 chars, lowercased for uniqueness.
- `password`: 8–128 chars; HIBP-screened; no composition rules.
- `authToken`: ≤ 4096 chars, opaque; never persisted.
- `expiresInDays`: 1–90.

## 7. Structured error contract (consumed by UI)
422 responses carry `errors: { field: message }`; the frontend binds each message
to the offending control (e.g. the URL field), never a single generic line. 400
`SSRF_BLOCKED` renders a specific, non-leaky message ("That host resolves to a
private/internal address and can't be scanned."). 401 → redirect to login. 5xx →
designed error surface with retry.

## 8. Detection rules (deterministic; `rule_version` recorded)

| rule_id | Severity (default) | Signal | Framework |
|---------|-------------------|--------|-----------|
| `tool-poisoning` | high→critical | Imperative-override / hidden-instruction / data-exfil phrases + invisible-Unicode tags in tool/prompt descriptions, length-capped, `re2` matched | OWASP MCP03 / LLM01 |
| `excessive-scope` | medium→high | Wildcard scopes, shell/fs/network tools with unbounded params, broad permission hints | MCP02 / LLM06 |
| `secret-leak` | high | Known-prefix (`sk-`,`ghp_`,`AKIA`,JWT) or high-entropy tokens in tool output / resource contents | MCP01 / LLM02 |
| `rug-pull` | medium→high | Tool `definition_hash` changed vs prior scan, or `tools.listChanged` declared without auth | MCP03 |
| `missing-auth` | high | Sensitive (write/delete/exec/exfil) tools reachable with no 401/`WWW-Authenticate`/PRM challenge | MCP07 |
| `transport-origin` | low | Server accepts requests without `Origin` validation signal | spec best-practices |
| `no-protocol-version` | info | Server accepts requests lacking `MCP-Protocol-Version` | spec |

Grader: any critical→F; any high→D; medium-only→C; low-only→B; none→A.

## 9. Performance targets (from Success Metrics)
- First scan of a typical server completes in < 10s (hard deadline enforced).
- API p95 (non-scan) < 200ms at small scale.
- Lighthouse ≥ 95 (design target).
- Zero false positives on a curated clean reference server; zero false negatives
  on the bundled known-bad fixture server (smoke-tested).

## 10. UI Surface

### Design System (binding — DESIGN.md saas archetype + verbatim template tokens)
- **Tokens:** the `:root` / `[data-theme="dark"]` block copied verbatim from
  `DESIGN-TEMPLATE.html` (recorded ADR-009). Emitted once as
  `apps/web/src/styles/tokens.css` + Tailwind theme; consumed everywhere (INV-10).
- **Fonts:** Manrope (display/headings, `.btn`, brand), Inter (body, 17px/1.6),
  JetBrains Mono (eyebrows, code, URLs, severity pills, grade chips). Self-hosted,
  `font-display: swap`, preconnect — no FOUT/layout shift.
- **Layout:** saas app shell — persistent left sidebar (224px) + contextual header
  (title, primary action) + content region; landing/home surface ports the
  template scaffold (sticky nav, two-column hero w/ product mockup, bento feature
  grid, risk strip, CTA block, footer). Max width 1200px.
- **Spacing unit:** 4px scale (`--space-1..9`). **Radius:** 6/12/20/999.
  **Shadows:** three-tier. **Motion:** moderate/functional — route/panel
  transitions, optimistic feedback, skeleton loads; 120–320ms eased;
  `prefers-reduced-motion` honored.
- **Theme:** light + dark via `[data-theme]`, default follows
  `prefers-color-scheme`, explicit choice persisted to `localStorage`.
- **Component states:** every interactive element designs default/hover/active/
  `:focus-visible`/disabled/loading; hit targets ≥ 44px; WCAG 2.2 AA contrast.
- **Anti-slop:** characterful display face (Manrope, not raw system-ui);
  template-derived palette (no AI-default gradient); custom-themed components (not
  stock library look); every empty/loading/error/success state designed.

### Screen inventory

| Screen | Route | States | Binds endpoints | Workflow step | RBAC |
|--------|-------|--------|-----------------|---------------|------|
| Landing/Home | `/` | static (marketing scaffold) + CTA → register/scan | — | entry | public |
| Register | `/register` | default, validating, error(field), submitting, success→dashboard | POST /api/auth/register | W1 setup | public |
| Login | `/login` | default, error(uniform), submitting | POST /api/auth/login | — | public |
| Dashboard | `/app` | empty(no scans→new-scan prompt), loading(skeleton), list, error | GET /api/scans, GET /api/endpoints | W1 start | user |
| New scan (form on dashboard + modal) | `/app` | default, validating(url), submitting, SSRF/error inline, success→report | POST /api/scans | W1 | user |
| Scan in progress | `/app/scans/:id` | running(progress), completed→report, failed(designed error w/ code) | GET /api/scans/:id | W1 | owner |
| Report | `/app/scans/:id` | loading, completed(grade+findings list), empty(0 items explained), failed | GET /api/scans/:id | W1, W2 terminal | owner |
| Finding drill-down | `/app/scans/:id` (expand) + raw viewer | collapsed, expanded(evidence+rule+remediation), raw JSON-RPC modal | GET /api/scans/:id | W2 terminal | owner |
| Endpoint history | `/app/endpoints/:id` | loading, timeline+grade-trend chart, empty | GET /api/endpoints/:id | W4 terminal | owner |
| Compare / diff | `/app/scans/:id?compare=:prevId` | loading, diff(added/resolved/changed), no-prior(explained) | GET /api/scans/:id (diff) | W3 terminal | owner |
| Share / export | report header actions | idle, creating-link, link-created(copy), export-download | POST /api/scans/:id/share, GET .../export | W5 terminal | owner |
| Shared report (public) | `/share/:token` | loading, read-only report, 404/410 expired | GET /api/share/:token | W5 view | public(token) |
| Settings (theme) | `/app/settings` | theme toggle, logout | POST /api/auth/logout | — | user |

Every Key Workflow W1–W5 is completable end-to-end through rendered UI including
its terminal step (report render / evidence panel / diff view / history trend /
export-or-share). No non-internal endpoint is API-only. (INV-11)

## 11. Environment & Configuration

`.env.example` (every variable; all secrets via env per `secrets_management`):

| Var | Required | Example | Owner | Description |
|-----|----------|---------|-------|-------------|
| `NODE_ENV` | yes | `production` | api | runtime mode |
| `PORT` | yes | `8787` | api | API listen port |
| `DATABASE_URL` | yes | `postgres://sentinel:pass@db:5432/sentinel` | api/db | Postgres DSN |
| `SESSION_SECRET` | yes | `<32+ byte random>` | api | session cookie signing |
| `CSRF_SECRET` | yes | `<32+ byte random>` | api | double-submit token |
| `APP_ORIGIN` | yes | `https://sentinel.example.com` | api | allowed Origin for CSRF + cookie domain |
| `SCAN_TIMEOUT_MS` | no | `10000` | api | total scan deadline |
| `SCAN_MAX_BYTES` | no | `5242880` | api | response byte ceiling |
| `SCAN_MAX_CONCURRENCY` | no | `3` | api | per-user concurrent scans |
| `RATE_LIMIT_WINDOW_MS` | no | `60000` | api | rate-limit window |
| `HIBP_ENABLED` | no | `true` | api | toggle breach screening (offline dev) |
| `LOG_LEVEL` | no | `info` | api | pino level |
| `VITE_API_BASE` | yes | `/api` | web | frontend API base |

Every code `process.env.*` reference has a matching `.env.example` row (enforced
at Pre-Delivery Gate). `docker-compose.yml` wires all three services + health
checks. Header-buffer sizing applied to the serving path (Node
`--max-http-header-size=32768`; nginx buffers if proxied).

## 12. Test plan (test-after; Vitest + Playwright for e2e)

- **Unit:** SafeConnector IP-deny matrix (loopback, RFC1918, 169.254.169.254,
  IPv4-mapped-IPv6, public-allow); each detector on poisoned + clean fixtures;
  grader mapping; redaction; zod boundaries; ReDoS fixture (catastrophic string
  returns under time cap).
- **Integration:** scan against a bundled in-process fixture MCP server
  (known-good + known-bad) over real HTTP; auth flow; tenant isolation (user B
  cannot read user A's scan → 404); rate-limit fires; CSRF rejects missing token.
- **e2e (Playwright, 1:1 with workflows):** `W1 scan→report`, `W2 drill-down`,
  `W3 rescan→diff`, `W4 history`, `W5 share/export` — each drives the rendered UI
  to its terminal step. Plus negative paths: invalid URL field error, SSRF-blocked
  message, unauth redirect.
- **Smoke:** `smoke-test.sh` runs auth bootstrap, primary CRUD (scan create→list→
  detail→delete endpoint), SSRF-blocked negative, large-header resilience, and the
  capability check (scan the bundled known-bad fixture → expect findings; scan
  known-good → expect grade A / zero false positives).
