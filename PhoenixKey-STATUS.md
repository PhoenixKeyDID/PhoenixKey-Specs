# PhoenixKey — STATUS (hiện trạng & tiến độ)

> **File này là báo-cáo hiện-trạng, KHÔNG phải đặc-tả.** Bộ spec (`*-Vi-Feat/-Math/-Tech/-Exec`) là **kim-chỉ-nam thiết-kế** — mô tả hệ thống ĐÍCH mà các đội dev xây tới. File này ghi *đang ở đâu trên đường tới đó*: cái gì đã chạy, chặn bởi ai, bằng chứng test. Khi hai bên lệch → spec là mục-tiêu, STATUS là thực-tại.
>
> **Cập nhật: 2026-09-02.** Mọi số dưới đây đo lại từ đầu trên `main` của từng kho, không kế thừa bản trước. Bản 2026-08-12 có **8/9 hash validator đã lỗi thời** và bốn chỗ mô tả sai chiều — xem §7.

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

## 1. Hệ đang đứng ở tầng v1.0, và chưa có mốc chuyển sang v1.1

Đây là điều quan trọng nhất trong toàn bộ file, và nó không nằm ở module nào cả.

**Phương pháp neo DID có HAI TẦNG, và điều đó là chủ đích.** `PhoenixKey-SDK/method.md:144-148` (did:phoenix v1.0, Implementors Draft) đặc tả resolver hai chế độ: TAAD UTxO datum là **v1.1, có thẩm quyền khi tồn tại**; metadata nhãn 6789 là **v1.0 MVP, tầng đang chạy**; trong giai đoạn chuyển tiếp hai neo cùng tồn tại và TAAD thắng nếu có. Vậy nên bản thân metadata-6789 **không phải một sai lệch** — nó là tầng đã chốt.

`PhoenixKey-Database/.../cardano/CardanoServiceImpl.java:70-111` — `publishDocument()` trả tiền cho chính ví phí backend, gắn metadata nhãn 6789, ký bằng ví phí rồi submit không chờ xác nhận. **Không mint state-NFT, không gọi `taad`.** Đường này dùng chung cho cả ba loại DID đang cấp được: PersonDID (`/identity/register`), OrgDID (`/org/create`), AssetDID (`/identity/asset/create`).

Mã tự khai đúng như vậy — `CardanoService.java:9-13`: *"Cơ chế hiện tại (transitional) … Sẽ chuyển sang TAAD UTxO datum khi validator state-NFT xong."*

`PhoenixKey-Validator/deploy/README.md:23` — bản hash TAAD preprod duy nhất từng được chốt trong `deploy/` tự khai **"ĐÃ CHẾT — đừng deploy"** (lý do: `ValidatorParams` đổi 6→7 trường). Tức bản đó lỗi thời *trước khi* deploy.

**Vấn đề không phải v1.0. Vấn đề là ba điều sau.**

1. 🔴 **Tầng v1.1 chưa tồn tại ở phía backend.** `git grep 'GenesisPerson\|GenesisChild\|StateNftRedeemer'` trong mảng Java = **0**. Validator đã build và test xanh, nhưng không có bên nào gọi được nó. PoP-bind, CanOwn, 2-of-2, uniqueness anchor do đó là **thiết kế đã hoàn thiện nhưng chưa được thi hành trên chuỗi**; bất-biến danh tính hôm nay do Postgres giữ.

2. 🔴 **Tầng v1.0 đang sinh ra dữ liệu mà tầng v1.1 sẽ từ chối.** Xem đoạn khuôn `did` ngay dưới. Càng cấp thêm DID theo khuôn cũ thì cái giá của đợt chuyển càng lớn — đây là món nợ **tăng theo thời gian**, không đứng yên.

3. 🔴 **Chưa có mốc chuyển.** Đợt redeploy mang tên PA-1 là chỗ v1.0 → v1.1 xảy ra, nhưng nó đang chờ bốn việc chưa chốt: schema `TAADDatum`, nâng `taad_did.rs` từ 10 lên 15 trường, chọn `uniqueness_bootstrap_seed`, xử hai UTxO Preview còn lại — cộng thêm khuôn `did` (`PhoenixKey-Validator` #78, OPEN từ 2026-08-12, hỏi thẳng *"bao giờ luồng thật chuyển sang đúc anchor qua `state_nft_logic`"*).

Nói gọn: nguyên tắc **an ninh trước trải nghiệm, quyền phải ở ledger** không bị v1.0 vi phạm — nó bị vi phạm nếu hệ **dừng lại ở v1.0 mà không có ngày chuyển**.

**Điều kiện tiên quyết bị bỏ quên — khuôn `did` lệch.** Khuôn canonical DUY NHẤT là **PoP-bind** (`pop_bind.ak`: tiền-ảnh `enc_type ‖ enc_creator ‖ be8(genesis_ms) ‖ rand_256 ‖ controller_pkh`, mốc POSIX-**ms** render **base32 8-byte-BE ⟹ 13 ký tự**, hex thường). Backend/app/frontend vẫn sinh did theo khuôn CŨ đã bỏ (`DidPhoenixGenerator.java:43,102`, `taad_did.rs:129-167`): mốc là **slot**, creator nối thô hoặc literal `"root"`, và **không có `controller_pkh`** trong tiền-ảnh. Hai khuôn CÙNG hình dạng chuỗi (`[a-z2-7]{13}` + 64 hex) nên regex không phân biệt được — lệch chỉ lộ ở giá trị hash. Bản Java theo khuôn mới **có mã nhưng đang tắt**: `application.yml:171 pop-bind-encoder-enabled: false`. Bản Rust theo khuôn mới có ở `rust_core/src/pop_bind.rs` (đối byte với vector Aiken, `docs/vectors/canonical_did.json`) nhưng **cố ý chưa nối FFI**, chỉ gọi từ test. ⟹ **mọi DID cấp hôm nay không mint được anchor dưới validator hiện hành.** Và vì `N(did) = blake2b_256(did)` là apply-param của `did_payment`, đổi khuôn = đổi địa chỉ ví của mọi DID ⟹ việc này phải đi qua cổng THỜI-CHÍNH, không vá lẻ. Theo dõi ở `PhoenixKey-Validator` #78.

---

## 2. Bảng trạng-thái 8 module

| Module | Nền đã chạy được | Blocker chính | Production |
|---|---|---|---|
| **Anchorme** | `taad` Design-2 + PC (uniqueness anchor) + PoP-bind, đã nối và test xanh; resolver W3C; register metadata-6789 (tầng v1.0) | tầng v1.1 chưa có bên gọi; khuôn `did` lệch (#78); DeviceDID; resolve-by-hash | NO-GO — chặn ở đợt PA-1 (§1), không ở CID-1 |
| **Rebirthme** | ví theo-DID `did_payment`, đóng-băng theo trạng-thái, guardian recovery ngưỡng+timelock, P-256 low-s, `limit_meter_vault` + `did_stake` build được | tầng **Dart** của khoá thiết bị chưa có (xem §3); `did_subaddr.ak` chưa tồn tại trên `main` | NO-GO ví-giá-trị-lớn |
| **Wakeme** | `wakeme_vault`/`wakeme_logic` merge, test xanh; backend có **hiện thực thật** (`ActivationVaultServiceImpl` `@Primary` + preflight + tx-builder + chọn pot UTxO) | 3 biến `activation-vault-cbor-hex`/`pot-address`/`keeper-pkh` còn rỗng (`application.yml:142-144`) ⟹ trả 501 — **chặn ở deploy, không ở mã**; app không có màn hình Wakeme; **không có mắt nối sang Gen MAGIC** | NO-GO |
| **Feecover** | ConsumeMAGIC lõi (kế thừa) | **0 validator Feecover tồn tại**; CARP chưa có policy-id thật; hai đầu tiêu thụ chưa cắm vào | NO-GO |
| **Protectme** | `protectme_logic` + `protectme_payout` + **`protectme_beacon_logic` (889 dòng, 26 test, ĐÃ NỐI)** — 75 test trên `main` | 2-bucket Treasury + resolver claim + UI chưa code; 11 quyết-định PROT-1..11 | NO-GO — nhưng blocker beacon **đã gỡ** |
| **Knowme** | Mức 1+2 SD-VC có code+test, demo `/vc` — 425 test PASS | lib BBS (Mức 3); LampNet gateway; StampRecord | M1 chạy |
| **Easteregg** | `did_pool` build được | `did_subaddr.ak` chưa tồn tại; ZK Tầng 2 verifier chưa viết; 0 test Easteregg | NO-GO |
| **Smartsend** | validator `smartsend` build được | verifier Glint/Spectra (Phase 2); phụ thuộc nền ví + guardian | NO-GO; money-critical |

---

### 2b. AssetDID + phả hệ nông sản (mới 2026-08-20)

Không phải module thứ 9 — là một năng lực cắt ngang Anchorme, dựng cho truy xuất nguồn gốc nông sản (OriLifeTrace).

| Thứ | Trạng thái | Bằng chứng |
|---|---|---|
| `EntityType.Asset` on-chain | **có sẵn**, `can_own(Person\|Org\|Service, Asset) = True` | `state_nft_logic.ak:92-135` |
| `can_own(Asset, Asset)` | **False** — Asset là lá thụ động (§22.1) | `state_nft_logic.ak:131`, `PhoenixKey-Math.md:2209-2233` |
| `POST /identity/asset/create` | có, đòi Bearer (interceptor mặc-định-từ-chối) | `IdentityController.java`, test `assetCreate_noBearer_401` |
| 8 lỗ trên đường đó | **đã vá**, PR Database #200 chưa merge (**659** test, 0 đỏ) | 5 lỗ đầu: chữ ký thiếu `locationProof` · khoá active-đầu-tiên ký thay chủ · `assetClass` tự phong · owner ép Person · neo rỗng bằng chứng. 3 lỗ do chính bản vá mở ra: cam kết không muối lộ ô GPS · hai mã lỗi mới trùng số · chưa ai đọc cam kết để đối chiếu |
| Muối cam kết on-chain | **có**, 32 byte mỗi bản ghi, migration `V42` | cam kết không muối cho đoán-và-đối-chiếu ra đúng ô GPS vườn: `assetClass` 13 giá trị · `physicalIdHash` tính lại được từ ảnh cây công khai · `locationProof` = H(ô ~111 m), không gian ~10⁷ |
| `GET /identity/{did}/commitment-check` | **có**, đòi Bearer, **6** trạng thái | `MATCH · MISMATCH · NO_COMMITMENT_LEGACY · NO_SALT_UNVERIFIABLE · NOT_ANCHORED · NOT_AN_ASSET` |
| SDK client | **có**, PR SDK #16 chưa merge (107 test, 0 đỏ) | `verifyAssetCommitment` đối chiếu CHÉO NGÔN NGỮ với hàm Java sản xuất chạy qua `jshell` — cùng chuỗi byte |
| UI Flutter | `entityType` hard-code `'PERSON'` | `create_identity_screen.dart:261` |
| Validator phả-hệ `lineage` | **đã dựng + đã tấn công + đã vá**, PR Validator #89 chưa merge | 871 test 0 đỏ; hash `5cff3fc3…`; **9 hash validator cũ giữ nguyên từng byte** |

**Hai số quyết định chi phí, đo được:**
- **min-ADA ≈ 2,36 ADA / DID** — `(388+160)×4310`; 388 byte là CBOR `TxOut` đầy đủ theo CDDL, 4310 là `coinsPerUTxOByte` lấy sống từ Koios mainnet epoch 650.
- **Đúc được ĐÚNG 1 state-NFT / giao dịch** — `state_nft_logic.ak:246-252` (`has_non_unit`). Ràng buộc **thiết kế**, không phải tài nguyên: ExUnit mới dùng mem 6,8% / cpu 3% trần.

🔴 **min-ADA KHÔNG thu hồi được.** `taad_logic.ak:97-98` ép output tiếp diễn mang đúng NFT ở đúng địa chỉ script **vô điều kiện, trước mọi nhánh redeemer**; `TAADRedeemer` (`types.ak:161-277`) **không có Burn**; `Deactivate` (`:310-334`) chỉ đổi `status`. ⇒ 100.000 quả = **236.188 ADA khoá vĩnh viễn**. Cấp TAAD AssetDID cho từng quả là không khả thi — phải phân tầng, quả nằm trong lá Merkle của UTxO lô.

**Phả hệ `lineage` — 8 lỗ do audit + red-team dựng PoC, đã vá TRƯỚC khi deploy** (validator chưa lên chuỗi nên sửa còn miễn phí):

| | Lỗ | Vá |
|---|---|---|
| V1 🔴 | `waste` rửa NGƯỢC thành `mass` — đẳng thức đặt trên TỔNG nên hai sổ khả hoán hai chiều; bội số rửa hàng bằng đúng phân số hao hụt sinh học (cà phê ~4,5×) | `waste` đơn điệu tăng |
| V2 🔴 | Rút một lá vô số lần — `Split` nhân bản cả `merkle_root` lẫn `next_index` | Con luôn `next_index == 0`, `merkle_root == None` |
| V3 🔴 | Nhân bản trùng byte mọi trường trừ `attestor_did` — `unit_id` con tự chọn, `origin_root` thành tự-khai gián tiếp | **Suy `unit_id` theo đường đi**, duy-nhất chứng minh bằng quy nạp từ gốc one-shot |
| V4 🔴 | Lá Merkle không cam kết khối lượng | `lá = H(0x00 ‖ unit_id_con ‖ be8(mass))` |
| V5 🟡 | Thu hồi DID bên chứng thực hoá gạch cả `Destroy` ⇒ phạt đúng hành vi bảo mật đúng đắn | `Destroy` nhận anchor mọi trạng thái, vẫn đòi controller ký |
| V6 🟡 | Trường `did` tự khai ⇒ mạo danh tài sản | Con kế thừa `did` của cha; đặt mới chỉ ở `Genesis` có cổng |
| V7 🟡 | L-2 chỉ ghim `payment_credential` ⇒ đổi stake credential là biến mất khỏi bên giám sát quét theo địa chỉ | So toàn địa chỉ |
| V8 🟡 | Trần Merge 20 không bao giờ đạt được | Tách `max_merge_parents = 4` |

**Trần Merge là trần THỰC THI, không phải trần logic.** Merge chi n UTxO ⇒ ledger chạy validator n+1 lần, mỗi lần lại O(n) ⇒ bậc hai. Đo mem toàn tx, trần 14 M: n=4 → 62,3 % · n=6 → **111,3 % ✗** (hình dạng xấu nhất: 5 output ví × 30 policy). Đại lượng đội chi phí là **số policy** trên một output, không phải số token — `assets.tokens` tra một khoá trong từ điển policy.

**`Commit` là đường DUY NHẤT sinh ra một lô** (L-21), một chiều `None → Some`. Nhánh này **không** kiểm Σ khối lượng lá, có chủ ý: `promote_leaf_ok` trừ dần `lot.mass_mg` ở mỗi lần rút, nên trần nằm ở khối lượng lô chứ không ở cây — cam kết một cây khai 10 tấn trên lô 1 tấn vẫn chỉ rút được 1 tấn. Kiểm Σ đòi cả cây lên chuỗi, mất sạch ý nghĩa ở quy mô 100.000 lá.

**CIP-113 đã khảo sát và LOẠI:** Status *Proposed*, PR `cardano-foundation/CIPs#444` chưa merge từ 2023-01, `cips.cardano.org/cip/CIP-113` trả 404 (đối chứng CIP-68 tải được), repo chính chủ tự ghi *"do not deploy to mainnet or use with real assets"*; và nó là chuẩn **quyền chuyển**, không có ngữ nghĩa transformation/bảo toàn khối lượng. CIP-143 = *Inactive*. CIP-25 = label 721, Plutus không đọc được. CIP-68 = *Active*, dùng làm lớp trình bày — ⚠ label 444 (RFT) là *fungible* nên **không dùng cho hộp bán lẻ** (mất định danh cá thể = mở đường rửa nguồn gốc). GS1 EPCIS `TransformationEvent` đúng ngữ nghĩa nhưng mọi trường Optional, **không ràng buộc số học nào**.

**Ranh giới phải nói với người ngoài:** on-chain chỉ bảo đảm không tạo được khối lượng từ hư không **sau khi đã nhập**. Cái cân nằm ngoài chuỗi; điểm nhập là oracle. Đường tấn công rẻ nhất — thuê một nông dân thật, chở hàng ngoài tới vườn, đăng ký tại gốc — làm mọi cổng đều xanh, và không cơ chế on-chain nào chạm tới.

**Bốn ranh giới còn mở, chưa có cơ chế** (nêu để không ai đọc quá lời):
1. `lineage` **không có khái niệm chủ sở hữu** — cổng duy nhất là chữ ký bên chứng thực, và `Destroy` không ràng buộc ADA chảy đi đâu ⇒ về kinh tế min-ADA của mọi đơn vị thuộc về bên chứng thực.
2. **Merge không mang tỉ phần** — gộp hai lô cùng bên chứng thực, cùng tầng thì con ra là một con số khối lượng duy nhất; mọi hộp con sau đó bit-identical.
3. **Trần `depth_phys = 8`** — chuỗi nông trại→thửa→cây→lô→quả→phân hạng→đóng gói→thùng→hộp đã ăn hết 8 trước hạ nguồn, chưa trừ đóng gói lại.
4. **Cổng bên chứng thực là 1-of-1** (`controller_pkh`), trong khi `did_payment`/`did_stake`/`wakeme_vault` đều 2-of-2 qua `auth_logic.anchor_controller_ok`.

**Chưa dựng tx thật trên Preview** — mọi số ExUnit là aiken tự đánh giá, chỉ báo chứ không phải số ledger thật.

---

---

## 3. Ma-trận blocker xuyên-module

| Blocker | Chặn | Đội gỡ | Trạng thái đo được |
|---|---|---|---|
| 🔴 **Đường đúc không qua validator** (§1) | TẤT CẢ | on-chain + backend | `CardanoServiceImpl.java:70-111`. Chưa có phương án chốt (xem §6) |
| 🔴 **Khuôn `did` lệch** | Anchorme, Rebirthme | on-chain + backend + app | `application.yml:171` đang `false`; Validator #78 |
| 🔴 **Khoá thiết bị — tầng Dart** | Rebirthme (ví Phoenix) | app | Rust **đã xong**: `device_key.rs` 893 dòng, `device_pkh` 67 lần / 4 tệp, và **đã xuất FFI C** 4 hàm (`lib.rs:955,996,1039,1060`). Tầng Dart: `git grep taad_device_key -- lib` = **0**. Chỗ đứt là *binding Dart*, không phải mã lõi |
| 🔴 **Lỗ đang NGỦ: cổng ví gác bằng cấu hình, không gác bằng sự thật trên chuỗi** | Rebirthme | backend + app | DID sinh theo khuôn cũ suy ra một địa chỉ mà anchor tương ứng **không bao giờ tồn tại**. Hôm nay chưa mất gì vì bộ suy địa chỉ trả rỗng — nhưng nó rỗng vì **thiếu cấu hình**, không vì thiếu mã: bộ suy thật đã là thành phần chính đang chạy, chỉ đứng im khi hai giá trị cấu hình chưa đặt. Nghĩa là cổng này lật được bằng cấu hình, không cần một dòng mã nào và không qua một lượt duyệt nào — trong khi tác vụ rót ví cho người dùng cũ đã sẵn sàng. Cổng phải kiểm **anchor có tồn tại trên chuỗi không**, không kiểm biến cấu hình có rỗng không. Kèm theo: bản CBOR đang ghim là bản cũ, thiếu vế khoá thiết bị, nên phải ghim lại cùng lượt với vector kiểm thử phía backend |
| 🔴 **Đúc LAMP bằng OrgDID — BẤT KHẢ với token đang sống, không phải "chưa nối"** (đo 2026-08-18) | mục tiêu "OrgDID đúc LAMP" | chủ nhân (quyết định kinh tế) | Bản LAMP đang sống trên mainnet là policy `55d3e01bb6c469e02665e4b6573ce65bbaf7a50ad2024e247eb180f0`, **8 tham số**, WHO-gate = **một pkh nướng cứng** + `extra_signatories` — **không đọc registry, không đọc DID, không reference input nào** (`LAMP/Genesis/offchain/src/deployed.ts:75`). Bản registry-gate **12 tham số** (`Genesis/onchain/validators/lamp_mint.ak`, main `44ccd79`) mới đúng nghĩa "đúc qua OrgDID" nhưng **chưa deploy ở bất kỳ mạng nào** và deploy nó = **policy-id khác = một token KHÁC**. SupplyState đã bootstrap thật trên mainnet (tx `db0610c2…`, 1 triệu LAMP) nhưng dưới policy 8-tham-số. ⇒ không có đường vá; chỉ có đường ra token mới |
| 🔴 **Một lời gọi công khai khoá vĩnh viễn đường khôi phục-bằng-24-từ của người khác** | khôi phục thiết bị | backend | TAAD pubkey được publish công khai làm `#controller-key` (`W3CDocumentBuilder.java:158-168`, `/identifiers/**` công khai). `POST /identity/register` công khai và **không kiểm sở hữu** `taadPublicKeyHex` — javadoc `IdentityRegisterRequest` ghi thẳng, kèm kết luận "không ảnh hưởng user khác". Cột cố ý **không UNIQUE** (`V38`). `ResolveByControllerService` fail-closed 404 khi `matches.size() > 1`. Ghép lại: đọc khoá của nạn nhân rồi ghi vào bản ghi của mình ⇒ nạn nhân cầm đúng 24 từ vẫn nhận 404 vĩnh viễn. Vá tối thiểu: UNIQUE theo khuôn `V36`. Vá gốc: challenge ký bằng chính TAAD_Key. Database Issue #213 |
| 🟡 **Vai `manager`/`viewer` không được cưỡng chế ở đâu** — *bản vá đã có, chờ gộp* | đa thiết bị (§5) | backend | **Trên `main` lỗ vẫn nguyên:** `SessionServiceImpl.java:167-168` lập phiên không xét vai; chỉ **hai** tệp trong `src/main/java` đọc tới vai; `AuthenticatedUser` chỉ mang DID ⟹ phiên lập bằng khoá `viewer` gọi được đúng những gì phiên khoá chủ gọi được. **Bản vá: PR #215** (6 commit, 875/875 test, 6 ca trên Postgres thật) — vai vào phiên, cưỡng chế ở một chỗ (`AuthRequiredInterceptor`), ba đường vòng đời thiết bị, và bịt sáu đường vòng tự soi ra được (xem §8). Blocker này chỉ được xoá khỏi bảng **sau khi #215 gộp**. Database Issue #211 |
| 🟡 **`op-seq`: mã nói công khai · cổng trả 401 · `API.md` nói đã bị bác** | app khôi phục mốc chống phát lại | backend | `GET /identity/{did}/op-seq` tồn tại với javadoc lập luận nó **phải** công khai, nhưng không có trong `PUBLIC_GET` ⇒ trả 401 cho đúng client nó phục vụ; `API.md:741-755` vẫn ghi đề xuất này ĐÃ BỊ BÁC. Không ca kiểm nào chốt hướng nào. Có ứng dụng ngoài đang giữ PR treo chờ. Database Issue #210 |
| 🔴 **ServiceDID / AgentDID / DeviceDID = 0 dòng backend** | Anchorme, LampNet, Knowme | backend | `types.ak` đã đủ 5 loại + ma trận `can_own` (validator sẵn 100%); `grep DidType.SERVICE\|AGENT\|DEVICE` trong `src/main/java` = **0**. `DeviceController.java` là push-token FCM, **không phải** DeviceDID |
| 🔴 **Khoá phiên đứt ở HAI chỗ độc lập** | khoá phiên (§5) | backend | (a) `/auth/token/exchange` nhận `sessionToken` trong BODY nhưng không có trong `PUBLIC_POST` (`AuthRequiredInterceptor.java:108-118`) ⟹ đo thực địa trả `401 {"code":1304}`, không bao giờ chạm `TokenExchangeService`. (b) Không API nào ghi được `service[] type=SsoRedirect` lên DID Document ⟹ dù mở (a) thì vẫn chưa đăng ký được website nào. Gỡ một chỗ không đủ |
| 🔴 **B1 — MAGIC engine đọc-số-dư** | Wakeme, Feecover, Protectme | MAGIC team | Model = account-in-vault (không native) |
| 🔴 **B2 — CARP policy-id thật** | Feecover, Protectme, Rebirthme, Wakeme | CARP team | Đang để all-zero fail-closed; test dùng hằng giả |
| 🔴 **FROST vs VSS — hai bên xây hai nguyên thuỷ khác nhau** | phân tán seed (§5) | PhoenixKey + LampNet | Chốt là FROST-sign; bản đã build là VSS/Shamir. **FROST không có hiện thực ở nhánh nào**: quét toàn bộ lịch sử Core/Validator/Database, chuỗi `FROST` chỉ xuất hiện trong tên một tài liệu thiết kế được trích ở chú thích (`commitment_tree.ak:5`, `pa2_smt.ak:5`); mọi kết quả còn lại là `Blockfrost`. `seed_envelope.rs` 1284 dòng đã merge nhưng **0 hàm xuất FFI** ⟹ Dart không gọi tới được |
| 🟡 `GenesisChild` 1 chữ ký | Anchorme | on-chain | `state_nft_logic.ak:198` đòi 1 chữ ký, trong khi mọi đường CHI là 2-of-2 (`auth_logic.ak:137,143`) |
| 🟡 `app_token` là bearer thuần | khoá phiên | backend + SDK | `PhoenixKey-SDK/INTEGRATION.md:113` tuyên bố đã có DPoP — **tài liệu nói quá mã** |
| ~~PA2 UniquenessThread~~ | — | — | ✅ loại vĩnh viễn, thay bằng PC (đã land + nối) |
| ~~PA5-a entity-gate~~ | — | — | ✅ chốt không nối — `entity_type` đã cam-kết ở tầng mint (`pop_bind.inner_hash`) |
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
| Tạo ví **Phoenix**, nhận tiền | **CHƯA BẬT** | Không phải hỏng — là chưa mở. `AikenPhoenixCustodyDeriver` **đã là bean chính** (`@Component @Primary`) và trả rỗng chỉ vì hai biến `did-payment-cbor-hex` và `taad-anchor-policy` còn rỗng (`:119,126`) — đặt hai biến đó là bật, không cần sửa mã; app hiện đúng dòng *"Chờ TAAD deploy trên PreProd"* (`wallet_screen.dart:813`); `send_screen.dart` không có tham chiếu nào tới địa chỉ ví Phoenix |
| **Chi tiền từ ví Phoenix** (2 yếu tố) | **KHÔNG** | Rust + FFI đã xong; **binding Dart chưa viết** ⟹ app không gọi được |
| **Một người một DID** | **KHÔNG** | Cổng duy nhất là `existsByPublicKeyHex` (`IdentityServiceImpl.java:107-113`) ⟹ thực chất **một MÁY một DID**. Registry giấy-tờ chưa nối vào `/identity/register` nhưng đường ghi trực tiếp đã mở và chưa an toàn (§4b). Mất máy = tạo DID mới = chính nguồn sinh trùng |
| **Khôi phục trên máy thứ hai ra đúng DID cũ** | **ĐƯỢC, phải TRA** | `did` gấp 32 byte ngẫu nhiên **và** một mốc thời gian trên chuỗi vào tiền-ảnh ⇒ không suy lại được từ seed. Cửa tra theo khoá điều khiển đã có: `POST /identity/resolve-by-controller` (Database PR #171, 2026-08-12) |
| **Người thứ hai tạo DID riêng trên máy người trước** | **KHÔNG** (đã vá, chờ gộp) | Backend và validator vốn không ràng buộc duy nhất theo thiết bị; chặn nằm toàn bộ ở ứng dụng. Core PR #56 vá, CI xanh, chưa gộp |
| Tạo **OrgDID** | **ĐƯỢC** | nhưng chỉ là giao dịch metadata (§1) |
| Tạo **AssetDID** | **ĐƯỢC** | ⚠ không kiểm quyền sở hữu (§4a) |
| Tạo **ServiceDID / AgentDID / DeviceDID** | **KHÔNG** | 0 dòng backend — `/devices/register` (`DeviceController`) là push-token FCM/APNs, KHÔNG tạo DeviceDID; `grep` `DidType.DEVICE` = 0. Validator sẵn sàng ở tầng `can_own`, nhưng **client cũng chặn**: `rust_core` `build_create_taad_utxo_tx` (`taad_did.rs:1631-1636`) trả `Err("only Person genesis wired for mint; child genesis = follow-up")` cho mọi `entity_type != 0` ⟹ ngay cả khi backend chuyển sang đúc anchor thật, hôm nay chỉ dựng được tx cho Person |
| **Phân tán bộ seed trên LampNet** | **KHÔNG** | Backup đang chạy là **một blob tới một CID**, khoá dẫn từ Master_KEK (`derive_x25519_static_from_kek`, `lampnet.rs:149`) ⟹ ai có 24 từ là mở được. Không k-of-n. Nguyên liệu để sửa **đã có sẵn cùng crate** — `seed_envelope::generate_k_env()` (`lampnet.rs:141-143` xác nhận), còn thiếu là nối + di trú blob v1. Và hai bên đang xây hai nguyên thuỷ khác nhau (§3) |
| **Cấp khoá phiên** cho app/website | **KHÔNG** | `/auth/token/exchange` trả 401 vì thiếu 1 dòng whitelist (§3); và không API nào ghi được `service[] type=SsoRedirect` lên DID Document ⟹ chưa đăng ký được website nào |
| Ứng dụng **bên thứ ba** đăng nhập bằng PhoenixKey | **KHÔNG** | cả hai chỗ đứt ở §3 đều phải gỡ; thêm nữa chưa có màn hình đồng ý cấp quyền |
| **Nhận LAMP** từ ETD / Airdrop / SRCL | **KHÔNG**, cả ba | không có đường `POST claim` nào ở phía PhoenixKey |
| Đăng nhập web PhoenixKey bằng QR | **ĐƯỢC** | đo thực địa 2026-08-28: `POST /auth/session/init` → 200; `GET /api/v1/.well-known/jwks.json` → 200, `kid=phoenixkey-ed25519-1` |
| **Wakeme / mượn 1001 LAMP** | **KHÔNG** — bốn lớp chặn ĐỘC LẬP | (1) `AikenActivationVaultDeriver.init()` (`:94-128`) đòi **5** biến non-blank + parse hex 28-byte + bech32 hợp lệ mới `active = true`; `application.yml:142-144` để trống 3 biến bằng default `${…:}`, hai biến `taad-anchor-policy`/`lamp-policy-id` không xuất hiện trong `application.yml` luôn ⟹ rơi về `NoopActivationVaultDeriver` ⟹ `requireDeriverActive()` (`ActivationVaultServiceImpl.java:207-213`) ném `NOT_YET_IMPLEMENTED`. (2) **Đặt đủ 5 biến vẫn chưa chạy**: `TaadDatumReader`/`TaadAnchorLocator` cần một **TAAD anchor UTxO thật** làm reference-input, mà 100% DID đang sống là metadata-6789 (§1) ⟹ không user thật nào có anchor để tham chiếu. (3) App không có màn hình Wakeme — `grep -rli wakeme` trên `rust_core/src/` = **0**, trên `lib/**/*.dart` = 1 kết quả và là một dòng chú-thích (`wallet_screen.dart:958`). (4) Công thức D không bị ép on-chain (xem bảng lệch spec↔mã bên dưới). Pot 1.001 tỷ LAMP chỉ có trong tài liệu, chưa có bằng chứng deploy |
| **Gen MAGIC / Consume MAGIC** qua Wakeme | **KHÔNG** | `grep wakeme` trong toàn bộ validator của kho MAGIC = **0**. MAGIC dùng vault riêng ⟹ **không có mắt nối** giữa hai bên |
| **Feecover trả hộ phí** cho AladinWork / OriLife | **KHÔNG** — nhưng lớp on-chain đã có mã, đang chờ gộp | **Validator: có, chưa gộp.** `validators/feecover.ak` + `lib/phoenixkey/feecover_{types,logic}.ak` trên nhánh `claude/feecover-validator` (PR Validator #95, 2026-09-02): `aiken check` → **980 checks, 0 errors** (feecover 67/67, feecover_logic 11/11), mỗi dòng bảng §5 của `PhoenixKey-Feecover-Math.md` có test dương + test âm. FG-4 ép được cả ba vế (settle permissionless, `total_carp == Σ accrual` lấy value làm sự thật, đích + `stabilizer_ref` bất-biến). Trước đó kho có **0 validator Feecover** — 4 chỗ có chữ `feecover` chỉ là chú-thích `[FEECOVER-HOOK]` (`types.ak:360-365,394-398`, `taad.ak:28-29,124-129`, `taad_logic.ak:108-116`, `smartsend_logic.ak:109-110`).<br>**Còn chặn:** validator nằm trong `STUB_VALIDATOR_TITLES` vì `carp_policy` (B2) và bảng `fee_magic` (D1) chưa chốt — nhận làm apply-param, không chặn việc viết mã nhưng chặn deploy. `feecoverAdvance` (FEECOVER-PRICE-2) vẫn chỉ là tên hàm trong spec — `grep -rn feecoverAdvance` trên toàn `PhoenixKeyDID` + `MagicLampEco` = **0 tệp mã**. Backend: `grep -rln "feecover\|consumemagic" -i` trên `PhoenixKey-Database/src` = **0**. FEECOVER-PRICE-1 (`did.rotate` không nhân theo cầu) **không làm được ở phía Feecover** — nó đòi ConsumeMAGIC có lớp giá cố-định; tự làm bên này = tạo bảng giá thứ hai và phá `magic_consumed == fee_magic`. Phần "một ví đứng ra trả hộ phí mạng" vẫn **chưa được thiết kế ở tài liệu nào** — Feecover Tech §10 chỉ định tuyến CARP |
| **Consume MAGIC** để gắn vào **AladinWork** | **KHÔNG trên chuỗi**, off-chain đã xong và hỏng-đóng đúng | `AladinWork/Core/consume-magic.js` (312 dòng) là adapter tái hiện-thực công-thức `ConsumeMAGIC/CONTRACT.md §A`, tự khai rõ (`:6-30`) là KHÔNG `require` gói thật vì `@magiclamp/consumemagic-pricing` chưa build được. Nhánh THẬT ném lỗi có chủ đích: `integration/e2e/moneyLayer.js:115,213` → `MISSING_INTERFACE: MAGIC.consume`. Test 792/792 xanh nhưng chạy ở chế-độ **MOCK** (sổ MAGIC trong bộ nhớ, `moneyLayer.js:139-146`). Chặn: MAGIC chưa publish gói, chưa deploy beacon `PriceParam` nào (`MAGIC/scripts/DEPLOYED.md` không có mục ConsumeMAGIC) |
| **Consume MAGIC** để gắn vào **OriLife** | **KHÔNG** — và đang dùng cơ chế KHÁC | `grep -rn ConsumeMAGIC` trên `OriLifeTrace/orilife-fee/src` = **0**. Cơ chế phí đang chạy thật là **LAMP Treasury Collect** (custody CARP trên Preview), không phải ConsumeMAGIC. Spec tự ghi "nguồn tín hiệu thật từ MAGIC ConsumeMAGIC chưa wire" (`OriLife-Specs/Fee/FeeMechanism-FEAT.md:310`) |
| **Mint LAMP vào kho** để phát hành trên **mainnet** | **ĐƯỢC MỘT PHẦN** | Có 1.000.000 LAMP thật trong kho từ tx genesis `db0610c2…` (2026-06-18), policy `55d3e01bb6c469e02665e4b6573ce65bbaf7a50ad2024e247eb180f0` trên **mainnet**, đối chiếu byte 2026-08-12 khớp 2121/2121 với commit `457f312` (`LAMP/Genesis/offchain/src/deployed.ts:57-108`). Nhánh DistributionVest kỹ-thuật đã thông (còn cap 26,369 tỷ, cần 1 chữ ký), công-cụ dựng tx đã vá ở `bd8eabc`. **Chưa có lượt mint thêm nào** kể từ genesis — `dist_minted` vẫn 1e12 oildrop. ⚠ Nhánh Reserve **không vào được** trên policy đang chạy: `meter_nft_policy` = 28 byte 0, mà một token dưới policy đó đòi tiền-ảnh blake2b-224 của 28 byte 0 — chặn bằng ĐỊNH LÝ, không phải bằng độ khó. **Không có LAMP nào bị khoá hay mất**: `reserve_minted` = 0, phần Reserve chưa hề được mint, không ai đang giữ hay mất gì. Cái thật sự mất là **trần hiệu dụng**: 26,37 tỷ thay vì 36 tỷ, cho tới khi có policy MỚI + di-trú (Cardano không cho đổi tham-số sau apply-param) |
| **Mint CARP** qua **CarpetMint** | **ĐƯỢC trên testnet, CHƯA trên mainnet** | Preview: mint tx `225393631cad…`, Dutch full-clear 6 tx thật 2026-08-28. Preprod: instance sống `CARP@bootstrap-tlamp`, mint tx `005159c40c6d…`, đúc thật từ trình duyệt qua ví CIP-30 (tx `11d82eecdd9e…`). `aiken check` trên `origin/main` `b2e3dc5` → **352/352, 0 lỗi**. Mainnet: `CarpetMint/DevStatus.md:52` ghi thẳng "chưa làm". ⚠ Ghi chú "`carp_policy` chưa deploy = BLOCKER-2" trong `Feecover/DevStatus.md` và `Protectme/DevStatus.md` dẫn `MAGIC/SPEC/Carpet-Tech-Vi.md §T1` nay **trỏ sai nhà**: CarpetMint đã tách khỏi `MAGIC/` sang kho riêng. Blocker vẫn đứng cho production, nhưng lý do là "chưa lên mainnet", không phải "chưa có mã" |
| Xem **danh sách người bảo trợ** | **KHÔNG** | không có `GET /guardians`; `/add`+`/remove` chỉ trả về SỐ LƯỢNG — mà đây là màn hình bắt buộc trong luồng khôi phục |
| **Từ chối** một yêu cầu ký | **KHÔNG** | không có `POST /sign/{id}/reject`; chỉ `approve`/`cancel`, dù enum trạng thái đã có sẵn `"rejected"` |

### Tài liệu công khai đang nói quá mã

`PhoenixKey-DIDMethod-W3C.md` — bản đã đăng ký ở `w3c/did-extensions` — mô tả mỗi DID được neo **một-đối-một bằng một NFT singleton trên Cardano** (dòng 5), và §6.2 ghi *"The authoritative state is always the on-chain TAADDatum"*. `grep 6789` trong tệp đó = **0**.

Nhưng **100% DID đang sống là metadata-6789**. Bản đăng ký công khai mô tả tầng v1.1 như thể nó là tầng duy nhất.

Nặng thêm: hai đặc tả công khai của cùng một phương pháp nói hai điều khác nhau — `PhoenixKey-SDK/method.md:61-64` gọi TAAD là "v1.1 (target)" và metadata-6789 là "v1.0 live today", còn `PhoenixKey-DIDMethod-W3C.md` chỉ có một bản. Người ngoài đọc bản W3C rồi viết resolver theo nó sẽ không giải được DID nào đang tồn tại.

Chiều NGƯỢC lại cũng có, và nó nguy hơn vì không ai đi kiểm chiều đó: `PhoenixKey-DIDMethod-W3C.md` §5.1/§5.3 ghi `"service": []` và *"resolver publishes an empty `service` array when none is registered"*. Mã thì luôn publish `#cardano-wallet` mang **địa chỉ ví thật** (`W3CDocumentBuilder.java:81-89`, gọi vô điều kiện từ `ResolverServiceImpl`), và `/identifiers/**` là đường công khai. Nghĩa là **công bố một chuỗi DID = công bố địa chỉ ví = công bố toàn bộ lịch sử giao dịch của người đó**. Một nhà bên ngoài đọc đặc tả rồi kết luận đây là lựa chọn "còn treo, chưa bật" — nó đã bật từ lâu. Đặc tả đã sửa cùng lượt này. Phần lượt phân giải **hiện tại** có nên tiếp tục trả ví hay không là quyết-định của chủ dự án, không phải bản vá kỹ thuật — xem §6. Lượt phân giải theo mốc quá khứ đã thôi trả `service[]` (Database PR #214).

Và `PhoenixKey-Anchorme-Tech.md:355` — schema trả về của resolver chỉ có `"resolved_from": "onchain-cache | metadata-6789"`, **không có giá trị nào cho TAAD UTxO datum**. Tức schema hiện tại không diễn đạt nổi tầng đích.

### Lệch spec ↔ mã đang sống

| Chỗ | Spec ghi | Mã chạy | Bên nào đúng |
|---|---|---|---|
| Công thức D của Wakeme | `min(1001, ⌊pot×1001/10⁹⌋)` (`Wakeme-Exec.md:17`) | **validator KHÔNG ép công thức này.** `wakeme_logic.ak:811-818` chỉ ép `1 ≤ d_unit ≤ oil_per_lamp` và `conditional_lamp == 1001·d_unit`; `d_unit` là trường datum do builder khai (`:52-53`). Tỷ lệ theo số dư pot nằm hoàn toàn off-chain, tin builder | spec đúng nhưng **chưa được thi hành** — `Wakeme-Exec.md:19` tự ghi việc chốt-cứng với validator để ở PA-1. Ngoài ra bản CŨ `/10⁶` còn sống trong tài liệu và mã Database (vd `ActivationVaultController.java:40`, `ActivationVaultDtos.java:44`) |
| Đơn vị thời gian Wakeme | `vest_start_slot` | `vest_start_ms` (`wakeme_logic.ak:43,131,192-212`) | **mã đúng** — Plutus `validity_range` trả POSIXTime (ms), không lộ slot ⟹ sửa spec |
| DPoP trong SDK | "đã có" (`INTEGRATION.md:91`) | bearer thuần | **mã đúng**, tài liệu nói quá |
| Vai của Glint (VeData) | 17 tệp spec PhoenixKey mô tả Glint là dịch-vụ **phát hiện deepfake / khớp khuôn mặt**, và `Smartsend-Tech.md:104` đặc-tả public-input `FaceMatch/SecretSelfie/DeviceGeo` | Vai media-authenticity **đã rời Glint sang Spectra (LampNet)** từ **2026-07-28** (`VeDataIO/Specs/Glint-Math.md:21`, `:261`). Glint nay là **thư viện primitive ZK-proof**: P1 commit · P2 range · P3 membership · P4 linear · P6 nullifier. Ba tên API kia **không tồn tại** trong catalog. Mã `glint-core` có thật từ 2026-07-30 (`cargo test` → 43 passed) nhưng tự khai **"deterministic (non-ZK)"** — không mạch, không prover, không verifier (`glint-core/src/lib.rs:3-12`) | **Glint đúng, spec PhoenixKey sai** — đã sửa cùng lượt này (nhánh `claude/glint-vai-zk-dinh-chinh`) |

---

## 6. Quyết-định đang chặn tiến độ (không phải việc dev)

Những việc dưới đây **không** thiếu người làm — chúng thiếu một lựa chọn được chốt. Liệt kê ở đây để không ai chờ nhầm.

1. **Ngày cho đợt PA-1** — mốc v1.0 → v1.1. Bốn việc chặn nó đều là quyết định, không phải việc thiếu người: chốt schema `TAADDatum`, nâng `taad_did.rs` 10 → 15 trường, chọn `uniqueness_bootstrap_seed`, xử hai UTxO Preview còn lại.
2. **Khuôn `did`** — bật `pop-bind-encoder`, và di trú các DID đã cấp theo khuôn cũ thế nào: giữ song song, cấp lại, hay đóng băng (Validator #78).
2b. **Ai trả min-ADA cho anchor.** Quy tắc đã chốt là *PersonDID không tính phí, không khoá ADA* — đó chính là lý do tầng v1.0 bỏ đường khoá ADA. Nhưng mỗi anchor TAAD là một UTxO, nên nó **phải** khoá min-ADA. Tiền đó lấy lại được (`GenesisBurn` cho phép đốt thuần, `state_nft_logic.ak:35`; burn theo vòng đời do TAAD spend validator gác), nhưng chỉ khi DID chấm dứt — còn suốt đời DID thì nó nằm đó. Chưa ai chốt ví nào ứng khoản này cho hàng triệu PersonDID, và ứng rồi thì thu về theo đường nào.
3. **Luật đòi lại AssetDID** khi bị đăng ký nhầm/chiếm chỗ (§4a).
4. **Điều kiện được ghi vào registry giấy-tờ** (§4b).
5. **FROST hay VSS** cho phân tán seed — hiện có hai nguyên thuỷ song song, phải chọn một trước khi xây tiếp.
6. **`GenesisChild` có nâng lên 2-of-2 không** — nếu có thì đổi hash, đi cổng THỜI-CHÍNH.
7. **Lượt phân giải hiện tại có tiếp tục trả địa chỉ ví không.** Hôm nay `GET /identifiers/{did}` công khai trả `#cardano-wallet` mang địa chỉ ví thật ⇒ ai biết chuỗi DID là biết ví và toàn bộ lịch sử giao dịch. Đây là hợp đồng công khai đang có bên tích hợp dùng, nên siết nó **không** phải bản vá kỹ thuật: hoặc giữ (và ghi rõ vào tài liệu tích hợp rằng công bố DID = công bố ví), hoặc cho chính chủ tự bật/tắt, hoặc bỏ hẳn khỏi tài liệu công khai và cấp qua đường có xác thực. Lượt phân giải theo mốc quá khứ đã thôi trả (Database PR #214) — phần còn lại chờ chốt.
8. **Paymaster giữ hay bỏ** — treo từ 2026-08-12; phần "trả hộ phí mạng" của Feecover phụ thuộc câu trả lời.

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
- **"đường đúc DID đi sai thiết kế"** — cách nói đó SAI, và bản nháp của chính tài liệu này từng dùng nó. Hai tầng neo là chủ đích, có đặc tả (`PhoenixKey-SDK/method.md:144-148`) và có quyết định gốc (`PhoenixKey-Database` #18, đóng 2026-06-02 — thứ bị bỏ khi đó là một UTxO 5 ADA đậu datum thô không validator nào gác, **không phải** validator `taad`). Cái sai thật là hệ **dừng ở v1.0 mà chưa có ngày chuyển**, và tầng v1.0 đang tích nợ cho đợt chuyển đó.
- **"ví Phoenix đang khoá vốn"** — cũng SAI. Ví Phoenix **chưa bật**: deriver thật đã là bean chính nhưng trả rỗng vì hai biến môi trường còn rỗng, app hiện dòng chờ hạ tầng, `send_screen.dart` không có đường chi. Rủi ro thật là **bật cổng trước khi có anchor** (§3), không phải tiền đang kẹt.

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
| 2026-08-30 | Frontend `22a606b` | Đã gỡ đường tự đúc phiên ở phía trình duyệt (`login/page.tsx`: JWT `alg:"none"` dựng bằng JS rồi `setSession`, kèm auto-bypass qua `?dev=`). Cổng CI cũ grep trong `.next` nên **luôn xanh**; thay bằng cổng quét mã nguồn. Đo lại: lint 0 lỗi, typecheck 0, test **425/425**, build exit 0. PR #23 |
| 2026-08-30 | SDK `f1d3bf0` | `verifier.ts` trước đây chỉ đọc `public_key_hex` và **bỏ qua `status`** ⟹ chữ ký ký bằng khoá đã thu hồi vẫn `valid:true`. Đã trả về bản ghi đủ trường, từ chối `key_revoked`, thêm 5 ca kiểm ký bằng khoá P-256 thật. `npm test` **26/26**, typecheck 0, build 0; phiên bản 0.3.0 → 0.3.1. PR #17 |
| 2026-08-30 | Database `9a27e9d` | `@Operation` của `GET /identity/{did}/pubkey` ghi *"Trả Hardware public key active"* nhưng cài đặt gọi `findLatestOwnerByUserDidAnyStatus` — trả cả khoá đã thu hồi. Đây là **nguồn gốc** của lỗi SDK ở dòng trên: người tích hợp làm đúng theo hợp đồng đã công bố. Đã sửa javadoc + `@Operation` + `API.md`. `./mvnw -o compile` exit 0. PR #212 |
| 2026-08-30 | Database `99b33e2` | `SessionServiceImpl.java:167-168` gọi `existsByUserDidAndPublicKeyHexAndStatus(did, pubkey, "active")` — ba tham số, **không có vai**. `git grep -l 'key_role\|keyRole' src/main/java` = 2 tệp (`KeyServiceImpl`, `IdentityServiceImpl`), `AuthenticatedUser` chỉ mang DID ⟹ vai `manager`/`viewer` không cưỡng chế ở đâu. Issue #211 |
| 2026-08-30 | Database `99b33e2` | `W3CDocumentBuilder.java:158-168` publish TAAD pubkey làm `#controller-key`; `IdentityRegisterRequest` javadoc ghi rõ backend không verify quyền sở hữu `taadPublicKeyHex`; `V38__taad_keys_pubkey_index.sql` cố ý **không** UNIQUE; `ResolveByControllerService` ném 404 khi `matches.size() > 1` ⟹ ba quyết định đúng-riêng-lẻ ghép thành đường khoá vĩnh viễn khôi phục của người khác. Issue #213 |
| 2026-08-30 | Database `43093c7` (nhánh, chưa gộp) | Đăng nhập nhiều thiết bị / nhiều app: vai khoá vào phiên (`key_id`+`key_role`), cưỡng chế tại **một** chỗ `AuthRequiredInterceptor`, ba đường `GET|POST /keys/devices/**`, vai đi theo `app_token` ở `/auth/token/exchange` (không nâng vai). `./mvnw -o test` **875/875**, 0 fail. PR #215 |
| 2026-08-30 | Database `43093c7` | Tham số ma trận đi vòng cả bảng quyền — đo bằng MockMvc: `POST /keys/devices;x=1/abc/revoke` được Spring định tuyến vào đúng controller (`200`) trong khi `EndpointRolePolicy.requiredRole()` đọc ra `VIEWER`. `getRequestURI()` giữ `;k=v`, bộ định tuyến thì gỡ — hai cách đọc một chuỗi. Vá bằng hai lớp độc lập (chuẩn hoá đường thô + đối chiếu `BEST_MATCHING_PATTERN_ATTRIBUTE`, lấy mức cao hơn) |
| 2026-08-30 | Database `43093c7` (Postgres thật) | `AuthorizedKeyReauthorizePostgresTest` **6/6** qua Testcontainers: dựng lại được lỗi trước V48 (thu hồi rồi cấp lại cùng thiết bị → vi phạm `uq_did_pubkey` = HTTP 500), chứng minh V48 cho cấp lại (2 dòng: 1 lịch sử + 1 hiệu lực, `key_id` mới nên token cũ không sống lại), và đo riêng index mới có hiệu lực chứ không ăn theo V27. Lỗi này ở **schema**, không ở Java — mọi test mock repository đều xanh với nó |
| 2026-08-30 | Database `43093c7` | Kiểm phủ bằng cách gỡ cổng rồi đếm test đỏ: gỡ hai lớp chống tham số ma trận → **5/9** đỏ; gỡ lọc ký tự vô hình trong tên thiết bị → **13/28** đỏ; gỡ chuyển vai sang `app_token` → **4/4** đỏ; gỡ cổng vai ở bước dựng yêu cầu ký → **2/7** đỏ; gỡ tập đóng `intent.type` → **1/7** đỏ |
| 2026-08-30 | SDK `fdd28f5` (nhánh, chưa gộp) | `KeyRole` + `keyRoleFromClaim` (fail-safe: thiếu/rỗng/lạ → `viewer`, không bao giờ `owner`) + `AppTokenVerifier` qua JWKS + `DeviceModule`. `npm test` **51/51**, `tsc --noEmit` 0, build ra đủ CJS+ESM+`.d.ts`. `npm run lint` **không chạy được** — `eslint` thiếu trong `devDependencies`, lỗi có sẵn trên `main`. Phiên bản 0.3.0 → 0.4.0. PR #18 |
| 2026-08-30 | SDK — thứ tự gộp | PR #18 tách từ `main` nên **chưa có** bản vá của PR #17; hai nhánh đụng `package.json`, `src/types.ts`, `src/verifier.ts`, `README.md`, và số phiên bản xung đột chắc chắn (#17: 0.3.0→0.3.1; #18: 0.3.0→0.4.0). Gộp #17 trước |

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._

---

## 9. Chi tiết từng module và bảng đặc-tả toán

> Phần này giữ lại từ bản trước để không mất chi tiết theo module. Số đo tổng ở §0–§5 mới hơn; chỗ nào lệch thì §0–§5 đúng.

### Anchorme

**Test bắt-buộc:** module danh-tính (`taad_logic` + `state_nft_logic` + `attack_tests`) phủ GenesisPerson/GenesisChild/can_own/Rotate/Cancel/Finalize/Deactivate + regression Bug#3. Số test toàn-repo Validator = 173/173 PASS (2026-07-08).

**Đã build:** validator `taad` Design-2 (genesis Người/con, rotate, transfer 2-of-2, deactivate, CanOwn); resolver W3C backend; PersonDID register (metadata-6789).

**CID-1 — đo lại 2026-08-12, ĐÃ ĐÓNG (hạ mức từ 🟡 xuống 🟢).** Bản 2026-08-10 ghi *"same-entity đã đóng, cross-entity còn hở tới khi nối PA5-a"*. **Kết-luận đó đã bị bác bởi `PhoenixKey-Validator` PR #73 (MERGED 2026-08-12T00:13:44Z):** cross-entity KHÔNG cần một cổng riêng để đóng, vì PoP-bind (đã land từ trước) cam-kết CẢ `entity_type` LẪN `controller_pkh` vào cùng tiền-ảnh `did` ngay lúc mint (`pop_bind.inner_hash`) — tên anchor `N(did)` do đó đã cam-kết cả hai trường trước khi tồn-tại để mà chi, không riêng `controller_pkh`.

Trạng-thái đo được trên `main` của kho Validator:

| Việc | Trạng-thái | Bằng chứng |
|---|---|---|
| **PC** (at-most-one anchor mỗi tên) | ✅ đã land **và đã nối** | `validators/taad.ak` handler `mint` — `own_policy` do ledger cấp, rồi AND `genesis_uniqueness_ok` (phủ cả Person lẫn Child) |
| **PoP-bind** (the-rightful-one — did tính lại được từ `controller_pkh` **và** `entity_type`) | ✅ đã land **và đã nối** | `lib/phoenixkey/pop_bind.inner_hash` (`enc_type` vào tiền-ảnh), gọi từ `state_nft_logic.validate_mint` nhánh `GenesisPerson` / `GenesisChild` |
| Cổng địa-chỉ ref-input (`find_anchor_datum` ép đúng `Script(taad)`) | ✅ đã land **và đã nối** | `lib/phoenixkey/auth_logic.ak`, PR #74 MERGED 2026-08-12T00:38:19Z |
| **PA5-a** (entity-gate ở tầng spend) | ⛔ **chốt KHÔNG nối — dư thừa, không phải thiếu-sót** | PR #73 MERGED 2026-08-12T00:13:44Z: `entity_type` đã cam-kết ở tầng mint, đọc lại lúc chi chỉ kiểm-lại thứ apply-param đã ép sẵn. `auth_logic.anchor_controller_ok_entities` giữ deprecated làm test đối-kháng, 0 call-site thật |

⟹ **Same-entity VÀ cross-entity collision ĐÃ ĐÓNG cùng một cơ-chế** (PoP-bind), không phải hai cơ-chế riêng. Không còn việc "nối" nào treo cho CID-1. Chi-tiết đầy-đủ + rủi-ro-còn-lại (thiếu bộ test giả-mạo chuyên-trách, không phải một đường tấn-công): `PhoenixKey-Anchorme-Math.md` §8 (T-3) / §9 (CID-1).

**Blocker mở (không còn CID-1):** B2 resolve-by-hash + point-in-time V16 (backend). B3 DeviceDID `Op_create_device` (on-chain) + hw_cert endpoint (backend). B4 Full_Authority `⊑` + type-code canonical (Math v4.7). **B5 duy-nhất-người ở mức NGƯỜI — chưa có gì cưỡng-chế** (mới ghi 2026-08-13, xem dưới).

**🔴 B5 — một người vẫn tạo được nhiều PersonDID (Anchorme-Exec R8).** CID-1 đóng nghĩa là *không đúc được anchor thứ hai cho CÙNG một did-string*; nó KHÔNG có nghĩa *một người chỉ tạo được một did-string*. Một người dùng N thiết-bị vẫn đúc được N PersonDID đều hợp-lệ, không cơ-chế nào on-chain chặn. Khoá sinh trong Secure Enclave chỉ gác **mỗi thiết-bị một danh-tính** (mẫu sinh-trắc không rời máy, không có bảng đối-chiếu chéo) — **trước 2026-08-13 bộ spec ghi nhầm rằng sinh-trắc đã đủ chống trùng người, nay đã gỡ ở 8 chỗ**. Ràng-buộc một-người-một-danh-tính v1 thiết-kế nằm ở **Knowme** (neo giấy-tờ tuỳ-thân) — trạng-thái code xem mục Knowme dưới; các lớp person-level còn lại + cổng chặn production xem `PhoenixKey-Wakeme-Exec.md` §7.

**Bug live đã biết:** một đường tra khoá công khai trả 500 với một nhóm người dùng nhất định; đội backend đang xử. Chi tiết đường dẫn và điều kiện kích hoạt giữ ở kênh nội bộ — nêu ở tài liệu công khai là chỉ luôn cho người ngoài một cách dò trạng thái tài khoản.

**Byte-9 `Character`→`Avatar` — CHỐT 2026-07-10** (xem `PhoenixKey-Math.md` §21): ranh giới Asset/Avatar dựa **nơi-ra-quyết-định** (locus-of-control, không dùng "agency" — dễ lộn AgentDID byte-6): Avatar = chỉ hành động khi nhận lệnh trực tiếp từ controller ngoài; Asset = không nhận lệnh, chỉ transfer/consume. Avatar chỉ do PersonDID/OrgDID vận hành (I-CHAR-1 sửa `{Person,Service}`→`{Person,Org}` — CanOwn §22.1 + `can_own()` on-chain vốn đã đúng, I-CHAR-1 là bên sai, đã vá). "Sống→chết" = burn AvatarDID + mint N AssetDID với `derived_from` nối phả hệ (không phải type-transition tại chỗ). Đã sửa xong Math.md (10 chỗ) + Anchorme-Math/Tech/Exec.md + DIDMethod-W3C.md, push nhánh `claude/spec-northstar-2026-07-10`. **Việc tồn đọng riêng (chưa quyết, không nằm trong đợt này):** tách owner/operator cho sinh vật hoang dã không ai đứng tên; uỷ quyền Service ký hộ Org khi mint Avatar hàng loạt (đẩy sang Tech.md).

**Câu hỏi thiết-kế MỞ — Byte-4 `Asset` chỉ physical** (còn treo, KHÔNG còn phụ thuộc byte-9 nữa vì byte-9 đã chốt độc lập): lỗ hổng phân-loại — tài-sản-số thụ-động (file/dataset/media/NFT/VC-schema/model-weights) rơi khe (≠Asset physical, ≠Bot/Agent tự-chủ, ≠Service sản-phẩm, ≠Avatar). Chọn: (a) nới định nghĩa Asset → physical HOẶC digital (thêm `asset_domain: Physical|Digital`, `physical_id`/`location_proof` chuyển Optional — đề xuất 2026-07-10, chưa chốt câu chữ cuối); hay (b) digital = VC/metadata dưới DID khác (out-of-scope, ranh giới hẹp). Byte-value bất biến → hash-safe dù chọn hướng nào; lan tới Math §17 + Aiken `types.ak` + Java `DidPhoenixGenerator`. `AI`→`Agent` (byte-6) đã chốt đổi (issue đội backend).

### Rebirthme

**Nền đã chạy (173/173 Aiken PASS, 2026-07-08):** ví theo-DID `did_payment` (chi khi Active + controller ký; tài-sản sống qua rotate; địa chỉ bất-biến); đóng-băng theo trạng-thái (Recovering/Migrated/Revoked chặn chi); singleton-anchor I-WALLET-4/5; guardian recovery Init/Cancel/Finalize/UpdateGuardians(≤5) + timelock 3600 slot + collateral 50 ADA (bỏ Shamir); ví Standard + Rotation Account; P-256 low-s (I-SIGN-LOWS); `lampnet.rs` fail-closed (I-VAULT-4); Ed25519 dalek deterministic.

**Chưa có code:** 🔴 `did_subaddr.ak` (L3 unlinkable, chờ chốt [DEP-2]); 🔴 `did_stake.ak` (stake theo-DID). 🟡 I-CURVE-5 chưa enforce builder; kho bí-mật/phả-hệ seed chưa hợp-nhất; export re-key UI chưa cắm mặc-định; guardian nâng-cao (trọng-số/veto/cap) Todo. 🔴 **Đo 2026-08-30 — đường khôi-phục DÙNG ĐƯỢC hôm nay chỉ còn MỘT: guardian-threshold.** Spec trước đếm "VC-Glint" ngang hàng guardian, nhưng kênh đó dựa **Glint P-thr** (quorum k-of-n ẩn danh) vốn còn `[CONSTRUCTION-PENDING]` (`Glint-Math.md:107`, `:199`), và verifier **BẮT BUỘC reject** `circuit_id` treo (`:196`) ⇒ không có cách nào dùng hợp-lệ hôm nay. Midnight ghi "khi tích-hợp" — chưa xong. Đã sửa I-WALLET-7/8 để thôi đếm thừa; **KHÔNG hạ điều-kiện chặn export**. ⚪ legacy-migration, on-ramp mandate, pool-ops (KES/VRF) build-ready-Todo.

**CIP-30 connector — nay CÓ CODE, chưa lên sản-xuất (2026-08-15).** Kho `PhoenixKeyDID/Wallet` dựng xong lớp dApp-connector: kết nối ví mở rộng (Lace/Eternl/Typhon/…), xem-bằng-khoá-công-khai, gửi ADA/token, staking, uỷ-quyền dRep, gửi Governance action — 178/178 test PASS, `tsc --noEmit` sạch, 4 ngôn ngữ ngang khoá. Kho `PhoenixKey-Frontend` gắn kho này bằng **submodule** và mở hai đường `/wallet` + `/night` (`bun run build` xanh, 16 route). **Cả hai còn ở PR chưa merge** (Wallet #18, Frontend #19) ⇒ phoenixkey.me **chưa** có đường nào chạm tới ví. Đây là ví **ngoài** (khoá nằm ở tiện-ích mở rộng của người dùng) — KHÔNG phải ví Phoenix theo-DID, và **không** đụng tới blocker khoá-thiết-bị bên dưới.

**Blocker ngoài:** CARP policy-id, stake-state indexer (backend), Merkle LAMP (LAMP), schema anchor mới vào TAADDatum (backend, chờ duyệt), crate KES/VRF (PoC).

**Lộ-trình:** M1 (resolver L1/L2 + wallet API v2 + blob-đơn) → M2 (`limit_meter.ak` + I-CURVE-5) → M3 (`did_stake.ak` + export re-key UI) → M4 (phả-hệ seed + Strata) → M5 (`did_subaddr.ak` + registry-lib mode-2 + pool KES).

**Deprecate (bỏ dùng):** endpoint `/wallet/magic/claim` (MAGIC claim custodial) — sai model, MAGIC là account-in-vault chứ không native trong ví; app phải lấy MAGIC từ vault, không qua endpoint ví.

### Wakeme

**Validator:** `activation_vault.ak`+`activation_logic.ak` — 5 spend redeemer (GenDrip/Reclaim/VestToOwner/ClaimVested/ForfeitPhase2) + 2 mint-gate, datum 9-field, đồng-hồ NGÀY+EPOCH, vest-gated-per-epoch + forfeit-1001-idle-epoch, chống-double-satisfaction; `plutus.json` khớp code; 69 test riêng `activation_logic` PASS; qua red-team nội bộ. Còn: apply-param builder, sửa comment sai đầu file. PR chờ đội on-chain duyệt.

**Backend/Core:** GetLAMP orchestration, anti-idle PHA-1, vest/forfeit PHA-2, ClaimVested, GetMAGIC — chờ backend + Core Enclave. Chưa có evidence `curl`.

**Blocker:** B1 engine Gen đọc-số-dư (MAGIC/CARP-team). B2 Registry dịch-vụ-tiêu-tài-nguyên (`has_counterparty_consume` placeholder). B3 GetLAMP-PersonDID chờ PA2. B4 GreenBack settlement + fee_refill phản-chu-kỳ.

**AbandonPhase1:** không có redeemer on-chain trong thiết-kế hiện tại — thoát-sớm PHA-1 qua anti-idle tự thu-hồi (không có nút chủ-động). **Rủi ro theo dõi:** pot cạn khi nhiều user cùng PHA-2 (R1); wash-rỗng nếu Registry lỏng (R2).

### Feecover

**Spec:** MERGED (#14) — 4 doc chuẩn-hoá. **Code:** ConsumeMAGIC lõi (C-CM-1..5) Done (kế thừa đội ConsumeMAGIC, không chứng-minh Feecover đúng). Layer Feecover (`ServiceFeeSchedule`/`FeecoverGate`/`FeecoverAccrual`/`FeecoverEpochSettle`/quy-đổi-CARP) — 0 dòng, chưa test. `EngageDatum.did_commit` field có, immutable-enforced, MVP nội-dung sentinel rỗng.

**Blocker:** B1 MAGIC-model, B2 CARP policy-id, B3 enforce nội-dung `did_commit` per-DID. **Hở nội-tại nặng nhất:** FG-4 — EpochSettle pseudo-code, 0 validator, dựa provider trung-thực (KHÔNG blocker đội khác — Feecover tự vá). Phụ-thuộc mềm: Resolve API point-in-time (backend).

**Giá 2 thao-tác DID — CHỐT 2026-08-10** (khép một phần D1/D3, `PhoenixKey-Feecover-Math.md §7.2bis`): `did.rotate` = 2 MAGIC **giá cố-định**, `did.transfer` = 10 MAGIC (nhân theo cầu bình-thường), `UpdateGuardians` **miễn phí**. Tỉ-lệ 1:5 giữ nguyên từ bảng ADA cũ (§36) để việc bỏ đường ADA không đồng-thời là một đợt đổi giá ngầm. Hai bất-biến mới: FEECOVER-PRICE-1 (rotate KHÔNG nhân theo cầu — `demand_mult` là một số dùng chung toàn hệ, nên tải của module khác sẽ định giá thao-tác an-ninh của DID, và một đợt lộ khoá hàng loạt tự đẩy giá lên đúng lúc cần rẻ nhất); FEECOVER-PRICE-2 (thiếu MAGIC KHÔNG được chặn Rotate — nếu chặn thì kẻ đã lộ khoá nạn-nhân chỉ cần làm cạn MAGIC của họ là khoá lộ không bao giờ bị xoay).

**Blocker MỚI (chưa từng ghi):** B4 — ConsumeMAGIC **chưa có lớp giá cố-định**; công-thức áp `demand_mult` cho mọi `op_type`, không loại-trừ. Đây là **điều-kiện tiên-quyết** để nối `did.rotate` (đã gửi yêu-cầu sửa hợp-đồng sang MAGIC 2026-08-10). `did.transfer` KHÔNG phụ-thuộc B4, đi trước được. B5 — `op_type` 7/8 chưa được MAGIC cấp; bảng `op_prices` sắp tăng ngặt, trần 16 dòng, hiện dùng 6.

**Điều-kiện wiring (W1-W4):** W1 nối đường MAGIC **cùng đợt** với gỡ đường ADA, không song-song (hai cơ-chế phí cùng gắn một redeemer ⟹ thu kép hoặc bên-nào-rẻ-hơn-thắng tuỳ builder off-chain). W2 committee `PriceParam` phải > 1-of-N trước mạng chính — ngưỡng 1 cho phép một khoá chi UTxO beacon ngay trước tx nạn-nhân ⟹ từ-chối xoay khoá nhắm đúng một người, lặp vô hạn.

**Lộ-trình:** P0 (chốt D1-D6) → P1 (B1/B2/B3/**B4/B5**) → P2 (build + vá FG-4) → P3 (test) → P4 (per-DID) → P5 (production).

### Protectme

Cổng chi-trả `protectme_logic.ak`+`protectme_beacon_logic.ak`+`protectme_payout.ak` (branch `feat/protectme-payout`) — khối duy nhất có code+test đối-kháng sạch (double-satisfaction, cred-collision, ADA-skim, miền-số, cross-bucket đều chặn): **72 test (23 beacon + 14 logic + 35 payout)** (`aiken check` 2026-08-19). Beacon one-shot per claim_id ĐÃ NỐI: `protectme_beacon_logic.ak` (733 dòng) mint-gate được `protectme_payout.ak` gọi trong cùng validator (`:49-53`, own_policy == own_cred) — đóng double-satisfaction TRONG-CÙNG-tx. Phần còn hở: đúc trùng `claim_id` giữa HAI giao-dịch khác nhau — mint-gate không có bộ nhớ liên-giao-dịch nên committee (sai sót off-chain) ký MintClaim hai lần cho cùng claim_id thật vẫn tạo hai escrow độc-lập; mỗi lần trừ đúng `amount` thật từ pot (không tạo giá-trị từ hư-không — solvency giữ), nhưng claimant được trả kép từ pot cộng-đồng (`protectme_beacon_logic.ak:36-45`). Đóng nốt cần uniqueness liên-giao-dịch (Treasury phiếu-duyệt one-shot / SMT claim_id). 2-bucket Treasury + Feecover premium wiring + resolver claim + UI — chưa code (backend/UI). 11 quyết-định PROT-1..11 chờ chốt (🔴 PROT-10 evidence-bar, PROT-11 cohort, PROT-4 ngưỡng SYS/USER). Blocker hạ-tầng: MAGIC-model, CARP policy-id. **NO-GO tới khi uniqueness liên-tx + tất cả quyết-định chốt.**

### Knowme

**Code (verify 2026-07-09):** Mức 1 (tự-khai) + Mức 2 (xuất-trình chọn-lọc) có code+test, demo `/vc`. Evidence: `npx vitest run src/lib/sdvc/` → **20 file / 415 test PASS** (~1.2s). Con-số "135" cũ trong `SD-VC-ALGORITHM-v1.md` là snapshot lỗi-thời. Lớp tài-liệu: nền có code+test (`dossier.ts`/`fingerprint.ts`/`eciesSeal`); tiết-lộ-chọn-lọc-tài-liệu + versioning Strata + re-seal = chưa code. Mức 3 ZK (BBS+): chưa code. Query gateway (VeData): chưa code.

**Blocker:** B1 lib BBS+prover (Mức 3), B2 LampNet gateway (lớp tài-liệu), B3 Glint/Spectra (VeData), B4 StampRecord Strata. **Mốc:** M1 (Mức1+2+`/vc`) chạy; M2-M7 chờ blocker.

**🔴 B5 duy-nhất-người v1 — mã đã có, nhưng KHÔNG cưỡng-chế (đo lại 2026-09-02).** Spec giao Knowme giữ bất-biến "một giấy-tờ tuỳ-thân ⇒ nhiều nhất một PersonDID" (`PhoenixKey-Knowme-Math.md` Đ-7, I-KNOW-12..16).

Bản trước ghi `fingerprint.ts`/`dossier.ts`/`UniquenessRegistry` **không tồn tại trên `origin/main`** của `PhoenixKey-Frontend`. **Dữ-kiện đó sai từ khi nhánh `cccd-uniqueness-v1` được gộp** — `git ls-tree -r --name-only origin/main | grep -E 'fingerprint|dossier'` nay trả về `src/lib/sdvc/fingerprint.ts`, `src/lib/sdvc/dossier.ts`, `src/lib/sdvc/__tests__/fingerprint.test.ts`.

**Kết-luận thì vẫn đứng, vì lý-do khác.** `UniquenessRegistry` (`fingerprint.ts:106`) và `AnchorRegistry` (`anchor.ts:500`) đều là `Map` **trong bộ nhớ tiến-trình**, chỉ export ở `index.ts:65,84`, và **0 nơi dùng ngoài `__tests__`** (`grep -rn UniquenessRegistry src/ --include='*.ts' | grep -v __tests__` → chỉ 2 dòng: chỗ định-nghĩa và chỗ export). Chúng là bản tham-chiếu minh-hoạ, đúng như `PhoenixKey-Knowme-Math.md:204` tự gắn nhãn "(minh-hoạ)".

Phía backend thì ngược lại — mã cưỡng-chế THẬT có và đúng chiều: `PhoenixKey-Database` `FingerprintRegistryService.java:282-307` trả `CONFLICT_OTHER_OWNER` khi một giấy-tờ đã thuộc DID khác, `V35__document_fingerprints.sql:40` đặt `fp VARCHAR(44) PRIMARY KEY`, và pepper hỏng-đóng ngoài profile dev (`:127-168` ném `IllegalStateException` khi thiếu `PHOENIXKEY_KNOWME_PEPPER_V1`). **Nhưng nó là ốc-đảo**: `FingerprintRegistryService` chỉ có MỘT nơi dùng trong `src/main/java` — `KnowmeFingerprintController.java:41`, tức chính endpoint phơi ra nó. `/identity/register` không gọi sang, và `register(auth.userDid(), fpHex)` nhận DID **đã tồn tại** làm tham-số ⇒ nó ghi-nhận SAU, không gác TRƯỚC. Bỏ qua bước đó thì vẫn có DID.

⟹ **hôm nay không có gì cưỡng-chế duy-nhất-người**, kể cả ở tầng ứng-dụng. Đây là mặt còn lại của Anchorme B5. Việc gỡ rẻ nhất trong cả nhóm: nối `FingerprintRegistryService.register` vào đường tạo PersonDID và từ-chối khi `CONFLICT_OTHER_OWNER` — mã từ-chối, pepper hỏng-đóng và khoá chính đều đã sẵn, chỉ thiếu dây nối.

### Easteregg

**Spec:** 4 doc hợp nhất mô hình "mức riêng-tư của ví Phoenix" (không phải ví thứ ba), chốt 2026-07-09. **Code on-chain:** `did_pool.ak` (T1 MST) + `did_subaddr.ak` (T0/L3) — chưa tồn tại. **Off-chain:** Indexer/Accountant, sweep crank, withdraw builder — chưa có. **ZK T2:** verifier Aiken chưa viết; ExUnit 2.842B là đo của Easteregg-ZK bên VeData (độc lập); ceremony chưa chạy. **Test:** 0 test Easteregg. **PoC:** 1 PoC Python trên Preview (3 tx-hash) minh-hoạ ẩn-số-dư + gated-proof, KHÔNG validator, chưa chứng-minh operator-không-rút. **Gap:** G1 (fee-split), G3 (sweep per-pair), G5 (salt-recovery) 🔴 chưa vá; G2/G4 🟡. **NO-GO toàn module**; chỉ GO build+test Preview T1 + T3-mode-1.

### Smartsend

**Vị-trí:** module độc-lập thứ 8 (chốt 2026-07-09), tách từ Rebirthme, dùng chung hạ-tầng ví/guardian/anti-drain. **Build:** `smartsend_escrow.ak` — spec đầy-đủ (SS-1..12 + SSR-4 hợp-nhất), CHƯA code, 0 test.

**Bất-biến đã hợp-nhất (không còn "vá đỏ" treo):** SS-1/SS-5′/SS-12 (value-conservation byte-perfect, `min_ada` tách field, `fee_covered` chỉ audit); SS-7′ (escrow-1-lần, chống double-satisfaction batch); SS-9′ (Accept verify controller-sig qua anchor); SS-11 (`reclaim_deadline`+`ReclaimTimeout`); SS-8/SS-8′ (Freeze trong cửa-sổ-veto; thoát qua guardian-quorum hoặc `freeze_deadline` auto-hoàn); SS-10 (`window ≥ min_window_floor`); SS-2 (veto-race biên); SS-3/SSR-4/SSR-13 (factor Cancel neo anchor-enroll).

**Phụ-thuộc-chặn ngoài:** `limit_meter_vault` (Rebirthme — nay đã build được, xem §0); nền `did_payment`+guardian (173/173 PASS); verifier Glint (VeData) + Spectra (**LampNet** — `Glint-Math.md:21`), Phase 2 — bind `blake2b_256(own_ref ‖ escrow_datum_hash)` SSR-12); guardian ResolveFreeze quorum (chưa build); enroll-set factor trong TAADDatum (Core Anchorme/Validator).

**CẦN CHỐT:** `reclaim_deadline` tương-đối `veto_deadline`; `window` mặc-định + `min_window_floor`; `freeze_deadline`; thứ-tự land vs anti-drain; ưu-tiên Glint sớm hay guardian-factor đủ bản đầu.

### Math (đặc-tả tổng — `PhoenixKey-Math.md`)

Hiện-trạng triển-khai các phần của đặc-tả toán v4.6 (đã tách khỏi Math.md):

| Area | Spec | Hiện trạng |
|---|---|---|
| Crypto primitives (HKDF, Ed25519, BLAKE2b, P-256 verify, CIP-1852) | §1, §6, §8 | Implemented (`rust_core`) |
| DID Document publish (metadata label 6789) | §2 | Implemented — live preprod + preview (PhoenixKey-PoC) |
| TAAD UTxO state machine + Rotate redeemer | §10 | Validator compiles; tx-builder trên feature branch, chưa merge |
| Tiered recovery (Tier 1–5) | §11 (module Rebirthme) | Spec-only — chưa có recovery code path |
| §36 fee architecture (30/70 split, Phoenix Treasury) | §36 | Spec-only — enforcement (fee-receipt minting policy + ExUnits benchmark) chờ Validator Issue #7. Ước tính mem ~150–400, CPU ~80K–200K (+3–12% baseline ~0.17 ADA) |

---
