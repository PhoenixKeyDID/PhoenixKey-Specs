# PhoenixKey
## Formal Mathematical Specification — v4.6

---

| Field | Value |
|-------|-------|
| **Version** | 4.6 |
| **Date** | 2026-05-30 |
| **Basis** | v4.4 + Tuân peer review: 3 critical spec bugs + 2 design trade-offs + 4 editorial fixes |
| **Status** | Testnet-Ready — Normative |
| **Network** | Cardano L1 / Aiken–PlutusV3 |
| **Scope** | Complete mathematical specification: cryptographic core + 10 DID types + full lifecycle + compliance + recovery completeness |
| **Review** | v4.2: Gemon (MIT Math), Serat (Security); v4.4–4.5: Sambo (BA MIT Sloan), Tuân (Validator Engineer) |
| **Fixes over v4.4** | **[Bug]** salt_pw₁ circular dependency removed — all salts now on-chain, protection by Argon2id memory-hardness; **[Bug]** `secondary_wallet_enrolled:𝔹` → `secondary_wallets:List<SecondaryWallet>` with pkh commitment; **[Bug]** Email upgraded from knowledge factor to possession factor via EmailOracle OTP (VeData-pattern); **[Bug]** Cancel Option A removed from §11.7 — knowledge factors cannot distinguish owner from attacker; **[Design]** I-RECOVERY-4 grace period (30-day backup_deadline, restricted scope, not hard block); **[Design]** I-RECOVERY-5 recursive orphan edge case; **[Editorial]** Argon2id m unit MiB; HIBP k-anonymity reference; I-TIER5-PW-5 removed |
| **Added in v4.6** | **[New]** §36 Transaction Fee Architecture — per-operation PhoenixKey fee with 30/70 split: 30% to Cardano Treasury via Conway-era native treasury-donation (tx body field 20), 70% to Phoenix Treasury (Plutus V3 script, OrgDID m-of-n). **Fees are on-chain adjustable parameters** (`FeeParams` reference UTxO) updatable by **DAO governance vote OR an algorithmic module** — no script-hash rebake. **PersonDID create fee = 0 at launch** (user-acquisition incentive; zero-fee ops skip the fee mechanism, like `Deactivate`). I-FEE-1/2 enforced on-chain by a fee-receipt minting policy (Aiken stdlib v2 `treasury_donation` accessor). Theorem 36.1 fee accountability. **[Reconcile §29]** §29 clarified as PhoenixKey-side DID-mapping layer; reward math is canonical MAGIC AppEconomics v2.1 (`computeW` 5-factor + 30% cap), DID-agnostic — legacy v1.0-SA `holder_share_bps` superseded. |
| **Pending v4.7 (this patch)** | **[Bug, Errata CID-6]** §3.3 (new): defines capability subsumption `⊑` — a preorder where `c ⊑ Full_Authority` for all `c`, backward-compatible with plain set-⊆. §4.4 `DelegationToken_valid` and §23 `Op_delegate`/sub-delegation switched from `⊆` to `⊑`: previously a PersonDID (`scope={Full_Authority}` per C-SCOPE-3) could not delegate *any* concrete capability, since `{Read_DID(Asset)} ⊆ {Full_Authority}` is False under plain set membership — blocking all PersonDID-issued delegation through §4.4/§23 while §4.2 `Authority` already handled `Full_Authority` via an ad-hoc disjunct (internal inconsistency). Scope of this patch is deliberately narrow: only §1.1 (notation), §3.3 (new), §4.4, §23, and TV-6 changed. §3.2/§4.2/§19/§20 and all proofs referencing set-⊆ properties (cascade scope, Theorem 27.1) are **not** touched by this patch — **[CẦN CHỐT]** whether/how those should also move to `⊑` is a separate follow-up (see `spec-proposals/ServiceDID-SelfService-and-Delegation-DRAFT.md` §3.3 for the wider proposal this patch was extracted from). **Formal version-number bump (title/header) intentionally deferred** — this patch touches 2 sections; maintainer should decide whether to bump to v4.7 now or bundle with a larger release. |

---

> **→ Hiện-trạng triển-khai từng phần:** xem [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md).

---

## Abstract

PhoenixKey is a decentralized identity and asset management system on Cardano that eliminates seed phrase memorization through hardware-secured key generation and tiered recovery. This document is the definitive mathematical specification. v4.5 closes four spec bugs identified in v4.4 peer review: (1) salt_pw₁ circular dependency removed — all salts on-chain, protection via Argon2id memory-hardness; (2) boolean secondary_wallet field replaced with SecondaryWallet commitment list; (3) email upgraded from knowledge to possession factor via EmailOracle OTP; (4) cancel Option A removed. Additionally: grace-period enforcement (I-RECOVERY-4 revised), recursive orphan (I-RECOVERY-5), editorial fixes.

**Key results:** Theorems 27.1–27.7, 28.1–28.2. TAAD ASM Invariants A-1..A-6. I-RECOVERY-1..5. I-TIER5-PW-1..4, I-TIER5-CANCEL-1..3.

**Out of scope (separate documents):**
- PhoenixKey DID Method Specification v1.0 — resolver protocol, I-RESOLVER-1 implementation
- PhoenixKey Cardano Validator Reference v1.0 — Aiken/PlutusV3 implementation
- VeData Formal Specification — biometric uniqueness proofs, AI liveness model, environmental sensing internals
- LampNet Network Specification — routing parameters (K, β, c, δ), storage economics
- PhoenixKey UI Specification — UX flows, screen designs (not mathematical)
- Service pricing table — MAGIC per operation (operational decision, not protocol)
- Mechanized proofs (Coq/Isabelle) — target v5.0

---

## Normative Status

**[N]** Normative — implementations MUST satisfy.
**[I]** Informative — design rationale, not binding.
All appendices non-normative unless marked [N].

---

## Table of Contents

**Part I — Foundations**
§1 Notation · §2 Universal DID Base · §3 Capability and Scope · §4 Authority Model · §5 Lifecycle

**Part II — Core Cryptography**
§6 Key Hierarchy · §7 LampNet Distributed Storage · §8 Cardano Wallet · §9 Seed Export and Import

**Part III — TAAD Protocol**
§10 TAAD State Machine · §11 Recovery Protocol
  - §11.0 Recovery Scope (PersonDID only; non-Person; orphan; biometric clarification)
  - §11.1 Tier Selection cascade · §11.2 Tier 1 (secondary wallet) · §11.3 Tier 2/3 (guardian)
  - §11.4 Post-recovery update · §11.5 VeData integration
  - §11.6 Tier 5 (knowledge-based: password + email OTP) · §11.7 Tier 5 cancel mechanism

**Part IV — DID Type Catalog**
§12 PersonDID · §13 OrgDID · §14 ContextDID · §15 DeviceDID · §16 MachineDID
§17 AssetDID · §18 BotDID · §19 AgentDID · §20 ServiceDID · §21 CharDID

**Part V — Cross-Type Interactions**
§22 Ownership Graph (§22.3 Taxonomy Rationale: Bot/Agent/"Robot") · §23 Delegation (§23.1 DID Authorization Registry) · §24 Revocation · §25 MAGIC Attribution

**Part VI — Security Analysis**
§26 Global Invariants · §27 Theorems · §28 Attack Model

**Part VII — Integration**
§29 AppTokenEconomics · §30 PPA Claim Mapping · §31 Protocol Boundary

**Part VIII — Lifecycle Completeness** *(new v4.3)*
§32 End-of-Life Protocols · §33 Emergency Vault · §34 Privacy and Compliance · §35 Layer 0 Architecture Boundary · §36 Transaction Fee Architecture *(v4.6)*

**Appendices**
A: MECE Coverage · B: DID Construction · C: Capabilities · D: Breaking Changes · E: Glossary · F: Test Vectors · G: References

---

# PART I — FOUNDATIONS

## §1 Notation and Primitives [N]

### §1.1 Mathematical Notation

```
ℕ            natural numbers {0, 1, 2, ...}
𝔹            booleans {True, False}
{0,1}^n      bit strings of length n
ByteArray   := {0,1}*   arbitrary-length bit string
Option<T>   := T | None
Set<T>       finite unordered set
List<T>      ordered sequence
Map<K,V>     finite partial function

|S|          cardinality
⊆, ∈, ∪, ∩, ∅   standard set operations
⊑            capability subsumption ("is covered by") — see §3.3 [N, v4.7]
∀, ∃, ∧, ∨, ¬, →   logical operators
≜            definitional equality
(a ?? b)    ≜  if a ≠ None then a else b
```

### §1.2 Cryptographic Primitives

```
-- Hash functions:
H(x)         := BLAKE2b-256(x)  → {0,1}^256   (default)
H_sha256(x)  := SHA-256(x)      → {0,1}^256

-- Digital signatures (Ed25519):
Sign(sk, m)   → {0,1}^512
Verify(pk, m, σ) → 𝔹
pk = Ed25519.PubKey(sk)

-- Sentinel key values:
NULL_KEY : Ed25519PubKey ≜ 0^256  -- all-zeros; used where no real key applies (e.g. Posthumous recovery)
NULL_PKH : {0,1}^224       ≜ 0^224  -- all-zeros payment key hash

-- Key derivation:
HKDF(key, info, salt) → {0,1}^256    (RFC 5869, HMAC-SHA256)
BIP32_Derive(seed, path) → Ed25519PubKey   (CIP-0003, IOHK variant)

-- Authenticated encryption (cryptographically agnostic, cf. §34):
Enc(k, m) → ByteArray     (AES-256-GCM or ChaCha20-Poly1305)
Dec(k, c) → ByteArray | ⊥

-- BIP-39:
BIP39_Encode(entropy: {0,1}^256) → List<Word>   (24 words)
BIP39_ToEntropy(words: List<Word>) → {0,1}^256

-- Threshold secret sharing (Shamir):
TSS_Share(secret, k, n)        → List<{0,1}^256>   (k-of-n Shamir shares)
TSS_Reconstruct(shares, k)     → {0,1}^256 | ⊥
  ≜ if |shares| < k then ⊥ else Shamir_interpolate(shares, k)

-- Cardano-specific:
Blake2b224(x)  → {0,1}^224   (VerificationKeyHash on Cardano)
Bech32m(x)     → String       (address encoding)
```

### §1.3 Cardano Primitives

```
SlotNo      := ℕ            Cardano slot number
PolicyID    := {0,1}^224    Cardano minting policy
ScriptHash  := {0,1}^224    Plutus script hash
Address     := ByteArray     Bech32m address

slot_to_epoch(s) → ℕ        1 epoch = 432,000 slots ≈ 5 days
current_slot : SlotNo

-- CIP-0031 Reference Inputs: read-only UTxO access.
-- Scope checks read ancestor datums at block-start UTxO set (not intra-block).
```

### §1.4 PhoenixKey Constants

```
-- Core bounds:
Q                       = 10^9         Q-format precision
MAX_DELEGATION_DEPTH    = 10           ownership chain cap
MAX_GUARDIAN_COUNT      = 7            max guardians per PersonDID
MIN_GUARDIAN_THRESHOLD  = 2
MIN_GUARDIAN_COLLATERAL = 50_000_000   lovelace (50 ADA)

-- TAAD timelocks (1 epoch = 432,000 slots ≈ 5 days):
TAAD_TIMELOCK_T1         = 7  × 432_000  -- ~7 days  (Tier 1)
TAAD_TIMELOCK_T2         = 10 × 432_000  -- ~10 days (Tier 2)
TAAD_TIMELOCK_T3         = 14 × 432_000  -- ~14 days (Tier 3)
TIER5_TIMELOCK           = 21 × 432_000  -- ~21 days (Tier 5: weakest)

-- Backup grace period:
TIER5_BACKUP_GRACE_SLOTS = 30 × 432_000  -- ~30 days after registration
HIGH_STAKES_THRESHOLD    = 100_000_000   -- 100 ADA in lovelace (governance param)

-- End-of-life timelocks:
DEATH_TIMELOCK           = 30 × 432_000  -- ~30 days observation before Posthumous finalizes
INCAPACITY_MIN_SLOTS     = 1  × 432_000  -- minimum 1 epoch for incapacity duration

-- BotDID / ServiceDID:
BOT_REFILL_WINDOW = 1000   slots (~3.3 hours)
MAX_SERVICE_DEPTH = 3      Service→Service chain limit

-- Oracle public keys (governance-controlled; rotate via PhoenixKey OrgDID multisig):
-- VEDATA_ORACLE_PK : Ed25519PubKey  (see §11.5; VeData env attestation)
-- EMAIL_ORACLE_PK  : Ed25519PubKey  (see §11.6.2; Tier 5 email possession)
-- Both are known public constants deployed with the validator.
-- Rotation: PhoenixKey OrgDID governance vote (not unilateral; see §35.2 M-7).
```

---

## §2 Universal DID Base Structure [N]

### §2.1 DID Identifier

```
DID : ByteArray

DID_construct(type, creator, slot) → DID ≜
  "did:phoenix:" ++ base32(slot) ++ ":" ++
  hex(H(encode(type) ++ (creator ?? "root") ++ encode(slot) ++ rand_256))
  where rand_256 ← SecureRandom(256)

Collision: P(any two collide | n DIDs) ≤ n²/2^257 ≈ 5.4×10^{-54} at n=10^12
```

### §2.2 DID Types

```
DIDType ≜
  | Person     -- Layer 0: root of trust (biological human)
  | Org        -- Layer 1: collective entity (company, DAO, government)
  | Context    -- Layer 1: temporal bounded context (event, project)
  | Device     -- Layer 2: computing hardware (phone, server, sensor)
  | Machine    -- Layer 2: physical machine (vehicle, robot)
  | Asset      -- Layer 2: passive physical asset (land, crop, good)
  | Bot        -- Layer 3: rule-based automation (ProofChat bot, keeper)
  | Agent      -- Layer 3: autonomous AI (TigerAgent, MagicClaw)
  | Service    -- Layer 3: digital service (OriLife, AladinWork)
  | Character  -- Layer 4: virtual entity (game character, avatar)
```

### §2.3 DID Base Record [N]

```
-- SuspendedByInfo: union type prevents mismatch between DID and hash (errata E-01):
SuspendedByInfo ≜
  | Suspender(did: DID)                   -- non-PersonDID: the suspending owner DID
  | GuardianCoalition(proof: {0,1}^256)   -- PersonDID: H(signer_set ++ slot)

DIDBase ≜ {
  did             : DID,
  type            : DIDType,
  owner           : Option<DID>,              -- None iff type = Person
  scope           : Set<Capability>,
  security_level  : SecurityLevel,
  created_slot    : SlotNo,
  revoked_slot    : Option<SlotNo>,           -- None = active
  suspended_by    : Option<SuspendedByInfo>,  -- None = not suspended
  suspended_until : Option<SlotNo>,           -- None = indefinite suspension
  seq             : ℕ,
  on_chain        : Address,
  metadata_hash   : {0,1}^256,               -- H(off-chain extended metadata)
}

C-SEQ [N]: ∀ state transition: seq_after = seq_before + 1   (canonical, replaces old C-BASE-3)
C-BASE-1:  type(did) = Person ↔ owner(did) = None
C-BASE-2:  owner(did) ≠ did
```

### §2.4 Security Levels [N]

```
SecurityLevel ≜
  | Biometric_Hardware      -- Secure Enclave + biometric (PersonDID)
  | Hardware_Attestation    -- TPM/SEP quote, no biometric (DeviceDID)
  | Threshold(m, n)         -- m-of-n multisig (OrgDID, MachineDID)
  | Software_HSM            -- software key in HSM (AgentDID, ServiceDID)
  | Software                -- software key (BotDID)
  | Owner_Delegated         -- inherits owner's level (AssetDID, CharDID, ContextDID)

SecurityLevel_score(level) → ℚ ≜
  | Biometric_Hardware  → 6
  | Hardware_Attestation → 5
  | Threshold(m, n)     → 1 + 4×(m/n)    ∈ (1,5]; Threshold(n,n) = 5
  | Software_HSM        → 2
  | Software            → 1
  | Owner_Delegated     → 0
```

---

## §3 Capability and Scope System [N]

### §3.1 Capability Type

```
Capability ≜
  -- Transaction:
  | Sign_Tx(max_value: ℕ)          | Mint_Token(policy: PolicyID)
  | Burn_MAGIC(max_nanogic: ℕ)

  -- Identity:
  | Read_DID(target_type: DIDType) | Write_DID(target: DID)
  | Issue_VC(schema: ByteArray)    | Verify_Attestation
  | Read_Evidence(asset: DID, archetype_set: Option<Set<ArchetypeId>>,
                  time_window: Option<(SlotNo, SlotNo)>)
                                   -- đọc dữ-liệu-bằng-chứng (evidence record) của 1
                                   -- AssetDID, giới hạn theo tập archetype + khoảng
                                   -- slot. `ArchetypeId` là MÃ THAM CHIẾU archetype
                                   -- do RealStamp/VeData định nghĩa — PhoenixKey KHÔNG
                                   -- định nghĩa format record (ranh giới §35.2 [MN-2]).
                                   -- Enforcement lúc ĐỌC thuộc VeData; PhoenixKey chỉ
                                   -- là authority/lifecycle lúc CẤP (Op_delegate Pre).

  -- Asset:
  | Attest_Asset(class: AssetClass)
  | Transfer_Asset(asset: DID)     | Transfer_Machine(machine: DID)
  | Transfer_Char(char: DID)

  -- Control:
  | Operate_Device(device: DID)    | Operate_Machine(machine: DID)
  | Access_Service(service: DID)   | Cancel_Suspension

  -- Deployment:
  | Deploy_Agent(max_cap: Set<Capability>)
  | Deploy_Bot(max_cap: Set<Capability>)

  | Full_Authority    -- PersonDID root only; subsumes all
```

### §3.2 Scope Constraints [N]

```
C-SCOPE-1: scope(d) ⊆ scope(owner(d))    for all non-root d  [checked at creation]
C-SCOPE-2: Full_Authority ∈ scope(d) → type(d) = Person
C-SCOPE-3: scope(PersonDID) = {Full_Authority}
```

### §3.3 Capability Subsumption ⊑ [N, v4.7]

> **Errata CID-6 (v4.7).** §4.4 `DelegationToken_valid` and §23 `Op_delegate`
> historically checked `granted_cap ⊆ scope(from, s)` using plain set
> membership. Because `C-SCOPE-3` gives `scope(PersonDID) = {Full_Authority}`
> (a single opaque token, not the set of everything it grants), a PersonDID
> could never pass that check for any concrete capability:
> `{Read_DID(Asset)} ⊆ {Full_Authority}` is **False**, since `Read_DID(Asset)`
> is not literally the element `Full_Authority`. Net effect: the DID with the
> most authority (PersonDID) was the one DID type that could not delegate
> *any* specific capability through §4.4/§23 — while §4.2 `Authority` already
> special-cased `Full_Authority` via an ad-hoc disjunct (`op ∈ scope(did) ∨
> Full_Authority ∈ scope(did)`, `:367`), leaving the model internally
> inconsistent. This section fixes it by replacing plain set-⊆ with a
> subsumption (refinement) relation `⊑` at the two delegation check sites
> (§4.4, §23). Scope beyond those two sites — e.g. re-expressing §3.2
> C-SCOPE-1, §4.2 Authority's disjunct, or §19/§20 owner-scope invariants in
> terms of `⊑` — is **not** part of this fix and is left for a future spec
> pass if desired; those sites already function correctly today because they
> either special-case `Full_Authority` explicitly (§4.2) or are unaffected by
> this bug (§19/§20 do not gate PersonDID-authored delegation).

```
-- c1 ⊑ c2  ≜  "c2 subsumes c1": whoever holds c2 can do everything c1 permits.

c ⊑ Full_Authority                                  ∀ c   -- Full_Authority subsumes every cap
c ⊑ c                                                      -- reflexive

-- capability-specific refinement (parametrized capabilities only get
-- narrower under ⊑; unparametrized capabilities fall back to identity):
Read_DID(t)  ⊑ Read_DID(t)
Sign_Tx(v1)  ⊑ Sign_Tx(v2)          ⟺ v1 ≤ v2
-- all other capability constructors: c1 ⊑ c2 ⟺ c1 = c2 (identity),
-- unless a future revision defines a narrower refinement rule for them.

-- lifted to sets (used at all delegation check sites):
A ⊑ B  ≜  ∀ a ∈ A : ∃ b ∈ B : a ⊑ b     -- every cap in A is subsumed by some cap in B
```

`⊑` is a **preorder** (reflexive + transitive), which sub-delegation chains
rely on. Plain set-⊆ is the special case of `⊑` where no capability subsumes
a distinct one — so every delegation valid under the old `⊆` check remains
valid under `⊑` (backward compatible); `⊑` only *admits* new cases, via the
`c ⊑ Full_Authority` clause.

---

## §4 Authority Model [N]

### §4.1 Active Predicate

```
Active(did, s) ≜
  (revoked_slot(did) = None ∨ revoked_slot(did) > s)
  ∧ (suspended_by(did) = None
     ∨ (suspended_until(did) ≠ None ∧ suspended_until(did) < s))

-- Rotating:    Active = True  (current key valid, owner in control)
-- Transferring: Active = True  (current owner in control)
-- Suspended:   Active = False
-- Revoked:     Active = False
-- Expired (ContextDID): Active = False when s > end_slot
```

### §4.2 Authority Predicate [N]

```
-- F-47: PersonDID Privileged Operations bypass Active() check:
PersonDID_Privileged_Ops ≜ {
  TAAD_Init_Recovery,    TAAD_Cancel_Recovery,
  TAAD_Finalize_Recovery, Cancel_Suspension,
}

Authority(did: DID, op: Capability, s: SlotNo) : 𝔹 ≜

  -- F-47: Recovery ops bypass suspension:
  IF type(did) = Person ∧ op ∈ PersonDID_Privileged_Ops:
    RETURN TAAD_state_machine_valid(did, op, s)

  -- Standard path:
  Active(did, s)
  ∧ (op ∈ scope(did) ∨ Full_Authority ∈ scope(did))
  ∧ Auth_satisfied(did, op, s)
  ∧ ∀ anc ∈ Ancestors(did) : Active(anc, s)        -- lazy revocation cascade
  ∧ ∀ anc ∈ Ancestors(did) : op ∈ scope(anc)        -- F-01: runtime scope check

-- I-RESOLVER-1 [N]: Conformant did:phoenix resolver MUST set:
--   deactivated(did, s) = ¬Active(did, s) ∨ (∃ anc ∈ Ancestors(did): ¬Active(anc, s))
--   Full protocol: PhoenixKey DID Method Specification §7.
```

### §4.3 Auth_satisfied Dispatch

```
Auth_satisfied(did, op, s) ≜ case type(did) of
  | Person    → Biometric_HW_sig(did, op, s)
  | Org       → Threshold_sig(did, op, s)
  | Context   → Admin_sig(did, op, s) ∧ s ∈ [start(did), end(did)]
  | Device    → HW_Attestation(did, op, s)
  | Machine   → Stakeholder_sig(did, op, s)
  | Asset     → Owner_sig(did, op, s)
  | Bot       → API_sig(did, op, s) ∧ Rate_ok(did, s)
  | Agent     → Capability_token(did, op, s) ∧ Value_ok(did, op)
  | Service   → Service_sig(did, op, s)
  | Character → Owner_sig(did, op, s)
```

### §4.4 Delegation Token [N]

```
DelegationToken ≜ {
  from, to           : DID,
  granted_cap        : Set<Capability>,
  valid_from         : SlotNo,
  valid_until        : SlotNo,
  nonce              : {0,1}^256,   -- on-chain verified (F-22)
  signature          : {0,1}^512,
  sub_delegatable    : 𝔹,
  depth_remaining    : ℕ,
  data_scope         : Option<DataScope>,  -- [N, v4.7] chi-tiết-hoá phạm-vi cho cap
                                            -- kiểu Read_Evidence; None cho mọi cap khác.
}

DataScope ≜ {
  archetype   : Set<ArchetypeId>,   -- rỗng ⇒ mọi archetype
  time_range  : (SlotNo, SlotNo),
  asset       : Option<DID>,        -- None ⇒ áp cho asset trong granted_cap gốc
}

-- token con không được vượt phạm vi cha (archetype/time_window/asset đều phải
-- refine, không được mở rộng):
DataScope_refines(ds, asset0, arch0, tw0) ≜
  (ds.asset = Some(asset0) ∨ ds.asset = None)
  ∧ (arch0 = None ∨ ds.archetype ⊆ arch0.unwrap())
  ∧ ds.time_range ⊆ tw0

DelegationToken_valid(dt, s) ≜
  Verify(current_key(dt.from), H(dt.fields_excl_sig), dt.signature)
  ∧ s ∈ [dt.valid_from, dt.valid_until]
  ∧ dt.granted_cap ⊑ scope(dt.from, s)        -- F-37: CURRENT scope at validation time
                                               -- v4.7 (Errata CID-6): ⊆ → ⊑ (§3.3) so a
                                               -- PersonDID (scope={Full_Authority}) can
                                               -- delegate a concrete capability; ⊆ was
                                               -- always False here since Full_Authority
                                               -- is a single opaque token, not the set of
                                               -- everything it grants.
  ∧ dt.depth_remaining ≤ MAX_DELEGATION_DEPTH - depth(dt.from)
  ∧ nonce_not_used_on_chain(dt.nonce, dt.from)
  ∧ (dt.data_scope ≠ None →                   -- [N, v4.7] nếu có data_scope thì nó
        ∃ Read_Evidence(a, as, tw) ∈ effective_caps(dt.from, s):  -- phải refine cap gốc
            DataScope_refines(dt.data_scope, a, as, tw))

-- Ai kiểm token khi ĐỌC = VeData (bên đọc evidence), dùng DelegationToken_valid +
-- DataScope_refines của PhoenixKey. PhoenixKey kiểm khi CẤP (Op_delegate Pre) và là
-- nguồn trạng-thái revoke. Custodian §17 (default {Read_DID}) có thể nâng lên
-- {Read_DID, Read_Evidence(...)} khi cần — quyết định riêng, không bắt buộc.

-- F-37 note: scope(dt.from, s) read via CIP-0031 Reference Input.
-- Evaluated at block-start UTxO set; intra-block scope changes do not affect
-- same-block delegation validation (deliberate for determinism, CIP-0031 §3.2).
```

---

## §5 Lifecycle Model [N]

### §5.1 DID State Machine

```
DIDState ≜
  | Active
  | Rotating(pending_key: Ed25519PubKey, deadline: SlotNo, tier: ℕ)
  | Transferring(pending_owner: DID, deadline: SlotNo)
  | Suspended(by: SuspendedByInfo, until: Option<SlotNo>)
  | Revoked(at: SlotNo, by: DID)
  | Expired    -- ContextDID only

-- Op_suspend (errata E-02 split):
Op_suspend(did: DID where type ≠ Person, until, by, s):
  Requires: Auth_satisfied(owner(did), Write_DID(did), s)
  Effect:   suspended_by ← Suspender(by); suspended_until ← until; seq++

Op_suspend(did: PersonDID, until, s):
  Requires:
    let S = {g ∈ guardians(did) | g.suspension_eligible = True}
    |{sigs from S}| ≥ max(policy.threshold + 1, ⌈|S|/2⌉ + 1)
  Effect: suspended_by ← GuardianCoalition(H(S_signers ++ encode(s)))
          suspended_until ← until; seq++

-- Op_unsuspend (errata E-02 split — PersonDID has no owner):
Op_unsuspend(did: PersonDID, s):
  Requires: TAAD_state_machine_valid(did, Cancel_Suspension, s)
  Effect:   suspended_by ← None; suspended_until ← None; seq++

Op_unsuspend(did: DID where type ≠ Person, s):
  Requires: TAAD_state_machine_valid(did, Cancel_Suspension, s)
            ∨ Auth_satisfied(owner(did), Cancel_Suspension, s)
  Effect:   suspended_by ← None; suspended_until ← None; seq++
```

### §5.2 Type-Specific Lifecycle

| DIDType | Rotatable | Transferable | Expires | Suspended | Cascade | End-of-Life *(v4.3)* |
|---------|-----------|--------------|---------|-----------|---------|---------------------|
| Person | ✓ (TAAD) | ✗ | ✗ | ✓ guardian | source | Posthumous §32.1 / Incapacity §32.2 |
| Org | ✓ | ✗ | ✗ | ✓ owner | both | Dissolution §32.3 |
| Context | ✗ | ✗ | ✓ | ✓ admin | sink | Auto-dissolve at end_slot |
| Device | ✓ owner | ✗ | ✗ | ✓ owner | sink | Decommission §32.5 |
| Machine | ✓ | ✓ | ✗ | ✓ owner | both | Destruction §32.4 |
| Asset | ✓ owner | ✓ | ✗ | ✗ | sink | Destruction §32.4 |
| Bot | ✓ owner | ✗ | ✗ | ✓ owner | sink | Revoke |
| Agent | ✓ owner | ✗ | ✗ | ✓ owner | sink | Revoke |
| Service | ✓ owner | ✗ | ✗ | ✓ owner | both | Revoke / Archive |
| Character | ✓ owner | ✓ | ✗ | ✗ | sink | Burn NFT |

---

# PART II — CORE CRYPTOGRAPHY

## §6 Key Hierarchy and Master Credential [N]

### §6.1 Master Key (Root Secret)

```
-- Mode A — Generate (default, maximum security):
Master_KEK ← SecureEnclave.Generate()   ∈ {0,1}^256
-- 256 bits hardware random; never exposed in plaintext outside Secure Enclave.

-- Mode B — Import (from Yoroi, Eternl, or any BIP-39 wallet):
user_24_words : List<Word>              -- user-provided BIP-39 mnemonic
Master_KEK = BIP39_ToEntropy(user_24_words)   ∈ {0,1}^256
-- Security note: entropy depends on original wallet's key generation.
-- See I-PERSON-6 for scope restrictions on import_mode PersonDIDs.

-- In both modes: Master_KEK serves as the Cardano HD wallet seed.
-- 24-word export: BIP39_Encode(Master_KEK)  (§9)
```

### §6.2 Device Encryption Key

```
-- Cloud_Secret: high-entropy random, stored in hardware-backed Keychain:
Cloud_Secret ← SecureRandom(256)   -- generated once per device
-- Storage: iOS Keychain (hardware-backed, NOT iCloud sync), Android Keystore
-- NOT derived from user password; NOT stored in cloud

HW_UID = SecureEnclave.HardwareIdentifier()   -- device-bound, non-copyable

-- Device_KEK: encrypts Master_KEK for local storage and LampNet distribution:
Device_KEK = HKDF(
  key  = Cloud_Secret ∥ HW_UID,
  info = "device-kek-v1",
  salt = H(DID ∥ "device-kek-salt-v1")   -- fix M03: DID-specific salt
)   → {0,1}^256
```

### §6.3 TAAD and Hardware Keys

```
-- TAAD Controller Key (recoverable from Master_KEK):
TAAD_Key = Ed25519.FromSeed(
  HKDF(Master_KEK, "taad-controller-v1", H(DID))
)
controller_pkh = Blake2b224(TAAD_Key.pub)   -- on-chain TAAD UTxO controller

-- Hardware Signing Key (device-bound, biometric-gated):
HW_Key = SecureEnclave.Generate()          -- never exported
hw_pub = HW_Key.publicKey()                -- stored in PersonDID datum

-- Two-Key Architecture (fix M04):
-- HW_Key:   daily biometric signing; NOT recoverable; tied to device hardware.
-- TAAD_Key: governance and recovery; derived from Master_KEK; survives device loss.
-- Recovery restores Master_KEK → TAAD_Key re-derived unchanged → no TAAD rotate needed.
```

### §6.4 Encrypted Seed Construction

```
EncSeed = Enc(Device_KEK, Master_KEK ∥ H(DID))   -- authenticated encryption

-- EncSeed is the artifact distributed to LampNet (§7).
-- Properties:
--   Master_KEK never leaves device in plaintext.
--   EncSeed is meaningless without Device_KEK.
--   Device_KEK requires Cloud_Secret (hardware Keychain) + HW_UID.
```

### §6.5 Key Security Properties

| Key | Entropy | Storage | Exportable | Recovery Path |
|-----|---------|---------|------------|---------------|
| Master_KEK | 256 bits | Device (encrypted) | Yes (24 words, explicit consent) | Guardian TSS + TAAD |
| Cloud_Secret | 256 bits | Hardware Keychain | No | Guardian-encrypted share |
| Device_KEK | derived | In-memory only | No | Re-derive after recovery |
| TAAD_Key | derived | Derived at use | No | Re-derive from Master_KEK |
| HW_Key | 256 bits | Secure Enclave | No | Generate fresh on new device |

### §6.6 Guardian-Based Master_KEK Protection

```
-- Master_KEK is split across guardians for recovery:
k = policy.threshold          -- minimum shares for reconstruction
n = |guardians|               -- total shares

shares = TSS_Share(Master_KEK, k, n)   -- Shamir k-of-n

∀ i ∈ [0, n):
  share_encrypted_i = Enc(guardian_i.public_key, shares[i])
  -- encrypted under guardian's Ed25519 public key via ECIES
  -- stored in recovery credential (§10) and distributed to guardians

Recovery reconstruction:
  collect {share_i : i ∈ S} where |S| ≥ k
  Master_KEK = TSS_Reconstruct({Dec(guardian_i.private_key, share_encrypted_i) : i ∈ S}, k)
```

---

## §7 LampNet Distributed Storage [N]

### §7.1 LT Code Encoding

```
-- Rateless Luby Transform fountain code:
k   = 50    -- minimum droplets for recovery
n   = 1000  -- droplets generated per EncSeed

Droplet ≜ (index: ℕ, data: ByteArray)

LT_Encode(data: ByteArray, k, n) → List<Droplet>
  -- Produces n Droplets; any k+ε sufficient for reconstruction.
  -- Degree distribution: Robust Soliton with parameters μ = k, δ = 0.01.

LT_Decode(droplets: List<Droplet>, k) → ByteArray | ⊥
  -- Gaussian elimination; ⊥ if |droplets| < k or inconsistent.
```

### §7.2 Storage and Retrieval

```
-- LocatorSecret: deterministic per (device, DID) pair:
LocatorSecret = HKDF(HW_UID ∥ DID, "lampnet-locator-v1")
-- NOT stored in cloud; re-derived from hardware UID + DID.
-- During recovery: included (encrypted) in recovery credential (§10.3).

-- Upload (at registration):
Droplets = LT_Encode(EncSeed, k=50, n=1000)
Upload(Droplets, LocatorSecret) → LampNet   -- distributed to available nodes

-- Retrieval (on same device):
raw_droplets = Download(LocatorSecret, min_count=60)   -- request 60 for margin
EncSeed      = LT_Decode(raw_droplets, k=50)
Master_KEK   = Dec(Device_KEK, EncSeed)

-- Recovery on new device (§11 full protocol):
-- LocatorSecret recovered from recovery credential after TAAD verification.
-- New Device_KEK derived with new Cloud_Secret + new HW_UID.
-- New EncSeed = Enc(new_Device_KEK, Master_KEK) re-uploaded.

I-LAMP-1: |Droplets| = 1000 at upload time
I-LAMP-2: LT_Decode(LT_Encode(data, k, n), k) = data  (correctness)
I-LAMP-3: LocatorSecret never transmitted in plaintext
```

---

## §8 Cardano Wallet Integration [N]

### §8.1 Key Derivation (CIP-1852)

```
-- Standard Cardano HD wallet derivation from Master_KEK:
-- Path: m / purpose' / coin_type' / account' / role / index

-- Payment keys (external chain):
payment_key(account, index) = BIP32_Derive(Master_KEK, [1852h, 1815h, account, 0, index])

-- Staking key:
staking_key(account) = BIP32_Derive(Master_KEK, [1852h, 1815h, account, 2, 0])

-- Standard enterprise address (no staking):
address_enterprise(account, index) =
  Bech32m("addr", Blake2b224(payment_key(account, index).pub) ∥ 0x60)

-- Base address (with staking):
address_base(account, index) =
  Bech32m("addr",
    Blake2b224(payment_key(account, index).pub) ∥
    Blake2b224(staking_key(account).pub) ∥ 0x00)
```

### §8.2 DID-Wallet Relationship

```
-- PhoenixKey DID and Cardano wallet are derived from the same Master_KEK:
DID = DID_construct(Person, None, registration_slot)      -- §2.1
SeedData = Master_KEK                                     -- same bits
24_words = BIP39_Encode(Master_KEK)                       -- human-readable form

-- Compatibility proof (§9.2): exporting 24 words and importing into Eternl/Yoroi/Lace
-- yields the exact same wallet addresses → no trust assumption needed.
```

---

## §9 Seed Phrase Export and Import [N]

### §9.1 Export Protocol (24 words)

```
-- §6.1: SeedData = Master_KEK. Export = BIP39_Encode(Master_KEK).

Op_export_seed(did: PersonDID, s: SlotNo):
  Pre:
    Biometric_HW_sig(did, Export_Seed_Cap, s)   -- biometric required
    ∧ Consent_confirmed(did, [security_warning_1, security_warning_2])
  Effect:
    words = BIP39_Encode(Master_KEK)             -- 24 BIP-39 words
    seed_exported ← s                            -- recorded on-chain
    Display(words, blur=True, timer=30s)          -- secure display
    security_note on-chain: "seed exported at slot s; hardware model degraded"

I-EXPORT-1: Exporting seed does not reveal HW_Key, Device_KEK, or TAAD_Key.
I-EXPORT-2: Externally used seed detected (monitoring) → proactive rotation notification.
I-EXPORT-3: Hardware security model degrades to software-level for the exported mnemonic only.
            PhoenixKey signing path continues to use HW_Key (unaffected).
```

### §9.2 Import Protocol (from external wallet)

```
-- Import: user provides existing 24-word mnemonic from Yoroi, Eternl, etc.

Op_import_seed(user_24_words: List<Word>, s: SlotNo) → PersonDID:
  -- Step 1: Validate BIP-39 checksum
  Master_KEK = BIP39_ToEntropy(user_24_words)
  assert BIP39_Verify(user_24_words)

  -- Step 2: Derive Cardano spending key and construct ownership proof
  spend_key = BIP32_Derive(Master_KEK, [1852h, 1815h, 0h, 0, 0])
  challenge = H("phoenixkey-import-v1" ∥ DID_new ∥ nonce_server ∥ encode(s))
  proof     = Sign(spend_key, challenge)

  -- Step 3: Verify proof and on-chain history
  assert Verify(spend_key.pub, challenge, proof)
  address = address_base(0, 0)           -- first address from seed
  assert on_chain_tx_history(address) ≠ ∅   -- proves real wallet ownership

  -- Step 4: Wrap in PhoenixKey security (same path as generate mode)
  Cloud_Secret ← SecureRandom(256)
  Device_KEK   = HKDF(Cloud_Secret ∥ HW_UID, "device-kek-v1", H(DID_new))
  EncSeed      = Enc(Device_KEK, Master_KEK ∥ H(DID_new))
  Droplets     = LT_Encode(EncSeed, k=50, n=1000)
  Upload(Droplets, LocatorSecret)

  -- TAAD_Key independently generated (not from imported seed):
  TAAD_Key = Ed25519.FromSeed(HKDF(Master_KEK, "taad-controller-v1", H(DID_new)))
  HW_Key   = SecureEnclave.Generate()
  RETURN create_PersonDID(DID_new, hw_pub, taad_key, import_mode=True)

I-IMPORT-1: Imported wallet retains same Cardano address → no fund migration needed.
I-IMPORT-2: import_mode = True → restrictions on high-stakes OrgDID admin (I-PERSON-6).
I-IMPORT-3: nonce_server fresh per session (prevents replay of ownership proof).
```

---

# PART III — TAAD PROTOCOL

## §10 TAAD State Machine [N]

### §10.1 TAAD UTxO Structure

```
-- TAAD (Token-Anchored Authority Delegation) is an on-chain UTxO state machine.
-- One TAAD UTxO per PersonDID; address = Script(TAAD_Validator).

-- v4.5: SecondaryWallet commitment type [N]
SecondaryWallet ≜ {
  wallet_pkh_commitment : {0,1}^256,  -- BLAKE2b-256(pkh ∥ did ∥ enrolled_slot)
  enrolled_slot         : SlotNo,
}
-- Commitment binds the specific wallet to this DID at enrollment time.
-- Pre-image (pkh, did, enrolled_slot) revealed at Tier 1 recovery.
-- Validator verifies: BLAKE2b-256(pkh ∥ did ∥ slot) = commitment
--                     AND recovery tx signed by pkh.

TAADDatum ≜ {
  did                : DID,
  controller_pkh     : {0,1}^224,
  hw_pub             : Ed25519PubKey,
  seq                : ℕ,
  guardians          : List<GuardianConfig>,
  state              : TAADState,
  seed_exported      : Option<SlotNo>,
  import_mode        : 𝔹,
  suspended_by       : Option<SuspendedByInfo>,
  suspended_until    : Option<SlotNo>,
  -- v4.3:
  jurisdiction_flags : Set<JurisdictionCode>,
  guardian_did       : Option<DID>,
  estate_config      : Option<EstateConfig>,
  -- v4.4:
  knowledge_factors  : Option<KnowledgeFactors>,
  -- v4.5:
  secondary_wallets  : List<SecondaryWallet>,    -- replaces secondary_wallet_enrolled:𝔹
  backup_deadline    : Option<SlotNo>,           -- None = backup configured; Some(s) = grace period until s
}

TAADState ≜
  | Active
  | Recovering {
      pending_hw_pub     : Ed25519PubKey,
      recovery_deadline  : SlotNo,
      verification_tier  : ℕ,      -- 1 | 2 | 3 | 5 | POSTHUMOUS_TIER
      crypto_proofs      : ByteArray,
      cancel_nonce       : {0,1}^256,
    }
  | Posthumous {
      beneficiary    : DID,
      confirmed_slot : SlotNo,
    }
  | Incapacitated {
      config         : IncapacityConfig,
      declared_slot  : SlotNo,
    }
  | Archived       -- terminal: OrgDID dissolved; non-Person DID orphaned; write-blocked

POSTHUMOUS_TIER : ℕ ≜ 100  -- sentinel value for Posthumous recovery path; distinct from
                            -- capability tiers (1–4 in §3) and recovery tiers (1,2,3,5)
```

### §10.2 TAAD Transitions [N]

```
-- Transition: Active → Recovering (Init):
TAAD_init_recovery(did, new_hw_pub, tier, proofs, s):
  Pre:
    tier ∈ {1, 2, 3, 5}
    ∧ verify_tier_proofs(tier, proofs, did, new_hw_pub, s)   -- §11
    ∧ TAADDatum.state = Active
    ∧ seq_in_tx = TAADDatum.seq + 1
  Effect: TAADDatum.state ← Recovering {
    pending_hw_pub    = new_hw_pub,
    recovery_deadline = s + TAAD_TIMELOCK[tier],
    verification_tier = tier,
    crypto_proofs     = proofs,
    cancel_nonce      = H(proofs ∥ encode(s)),
  }

-- Transition: Recovering → Active (Cancel):
-- DISPATCH: if verification_tier = 5, use TAAD_cancel_tier5 (§11.7) instead.
-- TAAD_cancel_tier5 does NOT require hw_pub; uses email token or higher-tier proof.
TAAD_cancel_recovery(did, cancel_sig, s):
  Pre:
    TAADDatum.state = Recovering
    ∧ state.verification_tier ≠ 5    -- Tier 5 uses TAAD_cancel_tier5 (§11.7)
    ∧ state.verification_tier ≠ POSTHUMOUS_TIER  -- Posthumous: no cancel by owner
    ∧ Verify(TAADDatum.hw_pub, H("cancel" ∥ encode(s) ∥ encode(TAADDatum.seq)), cancel_sig)
    ∧ s < state.recovery_deadline   -- within timelock
  Effect:
    state ← Active; seq ← seq + 1
    -- Cancel is valid even if PersonDID is Suspended (bypass Active())

-- Transition: Recovering → Active (Finalize):
TAAD_finalize_recovery(did, finalize_sig, s):
  Pre:
    TAADDatum.state = Recovering
    ∧ s ≥ state.recovery_deadline   -- timelock expired
    ∧ Verify(state.pending_hw_pub, H("finalize" ∥ encode(s) ∥ encode(TAADDatum.seq)), finalize_sig)
  Effect:
    hw_pub        ← state.pending_hw_pub
    controller_pkh ← Blake2b224(TAAD_Key_new.pub)  -- re-derived from Master_KEK
    state         ← Active
    seq           ← seq + 1

TAAD_TIMELOCK : Map<ℕ, ℕ> ≜ {1 → TAAD_TIMELOCK_T1, 2 → TAAD_TIMELOCK_T2, 3 → TAAD_TIMELOCK_T3, 5 → TIER5_TIMELOCK}

-- Replay prevention: C-SEQ enforced; any tx with seq ≤ on-chain seq is rejected.
```

### §10.4 TAAD ASM Invariants [N]

The following invariants are normative constraints on every valid TAAD UTxO and every TAAD transition. These are part of the **Fully Open** protocol (see §31).

```
-- A-1 (Singleton): Exactly one TAAD UTxO per active PersonDID at all times.
A-1: ∀ did: PersonDID, ∀ s: SlotNo:
  |{utxo ∈ UTxO_set(s) | utxo.datum.did = did ∧ utxo.address = Script(TAAD_Validator)}| = 1

-- A-2 (Sequence Monotonicity): seq strictly increases across all TAAD transitions.
A-2: ∀ tx spending TAAD UTxO: seq_output = seq_input + 1
  [This is C-SEQ applied to TAAD; restated here for explicit TAAD scope.]

-- A-3 (Address Enforcement): TAAD UTxO must reside at the TAAD validator script address.
A-3: ∀ TAAD UTxO output u: u.address = Script(TAAD_Validator)
  -- Prevents TAAD token from being sent to arbitrary addresses.
  -- Enforced by Cardano native token / minting policy binding.

-- A-4 (State Exclusivity): TAAD UTxO is in exactly one defined TAADState at all times.
A-4: ∀ did: PersonDID, ∀ s: SlotNo:
  state(TAAD(did), s) ∈ {Active, Recovering, Posthumous, Incapacitated, Archived}
  -- No undefined intermediate states. Transitions are atomic (EUTxO).
  -- Terminal states: Posthumous (write-blocked, beneficiary access), Archived (read-only).

-- A-5 (Collateral Lock): Guardian recovery_collateral is locked from Tier2/3 init until
--                         finalize or cancel. No partial withdrawal during recovery.
A-5: ∀ TAAD UTxO in state Recovering, verification_tier ∈ {2, 3}:
  ∀ guardian g ∈ signers(crypto_proofs):
    g.recovery_collateral_utxo remains unspent until state returns to Active

-- A-6 (Cancel Window): TAAD_Cancel_Recovery is only valid strictly before the deadline.
A-6: TAAD_cancel_recovery valid at slot s →
  s < recovery_deadline ∧ TAADDatum.state = Recovering
  -- After deadline, only Finalize is valid (owner had their chance to cancel).
```

**Invariant interaction with F-47:** A-2 ensures seq increments on all state changes including suspension. A-4 and A-6 together ensure that Cancel_Recovery remains available (bypasses Active()) regardless of suspension, as long as the timelock window is open.

### §10.5 TAAD State Machine Predicate [N]

```
-- Errata E-03: formal definition of TAAD_state_machine_valid (referenced throughout):
TAAD_state_machine_valid(did: PersonDID, op: Capability, s: SlotNo) : 𝔹 ≜
  case op of
  | TAAD_Init_Recovery →
      TAADDatum.state = Active
      ∧ verify_tier_proofs(selected_tier, proofs, did, new_hw_pub, s)
      ∧ seq_in_tx = TAADDatum.seq + 1

  | TAAD_Cancel_Recovery →
      TAADDatum.state = Recovering
      ∧ Verify(TAADDatum.hw_pub, H("cancel" ∥ encode(s) ∥ encode(seq)), sig)
      -- Bypasses Active(); valid even while Suspended.

  | TAAD_Finalize_Recovery →
      TAADDatum.state = Recovering
      ∧ s ≥ state.recovery_deadline
      ∧ Verify(state.pending_hw_pub, H("finalize" ∥ encode(s) ∥ encode(seq)), sig)

  | Cancel_Suspension →
      Verify(TAADDatum.hw_pub, H("cancel-suspension" ∥ encode(s) ∥ encode(seq)), sig)
      -- hw_pub signature only; guardians not required.
      -- Bypasses Active(); valid even while Suspended.

-- Full TAAD validator logic: PhoenixKey DID Method Specification §5.
-- This spec defines the predicate signature; implementation in Aiken/PlutusV3.
```

---

## §11 Recovery Protocol — module Rebirthme [N]

§11 defines the tiered recovery protocol for PersonDID holders who have lost access to their device. The protocol is TAAD-based (§10) and applies exclusively to PersonDID. Recovery tiers are ordered from strongest to weakest; the cascade (§11.1) tries each tier in order. Tier 4 (Posthumous, §32.1) is a special administrative path triggered by a government OrgDID death certificate, not by the user.

## §11.0 Recovery Scope and Mandatory Backup [N] *(v4.4, revised v4.5)*

### §11.0.1 TAAD Scope — PersonDID Only

```
§11 Recovery Protocol applies EXCLUSIVELY to PersonDID.
All PersonDID recovery tiers (§11.1–§11.6) assume a TAAD UTxO exists.

Non-Person DID recovery is NOT governed by §11.
```

### §11.0.2 Non-Person DID Recovery

```
-- For all DID types where type ≠ Person:
-- Key recovery = key rotation executed by the active owner.
-- No TAAD UTxO. No timelock. No guardian collateral.

NonPerson_Recover(did: D where type(D) ≠ Person, new_pub, s):
  Pre:  Authority(owner(did), Write_DID(did), s)
  Effect: same as Op_rotate(did, new_pub, s) per §5.1
  Security: inherits owner's security level

I-RECOVERY-1 [N]: Non-Person DID MUST NOT have a TAAD UTxO.
I-RECOVERY-2 [N]: Non-Person DID rotation is O(1), no timelock.
                  Owner authority (§4.2) is the sole gate.
```

### §11.0.3 Orphaned Non-Person DID

```
-- A non-Person DID becomes orphaned when its owner is:
--   (a) PersonDID in Posthumous state (§32.1) — write-blocked
--   (b) PersonDID in Incapacitated state (§32.2) — scope limited
--   (c) OrgDID in Dissolved/Archived state (§32.3) — write-blocked

Orphan_recover(did: D, s):
  Case (a) Posthumous owner:
    beneficiary = estate_config.beneficiary_did
    IF Write_DID(did) ∈ estate_config.scope_granted:
      Authority(beneficiary, Write_DID(did), s) → NonPerson_Recover
    ELSE: did transitions to Archived state (records preserved, no new ops)

  Case (b) Incapacitated owner:
    conservator = IncapacityConfig.conservator_did
    IF Write_DID(did) ∉ IncapacityConfig.conservator_scope:
      did suspended until incapacity resolved (§32.2 auto-expiry)
    ELSE: conservator executes NonPerson_Recover

  Case (c) Dissolved OrgDID owner:
    OrgDID dissolution (§32.3) MUST transfer or revoke all child DIDs
    before Op_complete_dissolution is valid (I-DISS-4 enforcement)
    If missed: did transitions to Archived state

I-RECOVERY-3 [N]: estate_config.scope_granted for Posthumous owners SHOULD
                  include Write_DID for critical operational non-Person DIDs.
                  PhoenixKey app MUST prompt user to configure this at
                  estate setup time (§32.1).

I-RECOVERY-5 [N]: Recursive orphan prevention.
  If beneficiary_did itself is in state Posthumous, Incapacitated, or Archived
  at the time Op_finalize_posthumous executes:
    did → Archived immediately.
  No recursive orphan chain is permitted.
  Rationale: simultaneous deaths (accident, disaster) must not create
  unresolvable circular ownership states on-chain.
```

### §11.0.4 Biometric Is Not a Recovery Mechanism [N]

```
-- Biometric (face, fingerprint, iris) in PhoenixKey serves ONE purpose:
-- gating HW_Key on device for daily authentication (§6.3).
-- Biometric is NOT used in any recovery tier (§11.1–§11.6).

Property: permanent biometric loss does NOT block TAAD recovery
          if any Tier 1–5 factor is enrolled.

Counter-property: if NO recovery factor is enrolled,
                  device loss + biometric loss = DID permanently unrecoverable.
                  This is a user-configuration failure, not a protocol flaw.
                  No self-sovereign identity system can recover an identity
                  without at least one pre-registered recovery factor.

I-RECOVERY-4 [N]: Mandatory backup — grace period model (v4.5 revised).

  At PersonDID registration:
    IF secondary_wallets = [] AND |guardians| < 2 AND knowledge_factors = None:
      backup_deadline = created_slot + 30 × 432_000  -- ~30 days grace period
    ELSE:
      backup_deadline = None  -- backup already configured; no restriction

  Validator enforces — when s > backup_deadline AND backup_deadline ≠ None
  AND secondary_wallets = [] AND |guardians| < 2 AND knowledge_factors = None:
    HIGH_STAKES_OPS BLOCKED:
      Sign_Tx(v > HIGH_STAKES_THRESHOLD), Transfer_Asset,
      Transfer_Machine, Mint_Token, Deploy_Agent(cap)
    PERMITTED OPS (no restriction):
      Read_DID, Issue_VC, Verify_Attestation, Access_Service,
      TAAD recovery, Cancel_Suspension

  HIGH_STAKES_THRESHOLD = 100 ADA equivalent (governance parameter)

  Rationale: hard block at registration increases onboarding friction
  by ~5-10 minutes (mobile UX data: 20-40% conversion drop).
  Grace period allows user to start using DID immediately while
  creating a bounded window to complete backup setup.
  After deadline, only high-stakes ops are restricted — DID remains
  functional for everyday use, preventing total lockout.
```

---

### §11.1 Tier Selection

```
select_recovery_tier(did: PersonDID, new_hw_pub, s):
  datum            = TAADDatum(did)         -- on-chain TAAD datum
  secondary_wallets = datum.secondary_wallets  -- v4.5: commitment list
  guardians        = datum.guardians
  threshold        = MIN_GUARDIAN_THRESHOLD    -- default 2; user may configure higher

  -- Tier 1: secondary wallet key possession
  IF |secondary_wallets| ≥ 1:
    RETURN tier_1_recovery(did, secondary_wallets, new_hw_pub, s)

  -- Tier 2: guardian quorum (standard threshold)
  IF |{g ∈ guardians : 2 ∈ g.tier_eligible}| ≥ threshold:
    RETURN tier_2_recovery(did, guardians, new_hw_pub, s)

  -- Tier 3: guardian quorum (elevated threshold)
  IF |{g ∈ guardians : 3 ∈ g.tier_eligible}| ≥ threshold + 1:
    RETURN tier_3_recovery(did, guardians, new_hw_pub, s)

  -- Tier 5: knowledge-based fallback (weakest; longest timelock)
  IF datum.knowledge_factors ≠ None:
    RETURN tier_5_recovery(did, datum.knowledge_factors, new_hw_pub, s)

  RETURN recovery_not_possible()
  -- recovery_not_possible() → DID is unrecoverable without external action.
  -- Posthumous recovery (§32.1) requires death certificate from government OrgDID.
```

### §11.2 Tier 1 — Secondary Wallet Verification

```
tier_1_recovery(did, candidates, new_hw_pub, s):
  -- v4.5: candidates come from TAADDatum.secondary_wallets list
  --        (not from analyze_tx_graph which was underspecified)
  enrolled = TAADDatum.secondary_wallets
  assert |enrolled| ≥ 1

  -- User provides: pkh, enrolled_slot, signature
  pkh          = redeemer.secondary_pkh
  enroll_slot  = redeemer.enrolled_slot

  -- On-chain verification (validator):
  commitment = BLAKE2b-256(pkh ∥ did ∥ enroll_slot)
  assert ∃ w ∈ enrolled : w.wallet_pkh_commitment = commitment
                         ∧ w.enrolled_slot = enroll_slot
  -- Recovery tx must be signed by pkh:
  assert pkh ∈ tx.extra_signatories

  RETURN (tier=1, proof=serialize({pkh, enroll_slot, commitment}))
```

### §11.3 Tier 2 — Guardian Threshold

```
GuardianConfig ≜ {
  guardian_did          : DID,
  guardian_pub          : Ed25519PubKey,
  recovery_collateral   : ℕ,           -- MIN_GUARDIAN_COLLATERAL = 50 ADA
  suspension_collateral : ℕ,           -- F-47: separate from recovery
  tier_eligible         : Set<ℕ>,
  suspension_eligible   : 𝔹,           -- F-47: explicit opt-in, default False
}

-- F-39: Formal guardian slashing:
Slash(guardian: GuardianConfig, victim: PersonDID, evidence: ByteArray, s: SlotNo):
  Pre:
    evidence_proves_fraudulent_action(evidence, guardian.guardian_did, s)
    ∧ guardian.recovery_collateral ≥ MIN_GUARDIAN_COLLATERAL
  Effect:
    confiscate(guardian.recovery_collateral) → victim_or_beneficiary
    guardian.tier_eligible ← ∅          -- permanent disqualification
    slash_record on-chain: (guardian.did, s, evidence_hash)

evidence_proves_fraudulent_action(evidence, guardian_did, s) ≜
  evidence contains on-chain proof of:
    | FraudulentRecoverySig   -- guardian signed fraudulent recovery Tx
    | FraudulentSuspensionSig -- guardian signed fraudulent suspension
    | DoubleSignedRecovery    -- guardian signed conflicting recovery requests
```

### §11.3.1 Guardian Incentive Model — Nash Equilibrium [N]

The collateral mechanism creates a dominant strategy equilibrium where honest behavior is rational. Agreed in FormalSpec v3.0 §MT10.

```
-- Parameters:
V         : ℕ      -- value at stake (lovelace in protected assets)
m         : ℕ      -- number of guardians in threshold set
C*        : ℕ      -- required collateral per guardian (in MAGIC equivalent)
p_absent  : ℚ      -- probability guardian is unavailable at signing time
p_present : ℚ      -- 1 - p_absent

-- Guardian expected payoff analysis:
E[payoff_malicious] = p_absent × (V / m) - p_present × C*
-- (Expected gain from undetected fraud) - (Expected loss when caught)

-- Nash equilibrium condition (fraud never profitable):
C* ≥ (p_absent / p_present) × (V / m) × 1.2    -- 1.2 = 20% safety margin

-- Alert incentive (F-39 extension): first honest guardian who alerts owner
-- when others attempt fraudulent recovery receives:
Alert_reward = 2% × confiscated_collateral
-- This makes detection profitable, creating a dominant strategy for honest guardians.

-- Constraint:
I-GUARDIAN-NASH: MIN_GUARDIAN_COLLATERAL ≥ C*(V_typical, m, λ, m)
  where V_typical = typical assets under management for target user segment
  -- Concrete: MIN_GUARDIAN_COLLATERAL = 50 ADA covers typical DeFi holdings.
  -- High-value users should configure higher collateral via policy.threshold.
```

```
tier_2_recovery(did, guardians, new_hw_pub, s):
  challenge = H("tier2" ∥ did ∥ new_hw_pub ∥ nonce ∥ encode(s))
  guardian_sigs = collect_guardian_signatures(guardians, challenge, timeout=24h)
  assert |guardian_sigs| ≥ policy.threshold
  ∀ sig ∈ guardian_sigs: Verify(sig.guardian_pub, challenge, sig.signature)
  -- Out-of-band identity verification by guardians required before signing.
  RETURN (tier=2, proof=serialize(guardian_sigs))
```

### §11.4 Post-Recovery State Update

```
-- After TAAD_finalize_recovery:
1. New Master_KEK: recovered via TSS from guardians (§6.6) OR transferred
   from recovery credential (encrypted under old TAAD_Key, decryptable by tier proofs).
2. On new device: new_Cloud_Secret ← SecureRandom(256), new_HW_UID ← device hardware ID
   New Device_KEK = HKDF(new_Cloud_Secret ∥ new_HW_UID, "device-kek-v1", H(DID))
3. New EncSeed = Enc(new_Device_KEK, Master_KEK ∥ H(DID))  → LampNet upload
4. TAAD_Key re-derived from Master_KEK (same as before → controller_pkh unchanged)
5. New HW_Key generated in new device's Secure Enclave
6. PersonDID datum updated: hw_pub ← new_hw_pub; seq++

-- Identity continuity: DID unchanged, controller_pkh unchanged, Cardano addresses unchanged.
-- No fund transfers needed; no external party notification needed.
```

### §11.5 Environmental Attestation (VeData Integration)

```
-- VeData is an external context oracle service. PhoenixKey delegates all environmental
-- sensing to VeData via a well-defined API boundary. VeData internals are proprietary (§31).

-- Proof request structure (PhoenixKey → VeData):
ProofRequest ≜ {
  challenge_type        : 'recovery_attestation',
  historical_fingerprint: HistoricalFingerprint,
  required_signals      : {BLE_Beacons, WiFi_Networks, ThermalSignature, CoarseLocation},
  tolerance_level       : ℚ,           -- minimum confidence score; default 0.7
  verification_mode     : SilentBackground,
  nonce                 : {0,1}^256,   -- fresh per request; prevents replay
}

-- VeData response:
VeData_attest(request: ProofRequest, s) → ContextProof | ⊥

ContextProof ≜ {
  confidence_score   : ℚ ∈ [0,1],
  timestamp          : SlotNo,
  oracle_signature   : {0,1}^512,   -- Sign(VEDATA_ORACLE_KEY, H(proof_fields))
  referenced_fp_hash : {0,1}^256,   -- H(request.historical_fingerprint)
  nonce_echo         : {0,1}^256,   -- echoes request.nonce; anti-replay
}

-- PhoenixKey verification (no environmental sensing code in PhoenixKey):
verify_context_proof(proof, request, s) ≜
  Verify(VEDATA_ORACLE_PK, H(proof.fields_excl_sig), proof.oracle_signature)
  ∧ proof.confidence_score ≥ request.tolerance_level
  ∧ |proof.timestamp - s| ≤ 300             -- within 5 slots (~100 seconds)
  ∧ proof.referenced_fp_hash = H(request.historical_fingerprint)
  ∧ proof.nonce_echo = request.nonce        -- prevents replay attack

-- Historical fingerprint (stored in recovery credential at registration):
HistoricalFingerprint ≜ {
  ble_beacon_hash  : {0,1}^256,   -- H(set of detected BLE beacon identifiers)
  wifi_hash        : {0,1}^256,   -- H(set of WiFi SSIDs/BSSIDs)
  thermal_hash     : {0,1}^256,   -- H(device temperature pattern)
  location_zone    : {0,1}^256,   -- H(coarse GPS zone; quantized to 10km cell)
}
-- Raw signals never stored; only hashes. Privacy-preserving.

I-VEDATA-1: PhoenixKey MUST NOT implement SignalType collection code.
I-VEDATA-2: PhoenixKey MUST NOT request OS sensor permissions directly.
I-VEDATA-3: All VeData interaction via ProofRequest/ContextProof boundary only.
I-VEDATA-4: VEDATA_ORACLE_PK is a known public constant; key rotation by OrgDID governance.
```

---

## §11.6 Tier 5 — Knowledge-Based Recovery [N] *(v4.5)*

### §11.6.1 Overview

```
Tier 5 is the weakest recovery tier. It is the fallback for PersonDID
holders with no secondary wallet and no guardian quorum.
Enrollment is optional but strongly recommended for all users.
Tier 5 enrollment is optional at registration and CAN be added or updated
post-hoc via a signed TAAD state transition (seq++, controller_pkh must sign).
Enrolling at registration is strongly recommended to ensure recovery coverage
from day one. Updating enrollment after registration requires the user to still
have access to their current device (HW_Key).

Cascade position: invoked only after Tiers 1, 2, 3 are all unavailable
(see §11.1 updated cascade).

Deliberate design choice: Tier 5 is last resort, not first resort.
Users with guardians or secondary wallets should rely on those first.
```

### §11.6.2 KnowledgeFactors Type [N] *(v4.5 revised)*

```
-- EmailOracle: VeData-pattern oracle for email possession verification.
-- Analogous to VeData ContextProof (§11.5) — PhoenixKey verifies signature,
-- does not implement email sending logic.
EMAIL_ORACLE_PK : Ed25519PubKey  -- known public constant; governed by PhoenixKey OrgDID

EmailAccessProof ≜ {
  email_commitment_hash : {0,1}^256,  -- must match datum.email_commitment
  recovery_challenge    : {0,1}^256,  -- H(did ∥ new_hw_pub ∥ nonce)
  verified_at_slot      : SlotNo,
  nonce_echo            : {0,1}^256,  -- anti-replay: echoes challenge nonce
  oracle_signature      : {0,1}^512,  -- Sign(EMAIL_ORACLE_KEY, H(proof_fields))
}

KnowledgeFactors ≜ {
  -- Factor 1: Password (knowledge)
  password_commitment : {0,1}^256,  -- BLAKE2b-256(Argon2id_out ∥ salt_pw)
  password_salt       : {0,1}^128,  -- on-chain; see §11.6.3 rationale

  -- Factor 2: Recovery Email (possession via EmailOracle OTP)
  email_commitment    : {0,1}^256,  -- BLAKE2b-256(email_address ∥ salt_em)
  email_salt          : {0,1}^128,  -- on-chain

  enrolled_at         : SlotNo,
  enrolled_seq        : ℕ,
}

-- v4.5 KEY CHANGE: All salts are ON-CHAIN.
-- v4.4 stored salt_pw₁ encrypted with Master_KEK → circular dependency:
--   Tier 5 invoked when Master_KEK unavailable → salt unrecoverable → DEADLOCK.
-- v4.5 fix: salts on-chain. Security is provided by Argon2id memory-hardness,
--   not by salt-hiding. See §11.6.3 for security analysis.

-- Email factor semantics change (v4.5):
--   v4.4: email was knowledge factor (knowing email address string is sufficient)
--   v4.5: email is possession factor (active mailbox access required via OTP)
--   Init Tier 5 requires EmailAccessProof from oracle, not just email string.
```

### §11.6.3 Commitment Construction — EXCLUSIVELY Off-Chain [N] *(v4.5)*

```
-- All computations run on mobile/SDK. NEVER in Plutus validator.

Password commitment (at enrollment):
  pw_argon2id_out     = Argon2id(
    password,
    salt = password_salt,    -- password_salt is randomly generated at enrollment
    m    = 65536,            -- 64 MiB (KiB units per RFC 9106 §3.1)
    t    = 3,                -- 3 iterations
    p    = 4                 -- 4 parallelism threads
  )
  password_commitment = BLAKE2b-256(pw_argon2id_out ∥ password_salt)

Email commitment (at enrollment):
  email_commitment    = BLAKE2b-256(email_address ∥ email_salt)
  -- email_salt randomly generated at enrollment

-- What goes ON-CHAIN (all of these):
--   {password_commitment, password_salt, email_commitment, email_salt}
-- What stays OFF-CHAIN: password plaintext, email plaintext

-- Security rationale — why salts on-chain is acceptable (v4.5 change):
--   Attacker with on-chain data has: password_commitment + password_salt
--   Attack: run Argon2id(guess, password_salt) and compare
--   Cost: ~0.3s per attempt on dedicated GPU (64 MiB memory-hard)
--   12+ char password, zxcvbn ≥ 3 → ~60 bits entropy
--   2^60 attempts × 0.3s ≈ 10^18 seconds → computationally infeasible
--   Protection source: Argon2id memory-hardness, not salt-hiding.
--   Salt-hiding in v4.4 was unnecessary layering that created deadlock.

Email OTP flow (at Tier 5 initiation — off-chain):
  1. User provides email_address + password to PhoenixKey backend
  2. Backend verifies: BLAKE2b-256(email_address ∥ datum.knowledge_factors.email_salt) = email_commitment
  3. Backend generates: otp ← SecureRandom(6 digits)
                        nonce ← SecureRandom(256)
  4. Backend sends OTP to email_address (EmailOracle service)
  5. User enters OTP in app → backend verifies → backend signs EmailAccessProof
  6. EmailAccessProof submitted on-chain with recovery tx
```

### §11.6.4 Recovery Execution [N] *(v4.5)*

```
Tier5Proofs ≜ {
  pw_argon2id_out  : ByteArray,      -- off-chain Argon2id output for password
  email_access     : EmailAccessProof, -- oracle-signed proof of mailbox access
}

tier_5_recovery(did: PersonDID, factors: KnowledgeFactors,
                new_hw_pub: Ed25519PubKey,
                proofs: Tier5Proofs, s: SlotNo):
  Pre:
    TAADDatum.knowledge_factors ≠ None
    ∧ factors = TAADDatum.knowledge_factors

    -- Password: validator verifies BLAKE2b (no Argon2id on-chain):
    ∧ BLAKE2b-256(proofs.pw_argon2id_out ∥ factors.password_salt)
        = factors.password_commitment

    -- Email: validator verifies EmailOracle signature:
    ∧ Verify(EMAIL_ORACLE_PK,
             H(proofs.email_access.fields_excl_sig),
             proofs.email_access.oracle_signature)
    ∧ proofs.email_access.email_commitment_hash = factors.email_commitment
    ∧ proofs.email_access.recovery_challenge
        = H(did ∥ new_hw_pub ∥ proofs.email_access.nonce_echo)
    ∧ |s - proofs.email_access.verified_at_slot| ≤ 300  -- freshness: ~100s

    -- Both factors required (AND logic — neither alone is sufficient)

  Effect:
    TAADDatum.state ← Recovering {
      pending_hw_pub    = new_hw_pub,
      recovery_deadline = s + TIER5_TIMELOCK,  -- §1.4: 21 × 432,000 slots
      verification_tier = 5,
      crypto_proofs     = H(proofs),
      cancel_nonce      = H(proofs ∥ encode(s)),
    }
    seq++

-- TIER5_TIMELOCK = 21 × 432_000 slots  (defined in §1.4)
```

### §11.6.5 Security Invariants [N] *(v4.5)*

```
I-TIER5-PW-1 [N]: Argon2id MUST execute exclusively off-chain (mobile/SDK).
  Plutus validator MUST NOT execute Argon2id.
  Validator only verifies:
    BLAKE2b-256(proof.argon2id_out ∥ datum.password_salt) = password_commitment
  Rationale: Argon2id requires 64 MiB RAM and ~300ms CPU.
  Plutus execution budget: ~10ms CPU equivalent, no memory-hard operations.
  Attempting on-chain = immediate script budget exhaustion.

I-TIER5-PW-2 [N]: Argon2id parameters at enrollment MUST satisfy:
  m ≥ 65536 (= 64 MiB, unit: KiB as per Argon2 spec RFC 9106),
  t ≥ 3 (iterations), p ≥ 4 (parallelism threads).
  Rationale: These parameters yield ~0.3s per attempt on dedicated GPU.
  Combined with 12+ char password (~60 bits entropy):
    2^60 attempts × 0.3s ≈ 10^18 seconds → computationally infeasible.
  Ref: RFC 9106 §4 — https://www.rfc-editor.org/rfc/rfc9106

I-TIER5-PW-3 [N-UX]: Password strength enforced at enrollment (UX layer only):
  ≥ 12 characters.
  zxcvbn score ≥ 3 — ref: https://github.com/dropbox/zxcvbn
  Not present in HaveIBeenPwned breach corpus — checked via k-anonymity
  SHA-1 prefix protocol (first 5 hex chars of SHA-1(password) sent to API;
  full hash never transmitted) — ref: https://haveibeenpwned.com/API/v3#PwnedPasswords
  Validator does NOT enforce strength. UX layer is sole enforcement gate.

I-TIER5-PW-4 [N]: Post-recovery password and email re-enrollment mandatory.
  PhoenixKey app MUST force re-enrollment of:
    new password_commitment, new email_commitment
  within first authenticated session post-recovery.
  Old knowledge_factors invalidated upon Finalize_Recovery (seq++).
  Validator detects stale factors: knowledge_factors.enrolled_seq < TAADDatum.seq
  → app blocks until re-enrollment complete.

-- NOTE: I-TIER5-PW-5 from v4.4 (salt encryption on LampNet) is REMOVED.
-- v4.4 encrypted salt_pw₁ with Master_KEK — created circular dependency
-- (Tier 5 invoked when Master_KEK unavailable → cannot decrypt salt → deadlock).
-- v4.5 fix: all salts on-chain; security via Argon2id memory-hardness (see §11.6.3).

I-TIER5-EMAIL-1 [N]: Email factor MUST be treated as possession, not knowledge.
  Validator MUST verify EmailAccessProof oracle signature (§11.6.4).
  Validator MUST NOT accept email address string as standalone proof.
  Rationale: email address is low-entropy (often public) and provides
  near-zero additional security as a knowledge factor.
  Email as possession (OTP from mailbox) requires active mailbox access.

I-TIER5-EMAIL-2 [N]: EMAIL_ORACLE_PK is a known public constant.
  Key rotation requires PhoenixKey OrgDID governance vote.
  All EmailAccessProof signatures verified against current EMAIL_ORACLE_PK.
  Analogous to VEDATA_ORACLE_PK (I-VEDATA-4, §11.5).
```

---

## §11.7 Tier 5 — Cancel Mechanism [N] *(v4.5)*

```
-- Standard TAAD_cancel_recovery (§10.2) requires hw_pub signature.
-- Tier 5 is invoked when device is UNAVAILABLE → hw_pub sig not possible.
-- v4.4 included "Cancel Option A: same knowledge factors" — REMOVED in v4.5.
-- Reason: attacker who initiated with (password + email OTP) possesses
-- the same knowledge factors → Option A cannot distinguish owner from attacker.

-- Cancel authority for Tier 5 (v4.5):
TAAD_cancel_tier5(did: PersonDID, cancel_proof: Tier5CancelProof, s: SlotNo):
  Pre:
    TAADDatum.state = Recovering { verification_tier = 5 }
    ∧ s < recovery_deadline
    ∧ (
      -- Option 1: Email notification cancel token (PRIMARY mechanism)
      -- Backend generates cancel_token = H(cancel_nonce ∥ "cancel-tier5")
      -- and embeds in notification email sent at initiation.
      -- Legitimate owner receives email → clicks cancel link → submits token.
      -- Attacker who initiated cannot suppress notification unless
      -- they have full control of email account (already possible via OTP).
      H(cancel_proof.token ∥ "cancel-tier5") = H(state.cancel_nonce ∥ "cancel-tier5")

      -- Option 2: Higher tier became available (device found, guardian reached)
      ∨ tier_1_proof_valid(did, cancel_proof.secondary_sig)
      ∨ guardian_threshold_signed(did, cancel_proof.guardian_sigs)
    )
  Effect: state ← Active; seq++

-- Security note on Option 1:
-- If attacker controls email account: they can initiate AND suppress cancel.
-- This is a fundamental limitation of email-based recovery.
-- Mitigations: 21-day window (long detection period), email notification
-- to any linked secondary contact (if configured), LampNet activity log.
-- Residual risk: email account security is the last line of defense for Tier 5.
-- This risk is accepted: Tier 5 is the weakest tier by design.
-- Users with higher security requirements should configure Tier 1/2/3.

I-TIER5-CANCEL-1 [N]: Tier 5 recovery initiation MUST trigger email
  notification to enrolled email within 24h (backend responsibility).
  Notification MUST include: cancel link containing cancel_token.

I-TIER5-CANCEL-2 [N]: Tier 5 cancel MUST NOT require hw_pub signature.
  Standard TAAD_cancel_recovery (§10.2, hw_pub based) remains available
  as an additional path if user has another device, but is not required.

I-TIER5-CANCEL-3 [N]: Cancel Option A (knowledge factors as cancel proof)
  is PROHIBITED. Knowledge factors cannot differentiate the legitimate owner
  from the attacker who initiated recovery (both possess same factors).
  Cancel authority = email notification token OR higher tier proof.
```

---

# PART IV — DID TYPE CATALOG

## §12 PersonDID — Root of Trust [N]

```
-- Relationship to TAADDatum: PersonDID is the canonical conceptual entity.
-- TAADDatum (§10.1) is its on-chain Plutus datum. All PersonDID state lives in TAADDatum.
-- Fields marked with (*) are stored in TAADDatum on-chain; others are off-chain (LampNet).

PersonDID ≜ DIDBase where { type=Person, owner=None, security_level=Biometric_Hardware,
                             scope={Full_Authority} }
  extended by {
    hw_pub         : Ed25519PubKey,   -- (*) HW_Key: Secure Enclave, biometric-gated
    taad_key       : Ed25519PubKey,   -- TAAD_Key: recoverable (only hash=controller_pkh on-chain)
    guardians      : List<GuardianConfig>,
    biometric_hash : {0,1}^256,       -- H(biometric enrollment template); never raw
    seed_exported  : Option<SlotNo>,
    import_mode    : 𝔹,
    linked_dids    : List<DIDLink>,   -- cross-chain identity links (§12.1)
    -- v4.3 additions:
    emergency_vault_key : {0,1}^256,  -- HKDF-derived; see §33.1; NOT Master_KEK
    guardian_did        : Option<DID>,-- (*) non-None iff minor; see §34.3
    birth_slot          : Option<SlotNo>, -- for minor age computation; privacy: optional
    jurisdiction        : Set<JurisdictionCode>, -- applicable legal jurisdictions
    estate_config       : Option<EstateConfig>,  -- (*) posthumous designation; see §32.1
    -- v4.4 additions:
    knowledge_factors   : Option<KnowledgeFactors>, -- (*) Tier 5 recovery; see §11.6
    -- v4.5 additions:
    secondary_wallets   : List<SecondaryWallet>,    -- (*) Tier 1 recovery commitment list; see §11.2
    backup_deadline     : Option<SlotNo>,           -- (*) None=backed up; Some(s)=grace period
  }
  -- Note: (*) fields stored in TAADDatum on-chain; remaining fields in LampNet DID Document.

-- §12.1: Cross-chain DID Linking
-- A user may own DIDs on multiple chains (did:ethr, did:prism, did:sol...).
-- Linking proves these are the same person without merging the ecosystems.
DIDLink ≜ {
  external_did    : ByteArray,        -- e.g., "did:ethr:0xabc..." or "did:prism:def..."
  proof_slot      : SlotNo,           -- when the link was established
  link_proof      : {0,1}^512,        -- Sign(external_private_key, H("link-to" ∥ phoenix_did ∥ slot))
  link_type       : SameAs | Controls | SeeAlso,  -- W3C DID alsoKnownAs semantics
}

-- DIDLink verification:
verify_link(link: DIDLink, phoenix_did: DID) : 𝔹 ≜
  external_pub = resolve_external_key(link.external_did)   -- via universal resolver
  ∧ Verify(external_pub, H("link-to" ∥ phoenix_did ∥ encode(link.proof_slot)), link.link_proof)
  -- Cross-chain proof: external key signs "I am also did:phoenix:..."
  -- This is a one-way proof; the external DID document may also declare alsoKnownAs

-- Invariants:
I-LINK-1: link_proof MUST be verified against a fresh slot (proof_slot ≥ created_slot)
I-LINK-2: link_type = SameAs implies the user asserts legal equivalence (same natural person)
I-LINK-3: linked_dids entries are user-declared; PhoenixKey does NOT auto-verify external chains

-- Biometric_HW_sig (F-13: SecureEnclaveSession explicit):
Biometric_HW_sig(did, op, s) ≜
  ∃ (session: SecureEnclaveSession) :
    session.biometric_verified
    ∧ session.signed(H(op ++ encode(s) ++ encode(did.seq)), sig)  -- atomic in hardware
    ∧ session.timestamp ∈ [s - 60_slots, s]
    ∧ Verify(did.hw_pub, H(op ++ encode(s) ++ encode(did.seq)), sig)
    -- Biometric + signing = one hardware session; software cannot interleave.

Invariants:
  I-PERSON-1: scope = {Full_Authority}
  I-PERSON-2: owner = None
  I-PERSON-3: no nested PersonDIDs (PersonDID cannot own PersonDID)
  I-PERSON-4: seed_exported ≠ None → security_note on-chain
  I-PERSON-5: One PersonDID per biometric_hash (VeData + ZK at creation)
  I-PERSON-6: import_mode=True → MUST NOT admin OrgDID with {Full_Authority, Mint_Token}
  I-PERSON-7: let S_susp = {g | g.suspension_eligible=True};
              |S_susp| = 0 ∨ |S_susp| ≥ 3  (disable OR min 3 eligible)
  I-PERSON-8: suspension_threshold > recovery_threshold
  I-PERSON-9: PersonDID_Privileged_Ops bypass Active() always
  -- v4.3:
  I-PERSON-10: emergency_vault_key = HKDF(Master_KEK, "emergency-vault-v1", H(DID ∥ "ev-salt-v1"))
               -- Derived at registration; updated on key rotation; never exported raw
  I-PERSON-11: guardian_did ≠ None → type(guardian_did) ∈ {Person, Org}
               -- Guardian must be an existing active DID
  I-PERSON-12: estate_config ≠ None → estate_config.beneficiary ≠ did
               -- Cannot designate self as beneficiary
```

---

## §13 OrgDID — Collectives [N]

```
OrgDID ≜ DIDBase { type=Org, security_level=Threshold(m,n) }
  extended by {
    policy          : { threshold: ℕ, total: ℕ, time_window: ℕ },
    members         : Set<DID>,   -- PersonDID or OrgDID
    parent          : Option<DID>,
    org_class       : Company|DAO|Government|NGO|Cooperative|Partnership|Trust|Informal,
    legal_hash      : Option<{0,1}^256>,
    admin_emergency : Option<EmergencyAdmin>,
  }

-- F-04: Emergency admin (replaces unconstrained admin_key):
EmergencyAdmin ≜ {
  admin_did           : PersonDID,
  admin_emergency_cap : Set<Capability>,   -- strict subset of org scope
  admin_timelock_tier : {T2, T3},          -- default T2; governance can raise to T3
  -- CANNOT include: Full_Authority, Mint_Token, Transfer_Machine
  -- Permitted: Write_DID, Operate_Device, Access_Service, Read_DID
}

Threshold_sig(did, op, s) ≜
  ∃ S ⊆ members: |S| ≥ threshold ∧ ∀ m∈S: Verify(key(m,s), H(op++seq(did)), sig_m)
  ∧ all_sigs_within(S, time_window, s) ∧ ∀ m∈S: Active(m, s)

-- F-14: Constitutional threshold for policy changes:
constitutional_threshold(n) = ⌊2n/3⌋ + 1   -- strict supermajority; n=3→3, n=7→5

Invariants:
  I-ORG-1: threshold ≤ |members|; I-ORG-2: threshold ≥ 2
  I-ORG-3: type(m) ∈ {Person, Org} for m ∈ members
  I-ORG-4: scope(OrgDID) ⊆ scope(parent)
  I-ORG-5: no circular membership
  I-ORG-6: admin_emergency_cap ⊊ scope(OrgDID)
  I-ORG-7: all emergency ops subject to admin_timelock_tier
```

---

## §14 ContextDID — Temporal Contexts [N]

```
ContextDID ≜ DIDBase { type=Context, security_level=Owner_Delegated }
  extended by {
    purpose    : ByteArray,
    admin      : DID,             -- PersonDID or OrgDID
    members    : Set<DID>,        -- any type; cross-org allowed by design
    start_slot : SlotNo,
    end_slot   : SlotNo,          -- I-CTX-1: ≥ start_slot + 432_000 (min 1 epoch)
    auto_dissolve : 𝔹,
    transcript_hash : Option<{0,1}^256>,
  }

Active(c: ContextDID, s) additionally requires s ≤ c.end_slot.
Auth requires Admin_sig(did, op, s) ∧ s ∈ [start_slot, end_slot].

Invariants:
  I-CTX-1: end_slot ≥ start_slot + 432_000
  I-CTX-2: scope(ContextDID) ⊆ scope(admin)
  I-CTX-3: ∀ m ∈ members: scope_granted(m) ⊆ scope(ContextDID)
  I-CTX-4: s > end_slot → Authority = False (hard expiry)
  I-CTX-5: ContextDID cannot own Person or Org DIDs
  Note: member diversity across ownership trees is by design (cross-org collaboration).
```

---

## §15 DeviceDID — Computing Hardware [N]

```
DeviceDID ≜ DIDBase { type=Device, security_level=Hardware_Attestation }
  extended by {
    owner            : DID,
    device_class     : Mobile|Laptop|Server|VPS|Sensor|Drone|Wearable|Gateway|LampNetNode,
    hw_cert          : TPM2_Quote | SEP_Cert | FIDO2_Assertion,
    cert_issued_slot : SlotNo,   -- F-09: slot-based (not epoch-based)
    device_id_hash   : {0,1}^256,
    provisioner      : DID,
  }

-- F-09: Slot-based TTL per device class:
MAX_CERT_AGE_SLOTS ≜ {
  Mobile→4320, Server→2160, VPS→2160, LampNetNode→8640,
  Drone→8640, Gateway→17280, Wearable→17280, Laptop→8640,
  Sensor→43200  -- ~2.5 days; IoT constraint (see §15.1)
}

HW_Attestation(did, op, s) ≜
  HW_cert_valid(did.hw_cert, H(op++seq(did)))
  ∧ s < did.cert_issued_slot + MAX_CERT_AGE_SLOTS[did.device_class]
  ∧ Active(did, s)
-- No biometric. Authenticates hardware identity, not human operator.
-- F-21: TRUSTED_PCR_REGISTRY governed on-chain by OrgDID (Security Council).

Invariants: I-DEV-1 through I-DEV-6 (device_id_hash immutable, PCR attest for LampNetNode, etc.)
```

### §15.1 Sensor TTL Rationale [I]

Sensor TTL = 43,200 slots (~2.5 days). Low-power IoT cannot re-attest hourly. Owner detects compromise and revokes within 2.5 days. Future: offline attestation batch model (v5.0).

---

## §16 MachineDID — Physical Machines [N]

```
MachineDID ≜ DIDBase { type=Machine, security_level=Threshold(m,n) }
  extended by {
    owner                : DID,
    operator             : Option<DID>,
    operator_max_tx_value: ℕ,          -- F-10: cap on operator's Sign_Tx authority
    operator_scope       : Set<Capability>,  -- strict subset of machine scope
    regulator            : Option<DID>,
    machine_class        : LandVehicle|AirVehicle|WaterVehicle|IndustrialRobot|
                           AgriculturalMachine|MedicalDevice|ConstructionMachine|Other,
    serial_hash          : {0,1}^256,  -- immutable
    transferable         : 𝔹,
    transfer_policy      : TransferPolicy,
  }

Stakeholder_sig(did, op, s) ≜ case op of
  | Operate_Machine → operator_sig ∨ owner_sig
  | Sign_Tx(v) →
      IF v ≤ operator_max_tx_value: operator_sig ∨ owner_sig
      ELSE: owner_sig only
  | Transfer_Machine → owner_sig ∧ (regulator_sig if required by transfer_policy)
  | Write_DID → owner_sig

Invariants:
  I-MACH-AGENT [N]: Machine-owned Agent Deploy_Agent → requires owner sig, not operator
  I-MACH-AGENT-2 [N]: Agent under Machine: Sign_Tx(v) via operator auth → v ≤ operator_max_tx_value
  I-MACH-4: Machine can own DeviceDID, BotDID, AgentDID (CanOwn, F-12)
```

---

## §17 AssetDID — Passive Physical Assets [N]

```
-- F-16: Attestation ≜ { attester: ServiceDID, attested_slot, attest_hash, signature }
ATTESTATION_MIN ≜ {Crop→3, Land→2, Good→1, _→1}

AssetDID ≜ DIDBase { type=Asset, security_level=Owner_Delegated }
  extended by {
    owner          : DID,
    custodian      : Option<DID>,
    custodian_cap  : Set<Capability>,  -- default {Read_DID} (F-28)
    asset_class    : Land|Crop|Good|Container|Property|Commodity,
    location_proof : {0,1}^256,        -- H(GPS zone)
    physical_id    : {0,1}^256,        -- immutable
    attestations   : List<Attestation>, -- multi-oracle (F-16)
    tokenized      : Option<PolicyID>,
    fraction_policy: Option<PolicyID>,
    transferable   : 𝔹,
  }

-- F-18: Base NFT burn constraint:
Op_burn_base_NFT(did, s):
  Pre: fraction_policy=None ∨ all_fractions_burned(did)
  Effect: base NFT burned; revoked_slot ← s

-- Atomic transfer (I-ASSET-4): NFT transfer ∧ owner update in same Tx.
-- CropNFT = AssetDID { asset_class=Crop, attestations≥3, tokenized≠None }

Invariants: I-ASSET-1..7 (physical_id immutable, multi-attestation, transfer atomicity, etc.)
```

---

## §18 BotDID — Software Automation [N]

```
-- F-06-R: Token bucket rate limiting with Q-format (avoids REFILL_RATE=0 edge case):
BotDID ≜ DIDBase { type=Bot, security_level=Software }
  extended by {
    owner          : DID,   -- PersonDID, OrgDID, or ServiceDID
    bot_class      : Chat|API|Webhook|Automation|Indexer|Keeper|Oracle|Other,
    api_key        : Ed25519PubKey,
    max_rate       : ℕ,     -- ops per BOT_REFILL_WINDOW slots
    whitelist      : Set<DID>,
    token_bucket_Q : ℕ,     -- current tokens × Q; initial = max_rate × Q (owner trusted at creation)
    last_refill_slot: SlotNo,
  }

REFILL_RATE_Q(did) ≜ did.max_rate × Q // BOT_REFILL_WINDOW   -- Q-format; always ≥ 1 if max_rate ≥ 1

Rate_ok(did, s) ≜
  let elapsed   = s - did.last_refill_slot
      refilled  = did.token_bucket_Q + elapsed × REFILL_RATE_Q(did)
      current_Q = min(did.max_rate × Q, refilled)
  in current_Q ≥ Q

-- State update: token_bucket_Q ← current_Q - Q; last_refill_slot ← s; seq++
-- No burst: tokens refill continuously; no hard boundary reset event.

Invariants: I-BOT-1..5 (scope strict subset, token_bucket_Q ∈ [0, max_rate×Q], no hw attestation)
```

---

## §19 AgentDID — Autonomous AI [N]

```
-- F-17: ModelHashMode distinguishes declaration vs TEE attestation:
ModelHashMode ≜ Declaration | TEE_Attested

AgentDID ≜ DIDBase { type=Agent, security_level=Software_HSM }
  extended by {
    owner              : DID,
    model_hash         : {0,1}^256,
    model_hash_mode    : ModelHashMode,
    capability_set     : Set<Capability>,
    audit_contract     : Address,
    audit_funding      : { source: DID, min_balance_ada: ℕ, auto_refill: 𝔹 },
    max_value_per_tx   : ℕ,
    sub_depth          : ℕ,
  }

-- F-02-R: Non-consuming audit output (no throughput bottleneck):
Audit_record(did, op, s) ≜
  ∃ output ∈ tx.outputs :
    output.address = did.audit_contract
    ∧ output.datum = { agent: did, op, slot: s, tx_hash }
  -- Non-consuming: multiple agents produce audit outputs independently in same block.

Capability_token(did, op, s) ≜
  op ∈ did.capability_set ∧ Value_ok(did, op) ∧ Active(did, s) ∧ Audit_record(did, op, s)
  -- audit co-produced in same Tx (not "before"; EUTxO atomicity ensures simultaneity)

-- Sub-agent: new AgentDID with capability_set ⊆ parent's; sub_depth decremented.

Invariants:
  I-AGENT-1: capability_set ⊆ capability_set(owner)
  I-AGENT-2: audit output in every action Tx (non-consuming output pattern)
  I-AGENT-4: revocation O(1); no timelock
  I-AGENT-6: model_hash_mode=Declaration → security_note; TEE_Attested → TEE proof
```

---

## §20 ServiceDID — Digital Services [N]

```
ServiceDID ≜ DIDBase { type=Service, security_level=Software_HSM }
  extended by {
    owner          : DID,   -- PersonDID or OrgDID (the company)
    service_name   : ByteArray,
    version        : ℕ,     -- monotone increasing; no rollback
    capability_set : Set<Capability>,
    api_hash       : {0,1}^256,
    audit_contract : Address,
    app_token      : Option<PolicyID>,
    service_key    : Ed25519PubKey,
  }
-- ServiceDID ≠ OrgDID: ServiceDID = product (OriLife); OrgDID = company (Greensun Tech).
-- ServiceDID is the formal AppTokenEconomics actor (resolves app_id gap from ATE-SA-v1.0).

-- F-12: Service→Service chain limit:
I-SVC-CHAIN [N]:
  |{anc ∈ Ancestors(did) | type(anc) = Service}| ≤ MAX_SERVICE_DEPTH = 3
  -- Max 4 ServiceDIDs in any chain (3 Service ancestors of leaf).
  -- Sufficient for: OriLife → SatelliteAPI → WeatherAPI.

Invariants: I-SVC-1..6 (scope subset of owner, version monotone, ServiceDID can own Bot/Agent)
```

---

## §21 CharDID — Virtual Entities [N]

```
CharDID ≜ DIDBase { type=Character, security_level=Owner_Delegated }
  extended by {
    owner           : DID,   -- F-19: PersonDID (players) or ServiceDID (NPCs)
    world_id        : {0,1}^256,
    char_class      : PlayerCharacter|NPC|Avatar|VirtualItem|Collectible|Other,
    state_cid       : ByteArray,   -- IPFS CID; min 2 independent pinners required
    on_chain_attrs  : Map<ByteArray, ℕ>,
    transferable    : 𝔹,
    cross_world     : 𝔹,
    nft_policy      : Option<PolicyID>,   -- nft_policy ≠ None ↔ transferable
    authorized_worlds: Set<{0,1}^256>,
  }

-- F-19: NPC ownership:
I-CHAR-1: PlayerCharacter → type(owner) = Person
          NPC             → type(owner) ∈ {Person, Service}
          Others          → type(owner) ∈ {Person, Service}

-- Transfer: NFT transfer ∧ owner update in same atomic Tx (I-CHAR-4).
```

---

# PART V — CROSS-TYPE INTERACTIONS

## §22 Ownership Graph [N]

### §22.1 CanOwn Matrix

```
CanOwn(owner_type, child_type):
               Pers  Org  Ctx  Dev  Mach  Asset  Bot  Agent  Svc  Char
Person       [  ✗    ✓    ✓    ✓    ✓     ✓      ✓    ✓      ✓    ✓  ]
Org          [  ✗    ✓    ✓    ✓    ✓     ✓      ✓    ✓      ✓    ✓  ]
Context      [  ✗    ✗    ✗    ✗    ✗     ✗      ✗    ✗      ✗    ✗  ] leaf
Device       [  ✗    ✗    ✗    ✗    ✗     ✗      ✗    ✗      ✗    ✗  ] leaf
Machine      [  ✗    ✗    ✗    ✓    ✗     ✗      ✓    ✓      ✗    ✗  ] F-12: embedded AI
Asset        [  ✗    ✗    ✗    ✗    ✗     ✗      ✗    ✗      ✗    ✗  ] passive leaf
Bot          [  ✗    ✗    ✗    ✗    ✗     ✗      ✗    ✗      ✗    ✗  ] automation leaf
Agent        [  ✗    ✗    ✗    ✗    ✗     ✗      ✓    ✓      ✗    ✗  ] tool deployment
Service      [  ✗    ✗    ✗    ✓    ✗     ✓      ✓    ✓      ✓    ✗  ] F-12: +Svc→Svc (I-SVC-CHAIN)
Character    [  ✗    ✗    ✗    ✗    ✗     ✗      ✗    ✗      ✗    ✗  ] virtual leaf

Key rationale:
  Machine→Bot,Agent ✓: Vehicles/robots need embedded firmware bots and AI reasoning.
  Service→Service ✓:   Microservice architectures (bounded by I-SVC-CHAIN ≤ 3 hops).
  Agent→Bot ✓:         AI agents deploy tool bots for specific sub-tasks.
  Machine→Machine ✗:   No nested machines (use OrgDID for fleets).
```

### §22.2 Depth Constraint

```
depth(PersonDID) = 0
depth(did) = depth(owner(did)) + 1

C-DEPTH: ∀ did: depth(did) ≤ MAX_DELEGATION_DEPTH = 10
```

### §22.3 Taxonomy Rationale — Bot vs Agent vs "Robot"; loại còn thiếu [I]

*(đề xuất `spec-proposals/PhoenixKey-Spec-Addendum-v4.7-DRAFT.md` §C, chưa chốt version bump)*

Làm rõ vì sao §18 BotDID và §19 AgentDID tách riêng, vì sao KHÔNG có
"RobotDID", và đánh giá các ứng viên loại-DID-mới hay gặp. Bảng CanOwn (§22.1)
đã cho phép `Machine → Bot, Agent` (F-12) — mục này giải thích rationale đằng
sau quan hệ đó.

Ba khái niệm khác nhau theo **bản chất tác tử + thân vật lý**, không theo tên gọi:

| | Tác tử (ra quyết định) | Thân vật lý | DID đúng |
|---|---|---|---|
| **BotDID** (§18) | luật cứng / kịch bản (KHÔNG tự học) | không | Bot |
| **AgentDID** (§19) | AI tự hành (mục tiêu mở, quyết định động) | không | Agent |
| **"Robot"** (con robot lau nhà, xe tự lái) | — | **CÓ** (máy vật lý) | **MachineDID (§16)** sở hữu Bot/Agent (firmware/AI nhúng) |

**Không cần "RobotDID".** Một robot = **MachineDID** (thân máy, transferable,
CanOwn cho phép Machine sở hữu Device/Bot/Agent) + (tuỳ chọn) một
**AgentDID/BotDID** nó sở hữu cho phần "trí não". Tách thân/não cho phép: bán
robot (transfer Machine, §16 `transferable`/`transfer_policy`) mà **giữ hoặc
thay** AI bên trong; một AI điều khiển nhiều thân; quy trách nhiệm (§25 Attr*
dừng ở Person/Org/Service — robot quy về chủ qua `Attr*(owner)`).

*Nếu gộp thì sao:* gộp Bot+Agent → mất phân biệt "tự động theo luật" vs "tự
hành học được", không bound được `Deploy_Agent(max_cap)` khác `Deploy_Bot(max_cap)`
(Appendix C Tier 5); rủi ro cấp quyền AI mở cho thứ đáng lẽ chỉ chạy luật. Gộp
Machine+Agent (làm "RobotDID") → không transfer được thân mà giữ não; AI nhúng
và AI đám mây phải dùng hai loại khác nhau dù cùng bản chất tác tử; vỡ MECE
(Appendix A): "vật lý" vs "số" bị trộn. → Giữ tách là quyết định MECE có chủ
đích, không phải dư thừa.

**Loại cần DID nhưng chưa nằm trong 10 loại — đánh giá.** Range type
`0x0A–0x7F` reserved (§2.2). Ứng viên hay gặp + ánh xạ:

| Ứng viên | Có cần loại mới? | Khuyến nghị |
|---|---|---|
| Tài khoản/ví tài chính | Không | thuộc ví Cardano + Person/Org owner |
| Place/địa điểm (cửa hàng, toà nhà) | Cân nhắc | tạm dùng **AssetDID** (vật thụ động) hoặc ContextDID nếu có thời hạn; loại "PlaceDID" chỉ khi cần thuộc tính không-gian normative riêng |
| Sự kiện (event, vé) | Không | **ContextDID** (§14, có expiry) đúng bản chất |
| Tài liệu/giấy tờ/credential | **Không** (quan trọng) | KHÔNG phải DID — là **VC** do issuer phát hành về một subject-DID (§35.2 [M-3]) |
| Nhóm/cộng đồng phi pháp-nhân | Không | **OrgDID** |
| Mô hình AI / dataset | Cân nhắc | nếu là "tác tử" → Agent; nếu là tài sản số thụ động → Asset; nếu là dịch vụ → Service |
| Tài sản thật token-hoá (RWA: nhà, vàng) | Không (phần lớn) | **AssetDID** + attestation; pháp lý ở VC/§34 |
| Tài khoản/khoá liên-chuỗi | Không | `linked_dids`/DIDLink của PersonDID (§12) |

→ 10 loại + reserved range hiện ĐỦ cho gần hết nhu cầu; phần lớn "loại mới"
thực ra là **VC về một DID** chứ không phải DID mới. Chỉ nên thêm type mới khi
có **thuộc tính/lifecycle normative riêng** không gói được trong 10 loại (ứng
viên thật sự nhất: **PlaceDID** nếu hệ sinh thái cần quyền theo không gian).
Trước khi thêm type mới: cập nhật Appendix A (MECE) + CanOwn matrix (§22.1) +
Attr* (§25.2).

---

## §23 Delegation Protocol [N]

```
Op_delegate(from, to, cap, valid_until, sub_del, s) → DelegationToken:
  Pre: Authority(from, Deploy_Agent(cap) ∨ Deploy_Bot(cap), s)
       ∧ cap ⊑ scope(from, s)         -- current scope (F-37); v4.7 (Errata CID-6): ⊆ → ⊑ (§3.3)
       ∧ valid_until > s
       ∧ depth(to) ≤ MAX_DELEGATION_DEPTH
       ∧ CanOwn(type(from), type(to))
       ∧ nonce_not_used_on_chain(nonce, from)    -- F-22

-- Sub-delegation: granted_cap ⊑ parent token's granted_cap (v4.7, Errata CID-6:
-- ⊆ → ⊑, §3.3); valid_until' ≤ valid_until. depth_remaining decremented per hop.
```

### §23.1 DID Authorization Registry — ACL on-chain, pull-model [N]

*(đề xuất `spec-proposals/PhoenixKey-Spec-Addendum-v4.7-DRAFT.md` §A, chưa chốt version bump)*

§3 định nghĩa `Capability` (từ vựng quyền) và §23 `DelegationToken` (uỷ quyền
dạng **token bearer, push** — người giữ xuất trình). Cả hai KHÔNG mô tả một
**ACL on-chain, pull, controller sửa được** để trả lời "AI đang được uỷ quyền
mint token X / chi quỹ / điều khiển thiết bị nhân danh DID này, NGAY BÂY GIỜ".
Registry lấp đúng khoảng đó và khớp triết lý §4.2 (F-01/F-37: luôn đọc scope
HIỆN TẠI lúc validate).

```
RegistryDatum ≜ {
  governing_did : DID,                        -- DID sở hữu bảng quyền này
  entries       : Map<action_tag, Authorization>,
}
Authorization ≜
  | SinglePkh(pkh: VerificationKeyHash)        -- 1 khoá
  | MultiSig(pkhs: List<VerificationKeyHash>, threshold: ℕ)  -- m-of-n
  | Revoked                                    -- vô hiệu (giữ lịch sử)

action_tag : ByteArray                         -- nhãn LOGIC (vd "LAMP"), TÁCH khỏi
                                                -- token_asset_name
```

- **Vật neo:** Registry là một UTxO mang **Registry-NFT one-shot** (policy ≡
  hash registry validator, name = `blake2b_256(governing_did)`), inline
  `RegistryDatum`. NFT bảo đảm tính duy nhất (một bảng / `governing_did`).
- **OPT-IN:** chỉ org issuer cần; PersonDID/anchor thường KHÔNG có registry →
  identity anchor giữ gọn (lean).

**Bất biến:**
```
I-REG-1 [N]: spend Registry để cập nhật entries HỢP LỆ ⟺ controller hiện tại
  của governing_did ký (đọc động qua reference input anchor — rotatable;
  khoá cũ đã xoay KHÔNG sửa được).
I-REG-2 [N]: token policy did_token_mint(registry_nft_policy, registry_nft_name,
  action_tag, …) mint HỢP LỆ ⟺ đọc Registry qua reference input, lấy
  entries[action_tag], và Authorization được thoả (đủ chữ ký SinglePkh /
  MultiSig; Revoked ⇒ luôn fail).
I-REG-3 [N]: Registry chỉ gate AI mint, KHÔNG gate BAO NHIÊU. Cap tổng cung
  thuộc tầng SupplyState riêng (ngoài phạm vi PhoenixKey — xem §23.1
  "Ranh giới" bên dưới). Hai lớp trực giao.
I-REG-4 [N]: Cấm bake hash-script-khác làm tham số chéo vòng (vd policy mint
  của token PHẢI nằm trong datum của tầng supply/cap, không phải param) — nếu
  không sẽ circular, không deploy được. `aiken check` KHÔNG bắt lỗi này (test
  dùng hash hằng) → BẮT BUỘC verify thủ công chuỗi phụ thuộc tham số trước mỗi
  deploy multi-script.
```

**Quan hệ với §3/§23.** Registry KHÔNG thay `Capability` (§3) hay
`DelegationToken` (§23); nó là tầng **governance/policy** TRÊN chúng:

| | DelegationToken (§23) | Authorization Registry (§23.1) |
|---|---|---|
| Mô hình | token bearer (push) | ACL on-chain (pull) |
| Vòng đời | tạm thời, hết hạn | bền, sửa bằng spend |
| Thu hồi | hết hạn / nonce | set `Revoked` (controller ký) |
| Đọc lúc | xuất trình | reference input lúc validate |

**Tổng quát hoá.** Cùng primitive dùng cho mint token / chi quỹ DN (treasury
m-of-n) / DAO-exec / thiết-bị-chung (xe·két·robot) / di sản. Verifier-agnostic
— mỗi verifier (token policy, treasury validator, device validator) tự đọc
registry.

**Ranh giới Layer 0 (§35).** Quyền mint token / chi quỹ là **kinh tế**
(Layer 1+), KHÔNG phải identity (Layer 0). → primitive registry tổng quát CÓ
THỂ là lib dùng chung gần Layer 0, NHƯNG phần đặc thù token (SupplyState/cap
cụ thể của một token nào đó) PHẢI ở lớp ứng dụng token đó (vd MagicLamp cho
LAMP), KHÔNG nhét vào PhoenixKey core — nhất quán với §35.2 [MN-6].

---

## §24 Revocation [N]

### §24.1 Lazy Revocation (F-03)

```
Revoke(did, at, by):
  Pre: Auth_satisfied(by, Write_DID(did), at)
  Effect: revoked_slot ← at; seq++
  -- NO cascade. O(1) cost.

-- Lazy cascade: §4.2 Authority checks ∀ anc: Active(anc, s).
-- Once parent revoked → Active(parent) = False → Authority(all descendants) = False.
```

**Lemma 24.1 (Lazy Cascade Safety).** If `Revoke(did, at, _)` then `∀ desc ∈ Descendants(did), ∀ s ≥ at: Authority(desc, _, s) = False`.
*Proof.* §4.2 requires `∀ anc: Active(anc, s)`. `revoked_slot(did) = at ≤ s → Active(did,s) = False`. ∎

---

## §25 MAGIC Attribution [N]

### §25.1 Attr* Function

```
Attr*(did: DID) → DID ≜
  if type(did) ∈ {Person, Org, Service} then did
  else Attr*(owner(did))
  -- Termination: G is a DAG; depth bounded by MAX_DELEGATION_DEPTH. PersonDID.owner = None. ∎

MAGIC_attribution(op: BurnOp, s) → DID ≜ Attr*(primary_signer(op))
```

### §25.2 Attribution Table

| Type | Attributed To |
|------|---------------|
| Person | PersonDID.pkh |
| Org | OrgDID |
| Context | Attr*(admin) |
| Device | Attr*(owner) |
| Machine | operator ?? Attr*(owner) |
| Asset | N/A (cannot burn) |
| Bot | Attr*(owner) |
| Agent | Attr*(owner) |
| Service | ServiceDID |
| Character | Attr*(owner) |

---

# PART VI — SECURITY ANALYSIS

## §26 Global Invariants [N]

```
-- Ownership:
I-OWN-1: type = Person ↔ owner = None
I-OWN-2: owner(did) ≠ did
I-OWN-3: scope(did) ⊆ scope(owner(did))  [creation time]
I-OWN-4: revoked(owner) → Authority(child) = False  [lazy, via §4.2]
I-OWN-5: depth(did) ≤ 10
I-OWN-6: CanOwn(type(owner), type(child)) = True

-- Authority:
I-AUTH-1: Authority(did, op, s) → Active(did, s)  [except PersonDID_Privileged_Ops]
I-AUTH-2: Authority(did, op, s) → op ∈ scope(did)
I-AUTH-3: Authority(did, op, s) → ∀ anc: Active(anc, s) ∧ op ∈ scope(anc)
I-AUTH-4: Full_Authority ∈ scope(did) → type(did) = Person

-- Sequence: C-SEQ: seq_after = seq_before + 1 per transition (replay prevention)

-- Structural:
I-STR-1: G(s) acyclic ∀ s
I-STR-2: Each non-root DID has exactly one owner
I-STR-3: Roots of G(s) = active PersonDIDs

-- Resolver:
I-RESOLVER-1 [N]: did:phoenix resolver MUST set deactivated(did,s) = True
  iff ¬Active(did,s) ∨ ∃ anc ∈ Ancestors(did): ¬Active(anc,s).
```

---

## §27 Theorems and Proofs [N]

**Theorem 27.1 — Runtime Scope Monotonicity.**
`Authority(did, op, s) = True → ∀ anc ∈ Ancestors(did): op ∈ scope(anc, s)`

*Proof.* Direct from §4.2 Authority predicate. ∎

**Corollary 27.1.1 — Non-Escalation.**
`Authority(did, op, s) → op ∈ scope(owner(did), s)`.
*Proof.* `owner(did) ∈ Ancestors(did)`. Apply Theorem 27.1. ∎

**Corollary 27.1.2 — AgentDID Non-Escalation.**
`Authority(AgentDID A, op, s) → op ∈ capability_set(owner(A))`.
*Proof.* From Corollary 27.1.1 and I-AGENT-1. ∎

**Theorem 27.2 — Revocation Cascade Soundness (Lazy).**
`Revoke(did, at, _) → ∀ desc ∈ Descendants(did), ∀ s ≥ at: Authority(desc, op, s) = False`.
*Proof.* §4.2 Authority requires `∀ anc ∈ Ancestors(desc): Active(anc, s)`. Since `did ∈ Ancestors(desc)` and `revoked_slot(did) = at ≤ s`, `Active(did, s) = False`. Cost: O(1) revocation, O(depth) per authority check. ∎

**Theorem 27.3 — Guardian Suspension Cannot Block Recovery.**
`Op_suspend(PersonDID, until, s) → TAAD_Cancel_Recovery remains available to device holder`.

*Conditions:*
- TAAD_Cancel_Recovery available iff owner retains physical access to hw_pub device.
- TAAD_Init_Recovery (Tier 1) available iff owner retains a secondary wallet.
- In the edge case where device is simultaneously lost AND guardian coalition is malicious AND no secondary wallet exists: recovery path reduces to Tier 2/3, requiring non-colluding guardians. This is a user setup constraint, not a spec flaw.

*Proof.* By §4.2: `op = TAAD_Cancel_Recovery ∈ PersonDID_Privileged_Ops → bypasses Active()`. `TAAD_state_machine_valid(did, TAAD_Cancel_Recovery, s) = Verify(hw_pub, challenge, sig)` — requires hw_pub signature. Guardian coalition controls `suspended_by` but not `hw_pub` (held by owner's physical device). Therefore guardian suspension cannot block cancel while owner has device access. ∎

**Theorem 27.4 — Asset Transfer Uniqueness.**
`∀ AssetDID a, ∀ t: ∃! owner(a,t)`.
*Proof.* I-ASSET-4: NFT transfer and owner update are atomic Tx. Cardano UTxO prevents double-spend. ∎

**Theorem 27.5 — Agent Audit Atomicity.**
If AgentDID `a` executes `op` via Tx `tx` at slot `s`, then `tx` produces audit output at `a.audit_contract`.
*Proof.* `Capability_token` requires `Audit_record(a, op, s)`. By §19: this requires an output at `audit_contract`. Plutus validation: missing output → Tx fails entirely (EUTxO atomicity). ∎
*Note: "co-produced in same Tx" — EUTxO has no intra-Tx ordering.*

**Theorem 27.6 — Context Hard Expiry.**
`∀ ContextDID c, s > c.end_slot → Authority(c, op, s) = False`.
*Proof.* `Active(c,s) = False` when `s > end_slot` (§4.1, hard expiry) OR when `c` is Suspended (F-33). Authority requires `Active`. ∎

**Theorem 27.7 — EUF-TAAD Security** [informal; mechanized proof target v5.0]

*Statement.* No probabilistic polynomial-time (PPT) adversary A can forge a valid `TAAD_Finalize_Recovery` transaction for a PersonDID `d` without one of the following:
- **(a) Key possession:** A possesses the current `hw_pub` private key of `d`, OR
- **(b) Guardian threshold:** A has collected ≥ `policy.threshold` valid guardian signatures on a current-epoch challenge for `d`, AND the recovery timelock T1/T2/T3 has elapsed without cancellation, OR
- **(c) Cryptographic break:** A breaks the underlying Ed25519 signature scheme or BLAKE2b-256 hash function.

*Informal proof sketch.*
1. `TAAD_finalize_recovery` requires `Verify(pending_hw_pub, H("finalize" ∥ s ∥ seq), sig)` where `pending_hw_pub` was bound at `TAAD_init_recovery` time.
2. `TAAD_init_recovery` requires tier-appropriate proofs (§11). Tier 1 requires the recovery tx to be signed by a pkh whose commitment `BLAKE2b-256(pkh ∥ did ∥ enrolled_slot)` is in `TAADDatum.secondary_wallets` (v4.5 commitment-based approach; see §11.2). Tier 2/3 requires ≥ threshold guardian signatures on a fresh nonce.
3. The A-6 invariant (§10.4) enforces that A cannot finalize before `recovery_deadline`.
4. The C-SEQ invariant prevents replay of old recovery proofs.
5. Therefore A must either (a) forge a signature without the private key — impossible by Ed25519 security — or (b) collect threshold guardian signatures — requires social engineering ≥ threshold guardians — or (c) break the hash function.

*Security note.* The device-loss + guardian-collusion edge case (no secondary wallet, colluding guardians) reduces TAAD security to the physical security of the guardian keys and the collateral disincentive (§11.3.1). This is a user configuration constraint, not a protocol flaw. Formal reduction to Ed25519-UF and SHA3-collision resistance: target v5.0.

---

## §28 Attack Model [I]

| Attack | DID Type | Mitigation |
|--------|----------|------------|
| Guardian coalition suspend + fraudulent recovery | PersonDID | T27.3: Cancel_Recovery bypasses Active(); timelock; guardian slashing |
| Scope temporal attack (parent reduces capability) | All | T27.1: ancestor scope checked at runtime |
| Audit omission | AgentDID | T27.5: non-consuming output in same Tx |
| Cascade revocation DoS | All | F-03: O(1) lazy revocation; no cascade Txs |
| OrgDID admin_key bypass | OrgDID | F-04: bounded emergency_cap + T2 timelock |
| 5-day device cert window | DeviceDID | F-09: slot-based TTL per class |
| Operator exceeds cap via Agent | MachineDID | I-MACH-AGENT-2 |
| VC resolver shows "active" for ancestor-revoked DID | All | I-RESOLVER-1 |
| Epoch-boundary rate burst | BotDID | F-06-R: token bucket (no hard boundary) |
| Base NFT burned leaving ghost fractions | AssetDID | F-18: must burn all fractions first |
| Single oracle compromise | AssetDID | F-16: min 3 attestors for Crop |
| Infinite Service chain | ServiceDID | I-SVC-CHAIN ≤ 3 hops |
| Agent exceeds granted capability | AgentDID | I-AGENT-1: capability_set ⊆ owner's |
| DelegationToken with revoked scope | All | F-37: DelegationToken_valid uses current scope |
| Emergency vault reveals Master_KEK | PersonDID | T28.1: HKDF one-way; vault key ≠ master key |
| GDPR erasure claim vs on-chain hash | All | T28.2: hash ≠ personal data per GDPR Rec.26 |
| Posthumous impersonation | PersonDID | §32.1: death event requires OrgDID attestation + timelock |
| Minor bypass guardian control | PersonDID (minor) | §34.3: MinorControl predicate; guardian approval required |
| Tier 5: offline password brute-force | PersonDID | Argon2id 64 MiB memory-hard (I-TIER5-PW-2); 12+ char password ~60 bits → 10^18s at 0.3s/attempt |
| Tier 5: email account takeover → init + suppress cancel | PersonDID | Residual risk acknowledged; 21-day window; user should configure Tier 1/2/3 for higher security |
| Tier 5: EMAIL_ORACLE_PK compromise | PersonDID | EmailAccessProof forged → invalid Tier 5 inits; mitigation: oracle key rotation via PhoenixKey OrgDID governance; I-TIER5-EMAIL-2 |
| Recursive orphan chain (cha mẹ và con cùng chết) | PersonDID (minor/estate) | I-RECOVERY-5: beneficiary Posthumous → DID → Archived immediately |

---

## §28.1 Theorems (v4.3 additions) [N]

**Theorem 28.1 — Emergency Vault Key Isolation.**

*Statement.* Knowledge of `emergency_vault_key` does not reveal `Master_KEK`, `TAAD_Key`, `HW_Key`, or `Cloud_Secret`.

*Proof.*
`emergency_vault_key = HKDF(Master_KEK, "emergency-vault-v1", salt)` where HKDF is defined per RFC 5869 [https://www.rfc-editor.org/rfc/rfc5869] using HMAC-SHA256 as the PRF.

By the PRF security of HMAC-SHA256 (under the assumption that SHA-256 is a collision-resistant hash function and HMAC is a PRF, per Bellare et al. 1996 [https://dl.acm.org/doi/10.1145/234525.234532]):

`HKDF(key, info₁, salt)` is computationally indistinguishable from a random oracle over the output domain. Given `emergency_vault_key`, recovering `Master_KEK` requires inverting HKDF, which reduces to breaking HMAC-SHA256 as a PRF — computationally infeasible for 256-bit keys.

`TAAD_Key = Ed25519.FromSeed(HKDF(Master_KEK, "taad-controller-v1", H(DID)))` uses a different `info` string; the two HKDF outputs are independent (distinct `info` → distinct PRF evaluations). Similarly, `HW_Key` is generated independently in Secure Enclave and never derived from `emergency_vault_key`. `Cloud_Secret` is hardware Keychain-resident and not derivable from any HKDF output. ∎

**Theorem 28.2 — GDPR Tombstone Integrity.**

*Statement.* After a valid GDPR erasure operation on record `r`, no personal data contained in `r` is accessible from on-chain state, while proof of `r`'s existence is preserved.

*Proof.*
The on-chain state post-erasure is `TombstoneRecord = {record_id = H(content_r), erasure_slot, requester_did_hash, ...}` as defined in §34.1. The off-chain content `content_r` is deleted from all LampNet nodes holding encrypted fragments.

(1) **Personal data inaccessibility:** `record_id = H(content_r) = BLAKE2b-256(content_r)`. BLAKE2b-256 is a cryptographic hash function; inverting it to recover `content_r` requires O(2^256) operations (pre-image resistance, per Aumasson et al. 2013 [https://www.blake2.net/blake2.pdf]). The encrypted LampNet fragments are deleted; without the decryption key and fragments, `content_r` is computationally inaccessible. Therefore no personal data is accessible from on-chain state.

(2) **Proof of existence:** `TombstoneRecord` contains `record_id` (binding to the original content via collision resistance), `erasure_slot` (temporal proof), and `requester_did_hash` (accountability). These constitute proof that record `r` existed and was erased at `erasure_slot`.

(3) **GDPR compliance:** Per GDPR Recital 26 [https://gdpr-info.eu/recitals/no-26/], "anonymous information... does not relate to an identified or identifiable natural person." A cryptographic hash of deleted plaintext personal data, where the plaintext is computationally inaccessible, constitutes anonymous information. Therefore `TombstoneRecord` does not constitute personal data under GDPR. ∎

---

# PART VII — INTEGRATION

## §29 AppTokenEconomics Integration [N]

> **Reconciliation with MAGIC AppEconomics v2.1 (canonical).** The reward
> COMPUTATION engine is **MagicLampNetwork/MAGIC AppEconomics v2.1**:
> `computeW = V_d × Φ_util × Φ_users × Φ_dispute × κ_tier × Φ_age`, with
> iterative distribution capped at `MAX_SINGLE_APP_REWARD_BPS` (30%). That
> engine is **DID-agnostic** and lives in the MAGIC repo — it is shared by any
> DID method, not just PhoenixKey. This §29 is **only the PhoenixKey-side
> DID-mapping layer**: it defines which DID / ServiceDID is the app actor that
> the v2.1 engine credits (ATE-1/2/3), and the Genesis protocol (§29.3).
>
> The legacy `holder_share_bps` / `C-TOKEN-CONFIG-2 ≥ 1000` reward semantics
> referenced below come from the earlier "AppTokenEconomics v1.0-SA" draft and
> are **superseded** by the v2.1 distribution cap. They are retained here only
> for the type extensions (ServiceDID/OrgDID holder identities) that thread
> PhoenixKey DIDs into the holder model. **Reward percentages MUST follow MAGIC
> AppEconomics v2.1, not the v1.0-SA constants.** Boundary rule: PhoenixKey
> supplies the DID↔actor mapping (and `Attr*`, §25); MAGIC owns the math.

### §29.1 DID-mapping extensions to the holder model (was: "v1.0-SA changes")

```
-- ATE-1: AppTokenConfig adds service_did:
type AppTokenConfig {
  ...existing fields...
  service_did : ByteArray,   -- NEW: ServiceDID of the app
}

-- ATE-2: HolderIdentity extended:
type HolderIdentity {
  | PersonHolder(pkh: VerificationKeyHash)   -- existing
  | OrgHolder(org_did: ByteArray)            -- NEW
  | ServiceHolder(svc_did: ByteArray)        -- NEW
}

-- ATE-3: InfrastructureApp exception:
type AppRegistrationType {
  | StandardApp(AppTokenConfig)             -- existing (ORILIFE, ALADINWORK)
  | InfrastructureApp {                     -- NEW: PhoenixKey
      service_did: ByteArray,
      treasury_address: ByteArray,
      governance: ByteArray,
    }
}
-- InfrastructureApp exempt from C-TOKEN-CONFIG-2 (holder_share_bps ≥ 1000).
```

### §29.2 CropNFT → AssetDID Formalization

```
CropDatum (AppTokenEconomics §7.1) ≜
  AssetDID {
    asset_class    = Crop(species, planted_slot),
    attestations   = [OriLife_att₁, OriLife_att₂, AgriBot_att],  -- ≥ 3 required
    tokenized      = fraction_policy_id,
    transferable   = True,
  } ∪ harvest_lifecycle_fields   -- expected_harvest_epoch, total_proceeds, etc.
```

### §29.3 PhoenixKey ServiceDID Genesis Protocol

```
Bootstrap (one-time ceremony):
  1. Greensun Tech founders (≥ 3) form GenesisOrgDID (threshold 2-of-3)
  2. GenesisOrgDID creates PhoenixKey ServiceDID
  3. Registered as InfrastructureApp in AppTokenEconomics
  4. GenesisOrgDID governs TRUSTED_PCR_REGISTRY (F-21)
  5. Genesis Tx recorded in Cardano blockchain + DID Method Spec §3

  Risk mitigation: PhoenixKey ServiceDID cannot be unilaterally revoked by any single founder.
  Governance: constitutional threshold (⌊2n/3⌋ + 1 of GenesisOrgDID members) for revocation.
```

---

## §30 PPA Claim Mapping [I]

| PPA Claim | v4.1 Final Section | DID Type | Status |
|-----------|-------------------|----------|--------|
| Claim 1 (Key Management) | §6, §10 | PersonDID | Existing; amended |
| Claim 2 (System Architecture) | §2, §4 | All | Existing |
| Claim 3 (Tiered Recovery) | §11 | PersonDID | Existing |
| Claim 4 (Environmental Attestation) | §11.5 | PersonDID | Existing |
| Claim 13 (Export) | §9.1 | PersonDID | Amended |
| Claim 15 (Multi-Key) | §6.3 | PersonDID | Amended |
| Claim 16 (Delegation) | §23 | All | Amended |
| **Claim 17** | §18 | BotDID | New |
| **Claim 18** | §15 | DeviceDID | New |
| **Claim 19** | §16 | MachineDID | New |
| **Claim 20** | §17 | AssetDID | New |
| **Claim 21** | §19 | AgentDID | New |
| **Claim 22** | §20 | ServiceDID | New |
| **Claim 23** | §21 | CharDID | New |
| **Claim 24** | §13 | OrgDID | New |
| **Claim 25** | §14 | ContextDID | New |

---

## §31 Protocol Boundary [N]

This section classifies all components of the PhoenixKey ecosystem by openness level. This classification governs licensing, interoperability, and independent audit requirements.

### §31.1 Fully Open — Apache 2.0 License

The following components are fully specified in this document and must be open for independent implementation and auditing:

| Component | Sections | Rationale |
|-----------|----------|-----------|
| TAAD Standard (AuthToken singleton, ASM A-1..A-6) | §10 | Interoperability requires public standard |
| Guardian Protocol (threshold recovery, collateral, slashing) | §11.2, §11.3 | User safety requires auditable guardian system |
| Guardian Incentive Model (Nash equilibrium, alert reward) | §11.3.1 | Economic analysis must be public |
| All 10 DID Type Specifications | §12–§21 | Ecosystem composability |
| Authority Model and Capability System | §3, §4 | Smart contract compatibility |
| Ownership Graph and Delegation Protocol | §22, §23 | Cross-app interoperability |
| MAGIC Attribution (Attr* function) | §25 | AppTokenEconomics integration |
| LT Codes Algorithm structure | §7.1 | Reference implementation must be auditable |
| Cardano wallet derivation (CIP-1852 paths) | §8 | Standard compatibility |
| Seed export/import protocol | §9 | User sovereignty |
| AppTokenEconomics extensions (ATE-1..3) | §29 | Economic integration |

### §31.2 Semi-Open — Documented, Reference Implementation Available

The following are specified in this document but implementation details are reference-quality and may be optimized by MagicLamp:

| Component | Sections | Notes |
|-----------|----------|-------|
| Master KEK derivation paths | §6 | HKDF parameters are public; optimization space in Secure Enclave binding |
| LampNet storage parameters (k=50, n=1000) | §7.2 | Functional parameters public; routing optimization proprietary |
| Device_KEK salt construction | §6.2 | Fully specified; implementation may add hardware-specific binding |
| Guardian collateral amounts | §11.3 | MIN_GUARDIAN_COLLATERAL is a reference; governance may tune |

### §31.3 Proprietary — MagicLamp Only

The following are referenced by this spec but NOT specified here. They are proprietary to MagicLamp and/or VeData:

| Component | Location | Notes |
|-----------|----------|-------|
| LampNet routing parameters (K, β, c, δ) | LampNet Network Spec | Competitive advantage |
| LocatorSecret additional salt values | MagicLamp implementation | Anti-analysis |
| VeData AI model and liveness detection | VeData Formal Spec | Separate patent (VeData application) |
| Anti-deepfake biometric verification internals | VeData Formal Spec | Proprietary sensing algorithms |
| Environmental signal processing details | VeData Formal Spec | Proprietary signal fusion |

### §31.4 Explicitly Out of Scope

The following topics are intentionally not covered in this document:

| Topic | Reason | Reference |
|-------|--------|-----------|
| DID resolver protocol (I-RESOLVER-1 implementation) | Separate spec | PhoenixKey DID Method Specification v1.0 |
| Aiken/PlutusV3 validator code | Separate spec | PhoenixKey Cardano Validator Reference v1.0 |
| UI/UX flows and screen design | Not mathematical | PhoenixKey UI Specification v1.4.x |
| PostgreSQL schema | Not mathematical | PhoenixKey DB v1.5 |
| Service pricing table (MAGIC per operation) | Operational decision | Governance proposal |
| Mechanized proofs (Coq/Isabelle) | Target v5.0 | All theorems are informal + proof sketch |
| Cross-chain / bridge semantics | Future work | v5.0 target |
| BCH biometric parameters | VeData scope | Not used as key material in PhoenixKey |

---

# PART VIII — LIFECYCLE COMPLETENESS *(v4.3)*

## §32 End-of-Life Protocols [N]

### §32.1 PersonDID — Posthumous Protocol

```
-- EstateConfig: designated before death; stored in TAADDatum.
EstateConfig ≜ {
  beneficiary_did    : DID,              -- PersonDID or OrgDID to receive estate access
  estate_guardian_did: Option<DID>,      -- executor (legal representative); may differ from beneficiary
  death_timelock     : SlotNo,           -- DEATH_TIMELOCK = 30 × 432_000 ≈ 30 days
  scope_granted      : Set<Capability>,  -- capabilities transferred to beneficiary (strict subset)
  -- Typically: {Read_DID, Sign_Tx(max_inheritance_value)}; NOT Full_Authority
}

-- TAADState extended (v4.5): Posthumous is now a TAADState variant (§10.1).
-- Validator sets TAADDatum.state ← Posthumous{...} upon finalization.

-- Op_declare_death: triggered by OrgDID with death certificate authority.
Op_declare_death(did: PersonDID, death_cert_org: OrgDID, cert_hash: {0,1}^256, s: SlotNo):
  Pre:
    death_cert_org.org_class ∈ {Government}
    ∧ Authority(death_cert_org, Write_DID(did), s)
    ∧ Active(did, s) ∨ Suspended(did)   -- works even if suspended
    ∧ TAADDatum.estate_config ≠ None
  Effect:
    TAADDatum.state ← Recovering {
      pending_hw_pub    = NULL_KEY,     -- no new hardware key
      recovery_deadline = s + TAADDatum.estate_config.death_timelock,
      verification_tier = POSTHUMOUS_TIER,  -- §1.4: ≜ 100; distinct from capability tiers and recovery tiers 1,2,3,5
      crypto_proofs     = cert_hash,
    }
    death_cert_hash ← cert_hash
    seq++

-- Op_finalize_posthumous: after timelock; no cancellation possible.
Op_finalize_posthumous(did: PersonDID, s: SlotNo):
  Pre:
    TAADDatum.state = Recovering { verification_tier = POSTHUMOUS_TIER }
    ∧ s ≥ recovery_deadline
  Effect:
    TAADDatum.state ← Posthumous(
      beneficiary   = estate_config.beneficiary_did,
      confirmed_slot = s
    )
    -- beneficiary inherits scope_granted; Full_Authority NOT transferable
    -- PersonDID remains resolvable (historical records stay verifiable)
    seq++

-- Posthumous constraints:
I-POST-1: Posthumous PersonDID CANNOT create new records (write-blocked)
I-POST-2: Posthumous PersonDID CANNOT issue new VCs
I-POST-3: beneficiary can READ records per estate_config.scope_granted
I-POST-4: estate_config.scope_granted ⊊ {Full_Authority}  -- Full_Authority never transfers
I-POST-5: On-chain records about did remain accessible per creator liability period (§34.2)
I-POST-6: If TAADDatum.estate_config = None and death declared → DID transitions to Revoked
          (no beneficiary; records remain on-chain but no new controller)
```

### §32.2 PersonDID — Incapacity / Conservatorship

```
-- Incapacity: medical/legal incapacity; different from guardian suspension.
-- Requires: court order (OrgDID of court) + medical certificate (OrgDID of hospital).
-- Duration: time-bounded; renewable.

IncapacityConfig ≜ {
  conservator_did    : DID,              -- court-appointed conservator (PersonDID or OrgDID)
  conservator_scope  : Set<Capability>, -- strict subset; excludes: access to medical records
  court_order_hash   : {0,1}^256,       -- H(court order document)
  medical_cert_hash  : {0,1}^256,       -- H(medical certificate)
  valid_until        : SlotNo,          -- bounded; must be renewed
  issuing_court      : OrgDID,
}

-- TAADState extended (v4.5): Incapacitated is now a TAADState variant (§10.1).
-- Validator sets TAADDatum.state ← Incapacitated{...} upon declaration.

Op_declare_incapacity(did: PersonDID, config: IncapacityConfig, s: SlotNo):
  Pre:
    Authority(config.issuing_court, Write_DID(did), s)
    ∧ config.issuing_court.org_class = Government
    ∧ config.conservator_scope ⊊ scope(did)
    ∧ Medical_Privacy ∉ config.conservator_scope   -- conservator cannot access private medical records
    ∧ config.valid_until > s + 432_000             -- min 1 epoch
  Effect:
    TAADDatum.state ← Incapacitated(config, s)
    -- Conservator gains conservator_scope for duration
    seq++

Op_restore_capacity(did: PersonDID, restoration_cert_hash: {0,1}^256, s: SlotNo):
  Pre:
    TAADDatum.state = Incapacitated(config, _)
    ∧ Authority(config.issuing_court, Write_DID(did), s)
    -- Restoration requires same court that declared incapacity
  Effect:
    TAADDatum.state ← Active
    incapacity_history.append({config, restoration_slot: s})  -- off-chain record
    seq++

-- Auto-expiry: if s > config.valid_until → state transitions to Active (no tx needed)
-- TAAD validator enforces: Incapacitated(config) ∧ s > config.valid_until → treat as Active

I-INCAP-1: Incapacity does NOT allow conservator to TAAD_Cancel_Recovery
           (TAAD Cancel is PersonDID_Privileged_Ops; requires hw_pub sig)
I-INCAP-2: Incapacity does NOT transfer Full_Authority
I-INCAP-3: conservator_scope excludes D9=private records of any domain
I-INCAP-4: Multiple incapacity declarations require sequential validity
           (cannot have overlapping incapacity periods)
```

### §32.3 OrgDID — Dissolution Protocol

```
-- OrgDID dissolution: formal wind-down; different from Revoke.
-- Records created by OrgDID remain verifiable after dissolution (archival).

DissolutionConfig ≜ {
  dissolution_authority : OrgDID,           -- parent OrgDID or governing body
  asset_distribution    : Map<DID, {0,1}^256>,  -- member DID → asset claim hash
  record_archive_did    : Option<DID>,      -- OrgDID that takes custody of records
  dissolution_slot      : SlotNo,           -- when dissolution takes effect
  notice_period         : SlotNo,           -- min 30 × 432_000 (30 days)
}

Op_initiate_dissolution(org: OrgDID, config: DissolutionConfig, s: SlotNo):
  Pre:
    constitutional_threshold(|org.members|) members approve
    ∧ config.dissolution_slot ≥ s + config.notice_period
    ∧ all_assets_accounted(org, config.asset_distribution)
    -- all child AssetDIDs either transferred or destruction-scheduled
  Effect:
    org.state ← Dissolving(config)
    seq++

Op_complete_dissolution(org: OrgDID, s: SlotNo):
  Pre:
    org.state = Dissolving(config)
    ∧ s ≥ config.dissolution_slot
    ∧ ∀ child ∈ children(org): transferred(child) ∨ revoked(child)
  Effect:
    org.state ← Archived(archived_slot = s)
    -- Archived ≠ Revoked: DID resolves to {status: "archived", ...}
    -- Records verifiable; new operations blocked
    seq++

-- Archived OrgDID invariants:
I-DISS-1: Archived OrgDID cannot issue new VCs
I-DISS-2: Archived OrgDID cannot create new child DIDs
I-DISS-3: Records bearing org's producer_id remain verifiable indefinitely
I-DISS-4: Creator liability period (§34.2) continues for archived OrgDID records
I-DISS-5: constitutional_threshold(n) = ⌊2n/3⌋ + 1 required for dissolution initiation
```

### §32.4 AssetDID — Physical Destruction Protocol

```
-- Physical destruction: fire, crash, consumption, expiry.
-- Destruction creates an immutable record; DID not reused.

DestructionRecord ≜ {
  asset_did       : DID,
  destruction_type: Fire|Accident|Consumption|NaturalExpiry|Confiscation|Other,
  destruction_slot: SlotNo,
  evidence_hash   : {0,1}^256,    -- H(photographic/video evidence, off-chain on LampNet)
  attester        : DID,           -- ServiceDID or OrgDID; minimum 1 attester
  insurance_claim : Option<{did: DID, claim_ref: ByteArray}>,  -- AssetDID of insurance policy
}

Op_destroy_asset(asset: AssetDID, record: DestructionRecord, s: SlotNo):
  Pre:
    Authority(record.attester, Attest_Asset(asset.asset_class), s)
    ∧ Active(asset, s)
    ∧ record.destruction_slot ≤ s
    -- Tokenized assets: all fraction tokens must be burned first (F-18)
    ∧ (asset.tokenized = None ∨ all_fractions_burned(asset))
  Effect:
    asset.revoked_slot ← s
    DestructionRecord written on-chain (LampNet anchor)
    -- asset.owner's insurance_claim reference updated if provided

I-DESTR-1: Destroyed AssetDID cannot be transferred or re-activated
I-DESTR-2: evidence_hash must reference off-chain evidence stored on LampNet
I-DESTR-3: If insurance_claim ≠ None: insurance AssetDID must be active at destruction time
I-DESTR-4: Historical records about destroyed asset remain accessible per liability period
```

### §32.5 DeviceDID — End-of-Life / Decommission Protocol

```
-- Device decommission: secure wipe, key destruction, hardware disposal.

DecommissionRecord ≜ {
  device_did      : DID,
  decommission_type: SecureWipe|LostOrStolen|HardwareFailure|UpgradedDevice|Other,
  wipe_cert_hash  : Option<{0,1}^256>,  -- H(secure wipe certificate)
  new_device_did  : Option<DID>,        -- if UpgradedDevice: successor DeviceDID
  hw_key_destroyed: 𝔹,                 -- True iff HW_Key confirmed destroyed
  performed_slot  : SlotNo,
}

Op_decommission_device(device: DeviceDID, record: DecommissionRecord, s: SlotNo):
  Pre:
    Authority(device.owner, Write_DID(device), s)
    ∧ Active(device, s)
    ∧ record.performed_slot ≤ s
  Effect:
    device.revoked_slot ← s
    DecommissionRecord written on-chain
    -- If UpgradedDevice: all capabilities migrated to new_device_did
    -- LocatorSecret for decommissioned device invalidated on LampNet

I-DECOMM-1: Decommissioned DeviceDID cannot be reactivated
I-DECOMM-2: Decommission of a LampNetNode DeviceDID requires transfer of stored fragments
            to replacement node before decommission completes
I-DECOMM-3: If hw_key_destroyed = False: security_note logged on-chain
I-DECOMM-4: new_device_did ≠ device.did (cannot "replace" with self)
```

---

## §33 Emergency Vault [N]

### §33.1 Key Derivation

```
-- Emergency Vault Key: derived from Master_KEK; isolated from TAAD_Key and HW_Key.
-- Purpose: decrypt emergency medical data WITHOUT blockchain query.
-- See Theorem 28.1 for isolation proof.

EmergencyVaultKey(did: PersonDID) → {0,1}^256 ≜
  HKDF(
    key  = Master_KEK,
    info = "emergency-vault-v1",
    salt = H(did ∥ "ev-salt-v1")
  )
-- RFC 5869: https://www.rfc-editor.org/rfc/rfc5869
-- Re-derived after every key rotation (§11.4 post-recovery step 2b: EmergencyVaultKey updated).
-- Never stored externally; always re-derived from Master_KEK.
-- NOTE: updating I-PERSON-10 requirement: after Op_export_seed, EmergencyVaultKey MUST be rotated.
```

### §33.2 Vault Structure

```
EmergencyVaultData ≜ {
  blood_type        : Option<BloodType>,         -- ABO+Rh; e.g., "A+"
  allergies         : List<ByteArray>,           -- plaintext strings; e.g., ["penicillin", "latex"]
  critical_meds     : List<ByteArray>,           -- current critical medications
  chronic_conditions: List<ByteArray>,           -- conditions emergency physician must know
  emergency_contacts: List<{name: ByteArray, phone: ByteArray}>,
  dnr_status        : Option<{status: 𝔹, document_hash: {0,1}^256}>,  -- Do Not Resuscitate
  organ_donor       : Option<𝔹>,
  physician_contact : Option<ByteArray>,         -- primary care physician contact
  last_updated_slot : SlotNo,
  version           : ℕ,
}

-- Vault is encrypted before storage:
EmergencyVaultCiphertext ≜ Enc(EmergencyVaultKey, serialize(EmergencyVaultData))
-- AES-256-GCM; authenticated; nonce = H(EmergencyVaultKey ∥ version ∥ "ev-nonce")

-- Storage (TWO redundant paths):
-- Path A: Encrypted bytes in QR code on device lock screen
-- Path B: Encrypted fragments on LampNet (locator = H(EmergencyVaultKey ∥ "ev-locator"))
```

### §33.3 Access Protocol (Offline Capable)

```
-- Emergency access: NO blockchain query in critical path.
-- Design principle: Safety > Security in emergency path (see §33.4 for accountability).

EmergencyAccess(qr_or_nfc_payload: ByteArray):
  -- Step 1: Decode EmergencyVaultCiphertext from QR/NFC
  ciphertext = decode(qr_or_nfc_payload)

  -- Step 2: Derive decryption key (offline; requires Master_KEK on device)
  -- OR: accept pre-printed decryption key in emergency wallet card
  vault_key = EmergencyVaultKey(did)   -- requires device access
  -- Alternative: owner pre-generates a "emergency access token" = EmergencyVaultKey
  --              and prints/stores in physical emergency card (NOT same as Master_KEK)

  -- Step 3: Decrypt and display (offline; no internet required)
  data = Dec(vault_key, ciphertext)
  Display(data.blood_type, data.allergies, data.critical_meds, data.conditions)

  -- Step 4: Log access (async; when network available)
  EmergencyAccessLog.append({
    accessor_credential: optional,      -- physician ID if available
    access_slot:         current_slot ?? estimated_slot,
    data_fields_accessed: [field_names],
    access_method:       QR | NFC | LampNet,
    device_id_hash:      H(accessor_device_id),  -- anonymous device fingerprint
  })
  -- Log written to LampNet; owner notified asynchronously

-- I-EVAULT-1: NO credential check in emergency access path
-- I-EVAULT-2: Access log submitted asynchronously; cannot block access
-- I-EVAULT-3: EmergencyVaultData MUST NOT contain: financial keys, seed phrase, TAAD_Key
-- I-EVAULT-4: EmergencyVaultCiphertext MUST be re-encrypted when EmergencyVaultKey rotates
-- I-EVAULT-5: LampNet path MUST use separate locator from EncSeed locator (isolation)
```

### §33.4 Emergency Access Accountability

```
-- Trade-off formalized:
SecurityVsSafety ≜
  IF emergency_path:
    Safety = Maximize (data accessible in < 5 seconds)
    Security = Async (accountability after the fact)
  ELSE:
    Security = Maximize (credential verification before access)

-- Accountability mechanism:
-- 1. All emergency accesses logged (async) on LampNet
-- 2. Owner notified when network connectivity restored
-- 3. Accessor credential (if provided) matched against medical license registry
-- 4. Invalid access (non-medical personnel) → flag for audit
-- 5. Audit trail: immutable on LampNet; anchored to Cardano per batch

-- No penalty for accessing in genuine emergency (safety principle)
-- Potential penalty for accessing without clinical necessity (accountability principle)
```

---

## §34 Privacy and Compliance [N]

### §34.1 GDPR Tombstone Pattern [N]

```
-- GDPR Article 17 (Right to Erasure): https://gdpr-info.eu/art-17-gdpr/
-- Applicable when: jurisdiction_flags contains "EU" or equivalent
--                  AND record.domain has no Art.17(3) exception

JurisdictionCode ≜ ByteArray   -- ISO 3166-1 alpha-2; e.g., "VN", "EU", "US", "SG"

GDPRException ≜
  | FreedomOfExpression
  | LegalObligation         -- creator liability period (§34.2)
  | PublicHealthResearch
  | ArchivingPublicInterest
  | LegalDefense

TombstoneRecord ≜ {
  record_id          : {0,1}^256,     -- H(original_content); no personal data (T28.2)
  created_slot       : SlotNo,        -- original creation time
  erasure_slot       : SlotNo,        -- when erasure was executed
  requester_did_hash : {0,1}^256,     -- H(subject_did); not the DID itself
  erasure_reason     : GDPRArticle17Sub,  -- Art.17 sub-paragraph
  jurisdiction       : JurisdictionCode,
  exception_applied  : Option<GDPRException>,
}

Op_request_erasure(subject_did: PersonDID, record_id: {0,1}^256, reason: GDPRArticle17Sub, s):
  Pre:
    subject_did = record.entity_id               -- only subject can request erasure of own records
    ∧ jurisdiction_applicable(subject_did, "EU") -- GDPR applies
    ∧ exception_applies(record, reason) = None   -- no Art.17(3) exception
    -- Creator liability period is an exception: cannot erase if within liability period (§34.2)
  Effect:
    LampNet: delete all encrypted fragments of record
    On-chain: write TombstoneRecord (no personal data)
    -- record_id = H(original_content) remains; proves event existed (T28.2)
    seq++ on subject's TAAD (erasure audit trail)

-- Erasure exceptions (Art.17(3)):
exception_applies(record, reason) : Option<GDPRException> ≜
  IF record.domain ∈ Medical ∧ creator_liability_active(record, s):
    RETURN Some(LegalObligation)    -- §34.2: creator must retain for liability
  ELIF record.domain ∈ LegalProceedings:
    RETURN Some(LegalDefense)
  ELIF record.archetype ∈ {LEGAL_DECLARATION, COMPLIANCE_CHECKPOINT}:
    RETURN Some(ArchivingPublicInterest)  -- public interest records
  ELSE: RETURN None

-- See Theorem 28.2 for proof that TombstoneRecord is GDPR-compliant.

I-GDPR-1: Only subject PersonDID can initiate erasure of records where entity_id = subject
I-GDPR-2: TombstoneRecord MUST NOT contain personal data (enforced by using H(content))
I-GDPR-3: exception_applies check is mandatory before erasure; cannot be bypassed
I-GDPR-4: Erasure request itself is logged in subject's audit trail (accountability)
I-GDPR-5: D9=public records: erasure possible but third-party caches may have copies;
          system can only control its own LampNet nodes
```

### §34.2 Creator Liability Period Access [N]

```
-- Professional creators retain READ-ONLY access to records they created
-- during the applicable liability period, regardless of subject revocation.
-- Legal basis: professional liability law; medical records retention requirements.
-- References:
--   Vietnam Decree 96/2023/ND-CP Art.59 (medical records retention)
--   WHO Medical Records Manual 2006: https://www.who.int/healthinfo/documentation/who_mrd_06.1_eng.pdf
--   HIPAA 45 CFR §164.530(j) (US medical records)

Domain ≜ Medical | Education | Legal | Employment | Financial | Environmental | General

LIABILITY_PERIOD_SLOTS : Map<Domain, ℕ> ≜ {
  Medical      → 10 × 365 × 17_280,   -- 10 years in slots (~5 days/epoch, 17280 slots/day)
  Education    → 5  × 365 × 17_280,   -- 5 years
  Legal        → 30 × 365 × 17_280,   -- 30 years
  Employment   → 5  × 365 × 17_280,   -- 5 years
  Financial    → 7  × 365 × 17_280,   -- 7 years (typical AML/KYC retention)
  Environmental→ 10 × 365 × 17_280,   -- 10 years (carbon credits, environmental compliance)
  General      → 3  × 365 × 17_280,   -- 3 years
}
-- [PARAM]: jurisdiction-specific overrides via JurisdictionPolicy (§34.4)

creator_liability_active(record, s: SlotNo) : 𝔹 ≜
  s < record.created_slot + LIABILITY_PERIOD_SLOTS[record.domain]

CreatorAccess(record, requester_did: DID, s: SlotNo) : 𝔹 ≜
  record.producer_id = requester_did
  ∧ creator_liability_active(record, s)
  -- READ-ONLY: creator cannot share with third parties without subject consent
  -- READ-ONLY: creator cannot modify or delete

-- Creator access constraints:
I-CREAT-1: CreatorAccess grants READ permission ONLY; no write, no share
I-CREAT-2: Creator sharing requires explicit subject consent (new DelegationToken from subject)
I-CREAT-3: CreatorAccess persists even after subject revokes standard access
I-CREAT-4: CreatorAccess does NOT override GDPR erasure exceptions check (§34.1)
           -- If creator is within liability period → erasure blocked (GDPRException.LegalObligation)
I-CREAT-5: After liability period expires: creator access reverts to standard subject-controlled model
```

### §34.3 Minor Guardian Model [N]

```
-- GDPR Article 8: https://gdpr-info.eu/art-8-gdpr/ (parental consent for minors)
-- UNCRC Article 5: https://www.ohchr.org/en/instruments-mechanisms/instruments/convention-rights-child
-- Vietnam Civil Code 2015 Art.21: minors under 18 have limited civil capacity

-- Age thresholds (slot-based; jurisdiction-specific via §34.4):
MINOR_THRESHOLD_SLOTS    : ℕ ≜ 18 × 365 × 17_280   -- 18 years (default; most jurisdictions)
CO_CONSENT_ONSET_SLOTS   : ℕ ≜ 14 × 365 × 17_280   -- 14 years (co-consent begins)
-- [PARAM]: overridden by JurisdictionPolicy.minor_age_slots

-- Minor control predicate (replaces standard Authority for PersonDID with guardian_did ≠ None):
MinorControl(did: PersonDID, op: Capability, s: SlotNo) : 𝔹 ≜
  IF guardian_did(did) = None:
    RETURN standard Authority predicate (§4.2)   -- adult; no guardian
  ELSE:
    LET age_slots = s - birth_slot(did) ?? 0  -- if birth_slot None: assume adult (conservative)
    IN
    IF birth_slot(did) = None:
      RETURN standard Authority predicate      -- unknown age: treat as adult
    ELIF age_slots < CO_CONSENT_ONSET_SLOTS:
      -- Guardian is primary controller; minor has read-only access
      RETURN Authority(guardian_did(did), op, s)
    ELIF age_slots < MINOR_THRESHOLD_SLOTS:
      -- Co-consent: minor expresses preference, guardian approves
      RETURN Biometric_HW_sig(did, op, s) ∧ Guardian_approval(guardian_did(did), op, s)
    ELSE:
      -- Auto-transfer: age ≥ threshold → guardian_did no longer applies
      RETURN standard Authority predicate      -- minor is now adult

-- Auto-transfer (no transaction required):
-- TAAD validator recognizes age_slots ≥ MINOR_THRESHOLD_SLOTS → ignores guardian_did
-- On-chain cleanup: owner executes Op_remove_guardian_did to formally clear datum
Op_remove_guardian_did(did: PersonDID, s: SlotNo):
  Pre:
    s - birth_slot(did) ≥ MINOR_THRESHOLD_SLOTS
    ∧ Biometric_HW_sig(did, Remove_Guardian, s)
  Effect:
    guardian_did ← None; seq++

-- Guardian_approval predicate:
Guardian_approval(guardian_did: DID, op: Capability, s: SlotNo) : 𝔹 ≜
  Authority(guardian_did, op, s)   -- guardian must have active authority for the op
  -- Guardian signs a co-consent DelegationToken for the specific op

-- Edge cases:
-- Orphaned minor (no guardian DID): court-appointed guardian → OrgDID of court as guardian_did
-- Emancipated minor (married, court order): Op_remove_guardian_did with court OrgDID attestation
-- Conflicting guardian and minor (14-17): minor can appeal to court (off-chain; not protocol)

I-MINOR-1: guardian_did ≠ None → MinorControl applies (replaces Authority)
I-MINOR-2: guardian_did PersonDID must be active and adult (guardian_did.guardian_did = None)
I-MINOR-3: Auto-transfer is enforced by TAAD validator without on-chain transaction
I-MINOR-4: Guardian cannot grant capabilities exceeding their own scope to the minor
I-MINOR-5: Minor's records created during minority: subject remains entity_id; guardian_did is creator_access holder
```

### §34.4 Cross-Jurisdiction Flags [N]

```
-- Jurisdiction-specific legal requirements vary significantly.
-- PhoenixKey encodes applicable jurisdictions per DID to enable correct compliance enforcement.

JurisdictionPolicy ≜ {
  code               : JurisdictionCode,   -- ISO 3166-1; "EU" for all EU member states
  gdpr_applicable    : 𝔹,
  data_residency     : Option<Region>,     -- required data residency for LampNet nodes
  minor_age_slots    : ℕ,                 -- jurisdiction-specific age of majority
  liability_period_overrides : Map<Domain, SlotNo>,  -- override LIABILITY_PERIOD_SLOTS
  specific_laws      : List<ByteArray>,   -- e.g., ["HIPAA", "PDPA_SG", "NĐ13/2023_VN"]
}

-- Reference jurisdictions:
JURISDICTIONS : Map<JurisdictionCode, JurisdictionPolicy> ≜ {
  "EU" → { gdpr_applicable=True, minor_age_slots=16×365×17_280,   -- GDPR Art.8: 16 for most EU states
            liability_period_overrides={Medical: 10×365×17_280} },
  "VN" → { gdpr_applicable=False,  -- Decree 13/2023/ND-CP applies instead
            minor_age_slots=18×365×17_280,
            liability_period_overrides={Medical: 20×365×17_280},  -- Decree 96/2023
            specific_laws=["NĐ13/2023/ND-CP", "NĐ96/2023/ND-CP"] },
  "US" → { gdpr_applicable=False,  -- HIPAA applies for medical
            minor_age_slots=18×365×17_280,
            liability_period_overrides={Medical: 10×365×17_280},  -- HIPAA minimum
            specific_laws=["HIPAA_45CFR164"] },
  "SG" → { gdpr_applicable=False,  -- PDPA applies
            minor_age_slots=18×365×17_280,
            specific_laws=["PDPA_SG_2012"] },
}
-- [PARAM]: JurisdictionPolicy values are governance parameters; updated via OrgDID governance

-- Applicable policy resolution:
resolve_jurisdiction_policy(did: DID) → List<JurisdictionPolicy> ≜
  [JURISDICTIONS[code] : code ∈ did.jurisdiction]
  -- Multiple jurisdictions may apply (e.g., EU citizen living in Vietnam)
  -- Strictest applicable rule wins (conservative compliance)

-- When jurisdiction_flags is empty: no GDPR obligation (but other laws may apply)
-- Platform SHOULD prompt user to set jurisdiction_flags at registration

I-JURI-1: jurisdiction_flags ⊆ KNOWN_JURISDICTION_CODES (validated at DID creation)
I-JURI-2: "EU" in jurisdiction_flags → GDPR tombstone pattern MUST be supported for this DID
I-JURI-3: data_residency ≠ None → LampNet MUST store fragments only in compliant nodes
I-JURI-4: Jurisdiction updates require Biometric_HW_sig + optional guardian approval (if minor)
```

---

## §35 Layer 0 Architecture Boundary [N]

### §35.1 PhoenixKey as Identity Foundation

```
-- PhoenixKey is the Layer 0 identity infrastructure.
-- All other MagicLamp applications USE PhoenixKey; they do not extend it.

Layer 0 — Identity (PhoenixKey):
  Provides: DID (did:phoenix:SLOT:HASH), TAAD authority management,
            VC issuance/verification, access control via signed messages
  Does NOT provide: data storage, record format, business logic

Layer 1 — Data Layer (VeData/RealStamp):
  Provides: record types (19 archetypes), trust scoring, Mosaic anchoring
  Depends on: PhoenixKey DID as entity_id and producer_id

Layer 2 — Business Logic (GenieTask, AladinEye, OriLife, etc.):
  Provides: domain-specific features
  Depends on: PhoenixKey (identity) + VeData (data)

Layer 3 — Storage (LampNet):
  Provides: distributed storage, CID-addressed content
  Depends on: PhoenixKey DID as access control reference
```

### §35.2 Responsibility Boundary [N]

```
-- Formal boundary: what PhoenixKey MUST and MUST NOT do.

PhoenixKey MUST:
  [M-1] Provide DID creation, resolution, deactivation per did:phoenix spec
  [M-2] Enforce TAAD authority (controller_pkh verification) for all identity operations
  [M-3] Provide VC issuance and verification using PhoenixKey DID as root of trust
  [M-4] Enforce access control (grant/revoke) via signed DelegationTokens
  [M-5] Enforce lifecycle protocols (§32): posthumous, incapacity, dissolution, destruction
  [M-6] Provide emergency vault key derivation (§33)
  [M-7] Enforce jurisdiction-appropriate compliance (§34)

PhoenixKey MUST NOT:
  [MN-1] Store application data (records, attestations, credentials content)
           → These belong to LampNet/VeData
  [MN-2] Define record formats (archetypes, DataSpec)
           → These belong to RealStamp Math Spec
  [MN-3] Implement trust scoring (V(r), Bayesian update)
           → These belong to Trust Score Module
  [MN-4] Implement task payment logic (TaskUTxO, escrow)
           → These belong to GenieTask
  [MN-5] Implement achievement logic (GenieJem NFTs)
           → These belong to GenieJem/GenieTask
  [MN-6] Implement domain-specific business rules
           → These belong to domain application specs
```

### §35.3 Interaction Protocol [N]

```
-- How applications interact with PhoenixKey (normative):

Application → PhoenixKey:
  (a) Verify DID ownership: request signed challenge from DID controller
      challenge = H("auth" ∥ app_id ∥ nonce ∥ s)
      valid = Verify(controller_pkh, challenge, sig)

  (b) Issue VC: app (as ServiceDID or OrgDID) signs VC addressed to PersonDID
      VC.issuer = app_did; VC.subject = person_did
      VC.proof = Sign(app_service_key, H(VC_fields))

  (c) Verify VC: resolve issuer DID; check issuer Active; verify signature
      valid = Active(issuer_did, s) ∧ Verify(issuer.service_key, H(VC_fields), VC.proof)

  (d) Request access: PersonDID grants DelegationToken to ServiceDID
      DelegationToken { from: person_did, to: service_did, granted_cap, valid_until }

  (e) Record entity_id: use PhoenixKey DID as entity_id in RealStamp records
      record.entity_id = person_did   -- no additional PhoenixKey call needed

-- PhoenixKey does NOT have an oracle or callback mechanism.
-- All interactions are pull-based (resolve) or push-based (sign).
-- No PhoenixKey private data is accessible to applications without explicit grant.
```

---

## §36 Transaction Fee Architecture [N] *(v4.6)*

This section defines the **PhoenixKey fee** charged by every state-changing
PhoenixKey operation, and the **split** of that fee between the **Cardano
Treasury** (via the Conway-era native treasury-donation field) and the
**Phoenix Treasury** (a Plutus V3 script address governed by PhoenixKey
OrgDID multisig).

The fee is **in addition to**, and disjoint from, the Cardano protocol
`fee` paid to stake pool operators. PhoenixKey fee is a value-flow on top
of the underlying ledger; ledger `fee` is unchanged.

---

### §36.1 Fee Schedule — On-Chain Adjustable Parameters [N]

`phoenix_fee : Operation → Lovelace` is a total function whose values are
**on-chain adjustable parameters**, NOT compile-time constants. They are
held in a `FeeParams` record at a single governance-controlled reference
UTxO (`fee_params_ref`) and read by the fee-receipt minting policy (§36.5)
via a CIP-0031 reference input. Adjusting any fee therefore updates the
`FeeParams` datum **without** rebaking the validator/policy script and
**without** changing any script hash or address.

```
-- FeeParams: on-chain record at fee_params_ref (governance UTxO).
FeeParams ≜ {
  fee_create_person_did     : Lovelace,   -- launch incentive: 0 (see below)
  fee_create_non_person_did : Lovelace,
  fee_rotate                : Lovelace,
  fee_transfer_service      : Lovelace,
  fee_update_guardians      : Lovelace,
  fee_init_recovery_t1      : Lovelace,
  fee_init_recovery_t2      : Lovelace,
  fee_init_recovery_t3      : Lovelace,
  fee_init_recovery_t5      : Lovelace,
  fee_cancel_recovery       : Lovelace,
  fee_finalize_recovery     : Lovelace,
  fee_mint_magic            : Lovelace,
  fee_issue_vc_anchor       : Lovelace,
  cardano_bps               : ℕ,          -- split to Cardano Treasury (default 3000 = 30%)
  bps_denom                 : ℕ,          -- = 10_000
  policy_version            : ℕ,          -- monotone; increments each update
  update_authority          : FeeUpdateAuthority,
}

-- Two adjustment paths (governance selects which is active):
FeeUpdateAuthority ≜
  | DAO(gov_org_did: DID, threshold_m: ℕ, n: ℕ)
      -- PhoenixKey OrgDID m-of-n governance vote sets new values (§13, §35.2).
  | Algorithmic(formula_script: ScriptHash, oracle_refs: List<OutputReference>)
      -- A governance-installed Plutus module derives fees from on-chain
      -- signals (e.g. ADA/USD oracle, recent DID-creation rate, treasury
      -- target). The formula itself is a governance-installed parameter;
      -- the specific economic model is out of scope here (operational /
      -- DAO decision), defaulting to DAO until an Algorithmic module is voted in.
```

**Genesis defaults (lovelace; revisable per above):**

```
fee_create_person_did     = 0           -- LAUNCH INCENTIVE: PersonDID free
fee_create_non_person_did = 5_000_000   -- 5 ADA  (Org/Service/Device/...)
fee_rotate                = 1_000_000   -- 1 ADA
fee_transfer_service      = 5_000_000   -- 5 ADA  (ServiceDID Transfer)
fee_update_guardians      = 500_000     -- 0.5 ADA
fee_init_recovery_t1/t2/t3= 1_000_000   -- 1 ADA each
fee_init_recovery_t5      = 3_000_000   -- 3 ADA  (Tier 5 — deterrent)
fee_cancel_recovery       = 500_000     -- 0.5 ADA
fee_finalize_recovery     = 500_000     -- 0.5 ADA
fee_mint_magic            = 500_000     -- 0.5 ADA  (per mint tx)
fee_issue_vc_anchor       = 1_000_000   -- 1 ADA  (per anchor tx)
cardano_bps               = 3000        -- 30%; phoenix share = bps_denom − cardano_bps
bps_denom                 = 10_000
```

| Operation | Redeemer (`types.ak`) | `phoenix_fee` field |
|---|---|---|
| **Create PersonDID** | (publish-did metadata tx) | `fee_create_person_did` — **0 at launch** |
| Create non-Person DID (Org / Service / Device / Machine / Asset / Bot / Agent / Context / Char) | (publish-did metadata tx) | `fee_create_non_person_did` |
| TAAD Rotate | `Rotate` | `fee_rotate` |
| TAAD Transfer (ServiceDID only) | `Transfer` | `fee_transfer_service` |
| Update Guardians | `UpdateGuardians` | `fee_update_guardians` |
| Init Recovery — Tier 1 / 2 / 3 | `InitRecovery` (tier proof) | `fee_init_recovery_t{1,2,3}` |
| Init Recovery — Tier 5 | `InitRecovery` (Tier 5 proof) | `fee_init_recovery_t5` |
| Cancel / Finalize Recovery | `CancelRecovery` / `FinalizeRecovery` | `fee_cancel_recovery` / `fee_finalize_recovery` |
| MAGIC mint | (`lamp_policy` / `magic_policy` mint redeemer) | `fee_mint_magic` |
| Issue VC anchor | (off-chain VC + on-chain anchor metadata tx) | `fee_issue_vc_anchor` |

*Note (PersonDID = 0 at launch):* PersonDID creation fee is **0** as a
user-acquisition incentive. PersonDID is the root of trust and the entry
point for human users; a zero fee removes onboarding friction. Governance
(DAO or Algorithmic) may raise it later via a `FeeParams` update — no
script redeploy needed. While a fee is `0`, the operation is handled
exactly like `Deactivate` (next note): no donation, no Phoenix output, no
fee-receipt mint.

*Note (zero-fee ops):* whenever `phoenix_fee = 0` (PersonDID at launch, or
the `Deactivate` redeemer which is permanently fee-exempt for GDPR-erasure
parity, §34.1), the fee mechanism is **skipped entirely** — no
`treasury_donation` (Conway field 20 requires `positive_coin > 0`), no
Phoenix Treasury output, no fee-receipt mint. The split invariants (§36.5)
are vacuously satisfied.

*Note (entity_type discrimination):* the Create distinction is **off-chain**
(the tx builder selects the field from `EntityType`) — there is no on-chain
`Create` redeemer. The fee-receipt minting policy (§36.5) reads the active
`FeeParams` from `fee_params_ref` and enforces the split on a non-zero fee;
the absolute per-operation amount is asserted off-chain (SDK) and observed
by the resolver (I-FEE-OBS-1).

*Note (governance update integrity):* every `FeeParams` update MUST
increment `policy_version` and be authorised per `update_authority`
(DAO m-of-n signatures, or a valid Algorithmic-module spend). Clients read
the current `FeeParams` from `fee_params_ref` at build time; an update is a
single spend of that UTxO producing a new `FeeParams` datum.

---

### §36.2 Split Formula (Integer Arithmetic) [N]

For any `phoenix_fee ∈ Lovelace`:

```
cardano_share : Lovelace ≜ ⌊ phoenix_fee × FeeParams.cardano_bps
                              / FeeParams.bps_denom ⌋
phoenix_share : Lovelace ≜ phoenix_fee − cardano_share
-- cardano_bps, bps_denom read from the active FeeParams (§36.1).
-- When phoenix_fee = 0 ⇒ cardano_share = phoenix_share = 0 (fee-exempt op).
```

All arithmetic is integer (Z); no floats, no rationals. The floor in
`cardano_share` resolves rounding **in favour of Phoenix Treasury** —
this is the deliberate convention so the Cardano-side donation can never
exceed the user-promised 30 %, and the leftover lovelace (≤ 1) lands on
the Phoenix side where it remains governable.

Numeric example (`phoenix_fee = 2_000_000`):

```
cardano_share = ⌊ 2_000_000 × 3000 / 10_000 ⌋ = 600_000
phoenix_share = 2_000_000 − 600_000           = 1_400_000
                                                ─────────
                                       sum    = 2_000_000   ✓
```

For all 13 entries of §36.1 the residual `phoenix_fee mod 10` is 0, so
the split is exact and `phoenix_share / cardano_share = 70/30` with zero
truncation loss. The floor convention exists for future fee revisions
that may not divide evenly by 10_000.

---

### §36.3 Cardano Treasury Donation [N]

The `cardano_share` MUST be paid via the Conway-era native
**treasury-donation** field of the transaction body, **not** as a UTxO
output to any address. This is the only ledger-recognised path that
deposits lovelace directly into the Cardano Treasury (CIP-1694 §
Treasury; conway.cddl key `20`).

Conway `transaction_body` (excerpt, normative):

```
transaction_body =
  { 0 : set<transaction_input>            ; inputs
  , 1 : [* transaction_output]            ; outputs
  , 2 : coin                              ; fee
  , ...
  , ? 20 : positive_coin                  ; treasury_donation  (Conway)
  }
```

**Encoding requirement.** For any tx `T` that exercises a fee-bearing
PhoenixKey operation **with `phoenix_fee > 0`**:

```
treasury_donation(T) = cardano_share                     -- field 20
∃ phoenix_output ∈ T.outputs:
    address(phoenix_output) = phoenix_treasury_address    -- §36.4
    lovelace(phoenix_output.value) ≥ phoenix_share
```

`positive_coin` requires the donation be strictly positive. For any
`phoenix_fee ≥ FeeParams.bps_denom` (= 10 000 lovelace = 0.01 ADA) the
§36.2 formula yields `cardano_share ≥ 1`; e.g. the smallest non-zero
default `fee_update_guardians = 500_000` maps to `cardano_share = 150_000`,
well above the floor.

**Zero-fee operations** (PersonDID at launch with `fee_create_person_did
= 0`, and the permanently fee-exempt `Deactivate`): `phoenix_fee = 0` ⇒
`cardano_share = 0`. Field 20 (`positive_coin`) **MUST be omitted**
(0 is not a valid `positive_coin`), no Phoenix Treasury output is
required, and no fee-receipt mint is performed. The tx carries only the
Cardano protocol fee. Should governance later raise
`fee_create_person_did` above 0 via a `FeeParams` update, PersonDID
creation automatically becomes fee-bearing under this same rule — no
script change.

**Why a workaround would be wrong.** Pre-Conway implementations
sometimes pay "Cardano dev fund" via a UTxO output to a multisig
controlled by a foundation wallet. That is **not** a donation to the
Cardano Treasury; the lovelace remains in the UTxO set and is
indistinguishable from any other transfer. Field 20 is the only encoding
that increases `Treasury` in the ledger state. PhoenixKey MUST use field
20 on Conway-era networks (mainnet, preprod, preview from epoch ≥ the
Conway hard fork).

---

### §36.4 Phoenix Treasury Address [N]

`phoenix_treasury_address` is the script address of the **Phoenix
Treasury validator** (Plutus V3, parameterised by PhoenixKey OrgDID).
Derivation:

```
phoenix_treasury_address ≜
    addr_credential( ScriptHash(blake2b_224(
        phoenix_treasury_validator_cbor( params ))) )
  where params =
    { gov_org_did            : DID            -- PhoenixKey ServiceDID's
                                                governing OrgDID (§29.3)
    , withdrawal_threshold_m : Int            -- m of n multisig
    , withdrawal_n_signers   : Int            -- = |OrgDID members| at deploy
    , min_withdrawal_lovelace: Int            -- anti-dust
    , vote_anchor_required   : Bool           -- if True, Withdraw must carry
                                                a governance-anchor field
    }
```

**Spend conditions (validator design — full pseudocode in companion
document `phoenix-treasury-validator-design.md`):**

| Redeemer | Validator check |
|---|---|
| `Lock` | always accepts (anyone may pay into Phoenix Treasury). |
| `Withdraw { amount, recipient, sig_set, vote_proof }` | (a) `count_signed_org_members(self, gov_org_did) ≥ withdrawal_threshold_m`; (b) `vote_proof` resolves to an executed PhoenixKey OrgDID governance vote permitting `amount` to `recipient`; (c) `amount ≥ min_withdrawal_lovelace`; (d) continuing-output value reduced by exactly `amount`; (e) `last_withdrawal_slot` strictly increases. |

**Lock semantics.** A `Lock` spend keeps Phoenix Treasury composable
with arbitrary fee-bearing tx builders: the SDK may consume an existing
Phoenix Treasury UTxO and re-deposit its full value alongside the new
fee in the same tx (e.g. to merge dust outputs), without needing any
governance signature. The continuing-output invariant under `Lock` is
`output.lovelace ≥ input.lovelace` (cannot leak), enforced on-script.

`gov_org_did = PhoenixKey OrgDID` is the same OrgDID that governs the
PhoenixKey ServiceDID per §29.3 Genesis Protocol. Member set and
threshold are read off the OrgDID's on-chain datum (resolved by the
validator via reference input — the OrgDID UTxO is included in the tx
as a read-only reference, no spend).

---

### §36.5 Invariants [N]

```
-- I-FEE-1: Cardano share floor.
-- For every PhoenixKey-fee-bearing tx T with declared phoenix_fee F > 0:
--   treasury_donation(T) ≥ ⌊ F × FeeParams.cardano_bps
--                              / FeeParams.bps_denom ⌋
-- (F = 0 ⇒ fee-exempt op: field 20 omitted, no Phoenix output — §36.3.)

-- I-FEE-2: Conservation (no leakage).
-- For every PhoenixKey-fee-bearing tx T with declared phoenix_fee F:
--   treasury_donation(T)                                -- field 20
-- + Σ { lovelace(o.value) | o ∈ T.outputs,
--                           o.address = phoenix_treasury_address }
--   ≥ F
-- (Equality holds modulo Phoenix-Treasury anti-dust top-ups; a strict-
-- equality variant I-FEE-2-STRICT is enforced by the off-chain SDK
-- before signing, see §36.6.)

-- I-FEE-3: Phoenix Treasury withdrawal authorisation.
-- For every spend of a phoenix_treasury_address UTxO U with redeemer
-- Withdraw{ amount, recipient, sig_set, vote_proof }:
--   count_signed_org_members(T, params.gov_org_did)
--     ≥ params.withdrawal_threshold_m
-- ∧ vote_proof binds amount and recipient to an executed PhoenixKey
--   OrgDID governance vote
-- ∧ no continuing-output value loss other than the authorised amount.

-- I-FEE-OBS-1: Resolver observability.
-- The did:phoenix resolver MUST surface, for every state-changing tx in
-- a DID's history, the phoenix_fee paid (parsed from field 20 +
-- phoenix_treasury_address output) and reject from "valid PhoenixKey
-- op" status any tx that does not satisfy I-FEE-1 ∧ I-FEE-2.
```

`count_signed_org_members(T, org_did)` is the existing OrgDID helper
from §13: it walks the OrgDID datum from a reference input in `T`,
loads the member key-hash set, and counts how many are present in
`T.required_signers` (the Cardano transaction's `extra_signatories`).

**Note on enforcement layer.** I-FEE-1 and I-FEE-2 are enforced
**on-chain** by a dedicated **fee-receipt minting policy** (Plutus V3),
not by the TAAD validator (which retains single-responsibility for
identity authority only). Every fee-bearing PhoenixKey operation MUST
mint exactly one `fee-ok` token under this policy, with an atomic
burn-on-mint so no token state accumulates. The minting policy validates,
in the same transaction:

```
-- fee-receipt minting policy spend check:
expect Some(donated) = transaction.treasury_donation          -- field 20
expect donated == cardano_share(declared_phoenix_fee)         -- I-FEE-1 equality
expect Σ { lovelace(o.value) | o ∈ transaction.outputs,
           o.address = phoenix_treasury_address } ≥ phoenix_share(declared_phoenix_fee)
-- I-FEE-2 conservation: donated + phoenix_output ≥ declared_phoenix_fee
```

`transaction.treasury_donation : Option<Lovelace>` is a typed accessor
in the Aiken stdlib `cardano/transaction` module (stdlib v2, compiler
v1.1.7), so no raw script-context Data parsing is required. The policy
optionally references the TAAD validator script hash to ensure it only
activates for genuine PhoenixKey operations (not arbitrary external txs).

I-FEE-3 is enforced on-chain by the Phoenix Treasury validator (§36.4).
I-FEE-OBS-1 (resolver observability) remains an off-chain indexing
requirement layered on top of the on-chain guarantees above.

The SDK enforces I-FEE-1/I-FEE-2 at tx-building time as a transitional
safeguard until the on-chain fee-receipt minting policy lands; the
resolver marks any tx violating the split as `INVALID_FEE_SPLIT`. The
economic incentive to cheat is already negative (an adversary
under-donating still loses the full `phoenix_fee` to the Phoenix-side
output and gains nothing), so the on-chain policy hardens an
already-safe position rather than closing an exploitable gap.

---

### §36.6 Theorem 36.1 — Fee Accountability [N]

The theorem below states a property of the *specified* design that a
conformant implementation MUST satisfy.

**Statement.** Under a conformant implementation of the §36 fee mechanism,
every fee-bearing PhoenixKey operation contributes simultaneously to
(a) Cardano network sustainability via the Conway treasury and
(b) PhoenixKey ecosystem sustainability via the Phoenix Treasury, with
on-chain proof of both contributions and zero leakage.

*Formally.* For every PhoenixKey state-changing transaction `T` with
declared `phoenix_fee = F > 0`, a conformant implementation builds `T`
such that:

```
(1) treasury_donation(T) ≥ ⌊ F × 3000 / 10_000 ⌋             [I-FEE-1]
(2) Σ phoenix_treasury_outputs(T) = F − treasury_donation(T)  [I-FEE-2-STRICT]
(3) The Cardano ledger atomically:
      − reduces inputs by F (+ protocol fee + min-ADA outputs),
      − increases Treasury by treasury_donation(T),
      − adds outputs at phoenix_treasury_address summing to phoenix_share.
```

*Proof.*
- (1) and (2) are conditions a conformant implementation MUST enforce —
  at minimum off-chain (the reference SDK rejects the tx before signing,
  `build_*_tx` returns `Err`) and, once the fee-receipt minting policy
  lands, on-chain (the policy fails the mint if either is violated).
  See the §36.5 enforcement note.
- (3) follows from Cardano ledger atomicity (Conway era):
  `transaction_body` field 20 is processed in the same ledger
  state-transition that processes inputs and outputs; either the entire
  tx succeeds and Treasury increases by exactly `treasury_donation(T)`,
  or the tx fails and no state change occurs (CDDL `positive_coin`
  rejects 0; ledger validation rejects underfunded inputs).
- The Phoenix-side accountability follows from the Phoenix Treasury
  validator (`Lock` accepts the output; subsequent `Withdraw` requires
  I-FEE-3 — multisig + governance vote).

Therefore every PhoenixKey op that successfully lands on-chain has, by
construction, contributed to both sustainability pools, and any tx that
fails to so contribute is rejected (off-chain by SDK; on-chain by
ledger if field 20 is malformed; downstream by resolver if any §36.5
invariant is violated). ∎

**Corollary 36.1.1 — No Free Lunch.** A PhoenixKey operation cannot
land on-chain without paying both shares. In particular, an
implementation cannot "save gas" by setting `treasury_donation = 0`:
the SDK refuses to sign and the resolver refuses to index. ∎

**Corollary 36.1.2 — Boundary Compatibility with §31.1.** Fee logic
falls entirely within §31.1 Fully Open scope: §36.1 constants, §36.2
formula, §36.3 encoding, §36.4 validator design, and §36.5 invariants
are all specified in this document and licensed Apache 2.0. Any
third-party PhoenixKey implementation MUST satisfy §36 to be
interoperable with the canonical resolver. ∎

---

*End §36.*

---

## Appendix A — MECE Coverage Matrix [N]

**Entity Type × Agency Level (10 non-empty cells from MECE analysis):**

| | Autonomous | Delegated | Passive |
|---|---|---|---|
| **Biological** | PersonDID | — | — |
| **Collective** | OrgDID | — | ContextDID |
| **Physical compute** | — | MachineDID | DeviceDID |
| **Physical object** | — | — | AssetDID |
| **Digital** | AgentDID | BotDID, ServiceDID | — |
| **Virtual** | — | — | CharDID |

**Lifecycle Patterns:**

| Pattern | Types |
|---------|-------|
| Indefinite, non-transferable | PersonDID, OrgDID, BotDID, AgentDID, ServiceDID |
| Indefinite, transferable | MachineDID, AssetDID, CharDID |
| Time-bounded | ContextDID |
| Cert-rotatable | DeviceDID |

---

## Appendix B — DID Construction [N]

```
DID_construct(type, creator, slot) → DID:
  rand_256 ← SecureRandom(256)
  raw = H(encode(type) ++ (creator ?? "root") ++ encode(slot) ++ rand_256)
  RETURN "did:phoenix:" ++ base32(slot) ++ ":" ++ hex(raw)

Collision: P(n DIDs) ≤ n²/2^257;  at n=10^12: P ≤ 5.4×10^{-54}
```

---

## Appendix C — Capability Taxonomy [N]

```
-- Tier 1 — Transaction: Sign_Tx(v), Mint_Token(p), Burn_MAGIC(n)
-- Tier 2 — Identity:    Read_DID(t), Write_DID(d), Issue_VC(s), Verify_Attestation
-- Tier 3 — Asset:       Attest_Asset(c), Transfer_Asset(a), Transfer_Machine(m), Transfer_Char(c)
-- Tier 4 — Control:     Operate_Device(d), Operate_Machine(m), Access_Service(s), Cancel_Suspension
-- Tier 5 — Deploy:      Deploy_Agent(cap), Deploy_Bot(cap)
-- Special:              Full_Authority  (PersonDID only; subsumes all)
```

---

## Appendix D — Breaking Changes Classification [N]

| Category | Fixes | Deployment Requirement |
|----------|-------|----------------------|
| **Schema** (datum format) | F-33, F-06-R, F-10, F-16, F-20, F-47, E-01 | Validator redeployment |
| **Validator logic** (consensus-breaking) | F-01, F-02-R, F-03, F-04, F-37, F-47 | Coordinated hard fork |
| **Additive invariants** | I-SVC-CHAIN, I-MACH-AGENT, I-MACH-AGENT-2 | New contracts only |
| **Spec clarifications** | F-05,F-07–F-09,F-11,F-13–F-32,F-41,F-42,E-02–E-04 | Doc update only |
| **v4.3 Schema** (datum additions) | TAADDatum: jurisdiction_flags, guardian_did, estate_config | Validator redeployment |
| **v4.3 PersonDID** (field additions) | emergency_vault_key, birth_slot, jurisdiction | Validator redeployment |
| **v4.3 New ops** (End-of-Life) | Op_declare_death, Op_finalize_posthumous, Op_declare_incapacity, Op_restore_capacity, Op_initiate_dissolution, Op_complete_dissolution, Op_destroy_asset, Op_decommission_device, Op_remove_guardian_did | New contracts only |
| **v4.3 New states** | Posthumous, Incapacitated, Dissolving, Archived, Decommissioned | Validator redeployment |
| **v4.3 Additive** | §33 Emergency Vault (client-side), §34 Compliance (policy), §35 Architecture boundary | Doc/client only |
| **v4.5 Bug fixes** | salt_pw₁ circular dependency removed (salts on-chain); `secondary_wallet_enrolled:𝔹` → `secondary_wallets:List<SecondaryWallet>`; email possession factor + EmailAccessProof; Cancel Option A removed | Validator redeployment |
| **v4.5 Design** | I-RECOVERY-4 revised: grace period (backup_deadline field, 30-day window, restricted ops); I-RECOVERY-5: recursive orphan prevention | New contracts only |
| **v4.5 Editorial** | Argon2id MiB unit fix; HIBP k-anonymity reference; I-TIER5-PW-5 removed | Doc update only |
| **v4.4 Validator logic** | Tier 5 path in select_recovery_tier; I-RECOVERY-4 enforcement at registration; TAAD_cancel_tier5 | Coordinated hard fork |
| **v4.4 Spec clarifications** | §11.0 Recovery scope; I-RECOVERY-1..4; I-TIER5-PW-1..5; I-TIER5-CANCEL-1..3 | Doc update only |
| **v4.4 Additive** | §11.6 Tier 5 (off-chain commitment, validator BLAKE2b only); §11.7 Tier 5 cancel | New contracts only |

*Testnet: All Schema + Validator Logic changes deployed atomically. No live data → migration cost = 0.*

---

## Appendix E — Glossary [I]

| Term | Definition |
|------|------------|
| Master_KEK | 256-bit root secret; source of Cardano wallet and TAAD key derivation |
| Cloud_Secret | 256-bit random; stored in hardware Keychain; per-device |
| Device_KEK | HKDF(Cloud_Secret ∥ HW_UID); encrypts Master_KEK for LampNet |
| TAAD_Key | Ed25519 key derived from Master_KEK; on-chain controller; survives recovery |
| HW_Key | Secure Enclave key; biometric-gated; never exported; per-device |
| EncSeed | Enc(Device_KEK, Master_KEK); artifact stored on LampNet |
| LampNet | Decentralized storage network using LT fountain codes |
| TAAD | Token-Anchored Authority Delegation; on-chain state machine for PersonDID |
| Attr* | Recursive function mapping DID → economic principal (Person/Org/Service) |
| Lazy Revocation | O(1) revocation where cascade enforced at Authority check time |
| Token Bucket | Q-format rate limiting; continuous refill; no burst boundary |
| PersonDID_Privileged_Ops | TAAD recovery + Cancel_Suspension; bypass Active() |
| SuspendedByInfo | Union: Suspender(DID) \| GuardianCoalition({0,1}^256) |
| suspension_eligible | Guardian opt-in for suspension authority (default False) |
| I-RESOLVER-1 | Normative: did:phoenix resolver MUST traverse ancestry for deactivated |
| MECE | Mutually Exclusive, Collectively Exhaustive; taxonomy derivation method |
| EmergencyVaultKey | HKDF(Master_KEK, "emergency-vault-v1", salt); isolated from TAAD_Key; see §33.1 |
| EmergencyVaultData | Encrypted medical/emergency data; accessible offline without blockchain; see §33.2 |
| TombstoneRecord | On-chain record after GDPR erasure; contains H(content) only; no personal data; see §34.1 |
| CreatorAccess | READ-ONLY access retained by record creator during liability period; see §34.2 |
| MinorControl | Authority predicate for PersonDID with guardian_did ≠ None; see §34.3 |
| JurisdictionCode | ISO 3166-1 alpha-2 jurisdiction identifier; e.g., "VN", "EU", "US" |
| JurisdictionPolicy | Per-jurisdiction configuration: GDPR applicability, minor age, liability periods; see §34.4 |
| EstateConfig | Posthumous configuration: beneficiary, estate guardian, scope; see §32.1 |
| IncapacityConfig | Conservatorship configuration: conservator, scope, valid_until; see §32.2 |
| DissolutionConfig | OrgDID wind-down configuration: asset distribution, archive DID; see §32.3 |
| DestructionRecord | AssetDID end-of-life: evidence, destruction type, insurance link; see §32.4 |
| DecommissionRecord | DeviceDID end-of-life: wipe certificate, successor device; see §32.5 |
| Posthumous | DIDState for deceased PersonDID; beneficiary has limited scope; see §32.1 |
| Incapacitated | DIDState for PersonDID under conservatorship; time-bounded; see §32.2 |
| Archived | Final state of dissolved OrgDID; records remain verifiable; see §32.3 |
| Layer 0 | PhoenixKey identity layer; foundation for all MagicLamp apps; see §35 |
| co-consent | Decision mode for minors aged CO_CONSENT_ONSET to MINOR_THRESHOLD; see §34.3 |
| GDPRException | Art.17(3) exception to erasure right: LegalObligation, LegalDefense, etc.; see §34.1 |
| liability_period_active | creator_liability_active(record, s): s < created_slot + LIABILITY_PERIOD_SLOTS[domain] |
| KnowledgeFactors | Tier 5 recovery data: password_commitment + email_commitment + salts; see §11.6.2 |
| Tier 5 | Knowledge-based recovery fallback; weakest tier; 21-day timelock; see §11.6 |
| TIER5_TIMELOCK | 21 × 432,000 slots ≈ 21 days; mandatory for Tier 5 |
| Argon2id_out | Off-chain output of Argon2id(password, salt); NEVER stored on-chain; see I-TIER5-PW-1 |
| password_commitment | BLAKE2b-256(Argon2id_out ∥ salt₂); on-chain; cannot reverse to password |
| Orphaned DID | Non-Person DID whose owner is Posthumous/Incapacitated/Dissolved; see §11.0.3 |
| NonPerson_Recover | Owner-sig rotation for non-Person DID; no TAAD; no timelock; see §11.0.2 |
| I-RECOVERY-4 | Mandatory: at least one of {secondary_wallet, guardians≥2, knowledge_factors} at creation |
| SecondaryWallet | Commitment binding a recovery wallet to a PersonDID: {wallet_pkh_commitment, enrolled_slot}; see §10.1 |
| wallet_pkh_commitment | BLAKE2b-256(pkh ∥ did ∥ enrolled_slot); on-chain commitment to secondary wallet pubkey hash |
| EmailAccessProof | Oracle-signed proof of email mailbox access for Tier 5 recovery; see §11.6.2 |
| EMAIL_ORACLE_PK | Public key of PhoenixKey email possession oracle; governance-controlled; see §1.4 |
| backup_deadline | TAADDatum field: None if backup configured; Some(slot) = grace period end; see I-RECOVERY-4 |
| TIER5_TIMELOCK | 21 × 432,000 slots ≈ 21 days; mandatory Tier 5 observation window |
| TIER5_BACKUP_GRACE_SLOTS | 30 × 432,000 slots ≈ 30 days; grace period to configure backup after registration |
| HIGH_STAKES_THRESHOLD | 100 ADA (governance parameter); ops above this threshold restricted during backup grace period |
| NonPerson_Recover | Owner-signed rotation for non-Person DID; no TAAD; no timelock; see §11.0.2 |
| Orphaned DID | Non-Person DID whose owner is Posthumous/Incapacitated/Dissolved; see §11.0.3 |
| TIER5_BACKUP_GRACE_SLOTS | 30 days grace period after DID creation to setup backup; see I-RECOVERY-4 |
| I-RECOVERY-5 | Recursive orphan prevention: beneficiary Posthumous/Incapacitated → DID → Archived |
| I-TIER5-EMAIL-1 | Email must be possession factor (OTP), not knowledge factor (string); see §11.6.5 |
| I-TIER5-EMAIL-2 | EMAIL_ORACLE_PK is known constant; rotation via governance |

---

## Appendix F — Test Vectors [N]

### TV-1: Runtime Scope Check (F-01)

```
Chain: P→O→S→A, op=Mint_Token(X)
Slot 200: scope(O) reduced to {Sign_Tx} (Mint_Token revoked)
Authority(A, Mint_Token, 201):
  op ∈ scope(A)?  → True (A's datum unchanged)
  op ∈ scope(S)?  → True (S unchanged)
  op ∈ scope(O)?  → False (O updated) → Authority = False ✓
```

### TV-2: Guardian Suspension + Recovery Bypass (F-47)

```
Slot 100: Guardians suspend PersonDID P → Active(P) = False
Slot 101: Attacker initiates Tier 2 fraudulent recovery
Slot 102: Owner executes TAAD_Cancel_Recovery:
  op ∈ PersonDID_Privileged_Ops → bypass Active()
  TAAD_state_machine_valid: Verify(hw_pub, challenge, sig) → True
  Recovery cancelled ✓  (guardian suspension cannot block this)
```

### TV-3: Token Bucket Rate Limit (F-06-R)

```
max_rate=10, BOT_REFILL_WINDOW=1000, Q=10^9
REFILL_RATE_Q = 10×Q//1000 = 10_000_000

Initial: token_bucket_Q = 10×Q = 10_000_000_000
Slot 0–9: 10 ops → token_bucket_Q = 0
Slot 100: elapsed=90, refilled=900_000_000 < Q → Rate_ok=False (must wait) ✓
Slot 200: elapsed=200, refilled=2_000_000_000 > Q → Rate_ok=True ✓
No boundary burst: refill is continuous, not epoch-reset ✓
```

### TV-4: Attr* Attribution Chain (F-07)

```
Chain: Person P → Org O → Service S → Agent A1 → SubAgent A2
MAGIC burn by A2:
  Attr*(A2) = Attr*(A1) = Attr*(S) = S  [type=Service → base case]
  AppTokenEconomics: V(ServiceDID_S, epoch) += burn_amount ✓
```

### TV-5: Op_unsuspend PersonDID (Errata E-02)

```
PersonDID P is suspended. Owner initiates Cancel_Suspension:
  Op_unsuspend(P, s):
    Requires: TAAD_state_machine_valid(P, Cancel_Suspension, s)
    = Verify(P.hw_pub, H("cancel-suspension" ++ s ++ seq), sig) → True
  Effect: P.suspended_by ← None; seq++ ✓

No call to Auth_satisfied(owner(P), ...) because owner(P) = None ✓
```

### TV-6: DelegationToken Scope After Parent Reduction (F-37)

```
Slot 100: O issues DelegationToken to B: granted_cap={Mint_Token}
Slot 200: scope(O) ← {Sign_Tx} (Mint_Token removed)
Slot 201: DelegationToken_valid(dt, 201):
  dt.granted_cap ⊑ scope(O, 201)?         -- v4.7: ⊆ → ⊑ (§3.3, Errata CID-6)
  = {Mint_Token(p)} ⊑ {Sign_Tx(v)}?
  = Mint_Token(p) ⊑ Sign_Tx(v)? → False (different capability constructors;
    §3.3 only defines refinement within the same constructor, or via the
    universal c ⊑ Full_Authority clause — neither applies here) → token invalid ✓
  F-37 blocks stale delegation even without explicit revocation ✓
  (Same result as pre-v4.7 ⊆ check — ⊑ only ADDS the Full_Authority case,
  it never turns a previously-invalid check into valid for unrelated caps.)
```

---


---

# APPENDIX G — References [I]

All links valid as of 2026-05-28. Click to open.

## G.1 Standards and Specifications

| Ref | Document | URL |
|-----|----------|-----|
| [DID-CORE] | W3C DID Core v1.0 | https://www.w3.org/TR/did-core/ |
| [VC-DATA] | W3C Verifiable Credentials Data Model v2.0 | https://www.w3.org/TR/vc-data-model-2.0/ |
| [BBS-CRYPT] | W3C BBS Cryptosuite (selective disclosure) | https://www.w3.org/TR/vc-di-bbs/ |
| [STATUS-LIST] | W3C Status List 2021 | https://www.w3.org/TR/vc-status-list/ |
| [RFC5869] | HKDF — RFC 5869 | https://www.rfc-editor.org/rfc/rfc5869 |
| [RFC8032] | Ed25519 — RFC 8032 | https://www.rfc-editor.org/rfc/rfc8032 |
| [RFC9106] | Argon2 — RFC 9106 | https://www.rfc-editor.org/rfc/rfc9106 |
| [NIST-63B] | NIST SP 800-63B (Authentication) | https://pages.nist.gov/800-63-3/sp800-63b.html |
| [GDPR-17] | GDPR Article 17 — Right to Erasure | https://gdpr-info.eu/art-17-gdpr/ |
| [GDPR-REC26] | GDPR Recital 26 — Anonymous Data | https://gdpr-info.eu/recitals/no-26/ |
| [GDPR-ART8] | GDPR Article 8 — Children's Consent | https://gdpr-info.eu/art-8-gdpr/ |
| [CIP-0003] | Cardano Wallet Key Derivation | https://github.com/cardano-foundation/CIPs/blob/master/CIP-0003/ |
| [CIP-0049] | ECDSA/Schnorr secp256k1 in Plutus | https://github.com/cardano-foundation/CIPs/blob/master/CIP-0049/ |
| [CIP-0055] | Cardano Min-ADA Calculation | https://github.com/cardano-foundation/CIPs/blob/master/CIP-0055/ |
| [CIP-0068] | Cardano Datum Metadata Standard | https://github.com/cardano-foundation/CIPs/blob/master/CIP-0068/ |
| [CIP-0381] | BLS12-381 in Plutus | https://github.com/cardano-foundation/CIPs/blob/master/CIP-0381/ |
| [EUTXO] | Extended UTxO Model (IOHK) | https://iohk.io/en/research/library/papers/the-extended-utxo-model/ |
| [EIDAS] | eIDAS 2.0 EU Digital Identity | https://digital-strategy.ec.europa.eu/en/policies/eidas-regulation |

## G.2 Security and Cryptography

| Ref | Document | URL |
|-----|----------|-----|
| [BLAKE2] | BLAKE2 Specification | https://www.blake2.net/blake2.pdf |
| [SHAMIR79] | Shamir (1979) — How to Share a Secret | https://dl.acm.org/doi/10.1145/359168.359176 |
| [BELLARE96] | Bellare et al. — HMAC as PRF (1996) | https://dl.acm.org/doi/10.1145/234525.234532 |
| [NASH50] | Nash (1950) — Equilibrium in N-Person Games | https://www.pnas.org/doi/10.1073/pnas.36.1.48 |
| [NIST-PQC] | NIST Post-Quantum Cryptography Standard 2024 | https://csrc.nist.gov/projects/post-quantum-cryptography |
| [SAFECURVES] | SafeCurves — choosing safe elliptic curves | https://safecurves.cr.yp.to/ |
| [ZXCVBN] | zxcvbn — realistic password strength | https://github.com/dropbox/zxcvbn |
| [HIBP-API] | HaveIBeenPwned API v3 (k-anonymity) | https://haveibeenpwned.com/API/v3#PwnedPasswords |

## G.3 Platform Documentation

| Ref | Document | URL |
|-----|----------|-----|
| [APPLE-SE] | Apple Secure Enclave Guide | https://support.apple.com/guide/security/secure-enclave-sec59b0b31ff/web |
| [FIDO2] | FIDO2 / WebAuthn Specification | https://fidoalliance.org/fido2/ |
| [DIF-RESOLVER] | DIF Universal Resolver | https://github.com/decentralized-identity/universal-resolver |
| [OID4VC] | OpenID for Verifiable Credentials | https://openid.net/sg/openid4vc/ |

## G.4 Legal References

| Ref | Document | URL |
|-----|----------|-----|
| [VN-DECREE-96] | Vietnam Decree 96/2023/ND-CP (Medical Records) | https://vanban.chinhphu.vn/ |
| [VN-DECREE-13] | Vietnam Decree 13/2023/ND-CP (Personal Data) | https://vanban.chinhphu.vn/ |
| [WHO-RECORDS] | WHO Medical Records Manual 2006 | https://www.who.int/healthinfo/documentation/who_mrd_06.1_eng.pdf |
| [UNCRC] | UN Convention on Rights of the Child Art.5 | https://www.ohchr.org/en/instruments-mechanisms/instruments/convention-rights-child |
| [VN-CIVIL] | Vietnam Civil Code 2015, Art.21 | https://vanban.chinhphu.vn/ |

---

*PhoenixKey Formal Mathematical Specification v4.5*
*© 2026 Greensun Tech Corporation. Inventor: Nguyen Duc Thanh.*

*Peer Review: 5 rounds v4.2 — Gemon (MIT Mathematics & EECS), Serat (Principal Security Architect). v4.3–4.4: Sambo (Senior BA, MIT Sloan), Tuân (Validator Engineer). v4.4 security review pending.*

*v4.5 fixes: salt circular dependency, secondary_wallet boolean→list, email knowledge→possession, cancel Option A removed, grace period, recursive orphan, MiB, HIBP k-anon.*

*Normative: §1–§35, Appendices A–D, F.*
*Non-normative: §28, Appendix E.*

*Deferred to v5.0:*
*— Mechanized proofs (Coq/Isabelle) for Theorems 27.1, 27.5, 27.7, 28.1, 28.2*
*— ZK Groth16/Poseidon circuits for Tier 5 (replace commitment-reveal with ZK — Phase 2)*
*— Theorem 27.3: device-loss + guardian-collusion formal reduction*
*— Cross-chain bridge semantics*
*— IoT Sensor offline attestation batch model*
*— Jurisdiction policy governance protocol*
*— Tier 5 with ZK: password never appears in redeemer (Phase 2 migration path)*

*Successor documents:*
*— PhoenixKey DID Method Specification v1.0 (I-RESOLVER-1, resolver ancestry protocol)*
*— PhoenixKey Cardano Validator Reference v1.0 (Aiken/PlutusV3)*
*— PhoenixKey BA Spec v2.0 (business rules, user journeys)*
*— PhoenixKey Validator PR: §11.0 non-Person recovery + §11.6 Tier 5 + I-RECOVERY-4 enforcement*
