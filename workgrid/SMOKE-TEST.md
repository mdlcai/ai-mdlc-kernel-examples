# SMOKE-TEST.md — WorkGrid

Manual + automated verification checklist. The automated script is
`smoke-test.sh` (run via `/smoke` or `bash smoke-test.sh`); results are written to
`smoke-test.log`. Target defaults to `https://localhost:8444` (override with
`SMOKE_BASE_URL`). UI workflows are covered by `apps/web/e2e/workflows.spec.ts`
(`npm run test:e2e -w apps/web`, `E2E_BASE_URL=https://localhost:8444`).

## Checklist
- [x] Application starts without errors (`docker compose up -d`; all services healthy)
- [x] Homepage loads in a browser (landing renders, title "WorkGrid — One workspace to move work forward")
- [x] HTTP→HTTPS redirect (301) and HTTPS-only serving
- [x] User registration and login work (W1)
- [x] `/api/health` returns 200 with `{status:"ok", services:{db:"up",redis:"up"}}`
- [x] At least one protected endpoint rejects unauthenticated requests (GET /v1/tickets → 401)
- [x] Main CRUD works end-to-end (create → read → update/resolve a ticket) (W2/W7)
- [x] Tenant isolation: user B cannot read user A's org resource (→ 404)
- [x] Inbound webhook rejects an invalid signature (→ 403)
- [x] Validated surface returns the structured problem+json field error contract
- [x] Large 16KB cookie header → 200 (no 431; header buffers sized)
- [x] Database contains seed data after startup (demo org, projects, tickets, workflow)
- [x] Capability check: SLA auto-routing assigns + sets due_at on ticket create (non-seed input)

## Key Workflows (UI e2e — 1:1 with SPEC §5, all passing)
- [x] W1 Auth bootstrap & org creation
- [x] W2 Incident intake → queue
- [x] W3 Project with linked tasks
- [x] W4 Workflow surfaces states
- [x] W5 Approval inbox reachable
- [x] W6 Executive portfolio dashboard
- [x] W7 Service-desk resolution (create → resolve through the UI)
- [x] W8 Report export

Last run: 15/15 functional smoke checks passed; 9/9 Playwright UI workflows passed.
