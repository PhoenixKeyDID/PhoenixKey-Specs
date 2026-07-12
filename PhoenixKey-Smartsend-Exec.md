# PhoenixKey — Smartsend — Điều hành (Exec)

> **Module:** Smartsend (gửi có bảo vệ). **Loại doc:** Điều hành cho lãnh đạo. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** maintainer + lãnh đạo. Quyết định, rủi ro, lộ trình. Chi tiết ở [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md) / [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) / [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md).

---

## 1. Tóm tắt một trang

Module **Smartsend** là lớp **gửi có bảo vệ (opt-in)** của PhoenixKey: khoản gửi được khoá tạm vào một hòm ký quỹ có **cửa sổ-veto** — trong cửa sổ người gửi huỷ được (đa yếu tố), khoản lớn đòi người nhận đồng-ý, hết cửa sổ mới hoàn tất về người nhận. Điểm bán: **gửi nhầm không còn là mất trắng**, và **lộ khoá gửi giữa chừng còn cửa chặn** (huỷ đòi yếu tố khác gốc + guardian Freeze có lối thoát chống kẹt).

**Vị trí:** Smartsend là **module độc lập thứ 8** (chốt 2026-07-09). Trước đó nội dung nằm rải trong module Rebirthme; nay tách ra vì cơ chế đủ lớn và có novelty PPA riêng, dù vẫn **dùng chung hạ tầng** ví/guardian/anti-drain của Rebirthme.

**Ràng buộc phải nói thẳng:** chống trộm của Smartsend **phụ thuộc anti-drain** (`limit_meter.ak`, thuộc Rebirthme) làm nền — hai lớp bổ trợ nhau, Smartsend không phải lá chắn chống trộm đủ một mình tách rời khỏi nền đó (tiền đề (c), [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §6).

---

## 2. Bảng quyết định

| Quyết định | Lý do (4 trục: a dài hạn / b first-principles / c tối ưu / d user) | Đánh đổi |
|---|---|---|
| **Smartsend opt-in (không bắt cực chỉ-Phoenix)** | (a) giữ peg CARP + thanh khoản DEX; (b) không ép mọi giao dịch chậm lại; (c) chỉ áp lệnh user chọn; (d) ai không bật gửi thường một chữ ký | Chống trộm phụ thuộc anti-drain + guardian |
| **Tách Smartsend thành module độc lập thứ 8** | (a) cơ chế đủ lớn + novelty PPA riêng, dễ mở SDK; (b) escrow-veto khác trục với "giữ tiền" của ví; (c) doc gọn, ranh giới MECE rõ; (d) người đọc tìm đúng chỗ | Phải khai dependency rõ về Rebirthme (ví/guardian/anti-drain) — không nhân bản |
| **Huỷ đa yếu tố KHÁC gốc seed, neo anchor-enroll (SS-6/SSR-4)** | (b) cùng gốc seed = một điểm hỏng → huỷ vô nghĩa nếu kẻ trộm có luôn; tin cờ datum lúc Open thì kẻ Open (kẻ trộm) tự đặt factor dễ; (d) "bảo vệ" phải thật | Cần factor độc lập (guardian/ZK) + enforce ở builder |
| **Hoàn tất byte-perfect + escrow-tiêu một lần, min_ada tách riêng (SS-1/5′/7′/12)** | (b) chốt phải trả đúng người nhận đúng số kể cả phần ký quỹ ADA; (c) đóng double-satisfaction + rò minADA | Thêm field `min_ada`, ép biểu thức output nghiêm ngặt hơn |
| **ReclaimTimeout chống kẹt (SS-11)** | (b) khoản lớn không consent không được kẹt vĩnh viễn; (d) tiền luôn có đường về | Thêm `reclaim_deadline` vào datum |
| **Freeze có lối thoát bắt buộc: quorum hoặc timeout (SS-8′)** | (b) một guardian đơn lẻ không được phép khoá tiền người khác vô thời hạn; (d) tránh grief | Thêm `freeze_deadline` + redeemer `ResolveFreeze`, cần guardian-quorum m-of-n |
| **`window` có sàn cứng on-chain (SS-10)** | (b) "2-bên thoả" không được dùng để rút cửa sổ về 0 vô hiệu hoá veto; (d) bảo vệ tối thiểu áp cho mọi lệnh | `min_window_floor` cố định, chỉ nới rộng được không rút ngắn |
| **`fee_covered` chỉ là số audit, không vào biểu thức output (SS-12)** | (b) tách bạch ghi sổ phí khỏi conservation-tiền — trộn hai thứ tạo kẽ hở thao túng | Cần nguồn trả phí riêng (Feecover-vault) nếu escrow không tự đủ ADA |

---

## 3. Ma trận rủi ro

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| **Smartsend over-claim chống trộm** | 🟡 | Ghi rõ 3 tiền đề ([PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §6); chống trộm chỉ mạnh khi anti-drain + guardian sẵn sàng |
| **Phụ thuộc anti-drain của Rebirthme** | 🟡 (kế thừa) | `limit_meter.ak` là ưu tiên của module Rebirthme; Smartsend land sau hoặc song song; không tuyên chống trộm đủ tách rời nền đó |
| **Factor Cancel cùng gốc seed** | 🟡 | SS-6/SSR-4/I-CURVE-5 buộc khác gốc + neo anchor-enroll; cần enforce ở builder |
| **Kẹt khoản lớn (deadlock)** | ⚪ (đóng bằng thiết kế) | ReclaimTimeout (SS-11) hoàn sender khi quá hạn không consent |
| **Freeze grief (guardian khoá tiền vô hạn)** | ⚪ (đóng bằng thiết kế) | SS-8′: guardian-quorum m-of-n hoặc `freeze_deadline` auto-hoàn sender |
| **Factor bối cảnh giả (ảnh AI)** | 🟡 | Spectra liveness + Glint anti-replay ZK, bind escrow-ref (chờ VeData) |
| **Double-satisfaction / redirect / minADA rò** | ⚪ (đóng bằng thiết kế) | SS-7′ đếm input==1 + SS-5′/SS-12 ép Σ→receiver==amount+min_ada byte-perfect |
| **`window`=0 vô hiệu hoá veto qua "2-bên thoả"** | ⚪ (đóng bằng thiết kế) | SS-10 sàn cứng `min_window_floor`, chỉ nới rộng |

---

## 4. Lộ trình mốc

- **M1 (dựng):** `smartsend_escrow.ak` (7 đường: Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout) theo bất biến SS-1..12 + SSR-4 hợp nhất; builder Open/Cancel/Accept/Finalize/ResolveFreeze/ReclaimTimeout + UI công tắc; enforce I-CURVE-5 ở builder.
- **M2 (phụ thuộc):** chờ/đồng bộ anti-drain `limit_meter.ak` (Rebirthme) — nền chống trộm.
- **M3 (bối cảnh):** verifier Glint + Spectra factor bối cảnh (VeData) cắm vào `unlock_policy`, public-input bind escrow-ref.

---

## 5. Câu hỏi cần LÃNH ĐẠO chốt

1. **Vị trí module** — ✅ **ĐÃ CHỐT (2026-07-09): Smartsend là module độc lập thứ 8.** Lý do: Smartsend = gửi có bảo vệ (escrow + cửa sổ-veto + đa yếu tố) → **NGĂN** tổn thất TRƯỚC khi tiền đi; Protectme = bảo hiểm **PHỦ** tổn thất SAU (payout). Hai cơ chế khác trục (phòng ngừa vs bồi hoàn) → MECE. Smartsend **dùng chung** guardian-factor / anti-drain / `did_payment` của Rebirthme — khai dependency rõ ở [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md) §7 + [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §4.B, KHÔNG nhân bản. Novelty PPA riêng đủ mạnh để đứng thành module.
2. **Thứ tự với anti-drain** — Smartsend land sau khi `limit_meter.ak` xong, hay song song (chấp nhận chống trộm hở tạm tới khi anti-drain có)?
3. **`window` mặc định + `min_window_floor` + `reclaim_deadline`** — chốt giá trị (24/48/72h + sàn cứng + khoảng quá hạn)?
4. **`freeze_deadline`** — chốt khoảng thời gian tương đối lúc Freeze (vd 30 ngày) trước khi auto-hoàn sender?
5. **Factor bối cảnh ZK** — ưu tiên Glint (VeData) sớm hay để guardian-factor đủ cho bản đầu?

---

## Nguồn

Chi tiết: [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md), [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md), [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md).
Hạ tầng nền (dẫn chiếu): [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md), [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) (ví/guardian/anti-drain). Code: `PhoenixKey-Validator/validators/smartsend_escrow.ak`.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#smartsend)

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
