# C10 OpenAPI evidence

| Field | Value |
|-------|--------|
| Gate | C10 OpenAPI |
| Verdict | **Closed** (follow-up after Gate C GO) |
| Date | 2026-07-26 |
| API PRs | roamkit-api #23 (infra), #24 (annotations), #25 (quality gate) |
| Docs PR | this change set |

## Deliverables

| Deliverable | Evidence |
|-------------|----------|
| Generator | pinned `drf-spectacular==0.28.0` |
| Artifact | `roamkit-api/openapi/openapi.yaml` |
| Staging URLs | `/api/schema/`, `/api/docs/`, `/api/redoc/` on `api.staging.roamkit.net` (after `develop` deploy) |
| CI | `openapi` job in `ci-python.yml`: generate, drift, Spectral (`@stoplight/spectral-cli@6.16.2`), artifact upload |
| Architecture tests | `tests/test_openapi_architecture.py` — 100% `/api/v1/` coverage + security docs |

## Notes

- Gate C already GO with C10 deferred; this evidence closes the deferred item without re-running Gate C.
- Production publish of docs UI follows freeze / Hypercare policy (staging-first).
- Frontend typegen remains Wave 2 backlog — see [api-versioning.md](../../../standards/api-versioning.md).
