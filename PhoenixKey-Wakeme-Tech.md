# PhoenixKey — Wakeme · Đặc-tả KỸ-THUẬT (2-PHA GATED + forfeit)

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
│  ký: GetLAMP-genesis (owner-witness), ClaimVested (owner-witness qua anchor)          │
└───────────────┬──────────────────────────────────────────────────────────────────────┘
                │ build-unsigned → witness → submit
                ▼
┌─────────────────────── BACKEND (PhoenixKey-Database, Java — đội backend) ──────────────┐
│  ActivationController /api/v1/activation/*                                             │
│  · GetLAMP orchestration: đọc pot → D=min(1001,⌊pot/1e6⌋) → build tx genesis          │
│  · Anti-idle job NGÀY (PHA-1): scan vault n≤1001, idle → Reclaim tx (keeper-sig)       │
│  · Vest job PHA-2 (n≥1002): epoch-active → VestToOwner (keeper-sig); gap≥1001ep→Forfeit│
│  · ClaimVested build/submit; GetMAGIC (fiat→CARP)                                      │
│  keeper wallet (system-authority MVP) ký Reclaim / VestToOwner / ForfeitPhase2         │
└───┬───────────────────────────────────────────────────────────────┬───────────────────┘
    │ submit tx                                                       │ reference-input (đọc)
    ▼                                                                 ▼
┌── CARDANO (Plutus V3, Preview) ──────────────────────────────────────────────────────┐
│  validator activation_vault (apply-param per-DID)                                     │
│    mint-gate: GenesisVault | CloseVault  (own_policy ≡ script-hash = vault-NFT policy) │
│    spend: GenDrip | Reclaim | VestToOwner | ClaimVested | ForfeitPhase2                │
│  VAULT UTxO = Script(own_hash) + [vault-NFT(own,owner_commit)=1] + minADA + LAMP khoá  │
│                    ▲ reference-input                    ▲ reference-input               │
│  ┌── taad anchor ──┘ (owner-sig canonical)   ┌── pot (dist_treasury) ──┘ (Reclaim/Forfeit đích)│
│    anchor NFT (anchor_nft_policy, blake2b_256(did)) → controller_pkh ký ClaimVested    │
└──────────────────────────────────────────────────────────────────────────────────────┘
        ▲ ĐỌC-số-dư (reference-input VaultDatum) → drip MAGIC → KHÔNG spend LAMP
┌── MAGIC/CARP engine (repo /CARP — Gen, ngoài validator này) ──────────────────────────┐
│  InstantGen + ScheduleGen đọc (conditional_lamp+vested_unlocked) → magic_batches       │
│  MAGIC = account-trong-Vault (nanogic = MAGIC×10⁹), KHÔNG mint token / KHÔNG policy-id  │
│  [BLOCKER §3.7: chưa spell-out on-chain đọc-datum → drip]                              │
└──────────────────────────────────────────────────────────────────────────────────────┘
┌── Registry-team ── chuẩn dịch-vụ-tiêu-tài-nguyên-thật → keeper attest active [BLOCKER I-ACT-3]┐
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Bất-biến kiến-trúc (load-bearing)

- **1 vault = 1 UTxO** tại `Script(own_hash)`, mang **vault-NFT singleton** `(policy = own_hash, name = owner_commit = did_commit)`, min-ADA, và `(conditional_lamp + vested_unlocked)` LAMP token khoá. Neo: `activation_logic.ak:27-49`, `has_vault_nft` (`:199`), `find_vault_output` (`:205`).
- **Sổ ↔ value đồng-bộ (bất-biến cốt-lõi):** `lamp_token_in_vault == conditional_lamp + vested_unlocked` — mọi redeemer ép lại. Neo: `activation_logic.ak:48`.
- **own_policy ≡ script-hash** (multi-purpose, mẫu `taad`/`state_nft`): vault-NFT policy CHÍNH là hash validator → validator PHẢI có `mint` handler, nếu không mọi tx Mint rơi `else→fail`. Ép ở spend qua `expect Script(own_policy) = own_input.output.address.payment_credential` (`activation_vault.ak:99`); mint handler `activation_vault.ak:86-88`; giải-thích `activation_logic.ak:696-701`.
- **conditional_lamp CHỈ GIẢM** (Reclaim→pot | VestToOwner→vested | ForfeitPhase2→pot); KHÔNG có đường tăng. **vested_unlocked** tăng (VestToOwner) / giảm (ClaimVested→user). Neo: `activation_logic.ak:40-51`.
- **KHÔNG có đường `conditional_lamp → user`** — chỉ `→ pot` (Reclaim/Forfeit) hoặc `→ vested` (VestToOwner). LAMP→user DUY NHẤT qua ClaimVested (phần đã-vest). Neo: `activation_vault.ak:55`.
- **LAMP 36B không-burn:** mọi LAMP rời vault về {pot | ví-owner}, không đích đốt. GenDrip sinh MAGIC không spend/đốt LAMP (I-ACT-7).

### 1.3 Đồng-hồ (2 thang: NGÀY + EPOCH)

| Thang | Hằng (slot) | Dùng cho | Hàm |
|---|---|---|---|
| NGÀY | `slots_per_day = 86_400` | ranh-giới-pha, anti-idle tick, vest mở-khoá 1/ngày, monotonic `last_tick_day` | `days_elapsed(lo, vest_start)` (`:167`) |
| EPOCH | `slots_per_epoch = 432_000` (5 ngày) | gate epoch-active PHA-2, đếm forfeit | `p2_epoch(lo, vest_start)` (`:181`) |

- `n = ⌊(lo − vest_start_slot) / 86_400⌋` — số NGÀY (≥0). `lo` = **lower-bound HỮU-HẠN** của validity-range (`tx_lo`, `:159`); `−∞` → `None` → REJECT (chống khai-man thời-gian).
- `p2_epoch` tính TỪ ĐẦU PHA-2 (ngày 1002): `off = lo − vest_start − 1001×86400`; `e = ⌊off / 432000⌋`. `off < 0` → `-1` (chưa tới PHA-2). Neo: `:181-188`.
- **Ranh-giới pha:** PHA-1 = `n ≤ 1001` (`phase1_last`); PHA-2 = `n ≥ 1002` (`n > phase1_last`). Bất-biến bất-kể D. Neo: `:93-97`.

### 1.4 Hằng-số (nguồn: `activation_logic.ak:81-106`)

| Hằng | Giá-trị | Ý-nghĩa |
|---|---|---|
| `slots_per_day` | 86_400 | 1 NGÀY |
| `grace_days` | 7 | onboarding miễn anti-idle (PHA-1) |
| `reclaim_unit` | 1 | 1 conditional_lamp/NGÀY-idle → pot |
| `d_cap` | 1001 | trần D (= 1001 ngày cam-kết) |
| `phase1_last` | 1001 | ranh-giới PHA-1/PHA-2 |
| `slots_per_epoch` | 432_000 | 1 EPOCH (5 ngày) — gate + forfeit PHA-2 |
| `forfeit_idle_epochs` | 1001 | gap-epoch ≥ ngưỡng → forfeit toàn-bộ conditional |

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
| 3 | `conditional_lamp` | `Int` | `int` | PHA-1 chưa-vest, khoá (oildrop) | `D` (∈ [1, 1001]) |
| 4 | `vested_unlocked` | `Int` | `int` | PHA-2 đã-sở-hữu (oildrop) | `0` |
| 5 | `reclaimed_to_pot` | `Int` | `int` | anti-idle PHA-1 + forfeit PHA-2 luỹ-kế (audit) | `0` |
| 6 | `last_tick_day` | `Int` | `int` | NGÀY tick gần nhất (monotonic) | `0` |
| 7 | `idle_epochs_p2` | `Int` | `int` | **[v4.1]** audit forfeit; reset=0 mỗi ghi (KHÔNG load-bearing) | `0` |
| 8 | `last_tick_epoch` | `Int` | `int` | **[v4.1]** p2-epoch cuối CÓ vest (proven-active); mốc đo forfeit | `0` |

**Đơn-vị:** on-chain LAMP = **oildrop** (`LAMP × 10⁶`). Datum lưu oildrop; API trả cả `lamp` + `oildrop`.

**Diễn-CBOR (Genesis, D=1001 LAMP = 1001000000 oildrop, ví-dụ):**
```
d8799f                                    # Constr 0 (tag 121), 9 field
  581f <owner_commit 32B hex>             # bytes owner_commit
  581f <did_commit 32B hex>               # bytes did_commit
  1a<vest_start_slot>                     # int slot
  1a3b9c3b80                              # int 1001000000 (conditional_lamp oildrop)
  00                                      # int 0 (vested_unlocked)
  00                                      # int 0 (reclaimed_to_pot)
  00                                      # int 0 (last_tick_day)
  00                                      # int 0 (idle_epochs_p2)
  00                                      # int 0 (last_tick_epoch)
ff
```
(Số byte-len prefix minh-hoạ; encoder rust_core tự tính. Điểm load-bearing = **9 field, đúng thứ-tự, đúng khởi-tạo Genesis** để pass `genesis_vault_ok:717`.)

### 2.2 `ActivationRedeemer` (spend) — 5 nhánh

Neo: `activation_logic.ak:124-140`. Tất-cả **không field** (`Constr idx []`):

| idx | Redeemer | CBOR | Ai dùng |
|---|---|---|---|
| 0 | `GenDrip` | `d87980` | engine Gen (spend+recreate, LAMP-preserved) |
| 1 | `Reclaim` | `d87a80` | keeper (anti-idle PHA-1) |
| 2 | `VestToOwner` | `d87b80` | keeper (vest PHA-2 gated) |
| 3 | `ClaimVested` | `d87c80` (Constr 3) | owner (rút vested) |
| 4 | `ForfeitPhase2` | `d87d80` (Constr 4) | keeper (forfeit PHA-2) |

> `plutus.json` v4.1-current export đủ **5 redeemer** (có `ForfeitPhase2` idx 4). rust_core/backend dùng idx trên.

### 2.3 `VaultMintRedeemer` (mint-gate) — 2 nhánh

Neo: `activation_logic.ak:145-153`:

| idx | Redeemer | CBOR | Dùng |
|---|---|---|---|
| 0 | `GenesisVault` | `d87980` | GetLAMP — đúc 1 vault-NFT + ép khuôn output |
| 1 | `CloseVault` | `d87a80` | ClaimVested/ForfeitPhase2 rút-hết-cuối — pure-burn NFT |

### 2.4 Tham-số apply-param (per-DID, đóng lúc GetLAMP)

Neo: `activation_vault.ak:76-84` (khối tham-số validator). Backend apply 7 tham-số → sinh script-hash + address RIÊNG mỗi DID (mẫu did_payment per-DID):

| Param | Type | Nguồn | Ghi-chú |
|---|---|---|---|
| `anchor_nft_policy` | `PolicyId` | hằng TOÀN-HỆ (`taad` Design-2 hash) | đọc controller ký ClaimVested |
| `anchor_nft_name` | `AssetName` | `blake2b_256(did)` | per-DID anchor |
| `lamp_policy` | `PolicyId` | LAMP canonical | LAMP khoá trong vault |
| `lamp_name` | `AssetName` | asset-name LAMP | |
| `pot_address` | `Address` | pot User (instance `dist_treasury`) | Reclaim/Forfeit BUỘC đích về đây |
| `owner_address` | `Address` | **ví Phoenix của DID** (`did_payment`) | ClaimVested BUỘC vested về đây; **DID-bound, rotation-safe** (xoay khoá controller KHÔNG đổi đích vested) |
| `keeper_pkh` | `ByteArray` (pkh) | system-authority (MVP) | ký Reclaim/Vest/Forfeit; **sau thay bằng consume-event Registry ref-input** |

---

## 3. Từng redeemer — điều-kiện + shape tx + ai-ký

Ký-hiệu: `d_in`/`d_out` = datum vào/ra; `lamp_in`/`lamp_out` = LAMP oildrop trong vault vào/ra; `n = days_elapsed`; `e = p2_epoch`.

### 3.0 Mint-gate

#### `GenesisVault` (GetLAMP) — `genesis_vault_ok` (`:717`)
- **Điều-kiện:** (1) mint policy own = **đúng 1 movement, qty +1** (không bundle burn/mint 2); (2) carrier output DUY NHẤT tại `Script(own_policy)` mang đúng 1 NFT(name) + datum well-formed; (3) `name == owner_commit == datum.owner_commit`; (4) khuôn khởi-tạo: `conditional_lamp ∈ [1,1001]`, `vested_unlocked=0`, `reclaimed_to_pot=0`, `last_tick_day=0`, `idle_epochs_p2=0`, `last_tick_epoch=0`, `did_commit ≠ #""`; (5) `lamp_locked == conditional_lamp` (pot cấp LAMP THẬT — chống forge datum bịa D); (6) output chỉ policy ⊆ {ada, lamp, own}.
- **Ký:** owner-witness (Enclave). Pot chi LAMP do **validator POT** tự gác (không thuộc handler này).
- **Shape tx:**
  ```
  in:  [pot UTxO (D LAMP)] + [ví Phoenix (fee/collateral)]
  mint: +1 vault-NFT(own_policy, owner_commit)   redeemer=GenesisVault
  out: [VAULT: Script(own) + NFT + minADA + D-oildrop LAMP + inline datum(9-field Genesis)]
       [pot recreate (giảm D)]  [change]
  signer: owner (did_payment)   validity: [lo, lo+ttl]
  ```

#### `CloseVault` — `close_vault_ok` (`:765`)
- **Điều-kiện:** pure-burn — `≥1 movement` policy own, **mọi movement ÂM** (`p.2nd < 0`). Nối ClaimVested-close hoặc ForfeitPhase2-close.
- **Ký:** đi-kèm redeemer spend đóng (owner hoặc keeper tuỳ nhánh).

### 3.1 `GenDrip` — LAMP-preserved (MAGIC yield) — `gen_drip_ok` (`:646`)
- **Điều-kiện:** (1) LAMP KHÔNG rời: `conditional'==conditional`, `vested'==vested`, `lamp_out==lamp_in==conditional'+vested'`; (2) `reclaimed`/`last_tick_day`/`idle_epochs_p2`/`last_tick_epoch`/DID/mốc **bất-biến** (Gen chỉ ĐỌC); (3) anti-drain (min-ADA giữ + chỉ policy hợp-lệ).
- **Ký:** engine Gen (interface MVP — §3.7 chưa chốt cơ-chế MAGIC). **Điều-kiện tối-thiểu load-bearing = "LAMP không rời qua GenDrip"** (I-ACT-7). Khi §3.7 chốt: thêm điều-kiện đọc/ghi `magic_batches`. **MAGIC = account-trong-Vault (KHÔNG mint token).**
- **Shape:** spend vault + recreate y-hệt (chỉ tầng MAGIC ngoài đổi). **Khuyến-nghị dùng reference-input** cho Gen đọc-số-dư (KHÔNG spend) — xem §5 [CẦN CHỐT §3.7].

### 3.2 `Reclaim` — anti-idle PHA-1 — `reclaim_ok` (`:319`)
- **Điều-kiện (10):** (1) **keeper ký**; (2) PHA-1 `n ≤ 1001`; (3) ngoài grace `n ≥ 7`; (4) monotonic `n > last_tick_day` (chống double-tick); (5) `conditional_lamp ≥ 1`; (6) sổ: `conditional'=conditional−1`, `reclaimed'=reclaimed+1`, `vested'=vested`, `last_tick_day'=n`, epoch-tracking bất-biến, DID/mốc bất-biến; (7) `lamp_out==lamp_in−1==conditional'+vested'`; (8) **đích:** `≥1 LAMP tới pot_address` (LỖ-1 fix — không vào ví keeper); (9) anti-drain.
- **Ký:** keeper (system-authority MVP; `keeper_signed = list.has(self.extra_signatories, keeper_pkh)`, `activation_vault.ak:122`).
- **Shape tx:**
  ```
  in:  [VAULT (lamp_in LAMP)]   redeemer=Reclaim
  out: [VAULT recreate: conditional−1, last_tick_day=n]  [pot: +1 LAMP]  [change]
  signer: keeper   validity.lo hữu-hạn (bắt buộc — days_elapsed)
  ```

### 3.3 `VestToOwner` — vest PHA-2 GATED — `vest_ok` (`:391`)
- **Điều-kiện (9):** (1) PHA-2 `n ≥ 1002`; (2) **keeper ký** (attest epoch-active — MVP, KHÔNG permissionless); (3) monotonic `n > last_tick_day`; (4) `k = vested'−vested ≥ 1` và `k ≤ conditional_lamp`; (5) **trần luỹ-kế** `vested' ≤ n − 1001` (≤1 LAMP/ngày kể từ ngày 1002); (6) sổ LAMP: `conditional'=conditional−k`, `vested'=vested+k`, `reclaimed` bất-biến; (7) sổ EPOCH: `last_tick_epoch'=e` (proven-active), `idle_epochs_p2'=0` (reset), `last_tick_day'=n`, DID/mốc bất-biến; (8) **TỔNG LAMP vault BẤT-BIẾN** `lamp_out==lamp_in==conditional'+vested'` (chỉ đổi SỔ nội-bộ); (9) anti-drain.
- **Ký:** keeper. `k` thường = 1/tx/ngày; batch nhiều ngày ⟹ tăng `k` nhưng vẫn `≤ n−1001`.
- **Shape tx:**
  ```
  in:  [VAULT]   redeemer=VestToOwner
  out: [VAULT recreate: conditional−k, vested+k, last_tick_epoch=e, idle_epochs_p2=0, last_tick_day=n]
  signer: keeper   validity.lo hữu-hạn
  (KHÔNG output LAMP rời — chỉ đổi sổ)
  ```

### 3.4 `ClaimVested` — rút vested → owner — `claim_ok` (`:452`) / `claim_close_ok` (`:502`)
- **2 nhánh** (theo `find_vault_output`):
  - **SỐNG (recreate)** — `claim_ok`: (1) **owner ký** (`anchor_controller_ok` — controller hiện-tại của DID qua anchor); (2) `1 ≤ amount ≤ vested_unlocked`; (3) `conditional_lamp` **BẤT-BIẾN** (không chạm phần chưa-kiếm); (4) `vested'=vested−amount`, reclaimed/tick/epoch/DID/mốc bất-biến; (5) `lamp_out==lamp_in−amount==conditional'+vested'`; (6) **đích:** `≥amount LAMP tới owner_address`; (7) anti-drain. `amount = d_in.vested − d_out.vested` (suy từ datum, `activation_vault.ak:166`).
  - **ĐÓNG (burn)** — `claim_close_ok` + `close_vault_ok`: `conditional_lamp==0` ∧ `amount==vested_unlocked==lamp_in` (rút hết) → KHÔNG recreate + burn vault-NFT.
- **Ký:** **OWNER** (không keeper). Đây là điểm duy-nhất owner-sig canonical qua `auth_logic.anchor_controller_ok` — cần **anchor NFT làm reference-input** + controller_pkh ∈ signatories + anchor `status=Active`.
- **Shape tx:**
  ```
  ref-input: [taad anchor NFT(anchor_nft_policy, blake2b_256(did))]  # đọc controller
  in:  [VAULT]   redeemer=ClaimVested
  out (SỐNG):  [VAULT recreate: vested−amount]  [owner_address: +amount LAMP]  [change]
  out (ĐÓNG):  [owner_address: +vested LAMP]  [change]   + mint: −1 vault-NFT (CloseVault)
  signer: controller_pkh (Enclave)   validity: thường
  ```

### 3.5 `ForfeitPhase2` — thu-hồi PHA-2 — `forfeit_ok` (`:550`) / `forfeit_close_ok` (`:604`)
- **2 nhánh:**
  - **SỐNG (vested>0, recreate)** — `forfeit_ok`: (1) **keeper ký** (MVP attest idle); (2) PHA-2 `n>1001`; (3) **idle gap** `e − last_tick_epoch ≥ 1001` (`forfeit_idle_epochs`); (4) `conditional_lamp ≥ 1`; (5) `conditional'=0` (thu HẾT), `reclaimed'=reclaimed+conditional_in`, `vested'=vested` (BẤT-BIẾN — đã-kiếm), `last_tick_day'=n`, `last_tick_epoch'=e`, `idle_epochs_p2'=0`; (6) `lamp_out==lamp_in−conditional_in==vested'`; (7) **đích:** `≥conditional_in LAMP tới pot`; (8) anti-drain.
  - **ĐÓNG (vested==0, burn)** — `forfeit_close_ok` + `close_vault_ok`: `vested==0`, `conditional≥1==lamp_in` → toàn-bộ về pot + burn NFT.
- **Ký:** keeper.
- **Đo idle = GAP (LAZY, không counter tăng-dần):** `last_tick_epoch` chỉ tiến qua VestToOwner. Genesis=0 ⟹ user CHƯA vest lần nào forfeit khi `p2_epoch ≥ 1001`. **Backend tính idle từ `last_tick_epoch` (gap), KHÔNG dựa `idle_epochs_p2` (chỉ audit).** Neo: guard gap ở `forfeit_ok` (`activation_logic.ak:550-591`), `te`-tiến chỉ ở `vest_ok`.
- **Shape tx:**
  ```
  in:  [VAULT]   redeemer=ForfeitPhase2
  out (SỐNG): [VAULT recreate: conditional=0, vested giữ]  [pot: +conditional_in LAMP]
  out (ĐÓNG): [pot: +conditional_in LAMP]  + mint: −1 vault-NFT (CloseVault)
  signer: keeper   validity.lo hữu-hạn
  ```

### 3.6 Bảng ai-ký + đích LAMP

| Redeemer | Ký | LAMP di-chuyển | Đích ép | Pha |
|---|---|---|---|---|
| GenesisVault | owner | pot → vault (D) | vault (khuôn) | tạo |
| GenDrip | engine (MVP) | KHÔNG | — | cả 2 |
| Reclaim | **keeper** | vault → pot (1) | `pot_address` | PHA-1 (n≤1001) |
| VestToOwner | **keeper** | KHÔNG (đổi sổ) | — | PHA-2 (n≥1002) |
| ClaimVested | **owner** (anchor) | vault → owner (amount) | `owner_address` | PHA-2 |
| ForfeitPhase2 | **keeper** | vault → pot (conditional) | `pot_address` | PHA-2, gap≥1001ep |
| CloseVault | (kèm spend) | burn NFT | — | đóng |

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
  active = M_profile(n) ≥ MIN_MAGIC_TX  (tiêu qua dịch-vụ Registry §3.3a) [BLOCKER I-ACT-3]
  nếu !active ∧ n≥7 ∧ n>last_tick_day ∧ conditional≥1:
    build Reclaim tx (keeper-sig): conditional−1 → pot, last_tick_day=n
Ngày ≥1002: dừng anti-idle → chuyển PHA-2 (4.4).
```

### 4.4 Vest PHA-2 (job — backend, n≥1002)
```
Mỗi NGÀY n≥1002, mỗi vault:
  e = p2_epoch(now)
  nếu epoch_active(e) ∧ conditional≥1 ∧ n>last_tick_day:
    k = min(1, ...) sao cho vested'≤n−1001
    build VestToOwner tx (keeper-sig): conditional−k, vested+k, last_tick_epoch=e, idle=0
  nếu !epoch_active nhiều epoch:
    khi (e − last_tick_epoch ≥ 1001) ∧ conditional≥1:
      build ForfeitPhase2 tx (keeper-sig): conditional→pot (SỐNG nếu vested>0 / ĐÓNG+burn nếu vested==0)
```

### 4.5 ClaimVested (user rút — PHA-2)
```
4a POST /activation/claim-vested/build {amount_lamp, destination}:
   precondition: PHA-2, vested_unlocked>0, amount≤vested
   build unsigned: ref-input taad anchor → spend VAULT ClaimVested → owner_address +amount
   (nhánh ĐÓNG nếu conditional==0 ∧ amount==vested: +CloseVault burn)
Core: Enclave witness (controller)
4b POST /activation/claim-vested/submit → {cardano_tx_hash, CLAIMED}
```

### 4.6 GetMAGIC (fiat→CARP — backend, ngoài validator)
```
5a quote (fiat↔CARP, FX buffer) → 5b checkout (VietQR/gateway) → 5c poll tới CARP_DELIVERED
CARP về account user. KHÔNG mint CARP tự-do; backing qua GreenBack backed-path [CARP-team].
```

### 4.7 Vòng đời tổng
```
GetLAMP → [PHA-1: Gen yield + Reclaim idle→pot] --ngày 1001--> [PHA-2: VestToOwner gated 1/ngày
  → ClaimVested rút owner]  ‖ idle 1001 epoch → ForfeitPhase2 → pot (+burn nếu vested=0)
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
| 4a | POST | `/activation/claim-vested/build` | ClaimVested | auth | validator |
| 4b | POST | `/activation/claim-vested/submit` | (submit) | auth | — |
| 4c | POST | `/activation/abandon-phase1/build` | *(chưa hiện-thực on-chain — xem dưới)* | auth | validator |
| 5a-c | — | `/activation/getmagic/*` | (fiat→CARP) | auth | GreenBack/CARP |
| 7 | GET | `/activation/pot` | (đọc pot) | public | pot deploy |

**Endpoint 2 (`vault/{did}`) — map field validator → JSON:** `phase = (days_elapsed≤1001?1:2)`; `conditional_lamp`, `vested_unlocked` (datum field 3,4); `reclaimed_to_pot_lamp` (field 5); `days_elapsed = ⌊(now−vest_start)/86400⌋`; `activity_gate.idle_epochs_p2` (field 7, chỉ audit — cảnh-báo forfeit tính từ `last_tick_epoch` gap); `activity_gate.used_this_period = null` tới khi Registry-team có chuẩn. Field `activity_gate` áp **cả 2 pha** (thay `anti_idle` chỉ-PHA-1); self-consumption HỢP-LỆ (KHÔNG đòi counterparty≠owner). Mã-lỗi mới nhánh **135x**.

**Nhóm endpoint (chi-tiết request/response ở API-for-SuperApp gốc):**
- **1a getlamp/build** → `{unsigned_tx_cbor, required_signer_key_hash, vault_address, d_lamp, d_oildrop, pot_balance_lamp, vest_start_slot, phase1_days, ttl_slot}`. Precondition 1-DID-1-vault (`VAULT_ALREADY_EXISTS` 1350).
- **2 vault/{did}** → dashboard 2-pha: `phase, days_elapsed, conditional_lamp, vested_unlocked, reclaimed_to_pot_lamp, magic_*, activity_gate{...}`. `phase`/`conditional_lamp`/`vested_unlocked`/`idle_epochs_p2` tính được NGAY từ validator (không chờ Gen); `magic_*` + `activity_gate.used_this_period` treo tới khi MAGIC/Registry-team nối.
- **3 vault/{did}/magic** → MAGIC-yield ĐỌC-số-dư (`gen_basis_lamp = conditional + vested chưa-rút`), note "Gen CHỈ ĐỌC — không burn LAMP".
- **4a claim-vested/build** → `{amount_lamp ≤ vested, destination: wallet|keep_in_vault}`; chỉ PHA-2 (`NOT_IN_PHASE2` 1353). KHÔNG phụ-thuộc Gen/GreenBack.
- **5a-c getmagic** → quote (FX buffer) → checkout (VietQR/gateway) → poll tới `CARP_DELIVERED`. `1 CARP = 1 MAGIC`; KHÔNG mint CARP tự-do.
- **7 pot** → `{pot_balance_lamp, current_d_lamp = min(1001,⌊pot/1e6⌋), d_cap, scale, saturated}`.

**Mã-lỗi mới 135x (đội backend thêm `ErrorCode.java`):** `VAULT_ALREADY_EXISTS`(1350), `VAULT_NOT_FOUND`(1351), `POT_UNAVAILABLE`(1352), `NOT_IN_PHASE2`(1353), `CLAIM_AMOUNT_EXCEEDS_VESTED`(1354), `ALREADY_IN_PHASE2`(1355), `GETMAGIC_*`(1360-1363). Tái dùng: `UNAUTHORIZED`(1304), `WALLET_NOT_REGISTERED`(1320), `USER_DID_NOT_FOUND`(2002), `SIGNATURE_INVALID`(1403), `CARDANO_TX_FAILED`(5101).

**⚠ Lệch spec-vs-code cần đội backend biết:**
- **`abandon-phase1` (4c) CHƯA có redeemer on-chain.** Feat-Math §3.5 + Redeemer-list §10 nêu `AbandonPhase1` (tuỳ-chọn) nhưng code v4.1 **KHÔNG hiện-thực** (comment `activation_vault.ak:55` "BỎ CloseBorrow"; `activation_logic.ak:62-63` "AbandonPhase1 chưa hiện-thực — mặc-định để anti-idle tự thu-hồi"). → Endpoint 4c **không build được tới khi đội on-chain thêm redeemer**, HOẶC bỏ endpoint (anti-idle PHA-1 tự thu-hồi). **Quyết-định: mặc-định BỎ 4c** (đúng code); nếu cần thoát-sớm-tự-nguyện → giao đội on-chain thêm `AbandonPhase1`.
- **Tên path là đề-xuất** — chốt với đội backend trước khi hard-code.

**Luồng UI SuperApp (đội giao-diện dựng, mock trước):** Onboarding (keygen vân tay) → GetLAMP (1 nút, banner từ `/pot`) → Vault Dashboard (`/vault/{did}` màn chính, phân-biệt rõ `conditional_lamp` khoá vs `vested_unlocked` rút-được; PHA-1 KHÔNG nút rút; PHA-2 nút [Rút LAMP]) → GetMAGIC (5a-c). KHÔNG hiển thị phí ADA (Feecover lo).

---

## 6. Ranh-giới giao-việc

| Tầng | Việc | Đội | Phụ-thuộc-chặn |
|---|---|---|---|
| **On-chain (Aiken)** | validator vault 2-pha: 5 redeemer spend + 2 mint-gate, đồng-hồ NGÀY+EPOCH, vest-gated + forfeit + `idle_epochs_p2`, ClaimVested owner-sig. + validator pot (`dist_treasury` kế-thừa). *(AbandonPhase1 nếu cần thoát-sớm)* | **đội on-chain** | apply-param builder + `plutus.json` phải khớp code `.ak` (9-field datum, 5-redeemer) |
| **Backend (Java)** | GetLAMP orchestration (đọc pot→D→apply-param→build genesis); **anti-idle job NGÀY** (PHA-1, keeper-sig Reclaim); **vest job PHA-2** (kiểm epoch-active + đếm gap `last_tick_epoch` → VestToOwner / ForfeitPhase2, keeper-sig); ClaimVested build/submit; GetMAGIC; keeper wallet. `curl` verify sau deploy. | **đội backend** | — |
| **Core / Enclave** | keygen vân tay (Master_KEK); ký GenesisVault (owner-witness) + ClaimVested (controller qua anchor); UI 1-nút GetLAMP + dashboard 2-pha | **Core** | — |
| **MAGIC/CARP-team** | engine Gen ĐỌC-số-dư (reference-input VaultDatum → drip MAGIC → **KHÔNG spend LAMP, KHÔNG mint token**); `did_commit` per-DID | MAGIC/CARP | kiến-trúc §3.7 — engine phải spell-out on-chain trước khi nối production |
| **Registry-team** | chuẩn danh-mục dịch-vụ-tiêu-tài-nguyên-thật + cổng duyệt → keeper attest active/idle | Registry-team | I-ACT-3 — anti-idle/vest-gate production cần chuẩn dịch-vụ trước khi bật |
| **CARP-team** | GreenBack settlement interface (backed-path) + shadow-price | CARP | interface-only |
| **LAMP-team** | nạp pot lần-đầu (`dist_treasury`) + `fee_refill_lamp` phản-chu-kỳ + cân-đối dòng-vest-ra | LAMP team + đội backend | tham-số §6.5 |

**Ranh-giới ký (tóm):** owner-witness = GenesisVault + ClaimVested (qua anchor `anchor_controller_ok`); keeper-sig = Reclaim + VestToOwner + ForfeitPhase2 (system-authority MVP, thay bằng Registry consume-event sau).

> **Ranh-giới sửa code:** validator (đội on-chain) + backend (đội backend) thuộc PhoenixKey backend — tài-liệu này chỉ đặc-tả, KHÔNG sửa code. Phát-hiện lỗi → báo maintainer / Issue giao đội backend/on-chain.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 7. Thứ-tự deploy + phụ-thuộc-chặn

**Build/deploy được NGAY (không chờ blocker):**
1. Deploy validator pot (`dist_treasury` instance) + nạp-vốn-đầu (LAMP-team). — chặn: 1a, 7.
2. Publish reference-script `activation_vault` (hoặc inline per-DID). Rebuild `plutus.json` khớp v4.1.
3. Backend apply-param builder (7 tham-số per-DID → script-hash + address).
4. GetLAMP build/submit (1a/1b) — chỉ cần pot + validator + anchor.
5. ClaimVested (4a/4b) — chỉ cần validator vault + anchor (KHÔNG chờ Gen/GreenBack).

**Chặn bởi blocker (không bật production tới khi có):**
6. **Anti-idle (Reclaim) + vest-gate (VestToOwner) + forfeit** — chờ **Registry-team** (I-ACT-3): keeper attest active/idle cần chuẩn dịch-vụ. MVP có thể chạy keeper-thủ-công (attest tay) để test testnet.
7. **Gen production (GenDrip / MAGIC)** — chờ **MAGIC/CARP-team** spell-out engine đọc-số-dư on-chain (§3.7). Trước đó: MAGIC field trả null.
8. **GetMAGIC (5a-c)** — chờ CARP/GreenBack + gateway fiat.
9. **fee_refill_lamp + cân-đối dòng-vest-ra** — chờ LAMP team + đội backend (§6.5, sub-open pot cạn).
10. **GetLAMP-PersonDID production CHẶN** tới khi PA2 UniquenessThread land (lỗ anchor-uniqueness, xem `-Exec.md` §7). Org/Service/Enterprise DID (parent-sig) KHÔNG chặn.

**Đồ-thị chặn:**
```
pot deploy ──┬──► GetLAMP (1a/1b) ──► ClaimVested (4a/4b)   [độc-lập, deploy sớm]
validator ───┘        │
                      ▼
   Registry-team ──► anti-idle + vest-gate + forfeit (Reclaim/Vest/Forfeit) [BLOCKER]
   MAGIC/CARP ─────► Gen/MAGIC (GenDrip)                                    [BLOCKER §3.7]
   CARP/GreenBack ─► GetMAGIC (5a-c)
   PA2 (anchor) ───► GetLAMP-PersonDID production                          [BLOCKER GV1]
```

---

## 8. Test / evidence

**Đã có (verify THẬT `aiken check` 2026-07-07):**
```
OVERALL: total=173  passed=173  failed=0   (kind: unit 173)
phoenixkey/activation_logic: total=69  pass=69  fail=0
```
Bao-phủ (`activation_logic.ak:788-811` — bảng ca):
- **POSITIVE (6):** P1 reclaim idle→pot; P2 vest 1/ngày (tổng LAMP bất-biến); P3 claim vested→owner; P4 gen_drip LAMP-preserved; P5 reclaim conditional cuối; P6 claim-close vault đóng.
- **NEGATIVE (Reclaim N1-N14):** đích keeper/không-pot, rút>1, trong-grace, double-tick, tick-sai, conditional-cạn, không-keeper-sig, tự-sinh, drain-ADA, policy-lạ, đổi-DID, reclaim-PHA-2, chạm-vested.
- **Vest N15-N20:** vest-PHA-1, vest>1/ngày, đổi-tổng-LAMP, double-tick, k>conditional, chạm-reclaimed.
- **Claim N21-N25:** claim>vested, chạm-conditional, non-owner, sai-đích, đổi-tick.
- **GenDrip N26-N30:** đổi-tổng, rút-LAMP, reset-đồng-hồ, drain-ADA, tăng-vested.
- **+ ca v4.1** (gated-per-epoch + forfeit gap + close-burn nhánh) + `double_sat_two_vault_inputs_rejected` (chống-double-satisfaction).

**CẦN thêm (backend/integration — chưa có):**
- **`plutus.json` đã v4.1-current** (datum 9-field + 5 redeemer, `aiken build` no-diff) — backend/Core dùng trực-tiếp được. (Chỉ cần rebuild nếu code `.ak` đổi tiếp.)
- Round-trip CBOR aiken↔rust_core (encode datum 9-field, thứ-tự đúng; decode redeemer idx).
- Testnet e2e Preview: GetLAMP genesis → Reclaim (keeper) → vest-tick → ClaimVested → forfeit-close. Verify `curl` từng endpoint sau deploy (per working-principle: 1 lệnh curl 30s).
- Apply-param determinism: cùng DID → cùng script-hash + address.

---

## 9. Luật ràng-buộc khi nối phần phụ-thuộc đội khác

- **§3.7 (MAGIC/CARP):** engine Gen PHẢI dùng **reference-input** (đọc VaultDatum, KHÔNG spend) → drip magic_batches; nếu cần spend → dùng đúng redeemer `GenDrip` (đã ép LAMP-preserved). MAGIC = account-trong-Vault (KHÔNG mint token) — KHÔNG được tái dùng mẫu fire→Treasury (spend/đốt LAMP) của thiết-kế cũ.
- **I-ACT-3 (Registry):** anti-idle/vest-gate production CHỈ bật sau khi Registry có chuẩn dịch-vụ-tiêu-tài-nguyên-thật nối vào `has_counterparty_consume`. Trước đó, MVP dùng keeper attest (system-authority tạm, thay bằng Registry consume-event khi sẵn sàng).
- **GV1 (PA2):** GetLAMP-PersonDID production KHÔNG được mở tới khi PA2 UniquenessThread land (đóng lỗ anchor-uniqueness, KHÔNG phải sybil-sinh-trắc). Org/Service/Enterprise (parent-sig) không bị ràng-buộc này.
- **AbandonPhase1:** KHÔNG có redeemer on-chain riêng — thoát-sớm PHA-1 đi qua cơ-chế anti-idle tự thu-hồi (không phải nút chủ-động). Nếu cần thoát-sớm tự-nguyện thật, phải giao đội on-chain thêm redeemer mới, không giả-định endpoint 4c tồn tại.
- **Pot cạn dòng-vest-ra PHA-2 (§6.5):** LAMP-vest rời-hệ sang user vĩnh viễn không quay lại pot qua đường vest — pot PHẢI có nguồn bù phản-chu-kỳ (phí-theo-giá-trị + anti-idle/forfeit + treasury-topup) độc lập với dòng vest-ra.

---

## Nguồn

- Code (nguồn chân-lý): `PhoenixKey-Validator/validators/activation_vault.ak`, `lib/phoenixkey/activation_logic.ak`, `lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok`); `plutus.json`.
- Nguồn thiết-kế nội-bộ (không công khai). (đã qua rà-soát nội-bộ)
- Convention backend: `PhoenixKey-Database/.../controller/ActivationController.java`, `WalletController.java`, `common/DataResponse.java`, `security/AuthenticatedUser.java`, `exception/ErrorCode.java`.
- Cross-ref canonical: `PhoenixKey-Specs/PhoenixKey-Math.md` (TAAD §10, recovery §11, fee §36).
- Tài-liệu cùng bộ: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md), [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md), [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).
