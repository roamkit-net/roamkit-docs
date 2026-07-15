# ADR 001: Multi-repo split

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

RoamKit spans a Django API, Next.js web app, infrastructure, and shared documentation. A monorepo would couple unrelated release cycles and blur ownership of deploy vs application code.

## Decision

Use **four application/infrastructure repos** plus an org-level `.github` repo:

| Repo | Contents |
|------|----------|
| `roamkit-docs` | ADR, RFC, architecture, standards |
| `roamkit-infra` | Bootstrap, compose, nginx, CI templates, deploy scripts |
| `roamkit-api` | Django + DRF + Celery |
| `roamkit-web` | Next.js 15 App Router |
| `.github` | Reusable workflows, PR template, CODEOWNERS |

**Not included yet:** `roamkit-mobile` (Flutter). Open a new repo when mobile development starts.

Documentation is centralized in `roamkit-docs` — not duplicated in each repo README beyond pointers.

## Consequences

### Positive

- Independent versioning and CI per service.
- Infra/bootstrap can exist before application code (Faza -1a before Faza 0).
- Clear boundary: `roamkit-infra` owns deploy; api/web own runtime code.

### Negative

- Cross-repo changes (API contract + web client) need coordinated PRs.
- Shared types are not in a single package; web uses OpenAPI or hand-maintained client types.

### Mitigations

- API versioning standard and staging as integration environment.
- RFC/ADR process for contract changes.
- Org reusable workflows in `.github` reduce CI drift.

## Related

- [ADR 008](./008-bootstrap-iac-gh-cli.md) — repo creation via `gh`
- [Directory structure](../architecture/directory-structure.md)
