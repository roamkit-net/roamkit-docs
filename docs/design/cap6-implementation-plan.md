# Cap6 — Cross-Route Consistency Review Implementation Plan

| Field | Value |
|-------|-------|
| Status | **Draft — awaiting acceptance** |
| Date | 2026-08 |
| Prerequisite | [Design Lock](./cap6-consistency-review-lock.md) = **Accepted** |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), [capability-ledger.md](./capability-ledger.md), close artifact [design-system-v1-review.md](./design-system-v1-review.md) (created at close) |
| Repo | Primarily `roamkit-docs` (audit + review). `roamkit-web` **only** for dispositioned **REGRESSION** bugfix PRs |

**Discipline:** Design Lock is **closed** — no new design decisions.  
**Capability type:** Governance / audit. “Implementation” = **running the audit**, not shipping features.

```text
Design Lock ✅ Accepted
  → Implementation Plan (this doc) — Draft → Accepted
    → Cap6.1 → Cap6.2 → Cap6.3 (audit slices)
      → design-system-v1-review.md + Cap6 CLOSE
```

**Do not open Cap6 audit execution until this Plan is Accepted.**  
**Do not open polish/refactor `roamkit-web` PRs under Cap6** — only REGRESSION bugfixes (separate PRs).

---

## Whole-capability acceptance

Cap6 is done when:

- Audit order completed (Routes → … → Documentation)
- All Design Lock matrices filled
- Every finding has a disposition (table below)
- Close artifact `docs/design/design-system-v1-review.md` merged
- Close gate: Architecture / Operations / Product / Audit **PASS**
- Cap1–Cap5 remain CLOSED (except documented REGRESSION fixes outside Cap6)

Zero `roamkit-web` change is a **valid** Cap6 success.

---

## Disposition rule (locked for execution)

Every finding must land in exactly one row. No debate mid-audit.

| Finding | Disposition |
|---------|-------------|
| **PASS** | Close — no action |
| **MINOR** | Document in review doc and/or open a **tiny** follow-up PR (optional; not a Cap6 slice) |
| **MAJOR** | New capability or named backlog item — **not** Cap6 |
| **REGRESSION** | Separate **bugfix** PR — not Cap6 redesign |

---

## Audit order (strict)

Always execute in this sequence. Do not skip ahead to “interesting” legacy finds.

```text
Routes
  ↓
Primitives
  ↓
Tokens
  ↓
Accessibility
  ↓
Legacy
  ↓
Documentation
```

---

## Work order (slices)

```text
Cap6.1  Routes + Primitives matrices
  ↓
Cap6.2  Tokens + Accessibility + Legacy matrices
  ↓
Cap6.3  Documentation review + findings disposition + design-system-v1-review.md + close
```

| Slice | Goal | Must not |
|-------|------|----------|
| Cap6.1 | Fill route + primitive matrices; classify finds | Redesign UI; Cap6 code polish |
| Cap6.2 | Fill token / a11y / legacy matrices; classify finds | Absorb MAJOR into Cap6 |
| Cap6.3 | Docs review; write review doc; Cap6 CLOSE | New design decisions |

One docs PR per slice is enough unless a REGRESSION needs a separate web fix PR.

---

## Cap6.1 — Routes + Primitives

### Routes matrix

| Area | Probe (examples) | Status | Finding class | Notes |
|------|------------------|--------|---------------|-------|
| Landing | `/` — `--landing-*`, no AppShell/AuthShell | ☐ | | |
| Auth | `/login` (+ sample auth routes) — AuthShell, `--auth-*` | ☐ | | |
| Plans | `/plans` — AppShell, Cap5 tabs `--app-*` | ☐ | | |
| eSIM detail | `/me/esims/[id]` or location detail | ☐ | | |
| Setup | `/me/esims/[id]/setup` — Cap5 stepper | ☐ | | |
| Deposit | `/me/deposit` — AppShell; note debt, don’t redesign | ☐ | | |
| Account | `/me/esims` + TopBar avatar | ☐ | | |

Evidence: staging SHA (if used), HTTP 200, token/class markers, short notes.

### Primitives matrix

| Primitive | Status | Finding class | Notes |
|-----------|--------|---------------|-------|
| Button | ☐ | | Cap2 + app/auth tones |
| Input | ☐ | | |
| Alert | ☐ | | |
| Card | ☐ | | |
| Tabs | ☐ | | Cap5.1 |
| Stepper | ☐ | | Cap5.2 |
| Avatar | ☐ | | Cap5.3 trigger only |

### Cap6.1 done when

- [ ] Both matrices filled (no empty Status)
- [ ] Every non-PASS row has Finding class + Disposition
- [ ] No Cap6 redesign PRs opened

---

## Cap6.2 — Tokens + Accessibility + Legacy

### Tokens / legacy matrix

| Audit | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| Legacy colors | ☐ | | e.g. leftover `text-sky-*` (Cap6 note ≠ Cap5 pill allowlist) |
| Legacy radius | ☐ | | |
| Legacy spacing | ☐ | | |
| Legacy shadows | ☐ | | |
| Hardcoded chrome | ☐ | | Bypass of `--app-*` / `--auth-*` / `--landing-*` on Cap1–Cap5 surfaces |

### Accessibility (consistency spot-check)

| Check | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| Focus rings on Cap2 + Cap5 chrome | ☐ | | |
| Keyboard on tabs / auth / avatar trigger | ☐ | | |
| Contrast expectations (brand primary / elevated) | ☐ | | |
| `prefers-reduced-motion` escapes still present | ☐ | | |

### Cap6.2 done when

- [ ] Token + a11y + legacy rows filled
- [ ] Findings dispositioned
- [ ] REGRESSION (if any) filed as separate bugfix — not Cap6 polish

---

## Cap6.3 — Documentation + close

### Documentation review

| Doc | Check | Result |
|-----|-------|--------|
| Cap1–Cap5 Design Locks / Plans / status | Match shipped system | ☐ |
| Cap2 Review / API Freeze | Still held | ☐ |
| Capability ledger | Cap1–Cap5 ✅; Cap6 close pending | ☐ |
| ADR 016 | Not contradicted by Cap6 | ☐ |

### Close artifact

Create and merge:

```text
docs/design/design-system-v1-review.md
```

Minimum sections:

1. Scope & method (audit order)
2. Matrices (final)
3. Findings (classified + disposition)
4. Accepted as-is
5. Backlog / follow-up capabilities
6. Close gate assessment (Architecture / Operations / Product / Audit)

### Close gate

| Gate | Result |
|------|--------|
| Architecture | ☐ PASS |
| Operations | ☐ PASS |
| Product | ☐ PASS |
| Audit | ☐ PASS |

### Cap6.3 done when

- [ ] Review doc merged
- [ ] Cap6 marked **CLOSED** in [status.md](./status.md)
- [ ] Ledger Cap6 ✅
- [ ] No Cap6 mega-refactor left open

---

## Evidence conventions

Prefer:

- Staging `/version` SHA when claiming live UI
- Source locks already in Cap3.5 / Cap4.3 / Cap5.4 test suites (cite; don’t reinvent)
- Short notes per matrix cell — not essays

Avoid:

- “Looks good” without matrix fill
- Mixing Cap6 docs with unrelated ADR / product work in the same PR

---

## Explicit non-goals (every slice)

- New tokens / primitives / layouts / UX
- Cap5.5 polish under Cap6 name
- Absorbing MAJOR into Cap6
- Design System v2.0 planning inside Cap6 close (backlog pointer only)

---

## PR hygiene

- Branch from `develop`; PR into `develop`
- Conventional Commits: `docs(design): Cap6.N …`
- Prefer docs-only PRs
- Web: `fix(ui): …` only for dispositioned REGRESSION
- Squash-merge when CI green

---

## Status

| State | Meaning |
|-------|---------|
| **Draft — awaiting acceptance** | **Current** |
| Accepted | Cap6.1 audit may start |
| Done | Cap6.3 closed Cap6 |

**Next after Accept:** Cap6.1 Routes + Primitives matrix pass (docs evidence; no web polish).
