# PhoenixKey — Anchorme: Danh tính của bạn, neo trên chuỗi

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản-phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu **danh tính lõi (Anchorme) là gì**, giải quyết gì, dùng ra sao, bằng ngôn ngữ đời thường.
> **Module:** Anchorme (danh-tính lõi). **Loại doc:** Feature (hướng người dùng/sản phẩm). **Ngày:** 2026-07-09.
> Đặc-tả toán ở [PhoenixKey-Anchorme-Math.md](./PhoenixKey-Anchorme-Math.md); kỹ-thuật ở [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md); điều-hành ở [PhoenixKey-Anchorme-Exec.md](./PhoenixKey-Anchorme-Exec.md).
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 1. Một câu là gì

**Danh tính lõi (Anchorme) là "chứng minh thư số" của bạn được neo thẳng lên blockchain Cardano: một danh tính (DID) mà chỉ điện thoại của bạn — qua vân tay và con chip an toàn — mới điều khiển được, không ai cấp phát, không ai thu hồi, và đổi được thiết bị mà vẫn giữ nguyên con người.**

Hình dung đơn giản:
- **DID** = số danh tính của bạn, dạng `did:phoenix:...`, độc nhất và vĩnh viễn.
- **Anchor (neo)** = một "con dấu" độc nhất (NFT) khóa danh tính đó vào đúng một ô trên chuỗi — không ai làm giả ô đó ở ví lạ.
- **Controller (người điều khiển)** = chiếc chìa khóa hiện tại của bạn. Mất máy, đổi máy → thay chìa mà số danh tính giữ nguyên.

---

## 2. Vấn đề nó giải

Danh tính số hôm nay có ba cái vướng:

1. **Phụ thuộc một công ty.** Tài khoản Google/Facebook do họ giữ; họ khóa là bạn mất. Danh tính không thật sự của bạn.
2. **Mất mật khẩu = mất tất cả.** Ví tiền mã hóa dựa 24 từ giấy: mất giấy là mất sạch, lộ giấy là bị cướp.
3. **Không phân biệt được người, tổ chức, dịch vụ, thiết bị.** Một khung phẳng không mô-hình-hóa được "công ty A sở hữu dịch vụ B, dịch vụ B vận hành thiết bị C".

**Danh tính lõi giải cả ba:**
- **Tự chủ thật.** Bạn tự tạo DID gốc bằng vân tay, không xin phép ai. Không có bên cấp phát nào thu hồi được.
- **Đổi chìa không mất người.** Khóa lộ hay mất máy → xoay chìa (rotation), số danh tính bất biến, mọi quan hệ đã neo giữ nguyên.
- **Một khung, nhiều loại.** Cá nhân, tổ chức, dịch vụ — cộng cả thiết bị, máy, tài sản, bot, AI — mỗi loại có luật quyền riêng, xếp thành cây sở hữu chứng minh được trên chuỗi.

---

## 3. Hành trình của bạn — từng bước bấm gì, thấy gì

### Bước 1 — Tạo danh tính bằng vân tay (GenesisPerson)
Bạn chạm vân tay. Điện thoại sinh khóa bảo mật ngay trong Secure Enclave (con chip an toàn của máy, khóa không rời được phần cứng). Một danh tính `did:phoenix:...` ra đời, kèm một "con dấu" (anchor NFT) khóa nó vào chuỗi. Không mật khẩu, không giấy 24-từ bắt buộc.

Màn hình báo: *"Danh tính của bạn đã được tạo và neo trên Cardano. Chỉ vân tay của bạn mở được."*

### Bước 2 — Dùng danh tính để đăng nhập, ký, chứng minh
Từ giờ bạn "Đăng nhập bằng PhoenixKey" vào các app trong hệ sinh thái (ProofChat, OriLife, AladinWork...). App hỏi chuỗi danh tính của bạn (resolve), thấy khóa công khai đã neo trên chuỗi, và tin chữ ký của bạn — **không cần tin máy chủ trung gian nào**.

### Bước 3 — Đổi thiết bị / xoay chìa (Rotate)
Đổi điện thoại, hoặc nghi khóa lộ? Bạn xoay chìa: khóa cũ ký một lần cuối để đặt khóa mới. Số danh tính `did:phoenix:...` **không đổi**. Mọi thứ neo dưới danh tính này — dịch vụ, thiết bị, quan hệ — theo bạn sang máy mới.

> Cơ chế khôi phục khi **mất hẳn máy** (guardian, backup tự động) thuộc module [Rebirthme](./PhoenixKey-Rebirthme-Vi-Feat.md) — xem tài liệu đó. Ở đây chỉ nói phần "xoay chìa khi bạn còn kiểm soát".

### Bước 4 — Tạo danh tính con dưới bạn (GenesisChild)
Bạn (hoặc tổ chức của bạn) đẻ ra danh tính con: một **dịch vụ** (app bạn viết), một **thiết bị** (máy chủ, cảm biến), một **tài sản** (mảnh vườn, lô hàng), một **bot/AI**. Danh tính con ghi cha là bạn — bất biến trên chuỗi. Đây là cách dựng "cây danh tính" của một cá nhân hay doanh nghiệp.

### Bước 5 — Đóng băng khi cần (Deactivate)
Danh tính bị lộ hoặc hết vòng đời? Bạn đóng băng nó (Deactivate). Từ đó không ai chi tiêu / lạm dụng danh tính đó được nữa — kể cả kẻ vừa cướp khóa.

---

## 4. Các cơ-chế lõi — kể bằng ẩn dụ đời thường

### Con dấu độc nhất (Anchor TAAD + State-NFT singleton)
Mỗi danh tính = đúng **một con dấu** (NFT) mang tên bằng đúng mã băm của DID. Con dấu này **bị khóa cứng vào địa chỉ của cỗ máy trạng thái** ngay lúc đúc — không thể mang con dấu đi cất ở ví lạ. Khác các hệ danh tính cũ ghi trạng thái vào một sổ chung có thể đọc nhập nhằng, ở đây mỗi danh tính là một ô riêng, một con dấu riêng, ai cũng đối chiếu thẳng trên chuỗi được, không cần tin máy chủ.

### Chìa hiện tại, không phải chìa cố định (Controller + Rotation)
Danh tính không dính vĩnh viễn vào một chiếc chìa. Con dấu ghi "chìa nào đang giữ quyền". Xoay chìa = đổi dòng ghi đó, số danh tính bất biến. Như đổi ổ khóa nhà mà địa chỉ nhà không đổi.

### Khóa gốc nằm trong két phần cứng (HW-key P-256 Secure Enclave)
Chìa gốc của bạn sinh ra và **không bao giờ rời con chip an toàn** (Secure Enclave của iPhone / StrongBox của Android). Khác mật khẩu hay giấy 24-từ, khóa này không xuất ra được → chống lừa đảo, chống keylogger ngay ở tầng gốc.

### Đọc danh tính "tại đúng thời điểm" (Point-in-time resolver)
Một chữ ký cũ được ký bằng chìa cũ, trước khi bạn xoay chìa. Hệ thống lưu lịch sử: "tại thời điểm ký, chìa nào hợp lệ?" — nên hợp đồng cũ vẫn kiểm tra đúng dù bạn đã đổi chìa nhiều lần. Không bao giờ sửa quá khứ.

### Mười loại danh tính, một cây sở hữu
Không phải danh tính nào cũng là người. Có **Người** (gốc tin cậy), **Tổ chức**, **Dịch vụ**, và các loại phi-người: **Thiết bị, Máy, Tài sản, Bot, AI (Agent), Ngữ cảnh, Nhân vật ảo**. Luật "ai đẻ được ai" (CanOwn) xếp chúng thành cây: Người/Tổ chức đẻ được mọi thứ trừ Người; Dịch vụ đẻ được thiết bị/tài sản/bot/AI/dịch-vụ-con...

---

## 5. Các cải tiến — bảng cũ vs mới

| Điểm | Danh tính truyền thống | Anchorme PhoenixKey |
|---|---|---|
| **Ai giữ danh tính** | Công ty (Google/Facebook) | **Bạn** — neo trên chuỗi, không ai thu hồi |
| **Mất thiết bị** | Mất tài khoản / phải cầu cứu tổng đài | **Xoay chìa / khôi phục** — số danh tính giữ nguyên |
| **Khóa gốc** | Mật khẩu / giấy 24-từ (lộ được) | **P-256 trong Secure Enclave** — không xuất được |
| **Kiểm chữ ký cũ** | Chỉ biết trạng thái hiện tại | **Point-in-time** — biết chìa nào hợp lệ tại lúc ký |
| **Phân loại thực thể** | Một loại phẳng | **10 loại** + cây sở hữu chứng minh được |
| **Bàn giao dịch vụ** | Tin nhau ngoài chuỗi | **Transfer 2-of-2** — ledger đảm bảo cả hai đồng thuận |
| **Chống làm giả danh tính** | Dựa máy chủ trung tâm | Con dấu khóa cứng vào chuỗi. Với danh tính Người, tính-độc-nhất của con dấu được đóng thêm bằng **PA2 UniquenessThread** (ép on-chain, không mint được hai con dấu cùng tên); **PA5-a entity-gate** thu hẹp bề mặt cho đường ví đi-theo-DID. Tổ chức/Dịch vụ vốn an toàn nhờ chữ ký cha (xem `PhoenixKey-Anchorme-Math.md`) |

---

## 6. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Một danh tính gốc `did:phoenix:...` của riêng bạn, miễn phí tạo, không ai thu hồi.
- Khóa gốc nằm trong két phần cứng — không xuất được, chống phishing tầng gốc.
- Đổi thiết bị / xoay chìa mà giữ nguyên con người và mọi quan hệ.
- Đẻ danh tính con (dịch vụ, thiết bị, tài sản) dưới bạn, xếp thành cây chứng minh được.
- Đóng băng danh tính bị lộ để chặn lạm dụng.

**Bạn cần:**
- Hiểu **danh tính Người là gốc tin cậy** — hãy giữ thiết bị và cấu hình khôi phục cẩn thận (chi tiết ở module [Rebirthme](./PhoenixKey-Rebirthme-Vi-Feat.md)).
- Hiểu **đóng băng ≠ xóa**: Deactivate làm danh tính không dùng được nữa, không "phục sinh" bằng khóa cũ.
- Với dịch vụ/tổ chức: nắm quan hệ cha-con (owner) — con thừa hưởng và bị giới hạn theo quyền của cha.

---

## 7. Câu hỏi thường gặp

**Hỏi: DID của tôi có bị công ty nào thu hồi được không?**
Đáp: Không. Danh tính Người là **tự-tạo** (self-genesis) — bạn ký sinh ra nó bằng vân tay, không có bên cấp phát. Không ai "khóa tài khoản" bạn được ở tầng danh tính.

**Hỏi: Đổi điện thoại thì mất danh tính không?**
Đáp: Không. Bạn **xoay chìa** (Rotate) sang máy mới; số danh tính giữ nguyên. Nếu mất hẳn máy cũ trước khi kịp xoay → có đường **khôi phục** (guardian / backup tự động), thuộc module [Rebirthme](./PhoenixKey-Rebirthme-Vi-Feat.md).

**Hỏi: Khóa gốc của tôi có bị hacker lấy được không?**
Đáp: Khóa gốc (P-256) sinh và nằm trong Secure Enclave — **không xuất ra được**, kể cả bạn. Hacker không copy khóa đi được. (Lưu ý: bảo vệ giá-trị-tài-sản lớn thuộc anti-drain ở module [Rebirthme](./PhoenixKey-Rebirthme-Vi-Feat.md) — xem đó.)

**Hỏi: "Anchor" / "con dấu NFT" để làm gì?**
Đáp: Nó khóa danh tính vào đúng một ô trên chuỗi, mang tên bằng mã băm DID, và bị ghim vào địa chỉ cỗ máy trạng thái. Nhờ vậy ai cũng đối chiếu danh tính thẳng trên chuỗi được, không cần tin máy chủ.

**Hỏi: Tôi làm app / dịch vụ thì tạo danh tính cho nó thế nào?**
Đáp: Bạn (hoặc OrgDID của bạn) đẻ **ServiceDID** con (GenesisChild). Sau này bàn giao được cho người khác qua **Transfer 2-of-2** (cả bên giao và bên nhận cùng ký). Đường tự-phục-vụ tạo ServiceDID qua API — chi tiết ở [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md).

**Hỏi: Chữ ký tôi ký năm ngoái, giờ tôi đổi chìa rồi, có còn kiểm được không?**
Đáp: Có — nhờ **point-in-time resolver**: hệ thống tra "tại thời điểm ký, chìa nào hợp lệ". Đặc tả đầy đủ ở [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md).

**Hỏi: Nghe nói danh tính Người còn một lỗ bảo mật?**
Đáp: Đúng, và chúng tôi nói thẳng. Khâu đúc con dấu của danh tính **Người** cần thêm lớp tính-độc-nhất on-chain (xem §8) — danh tính **Tổ chức/Dịch vụ** KHÔNG dính (có chữ ký cha xác thực).

---

## 8. Ranh giới thiết kế — luật cho danh tính Người

Nền tảng của Anchorme là: **anchor + state-NFT, xoay chìa, khóa phần cứng, 10 loại DID + cây sở hữu, Transfer/Deactivate** — đây là mô hình cố định, không đổi theo thời gian.

Riêng danh tính **Người** có một ràng buộc thiết kế bắt buộc: khâu đúc con dấu (mint) tự nó không kiểm được on-chain rằng "did-string thuộc về đúng người sở hữu" — vì khóa phần cứng P-256 chỉ được mang theo (carry), không verify bằng đường cong trên chuỗi. Nếu không có thêm lớp tính-độc-nhất, kẻ tấn công có thể đúc một con dấu cùng tên với danh tính nạn nhân, dùng chìa của mình, rồi chiếm quyền / rút tài sản do danh tính đó giữ. Danh tính **Tổ chức/Dịch vụ không dính luật này** — con dấu của chúng luôn đòi chữ ký của cha (owner) khi đúc.

**Luật thiết kế:** tính-độc-nhất của con dấu Người phải được ép on-chain bằng một cơ chế cấu-trúc riêng (**PA2 UniquenessThread**), không phải chỉ dựa vào khóa phần cứng. Song song, đường ví "đi-theo-DID" thu hẹp bề mặt tấn công bằng **PA5-a entity-gate** (giới hạn loại thực thể được chấp nhận làm anchor tham chiếu). Đặc tả đầy đủ hai cơ chế này ở [PhoenixKey-Anchorme-Math.md](./PhoenixKey-Anchorme-Math.md) §8-9 và [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md).

Các mảnh còn lại của hệ (resolve-by-hash, point-in-time resolver, DeviceDID, Authorization/Mint-Authority Registry, Permission & Consent) là đặc tả mở rộng thuộc ranh giới đội backend/on-chain — chi tiết ở [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md) §5-7.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai).
- `PhoenixKey-Specs/PhoenixKey-Math.md` §2–§5, §10, §12–§24.
- `PhoenixKey-Validator`: `validators/taad.ak`, `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak`.
