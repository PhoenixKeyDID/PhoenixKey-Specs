# PhoenixKey — Wakeme · Đặc-tả KỸ-THUẬT (PHA-1 Ngày + PHA-2 Kỳ 5-LAMP/epoch)

> **⚠ Sửa lớn cơ-chế PHA-2 (2026-07-12) — north-star:** PHA-2 chuyển từ vest-ghi-sổ-1/ngày + ClaimVested sang **OwnEpoch/ReclaimEpoch** theo EPOCH (1 Epoch = 5 ngày = 432.000 slot), `q = min(5, c)` LAMP/epoch. Chi-tiết đổi + ánh-xạ redeemer: xem changelog §0 của [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md). **Validator/backend hiện-hành còn ở mô-hình cũ** — doc này mô-tả THIẾT-KẾ ĐÍCH để đội on-chain + backend rebuild; neo `file:hàm` trỏ code cũ tới khi rebuild.

> **Module:** Wakeme (GetLAMP — kích-hoạt nhận LAMP). **Loại doc:** Kỹ-thuật. **Ngày:** 2026-07-09.
> **Đối-tượng đọc:** KỸ SƯ triển khai (đội on-chain, đội backend, Core/Enclave, MAGIC/CARP/Registry-team).
> **Mục đích:** kiến-trúc + datum/redeemer CBOR + luồng tx + API backend + ranh-giới + thứ-tự deploy — đủ để code, không cần đọc lại Feat/Math. Toán/bất-biến: [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md). Người-dùng: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md). Điều-hành: [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).
> **Nguồn đối-chiếu (code = nguồn chân-lý):**
> - `PhoenixKey-Validator/validators/activation_vault.ak` (validator dispatch: 5 spend redeemer + 2 mint redeemer)
> - `PhoenixKey-Validator/lib/phoenixkey/activation_logic.ak` (toán + invariant, `*_ok`)
> - `PhoenixKey-Validator/lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok` — owner-sig canonical)
> - Nguồn thiết-kế nội-bộ (không công khai) — toán/invariant/ranh-giới/phụ-thuộc + API endpoint SuperApp (gộp vào §5 dưới).
>
> **Khi văn ≠ code → code thắng.** Struct/enum thật neo tại `activation_logic.ak:110-140`.
>
> → Trạng-thái build/test/deploy hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 1. Kiến-trúc

### 1.1 Sơ-đồ thành-phần

```
┌─────────────────────── CORE / ENCLAVE (Flutter + rust_core) ────────────────────────┐
│  vân tay → Secure Enclave (Master_KEK) → khoá controller DID → ví Phoenix (did_payment)│
│  ký: GetLAMP-genesis (owner-witness). PHA-2 sở-hữu tự vào ví (keeper-driven OwnEpoch) │
└───────────────┬──────────────────────────────────────────────────────────────────────┘
                │ build-unsigned → witness → submit
                ▼
┌─────────────────────── BACKEND (PhoenixKey-Database, Java — đội backend) ──────────────┐
│  ActivationController /api/v1/activation/*                                             │
│  · GetLAMP orchestration: đọc pot → D=min(1001,⌊pot/1e6⌋) → build tx genesis          │
│  · Anti-idle job NGÀY (PHA-1): scan vault n≤1001, idle → Reclaim tx (keeper-sig)       │
│  · Epoch job PHA-2 (n≥1002): mỗi epoch mới (e>te), active → OwnEpoch (q=min(5,c)→ví    │
│    Phoenix DID); idle → ReclaimEpoch (q=min(5,c)→pot) (keeper-sig)                     │
│  · GetMAGIC (fiat→CARP)                                                                │
│  keeper wallet (system-authority MVP) ký Reclaim / OwnEpoch / ReclaimEpoch             │
└───┬───────────────────────────────────────────────────────────────┬───────────────────┘
    │ submit tx                                                       │ reference-input (đọc)
    ▼                                                                 ▼
┌── CARDANO (Plutus V3, Preview) ──────────────────────────────────────────────────────┐
│  validator activation_vault (apply-param per-DID)                                     │
│    mint-gate: GenesisVault | CloseVault  (own_policy ≡ script-hash = vault-NFT policy) │
│    spend: GenDrip | Reclaim | OwnEpoch | ReclaimEpoch   (4 spend redeemer)             │
│  VAULT UTxO = Script(own_hash) + [vault-NFT(own,owner_commit)=1] + minADA + c-LAMP khoá│
│                                                        ▲ reference-input               │
│                       ┌── pot (dist_treasury) ──┘ (Reclaim/ReclaimEpoch đích)          │
│    OwnEpoch đích = owner_address = ví Phoenix của DID (did_payment, apply-param)       │
└──────────────────────────────────────────────────────────────────────────────────────┘
        ▲ ĐỌC-số-dư (reference-input VaultDatum) → drip MAGIC → KHÔNG spend LAMP
┌── MAGIC/CARP engine (repo /CARP — Gen, ngoài validator này) ──────────────────────────┐
│  InstantGen + ScheduleGen đọc conditional_lamp → magic_batches                         │
│  MAGIC = account-trong-Vault (nanogic = MAGIC×10⁹), KHÔNG mint token / KHÔNG policy-id  │
│  [BLOCKER §3.7: chưa spell-out on-chain đọc-datum → drip]                              │
└──────────────────────────────────────────────────────────────────────────────────────┘
┌── Registry-team ── chuẩn dịch-vụ-tiêu-tài-nguyên-thật → keeper attest active [BLOCKER I-ACT-3]┐
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Bất-biến kiến-trúc (load-bearing)

- **1 vault = 1 UTxO** tại `Script(own_hash)`, mang **vault-NFT singleton** `(policy = own_hash, name = owner_commit = did_commit)`, min-ADA, và `conditional_lamp` LAMP token khoá (vault sống chỉ giữ conditional). Neo: `has_vault_nft`, `find_vault_output`.
- **Sổ ↔ value đồng-bộ (bất-biến cốt-lõi):** `lamp_token_in_vault == conditional_lamp` — mọi redeemer ép lại. Per-vault bảo-toàn: `D == conditional_lamp + owned_out + reclaimed_to_pot`.
- **own_policy ≡ script-hash** (multi-purpose, mẫu `taad`/`state_nft`): vault-NFT policy CHÍNH là hash validator → validator PHẢI có `mint` handler, nếu không mọi tx Mint rơi `else→fail`. Ép ở spend qua `expect Script(own_policy) = own_input.output.address.payment_credential`; mint handler ở `activation_vault.ak`.
- **conditional_lamp CHỈ GIẢM** (Reclaim→pot 1/ngày | OwnEpoch→ví Phoenix DID q/epoch | ReclaimEpoch→pot q/epoch); KHÔNG có đường tăng. **owned_out** (field 4, đổi tên từ `vested_unlocked`) chỉ TĂNG — bộ-đếm AUDIT luỹ-kế LAMP đã ra ví, KHÔNG là số-dư-trong-vault.
- **KHÔNG có đường `conditional_lamp → user` sai đích** — chỉ `→ pot` (Reclaim/ReclaimEpoch) hoặc `→ ví Phoenix của DID` (OwnEpoch, đích cứng `owner_address`). LAMP đã ra ví = NGOÀI vault, không redeemer nào đụng lại (bất-khả-xâm-phạm). KHÔNG còn ClaimVested.
- **LAMP 36B không-burn:** mọi LAMP rời vault về {pot | ví-Phoenix-của-DID}, không đích đốt. GenDrip sinh MAGIC không spend/đốt LAMP (I-ACT-7).

### 1.3 Đồng-hồ (2 thang: NGÀY + EPOCH)

| Thang | Hằng (slot) | Dùng cho | Hàm |
|---|---|---|---|
| NGÀY | `slots_per_day = 86_400` | ranh-giới-pha, anti-idle tick PHA-1 1/ngày, monotonic `last_tick_day` | `days_elapsed(lo, vest_start)` |
| EPOCH | `slots_per_epoch = 432_000` (**1 Epoch = 5 NGÀY = 432.000 slot**) | nhịp PHA-2: gate epoch-active/idle, OwnEpoch/ReclaimEpoch, monotonic `last_tick_epoch` | `p2_epoch(lo, vest_start)` |

- `n = ⌊(lo − vest_start_slot) / 86_400⌋` — số NGÀY (≥0). `lo` = **lower-bound HỮU-HẠN** của validity-range (`tx_lo`); `−∞` → `None` → REJECT (chống khai-man thời-gian).
- `p2_epoch` tính TỪ ĐẦU PHA-2 (ngày 1002): `off = lo − vest_start − 1001×86400`; `e = ⌊off / 432000⌋`. `off < 0` → `-1` (chưa tới PHA-2).
- **Ranh-giới pha:** PHA-1 "Giai đoạn Ngày" = `n ≤ 1001` (`phase1_last`); PHA-2 "Giai đoạn Kỳ" = `n ≥ 1002` (`n > phase1_last`). Bất-biến bất-kể D.
- **Đơn-điệu epoch (PHA-2):** mỗi epoch xử ĐÚNG một lần — guard `e > last_tick_epoch`, hậu-điều-kiện `last_tick_epoch′ = e`. Cả OwnEpoch (active) lẫn ReclaimEpoch (idle) đều tiến `te`.

### 1.4 Hằng-số (nguồn: `activation_logic.ak:81-106`)

| Hằng | Giá-trị | Ý-nghĩa |
|---|---|---|
| `slots_per_day` | 86_400 | 1 NGÀY |
| `grace_days` | 7 | onboarding miễn anti-idle (PHA-1) |
| `reclaim_unit` | 1 | PHA-1: 1 conditional_lamp/NGÀY-idle → pot |
| `d_cap` | 1001 | trần D (= 1001 ngày cam-kết) |
| `phase1_last` | 1001 | ranh-giới PHA-1/PHA-2 |
| `slots_per_epoch` | 432_000 | **1 EPOCH = 5 NGÀY = 432.000 slot** — nhịp PHA-2 |
| `epoch_unit` | 5 | **PHA-2: `q = min(5, c)` LAMP/epoch** (1/ngày × 5 ngày) — OwnEpoch (→ví) hoặc ReclaimEpoch (→pot) |

> **Đã BỎ** `forfeit_idle_epochs = 1001` (thu-hồi PHA-2 nay per-epoch idle, không chờ gap-1001). **Đã THÊM** `epoch_unit = 5`.

---

## 2. Datum / Redeemer — khuôn CBOR (khớp aiken ↔ rust_core)

> **NGUỒN CHUẨN = code `.ak`**; `plutus.json` v4.1-current đã khớp (mục 0). Plutus V3 dùng `Constr` cho tất cả.

### 2.1 `ActivationVaultDatum` — 9 field (thứ-tự CBOR CỐ ĐỊNH)

Neo: `activation_logic.ak:110-122`. `Constr 0 [...]` — thứ-tự field LÀ thứ-tự CBOR (rust_core PHẢI encode đúng thứ-tự này):

| # | Field | Aiken type | CBOR | Ý-nghĩa | Khởi-tạo (Genesis) |
|---|---|---|---|---|---|
| 0 | `owner_commit` | `ByteArray` | `bytes` | did_commit gắn vault + **vault-NFT name** (I-ACT-10) | `blake2b_256(did)` (32B) |
| 1 | `did_commit` | `ByteArray` | `bytes` | con-trỏ DID (attribution) — MVP `== owner_commit`, phải `≠ #""` | `blake2b_256(did)` |
| 2 | `vest_start_slot` | `Int` | `int` | mốc 0 đồng-hồ (slot lúc GetLAMP) | `now` (slot submit) |
| 3 | `conditional_lamp` | `Int` | `int` | LAMP còn khoá điều-kiện = **toàn-bộ LAMP trong vault sống** (oildrop) | `D` (∈ [1, 1001]) |
| 4 | `owned_out` *(cũ `vested_unlocked`)* | `Int` | `int` | **luỹ-kế** LAMP đã chuyển RA ví Phoenix (đã-sở-hữu, ngoài vault); AUDIT chỉ-tăng, KHÔNG là số-dư-vault | `0` |
| 5 | `reclaimed_to_pot` | `Int` | `int` | Reclaim PHA-1 + ReclaimEpoch PHA-2 luỹ-kế (audit) | `0` |
| 6 | `last_tick_day` | `Int` | `int` | NGÀY tick gần nhất (monotonic, PHA-1) | `0` |
| 7 | `idle_epochs_p2` | `Int` | `int` | audit: luỹ-kế epoch idle đã ReclaimEpoch (KHÔNG load-bearing) | `0` |
| 8 | `last_tick_epoch` | `Int` | `int` | p2-epoch cuối **ĐÃ XỬ-LÝ** (OwnEpoch active HOẶC ReclaimEpoch idle); mốc đơn-điệu PHA-2 | `0` |

**Đơn-vị:** on-chain LAMP = **oildrop** (`LAMP × 10⁶`). Datum lưu oildrop; API trả cả `lamp` + `oildrop`.

**Diễn-CBOR (Genesis, D=1001 LAMP = 1001000000 oildrop, ví-dụ):**
```
d8799f                                    # Constr 0 (tag 121), 9 field
  581f <owner_commit 32B hex>             # bytes owner_commit
  581f <did_commit 32B hex>               # bytes did_commit
  1a<vest_start_slot>                     # int slot
  1a3b9c3b80                              # int 1001000000 (conditional_lamp oildrop)
  00                                      # int 0 (owned_out — cũ vested_unlocked)
  00                                      # int 0 (reclaimed_to_pot)
  00                                      # int 0 (last_tick_day)
  00                                      # int 0 (idle_epochs_p2)
  00                                      # int 0 (last_tick_epoch)
ff
```
(Số byte-len prefix minh-hoạ; encoder rust_core tự tính. Điểm load-bearing = **9 field, đúng thứ-tự, đúng khởi-tạo Genesis** để pass `genesis_vault_ok`. Rebuild: đổi tên field 4 `vested_unlocked → owned_out` — CBOR layout KHÔNG đổi, chỉ nhãn.)

### 2.2 `ActivationRedeemer` (spend) — **4 nhánh** (mô-hình mới)

Tất-cả **không field** (`Constr idx []`). **Ánh-xạ từ bộ cũ:** `VestToOwner → OwnEpoch`; `ClaimVested → BỎ` (hấp-thu vào OwnEpoch); `ForfeitPhase2 → ReclaimEpoch`.

| idx | Redeemer | CBOR | Ai dùng |
|---|---|---|---|
| 0 | `GenDrip` | `d87980` | engine Gen (spend+recreate, LAMP-preserved) |
| 1 | `Reclaim` | `d87a80` | keeper (anti-idle PHA-1 Ngày, 1/ngày → pot) |
| 2 | `OwnEpoch` | `d87b80` | keeper attest ACTIVE (PHA-2 Kỳ; `q=min(5,c)` → ví Phoenix DID) |
| 3 | `ReclaimEpoch` | `d87c80` (Constr 3) | keeper attest IDLE (PHA-2 Kỳ; `q=min(5,c)` → pot) |

> Bộ cũ export 5 redeemer (có `ClaimVested`+`ForfeitPhase2`). Rebuild → 4 redeemer trên; `plutus.json` regen cho khớp. rust_core/backend dùng idx mới.

### 2.3 `VaultMintRedeemer` (mint-gate) — 2 nhánh

| idx | Redeemer | CBOR | Dùng |
|---|---|---|---|
| 0 | `GenesisVault` | `d87980` | GetLAMP — đúc 1 vault-NFT + ép khuôn output |
| 1 | `CloseVault` | `d87a80` | OwnEpoch/ReclaimEpoch đưa `c → 0` — pure-burn NFT |

### 2.4 Tham-số apply-param (per-DID, đóng lúc GetLAMP)

Backend apply 7 tham-số → sinh script-hash + address RIÊNG mỗi DID (mẫu did_payment per-DID):

| Param | Type | Nguồn | Ghi-chú |
|---|---|---|---|
| `anchor_nft_policy` | `PolicyId` | hằng TOÀN-HỆ (`taad` Design-2 hash) | (giữ tham-số; mô-hình mới không dùng owner-sig cho spend — có thể loại khi rebuild nếu không cần đọc controller) |
| `anchor_nft_name` | `AssetName` | `blake2b_256(did)` | per-DID anchor |
| `lamp_policy` | `PolicyId` | LAMP canonical | LAMP khoá trong vault |
| `lamp_name` | `AssetName` | asset-name LAMP | |
| `pot_address` | `Address` | pot User (instance `dist_treasury`) | Reclaim/ReclaimEpoch BUỘC đích về đây |
| `owner_address` | `Address` | **ví Phoenix của DID** (`did_payment`) | **OwnEpoch BUỘC LAMP-sở-hữu về đây**; DID-bound, rotation-safe (xoay khoá controller KHÔNG đổi đích) |
| `keeper_pkh` | `ByteArray` (pkh) | system-authority (MVP) | ký Reclaim/OwnEpoch/ReclaimEpoch; **sau thay bằng consume-event Registry ref-input** |

---

## 3. Từng redeemer — điều-kiện + shape tx + ai-ký

Ký-hiệu: `d_in`/`d_out` = datum vào/ra; `lamp_in`/`lamp_out` = LAMP oildrop trong vault vào/ra; `n = days_elapsed`; `e = p2_epoch`.

### 3.0 Mint-gate

#### `GenesisVault` (GetLAMP) — `genesis_vault_ok`
- **Điều-kiện:** (1) mint policy own = **đúng 1 movement, qty +1** (không bundle burn/mint 2); (2) carrier output DUY NHẤT tại `Script(own_policy)` mang đúng 1 NFT(name) + datum well-formed; (3) `name == owner_commit == datum.owner_commit`; (4) khuôn khởi-tạo: `conditional_lamp ∈ [1,1001]`, `owned_out=0`, `reclaimed_to_pot=0`, `last_tick_day=0`, `idle_epochs_p2=0`, `last_tick_epoch=0`, `did_commit ≠ #""`; (5) `lamp_locked == conditional_lamp` (pot cấp LAMP THẬT — chống forge datum bịa D); (6) output chỉ policy ⊆ {ada, lamp, own}.
- **Ký:** owner-witness (Enclave). Pot chi LAMP do **validator POT** tự gác (không thuộc handler này).
- **Shape tx:**
  ```
  in:  [pot UTxO (D LAMP)] + [ví Phoenix (fee/collateral)]
  mint: +1 vault-NFT(own_policy, owner_commit)   redeemer=GenesisVault
  out: [VAULT: Script(own) + NFT + minADA + D-oildrop LAMP + inline datum(9-field Genesis)]
       [pot recreate (giảm D)]  [change]
  signer: owner (did_payment)   validity: [lo, lo+ttl]
  ```

#### `CloseVault` — `close_vault_ok`
- **Điều-kiện:** pure-burn — `≥1 movement` policy own, **mọi movement ÂM** (`p.2nd < 0`). Nối OwnEpoch-close hoặc ReclaimEpoch-close (khi `c → 0`).
- **Ký:** đi-kèm redeemer spend đóng (keeper).

### 3.1 `GenDrip` — LAMP-preserved (MAGIC yield) — `gen_drip_ok`
- **Điều-kiện:** (1) LAMP KHÔNG rời: `conditional'==conditional`, `owned_out'==owned_out`, `lamp_out==lamp_in==conditional'`; (2) `reclaimed`/`last_tick_day`/`idle_epochs_p2`/`last_tick_epoch`/DID/mốc **bất-biến** (Gen chỉ ĐỌC); (3) anti-drain (min-ADA giữ + chỉ policy hợp-lệ).
- **Ký:** engine Gen (interface MVP — §3.7 chưa chốt cơ-chế MAGIC). **Điều-kiện tối-thiểu load-bearing = "LAMP không rời qua GenDrip"** (I-ACT-7). Gen đọc `conditional_lamp` làm cơ-số sinh MAGIC. Khi §3.7 chốt: thêm điều-kiện đọc/ghi `magic_batches`. **MAGIC = account-trong-Vault (KHÔNG mint token).**
- **Shape:** spend vault + recreate y-hệt (chỉ tầng MAGIC ngoài đổi). **Khuyến-nghị dùng reference-input** cho Gen đọc-số-dư (KHÔNG spend) — xem §5 [CẦN CHỐT §3.7].

### 3.2 `Reclaim` — anti-idle PHA-1 (Ngày) — `reclaim_ok`
- **Điều-kiện (10):** (1) **keeper ký**; (2) PHA-1 `n ≤ 1001`; (3) ngoài grace `n ≥ 7`; (4) monotonic `n > last_tick_day` (chống double-tick); (5) `conditional_lamp ≥ 1`; (6) sổ: `conditional'=conditional−1`, `reclaimed'=reclaimed+1`, `owned_out'=owned_out`, `last_tick_day'=n`, epoch-tracking bất-biến, DID/mốc bất-biến; (7) `lamp_out==lamp_in−1==conditional'`; (8) **đích:** `≥1 LAMP tới pot_address` (LỖ-1 fix — không vào ví keeper); (9) anti-drain.
- **Ký:** keeper (system-authority MVP; `keeper_signed = list.has(self.extra_signatories, keeper_pkh)`).
- **Shape tx:**
  ```
  in:  [VAULT (lamp_in LAMP)]   redeemer=Reclaim
  out: [VAULT recreate: conditional−1, last_tick_day=n]  [pot: +1 LAMP]  [change]
  signer: keeper   validity.lo hữu-hạn (bắt buộc — days_elapsed)
  ```

### 3.3 `OwnEpoch` — giao-quyền PHA-2 (Kỳ, ACTIVE) — `own_epoch_ok` / `own_epoch_close_ok`
- **`q = min(epoch_unit=5, conditional_lamp)`** — 1 LAMP/ngày × 5 ngày/epoch.
- **Điều-kiện (9):** (1) PHA-2 `n > 1001`; (2) **keeper ký** (attest epoch ACTIVE — MVP, KHÔNG permissionless); (3) **đơn-điệu epoch** `e > last_tick_epoch` (mỗi epoch xử ĐÚNG một lần); (4) `q ≥ 1` (còn `conditional`); (5) sổ LAMP: `conditional'=conditional−q`, `owned_out'=owned_out+q`, `reclaimed` bất-biến (Δc+Δo=0); (6) sổ EPOCH: `last_tick_epoch'=e`, `last_tick_day'=n`, `idle_epochs_p2` bất-biến, DID/mốc bất-biến; (7) **`lamp_out==lamp_in−q==conditional'`** (q LAMP RỜI vault); (8) **đích:** `≥q LAMP tới owner_address` (ví Phoenix của DID — đích cứng, keeper KHÔNG chọn được); (9) anti-drain.
- **2 nhánh:** SỐNG (`c−q>0`, recreate) — `own_epoch_ok`; ĐÓNG (`c−q==0`, burn) — `own_epoch_close_ok` + `close_vault_ok`.
- **Ký:** keeper. **Không cần owner-sig** vì đích cứng = ví Phoenix của DID (apply-param, DID-bound) ⟹ keeper không đoạt được.
- **Shape tx:**
  ```
  in:  [VAULT]   redeemer=OwnEpoch
  out (SỐNG): [VAULT recreate: conditional−q, owned_out+q, last_tick_epoch=e, last_tick_day=n]
              [owner_address (ví Phoenix DID): +q LAMP]  [change]
  out (ĐÓNG): [owner_address: +q LAMP]  + mint: −1 vault-NFT (CloseVault)
  signer: keeper   validity.lo hữu-hạn
  ```

### 3.4 `ReclaimEpoch` — thu-hồi PHA-2 (Kỳ, IDLE) — `reclaim_epoch_ok` / `reclaim_epoch_close_ok`
- **`q = min(5, conditional_lamp)`.**
- **Điều-kiện (8):** (1) PHA-2 `n > 1001`; (2) **keeper ký** (attest epoch IDLE); (3) **đơn-điệu epoch** `e > last_tick_epoch`; (4) `q ≥ 1`; (5) sổ: `conditional'=conditional−q`, `reclaimed'=reclaimed+q`, `owned_out'=owned_out` (BẤT-BIẾN — phần đã ra ví không đụng); (6) sổ EPOCH: `last_tick_epoch'=e`, `idle_epochs_p2'=idle_epochs_p2+1`, `last_tick_day'=n`; (7) **`lamp_out==lamp_in−q==conditional'`**; (8) **đích:** `≥q LAMP tới pot`; anti-drain.
- **2 nhánh:** SỐNG (`c−q>0`) — `reclaim_epoch_ok`; ĐÓNG (`c−q==0`, burn) — `reclaim_epoch_close_ok`.
- **Ký:** keeper.
- **Đo idle per-EPOCH (không gap-1001):** khác mô-hình cũ — thu-hồi MỖI epoch idle tức-thời `q=min(5,c)`, không chờ gap ≥ 1001 epoch. `last_tick_epoch` tiến qua CẢ OwnEpoch lẫn ReclaimEpoch ⟹ đơn-điệu, không xử-lại epoch. **Backend chọn OwnEpoch vs ReclaimEpoch theo attest active/idle của epoch `e` hiện-tại.**
- **Shape tx:**
  ```
  in:  [VAULT]   redeemer=ReclaimEpoch
  out (SỐNG): [VAULT recreate: conditional−q, last_tick_epoch=e, idle_epochs_p2+1]  [pot: +q LAMP]
  out (ĐÓNG): [pot: +q LAMP]  + mint: −1 vault-NFT (CloseVault)
  signer: keeper   validity.lo hữu-hạn
  ```

### 3.6 Bảng ai-ký + đích LAMP

| Redeemer | Ký | LAMP di-chuyển | Đích ép | Pha |
|---|---|---|---|---|
| GenesisVault | owner | pot → vault (D) | vault (khuôn) | tạo |
| GenDrip | engine (MVP) | KHÔNG | — | cả 2 |
| Reclaim | **keeper** | vault → pot (1) | `pot_address` | PHA-1 Ngày (n≤1001) |
| OwnEpoch | **keeper** (attest active) | vault → ví Phoenix DID (q=min(5,c)) | `owner_address` | PHA-2 Kỳ (n≥1002) |
| ReclaimEpoch | **keeper** (attest idle) | vault → pot (q=min(5,c)) | `pot_address` | PHA-2 Kỳ (n≥1002) |
| CloseVault | (kèm spend) | burn NFT | — | đóng (c=0) |

### 3.7 Kiến-trúc SCALE engine Gen/MAGIC — off-chain accounting + Merkle-anchor (giải BLOCKER §3.7)

> Trả lời câu hỏi "cập nhật hàng triệu MAGIC/giây trên vault toàn cầu có tải nổi không, phí nhiều không?" — phân tích first-principles, **chưa code**, cần chốt §3.7d trước khi engine Gen production.

**(a) Vì sao 1-L1-tx-mỗi-sự-kiện bất-khả-thi:**
- Cardano L1: block interval ~20s (`f=0.05`), throughput thực tế **~vài chục tx/giây (bậc 10¹)**. "Hàng triệu balance-delta/giây" vượt trần **10⁴–10⁵ lần**.
- **Hot-UTxO contention độc-lập với throughput**: VAULT là **1 UTxO** (bất-biến kiến-trúc §1.2) — eUTXO chỉ cho **một** tx chi nó mỗi block. Nếu nhiều sự-kiện MAGIC cùng chạm 1 vault trong 1 block → serialize, throughput per-vault sụp về ~0,05 update/giây. GenDrip theo mẫu **spend+recreate y-hệt** (§3.1) mang đúng rủi ro này nếu dùng ở tần-suất cao — đây là lý do khuyến-nghị reference-input đã ghi ở §3.1 KHÔNG chỉ là tối-ưu, mà là **điều-kiện sống-còn** để Gen chạy tần-suất cao.

**(b) Kiến-trúc giải: off-chain accounting per-provider + settlement định-kỳ (Merkle-anchor) + sharding.** Không đặt balance-delta lên L1 theo sự-kiện; gộp:
```
Off-chain (per provider/App):
  signed event log (append-only) — leaf = H(did_key ‖ Δmagic ‖ nonce ‖ user_cosign_sig ‖ prev_commit)
  → Lazy-MMR accumulator (single-writer per DID-shard, WAL durable)
  → current_root cập-nhật mỗi append (KHÔNG chạm L1)
                    │  [trigger: epoch-end]
                    ▼
On-chain (1 tx/provider/epoch):
  settlement: datum commit {anchored_root, epoch, net_CARP} + CARP→Treasury
  (VAULT LAMP-preserved — GenDrip KHÔNG đổi conditional/vested, chỉ tầng MAGIC ngoài ghi root)
```
- **Sharding theo DID**: `hash(did_key) mod K` → mỗi DID thuộc đúng 1 accumulator shard, single-writer nên không cần consensus phân-tán cho số-dư 1 user; K shard chạy song-song, K× throughput.
- **Sharding theo provider**: mỗi App/Platform vault độc-lập, không hot-spot chung.
- Kết quả: **O(N) on-chain → O(1) on-chain/provider/epoch**, verify 1 sự-kiện = inclusion proof O(log N) off-chain.

**(c) Tamper-evidence (chống accounting authority gian-lận sổ off-chain):**

| Cơ-chế | Chống gì |
|---|---|
| User cosign mỗi delta (`user_cosign_sig`) | operator bịa "user tiêu X" |
| Leaf bind `prev_commit` (hash-chain) | operator xoá/đảo thứ-tự delta |
| Merkle/MMR root anchor L1 mỗi epoch | operator sửa sổ sau khi công-bố |
| Nonce đơn-điệu per-DID | replay một delta |
| Operator bond + fraud-proof window | operator gian-lận `net_CARP` cuối epoch |

⟹ Operator có thể **DoS** (từ-chối ghi) hoặc **withhold** (giấu delta) nhưng **KHÔNG forge/rewrite** được một delta đã cosign+anchor.

**(d) Ước-tính phí (bậc độ-lớn, giả-định N_provider~1.000, epoch=5 ngày~73 epoch/năm):** thiết-kế gộp-lô ~**1,5×10⁴ ADA/năm toàn mạng** (≈`5×10⁻¹⁰` ADA/event — coi như miễn-phí on-chain); thiết-kế naive per-event ước ~`5×10¹²` ADA/năm và **vẫn không chạy nổi** vì trần throughput (a). Chênh ~9 bậc độ-lớn/sự-kiện. Chi-phí thật nằm ở off-chain compute (accumulator/WAL — phân-bổ tuyến-tính theo shard), KHÔNG ở L1.

**(e) Tham-khảo lớp anchoring sẵn có (VeData Mosaic)** — Lazy-MMR (accumulate off-chain, anchor theo trigger) + Strategy C (chu-kỳ 4–8h, ~0,001 ADA/record) khớp gần đúng với "hạch-toán cuối epoch". Ranh-giới đề-xuất nếu dùng: **Mosaic = anchoring layer (root commit)**, **MAGIC = accounting layer (số-dư + cosign + CARP)** — không đẩy balance semantics sang Mosaic.

**(f) Test bắt-buộc trước khi tuyên-bố Gen production sẵn-sàng** (evidence output thật, không chỉ structure):

| Test | Pass-condition |
|---|---|
| T-THROUGHPUT | accumulator sustain ≥10.000 signed-delta/s, p99<50ms, 0 mất khi crash-recovery |
| T-SHARD | K shard × tổng 10⁶ delta/s, linear scale, không cross-shard balance race |
| T-ANCHOR-COST | settlement tx thật trên preprod ≤0,2 ADA/tx, 1 tx/provider/epoch |
| T-TAMPER | 4 kịch-bản (bịa không cosign / đảo thứ-tự / replay nonce / sửa sổ sau anchor) đều bị reject/phát-hiện |
| T-DOUBLESPEND | user tiêu vượt số-dư trong-epoch (concurrent 1 shard) → single-writer reject, 0 balance âm |
| T-RECONCILE | cuối epoch: net MAGIC→CARP đúng, CARP về Treasury khớp tổng, `lamp_out==conditional'` không đổi qua GenDrip |

**Giới-hạn/rủi-ro mở (chưa chốt — không phải blocker kiến-trúc, là quyết-định vận-hành):**
- **Thẩm-quyền kế-toán off-chain**: App/Platform operator giữ sổ + cosign user + bond — mô-hình này cần anh duyệt.
- **Finality độ-trễ epoch**: số-dư trong-epoch là *provisional*; Feecover phải chấp-nhận optimistic accounting (tin sổ ngay, reconcile cuối epoch) hoặc dùng Hydra fast-path cho thanh-toán lớn.
- **N_provider + epoch cadence thật** để chốt con số phí (§3.7d dùng 1.000 provider giả-định, không phải cam-kết).
- **Có mời VeData Mosaic tư-vấn interface `append/anchor/proof` không** — khuyến-nghị có, tránh tự dựng lại lớp anchoring.

Nguồn phân-tích đầy-đủ: `spec-proposals/PhoenixKey-MAGIC-Vault-Scale-Analysis.md` (v0.1 DRAFT, 2026-07-01).

---

## 4. Luồng end-to-end

### 4.1 GetLAMP (tạo vault)
```
Core: keygen vân tay → ví Phoenix
1a POST /activation/getlamp/build:
   backend đọc pot_balance → D=min(1001,⌊pot/1e6⌋)
   → apply-param 7 tham-số per-DID → sinh vault script-hash + address
   → build unsigned tx: spend pot(D LAMP) → mint GenesisVault → VAULT(datum 9-field Genesis)
   → trả {unsigned_tx_cbor, required_signer_key_hash, vault_address, d_lamp, vest_start_slot}
Core: Enclave witness (owner)
1b POST /activation/getlamp/submit {signed_tx_cbor} → submit → {cardano_tx_hash, SUBMITTED}
Kết-quả: VAULT conditional_lamp=D, vested=0, vest_start=now.
```

### 4.2 Gen (MAGIC yield — nền, ngoài Wakeme API)
```
engine /CARP đọc VaultDatum(conditional+vested) qua reference-input → drip MAGIC → magic_batches
KHÔNG spend LAMP, KHÔNG mint token (MAGIC = account-trong-Vault).
[BLOCKER §3.7 — nếu spend+recreate: dùng GenDrip redeemer, ép LAMP-preserved]
```

### 4.3 Anti-idle PHA-1 (job NGÀY — backend)
```
Vòng NGÀY, mỗi vault n≤1001:
  active = consume(n) ≥ MIN_MAGIC_CONSUME  (tiêu qua dịch-vụ TRẢ PHÍ Registry §3.3a) [BLOCKER I-ACT-3]
  nếu !active ∧ n≥7 ∧ n>last_tick_day ∧ conditional≥1:
    build Reclaim tx (keeper-sig): conditional−1 → pot, last_tick_day=n
Ngày ≥1002: dừng anti-idle NGÀY → chuyển nhịp KỲ PHA-2 (4.4).
```

### 4.4 Epoch job PHA-2 (job — backend, n≥1002; nhịp KỲ, 1 Kỳ = 5 ngày)
```
Mỗi epoch mới, mỗi vault (chỉ khi e = p2_epoch(now) > last_tick_epoch — xử ĐÚNG một lần):
  q = min(5, conditional)   (5 = 1 LAMP/ngày × 5 ngày)
  active_epoch = consume trong epoch e ≥ MIN_MAGIC_CONSUME (Registry) [BLOCKER I-ACT-3]
  nếu conditional≥1:
    nếu active_epoch:
      build OwnEpoch tx (keeper-sig): conditional−q, owned_out+q, last_tick_epoch=e
        → q LAMP RA ví Phoenix của DID (owner_address); ĐÓNG+burn nếu conditional−q==0
    ngược lại (idle_epoch):
      build ReclaimEpoch tx (keeper-sig): conditional−q → pot, last_tick_epoch=e, idle_epochs_p2+1
        → ĐÓNG+burn nếu conditional−q==0
```
> LAMP đã-kiếm vào THẲNG ví Phoenix của DID mỗi Kỳ active — KHÔNG còn bước ClaimVested user rút. Không có endpoint claim.

### 4.5 (đã BỎ ClaimVested) — LAMP-sở-hữu tự vào ví
Mô-hình mới KHÔNG có luồng user-rút riêng: OwnEpoch (job backend, §4.4) chuyển LAMP đã-kiếm thẳng vào ví Phoenix của DID mỗi Kỳ active. Endpoint `claim-vested/*` cũ **loại bỏ**. Dashboard chỉ hiển-thị `owned_out` (luỹ-kế đã vào ví) + số-dư LAMP trong ví Phoenix (đọc on-chain thường).

### 4.6 GetMAGIC (fiat→CARP — backend, ngoài validator)
```
5a quote (fiat↔CARP, FX buffer) → 5b checkout (VietQR/gateway) → 5c poll tới CARP_DELIVERED
CARP về account user. KHÔNG mint CARP tự-do; backing qua GreenBack backed-path [CARP-team].
```

### 4.7 Vòng đời tổng
```
GetLAMP → [PHA-1 Ngày: Gen yield + Reclaim idle→pot 1/ngày] --ngày 1001-->
[PHA-2 Kỳ (5 ngày/epoch): mỗi epoch → active? OwnEpoch q=min(5,c)→ví Phoenix DID
                                        : idle?  ReclaimEpoch q=min(5,c)→pot]
→ burn NFT khi conditional=0 (mọi conditional đã về chủ hoặc về pot)
```

---

## 5. API backend (gộp API-for-SuperApp)

Prefix `/api/v1`, body snake_case, `DataResponse<T>{code,message,result}` (`code=1000`=OK). Chuyển-giá-trị theo mẫu **build-unsigned → Enclave witness → submit**. Đơn-vị: API trả cả `_lamp` + `_oildrop` (LAMP×10⁶); MAGIC theo `nanogic = MAGIC×10⁹`.

| # | Method | Path | Redeemer/tx | Auth | Chặn bởi |
|---|---|---|---|---|---|
| 1a | POST | `/activation/getlamp/build` | GenesisVault | auth | pot deploy, validator |
| 1b | POST | `/activation/getlamp/submit` | (submit) | auth | — |
| 2 | GET | `/activation/vault/{did}` | (đọc datum) | public | đội backend |
| 3 | GET | `/activation/vault/{did}/magic` | (đọc Gen) | public | MAGIC-team |
| 5a-c | — | `/activation/getmagic/*` | (fiat→CARP) | auth | GreenBack/CARP |
| 7 | GET | `/activation/pot` | (đọc pot) | public | pot deploy |

> **BỎ endpoint claim-vested (4a/4b) + abandon-phase1 (4c):** mô-hình mới không có user-rút (OwnEpoch tự đẩy LAMP vào ví Phoenix DID mỗi Kỳ active); PHA-1 thoát-sớm vẫn qua anti-idle tự thu-hồi. Không cần redeemer/endpoint riêng.

**Endpoint 2 (`vault/{did}`) — map field validator → JSON:** `phase = (days_elapsed≤1001?1:2)` (1=Giai đoạn Ngày, 2=Giai đoạn Kỳ); `conditional_lamp` (field 3), `owned_out_lamp` (field 4 — luỹ-kế đã vào ví Phoenix), `reclaimed_to_pot_lamp` (field 5); `days_elapsed = ⌊(now−vest_start)/86400⌋`; `current_epoch = p2_epoch(now)`, `last_tick_epoch` (field 8) + `idle_epochs_p2` (field 7, audit); `activity_gate.used_this_period = null` tới khi Registry-team có chuẩn. Field `activity_gate` áp **cả 2 pha** (PHA-1 per-NGÀY, PHA-2 per-KỲ); self-consumption HỢP-LỆ. Mã-lỗi mới nhánh **135x**.

**Nhóm endpoint (chi-tiết request/response ở API-for-SuperApp gốc):**
- **1a getlamp/build** → `{unsigned_tx_cbor, required_signer_key_hash, vault_address, d_lamp, d_oildrop, pot_balance_lamp, vest_start_slot, phase1_days, ttl_slot}`. Precondition 1-DID-1-vault (`VAULT_ALREADY_EXISTS` 1350).
- **2 vault/{did}** → dashboard 2-pha: `phase, days_elapsed, current_epoch, conditional_lamp, owned_out_lamp, reclaimed_to_pot_lamp, magic_*, activity_gate{...}`. `phase`/`conditional_lamp`/`owned_out_lamp`/`idle_epochs_p2`/`last_tick_epoch` tính được NGAY từ validator (không chờ Gen); `magic_*` + `activity_gate.used_this_period` treo tới khi MAGIC/Registry-team nối.
- **3 vault/{did}/magic** → MAGIC-yield ĐỌC-số-dư (`gen_basis_lamp = conditional_lamp`), note "Gen CHỈ ĐỌC — không burn LAMP".
- **5a-c getmagic** → quote (FX buffer) → checkout (VietQR/gateway) → poll tới `CARP_DELIVERED`. `1 CARP = 1 MAGIC`; KHÔNG mint CARP tự-do.
- **7 pot** → `{pot_balance_lamp, current_d_lamp = min(1001,⌊pot/1e6⌋), d_cap, scale, saturated}`.

**Mã-lỗi mới 135x (đội backend thêm `ErrorCode.java`):** `VAULT_ALREADY_EXISTS`(1350), `VAULT_NOT_FOUND`(1351), `POT_UNAVAILABLE`(1352), `ALREADY_IN_PHASE2`(1355), `GETMAGIC_*`(1360-1363). Tái dùng: `UNAUTHORIZED`(1304), `WALLET_NOT_REGISTERED`(1320), `USER_DID_NOT_FOUND`(2002), `SIGNATURE_INVALID`(1403), `CARDANO_TX_FAILED`(5101). *(BỎ `NOT_IN_PHASE2`/`CLAIM_AMOUNT_EXCEEDS_VESTED` — không còn user-claim.)*

**Luồng UI SuperApp (đội giao-diện dựng, mock trước):** Onboarding (keygen vân tay) → GetLAMP (1 nút, banner từ `/pot`) → Vault Dashboard (`/vault/{did}` màn chính, phân-biệt rõ `conditional_lamp` (còn-khoá, sinh MAGIC) vs `owned_out_lamp` (đã vào ví Phoenix, tiêu/bán được); KHÔNG nút rút ở cả 2 pha — LAMP-sở-hữu tự vào ví) → GetMAGIC (5a-c). KHÔNG hiển thị phí ADA (Feecover lo).

---

## 6. Ranh-giới giao-việc

| Tầng | Việc | Đội | Phụ-thuộc-chặn |
|---|---|---|---|
| **On-chain (Aiken)** | **REBUILD** validator vault theo mô-hình mới: **4 redeemer spend** (GenDrip/Reclaim/**OwnEpoch**/**ReclaimEpoch**) + 2 mint-gate, đồng-hồ NGÀY(PHA-1)+EPOCH(PHA-2), OwnEpoch→ví Phoenix DID + ReclaimEpoch per-epoch idle→pot, `q=min(5,c)`, đơn-điệu `last_tick_epoch`, rename field `vested_unlocked→owned_out`, close khi `c=0`. + validator pot (`dist_treasury` kế-thừa). | **đội on-chain** | apply-param builder + `plutus.json` phải khớp code `.ak` mới (9-field datum, 4-redeemer) |
| **Backend (Java)** | GetLAMP orchestration (đọc pot→D→apply-param→build genesis); **anti-idle job NGÀY** (PHA-1, keeper-sig Reclaim); **epoch job PHA-2** (mỗi epoch mới e>te: attest active→OwnEpoch / idle→ReclaimEpoch, keeper-sig); GetMAGIC; keeper wallet. **BỎ** ClaimVested build/submit. `curl` verify sau deploy. | **đội backend** | — |
| **Core / Enclave** | keygen vân tay (Master_KEK); ký GenesisVault (owner-witness); UI 1-nút GetLAMP + dashboard 2-pha (owned_out + số-dư ví Phoenix). **BỎ** ký ClaimVested (không còn user-rút) | **Core** | — |
| **MAGIC/CARP-team** | engine Gen ĐỌC-số-dư (reference-input VaultDatum → drip MAGIC → **KHÔNG spend LAMP, KHÔNG mint token**); `did_commit` per-DID | MAGIC/CARP | kiến-trúc §3.7 — engine phải spell-out on-chain trước khi nối production |
| **Registry-team** | chuẩn danh-mục dịch-vụ-**trả-phí**-tiêu-tài-nguyên-thật + cổng duyệt → keeper attest active/idle (ngưỡng `MIN_MAGIC_CONSUME`) | Registry-team | I-ACT-3 — anti-idle/epoch-gate production cần chuẩn dịch-vụ trước khi bật |
| **CARP-team** | GreenBack settlement interface (backed-path) + shadow-price | CARP | interface-only |
| **LAMP-team** | nạp pot lần-đầu (`dist_treasury`) + `fee_refill_lamp` phản-chu-kỳ + cân-đối dòng-sở-hữu-ra | LAMP team + đội backend | tham-số §6.5 |

**Ranh-giới ký (tóm):** owner-witness = GenesisVault (Enclave); keeper-sig = Reclaim + OwnEpoch + ReclaimEpoch (system-authority MVP, thay bằng Registry consume-event sau). **KHÔNG còn owner-sig cho spend** (OwnEpoch đích cứng = ví Phoenix DID nên keeper không đoạt).

> **Ranh-giới sửa code:** validator (đội on-chain) + backend (đội backend) thuộc PhoenixKey backend — tài-liệu này chỉ đặc-tả, KHÔNG sửa code. Phát-hiện lỗi → báo maintainer / Issue giao đội backend/on-chain.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 7. Thứ-tự deploy + phụ-thuộc-chặn

**Build/deploy được NGAY (không chờ blocker):**
1. Deploy validator pot (`dist_treasury` instance) + nạp-vốn-đầu (LAMP-team). — chặn: 1a, 7.
2. **REBUILD** validator `activation_vault` theo mô-hình mới (4 redeemer). Publish reference-script. Rebuild `plutus.json` khớp.
3. Backend apply-param builder (7 tham-số per-DID → script-hash + address).
4. GetLAMP build/submit (1a/1b) — chỉ cần pot + validator.
5. (Không còn ClaimVested — LAMP-sở-hữu tự vào ví qua OwnEpoch job.)

**Chặn bởi blocker (không bật production tới khi có):**
6. **Anti-idle (Reclaim) + epoch-gate (OwnEpoch/ReclaimEpoch)** — chờ **Registry-team** (I-ACT-3): keeper attest active/idle cần chuẩn dịch-vụ. MVP có thể chạy keeper-thủ-công (attest tay) để test testnet.
7. **Gen production (GenDrip / MAGIC)** — chờ **MAGIC/CARP-team** spell-out engine đọc-số-dư on-chain (§3.7). Trước đó: MAGIC field trả null.
8. **GetMAGIC (5a-c)** — chờ CARP/GreenBack + gateway fiat.
9. **fee_refill_lamp + cân-đối dòng-vest-ra** — chờ LAMP team + đội backend (§6.5, sub-open pot cạn).
10. **GetLAMP-PersonDID production CHẶN** tới khi PA2 UniquenessThread land (lỗ anchor-uniqueness, xem `-Exec.md` §7). Org/Service/Enterprise DID (parent-sig) KHÔNG chặn.

**Đồ-thị chặn:**
```
pot deploy ──┬──► GetLAMP (1a/1b)   [độc-lập, deploy sớm]
validator ───┘        │
                      ▼
   Registry-team ──► anti-idle + epoch-gate (Reclaim/OwnEpoch/ReclaimEpoch) [BLOCKER]
   MAGIC/CARP ─────► Gen/MAGIC (GenDrip)                                    [BLOCKER §3.7]
   CARP/GreenBack ─► GetMAGIC (5a-c)
   PA2 (anchor) ───► GetLAMP-PersonDID production                          [BLOCKER GV1]
```

---

## 8. Test / evidence

**Test mô-hình CŨ (đã có, `aiken check` 2026-07-07 — sẽ THAY khi rebuild):**
```
phoenixkey/activation_logic: total=69  pass=69  fail=0   (mô-hình cũ: vest 1/ngày + ClaimVested + forfeit gap-1001)
```
Bộ test cũ bao-phủ Reclaim/Vest/Claim/GenDrip/Forfeit của mô-hình cũ — **KHÔNG còn hợp-lệ cho OwnEpoch/ReclaimEpoch mới**; rebuild sẽ viết lại.

**CẦN có sau rebuild (bộ test mô-hình mới):**
- **POSITIVE:** P1 Reclaim PHA-1 idle→pot 1/ngày; P2 OwnEpoch active q=5→ví Phoenix DID (`owned_out+=5`, `L(vault)−=5`); P3 ReclaimEpoch idle q=5→pot; P4 GenDrip LAMP-preserved (`L(vault)` bất-biến); P5 OwnEpoch đưa `c→0` → close+burn; P6 ReclaimEpoch đưa `c→0` → close+burn; P7 epoch cuối `q=min(5,c)` rút phần lẻ.
- **NEGATIVE:** OwnEpoch/ReclaimEpoch ở PHA-1 (n≤1001); Reclaim ở PHA-2; xử-lại epoch `e ≤ last_tick_epoch` (double-process); `q>5` (per-epoch cap); `q>c`; OwnEpoch đích ≠ `owner_address` (keeper-đoạt); ReclaimEpoch đích ≠ pot; non-keeper-sig; tự-sinh `c'>c`; drain-ADA; policy-lạ; đổi-DID/mốc; burn khi `c≠0`; `double_sat_two_vault_inputs_rejected`.
- Round-trip CBOR aiken↔rust_core (datum 9-field field-4 = `owned_out`; 4 redeemer idx mới).
- Testnet e2e Preview: GetLAMP genesis → Reclaim (keeper) → OwnEpoch (active) → ReclaimEpoch (idle) → close-burn. Verify `curl` từng endpoint sau deploy.
- Apply-param determinism: cùng DID → cùng script-hash + address.

---

## 9. Luật ràng-buộc khi nối phần phụ-thuộc đội khác

- **§3.7 (MAGIC/CARP):** engine Gen PHẢI dùng **reference-input** (đọc VaultDatum, KHÔNG spend) → drip magic_batches; nếu cần spend → dùng đúng redeemer `GenDrip` (đã ép LAMP-preserved). MAGIC = account-trong-Vault (KHÔNG mint token) — KHÔNG được tái dùng mẫu fire→Treasury (spend/đốt LAMP) của thiết-kế cũ.
- **I-ACT-3 (Registry):** anti-idle/epoch-gate production CHỈ bật sau khi Registry có chuẩn dịch-vụ-**trả-phí**-tiêu-tài-nguyên-thật nối vào `has_counterparty_consume` (ngưỡng `MIN_MAGIC_CONSUME`). Trước đó, MVP dùng keeper attest (system-authority tạm, thay bằng Registry consume-event khi sẵn sàng).
- **GV1 (PA2):** GetLAMP-PersonDID production KHÔNG được mở tới khi PA2 UniquenessThread land (đóng lỗ anchor-uniqueness, KHÔNG phải sybil-sinh-trắc). Org/Service/Enterprise (parent-sig) không bị ràng-buộc này.
- **Thoát-sớm PHA-1:** KHÔNG có redeemer chủ-động — đi qua cơ-chế anti-idle tự thu-hồi 1/ngày.
- **Pot cạn dòng-sở-hữu-ra PHA-2 (§6.5):** LAMP-sở-hữu (OwnEpoch) rời-hệ sang ví Phoenix user vĩnh viễn không quay lại pot — pot PHẢI có nguồn bù phản-chu-kỳ (phí-theo-giá-trị + anti-idle/ReclaimEpoch + treasury-topup) độc lập với dòng sở-hữu-ra. ReclaimEpoch (idle) tái-tuần-hoàn NHANH hơn mô-hình cũ (per-epoch thay gap-1001).

---

## Nguồn

- Code (nguồn chân-lý): `PhoenixKey-Validator/validators/activation_vault.ak`, `lib/phoenixkey/activation_logic.ak`, `lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok`); `plutus.json`.
- Nguồn thiết-kế nội-bộ (không công khai). (đã qua rà-soát nội-bộ)
- Convention backend: `PhoenixKey-Database/.../controller/ActivationController.java`, `WalletController.java`, `common/DataResponse.java`, `security/AuthenticatedUser.java`, `exception/ErrorCode.java`.
- Cross-ref canonical: `PhoenixKey-Specs/PhoenixKey-Math.md` (TAAD §10, recovery §11, fee §36).
- Tài-liệu cùng bộ: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md), [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md), [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).
