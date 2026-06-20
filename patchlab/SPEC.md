# SPEC.md — PatchLAB (comprehensive)

**Behavioral contract.** Another engineer (or AI) could rebuild PatchLAB from this document alone. Precedence: RESEARCH.md → ARCHITECTURE.md → SPEC.md. Build depth: **comprehensive** — every state transition, error path, race/idempotency guarantee, validation boundary, and per-feature security/trust boundary is specified.

---

## 1. Feature Inventory

| # | Feature | Slice | Maps to |
|---|---------|-------|---------|
| F1 | Foundation & config | infra | ARCHITECTURE §2,§6 |
| F2 | Auth (register/login/logout/refresh/me) | vertical (UI: login, register) | W-auth |
| F3 | Sound generation (prompt → N WAV variations) + Studio UI | vertical (UI: /studio) | W1 |
| F4 | Library (search/list/detail/download/delete) + UI | vertical (UI: /library) | W5, W6 |
| F5 | Refine an existing sound + UI | vertical (UI: /studio refine) | W2 |
| F6 | Synth preset generation (.vital) + UI | vertical (UI: /presets) | W3 |
| F7 | Sample pack generation (ZIP) + UI | vertical (UI: /packs) | W4, W6 |
| F8 | Landing page (template scaffold) + theme toggle + anon "try it" | vertical (UI: /) | W1 (anon) |

Non-Goals (RESEARCH) excluded: full-song generation, DAW replacement, mixing/mastering, real-time in-DAW, VST plugin, marketplace, collaboration, multi-DAW, education.

---

## 2. Data Model & Validation (boundary values)

(Tables per ARCHITECTURE §3.) Validation rules, with boundaries:

**User**
- `email`: RFC-5322-ish, ≤254 chars, lowercased+unique (INV-5). Invalid/duplicate → see §7.
- `password`: length ≥ 8, ≤ 200 (accept long passphrases; argon2 has no 72-byte bcrypt limit), **no composition rules** (NIST 800-63B). Stored as Argon2id hash only.
- `display_name`: optional, ≤ 80 chars, trimmed.

**Sound**
- `prompt`: 1–500 chars, trimmed; treated as untrusted data (parsed by allow-list, never eval'd — INV-2).
- `category` ∈ {kick,bass,perc,fx,pad,lead,pluck,loop,oneshot}; `kind` ∈ {sample,loop,oneshot,preset}.
- `duration_ms`: 100 … `MAX_DURATION_S*1000` (default cap 8 s). `sample_rate` ∈ {44100, 48000} (≤ `MAX_SAMPLE_RATE`). `bit_depth` ∈ {16, 24}.
- `seed`: int64 ≥ 0; same (prompt, seed, params) ⇒ byte-identical render (determinism).
- `params` JSONB: clamped `SynthParams` (osc type, detune, cutoff 20–20000 Hz, resonance 0–1, ADSR each 0–`MAX_DURATION_S`, distortion 0–1, reverb 0–1, gain ≤ 0 dBFS).

**Generation request:** `variations`: 1 … `MAX_VARIATIONS` (default 4). Total guard: `duration_ms/1000 × sample_rate × channels × variations ≤ MAX_TOTAL_SAMPLES` else 422.

**Pack:** `name` 1–80, `theme` 1–80, `categories` non-empty subset of category enum, `count_per_category` 1 … `MAX_PACK_ITEMS_PER_CAT` (default 4); total items ≤ `MAX_PACK_ITEMS` (default 24); assembled ZIP uncompressed bytes ≤ `MAX_PACK_BYTES` (default 64 MiB) — abort with 422 if exceeded (ZIP-bomb guard).

---

## 3. API Surface (REST `/v1/`, JSON; binary for downloads)

Auth via HttpOnly cookies (`pl_access`, `pl_refresh`). All mutating endpoints require same-site cookie; CSRF mitigated by SameSite=Lax + custom `X-Requested-With` check on state-mutating routes. Errors are RFC 9457 `application/problem+json` (§7).

### Health
- `GET /api/health` → `200 {"status":"ok","db":"up|down","asset_store":"up|down","version":"0.1.0","time":"<iso>"}`. No auth. Used by deploy healthcheck & smoke test.

### Auth (F2)
- `POST /v1/auth/register` — body `{email,password,display_name?}`. **201** `{id,email,display_name}` + sets cookies. Anti-enumeration: if email exists, returns the **same 201-shaped** success envelope (a notification is "sent" to the existing address; no distinguishable error) — existence never leaked. Rate-limit `(ip,email)` 5/min. Timing padded.
- `POST /v1/auth/login` — `{email,password}`. **200** `{id,email,display_name}` + cookies. Failure (unknown email OR wrong password OR rate-limited) → identical `401 {type:"/errors/invalid-credentials"}` envelope; rate-limit fires **before** Argon2 verify; no-op path timing-padded.
- `POST /v1/auth/refresh` — uses `pl_refresh`; rotates (old `jti` revoked, new issued). Reuse of a revoked `jti` → 401 + all user refresh tokens revoked (reuse detection).
- `POST /v1/auth/logout` — revokes current refresh `jti`, clears cookies. **204**.
- `GET /v1/auth/me` — **200** `{id,email,display_name}` or 401.

### Sounds (F3/F4/F5/F6)
- `POST /v1/sounds/generate` — `{prompt, category?, kind?, variations?, duration_ms?, sample_rate?, bit_depth?, seed?}`. Auth **optional**. **201** `{variations:[{id,name,category,kind,duration_ms,sample_rate,bit_depth,size_bytes,seed,download_url,preset_url?,created_at,expires_at?}]}`. Authenticated → persisted to library (`owner_id`). Anonymous → `owner_id NULL`, `expires_at` set, stricter caps. Rate-limit: auth `(user)` 30/min, anon `(ip)` 8/min.
- `POST /v1/sounds/{id}/refine` — `{instruction, variations?}` (instruction e.g. "make it punchier", "more reverb"). Owner only. Parses instruction via the same allow-list delta map applied to the parent's `params`; creates child sounds with `parent_id={id}`. **201** same shape as generate. 404 if not owner (not 403 — no existence leak across tenants).
- `GET /v1/sounds` — query `q?, category?, kind?, page?(≥1), page_size?(1–100,default 20)`. Owner only. **200** `{items:[SoundSummary], page, page_size, total}`. Search matches prompt/name (case-insensitive).
- `GET /v1/sounds/{id}` — metadata. Owner (or anon asset by capability UUID). 404 otherwise.
- `GET /v1/sounds/{id}/download` — streams WAV (`FileResponse`, `audio/wav`, `Content-Disposition: attachment; filename="<safe>.wav"`). Owner or valid anon-capability. Path built only by storage resolver (INV-11).
- `GET /v1/sounds/{id}/preset` — streams `.vital` JSON (`application/json`, attachment). Owner only; 404 if no preset for asset.
- `DELETE /v1/sounds/{id}` — owner only. Deletes row + file. **204**. Idempotent (already-gone → 404).

### Packs (F7)
- `POST /v1/packs` — `{name, theme, categories:[...], count_per_category?}`. Auth required. Generates sounds per category, assembles metadata; **201** `{id,name,theme,items:[SoundSummary],created_at}`. Pack ZIP built lazily on download. Rate-limit `(user)` 6/min.
- `GET /v1/packs` — owner list. **200** `{items:[PackSummary]}`.
- `GET /v1/packs/{id}` — owner detail + items. 404 otherwise.
- `GET /v1/packs/{id}/download` — assembles + streams ZIP with running byte budget (abort 422 on overflow). `application/zip`. Owner only.
- `DELETE /v1/packs/{id}` — owner; deletes pack + member files + zip. **204**.

**Internal (non-UI) endpoints:** `/api/health` only. All `/v1/*` are UI-reachable.

### Per-endpoint performance targets (comprehensive)
| Endpoint | Target (p95, small tier) |
|----------|--------------------------|
| `/api/health`, auth, list/detail | < 150 ms |
| `/v1/sounds/generate` (4 var, 2 s, 44.1k) | < 1200 ms server render |
| `/v1/sounds/{id}/download` | < 200 ms TTFB streaming |
| `/v1/packs/{id}/download` (24 items) | < 3 s |

---

## 4. Security Requirements (mapped in COMPLIANCE.md)

| ID | Requirement | Implementation |
|----|-------------|----------------|
| SEC-1 | Passwords stored with a memory-hard hash (Argon2id), never plaintext/reversible | `core/security.py` argon2-cffi |
| SEC-2 | Password policy: length-based (≥8, accept long), no composition rules, no forced rotation (NIST 800-63B) | register/refine validators |
| SEC-3 | JWT verified with pinned algorithm; `alg=none` rejected; secret from env (INV-13) | `core/security.py` PyJWT |
| SEC-4 | Session tokens in HttpOnly+Secure+SameSite cookies; refresh rotation + reuse detection | auth service |
| SEC-5 | Auth anti-enumeration: identical response shape & padded timing for unknown email vs wrong password; register never leaks existence | auth service |
| SEC-6 | Rate limiting on auth (before hash), generation, packs, download with SPEC key dimensions (INV: slowapi) | `core/ratelimit.py`, decorators |
| SEC-7 | Object-level authorization (ownership) on every sound/pack read/mutate/download + sub-resources; cross-tenant returns 404 not 403 | `services` `assert_owner` |
| SEC-8 | Path-traversal-safe asset access; opaque UUID ids; realpath confined to ASSET_ROOT (INV-11) | `core/storage.py` |
| SEC-9 | Prompt/instruction parsed by allow-list, never eval/exec (INV-2); all synth params clamped | `audio/prompt_parser.py` |
| SEC-10 | Resource caps (duration/rate/variations/total-samples/pack-bytes) enforced pre-allocation (DoS, ZIP-bomb) | `audio/render.py`, `audio/pack.py` |
| SEC-11 | No SQL string interpolation; parameterized queries only (INV-1) | repositories |
| SEC-12 | Structured JSON logs redact secrets/tokens/passwords; no request bodies | `core/logging.py` |
| SEC-13 | HTTPS only: TLS termination, HTTP→HTTPS redirect, HSTS (INV-14) | `infra/nginx/nginx.conf` |
| SEC-14 | CSRF mitigation on state-mutating routes (SameSite=Lax + `X-Requested-With`) | auth dependency |
| SEC-15 | Per-user storage quota + anonymous-asset TTL cleanup (storage exhaustion) | cleanup job, generate service |

OWASP ASVS L1/L2 floor applies. No named compliance regime (HIPAA/PCI/GDPR) declared in RESEARCH → only the OWASP baseline (logged).

---

## 5. Key Workflows (1:1 with e2e smoke tests)

Each has exactly one `test('W<n> …')` driving it through the rendered UI to its terminal step.

- **W1 — Generate a Custom Sound.** `/studio` (or landing anon): enter prompt → POST `/v1/sounds/generate` → variation table renders (S4) → preview each (waveform + audio) → **terminal: download WAV** (`/v1/sounds/{id}/download`). States: empty (no prompt), loading (rendering bar), success (variations), error (invalid prompt 422 / rate-limited 429). Screens: S4 (+S1 anon).
- **W2 — Refine an Existing Sound.** From a generated/library sound: enter instruction ("make it punchier") → POST `/v1/sounds/{id}/refine` → new variations appear linked to parent → **terminal: download refined WAV**. Screens: S4.
- **W3 — Generate a Synth Preset.** Generate (or open a sound) → request preset → GET `/v1/sounds/{id}/preset` → **terminal: download `.vital` file**. Preset view lists preset-kind assets (S7). Screens: S4, S7.
- **W4 — Create a Sample Pack.** `/packs`: name + theme + pick categories → POST `/v1/packs` → pack detail with items (S6) → **terminal: download ZIP** (`/v1/packs/{id}/download`). Screens: S6.
- **W5 — Search and Reuse Assets.** `/library`: search/filter → results (S5) → open detail → **terminal: preview & re-download / regenerate variation**. Screens: S5.
- **W6 — Export to Production Workflow.** Select asset(s) → download WAV / `.vital` / pack ZIP in FL-compatible format → **terminal: file saved locally** (asserted via download response). Screens: S5, S6, S7.

(Auth bootstrap — register→login→me — is the Auth-bootstrap smoke flow, prerequisite to W2–W6.)

---

## 6. UI Surface

App shell: persistent left sidebar (224px) + contextual header (page title + primary action) + content region (saas Layout Doctrine). Dark theme default; runtime toggle. All screens consume the design-token layer (Design System §6.1); no screen invents tokens.

| ID | Route | Purpose | Binds endpoints | Workflow step(s) | RBAC | States (empty/loading/error/success) |
|----|-------|---------|-----------------|------------------|------|--------------------------------------|
| S1 | `/` | Landing (template scaffold) + anon "try it" prompt | `POST /v1/sounds/generate` (anon) | W1 (anon entry) | public | — / generating / 429 or 422 inline / variations preview |
| S2 | `/login` | Sign in | `POST /v1/auth/login` | W-auth | public | — / submitting / invalid-credentials field error / redirect to /studio |
| S3 | `/register` | Create account | `POST /v1/auth/register` | W-auth | public | — / submitting / field errors / redirect |
| S4 | `/studio` | Generate + refine; variation table, waveform, player | `generate`,`{id}/refine`,`{id}/download`,`{id}/preset` | W1, W2, W3 | auth (+anon preview) | empty prompt CTA / render progress / error toast+field / variations table |
| S5 | `/library` | Search/filter personal sounds, detail, download, delete | `GET /v1/sounds`,`{id}`,`{id}/download`,`DELETE` | W5, W6 | auth | "No sounds yet — generate your first" / skeleton rows / error / table |
| S6 | `/packs` | Create + list packs, detail, download ZIP, delete | `POST /v1/packs`,`GET /v1/packs`,`{id}`,`{id}/download`,`DELETE` | W4, W6 | auth | empty / building / error / pack cards + items |
| S7 | `/presets` | List preset-kind assets, download `.vital` | `GET /v1/sounds?kind=preset`,`{id}/preset` | W3, W6 | auth | empty / skeleton / error / preset table |
| S8 | `/settings` | Profile, theme, account, logout | `GET /v1/auth/me`,`POST /v1/auth/logout` | — | auth | loaded form / saving / error / saved |

Every non-internal endpoint is reachable from ≥1 screen; every §5 workflow completes end-to-end through the UI including its terminal (download/export) step. "The UI" is never a single item.

### 6.1 Design System (binding, precedence-bound)

- **Tokens:** copied verbatim from `DESIGN-TEMPLATE.html` (`:root` + `[data-theme="dark"]`) into `frontend/src/app/globals.css` — full palette in DECISIONS.md ADR-013. Accent `#2563eb`; teal `#0d9488`/`#2dd4bf` for audio/waveform chrome; semantic success/warning/error.
- **Type:** Manrope (display/headings/buttons, 500–800), Inter (body, 400–600), JetBrains Mono (eyebrows/labels/metrics/code). Base body 16px / line-height 1.55. Modular scale per template.
- **Spacing:** 4–64px scale (`--space-1…8`), 8px rhythm. **Radius:** sharp — 2/4/6px. **Shadow:** minimal (`--shadow-sm/md/lg`).
- **Motion:** functional, 0.2s ease-out; `prefers-reduced-motion` honored. **Breakpoints:** 640/768/1024/1280; collapse multi-col to single ≤ 940px (template), hide sidebar→drawer on mobile.
- **Archetype (saas):** app shell + nav with active state; data tables (sort/filter/empty/loading/error, row actions) as primary info architecture; forms with inline validation + grouped fieldsets; metric/stat blocks; inline audio player with waveform (teal). Full dark mode.
- **Signature components:** AppShell, Sidebar (active state), PromptBar, VariationTable, AudioPlayer (transport + waveform), PackBuilder form, LibraryTable, ThemeToggle, Toast, Dialog/Drawer.
- **Component states:** default/hover/active/`:focus-visible`(3px accent ring)/disabled/loading designed for every interactive element; hit targets ≥44×44px. WCAG 2.2 AA contrast.

---

## 7. Error Contract (RFC 9457 `application/problem+json`)

```json
{ "type": "/errors/<slug>", "title": "<human title>", "status": <code>,
  "detail": "<message>", "errors": { "<field>": "<field message>" } }
```
- `422 /errors/validation` — `errors` maps each offending field (email, password, prompt, duration_ms, …) to a specific message. UI MUST attach each to its control (not a generic banner).
- `401 /errors/invalid-credentials` (login), `401 /errors/unauthenticated` (no/expired session).
- `404 /errors/not-found` — used for both missing and cross-owner resources (no existence leak).
- `429 /errors/rate-limited` — includes `Retry-After`; UI shows "Slow down — try again in Ns".
- `413 /errors/payload-too-large` / `422 /errors/resource-cap` — generation/pack exceeds caps.
- `500 /errors/internal` — generic; no stack trace to client; logged server-side with request id.

State machines:
- **Generation:** `idle → validating → rendering → ready(variations) | error(field|cap|rate)`. Cancellable client-side (abort fetch); server render is bounded by caps.
- **Refine:** requires a parent in `ready`; produces children; parent unchanged (immutable lineage).
- **Auth session:** `anonymous → authenticated(access valid) → refreshing → authenticated | expired→anonymous`. Refresh reuse → full revoke.
- **Pack:** `draft(items generated) → zipping → downloadable | error(cap)`.

Concurrency/idempotency:
- Register: unique `users.email` (INV-5) closes the TOCTOU race (two simultaneous signups → one 201, other sees anti-enumeration success path; DB constraint authoritative).
- Refresh rotation: `jti` UNIQUE (INV-6) + row `revoked` flag; concurrent refresh with same token → one succeeds, reuse detected → revoke-all.
- Delete: idempotent; second delete → 404.
- Generation determinism: `(prompt, seed, params)` reproducible; concurrent identical requests produce independent rows (no shared mutable state in the engine).

---

## 8. Environment & Configuration

All config via env (pydantic-settings); every var has a `.env.example` entry (enforced).

| Var | Service | Required | Default / example | Purpose |
|-----|---------|----------|-------------------|---------|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | db,api | yes | patchlab / `change-me` / patchlab | DB credentials |
| `DATABASE_URL` | api | yes | `postgresql+asyncpg://patchlab:change-me@db:5432/patchlab` | Async DSN |
| `JWT_SECRET` | api | yes | `change-me-32+chars` | HS256 signing secret (≥32 chars) |
| `JWT_ALGORITHM` | api | no | `HS256` | Pinned alg |
| `ACCESS_TOKEN_TTL_MIN` | api | no | `15` | Access cookie lifetime |
| `REFRESH_TOKEN_TTL_DAYS` | api | no | `7` | Refresh lifetime |
| `COOKIE_SECURE` | api | no | `true` | Secure flag (true in HTTPS) |
| `ASSET_ROOT` | api | yes | `/data/assets` | Asset store root (volume) |
| `ANON_ASSET_TTL_HOURS` | api | no | `24` | Anonymous asset expiry |
| `USER_STORAGE_QUOTA_MB` | api | no | `512` | Per-user quota |
| `MAX_DURATION_S` | api | no | `8` | Generation duration cap |
| `MAX_SAMPLE_RATE` | api | no | `48000` | Sample-rate cap |
| `MAX_VARIATIONS` | api | no | `8` | Variations cap |
| `MAX_TOTAL_SAMPLES` | api | no | `40000000` | Total-samples guard |
| `MAX_PACK_ITEMS` | api | no | `24` | Pack item cap |
| `MAX_PACK_BYTES` | api | no | `67108864` | Pack ZIP uncompressed byte budget |
| `RATE_LIMIT_*` | api | no | per §3 | slowapi rates |
| `CORS_ORIGINS` | api | no | `https://localhost` | Allowed origins |
| `TRUSTED_PROXY` | api | no | `true` | Trust XFF from nginx only |
| `LOG_LEVEL` | api | no | `INFO` | Logging |
| `NEXT_PUBLIC_API_BASE` | web | no | `/v1` (same-origin via proxy) | Frontend API base |
| `TLS_CERT_PATH` / `TLS_KEY_PATH` | proxy | yes (prod) | `/etc/nginx/certs/...` | TLS cert/key |

**Deployment:** `docker compose up` → `db`, `api` (runs `alembic upgrade head` on start), `web` (Next.js standalone), `proxy` (nginx, TLS, HTTP→HTTPS, HSTS, large header buffers). Served at `https://localhost`. Health: `GET /api/health`. Protocol: **HTTPS only** (smoke test hits the https origin; local dev uses mkcert/self-signed).

---

## 9. Test Plan (test-after, comprehensive)

- **Unit:** prompt parser (keyword→param, unknown-token ignore, clamping, no-eval); synth determinism (same seed → identical bytes); render caps (oversize → error); WAV header validity; pack byte-budget abort; storage resolver traversal rejection; security (argon2 verify, jwt pin, anti-enumeration timing).
- **Integration (httpx + test DB):** auth flow incl. anti-enumeration + refresh rotation/reuse; generate (auth + anon) incl. cap 422 + rate 429; ownership 404 on cross-user read/download/refine/delete + sub-resources; library search; pack create/download; idempotent delete.
- **Security tests:** IDOR matrix (B cannot read/download/delete A's sound/pack), path traversal attempt, SQL-ish prompt injection (no effect), oversized request rejection, alg=none token rejection.
- **e2e (Playwright):** one `test('W<n> …')` per workflow W1–W6 through rendered UI to terminal download; plus auth bootstrap. Drives `ui-coverage` invariant (INV-10).
- **Invariant lint:** `scripts/invariant-lint.mjs` evaluates all 12 machine-checkable invariants.
