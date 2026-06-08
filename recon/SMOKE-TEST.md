# SMOKE-TEST.md — Recon

Manual + automated verification checklist. The automated script is
`smoke-test.sh`; full request/response evidence is written to `smoke-test.log`.
Target is the local HTTPS stack (`https://localhost`, nginx TLS per ADR-011).

## Checklist
- [x] Docker daemon reachable (`docker info` exit 0)
- [x] `docker compose up -d` → 5 services healthy (db, redis healthy; api, web, nginx up)
- [x] `/api/v1/health` returns 200 `{status:"ok", services:{api:"up", db:"up", redis:"up"}}`
- [x] HTTP→HTTPS redirect (301) and HTTPS-only serving
- [x] User registration and login work (W1 register 201, login 200, `/me` 200, email match)
- [x] Asset discovery: authorize → DNS-TXT verify → authorized (W2)
- [x] Continuous scanning: schedule 200, scan completed — 8 hosts, 6 open ports (real scan of the stack subnet) (W3)
- [x] Change detection: baseline accept → `PORT_OPENED` change → acknowledged (W4)
- [x] Alert routing: channel created; delivery row present (status=failed is contract-correct for the self-signed target) (W5)
- [x] Reporting: report ready; download 200 (W6)
- [x] Tenant isolation: tenant B reads tenant A's asset → 404 (runtime Postgres RLS) (W7)

## Key Workflows (1:1 with SPEC §5, all passing)
- [x] W1 Auth bootstrap & org creation
- [x] W2 Asset discovery & authorization
- [x] W3 Continuous scanning
- [x] W4 Change detection
- [x] W5 Alert routing
- [x] W6 Reporting
- [x] W7 Tenant isolation

Last run: 8/8 functional smoke flows passed (against `https://localhost`). Full request/response evidence in `smoke-test.log`.
