# PhoenixKey — Feecover (MATH, đặc tả hình thức cho AUDITOR)

> **Module:** Feecover (fee-abstraction).
> **Loại doc:** Math — định nghĩa hình thức, bất biến neo-file, công thức phí + quy đổi MAGIC↔CARP, chứng minh conservation.
> **Ngày:** 2026-07-09.
> **Đối tượng đọc:** auditor, kiểm thử hình thức.
> **Phạm vi chứng minh:** mọi định lý dưới đây chứng minh phần Feecover THÊM, **giả định** C-CM-1..5 (ConsumeMAGIC) đúng (đã audit ở nguồn) và KHÔNG chạm chúng.

Tài liệu cùng bộ: [PhoenixKey-Feecover-Vi-Feat.md](./PhoenixKey-Feecover-Vi-Feat.md), [PhoenixKey-Feecover-Tech.md](./PhoenixKey-Feecover-Tech.md), [PhoenixKey-Feecover-Exec.md](./PhoenixKey-Feecover-Exec.md).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

> **Ranh giới cứng:** Feecover KHÔNG chạm `consume.ak` (C-CM-1..5) và KHÔNG chạm bất biến CARP-lõi (INV-PEG-BY-DEMAND, INV-NO-LAMP-PEG-DEFENSE, `1 CARP = 1 MAGIC`). Feecover đọc `PriceParam` + `ServiceFeeSchedule` + `FeecoverGate` + DID credential dạng **reference input**, spend `FeecoverAccrual` của chính nó, CÙNG tx với consume.

> **Namespace bất biến:** module này dùng DUY NHẤT tiền tố `FEECOVER-*` (đã ăn sâu 4 doc + gaming eval + code plan). KHÔNG dùng mã `I-FEE-n` — mã đó thuộc `PhoenixKey-Math.md §36` cho một **kiến trúc phí KHÁC** (phí giao thức PhoenixKey per-operation, split 30/70 Cardano/Phoenix Treasury, đo bằng Lovelace), KHÔNG phải Feecover (fee-abstraction MAGIC→CARP). Hai lớp phí là hai module tách bạch; ranh giới chờ maintainer chốt (xem §9).

---

## 1. Ký hiệu (notation)

### 1.1 Đơn vị + hằng số

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `Q` | `10^9` — scale factor Q-format | (không thứ nguyên) | `ConsumeMAGIC/MATH.md §1` |
| nanogic | Đơn vị nhỏ nhất MAGIC: `1 MAGIC = 10^9 nanogic` | nanogic | `MATH.md §1` |
| — | `1 nanogic = 1 byte·ngày` lưu lạnh → `1 MAGIC = 1 GB·ngày` | byte·ngày | Whitepaper §4 |
| thread | Đơn vị nhỏ nhất CARP; neo `1 thread = 1 nanogic` | thread | Feat §4.1 |
| — | Neo hiến định: `1 CARP = 1 MAGIC` (sức mua nội sinh, KHÔNG neo fiat) | — | Whitepaper §5 |

### 1.2 Biến giá + phí

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `t` | `op_type` (nghiệp vụ generic trong ConsumeMAGIC) | — | `consume.ak:47` |
| `s` | `service_id` (định danh dịch vụ PhoenixKey, vd `did.anchor`) | ByteArray | Tech §3.1 |
| `base_price[t]` | Giá danh nghĩa op `t`, governance param, `≥ 0` | nanogic | `pricing.ak:valid_param` |
| `demand_mult` | Hệ số cung cầu, Q-format, `∈ [m_min, m_max]` | Q-format | `MATH.md §2.4` |
| `price(t)` | Giá đơn vị op `t` (từ beacon `PriceParam`) | nanogic | `MATH.md §2.1` |
| `n` | `op_count` của 1 vault input (`≥ 1`) | — | `consume.ak:48` |
| `required(t,n)` | Tổng nanogic cần đốt cho `n` op loại `t` | nanogic | `MATH.md §2.2` |
| **`fee_magic[s]`** | **Phí danh nghĩa dịch vụ `s`** (bảng Feecover) | nanogic | **Tech §3.1** |
| **`magic_consumed`** | Lượng MAGIC THẬT một tx consume giảm cho op này | nanogic | `consume.ak` (= required) |
| **`carp_paid`** | Lượng CARP user trả trong tx Feecover | thread | **Tech §3.3** |
| `σ(s)` | Ánh xạ dịch vụ→op: `service_id s ↦ op_type t` | — | §7.3 (đồng bộ 2 bảng) |

### 1.3 Kế toán gom (accrual) + settle

| Ký hiệu | Định nghĩa | Đơn vị | Nguồn |
|---|---|---|---|
| `A_App(e)` | `FeecoverAccrual.carp_accrued`, tầng App, epoch `e` | thread | Tech §3.3 |
| `A_Plat(e)` | `FeecoverAccrual.carp_accrued`, tầng Platform, epoch `e` | thread | Tech §3.3 |
| `B_s(e)` | `breakdown[s]` — Σ CARP gom cho dịch vụ `s` trong epoch `e` | thread | Tech §3.3 |
| `T(e)` | `carp_to_treasury` sau khi settle epoch `e` (append-only) | thread | Tech §3.4 |
| `L_settled` | `last_settled_epoch` (monotonic +1) | epoch | Tech §3.4 |
| `𝕋x(e)` | Tập mọi tx Feecover hợp lệ đóng trong epoch `e` | (tập) | — |
| `e` | Chỉ số epoch hiện hành (`current_epoch`) | epoch | `consume.ak:52` |

---

## 2. Định nghĩa hình thức các hàm/tập

### 2.1 Giá đơn vị (thừa kế ConsumeMAGIC, KHÔNG viết lại)

```
price(t) = ⌊ base_price[t] × demand_mult / Q ⌋           (nanogic)        [F-1]
required(t, n) = price(t) × n                              (nanogic)        [F-2]
```

Floor division BigInt (không float). Đơn điệu không giảm theo `demand_mult`. Nguồn `pricing.ak:price_of`, `MATH.md §2.1–2.2`.

### 2.2 Phí Feecover (danh nghĩa, zero-oracle)

Với dịch vụ `s`, phí danh nghĩa lấy TRỰC TIẾP từ beacon `ServiceFeeSchedule` (KHÔNG nhân `demand_mult` lại — phí là hằng danh nghĩa cho tới khi DAO re-anchor):

```
fee_magic[s]  =  entry(s).fee_magic     nếu entry(s).enabled       (nanogic)   [F-3]
              =  ⊥ (fail)               nếu entry vắng/disabled
```

### 2.3 Ràng buộc khớp op (đồng bộ 2 bảng — §7.3)

Feecover ép lượng MAGIC THẬT `consume.ak` giảm phải bằng đúng phí danh nghĩa:

```
magic_consumed  =  required(σ(s), 1)  ==  fee_magic[s]                         [F-4]
```

`[F-4]` buộc `base_price` (ConsumeMAGIC) và `fee_magic` (Feecover) đồng bộ qua governance cùng cadence (§7.3). Lệch hai bảng ⇒ `[F-4]` fail (an toàn đóng, không drain).

### 2.4 Quy đổi MAGIC → CARP (neo 1:1)

Vì `1 CARP = 1 MAGIC` (hiến định) và `1 thread = 1 nanogic`, phép quy đổi là **hàm đồng nhất số học**:

```
carp_paid  ==  fee_magic[s]              (thread ≡ nanogic, tỉ-giá 1)         [F-5]
```

**Không có bước tỷ giá, không oracle.** `[F-5]` = bất biến FEECOVER-PAY-1. Thiếu/thừa ⇒ fail.

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
  require  e_c == L_settled + 1                          (tuần-tự)  [F-8]  (FEECOVER-SETTLE-1)
  total_carp(e_c) = Σ_{Platform} A_Plat(e_c)                        [F-9]
  chuyển total_carp(e_c) CARP → Phoenix Treasury (địa-chỉ hiến-định)
  T(e_c) = T(e_c−1) + total_carp(e_c)                    (append-only) [F-10] (FEECOVER-SETTLE-3)
  L_settled ← e_c
  reset A_Plat(e_c), A_App(e_c) ← 0
```

Chỉ **CARP** rời hệ về Treasury — KHÔNG LAMP, KHÔNG MAGIC (FEECOVER-SETTLE-2).

---

## 3. Bất biến sổ sách LÕI (conservation)

Bất biến bảo toàn nền của Feecover: **phí không tự sinh, không tự mất** trên toàn dòng đời. Diễn đạt hình thức:

```
∀ epoch 1..E đã settle liên-tục:
  T(E)  =  Σ_{e=1}^{E} Σ_{x ∈ 𝕋x(e)} carp_paid(x)
        =  Σ_{e=1}^{E} Σ_{x ∈ 𝕋x(e)} magic_consumed(x)                       [CONS]
```

`[CONS]` = kết quả Định lý T4 (§6). Neo `1:1` `[F-5]` là bản lề chuyển đơn vị đo (MAGIC) sang đơn vị thanh toán (CARP) không hao. Firewall CARP-lõi (T6) đảm bảo Feecover KHÔNG tạo/huỷ CARP ngoài luồng phí.

---

## 4. Bảng bất biến (mã hoá + mô tả + neo dòng code nếu có)

### 4.1 Tái dùng nguyên trạng từ ConsumeMAGIC (KHÔNG sửa — aliasing, nguồn `ConsumeMAGIC/MATH.md`)

| ID | Bất biến | Neo |
|---|---|---|
| C-CM-1 | Value non-MAGIC (ADA/LAMP/CARP) trong Vault bảo toàn byte-perfect | `consume.ak:101,244–257 non_magic_value_preserved` |
| C-CM-2 | `magic_burned ≥ total_required`; giá LẤY TỪ BEACON `PriceParam`, KHÔNG tin amount client | `consume.ak:56,71,73`; `sum_required_over_vault_inputs:142` |
| C-CM-3 | Chống double-satisfaction N-vault-input: `#out@script == #in@script` ∧ `Σ engageNFT(out)==Σ engageNFT(in)` | `consume.ak:85–92` |
| C-CM-4 | Thread NFT bảo toàn + state tăng đúng tổng; `last_epoch = current`; owner giữ | `consume.ak:105–108,300–331 enforce_engagement` |
| C-CM-5 | Stale-price gate: `0 ≤ current_epoch − pp.epoch ≤ max_price_stale` | `consume.ak:60–61` |
| — | `base_price ≥ 0` (giá âm ⇒ required âm ⇒ drain miễn phí) | `pricing.ak:valid_param`; `MATH.md §2.5` |

### 4.2 Feecover THÊM (`FEECOVER-*` — Feecover-only, KHÔNG là input của `consume.ak`)

> Bộ mã chính thức module (namespace `FEECOVER-*`).

**P-LAYER-1 [định nghĩa, thêm 2026-07-12 — trước đây được Định lý T5 §6 trích dẫn nhưng chưa
từng định nghĩa ở đâu trong tài liệu]:** *Feecover không nằm trong tập input của `consume.ak`* —
tức `consume.ak` không tham chiếu, không đọc, không tiêu bất kỳ UTxO/script nào thuộc Feecover
khi thực thi. Đây là một **ràng buộc kiến trúc / phạm vi** (architectural scope boundary), khác
loại với các bất biến `FEECOVER-*` khác trong bảng dưới — nó được kiểm bằng cách đọc mã nguồn
`consume.ak` (không import/không gọi module Feecover), KHÔNG phải một điều kiện được validator
ép runtime bằng một guard cụ thể. Kiểm auditor: `grep` `consume.ak` không có tham chiếu tới
`feecover.ak`/`FeecoverAccrual`/`FeecoverGate`. Nếu điều này đổi trong tương lai (Feecover được
nối vào `consume.ak`), Định lý T5 (§6) phải được xét lại từ đầu — nó phụ thuộc trực tiếp vào
P-LAYER-1 đúng như đang phát biểu.

| ID canonical | Bất biến | Neo |
|---|---|---|
| FEECOVER-GATE-1 | Account owner gắn PhoenixKey DID hợp lệ point-in-time: `did_active(did,e)=active` ∧ (nếu ký khoá) `key_authorized(key,did,e)`; đọc qua reference input | Tech §3.2; VeData §2.1 |
| FEECOVER-GATE-2 | Freshness cổng: `current_epoch − FeecoverGate.epoch_valid_until ≤ 0` (mẫu C-CM-5) | Tech §3.2 |
| FEECOVER-GATE-3 | DID credential đọc qua **reference input** (CIP-31), KHÔNG tiêu/KHÔNG mint trong tx Feecover (chống tự đúc DID) | Tech §3.2 |
| FEECOVER-PAY-1 | `carp_paid == fee_magic[s]` (neo 1:1 thread↔nanogic); thiếu/thừa → fail `[F-5]` | Tech §4.1 |
| FEECOVER-PAY-1b | `magic_consumed == fee_magic[s]` = `required(σ(s),1)` `[F-4]` (khớp op, đồng bộ 2 bảng) | Tech §4.1; §7.3 |
| FEECOVER-PAY-2 | CARP chỉ accrue đúng `(tier,epoch)`; `carp_accrued` chỉ tăng trong epoch, reset sau settle | Tech §4.1 |
| FEECOVER-SETTLE-1 | `epoch_settle` tuần tự: `e_c == L_settled + 1` (không bỏ/nhảy/double-settle) `[F-8]` | Tech §4.3 |
| FEECOVER-SETTLE-2 | Chỉ **CARP** về Phoenix Treasury; KHÔNG LAMP/MAGIC | Tech §4.3 |
| FEECOVER-SETTLE-3 | `carp_to_treasury / deposited_l3 / topup_requested` append-only (chỉ tăng) `[F-10]` | Tech §4.3 |
| FEECOVER-HO-1 | `deposit` chỉ chuyển CARP ĐÃ ở Phoenix Treasury; release qua Governance `status==Executed` | Tech §4.4 |
| FEECOVER-HO-2 | `stabilizer_ref` chỉ đổi qua Governance | Tech §4.4 |
| FEECOVER-HO-3 | Feecover KHÔNG mint/burn CARP/LAMP; KHÔNG đọc oracle giá (giữ INV-PEG-BY-DEMAND) | Tech §4.4 |
| FEECOVER-ACC-2 | `quantity_of(accrual.value, carp_policy, carp_name) == carp_accrued` (datum khớp value CARP thật, chống datum-nói dối) | Tech §4.1 (chờ B2 policy-id) |

> **Ràng buộc thứ tự:** riêng FEECOVER-ACC-2 + FEECOVER-PAY-1 chỉ verify được asset-identity SAU khi B2 (CARP policy-id) land — xem §9. Điểm cần validator vá chặt nhất là khâu settle (FEECOVER-SETTLE-1..3) — xem §9 + Tech §7.

### 4.3 CARP-lõi KHÔNG chạm (giữ nguyên, chỉ liệt để auditor kiểm biên)

| ID | Bất biến | Neo |
|---|---|---|
| INV-PEG-BY-DEMAND | Peg-core giữ bởi cầu dịch vụ THỰC (utility-floored), oracle-free | Whitepaper §8.1/§8.3 |
| INV-NO-LAMP-PEG-DEFENSE | Tuyệt đối không đỡ-peg bằng LAMP | Whitepaper §10 F9 |
| `1 CARP = 1 MAGIC` | Sức mua nội sinh, bất biến hiến định | Whitepaper §5 |

---

## 5. Mệnh đề ép từng thao tác (đối chiếu code plan)

> Đây là mệnh đề spec cho `feecover.ak`. Guard load-bearing neo lại dòng code khi triển khai.

| Thao tác | Mệnh đề ép (fail nếu vi phạm) | Bất biến |
|---|---|---|
| Consume+Feecover (per-tx) | `magic_consumed == fee_magic[s]` (đọc Σ burns THẬT, không tin redeemer) | FEECOVER-PAY-1b |
| Consume+Feecover | `carp_paid == fee_magic[s]`; CARP-fee từ input NGOÀI Vault-account | FEECOVER-PAY-1 |
| Accrue | `A'.carp_accrued == A.carp_accrued + carp_paid` ∧ `A'.epoch == current_epoch` | FEECOVER-PAY-2 |
| Accrue | `quantity_of(A'.value, carp) == A'.carp_accrued` | FEECOVER-ACC-2 |
| Gate | DID credential present (ref input, `policy==did_registry_policy`), không mint/tiêu | FEECOVER-GATE-1/3 |
| Schedule read | `fee_magic == price(entry.op_type)` từ `PriceParam` (chống lệch 2 bảng) | FEECOVER-PAY-1b + §7.3 |
| Settle | `closed_epoch == last_settled+1` ∧ chỉ CARP → Treasury hiến định | FEECOVER-SETTLE-1/2 |

**Điểm nối load-bearing:** FEECOVER-PAY-1b + schedule-read cùng đọc `PriceParam` qua `price_ref` mà `consume.ak` dùng. Vì `consume.ak` ĐÃ ép `total_burned == total_required` (C-CM-2, dấu `==` nghiêm ở bản canonical), chuỗi ràng đóng kín: `Σburns == required == fee_magic == carp_paid`. Không có kẽ khai service rẻ mà tiêu nhiều (gaming eval FG-2 🟢 CHẶN trên bản canonical).

---

## 6. Định lý conservation (chứng minh phác thảo)

Ký hiệu tập tx epoch `e`: `𝕋x(e) = { x : x hợp-lệ, đóng trong e }`. Mỗi `x` mang `carp_paid(x)`, `fee_magic(x) = fee_magic[s(x)]`, `magic_consumed(x)`.

### Định lý T1 (Peg per-tx — phí không tự sinh/mất trong 1 tx)

**Mệnh đề.** Với mọi tx Feecover hợp lệ `x`: `carp_paid(x) = magic_consumed(x) = fee_magic[s(x)]`.

**Chứng minh.**
1. `x` hợp lệ ⇒ thoả FEECOVER-PAY-1b `[F-4]`: `magic_consumed(x) == fee_magic[s(x)]`.
2. `x` hợp lệ ⇒ thoả FEECOVER-PAY-1 `[F-5]`: `carp_paid(x) == fee_magic[s(x)]`.
3. Từ (1),(2) bắc cầu: `carp_paid(x) = magic_consumed(x) = fee_magic[s(x)]`. ∎

**Hệ quả.** Không tồn tại tx hợp lệ nào trả CARP lệch lượng MAGIC tiêu. Một vế lệch ⇒ tx fail (an toàn đóng).

### Định lý T2 (Accrual conservation — gom đúng Σ trong epoch)

**Mệnh đề.** Cuối epoch `e` (trước settle): `Σ_Platform A_Plat(e) = Σ_{x ∈ 𝕋x(e)} carp_paid(x)`.

**Chứng minh.**
1. Khởi tạo epoch: `A_App(e) = A_Plat(e) = 0` (reset sau settle epoch trước, `[F-10]`).
2. Mỗi `x ∈ 𝕋x(e)` áp đúng một lần `[F-6]` (FEECOVER-PAY-2 buộc accrue đúng `(tier,epoch)`, đúng một lần — thread NFT accrual chống replay, cùng khuôn C-CM-4).
3. App→Platform là cộng dồn không mất mát: `Σ_Platform A_Plat(e) = Σ_App A_App(e) = Σ_x carp_paid(x)`.
4. `carp_accrued` chỉ tăng trong epoch ⇒ không trừ ẩn ⇒ tổng bảo toàn. ∎

**Hệ quả (breakdown).** `Σ_s B_s(e) = Σ_x carp_paid(x)` (mỗi tx cộng đúng 1 lần vào `B_{s(x)}` `[F-7]`).

### Định lý T3 (Treasury nhận đúng Σ — settle bảo toàn, không double)

**Mệnh đề.** Với mọi epoch đã settle `e_c`: `T(e_c) − T(e_c−1) = Σ_{x ∈ 𝕋x(e_c)} carp_paid(x)`, Treasury nhận đúng CARP đó (KHÔNG LAMP/MAGIC), không settle hai lần.

**Chứng minh.**
1. `[F-8]` (SETTLE-1): `e_c == L_settled + 1`; `L_settled` monotonic +1 ⇒ mỗi epoch settle đúng một lần.
2. `total_carp(e_c) = Σ_Platform A_Plat(e_c)` `[F-9]` `= Σ_x carp_paid(x)` (T2).
3. `[F-10]`: `T(e_c) = T(e_c−1) + total_carp(e_c)` ⇒ hiệu = `Σ_x carp_paid(x)`.
4. SETTLE-2 ⇒ tài sản chuyển là CARP thuần (LAMP/MAGIC bị chặn ở vị từ settle).
5. `[F-10]` append-only (SETTLE-3) ⇒ `T` không giảm ⇒ không rút ngược qua đường settle. ∎

### Định lý T4 (Global conservation — end-to-end, không rò)

**Mệnh đề.** Qua chuỗi epoch `1..E` đã settle liên tục: `[CONS]` (§3).

**Chứng minh.**
1. Telescoping T3 qua `e = 1..E` (settle tuần tự `[F-8]`, `T(0)=0`): `T(E) = Σ_e Σ_x carp_paid(x)`.
2. Áp T1 cho mọi `x`: `carp_paid(x) = magic_consumed(x)` ⇒ vế phải thứ hai. ∎

**Diễn giải auditor.** Tổng CARP về Treasury = **đúng** tổng MAGIC đã tiêu trên toàn dòng đời — phí không tự sinh (nếu sinh, T1 vỡ) và không tự mất (nếu mất, T2/T3 vỡ).

### Định lý T5 (Vault non-drain — Feecover không rút value khỏi Vault)

**Mệnh đề.** Nhánh CARP của Feecover KHÔNG làm giảm value non-MAGIC trong Vault account.

**Chứng minh.**
1. CARP-fee đến từ **input riêng của user** (ví ngoài Vault-account-script), chảy vào `FeecoverAccrual` — KHÔNG rút từ Vault UTxO. *(Rigor note, thêm 2026-07-12: điều kiện "CARP-fee từ input NGOÀI Vault-account" xuất hiện ở mệnh đề ép §5 dòng FEECOVER-PAY-1, nhưng KHÔNG có mặt trong phát biểu hình thức của FEECOVER-PAY-1 ở bảng §4.2 — chỉ ghi `carp_paid == fee_magic[s]` [F-5]. Đây là một khoảng trống rigor thật: tiền đề bước 1 dựa vào một ràng buộc mà bảng bất biến chính thức chưa ép rõ. Cần đồng bộ: hoặc thêm điều kiện nguồn-input vào phát biểu FEECOVER-PAY-1 ở §4.2, hoặc tách thành một bất biến `FEECOVER-PAY-1c` riêng, trước khi coi T5 là đã chứng minh đầy đủ.)*
2. C-CM-1 ép value non-MAGIC @Vault bảo toàn byte-perfect; CARP nếu nằm trong Vault thuộc lớp non-MAGIC ⇒ được C-CM-1 bảo vệ.
3. Feecover KHÔNG là input của `consume.ak` (P-LAYER-1, định nghĩa §4.2) ⇒ không nới C-CM-1.
∴ mọi CARP Feecover di chuyển là CARP-input-user, không phải Vault-value. ∎

### Bổ đề T6 (Firewall — Feecover không chạm peg CARP-lõi)

FEECOVER-HO-3: Feecover KHÔNG mint/burn CARP/LAMP, KHÔNG đọc oracle ⇒ không có input nào từ Feecover vào INV-PEG-BY-DEMAND / INV-NO-LAMP-PEG-DEFENSE / `1 CARP=1 MAGIC`. **Kiểm auditor:** grep validator Feecover không import module CARP peg/oracle; `deposit`/`request_topup` chỉ ghi sổ + trỏ `stabilizer_ref` (L3), không tự kéo tài sản.

---

## 7. Không gian trạng thái + tham số

### 7.1 Đồ thị chuyển trạng thái (FeecoverAccrual / FeecoverEpochSettle)

```
FeecoverAccrual(provider, tier, epoch e):
   [init 0] ──Accrue(x)──▶ [carp_accrued += carp_paid(x)]  (lặp, chỉ tăng, trong e)
                                    │
                          epoch e đóng, Settle(e)
                                    ▼
   [reset 0 cho epoch e+1]  ◀────  chuyển Σ CARP → Phoenix Treasury

FeecoverEpochSettle:
   [L_settled = k] ──Settle(k+1)──▶ [L_settled = k+1, carp_to_treasury += total_carp]
   (chỉ nhận e_c == L_settled+1 — mọi e_c khác REJECT: chống nhảy/bỏ/double)
```

### 7.2 Tham số

| Tham số | Ý-nghĩa | Nguồn / [CẦN CHỐT] |
|---|---|---|
| `Q = 10^9` | Scale Q-format | cố định, `MATH.md §1` |
| `fee_magic[s]` | Phí mỗi dịch vụ (nanogic) | **[CẦN CHỐT D1]** — bảng Feat §4.3 là minh hoạ |
| `max_stale` cổng | Freshness `ServiceFeeSchedule`/`FeecoverGate` | mẫu C-CM-5; **[CẦN CHỐT D2]** cadence |
| `σ(s)` | Ánh xạ service_id→op_type | **[CẦN CHỐT D6]** cơ chế đồng bộ 2 bảng |
| tập `s` phủ | did.anchor/did.rotate/vc.verify/custody.op | **[CẦN CHỐT D3]** |
| ranh giới App/Platform | tầng gom accrual | **[CẦN CHỐT D4]** |
| `stabilizer_ref` | địa chỉ L3 + chữ ký interface | **[CẦN CHỐT D5]** |
| Phoenix Treasury addr | địa chỉ hiến định nhận CARP | hiến định, chờ neo on-chain |

### 7.3 Đồng bộ 2 bảng giá (`PriceParam.base_price` ↔ `ServiceFeeSchedule.fee_magic`)

Mỗi `ServiceFeeEntry` mang `op_type` trỏ dòng `OpPrice` trong `PriceParam`; validator đọc `required = price(op_type)` (như `consume.ak`) và ép `fee_magic == required`. Cadence gov: một tx governance đổi cả hai HAY ràng khớp on-chain — **[CẦN CHỐT D6]**.

---

## 8. Giả định tin cậy (ngoài phạm vi chứng minh)

- **C-CM-1..5 đúng** (đã audit ở ConsumeMAGIC bản canonical `MAGIC/ConsumeMAGIC`). Nếu dùng nhầm bản CŨ `MAGIC-paymaster/ConsumeMAGIC` (mint-burn, `≥`, không did_commit) → T1/T5 cần soát lại (gaming eval FG-1).
- **CARP peg giữ** bởi lớp L3 (INV-PEG-BY-DEMAND). Feecover KHÔNG bảo vệ giá trị nếu CARP peg vỡ (chủ đích — T6 firewall).
- **Resolve API point-in-time** (`did_active`/`key_authorized`) trung thực và fail-closed. GATE-1 dựa API này (đội backend implement, chưa live).
- **Provider trung thực ở khâu settle** — validator PHẢI tự ép (luật thiết kế FG-4, §9), không được để lộ chỗ trung thực tự khai.

---

## 9. [CẦN CHỐT] còn treo (blocker / dependency)

- **[CẦN CHỐT B1] MAGIC-model reconcile.** Mọi định lý giả định mô hình 3-token (MAGIC = số dư-account trong Vault, non-transferable; CARP = token thanh toán). Nếu MAGIC-model đảo lại native, `[F-5]` neo 1:1 và T1 cần soát lại vế `magic_consumed`. Chốt canonical: Whitepaper Tokenomic §4–5.
- **[CẦN CHỐT B2] CARP policy-id.** `carp_paid`/`A_*(e)`/`T(e)` đo bằng "thread CARP" nhưng policy-id CARP chưa chốt on-chain. Trước đó FEECOVER-PAY-1/SETTLE-2/ACC-2 chỉ verify được ở mức lượng, chưa verify được asset-identity. Blocker on-chain.
- **[CẦN CHỐT B3] did_commit attribution.** `EngageDatum` **ĐÃ CÓ** field `did_commit : ByteArray` (append-only, `types.ak:46-51`), validator **ĐÃ ép immutable** (`consume.ak:409`). MVP nội dung = `#""`. Blocker thật **KHÔNG phải thêm field** mà là **enforce NỘI DUNG per-DID**: MAGIC-team + đội backend ràng `did_commit = blake2b256(DID‖salt)` + verify signer khớp DID qua resolve point-in-time. Trước đó FEECOVER-GATE-1 + breakdown per-DID (Hệ quả T2) chỉ đạt attribution-thô theo-owner → một owner mở nhiều account né trần (gaming eval FG-3).
- **[LUẬT THIẾT KẾ FG-4 — hở nội tại, KHÔNG blocker đội khác che] EpochSettle PHẢI có validator ép, không dựa provider tự khai.** T2/T3/T4 giả định provider chạy settle trung thực; validator `epoch_settle` PHẢI ép: (a) forced-settle (chặn provider ngâm CARP vô hạn); (b) `total_carp == Σ accrual` on-chain (chống skim khai khống); (c) đích Treasury/`stabilizer_ref` bất biến trừ DAO. **KHÔNG mượn khuôn `fee_treasury.ak:108-118` Withdraw-operator** (rút tự do trên floor). Đội Feecover PHẢI tự vá validator settle — KHÔNG tái dùng khuôn nào khác. Nguồn: `_feecover-gaming-evaluation.md` FG-4.
- **[CẦN CHỐT — ranh giới 2 lớp phí] Feecover vs `PhoenixKey-Math.md §36`:** §36 là phí giao thức per-operation đo bằng Lovelace (split 30/70), namespace `I-FEE-n` thuộc §36 và KHÔNG dùng trong module này. Feecover là fee-abstraction MAGIC→CARP, namespace `FEECOVER-*`. Ranh giới Feecover-CARP-settle vs §36-Lovelace-fee (+ `fee_treasury.ak` ADA-FX, FG-4b/FG-8) chờ maintainer chốt.

---

## 10. Checklist cho auditor

- [ ] Xác nhận validator Feecover đọc `PriceParam` qua CÙNG `price_ref` mà `consume.ak` dùng (điểm nối §5) — không bảng giá riêng.
- [ ] Xác nhận `carp_paid == fee_magic` bằng `==` (không `≥`/`≤`) — FEECOVER-PAY-1.
- [ ] Xác nhận `quantity_of(accrual.value, carp) == carp_accrued` mỗi accrue — FEECOVER-ACC-2 (chờ B2).
- [ ] Xác nhận settle ép `closed_epoch == last_settled+1` monotonic + chỉ CARP về đúng địa chỉ Treasury hiến định — FEECOVER-SETTLE-1/2.
- [ ] Xác nhận settle KHÔNG dùng Withdraw-operator-cosign (FG-4 vá).
- [ ] Xác nhận `deposit`/`request_topup` chỉ ghi sổ + gov-release, không mint/burn/oracle — FEECOVER-HO-1/2/3 (T6).
- [ ] Xác nhận dùng bản canonical `MAGIC/ConsumeMAGIC` (no-mint, `==`, có did_commit), KHÔNG bản `MAGIC-paymaster/ConsumeMAGIC` (FG-1).
- [ ] Xác nhận DID credential đọc qua reference input, không mint/tiêu trong tx — FEECOVER-GATE-3.

---

## Phụ lục: đối chiếu invariant ↔ neo dòng code

| Invariant | Neo dòng code |
|---|---|
| C-CM-1..5 | `MAGIC/ConsumeMAGIC/onchain/validators/consume.ak` |
| FEECOVER-* (GATE/PAY/SETTLE/HO/ACC) | `feecover/onchain/validators/feecover.ak` |
| `EngageDatum.did_commit` field (immutable-enforced, MVP `#""`) | `types.ak:46-51`, `consume.ak:409` |

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai) — bản gốc Math (đã qua rà soát nội bộ): ký hiệu, F-1..F-10, T1..T6; mô hình 3-token, bất biến, blocker; gaming eval (FG-1..FG-8, đặc biệt FG-4 settle-hở).
- Engine tái dùng: `MAGIC/ConsumeMAGIC/onchain/validators/consume.ak` + `lib/magiclamp/consume/{types,pricing,util}.ak` (C-CM-1..5); `types.ak:46-51`, `consume.ak:409` (`did_commit`).
- Đơn vị/peg canonical: `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` §4/§5/§8.
- Đụng mã: `PhoenixKey-Specs/PhoenixKey-Math.md §36` (kiến trúc phí Lovelace 30/70, KHÁC module).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
