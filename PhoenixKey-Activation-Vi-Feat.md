# PhoenixKey — Activation: Chiếc Đèn và Điều ước của bạn

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản-phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **Activation là gì**, **được lợi gì**, **làm sao giữ**, bằng ngôn ngữ đời thường.
> **Bản v4.1 (2026-07-07).** Nội dung bám đặc-tả kỹ-thuật `PhoenixKey-Activation-Feat-Math.md` (v4.1) và `PhoenixKey-Activation-API-for-SuperApp.md`. Phần nào **chưa chạy** được đánh dấu rõ.

---

## 1. Một câu là gì

**Activation là bước "thắp đèn": bạn tạo ví bằng vân tay, bấm một nút, và nhận về một túi LAMP vào ví của chính mình. Túi LAMP đó mỗi ngày sinh ra "điều ước" (MAGIC) để bạn dùng dịch vụ — và nếu bạn dùng đều đặn đủ lâu, phần LAMP sống sót sẽ thật sự thành CỦA BẠN.**

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

**Activation mới bỏ hết ba cái đó:**
- **Miễn phí khởi tạo.** Không nạp đồng nào. Bấm một nút là có đèn.
- **Không cần hiểu tỷ giá.** Bạn chỉ thấy một nút và dòng điều ước hằng ngày. Phí mạng do hệ thống (Feecover) lo thay bạn.
- **Tự nuôi nhau.** Ai bỏ cuộc thì phần chưa-kiếm-được quay về "hồ chung" nuôi người mới. Ai kiên trì thì được thưởng đèn thật.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Tạo ví bằng vân tay
Bạn chạm vân tay. Điện thoại tự sinh khoá bảo mật trong Secure Enclave (con chip an toàn của máy). Ví Phoenix ra đời. Không mật khẩu, không giấy 24-từ bắt buộc.

### Bước 2 — Bấm "Nhận LAMP" (GetLAMP)
Một nút duy nhất. Bấm xong, bạn nhận **D chiếc LAMP vào ví của mình** (túi LAMP có khoá tạm). D nhiều hay ít tuỳ hồ chung lúc đó, **tối đa 1001**.

Màn hình báo: *"Nhận D LAMP vào ví của bạn — dùng dịch vụ mỗi ngày để giữ trọn. Sau 1001 ngày, phần còn lại thành CỦA BẠN (rút/bán được)."*

> **"Mượn" chỉ là cách nói.** Đèn nằm trong ví bạn ngay từ đầu. Chỉ là **có điều kiện** — chưa hẳn của bạn cho tới khi bạn kiếm được qua việc dùng thật.

### Bước 3 — Đèn sinh điều ước, bạn dùng dịch vụ
Mỗi ngày túi LAMP tự sinh ra **MAGIC** (điều ước). Bạn tiêu MAGIC để dùng các dịch vụ trong hệ sinh thái. Không phải bấm gì để "bật đèn" — nó tự chạy nền.

### Bước 4 — Xoa đèn đều đặn để giữ
Ngày nào bạn **không dùng dịch vụ đủ**, cuối ngày **1 LAMP quay về hồ chung**. Đây là phần bạn **chưa kiếm được**, không phải tài sản bị tịch thu. Dùng lại → ngừng mất ngay.

### Bước 5 — Sau 1001 ngày: đèn bắt đầu thành của bạn
Từ ngày 1002, phần LAMP còn sống sót **mỗi ngày nhả 1 chiếc thành SỞ HỮU thật của bạn** — rút ra ví ngoài, bán, hoặc giữ lại cho sinh tiếp điều ước. Nút **[Rút LAMP]** xuất hiện.

### (Tuỳ chọn) Mua thêm điều ước — GetMAGIC
Muốn tiêu nhiều hơn dòng đèn tự sinh? Bạn có thể **mua CARP bằng tiền thật** (VietQR/thẻ) để tiêu dần. Đây là lựa chọn, không bắt buộc.

---

## 4. Hai pha — kể bằng ẩn dụ chiếc đèn

### PHA-1 · "Đang kiếm" (ngày 1 → 1001)

Chiếc đèn **ở trong ví bạn nhưng còn khoá tạm**. Nó vẫn sáng — mỗi ngày sinh điều ước cho bạn dùng. Nhưng đèn **chưa hẳn của bạn**: bạn chưa rút/bán được.

- **Bạn hưởng:** dòng MAGIC hằng ngày để dùng dịch vụ.
- **Luật giữ đèn:** ngày nào không xoa đèn (không dùng dịch vụ đủ) → **1 LAMP rơi về hồ chung**. Đèn nhỏ đi thì điều ước hằng ngày cũng ít đi theo.
- **Có 7 ngày làm quen** đầu tiên, chưa mất gì.

*Ví von:* đèn đang "thử việc" với bạn. Chăm dùng thì đèn ở lại; bỏ bê thì đèn hao dần về hồ chung cho người khác.

### PHA-2 · "Đang sở hữu" (từ ngày 1002)

Bạn đã qua đủ 1001 ngày cam kết. Giờ **mỗi ngày 1 chiếc LAMP tháo khoá, thành của bạn thật sự** — rút, bán, hay giữ tuỳ ý.

Nhưng vẫn có một điều kiện nhẹ để nhả đèn: **mỗi kỳ (khoảng 5 ngày) bạn phải còn dùng dịch vụ đủ.** Kỳ nào bạn nghỉ hẳn → việc nhả đèn **tạm dừng** kỳ đó (đèn không mất, chỉ chưa tháo thêm).

- Phần đã thành của bạn (**đã tháo khoá**) thì **an toàn tuyệt đối** — không bao giờ bị đụng, kể cả khi bạn nghỉ hẳn.
- Chỉ khi bạn **bỏ bê rất lâu — 1001 kỳ liên tục không dùng gì** — thì phần đèn **còn khoá** (chưa kiếm được) mới quay về hồ chung.

*Ví von:* đèn đã "vào biên chế". Mỗi ngày một mảnh thành của bạn, miễn bạn còn ghé dùng. Bỏ hẳn thật lâu thì phần chưa-tháo mới trả lại — phần đã cầm trong tay thì giữ mãi.

---

## 5. Các cải tiến — bảng cũ vs mới

| Điểm | Bản cũ | Bản mới (v4.1) |
|---|---|---|
| **Khởi tạo** | Nạp 200.000đ đổi LAMP + ADA | **Miễn phí** — bấm một nút |
| **LAMP thuộc về ai** | Mượn, luôn "thuộc hồ chung", không bao giờ của bạn | **Vào ví bạn ngay**; sau 1001 ngày phần sống sót **thành của bạn** |
| **Phần thưởng trung thành** | Không có — dùng mãi vẫn chỉ hưởng dòng, không sở hữu | **Có** — kiên trì đủ 1001 ngày → sở hữu đèn thật (rút/bán được) |
| **Rút LAMP** | Không có đường rút | **Rút được** ở PHA-2 (nút [Rút LAMP]) |
| **Nếu bỏ cuộc** | Trả toàn bộ về hồ chung | Chỉ phần **chưa kiếm được** về hồ; phần **đã sở hữu giữ nguyên** |
| **Phí mạng (ADA)** | Bạn phải lo | **Feecover lo thay** — bạn không thấy phí ADA |
| **Tự tạo app tiêu điều ước của mình** | Bị nghi ngờ / chặn | **Hợp lệ + khuyến khích** — miễn app đăng ký đúng chuẩn (xem §7) |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Túi LAMP miễn phí vào ví (tối đa 1001 chiếc).
- Dòng MAGIC hằng ngày để dùng dịch vụ — cả hai pha.
- Sau 1001 ngày dùng thật: **sở hữu phần LAMP sống sót**, rút/bán/giữ tuỳ ý.
- Không phải lo phí mạng, không phải hiểu tỷ giá.

**Bạn cần:**
- **Dùng dịch vụ đều đặn.** PHA-1: mỗi ngày. PHA-2: mỗi kỳ (~5 ngày). Không dùng thì đèn hao dần (phần chưa-kiếm).
- Hiểu rõ **hai loại đèn** trên màn hình:
  - **LAMP điều-kiện** (khoá, đang sinh điều ước) — chưa của bạn, **không rút được**.
  - **LAMP đã-của-bạn** (đã tháo khoá) — của bạn thật, **rút/bán được**.

> Một điều công bằng (ý-định thiết-kế): **một người = một suất.** Suất đèn tính theo danh tính sinh-trắc (vân tay) của bạn, không theo số ví — tạo nhiều ví không nhân suất. *Lưu ý: lớp bảo-đảm-duy-nhất trên chuỗi cho DID cá-nhân đang hoàn thiện (xem §8) — nên GetLAMP rộng-rãi cho cá-nhân tạm hoãn tới khi xong.*

---

## 7. Câu hỏi thường gặp

**Hỏi: "Mượn" nghĩa là tôi phải trả lại?**
Đáp: Không phải trả tiền. Đèn nằm trong ví bạn ngay. "Có điều kiện" nghĩa là: dùng đều đặn thì giữ và cuối cùng sở hữu; bỏ bê thì phần chưa-kiếm hao về hồ chung. Không có nợ, không có lãi.

**Hỏi: Tôi mất gì nếu quên dùng vài ngày?**
Đáp: PHA-1 — mỗi ngày quên mất 1 LAMP (phần chưa-kiếm), dùng lại là ngừng mất. Có 7 ngày đầu miễn trừ. PHA-2 — quên một kỳ chỉ làm việc nhả đèn tạm dừng, đèn không mất; phần đã sở hữu thì an toàn tuyệt đối dù bạn nghỉ bao lâu.

**Hỏi: Bao giờ tôi thật sự sở hữu LAMP?**
Đáp: Từ ngày 1002 trở đi, mỗi ngày 1 chiếc thành của bạn (miễn kỳ đó bạn còn dùng dịch vụ). Kiên trì đủ và không bị hao ngày nào ở PHA-1 → có thể sở hữu tối đa 1001 chiếc.

**Hỏi: MAGIC (điều ước) là gì? Khác LAMP thế nào?**
Đáp: **LAMP** là tài sản nền, tổng cung cố định 36 tỷ. **MAGIC** là "quyền dùng dịch vụ" gắn với danh tính bạn — không chuyển cho người khác được, và **tan biến nếu không dùng** (dùng-hay-mất). Đèn (LAMP) sinh ra điều ước (MAGIC) để bạn tiêu.

**Hỏi: Chiếc đèn có bị "đốt" khi sinh điều ước không?**
Đáp: Không. Đèn **đứng yên** trong ví. Hệ thống chỉ **đọc số dư đèn** để tính bao nhiêu điều ước — không hề đốt, không chuyển LAMP đi đâu. Đèn to thì điều ước nhiều; đèn hao thì điều ước ít theo.

**Hỏi: Tôi tự làm app rồi tiêu điều ước của chính mình có được không?**
Đáp: **Được, và được khuyến khích** — miễn app của bạn đăng ký qua Registry và thật sự tiêu tài nguyên thật (lưu trữ, băng thông, tính toán, sức lao động…). Mỗi lượt tiêu đều tốn phí về ngân quỹ chung nên hệ thống có lợi. Chỉ "tiêu rỗng" không tài nguyên mới bị loại — ngay ở khâu duyệt đăng ký.

**Hỏi: Tôi có phải trả phí mạng (ADA) không?**
Đáp: Không. Feecover lo phí mạng thay bạn.

**Hỏi: Tôi muốn dừng giữa chừng thì sao?**
Đáp: PHA-1 bạn có thể "từ bỏ" — phần đèn còn lại về hồ chung (vì chưa kiếm được). PHA-2 bạn rút phần đã-của-mình ra bất cứ lúc nào; phần chưa tháo khoá cứ để đó, hệ thống tự xử lý.

---

## 8. Ranh giới trung thực — cái gì chưa chạy

Để không hứa hão, đây là trạng thái thật:

- **Toàn bộ Activation mới đang là đặc-tả (spec), CHƯA có nút nào chạy thật.** Đội sản-phẩm đang dựng giao diện trước; nối hệ thống thật sau.
- **Điều ước (MAGIC) sinh ra sao — đang phát triển.** Nguyên lý "đèn đứng yên, chỉ đọc số dư" đã chốt, nhưng cơ chế kỹ thuật cụ thể còn chờ đội MAGIC/CARP hoàn thiện.
- **Chuẩn "dịch vụ tiêu tài nguyên thật" (Registry) — đang phát triển.** Danh mục loại dịch vụ hợp lệ và cổng duyệt chưa có. Trước khi có, cảnh báo "hôm nay chưa dùng dịch vụ" tạm chưa bật.
- **Mua điều ước bằng tiền thật (GetMAGIC) — đang phát triển.** Cần cổng thanh toán và lớp bảo chứng GreenBack hoàn thiện.
- **Cân đối hồ chung khi nhiều người qua PHA-2 — đang theo dõi.** Vì đèn PHA-2 rời hệ thống sang tay người dùng, hồ chung cần nguồn nạp bù. Đây là điểm đội vận hành đang tính toán.
- **"Một người một suất" (chống tạo-nhiều-suất-giả) — đang phát triển.** Ý-định thiết-kế là mỗi người sinh-trắc chỉ một suất, nhưng lớp bảo-đảm-duy-nhất trên chuỗi (UniquenessThread) **chưa hoàn thiện** cho DID cá-nhân. Trước khi nó land, việc mở GetLAMP rộng-rãi cho DID cá-nhân tạm hoãn (DID doanh-nghiệp/tổ-chức không dính vì có chữ-ký-cha xác thực).

Những gì **đã chắc**: mô hình 2 pha, đèn vào ví bạn, phần thưởng sở hữu sau 1001 ngày, miễn phí khởi tạo, Feecover lo phí mạng.
