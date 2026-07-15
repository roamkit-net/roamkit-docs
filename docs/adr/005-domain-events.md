# ADR 005: In-process domain event bus

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

Order completion triggers multiple reactions: notifications, analytics, future webhooks. Inline calls from `OrderService` to email, Slack, and metrics create tight coupling and complicate testing.

## Decision

Use an **in-process domain event bus** in `src/shared/events/`:

```python
@dataclass
class AiraloOrderCreated(DomainEvent):
    order_id: str
    iccid: str
    customer_id: str

@dataclass
class PackagesSynced(DomainEvent):
    package_count: int
    source: str
```

- `event_bus.py` dispatches events to registered handlers synchronously at first.
- Handlers live in dedicated modules (e.g. `notifications/handlers.py`).
- Services publish events after successful state changes; they do not call handlers directly.

**Phase 1:** In-process only.  
**Later:** Heavy handlers (email send, external webhooks) move to Celery tasks subscribed via the same event types without changing publishers.

## Consequences

### Positive

- `OrderService` stays focused on order invariants.
- New side effects = new handler registration, no service edits.
- Clear audit trail of domain occurrences for future outbox pattern.

### Negative

- In-process handlers block the request/task until complete (mitigate with thin handlers that enqueue Celery work).
- No distributed transaction between DB commit and event delivery until outbox is added.

### Guidelines

- Events are past-tense facts (`OrderCreated`, not `CreateOrder`).
- Handlers must be idempotent where they touch external systems.
- Do not use the bus for queries — only state-change notifications.

## Related

- [Provider abstractions](../architecture/provider-abstractions.md)
- RFC [001-self-service-esim-flow](../rfcs/001-self-service-esim-flow.md)
