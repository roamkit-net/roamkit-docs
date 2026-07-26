# ADR 015: Google OAuth via Google Identity Services (ID token)

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Auth capability design lock (post-Turnstile) |

## Context

RoamKit auth is email/password registration plus SimpleJWT (`POST /api/v1/auth/token/`), with JWTs stored in the web app’s `localStorage`. Users expect **Continue with Google** without introducing server sessions, redirect callbacks, or a parallel token system.

django-allauth / Authorization Code / PKCE would add callback infrastructure and session coupling that the SPA stack does not use. Identity linking (existing password account vs new Google user) must be explicit and safe.

## Decision

Adopt **Google Identity Services (GIS) ID-token** sign-in:

1. Browser obtains a Google ID token (`credential`).
2. Web posts it to `POST /api/v1/auth/google/`.
3. API verifies the JWT (signature, `iss`, `aud`, `exp`/`iat` with skew), resolves or creates the user, and returns the **same** `{ "access", "refresh" }` SimpleJWT pair as password login.

### Product decisions

- **A. One Google account = one RoamKit user.** The same `google_sub` never links to two users (DB unique + HTTP 409). Admin reassignment only via explicit migration/runbook.
- **B. Password and Google are equal auth methods.** Linking Google does not remove or replace a usable password. Google is an additional path into the same JWT pipeline.

### Architecture Constraints (immutable)

1. Google OAuth **never** issues its own tokens; always SimpleJWT (`RefreshToken.for_user` / same pair shape as `/auth/token/`).
2. `google_sub` is the **only** durable Google identity.
3. **No other app** (`billing`, `orders`, `esims`, …) may import Google provider code — Auth/`accounts` only.
4. Capability fully off via `GOOGLE_OAUTH_ENABLED` (API **404** + web hides button when client id empty / flag off).
5. Adding Apple/etc. = **new provider module + new endpoint**; must not require rewriting Google module behavior.
6. Endpoint `/auth/google/` behavior is **stable** across future providers (versioning by additive endpoints only).

### Endpoint and flags

| Item | Value |
|------|--------|
| Endpoint | `POST /api/v1/auth/google/` |
| Body | `{ "credential": "<google_id_token>" }` |
| Success | `{ "access": "…", "refresh": "…" }` |
| Flag | `GOOGLE_OAUTH_ENABLED` (default `false`) |
| Audience | `GOOGLE_OAUTH_CLIENT_ID` |
| Production | Fail-fast if enabled without client id |
| Turnstile | **Not** required on `/auth/google/` (password `/token/` keeps Turnstile) |
| Throttle | Separate bucket `auth_google` / `AUTH_GOOGLE_RATE` (≠ `auth_token`) |

### Schema (`accounts.User`)

| Field | Notes |
|-------|--------|
| `google_sub` | Nullable unique; **immutable** once set; primary Google identity |
| `google_picture`, `google_name` | UI cache only; refresh on successful Google login; never authorize from these |
| `last_login_provider` | `password` \| `google` (last used, not “primary”) |
| `last_google_login_at` | Set on successful Google login |

New Google-only users: `is_active=True`, unusable password, `ensure_billing_account`. Pending email-register users (`is_active=False`): on auto-link, set `google_sub`, activate, leave password unusable.

### Rules (identity)

1. `google_sub` immutable once set (login path never clears/reassigns).
2. `google_sub` already on another user → **409** `google_sub_conflict`.
3. Primary identity = `google_sub`. Email is used only for the **first** auto-link when `email_verified=true` and the target user has no `google_sub`.
4. If Google later changes email: login still succeeds by `google_sub`. **v1 does not mutate `User.email`.**
5. Users with unusable password work after link (`RefreshToken.for_user`).

### Email and hosted domain

- Normalize email before any lookup/link: Django `normalize_email` + strip/lowercase — never compare raw token email strings.
- Google `hd` (hosted domain) is **ignored in v1** (future Business allowlists).

### Soft delete / session policy

- When `is_deleted` / disable flags exist later, Google login must obey the **same** gates as password login.
- Today: refuse if `is_active=False` with **HTTP 401** (same as password `/auth/token/`).
- Unlink UI is out of v1. Future unlink that must kill refresh tokens requires blacklist or token versioning (not in this ADR’s implementation scope). Prefer flag/image rollback over reverse-migrating columns in production.

### Claim whitelist

Use only: `sub`, `email`, `email_verified`, `name`, `picture`, `iss`, `aud`, `exp`, `iat`. Ignore all other claims.

### Privacy

Never persist Google access/refresh tokens, raw credential, or ID token. Flow: verify → extract whitelisted claims → **discard JWT**. Store only `google_sub` + cache name/picture + timestamps.

### Verify (JWKS, skew, timeout)

- Use `google-auth` (or equivalent) so Google certs are cached and Cache-Control respected.
- Clock skew: **±60 seconds** on `exp` / `iat`.
- Bounded HTTP timeout (≈2–3s). On timeout/network → **503** `google_verify_unavailable`, never hang.
- Performance budget (target, not hard merge gate): verify path **p95 &lt; 300 ms** on JWKS cache hit.

### Login transaction (concurrency)

```text
verify token (outside or before lock; timeout bounded)
BEGIN
  select_for_update on matched user row(s) as needed
  link or create (unique google_sub)
  update profile cache / last_* fields
COMMIT
issue JWT after commit
```

Idempotent: repeated identical successful credential → same user, one `google_sub`, JWT each time (no duplicate users).

### Resolution algorithm

```text
verify id_token (sig, iss, aud=CLIENT_ID, exp/iat ±60s, cached JWKS)
require email_verified else 400 google_email_not_verified
normalize email
lookup by google_sub:
  hit → if inactive: 401 google_account_disabled
        else refresh profile cache; last_login_provider=google; last_google_login_at=now; JWT
  miss →
    if email matches User:
      select_for_update
      if inactive (and not pending-activate path): 401
      if user.google_sub set and != sub → 409 google_sub_conflict
      else first-link google_sub (+ activate if pending); profile cache; JWT
    else create User + billing; JWT
emit events + metrics; structured log (no email)
```

### Error contract + HTTP statuses (locked — do not drift)

Stable machine `code` + human `detail`. Changing statuses requires a new ADR.

| HTTP | `code` | When |
|------|--------|------|
| 400 | `google_invalid_token` | bad/expired/wrong aud/iss/sig / malformed credential |
| 400 | `google_email_not_verified` | `email_verified != true` |
| 401 | `google_account_disabled` | inactive / future soft-delete (match password `/token/`) |
| 404 | `google_feature_disabled` | `GOOGLE_OAUTH_ENABLED=false` (opaque; body optional) |
| 409 | `google_sub_conflict` | sub already on another user / email owned with different sub |
| 503 | `google_verify_unavailable` | JWKS/network/timeout |

```json
{ "code": "google_email_not_verified", "detail": "Google account email is not verified." }
```

Clients map on `code`, not free-text `detail`.

### OpenAPI

`operation_id=auth_google` must document:

- Request example: `{ "credential": "<google_id_token>" }`
- 200 example: `{ "access": "…", "refresh": "…" }`
- One example per error code above (400 / 401 / 404 / 409 / 503)

### Migration rollback

- Forward migration adds nullable Google fields; **reversible**.
- Rollback **must not** delete `User` rows or billing accounts.
- Prefer: reverse deploy (flag off / previous image) **before** reversing migration in production.
- Reversing migration drops `google_*` / `last_*` columns only (data loss for those fields, not the User account).

### Module layout

```text
apps/accounts/providers/google/
  verify.py
  service.py
  errors.py
```

### Domain events

Publish via in-process bus (ADR 005), e.g.:

- `GoogleLoginSucceeded` — `user_id`, `google_sub`
- `GoogleLoginFailed` — `reason`
- `GoogleAccountLinked` — first auto-link
- `GoogleAccountCreated`
- `GoogleLoginConflict`

### Metrics (`core.metrics`)

- `google_login_success_total`
- `google_login_failure_total` (+ `reason`)
- `google_auto_link_total`
- `google_new_user_total`
- `google_conflict_total`

### Structured logging

Fields: `provider=google`, `google_sub` (when known), `user_id` (when known), `result`, `reason`. **Never log email.**

### Feature-flag rollout (v1 = boolean only)

```text
internal (flag off)
  → staging enable (OAuth Testing + test users)
  → production dark (flag false)
  → production enable (100%)
```

Percentage traffic splitting is a future option (edge/flag service), not v1 code.

### Future providers

Thin seam: verify external credential → identity (`provider`, `subject`, email, verified, profile) → resolve/link/issue JWT. v1 implements Google only. Next provider = sibling module + `POST /auth/<provider>/` without changing `/auth/google/` behavior.

### Security checklist

- [ ] Verify signature
- [ ] Verify issuer
- [ ] Verify audience
- [ ] Verify expiration (with skew)
- [ ] Require `email_verified`
- [ ] Never trust frontend payload without verify
- [ ] Never decode JWT without signature verification
- [ ] Never use email as permanent identity
- [ ] `google_sub` immutable; one sub ↔ one user

## Rejected alternatives

| Alternative | Why not |
|-------------|---------|
| Authorization Code + redirect | Needs callback routes/CSRF state; SPA already uses Bearer JWT in localStorage |
| PKCE | Same redirect surface; unnecessary for GIS ID-token verify |
| django-allauth / SocialAccount | Heavy session-oriented stack; conflicts with SimpleJWT-only auth |
| Authlib server session | Parallel session model |
| Email as permanent Google identity | Account takeover / email change hazards; `sub` is stable |

## Consequences

### Positive

- One JWT pipeline for password and Google.
- Clear error contract for web and support.
- Feature flag allows dark deploy.

### Negative / follow-ups

- Unlink and refresh-token invalidation deferred.
- OAuth consent “Testing” limits smoke users until production publish.
- Profile cache (`name`/`picture`) can drift until next Google login.

## References

- Ops: `docs/ops/google-oauth.md`
- Related: ADR 005 (domain events), Cloudflare Turnstile ops (password path only)
