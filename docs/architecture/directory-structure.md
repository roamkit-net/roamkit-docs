# Directory Structure

Canonical layout for RoamKit repos. Deviations require an ADR update or a new ADR.

## Workspace (WSL)

```
~/projects/roamkit-net/
├── roamkit-docs/      # This repo
├── roamkit-infra/     # Bootstrap, compose, CI, deploy
├── roamkit-api/       # Django API (Faza 0+)
└── roamkit-web/       # Next.js (Faza 0+)
```

Code lives under `~/projects/`, not `/mnt/c/`. See workspace README.

## roamkit-infra

```
roamkit-infra/
├── bootstrap/
│   ├── github/           # create-org, create-repos, branch-protection, labels
│   ├── docker/           # verify-local.sh
│   └── hetzner/          # init-staging-stack, smoke-test, prerequisites
├── docker/
│   ├── docker-compose.dev.yml
│   ├── docker-compose.staging.yml
│   ├── docker-compose.test.yml
│   └── .env.example
├── ci/workflows/         # Templates copied to repos or org .github
├── scripts/
│   └── deploy-staging.sh
└── README.md
```

**Hetzner staging path:** `/opt/stacks/roamkit-net/` (compose, `.env`, data volumes). Routing via shared Traefik (`proxy` network).

## roamkit-api (`src/` layout)

See [ADR 003](../adr/003-src-layout.md).

```
roamkit-api/
├── src/
│   ├── config/                 # settings: base, dev, staging, production
│   ├── core/                   # base models, exceptions, permissions
│   ├── shared/
│   │   ├── events/
│   │   │   ├── event_bus.py
│   │   │   └── events.py
│   │   └── utils/
│   └── apps/
│       ├── accounts/
│       │   └── services/
│       ├── catalog/
│       │   └── services/       # PackageSyncService
│       ├── orders/
│       │   └── services/       # OrderService
│       ├── esims/
│       │   └── services/       # UsageService, TopupService
│       ├── billing/
│       │   └── services/
│       └── integrations/
│           ├── airalo/
│           │   ├── client.py
│           │   ├── providers.py
│           │   └── services/
│           └── stripe/         # later
├── tests/
├── requirements/
│   ├── base.txt
│   └── dev.txt
└── manage.py
```

### App responsibilities

| App | Owns |
|-----|------|
| `accounts` | Users, registration, JWT |
| `catalog` | Package listing, sync from provider |
| `orders` | Order lifecycle |
| `esims` | ICCID, usage, top-ups |
| `billing` | Invoices, Stripe (later) |
| `integrations` | Third-party HTTP clients and provider implementations |

Business logic lives in `services/` modules, not in views or serializers.

## roamkit-web

```
roamkit-web/
├── app/
│   ├── page.tsx
│   ├── plans/page.tsx
│   ├── how-it-works/page.tsx
│   ├── contact/page.tsx
│   └── layout.tsx
├── lib/
│   └── api.ts
├── components/
└── ...
```

Next.js 15 App Router. API client in `lib/api.ts` targets `NEXT_PUBLIC_API_URL`.

## roamkit-docs

```
roamkit-docs/
├── ADR_INDEX.md
├── docs/
│   ├── architecture/
│   ├── adr/
│   └── rfcs/
├── standards/
└── README.md
```

## Tests

| Repo | Location | Runner |
|------|----------|--------|
| `roamkit-api` | `tests/` | pytest with `docker-compose.test.yml` in CI |
| `roamkit-web` | colocated or `__tests__/` | vitest/jest per project setup |

CI uses the same PostGIS and Redis images as local dev. See [Docker standard](../../standards/docker-standard.md).
