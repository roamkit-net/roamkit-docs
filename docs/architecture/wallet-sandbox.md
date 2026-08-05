# Wallet Sandbox

| Field | Value |
|-------|-------|
| Status | Draft / research framework (non-normative) |
| Date | 2026-08 |
| Parents | [Wallet Vision](./roamkit-wallet-platform-vision.md), [Conversion Boundary](./wallet-conversion-boundary.md), [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) |

> The Sandbox converts **unknowns into decisions**. It is not a place to grow production code or to bypass ADRs.

After this document is accepted as the framework, treat **Vision**, **RFC 003**, **Conversion Boundary**, **RFC 004**, and **this Sandbox framework** as **frozen**. See [Wallet Architecture Freeze](./wallet-architecture-freeze.md).

> **Further modifications require evidence from subsequent research tracks or a new ADR proposal.**

New ideas go through: Research Track → Evidence → Decision → RFC amendment (005+) or new ADR.

---

## Research Principles

1. **Research does not imply implementation.**
2. **Evidence outweighs assumptions.**
3. **A rejected experiment is a successful research outcome.**
4. **Production architecture changes only through ADRs.**
5. **Sandbox code is disposable.**

---

## Purpose

Answer concrete questions that unblock **RFC 004–006** (and later ADRs), under the Vision’s Funding Boundary: blockchain ends at credit conversion; product spend stays on the Credits ledger.

---

## No production dependencies

Sandbox work **must not**:

- use production billing,
- write to the Credits ledger / `CreditService`,
- treat Vision or RFC 003 as “done” because a PoC existed,
- create an implementation obligation (“we already built it in Sandbox”).

Sandbox artifacts live outside the production money path (separate repo, branch, or disposable lab). Merging Sandbox code into production Billing is out of scope by definition.

---

## Evidence Required

No Decision Register entry without evidence. Acceptable forms include:

- Provider API response (sanitized)
- Provider / chain documentation link
- Test or PoC log
- Diagram
- Timed measurement notes

Link or path the artifact from the Exit Artifact.

---

## Research Tracks

| Track | Research Question | Goal | Output |
|-------|-------------------|------|--------|
| Exchange APIs | What can a provider actually do, and what are the limits? | Evaluate funding providers as *providers*, not Credits SoT | Short report + Exit Artifact |
| Deposit Detection | What is the authoritative event source? | Compare RPC / indexer / explorer roles | Findings + Exit Artifact |
| Wallet Generation | How do WalletIdentity and WalletAddress come into being? | Options for identity vs address materialization | Findings + Exit Artifact |
| Chain Evaluation | What must be generic vs chain-specific? | Polygon first; future networks additive | Comparison + Exit Artifact |
| Key Management | Which keys exist, and who controls them? | **Platform Deposit Keys** / sweeps only — never User Spending Keys | Recommendation → RFC 004 + Exit Artifact |

---

## Success definition

A track succeeds when its **Research Question** is answered (DA / NE / measured finding / clear deferral), **not** when “something ran.”

Examples of good answers:

- Can provider X issue unique deposit addresses? → DA/NE + evidence  
- Can RPC meet latency/reliability for detection? → DA/NE + numbers  
- Is an explorer fallback required? → DA/NE + reason  

---

## Exit Artifact (required per track)

Every closed track produces the same artifact:

| Field | Description |
|-------|-------------|
| Question | What we tried to answer |
| Evidence | What the conclusion rests on (links/paths) |
| Decision | DA / NE / Defer |
| Impact | Which RFC or ADR is affected (if any) |
| Next Action | Discard / RFC amendment / New ADR proposal |
| Knowledge capture | **What did we learn that was previously unknown?** |

Store Exit Artifacts under a consistent docs path (e.g. `docs/architecture/wallet-sandbox-artifacts/`) or link from the Decision Register.

---

## Promotion criteria

Each experiment ends in **exactly one** path:

```text
Research → Discard
Research → RFC amendment
Research → New ADR proposal
```

There is no fourth path of “merge to production because the PoC worked.”

---

## Sandbox Exit Criteria (per PoC)

A PoC is finished when:

1. The Research Question has an answer (including Defer),
2. An Exit Artifact exists with Evidence,
3. **No production code** was merged from the experiment.

---

## Risk Register

Lightweight place to record risks found while researching (expand as tracks run):

| Risk | Impact | Mitigation | Track |
|------|--------|------------|-------|
| Vendor lock-in | High coupling to one funding/custody vendor | Prefer interfaces (RFC 005); Address Provider abstraction | Exchange / Keys |
| API rate limiting | Detection or funding UX fails under load | Backoff, caching, multi-adapter | Exchange / Detection |
| Chain reorganizations | False Confirm → Credited | Confirmation policy; reversible until Credited (RFC 003) | Detection / Chain |
| Platform Deposit Key compromise | Loss/theft of inbound funds before sweep | Key Management track → RFC 004; HSM roadmap | Keys |
| Confusing User Spending Keys with Platform Deposit Keys | Wrong custody model | Vision key-types; never store user spend keys | Keys |

---

## Decision Register

| Topic | Decision | Reason | Date | Link |
|-------|----------|--------|------|------|
| Exchange-as-SoT (e.g. broker credits without Wallet deposit) | *Not current platform direction* | See Vision Funding Boundary. **Any change requires a new ADR.** Sandbox may still research exchanges as Funding Providers. | 2026-08 | [Vision](./roamkit-wallet-platform-vision.md) |
| Etherscan / explorer APIs | Deferred (adapter candidate) | Useful as detection/ops adapter; not Credits SoT | | |
| WalletAddress Assignment (Track 1) | In-house HD Preferred; Index Registry platform state | Exit Artifact | 2026-08 | [Track 1](./wallet-sandbox-artifacts/01-wallet-address-assignment.md) |
| Platform Wallet Infrastructure (RFC 004) | Architecture Review Passed / Frozen | Allocation policy + ownership complete; no blocking infra gaps | 2026-08 | [RFC 004](../rfcs/004-platform-wallet-infrastructure.md), [Freeze](./wallet-architecture-freeze.md) |
| Wallet foundation freeze | Vision + Conversion Boundary + RFC 003 + RFC 004 frozen | Discipline: no change without research evidence or ADR proposal | 2026-08 | [Freeze](./wallet-architecture-freeze.md) |
| Funding Provider Interface (RFC 005) | Architecture Review Passed / Frozen | Generic adapter contract; destination = WalletAddress; never Credits SoT | 2026-08 | [RFC 005](../rfcs/005-funding-provider-interface.md), [Freeze](./wallet-architecture-freeze.md) |

Add rows as Exit Artifacts close. Always fill **Reason**.

---

## Unknowns Remaining

What this framework has **not** answered yet (prioritized for sequencing RFC work):

| Unknown | Priority |
|---------|----------|
| Confirmation policy parameters (depth/finality) and primary observation adapter | High → **RFC 006** + research |
| Sweep policy and treasury destination (ADR detail under RFC 004) | Medium |
| Multi-chain beyond Polygon | Medium |
| On-chain Withdraw to user | Low |
| Card on-ramp / first Funding Provider vendor choice | Low |

**Answered (removed from unknowns):** Platform Wallet Infrastructure (Track 1 / RFC 004); Funding Provider interface shape (RFC 005); Funding Providers never define WalletIdentity.

Update this table as tracks close; move answered items into the Decision Register.

---

## Relationship to RFC 004 / 005 / 006

[RFC 004](../rfcs/004-platform-wallet-infrastructure.md) and [RFC 005](../rfcs/005-funding-provider-interface.md) are **Architecture Review Passed / Frozen**.

Current Draft:

**[RFC 006 — Deposit Observation & Confirmation](../rfcs/006-deposit-observation-confirmation.md)**

In scope: when a deposit is Confirmed enough for Credit Conversion; observation adapters are non-authoritative alone.

Then Wallet Platform ADR (cutover from ADR 010) when implementing.

---

## Related

- [Wallet Architecture Freeze](./wallet-architecture-freeze.md)
- [RoamKit Wallet Platform Vision](./roamkit-wallet-platform-vision.md)
- [Wallet Conversion Boundary](./wallet-conversion-boundary.md)
- [RFC 003 — Wallet Domain & Ownership Model](../rfcs/003-wallet-domain-ownership-model.md)
- [RFC 004 — Platform Wallet Infrastructure](../rfcs/004-platform-wallet-infrastructure.md)
- [RFC 005 — Funding Provider Interface](../rfcs/005-funding-provider-interface.md)
- [RFC 006 — Deposit Observation & Confirmation](../rfcs/006-deposit-observation-confirmation.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
