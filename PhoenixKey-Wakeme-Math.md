# PhoenixKey — Wakeme · Đặc tả TOÁN hình thức (cho AUDITOR)

> **Module:** Wakeme (GetLAMP — kích hoạt nhận LAMP). **Loại doc:** Toán hình thức. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** auditor smart-contract + nhà kiểm toán tokenomic. Đây là đặc tả TOÁN (định nghĩa, bất biến, công thức, chứng minh money-safety), KHÔNG phải doc thiết kế. Thiết kế/động cơ ở [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md); kỹ thuật ở [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md); điều hành ở [PhoenixKey-Wakeme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Wakeme-Specs/blob/main/PhoenixKey-Wakeme-Exec.md).
>
> **Nguồn chân lý = CODE, không phải văn:** mọi bất biến ở đây neo trực tiếp về `file:hàm` trong validator. Khi văn ≠ code → **code thắng**. Nếu auditor thấy chênh, báo lỗi CODE hoặc lỗi SPEC, không tự hoà.
>
> **Code neo (nguồn chân lý):**
> - `PhoenixKey-Validator/lib/phoenixkey/activation_logic.ak` — toàn bộ toán + cơ chế ép (`*_ok`). Auditor tự chạy `aiken check` để xác nhận bao phủ test.
> - `PhoenixKey-Validator/validators/activation_vault.ak` — thin validator (dispatch 5 spend redeemer + 2 mint redeemer).
>
> **Phạm vi money-safety đặc tả này CHỨNG:** LAMP không tự sinh; conditional-LAMP không rời sang user; vested-LAMP đã sở hữu bất khả xâm phạm; forfeit chỉ về pot. **NGOÀI phạm vi (giả định tin cậy, ghi rõ §8):** keeper trung thực (attest active/idle đúng), engine Gen `§3.7` chưa spell-out on-chain, uniqueness PersonDID (sinh trắc Enclave — xem T-3 + §8), pot-validator (`dist_treasury`) tự gác quyền cấp LAMP.

---

## 1. Ký hiệu

| Ký hiệu | Kiểu | Nghĩa | Neo code |
|---|---|---|---|
| `D` | ℤ⁺ | lượng LAMP cấp mỗi GetLAMP; `D = min(1001, ⌊pot/10⁶⌋)` | `d_cap = 1001`, `genesis_vault_ok` (1 ≤ conditional ≤ d_cap) |
| `s₀` | slot | `vest_start_slot` — mốc-0 đồng hồ (slot lúc GetLAMP) | `ActivationVaultDatum.vest_start_slot` |
| `lo` | slot | lower-bound HỮU HẠN của validity-interval tx = "now" | `tx_lo` (None nếu −∞ → REJECT) |
| `n` | ℤ≥0 | số NGÀY đã trôi = `days_elapsed(lo, s₀)` | `days_elapsed` |
| `e` | ℤ≥−1 | p2-EPOCH tương đối từ đầu PHA-2 = `p2_epoch(lo, s₀)` | `p2_epoch` |
| `c` | ℤ≥0 (oildrop¹) | `conditional_lamp` — LAMP khoá có điều kiện (chưa-vest) | `.conditional_lamp` |
| `v` | ℤ≥0 | `vested_unlocked` — LAMP đã sở hữu (rút được) | `.vested_unlocked` |
| `r` | ℤ≥0 | `reclaimed_to_pot` — luỹ kế về pot (audit, chỉ tăng) | `.reclaimed_to_pot` |
| `td` | ℤ≥0 | `last_tick_day` — NGÀY tick gần nhất (monotonic) | `.last_tick_day` |
| `ie` | ℤ≥0 | `idle_epochs_p2` — AUDIT-field (KHÔNG load-bearing) | `.idle_epochs_p2` |
| `te` | ℤ≥0 | `last_tick_epoch` — p2-epoch cuối CÓ vest (proven-active) | `.last_tick_epoch` |
| `L(x)` | ℤ≥0 | lượng LAMP-token thật trong Value `x` = `quantity_of(x, lamp_policy, lamp_name)` | `lamp_in` |
| primed `x′` | — | giá trị field/Value ở output tái tạo (post-state) | `d_out`, `out_v` |

¹ Đơn vị on-chain là **oildrop** (`LAMP × 10⁶`, spec §3.1); mọi số học integer floor-div, không float. Đặc tả này viết "LAMP" cho gọn — nhân 10⁶ để ra oildrop.

> **🔴 [CẦN CHỐT — rigor gap thật, gắn 2026-07-12]** Quy ước quy đổi đơn vị (footnote ¹) KHÔNG
> được áp dụng nhất quán tại chính bất biến lõi nhất của module (I-ACT-1, §4). Cụ thể:
> - `D` được định nghĩa `D = min(1001, ⌊pot/10⁶⌋)` — công thức này TỰ nó đã chia `pot` cho 10⁶,
>   tức **`D` là số LAMP** (tối đa 1001 LAMP, đơn vị người dùng nhìn thấy).
> - Nhưng I-ACT-1 (§4) viết thẳng `c = D` và `L(vault) = D`, trong khi `c` và `L(x)` đã khai kiểu
>   ở bảng trên là **oildrop** (theo footnote ¹) — tức đẳng thức này gán một đại lượng LAMP (`D`)
>   cho một đại lượng oildrop (`c`, `L`) mà KHÔNG có bước nhân 10⁶ nào được viết ra.
> - Cột "Neo code" của `d_cap` ghi thẳng `d_cap = 1001` là hằng số Aiken, so sánh trực tiếp với
>   `conditional_lamp` (trường on-chain, tự nhiên ở đơn vị oildrop theo mọi neo code khác trong
>   module này).
>
> Từ văn bản spec đơn thuần, auditor **không thể xác định** `d_cap = 1001` là 1001 oildrop
> (≈0.001 LAMP — vô lý với một chương trình quảng cáo "tới 1001 LAMP") hay đã ngầm hiểu là
> `1001 × 10⁶` oildrop. Đây KHÔNG phải lỗi biên tập nhỏ — nó nằm ngay ở bất biến SỔ-VALUE cốt lõi
> nhất (I-ACT-1). **Cần xác nhận với code thật** (`activation_logic.ak`, hằng `d_cap` và biểu thức
> `genesis_vault_ok`) xem `d_cap` trong Aiken source thực sự là `1001` hay `1001_000000`, rồi sửa
> văn bản spec khớp — KHÔNG tự suy đoán ở đây vì đây là quyết định ảnh hưởng an toàn tiền thật,
> không phải chỉ làm rõ câu chữ.

**Hằng (code `activation_logic.ak` §HẰNG):**
```
slots_per_day       = 86_400        (1 slot = 1 giây)
slots_per_epoch     = 432_000       (1 epoch Cardano = 5 ngày)
grace_days          = 7
reclaim_unit        = 1
d_cap               = 1001
phase1_last         = 1001          (PHA-1: n ≤ 1001;  PHA-2: n ≥ 1002)
forfeit_idle_epochs = 1001
```

---

## 2. Hàm đồng hồ (định nghĩa hình thức)

**Đ-1 · days_elapsed** (`activation_logic.ak:167`):
```
days_elapsed(lo, s₀) = ⌊(lo − s₀) / 86_400⌋   nếu lo − s₀ ≥ 0
                     = 0                         nếu lo − s₀ < 0     (clamp — chống man-thời-gian âm)
```

**Đ-2 · p2_epoch** (`activation_logic.ak:181`) — epoch tương đối tính từ ĐẦU PHA-2 (ngày 1002 = epoch 0):
```
off = lo − s₀ − 1001 × 86_400          (slot đã trôi kể từ mốc PHA-2)
p2_epoch(lo, s₀) = ⌊off / 432_000⌋      nếu off ≥ 0
                 = −1                     nếu off < 0    (chưa tới PHA-2 — sentinel)
```
Bổ đề Đ-2a: forfeit/vest chỉ gọi khi `n > phase1_last` ⟹ `off > 0` ⟹ `e ≥ 0` (sentinel −1 không lọt vào so sánh ngưỡng). Chứng minh: `n > 1001 ⟹ lo − s₀ ≥ 1002·86400 > 1001·86400 ⟹ off > 0`. ∎

**Ranh giới pha (I-ACT-2):** PHA-1 ⟺ `n ≤ 1001`; PHA-2 ⟺ `n ≥ 1002`. Tách rời tuyệt đối, không epoch nào thuộc cả hai (ép ở guard mỗi redeemer, §5).

---

## 3. Bất biến sổ sách LÕI (money-conservation)

**Bất biến neo (SỔ ↔ VALUE):** với MỌI vault sống,
```
(SỔ-VALUE)      L(vault) == c + v
```
Ép ở MỌI redeemer giữ-vault-sống qua mệnh đề `lamp_out_amt == d_out.conditional_lamp + d_out.vested_unlocked`:
`gen_drip_ok:661`, `reclaim_ok:356`, `vest_ok:428`, `claim_ok:484`. Genesis lập bất biến: `genesis_vault_ok:748` ép `lamp_locked == conditional_lamp` với `vested = 0` ⟹ `L = c + 0`.

Từ (SỔ-VALUE) + bất biến đơn vị mỗi redeemer suy ra 3 định lý §7.

**Đơn điệu (I-ACT-6):**
```
(MONO-c)   c chỉ GIẢM          (Reclaim −1 | Vest −k | Forfeit → 0);  KHÔNG đường tăng
(MONO-r)   r chỉ TĂNG          (Reclaim +1 | Forfeit += c_in)
(MONO-td)  td strict-tăng      (mọi tick: n > td_in)
```

**Ghi chú LAMP tổng thể (hard-constraint):** LAMP tổng cung cố định 36 tỷ, **KHÔNG burn**. Mọi LAMP rời vault đi ĐÚNG một trong hai đích {pot (kế toán Treasury/hồ chung), ví-owner} — KHÔNG có đường đốt. GenDrip (I-ACT-7) sinh MAGIC mà KHÔNG spend/đốt LAMP; MAGIC = account-trong-Vault (non-transferable), `nanogic = MAGIC × 10⁹`, KHÔNG mint MAGIC token, KHÔNG policy-id.

---

## 4. Bảng bất biến I-ACT-1 .. I-ACT-8b (cơ chế ép + neo dòng)

| ID | Bất biến (hình thức) | Cơ chế ép on-chain | Neo `file:hàm:dòng` |
|---|---|---|---|
| **I-ACT-1** | Genesis: `c = D ∈ [1,1001] ∧ v = 0 ∧ r = 0 ∧ td = 0 ∧ ie = 0 ∧ te = 0 ∧ L(vault) = D ∧ did_commit ≠ ∅` | mint-gate GenesisVault ép ĐÚNG 1 NFT (name = owner_commit) + khuôn datum + `lamp_locked == conditional_lamp` | `genesis_vault_ok:717` (mệnh đề 738-750) |
| **I-ACT-2** | Ranh giới pha: Reclaim ⟹ `n ≤ 1001`; Vest/Forfeit ⟹ `n > 1001`. Không chồng lấn. | guard `n <= phase1_last` (Reclaim) ; `n > phase1_last` (Vest/Forfeit) | `reclaim_ok:339` ; `vest_ok:408` ; `forfeit_ok:570`, `forfeit_close_ok:620` |
| **I-ACT-3** | (Registry-gate) `active` = tiêu qua dịch vụ Registry, self-consumption hợp lệ | **NGOÀI on-chain MVP** — keeper attest (tin cậy §8). Placeholder `has_counterparty_consume → False`. | `has_counterparty_consume:282`; dispatch `keeper_signed` |
| **I-ACT-4** | Reclaim (PHA-1): `c′ = c − 1 ∧ r′ = r + 1 ∧ v′ = v ∧ td′ = n`; đúng 1 LAMP → pot; grace + monotonic + đủ số dư | 10 mệnh đề (keeper, n≤1001, n≥grace, n>td, c≥1, sổ, sổ↔value, đích-pot, anti-drain) | `reclaim_ok:319` |
| **I-ACT-5** | Conservation toàn cục: `Σ D_phát = Σ nạp-pot + Σ vested_claimed→user + Δ(L vault-sống)`. LAMP không tự sinh. | hệ quả (SỔ-VALUE) + mỗi redeemer đích đúng (pot|owner) — §7 Định lý 1 | tổng hợp; đích: `lamp_to_addr:239` |
| **I-ACT-6** | D-cap: `D ≤ 1001`; `c` đơn điệu giảm; ranh giới-pha tại ngày 1001 bất kể D | `d.conditional_lamp <= d_cap` genesis; (MONO-c) §3 | `genesis_vault_ok:740` |
| **I-ACT-7** | GenDrip LAMP-preserved: `c′ = c ∧ v′ = v ∧ L(vault′) = L(vault)`; mọi sổ + DID/mốc bất biến | 8 mệnh đề: conditional/vested/L bất biến + sổ bất biến + anti-drain | `gen_drip_ok:646` |
| **I-ACT-8** | VestToOwner (PHA-2 GATED): `c′ = c − k ∧ v′ = v + k`, `k ≥ 1`, luỹ kế `v′ ≤ n − 1001`, `L(vault′) = L(vault)` (chỉ đổi sổ), `te′ = e ∧ ie′ = 0` | 9 mệnh đề: n>1001, keeper, monotonic, k≥1, k≤c, trần luỹ kế, sổ, sổ-epoch, TỔNG-L-bất biến, anti-drain | `vest_ok:391` |
| **I-ACT-8b** | ForfeitPhase2 (PHA-2): idle `e − te ≥ 1001` ⟹ `c′ = 0 ∧ r′ = r + c ∧ v′ = v` (vested BẤT BIẾN); toàn bộ `c` → pot | recreate: 8 mệnh đề; close: 6 mệnh đề (vested=0 → burn NFT) | `forfeit_ok:550` ; `forfeit_close_ok:604` |

**ClaimVested (I-ACT-8, đường LAMP→user DUY NHẤT):** `v′ = v − a ∧ c′ = c` (conditional BẤT BIẾN), `1 ≤ a ≤ v`, `L(vault′) = L(vault) − a`, ≥ a LAMP → ví owner (owner ký). Neo `claim_ok:452` (recreate) + `claim_close_ok:502` (đóng+burn khi `c = 0 ∧ a = v`).

> **Mã bất biến tiền tố `I-ACT-*` giữ nguyên** như bộ gốc đã duyệt (không đổi số, không alias). I-ACT-9 (settlement CARP, KHÔNG mint tự do) mô tả ở `PhoenixKey-Wakeme-Tech.md`/`-Exec.md` (không đụng money-safety on-chain của vault). I-ACT-10 (1-DID-1-vault) ép qua `owner_commit == did_commit == name` ở `genesis_vault_ok`.

---

## 5. Mệnh đề ép từng redeemer (đối chiếu code — trích guard load-bearing)

### 5.1 Reclaim — `reclaim_ok` (anti-idle PHA-1)
```
keeper_signed
∧ n ≤ 1001                                  -- (I-ACT-2) anti-idle DỪNG ở PHA-2
∧ n ≥ 7                                      -- grace onboarding
∧ n > td_in                                  -- (MONO-td) chống double-tick
∧ c_in ≥ 1                                   -- còn conditional để thu
∧ c′ = c_in − 1  ∧  r′ = r_in + 1  ∧  v′ = v_in
∧ td′ = n  ∧  ie′ = ie_in  ∧  te′ = te_in    -- PHA-1 KHÔNG chạm epoch-tracking
∧ identity_preserved                          -- owner_commit, vest_start, did_commit bất-biến
∧ L(out) = L(in) − 1  ∧  L(out) = c′ + v′     -- (SỔ-VALUE)
∧ lamp_to_addr(pot) ≥ 1                        -- ĐÍCH: về pot, KHÔNG vào ví keeper (LỖ-1 fix)
∧ nonlamp_preserved ∧ only_expected_policies   -- anti-drain
```
Neo: `activation_logic.ak:335-365`.

### 5.2 VestToOwner — `vest_ok` (PHA-2, GATED per-epoch)
```
let k = v′ − v_in
n > 1001 ∧ keeper_signed ∧ n > td_in
∧ k ≥ 1  ∧  k ≤ c_in                          -- tiến-triển thật + không tự-sinh vested
∧ v′ ≤ n − 1001                               -- TRẦN luỹ-kế 1 LAMP/ngày kể từ ngày 1002
∧ c′ = c_in − k  ∧  r′ = r_in                 -- reclaimed bất-biến
∧ te′ = e  ∧  ie′ = 0  ∧  td′ = n              -- mốc proven-active; reset idle
∧ identity_preserved
∧ L(out) = L(in)  ∧  L(out) = c′ + v′          -- TỔNG L bất-biến (chỉ đổi sổ nội-bộ)
∧ nonlamp_preserved ∧ only_expected_policies
```
Neo: `activation_logic.ak:406-432`. **Chú-ý auditor:** `L(out)=L(in)` — vest KHÔNG rời LAMP khỏi vault; chỉ flip `is_locked`. Đường LAMP→user chỉ mở ở ClaimVested sau đó.

### 5.3 ForfeitPhase2 — `forfeit_ok` / `forfeit_close_ok`
```
keeper_signed ∧ n > 1001
∧ e − te_in ≥ 1001                            -- idle GAP (lazy, không counter tăng-dần)
∧ c_in ≥ 1
∧ c′ = 0  ∧  r′ = r_in + c_in  ∧  v′ = v_in    -- vested BẤT BIẾN (đã-kiếm)
∧ te′ = e ∧ ie′ = 0 ∧ td′ = n
∧ L(out) = L(in) − c_in  ∧  L(out) = v′        -- toàn-bộ conditional rời; còn lại = vested
∧ lamp_to_addr(pot) ≥ c_in                     -- ĐÍCH pot
∧ nonlamp_preserved ∧ only_expected_policies
```
Nhánh close (`forfeit_close_ok`): `v_in = 0 ∧ L(in) = c_in ∧ ¬vault_recreated` → burn NFT. Neo: `567-591`, `617-631`.

### 5.4 ClaimVested — `claim_ok` / `claim_close_ok`
```
owner_signed                                  -- controller DID (anchor TAAD), KHÔNG keeper
∧ 1 ≤ a ≤ v_in
∧ c′ = c_in                                    -- conditional BẤT-KHẢ-XÂM (không rút phần chưa-kiếm)
∧ v′ = v_in − a  ∧  r′ = r_in  ∧  td′ = td_in
∧ ie′ = ie_in ∧ te′ = te_in                    -- rút vested KHÔNG reset đồng-hồ hoạt-động
∧ identity_preserved
∧ L(out) = L(in) − a  ∧  L(out) = c′ + v′
∧ lamp_to_addr(owner) ≥ a                       -- ĐÍCH ví owner (không pot/keeper)
∧ nonlamp_preserved ∧ only_expected_policies
```
Nhánh close (`claim_close_ok`): `c_in = 0 ∧ a = v_in ∧ L(in) = v_in ∧ ¬vault_recreated` → burn NFT. Neo: `467-490`, `513-524`.

### 5.5 GenDrip — `gen_drip_ok`
```
c′ = c_in ∧ v′ = v_in ∧ L(out) = L(in) ∧ L(out) = c′ + v′   -- LAMP đứng yên (I-ACT-7)
∧ r′ = r_in ∧ td′ = td_in ∧ ie′ = ie_in ∧ te′ = te_in         -- Gen chỉ ĐỌC, không đổi trạng-thái
∧ identity_preserved ∧ nonlamp_preserved ∧ only_expected_policies
```
Neo: `activation_logic.ak:656-671`. Engine MAGIC (magic_batches) NGOÀI validator này (§3.7 chưa chốt) — GenDrip chỉ chứng "LAMP không rời qua Gen". MAGIC = account-trong-Vault (KHÔNG mint token) — xem §3 ghi chú.

### 5.6 Mint-gate — `genesis_vault_ok` / `close_vault_ok`
- **GenesisVault:** ĐÚNG 1 movement `+1` dưới own_policy; carrier output well-formed tại `Script(own_policy)`; `name == owner_commit`; khuôn I-ACT-1; `lamp_locked == conditional_lamp`. Neo `717-758`.
- **CloseVault:** PURE-BURN — `list.length(moved) > 0 ∧ ∀ qty < 0` (mọi movement âm). Nối các nhánh close (`¬vault_recreated`). Neo `765-771`.

---

## 6. Không gian trạng thái + đồ thị chuyển (auditor coverage)

```
             GetLAMP                     Reclaim×(n≤1001)         Vest×(n≥1002,active)      ClaimVested
   [pot] ───────────────► (c=D,v=0) ─────────────────────► (c↓,v=0) ─────────────────► (c↓,v↑) ─────────────► (v↓)
                              │  GenDrip (loop, L bất-biến)      │  ┌─ Forfeit (e−te≥1001): c→0, →pot
                              └───────────────────────────────────┘  └─ ClaimVested-close (c=0∧a=v): burn NFT
```
Chuyển hợp lệ (đầy đủ):
- `GenDrip`: mọi pha, `(c,v) → (c,v)` — self-loop, L bất biến.
- `Reclaim`: chỉ PHA-1, `(c,v) → (c−1,v)`, 1 LAMP → **pot**.
- `VestToOwner`: chỉ PHA-2, `(c,v) → (c−k,v+k)`, L bất biến, k ≤ min(c, n−1001−v).
- `ClaimVested`: `(c,v) → (c,v−a)`, a LAMP → **ví owner**; đóng+burn nếu `c=0 ∧ a=v`.
- `ForfeitPhase2`: chỉ PHA-2 + idle, `(c,v) → (0,v)`, c LAMP → **pot**; đóng+burn nếu `v=0`.

Chuyển BỊ CẤM (auditor cần xác nhận REJECT — có test âm N1..N30 trong code):
`c → user` trực tiếp (chỉ pot|vested) · L tự sinh · Reclaim khi n>1001 · Vest khi n<1002 · vest > 1/ngày luỹ kế · claim > v · claim chạm c · non-owner claim · đổi DID/mốc · drain ADA/token-lạ · double-tick.

---

## 7. Ba định lý MONEY-SAFETY (+ chứng minh phác thảo)

### Định lý 1 (NO-DRAIN / conservation) — LAMP không tự sinh, không rò rỉ khỏi đường hợp lệ.
> **Phát biểu.** Với mọi tx spend hợp lệ trên vault sống, tổng LAMP hệ thống bảo toàn: mỗi LAMP rời vault đi ĐÚNG một trong hai đích {pot, ví-owner}; mỗi LAMP vào vault chỉ từ GetLAMP (pot). Không redeemer nào tạo LAMP. (Đồng nhất hard-constraint LAMP 36B không-burn: không đích nào là đốt.)

**Chứng minh (phác thảo, quy nạp theo redeemer).**
1. *Genesis lập bất biến* (SỔ-VALUE): `genesis_vault_ok:748` ép `L = c` với `v=0` ⟹ `L = c+v`. LAMP vào vault = D lấy từ pot (pot-validator gác — §8).
2. *Mỗi redeemer giữ (SỔ-VALUE):* các mệnh đề `L(out) = c′+v′` (§5.1-5.5) ⟹ nếu pre-state thoả `L(in)=c+v` thì post-state thoả `L(out)=c′+v′`. Vậy bất biến bảo toàn qua mọi bước.
3. *Δ LAMP rời vault = đích đúng:*
   - Reclaim: `L(out)=L(in)−1` **và** `lamp_to_addr(pot) ≥ 1` ⟹ đúng 1 LAMP tới pot, không nơi khác (vì `only_expected_policies` + `L(out)` đã kế toán phần còn lại).
   - Forfeit: `L(out)=L(in)−c_in` **và** `lamp_to_addr(pot) ≥ c_in` ⟹ c_in LAMP tới pot.
   - Claim: `L(out)=L(in)−a` **và** `lamp_to_addr(owner) ≥ a` ⟹ a LAMP tới ví owner.
   - Vest, GenDrip: `L(out)=L(in)` ⟹ 0 LAMP rời.
4. *Không tự sinh:* không mệnh đề nào cho `c′ > c` hay `L(out) > L(in)`. `c` chỉ giảm (MONO-c); `v` tăng chỉ qua Vest với `v′=v+k ∧ c′=c−k` (bù trừ, `c+v` bất biến). ∎

**Điểm auditor kiểm:** `lamp_to_addr` cộng LAMP tới ĐÚNG địa chỉ đích (`==target`), nên `≥ threshold` + `only_expected_policies` chặn "keeper nhét LAMP vào output thứ ba" (LỖ-1 lịch sử). Reclaim/Forfeit đích = `pot_address` (apply-param); Claim đích = `owner_address` (apply-param, neo theo DID không theo khoá).

### Định lý 2 (VESTED-UNTOUCHABLE) — LAMP đã sở hữu bất khả xâm bởi keeper/anti-idle/forfeit.
> **Phát biểu.** `vested_unlocked` chỉ giảm qua ClaimVested (owner ký) tới ví owner. Không redeemer keeper-gated nào (Reclaim, Vest, Forfeit) làm giảm `v`, cũng không chuyển `v` ra ngoài trừ ClaimVested.

**Chứng minh.**
- Reclaim: ép `v′ = v` (`reclaim_ok:348`). ⟹ v bất biến.
- Vest: `v′ = v + k, k ≥ 1` (`vest_ok`) ⟹ v chỉ TĂNG.
- Forfeit: ép `v′ = v` (`forfeit_ok:578`; nhánh close chỉ chạy khi `v = 0`) ⟹ v bất biến, phần đã sở hữu không bị forfeit.
- Claim: `v′ = v − a, a ≤ v`, đích **ví owner**, guard **owner_signed** (`auth_logic.anchor_controller_ok` — controller DID hiện tại). Keeper KHÔNG ký được đường này.
Vậy giảm-v ⟺ ClaimVested ⟺ owner ký ⟹ user toàn quyền phần đã kiếm. ∎

**Điểm auditor kiểm:** guard chữ ký. Claim dùng `owner_signed = anchor_controller_ok(...)` (đọc động controller qua anchor TAAD — xoay khoá không mất quyền); Reclaim/Vest/Forfeit dùng `keeper_signed = list.has(extra_signatories, keeper_pkh)`. Hai tập tách rời ⟹ keeper không giả mạo owner.

### Định lý 3 (FORFEIT-ONLY-TO-POT) — thu hồi PHA-2 chỉ về pot, chỉ phần chưa kiếm, chỉ khi idle đủ.
> **Phát biểu.** ForfeitPhase2 chỉ hợp lệ khi `n > 1001 ∧ e − te ≥ 1001`; hệ quả `c → 0` với toàn bộ c tới **pot** (không ví keeper), `v` bất biến, và `r += c` (audit).

**Chứng minh.**
- *Điều kiện idle load-bearing = GAP:* guard `e − te_in ≥ 1001` (`forfeit_ok:572`). `te` (proven-active) CHỈ tiến qua Vest (`vest_ok:422` set `te′ = e`). Nếu user thật-active mỗi epoch, Vest cập nhật `te` sát `e` ⟹ gap < 1001 ⟹ forfeit REJECT. Nếu idle ≥ 1001 epoch liên tục, `te` tụt lại ⟹ gap ≥ 1001 ⟹ forfeit cho phép. `ie` (idle_epochs_p2) là AUDIT-field, KHÔNG vào guard (lazy design — rẻ gas, không cần 1001 tx tick-idle).
- *Chỉ về pot:* `L(out)=L(in)−c_in` **và** `lamp_to_addr(pot) ≥ c_in` (§5.3) ⟹ c_in LAMP tới pot (Định lý 1, mục 3).
- *Chỉ phần chưa kiếm:* `v′ = v` ⟹ vested không bị đụng (Định lý 2).
- *Không keeper-đoạt:* đích cứng `pot_address` (apply-param); `only_expected_policies` chặn token-lạ. Keeper ký để attest idle (§8 giả định) nhưng KHÔNG chọn được đích.
∎

**Chú-ý an toàn (auditor):** genesis `te = 0`. User CHƯA vest lần nào ⟹ `te` đứng 0 ⟹ forfeit cho phép khi `e ≥ 1001` (≈ 1001 epoch ≈ 5005 ngày từ đầu PHA-2). Đây là hành vi ĐÚNG: user vào PHA-2 mà chẳng bao giờ active → phần chưa kiếm về pot. Không có off-by-one có lợi cho-attacker vì ngưỡng dùng `≥` và `te` chỉ tiến khi có vest thật.

---

## 8. Giả định tin cậy (NGOÀI phạm vi chứng minh — auditor phải soi riêng)

| # | Giả định | Rủi ro nếu vỡ | Trạng thái |
|---|---|---|---|
| T-1 | **Keeper trung thực** attest active (Vest) / idle (Reclaim, Forfeit). MVP: `keeper_signed`. | Keeper gian dối → Reclaim/Forfeit oan phần chưa kiếm (về **pot**, không về keeper — Đ-lý 1/3 chặn đoạt), hoặc Vest sai epoch. KHÔNG drain được sang keeper. | MVP tin keeper; §6.1 spec: thay bằng consume-event Registry reference-input |
| T-2 | **Pot-validator** (`dist_treasury`) tự gác quyền cấp LAMP đúng D. `genesis_vault_ok` chỉ gác VAULT dựng đúng khuôn + `lamp_locked == conditional_lamp`, KHÔNG gác pot chi đúng. | Pot chi sai D → vault khai-man; nhưng `lamp_locked==conditional_lamp` ràng datum ↔ LAMP thật ⟹ không bịa D mà không khoá LAMP. | pot-validator riêng (§6.4) |
| T-3 | **Uniqueness anchor PersonDID.** D keyed per-PersonDID. Lỗ ở tầng **mã hoá anchor** (KHÔNG phải sinh trắc): `GenesisPerson` đúc được anchor did-string bất kỳ với controller của attacker vì HW_Key P-256 KHÔNG verify on-chain (I-CURVE-4 carry-by-equality). | N anchor-giả → N×D LAMP rút khỏi pot (GV1, xem `-Exec.md` §7). Đóng ở tầng structural/cryptographic (PA2 UniquenessThread), KHÔNG phải chống-sybil-sinh trắc. | NGOÀI phạm vi vault; chờ PA2 land — **[CẦN CHỐT]** §9 |
| T-4 | **Engine Gen §3.7** chưa spell-out on-chain. GenDrip validator chỉ ép "LAMP không rời" (I-ACT-7). MAGIC = account-trong-Vault (KHÔNG mint token). | Nếu engine Gen production spend UTXO-LAMP (như code MAGIC cũ fire→Treasury) → vi phạm I-ACT-7. | **[CẦN CHỐT]** §9 |
| T-5 | **`has_counterparty_consume`** (`activation_logic.ak:282`) là cổng Registry-gate (I-ACT-3): PHẢI trả về đúng "tiêu thật qua dịch vụ Registry đăng ký hợp chuẩn" trước khi anti-idle/vest-gate dựa vào nó thay vì chỉ dựa keeper attest. Xem yêu cầu nối đủ ở §9 mục 6.1. | anti-idle không phân biệt "tiêu thật qua Registry" on-chain nếu cổng chưa nối đúng; dựa keeper. | Registry-team — xem §9 mục 6.1 |

**Kết luận phạm vi:** trong mô hình tin cậy {T-1..T-5}, ba định lý §7 GIỮ. Đặc biệt: **kể cả keeper ác-ý, LAMP không chảy sang keeper** — mọi đích thu hồi cứng = pot; mọi đích rút cứng = ví owner (owner ký). Đây là tuyến phòng thủ money-safety mạnh nhất của thiết kế. **Lưu ý phân tách:** T-3 là lỗ ANCHOR-uniqueness (mã hoá), KHÔNG phải lo ngại sybil-sinh trắc — sinh trắc Secure Enclave đủ chống trùng người; lỗ nằm ở anchor did-string không ràng khoá gốc.

---

## 9. [CẦN CHỐT] còn treo (auditor ghi nhận — chưa đóng)

| # | Mục | Ảnh hưởng money-safety | Chủ |
|---|---|---|---|
| **§3.7-1** | Spell-out **engine Gen ĐỌC số dư on-chain** (đọc VaultDatum reference-input → drip MAGIC → KHÔNG-spend UTXO-LAMP). Code MAGIC-repo cũ fire-LAMP→Treasury (TIÊU THỤ) LỖI THỜI — nếu tái dùng sẽ vi phạm I-ACT-7. MAGIC = account-trong-Vault, KHÔNG mint token. | CAO — quyết định I-ACT-7 có giữ ở production không. GenDrip validator hiện chỉ ép bất biến "LAMP không rời", chưa nối accounting MAGIC. | MAGIC/CARP-team |
| **§3.7-2** | Xác nhận CẢ InstantGen + ScheduleGen /CARP đều đọc số dư (không nhánh nào burn LAMP). | CAO — cùng T-4. | xác nhận /CARP |
| **§3.7-3** | Granularity: tick NGÀY (anti-idle/vest) + drip EPOCH (mặc định) hay drip-daily. | Thấp — không đụng conservation; chỉ nhịp MAGIC. | maintainer |
| **GV1/PA2** | **Uniqueness anchor PersonDID** (T-3) — PA2 UniquenessThread đóng lỗ đúc-anchor-did-bất kỳ ở tầng structural/cryptographic TRƯỚC khi mở GetLAMP-PersonDID production. | CAO — hệ quả kinh-tế (N×D rút ròng), KHÔNG đụng 3 định lý on-chain (validator đúng). | đội Core/on-chain (PA2) |
| **6.1** | **Registry-chuẩn dịch vụ** (thay counterparty-gate) — chi tiết yêu cầu ở T-5 (§8). Cổng chống-wash = dịch vụ Registry tiêu tài nguyên thật. | Trung — không đụng 3 định lý (keeper vẫn không đoạt tiền); đụng ĐỘNG CƠ anti-idle (đếm active đúng). | Registry-team |
| **6.5** | Cân đối tốc độ-vest-ra (PHA-2 rời hệ) vs nạp pot. Rủi ro F: pot cạn khi nhiều user qua PHA-2. | Trung — không phải drain (LAMP-vest đã kiếm hợp lệ rời hệ), mà là thanh khoản pot onboarding. | LAMP/backend |
| **MIN_MAGIC_TX** | = 10% MAGIC-gen-able từ `conditional_lamp`; granularity daily-gen-able vs cumulative. | Thấp — tham số ngưỡng active, không đụng conservation. | TẠM (07-06) |

---

## 10. Danh mục kiểm cho auditor (checklist rút gọn)

1. **(SỔ-VALUE)** `L(vault) == c + v` giữ ở genesis + mọi redeemer? → grep `lamp_out_amt == d_out.conditional_lamp + d_out.vested_unlocked` (phải xuất hiện ở `gen_drip_ok`, `reclaim_ok`, `vest_ok`, `claim_ok`).
2. **Đích đúng** mọi đường LAMP-rời: Reclaim/Forfeit → `pot_address`; Claim → `owner_address`; Vest/GenDrip → không rời. → `lamp_to_addr(...) ≥ threshold` + `only_expected_policies`.
3. **Chữ ký tách rời:** Claim = owner (`anchor_controller_ok`); Reclaim/Vest/Forfeit = keeper. Không nhánh nào cho keeper rút vested.
4. **Ranh giới pha:** Reclaim `n ≤ 1001`; Vest/Forfeit `n > 1001`. Test N13 (Reclaim PHA-2), N15 (Vest PHA-1).
5. **Trần vest luỹ kế** `v′ ≤ n − 1001` (không vest > 1/ngày). Test N16.
6. **Forfeit gap** `e − te ≥ 1001` load-bearing (không `ie`). `te` chỉ tiến qua Vest.
7. **Monotonic td** `n > td_in` mọi tick (chống double-tick/tua). Test N5, N18.
8. **Anti-drain** `nonlamp_preserved` (ADA) + `only_expected_policies` (token-lạ) mọi redeemer. Test N10/N11/N29.
9. **Close/burn** đúng: `claim_close_ok`/`forfeit_close_ok` chỉ khi vault thực rỗng (`c=0∧a=v` / `v=0`), nối `close_vault_ok` pure-burn (`¬vault_recreated`). Test P6.
10. **Man-thời gian:** `tx_lo` None (−∞) → REJECT; `days_elapsed` clamp âm → 0.

---

## Phụ lục A — Bảng đối chiếu bất biến ↔ dòng code

| Bất biến | `activation_logic.ak` |
|---|---|
| days_elapsed / p2_epoch | `:167` / `:181` |
| (SỔ-VALUE) L=c+v | genesis `:748`; gen `:661`; reclaim `:356`; vest `:428`; claim `:484` |
| I-ACT-1 genesis khuôn | `:738-750` |
| I-ACT-4 reclaim | `:335-365` |
| I-ACT-7 gen-drip | `:656-671` |
| I-ACT-8 vest gated | `:406-432` |
| I-ACT-8b forfeit | `:567-591` (recreate), `:617-631` (close) |
| ClaimVested | `:467-490` (recreate), `:513-524` (close) |
| mint-gate genesis/close | `:717-758` / `:765-771` |
| dispatch spend/mint | `activation_vault.ak:86` (mint), `:124-206` (spend `when redeemer`) |

---

## Nguồn

- Code (nguồn chân lý): `PhoenixKey-Validator/lib/phoenixkey/activation_logic.ak` (~1859 dòng, 69 test), `PhoenixKey-Validator/validators/activation_vault.ak` (212 dòng), `PhoenixKey-Validator/lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok`).
- Nguồn thiết kế nội bộ (không công khai). (đã qua rà soát nội bộ)
- Đánh giá gaming nội bộ (không công khai) (GV1/GV2 khung anchor-uniqueness + wash-Registry).
- Cross-ref canonical: `PhoenixKey-Specs/PhoenixKey-Math.md` (key hierarchy §6, TAAD §10, type catalog §12–21).
- Tài liệu cùng bộ: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md), [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md), [PhoenixKey-Wakeme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Wakeme-Specs/blob/main/PhoenixKey-Wakeme-Exec.md).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Wakeme-Specs/blob/main/PhoenixKey-STATUS.md)

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
