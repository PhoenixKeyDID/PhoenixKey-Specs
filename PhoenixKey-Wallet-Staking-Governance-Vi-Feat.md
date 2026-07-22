# PhoenixKey — Ví tham gia mạng Cardano: Staking, Quản trị (Voting), Pool, SRCL

> **Tài liệu này viết cho ai:** người dùng PhoenixKey và đội sản phẩm — KHÔNG phải kỹ sư hay auditor. Mục tiêu: hiểu PhoenixKey cho bạn tham gia được gì vào mạng Cardano ngoài giữ tiền, bằng ngôn ngữ đời thường.
> **Đây KHÔNG phải module thứ 9.** Kiến trúc PhoenixKey giữ nguyên **8 module** (xem [PhoenixKey-Whitepaper.md](./PhoenixKey-Whitepaper.md) §2). Tài liệu này mô tả **năng lực ví mở rộng của Rebirthme** (module ví) — cùng cách [PhoenixKey-Wallet-API-v2-Feat.md](./PhoenixKey-Wallet-API-v2-Feat.md) mô tả tầng API mà không phải module riêng.
> **Loại doc:** Feature (hướng người dùng/sản phẩm). **Ngày:** 2026-07-12. **Trạng thái:** định hướng (kim chỉ nam) — một phần đã có nền thật trong code (xem §5), phần còn lại đang xây.
> Nền tảng ví (địa chỉ, khôi phục, anti-drain): [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md).

---

## 1. Một câu là gì

**Ví Phượng hoàng không chỉ giữ tiền — nó còn là cách bạn tham gia vận hành và biểu quyết trên chính mạng Cardano: uỷ thác ADA, nhận thưởng, chọn người đại diện quản trị (DRep) hoặc tự mình quản trị, và nếu bạn là nhà vận hành pool thì điều hành cả pool — tất cả ký bằng danh tính sinh trắc PhoenixKey, không cần một ví lạnh riêng cắm dây.**

---

## 2. Bốn năng lực

### 2.1 Staking — uỷ thác và nhận thưởng
Uỷ thác ADA vào một hoặc nhiều stake pool cùng lúc (đa-pool), đổi pool, ngừng uỷ thác, rút thưởng staking về ví. Đây là năng lực Cardano cơ bản nhất — vốn ADA không rời ví bạn, chỉ phần **ủy thác** (stake credential) được gắn vào pool.

### 2.2 Quản trị Cardano (Voltaire / CIP-1694) — KHÁC quản trị nội bộ LAMP
Từ kỷ nguyên Voltaire, Cardano có tầng quản trị on-chain riêng: **DRep** (Delegated Representative — người đại diện bỏ phiếu hộ) và **GA** (Governance Action — đề xuất thay đổi tham số giao thức, rút quỹ Treasury, v.v., mọi người dùng ADA/DRep đều đọc và bỏ phiếu được). PhoenixKey cho bạn:
- Uỷ quyền phiếu (delegate) cho một DRep bạn tin tưởng, hoặc tự đăng ký làm DRep cho chính mình.
- Đọc danh sách GA đang mở.
- Bỏ phiếu (Yes/No/Abstain) cho GA nếu bạn đã đăng ký DRep hoặc có ADA đủ điều kiện.
- Tự nộp một GA mới (nếu đủ điều kiện đặt cọc theo luật giao thức Cardano).

> **Phân biệt bắt buộc — không được nhầm hai cơ chế:** đây là quản trị **Cardano layer 1** (CIP-1694, DRep on-chain, mỗi phiếu tính theo ADA đã uỷ thác). Quản trị nội bộ hệ MagicLamp (LAMP/MAGIC) là cơ chế **hoàn toàn khác** — KHÔNG token-weighted, cử tri là cá nhân qua PhoenixKey DID sinh trắc, quyền biểu quyết (VP) tính theo tích 4 tham số (MAGIC tiêu thụ, LAMP cam kết, uy tín, LAMP nắm giữ có trần) — xem `LAMP/Governance/VotingPower/CONTRACT.md`. Hai hệ quản trị này **không cộng dồn, không thay thế nhau**, chỉ cùng dùng chung ví PhoenixKey để ký.

### 2.3 Tab Pool — vận hành stake pool ngay trên điện thoại
Với người vận hành pool (SPO — Stake Pool Operator): tạo pool mới, cập nhật tham số pool (margin, cost, pledge), xoay khoá vận hành (KES rotate), rút lui pool (retire) — tất cả trên mobile, **không cần máy air-gap riêng**.

`[CẦN CHỐT]` — đây là điểm cần cân nhắc kỹ trước khi triển khai: khoá cold của pool theo thông lệ ngành khuyến nghị giữ offline (air-gapped) vì phạm vi ảnh hưởng lớn hơn hẳn một ví cá nhân — lộ khoá cold của pool ảnh hưởng **toàn bộ người uỷ thác vào pool đó**, không chỉ chủ pool. Bỏ air-gap vật lý phải được bù bằng lớp bảo vệ tương đương trên PhoenixKey (Secure Enclave non-extractable + đa yếu tố khi ký thao tác pool + phát hiện jailbreak/root chặn cứng thao tác pool trên máy nghi ngờ + xoay khoá nhanh khi nghi lộ) — không đơn thuần "bỏ air-gap cho tiện". Thiết kế chi tiết mức bảo vệ này thuộc Math/Tech doc kế tiếp, chưa chốt ở bản Feat này.

### 2.4 Vai trò ký cho SRCL
SRCL (**Staking Reward Contribution Launch**) là cơ chế ra mắt của hệ sinh thái MagicLamp: người đang uỷ thác ADA đóng góp **phần thưởng staking** (không phải vốn gốc) cho một đợt Launch, nhận lại LAMP theo tỉ lệ đóng góp. Cơ chế + kinh tế + pháp lý của SRCL là **canonical tại LAMP**, xem [LAMP/SPEC/SRCL-Spec-Vi.md](../../LAMP/SPEC/SRCL-Spec-Vi.md) — tài liệu này KHÔNG lặp lại, chỉ mô tả phần **PhoenixKey phải cung cấp**:

1. **Ký uỷ quyền một lần:** giao dịch nối phần ủy thác (stake credential) của bạn tới script của đợt Launch — vốn gốc (phần chi tiêu) không bị đụng, chỉ phần ủy thác.
2. **Ký nhận LAMP theo lịch:** mỗi khi tới lượt nhận LAMP theo lịch nhả của đợt, PhoenixKey đưa ra thông báo, bạn **một chạm để duyệt** — vẫn là chữ ký tươi của chính bạn mỗi lần, KHÔNG phải PhoenixKey giữ khoá ký thay hay tự động ký không cần bạn. Trải nghiệm được rút gọn (không phải tự dựng giao dịch thủ công mỗi epoch), nhưng bất biến **non-custodial** của toàn hệ PhoenixKey giữ nguyên — không có ngoại lệ cho SRCL.

---

## 3. Bảng phân biệt nhanh

| | Staking (2.1) | Quản trị Cardano (2.2) | Pool (2.3) | SRCL (2.4) |
|---|---|---|---|---|
| Chạm tới vốn gốc? | Không | Không | Không (chỉ khoá vận hành pool) | Không |
| Ai định nghĩa cơ chế | Giao thức Cardano | CIP-1694 (Cardano) | Giao thức Cardano | LAMP/Launch Framework |
| PhoenixKey làm gì | Ký lệnh delegate/rút thưởng | Ký lệnh delegate DRep/vote/nộp GA | Ký lệnh vận hành pool | Ký uỷ quyền 1 lần + ký claim theo lịch |
| Trạng thái | Đã có nền (multi-pool + DRep trong `rust_core/src/staking.rs`) | Một phần nền (DRep delegate đã có, đọc/vote/nộp GA — đang xây) | Đang xây, có điểm `[CẦN CHỐT]` | Đang xây |

---

## 4. Bất biến bắt buộc

- **Không custodial ở bất kỳ năng lực nào trong 4 mục trên.** Mọi chữ ký đều là chữ ký tươi của khoá do chính bạn giữ (Secure Enclave-gated) — kể cả khi trải nghiệm được rút gọn thành "một chạm" (như claim SRCL theo lịch).
- **Vốn ADA gốc (phần chi tiêu) không bao giờ bị các cơ chế này chạm tới** — chỉ phần ủy thác (stake credential) hoặc khoá vận hành pool.
- **Quản trị Cardano (CIP-1694) và quản trị nội bộ LAMP là hai hệ tách biệt, không trộn** — xem §2.2.

---

## 5. Trạng thái thật hôm nay

Đã có nền thật trong code (`PhoenixKey-Core/Enclave/rust_core/src/staking.rs`): xây giao dịch stake delegation đa-pool, rút thưởng, đăng ký/huỷ đăng ký stake key, delegate vote cho DRep (Conway era). Chưa có: đọc/hiển thị GA đang mở, tự bỏ phiếu GA, tự nộp GA, toàn bộ tab Pool, và phần ký SRCL. → Trạng thái & tiến độ chi tiết: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md).

---

## Nguồn

Cơ chế SRCL (canonical): `LAMP/SPEC/SRCL-Spec-Vi.md`, `LAMP/SPEC/Launch-Framework-Vi.md`.
Quản trị nội bộ LAMP (để phân biệt, không phải nội dung tài liệu này): `LAMP/Governance/VotingPower/CONTRACT.md`.
Nền tảng ví: [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md).
Code nền hiện có: `PhoenixKey-Core/Enclave/rust_core/src/staking.rs`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
