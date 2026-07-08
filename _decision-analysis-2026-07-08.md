# Phân-tích quyết-định — cho anh Aladin chốt (2026-07-08)

> Nền: 2 gaming-eval (Protectme 6🔴 / Easteregg 5🔴) + finding code Protectme (beacon 0-dòng,
> escrow lệch B.6) + trạng-thái validator (Activation 173/173, SRCL vá F1/F2). Mỗi mục:
> **đề-xuất chốt** + lý-do 4-trục (a dài-hạn · b first-principles · c tối-ưu · d user+bền)
> + đánh-đổi. Đây là khuyến-nghị — anh quyết.

---

## A. MERGE PR#16 / #17 / #18 — thứ-tự + cổng

**Bản chất khác nhau:** #16/#17 = **spec docs** (Specs repo, rủi-ro thấp), #18 = **validator code** (money-critical).

| PR | Nội dung | Đề-xuất | Cổng bắt buộc |
|---|---|---|---|
| **#18** (Validator, Activation v4.1) | code money-critical, Tuân đã bắt double-sat, vá `82f05d2` (173/173) | **MERGE khi Tuân ✅ lại vá double-sat** | Tuân xác-nhận `82f05d2` + count==1 guard đủ. Config-clock (task-6) là **follow-up riêng, KHÔNG chặn #18** |
| **#16** (Specs Activation 4-doc + bump 173) | docs | **MERGE sau khi anh đọc** — rủi-ro thấp | anh liếc bộ 4-doc; không cần cổng kỹ-thuật |
| **#17** (Specs Feecover/Protectme 4-doc) | docs | **MERGE sau khi anh đọc** | như trên |

**Lý-do (4 trục):** (a) tách docs↔code để docs không kẹt theo review code; (c) #18 merge sớm giải-phóng Tuân sang PA2/entity-gate; (b) code money-critical PHẢI có ✅ người-review độc-lập trước merge — KHÔNG tự merge #18. (d) docs merge sớm cho Long/VeData có nguồn làm việc.

**Chốt đề-xuất:** merge #16/#17 ngay khi anh đọc xong; #18 chỉ merge sau Tuân re-approve vá double-sat. Thứ-tự không ràng buộc (độc-lập).

---

## B. PHẦN-D PROTECTME — 11 quyết-định

Nền mới: gaming-eval xác-nhận **NO-GO mọi loại DID** + code thật (payout sạch 39 test, **beacon 0-dòng**, **escrow lệch B.6**). Đề-xuất từng QĐ:

### 3 🔴 van-pot-wide (chốt KỸ nhất — quyết định sống-chết pot)

**PROT-10 🔴 — Evidence bar gán-SYS + per-incident cap.**
- **Đề-xuất:** (1) Nhãn SYSTEM_FLAW CHỈ hợp-lệ khi có **reproducer độc-lập** (≥2 bên tái-hiện) + postmortem ký + phạm-vi khớp on-chain log; (2) **CAP_INCIDENT_SYS cứng** (trần tuyệt-đối mỗi sự-cố, vd ≤ X% pot_system/incident) + **pro-rata** nếu vượt; (3) **vẫn soi từng-user trong batch** (không auto-cover cả cohort); (4) challenge-window RIÊNG cho nhãn SYS.
- **4 trục:** (b) SYS được phủ-mạnh-không-co-pay ⇒ nhãn sai = khuếch-đại toàn-pot ⇒ phải chặn ở NGUỒN (evidence bar) chứ không ở trần; (c) cap+pro-rata là backstop toán-học một-nhãn-sai-không-thành-thảm-hoạ; (d) minh-bạch tiêu-chí để user tin.
- **Đánh-đổi:** SYS thật cũng phải chờ reproducer → payout chậm hơn. Chấp-nhận (đúng: SYS hiếm, giá-trị lớn, cần chắc).

**PROT-11 🔴 — Cohort single-factor.**
- **Đề-xuất:** (1) **BẮT BUỘC anti-drain (time-buffer) làm điều-kiện phủ USER** cho DID single-factor — đây là lớp thật DUY NHẤT còn lại khi guardian/secondary cùng-seed; (2) chiết-khấu phí CHỈ cho factor **độc-lập-với-seed** (đọc I-CURVE-5); (3) `CAP_USER_SINGLE` thấp + `single_factor_loading` (phụ-phí rủi-ro).
- **4 trục:** (b) guardian cùng-seed sập ĐỒNG-THỜI khi lộ seed ⇒ "nhiều lớp" là ảo ⇒ định-giá theo **độc-lập** không theo **hiện-diện**; (d) nông-dân/hộ-một-người (PersonDID) là nhóm rủi-ro cao nhất — bắt anti-drain bảo vệ họ thật, không bán ảo-tưởng.
- **Đánh-đổi:** solo-một-thiết-bị phí cao hơn + buộc bật anti-drain. Đúng bản-chất rủi-ro.

**PROT-4 🔴 — Ngưỡng SYS vs USER.**
- **Đề-xuất:** gắn CHẶT với PROT-10 (cùng evidence bar). Mặc-định ca GREY xử **bảo-thủ về USER** (co-pay + trần thấp), user kháng lên SYS phải tự đưa reproducer. Nâng GREY→SYS chỉ qua committee + challenge.
- **4 trục:** (b) on-chain KHÔNG phân-biệt trộm-thật vs ví-mình ⇒ mặc-định-USER an-toàn-đóng; (d) chống user ép ca của mình vào nhãn phủ-mạnh (gaming VG-P4).

### 8 QĐ còn lại (đề-xuất gọn)

| QĐ | Đề-xuất | Trục chính |
|---|---|---|
| PROT-1 trần CAP_SYS/CAP_USER | CAP_USER **thấp** (phủ hạn-chế), CAP_SYS cao hơn nhưng ≤ CAP_INCIDENT_SYS. Tỷ-lệ theo loại DID: Org/Service cao hơn Person (rủi-ro self-theft thấp hơn) | d bền-pot |
| PROT-2 ai phân-xử | Committee **chọn-ngẫu-nhiên/luân-phiên** + recuse + vote on-chain **VP-PersonDID không-token-weighted** | a governance-gốc |
| PROT-3 tỷ-lệ phí | `base_rate` 0.5–1%/năm (khởi thấp, hiệu-chỉnh theo tỷ-lệ-chi-trả thật); co-pay 30–50% USER; deductible có | d bền |
| PROT-5 thời-gian | `T_report` 30d, `T_wait_activate` 30d (chống mua-sau-khi-trộm), challenge 14d (SYS 14d riêng) | b anti-adverse |
| PROT-6 circuit-breaker | `POOL_CAP_PER_WINDOW` + pro-rata + **no-cross-bucket** (đã chốt). Escrow-vs-Feecover B.6 chốt cùng finding code (dưới) | c an-toàn |
| PROT-7 opt-in vs bundle | **Bundle mức-nền mặc-định cho MỌI user** + cap-cao opt-in-có-waiting → chống xoáy-tử-thần | d sống-chết-pot |
| PROT-8 subrogation | Thu-hồi về **pot** (không chia user) — user đã được đền, không lời gấp-đôi | b |
| PROT-9 KYC claim lớn | KYC-nhẹ chỉ claim USER **lớn** (ngưỡng cao); cân riêng-tư + không-token-weighted. Claim nhỏ + SYS không KYC | d cân-riêng-tư |

**Chốt tổng Protectme:** 3🔴 nên chốt TRƯỚC (khoá van-pot-wide) rồi mới build. Ngay cả khi chốt hết, **vẫn NO-GO production** tới khi beacon + escrow vá (finding code dưới) + MAGIC-model/CARP land.

---

## C. FINDING CODE PROTECTME → Issue (đã anh duyệt tạo)

2 lỗ code thật (agent verify) — tách khỏi 11-QĐ-chính-sách:
1. **beacon 0-dòng** (`protectme_beacon.ak` rỗng mọi branch) → double-payout không chặn on-chain → **định-lý solvency VỠ nếu nạp trùng `claim_id`**. Van chặn-merge, Tuân đóng-được-ngay.
2. **escrow-nạp lệch spec B.6**: spec = release-from-bucket; code = escrow-tự-giữ (chỉ kiểm `escrow_carp==amount`, không kiểm nguồn) → gốc lỗ escrow-giả.

→ Issue giao **Tuân (beacon) + Long (escrow wire)**, dán link gaming-eval + code lines.

---

## D. reserve_dest (đích Treasury cho value dự-trữ/thu-hồi)

**Bối cảnh:** LAMP **cố-định 36 tỷ, KHÔNG burn** — giảm lưu-hành = chuyển vào Treasury (kế-toán). Nhiều đường value về Treasury: Activation forfeit/reclaim→pot, Feecover epoch-settle→Phoenix Treasury, Protectme subrogation→pot. `reserve_dest` = **địa-chỉ hiến-định** nhận các dòng này.

**Đề-xuất:**
- **KHÔNG hard-code một khoá đơn.** `reserve_dest` phải là **script multisig governance-gated** (rút chỉ qua Governance `status==Executed`), KHÔNG phải PKH cá-nhân.
- **Tách Treasury theo chức-năng** (đã có tiền-lệ): Phoenix Treasury (Feecover CARP) ĐỘC-LẬP pot Activation (LAMP) ĐỘC-LẬP pot Protectme (2 bucket). reserve_dest KHÔNG gộp-một — mỗi dòng về đúng sổ (no-cross-bucket).
- **Neo tạm bằng placeholder governance-multisig** tới khi chốt **Foundation (pháp-nhân/DAO)** — vì chủ-thể-pháp-lý của Treasury là quyết-định pháp-lý, không chỉ kỹ-thuật.
- **4 trục:** (a) Treasury là tài-sản dài-hạn của hệ ⇒ không giao khoá đơn; (b) rút-Treasury phải qua governance-người (VP-PersonDID) không token-weighted; (d) minh-bạch + bền.

**Đánh-đổi:** chưa chốt Foundation ⇒ reserve_dest tạm placeholder ⇒ chưa mainnet-deploy phần đụng Treasury. Chấp-nhận (mainnet đằng nào cũng HOÃN chờ audit).

---

## E. GA WITHDRAW TREASURY — khung đề-xuất

**Mục-đích:** rút LAMP/tài-nguyên từ Treasury cấp cho các gói tạo giá-trị. Nền: `_work-packages-status.md §G`.

**Đề-xuất khung GA (4 hạng-mục xin, ưu-tiên):**
1. **Audit độc-lập** (bắt-buộc trước mainnet) — registry/supply_state/activation/did_payment/srcl. Tốn phí thật. **Ưu-tiên-1** (chặn mọi mainnet).
2. **Deploy + vận-hành hạ-tầng** — Blockfrost key, indexer, host, funding min-UTXO DID mới (mainnet không faucet).
3. **Launch pots** — SRCL GreenSun 360M LAMP · ETD/Airdrop. Nguồn từ quỹ Distribution/Reserve.
4. **Đội build còn lại** — Long/Tuân/VeData/MAGIC-CARP engine.

**KHÔNG đưa vào GA (chưa qua cổng):** mainnet-deploy-ngay, GetLAMP-PersonDID-production (chờ PA2+Registry đóng GV1/GV2).

**4 trục:** (a) audit-trước-mainnet là điều-kiện-sống-còn uy-tín; (c) xin đúng gói tạo-giá-trị-đo-được, không xin dàn-trải; (d) launch pots nuôi cộng-đồng SPO/delegator bền.

**Cần anh chốt trước khi viết GA:** (1) reserve_dest/Foundation (mục D) — vì GA rút TỪ Treasury cần biết chủ-thể; (2) ngân-sách audit ước-lượng; (3) nguồn LAMP cho launch pots (quỹ nào).

---

## Thứ-tự đề-xuất anh xử

1. **Đọc + duyệt merge** #16/#17 (docs, nhanh); Tuân re-approve → merge #18.
2. **Chốt 3🔴 Protectme** (PROT-10/11/4) — khoá van-pot-wide, chặn build sai.
3. **Chốt reserve_dest = governance-multisig + quyết Foundation** — mở đường GA.
4. **Viết GA Withdraw** theo khung E sau khi có (2)+(3).
8 QĐ Protectme còn lại + PROT lẻ có thể chốt cùng đợt build.
