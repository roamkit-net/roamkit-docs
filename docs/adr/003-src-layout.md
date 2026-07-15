# ADR 003: Django `src/` layout

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

Django projects often place apps at the repository root, which mixes application code with `manage.py`, `requirements/`, and tooling. A `src/` layout keeps the import root explicit and matches common Python packaging practice.

## Decision

`roamkit-api` uses a **`src/` layout**:

```
roamkit-api/
├── src/
│   ├── config/          # Django project (settings, urls, wsgi, celery)
│   ├── core/
│   ├── shared/
│   └── apps/            # accounts, catalog, orders, esims, billing, integrations
├── tests/
├── requirements/
└── manage.py
```

- `PYTHONPATH` or `manage.py` adds `src/` to the path.
- Django apps live under `src/apps/<name>/`.
- Settings modules: `config.settings.base`, `.dev`, `.staging`, `.production`.

## Consequences

### Positive

- Clear separation of deployable code vs tests and requirements.
- Easier to enforce import boundaries (e.g. `apps/*` must not import from sibling apps except via services/events).
- Aligns with ruff/black running against `src/` and `tests/`.

### Negative

- IDE and pytest need explicit path configuration (`pytest.ini`, `pyproject.toml`).

### Conventions

- Business logic in `<app>/services/`, not in views.
- `integrations/` is the only place for third-party HTTP/SDK code.
- Shared cross-cutting code in `shared/` and `core/`.

## Related

- [Directory structure](../architecture/directory-structure.md)
- [Python coding standard](../../standards/coding-standard-python.md)
