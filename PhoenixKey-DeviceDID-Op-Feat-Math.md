# PhoenixKey — DeviceDID Creation + Verification (Feat + Math)

> **PHÂN LOẠI: ONCHAIN (`Op_create_device`) → Tuân · OFFCHAIN (endpoints) → Long.**

**Trạng thái**: DEV-READY SPEC PROPOSAL (chốt datum/redeemer + endpoint; sẵn sàng code)
**Ngày**: 2026-07-02
**Phiên bản**: v1.0
**Nền tảng (KHÔNG phát minh lại)**:
- `PhoenixKey-Specs/PhoenixKey-Math.md` §15 (DeviceDID), §2.1 (DID_construct, type byte), §2.5 (metadata-6789), §5.1 (Op_create_person — mẫu genesis để nhân bản), §13.1 (`OP_CREATE_ORG` challenge — mẫu byte-exact).
- `LampNet-Network-Spec-DRAFT.md` §3.3 (node cần DeviceDID `device_class=LampNetNode`; `hw_cert` là nguồn `AttestationMode::Hardware`; `lampnet-join` đã tiêu thụ cái này).
- Backend convention: prefix `/api/v1`, JSON **snake_case**, bọc `DataResponse<T>{code,message,result}` (theo `PhoenixKey-API-Catalog.md`).

> **Ranh giới cứng (đọc trước):**
> - ONCHAIN — validator/mint op `Op_create_device` → **Tuân** (`PhoenixKey-Validator`).
> - OFFCHAIN — endpoint build/submit + resolve/verify + pubsub → **Long** (`PhoenixKey-Database`).
> - Interface contract (datum schema, canonical challenge string, type byte `0x03`, invariant IDs `DEV-*`) do orchestrator giữ; hai bên KHÔNG tự đổi.

---

## §1. Bối cảnh + consumers

### 1.1 Vấn đề (đây là bottleneck)

Ba nhánh đang **kẹt chờ** DeviceDID on-chain — hiện chưa có op nào mint được nó:

| Consumer | Cần gì từ DeviceDID | Nếu thiếu |
|---|---|---|
| **LampNet node admission** (`lampnet-join`) | node đã admission PHẢI có DeviceDID `device_class=LampNetNode`; `hw_cert` là nguồn `AttestationMode::Hardware` → floor tier Edge (`attestation.rs:118`). `NodeProfile.person_did` bind về chủ. | Không phát được node cert on-chain sau khi committee cấp `MembershipCertificate` (§3.3 [OPEN], `lib.rs:165`). Node không có danh tính neo chuỗi → không thu hồi được, reward không về đúng DID. |
| **Knowme device** | gắn tài liệu/chứng nhận vào Device/Product DID (Knowme §1.2 bảng schema). | Không có Device DID để làm chủ thể của `DocumentClaim`. |
| **Rada** | verify một thiết bị thuộc đúng một PersonDID (binding device ↔ owner). | Không xác thực được thiết bị hợp lệ của người dùng. |

Tất cả **wait on this**. Đây là spec mở khoá cả ba.

### 1.2 DeviceDID làm gì (Math §15)

Neo một danh tính **phần cứng** (không sinh trắc, không người vận hành) lên Cardano:
- Bind `device_pubkey` ↔ `DeviceDID` (khoá thiết bị = danh tính thiết bị).
- Gắn về `owner` (PersonDID hoặc OrgDID đang Active).
- Mang `hw_cert` (chứng thực phần cứng: TPM2 / Secure Enclave / FIDO2) băm lên chuỗi.
- `provisioner` ký phát hành (thực thể cấp thiết bị — PersonDID/OrgDID/ServiceDID).
- Có TTL theo `device_class` (`MAX_CERT_AGE_SLOTS`, §15) → buộc **re-attest** định kỳ.

### 1.3 KHÔNG thuộc phạm vi (ranh giới cứng)

- 🔴 KHÔNG sinh trắc, KHÔNG uniqueness/dedup người. DeviceDID xác thực **phần cứng**, không phải người (Math §15: "Authenticates hardware identity, not human operator").
- KHÔNG re-attestation flow (rotate cert khi hết TTL) — spec riêng, v1.1. Bản này chỉ **create + verify**.
- KHÔNG decommission (§32.5) — spec riêng.
- KHÔNG phát minh crypto: dùng đúng verifier TPM2/SEP/FIDO2 chuẩn (§4).

---

## §2. Kiến trúc (2 phần tách bạch)

```
   Mobile / Node provisioner                        Rada / LampNet / Knowme
            │                                                  │
            │ POST /api/v1/devices/did                         │ GET .../verify?person_did=
            ▼                                                  ▼
 ┌──────────────────────── OFFCHAIN → Long (PhoenixKey-Database) ───────────────────────┐
 │  build Op_create_device tx · verify hw_cert (§4 backend-side) · resolve · verify      │
 │  bind · submit · emit pubsub device_did_bound_to_person                               │
 └───────────────────────────────────────┬──────────────────────────────────────────────┘
                                          │ signed tx (mint anchor NFT + metadata-6789 + datum)
                                          ▼
 ┌──────────────────────── ONCHAIN → Tuân (PhoenixKey-Validator) ───────────────────────┐
 │  Op_create_device: mint policy + DeviceDID datum · invariants DEV-1..DEV-8            │
 │  provisioner sig · owner Active · hw_cert present (Hardware) · device_id_hash unique  │
 └───────────────────────────────────────────────────────────────────────────────────────┘
```

Mẫu neo (anchor) **nhân bản PersonDID genesis (§5.1) + AssetDID first-claim (§17)**: mint một **anchor NFT** (một token duy nhất, `token_name = device_id_hash`) + publish **DID Document metadata-6789** (§2.5) + đặt **DeviceDID datum** tại địa chỉ script. Anchor NFT là canonical binding; datum mang trạng thái cert.

---

## §3. ONCHAIN — `Op_create_device` → Tuân

> **File dự kiến**: `PhoenixKey-Validator/validators/device_did.ak` (validator + mint policy `device_did_mint`).

### §3.1 DeviceDID datum [N]

Nhân bản `DIDBase` (§2.3) + phần mở rộng §15. Tất cả field tên **English**, byte-exact.

```
DeviceDIDDatum ≜ {
  -- DIDBase (§2.3) subset neo trong datum:
  did                : DID,              -- did:phoenix:SLOT:HASH (§2.1), type byte = Device = 0x03
  schema_version     : ℕ,               -- = 1 (v1.0 MVP; §2.3 C-SCHEMA-1)
  type               : DIDType,          -- = Device (0x03), IMMUTABLE — I-DEV-8
  owner              : DID,              -- PersonDID (0x00) | OrgDID (0x01), Active tại create — I-DEV-2
  security_level     : SecurityLevel,    -- = Hardware_Attestation (§2.4, score 5)
  created_slot       : SlotNo,
  seq                : ℕ,               -- = 0 tại genesis (C-SEQ)
  metadata_hash      : {0,1}^256,        -- H(off-chain extended metadata), khớp metadata-6789

  -- §15 device-specific:
  device_pubkey      : Ed25519PubKey,    -- khoá công khai của thiết bị (bind ↔ did) — I-DEV-6
  device_class       : DeviceClass,      -- Mobile|Laptop|Server|VPS|Sensor|Drone|Wearable|Gateway|LampNetNode
  hw_cert_hash       : {0,1}^256,        -- BLAKE2b-256 của hw_cert đầy đủ (§3.4) — I-DEV-3
  hw_cert_kind       : HwCertKind,       -- TPM2_Quote | SEP_Cert | FIDO2_Assertion
  cert_issued_slot   : SlotNo,           -- F-09 slot-based; TTL = + MAX_CERT_AGE_SLOTS[device_class] (§15)
  device_id_hash     : {0,1}^256,        -- fingerprint duy nhất; = anchor NFT token_name — I-DEV-4, I-DEV-6
  provisioner        : DID,              -- thực thể cấp phát; PersonDID|OrgDID|ServiceDID — I-DEV-1
}

DeviceClass ≜ Mobile | Laptop | Server | VPS | Sensor | Drone | Wearable | Gateway | LampNetNode
HwCertKind  ≜ TPM2_Quote | SEP_Cert | FIDO2_Assertion

MAX_CERT_AGE_SLOTS ≜ {  -- Math §15, byte-exact
  Mobile→4320, Laptop→8640, Server→2160, VPS→2160, LampNetNode→8640,
  Drone→8640, Gateway→17280, Wearable→17280, Sensor→43200
}
```

> **Ghi chú anchor NFT.** `device_id_hash` = `token_name` của anchor NFT dưới `device_did_mint` policy. Một `device_id_hash` ⇒ một token ⇒ **uniqueness on-chain** (I-DEV-4) do Cardano native-token đảm bảo — không mint được hai token cùng name dưới cùng policy nếu policy chặn (xem redeemer). Đây là chỗ thay first-claim-wins của AssetDID (§17).

### §3.2 Redeemer [N]

```
DeviceDIDRedeemer ≜ Create {
  provisioner_sig   : Signature,     -- provisioner ký CHALLENGE_STR (§3.3) — I-DEV-1
  owner_pkh         : {0,1}^224,     -- payment key hash của owner, để định vị owner cho reference-input
  nonce             : {0,1}^256,     -- anti-replay, đi vào CHALLENGE_STR
}
```

> v1.0 redeemer chỉ có `Create`. `Rotate`/`Decommission` để dành cho spec riêng (giữ enum mở).

### §3.3 CHALLENGE STRING chuẩn (byte-exact) [N]

**Provisioner ký chuỗi này**, nhân bản mẫu `PHOENIXKEY_GENESIS:` (§5.1) và `OP_CREATE_ORG` (§13.1). NORMATIVE — mọi implementation PHẢI khớp **byte-for-byte**:

```
CHALLENGE_STR ≜
    "PHOENIXKEY_DEVICE_CREATE:"
  ++ hex(device_pubkey)          -- 64 hex chars (32-byte Ed25519 pubkey), lowercase
  ++ ":" ++ owner                 -- DID string của owner (UTF-8, did:phoenix:…)
  ++ ":" ++ hex(hw_cert_hash)     -- 64 hex chars (BLAKE2b-256), lowercase
  ++ ":" ++ device_class_tag      -- tên class dạng ASCII: "Mobile"|"Laptop"|…|"LampNetNode"
  ++ ":" ++ nonce                 -- hex(nonce), 64 hex chars, lowercase

provisioner_sig = Sign(provisioner_key, CHALLENGE_STR)
```

- `hex(·)` = lowercase, no-`0x`-prefix, độ dài cố định (khớp §2.1 encode + §5.1 genesis).
- `device_class_tag` = tên biến thể ASCII đúng như bảng §3.1 (case-sensitive).
- Validator verify: `Verify(key(provisioner, s), CHALLENGE_STR, provisioner_sig)` với `key(provisioner, s)` = controller pubkey của provisioner tại slot `s` (đọc qua reference-input DID document của provisioner).

> **Vì sao chuỗi ràng đủ 5 thành phần**: chống tách-ghép (một chữ ký provisioner không thể tái dùng cho device_pubkey / owner / cert / class khác) và chống replay (nonce). Bám nguyên tắc §5.1: chữ ký tự-chứng-minh phải bao trọn mọi field bind.

### §3.4 Invariants (IDs DEV-*) [N]

Enforce **on-chain** bởi validator + mint policy `Op_create_device`:

| ID | Invariant | Enforce |
|---|---|---|
| **I-DEV-1** | `provisioner` PHẢI ký `CHALLENGE_STR` (§3.3); `Verify(key(provisioner,s), CHALLENGE_STR, provisioner_sig)`. `provisioner` phải Active tại `s`. | mint policy |
| **I-DEV-2** | `owner` PHẢI là DID đang **Active** tại slot `s`, và `type(owner) ∈ {Person, Org}` (§22 CanOwn: Device là sink của Person/Org). Đọc qua reference-input owner TAAD/DID doc. | validator |
| **I-DEV-3** | `hw_cert` PHẢI có mặt cho `security_level = Hardware_Attestation`: `hw_cert_hash ≠ 0` và `hw_cert_kind` khớp `hw_cert_hash` (băm được backend §4 tính; validator băm lại `metadata`-carried cert nếu inline, hoặc tin `hw_cert_hash` + backend gate — xem §4 ranh giới). | mint policy |
| **I-DEV-4** | `device_id_hash` **duy nhất**: mint đúng **1** anchor NFT `token_name = device_id_hash`, `quantity = 1`, dưới `device_did_mint`. Cardano policy chặn double-mint cùng name. | mint policy |
| **I-DEV-5** | `cert_issued_slot ≤ s` và `s < cert_issued_slot + MAX_CERT_AGE_SLOTS[device_class]` (cert chưa hết hạn ngay lúc phát) — Math §15 `HW_Attestation`. | validator (validity range) |
| **I-DEV-6** | Binding `device_pubkey ↔ did`: `did = DID_construct(Device, provisioner, s)` (§2.1); `device_pubkey` và `device_id_hash` **IMMUTABLE** sau create (không op nào bản này đổi). | validator |
| **I-DEV-7** | LampNetNode floor: `device_class = LampNetNode ⇒ hw_cert_kind ∈ {TPM2_Quote, SEP_Cert, FIDO2_Assertion}` (Hardware bắt buộc; không Software). Khớp §3.3 LampNet: Hardware → floor Edge. (Math §15 I-DEV "PCR attest for LampNetNode".) | mint policy |
| **I-DEV-8** | `type` byte = **Device = 0x03** trong `did` (§2.1 encode(type)); `schema_version = 1`; `seq = 0`; `security_level = Hardware_Attestation`. | mint policy |

> **PCR registry (F-21, §15)**: `TRUSTED_PCR_REGISTRY` do OrgDID Security Council quản trị on-chain. v1.0: validator kiểm `hw_cert_hash ≠ 0` + I-DEV-7; **so-khớp giá trị PCR cụ thể** với registry là backend-side (§4) + roadmap on-chain reference-input v1.1. Ghi rõ ở §4 ai check cái gì.

### §3.5 Effect (nhân bản §5.1 Op_create_person)

```
Op_create_device(device_pubkey, owner, device_class, hw_cert_hash, hw_cert_kind,
                 cert_issued_slot, device_id_hash, provisioner, provisioner_sig, nonce, s):
  Requires: I-DEV-1 .. I-DEV-8 (§3.4)
  Effect:
    did ← DID_construct(Device, provisioner, s)                 -- §2.1, type byte 0x03
    mint 1× anchor NFT under device_did_mint, token_name = device_id_hash   -- I-DEV-4
    create DeviceDIDDatum{ did, schema_version=1, type=Device, owner,
                           security_level=Hardware_Attestation, created_slot=s, seq=0,
                           metadata_hash, device_pubkey, device_class, hw_cert_hash,
                           hw_cert_kind, cert_issued_slot, device_id_hash, provisioner }
    publish DIDDocMetadata(did) to metadata label 6789 (§2.5)   -- controller = owner;
                                                                -- verificationMethod = device_pubkey (Ed25519)
  Post: I-DEV-1..8 hold. owner(did) = owner (§22, Device = sink).
```

---

## §4. hw_cert verify — backend vs validator (§4)

Ranh giới rõ **ai kiểm cái gì**. Validator KHÔNG parse được chứng thực nhà sản xuất (quá đắt ExUnit + cần root-CA); backend làm phần nặng, validator neo cam kết.

| Bước | TPM2_Quote | SEP_Cert (Secure Enclave) | FIDO2_Assertion | Ai làm |
|---|---|---|---|---|
| Parse cert / attestation object | parse TPMS_ATTEST + TPMT_SIGNATURE | parse Apple App Attest / DCAppAttest CBOR | parse WebAuthn `attestationObject` + `authenticatorData` | **Backend** (Long) |
| Verify chữ ký nhà SX / AK | verify AK sig; AK cert chain → TPM manufacturer root (Infineon/STM/…) | verify Apple root cert chain (`Apple App Attestation Root CA`) | verify attestation statement (packed/tpm/apple-anonymous) → FIDO MDS root | **Backend** |
| Kiểm PCR / nonce binding | `TPMS_ATTEST.extraData == H(CHALLENGE_STR)`; PCR ∈ `TRUSTED_PCR_REGISTRY` (F-21) | nonce trong `clientDataHash` khớp challenge; RP-ID hợp lệ | `authenticatorData.rpIdHash` + `challenge` khớp | **Backend** |
| Freshness | cert < TTL (`cert_issued_slot`) | như trên | counter monotonic (chống clone) | **Backend** |
| **Băm cam kết** → `hw_cert_hash = BLAKE2b-256(hw_cert_canonical_bytes)` | ✓ | ✓ | ✓ | **Backend** tính, **Validator** neo |
| `hw_cert_hash ≠ 0`, `hw_cert_kind` hợp lệ, LampNetNode ⇒ Hardware | I-DEV-3, I-DEV-7 | — | — | **Validator** |
| TTL slot-based `s < cert_issued_slot + MAX_CERT_AGE_SLOTS` | I-DEV-5 | — | — | **Validator** |

**Chốt ranh giới (đọc kỹ):**
- **Validator (Tuân) tin `hw_cert_hash` là đại diện cert hợp lệ** — chỉ enforce cấu trúc (I-DEV-3/5/7). Nó KHÔNG verify chuỗi cert nhà SX.
- **Backend (Long) là gatekeeper cert thật**: build tx CHỈ khi cert verify pass đủ 4 bước đầu. `hw_cert_canonical_bytes` (byte chuẩn để băm) = CBOR canonical của attestation object đầy đủ; backend lưu off-chain (metadata-6789 `hw_cert` reference hoặc DB), băm ra `hw_cert_hash`.
- v1.1 roadmap: đưa `TRUSTED_PCR_REGISTRY` thành reference-input để validator so PCR on-chain (giảm tin backend).

> **Canonical hw_cert bytes**: TPM2 = `TPMS_ATTEST ‖ signature ‖ ak_cert_der`; SEP = App-Attest CBOR; FIDO2 = `attestationObject` CBOR. Băm BLAKE2b-256 → 32 byte → `hw_cert_hash`. Tài liệu byte-exact cho từng kind ở appendix backend (Long viết theo lib chuẩn: `tpm2-tss`, Apple DeviceCheck, `webauthn` verifier).

---

## §5. OFFCHAIN — endpoints → Long

> Prefix `/api/v1`, JSON **snake_case**, bọc `DataResponse<T>{code,message,result}`. File dự kiến: `PhoenixKey-Database` controller `DeviceDIDController`.

### §5.1 POST `/api/v1/devices/did` — build + submit `Op_create_device`

Build tx `Op_create_device`, verify hw_cert (§4), submit, trả DeviceDID + txHash.

**Request**
```json
{
  "owner_did":             "did:phoenix:...:...",
  "device_class":          "LampNetNode",
  "hw_cert":               { "kind": "TPM2_Quote", "attestation_b64": "..." },
  "device_pubkey":         "hex(32-byte Ed25519)",
  "provisioner_signature": "hex(64-byte)  // ký CHALLENGE_STR §3.3",
  "nonce":                 "hex(32-byte)"
}
```
`provisioner_did` suy ra từ auth context (session token của provisioner). `device_id_hash` backend tính = `BLAKE2b-256(device_pubkey ‖ hw_cert_hash)` (deterministic, cho phép resolve lại).

**Server flow**
1. Verify `hw_cert` đủ 4 bước §4 → tính `hw_cert_hash`, `hw_cert_kind`. Fail → `4001 hw_cert_invalid`.
2. Dựng `CHALLENGE_STR` (§3.3) từ `device_pubkey ‖ owner_did ‖ hw_cert_hash ‖ device_class ‖ nonce`; verify `provisioner_signature`. Fail → `4002 provisioner_sig_invalid`.
3. Kiểm `owner_did` Active + `type ∈ {Person, Org}` (`did_active` §resolve). Fail → `4003 owner_not_active`.
4. Kiểm `device_id_hash` chưa tồn tại (query indexer). Trùng → `4004 device_already_exists`.
5. LampNetNode ⇒ ép Hardware (I-DEV-7). Fail → `4005 lampnetnode_requires_hardware`.
6. Build tx (mint anchor NFT + datum + metadata-6789), fee-wallet ký phần build, submit qua node.
7. Emit pubsub `device_did_bound_to_person` (§5.4).

**Response**
```json
{ "code": 0, "message": "ok",
  "result": { "device_did": "did:phoenix:...:...", "tx_hash": "hex(32-byte)" } }
```

### §5.2 GET `/api/v1/device/{device_did}` — resolve

Trả cert + binding + owner + status.

**Response `result`**
```json
{
  "device_did":       "did:phoenix:...",
  "owner_did":        "did:phoenix:...",
  "device_class":     "LampNetNode",
  "device_pubkey":    "hex",
  "device_id_hash":   "hex",
  "hw_cert_kind":     "TPM2_Quote",
  "hw_cert_hash":     "hex",
  "cert_issued_slot": 145000000,
  "cert_expiry_slot": 145008640,      // cert_issued_slot + MAX_CERT_AGE_SLOTS[class]
  "provisioner_did":  "did:phoenix:...",
  "status":           "ACTIVE",       // ACTIVE | CERT_EXPIRED | REVOKED
  "anchor_nft":       { "policy_id": "hex", "asset_name": "hex(device_id_hash)" },
  "schema_version":   1
}
```
`status = CERT_EXPIRED` khi `now_slot ≥ cert_expiry_slot` (không cần tx — tính từ slot). Không tìm thấy → `4040 device_not_found`.

### §5.3 GET `/api/v1/device/{device_did}/verify?person_did=` — verify binding

Cho **Rada / LampNet**: xác thực thiết bị thuộc đúng lineage của một PersonDID.

**Query**: `person_did` (bắt buộc). Tuỳ chọn `at_slot` (verify tại slot cụ thể; mặc định now).

**Logic**
```
1. resolve device_did → owner_did, status, cert_expiry_slot
2. binding_ok  = (owner_did == person_did)
                 OR (person_did ∈ Ancestors(owner_did))   // owner là OrgDID mà person là root/member
3. cert_valid  = status == ACTIVE AND at_slot < cert_expiry_slot
4. owner_active = did_active(owner_did, at_slot) AND did_active(person_did, at_slot)
5. verified    = binding_ok AND cert_valid AND owner_active
```
> Lineage `Ancestors(·)` bám §22.3 owner-graph. Với LampNet: `person_did` = `NodeProfile.person_did`; verified ⇒ node hợp lệ để cấp quyền ký.

**Response `result`**
```json
{
  "verified":        true,
  "binding_ok":      true,
  "cert_valid":      true,
  "owner_active":    true,
  "owner_did":       "did:phoenix:...",
  "device_class":    "LampNetNode",
  "checked_at_slot": 145000123,
  "reason":          null            // nếu false: "binding_mismatch"|"cert_expired"|"owner_inactive"|"revoked"
}
```

### §5.4 Pubsub `device_did_bound_to_person` (§5.4)

Emit sau khi tx `Op_create_device` confirm (đã reserve trong `PhoenixKey-API-Catalog.md`: consumer Score/Stamp/Query + LampNet).

```json
{ "event": "device_did_bound_to_person",
  "device_did":   "did:phoenix:...",
  "owner_did":    "did:phoenix:...",   // PersonDID hoặc OrgDID
  "person_did":   "did:phoenix:...",   // root PersonDID của lineage (nếu owner là Org → root person; nếu None → owner_did)
  "device_class": "LampNetNode",
  "tx_hash":      "hex",
  "slot":         145000000 }
```
Emit **sau confirm on-chain** (không optimistic), để consumer tin binding đã neo chuỗi.

---

## §6. Test plan (evidence thật, preview)

Mục tiêu: mint một **LampNetNode** DeviceDID trên preview, `lampnet-join` tiêu thụ nó, verify binding. Không tuyên bố PASS nếu chưa có output thật (theo nguyên tắc verify-behavior).

### §6.1 Onchain (Tuân — validator/policy)

| # | Test | Kỳ vọng |
|---|---|---|
| T-ON-1 | Mint DeviceDID `device_class=LampNetNode`, `hw_cert_kind=TPM2_Quote`, provisioner ký `CHALLENGE_STR` hợp lệ. | tx accept; 1 anchor NFT `token_name=device_id_hash`; datum đúng field. |
| T-ON-2 | provisioner_sig sai (ký nonce khác). | **reject** I-DEV-1. |
| T-ON-3 | owner là DID đã Revoked. | **reject** I-DEV-2. |
| T-ON-4 | `hw_cert_hash = 0`. | **reject** I-DEV-3. |
| T-ON-5 | Mint lần 2 cùng `device_id_hash`. | **reject** I-DEV-4 (double-mint). |
| T-ON-6 | `s ≥ cert_issued_slot + MAX_CERT_AGE_SLOTS[LampNetNode]` (8640). | **reject** I-DEV-5. |
| T-ON-7 | `device_class=LampNetNode` nhưng `hw_cert_kind` = Software (giả). | **reject** I-DEV-7. |
| T-ON-8 | type byte ≠ 0x03 trong did. | **reject** I-DEV-8. |

Evidence: `aiken check` pass + tx hash preview + `cardano-cli`/Blockfrost query datum + asset.

### §6.2 Offchain (Long — endpoints)

| # | Test | Kỳ vọng |
|---|---|---|
| T-OFF-1 | POST `/devices/did` với TPM2 quote thật (hoặc test-vector) → mint. | `200`, trả `device_did` + `tx_hash`; tx confirm trên preview. |
| T-OFF-2 | GET `/device/{device_did}` sau confirm. | resolve đủ field; `status=ACTIVE`. |
| T-OFF-3 | POST với hw_cert giả chữ ký. | `4001 hw_cert_invalid`, KHÔNG build tx. |
| T-OFF-4 | Chờ qua `cert_expiry_slot`, GET resolve. | `status=CERT_EXPIRED` (không cần tx). |
| T-OFF-5 | Pubsub `device_did_bound_to_person` phát sau confirm. | broker nhận event đúng payload. |

### §6.3 Integration end-to-end (bottleneck test)

| # | Test | Kỳ vọng |
|---|---|---|
| T-E2E-1 | `lampnet-join` chạy admission → cấp `MembershipCertificate` → gọi POST `/devices/did` phát DeviceDID `LampNetNode`, bind `NodeProfile.person_did`. | DeviceDID on-chain; `NodeProfile.person_did` = owner. |
| T-E2E-2 | GET `/device/{device_did}/verify?person_did=<node.person_did>`. | `verified=true`, `binding_ok=true`. |
| T-E2E-3 | Rada verify một PersonDID sở hữu device qua cùng endpoint. | `verified=true`. |
| T-E2E-4 | Owner PersonDID sai (không phải chủ node). | `verified=false`, `reason="binding_mismatch"`. |

**Chỉ tuyên bố PASS khi**: T-ON-* (aiken + tx preview), T-OFF-* (curl output thật sau deploy), T-E2E-1..4 (join consume + verify) đều xanh với evidence output.

---

## §7. Ranh giới (chốt cuối)

| Phần | Chủ | Repo | Deliverable |
|---|---|---|---|
| **ONCHAIN** `Op_create_device` (datum §3.1, redeemer §3.2, CHALLENGE_STR §3.3 byte-exact, invariant DEV-1..8 §3.4, anchor-NFT policy) | **Tuân** | `PhoenixKey-Validator` | `device_did.ak` validator + `device_did_mint` policy; aiken check pass; test T-ON-1..8. |
| **OFFCHAIN** endpoints §5.1–§5.4 (build/submit, resolve, verify, pubsub) + hw_cert verify §4 (4 bước nặng) | **Long** | `PhoenixKey-Database` | `DeviceDIDController`; verify TPM2/SEP/FIDO2; test T-OFF-1..5. |
| Interface contract (datum schema, CHALLENGE_STR, type 0x03, DEV-* IDs) | Orchestrator | spec này | không đổi đơn phương. |

**Chốt**: Bản này DEV-READY. Tuân bắt đầu validator từ §3 (mirror §5.1 anchor + §17 first-claim); Long bắt đầu endpoint từ §5 + verifier §4. Cả `lampnet-join`, Knowme device, Rada mở khoá sau khi T-E2E-1..4 xanh.
