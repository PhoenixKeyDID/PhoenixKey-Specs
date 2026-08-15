# PhoenixKey — Wakeme · ĐẶC-TẢ ĐIỀU-HÀNH (cho lãnh-đạo)

> **Module:** Wakeme (kích-hoạt nhận LAMP). **Loại doc:** Điều-hành. **Mô hình:** BẢN A — vest-thành-sở-hữu theo epoch (chốt 2026-07-30, Issue #67; đính-chính WakemeUsageRight + bucket `owned` 2026-07-31). **Cập nhật:** 2026-07-31.
> **Đối-tượng đọc:** lãnh-đạo — quyết-định + lý-do + đánh-đổi + rủi-ro + lộ-trình. Chi-tiết toán/invariant: [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md). Kỹ-thuật: [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md). Người-dùng: [PhoenixKey-Wakeme-Vi-Feat.md](./PhoenixKey-Wakeme-Vi-Feat.md).
>
> Tài-liệu này KHÔNG lặp toán. Chỉ nêu điều lãnh-đạo cần để duyệt và chốt.
>
> Khung **closed-loop-pot / "tấm-pin"** (2026-07-17, "user KHÔNG BAO GIỜ sở hữu LAMP") **ĐÃ BỊ LOẠI** (chốt 2026-07-30) → lưu ở `Legacy/wakeme-modelB-closed-loop-pot-2026-07-17/`. Khung v4.1 (vest 1 LAMP/ngày, redeemer `VestToOwner`/`ClaimVested`/`ForfeitPhase2`) **lỗi thời** — thay bằng bản A dưới đây.

---

## 1. Tóm tắt một trang

**Wakeme là gì.** Luồng khởi-tạo người-dùng của PhoenixKey. User tạo khoá bằng vân tay (Secure Enclave), bấm **một nút Wakeme** → nhận **`WakemeUsageRight`** LAMP vào **vault của chính mình** (khoá có-điều-kiện). **Khởi tạo MIỄN PHÍ** — Feecover lo phí gas, user không cần nạp tiền, không cần hiểu tỷ giá.

**Lượng cấp quyền-dùng `WakemeUsageRight` (chốt 2026-07-31).**
- `WakemeUsageRight = min(1001 LAMP, ⌊remaining_pot × 1001 / 10⁹⌋)` — tức **1001 phần tỷ** số-dư pot còn lại, **trần cứng 1001 LAMP**.
- Nhịp nhỏ-giọt mỗi đêm `D = WakemeUsageRight / 1001`; nhịp mỗi epoch (5 đêm) `= 5 × D`. Chọn tử-số **1001** (thay `10⁶` = 1000 phần tỷ cũ) **chính để `WakemeUsageRight ⋮ 1001`** → chia CHẴN ra **số-nguyên oildrop** mỗi đêm/epoch (đơn-vị on-chain nhỏ nhất; **1 LAMP = 10⁶ oildrop**). Với user sau (pot vơi) `D` có thể **< 1 LAMP** nhưng LUÔN là số-nguyên oildrop — "không lẻ dưới oildrop". Datum lưu LAMP theo **oildrop** (khớp footnote¹ Math).
- **`1 LAMP/đêm` và `5 LAMP/epoch` chỉ là TRƯỜNG-HỢP TỐI-ĐA** (khi `WakemeUsageRight = 1001 LAMP`) — đúng chắc-chắn cho **~1 triệu user đầu** (pot còn ≥ 1 tỷ LAMP nên trần luôn dính). User sau đó (pot vơi) nhận `WakemeUsageRight < 1001` → nhịp `D < 1 LAMP/đêm` theo tỷ-lệ, KHÔNG cố-định 1. Số-học số-nguyên (đơn-vị oildrop) **chốt-cứng với validator ở PA-1** (khoá `aiken check`).

**Ngưỡng "tiêu thật" — `MIN_MAGIC_CONSUME` (MIN).** Mức tiêu MAGIC tối thiểu để một mốc thời gian được tính là **active**. Là **THAM SỐ điều-chỉnh-được** (lý-tưởng: hàm biến-động theo cung-cầu MAGIC toàn cục), **áp cho CẢ Daily lẫn Epochy** — vì epoch dài hơn ngày, áp-lực-tiêu-thụ/đơn-vị-thời-gian ở Epochy nhẹ hơn Daily. **Tạm đặt = 1 MAGIC/ngày** tới khi chốt cơ chế biến-động (không kẹt sản phẩm ở việc chọn số).

**Cơ chế hai pha (BẢN A).**
- **Daily (ngày 1 → 1001):** LAMP khoá "có-điều-kiện" (`conditional`) sinh MAGIC — engine chỉ ĐỌC số dư, không đụng/không đốt LAMP. User hưởng = dòng MAGIC hằng ngày. Ngày nào tiêu < MIN (idle) → thu-hồi **`D` LAMP về pot** (`D` = nhịp đêm = `WakemeUsageRight/1001`, không cố-định 1) (anti-idle, `Reclaim`). Grace 7 ngày đầu. **Chưa có đường LAMP → owner ở Daily.**
- **Epochy (sau 1001 ngày đầu; 1 epoch = 5 ngày):**
  - Epoch **active** (tiêu ≥ MIN trong epoch) → `q = min(5·D, conditional)` LAMP chuyển từ `conditional` sang **`owned` (sở-hữu-hẳn, không bao giờ forfeit)** (`OwnEpoch`); `conditional −= q`, `owned += q`.
  - Epoch **idle** (tiêu < MIN) → **KHÔNG mất gì, DỒN sang epoch sau** (conditional giữ nguyên; KHÔNG về pot).
  - **Forfeit:** nếu **1001 epoch LIÊN TỤC** idle (gap `e − last_active_epoch ≥ 1001`) → toàn bộ `conditional` (KHÔNG đụng `owned`) → **pot**, đóng vault.
- **LAMP `owned` là của user thật — 3 lựa-chọn định-đoạt (anh chốt 2026-07-31):** (1) **để nguyên trong vault** → tiếp-tục sinh MAGIC với **tuổi-LAMP cao nhất** (giữ càng lâu, phần thưởng Gen càng lớn); (2) **redeem về ví** để giao-dịch; (3) dùng **mint CARP**. Không bao giờ bị pot/forfeit đụng. ⟹ "sở hữu" = LAMP rời `conditional` sang bucket `owned` bất-khả-thu-hồi, **KHÔNG bắt-buộc rời vault**.

**Bản chất kinh-tế.** **Hợp-đồng-trung-thành**: cam-kết tiêu-thật đủ lâu → được thưởng **sở-hữu LAMP thật** ở Epochy. Người bỏ-cuộc trả phần-chưa-kiếm nuôi người-mới (Daily thu 1/ngày; Epochy chỉ thu sau 1001 epoch liên-tục idle — rất khoan-dung cho người tạm nghỉ). Tách rõ **quyền-tiêu-dịch-vụ (MAGIC — account-trong-Vault, non-transferable, KHÔNG mint token)** khỏi **sở-hữu-LAMP** — giá-trị đến từ TIÊU dịch-vụ thật. LAMP tổng-cung cố-định 36 tỷ, KHÔNG burn (giảm lưu-hành = chuyển vào pot/Treasury, kế-toán).

**Nguồn nuôi pot (bền-vững).** Nguồn **chủ-yếu** làm pot đầy lên = **thặng-dư LAMP từ Feecover**: mỗi giao-dịch user tiêu MAGIC, hệ thu **phí cố-định bằng CARP** tương-ứng → dùng **mua lại LAMP khi giá thấp** (phản-chu-kỳ) hoặc **redeem LAMP từ GreenCheck** → nạp vào pot. Càng nhiều tiêu-thật, pot càng được bù → vòng bền-vững, KHÔNG in thêm LAMP. Cơ-chế mua-lại thuộc **Feecover/LAMP**; Wakeme là bên NHẬN (kế-toán dòng ở Math §9).

**Đã BỎ.** (a) Model VND-Genie (nạp 200k) — nay miễn phí; (b) closed-loop-pot "tấm-pin" (không cho sở hữu) — phản cam-kết Daily; (c) v4.1 vest-2-bước **cưỡng-bức** (VestToOwner→ClaimVested bắt-buộc): thay bằng `OwnEpoch` chuyển `conditional→owned` **một-bước** mỗi epoch active — LAMP `owned` **được giữ trong vault sinh MAGIC**, redeem về ví / mint CARP là **TUỲ-CHỌN** của user (không phải bước bắt-buộc).

> 🔴 **CỔNG GO/NO-GO (đọc trước khi duyệt bật production):** validator an-toàn tiền-tệ nhưng **KHÔNG được mở Wakeme-PersonDID trên production tới khi uniqueness person-level + Registry-chuẩn land.** Lý do: PersonDID **giả-mạo được ở tầng neo-anchor** (did-string đúc anchor bất-kỳ, HW_Key P-256 không verify on-chain — KHÔNG phải lỗ sinh-trắc) + cổng chống-wash chưa tồn tại → kẻ tấn-công rút-ròng nhiều suất D. Enterprise/Org/Service DID (có parent-sig) KHÔNG dính lỗ này. Xem §7.

→ Trạng-thái & tiến-độ hiện tại: [Wakeme STATUS](./PhoenixKey-STATUS.md#wakeme)

---

## 2. Bảng quyết-định (quyết | lý-do 4 trục | đánh-đổi)

| # | Quyết-định | Lý-do (a dài-hạn · b first-principles · c tối-ưu · d user+bền-vững) | Đánh-đổi (trung-thực) |
|---|---|---|---|
| **Q1** | **BẢN A: Epochy tiêu ≥ MIN → 5 LAMP về owner sở-hữu-hẳn; idle → DỒN; 1001-epoch-liên-tục → pot** (thay closed-loop-pot đã loại) | (a) giữ đúng cam-kết Daily "1001 đêm → cơ hội sở-hữu hoàn toàn"; (b) LAMP là phần-thưởng-trung-thành thật, không phải chỉ gate; (d) khoan-dung người tạm-nghỉ (dồn, không mất), chỉ mất khi bỏ hẳn rất lâu. | Pot chi ra LAMP thật cho người kiên-trì → cần nguồn nạp bù (xem R1). Vòng đời user dài. |
| **Q2** | **Tách quyền-tiêu-dịch-vụ (MAGIC) khỏi sở-hữu-LAMP** | (b) LAMP = GATE mở tư-cách; biến quyết-định = lượng-MAGIC-tiêu. Gen chỉ ĐỌC số dư → LAMP không cạn vì Gen. MAGIC = account-trong-Vault (KHÔNG native token). | Cần engine Gen (đọc-số-dư) on-chain — blocker kiến-trúc B1. |
| **Q3** | **Epoch active → `conditional` sang bucket `owned` (sở-hữu-hẳn, trong vault); user TỰ-CHỌN: giữ vault gen MAGIC / redeem về ví / mint CARP** | (b)(d) "sở hữu" = bất-khả-forfeit, KHÔNG buộc rời vault; giữ lại → **tuổi-LAMP cao nhất → Gen lợi nhất**; (c) một-bước `OwnEpoch` thay 2-bước cưỡng-bức v4.1. | Cần field `owned` trong datum + redeemer `Redeem` (owned→ví) TUỲ-CHỌN → datum/redeemer chốt-cứng ở PA-1 (`aiken check`). |
| **Q4** | **`MIN_MAGIC_CONSUME` = tham-số, áp cả Daily lẫn Epochy, tạm 1 MAGIC/ngày** | (c)(d) một ngưỡng thống-nhất; điều-chỉnh được theo cung-cầu về sau; không kẹt sản phẩm ở việc chốt số ngay. | Số tạm chưa tối-ưu; hàm biến-động theo cung-cầu là việc thiết-kế sau (anh chốt). |
| **Q5** | **Self-consumption HỢP-LỆ + khuyến-khích; cổng chống-wash = chuẩn Registry + counterparty ≠ owner** | (b)(d) mỗi lượt tiêu tốn-phí → về Treasury → hệ CÓ LỢI. Cổng đúng = dịch-vụ tiêu tài-nguyên THẬT (duyệt Registry) + đối-tác tiêu ≠ chính chủ. | Đẩy trách-nhiệm chống-lạm-dụng sang chuẩn Registry — blocker Registry-team (R2). |
| **Q6** | **D keyed per-PersonDID; 1 người = 1 PersonDID, 1 tổ-chức = 1 OrgDID** | (b) đa-địa-chỉ/đa-DID KHÔNG nhân suất. Uniqueness person-level = 2FA + Glint (VeData) + Spectra (LampNet) + MCR (OriLife). | Uniqueness person-level chưa land → hở tới khi các lớp trên vào (xem §7). Sinh-trắc-per-máy KHÔNG đủ. |

---

## 3. Ma-trận rủi-ro (rủi-ro | mức | giảm-thiểu)

| ID | Rủi-ro | Mức | Giảm-thiểu |
|---|---|---|---|
| **R1** | **Pot cạn khi nhiều user qua Epochy** (LAMP `owned` redeem về ví, rời pot) | 🟡 TRUNG (rủi-ro tham-số) | **Nguồn CHỦ-YẾU = thặng-dư Feecover** (phí cố-định CARP theo MAGIC tiêu-thụ/tx → mua lại LAMP giá thấp / redeem LAMP từ GreenCheck → pot) + anti-idle Daily + forfeit-1001-epoch + Treasury-topup. Van an-toàn nội-tại: `D` neo pot còn-lại → pot vơi thì `D` tự giảm. **[Cần mô-phỏng]** cân-đối tốc-độ-rời vs tốc-độ-nạp (kế-toán Math §9). |
| **R2** | **Wash-rỗng lọt nếu chuẩn Registry lỏng** | 🟡 TRUNG | Chống-wash = duyệt Registry (dịch-vụ rỗng không đăng-ký-được) + `has_counterparty_consume` (counterparty ≠ owner). Chưa bật anti-idle/epoch-gate production tới khi Registry-chuẩn + nối `has_counterparty_consume`. |
| **R3** | **Engine Gen chưa spell-out on-chain** | 🔴 BLOCKER kiến-trúc | Không nối Gen production tới khi MAGIC/CARP-team spell-out: đọc VaultDatum reference → drip MAGIC → KHÔNG-spend UTXO-LAMP, KHÔNG mint token. Validator vault build/test ĐỘC-LẬP trước. |
| **R4** | **Settlement mint-CARP-tự-do (nghi Terra)** | 🟢 THẤP | User trả CARP-đã-có, backing qua GreenBack. Wakeme KHÔNG mint/burn CARP/LAMP tự do. |
| **R5** | **Keeper MVP tin system-authority** — chưa có consume-event Registry thật | 🟡 TRUNG (nợ MVP) | Chấp-nhận cho MVP; owner escape-hatch đảm bảo user luôn nhận được phần đã-kiếm. Đo idle bằng GAP (lazy). Hướng production: thay `keeper_signed` bằng provider consume-event Merkle-inclusion + Registry-bonded. |
| **R6** | **Forfeit 1001 epoch ≈ 13.7 năm — rất dài** | 🟡 TRUNG (có chủ-đích) | Khoan-dung người tạm-nghỉ; chỉ thu phần CHƯA-kiếm (`conditional` còn lại), KHÔNG đụng LAMP đã về ví owner. **[Lãnh-đạo có thể muốn rút ngắn]** — Q-C. |
| **R7** | **Uniqueness anchor PersonDID** — did-string đúc anchor bất-kỳ (lỗ mã-hoá) | 🔴 GATE (xem §7) | Uniqueness person-level (2FA + Glint + Spectra + MCR) + PA-2 UniquenessThread (đóng CID-1) chặn Wakeme-PersonDID production tới khi land. Org/Service DID (parent-sig) không dính. |

---

## 4. Luật chặn production (áp-dụng khi nối phần phụ-thuộc đội khác)

- **B1 — Engine Gen đọc-số-dư (MAGIC/CARP-team).** Chỉ nối Gen production sau khi spell-out validator on-chain đọc-số-dư (KHÔNG spend/đốt LAMP). Validator vault build/chạy độc-lập.
- **B2 — Chuẩn Registry dịch-vụ tiêu-tài-nguyên-thật + nối `has_counterparty_consume`.** Anti-idle/epoch-gate CHỈ bật production sau khi land.
- **B3 — Wakeme-PersonDID production chặn tới khi uniqueness person-level land** (2FA + Glint/VeData + Spectra/LampNet + MCR/OriLife; PA-2 UniquenessThread đóng CID-1 là điều-kiện CẦN). Org/Service/Enterprise DID (parent-sig) không dính.
- **B4 (phụ) — GreenBack settlement + shadow-price (CARP-team); nạp pot ban đầu + ramp-up (LAMP-team)** trước khi bật GetMAGIC production.

---

## 5. Lộ-trình mốc

**Nguyên-tắc thứ-tự:** phần vault (khoá+Epochy) build/test ĐỘC-LẬP trước → spell-out engine Gen → nối Gen → chuẩn Registry + bật anti-idle → GreenBack → uniqueness person-level → mở PersonDID.

| Mốc | Nội-dung | Phụ-thuộc |
|---|---|---|
| **M1 (PA-1)** | Validator vault bản A: Daily Reclaim + Epochy `OwnEpoch`(active→owner) + **DỒN idle** + **forfeit gap-1001** (bỏ idle→pot-ngay) + nối `has_counterparty_consume`; gom cùng 2-of-2 device-key. `aiken check` xanh + Math/Tech khoá theo code. | đội on-chain |
| **M2** | Pot (`dist_treasury`) + Wakeme backend + conservation/D-cap test | LAMP/backend |
| **M3** | MAGIC/CARP-team spell-out engine Gen đọc-số-dư → nối Gen | **B1** |
| **M4** | Registry-chuẩn danh-mục + nối `has_counterparty_consume` → bật anti-idle/epoch-gate | **B2** |
| **M5** | Uniqueness person-level land → mở Wakeme-PersonDID production | **B3** |
| **M6** | GreenBack settlement (CARP) + shadow-price | B4 |
| **M7** | pot ramp-up + mô-phỏng cân-đối dòng-vest-ra (R1) | LAMP/backend |

---

## 6. Câu hỏi cần LÃNH-ĐẠO chốt

| # | Câu hỏi | Đề-xuất mặc-định (spec) | Vì sao cần lãnh-đạo |
|---|---|---|---|
| **Q-A** | **Engine Gen đọc-số-dư** — xác nhận Instant + Schedule /CARP đều CHỈ ĐỌC số dư; giao MAGIC/CARP-team spell-out validator. | Cả 2 đọc-số-dư. | Blocker kiến-trúc B1. |
| **Q-B** | **`MIN_MAGIC_CONSUME`: giữ tham-số cố-định (tạm 1 MAGIC/ngày) hay chuyển hàm biến-động theo cung-cầu.** | Tham-số cố-định trước, hàm biến-động sau. | Quyết-định thiết-kế kinh-tế (anh chốt). |
| **Q-C** | **Forfeit 1001 epoch ≈ 13.7 năm — giữ hay rút ngắn?** | Giữ 1001 (đối-xứng 1001 ngày Daily). | Đánh-đổi khoan-dung vs pot-tái-tuần-hoàn. Chỉ đụng phần chưa-kiếm. |
| **Q-D** | **`q` Epochy (lượng chuyển `conditional→owned` mỗi epoch active)** | `q = min(5·D, conditional)` (5 đêm × nhịp `D=WakemeUsageRight/1001`; `=5 LAMP` chỉ khi `WakemeUsageRight=1001`). | Nhịp vest-ra ảnh-hưởng R1. |

---

## 7. Điều-kiện-tiên-quyết chặn production (GO/NO-GO)

- **🔴 KHÔNG mở Wakeme-PersonDID production tới khi uniqueness person-level + Registry-chuẩn land.** Lý do: (GV1) PersonDID **giả-mạo được ở tầng neo-anchor** — `GenesisPerson` đúc anchor did-string bất-kỳ với controller attacker vì HW_Key P-256 không verify on-chain → "1 người 1 suất" CHƯA đúng. **Lỗ MÃ-HOÁ-ANCHOR, KHÔNG phải sybil-sinh-trắc** — nhưng **sinh-trắc-per-thiết-bị cũng KHÔNG đủ** (chỉ đảm bảo 1-DID/máy; 1 người N máy → N DID). Person-level thật cần: **2FA + Glint (ZK-uniqueness, VeData) + Spectra (chứng-từ, LampNet) + MCR (nhận-diện đa-đặc-điểm trên 1 video selfie, thuật-toán tất-định, OriLife).** PA-2 UniquenessThread (SMT non-membership) đóng "anchor-thứ-hai" (CID-1) = điều-kiện CẦN, chưa ĐỦ. (GV2) cổng chống-wash Registry + `has_counterparty_consume` phải land.
- **Keeper MVP:** tin system-authority tới khi có consume-event Registry thật (nợ MVP có chủ-đích).
- **On-chain money-safety đã sạch:** LAMP không rời-hệ trái-phép (đích cứng pot | ví owner); sybil-đa-địa-chỉ vô-ích *một-khi* uniqueness anchor thoả; keeper KHÔNG đoạt được LAMP. Nhưng KHÔNG "hết lỗ" — GV1×GV2 còn HỞ tới khi person-level + Registry land.

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai). Mô hình bản A: Issue #67 (chốt 2026-07-30).
- Code (nguồn chân-lý): `PhoenixKey-Validator/lib/phoenixkey/wakeme_logic.ak` + `validators/wakeme_vault.ak` (BẢN A, 491 checks/0 err; rename định-danh activation→wakeme 2026-07-31, hash bất-biến `3f6e5bf6…f23`; on-chain redeploy đi theo PA-1).
- Tài-liệu cùng bộ: [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md), [Math](./PhoenixKey-Wakeme-Math.md), [Tech](./PhoenixKey-Wakeme-Tech.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
