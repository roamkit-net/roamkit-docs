# Cap6 — Cross-Route Consistency Review Implementation Plan

| Field | Value |
|-------|-------|
| Status | **Done** — Cap6 CLOSED |
| Date | 2026-08 |
| Accepted | 2026-08-05 |
| Closed | 2026-08-05 |
| Prerequisite | [Design Lock](./cap6-consistency-review-lock.md) = **Accepted** |
| Related | [ADR 016](../adr/016-web-design-tokens.md), [status.md](./status.md), [capability-ledger.md](./capability-ledger.md), close artifact [design-system-v1-review.md](./design-system-v1-review.md) (created at close) |
| Repo | Primarily `roamkit-docs` (audit + review). `roamkit-web` **only** for dispositioned **REGRESSION** bugfix PRs |

**Discipline:** Design Lock is **closed** — no new design decisions.  
**Capability type:** Governance / audit. “Implementation” = **running the audit**, not shipping features.

```text
Design Lock ✅ Accepted
  → Implementation Plan ✅ Accepted (this doc)
    → Cap6.1 → Cap6.2 → Cap6.3 (audit slices)
      → design-system-v1-review.md + Cap6 CLOSE
```

**Cap6.1 may start.** Do not reopen Design Lock decisions.  
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

**Executed:** 2026-08-05 · Staging SHA `fb86a52` (Cap5.4 validation on develop) · Source locks via Cap2/Cap5 suites.

### Routes matrix

| Area | Probe (examples) | Status | Finding class | Notes |
|------|------------------|--------|---------------|-------|
| Landing | `/` — `--landing-*`, no AppShell/AuthShell | ✅ | **PASS** | HTTP 200; `landing-cta` / `--landing-ink`; no AppShell/AuthShell |
| Auth | `/login` (+ sample auth routes) — AuthShell, `--auth-*` | ✅ | **PASS** | HTTP 200; `auth-page-bg` / `auth-shell-panel` / `--auth-chrome`; no AppShell |
| Plans | `/plans` — AppShell, Cap5 tabs `--app-*` | ✅ | **PASS** | HTTP 200; `--app-chrome-text`; tabs `bg-[var(--app-primary)]`; no `bg-sky-700` |
| eSIM detail | `/croatia-esim` (+ `/me/esims/[id]` source) | ✅ | **PASS** | ServiceTab `--app-primary` border; AppShell chrome on detail/me routes |
| Setup | `/me/esims/[id]/setup` chunk | ✅ | **PASS** | Chunk: active/completed/inactive `--app-*`; no `bg-sky-700` |
| Deposit | `/me/deposit` | ✅ | **PASS** | HTTP 200; AppShell + `--app-chrome-text*` in source (client HTML may omit until hydrate) |
| Account | `/me/esims` + TopBar avatar | ✅ | **PASS** | HTTP 200; AppShell chrome; avatar trigger `--app-primary` in shared chunk |

Evidence: staging `/version` `fb86a52…`; no Cap6.1 **REGRESSION**.

### Primitives matrix

| Primitive | Status | Finding class | Notes |
|-----------|--------|---------------|-------|
| Button | ✅ | **PASS** | Cap2; `--app-primary` + `--auth-primary` tones |
| Input | ✅ | **PASS** | Cap2 `InputTone` app \| auth |
| Alert | ✅ | **PASS** | Cap2 `ui/Alert` |
| Card | ✅ | **PASS** | Cap2 `ui/Card` |
| Tabs | ✅ | **PASS** | Cap5.1 PlansStore + LocationDetail ServiceTab |
| Stepper | ✅ | **PASS** | Cap5.2 setup pills only; no `ui/Stepper` |
| Avatar | ✅ | **PASS** | Cap5.3 trigger `--app-*`; menu panel unchanged (slate/white — Cap6.2 legacy note if needed) |

### Cap6.1 done when

- [x] Both matrices filled (no empty Status)
- [x] Every non-PASS row has Finding class + Disposition
- [x] No Cap6 redesign PRs opened

**Cap6.1 result:** all **PASS**. No REGRESSION. Proceed to Cap6.2 (Tokens / A11y / Legacy) — do not invent Cap6 code.

---

## Cap6.2 — Tokens + Accessibility + Legacy

**Executed:** 2026-08-05 · Source on `develop` @ `fb86a52` (+ Cap5.4 locks) · Staging same SHA as Cap6.1.

Audit order inside Cap6.2: **Tokens → Accessibility → Legacy**.

### Tokens

| Check | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| Semantic theme aliases | ✅ | **PASS** | `--landing-*` / `--app-*` / `--auth-*` bind brand (`--color-primary`, etc.) in `globals.css` |
| Cap1–Cap5 surfaces use theme aliases | ✅ | **PASS** | Cap3–Cap5 chrome + Cap2 primary tones; Cap5.4 suite locks boundaries |
| No new Cap6 tokens / primitives | ✅ | **PASS** | None introduced |

### Accessibility (consistency spot-check — not full WCAG)

| Check | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| Focus rings on Cap2 primary + Cap5 chrome | ✅ | **PASS** | `--app-focus-ring` / `--auth-focus-ring` on Button primary, tabs, avatar, ServiceTab |
| Keyboard affordances (tabs / auth / avatar) | ✅ | **PASS** | `role="tablist"` / `aria-*` / avatar `aria-haspopup`; Cap6.1 route probes green |
| Contrast (brand primary) | ✅ | **PASS** | Documented **11.16:1** (`#22d3ee` on dark shell) — Cap3/Cap4 close notes |
| `prefers-reduced-motion` | ✅ | **PASS** | ≥3 reduce blocks in `globals.css` (shell / landing / auth) |

| Check | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| App Button secondary/ghost still `ring-sky-500` | ✅ noted | **MINOR** | Cap2 freeze held; not Cap5 REGRESSION — optional follow-up / Cap2 debt |

### Legacy

| Audit | Result | Finding class | Notes |
|-------|--------|---------------|-------|
| Cap3.5 `SKY700_ALLOWLIST` | ✅ | **PASS** | Empty (Cap5.3/5.4) |
| `bg-sky-700` in non-test source | ✅ | **PASS** | **0** hits — Cap5 CTA/pill debt cleared |
| Legacy colors (`text-sky-700`, `text-cyan-700`, occasional `bg-cyan-500`) | ✅ noted | **MINOR** | Deposit links, PlanCard eyebrows, some auth text links / reset CTA — **out of Cap5**; document for backlog (not Cap6 redesign) |
| Legacy radius | ✅ | **PASS** | No Cap1–Cap5 shell radius regression; ad hoc `rounded-*` on product chrome accepted |
| Legacy spacing | ✅ | **PASS** | App/Auth spacing SoT aliases intact |
| Legacy shadows | ✅ noted | **MINOR** | e.g. UserMenu panel `shadow-lg` — Cap5 avatar **out of scope**; accepted as-is or tiny follow-up |
| Hardcoded chrome on Cap5 surfaces | ✅ | **PASS** | Tabs / stepper / avatar trigger token-bound |

### Cap6.2 disposition summary

| Finding class | Count | Disposition |
|---------------|-------|-------------|
| PASS | majority | Close |
| MINOR | 3 themes | Document in Cap6.3 review; optional tiny follow-ups — **not** Cap6 code |
| MAJOR | 0 | — |
| REGRESSION | 0 | — |

### Cap6.2 done when

- [x] Token + a11y + legacy rows filled
- [x] Findings dispositioned
- [x] REGRESSION (if any) filed as separate bugfix — **none**

**Cap6.2 result:** **PASS** overall (with documented **MINOR** legacy/link debt). No REGRESSION. No `roamkit-web` PR. Proceed to Cap6.3 (review doc + close).

---

## Cap6.3 — Documentation + close

### Documentation review

| Doc | Check | Result |
|-----|-------|--------|
| Cap1–Cap5 Design Locks / Plans / status | Match shipped system | ✅ PASS |
| Cap2 Review / API Freeze | Still held | ✅ PASS |
| Capability ledger | Cap1–Cap6 ✅ | ✅ PASS |
| ADR 016 | Not contradicted by Cap6 | ✅ PASS |

### Close artifact

Published: [design-system-v1-review.md](./design-system-v1-review.md) — **APPROVED**.

### Close gate

| Gate | Result |
|------|--------|
| Architecture | ✅ PASS |
| Operations | ✅ PASS |
| Product | ✅ PASS |
| Audit | ✅ PASS |

### Cap6.3 done when

- [x] Review doc merged
- [x] Cap6 marked **CLOSED** in [status.md](./status.md)
- [x] Ledger Cap6 ✅
- [x] No Cap6 mega-refactor left open

**Cap6 CLOSED.** Design System v1.0 **COMPLETE**. No Cap7.

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
| Draft — awaiting acceptance | Closed |
| Accepted | Closed |
| Cap6.1 / Cap6.2 done | Closed |
| **Done** | **Current** — Cap6 CLOSED; Design System v1.0 COMPLETE |

**Cap6 CLOSED.** No Cap7 Design System capability.
