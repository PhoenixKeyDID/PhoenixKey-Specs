# PhoenixKey — Wakeme · ĐẶC TẢ ĐIỀU HÀNH (cho lãnh đạo)

> **Module:** Wakeme (GetLAMP — kích hoạt nhận LAMP). **Loại doc:** Điều hành. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** lãnh đạo — quyết định + lý do + đánh đổi + rủi ro + lộ trình. Chi tiết toán/invariant: [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md). Kỹ thuật: [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md). Người dùng: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md). Đánh giá gaming nội bộ (không công khai).
>
> Tài liệu này KHÔNG lặp toán. Chỉ nêu điều lãnh đạo cần để duyệt và chốt.

---

## 1. Tóm tắt một trang

**Wakeme là gì.** Luồng khởi tạo người dùng của PhoenixKey. User tạo khoá bằng vân tay (Secure Enclave), bấm **một nút GetLAMP** → nhận `D` LAMP vào **vault của chính mình** (khoá có điều kiện). **Khởi tạo MIỄN PHÍ** — Feecover lo phí gas, user không cần nạp tiền, không cần hiểu tỷ giá.

**Cơ chế hai pha.** `D = min(1001, ⌊pot/1_000_000⌋)`.
- **PHA-1 (ngày 1 → 1001):** LAMP khoá "có điều kiện" (chưa sở hữu). Số dư LAMP **sinh MAGIC** (quyền tiêu dịch vụ) — engine chỉ ĐỌC số dư, không đụng/không đốt LAMP. User hưởng = dòng MAGIC hằng ngày. Ngày nào không tiêu thật đủ mức → thu hồi 1 LAMP về pot (anti-idle).
- **PHA-2 (từ ngày 1002):** phần LAMP sống sót **vest thành SỞ HỮU thật của user, 1 LAMP/ngày** — nhưng **có điều kiện duy trì**: mỗi epoch phải tiêu thật đủ mới mở khoá. Bỏ bê 1001 epoch liên tục → thu hồi phần chưa mở khoá về pot (phần đã sở hữu KHÔNG bị đụng). LAMP đã vest → user rút/bán/tự-Gen tuỳ ý.

**Bản chất kinh-tế.** Đây là **hợp đồng trung thành**: cam kết dùng thật 1001 ngày → được thưởng sở hữu LAMP. Người bỏ cuộc trả phần chưa kiếm nuôi người mới (pot tự nuôi, phản chu kỳ). Tách rõ **quyền tiêu dịch vụ (MAGIC — account-trong-Vault, non-transferable, KHÔNG mint token)** khỏi **sở hữu-LAMP** — giá trị đến từ TIÊU dịch vụ thật, không từ nắm giữ thụ động. LAMP tổng cung cố định 36 tỷ, KHÔNG burn (giảm lưu hành = chuyển vào pot/Treasury).

**Ba thứ đã BỎ.** (a) Model VND-Genie cũ (nạp 200k VND) — nay miễn phí; (b) v2 unlock-bán ngay — tạo cung chực bán; (c) v3 mượn không sở hữu — không có phần thưởng cho người kiên trì.

> 🔴 **CỔNG GO/NO-GO (đọc trước khi duyệt bật production):** validator an toàn nhưng **KHÔNG được mở GetLAMP-PersonDID trên production tới khi PA2 UniquenessThread + Registry-chuẩn land.** Lý do: PersonDID **giả mạo được ở tầng neo-anchor** (did-string đúc anchor bất kỳ, HW_Key P-256 không verify on-chain — KHÔNG phải lỗ sinh trắc) + cổng chống-wash chưa tồn tại → kẻ tấn công có thể rút ròng nhiều lần D (chi tiết §7 + verdict gaming). Enterprise/Org/Service DID (có parent-sig) KHÔNG dính lỗ này.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 2. Bảng quyết định (quyết | lý do 4 trục | đánh đổi)

| # | Quyết định | Lý do (a dài hạn · b first-principles · c tối ưu · d user+bền vững) | Đánh đổi (trung thực) |
|---|---|---|---|
| **Q1** | **Model 2-PHA: có điều kiện → vesting-sở hữu** (thay borrow-only v3) | (a) tạo hợp đồng trung thành: kiên trì 1001 ngày → sở hữu thật; (d) người bỏ cuộc nuôi người mới. Bền hơn borrow-only (không điểm thưởng) và vesting-bán ngay (cung tức thời). | Phức tạp hơn borrow-only: thêm PHA-2, forfeit, 2 field datum. Vòng đời user rất dài. |
| **Q2** | **Tách quyền tiêu dịch vụ (MAGIC) khỏi sở hữu-LAMP** | (b) first-principles: LAMP = GATE mở tư cách, biến quyết định = lượng-MAGIC-tiêu (WP §7.2). Gen chỉ ĐỌC số dư → LAMP không cạn vì Gen. MAGIC = account-trong-Vault (KHÔNG native token). | Cần engine Gen mới (đọc số dư) — code MAGIC-repo cũ (fire→Treasury) lỗi thời, phải viết lại. Blocker kiến trúc. |
| **Q3** | **Cung bán hoãn qua rào cam kết** (không cung tức thời) | (a)(b) chỉ user tiêu thật đủ 1001 ngày mới có LAMP-sống sót để bán; vest nhỏ giọt 1/ngày. Cung rải đều ≥1001 ngày, không sốc. | Người dùng thật phải chờ rất lâu mới sở hữu — trải nghiệm "chậm được thưởng". |
| **Q4** | **Người bỏ cuộc nuôi người mới** (pot tự nuôi phản chu kỳ) | (a)(d) anti-idle PHA-1 + forfeit PHA-2 + phí thu bằng-LAMP-theo giá trị (giá xuống → thu nhiều LAMP → pot phình đúng lúc thị trường lạnh). LAMP về pot = kế toán, KHÔNG đốt. | PHA-2 LAMP-vest RỜI HỆ sang user → pot cần nguồn bù đủ. Rủi ro pot-cạn thật hơn v3 (xem R1). |
| **Q5** | **Self-consumption HỢP LỆ + khuyến khích; cổng chống-wash = chuẩn Registry** (bỏ counterparty≠owner) | (b)(d) mỗi lượt tiêu tốn phí → CARP/LAMP về Treasury → hệ CÓ LỢI. Cổng đúng = dịch vụ phải tiêu tài nguyên THẬT (lưu trữ/băng thông/tính toán/lao động), duyệt ở tầng Registry. | Đẩy trách nhiệm chống lạm dụng sang **chuẩn Registry** — blocker mới ở Registry-team. Nếu Registry lỏng, wash-rỗng lọt. |
| **Q6** | **D keyed per-PersonDID** (1 người sinh trắc = 1 suất) | (b) đa địa chỉ/đa-DID (Org/Device) KHÔNG nhân suất; sinh trắc Secure Enclave lo trùng người. | Uniqueness ở tầng **neo-anchor** (did-string ↔ khoá gốc) nằm NGOÀI phạm vi vault — hở tới khi PA2 land (KHÔNG phải lo sinh trắc). |

---

## 3. Ma trận rủi ro (rủi ro | mức | giảm thiểu)

| ID | Rủi ro | Mức | Giảm thiểu |
|---|---|---|---|
| **R1** | **Pot cạn khi nhiều user qua PHA-2 cùng lúc** (LAMP-vest rời hệ, không quay lại pot) | 🟡 TRUNG (rủi ro tham số, không phải lỗi thiết kế) | 3 nguồn nạp: phí theo giá trị (phản chu kỳ) + anti-idle/forfeit + Treasury-topup. **[Cần mô phỏng]** cân đối tốc độ-vest-ra vs tốc độ nạp; ngưỡng treasury-topup. |
| **R2** | **Wash-rỗng lọt nếu chuẩn Registry lỏng** (dịch vụ giả không tiêu tài nguyên thật) | 🟡 TRUNG | Chống-wash chuyển sang **duyệt Registry** — dịch vụ rỗng không đăng ký được. Chưa bật anti-idle/epoch-gate production tới khi Registry-team có chuẩn danh mục. |
| **R3** | **Engine Gen chưa spell-out on-chain** — /CARP mới có nguyên lý "đọc số dư", chưa có validator; code cũ đốt LAMP | 🔴 BLOCKER kiến trúc | Không nối Gen production tới khi MAGIC/CARP-team spell-out: đọc VaultDatum reference → drip MAGIC → KHÔNG-spend UTXO-LAMP, KHÔNG mint token. Validator vault (khoá+vest thuần) build/test được ĐỘC LẬP trước. |
| **R4** | **Settlement mint-CARP-tự do (nghi Terra)** | 🟢 THẤP | I-ACT-9: user trả CARP-đã có, backing qua GreenBack backed-path (LAMP-haircut + tài sản cứng + 3-phanh). Wakeme KHÔNG mint/burn CARP/LAMP tự do. |
| **R5** | **Keeper MVP tin system-authority** — chưa có consume-event Registry thật, backend tự attest active/idle | 🟡 TRUNG (nợ MVP) | Chấp nhận cho MVP; khoá cứng khi Registry consume-event sẵn sàng. Validator đo idle bằng GAP (lazy, an toàn), không counter tăng dần. Keeper KHÔNG đoạt được LAMP (đích cứng pot/owner). |
| **R6** | **Forfeit 1001 epoch ≈ 13.7 năm KỂ TỪ đầu PHA-2 (≈16.4 năm kể từ GetLAMP) — rất dài** | 🟡 TRUNG (theo spec, có chủ đích) | Theo spec: khoan-dung với người tạm nghỉ; chỉ thu hồi phần CHƯA mở khoá, KHÔNG đụng phần đã sở hữu. **[Lãnh đạo có thể muốn rút ngắn]** — xem câu hỏi Q-C. |
| **R7** | **Uniqueness anchor PersonDID** — did-string đúc anchor bất kỳ (lỗ mã hoá, KHÔNG phải sinh trắc) | 🔴 GATE (xem §7) | PA2 UniquenessThread đóng ở tầng structural/cryptographic; chặn GetLAMP-PersonDID production tới khi land. Sinh trắc Secure Enclave đủ chống trùng người; lỗ nằm ở anchor không ràng khoá gốc. |

---

## 4. Luật chặn production (áp dụng khi nối phần phụ thuộc đội khác)

- **B1 — Engine Gen đọc số dư (MAGIC/CARP-team).** Chỉ nối Gen production sau khi MAGIC/CARP-team spell-out validator on-chain đọc số dư (KHÔNG spend/đốt LAMP — mẫu cũ fire→Treasury bị cấm tái dùng). Validator vault khoá+vest thuần build/chạy độc lập, không cần chờ Gen.
- **B2 — Chuẩn Registry dịch vụ tiêu tài nguyên thật (Registry-team).** Cổng chống-wash = dịch vụ Registry, thay counterparty-gate. Anti-idle/epoch-gate CHỈ bật production sau khi chuẩn này tồn tại.
- **B3 — GetLAMP-PersonDID production chặn tới khi PA2 UniquenessThread land** (đóng lỗ anchor-uniqueness, xem §7). Org/Service/Enterprise DID (parent-sig) không dính luật này.
- **B4 (phụ) — GreenBack settlement + shadow-price (CARP-team); vesting-release kho→pot + phí bằng-LAMP (LAMP-team)** phải sẵn sàng trước khi bật GetMAGIC + phản chu kỳ production.

---

## 5. Lộ trình mốc

**Nguyên tắc thứ tự:** phần khoá+vest thuần build/test ĐỘC LẬP trước (không cần Gen) → spell-out engine Gen → nối Gen → chuẩn Registry + bật anti-idle → GreenBack → phản chu kỳ.

| Mốc | Nội dung | Phụ thuộc |
|---|---|---|
| **M1** | Validator pot (kế thừa `dist_treasury.ak`) + validator vault 2-pha + GetLAMP backend + conservation/D-cap test + ClaimVested | — (build ngay) |
| **M2** | Maintainer chốt các điểm [CẦN CHỐT] §6; đội on-chain duyệt PR validator | maintainer + đội on-chain |
| **M3** | MAGIC/CARP-team spell-out engine Gen đọc số dư → nối Gen | **B1** |
| **M4** | Registry-team chuẩn danh mục dịch vụ + cổng duyệt → bật anti-idle/epoch-gate production | **B2** |
| **M5** | PA2 UniquenessThread land → mở GetLAMP-PersonDID production | **B3** |
| **M6** | GreenBack settlement (CARP) + shadow-price PHA-2 | B4 |
| **M7** | fee_refill_lamp phản chu kỳ + mô phỏng cân đối dòng-vest-ra (R1) | LAMP/backend |

→ Trạng thái & tiến độ từng mốc: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 6. Câu hỏi cần LÃNH ĐẠO chốt

| # | Câu hỏi | Đề xuất mặc định (spec) | Vì sao cần maintainer |
|---|---|---|---|
| **Q-A** | **Engine Gen đọc số dư** — xác nhận cả Instant + Schedule /CARP đều CHỈ ĐỌC số dư (không nhánh nào còn đốt LAMP), và giao MAGIC/CARP-team spell-out validator on-chain (đọc reference-input → drip → không-spend, KHÔNG mint token). | Cả 2 đọc số dư; giao MAGIC/CARP-team viết engine. | Blocker kiến trúc B1; quyết định chiến lược ai viết cái gì. |
| **Q-B** | **Nhịp MAGIC: daily thật hay daily-tick + epoch-drip.** Maintainer muốn "MAGIC hàng ngày"; anti-idle/vest tick theo NGÀY nhưng gate-active + forfeit theo EPOCH. | tick-daily + drip-epoch, buffer-daily ở backend (không sửa on-chain). | Ảnh hưởng cảm giác user + độ phức tạp on-chain. |
| **Q-C** | **Forfeit 1001 epoch ≈ 13.7 năm — giữ hay rút ngắn?** Rất dài; có chủ đích khoan-dung nhưng có thể quá lỏng. | Giữ 1001 (đối xứng 1001 ngày PHA-1). | Đánh đổi khoan-dung vs pot-tái tuần hoàn. Chỉ đụng phần chưa mở khoá. |
| **Q-D** | **`MIN_MAGIC_TX` = 10% gen-able + granularity** (daily-gen-able vs cumulative). | 10% daily-gen-able từ `conditional_lamp`. | Tham số TẠM (maintainer đặt 07-06); sub-open, buộc với nhịp Q-B. |
| **Q-E** | **Xác nhận PHA-2-vest-sở hữu khớp WP §7.2** — đọc "cung hoãn qua cam kết 1001 ngày + nhỏ giọt" là HỢP LỆ, khác "cung tồn đọng chực bán". | Hợp lệ (đã qua rào cam kết + vest nhỏ giọt). | Diễn giải whitepaper — cần maintainer xác nhận vì "vest-sở hữu" là directive Wakeme-layer, WP không spell-out. |

---

## 7. Điều kiện tiên quyết chặn production (GO/NO-GO)

- **Whitepaper-gap khai báo:** "GetLAMP / pot / 2-pha / vest-sở hữu sau-1001-ngày / phí thu bằng-LAMP" KHÔNG có trong whitepaper — là **directive Wakeme-layer nội bộ**. WP §7.2/§7.3 + /CARP hậu thuẫn **nguyên lý Gen-đọc số dư** rõ ràng; /CARP-team chịu trách nhiệm spell-out cơ chế on-chain (giao ở B1, §4).
- **Keeper MVP:** tin system-authority tới khi có consume-event Registry thật (nợ MVP có chủ đích) — đóng khi B2 (§4) xong.
- **🔴 ĐIỀU KIỆN TIÊN QUYẾT chặn production (verdict gaming, `_activation-gaming-evaluation-v41.md`):** **KHÔNG mở GetLAMP-PersonDID production tới khi PA2 UniquenessThread + Registry-chuẩn dịch vụ land.** Lý do: (GV1) PersonDID **giả mạo được ở tầng neo-anchor** — `GenesisPerson` đúc được anchor did-string bất kỳ với controller của attacker vì HW_Key P-256 không verify on-chain → "1 người 1 suất" CHƯA đúng. **Đây là lỗ MÃ HOÁ-ANCHOR, KHÔNG phải lo ngại sybil-sinh trắc** (sinh trắc Secure Enclave đủ chống trùng người). (GV2) cổng chống-wash = Registry-tài nguyên thật phải land trước khi bật (`has_counterparty_consume` phải nối đúng — xem Math T-5). Cộng hưởng GV1×GV2 nếu cả hai chưa đóng = N-anchor-giả × wash-vỏ = rút ròng N×full-vault ~ phí-tx. Validator on-chain ĐÚNG (chứng minh money-safety độc lập — xem Math §7), lỗ nằm ở **giả định tầng trên chưa thoả**.
- **Gaming đã sạch phần on-chain:** self-consumption (GV) hợp lệ + khuyến khích; sybil-đa địa chỉ vô ích (D per-PersonDID *một khi* uniqueness anchor thoả); catch-up-burst chặn (trần luỹ kế `vest_ok`); không phải Terra (GreenBack có-back). Nhưng KHÔNG "hết lỗ" — 2 điều kiện trên + GV5 (hút-yield tối thiểu, kinh-tế) còn HỞ tới khi PA2+Registry land.
- **AbandonPhase1:** KHÔNG có redeemer on-chain riêng trong thiết kế hiện tại → endpoint `abandon-phase1` mặc định BỎ, thoát sớm PHA-1 đi qua anti-idle tự thu hồi (giao đội on-chain thêm redeemer nếu cần thoát sớm tự nguyện thật).

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai). (đã qua rà soát nội bộ)
- Đánh giá gaming nội bộ (không công khai) (GV1 anchor-uniqueness, GV2 wash-Registry, GV3 keeper-MVP, GV5 hút-yield).
- Memory: `anchor-uniqueness-limit`, `no-sybil-concern`, `magic-not-native-token`, `activation-getlamp-model`.
- Code (nguồn chân lý): `PhoenixKey-Validator/validators/activation_vault.ak` + `lib/phoenixkey/activation_logic.ak`.
- Tài liệu cùng bộ: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md), [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md), [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
