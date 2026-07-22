# PhoenixKey — Tự lưu seed phân tán (Self-Custody) — Feat

> **Module:** mở rộng của hạ tầng custody (Rebirthme + SeedDistribution). **Đối
> tượng:** người dùng hoài nghi (không muốn tin PhoenixKey/LampNet tự động giữ
> hộ bí mật của mình) + đội build PhoenixKey + đội LampNet (bên nhận yêu cầu
> giao diện ở §5). **Loại doc:** Feat — thiết kế + **INTERFACE CONTRACT**,
> KHÔNG phải code. **Ngày:** 2026-07-14.
>
> **Vì sao viết bây giờ dù LampNet chưa có mạng LAN thật:** mạng LAN riêng của
> LampNet (offline, do một người tạo, giới hạn thiết bị) đang ở giai đoạn thiết
> kế. Tài liệu này giả định năng lực đó **sẽ có** và đặc tả PhoenixKey **cần
> gì** ở nó, để khi LampNet build xong, ghép được ngay — không phải thiết kế
> lại từ đầu.
>
> **Quan hệ với 2 tài liệu đã có (đọc trước, KHÔNG lặp lại nội dung):**
> - [`PhoenixKey-SeedDistribution-Tech-Math.md`](./PhoenixKey-SeedDistribution-Tech-Math.md)
>   — **Trục A** (durability, LT-fountain) + **Trục B** (custody chữ ký,
>   FROST-Ed25519 t-of-n). Tài liệu đó lo **ai được KÝ**.
> - [`PhoenixKey-Rebirthme-Vi-Feat.md`](./PhoenixKey-Rebirthme-Vi-Feat.md) §4.4
>   — **I-WALLET-8**: khôi phục qua **guardian = uỷ quyền rotate, KHÔNG giữ
>   mảnh** (đã bỏ hẳn Shamir khỏi đường guardian).
> - **Tài liệu NÀY mở một trục thứ ba, song song, KHÔNG thay hai cái trên:**
>   **Trục C — tự lưu bản sao SEED THẬT** (không phải khoá ký, không phải
>   quyền rotate) dưới dạng mảnh Shamir cổ điển, do **chính user chọn nơi giữ**
>   (thiết bị của mình / người có tên / pool ẩn danh LampNet), cho ai **không
>   muốn phụ thuộc** cả guardian lẫn sao lưu-tự-động mặc định. Đây là lựa chọn
>   **opt-in**, xếp CHỒNG lên các trục kia, không xoá bỏ khuyến nghị vẫn nên
>   bật guardian (§6).

---

## 0. TL;DR

1. **Vấn đề kép:** (a) user hoài nghi không tin ai giữ hộ bí mật của mình dù
   đã mã hoá — muốn **tự tay** rải mảnh cho người/thiết bị họ chọn; (b) hệ
   thống rải không được **tự khai** cho kẻ tấn công biết "đây là một vault
   bí mật" qua metadata (tên trường, thư mục riêng) — hiện tại **đang khai**
   (xem §4.1).
2. **Kiến trúc:** client tự cắt seed thành mảnh bằng Shamir Secret Sharing
   (crate `sharks`), mỗi mảnh trông như một blob bulk bình thường khi upload;
   một **manifest mã hoá** (không phải Master_KEK — xem lý do §4.3) ghi lại
   *ai giữ mảnh nào*, định vị qua **LocatorID tất định từ DID** (không phải
   `HW_UID` máy cũ — vá đúng lỗ đang có).
3. **3 chế độ (a/b/c) do user chọn**, không hỏi "k mấy n mấy" — chỉ đưa nhãn
   hệ quả. 3 tier ẩn danh mặc định (3-of-5 / 5-of-9 / 20-of-64) + 1 tier
   "không phân tán pool nào" cho tài sản lớn.
4. **🔴 Lỗ khôi phục nghiêm trọng đang có phải vá TRƯỚC khi tính năng này
   chạy thật:** LampNet chưa có dịch vụ tra `LocatorID → CID`; CID hiện chỉ
   tồn tại phía client (`TODO(cid-anchoring)` ngay trong code). Mất máy = mất
   luôn đường tìm lại dữ liệu, bất kể mã hoá tốt đến đâu.
5. **§5 là phần quan trọng nhất** — liệt kê chính xác PhoenixKey cần LampNet
   xây gì.
6. **Trung thực:** giả trang (disguise) là *obscurity*, không phải *security*;
   Shamir cổ điển không chống "gom mảnh dần theo năm" nếu chưa có proactive
   refresh (PSS) — LampNet chưa có, phải nói thẳng cho tier tài sản lớn.

---

## 1. Một câu là gì + vấn đề

**Tính năng này cho user tự quyết định NƠI cất bản sao seed của mình — trên
chính thiết bị họ, trên máy người thân họ đặt tên, hoặc rải ẩn danh lên mạng
LampNet — thay vì phải tin tưởng mù quáng vào cơ chế sao lưu tự động mặc định
của PhoenixKey.**

Hai loại người cần tính năng này:
- **User hoài nghi tự-lưu-hộ.** Họ chấp nhận PhoenixKey mã hoá tốt, nhưng
  không muốn *bất kỳ hệ thống nào* — kể cả PhoenixKey — là bên duy nhất đứng
  giữa họ và bản sao lưu. Họ muốn nhìn thấy, kiểm soát, tự chọn ai/cái gì giữ
  mảnh nào.
- **User tài sản lớn.** Họ chấp nhận đánh đổi tiện lợi lấy việc **không** đưa
  bất kỳ mảnh nào vào pool công cộng ẩn danh — chỉ Enclave phần cứng + người/
  tổ chức họ đích danh tin tưởng.

**Vấn đề kỹ thuật đứng sau (hai nửa, dễ nhầm là một):**
1. **Không tự khai cho kẻ tấn công.** Một hệ rải mảnh mà gắn nhãn/thư mục
   riêng biệt ("đây là vault_sss", header `X-LampNet-DataClass: vault`) đã tự
   tố cáo *DID này có tài sản đáng rải mảnh bảo vệ* — trước khi kẻ tấn công
   giải được gì. Giả trang media (ảnh/audio nguỵ trang) là lớp đắt nhất nhưng
   giá trị cận biên thấp vì payload upload **đã** là ciphertext mù — hội đồng
   3 agent (2026-07-13) kết luận lỗ thật nằm ở **metadata**, không ở tầng
   payload.
2. **Cho user hoài nghi niềm tin.** Kể cả nếu metadata sạch, nhiều user vẫn
   không muốn seed của họ đi qua một pipeline họ không tự chọn từng bước. Câu
   trả lời không phải "thuyết phục họ tin" mà là **cho họ nút bấm tự chọn**,
   với cảnh báo trung thực về đánh đổi ở mỗi mức.

---

## 2. Kiến trúc

### 2.1 Sơ đồ luồng (tổng quan)

```
[Máy user]
  seed (256-bit entropy đằng sau 24 từ / Master_KEK gốc)
        │
        ▼
  Shamir_Split(secret=seed, k, n)      -- crate `sharks`, client-side, KHÔNG rời máy dạng rõ
        │
        ├─► share_1 ──┐
        ├─► share_2 ──┤   mỗi mảnh ~61 byte (không đổi theo n — kích thước Shamir
        ├─► ...       │   tuyến tính với ĐỘ DÀI secret, không với SỐ mảnh)
        └─► share_n ──┘
              │
              ▼
    Upload MỖI mảnh như một blob BULK THƯỜNG
    (data_class="bulk", KHÔNG có trường/thư mục riêng đánh dấu "đây là share")
              │
              ▼
    Đích đến theo lựa chọn user (§3): thiết bị LAN riêng / người có tên / pool ẩn danh
              │
              ▼
    Manifest = { locator_i cho từng mảnh, holder_label, k, n, version }
    → mã hoá bằng khoá dẫn xuất từ MẬT KHẨU RIÊNG user đặt lúc bật tính năng
      (Recovery Manifest Passphrase — RMP, KHÔNG phải Master_KEK, lý do §4.3)
              │
              ▼
    Upload Manifest, định vị qua LocatorID TẤT ĐỊNH từ DID (KHÔNG từ HW_UID — §4.2)
```

### 2.2 Vì sao Shamir ở ĐÂY hợp lý dù đã "bỏ Shamir" ở trục guardian

I-WALLET-8 bỏ Shamir cho **trục guardian** vì ở đó guardian giữ mảnh của
**quyền rotate**, và việc dựng lại (reconstruct) tạo ra một "điểm nguyên vẹn"
nguy hiểm mà DKG/FROST loại được sạch hơn (xem `SeedDistribution-Tech-Math.md`
§I.3.2–§I.3.3). Ở tài liệu này, đối tượng bị chia KHÔNG phải khoá ký dùng lặp
lại nhiều lần — mà là **bản sao lưu tĩnh của seed**, dùng ĐÚNG MỘT LẦN lúc mất
máy để dựng lại và bơm vào Enclave máy mới (giống hệt việc dựng lại từ 24 từ
giấy hôm nay). "Điểm nguyên vẹn lúc dựng lại" tồn tại **thoáng qua** trong bộ
nhớ máy mới — đúng bản chất của MỌI đường khôi phục seed (kể cả gõ tay 24 từ),
không phải lỗ riêng của Shamir. Vì vậy Shamir SSS (Shamir 1979) — không phải
FROST — là công cụ đúng: nó chia **dữ liệu tĩnh**, không chia **năng lực ký
lặp lại**.

### 2.3 Vì sao mảnh không cần giả trang để "vô hình"

Với `secret` là 256-bit entropy gần như ngẫu nhiên tuyệt đối, mỗi mảnh Shamir
(giá trị trên trường hữu hạn) **không phân biệt được thống kê** với chuỗi byte
ngẫu nhiên bất kỳ (tính chất perfect-secrecy của Shamir SSS — §7). Nghĩa là
mảnh tự nó đã "vô hình" trong biển dữ liệu ngẫu nhiên. Cái làm lộ nó KHÔNG phải
nội dung mảnh mà là **cách hệ thống gắn nhãn nó khi upload/lưu trữ** — đúng
điểm hội đồng chỉ ra. ⟹ Ưu tiên #1 là dọn metadata (§4.1), KHÔNG phải mua thêm
lớp giả trang đắt đỏ.

---

## 3. Ba chế độ + tier — UX chọn bằng NHÃN, không hỏi k/n

### 3.1 Ba chế độ (trục "ở đâu")

| Chế độ | Mô tả | Phụ thuộc mạng | Khi nào chọn |
|---|---|---|---|
| **(a) Toàn bộ trong tầm tay** | Mọi mảnh nằm trên thiết bị **chính chủ user**, nối qua **mạng LAN riêng LampNet** (offline được, không cần Internet). | KHÔNG phụ thuộc pool công cộng — hoạt động cả khi LampNet.cloud sập. | User có ≥2-3 thiết bị riêng (điện thoại cũ, máy tính bảng, laptop) và không muốn dữ liệu rời khỏi tài sản của họ. |
| **(b) Phần lớn off + ít on** | Đa số mảnh trên thiết bị user (mode a), một số ít mảnh gửi cho người có tên hoặc pool. | Phụ thuộc một phần — mảnh "on" cần LampNet còn sống để khôi phục cửa sổ đó. | User muốn dự phòng cho trường hợp mất TẤT CẢ thiết bị cùng lúc (cháy nhà, trộm). |
| **(c) Phân tán hẳn lên pool** | Phần lớn/toàn bộ mảnh rải ẩn danh trên pool LampNet công cộng. | Phụ thuộc LampNet sống + tìm đủ node online lúc khôi phục. | User không có nhiều thiết bị riêng, chấp nhận mô hình mặc định gần giống sao lưu tự động nhưng vẫn TỰ chọn bật/tắt, tự chọn tier. |

Ba chế độ này là **tỷ lệ off/on do user khai báo**, không phải 3 sản phẩm khác
nhau — cùng một cơ chế Shamir + manifest, chỉ khác **holder set**.

### 3.2 Tier ẩn danh mặc định (trục "bao nhiêu mảnh, ngưỡng bao nhiêu")

| Tier | k-of-n | Đặc điểm | Nhãn UX hiển thị cho user |
|---|---|---|---|
| **Tự chọn (riêng tư nhất)** | 3-of-5 | Holder do user **đích danh** chọn (thiết bị/người); không vào pool ẩn danh. | *"Riêng tư tối đa — chỉ người/thiết bị tôi tự chọn"* |
| **Cân bằng (mặc định gợi ý)** | 5-of-9 | Pool ẩn danh LampNet, có cơ chế sửa mảnh khi node rớt (MSR repair — §7). | *"Tiêu chuẩn — cân bằng an toàn & tiện lợi"* |
| **Kho sâu** | 20-of-64 | Pool ẩn danh quy mô lớn, chống churn (node rớt/vào liên tục) tốt hơn nhờ số dư lớn — **không** phải chống thông đồng tốt hơn (§6). | *"Kho sâu — bền bỉ nhất, khôi phục có thể chậm hơn"* |

`n` cao ở tier "Kho sâu" là để **chống churn của pool ẩn danh** (node vào/ra
liên tục — cần dư nhiều mảnh để luôn có ≥k mảnh sống), **KHÔNG PHẢI** để chống
thông đồng — thông đồng chỉ cần ≥k node hợp tác bất kể n lớn cỡ nào (xem §6).

### 3.3 Tier "không phân tán pool nào" (tài sản lớn)

Dành riêng cho tài sản mà user không chấp nhận bất kỳ mảnh nào rơi vào tay ẩn
danh, kể cả xác suất nhỏ:
- Toàn bộ mảnh chỉ nằm ở: **Enclave phần cứng của chính user** + **người/tổ
  chức CÓ TÊN** user tự liệt kê thủ công (không qua thuật toán chọn ngẫu
  nhiên của LampNet).
- **KHÔNG** dùng pool anonymit dưới bất kỳ hình thức nào cho tier này.
- Cảnh báo cứng bắt buộc hiển thị (§3.4) trước khi kích hoạt.

### 3.4 UX chọn — chỉ đưa nhãn hệ quả

Màn hình chọn tuyệt đối **KHÔNG hỏi** "bạn muốn k mấy n mấy". User chỉ thấy 4
thẻ, mỗi thẻ là một câu hệ quả + 1 dòng cảnh báo nếu cần:

```
┌─────────────────────────────────────────────┐
│ ○ Riêng tư tối đa                            │
│   Chỉ người/thiết bị tôi TỰ CHỌN giữ. Mất     │
│   quá nửa số họ = mất khả năng khôi phục.     │
├─────────────────────────────────────────────┤
│ ● Tiêu chuẩn (khuyên dùng)                    │
│   Rải ẩn danh trên mạng LampNet, tự sửa khi   │
│   có node rớt.                                │
├─────────────────────────────────────────────┤
│ ○ Kho sâu                                     │
│   Bền bỉ nhất cho tài sản giữ lâu dài. Khôi    │
│   phục có thể mất thêm thời gian tìm mảnh.     │
├─────────────────────────────────────────────┤
│ ○ Không rời khỏi tầm tay tôi  ⚠️              │
│   KHÔNG dùng mạng ẩn danh. Chỉ Enclave +      │
│   người tôi đích danh chọn. TỰ CHỊU TRÁCH     │
│   NHIỆM hoàn toàn nếu quản lý sai.             │
└─────────────────────────────────────────────┘
```

**Cảnh báo cứng bắt buộc** khi user chọn "Riêng tư tối đa" hoặc "Không rời
khỏi tầm tay tôi" (mức thấp/tự chịu trách nhiệm) — không cho bấm tiếp nếu chưa
tick xác nhận đã đọc:

> *"Bạn đang chọn mức KHÔNG có mạng lưới rộng đứng sau. Nếu quá nửa số người/
> thiết bị bạn chọn cùng lúc không còn khả dụng (mất máy, người mất liên lạc),
> **PhoenixKey và LampNet KHÔNG THỂ giúp bạn khôi phục seed này**. Đây là lựa
> chọn tự-chủ hoàn toàn (self-custody) — trách nhiệm quản lý thuộc về bạn."*

---

## 4. Vá lỗ khôi phục — bắt buộc trước khi tính năng này chạy thật

### 4.1 Metadata không được tự khai

**Hiện trạng đã xác nhận trong code** (`Enclave/rust_core/src/lampnet.rs:297`,
`PhoenixKey-Core` và mọi worktree cùng branch):

```rust
"multipart": {
    "file": ciphertext_hex,
    "data_class": "bulk",     // hard-code — nhưng field NÀY tồn tại và
    "redundancy": "2.5",      // các proposal khác (Legacy/.../Secret-Vault...)
}                              // đã đề xuất data_class="recovery" RIÊNG cho
                               // vault — đó chính là điểm tự khai cần TRÁNH.
```

**Yêu cầu cho tính năng này:** mảnh Shamir + manifest của Self-Custody **PHẢI**
đi qua đúng con đường `data_class="bulk"` **giống hệt** mọi blob thường khác
(DID Document off-chain, droplet khác) — **KHÔNG** tạo `data_class` mới
(`"vault"`, `"seed-share"`...), **KHÔNG** tạo thư mục/endpoint riêng biệt cho
"vault". Nếu về sau cần tách `data_class="recovery"` cho mục đích durability
khác (K=8 thay vì K=4, theo bảng `Rebirthme-Tech.md`), phải cân nhắc đánh đổi
đó **làm lộ tín hiệu "đây là vault"** — ghi rõ trong interface contract §5.5.

### 4.2 LocatorID phải tất định từ DID, KHÔNG từ HW_UID

**Lỗ đang có, xác nhận trong code** (`lampnet.rs:61-63, 144-157`):

```rust
// Math Spec §7.2 (LocatorID derivation — unchanged):
//   LocatorSecret = HKDF(HW_UID ∥ DID, "lampnet-locator-v1")
//   LocatorID     = SHA-256(DID ∥ Epoch ∥ LocatorSecret)
pub fn derive_locator(hw_uid: &[u8], did: &str) -> Locator { ... }
```

`hw_uid` là UID phần cứng của **thiết bị hiện tại**. Máy mới sau khi mất máy
cũ có `HW_UID` khác → `derive_locator` cho **LocatorID khác** → không tra lại
được gì đã upload từ máy cũ. Đây đúng là lỗ đã ghi nhận ở
`SESSION-STATE-phoenixkey-2026-07-13.md` §5, xác nhận lại bằng code thật.

**Vá cho tính năng này (Manifest Locator — không phụ thuộc thiết bị):**

```
ManifestLocatorSecret = HKDF( ikm  = DID,
                               info = "phoenixkey-selfcustody-manifest-locator-v1",
                               salt = ∅ )                          ∈ {0,1}^256

ManifestLocatorID = SHA-256( DID ∥ "selfcustody-manifest-v1" ∥ ManifestLocatorSecret )
```

Chỉ phụ thuộc **DID** (chuỗi công khai, user gõ lại được trên máy mới — đúng
luồng đã mô tả ở `Rebirthme-Vi-Feat.md` §5 "nhập DID cần khôi phục") — **KHÔNG**
phụ thuộc `HW_UID`. Máy nào biết DID cũng tính ra đúng một `ManifestLocatorID`.

> **Khuyến nghị ngoài phạm vi trực tiếp tài liệu này nhưng LIÊN QUAN chặt:**
> lỗ `HW_UID` ở `derive_locator()` là lỗ **tổng quát**, ảnh hưởng toàn bộ
> pipeline LampNet (không riêng tính năng này). Vá cho manifest ở đây không
> thay thế việc vá gốc. Báo riêng cho Core/Long — KHÔNG thuộc phạm vi Claude
> tự sửa (ranh giới backend/Core).

### 4.3 Vì sao khoá mã hoá manifest KHÔNG được là Master_KEK

Bài toán con gà-quả trứng: lúc cần khôi phục (mất máy), user **chưa có**
`Master_KEK` — đó chính là thứ đang cố lấy lại. Nếu khoá giải mã manifest lại
là `Master_KEK`, user không thể mở được manifest để tìm mảnh → vòng lặp bế tắc.

**Giải pháp (tái dùng nguyên xi mẫu đã có trong sản phẩm — không phát minh
mới):** dùng đúng cơ chế **"Xuất bản mã hoá để tự cất riêng"** đã có ở
`Rebirthme-Vi-Feat.md` §4.7 — một **mật khẩu riêng do user đặt** lúc bật tính
năng này (gọi tắt **RMP — Recovery Manifest Passphrase**, khác PIN ví, khác
seed), dẫn xuất khoá qua Argon2id/PBKDF2, mã hoá manifest bằng AES-256-GCM.
User tự cất RMP theo cách của họ (đây chính là điểm "tự chủ" mà tính năng này
hướng tới — không có RMP thứ hai nào PhoenixKey giữ hộ).

```
ManifestKey = Argon2id( RMP, salt = H(DID ∥ "selfcustody-manifest-salt-v1") )
EncManifest = AES-256-GCM( ManifestKey, Manifest )
Upload( EncManifest ) @ ManifestLocatorID   →  cần LampNet trả CID + LƯU map
```

### 4.4 🔴 Lỗ nghiêm trọng nhất: chưa có dịch vụ `LocatorID → CID`

**Xác nhận trong code, TODO còn nguyên** (`lampnet.rs:45-56, 72-75`):

> `LampNet là content-addressed; CID được gán lúc upload, KHÔNG tất định
> (chứa ephemeral pubkey + nonce ngẫu nhiên) — KHÔNG re-derive được từ vật
> liệu phía client. Caller PHẢI PERSIST CID lúc upload và cung cấp lại lúc
> khôi phục. `TODO(cid-anchoring)`: quyết định + nối anchor (field trong TAAD
> datum vs. mapping phía backend) — CHƯA làm.`

`grep` toàn repo cho `lampnet-mirage` phía server = 0 kết quả — **không có
dịch vụ nào** hiện nay để tra `LocatorID → CID`. Hệ quả trực tiếp cho tính
năng này: dù `ManifestLocatorID` đã tất định (§4.2), user vẫn **không có nơi
nào để hỏi** "CID của manifest ứng với LocatorID này là gì" sau khi mất máy
cũ (máy cũ là nơi duy nhất từng biết CID đó).

**⟹ Đây là điều kiện TIÊN QUYẾT, không phải optional, cho toàn bộ tính năng
Self-Custody chạy được ở chế độ (b)/(c) (và cả (a) nếu thiết bị chỉ có
`ManifestLocatorID` không đủ, cần một trạm tra cứu trong LAN riêng).** Xem
yêu cầu cụ thể ở §5.4.

### 4.5 Luồng khôi phục đầu-cuối sau khi mất máy

```
1. User cài PhoenixKey trên máy mới, chọn "Khôi phục từ Self-Custody".
2. Nhập DID (chuỗi công khai, user còn nhớ/lưu ở đâu đó — không bí mật).
3. Client tính ManifestLocatorID = f(DID)  (§4.2, không cần HW_UID).
4. Client hỏi LampNet: GET locator/{ManifestLocatorID} → CID_manifest
                                              (dịch vụ MỚI, §5.4 — hiện CHƯA có)
5. Client tải EncManifest tại CID_manifest.
6. User nhập RMP (mật khẩu tự đặt lúc bật tính năng) → giải mã Manifest.
7. Manifest cho danh sách { locator_share_i, holder_label_i, k, n }.
8. Với chế độ (a): client tự nối LAN riêng, hỏi trực tiếp từng thiết bị.
   Với chế độ (b)/(c): với mỗi locator_share_i, lặp lại bước 4 (tra CID),
   tải mảnh — cho tới khi đủ k mảnh.
9. sharks::recover(shares) → seed gốc.
10. Client derive lại Master_KEK, ví "sống lại" tại đúng địa chỉ cũ
    (giống mô tả Rebirthme-Vi-Feat.md §5, không cần đổi khoá on-chain nếu
    seed = khoá gốc còn hợp lệ; nếu muốn, user có thể rotate ngay sau đó).
```

---

## 5. INTERFACE CONTRACT — PhoenixKey cần LampNet cung cấp CHÍNH XÁC gì

> Đây là phần đội LampNet cần đọc kỹ nhất. Mỗi mục ghi rõ: **API/hàm cần có**,
> **input/output**, **lý do** (map về chế độ/tier ở §3), **mức ưu tiên**.

### 5.1 Mạng LAN riêng (chế độ a) — tạo + tham gia

```
create_vault_network(owner_did, device_pubkey) -> network_id
join_vault_network(network_id, invite_token, device_pubkey) -> ok | denied
list_network_devices(network_id, requester_sig) -> [device_id...]
revoke_device(network_id, device_id, owner_sig) -> ok
```
- **Bắt buộc hoạt động OFFLINE hoàn toàn** (không đòi `lampnet.cloud` sống) —
  đây là lý do chọn chế độ (a). Đồng bộ giữa các thiết bị trong cùng
  `network_id` phải qua kênh LAN cục bộ (mDNS/Bluetooth/QR-relay — tuỳ LampNet
  chọn cơ chế), KHÔNG bắt buộc Internet.
- **Ưu tiên: BẮT BUỘC** cho chế độ (a). Không có mục này, chế độ (a) không tồn
  tại — user "toàn bộ trong tầm tay" vẫn phải cậy Internet, phá đúng lời hứa.

### 5.2 Chỉ định holder set (chế độ a/b)

```
assign_share_holder(manifest_id, share_index, target) -> ok
  target = { device_id (trong network riêng) | named_contact_pubkey }
```
- Cho user **tự tay** gán mảnh N cho thiết bị/người cụ thể — không qua thuật
  toán chọn ngẫu nhiên của LampNet. Khác hẳn tier pool (§5.3) nơi LampNet tự
  chọn node.
- **Ưu tiên: BẮT BUỘC** cho tier "Tự chọn" (3-of-5) và tier "Không rời khỏi
  tầm tay tôi" (§3.3).

### 5.3 Khai tỷ lệ off/on + join pool ẩn danh (chế độ b/c)

```
declare_offline_ratio(manifest_id, off_shares:[i...], on_shares:[i...]) -> ok
join_anonymous_pool(tier: "5-of-9" | "20-of-64", blob) -> [locator_i...]
```
- `join_anonymous_pool`: LampNet tự chọn node theo tier khai báo (5-of-9 hoặc
  20-of-64 — §3.2), trả về danh sách locator client dùng để ghi vào manifest.
  LampNet **không cần biết** đây là share Shamir hay blob thường (§4.1) —
  input/output giống hệt API upload bulk hiện có, chỉ khác ở việc client gọi
  nó `n` lần cho `n` mảnh.
- `declare_offline_ratio`: cho chế độ (b), LampNet cần biết mảnh nào kỳ vọng
  "phần lớn offline" để KHÔNG báo động/loại nó khỏi churn-repair quá sớm (một
  thiết bị offline 3 tháng vẫn hợp lệ, không phải "node chết" theo nghĩa pool
  công cộng).
- **Ưu tiên: BẮT BUỘC** cho chế độ (b)/(c).

### 5.4 Dịch vụ `LocatorID → CID` (🔴 BLOCKING — ưu tiên cao nhất)

```
PUT  locator/{locator_id}   body: { cid, owner_sig }     -> ok | conflict
GET  locator/{locator_id}                                -> { cid } | not_found
```
- **Lý do:** §4.4 — hiện KHÔNG tồn tại (grep = 0). Không có mục này, TOÀN BỘ
  luồng khôi phục ở §4.5 bước 4 gãy — mất máy = mất luôn đường tìm dữ liệu,
  bất kể mã hoá/Shamir tốt đến đâu.
- **Cần chống squat/ghi đè bởi bên thứ ba:** `PUT` phải xác thực `owner_sig`
  (chữ ký từ khoá controller hiện tại của DID tương ứng — LampNet có thể verify
  qua anchor on-chain đã public) — nếu không, kẻ tấn công biết trước
  `LocatorID` tất định (vì derive từ DID công khai — §4.2) có thể **ghi đè
  trước** bằng CID rác, chặn user hợp lệ.
- Cần hỗ trợ **versioning** (nhiều lần `PUT` cho cùng `locator_id` khi user
  cập nhật manifest — ví dụ đổi holder set) — trả về CID mới nhất, giữ lịch sử
  tối thiểu để rollback nếu ghi nhầm.
- **Ưu tiên: BLOCKING.** Đây là điều kiện để bất kỳ phần nào khác của tài
  liệu này có ý nghĩa thực tế cho chế độ (b)/(c).

### 5.5 Không tạo `data_class` riêng cho vault (ràng buộc, không phải API mới)

- **Yêu cầu:** LampNet giữ nguyên `data_class="bulk"` cho mảnh Shamir + manifest
  của tính năng này — **không** thêm giá trị mới kiểu `"vault"`/`"seed-share"`
  cho `data_class`, **không** tạo thư mục/route riêng biệt.
- Nếu vì lý do durability (K=8 thay vì K=4 — bảng `Rebirthme-Tech.md`) LampNet
  MUỐN xử lý các blob này khác đi, phải làm bằng cách **nâng K cho toàn bộ
  class `bulk`** hoặc một cơ chế không lộ field phân biệt riêng — KHÔNG được
  đánh đổi lấy một nhãn tự-tố-cáo. Đây là ràng buộc thiết kế, cần LampNet xác
  nhận đã hiểu trước khi build.

### 5.6 MSR repair cho pool tier (5-of-9, 20-of-64)

```
notify_share_at_risk(locator_id, callback) -> subscription_id
   (LampNet gọi callback khi node giữ mảnh đó sắp/đã rời pool)
trigger_regenerate(locator_id, helper_locators:[...]) -> new_locator
   (tái sinh một mảnh thay thế TỪ các mảnh còn sống, KHÔNG cần tải hết
    secret gốc về — mô hình Regenerating Codes / MSR, §7)
```
- **Lý do:** tier 5-of-9/20-of-64 cần tự sửa khi churn xảy ra để không rơi
  dưới ngưỡng `k` theo thời gian mà user không biết. Repair kiểu MSR tiết
  kiệm băng thông so với tải lại toàn bộ `k` mảnh để dựng `n`-mới.
- **Ưu tiên: khuyến nghị mạnh cho tier pool, KHÔNG blocking cho MVP** — MVP
  có thể tạm thời dùng cảnh báo thủ công "mảnh X không phản hồi, cân nhắc
  refresh" thay vì tự động, nhưng phải có route cho user CHỦ ĐỘNG trigger lại
  toàn bộ (re-split + re-upload) như phương án tạm.

### 5.7 Timeline PSS (Proactive Secret Sharing) — câu hỏi cần LampNet trả lời

- Shamir cổ điển (§7) **không** chống được đối thủ gom đủ `k` mảnh **dần theo
  thời gian** (một node hôm nay, một node năm sau) nếu mảnh không đổi. Giải
  pháp chuẩn là **proactive refresh** định kỳ (Herzberg et al. 1995 — §7):
  làm mới toàn bộ `n` mảnh định kỳ sao cho mảnh cũ vô dụng với mảnh mới, mà
  **secret gốc không đổi**.
- PhoenixKey **KHÔNG** yêu cầu LampNet có PSS ngay cho MVP — nhưng cần biết
  **lộ trình có/không có, timeline dự kiến**, để:
  (a) quyết định có mở tier "Kho sâu" (20-of-64, giữ lâu dài) cho sản xuất
  hay chỉ để beta cảnh báo rõ; (b) nếu không có PSS trong tầm nhìn gần,
  PhoenixKey tự làm proactive refresh ở tầng client (re-split định kỳ, tự
  upload lại) — cần biết trước để thiết kế lịch trình gọi §5.3/§5.6.
- **Ưu tiên: câu hỏi cần trả lời, không phải API cần build ngay.**

---

## 6. Ranh giới trung thực — không che

- **Giả trang = obscurity, không phải security.** Ẩn payload trong ảnh/audio
  không đổi bản chất toán học của Shamir; nó chỉ làm khó hơn việc *phát hiện*
  có mảnh tồn tại. Một đối thủ đã biết DID mục tiêu và có quyền truy cập rộng
  vào LampNet (ví dụ chính node vận hành) vẫn thấy được blob — giả trang không
  chặn họ, chỉ chặn kẻ dò quét ngẫu nhiên. **Không quảng cáo giả trang là lớp
  bảo mật chính** — lớp chính là §4.1 (dọn metadata) + ngưỡng k-of-n.
- **PSS chưa có → cảnh báo cho tài sản lớn.** Không có proactive refresh,
  Shamir cổ điển hở trước đối thủ kiên nhẫn nhiều năm, đặc biệt tier "Kho sâu"
  (20-of-64, giữ lâu dài = thời gian gom mảnh dài hơn). Cảnh báo này PHẢI hiện
  lại (không chỉ lúc chọn tier) định kỳ, ví dụ mỗi 6 tháng, nhắc user cân nhắc
  tự re-split thủ công nếu LampNet chưa có PSS tự động (§5.7).
- **Tự-custody = tự chịu trách nhiệm.** Với tier "Tự chọn" và "Không rời khỏi
  tầm tay tôi", KHÔNG có bên thứ ba (kể cả guardian I-WALLET-8) tự động cứu
  nếu user quản lý sai (mất quá `n-k+1` thiết bị/người cùng lúc, quên RMP).
  Đây là điểm khác biệt cốt lõi với model guardian mặc định — nói rõ trong
  cảnh báo UX (§3.4), không diễn giải mềm đi.
- **Tính năng này KHÔNG thay guardian.** Guardian (I-WALLET-8) vẫn nên được
  bật song song — hai trục độc lập, không loại trừ nhau. Self-Custody không
  phải "bản nâng cấp" của guardian, mà là một lựa chọn **thêm** cho user muốn
  kiểm soát trực tiếp hơn phần backup dữ liệu, tách bạch khỏi quyền rotate.
- **Chưa vá lỗ khác.** Tài liệu này không vá anti-drain, guardian-dup,
  timelock-floor (các lỗ riêng, PR riêng — xem MEMORY `redteam-guardian-dup-live-poc`).
  Self-Custody giảm rủi ro *mất bản sao seed*, KHÔNG thay các lớp chống rút
  cạn khi khoá đã lộ.

---

## 7. Bảo mật khoa học (nguồn thật)

- **Shamir Secret Sharing** — Shamir, A. *"How to Share a Secret."* CACM
  22(11), 1979, tr. 612–613. DOI
  [10.1145/359168.359176](https://doi.org/10.1145/359168.359176). Tính chất
  cốt lõi dùng ở §2.3: với `< k` mảnh, không có thông tin gì (perfect secrecy,
  information-theoretic) về secret — không phải chỉ "tính toán khó", mà
  **không thể** dù có sức mạnh tính toán vô hạn.
- **Proactive Secret Sharing (PSS)** — Herzberg, A., Jarecki, S., Krawczyk, H.,
  Yung, M. *"Proactive Secret Sharing Or: How to Cope With Perpetual Leakage."*
  CRYPTO'95, LNCS 963, tr. 339–352. DOI
  [10.1007/3-540-44750-4_27](https://doi.org/10.1007/3-540-44750-4_27). Cơ sở
  cho §5.7/§6: làm mới mảnh định kỳ mà secret không đổi, vô hiệu hoá mảnh kẻ
  tấn công gom được từ chu kỳ trước.
- **Regenerating Codes / MSR** — Dimakis, A. G., Godfrey, P. B., Wu, Y.,
  Wainwright, M. J., Ramchandran, K. *"Network Coding for Distributed Storage
  Systems."* IEEE Trans. Inf. Theory 56(9), 2010, tr. 4539–4551. DOI
  [10.1109/TIT.2010.2054295](https://doi.org/10.1109/TIT.2010.2054295). Cơ sở
  cho §5.6: tái sinh mảnh bị mất với băng thông thấp hơn tải lại toàn bộ `k`
  mảnh gốc.
- **HKDF** — Krawczyk & Eronen,
  [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869) — dùng cho
  `ManifestLocatorSecret` (§4.2), cùng họ với `LocatorSecret`/`Device_KEK`
  hiện có trong `lampnet.rs`.
- **Argon2id** — Biryukov, A., Dinu, D., Khovratovich, D. *"Argon2: New
  Generation of Memory-Hard Functions."* IEEE EuroS&P 2016; chuẩn hoá
  [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106) — dùng dẫn xuất
  `ManifestKey` từ RMP (§4.3), cùng mẫu đã dùng ở tính năng xuất seed mã hoá
  (`Rebirthme-Vi-Feat.md` §4.7).
- **Vì sao ngưỡng + đa dạng holder quan trọng:** với `p` = xác suất một
  holder khả dụng lúc cần khôi phục, xác suất khôi phục thành công là
  `P = Σ_{i=k}^{n} C(n,i)·p^i·(1-p)^{n-i}` (cùng công thức nhị thức đã dùng ở
  `SeedDistribution-Tech-Math.md` §III.3.3 cho liveness FROST). Ngưỡng `k`
  thấp tăng tiện lợi nhưng giảm biên an toàn trước thông đồng; holder cùng
  một hộ gia đình/cùng ISP/cùng khu vực làm giảm tính độc lập của `p` thực tế
  (tương quan lỗi — xem giả định A3 cùng tài liệu) — khuyến nghị UX nhắc user
  chọn holder **đa dạng vị trí/loại thiết bị** khi ở tier "Tự chọn".

---

**Thư viện đề xuất:** [`sharks`](https://crates.io/crates/sharks) (Shamir SSS
thuần Rust, đã có sẵn trong hệ sinh thái `rust_core`) · `hkdf` · `argon2` ·
AEAD hiện có (`aes-gcm`, khớp `lampnet.rs`).

**Mã tham chiếu (đã đọc, dùng làm bằng chứng cho §4):**
`Enclave/rust_core/src/lampnet.rs` (mọi worktree cùng branch + `PhoenixKey-Core`)
— `derive_locator()` dòng 144–160, `data_class="bulk"` hard-code dòng 297,
comment `TODO(cid-anchoring)` dòng 72–75.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời
USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291; GS-LAMPNET-01:
Application No. 64/031,472._
