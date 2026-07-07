# Đánh giá GAMING (mechanism-design / adversary-tokenomics) — Feecover · 2026-07-07

> **Phạm vi:** săn cách khai-thác **cơ-chế phí Feecover** để RÚT giá-trị / NÉ phí / PHÁ kế-toán. Đây là lỗ **kinh-tế + kế-toán**, tách khỏi red-team money-drain on-chain (đã sạch ở tầng consume — C-CM-1..5).
>
> **Nguồn đã ĐỌC + verify (không bịa, neo file:dòng):**
> - `spec-proposals/PhoenixKey-Feecover-Feat-Math.md` (§1–§10) + `PhoenixKey-Feecover-Excu.md` (Excu: MERGED #14, CODE CHƯA build, 3 blocker B1/B2/B3).
> - **HAI bản CODE ConsumeMAGIC (PHÂN KỲ — xem FG-1):**
>   - **CANONICAL (rewrite D1, PR#13):** `MAGIC/ConsumeMAGIC/onchain/validators/consume.ak` + `.../lib/magiclamp/consume/{types,pricing}.ak`. Mô hình ĐÚNG khớp spec: **KHÔNG mint**, ConsumeMAGIC = lớp PRICING+ENGAGEMENT, MAGIC giảm ở **BurnBatch của VAULT validator** riêng; `total_burned == required` (**==**); `EngageDatum` CÓ field `did_commit` (immutable, MVP `#""`).
>   - **CŨ/SUPERSEDED (token-burn):** `MAGIC-paymaster/ConsumeMAGIC/onchain/validators/consume.ak` + `.../types.ak`. Mint âm MAGIC (`check_only_magic_burn`), `magic_burned >= required` (**≥**, over-burn OK), `EngageDatum` KHÔNG có did_commit. Bản này TRÁI spec — bị PR#13 thay.
>   - `MAGIC/ConsumeMAGIC/.../price_param.ak` (ai post giá + demand_mult).
> - **Quỹ phí đang tồn tại (trùng tên/khác model):** `MAGIC/FeeTreasury/onchain/validators/fee_treasury.ak` (5 redeemer FX-fund ADA, operator-cosign).
> - `spec-proposals/_msg-to-CARP-policy-id.md` (CARP policy-id CHƯA chốt preprod — B2).
> - `spec-proposals/VeData-Reply-did-commit-resolve.md` (did_commit nội-dung chưa land — B3).
> - Memory: [[anchor-uniqueness-limit]], [[two-lamp-policies-tension]], [[magic-not-native-token]], [[activation-getlamp-model]].
>
> **grep sự-thật:** `ServiceFeeSchedule`/`FeecoverGate`/`FeecoverAccrual`/`carp_paid`/`fee_magic` = **0 dòng code** — Feecover thuần spec (Excu tự xác nhận "chưa có mã Feecover nào"). `did_commit` CÓ field ở bản CANONICAL (`types.ak:50`) nhưng nội-dung = sentinel `#""`, validator chỉ ép immutable, KHÔNG ràng nội-dung (MVP). `fee_treasury.ak` tồn tại nhưng model KHÁC (FX-fund ADA, không CARP-accrual per-DID).

---

## Kết luận đầu trang — 5 vector nguy nhất

**Lỗ THẬT còn hở ở Feecover (xếp theo mức độ phá-hệ):**

1. **🔴 FG-4 — EpochSettle + provider-vault = TIN-CẬY thuần, KHÔNG cơ-chế ép trung-thực.** §6.2 `epoch_settle` chỉ có pseudo-code; **0 validator**. "Cuối epoch chuyển CARP về Treasury" dựa vào provider (App→Platform) TỰ chạy settle. Không ai on-chain ÉP: (a) provider phải settle (có thể ngâm CARP vô hạn, ăn yield/thanh-khoản); (b) `total_carp` khai đúng Σ accrual (skim phần chênh); (c) chống provider tự-đổi `stabilizer_ref`/địa-chỉ-Treasury. So-chiếu `fee_treasury.ak:108-118` (đã tồn tại): `Withdraw` chỉ cần **`operator` cosign** + trên floor → **operator rút quỹ tự do trên floor**. Nếu Feecover settle mượn khuôn này → provider-operator rút CARP trước khi tới Treasury. Đây là lỗ **kế-toán** đúng câu hỏi anh — và là lỗ HỞ NẶNG NHẤT còn nguyên (không blocker đội khác che).

2. **🔴 FG-3 — FeecoverGate + attribution per-DID chưa thực-thi (`did_commit` nội-dung = sentinel `#""`).** Bản CANONICAL CÓ field `did_commit` (`types.ak:50`) + ép **immutable** qua consume (`consume.ak:119`, `enforce_engagement`), NHƯNG nội-dung MVP = `#""` và validator **KHÔNG ràng nội-dung** (chú thích `types.ak:44` "KHÔNG ràng buộc NỘI DUNG ở MVP"). Blocker B3 (Excu §4). Nên: (a) FeecoverGate ép "chỉ PhoenixKey DID" chỉ ràng được `owner` (khoá thô) — 1 người mở N account né mọi trần/attribution; (b) `FeecoverAccrual.breakdown` per-DID (§3.3, §6.1) là **hư cấu** tới khi did_commit chứa blake2b256(DID) thật + PhoenixKey resolve point-in-time land (`VeData-Reply:16-17` xác nhận phần point-in-time CHƯA implement). Gate hiện = **cổng owner-thô**.

3. **🔴 FG-4b/FG-8 — Trùng model với `fee_treasury.ak` đã deploy + settle chưa phân-định.** Đã tồn tại `MAGIC/FeeTreasury/onchain/validators/fee_treasury.ak` NORMATIVE trỏ "SPEC PhoenixKey-Fee-Abstraction" (tên CŨ trước Payyou→Feecover), nhưng là model FX-fund **ADA** (không CARP-accrual). Hai "fee treasury" cùng hệ, một deploy (ADA-FX operator-cosign) một spec (CARP-settle per-tier) → nếu Feecover-settle vô-tình kế-thừa Withdraw-operator-model của bản deploy → FG-4 hiện-thực-hoá. Ranh giới CARP-về-PhoenixKey-Treasury vs ADA-FX-fund-App CHƯA chốt.

4. **🟡 FG-5 — Re-anchor / demand_mult là đòn-bẩy quản-trị KHÔNG DAO-gated ở tầng beacon tái-dùng.** Spec §4.3 nói fee đổi "chỉ qua Governance `status==Executed`". Nhưng Feecover định-giá THỰC = `base_price × demand_mult / Q` (`pricing.ak:42-47`), và `demand_mult` do **committee M-of-N keeper** post (`price_param.ak:36-37`, `count_sigs >= threshold`) — **KHÔNG** đọc `status==Executed`, chỉ cần đủ chữ-ký keeper, và KHÔNG rào biên-độ-đổi-mỗi-epoch (khác `fee_treasury.ak:154` có band ±5%). Keeper đẩy `demand_mult` tới `m_max` (2× clamp) → **giá THẬT gấp đôi** mà KHÔNG qua DAO. Feecover ràng `carp_paid==fee_magic(service_id)` từ bảng tĩnh, nhưng MAGIC burn thật chạy theo `demand_mult` → **hai bảng lệch** (§8.3/D6 treo). Trục lợi hoặc phá kế-toán qua khe này (câu hỏi 6 ở tầng KEEPER, dễ hơn DAO).

5. **🟡 FG-1 — HAI bản ConsumeMAGIC phân-kỳ; spec neo mơ-hồ.** Có 2 copy: CANONICAL (`MAGIC/ConsumeMAGIC`, rewrite D1/PR#13 — no-mint, `==`, có did_commit, khớp spec) và CŨ (`MAGIC-paymaster/ConsumeMAGIC` — mint-burn native, `≥`, không did_commit, TRÁI spec). Feecover §0/§35 trỏ path `MAGIC-paymaster/ConsumeMAGIC` (bản CŨ). Rủi ro: dev/reviewer đọc nhầm bản CŨ → build Feecover trên model mint-burn/over-burn đã bỏ; hoặc B1 "MAGIC-model reconcile" (Excu §4) chưa dứt-điểm khiến 2 bản cùng sống. **Không phải lỗ gaming — là nợ-nguồn-chân-lý phải dọn trước khi build.**

**Cái Feecover/ConsumeMAGIC CHẶN TỐT (trung thực, bản CANONICAL):**
- **Stale-price + rollback (câu hỏi 1):** `consume.ak:73-75` ép `0 ≤ current_epoch − pp.epoch ≤ max_price_stale` + `price_param.ak` ép `epoch` **đơn-điệu-tăng** (chống rollback về giá thấp). Dùng bảng CŨ giá thấp → fail cả hai phía. **Stale-gate MẠNH** cho `PriceParam`; Feecover mượn khuôn cho `ServiceFeeSchedule` hợp lý — NHƯNG chỉ khi `ServiceFeeSchedule` được cấp validator tương tự (chưa có).
- **Client tự-khai giá + carp_paid==fee_magic (câu hỏi 2):** giá LẤY TỪ BEACON, không tin redeemer amount (`consume.ak:68-71`); và bản CANONICAL ép **`total_burned == required`** (`consume.ak:95`, **==** nghiêm, KHÔNG over-burn) qua liên-kết vault BurnBatch. Đây là **CHẶN mạnh hơn tôi lo ban đầu**: không over-burn, không multi-engage→single-vault under-charge (test `pay-once-consume-N` REJECT `consume.ak:719`). carp_paid==fee_magic THỰC-THI-ĐƯỢC nếu Feecover đọc chính `required`/`total_burned` này — miễn ServiceFeeSchedule.fee_magic khớp beacon (D6).
- **Drain Vault qua nhánh mới:** `engage_value_preserved` (`consume.ak:113`) ép engage-UTxO bảo toàn tuyệt đối (chỉ ADA+thread-NFT, KHÔNG chứa MAGIC/LAMP). CARP-fee đến từ input NGOÀI engage → không drain (§7-B đúng).
- **Double-satisfaction:** `#in==#out` + Σthread-NFT bảo toàn (`consume.ak:98-108`) + `fee_treasury.ak:53-59` FT-THREAD sẵn có → nếu settle mượn khuôn, chống double-settle được (nhưng CHƯA viết).

---

## Bảng tổng vector

| # | Vector | Mức | Chặn/Hở | Neo file:dòng | Vá |
|---|---|---|---|---|---|
| **FG-4** | EpochSettle/provider-vault = trust thuần, không ép settle/không chống skim | 🔴 **HỞ** | HỞ — settle chỉ pseudo-code (0 validator); Withdraw-operator sẵn có cho rút tự do | Feecover §6.2; `fee_treasury.ak:108-118` (Withdraw operator-cosign) | Settle validator ép: (a) forced-settle theo epoch (thread NFT + last_settled monotonic); (b) `total_carp == Σ accrual` on-chain (không tin provider khai, mẫu `fee_treasury.ak:63-66`); (c) đích Treasury + stabilizer_ref bất-biến trừ DAO; (d) KHÔNG mượn Withdraw-operator-model |
| **FG-3** | Gate DID + attribution per-DID chưa thực-thi (did_commit nội-dung `#""`) | 🔴 **HỞ** | HỞ — field có + immutable, nhưng nội-dung MVP rỗng; resolve point-in-time chưa land | `types.ak:44,50`; `consume.ak:119`; `VeData-Reply:16-17`; Feecover §5/§8.1; Excu B3 | Blocker cứng: KHÔNG bật FeecoverGate/breakdown production tới khi did_commit chứa blake2b256(DID) thật + PhoenixKey resolve point-in-time land |
| **FG-4b/FG-8** | Trùng model với `fee_treasury.ak` deploy (FX-fund ADA operator-cosign) | 🔴/🟡 | HỞ — 2 "fee treasury" cùng hệ khác bản-chất; ranh-giới chưa chốt | `MAGIC/FeeTreasury/onchain/validators/fee_treasury.ak` toàn-file | Chốt Feecover-CARP-settle KHÁC FeeTreasury-ADA-FX; đặt tên tách bạch; KHÔNG kế-thừa Withdraw-operator cho CARP |
| **FG-5** | demand_mult keeper (không DAO-gate, không band) đội giá thật + lệch 2 bảng | 🟡 **HỞ** | HỞ — beacon chỉ keeper M-of-N, không `status==Executed`, không band ±% | `pricing.ak:42-47`; `price_param.ak:36-37`; Feecover §4.3, §8.3, D6 | Feecover đọc GIÁ-THẬT-SAU-demand (`required`/`total_burned`) chứ không bảng tĩnh; hoặc ép on-chain `base_price×demand_mult == fee_magic` cùng tx; DAO-gate + band cho demand_mult nếu làm cơ-sở-phí |
| **FG-6** | FeecoverGate bypass — account non-DID / DID giả lọt cổng | 🟡 **HỞ (đôi)** | HỞ tầng owner (FG-3) + HỞ tầng anchor-giả (PersonDID) | Feecover §5; [[anchor-uniqueness-limit]] `state_nft_logic.ak:138-148` | (a) chờ did_commit nội-dung (FG-3); (b) chờ PA2 UniquenessThread cho PersonDID — nếu không, DID-string giả tự-đúc vẫn "hợp-lệ resolve" |
| **FG-7** | Self-service wash — tự tạo service_id, tiêu MAGIC của mình | 🟢/🟡 | Phần-lớn CHẶN (phí về Treasury = tự-đốt); hở nếu có yield-đối-ứng | Feecover §7-A; [[activation-getlamp-model]] GV2/GV7 | Ngăn provider tự-đăng-ký service khống nhận lại phần phí; xem tương-tác với Activation-yield |
| **FG-1** | Hai bản ConsumeMAGIC phân-kỳ (canonical no-mint vs cũ mint-burn); spec neo bản cũ | 🟡 **nợ-nguồn** | HỞ — B1 chưa dứt-điểm; Feecover trỏ path bản cũ | canonical `MAGIC/ConsumeMAGIC/.../consume.ak:95,119`; cũ `MAGIC-paymaster/.../consume.ak:66,73`; Feecover §0/§35; Excu B1 | Chốt B1: canonical = `MAGIC/ConsumeMAGIC` (PR#13); xoá/đóng-băng bản cũ; sửa path-neo trong Feecover spec |
| **FG-2** | `carp_paid==fee_magic` thực-thi được không? | 🟢 CHẶN (canonical) | CHẶN — canonical ép `total_burned == required` (**==**), không over-burn | `MAGIC/ConsumeMAGIC/.../consume.ak:95`; test `consume.ak:719` | Feecover đọc chính `required` beacon → carp_paid==fee_magic ràng được; đảm bảo ServiceFeeSchedule khớp beacon (D6) |

Phân loại: **lỗ-HỞ-THẬT (build-blocking, không blocker-đội-khác che):** FG-4, FG-4b. **blocker-đội-khác (B3):** FG-3. **hở-tham-số/quản-trị:** FG-5, FG-6. **kinh-tế-phụ-thuộc:** FG-7. **nợ-nguồn/nợ-KT:** FG-1, FG-8. **đã CHẶN (canonical):** FG-2.

---

## Chi tiết từng vector (khai-thác → lợi-ích → mức → chặn/hở → vá)

### FG-1 🟡 — Hai bản ConsumeMAGIC phân-kỳ; spec neo bản cũ (NỢ-NGUỒN)

**Vấn đề:** tồn tại HAI copy ConsumeMAGIC với model đối-nghịch:
- **CANONICAL** `MAGIC/ConsumeMAGIC/onchain/validators/consume.ak` (rewrite D1, PR#13): dòng 1-13 ghi rõ "KHÔNG token, KHÔNG tx.mint. MAGIC = số kế toán trong `VaultDatum.magic_batches`. Tiêu MAGIC = handler BurnBatch của VAULT validator". ConsumeMAGIC chỉ là lớp PRICING+ENGAGEMENT; ép `total_burned == total_required` (`consume.ak:95`). `EngageDatum` CÓ `did_commit` (`types.ak:50`). **KHỚP spec Feecover §0.**
- **CŨ/SUPERSEDED** `MAGIC-paymaster/ConsumeMAGIC/onchain/validators/consume.ak`: `check_only_magic_burn` (dòng 118-124) mint âm MAGIC native (có policy-id), `magic_burned >= total_required` (dòng 73, **≥** over-burn OK), `EngageDatum` KHÔNG có did_commit. **TRÁI spec** — bị PR#13 thay.

Feecover §0 (dòng 35) + §1.2 trỏ path `MAGIC-paymaster/ConsumeMAGIC/...` — tức **bản CŨ**. Excu §4 B1 "MAGIC-model reconcile" xác nhận trục account-vs-native **chưa dứt-điểm**.

**Rủi ro:** dev/reviewer đọc nhầm bản CŨ → build Feecover trên model mint-burn/over-burn đã bỏ; hoặc 2 bản cùng sống gây lệch audit. Nhất-quán với memory [[magic-not-native-token]] (anh đảo lại "MAGIC KHÔNG native") + [[two-lamp-policies-tension]] (nhiều bản policy song song). **Không phải lỗ gaming — nợ-nguồn-chân-lý phải dọn TRƯỚC khi build.**

**Mức 🟡 — nợ-nguồn (không gaming, nhưng chặn build sạch).**

**Vá:** (1) chốt B1: canonical = `MAGIC/ConsumeMAGIC` (PR#13, no-mint); (2) xoá/đóng-băng `MAGIC-paymaster/ConsumeMAGIC` để tránh nhầm; (3) sửa path-neo trong `PhoenixKey-Feecover-Feat-Math.md` §0/§1.2/§35 trỏ đúng bản canonical.

---

### FG-2 🟢 — `carp_paid == fee_magic` THỰC-THI-ĐƯỢC trên bản canonical (CHẶN)

**Kiểm-chứng (câu hỏi 2):** bản canonical `MAGIC/ConsumeMAGIC/.../consume.ak:84-95` ép:
- Giá LẤY TỪ BEACON, không tin redeemer amount (`consume.ak:68-71`, `read_price_param` + `valid_param`).
- **`total_burned == total_required`** (`consume.ak:95`, dấu **==** nghiêm — KHÔNG over-burn, KHÁC hẳn bản cũ `≥`). `total_required` = Σ trên MỌI Engage input `price(op_type_i)×op_count_i`; `total_burned` = Σ burns trên MỌI vault_ref phân-biệt (liên-kết vault BurnBatch). Aggregate → idempotent, chống multi-engage→single-vault under-charge (test dòng 719 `pay-once-consume-N` REJECT).
- Mọi Engage input PHẢI cùng `price_ref` (cùng beacon) — không trộn nhiều bảng giá (`consume.ak:89`).

**Kết luận:** vì lượng-tiêu-thật = `required` (đọc-được từ beacon, khớp `total_burned`), Feecover-validator có thể ràng `carp_paid == required == fee_magic(service_id)` một cách **thực-thi-được** — KHÔNG cần chạm consume.ak (giữ P-LAYER-1). Cửa hở "khai service_id rẻ tiêu op đắt" ĐÓNG: op_type→required cố-định theo beacon, service_id→fee_magic phải khớp cùng cadence (D6, xem FG-5).

**Mức 🟢 — CHẶN (bản canonical).** Điều-kiện DUY NHẤT còn lại: ServiceFeeSchedule.fee_magic phải khớp PriceParam beacon-sau-demand (chuyển sang FG-5). Đây là finding tôi ban đầu lo 🔴 nhưng bản canonical đã đóng — trung-thực ghi nhận.

**Vá:** đảm bảo Feecover đọc chính `required`/`total_burned` của tx (không bảng tĩnh riêng); D6 đồng-bộ 2 bảng.

---

### FG-3 🔴 — Gate DID + attribution per-DID chưa thực-thi (did_commit nội-dung `#""`)

**Khai thác:** bản canonical CÓ field `did_commit : ByteArray` (`types.ak:50`, field-3 append-only) + validator ép **immutable** (`consume.ak:119` truyền `in_datum.did_commit` vào `enforce_engagement`, ép `output.did_commit == input.did_commit`). NHƯNG:
- Nội-dung MVP = `#""` (rỗng); `types.ak:44` chú thích thẳng "Validator KHÔNG ràng buộc NỘI DUNG ở MVP". Blocker B3 (Excu §4).
- FeecoverGate (§5) do đó chỉ ràng được `owner` (khoá thô). 1 người sinh N khoá owner → N account "hợp-lệ" → né mọi trần/attribution.
- `FeecoverAccrual.breakdown : List<(service_id, Int)>` per-DID (§3.3) + "truy về từng DID" (§5, §6.1) = **hư cấu** tới khi did_commit chứa blake2b256(DID) thật.
- FEECOVER-GATE-1 point-in-time `did_active`/`key_authorized` (§5) dựa PhoenixKey resolve API mà `VeData-Reply-did-commit-resolve.md:16-17` xác-nhận PhoenixKey-Database **mới có endpoint state-hiện-tại, phần point-in-time chưa implement** (giao Long).

**Lợi-ích kẻ tấn-công:** farm-attribution / né-trần bằng multi-owner; hoặc không-DID vẫn tiêu nếu gate chỉ check `owner`-có-mặt chứ chưa-resolve-DID-thật.

**Mức 🔴 — HỞ (blocker B3).** §7-A + Excu-B3 tự nhận "chưa gắn did_commit → một owner mở nhiều account". Cấu-trúc field đã có (tốt hơn tôi lo), nhưng nội-dung + resolve chưa land → gate vẫn owner-thô.

**Vá:** blocker cứng — KHÔNG bật FeecoverGate/breakdown production tới khi (1) MAGIC-team + Long điền did_commit = blake2b256(DID‖salt) thật + validator ép nội-dung khớp DID resolve, (2) PhoenixKey resolve point-in-time land. Trước đó Feecover chỉ owner-gating thô, PHẢI ghi rõ "attribution không tin-được".

---

### FG-4 🔴 — EpochSettle + provider-vault: kế-toán dựa trust, không cơ-chế ép

**Khai thác (đúng câu hỏi 4):** §6.2 `epoch_settle` là **pseudo-code, 0 validator**. Luồng "gom vào Vault provider (App→Platform) → cuối epoch chuyển CARP về Treasury" đặt provider vào vị-trí custody CARP giữa-chừng. Không cơ-chế on-chain ép:
- **(a) Forced-settle:** provider có thể KHÔNG chạy settle, ngâm CARP trong vault provider vô-hạn (ăn yield/thanh-khoản, hoặc dùng CARP cho việc khác). `last_settled_epoch` monotonic (FEECOVER-SETTLE-1) chỉ chống double/skip-KHI-settle, KHÔNG ép PHẢI-settle.
- **(b) Skim khai-khống:** `total_carp = Σ FeecoverAccrual(Platform).carp_accrued` — nếu provider tự-khai `carp_accrued` (không ép `==` số CARP thật vào vault mỗi tx), provider báo Σ thấp, giữ phần chênh. §3.3 `carp_accrued:Int` là con-số-datum, cần ép khớp value-CARP-thật (như `fee_treasury.ak:63-66` ép `balance == lovelace_of(value)` chống datum-nói-dối) — Feecover CHƯA có ràng này.
- **(c) Đổi đích:** không rào provider tự-đổi địa-chỉ-Treasury/`stabilizer_ref` (FEECOVER-HO-2 nói "chỉ đổi qua Governance" nhưng 0 code).

**So-chiếu code sẵn có `fee_treasury.ak:108-118` (`Withdraw`):** chỉ cần `list.has(tx.extra_signatories, operator)` + `bal_after >= floor` → **operator rút quỹ tự-do trên floor, không cần DAO**. Nếu Feecover-settle mượn khuôn này, provider-operator **rút CARP trước khi tới Treasury** — lỗ kế-toán trực-tiếp.

**Mức 🔴 — HỞ.** §7-C tuyên "FEECOVER-SETTLE-1 + HO-1 chặn drain" nhưng đó là bất-biến GIẤY chưa validator; và chúng chống double-settle, KHÔNG chống provider-ngâm/skim/rút-operator.

**Vá:** settle PHẢI có validator ép: (1) forced-settle — thread-NFT + ép mọi FeecoverAccrual-của-epoch-đóng phải flush trong tx settle, không cho ngâm qua epoch-N+2; (2) `carp_accrued` khớp value-CARP-thật-trong-vault mỗi accrue (chống datum-nói-dối, mẫu `fee_treasury.ak:63-66`); (3) đích Treasury + stabilizer_ref là **param cứng/DAO-gated**, không operator-cosign; (4) KHÔNG mượn `Withdraw`-operator-model cho CARP-fee.

---

### FG-5 🟡 — demand_mult: đội phí thật không-DAO + lệch 2 bảng

**Khai thác (câu hỏi 1 + 6):** phí THỰC mà user burn = `price_of = base_price × demand_mult / Q` (`pricing.ak:42-47`). `demand_mult` clamp `[m_min, m_max]` (mặc-định 0.5×–2.0× trong test) do **committee keeper** post: `price_param.ak` `expect count_sigs(committee, tx.extra_signatories) >= threshold` — **chỉ chữ-ký keeper M-of-N**, KHÔNG đọc Governance `status==Executed`. `price_param.ak` ép `epoch` tăng (chống rollback) nhưng KHÔNG rào biên-độ-thay-đổi-mỗi-epoch cho demand_mult (khác `fee_treasury.ak:154-155` có `step_within_band ±5%`).

Feecover ràng `carp_paid == fee_magic(service_id)` từ **bảng ServiceFeeSchedule tĩnh** (§4.1), nhưng MAGIC burn thật chạy theo `demand_mult` động. §8.3 tự nhận phải "đồng-bộ 2 bảng" nhưng D6 (§9) treo, chưa cơ-chế.

**Lợi-ích kẻ tấn-công:** (a) kẻ kiểm-soát ≥threshold keeper đẩy `demand_mult→m_max` → user burn 2× MAGIC nhưng ServiceFeeSchedule.fee_magic vẫn cũ → **hoặc user trả CARP thiếu (né phí), hoặc kế-toán lệch** (MAGIC-burn ≠ CARP-accrued). (b) Đội-giá thiên-vị: keeper hạ demand_mult cho tx của phe mình, nâng cho phe khác — "đội-giá/hạ-giá thiên-vị" đúng câu hỏi 6, nhưng ở tầng KEEPER (dễ hơn DAO), không phải tầng DAO base_price.

**Mức 🟡 — HỞ.** Không sập-hệ ngay (clamp giới-hạn 2×) nhưng phá neo kế-toán + là đòn quản-trị dưới-DAO.

**Vá:** (a) Feecover đọc **giá-thật-sau-demand** (`magic_burned` từ tx.mint) làm cơ-sở CARP thay vì bảng tĩnh → tự-đồng-bộ, loại lệch. (b) Nếu giữ bảng tĩnh: ép on-chain `base_price[op]×demand_mult/Q == fee_magic[service]` cùng tx (D6). (c) Cân-nhắc DAO-gate demand_mult khi nó thành cơ-sở-phí Feecover (hiện chỉ keeper).

---

### FG-6 🟡 — FeecoverGate bypass (đôi tầng)

**Khai thác (câu hỏi 3):** gate hở ở HAI tầng độc-lập:
- **Tầng owner (= FG-3):** chưa did_commit → gate chỉ biết `owner`; account không-map-DID vẫn lọt nếu gate chỉ check "owner có mặt".
- **Tầng anchor-giả:** kể cả khi resolve API land, [[anchor-uniqueness-limit]] (`state_nft_logic.ak:138-148`, verify live 2026-07-04) chỉ ra PersonDID neo HW_Key P-256 KHÔNG verify on-chain → ai cũng đúc anchor Person với did-string bất-kỳ + controller mình. `did_active(did_giả)` sẽ trả `active` (vì anchor thật tồn-tại on-chain) → **DID-giả PASS gate**. FEECOVER-GATE-3 "đọc reference, không mint" chỉ chống Feecover-tự-đúc-DID-trong-tx, KHÔNG chống DID-giả-đã-đúc-trước.

**Lợi-ích:** né "chỉ PhoenixKey DID" bằng DID tự-đúc; nếu Feecover về sau gắn ưu-đãi/trần per-DID → farm qua N DID-giả.

**Mức 🟡 — HỞ.** Org/Service/Device AN TOÀN (parent-sig verify được — [[anchor-uniqueness-limit]]); chỉ PersonDID hở, mà Feecover user cá-nhân nhắm đúng PersonDID.

**Vá:** (a) FG-3 (did_commit) trước; (b) với PersonDID chờ PA2 UniquenessThread; (c) MVP gate qua entity-đã-verify-parent-sig (Org/Service) trong lúc chờ.

---

### FG-7 🟢/🟡 — Self-service wash (tự tạo service_id tiêu MAGIC của mình)

**Khai thác (câu hỏi 7):** khác Activation (ở đó wash rút LAMP/yield — GV1/GV2/GV7 [[activation-getlamp-model]]), ở Feecover phí về **Treasury**. Provider tự đăng-ký `service_id` khống, tự tiêu MAGIC của chính mình:
- MAGIC bị burn thật + CARP trả thật về Treasury → **tự-đốt tài-nguyên**, không lời trực-tiếp. "Phí về Treasury = luôn tốt?" — **phần lớn ĐÚNG** (là tự-nguyện đốt).
- **NHƯNG hở nếu có yield-đối-ứng:** nếu provider vừa-là-App-nhận-phí vừa-là-user-trả-phí, và cuối-epoch Treasury phân-phối lại về provider theo tỷ-lệ-đóng-góp (chưa thấy cơ-chế này trong spec, nhưng Activation có MAGIC-yield §3.3), thì wash = **bơm số-liệu-đóng-góp để hút phân-phối** → giống GV5/GV7 Activation. Feecover-spec KHÔNG có phân-phối-ngược nên hiện 🟢; thành 🟡 NẾU nối với yield.

**Mức 🟢 (độc-lập) → 🟡 (nối Activation-yield).**

**Vá:** cấm provider tự-đăng-ký service khống nhận-lại; nếu có phân-phối-ngược từ Treasury, attribution PHẢI cross-DID (không self-loop) — cùng điều-kiện did_commit (FG-3).

---

### FG-8 🟡 — Trùng tên/model với fee_treasury.ak đã deploy

**Vấn đề:** `MAGIC/FeeTreasury/onchain/validators/fee_treasury.ak` ĐÃ TỒN TẠI + có test, nhưng là **model KHÁC**: quỹ FX gom **ADA** (không CARP), 5 redeemer `CollectFee/Withdraw/SwapToAda/CommitStep/RevealStep`, TWAP-beacon, operator-cosign, floor-buffer. NORMATIVE của nó trỏ "SPEC PhoenixKey-Fee-Abstraction" (`fee_treasury.ak:4`) — **tên cũ trước khi Payyou→Feecover** ([[module-naming-feecover]]). Feecover-spec §6 mô-tả CARP-accrual-per-tier hoàn-toàn-khác.

**Rủi ro:** hai "fee treasury" cùng hệ, một deploy (ADA-FX) một spec (CARP-settle) → reviewer/dev nhầm; luồng "phí về Phoenix Treasury" (CARP) vs "quỹ FX App" (ADA) chưa phân-định. Đâu là quan-hệ: FeeTreasury-FX đệm ADA cho Feecover (L3 §6.3?) hay hai thứ tách rời?

**Mức 🟡 — nợ kỹ-thuật/mơ-hồ-ranh-giới.**

**Vá:** chốt tách bạch: (1) `fee_treasury.ak` (ADA-FX-fund per-App) = tầng L3-đệm-ADA / hoặc Feecover-legacy cần rename; (2) Feecover-CARP-settle = validator MỚI. Xác định luồng: CARP-fee → Phoenix Treasury (§6.2) có đi qua FX-fund không, hay song-song. Nếu FeeTreasury chính là "L3 stabilizer" §6.3 thì spec phải trỏ đích-danh file này.

---

## Đối chiếu 8 câu hỏi anh giao

1. **Stale-price/beacon gaming:** stale-gate **MẠNH** (`consume.ak:73-75` + `price_param.ak` epoch-monotonic chống rollback). Nhưng chỉ cho `PriceParam`; `ServiceFeeSchedule` chưa có validator (phải mượn khuôn). Riêng **demand_mult keeper** là khe re-anchor không-DAO (FG-5).
2. **carp_paid < fee_magic:** phần MAGIC chặn client-khai-giá + bản canonical ép **`total_burned == required`** (==, không over-burn). carp_paid==fee_magic **THỰC-THI-ĐƯỢC** nếu Feecover đọc chính `required` beacon (FG-2 🟢). Điều-kiện: 2 bảng khớp (FG-5/D6).
3. **FeecoverGate bypass:** **HỞ đôi** — owner-thô (did_commit nội-dung rỗng, FG-3) + anchor-giả PersonDID (FG-6).
4. **Accrual/EpochSettle skim:** **HỞ nặng nhất** — 0 validator, provider-trust thuần, không forced-settle/không chống-skim/Withdraw-operator-model rút tự-do (FG-4). Đây là lỗ THẬT không blocker-đội-khác che.
5. **Attribution gaming:** did_commit có field nhưng nội-dung rỗng → breakdown per-DID hư-cấu, farm-attribution multi-owner mở (FG-3, B3).
6. **Re-anchor quản-trị:** DAO base_price cadence chậm là rủi-ro §7-D; nhưng đòn NHANH hơn là **keeper demand_mult** dưới-DAO, không band ±% (FG-5).
7. **Self-service wash:** phần-lớn CHẶN (tự-đốt về Treasury); hở nếu nối yield-phân-phối-ngược (FG-7).
8. **Vector khác:** FG-1 (2 bản ConsumeMAGIC phân-kỳ), FG-8 (trùng fee_treasury deploy).

## Chốt trung thực

- **CHẶN TỐT (bản canonical `MAGIC/ConsumeMAGIC` PR#13, đã test):** stale/rollback giá, client-khai-giá + `total_burned == required` (== nghiêm, không over-burn), engage-value-preserved (chống drain vault), double-satisfaction, did_commit-immutable. Nền vững Feecover mượn được; carp_paid==fee_magic ràng được KHÔNG cần chạm consume.ak (FG-2 🟢, giữ P-LAYER-1).
- **HỞ THẬT còn nguyên (không blocker-đội-khác che):** **FG-4 EpochSettle/provider-vault** = trust thuần, 0 validator, Withdraw-operator-model có sẵn cho rút tự-do — đây là lỗ **kế-toán** nguy nhất và Feecover-team PHẢI tự vá khi build P2. **FG-4b/FG-8** trùng model fee_treasury deploy → phải chốt ranh-giới trước.
- **HỞ do blocker đội khác:** FG-3 (B3 did_commit nội-dung + resolve), FG-6 (PA2 anchor-uniqueness PersonDID).
- **Nợ phải dọn trước build:** FG-1 (2 bản ConsumeMAGIC — chốt B1 canonical, sửa path-neo spec), FG-5 (đồng-bộ 2 bảng giá D6 + cân DAO-gate demand_mult).
- Feecover tự-nhận "lớp mỏng tái-dùng nguyên-trạng" — phần TÁI-DÙNG (canonical consume) ĐÚNG và vững hơn tôi lo ban đầu; nhưng 5 phần MỚI (bảng-phí, gate-DID, carp-đổi, gom-tầng, settle) **chưa 1 dòng validator**, và **FG-4 settle là nơi Feecover-team gánh trách-nhiệm chống-drain KHÔNG mượn được từ ConsumeMAGIC**.
