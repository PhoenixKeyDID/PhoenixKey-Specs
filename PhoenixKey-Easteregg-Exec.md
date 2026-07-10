# PhoenixKey — Easteregg · Đặc-tả ĐIỀU-HÀNH (Executive)

> Module: **Easteregg** (các mức riêng-tư của ví Phoenix — KHÔNG phải loại ví riêng) · Loại doc: Điều-hành (cho lãnh đạo) · Ngày: 2026-07-09
> Đối tượng đọc: lãnh đạo — quyết định + rủi ro + lộ trình. Tài-liệu cùng bộ:
> [PhoenixKey-Easteregg-Vi-Feat.md](./PhoenixKey-Easteregg-Vi-Feat.md) ·
> [PhoenixKey-Easteregg-Math.md](./PhoenixKey-Easteregg-Math.md) ·
> [PhoenixKey-Easteregg-Tech.md](./PhoenixKey-Easteregg-Tech.md).
> Bản này KHÔNG khoe sẵn sàng — điều-kiện-tiên-quyết chặn production ở §4.
>
> **Quyết-định maintainer (2026-07-09):** ví hệ-thống CHỈ 2 LOẠI — `Phoenix`/`Standard`
> (`PhoenixKey-Rebirthme-Exec.md`). Easteregg KHÔNG phải "module ví thứ ba" — đây là **các mức
> ẩn-danh mà ví Phoenix có thể mang** (mô hình 2 tầng: loại ví × mức riêng-tư). "Tầng 0" dưới đây
> = **L3** (`did_subaddr`) trong hệ địa-chỉ Phoenix L1/L2/L3 — cùng một validator, khác tên gọi
> theo ngữ-cảnh tài-liệu. `did_pool` (Tầng 1) là pool kế-toán dùng chung, KHÔNG phải một loại
> địa-chỉ hay loại ví.
>
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 1. Tóm-tắt một trang

**Easteregg là gì.** Kho tài sản **riêng-tư gắn khoá DID** của PhoenixKey. Đối xứng: **PhoenixKey
khôi phục KHOÁ · Easteregg khôi phục TÀI-SẢN của khoá**. Che số dư khỏi explorer công khai bằng
**bằng-chứng mật-mã** (không phải obscurity), mà tài sản vẫn tự đi theo DID khi rotate/recovery.

**Vấn đề nó giải.** Trên Cardano mọi số dư/dòng tiền công khai — với doanh nghiệp là rò rỉ bí mật
kinh doanh. Easteregg cho **ẩn tồn dư + mở-theo-yêu-cầu** cho ngân hàng/kiểm toán, tất cả **trên
Cardano L1** — không chain riêng, không bridge.

**Kiến trúc: một tập mức riêng-tư của ví Phoenix, bốn tầng (không chọn-một-bỏ-một, KHÔNG phải
loại ví riêng).** Các "phiên bản" cũ (Egg của VeData, Shielded-Custody, sub-address) KHÔNG loại
nhau — chúng là **các tầng riêng-tư áp lên CÙNG một ví Phoenix**, đánh đổi khác nhau:
- **Tầng 0 — địa chỉ riêng** (`did_subaddr`/eAddr — = **địa chỉ L3** trong hệ L1/L2/L3 của ví Phoenix, xem `PhoenixKey-Rebirthme-Exec.md`): người ngoài không nhóm được địa chỉ nhận về một chủ.
- **Tầng 1 — custody lõi (MẶC ĐỊNH):** Merkle-Sum-Tree pool + chữ ký authority DID. Ẩn số dư,
  khôi phục-theo-DID, **chi phí cận 0** (rút = 1 chữ ký, KHÔNG ZK).
- **Tầng 2 — ẩn dòng tiền (opt-in):** overlay ZK (commitment + nullifier + Groth16) trên cùng pool.
  Ẩn ghép cặp nạp↔rút + operator-blind. Trên L1 → **bỏ phụ thuộc Midnight**.
- **Tầng 3 — mở cho kiểm toán:** viewing-key grant + proof-only ≥ ngưỡng (Groth16). Dùng chung 2 tầng.

**Vì sao Tầng 1 là mặc định.** Đạt cả 6 tiêu chí — đặc biệt **chi-phí-~0** và **recovery-theo-khoá**:
tài sản do DID authority kiểm soát ⇒ sống qua rotate/recovery, user KHÔNG giữ thêm secret nào. Tầng 2
phục vụ privacy-tối-đa, đổi lấy chi phí Groth16 + recovery khó hơn — không bắt buộc.

**Tái dùng, không viết lại.** Easteregg **consume** primitive: MST/MMR (LampNet/Strata), ZK circuit +
prover + verifier (VeData/Glint), gated-read (VeData Query). PhoenixKey chỉ giữ tiền + địa chỉ +
recovery + UX. **VeData/Glint KHÔNG giữ custody/tiền.**

**Trung thực về mức bảo-đảm.** Mọi bảo đảm an-toàn (SHIELD-1..9, no-backdoor) đúng theo THIẾT-KẾ,
có hiệu lực khi validator cài đúng bất-biến (§4 nêu điều-kiện-tiên-quyết trước khi bật production
từng tầng). PoC Python off-chain minh-hoạ nguyên-lý ẩn-số-dư + gated-proof trên Preview — không
thay thế validator, chưa chứng-minh operator-không-rút. **KHÔNG over-claim.**

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 2. Bảng quyết-định (quyết | lý-do 4-trục | đánh-đổi)

4 trục: (a) định hướng dài hạn · (b) first-principles · (c) tối ưu eUTXO/ExUnit/phí/đơn giản ·
(d) lợi ích user + bền vững.

| Quyết định | Lý do (4 trục) | Đánh-đổi |
|---|---|---|
| **(a) Phân TẦNG, không chọn-một** | (b) mỗi phiên-bản là điểm đánh-đổi hợp lệ, ép-một = mất use-case; (d) user chọn mức riêng-tư như chọn ổ khoá; (a) một kiến trúc nhất quán mở đường nâng cấp | Module 4 tầng phức tạp; phải giữ ranh giới tầng sạch (I-EGG-23) |
| **(b) Tầng 1 (MST) làm MẶC ĐỊNH** | (c) rút = 1 chữ ký + Merkle-Sum proof, KHÔNG Groth16/tx phụ → chi phí cận 0; (d) recovery-theo-DID; (b) ẩn tồn dư bằng sum-invariant (mật mã) | Ẩn STOCK không ẩn FLOW; operator biết plaintext số dư (privacy vs explorer, KHÔNG vs operator) |
| **(c) Tầng 2 ZK trên L1, BỎ Midnight khỏi critical-path** | (b) flow-privacy trên chính L1 → không bridge/relayer-điểm-biết-hết/wCARP-IOU; (a) không nợ kỹ-thuật cầu bán-tin-cậy; (c) Groth16 đo thật 2.842B < trần 10B | Cần ceremony phase-2 + verifier Aiken tự-audit; recovery Tầng 2 khó |
| **(d) Tái dùng primitive toàn hệ** | (c) không dựng lại bánh xe → ít bề mặt tấn công + rẻ; (a) mỗi đội giữ chuyên môn; (b) ranh giới sạch: VeData cấp circuit KHÔNG giữ tiền | Phụ thuộc tiến độ Strata/Glint/Query; cần đồng bộ liên đội |
| **(e) Recovery là trục phân tầng, nêu trade-off THẲNG** | (d) Tầng 1 tài sản đi theo khoá = điểm bán; Tầng 2 nghịch-lý recover⟺backdoor (single-party-derive = drain, CẤM) → user CHỌN self-custody vs k-of-n | Tầng 2 mất secret tier-1 = mất tiền; phải hiển thị trade-off trước opt-in |
| **(f) Feecover KHÔNG phục vụ M2-withdraw** | (b) Feecover resolve DID point-in-time ⇒ lộ DID, phá unlinkability vừa mua bằng Groth16; (c) thay bằng relayer-lane fee rời-rạc bind-in-PI | M2-withdraw: user tự trả hoặc relayer-lane (có thể do PhoenixKey vận hành) |

---

## 3. Ma-trận rủi-ro

| Rủi ro | Mức | Cơ chế / giảm thiểu | Điều-kiện đóng lỗ |
|---|---|---|---|
| 🔴 **Sweep ghi lệch phân-bổ (G3 / VG-E1)** | **CAO** | Pseudocode Sweep chỉ ràng TỔNG, chưa ràng per-pair (DID_i↔amount_i). Crank permissionless ác-ý ghi lệch giữa các DID cùng batch | Zero-sum (tiền không mất) nhưng tranh-chấp-phân-bổ nếu hở. **Chặn-merge Sweep** — cài per-pair (I-EGG-4') NGAY từ bản đầu, không "sửa sau" |
| 🔴 **Withdraw không cấn được phí (G1 / VG-E2)** | **CAO** | User chỉ có eADA/LAMP trong pool, không ADA rời trả phí L1 → không tự rút nếu thiếu fee-split/FEE_FLOOR | Cần fee-split + FEE_FLOOR + nối Feecover. Phụ thuộc `fee_asset` (LAMP cần oracle / CARP ổn hơn — xem D6) |
| 🔴 **Mất witness-data (salt) → kẹt tiền (G5 / VG-E4)** | **CAO** | Salt off-chain (`leaves.json`) mất → không dựng được proof dù tiền còn nguyên trong pool | Vá: salt-deterministic từ Master_KEK (tái sinh) + multi-indexer, không phụ thuộc một bản sao duy nhất |
| 🔴 **In-tiền-ảo chưa cưỡng-chế (VG-E5)** | (thiết-kế chặn) | SHIELD-1 sum-invariant + solvency chặn CỨNG theo thiết-kế | Cưỡng-chế thật khi validator cài đúng I-EGG-1. Không quảng-cáo là đã cưỡng-chế trước đó |
| **Tầng 2: k đếm ≠ k' hiệu dụng (VG-E3)** | TRUNG | K_min=32 (sơ bộ) enforce on-chain chặn `k`, KHÔNG bảo đảm `k'` | Self-fill là residual kinh-tế/governance, không giải bằng crypto (OQ-EGG-5) |
| **Ẩn STOCK không ẩn FLOW (Tầng 1, VG-E7)** | TRUNG | Ranh giới vật lý L1: timing + amount mỗi khoản NHẬN lộ trước sweep | Che flow ≈ 0 khi thưa. Muốn ẩn flow → Tầng 2 |
| **Operator biết plaintext số dư** | TRUNG | Indexer giữ mọi lá. Privacy vs explorer, KHÔNG vs operator | Giảm nhẹ khả-thi: mã-hoá-lá + multi-indexer. Muốn operator-blind → Tầng 2 |
| **Tầng 2: circuit/ceremony** | (chặn Tầng 2) | Groth16 CHỐT (2.842B < 10B), N_batch=3; verifier Aiken TỰ VIẾT + AUDIT | Cần ceremony PPoT+phase-2 + uplc-Conway-confirm trước khi mở Tầng 2 |
| **Recovery Tầng 2 backdoor (VG-E8)** | THẤP 🟢 | G.3 CẤM single-party-derive sk; k≥2 + time-lock + veto | Điểm mạnh nhất module (thiết-kế cấm) |
| **Viewing-key leak (VG-E9)** | THẤP 🟢/🟡 | viewing⊥spending (I-EGG-20) → không drain | Hở privacy quá-khứ (khoá đã trao); audit-key threshold treo (OQ-EGG-9) |
| **Anchor-giả PersonDID (VG-E10)** | (kế thừa) | Lỗ MÃ-HOÁ anchor (kế thừa toàn hệ) — đóng bởi UniquenessThread PA2 | Chờ đội khác (ngoài Easteregg) |

---

## 4. ⚠ ĐIỀU-KIỆN-TIÊN-QUYẾT / blocker chặn production

> **KHÔNG bật production tới khi các blocker dưới land.** Bảng dưới là điều-kiện bắt buộc,
> không phải hạng mục tuỳ-chọn.

### 4a. Blocker Tầng 1 (mặc định — build trước)

| # | Blocker | Chủ | Ảnh hưởng nếu thiếu |
|---|---|---|---|
| **B1** | `did_pool` validator (I-EGG-1..9, Aiken v1.1.21) | đội on-chain | Không có lõi giữ tiền — toàn Tầng 1 treo |
| **B2** | G3 Sweep per-pair binding (I-EGG-4') | đội on-chain | Crank permissionless ghi lệch. **Chặn-merge Sweep** |
| **B3** | G1 Withdraw fee-split + FEE_FLOOR + nối Feecover | đội on-chain+backend + chốt `fee_asset` | User không tự rút được nếu ví rỗng ADA |
| **B4** | Indexer/Accountant + sweep crank + withdraw builder (Rust, Strata) | đội backend | Không dựng được cây MST / proof / gated-read |
| **B5** | G5 witness-data recovery (salt-from-Master_KEK) | đội backend + xác nhận đội on-chain | Mất salt off-chain = kẹt tiền (đã xảy ra PoC) |

### 4b. Blocker Tầng 0 + Tầng 2 (sau)

| # | Blocker | Chủ | Ghi chú |
|---|---|---|---|
| **B6** | `did_subaddr.ak` (eAddr unlinkable, DEP-2) | đội on-chain | Trước khi có: MVP "riêng-tư mức-resolver" |
| **B7** | Tầng 2: verifier Aiken TỰ VIẾT + AUDIT + ceremony PPoT/phase-2 MPC ≥2 bên | VeData/Glint | KHÔNG dùng thẳng scratch; circuit đóng-băng sớm |
| **B8** | uplc-Conway-confirm — xác nhận 2.842B trên params mainnet thật | VeData | Rủi ro <5%; chốt trước lock N_batch |
| **B9** | OQ-EGG-5/8/9 — ước `k'`, đo 5 hạng-mục non-ZK, threshold-decrypt | VeData | Còn mở |

**Hệ quả điều hành:** build Tầng 1 (B1–B5) trước + test Preview 7-bước, Tầng 0 (B6) + Tầng 2 (B7–B9)
sau. KHÔNG tuyên bố production Tầng nào trước khi blocker của tầng đó land + evidence tx-hash thật.

---

## 5. Luật GO/NO-GO theo tầng

**Nguyên tắc:** không tầng nào được tuyên bố production trước khi điều-kiện-tiên-quyết của tầng đó
(§4) land + evidence tx-hash thật + red-team pass (7 bước, xem Tech §9). Thứ tự bắt buộc: build+test
Preview **Tầng 1 + Tầng 3 chế-độ-1** trước (phasing P1), rồi mới tới Tầng 0, cuối cùng Tầng 2.

→ Trạng-thái & tiến-độ hiện tại theo từng hạng-mục (spec/code/test/PoC/ZK):
[PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 6. Lộ-trình

| Giai đoạn | Nội dung | Điều kiện vào |
|---|---|---|
| **P0 — Chốt chiến lược** | Maintainer duyệt spec hợp nhất làm canonical + 5 gap addendum + Mode Matrix | Spec draft (xong) |
| **P1 — Build Tầng 1** | `did_pool` (B1+B2) + indexer/sweep/withdraw (B4) + fee-split (B3) + salt-recovery (B5) + viewing-key (Tầng 3 chế-độ-1) | Maintainer duyệt + đội on-chain xác nhận G3 |
| **P1-test** | 7 bước Preview + red-team 4 mục fail-closed (evidence tx-hash thật) | P1 code xong |
| **P1.5 — Tầng 0** | `did_subaddr.ak` eAddr unlinkable (B6) | Maintainer duyệt DEP-2 |
| **P2 — Tầng 2 (opt-in)** | pool ZK deposit/withdraw + proof-only + association-set | B7+B8+B9 land → audit verifier → preprod E2E (double-withdraw REJECT, solvency, K_min) |
| **P3** | sybil-hardening (fee luỹ tiến), token-agnostic, threshold audit-lane | OQ-EGG-5/7/9 |

---

## 7. Câu hỏi cần LÃNH-ĐẠO chốt

| # | Quyết định | Gợi ý |
|---|---|---|
| **D1** | Duyệt spec hợp nhất Easteregg làm canonical (PEER Protectme/Feecover/Knowme)? | — |
| **D2** | Phase bật **Tầng 2 opt-in** (quyền PhoenixKey) | Sau khi P1 pass Preview + audit verifier; không block P1 |
| **D3** | Duyệt **DEP-2 `did_subaddr.ak`** để đội on-chain code eAddr unlinkable? | Chốt cùng đợt did_pool để test chung |
| **D4** | Recovery-tier **default Tầng 2** (self-custody vs k-of-n) | Đề xuất: default k-of-n (PhoenixKey=1-of-n) khớp "không bắt user giữ secret"; tier-1 cho power-user |
| **D5** | 5 gap addendum: duyệt G1🔴+G3🔴+G5🔴 ưu tiên → giao đội on-chain/backend? | G3 chặn-merge Sweep; G1 chặn user tự-rút; G5 chống kẹt tiền |
| **D6** | `fee_asset` cho Withdraw = LAMP (cần oracle) hay CARP (ổn hơn)? | Đề xuất CARP (neo 1 CARP=1 MAGIC) hoặc LAMP biên an-toàn 1.2× |
| **D7** | I-EGG-26 FEECOVER-BOUNDARY (Feecover không phục vụ M2-withdraw) — báo team MAGIC/Feecover một dòng? | Duyệt |
| **D8** | Khung External-Adapter (4 bất biến + cửa quyết định EXT-3 Midnight ngủ-đông)? | Duyệt để đóng tranh luận "có cần Midnight không" |

---

## 8. Ghi trung-thực (không over-claim)

- Mọi bảo đảm an-toàn (SHIELD-1..9, no-backdoor) đúng theo THIẾT-KẾ, có hiệu lực khi validator cài
  đúng bất-biến (§4). Mọi verdict "chặn" là điều-kiện trên đó — không phải trạng-thái đã đạt.
- **Số 2.842B ExUnit là đo THẬT của circuit VeData**, độc lập với tiến-độ verifier Aiken phía
  PhoenixKey (hạng-mục riêng, xem B7 §4b).
- PoC Preview (3 tx-hash) minh-hoạ nguyên-lý ẩn-số-dư + gated-proof, **KHÔNG** minh-hoạ operator-không-rút
  (không thay validator). "PoC ≠ sản phẩm".
- 4 lỗ 🔴 (G1 phí, G3 sweep-bind, G5 salt-kẹt-tiền, in-tiền-ảo chưa-cưỡng-chế) tồn tại vì bảo-đảm
  NẰM TRÊN GIẤY cho tới khi validator cài đúng — không vì thiết-kế sai. G3 là **điều-kiện-tiên-quyết
  build** (cài per-pair NGAY, không "sửa sau").

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## Nguồn

- Gộp từ nguồn thiết-kế nội-bộ (không công khai).
  Feecover-boundary + External-Adapter + quyết định, 5 gap, 5 vector nguy nhất + GO/NO-GO
  + VG-E1..E10.
