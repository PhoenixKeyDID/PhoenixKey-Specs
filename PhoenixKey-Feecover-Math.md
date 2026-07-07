# PhoenixKey — Feecover MATH (đặc-tả hình-thức cho AUDITOR)

> Trạng thái: SPEC PROPOSAL · 2026-07-07. Bản dành riêng cho **auditor**: định nghĩa hình thức, bất biến neo-file, công thức phí + quy-đổi MAGIC↔CARP, và **chứng minh conservation** (phí không tự-sinh/mất; `carp_paid == fee_magic`; Treasury nhận đúng Σ).
>
> **Đọc kèm:** `PhoenixKey-Feecover-Feat-Math.md` (§ toán, bất biến FEECOVER-*, C-CM-*), `MAGIC-paymaster/ConsumeMAGIC/MATH.md` (§1–5), `MAGIC-paymaster/ConsumeMAGIC/onchain/validators/consume.ak` (C-CM-1..5, đã đối chiếu byte).
>
> **Ranh giới cứng:** Feecover KHÔNG chạm `consume.ak` (C-CM-1..5) và KHÔNG chạm bất biến CARP-lõi (INV-PEG-BY-DEMAND §8.1, INV-NO-LAMP-PEG-DEFENSE F9, `1 CARP = 1 MAGIC`). Mọi định-lý dưới đây **giả định** C-CM-1..5 đúng (đã audit ở ConsumeMAGIC) và chỉ chứng minh phần Feecover THÊM.

---

## 1. Ký hiệu (formal symbols)

### 1.1 Đơn vị + hằng số

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `Q` | `10^9` — scale factor Q-format | (không thứ nguyên) | `ConsumeMAGIC/MATH.md §1` |
| nanogic | Đơn vị nhỏ nhất MAGIC: `1 MAGIC = 10^9 nanogic` | nanogic | `MATH.md §1` |
| — | `1 nanogic = 1 byte·ngày` lưu-lạnh → `1 MAGIC = 1 GB·ngày` | byte·ngày | Whitepaper §4 |
| thread | Đơn vị nhỏ nhất CARP; neo `1 thread = 1 nanogic` | thread | Feat §3.1 |
| — | Neo hiến pháp: `1 CARP = 1 MAGIC` (sức-mua nội sinh, KHÔNG neo fiat) | — | Whitepaper §5 |

### 1.2 Biến giá + phí

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `t` | `op_type` (nghiệp vụ generic trong ConsumeMAGIC) | — | `consume.ak:47` |
| `s` | `service_id` (định danh dịch vụ PhoenixKey, vd `did.anchor`) | ByteArray | Feat §3.1 |
| `base_price[t]` | Giá danh nghĩa op `t`, governance param, `≥ 0` | nanogic | `pricing.ak:valid_param` |
| `demand_mult` | Hệ số cung-cầu, Q-format, `∈ [m_min, m_max]` | Q-format | `MATH.md §2.4` |
| `price(t)` | Giá đơn vị op `t` (từ beacon `PriceParam`) | nanogic | `MATH.md §2.1` |
| `n` | `op_count` của 1 vault input (`≥ 1`) | — | `consume.ak:48` |
| `required(t,n)` | Tổng nanogic cần đốt cho `n` op loại `t` | nanogic | `MATH.md §2.2` |
| `magic_burned` | `−mint(MAGIC)` trong tx (`> 0`) | nanogic | `consume.ak:66` |
| **`fee_magic[s]`** | **Phí danh nghĩa dịch vụ `s`** (bảng Feecover) | nanogic | **Feat §3.1** |
| **`magic_consumed`** | Lượng MAGIC THẬT một tx consume giảm cho op này | nanogic | `consume.ak` (= required) |
| **`carp_paid`** | Lượng CARP user trả trong tx Feecover | thread | **Feat §3.3** |
| `σ(s)` | Ánh xạ dịch vụ→op: `service_id s ↦ op_type t` | — | §8.3 (đồng bộ 2 bảng) |

### 1.3 Kế toán gom (accrual) + settle

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `A_App(e)` | `FeecoverAccrual.carp_accrued`, tầng App, epoch `e` | thread | Feat §3.3 |
| `A_Plat(e)` | `FeecoverAccrual.carp_accrued`, tầng Platform, epoch `e` | thread | Feat §3.3 |
| `B_s(e)` | `breakdown[s]` — Σ CARP gom cho dịch vụ `s` trong epoch `e` | thread | Feat §3.3 |
| `T(e)` | `carp_to_treasury` sau khi settle epoch `e` (append-only) | thread | Feat §3.4 |
| `L_settled` | `last_settled_epoch` (monotonic +1) | epoch | Feat §3.4 |
| `𝕋x(e)` | Tập mọi tx Feecover hợp lệ đóng trong epoch `e` | (tập) | — |
| `e` | Chỉ số epoch hiện hành (`current_epoch`) | epoch | `consume.ak:52` |

---

## 2. Công thức phí + quy-đổi MAGIC↔CARP

### 2.1 Giá đơn vị (thừa kế ConsumeMAGIC, KHÔNG viết lại)

```
price(t) = ⌊ base_price[t] × demand_mult / Q ⌋           (nanogic)        [F-1]
required(t, n) = price(t) × n                              (nanogic)        [F-2]
```

Floor division BigInt (không float). Đơn điệu không-giảm theo `demand_mult`. Nguồn `pricing.ak:price_of`, `MATH.md §2.1–2.2`.

### 2.2 Phí Feecover (danh nghĩa, zero-oracle)

Với dịch vụ `s`, phí danh nghĩa lấy TRỰC TIẾP từ beacon `ServiceFeeSchedule` (KHÔNG nhân `demand_mult` lại — phí là hằng danh nghĩa cho tới khi DAO re-anchor):

```
fee_magic[s]  =  entry(s).fee_magic     nếu entry(s).enabled       (nanogic)   [F-3]
              =  ⊥ (fail)               nếu entry vắng/disabled
```

### 2.3 Ràng buộc khớp op (đồng bộ 2 bảng — §8.3)

Feecover ép lượng MAGIC THẬT `consume.ak` giảm phải bằng đúng phí danh nghĩa:

```
magic_consumed  =  required(σ(s), 1)  ==  fee_magic[s]                         [F-4]
```

`[F-4]` buộc `base_price` (ConsumeMAGIC) và `fee_magic` (Feecover) đồng bộ qua governance cùng cadence (§8.3). Lệch hai bảng ⇒ `[F-4]` fail (an-toàn-đóng, không drain).

### 2.4 Quy-đổi MAGIC → CARP (neo 1:1)

Vì `1 CARP = 1 MAGIC` (hiến pháp) và `1 thread = 1 nanogic`, phép quy-đổi là **hàm đồng-nhất số học**:

```
carp_paid  ==  fee_magic[s]              (thread ≡ nanogic, tỉ giá 1)         [F-5]
```

**Không có bước tỷ-giá, không oracle.** `[F-5]` = bất biến FEECOVER-PAY-1. Thiếu/thừa ⇒ fail.

### 2.5 Gom per-tx (accrual)

Với tx `x` phục vụ dịch vụ `s(x)`, tầng App của provider:

```
A_App(e) ← A_App(e) + carp_paid(x)                                            [F-6]
B_{s(x)}(e) ← B_{s(x)}(e) + carp_paid(x)                                       [F-7]
```

`carp_accrued` **chỉ tăng trong epoch** (FEECOVER-PAY-2), reset sau settle.

### 2.6 Settle cuối epoch → Phoenix Treasury

```
epoch_settle(e_c):
  require  e_c == L_settled + 1                          (tuần tự)  [F-8]  (FEECOVER-SETTLE-1)
  total_carp(e_c) = Σ_{Platform} A_Plat(e_c)                        [F-9]
  chuyển total_carp(e_c) CARP → Phoenix Treasury (địa chỉ hiến định)
  T(e_c) = T(e_c−1) + total_carp(e_c)                    (append-only) [F-10] (FEECOVER-SETTLE-3)
  L_settled ← e_c
  reset A_Plat(e_c), A_App(e_c) ← 0
```

Chỉ **CARP** rời hệ về Treasury — KHÔNG LAMP, KHÔNG MAGIC (FEECOVER-SETTLE-2).

---

## 3. Bảng bất biến (neo file + dòng)

### 3.1 Tái-dùng nguyên trạng từ ConsumeMAGIC (KHÔNG sửa)

| ID | Bất biến | Neo |
|---|---|---|
| C-CM-1 | Value non-MAGIC (ADA/LAMP/CARP) trong Vault bảo toàn byte-perfect: `Σ v(out@script) − Σ v(in@script)` zero sau khi trừ thành phần MAGIC | `consume.ak:101,244–257 non_magic_value_preserved` |
| C-CM-2 | `magic_burned ≥ total_required`; giá LẤY TỪ BEACON `PriceParam`, KHÔNG tin amount client | `consume.ak:56,71,73`; `sum_required_over_vault_inputs:142` |
| C-CM-3 | Chống double-satisfaction N-vault-input: `#out@script == #in@script` ∧ `Σ engageNFT(out)==Σ engageNFT(in)` | `consume.ak:85–92` |
| C-CM-4 | Thread NFT bảo toàn + state tăng đúng tổng: `Σ consumed_out == Σ consumed_in + Σ op_count`; `last_epoch = current`; owner giữ | `consume.ak:105–108,300–331 enforce_engagement` |
| C-CM-5 | Stale-price gate: `0 ≤ current_epoch − pp.epoch ≤ max_price_stale` | `consume.ak:60–61` |
| — | `base_price ≥ 0` (giá âm ⇒ required âm ⇒ drain miễn phí) | `pricing.ak:valid_param`; `MATH.md §2.5` |

### 3.2 Feecover THÊM (Feecover-only, không là input của `consume.ak`)

| ID | Bất biến | Neo |
|---|---|---|
| FEECOVER-GATE-1 | Account owner gắn PhoenixKey DID hợp lệ point-in-time: `did_active(did, e)=active` ∧ (nếu ký khoá) `key_authorized(key, did, e)`; đọc qua reference input | Feat §5; VeData §2.1 |
| FEECOVER-GATE-2 | Freshness cổng: `current_epoch − FeecoverGate.epoch_valid_until ≤ 0` (mẫu C-CM-5) | Feat §5 |
| FEECOVER-GATE-3 | DID credential đọc qua **reference input** (CIP-31), KHÔNG tiêu/KHÔNG mint trong tx Feecover (chống tự-đúc DID) | Feat §5 |
| FEECOVER-PAY-1 | `carp_paid == fee_magic[s]` (neo 1:1 thread↔nanogic); thiếu/thừa → fail `[F-5]` | Feat §6.4 |
| FEECOVER-PAY-1b | `magic_consumed == fee_magic[s]` = `required(σ(s),1)` `[F-4]` (khớp op, đồng bộ 2 bảng) | Feat §7-E; §8.3 |
| FEECOVER-PAY-2 | CARP chỉ accrue đúng `(tier, epoch)`; `carp_accrued` chỉ tăng trong epoch, reset sau settle | Feat §6.4 |
| FEECOVER-SETTLE-1 | `epoch_settle` tuần tự: `e_c == L_settled + 1` (không bỏ/nhảy/double-settle) `[F-8]` | Feat §6.4 |
| FEECOVER-SETTLE-2 | Chỉ **CARP** về Phoenix Treasury; KHÔNG LAMP/MAGIC | Feat §6.4 |
| FEECOVER-SETTLE-3 | `carp_to_treasury / deposited_L3 / topup_requested` append-only (chỉ tăng) `[F-10]` | Feat §6.4 |
| FEECOVER-HO-1 | `deposit` chỉ chuyển CARP ĐÃ ở Phoenix Treasury; release qua Governance `status==Executed` | Feat §6.4 |
| FEECOVER-HO-2 | `stabilizer_ref` chỉ đổi qua Governance | Feat §6.4 |
| FEECOVER-HO-3 | Feecover KHÔNG mint/burn CARP/LAMP; KHÔNG đọc oracle giá (giữ INV-PEG-BY-DEMAND) | Feat §6.4 |

### 3.3 CARP-lõi KHÔNG chạm (giữ nguyên, chỉ liệt để auditor kiểm biên)

| ID | Bất biến | Neo |
|---|---|---|
| INV-PEG-BY-DEMAND | Peg-core giữ bởi cầu-dịch-vụ THỰC (utility-floored), oracle-free | Whitepaper §8.1/§8.3 |
| INV-NO-LAMP-PEG-DEFENSE | Tuyệt đối không đỡ-peg bằng LAMP | Whitepaper §10 F9 |
| `1 CARP = 1 MAGIC` | Sức mua nội sinh, bất biến hiến pháp | Whitepaper §5 |

---

## 4. Định-lý conservation (chứng minh)

Ký hiệu tập tx epoch `e`: `𝕋x(e) = { x : x hợp lệ, đóng trong e }`. Mỗi `x` mang `carp_paid(x)`, `fee_magic(x) = fee_magic[s(x)]`, `magic_consumed(x)`.

---

### Định-lý T1 (Peg per-tx — phí không tự-sinh/mất trong 1 tx)

**Mệnh đề.** Với mọi tx Feecover hợp lệ `x`:
`carp_paid(x) = magic_consumed(x) = fee_magic[s(x)]`.

**Chứng minh.**
1. `x` hợp lệ ⇒ thoả FEECOVER-PAY-1b `[F-4]`: `magic_consumed(x) == fee_magic[s(x)]`.
2. `x` hợp lệ ⇒ thoả FEECOVER-PAY-1 `[F-5]`: `carp_paid(x) == fee_magic[s(x)]`.
3. Từ (1),(2) bằng bắc cầu: `carp_paid(x) = magic_consumed(x) = fee_magic[s(x)]`. ∎

**Hệ quả.** Không tồn tại tx hợp lệ nào trả CARP lệch lượng MAGIC tiêu (chống gaming §7-E). Một trong hai vế lệch ⇒ tx fail (an-toàn-đóng).

---

### Định-lý T2 (Accrual conservation — gom đúng Σ trong epoch)

**Mệnh đề.** Cuối epoch `e` (trước settle), với mọi tầng Platform gộp hết App:
`Σ_Platform A_Plat(e) = Σ_{x ∈ 𝕋x(e)} carp_paid(x)`.

**Chứng minh.**
1. Khởi tạo epoch: `A_App(e) = A_Plat(e) = 0` (reset sau settle epoch trước, `[F-10]` bước reset).
2. Mỗi `x ∈ 𝕋x(e)` áp đúng một lần `[F-6]`: `A_App ← A_App + carp_paid(x)` (FEECOVER-PAY-2 buộc accrue đúng `(tier,epoch)`, đúng một lần — thread NFT accrual chống replay, cùng khuôn C-CM-4).
3. App→Platform là phép cộng dồn không mất-mát (Platform gom mọi App, mỗi App gom trong `e`):
   `Σ_Platform A_Plat(e) = Σ_App A_App(e) = Σ_{x} carp_paid(x)`.
4. `carp_accrued` chỉ tăng trong epoch (FEECOVER-PAY-2) ⇒ không có trừ ẩn ⇒ tổng bảo toàn. ∎

**Hệ quả (breakdown).** `Σ_s B_s(e) = Σ_{x} carp_paid(x)` (mỗi tx cộng đúng 1 lần vào `B_{s(x)}` `[F-7]`). Attribution phân-rã-theo-dịch-vụ khớp tổng.

---

### Định-lý T3 (Treasury nhận đúng Σ — settle bảo toàn, không double)

**Mệnh đề.** Với mọi epoch đã settle `e_c`:
`T(e_c) − T(e_c−1) = Σ_{x ∈ 𝕋x(e_c)} carp_paid(x)`,
và Treasury nhận đúng CARP đó, KHÔNG LAMP/MAGIC, không lần nào bị settle hai lần.

**Chứng minh.**
1. `epoch_settle(e_c)` chạy ⇒ `[F-8]` (FEECOVER-SETTLE-1): `e_c == L_settled + 1`. Vì `L_settled` monotonic +1, mỗi epoch settle **đúng một lần** (không double, không nhảy).
2. `total_carp(e_c) = Σ_Platform A_Plat(e_c)` `[F-9]` `= Σ_{x} carp_paid(x)` (Định-lý T2).
3. `[F-10]`: `T(e_c) = T(e_c−1) + total_carp(e_c)` ⇒ `T(e_c) − T(e_c−1) = Σ_{x} carp_paid(x)`.
4. FEECOVER-SETTLE-2 ⇒ tài sản chuyển là CARP thuần (LAMP/MAGIC bị chặn ở vị từ settle).
5. `[F-10]` append-only (FEECOVER-SETTLE-3) ⇒ `T` không giảm ⇒ không rút ngược khỏi Treasury qua đường settle. ∎

---

### Định-lý T4 (Global conservation — end-to-end, không rò)

**Mệnh đề.** Qua chuỗi epoch `1..E` đã settle liên tục:
`T(E) = Σ_{e=1}^{E} Σ_{x ∈ 𝕋x(e)} carp_paid(x) = Σ_{e=1}^{E} Σ_{x ∈ 𝕋x(e)} magic_consumed(x)`.

**Chứng minh.**
1. Telescoping T3 qua `e = 1..E` (settle tuần tự `[F-8]`, `T(0)=0`):
   `T(E) = Σ_{e=1}^{E} (T(e) − T(e−1)) = Σ_{e=1}^{E} Σ_{x} carp_paid(x)`.
2. Áp T1 cho mọi `x`: `carp_paid(x) = magic_consumed(x)` ⇒ vế phải thứ hai. ∎

**Diễn giải cho auditor.** Tổng CARP về Treasury bằng **đúng** tổng MAGIC đã tiêu (đo bằng đơn-vị-kế-toán) trên toàn dòng đời — phí không tự-sinh (nếu sinh, T1 vỡ ở tx nào đó) và không tự-mất (nếu mất, T2/T3 vỡ ở accrual/settle). Neo `1:1` `[F-5]` là bản-lề chuyển đơn-vị-đo (MAGIC) sang đơn-vị-thanh-toán (CARP) không hao.

---

### Định-lý T5 (Vault non-drain — Feecover không rút value khỏi Vault)

**Mệnh đề.** Nhánh CARP của Feecover KHÔNG làm giảm value non-MAGIC trong Vault account.

**Chứng minh.**
1. CARP-fee đến từ **input riêng của user** (ví ngoài Vault-account-script), chảy vào `FeecoverAccrual` — KHÔNG rút từ Vault UTxO.
2. C-CM-1 (`non_magic_value_preserved`, `consume.ak:101`) ép value non-MAGIC tại `script@Vault` bảo toàn byte-perfect qua consume; CARP nếu nằm trong Vault thuộc lớp non-MAGIC ⇒ được C-CM-1 bảo vệ.
3. Feecover KHÔNG là input của `consume.ak` (P-LAYER-1, Feat §2) ⇒ không nới C-CM-1.
∴ mọi CARP Feecover di chuyển là CARP-input-user, không phải Vault-value. ∎

---

### Bổ đề T6 (Firewall — Feecover không chạm peg CARP-lõi)

FEECOVER-HO-3: Feecover KHÔNG mint/burn CARP/LAMP, KHÔNG đọc oracle. ⇒ Không có input nào từ Feecover vào INV-PEG-BY-DEMAND / INV-NO-LAMP-PEG-DEFENSE / `1 CARP=1 MAGIC`. **Kiểm tra auditor:** grep validator Feecover không import module CARP peg/oracle; `deposit`/`request_topup` chỉ ghi sổ + trỏ `stabilizer_ref` (L3), không tự kéo tài sản.

---

## 5. Tham số (parameters)

| Tham số | Ý nghĩa | Nguồn / [CẦN CHỐT] |
|---|---|---|
| `Q = 10^9` | Scale Q-format | cố định, `MATH.md §1` |
| `fee_magic[s]` | Phí mỗi dịch vụ (nanogic) | **[CẦN CHỐT D1]** — bảng §4.1 Feat là minh hoạ |
| `max_stale` cổng | Freshness `ServiceFeeSchedule`/`FeecoverGate` | mẫu C-CM-5; **[CẦN CHỐT D2]** cadence |
| `σ(s)` | Ánh xạ service_id→op_type | **[CẦN CHỐT D6]** cơ chế đồng bộ 2 bảng (§8.3) |
| tập `s` phủ | did.anchor/did.rotate/vc.verify/custody.op | **[CẦN CHỐT D3]** Feat §9 |
| ranh giới App/Platform | tầng gom accrual | **[CẦN CHỐT D4]** Feat §9 |
| `stabilizer_ref` | địa chỉ L3 + chữ ký interface | **[CẦN CHỐT D5]** Feat §9 |
| Phoenix Treasury addr | địa chỉ hiến định nhận CARP | hiến định, chờ neo on-chain |

---

## 6. [CẦN CHỐT] treo (blocker / dependency)

- **[CẦN CHỐT] MAGIC-model reconcile.** Toàn bộ định-lý giả định mô hình 3-token đúng (MAGIC = số-dư-account trong Vault, non-transferable; CARP = token thanh-toán). Nếu MAGIC-model đảo lại (từng có tranh chấp native vs account, memory `magic-not-native-token`), `[F-5]` neo 1:1 và T1 cần soát lại vế `magic_consumed`. Chốt canonical: Whitepaper Tokenomic §4–5.
- **[CẦN CHỐT] CARP policy-id.** `carp_paid`/`A_*(e)`/`T(e)` đo bằng "thread CARP" nhưng **policy-id CARP chưa chốt on-chain**. Auditor không thể ràng buộc "CARP thật" trong tx cho tới khi có `carp_policy` cố định. Trước đó FEECOVER-PAY-1/SETTLE-2 chỉ verify được ở mức lượng, chưa verify được asset-identity. Blocker on-chain.
- **[CẦN CHỐT] did_commit attribution (B3).** `EngageDatum` **ĐÃ CÓ** field `did_commit : ByteArray` (field 4, append-only — `types.ak:50` bản canonical `MAGIC/ConsumeMAGIC`), validator **ĐÃ ép immutable** (`consume.ak:409`). MVP nội-dung = `#""`. Blocker thật **KHÔNG phải thêm field** mà là **enforce NỘI DUNG per-DID**: MAGIC-team + Long ràng `did_commit = blake2b256(DID‖salt)` + verify signer khớp DID qua resolve point-in-time. Trước đó FEECOVER-GATE-1 + breakdown per-DID (Hệ quả T2) chỉ đạt attribution-thô-theo-owner → một owner mở nhiều account né trần (Feat §7-A hở).
