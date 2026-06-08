# SMOKE-TEST.md — Pulse

Manual + automated verification checklist. The automated script is
`smoke-test.sh` (run via `/smoke` or `bash smoke-test.sh`); full request/response
evidence is written to `smoke-test.log`. Target defaults to `https://localhost:8443`
(override with `SMOKE_BASE_URL`). UI workflows are covered by Playwright e2e
(`npm run test:e2e -w apps/web`).

## Checklist
- [x] Application starts without errors (`docker compose up -d`; db, redis, api, worker, web, nginx healthy)
- [x] `/api/health` returns 200 with `{status:"ok", services:{postgres:"up", redis:"up"}}`
- [x] User registration and login work (W1 register 201 + session cookie; `/me` returns the user)
- [x] Tenant isolation: tenant B gets 404 on tenant A's monitor
- [x] Main CRUD works end-to-end: create monitor 201 → list → patch 200 (W2)
- [x] Background worker executes: a check result ("up") is produced for the monitor
- [x] Alerting config: create channel 201 (secret redacted in response) + create rule 201 (W3)
- [x] Incident lifecycle: incident opened for a failing monitor → acknowledge 200 → resolve 200 (W4)
- [x] Reports: uptime CSV export + incidents JSON report (W5)
- [x] API key + metric ingest: key created (shown once) → ingest 202 with the key → metrics visible on the monitor (W6)
- [x] Inbound webhook (`/v1/webhooks/twilio/status`): signature verified + idempotent (204, replay deduped 204)
- [x] Large ≥16KB header → 200 (no 431; header buffers sized)

## Key Workflows (UI e2e — 1:1 with SPEC §5, all passing)
- [x] W1 Auth bootstrap & org creation
- [x] W2 Monitor CRUD → live status
- [x] W3 Alerting channels & rules
- [x] W4 Incident lifecycle (open → acknowledge → resolve)
- [x] W5 Reports & CSV export
- [x] W6 API key + metric ingest

Last run: 21/21 functional smoke checks passed; 6/6 Playwright UI workflows passed (against the deployed HTTPS stack, `https://localhost:8443`).
