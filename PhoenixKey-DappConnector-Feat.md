# PhoenixKey — dApp Connector: mở ví PhoenixKey cho sàn/dApp Cardano

> **Tài liệu này viết cho ai:** người dùng PhoenixKey, đội sản phẩm, và kỹ sư tích hợp — gộp cả phần đời thường lẫn phần kỹ thuật (API surface + lớp bảo mật) vì đây là một cấu phần mới chưa có đặc tả trước đó.
> **Thuộc:** hạ tầng ví (Rebirthme) — dùng chung ví Standard (`m/1852'/1815'/…`) đã có, không dựng ví riêng. **Loại doc:** Feature + kỹ thuật gộp. **Ngày:** 2026-07-14.
> **Nguồn đối chiếu code thật** (nhánh local `claude/dapp-connector-webview`, worktree `wt-dapp-connector/Enclave`):
> `lib/screens/dapp_browser_screen.dart`, `lib/services/dapp_connector_service.dart`, `lib/services/cip30_injection.dart`, `lib/screens/dapp_sign_tx_screen.dart`, `lib/screens/dapp_sign_data_screen.dart`, `rust_core/src/dapp_connector.rs`, `test/dapp_connector_service_test.dart`.
> **Đây là "Hướng-2" đã nhắc trước ở PhoenixKey-Rebirthme-Math.md §4.L (repo module nội bộ — private, chưa public) (I-CONN-3, I-CONN-5) — "PhoenixKey là bên ký cho dApp ngoài", tức PhoenixKey đóng vai **ví**, dApp bên ngoài (Minswap, JPG Store…) là bên **kết nối tới**. Khác "Hướng-1" ở PhoenixKey-Rebirthme-Tech.md §5.7 (repo module nội bộ — private, chưa public) (PhoenixKey là dApp kết nối TỚI ví Lace, dùng cho di cư ví cũ) — hai hướng dùng chung khái niệm CIP-30 nhưng ngược vai, đừng lẫn.
> **Đính chính giả định cũ:** Rebirthme-Tech §5.7-A viết "Mobile không có `window.cardano` → dùng CIP-45 pairing (Phase-2)" — giả định đó áp cho lúc PhoenixKey **là dApp** (Hướng-1). Cho Hướng-2 (tài liệu này), giải pháp đã chọn khác: **WebView riêng + tiêm shim**, không cần CIP-45 — lý do ở mục 3.

---

## 1. Một câu là gì

**dApp Connector là "trình duyệt dApp" riêng trong app PhoenixKey: mở một sàn/dApp Cardano (Minswap, SundaeSwap, JPG Store…) bên trong app, ví PhoenixKey tự xuất hiện như một ví cài sẵn — kết nối, xem số dư, ký giao dịch — không cần cài extension, không cần lộ khoá ra ngoài app.**

Hình dung: bình thường muốn dùng ví PhoenixKey với một sàn DEX, bạn phải copy địa chỉ, chuyển tiền qua ví khác (Lace/Eternl) rồi thao tác ở đó — vòng vèo, và tiền phải "rời" ví PhoenixKey trước đã. dApp Connector giống như **mở một chi nhánh ngân hàng ngay trong siêu thị**: bạn không cần cầm tiền mặt chạy sang ngân hàng, đứng tại quầy siêu thị vẫn giao dịch được với đúng ví của mình.

---

## 2. Vấn đề nó giải

Ba chỗ đau nếu không có dApp Connector:

1. **Ví PhoenixKey bị cô lập khỏi hệ sinh thái DeFi/NFT Cardano.** Muốn swap, mua NFT, stake vào một pool ngoài — phải rút tiền ra ví khác trước, PhoenixKey chỉ là nơi "cất giữ", không "dùng được".
2. **Không có chuẩn nào cho ví trên di động kết nối dApp.** CIP-30 (chuẩn `window.cardano.*` mà mọi dApp Cardano đều gọi) được thiết kế cho **extension trình duyệt desktop** — mobile không có khái niệm "extension chèn JS vào trang ngoài".
3. **Nếu làm ẩu, dễ thành ký mù.** Ghép ví vào một trình duyệt mà không kiểm domain thật, không hiện nội dung trước khi ký — biến thành đúng thứ nguy hiểm nhất trong crypto: bấm ký mà không biết mình ký gì, cho ai.

**dApp Connector giải quyết:**
- Ví PhoenixKey **dùng được trực tiếp** với dApp Cardano, không cần rút tiền ra ngoài.
- Interop qua đúng chuẩn cộng đồng (**CIP-30**) — mọi dApp hỗ trợ Lace/Eternl/Typhon hỗ trợ luôn PhoenixKey, không cần dApp sửa code riêng cho PhoenixKey.
- Không đánh đổi an toàn: domain hiển thị luôn là domain thật (không spoof được), mọi lệnh ký đều qua màn xác nhận hiện rõ nội dung + vân tay/PIN riêng.

---

## 3. Vì sao WebView + tiêm CIP-30, không phải CIP-45

Có hai cách công khai để một ví di động "nói chuyện" CIP-30 với dApp:

| Cách | Ý tưởng | Thực tế |
|---|---|---|
| **CIP-45** (WalletConnect-kiểu, qua relay) | dApp chạy trên trình duyệt máy tính bình thường, ví di động "ghép cặp" từ xa qua QR/relay | Theo CPS-0010 (đề xuất cải tiến CIP chính thức), mức dùng thực tế **gần như bằng không** — không ví di động lớn nào (Yoroi/Eternl/Typhon) coi đây là đường chính |
| **WebView riêng + tiêm shim** (đã chọn) | Mở dApp NGAY TRONG một WebView của chính app, tiêm `window.cardano.phoenixkey` vào trang đó | Đúng cách các ví di động lớn (Yoroi/Eternl/Typhon-kiểu) đang làm thật — dApp thấy y hệt một extension bình thường, chỉ khác là chạy trong WebView của ví thay vì Chrome/Safari |

**Chọn WebView-tiêm vì:** khớp thực tế thị trường (dApp không cần sửa gì để hỗ trợ PhoenixKey — chúng đã tự tìm `window.cardano.*`), không phụ thuộc hạ tầng relay bên thứ ba, và giữ toàn bộ luồng ký trong app (không cần app-switch qua ví ngoài).

**Đánh đổi phải chấp nhận:** người dùng phải duyệt dApp **bên trong** app PhoenixKey (không dùng được dApp qua Chrome/Safari ngoài app) — khác CIP-45 vốn cho ghép với trình duyệt máy tính từ xa. CIP-45 vẫn có thể bổ sung sau như một đường **khác**, không thay thế đường này.

---

## 4. Hành trình người dùng — từng bước

### Bước 1 — Mở "Trình duyệt dApp"
Chọn một gợi ý sẵn (Minswap/SundaeSwap/WingRiders/JPG Store) hoặc tự gõ địa chỉ. App **không kiểm duyệt** nội dung trang tự nhập — cảnh báo rõ ngay màn nhập URL: chỉ bấm "Cho phép" với trang bạn thực sự tin.

### Bước 2 — dApp xin kết nối (enable)
Trang gọi `window.cardano.phoenixkey.enable()`. App hiện hộp thoại: **domain thật** (đọc từ chính WebView, không phải trang tự khai) + cảnh báo "trang này sẽ xem địa chỉ, số dư, xin ký giao dịch". Bấm "Cho phép" → nhập PIN/vân tay → phiên kết nối mở CHO ĐÚNG domain đó, một lần mỗi phiên (điều hướng sang domain khác là phải xin lại).

### Bước 3 — dApp đọc ví (watch-only)
Sau enable, dApp gọi được `getUtxos`/`getBalance`/`getUsedAddresses`/`getRewardAddresses`… — toàn bộ nhóm này **không hiện popup**, chỉ đọc dữ liệu công khai của ví (giống xem số dư), không cần thêm xác nhận mỗi lần gọi.

### Bước 4 — dApp xin ký giao dịch (signTx)
Trang gọi `signTx(cbor)` — ví dụ bấm "Swap" trên Minswap. App **không ký ngay**: mở màn xác nhận riêng, hiện phí mạng, từng người nhận (địa chỉ + số ADA/token), có mint/burn hay không, có chữ ký khác đã gắn sẵn hay không. Ký cần vân tay/FaceID + PIN riêng cho LẦN NÀY — không dùng lại lượt xác thực lúc enable().

### Bước 5 — dApp xin ký dữ liệu (signData) — vd đăng nhập bằng ví
Trang gọi `signData(addr, payload)`. Màn xác nhận hiện domain + nội dung cần ký (chữ nếu đọc được, hex thô nếu không) + địa chỉ dùng để ký. Nếu địa chỉ dApp yêu cầu **không phải địa chỉ của ví này**, app từ chối ký — không đoán, không ký thay.

### Bước 6 — dApp gửi giao dịch lên mạng (submitTx)
Sau khi có đủ chữ ký, dApp gọi `submitTx`. App đẩy lên mạng qua hạ tầng chain-data sẵn có, trả `tx_hash` cho dApp hiển thị.

---

## 5. API surface CIP-30 đã cài

Khớp đúng bề mặt chuẩn CIP-30 (CIP-0030, github.com/cardano-foundation/CIPs) — đối chiếu `cip30_injection.dart:12-25`, dispatch ở `dapp_connector_service.dart:185-227`:

| Method | Loại | Cần `enable()` trước? | Hiện popup riêng? |
|---|---|---|---|
| `enable()` / `isEnabled()` | cấp quyền | — | `enable()` CÓ (mỗi phiên/domain) |
| `getExtensions` | watch-only | có | không (luôn trả `[]` — không hỗ trợ extension CIP-30 nào) |
| `getNetworkId` | watch-only | có | không |
| `getUtxos` / `getBalance` | watch-only | có | không |
| `getUsedAddresses` / `getUnusedAddresses` / `getChangeAddress` / `getRewardAddresses` | watch-only | có | không |
| `signTx(tx, partialSign?)` | request-sign | có | CÓ — màn riêng, chống ký mù |
| `signData(addr, payload)` | request-sign | có | CÓ — màn riêng, chống ký mù |
| `submitTx(tx)` | broadcast | có | không (đã ký rồi mới gọi) |

**Cố ý KHÔNG cài `getCollateral`** — CIP-30 đã đánh dấu deprecated (chọn collateral nay nằm phía tx-builder của dApp), không dApp swap/mint hiện đại nào cần ví trả API này để hoàn tất luồng chuẩn.

---

## 6. Các lớp bảo mật

### 6.1 Domain thật, không spoof được
`_currentOrigin` chỉ đọc từ chính `WebViewController` (URL đang thật sự tải), KHÔNG bao giờ lấy từ bất cứ thứ gì trang JS tự khai báo (`dapp_browser_screen.dart:154-164`). Cùng nguyên tắc đã sửa một lần cho `web_login_scan_screen.dart` (ký `challenge:domain:timestamp` trên domain app tự quan sát, không tin domain đối phương khai).

### 6.2 enable() luôn cần duyệt UI + PIN/vân tay
Không có "tự động kết nối". Mỗi domain phải qua hộp thoại + xác thực riêng; điều hướng sang domain khác reset trạng thái kết nối (`dapp_connector_service.dart:170-174` `resetForNavigation`) — không có danh sách "đã cho phép vĩnh viễn" tồn qua phiên khác hay app restart.

### 6.3 Chống ký mù (signTx)
`decode_tx_summary` (Rust, `dapp_connector.rs:114-226`) giải mã CBOR giao dịch **cục bộ trên máy**, không qua backend, hiện: phí, danh sách outputs (địa chỉ + lovelace + số native asset), mint/burn, số chữ ký đã có sẵn. Nếu KHÔNG giải mã được → trả rỗng, màn ký khoá nút "Ký" mặc định, chỉ mở khi người dùng tự tích "Tôi hiểu rủi ro và vẫn muốn tiếp tục" (`dapp_sign_tx_screen.dart:56-61` `_canSign`).

### 6.4 Chống ký mù (signData)
Hiện domain thật + nội dung cần ký (giải mã UTF-8 nếu đọc được, hex thô nếu không — không suy diễn/nắn nội dung) + địa chỉ dùng ký, trước khi xin PIN/vân tay (`dapp_sign_data_screen.dart:41-55`).

### 6.5 Từ chối ký cho địa chỉ lạ
`sign_data_cip30` (Rust) chỉ ký khi `address_bech32` khớp ĐÚNG địa chỉ payment hoặc stake tự derive được từ seed ví của account đó — không khớp thì trả rỗng, không đoán/không ký thay cho địa chỉ khác (`dapp_connector.rs:283-322`, test `sign_data_refuses_unrelated_address`).

### 6.6 Ký lại mỗi lần, không tái dùng phiên
Khoá phiên (`_sessionWalletSeed`) mở ra lúc enable() chỉ dùng để **đọc** ví (getUtxos/getBalance…) — signTx và signData KHÔNG dùng lại khoá phiên này, mỗi lệnh ký tự mở khoá riêng bằng một lượt PIN/vân tay mới (`dapp_browser_screen.dart:85-88`, `dapp_sign_tx_screen.dart:63-123`).

---

## 7. Bảng cải tiến (không có Connector vs có dApp Connector)

| Điểm | Không có Connector | Có dApp Connector |
|---|---|---|
| Dùng dApp Cardano | Phải rút ra ví khác trước | Dùng trực tiếp từ PhoenixKey |
| Kết nối | Không chuẩn nào cho mobile | CIP-30 tiêu chuẩn, dApp không cần sửa gì |
| Domain hiển thị | — | Luôn đọc từ WebView thật, không spoof được |
| Ký giao dịch | — | Màn riêng, hiện phí/outputs/mint, PIN/vân tay riêng mỗi lần |
| Ký cho địa chỉ không phải của mình | — | Từ chối ở tầng Rust, không đoán |

---

## 8. Quyền lợi và nghĩa vụ của bạn

**Bạn được:**
- Dùng ví PhoenixKey trực tiếp với dApp/sàn Cardano, không phải rút tiền ra ngoài trước.
- Domain kết nối luôn là domain thật đang mở, không có cách nào trang giả mạo tên domain khác.
- Mọi lệnh ký đều dừng lại chờ bạn xem nội dung + xác thực riêng — không có ký ngầm.

**Bạn cần:**
- Tự chịu trách nhiệm chọn trang mình tin — app không kiểm duyệt nội dung URL bạn tự nhập, chỉ bảo vệ **cách ví phản ứng** với trang đó.
- Đọc kỹ màn xác nhận trước khi ký, đặc biệt khi thấy cảnh báo "không đọc được nội dung" — đây là tín hiệu nguy hiểm thật, không phải lỗi vặt.
- Hiểu rằng dùng dApp qua trình duyệt PhoenixKey KHÔNG dùng được cho dApp mở ở Chrome/Safari ngoài app (đó là giới hạn cách chọn kiến trúc, xem mục 3).

---

## 9. Câu hỏi thường gặp

**Hỏi: dApp có thấy được PIN/vân tay của tôi không?**
Đáp: Không. PIN/vân tay chỉ mở khoá Master_KEK trong Secure Enclave của máy, không rời khỏi lớp native Flutter — trang web trong WebView không có đường nào chạm tới bước này.

**Hỏi: Tôi bấm "Cho phép" nhầm ở một trang lạ, có nguy hiểm không?**
Đáp: enable() chỉ cho phép trang đó ĐỌC địa chỉ/số dư (watch-only). Muốn CHUYỂN tiền, trang phải xin `signTx` riêng — bạn còn một lớp xác nhận + PIN/vân tay nữa mới ký thật.

**Hỏi: Vì sao không mở dApp ở Chrome/Safari luôn cho tiện?**
Đáp: CIP-30 chỉ hoạt động khi có ai đó tiêm `window.cardano.*` vào trang — trình duyệt hệ thống (Chrome/Safari) không cho app ngoài làm việc đó. Phải mở trong WebView của chính app mới tiêm được (mục 3).

**Hỏi: Có dùng được với ví Lace/Eternl khác song song không?**
Đáp: Không xung đột — Connector chỉ thêm `window.cardano.phoenixkey`, dApp nào cho chọn nhiều ví thì người dùng tự chọn ví muốn dùng ở mỗi lượt.

---

## 10. Rủi ro còn lại (đối kháng — chưa vá, cần theo dõi)

Đây là các đường tấn công còn mở SAU khi đã có đủ 6 lớp bảo mật ở mục 6 — không phải lỗi thiết kế cơ bản, mà là giới hạn thật của phiên bản hiện tại, ghi rõ để không hứa hão:

### 10.1 [CAO] `decode_tx_summary` không hiện certificate/withdrawal/collateral — ký mù "hợp cú pháp" vẫn có thể hại
`decode_tx_summary_inner` (`dapp_connector.rs:118-226`) chỉ đọc `fee/ttl/inputs/outputs/mint/existing_vkey_witnesses/auxiliary_data_present` từ `body()`. Nó **không đọc và không hiện**: `certificates()` (StakeDelegation, VoteDelegation/DRep, StakeDeregistration, PoolRegistration/Retirement), `withdrawals()`, `collateral()`, `reference_inputs()`, hay dữ liệu script/redeemer.

Hệ quả cụ thể: một dApp độc có thể mời ký một giao dịch **trông vô hại** — ví dụ "claim 2 ADA reward", phí thấp, một output nhỏ về đúng ví bạn — nhưng gộp kèm một `VoteDelegation` chuyển phiếu governance sang DRep của kẻ tấn công, hoặc một `StakeDelegation` sang pool lạ, hoặc một `StakeDeregistration` rút cọc về nơi khác. Màn xác nhận hiện tại (`dapp_sign_tx_screen.dart:283-332` `_summaryWidgets`) sẽ cho thấy y hệt một giao dịch bình thường — người dùng KHÔNG có cách nào nhìn ra cert/withdrawal ẩn bên trong chỉ từ UI này. Đây đúng kịch bản "hợp cú pháp nhưng rút sạch quyền kiểm soát" mà lớp chống-ký-mù hiện tại chưa phủ tới.

**Gợi ý vá (không tự sửa):** mở rộng `decode_tx_summary` để liệt kê certs (loại + tham số chính) và withdrawals (địa chỉ + số tiền), đồng thời cảnh báo riêng khi tx có collateral hoặc script witness (rủi ro mất collateral nếu script fail on-chain).

### 10.2 [TRUNG BÌNH] JavaScriptChannel không phân biệt frame — iframe bên thứ ba trong trang có thể tự gọi thẳng kênh, gắn mác domain top-level
`addJavaScriptChannel(_kChannelName, ...)` (`dapp_browser_screen.dart:119`) đăng ký kênh cho TOÀN BỘ JS context của trang đang tải trong WebView — theo hành vi mặc định của `webview_flutter`, kênh này **không cô lập theo frame/origin**: một `<iframe>` quảng cáo/widget bên thứ ba nhúng trong một trang dApp hợp pháp (vd một banner quảng cáo độc hại trên `minswap.org`) có thể tự gọi thẳng `window.PhoenixKeyCIP30.postMessage(...)` — kể cả không cần shim của chính dApp — mà không có cách nào ở tầng Dart phân biệt "lệnh này tới từ script của chính minswap.org hay từ iframe lạ nhúng trong đó". Yêu cầu vẫn bị gán `origin = "https://minswap.org"` (đúng domain top-level thật, không bị giả mạo tên) — nên hộp thoại vẫn hiện tên domain đúng, nhưng **domain đúng không đồng nghĩa mã đang gọi là mã của chính trang đó**. Kết quả: một widget/quảng cáo bị compromise trong một trang vốn đáng tin có thể tự ý bắn `enable()`/`signTx()` mà chủ trang dApp không hề chủ động yêu cầu.

Mức hại bị chặn một phần vì mọi bước ký vẫn qua PIN/vân tay + màn xác nhận riêng — nhưng nó phá vỡ giả định ngầm "domain hiện đúng nghĩa là first-party code của domain đó đang xin", mở đường cho tấn công kiểu MFA-fatigue (mục 10.3) từ một nguồn không phải chính dApp.

### 10.3 [TRUNG BÌNH] Không giới hạn tần suất `signTx` — spam yêu cầu ký gây mỏi mệt, dồn về nút "bỏ qua cảnh báo"
Không có cơ chế cooldown/rate-limit nào giữa các lệnh `enable()`/`signTx()` liên tiếp từ cùng một origin đã kết nối (`dapp_connector_service.dart:215-217`, không có bộ đếm hay giới hạn). Một dApp độc có thể bắn liên tục nhiều yêu cầu ký — đặc biệt các tx cố tình khiến `decode_tx_summary` thất bại (CBOR khác thường nhưng vẫn hợp lệ để ký) — khiến người dùng liên tục thấy cảnh báo "Không đọc được nội dung giao dịch" (`dapp_sign_tx_screen.dart:238-281`). Hiệu ứng mỏi-mệt-do-cảnh-báo (alert fatigue, cùng họ với tấn công MFA-fatigue) khiến người dùng có xu hướng tích ô "Tôi hiểu rủi ro và vẫn muốn tiếp tục" theo phản xạ ở một trong nhiều lần thay vì đọc kỹ — đúng lúc đó lại là giao dịch thật gây hại.

### 10.4 [THẤP] Rút gọn địa chỉ trong màn xác nhận có thể bị lợi dụng bằng địa chỉ "nhái"
`_truncateAddr` (`dapp_sign_tx_screen.dart:351-354`) chỉ hiện 14 ký tự đầu + 8 ký tự cuối của mỗi địa chỉ output. Về lý thuyết, kẻ tấn công có thể tốn công sinh (grind) một địa chỉ Cardano có đúng phần đầu+cuối trùng với địa chỉ quen thuộc của người dùng (địa chỉ ví chính họ hoặc một sàn quen) nhưng phần giữa khác hẳn — bản rút gọn hiển thị y hệt bản thật, người dùng lướt qua dễ tưởng nhầm là gửi về đúng ví mình. Chi phí grind một khớp 14+8 ký tự bech32 là đáng kể nhưng không phải bất khả thi với tài nguyên đủ lớn — xếp mức THẤP vì cần đầu tư tính toán trước và nhắm đúng nạn nhân cụ thể, không phải tấn công hàng loạt rẻ tiền.

**Không phải rủi ro** (đã kiểm, loại trừ rõ): clickjacking đè lên màn xác nhận enable()/signTx()/signData() — cả ba đều là dialog/màn hình **native Flutter** (`AlertDialog`, `Navigator.push`), nằm ngoài DOM của WebView, trang web bên trong WebView không có cách nào vẽ đè lên native UI của chính app.

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai) + code tham chiếu trực tiếp (nhánh local `claude/dapp-connector-webview`, chưa merge/push):
`Enclave/lib/screens/dapp_browser_screen.dart`, `Enclave/lib/services/dapp_connector_service.dart`, `Enclave/lib/services/cip30_injection.dart`, `Enclave/lib/screens/dapp_sign_tx_screen.dart`, `Enclave/lib/screens/dapp_sign_data_screen.dart`, `Enclave/rust_core/src/dapp_connector.rs`, `Enclave/test/dapp_connector_service_test.dart`.
Chuẩn tham chiếu: CIP-0030 (`github.com/cardano-foundation/CIPs/blob/master/CIP-0030/README.md`), CPS-0010 (mức áp dụng CIP-45 thực tế).
Đối chiếu bất biến/định hướng trước đó (cần soát lại cho khớp — xem hộp đính chính đầu file): PhoenixKey-Rebirthme-Math.md §4.L (repo module nội bộ — private, chưa public) (I-CONN-1..7, Hướng-1/Hướng-2), PhoenixKey-Rebirthme-Tech.md §5.7 (repo module nội bộ — private, chưa public) (kiến trúc CIP-30 Lace connector — Hướng-1, khác chiều với tài liệu này).
Tài liệu liên quan: [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md) (ví đa địa chỉ theo DID, hạ tầng dùng chung).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
