# PhoenixKey — Feecover (FEATURE, tiếng Việt)

> **Module:** Feecover (Trừu tượng hoá phí / fee-abstraction).
> **Loại doc:** Feature — hướng người dùng + đội sản phẩm. Ngôn ngữ đời thường, ít thuật ngữ.
> **Ngày:** 2026-07-09.
> **Đối tượng đọc:** người dùng cuối, đội sản phẩm, người viết tài liệu.

Tài liệu cùng bộ: [PhoenixKey-Feecover-Math.md](./PhoenixKey-Feecover-Math.md) (toán + bất biến), [PhoenixKey-Feecover-Tech.md](./PhoenixKey-Feecover-Tech.md) (kỹ thuật), [PhoenixKey-Feecover-Exec.md](./PhoenixKey-Feecover-Exec.md) (điều hành).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## 1. Feecover là gì (một câu)

Feecover là lớp **trả phí gọn** của PhoenixKey: bạn chỉ cần biết một loại đơn vị duy nhất (MAGIC), còn hệ thống tự lo mọi thứ phía sau — phí mạng, phí lưu trữ, phí dịch vụ khác.

---

## 2. Vấn đề Feecover giải

Bình thường, muốn dùng một dịch vụ trên blockchain bạn phải:

- Giữ sẵn một loại "xăng" (ADA) để trả phí mạng.
- Hiểu tỷ giá lên xuống mỗi ngày.
- Tính xem thao tác này tốn bao nhiêu, đủ số dư chưa.

Ba việc này khiến người bình thường ngại dùng. Feecover xoá cả ba:

- Bạn **không cần giữ ADA**, không cần biết ADA là gì.
- Bạn **không cần theo dõi tỷ giá** — phí dịch vụ ổn định theo sức mua, không trôi theo giá thị trường.
- Bạn chỉ nhìn **một con số duy nhất** cho mỗi dịch vụ.

---

## 3. Hành trình của bạn — bấm gì, thấy gì

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

## 4. Các cơ chế kể bằng đời thường

### 4.1 Hai đơn vị, dễ phân biệt

| | MAGIC | CARP |
|---|---|---|
| Là gì | "Điều ước" — số dư trong ví trả trước của bạn | Đồng lưu thông của hệ, chuyển được |
| Vai trò | **Thước đo phí** (bạn nhìn con số này) | **Đồng thanh toán** (hệ thống trả bằng cái này) |
| Chuyển cho người khác được không | Không (gắn riêng bạn, để lâu sẽ tan dần) | Có |
| Bạn phải quan tâm không | Chỉ nhìn để biết phí | Không — hệ thống tự lo |

Neo cố định: **1 CARP = 1 MAGIC**. Một MAGIC được định nghĩa bằng công dịch vụ nền (theo whitepaper: *1 MAGIC = 1 GB·ngày lưu trữ lạnh*). "Nanogic" là **đơn vị nhỏ nhất** của MAGIC — giống 1 đồng so với 1 triệu đồng: `1 MAGIC = 10^9 nanogic` (nano = một phần tỷ), và quy ước `1 nanogic = 1 byte·ngày` (đơn vị nhỏ nhất khớp với 1 byte lưu trữ trong 1 ngày). MAGIC **không neo theo USD hay bất kỳ tiền pháp định nào**. Đó là lý do phí giữ ổn định theo sức mua dịch vụ chứ không nhảy theo giá thị trường.

### 4.2 Ẩn dụ: thẻ nạp tiền điện thoại

Hãy tưởng tượng một **thẻ trả trước dùng dịch vụ**:

- Bạn nạp một số "phút" vào thẻ (đây là **MAGIC** — số dư của riêng bạn).
- Mỗi lần dùng dịch vụ, thẻ trừ đúng số phút theo bảng giá niêm yết.
- Bạn **không phải quan tâm** nhà mạng trả tiền điện, tiền thuê trạm phát sóng thế nào — họ lo hết (phần **CARP** và phí mạng hệ thống gánh).
- "Phút" của bạn để lâu không dùng thì hết hạn — giống MAGIC để lâu sẽ tan dần.

Feecover chính là "quầy tính tiền" đặt trước cỗ máy trừ số dư đã có sẵn: nó **niêm yết bảng giá rõ ràng, chỉ cho khách PhoenixKey vào, và gom tiền về đúng quỹ** — chứ không phải một cỗ máy tính phí mới.

### 4.3 Bảng phí ví dụ

> Số dưới đây là **minh hoạ**, chưa phải giá chính thức — mức thật đang chờ chốt (D1).

| Dịch vụ | Bạn thấy | Hệ thống trả |
|---|---|---|
| Neo / cập nhật danh tính lên chain | 5 MAGIC | 5 CARP |
| Đổi khoá thành viên | 2 MAGIC | 2 CARP |
| Xác thực chứng nhận (chọn lọc tiết lộ) | 1 MAGIC | 1 CARP |
| Thao tác giữ hộ (custody) | 3 MAGIC | 3 CARP |

Cột "bạn thấy" và cột "hệ thống trả" **luôn bằng nhau** — đó là điểm mấu chốt giúp bạn không phải tính tỷ giá.

---

## 5. Bảng cải tiến (so với thứ đã có)

Cỗ máy trừ số dư MAGIC **đã tồn tại** (ConsumeMAGIC). Feecover là **lớp mỏng** đặt lên trên, chỉ thêm:

| | Trước Feecover | Có Feecover |
|---|---|---|
| Đơn vị bạn giữ | Phải có ADA làm "xăng" | Chỉ cần số dư MAGIC |
| Giá dịch vụ | Trôi theo thị trường/tắc nghẽn | Một con số danh nghĩa ổn định (nanogic) |
| Cách trả | Tự quy đổi tỷ giá | Hệ thống trả CARP, neo 1:1, không phải quy đổi |
| Ai được dùng | Ví bất kỳ | Chỉ tài khoản có danh tính PhoenixKey |
| Phí về đâu | (không có tầng gom) | Gom theo tầng App→Platform, cuối epoch về Quỹ Phoenix |

Ba việc Feecover thêm:

1. **Bảng phí ổn định theo loại dịch vụ** — mỗi dịch vụ PhoenixKey có một mức phí rõ ràng, ổn định dài hạn, chỉ đổi qua biểu quyết cộng đồng.
2. **Cổng chỉ dành cho người dùng PhoenixKey** — chỉ tài khoản có danh tính PhoenixKey hợp lệ mới dùng được Feecover.
3. **Gom phí về đúng quỹ** — phí gom theo tầng (ứng dụng → nền tảng), cuối mỗi chu kỳ đối soát và chuyển về **Quỹ Phoenix** (chỉ CARP).

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn nhận được:**

- **Không cần giữ ADA** hay hiểu "xăng blockchain" là gì.
- **Không cần theo dõi tỷ giá** — một con số duy nhất, ổn định theo sức mua dịch vụ.
- **Phí không trôi theo giá thị trường** — biến động thị trường do lớp ổn định giá riêng của hệ gánh, không đội lên phí bạn trả.
- **Minh bạch** — bảng phí công khai, chỉ đổi qua quản trị cộng đồng.

**Nghĩa vụ của bạn:**

- Có một **danh tính PhoenixKey** hợp lệ (cổng bảo vệ, không phải rào cản — tạo danh tính là bước bình thường trong app).
- Giữ đủ số dư MAGIC trong ví trả trước cho dịch vụ muốn dùng.

---

## 7. Câu hỏi thường gặp (FAQ)

**MAGIC là tiền à? Tôi rút ra được không?**
Không. MAGIC không phải token chuyển được — nó là **số dư trong ví trả trước** gắn riêng bạn, để đo và trả phí dịch vụ. Giống "phút" trong thẻ nạp: dùng được, nhưng không bán lại được.

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

## 8. Ranh giới thiết kế — luật bắt buộc khi build

- **Feecover là lớp mỏng đặt lên trên ConsumeMAGIC** — dùng lại nguyên trạng cỗ máy trừ số dư MAGIC lõi (ConsumeMAGIC — C-CM-1..5), KHÔNG dựng engine mới, KHÔNG chạm code ConsumeMAGIC.
- **5 cấu trúc của Feecover** (bảng phí `ServiceFeeSchedule`, cổng DID `FeecoverGate`, gom phí `FeecoverAccrual`, đối soát `FeecoverEpochSettle`, quy đổi CARP) là 5 thành phần chính sách Feecover PHẢI thêm — đặc tả đầy đủ ở Tech §3.
- **Ba phụ thuộc kỹ thuật bắt buộc trước khi thanh toán CARP chạy được (B1/B2/B3):**
  1. **B1 — Mô hình MAGIC** phải là **số dư trong Vault** (một sổ cái nội bộ ghi số dư theo từng tài khoản, giống số dư ngân hàng — không phải một loại token riêng trên chuỗi), **non-transferable** (không chuyển được cho người khác, chỉ tiêu được), **KHÔNG có policy-id** (policy-id = "mã định danh loại token" trên Cardano; MAGIC cố tình KHÔNG có mã này vì nó không phải một token lưu thông độc lập, chỉ là con số kế toán nội bộ).
  2. **B2 — Đồng CARP** phải có định danh chính thức trên chain (policy-id — mã định danh loại token trên Cardano) trước khi luồng thanh toán chạy được.
  3. **B3 — Gắn danh tính vào từng lần tiêu phí** (`did_commit` per-DID — mỗi lần trừ phí đều ghi kèm DID của đúng người tiêu, không chỉ ghi chung theo "chủ ví"): gom phí phải truy được về từng người thật, không chỉ ở mức "chủ ví" (owner).
- **Feecover KHÔNG tự làm những việc này** — nó dùng lại hạ tầng của các đội khác, chỉ thêm bảng phí + cổng DID + gom phí.
- **Mức phí trong bảng là minh hoạ**, chưa phải giá chính thức — xem D1 (Exec §7). Giá thật, chu kỳ điều chỉnh, và ranh giới tầng ứng dụng/nền tảng chốt qua quản trị cộng đồng (D2, D4).
- **Việc ổn định giá đồng CARP nằm NGOÀI Feecover** — do một lớp riêng của hệ (L3, cơ chế **GreenPeg/RedPeg** — hai cơ chế bơm/hút CARP theo hai chiều để giữ giá CARP ổn định, thuộc phạm vi module khác, không phải Feecover) đảm nhận. Feecover chỉ bàn giao phần dư về Quỹ Phoenix.
- **Luật thiết kế khâu đối soát cuối-epoch:** không được dựa vào provider tự trung thực — validator settle PHẢI ép forced-settle + khớp Σ CARP đúng datum, chi tiết bất biến ở Math §4.2 (FEECOVER-SETTLE-*) / Tech §4.3.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#feecover)

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai) — mô hình 3-token, ConsumeMAGIC tái dùng (đã qua rà soát nội bộ).
- Đơn vị/peg canonical: `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` §4 (MAGIC), §5 (CARP).
- Đối chiếu tài liệu cùng bộ: [PhoenixKey-Feecover-Math.md](./PhoenixKey-Feecover-Math.md), [PhoenixKey-Feecover-Tech.md](./PhoenixKey-Feecover-Tech.md), [PhoenixKey-Feecover-Exec.md](./PhoenixKey-Feecover-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
