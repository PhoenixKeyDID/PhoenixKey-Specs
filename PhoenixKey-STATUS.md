# PhoenixKey — STATUS (hiện trạng & tiến độ)

> **File này là báo-cáo hiện-trạng, KHÔNG phải đặc-tả.** Bộ spec (`*-Vi-Feat/-Math/-Tech/-Exec`) là **kim-chỉ-nam thiết-kế** — mô tả hệ thống ĐÍCH mà các đội dev xây tới. File này ghi *đang ở đâu trên đường tới đó*: cái gì đã chạy, chặn bởi ai, bằng chứng test. Khi hai bên lệch → spec là mục-tiêu, STATUS là thực-tại.
>
> **Cập nhật: 2026-08-28.** Mọi số dưới đây đo lại từ đầu trên `main` của từng kho, không kế thừa bản trước. Bản 2026-08-12 có **8/9 hash validator đã lỗi thời** và bốn chỗ mô tả sai chiều — xem §7.

---

## 0. Mốc đo — mọi con số trong file này neo vào đây

Bảng này là **cổng**: một khẳng định trong file không neo được vào một dòng ở đây thì nó là suy đoán, không phải hiện trạng.

| Kho | `main` | Kiểm chạy được | Kết quả | CI |
|---|---|---|---|---|
| PhoenixKey-Validator | `cf5dceb` | `aiken check` / `aiken build` | **658/658 PASS**, build exit 0, 9 validator / 23 mục blueprint | 🔴 **0 workflow** |
| PhoenixKey-Database | `99b33e2` | `mvn test` | **668 test**, 0 fail | có |
| PhoenixKey-Core | `bd8ae3d` | `cargo test` / `flutter test` | **461 passed** (1 ignored) / **296 passed** | có |
| PhoenixKey-Frontend | `6be695a` | `vitest src/lib/sdvc/` | **425 test PASS** | có |
| PhoenixKey-SDK | `301fb78` | `jest` | **20/20 PASS** | 🔴 **0 workflow** |
| Wallet (kho `PhoenixKeyDID/Wallet`) | `e498b00` | test bộ ví | **178/178 PASS** | có |
| PhoenixKey-Specs | `45fafaf` | — (kho tài liệu) | — | — |

Hai kho **Validator** và **SDK** không có một workflow nào: mọi con số của chúng ở trên là **đo tay tại một thời điểm**, không có gì chặn một commit sau đó làm gãy mà không ai biết. Đây là việc hạ-tầng ưu tiên cao nhất ngoài mã.

### Hash 9 validator — đo lại trên `cf5dceb`, `aiken build` môi trường mặc định

| Validator | Hash (chưa apply-param) |
|---|---|
| `did_payment` | `0b3e301b447d1a45eddaa65b1e5c2abce1898859729144e494420f4a` |
| `did_pool` | `4b4030d7892c86328fd4de54e79f29443a2d965c26343ab6831fabbe` |
| `did_stake` | `17f80f889e36aa5fdd4969fbc2a7748d343528b74d986a51ed995b29` |
| `lamp_policy` | `ba0dd83a39a54022f8d07fbcf5079f7f120bf51b9d2016aad7d97e13` |
| `limit_meter_vault` | `45138d589f2fdb6646b1defb155db398436072d64466e28c4aaad0c0` |
| `protectme_payout` | `cd4f2c2415256692ba3a726cec062a4a8ab9171f1a7df6913a31c675` |
| `smartsend` | `5380d67aa35ed5378d64dad05b3c44c9d5902525d6c53cbbab104b91` |
| `taad` | `b7a582805e935584689268275262f2db269c91e89280b7ff0ec7ea33` |
| `wakeme_vault` | `84f97191c28df3ce46ad4ae1fdc2cde19ae15fcbb54f2ad402bc39a3` |

⚠ **`wakeme_vault` phụ thuộc môi trường build.** Cùng `cf5dceb`, `aiken build --env preview` cho `04fefb8624bcacdcc843acd97cb308e96288a6f31bc08017f829d9b9` — khác hoàn toàn, vì `[config.preview] ms_per_day = 60000` được mã đọc thật (`wakeme_logic.ak:87,95,100`). ⟹ **mọi phát biểu về hash `wakeme_vault` phải kèm tên môi trường**, không có "hash của wakeme_vault" nói trống.

`plutus.json` có **23 mục** nhưng chỉ **9 script-hash phân biệt** — mỗi validator xuất nhiều handler (mint/spend/withdraw…) dùng chung một hash. Đếm 23 là đếm handler, không phải đếm validator.

---

## 1. Sự thật cấu trúc lớn nhất — đường đúc DID đang sống KHÔNG đi qua validator

Đây là điều quan trọng nhất trong toàn bộ file, và nó không nằm ở module nào cả.

`PhoenixKey-Database/.../cardano/CardanoServiceImpl.java:70-111` — `publishDocument()` trả tiền cho chính ví phí backend, gắn metadata nhãn 6789, ký bằng ví phí rồi submit không chờ xác nhận. **Không mint state-NFT, không gọi `taad`.** Đường này dùng chung cho cả ba loại DID đang cấp được: PersonDID (`/identity/register`), OrgDID (`/org/create`), AssetDID (`/identity/asset/create`).

`PhoenixKey-Validator/deploy/README.md:23` — bản hash TAAD preprod duy nhất từng được chốt trong `deploy/` tự khai **"ĐÃ CHẾT — đừng deploy"** (lý do: `ValidatorParams` đổi 6→7 trường). Tức bản đó lỗi thời *trước khi* deploy, không phải deploy rồi hỏng.

Hệ quả phải nói thẳng:

- PoP-bind, CanOwn, 2-of-2, uniqueness anchor là **thiết kế có thật, đã build, test xanh — nhưng chưa được thi hành trên chuỗi**.
- Bất-biến danh tính hôm nay được giữ bởi **Postgres**, không phải sổ cái. Một bản ghi DB sai hoặc bị sửa thì không có lớp thứ hai bắt lại.
- Điều này đi ngược thứ tự ưu tiên đã chốt cho toàn hệ (an-ninh trước trải-nghiệm, quyền phải ở ledger).
- 🔴 **Và nó khoá vốn.** `did_payment.ak:55` gọi `auth_logic.anchor_controller_ok`, mà hàm đó mở bằng `expect Some(anchor) = find_anchor_datum(tx.reference_inputs, …)` (`auth_logic.ak:130-131`) rồi đòi `is_active(anchor.status)`. Không có anchor NFT trên chuỗi ⟹ **không ai chi được từ một địa chỉ ví Phoenix — kể cả người giữ đủ 24 từ.** Đây là hệ quả nặng nhất của §1 và nó đi ngược chiều với trực giác thông thường: chỗ này không phải "kém an toàn", mà là "tiền vào thì được, ra thì chưa chứng minh được đường nào".

**Điều kiện tiên quyết bị bỏ quên — khuôn `did` lệch.** Backend/app/frontend sinh did theo khuôn `base32(slot)+hex` (`DidPhoenixGenerator.java:43,102`, `taad_did.rs:129-167`). `pop_bind.ak:20-30` đòi khuôn tự-chứng (POSIX-ms thập phân + `controller_pkh` trong tiền-ảnh). Bản Java theo khuôn mới **có mã nhưng đang tắt**: `application.yml:171 pop-bind-encoder-enabled: false`. ⟹ **mọi DID cấp hôm nay không mint được anchor dưới validator hiện hành.** Và vì `N(did) = blake2b_256(did)` là apply-param của `did_payment`, đổi khuôn = đổi địa chỉ ví của mọi DID ⟹ việc này phải đi qua cổng THỜI-CHÍNH, không vá lẻ. Theo dõi ở `PhoenixKey-Validator` #78.

---

## 2. Bảng trạng-thái 8 module

| Module | Nền đã chạy được | Blocker chính | Production |
|---|---|---|---|
| **Anchorme** | `taad` Design-2 + PC (uniqueness anchor) + PoP-bind, đã nối; resolver W3C; register metadata-6789 | §1 (anchor chưa thi hành trên chuỗi); khuôn `did` lệch (#78); DeviceDID; resolve-by-hash | NO-GO — chặn ở §1, không ở CID-1 |
| **Rebirthme** | ví theo-DID `did_payment`, đóng-băng theo trạng-thái, guardian recovery ngưỡng+timelock, P-256 low-s, `limit_meter_vault` + `did_stake` build được | tầng **Dart** của khoá thiết bị chưa có (xem §3); `did_subaddr.ak` chưa tồn tại trên `main` | NO-GO ví-giá-trị-lớn |
| **Wakeme** | `wakeme_vault`/`wakeme_logic` merge, test xanh; backend có **hiện thực thật** (`ActivationVaultServiceImpl` `@Primary` + preflight + tx-builder + chọn pot UTxO) | 3 biến `activation-vault-cbor-hex`/`pot-address`/`keeper-pkh` còn rỗng (`application.yml:142-144`) ⟹ trả 501 — **chặn ở deploy, không ở mã**; app không có màn hình Wakeme; **không có mắt nối sang Gen MAGIC** | NO-GO |
| **Feecover** | ConsumeMAGIC lõi (kế thừa) | **0 validator Feecover tồn tại**; CARP chưa có policy-id thật; hai đầu tiêu thụ chưa cắm vào | NO-GO |
| **Protectme** | `protectme_logic` + `protectme_payout` + **`protectme_beacon_logic` (889 dòng, 26 test, ĐÃ NỐI)** — 75 test trên `main` | 2-bucket Treasury + resolver claim + UI chưa code; 11 quyết-định PROT-1..11 | NO-GO — nhưng blocker beacon **đã gỡ** |
| **Knowme** | Mức 1+2 SD-VC có code+test, demo `/vc` — 425 test PASS | lib BBS (Mức 3); LampNet gateway; StampRecord | M1 chạy |
| **Easteregg** | `did_pool` build được | `did_subaddr.ak` chưa tồn tại; ZK Tầng 2 verifier chưa viết; 0 test Easteregg | NO-GO |
| **Smartsend** | validator `smartsend` build được | verifier Glint/Spectra (Phase 2); phụ thuộc nền ví + guardian | NO-GO; money-critical |

---

## 3. Ma-trận blocker xuyên-module

| Blocker | Chặn | Đội gỡ | Trạng thái đo được |
|---|---|---|---|
| 🔴 **Đường đúc không qua validator** (§1) | TẤT CẢ | on-chain + backend | `CardanoServiceImpl.java:70-111`. Chưa có phương án chốt (xem §6) |
| 🔴 **Khuôn `did` lệch** | Anchorme, Rebirthme | on-chain + backend + app | `application.yml:171` đang `false`; Validator #78 |
| 🔴 **Khoá thiết bị — tầng Dart** | Rebirthme (ví Phoenix) | app | Rust **đã xong**: `device_key.rs` 893 dòng, `device_pkh` 67 lần / 4 tệp, và **đã xuất FFI C** 4 hàm (`lib.rs:955,996,1039,1060`). Tầng Dart: `git grep taad_device_key -- lib` = **0**. Chỗ đứt là *binding Dart*, không phải mã lõi |
| 🔴 **CBOR `did_payment` đang dùng là bản cũ 1-chữ-ký** | Rebirthme, Anchorme | on-chain + app + backend | Địa chỉ Phoenix đã cấp dẫn ứng với bản validator chỉ đòi 1 chữ ký (thiếu vế `device_pkh`, `auth_logic.ak:143`). Nhưng vế **anchor Active vẫn còn trong cả hai bản** ⟹ kết hợp với §1 thì hôm nay không đường nào chi được, không phải "seed-alone chi được". Re-pin cùng lượt: asset CBOR + vector test backend + env `DID_PAYMENT_CBOR_HEX` |
| 🔴 **ServiceDID / AgentDID / DeviceDID = 0 dòng backend** | Anchorme, LampNet, Knowme | backend | `types.ak` đã đủ 5 loại + ma trận `can_own` (validator sẵn 100%); `grep DidType.SERVICE\|AGENT\|DEVICE` trong `src/main/java` = **0**. `DeviceController.java` là push-token FCM, **không phải** DeviceDID |
| 🔴 **Khoá phiên đứt ở HAI chỗ độc lập** | khoá phiên (§5) | backend | (a) `/auth/token/exchange` nhận `sessionToken` trong BODY nhưng không có trong `PUBLIC_POST` (`AuthRequiredInterceptor.java:108-118`) ⟹ đo thực địa trả `401 {"code":1304}`, không bao giờ chạm `TokenExchangeService`. (b) Không API nào ghi được `service[] type=SsoRedirect` lên DID Document ⟹ dù mở (a) thì vẫn chưa đăng ký được website nào. Gỡ một chỗ không đủ |
| 🔴 **B1 — MAGIC engine đọc-số-dư** | Wakeme, Feecover, Protectme | MAGIC team | Model = account-in-vault (không native) |
| 🔴 **B2 — CARP policy-id thật** | Feecover, Protectme, Rebirthme, Wakeme | CARP team | Đang để all-zero fail-closed; test dùng hằng giả |
| 🔴 **FROST vs VSS — hai bên xây hai nguyên thuỷ khác nhau** | phân tán seed (§5) | PhoenixKey + LampNet | Chốt là FROST-sign; bản đã build là VSS/Shamir. **FROST không có hiện thực ở nhánh nào**: quét toàn bộ lịch sử Core/Validator/Database, chuỗi `FROST` chỉ xuất hiện trong tên một tài liệu thiết kế được trích ở chú thích (`commitment_tree.ak:5`, `pa2_smt.ak:5`); mọi kết quả còn lại là `Blockfrost`. `seed_envelope.rs` 1284 dòng đã merge nhưng **0 hàm xuất FFI** ⟹ Dart không gọi tới được |
| 🟡 `GenesisChild` 1 chữ ký | Anchorme | on-chain | `state_nft_logic.ak:198` đòi 1 chữ ký, trong khi mọi đường CHI là 2-of-2 (`auth_logic.ak:137,143`) |
| 🟡 `app_token` là bearer thuần | khoá phiên | backend + SDK | `PhoenixKey-SDK/INTEGRATION.md:113` tuyên bố đã có DPoP — **tài liệu nói quá mã** |
| ~~PA2 UniquenessThread~~ | — | — | ✅ loại vĩnh viễn, thay bằng PC (đã land + nối) |
| ~~PA5-a entity-gate~~ | — | — | ✅ chốt không nối — `entity_type` đã cam-kết ở tầng mint (`pop_bind.ak:109`) |
| ~~`limit_meter.ak` anti-drain~~ | — | — | ✅ build được trên `main`, nằm trong 658/658 |
| ~~`protectme_beacon.ak` 0 dòng~~ | — | — | ✅ **gỡ 2026-08-28** — `protectme_beacon_logic.ak` 889 dòng / 26 test, đã nối tại `protectme_payout.ak:54` |

---

## 4. Hai lỗ hổng ở tầng nghiệp vụ cần chủ kho vá

Hai lỗ dưới đây **không** phải lỗ mật mã và **không** chặn build — chúng nằm ở luật nghiệp vụ, và đều có chung một hình dạng: *ai đến trước được, không ai kiểm quyền, không có đường đòi lại*.

**(a) Đăng ký AssetDID theo `physicalIdHash` — thiếu bước chứng minh quyền trên vật.** Endpoint **có** xác thực: `/identity/asset/create` không nằm trong `PUBLIC_POST` (`AuthRequiredInterceptor.java:108-118`) nên đòi phiên hợp lệ, và `IdentityController.java:265-271` còn đòi chữ ký owner trên challenge chứa `physicalIdHash`. Cái thiếu hẹp hơn nhưng vẫn nghiêm trọng: **không có bước nào chứng minh người ký thật sự sở hữu vật đó.** `AssetServiceImpl.java:98` nhận first-claim-wins trên `physicalIdHash` với UNIQUE ở `V14__add_asset_dids.sql:20` ⟹ một DID đã đăng ký bất kỳ vẫn gắn được định danh cho một vật của người khác, và chủ thật chỉ nhận 409, không có đường đòi lại. **Cần:** một cách chứng minh quyền trước khi ghi + một luật chuyển giao khi có tranh chấp. Đây là quyết-định nghiệp-vụ, không phải vá kỹ-thuật.

**(b) Registry dấu giấy-tờ — client tự khai, và đường ghi ĐANG MỞ.** `FingerprintRegistryService.java` (344 dòng, migration V35/V37/V40/V41) chưa được nối vào `/identity/register`, và app Flutter chưa gọi. Nhưng nó **không hề đóng**: `KnowmeFingerprintController.java:35,57` đã wire `POST /identity/fingerprint/register`, và `GET /identity/fingerprint/status` nằm trong `PUBLIC_GET` (`AuthRequiredInterceptor.java:94`) nên tra được không cần xác thực (đo thực địa: trả 400 vì tham số, không phải 401).

⟹ Lỗ "client tự khai số giấy tờ, ghi rồi không bao giờ xoá" **đang mở ở dạng gọi thẳng**, không phải đang chờ được nối. `V37__..._intent.sql:11` ghi *"Đừng nối theo cách nào trước khi chốt"* — lời cảnh báo đúng, nhưng nó chỉ chặn đường nối tự động, không chặn đường gọi trực tiếp. **Cần:** một cơ chế chứng minh quyền trên giấy tờ trước khi ghi, và trong lúc chờ thì đóng hoặc hạn chế đường ghi trực tiếp.

Ngoài ra `person_did` cố ý **không** UNIQUE ⟹ bất-biến thực tế hôm nay là **"1 giấy-tờ → 1 DID"**, không phải "1 người → 1 DID". Một người khai cả CCCD lẫn hộ chiếu vẫn ra 2 DID.

---

## 5. Đo hiện trạng năng lực đầu-cuối (2026-08-28)

Mục 2–4 tổ chức theo module. Mục này tổ chức theo **việc người dùng làm được**, vì một module xanh không có nghĩa người dùng bấm được.

| Người dùng làm được gì | Hiện trạng | Chỗ đứt |
|---|---|---|
| Tạo ví **Standard** và chi tiền | **ĐƯỢC** | đường chi duy nhất đang chạy — không phụ thuộc anchor |
| Tạo ví **Phoenix**, nhận tiền | ⚠ **NHẬN ĐƯỢC, CHƯA CHỨNG MINH CHI RA ĐƯỢC** | `did_payment.ak:55` đòi anchor Active qua reference input; §1 cho thấy chưa có đường đúc anchor nào trên chuỗi. Cho tới khi đo được một UTxO anchor thật, phải coi đây là **rủi ro khoá vốn**, không phải ô xanh |
| **Chi tiền từ ví Phoenix** (2 yếu tố) | **KHÔNG** | Rust + FFI đã xong; **binding Dart chưa viết** ⟹ app không gọi được |
| **Một người một DID** | **KHÔNG** | Cổng duy nhất là `existsByPublicKeyHex` (`IdentityServiceImpl.java:107-113`) ⟹ thực chất **một MÁY một DID**. Registry giấy-tờ chưa nối vào `/identity/register` nhưng đường ghi trực tiếp đã mở và chưa an toàn (§4b). Mất máy = tạo DID mới = chính nguồn sinh trùng |
| Tạo **OrgDID** | **ĐƯỢC** | nhưng chỉ là giao dịch metadata (§1) |
| Tạo **AssetDID** | **ĐƯỢC** | ⚠ không kiểm quyền sở hữu (§4a) |
| Tạo **ServiceDID / AgentDID / DeviceDID** | **KHÔNG** | 0 dòng backend; validator đã sẵn sàng |
| **Phân tán bộ seed trên LampNet** | **KHÔNG** | Backup đang chạy là **một blob tới một CID**, khoá dẫn từ Master_KEK (`derive_x25519_static_from_kek`, `lampnet.rs:149`) ⟹ ai có 24 từ là mở được. Không k-of-n. Nguyên liệu để sửa **đã có sẵn cùng crate** — `seed_envelope::generate_k_env()` (`lampnet.rs:141-143` xác nhận), còn thiếu là nối + di trú blob v1. Và hai bên đang xây hai nguyên thuỷ khác nhau (§3) |
| **Cấp khoá phiên** cho app/website | **KHÔNG** | `/auth/token/exchange` trả 401 vì thiếu 1 dòng whitelist (§3); và không API nào ghi được `service[] type=SsoRedirect` lên DID Document ⟹ chưa đăng ký được website nào |
| Ứng dụng **bên thứ ba** đăng nhập bằng PhoenixKey | **KHÔNG** | cả hai chỗ đứt ở §3 đều phải gỡ; thêm nữa chưa có màn hình đồng ý cấp quyền |
| **Nhận LAMP** từ ETD / Airdrop / SRCL | **KHÔNG**, cả ba | không có đường `POST claim` nào ở phía PhoenixKey |
| Đăng nhập web PhoenixKey bằng QR | **ĐƯỢC** | đo thực địa 2026-08-28: `POST /auth/session/init` → 200; `GET /api/v1/.well-known/jwks.json` → 200, `kid=phoenixkey-ed25519-1` |
| **Wakeme / mượn 1001 LAMP** | **KHÔNG** | mã backend có thật và là bean được chọn (`ActivationVaultServiceImpl.java:40-99`), nhưng 3 biến môi trường vault còn rỗng ⟹ `NOT_YET_IMPLEMENTED`; app không có màn hình Wakeme; pot 1.001 tỷ LAMP chỉ có trong tài liệu, chưa có bằng chứng deploy |
| **Gen MAGIC / Consume MAGIC** qua Wakeme | **KHÔNG** | `grep wakeme` trong toàn bộ validator của kho MAGIC = **0**. MAGIC dùng vault riêng ⟹ **không có mắt nối** giữa hai bên |
| **Feecover trả hộ phí** cho AladinWork / OriLife | **KHÔNG** | 0 validator Feecover; AladinWork chạy `phoenixkey.mock:true`; OriLife có đường phí RIÊNG bằng LAMP, `grep feecover` trong OriLifeTrace = **0**. Thêm nữa, phần "một ví đứng ra trả hộ phí mạng" **chưa được thiết kế ở tài liệu nào** — Feecover Tech §10 chỉ định tuyến CARP |
| Xem **danh sách người bảo trợ** | **KHÔNG** | không có `GET /guardians`; `/add`+`/remove` chỉ trả về SỐ LƯỢNG — mà đây là màn hình bắt buộc trong luồng khôi phục |
| **Từ chối** một yêu cầu ký | **KHÔNG** | không có `POST /sign/{id}/reject`; chỉ `approve`/`cancel`, dù enum trạng thái đã có sẵn `"rejected"` |

### Lệch spec ↔ mã đang sống

| Chỗ | Spec ghi | Mã chạy | Bên nào đúng |
|---|---|---|---|
| Công thức D của Wakeme | `min(1001, ⌊pot×1001/10⁹⌋)` (`Wakeme-Exec.md:17`) | **validator KHÔNG ép công thức này.** `wakeme_logic.ak:811-818` chỉ ép `1 ≤ d_unit ≤ oil_per_lamp` và `conditional_lamp == 1001·d_unit`; `d_unit` là trường datum do builder khai (`:52-53`). Tỷ lệ theo số dư pot nằm hoàn toàn off-chain, tin builder | spec đúng nhưng **chưa được thi hành** — `Wakeme-Exec.md:19` tự ghi việc chốt-cứng với validator để ở PA-1. Ngoài ra bản CŨ `/10⁶` còn sống trong tài liệu và mã Database (vd `ActivationVaultController.java:40`, `ActivationVaultDtos.java:44`) |
| Đơn vị thời gian Wakeme | `vest_start_slot` | `vest_start_ms` (`wakeme_logic.ak:43,131,192-212`) | **mã đúng** — Plutus `validity_range` trả POSIXTime (ms), không lộ slot ⟹ sửa spec |
| DPoP trong SDK | "đã có" (`INTEGRATION.md:91`) | bearer thuần | **mã đúng**, tài liệu nói quá |

---

## 6. Quyết-định đang chặn tiến độ (không phải việc dev)

Những việc dưới đây **không** thiếu người làm — chúng thiếu một lựa chọn được chốt. Liệt kê ở đây để không ai chờ nhầm.

1. **Đường đúc DID** — giữ metadata-6789, chuyển sang TAAD UTxO thật, hay phân tầng theo loại DID. Quyết định này định đoạt §1 và kéo theo cổng THỜI-CHÍNH.
2. **Khuôn `did`** — bật `pop-bind-encoder`, và di trú các DID đã cấp thế nào (Validator #78).
3. **Luật đòi lại AssetDID** khi bị đăng ký nhầm/chiếm chỗ (§4a).
4. **Điều kiện được ghi vào registry giấy-tờ** (§4b).
5. **FROST hay VSS** cho phân tán seed — hiện có hai nguyên thuỷ song song, phải chọn một trước khi xây tiếp.
6. **`GenesisChild` có nâng lên 2-of-2 không** — nếu có thì đổi hash, đi cổng THỜI-CHÍNH.
7. **Paymaster giữ hay bỏ** — treo từ 2026-08-12; phần "trả hộ phí mạng" của Feecover phụ thuộc câu trả lời.

---

## 7. Những khẳng định của bản trước đã bị bác

Ghi lại để không ai dựng lập luận trên nền cũ:

- **"9 hash validator"** của bản 2026-08-12 — **8 trong 9 đã đổi**, không phải cả 9. `taad` từ `5ac17898…` sang `b7a58280…`, `did_payment` từ `bac16cec…` sang `0b3e301b…`. Ngoại lệ đáng giữ: **`lamp_policy` vẫn là `ba0dd83a…`, không đổi qua hai lần đo.** Riêng `wakeme_vault` thì **không so được** — giá trị cũ `8655974a…` không ghi kèm môi trường build, mà hash này phụ thuộc môi trường (§0).
- **"577/577 test"** → nay **658/658**.
- **"`grep device_pkh` trong `rust_core/src` = 0"** — SAI kể từ 2026-08-13. `device_key.rs` 893 dòng, đã xuất FFI. Phần còn đúng: tầng **Dart** vẫn = 0.
- **"`protectme_beacon.ak` 0 dòng, chặn-merge"** — SAI. 889 dòng, 26 test, đã nối.
- **"CID-1 đã đóng ⟹ một người một DID"** — SAI. `pop_bind.ak:73-77` tự ghi phạm vi: *không* đóng 1-người-1-DID ở mức người.
- **"duy-nhất-người không phụ thuộc bí mật nào"** — đúng cho trường hợp **lộ**, sai cho trường hợp **mất**: `FingerprintRegistryService.java:74-101` ghi rõ mất pepper cũ là mất khả năng nhận ra người cũ.
- **"`activation_logic.ak` 69 test"** — tệp đó không còn tồn tại; Wakeme nay là `wakeme_logic.ak`.
- **"Nút Claim MAGIC trong app gọi endpoint luôn trả 410"** — phía app **đã sạch**: `git grep 'magic/claim\|claimMagic' -- lib` trên `bd8ae3d` = **0**. Endpoint `/wallet/magic/claim` vẫn còn ở backend nhưng là bia mộ có chủ đích: `@Deprecated(forRemoval=true)`, thông báo chỉ thẳng sang `/wallet/{did}/all`. Không còn là lỗi người dùng gặp.
- **"đường đúc không qua validator" chỉ là chuyện kiến trúc** — không phải. Nó kéo theo hệ quả khoá vốn ở §1, và hệ quả đó chưa từng được ghi ở bản nào trước.

---

## 8. Nhật-ký bằng chứng

| Ngày | Kho | Bằng chứng |
|---|---|---|
| 2026-08-28 | Validator `cf5dceb` | `aiken check` **658/658 PASS**; `aiken build` exit 0; 9 hash ở §0; `wakeme_vault` khác nhau giữa env mặc định và `--env preview` |
| 2026-08-28 | Validator `cf5dceb` | `protectme_beacon_logic.ak` 889 dòng / 26 test, nối tại `protectme_payout.ak:54`; Protectme tổng **75 test** |
| 2026-08-28 | Database `99b33e2` | **668 test**, 0 fail; 41 tệp migration, cao nhất V42; `DidType.SERVICE\|AGENT\|DEVICE` = 0 |
| 2026-08-28 | Database (chạy thật) | `POST /api/v1/auth/token/exchange` → **`401 {"code":1304}`** — chặn bởi whitelist, không phải bởi logic |
| 2026-08-28 | Core `bd8ae3d` | `cargo test` **461 passed / 1 ignored**; `flutter test` **296 passed**; `device_key.rs` 893 dòng; 4 hàm FFI ở `lib.rs`; `git grep taad_device_key -- lib` = **0** |
| 2026-08-28 | Core `bd8ae3d` | `seed_envelope.rs` 1284 dòng, `grep 'seed_envelope::' lib.rs` = **0** ⟹ chưa xuất FFI |
| 2026-08-28 | Frontend `6be695a` | SD-VC **425 test PASS** |
| 2026-08-28 | SDK `301fb78` / Validator | `ls .github/workflows` = **0** ở cả hai kho |
| 2026-08-28 | Wallet `e498b00` | **178/178 PASS** |
| 2026-08-28 | Validator `cf5dceb` | `did_payment.ak:55` → `auth_logic.anchor_controller_ok`; `auth_logic.ak:130-131` mở bằng `expect Some(anchor) = find_anchor_datum(tx.reference_inputs,…)`, `:135` `is_active(anchor.status)` ⟹ không anchor thì không chi được |
| 2026-08-28 | Validator `cf5dceb` | `wakeme_logic.ak:811-818` chỉ ép `d_unit ≥ 1`, `d_unit ≤ oil_per_lamp`, `conditional_lamp == 1001·d_unit` — không có mệnh đề nào buộc `d_unit` theo số dư pot |
| 2026-08-28 | Database `99b33e2` | `PUBLIC_POST` (`AuthRequiredInterceptor.java:108-118`) gồm 9 đường, **không** có `/identity/asset/create` và **không** có `/auth/token/exchange` |
| 2026-08-28 | Database (chạy thật) | `GET /api/v1/identity/fingerprint/status` → **400** (tham số sai), không phải 401 ⟹ đường tra công khai thật sự mở |
| 2026-08-28 | Database `99b33e2` | `ActivationVaultServiceImpl` mang `@Service @Primary`; `buildGetLamp` gọi `blockfrost.getCurrentSlot()` → `preflight.run()` → `txBuilder.buildUnsignedTx()`. 501 đến từ 3 biến rỗng ở `application.yml:142-144` |
| 2026-08-28 | Core `bd8ae3d` | `git grep 'magic/claim\|claimMagic' -- lib` = **0** — app không còn gọi endpoint 410 |
| 2026-08-28 | Core + Validator + Database | quét `git grep -i frost` trên toàn bộ `git rev-list --all`: mọi kết quả là `Blockfrost`, trừ chuỗi `FROST` trong tên tài liệu thiết kế ở chú thích `commitment_tree.ak:5` / `pa2_smt.ak:5`. Không có hiện thực |

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
