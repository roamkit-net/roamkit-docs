# Wallet Sandbox

| Field | Value |
|-------|-------|
| Status | Draft / research framework (non-normative) |
| Date | 2026-08 |
| Parents | [Wallet Vision](./roamkit-wallet-platform-vision.md), [Conversion Boundary](./wallet-conversion-boundary.md), [RFC 003](../rfcs/003-wallet-domain-ownership-model.md) |

> The Sandbox converts **unknowns into decisions**. It is not a place to grow production code or to bypass ADRs.

After this document is accepted as the framework, treat **Vision**, **RFC 003**, **Conversion Boundary**, and **this Sandbox framework** as **frozen**. New ideas go through: Research Track → Evidence → Decision → RFC amendment or new ADR.

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
| Platform Deposit Keys | Feed RFC 004 | Infrastructure for receive/sweep if self-issued addresses | | [RFC 004 intent](#relationship-to-rfc-004) |

Add rows as Exit Artifacts close. Always fill **Reason**.

---

## Unknowns Remaining

What this framework has **not** answered yet (prioritized for sequencing RFC work):

| Unknown | Priority |
|---------|----------|
| Whether / how Platform Deposit Keys are held (in-house vs Address Provider) | High |
| Authoritative deposit event source (RPC vs indexer vs hybrid) | High |
| Address materialization policy (eager / lazy / HD / per-payment) | High |
| Funding Provider shortlist and hard limits | Medium |
| Multi-chain beyond Polygon | Medium |
| On-chain Withdraw to user | Low |
| Card on-ramp vendor choice | Low |

Update this table as tracks close; move answered items into the Decision Register.

---

## Relationship to RFC 004

After Key Management (and related) tracks produce Exit Artifacts, draft:

**RFC 004 — Platform Wallet Key Management**

In scope: Platform Deposit Keys, sweep keys, platform signing API for *infrastructure*, keystore, HSM roadmap, rotation, audit, backup.

**Out of scope:** User Spending Keys, on-chain auto renew, Credits mutations.

Then RFC 005 (Funding Provider Interface), RFC 006 (Deposit Detection), then the first Wallet Platform ADR.

---

## Related

- [RoamKit Wallet Platform Vision](./roamkit-wallet-platform-vision.md)
- [Wallet Conversion Boundary](./wallet-conversion-boundary.md)
- [RFC 003 — Wallet Domain & Ownership Model](../rfcs/003-wallet-domain-ownership-model.md)
- [ADR 010](../adr/010-polygon-usdt-prepaid-credits.md)
