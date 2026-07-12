# PhoenixKey — Rebirthme: Chiếc ví đi theo bạn, không đi theo chìa khoá

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản-phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **Ví Phượng-hoàng là gì**, **khôi phục ra sao khi mất máy/mất seed**, **tiền được bảo vệ thế nào**, bằng ngôn ngữ đời thường.
> **Module:** Rebirthme (slug `Rebirthme`). **Ngày:** 2026-07-09.
> Nội dung bám các đặc-tả cùng bộ: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) (toán), [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md) (kỹ-thuật), [PhoenixKey-Rebirthme-Exec.md](./PhoenixKey-Rebirthme-Exec.md) (điều-hành).
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 1. Một câu là gì

**Ví Phượng-hoàng là ví mà tài sản đi theo DANH TÍNH của bạn, không đi theo chìa-khoá. Địa chỉ ví bất-biến suốt đời; mất máy/mất seed thì khôi-phục xong tiêu tiếp; kẻ trộm chiếm được khoá cũng không rút sạch được tức thì. Và mọi bí-mật của bạn — seed, khoá mạng khác, tài-liệu mật — được sao-lưu tự-động, mã-hoá, phân-tán, để mất máy không bao giờ là mất trắng.**

Hình dung đơn giản: ví thường như một **cái két có đúng một chìa** — mất chìa là mất két. Ví Phượng-hoàng như một **cái két gắn với KHUÔN MẶT bạn** — đổi chìa bao nhiêu lần, mất chìa, làm lại chìa mới, két vẫn của bạn, tiền vẫn nằm nguyên trong đó.

---

## 2. Vấn đề nó giải

Ví crypto truyền-thống có ba chỗ đau chí-mạng:

1. **Mất seed = mất tất-cả, vĩnh-viễn.** Chép nhầm một từ trong 24 từ, cháy nhà mất tờ giấy, quên chỗ giấu — không ai cứu được. Không có "quên mật khẩu, bấm khôi-phục".
2. **Lộ seed = mất tất-cả, tức-thì.** Ai thấy 24 từ của bạn là chi được sạch ví trong một block. Không có trần, không có delay, không có nút "đóng băng".
3. **Địa chỉ chết theo khoá.** Đổi khoá (rotate) thường phải đổi địa chỉ; địa chỉ cũ vẫn nhận tiền tương-lai mà bạn quên mất → mất số tiền vào đó.

**Ví Phượng-hoàng lật ngược cả ba:**
- **Mất máy/seed → khôi-phục được** nhờ người-bảo-chứng (guardian) uỷ-quyền đổi khoá, cộng sao-lưu tự-động. Không cần chép tay 24 từ để sống sót.
- **Lộ khoá → không mất sạch.** Anti-drain đặt trần rút mỗi cửa-sổ; rút lớn cần delay hoặc khoá phụ; nghi tấn-công thì đóng-băng, chỉ cho rút về nơi an-toàn khai-báo-trước.
- **Địa chỉ bất-biến suốt đời DID.** Đổi khoá vẫn cùng địa chỉ. Và phả-hệ khoá cũ được giữ lại để quét-rút tiền vào địa chỉ cũ.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Có ví ngay khi tạo danh-tính
Bạn tạo danh-tính bằng vân-tay (xem module Core Anchorme). Cùng lúc, một **Ví Phượng-hoàng** ra đời — một địa chỉ gắn thẳng vào DID của bạn. Không phải nhớ gì thêm.

### Bước 2 — Nhận và gửi như ví thường
Bạn có một địa chỉ để nhận ADA/LAMP/CARP/token. Gửi đi thì chọn ví nguồn → địa chỉ nhận → số lượng → xác-nhận bằng vân-tay. Giống mọi ví.

### Bước 3 — Đặt người-bảo-chứng (guardian) để phòng mất máy
Bạn chọn 2-3 người thân, hoặc một tổ-chức/Agent tin-cậy, làm **người-bảo-chứng**. Họ KHÔNG giữ tiền, KHÔNG giữ chìa của bạn — họ chỉ giữ một khoá của chính họ và đồng-ý ký khi bạn cần lấy lại quyền. Ứng-dụng tự trao-đổi khoá qua QR; bạn chỉ thấy "Đã thêm [Anh trai] làm người bảo chứng ✓".

### Bước 4 — Bí-mật được sao-lưu tự-động
Seed, khoá mạng khác (BTC/ETH/SOL), tài-liệu mật của bạn được **mã-hoá ngay trên máy** rồi rải lên mạng lưu-trữ phân-tán (LampNet). Node giữ hộ chỉ thấy dữ-liệu đã khoá, không đọc được nội-dung. Bạn thấy "Đã sao-lưu phân-tán ✓".

### Bước 5 — Khi mất máy: lấy lại quyền
Máy mới, bạn nhập DID cần khôi-phục. Máy mới sinh khoá mới. Nhờ **đủ số người-bảo-chứng ký** (hoặc một phương-thức khác), qua một khoảng chờ an-toàn, khoá mới thay khoá cũ. **Tài-sản ở Ví Phượng-hoàng tự về tay** vì địa chỉ không đổi — controller mới chi được.

### Bước 6 — Khi nghi bị trộm: đóng-băng
Nghi khoá đã lộ, bạn bấm **Đóng-băng ngay**. Mọi đường chi bị chặn TRỪ một đường: rút về **địa chỉ an-toàn** bạn đã khai-báo trước. Kẻ trộm không chuyển đi đâu khác được. Chỉ bạn gỡ băng.

---

## 4. Các cơ-chế — kể bằng đời thường

### 4.1 Ví đi theo danh-tính (không theo chìa)
Địa chỉ Ví Phượng-hoàng là một "cái khoá thông-minh" đọc DANH TÍNH bạn mỗi lần chi: nó hỏi "DID này còn sống không, và khoá đang-điều-khiển hiện-tại có ký không". Vì nó đọc **khoá hiện-tại** chứ không phải một khoá cứng, đổi khoá bao nhiêu lần vẫn chi được cùng một địa chỉ. Ví thường: địa chỉ dán chết vào khoá, đổi khoá là đổi địa chỉ.

### 4.2 Ví thường vs Ví Phượng-hoàng (hai kiểu song song)
Bạn có CẢ HAI, chọn theo nhu-cầu:
- **Ví tiêu-chuẩn (Standard)** — ví theo cụm seed, tương-thích chuẩn BIP39/CIP-1852. Để nhận/quét địa chỉ seed cũ, dùng chung hệ-sinh-thái ví HD.
- **Ví Phượng-hoàng (Phoenix)** — ví theo danh-tính, bất-tử, địa chỉ vĩnh-viễn.

### 4.3 Ba tầng địa chỉ CỦA Ví Phượng-hoàng (cho cả cá-nhân lẫn doanh-nghiệp)
Đây là ba tầng địa-chỉ **bên trong loại ví Phoenix** — KHÔNG phải ba loại ví. Ví CHỈ có 2 loại (§4.2: Standard / Phoenix); Phoenix có 3 tầng địa-chỉ dưới đây, chọn theo mức công-khai/riêng-tư bạn cần:
- **L1 — Địa chỉ công-khai của bạn (danh-tính công-khai):** vừa nhận tiền vừa staking được, công-bố trong hồ-sơ DID. Là "gương mặt" của bạn.
- **L2 — Địa chỉ công-khai doanh-nghiệp:** giống L1 nhưng cho tổ-chức (in lên hoá-đơn, README).
- **L3 — Địa chỉ riêng-tư ×N:** doanh-nghiệp tạo một địa chỉ riêng cho mỗi khách-hàng để đối-soát, KHÔNG công-bố, và trên chuỗi không lộ chung một chủ, được kiểm-soát bởi validator `did_subaddr` (validator = một đoạn hợp đồng thông minh chạy ngay trên chuỗi, tự động kiểm luật cho mỗi giao dịch, không ai sửa tay được). Đây cũng chính là địa chỉ mà **Easteregg** dùng làm "địa chỉ riêng" (module Easteregg gọi biến-thể-ẩn dựa trên tầng này — xem [PhoenixKey-Easteregg-Vi-Feat.md](./PhoenixKey-Easteregg-Vi-Feat.md)).

> **Easteregg** không phải một loại ví thứ ba — nó là **các biến-thể ẩn-danh của chính Ví Phượng-hoàng**, chọn theo mức riêng-tư (công khai / ẩn danh / vào kho). Xem [PhoenixKey-Easteregg-Vi-Feat.md](./PhoenixKey-Easteregg-Vi-Feat.md).
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

### 4.4 Người-bảo-chứng = uỷ-quyền, KHÔNG giữ mảnh
Nhiều hệ recovery bắt người-bảo-chứng giữ một "mảnh" bí-mật của bạn — nếu họ thông-đồng hoặc bị hack, mất. PhoenixKey **đã bỏ hẳn mô-hình mảnh (Shamir)**. Người-bảo-chứng chỉ là một **khoá thành-viên ngủ-đông**: khi bạn còn khoẻ, khoá họ vô-hiệu; khi bạn cần, đủ số họ ký để uỷ-quyền đổi khoá cho bạn. Họ **không bao giờ thấy hay giữ** khoá/tài-sản của bạn. Đủ MỘT phương-thức là khôi-phục được (người-bảo-chứng / chứng-thực VeData-Glint — dịch-vụ xác-thực khuôn mặt/ảnh của đội VeData / Midnight — một mạng blockchain khác, chuyên về giao-dịch riêng-tư, dùng ở đây làm một kênh chứng-thực độc-lập bên ngoài PhoenixKey).

### 4.5 Chống rút-sạch (anti-drain) — cái két có van
Ngay cả khi kẻ trộm có khoá, cái két có một **cái van**: mỗi cửa-sổ thời-gian chỉ chảy ra tối-đa một mức trần. Muốn rút nhiều hơn phải (a) mở yêu-cầu công-khai rồi chờ — trong lúc chờ bạn thấy và HUỶ được, hoặc (b) có thêm một khoá phụ/người-bảo-chứng ký. Rút nhỏ đời-thường vẫn một chữ-ký, nhanh gọn. Đây là lớp bảo-vệ CỐT-TỬ, bắt-buộc cho mọi DID giữ giá-trị đáng-kể — không phải tính-năng tuỳ-chọn (xem `PhoenixKey-Rebirthme-Math.md` I-CURVE-4).
→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

> **Smartsend** (gửi-có-bảo-vệ — nút "hoàn-tác" cho giao-dịch) nay là module riêng — xem [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md).

### 4.6 Phả-hệ khoá — giữ mọi khoá cũ, không xoá
Đổi khoá thì khoá cũ KHÔNG bị xoá — nó được đánh dấu "đã xoay" và giữ lại. Vì địa chỉ cũ vẫn có thể nhận tiền tương-lai (airdrop, người trả nợ). Màn phả-hệ hiện dòng thời-gian mọi khoá của bạn, cảnh-báo khi địa chỉ cũ có tiền vào, cho nút "Quét & rút". Khoá cũ lộ cũng không mất DID (khoá đã đổi trên chuỗi).

### 4.7 Xuất seed — vô-hiệu TRƯỚC khi hiện
PhoenixKey **không bắt** bạn chép 24 từ (sao-lưu tự-động đã lo khôi-phục). Khi bạn CHỦ-ĐỘNG bấm Xuất seed, mặc-định hệ-thống **đổi khoá + di-tản tài-sản TRƯỚC** rồi mới hiện — nên cụm 24 từ hiện ra đã **bị vô-hiệu** với DID (chỉ để lưu-niệm). Hé-lộ seed không còn là hé-lộ chìa-khoá tài-sản.

---

## 5. Bảng cải-tiến (ví thường vs Ví Phượng-hoàng)

| Điểm | Ví seed-phrase / ví cứng thường | Ví Phượng-hoàng (PhoenixKey) |
|---|---|---|
| **Mất seed/máy** | Mất tất-cả, vĩnh-viễn | Khôi-phục qua người-bảo-chứng + sao-lưu tự-động |
| **Lộ khoá** | Mất sạch tức-thì | Anti-drain giới-hạn thiệt-hại mỗi cửa-sổ; đóng-băng được, chỉ rút về địa chỉ an-toàn khai-báo-trước |
| **Địa chỉ khi đổi khoá** | Đổi khoá = đổi địa chỉ | Bất-biến suốt đời DID |
| **Người-bảo-chứng** | Giữ mảnh bí-mật (rủi-ro thông-đồng) | Chỉ uỷ-quyền, KHÔNG giữ mảnh (đã bỏ Shamir) |
| **Sao-lưu bí-mật** | Chép tay / cloud đọc được | Mã-hoá trên máy + phân-tán, node không đọc được |
| **Khoá cũ** | Quên = mất tiền vào địa chỉ cũ | Giữ phả-hệ, quét-rút được |
| **Xuất seed** | Lộ 24 từ = mất hết | Vô-hiệu trước khi hiện (rotate-before-reveal) |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Một ví bất-tử, địa chỉ vĩnh-viễn, sống qua đổi-khoá/khôi-phục.
- Đường khôi-phục không cần chép tay 24 từ.
- Sao-lưu tự-động, tự-chủ cho MỌI bí-mật (không riêng seed Cardano).
- Các lớp chống-trộm: trần rút, đóng-băng. Gửi-có-bảo-vệ nay ở module riêng — xem [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md).

**Bạn cần:**
- **Đặt ít nhất 2 người-bảo-chứng** (hoặc bật một phương-thức khôi-phục khác) — 1 người là chưa an-toàn. Có 30 ngày ân-hạn nhắc-nhở.
- Khai-báo **địa chỉ an-toàn** để đóng-băng có đích rút về.
- Hiểu rằng khi CHỦ-ĐỘNG dùng khoá mạng khác (BTC/ETH) trong kho, PhoenixKey chỉ giữ-hộ, KHÔNG tự ký thay — bạn mở từng lần.

---

## 7. Câu hỏi thường gặp

**Hỏi: Người-bảo-chứng có xem được tiền/khoá của tôi không?**
Đáp: Không. Họ chỉ giữ khoá của chính họ và bỏ phiếu giúp bạn lấy lại quyền. Một mình không ai mở được danh-tính bạn; cần đủ số người theo ngưỡng bạn đặt.

**Hỏi: Nếu người-bảo-chứng thông-đồng thì sao?**
Đáp: Có timelock (khoảng chờ) và nút Huỷ trong cửa-sổ đó. Vai "xác-nhận-còn-sống" (veto) chặn thông-đồng mạnh hơn — xem `PhoenixKey-Rebirthme-Math.md` I-GUARD-VETO. Và họ không giữ mảnh bí-mật nào, nên dù thông-đồng cũng không tự dựng lại khoá của bạn được.

**Hỏi: Mất điện thoại thì tiền có mất không?**
Đáp: Không. Tài-sản ở Ví Phượng-hoàng đi theo danh-tính. Khôi-phục xong (khoá mới), địa chỉ cũ vẫn là của bạn, chi tiếp bình-thường.

**Hỏi: Kẻ trộm có khoá của tôi thì rút sạch được không?**
Đáp: Không thể rút sạch tức-thì. Lớp anti-drain (trần rút + đóng-băng) giới-hạn kẻ trộm chỉ rút được tối-đa một cửa-sổ; rút lớn hơn cần delay hoặc khoá-phụ/người-bảo-chứng ký. Nghi bị trộm, bạn đóng-băng ngay — chỉ còn đường rút về địa chỉ an-toàn đã khai-báo trước.

**Hỏi: Tôi cất được khoá Bitcoin/Ethereum vào PhoenixKey không?**
Đáp: Được — để cất + khôi-phục. PhoenixKey mã-hoá trên máy rồi phân-tán. Nhưng PhoenixKey KHÔNG tự ký giao-dịch mạng khác; bạn muốn dùng thì mở từng lần.

**Hỏi: Xuất seed có nguy-hiểm không?**
Đáp: Mặc-định an-toàn: hệ-thống đổi khoá + di-tản tài-sản trước khi hiện, nên seed hiện ra đã vô-hiệu với DID. Bạn có thể tắt hành-vi này nếu hiểu rõ rủi-ro.

---

## 8. Luật thiết-kế cốt-tử

Ba luật sau là điều-kiện-bắt-buộc để lời hứa ở §2 ("mất seed không mất trắng, lộ khoá không mất sạch") đúng trong thực-tế — không phải tính-năng phụ:

- **Anti-drain là LOAD-BEARING cho mọi DID giữ giá-trị đáng-kể**, không phải tính-năng trang-trí. Vì khoá phần-cứng (P-256, Secure Enclave) không verify được on-chain, quyền chi value quy về khoá gốc (seed, Ed25519 controller) — nên trần rút + đóng-băng chủ-động là lớp cản DUY-NHẤT giữa lộ seed và mất sạch. Xem `PhoenixKey-Rebirthme-Math.md` I-CURVE-4.
- **Second-factor rút-lớn PHẢI khác gốc seed.** Khoá phụ/người-bảo-chứng dùng để duyệt rút-lớn không được dẫn-xuất từ cùng **Master_KEK** (Master Key-Encryption-Key — chiếc khoá gốc mã-hoá mọi khoá con khác của bạn, sinh ra từ cùng một seed) — nếu không, một lần lộ seed sập luôn cả hai lớp bảo-vệ. Yêu-cầu phần-cứng độc-lập hoặc kênh khác (VeData-Glint, EmailOracle). Xem I-CURVE-5.
- **L3 (địa chỉ riêng-tư ×N) và Easteregg dùng chung một validator `did_subaddr`** — không phải hai cơ-chế khác nhau. Thiết-kế phải giữ tính unlinkable (không lần ra được các địa chỉ đó là cùng một chủ) khi NHẬN; khi CHI, lộ tối-thiểu một `tag_i` (một nhãn nhận-dạng gắn theo mỗi địa chỉ con) là giới-hạn bản-chất của **eUTXO** — mô hình sổ-cái của Cardano, nơi mỗi khoản tiền là một "tờ tiền" rời rạc phải tiêu trọn rồi trả lại tiền thừa, khác mô hình "số dư tài khoản" quen thuộc; việc tiêu một tờ luôn để lại dấu-vết trên chuỗi về việc nó đã bị tiêu (ẩn cả lúc chi cần ZK — bằng-chứng mật-mã không tiết-lộ dữ-liệu gốc — hoặc Midnight, ngoài phạm-vi module này).

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## Nguồn

Nguồn thiết-kế nội-bộ (không công khai).
Code: `PhoenixKey-Validator/validators/did_payment.ak`, `lib/phoenixkey/{taad_logic,auth_logic}.ak`.
Tài-liệu cùng bộ: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md), [PhoenixKey-Rebirthme-Exec.md](./PhoenixKey-Rebirthme-Exec.md).
