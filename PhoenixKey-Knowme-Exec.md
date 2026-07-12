# PhoenixKey — Knowme · ĐẶC-TẢ ĐIỀU-HÀNH (cho lãnh-đạo)

> Bản điều-hành: quyết-định + lý-do + đánh-đổi + rủi-ro + lộ-trình. Chi-tiết toán/invariant: `PhoenixKey-Knowme-Math.md`; kỹ-thuật: `PhoenixKey-Knowme-Tech.md`; người-dùng: `PhoenixKey-Knowme-Vi-Feat.md`.
> Tài-liệu này KHÔNG lặp toán. Phân biệt rõ **lớp lõi** vs **spec-chờ-duyệt** vs **phụ-thuộc hợp-đồng liên-module**.

---

## 1. Tóm tắt một trang

**Knowme là gì.** Module `-me` user-facing (đối xứng Protectme): kho danh-tính tự-chủ. Chủ DID tự-khai KYC/KYB, đính giấy-tờ, và **xuất-trình chọn-lọc** — chỉ đưa đúng trường cần, phần còn lại vẫn ẩn. Khác biệt gốc so với KYC tập-trung: dữ-liệu KHÔNG ở kho bên-thứ-ba; chủ DID giữ khoá, tái dùng một credential nhiều nơi mà không cần issuer online.

**Kiến-trúc 3 mức + lớp tài-liệu.**
- **Mức 1 — tự-khai** (`iss = sub`, holder tự ký). Lớp lõi.
- **Mức 2 — xuất-trình chọn-lọc** (salt + băm mỗi trường; trình đúng trường được chọn, verifier kiểm membership; khoá holder phân-giải từ DID Document chống mạo-danh; `aud`/`nonce` chống replay). Lớp lõi, tại `/vc`.
- **Lớp tài-liệu** (đính ảnh/PDF; bytes mã-hoá ECIES lưu LampNet; `docHash` vào commitment; phiên-bản bất-biến Strata; re-seal chọn-lọc). Nền: `dossier`/`fingerprint` (đính ảnh mã-hoá + fingerprint + neo dossier). Mở-rộng **[SPEC]**: tiết-lộ-chọn-lọc-tài-liệu + version + re-seal.
- **Mức 3 — ZK predicate** (chứng "đủ 18"/"quốc tịch VN" không lộ con số; nền **BBS+** chuẩn W3C [BBS-CRYPT] — tại thời-điểm viết là W3C Editor's Draft, CHƯA phải Recommendation chính-thức (cách ghi trạng-thái draft tham-khảo `PhoenixKey-DIDMethod-W3C.md §6.2` cho W3C DID Resolution); xuất-trình không-liên-kết; neo ZK-anchor + xoá GDPR). **[SPEC]**.

**Phân định ranh-giới (không lấn VeData).** PhoenixKey Knowme lo **đóng-gói + phân-giải + tiết-lộ chọn-lọc**. **VeData** lo **catalog VC + issuer thật** + các module Stamp/Query/Glint/Spectra. Knowme dẫn-chiếu, KHÔNG gọi thẳng Mosaic (chỉ qua Stamp intake công-khai).

Aligns LampNetCloud + VeDataIO.

→ Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)

> 🟡 **Điểm phải nói thẳng với lãnh-đạo:** với **ảnh giấy-tờ mắt-người-xem-được**, "thu-hồi" chỉ chặn đọc-lại — người nhận đã xem có thể chụp màn-hình. Đây là giới-hạn **cố-hữu không vá triệt-để được** (mọi tài-liệu human-viewable đều vậy). Đường thoát thật: **ưu-tiên gửi bằng-chứng-thuộc-tính (Mức 3) thay vì ảnh.** Cần chủ-trương "mặc-định proof, chỉ lộ ảnh khi luật buộc".

---

## 2. Bảng quyết-định (quyết | lý-do 4 trục | đánh-đổi)

| # | Quyết-định | Lý-do (a dài-hạn · b first-principles · c tối-ưu · d user+bền-vững) | Đánh-đổi (trung-thực) |
|---|---|---|---|
| **Q1** | **Tự-chủ dữ-liệu: giữ ở máy chủ DID, server chỉ ciphertext** (thay kho KYC tập-trung) | (a) open SDK cho mọi team Cardano; (b) không kho trung-tâm = không điểm rò-rỉ hàng-loạt; (d) chủ DID giữ chìa. | Backup/khôi-phục thuộc module khác; mất máy → dựa recovery riêng (dẫn-chiếu Rebirthme). |
| **Q2** | **BBS+ là canonical Mức 3** (SNARK/Midnight là tuỳ-chọn) | (a) BBS Cryptosuite chuẩn W3C [BBS-CRYPT] (Editor's Draft, chưa Recommendation), không trusted-setup; (b) predicate KYC thật (tuổi/quốc-tịch/hiệu-lực DN) = so-sánh + set-membership = đúng vùng mạnh BBS; (c) prover/verifier nhẹ, offchain, giải LUÔN linkability; (d) unlinkable multi-show. | BBS giới-hạn ở predicate đơn-giản; số-học phức-tạp cần SNARK. Khoá BBS (BLS12-381) song song Ed25519 — quản 2 khoá issuer. |
| **Q3** | **Tài-liệu là một claim, KHÔNG kiểu dữ-liệu mới** (commit `docHash`, bytes off-chain) | (b) selective disclosure phủ tài-liệu miễn-phí (một leaf trong `sd[]`); (c) on-chain chỉ 1 root 32 byte, eUTXO-thân-thiện; (a) tái dùng nguyên `digestOf`. | Verify tài-liệu buộc lộ plaintext cho người nhận (Q-liên-quan giới-hạn thu-hồi). |
| **Q4** | **Anchor tài-liệu QUA Stamp, KHÔNG gọi thẳng Mosaic** | (b) Mosaic chỉ nhận `anchor_request` từ Stamp (SSoT); (a) đi qua intake công-khai VeData = không cần mở hợp-đồng mới. | Cần chọn archetype StampRecord phù-hợp (open item K-3). |
| **Q5** | **KHÔNG dedup/uniqueness PersonDID trong Knowme** | (b) sinh-trắc Secure Enclave đã đủ chống trùng người; Knowme nói về **tài-liệu**, không phải "cảnh-sát danh-tính". | Chống "ảnh thật của người khác" dựa Glint (tín-hiệu) + issued (nâng-cấp), không tuyệt-đối. |
| **Q6** | **Mặc-định proof-thay-ảnh cho thứ nhạy-cảm** | (d) least-disclosure thật; (b) đừng gửi ảnh nếu chỉ cần một thuộc-tính. | Trước khi Mức 3 sẵn-sàng: chỉ có tiết-lộ chọn-lọc trường vô-hướng (Mức 2). |

---

## 3. Ma-trận rủi-ro

| ID | Rủi-ro | Mức | Giảm-thiểu |
|---|---|---|---|
| **R1** | **Verify-time plaintext resale:** người nhận đã xem ảnh giấy-tờ có thể chụp/bán lại; thu-hồi chỉ chặn đọc-lại. | 🔴 CAO (cố-hữu, không vá triệt-để) | Watermark truy-vết buộc-danh-tính-R (răn-đe pháp-lý) + `purpose`/`exp` ký kèm + **ưu-tiên proof-thay-ảnh**. UI PHẢI nói thẳng. |
| **R2** | **Tự-khai bị lạm-dụng** (nộp ảnh người khác / ảnh dựng AI). | 🟡 TRUNG | Tầng 1: Glint phát media tổng-hợp (S1/H1) — chặn deepfake, KHÔNG chặn "ảnh thật của người khác". Tầng 2: issued credential (cơ-quan ký). Kết-luận cuối thuộc bên xác-minh. |
| **R3** | **Phụ-thuộc hợp-đồng VeData** (Glint/Spectra chủ-động, Query đọc). | 🟡 TRUNG | Anchor-qua-Stamp + Query-read thoả HÔM NAY (intake công-khai). Glint/Spectra chủ-động: chạy on-device (ảnh không rời máy) hoặc chờ VeData mở hợp-đồng. |
| **R4** | **Mức 3 (BBS) phụ-thuộc lib ngoài** — I-KYC-PRIVATE/UNLINKABLE cần lib để chứng bằng code. | 🟡 TRUNG | Bọc lib BBS production (MATTR/Digital Bazaar), KHÔNG tự viết mạch. Giao đội on-chain/VeData. |
| **R5** | **Canonical JSON lệch TS ↔ Java** → membership vỡ cross-language. | 🟢 THẤP | Đã căn `canonical.ts` ≡ Jackson `ORDER_MAP_ENTRIES_BY_KEYS`. Giữ kỷ-luật khi thêm field. |
| **R6** | **Tương-quan metadata** (kích-thước ciphertext, ref_id, anchor slot lộ chùm). | 🟢 THẤP | Padding kích-thước Envelope (K-1), per-context ref alias (K-2), batch anchor mặc-định. Query có `B_pad`+DP. |

---

## 4. Phạm-vi + phụ-thuộc-chặn

**Lớp lõi** (Mức 1 tự-khai + Mức 2 xuất-trình chọn-lọc): `credential.ts`, `disclose.ts`, `commit.ts`; demo `/vc` (khai → chọn trường → disclose+seal → verify → revoke). Đi kèm: neo DID chống mạo-danh (`didDoc.ts`, `didResolver.ts`; binding + nonce trong `disclose.ts`); credential có issuer (issued) qua TrustList (`trust.ts`, `trustList.ts`); thu-hồi qua `RevocationRegistry` + `StatusList` (append-only).

**Mở-rộng [SPEC]:**
- **Lớp tài-liệu** — nền: `dossier.ts`/`fingerprint.ts` (đính ảnh mã-hoá lên LampNet + fingerprint one-hash-per-doc + neo dossier vào DID Document). Mở-rộng: tiết-lộ-chọn-lọc-tài-liệu (`DocumentClaim`/`docDigestOf` gập vào `sd[]`) + versioning Strata + re-seal per-recipient (thiết-kế Knowme-Feat-Math).
- **Mức 3 ZK** (BBS+ predicate/unlinkable/ZkAnchor/GDPR) — spec KYC-KYB-ZK.
- **Resolve point-in-time + TrustList/StatusList backend** — giao đội backend.

**Phụ-thuộc-chặn (chờ đội khác / hợp-đồng):**
- **B1 — lib BBS + prover/verifier** (đội on-chain/VeData/Frontend). Chặn toàn bộ Mức 3.
- **B2 — LampNet gateway wiring** (đội backend). Chặn lớp tài-liệu.
- **B3 — Glint/Spectra chủ-động** phụ-thuộc hợp-đồng VeData; hoặc on-device.
- **B4 — archetype StampRecord cho notarization Strata head** (đối-chiếu 21 archetype Stamp).

---

## 5. Lộ-trình mốc

| Mốc | Nội-dung | Phụ-thuộc |
|---|---|---|
| **M1** | Mức 1+2 + `/vc` | — |
| **M2** | Resolve point-in-time + TrustList/StatusList backend | đội backend |
| **M3** | Lớp tài-liệu SDK + UI "Giấy tờ của tôi" | B2 (LampNet gateway) |
| **M4** | Versioning Strata | lib Strata + B4 |
| **M5** | Mức 3 ZK (BBS lib + predicate + ZkAnchor + UI) | B1 |
| **M6** | Query read gateway (D9+DP+audit) | VeData Query (một phần B3) |

→ Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)

---

## 6. Câu hỏi cần LÃNH-ĐẠO chốt

| # | Câu hỏi | Đề-xuất mặc-định | Vì sao cần maintainer |
|---|---|---|---|
| **Q-A** | **Chủ-trương "mặc-định proof, chỉ lộ ảnh khi luật buộc"?** (R1 — giới-hạn thu-hồi ảnh cố-hữu). | Có: UI nghiêng proof-of-attribute; ảnh chỉ khi bắt-buộc, kèm cảnh-báo thu-hồi. | Định-hướng sản-phẩm + kỳ-vọng pháp-lý. |
| **Q-B** | **Padding kích-thước Envelope** cho docType tùy-thân (chống suy-loại)? | Bật cho tùy-thân, tắt phần còn lại. | Đánh-đổi dung-lượng LampNet vs riêng-tư. |
| **Q-C** | **Per-context ref alias** (giống external-nullifier)? | Làm — chống hai bên nhận đối-chiếu "cùng hồ-sơ". | Tăng riêng-tư vs phức-tạp SDK. |
| **Q-D** | **Đường issued cho DocumentClaim** (cơ-quan ký ảnh) phase này hay sau? | Phase sau (sau lớp tài-liệu cơ-bản). | Lộ-trình tích-hợp cơ-quan. |
| **Q-E** | **Nơi chạy Spectra/Glint cho ảnh nhạy-cảm** (on-device vs TEE)? | On-device cho tùy-thân (không phụ-thuộc contract VeData). | Tránh vô-tình gửi plaintext ra ngoài. |

---

## 7. Ghi trung-thực (không over-claim)

- **Mức 1+2 là lớp lõi** (code + demo `/vc`). Lớp tài-liệu có *nền* trong `dossier`/`fingerprint`; phần tiết-lộ-chọn-lọc-tài-liệu + Mức 3 + Query là **spec** — ranh-giới: không `zk/`, không `BBS`/`ZkCredential` trong `src/`.
- → Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)
- **Giới-hạn thu-hồi ảnh (R1)** là cố-hữu, **không vá triệt-để được** — không hứa chống được. Watermark = truy-vết, không phòng-ngừa.
- **KHÔNG dedup/uniqueness/sybil PersonDID** trong Knowme (chủ-trương dự-án). Sinh-trắc Secure Enclave là đủ.
- **VeData sở-hữu catalog VC + issuer + Stamp/Query/Glint/Spectra.** Knowme dẫn-chiếu; một phần tích-hợp phụ-thuộc hợp-đồng liên-module.
- **Custody/recovery/anchor DID** thuộc module khác (Anchorme/Rebirthme) — Knowme chỉ dẫn-chiếu.

---

## Nguồn

- Nguồn thiết-kế nội-bộ (không công khai): Knowme-Feat-Math, KYC-KYB-ZK-Feat-Math, Midnight-Privacy-Feat-DRAFT; VeData-Reply-did-commit-resolve, VeData-Reply-shielded-address.
- Code: `PhoenixKey-Frontend/src/lib/sdvc/*`, `src/app/verify/page.tsx`.
- Tài-liệu cùng bộ: `PhoenixKey-Knowme-Vi-Feat.md`, `PhoenixKey-Knowme-Math.md`, `PhoenixKey-Knowme-Tech.md`.
