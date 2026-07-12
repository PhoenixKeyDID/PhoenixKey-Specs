# PhoenixKey — Wakeme · Đặc-tả TOÁN hình-thức (cho AUDITOR)

> **Module:** Wakeme (GetLAMP — kích-hoạt nhận LAMP). **Loại doc:** Toán hình-thức. **Ngày:** 2026-07-09 · **Sửa lớn cơ-chế PHA-2:** 2026-07-12 · **Khớp code v5 (7-field + đơn-vị oildrop + te=−1):** 2026-07-12.
> **Đối-tượng đọc:** auditor smart-contract + nhà kiểm-toán tokenomic. Đây là đặc-tả TOÁN (định-nghĩa, bất-biến, công-thức, chứng-minh money-safety), KHÔNG phải doc thiết-kế. Thiết-kế/động-cơ ở [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md); kỹ-thuật ở [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md); điều-hành ở [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).
>
> **Doc KHỚP CODE:** đặc-tả này mô-tả cơ-chế PHA-2 **5-LAMP/epoch** đã chốt 2026-07-12 và **đã hiện-thực xong** trên nhánh `claude/wakeme-5lamp-epoch` (`activation_logic.ak` 1796 dòng / 69 test + `activation_vault.ak` 280 dòng / 1 test, `aiken check` 2026-07-12 **216/216 pass**). Các neo `hàm:dòng` trỏ ĐÚNG code v5. **Datum v5 = 7 FIELD** (BỎ `vested_unlocked`/`owned_out` + `idle_epochs_p2` của bản nháp trước); `conditional_lamp` đếm **NGUYÊN-LAMP** (không phải oildrop); genesis `last_tick_epoch = −1` (sentinel). Khi văn ≠ code → **code thắng**.
>
> **Phạm-vi money-safety đặc-tả này CHỨNG:** LAMP không tự-sinh; conditional-LAMP rời vault CHỈ về hai đích {pot, ví-Phoenix-của-DID}; LAMP đã chuyển ra ví Phoenix = bất-khả-xâm-phạm (ngoài vault); reclaim per-epoch chỉ về pot. **NGOÀI phạm-vi (giả-định tin-cậy, ghi rõ §8):** keeper trung-thực (attest active/idle đúng), engine Gen `§3.7` chưa spell-out on-chain, uniqueness PersonDID (sinh-trắc Enclave — xem T-3 + §8), pot-validator (`dist_treasury`) tự gác quyền cấp LAMP.

---

## 0. Changelog

### 0.1 Khớp code v5 (2026-07-12) — 4 vá đóng lỗ audit bản nháp trước

Bản nháp changelog §0.2 (viết TRƯỚC khi code v5 hoàn-thành) dùng mô-hình datum **8-field** với hai field `owned_out` (o) + `idle_epochs_p2` (ie) và genesis `te = 0`. Code v5 thực-tế **KHÁC** — 4 điểm sau đã sửa văn cho khớp code (code = nguồn chân-lý):

| # | Lỗ nháp trước | Code v5 (đã khớp) | Trục vì-sao |
|---|---|---|---|
| **1 · Đơn-vị I-ACT-1** | `c = D ∧ L(vault) = D` (footnote nói `c`,`L` đều oildrop → nhập-nhằng `d_cap=1001` là oildrop hay ×10⁶) | `conditional_lamp` đếm **NGUYÊN-LAMP** (`c ∈ [1,1001]`); LAMP-token đo **oildrop**; bất-biến `L(vault) = c × oil_per_lamp`, `oil_per_lamp = 10⁶`. Genesis ép `lamp_locked == conditional_lamp × oil_per_lamp`. | (b) một đại-lượng một đơn-vị — auditor không còn phải đoán `d_cap` là 0.001 LAMP hay 1001 LAMP; (d) khoá đúng lượng LAMP thật (bản v4.1 khoá thiếu 10⁶ lần). |
| **2 · te=−1 + tuần-tự** | genesis `te = 0`; guard `e > te`, hậu-điều-kiện `te′ = e` (GỘP-NHẢY qua nhiều epoch idle trong 1 tx) | genesis `te = −1` (sentinel "chưa xử epoch nào"); guard `j = te+1 ≤ e_now`, hậu-điều-kiện `te′ = te+1` (xử **ĐÚNG 1 epoch/tx**, tuần-tự, KHÔNG nhảy). Epoch đầu `j = 0`. | (b) đóng off-by-one (epoch 0 xử được) + gap-collapse (không keeper gộp nhiều epoch bỏ sót); (c) mỗi tx đúng 1 bước 5-LAMP, cap chống rút gộp. |
| **3 · Datum 7-field** | 8-field: thêm `owned_out` (o) + `idle_epochs_p2` (ie) | **7-field**: `owner_commit, did_commit, vest_start_slot, conditional_lamp, reclaimed_to_pot, last_tick_day, last_tick_epoch`. BỎ `owned_out` + `idle_epochs_p2` (dư-thừa: `o` suy được `= D − c − r`; `ie` chỉ audit không load-bearing). | (b) ít field = ít bề-mặt-lỗi + datum nhỏ hơn (phí min-ADA); (c) không lưu đại-lượng suy-được-từ đại-lượng khác. |
| **4 · Nhất-quán 4 doc** | Tech tả 9-field + `vested`/`ClaimVested`/`owned_out`; Vi-Feat "sau 1001 ngày sở-hữu trọn" (mâu-thuẫn nhịp-Kỳ) | Tech = 7-field, không `vested`/`ClaimVested`/`owned_out`; Vi-Feat "sở-hữu DẦN mỗi Kỳ từ ngày 1002" (≤5 LAMP/Kỳ). Vi-Feat↔Math↔Tech↔Exec khớp. | (d) người-dùng không hiểu-lầm nhận-trọn-một-lần; auditor đọc 4 doc không mâu-thuẫn. |

**Bảo-toàn per-vault (7-field):** `D = c + (Σ q→owner) + r`. Vault chỉ lưu `c` (còn-khoá) + `r` (đã-về-pot); phần đã-sở-hữu-ra-ví KHÔNG là field (suy `= D − c − r`) — nó đã RỜI vault, nằm trong ví Phoenix của DID. Mỗi LAMP rời vault đi đúng một trong {pot, ví-Phoenix-DID}, không đường đốt (LAMP 36B không-burn). Chứng-minh: §7.

### 0.2 Cơ-chế PHA-2 (2026-07-12) — từ vest-ghi-sổ sang 5-LAMP/epoch

**Đổi gì.** PHA-2 chuyển nhịp từ NGÀY (vest 1 LAMP/ngày ghi-sổ trong vault + ClaimVested owner rút sau) sang **EPOCH** (1 epoch = 5 ngày = 432.000 slot). Mỗi epoch xử-lý **đúng một lần**, tiến đơn-điệu:

| Trục | Mô-hình CŨ (tới 07-11) | Mô-hình MỚI (chốt 07-12, code v5) |
|---|---|---|
| Nhịp PHA-2 | NGÀY (`n > last_tick_day`) | EPOCH tuần-tự (`j = te+1 ≤ e_now`, `te′ = te+1`), 1 epoch = 5 ngày = 432.000 slot |
| Giao-quyền-sở-hữu | VestToOwner **ghi-sổ trong vault** (`v` tăng, LAMP ĐỨNG YÊN) → ClaimVested owner rút ra ví | **OwnEpoch**: epoch ACTIVE → chuyển `q = min(5, c)` LAMP **RA KHỎI vault** thẳng về **ví Phoenix của DID** (`owner_address`) = sở-hữu-hoàn-toàn tức-thời. KHÔNG còn bước ClaimVested. |
| Thu-hồi PHA-2 | ForfeitPhase2 chỉ khi idle-gap **≥ 1001 epoch** | **ReclaimEpoch**: **MỖI** epoch IDLE → `q = min(5, c)` LAMP về **pot** (tức-thời, không chờ 1001 epoch) |
| Đơn-vị bước | 1 LAMP/ngày | `q = min(5, c)` NGUYÊN-LAMP/epoch (5 = 1 LAMP/ngày × 5 ngày/epoch — giữ nguyên nhịp vest cũ, gom theo epoch) |
| Field số-dư | `vested_unlocked` = số-dư đã-vest CÒN TRONG vault (load-bearing) | KHÔNG field: phần đã-sở-hữu đã RA ví (ngoài vault), suy `= D − c − r`; vault chỉ giữ `conditional_lamp` + `reclaimed_to_pot` |
| Đóng vault | burn khi `c = 0 ∧ v = 0` | burn khi **`c = 0`** (mọi LAMP điều-kiện đã giải-quyết xong — về chủ hoặc về pot) |
| Redeemer spend | 5 (GenDrip/Reclaim/VestToOwner/ClaimVested/ForfeitPhase2) | **4** (GenDrip/Reclaim/**OwnEpoch**/**ReclaimEpoch**) |

**Ánh-xạ redeemer:** `VestToOwner → OwnEpoch` (nay chuyển-ra-ví thay vì ghi-sổ); `ClaimVested → BỎ` (hấp-thu vào OwnEpoch — LAMP tới ví tự-động, không cần user rút); `ForfeitPhase2 → ReclaimEpoch` (per-epoch thay gap-1001). `GenDrip`/`Reclaim` giữ nguyên.

**Vì sao (4 trục).**
- **(a) Định-hướng dài-hạn:** LAMP đã-kiếm nằm THẲNG trong ví người-dùng (bất-khả-xâm-phạm tuyệt-đối vì ngoài vault) thay vì "đã-vest nhưng còn trong vault chờ rút" — quyền-sở-hữu rõ-ràng, không phụ-thuộc user nhớ bấm [Rút].
- **(b) First-principles:** bỏ trạng-thái trung-gian "vested-in-vault" và redeemer ClaimVested ⟹ ít bề-mặt-tấn-công hơn (4 redeemer thay 5), bất-biến bảo-toàn đơn-giản hơn (`L(vault) = c × oil`, không còn `c + v`).
- **(c) Tối-ưu:** nhịp EPOCH khớp epoch Cardano (gate-active tự-nhiên theo epoch), gom 5-ngày/1-tx ⟹ giảm 5× số tx keeper so với tick-daily; forfeit per-epoch tức-thời ⟹ pot tái-tuần-hoàn nhanh, không đọng 1001-epoch.
- **(d) Lợi-ích user + bền-vững:** LAMP tới ví ngay khi active (không chờ, không thao-tác); phần chưa-kiếm quay về pot sớm nuôi người-mới.

**Bất-biến bảo-toàn KHÔNG vỡ:** với mỗi vault, `D = c + (Σ q→owner) + r`. Mỗi LAMP rời vault đi đúng một trong {pot, ví-Phoenix-DID}, không đường đốt (LAMP 36B không-burn giữ nguyên). Chứng-minh: §7.

---

## 1. Ký-hiệu

| Ký-hiệu | Kiểu | Nghĩa | Neo code |
|---|---|---|---|
| `D` | ℤ⁺ | lượng LAMP cấp mỗi GetLAMP; `D = min(1001, ⌊pot/10⁶⌋)` | `d_cap = 1001`, `genesis_vault_ok` (1 ≤ conditional ≤ d_cap) |
| `s₀` | slot | `vest_start_slot` — mốc-0 đồng-hồ (slot lúc GetLAMP) | `ActivationVaultDatum.vest_start_slot` |
| `lo` | slot | lower-bound HỮU-HẠN của validity-interval tx = "now" | `tx_lo` (None nếu −∞ → REJECT) |
| `n` | ℤ≥0 | số NGÀY đã trôi = `days_elapsed(lo, s₀)` | `days_elapsed` |
| `e` | ℤ≥−1 | p2-EPOCH tương-đối từ đầu PHA-2 = `p2_epoch(lo, s₀)` | `p2_epoch` |
| `c` | ℤ≥0 (**NGUYÊN-LAMP**¹) | `conditional_lamp` — LAMP khoá có-điều-kiện (chưa-sở-hữu), đếm **NGUYÊN-LAMP** ∈ [0,1001] | `.conditional_lamp` |
| `r` | ℤ≥0 (NGUYÊN-LAMP) | `reclaimed_to_pot` — luỹ-kế NGUYÊN-LAMP về pot (audit, chỉ tăng): Reclaim PHA-1 + ReclaimEpoch PHA-2 | `.reclaimed_to_pot` |
| `td` | ℤ≥0 | `last_tick_day` — NGÀY tick gần nhất (monotonic, **PHA-1**) | `.last_tick_day` |
| `te` | ℤ≥−1 | `last_tick_epoch` — p2-epoch cuối **ĐÃ XỬ-LÝ** (OwnEpoch active HOẶC ReclaimEpoch idle); **genesis = −1** (sentinel "chưa xử epoch nào"); tuần-tự `+1`/epoch | `.last_tick_epoch` |
| `L(x)` | ℤ≥0 (**oildrop**) | lượng LAMP-token thật (oildrop) trong Value `x` = `quantity_of(x, lamp_policy, lamp_name)` | `lamp_in` |
| `q` | ℤ≥0 (NGUYÊN-LAMP) | bước PHA-2 mỗi epoch = `min(5, c)` (5 = 1 LAMP/ngày × 5 ngày) | `epoch_release`, `epoch_unit = 5` |
| primed `x′` | — | giá-trị field/Value ở output tái-tạo (post-state) | `d_out`, `out_v` |

¹ **Hai đơn-vị TÁCH BẠCH (bất-biến cốt-lõi, đã khớp code v5):** sổ-đếm (`c`, `r`, `q`, `D`) tính **NGUYÊN-LAMP** (đơn-vị người-dùng-nhìn-thấy, `d_cap = 1001` LAMP); LAMP-token thật trong Value vault (`L(x)`) đo **oildrop** (`1 LAMP = 10⁶ oildrop`, `oil_per_lamp`). Cầu-nối = bất-biến `L(vault) = c × oil_per_lamp` (§3). Không còn nhập-nhằng: `d_cap = 1001` là **1001 NGUYÊN-LAMP** = `1001 × 10⁶` oildrop khoá thật. Mọi số-học integer floor-div, không float.

**Hằng (khớp `activation_logic.ak:79-102`):**
```
slots_per_day    = 86_400        (1 slot = 1 giây; đọc từ config, network-aware)
slots_per_epoch  = 432_000       (1 EPOCH = 5 NGÀY = 432.000 slot)   ← TUYỆT-ĐỐI không gọi "tuần"
oil_per_lamp     = 1_000_000     (1 NGUYÊN-LAMP = 10⁶ oildrop — cầu sổ↔value)
epoch_unit       = 5             (PHA-2: q = min(5, c) NGUYÊN-LAMP/epoch — 1 LAMP/ngày × 5 ngày)
grace_days       = 7             (PHA-1 onboarding miễn anti-idle)
reclaim_unit     = 1             (PHA-1: 1 NGUYÊN-LAMP/ngày-idle → pot)
d_cap            = 1001          (trần D, NGUYÊN-LAMP)
phase1_last      = 1001          (PHA-1 "Giai đoạn Ngày": n ≤ 1001;  PHA-2 "Giai đoạn Kỳ": n ≥ 1002)
```
> **Đã BỎ** hằng `forfeit_idle_epochs = 1001` của mô-hình cũ (thu-hồi PHA-2 nay per-epoch, không chờ gap-1001). **Đã THÊM** `epoch_unit = 5` + `oil_per_lamp = 10⁶`.

---

## 2. Hàm đồng-hồ (định-nghĩa hình-thức)

**Đ-1 · days_elapsed** (`activation_logic.ak:148`):
```
days_elapsed(lo, s₀) = ⌊(lo − s₀) / 86_400⌋   nếu lo − s₀ ≥ 0
                     = 0                         nếu lo − s₀ < 0     (clamp — chống man-thời-gian âm)
```

**Đ-2 · p2_epoch** (`activation_logic.ak:161`) — epoch tương-đối tính từ ĐẦU PHA-2 (ngày 1002 = epoch 0):
```
off = lo − s₀ − 1001 × 86_400          (slot đã trôi kể từ mốc PHA-2)
p2_epoch(lo, s₀) = ⌊off / 432_000⌋      nếu off ≥ 0
                 = −1                     nếu off < 0    (chưa tới PHA-2 — sentinel)
```
Bổ-đề Đ-2a: forfeit/vest chỉ gọi khi `n > phase1_last` ⟹ `off > 0` ⟹ `e ≥ 0` (sentinel −1 không lọt vào so-sánh ngưỡng). Chứng-minh: `n > 1001 ⟹ lo − s₀ ≥ 1002·86400 > 1001·86400 ⟹ off > 0`. ∎

**Ranh-giới pha (I-ACT-2):** PHA-1 (**"Giai đoạn Ngày" / Daily**) ⟺ `n ≤ 1001`; PHA-2 (**"Giai đoạn Kỳ" / Epoch**, 1 Epoch = 5 ngày = 432.000 slot) ⟺ `n ≥ 1002`. Tách rời tuyệt-đối, không epoch nào thuộc cả hai (ép ở guard mỗi redeemer, §5). Token hình-thức PHA-1/PHA-2 giữ nguyên để neo tham-chiếu; nhãn hiển-thị "Giai đoạn Ngày/Kỳ" chỉ THÊM cho người ngoài đọc.

---

## 3. Bất-biến sổ-sách LÕI (money-conservation)

**Bất-biến neo (SỔ ↔ VALUE):** với MỌI vault sống,
```
(SỔ-VALUE)      L(vault) == c × oil_per_lamp        (c NGUYÊN-LAMP; L oildrop; oil_per_lamp = 10⁶)
```
Vault sống chỉ giữ **conditional LAMP** (`c` NGUYÊN-LAMP, khoá thật `c × 10⁶` oildrop). LAMP đã-sở-hữu ĐÃ RỜI vault (nằm trong ví Phoenix của DID) — KHÔNG là field datum, KHÔNG cộng vào Value vault. Ép ở MỌI redeemer giữ-vault-sống qua `lamp_out_amt == d_out.conditional_lamp * oil_per_lamp`. Genesis lập bất-biến: ép `lamp_locked == conditional_lamp * oil_per_lamp` ⟹ `L = c × 10⁶`.

**Bảo-toàn per-vault (thay cho `L = c + v` cũ):** đại-lượng luỹ-kế bảo-toàn (đếm NGUYÊN-LAMP) là
```
(BẢO-TOÀN)      D == c + o + r      (initial-D = conditional-còn + đã-sở-hữu-ra-ví + đã-reclaim-pot)
```
trong đó `o = Σ(q → ví-Phoenix-DID)` là phần đã-sở-hữu-ra-ví — **KHÔNG là field on-chain** (datum 7-field không lưu; suy `o = D − c − r`). Genesis: `c = D, r = 0` (⟹ `o = 0`). Mỗi redeemer giữ `Δc + Δo + Δr = 0` (§5). Từ (SỔ-VALUE) + (BẢO-TOÀN) suy ra 3 định-lý §7.

**Đơn-điệu (I-ACT-6):**
```
(MONO-c)   c chỉ GIẢM          (Reclaim −1 | OwnEpoch −q | ReclaimEpoch −q);  KHÔNG đường tăng
(MONO-r)   r chỉ TĂNG          (Reclaim +1 | ReclaimEpoch += q)
(MONO-td)  td strict-tăng      (mọi tick PHA-1: n > td_in)
(MONO-te)  te tăng ĐÚNG +1     (mọi xử-lý PHA-2: te′ = te+1, với ràng-buộc te+1 ≤ e_now — tuần-tự, mỗi epoch xử ĐÚNG một lần)
```
Hệ-quả (MONO-c): `o = D − c − r` chỉ TĂNG khi OwnEpoch (`Δc = −q`, `Δr = 0` ⟹ `Δo = +q`), bất-biến khi Reclaim/ReclaimEpoch (`Δc + Δr = 0` ⟹ `Δo = 0`).

**Ghi-chú LAMP tổng-thể (hard-constraint):** LAMP tổng-cung cố-định 36 tỷ, **KHÔNG burn**. Mọi LAMP rời vault đi ĐÚNG một trong hai đích {pot (kế-toán Treasury/hồ-chung), ví-Phoenix-của-DID (`owner_address`)} — KHÔNG có đường đốt. GenDrip (I-ACT-7) sinh MAGIC mà KHÔNG spend/đốt LAMP; MAGIC = account-trong-Vault (non-transferable), `nanogic = MAGIC × 10⁹`, KHÔNG mint MAGIC token, KHÔNG policy-id.

---

## 4. Bảng bất-biến I-ACT-1 .. I-ACT-8b (cơ-chế-ép + neo dòng)

| ID | Bất-biến (hình-thức) | Cơ-chế-ép on-chain | Neo `hàm` (khớp code v5) |
|---|---|---|---|
| **I-ACT-1** | Genesis: `c = D ∈ [1,1001] ∧ r = 0 ∧ td = 0 ∧ te = −1 ∧ L(vault) = D × oil_per_lamp ∧ did_commit ≠ ∅` (7-field; không `owned_out`/`idle_epochs_p2`) | mint-gate GenesisVault ép ĐÚNG 1 NFT (name = owner_commit) + khuôn datum + `lamp_locked == conditional_lamp * oil_per_lamp` + `anchor_controller_ok` (controller ký) | `genesis_vault_ok` |
| **I-ACT-2** | Ranh-giới pha: Reclaim (PHA-1 Ngày) ⟹ `n ≤ 1001`; OwnEpoch/ReclaimEpoch (PHA-2 Kỳ) ⟹ `n > 1001`. Không chồng-lấn. | guard `n <= phase1_last` (Reclaim) ; `n > phase1_last` (OwnEpoch/ReclaimEpoch) | `reclaim_ok` ; `own_epoch_ok` ; `reclaim_epoch_ok` |
| **I-ACT-3** | (Registry-gate) `active` = tiêu ≥ `MIN_MAGIC_CONSUME` qua dịch-vụ **trả phí** Registry; self-consumption hợp-lệ | **NGOÀI on-chain MVP** — keeper attest (tin-cậy §8). Placeholder `has_counterparty_consume → False`. | `has_counterparty_consume`; dispatch `keeper_signed` |
| **I-ACT-4** | Reclaim (PHA-1 Ngày): `c′ = c − 1 ∧ r′ = r + 1 ∧ td′ = n ∧ te′ = te`; đúng 1 LAMP (×oil) → pot; grace + monotonic + đủ-số-dư | 10 mệnh-đề (keeper, n≤1001, n≥grace, n>td, c≥1, sổ, sổ↔value ×oil, đích-pot-tag, anti-drain) | `reclaim_ok` |
| **I-ACT-5** | Conservation toàn-cục: `Σ D_phát = Σ (→pot) + Σ (→ví-Phoenix-DID) + Σ(c vault-sống)`; per-vault `D = c + o + r` (`o = D−c−r`, không lưu). LAMP không tự-sinh. | hệ-quả (SỔ-VALUE)+(BẢO-TOÀN) + mỗi redeemer đích-đúng (pot\|owner) — §7 Định-lý 1 | tổng-hợp; đích: `lamp_to_addr` / `lamp_to_addr_tagged` |
| **I-ACT-6** | D-cap + trần-luỹ-kế: `D ≤ 1001`; `c` đơn-điệu-giảm ⟹ tổng-sở-hữu `o = D − c − r ≤ D ≤ 1001` (không vượt D-đầu-PHA-2); ranh-giới-pha tại ngày 1001 bất-kể D | `d.conditional_lamp <= d_cap` genesis; (MONO-c) §3 | `genesis_vault_ok` |
| **I-ACT-7** | GenDrip LAMP-preserved: `c′ = c ∧ L(vault′) = L(vault) = c′ × oil`; mọi sổ (`r,td,te`) + DID/mốc bất-biến | 8 mệnh-đề: conditional/L bất-biến (×oil) + sổ bất-biến + anti-drain | `gen_drip_ok` |
| **I-ACT-8** | **OwnEpoch** (PHA-2 Kỳ, ACTIVE, GATED): `j = te+1 ≤ e_now ∧ q = min(5, c) ∧ q ≥ 1` ⟹ `c′ = c − q ∧ r′ = r ∧ td′ = td`, `L(vault′) = L(vault) − q × oil`, `q` LAMP (×oil) → **ví Phoenix DID** (`owner_address`), `te′ = te + 1` (KHÔNG `te′ = e_now`) | 9 mệnh-đề: n>1001, e_now≥0, keeper-attest-active, `te+1 ≤ e_now` (tuần-tự), q=min(5,c), q≥1, sổ `Δc+Δo=0`, đích-owner ×oil, `L′=L−q×oil`, anti-drain | `own_epoch_ok` (recreate) ; `own_epoch_close_ok` (c′=0 → burn) |
| **I-ACT-8b** | **ReclaimEpoch** (PHA-2 Kỳ, IDLE): `j = te+1 ≤ e_now ∧ q = min(5, c) ∧ q ≥ 1` ⟹ `c′ = c − q ∧ r′ = r + q ∧ td′ = td`, `L(vault′) = L(vault) − q × oil`, `q` LAMP (×oil) → **pot** (tag owner_commit), `te′ = te + 1`. Auth: keeper HOẶC owner (escape-hatch). | 8 mệnh-đề: n>1001, e_now≥0, keeper∨owner, `te+1 ≤ e_now`, q=min(5,c), sổ `Δc+Δr=0`, đích-pot-tag ×oil, `L′=L−q×oil`, anti-drain | `reclaim_epoch_ok` (recreate) ; `reclaim_epoch_close_ok` (c′=0 → burn) |

**KHÔNG còn ClaimVested.** Đường LAMP→user DUY-NHẤT nay là **OwnEpoch** — chuyển thẳng ra ví Phoenix của DID mỗi epoch ACTIVE, không có bước rút riêng, không có số-dư "vested-in-vault" để rút. LAMP một khi ra ví = **ngoài** phạm-vi validator vault (bất-khả-xâm-phạm tuyệt-đối). Đóng+burn NFT khi **`c = 0`** (mọi conditional đã giải-quyết — về chủ hoặc về pot), qua nhánh `own_epoch_close_ok` / `reclaim_epoch_close_ok` nối `close_vault_ok` pure-burn.

> **Mã bất-biến tiền-tố `I-ACT-*` giữ nguyên** (không đổi số): I-ACT-8 nay là **OwnEpoch** (thay VestToOwner), I-ACT-8b nay là **ReclaimEpoch** (thay ForfeitPhase2). I-ACT-9 (settlement CARP, KHÔNG mint tự-do) ở `-Tech.md`/`-Exec.md`. I-ACT-10 (1-DID-1-vault) ép qua `owner_commit == did_commit == name` ở `genesis_vault_ok`.

---

## 5. Mệnh-đề-ép từng redeemer (đối-chiếu code — trích guard load-bearing)

### 5.1 Reclaim — `reclaim_ok` (anti-idle PHA-1)
```
keeper_signed
∧ n ≤ 1001                                  -- (I-ACT-2) anti-idle DỪNG ở PHA-2
∧ n ≥ 7                                      -- grace onboarding
∧ n > td_in                                  -- (MONO-td) chống double-tick
∧ c_in ≥ 1                                   -- còn conditional để thu
∧ c′ = c_in − 1  ∧  r′ = r_in + 1
∧ td′ = n  ∧  te′ = te_in                     -- PHA-1 KHÔNG chạm epoch-tracking (te giữ nguyên, kể cả −1)
∧ identity_preserved                          -- owner_commit, vest_start, did_commit bất-biến
∧ L(out) = L(in) − 1·oil  ∧  L(out) = c′·oil  -- (SỔ-VALUE) vault chỉ giữ c NGUYÊN-LAMP (×oil oildrop)
∧ lamp_to_addr_tagged(pot, owner_commit) ≥ 1·oil  -- ĐÍCH: về pot GẮN-TAG, KHÔNG ví keeper (LỖ-1 fix)
∧ nonlamp_preserved ∧ only_expected_policies   -- anti-drain
```
Neo: `reclaim_ok` (`oil = oil_per_lamp = 10⁶`; PHA-1 giữ nguyên logic mô-hình cũ, chỉ khớp đơn-vị ×oil + đích pot GẮN-TAG `owner_commit` chống double-satisfaction).

### 5.2 OwnEpoch — `own_epoch_ok` / `own_epoch_close_ok` (PHA-2 Kỳ, ACTIVE)
```
let q = min(5, c_in)                           -- epoch_unit = 5 (1 LAMP/ngày × 5 ngày)
keeper_signed                                  -- keeper attest epoch ACTIVE (Registry-gate §8)
∧ n > 1001                                     -- PHA-2 (I-ACT-2)
∧ e_now ≥ 0                                     -- đã tới PHA-2 (sentinel −1 loại)
∧ te_in + 1 ≤ e_now                            -- (MONO-te) xử epoch KẾ-TIẾP j = te+1, KHÔNG nhảy
∧ q ≥ 1                                         -- còn conditional để giao (c_in ≥ 1)
∧ c′ = c_in − q  ∧  r′ = r_in                  -- sổ: Δc = −q, r bất-biến (⟹ Δo = +q, o không lưu)
∧ te′ = te_in + 1  ∧  td′ = td_in              -- mốc epoch tiến ĐÚNG +1 (KHÔNG te′ = e_now)
∧ identity_preserved
∧ L(out) = L(in) − q·oil  ∧  L(out) = c′·oil    -- q NGUYÊN-LAMP (×oil) RỜI vault; còn lại = c′·oil
∧ lamp_to_addr(owner_address) ≥ q·oil           -- ĐÍCH: ví Phoenix của DID (apply-param, DID-bound)
∧ nonlamp_preserved ∧ only_expected_policies    -- anti-drain
```
**Chú-ý auditor:** khác mô-hình cũ — LAMP RỜI vault THẲNG về ví Phoenix của DID (không còn "vest ghi-sổ trong vault" rồi ClaimVested). Đích cứng `owner_address` (apply-param) ⟹ keeper ký attest-active nhưng KHÔNG chọn được đích (không thể chuyển sang ví keeper). `q ≤ 5` là anti-drain per-epoch. **Tuần-tự (fix gap-collapse):** hậu-điều-kiện `te′ = te_in + 1` (KHÔNG `te′ = e_now`) ⟹ mỗi tx xử ĐÚNG 1 epoch; keeper KHÔNG gộp nhiều epoch idle-bỏ-sót vào 1 bước-5-LAMP. Genesis `te = −1` ⟹ epoch đầu `j = 0` (fix off-by-one — epoch 0 xử được). Nhánh close (`own_epoch_close_ok`): `c′ = 0 ∧ ¬vault_recreated` → burn NFT (`close_vault_ok`). Neo: `own_epoch_ok` (recreate) ; `own_epoch_close_ok` (close).

### 5.3 ReclaimEpoch — `reclaim_epoch_ok` / `reclaim_epoch_close_ok` (PHA-2 Kỳ, IDLE)
```
let q = min(5, c_in)
keeper_signed ∨ owner_signed                    -- keeper attest IDLE HOẶC owner escape-hatch (tự trả pot)
∧ n > 1001                                     -- PHA-2
∧ e_now ≥ 0                                     -- đã tới PHA-2
∧ te_in + 1 ≤ e_now                            -- (MONO-te) xử epoch KẾ-TIẾP j = te+1, tuần-tự
∧ q ≥ 1                                         -- còn conditional để thu (c_in ≥ 1)
∧ c′ = c_in − q  ∧  r′ = r_in + q              -- sổ: Δc + Δr = 0 (o = D−c−r bất-biến)
∧ te′ = te_in + 1  ∧  td′ = td_in              -- mốc epoch tiến ĐÚNG +1 (KHÔNG te′ = e_now)
∧ identity_preserved
∧ L(out) = L(in) − q·oil  ∧  L(out) = c′·oil    -- q NGUYÊN-LAMP (×oil) RỜI vault về pot
∧ lamp_to_addr_tagged(pot, owner_commit) ≥ q·oil -- ĐÍCH pot GẮN-TAG (apply-param), KHÔNG ví keeper
∧ nonlamp_preserved ∧ only_expected_policies
```
**Khác mô-hình cũ:** thu-hồi MỖI epoch idle (tức-thời `q = min(5,c) → pot`), KHÔNG chờ gap-1001-epoch. Phần đã-sở-hữu `o = D−c−r` bất-biến (đã ra ví Phoenix — vốn NGOÀI vault). **Auth 2 nhánh:** keeper attest-idle HOẶC owner tự-ký (escape-hatch — owner chủ-động trả phần chưa-kiếm về pot dù keeper vắng). Nhánh close (`reclaim_epoch_close_ok`): `c′ = 0 ∧ ¬vault_recreated` → burn NFT, LAMP → pot (tag), **min-ADA → owner_address** (không mắc-kẹt). Neo: `reclaim_epoch_ok` (recreate) ; `reclaim_epoch_close_ok` (close).

### 5.4 GenDrip — `gen_drip_ok`
```
c′ = c_in ∧ L(out) = L(in) ∧ L(out) = c′·oil            -- LAMP đứng yên ×oil (I-ACT-7)
∧ r′ = r_in ∧ td′ = td_in ∧ te′ = te_in                 -- Gen chỉ ĐỌC, không đổi trạng-thái
∧ identity_preserved ∧ nonlamp_preserved ∧ only_expected_policies
```
Neo: `gen_drip_ok`. Engine MAGIC (magic_batches) NGOÀI validator này (§3.7 chưa chốt) — GenDrip chỉ chứng "LAMP không rời qua Gen". MAGIC = account-trong-Vault (KHÔNG mint token) — xem §3 ghi-chú. Gen đọc `c` (conditional còn trong vault) làm cơ-số sinh MAGIC.

### 5.5 Mint-gate — `genesis_vault_ok` / `close_vault_ok`
- **GenesisVault:** ĐÚNG 1 movement `+1` dưới own_policy; carrier output well-formed tại `Script(own_policy)`; `name == owner_commit`; khuôn I-ACT-1 (`c ∈ [1,1001]`, `r = 0`, `td = 0`, `te = −1`, `did_commit ≠ ∅`); `lamp_locked == conditional_lamp × oil_per_lamp`; `anchor_controller_ok` (controller DID hiện-tại ký genesis). Neo `genesis_vault_ok`.
- **CloseVault:** PURE-BURN — `list.length(moved) > 0 ∧ ∀ qty < 0` (mọi movement âm). Nối nhánh close khi `c′ = 0` (`own_epoch_close_ok` / `reclaim_epoch_close_ok`, `¬vault_recreated`). Neo `close_vault_ok`.

---

## 6. Không-gian trạng-thái + đồ-thị chuyển (auditor coverage)

```
   PHA-1 (Ngày, n≤1001)                     PHA-2 (Kỳ, n≥1002, 1 Kỳ = 5 ngày)
             GetLAMP        Reclaim×(idle)      OwnEpoch×(active)          ReclaimEpoch×(idle)
   [pot] ───────────► (c=D) ──────────► (c↓) ──q=min(5,c)──► ví-Phoenix-DID   ──q=min(5,c)──► pot
                         │  GenDrip (loop, L bất-biến, đọc c)   (c↓, o↑)          (c↓, r↑)
                         └────────────────────────────────────────┘  → burn NFT khi c=0 (mọi nhánh)
```
Chuyển hợp-lệ (đầy-đủ) — trạng-thái vault sống mô-tả bằng `c` (r là bộ-đếm audit; `o = D−c−r` suy-được, KHÔNG lưu):
- `GenDrip`: mọi pha, `c → c` — self-loop, `L(vault)` bất-biến (đọc `c` sinh MAGIC).
- `Reclaim`: chỉ PHA-1 (`n≤1001`), `c → c−1`, 1 LAMP → **pot** (tag), `r += 1`.
- `OwnEpoch`: chỉ PHA-2 (`n>1001`), epoch ACTIVE + `te+1 ≤ e_now`, `c → c−q` với `q=min(5,c)`, q LAMP → **ví Phoenix DID**, `te += 1`; đóng+burn nếu `c−q=0`.
- `ReclaimEpoch`: chỉ PHA-2, epoch IDLE + `te+1 ≤ e_now` (keeper∨owner), `c → c−q` với `q=min(5,c)`, q LAMP → **pot** (tag), `r += q`, `te += 1`; đóng+burn nếu `c−q=0`.

Chuyển BỊ CẤM (auditor xác nhận REJECT — test âm §… trong `activation_logic.ak`, v5 **69/69 pass**):
`c → user` sai đích (OwnEpoch chỉ về `owner_address`) · L tự-sinh · Reclaim khi n>1001 · OwnEpoch/ReclaimEpoch khi n<1002 · `q > 5` (per-epoch cap) · `q > c` · nhảy epoch `te+1 > e_now` (xử epoch tương-lai) · `te′ ≠ te+1` (gộp-nhảy/xử-lại) · non-keeper ký (trừ ReclaimEpoch owner escape-hatch) · OwnEpoch đích ≠ owner_address (keeper-đoạt) · ReclaimEpoch đích ≠ pot / sai tag · đổi DID/mốc · drain ADA/token-lạ · burn khi c≠0.

---

## 7. Ba định-lý MONEY-SAFETY (+ chứng-minh phác-thảo)

### Định-lý 1 (NO-DRAIN / conservation) — LAMP không tự-sinh, không rò-rỉ khỏi đường hợp-lệ.
> **Phát-biểu.** Với mọi tx spend hợp-lệ trên vault sống, tổng LAMP hệ-thống bảo-toàn: mỗi LAMP rời vault đi ĐÚNG một trong hai đích {pot, ví-Phoenix-của-DID}; mỗi LAMP vào vault chỉ từ GetLAMP (pot). Không redeemer nào tạo LAMP. (Đồng-nhất hard-constraint LAMP 36B không-burn: không đích nào là đốt.)

**Chứng-minh (phác-thảo, quy-nạp theo redeemer).**
1. *Genesis lập bất-biến* (SỔ-VALUE): `genesis_vault_ok` ép `L = c` với `o=0, r=0` ⟹ `D = c + o + r`. LAMP vào vault = D lấy từ pot (pot-validator gác — §8).
2. *Mỗi redeemer giữ (SỔ-VALUE):* các mệnh-đề `L(out) = c′` (§5.1-5.4) ⟹ vault sống luôn giữ đúng `c` LAMP; đồng-thời `Δc + Δo + Δr = 0` giữ `D = c + o + r`. Vậy bất-biến bảo-toàn qua mọi bước.
3. *Δ LAMP rời vault = đích-đúng:*
   - Reclaim: `L(out)=L(in)−1` **và** `lamp_to_addr(pot) ≥ 1` ⟹ đúng 1 LAMP tới pot, không nơi khác (vì `only_expected_policies` + `L(out)` đã kế-toán phần còn lại).
   - ReclaimEpoch: `L(out)=L(in)−q` **và** `lamp_to_addr(pot) ≥ q` ⟹ q LAMP tới pot.
   - OwnEpoch: `L(out)=L(in)−q` **và** `lamp_to_addr(owner_address) ≥ q` ⟹ q LAMP tới ví Phoenix của DID.
   - GenDrip: `L(out)=L(in)` ⟹ 0 LAMP rời.
4. *Không tự-sinh:* không mệnh-đề nào cho `c′ > c` hay `L(out) > L(in)`. `c` chỉ giảm (MONO-c). Bù-trừ per-vault: mỗi bước giữ `Δc + Δo + Δr = 0` ⟹ `D = c + o + r` bất-biến (Reclaim: Δc=−1,Δr=+1; OwnEpoch: Δc=−q,Δo=+q; ReclaimEpoch: Δc=−q,Δr=+q). ∎

**Điểm auditor kiểm:** `lamp_to_addr` cộng LAMP tới ĐÚNG địa-chỉ đích (`==target`), nên `≥ threshold` + `only_expected_policies` chặn "keeper nhét LAMP vào output thứ ba" (LỖ-1 lịch-sử). Reclaim/ReclaimEpoch đích = `pot_address` (apply-param); OwnEpoch đích = `owner_address` (apply-param = ví Phoenix của DID, neo theo DID không theo khoá — rotation-safe).

### Định-lý 2 (OWNED-UNTOUCHABLE) — LAMP đã-sở-hữu bất-khả-xâm vì đã RỜI vault về ví Phoenix của DID.
> **Phát-biểu.** LAMP đã-sở-hữu (`o` luỹ-kế) đã được OwnEpoch chuyển RA ví Phoenix của DID (`owner_address`) — nằm HOÀN-TOÀN ngoài validator vault. Không redeemer nào (Reclaim, OwnEpoch, ReclaimEpoch, GenDrip) chạm được LAMP đã ra ví; không có đường nào kéo nó về vault hay về pot.

**Chứng-minh.**
- Mô-hình mới KHÔNG giữ LAMP đã-sở-hữu trong vault: OwnEpoch chuyển `q` LAMP ra `owner_address` NGAY trong tx active-epoch (`L(out)=L(in)−q·oil`). Ví Phoenix của DID là địa-chỉ user tự kiểm-soát — ngoài phạm-vi validator vault ⟹ keeper/anti-idle/reclaim không có redeemer nào đụng tới. Đây là bảo-đảm MẠNH hơn mô-hình cũ ("vested còn trong vault, bảo-vệ bằng owner-sig").
- Phần đã-sở-hữu `o = D − c − r` là đại-lượng SUY-ĐƯỢC (không field on-chain), chỉ TĂNG (hệ-quả MONO-c §3): OwnEpoch `Δc=−q, Δr=0 ⟹ Δo=+q`; Reclaim/ReclaimEpoch `Δc+Δr=0 ⟹ Δo=0`. Không redeemer nào cho `Δo < 0` — không có đường kéo LAMP đã-ra-ví về lại vault/pot.
- Đích OwnEpoch cứng = `owner_address` (apply-param) ⟹ keeper ký attest-active nhưng KHÔNG chuyển được sang ví keeper.
Vậy LAMP một-khi-đã-sở-hữu = trong tay user, không cơ-chế nào lấy lại. ∎

**Điểm auditor kiểm:** đích cứng OwnEpoch = `owner_address` (không phải arbitrary destination do keeper chọn). So mô-hình cũ: bỏ được redeemer ClaimVested owner-sig + trạng-thái vested-in-vault ⟹ ít bề-mặt hơn, bảo-đảm sở-hữu bằng "vật-lý ngoài vault" thay vì "guard chữ-ký trong vault".

### Định-lý 3 (RECLAIM-ONLY-TO-POT, per-epoch) — thu-hồi PHA-2 chỉ về pot, chỉ phần chưa-kiếm, mỗi epoch idle, ≤ 5/epoch.
> **Phát-biểu.** ReclaimEpoch chỉ hợp-lệ khi `n > 1001 ∧ te+1 ≤ e_now` (epoch KẾ-TIẾP, chưa xử-lý) ∧ (keeper-attest-idle ∨ owner-escape); hệ-quả `c → c−q` với `q=min(5,c)` toàn-bộ tới **pot** (không ví keeper), `o = D−c−r` bất-biến (phần đã ra ví không đụng), `r += q` (audit).

**Chứng-minh.**
- *Mỗi epoch xử ĐÚNG một lần (tuần-tự):* guard `te_in+1 ≤ e_now` + hậu-điều-kiện `te′ = te_in + 1` (KHÔNG `te′ = e_now`). Sau khi xử epoch `j = te_in+1`, `te` tăng đúng 1 ⟹ tx sau đòi `j_next = te+2`; muốn xử-lại `j` phải có `j ≤ te` — REJECT. Vì `te′ = te+1` và guard đòi `te+1 ≤ e_now`, chuỗi xử-lý tiến TUẦN-TỰ 0,1,2,…,e_now, KHÔNG bỏ sót, KHÔNG gộp-nhảy nhiều epoch vào 1 bước-5-LAMP, KHÔNG xử-lại. (Genesis `te=−1` ⟹ epoch đầu `j=0` — fix off-by-one.)
- *≤ 5/epoch (anti-drain):* `q = min(5, c)` ⟹ mỗi tx rút tối-đa 5 LAMP; keeper không thể rút nhiều hơn 1-Kỳ-worth (= 5 ngày × 1 LAMP/ngày) trong một lần.
- *Chỉ về pot:* `L(out)=L(in)−q·oil` **và** `lamp_to_addr_tagged(pot, owner_commit) ≥ q·oil` (§5.3) ⟹ q LAMP tới pot đúng tag (Định-lý 1, mục 3).
- *Chỉ phần chưa-kiếm:* `Δc+Δr=0 ⟹ Δo=0` ⟹ LAMP đã ra ví không bị đụng (Định-lý 2). ReclaimEpoch chỉ rút từ `c` (conditional còn khoá).
- *Không keeper-đoạt:* đích cứng `pot_address` (apply-param); `only_expected_policies` chặn token-lạ. Keeper ký để attest idle (§8 giả-định), owner có escape-hatch tự-trả — cả hai KHÔNG chọn được đích.
∎

**Chú-ý an-toàn (auditor):** khác mô-hình cũ (chờ gap ≥ 1001 epoch mới forfeit toàn-bộ), mô-hình mới thu-hồi TỪNG epoch idle tức-thời `min(5,c)`. Hệ-quả: user idle liên-tục thì `c` giảm 5/epoch cho tới 0 (~ tối-đa ⌈1001/5⌉ ≈ 201 epoch ≈ 1005 ngày để cạn từ D=1001). Đây là chủ-đích: pot tái-tuần-hoàn nhanh, không đọng LAMP-chưa-kiếm 13.7 năm. `q=min(5,c)` bảo-đảm epoch cuối rút phần lẻ còn lại, không âm.

---

## 8. Giả-định tin-cậy (NGOÀI phạm-vi chứng-minh — auditor phải soi riêng)

| # | Giả-định | Rủi-ro nếu vỡ | Trạng-thái |
|---|---|---|---|
| T-1 | **Keeper trung-thực** attest active (OwnEpoch) / idle (Reclaim, ReclaimEpoch). MVP: `keeper_signed`. | Keeper gian dối → ReclaimEpoch oan phần chưa-kiếm (về **pot**, không về keeper — Đ-lý 1/3 chặn đoạt), hoặc OwnEpoch sai epoch (nhưng đích cứng = ví Phoenix của DID, keeper không đoạt). KHÔNG drain được sang keeper. | MVP tin keeper; §6.1 spec: thay bằng consume-event Registry reference-input |
| T-2 | **Pot-validator** (`dist_treasury`) tự gác quyền cấp LAMP đúng D. `genesis_vault_ok` chỉ gác VAULT dựng đúng khuôn + `lamp_locked == conditional_lamp`, KHÔNG gác pot chi đúng. | Pot chi sai D → vault khai-man; nhưng `lamp_locked==conditional_lamp` ràng datum ↔ LAMP thật ⟹ không bịa D mà không khoá LAMP. | pot-validator riêng (§6.4) |
| T-3 | **Uniqueness anchor PersonDID.** D keyed per-PersonDID. Lỗ ở tầng **mã-hoá anchor** (KHÔNG phải sinh-trắc): `GenesisPerson` đúc được anchor did-string bất-kỳ với controller của attacker vì HW_Key P-256 KHÔNG verify on-chain (I-CURVE-4 carry-by-equality). | N anchor-giả → N×D LAMP rút khỏi pot (GV1, xem `-Exec.md` §7). Đóng ở tầng structural/cryptographic (PA2 UniquenessThread), KHÔNG phải chống-sybil-sinh-trắc. | NGOÀI phạm-vi vault; chờ PA2 land — **[CẦN CHỐT]** §9 |
| T-4 | **Engine Gen §3.7** chưa spell-out on-chain. GenDrip validator chỉ ép "LAMP không rời" (I-ACT-7). MAGIC = account-trong-Vault (KHÔNG mint token). | Nếu engine Gen production spend UTXO-LAMP (như code MAGIC cũ fire→Treasury) → vi-phạm I-ACT-7. | **[CẦN CHỐT]** §9 |
| T-5 | **`has_counterparty_consume`** là cổng Registry-gate (I-ACT-3): PHẢI trả về đúng "tiêu ≥ `MIN_MAGIC_CONSUME` qua dịch-vụ **trả phí** Registry đăng-ký hợp-chuẩn" trước khi anti-idle (Ngày) / active-gate (Kỳ) dựa vào nó thay vì chỉ dựa keeper attest. Xem yêu-cầu nối đủ ở §9 mục 6.1. | active/idle không phân-biệt "tiêu-thật qua Registry" on-chain nếu cổng chưa nối đúng; dựa keeper. | Registry-team — xem §9 mục 6.1 |

**Kết-luận phạm-vi:** trong mô-hình tin-cậy {T-1..T-5}, ba định-lý §7 GIỮ. Đặc-biệt: **kể cả keeper ác-ý, LAMP không chảy sang keeper** — mọi đích thu-hồi cứng = pot; mọi đích giao-quyền cứng = ví Phoenix của DID (`owner_address`, apply-param). Đây là tuyến phòng-thủ money-safety mạnh nhất của thiết-kế. **Lưu ý phân-tách:** T-3 là lỗ ANCHOR-uniqueness (mã-hoá), KHÔNG phải lo-ngại sybil-sinh-trắc — sinh-trắc Secure Enclave đủ chống trùng người; lỗ nằm ở anchor did-string không ràng khoá-gốc.

---

## 9. [CẦN CHỐT] còn treo (auditor ghi nhận — chưa đóng)

| # | Mục | Ảnh-hưởng money-safety | Chủ |
|---|---|---|---|
| **§3.7-1** | Spell-out **engine Gen ĐỌC-số-dư on-chain** (đọc VaultDatum reference-input → drip MAGIC → KHÔNG-spend UTXO-LAMP). Code MAGIC-repo cũ fire-LAMP→Treasury (TIÊU-THỤ) LỖI THỜI — nếu tái dùng sẽ vi-phạm I-ACT-7. MAGIC = account-trong-Vault, KHÔNG mint token. | CAO — quyết định I-ACT-7 có giữ ở production không. GenDrip validator hiện chỉ ép bất-biến "LAMP không rời", chưa nối accounting MAGIC. | MAGIC/CARP-team |
| **§3.7-2** | Xác nhận CẢ InstantGen + ScheduleGen /CARP đều đọc-số-dư (không nhánh nào burn LAMP). | CAO — cùng T-4. | xác-nhận /CARP |
| **§3.7-3** | Granularity: anti-idle tick NGÀY (PHA-1) + OwnEpoch/ReclaimEpoch theo EPOCH (PHA-2) + drip MAGIC theo EPOCH (mặc định) hay drip-daily. | Thấp — không đụng conservation; chỉ nhịp MAGIC. | maintainer |
| **GV1/PA2** | **Uniqueness anchor PersonDID** (T-3) — PA2 UniquenessThread đóng lỗ đúc-anchor-did-bất-kỳ ở tầng structural/cryptographic TRƯỚC khi mở GetLAMP-PersonDID production. | CAO — hệ-quả kinh-tế (N×D rút-ròng), KHÔNG đụng 3 định-lý on-chain (validator đúng). | đội Core/on-chain (PA2) |
| **6.1** | **Registry-chuẩn-dịch-vụ** (thay counterparty-gate) — chi-tiết yêu-cầu ở T-5 (§8). Cổng chống-wash = dịch-vụ Registry tiêu tài-nguyên thật. | Trung — không đụng 3 định-lý (keeper vẫn không đoạt tiền); đụng ĐỘNG-CƠ anti-idle (đếm active đúng). | Registry-team |
| **6.5** | Cân-đối tốc-độ-vest-ra (PHA-2 rời-hệ) vs nạp pot. Rủi-ro F: pot cạn khi nhiều user qua PHA-2. | Trung — không phải drain (LAMP-vest đã-kiếm hợp-lệ rời-hệ), mà là thanh-khoản pot onboarding. | LAMP/backend |
| **MIN_MAGIC_CONSUME** | = 10% MAGIC-gen-able từ `conditional_lamp`; ngưỡng tiêu qua dịch-vụ **trả phí** để đánh dấu period ACTIVE. Granularity: PHA-1 per-NGÀY, PHA-2 per-EPOCH (tiêu-đủ trong ≥1 ngày của Kỳ, hay cumulative cả Kỳ). | Thấp — tham-số ngưỡng active, không đụng conservation. | TẠM (07-06) |

---

## 10. Danh-mục kiểm cho auditor (checklist rút gọn)

1. **(SỔ-VALUE)** `L(vault) == c × oil_per_lamp` giữ ở genesis + mọi redeemer? → grep `lamp_out_amt == d_out.conditional_lamp * oil_per_lamp` (ở `gen_drip_ok`, `reclaim_ok`, `own_epoch_ok`, `reclaim_epoch_ok`). Per-vault `D = c + o + r` (`o = D−c−r`, không lưu).
2. **Đích-đúng** mọi đường LAMP-rời: Reclaim/ReclaimEpoch → `pot_address` (tag `owner_commit`); OwnEpoch → `owner_address` (ví Phoenix DID); GenDrip → không rời. → `lamp_to_addr(...)` / `lamp_to_addr_tagged(...) ≥ q·oil` + `only_expected_policies`.
3. **Chữ-ký:** Reclaim/OwnEpoch = keeper (attest active/idle); ReclaimEpoch = keeper HOẶC owner (escape-hatch). Genesis = controller ký (`anchor_controller_ok`). KHÔNG còn ClaimVested owner-sig — sở-hữu đi thẳng ra ví Phoenix của DID (đích cứng ⟹ keeper không đoạt).
4. **Ranh-giới pha:** Reclaim `n ≤ 1001`; OwnEpoch/ReclaimEpoch `n > 1001`. Test: Reclaim-PHA-2 reject (`n13`), OwnEpoch-PHA-1 reject (`oe_nph1`).
5. **Cap per-epoch** `q = min(5, c)` (không rút > 5/epoch). Test: q>5 reject (`oe_nover`), q>c reject.
6. **Tuần-tự epoch** `te+1 ≤ e_now` + `te′ = te+1` (KHÔNG `te′ = e_now`); genesis `te=−1` ⟹ epoch đầu `j=0`. Test: nhảy `te+1 > e_now` reject (`oe_nseq2`), `te′ ≠ te+1` reject (`oe_nseq1`).
7. **Monotonic td** `n > td_in` mọi tick PHA-1 (chống double-tick/tua).
8. **Anti-drain** `nonlamp_preserved` (ADA) + `only_expected_policies` (token-lạ) mọi redeemer.
9. **Close/burn** đúng: `own_epoch_close_ok`/`reclaim_epoch_close_ok` chỉ khi `c′ = 0`, nối `close_vault_ok` pure-burn (`¬vault_recreated`).
10. **Man-thời-gian:** `tx_lo` None (−∞) → REJECT; `days_elapsed` clamp âm → 0.

---

## Phụ-lục A — Bảng đối-chiếu bất-biến ↔ hàm (khớp code v5, neo dòng thật)

| Bất-biến | Hàm (`activation_logic.ak`) | Dòng |
|---|---|---|
| days_elapsed / p2_epoch / epoch_release | `days_elapsed` / `p2_epoch` / `epoch_release` | 148 / 161 / 171 |
| (SỔ-VALUE) L = c×oil ; per-vault D=c+o+r | genesis; gen; reclaim; own_epoch; reclaim_epoch | — |
| I-ACT-1 genesis khuôn (`te=−1`, `L=c×oil`) | `genesis_vault_ok` | 645 |
| I-ACT-4 reclaim (PHA-1) | `reclaim_ok` | 339 |
| I-ACT-7 gen-drip | `gen_drip_ok` | 581 |
| I-ACT-8 OwnEpoch (active, →ví Phoenix DID) | `own_epoch_ok` (recreate) / `own_epoch_close_ok` (close) | 399 / 448 |
| I-ACT-8b ReclaimEpoch (idle, →pot) | `reclaim_epoch_ok` (recreate) / `reclaim_epoch_close_ok` (close) | 481 / 536 |
| mint-gate genesis/close | `genesis_vault_ok` / `close_vault_ok` | 645 / 693 |
| dispatch spend/mint (+ guard double-satisfaction) | `activation_vault.ak` (mint handler + spend `when redeemer`, 4 spend redeemer) | — |

---

## Nguồn

- Code (nguồn chân-lý, nhánh `claude/wakeme-5lamp-epoch`): `PhoenixKey-Validator/lib/phoenixkey/activation_logic.ak` (1796 dòng, 69 test), `PhoenixKey-Validator/validators/activation_vault.ak` (280 dòng, 1 test double-satisfaction), `PhoenixKey-Validator/lib/phoenixkey/auth_logic.ak` (`anchor_controller_ok`). `aiken check` 2026-07-12: **216/216 pass** toàn repo.
- Nguồn thiết-kế nội-bộ (không công khai). (đã qua rà-soát nội-bộ)
- Đánh-giá gaming nội-bộ (không công khai) (GV1/GV2 khung anchor-uniqueness + wash-Registry).
- Cross-ref canonical: `PhoenixKey-Specs/PhoenixKey-Math.md` (key hierarchy §6, TAAD §10, type catalog §12–21).
- Tài-liệu cùng bộ: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md), [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md), [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)
