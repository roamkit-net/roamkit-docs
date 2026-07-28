# Google OAuth (GIS ID token) — ops runbook

Application auth lives in `roamkit-api` / `roamkit-web` per
[ADR 015](../adr/015-google-oauth-gis.md). This note covers **Google Cloud
clients**, **env/bake**, **enable order**, **support**, and **observability**.

## Architecture (ops view)

```text
Browser GIS button → credential (ID token)
        ↓
POST /api/v1/auth/google/  { "credential": "…" }
        ↓
API verifies (JWKS cache) → link/create User → SimpleJWT { access, refresh }
```

No Google client **secret** is required for v1 ID-token verify. Do not put
secrets from Google Cloud JSON downloads into git or stack `.env`.

## Feature flags / env

| Variable | Where | Notes |
|----------|--------|--------|
| `GOOGLE_OAUTH_ENABLED` | API | Default `false`; production fail-fast if true without client id |
| `GOOGLE_OAUTH_CLIENT_ID` | API | Audience for ID token verify |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Web bake | Empty → hide Continue with Google |
| `AUTH_GOOGLE_RATE` | API | Separate throttle from password (`auth_token`) |
| `GOOGLE_OAUTH_VERIFY_TIMEOUT` | API | Default `2.5` seconds |
| `GOOGLE_OAUTH_CLOCK_SKEW_SECONDS` | API | Default `60` |

Diagnostic: `GET /health/google-oauth` — config only (`enabled`, `client_id_configured`), no Google network call.

Web bake: pass `NEXT_PUBLIC_GOOGLE_CLIENT_ID` as Docker build-arg (staging vs
production GitHub Actions vars). Runtime `.env` alone is not enough for the
client bundle.

| Bake var | Branch |
|----------|--------|
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | non-`main` (staging) |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID_PRODUCTION` | `main` |

CI fails if `main` bakes the staging client id or `develop` bakes the production client id.

## Client inventory (public)

**GCP:** dedicated RoamKit project under `qubitsecured-org` (not Predix).

**OAuth consent contact / Testing test user:** `qubitsecured@gmail.com`

| Env | Client name | Client ID | Authorized JavaScript origins |
|-----|-------------|-----------|-------------------------------|
| Staging | `roamkit-staging-web` (confirm) | `1077069744172-tdjo47kd9r7u5vkecldr53vctecdmnue.apps.googleusercontent.com` | `https://staging.roamkit.net` |
| Production | `roamkit-production-web` | `1077069744172-2fggulf6olc93i102e2d6gfok808u3t7.apps.googleusercontent.com` | `https://roamkit.net`, `https://www.roamkit.net` |

Authorized redirect URIs: leave empty for GIS ID-token flow.

While the OAuth app is in **Testing**, only listed test users can complete Google
sign-in. Publish to **In production** before public launch.

## Enable order (staging)

```text
API dark (GOOGLE_OAUTH_ENABLED=false) → deploy
        ↓
set GOOGLE_OAUTH_CLIENT_ID + bake NEXT_PUBLIC_GOOGLE_CLIENT_ID
        ↓
GOOGLE_OAUTH_ENABLED=true → recreate API
        ↓
redeploy web with baked client id
        ↓
smoke Continue with Google on /login and /register
        ↓
confirm JWT + GET /api/v1/auth/me/
```

Staging example:

```bash
GOOGLE_OAUTH_ENABLED=true
GOOGLE_OAUTH_CLIENT_ID=1077069744172-tdjo47kd9r7u5vkecldr53vctecdmnue.apps.googleusercontent.com
NEXT_PUBLIC_GOOGLE_CLIENT_ID=1077069744172-tdjo47kd9r7u5vkecldr53vctecdmnue.apps.googleusercontent.com
```

Prefer **flag off / previous image** rollback over reversing the `google_*`
migration in production (reverse migration drops column values only; User rows
remain).

## Support runbook

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| “Access blocked” / app not verified | OAuth app still in Testing; user not a test user | Add email under Audience → Test users, or publish app |
| Google email changed in Google account | Login still works via `google_sub`; RoamKit email unchanged (v1) | Explain; do not “fix” email via Google claims |
| Google account deleted at Google | Cannot obtain new credential | User may still password-login if they have a password; else support reset/register |
| Already linked | Same Google account signs in to existing RoamKit user | Expected; password still works if set |
| Conflict (`google_sub_conflict` / 409) | Email belongs to a user already linked to a **different** `google_sub` | Do not auto-merge; investigate in admin (`google_sub` readonly) |
| Disabled account (401) | `is_active=false` with usable password, or linked inactive | Same as password login; re-enable in admin if legitimate |
| Button missing | Empty bake client id or script blocked | Check `NEXT_PUBLIC_GOOGLE_CLIENT_ID`, CSP, network |
| 404 on `/auth/google/` | `GOOGLE_OAUTH_ENABLED=false` | Enable flag after client id set |
| 503 `google_verify_unavailable` | JWKS/network timeout | Retry; check egress to Google |

Never ask users to paste ID tokens into tickets. Never log emails from Google
auth paths (structured logs use `google_sub` / `user_id` / `result` / `reason`).

## Observability (panel list)

Source: `core.metrics` / structured logs (Sentry metrics when available). Doc-only
dashboard list for v1:

| Panel | Metric / signal |
|-------|-----------------|
| Google login / min | `google_login_success_total` rate |
| Success % | success / (success + failure) |
| Failure % by code | `google_login_failure_total` `reason` tag |
| New users | `google_new_user_total` |
| Auto-link | `google_auto_link_total` |
| Conflicts | `google_conflict_total` |
| Verify latency | timing logs / future histogram (p95 target &lt; 300 ms on cache hit) |
| Throttle blocks | `auth_throttle_block_total` scope=`auth_google` |

## Related

- [ADR 015](../adr/015-google-oauth-gis.md)
- [Cloudflare auth protection](./cloudflare-auth-protection.md) (password path Turnstile; not on `/auth/google/`)
