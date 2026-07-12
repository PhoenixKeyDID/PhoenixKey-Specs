# The `did:phoenix` Method Specification

## Abstract

`did:phoenix` is a DID method in which each decentralized identifier is a string identifier anchored one-to-one by a singleton non-fungible token (NFT) on the Cardano blockchain, using Plutus V3 smart contracts. Cardano's ledger model is UTxO-based (Unspent Transaction Output — each coin/asset exists as a discrete, uniquely-referenceable output that a transaction consumes in full and replaces with new outputs, as opposed to an account-balance model); this method represents each identity as exactly one such output, described next. The DID is not itself an NFT; the NFT is its on-chain anchor. Each identity — human (`Person`) or non-human (`Org`, `Device`, `Service`, and other entity types) — is represented on-chain by exactly one UTxO holding the anchor NFT and an inline datum (in Cardano/Plutus, a small piece of structured data attached to a UTxO) — called the "TAAD" (Token-Anchored Authority Delegation) in this method — that holds the current controller key, status, guardian set, and lineage. The method defines deterministic derivation of the DID string and its on-chain asset name, a 7-transition state machine for lifecycle operations (rotate, recover, deactivate, transfer, update guardians), and a W3C DID Resolution-conformant HTTP resolver.

This is version 1 of the `did:phoenix` method specification, published at `https://phoenixkey.me/did-method/v1`.

---

## 1. Status of This Document

This is version **v1** of the `did:phoenix` method specification, published at its canonical URL to back the registry entry merged into the W3C DID Method Registry:

- Registry file: `w3c/did-extensions` → `methods/phoenix.json`
- Registered fields: `"status": "registered"`, `"specification": "https://phoenixkey.me/did-method/v1"`, `"ledger": "Cardano Blockchain"`, contact: GreenSun Tech (`contact@greensun.tech`)
- Registry commit: `97c8197` (2026-07-04)

---

## 2. Introduction

`did:phoenix` identities are minted and managed through a single multi-purpose Plutus V3 validator — internally called **Design-2** (an internal engineering label used across the PhoenixKey codebase and specs to name this specific unified mint-and-spend architecture; it is not a public versioning of the DID method itself, and no separate "Design-1" is part of this specification) — where the NFT minting policy is defined to be identical to the validator's own script hash (a unified mint-and-spend design: the same script both mints the anchor NFT at genesis and guards its lifecycle thereafter). This removes the circular dependency of separately baking a policy ID into the spending validator, and lets genesis mint hard-bind the anchor NFT to the validator address in the same transaction that creates it. ("Design-2" is the term of record for this design across the PhoenixKey specification set — see the `Anchorme` module's Tech/Math specs.)

Ten entity types are supported. Each maps to a `*DID` name convention (`PersonDID`, `OrgDID`, `AgentDID`, …) used throughout the wider PhoenixKey documentation set (the Math specification and the per-module Feature specs); this document itself mostly uses the bare type name (`Person`, `Org`, …) to match the on-chain `entity_type` field, but the two forms denote the same type:

"Trust layer" (`L0`–`L4`) below indicates how an identity is authorized: `L0` is the sole root-of-trust type (self-genesis, no parent required); `L1`–`L4` are all descendant types, each created only by a co-signing parent identity one or more layers below `L0` (see §6.1, `GenesisChild`). The layer number is documentation-only — it is not an on-chain field — but it fixes the ownership hierarchy the `CanOwn` relation (§6.1) enforces.

| Byte | `entity_type` | DID name | Trust layer | Meaning |
|------|---------------|----------|-------------|---------|
| 0 | `Person` | `PersonDID` | L0 (root of trust) | Biological human; self-genesis, no parent |
| 1 | `Org` | `OrgDID` | L1 | Collective entity (company, DAO, government) |
| 2 | `Device` | `DeviceDID` | L2 | Computing hardware (phone, server, sensor) |
| 3 | `Machine` | `MachineDID` | L2 | Physical machine (vehicle, **robot**) |
| 4 | `Asset` | `AssetDID` | L2 | Passive physical asset (land, crop, good) |
| 5 | `Bot` | `BotDID` | L3 | Rule-based automation (ProofChat bot, keeper) |
| 6 | `Agent` | `AgentDID` | L3 | Autonomous AI (e.g. TigerAgent, MagicClaw) |
| 7 | `Service` | `ServiceDID` | L3 | Digital service (OriLife, AladinWork) |
| 8 | `Context` | `ContextDID` | L1 | Temporal bounded context (event, project) |
| 9 | `Avatar` | `AvatarDID` | L4 | Externally-operated entity (game character, pet, livestock) |

**`Bot` vs. `Agent` vs. robot.** `Bot` and `Agent` are both Layer-3 software automations but differ in autonomy: a `BotDID` identifies **rule-based automation** — a deterministic, script-driven process such as a ProofChat bot or a keeper/relayer that fires on fixed triggers. An `AgentDID` identifies **autonomous AI** — a system that plans, reasons, or acts toward open-ended goals (e.g. TigerAgent, MagicClaw). Neither is the same as a physical *robot*: a robot is a `MachineDID` (Layer 2, physical machine), because the `Bot`/`Agent` types classify software autonomy, not physical embodiment. An AI-driven robot is therefore two linked identities — a `MachineDID` for the physical unit and an `AgentDID` for its controlling software.

Non-`Person` entities are always created as children of an existing, active parent identity (an ownership edge enforced on-chain); `Person` identities are self-genesis (no parent).

The identity module that implements this method is internally named **Anchorme** (formerly "Identity") in the PhoenixKey specification set.

---

## 3. Method Name

The namestring that identifies this DID method is: `phoenix`.

A DID that uses this method MUST begin with the prefix:

```
did:phoenix:
```

---

## 4. Method-Specific Identifier

### 4.1 Syntax

The method-specific identifier consists of two colon-separated components: a 13-character base32 slot and a 64-character lowercase hex hash.

```
did:phoenix:<slot>:<hash>
```

- `<slot>` — 13 characters, alphabet `[a-z2-7]` (RFC 4648 base32, lowercase, no padding).
- `<hash>` — 64 lowercase hex characters (32 bytes), computed as `blake2b_256` over an encoding described in §4.3.

### 4.2 ABNF

```abnf
phoenix-did       = "did:phoenix:" phoenix-slot ":" phoenix-hash
phoenix-slot      = 13base32-lower
phoenix-hash      = 64HEXDIGLC
base32-lower      = %x61-7A / %x32-37   ; a-z / 2-7
HEXDIGLC          = DIGIT / %x61-66     ; 0-9 / a-f
```

### 4.3 Regular expression

```
^did:phoenix:[a-z2-7]{13}:[0-9a-f]{64}$
```

### 4.4 Derivation

The identifier is generated client-side inside a hardware-backed secure element (iOS Secure Enclave / Android StrongBox), never on-chain:

```
did:phoenix:<slot>:<hash>
  where hash = H( encode(entity_type) ‖ creator ‖ encode(slot) ‖ rand_256 )
```

(`‖` denotes byte-string concatenation — join the operands end-to-end into one byte sequence before hashing; it carries no other meaning here.)

`H` is the same primitive as the on-chain anchor name function (§4.5); `entity_type` is encoded as a fixed single byte per the canonical byte table in §2 (`Person=0, Org=1, Device=2, Machine=3, Asset=4, Bot=5, Agent=6, Service=7, Context=8, Avatar=9`). The byte **value** is the hash-contract invariant consumed on-chain; the type's display label (`Agent`, `Avatar`) is documentation-only and carries no on-chain meaning, so labelling is fixed by the value, not the name.

**Note (informative) — key material.** The identifier above is distinct from the key material that backs the verification methods in §5. The controller/verification keys and the associated Cardano wallet are produced off-ledger inside the secure element by a deterministic chain: a hardware-random 256-bit root secret is HKDF-separated ([RFC 5869](https://www.rfc-editor.org/rfc/rfc5869), domain label `wallet-seed-v1`) into a distinct HD seed, which then feeds the standard [BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) / [CIP-1852](https://cips.cardano.org/cip/CIP-1852) ([CIP-3](https://cips.cardano.org/cip/CIP-3) Icarus) derivation. Biometric authentication only unlocks the secure element that releases the root secret; it does not itself derive any key. The normative construction, with its per-step invariants, is specified in the PhoenixKey Formal Mathematical Specification §6.1.1 and §8.1.

### 4.5 On-chain anchor name — `N(did)`

Every `did:phoenix` identifier has a corresponding deterministic 32-byte **anchor name**, which is the exact asset name of its NFT on Cardano:

```
N(did) = blake2b_256( UTF-8(did) )
```

No salt, no double-hashing, no method other than a single BLAKE2b-256 digest over the UTF-8 bytes of the full DID string (including the `did:phoenix:` prefix). This value is used both as the Cardano native-asset name and as the lookup key (`did_hash`) for off-chain resolution by hash.

A published test vector: `blake2b_256("did:phoenix:person:alice")` starts with `1643c0c9…c470` (see the Anchorme module's Feature spec §1.3 for the full vector set).

---

## 5. DID Document

### 5.1 Overview

Resolving a `did:phoenix` identifier produces a W3C-conformant DID Document derived from the on-chain TAAD datum (see §6.2). A representative document:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/jws-2020/v1"
  ],
  "id": "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470",
  "verificationMethod": [
    {
      "id": "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470#device-key",
      "type": "JsonWebKey2020",
      "controller": "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470",
      "publicKeyJwk": { "kty": "EC", "crv": "P-256", "x": "...", "y": "..." }
    },
    {
      "id": "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470#controller-key",
      "type": "JsonWebKey2020",
      "controller": "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470",
      "publicKeyJwk": { "kty": "OKP", "crv": "Ed25519", "x": "..." }
    }
  ],
  "authentication": [
    "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470#device-key"
  ],
  "capabilityInvocation": [
    "did:phoenix:aaaaaaaaaaaaa:1643c0c9...c470#controller-key"
  ],
  "service": []
}
```

### 5.2 Verification method mapping

| DID Document relationship | On-chain source | Key type | Datum field |
|---|---|---|---|
| `authentication` | `HW_Key` (internal short name for the device-bound hardware key) | P-256 (secp256r1), Secure Enclave / StrongBox | `hw_key_pubkey` (SEC1 — Standards for Efficient Cryptography 1, the standard byte-encoding for elliptic-curve public keys; 33B compressed or 65B uncompressed) |
| `capabilityInvocation` | `TAAD_Key` (internal short name for the on-chain controller signing key) | Ed25519 | `controller_pkh` (a verification-key hash) |

`HW_Key` (P-256) is device-bound and used for local authentication/biometric gating. `TAAD_Key` (Ed25519) is the on-chain signing key whose hash (`controller_pkh`) the validator checks against transaction `extra_signatories` (the ledger's list of key hashes that must have signed the transaction) for every state transition. **The validator carries `hw_key_pubkey` by byte-equality only — it does not decode the curve or verify a P-256 signature on-chain** (see §7, Assumption T-2 / invariant I-CID-11).

**Fragment naming convention.** Each verification method is named by its **purpose**, not by an internal key name or an ordinal: `#device-key` for the device-bound P-256 authentication key (`HW_Key`), and `#controller-key` for the Ed25519 on-chain controller/authority key (`TAAD_Key`). Purpose-based fragments are chosen deliberately over both (a) internal code names such as `taad-key`, which would export PhoenixKey-internal vocabulary into a public specification, and (b) RFC 7638 JWK-thumbprint fragments, which change on every rotation because they bind to the key material itself. A purpose fragment instead stays **constant across `Rotate`/recovery**, so a relying party can hard-code `did#controller-key` as a durable reference that survives key rotation — the very property PhoenixKey's rotation-invariant identity exists to provide. The current model has exactly one key per purpose (one `hw_key_pubkey` and one `controller_pkh` per anchor). A future revision introducing multiple concurrent keys of the same purpose (e.g. multi-device custody) MUST define a **stable per-key suffix** — a device tag or JWK thumbprint, never a bare ordinal — at that time. Fragment strings carry no confidentiality (see §8).

### 5.3 Service endpoints

`service` entries, when present, are populated from off-chain registry data associated with the DID (not part of the on-chain TAAD datum in the current version). The resolver publishes an empty `service` array when none is registered.

---

## 6. CRUD Operations

All operations are Cardano transactions against a single multi-purpose Plutus V3 validator (`taad`), whose script hash simultaneously serves as the NFT minting policy ID ("Design-2"). Every identity is one UTxO at `Script(π)` (`π` = validator script hash) holding exactly one NFT `(π, N(did))`, minimum ADA, and an inline `TAADDatum`.

### 6.1 Create

Two mint paths, discriminated by the `StateNftRedeemer` (a *redeemer* is the Plutus term for the argument a transaction supplies to tell a validator script which action it is invoking):

**`GenesisPerson`** (self-genesis, root identities only):
- Preconditions: exactly one NFT minted under `π`; the sole output carrying that NFT sits at `Script(π)`; asset name equals `N(did)`; `entity_type = Person`; `parent_did = None`; the datum's `controller_pkh` is present in the transaction's signers (self-sign); the datum is fresh (`sequence = 0`, `status = Active`, `revoked_slot = None`).
- Signer: the identity's own controller key (Enclave-produced witness).
- This path does not bind the DID string to a global uniqueness check at mint time; see §7.2 (Security Considerations).

**`GenesisChild { owner_did }`** (all non-`Person` entity types):
- Preconditions: a reference input carries the owner's live anchor NFT with `status = Active`; the owner's `controller_pkh` is among the transaction signers (parent-sig); the owner/child entity-type pair satisfies the method's ownership relation ("CanOwn"); the child's `parent_did` equals `Some(owner_did)`; the child's `entity_type ≠ Person`; the child datum is fresh.
- Signer: the parent (owner) identity's current controller.
- This path is **not** subject to the uniqueness limitation described in §7 — the parent signature requirement closes that gap for every non-`Person` entity type.

### 6.2 Read (Resolve)

Resolution follows the W3C DID Resolution specification (draft/CR, referenced as v0.3 at the time of writing). `did:phoenix` is designed for **multi-modal resolution**, because it runs across environments with different trust and availability assumptions — plain web/HTTP clients, on-chain dApp validators, and the LampNet/LAMP smart-contract runtime. The authoritative state is always the on-chain `TAADDatum`; each mode below is a different way of reading that same state, trading convenience against trust:

1. **HTTP resolver (live, trusted intermediary):** `GET /identifiers/{did}` returns a DID Document per §5; convenience endpoints `GET /identity/{did}/pubkey|document|status`. This is the simplest path for ordinary web clients, but the resolver is a trusted party — it can omit, delay, or misreport state. A client with integrity requirements SHOULD cross-check the returned document against the chain (mode 2 or 3) rather than trust the HTTP response alone.
2. **On-chain resolution (live, trustless):** any transaction may read an identity's current authoritative state by placing its anchor UTxO as a **reference input** (an input a transaction reads without consuming) and reading the inline `TAADDatum` directly. This needs no HTTP and no trusted resolver — the ledger itself guarantees the datum is current. It is how `did`-scoped spending logic (e.g. `did_payment`, `did_stake` validators) authorizes transactions without consuming the anchor, and it is the canonical resolution path inside the **LampNet/LAMP** contract runtime, where an HTTP resolver is neither available nor trusted.
3. **Direct chain query (dApp-native):** a dApp or wallet already connected to Cardano — through a CIP-30 wallet connector (CIP-30 is the Cardano community standard browser-wallet API that lets a web page request signing/data access from an installed wallet extension), a chain-data provider (Blockfrost / Koios / Ogmios — three independent hosted/self-hostable services that let an application query Cardano chain state over a plain HTTP/WebSocket API instead of running a full node), or a light client — can fetch the anchor UTxO and decode the `TAADDatum` itself, reconstructing the DID Document client-side with no PhoenixKey-operated resolver in the trust path. This gives dApp integrations HTTP-level convenience at on-chain-level trust.
4. **Universal Resolver driver (roadmap):** a Decentralized Identity Foundation (DIF) Universal Resolver driver for `did:phoenix` is planned as the ecosystem-interop path, so third-party resolvers can resolve the method through the standard driver interface. Not yet implemented.

A resolve-by-hash endpoint (`GET /v1/registry/resolve/{did_hash}`, keyed on `N(did)`) and point-in-time historical resolution endpoints are specified but **not yet implemented** (backend team, tracked as future work).

### 6.3 Update

Update operations spend the existing anchor UTxO and recreate it at the same script address with an incremented `sequence` counter (strict `+1`, replay protection) and the `did`/`entity_type` fields held constant (identity persists across key changes). Five update redeemers:

| Redeemer | Effect | Signer(s) | Applicable entity types |
|---|---|---|---|
| `Rotate` | Replace `controller_pkh` + `hw_key_pubkey` | current controller | all |
| `UpdateGuardians` | Replace guardian set (max 5) | current controller | all |
| `Transfer` | Replace controller + HW key + guardian set (ownership handoff) | current controller **and** new controller (atomic 2-of-2) | `Service` only |
| `InitRecovery` | Enter `Recovering` state with a pending controller/HW key, timelock deadline, and locked collateral | ≥ guardian threshold | all with guardians configured |
| `CancelRecovery` | Return to `Active` before the recovery deadline | current controller | all (during `Recovering`) |
| `FinalizeRecovery` | Install the pending controller/HW key after the recovery deadline | pending controller | all (during `Recovering`) |

Recovery (`InitRecovery`/`CancelRecovery`/`FinalizeRecovery`) is a guardian-assisted key-recovery sub-flow bounded by a timelock and locked collateral; it is documented in full in the method's companion Rebirthme module specification and is only summarized here for completeness of the redeemer enumeration.

### 6.4 Deactivate

The `Deactivate` redeemer transitions `status` from `Active` to `Revoked` and sets `revoked_slot` to a finite lower bound derived from the transaction's validity interval. **`Revoked` is a dead end**: no update redeemer's guard matches `(_, Revoked)`, so a deactivated anchor can never again be rotated, transferred, recovered, or otherwise spent through the lifecycle paths — only a pure-burn path (`GenesisBurn`, all value movements negative) can remove the NFT from circulation entirely. Readers that gate on liveness (e.g. wallet spending authorization) treat any non-`Active` status as non-authoritative.

---

## 7. Security Considerations

### 7.1 Structural integrity (proven invariants)

The validator enforces, for every valid transaction: (a) singleton spend (the invariant that a transaction consumes and recreates exactly one instance of the anchor NFT — never zero, never more than one) — each transition consumes exactly one anchor NFT and recreates exactly one at the validator address; (b) strict sequence monotonicity — replay of a stale transaction is rejected because it can never match the current on-chain sequence; (c) NFT non-escape — the anchor NFT cannot leave the validator's script address through any lifecycle path other than an explicit burn; (d) identity persistence — the `did` string and `entity_type` are immutable across every update operation, so key rotation never changes which identity a DID resolves to; (e) dead-end deactivation — a revoked anchor has no path back to a spendable state.

### 7.2 On-chain anchor uniqueness

**Plain-language summary:** for `Person` identities only, nothing on-chain today stops someone else from minting a second, independent identity with the exact same DID string as yours, under their own key. Anything that relies on "this DID string always means this one controller" — most importantly, custody of value held under a `Person` DID — is at risk until the fix described below (a global uniqueness accumulator) ships. `Org`/`Service`/`Device`/every other non-`Person` type is unaffected, because minting those always requires a co-signature from an existing parent identity, which already rules out this attack.

The current implementation does not enforce **global on-chain anchor uniqueness** at mint time for the `GenesisPerson` path. Because a Cardano minting policy cannot inspect the full UTxO set at mint time, nothing on-chain today prevents a second, independently-minted anchor from being created with the same asset name `N(did)` (i.e., the same DID string) but a different controller key. A resolver, indexer, or wallet-side reference-input lookup that encounters two live anchors sharing the same `N(did)` is not currently instructed by the protocol which one is authoritative.

This limitation is scoped **narrowly to the `Person` (self-genesis) creation path**. It does **not** apply to `Org`, `Service`, `Device`, or any other entity type created via `GenesisChild`, because that path requires a live, active parent anchor to co-sign the transaction — an attacker cannot mint a colliding child anchor without also controlling (or being authorized by) the parent identity.

This is an **encoding-layer / on-chain-anchor limitation**, independent of and orthogonal to how the underlying key material for a `Person` DID is generated or held (e.g., secure-element-backed key custody on the holder's device is out of scope for this section and is not the mechanism being discussed here).

**Resolver guidance while this is unresolved:**
- Resolvers and indexers SHOULD treat the presence of more than one live anchor UTxO sharing the same `N(did)` as an exceptional condition, not silently pick one.
- Two candidate policies for implementers to consider until a protocol-level fix lands: (i) **fail-closed** — refuse to resolve / refuse to authorize spend when a collision is detected, surfacing an error to the caller; (ii) **first-anchor-wins** — treat the earliest-minted (by slot) live anchor for a given `N(did)` as authoritative and treat any later one as invalid, provided the resolver can reliably establish mint order from chain history.
- The method specifies a structural, protocol-level fix — a spend-based global uniqueness accumulator across the `Person` namespace — as the mechanism that closes this at mint/resolution time. Until a given deployment enables that mechanism, resolvers apply the interim policies above; production deployments that rely on `Person`-type custody should require it before treating `Person` DIDs as fully collision-resistant.

### 7.3 Key material carried, not verified, on-chain

The validator carries the P-256 `hw_key_pubkey` field only by byte-equality; it does not decode the curve, verify a signature against it, or enforce a length constraint on-chain. Authorization for every state transition is anchored entirely in the Ed25519 `controller_pkh` (`TAAD_Key`) checked against native Cardano transaction signatures. Implementers should not treat the presence of a `hw_key_pubkey` value in a datum as itself cryptographically authenticated by the ledger.

### 7.4 Replay and front-running

Strict `sequence` incrementing on every transition, combined with singleton-spend enforcement, is the primary replay defense described in §7.1. Standard Cardano transaction-ordering considerations (UTxO contention, mempool visibility) apply as they would to any smart-contract-mediated state machine.

---

## 8. Privacy Considerations

- **On-chain linkage.** The DID string, its derived anchor name `N(did)`, the controller key hash, and the guardian set are all publicly visible on-chain in the inline datum. `did:phoenix` provides no ledger-level confidentiality; privacy-sensitive deployments must apply off-chain measures (selective disclosure, off-chain service data, unlinkable addressing at the wallet layer) independently of DID resolution.
- **Lineage visibility.** The `parent_did` field makes ownership/creation relationships between identities (e.g., an organization and the service identities it creates) publicly traceable on-chain.
- **Key rotation does not unlink history.** Because `did` and `entity_type` persist across `Rotate`/`Transfer`/recovery, historical linkage to a given DID persists even after its signing keys change — rotation defends against key compromise, not against on-chain observability of past activity under that DID.
- **Resolver-side data minimization.** Implementers of resolver/indexer infrastructure should avoid retaining more off-chain metadata (e.g., IP addresses of resolution requests) than operationally necessary.
- **Verification-method fragments are public role labels, not secrets.** The entire DID Document — including every `verificationMethod.id` fragment — is returned in full on every resolution, so fragment names carry no confidentiality property and "guessing" a fragment is not a meaningful attack: a resolver client already sees the complete list, and the key type/count are already disclosed by the adjacent `publicKeyJwk`/`type` fields and the array length. Fragments (`#device-key`, `#controller-key`) are stable, human-readable purpose labels only (see §5.2) and MUST NOT be used to encode information not already public in the document.

---

## 9. Conformance

An implementation of `did:phoenix` claiming conformance to this method specification MUST:

1. Generate identifiers matching the ABNF/regular expression in §4.2/§4.3.
2. Compute the anchor name as `N(did) = blake2b_256(UTF-8(did))` with no salting or alternate hash function, and use this value verbatim as the Cardano native-asset name.
3. Implement resolution consistent with **W3C DID Resolution v0.3** semantics, returning a DID Document matching §5 for any live, `Active` anchor, and an appropriate not-found/deactivated result otherwise.
4. Enforce the CRUD transition guards of §6 exactly as specified (singleton spend, strict `sequence + 1`, immutable `did`/`entity_type`, dead-end `Revoked`) when implementing or verifying the on-chain validator.

**Test vector (informative):** `blake2b_256("did:phoenix:person:alice")` = `1643c0c9…c470` (32 bytes, hex, truncated for display — full vector set published alongside the Anchorme module's feature specification).

---

## 10. Reference Implementation

- **On-chain validator:** `github.com/PhoenixKeyDID/PhoenixKey-Validator` — `validators/taad.ak` and `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak`. Aiken 1.1.x, Plutus V3.
- **Resolver:** live HTTP resolver implementing `GET /identifiers/{did}` per §6.2 (PhoenixKey backend).
- **Test coverage (informative, not a conformance requirement):** the reference implementation includes a unit/property-style `aiken check` test suite covering the validator logic described in this document — genesis (Person and child), rotation, recovery, guardian updates, transfer, deactivation, and adversarial cases. Current pass counts are published in the implementation repository. This coverage is **not** a claim of external security audit completion, and it does not by itself close the anchor-uniqueness item described in §7.2.

The reference implementation's coverage of each operation is tracked in the project's status documentation; statements in this document should not be read as asserting production-readiness beyond what is explicitly scoped in §6–§7.

---

## 11. Registry Considerations

This method is registered in the W3C DID Method Registry (`w3c/did-extensions`, `methods/phoenix.json`, commit `97c8197`, 2026-07-04) with `"status": "registered"` and `"specification"` pointing at `https://phoenixkey.me/did-method/v1` — the canonical location of this specification. Publication of this document at that URL, and any future revision to it, should be reflected back into the registry entry (`specification` URL / version notes) so the two stay in sync. Contact for method-registry correspondence: GreenSun Tech, `contact@greensun.tech`.

---

## Appendix — Terminology cross-reference

| Term used here | Internal / code name |
|---|---|
| Identity module | **Anchorme** (module name as of 2026-07; previously "Identity") |
| Anchor datum | `TAADDatum` (Token-Anchored Authority Delegation), `types.ak` |
| Anchor name | `N(did) = blake2b_256(UTF-8(did))` |
| Multi-purpose validator | `taad` (`validators/taad.ak`) |
| Mint-gate logic | `lib/phoenixkey/state_nft_logic.ak` |
| Spend state-machine logic | `lib/phoenixkey/taad_logic.ak` |
| Wallet-side authorization read | `auth_logic.anchor_controller_ok`, used by `did_payment.ak` / `did_stake.ak` |

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
