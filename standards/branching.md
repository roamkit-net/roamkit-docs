# Branching Standard

Git workflow for all RoamKit repositories.

## Branches

| Branch | Purpose | Deploy |
|--------|---------|--------|
| `main` | Production release line | Auto-deploy to production when stack + CI exist ([ADR 013](../docs/adr/013-production-launch.md)) |
| `develop` | Integration branch for staging | Auto-deploy to staging on merge |
| `feature/<name>` | Short-lived work branches | None |

## Rules

1. **Never push directly to `main` or `develop`** — branch protection enforces PRs (see `roamkit-infra/bootstrap/github/branch-protection.sh`).
2. Branch from `develop` for features; hotfixes from `main` only when explicitly agreed.
3. Delete `feature/*` branches after merge.
4. Keep PRs small and focused; one logical change per PR when possible.
5. Promote releases with PR `develop` → `main` after staging verification.
   Use the [promote parity checklist](../docs/ops/promote-parity-checklist.md)
   (content diff + bake/flag matrix — do not trust commit-ahead counts).

## PR flow

```
feature/<name>
    → PR → develop   (CI: lint, test, security, build)
    → merge          (deploy staging)
    → later: PR develop → main (release → deploy production)
```

## Commit messages

- Use imperative mood: `Add package sync service`, `Fix health ready check`.
- Reference issue numbers when applicable: `Fix #42: handle Airalo 429`.
- No secrets, tokens, or `.env` values in commits.

## Repository parity

Same branch strategy applies to:

- `roamkit-api`
- `roamkit-web`
- `roamkit-infra`
- `roamkit-docs`
- `.github`

## Coordinating cross-repo changes

When API and web change together:

1. Merge API change to `develop` first (or ensure backward-compatible API), including the committed `openapi/openapi.yaml`.
2. Merge web change that consumes the new contract: vendor that YAML and regenerate `src/api/generated/*` (Wave 2; see [api-versioning.md](./api-versioning.md)).
3. Verify on staging before promoting `main`.

## Related

- [ADR 013](../docs/adr/013-production-launch.md) — production launch (current deploy matrix)
- [ADR 007](../docs/adr/007-staging-only-until-launch.md) — superseded staging-only gate
- [Production go-live checklist](../docs/ops/production-go-live-checklist.md)
- [Promote parity checklist](../docs/ops/promote-parity-checklist.md)
- [CI standard](./ci-standard.md)
