# PhoenixKey — Wakeme: Chiếc Đèn và Điều ước của bạn

> **Module:** Wakeme (kích-hoạt nhận LAMP). **Loại doc:** Feature (tiếng Việt, hướng người-dùng/sản-phẩm). **Ngày:** 2026-07-31.
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

> **Hồ chung lấy đèn ở đâu ra để nuôi mãi?** Nguồn **chủ yếu** làm hồ chung đầy lên là **thặng-dư LAMP từ Feecover** (cơ-chế lo-phí của hệ). Mỗi giao-dịch bạn tiêu điều-ước, hệ thu một khoản **phí cố-định bằng CARP** tương-ứng lượng MAGIC tiêu-thụ. Khoản CARP đó được hệ dùng để **mua lại LAMP khi giá xuống thấp** hoặc **đổi (redeem) ra LAMP từ GreenCheck khi phù-hợp** — phần LAMP thu về đó **chảy vào hồ chung**. Nói cách khác: càng nhiều người dùng dịch-vụ thật, hồ chung càng được nạp bù → càng đủ đèn cho người mới. Đây là vòng bền-vững, không phải in thêm LAMP (tổng-cung LAMP cố-định 36 tỷ, không đốt).

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Tạo ví bằng vân tay
Bạn chạm vân tay. Điện thoại tự sinh khoá bảo mật trong Secure Enclave (con chip an toàn của máy). Ví Phoenix ra đời. Không mật khẩu, không giấy 24-từ bắt buộc.

### Bước 2 — Bấm "Nhận LAMP" (Wakeme)
Một nút duy nhất. Bấm xong, bạn nhận **D chiếc LAMP vào ví của mình** (túi LAMP có khoá tạm). D nhiều hay ít tuỳ hồ chung lúc đó, **tối đa 1001**.

Màn hình báo: *"Nhận D LAMP vào ví của bạn — dùng dịch vụ mỗi ngày để giữ trọn. Sau 1001 ngày, phần còn lại thành CỦA BẠN (rút/bán được)."*

> **Đây là "quyền dùng", không phải cho vay.** Đèn nằm trong ví bạn ngay từ đầu, nhưng phần khoá-tạm này bạn mới chỉ được **cấp quyền DÙNG** nó để sinh điều-ước — **chưa SỞ HỮU**: chưa rút/bán/mang đi được, cũng không dùng vào việc khác (không đúc token riêng, không biểu-quyết). Dùng thật đủ lâu → phần sống sót mới thành SỞ HỮU của bạn. **Không nợ, không lãi, không phải trả lại.**

### Bước 3 — Đèn sinh điều ước, bạn dùng dịch vụ
Mỗi ngày túi LAMP tự sinh ra **MAGIC** (điều ước). Bạn tiêu MAGIC để dùng các dịch vụ trong hệ sinh thái. Không phải bấm gì để "bật đèn" — nó tự chạy nền.

### Bước 4 — Xoa đèn đều đặn để giữ
Ngày nào bạn **không dùng dịch vụ đủ**, cuối ngày **1 LAMP quay về hồ chung**. Đây là phần bạn **chưa kiếm được**, không phải tài sản bị tịch thu. Dùng lại → ngừng mất ngay.

### Bước 5 — Sau 1001 ngày: đèn bắt đầu thành của bạn
**Sau 1001 ngày đầu**, đèn vào giai đoạn **theo kỳ (mỗi kỳ 5 ngày)**. Kỳ nào bạn **còn dùng dịch vụ đủ** → tối đa **5 chiếc LAMP thành SỞ HỮU thật của bạn** *(5 là mức tối đa — dành cho người tham gia sớm; người vào sau nhận theo tỷ lệ phần đèn được cấp quyền dùng)*. Đèn đã sở-hữu bạn **tự chọn**: **giữ trong ví-đèn** để tiếp-tục sinh MAGIC (đèn càng giữ lâu, tuổi càng cao, sinh MAGIC càng lợi), **rút về ví** để giao-dịch, hoặc **đúc CARP**. Kỳ nào bạn **nghỉ** → phần đèn còn khoá **dồn sang kỳ sau, KHÔNG mất**.

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

### PHA-2 · "Đang sở hữu" (sau 1001 ngày đầu, tính theo KỲ 5 ngày)

Bạn đã qua đủ 1001 ngày cam kết. Giờ đèn nhả theo **kỳ (mỗi kỳ 5 ngày)**:

- **Kỳ bạn dùng dịch vụ đủ (tiêu đủ mức tối thiểu):** tối đa **5 chiếc LAMP thành của bạn thật sự** *(5 là mức tối đa cho người tham gia sớm; người sau nhận theo tỷ lệ phần đèn được cấp quyền dùng)*. Đèn đã sở-hữu bạn **tự chọn**: **giữ trong ví-đèn** để tiếp-tục sinh MAGIC (giữ càng lâu, tuổi đèn càng cao, sinh MAGIC càng lợi), **rút về ví** giao-dịch, hoặc **đúc CARP**.
- **Kỳ bạn nghỉ:** phần đèn còn khoá **DỒN sang kỳ sau, KHÔNG mất gì** — cứ ghé lại là nhả tiếp.
- Phần LAMP **đã sở-hữu** thì **an toàn tuyệt đối** — không bao giờ bị đụng, kể cả khi bạn nghỉ hẳn (dù bạn để nó trong ví-đèn hay đã rút ra).
- Chỉ khi bạn **bỏ bê rất lâu — 1001 kỳ LIÊN TỤC không dùng gì** — thì phần đèn **còn khoá** (chưa kiếm được) mới quay về hồ chung.

*Ví von:* đèn đã "vào biên chế". Mỗi kỳ ghé dùng thì được cầm 5 mảnh về tay; nghỉ kỳ nào thì mảnh đó chờ bạn ở kỳ sau. Bỏ hẳn thật lâu thì phần chưa-cầm mới trả lại — phần đã cầm trong tay thì giữ mãi.

---

## 5. Các cải tiến — bảng cũ vs mới

| Điểm | Bản cũ | Bản mới (A) |
|---|---|---|
| **Khởi tạo** | Nạp 200.000đ đổi LAMP + ADA | **Miễn phí** — bấm một nút |
| **LAMP thuộc về ai** | Mượn, luôn "thuộc hồ chung", không bao giờ của bạn | Khoá-tạm trong ví-đèn của bạn; **sau 1001 ngày đầu**, mỗi kỳ dùng đủ → tối đa **5 chiếc thành SỞ HỮU thật** (mức tối đa cho người sớm) |
| **Phần thưởng trung thành** | Không có — dùng mãi vẫn chỉ hưởng dòng, không sở hữu | **Có** — kiên trì đủ 1001 ngày → sở hữu đèn thật |
| **Đèn đã sở-hữu dùng được gì** | Không có đường rút | **Giữ trong ví-đèn** (sinh MAGIC, tuổi cao càng lợi) / **rút về ví** giao-dịch / **đúc CARP** — bạn tự chọn |
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

> Một điều công bằng (mục-tiêu thiết-kế): **mỗi người chỉ một suất.** Đây là hướng đang xây, **chưa đạt đủ ở tầng hiện tại** — sinh-trắc trên máy chỉ đảm bảo một-suất-mỗi-máy, chưa chặn được một người dùng nhiều máy. Để đạt "một người một suất" thật, hệ dùng nhiều lớp: xác-thực hai-yếu-tố, kiểm-tra trùng-người bằng công-nghệ bảo-mật (Glint) và chứng-từ (Spectra), và về sau là nhận-diện đa-đặc-điểm trên một video selfie. *Trong lúc các lớp này hoàn thiện, việc mở Wakeme rộng-rãi cho cá-nhân tạm hoãn. DID doanh-nghiệp/tổ-chức không dính vì đã có chữ-ký-cha xác thực (mỗi tổ-chức một OrgDID).*

---

## 7. Câu hỏi thường gặp

**Hỏi: Đèn khoá-tạm là "cho vay"? Tôi phải trả lại?**
Đáp: Không. Đây là **quyền DÙNG**, không phải khoản vay. Đèn nằm trong ví bạn ngay. Phần khoá-tạm bạn được **cấp quyền dùng** để sinh điều-ước — nhưng **chưa phải của bạn**: không rút/bán/mang đi được, và **không dùng vào việc khác** (không đúc token riêng, không biểu-quyết). Dùng đều đặn thì giữ và cuối cùng **sở hữu** phần sống sót; bỏ bê thì phần chưa-kiếm hao về hồ chung. **Không nợ, không lãi, không phải trả lại.**

**Hỏi: Tôi mất gì nếu quên dùng vài ngày?**
Đáp: PHA-1 — mỗi ngày quên mất 1 LAMP (phần chưa-kiếm), dùng lại là ngừng mất. Có 7 ngày đầu miễn trừ. PHA-2 — quên một kỳ chỉ làm việc nhả đèn tạm dừng, đèn không mất; phần đã sở hữu thì an toàn tuyệt đối dù bạn nghỉ bao lâu.

**Hỏi: Bao giờ tôi thật sự sở hữu LAMP?**
Đáp: **Sau 1001 ngày đầu**, tính theo **kỳ (5 ngày)**: kỳ nào bạn dùng dịch vụ đủ → tối đa 5 chiếc thành **của bạn thật sự** (giữ trong ví-đèn để sinh MAGIC, rút về ví, hoặc đúc CARP — tuỳ bạn). Kỳ nghỉ thì phần còn khoá dồn sang kỳ sau, không mất. Kiên trì đủ và không bị hao ngày nào ở PHA-1 → có thể sở hữu tối đa 1001 chiếc *(mức trần cho người tham gia sớm; người vào sau nhận theo tỷ lệ phần đèn được cấp quyền dùng)*.

**Hỏi: MAGIC (điều ước) là gì? Khác LAMP thế nào?**
Đáp: **LAMP** là tài sản nền, tổng cung cố định 36 tỷ (không đốt). **MAGIC** là "quyền dùng dịch vụ" gắn với danh tính bạn — là số dư trong ví (account), **không chuyển cho người khác được**, và **tan biến nếu không dùng** (dùng-hay-mất). Đèn (LAMP) sinh ra điều ước (MAGIC) để bạn tiêu; khi thanh toán dịch vụ, hệ trả bằng CARP (1 CARP = 1 MAGIC).

**Hỏi: Chiếc đèn có bị "đốt" khi sinh điều ước không?**
Đáp: Không. Đèn **đứng yên** trong ví. Hệ thống chỉ **đọc số dư đèn** để tính bao nhiêu điều ước — không hề đốt, không chuyển LAMP đi đâu. Đèn to thì điều ước nhiều; đèn hao thì điều ước ít theo.

**Hỏi: Tôi tự làm app rồi tiêu điều ước của chính mình có được không?**
Đáp: **Được, và được khuyến khích** — miễn app của bạn đăng ký qua Registry (danh sách các dịch vụ đã được duyệt là "tiêu tài nguyên thật", do đội vận hành PhoenixKey xét duyệt trước khi cho phép tính vào anti-idle/vest-gate — không phải một trang tự-đăng-ký công khai) và thật sự tiêu tài nguyên thật (lưu trữ, băng thông, tính toán, sức lao động…). Mỗi lượt tiêu đều tốn phí về ngân quỹ chung nên hệ thống có lợi. Chỉ "tiêu rỗng" không tài nguyên mới bị loại — ngay ở khâu duyệt đăng ký.

**Hỏi: Tôi có phải trả phí mạng (ADA) không?**
Đáp: Không. Feecover lo phí mạng thay bạn.

**Hỏi: Tôi muốn dừng giữa chừng thì sao?**
Đáp: PHA-1 không có nút "từ bỏ" chủ động — bạn chỉ cần ngừng dùng, cơ chế anti-idle tự xử lý: mỗi ngày không dùng đủ, hệ thống tự thu 1 LAMP (phần chưa-kiếm) về hồ chung, không cần bạn bấm gì. PHA-2 bạn rút phần đã-của-mình ra bất cứ lúc nào; phần chưa tháo khoá cứ để đó, hệ thống tự xử lý.

---

## 8. Ranh giới thiết kế — luật cho các phần phụ thuộc đội khác

Vài phần của Wakeme phụ thuộc thiết-kế/hạ-tầng ở đội khác. Luật áp dụng khi các phần đó nối vào:

- **Validator lõi (khoá/vest/rút trên chuỗi)** là nguồn chân-lý duy nhất cho sổ-sách LAMP: mọi app/backend PHẢI đọc-ghi qua đúng redeemer của validator, không được tự suy luận số dư.
- **Điều ước (MAGIC) sinh ra sao:** nguyên lý cố định là "đèn đứng yên, chỉ đọc số dư, không đúc thêm token" — engine Gen KHÔNG được spend/đốt LAMP dưới bất kỳ hình thức nào, chỉ đọc.
- **Chuẩn "dịch vụ tiêu tài nguyên thật" (Registry):** Registry là danh sách các dịch vụ đã qua xét-duyệt của đội vận hành PhoenixKey, xác nhận dịch vụ đó tiêu tài nguyên thật (lưu trữ/băng thông/tính toán/lao động) chứ không phải giao dịch giả tạo dựng để "cày" phần thưởng. Chỉ dịch vụ có tên trong Registry mới được tính là "đã dùng dịch vụ" cho anti-idle/vest-gate. Dịch vụ "tiêu rỗng" không tài nguyên bị loại ngay ở khâu duyệt đăng ký.
- **Mua điều ước bằng tiền thật (GetMAGIC):** hệ **không đúc CARP tự do** — user luôn trả CARP đã có, có bảo chứng qua GreenBack (hệ thống/đối tác giữ dự trữ đối ứng cho CARP, đảm bảo mỗi CARP lưu hành có giá trị thật đứng sau, không phát hành khống).
- **Cân đối hồ chung PHA-2:** vì đèn PHA-2 rời hệ thống sang tay người dùng, hồ chung cần nguồn nạp bù chủ động (không được để hồ chung âm). **Nguồn nạp CHỦ YẾU = thặng-dư LAMP từ Feecover:** hệ thu **phí cố-định bằng CARP** tương-ứng lượng MAGIC tiêu-thụ mỗi giao-dịch → **mua lại LAMP khi giá thấp** hoặc **redeem LAMP từ GreenCheck khi phù-hợp** → phần LAMP thu về nạp vào hồ chung. Cơ-chế mua-lại/redeem nằm ở tầng **Feecover/LAMP** (Wakeme chỉ là bên NHẬN nguồn); toán cân-đối tốc-độ-nạp vs tốc-độ-đèn-rời-hệ ở [Math](./PhoenixKey-Wakeme-Math.md).
- **"Một người một suất" (1 PersonDID/người, 1 OrgDID/tổ-chức):** D keyed theo PersonDID, không theo số ví — đa-ví/đa-DID không nhân suất. **Sinh-trắc-per-thiết-bị KHÔNG đủ** (chỉ 1-DID/máy). Uniqueness person-level thật cần nhiều lớp: 2FA + Glint (ZK-uniqueness, VeData) + Spectra (chứng-từ, LampNet) + MCR (nhận-diện đa-đặc-điểm trên 1 video selfie, thuật-toán tất-định, OriLife). Điều-kiện CẦN đã có phần: PA-2 UniquenessThread (SMT non-membership, đóng đúc-anchor-thứ-hai). Wakeme-PersonDID production chặn tới khi các lớp person-level land. DID doanh-nghiệp/tổ-chức không cần vì đã có chữ-ký-cha xác thực.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#wakeme)

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai). (đã qua rà-soát nội-bộ)
- Tài-liệu cùng bộ: [PhoenixKey-Wakeme-Math.md](./PhoenixKey-Wakeme-Math.md), [PhoenixKey-Wakeme-Tech.md](./PhoenixKey-Wakeme-Tech.md), [PhoenixKey-Wakeme-Exec.md](./PhoenixKey-Wakeme-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
