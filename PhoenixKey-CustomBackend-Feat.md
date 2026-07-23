# PhoenixKey — Nguồn dữ liệu chain tự chọn (Custom Backend)

> **Tài liệu này viết cho ai:** người dùng PhoenixKey, đội sản phẩm, VÀ kỹ sư tích hợp — gộp chung vì đây là một tính năng hạ tầng nhỏ, không phải một module trong 8 module chính. Mục tiêu: hiểu **vì sao ví cần cho chọn nguồn dữ liệu chuỗi riêng**, **cơ chế hoạt động**, **bảo mật ra sao**, và **giới hạn thật của thiết kế**.
> **Thuộc hạ tầng ví:** dùng chung bởi Ví Phượng hoàng (Phoenix) lẫn Ví tiêu chuẩn (Standard) — xem [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md) cho tổng quan ví. **Ngày:** 2026-07-14.
> Đối chiếu code thật: `Enclave/lib/services/backend_config_service.dart`, `Enclave/lib/screens/backend_settings_screen.dart`, `Enclave/lib/bridge/cardano_tx_builder.dart` — nhánh `claude/custom-backend-config`.
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md).

---

## 1. Một câu là gì

**Bạn được TỰ CHỌN nơi ví PhoenixKey đọc số dư/UTxO và gửi giao dịch Cardano — dùng Blockfrost mặc định của app, hoặc tự nhập một nguồn tương thích Blockfrost của riêng bạn — thay vì bị BẮT BUỘC đi qua một nhà cung cấp duy nhất do GreenSun Tech kiểm soát.**

Hình dung đơn giản: ví crypto thường như một cái điện thoại chỉ gọi được qua đúng MỘT nhà mạng — nhà mạng đó sập, chậm, hoặc chặn số của bạn là bạn hết cách. Tính năng này là khe cắm SIM: mặc định dùng SIM có sẵn (Blockfrost), nhưng bạn lắp SIM khác được nếu muốn tự chủ hơn.

---

## 2. Vấn đề nó giải

Một ví Cardano thật sự "cộng đồng" (không phải ví-của-một-công-ty-đóng-kín) phải đáp ứng một tiêu chí quen thuộc với các ví mã nguồn mở lớn trên Cardano (Eternl, Typhon, Nami…): **người dùng không bị khoá cứng vào một nhà cung cấp dữ liệu chuỗi duy nhất.**

Trước tính năng này, `CardanoTxBuilder` hardcode thẳng URL Blockfrost + một API key do GreenSun Tech sở hữu cho MỌI lần đọc UTxO, đọc protocol params, và gửi giao dịch. Ba hệ quả:

1. **Điểm lỗi duy nhất (single point of failure).** Blockfrost rớt/rate-limit/thay đổi chính sách → toàn bộ ví PhoenixKey ngưng hoạt động, không ai tự cứu được.
2. **Không có lựa chọn tự chủ.** Ai muốn tự chạy node riêng (dbsync-api, Blockfrost self-host, hoặc dùng project Blockfrost riêng của họ) để không phụ thuộc key của GreenSun Tech — trước đây không có đường nào làm vậy.
3. **Không đạt tiêu chí "ví cộng đồng mở".** Đây là một tiêu chí thường được nhắc tới khi đánh giá ví Cardano nghiêm túc, độc lập với hệ sinh thái PhoenixKey đang cố đạt.

Tính năng này thêm một khe cắm cấu hình, KHÔNG đổi hành vi mặc định — không cấu hình gì thì ví chạy y hệt trước đây.

---

## 3. Ba nguồn để chọn

Màn **Cài đặt → Nguồn dữ liệu chain** (`BackendSettingsScreen`) cho chọn giữa ba lựa chọn:

| Nguồn | Cần cấu hình? | Trạng thái |
|---|---|---|
| **Blockfrost (mặc định)** | Không — dùng key hardcode sẵn của app | Hoạt động, KHÔNG đổi so với trước |
| **Tự nhập (Custom)** | Có — Base URL + API key/project_id, theo TỪNG network | Hoạt động |
| **LampNet (sắp có)** | — | Chỉ có mặt trong danh sách, Ô CHỌN BỊ KHOÁ (disabled) |

**Vì sao LampNet bị khoá:** LampNet hiện mới phục vụ sao lưu danh tính (backup), CHƯA phục vụ đọc UTxO hay gửi giao dịch. Chừa chỗ trong enum + UI để khi LampNet có năng lực đó, chỉ cần mở khoá ô chọn — không phải thiết kế lại màn hình hay đổi schema lưu trữ.

**Vì sao chỉ hỗ trợ dạng "tương thích Blockfrost":** đây là giao thức HTTP đơn giản (base URL + 1 header `project_id`) mà nhiều dịch vụ implement lại được (self-hosted dbsync-api, Blockfrost-clone, một project Blockfrost khác). Có giao thức khác hẳn về bản chất (như Ogmios/Kupo, giao thức WebSocket theo kiểu khác) — bản này CHỦ ĐỘNG không cố hỗ trợ, vì mục tiêu là "cho người dùng tự chọn nguồn", không phải "hỗ trợ mọi giao thức tồn tại". Có thể mở rộng sau nếu nhu cầu thật xuất hiện.

---

## 4. Cơ chế — theo từng network, có phòng thủ hai tầng

### 4.1 Cấu hình tách theo network
Cấu hình "Tự nhập" được lưu **riêng cho từng network** (preprod / mainnet / preview — đúng 3 network PhoenixKey đang chạy). Bạn có thể tự nhập nguồn cho preprod (để thử nghiệm) mà không đụng tới mainnet, hoặc ngược lại. Network nào chưa cấu hình thì tự rơi về Blockfrost mặc định cho ĐÚNG network đó — không bao giờ vì thiếu cấu hình 1 network mà cả ví liệt.

### 4.2 Bắt buộc "Kiểm tra kết nối" trước khi Lưu
Nút **Lưu** chỉ bật SAU KHI bấm **Kiểm tra kết nối** và nhận kết quả thành công (gọi thử `GET /blocks/latest` với đúng URL + key bạn nhập, timeout 10 giây). Đổi lại Base URL/API key sau khi test → phải test lại. Mục đích: tránh lưu một URL sai/gõ nhầm rồi ví không hoạt động mà không rõ vì sao.

### 4.3 Endpoint thực tế dùng khi gửi/đọc giao dịch
Mỗi lần cần đọc UTxO, đọc protocol params, hoặc gửi giao dịch trực tiếp, `CardanoTxBuilder` gọi `resolveActiveEndpoint`, xác định endpoint theo thứ tự:
1. Nếu bạn chọn **Tự nhập** VÀ đã lưu cấu hình hợp lệ cho ĐÚNG network đang chạy → dùng cấu hình đó.
2. Mọi trường hợp khác (chưa cấu hình / chọn Blockfrost / chọn LampNet chưa wiring / cấu hình cũ vi phạm chính sách bảo mật — xem §5.2) → rơi về Blockfrost mặc định.

Luồng kích hoạt (activation) vẫn nộp giao dịch đã ký qua backend GreenSun Tech (`/activation/{id}/submit-tx`) NHƯ CŨ — nguồn tự chọn chỉ thay đổi nơi ĐỌC UTxO/protocol params cho luồng đó, không đổi nơi NỘP. Chỉ hàm gửi trực tiếp (`submitViaBlockfrost`, dùng cho ví thường ngoài luồng kích hoạt) mới gửi cả tx đã ký qua đúng endpoint bạn chọn.

---

## 5. Bảo mật

### 5.1 Lưu trữ
Base URL + API key lưu trong **Keychain (iOS) / Keystore (Android)** qua `flutter_secure_storage`, với `KeychainAccessibility.first_unlock_this_device` — dữ liệu chỉ đọc được sau khi thiết bị đã unlock ít nhất một lần kể từ khi khởi động, và **KHÔNG đồng bộ iCloud Keychain** sang thiết bị khác (đúng khoá bí mật, không rời máy vô ý qua backup cloud).

### 5.2 Chính sách https bắt buộc — hai tầng phòng thủ độc lập
Gửi API key + dữ liệu UTxO/giao dịch qua `http://` (không mã hoá) tới một host công khai là rủi ro nghe lén rõ ràng trên mạng công cộng (Wi-Fi quán cà phê, MITM). Chính sách: **`https://` bắt buộc cho mọi host công khai; `http://` chỉ được chấp nhận khi host chắc chắn là loopback hoặc mạng LAN riêng** (127.0.0.1, `localhost`, dải RFC 1918: 10.x.x.x / 172.16-31.x.x / 192.168.x.x) — trường hợp bạn tự chạy node ngay trong mạng riêng của mình, tự chịu rủi ro mạng đó.

Luật này được kiểm tra ở **hai nơi độc lập**, không phải một:
- **Tầng UI** — `testConnection` (nút Kiểm tra kết nối) chặn ngay nếu vi phạm, trước khi cho bấm Lưu.
- **Tầng service** — `saveCustomConfig` tự kiểm tra lại, KHÔNG tin rằng caller đã đi qua tầng UI trước đó (phòng trường hợp một đường code khác trong tương lai — màn debug, import/export cấu hình, refactor lỡ tay — gọi thẳng hàm lưu mà bỏ qua màn Cài đặt).
- **Khi đọc lại để dùng** — `resolveActiveEndpoint` KHÔNG tin thẳng dữ liệu đã có sẵn trong storage: một cấu hình lưu từ TRƯỚC KHI có luật này (phiên bản app cũ) vẫn có thể còn `http://` công khai trong bộ nhớ máy; đọc lại phải tự kiểm tra lần nữa — vi phạm thì bỏ qua, rơi về Blockfrost mặc định, thay vì "tin" vì nó đã từng được lưu.

### 5.3 Kiểm thử
37/37 test tự động PASS (`Enclave/test/backend_config_service_test.dart`, xác nhận lại bằng `flutter test` khi viết tài liệu này), bao phủ: provider mặc định khi chưa cấu hình, đọc/ghi/cách ly theo network, dữ liệu hỏng trong storage không làm crash, toàn bộ ma trận scheme (https bất kỳ host / http chỉ loopback+LAN / http bị từ chối với host công khai và IP công khai), và các nhánh lỗi của `testConnection` (401/403/404/500/timeout/lỗi mạng).

---

## 6. Hành trình người dùng

1. Vào **Cài đặt → Nguồn dữ liệu chain**. Thấy 3 lựa chọn, mặc định đang chọn "Blockfrost (mặc định)".
2. Chọn **Tự nhập** → hiện khối cấu hình theo network (mặc định preprod), nhập Base URL + API key/project_id.
3. Bấm **Kiểm tra kết nối** → chờ tối đa 10 giây → thấy thông báo xanh "Kết nối thành công" hoặc đỏ kèm lý do cụ thể (key sai / URL sai / hết thời gian chờ / lỗi mạng).
4. Chỉ khi thành công, nút **Lưu** mới bật. Bấm Lưu → cấu hình áp dụng ngay cho các lần đọc UTxO/gửi giao dịch tiếp theo trên network đó.
5. Muốn quay lại mặc định: chọn lại "Blockfrost (mặc định)" — không cần xoá cấu hình đã nhập, chỉ cần đổi provider đang chọn.

---

## 7. Quyền lợi và giới hạn của bạn

**Bạn được:**
- Không bị khoá cứng vào một nhà cung cấp dữ liệu chuỗi duy nhất.
- Cấu hình độc lập theo từng network, không sợ vỡ luồng vì thiếu 1 network.
- Validate sớm (test trước khi lưu) — tránh tự làm ví của mình không hoạt động.

**Bạn cần hiểu:**
- Đây là tính năng cho người dùng **có nhu cầu và hiểu rõ hệ quả** khi tự chọn một nguồn dữ liệu ngoài GreenSun Tech kiểm soát — xem §8 dưới đây trước khi dùng cho ví giữ giá trị lớn.
- Nguồn "Tự nhập" chỉ nhận dạng endpoint tương thích Blockfrost — không phải mọi loại node/API Cardano.

---

## 8. Ranh giới trung thực / rủi ro còn lại

Thiết kế này giải quyết đúng vấn đề "không phụ thuộc một nhà cung cấp" — nhưng "tự chọn nguồn" tự nó mở ra một lớp rủi ro mới, không phải phép thuật miễn phí. Ba điểm sau CHƯA có giải pháp trong bản này, cần biết trước khi dùng:

- **Nguồn tự nhập không trung thực có thể làm bạn trả phí thật cao hơn cần thiết — không phải chỉ "làm mất kết nối".** `CardanoTxBuilder`/rust core tính phí giao dịch bằng công thức tuyến tính `fee = min_fee_a + min_fee_b × kích thước tx`, lấy thẳng `min_fee_a`/`min_fee_b`/`coins_per_utxo_size` từ **protocol params do chính endpoint bạn cấu hình trả về** — KHÔNG có mức trần sanity-check nào đối chiếu với dải giá trị hợp lý thực tế của Cardano. Một nguồn tự nhập ác ý (hoặc bị tấn công) có thể trả về `min_fee_a`/`min_fee_b` bị thổi phồng, khiến ví tính và KÝ một giao dịch trả phí mạng cao hơn nhiều lần mức cần thiết — số ADA đó là THẬT, mất thật khi giao dịch được chấp nhận (kể cả khi giao dịch cuối cùng vẫn được NỘP qua backend GreenSun Tech đáng tin, như luồng activation — vì backend không kiểm tra lại "phí này có hợp lý không", nó chỉ tiếp nhận đúng tx đã ký). Đây là đường tấn công gây thiệt hại tiền thật cụ thể nhất trong thiết kế hiện tại — chưa có giải pháp (vd: đối chiếu chéo với 1 nguồn tham chiếu, hoặc đặt trần sanity-check hợp lý cho các tham số phí) ở bản này.
- **Không có cách nào xác minh độc lập rằng dữ liệu UTxO/số dư nguồn tự nhập trả về là đúng sự thật trên chuỗi** (không SPV, không đối chiếu chéo nhiều nguồn) — đây là mô hình tin-tưởng-nguồn-đã-chọn giống mọi ví cho phép custom endpoint khác (Eternl, Typhon…), không phải một khiếm khuyết riêng của PhoenixKey. Hệ quả thực tế của việc trỏ vào một nguồn không đáng tin: nguồn đó biết TOÀN BỘ địa chỉ, UTxO, số dư, và mọi giao dịch bạn gửi qua nó (rủi ro lộ danh tính/liên kết ví — deanonymization), có thể từ chối phục vụ hoặc trả dữ liệu sai khiến bạn không dựng được giao dịch (từ chối dịch vụ). Nó KHÔNG thể tự ý đổi người nhận/số tiền để rút trộm — địa chỉ nhận, số lượng, và chữ ký đều do bạn/thiết bị bạn quyết định — và nếu nó bịa ra UTxO không tồn tại thật, mạng Cardano thật sẽ từ chối khi giao dịch được nộp thật. `https://` chỉ đảm bảo đường truyền được mã hoá, KHÔNG đảm bảo máy chủ ở đầu kia đáng tin.
- **`testConnection` cũng gửi API key thật tới host bạn nhập, TRƯỚC CẢ KHI bạn bấm "Lưu".** Chỉ cần gõ nhầm một domain giả trông giống nguồn Blockfrost-tương-thích và bấm "Kiểm tra kết nối" (chưa từng bấm Lưu) là API key đã rời máy. `https://` không chặn được trường hợp domain đó CHÍNH LÀ kẻ tấn công — chứng chỉ TLS hợp lệ chỉ chứng minh đúng tên miền, không chứng minh máy chủ đó tử tế. Thêm nữa, màn Cài đặt hiện KHÔNG hiển thị "đang dùng nguồn nào" tại thời điểm ký một giao dịch cụ thể — dễ quên rằng mình từng chuyển sang Tự nhập cho một network, rồi vô tình gửi một giao dịch quan trọng qua nguồn đó thay vì Blockfrost mặc định.

Cả ba điểm trên là đánh đổi cố hữu của việc cho phép "tự chọn nguồn dữ liệu" — không phải lỗi lập trình có thể vá bằng một dòng code, mà là câu hỏi thiết kế còn mở (có cần trần sanity-check cho phí, có cần cảnh báo rõ hơn ở màn Kiểm tra kết nối, có cần hiển thị nguồn đang dùng ở màn ký giao dịch hay không) — chưa chốt trong bản này.

---

## Nguồn

Code: `Enclave/lib/services/backend_config_service.dart`, `Enclave/lib/screens/backend_settings_screen.dart`, `Enclave/lib/bridge/cardano_tx_builder.dart`, `Enclave/rust_core/src/transfer.rs` (công thức phí). Test: `Enclave/test/backend_config_service_test.dart` (37/37 PASS). Nhánh: `claude/custom-backend-config`.
Tài liệu liên quan: [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Wallet-API-v2-Feat.md](./PhoenixKey-Wallet-API-v2-Feat.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
