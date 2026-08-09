# PhoenixKey — Đặc tả màn hình tích hợp

> Tài liệu này mô tả **màn hình + luồng dữ liệu + luồng hành động** cho 5 luồng người dùng phổ biến khi một app tích hợp PhoenixKey (đăng nhập, tạo danh tính, ví, ký giao dịch qua trung gian, người bảo trợ). Mục tiêu: dev dựng được UI mà không phải đoán shape API hay hành vi lỗi.
>
> Mọi thứ trong tài liệu này đọc trực tiếp từ mã nguồn (`PhoenixKey-Database`) hoặc tài liệu API đã có (`API.md`, `PhoenixKey-SDK/INTEGRATION.md`). Chỗ nào chưa xác nhận được từ hai nguồn đó, tài liệu ghi thẳng **CHƯA CÓ** — đừng dựng UI lên một endpoint không tồn tại.

## 0. Quy ước chung

- **Base URL prod:** `https://api.phoenixkey.me/api/v1`. Mọi đường dẫn trong tài liệu này viết tắt bỏ base, ví dụ `/identity/register` nghĩa là `https://api.phoenixkey.me/api/v1/identity/register`.
- **Response wrapper:** mọi response bọc trong `{ code, message, result? }`; `code = 1000` là thành công.
- **⚠ Định dạng trường trên dây là `snake_case`, KHÔNG phải `camelCase`.** Backend cấu hình Jackson `SNAKE_CASE` toàn cục. Trường một-từ (`nonce`, `action`) thì hai cách viết trùng nhau nên dễ bị bỏ qua lỗi; trường nhiều-từ (`user_did`, `owner_signature`, `amount_lamp`) sai cách viết sẽ rớt validate âm thầm. Hai ngoại lệ giữ `camelCase`: W3C DID Document và JWKS (theo chuẩn ngoài quy định). Nguồn: `PhoenixKey-SDK/INTEGRATION.md` §"Quy ước đặt tên trên dây".
- **Số lượng on-chain (đơn vị nhỏ nhất) luôn là JSON string**, không phải number — `lovelace`, `lamp`, `carp`, `magic.available`, `magic.accrued`, `amount_lamp`... Tổng cung LAMP (3,6×10¹⁶ oildrop) vượt `Number.MAX_SAFE_INTEGER`. Client phải parse bằng `BigInt`. Trường slot/ngày/đếm vẫn là number thường.
- **Auth:** `Authorization: Bearer <jwt>` cho endpoint có 🔒. Ba loại token:

| Loại | Thời hạn | Cấp bởi | Dùng cho |
|---|---|---|---|
| `temp` | 5 phút (300s) | `POST /auth/session/init` | SSE stream + `/status` fallback trong lúc chờ QR được quét |
| `session` | 1 giờ (3600s) | `POST /auth/session/{id}/approve` | Mọi API mutation cần đăng nhập |
| `linked-device` | 30 ngày | `POST /auth/session/{id}/approve` | Đăng nhập lần sau bằng push thay vì quét lại QR |

  Nguồn TTL: `PhoenixKey-Database/src/main/resources/application.yml:93,95` (session=3600s, challenge=300s) và `PhoenixKey-Database/src/main/java/com/magiclamp/phoenixkey_db/security/JwtServiceImpl.java:60` (`LINKED_DEVICE_TTL = Duration.ofDays(30)`).

- **Trạng thái sẵn sàng** dùng 3 nhãn: **DÙNG ĐƯỢC** (gọi được, có nghiệp vụ thật) · **KHUNG — chưa nối** (route tồn tại nhưng trả `9501` "chưa cài đặt", hoặc trả dữ liệu rỗng/giả cho tới khi phần phụ thuộc xong) · **CHƯA CÓ** (không tìm thấy route/khả năng nào, không đoán).

---

## 1. Đăng nhập bằng PhoenixKey (QR pairing)

### Mục đích

Người dùng đăng nhập vào một web/app khác bằng chính danh tính PhoenixKey trên điện thoại — không nhập mật khẩu, quét QR + xác nhận sinh trắc trên máy của họ.

### Wireframe (trạng thái chính)

```
[ Logo / tiêu đề: "Đăng nhập bằng PhoenixKey" ]
[ Khung QR code ]
[ Đồng hồ đếm ngược  04:59 → 00:00 ]
[ Dòng phụ: "Mở app PhoenixKey trên điện thoại và quét mã này" ]
[ Link nhỏ: "Không quét được? Tạo mã mới" ]
```

### Các trạng thái màn hình

- **Trống**: trang đăng nhập, nút "Đăng nhập bằng PhoenixKey" chưa bấm.
- **Đang tải (khởi tạo)**: gọi `/auth/session/init`, spinner ngắn.
- **QR hiển thị + đếm ngược**: đã có QR, đang mở SSE, chờ mobile duyệt.
- **Thành công**: nhận `session_token`, chuyển vào khu vực đã đăng nhập.
- **QR hết hạn**: quá 5 phút chưa quét → QR cũ vô hiệu, hiện nút "Tạo mã mới".
- **Lỗi riêng**:
  - `1301 SESSION_NOT_FOUND` (404, HTTP) — `session_id` sai hoặc đã hết hạn.
  - `1303 SESSION_ALREADY_APPROVED` (409) — gọi `/approve` trùng (thường do bug client, hiếm gặp ở web).
  - `1304 UNAUTHORIZED` (401) — Bearer `temp_token` thiếu/sai.
  - `1332 RATE_LIMITED` (429) — vượt 8 lần approve / `session_id` / cửa sổ 1 giờ.
  - Phía mobile (web cần biết để hiểu vì sao mobile báo lỗi): `403` chữ ký sai, `9800` thiếu field khi gọi `/approve`.
  - Mất kết nối SSE: fallback sang polling `/status`.

### Luồng dữ liệu

1. Web: `POST /auth/session/init` — request rỗng.
   Response: `{ session_id, challenge, temp_token, expires_at }`.
2. Web dựng payload QR: `{ v:1, sid: session_id, ch: challenge, dom: <domain của web>, exp: expires_at }`, render thành mã QR.
3. Web: `GET /auth/session/{session_id}/stream` — Header `Authorization: Bearer {temp_token}` — SSE. Đợi event `approved`, data `{ session_token, linked_device_token, user_did }`. Heartbeat `:ping` mỗi 30 giây.
   - Fallback khi SSE reconnect: `GET /auth/session/{session_id}/status` (Bearer `temp_token`) → `{ session_id, status: "pending"|"approved"|"expired", session_token?, linked_device_token?, user_did? }`.
4. Mobile (ngoài UI web nhưng cần biết để hiểu luồng): quét QR → ký `challenge + ":" + domain + ":" + timestamp` (DER ECDSA, lệch giờ ≤ 60s) → `POST /auth/session/{session_id}/approve` `{ user_did, public_key_hex, signature, domain, timestamp }` → nhận `{ status: "approved", session_token, linked_device_token }`.
5. Web nhận `session_token` qua SSE (hoặc `/status`) → dùng làm Bearer cho mọi API tiếp theo.
6. (Tuỳ chọn) Web lưu `linked_device_token` cho lần sau: `POST /auth/session/push` `{ session_id, linked_device_token }` → server đẩy push tới mobile đã liên kết, user chỉ cần duyệt, không cần quét QR nữa.

### Luồng hành động

- Bấm "Đăng nhập bằng PhoenixKey" → gọi `init` → hiện QR + đếm ngược 5:00 → mở SSE song song.
- User mở app PhoenixKey → quét mã (hoặc nhận push nếu đã liên kết trước) → xác thực sinh trắc cục bộ trên máy → app tự ký và gọi `approve`.
- Web nhận sự kiện `approved` → điều hướng vào khu vực đã đăng nhập.
- QR hết hạn (>5 phút chưa quét) → web tự vô hiệu QR, hiện nút "Tạo mã mới" gọi lại `init`.
- **Huỷ giữa dòng**: user đóng modal/rời trang trước khi quét → web chỉ đóng kết nối SSE cục bộ (đóng `EventSource`). **CHƯA CÓ** endpoint huỷ session tường minh phía server — session tự hết hạn theo TTL 300s; không rò rỉ gì vì chưa `approve` thì chưa phát token nào.

### Thời hạn

- `temp_token` / QR challenge: **5 phút**. Hết hạn → phải tạo QR mới.
- `session_token`: **1 giờ**. Hết hạn → mọi API 🔒 trả `1304`; **không có endpoint refresh-token được công bố** — web phải điều hướng lại toàn bộ luồng đăng nhập.
- `linked_device_token`: **30 ngày**.
- SSE heartbeat: `:ping` mỗi **30 giây** — mất heartbeat kéo dài là dấu hiệu kết nối chết, nên fallback `/status`.
- Rate limit `/approve`: tối đa **8 lần / `session_id` / cửa sổ 3600 giây**.

### Trạng thái sẵn sàng

**DÙNG ĐƯỢC** — `PhoenixKey-SDK/INTEGRATION.md:69` xếp `READY`. Xác nhận qua `PhoenixKey-Database/src/main/java/com/magiclamp/phoenixkey_db/controller/SessionController.java` — cả 5 route (`init`, `stream`, `status`, `approve`, `push`) có mã thật, không phải khung.

---

## 2. Tạo DID (danh tính cá nhân)

### Mục đích

Người dùng tạo danh tính PhoenixKey lần đầu: sinh cặp khoá trong Secure Enclave của thiết bị, đăng ký lên Cardano, nhận `did:phoenix:<slot>:<hash>`.

> **Phạm vi**: đây là thao tác **chỉ thực hiện được trên app PhoenixKey (mobile)** — khoá gốc sinh và ở lại trong Secure Enclave của điện thoại. Một app/web khác **không tự tạo DID hộ user được**; chỉ có thể điều hướng/deep-link user sang app PhoenixKey để họ tự làm, tương tự cách luồng "Đăng nhập" dùng QR ở mục 1. Mô tả dưới đây là màn hình TRÊN APP PhoenixKey — dev app khác cần biết chính xác để hiểu deep-link dẫn tới cái gì.

### Wireframe

```
[ Tiêu đề: "Tạo danh tính PhoenixKey" ]
[ Icon khiên / Secure Enclave ]
[ Đoạn mô tả ngắn: khoá không rời máy ]
[ Nút chính: "Tạo danh tính" ]

--- sau khi thành công ---
[ Icon xác nhận ]
[ DID hiển thị rút gọn + nút copy ]
[ tx_hash + link explorer ]
[ Nút: "Tiếp tục" ]
```

### Các trạng thái màn hình

- **Trống**: màn giới thiệu, chưa bấm.
- **Đang sinh khoá** (cục bộ, không cần mạng).
- **Đang đăng ký** (đang publish lên Cardano — có độ trễ, có thể vài giây tới vài chục giây).
- **Thành công**: hiện `user_did` + `tx_hash`.
- **Lỗi riêng**:
  - `9800` (400) — thiếu field / sai định dạng (`public_key_hex` owner PHẢI là P-256 uncompressed: `04` + 128 hex).
  - `1403 SIGNATURE_INVALID` (403) — `added_by_signature` không khớp — thường là lỗi Enclave, nên cho retry bằng cách sinh khoá mới.
  - `3005 KEY_ALREADY_REGISTERED` (409) — khoá này đã dùng để đăng ký DID khác; không nên xảy ra với khoá vừa sinh, chỉ ra bug tái dùng khoá.
  - `1347 NON_PERSON_REGISTER_DEPRECATED` (410) — lỗi client (gửi sai `entity_type`); màn này PHẢI luôn gửi `"PERSON"`.
  - `5101 CARDANO_TX_FAILED` (502) — backend publish lên chain thất bại (mạng/Blockfrost) → cho retry.

### Luồng dữ liệu

1. App sinh cặp khoá P-256 trong Secure Enclave, khoá riêng không rời thiết bị.
2. App build message `"PHOENIXKEY_GENESIS:" + public_key_hex + ":" + nonce` → ký DER ECDSA → `added_by_signature`.
3. `POST /identity/register`
   Request: `{ public_key_hex, key_origin: "SECURE_ENCLAVE", key_role: "owner", added_by_signature, taad_public_key_hex?, nonce, entity_type: "PERSON", owner_did: null }`
   Response: `{ user_id, user_did, tx_hash }`.
4. Backend tự derive địa chỉ ví Phoenix custody theo DID (app không cần gọi thêm gì) — địa chỉ này xuất hiện sau đó ở `GET /wallet/{did}/all` dưới `kind: "phoenix"` (xem mục 3).

### Luồng hành động

- Bấm "Tạo danh tính" → app kiểm tra Enclave khả dụng → yêu cầu xác thực sinh trắc/PIN cục bộ trước khi sinh khoá → sinh khoá → gọi `register` → loading → thành công hiện DID, gợi ý bước tiếp theo (thêm người bảo trợ — mục 5).
- **Huỷ giữa dòng**: nếu user thoát app khi request đang treo, không có cách huỷ request đã gửi. Khi mở lại app: nếu request trước đó thực ra đã thành công trên server nhưng app chưa nhận response, gọi lại `register` với CÙNG khoá sẽ trả `409 KEY_ALREADY_REGISTERED`. **CHƯA CÓ** endpoint "tra DID theo public key" công khai để app tự phục hồi trạng thái trong trường hợp này — app nên tự lưu "đang chờ xác nhận" cục bộ TRƯỚC khi gọi API để không mất dấu nếu mạng rớt giữa chừng.

### Thời hạn

Không có TTL phía server cho request này — chữ ký gắn với `nonce` dùng-một-lần (chặn tái dùng, không hết hạn theo thời gian).

### Trạng thái sẵn sàng

**DÙNG ĐƯỢC** — `PhoenixKey-SDK/INTEGRATION.md:68` xếp `READY`. Verify qua `PhoenixKey-Database/src/main/java/com/magiclamp/phoenixkey_db/controller/IdentityController.java:60-71` và `service/identity/IdentityServiceImpl.java` (các dòng ném lỗi thật: 101 `KEY_ALREADY_REGISTERED`, 110 `NON_PERSON_REGISTER_DEPRECATED`, 126 `CARDANO_TX_FAILED`, 163 `SIGNATURE_INVALID`).

---

## 3. Ví — số dư + gửi tiền

### Mục đích

Người dùng xem số dư ADA/LAMP/CARP (và MAGIC khi sẵn sàng) và gửi tiền đi.

### ⚠ Hai loại ví — KHÁC NHAU về khả năng, đừng vẽ giống nhau

| | Ví Standard | Ví Phoenix |
|---|---|---|
| Nguồn gốc địa chỉ | Dẫn từ seed (CIP-1852, chuẩn HD wallet ngoài) | Dẫn từ DID (script address theo `did_payment`) |
| Nhận tiền | Có | Có |
| Xem số dư | Có | Có |
| **Gửi tiền hôm nay** | **Có** (ký bằng CIP-30 hoặc ví cục bộ) | **KHÔNG — chỉ xem** |
| Vì sao chưa gửi được | — | Thiết kế đích cần **2 chữ ký** (controller + khoá thiết bị). Khoá thiết bị tuy đã có API lưu (`POST /identity/{did}/device-key`) nhưng **chưa có client nào build được** giao dịch tiêu ví này bằng 2 chữ ký, và validator 2-of-2 chưa deploy — ví Phoenix đang sống trên chain vẫn dùng script 1-of-1 cũ |
| Giá trị `kind` trong `GET /wallet/{did}/all` | `"standard"` | `"phoenix"` |

Bằng chứng ví Phoenix view-only: `PhoenixKeyDID/Wallet/src/components/wallet/PhoenixCustodyPanel.tsx:12-17` (docstring trong code): *"View-only in v1: the address + public balances come from the backend; spending needs the controller key (mobile / air-gap), which is phase 2."* Không tồn tại nút "Gửi" nào gắn với ví Phoenix trong repo `Wallet` lẫn `PhoenixKey-Core` hiện tại.

⇒ **Bắt buộc ở tầng giao diện**: card ví Phoenix không có nút "Gửi" (ẩn hoặc disabled kèm giải thích). Card ví Standard có đủ luồng gửi. Hai card KHÔNG được trông giống hệt nhau.

### 3.1 Xem tất cả ví

**Wireframe:**
```
[ Tab "Ví" ]
[ Card "Ví Standard" ]
    Số dư: ADA · LAMP · CARP
    [ Nhận ]  [ Gửi ]
[ Card "Ví Phượng hoàng (Phoenix)" ]
    Số dư: ADA · LAMP · CARP
    [ Nhận ]                      ← không có nút Gửi
    Banner: "Chỉ nhận — gửi sẽ mở sau"
[ Mục MAGIC ]
    Số MAGIC khả dụng (hiện luôn 0)
```

**Các trạng thái màn hình**: trống (chưa có ví nào, hiếm) · đang tải · thành công (liệt kê card theo `wallets[]`) · lỗi `1304 UNAUTHORIZED` (caller ≠ path DID — bug client, không nên lộ ra cho user) · lỗi mạng chung.

**Luồng dữ liệu**: `GET /wallet/{user_did}/all` 🔒 Bearer `session` — path `user_did` PHẢI khớp DID trong token (khác → 401).

Response:
```json
{
  "wallets": [
    { "kind": "phoenix",  "addresses": { "fixed": "addr_test1w..." }, "balances": { "lovelace": "500000", "lamp": "100", "carp": "50" } },
    { "kind": "standard", "addresses": { "fixed": "addr...", "active": "addr...", "stake": "stake..." }, "balances": { "lovelace": "3000000", "lamp": "800", "carp": "100" } }
  ],
  "magic": { "source": "vault", "available": "0", "accrued": "0" }
}
```

`lovelace`/`lamp`/`carp`/`magic.available`/`magic.accrued` là **chuỗi**, parse bằng `BigInt`. Ví Phoenix chỉ có `addresses.fixed` — **không có trường `addresses.custody`** trong contract backend (`WalletDtos.java` → record `StandardAddresses(fixed, active, stake)`, Phoenix chỉ điền `fixed`).

**Luồng hành động**: vào tab Ví → tự động gọi `/all` → hiện card theo dữ liệu trả về → bấm card Standard vào chi tiết + Gửi/Nhận; bấm card Phoenix vào chi tiết CHỈ có Nhận.

**Thời hạn**: theo `session_token` (1 giờ) — hết hạn thì màn Ví yêu cầu đăng nhập lại.

**Trạng thái sẵn sàng**: **DÙNG ĐƯỢC** — `PhoenixKey-SDK/INTEGRATION.md:70` "READY (string-serialize: Database PR #102 đã merge 2026-07-30)". Verify `WalletController.java:120-131` (`getAllWallets`).

### 3.2 Đăng ký ví Standard

Bước cần trước khi ví Standard xuất hiện trong `/all`, nếu user chưa từng đăng ký.

**Các trạng thái màn hình**: trống · đang derive địa chỉ (cục bộ, không cần mạng) · đang gửi đăng ký · thành công · lỗi riêng:
- `403 WALLET_PAYMENT_SIGNATURE_INVALID` (1326) — chữ ký hoặc key không khớp địa chỉ.
- `409 WALLET_FIXED_ADDRESS_LOCKED` (1325) — cố đổi `fixed_address` sau khi đã đăng ký (bất biến).
- `409 NONCE_ALREADY_USED` (3006).
- `404 USER_DID_NOT_FOUND` (2002).

**Luồng dữ liệu**:
1. Client (mobile `dart:ffi` hoặc ví web có seed cục bộ) derive địa chỉ CIP-1852 account 0 (`fixed_address`, bắt buộc) + tuỳ chọn `active_address`/`stake_address`.
2. Ký challenge `"PHOENIXKEY_WALLET_STANDARD_REGISTER:" + user_did + ":" + fixed_address + ":" + nonce` bằng payment private key (Ed25519).
3. `POST /wallet/standard/register` 🔒 Bearer `session`
   Request: `{ fixed_address, active_address?, stake_address?, payment_public_key_hex, signature, nonce }`
   Response: `{ code: 1000, message: "Standard wallet registered" }`.
   Idempotent theo `fixed_address` — gọi lại được để cập nhật `active_address`/`stake_address`, KHÔNG đổi được `fixed_address`.

**Luồng hành động**: user bấm "Kích hoạt ví Standard" (nếu chưa có) → app derive địa chỉ cục bộ → gọi register → thành công → quay lại màn Ví, card Standard xuất hiện.

**Thời hạn**: `nonce` không hết hạn theo thời gian, chỉ dùng-một-lần.

**Trạng thái sẵn sàng**: **DÙNG ĐƯỢC** — `WalletController.java:96-105`.

### 3.3 Gửi tiền — CHỈ ví Standard

**Wireframe (màn xem lại — bắt buộc trước khi ký):**
```
[ Tiêu đề: "Xem lại giao dịch" ]
[ Người nhận #1: địa chỉ ĐẦY ĐỦ + nút copy ]
[ Số lượng ADA / từng token ]
[ Phí ước tính ]
[ Tổng cộng ]
[ Ô nhập lại đuôi địa chỉ để xác nhận ]
[ Checkbox: "Tôi đã kiểm tra địa chỉ" ]
[ Nút [Huỷ]      [Xác nhận gửi] ]
```

**Các trạng thái màn hình**: trống (form nhập người nhận) · đang tải (build tx: lấy UTXO + protocol params + tip slot) · **màn xem lại** (bắt buộc hiện đủ địa chỉ/số lượng/phí trước khi ký — chống ký-mù) · đang ký & gửi · thành công (`cardano_tx_hash`) · lỗi riêng:
- lỗi client-side "không có UTXO" (ví rỗng).
- lỗi client-side "số lượng không hợp lệ".
- `502 CARDANO_TX_FAILED` (5101) — Blockfrost từ chối / mạng lỗi, chi tiết trong `message`.
- huỷ ở bước xem lại (nút "Huỷ" quay lại form, không gọi API).

**Luồng dữ liệu**:
1. Client đọc UTXO từ ví CIP-30 đã kết nối (`getUtxos()`) — bước này KHÔNG qua backend PhoenixKey.
2. Client build tx cục bộ (chọn input, tính phí, output theo min-UTxO), lấy protocol params + tip slot từ nguồn Cardano (không phải PhoenixKey backend), set `ttl = tip + 7200` slot (~2 giờ — TTL Cardano-native, KHÔNG phải TTL của PhoenixKey).
3. Hiện màn xem lại: từng người nhận/ADA/token + tổng phí — user tick "đã kiểm tra địa chỉ" + gõ lại đuôi địa chỉ người nhận đầu tiên (chống address-poisoning).
4. Ví CIP-30 ký (`signTx`) → client merge witness → `POST /wallet/tx/submit` 🔒 Bearer `session`
   Request: `{ signed_tx_cbor }`
   Response: `{ cardano_tx_hash }`.

**Luồng hành động**: điền người nhận + số lượng → bấm "Xem lại" → build tx → hiện xem lại đầy đủ → tick xác nhận + gõ lại đuôi địa chỉ → bấm "Xác nhận gửi" → popup ví ký → submit → thành công hiện `cardano_tx_hash`, quay lại form trống. Nhánh huỷ: bấm "Huỷ" ở màn xem lại → quay lại form giữ nguyên input, KHÔNG gọi submit. Nhánh submit lỗi: xoá tx đã build cục bộ (tránh double-witness khi build lại) → yêu cầu build lại từ đầu.

**Thời hạn**: TTL Cardano-native của tx = `tip + 7200` slot (≈2 giờ) — hết hạn thì tx build cũ không submit được nữa, phải build lại (lấy tip mới).

**Trạng thái sẵn sàng**: **DÙNG ĐƯỢC nhưng CHƯA AUDIT BẢO MẬT.** `/wallet/tx/submit` là `READY` (`PhoenixKey-SDK/INTEGRATION.md:82`, Database PR #76 merge 2026-07-24). Giao diện Send đã build đầy đủ ở `Wallet/src/components/wallet/SendPanel.tsx`, nhưng chính docstring dòng 38 ghi: *"⚠️ Beta: not yet security-audited."* CHỈ áp dụng cho ví Standard — client cần đọc UTxO qua ví CIP-30 đã kết nối; ví Phoenix không có đường ký tương đương (xem bảng đầu mục 3).

### 3.4 Ví Phoenix — nhận + xem (không gửi)

**Các trạng thái màn hình**: trống (hiếm, do thiếu cấu hình backend) · đang tải · thành công (hiện địa chỉ `fixed` + số dư, QR để nhận) · lỗi: theo 3.1 (dùng chung endpoint `/all`).

**Luồng dữ liệu**: đọc từ cùng `GET /wallet/{did}/all`, lọc phần tử `kind === "phoenix"`, dùng `addresses.fixed` làm địa chỉ nhận (QR code).

**Luồng hành động**: user bấm card Phoenix → màn chi tiết CHỈ có "Nhận" (QR + copy địa chỉ) + banner giải thích lý do chưa gửi được, dùng ngôn ngữ người dùng hiểu được (kiểu: "Ví Phượng hoàng hiện chỉ nhận, tính năng gửi sẽ mở sau"), không phơi chi tiết kỹ thuật 2-of-2/validator ra UI.

**Trạng thái sẵn sàng**: **CHƯA CÓ** (gửi) — bằng chứng `Wallet/src/components/wallet/PhoenixCustodyPanel.tsx:16-17`. Xem/nhận **DÙNG ĐƯỢC** như 3.1.

### 3.5 MAGIC trong màn Ví

**Trạng thái sẵn sàng**: **KHUNG — chưa nối.** Trường `magic` luôn trả về nhưng `available`/`accrued` cố định `"0"` (`PhoenixKey-SDK/INTEGRATION.md:81`: *"Trường `magic` trong `GET /wallet/{did}/all` hiện trả 0"*). UI nên hiện mục MAGIC nhưng đừng coi số 0 là lỗi, và đừng build màn "quy đổi/tiêu MAGIC" cho tới khi có route đọc/ghi riêng đổi trạng thái này.

---

## 4. Ký giao dịch qua trung gian (Sign-relay)

### Mục đích

Một ứng dụng bên thứ ba (web dApp, dịch vụ tích hợp PhoenixKey) cần người dùng ký một hành động (chuyển khoản, xoay khoá, xuất seed...) mà không giữ khoá của họ — app tạo "ý định ký" (intent), người dùng duyệt trên chính điện thoại của mình, app nhận lại chữ ký qua kênh đã mở sẵn.

### Điều kiện tiên quyết — QUAN TRỌNG

App phải đã có `session_id` + kênh SSE **đang mở** từ luồng Đăng nhập (mục 1). Sign-relay **không mở SSE riêng** — nó phát sự kiện `signed`/`cancelled` LÊN CÙNG kênh `GET /auth/session/{id}/stream` đã mở lúc đăng nhập (xác nhận: `SignRequestServiceImpl.java` dùng chung `SseEmitterRegistry`, khoá theo `session_id` — dòng 226 `sseRegistry.emit(payload.sessionId(), EVENT_SIGNED, ...)`, dòng 274 tương tự cho `EVENT_CANCELLED`). Nếu app đã đóng kênh SSE đó, phải mở lại (chạy lại luồng đăng nhập QR) trước khi tạo sign request.

### Wireframe

```
[ Modal: "Đang chờ xác nhận trên điện thoại của bạn" ]
[ Icon điện thoại ]
[ display_text của intent — vd "Chuyển 100 LAMP đến addr1q..." ]
[ Đồng hồ đếm ngược  01:59 → 00:00 ]
[ Nút: "Huỷ yêu cầu" ]
```

### Các trạng thái màn hình

- **Trống**: chưa có yêu cầu ký nào đang chờ.
- **Đang tạo yêu cầu**: gọi `/sign/request`.
- **Đang chờ mobile duyệt**: đã có `request_id`, đếm ngược 120s, lắng nghe SSE.
- **Thành công**: nhận sự kiện `signed`, hiện chữ ký/kết quả.
- **Đã huỷ**: app tự huỷ (hoặc user bấm "Huỷ yêu cầu").
- **Hết hạn**: quá 120s chưa duyệt.
- **Lỗi riêng**:
  - `403` (Invalid session token) khi tạo request — Bearer sai.
  - `1401 SIGN_REQUEST_NOT_FOUND` (404) — id sai hoặc dữ liệu đã dọn.
  - `1402 SIGN_REQUEST_EXPIRED` (410) — mobile duyệt trễ quá 120s.
  - `1403 SIGNATURE_INVALID` (403) — chữ ký mobile gửi lên không khớp intent (lỗi phía mobile; web cần hiện "duyệt thất bại").
  - `409` (Nonce đã dùng) khi mobile approve trùng.

### Luồng dữ liệu

1. App (đã đăng nhập, có `session_token` + `session_id` đang mở SSE): `POST /sign/request` 🔒 Bearer `session`
   Request: `{ session_id, intent: { type, body, domain, app_id, nonce, timestamp, display_text } }`
   — `type` ∈ `TRANSFER | SEED_EXPORT | KEY_ROTATE | CUSTOM`.
   — `display_text` **bắt buộc**, là câu người-đọc-được app hiện lại y hệt trên mobile (chống ký-mù), vd `"Chuyển 100 LAMP đến addr1q..."`.
   Response: `{ request_id, expires_at }` (TTL 120 giây).
2. Server push notification tới mobile — push **chỉ chứa `request_id`**, không chứa nội dung intent (tránh rò rỉ qua notification service).
3. Mobile: `GET /sign/request/{request_id}` → `{ request_id, user_did, session_id, intent, status: "pending"|"approved"|"rejected"|"expired", expires_at }` — mobile hiện `intent.display_text` cho user xem trước khi ký.
4. Mobile ký JSON canonical của `intent` (key sorted, không khoảng trắng) bằng Hardware Key sau khi user xác thực sinh trắc → `POST /sign/{request_id}/approve` `{ public_key_hex, signature }` → server verify → phát sự kiện `signed` lên kênh SSE của `session_id` gốc.
5. App nhận `signed` qua SSE đang mở → lấy chữ ký, dùng cho mục đích của mình (vd tiếp tục build/submit tx qua `/wallet/tx/submit`, mục 3.3, nếu `type = TRANSFER`).
6. **Huỷ giữa dòng**: App 🔒 Bearer `session`: `POST /sign/{request_id}/cancel` — chỉ chủ request (theo `session_token`) mới huỷ được → phát sự kiện `cancelled` qua SSE cho các listener khác biết.

### Luồng hành động

App gọi API tạo request (thường tự động sau một thao tác khác của user, ví dụ "Xác nhận chuyển khoản") → hiện modal "Đang chờ xác nhận trên điện thoại của bạn" kèm đếm ngược 120s → user mở app, đọc `display_text`, xác thực sinh trắc, duyệt → web nhận sự kiện, đóng modal, tiếp tục luồng nghiệp vụ.

Nếu user **từ chối** trên mobile: trường `status` trong `SignRequestPayload` liệt kê giá trị `"rejected"`, nhưng **CHƯA CÓ** endpoint `POST /sign/{id}/reject` nào trong `SignRequestController.java` — chỉ có `approve` và `cancel` (do phía web/app gọi, không phải mobile). Web KHÔNG nên giả định có sự kiện "reject" riêng phát ra từ mobile; chỉ nên dựa vào hết hạn (`1402`) hoặc `cancel` (do chính app gọi) để thoát modal.

### Thời hạn

Sign request TTL cố định **120 giây** (`application.yml:97`) — hết hạn, modal phải tự đóng và báo "yêu cầu đã hết hạn, thử lại".

### Trạng thái sẵn sàng

**DÙNG ĐƯỢC** — `PhoenixKey-SDK/INTEGRATION.md:78` xếp `READY`. Verify `SignRequestController.java` (4 route: `create`, `get`, `approve`, `cancel` đều có mã thật) + `SignRequestServiceImpl.java` (emit SSE dòng 226, 274).

---

## 5. Người bảo trợ (Guardian)

### Mục đích

Người dùng chỉ định (hoặc gỡ) một DID khác làm "người bảo trợ" — dùng cho khôi phục tài khoản qua mạng lưới xã hội, thay vì chỉ dựa vào seed phrase. Khuyến nghị 3–5 người bảo trợ; ít hơn 3 là không đủ an toàn.

### Wireframe

```
[ Tiêu đề: "Người bảo trợ" ]
[ Cảnh báo nếu < 3: "Cần thêm người bảo trợ để đủ an toàn" ]
[ Danh sách người bảo trợ (DID rút gọn + nút "Gỡ") ]
[ Nút: "+ Thêm người bảo trợ" ]

--- modal thêm ---
[ Ô nhập DID  /  nút quét QR ]
[ Nút [Huỷ]      [Xác nhận] ]
```

### Các trạng thái màn hình

- **Trống**: chưa có người bảo trợ nào (0/3 — cảnh báo chưa đủ an toàn).
- **Đang tải danh sách**: xem lưu ý quan trọng bên dưới — server KHÔNG có endpoint liệt kê.
- **Đang thêm/gỡ**: gọi `/add` hoặc `/remove`.
- **Thành công**: hiện `guardian_count` mới.
- **Lỗi riêng**:
  - `4003 GUARDIAN_SELF_NOT_ALLOWED` (400) — user chọn chính mình làm người bảo trợ.
  - `1403 SIGNATURE_INVALID` (403) — chữ ký `proof_signature` sai.
  - `2002 USER_DID_NOT_FOUND` (404) — `user_did` không tồn tại.
  - `3002 KEY_NOT_FOUND` (404) — user chưa có owner key active để verify chữ ký (tài khoản hỏng, hiếm).
  - `4002 GUARDIAN_ALREADY_EXISTS` (409) — thêm trùng người bảo trợ đã có (chỉ ở `/add`).
  - `4001 GUARDIAN_NOT_FOUND` (404) — gỡ người bảo trợ không tồn tại/đã gỡ (chỉ ở `/remove`).

### ⚠ Khoảng trống quan trọng — đọc trước khi thiết kế màn danh sách

**CHƯA CÓ** endpoint liệt kê danh sách người bảo trợ hiện có của một DID. `GuardianController.java` chỉ có `POST /guardians/add` và `POST /guardians/remove` — cả hai chỉ trả về **số lượng** (`guardian_count`), không trả về **danh sách DID**. Không có `GET /guardians` hay tương đương trong `API.md` lẫn source. Nghĩa là: nếu người dùng đổi máy hoặc mất cache cục bộ, **app không có cách nào hỏi lại server "tôi đang có những ai làm người bảo trợ"**. App phải tự lưu danh sách cục bộ ngay lúc thêm, và đây là nguồn duy nhất — không nên build UI giả định có thể đồng bộ lại từ server.

### Luồng dữ liệu — Thêm người bảo trợ

1. User chọn 1 DID khác (nhập thủ công hoặc quét QR profile — **CHƯA CÓ** cơ chế "tìm bạn" trong API hiện tại, ngoài `GET /identity/by-username/{username}` để resolve username → DID).
2. App ký `"PHOENIXKEY_GUARDIAN_ADD:" + user_did + ":" + guardian_did + ":" + nonce` bằng owner key active hiện tại.
3. `POST /guardians/add`
   Request: `{ user_did, guardian_did, proof_signature, nonce }`
   Response: `{ guardian_count }`.

### Luồng dữ liệu — Gỡ người bảo trợ

1. App ký `"PHOENIXKEY_GUARDIAN_REMOVE:" + user_did + ":" + guardian_did + ":" + nonce` bằng owner key active.
2. `POST /guardians/remove`
   Request: `{ user_did, guardian_did, proof_signature, nonce }`
   Response: `{ guardian_count }` — soft revoke (đổi `status` thành `"revoked"` trong DB), không xoá vĩnh viễn.

### Luồng hành động

Vào màn "Người bảo trợ" → app hiện danh sách đã lưu cục bộ (nguồn duy nhất, xem lưu ý ở trên) → bấm "+ Thêm người bảo trợ" → nhập/quét DID → xác thực sinh trắc cục bộ → app ký + gọi `/add` → thành công: cập nhật `guardian_count` + thêm vào danh sách cục bộ. Bấm "Gỡ" một người bảo trợ trong danh sách → xác thực sinh trắc → gọi `/remove` → thành công: xoá khỏi danh sách cục bộ. Nếu `guardian_count` trả về sau `/remove` nhỏ hơn 3 → app hiện cảnh báo "tài khoản có nguy cơ mất nếu không đủ người bảo trợ để khôi phục".

**Huỷ giữa dòng**: bấm "Huỷ" ở modal nhập DID → không gọi API, không có gì để rollback.

### Thời hạn

`nonce` dùng-một-lần, không hết hạn theo thời gian.

### Trạng thái sẵn sàng

**DÙNG ĐƯỢC** (thêm/gỡ) — `PhoenixKey-SDK/INTEGRATION.md:74` xếp `READY`. Verify `GuardianController.java` + `GuardianServiceImpl.java` (dòng ném lỗi thật: 63, 68, 75, 84, 91, 130, 137, 146, 155).

**CHƯA CÓ**: endpoint liệt kê danh sách người bảo trợ hiện có của một DID (xem cảnh báo phía trên).

---

## Tổng kết những gì CHƯA CÓ (để bàn giao)

| # | Khoảng trống | Ảnh hưởng UI |
|---|---|---|
| 1 | Endpoint huỷ session QR tường minh (mục 1) | Web chỉ đóng SSE cục bộ khi user rời màn; không có gì để dọn phía server, nhưng vô hại vì token chưa phát |
| 2 | Endpoint tra DID theo public key sau khi `/identity/register` treo giữa chừng (mục 2) | App phải tự lưu trạng thái "đang chờ xác nhận" cục bộ trước khi gọi API |
| 3 | Gửi tiền từ ví Phoenix (mục 3.4) | Nút "Gửi" phải ẩn/disabled trên card ví Phoenix; chỉ "Nhận" |
| 4 | `POST /sign/{id}/reject` — mobile từ chối tường minh (mục 4) | Web chỉ dựa vào hết hạn 120s hoặc tự `cancel`, không có sự kiện "reject" riêng từ mobile |
| 5 | `GET /guardians` — liệt kê danh sách người bảo trợ hiện có (mục 5) | App phải tự giữ danh sách cục bộ làm nguồn duy nhất; mất cache cục bộ = mất danh sách |

## Chỗ chưa chắc chắn (đáng kiểm lại trước khi build)

- Mục 4: liệu app có cách nào biết mobile đã MỞ payload (`GET /sign/request/{id}`) nhưng chưa quyết định, để hiện trạng thái trung gian "đang xem trên điện thoại" thay vì chỉ "đang chờ" xuyên suốt 120s? `SignRequestPayload.status` có giá trị `"pending"` nhưng không rõ server có cập nhật riêng một trạng thái "đã mở, chưa quyết" hay không — tài liệu này giữ nguyên 2 trạng thái UI (chờ/xong) cho an toàn.
- Mục 3.1: trường hợp `wallets` rỗng hoàn toàn (cả Phoenix lẫn Standard đều chưa có) — về lý thuyết hiếm vì Phoenix custody derive tự động lúc tạo DID, nhưng có thể xảy ra nếu backend thiếu biến môi trường `PHOENIXKEY_CARDANO_DID_PAYMENT_CBOR_HEX`/`PHOENIXKEY_CARDANO_TAAD_ANCHOR_POLICY` (theo `API.md` §1, ghi chú `POST /identity/register`) — trường hợp này không có mã lỗi riêng, chỉ là mảng rỗng, khó phân biệt với "user thật sự chưa có ví" trên UI.
