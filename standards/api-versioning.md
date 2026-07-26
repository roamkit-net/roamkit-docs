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

Staging `/api/schema/`, `/api/docs/`, and `/api/redoc/` are for **human browsing** only. They are not inputs to type generation, CI, or release.

### Conventions

- **operationId:** explicit snake_case `{domain}_{action}` (e.g. `billing_balance`, `orders_create`). Never rely on spectacular auto-suffixes (`list_1`).
- **operationId freeze:** once published, an `operationId` is part of the public contract and must not change without a breaking-change rationale (generators often use it as the client function name).
- **Tags (order):** Authentication, Billing, Orders, Catalog, eSIM, Users.
- **Security:** HTTP Bearer JWT (`bearerAuth`) on authenticated operations; public ops clear security.
- **CI:** generate + validate + drift check; Spectral errors fail, warnings report-only; architecture tests require 100% `/api/v1/` path coverage.

### Contract ownership

Backend is the **contract owner**. Frontend is the **contract consumer**.

```text
Backend PR
    ↓
update OpenAPI annotations
    ↓
generate openapi.yaml
    ↓
commit openapi.yaml
    ↓
merge

Frontend PR
    ↓
use committed openapi.yaml
    ↓
generate src/api/generated/*
    ↓
commit generated client
    ↓
merge
```

- Never generate types from a live deploy (staging or production).
- Never invent a parallel contract outside `openapi.yaml`.

### Artifact ownership

| Artifact | Owner |
|----------|--------|
| Django views/serializers | Backend |
| `openapi/openapi.yaml` | Backend |
| `src/api/generated/*` | Frontend |
| Typed wrapper | Frontend |
| Business API helpers | Frontend domain |

### Symmetric no-hand-edit

Generated artifacts are never edited by hand on either side:

- **Backend:** `roamkit-api/openapi/openapi.yaml` is never edited manually. The only allowed change path is regenerate from Django code via `./scripts/generate_openapi.sh` (enforced by API CI drift check).
- **Frontend:** `roamkit-web/src/api/generated/*` is never edited manually. Wave 2 CI must run generate then `git diff --exit-code` on the generated output; any diff fails the job.

### Sole source for type generation

`openapi.yaml` is the **only** allowed source for type generation.

Generating from `/api/schema/` is allowed only for local exploration or manual checks — **never** in CI and **never** in the official release process.

### Cross-repo artifact handoff

Same mirror pattern as the billing-config JSON Schema:

- Frontend keeps a **vendored copy** of the contract YAML in-repo (exact path set in Wave 2 PR1).
- Frontend PRs that bump types copy from a known `roamkit-api` commit or tag of `openapi/openapi.yaml` — not from HTTP schema endpoints.
- Local workspace checkouts may read `../roamkit-api/openapi/openapi.yaml` for convenience; CI and release must use the **committed vendored file** inside `roamkit-web`.

### Long-term contract policies

| Policy | Rule |
|--------|------|
| Pinned generator toolchain | Pin `openapi-typescript`, Node, and the package manager via lockfile so the same YAML always yields the same output. |
| Import boundary | `src/api/generated/*` is **frontend-only**. Domain clients may import it; shared utilities and backend helpers must not. |
| Thin wrapper | Typed fetch wrapper may do auth, retry, timeout, error mapping, and serialization only. **No business logic** — that stays in domain modules (`lib/billing`, `lib/orders`, etc.). |
| Deprecation | Lifecycle: `NEW` → `deprecated` → `removed`. Mark deprecated first; remove only in the next **major** API release. |
| Enum | Existing enum values must not change meaning; new values may be added; renaming a value is a **breaking** change. |
| Nullable / required | `required` / `optional` / `nullable` must match across serializer, OpenAPI, and generated TypeScript. No hand exceptions. |

After these rules are locked, do not add more OpenAPI Wave 2 policies unless a real incident or contract break requires it.

### Wave 2 (frontend) — backlog

Not part of C10. Delivery sequence (one responsibility per PR):

1. **PR1** — `openapi-typescript` + generate into `src/api/generated/` + vendored YAML path + drift CI (`generate` then `git diff --exit-code`) + sole-source scripts.
2. **PR2** — typed API client wrapper around existing `fetchApi` (thin: auth / retry / timeout / error mapping / serialization only).
3. **PR3** — migrate billing, orders, and auth call sites to generated types / wrapper.
4. **PR4** — remove hand-written types where generated types fully replace them.

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
