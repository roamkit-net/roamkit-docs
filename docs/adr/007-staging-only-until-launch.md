# ADR 007: Staging-only deploy until production launch

| Field | Value |
|-------|-------|
| Status | Superseded |
| Date | 2026-07 |
| Deciders | Architecture Freeze |
| Superseded by | [ADR 013](./013-production-launch.md) |

## Context

RoamKit needs a shared integration environment before public launch. Auto-deploying `main` to production too early increases risk while the product and partner credentials are still in sandbox/test mode.

## Decision (historical)

| Branch | CI | Auto-deploy |
|--------|-----|-------------|
| `feature/*` | On PR (lint, test, security, build) | No |
| `develop` | Full pipeline | **Yes → staging** |
| `main` | Full pipeline | **No** until Faza 4 |

**Staging URLs:**

- `https://staging.roamkit.net` — Next.js
- `https://api.staging.roamkit.net` — Django API

**Production (later):**

- `https://roamkit.net`, `https://api.roamkit.net`
- Stack at `/opt/stacks/roamkit-production/` — separate from staging, no shared volumes.

Airalo sandbox credentials stay on staging until production launch. Early drafts mentioned Stripe test keys; Faza 3 replaced cards with Polygon USDT prepaid credits ([ADR 010](./010-polygon-usdt-prepaid-credits.md)).

## Supersession

[ADR 013](./013-production-launch.md) accepts production URLs, `/opt/stacks/roamkit-production/`, and **`main` → production auto-deploy** once the production stack and deploy CI exist. Staging continues to deploy from `develop`.

The gate conditions below are satisfied by ADR 013 + the [go-live checklist](../ops/production-go-live-checklist.md) (infra still must be provisioned before wiring deploy).

## Consequences

### Positive

- `main` remained free of customer-facing deploy risk during Faza 0–3.
- Staging was the contract-testing environment for web + API + partner sandboxes.

### Negative

- Releases to real users waited until the production stack and ADR 013 existed.

### Gate to change (met by ADR 013)

Production auto-deploy requires:

1. New or superseding ADR accepting production URLs and deploy trigger → **ADR 013**.
2. Production stack provisioned in `roamkit-infra` → Faza 4 PR1+.
3. Live partner credentials in production `.env` only → Airalo production + Polygon USDT (not Stripe).

## Related

- [ADR 013](./013-production-launch.md) — production launch (current)
- [Branching standard](../../standards/branching.md)
- [Definition of Done](../../standards/definition-of-done.md)
