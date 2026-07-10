# PhoenixKey — Feecover (EXECUTIVE, đặc-tả điều-hành)

> **Module:** Feecover (fee-abstraction).
> **Loại doc:** Executive — quyết-định, rủi-ro, lộ-trình. Cho lãnh-đạo.
> **Ngày:** 2026-07-09.
> **Đối tượng đọc:** lãnh-đạo, người ra quyết-định chiến-lược.

Tài-liệu cùng bộ: [PhoenixKey-Feecover-Vi-Feat.md](./PhoenixKey-Feecover-Vi-Feat.md), [PhoenixKey-Feecover-Math.md](./PhoenixKey-Feecover-Math.md), [PhoenixKey-Feecover-Tech.md](./PhoenixKey-Feecover-Tech.md).

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## 1. Tóm-tắt một trang

**Feecover là gì.** Lớp **trừu-tượng-hoá phí (fee-abstraction)** của riêng PhoenixKey: user **định-giá dịch-vụ bằng MAGIC** (đơn-vị-kế-toán, ổn-định-sức-mua), nhưng **thanh-toán bằng CARP** (đồng lưu-thông); hệ thống lo phần ADA/DUST/phí hạ-tầng khác. User chỉ thấy MỘT loại đơn-vị, không phải hiểu tỷ-giá.

**Nguyên-tắc cốt-lõi.** Feecover **ĐẶT LÊN TRÊN** cơ-chế `ConsumeMAGIC` đã có — KHÔNG phải engine phí mới. Phần lõi đo-tiêu (engine đo-tiêu, beacon giá, bảo-toàn value, chống replay/double-satisfaction) tái-dùng nguyên trạng (bảng thành-phần Tech §2: 6/17 thành-phần tái-dùng). Feecover chỉ thêm 5 phần chính-sách mỏng: bảng phí theo dịch-vụ, cổng chỉ-DID, quy-đổi-CARP, gom-phí-theo-tầng, policy ổn-định.

**Đổi tên.** Payyou → Feecover (maintainer chốt 2026-07-03). "Paymaster" giữ cho tác-vụ khác, KHÔNG brand Feecover là paymaster.

**Cách tiền chảy.** User consume → giảm số dư MAGIC account trong Vault + trả CARP `= fee_magic` (neo `1 CARP = 1 MAGIC`) → gom vào Vault provider theo tầng **App → Platform** → **cuối mỗi epoch** đối-soát chuyển **CARP về Phoenix Treasury**. KHÔNG LAMP, KHÔNG MAGIC rời account. Đã BỎ `collectToTreasury`.

**Ổn-định phí thế nào.** Phí neo **danh-nghĩa theo nanogic** (`1 nanogic = 1 byte·ngày` lưu-lạnh → `1 MAGIC = 1 GB·ngày`) — **zero-oracle**, KHÔNG bám giá thị-trường. Biến-động thị-trường là việc của giao-thức ổn-định-giá CARP (L3, ngoài phạm-vi). Khi giá hạ-tầng nền thực trôi → re-anchor qua **quản-trị** (con người, theo cadence), không phải hàm giá tự-động.

**Điều-kiện bật production.** Chặn bởi **3 blocker của đội khác** (xem §4) — không bật production trước khi cả 3 land.

---

## 2. Bảng quyết-định (quyết | lý-do 4-trục | đánh-đổi)

4 trục: (a) định-hướng dài-hạn · (b) first-principles · (c) tối-ưu eUTXO/ExUnit/phí/đơn-giản · (d) lợi-ích user + bền-vững.

| Quyết-định | Lý-do (4 trục) | Đánh-đổi |
|---|---|---|
| **(a) Tái-dùng `ConsumeMAGIC`, KHÔNG dựng engine mới** | (b) engine consume + kế-toán state + bảo-toàn value đã đúng & đã audit (C-CM-1..5); (c) engine thứ hai = nợ kỹ-thuật + bề-mặt tấn-công đôi; (a) một lõi để bảo-trì dài-hạn | Feecover phụ-thuộc tiến-độ + hình-dạng dữ-liệu của ConsumeMAGIC (`did_commit`); không tự-chủ hoàn-toàn |
| **(b) Phí ổn-định-sức-mua danh-nghĩa (neo nanogic), zero-oracle** | (b) MAGIC neo sức-mua-dịch-vụ-nền cố-định danh-nghĩa, không giao-dịch nên không trôi giá; (c) bỏ oracle = ít bề-mặt tấn-công + rẻ ExUnit; (d) user luôn thấy phí ổn-định, không chịu sốc giá thị-trường | base_price nền có thể trôi mà `fee_magic` giữ nguyên → cần re-anchor thủ-công; không phản-ứng tức-thời với biến-động chi-phí thật |
| **(c) Gom CARP về Treasury theo tầng App→Platform, cuối-epoch** | (b) kế-toán một-nguồn, đối-soát tuần-tự (`last_settled_epoch` monotonic); (c) accrue per-tx nhẹ, settle gộp cuối epoch tiết-kiệm; (a) Treasury một cửa CARP dễ kiểm-toán | Độ trễ epoch giữa thu phí và về Treasury; cần cấu-trúc `FeecoverAccrual` 2 tầng |
| **(d) Chỉ CARP về Treasury (KHÔNG LAMP/MAGIC)** | (b) MAGIC là số dư account không rời thành token, LAMP là backing tầng khác; (d) ranh-giới sạch theo mô hình 3-token | Mọi settlement buộc quy về CARP; phụ-thuộc CARP peg giữ ổn-định |
| **(e) Ổn-định-giá CARP = NGOÀI PHẠM VI (chỉ định 2 mũi tên `deposit`/`request_topup`)** | (b) tách trách-nhiệm: peg-core giữ bởi cầu-thực (INV-PEG-BY-DEMAND), Feecover không chạm; (a) không nợ kỹ-thuật oracle/buyback; (c) đơn-giản | Feecover không tự bảo-vệ giá-trị nếu CARP peg lệch — hoàn-toàn trông vào L3 |

---

## 3. Ma-trận rủi-ro

| Rủi-ro | Mức | Cơ-chế / giảm-thiểu | Còn hở |
|---|---|---|---|
| **base_price nền trôi mà `fee_magic` giữ nguyên** → phí lệch sức-mua thực | TRUNG BÌNH | Re-anchor quản-trị đồng-nhịp base_price. KHÔNG dùng oracle (giữ INV-PEG-BY-DEMAND) | **Cadence quá chậm** → provider lỗ hoặc user trả đắt. Cần D2 chốt biên + nhịp |
| **Phụ-thuộc CARP peg** (settlement bằng CARP) | TRUNG BÌNH | Ngoài phạm-vi — tầng CARP (GreenPeg/RedPeg) lo. Feecover chỉ bàn-giao `deposit`/`request_topup` | Nếu CARP peg vỡ, giá-trị thu về Treasury lệch. Feecover không có phòng-thủ riêng (chủ đích) |
| **Fee spam** (tiêu vô-nghĩa bơm accrual) | THẤP | Mỗi op giảm MAGIC thật + trả CARP thật = tự-tiêu tài-nguyên; gate DID chặn account ảo | Khi `did_commit` chưa gắn nội-dung, gate chỉ ràng `owner` → một owner mở nhiều account. Chờ B3 |
| **Lệch 2 bảng giá** (`PriceParam` ↔ `ServiceFeeSchedule`) | TRUNG BÌNH | Cả hai versioned + gov-gated; ràng `magic_consumed == fee_magic` **và** `carp_paid == fee_magic` | Cơ-chế đồng-bộ chưa chốt (D6): 1-tx-governance-đổi-cả-hai HAY ràng khớp on-chain. Đòn NHANH hơn là **keeper `demand_mult`** dưới-DAO không band (gaming FG-5) |
| **Consume-side drain** (rút Vault qua nhánh tiêu) | THẤP | C-CM-1 bảo-toàn value non-MAGIC byte-perfect; CARP-fee là input riêng user, không rút từ Vault | Ổn ở tầng consume (ConsumeMAGIC đã audit) |
| 🔴 **EpochSettle provider-trust (FG-4)** | **CAO** | Validator settle PHẢI ép forced-settle + chống skim `total_carp`, KHÔNG mượn khuôn `fee_treasury.ak:108-118` (Withdraw operator-cosign, rút tự-do) | Provider có thể giữ CARP giữa-chừng nếu luật này không được ép đủ chặt. **Lỗ kế-toán nặng nhất — Feecover-team PHẢI tự vá validator settle, không blocker đội khác che.** Neo `_feecover-gaming-evaluation.md` FG-4 |
| 🔴 **Trùng `fee_treasury.ak` deployed (FG-4b/FG-8)** | CAO | Ranh-giới CARP-settle (Feecover) vs ADA-FX (fee_treasury) chờ maintainer chốt | Đã tồn-tại `MAGIC/FeeTreasury/onchain/validators/fee_treasury.ak` (model FX-fund ADA, operator-cosign, NORMATIVE trỏ tên cũ "PhoenixKey-Fee-Abstraction"). Hai "fee treasury" cùng hệ |
| **Giả-mạo DID lọt cổng** | THẤP–TRUNG BÌNH | DID đọc qua reference input (không mint/tiêu); point-in-time `did_active`/`key_authorized` chống khoá đã rotate | Phụ-thuộc an-toàn resolve API; PersonDID anchor-giả (gaming FG-6, chờ PA2 UniquenessThread) |

---

## 4. Điều-kiện-tiên-quyết / blocker chặn production

> **KHÔNG bật production tới khi cả 3 blocker land.**

| # | Blocker | Chủ | Ảnh-hưởng nếu thiếu |
|---|---|---|---|
| **B1** | **MAGIC-model reconcile** — chốt dứt-điểm mô hình account-in-vault vs native (mô hình đúng = số dư account, non-transferable, KHÔNG policy-id) | MAGIC-team | Nền-tảng của toàn bộ đo-tiêu; chưa reconcile thì mọi ràng-buộc consume treo |
| **B2** | **CARP policy-id** — token CARP settlement cần policy-id canonical để ràng `carp_paid` | CARP-team | Không có policy-id thì không ép được nhánh thanh-toán CARP `= fee_magic` |
| **B3** | **`did_commit` attribution** — `EngageDatum.did_commit` hiện = `#""` (sentinel rỗng, field ĐÃ CÓ + immutable); cần enforce NỘI DUNG blake2b256 DID thật per-DID | MAGIC-team (+ PhoenixKey backend) | Chỉ gate được ở mức `owner`, chưa per-DID → hở fee-spam/né-trần; attribution breakdown thô |

**Hệ quả điều-hành:** Feecover có thể build lớp policy + gate ở mức `owner`-gating tạm, nhưng **attribution per-DID, chống-farm chặt, và settlement CARP thực đều đòi B1–B3 land trước**. Không tuyên-bố production trước mốc đó.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## 6. Lộ-trình mốc

| Giai-đoạn | Nội-dung | Điều-kiện vào |
|---|---|---|
| **P0 — Chốt chiến-lược** | Lãnh-đạo quyết 6 câu hỏi §7 (mức phí, cadence, tầng, stabilizer, đồng-bộ bảng giá) | Spec đã merged (xong) |
| **P1 — Reconcile blocker** | B1 (MAGIC-model), B2 (CARP policy-id), B3 (`did_commit`) land ở đội tương-ứng | Cần điều-phối liên-đội |
| **P2 — Build lớp Feecover** | Viết `ServiceFeeSchedule` + `FeecoverGate` + accrual/settle; ràng `carp_paid == fee_magic`; **tự vá settle chống FG-4**; KHÔNG chạm `consume.ak` | B1/B2 land (B3 có thể sau, chạy tạm `owner`-gating) |
| **P3 — Test hành-vi** | Test on-chain: neo 1:1, gom theo tầng, settle tuần-tự + forced-settle, gate DID point-in-time; evidence output thật | P2 xong |
| **P4 — Nâng per-DID** | Gắn `did_commit` thật → attribution per-DID, đóng hở fee-spam | B3 land |
| **P5 — Production** | Chỉ bật khi B1+B2+B3 land + P3 xanh + 6 quyết-định chốt | Tất cả trên |

---

## 7. Câu hỏi cần lãnh-đạo chốt (6, chiến-lược)

| # | Quyết-định | Gợi-ý |
|---|---|---|
| **D1** | Mức phí từng dịch-vụ (`fee_magic` mỗi `service_id`) | Bảng Feat §4.3 là minh-hoạ; cần số thật, neo qua base_price nền |
| **D2** | Cadence + biên re-anchor phí | Mượn khung base_price CARP (±10%/quý + siêu-đa-số) HAY khung riêng PhoenixKey — cân "ổn-định user" vs "provider không lỗ" |
| **D3** | Dịch-vụ nào Feecover phủ | Khởi-đầu: `did.anchor`, `did.rotate`, `vc.verify`, `custody.op` |
| **D4** | Ranh-giới tầng App vs Platform | Ai là App (provider), ai là Platform (gom nhiều App)? Ảnh-hưởng cấu-trúc accrual |
| **D5** | `stabilizer_ref` là ai + chữ-ký interface L3 | RedPeg? tổng-hợp GreenPeg/RedPeg? Feecover chỉ định interface, L3 implement |
| **D6** | Cơ-chế đồng-bộ `PriceParam` ↔ `ServiceFeeSchedule` | 1-tx-governance-đổi-cả-hai HAY ràng khớp on-chain (chống lệch 2 bảng); cân DAO-gate `demand_mult` (gaming FG-5) |

**+ 1 câu hỏi ranh-giới (mới nổi):** Feecover-CARP-settle vs `fee_treasury.ak` (ADA-FX đã deploy) là hai thứ tách rời hay `fee_treasury.ak` chính là "L3 stabilizer"? (FG-4b/FG-8 — chờ chốt trước khi build settle.)

---

## 8. Ranh-giới cứng (không over-claim)

- **Điểm hở nội-tại nặng nhất (FG-4):** khâu settle PHẢI có validator ép, KHÔNG dựa provider-trust thuần. KHÔNG blocker đội khác che — đội Feecover gánh, phải vá ở P2.
- **KHÔNG chạm** `consume.ak` (C-CM-1..5) và bất-biến CARP-lõi (INV-PEG-BY-DEMAND, INV-NO-LAMP-PEG-DEFENSE, `1 CARP = 1 MAGIC`). Feecover là lớp mỏng, ranh-giới cứng.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai) — bản gốc Exec (đã qua rà-soát nội-bộ), §9 (D1–D6), gaming eval (FG-4/FG-4b/FG-5/FG-6/FG-8).
- Đơn-vị/peg: `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` §4/§5.
- Tài-liệu cùng bộ: [PhoenixKey-Feecover-Vi-Feat.md](./PhoenixKey-Feecover-Vi-Feat.md), [PhoenixKey-Feecover-Math.md](./PhoenixKey-Feecover-Math.md), [PhoenixKey-Feecover-Tech.md](./PhoenixKey-Feecover-Tech.md).
