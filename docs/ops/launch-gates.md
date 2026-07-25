# Launch Gates

Shared language for every RoamKit release — not only billing. Architecture stays in
ADRs; this document defines **when** we may advance and **who** says GO.

**Verdict vocabulary:** **GO** · **GO WITH CONDITIONS** · **NO-GO**

| Gate | Meaning | Owner | Status (first production) |
|------|---------|-------|---------------------------|
| A | Architecture Ready | Architecture | ✅ closed |
| B | Infrastructure Ready | Infrastructure | ✅ closed (PR1) |
| C | Production Ready | Engineering | ⏳ open |
| D | Customer Traffic | Operations / Release | ⏳ blocked on Gate C GO |

Map to Faza 4 PRs: A ≈ PR0/PR0.5 · B ≈ PR1 · C ≈ app readiness + E2E + observability · D ≈ cutover.

## Release Criteria (Entry / Exit)

| Gate | Entry Criteria | Exit Criteria |
|------|----------------|---------------|
| A | ADR set proposed / review open | ADR 010–013 Accepted (+ relevant supersedes, e.g. 007) |
| B | Gate A GO; PR0.5 not NO-GO | PR1 production platform merged; compose/bootstrap ready |
| C | PR1 merged (Gate B GO) | All relevant readiness conditions C* closed; Engineering **GO** |
| D | Gate C = GO; Production Freeze active | Traffic stable through **Hypercare** (24h); Release Manifest filled; freeze closed |

Do not declare a Gate closed on subjective “we feel ready.” Use the table.
When a Gate (or named milestone) closes, file an [Evidence of Gate](./evidence-of-gate.md) pack under `releases/<version>/evidence/` and append a row to the [Release Decision Log](./release-decision-log.md).

## Gate details

### Gate A — Architecture Ready

- **In:** design discussions, ADR drafts.
- **Out:** Accepted money-path and launch ADRs; no alternative implementations.
- **Evidence:** [ADR_INDEX](../../ADR_INDEX.md), [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md), [ADR 012](../adr/012-billing-extensibility-rules.md), [ADR 013](../adr/013-production-launch.md).

### Gate B — Infrastructure Ready

- **In:** Gate A GO; readiness review not NO-GO.
- **Out:** `/opt/stacks/roamkit-production/` compose, isolation rules, secrets docs.
- **Evidence:** infra PR1; runbook `roamkit-infra/bootstrap/hetzner/PRODUCTION_PLAN.md`.

### Gate C — Production Ready

- **In:** PR1 merged.
- **Out:** C1–C9 closed (C10 N/A/deferred per readiness review). Includes `/version`, production settings, Migration Ready, billing E2E, observability, incident runbook, Release Manifest **draft**.
- **Evidence:** [production-readiness-review.md](./production-readiness-review.md) conditions; [capability-status.md](./capability-status.md).
- **C8 note:** backup-before-migrate + documented restore are required. Full [Disaster Day](./disaster-day.md) simulation is **not** a cutover hard blocker — run after stable production (annual).

### Gate D — Customer Traffic

- **In:** Gate C GO + [Production Freeze](#production-freeze) active.
- **Out:** Hypercare complete without unresolved P0/P1; filled [Release Manifest](./release-manifest.md); freeze lifted.
- **Runbook:** [gate-d-cutover.md](./gate-d-cutover.md).

## Gate D time-box (defaults)

Defaults for the first production release. Override only in the Release Manifest for that release.

| Window | Default |
|--------|---------|
| Deployment window | **09:00–11:00 UTC** (weekday) |
| Rollback window | **30 minutes** after cutover |
| Hypercare | **24 hours** heightened monitoring |

**Gate D Exit** = traffic stable through Hypercare (no unresolved P0/P1; rollback not required, or rollback succeeded and cutover deferred).

## Gate C focus (before host execution)

While Gate C is **open** (API slice done; host verification in progress):

- **No** parallel work on other release candidates or non-essential features.
- Release process focus stays on closing Gate C, then Gate D.
- **Allowed:** critical bugfixes, Gate C evidence packs / host ops, release/hotfix only.
- **Forbidden:** voucher PRs, refactors, new feature merges that move the goalposts.

This is the early form of [Production Freeze](#production-freeze) — apply it for the
final host checks, not only on cutover day.

## Production Freeze

From the start of the Gate D **deployment window** until the end of the **rollback window**:

- **No** new feature merges into `develop`.
- **Allowed:** bugfix, release/hotfix PRs, cutover ops only.
- **Forbidden:** voucher work, refactors, non-essential features.

Freeze lifts after rollback window closes successfully (Hypercare may continue with normal merge discipline restored unless operator extends freeze).

## Related

- [Operations Handbook index](./README.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md) · [ADR 012](../adr/012-billing-extensibility-rules.md) · [ADR 013](../adr/013-production-launch.md)
- [Production readiness review](./production-readiness-review.md)
- [Capability / Maturity Matrix](./capability-status.md)
- [Go-live checklist](./production-go-live-checklist.md)
