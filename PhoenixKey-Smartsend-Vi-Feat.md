# PhoenixKey — Smartsend: Nút "hoàn tác" cho giao dịch, gửi có bảo vệ

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **Smartsend là gì**, giải quyết gì, dùng ra sao, bằng ngôn ngữ đời thường.
> **Module:** Smartsend (gửi có bảo vệ). **Loại doc:** Feature (hướng người dùng/sản phẩm). **Ngày:** 2026-07-09.
> Đặc tả toán ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md); kỹ thuật ở [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md); điều hành ở [PhoenixKey-Smartsend-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Smartsend-Specs/blob/main/PhoenixKey-Smartsend-Exec.md).
> Smartsend dùng chung hạ tầng ví (`did_payment` — validator: một đoạn hợp đồng thông minh chạy trên chuỗi, tự động kiểm luật cho mỗi giao dịch, không ai sửa tay được), guardian và anti-drain của module **Rebirthme** — xem [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md).

---

## 1. Một câu là gì

**Smartsend là cách gửi tiền có một "cửa sổ để huỷ": khoản gửi được khoá tạm trong một hòm ký quỹ; trong cửa sổ đó bạn đổi ý thì huỷ và tiền hoàn về bạn, còn với khoản lớn thì người nhận phải bấm đồng-ý mới nhận được. Gửi nhầm không còn là mất trắng.**

Hình dung đơn giản: chuyển tiền crypto bình thường như **thả thư vào thùng thư không có nắp** — buông tay là đi luôn, sai địa chỉ cũng không đòi lại được. Smartsend như **gửi bảo đảm có thời gian giữ tại bưu điện** — trong lúc giữ, bạn còn kịp rút lại nếu ghi nhầm người nhận.

---

## 2. Vấn đề nó giải

Giao dịch crypto bình thường có ba chỗ đau:

1. **Bất khả đảo tuyệt đối.** Gửi nhầm một ký tự trong địa chỉ, dán nhầm ô, bị lừa đổi địa chỉ nhận — tiền đi là mất, không có "gọi ngân hàng chặn lệnh".
2. **Không có bước xác nhận hai chiều.** Người gửi bấm là xong; người nhận không cần đồng-ý, không có cơ hội từ chối khoản đáng ngờ.
3. **Khoá lộ → lệnh gửi độc chạy ngay.** Kẻ chiếm được khoá gửi thẳng đi, nạn nhân không có cửa sổ nào để chặn.

**Smartsend lật ngược cả ba:**
- **Gửi nhầm → huỷ được** trong cửa sổ-veto, tiền hoàn nguyên về người gửi.
- **Khoản lớn → hai chiều.** Người nhận phải bấm đồng-ý (Accept) thì mới nhận; không đồng-ý, hết hạn thì tiền tự về người gửi.
- **Lộ khoá → còn cửa chặn.** Muốn HUỶ trong cửa sổ có thể đòi thêm yếu tố **khác gốc khoá gửi** (guardian / bối cảnh-ZK) → kẻ trộm chỉ có khoá gửi vẫn không lách được lớp bảo vệ.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Bật Smartsend khi gửi (tuỳ chọn)
Ở màn gửi, ngoài "Gửi thường" bạn thấy một công tắc **Gửi có bảo vệ**. Bật lên, chọn **cửa sổ huỷ** (24 / 48 / 72 giờ). Ai không bật thì gửi như thường, một chữ ký, đi ngay.

### Bước 2 — Khoản được khoá vào hòm ký quỹ
Bấm gửi: tiền KHÔNG tới thẳng người nhận mà vào một **hòm ký quỹ** (escrow) có hẹn giờ. Bạn thấy "Đang giữ — huỷ được đến [thời điểm]".

### Bước 3a — Đổi ý: Huỷ trong cửa sổ
Còn trong cửa sổ, bạn bấm **Huỷ**. Với khoản lớn hoặc khi bạn bật bảo vệ mạnh, Huỷ cần thêm một **yếu tố** (guardian ký / xác thực bối cảnh). Huỷ xong, tiền **hoàn nguyên về bạn** — đúng từng đồng.

### Bước 3b — Người nhận đồng-ý (khoản lớn)
Với khoản vượt ngưỡng lớn, người nhận thấy "Có khoản chờ bạn nhận" và bấm **Đồng-ý** (Accept). Chỉ khi đã đồng-ý, sau khi hết cửa sổ, tiền mới về được tay họ.

### Bước 4 — Hết cửa sổ: hoàn tất
Qua mốc hết cửa sổ mà không ai huỷ (và đã có đồng-ý nếu là khoản lớn), giao dịch **hoàn tất** — tiền về đúng người nhận. Bước này không cần bạn bấm; ai cũng có thể "chốt" hộ vì đích đã cố định trong hòm.

### Bước 5 — Kẹt lâu: tự hoàn về người gửi
Nếu là khoản lớn mà người nhận mãi không đồng-ý, tới một mốc **quá hạn** khoản sẽ **tự hoàn về người gửi** — không kẹt vĩnh viễn trong hòm.

### Bước 6 — Nghi bị trộm giữa chừng: đóng băng khoản
Nghi khoản gửi là do khoá lộ, guardian có thể **Freeze** hòm ký quỹ đó — treo lại, chặn hoàn tất, để xử qua đường khôi phục. Freeze chỉ dùng được trong lúc cửa sổ còn mở. Việc treo không kéo dài vô tận: nhóm guardian xử xong thì gỡ, còn nếu quá một mốc chờ mà chưa xử được thì khoản **tự hoàn về người gửi** — không ai khoá được tiền của bạn mãi mãi bằng cách treo hộ.

---

## 4. Các cơ chế — kể bằng đời thường

### 4.1 Hòm ký quỹ có hẹn giờ (escrow + cửa sổ-veto)
Khoản gửi nằm trong một hòm khoá bằng hợp đồng. Hòm có một mốc **hết cửa sổ**: TRƯỚC mốc chỉ cho **Huỷ** (về người gửi); TỪ mốc trở đi mới cho **Hoàn tất** (về người nhận). Hai giai đoạn không chồng lên nhau — không có chuyện vừa huỷ được vừa chốt được cùng lúc.

### 4.2 Huỷ đa yếu tố (khác gốc khoá gửi)
Với khoản cần bảo vệ, **Huỷ** không chỉ một chữ ký gửi mà đòi đủ số **yếu tố** đã cài trước: có thể là guardian ký, hoặc bằng chứng **bối cảnh** (khuôn mặt / ảnh bí mật / vị trí thiết bị) dạng **ZK** (Zero-Knowledge — một loại bằng chứng mật mã chứng minh một điều là đúng mà không phải tiết lộ dữ liệu gốc đứng sau nó; ví dụ chứng "đúng khuôn mặt bạn" mà không gửi luôn ảnh khuôn mặt đi). Điểm cốt tử: các yếu tố này **không sinh ra từ cùng gốc với khoá gửi** — nên một lần lộ khoá gửi KHÔNG mở được luôn quyền huỷ.

### 4.3 Khoản lớn cần người nhận đồng-ý (Accept)
Vượt một **ngưỡng lớn** do bạn đặt, chỉ gửi thôi chưa đủ: người nhận phải chủ động bấm **Đồng-ý**. Tránh dúi nhầm tiền cho người lạ, và cho người nhận quyền từ chối khoản đáng ngờ.

### 4.4 Hoàn tất "byte-perfect" — không đổi hướng
Khi chốt, hợp đồng ép trả **đúng số tiền, đúng người nhận** đã ghi trong hòm — không ai chèn đổi địa chỉ nhận, không rút bớt. Người "chốt hộ" chỉ đẩy được tiền tới đúng đích đã cố định.

### 4.5 Chống kẹt vĩnh viễn (ReclaimTimeout)
Khoản lớn mà người nhận không bao giờ đồng-ý sẽ không nằm chết trong hòm: tới mốc **quá hạn**, người gửi lấy lại được. Không tạo ra "tiền ma" kẹt trên chuỗi.

### 4.6 Đóng băng bởi guardian (Freeze), có lối thoát
Nếu nghi khoản gửi là do bị chiếm khoá, guardian **treo** hòm ký quỹ đó, chặn hoàn tất cho tới khi đường khôi phục xử lý xong. Đây là lớp nối với guardian-recovery của module Rebirthme. Để tránh một guardian đơn lẻ lợi dụng Freeze khoá tiền người khác, việc gỡ treo cần **nhóm guardian đồng thuận** (không phải một người); và nếu treo quá lâu không ai xử, khoản **tự hoàn về người gửi** ở một mốc chờ cố định — Freeze không thể biến thành khoá vĩnh viễn.

---

## 5. Bảng cải tiến (gửi thường vs Smartsend)

| Điểm | Gửi crypto thường | Smartsend (gửi có bảo vệ) |
|---|---|---|
| **Gửi nhầm địa chỉ** | Mất trắng, không đòi được | Huỷ trong cửa sổ, tiền hoàn nguyên |
| **Xác nhận người nhận** | Không có | Khoản lớn đòi người nhận Đồng-ý |
| **Khoá gửi lộ giữa chừng** | Lệnh chạy ngay | Huỷ đa yếu tố khác gốc + guardian Freeze |
| **Đổi hướng khi chốt** | — | Ép trả đúng người nhận + đúng số (byte-perfect) |
| **Kẹt trong hòm** | — | Quá hạn tự hoàn về người gửi |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Một cửa sổ để sửa sai khi gửi nhầm — thứ crypto thường không có.
- Lớp chống trộm khi khoá gửi lộ: huỷ đòi yếu tố khác gốc + guardian đóng băng.
- Quyền từ chối (bên nhận) với khoản lớn đáng ngờ.

**Bạn cần:**
- Hiểu Smartsend là **tuỳ chọn** — bật khi gửi khoản đáng lo, khoản nhỏ đời thường vẫn gửi thường cho nhanh.
- Cài trước **yếu tố huỷ** (guardian / bối cảnh) nếu muốn lớp chống trộm — dựa hạ tầng guardian của Rebirthme.
- Chấp nhận tiền tới người nhận **chậm hơn** đúng bằng cửa sổ bạn chọn.

---

## 7. Câu hỏi thường gặp

**Hỏi: Smartsend khác Protectme thế nào?**
Đáp: Smartsend **NGĂN** tổn thất TRƯỚC (cửa sổ huỷ trước khi tiền đi hẳn); Protectme **PHỦ** tổn thất SAU (bồi hoàn khi đã mất). Hai cơ chế khác trục — phòng ngừa vs bồi hoàn — và bổ sung nhau.

**Hỏi: Người nhận có phải làm gì không?**
Đáp: Khoản nhỏ thì không — hết cửa sổ là tự về họ. Khoản lớn (vượt ngưỡng bạn đặt) thì người nhận phải bấm Đồng-ý mới nhận được.

**Hỏi: Kẻ trộm có khoá gửi của tôi thì Smartsend chặn được không?**
Đáp: Muốn HUỶ khoản trong cửa sổ cần yếu tố **khác gốc khoá gửi** (guardian/bối cảnh), nên chỉ có khoá gửi thì kẻ trộm không lách được; guardian còn Freeze được hòm. Điều kiện: bạn phải bật Smartsend và cài yếu tố huỷ **TRƯỚC** khi khoá bị lộ — bật sau khi đã mất khoá thì không cứu được khoản đã đi. Chống trộm của Smartsend còn dựa lớp anti-drain của Rebirthme làm nền.

**Hỏi: Tiền có kẹt trong hòm mãi không?**
Đáp: Không. Khoản lớn mà người nhận không đồng-ý sẽ tự hoàn về bạn khi quá hạn (ReclaimTimeout).

**Hỏi: Gửi khoản nhỏ hằng ngày có phải qua Smartsend không?**
Đáp: Không bắt buộc. Smartsend là opt-in; bật khi cần, còn lại gửi thường một chữ ký đi ngay.

---

## 8. Ranh giới trung thực — điều kiện để lớp chống trộm có hiệu lực

Để không hứa hão, đây là điều kiện thật của lớp bảo vệ:

- **Bật trước khi mất khoá.** Smartsend chỉ cứu được khoản gửi SAU khi bạn đã bật + cài yếu tố huỷ. Bật sau khi khoá đã lộ không cứu được khoản đã đi — đây là giới hạn bản chất của mọi cơ chế huỷ trong cửa sổ, không phải lỗi thiếu tính năng.
- **Chống trộm của Smartsend dựa lớp anti-drain** của module Rebirthme làm nền — hai lớp bổ trợ nhau: anti-drain giới hạn thiệt hại nếu huỷ không kịp, Smartsend cho cửa sổ để huỷ trước. Đừng coi Smartsend là lá chắn chống trộm đủ một mình tách rời khỏi nền đó.
- **Yếu tố bối cảnh-ZK (khuôn mặt / ảnh bí mật / vị trí)** dựa hai dịch vụ nội bộ của đội VeData: **Glint** (sinh bằng chứng ZK cho khuôn mặt/ảnh bí mật/vị trí thiết bị, chứng minh đúng người mà không lộ dữ liệu gốc) và **Spectra** (kiểm tra "còn sống, không phải phát lại" — liveness/anti-replay — để chặn dùng ảnh/video giả hoặc dùng lại một bằng chứng cũ) — lớp này độc lập, có thể dùng guardian-factor trước khi VeData sẵn sàng.
- **Guardian Freeze cho hòm ký quỹ** dùng chung hạ tầng guardian của Rebirthme; gỡ treo cần nhóm đồng thuận, không phải một guardian đơn lẻ.

**Nền tái dùng:** hạ tầng ví theo-DID `did_payment`, guardian-recovery theo ngưỡng + timelock + Cancel của module Rebirthme; Smartsend cắm lên nền này, không dựng lại.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Smartsend-Specs/blob/main/PhoenixKey-STATUS.md)

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai).
Hạ tầng nền (ví/guardian/anti-drain): [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md), [PhoenixKey-Rebirthme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Rebirthme-Specs/blob/main/PhoenixKey-Rebirthme-Exec.md).
Tài liệu cùng bộ: [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md), [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md), [PhoenixKey-Smartsend-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Smartsend-Specs/blob/main/PhoenixKey-Smartsend-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
