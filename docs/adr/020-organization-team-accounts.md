# ADR 020: Organization / Team accounts

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-08 |
| Deciders | Solo operator (architecture lock before schema / API) |

## Context

RoamKit today is single-user B2C:

- Identity: `User`
- Money: `billing.Account` OneToOne with `User` ([ADR 010](./010-polygon-usdt-prepaid-credits.md))
- eSIM inventory: `Esim.user` FK

[ADR 010](./010-polygon-usdt-prepaid-credits.md) already keeps all financial FKs Account-centric so Business / Team / Reseller accounts can arrive later. Multi-user Account membership tables were **explicitly out of scope** there. [ADR 012](./012-billing-extensibility-rules.md) forbids new money FKs to `User` for the same reason. [ADR 013](./013-production-launch.md) still lists Business / Team accounts as not built.

The product goal that unlocks this ADR is **fleet / multi-person inventory**: an operator manages eSIMs for several people (later bound to managed devices via UEM and a Flutter status widget). That requires a collaboration boundary, shared inventory, and shared spend — without inventing a second money owner or treating presentation tags (labels) as architecture.

Labels, device binding, UEM integration, and Flutter are **downstream** of this ownership lock. They must not ship before this ADR is Accepted and inventory ownership is Account-scoped.

## Decision

Introduce **Organization** as the collaboration aggregate and **extend** `billing.Account` identity so a team has exactly one dedicated team Account. **Organization is not the financial owner.** `billing.Account` remains the only owner of money and of eSIM inventory.

```text
User (identity)
  │
  ├── personal billing.Account          (kind=personal, 1:1 User — today's B2C)
  │         └── personal eSIM inventory
  │
  └── Membership(role)* ──► Organization   (User may have many org memberships)
                                   │
                                   └── team billing.Account   (kind=organization)
                                             └── team eSIM inventory
                                                       └── DeviceBinding (reserved; later)
                                                                 └── UEM / Flutter (later)
```

| Concern | Owner |
|---------|--------|
| Money (ledger, orders, top-ups, auto-topup spend) | `billing.Account` — [ADR 010](./010-polygon-usdt-prepaid-credits.md) / [ADR 012](./012-billing-extensibility-rules.md) unchanged |
| Team collaboration boundary | `Organization` |
| Who may act | `Membership` + role on `Organization` |
| eSIM inventory | `Account` (personal or team) — **not** bare `User` |
| Presentation tags | Org-scoped Labels — **later**, after inventory ownership |

This ADR **does not** revise ADR 010 money invariants (`CreditService`, ledger SoT, append-only, Account-only financial FKs). It **does** extend today’s concrete schema facts (`Account.user` OneToOne; `Esim.user` as sole inventory owner) in a controlled way.

### Architecture lock

```text
Architecture: LOCKED (Accepted)
Organization ≠ Account (collaboration vs money/inventory)
Account remains sole money + eSIM inventory owner
No Labels / DeviceBinding / UEM / Flutter ahead of the ordered PRs below
No parallel ownership system
```

### Organization ≠ Account

- **Organization** = collaboration aggregate (name, status, memberships, invites).
- **Team Account** = exactly one `billing.Account` with `kind=organization`, linked OneToOne from `Organization.account`.
- **Personal Account** = existing B2C Account with `kind=personal` and required `user` OneToOne.
- Creating an Organization creates a **new** empty team Account. It does **not** convert or merge the creator’s personal Account.
- Organization must **never** hold balance, ledger rows, or act as a spend/credit target.

### Account identity extension

| Kind | `user` | Link | Purpose |
|------|--------|------|---------|
| `personal` | Required OneToOne | `User.billing_account` | Today’s B2C wallet + personal inventory |
| `organization` | Null | `Organization.account` OneToOne | Shared team wallet + team inventory |

All credits and debits for either kind still go only through `CreditService` on that Account.

### Organization + Account lifecycle

| Rule | Normative |
|------|-----------|
| Soft lifecycle | `Organization.status ∈ {active, suspended, archived}` (or equivalent) |
| No casual hard delete | Organization is **not** physically deleted while it has a linked Account / eSIMs |
| Account binding | Team Account remains bound to Organization for the org’s lifetime |
| Archive / suspend preserve audit | Must **not** delete balance, ledger rows, or eSIMs |
| Archived (and typically suspended) | No purchases, no top-ups, no membership mutations |
| Permanent delete | Explicit **later** decision — out of scope for v1 |

This preserves ADR 010 auditability: Account and ledger outlive org operational closure.

### Active Account context (server-side authorization)

A User may have a personal Account **and** Memberships in many Organizations.

```text
current account context
    → personal Account
    or
    → Organization X → team Account X
```

**Normative authorization rules:**

1. API resolves **active Account context** (personal vs a specific Organization’s team Account).
2. Client-supplied `account_id` is **never** sufficient proof of authorization.
3. For team context the server must verify: authenticated User → **active** Membership → Organization → `organization.account`.
4. Spend and inventory mutations execute only against the resolved, authorized Account.
5. Suspended / revoked members have **no** access to the team Account.

### Membership invariants

Roles: `owner` | `admin` | `member` | `viewer`.

| Invariant | Rule |
|-----------|------|
| Uniqueness | At most one Membership per `(User, Organization)` |
| Owner cardinality | After every successful membership mutation: **exactly one** active `owner` |
| Owner removal | Owner cannot be removed without a prior ownership transfer |
| Last owner | Sole / last owner cannot be deactivated or removed |
| Revoked access | Suspended / revoked members cannot use team Account APIs |

#### Permission boundary (normative matrix)

| Action | owner | admin | member | viewer |
|--------|-------|-------|--------|--------|
| View org, members, inventory, usage | yes | yes | yes | yes |
| Buy eSIM / order against team Account | yes | yes | yes | no |
| Top-up / auto-topup policy on team eSIMs | yes | yes | yes | no |
| Invite members | yes | yes | no | no |
| Change roles / remove members | yes | yes (not owner) | no | no |
| Transfer ownership | yes | no | no | no |
| Suspend / archive Organization | yes | no | no | no |
| Assign eSIM assignee (presentation) | yes | yes | yes | no |
| Device bind (when introduced) | yes | yes | no | no |

Exact API shapes are implementation detail; this matrix is the Accept lock for v1.

### Invite security

Invites create Membership only after accept. Normative rules:

- Token is **single-use** and has an **expiry**.
- Revoked or expired invites **cannot** be accepted.
- Accept is **idempotent**.
- Invite grants **no** access before accept.
- Accepting identity must match the invited email/identity, or an identity change must be an explicit confirmed path.
- Invite does **not** merge personal wallets.
- Accept does **not** auto-move personal eSIMs onto the team Account.

### eSIM inventory ownership

Target model:

- `Esim.account` FK → `billing.Account` (personal or organization) is the **inventory owner**.
- Optional `assigned_user` (or equivalent) = who uses the SIM (presentation / ops). **Not** financial or inventory owner.
- Lifecycle, billing, provider sync, and auto-topup remain Account-gated; assignee never bypasses `CreditService`.

#### Migration sketch (implementation after Accept)

1. Add nullable `Esim.account`.
2. Backfill every existing eSIM to the owning user’s **personal** Account.
3. Enforce non-null `Esim.account`.
4. Cut over authorization from `Esim.user` to Account context + membership checks.
5. Drop or demote `Esim.user` only when dual-read is no longer needed (prefer a single ownership rule).

### Personal ↔ team boundary and eSIM transfer

- Leaving a team or revoking membership **never** moves eSIM ownership or wallet balances.
- eSIMs stay on the team Account when a member leaves.
- Ownership change is only an **explicit, audited Account → Account transfer** operation.

```text
Personal Account  --explicit transfer-->  Team Account
```

**Forbidden:**

```text
User leaves Organization     → automatic eSIM transfer
Membership revoked           → eSIM ownership changes
Invite accepted              → personal eSIMs auto-move
```

v1 may ship “purchase into team Account context” before a transfer API exists. Transfer remains a defined future capability under these rules.

### Cross-organization isolation

> Membership in Organization A grants access only to Organization A and its Account.

A User in Org A and Org B must not reach Org B objects via client-supplied `account_id`, `esim_id`, `order_id`, or `device_id` from an Org A context (or from personal context without a valid membership path). This is normative for later API authorization tests.

### Auditability (requirement, not first-PR scope)

Organization-domain mutations must be **auditable** (event/log contract). Minimum set:

- Organization created
- Owner transferred
- Member invited
- Invite accepted
- Member role changed
- Member removed
- Organization suspended / archived
- eSIM Account transfer
- Device binding (when introduced)

Building a full audit product is **not** required in the first schema PR; implementations must not ship these mutations without an auditable trail.

### Permission / API boundary (design)

- Membership and invites under a clear family (e.g. `/api/v1/orgs/…`).
- Inventory and spend remain Account-scoped, authorized via active Account context + membership checks (never trust bare ids).
- Org product surfaces gated by a feature flag until ready.

### Device / UEM extension surface (reserved)

Reserved for later PRs after inventory ownership ships:

- `DeviceBinding` attaches to Account-owned eSIMs via a RoamKit-issued `device_external_id`.
- Managed device profile (e.g. UEM) carries that id; a client reads managed config and calls a status API.
- Status payload can reuse existing eSIM usage cache and auto-topup policy signals (package/location title, unlimited vs remaining data, days to expiry, renew enabled).
- No BlackBerry UEM admin API integration in v1 of this ADR.

### Non-goals

- Org-scoped Labels (after inventory ownership)
- Flutter widget implementation
- BlackBerry UEM admin sync
- B2B reseller portal
- Google OAuth `hd` allowlists ([ADR 015](./015-google-oauth-gis.md) remains unchanged)
- Treating Organization as a balance or inventory owner
- Permanent Organization hard-delete
- Revising ADR 010 `CreditService` / ledger invariants
- Implementing DeviceBinding, transfer API, or audit store in the ADR Accept step itself

### Out of scope for first implementation PRs after Accept

- Labels, Flutter, UEM admin, device schema before the ordered sequence below
- Automatic wallet merge or automatic eSIM move on membership changes

## Implementation sequence after Accept

Normative ordering (one responsibility per PR; no schema before Accept):

```text
1. Organization + Membership + Account.kind + Organization.status schema
2. Team permissions + active Account context + Esim.account inventory migration
3. Device binding
4. UEM status API
5. Flutter widget
6. Org-scoped Labels
```

## Consequences

### Positive

- Unlocks fleet / multi-user inventory without a second money path.
- Preserves ADR 010 Account-centric financial ownership and ledger auditability.
- Clear server-side Account context and cross-org isolation reduce IDOR risk.
- Membership and transfer invariants prevent silent ownership changes.
- Labels and UEM stay presentation / delivery layers, not ownership foundations.

### Negative / trade-offs

- `Account.user` OneToOne must be relaxed for organization Accounts (identity extension).
- `Esim.user` → `Esim.account` migration touches inventory authorization across API/web.
- Active Account context adds API/session complexity versus today’s implicit personal Account.
- Soft org lifecycle means operational cleanup processes are needed later for true purge.

### Binding after Accept

Once Status is **Accepted**:

- Do not redesign Organization vs Account ownership without revising this ADR.
- Do not implement Labels, DeviceBinding, UEM, or Flutter ahead of the sequence above in a way that assumes `User`-owned team inventory.
- Do not authorize spend/inventory from client-supplied `account_id` alone.

## Related

- [ADR 010](./010-polygon-usdt-prepaid-credits.md) — money path; Account financial owner; membership was out of scope
- [ADR 012](./012-billing-extensibility-rules.md) — no User money FKs; extensibility constitution
- [ADR 013](./013-production-launch.md) — Business / Team not built at launch matrix
- [ADR 014](./014-esim-lifecycle-install-telemetry.md) — eSIM lifecycle (unchanged by org collaboration)
- [ADR 019](./019-account-pricing-profiles.md) — per-Account pricing; team Accounts may share profiles later
- [ADR index](../../ADR_INDEX.md)
