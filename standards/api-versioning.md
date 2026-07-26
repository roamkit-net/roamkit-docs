# API Versioning Standard

REST API conventions for `roamkit-api`.

## URL prefix

All public REST endpoints use versioned paths:

```
/api/v1/<resource>/
```

- **v1** — initial MVP through self-service launch.
- Breaking changes require **v2** prefix; v1 maintained for a defined deprecation window.

## Versioning rules

| Change type | Version bump? |
|-------------|---------------|
| Add optional JSON field | No |
| Add new endpoint | No |
| Rename/remove field | Yes — new major version |
| Change semantics of existing field | Yes |
| Change auth requirements | Usually yes |

Document breaking changes in `roamkit-docs` and changelog; coordinate `roamkit-web` deploy.

**Patch release policy:** a patch release must not change the existing API contract except documentation-only fixes (OpenAPI descriptions, examples, tags).

### Breaking-change checklist (PR review)

If any item is checked, treat as an API breaking change → major version / `/api/v2/` per rules above:

```text
□ removed endpoint
□ renamed endpoint
□ removed field
□ changed required field
□ changed enum value
□ changed status code
```

## Response format

### Success (DRF default)

```json
{
  "count": 42,
  "next": null,
  "previous": null,
  "results": []
}
```

List endpoints use pagination; detail endpoints return a single object.

### Error

```json
{
  "detail": "Human-readable message"
}
```

Field validation errors use DRF standard `{"field": ["message"]}` shape.

Use appropriate HTTP status codes: `400`, `401`, `403`, `404`, `409`, `422`, `429`, `500`.

## Authentication

| Phase | Mechanism |
|-------|-----------|
| Faza 1 | Public read for packages (if product allows) |
| Faza 2+ | JWT — `POST /api/v1/auth/token/`, `POST /api/v1/auth/register/` |

- Bearer token in `Authorization` header.
- Token refresh strategy documented when implemented.

## Health endpoints (unversioned)

```
GET /health/live
GET /health/ready
```

Not under `/api/v1/` — used by infrastructure probes. Excluded from the public OpenAPI contract.

## OpenAPI

Schema is generated with **drf-spectacular** (pinned in `roamkit-api` `requirements/base.txt`).

| Item | Location |
|------|----------|
| Committed artifact | `roamkit-api/openapi/openapi.yaml` |
| Generate (only supported path) | `./scripts/generate_openapi.sh` |
| Staging schema | `https://api.staging.roamkit.net/api/schema/` |
| Staging Swagger UI | `https://api.staging.roamkit.net/api/docs/` |
| Staging ReDoc | `https://api.staging.roamkit.net/api/redoc/` |
| Changelog | [openapi-changelog.md](../docs/api/openapi-changelog.md) |

### Conventions

- **operationId:** explicit snake_case `{domain}_{action}` (e.g. `billing_balance`, `orders_create`). Never rely on spectacular auto-suffixes (`list_1`).
- **Tags (order):** Authentication, Billing, Orders, Catalog, eSIM, Users.
- **Security:** HTTP Bearer JWT (`bearerAuth`) on authenticated operations; public ops clear security.
- **CI:** generate + validate + drift check; Spectral errors fail, warnings report-only; architecture tests require 100% `/api/v1/` path coverage.

### Wave 2 (frontend) — backlog

Not part of C10. When ready:

- Generate TypeScript types with `openapi-typescript` from the committed `openapi.yaml`.
- Output under `roamkit-web/src/api/generated/` must be **read-only** (no hand edits; regenerate only from YAML).
- Frontend CI should fail if generated clients drift from the schema.

## CORS

- Staging: allow `https://staging.roamkit.net`
- Local dev: `http://localhost:3000`
- Production origins added at launch — not wildcard in production.

## Rate limiting

- Apply throttling on auth and order endpoints when exposed publicly.
- Document limits in response headers when implemented.

## Related

- [Architecture overview](../docs/architecture/overview.md)
- [Python coding standard](./coding-standard-python.md)
- [OpenAPI changelog](../docs/api/openapi-changelog.md)
