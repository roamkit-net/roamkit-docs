# Python Coding Standard

Applies to `roamkit-api`.

## Runtime

- Python 3.12+ (align with CI image when defined).
- Django + Django REST Framework + Celery.

## Layout

Follow [ADR 003](../docs/adr/003-src-layout.md):

- Application code under `src/`.
- Tests under `tests/` mirroring app structure.
- Requirements split: `requirements/base.txt`, `requirements/dev.txt`.

## Style and lint

| Tool | Scope |
|------|-------|
| **Ruff** | Lint + import order |
| **Black** | Formatting (line length 88 or project default) |

Run before PR:

```bash
ruff check src tests
black --check src tests
```

## Naming

| Element | Convention | Example |
|---------|------------|---------|
| Modules | `snake_case` | `package_sync.py` |
| Classes | `PascalCase` | `PackageSyncService` |
| Functions | `snake_case` | `list_packages` |
| Constants | `UPPER_SNAKE` | `DEFAULT_PAGE_SIZE` |

## Architecture rules

1. **Services own business logic** — views and serializers stay thin.
2. **No Airalo imports outside `integrations/airalo/`** — use provider protocols.
3. **Publish domain events** after successful mutations; do not call notification code inline.
4. **Settings** — `config.settings.base` + environment-specific modules; secrets from env vars only.

## Django specifics

- Custom user model in `accounts` if needed; set `AUTH_USER_MODEL` early.
- Migrations committed with model changes; no manual SQL on staging without review.
- Use `select_related` / `prefetch_related` in list endpoints to avoid N+1.
- Permissions: default deny; explicit `permission_classes` on every view.

## Celery

- Tasks in `<app>/tasks.py` or `tasks/` package.
- Tasks call services, not HTTP clients directly.
- Idempotent tasks where external APIs allow retries.
- Task names: `app.module.verb` e.g. `catalog.sync_airalo_packages`.

## Testing

- **pytest** + **pytest-django**.
- Use factory_boy or fixtures for models.
- Mock providers at service boundary; test Airalo client separately.
- Minimum coverage threshold enforced in CI (target: 80% when codebase grows).

## Security

- No secrets in code or logs.
- Validate and sanitize all external input.
- Parameterized queries only (ORM).
- Bandit scan in CI — no high-severity findings without documented exception.

## Related

- [Provider abstractions](../docs/architecture/provider-abstractions.md)
- [API versioning](./api-versioning.md)
- [Definition of Done](./definition-of-done.md)
