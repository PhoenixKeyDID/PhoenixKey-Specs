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

### 5.1 Resolve-by-hash — chi-tiết (`DID-Registry-Resolver-Feat.md §1-2`)

**5 test-vector thật** (sinh bằng `rust_core::anchor_name_from_did`; vector #1 khớp hằng `VEC_NAME` trong code — tự-kiểm):

| # | `did_canonical` | `did_hash = blake2b_256(UTF-8(did))` |
|---|---|---|
| 1 | `did:phoenix:person:alice` | `1643c0c9f776c4b173c475019e8d83b59e0269fa5d21f7fbe8e15af14c9ac470` |
| 2 | `did:phoenix:aaaaaaahl4nn6:ccd1feb65ec57c150b2773b4906630c98cf03e5c1b765f6f5d7ee87092466537` | `46cfd5bda656aa19ed4f4fc55aec3d32a4e7985d5516a95fee8ee5bc16b7ff41` |
| 3 | `did:phoenix:aaaaaaac3wnaa:0000000000000000000000000000000000000000000000000000000000000000` | `8d00ada26fea7b3d1305569da0d0529c4ddda23b8b3f8c30c881141702ec0cae` |
| 4 | `did:phoenix:person:bob` | `15b76ce74d85d716b2c3877b553d26b756acdebbd9b5b245c835e2aaf3647d56` |
| 5 | `did:phoenix:bacaajbsuz4pg:1111111111111111111111111111111111111111111111111111111111111111` | `3a515ee8ef60b9382efe80ceb12deb2f03cdbda8aa7390d78d962f240ddfa108` |

> #1/#4 dạng dev `did:phoenix:person:<name>`; #2/#3/#5 dạng canonical production `did:phoenix:<slot13>:<hash64>` (regex `^did:phoenix:[a-z2-7]{13}:[0-9a-f]{64}$`). Hàm băm không phân biệt dạng — băm UTF-8 nguyên văn preimage.

**Response schema (200)** — `GET /v1/registry/resolve/{did_hash}[?at=<unix_ts>]`:

```jsonc
{
  "did":        "did:phoenix:aaaaaaahl4nn6:ccd1feb6...",   // DID phản-tra từ hash
  "did_hash":   "46cfd5bd...ff41",
  "keys": [
    {
      "key_id":     "did:phoenix:...#hw-key-1",
      "pubkey_hex": "04a1b2...",          // sec1 (P-256) hoặc raw 32B (Ed25519)
      "curve":      "P-256",              // "Ed25519" | "P-256" | "secp256k1"
      "key_role":   "owner",              // owner | manager | viewer (AuthorizedKey.keyRole)
      "purpose":    ["authentication","assertionMethod"],   // W3C purpose
      "valid_from": 512,                  // Cardano epoch key có hiệu lực (điểm rotate)
      "valid_to":   null,                 // epoch key hết hiệu lực (null = còn active)
      "revoked_at": null,                 // epoch bị revoke; null nếu chưa
      "low_s":      "n/a"                 // "required" cho P-256/secp256k1; "n/a" cho Ed25519
    }
  ],
  "delegation_chain": [],                  // rỗng nếu DID không phải delegate; entry = DelegationToken (Math §4.4/§23)
  "status":       "active",                // active | revoked | recovering | migrated
  "at_epoch":     460,                     // epoch response được đánh giá (nếu ?at= truyền)
  "resolved_from": "onchain-cache"         // onchain-cache | metadata-6789
}
```

**Bảng HTTP status:**

| Code | Khi nào |
|---|---|
| 200 | Resolve OK, `keys[]` có giá trị |
| 400 | `did_hash` sai format (không phải 64 hex) |
| 404 | `did_hash` không map tới DID nào đã đăng ký |
| 410 | DID tồn tại nhưng đã deactivate/revoke tại `at` (trả `status=revoked`) |
| 503 | Index tụt hậu — chưa đủ dữ liệu khẳng định state tại `at` (fail-closed, khớp PIT-6) |

> **424 mà Strata gặp** = daemon trả "Failed Dependency" vì resolve upstream chưa có; hết 424 sau khi endpoint này live.

**Time-keyed `?at=` — ánh-xạ ts→epoch + chọn key hiệu-lực** (`DID-Registry-Resolver-Feat.md §3`, khớp Math §5.1 rotate + point-in-time §5): xoay khoá KHÔNG phá verify version cũ — mỗi key trong `keys[]` (schema trên) hiệu-lực tại thời-điểm `at` theo:

```
key_valid_at(k, at) ⟺
   k.valid_from ≤ at
   ∧ (k.valid_to = null ∨ at < k.valid_to)
   ∧ (k.revoked_at = null ∨ at < k.revoked_at)
```

Server chấp-nhận CẢ HAI `?at=<unix_ts>` VÀ `?epoch=<n>` (point-in-time §5.2 dùng `?epoch=` thuần) — dịch nội-bộ `ts → slot → epoch`: `slot = (ts − shelley_start_ts) + shelley_start_slot`; `epoch = shelley_start_epoch + (slot − shelley_start_slot) / epoch_length`, với `epoch_length = 432000 slot` (5 ngày), `1 slot = 1s`; hằng `shelley_start_*` KHÁC nhau giữa Preprod/mainnet. Lý-do nhận cả `?at=`: Strata daemon có timestamp record sẵn (không có epoch), nội-bộ vẫn quy về epoch để tra đúng `did_state_history`.

**🔴 Cần đối chiếu trước khi mint author-DID phi-nhân (Bot/Agent/Service):** `DID-Registry-Resolver-Feat.md §7.2` chỉ ra bảng type-byte trong `DidPhoenixGenerator.java:56-65` KHÔNG khớp Math §2.1 ở Context/Device/Machine/Asset/Bot/Agent/Service (Person/Org/Character trùng nên PersonDID không bị ảnh hưởng — bug chưa lộ). §1.3/§9 tài-liệu này ghi type-byte Java↔Aiken khớp và quy lệch về riêng văn Math §2.2; đây là 2 phép so-sánh khác cặp nguồn (Java↔Aiken vs Java↔Math §2.1) — **cần Long/Core đối chiếu lại cả 3 nguồn (Math §2.1, `DidPhoenixGenerator.java`, `types.ak`) trước khi mint author-DID phi-nhân đầu tiên**, không tự suy diễn khớp/lệch ở đây.

### 5.2 Point-in-time — response schema + fail-closed contract (`PointInTime-Resolve-API-Feat-Math.md`, migration V16)

Bảng lưu (Postgres, append-only, đội backend dựng): `did_state_history` (controller + active/revoked theo `effective_epoch`), `did_lineage` (PersonDID↔member_did, whitewashing defense), `did_ownership_edge` (cạnh sở-hữu per-epoch, ownership graph), `pubsub_event_log`/`pubsub_subscription` (outbox + ACK). App-user chỉ có `INSERT`+`SELECT` trên các bảng history — KHÔNG `UPDATE`/`DELETE` (chặn ở tầng GRANT DB, ngoại-lệ duy-nhất: GC role `phx_gc` re-sync khi reorg).

**Ngữ-nghĩa chọn dòng tại epoch `q`:** `row(d,q) = argmax{effective_epoch ≤ q}`; rỗng ⇒ `never_existed=true`. `active(d,q) = row tồn-tại ∧ ¬(is_revoked ∧ revoked_epoch ≤ q)`. `key_authorized(k,d,q) = active(d,q) ∧ (k == controller_pkh ∨ k == controller_pubkey_hex)` của `row(d,q)` — đây là call MAGIC `did_commit` bind và Rada revocation dùng CHUNG.

**8 endpoint, response `data` (snake_case, bọc `DataResponse`):**

| Endpoint | `data` trả về |
|---|---|
| `GET /identity/{did}/active?epoch=` | `{active, revoked_at, never_existed}` |
| `GET /identity/{did}/key-authorized?key=&epoch=` | `{authorized}` (+ `rotation_gap_violation` nếu vi-phạm `MAX_KEY_ROTATION_GAP=2`) |
| `GET /identity/{did}/revocation?epoch=` | `{status: not_revoked\|revoked, reason}` |
| `GET /identity/{did}/lineage` | `{person_did, lineage: [{did, registration_epoch, revocation_epoch, revocation_reason}]}` |
| `GET /identity/{did}/dependency-chain` | `{chain: [{did, edge_type}], terminal_person_did}` — traverse `did_ownership_edge` lên PersonDID, giới-hạn 32 hop chống vòng-lặp (`422 DEPENDENCY_CHAIN_CYCLE`) |
| `GET /identity/ownership-graph-snapshot?epoch=` (bắt-buộc) | `{epoch, nodes[], edges: [{parent_did, child_did, edge_type}]}` |
| `GET /identity/are-independent?dids=&hop_threshold=2` | `{independent}` — false nếu tồn-tại cặp share ancestor ≤ hop_threshold |
| `GET /identity/shared-ancestor?did_a=&did_b=&max_hop=2` | `{shared_ancestor}` (null nếu không có) |

**Fail-closed 503 (PIT-6) — phân-biệt với `never_existed`:** `never_existed=true` là câu trả-lời XÁC-ĐỊNH (đã sync qua epoch đó, DID không tồn-tại). `503` là khi CHƯA sync tới epoch được hỏi — `latest_synced_epoch < requested_epoch − k` (`k` đề-xuất =1, khớp SLA ≤1-epoch-staleness). Body:
```
503 { "code": 5030, "message": "resolve_unavailable", "data": {
    "reason": "index_behind" | "data_gap" | "backend_down",
    "latest_synced_epoch": 458,
    "requested_epoch": 460
}}
```
Consumer nhận 503 ⇒ **REJECT, không proceed** (fail-closed) — Stamp/MAGIC/Rada/LampNet đều hard-fail; **ngoại-lệ Score** được dùng cached snapshot ≤5 epoch cũ rồi downweight thay vì hard-fail.

**Pubsub — 4 topic, transactional outbox** (ghi `pubsub_event_log` CÙNG transaction với append history): `did_created`, `did_lineage_updated`, `did_revoked` (**bắt-buộc `dao_multisig` 2-of-3**, thiếu ⇒ dispatcher DROP + log `REVOKE_MULTISIG_MISSING`), `device_did_bound_to_person`. Delivery at-least-once, ACK 30s (quá hạn ⇒ replay backoff `1s→60s` cap); `POST /pubsub/ack`, `POST /pubsub/replay {subscriber_id, topic, from_slot}`; anti-replay theo `(cardano_slot, event_nonce)` đơn-điệu unique.

**Finality (chống reorg làm sai lịch sử):** indexer chỉ ghi `did_state_history` từ block đã **finalize** (neo Mithril snapshot / độ sâu xác-nhận k-block); `latest_synced_epoch` chỉ tăng khi block finalize. Nếu reorg xảy ra tại epoch đã ghi, GC-role `phx_gc` xoá các dòng history có `cardano_slot ≥ reorg_slot` rồi re-append từ chain — đây là **ngoại-lệ DELETE duy-nhất**, có audit log.

**Retention/GC:** `did_state_history`/`did_lineage` giữ ≥365 epoch sau revocation; `did_ownership_edge` snapshot giữ ≥730 epoch. GC do role DB riêng `phx_gc` (không phải app-user); app-user không có `DELETE`.

**Invariants PIT-* (bắt-buộc verify trước khi báo PASS):**

| ID | Invariant | Enforce ở đâu |
|---|---|---|
| PIT-1 | Append-only — không UPDATE/DELETE dòng quá khứ `did_state_history`/`did_ownership_edge`; thay đổi = INSERT dòng mới. | GRANT DB (app-user không UPDATE/DELETE) + `uq_did_epoch`; ngoại-lệ duy-nhất = reorg re-sync bởi `phx_gc` có audit log. |
| PIT-2 | Point-in-time correctness — `row(d,q)` luôn là dòng `effective_epoch` lớn nhất `≤ q`, không lẫn state tương-lai. | Index `(did, effective_epoch DESC)` + `WHERE effective_epoch ≤ :q ORDER BY … LIMIT 1`. |
| PIT-3 | Monotonic epoch — `latest_synced_epoch` chỉ tăng; không ghi từ block chưa finalize; `effective_epoch` mới ≥ đã có cho cùng DID. | Indexer check trước INSERT + optimistic lock `last_synced_block`. |
| PIT-4 | Revocation monotonic — `revoked_epoch`/`revocation_epoch` một khi set thì immutable (chỉ NULL→value). | Trigger DB reject update từ non-null. |
| PIT-5 | Lineage immutable — entry `did_lineage` không revise; đổi lineage = append `member_did` mới + emit `did_lineage_updated`. | `uq_person_member`; không cấp UPDATE cho `member_did`/`registration_epoch`. |
| PIT-6 | Fail-closed — thiếu dữ-liệu/index behind ⇒ 503, không trả state đoán. | Check `latest_synced_epoch` vs `query_epoch` (§5.2 trên). |
| PIT-7 | Outbox atomicity — event pubsub ghi CÙNG transaction với append history ⇒ không mất/không thừa event. | `@Transactional` bao cả append + INSERT `pubsub_event_log`. |
| PIT-8 | Anti-replay — `(cardano_slot, event_nonce)` đơn-điệu, unique. | `uq_slot_nonce`. |

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

## 10. DID Authorization Registry — primitive uỷ-quyền dùng chung (PROPOSAL, CHỜ DUYỆT)

> **Trạng-thái: PROPOSAL — chờ Long/anh Aladin chốt. CHƯA build, CHƯA đụng repo Validator.** Nguồn: `spec-proposals/DID-Authorization-Registry-DRAFT.md`. Tổng-quát-hoá Mint-Authority Registry (`registry.ak`, đã build & test trong LAMP) thành một bảng uỷ-quyền do DID quản, đọc qua reference-input, dùng chung cho nhiều loại verifier (không chỉ mint token). Đây là **primitive MỚI, tách khỏi `TAADDatum`** (§2) — registry là UTxO riêng, không phải trường trong anchor.

### 10.1 Vì sao ACL, không phải capability

Registry = **Access-Control List do DID-quản giữ**: người-được-uỷ-quyền không cầm token gì, chỉ ký tx; verifier tự tra bảng qua reference-input. Khác hẳn **object-capability** (`DelegationToken`, PhoenixKey-Math §23) — nơi người-được-uỷ-quyền CẦM bearer-credential và xuất trình. Hai paradigm bổ trợ nhau, không thay thế: cần verifier-tra-bảng-ổn-định → dùng registry; cần uỷ-quyền-narrow-chuyền-tay → dùng `DelegationToken`.

### 10.2 Datum (đề xuất, byte layout kế-thừa `registry.ak` v1)

```aiken
// PROPOSED — đổi tên khái niệm từ registry.ak (Authority→Authorization, token_tag→action_tag);
// byte layout CBOR GIỮ NGUYÊN so với v1 đã build.

pub type Authorization {
  SinglePkh(VerificationKeyHash)                                    // [constr 0]
  MultiSig { keys: List<VerificationKeyHash>, threshold: Int }      // [constr 1]
  Revoked                                                            // [constr 2]
  // tương lai, chỉ nối-tiếp constr index: SpendLimit [3], TimeBox [4]
}

pub type RegistryEntry {
  action_tag: ByteArray,        // hành-động/tài-nguyên, namespace bằng tiền-tố: "mint:T", "spend:treasury-ops", "device:vault-7:open"
  authorization: Authorization,
}

pub type RegistryDatum {
  governing_did: ByteArray,        // DID-quản; tên NFT singleton = blake2b_256(governing_did)
  entries: List<RegistryEntry>,    // key action_tag DUY NHẤT trong 1 registry
}
```

**Mỗi registry đúng-một `governing_did`** (per-DID, KHÔNG global/PKI trung ương — liên-bang các ACL nhỏ, mỗi cái neo vào 1 DID chủ-quyền). NFT singleton hard-bind vào `Script(own)`; update giữ `governing_did` bất-biến; entries không trùng `action_tag`.

### 10.3 Update — neo vào controller hiện-tại của DID-quản (rotatable/revocable/recoverable)

Update-quyền đọc `controller_pkh` **động** từ anchor NFT của `governing_did` qua reference-input (đúng cơ-chế `anchor_controller_ok`, §3.6) — KHÔNG có khoá-riêng cho registry. Hệ-quả trực-tiếp từ kiến-trúc Anchorme:
- **Rotatable:** `Rotate` (§3.1) trên anchor → update registry kế tự nhận `controller_pkh` mới, không redeploy registry.
- **Revocable:** set entry = `Revoked` → tức-thì, không cần thu-hồi vật-mang.
- **Recoverable:** mất khoá controller → guardian recovery TAAD (`InitRecovery`/`FinalizeRecovery`, §3.5, module Rebirthme) → controller mới tiếp-tục quản bảng. Registry **không có recovery-surface riêng** → một đường-recovery-duy-nhất cho mọi action_tag/thiết-bị treo trên registry đó.
- Đề-xuất v1 KHÔNG ép `status==Active` của `governing_did` tại thời-điểm update registry (bảo-toàn quyền-liên-tục qua cửa-sổ recovery); mỗi verifier tự quyết có double-check Active hay không tuỳ độ nhạy-cảm action_tag.

### 10.4 Verifier-agnostic — ba mẫu đề-xuất

Mọi verifier dùng chung một hàm thuần `authorization_satisfied(authz, signers)`, khác nhau ở `action_tag` quan-tâm + gate phụ tầng riêng:

| Verifier | action_tag ví-dụ | Gate phụ | Trạng-thái |
|---|---|---|---|
| **token-mint** | `mint:T` | asset-name == token_asset_name, qty>0 | ĐÃ build & test trong LAMP `registry.ak` (16 test pass) |
| **treasury-spend** | `spend:treasury-ops` | output trả về treasury-addr; (tương lai `SpendLimit`) max/cửa-sổ | PROPOSED |
| **device-action** | `device:vault-7:open` (routine) / `device:vault-7:reset-owner` (sensitive, bắt-buộc on-chain) | firmware cache snapshot cho routine (offline-tolerant); on-chain bắt-buộc cho sensitive | PROPOSED |

Vật-lý qua cầu nối (thiết bị không chạy Plutus): firmware DeviceDID cache snapshot registry định-kỳ qua cầu-nối tin-cậy, ép cục-bộ ngay cả offline; thao-tác nhạy-cảm (đổi chủ thiết bị, reset firmware) vẫn PHẢI là tx on-chain thật để có dấu-vết.

### 10.5 Giới-hạn ACL trung-thực (không che)

- **Confused-deputy:** verifier phải ép `action_tag` CỐ-ĐỊNH trong logic của nó (không nhận tag tuỳ-ý từ redeemer người gọi) — một tag = một hành-động hẹp nhất có nghĩa.
- **Uỷ-quyền thô:** `SinglePkh`/`MultiSig` không diễn-đạt được "chỉ lần này"/"tối đa N"/"cửa-sổ T". Giảm-thiểu: mở-rộng `Authorization` (`SpendLimit`/`TimeBox`, nối-tiếp constr index) HOẶC bổ-trợ `DelegationToken` (Math §23) khi cần chuyền-tay hẹp.
- **Single-object serialization:** registry là 1 UTxO → update tuần-tự; đọc reference-input KHÔNG tiêu UTxO nên verify song-song không giới-hạn. Update kỳ-vọng hiếm (sự-kiện vận-hành).
- **Ranh-giới cứng:** registry gate **AI** được làm action (ACL) — KHÔNG gate **BAO NHIÊU**. Cap tổng-cung (vd LAMP 36 tỷ cố-định, `LAMP/Treasury/CONTRACT.md §5`) vẫn là tầng SupplyState/Genesis độc-lập, compose bên ngoài registry.
- **KHÔNG hợp** cho: tính phiếu trọng-số (VotingPower là tích-nhân C1-C4, không token-weighted — tầng riêng), quyết-định dưới-giây on-chain thuần (cần firmware cache), uỷ-quyền unlinkable (registry công-khai ánh-xạ khoá↔action, không riêng-tư — cần ZK ngoài phạm-vi primitive này).

### 10.6 Việc cần chốt trước khi build

1. Đổi tên `Authority→Authorization`, `token_tag→action_tag` trong `registry.ak` (LAMP) + mirror Validator — duyệt trước khi đụng code (đổi tên cosmetic, CBOR không đổi).
2. Verifier nào bắt-buộc double-check `governing_did.status==Active` (đề-xuất: action nhạy-cảm CÓ, routine tuỳ).
3. Chuẩn cầu-nối firmware cache snapshot + ngưỡng chống-cache-cũ cho §10.4 device-action.
4. Thứ-tự constr cho `SpendLimit`/`TimeBox` khi mở-rộng `Authorization`.
5. **Chống datum phình khi `entries` dài** (nhiều action_tag trong 1 registry): (a) mặc-định — tách nhiều Registry NFT theo namespace (1 registry/nhóm action_tag liên-quan, vd 1 registry/OrgDID cho token nội-bộ, registry riêng cho token đối-tác); (b) khi N rất lớn — đổi `entries` sang lưu Merkle-root trong datum, verifier nhận proof entry kèm redeemer (đánh-đổi: mỗi lần dùng phải mang proof). (a) là lựa-chọn mặc-định; (b) chỉ cân-nhắc khi (a) không đủ. Nguồn: `Mint-Authority-Registry-DRAFT.md §5`.

→ Xem đầy-đủ: `spec-proposals/DID-Authorization-Registry-DRAFT.md`.

---

## 11. Permission & Consent — luồng app↔user cấp quyền (PROPOSAL, CHỜ DUYỆT)

> **Trạng-thái: PROPOSAL — chờ Long/anh Aladin chốt. CHƯA build.** Nguồn: `spec-proposals/PhoenixKey-Permission-and-Consent-Spec.md`. Khác **§10 (DID Authorization Registry)**: §10 là ACL on-chain cho *authority* (ai được mint/spend/vận hành thiết-bị on-chain), verifier = validator. Mục này là *off-chain data-access consent* (ai được xem/sửa tài-nguyên app — hồ-sơ trang-trại, ảnh cây…) + đăng-nhập, verifier = backend app. Hai tầng dùng **chung một hình-thức Grant** (§11.2) nhưng khác kênh thực-thi.

### 11.1 Một primitive, hai tầng thực-thi

| Loại quyền | Ví-dụ | Nơi lưu/đọc | Ai ép |
|---|---|---|---|
| **On-chain authority** | mint token, chi quỹ, vận-hành thiết-bị on-chain | DID Authorization Registry (§10, UTxO) | validator (pull, reference-input) |
| **Off-chain data-access** | xem hồ-sơ trang-trại, cập-nhật ảnh cây | Consent record (ký bởi controller, lưu LampNet/backend) | backend app (đọc + verify) |

Data-access KHÔNG cần tx on-chain mỗi lần cấp (rẻ, nhanh); on-chain authority thì validator ép thật.

### 11.2 Schema `Grant`

```
Grant ≜ {
  grantor_did   : DID,            -- người cấp (chủ trang trại)
  grantee_did   : DID,            -- bên nhận (Sở NN / app / thợ)
  action        : ActionTag,      -- "view" | "update_photo" | "operate" | "mint:LAMP" …
  resource      : ResourceRef,    -- phạm-vi: DID (Asset/Farm) hoặc id tài-nguyên app
  valid_from    : SlotNo,
  valid_until   : Option<SlotNo>, -- None = vô-thời-hạn (khuyến-nghị LUÔN đặt hạn)
  nonce         : Bytes32,        -- chống phát-lại
  revocable     : Bool,           -- mặc-định true
  signature     : Bytes64,        -- ký bởi CONTROLLER HIỆN-TẠI của grantor_did
}
```
`resource` cho phép phạm-vi HẸP ("ảnh cây CROP-123" → resource = AssetDID của cây đó, KHÔNG phải cả trang-trại). `action` dùng khoá thuật-ngữ chuẩn (`pk.cap.*`) + cho phép app mở-rộng nhãn riêng.

### 11.3 Luồng xin quyền (Permission Request)

```
App                          PhoenixKey app (thiết-bị user)            App backend
 │ 1. tạo PermissionRequest{grantee=app_did, action, resource, valid_until}
 │ ─────── QR / deep-link ───────►  2. user XEM rõ: "App X xin [view]
 │                                     trên [Trang trại của bạn] tới
 │                                     [30 ngày]". Đồng ý?
 │                                  3. user mở Master_KEK → ký Grant
 │ ◄────── trả Grant (đã ký) ─────  4. (off-chain) cập-nhật Consent record
 │ 5. lưu Grant                                                         │
 │ ──────────────────── gửi Grant kèm request ──────────────────────►  │
 │                                      6. backend resolve grantor_did →
 │                                         controller hiện-tại (§3.6
 │                                         `anchor_controller_ok`) →
 │                                         verify chữ-ký + còn hạn +
 │                                         chưa thu-hồi → CHO PHÉP
```
`PermissionRequest` = Grant chưa-ký (thiếu `signature`); app tạo, user biến thành Grant bằng cách ký. Bước 2 là màn đồng-thuận: hiện đúng ai-làm-gì-trên-gì-tới-bao-giờ, ngôn-ngữ người thường, KHÔNG ký mù.

### 11.4 `verify(grant)` phía app/backend

1. `resolve(grant.grantor_did)` → DID Document → `controller_pkh` HIỆN-TẠI (cùng cơ-chế resolve dùng bởi `anchor_controller_ok`, §3.6 — đọc anchor sống, sống-sót qua Rotate/Recovery).
2. Verify `grant.signature` trên `H(grant trừ signature)` bằng controller key.
3. `valid_from ≤ now ≤ valid_until` (nếu có hạn).
4. Tra revocation (§11.5): grant chưa bị thu-hồi.

Vì verify đọc controller HIỆN-TẠI, một Grant ký trước đó vẫn hợp-lệ sau khi grantor Rotate — nhưng muốn revoke-được-tức-thì phải dùng pull-model §11.5 (đọc trạng-thái sống mỗi lần verify, không cache lâu hơn TTL).

### 11.5 Thu-hồi (Revoke)

- **Off-chain:** controller cập-nhật Consent record (đánh dấu `Revoked` cho nonce/grant đó) trên LampNet/backend; app đọc live khi verify — tức-thì.
- **On-chain:** set entry `Revoked` trong DID Authorization Registry (§10.2, controller ký) — validator/đối-tác đọc reference-input → fail ngay.
- Thu-hồi ≠ giết DID (Deactivate §3.2 đóng-băng mọi thứ; thu-hồi chỉ gỡ một quyền).

### 11.6 Liệt-kê & minh-bạch

- **Quyền RA (outgoing):** "Tôi đã cấp gì cho ai" — dashboard đọc Consent record + registry entry (§10) của `grantor_did`. Phải xem được: `grantee_did`, `action`, `resource`, hạn (`valid_until`), trạng-thái; nút **Thu-hồi** mỗi dòng.
- **Quyền VÀO (incoming):** "Ai cho tôi quyền gì" — cùng dashboard, lọc theo DID mình là `grantee_did`.
- **I-GRANT-LIST:** user PHẢI xem được TOÀN-BỘ quyền đang còn hiệu-lực mình đã cấp, bất-cứ lúc nào (yêu-cầu thực-tế: "ai đang truy-cập cái gì").

### 11.7 Đăng-nhập "Sign in with PhoenixKey"

Trường-hợp riêng của §11.3 với `action="login"`:
```
SignRequest ≜ {
  origin   : String,   -- domain/app id xin đăng-nhập
  challenge: Bytes32,  -- ngẫu-nhiên, chống phát-lại
  nonce    : Bytes32,
  exp      : SlotNo,
}
```
App hiện QR chứa `SignRequest`. User quét → xem `origin` → ký challenge bằng **HW_Key (P-256, sinh-trắc)** → trả `{did, signature, challenge}`. App verify: resolve `did` → `authentication` key (HW_Key, theo `verificationMethod` trong DID Document — xem §4.3) → verify chữ-ký trên challenge → trả session.

**Định-dạng chữ-ký theo đúng khoá `verificationMethod`:** P-256 ECDSA (HW_Key) cho login/assertion; Ed25519 (TAAD_Key/controller) cho tx/Grant — khớp field 3 `hw_key_pubkey` (P-256) và field 2 `controller_pkh` (băm Ed25519 TAAD_Key) của `TAADDatum` (§2.1). Kênh trả: deep-link callback (mobile) hoặc websocket/poll (web).

### 11.8 Thẩm-quyền theo CHỨC-VỤ (role/seat-based authority)

Thẩm-quyền có thể gắn vào một **chức-vụ (seat)** thay vì cá-nhân: người giữ chức nắm thẩm-quyền, rời chức thì người kế-nhiệm nắm, khoá cũ mất quyền.

- Một chức-vụ = một **RoleDID** (DID riêng cho ghế, vd "CFO của Cty X"). `controller` của RoleDID = khoá người ĐANG giữ chức — RoleDID dùng đúng cơ-chế anchor/controller của §1-§3 (không phải kiểu DID mới).
- Thẩm-quyền cấp cho chức = Grant/registry-entry với **`grantee`/`grantee_did` = RoleDID** (không phải pkh cá-nhân). Verifier resolve RoleDID → controller hiện-tại → đó là người được hành-xử quyền.
- **Bàn-giao chức** = `Rotate` (§3.1) `controller` của RoleDID sang người kế-nhiệm. Vì mọi verifier đọc controller HIỆN-TẠI, khoá người cũ **mất quyền ngay** (revoke ngầm qua rotation); người mới thừa-hưởng đủ quyền của chức, không cần thu-hồi từng grant.
- **I-ROLE-2 (đề-xuất):** RoleDID nên thuộc một OrgDID (chức trong tổ-chức, dùng `GenesisChild{owner_did}` §3.0) → bàn-giao tuân governance Org, tránh một người tự-ý đổi người giữ chức.
- **Mở-rộng cần (chưa build):** guardian-as-RoleDID (vd "Công an xã" làm guardian thay vì pkh cá-nhân) đòi `TAADDatum.guardians` (field 6, hiện lưu `List<VerificationKeyHash>` tĩnh) chuyển sang tham-chiếu DID + resolve controller hiện-tại qua reference-input lúc `InitRecovery` — đổi kiểu field 6, cần đặc-tả riêng ở Rebirthme trước khi đụng `taad_logic.ak`.

### 11.9 Ranh-giới (đừng chồng VeData)

PhoenixKey cấp **danh-tính + chữ-ký Grant + resolve khoá**. KHÔNG lưu hồ-sơ trang-trại, KHÔNG định-nghĩa "ảnh cây" là gì — đó là VeData/app. `resource` trỏ tới tài-nguyên do app/VeData định-nghĩa; PhoenixKey chỉ ký rằng "controller của grantor đồng-ý cho grantee action trên resource đó".

### 11.10 Bất-biến

- **I-GRANT-1:** Grant ký bởi CONTROLLER HIỆN-TẠI của `grantor_did` (đọc động → rotatable, cùng cơ-chế §3.6).
- **I-GRANT-2:** màn đồng-thuận hiện đủ ai-gì-trên-gì-tới-bao-giờ, ngôn-ngữ người thường; KHÔNG ký mù.
- **I-GRANT-3:** mặc-định đặt `valid_until` (quyền có hạn); vô-thời-hạn phải user chủ-động chọn.
- **I-GRANT-4:** mọi quyền `revocable=true` phải tra trạng-thái sống trước khi tin; app KHÔNG cache quyết-định cho-phép quá `min(valid_until, TTL ngắn)`.
- **I-GRANT-5:** scope theo `resource` cụ-thể — không ép user cấp rộng hơn mức xin.
- **I-GRANT-LIST:** user PHẢI xem được TOÀN-BỘ quyền (RA + VÀO) đang còn hiệu-lực, bất-cứ lúc nào (§11.6).
- **I-ROLE-1:** thẩm-quyền của chức đọc theo `controller` HIỆN-TẠI của RoleDID; bàn-giao = `Rotate`, không cần thu-hồi từng grant riêng-lẻ.

→ Xem đầy-đủ: `spec-proposals/PhoenixKey-Permission-and-Consent-Spec.md`.

---

## 12. Chuẩn-hoá thuật-ngữ & i18n (VI/EN) — nhãn hiển-thị cho `EntityType`/`TAADStatus` (PROPOSAL, CHỜ DUYỆT)

> **Trạng-thái: PROPOSAL — chờ duyệt.** Nguồn: `spec-proposals/PhoenixKey-Terminology-i18n-Spec.md`. Áp-dụng cho mọi chuỗi hiển-thị người-đọc trong toàn hệ (app, SDK, frontend, resolver) ánh-xạ từ giá-trị canonical on-chain/backend — ví-dụ nhãn 10 `EntityType` (§1.3, field 1 `TAADDatum`) và 4 trạng-thái `TAADStatus` (§2.1 field 5: Active/Recovering/Migrated/Revoked, cộng các trạng-thái vòng-đời mở-rộng ở tầng point-in-time §5.2). Đây là tầng **trình-bày**, KHÔNG đổi giá-trị canonical on-chain (byte enum `types.ak:21-32`, §1.3) hay giá-trị `status` trong response API (§5.1/§5.2 vẫn trả `active`/`revoked`/`recovering`/`migrated` dạng máy-đọc).

### 12.1 Cơ-chế

- **Khoá ổn-định (stable key)** dạng `pk.<miền>.<tên>` — KHÔNG dịch, KHÔNG đổi. Code/log/spec tham-chiếu KHOÁ, không nhúng chuỗi cứng người-đọc.
- **Bảng chuẩn (canonical)** PhoenixKey cấp: mỗi khoá có cả **`vi`** và **`en`** (bắt-buộc cả hai) — `pk.locale.canonical`, phát-hành cùng SDK dạng `pk.vi.json` / `pk.en.json`.
- **Bảng phủ app (overlay):** app cấp `app.<id>.<locale>` để thêm locale mới hoặc đổi nhãn theo thương-hiệu; chồng lên canonical theo từng khoá, khoá không phủ rơi về canonical.
- **Chuỗi tra-cứu (fallback):** `app overlay[locale]` → `canonical[locale]` → `canonical['en']`. Không bao giờ hiện khoá thô cho người dùng.

### 12.2 Namespace liên-quan Anchorme

| Miền | Tiền-tố | Neo dữ-liệu canonical | Ví-dụ |
|---|---|---|---|
| Loại DID | `pk.didtype.*` | `TAADDatum.entity_type` (§2.1 field 1, `types.ak:21-32`) — 10 loại | `pk.didtype.person` → "Cá nhân" / "Person" |
| Trạng-thái vòng-đời | `pk.status.*` | `TAADDatum.status` (§2.1 field 5) + trạng-thái mở-rộng point-in-time (§5.2) | `pk.status.recovering` → "Đang khôi phục" / "Recovering" |
| Khoá & danh-tính | `pk.key.*` | `controller_pkh`/`hw_key_pubkey` (§2.1 field 2-3) | `pk.key.hw_key` → "Khoá phần cứng (sinh trắc)" / "Hardware key (biometric)" |
| Khôi-phục/sao-lưu | `pk.recovery.*` | luồng Rebirthme (§3.5) | `pk.recovery.guardian` |

> Bảng đầy-đủ 10 loại DID + 9 trạng-thái + các namespace khác (`pk.cap.*`, `pk.addr.*`, `pk.screen.*`, `pk.msg.*`) — xem `spec-proposals/PhoenixKey-Terminology-i18n-Spec.md §3`. Chú-ý nhãn `agent`: Math gọi **Agent**, validator hiện đặt biến **AI** — chuẩn hiển-thị là "Tác tử AI / AI Agent", khoá ổn-định `pk.didtype.agent`.

### 12.3 Trách-nhiệm & vị-trí file

- **Bảng chuẩn** PhoenixKey duy-trì `pk.vi.json` / `pk.en.json`, phát-hành cùng SDK, là một phần interface công-khai.
- **Frontend** (đã có react-i18next): import bảng chuẩn làm namespace `pk`.
- **Flutter app / Core** (chưa có i18n): thêm gói intl/l10n, nạp `pk.*.json` + overlay — hạng-mục UI riêng, không thuộc phạm-vi Tech này.
- **App đối-tác:** cấp `app.<id>.<locale>.json`; thiếu khoá khi thêm locale mới → rơi canonical `en` + log cảnh-báo.

### 12.4 Quy-tắc bất-biến [N]

- **I-I18N-1:** Code/spec/log tham-chiếu KHOÁ ổn-định, không chuỗi cứng người-đọc.
- **I-I18N-2:** Mỗi khoá canonical PHẢI có cả `vi` và `en`.
- **I-I18N-3:** Khoá ổn-định không đổi tên sau khi phát-hành (chỉ thêm mới / deprecate, không sửa nghĩa).
- **I-I18N-4:** Overlay app chỉ ĐÈ nhãn, KHÔNG đổi ngữ-nghĩa khoá (vd không được map `pk.cap.mint_token` sang nghĩa khác).
- **I-I18N-5 (ràng-buộc riêng Anchorme):** khoá `pk.didtype.*`/`pk.status.*` ánh-xạ 1-1 vào giá-trị canonical on-chain (§1.3 type-byte, §2.1 field 5) — đổi thứ-tự/thêm giá-trị enum ở `types.ak` PHẢI đồng-bộ thêm khoá tương-ứng, KHÔNG tái-dùng khoá cũ cho nghĩa mới.

→ Xem đầy-đủ: `spec-proposals/PhoenixKey-Terminology-i18n-Spec.md`.

---

## Nguồn

- `PhoenixKey-Validator`: `validators/taad.ak`, `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak`.
- `PhoenixKey-Specs/PhoenixKey-Math.md` §2, §4–§5, §10, §22.
- Nguồn thiết-kế nội-bộ (không công khai).
- `PhoenixKey-Core/Enclave/rust_core`: `phoenix_address.rs:52`, `crypto.rs:339`; `PhoenixKey-Database`: `DidPhoenixGenerator.java`, `ResolverController.java`.
- `spec-proposals/PhoenixKey-Terminology-i18n-Spec.md` (§12).
