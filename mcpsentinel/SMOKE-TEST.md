# SMOKE-TEST.md — MCP Sentinel (manual checklist)

Run against a live deploy (default `http://localhost:8787`). The automated form is
`smoke-test.sh`. Every Key Workflow (SPEC §5) must pass end-to-end before ship.

## Preconditions
- Stack up (`docker compose up -d --build` or `npm run dev` + Postgres migrated).
- Fixture MCP servers running: `node --import tsx apps/api/src/scanner/__fixtures__/serve-fixtures.ts`
  (known-good on :9101, known-bad on :9102).
- `SCAN_ALLOW_HOSTS=127.0.0.1` so Sentinel may scan the loopback fixtures.

## Checklist
- [ ] App starts; `GET /api/health` → 200 `{"status":"ok","db":"up",...}`.
- [ ] Homepage `/` loads (landing renders, nav + hero visible).
- [ ] **Register** a new account → redirected to `/app`.
- [ ] **Login/logout** work; a protected route (`/api/scans`) rejects unauthenticated (401).
- [ ] **W1 Scan** the known-bad fixture (`http://127.0.0.1:9102/`) → report renders
      grade **F** with findings across tool-poisoning, excessive-scope, secret-leak,
      missing-auth, rug-pull.
- [ ] Scan the known-good fixture (`http://127.0.0.1:9101/`) → grade **A**, zero
      findings (no false positives).
- [ ] **W2 Drill-down:** expand a finding → evidence + remediation + raw JSON-RPC.
- [ ] **W3 Re-scan & compare:** re-scan → diff shows changes (or none).
- [ ] **W4 History:** endpoint history shows the grade trend + timeline.
- [ ] **W5 Share/export:** create a share link → opens read-only; export downloads JSON.
- [ ] **SSRF blocked:** scanning `http://169.254.169.254/` or `http://10.0.0.1/`
      → failed scan with `SSRF_BLOCKED` (not a successful internal fetch).
- [ ] **Tenant isolation:** user B cannot read user A's scan (404).
- [ ] **Secret redaction:** the leaked-secret finding shows `[REDACTED:…]`, never the raw token.
- [ ] **Large-header resilience:** homepage + a deep route with a ≥16 KB `Cookie:`
      header → 200 (not 400/431).
- [ ] Unhappy path: submit an invalid URL → field-level error bound to the input.
