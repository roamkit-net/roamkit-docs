# Research Track 1 — WalletAddress Assignment

**Status:** Track closed · Research complete  
**Feeds:** RFC 004 — Platform Wallet Infrastructure (draft after this artifact is merged)  
**Does not feed:** Vision amendments (unless evidence forces it) · No RFC 004 in this PR

---

## Research Question

> **How does RoamKit assign and manage `WalletAddress` per Account?**

HD wallet, custody provider, MPC/TEE infra, and funding-provider addresses are **candidate implementations**, not presupposed answers.

---

## Working assumptions (already agreed)

| Assumption | Status |
|------------|--------|
| Target architecture is **Option A** (RoamKit Wallet → Credits → Platform) | Locked |
| Blockchain ends at convert-to-Credits; product ops use Credits only | Locked |
| **One active address per Account + Chain** (Polygon first); rotation later | Preferred for v1 |
| Brand Design System is out of scope for this track | Locked |

---

## Terminology

Prefer **Platform Wallet Infrastructure** over “Platform Deposit Key.”

Infrastructure may be implemented as HD seed, MPC, HSM/KMS-wrapped material, custody provider, or a combination. RFC 004 should stay correct regardless of which option this track prefers.

---

## Candidate options

| # | Option | What it means | Role in comparison |
|---|--------|---------------|--------------------|
| 1 | **In-house HD / derived addresses** | Platform derives per-Account addresses from controlled material | Core candidate · **benchmark** |
| 2 | **Custody Address Provider** | Third party issues/holds receive addresses (classic custody vault UX) | Core candidate (Fireblocks-style) |
| 3 | **Funding Provider address** | e.g. exchange deposit mapped to Account | Compare; not presumed core |
| 4 | **MPC / TEE Provider** | Fireblocks (MPC), DFNS (MPC), Turnkey (TEE+HD), Privy (TEE/embedded) | Reference set — same matrix |

---

## Must-answer questions (answered)

### 1. Can `WalletAddress` be regenerated deterministically?

| Option | Answer | Evidence |
|--------|--------|----------|
| **In-house HD** | **DA** | Same BIP39 seed + **persisted** derivation index → same EVM address. Lab PoC below. **Index allocation is platform state** (see Index Registry). |
| Custody / Fireblocks | **Conditional** | Workspace recovery kits / vault structure; addresses tied to provider vault IDs. Not regenerable offline without recovery material. |
| Turnkey | **DA (via provider)** | HD wallets in TEE; export mnemonic/account under policy ([export docs](https://docs.turnkey.com/wallets/export-wallets)). Offline regen only after export. |
| DFNS | **Conditional** | Native wallets are MPC shares (not a single offline mnemonic). **Import** of HD master possible (Enterprise) → then BIP-44 derive ([import keys](https://docs.dfns.co/guides/developers/import-keys)). |
| Privy | **Weak / product-mismatch** | Oriented to user embedded wallets + export; not a clean platform deposit-address model for Option A. |
| Funding Provider | **NE (as RoamKit core)** | Address ownership/lifecycle is the exchange’s. |

**Lab PoC (2026-08-05):** `eth-account` BIP44 `m/44'/60'/0'/0/{index}` from fixed mnemonic regenerated identical addresses across two runs. Polygon PoS uses the same EVM address (coin type 60); **Chain is metadata on `WalletAddress`, not a separate key** for Polygon-first.

### 2. Migration cost (into / out of)

| Direction | Feasible? | Cost | Notes |
|-----------|-----------|------|-------|
| HD → Turnkey / DFNS / Fireblocks | Yes, with caveats | Medium–High | Keep **published** addresses live for a watch window; import seed or per-key where supported; or cut over by issuing **new** addresses (UX pain). DFNS documents Fireblocks HD import ([migrate Fireblocks→DFNS](https://dfns.co/article/how-to-migrate-from-fireblocks-to-dfns)). Turnkey supports import + export. |
| Fireblocks → HD | Hard | High | Need Recovery Kit / export path; operationally heavy; in-flight deposits on old vault addresses must keep being watched. |
| HD → “HSM-native BIP32” | **Not clean** | High | **AWS CloudHSM does not support BIP32 derivation** ([AWS re:Post](https://repost.aws/questions/QUWfgjjRv4RgWnNxKWULnedQ/does-aws-cloudhsm-support-bip32-key-derivation)). Realistic path: **KMS-wrap the seed** (or encrypt seed at rest) and derive in a tightly controlled process; or move signing to provider (Turnkey/Fireblocks). AWS KMS can sign secp256k1 **per key**, not HD trees. |
| Stay on Funding Provider addresses as core | N/A | Strategic | Conflicts with Option A; attribution and SoT leave RoamKit. |

**Summary:** Deterministic HD makes **outbound** migration to providers that accept HD import the cheapest path. **Inbound** from Fireblocks without recovery material is painful. Plan for **address watch continuity** during any cutover.

### 3. Lifecycle under “one active per Account + Chain”

Recommended model (fits HD benchmark; portable to providers):

```text
Account
  └── WalletIdentity          (stable domain id; not a key)
        └── WalletAddress[]   (Polygon first)
              active | retired
```

| Event | Behavior |
|-------|----------|
| First materialize | Lazy: first “Add funds” → allocate next derivation index → one **active** Polygon address |
| Rotate | New index → new active; old → **retired** but **still monitored** for late deposits |
| Never | New address per payment (v1) |
| Sweep | Platform infra moves funds to treasury (ops); Credits still from detection→convert |

---

## 1) In-house HD — detailed findings (benchmark)

### How is `WalletIdentity` generated?

- **Domain object**, not a blockchain key: create when Account is eligible for Wallet (e.g. registration or first funding intent).
- Stable UUID (or similar) owned by RoamKit; maps 1:1 to Account (per RFC 003).
- Does **not** require seed access to create.

### How is `WalletAddress` derived per Account + Chain?

- Platform holds **one master seed** (Platform Wallet Infrastructure).
- Persist `derivation_index` (monotonic integer) on `WalletAddress` / Account mapping — **never** invent addresses without storing the index.
- Suggested path (EVM / Polygon): `m/44'/60'/0'/0/{derivation_index}` (BIP44; industry default for EVM).
- `Chain=Polygon` is a field on the address record; same bytes as ETH address format.
- **WalletIdentity stays chain-agnostic**; Polygon is metadata, which keeps RFC 003’s model clean.

### Index Registry (architectural requirement for RFC 004)

It is not enough to say “same seed + same index = same address.”

> **Index allocation is part of the platform state.**

Recovery is **not**:

```text
Seed → WalletAddress
```

Recovery **is**:

```text
Seed + Index Registry → WalletAddress
```

| Component | Role |
|-----------|------|
| Master seed | Cryptographic root (backup/restore) |
| **Index Registry** | Durable map: Account / WalletIdentity ↔ derivation_index ↔ address string (+ lifecycle) |

RFC 004 must treat the Index Registry as a first-class platform store, not an implementation detail of “HD crypto.”

### Backup and recovery

| Asset | Backup |
|-------|--------|
| Master seed | Encrypted at rest (e.g. KMS-wrapped); offline / Shamir or dual-control cold copy; access audited |
| Index Registry | Database (Account ↔ derivation_index ↔ address string) — **required** for recovery even with seed |
| Recovery drill | From seed + Index Registry, recompute all addresses; confirm match DB; resume watchers |

Without the Index Registry, gap-limit scanning is possible but operationally worse — **treat the registry as first-class platform state**.

### Address rotation

- Increment `derivation_index` (or address_index); mark previous address retired.
- Keep detecting deposits on retired addresses for a defined window (ops policy → RFC 004).
- User-facing UI shows only the active address.

### Platform Wallet Infrastructure evolution (not “HSM implementation”)

Reality check: **HSM ≠ HD wallet.** AWS CloudHSM does not support BIP32. RFC 004 must **not** frame the end state as “HSM implementation,” but as **evolution of Platform Wallet Infrastructure** (wrap seed → enclaves/MPC provider → whatever evidence later supports).

| Approach | Fit |
|----------|-----|
| KMS-wrap seed; derive/sign in controlled runtime | Practical near-term hardening |
| Per-address key in AWS KMS (`ECC_SECG_P256K1`) | Possible for sweeps; loses cheap unlimited HD; EIP-2 `s`-value handling required ([aws-kms-ethereum-accounts](https://github.com/aws-samples/aws-kms-ethereum-accounts)) |
| Native BIP32 on CloudHSM | **Not supported** |
| Move to Turnkey / DFNS / Fireblocks later | Migration path if ops maturity requires it |

### Operational cost (order-of-magnitude)

| Item | HD in-house |
|------|-------------|
| Vendor fees | None (infra only) |
| Eng | Seed custody procedures, watcher, sweep, runbooks |
| Gas | Sweep + gas top-up for ERC-20 receive addresses |
| Compliance | You own key-ceremony / access control story |
| Scale | Address generation is O(1) crypto; bottleneck is detection/sweep ops |

---

## 2) Custody / MPC / TEE providers — comparison notes

Not a product bake-off score; capability fit for **platform receive addresses** (Option A).

| Provider | Model | Per-user Polygon USDT address | Deterministic / exit | Rough cost signal | Fit for RoamKit v1 |
|----------|-------|--------------------------------|----------------------|-------------------|--------------------|
| **Fireblocks** | MPC custody | Account-based assets: **vault account per user** then wallet; omnibus + sweep ([deposits at scale](https://developers.fireblocks.com/docs/manage-deposits-at-scale)) | Recovery kit / enterprise exit; migration documented by others (e.g. DFNS) | Enterprise sales (typically high) | **Deferred due to current product scope** (not rejected) |
| **Turnkey** | TEE + **HD** (not classic MPC) | Derive many EVM accounts via BIP32 API | Export wallet/mnemonic under policy; import for DR | Public: from ~$0.05–0.10/signature; wallet tiers ([pricing](https://www.turnkey.com/pricing)) | **Future migration candidate** from in-house HD |
| **DFNS** | MPC | Create wallets via API; HD import Enterprise | Import/export paths; Fireblocks→DFNS story | Enterprise | **Future migration candidate**; import can preserve addresses if seeded from HD |
| **Privy** | TEE / embedded (+ optional licensed custodian) | User/auth-centric wallets; MAU pricing ([pricing](https://www.privy.io/pricing)) | User export flows | Free→$299+/mo MAU bands | **Weak fit** for platform deposit→Credits core (product is user wallet UX) |

**Custody vs MPC column in matrix:** Fireblocks exemplifies “Custody Address Provider”; Turnkey/DFNS exemplify managed infra; matrix rows below collapse nuances into Custody vs MPC where needed, with notes.

---

## 3) Funding Provider (MEXC) — short UX evidence (not architecture SoT)

Used only to score the **Funding Provider address** column.

> **Funding Providers never define `WalletIdentity`.**

This is not MEXC-specific. It applies equally to Binance, OKX, MoonPay, and any future on-ramp. They remain **adapters** that help value reach a RoamKit-owned `WalletAddress`.

| Question | Finding |
|----------|---------|
| Buy USDT with card? | Yes — Quick Buy / card; KYC required ([MEXC how-to-buy](https://www.mexc.co/how-to-buy)) |
| Withdraw to external address? | Yes — Assets → Withdraw → USDT → **must select Polygon** |
| User steps (typical) | Register → KYC → card buy USDT → withdraw → pick network → paste RoamKit address → confirm |
| Polygon support | Supported; wrong network = loss risk (ops copy already relevant in Deposit UX) |
| Per-user exchange address as RoamKit core? | **No** — does not give RoamKit-owned `WalletAddress` lifecycle |

Full Funding Provider track (card friction, fees, limits) can deepen later; **not blocking** address-assignment decision.

---

## Explicitly out of scope (this track)

- Withdraw to user · Swap · Staking  
- Multi-chain beyond “Polygon first”  
- Deposit Detection / RPC vs Indexer / Etherscan (next track, after this decision)  
- Production implementation in `roamkit-api` / `roamkit-web`

---

## Evaluation Matrix

Scale: **Strong** / **Acceptable** / **Weak** / **N/A** (+ note).

| Criterion | HD / derived | Custody Provider (e.g. Fireblocks) | MPC/TEE Provider (Turnkey / DFNS / Privy*) | Funding Provider address |
|-----------|--------------|------------------------------------|---------------------------------------------|--------------------------|
| Ownership | **Strong** — RoamKit controls seed + index map | **Weak–Acceptable** — provider workspace owns keys | **Acceptable** — policy/API control; material in enclave/MPC | **Weak** — exchange owns deposit rail |
| Vendor lock-in | **Strong** (low) — standards BIP32/44 | **Weak** — vault IDs, recovery kit, commercial | **Acceptable** — Turnkey/DFNS have export/import; still vendor runtime | **Weak** — product tied to that exchange |
| Cost | **Strong** — infra + eng; no per-wallet SaaS | **Weak** — enterprise custody pricing | **Acceptable** — usage (signatures/wallets); Privy MAU | **Acceptable** — user pays exchange fees; RoamKit pays integration |
| Scalability | **Strong** — derive O(1); ops bottleneck is watch/sweep | **Acceptable** — pre-create vault pools; API limits | **Acceptable** — API scale; watch still yours or theirs | **Weak** for *RoamKit* scale of unique addresses |
| Security | **Acceptable** — excellent if seed ops are disciplined; single-seed blast radius | **Strong** — MPC, policies, institutional controls | **Strong** — TEE/MPC; depends on policy hygiene | **N/A** to RoamKit key security |
| UX (deposit attribution) | **Strong** — unique address → Account | **Strong** — same pattern via vault mapping | **Strong** — same if you issue per-user addresses | **Weak** — memo/shared/cex UX; not RoamKit Wallet |
| Recovery | **Strong** — seed + index map | **Acceptable** — provider recovery procedures | **Acceptable–Strong** — export/DR products | **Weak** — support tickets with exchange |
| Deterministic regeneration | **Strong** — **DA** (PoC) | **Weak–Acceptable** — not offline BIP44 without kit | **Acceptable** — HD-capable (Turnkey); MPC native ≠ mnemonic | **N/A** / **NE** for RoamKit |
| Migration cost (into / out of) | **Strong outbound** to HD-import providers; HSM-native BIP32 **Weak** | **Weak outbound** without recovery; inbound from HD possible | **Acceptable** both ways if HD seed preserved | **Strong** to leave (just stop using); **Weak** as core to enter |

\*Privy: treat as **Weak** overall fit for this research question (platform deposit infrastructure), despite solid embedded-wallet product.

---

## Architecture Decision Matrix (Exit)

| Option | Result | One-line reason |
|--------|--------|-----------------|
| **In-house HD / derived** | **Preferred** (v1 benchmark) | Deterministic regen, ownership, low lock-in, matches one-address-per-Account+Chain; HSM = wrap/process, not BIP32 fairy tale |
| **Custody Provider** (Fireblocks-class) | **Deferred** (current product scope) | Not rejected — institutional controls may matter later; unjustified while Credits are the only spend path |
| **MPC / TEE Provider** (Turnkey / DFNS; Privy out) | **Future migration candidate** | Natural upgrade path **from** HD (import seed / accounts) when ops or compliance demand enclaves/MPC |
| **Funding Provider address** | **Not core architecture** | Valid UX rail to fund a RoamKit address; **Funding Providers never define `WalletIdentity`** |

---

## Track closure checklist

Did this research answer every question the Track posed?

| Track question | Answered? |
|----------------|-----------|
| How does RoamKit assign and manage `WalletAddress` per Account? | **DA** — Preferred: in-house HD + Index Registry |
| Evaluation Matrix (Ownership…Migration) filled with evidence? | **DA** |
| Deterministic regeneration? | **DA** for HD iff seed + Index Registry |
| Migration cost HD ↔ providers / “HSM”? | **DA** |
| MPC/custody options compared (not surface skim)? | **DA** |
| Funding Provider as core address source? | **NE** — adapters only |
| Withdraw / swap / staking / multi-chain / Deposit Detection? | Explicitly out of scope |

**Track closed.** Next: merge this artifact, then open **RFC 004 — Platform Wallet Infrastructure** from consequences below. Do **not** start Deposit Detection until RFC 004 scope is drafted from this decision.

---

## Architectural Consequences

### Accepted

- `WalletAddress` ownership belongs to RoamKit.
- In-house HD is the v1 assignment mechanism.
- **Index Registry is platform state** (seed alone is insufficient for recovery).
- One active address per Account + Chain (Polygon first); rotation later.
- `WalletIdentity` remains provider-agnostic; Chain (e.g. Polygon) is metadata on `WalletAddress`.
- Funding Providers are interchangeable adapters; they **never** define `WalletIdentity`.
- Platform Wallet Infrastructure evolves over time (wrap → provider/enclave); not framed as a fixed “HSM implementation.”

### Deferred

- Fireblocks-class custody (**deferred due to current product scope**, not rejected)
- MPC / TEE adoption as day-one runtime (Turnkey / DFNS = future migration candidates)
- Native HSM BIP32 (not available on AWS CloudHSM; not a v1 requirement)

### Out of scope (for this track and for early RFC 004 drafting)

- Withdrawals to users
- User Spending Keys
- Multi-chain beyond Polygon-first
- Deposit Detection stack (RPC / indexer / explorer) — separate track after RFC 004 intake
- Swap / staking

---

## Exit Artifact (required fields)

| Field | Content |
|-------|---------|
| Question | How does RoamKit assign and manage `WalletAddress` per Account? |
| Evidence | BIP44/EVM path standards; lab PoC 2026-08-05; Fireblocks deposits-at-scale docs; Turnkey wallets/export/pricing; DFNS import + Fireblocks migration article; AWS CloudHSM BIP32 re:Post; Privy wallet types/pricing; MEXC buy/withdraw public guides |
| Decision | **Preferred: in-house HD** with `WalletIdentity` + Index Registry + one active `WalletAddress` per Account+Chain; custody Deferred (scope); MPC/TEE Future migration; Funding Providers never define WalletIdentity |
| Impact | Enables draft **RFC 004 — Platform Wallet Infrastructure** (separate PR): seed custody, Index Registry, derivation policy, rotation, sweep, migration hooks |
| Next Action | **Track closed** → merge this artifact → draft RFC 004 (no Vision rewrite; no Deposit Detection yet) |
| Knowledge capture | (1) Polygon address = EVM address (coin 60), chain is metadata. (2) Recovery = Seed + Index Registry. (3) HSM ≠ BIP32. (4) Fireblocks deferred by scope, not rejected. (5) Turnkey/DFNS = migration candidates. (6) Funding Providers never define WalletIdentity. (7) Privy poor fit for platform deposit core. |

Also recorded:

- **Deterministic regeneration:** DA for Preferred option (HD) **iff** Index Registry is intact.  
- **Migration cost:** Medium outbound HD→provider if addresses preserved via HD import; High if renumbering; HSM-native BIP32 not available on AWS CloudHSM.

---

## Related

- [Wallet Sandbox](../wallet-sandbox.md)
- [RFC 003 — Domain & Ownership](../../rfcs/003-wallet-domain-ownership-model.md)
- [Wallet Conversion Boundary](../wallet-conversion-boundary.md)
- [Vision](../roamkit-wallet-platform-vision.md) (frozen; Option A / Funding Boundary)
