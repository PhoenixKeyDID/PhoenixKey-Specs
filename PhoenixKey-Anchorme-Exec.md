# PhoenixKey — Anchorme · ĐẶC-TẢ ĐIỀU-HÀNH (cho lãnh-đạo)

> **Module:** Anchorme (danh-tính lõi). **Ngày 2026-07-09.** Bản điều-hành: quyết-định + lý-do + đánh-đổi + rủi-ro + lộ-trình. Chi-tiết toán/invariant: [PhoenixKey-Anchorme-Math.md](./PhoenixKey-Anchorme-Math.md); kỹ-thuật: [PhoenixKey-Anchorme-Tech.md](./PhoenixKey-Anchorme-Tech.md); sản-phẩm: [PhoenixKey-Anchorme-Vi-Feat.md](./PhoenixKey-Anchorme-Vi-Feat.md).
> Tài-liệu này KHÔNG lặp toán.
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 1. Tóm tắt một trang

**Anchorme (danh-tính lõi) là gì.** Tầng danh-tính lõi của PhoenixKey: mỗi thực-thể (người, tổ-chức, dịch-vụ, thiết-bị…) có một DID `did:phoenix:...` **neo thẳng trên Cardano** bằng một anchor NFT độc-nhất, do khóa trong Secure Enclave điều-khiển. Không bên nào cấp-phát hay thu-hồi được danh-tính Người; đổi thiết-bị vẫn giữ nguyên con người.

**Ba trụ kỹ-thuật.**
- **Anchor TAAD + State-NFT singleton:** 1 danh-tính = 1 UTxO mang NFT `(policy = script-hash, name = blake2b_256(did))`, **khóa cứng vào địa-chỉ validator** ngay lúc đúc (Design-2). Ai cũng đối-chiếu on-chain, không cần tin máy-chủ.
- **Controller + Rotation:** danh-tính neo vào "chìa hiện-tại", xoay chìa mà DID bất-biến. Mất máy → khôi-phục (module Rebirthme), con-người giữ nguyên.
- **10 loại DID + cây sở-hữu:** Người/Tổ-chức/Dịch-vụ + phi-người (Thiết-bị/Máy/Tài-sản/Bot/AI/Ngữ-cảnh/Nhân-vật), luật CanOwn ép quan-hệ cha-con on-chain.

**Bản chất.** Đây là **hạ-tầng danh-tính Layer-0** cho mục-tiêu "Open SDK cho mọi Cardano team": danh-tính tự-chủ, chống-làm-giả (byte-for-byte on-chain), xoay-chìa-không-mất-người, mô-hình-hóa được cả cá-nhân lẫn doanh-nghiệp lẫn máy.

> 🔴 **CỔNG GO/NO-GO (luật cố-định, không đổi theo thời-gian):** Validator an-toàn về cấu-trúc, NHƯNG **KHÔNG mở tạo/custody danh-tính Người trên production tới khi PA2 UniquenessThread land.** Lý-do: GenesisPerson đúc được anchor-Person-giả cùng name với nạn-nhân + controller attacker (khóa phần-cứng P-256 không verify on-chain) → chiếm quyền, rút custody. **Danh-tính Tổ-chức/Dịch-vụ KHÔNG dính** (có chữ-ký-cha xác thực). Đây là lỗ **mã-hoá anchor**, KHÔNG phải sinh-trắc — sinh-trắc Secure Enclave đã đủ chống trùng người. Chi-tiết: [PhoenixKey-Anchorme-Math.md](./PhoenixKey-Anchorme-Math.md) §8-9.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 2. Bảng quyết-định (quyết | lý-do 4 trục | đánh-đổi)

Trục: (a) định-hướng dài-hạn · (b) first-principles · (c) tối-ưu eUTXO/ExUnit/phí/đơn-giản · (d) lợi-ích user + bền-vững.

| # | Quyết-định | Lý-do | Đánh-đổi (trung-thực) |
|---|---|---|---|
| **Q1** | **Anchor Design-2 đa-mục-đích** (1 validator vừa mint vừa spend; policy ≡ script-hash) | (b) NFT-policy = chính script-hash → genesis HARD-BIND NFT vào địa-chỉ validator, đóng lỗ Design-1 (NFT ra ví lạ). (c) 1 validator, không phụ-thuộc vòng. | Mọi tx Mint phải qua đúng handler; lỗi handler = mọi mint rơi fail. |
| **Q2** | **Anchor-name tất-định `blake2b_256(did)`** (KHÔNG BLAKE3) | (b) name off-chain = asset-name on-chain → cross-check thẳng TAAD UTxO, resolver không cần trust-server. (a) binding toàn-hệ (ví, mint LAMP, resolver). | Chính tính tất-định này khiến uniqueness-toàn-cục KHÔNG ép được ở mint (gốc lỗ Q6/R1). |
| **Q3** | **Controller đọc-động + Rotation; DID bất-biến** | (a) mất-máy không mất danh-tính; (d) mọi quan-hệ neo-theo-DID sống-sót xoay-chìa. (b) danh-tính ≠ khóa. | Vòng-đời dài; cần state-machine seq-monotonic chống replay. |
| **Q4** | **HW_Key P-256 carry-by-equality** (validator KHÔNG decode curve) | (c) tránh ExUnit decode P-256 on-chain; (b) khóa gốc trong Enclave, không xuất được. | 🔴 Trực-tiếp là gốc lỗ R1: không có ràng-buộc on-chain buộc did-string ↔ HW thật. |
| **Q5** | **10 loại DID + CanOwn on-chain** (thay 1-loại-phẳng) | (a) mô-hình-hóa tổ-chức/dịch-vụ/máy; (b) cây sở-hữu chứng-minh-được, luật authority riêng mỗi loại. | Ma-trận CanOwn + type-byte phải đồng-bộ 3-impl (hiện LỆCH — R4). |
| **Q6** | **Đóng lỗ anchor-Person qua PA2 structural** (thread spend-based), KHÔNG qua mint-check | (b) mint-policy Cardano không đọc UTXO-set → uniqueness-toàn-cục ĐÒI global-state chuỗi-hoá (thread). PA2 giữ địa-chỉ ví nguyên bytes. (d) đóng crypto-cứng. | Contention 1-thread → phải shard; sorted-list scale ~triệu, dân-số phải Merkle (chi-phí off-chain). |

---

## 3. Ma-trận rủi-ro (rủi-ro | mức | giảm-thiểu)

| ID | Rủi-ro | Mức | Giảm-thiểu |
|---|---|---|---|
| **R1** | **Lỗ mã-hoá-anchor Person-over-Person.** GenesisPerson đúc anchor-Person-giả cùng name + controller attacker → chiếm ví/custody PersonDID. ĐÃ verify PASS (red-team). | 🔴 CAO (rút-được tiền) | **KHÔNG mở PersonDID production tới khi PA2 land.** PA5-a (thu-hẹp cross-entity) làm ngay. Org/Service AN-TOÀN (parent-sig). **KHÔNG phải lỗ sinh-trắc** — sinh-trắc Enclave đủ chống trùng người. |
| **R2** | **PA2 sorted-list không scale dân-số** (N/shard ≤ ~400, K=20 triệu shard cho 8 tỷ người = bất-khả-thi). | 🟡 TRUNG (tham-số) | Enterprise/early Person: sorted-list K=256 (~triệu). Dân-số: Merkle-root-in-datum (datum O(1), witness off-chain). Cần maintainer chốt trục chi-phí. |
| **R3** | **resolve-by-hash + point-in-time chưa dựng** → Strata trả 424, chặn ProofChat/OriLife/VeData/MAGIC did_commit. | 🟡 TRUNG | Giao đội backend: index ngược `did_hash→did` + `did_state_history` V16 (append-only, fail-closed 503). Spec ready. |
| **R4** | **Type-code: văn Math lệch, impl KHÔNG lệch.** ĐÃ verify byte-exact: generator Java ≡ validator Aiken (Device=2…Service=7,Context=8). Chênh chỉ ở **văn Math §2.2** (thứ-tự Context=2). Rủi-ro: bản Rust/mobile bám nhầm văn Math. | 🟡 TRUNG | Person/Org/Avatar (0,1,9) TRÙNG → chưa lộ. **Chốt bảng-byte code làm canonical, sửa văn Math** (rẻ hơn rebake code); rà Rust/mobile trước khi mint author-DID phi-nhân (ProofChat v2). |
| **R5** | **DeviceDID chưa on-chain** → LampNet node admission / Knowme device / Rada kẹt. | 🟡 TRUNG | Giao đội on-chain (`Op_create_device`) + đội backend (hw_cert verify TPM2/SEP/FIDO2, validator chỉ neo hash). Spec dev-ready. |
| **R6** | **Validator tin backend cho DeviceDID hw_cert** (validator không parse chuỗi cert nhà-SX). | 🟢 THẤP | v1.0: backend gatekeeper cert; validator ép cấu-trúc (hash≠0, TTL, LampNetNode⇒Hardware). v1.1: PCR registry reference-input. |
| **R7** | **`Migrated` status chết** (định-nghĩa, không transition tới). | 🟢 THẤP | Không ảnh-hưởng an-toàn hiện-tại; chốt ai/khi-nào Active→Migrated hoặc bỏ field. |

---

## 4. Ranh-giới blocker (luật phụ-thuộc, không đổi theo thời-gian)

- **B1 — PA2 (đội on-chain + audit):** chặn mở PersonDID-custody production (lỗ R1). Org/Service KHÔNG chặn.
- **B2 — resolve-by-hash + point-in-time (đội backend):** chặn Strata/VeData/MAGIC did_commit/Rada.
- **B3 — DeviceDID (đội on-chain + đội backend):** chặn LampNet node / Knowme device / Rada device-binding.
- **B4 (Math) — Full_Authority `⊑` fix + type-code canonical:** chặn delegation PersonDID + author-DID phi-nhân.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 5. Lộ-trình mốc

**Nguyên-tắc thứ-tự (luật cố-định):** lõi anchor build-test ĐỘC-LẬP trước → vá lỗ Person (PA5-a nhanh → PA2 structural) → dựng resolver-nâng-cao → DeviceDID → registries.

| Mốc | Nội-dung | Phụ-thuộc |
|---|---|---|
| **M1** | Validator `taad` Design-2 + genesis/rotate/transfer/deactivate + resolver W3C | — |
| **M2** | Maintainer chốt PA2 trục chi-phí; PA5-a vào code; type-code canonical (bảng-byte code) | maintainer + đội on-chain |
| **M3** | PA2 wiring `taad.mint` (thread + register/burn) → **mở PersonDID production** | **B1** |
| **M4** | resolve-by-hash + point-in-time V16 (đội backend) → Strata/VeData/MAGIC hết 424 | **B2** |
| **M5** | DeviceDID on-chain + endpoints → LampNet/Knowme/Rada | **B3** |
| **M6** | ServiceDID self-service + Authorization/Mint-Authority Registry + Permission-Consent | duyệt |

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 6. Câu hỏi cần LÃNH-ĐẠO chốt

| # | Câu hỏi | Đề-xuất mặc-định | Vì sao cần maintainer |
|---|---|---|---|
| **Q-A** | **PA2 trục chi-phí:** sorted-list-sharded (K=256, ~triệu, làm được ngay) hay Merkle-root (dân-số, cần indexer giữ cây)? | Sorted-list cho enterprise/early Person; Merkle khi vượt triệu. | Đổi kiến-trúc genesis; quyết crypto-cứng vs uniqueness-mềm resolver-first-wins. |
| **Q-B** | **Mở PersonDID production khi nào?** Trước PA2 (rủi-ro R1 mở) hay chỉ sau? | CHỈ sau PA2 (hoặc uniqueness-mềm + PA5 nếu chấp-nhận 🟡 tồn-dư). | Cổng GO/NO-GO an-toàn tài-sản. |
| **Q-C** | **Type-code canonical:** generator Java + validator Aiken đã KHỚP byte-exact (Device=2…Service=7,Context=8); chỉ **văn Math §2.2** lệch thứ-tự. Lấy bên nào? | **Bảng-byte code canonical → sửa văn Math §2.2** (KHÔNG rebake code + KHÔNG regen vector). | Chặn author-DID phi-nhân; nếu chọn Math sẽ phải rebake 2 impl đang đúng — đắt và rủi-ro hơn. Cần maintainer rà bản Rust/mobile có bám nhầm văn Math không. |
| **Q-D** | **ServiceDID anchor:** metadata-6789 (live ngay) hay TAAD anchor NFT entity=Service (canonical)? | 2 giai-đoạn: GĐ1 metadata-6789, GĐ2 TAAD anchor khi tx-builder merge; `client_id ≜ service_did` bất-biến. | Directive lifecycle; ảnh-hưởng accountability + self-service. |
| **Q-E** | **Full_Authority `⊑` fix + đưa vào Math v4.7?** | Vào core v4.7 (bug authority-model, không tách lớp). | Chặn mọi delegation PersonDID; cần rà theorem dùng set-⊆. |

---

## 7. Ghi trung-thực (không over-claim)

- **🔴 Lỗ R1 (CID-1) là lỗ MÃ-HOÁ ANCHOR, KHÔNG phải sinh-trắc/sybil.** Sinh-trắc Secure Enclave đủ chống trùng người. Lỗ nằm ở: GenesisPerson đúc anchor did-string bất-kỳ + controller attacker vì HW_Key P-256 không verify on-chain → drain custody PersonDID. Org/Service AN-TOÀN (parent-sig G-1). Validator on-chain ĐÚNG về cấu-trúc; lỗ ở **giả-định-tầng-uniqueness chưa-thoả**. PA2 đóng structural; PA5-a thu-hẹp bề-mặt.
- **PA2 KHÔNG breaking địa-chỉ ví:** did_payment/stake/subaddr giữ nguyên bytes (không đổi compile-param); chỉ genesis-tx thêm spend thread. Đây là ưu-điểm quyết-định so với phương-án PA4 (đã loại vì breaking).
- **Ranh-giới MECE:** Core-Anchorme = TAAD/anchor/genesis/rotate/transfer/deactivate/resolver/DeviceDID/PA2/PA5/registry/consent. Recovery-guardian-flow (Init/Cancel/Finalize + guardian + backup + anti-drain + seed) thuộc **Rebirthme** — chỉ dẫn-chiếu.
- **Ranh-giới sửa code:** validator (đội on-chain) + backend (đội backend) thuộc PhoenixKey backend — nhóm tài-liệu KHÔNG sửa; phát-hiện lỗi → báo maintainer / tạo Issue. Tài-liệu này chỉ đặc-tả.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## Nguồn

- `PhoenixKey-Anchorme-{Vi-Feat,Math,Tech}.md`.
- `PhoenixKey-Validator`: `validators/taad.ak`, `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak`.
- Nguồn thiết-kế nội-bộ (không công khai).
- `PhoenixKey-Specs/PhoenixKey-Math.md` §2, §4–§5, §10, §22.
