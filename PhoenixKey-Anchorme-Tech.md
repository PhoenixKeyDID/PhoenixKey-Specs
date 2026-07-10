# PhoenixKey — Anchorme (Đặc-tả KỸ-THUẬT)

> **Module:** Anchorme (danh-tính lõi). **Đối tượng:** KỸ SƯ triển khai (đội on-chain, đội backend, Core/Enclave). **Loại doc:** Tech. **Ngày:** 2026-07-09.
> **Mục đích:** kiến-trúc + datum/redeemer CBOR + luồng tx + API backend + ranh-giới + thứ-tự deploy — đủ để code, không cần đọc lại Feat-Math.
> **Nguồn đối-chiếu (đã verify CODE THẬT):**
> - `PhoenixKey-Validator/validators/taad.ak` (63 dòng, Design-2 mint+spend)
> - `lib/phoenixkey/state_nft_logic.ak` (683 dòng — mint-gate + can_own)
> - `lib/phoenixkey/taad_logic.ak` (802 dòng — spend state-machine)
> - `lib/phoenixkey/auth_logic.ak` (87 dòng — anchor_controller_ok)
> - `lib/phoenixkey/types.ak` (144 dòng — TAADDatum/redeemer/EntityType)
> - Nguồn thiết-kế nội-bộ (không công khai).
>
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 1. Kiến-trúc

### 1.1 Sơ-đồ thành-phần

```
┌──────────────── CORE / ENCLAVE (Flutter + rust_core) ─────────────────┐
│  vân tay → Secure Enclave: HW_Key (P-256) + TAAD_Key (Ed25519)         │
│  DID = did:phoenix:base32(slot):hex(hash)  (rust_core taad_did.rs)     │
│  N(did) = blake2b_256(UTF-8(did))  (phoenix_address.rs:52)             │
│  ký: GenesisPerson(self), GenesisChild(owner), Rotate/Deactivate/Transfer│
└───────────────┬────────────────────────────────────────────────────────┘
                │ build-unsigned → Enclave witness → submit
                ▼
┌──────────────── BACKEND (PhoenixKey-Database, Java — đội backend) ─────┐
│  IdentityController /identity/register (PersonDID)                     │
│  ResolverController /identifiers/{did} (W3C DID Doc)                   │
│  IdentityController /identity/{did}/pubkey|document|status             │
│  /v1/registry/resolve/:did_hash (resolve-by-hash)                      │
│  PointInTimeController /identity/{did}/active|key-authorized|...       │
│  /identity/service/create (ServiceDID self-service)                    │
│  indexer #syncTaad → onchain_taad_state_cache (+ did_state_history)    │
└───┬──────────────────────────────────────────────────┬─────────────────┘
    │ submit tx                                         │ reference-input (đọc)
    ▼                                                   ▼
┌── CARDANO (Plutus V3, Preview) ───────────────────────────────────────┐
│  validator taad (Design-2, apply-param = ValidatorParams)             │
│    mint : GenesisPerson | GenesisChild{owner_did} | GenesisBurn        │
│    spend: Rotate | InitRecovery | CancelRecovery | FinalizeRecovery    │
│           | Deactivate | UpdateGuardians | Transfer                    │
│  own_policy ≡ script-hash → anchor NFT (π, N(did)) HARD-BIND Script(π) │
│  ANCHOR UTxO = Script(π) + [NFT(π,N(did))=1] + minADA + inline TAADDatum│
│      ▲ reference-input (ví did_payment/did_stake đọc controller)       │
│  did_payment.ak / did_stake.ak → auth_logic.anchor_controller_ok       │
└────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Bất-biến kiến-trúc (load-bearing)

- **1 danh-tính = 1 UTxO** tại `Script(π)`, mang **NFT singleton** `(π = own_hash, name = blake2b_256(did))`, min-ADA, inline `TAADDatum`. Neo `taad_logic.ak:55-58`, `state_nft_logic.ak:228-253`.
- **own_policy ≡ script-hash** (Design-2 đa-mục-đích): NFT policy CHÍNH là hash validator → validator PHẢI có `mint` handler; genesis HARD-BIND NFT vào `Script(own_policy)`. Neo `taad.ak:42`, `state_nft_logic.ak:233-248`. Đây là điểm đóng lỗ Design-1 (NFT ra ví lạ).
- **seq strict +1** mọi transition (chống replay); tx `seq ≤ on-chain` trượt. Neo mọi nhánh `taad_logic.ak`.
- **did + entity_type BẤT-BIẾN** qua mọi spend (`immutable_fields_preserved:310-320`) → xoay chìa/transfer không đổi danh-tính.
- **Revoked = dead-end:** `when (redeemer, status)` không khớp `(_,Revoked)` → `_ -> False` (`:222`).
- **🔴 auth_logic single-carrier** (`:59-70`) chống 2-anchor-TRONG-tx, KHÔNG chống 2-anchor-live-toàn-cục (gốc lỗ CID-1 §10).

### 1.3 Đường sinh DID + băm (Core — rust_core)

| Thành-phần | Công-thức | Nguồn |
|---|---|---|
| DID canonical | `did:phoenix:base32(slot):hex(H(encode(type) ‖ creator ‖ encode(slot) ‖ rand_256))` | Math §2.1; `DidPhoenixGenerator.java:55-95` |
| Regex | `^did:phoenix:[a-z2-7]{13}:[0-9a-f]{64}$` | `DID-Registry-Resolver-Feat §1.2` |
| `N(did)` / `did_hash` | `blake2b_256(UTF-8(did))` — 32B, KHÔNG salt, KHÔNG BLAKE3 | `phoenix_address.rs:52`; `…-Feat §1.1` |
| type-byte trong hash | `encode(type)` = BYTE enum. Java+Aiken **KHỚP byte-exact** (Person=0…Device=2,Machine=3,Asset=4,Bot=5,AI=6,Service=7,Context=8,Character=9); chỉ **văn Math §2.2 lệch thứ-tự** (Context=2). §10 CID-4. | `DidPhoenixGenerator.java:56-65` ≡ `types.ak:21-32`; văn Math §2.2 |

---

## 2. Datum / Redeemer — khuôn CBOR (khớp aiken ↔ rust_core)

> **NGUỒN CHUẨN = code `.ak`.** Plutus V3 dùng `Constr`. Thứ-tự field LÀ thứ-tự CBOR (rust_core PHẢI encode đúng thứ-tự).

### 2.1 `TAADDatum` — 10 field (thứ-tự CBOR CỐ ĐỊNH)

Neo `types.ak:34-63`. `Constr 0 [...]`:

| # | Field | Aiken type | CBOR | Ý-nghĩa | Genesis |
|---|---|---|---|---|---|
| 0 | `did` | `ByteArray` | bytes | `did:phoenix:...` UTF-8 | (DID mới) |
| 1 | `entity_type` | `EntityType` | Constr idx | 0=Person…9=Character (thứ-tự `types.ak:21-32`) | theo loại |
| 2 | `controller_pkh` | `VerificationKeyHash` | bytes(28) | băm TAAD_Key hiện-tại | băm ký-genesis |
| 3 | `hw_key_pubkey` | `ByteArray` | bytes | HW_Key P-256 SEC1 (33B nén / 65B thô) — carry-only | pubkey Enclave |
| 4 | `sequence` | `Int` | int | bộ đếm đơn-điệu | `0` |
| 5 | `status` | `TAADStatus` | Constr | 0=Active,1=Recovering{...},2=Migrated,3=Revoked | `Active` (idx 0) |
| 6 | `guardians` | `List<VKH>` | list | ≤5 guardian pkh | `[]` hoặc cấu-hình |
| 7 | `parent_did` | `Option<ByteArray>` | Constr 0/1 | `None`=Person; `Some(owner_did)`=con | `None` (Person) / `Some` (Child) |
| 8 | `revoked_slot` | `Option<Int>` | Constr 0/1 | slot Deactivate; `None`=sống | `None` |
| 9 | `recovery_anchor` | `Option<ByteArray>` | Constr 0/1 | LampNet CID backup (carry-only) | `None` |

> **Load-bearing genesis (pass `is_fresh`, `state_nft_logic.ak:294-300`):** `sequence=0 ∧ status=Active ∧ revoked_slot=None`. Field 9 `recovery_anchor` là con-trỏ off-chain, KHÔNG bị state-machine ràng (mirror rust_core `encode_taad_datum_create` field 9).

**Diễn-CBOR (GenesisPerson, ví-dụ rút gọn):**
```
d8799f                              # Constr 0 (tag 121), 10 field
  58xx <did UTF-8 bytes>            # did
  d87980                            # Constr 0 [] = entity_type Person
  581c <controller_pkh 28B>         # controller_pkh
  58xx <hw_key_pubkey SEC1>         # hw_key_pubkey (33B/65B)
  00                                # sequence 0
  d87980                            # Constr 0 [] = status Active
  80                                # guardians []
  d87a80                            # Constr 1 [] = parent_did None  (Option::None = Constr 1)
  d87a80                            # revoked_slot None
  d87a80                            # recovery_anchor None
ff
```
> **Chú-ý Option encoding:** aiken `Option` → `Some = Constr 0 [x]`, `None = Constr 1 []`. Điểm load-bearing = **10 field, đúng thứ-tự, fresh** để pass `find_single_minted_did` + genesis gate.

### 2.2 `StateNftRedeemer` (mint) — 3 nhánh

Neo `state_nft_logic.ak:69-77`:

| idx | Redeemer | CBOR | Ai dùng |
|---|---|---|---|
| 0 | `GenesisPerson` | `d87980` | Core (self-genesis PersonDID) |
| 1 | `GenesisChild { owner_did }` | `d87a9f <owner_did> ff` | owner (đẻ con non-Person) |
| 2 | `GenesisBurn` | `d87b80` | đóng anchor (mọi movement âm) |

### 2.3 `TAADRedeemer` (spend) — 7 nhánh

Neo `types.ak:65-129`:

| idx | Redeemer | Field | Ai ký |
|---|---|---|---|
| 0 | `Rotate` | `{new_controller_pkh, new_hw_pubkey}` | controller cũ |
| 1 | `InitRecovery` | `{new_controller_pkh, new_hw_pubkey, guardian_sig_count, collateral_lovelace}` | guardians (≥threshold) |
| 2 | `CancelRecovery` | — | controller (trước deadline) |
| 3 | `FinalizeRecovery` | — | pending_controller (sau deadline) |
| 4 | `Deactivate` | — | controller |
| 5 | `UpdateGuardians` | `{new_guardians}` | controller |
| 6 | `Transfer` | `{new_controller_pkh, new_hw_pubkey, new_guardians}` | controller cũ **+** mới (2-of-2) |

> Recovery (idx 1-3) = luồng module **Rebirthme**; ở đây chỉ liệt-kê để rust_core encode đủ enum. Thứ-tự biến-thể LÀ idx CBOR.

### 2.4 Tham-số apply-param (`ValidatorParams`)

Neo `types.ak:131-135`. Design-2 KHÔNG bake NFT-policy (policy ≡ script-hash tự-sinh):

| Param | Type | Nguồn | Ghi-chú |
|---|---|---|---|
| `recovery_timelock_slots` | `Int` | hằng hệ | timelock recovery (Rebirthme) |
| `min_guardian_sigs` | `Int` | hằng hệ | ngưỡng guardian |
| `recovery_collateral_lovelace` | `Int` | hằng hệ | collateral A-5 |

> **Địa-chỉ = f(ValidatorParams).** Cùng params → cùng script-hash = cùng policy = cùng address. `taad` là 1 validator toàn-hệ (KHÔNG per-DID apply-param) — per-DID tách qua NFT-name `N(did)`. (Khác `activation_vault`/`did_payment` per-DID.)

---

## 3. Từng thao-tác — điều-kiện + shape tx + ai-ký

Ký-hiệu: `d`/`d′` = datum vào/ra; `π` = own_policy; `N=blake2b_256(did)`.

### 3.0 Mint-gate (genesis)

#### `GenesisPerson` — `validate_mint(GenesisPerson)` (`:138-152`)
- **Điều-kiện:** (1) đúng 1 mint `+1` dưới `π`, carrier output DUY NHẤT tại `Script(π)`; (2) `name == blake2b_256(child.did)`; (3) `entity_type == Person`; (4) `parent_did == None`; (5) `controller_pkh ∈ extra_signatories` (self-sign); (6) fresh (seq=0,Active,revoked=None).
- **Ký:** owner-witness (Enclave, khóa tự-khai).
- **🔴 Gap:** KHÔNG kiểm did-string ↔ người thật; KHÔNG verify `hw`. Lỗ CID-1.
- **Shape:**
  ```
  mint: +1 NFT(π, N(did))   redeemer=GenesisPerson
  out:  [ANCHOR: Script(π) + NFT + minADA + inline TAADDatum(Person,fresh)]  [change]
  signer: controller_pkh (Enclave)
  ```

#### `GenesisChild { owner_did }` — (`:154-177`)
- **Điều-kiện:** như trên + (G-owner) `find_owner_reference` trả owner NFT ở `Script(π)`, `owner.did == owner_did`, `status Active`; (G-1) `owner.controller_pkh ∈ signers`; (G-2) `can_own(owner.entity, child.entity)`; (G-3) `child.parent_did == Some(owner_did)`; (G-4) `child.entity ≠ Person`.
- **Ký:** owner-controller (parent-sig). **AN-TOÀN với lỗ CID-1.**
- **Shape:**
  ```
  ref-input: [owner ANCHOR NFT(π, N(owner_did))]   # đọc owner authority (CIP-31)
  mint: +1 NFT(π, N(child_did))   redeemer=GenesisChild{owner_did}
  out:  [child ANCHOR: Script(π) + NFT + minADA + TAADDatum(entity≠Person, parent=Some(owner))]
  signer: owner.controller_pkh
  ```

#### `GenesisBurn` — (`:136,186-192`)
- **Điều-kiện:** pure-burn — `list.length(moved) > 0 ∧ ∀ movement < 0`. Đóng anchor.

### 3.1 `Rotate` — xoay chìa — (`:62-73`)
- **Điều-kiện (7):** `(Rotate,Active)`; ctrl cũ ký; `seq+1`; core immut; `ctrl′=new_ctrl`, `hw′=new_hw`; `status Active`; guardians giữ.
- **Ký:** controller hiện-tại. `did` giữ ⟹ danh-tính bất-biến.
- **Shape:** `in:[ANCHOR] redeemer=Rotate → out:[ANCHOR recreate: ctrl+hw mới, seq+1]  signer: ctrl cũ`.

### 3.2 `Deactivate` — đóng-băng — (`:163-183`)
- **Điều-kiện (9):** `(Deactivate,Active)`; ctrl ký; `seq+1`; did/entity/parent/ctrl/hw/guardians giữ; `status Revoked`; `revoked_slot = Some(lb)` (lb = finite lower-bound).
- **Ký:** controller. Revoked = dead-end.

### 3.3 `UpdateGuardians` — (`:186-198`)
- **Điều-kiện:** `(UpdateGuardians,Active)`; ctrl ký; `seq+1`; immut; `guardians′=new_guardians`; `len ≤ 5`; Active.

### 3.4 `Transfer` — bàn-giao ServiceDID 2-of-2 — (`:202-219`)
- **Điều-kiện (11):** `entity_type == Service`; **ctrl cũ VÀ ctrl mới cùng ký**; `new_ctrl ≠ ctrl`; `seq+1`; immut; `ctrl′=new_ctrl`, `hw′=new_hw`; Active; `value′ ≥ value`; guardian mới atomic (≤5).
- **Ký:** 2-of-2 (native Ed25519 witness — KHÔNG verify thêm on-chain). Cài guardian mới cùng tx tránh cửa-sổ zero-recovery.
- **Shape:**
  ```
  in:  [ANCHOR(Service)]   redeemer=Transfer{new_ctrl,new_hw,new_guardians}
  out: [ANCHOR recreate: ctrl+hw+guardians mới, seq+1]
  signers: [ctrl cũ, ctrl mới]
  ```

### 3.5 Recovery (Init/Cancel/Finalize) — tóm, chi-tiết ở Rebirthme
`InitRecovery` (`:76-110`): ≥`min_guardian_sigs` guardian ký, collateral ≥ ngưỡng khóa vào continuing UTxO (A-5), status→Recovering. `CancelRecovery` (`:116-131`): `ub < deadline_slot` (A-6), ctrl ký, →Active. `FinalizeRecovery` (`:134-157`): `lb > deadline_slot`, `pending_controller_pkh` ký, cài hw/ctrl pending, →Active. **Ranh-giới MECE: module Rebirthme sở-hữu; Core-Anchorme chỉ neo 3 nhánh này giữ A-1/A-2.**

### 3.6 `anchor_controller_ok` — ví "đi-theo-DID" đọc anchor (`auth_logic.ak:37-52`)
```
find_anchor_datum(refs, π, N(did)) = Some(anchor)   # ĐÚNG 1 carrier, else None→REJECT
∧ is_active(anchor.status)
∧ list.has(extra_signatories, anchor.controller_pkh)
```
Dùng bởi `did_payment.ak`/`did_stake.ak` (module Rebirthme). KHÔNG ép entity_type (`:16-18`) → mọi loại DID chi được — đây là bề mặt CID-1, đóng bằng PA2/PA5-a (→ [STATUS](./PhoenixKey-STATUS.md#anchorme)).

### 3.7 Bảng ai-ký

| Thao-tác | Ký | Điều-kiện pha |
|---|---|---|
| GenesisPerson | controller (self) | mint |
| GenesisChild | owner.controller | mint, owner Active |
| Rotate | controller cũ | Active |
| Deactivate | controller | Active |
| UpdateGuardians | controller | Active |
| Transfer | ctrl cũ **+** mới | Active, Service |
| Init/Cancel/Finalize | guardian / ctrl / pending | (Rebirthme) |

---

## 4. Luồng end-to-end

### 4.1 Tạo PersonDID (self-genesis)
```
Core: keygen vân tay → HW_Key(P-256) + TAAD_Key(Ed25519) trong Enclave
  → DID = did:phoenix:...  →  N(did)=blake2b_256(did)
1a POST /identity/register: backend verify Genesis-sig → build tx mint GenesisPerson
   → hoặc publish metadata-6789 (đường PersonDID đang live)
Core: Enclave witness (controller) → submit
Kết-quả: ANCHOR(Active, seq=0), DID Document resolve được.
```

### 4.2 Đẻ ServiceDID / con (GenesisChild)
```
Owner (Person/Org) đã có ANCHOR live.
1 build tx: ref-input owner ANCHOR → mint GenesisChild{owner_did}
   → child ANCHOR(parent=Some(owner), entity=Service/Device/...)
signer: owner.controller
endpoint /identity/service/create (ServiceDID self-service, ServiceDID-…-DRAFT §1.3)
```

### 4.3 Resolve W3C (LIVE)
```
GET /identifiers/{did}  → DID Document (verificationMethod: HW_Key→authentication;
                          TAAD_Key→capabilityInvocation; serviceEndpoint[])
GET /identity/{did}/pubkey|document|status
```

### 4.4 Resolve-by-hash (đội backend)
```
GET /v1/registry/resolve/{did_hash}         # did_hash = blake2b_256(UTF-8(did)) 64-hex
GET /v1/registry/resolve/{did_hash}?at=<ts> # time-keyed (ts→slot→epoch)
→ cần bảng index ngược did_hash→did (chưa dựng); Strata daemon hết 424.
```

### 4.5 Point-in-time (đội backend, migration V16)
```
GET /identity/{did}/active?epoch=      → active/revoked_at/never_existed
GET /identity/{did}/key-authorized?key=&epoch=   → MAGIC did_commit + Rada dùng CHUNG
GET /identity/{did}/revocation?epoch=  · /lineage · /dependency-chain · /ownership-graph-snapshot
Bảng append-only: did_state_history, did_lineage, did_ownership_edge; fail-closed 503.
```

### 4.6 Xoay chìa / đóng-băng
```
Rotate:     build tx spend ANCHOR redeemer=Rotate → recreate ctrl+hw mới, seq+1  (ctrl cũ ký)
Deactivate: build tx spend ANCHOR redeemer=Deactivate → Revoked, revoked_slot=lb  (ctrl ký)
```

---

## 5. API backend (tham-chiếu)

Prefix `/api/v1` (một số resolver dùng `/identifiers`, `/identity`), body snake_case, `DataResponse<T>{code,message,result}` (`code=1000` OK). Mẫu chuyển-giá-trị **build-unsigned → Enclave witness → submit**.

| # | Method | Path | Thao-tác | Chủ |
|---|---|---|---|---|
| 1 | POST | `/identity/register` | mint GenesisPerson / metadata-6789 | đội backend |
| 2 | GET | `/identifiers/{did}` | W3C DID Document | đội backend |
| 3 | GET | `/identity/{did}/pubkey\|document\|status` | resolve field | đội backend |
| 4 | GET | `/v1/registry/resolve/:did_hash` | resolve-by-hash | đội backend |
| 5 | GET | `/identity/{did}/active\|key-authorized\|revocation\|lineage\|...` | point-in-time (8 endpoint) | đội backend |
| 6 | POST | `/identity/service/create` | ServiceDID self-service | đội backend |
| 7 | POST | `/api/v1/devices/did` | DeviceDID create (Op_create_device) | đội backend + đội on-chain |
| 8 | GET | `/api/v1/device/{device_did}/verify?person_did=` | verify device↔owner | đội backend |

**Point-in-time semantics (§5 point-in-time spec):** `row(d,q) = argmax{e_i ≤ q}`; append-only (PIT-1, không UPDATE/DELETE); fail-closed 503 khi `latest_synced_epoch < epoch(at) − 1` (PIT-6). Pubsub 4 topic outbox + ACK 30s; `did_revoked` cần 2-of-3 DAO multisig.

**Resolve-by-hash (§2 resolver spec):** `did_hash = blake2b_256(UTF-8(did))`, 64-hex lowercase (KHÔNG BLAKE3 — trùng anchor-name on-chain, cross-check TAAD UTxO). 5 test-vector thật (`…-Feat §1.3`). low-S per-curve: P-256/secp256k1 bắt-buộc low-S; Ed25519 n/a (`crypto.rs:339`).

---

## 6. Ranh-giới giao-việc

| Tầng | Việc | Đội |
|---|---|---|
| **On-chain (Aiken)** | validator `taad` Design-2: 3 mint-gate + 7 spend, A-1/A-2/A-3, can_own, Transfer 2-of-2, Deactivate. | **đội on-chain** |
| **On-chain** | **PA5-a** entity-gate (`anchor_controller_ok` + allowed-entities); **PA2** UniquenessThread (spend-based, sorted-list K=256 / Merkle dân-số); **DeviceDID** `device_did.ak` + `device_did_mint`. | **đội on-chain** |
| **Backend (Java)** | register, resolver W3C; resolve-by-hash index, point-in-time V16 (5 bảng append-only + 8 endpoint + pubsub), ServiceDID self-service, DeviceDID endpoints + hw_cert verify. `curl` verify sau deploy. | **đội backend** |
| **Core / Enclave** | keygen vân tay (HW_Key P-256 + TAAD_Key Ed25519); ký genesis/rotate/transfer/deactivate; `N(did)` băm; type-byte đồng-bộ. | **Core** |
| **Math** | Full_Authority `⊑` fix (v4.7); type-code canonical §2.1. | maintainer |

**Ranh-giới ký:** self-genesis = Enclave owner-witness; GenesisChild = owner-controller; Rotate/Deactivate/UpdateGuardians = controller hiện-tại; Transfer = 2-of-2 native witness.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 7. Thứ-tự deploy + phụ-thuộc-chặn

**Thứ-tự triển-khai (không phụ-thuộc, làm song-song được):**
1. Deploy validator `taad` (Design-2, apply `ValidatorParams`). `plutus.json` phải khớp 10-field datum + 3-mint + 7-spend redeemer.
2. Backend register PersonDID + resolver W3C.
3. GenesisChild (Org/Service/Device...) — chỉ cần owner ANCHOR live.
4. Rotate / Deactivate / Transfer / UpdateGuardians — chỉ cần validator + anchor.

**Phụ-thuộc-chặn (thứ-tự bắt buộc):**
5. **resolve-by-hash + point-in-time** — đội backend (bảng index ngược + did_state_history V16). Chặn Strata/VeData/MAGIC did_commit.
6. **PA5-a + PA2** — đội on-chain. **Chặn mở GetLAMP-PersonDID + custody-PersonDID production** (lỗ CID-1, xem `PhoenixKey-Anchorme-Math.md` §9). Org/Service KHÔNG chặn.
7. **DeviceDID** — đội on-chain (`Op_create_device`) + đội backend (hw_cert verify). Chặn LampNet node admission / Knowme device / Rada.
8. **Registries + Permission/Consent + ServiceDID self-service** — chờ duyệt.

**Đồ-thị chặn:**
```
taad validator ──┬──► register/resolve ──► GenesisChild (Org/Service)
                 │
                 ▼
   PA2/PA5 ─────► mở PersonDID-custody production   [BLOCKER CID-1]
   backend V16 ─► resolve-by-hash + point-in-time    [BLOCKER Strata/VeData/MAGIC]
   on-chain+backend ► DeviceDID                       [BLOCKER LampNet/Knowme/Rada]
```

---

## 8. Test / evidence — bao-phủ bắt-buộc

Bộ test module danh-tính (`taad_logic`, `state_nft_logic`, `attack_tests`) phải phủ:
- **GenesisPerson:** happy-path; NFT off-script FAIL (Bug#3 regression); no-sig FAIL; parent≠None FAIL; name≠N(did) FAIL (A-1); mint-2 FAIL; two-DID FAIL; seq≠0 FAIL.
- **GenesisChild:** happy-path; no-owner-sig FAIL (G-1); CanOwn-vi-phạm FAIL (G-2); parent sai FAIL (G-3); missing-ref FAIL; Org→Person FAIL; Service→Asset OK.
- **can_own:** Person→Org=T, Person→Person=F, Machine→Device=T, leaf→any=F.
- **GenesisBurn:** pure-burn OK; burn+mint FAIL.
- **Rotate:** NFT plumbing, recovery_anchor carry, NFT-missing FAIL.
- **CancelRecovery / FinalizeRecovery / Deactivate:** A-6 window, pending-sig, seq/revoked_slot binding.

Bổ-sung bắt-buộc trước khi mở production:
- Round-trip CBOR aiken↔rust_core: encode `TAADDatum` 10-field đúng thứ-tự + Option encoding (`Some=Constr0`, `None=Constr1`); decode redeemer idx.
- Testnet e2e Preview: GenesisPerson → Rotate → GenesisChild → Transfer(Service) → Deactivate. `curl` từng resolver-endpoint sau deploy.
- **Red-team CID-1 phải re-run và xác-nhận CLOSED** (`redteam_mint.collide_person_over_org_name`) — auditor xác-nhận trước khi mở production PersonDID.
- Test-vector `did_hash` (5 vector, `…-Feat §1.3`) cho Strata mock-registry.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 9. Ranh-giới + luật thiết-kế bắt-buộc

- **CID-1 [cổng GO/NO-GO production PersonDID]:** khâu đúc con dấu Người phải được đóng bằng **PA2 UniquenessThread** trước khi mở GetLAMP/custody PersonDID production. Org/Service AN-TOÀN (parent-sig), không bị chặn. Đây là lỗ **mã-hoá anchor**, KHÔNG phải sinh-trắc/sybil — xem `PhoenixKey-Anchorme-Math.md` §8-9.
- **PA2:** UniquenessThread spend-based. Sorted-list K=256 scale ~triệu (N/shard ≤ 400); dân-số → Merkle-root-in-datum. **Địa-chỉ ví PHẢI GIỮ NGUYÊN** (không đổi compile-param did_payment/stake/subaddr) — đây là ràng buộc thiết kế bắt buộc, không phải tuỳ chọn.
- **resolve-by-hash + point-in-time:** cần index ngược + `did_state_history` V16 (đội backend) trước khi Strata/VeData/MAGIC did_commit dùng được.
- **DeviceDID:** `Op_create_device` (đội on-chain) + hw_cert verify (đội backend, validator chỉ neo hash) — cả hai phải xong đồng thời để tránh cert giả lọt.
- **Type-code:** canonical LÀ bảng-byte `DidPhoenixGenerator.java:56-65` ≡ `types.ak:21-32` (Device=2…Service=7,Context=8). Văn Math §2.2 PHẢI bám bảng-byte này. Rà bản Rust/mobile bám đúng trước khi mint author-DID phi-nhân.
- **`Migrated` status:** cần maintainer chốt ai/khi-nào chuyển Active→Migrated, hoặc bỏ field.
- **Registries + Permission/Consent + ServiceDID self-service:** chờ duyệt trước khi triển khai.
- **Ranh-giới sửa code:** validator (đội on-chain) + backend (đội backend) thuộc PhoenixKey backend — nhóm tài-liệu KHÔNG sửa; phát-hiện lỗi → báo maintainer / tạo Issue. Tài-liệu này chỉ đặc-tả.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## Nguồn

- `PhoenixKey-Validator`: `validators/taad.ak`, `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak`.
- `PhoenixKey-Specs/PhoenixKey-Math.md` §2, §4–§5, §10, §22.
- Nguồn thiết-kế nội-bộ (không công khai).
- `PhoenixKey-Core/Enclave/rust_core`: `phoenix_address.rs:52`, `crypto.rs:339`; `PhoenixKey-Database`: `DidPhoenixGenerator.java`, `ResolverController.java`.
