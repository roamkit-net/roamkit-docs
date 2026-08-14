# Promote parity checklist (`develop` → `main`)

Use this **before every production promote** (web and/or API). Goal: production
has the same *intended* behaviour as staging — including bake-time flags — so
staging-only surprises (e.g. Android Install eSIM CTA) do not recur.

Architecture: [ADR 013](../adr/013-production-launch.md). Branching:
[standards/branching.md](../../standards/branching.md).

| Field | Value |
|-------|-------|
| Date (UTC) | |
| Operator | |
| Repos in this promote | ☐ `roamkit-web` ☐ `roamkit-api` ☐ other: ____ |
| Staging tip SHA (web) | |
| Staging tip SHA (api) | |
| Promote PR(s) | |

---

## Why commit-count lies

After squash-merge, `git rev-list origin/main..origin/develop` often shows dozens
of commits even when **file trees match**. Ignore ahead-count for parity.

**Source of truth for code:** two-dot content diff.

```bash
git fetch origin

# Empty output = trees identical (code parity)
git -C roamkit-web diff --stat origin/main origin/develop
git -C roamkit-api diff --stat origin/main origin/develop
```

Record the diff summary in the promote PR body (or “empty — content parity”).

---

## Pre-merge checklist

### A. Code parity

- [ ] `git diff --stat origin/main origin/develop` reviewed for each promoted repo
- [ ] Diff matches the intended release (no surprise files / unfinished work)
- [ ] If promoting only a slice: release branch from `main` + cherry-pick / checkout
  of specific paths is documented (avoid accidental mega-promote)

### B. Staging verification

- [ ] Staging deployed from current `develop` tip (`/version` matches tip SHA)
- [ ] Smoke the feature(s) this promote is meant to ship on **staging**
- [ ] Cross-repo: if API + web both changed, smoke the contract on staging first
  (see branching “Coordinating cross-repo changes”)

### C. Bake / flag matrix (web)

`NEXT_PUBLIC_*` is baked at **Docker build time**. Runtime `.env` alone does not
enable client CTAs. Staging and production use **separate** Actions vars.

| Capability | Staging Actions var | Production Actions var | Staging | Prod intended |
|------------|---------------------|------------------------|---------|---------------|
| API URL / app URL | (workflow hardcodes by branch) | (workflow hardcodes by branch) | staging hosts | prod hosts |
| Turnstile sitekey | `NEXT_PUBLIC_TURNSTILE_SITE_KEY` | `NEXT_PUBLIC_TURNSTILE_SITE_KEY_PRODUCTION` | | |
| Google OAuth client id | `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | `NEXT_PUBLIC_GOOGLE_CLIENT_ID_PRODUCTION` | | |
| Android Install eSIM CTA | `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK` | `NEXT_PUBLIC_ANDROID_LPA_DEEP_LINK_PRODUCTION` | | |

```bash
# Repo: roamkit-web
gh variable list | rg 'TURNSTILE|GOOGLE|ANDROID|PRODUCTION'
```

- [ ] For every staging-on feature that must be on in prod: matching `_PRODUCTION`
  var is set (or intentional OFF documented)
- [ ] Empty `_PRODUCTION` = kill switch / off — confirm that is deliberate
- [ ] CI bake guards still apply: `main` must not bake staging Turnstile/Google ids

Related: [Cloudflare auth](./cloudflare-auth-protection.md),
[Google OAuth](./google-oauth.md),
[Android LPA spike](./android-lpa-deep-link-spike.md).

### D. Runtime flags (API / host `.env`)

Web bake matrix above does **not** cover API feature flags. Check production
host `.env` (never commit secrets) against staging intent:

| Flag (examples) | Staging | Prod intended | Notes |
|-----------------|---------|---------------|-------|
| `BILLING_ENABLED` | | | |
| `TURNSTILE_ENABLED` | | | needs sitekey bake + secret |
| `WALLETCONNECT_ENABLED` | | | |
| `VOUCHERS_ENABLED` | | | |
| `SUBSCRIPTIONS_ENABLED` | | | |
| `PRICING_PROFILES_ENABLED` | | | ADR 019 — do not flip casually |
| Other new flags this promote | | | |

Baseline cutover matrix: [production-go-live-checklist.md](./production-go-live-checklist.md).

- [ ] New flags introduced on `develop` since last promote are listed above
- [ ] Prod value is ON only when product intends it (staging ON ≠ auto-enable prod)

### E. Promote PR hygiene

- [ ] PR targets `main`; title `release: promote … to production`
- [ ] Body includes: tip SHA(s), content-diff summary, bake/flag decisions
- [ ] Squash-merge when CI green (or true-merge only if ancestry requires it —
  document why)

---

## Post-merge checklist

- [ ] CI on `main` green: lint/tests + Docker build + `deploy-production`
- [ ] Live versions:

```bash
curl -sS https://staging.roamkit.net/version
curl -sS https://roamkit.net/version
# API (when public /version is the intended surface):
curl -sS https://api.staging.roamkit.net/version
curl -sS https://api.roamkit.net/version
```

- [ ] Prod `environment` is `production`; web `git_sha` matches the promote merge
  (or documented image pin)
- [ ] Repeat staging smoke on **production** for the shipped feature(s)
- [ ] If a bake flag was enabled: confirm UI/CTA present (e.g. Android Install eSIM
  on a real device)

### Host `.env` vs image bake

`ROAMKIT_*` / image pins in stack `.env` can **override** what the image baked
(see rollback evidence). After deploy:

- [ ] Host pins match the intended release SHA/tag (or are cleared so image bake wins)

---

## Quick copy-paste (operator)

```bash
# From workspace root
git -C roamkit-web fetch origin
git -C roamkit-api fetch origin

echo '=== CONTENT DIFF (must review) ==='
git -C roamkit-web diff --stat origin/main origin/develop
git -C roamkit-api diff --stat origin/main origin/develop

echo '=== LIVE /version ==='
curl -sS https://staging.roamkit.net/version; echo
curl -sS https://roamkit.net/version; echo

echo '=== WEB BAKE VARS ==='
gh -R roamkit-net/roamkit-web variable list | rg 'TURNSTILE|GOOGLE|ANDROID|PRODUCTION'
```

---

## Sign-off

- [ ] Code + bake/flags reviewed
- [ ] Staging smoke done
- [ ] Promote merged and prod smoke done
- **Signed off by:** ____ **Date:** ____
