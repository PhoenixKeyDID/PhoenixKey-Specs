# PhoenixKey — Wakeme · Đặc-tả KỸ-THUẬT (BẢN A — 2-pha + forfeit)

> **Module:** Wakeme (kích-hoạt nhận LAMP). **Loại doc:** Kỹ-thuật (kiến-trúc · datum/redeemer CBOR · luồng tx · API · ranh-giới · deploy). **Cập nhật:** 2026-07-31.
> **Đối-tượng đọc:** KỸ SƯ triển khai (đội on-chain, backend, Core/Enclave, MAGIC/CARP/Registry-team).
>
> **Doc này KHÔNG chứa công-thức toán** (SpecStandard: toán/bất-biến thuộc [Math](./PhoenixKey-Wakeme-Math.md)). Ở đây chỉ **cấu-trúc dữ-liệu, CBOR, shape tx, API, ranh-giới**. Công-thức `WakemeUsageRight`/`d_unit`/`q`/đồng-hồ → xem Math §1-2. Thiết-kế/động-cơ → [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md); điều-hành → [Exec](./PhoenixKey-Wakeme-Exec.md).
>
> **Thuật-ngữ (anh Aladin chốt 2026-07-31):** LAMP khoá-điều-kiện = **quyền dùng (usage right)**; lượng cấp genesis = **`WakemeUsageRight`** (= `1001·d_unit` oildrop); phần đã kiếm = **`owned`** (sở-hữu-hẳn). "activation"→"wakeme" toàn nhánh (validator/endpoint/type).
>
> **Nguồn đối-chiếu (code = nguồn chân-lý; văn ≠ code → code thắng):**
> - `PhoenixKey-Validator/validators/wakeme_vault.ak` — thin validator (mint-gate 2 + spend dispatch 4 redeemer). Tên khai-báo đã đổi: `validator wakeme_vault(...)` (`:64`).
> - `PhoenixKey-Validator/lib/phoenixkey/wakeme_logic.ak` — datum/redeemer type + `*_ok` (đo 2026-08-19: 94 bài test; cùng `wakeme_vault.ak` 18 bài = 112 bài, 0 lỗi).
> - `PhoenixKey-Validator/lib/phoenixkey/auth_logic.ak` — `anchor_controller_ok` (owner-sig = 2-of-2 controller+device).
>
> → Trạng-thái build/test/deploy: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 1. Kiến-trúc

### 1.1 Sơ-đồ thành-phần

```
┌───────────── CORE / ENCLAVE (Flutter + rust_core) ──────────────────────────┐
│  vân tay → Secure Enclave → khoá controller DID → ví Phoenix (did_payment)   │
│  ký: Wakeme-genesis (owner-witness) · Redeem (owner-witness qua anchor)       │
└──────────────┬───────────────────────────────────────────────────────────────┘
               │ build-unsigned → witness → submit
               ▼
┌───────────── BACKEND (PhoenixKey-Database, Java) ──────────────────────────────┐
│  WakemeController  /api/v1/wakeme/*                                            │
│  · genesis orchestration: đọc pot → tính WakemeUsageRight → build tx genesis   │
│  · Job NGÀY (PHA-1): scan vault n≤1001, idle → Reclaim tx (keeper-sig)         │
│  · Job EPOCH (PHA-2): active → OwnEpoch (keeper); gap≥1001ep → ReclaimEpoch     │
│  · Redeem build/submit (owner) ; GetMAGIC (fiat→CARP)                          │
│  keeper wallet (system-authority MVP) ký Reclaim / OwnEpoch / ReclaimEpoch      │
└───┬────────────────────────────────────────────────────────┬───────────────────┘
    │ submit tx                                                │ reference-input (ĐỌC)
    ▼                                                          ▼
┌── CARDANO (Plutus V3) ──────────────────────────────────────────────────────────┐
│  validator wakeme_vault (apply-param per-DID)                                    │
│    mint-gate: GenesisVault | CloseVault  (own_policy ≡ script-hash = vault-NFT)   │
│    spend:     Reclaim | OwnEpoch | ReclaimEpoch | Redeem   (KHÔNG còn GenDrip)    │
│  VAULT UTxO = Script(own_hash) + [vault-NFT(own,owner_commit)=1] + minADA + LAMP  │
│                  ▲ ref-input                       ▲ ref-input                     │
│  ┌── taad anchor ┘ (owner-sig: Redeem/ReclaimEpoch-owner)  ┌── pot (dist_treasury)┘│
└──────────────────────────────────────────────────────────────────────────────────┘
        ▲ ĐỌC-số-dư (reference-input WakemeVaultDatum) → drip MAGIC (KHÔNG spend LAMP)
┌── MAGIC/CARP engine (repo MAGIC — NGOÀI validator này) ────────────────────────────┐
│  Instant/Schedule/Prepaid Gen ĐỌC (conditional_lamp + owned_lamp) → sinh MAGIC     │
│  MAGIC = account-trong-Vault (nanogic = MAGIC×10⁹), KHÔNG mint token/KHÔNG policy   │
│  [Wakeme validator KHÔNG có redeemer Gen — Gen read-only qua reference-input]       │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Bất-biến kiến-trúc (load-bearing)

- **1 vault = 1 UTxO** tại `Script(own_hash)`, mang **vault-NFT singleton** `(policy = own_hash, name = owner_commit = did_commit)`, min-ADA, và `(conditional_lamp + owned_lamp)` LAMP token khoá. Neo: `wakeme_logic.ak` `has_vault_nft:237`, `find_vault_output:243`.
- **Sổ ↔ value đồng-bộ:** `lamp_token_in_vault == conditional_lamp + owned_lamp` — mọi redeemer ép (`vault_lamp_total:232`). (Toán: Math §3 SỔ-VALUE.)
- **own_policy ≡ script-hash** (multi-purpose, mẫu `taad`): vault-NFT policy CHÍNH là hash validator → validator PHẢI có `mint` handler. Ép ở spend: `expect Script(own_policy) = own_input.output.address.payment_credential` (`wakeme_vault.ak:95`).
- **conditional_lamp CHỈ GIẢM** (Reclaim→pot | OwnEpoch→owned | ReclaimEpoch→pot). **owned_lamp:** tăng (OwnEpoch) / giảm (Redeem→ví owner). KHÔNG đường `conditional → user` trực-tiếp. LAMP→user DUY-NHẤT qua **Redeem** (phần `owned`).
- **LAMP 36B không-burn:** mọi LAMP rời vault về {pot | ví-owner}. **Gen sinh MAGIC KHÔNG spend/đốt LAMP** (đọc reference-input) — Wakeme validator không có redeemer Gen (GenDrip **đã gỡ** 2026-07-31).

### 1.3 Đồng-hồ + hằng-số (chi-tiết công-thức: Math §1-2)

| Thang | Hằng (`wakeme_logic.ak`) | Dùng cho |
|---|---|---|
| NGÀY | `slots_per_day` (86_400) | ranh-giới-pha, anti-idle tick, monotonic `last_tick_day` |
| EPOCH | `slots_per_epoch` (432_000 = 5 ngày) | gate epoch-active PHA-2, đo gap forfeit |

Hằng khác: `oil_per_lamp=10⁶` (`:104`), `epoch_nights=5` (`:107`), `grace_days=7` (`:110`), `phase1_last=1001` (`:113`, ranh-giới PHA-1/PHA-2), `wakeme_nights=1001` (`:116`), `wakeme_use_right_cap=1001·10⁶` (`:119`, trần usage-right = 1001 LAMP), `forfeit_epoch_gap=1001` (`:122`). **Đơn-vị on-chain = oildrop** (`1 LAMP = 10⁶ oildrop`).

---

## 2. Datum / Redeemer — khuôn CBOR (khớp aiken ↔ rust_core)

> Plutus V3 dùng `Constr`. **NGUỒN CHUẨN = code `.ak`**; rebuild `plutus.json` bằng `aiken build` (script-hash `3f6e5bf6…f23`).

### 2.1 `WakemeVaultDatum` — 9 field (thứ-tự CBOR CỐ-ĐỊNH)

Neo: `wakeme_logic.ak:126-144`. `Constr 0 [...]` — thứ-tự field = thứ-tự CBOR (rust_core PHẢI encode đúng thứ-tự):

| # | Field | Aiken | CBOR | Ý-nghĩa | Genesis |
|---|---|---|---|---|---|
| 0 | `owner_commit` | `ByteArray` | bytes | commit DID + **vault-NFT name** (I-ACT-10) | `blake2b_256(did)` (32B) |
| 1 | `did_commit` | `ByteArray` | bytes | con-trỏ DID; MVP `== owner_commit`, `≠ #""` | `blake2b_256(did)` |
| 2 | `vest_start_slot` | `Int` | int | mốc-0 đồng-hồ; genesis ép `== tx_lo` | slot lúc submit |
| 3 | `conditional_lamp` | `Int` | int | quyền-dùng khoá (oildrop) | `WakemeUsageRight` = `1001·d_unit` |
| 4 | `reclaimed_to_pot` | `Int` | int | luỹ-kế về pot (audit, chỉ tăng) | `0` |
| 5 | `last_tick_day` | `Int` | int | NGÀY tick gần nhất (monotonic) | `0` |
| 6 | `last_tick_epoch` | `Int` | int | epoch cuối CÓ OwnEpoch (proven-active); **genesis = −1** (sentinel) | `-1` |
| 7 | `owned_lamp` | `Int` | int | LAMP SỞ-HỮU-HẲN, ở-lại vault (oildrop) | `0` |
| 8 | `d_unit` | `Int` | int | nhịp đêm (oildrop), CỐ-ĐỊNH per-vault | `WakemeUsageRight/1001` |

**⚠ Đổi so với bản trước (rust_core/backend PHẢI cập-nhật):** bản v4.1 có `vested_unlocked` (field 4) + `idle_epochs_p2` (field 7). BẢN A **thay** bằng `owned_lamp` (field 7) + `d_unit` (field 8), field 6 = `last_tick_epoch` (sentinel −1 genesis). Encode 7-field/v4.1 sẽ **decode FAIL** (Aiken khớp arity tuyệt-đối).

**Diễn-CBOR (Genesis, d_unit=10⁶ ⟹ WakemeUsageRight=1001·10⁶ oildrop):**
```
d8799f                          # Constr 0 (tag 121), 9 field
  581f <owner_commit 32B>       # bytes
  581f <did_commit 32B>         # bytes
  1a<vest_start_slot>           # int (= tx_lo)
  1b000000e8d4a51000            # int 1001000000 (conditional_lamp)
  00                            # int 0 (reclaimed_to_pot)
  00                            # int 0 (last_tick_day)
  20                            # int -1 (last_tick_epoch, sentinel genesis)
  00                            # int 0 (owned_lamp)
  1a000f4240                    # int 1000000 (d_unit)
ff
```
(Byte-len minh-hoạ; encoder rust_core tự tính. Load-bearing = **9 field, đúng thứ-tự, `last_tick_epoch=−1`, `owned=0`** để pass `genesis_vault_ok:776`.)

### 2.2 `WakemeRedeemer` (spend) — 4 nhánh

Neo: `wakeme_logic.ak:150-161`. Không field (`Constr idx []`):

| idx | Redeemer | CBOR | Ai ký |
|---|---|---|---|
| 0 | `Reclaim` | `d87980` | keeper (anti-idle PHA-1) |
| 1 | `OwnEpoch` | `d87a80` | keeper (chuyển sở-hữu PHA-2 active) |
| 2 | `ReclaimEpoch` | `d87b80` | keeper **HOẶC** owner (forfeit PHA-2, escape-hatch) |
| 3 | `Redeem` | `d87c80` | owner (rút owned → ví; tuỳ-chọn) |

**⚠ CBOR index DỊCH so với bản trước.** v4.1: `GenDrip=0, Reclaim=1, VestToOwner=2, ClaimVested=3, ForfeitPhase2=4`. BẢN A gỡ GenDrip + đổi tên ⟹ index mới như bảng. **rust_core encoder + backend PHẢI đổi bảng index** — kênh lỗi KHÔNG phải build-fail mà là "submit thành-công chạy NHẦM nhánh có hậu-quả tiền" nếu consumer bám index cũ. (MAGIC KHÔNG ảnh-hưởng: Gen chỉ decode datum reference-input, không dựng redeemer spend — xác-nhận `gen_magic.ak`.)

### 2.3 `VaultMintRedeemer` (mint-gate) — 2 nhánh

Neo: `wakeme_logic.ak:164-168`:

| idx | Redeemer | CBOR | Dùng |
|---|---|---|---|
| 0 | `GenesisVault` | `d87980` | Wakeme genesis — đúc 1 vault-NFT + ép khuôn output |
| 1 | `CloseVault` | `d87a80` | nhánh *-close rút-hết-cuối — pure-burn NFT |

### 2.4 Tham-số apply-param (per-DID, đóng lúc genesis)

Neo: `wakeme_vault.ak:64-71` (khối tham-số validator). Backend apply **7 tham-số** → script-hash + address RIÊNG mỗi DID (mẫu did_payment per-DID):

| # | Param | Type | Nguồn | Ghi-chú |
|---|---|---|---|---|
| 1 | `anchor_nft_policy` | `PolicyId` | hằng TOÀN-HỆ (`taad` Design-2) | đọc controller ký Redeem/ReclaimEpoch-owner |
| 2 | `anchor_nft_name` | `AssetName` | `blake2b_256(did)` | per-DID anchor |
| 3 | `lamp_policy` | `PolicyId` | LAMP canonical | LAMP khoá trong vault |
| 4 | `lamp_name` | `AssetName` | asset-name LAMP | mainnet `LAMP` vs testnet `tLAMP` (policy khác) |
| 5 | `pot_address` | `Address` | pot User (`dist_treasury`) | Reclaim/ReclaimEpoch BUỘC đích về đây (tagged) |
| 6 | `owner_address` | `Address` | **ví Phoenix của DID** (`did_payment`) | Redeem BUỘC owned về đây; **DID-bound, rotation-safe** |
| 7 | `keeper_pkh` | `ByteArray` | system-authority (MVP) | ký Reclaim/OwnEpoch/ReclaimEpoch; **sau thay bằng consume-event Registry** |

`pot_address`/`owner_address` PHẢI là `Script(_)`/địa-chỉ mạng đúng — ràng-buộc network ghi rõ khi apply.

---

## 3. Từng redeemer — shape tx + ai-ký (điều-kiện hình-thức: Math §5)

Ký-hiệu: `d_in`/`d_out` datum vào/ra; `lamp_in`/`lamp_out` LAMP oildrop vault vào/ra. **Đích LAMP gắn-tag** `payout_tag(own_policy, owner_commit)` (InlineDatum) chống double-satisfaction cross-instance (`lamp_to_addr_tagged:306`).

### 3.0 Mint-gate

**`GenesisVault` (Wakeme genesis)** `genesis_vault_ok:776` — đúc ĐÚNG 1 vault-NFT(name=owner_commit) + ép khuôn BẢN A (`s₀==tx_lo`, `conditional=1001·d_unit`, `owned=0`, `last_tick_epoch=−1`, `did_commit==owner_commit`) + `lamp_locked==conditional` + **`anchor_controller_ok`** (owner ký genesis = 2-of-2 controller+device). Ký: owner-witness.
```
in:  [pot UTxO (WakemeUsageRight LAMP)] + [ví Phoenix (fee)]
mint: +1 vault-NFT(own_policy, owner_commit)   redeemer=GenesisVault
out: [VAULT: Script(own) + NFT + minADA + conditional LAMP + inline datum(9-field)]
     [pot recreate]  [change]
signer: owner (controller+device)   validity: [lo, lo+ttl]  (vest_start = lo)
```

**`CloseVault`** `close_vault_ok:840` — pure-burn (mọi movement own_policy < 0). Nối nhánh *-close.

### 3.1 `Reclaim` — anti-idle PHA-1 (`reclaim_ok:388` / `reclaim_close_ok:447`)
Thu ĐÚNG `d_unit` (nhịp đêm) → pot khi NGÀY idle. `conditional−d_unit`, `owned` giữ, `last_tick_day=n`. Nếu `conditional` chạm 0 (còn đúng `d_unit`, `owned=0`) → nhánh ĐÓNG: toàn-bộ → pot + min-ADA → owner + burn. Ký: **keeper**.
```
in:  [VAULT]   redeemer=Reclaim
out: [VAULT recreate: conditional−d_unit, last_tick_day=n]  [pot(tag): +d_unit LAMP]  [change]
signer: keeper   validity.lo hữu-hạn (bắt buộc)
```

### 3.2 `OwnEpoch` — chuyển sở-hữu PHA-2 active (`own_epoch_ok:499`)
Epoch active → `q = min(5·d_unit, conditional)` chuyển `conditional→owned`; `last_tick_epoch=e_now`. **LAMP KHÔNG rời vault** (value bất-biến — chỉ đổi sổ nội-bộ). LUÔN recreate. Ký: **keeper** (attest active — MVP).
```
in:  [VAULT]   redeemer=OwnEpoch
out: [VAULT recreate: conditional−q, owned+q, last_tick_epoch=e_now]
signer: keeper   validity.lo hữu-hạn   (KHÔNG output LAMP rời)
```

### 3.3 `ReclaimEpoch` — forfeit PHA-2 (`reclaim_epoch_ok:546` / `_close_ok:605`)
Idle gap `e − last_tick_epoch ≥ 1001` epoch → toàn-bộ `conditional → pot`; **`owned` KHÔNG đụng**. Recreate nếu `owned≥1`; `owned=0` → ĐÓNG (min-ADA→owner + burn). Ký: **keeper HOẶC owner** (escape-hatch — owner tự đóng khi keeper vắng; đọc anchor để verify owner).
```
in:  [VAULT]   redeemer=ReclaimEpoch
out: [VAULT recreate: conditional=0, owned giữ]  [pot(tag): +conditional LAMP]
     (ĐÓNG nếu owned=0: [pot: +conditional] [owner: min-ADA] + mint −1 NFT)
signer: keeper | owner(anchor)   validity.lo hữu-hạn
```

### 3.4 `Redeem` — owner rút owned → ví (TUỲ-CHỌN) (`redeem_ok:655` / `_close_ok:703`)
`owned − k` (`1 ≤ k ≤ owned`) → ví owner; `conditional` **BẤT-BIẾN**. **KHÔNG phase-gate** (owned là tài-sản owner, rút bất-cứ-lúc-nào). `conditional=0 ∧ rút hết owned` → ĐÓNG + burn. Ký: **OWNER** (`anchor_controller_ok`, KHÔNG keeper).
```
ref-input: [taad anchor NFT(anchor_nft_policy, blake2b_256(did))]   # đọc controller
in:  [VAULT]   redeemer=Redeem
out: [VAULT recreate: owned−k]  [owner_address(tag): +k LAMP]  [change]
     (ĐÓNG nếu conditional=0 ∧ k=owned: [owner: +owned + min-ADA] + mint −1 NFT)
signer: controller+device (Enclave)
```

### 3.5 Bảng ai-ký + đích LAMP

| Redeemer | Ký | LAMP di-chuyển | Đích ép | Pha |
|---|---|---|---|---|
| GenesisVault | owner (2-of-2) | pot → vault (usage-right) | vault (khuôn) | tạo |
| Reclaim | **keeper** | vault → pot (d_unit) | `pot_address` (tag) | PHA-1 (n≤1001) |
| OwnEpoch | **keeper** | KHÔNG (đổi sổ) | — | PHA-2 (n>1001) active |
| ReclaimEpoch | **keeper ∨ owner** | vault → pot (conditional) | `pot_address` (tag) | PHA-2, gap≥1001ep |
| Redeem | **owner** (anchor) | vault → owner (k) | `owner_address` (tag) | mọi lúc (owned≥1) |
| CloseVault | (kèm spend) | burn NFT | — | đóng |

**Gen (MAGIC) KHÔNG là redeemer.** Engine Gen (MAGIC-team) đọc `WakemeVaultDatum` qua **reference-input** (read-only, không spend UTxO) → sinh MAGIC. Kiến-trúc scale (accumulator off-chain + settlement Merkle-anchor mỗi epoch) thuộc **spec MAGIC**, KHÔNG ở đây. Wakeme chỉ đảm bảo `(conditional+owned)` đọc được ổn-định.

---

## 4. Luồng end-to-end

### 4.1 Genesis (tạo vault)
```
Core: keygen vân tay → ví Phoenix
1a POST /wakeme/build:
   backend đọc pot_balance → tính WakemeUsageRight + d_unit (công-thức Math §1)
   → apply-param 7 tham-số per-DID → sinh vault script-hash + address
   → build unsigned tx: spend pot → mint GenesisVault → VAULT(datum 9-field)
   → trả {unsigned_tx_cbor, required_signer_key_hash, vault_address, usage_right_lamp, d_unit_oildrop, vest_start_slot}
Core: Enclave witness (controller+device = 2-of-2)
1b POST /wakeme/submit {signed_tx_cbor} → {cardano_tx_hash, SUBMITTED}
Kết-quả: VAULT conditional=WakemeUsageRight, owned=0, last_tick_epoch=−1, vest_start=lo.
```

### 4.2 Gen (MAGIC yield — nền, NGOÀI Wakeme API)
```
engine MAGIC đọc WakemeVaultDatum(conditional+owned) qua reference-input → sinh MAGIC
KHÔNG spend LAMP, KHÔNG mint token (MAGIC = account-trong-Vault).
[CẤM nối Wakeme vào InstantGen/ScheduleGen tới khi MAGIC pha-2 read-only xong — nhánh chuyển-LAMP cũ sẽ rút LAMP thật]
```

### 4.3 Anti-idle PHA-1 (job NGÀY — backend)
```
Mỗi vault n≤1001: active = tiêu MAGIC ≥ MIN (qua dịch-vụ Registry) [BLOCKER Registry]
  nếu !active ∧ n≥7 ∧ n>last_tick_day ∧ conditional≥d_unit:
    build Reclaim tx (keeper-sig): conditional−d_unit → pot(tag), last_tick_day=n
Ngày >1001: dừng anti-idle → PHA-2 (4.4).
```

### 4.4 PHA-2 (job EPOCH — backend, n>1001)
```
Mỗi vault n>1001, e = p2_epoch(now):
  nếu epoch_active(e) ∧ e>last_tick_epoch ∧ conditional≥1:
    build OwnEpoch tx (keeper-sig): conditional−q → owned+q, last_tick_epoch=e   (LAMP ở-lại vault)
  nếu idle: khi (e − last_tick_epoch ≥ 1001) ∧ conditional≥1:
    build ReclaimEpoch tx (keeper-sig): conditional → pot (recreate nếu owned>0 / ĐÓNG+burn nếu owned=0)
```

### 4.5 Redeem (owner rút owned — TUỲ-CHỌN, mọi lúc)
```
5a POST /wakeme/redeem/build {amount_lamp ≤ owned}:
   precondition: owned ≥ amount (KHÔNG phase-gate)
   build unsigned: ref-input taad anchor → spend VAULT Redeem → owner_address(tag) +amount
   (nhánh ĐÓNG nếu conditional=0 ∧ amount=owned: +CloseVault burn)
Core: Enclave witness (controller+device)
5b POST /wakeme/redeem/submit → {cardano_tx_hash, REDEEMED}
```

### 4.6 GetMAGIC (fiat→CARP — backend, NGOÀI validator)
```
quote (FX buffer) → checkout (VietQR/gateway) → poll tới CARP_DELIVERED
1 CARP = 1 MAGIC. KHÔNG mint CARP tự-do; backing qua GreenBack backed-path [CARP-team].
```

---

## 5. API backend

Prefix `/api/v1`, body snake_case, `DataResponse<T>{code,message,result}` (`code=1000`=OK). Mẫu **build-unsigned → Enclave witness → submit**. Đơn-vị: trả cả `_lamp` + `_oildrop` (×10⁶); MAGIC theo `nanogic = MAGIC×10⁹`.

> **⚠ BREAKING — đổi namespace `/activation/*` → `/wakeme/*`** (anh Aladin chốt 2026-07-31). SuperApp/SDK/Frontend đã mock `/activation/*` → PHẢI điều-phối đổi đồng-thời (issue Long/Tuân + inbox SuperApp/Thư). **Tên path dưới là ĐỀ-XUẤT — chốt với backend trước khi hard-code.**

| # | Method | Path | Redeemer/tx | Auth | Chặn bởi |
|---|---|---|---|---|---|
| 1a | POST | `/wakeme/build` | GenesisVault | auth | pot deploy, validator |
| 1b | POST | `/wakeme/submit` | (submit) | auth | — |
| 2 | GET | `/wakeme/vault/{did}` | (đọc datum) | public | backend |
| 3 | GET | `/wakeme/vault/{did}/magic` | (đọc Gen ref) | public | MAGIC-team |
| 5a | POST | `/wakeme/redeem/build` | Redeem | auth | validator |
| 5b | POST | `/wakeme/redeem/submit` | (submit) | auth | — |
| 6a-c | — | `/wakeme/getmagic/*` | (fiat→CARP) | auth | GreenBack/CARP |
| 7 | GET | `/wakeme/pot` | (đọc pot) | public | pot deploy |

**Endpoint 2 (`vault/{did}`) — map field validator → JSON:** `phase = (days_elapsed≤1001 ? 1 : 2)`; `conditional_lamp` (usage-right khoá, KHÔNG rút), `owned_lamp` (sở-hữu-hẳn, rút được qua Redeem) — 2 bucket phân-biệt RÕ trên UI; `reclaimed_to_pot_lamp`; `days_elapsed`; `d_unit_oildrop`; `activity_gate.used_this_period = null` tới khi Registry-team nối; cảnh-báo forfeit tính từ `last_tick_epoch` gap. Self-consumption HỢP-LỆ. Mã-lỗi nhánh **135x**.

**Nhóm endpoint (chi-tiết request/response chốt với backend):**
- **1a build** → `{unsigned_tx_cbor, required_signer_key_hash, vault_address, usage_right_lamp, usage_right_oildrop, d_unit_oildrop, pot_balance_lamp, vest_start_slot, ttl_slot}`. Precondition 1-DID-1-vault (`VAULT_ALREADY_EXISTS`).
- **2 vault/{did}** → dashboard 2-pha: `phase, days_elapsed, conditional_lamp, owned_lamp, reclaimed_to_pot_lamp, d_unit, magic_*, activity_gate{...}`. `phase/conditional/owned` tính NGAY từ validator; `magic_*` treo tới khi MAGIC/Registry nối.
- **3 vault/{did}/magic** → MAGIC-yield ĐỌC-số-dư (`gen_basis_lamp = conditional + owned`), note "Gen CHỈ ĐỌC — không burn LAMP".
- **5a redeem/build** → `{amount_lamp ≤ owned}`; KHÔNG phase-gate (`NO_OWNED_BALANCE` nếu owned=0). KHÔNG phụ-thuộc Gen/GreenBack.
- **6a-c getmagic** → quote (FX buffer) → checkout → poll `CARP_DELIVERED`. `1 CARP = 1 MAGIC`; KHÔNG mint CARP tự-do.
- **7 pot** → `{pot_balance_lamp, current_usage_right_lamp, cap_lamp: 1001, saturated}`.

**Mã-lỗi 135x (backend `ErrorCode.java`):** `VAULT_ALREADY_EXISTS`(1350), `VAULT_NOT_FOUND`(1351), `POT_UNAVAILABLE`(1352), `NO_OWNED_BALANCE`(1353), `REDEEM_EXCEEDS_OWNED`(1354), `GETMAGIC_*`(1360-1363). Tái dùng: `UNAUTHORIZED`(1304), `WALLET_NOT_REGISTERED`(1320), `USER_DID_NOT_FOUND`(2002), `SIGNATURE_INVALID`(1403), `CARDANO_TX_FAILED`(5101).

**⚠ Không còn `abandon-phase1` / `claim-vested`.** BẢN A không có redeemer thoát-sớm PHA-1 tự-nguyện (thoát PHA-1 = để anti-idle tự thu-hồi `d_unit`/ngày). "Rút LAMP" = **Redeem** phần `owned` (PHA-2 trở đi). Escape-hatch owner (ReclaimEpoch owner-sig) chỉ khi idle-gap ≥ 1001 epoch.

**Luồng UI SuperApp (đội giao-diện dựng, mock trước):** Onboarding (keygen vân tay) → Wakeme (1 nút "Nhận LAMP", banner từ `/pot`) → Vault Dashboard (`/vault/{did}`: phân-biệt RÕ `conditional_lamp` khoá vs `owned_lamp` rút-được; PHA-1 KHÔNG nút rút; PHA-2 nút [Rút LAMP] = Redeem `owned`) → GetMAGIC. KHÔNG hiển-thị phí ADA (Feecover lo).

---

## 6. Ranh-giới giao-việc

| Tầng | Việc | Đội | Phụ-thuộc-chặn |
|---|---|---|---|
| **On-chain (Aiken)** | validator vault BẢN A: 4 spend + 2 mint-gate, đồng-hồ NGÀY+EPOCH, OwnEpoch chuyển-sở-hữu + ReclaimEpoch forfeit gap-1001 + Redeem owner. + validator pot (`dist_treasury`). | đội on-chain | `plutus.json` khớp `.ak` (9-field datum, 4-redeemer); rust_core index mới |
| **Backend (Java)** | genesis orchestration (đọc pot→usage-right→apply-param→build); **job NGÀY** (Reclaim keeper); **job EPOCH** (OwnEpoch active / ReclaimEpoch gap, keeper); Redeem build/submit; GetMAGIC; keeper wallet. `curl` verify sau deploy. | đội backend | rename `/activation/*`→`/wakeme/*` |
| **Core / Enclave** | keygen vân tay; ký GenesisVault + Redeem (controller+device qua anchor); UI 1-nút + dashboard 2-pha (conditional vs owned) | Core | — |
| **MAGIC/CARP-team** | engine Gen ĐỌC-số-dư (reference-input → MAGIC, **KHÔNG spend LAMP, KHÔNG mint token**) | MAGIC/CARP | Gen phải read-only trước khi nối (nhánh chuyển-LAMP cũ = rút LAMP thật) |
| **Registry-team** | chuẩn danh-mục dịch-vụ-tiêu-tài-nguyên-thật → keeper attest active/idle | Registry-team | anti-idle/epoch-gate production cần chuẩn |
| **LAMP-team / Feecover** | nạp pot lần-đầu (`dist_treasury`) + **nguồn Feecover** (phí CARP → buyback/redeem LAMP → pot) + cân-đối dòng-vest-ra | LAMP + backend | Math §9 |

**Ranh-giới ký (tóm):** owner-witness (2-of-2 controller+device qua `anchor_controller_ok`) = GenesisVault + Redeem + ReclaimEpoch-owner-escape; keeper-sig = Reclaim + OwnEpoch + ReclaimEpoch-keeper (system-authority MVP, thay bằng Registry consume-event = B2).

> **Ranh-giới sửa code:** validator + backend thuộc PhoenixKey backend — doc này chỉ đặc-tả, KHÔNG sửa code. Lỗi → Issue giao đội backend/on-chain.

---

## 7. Thứ-tự deploy + phụ-thuộc-chặn

**Build/deploy được NGAY:**
1. Deploy validator pot (`dist_treasury`) + nạp-vốn-đầu (LAMP-team) — chặn 1a, 7.
2. Publish reference-script `wakeme_vault` (rebuild `plutus.json`, hash `3f6e5bf6…`).
3. Backend apply-param builder (7 tham-số per-DID → script-hash + address).
4. Genesis build/submit (1a/1b).
5. Redeem (5a/5b) — chỉ cần validator + anchor.

**Chặn bởi blocker:**
6. **Anti-idle (Reclaim) + OwnEpoch + ReclaimEpoch** — MVP chạy keeper-thủ-công test testnet; production chờ **Registry** (attest active/idle) = **B2**.
7. **Gen production (MAGIC)** — chờ MAGIC-team engine đọc-số-dư (read-only) = **B1**. Trước đó MAGIC field trả null.
8. **GetMAGIC** — chờ CARP/GreenBack + gateway fiat.
9. **Nguồn pot Feecover + cân-đối** — LAMP/Feecover (Math §9).
10. **Wakeme-PersonDID production CHẶN** tới khi uniqueness person-level land (lỗ anchor-uniqueness, Exec §7) = **B3**. Org/Service/Enterprise DID (parent-sig) KHÔNG chặn.

---

## 8. Test / evidence

**Đã có (verify THẬT `aiken check`, worktree BẢN A, 2026-07-31):**
```
Summary 491 checks, 0 errors, 10 warnings
```
Bao-phủ: POSITIVE (genesis khuôn, Reclaim thu d_unit + close, OwnEpoch conditional→owned value-bất-biến, ReclaimEpoch forfeit gap + close, Redeem owned→owner + close). NEGATIVE (đích-sai, rút>giới-hạn, sai-pha, thiếu-chữ-ký keeper/owner, thiếu device-sig 2-of-2, OwnEpoch rút-LAMP-khỏi-vault, OwnEpoch chuyển>q, gap<1001, Redeem chạm-conditional, back-date vest_start, drain-ADA/token-lạ, double-tick) + `double_sat_two_vault_inputs_rejected`. Script-hash bất-biến qua rename định-danh: `3f6e5bf6…f23`.

**CẦN thêm (backend/integration — chưa có):**
- Round-trip CBOR aiken↔rust_core: encode datum **9-field BẢN A** (owned/d_unit, last_tick_epoch=−1) + decode redeemer **index mới** (Reclaim=0…Redeem=3).
- Testnet e2e Preprod: genesis → Reclaim (keeper) → OwnEpoch → Redeem → ReclaimEpoch-close. `curl` từng endpoint sau deploy (1 lệnh 30s).
- Apply-param determinism: cùng DID → cùng script-hash + address.

---

## 9. Luật ràng-buộc khi nối phần phụ-thuộc đội khác

- **B1 (MAGIC/CARP):** engine Gen PHẢI dùng **reference-input** (đọc, KHÔNG spend/đốt LAMP). Wakeme validator KHÔNG có redeemer Gen — KHÔNG được tái dùng mẫu fire→Treasury (spend LAMP) cũ.
- **B2 (Registry):** anti-idle/epoch-gate production CHỈ bật sau khi Registry nối `has_counterparty_consume` (stub `False` hiện tại). MVP keeper attest.
- **B3 (uniqueness):** Wakeme-PersonDID production chặn tới khi uniqueness person-level land. Org/Service/Enterprise (parent-sig) không chặn.
- **Nguồn pot (Feecover):** pot bù bằng thặng-dư Feecover (phí CARP → buyback LAMP giá thấp / redeem GreenCheck → pot). Cơ-chế mua-lại thuộc Feecover/LAMP; Wakeme là bên NHẬN (kế-toán Math §9).

→ Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## Nguồn

- Code (nguồn chân-lý): `PhoenixKey-Validator/validators/wakeme_vault.ak`, `lib/phoenixkey/wakeme_logic.ak`, `lib/phoenixkey/auth_logic.ak`; `plutus.json` (hash `3f6e5bf6…f23`).
- Mô hình BẢN A: Issue #67 (2026-07-30); gỡ GenDrip + usage-right + rename wakeme (2026-07-31).
- Nguồn thiết-kế nội-bộ (không công khai).
- Tài-liệu cùng bộ: [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md), [Math](./PhoenixKey-Wakeme-Math.md), [Exec](./PhoenixKey-Wakeme-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
