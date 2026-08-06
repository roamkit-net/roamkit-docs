# Cross-RFC Consistency Review

**Status:** Complete · 2026-08  
**Type:** Docs-only review (no new architectural decisions)  
**Scope:** Vision → Conversion Boundary → RFC 003 → 004 → 005 → 006 (Draft)  
**Index:** [Wallet Architecture Index](../wallet-architecture-index.md)

---

## Review question

> Do the Wallet documents disagree on vocabulary, authority, or dependencies?

---

## 1. Vocabulary Matrix

| Termin | Defined in (SoT for meaning) | Used in | Consistent? |
|--------|------------------------------|---------|---------------|
| WalletIdentity | RFC 003 | Vision, 004, 005, 006 | ✅ |
| WalletAddress | RFC 003 | Vision, Conversion Boundary, 004, 005, 006, Contract appendix | ✅ |
| Deposit | RFC 003 | 005, 006 | ✅ |
| FundingSource | RFC 003 | 005 | ✅ |
| Funding Provider | RFC 005 (Vision names examples) | 006, Freeze, Sandbox | ✅ |
| Index Registry | RFC 004 | 005, 006 | ✅ |
| Platform Wallet Infrastructure | RFC 004 | Freeze, Track 1 | ✅ |
| Credit Conversion | Conversion Boundary (+ Vision) | 005, 006 | ✅ |
| Observation / Confirmation | RFC 006 | 005 (forward ref) | ✅ |
| Credits / Ledger | ADR 010 (+ Billing invariants in 003) | All | ✅ |
| Platform Deposit Key | Vision / Conversion Boundary (legacy term) | Superseded by **Platform Wallet Infrastructure** (RFC 004) | ⚠️ terminology debt |
| Deposit Detection | Vision / Sandbox track name / RFC 003 non-goal wording | Superseded name: **Observation & Confirmation** (RFC 006) | ⚠️ terminology debt |

**Verdict:** Core terms align. Legacy Vision/Sandbox phrases (“Platform Deposit Key”, “Deposit Detection”) are **stale labels**, not competing definitions. Cleanup only via freeze process (this review is evidence); do not reopen RFC 003–005 content for renames alone unless a freeze amendment is opened.

---

## 2. Authority Matrix

| Concept | Source of Truth (who owns the definition) |
|---------|-------------------------------------------|
| WalletIdentity / WalletAddress / Deposit model | **RFC 003** |
| Platform state (Seed, Index Registry, allocation) | **RFC 004** |
| Funding Provider adapter contract | **RFC 005** (+ Contract appendix for logical ops) |
| Deposit Confirmation policy (when Confirmed) | **RFC 006** (Draft) |
| Credits balance + ledger mutations | **ADR 010** / Billing (`CreditService`) |
| Why chain stops at Credits | **Conversion Boundary** |
| Long-term direction (non-normative) | **Vision** |
| Change control for frozen set | **Architecture Freeze** |

**Verdict:** No double ownership of the same concept across RFC 003–006. Vision must not override RFC definitions for implementation.

---

## 3. Dependency Graph

Architectural (not package) dependencies:

```text
Vision (parent, non-normative)
  └── Conversion Boundary
        └── RFC 003 Domain
              └── RFC 004 Infrastructure
                    └── RFC 005 Funding Interface
                          └── RFC 006 Observation & Confirmation (Draft)
```

| Edge | Direction | Cycle? |
|------|-----------|--------|
| 004 → 003 | uses domain entities | No |
| 005 → 003, 004 | destination = WalletAddress from 004 | No |
| 006 → 003, 004, 005 | confirmation on Deposit / address / ignores provider as SoT | No |
| 003 ↛ 005/006 for definitions | 003 only forward-references | No |

**Verdict:** ✅ Acyclic. Later RFCs depend on earlier ones; earlier frozen RFCs do not depend on later ones for meaning.

---

## 4. Missing Concepts

| Concept | Covered? | Where |
|---------|----------|--------|
| WalletIdentity | ✅ | RFC 003 |
| WalletAddress | ✅ | RFC 003 |
| Allocation / Index Registry | ✅ | RFC 004 |
| Funding (adapter interface) | ✅ | RFC 005 |
| Confirmation (Confirmed → convert) | ✅ Draft | RFC 006 |
| Credits / ledger | ✅ | ADR 010 |
| Observation adapters (RPC / indexer / explorer) | Future detail | RFC 006 lists classes; vendor choice → research/ADR |
| Chain adapters | Future | Vision intent; not a dedicated RFC yet |
| Sweep / treasury / gas | Open (ADR) | RFC 004 open questions |
| ADR 010 → per-user address cutover | Future Wallet ADR | Explicitly deferred |
| Withdraw / multi-chain / user spend keys | Out of scope | Freeze + RFCs |

**Verdict:** No hole in the **structure** chain. Remaining gaps are **behavior parameters** (RFC 006 open questions) and **implementation ADRs** — expected.

---

## 5. ADR Ownership (forward map)

| Area | Normative today | Future |
|------|-----------------|--------|
| Billing / Credits / current deposits | **ADR 010** (Accepted) | May extend via Billing ADRs |
| Wallet domain + infra + funding + observation | RFCs 003–006 (proposals) | **Future Wallet ADR(s)** at cutover |
| Funding vendor selection | — | Future ADR or ops decision under RFC 005 |
| Observation adapter topology + confirm depth | — | Future ADR under RFC 006 |
| Design tokens / web | ADR 016 etc. | Unrelated to Wallet money path |

**Verdict:** Clear split — do not put Wallet cutover rules into ADR 010 silently; do not treat RFCs as production-normative until a Wallet ADR is Accepted.

---

## 6. RFC 006 scope lock (consistency with review intent)

RFC 006 Draft already matches the intended lock:

| Must answer | In Draft? |
|-------------|-----------|
| What Deposit confirmation means | ✅ |
| When Credit Conversion may start | ✅ |
| Idempotency | ✅ |
| Chain reorg behavior | ✅ |
| Duplicate observation | ✅ (idempotency key) |

| Must not answer (adapters) | Kept out? |
|----------------------------|-----------|
| RPC / Etherscan / Alchemy / Infura / MEXC / Indexer as SoT | ✅ listed as adapters only |

**Note:** RFC 006 already exists as Draft on `develop` (PR #86). This review does **not** reopen writing it from scratch. Next step for 006 is Architecture Review of that Draft when ready — not another Vision/RFC 003–005 pass.

---

## Findings (non-blocking)

| ID | Finding | Action |
|----|---------|--------|
| F1 | Vision/Conversion Boundary still say “Platform Deposit Key” / old RFC 004 title | Defer rename to freeze amendment when convenient; meaning owned by RFC 004 |
| F2 | Vision/Sandbox still say “Deposit Detection” | Prefer “Observation & Confirmation” in new text; optional freeze cleanup |
| F3 | RFC 003 Open Questions #1 and #7 largely answered by Track 1 / RFC 004 | Defer checklist update under freeze; do not treat as open blockers |
| F4 | Event name `DepositConfirmed` (Vision) vs state `Confirmed` (RFC 003/006) | Equivalent; ADR may standardize event names |

No circular dependency. No competing SoT for Credits vs WalletIdentity.

---

## Exit

| Field | Content |
|-------|---------|
| Question | Are Vision→RFC 006 docs consistent? |
| Evidence | Vocabulary / Authority / Dependency / Missing / ADR matrices above |
| Decision | **PASS** — no blocking divergence |
| Impact | Safe to keep Freeze; continue RFC 006 as Draft toward its own Architecture Review |
| Next Action | Use [Architecture Index](../wallet-architecture-index.md); optional freeze terminology cleanup PR; RFC 006 Architecture Review when product prioritizes confirmation behavior |
| Knowledge capture | Structure chain is complete; remaining work is confirmation parameters + Wallet ADR cutover, not more domain RFCs |

---

## Related

- [Wallet Architecture Index](../wallet-architecture-index.md)
- [Wallet Architecture Freeze](../wallet-architecture-freeze.md)
- [RFC 006](../../rfcs/006-deposit-observation-confirmation.md)
