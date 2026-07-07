# Feecover — Trả phí một loại, hệ thống lo phần còn lại

> Bản dành cho người dùng + đội sản phẩm. Ngôn ngữ dễ hiểu, ít thuật ngữ.
> Nguồn kỹ thuật: `PhoenixKey-Feecover-Feat-Math.md`. Đơn vị neo theo `Whitepaper-MagicLamp-Tokenomic-Vi.md`.
> **Trạng thái: đề xuất tính năng (chưa xây). Đọc mục "Ranh giới trung thực" ở cuối.**

---

## Feecover là gì (1 câu)

Feecover là lớp **trả phí gọn** của PhoenixKey: bạn chỉ cần biết một loại đơn vị duy nhất (MAGIC), còn hệ thống tự lo mọi thứ phía sau — phí mạng, phí lưu trữ, phí dịch vụ khác.

---

## Vấn đề Feecover giải

Bình thường, muốn dùng một dịch vụ trên blockchain bạn phải:

- Giữ sẵn một loại "xăng" (ADA) để trả phí mạng.
- Hiểu tỷ giá lên xuống mỗi ngày.
- Tính xem thao tác này tốn bao nhiêu, đủ số dư chưa.

Ba việc này khiến người bình thường ngại dùng. Feecover xoá cả ba:

- Bạn **không cần giữ ADA**, không cần biết ADA là gì.
- Bạn **không cần theo dõi tỷ giá** — phí dịch vụ ổn định theo sức mua, không trôi theo giá thị trường.
- Bạn chỉ nhìn **một con số duy nhất** cho mỗi dịch vụ.

---

## Bạn thấy gì khi trả phí

Trải nghiệm trong ứng dụng rất đơn giản:

1. Bạn chọn một dịch vụ — ví dụ "Neo danh tính lên chain" hoặc "Xác thực chứng nhận".
2. App hiện **một mức phí duy nhất** theo đơn vị MAGIC, ví dụ *5 MAGIC*.
3. Bạn bấm xác nhận. Xong.

Phía sau (bạn không phải làm gì):

- Số dư MAGIC của tài khoản bạn **giảm đúng 5** — như trừ tiền trong một cái ví trả trước.
- Hệ thống tự thanh toán bằng **CARP** (đồng lưu thông của hệ), theo neo cố định **1 CARP = 1 MAGIC** — nên con số bạn thấy và con số hệ thống trả **luôn bằng nhau**.
- Mọi phí mạng lặt vặt (ADA, phí lưu trữ...) do hệ thống gánh, không đến tay bạn.

Nói ngắn: **bạn định giá và nhìn phí bằng MAGIC; hệ thống quyết toán bằng CARP; hai con số bằng nhau nên bạn không phải quy đổi gì cả.**

---

## Hai đơn vị, dễ phân biệt

| | MAGIC | CARP |
|---|---|---|
| Là gì | "Điều ước" — số dư trong ví trả-trước của bạn | Đồng lưu thông của hệ, chuyển được |
| Vai trò | **Thước đo phí** (bạn nhìn con số này) | **Đồng thanh toán** (hệ thống trả bằng cái này) |
| Chuyển cho người khác được không | Không (gắn riêng bạn, để lâu sẽ tan dần) | Có |
| Bạn phải quan tâm không | Chỉ nhìn để biết phí | Không — hệ thống tự lo |

Neo cố định: **1 CARP = 1 MAGIC**. Một MAGIC được định nghĩa bằng công dịch vụ nền (theo whitepaper: *1 MAGIC = 1 GB·ngày lưu trữ lạnh*), **không neo theo USD hay bất kỳ tiền pháp định nào**. Đó là lý do phí giữ ổn định theo sức mua dịch vụ chứ không nhảy theo giá thị trường.

---

## Ẩn dụ: thẻ nạp tiền điện thoại

Hãy tưởng tượng một **thẻ trả trước dùng dịch vụ**:

- Bạn nạp một số "phút" vào thẻ (đây là **MAGIC** — số dư của riêng bạn).
- Mỗi lần dùng dịch vụ, thẻ trừ đúng số phút theo bảng giá niêm yết.
- Bạn **không phải quan tâm** nhà mạng trả tiền điện, tiền thuê trạm phát sóng thế nào — họ lo hết (phần **CARP** và phí mạng hệ thống gánh).
- "Phút" của bạn để lâu không dùng thì hết hạn — giống MAGIC để lâu sẽ tan dần.

Feecover chính là "quầy tính tiền" đặt trước cỗ máy trừ số dư đã có sẵn: nó **niêm yết bảng giá rõ ràng, chỉ cho khách PhoenixKey vào, và gom tiền về đúng quỹ** — chứ không phải một cỗ máy tính phí mới.

---

## Bảng phí ví dụ

> Số dưới đây là **minh hoạ**, chưa phải giá chính thức — mức thật đang chờ chốt.

| Dịch vụ | Bạn thấy | Hệ thống trả |
|---|---|---|
| Neo / cập nhật danh tính lên chain | 5 MAGIC | 5 CARP |
| Đổi khoá thành viên | 2 MAGIC | 2 CARP |
| Xác thực chứng nhận (chọn-lọc-tiết-lộ) | 1 MAGIC | 1 CARP |
| Thao tác giữ hộ (custody) | 3 MAGIC | 3 CARP |

Cột "bạn thấy" và cột "hệ thống trả" **luôn bằng nhau** — đó là điểm mấu chốt giúp bạn không phải tính tỷ giá.

---

## Feecover thêm gì (so với thứ đã có)

Cỗ máy trừ số dư MAGIC **đã tồn tại**. Feecover là **lớp mỏng** đặt lên trên, chỉ thêm ba việc:

1. **Bảng phí ổn định theo loại dịch vụ** — mỗi dịch vụ PhoenixKey có một mức phí rõ ràng, ổn định dài hạn, chỉ đổi qua biểu quyết cộng đồng.
2. **Cổng chỉ dành cho người dùng PhoenixKey** — chỉ tài khoản có danh tính PhoenixKey hợp lệ mới dùng được Feecover.
3. **Gom phí về đúng quỹ** — phí gom theo tầng (ứng dụng → nền tảng), cuối mỗi chu kỳ đối soát và chuyển về **Quỹ Phoenix**.

---

## Cải tiến bạn nhận được

- **Không cần giữ ADA** hay hiểu "xăng blockchain" là gì.
- **Không cần theo dõi tỷ giá** — một con số duy nhất, ổn định theo sức mua dịch vụ.
- **Phí không trôi theo giá thị trường** — biến động thị trường do lớp ổn định giá riêng của hệ gánh, không đội lên phí bạn trả.
- **Minh bạch** — bảng phí công khai, chỉ đổi qua quản trị cộng đồng.

---

## Câu hỏi thường gặp

**MAGIC là tiền à? Tôi rút ra được không?**
Không. MAGIC không phải token chuyển được — nó là **số dư trong ví trả-trước** gắn riêng bạn, để đo và trả phí dịch vụ. Giống "phút" trong thẻ nạp: dùng được, nhưng không bán lại được.

**Tại sao tôi thấy MAGIC mà hệ thống lại trả CARP?**
Để bạn có **một thước đo ổn định** (MAGIC) mà không phải đụng tới đồng lưu thông thật (CARP). Vì neo cứng 1 CARP = 1 MAGIC, hai con số luôn bằng nhau — bạn không mất gì khi quy đổi.

**Phí có tăng khi giá thị trường tăng không?**
Không tự động. Phí neo theo **sức mua dịch vụ nền**, không theo giá USD hay giá token trên sàn. Phí chỉ đổi khi cộng đồng biểu quyết điều chỉnh — không phải một cỗ máy bám giá thị trường.

**Tôi cần chuẩn bị ADA trước khi dùng không?**
Không. Đó chính là điểm của Feecover: mọi phí mạng phía sau do hệ thống lo.

**MAGIC để lâu không dùng thì sao?**
Nó tan dần theo thời gian (như gói cước có hạn dùng). Thiết kế này để khuyến khích dùng thật, không tích trữ.

**Ai được dùng Feecover?**
Chỉ người dùng có **danh tính PhoenixKey** hợp lệ. Đây là cổng bảo vệ, không phải rào cản — tạo danh tính PhoenixKey là bước bình thường trong ứng dụng.

---

## Ranh giới trung thực

Nói thẳng để không hứa quá:

- **Feecover CHƯA được xây.** Đây là **đề xuất tính năng** (design doc), chưa có mã chạy được.
- **Đang chờ hai việc phía kỹ thuật:**
  1. Mô hình MAGIC on-chain cần được đội MAGIC chốt lại (gắn danh tính vào từng lần tiêu phí, hiện chưa có).
  2. Đồng CARP cần có định danh chính thức trên chain (policy-id) trước khi luồng thanh toán chạy được.
- **Feecover KHÔNG tự làm những việc này** — nó dùng lại hạ tầng của các đội khác, chỉ thêm bảng phí + cổng DID + gom phí.
- **Mức phí trong bảng là minh hoạ**, chưa phải giá chính thức. Giá thật, chu kỳ điều chỉnh, và ranh giới tầng ứng dụng/nền tảng còn chờ chốt.
- **Việc ổn định giá đồng CARP nằm NGOÀI Feecover** — do một lớp riêng của hệ đảm nhận. Feecover chỉ bàn giao phần dư về Quỹ Phoenix.

Khi hai phụ thuộc kỹ thuật ở trên sẵn sàng, Feecover mới chuyển từ đề xuất sang xây thật.
