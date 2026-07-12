# PhoenixKey — Đặc tả kỹ thuật (Specifications)

> Hạ tầng **danh tính phi tập trung (DID) và quản lý tài sản** trên Cardano — loại bỏ việc phải ghi nhớ cụm từ khôi phục (seed phrase) nhờ khoá sinh trong phần cứng an toàn (Secure Enclave) và khôi phục theo tầng.
>
> **Mục tiêu dài hạn:** một Open SDK cho mọi đội xây dựng trên Cardano.

Kho này chứa **đặc tả** (specification) của PhoenixKey — tài liệu định nghĩa hệ thống ở mức toán học, kỹ thuật, sản phẩm và điều hành. Đây KHÔNG phải mã nguồn; mã on-chain (Aiken/PlutusV3) và backend nằm ở các kho riêng.

---

## 1. Bắt đầu từ đâu

| Bạn là… | Đọc theo thứ tự |
|---|---|
| **Người mới / muốn toàn cảnh** | **[`PhoenixKey-Whitepaper.md`](./PhoenixKey-Whitepaper.md)** (ngôn ngữ đơn giản + sơ đồ, hiểu toàn hệ) → bấm link tới từng `-Vi-Feat.md` khi muốn xem sâu |
| **Nhà đầu tư / đánh giá dự án** | Các file `-Exec.md` (điều hành: quyết định, đánh đổi, lộ trình) — **nội bộ**, nằm ở repo private per-module (`PhoenixKey-<Module>-Specs`), liên hệ team để cấp quyền đọc → `README` mục 2 (bản đồ module) |
| **Người dùng / đội sản phẩm** | Các file `-Vi-Feat.md` (ngôn ngữ đời thường, hành trình người dùng) |
| **Kỹ sư tích hợp** | `-Tech.md` (kiến trúc, API, datum, luồng) → `-Math.md` khi cần bất biến chính xác |
| **Nhà khoa học / auditor** | `PhoenixKey-Math.md` (đặc tả toán học tổng, v4.6) → `<Module>-Math.md` từng module |
| **Muốn biết đang xây tới đâu** | **[`PhoenixKey-STATUS.md`](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md)** (hiện trạng, blocker, bằng chứng test — tách khỏi spec) |

---

## 2. Bản đồ 8 module

PhoenixKey chia thành 8 module. Mỗi module có bộ 3 tài liệu public (xem mục 3) + 1 tài liệu Exec nội bộ.

| Module | Một câu | Vai trò |
|---|---|---|
| **Anchorme** | Danh tính của bạn, neo thẳng trên chuỗi (`did:phoenix:…`), chỉ vân tay + Secure Enclave điều khiển. | Tầng lõi (Layer‑0): DID, anchor NFT, controller, xoay chìa (rotation), cây sở hữu. |
| **Rebirthme** | Chiếc ví đi theo bạn, không đi theo chìa khoá. | Ví đa địa chỉ theo DID + khôi phục theo tầng (guardian / phương thức đa dạng). |
| **Smartsend** | Gửi có bảo vệ — nút "hoàn tác" cho giao dịch. | Escrow cửa sổ‑veto + đa yếu tố, opt‑in: huỷ được khi gửi nhầm, chống trộm khi lộ khoá. Tái dùng ví/guardian/anti‑drain của Rebirthme. |
| **Wakeme** | Chiếc Đèn và Điều ước của bạn. | Kích hoạt / vesting → sinh MAGIC → tiêu dùng dịch vụ trong hệ sinh thái. |
| **Feecover** | Lớp phủ phí giao dịch. | Trừu tượng hoá phí (fee abstraction) — hạ tầng, người dùng không tự lo ADA. |
| **Protectme** | Lớp bồi hoàn cuối, phủ phần bị mất. | Bảo vệ / bồi hoàn tuỳ chọn. |
| **Knowme** | Kho giấy tờ của bạn, bạn giữ chìa. | Chứng chỉ có thể kiểm chứng (VC) + tiết lộ chọn lọc (selective disclosure). |
| **Easteregg** | Lớp trải nghiệm / quyền riêng tư mở rộng. | Chế độ riêng tư và tính năng bổ trợ. |

Ngoài 8 module: **`PhoenixKey-Math.md`** là đặc tả toán học tổng (v4.6, ~165 KB) — nguồn chuẩn (normative) cho phân cấp khoá, TAAD anchor, danh mục loại DID, và các bất biến toàn hệ.

---

## 3. Bốn loại tài liệu cho mỗi module

Mỗi module có 3 file public theo khuôn `PhoenixKey-<Module>-<Loại>.md`, cộng 1 file `-Exec.md` nội bộ:

| Loại | Tên file | Viết cho ai | Nội dung |
|---|---|---|---|
| **Vi-Feat** | `-Vi-Feat.md` | Người dùng, đội sản phẩm | Tính năng bằng ngôn ngữ đời thường: là gì, giải quyết gì, hành trình người dùng. |
| **Math** | `-Math.md` | Nhà khoa học, auditor | Đặc tả toán học: ký hiệu, định lý, bất biến (invariant), chứng minh. |
| **Tech** | `-Tech.md` | Kỹ sư tích hợp | Kiến trúc, API, cấu trúc dữ liệu (datum), luồng giao dịch. |
| **Exec** | `-Exec.md` | Lãnh đạo, người ra quyết định | Quyết định + lý do + đánh đổi + rủi ro + lộ trình triển khai — đủ cụ thể để đội dev thực thi. Không lặp toán. **Nội bộ — nằm ở repo private `PhoenixKey-<Module>-Specs`, không public.** Hiện trạng tách sang `PhoenixKey-STATUS.md` (cũng nội bộ, xem mục 4). |

**Kho công khai (repo này):** 8 module × 3 loại (Vi-Feat/Math/Tech) = **24 file** + 5 tài liệu cấp kho ([`Whitepaper`](./PhoenixKey-Whitepaper.md) tổng quan · `PhoenixKey-Math.md` toán tổng · [`PhoenixKey-DIDMethod-W3C.md`](./PhoenixKey-DIDMethod-W3C.md) chuẩn DID · [`PhoenixKey-SeedDistribution-Tech-Math.md`](./PhoenixKey-SeedDistribution-Tech-Math.md) phân tán seed/khoá · [`PhoenixKey-Wallet-API-v2-Feat.md`](./PhoenixKey-Wallet-API-v2-Feat.md) API ví) = **29 tài liệu spec** (+ `README`/`LICENSE`).
**Kho nội bộ (8 repo private `PhoenixKey-<Module>-Specs`):** 8 file `-Exec.md` + `PhoenixKey-STATUS.md` (đặt ở `PhoenixKey-Anchorme-Specs`, cross-module) + các tài liệu đề xuất/draft chưa duyệt (`spec-proposals/`, hiện đặt phẳng trong từng repo module tương ứng).

---

## 4. Spec là kim chỉ nam; hiện trạng ở STATUS

Bộ đặc tả này mô tả **hệ thống đích** — thiết kế mà các đội dev xây tới, đủ cụ thể để nhiều dev độc lập biết chính xác làm gì. **Không** trộn hiện trạng vào spec.

- **Hiện trạng & tiến độ** (đã build / chặn bởi ai / bằng chứng test): xem **[`PhoenixKey-STATUS.md`](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md)** — cập nhật riêng, không nằm rải rác trong spec.
- Mỗi mục spec liên quan có dòng "→ Trạng thái: `STATUS.md#<module>`" để tra nhanh.
- Nhãn `[N]` / `[CẦN CHỐT]` trong spec = **quyết định thiết kế cần chốt**, không phải trạng thái code.

---

## 5. Nền tảng kỹ thuật

- **Blockchain:** Cardano L1
- **Ngôn ngữ hợp đồng:** Aiken → PlutusV3
- **Danh tính:** phương thức `did:phoenix` — **đã đăng ký trong danh bạ DID Method của W3C** ("Known DID Methods", Group Note). Đặc tả method: `https://phoenixkey.me/did-method/v1` · sổ cái: Cardano · liên hệ: GreenSun Tech. Resolver theo chuẩn W3C DID Resolution v0.3.
- **Khoá:** sinh và giữ trong Secure Enclave (phần cứng), không rời thiết bị
- **Đường cong:** P‑256 (Secure Enclave) / Ed25519 / secp256k1 tuỳ ngữ cảnh

---

## 6. Giấy phép

Phát hành theo **Apache License 2.0** (xem `LICENSE`). Cho phép dùng lại, sửa đổi, phân phối kèm điều khoản sáng chế và ghi công — phù hợp mục tiêu Open SDK cho hệ Cardano.

---

*Cập nhật: 2026-07-09. Bộ đặc tả là kim chỉ nam thiết kế; hiện trạng từng tính năng: xem [`PhoenixKey-STATUS.md`](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md).*

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
