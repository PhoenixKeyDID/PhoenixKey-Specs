# PhoenixKey — Knowme: Kho giấy tờ của bạn, bạn giữ chìa

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **Knowme là gì**, **được lợi gì so với KYC tập trung**, **dùng thế nào an toàn**, bằng ngôn ngữ đời thường.
> Nội dung bám đặc tả [PhoenixKey-Knowme-Math.md](./PhoenixKey-Knowme-Math.md) + [PhoenixKey-Knowme-Tech.md](./PhoenixKey-Knowme-Tech.md) + [PhoenixKey-Knowme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-Knowme-Exec.md); code tham chiếu `PhoenixKey-Frontend/src/lib/sdvc/`.
> → Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## 1. Một câu là gì

**Knowme là kho danh tính tự chủ của bạn: bạn tự khai giấy tờ (họ tên, CCCD, mã số thuế…), đính kèm ảnh/tệp nếu cần, và khi ai đó hỏi thì bạn chỉ đưa ĐÚNG trường họ cần — phần còn lại vẫn khoá kín. Dữ liệu nằm ở máy bạn, chìa khoá trong tay bạn, không nằm ở một kho KYC tập trung nào để rò rỉ hàng loạt.**

Hình dung đơn giản:
- **Kho danh tính** = một tập "phiếu" bạn tự viết (mỗi trường một phiếu), bỏ trong tủ có khoá.
- **Xuất trình chọn lọc** = khi cần chứng "tôi là công dân VN", bạn rút đúng phiếu quốc tịch đưa ra, các phiếu khác vẫn trong tủ, người kia không thấy.
- **Chìa khoá** = khoá sinh ra từ vân tay của bạn (Secure Enclave). Không ai mở tủ thay bạn.

Knowme là module `-me` user-facing, đối xứng với **Protectme**: Protectme lo bảo vệ, Knowme lo **khai và xuất trình** danh tính.

---

## 2. Vấn đề nó giải

Với KYC truyền thống (tập trung), mỗi lần dùng dịch vụ bạn phải:

1. Nộp **trọn bộ hồ sơ** — ngày sinh, số giấy tờ, ảnh CCCD — dù bên kia chỉ cần biết bạn "đủ 18 tuổi".
2. Gửi bản sao giấy tờ cho **mỗi nơi**, mỗi nơi tự lưu một kho. Nhiều kho = nhiều chỗ rò rỉ.
3. Không kiểm soát được sau khi đã nộp: bên kia giữ dữ liệu bao lâu, chia sẻ cho ai, bạn không biết.

**Knowme lật lại ba cái đó:**
- **Chỉ đưa trường cần.** Cần "quốc tịch" thì đưa quốc tịch, ẩn ngày sinh + số giấy tờ. Phần không đưa vẫn là "phiếu úp" — bên kia không đọc được.
- **Không kho tập trung.** Giấy tờ sinh ra và nằm ở máy bạn. Server (nếu có) chỉ giữ **bản mã hoá** — không đọc được nội dung, không có gì để rò rỉ hàng loạt.
- **Bạn giữ chìa.** Mỗi lần chia sẻ là một hành động chủ động của bạn, có màn hình xác nhận liệt kê chính xác cái gì lộ. Tái dùng một hồ sơ ở nhiều nơi mà không phải xin lại cơ quan.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Tạo hồ sơ (tự khai)
Bạn mở Knowme, khai các trường danh tính: họ tên, năm sinh, CCCD, quốc tịch (với doanh nghiệp: tên công ty, mã số thuế, ngày cấp phép). Điện thoại **tự ký** tập khai này bằng khoá của bạn. Xong: bạn có một "thẻ danh tính tự khai" — chữ ký chứng minh "chính bạn đã khai".

> Đây là bước vào rẻ nhất, không chờ cơ quan. Về sau nếu cần độ tin cao hơn, một cơ quan có thể ký lên hồ sơ của bạn (Bước 5) mà không phải làm lại từ đầu.

### Bước 2 — (Tuỳ chọn) Đính giấy tờ
Nếu cần đính ảnh CCCD / hộ chiếu / giấy phép kinh-doanh: bạn chọn ảnh, Knowme **mã hoá ngay trên máy** rồi mới cất. Bản rõ không bao giờ rời máy dạng thường. Mỗi giấy tờ có một "dấu vân tay" riêng để đối chiếu khi tiết lộ chọn lọc.
→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

### Bước 3 — Ai đó hỏi, bạn chọn trường để đưa
Một cổng dịch vụ / đối tác gửi lời hỏi kèm một "mã phiên" riêng. Bạn thấy màn hình: *"Bên A muốn xem: [quốc tịch]. Bạn có đồng ý?"* Bạn tick đúng trường cần, bấm đồng-ý. Knowme tạo một "bằng xuất trình" chỉ chứa trường đó, khoá vào đúng bên A và đúng phiên đó.

### Bước 4 — Bên kia kiểm, không cần gọi cơ quan
Bên A nhận bằng xuất trình, tự kiểm: chữ ký của bạn đúng không, trường đó có thật thuộc hồ sơ đã ký không, có bị thu hồi chưa. Kiểm xong tại chỗ — **không cần gọi cơ quan cấp giấy**, không phải lưu dữ liệu của bạn.

### Bước 5 — (Nâng cấp) Cơ quan ký lên hồ sơ
Khi cần "đã xác minh" thật (không chỉ tự khai), một cơ quan/đối tác KYC (ví dụ Sở KH-ĐT, đối tác hộ chiếu) ký một trường **về bạn**. Từ đó bên kia biết "một cơ quan đáng tin đã chứng thực", nhưng bạn vẫn giữ bản hồ sơ — không phải xin lại mỗi lần.

### Bước 6 — Thu hồi khi cần
Bạn có thể thu hồi một lần chia sẻ (bên kia không đọc lại được), hoặc cơ quan có thể thu hồi credential đã cấp (mọi bằng xuất trình phái sinh từ nó thành vô hiệu).

> **Nói thẳng một giới hạn** (đọc kỹ ở §7 và §8): với **ảnh giấy tờ mắt người xem được**, một khi bên kia đã mở ảnh ra để kiểm, họ có thể chụp màn hình. "Thu hồi" chỉ chặn **đọc lại lần sau**, không lấy lại được cái họ đã thấy. Đường an toàn nhất: **đừng gửi ảnh nếu chỉ cần chứng một thuộc tính** — gửi "bằng chứng thuộc tính" thay ảnh (§4 dưới).

---

## 4. Ba mức riêng tư — kể bằng ẩn dụ "phiếu úp"

Knowme có ba mức, bạn chọn theo tình huống. Càng lên cao càng ít lộ.

### Mức 1 — Tự khai
Bạn tự viết phiếu, tự ký. Như tự khai tờ khai. Đủ cho cổng nhẹ (ví dụ đăng ký thành viên). Chữ ký chứng "bạn đã khai", chưa chứng "đúng sự thật".

### Mức 2 — Xuất trình chọn lọc
Cả tập phiếu được "úp" (mã hoá thành dấu vân tay không đọc ngược được). Khi cần, bạn **lật đúng một phiếu** cho người hỏi xem; các phiếu khác vẫn úp. Bên kia lật lại đúng phiếu đó, đối chiếu dấu vân tay là tin được — mà không thấy phiếu khác.
- *Ví von:* như đưa một quân bài ngửa trong bộ bài úp; người xem biết đúng quân đó thuộc bộ đã niêm phong, nhưng không thấy các quân còn lại.

### Mức 3 — Chứng thuộc tính, không lộ con số
Bạn chứng "đủ 18 tuổi" mà **không đưa ngày sinh** — người hỏi chỉ nhận một câu trả lời đúng/sai có bằng chứng mật mã. Đây là mức riêng tư nhất: đưa "kết luận" thay vì "dữ liệu gốc".
- *Ví von:* thay vì đưa giấy khai sinh để người ta tự tính tuổi, bạn đưa một con dấu "người này đủ 18" mà không nói sinh ngày nào.
- Nền là chuẩn **BBS+** của W3C [BBS-CRYPT] (tại thời điểm viết là bản Editor's Draft của W3C, chưa phải chuẩn Recommendation chính thức) — một loại chữ ký số đặc biệt cho phép ký một tập nhiều trường cùng lúc, rồi sau đó chứng minh riêng từng trường (hoặc một phép tính trên trường đó, như "tuổi ≥ 18") mà không phải lộ các trường còn lại hay lộ dữ liệu gốc.

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## 5. Các cải tiến — bảng cũ vs mới

| Điểm | KYC tập trung (cũ) | Knowme (mới) |
|---|---|---|
| **Dữ liệu ở đâu** | Kho của bên thứ ba, nhiều bản sao | Ở máy bạn; server chỉ giữ bản mã hoá |
| **Nộp bao nhiêu** | Trọn bộ hồ sơ mỗi nơi | Chỉ trường cần; phần còn lại vẫn ẩn |
| **Ai giữ chìa** | Bên thứ ba | Bạn (khoá từ vân tay) |
| **Tái dùng** | Xin lại giấy mỗi nơi | Một hồ sơ dùng nhiều nơi, không cần cơ quan online |
| **Chứng "đủ 18"** | Đưa cả ngày sinh | (Mức 3) đưa đúng một câu đúng/sai |
| **Thu hồi** | Dữ liệu đã nộp ngoài tầm bạn | Bạn/cơ quan chủ động rút (giới hạn ở ảnh — §7) |
| **Bên kiểm phải lưu gì** | Lưu PII của bạn → gánh tuân thủ | Không cần lưu PII → nhẹ tuân thủ |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Tạo hồ sơ danh tính ngay, không chờ cơ quan (Mức 1).
- Chỉ tiết lộ đúng trường cần, mỗi lần một quyết định chủ động của bạn (Mức 2).
- Giữ bản hồ sơ trong tay, tái dùng nhiều nơi mà không xin lại giấy.
- Thu hồi một lần chia sẻ hoặc để cơ quan thu hồi credential (với giới hạn ở ảnh — §7).
- Nhật ký "ai đã xem giấy tờ của mình" (qua cổng đọc Query — §8).

**Bạn cần:**
- Hiểu **tự khai ≠ đã xác minh.** Mức 1 chỉ chứng "bạn đã khai". Muốn độ tin cao thì cần cơ quan ký (Mức 5/issued).
- Hiểu **giới hạn thu hồi ảnh** (§7): ảnh đã cho xem thì không lấy lại được.
- Ưu tiên gửi **bằng chứng thuộc tính** thay vì ảnh khi có thể.

---

## 7. Câu hỏi thường gặp

**Hỏi: Dữ liệu của tôi có nằm trên server của hệ thống không?**
Đáp: Không nằm dạng rõ. Trường bạn khai nằm ở máy bạn. Nếu có đính ảnh/tệp, chúng được **mã hoá ngay trên máy** rồi mới cất; server chỉ giữ **bản mã hoá** (ciphertext) — không đọc được nội dung. Không có kho trung tâm chứa giấy tờ rõ để rò rỉ hàng loạt.

**Hỏi: "Tự khai" thì ai tin tôi?**
Đáp: Tự khai (Mức 1) chỉ chứng "chính bạn đã khai" — đủ cho cổng nhẹ. Khi cần tin thật, một cơ quan ký lên trường của bạn (issued) → chuyển thành "đã xác minh". Bên kiểm phân giải khoá cơ quan từ một danh sách tin cậy đã neo, không phải gọi cơ quan mỗi lần.

**Hỏi: Tôi chứng "đủ 18" mà không đưa ngày sinh được không?**
Đáp: Đó là **Mức 3** (chứng thuộc tính bằng ZK). Về nguyên lý được, và đây là hướng riêng tư nhất (§8). Bạn cũng có thể tiết lộ chọn lọc đúng một trường (Mức 2) — ví dụ chỉ "quốc tịch".

**Hỏi: Tôi chia sẻ ảnh CCCD rồi thu hồi, bên kia còn giữ được không?**
Đáp: **Có thể còn** — đây là điểm phải nói thẳng. Để tin ảnh là thật, bên kia phải mở ảnh ra xem; một khi đã xem, họ có thể chụp màn hình. Thu hồi chỉ chặn **đọc lại lần sau**, không xoá được bản họ đã thấy. Vì vậy: với thứ nhạy cảm, **đừng gửi ảnh nếu chỉ cần chứng một thuộc tính** — gửi bằng chứng thuộc tính. Nếu buộc phải gửi ảnh, Knowme sẽ nói rõ trên màn hình rằng "bạn không thu hồi được bản họ đã xem".

**Hỏi: Bên kiểm có phải gọi cơ quan mỗi lần không?**
Đáp: Không. Họ chỉ cần: khoá của bạn (từ hồ sơ DID), khoá cơ quan (từ danh sách tin cậy đã neo sẵn), và bằng xuất trình bạn đưa. Kiểm tại chỗ, không gọi mạng tới cơ quan.

**Hỏi: Knowme có tự chấm giấy tờ thật hay giả không?**
Đáp: Không tự chấm bằng con người. Với ảnh, có thể có một tín hiệu "media này là ảnh chụp thật hay bị dựng bằng AI" (từ Glint của VeData) — nhưng đó chỉ là **tín hiệu**, không phải phán quyết pháp lý. Kết luận "giấy tờ này có giá trị pháp lý" thuộc bên xác minh danh tính, không phải Knowme.

**Hỏi: Danh mục loại giấy tờ (CCCD, hộ chiếu…) do ai định?**
Đáp: PhoenixKey lo phần **"đóng gói + phân giải + tiết lộ chọn lọc"**. Danh mục các loại giấy tờ và cơ quan cấp (catalog VC + issuer) thuộc **VeData** — Knowme dẫn chiếu, không tự dựng.

---

## 8. Ranh giới trung thực — phạm vi từng lớp

Knowme vận hành theo hai lớp: **Mức 1 (tự khai) + Mức 2 (xuất trình chọn lọc)** là lớp lõi (`credential.ts`, `disclose.ts`), có demo tại màn `/vc`: khai → chọn trường → tiết lộ + niêm phong → kiểm → thu hồi. Đi kèm:
- **Neo DID chống mạo danh khi trình** (khoá người trình phân giải từ hồ sơ DID, không từ bằng xuất trình; có `aud`/`nonce` chống dùng lại sai chỗ).
- **Credential có cơ quan ký (issued)** — dựa danh sách tin cậy (Trust Registry).
- **Thu hồi** — cơ chế thu hồi theo lần chia sẻ + danh sách trạng thái của cơ quan.

**Phạm vi mở rộng (lớp tài liệu + Mức 3):**
- **Lớp tài liệu.** Nền: đính ảnh mã hoá lên kho phân tán + "dấu vân tay một-hash-mỗi giấy tờ" + neo bộ hồ sơ vào hồ sơ DID. Mở rộng: *tiết lộ chọn lọc cho tài liệu* (đưa đúng một ảnh trong khi các ảnh khác vẫn úp) + phiên bản bất biến + gửi lại riêng cho người nhận, màn "Giấy tờ của tôi".
- **Phiên bản hồ sơ bất biến (Strata).** Strata là một thư viện ngoài (tái dùng, không tự dựng lại) chuyên lưu các phiên bản kế tiếp của một hồ sơ theo kiểu "chỉ thêm, không sửa/xoá" — mỗi lần bạn cập nhật hồ sơ, bản cũ vẫn còn nguyên trong lịch sử, dùng để tra lại "hồ sơ tại thời điểm X trông thế nào".
- **Xuất trình lại chọn lọc tài liệu (re-seal cho người nhận).**
- **Mức 3 — chứng thuộc tính bằng ZK (BBS+), xuất trình không liên kết, neo ZK-anchor + xoá GDPR.**
- **Đọc qua cổng Query của VeData** (nhật ký "ai đã xem", cộng thêm nhiễu thống kê có kiểm soát vào số liệu tổng hợp — kỹ thuật gọi là *differential privacy* — để không ai suy ngược ra được một cá nhân cụ thể từ báo cáo tổng hợp).

Những gì **đã chắc** ở lớp lõi: bạn giữ chìa, dữ liệu ở máy bạn, tiết lộ chọn lọc từng trường, tái dùng nhiều nơi không cần cơ quan online — đó là Mức 1 + Mức 2.

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai): DocumentClaim/Strata/re-seal/anchor-qua-Stamp/Query — design; Mức 3 BBS+ ZK predicate — spec; lớp riêng tư on-chain — DRAFT.
- Code: `PhoenixKey-Frontend/src/lib/sdvc/{credential,disclose,commit,canonical,didDoc,didResolver,statusList,trust,trustList,crypto,lampnet,dossier,fingerprint,consent}.ts`; `src/app/verify/page.tsx`.
- Tài liệu cùng bộ: [PhoenixKey-Knowme-Math.md](./PhoenixKey-Knowme-Math.md), [PhoenixKey-Knowme-Tech.md](./PhoenixKey-Knowme-Tech.md), [PhoenixKey-Knowme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-Knowme-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
