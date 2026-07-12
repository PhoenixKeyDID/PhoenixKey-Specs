# PhoenixKey — STATUS (hiện trạng & tiến độ)

> **File này là báo cáo hiện trạng, KHÔNG phải đặc tả.** Bộ spec (`*-Vi-Feat/-Math/-Tech/-Exec`) là **kim chỉ nam thiết kế** — mô tả hệ thống ĐÍCH mà các đội dev xây tới. File này ghi *đang ở đâu trên đường tới đó*: cái gì đã chạy, chặn bởi ai, bằng chứng test. Khi hai bên lệch → spec là mục tiêu, STATUS là thực tại.
>
> Cập nhật: 2026-07-09. Nguồn: audit per-module + đối chiếu code/CI thật.

---

## 1. Bảng trạng thái 8 module

| Module | Nền đã chạy được | Blocker chính | Production |
|---|---|---|---|
| **Anchorme** | validator `taad` Design-2 (genesis Người/con, rotate, transfer 2-of-2, deactivate, CanOwn); resolver W3C; register metadata-6789 | PA2 (CID-1 anchor-forgery Person), DeviceDID, resolve-by-hash | NO-GO PersonDID-custody tới khi PA2/PA5-a land |
| **Rebirthme** | ví theo-DID `did_payment`, đóng băng theo trạng thái, guardian recovery ngưỡng+timelock, P-256 low-s, `lampnet.rs` (173/173 Aiken PASS) | 🔴 `limit_meter.ak` anti-drain KHÔNG tồn tại; `did_subaddr`/`did_stake` chưa có | NO-GO ví giá trị lớn tới khi anti-drain land |
| **Wakeme** | validator `activation_vault`+`activation_logic` (5 redeemer, vest-gated, forfeit; 69 test riêng) | B1 engine Gen đọc số dư, B2 Registry consume-gate, B3 PA2 cho GetLAMP-PersonDID | NO-GO tới khi Registry + PA2 land |
| **Feecover** | ConsumeMAGIC lõi (kế thừa) | Layer Feecover 0 dòng; B1 MAGIC-model, B2 CARP policy-id, B3 did_commit per-DID; FG-4 EpochSettle tự vá | NO-GO tới khi B1+B2+B3 + FG-4 |
| **Protectme** | cổng chi trả `protectme_logic`+`protectme_payout` (39 test đối kháng sạch, branch `feat/protectme-payout`) | 🔴 `protectme_beacon.ak` one-shot 0 dòng (chặn-merge); 2-bucket + resolver + UI chưa có; 11 quyết định PROT-1..11 | NO-GO tới khi beacon + blocker + quyết định |
| **Knowme** | Mức 1+2 SD-VC có code+test, demo `/vc` (20 file / 415 test PASS) | B1 lib BBS (Mức 3), B2 LampNet gateway (lớp tài liệu), B4 StampRecord | M1 chạy; Mức 3/tài liệu chờ blocker |
| **Easteregg** | 1 PoC Python off-chain trên Preview (3 tx-hash, không validator) | `did_pool.ak`/`did_subaddr.ak` chưa có; ZK Tầng 2 verifier chưa viết; G1/G3/G5 chưa vá | NO-GO; chỉ GO build+test Preview T1 + T3-mode-1 |
| **Smartsend** | spec đầy đủ SS-1..12 | `smartsend_escrow.ak` 0 dòng; phụ thuộc `limit_meter.ak` (Rebirthme); verifier Glint/Spectra (Phase 2) | NO-GO; money-critical, review trước code |

---

## 2. Ma trận blocker xuyên-module (ai gỡ)

| Blocker | Chặn module | Đội gỡ | Ghi chú |
|---|---|---|---|
| **B1 — MAGIC engine đọc số dư** | Wakeme, Feecover, Protectme | MAGIC team | Model chốt = account-in-vault (không native); còn spell-out engine reference-input, không spend/đốt LAMP |
| **B2 — CARP policy-id/asset-name/decimals** | Feecover, Protectme, Rebirthme, Wakeme | CARP team | preprod + mainnet; test đang dùng hằng giả |
| **B3 — Registry consume-gate + `did_commit` per-DID** | Wakeme, Feecover | MAGIC team + backend | `has_counterparty_consume` còn placeholder; `did_commit` field có, nội dung sentinel rỗng |
| **PA2 — UniquenessThread (anchor uniqueness on-chain)** | Anchorme (PersonDID-custody), Wakeme (GetLAMP-Person) | on-chain | Đóng hẳn CID-1 (giả mạo anchor did-string — KHÔNG phải sybil-sinh trắc). Chốt trục chi phí: sorted-list K=256 vs Merkle dân số |
| **PA5-a — entity-gate** | Anchorme (thu hẹp bề mặt PersonDID) | on-chain (Tuân) | PoC 4 test, chờ vào code |
| 🔴 **`limit_meter.ak` — anti-drain** | Rebirthme (I-CURVE-4 load-bearing), Smartsend (SS-6 chống trộm) | on-chain | KHÔNG tồn tại. Ưu tiên số 1 (M2). value chỉ bảo vệ mức SEED |
| **DeviceDID `Op_create_device`** | LampNet node, Knowme device, Rada | on-chain + backend | + hw_cert verify endpoint |
| **FG-4 — Feecover EpochSettle validator** | Feecover | đội Feecover | Pseudo-code, 0 validator, dựa provider trung thực — tự vá khi build |
| **B4 — Math `⊑` + type-code canonical** | delegation PersonDID, author-DID phi nhân | Math/maintainer | Chốt vào Math v4.7; bảng-byte code làm canonical |

---

## 3. Chi tiết từng module

## Anchorme

**Test bắt buộc:** module danh tính (`taad_logic` + `state_nft_logic` + `attack_tests`) phủ GenesisPerson/GenesisChild/can_own/Rotate/Cancel/Finalize/Deactivate + regression Bug#3. Số test toàn-repo Validator = 173/173 PASS (2026-07-08).

**Đã build:** validator `taad` Design-2 (genesis Người/con, rotate, transfer 2-of-2, deactivate, CanOwn); resolver W3C backend; PersonDID register (metadata-6789).

**Blocker mở:** B1 CID-1 (giả mạo anchor Person-over-Person; HW P-256 không verify on-chain → chặn PersonDID-custody; Org/Service an toàn nhờ parent-sig) — giải: PA2 (đóng hẳn) + PA5-a (thu hẹp). B2 resolve-by-hash + point-in-time V16 (backend). B3 DeviceDID `Op_create_device` (on-chain) + hw_cert endpoint (backend). B4 Full_Authority `⊑` + type-code canonical (Math v4.7).

**Bug live đã biết:** GET `/identity/{did}/pubkey` trả 500 với user đã qua recovery (consumer: backend bên thứ 3 OriLife/AladinWork) — cần Long vá.

**Byte-9 `Character`→`Avatar` — CHỐT 2026-07-10** (2 vòng phản biện đối kháng, xem `PhoenixKey-Math.md` §21): ranh giới Asset/Avatar dựa **nơi ra quyết định** (locus-of-control, không dùng "agency" — dễ lộn AgentDID byte-6): Avatar = chỉ hành động khi nhận lệnh trực tiếp từ controller ngoài; Asset = không nhận lệnh, chỉ transfer/consume. Avatar chỉ do PersonDID/OrgDID vận hành (I-CHAR-1 sửa `{Person,Service}`→`{Person,Org}` — CanOwn §22.1 + `can_own()` on-chain vốn đã đúng, I-CHAR-1 là bên sai, đã vá). "Sống→chết" = burn AvatarDID + mint N AssetDID với `derived_from` nối phả hệ (không phải type-transition tại chỗ). Đã sửa xong Math.md (10 chỗ) + Anchorme-Math/Tech/Exec.md + DIDMethod-W3C.md, push nhánh `claude/spec-northstar-2026-07-10`. **Việc tồn đọng riêng (chưa quyết, không nằm trong đợt này):** tách owner/operator cho sinh vật hoang dã không ai đứng tên; uỷ quyền Service ký hộ Org khi mint Avatar hàng loạt (đẩy sang Tech.md).

**Câu hỏi thiết kế MỞ — Byte-4 `Asset` chỉ physical** (còn treo, KHÔNG còn phụ thuộc byte-9 nữa vì byte-9 đã chốt độc lập): lỗ hổng MECE — tài sản số thụ động (file/dataset/media/NFT/VC-schema/model-weights) rơi khe (≠Asset physical, ≠Bot/Agent tự chủ, ≠Service sản phẩm, ≠Avatar). Chọn: (a) nới định nghĩa Asset → physical HOẶC digital (thêm `asset_domain: Physical|Digital`, `physical_id`/`location_proof` chuyển Optional — đề xuất từ panel phản biện 2026-07-10, chưa anh chốt câu chữ cuối); hay (b) digital = VC/metadata dưới DID khác (out-of-scope, ranh giới hẹp). Byte-value bất biến → hash-safe dù chọn hướng nào; lan tới Math §17 + Aiken `types.ak` + Java `DidPhoenixGenerator`. `AI`→`Agent` (byte-6) đã chốt đổi (issue Long).

## Rebirthme

**Nền đã chạy (173/173 Aiken PASS, 2026-07-08):** ví theo-DID `did_payment` (chi khi Active + controller ký; tài sản sống qua rotate; địa chỉ bất biến); đóng băng theo trạng thái (Recovering/Migrated/Revoked chặn chi); singleton-anchor I-WALLET-4/5; guardian recovery Init/Cancel/Finalize/UpdateGuardians(≤5) + timelock 3600 slot + collateral 50 ADA (bỏ Shamir); ví Standard + Rotation Account; P-256 low-s (I-SIGN-LOWS); `lampnet.rs` fail-closed (I-VAULT-4); Ed25519 dalek deterministic.

**Chưa có code:** 🔴 `limit_meter.ak` anti-drain — KHÔNG tồn tại (I-CURVE-4 load-bearing, hở HIGH, ưu tiên M2); 🔴 `did_subaddr.ak` (L3 unlinkable, chờ chốt [DEP-2]); 🔴 `did_stake.ak` (stake theo-DID). 🟡 I-CURVE-5 chưa enforce builder; kho bí mật/phả hệ seed chưa hợp nhất; export re-key UI chưa cắm mặc định; guardian nâng cao (trọng số/veto/cap) Todo; chứng thực VeData-Glint/Midnight chờ VeData. ⚪ CIP-30 connector, legacy-migration, on-ramp mandate, pool-ops (KES/VRF) build-ready-Todo.

**Blocker ngoài:** CARP policy-id, stake-state indexer (backend), Merkle LAMP (LAMP), schema anchor mới vào TAADDatum (backend, chờ duyệt), crate KES/VRF (PoC).

**Lộ trình:** M1 (resolver L1/L2 + wallet API v2 + blob-đơn) → M2 (`limit_meter.ak` + I-CURVE-5) → M3 (`did_stake.ak` + export re-key UI) → M4 (phả hệ seed + Strata) → M5 (`did_subaddr.ak` + registry-lib mode-2 + pool KES).

**Deprecate (bỏ dùng):** endpoint `/wallet/magic/claim` (MAGIC claim custodial) — sai model, MAGIC là account-in-vault chứ không native trong ví; app phải lấy MAGIC từ vault, không qua endpoint ví.

## Wakeme

**Validator:** `activation_vault.ak`+`activation_logic.ak` — 5 spend redeemer (GenDrip/Reclaim/VestToOwner/ClaimVested/ForfeitPhase2) + 2 mint-gate, datum 9-field, đồng hồ NGÀY+EPOCH, vest-gated-per-epoch + forfeit-1001-idle-epoch, chống-double-satisfaction; `plutus.json` khớp code; 69 test riêng `activation_logic` PASS; qua red-team nội bộ. Còn: apply-param builder, sửa comment sai đầu file. PR chờ đội on-chain duyệt.

**Backend/Core:** GetLAMP orchestration, anti-idle PHA-1, vest/forfeit PHA-2, ClaimVested, GetMAGIC — chờ backend + Core Enclave. Chưa có evidence `curl`.

**Blocker:** B1 engine Gen đọc số dư (MAGIC/CARP-team). B2 Registry dịch vụ tiêu tài nguyên (`has_counterparty_consume` placeholder). B3 GetLAMP-PersonDID chờ PA2. B4 GreenBack settlement + fee_refill phản chu kỳ.

**AbandonPhase1:** không có redeemer on-chain trong thiết kế hiện tại — thoát sớm PHA-1 qua anti-idle tự thu hồi (không có nút chủ động). **Rủi ro theo dõi:** pot cạn khi nhiều user cùng PHA-2 (R1); wash-rỗng nếu Registry lỏng (R2).

## Feecover

**Spec:** MERGED (#14) — 4 doc chuẩn hoá. **Code:** ConsumeMAGIC lõi (C-CM-1..5) Done (kế thừa đội ConsumeMAGIC, không chứng minh Feecover đúng). Layer Feecover (`ServiceFeeSchedule`/`FeecoverGate`/`FeecoverAccrual`/`FeecoverEpochSettle`/quy đổi-CARP) — 0 dòng, chưa test. `EngageDatum.did_commit` field có, immutable-enforced, MVP nội dung sentinel rỗng.

**Blocker:** B1 MAGIC-model, B2 CARP policy-id, B3 enforce nội dung `did_commit` per-DID. **Hở nội tại nặng nhất:** FG-4 — EpochSettle pseudo-code, 0 validator, dựa provider trung thực (KHÔNG blocker đội khác — Feecover tự vá). Phụ thuộc mềm: Resolve API point-in-time (backend).

**Lộ trình:** P0 (chốt D1-D6) → P1 (B1/B2/B3) → P2 (build + vá FG-4) → P3 (test) → P4 (per-DID) → P5 (production).

## Protectme

Cổng chi trả `protectme_logic.ak`+`protectme_payout.ak` (branch `feat/protectme-payout`) — khối duy nhất có code+test đối kháng sạch (double-satisfaction, cred-collision, ADA-skim, miền số, cross-bucket đều chặn): **39 test (14 logic + 25 validator)**. 🔴 `protectme_beacon.ak` one-shot per claim_id — 0 dòng, CHẶN-MERGE (thiếu → 1 claim_id nạp escrow 2 lần → solvency vỡ). 2-bucket Treasury + Feecover premium wiring + resolver claim + UI — chưa code (backend/UI). 11 quyết định PROT-1..11 chờ chốt (🔴 PROT-10 evidence-bar, PROT-11 cohort, PROT-4 ngưỡng SYS/USER). Blocker hạ tầng: MAGIC-model, CARP policy-id, Beacon. **NO-GO tới khi tất cả chốt.**

## Knowme

**Code (verify 2026-07-09):** Mức 1 (tự khai) + Mức 2 (xuất trình chọn lọc) có code+test, demo `/vc`. Evidence: `npx vitest run src/lib/sdvc/` → **20 file / 415 test PASS** (~1.2s). Con số "135" cũ trong `SD-VC-ALGORITHM-v1.md` là snapshot lỗi thời. Lớp tài liệu: nền có code+test (`dossier.ts`/`fingerprint.ts`/`eciesSeal`); tiết lộ chọn lọc tài liệu + versioning Strata + re-seal = chưa code. Mức 3 ZK (BBS+): chưa code. Query gateway (VeData): chưa code.

**Blocker:** B1 lib BBS+prover (Mức 3), B2 LampNet gateway (lớp tài liệu), B3 Glint/Spectra (VeData), B4 StampRecord Strata. **Mốc:** M1 (Mức1+2+`/vc`) chạy; M2-M7 chờ blocker.

## Easteregg

**Spec:** 4 doc hợp nhất mô hình "mức riêng tư của ví Phoenix" (không phải ví thứ ba), chốt 2026-07-09. **Code on-chain:** `did_pool.ak` (T1 MST) + `did_subaddr.ak` (T0/L3) — chưa tồn tại. **Off-chain:** Indexer/Accountant, sweep crank, withdraw builder — chưa có. **ZK T2:** verifier Aiken chưa viết; ExUnit 2.842B là đo của Easteregg-ZK bên VeData (độc lập); ceremony chưa chạy. **Test:** 0 test Easteregg. **PoC:** 1 PoC Python trên Preview (3 tx-hash) minh hoạ ẩn số dư + gated-proof, KHÔNG validator, chưa chứng minh operator-không rút. **Gap:** G1 (fee-split), G3 (sweep per-pair), G5 (salt-recovery) 🔴 chưa vá; G2/G4 🟡. **NO-GO toàn module**; chỉ GO build+test Preview T1 + T3-mode-1.

## Smartsend

**Vị trí:** module độc lập thứ 8 (chốt 2026-07-09), tách từ Rebirthme, dùng chung hạ tầng ví/guardian/anti-drain. **Build:** `smartsend_escrow.ak` — spec đầy đủ (SS-1..12 + SSR-4 hợp nhất), CHƯA code, 0 test.

**Bất biến đã hợp nhất (không còn "vá đỏ" treo):** SS-1/SS-5′/SS-12 (value-conservation byte-perfect, `min_ada` tách field, `fee_covered` chỉ audit); SS-7′ (escrow-1-lần, chống double-satisfaction batch); SS-9′ (Accept verify controller-sig qua anchor); SS-11 (`reclaim_deadline`+`ReclaimTimeout`); SS-8/SS-8′ (Freeze trong cửa sổ-veto; thoát qua guardian-quorum hoặc `freeze_deadline` auto-hoàn); SS-10 (`window ≥ min_window_floor`); SS-2 (veto-race biên); SS-3/SSR-4/SSR-13 (factor Cancel neo anchor-enroll).

**Phụ thuộc chặn ngoài:** `limit_meter.ak` (Rebirthme, hở HIGH); nền `did_payment`+guardian (173/173 PASS); verifier Glint/Spectra (VeData, Phase 2 — bind `blake2b_256(own_ref ‖ escrow_datum_hash)` SSR-12); guardian ResolveFreeze quorum (chưa build); enroll-set factor trong TAADDatum (Core Anchorme/Validator).

**CẦN CHỐT:** `reclaim_deadline` tương đối `veto_deadline`; `window` mặc định + `min_window_floor`; `freeze_deadline`; thứ tự land vs anti-drain; ưu tiên Glint sớm hay guardian-factor đủ bản đầu.

## Math (đặc tả tổng — `PhoenixKey-Math.md`)

Hiện trạng triển khai các phần của đặc tả toán v4.6 (đã tách khỏi Math.md):

| Area | Spec | Hiện trạng |
|---|---|---|
| Crypto primitives (HKDF, Ed25519, BLAKE2b, P-256 verify, CIP-1852) | §1, §6, §8 | Implemented (`rust_core`) |
| DID Document publish (metadata label 6789) | §2 | Implemented — live preprod + preview (PhoenixKey-PoC) |
| TAAD UTxO state machine + Rotate redeemer | §10 | Validator compiles; tx-builder trên feature branch, chưa merge |
| Tiered recovery (Tier 1–5) | §11 (module Rebirthme) | Spec-only — chưa có recovery code path |
| §36 fee architecture (30/70 split, Phoenix Treasury) | §36 | Spec-only — enforcement (fee-receipt minting policy + ExUnits benchmark) chờ Validator Issue #7. Ước tính mem ~150–400, CPU ~80K–200K (+3–12% baseline ~0.17 ADA) |

---

## 4. Nhật ký bằng chứng

| Ngày | Module | Bằng chứng |
|---|---|---|
| 2026-07-08 | Anchorme/Rebirthme | Validator `aiken check` 173/173 PASS |
| 2026-07-08 | Wakeme | `activation_logic` 69 test PASS, qua red-team nội bộ |
| 2026-07-09 | Protectme | 39 test đối kháng (14 logic + 25 validator), branch `feat/protectme-payout` |
| 2026-07-09 | Knowme | SD-VC `vitest` 20 file / 415 test PASS |
| 2026-07-09 | Feecover | Spec MERGED #14; layer Feecover 0 dòng (grep xác nhận) |
| 2026-07-09 | Easteregg | 1 PoC Python trên Preview (3 tx-hash), 0 validator/test |

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
