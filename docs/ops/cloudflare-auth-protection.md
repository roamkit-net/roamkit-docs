# Cloudflare auth protection (Turnstile + edge)

Protects public auth POSTs against credential stuffing, registration spam, and
password-reset email flooding. Application layers live in `roamkit-api` /
`roamkit-web`; this note covers **ops provisioning** and **edge** rules.

## Layers

1. **DRF throttles** (always on once API PR is deployed) — scoped rates via `AUTH_*_RATE` env.
2. **Turnstile widget + server siteverify** — when `TURNSTILE_ENABLED=true` and keys set.
3. **Cloudflare Managed Challenge** — WAF/custom rule on auth paths (dashboard or API).

Fail-open: if siteverify times out / network / 5xx, auth may proceed under a
stricter degraded IP gate (`AUTH_TURNSTILE_DEGRADED_RATE`). Invalid tokens always
fail closed (400).

## Staging soak (before enabling the widget)

After deploying API with `TURNSTILE_ENABLED=false`:

1. Leave Turnstile **off** for **~24 hours** on staging.
2. Watch `auth_throttle_block_total` (structured logs / Sentry metrics) and 429 rates
   on `/api/v1/auth/register/`, `/token/`, `/password-reset/`.
3. Tune `AUTH_*_RATE` via `.env` if legitimate traffic or smoke tests are blocked.
4. Only then merge/enable the web widget and set keys.

## Feature flags / env

| Variable | Where | Notes |
|----------|--------|--------|
| `TURNSTILE_ENABLED` | API | Default `false` |
| `TURNSTILE_SECRET_KEY` | API | From CF widget; never `NEXT_PUBLIC_*` |
| `TURNSTILE_SITE_KEY` | API (optional) | Same sitekey as web |
| `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | Web build/runtime | Empty → no widget |
| `TURNSTILE_BYPASS_SECRET` | API | Optional; empty = disabled |
| `AUTH_TOKEN_RATE` etc. | API | DRF `N/period` strings |

Diagnostic (not a probe): `GET /health/turnstile` — config + DNS only, no siteverify.

## Provision keys on the dedicated server (CF API)

Do **not** copy keys from the dashboard into git. On the Hetzner host, Traefik already
holds a Cloudflare API token (e.g. `CF_DNS_API_TOKEN` under `/opt/stacks/traefik`).

Token must allow **Account → Turnstile → Edit**. If the DNS-01 token is zone-only,
extend scopes or add a sibling secret next to it (still server-only).

### Flow: create → fetch → validate → `.env` → recreate

```text
create widget (or note existing sitekey)
        ↓
GET widget (read-back — do not trust create alone)
        ↓
validate sitekey + secret + hostnames + mode=managed
        ↓
write stack .env
        ↓
recreate api + web
        ↓
GET /health/turnstile + smoke login/register/forgot
```

Example (replace account id, token, hostnames):

```bash
# On the dedicated server — never commit output.
export CF_TOKEN="$(grep CF_DNS_API_TOKEN /opt/stacks/traefik/.env | cut -d= -f2-)"
export CF_ACCOUNT_ID="…"

# Create (skip if widget already exists)
curl -sS -X POST \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/challenges/widgets" \
  -H "Authorization: Bearer ${CF_TOKEN}" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "roamkit-staging-auth",
    "domains": ["staging.roamkit.net"],
    "mode": "managed"
  }' | tee /tmp/turnstile-create.json

SITEKEY="$(jq -r '.result.sitekey' /tmp/turnstile-create.json)"

# Read-back
curl -sS \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/challenges/widgets/${SITEKEY}" \
  -H "Authorization: Bearer ${CF_TOKEN}" | tee /tmp/turnstile-get.json

# Validate before writing .env
jq -e '
  .success == true
  and (.result.sitekey | length) > 0
  and (.result.secret | length) > 0
  and (.result.mode == "managed")
' /tmp/turnstile-get.json

# Then set on the stack (example paths):
#   /opt/stacks/roamkit-net/.env          (staging)
#   /opt/stacks/roamkit-production/.env   (production)
#
# TURNSTILE_ENABLED=true
# TURNSTILE_SECRET_KEY=<result.secret>
# TURNSTILE_SITE_KEY=<result.sitekey>
# NEXT_PUBLIC_TURNSTILE_SITE_KEY=<result.sitekey>
#
# Recreate api + web so NEXT_PUBLIC_* is visible to the browser bundle/runtime.
```

Abort if read-back validation fails. Never put `TURNSTILE_BYPASS_SECRET` or the
Turnstile secret in `NEXT_PUBLIC_*` or committed example files with real values.

### Internal bypass (smoke scripts only)

Header: `X-RoamKit-Internal: <TURNSTILE_BYPASS_SECRET>`

Honored only when the secret is non-empty and matches (constant-time). Skips
Turnstile verify; **throttles still apply**. Do not ship this secret to the web app.

## Managed Challenge (edge)

Prefer path-specific rules over a blanket `/api/v1/auth/*` if Cloudflare matching
allows, to avoid false positives on `POST /api/v1/auth/token/refresh/`:

- `/api/v1/auth/register/`
- `/api/v1/auth/token/` (obtain only — exclude refresh if possible)
- `/api/v1/auth/password-reset/`

Action: **Managed Challenge**. Same server CF token may create rules via API when
scopes allow; otherwise configure once in the dashboard and record the rule id here.

Client IP for app throttles: prefer `CF-Connecting-IP` via `get_client_ip()`.

## Metrics (watch these)

| Counter | Meaning |
|---------|---------|
| `turnstile_verify_success_total` | siteverify OK |
| `turnstile_verify_failed_total` | invalid / missing / replay |
| `turnstile_verify_unavailable_total` | timeout / network / 5xx |
| `turnstile_degraded_gate_block_total` | fail-open IP gate denied |
| `auth_throttle_block_total` | DRF auth throttle denied |

UNAVAILABLE logs include `request_id`, `ip_hash` (not raw IP), `endpoint`, `reason`.

## Vouchers (future — ADR 011)

Do **not** add Turnstile to voucher redeem in v1. Prefer per-user / per-IP throttle,
backoff, lockout after failed attempts, and audit log. Revisit Turnstile only if
abuse appears.

## Smoke checklist

- [ ] `TURNSTILE_ENABLED=false` deploy + staging soak ~24h (throttles only)
- [ ] Widget create → fetch → validate → `.env` → recreate
- [ ] `GET /health/turnstile` → `status: ok` when enabled
- [ ] Browser: login, register, forgot-password with widget
- [ ] Invalid / missing token → 400; burst → 429
- [ ] Managed Challenge rule live on staging, then production
