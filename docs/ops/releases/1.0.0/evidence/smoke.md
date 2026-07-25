# Evidence — Gate C smoke

| Field | Value |
|-------|-------|
| Release | 1.0.0 |
| Gate | C |
| Milestone | Production smoke |
| Verdict | GO (dress-rehearsal compose-exec / localhost) |
| Closed at (UTC) | 2026-07-25T21:38:20Z |
| GO by (role / name) | Engineering (solo operator) |
| API SHA | `caa3f1d0e3e4c53bacf76719a0f18b1473560207` |

## Runs

1. **During `deploy-production.sh`** — dress-rehearsal `smoke-test-production.sh` (compose exec):

```text
GET /version OK git_sha=caa3f1d0e3e4c53bacf76719a0f18b1473560207 environment=production
PRODUCTION SMOKE PASSED (dress-rehearsal)
```

2. **Localhost republish check** after `127.0.0.1:18000` / `:13000` ports added:

- `GET http://127.0.0.1:18000/health/live` → 200
- `GET http://127.0.0.1:18000/health/ready` → 200
- `GET http://127.0.0.1:18000/version` → non-empty `git_sha`
- `GET http://127.0.0.1:13000/` → 200
- `GET http://127.0.0.1:13000/me/deposit` → 200

## Deferred to Gate D

Public HTTPS smoke against `https://api.roamkit.net` / `https://roamkit.net` after Traefik Host + DNS cutover (production labels currently `traefik.enable=false`).
