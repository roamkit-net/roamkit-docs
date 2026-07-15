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

Not under `/api/v1/` — used by infrastructure probes.

## OpenAPI

- Generate schema with drf-spectacular or equivalent when API stabilizes.
- Publish schema URL on staging for web client type generation (optional).

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
