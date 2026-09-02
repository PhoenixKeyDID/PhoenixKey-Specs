# PhoenixKey — Wakeme · ĐẶC-TẢ ĐIỀU-HÀNH (cho lãnh-đạo)

> **Module:** Wakeme (kích-hoạt nhận LAMP). **Loại doc:** Điều-hành. **Mô hình:** BẢN A — vest-thành-sở-hữu theo epoch (chốt 2026-07-30, Issue #67; đính-chính WakemeUsageRight + bucket `owned` 2026-07-31). **Cập nhật:** 2026-09-02.
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

> 🔴 **CỔNG GO/NO-GO (đọc trước khi duyệt bật production):** validator an-toàn tiền-tệ nhưng **KHÔNG được mở Wakeme-PersonDID trên production tới khi uniqueness person-level + Registry-chuẩn land.** Lý do: **Sybil mức NGƯỜI** — một người tự mint được bao nhiêu PersonDID tuỳ ý, mỗi cái đều hợp khuôn (`pop_bind.ak:83-93`) → rút-ròng nhiều suất D; cộng cổng chống-wash chưa tồn tại. Enterprise/Org/Service DID (có parent-sig) KHÔNG dính lỗ này. Vế "giả-mạo ở tầng neo-anchor" của bản trước **đã đóng 2026-08-12** và không còn là lý do chặn. Chi tiết + bằng chứng: §7.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## 2. Bảng quyết-định (quyết | lý-do 4 trục | đánh-đổi)

| # | Quyết-định | Lý-do (a dài-hạn · b first-principles · c tối-ưu · d user+bền-vững) | Đánh-đổi (trung-thực) |
|---|---|---|---|
| **Q1** | **BẢN A: Epochy tiêu ≥ MIN → 5 LAMP về owner sở-hữu-hẳn; idle → DỒN; 1001-epoch-liên-tục → pot** (thay closed-loop-pot đã loại) | (a) giữ đúng cam-kết Daily "1001 đêm → cơ hội sở-hữu hoàn toàn"; (b) LAMP là phần-thưởng-trung-thành thật, không phải chỉ gate; (d) khoan-dung người tạm-nghỉ (dồn, không mất), chỉ mất khi bỏ hẳn rất lâu. | Pot chi ra LAMP thật cho người kiên-trì → cần nguồn nạp bù (xem R1). Vòng đời user dài. |
| **Q2** | **Tách quyền-tiêu-dịch-vụ (MAGIC) khỏi sở-hữu-LAMP** | (b) LAMP = GATE mở tư-cách; biến quyết-định = lượng-MAGIC-tiêu. Gen chỉ ĐỌC số dư → LAMP không cạn vì Gen. MAGIC = account-trong-Vault (KHÔNG native token). | Cần engine Gen (đọc-số-dư) on-chain — blocker kiến-trúc B1. |
| **Q3** | **Epoch active → `conditional` sang bucket `owned` (sở-hữu-hẳn, trong vault); user TỰ-CHỌN: giữ vault gen MAGIC / redeem về ví / mint CARP** | (b)(d) "sở hữu" = bất-khả-forfeit, KHÔNG buộc rời vault; giữ lại → **tuổi-LAMP cao nhất → Gen lợi nhất**; (c) một-bước `OwnEpoch` thay 2-bước cưỡng-bức v4.1. | Cần field `owned` trong datum + redeemer `Redeem` (owned→ví) TUỲ-CHỌN → datum/redeemer chốt-cứng ở PA-1 (`aiken check`). |
| **Q4** | **`MIN_MAGIC_CONSUME` = tham-số, áp cả Daily lẫn Epochy, tạm 1 MAGIC/ngày** | (c)(d) một ngưỡng thống-nhất; điều-chỉnh được theo cung-cầu về sau; không kẹt sản phẩm ở việc chốt số ngay. | Số tạm chưa tối-ưu; hàm biến-động theo cung-cầu là việc thiết-kế sau (anh chốt). |
| **Q5** | **Self-consumption HỢP-LỆ + khuyến-khích; cổng chống-wash = chuẩn Registry + counterparty ≠ owner** | (b)(d) mỗi lượt tiêu tốn-phí → về Treasury → hệ CÓ LỢI. Cổng đúng = dịch-vụ tiêu tài-nguyên THẬT (duyệt Registry) + đối-tác tiêu ≠ chính chủ. | Đẩy trách-nhiệm chống-lạm-dụng sang chuẩn Registry — blocker Registry-team (R2). |
| **Q6** | **D keyed per-PersonDID; 1 người = 1 PersonDID, 1 tổ-chức = 1 OrgDID** | (b) đa-địa-chỉ/đa-DID KHÔNG nhân suất. Uniqueness person-level = 2FA + Spectra (chứng-từ, LampNet) + MCR (nhận-diện đa-đặc-điểm, OriLife) + một kênh ZK-uniqueness **hôm nay CHƯA CÓ CHỦ**: catalog Glint chỉ có 5 primitive nguyên-tử (P1 commit, P2 range, P3 membership, P4 quan-hệ tuyến-tính, P6 nullifier) — không cái nào phát-biểu mệnh-đề về con người (đang sống/là chính chủ/duy-nhất một người, Glint-Math.md:95-113), trên nền uniqueness ở tầng neo-anchor. | Uniqueness person-level chưa land → hở tới khi các lớp trên vào (xem §7). Sinh-trắc-per-máy KHÔNG đủ. **MCR còn trong danh-sách nhưng số đo 2026-08-08 nói nó chưa gánh được vai này**: chưa có luồng cho người, và ở 1:N thì trần `N*` chưa tới 1 hồ-sơ tại điểm vận-hành đo được (§7.1). Giữ hay gỡ MCR khỏi danh-sách = **Q-E**. |

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
| **R7** | **Uniqueness anchor PersonDID** — did-string đúc anchor bất-kỳ (lỗ mã-hoá) | 🔴 GATE (xem §7) | Uniqueness person-level (2FA + Spectra + MCR — MCR chưa có số đo trên người, xem R8/Q-E; kênh ZK-uniqueness thứ ba dự-tính từ Glint hôm nay CHƯA CÓ CHỦ trong catalog, Glint-Math.md:95-113) + PA-2 UniquenessThread (đóng CID-1) chặn Wakeme-PersonDID production tới khi land. Org/Service DID (parent-sig) không dính. |
| **R8** | **Neo mốc vào năng-lực chưa có số đo trên ĐÚNG đối-tượng** — cụ-thể: coi nhận-diện đa-đặc-điểm (MCR) là cổng 1:N tìm trùng người | 🟡 TRUNG | Cổng 1:N như vậy **chưa tồn tại**: MCR chưa có luồng cho người, **0 mẫu người**, không có số chống-giả-mạo-trình-diện. Số duy nhất có là trên **cây**: tại điểm vận-hành `FAR 0,013` thì `FRR` đã `0,73`, trần `N* = 0,01/f ≈ 0,77` hồ-sơ (§7.1). **`FAR 0,013` là điểm vận-hành, KHÔNG phải sàn công-nghệ** — siết ngưỡng hạ được `f`, nhưng `FRR` tăng theo, nên ràng-buộc thật là cặp `(FAR, FRR)`. ⟹ mốc Wakeme chỉ nên neo vào năng-lực có **số đo trên đúng đối-tượng**; MCR có tiếp-tục đứng trong B3/M5/R7/Q6 hay không = **Q-E**. |

---

## 4. Luật chặn production (áp-dụng khi nối phần phụ-thuộc đội khác)

- **B1 — Engine Gen đọc-số-dư (MAGIC/CARP-team).** Chỉ nối Gen production sau khi spell-out validator on-chain đọc-số-dư (KHÔNG spend/đốt LAMP). Validator vault build/chạy độc-lập.
- **B2 — Chuẩn Registry dịch-vụ tiêu-tài-nguyên-thật + nối `has_counterparty_consume`.** Anti-idle/epoch-gate CHỈ bật production sau khi land.
- **B3 — Wakeme-PersonDID production chặn tới khi uniqueness person-level land.** Đang chờ **hai** thứ, không thứ nào đã xong ở mức production: (i) ít nhất MỘT kênh xác-nhận trùng-người ở mức NGƯỜI chạy thật — hôm nay ứng-viên thật duy-nhất là Spectra (chứng-từ, LampNet), verifier thuộc Phase 2 bên VeData, chưa nối. Kênh ZK-uniqueness từng dự-tính giao Glint **CHƯA CÓ CHỦ** trong catalog — 5 primitive nguyên-tử của Glint (P1/P2/P3/P4/P6) không cái nào phát-biểu mệnh-đề về con người (Glint-Math.md:95-113); đây là mục chờ, KHÔNG tính vào lộ-trình gỡ chặn B3. (ii) uniqueness ở tầng neo-anchor (PA-2 UniquenessThread, đóng CID-1) — điều-kiện CẦN, trạng-thái đo được ghi ở [PhoenixKey-STATUS.md §Anchorme](./PhoenixKey-STATUS.md#anchorme). **MCR vẫn nằm trong danh-sách lớp person-level, nhưng theo số đo hôm nay nó KHÔNG đóng được B3** (§7.1: 0 mẫu người; ở 1:N thì trần `N*` chưa tới 1 hồ-sơ tại điểm vận-hành đo trên cây) — đừng xếp lịch gỡ chặn quanh nó. Giữ hay gỡ MCR khỏi B3 = **Q-E**, chờ chủ dự-án. Org/Service/Enterprise DID (parent-sig) không dính.
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
| **M5** | Uniqueness person-level land → mở Wakeme-PersonDID production. **Chưa có ngày** — chờ verifier Spectra (**LampNet** — `Glint-Math.md:21`, Phase 2) + uniqueness tầng neo-anchor. Kênh ZK-uniqueness từng dự-tính giao Glint **CHƯA CÓ CHỦ** trong catalog (Glint-Math.md:95-113), nên không tính là đường gỡ chặn. MCR còn trong danh-sách lớp person-level nhưng chưa có số đo trên người, nên **không tính nó là đường gỡ chặn** (§7.1, Q-E). Mốc GIỮ NGUYÊN, chỉ ghi đúng thứ nó chờ. | **B3** |
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
| **Q-E** | **Nhận-diện đa-đặc-điểm (MCR) có tiếp-tục đứng trong danh-sách lớp person-level ở Q6 / R7 / B3 / M5 không?** Số đo 2026-08-08: 0 mẫu người, không có số chống-giả-mạo-trình-diện; trên cây, tại `FAR 0,013` thì `FRR 0,73` và trần 1:N `N* ≈ 0,77` hồ-sơ (§7.1). **P1 — GIỮ** MCR trong cả bốn chỗ, kèm ghi chú "chưa có số đo trên người, không tính là đường gỡ chặn" (đúng bản này). **P2 — GỠ** MCR khỏi cả bốn chỗ, chuyển nó sang vai 1:1 tăng-cường cho luồng chứng-từ ở Knowme. **Đề-xuất: P2** — B3/M5 là lịch chặn production, để trong đó một năng-lực chưa đo trên người thì lịch đọc ra lạc-quan hơn thực-tế. | Đây là **đổi phạm-vi lớp person-level**, không phải sửa số ⇒ thuộc quyền chủ dự-án. Không chốt thì Q6/R7/B3/M5 giữ nguyên P1 và mọi bên đọc lộ-trình vẫn thấy một đường gỡ chặn không có thật. |

---

## 7. Điều-kiện-tiên-quyết chặn production (GO/NO-GO)

- **🔴 KHÔNG mở Wakeme-PersonDID production tới khi uniqueness person-level + Registry-chuẩn land.** Lý do: (GV1) **Sybil mức NGƯỜI** — một người tự mint được bao nhiêu PersonDID tuỳ ý, mỗi cái một `rand_256` mới và controller của chính mình, và mỗi cái đều hợp khuôn. Validator tự ghi phạm-vi không đóng này ở `lib/phoenixkey/pop_bind.ak:83-86`. **Sinh-trắc-per-thiết-bị KHÔNG đủ**: mẫu nằm trong Secure Enclave từng máy và không so chéo máy, nên nó gác được "1 DID mỗi máy", không gác được "1 suất mỗi người" — 1 người N máy → N DID (`pop_bind.ak:88-93`).

**Vế "giả-mạo ở tầng neo-anchor" của GV1 ĐÃ ĐÓNG, không còn là lý do chặn.** Bản trước của dòng này ghi `GenesisPerson` đúc được anchor mang did-string bất-kỳ với controller của kẻ tấn công. Điều đó đúng cho tới 2026-08-12; nay `did` không còn là bytearray tự do: cả hai nhánh genesis đều ép `pop_bind.pop_bind_ok(child_datum, ub, rand_256)` (`lib/phoenixkey/state_nft_logic.ak:166` cho `GenesisPerson`, `:192` cho `GenesisChild`), và mọi đường tạo/huỷ anchor phải chi shard-thread tương ứng (`validators/taad.ak:83-86`). Chọn did-string của nạn nhân rồi đặt controller của mình nay đòi second-preimage blake2b. CID-1 🟢 đã đóng — `PhoenixKey-Anchorme-Math.md:625`.

Đây là **điều-kiện CẦN đã thoả, chưa ĐỦ**: uniqueness tên-anchor và uniqueness người là hai bài toán khác nhau, và chỉ bài đầu đã xong. Person-level thật cần: **2FA + Spectra (chứng-từ, LampNet) + MCR (nhận-diện đa-đặc-điểm, OriLife — hôm nay chỉ dùng được ở vai so-khớp 1:1, chưa có số đo trên người; §7.1) + một kênh ZK-uniqueness hôm nay CHƯA CÓ CHỦ trong catalog Glint (5 primitive nguyên-tử P1/P2/P3/P4/P6, không cái nào phát-biểu mệnh-đề về con người — Glint-Math.md:95-113).** PA-2 UniquenessThread (SMT non-membership) đóng "anchor-thứ-hai" (CID-1) = điều-kiện CẦN, chưa ĐỦ. (GV2) cổng chống-wash Registry + `has_counterparty_consume` phải land.
- **Keeper MVP:** tin system-authority tới khi có consume-event Registry thật (nợ MVP có chủ-đích).
- **On-chain money-safety đã sạch:** LAMP không rời-hệ trái-phép (đích cứng pot | ví owner); sybil-đa-địa-chỉ vô-ích *một-khi* uniqueness anchor thoả; keeper KHÔNG đoạt được LAMP. Nhưng KHÔNG "hết lỗ" — GV1×GV2 còn HỞ tới khi person-level + Registry land.

### 7.1 Nhận-diện đa-đặc-điểm: 1:1 được, 1:N thì không

**Năng-lực hiện có.** MCR (MultiComponentRecognition) trong hệ sinh-thái là bộ nhận-diện **cây / quả / vật-nuôi**: gộp nhiều đặc-điểm hình-ảnh của cùng một cá-thể để nhận lại nó về sau. Nó **không có luồng cho người**, không có tập mẫu người, và không có số liveness/chống-giả-mạo-trình-diện (PAD). Một điểm hay bị đọc sai: "tất-định" ở đây nghĩa là **chạy lại cho cùng kết-quả** (cố-định gieo hạt ngẫu-nhiên trong bước ước-lượng hình-học), KHÔNG có nghĩa "phán-quyết chắc-chắn" — quyết-định vẫn là **một ngưỡng đặt trên điểm giống-nhau liên-tục**, và ngưỡng đó còn đang hiệu-chỉnh.

**Vì sao 1:N không dùng được — ràng-buộc là một CẶP số, không phải một con số.** Một cổng 1:N tra `N` hồ-sơ, mỗi phép so có xác-suất nhận-nhầm `f`; cổng chỉ dùng được khi số nhận-nhầm kỳ-vọng `N·f` còn nhỏ — lấy ngân-sách 1% thì trần là `N* ≈ 0,01/f`.

Chỗ này phải đọc cho đúng, vì rất dễ đọc hỏng: **`f` KHÔNG phải sàn của công-nghệ.** Nó là một toạ-độ trên đường cong ngưỡng — siết ngưỡng thì `f` giảm và tỉ-lệ nhận-đúng `TAR` giảm theo. Đường cong đo được (luật đếm phiếu hai kênh tốt nhất, gallery **cây**, impostor lấy cùng vườn ≤25 m, đo 2026-08-08):

| ngưỡng | TAR (≈1−FRR) | FAR (`f`) | `N* = 0,01/f` |
|---|---|---|---|
| dino ≥0,50 · vỏ ≥3 | 0,425 | 0,087 | 0,11 |
| dino ≥0,55 · vỏ ≥4 | 0,375 | 0,050 | 0,20 |
| dino ≥0,60 · vỏ ≥5 | 0,267 | **0,013** | 0,77 |

`FAR 0,013` ở dòng cuối là **một điểm vận-hành**, không phải trần công-nghệ: nới ngưỡng thì `f` to hơn, siết ngưỡng thì `f` nhỏ hơn. Cái chặn thật là **cặp** `(FAR, FRR)` phải đạt cùng lúc. Một cổng ở mức người cần cỡ `FAR ≤ 1e-4` **đồng thời** `FRR ≤ 0,03`; ngay tại điểm lỏng nhất còn nghe được trong bảng — `FAR = 0,013`, tức **lỏng hơn mốc cần khoảng 130 lần** — `FRR` đã là **0,73**: gần ba phần tư người thật bị từ-chối. Siết ngưỡng cho `f` xuống thì `FRR` chỉ tệ thêm. Và toàn bộ bảng đo trên **cây**, ở điều-kiện dễ hơn người (cá-thể đứng yên, nền ổn-định, tập nhỏ); trên người thì **chưa có một mẫu nào**. Phát-biểu đúng vì thế là: khoảng-cách tới cổng-toàn-dân là nhiều bậc độ-lớn **và chưa từng được đo trên đúng đối-tượng** — không phải "lớp kỹ-thuật này có sàn `f = 1,3%`".

**Thêm kênh có nới trần hay không — tuỳ kênh mới lấy từ NGUỒN nào.** Cùng đợt đo 2026-08-08 cho hai kết-quả **ngược chiều nhau**, và gộp chúng thành một quy-luật chung là cách đọc sai:

- **Cùng nguồn thì KHÔNG.** Bốn kênh vùng `CTX/PLANT/BASE/LEAF` là bốn ô cắt hình-học của **cùng một khung hình** qua **cùng một mạng**: `ρ = 0,815` ⟹ `M_eff = 1,16 / 4 kênh`; tổng có trọng-số `AUC 0,7416` còn **thua** kênh đơn tốt nhất (`LEAF 0,7521`). Đây là một **kết-quả âm**, và nó chỉ nói về việc chồng thêm kênh **cùng nguồn**.
- **Khác nguồn thì CÓ.** Cặp `DINOv2 ⟂ vỏ-thân` — mạng học sâu so với SIFT/RANSAC đếm điểm khớp trên vùng ảnh khác, hai cơ-chế khác bản-chất — đo được `ρ = −0,086` ⟹ `M_eff = 2,19 / 2 kênh`, và gộp **hơn từng kênh đứng riêng ở MỌI mức FAR**, khoảng-cách còn **nới ra khi đòi FAR thấp hơn** (+16% ở `FAR 0,087` → +39% ở `FAR 0,013`).

Hệ-quả cho thiết-kế: đừng xin thêm "bộ phận" cắt ra từ cùng một ảnh qua cùng một mạng — bốn cái như thế đáng 1,16 cái. Muốn thêm một phiếu thật thì phải thêm **một loại bằng-chứng có nguyên-nhân khác** (chứng-từ, số định-danh, tín-hiệu ZK). Cùng lập-luận đó trả lời luôn ca **sinh đôi cùng trứng**: mọi đặc-điểm do gen quy-định đều chung một nguyên-nhân, tức `ρ → 1` ⟹ `M_eff → 1` bất kể thêm bao nhiêu bộ phận — thoát ra phải bằng thành-phần **không do gen quy-định**, không bằng cách đếm thêm.

Lưu ý kết-quả khác-nguồn này **không** cứu được cổng 1:N ở trên: `M_eff = 2,19` mới nhân đôi số phiếu hiệu-dụng, còn để chạy 1:N ở quy-mô dân-số thì `f` phải xuống nhiều bậc độ-lớn.

**Vai đúng: tầng độc-lập thứ hai cho phép so 1:1** — người trước ống kính ⟷ ảnh trong giấy-tờ đã nộp. Ở 1:1 chỉ có **một** phép so nên ngân-sách sai không bị nhân với `N`, và nó đứng cạnh một kênh khác hẳn về bản-chất (chứng-từ) — đúng cấu-hình khác-nguồn đã đo được là có lợi ở trên. Đây là lớp **tăng-cường** cho luồng chứng-từ; nó KHÔNG thay được uniqueness person-level.

> **Còn treo — chờ chủ dự-án chốt (Q-E, §6):** có gỡ nhận-diện đa-đặc-điểm khỏi danh-sách lớp person-level ở Q6 / R7 / B3 / M5 hay không. Bản này **giữ nguyên** các mốc và blocker, chỉ ghi đúng số đo — gỡ là quyết-định thiết-kế, không phải sửa số.

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai). Mô hình bản A: Issue #67 (chốt 2026-07-30).
- Code (nguồn chân-lý): `PhoenixKey-Validator/lib/phoenixkey/wakeme_logic.ak` + `validators/wakeme_vault.ak` (BẢN A, 491 checks/0 err; rename định-danh activation→wakeme 2026-07-31, hash bất-biến `3f6e5bf6…f23`; on-chain redeploy đi theo PA-1).
- Tài-liệu cùng bộ: [Vi-Feat](./PhoenixKey-Wakeme-Vi-Feat.md), [Math](./PhoenixKey-Wakeme-Math.md), [Tech](./PhoenixKey-Wakeme-Tech.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
