# SMOKE-TEST.md — Sentinel

Manual smoke checklist (companion to the automated `smoke-test.sh`). Run after deploy.
"Capability generalizes" items exercise the scanner pipeline against input the build did
not author.

## Automated (`bash smoke-test.sh`, evidence → `smoke-test.log`)
- [ ] `GET /api/health` → 200 `{status:"ok"}`
- [ ] **W1** register → 201 + session cookie; `/me` → 200 returns the user
- [ ] Anti-enumeration: wrong-password and unknown-email logins return the same 401 envelope
- [ ] **W1** create project → 201; project detail → 200
- [ ] **W2** add repo target → 201
- [ ] **W2** live target: scan blocked 422 until `verify-ownership` → 200 (INV-12)
- [ ] SSRF: `169.254.169.254` target rejected 422
- [ ] **W3** run scan → 202; findings ≥ 5, severity-sorted (critical first)
- [ ] **W4** triage: ignore-without-reason → 400; mark fixed → 200
- [ ] **W5** dashboard rollups → 200
- [ ] **W8** export JSON → 200; export PDF → 503 contract (not fabricated)
- [ ] Authorization isolation: cross-tenant read → 404
- [ ] Webhook: unsigned delivery → 401
- [ ] Large-cookie (18KB) request → 200 (not 400/431)

## Capability generalizes (non-authored input)
- [ ] Add a target whose manifest contains a DIFFERENT known-vulnerable package set than the seeded demo; confirm the SCA engine flags exactly the matching advisories and nothing for patched versions.
- [ ] Confirm dedup: a finding reported by two scanners collapses to one row listing both scanners.

## UI workflow (headless browser, against the running web app)
- [ ] `/register` → create account → redirected to `/app`
- [ ] `/app/projects` → create a project → appears in the list (empty state replaced)
- [ ] `/app/projects/:id` → add a repo target → "Run scan" → scan status reaches completed
- [ ] `/app/findings` → deduped severity-sorted table renders (not raw JSON); filter to a severity; zero-result state shows when a filter matches nothing
- [ ] `/app/findings/:id` → triage a finding to "fixed"; it leaves the open list
- [ ] Theme toggle switches light/dark and persists on reload
- [ ] Keyboard: tab through the findings table and filters; focus ring visible
