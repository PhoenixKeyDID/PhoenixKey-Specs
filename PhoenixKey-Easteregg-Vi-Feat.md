# PhoenixKey — Easteregg · Đặc-tả TÍNH-NĂNG (Feature, tiếng Việt)

> Module: **Easteregg** (biến-thể ẩn-danh của Ví Phượng-hoàng) · Loại doc: Feature (hướng người dùng/sản phẩm) · Ngày: 2026-07-09
> Đối tượng đọc: người dùng cuối + đội sản phẩm. Viết dễ hiểu, ít thuật ngữ.
> Tài-liệu cùng bộ: [PhoenixKey-Easteregg-Math.md](./PhoenixKey-Easteregg-Math.md) (toán) ·
> [PhoenixKey-Easteregg-Tech.md](./PhoenixKey-Easteregg-Tech.md) (kỹ thuật) ·
> [PhoenixKey-Easteregg-Exec.md](./PhoenixKey-Easteregg-Exec.md) (điều-hành).
>
> **Quyết định maintainer (2026-07-09):** ví CHỈ có **2 LOẠI** — `Phoenix` và `Standard` (xem
> `PhoenixKey-Rebirthme-Vi-Feat.md §4.2`). **Easteregg KHÔNG phải một loại ví thứ ba** — nó là
> **các biến-thể ẩn-danh của ví Phoenix**, chọn theo mức riêng-tư. Mô hình đúng là **2 tầng**:
> **loại ví (Phoenix / Standard) × chế-độ riêng-tư (công-khai / ẩn-danh / vào-kho)**. Tài-liệu
> này mô tả các chế-độ riêng-tư mà ví Phoenix có thể mang — KHÔNG phải một sản-phẩm ví riêng.
>
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 1. Một câu là gì

**Easteregg là các CHẾ-ĐỘ RIÊNG-TƯ ẩn-danh mà ví Phượng-hoàng (Phoenix) của bạn có thể mang —
số dư được che khỏi con mắt tò mò trên chuỗi bằng mật-mã (không phải giấu-giếm mập-mờ), nhưng
tài-sản vẫn thuộc về CÙNG một ví Phoenix và tự đi theo bạn khi bạn khôi phục danh-tính.**

Tên gọi có nghĩa: *Easter* = kho cất an toàn, *egg* = quả trứng tài-sản. Đối xứng với
PhoenixKey: **PhoenixKey khôi phục KHOÁ · Easteregg khôi phục TÀI-SẢN của khoá** — quả trứng
luôn thuộc về phượng hoàng. **Nói cách khác: Easteregg không phải "một ví khác" — nó là ví
Phoenix của bạn, bật thêm lớp ẩn-danh.**

---

## 2. Vấn đề nó giải

Trên Cardano mọi thứ công khai: ai mở explorer lên cũng thấy **một địa chỉ có bao nhiêu tiền,
nhận từ đâu, tiêu đi đâu**. Với người thường thì phiền; với doanh nghiệp thì đó là **rò rỉ bí
mật kinh doanh** — đối thủ đọc được doanh thu, đối tác biết được số dư, tất cả chỉ bằng một cú
tra cứu.

Easteregg che phần đó lại — nhưng **không phải bằng cách giấu-giếm mập-mờ**, mà bằng **bằng
chứng toán học**: số dư của bạn được khoá trong một "hồ chung", trên chuỗi chỉ thấy tổng của
cả hồ, không tách được phần của riêng ai. Khi cần (ví dụ ngân hàng hay kiểm toán yêu cầu), bạn
**tự mở cho họ xem đúng phần của mình** — không phải phơi hết cho cả thiên hạ.

---

## 3. Bốn mức ẩn-danh — bạn chọn mức riêng-tư ví Phoenix của mình cần

**Nhắc lại mô hình:** ví của bạn chỉ có 2 LOẠI — Phoenix hoặc Standard (xem `PhoenixKey-Rebirthme-Vi-Feat.md §4.2`).
Easteregg KHÔNG phải loại ví thứ ba — đây là **bốn mức ẩn-danh mà ví Phoenix của bạn có thể
bật**, bạn chọn mức nào tùy nhu cầu — như chọn ổ khoá cho cửa nhà: nhà thường một khoá, két sắt
thì nhiều lớp. Đúng mô hình: **loại ví (Phoenix) × mức riêng-tư (dưới đây)**.

| Mức riêng-tư | Che cái gì | Chi phí | Ai nên dùng |
|---|---|---|---|
| **Địa chỉ riêng** *(= địa chỉ L3, `did_subaddr`, tầng riêng-tư của ví Phoenix — xem `PhoenixKey-Rebirthme-Vi-Feat.md §4.3`)* | Người ngoài không nhóm được các địa chỉ nhận tiền của bạn về cùng một chủ | Gần như miễn phí | Ai muốn tách các nguồn thu |
| **Kho ẩn số dư** *(mặc định — vào kho did_pool)* | Explorer không đọc được **số dư** của bạn | **Cận 0** — rút chỉ cần 1 chữ ký | Mọi người, nhất là doanh nghiệp |
| **Ẩn dòng tiền** *(tùy chọn, lớp ZK)* | Người ngoài không ghép được "tiền vào" với "tiền ra" | Cao hơn (dùng bằng chứng ZK) | Ai cần riêng-tư dòng tiền tối đa |
| **Mở cho kiểm toán** | Cho phép ngân hàng/kiểm toán xem đúng phần của bạn theo yêu cầu | Thấp | Doanh nghiệp cần minh bạch có kiểm soát |

> **Ghi chú thuật ngữ:** bản này trước đây gọi các mức trên là "Tầng 0/1/2/3" của một "kho
> Easteregg" độc lập. Đã reframe (2026-07-09): đây là **các mức riêng-tư của ví Phoenix**, không
> phải tầng của một loại ví riêng. Tên kỹ-thuật giữ nguyên (`did_subaddr`, `did_pool`) — chỉ đổi
> cách trình-bày phân-loại. Xem chi-tiết đối-chiếu ở [PhoenixKey-Easteregg-Math.md §1](./PhoenixKey-Easteregg-Math.md#1-ký-hiệu-notation)
> · kiến-trúc 4 tầng (ai build gì) ở [PhoenixKey-Easteregg-Tech.md §1](./PhoenixKey-Easteregg-Tech.md#1-kiến-trúc--sơ-đồ-thành-phần-ai-build-gì)
> · bảng quyết-định ở [PhoenixKey-Easteregg-Exec.md §2](./PhoenixKey-Easteregg-Exec.md#2-bảng-quyết-định-quyết--lý-do-4-trục--đánh-đổi).

**Điểm mấu chốt: Tầng 1 là mặc định** vì nó đạt cả hai điều quan trọng nhất — **chi phí gần
như bằng 0** và **tài sản tự đi theo khoá** khi bạn khôi phục danh-tính. Các tầng cao hơn đổi
thêm chi phí (và khôi phục khó hơn) lấy riêng-tư nhiều hơn — chỉ bật khi bạn thật sự cần.

### 3.1 Bạn chọn "ẩn gì, với ai, giá bao nhiêu" — không chọn công nghệ

Bạn không cần biết "ZK" hay "Merkle-Sum" là gì. App hỏi bạn bằng ngôn ngữ nhu cầu ("Ai được
thấy số dư của bạn?") rồi tự dịch sang tầng thực-thi. Có 5 mức (Privacy-Mode) tương ứng 4 tầng:

| Mức (tên bạn thấy) | Ẩn gì | Che với ai | Chi phí | Khôi phục |
|---|---|---|---|---|
| **M0 — Công khai** | không gì (cố ý lộ danh-tính) | — | 0 | theo DID |
| **M1 — Kín số dư** *(mặc định)* | số dư | công chúng / explorer | ~0 (1 chữ ký/rút) | theo DID, tự động |
| **M1+ — Kín nguồn thu** | + liên kết các địa chỉ nhận | công chúng + đối tác | ~0 (+1 lần lập địa chỉ) | theo DID |
| **M2 — Kín dòng tiền** *(opt-in)* | + ghép cặp vào↔ra + số dư (cả với người vận hành) | công chúng + **người vận hành** | cao hơn (ZK) | bạn CHỌN: tự-giữ / nhờ-nhiều-người |
| **M3 — Vô hình** | + cả sự tồn tại giao dịch | mọi bên trên L1 | phí bên-thứ-3 | theo điều khoản adapter |

3 preset gợi ý: *Cá nhân* (M1) · *Doanh nghiệp* (M1+ + mở sẵn cho kiểm toán) · *Tối đa* (M2).

> **M3 "vô hình" hiện KHÔNG cung cấp trong hệ.** Nó cần một cầu nối sang mạng khác (Midnight
> hoặc tương đương) mà với công nghệ 2026 vẫn tạo ra một điểm "biết hết" mới — chỉ dịch niềm
> tin, chưa xoá. Vì thế M3 để ngủ-đông, chỉ mở khi có cầu nối đáng tin thật.

---

## 4. Che cái gì — KHÔNG che cái gì (phần quan trọng nhất)

### Easteregg CHE (Tầng 1, mặc định)
- **Số dư của bạn** — explorer chỉ thấy tổng cả hồ, không tách được phần riêng.
- **Với người ngoài** — không ai dò được bạn đang giữ bao nhiêu.

### Easteregg KHÔNG che (ở Tầng 1)
- **Lúc bạn nhận tiền** — mỗi khoản chảy vào địa chỉ công khai của bạn **vẫn lộ** (bao nhiêu,
  khi nào) trước khi được "quét" vào hồ. Muốn che cả cái này → cần **Tầng 2**.
- **Với người vận hành hệ thống** — máy chủ giữ sổ (indexer) vẫn biết số dư thật của bạn để
  dựng bằng chứng. Riêng-tư ở đây là **với explorer công khai, không phải với người vận hành**.
  Muốn giấu cả người vận hành → cần **Tầng 2**.

> **Nói thẳng:** Tầng 1 là "che sổ kế toán bằng mật mã" — ẩn *số dư đang có*, không ẩn *toàn bộ
> dòng chảy*. Đây là giới hạn vật lý của một chuỗi công khai, không phải lỗi. Ai cần ẩn cả dòng
> tiền thì bật Tầng 2 (trả thêm chi phí bằng chứng ZK). **Ngay cả Tầng 2 cũng chỉ ẩn ghép-cặp
> vào-ra và số dư — vẫn lộ ai nạp, ai nhận, mỗi khoản bao nhiêu.** Ai nói "vô hình tuyệt đối"
> là nói dối.

---

## 5. Hành trình của bạn — từng bước

### 1. Nhận tiền như bình thường
Bạn vẫn có một **địa chỉ nhận cố định gắn với danh-tính**. Khách/đối tác trả tiền vào đó như
mọi ví khác. Không cần họ biết gì về Easteregg.

### 2. Tiền tự "vào kho"
Hệ thống định kỳ **quét** tiền từ địa chỉ nhận vào **hồ chung** (bạn không phải bấm gì). Sau
bước này, explorer nhìn vào địa chỉ của bạn thấy ≈ 0, còn tiền nằm an toàn trong hồ — lẫn với
tiền của nhiều người khác, không tách ra được.

### 3. Xem số dư của mình
Trong app, bạn thấy đúng số dư của mình (kèm bằng chứng đối chiếu với dữ liệu trên chuỗi).
Người ngoài mở explorer thì **không** thấy con số này.

### 4. Rút tiền ra
Khi cần tiêu, bạn **ký một chữ ký** (bằng danh-tính của bạn), hệ thống kiểm tra bằng chứng rồi
chi đúng số tiền ra địa chỉ bạn muốn. Rút ở Tầng 1 **rất rẻ** — không cần bằng chứng ZK nặng nề.

### 5. Chuyển nội bộ cho người khác trong kho *(đề xuất — G2)*
Nếu người nhận cũng dùng Easteregg cùng hồ, bạn có thể chuyển thẳng cho họ mà không phải rút ra
rồi nạp lại (2 lần lộ). *Tính năng này còn ở dạng đề-xuất — xem mục "Nói thật".*

### 6. Mở cho ngân hàng/kiểm toán (khi cần)
Bạn cấp một **"chìa khoá xem"** cho ngân hàng — họ thấy **đúng phần của bạn** (số dư + lịch sử),
đối chiếu được với chuỗi, nhưng **không thấy của người khác** trong hồ. Bạn có thể **cắt quyền
xem về sau** (họ không xem được phần tương lai nữa). "Chìa khoá xem" CHỈ để xem, không tiêu được
tiền của bạn.

---

## 6. Chuyển giữa các mức — an toàn, luôn do bạn quyết

- **M0 ↔ M1:** gửi vào / rút khỏi hồ — giao dịch thường, 1 chữ ký.
- **M1 → M1+:** tạo địa chỉ nhận mới cho nguồn thu mới (địa chỉ cũ giữ nguyên).
- **M1 → M2 (nâng):** rút ở Tầng 1 rồi nạp vào hồ ZK — app nêu rõ hệ quả trước khi bạn ký.
- **M2 → M1 (hạ):** rút ZK về địa chỉ đích — bạn hiểu đây là **một lần lộ** (đích rút lộ).

**Hằng số không đổi ở mọi mức:** không bên nào ngoài bạn (hoặc nhóm khôi phục hợp lệ của bạn)
rút được tài sản. Chuyển mức chỉ đổi *mức ẩn*, không bao giờ đổi *quyền sở hữu*.

---

## 7. Khôi phục — vì sao Tầng 1 là mặc định

Đây là điểm Easteregg khác các ví ẩn-danh thông thường:

- **Tầng 1 (mặc định):** tài sản trong hồ do **danh-tính của bạn** kiểm soát. Khi bạn mất điện
  thoại và khôi phục danh-tính (qua cơ chế PhoenixKey — người thân bảo lãnh, v.v.), **quyền chi
  tiền trong hồ tự đi theo bạn**. Bạn **không phải giữ thêm bí mật nào** ngoài chính danh-tính
  đã có. Tài sản khôi phục cùng khoá. *(Điều kiện: cần vá G5 — hiện dữ-liệu-chứng phụ (salt) còn
  lưu ngoài chuỗi, mất nó thì tiền còn trong hồ nhưng tạm kẹt; cách vá là sinh lại salt từ khoá
  gốc của bạn. Xem mục "Nói thật".)*

- **Tầng 2 (tùy chọn, ẩn dòng tiền):** đây là cái giá của riêng-tư tối đa — để không ai (kể cả
  PhoenixKey) nhìn thấy dòng tiền, thì **không ai được có "cửa sau" khôi phục hộ bạn**. Bạn phải
  chọn: **tự giữ bí mật** (mất là mất luôn) hoặc **nhờ nhiều người giữ chung** (cần đủ số người
  đồng ý + chờ một thời gian mới khôi phục được). App sẽ **nói rõ điều này trước** khi bạn bật
  Tầng 2 — không giấu.

---

## 8. Quyền lợi và nghĩa vụ của bạn

**Quyền lợi:** ẩn số dư có bằng chứng · tài sản đi theo khoá khi khôi phục (Tầng 1) · tự chọn
ai được xem phần của mình · rút rẻ (Tầng 1 không ZK).

**Nghĩa vụ:** rút là một giao dịch thật trên Cardano — tốn phí mạng thật (được cấn trừ từ chính
tài sản bạn rút, bạn không cần giữ sẵn ADA lẻ — *cơ chế cấn phí còn đang chốt*). Nếu bật Tầng 2
và chọn tự-giữ bí mật, mất bí mật là mất tiền — hãy đọc kỹ cảnh báo trước khi bật.

---

## 9. Câu hỏi thường gặp (FAQ)

**Hỏi: Easteregg có phải một đồng coin mới không?**
Không. Nó là **cách cất giữ** tài sản bạn đã có (CARP, LAMP, ADA). Không có token mới.

**Hỏi: Tiền trong hồ chung, làm sao chắc phần của tôi không bị lấy mất?**
*Trước hết, nói thẳng: các bảo đảm dưới đây là **thiết-kế**, chưa phải điều đang chạy — cái
"khoá cứng" cưỡng-chế chúng (validator on-chain) CHƯA được xây (xem mục "Nói thật").* Khi xây xong,
theo thiết-kế: người vận hành hồ **sẽ không rút được** tiền của bạn — mọi lần rút đều cần **chữ ký
của chính bạn**. Họ cũng **không thể "in tiền ảo" hay giấu bớt tổng** — có một quy tắc toán học
buộc tổng trên sổ luôn khớp tổng tiền thật khoá trong hồ. Tệ nhất họ gây tranh chấp về phân bổ,
nhưng **tiền không mất** và bạn phát hiện được ngay. Cho tới khi validator được kiểm-thử xong, đừng
gửi tài sản thật vào Easteregg.

**Hỏi: Ai vận hành hồ có biết số dư của tôi không?**
Ở Tầng 1: **có** — máy chủ giữ sổ biết, để dựng bằng chứng cho bạn. Riêng-tư ở Tầng 1 là với
*explorer công khai*. Muốn giấu cả người vận hành → Tầng 2.

**Hỏi: Nếu tôi mất điện thoại thì tiền trong kho có mất không?**
Ở **Tầng 1: không** — khôi phục danh-tính là lấy lại được quyền chi. Ở **Tầng 2**: tùy bạn đã
chọn cách giữ bí mật nào (tự giữ hay nhờ nhiều người) — app cảnh báo trước khi bạn bật.

**Hỏi: Rút tiền có tốn phí không?**
Có — rút là một giao dịch thật trên Cardano, tốn phí mạng thật. Phí này được **cấn trừ từ chính
tài sản bạn rút** (bạn không cần giữ sẵn ADA lẻ). *Cơ chế cấn phí này còn đang chờ chốt — xem
mục "Nói thật".*

**Hỏi: Easteregg có che được HOÀN TOÀN mọi thứ không?**
**Không, và chúng tôi không hứa điều đó.** Tầng 1 ẩn số dư. Ngay cả Tầng 2 cũng chỉ ẩn *ghép cặp
vào-ra và số dư*, chứ **vẫn lộ** ai nạp, ai nhận, và mỗi khoản bao nhiêu. Ai nói "vô hình tuyệt
đối" là nói dối.

---

## 10. Nói thật với bạn (ranh giới trung thực — nguyên tắc thiết-kế)

- **Nguyên tắc chống-gian-lận.** Validator giữ tiền (`did_pool` — mức "kho ẩn số dư") và validator
  địa-chỉ-riêng (`did_subaddr` — địa chỉ L3 của ví Phoenix, xem §3) phải cưỡng-chế đủ 9 bất-biến
  chống-gian-lận (SHIELD-1..9) trước khi nhận tài sản thật. `did_pool` KHÔNG phải một "loại địa
  chỉ" — nó là pool kế-toán dùng chung (Merkle-Sum-Tree); tiền phải "quét" (sweep) từ địa-chỉ
  Phoenix công-khai vào, KHÔNG nhận trực-tiếp.

- **5 lỗ thiết-kế cần vá trước khi mở-khoá từng mức** (chi-tiết kỹ-thuật ở bản Tech):
  - **G1 — Cấn phí khi rút (🔴):** bạn rút được mà không cần giữ ADA lẻ, và không ai lách phí.
  - **G2 — Chuyển nội bộ (🟡):** cách chuyển thẳng trong hồ thay vì rút-ra-nạp-lại.
  - **G3 — Chống ghi lệch khi gom tiền (🔴):** bước "quét tiền vào hồ" phải bảo đảm ghi đúng phần
    cho đúng người. Đây là **điều kiện bắt buộc** khi xây, không phải tuỳ chọn.
  - **G4 — Thu hồi ADA lẻ vận hành (🟡):** tránh ADA lẻ kẹt dần trong hồ.
  - **G5 — Khôi phục dữ-liệu-chứng (🟡):** dữ-liệu-chứng phụ (salt) phải sinh lại được từ chính
    khoá gốc của bạn — không phụ thuộc một bản sao off-chain duy nhất.

- **Tầng 2 (ẩn dòng tiền) cần một buổi "lễ thiết lập" bảo mật** + kiểm toán mạch bằng chứng trước
  khi bật cho người dùng thật.

- **Không hứa quá:** Easteregg giúp bạn **giữ kín số dư** một cách có bằng chứng và **không mất
  tài sản khi khôi phục** — nhưng nó **không** làm bạn "vô hình" trên chuỗi. Đây là thiết-kế
  trung thực với giới hạn vật lý của một blockchain công khai.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## Nguồn

- Gộp từ nguồn thiết-kế nội-bộ (không công khai):
  `PhoenixKey-Easteregg-Gaps-Addendum-Feat.md` (G1–G5),
  `PhoenixKey-Easteregg-PrivacyModes-Addendum-Feat.md` (Mode Matrix M0–M3 + mode-transition),
  `PhoenixKey-Shielded-Custody-Feat-Math.md` (Tầng 1), `Easteregg/Easteregg-Unify-Architecture.md`.
