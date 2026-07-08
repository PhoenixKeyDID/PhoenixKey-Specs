# PhoenixKey — Activation · ĐẶC-TẢ ĐIỀU-HÀNH (cho lãnh-đạo)

> **CONTRACT v4.1 · 2026-07-07.** Bản điều-hành: quyết-định + lý-do + đánh-đổi + rủi-ro + trạng-thái + lộ-trình. Chi-tiết toán/invariant: `PhoenixKey-Activation-Feat-Math.md`. Đánh-giá gaming: `_activation-gaming-evaluation-v41.md`.
>
> Tài-liệu này KHÔNG lặp toán. Chỉ nêu điều lãnh-đạo cần để duyệt và chốt. Phân biệt rõ **đã-build** vs **spec-chờ-duyệt** vs **blocker-chờ-đội-khác**.

---

## 1. Tóm tắt một trang

**Activation là gì.** Luồng khởi-tạo người-dùng của PhoenixKey. User tạo khoá bằng vân tay (Secure Enclave), bấm **một nút GetLAMP** → nhận `D` LAMP vào **vault của chính mình** (khoá có-điều-kiện). **Khởi tạo MIỄN PHÍ** — Feecover lo phí gas, user không cần nạp tiền, không cần hiểu tỷ giá.

**Cơ chế hai pha.** `D = min(1001, ⌊pot/1_000_000⌋)`.
- **PHA-1 (ngày 1 → 1001):** LAMP khoá "có-điều-kiện" (chưa sở-hữu). Số dư LAMP **sinh MAGIC** (quyền tiêu dịch-vụ) — engine chỉ ĐỌC số dư, không đụng/không đốt LAMP. User hưởng = dòng MAGIC hằng ngày. Ngày nào không tiêu-thật đủ mức → thu-hồi 1 LAMP về pot (anti-idle).
- **PHA-2 (từ ngày 1002):** phần LAMP sống-sót **vest thành SỞ HỮU thật của user, 1 LAMP/ngày** — nhưng **có điều-kiện duy-trì**: mỗi epoch phải tiêu-thật đủ mới mở-khoá. Bỏ-bê 1001 epoch liên-tục → thu-hồi phần chưa-mở-khoá về pot (phần đã-sở-hữu KHÔNG bị đụng). LAMP đã vest → user rút/bán/tự-Gen tuỳ ý.

**Bản chất kinh-tế.** Đây là **hợp-đồng-trung-thành**: cam-kết dùng-thật 1001 ngày → được thưởng sở-hữu LAMP. Người bỏ-cuộc trả phần-chưa-kiếm nuôi người-mới (pot tự-nuôi, phản-chu-kỳ). Tách rõ **quyền-tiêu-dịch-vụ (MAGIC)** khỏi **sở-hữu-LAMP** — giá-trị đến từ TIÊU dịch-vụ thật, không từ nắm-giữ thụ-động.

**Ba thứ đã BỎ.** (a) Model VND-Genie cũ (nạp 200k VND) — nay miễn phí; (b) v2 unlock-bán-ngay — tạo cung chực-bán; (c) v3 mượn-không-sở-hữu — không có phần-thưởng cho người kiên-trì.

**Trạng thái một dòng.** Validator lõi **XONG** (173/173 test toàn-dự-án, 69/69 riêng activation_logic, + red-team sạch, PR chờ Tuân duyệt); spec chờ anh chốt; backend + Core ký đang chờ; **2 blocker kiến-trúc** ở đội MAGIC/CARP (engine Gen đọc-số-dư) và Registry (chuẩn dịch-vụ tiêu-tài-nguyên-thật).

> 🔴 **CỔNG GO/NO-GO (đọc trước khi duyệt bật production):** validator an-toàn nhưng **KHÔNG được mở GetLAMP-PersonDID trên production tới khi PA2 UniquenessThread + Registry-chuẩn land.** Hiện PersonDID giả-mạo được + cổng chống-wash chưa tồn tại → kẻ tấn-công có thể rút-ròng nhiều lần D (chi tiết §7 + verdict gaming). Enterprise/Org/Service DID (có parent-sig) KHÔNG dính lỗ này.

---

## 2. Bảng quyết-định (quyết | lý-do 4 trục | đánh-đổi)

| # | Quyết-định | Lý-do (a dài-hạn · b first-principles · c tối-ưu · d user+bền-vững) | Đánh-đổi (trung-thực) |
|---|---|---|---|
| **Q1** | **Model 2-PHA: có-điều-kiện → vesting-sở-hữu** (thay borrow-only v3) | (a) tạo hợp-đồng-trung-thành: kiên-trì 1001 ngày → sở-hữu thật; (d) người bỏ-cuộc nuôi người-mới. Bền hơn borrow-only (không điểm-thưởng) và vesting-bán-ngay (cung tức-thời). | Phức-tạp hơn borrow-only: thêm PHA-2, forfeit, 2 field datum. Vòng đời user rất dài. |
| **Q2** | **Tách quyền-tiêu-dịch-vụ (MAGIC) khỏi sở-hữu-LAMP** | (b) first-principles: LAMP = GATE mở tư-cách, biến quyết-định = lượng-MAGIC-tiêu (WP §7.2). Gen chỉ ĐỌC số dư → LAMP không cạn vì Gen. | Cần engine Gen mới (đọc-số-dư) — code MAGIC-repo cũ (fire→Treasury) lỗi-thời, phải viết lại. Blocker kiến-trúc. |
| **Q3** | **Cung-bán hoãn qua rào cam-kết** (không cung-tức-thời) | (a)(b) chỉ user tiêu-thật đủ 1001 ngày mới có LAMP-sống-sót để bán; vest nhỏ-giọt 1/ngày. Cung rải đều ≥1001 ngày, không sốc. | Người-dùng thật phải chờ rất lâu mới sở-hữu — trải-nghiệm "chậm được thưởng". |
| **Q4** | **Người bỏ-cuộc nuôi người-mới** (pot tự-nuôi phản-chu-kỳ) | (a)(d) anti-idle PHA-1 + forfeit PHA-2 + phí-thu-bằng-LAMP-theo-giá-trị (giá xuống → thu nhiều LAMP → pot phình đúng lúc thị-trường lạnh). | PHA-2 LAMP-vest RỜI-HỆ sang user → pot cần nguồn bù đủ. Rủi-ro pot-cạn thật hơn v3 (xem R2). |
| **Q5** | **Self-consumption HỢP-LỆ + khuyến-khích; cổng chống-wash = chuẩn Registry** (bỏ counterparty≠owner) | (b)(d) mỗi lượt tiêu tốn-phí → CARP/LAMP về Treasury → hệ CÓ LỢI. Cổng đúng = dịch-vụ phải tiêu tài-nguyên THẬT (lưu-trữ/băng-thông/tính-toán/lao-động), duyệt ở tầng Registry. | Đẩy trách-nhiệm chống-lạm-dụng sang **chuẩn Registry** — blocker mới ở Registry-team. Nếu Registry lỏng, wash-rỗng lọt. |
| **Q6** | **D keyed per-PersonDID** (1 người sinh-trắc = 1 suất) | (b) đa-địa-chỉ/đa-DID (Org/Device) KHÔNG nhân suất; sinh-trắc Secure Enclave lo uniqueness. | Uniqueness PersonDID nằm NGOÀI phạm-vi spec — dựa hoàn-toàn vào Secure Enclave. |

---

## 3. Ma-trận rủi-ro (rủi-ro | mức | giảm-thiểu)

| ID | Rủi-ro | Mức | Giảm-thiểu |
|---|---|---|---|
| **R1** | **Pot cạn khi nhiều user qua PHA-2 cùng lúc** (LAMP-vest rời-hệ, không quay lại pot) | 🟡 TRUNG (rủi-ro tham-số, không phải lỗi thiết-kế) | 3 nguồn nạp: phí-theo-giá-trị (phản-chu-kỳ) + anti-idle/forfeit + Treasury-topup. **[Cần mô-phỏng]** cân-đối tốc-độ-vest-ra vs tốc-độ-nạp; ngưỡng treasury-topup. |
| **R2** | **Wash-rỗng lọt nếu chuẩn Registry lỏng** (dịch-vụ giả không tiêu tài-nguyên thật) | 🟡 TRUNG | Chống-wash chuyển sang **duyệt Registry** — dịch-vụ rỗng không đăng-ký-được. Chưa bật anti-idle/epoch-gate production tới khi Registry-team có chuẩn danh-mục. |
| **R3** | **Engine Gen chưa spell-out on-chain** — /CARP mới có nguyên-lý "đọc-số-dư", chưa có validator; code cũ đốt LAMP | 🔴 BLOCKER kiến-trúc | Không nối Gen production tới khi MAGIC/CARP-team spell-out: đọc VaultDatum reference → drip MAGIC → KHÔNG-spend UTXO-LAMP. Validator vault (khoá+vest thuần) build/test được ĐỘC-LẬP trước. |
| **R4** | **Settlement mint-CARP-tự-do (nghi Terra)** | 🟢 THẤP | I-ACT-9: user trả CARP-đã-có, backing qua GreenBack backed-path (LAMP-haircut + tài-sản-cứng + 3-phanh). Activation KHÔNG mint/burn CARP/LAMP tự do. |
| **R5** | **Keeper MVP tin system-authority** — chưa có consume-event Registry thật, backend tự attest active/idle | 🟡 TRUNG (nợ MVP) | Chấp-nhận cho MVP; khoá cứng khi Registry consume-event sẵn sàng. Validator đo idle bằng GAP (lazy, an-toàn), không counter tăng-dần. |
| **R6** | **Forfeit 1001 epoch ≈ 13.7 năm KỂ TỪ đầu PHA-2 (≈16.4 năm kể từ GetLAMP) — rất dài** | 🟡 TRUNG (theo spec, có chủ-đích) | Theo spec: khoan-dung với người tạm-nghỉ; chỉ thu-hồi phần CHƯA-mở-khoá, KHÔNG đụng phần đã-sở-hữu. **[Lãnh-đạo có thể muốn rút ngắn]** — xem câu hỏi Q-C. |
| **R7** | **Uniqueness PersonDID ngoài phạm-vi** — dựa hoàn-toàn Secure Enclave | 🟢 THẤP (theo chủ-trương) | Chủ-trương: không nêu lo-ngại sybil; sinh-trắc Secure Enclave là đủ; `did_commit` per-PersonDID ràng 1-DID-1-vault. |

---

## 4. Trạng-thái triển-khai (xong / đang / chặn)

### ✅ XONG (có evidence)
- **Validator vault 2-pha** (`activation_vault.ak` + `activation_logic.ak`) — rework theo v4.1: `conditional_lamp`/`vested_unlocked`, ranh-giới-pha ngày 1001, anti-idle-reclaim PHA-1, **vest-GATED-per-epoch** + **forfeit-1001-idle-epoch**, ClaimVested ký-owner, Gen-đọc-số-dư-không-spend. **173/173 test pass + red-team sạch** (PR Validator #18). Đo idle bằng GAP (lazy), không counter tăng-dần.

### 🟡 ĐANG / CHỜ DUYỆT
- **Spec Feat-Math + gaming-eval** — PR #16, chờ anh chốt 4 điểm [CẦN CHỐT] (xem §6).
- **PR Validator #18** — chờ **Tuân** duyệt.
- **Backend (Long)** — GetLAMP orchestration, anti-idle job NGÀY (PHA-1), vest job (PHA-2), ClaimVested, fee_refill_lamp, GetMAGIC. Chờ triển-khai; verify bằng `curl` sau deploy.
- **Core / Enclave** — keygen vân tay, ký tx (GetLAMP/consume/ClaimVested), UI 1-nút + chỉ-báo-pha. Chờ ký.

### 🔴 CHẶN (blocker chờ đội khác)
- **B1 — Engine Gen đọc-số-dư (MAGIC/CARP-team).** /CARP chốt nguyên-lý, CHƯA spell-out validator on-chain. Code cũ đốt LAMP → lỗi-thời. Chặn việc nối Gen production. *(Validator vault khoá+vest thuần chạy được không cần Gen — build song song.)*
- **B2 — Chuẩn Registry dịch-vụ tiêu-tài-nguyên-thật (Registry-team).** Cổng chống-wash thay counterparty. Chặn bật anti-idle/epoch-gate production.
- **B3 (phụ) — GreenBack settlement + shadow-price (CARP-team); resolve API point-in-time (Database/Long); vesting-release kho→pot + phí-bằng-LAMP (LAMP-team).**

---

## 5. Lộ-trình mốc

**Nguyên-tắc thứ-tự:** phần khoá+vest thuần build/test ĐỘC-LẬP trước (không cần Gen) → spell-out engine Gen → nối Gen → chuẩn Registry + bật anti-idle → GreenBack → phản-chu-kỳ.

| Mốc | Nội-dung | Phụ-thuộc | Trạng-thái |
|---|---|---|---|
| **M1** | Validator pot (kế thừa `dist_treasury.ak`) + validator vault 2-pha + GetLAMP backend + conservation/D-cap test + ClaimVested | — (build ngay) | Validator vault ✅; pot + backend đang |
| **M2** | Anh chốt 4 điểm [CẦN CHỐT] §6; Tuân duyệt PR #18 | anh + Tuân | chờ |
| **M3** | MAGIC/CARP-team spell-out engine Gen đọc-số-dư → nối Gen | **B1** | chặn |
| **M4** | Registry-team chuẩn danh-mục dịch-vụ + cổng duyệt → bật anti-idle/epoch-gate production | **B2** | chặn |
| **M5** | GreenBack settlement (CARP) + shadow-price PHA-2 | B3 | chờ |
| **M6** | fee_refill_lamp phản-chu-kỳ + mô-phỏng cân-đối dòng-vest-ra (R1) | LAMP/backend | chờ |

---

## 6. Câu hỏi cần LÃNH-ĐẠO chốt

| # | Câu hỏi | Đề-xuất mặc-định (spec) | Vì sao cần anh |
|---|---|---|---|
| **Q-A** | **Engine Gen đọc-số-dư** — xác nhận cả Instant + Schedule /CARP đều CHỈ ĐỌC số dư (không nhánh nào còn đốt LAMP), và giao MAGIC/CARP-team spell-out validator on-chain (đọc reference-input → drip → không-spend). | Cả 2 đọc-số-dư; giao MAGIC/CARP-team viết engine. | Blocker kiến-trúc B1; quyết-định chiến-lược ai-viết-cái-gì. |
| **Q-B** | **Nhịp MAGIC: daily thật hay daily-tick + epoch-drip.** Anh muốn "MAGIC hàng ngày"; anti-idle/vest tick theo NGÀY nhưng gate-active + forfeit theo EPOCH. | tick-daily + drip-epoch, buffer-daily ở backend (không sửa on-chain). | Ảnh-hưởng cảm-giác user + độ phức-tạp on-chain. |
| **Q-C** | **Forfeit 1001 epoch ≈ 13.7 năm — giữ hay rút ngắn?** Rất dài; có chủ-đích khoan-dung nhưng có thể quá lỏng. | Giữ 1001 (đối-xứng 1001 ngày PHA-1). | Đánh-đổi khoan-dung vs pot-tái-tuần-hoàn. Chỉ đụng phần chưa-mở-khoá. |
| **Q-D** | **`MIN_MAGIC_TX` = 10% gen-able + granularity** (daily-gen-able vs cumulative). | 10% daily-gen-able từ `conditional_lamp`. | Tham-số TẠM (anh đặt 07-06); sub-open, buộc với nhịp Q-B. |
| **Q-E** | **Xác nhận PHA-2-vest-sở-hữu khớp WP §7.2** — đọc "cung-hoãn-qua-cam-kết 1001 ngày + nhỏ-giọt" là HỢP-LỆ, khác "cung-tồn-đọng chực-bán". | Hợp-lệ (đã qua rào cam-kết + vest nhỏ-giọt). | Diễn-giải whitepaper — cần anh xác nhận vì "vest-sở-hữu" là directive Activation-layer, WP không spell-out. |

---

## 7. Ghi trung-thực (không over-claim)

- **Đã-build vs spec vs blocker phân-biệt rõ:** chỉ validator vault lõi là ĐÃ-BUILD (173/173 + red-team sạch). Backend/Core/pot đang. Engine Gen + Registry là BLOCKER chờ đội khác.
- **Whitepaper-gap khai-báo:** "GetLAMP / pot / 2-pha / vest-sở-hữu-sau-1001-ngày / phí-thu-bằng-LAMP" KHÔNG có trong whitepaper — là **directive Activation-layer anh Aladin**. WP §7.2/§7.3 + /CARP hậu-thuẫn **nguyên-lý Gen-đọc-số-dư** rõ ràng, nhưng /CARP CHƯA spell-out cơ-chế on-chain (mới có nguyên-lý, chưa có validator).
- **Keeper MVP:** tin system-authority tới khi có consume-event Registry thật — nợ MVP có chủ-đích, đóng khi B2 xong.
- **🔴 ĐIỀU-KIỆN-TIÊN-QUYẾT chặn production (verdict gaming v4.1, `_activation-gaming-evaluation-v41.md`):** **KHÔNG mở GetLAMP-PersonDID production tới khi PA2 UniquenessThread + Registry-chuẩn-dịch-vụ land.** Lý do: (GV1) PersonDID hiện **giả-mạo được** (anchor P-256 không verify on-chain — [[anchor-uniqueness-limit]]) → "1 người 1 suất" CHƯA đúng; (GV2) cổng chống-wash = Registry-tài-nguyên-thật **chưa tồn tại** (`has_counterparty_consume`=False). Cộng-hưởng GV1×GV2 = N-DID-giả × wash-vỏ = **rút-ròng N×full-vault ~ phí-tx**. Validator on-chain ĐÚNG (red-team sạch), lỗ nằm ở **giả-định-tầng-trên chưa-thoả**.
- **Gaming đã sạch:** self-consumption (G3/GV) hợp-lệ + khuyến-khích; sybil-đa-địa-chỉ vô-ích (D per-PersonDID *một-khi* uniqueness thoả); không phải Terra (G1, GreenBack có-back). Nhưng KHÔNG "hết lỗ" — 2 blocker trên còn HỞ tới khi PA2+Registry land.
- **Artefact + gap-code cần biết trước khi backend dùng:** `AbandonPhase1` (thoát-sớm tự-nguyện PHA-1) **chưa có redeemer on-chain** trong v4.1 → endpoint `abandon-phase1` mặc-định BỎ (hoặc giao Tuân thêm nếu cần). `plutus.json` blueprint đã ở v4.1 đúng (verify `aiken build` không tạo diff — có ForfeitPhase2 + datum 9-field); nhưng **comment đầu file `activation_logic.ak` còn ghi "7 field/4 redeemer" (v4 cũ)** — báo Tuân sửa comment (không đổi struct/enum thực-tế đã 9-field/5-redeemer).
