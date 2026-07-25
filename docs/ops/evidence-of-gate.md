# Evidence of Gate

Lightweight **audit trail** when a Launch Gate closes (or a named Gate milestone
completes). Complements the [Release Manifest](./release-manifest.md) — Manifest
is what shipped; Evidence is **who approved which Gate, against which commits/CI**.

Not day-to-day runbook noise. Months later this answers:

- Which commit / PR set closed Gate C (or a Gate C slice)?
- Was CI fully green?
- Who gave GO, and when?

## Where to store

```text
docs/ops/releases/<release>/
  manifest.md                 # filled Release Manifest (at Gate D)
  evidence/
    gate-a.md
    gate-b.md
    gate-c.md                 # full Gate C GO
    gate-c-api.md             # optional milestone before full C GO
    gate-d.md
```

One file per closed Gate (or milestone). Keep each file short.

## Template

```markdown
# Evidence — Gate X ([optional milestone name])

| Field | Value |
|-------|-------|
| Release | X.Y.Z |
| Gate | A \| B \| C \| D |
| Milestone | (optional, e.g. API slice) |
| Verdict | GO \| GO WITH CONDITIONS |
| Closed at (UTC) | |
| GO by (role / name) | |
| Merge commit SHA | |
| CI run (green) | URL |
| Primary PR(s) | URL(s) |
| Related PRs | URL(s) |

### Notes

- (one or two lines)
```

## Rules

- File evidence when the Gate (or milestone) is **declared closed**, not at every PR.
- Prefer squash-merge commit on the target branch (`develop` / `main`) as the SHA.
- Link the CI run that was green on the merge PR (or promotion PR).
- Full Gate C GO still requires host checks in [gate-c-exit.md](./gate-c-exit.md); an API-slice evidence pack does **not** close Gate C by itself.

## Related

- [Launch Gates](./launch-gates.md)
- [Release Manifest](./release-manifest.md)
- [releases/](./releases/)
