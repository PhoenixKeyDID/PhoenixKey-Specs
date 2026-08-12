# PhoenixKey — Wakeme · Đặc-tả TOÁN hình-thức (cho AUDITOR)

> **Module:** Wakeme (kích-hoạt nhận LAMP). **Loại doc:** Toán hình-thức. **Mô hình:** BẢN A — vest-thành-sở-hữu theo epoch. **Cập nhật:** 2026-07-31.
> **Đối-tượng đọc:** auditor smart-contract + nhà kiểm-toán tokenomic. Đây là đặc-tả TOÁN (định-nghĩa, bất-biến, công-thức, chứng-minh money-safety), KHÔNG phải doc thiết-kế. Thiết-kế/động-cơ ở [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md); kỹ-thuật/API ở [Tech](./PhoenixKey-Wakeme-Tech.md); điều-hành ở [Exec](./PhoenixKey-Wakeme-Exec.md).
>
> **Thuật-ngữ (anh Aladin chốt 2026-07-31):** khoản LAMP khoá-điều-kiện = **"quyền dùng" (usage right)** — user được cấp QUYỀN DÙNG để sinh MAGIC, KHÔNG sở-hữu/định-đoạt, không lãi, không dùng việc khác. **`WakemeUsageRight`** = lượng quyền-dùng cấp lúc genesis (trước gọi "WakemeLent" — đã bỏ). Phần đã kiếm được chuyển sang **`owned` (sở-hữu-hẳn)**.
>
> **Nguồn chân-lý = CODE, không phải văn:** mọi bất-biến neo trực-tiếp `file:hàm:dòng` trong validator. Khi văn ≠ code → **code thắng**. Auditor thấy chênh → báo lỗi CODE hoặc lỗi SPEC, không tự hoà.
>
> **Code neo (nguồn chân-lý):**
> - `PhoenixKey-Validator/lib/phoenixkey/wakeme_logic.ak` — toàn bộ toán + cơ-chế-ép (`*_ok`). Auditor tự chạy `aiken check` (491 checks/0 errors) xác nhận bao-phủ.
> - `PhoenixKey-Validator/validators/wakeme_vault.ak` — thin validator (dispatch 4 spend redeemer + 2 mint redeemer).
> - `PhoenixKey-Validator/lib/phoenixkey/auth_logic.ak` — `anchor_controller_ok` (đồng-thuận owner = 2-of-2 controller+device).
>
> **Phạm-vi money-safety CHỨNG:** LAMP không tự-sinh; quyền-dùng (`conditional`) không rời sang user trái-đường; `owned` (đã-sở-hữu) bất-khả-xâm bởi keeper; forfeit chỉ về pot; OwnEpoch không rời LAMP khỏi vault. **NGOÀI phạm-vi (giả-định tin-cậy §8):** keeper trung-thực; engine Gen sinh MAGIC (đọc-số-dư reference-input, §7-T4); uniqueness PersonDID (§8 T-3); pot-validator gác quyền cấp LAMP.

---

## 1. Ký-hiệu

| Ký-hiệu | Kiểu | Nghĩa | Neo code (`wakeme_logic.ak`) |
|---|---|---|---|
| `D` | ℤ⁺ (oildrop¹) | `WakemeUsageRight` — lượng quyền-dùng cấp genesis; `D = 1001·d_unit` | genesis `:806` (`conditional == wakeme_nights·d_unit`) |
| `d_unit` | ℤ⁺ (oildrop) | nhịp đêm `D/1001`, CỐ-ĐỊNH per-vault, `∈ [1, oil_per_lamp]` | `.d_unit`; genesis `:804-805` |
| `s₀` | slot | `vest_start_slot` — mốc-0 đồng-hồ; genesis ép `== tx_lo` (chống back-date) | `.vest_start_slot`; genesis `:799-802` |
| `lo` | slot | lower-bound HỮU-HẠN của validity-interval tx = "now" | `tx_lo` (None nếu −∞ → REJECT) `:174` |
| `n` | ℤ≥0 | số NGÀY đã trôi = `days_elapsed(lo, s₀)` | `days_elapsed:192` |
| `e` | ℤ≥−1 | epoch tương-đối từ đầu PHA-2 = `p2_epoch(lo, s₀)` | `p2_epoch:205` |
| `c` | ℤ≥0 (oildrop) | `conditional_lamp` — quyền-dùng khoá (chưa-sở-hữu) | `.conditional_lamp` |
| `o` | ℤ≥0 (oildrop) | `owned_lamp` — LAMP SỞ-HỮU-HẲN, ở-lại vault sinh MAGIC / rút / mint CARP | `.owned_lamp` |
| `r` | ℤ≥0 (oildrop) | `reclaimed_to_pot` — luỹ-kế về pot (audit, chỉ tăng) | `.reclaimed_to_pot` |
| `td` | ℤ≥0 | `last_tick_day` — NGÀY tick gần nhất (monotonic) | `.last_tick_day` |
| `te` | ℤ≥−1 | `last_tick_epoch` — epoch cuối CÓ chuyển-sở-hữu (proven-active); genesis −1 | `.last_tick_epoch` |
| `q` | ℤ≥0 (oildrop) | `epoch_release(c, d_unit) = min(5·d_unit, c)` — lượng chuyển `conditional→owned` mỗi epoch active | `epoch_release:215` |
| `L(x)` | ℤ≥0 (oildrop) | lượng LAMP-token thật trong Value `x` | `lamp_in:227` |
| `L(vault)` | ℤ≥0 | `vault_lamp_total(d) = c + o` | `vault_lamp_total:232` |
| primed `x′` | — | giá-trị field/Value ở output tái-tạo (post-state) | `d_out` |

¹ **Đơn-vị on-chain = oildrop** (`1 LAMP = 10⁶ oildrop`, `oil_per_lamp:104`). Mọi field số-lượng (`c, o, r, d_unit, D`) đo bằng **oildrop** (số-nguyên; floor-div, không float). Trần `D ≤ 1001 LAMP` = `wakeme_use_right_cap = 1001·oil_per_lamp` oildrop (`:119`). `d_unit ≤ oil_per_lamp` (≤ 1 LAMP/đêm). Với user sau (pot vơi) `d_unit < oil_per_lamp` ⟹ `D < 1001 LAMP` nhưng LUÔN số-nguyên oildrop và `⋮ 1001` (vì `D = 1001·d_unit`).

**Hằng (`wakeme_logic.ak`):**
```
oil_per_lamp        = 1_000_000     (1 LAMP = 10⁶ oildrop)              :104
epoch_nights        = 5             (1 epoch = 5 đêm; q = 5·d_unit)     :107
grace_days          = 7                                                 :110
phase1_last         = 1001          (PHA-1: n ≤ 1001; PHA-2: n > 1001)  :113
wakeme_nights       = 1001          (D = 1001·d_unit ⟹ D ⋮ 1001)        :116
wakeme_use_right_cap= 1001·10⁶      (trần D = 1001 LAMP, oildrop)        :119
forfeit_epoch_gap   = 1001          (forfeit khi gap epoch ≥ 1001)      :122
slots_per_day, slots_per_epoch = config (86_400 / 432_000)              :95,:100
```

---

## 2. Hàm đồng-hồ (định-nghĩa hình-thức)

**Đ-1 · days_elapsed** (`:192`):
```
days_elapsed(lo, s₀) = ⌊(lo − s₀) / slots_per_day⌋   nếu lo − s₀ ≥ 0
                     = 0                                nếu lo − s₀ < 0   (clamp — chống man-thời-gian âm)
```

**Đ-2 · p2_epoch** (`:205`) — epoch tương-đối từ ĐẦU PHA-2 (ngày 1002 = epoch 0):
```
off = lo − s₀ − 1001·slots_per_day
p2_epoch(lo, s₀) = ⌊off / slots_per_epoch⌋   nếu off ≥ 0
                 = −1                          nếu off < 0   (chưa tới PHA-2 — sentinel)
```
**Bổ-đề Đ-2a:** OwnEpoch/ReclaimEpoch chỉ chạy khi `n > phase1_last` ⟹ `off > 0` ⟹ `e ≥ 0` (sentinel −1 không lọt so-sánh ngưỡng, guard `e_now >= 0` ở `:518,:571,:628`). Chứng-minh: `n > 1001 ⟹ lo − s₀ ≥ 1002·slots_per_day > 1001·slots_per_day ⟹ off > 0`. ∎

**Đ-3 · epoch_release** (`:215`): `q = epoch_release(c, d_unit) = min(epoch_nights·d_unit, c) = min(5·d_unit, c)`.

**Ranh-giới pha (I-ACT-2):** PHA-1 ⟺ `n ≤ 1001`; PHA-2 ⟺ `n > 1001`. Tách rời tuyệt-đối (ép guard mỗi redeemer, §5).

---

## 3. Bất-biến sổ-sách LÕI (money-conservation)

**Bất-biến neo (SỔ-VALUE):** với MỌI vault sống,
```
(SỔ-VALUE)      L(vault) == c + o
```
Ép ở MỌI redeemer giữ-vault-sống qua `lamp_out_amt == vault_lamp_total(d_out)`:
`reclaim_ok:427`, `own_epoch_ok:532`, `reclaim_epoch_ok:587`, `redeem_ok:686`. Genesis lập bất-biến: `genesis_vault_ok:820` ép `lamp_locked == conditional_lamp` với `owned = 0` ⟹ `L = c + 0`.

**Đơn-điệu:**
```
(MONO-c)   c chỉ GIẢM      (Reclaim −d_unit | OwnEpoch −q | ReclaimEpoch → 0);  KHÔNG đường tăng
(MONO-o)   o thay-đổi       (OwnEpoch +q ↑ | Redeem −k ↓); KHÔNG keeper-path nào giảm o
(MONO-r)   r chỉ TĂNG      (Reclaim +d_unit | ReclaimEpoch += c)
(MONO-td)  td strict-tăng  (Reclaim: n > td_in)
(MONO-te)  te strict-tăng qua OwnEpoch (e_now > te_in); ReclaimEpoch set te = e_now
```

**Ghi-chú LAMP tổng-thể (hard-constraint):** LAMP tổng-cung cố-định 36 tỷ, **KHÔNG burn**. Mọi LAMP rời vault đi ĐÚNG một trong hai đích {**pot** (kế-toán Treasury/hồ-chung), **ví-owner**} — KHÔNG có đường đốt. Sinh MAGIC KHÔNG spend/đốt LAMP: engine Gen ĐỌC số-dư qua reference-input, KHÔNG là redeemer của validator này (§7-T4). MAGIC = account-trong-Vault (non-transferable), `nanogic = MAGIC × 10⁹`, KHÔNG mint MAGIC token, KHÔNG policy-id.

> **Không còn redeemer `GenDrip`.** Bản trước có redeemer spend `GenDrip` (identity no-op ép `c′=c ∧ o′=o ∧ L bất-biến`) để "chứng LAMP không rời khi Gen". Đã **gỡ** (anh Aladin chốt 2026-07-31): Gen sinh MAGIC bằng cách ĐỌC vault qua reference-input (read-only), KHÔNG spend UTxO ⟹ một redeemer spend cho việc đó là thừa + bề-mặt griefing (churn UTxO miễn phí). Validator model-A chỉ còn **4 redeemer spend** {Reclaim, OwnEpoch, ReclaimEpoch, Redeem}. Bất-biến "LAMP không rời vault khi tương-tác không-tài-chính" nay do **OwnEpoch** (I-ACT-7, value-invariant) gánh + giả-định Gen-đọc-số-dư (T-4).

---

## 4. Bảng bất-biến I-ACT-1 .. I-ACT-10 (cơ-chế-ép + neo dòng)

| ID | Bất-biến (hình-thức) | Cơ-chế-ép on-chain | Neo `wakeme_logic.ak` |
|---|---|---|---|
| **I-ACT-1** | Genesis: `c = D = 1001·d_unit ∧ d_unit ∈ [1,10⁶] ∧ o = 0 ∧ r = 0 ∧ td = 0 ∧ te = −1 ∧ s₀ = tx_lo ∧ L(vault) = c ∧ did_commit ≠ ∅ ∧ did_commit = owner_commit = name` | mint-gate GenesisVault ép ĐÚNG 1 NFT (name=owner_commit) + khuôn + `lamp_locked == conditional` + **anchor_controller_ok** | `genesis_vault_ok:776` (mệnh-đề `:793-828`) |
| **I-ACT-2** | Ranh-giới pha: Reclaim ⟹ `n ≤ 1001`; OwnEpoch/ReclaimEpoch ⟹ `n > 1001`. Không chồng-lấn. | guard `n <= phase1_last` (Reclaim) ; `n > phase1_last` (còn lại) | `reclaim_ok:408`; `own_epoch_ok:517`; `reclaim_epoch_ok:570` |
| **I-ACT-3** | (Registry-gate) `active` = tiêu qua dịch-vụ Registry, counterparty ≠ owner | **NGOÀI on-chain MVP** — keeper attest (tin-cậy §8). Stub `has_counterparty_consume → False` (KHÔNG dùng như gate — sẽ brick). Blocker **B2**. | `has_counterparty_consume:360` |
| **I-ACT-4** | Reclaim (PHA-1 anti-idle): `c′ = c − d_unit ∧ r′ = r + d_unit ∧ o′ = o ∧ td′ = n ∧ te′ = te`; đúng `d_unit` → pot(tag); grace + monotonic + đủ-số-dư; `c′ ≥ 1` (else đóng) | 14 mệnh-đề (keeper, n≤1001, n≥grace, n>td, c≥d_unit, sổ, c′≥1, sổ↔value, đích-pot-tag, anti-drain) | `reclaim_ok:388` |
| **I-ACT-5** | Conservation toàn-cục: LAMP không tự-sinh; mỗi LAMP rời vault → {pot, ví-owner}; mỗi LAMP vào vault chỉ từ genesis (pot) | hệ-quả (SỔ-VALUE) + mỗi redeemer đích-đúng — §7 Đ-lý 1 | tổng-hợp; đích `lamp_to_addr_tagged:306` |
| **I-ACT-6** | D-cap: `D ≤ 1001 LAMP ∧ D ⋮ 1001 ∧ d_unit ≤ 10⁶`; `c` đơn-điệu-giảm; pha tại ngày 1001 | genesis `:804-810`; (MONO-c) §3 | `genesis_vault_ok:804-810` |
| **I-ACT-7** | OwnEpoch VALUE-preserved (PHA-2 active): `c′ = c − q ∧ o′ = o + q ∧ L(vault′) = L(vault)`; `te′ = e_now`; LAMP KHÔNG rời vault (chỉ đổi sổ nội-bộ conditional→owned) | 13 mệnh-đề: n>1001, e≥0, e>te (once/epoch), c≥1, sổ, `lamp_out == lamp_in` (value bất-biến), sổ↔value, anti-drain | `own_epoch_ok:499` |
| **I-ACT-8** | ReclaimEpoch (PHA-2 FORFEIT): idle `e − te ≥ 1001` ⟹ `c′ = 0 ∧ r′ = r + c ∧ o′ = o` (owned BẤT-BIẾN); toàn-bộ `c` → pot(tag); auth = keeper **HOẶC** owner (escape-hatch) | recreate: 15 mệnh-đề; close: 12 (owned=0 → burn NFT) | `reclaim_epoch_ok:546`; `reclaim_epoch_close_ok:605` |
| **I-ACT-9** | Redeem (đường LAMP `owned`→user DUY-NHẤT): `o′ = o − k ∧ c′ = c` (conditional BẤT-BIẾN), `1 ≤ k ≤ o`, `L(vault′) = L(vault) − k`, ≥ k LAMP → ví owner(tag), **owner ký**, KHÔNG phase-gate | recreate: 13 mệnh-đề; close: 8 (`c = 0 ∧ rút hết o` → burn) | `redeem_ok:655`; `redeem_close_ok:703` |
| **I-ACT-10** | 1-DID-1-vault: `owner_commit == did_commit == name` (MVP single-owner) | genesis `d.did_commit == d.owner_commit` + `name == d.owner_commit` | `genesis_vault_ok:795,818` |

> **Mã `I-ACT-*` giữ tiền-tố cũ** nhưng **ĐÃ remap sang model-A**: I-ACT-7 (trước = GenDrip LAMP-preserved) nay = **OwnEpoch value-invariant**; I-ACT-8 (trước = VestToOwner) + I-ACT-8b (ForfeitPhase2) **gộp** → I-ACT-8 (ReclaimEpoch forfeit) + I-ACT-7 (OwnEpoch chuyển sở-hữu); I-ACT-9 (trước = settlement CARP) nay = **Redeem** (đường owned→owner). Redeemer `VestToOwner`/`ClaimVested`/`ForfeitPhase2`/`GenDrip` của bản v4.1 **KHÔNG tồn tại** trong code model-A.

---

## 5. Mệnh-đề-ép từng redeemer (đối-chiếu code — guard load-bearing)

### 5.1 Reclaim — `reclaim_ok` (anti-idle PHA-1) `:388`
```
keeper_signed
∧ n ≤ 1001                                  -- (I-ACT-2) anti-idle DỪNG ở PHA-2
∧ n ≥ 7                                      -- grace onboarding
∧ n > td_in                                  -- (MONO-td) chống double-tick
∧ c_in ≥ d_unit                              -- còn ≥ 1 nhịp để thu
∧ c′ = c_in − d_unit  ∧  r′ = r_in + d_unit  ∧  o′ = o_in
∧ c′ ≥ 1                                     -- 🔴 nếu c′=0 (c_in=d_unit) → BẮT-BUỘC nhánh ĐÓNG
∧ td′ = n  ∧  te′ = te_in                    -- PHA-1 KHÔNG chạm epoch-tracking
∧ identity_preserved                          -- owner_commit, vest_start, did_commit, d_unit bất-biến
∧ L(out) = L(in) − d_unit  ∧  L(out) = c′ + o′   -- (SỔ-VALUE)
∧ lamp_to_addr_tagged(pot, tag) ≥ d_unit       -- ĐÍCH: pot, gắn payout_tag(own_policy, owner_commit)
∧ nonlamp_preserved ∧ only_expected_policies   -- anti-drain
```
**reclaim_close_ok `:447`:** khi `c_in = d_unit ∧ o_in = 0` ⟹ `c′ = 0` → toàn-bộ conditional → pot(tag) + min-ADA vault → **owner_address** (keeper KHÔNG hốt min-ADA) + `¬recreate` → burn NFT (`close_vault_ok`).

### 5.2 OwnEpoch — `own_epoch_ok` (PHA-2 ACTIVE, chuyển sở-hữu) `:499`
```
keeper_signed
∧ n > 1001  ∧  e_now ≥ 0
∧ e_now > te_in                              -- once-per-epoch, monotonic (reset đếm idle)
∧ c_in ≥ 1
∧ let q = min(5·d_unit, c_in)
∧ c′ = c_in − q  ∧  o′ = o_in + q            -- chuyển conditional→owned
∧ te′ = e_now  ∧  r′ = r_in  ∧  td′ = td_in  -- mốc proven-active
∧ identity_preserved
∧ L(out) = L(in)  ∧  L(out) = c′ + o′          -- 🔴 VALUE BẤT-BIẾN: LAMP KHÔNG rời vault
∧ nonlamp_preserved ∧ only_expected_policies
```
**Chú-ý auditor:** `L(out)=L(in)` — OwnEpoch chỉ flip sổ nội-bộ `conditional→owned`, LAMP ở-lại vault (sinh MAGIC tiếp). LUÔN recreate (vault không rỗng qua OwnEpoch). Đường LAMP→user chỉ mở ở Redeem.

### 5.3 ReclaimEpoch — `reclaim_epoch_ok` / `_close_ok` (PHA-2 FORFEIT) `:546` / `:605`
```
(keeper_signed ∨ owner_signed)               -- escape-hatch: owner tự-làm được nếu keeper vắng
∧ n > 1001  ∧  e_now ≥ 0
∧ e_now − te_in ≥ 1001                        -- idle GAP 1001 epoch LIÊN TỤC (gap < 1001 ⟹ DỒN, rút 0)
∧ c_in ≥ 1  ∧  o_in ≥ 1                       -- recreate (còn owned ⟹ giữ vault sinh MAGIC)
∧ c′ = 0  ∧  o′ = o_in  ∧  r′ = r_in + c_in   -- owned BẤT-BIẾN (đã-kiếm); toàn c → pot
∧ te′ = e_now  ∧  td′ = td_in
∧ L(out) = L(in) − c_in  ∧  L(out) = o_in
∧ lamp_to_addr_tagged(pot, tag) ≥ c_in         -- ĐÍCH pot(tag)
∧ nonlamp_preserved ∧ only_expected_policies
```
**_close_ok `:605`:** khi `o_in = 0` → toàn conditional → pot(tag) + min-ADA → owner_address + `¬recreate` → burn.

### 5.4 Redeem — `redeem_ok` / `_close_ok` (owner rút owned → ví; TUỲ-CHỌN) `:655` / `:703`
```
owner_signed                                  -- controller DID (anchor_controller_ok), KHÔNG keeper. KHÔNG phase-gate.
∧ let k = o_in − o_out  ∧  k ≥ 1  ∧  k ≤ o_in
∧ c′ = c_in                                    -- conditional BẤT-KHẢ-XÂM (không rút phần chưa-kiếm)
∧ r′ = r_in  ∧  td′ = td_in  ∧  te′ = te_in    -- rút owned KHÔNG reset đồng-hồ
∧ identity_preserved
∧ L(vault′) ≥ 1                                -- recreate (còn số dư)
∧ L(out) = L(in) − k  ∧  L(out) = c′ + o′
∧ lamp_to_addr_tagged(owner, tag) ≥ k          -- ĐÍCH ví owner (không pot/keeper)
∧ nonlamp_preserved ∧ only_expected_policies
```
**_close_ok `:703`:** khi `c_in = 0 ∧ rút TOÀN BỘ owned` → owned → owner(tag) + min-ADA → owner + `¬recreate` → burn.

### 5.5 Mint-gate — `genesis_vault_ok` / `close_vault_ok` `:776` / `:840`
- **GenesisVault:** ĐÚNG 1 movement `+1` dưới own_policy; carrier output DUY-NHẤT tại `Script(own_policy)`; `name == owner_commit`; `s₀ == tx_lo` (chống back-date → nhảy PHA-2 né anti-idle, ôm trọn `D` 1 phí — `:799-802`); khuôn I-ACT-1; `lamp_locked == conditional`; `only_expected_policies`; **`anchor_controller_ok`** (owner ký genesis, `:824`).
- **CloseVault:** PURE-BURN — `∀ movement own_policy < 0 ∧ len > 0`. Nối các nhánh close (`¬vault_recreated`). `:840`.

### 5.6 Đích gắn-tag (chống double-satisfaction) `lamp_to_addr_tagged:306`
Chỉ đếm output tới `target` mang `InlineDatum == payout_tag(own_policy, owner_commit)` (28B‖32B). Hai instance khác `own_policy` ⟹ khác tag ⟹ KHÔNG dùng chung output. **RÀNG-BUỘC off-chain:** deposit pot (Reclaim/ReclaimEpoch) VÀ payout owner (Redeem) PHẢI đặt `InlineDatum = payout_tag(...)`.

---

## 6. Không-gian trạng-thái + đồ-thị chuyển (auditor coverage)

```
              Genesis                Reclaim×(n≤1001)        OwnEpoch×(n>1001,active)      Redeem
   [pot] ──────────────► (c=D,o=0) ──────────────────────► (c↓,o↑) ─────────────────────► (o↓)
                             │                                 │   ┌─ ReclaimEpoch (e−te≥1001): c→0 →pot
                             └─────────────────────────────────┘   └─ Redeem-close (c=0∧rút hết o): burn NFT
```
Chuyển hợp-lệ (đầy-đủ 4 spend + 2 mint):
- `Reclaim` (PHA-1): `(c,o) → (c−d_unit, o)`, `d_unit` → **pot**; close khi `c=d_unit∧o=0` → burn.
- `OwnEpoch` (PHA-2 active): `(c,o) → (c−q, o+q)`, **L bất-biến**, `q = min(5·d_unit, c)`.
- `ReclaimEpoch` (PHA-2 idle-gap≥1001): `(c,o) → (0, o)`, `c` → **pot**; close khi `o=0` → burn.
- `Redeem` (owner, mọi lúc): `(c,o) → (c, o−k)`, `k` → **ví owner**; close khi `c=0∧k=o` → burn.

Chuyển BỊ CẤM (auditor xác nhận REJECT — test âm trong code, 491 checks tổng):
`c → user` trực-tiếp (chỉ pot|qua-owned) · `o` giảm không-owner (keeper rút owned) · L tự-sinh · Reclaim khi n>1001 · OwnEpoch khi n≤1001 · OwnEpoch chuyển > q · OwnEpoch rút LAMP khỏi vault · ReclaimEpoch khi gap<1001 · Redeem k>owned · Redeem chạm conditional · back-date s₀ · genesis không-owner-ký · đổi DID/d_unit · drain ADA/token-lạ · double-tick.

---

## 7. Ba định-lý MONEY-SAFETY (+ chứng-minh phác-thảo)

### Định-lý 1 (NO-DRAIN / conservation) — LAMP không tự-sinh, không rò-rỉ khỏi đường hợp-lệ.
> **Phát-biểu.** Mọi tx spend hợp-lệ trên vault sống bảo-toàn tổng LAMP: mỗi LAMP rời vault đi ĐÚNG {pot, ví-owner}; mỗi LAMP vào vault chỉ từ Genesis (pot). Không redeemer nào tạo LAMP.

**Chứng-minh (quy-nạp theo redeemer).**
1. *Genesis lập (SỔ-VALUE):* `:820` ép `L = c` với `o=0` ⟹ `L = c+o`. LAMP vào = `D` từ pot (pot-validator gác, §8 T-2).
2. *Mỗi redeemer giữ (SỔ-VALUE):* `L(out) = c′+o′` (`:427,:532,:587,:686`) ⟹ bất-biến bảo-toàn qua mọi bước.
3. *Δ LAMP rời = đích-đúng:*
   - Reclaim: `L(out)=L(in)−d_unit` **∧** `lamp_to_addr_tagged(pot) ≥ d_unit` ⟹ đúng `d_unit` tới pot.
   - ReclaimEpoch: `L(out)=L(in)−c` **∧** `pot(tag) ≥ c` ⟹ `c` tới pot.
   - Redeem: `L(out)=L(in)−k` **∧** `owner(tag) ≥ k` ⟹ `k` tới ví owner.
   - OwnEpoch: `L(out)=L(in)` ⟹ 0 LAMP rời (chỉ đổi sổ).
4. *Không tự-sinh:* không mệnh-đề nào cho `c′>c` hay `L(out)>L(in)`. `c` chỉ giảm (MONO-c); `o` tăng chỉ qua OwnEpoch với `o′=o+q ∧ c′=c−q` (bù-trừ, `c+o` bất-biến). ∎

**Điểm auditor kiểm:** `lamp_to_addr_tagged` chỉ cộng output tới ĐÚNG địa-chỉ + ĐÚNG tag ⟹ `≥ threshold` + `only_expected_policies` chặn "keeper nhét LAMP vào output thứ-ba / instance khác". Reclaim/ReclaimEpoch đích = `pot_address`; Redeem đích = `owner_address` (apply-param, neo theo DID).

### Định-lý 2 (OWNED-UNTOUCHABLE) — LAMP đã-sở-hữu bất-khả-xâm bởi keeper/anti-idle/forfeit.
> **Phát-biểu.** `owned_lamp` chỉ giảm qua Redeem (owner ký) tới ví owner. Không redeemer keeper-gated (Reclaim, OwnEpoch, ReclaimEpoch) làm giảm `o`.

**Chứng-minh.**
- Reclaim: `o′ = o` (`:420`). ⟹ bất-biến.
- OwnEpoch: `o′ = o + q, q ≥ 0` (`:525`) ⟹ chỉ TĂNG.
- ReclaimEpoch: `o′ = o` (`:580`; nhánh close chỉ chạy khi `o=0`) ⟹ bất-biến; owned không bị forfeit.
- Redeem: `o′ = o − k, k ≤ o`, đích **ví owner**, guard **owner_signed** (`anchor_controller_ok`). Keeper KHÔNG ký được.
Vậy giảm-`o` ⟺ Redeem ⟺ owner ký ⟹ user toàn-quyền phần đã-sở-hữu. ∎

**Hệ-quả chủ-quyền (nối directive "keeper mất khoá KHÔNG chặn dùng DID"):** Redeem **không phase-gate + không keeper** ⟹ dù keeper biến-mất, owner LUÔN rút được `owned`. Chỉ đường **conditional→owned** (OwnEpoch) còn phụ-thuộc keeper-attest (MVP) — đóng hẳn phụ-thuộc này = **B2** (consume-event Registry thay keeper, §9). ReclaimEpoch có nhánh `owner_signed` = escape-hatch (owner tự đóng vault khi idle-gap đủ, không cần keeper).

### Định-lý 3 (FORFEIT-ONLY-TO-POT) — thu-hồi PHA-2 chỉ về pot, chỉ phần chưa-kiếm, chỉ khi idle đủ.
> **Phát-biểu.** ReclaimEpoch chỉ hợp-lệ khi `n > 1001 ∧ e − te ≥ 1001`; hệ-quả `c → 0` toàn-bộ tới **pot**, `o` bất-biến, `r += c`.

**Chứng-minh.**
- *Idle load-bearing = GAP:* guard `e − te_in ≥ 1001` (`:573`). `te` CHỈ tiến qua OwnEpoch (`:526` set `te′ = e_now`). User active mỗi epoch → OwnEpoch cập-nhật `te` sát `e` ⟹ gap < 1001 ⟹ forfeit REJECT. Idle ≥ 1001 epoch liên-tục → `te` tụt ⟹ gap ≥ 1001 ⟹ cho phép.
- *Chỉ về pot:* `L(out)=L(in)−c` **∧** `pot(tag) ≥ c` ⟹ `c` tới pot (Đ-lý 1).
- *Chỉ phần chưa-kiếm:* `o′ = o` (Đ-lý 2).
- *Không keeper-đoạt:* đích cứng `pot_address`; `only_expected_policies` chặn token-lạ. Keeper/owner ký để attest idle nhưng KHÔNG chọn được đích.
∎

**Chú-ý an-toàn:** genesis `te = −1`. User CHƯA OwnEpoch lần nào ⟹ `te = −1` ⟹ forfeit cho phép khi `e ≥ 1000` (`e − (−1) ≥ 1001`). Hành-vi ĐÚNG: user vào PHA-2 mà không bao-giờ active → phần chưa-kiếm về pot sau ~1001 epoch. Ngưỡng `≥` + `te` chỉ tiến khi active thật ⟹ không off-by-one có-lợi-attacker.

---

## 8. Giả-định tin-cậy (NGOÀI phạm-vi chứng-minh)

| # | Giả-định | Rủi-ro nếu vỡ | Trạng-thái |
|---|---|---|---|
| T-1 | **Keeper trung-thực** attest active (OwnEpoch) / idle (Reclaim, ReclaimEpoch). MVP `keeper_signed`. | Keeper gian → Reclaim/ReclaimEpoch oan phần chưa-kiếm (về **pot**, KHÔNG về keeper — Đ-lý 1/3), hoặc OwnEpoch sai epoch. KHÔNG drain sang keeper. Owner escape-hatch (ReclaimEpoch owner_signed) + Redeem owner-only ⟹ keeper KHÔNG khoá được owned/DID. | MVP tin keeper; production thay bằng consume-event Registry (B2, §9) |
| T-2 | **Pot-validator** (`dist_treasury`) tự gác cấp `D` đúng. `genesis_vault_ok` gác VAULT đúng khuôn + `lamp_locked == conditional`, KHÔNG gác pot chi đúng. | Pot chi sai `D` → vault khai-man; nhưng `lamp_locked==conditional` ràng datum↔LAMP thật ⟹ không bịa `D` mà không khoá LAMP. | pot-validator riêng |
| T-3 | **Uniqueness anchor PersonDID.** `D` keyed per-PersonDID. Lỗ ở tầng **mã-hoá anchor** (KHÔNG phải sinh-trắc): `GenesisPerson` đúc anchor did-string bất-kỳ với controller attacker (HW_Key P-256 KHÔNG verify on-chain). | N anchor-giả → N×`D` rút khỏi pot (GV1). Đóng ở tầng structural/cryptographic (PA-2 UniquenessThread + person-level), KHÔNG phải chống-sybil-sinh-trắc. Org/Service DID (parent-sig) KHÔNG dính. | **[GATE production]** §9; NGOÀI phạm-vi vault |
| T-4 | **Engine Gen ĐỌC-số-dư.** Gen sinh MAGIC bằng đọc `WakemeVaultDatum` qua reference-input, KHÔNG spend UTxO-LAMP, KHÔNG mint token. Validator KHÔNG còn redeemer `GenDrip` (đã gỡ). | Nếu engine Gen production spend UTxO-LAMP (như code MAGIC cũ InstantGen/ScheduleGen còn nhánh chuyển-LAMP) → rút LAMP thật, vỡ conservation. **CẤM nối Wakeme vào cửa Gen tới khi MAGIC pha-2 (read-only) xong.** | **[CẦN CHỐT]** §9 — B1 |
| T-5 | **`has_counterparty_consume`** (`:360`) stub `False`. Cổng Registry-gate (I-ACT-3) PHẢI nối consume-event Registry-bonded (owner_commit, counterparty_did ≠ owner, epoch) trước khi anti-idle/epoch-gate dựa vào nó thay keeper. | anti-idle không phân-biệt "tiêu-thật qua Registry" on-chain nếu chưa nối; dựa keeper. | **[CẦN CHỐT]** §9 — B2, Registry-team |

**Kết-luận phạm-vi:** trong {T-1..T-5}, ba định-lý §7 GIỮ. **Kể cả keeper ác-ý, LAMP không chảy sang keeper** — mọi đích thu-hồi cứng = pot; mọi đích rút cứng = ví owner (owner ký). T-3 là lỗ ANCHOR-uniqueness (mã-hoá), KHÔNG phải sybil-sinh-trắc.

---

## 9. Nguồn nạp pot (Feecover surplus) — kế-toán + điều-kiện bền-vững

> Anh Aladin chốt 2026-07-31: **nguồn CHỦ-YẾU làm pot tăng = thặng-dư LAMP từ Feecover.** Mục này đặc-tả **kế-toán dòng** + **điều-kiện cân-đối**; cơ-chế mua-lại/redeem cụ-thể (chọn giá, khớp lệnh) thuộc **Feecover/LAMP** (Wakeme là bên NHẬN nguồn) — KHÔNG đặc-tả ở đây.

**Kế-toán pot `P` (oildrop).** Ký `Σ` trên tập vault + khoảng thời-gian:
```
P(t+1) = P(t)
        − Σ D              (Genesis: quyền-dùng rời pot vào vault)
        + Σ d_unit         (Reclaim: idle PHA-1 hoàn pot)
        + Σ c              (ReclaimEpoch: forfeit hoàn pot)
        + F(t)             (Feecover surplus nạp vào — nguồn CHỦ-YẾU)
```
`owned` rời-hệ qua **Redeem** (`k` → ví owner) là LAMP đã-kiếm hợp-lệ rời khỏi vòng pot — **đây là "chi-phí" chương-trình-trung-thành** mà `F(t)` phải bù.

**Dòng Feecover `F(t)`.** Mỗi giao-dịch user tiêu `m` MAGIC, hệ thu **phí cố-định bằng CARP** `= φ·m` (`φ` = suất-phí CARP/MAGIC, tham-số Feecover). Khoản CARP đó chuyển thành LAMP nạp pot qua một trong hai đường (điều-kiện, do Feecover chọn):
```
F(t) = Σ_giao-dịch  ΔL_pot,   với  ΔL_pot = buyback  khi  p_LAMP ≤ p*   (mua LAMP thị-trường giá thấp)
                                          = redeem   từ GreenCheck       (đổi ra LAMP khi phù-hợp)
```
- **Buyback** phản-chu-kỳ: chỉ mua khi giá LAMP ≤ ngưỡng `p*` ⟹ vừa nạp pot vừa đỡ giá (không đua giá đỉnh). `ΔL_pot ≈ (φ·m)/p_LAMP` (đơn-vị LAMP thu về / CARP chi).
- **Redeem GreenCheck:** đổi CARP↔LAMP qua dự-trữ đối-ứng khi thị-trường không thuận buyback.
- **KHÔNG in thêm LAMP:** tổng-cung 36 tỷ cố-định; `F(t)` chỉ **tái-phân-phối** LAMP đã tồn-tại về pot (mua lại / đổi từ dự-trữ), KHÔNG mint.

**Điều-kiện bền-vững (R1 — chống pot cạn).** Trên cửa-sổ thời-gian `W`:
```
(BỀN-VỮNG)   E[ F(t) + Σ d_unit + Σ c ]  ≥  E[ Σ D_genesis-mới + Σ k_redeem ]
```
tức tốc-độ-nạp (Feecover + idle-reclaim + forfeit) ≥ tốc-độ-rời (cấp mới cho onboarding + owned redeem ra ví). Vi-phạm kéo-dài ⟹ pot cạn ⟹ `D` cho user mới giảm dần (theo `D = min(1001 LAMP, ⌊remaining_pot × 1001/10⁹⌋)`, tự điều-tiết). **[CẦN MÔ-PHỎNG]** hiệu-chuẩn `φ, p*` vs tốc-độ onboarding + tỷ-lệ qua-PHA-2 — thuộc Feecover/LAMP + tokenomics.

> **Ghi-chú tự-điều-tiết:** công-thức `D` neo pot còn-lại (`1001 phần-tỷ pot`, trần 1001 LAMP) là **van an-toàn nội-tại**: pot vơi ⟹ `D` (và `d_unit`) tự giảm ⟹ chi onboarding tự thắt. `F(t)` quyết-định pot có **phục-hồi** để `D` không tụt về 0 hay không.

---

## 10. Danh-mục kiểm cho auditor (checklist rút gọn)

1. **(SỔ-VALUE)** `L(vault) == c + o` ở genesis + mọi redeemer? → grep `lamp_out_amt == vault_lamp_total(d_out)` (`reclaim_ok`, `own_epoch_ok`, `reclaim_epoch_ok`, `redeem_ok`).
2. **Đích-đúng + gắn-tag** mọi đường LAMP-rời: Reclaim/ReclaimEpoch → `pot_address`; Redeem → `owner_address`; OwnEpoch → không rời. → `lamp_to_addr_tagged(...) ≥ threshold` + `only_expected_policies`.
3. **Chữ-ký tách rời:** Redeem = owner (`anchor_controller_ok` = 2-of-2 controller+device); Reclaim/OwnEpoch = keeper; ReclaimEpoch = keeper ∨ owner (escape-hatch). Không nhánh nào cho keeper rút owned.
4. **Ranh-giới pha:** Reclaim `n ≤ 1001`; OwnEpoch/ReclaimEpoch `n > 1001`.
5. **OwnEpoch VALUE bất-biến** `L(out)=L(in)` (LAMP không rời vault) + `q = min(5·d_unit, c)` (không chuyển quá).
6. **ReclaimEpoch gap** `e − te ≥ 1001` load-bearing; `te` chỉ tiến qua OwnEpoch.
7. **Redeem** `1 ≤ k ≤ owned` + conditional bất-biến (`c′ = c`) + không phase-gate.
8. **Genesis khuôn:** `s₀ == tx_lo` (chống back-date) + `D = 1001·d_unit` (`⋮1001`, `≤ cap`) + `owned=0` + `te=−1` + `did_commit==owner_commit==name` + `anchor_controller_ok`.
9. **Monotonic** `td` (Reclaim), `te` (OwnEpoch once/epoch). Anti-drain `nonlamp_preserved` + `only_expected_policies` mọi redeemer.
10. **Close/burn** đúng: `*_close_ok` chỉ khi vault thực-rỗng (Reclaim: `c=d_unit∧o=0`; ReclaimEpoch: `o=0`; Redeem: `c=0∧k=o`) nối `close_vault_ok` pure-burn.
11. **KHÔNG còn `GenDrip`/`VestToOwner`/`ClaimVested`/`ForfeitPhase2`** trong code — grep phải rỗng (model v4.1 đã bỏ).
12. **Man-thời-gian:** `tx_lo` None (−∞) → REJECT; `days_elapsed`/`p2_epoch` clamp.

---

## Phụ-lục A — Bảng đối-chiếu bất-biến ↔ dòng code (`wakeme_logic.ak`)

| Bất-biến | Neo |
|---|---|
| days_elapsed / p2_epoch / epoch_release | `:192` / `:205` / `:215` |
| (SỔ-VALUE) L=c+o (`vault_lamp_total`) | `:232`; genesis `:820`; reclaim `:427`; ownEpoch `:532`; reclaimEpoch `:587`; redeem `:686` |
| I-ACT-1 genesis khuôn + anchor_controller_ok | `:793-828` |
| I-ACT-4 Reclaim | `:388-440` (close `:447-489`) |
| I-ACT-7 OwnEpoch value-invariant | `:499-537` |
| I-ACT-8 ReclaimEpoch forfeit | `:546-599` (close `:605-646`) |
| I-ACT-9 Redeem owned→owner | `:655-697` (close `:703-735`) |
| I-ACT-10 1-DID-1-vault | `:795,:818` |
| mint-gate genesis/close | `:776-836` / `:840-846` |
| dispatch spend/mint | `wakeme_vault.ak` (thin) ; `validate_vault_mint:849` |

---

## Nguồn

- Code (nguồn chân-lý): `PhoenixKey-Validator/lib/phoenixkey/wakeme_logic.ak` (~2852 dòng, 491 checks), `validators/wakeme_vault.ak`, `lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok`).
- Mô hình BẢN A: Issue #67 (anh Aladin chốt 2026-07-30); gỡ GenDrip + thuật-ngữ usage-right (2026-07-31).
- Nguồn thiết-kế nội-bộ (không công khai).
- Tài-liệu cùng bộ: [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md), [Tech](./PhoenixKey-Wakeme-Tech.md), [Exec](./PhoenixKey-Wakeme-Exec.md).

→ Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
