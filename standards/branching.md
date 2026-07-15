# Branching Standard

Git workflow for all RoamKit repositories.

## Branches

| Branch | Purpose | Deploy |
|--------|---------|--------|
| `main` | Stable, release-ready code | CI only — no auto-deploy until production launch |
| `develop` | Integration branch for staging | Auto-deploy to staging on merge |
| `feature/<name>` | Short-lived work branches | None |

## Rules

1. **Never push directly to `main` or `develop`** — branch protection enforces PRs (see `roamkit-infra/bootstrap/github/branch-protection.sh`).
2. Branch from `develop` for features; hotfixes from `main` only when explicitly agreed.
3. Delete `feature/*` branches after merge.
4. Keep PRs small and focused; one logical change per PR when possible.

## PR flow

```
feature/add-package-sync
    → PR → develop   (CI: lint, test, security, build)
    → merge          (deploy staging)
    → later: PR develop → main (release)
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

1. Merge API change to `develop` first (or ensure backward-compatible API).
2. Merge web change that consumes the new contract.
3. Verify on staging before promoting `main`.

## Related

- [ADR 007](../docs/adr/007-staging-only-until-launch.md)
- [CI standard](./ci-standard.md)
