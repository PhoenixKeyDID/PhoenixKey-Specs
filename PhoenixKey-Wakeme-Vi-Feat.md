# PhoenixKey — Wakeme: Chiếc Đèn và Điều ước của bạn

> **Module:** Wakeme (GetLAMP — kích-hoạt nhận LAMP). **Loại doc:** Feature (tiếng Việt, hướng người-dùng/sản-phẩm). **Ngày:** 2026-07-09 · **Sửa PHA-2 (sở-hữu dần mỗi Kỳ):** 2026-07-12.
> **Đối-tượng đọc:** người dùng PhoenixKey và đội sản-phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **Wakeme là gì**, **được lợi gì**, **làm sao giữ**, bằng ngôn ngữ đời thường.
> Chi-tiết toán/bất-biến ở [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md); kỹ-thuật + API ở [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md); quyết-định điều-hành ở [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).

---

## 1. Một câu là gì

**Wakeme là bước "thắp đèn": bạn tạo ví bằng vân tay, bấm một nút, và nhận về một túi LAMP vào ví của chính mình. Túi LAMP đó mỗi ngày sinh ra "điều ước" (MAGIC) để bạn dùng dịch vụ — và nếu bạn dùng đều đặn đủ lâu, phần LAMP sống sót sẽ thật sự thành CỦA BẠN.**

Hình dung đơn giản:
- **Chiếc đèn** = túi LAMP trong ví bạn.
- **Điều ước** = MAGIC chiếc đèn sinh ra mỗi ngày.
- Bạn **xoa đèn** (dùng dịch vụ) để điều ước không tan. Xoa đều đặn đủ lâu → **chiếc đèn thành của bạn**.

---

## 2. Vấn đề nó giải

Trước đây, muốn bắt đầu bạn phải **nạp 200.000đ** để đổi lấy LAMP và một ít phí mạng (mô hình "Genie" cũ). Ba cái vướng:

1. Người mới phải bỏ tiền trước khi hiểu mình nhận được gì.
2. Bạn phải tự hiểu tỷ giá, tự lo phí mạng — rối.
3. Hệ thống phải chi phí thật cho từng người mới → khó bền.

**Wakeme mới bỏ hết ba cái đó:**
- **Miễn phí khởi tạo.** Không nạp đồng nào. Bấm một nút là có đèn.
- **Không cần hiểu tỷ giá.** Bạn chỉ thấy một nút và dòng điều ước hằng ngày. Phí mạng do hệ thống (Feecover) lo thay bạn.
- **Tự nuôi nhau.** Ai bỏ cuộc thì phần chưa-kiếm-được quay về "hồ chung" nuôi người mới. Ai kiên trì thì được thưởng đèn thật.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Tạo ví bằng vân tay
Bạn chạm vân tay. Điện thoại tự sinh khoá bảo mật trong Secure Enclave (con chip an toàn của máy). Ví Phoenix ra đời. Không mật khẩu, không giấy 24-từ bắt buộc.

### Bước 2 — Bấm "Nhận LAMP" (GetLAMP)
Một nút duy nhất. Bấm xong, bạn nhận **D chiếc LAMP vào ví của mình** (túi LAMP có khoá tạm). D nhiều hay ít tuỳ hồ chung lúc đó, **tối đa 1001**.

Màn hình báo: *"Nhận D LAMP vào ví của bạn — dùng dịch vụ mỗi ngày để giữ. Từ ngày 1002, mỗi Kỳ (5 ngày) dùng đủ thì tối đa 5 LAMP dần thành CỦA BẠN (rút/bán được), tới khi hết phần còn lại."*

> **"Mượn" chỉ là cách nói.** Đèn nằm trong ví bạn ngay từ đầu. Chỉ là **có điều kiện** — chưa hẳn của bạn cho tới khi bạn kiếm được qua việc dùng thật.

### Bước 3 — Đèn sinh điều ước, bạn dùng dịch vụ
Mỗi ngày túi LAMP tự sinh ra **MAGIC** (điều ước). Bạn tiêu MAGIC để dùng các dịch vụ trong hệ sinh thái. Không phải bấm gì để "bật đèn" — nó tự chạy nền.

### Bước 4 — Xoa đèn đều đặn để giữ
Ngày nào bạn **không dùng dịch vụ đủ**, cuối ngày **1 LAMP quay về hồ chung**. Đây là phần bạn **chưa kiếm được**, không phải tài sản bị tịch thu. Dùng lại → ngừng mất ngay.

### Bước 5 — Từ ngày 1002: đèn bắt đầu thành của bạn DẦN từng Kỳ
Từ ngày 1002, hệ chuyển nhịp từ NGÀY sang **KỲ** (1 Kỳ = 5 ngày). Bạn **KHÔNG** nhận trọn một lần — mà **sở hữu DẦN**: **mỗi Kỳ bạn còn dùng dịch vụ đủ**, tối đa **5 chiếc LAMP tự động rơi thẳng vào ví Phoenix của bạn** thành SỞ HỮU thật (5 chiếc = 1 chiếc/ngày × 5 ngày). Không cần bấm "rút" — LAMP tới ví ngay, bán/giữ/tiêu tuỳ ý. Cứ thế từng Kỳ cho tới khi hết phần còn lại. Phần đã vào ví là **của bạn tuyệt đối**, không gì đụng tới được nữa.

### (Tuỳ chọn) Mua thêm điều ước — GetMAGIC
Muốn tiêu nhiều hơn dòng đèn tự sinh? Bạn có thể **mua CARP bằng tiền thật** (VietQR/thẻ) để tiêu dần. Đây là lựa chọn, không bắt buộc.

---

## 4. Hai pha — kể bằng ẩn dụ chiếc đèn

### PHA-1 · "Giai đoạn Ngày" — Đang kiếm (ngày 1 → 1001)

Chiếc đèn **ở trong ví bạn nhưng còn khoá tạm**. Nó vẫn sáng — mỗi ngày sinh điều ước cho bạn dùng. Nhưng đèn **chưa hẳn của bạn**: bạn chưa rút/bán được.

- **Bạn hưởng:** dòng MAGIC hằng ngày để dùng dịch vụ.
- **Luật giữ đèn (nhịp NGÀY):** ngày nào không xoa đèn (không tiêu đủ mức qua **dịch vụ trả phí**) → **1 LAMP rơi về hồ chung**. Đèn nhỏ đi thì điều ước hằng ngày cũng ít đi theo.
- **Có 7 ngày làm quen** đầu tiên, chưa mất gì.

*Ví von:* đèn đang "thử việc" với bạn. Chăm dùng thì đèn ở lại; bỏ bê thì đèn hao dần về hồ chung cho người khác.

### PHA-2 · "Giai đoạn Kỳ" — Đang sở hữu (từ ngày 1002; 1 Kỳ = 5 ngày = 432.000 slot)

Bạn đã qua đủ 1001 ngày cam kết. Giờ nhịp chuyển từ NGÀY sang **KỲ** (1 Kỳ = 5 ngày). Mỗi Kỳ xử lý **đúng một lần**:

- **Kỳ bạn còn dùng đủ (active):** tối đa **5 chiếc LAMP tự động vào ví Phoenix của bạn** thành của bạn thật sự (5 = 1 chiếc/ngày × 5 ngày). Không cần thao tác — LAMP tới ví ngay, rút/bán/giữ tuỳ ý.
- **Kỳ bạn nghỉ (idle):** tối đa **5 chiếc LAMP còn-khoá quay về hồ chung** ngay Kỳ đó (phần chưa-kiếm-được, nuôi người mới).
- Phần đã vào ví bạn thì **an toàn tuyệt đối** — nằm hẳn trong ví của bạn, không cơ chế nào đụng tới, kể cả khi bạn nghỉ hẳn.

*Ví von:* đèn đã "vào biên chế". Mỗi Kỳ bạn ghé dùng thì một mảnh đèn rơi thẳng vào tay bạn; Kỳ bạn vắng thì mảnh chưa-kiếm đó trả về hồ chung. Phần đã cầm trong tay thì giữ mãi.

> **Khác bản trước:** LAMP đã-kiếm nay **vào thẳng ví bạn mỗi Kỳ active** (không còn "tháo khoá trong đèn rồi bấm [Rút]"), và phần chưa-kiếm trả về hồ chung **ngay từng Kỳ nghỉ** (không còn chờ rất lâu). Đơn giản và nhanh hơn cho bạn.

---

## 5. Các cải tiến — bảng cũ vs mới

| Điểm | Bản cũ | Bản mới (v5) |
|---|---|---|
| **Khởi tạo** | Nạp 200.000đ đổi LAMP + ADA | **Miễn phí** — bấm một nút |
| **LAMP thuộc về ai** | Mượn, luôn "thuộc hồ chung", không bao giờ của bạn | **Vào ví bạn ngay** (còn khoá); từ ngày 1002 phần sống sót **thành của bạn dần từng Kỳ** |
| **Phần thưởng trung thành** | Không có — dùng mãi vẫn chỉ hưởng dòng, không sở hữu | **Có** — kiên trì → mỗi Kỳ (từ ngày 1002) sở hữu dần tối đa 5 LAMP thật (rút/bán được) |
| **Nhận LAMP đã-kiếm** | Không có đường rút | **Tự động vào ví** mỗi Kỳ active ở PHA-2 (tối đa 5 LAMP/Kỳ, không cần bấm rút) |
| **Nếu bỏ cuộc** | Trả toàn bộ về hồ chung | Chỉ phần **chưa kiếm được** về hồ (từng Kỳ nghỉ); phần **đã vào ví giữ nguyên** |
| **Phí mạng (ADA)** | Bạn phải lo | **Feecover lo thay** — bạn không thấy phí ADA |
| **Tự tạo app tiêu điều ước của mình** | Bị nghi ngờ / chặn | **Hợp lệ + khuyến khích** — miễn app đăng ký đúng chuẩn (xem §7) |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Túi LAMP miễn phí vào ví (tối đa 1001 chiếc).
- Dòng MAGIC hằng ngày để dùng dịch vụ — cả hai pha.
- Từ ngày 1002, mỗi Kỳ dùng thật: **sở hữu dần phần LAMP sống sót** (tối đa 5 LAMP/Kỳ vào ví), rút/bán/giữ tuỳ ý.
- Không phải lo phí mạng, không phải hiểu tỷ giá.

**Bạn cần:**
- **Dùng dịch vụ trả phí đều đặn.** PHA-1 (Giai đoạn Ngày): mỗi ngày. PHA-2 (Giai đoạn Kỳ): mỗi Kỳ (5 ngày). Không dùng thì đèn hao dần (phần chưa-kiếm).
- Hiểu rõ **hai loại LAMP**:
  - **LAMP điều-kiện** (còn khoá trong đèn, đang sinh điều ước) — chưa của bạn, **không rút được**.
  - **LAMP đã-của-bạn** (đã vào thẳng ví Phoenix mỗi Kỳ active) — của bạn thật, nằm trong ví bạn, **rút/bán/tiêu tuỳ ý**.

> Một điều công bằng (ý-định thiết-kế): **một người = một suất.** Suất đèn tính theo danh tính sinh-trắc (vân tay) của bạn, không theo số ví — tạo nhiều ví không nhân suất. *Lưu ý kỹ-thuật: lớp neo-danh-tính-trên-chuỗi (anchor) cho DID cá-nhân đang hoàn thiện (xem §8) — nên việc mở GetLAMP rộng-rãi cho cá-nhân tạm hoãn tới khi xong. DID doanh-nghiệp/tổ-chức không dính vì có chữ-ký-cha xác thực.*

---

## 7. Câu hỏi thường gặp

**Hỏi: "Mượn" nghĩa là tôi phải trả lại?**
Đáp: Không phải trả tiền. Đèn nằm trong ví bạn ngay. "Có điều kiện" nghĩa là: dùng đều đặn thì giữ và cuối cùng sở hữu; bỏ bê thì phần chưa-kiếm hao về hồ chung. Không có nợ, không có lãi.

**Hỏi: Tôi mất gì nếu quên dùng vài ngày?**
Đáp: PHA-1 (Giai đoạn Ngày) — mỗi ngày quên mất 1 LAMP (phần chưa-kiếm), dùng lại là ngừng mất. Có 7 ngày đầu miễn trừ. PHA-2 (Giai đoạn Kỳ) — mỗi Kỳ (5 ngày) bạn nghỉ, tối đa 5 LAMP còn-khoá quay về hồ chung Kỳ đó (phần chưa-kiếm); phần đã vào ví bạn thì an toàn tuyệt đối dù bạn nghỉ bao lâu.

**Hỏi: Bao giờ tôi thật sự sở hữu LAMP?**
Đáp: Từ ngày 1002 trở đi, mỗi Kỳ (5 ngày) bạn còn dùng dịch vụ đủ → tối đa 5 chiếc vào thẳng ví Phoenix của bạn (5 = 1 chiếc/ngày × 5 ngày). Kiên trì đủ và không bị hao ngày nào ở PHA-1 → có thể sở hữu tối đa 1001 chiếc.

**Hỏi: MAGIC (điều ước) là gì? Khác LAMP thế nào?**
Đáp: **LAMP** là tài sản nền, tổng cung cố định 36 tỷ (không đốt). **MAGIC** là "quyền dùng dịch vụ" gắn với danh tính bạn — là số dư trong ví (account), **không chuyển cho người khác được**, và **tan biến nếu không dùng** (dùng-hay-mất). Đèn (LAMP) sinh ra điều ước (MAGIC) để bạn tiêu; khi thanh toán dịch vụ, hệ trả bằng CARP (1 CARP = 1 MAGIC).

**Hỏi: Chiếc đèn có bị "đốt" khi sinh điều ước không?**
Đáp: Không. Đèn **đứng yên** trong ví. Hệ thống chỉ **đọc số dư đèn** để tính bao nhiêu điều ước — không hề đốt, không chuyển LAMP đi đâu. Đèn to thì điều ước nhiều; đèn hao thì điều ước ít theo.

**Hỏi: Tôi tự làm app rồi tiêu điều ước của chính mình có được không?**
Đáp: **Được, và được khuyến khích** — miễn app của bạn đăng ký qua Registry (danh sách các dịch vụ đã được duyệt là "tiêu tài nguyên thật", do đội vận hành PhoenixKey xét duyệt trước khi cho phép tính vào anti-idle/vest-gate — không phải một trang tự-đăng-ký công khai) và thật sự tiêu tài nguyên thật (lưu trữ, băng thông, tính toán, sức lao động…). Mỗi lượt tiêu đều tốn phí về ngân quỹ chung nên hệ thống có lợi. Chỉ "tiêu rỗng" không tài nguyên mới bị loại — ngay ở khâu duyệt đăng ký.

**Hỏi: Tôi có phải trả phí mạng (ADA) không?**
Đáp: Không. Feecover lo phí mạng thay bạn.

**Hỏi: Tôi muốn dừng giữa chừng thì sao?**
Đáp: PHA-1 không có nút "từ bỏ" chủ động — bạn chỉ cần ngừng dùng, cơ chế anti-idle tự xử lý: mỗi ngày không dùng đủ, hệ thống tự thu 1 LAMP (phần chưa-kiếm) về hồ chung, không cần bạn bấm gì. PHA-2 phần đã-của-mình đã nằm sẵn trong ví bạn (tiêu/bán tuỳ ý); phần chưa-kiếm-được, mỗi Kỳ bạn nghỉ hệ thống tự thu tối đa 5 LAMP về hồ chung, bạn không cần làm gì.

---

## 8. Ranh giới thiết kế — luật cho các phần phụ thuộc đội khác

Vài phần của Wakeme phụ thuộc thiết-kế/hạ-tầng ở đội khác. Luật áp dụng khi các phần đó nối vào:

- **Validator lõi (khoá/vest/rút trên chuỗi)** là nguồn chân-lý duy nhất cho sổ-sách LAMP: mọi app/backend PHẢI đọc-ghi qua đúng redeemer của validator, không được tự suy luận số dư.
- **Điều ước (MAGIC) sinh ra sao:** nguyên lý cố định là "đèn đứng yên, chỉ đọc số dư, không đúc thêm token" — engine Gen KHÔNG được spend/đốt LAMP dưới bất kỳ hình thức nào, chỉ đọc.
- **Chuẩn "dịch vụ tiêu tài nguyên thật" (Registry):** Registry là danh sách các dịch vụ đã qua xét-duyệt của đội vận hành PhoenixKey, xác nhận dịch vụ đó tiêu tài nguyên thật (lưu trữ/băng thông/tính toán/lao động) chứ không phải giao dịch giả tạo dựng để "cày" phần thưởng. Chỉ dịch vụ có tên trong Registry mới được tính là "đã dùng dịch vụ" cho anti-idle/vest-gate. Dịch vụ "tiêu rỗng" không tài nguyên bị loại ngay ở khâu duyệt đăng ký.
- **Mua điều ước bằng tiền thật (GetMAGIC):** hệ **không đúc CARP tự do** — user luôn trả CARP đã có, có bảo chứng qua GreenBack (hệ thống/đối tác giữ dự trữ đối ứng cho CARP, đảm bảo mỗi CARP lưu hành có giá trị thật đứng sau, không phát hành khống).
- **Cân đối hồ chung PHA-2:** vì đèn PHA-2 rời hệ thống sang tay người dùng, hồ chung cần nguồn nạp bù chủ động (không được để hồ chung âm).
- **"Một người một suất":** D keyed theo PersonDID sinh-trắc, không theo số ví — tạo nhiều ví không nhân suất. Điều kiện tiên quyết: lớp neo-danh-tính-trên-chuỗi (anchor) cho DID cá-nhân phải ràng đúng khoá-gốc (UniquenessThread/PA2) trước khi mở GetLAMP rộng-rãi cho cá-nhân. DID doanh-nghiệp/tổ-chức không cần điều kiện này vì đã có chữ-ký-cha xác thực.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai). (đã qua rà-soát nội-bộ)
- Tài-liệu cùng bộ: [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md), [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md), [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).
