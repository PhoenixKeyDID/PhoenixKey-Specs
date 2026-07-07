# Protectme — Bảo vệ tài sản, bồi hoàn phần bị mất

> Bản mô tả tính năng cho **người dùng**. Viết dễ hiểu, ít thuật ngữ.
> Đây là **bản đề xuất** — nhiều con số và quy chế còn **chờ chốt** (xem cuối bài, mục
> "Nói thật với bạn"). Đọc kỹ mục đó trước khi tin bất kỳ con số nào ở đây.

---

## Protectme là gì? (một câu)

**Protectme là một lớp bảo vệ dạng bảo hiểm: khi tài sản của bạn bị đánh cắp, Protectme
bồi hoàn phần còn thiếu — sau khi ví của bạn đã tự cứu lại được nhiều nhất có thể.**

Nói cách khác: nó không thay thế các lớp an toàn khác, mà **vá nốt phần còn lại** mà các
lớp kia không kéo về được.

---

## Vì sao cần Protectme?

Ví PhoenixKey đã có ba lớp an toàn tự động chồng lên nhau:

1. **Ví gắn liền với danh tính của bạn** — kể cả mất điện thoại hay lộ chìa khoá, bạn khôi
   phục lại quyền và tài sản trong ví vẫn còn nguyên.
2. **Chặn rút sạch** — kẻ trộm không thể rút hết một lúc; có hạn mức mỗi lần, có nút khoá
   khẩn cấp để bạn chặn lại.
3. **Người thân bảo lãnh (guardian)** — khi bạn mất chìa khoá, người thân giúp bạn lấy lại
   quyền.

Nhưng vẫn còn **một khe hở** ba lớp trên không bịt được: nếu kẻ trộm **kịp rút một phần**
trong lúc bạn chưa kịp khoá khẩn cấp, phần đó đã **rời khỏi ví, sang túi kẻ trộm** — không
kéo về được nữa. Cũng có tài sản để ở nơi khác (ví thường, chìa khoá mạng khác) mà bạn chủ
động mở ra dùng rồi bị lừa mất.

**Protectme phủ đúng phần bị mất này** — phần thực sự đã rời khỏi tay bạn.

---

## Phủ cái gì — KHÔNG phủ cái gì (phần quan trọng nhất)

### Protectme PHỦ

- **Tài sản bị đánh cắp thật** — có kẻ thứ ba chiếm đoạt, và phần đó đã rời ví, không kéo
  về được.
- Hai loại nguyên nhân, xử **khác nhau**:
  - **Lỗi của hệ thống** (viết tắt trong bài: *lỗi hệ thống*) — trộm được là do lỗ hổng
    của chính phần mềm/giao thức PhoenixKey. Đây là **trách nhiệm của hệ thống** → phủ
    **mạnh**, có thể tới toàn bộ phần bị mất (trong giới hạn quỹ).
  - **Sơ suất của bạn** (*sơ suất người dùng*) — trộm được là do bạn bị lừa: dán chìa khoá
    vào web giả, cấp quyền cho ứng dụng độc, bị dụ bấm ký. Hệ thống làm đúng, bạn bị lừa →
    phủ **có giới hạn** (trần thấp hơn, bạn chịu một phần).

### Protectme KHÔNG phủ

- **Mất giá thị trường** — coin/token rớt giá thì đó không phải trộm.
- **Chuyển nhầm** — bạn tự gửi nhầm địa chỉ.
- **Quên chìa khoá** — cái này lớp khôi phục / người thân bảo lãnh lo, không phải bồi hoàn.
- **Phần ví đã TỰ CỨU được** — nếu tài sản vẫn còn trong ví sau khi khôi phục, hoặc bạn đã
  kịp chặn/huỷ lệnh rút, thì **coi như không mất** → không bồi hoàn (vì có mất đâu mà đền).
- **Sự cố khi hợp đồng bảo vệ chưa có hiệu lực** — mới bật Protectme, còn trong thời gian
  chờ, hoặc đã ngừng đóng phí quá hạn.

> **Điểm mấu chốt:** Protectme chỉ đền **phần dư** — phần thực sự mất sau khi mọi lớp an
> toàn đã làm hết sức. Không đền trùng cái đã được cứu.

---

## Bạn sẽ thấy gì? (trải nghiệm thực tế)

### 1. Mua/bật bảo vệ

- Vào màn **Protectme**, bật bảo vệ và chọn **mức phủ** (đền tối đa bao nhiêu).
- App hiện **phí bảo vệ** (premium) — đóng định kỳ (tháng/năm). Phí trừ tự động, bạn không
  phải thao tác thủ công mỗi kỳ.
- App cho thấy **phí giảm khi bạn bật các lớp an toàn**: ví dụ "bật chặn-rút-sạch → giảm
  X%", "có người thân bảo lãnh → giảm thêm". Càng an toàn, phí càng rẻ.
- Sau khi bật có **thời gian chờ** (đề xuất 14–30 ngày) mới có hiệu lực — để không ai
  "mua bảo hiểm sau khi đã bị trộm".

### 2. Báo mất (mở yêu cầu bồi hoàn)

- Khi phát hiện bị trộm, mở **yêu cầu bồi hoàn** trong hạn (đề xuất trong 30 ngày kể từ khi
  bị trộm).
- App **tự điền phần lớn thông tin** từ nhật ký trên chuỗi (bị rút bao nhiêu, khi nào, đi
  đâu) — bạn không phải tự chứng minh con số này, nó đọc thẳng từ chuỗi.
- Bạn chọn nguyên nhân sơ bộ và đính kèm bằng chứng (ảnh trang web lừa, tin nhắn...).
- App hiện **ước tính phần được đền** — đã **trừ sẵn** phần ví tự cứu được, để bạn thấy rõ
  "phần này an toàn rồi, không cần đền".

### 3. Nhận bồi hoàn

- Yêu cầu đi qua các bước: **xác minh nguyên nhân → hội đồng phân xử → thời gian chờ →
  chi trả**. App hiện timeline từng bước.
- Có **thời gian phản-tố** (đề xuất 7–14 ngày) trước khi tiền ra — để ai phát hiện gian lận
  kịp lên tiếng.
- Được duyệt thì tiền bồi hoàn về ví. Mỗi yêu cầu chỉ chi **một lần** (có cơ chế chống chi
  hai lần).
- Nếu về sau truy được tài sản từ kẻ trộm, phần thu hồi trả lại quỹ (bạn không được lời gấp
  đôi: vừa đền vừa tự lấy lại).

---

## Ẩn dụ: giống bảo hiểm như thế nào

Hãy hình dung Protectme như **bảo hiểm nhà** — nhưng có vài khác biệt quan trọng:

- **Đóng phí định kỳ** để được bảo vệ, giống đóng phí bảo hiểm.
- **Nhà càng chắc, phí càng rẻ** — lắp khoá tốt, camera, báo động thì phí giảm. Ở Protectme,
  bật chặn-rút-sạch và người thân bảo lãnh cũng làm phí giảm y vậy.
- **Chỉ đền phần thật sự mất** — nếu hàng xóm cứu được nửa số đồ trong đám cháy, bảo hiểm chỉ
  đền nửa còn lại. Protectme cũng chỉ đền phần ví không tự cứu được.
- **Bạn chịu một phần** (với sơ suất) — như "mức miễn thường" trong bảo hiểm: bạn gánh một
  phần để còn giữ ý thức cẩn thận, không ỷ lại.
- **Quỹ có hạn** — bảo hiểm không đền vô hạn. Sự cố cực lớn có thể chia theo tỷ lệ, không
  phải 100% tức thì cho tất cả.

---

## Ví dụ tình huống

**Tình huống 1 — Bị rút một phần rồi kịp khoá (sơ suất):**
Bạn lỡ dán chìa khoá vào một web giả. Kẻ trộm rút được một khoản trước khi bạn kịp bấm khoá
khẩn cấp. App đọc trên chuỗi: rời ví 500 (đơn vị quy đổi), bạn đã chặn kịp 300, còn **200
thực mất**. Vì là sơ suất, có mức miễn thường và bạn chịu một phần → được đền phần còn lại
của 200 theo trần loại "sơ suất". Phần 300 đã chặn kịp coi như an toàn, không đền.

**Tình huống 2 — Lỗi phần mềm (lỗi hệ thống):**
Một lỗ hổng trong phần mềm cho phép chi sai luật, khiến nhiều người cùng bị mất. Sau khi hội
đồng kỹ thuật xác nhận đúng là lỗ hổng, **tất cả người bị dính** được xử chung, phủ **mạnh**
(có thể tới toàn bộ phần mất, không phải chịu phần nào) — vì đây là trách nhiệm của hệ thống.

**Tình huống 3 — Ví đã tự cứu hết (không mất):**
Bạn mất điện thoại. Nhờ ví gắn liền danh tính, bạn khôi phục quyền và toàn bộ tài sản vẫn
còn nguyên trong ví. **Không có gì bị mất → Protectme không cần đền.** Đây là kết quả tốt
nhất: ba lớp an toàn đã lo trọn.

---

## Câu hỏi thường gặp (FAQ)

**Hỏi: Protectme có đền 100% mọi lúc không?**
Không hứa điều đó. Quỹ có hạn. Lỗi hệ thống được phủ mạnh; sơ suất được phủ có giới hạn. Sự
cố cực lớn có thể **chia theo tỷ lệ**. Ai hứa "đền 100% mọi lúc" là nói dối.

**Hỏi: Vì sao sơ suất của tôi lại được đền ít hơn lỗi hệ thống?**
Vì nếu đền đủ cho sơ suất, người ta sẽ hết cẩn thận ("có bảo hiểm rồi, sợ gì"). Bạn chịu một
phần để còn giữ ý thức tự bảo vệ. Lỗi hệ thống thì khác — bạn không tạo ra được lỗi đó, nên
không có chuyện ỷ lại, phủ mạnh là an toàn.

**Hỏi: Làm sao hệ thống biết tôi thực sự bị trộm, không phải tự dàn cảnh?**
Trên chuỗi biết chắc **mất bao nhiêu, đi đâu, khi nào**. Nhưng **"có phải trộm thật không"**
thì chuỗi không tự phân biệt được (chữ ký của bạn và của kẻ trộm dùng chìa khoá lộ trông
giống hệt nhau). Vì vậy có **hội đồng phân xử** người thật, thời gian phản-tố, và hình phạt
nặng cho gian lận. Đây là lý do phần "sơ suất" cần xét kỹ, không tự động chi.

**Hỏi: Phí bảo vệ đóng bằng gì?**
Phí được **tính theo MAGIC** (đơn vị đo sức mua) nhưng **thanh toán bằng CARP**, trừ tự động
qua hệ thống thanh toán Feecover. Bạn không phải lo phần kỹ thuật — App lo phí mạng hộ bạn.

**Hỏi: Ngừng đóng phí thì sao?**
Ngừng đóng quá thời gian ân hạn → mất bảo vệ ở kỳ kế tiếp. Sự cố xảy ra trong lúc đã mất
bảo vệ thì không được phủ.

**Hỏi: Tôi chỉ có một điện thoại, không có thiết bị dự phòng thì sao?**
Trường hợp này (ví dụ nông dân, hộ kinh doanh một người) cần **bật chặn-rút-sạch** để được
phủ phần "sơ suất" — vì đó là lớp bảo vệ thật duy nhất còn lại. Phần tài sản bạn đã "khoá
theo lịch" (cam kết không rút được) thì **tự an toàn**, không cần Protectme. Protectme chỉ
tính phí và phủ trên **phần tiền linh hoạt** có thể bị rút.

**Hỏi: Nếu tôi khai guardian nhưng thực ra cùng một chìa khoá gốc?**
Sẽ **không được giảm phí** cho lớp đó. Giảm phí chỉ áp khi lớp bảo vệ thực sự **độc lập**
(khác thiết bị / khác nguồn chìa khoá). Trả giảm cho một lớp bảo vệ "ảo" là không công bằng
với quỹ chung.

---

## Nói thật với bạn (ranh giới trung thực)

Đây là bản **đề xuất tính năng**, chưa phải sản phẩm hoàn chỉnh. Cụ thể:

- **CHƯA xây xong:** phần chi trả trên chuỗi (validator payout) **đang trong quá trình rà
  soát**. Con đường thu phí, hai quỹ, và cơ chế phân xử mới là bản thiết kế, chưa chạy thật.
- **Nhiều con số trong bài là ĐỀ XUẤT, chưa chốt:** các mốc thời gian (30 ngày báo mất,
  14–30 ngày chờ, 7–14 ngày phản-tố), mức phí, trần phủ, tỷ lệ bạn tự chịu — tất cả còn
  **chờ quyết định**.
- **11 quyết định lớn còn chờ chốt** (phần D của bản kỹ thuật), quan trọng nhất là:
  **ngưỡng phân biệt "lỗi hệ thống" và "sơ suất người dùng"**. Ranh giới này quyết định bạn
  được phủ mạnh hay hạn chế, và nó cũng là điểm dễ bị lợi dụng nhất — nên phải chốt rất kỹ.
- **Không hứa quá:** Protectme **không** đảm bảo "an toàn tuyệt đối" hay "đền đủ mọi lúc".
  Nó là lớp bảo vệ cuối, giúp bạn **không mất trắng** khi có sự cố ngoài tầm — nhưng có
  trần, có phân xử, có giới hạn quỹ. Đây là thiết kế có chủ đích, để hệ thống sống lâu và
  không khuyến khích cẩu thả.

Khi các quyết định trên được chốt, bản này sẽ được cập nhật với con số chính thức.
