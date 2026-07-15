# Provider Abstractions

External eSIM and payment systems are accessed through **provider interfaces** (Python `Protocol` types). Domain code in `catalog`, `orders`, and `esims` never imports Airalo HTTP clients directly.

See [ADR 004](../adr/004-provider-interfaces.md).

## Why

- Swap or add providers (Airalo, future partners) without refactoring domain services.
- Test domain logic with fakes that implement the same protocols.
- Keep integration-specific retries, auth, and parsing inside `integrations/`.

## Package catalog

```python
# src/shared/providers/esim.py (or src/core/providers/)

class PackageProvider(Protocol):
    def list_packages(self, filters: PackageFilters) -> list[Package]: ...
```

**Consumer:** `catalog/services/PackageSyncService`  
**Implementation:** `integrations/airalo/providers.py` → `AiraloPackageProvider`

**Celery task:** `sync_airalo_packages` calls the service, not the HTTP client.

## Orders

```python
class OrderProvider(Protocol):
    def create_order(self, package_id: str, customer_ref: str) -> OrderResult: ...
```

**Consumer:** `orders/services/OrderService`  
**Implementation:** `AiraloOrderProvider`

## Top-ups

```python
class TopupProvider(Protocol):
    def list_topups(self, iccid: str) -> list[TopupPackage]: ...
    def submit_topup(self, iccid: str, package_id: str) -> TopupResult: ...
```

**Consumer:** `esims/services/TopupService`  
**Implementation:** Airalo (same module as order provider or split by concern).

## Payment (later)

```python
class PaymentProvider(Protocol):
    def create_checkout_session(self, order_id: str, amount: Money) -> CheckoutSession: ...
    def handle_webhook(self, payload: bytes, signature: str) -> WebhookResult: ...
```

**Implementation:** `integrations/stripe/` — not used until Faza 3.

## Wiring (dependency injection)

Settings or a small registry selects the active provider implementation:

```python
# config/settings/base.py — conceptual
PACKAGE_PROVIDER = "integrations.airalo.providers.AiraloPackageProvider"
```

Services receive providers via constructor injection or a thin factory in `core` — avoid global singletons in domain code.

## Data types

Provider methods return **domain DTOs** (`Package`, `OrderResult`, `TopupResult`), not raw API JSON. Mapping happens inside `integrations/airalo/`.

## Testing

| Layer | Test approach |
|-------|---------------|
| Domain services | Fake providers implementing protocols |
| Airalo client | HTTP mocks (responses from sandbox fixtures) |
| E2E staging | Sandbox credentials in staging `.env` only |

## Adding a new provider

1. Implement protocols in `integrations/<vendor>/providers.py`.
2. Add client + mapping in `integrations/<vendor>/client.py`.
3. Register in settings behind a feature flag or env var.
4. No changes to `catalog/`, `orders/`, or `esims` service signatures.

## Related

- [Directory structure](./directory-structure.md)
- [Domain events ADR](../adr/005-domain-events.md) — `AiraloOrderCreated` after successful order
