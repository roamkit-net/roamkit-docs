# ADR 008: Bootstrap Infrastructure as Code via `gh` CLI

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-07 |
| Deciders | Architecture Freeze |

## Context

GitHub organization setup (repos, branch protection, labels, secrets documentation) is error-prone when done manually in the web UI. RoamKit development is **WSL-first** with `gh` and `GH_TOKEN` already configured.

## Decision

All routine GitHub org administration is scripted under `roamkit-infra/bootstrap/github/`:

| Script | Purpose |
|--------|---------|
| `create-org.sh` | Ensure `roamkit` org exists |
| `create-repos.sh` | Create infra, docs, api, web, `.github` |
| `branch-protection.sh` | `main` + `develop` require PR |
| `labels.sh` | Standard issue labels |
| `milestones.sh` | Release milestones |
| `secrets.md` | Secret names and `gh secret set` instructions (no values) |

**Rules:**

- Run scripts from WSL with `GH_TOKEN` (org admin scope).
- Scripts are **idempotent** — safe to re-run.
- No routine use of GitHub web UI for repo creation or branch protection.
- `roamkit-infra` is the **first** repo created and pushed; api/web follow after bootstrap.

Hetzner staging initialization lives in `bootstrap/hetzner/init-staging-stack.sh`.

## Consequences

### Positive

- Reproducible org bootstrap for disaster recovery or new environments.
- Documented secret names in `secrets.md` align with CI workflows.
- Matches Infrastructure as Code mindset used for Docker and deploy scripts.

### Negative

- Scripts must be maintained when GitHub API changes.
- Some org settings (billing, SSO) may still require web console.

### Execution order (Faza -1c)

```bash
cd roamkit-infra/bootstrap/github
export GH_TOKEN=...
./create-org.sh
./create-repos.sh
./branch-protection.sh
./labels.sh
```

Then push `roamkit-infra` and `roamkit-docs` before initializing application repos.

## Related

- [ADR 001](./001-repo-split.md)
- `roamkit-infra/bootstrap/github/secrets.md`
