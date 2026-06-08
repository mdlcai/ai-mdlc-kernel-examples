# SMOKE-TEST.md — SmarCoders (manual checklist)

Run after `docker compose up -d --build` (see QUICKSTART.md). The automated equivalent is
`bash scripts/smoke-test.sh` (functional, every SPEC §5 workflow) + `apps/web/e2e` (Playwright UI).

| # | Check | Expected |
|---|-------|----------|
| 1 | `docker compose ps` | db, redis, api, worker, web, nginx all up; api healthy |
| 2 | `curl -sk https://localhost/api/health` | `{"status":"ok","services":{"postgres":"healthy",...}}` |
| 3 | Visit `https://localhost` | Landing page renders (nav, hero, feature grid, CTA, footer); synthetic-data note visible |
| 4 | Plain HTTP `curl -s http://localhost/` | 301 redirect to https (HTTPS only) |
| 5 | Register at `/register` (new email) | Account created, signed in, redirected to `/queue` |
| 6 | Login at `/login` as coder (coder@smartcoders.local / ChangeMe!Coder1) | Lands on `/queue` with assigned charts |
| 7 | Open a chart, add ICD-10 codes (E11.9 principal, I10, E78.5), Validate | Findings render; no blocking errors |
| 8 | Submit for audit | Chart → pending_audit; confirmation shown |
| 9 | Submit a chart with a bad code (e.g. `123`) | Blocking format error; Submit disabled; field-level message (no stack trace) |
| 10 | Login as supervisor, open `/audit`, audit the chart, Approve | Chart → completed; audit-log entry written |
| 11 | Login as manager, `/imports`, "Load demo data" | Import job runs to completed; new charts appear in queue |
| 12 | `/metrics` | Live stat cards update via SSE; CSV and PDF export download |
| 13 | `/audit-log` (manager) | Append-only entries + integrity badge "ok" |
| 14 | Protected endpoint unauthenticated: `curl -sk https://localhost/api/charts` | 401 `{"error":{"code":"UNAUTHENTICATED",...}}` |
| 15 | Cross-tenant: a second org's manager GETs another org's chart id | 404 (no leak) |
| 16 | Signed webhook to `/api/webhooks/test`; replay; bad signature | 200 → deduped → 403 |
| 17 | Large header: replay `/api/health` with an 18KB `Cookie:` | 200 (not 400/431) |
| 18 | Seed data present | Demo org has charts + 3 demo users |

Any failure ⇒ do not consider the build healthy; fix root cause and re-run the full matrix.
