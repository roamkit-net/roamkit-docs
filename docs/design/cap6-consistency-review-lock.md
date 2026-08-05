# Cap6 — Cross-Route Consistency Review Design Lock

| Field | Value |
|-------|-------|
| Status | **Accepted** |
| Date | 2026-08 |
| Accepted | 2026-08-05 |
| Capability | Cap6 — Cross-Route Consistency Review (**governance / audit only**) |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), Cap2 Review (`roamkit-web/components/ui/CAP2_REVIEW.md`), Cap1–Cap5 Design Locks (**CLOSED**), [Implementation Plan](./cap6-implementation-plan.md), [design-system-v1-review.md](./design-system-v1-review.md) (Cap6 close artifact — created at close) |
| Pattern | Same process skeleton: **Design Lock → Implementation Plan → execute → close** — but “execute” means **audit**, not product code |
| Prerequisite | Design System v1.0 **IMPLEMENTATION COMPLETE** (Cap1–Cap5 CLOSED) |

This document locks Cap6 decisions so the review does not wander into redesign.  
**Do not reopen locked decisions.** Audit execution detail lives in the Implementation Plan.

Cap1–Cap5 remain **CLOSED**. Cap6 must not reopen them except to document a true **REGRESSION** finding (then fix via a separate bugfix PR — not inside Cap6 redesign).

---

## One-sentence mission

> **Verify the completed Design System, not redesign it.**

Cap6 is a **governance capability**. Cap1–Cap5 were implementation capabilities. That distinction is locked.

---

## Context (locked)

```text
Brand Design System v1.0

Cap1  Brand Tokens                 ✅
Cap2  Shared Primitives            ✅
Cap3  AppShell Variant A           ✅
Cap4  Auth Polish                  ✅
Cap5  Design Consistency Polish    ✅

STATUS: IMPLEMENTATION COMPLETE
READY FOR LONG-TERM EVOLUTION
```

Cap6 does **not** extend the Design System. It **certifies** it.

---

## Guardrails (locked)

| Gate | Lock |
|------|------|
| **Mission** | Verify — do not redesign |
| **Capability type** | Governance / audit — not feature delivery |
| **Zero-code success** | Cap6 **may finish with zero `roamkit-web` changes** |
| **Finding disposition** | PASS / MINOR / MAJOR / REGRESSION only (table below) |
| **No mega-refactor** | Cap6 never absorbs “just one more polish” into itself |
| **Cap1–Cap5** | CLOSED — reopen only for documented REGRESSION |
| **Evidence** | Close requires filled matrices + review doc — not “looks fine” |
| **Close gate** | Architecture / Operations / Product / Audit **PASS** — not “merge” |

---

## Scope (locked)

### In

| Area | Notes |
|------|--------|
| Cross-route consistency audit | Landing, Auth, Plans, eSIM detail, Setup, Deposit, Account |
| Primitive consistency audit | Cap2 + Cap5 chrome (Button, Input, Alert, Card, Tabs, Stepper, Avatar) |
| Token consistency audit | Brand / `--landing-*` / `--auth-*` / `--app-*`; legacy color / radius / spacing / shadow / hardcoded chrome |
| Accessibility consistency audit | Focus, keyboard, contrast expectations already implied by Cap1–Cap5; spot-check consistency — not a full WCAG program |
| Legacy audit | Leftover `text-sky-*` / ad hoc chrome outside Cap5 backlog |
| Documentation review | Design Lock / Plan / status / Cap2 Review alignment with reality |

### Out

| Area | Why |
|------|-----|
| New tokens | Cap1 closed |
| New Cap2 primitives (`ui/Tabs`, etc.) | Cap2 closed |
| New layouts / shells | Cap3–Cap4 closed |
| New capabilities / UX redesign | Not Cap6 |
| Marketing / copy / IA | Product — not Cap6 |
| Landing / Auth / AppShell changes | Only if audit classifies **REGRESSION** → separate bugfix PR |
| Deposit / PlanCard / Dialog / Popover redesign | Cap6 may **note** MINOR/MAJOR; must not implement redesign in Cap6 |

---

## Finding classification (locked)

Every finding must be classified. Cap6 execution must not invent a fifth bucket.

| Severity | Meaning | Action |
|----------|---------|--------|
| **PASS** | Consistent with Design System v1.0 | Nothing |
| **MINOR** | Cosmetic or backlog debt; system still coherent | Docs note and/or **tiny** follow-up PR (optional) |
| **MAJOR** | Material inconsistency; needs deliberate work | **Separate capability** (or named backlog item) — **not** Cap6 scope creep |
| **REGRESSION** | Breaks a Cap1–Cap5 stop rule or production visual contract | **Bugfix PR** (separate from Cap6 audit docs) |

### Anti-patterns (forbidden)

- Folding MAJOR items into Cap6 “while we’re here”
- Opening Cap6.1/6.2 implementation slices for polish without a finding class
- Treating Cap6 as Cap5.5

---

## Zero-code success (locked)

> **Cap6 may finish without any change to `roamkit-web`.**

A green audit with evidence is a complete Cap6 outcome. Follow-up PRs are optional and **outside** Cap6 when they exist (except documenting REGRESSION fixes that already landed).

---

## Audit matrices (locked structure)

Fill during Cap6 execution; empty cells are not a close.

### Routes

| Area | Status | Notes |
|------|--------|-------|
| Landing | ☐ | |
| Auth | ☐ | |
| Plans | ☐ | |
| eSIM detail | ☐ | |
| Setup | ☐ | |
| Deposit | ☐ | |
| Account | ☐ | |

### Primitives

| Primitive | Status | Notes |
|-----------|--------|-------|
| Button | ☐ | |
| Input | ☐ | |
| Alert | ☐ | |
| Card | ☐ | |
| Tabs | ☐ | |
| Stepper | ☐ | |
| Avatar | ☐ | |

### Tokens / legacy

| Audit | Result | Notes |
|-------|--------|-------|
| Legacy colors | ☐ | |
| Legacy radius | ☐ | |
| Legacy spacing | ☐ | |
| Legacy shadows | ☐ | |
| Hardcoded chrome | ☐ | |

### Evidence checklist (close)

```text
Routes audited
Primitives audited
Tokens audited
Accessibility audited
Legacy audited
Findings classified
Review doc published
```

---

## Close artifact (locked)

Cap6 **must** produce:

```text
docs/design/design-system-v1-review.md
```

Not an ADR. Contents (minimum):

- What was audited (matrices)
- What was found (classified)
- What was accepted as-is
- What goes to backlog / follow-up capabilities
- Final assessment (close gate)

This becomes the reference for a future v1.1 / v2.0 — not a redesign brief.

---

## Close gate (locked)

Cap6 closes when **all** read PASS (governance sense — not git merge):

```text
Architecture   PASS
Operations     PASS
Product        PASS
Audit          PASS
```

| Gate | Meaning |
|------|---------|
| Architecture | Cap1–Cap5 stop rules held; no Cap6 redesign of shells/tokens/primitives |
| Operations | Staging evidence / smoke references recorded where relevant; no silent prod drift claimed |
| Product | No Cap6 UX invention; MAJOR/REGRESSION disposition agreed |
| Audit | Matrices filled; findings classified; `design-system-v1-review.md` merged |

“Merged Cap6 docs PR” is **evidence packaging**, not the definition of done.

---

## Process (locked)

```text
Design Lock (this doc) — Draft → Accepted
  → Implementation Plan (audit execution plan) — Draft → Accepted
    → Audit (fill matrices; classify findings)
      → Evidence + design-system-v1-review.md
        → Close Cap6
```

Same Cap1–Cap5 discipline; “implementation” = **running the audit**.

Do **not** open Cap6 Implementation Plan until this Design Lock is **Accepted**.  
Do **not** treat optional follow-up code PRs as Cap6 slices unless they are REGRESSION bugfixes already dispositioned.

Suggested Plan shape (non-binding until Plan Accepted):

```text
Cap6.1  Route + primitive matrix pass
Cap6.2  Token / legacy / a11y pass
Cap6.3  Findings disposition + review doc + close
```

---

## Explicit non-goals

- Design System v2.0
- New visual language
- Expanding Cap2 API
- Reopening Cap3 Variant A / Cap4 Auth / Cap5 chrome decisions without REGRESSION proof

---

## Status

| State | Meaning |
|-------|---------|
| Draft — awaiting acceptance | Closed |
| **Accepted** | Closed — Cap6 executed |
| **Done** | **Current** — Cap6 CLOSED; `design-system-v1-review.md` APPROVED |

**Cap6 CLOSED.** Design System v1.0 COMPLETE. No Cap7.
