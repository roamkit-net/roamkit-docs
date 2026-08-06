# Architecture Decision Records — Index

All ADRs live in [docs/adr/](./docs/adr/).

**Normative ADR status:** **Draft**, **Accepted**, **Superseded**, **Deprecated**.

**Delivery overlay** (ops / maturity — not a substitute for ADR status):
**Accepted → Implemented → Operational**. Track runtime maturity in the
[Operations Handbook](./docs/ops/README.md) Maturity Matrix. Revisit Accepted ADRs
every **6 months** (Architecture Decision Review: still valid?).

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [001](./docs/adr/001-repo-split.md) | Multi-repo split (api, web, infra, docs) | Accepted | 2026-07 |
| [002](./docs/adr/002-postgis-from-day-one.md) | PostGIS from day one | Accepted | 2026-07 |
| [003](./docs/adr/003-src-layout.md) | Django `src/` layout | Accepted | 2026-07 |
| [004](./docs/adr/004-provider-interfaces.md) | Provider interfaces for eSIM integrations | Accepted | 2026-07 |
| [005](./docs/adr/005-domain-events.md) | In-process domain event bus | Accepted | 2026-07 |
| [006](./docs/adr/006-ghcr-pull-only-deploy.md) | GHCR pull-only deploy (no server builds) | Accepted | 2026-07 |
| [007](./docs/adr/007-staging-only-until-launch.md) | Staging-only deploy until production launch | Superseded | 2026-07 |
| [008](./docs/adr/008-bootstrap-iac-gh-cli.md) | Bootstrap IaC via `gh` CLI | Accepted | 2026-07 |
| [009](./docs/adr/009-shared-traefik-edge.md) | Shared Traefik edge routing (proxy network) | Accepted | 2026-07 |
| [010](./docs/adr/010-polygon-usdt-prepaid-credits.md) | Polygon USDT prepaid credits | Accepted | 2026-07 |
| [011](./docs/adr/011-credit-vouchers-gift-codes.md) | Credit vouchers & gift codes | Accepted | 2026-07 |
| [012](./docs/adr/012-billing-extensibility-rules.md) | Billing extensibility rules | Accepted | 2026-07 |
| [013](./docs/adr/013-production-launch.md) | Production launch | Accepted | 2026-07 |
| [014](./docs/adr/014-esim-lifecycle-install-telemetry.md) | eSIM lifecycle and install telemetry | Accepted | 2026-07 |
| [015](./docs/adr/015-google-oauth-gis.md) | Google OAuth via GIS ID token | Accepted | 2026-07 |
| [016](./docs/adr/016-web-design-tokens.md) | RoamKit web design tokens | Accepted | 2026-08 |
| [017](./docs/adr/017-roamkit-wallet-platform.md) | RoamKit Wallet Platform (architecture adoption) | Accepted — Architecture Complete | 2026-08 |
| [018](./docs/adr/018-wallet-product-activation-strategy.md) | Wallet Product Activation Strategy | **Accepted** | 2026-08 |

## Billing ADR hierarchy

| Layer | ADR | Role |
|-------|-----|------|
| Foundation | [010](./docs/adr/010-polygon-usdt-prepaid-credits.md) | Money path (`CreditService`, ledger, Account ownership) |
| Constitution | [012](./docs/adr/012-billing-extensibility-rules.md) | Rules every future credit source must obey |
| First extension | [011](./docs/adr/011-credit-vouchers-gift-codes.md) | Vouchers / gift codes (must satisfy 012) |
| Wallet constitution | [017](./docs/adr/017-roamkit-wallet-platform.md) | Wallet Platform (RFC 003–006); intake cutover via [018](./docs/adr/018-wallet-product-activation-strategy.md) |
| Wallet activation | [018](./docs/adr/018-wallet-product-activation-strategy.md) | Product activation / ADR 010 shared-wallet cutover (**Accepted**) |

## Deploy ADR hierarchy

| Layer | ADR | Role |
|-------|-----|------|
| Historical gate | [007](./docs/adr/007-staging-only-until-launch.md) | Staging-only until launch (**Superseded**) |
| Current | [013](./docs/adr/013-production-launch.md) | `main` → production; staging from `develop` |
| Ops | [Operations Handbook](./docs/ops/README.md) | Release Engineering index (`docs/ops/`) |
| Ops | [Launch Gates](./docs/ops/launch-gates.md) | Gate A–D entry/exit, freeze, time-boxes |
| Ops | [Go-Live checklist](./docs/ops/production-go-live-checklist.md) | Cutover / must-haves / flags |
| Ops | [Production readiness review](./docs/ops/production-readiness-review.md) | PR0.5 go/no-go (GO WITH CONDITIONS) |
| Ops | [Capability / Maturity Matrix](./docs/ops/capability-status.md) | Design → Implemented → Production → Observed |
| Ops | [Airalo Go-Live Readiness](./docs/ops/airalo-go-live-readiness/README.md) | Partner Evidence Pack; Phase 5–7; Production Freeze |

## RFCs (proposals)

| RFC | Title | Status |
|-----|-------|--------|
| [001](./docs/rfcs/001-self-service-esim-flow.md) | Self-service eSIM purchase and top-up flow | Draft |
| [002](./docs/rfcs/002-post-purchase-onboarding.md) | Post-purchase eSIM onboarding | Accepted |
| [003](./docs/rfcs/003-wallet-domain-ownership-model.md) | Wallet Domain & Ownership Model | Draft |
| [TEMPLATE (Wallet)](./docs/rfcs/TEMPLATE-wallet.md) | Shared structure for Wallet RFCs 003+ | — |

## Architecture reference

- [System overview](./docs/architecture/overview.md)
- [Directory structure](./docs/architecture/directory-structure.md)
- [Provider abstractions](./docs/architecture/provider-abstractions.md)
- [eSIM Auto Top-up v1 — Design Lock](./docs/design/esim-auto-topup-v1-design-lock.md) — **Accepted** (≠ Subscription; Available-topups-only; does **not** amend [ADR 010](./docs/adr/010-polygon-usdt-prepaid-credits.md) money invariants)
- [RoamKit Wallet Platform — Architecture Vision](./docs/architecture/roamkit-wallet-platform-vision.md) — **Draft / Vision** (non-normative; does **not** amend [ADR 010](./docs/adr/010-polygon-usdt-prepaid-credits.md))
- [Wallet Conversion Boundary](./docs/architecture/wallet-conversion-boundary.md) — why product spend stays on Credits after convert
- [Wallet Sandbox](./docs/architecture/wallet-sandbox.md) — research framework (non-normative)

## Standards

- [Branching](./standards/branching.md)
- [Python coding standard](./standards/coding-standard-python.md)
- [TypeScript coding standard](./standards/coding-standard-typescript.md)
- [Docker standard](./standards/docker-standard.md)
- [CI standard](./standards/ci-standard.md)
- [API versioning](./standards/api-versioning.md)
- [Definition of Done](./standards/definition-of-done.md)
