# PhoenixKey — STATUS (hiện trạng & tiến độ)

> **File này là báo-cáo hiện-trạng, KHÔNG phải đặc-tả.** Bộ spec (`*-Vi-Feat/-Math/-Tech/-Exec`) là **kim-chỉ-nam thiết-kế** — mô tả hệ thống ĐÍCH mà các đội dev xây tới. File này ghi *đang ở đâu trên đường tới đó*: cái gì đã chạy, chặn bởi ai, bằng chứng test. Khi hai bên lệch → spec là mục-tiêu, STATUS là thực-tại.
>
> Cập nhật: 2026-08-15. Nguồn: audit per-module + đối chiếu code/CI thật; mục 5 đo lại đầu-cuối bằng `aiken check`/`aiken build` và gọi thật máy chủ đang chạy. Đợt 15/08 bổ sung: đọc chip giấy tờ (eMRTD), đầu dò tầm với, CORS tra-tên-công-khai, SDK username — **cả bốn đều là PR chưa merge**, xem mục 4. Riêng eMRTD đã **nối đầu-cuối** trong cùng ngày (PR Core #73, 2 commit thêm) — chỗ đứt còn lại là chưa chạm thẻ thật.

---

## 1. Bảng trạng-thái 8 module

| Module | Nền đã chạy được | Blocker chính | Production |
|---|---|---|---|
| **Anchorme** | validator `taad` Design-2 + **PC** (uniqueness anchor, đã nối) + **PoP-bind** (did tự chứng, đã nối) — CID-1 ĐÃ ĐÓNG cả same- và cross-entity; resolver W3C; register metadata-6789 | DeviceDID; resolve-by-hash | GO custody cho lỗ CID-1 (đóng 2026-08-12); NO-GO tổng-thể vẫn treo ở DeviceDID + resolve-by-hash |
| **Rebirthme** | ví theo-DID `did_payment`, đóng-băng theo trạng-thái, guardian recovery ngưỡng+timelock, P-256 low-s, `lampnet.rs`; **`limit_meter_vault` + `did_stake` nay build được** (hash `f3be6d6d…` / `eb535cc1…`) | `did_subaddr` chưa có; **khoá thiết bị (yếu-tố-2 chi tiêu) chưa tồn tại trong mã app** | NO-GO ví-giá-trị-lớn tới khi khoá thiết bị land |
| **Wakeme** | validator `wakeme_vault` build được (hash `8655974a…`); backend `buildGetLamp`/`submitGetLamp` là hiện thực thật | B1 engine Gen đọc-số-dư, B2 Registry consume-gate, B3 PA2 cho GetLAMP-PersonDID; **3 biến môi trường `ACTIVATION_*` rỗng ⇒ mọi lời gọi trả `501`** | NO-GO tới khi Registry + PA2 land |
| **Feecover** | ConsumeMAGIC lõi (kế thừa) | Layer Feecover 0 dòng; B1 MAGIC-model, B2 CARP policy-id, B3 did_commit per-DID; FG-4 EpochSettle tự-vá | NO-GO tới khi B1+B2+B3 + FG-4 |
| **Protectme** | cổng chi-trả `protectme_logic`+`protectme_payout` (39 test đối-kháng sạch, branch `feat/protectme-payout`) | 🔴 `protectme_beacon.ak` one-shot 0 dòng (chặn-merge); 2-bucket + resolver + UI chưa có; 11 quyết-định PROT-1..11 | NO-GO tới khi beacon + blocker + quyết-định |
| **Knowme** | Mức 1+2 SD-VC có code+test, demo `/vc` (20 file / 415 test PASS) | B1 lib BBS (Mức 3), B2 LampNet gateway (lớp tài-liệu), B4 StampRecord | M1 chạy; Mức 3/tài-liệu chờ blocker |
| **Easteregg** | 1 PoC Python off-chain trên Preview (3 tx-hash); **`did_pool` nay build được** (hash `9ba97ba4…`) | `did_subaddr.ak` chưa có; ZK Tầng 2 verifier chưa viết; G1/G3/G5 chưa vá | NO-GO; chỉ GO build+test Preview T1 + T3-mode-1 |
| **Smartsend** | spec đầy đủ SS-1..12; **validator `smartsend` nay build được** (hash `9ed1b56f…`) | verifier Glint/Spectra (Phase 2); SS-11 vừa khôi phục điều kiện `amount ≥ large_threshold` (PR #23) | NO-GO; money-critical, review trước code |

---

## 2. Ma-trận blocker xuyên-module (ai gỡ)

| Blocker | Chặn module | Đội gỡ | Ghi chú |
|---|---|---|---|
| **B1 — MAGIC engine đọc-số-dư** | Wakeme, Feecover, Protectme | MAGIC team | Model chốt = account-in-vault (không native); còn spell-out engine reference-input, không spend/đốt LAMP |
| **B2 — CARP policy-id/asset-name/decimals** | Feecover, Protectme, Rebirthme, Wakeme | CARP team | preprod + mainnet; test đang dùng hằng giả |
| **B3 — Registry consume-gate + `did_commit` per-DID** | Wakeme, Feecover | MAGIC team + backend | `has_counterparty_consume` còn placeholder; `did_commit` field có, nội-dung sentinel rỗng |
| ~~PA2 — UniquenessThread hai-validator~~ **LOẠI VĨNH VIỄN**, thay bằng **PC** — ✅ **đã land + đã nối** 2026-08-10 | — | — | PA2-hai-validator vô-nghiệm fixed-point hash (mỗi script bake hash của script kia). PC gộp một multi-purpose validator, `anchor_policy ≡ thread_policy ≡ own_policy`. K=32 (không phải 256 — bootstrap nguyên-tử K=256 ≈ 34 KB, vượt trần tx 16 KB). Đặc-tả §5.7ter |
| ~~PA5-a — entity-gate: viết xong nhưng chưa nối~~ **CHỐT KHÔNG NỐI** — ✅ **gỡ 2026-08-12** | — | — | PoP-bind (đã land + nối) cam-kết `entity_type` vào tiền-ảnh `did` ngay lúc mint (`pop_bind.ak:109`); một cổng đọc lại `entity_type` lúc chi chỉ kiểm lại thứ tên-anchor apply-param đã ép sẵn — không đóng thêm bề-mặt nào, đổi lại 4-5 script-hash không cần thiết (`PhoenixKey-Validator` PR #73). Hàm `anchor_controller_ok_entities` giữ deprecated làm test đối-kháng |
| ~~`limit_meter.ak` — anti-drain~~ **GỠ 2026-08-08** | — | — | `lib/phoenixkey/limit_meter.ak` + validator `limit_meter_vault` (hash `f3be6d6d…`) build được trên `main`; nằm trong 577/577 test PASS |
| 🔴 **Khoá thiết bị — yếu-tố-2 khi chi từ ví Phoenix** | Rebirthme (ví Phoenix), Anchorme (datum genesis) | app + on-chain | Aiken đã ép 2-of-2 (`auth_logic.ak:37-58`), nhưng phía app **chưa tồn tại**: grep `device_pkh`/`deviceKey` trong `rust_core/src` và `lib/` = 0. Thiếu cả sinh khoá, lưu trữ, API ký, và đưa `device_pkh` vào datum. Phải Ed25519/secp256k1 — P-256 không verify được on-chain |
| 🔴 **CBOR `did_payment` đang dùng là bản CŨ 1-chữ-ký** | Rebirthme, Anchorme | on-chain + app + backend | CBOR trong `rust_core/assets/` và vector test backend khớp byte-for-byte bản build 25-06, **trước** khi thêm 2-of-2 ⇒ mọi địa chỉ Phoenix đã dẫn ứng với validator chỉ cần 1 chữ ký seed. Phải re-pin cùng lượt: asset CBOR + 3 vector `AikenPhoenixCustodyDeriverTest` + env `DID_PAYMENT_CBOR_HEX` |
| **DeviceDID `Op_create_device`** | LampNet node, Knowme device, Rada | on-chain + backend | + hw_cert verify endpoint |
| **FG-4 — Feecover EpochSettle validator** | Feecover | đội Feecover | Pseudo-code, 0 validator, dựa provider trung-thực — tự vá khi build |
| **B4 — Math `⊑` + type-code canonical** | delegation PersonDID, author-DID phi-nhân | Math/maintainer | Chốt vào Math v4.7; bảng-byte code làm canonical |

---

## 3. Chi tiết từng module

## Anchorme

**Test bắt-buộc:** module danh-tính (`taad_logic` + `state_nft_logic` + `attack_tests`) phủ GenesisPerson/GenesisChild/can_own/Rotate/Cancel/Finalize/Deactivate + regression Bug#3. Số test toàn-repo Validator = 173/173 PASS (2026-07-08).

**Đã build:** validator `taad` Design-2 (genesis Người/con, rotate, transfer 2-of-2, deactivate, CanOwn); resolver W3C backend; PersonDID register (metadata-6789).

**CID-1 — đo lại 2026-08-12, ĐÃ ĐÓNG (hạ mức từ 🟡 xuống 🟢).** Bản 2026-08-10 ghi *"same-entity đã đóng, cross-entity còn hở tới khi nối PA5-a"*. **Kết-luận đó đã bị bác bởi `PhoenixKey-Validator` PR #73 (MERGED 2026-08-12T00:13:44Z):** cross-entity KHÔNG cần một cổng riêng để đóng, vì PoP-bind (đã land từ trước) cam-kết CẢ `entity_type` LẪN `controller_pkh` vào cùng tiền-ảnh `did` ngay lúc mint (`pop_bind.ak:109`) — tên anchor `N(did)` do đó đã cam-kết cả hai trường trước khi tồn-tại để mà chi, không riêng `controller_pkh`.

Trạng-thái đo được trên `main` của kho Validator:

| Việc | Trạng-thái | Bằng chứng |
|---|---|---|
| **PC** (at-most-one anchor mỗi tên) | ✅ đã land **và đã nối** | `validators/taad.ak` handler `mint` — `own_policy` do ledger cấp, rồi AND `genesis_uniqueness_ok` (phủ cả Person lẫn Child) |
| **PoP-bind** (the-rightful-one — did tính lại được từ `controller_pkh` **và** `entity_type`) | ✅ đã land **và đã nối** | `lib/phoenixkey/pop_bind.ak:109` (`enc_type` vào tiền-ảnh), gọi từ `state_nft_logic.ak:166` (Person) / `:192` (Child) |
| Cổng địa-chỉ ref-input (`find_anchor_datum` ép đúng `Script(taad)`) | ✅ đã land **và đã nối** | `lib/phoenixkey/auth_logic.ak`, PR #74 MERGED 2026-08-12T00:38:19Z |
| **PA5-a** (entity-gate ở tầng spend) | ⛔ **chốt KHÔNG nối — dư thừa, không phải thiếu-sót** | PR #73 MERGED 2026-08-12T00:13:44Z: `entity_type` đã cam-kết ở tầng mint, đọc lại lúc chi chỉ kiểm-lại thứ apply-param đã ép sẵn. `auth_logic.anchor_controller_ok_entities` giữ deprecated làm test đối-kháng, 0 call-site thật |

⟹ **Same-entity VÀ cross-entity collision ĐÃ ĐÓNG cùng một cơ-chế** (PoP-bind), không phải hai cơ-chế riêng. Không còn việc "nối" nào treo cho CID-1. Chi-tiết đầy-đủ + rủi-ro-còn-lại (thiếu bộ test giả-mạo chuyên-trách, không phải một đường tấn-công): `PhoenixKey-Anchorme-Math.md` §8 (T-3) / §9 (CID-1).

**Blocker mở (không còn CID-1):** B2 resolve-by-hash + point-in-time V16 (backend). B3 DeviceDID `Op_create_device` (on-chain) + hw_cert endpoint (backend). B4 Full_Authority `⊑` + type-code canonical (Math v4.7).

**Bug live đã biết:** GET `/identity/{did}/pubkey` trả 500 với user đã qua recovery (consumer: backend bên thứ 3 OriLife/AladinWork) — cần Long vá.

**Byte-9 `Character`→`Avatar` — CHỐT 2026-07-10** (xem `PhoenixKey-Math.md` §21): ranh giới Asset/Avatar dựa **nơi-ra-quyết-định** (locus-of-control, không dùng "agency" — dễ lộn AgentDID byte-6): Avatar = chỉ hành động khi nhận lệnh trực tiếp từ controller ngoài; Asset = không nhận lệnh, chỉ transfer/consume. Avatar chỉ do PersonDID/OrgDID vận hành (I-CHAR-1 sửa `{Person,Service}`→`{Person,Org}` — CanOwn §22.1 + `can_own()` on-chain vốn đã đúng, I-CHAR-1 là bên sai, đã vá). "Sống→chết" = burn AvatarDID + mint N AssetDID với `derived_from` nối phả hệ (không phải type-transition tại chỗ). Đã sửa xong Math.md (10 chỗ) + Anchorme-Math/Tech/Exec.md + DIDMethod-W3C.md, push nhánh `claude/spec-northstar-2026-07-10`. **Việc tồn đọng riêng (chưa quyết, không nằm trong đợt này):** tách owner/operator cho sinh vật hoang dã không ai đứng tên; uỷ quyền Service ký hộ Org khi mint Avatar hàng loạt (đẩy sang Tech.md).

**Câu hỏi thiết-kế MỞ — Byte-4 `Asset` chỉ physical** (còn treo, KHÔNG còn phụ thuộc byte-9 nữa vì byte-9 đã chốt độc lập): lỗ hổng phân-loại — tài-sản-số thụ-động (file/dataset/media/NFT/VC-schema/model-weights) rơi khe (≠Asset physical, ≠Bot/Agent tự-chủ, ≠Service sản-phẩm, ≠Avatar). Chọn: (a) nới định nghĩa Asset → physical HOẶC digital (thêm `asset_domain: Physical|Digital`, `physical_id`/`location_proof` chuyển Optional — đề xuất 2026-07-10, chưa chốt câu chữ cuối); hay (b) digital = VC/metadata dưới DID khác (out-of-scope, ranh giới hẹp). Byte-value bất biến → hash-safe dù chọn hướng nào; lan tới Math §17 + Aiken `types.ak` + Java `DidPhoenixGenerator`. `AI`→`Agent` (byte-6) đã chốt đổi (issue Long).

## Rebirthme

**Nền đã chạy (173/173 Aiken PASS, 2026-07-08):** ví theo-DID `did_payment` (chi khi Active + controller ký; tài-sản sống qua rotate; địa chỉ bất-biến); đóng-băng theo trạng-thái (Recovering/Migrated/Revoked chặn chi); singleton-anchor I-WALLET-4/5; guardian recovery Init/Cancel/Finalize/UpdateGuardians(≤5) + timelock 3600 slot + collateral 50 ADA (bỏ Shamir); ví Standard + Rotation Account; P-256 low-s (I-SIGN-LOWS); `lampnet.rs` fail-closed (I-VAULT-4); Ed25519 dalek deterministic.

**Chưa có code:** 🔴 `limit_meter.ak` anti-drain — KHÔNG tồn tại (I-CURVE-4 load-bearing, hở HIGH, ưu-tiên M2); 🔴 `did_subaddr.ak` (L3 unlinkable, chờ chốt [DEP-2]); 🔴 `did_stake.ak` (stake theo-DID). 🟡 I-CURVE-5 chưa enforce builder; kho bí-mật/phả-hệ seed chưa hợp-nhất; export re-key UI chưa cắm mặc-định; guardian nâng-cao (trọng-số/veto/cap) Todo; chứng-thực VeData-Glint/Midnight chờ VeData. ⚪ legacy-migration, on-ramp mandate, pool-ops (KES/VRF) build-ready-Todo.

**CIP-30 connector — nay CÓ CODE, chưa lên sản-xuất (2026-08-15).** Kho `PhoenixKeyDID/Wallet` dựng xong lớp dApp-connector: kết nối ví mở rộng (Lace/Eternl/Typhon/…), xem-bằng-khoá-công-khai, gửi ADA/token, staking, uỷ-quyền dRep, gửi Governance action — 178/178 test PASS, `tsc --noEmit` sạch, 4 ngôn ngữ ngang khoá. Kho `PhoenixKey-Frontend` gắn kho này bằng **submodule** và mở hai đường `/wallet` + `/night` (`bun run build` xanh, 16 route). **Cả hai còn ở PR chưa merge** (Wallet #18, Frontend #19) ⇒ phoenixkey.me **chưa** có đường nào chạm tới ví. Đây là ví **ngoài** (khoá nằm ở tiện-ích mở rộng của người dùng) — KHÔNG phải ví Phoenix theo-DID, và **không** đụng tới blocker khoá-thiết-bị bên dưới.

**Blocker ngoài:** CARP policy-id, stake-state indexer (backend), Merkle LAMP (LAMP), schema anchor mới vào TAADDatum (backend, chờ duyệt), crate KES/VRF (PoC).

**Lộ-trình:** M1 (resolver L1/L2 + wallet API v2 + blob-đơn) → M2 (`limit_meter.ak` + I-CURVE-5) → M3 (`did_stake.ak` + export re-key UI) → M4 (phả-hệ seed + Strata) → M5 (`did_subaddr.ak` + registry-lib mode-2 + pool KES).

**Deprecate (bỏ dùng):** endpoint `/wallet/magic/claim` (MAGIC claim custodial) — sai model, MAGIC là account-in-vault chứ không native trong ví; app phải lấy MAGIC từ vault, không qua endpoint ví.

## Wakeme

**Validator:** `activation_vault.ak`+`activation_logic.ak` — 5 spend redeemer (GenDrip/Reclaim/VestToOwner/ClaimVested/ForfeitPhase2) + 2 mint-gate, datum 9-field, đồng-hồ NGÀY+EPOCH, vest-gated-per-epoch + forfeit-1001-idle-epoch, chống-double-satisfaction; `plutus.json` khớp code; 69 test riêng `activation_logic` PASS; qua red-team nội bộ. Còn: apply-param builder, sửa comment sai đầu file. PR chờ đội on-chain duyệt.

**Backend/Core:** GetLAMP orchestration, anti-idle PHA-1, vest/forfeit PHA-2, ClaimVested, GetMAGIC — chờ backend + Core Enclave. Chưa có evidence `curl`.

**Blocker:** B1 engine Gen đọc-số-dư (MAGIC/CARP-team). B2 Registry dịch-vụ-tiêu-tài-nguyên (`has_counterparty_consume` placeholder). B3 GetLAMP-PersonDID chờ PA2. B4 GreenBack settlement + fee_refill phản-chu-kỳ.

**AbandonPhase1:** không có redeemer on-chain trong thiết-kế hiện tại — thoát-sớm PHA-1 qua anti-idle tự thu-hồi (không có nút chủ-động). **Rủi ro theo dõi:** pot cạn khi nhiều user cùng PHA-2 (R1); wash-rỗng nếu Registry lỏng (R2).

## Feecover

**Spec:** MERGED (#14) — 4 doc chuẩn-hoá. **Code:** ConsumeMAGIC lõi (C-CM-1..5) Done (kế thừa đội ConsumeMAGIC, không chứng-minh Feecover đúng). Layer Feecover (`ServiceFeeSchedule`/`FeecoverGate`/`FeecoverAccrual`/`FeecoverEpochSettle`/quy-đổi-CARP) — 0 dòng, chưa test. `EngageDatum.did_commit` field có, immutable-enforced, MVP nội-dung sentinel rỗng.

**Blocker:** B1 MAGIC-model, B2 CARP policy-id, B3 enforce nội-dung `did_commit` per-DID. **Hở nội-tại nặng nhất:** FG-4 — EpochSettle pseudo-code, 0 validator, dựa provider trung-thực (KHÔNG blocker đội khác — Feecover tự vá). Phụ-thuộc mềm: Resolve API point-in-time (backend).

**Giá 2 thao-tác DID — CHỐT 2026-08-10** (khép một phần D1/D3, `PhoenixKey-Feecover-Math.md §7.2bis`): `did.rotate` = 2 MAGIC **giá cố-định**, `did.transfer` = 10 MAGIC (nhân theo cầu bình-thường), `UpdateGuardians` **miễn phí**. Tỉ-lệ 1:5 giữ nguyên từ bảng ADA cũ (§36) để việc bỏ đường ADA không đồng-thời là một đợt đổi giá ngầm. Hai bất-biến mới: FEECOVER-PRICE-1 (rotate KHÔNG nhân theo cầu — `demand_mult` là một số dùng chung toàn hệ, nên tải của module khác sẽ định giá thao-tác an-ninh của DID, và một đợt lộ khoá hàng loạt tự đẩy giá lên đúng lúc cần rẻ nhất); FEECOVER-PRICE-2 (thiếu MAGIC KHÔNG được chặn Rotate — nếu chặn thì kẻ đã lộ khoá nạn-nhân chỉ cần làm cạn MAGIC của họ là khoá lộ không bao giờ bị xoay).

**Blocker MỚI (chưa từng ghi):** B4 — ConsumeMAGIC **chưa có lớp giá cố-định**; công-thức áp `demand_mult` cho mọi `op_type`, không loại-trừ. Đây là **điều-kiện tiên-quyết** để nối `did.rotate` (đã gửi yêu-cầu sửa hợp-đồng sang MAGIC 2026-08-10). `did.transfer` KHÔNG phụ-thuộc B4, đi trước được. B5 — `op_type` 7/8 chưa được MAGIC cấp; bảng `op_prices` sắp tăng ngặt, trần 16 dòng, hiện dùng 6.

**Điều-kiện wiring (W1-W4):** W1 nối đường MAGIC **cùng đợt** với gỡ đường ADA, không song-song (hai cơ-chế phí cùng gắn một redeemer ⟹ thu kép hoặc bên-nào-rẻ-hơn-thắng tuỳ builder off-chain). W2 committee `PriceParam` phải > 1-of-N trước mạng chính — ngưỡng 1 cho phép một khoá chi UTxO beacon ngay trước tx nạn-nhân ⟹ từ-chối xoay khoá nhắm đúng một người, lặp vô hạn.

**Lộ-trình:** P0 (chốt D1-D6) → P1 (B1/B2/B3/**B4/B5**) → P2 (build + vá FG-4) → P3 (test) → P4 (per-DID) → P5 (production).

## Protectme

Cổng chi-trả `protectme_logic.ak`+`protectme_payout.ak` (branch `feat/protectme-payout`) — khối duy nhất có code+test đối-kháng sạch (double-satisfaction, cred-collision, ADA-skim, miền-số, cross-bucket đều chặn): **39 test (14 logic + 25 validator)**. 🔴 `protectme_beacon.ak` one-shot per claim_id — 0 dòng, CHẶN-MERGE (thiếu → 1 claim_id nạp escrow 2 lần → solvency vỡ). 2-bucket Treasury + Feecover premium wiring + resolver claim + UI — chưa code (backend/UI). 11 quyết-định PROT-1..11 chờ chốt (🔴 PROT-10 evidence-bar, PROT-11 cohort, PROT-4 ngưỡng SYS/USER). Blocker hạ-tầng: MAGIC-model, CARP policy-id, Beacon. **NO-GO tới khi tất cả chốt.**

## Knowme

**Code (verify 2026-07-09):** Mức 1 (tự-khai) + Mức 2 (xuất-trình chọn-lọc) có code+test, demo `/vc`. Evidence: `npx vitest run src/lib/sdvc/` → **20 file / 415 test PASS** (~1.2s). Con-số "135" cũ trong `SD-VC-ALGORITHM-v1.md` là snapshot lỗi-thời. Lớp tài-liệu: nền có code+test (`dossier.ts`/`fingerprint.ts`/`eciesSeal`); tiết-lộ-chọn-lọc-tài-liệu + versioning Strata + re-seal = chưa code. Mức 3 ZK (BBS+): chưa code. Query gateway (VeData): chưa code.

**Blocker:** B1 lib BBS+prover (Mức 3), B2 LampNet gateway (lớp tài-liệu), B3 Glint/Spectra (VeData), B4 StampRecord Strata. **Mốc:** M1 (Mức1+2+`/vc`) chạy; M2-M7 chờ blocker.

## Easteregg

**Spec:** 4 doc hợp nhất mô hình "mức riêng-tư của ví Phoenix" (không phải ví thứ ba), chốt 2026-07-09. **Code on-chain:** `did_pool.ak` (T1 MST) + `did_subaddr.ak` (T0/L3) — chưa tồn tại. **Off-chain:** Indexer/Accountant, sweep crank, withdraw builder — chưa có. **ZK T2:** verifier Aiken chưa viết; ExUnit 2.842B là đo của Easteregg-ZK bên VeData (độc lập); ceremony chưa chạy. **Test:** 0 test Easteregg. **PoC:** 1 PoC Python trên Preview (3 tx-hash) minh-hoạ ẩn-số-dư + gated-proof, KHÔNG validator, chưa chứng-minh operator-không-rút. **Gap:** G1 (fee-split), G3 (sweep per-pair), G5 (salt-recovery) 🔴 chưa vá; G2/G4 🟡. **NO-GO toàn module**; chỉ GO build+test Preview T1 + T3-mode-1.

## Smartsend

**Vị-trí:** module độc-lập thứ 8 (chốt 2026-07-09), tách từ Rebirthme, dùng chung hạ-tầng ví/guardian/anti-drain. **Build:** `smartsend_escrow.ak` — spec đầy-đủ (SS-1..12 + SSR-4 hợp-nhất), CHƯA code, 0 test.

**Bất-biến đã hợp-nhất (không còn "vá đỏ" treo):** SS-1/SS-5′/SS-12 (value-conservation byte-perfect, `min_ada` tách field, `fee_covered` chỉ audit); SS-7′ (escrow-1-lần, chống double-satisfaction batch); SS-9′ (Accept verify controller-sig qua anchor); SS-11 (`reclaim_deadline`+`ReclaimTimeout`); SS-8/SS-8′ (Freeze trong cửa-sổ-veto; thoát qua guardian-quorum hoặc `freeze_deadline` auto-hoàn); SS-10 (`window ≥ min_window_floor`); SS-2 (veto-race biên); SS-3/SSR-4/SSR-13 (factor Cancel neo anchor-enroll).

**Phụ-thuộc-chặn ngoài:** `limit_meter.ak` (Rebirthme, hở HIGH); nền `did_payment`+guardian (173/173 PASS); verifier Glint/Spectra (VeData, Phase 2 — bind `blake2b_256(own_ref ‖ escrow_datum_hash)` SSR-12); guardian ResolveFreeze quorum (chưa build); enroll-set factor trong TAADDatum (Core Anchorme/Validator).

**CẦN CHỐT:** `reclaim_deadline` tương-đối `veto_deadline`; `window` mặc-định + `min_window_floor`; `freeze_deadline`; thứ-tự land vs anti-drain; ưu-tiên Glint sớm hay guardian-factor đủ bản đầu.

## Math (đặc-tả tổng — `PhoenixKey-Math.md`)

Hiện-trạng triển-khai các phần của đặc-tả toán v4.6 (đã tách khỏi Math.md):

| Area | Spec | Hiện trạng |
|---|---|---|
| Crypto primitives (HKDF, Ed25519, BLAKE2b, P-256 verify, CIP-1852) | §1, §6, §8 | Implemented (`rust_core`) |
| DID Document publish (metadata label 6789) | §2 | Implemented — live preprod + preview (PhoenixKey-PoC) |
| TAAD UTxO state machine + Rotate redeemer | §10 | Validator compiles; tx-builder trên feature branch, chưa merge |
| Tiered recovery (Tier 1–5) | §11 (module Rebirthme) | Spec-only — chưa có recovery code path |
| §36 fee architecture (30/70 split, Phoenix Treasury) | §36 | Spec-only — enforcement (fee-receipt minting policy + ExUnits benchmark) chờ Validator Issue #7. Ước tính mem ~150–400, CPU ~80K–200K (+3–12% baseline ~0.17 ADA) |

---

## 4. Nhật-ký bằng chứng

| Ngày | Module | Bằng chứng |
|---|---|---|
| 2026-07-08 | Anchorme/Rebirthme | Validator `aiken check` 173/173 PASS |
| 2026-07-08 | Wakeme | `activation_logic` 69 test PASS, qua red-team nội bộ |
| 2026-07-09 | Protectme | 39 test đối-kháng (14 logic + 25 validator), branch `feat/protectme-payout` |
| 2026-07-09 | Knowme | SD-VC `vitest` 20 file / 415 test PASS |
| 2026-07-09 | Feecover | Spec MERGED #14; layer Feecover 0 dòng (grep xác nhận) |
| 2026-07-09 | Easteregg | 1 PoC Python trên Preview (3 tx-hash), 0 validator/test |
| 2026-08-08 | Validator (toàn bộ) | `aiken check` **577/577 PASS**, `aiken build` exit 0, **9 validator** ra blueprint: `did_payment` `bac16cec…` · `did_pool` `9ba97ba4…` · `did_stake` `eb535cc1…` · `lamp_policy` `ba0dd83a…` · `limit_meter_vault` `f3be6d6d…` · `protectme_payout` `b1f90fca…` · `smartsend` `9ed1b56f…` · `taad` `5ac17898…` · `wakeme_vault` `8655974a…` (hash chưa-apply-param) |
| 2026-08-08 | Đăng nhập web | Gọi thật máy chủ đang chạy: `POST /auth/session/init` → 200; `GET /api/v1/.well-known/jwks.json` → 200, `kid=phoenixkey-ed25519-1`; `POST /auth/token/exchange` tồn tại (403 với token giả). ⚠ `/.well-known/jwks.json` ở **gốc miền → 404** |
| 2026-08-08 | Backend | `Tests run: 393, Failures: 0, Errors: 0, Skipped: 0` (CI run `31252652916`); `DidOpWatermarkUpsertPostgresTest` chạy trên Postgres thật 10,37s / 6 test / Skipped 0 |
| 2026-08-08 | Tài liệu | 67 endpoint có mã / 64 có tài liệu → nay 67/67 (PR Database #132); thêm 4 sequence diagram + đặc tả 5 màn hình (PR Specs #24) |
| 2026-08-15 | Wallet (CIP-30) | `vitest run`: **14 file / 178 test PASS**; `tsc --noEmit` sạch; `check:locales` → "Locale parity OK — 4 languages × 2 namespaces". Vá 3 lỗi treo-kết-nối CIP-30 (`enable()` không bao giờ settle, đổi mạng giữa chừng, submit không chắc chắn) + 4 điểm hội-đồng chỉ ra (cổng xác-nhận tự tắt không báo, DoS liệt-kê ví qua getter ném lỗi, QR rơi về hex, mạo-danh token). Kho `PhoenixKeyDID/Wallet` PR #18 — **chưa merge** |
| 2026-08-15 | Frontend | `bun run typecheck` sạch; `bun run build` → `Compiled successfully`, 16 route có `○ /wallet` + `○ /night`; chạy thật `localhost:3000` dựng đủ 3 chế độ, đổi tiếng Việt đúng, console 0 lỗi. Gắn `Wallet` bằng submodule. PR Frontend #19 — **chưa merge, chưa deploy** |
| 2026-08-15 | Core (đọc chip giấy tờ) | eMRTD theo ICAO Doc 9303, **đã nối đầu-cuối**: `rust_core/src/emrtd/` + `emrtd/reader.rs` (chọn file · cắt khối · status word) + `emrtd/session.rs` (phiên `poll ⇄ submit`) + 5 hàm `taad_emrtd_*` ở `lib.rs` + `lib/bridge/emrtd_reader.dart` (vòng bơm). Đo: `cargo test` **431 pass / 0 fail / 1 ignored** (bộ lọc `emrtd`: **127**), `flutter test` **244 pass / 0 fail**. Cầu NFC gốc: Kotlin 217 dòng + Swift 276 dòng + Dart `EmrtdNfcTransport` 123 dòng. PR Core **#73 — chưa merge** (2 commit `a1daf61` + `962dd42`). Callback-FFI **bất khả thi** (callback C không `await` được MethodChannel) ⇒ đảo chiều: Rust chạy giao thức trên luồng riêng, dừng chờ mỗi APDU. ⚠ Còn đúng một chỗ đứt: **chưa chạm thẻ thật** — mọi số là vector chuẩn ICAO + thẻ giả lập |
| 2026-08-15 | Core | Đầu dò tầm với trả BA trạng thái (có / mất / **chưa biết**) thay vì hai. CI **4/4 pass** (`rust_core` 1m50s · Flutter 1m13s). PR Core **#74 — chưa merge** |
| 2026-08-15 | Database | CORS mở theo **ĐƯỜNG** cho `/identity/by-username/**` (`allowedOrigins("*")` + `allowCredentials(false)`, đăng ký TRƯỚC `/**`) — thay vì thêm origin vào mapping `/**` vốn mang `allowCredentials(true)` cho **mọi** đường, tức là cấp cho một trang web quyền gọi toàn bộ API kèm chứng thực người dùng. Thêm `WebConfigCorsTest` 4 test gác thứ-tự đăng-ký; kho trước đó **0 test CORS**. CI `Build + test` **pass 1m5s**. PR **#187 — chưa merge**. Kèm issue **#188** (kiểm đặt-được-tên phải biết NGƯỜI GỌI, vì chính chủ lấy lại tên cũ thì được) + **#189** (chiều ngược DID→tên — chờ chốt chính sách) |
| 2026-08-15 | SDK | `resolveByUsername` + `lookupUsername` (nuốt theo **mã** `user_not_found`, không theo `status === 404`). `tsc --noEmit` sạch; `jest` **29/29 PASS** (9 test mới). PR SDK **#15 — chưa merge**. ⚠ kho SDK **không có CI nào** (`git ls-tree` = 0 workflow) và `npm run lint` hỏng sẵn trên main (không có tệp cấu hình eslint, eslint 10 đã bỏ `.eslintrc`) ⇒ số đo trên là chạy tay, không có máy nào chạy lại |

---

## 5. Đo hiện trạng năng lực đầu-cuối (2026-08-08, bổ-sung 2026-08-15)

Mục 1–3 tổ chức theo module. Mục này tổ chức theo **việc người dùng làm được**, vì một module xanh không có nghĩa người dùng bấm được.

| Người dùng làm được gì | Hiện trạng | Chỗ đứt |
|---|---|---|
| Tạo ví **Standard** và chi tiền | **ĐƯỢC** | — (đường chi duy nhất đang chạy) |
| Tạo ví **Phoenix**, nhận tiền | **ĐƯỢC** | ⚠ địa chỉ đang dẫn ứng với CBOR bản cũ 1-chữ-ký (xem blocker mục 2) |
| **Chi tiền từ ví Phoenix** | **KHÔNG** | khoá thiết bị chưa tồn tại trong mã; không có hàm dựng+ký giao dịch 2-chữ-ký ở cả app lẫn backend |
| Nối **ví ngoài** (Lace/Eternl/Typhon) vào web, gửi ADA/token, staking, uỷ-quyền dRep, gửi Governance action | **CÓ CODE, CHƯA LÊN WEB** (2026-08-15) | code + test xong ở `Wallet` (178/178) và điểm-gắn xong ở `Frontend`; nhưng PR #18/#19 **chưa merge** và **chưa deploy** ⇒ phoenixkey.me hôm nay trả trang không-tìm-thấy cho `/wallet` |
| **Tạo bộ seed mới** (ví base + enterprise) hoặc **khôi phục bộ khoá cũ** ngay trên web | **KHÔNG — và là cố ý** | web là **cửa sổ**, điện thoại là **két**: không trang nào của kho `Wallet`/`Frontend` được phép hỏi cụm từ khôi phục (`Wallet/README.md:39-45`, `Frontend/README.md:3`). Sinh/khôi phục seed nằm ở app Core (`rust_core`). Muốn có trên web thì phải đổi thiết kế, không phải thêm màn hình |
| Bấm **Wakeme / GetLAMP** | **KHÔNG** | 3 biến `ACTIVATION_*` rỗng ⇒ `501`; pot chưa có LAMP; giao diện web hiện trỏ luồng cũ đã ngừng dùng |
| **ScheduleGen / InstantGen** | **KHÔNG** | 0 dòng mã; và đang bị cấm nối tới khi MAGIC pha-2 chỉ-đọc xong (`PhoenixKey-Wakeme-Tech.md:239`) |
| **Chạm thẻ CCCD / hộ chiếu vào điện thoại để tự lấy số định danh** (khỏi gõ tay) | **CÓ CODE ĐẦU-CUỐI, CHƯA THỬ THẺ THẬT** (2026-08-15) | Chip **có** trả số định danh trong DG1 (MRZ) — không phải gõ tay. Đường đi đã liền từ ăng-ten NFC tới lõi giao thức: `emrtd/reader.rs` + `emrtd/session.rs` + 5 hàm `taad_emrtd_*` + `lib/bridge/emrtd_reader.dart` (PR Core #73, `cargo test` 431 pass · `flutter test` 244 pass). Chỗ đứt còn lại là **thực địa**: chưa lần nào chạm thẻ thật, mọi số là vector chuẩn ICAO + thẻ giả lập. Quyết-định thiết-kế trước đây treo (callback FFI hay Rust gọi thẳng Kotlin/Swift) đã tự đóng: **cả hai bất khả thi** vì callback C không `await` được MethodChannel ⇒ đảo chiều, host hỏi `poll` rồi `submit`. ⚠ Khoá mở BAC dựng từ **số giấy tờ + ngày sinh + ngày hết hạn** (Doc 9303-11 Appendix D.2, có test vector TD1 trong `mrz.rs:533`) ⇒ vẫn cần **một lần** quét OCR hoặc nhập tay ba trường đó ở mặt có MRZ trước khi chạm chip. Riêng hành vi thật của CCCD Việt Nam **chưa đo trên thẻ thật** — mọi số ở đây là test vector ICAO, không phải phép thử thực địa |
| Đăng nhập web PhoenixKey bằng QR | **ĐƯỢC** | — |
| Web **bên ngoài** tra **tên người dùng → DID** | **CÓ CODE, CHƯA MERGE** (2026-08-15) | Database PR #187 (CORS theo đường) + SDK PR #15. Hôm nay trình duyệt ở miền khác `phoenixkey.me` vẫn bị CORS chặn ⇒ ProofChat phải vòng qua máy chủ riêng |
| Ứng dụng **bên thứ ba** đăng nhập | **KHÔNG** | không có đường GHI `service[]` kiểu `SsoRedirect`; danh sách nạp-trước đang bị chú thích tắt; SDK chưa phát hành; chưa có màn hình đồng ý cấp quyền |
| Tạo **OrgDID** | **ĐƯỢC** | — |
| Đúc LAMP vào kho **qua OrgDID** | **KHÔNG** | endpoint ở 2 nhánh chưa merge |
| **Vô hiệu hoá bộ seed** sau khi đúc | **KHÔNG** | validator quyền-GHI registry chưa merge (LAMP PR #20); phía off-chain đã viết xong và đang chờ nó |
| **Nhận LAMP** từ ETD / Airdrop / SRCL | **KHÔNG**, cả ba | không có đường `POST claim` nào; SRCL còn 3 lỗ mở; phía PhoenixKey chưa có dòng nào (grep = 0) |
| Xem **danh sách người bảo trợ** của mình | **KHÔNG** | không có `GET /guardians`; `/add`+`/remove` chỉ trả về SỐ LƯỢNG. Đây là màn hình bắt buộc trong luồng khôi phục |
| **Từ chối** một yêu cầu ký | **KHÔNG** | không có `POST /sign/{id}/reject`; chỉ `approve`/`cancel`, dù enum trạng thái đã có sẵn `"rejected"` |

### Ba chỗ đang TRÊN ứng dụng mà hỏng

- Nút "Claim MAGIC" (`wallet_screen.dart:618`) gọi endpoint **luôn trả 410 Gone** (`WalletController.java:79-82`).
- `GetLampPanel.tsx` tên là GetLAMP nhưng gọi luồng VND cũ đã ngừng dùng, không gọi `/activation/getlamp/build`.
- Số dư MAGIC hiển thị **luôn 0**: `WalletV2ServiceImpl.java:182-183` gán cứng `0`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
