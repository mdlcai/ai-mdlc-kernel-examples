# SMOKE-TEST.md — PatchLAB manual smoke checklist

Run after deploy. Automated equivalent: `bash scripts/smoke-test.sh` (writes `smoke-test.log`).

| # | Check | Expected |
|---|-------|----------|
| 1 | App starts: `docker compose ps` | db, api, web, proxy all healthy |
| 2 | Health: `curl -k https://localhost/api/health` | `{"status":"ok","db":"up",...}` |
| 3 | Landing loads: visit `https://localhost/` | template-scaffold landing renders; "Generate free" works |
| 4 | Register + login: `/register` then `/login` | account created; redirected into the app shell |
| 5 | **W1** Generate (auth): `/studio`, enter a prompt, Generate | variation table renders; each row plays + downloads a RIFF WAV |
| 6 | **W2** Refine: refine a generated sound | new linked variations appear; download works |
| 7 | **W3** Preset: download `.vital` from a sound | valid JSON preset file |
| 8 | **W4** Pack: `/packs`, build a themed pack | pack created; ZIP download contains category folders |
| 9 | **W5** Library: `/library`, search/filter | matching sounds; preview + delete (confirm) |
| 10 | **W6** Export: download WAV / `.vital` / ZIP | files save in FL-compatible formats |
| 11 | Anonymous: landing "try it" without login | generates + downloads (TTL capability asset) |
| 12 | Auth required: open `/library` while signed out | redirected to `/login` |
| 13 | IDOR: user B requests user A's sound id | 404 (not 403) |
| 14 | Unhappy path: submit an invalid prompt/empty fields | field-level error on the offending control (not a generic banner) |
| 15 | Large cookie: replay a route with a ≥16KB `Cookie:` header | 200, not 400/431 |
| 16 | HTTPS: visit `http://localhost/` | 301 redirect to `https://` (HSTS set) |
