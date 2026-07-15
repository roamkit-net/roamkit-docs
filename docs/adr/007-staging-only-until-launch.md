# ADR 007: Staging-only deploy until production launch

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

RoamKit needs a shared integration environment before public launch. Auto-deploying `main` to production too early increases risk while the product and partner credentials are still in sandbox/test mode.

## Decision

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

Airalo sandbox and Stripe test keys stay on staging until production launch ADR amends this decision.

## Consequences

### Positive

- `main` remains a stable integration branch without customer-facing deploy risk.
- Staging is the contract-testing environment for web + API + partner sandboxes.
- Production cutover is an explicit Faza 4 milestone with its own checklist.

### Negative

- Releases to real users wait until production stack exists.
- `main` must be manually verified or promoted via release process before prod.

### Gate to change

Production auto-deploy requires:

1. New or superseding ADR accepting production URLs and deploy trigger.
2. Production stack provisioned in `roamkit-infra`.
3. Airalo production + Stripe live credentials in production `.env` only.

## Related

- [Branching standard](../../standards/branching.md)
- [Definition of Done](../../standards/definition-of-done.md)
