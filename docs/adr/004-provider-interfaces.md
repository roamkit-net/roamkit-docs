# ADR 004: Provider interfaces for eSIM integrations

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

RoamKit's first eSIM partner is Airalo (Partner API). The product may add other wholesalers or direct carrier integrations. If domain code calls Airalo endpoints directly, every new provider forces wide refactors across `catalog`, `orders`, and `esims`.

## Decision

Define **provider protocols** for external capabilities:

| Protocol | Methods | Domain consumer |
|----------|---------|-----------------|
| `PackageProvider` | `list_packages(filters)` | `PackageSyncService` |
| `OrderProvider` | `create_order(package_id, customer_ref)` | `OrderService` |
| `TopupProvider` | `list_topups(iccid)`, `submit_topup(...)` | `TopupService` |

Airalo implements these in `integrations/airalo/providers.py`. Methods return domain DTOs, not raw JSON.

Domain apps **must not** import `integrations.airalo.client`.

## Consequences

### Positive

- Unit tests use in-memory fakes.
- Second provider is a new integration package plus settings wiring.
- API rate limits and auth stay encapsulated.

### Negative

- Extra mapping layer between partner payloads and DTOs.
- Protocol changes require updating all implementations.

### Rules

1. New partner → new `integrations/<vendor>/` package.
2. Celery tasks call services; services call providers.
3. Payment providers follow the same pattern in Faza 3 (`PaymentProvider`).

## Related

- [Provider abstractions](../architecture/provider-abstractions.md)
- [ADR 005](./005-domain-events.md)
