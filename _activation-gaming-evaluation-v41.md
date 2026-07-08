# Đánh giá GAMING (kinh-tế/incentive) — Activation v4.1 · 2026-07-07

> **Phạm vi:** phân-tích mechanism-design / adversary-tokenomics của **Activation v4.1** (PhoenixKey GetLAMP, model 2-PHA GATED). Săn cách user/nhóm khai-thác incentive để **RÚT giá-trị mà KHÔNG tiêu-dịch-vụ-thật**. Đây là lỗ **kinh-tế/incentive** — KHÁC red-team money-drain on-chain (đã sạch, 173/173 test + 9 bất-biến, [[redteam-guardian-dup-live-poc]]).
>
> **Nguồn đã đọc (không bịa):**
> - `spec-proposals/PhoenixKey-Activation-Feat-Math.md` v4.1 (model + §3 toán + §4 I-ACT + §7 tham số + §9 self-adversarial).
> - `spec-proposals/_activation-gaming-evaluation.md` (G1–G8 cũ — KHÔNG rubber-stamp; tìm vector MỚI).
> - `PhoenixKey-Validator/lib/phoenixkey/activation_logic.ak` (cơ-chế THẬT — `reclaim_ok`/`vest_ok`/`forfeit_ok`/`claim_ok`/`gen_drip_ok`/`genesis_vault_ok`).
> - Memory: [[anchor-uniqueness-limit]], [[no-sybil-concern]], [[activation-getlamp-model]].
>
> **Khác bản G1–G7 cũ:** bản cũ đánh model 2-pha ở tầng **cung LAMP + settlement + self-loop**. Bản này săn tầng **incentive-extraction**: threshold tự-tham-chiếu, keeper-MVP-collusion, sybil-qua-anchor, wash-qua-Registry-yếu, yield-rút-trên-LAMP-chưa-sở-hữu. Nhiều vector cũ đóng thật; **3 vector còn HỞ ở tầng incentive** mà bản cũ KHÔNG soi.

---

## Kết luận đầu trang (vector nguy-hiểm nhất)

**Lỗ KINH-TẾ THẬT còn hở ở v4.1 (xếp theo mức độ sập-hệ):**

1. **🔴 GV1 — Sybil D-farming qua anchor-uniqueness.** D keyed per-PersonDID, NHƯNG PersonDID **giả-mạo được** (GenesisPerson đúc anchor did-string bất kỳ, HW_Key P-256 không verify on-chain — [[anchor-uniqueness-limit]] `state_nft_logic.ak:138-148`). N DID giả → N×D LAMP rút khỏi pot. **Đây là lỗ làm SẬP kinh-tế** (không phải lo-sybil-sinh-trắc mà [[no-sybil-concern]] bác — mà là **lỗ-mã-hoá-anchor đã verify live**). Phân xử ở GV1.

2. **🔴 GV2 — Wash-consumption qua Registry-tự-đăng-ký.** Cổng chống-wash DUY NHẤT = "chuẩn Registry dịch-vụ tiêu-tài-nguyên-thật" — mà **chuẩn này CHƯA tồn tại** (`§6.1` BLOCKER, Registry-team chưa có danh-mục + cổng duyệt). Không có chuẩn → attacker tự đăng-ký dịch-vụ-vỏ, tiêu MIN_MAGIC_TX túi-trái-túi-phải, giữ active + vest + hút MAGIC-yield vô hạn. Registry-gate là **lời-hứa-tương-lai**, chưa phải phanh.

3. **🟠 GV3 — Keeper-MVP là single-point vừa collude vừa grief.** `vest_ok`/`reclaim_ok`/`forfeit_ok` đều gate bằng `keeper_signed` (system-authority). Keeper attest active-giả → user vest không tiêu; keeper từ-chối attest → grief user tới forfeit. `has_counterparty_consume` trả `False` cứng (`activation_logic.ak:282-287`) → **on-chain KHÔNG có nguồn-sự-thật độc-lập**, tin keeper 100%.

4. **🟠 GV5 — Forfeit-avoidance / yield-hút tối-thiểu.** MIN_MAGIC_TX = 10% gen-able từ `conditional_lamp` (tự-tham-chiếu) → user tiêu ĐÚNG 10% để giữ 100% MAGIC-yield + vest. Đóng-góp 10%, hưởng yield 100% — **hút 90% yield** với chi-phí tối-thiểu. Threshold tự-co khi conditional giảm → càng dễ né.

5. **🟡 GV6 — Pot D-timing / phản-chu-kỳ bị đảo.** GetLAMP đúng lúc pot cao (D=1001), farm qua sybil (GV1) → rút cạn pot lúc nó đầy nhất; phản-chu-kỳ nạp-LAMP không kịp bù N×1001 dòng-ra đồng-thời.

**Cái v4.1 CHẶN TỐT:** cung-LAMP-tức-thời (rào 1001 ngày + vest nhỏ-giọt — validator ép trần luỹ-kế `vest_ok` (5)); catch-up-burst (trần `vested ≤ n−1001` ép rải-đều — GV4 🟢); LAMP-rời-vault-trái-phép (anti-drain + đích-đúng pot/owner, đã red-team sạch); Gen-rút-LAMP (`gen_drip_ok` ép LAMP-preserved). Sybil **đa-ĐỊA-CHỈ** vô ích (D per-PersonDID). Nhưng sybil **đa-PersonDID-GIẢ** thì KHÔNG (GV1).

---

## Bảng tổng

| # | Vector | Mức | Chặn/Hở | Neo | Vá đề-xuất |
|---|---|---|---|---|---|
| **GV1** | Sybil D-farming qua anchor-giả | 🔴 **HỞ** | HỞ — D per-PersonDID nhưng PersonDID giả-mạo được | [[anchor-uniqueness-limit]]; §1.1; §9.A | PA2 UniquenessThread land TRƯỚC khi mở GetLAMP-PersonDID; hoặc GetLAMP gate qua entity đã-verify-parent-sig |
| **GV2** | Wash-consumption qua Registry-vỏ | 🔴 **HỞ** | HỞ — chuẩn Registry chưa tồn tại (BLOCKER §6.1) | §3.3a; §6.1; `activation_logic.ak:282` | Không bật anti-idle/vest-gate production tới khi Registry-chuẩn land + có phí-đăng-ký/staking đủ răn |
| **GV3** | Keeper-MVP collude + grief | 🟠 **HỞ (MVP)** | HỞ có-chủ-đích ở MVP; tin keeper 100% | `activation_logic.ak:336,410,568`; §3.5 | Đa-keeper threshold; thay keeper-attest bằng consume-event reference-input; grief-guard (user tự chứng-minh active) |
| **GV4** | Vest catch-up burst | 🟢 CHẶN | validator ép trần luỹ-kế `vested ≤ n−1001` | `vest_ok` (5) `activation_logic.ak:417` | — (đã chặn on-chain) |
| **GV5** | Forfeit-avoidance / hút-yield tối-thiểu | 🟠 **HỞ (kinh-tế)** | HỞ — threshold 10% tự-tham-chiếu → hút 90% yield | §3.4; §7 (MIN_MAGIC_TX) | MIN neo giá-trị-tuyệt-đối (không %-of-self); hoặc yield tỷ-lệ-thuận mức-tiêu (không all-or-nothing) |
| **GV6** | Pot D-timing / đảo phản-chu-kỳ | 🟡 khuếch-đại GV1 | phụ-thuộc GV1; nếu GV1 vá thì 🟢 | §3.6; §6.5 | Rate-limit GetLAMP/epoch; D giảm theo tốc-độ-rút gần đây |
| **GV7** | MAGIC-yield extraction ròng (Feecover ADA) | 🟡 phụ-thuộc định-giá | HỞ nếu MAGIC-service-cost < ADA-phí Feecover trả | §3.3; I-ACT-9 | Feecover-cap/user; MIN_MAGIC_TX phải ≥ chi-phí-ADA-Feecover-biên |
| **GV8** | Epoch-boundary timing | 🟢 phần-lớn chặn | `p2_epoch` gap-đo ungameable; còn 1 mép nhỏ | `activation_logic.ak:181-188`; §3.7 | (xem GV8 — chốt độ-dài-epoch + đo-gap giữ nguyên) |
| GV9 | Abandon-rejoin re-D | 🟡 mở nếu D re-grant | phụ-thuộc 1-PersonDID-1-vault-đời | I-ACT-10; §3.5 | did_commit ghi-nhớ đã-cấp; abandon KHÔNG cho re-GetLAMP cùng PersonDID |

Phân loại: **lỗ-KINH-TẾ-THẬT (sập/rút-ròng):** GV1, GV2, GV5 (+GV6 khuếch-đại). **Tin-cậy-MVP (mitigatable):** GV3. **Phạt-tự-nhiên / đã-chặn:** GV4, GV8. **Phụ-thuộc-tham-số:** GV7, GV9.

---

## GV1 🔴 — Sybil D-farming qua anchor-uniqueness (LỖ SẬP KINH-TẾ)

**Cơ-chế khai-thác:**
- D keyed **per-PersonDID** (§1.1, §9.A) — thiết-kế giả-định "1 người sinh-trắc = 1 suất, uniqueness do Secure Enclave lo, ngoài phạm vi".
- NHƯNG [[anchor-uniqueness-limit]] (verify live 2026-07-04): PersonDID gốc neo vào **HW_Key P-256**; Plutus V3 **không verify P-256** → `GenesisPerson` (`state_nft_logic.ak:138-148`) chỉ tự-ký Ed25519 controller, **KHÔNG ép did-hash ↔ khoá-gốc-sinh-trắc**. `anchor_name = blake2b_256(UTF-8(did))`, did-string public → **ai cũng đúc anchor Person với did-string BẤT KỲ + controller của mình**.
- Vault Activation gắn `did_commit` (I-ACT-10) — nhưng `did_commit = blake2b_256(did‖salt)` chỉ ràng **1-vault-1-DID-string**, KHÔNG ràng DID-string đó là **người-thật-duy-nhất**. `genesis_vault_ok` (`activation_logic.ak:717-759`) chỉ kiểm `name == owner_commit == datum.owner_commit` + LAMP-khoá-đúng-D — **KHÔNG kiểm did-string tương-ứng người-sinh-trắc-có-thật-duy-nhất**.

**Lợi-ích-ròng kẻ tấn-công:** đúc N PersonDID-string khác nhau (miễn phí, chỉ tốn min-ADA + phí-tx) → mỗi cái GetLAMP nhận D=min(1001, pot/1e6) LAMP vào vault-riêng → **N×D LAMP rút khỏi pot**. Ở pot bão-hoà (≥1.001B LAMP) → N×1001 LAMP/attacker. Rồi tiêu-wash tối-thiểu (GV2) qua PHA-1→PHA-2 → **vest thành LAMP sở-hữu thật, rút ra ví, bán**. Đây là **rút giá-trị-ròng ra khỏi hệ** — chính xác cái "sập kinh-tế".

**Phân xử [[no-sybil-concern]] vs [[anchor-uniqueness-limit]]:**
- [[no-sybil-concern]] nói: "tạo PersonDID BUỘC có sinh-trắc vân tay; giả vân tay = sập cả thế-giới sinh-trắc; 1 người = 1 PersonDID là đủ". **Điều này đúng ở tầng THIẾT-BỊ-tạo-khoá** (khoá HW sinh sau FaceID/TouchID).
- NHƯNG [[anchor-uniqueness-limit]] chỉ ra lỗ **KHÁC tầng**: on-chain KHÔNG verify được "anchor này neo vào khoá-sinh-trắc". Attacker **KHÔNG cần giả vân tay** — chỉ cần đúc anchor với **controller Ed25519 của chính mình** (khoá thường, không cần sinh-trắc) + did-string tuỳ-chọn. Sinh-trắc bảo-vệ **khoá-thật-của-người-thật** khỏi bị chiếm, KHÔNG ngăn attacker **tạo did-string-mới-toanh mà mình tự-kiểm-soát**.
- **Phân xử:** ở tầng "chiếm DID người khác" → sinh-trắc đủ, [[no-sybil-concern]] đúng, KHÔNG nêu. Ở tầng **"tự đúc N DID-string mới mình kiểm-soát để farm D"** → sinh-trắc KHÔNG chặn (không đòi sinh-trắc khi đúc anchor-Person); đây là **lỗ-mã-hoá-anchor THẬT** (Person-over-anything), KHÔNG phải "lo-sybil-sinh-trắc" mà anh bác. **→ nêu, mức 🔴.**

**Cơ-chế-hệ có chặn không:** KHÔNG. §1.1 đẩy uniqueness "ngoài phạm vi" — nhưng chính-vì-ngoài-phạm-vi mà kinh-tế Activation **hở toang** khi land trên PersonDID chưa-có-PA2. Org/Service/Device AN TOÀN (parent-sig verify được — [[anchor-uniqueness-limit]]); chỉ **PersonDID hở**, và GetLAMP nhắm đúng PersonDID (user cá-nhân).

**Vá đề-xuất:**
1. **Chặn cứng:** KHÔNG mở GetLAMP cho PersonDID tới khi **PA2 UniquenessThread** land (ép at-most-one-anchor-per-name toàn-cục — [[anchor-uniqueness-limit]]). Đây là điều-kiện-tiên-quyết kinh-tế, không chỉ custody.
2. **Chặn mềm tạm:** GetLAMP gate qua **entity đã-verify-parent-sig** (Org/Service) trong lúc chờ PA2 — enterprise onboarding chạy trước, Person-tự-phục-vụ chờ.
3. **Rate-limit + phí-đúc-anchor** không đủ một-mình (attacker vẫn lời nếu D×giá-LAMP > phí×N), nhưng làm chậm.

**[GHI TRUNG THỰC]** Đây là lỗ **cross-spec**: Activation-spec đúng khi giả-định "PersonDID unique", nhưng giả-định đó **CHƯA đúng trên code hiện tại**. Không phải lỗi validator Activation — mà là **Activation không-được-deploy-PersonDID trước PA2**. Cùng cảnh-báo [[anchor-uniqueness-limit]] "đừng deploy did_payment/stake/subaddr custody cho PersonDID tới khi PA2 land" — **GetLAMP-PersonDID chịu CÙNG điều-kiện.**

---

## GV2 🔴 — Wash-consumption qua Registry-tự-đăng-ký (cổng-chống-wash chưa tồn tại)

**Cơ-chế khai-thác:**
- v4.1 hợp-thức-hoá self-consumption (đính-chính 07-07): "túi-trái-túi-phải TỐT, mỗi lượt tốn-phí→Treasury". Cổng chống-lạm-dụng chuyển từ "counterparty≠owner" sang **"tiêu qua dịch-vụ đăng-ký-Registry tiêu tài-nguyên-thật"** (§3.3a).
- **VẤN ĐỀ:** chuẩn Registry đó **CHƯA TỒN TẠI**. §6.1 ghi rõ BLOCKER: "cần chuẩn danh-mục loại tài-nguyên/dịch-vụ thật + cổng duyệt — Registry-team". `activation_logic.ak:278-287` `has_counterparty_consume` trả `False` cứng (placeholder). On-chain **KHÔNG có** kiểm "tiêu-này-qua-dịch-vụ-Registry-nào" — toàn-bộ dựa keeper-attest (GV3).
- Attacker: đăng-ký một "dịch-vụ" (khi Registry mở) tuyên-bố tiêu "tính-toán/lưu-trữ" nhưng thực-chất **rỗng hoặc chi-phí-biên ≈ 0** (ví: "dịch-vụ echo", "lưu 1 byte", self-hosted trên máy rẻ). Tiêu MAGIC-của-mình qua dịch-vụ-mình → mỗi lượt tốn phí (CARP/LAMP về Treasury) NHƯNG attacker **thu lại phí đó vì mình là provider-vault** (Feecover→provider-vault→epoch-end→Treasury — [[magic-not-native-token]]: phí→provider-vault App/Platform). Nếu attacker LÀ App-provider → phí quay về túi mình phần lớn.

**Cost-to-fake vs reward:**
- **Cost:** phí-đăng-ký-Registry (chưa định) + chi-phí-tài-nguyên-biên-thật (attacker tối-thiểu-hoá → gần 0) + phí-tx-tiêu.
- **Reward:** giữ active → không bị anti-idle rút (PHA-1) → giữ full `conditional_lamp` sinh full MAGIC-yield; giữ epoch-active (PHA-2) → vest LAMP thành sở-hữu-thật → rút/bán. **Reward >> cost** nếu Registry-duyệt lỏng.

**Registry-gate mạnh tới đâu:** **hiện = 0** (chưa có chuẩn). Sức mạnh TƯƠNG-LAI phụ-thuộc HOÀN-TOÀN vào:
- Chuẩn "tài-nguyên-thật" đo được KHÔNG bằng self-report (attacker khai láo). Phải đo **chi-phí-biên-thật** (khó — làm sao chứng-minh 1 tx thực-sự tiêu CPU/băng-thông?).
- Phí-đăng-ký / stake-đủ-lớn để dịch-vụ-vỏ lỗ-vốn.
- **[GHI TRUNG THỰC]** Đây là bài-toán mở khó nhất của v4.1. "Tiêu tài-nguyên-thật" là tiêu-chí **kinh-tế-xã-hội**, KHÔNG kiểm được on-chain thuần. Self-consumption-hợp-lệ (anh chốt đúng về nguyên-lý: nếu tiêu tài-nguyên-thật thì hệ có lợi) nhưng **ranh-giới "thật vs vỏ" chính là chỗ wash sống**.

**Lợi-ích-ròng:** nếu Registry lỏng → attacker giữ vault active vĩnh-viễn với chi-phí ~0, hút MAGIC-yield + vest LAMP. Kết-hợp GV1 (N vault) → **N × (full-yield + full-vest) rút-ròng**.

**Vá đề-xuất:**
1. **KHÔNG bật anti-idle/vest-gate production** tới khi Registry-chuẩn land (§8 đã ghi BLOCKER — giữ đúng).
2. Registry-duyệt phải có **stake/phí-đăng-ký ≥ giá-trị-vest-tối-đa-một-vault-có-thể-farm-qua-dịch-vụ-đó** → dịch-vụ-vỏ lỗ.
3. **Đo tiêu-thật bằng counterparty-đa-dạng** (không phải đơn-counterparty): dịch-vụ có ≥K user-độc-lập tiêu mới tính "thật". (Lưu ý: mâu-thuẫn tinh-thần "self-consumption hợp-lệ" — cần anh cân: self-consumption-TỐT nhưng self-consumption-ĐỘC-QUYỀN-100% = cờ-đỏ wash.)
4. Cap MAGIC-yield-từ-self-consumption theo tỷ-lệ (ví: ≤50% active-credit từ dịch-vụ-mình-sở-hữu).

---

## GV3 🟠 — Keeper-MVP: single-point vừa-collude vừa-grief

**Cơ-chế khai-thác:**
- `reclaim_ok` (`activation_logic.ak:336`), `vest_ok` (`:410`), `forfeit_ok` (`:568`) đều gate **`keeper_signed`** (system-authority). `has_counterparty_consume` trả `False` (`:282-287`) → **on-chain KHÔNG có nguồn-sự-thật độc-lập** về "user có tiêu thật không". Keeper là oracle duy-nhất.
- **Collude:** keeper (hoặc attacker mua-chuộc keeper) attest epoch-active GIẢ → `vest_ok` pass dù user KHÔNG tiêu → user vest LAMP thành sở-hữu **không đóng-góp gì**. Rút-ròng.
- **Grief:** keeper từ-chối attest active cho user-thật → user không vest được (VestToOwner cần keeper-sig) → tích idle-epoch → **forfeit** `conditional_lamp` dù user tiêu-thật. Keeper cũng có thể **KHÔNG chạy reclaim** cho user-idle (bỏ mặc) làm méo pot-conservation, hoặc **chạy forfeit sớm** nếu logic gap-đo sai.

**Mức tin-cậy MVP:** spec + validator ghi RÕ "MVP tin keeper" (`activation_logic.ak:281` "MVP tin keeper"; §3.5 "Keeper attest = MVP; sau thay consume-event Registry"). Đây là **hở CÓ-CHỦ-ĐÍCH** ở MVP, KHÔNG phải lỗ ẩn. Nhưng ở tầng kinh-tế: **toàn-bộ tính-đúng-đắn của vest/forfeit/anti-idle phụ-thuộc 1 keeper trung-thực** — nếu keeper = attacker (hoặc bị mua) → GV3 nuốt cả GV2/GV5 (không cần wash, keeper cứ attest).

**Lợi-ích-ròng:** nếu keeper-collude → attacker bỏ-qua toàn-bộ điều-kiện-tiêu, vest thẳng. Nếu keeper-độc-lập-trung-thực → GV3 = 0. **Rủi-ro = tập-trung-hoá keeper.**

**Vá đề-xuất:**
1. **Thay keeper-attest bằng consume-event reference-input** (spec đã hứa §3.5, §6.1) — on-chain đọc bằng-chứng-tiêu-thật, không tin keeper. Đây là fix gốc.
2. Trong lúc chờ: **đa-keeper M-of-N threshold** (không 1 keeper), giảm single-point.
3. **Grief-guard:** cho user tự-submit bằng-chứng-active (consume-event của chính mình) để mở vest / chặn forfeit — không phụ-thuộc keeper chủ-động.

---

## GV4 🟢 — Vest catch-up burst (CHẶN on-chain)

**Cơ-chế thử:** idle nhiều epoch, rồi burst 1 tx vest toàn-bộ catch-up (né tinh-thần "dùng-đều").

**Chặn:** `vest_ok` điều-kiện (5) `activation_logic.ak:417`: `d_out.vested_unlocked <= n - phase1_last` — **trần luỹ-kế theo ngày**. Ngày 1002 → vest ≤1; ngày 1500 → vest ≤499. Dù user gọi VestToOwner lúc nào, tổng-vest KHÔNG vượt (n−1001). Cộng `k >= 1` + `n > last_tick_day` (monotonic tick) → mỗi tx tiến ≤ số-ngày-trôi. **KHÔNG burst-catch-up-quá-trần được.** Idle rồi active-lại chỉ vest tiếp từ mốc hiện-tại, KHÔNG hồi-tố ngày-idle (vì cần epoch-active tại-thời-điểm). 🟢 đã chặn.

**Lưu ý mép:** trần là luỹ-kế-theo-NGÀY nhưng gate-active theo-EPOCH → user active 1 epoch có thể vest 5 ngày catch-up (VEST_UNIT=1/ngày × 5 ngày trong epoch). Đây KHÔNG phải lỗ (vẫn ≤ n−1001, vẫn cần epoch-đó-active) — chỉ là **granularity epoch-vs-ngày** (§3.7 [CẦN CHỐT]). Không rút-ròng.

---

## GV5 🟠 — Forfeit-avoidance / hút-yield tối-thiểu (threshold tự-tham-chiếu)

**Cơ-chế khai-thác:**
- `MIN_MAGIC_TX = 10% × MAGIC_genable(granted_LAMP)`, `granted_LAMP = conditional_lamp` (§3.4, §7). Threshold **tự-tham-chiếu** vào chính số-dư vault.
- User tiêu **ĐÚNG 10%** gen-able mỗi kỳ → `active = True` → giữ 100% `conditional_lamp` (không bị reclaim PHA-1 / forfeit PHA-2) → tiếp-tục Gen sinh **100% MAGIC-yield** + vest full. **Đóng-góp-tiêu 10%, hưởng-yield 100% → hút 90% yield** với chi-phí tối-thiểu.
- Threshold **tự-co**: nếu conditional giảm (do vest sang vested, hoặc reclaim), MIN_MAGIC_TX giảm theo → càng-về-sau càng-dễ-đạt-ngưỡng. Ở PHA-2, `vested_unlocked` KHÔNG tính vào granted_LAMP (§3.4 "KHÔNG gồm vested_unlocked") → user vest càng nhiều, conditional càng nhỏ, ngưỡng càng thấp → **né gần-như-miễn-phí** khi gần vest xong.

**Đây có phải "hút yield" không:** CÓ, ở tầng incentive. Hệ MUỐN user tiêu-thật NHIỀU (F8 giá-trị-từ-tiêu-dịch-vụ). Cơ-chế v4.1 chỉ đòi **10% tối-thiểu** → user tối-ưu-hoá tiêu đúng-sàn, giữ 90% MAGIC-quyền-tiêu **không dùng** (use-it-or-lose-it: mất suất-kỳ-đó — nhưng KHÔNG mất LAMP-điều-kiện vì đã active). Kết-hợp GV2 (10% đó cũng wash qua dịch-vụ-mình) → **đóng-góp-thật ≈ 0, giữ full vault**.

**Lợi-ích-ròng:** giữ full conditional_lamp → full MAGIC-yield-nền + full vest-tiềm-năng, chi-phí = 10%-wash. Nếu 10% wash-được (GV2) → chi-phí thực ≈ phí-tx. **Hút yield gần-100%.**

**Cơ-chế-hệ có chặn:** KHÔNG hoàn-toàn. use-it-or-lose-it (mất suất-kỳ) là phạt-tự-nhiên cho phần-KHÔNG-tiêu — nhưng KHÔNG phạt phần-vault (LAMP giữ nguyên). Threshold-tự-tham-chiếu làm sàn co theo vault → không có sàn-tuyệt-đối buộc đóng-góp-thật.

**Vá đề-xuất:**
1. **MIN_MAGIC_TX neo giá-trị-tuyệt-đối** (không %-of-self) — ví: sàn cố-định theo giá-trị-dịch-vụ-thật (§4 MAGIC neo-dịch-vụ), độc-lập conditional. → attacker không co-sàn-bằng-cách-vest.
2. **Yield tỷ-lệ-thuận mức-tiêu** thay all-or-nothing: MAGIC-yield / vest-tốc-độ ∝ tỷ-lệ-tiêu-thật kỳ trước (tiêu 10% → yield/vest 10%). Bỏ ngưỡng-nhị-phân → hết chỗ "đúng-sàn-hưởng-full". (Đây là thay-đổi model lớn — cần anh cân với "1 LAMP/ngày nhỏ-giọt".)
3. Tối-thiểu: MIN neo `D-gốc` (không `conditional-hiện-tại`) → sàn không-co khi vest.

---

## GV6 🟡 — Pot D-timing / đảo phản-chu-kỳ (khuếch-đại GV1)

**Cơ-chế:** D=min(1001, ⌊pot/1e6⌋) — D cao khi pot cao. Attacker (qua GV1 sybil) canh **pot đầy nhất** (sau treasury-topup / đợt-nạp-phản-chu-kỳ) → farm N×1001 đồng-loạt → rút cạn pot lúc nó vừa-đầy. Phản-chu-kỳ (nạp-LAMP-khi-giá-xuống) + treasury-topup **không kịp bù** N dòng-ra tức-thời. §9.F đã nhận "pot cạn khi nhiều user qua PHA-2" là rủi-ro thật; GV6 làm nó **tệ hơn** vì sybil khiến "nhiều user" = 1 attacker.

**Lợi-ích-ròng:** phụ-thuộc GV1. Nếu GV1 vá (PA2) → user-thật không farm-đồng-loạt được → GV6 co về rủi-ro-tham-số bình-thường (§6.5 sub-open). **GV6 KHÔNG độc-lập nguy-hiểm — nó khuếch-đại GV1.**

**Vá:** (a) vá GV1 trước (gốc); (b) **rate-limit GetLAMP/epoch** toàn-hệ (số vault mới/epoch có trần); (c) D giảm theo **tốc-độ-rút-pot-gần-đây** (D = f(pot, drain_rate)) → pot rút nhanh → D tự-giảm, chống-burst.

---

## GV7 🟡 — MAGIC-yield extraction ròng qua Feecover-trả-ADA

**Cơ-chế khai-thác:**
- PHA-1: user hưởng MAGIC (tiêu dịch-vụ) trên `conditional_lamp` **chưa-sở-hữu**. Feecover trả **phí ADA thật** thay user ([[activation-getlamp-model]]: "Feecover lo ADA"). Nếu **giá-trị-dịch-vụ user nhận (qua MAGIC) − chi-phí-ADA-Feecover-trả > 0**, và user rút được giá-trị đó → **rút-ròng ADA-value** dù chưa sở-hữu LAMP.
- Cụ-thể: user tiêu MAGIC vào dịch-vụ-của-mình (GV2), dịch-vụ nhận giá-trị-tiêu (CARP về provider-vault=mình), NHƯNG **Feecover đã trả ADA-phí-mạng** cho tx đó. Nếu Feecover-ADA-trả > chi-phí-thật-dịch-vụ → chênh-lệch = **ADA rút khỏi Feecover-pool**.

**Lợi-ích-ròng:** phụ-thuộc định-giá: nếu `MIN_MAGIC_TX-service-cost < ADA-phí-Feecover-biên` → mỗi tx wash rút ADA-ròng từ Feecover. Ở quy-mô N-vault (GV1) × nhiều-tx → **drain Feecover-ADA-pool**. Đây là lỗ **KHÁC pot-LAMP** — nhắm ADA-subsidy.

**Cơ-chế-hệ có chặn:** phụ-thuộc Feecover-policy (ngoài spec Activation này). Nếu Feecover cap-ADA/user hoặc chỉ-trả-cho-tiêu-thật-qua-Registry → giảm. Nhưng hiện **chưa nêu cap trong Activation-spec**.

**Vá:** (1) Feecover **cap ADA-subsidy/PersonDID/epoch**; (2) MIN_MAGIC_TX phải ≥ **chi-phí-ADA-Feecover-biên** (đảm-bảo mỗi tx-active tự-bù phí-mạng); (3) chỉ Feecover-trả cho tiêu-qua-Registry-hợp-chuẩn (nối GV2-fix).

---

## GV8 🟢 — Epoch-boundary timing (phần-lớn chặn)

**Cơ-chế thử:** canh mép epoch để (a) forfeit-đo-sai, (b) vest-2-lần-1-epoch, (c) reset-idle-rẻ.

**Chặn:**
- Forfeit đo bằng **GAP** `p2_epoch(now) − last_tick_epoch ≥ 1001` (`activation_logic.ak:572`, `:621`), KHÔNG counter-tăng-dần → **ungameable** (không cần tick 1001 lần; attacker không chèn-tx-giả-để-reset). `last_tick_epoch` chỉ tiến qua vest-thành-công (`vest_ok` (7) `:422`) → muốn giữ mốc phải **thật-sự vest = thật-sự active** (theo keeper, GV3). [[activation-getlamp-model]] xác nhận "idle đo bằng gap, ungameable".
- Vest monotonic theo NGÀY (`n > last_tick_day` `:412`) + trần luỹ-kế (`:417`) → không vest-2-lần-quá-trần trong 1 epoch.
- `p2_epoch` off<0 → −1 guard (`:183`), không tính-âm mép PHA-1→PHA-2.

**Mép còn lại:** gate-active theo EPOCH nhưng vest-tick theo NGÀY (§3.7 chưa-chốt độ-dài-epoch dùng-gate) → user active-1-tx-đầu-epoch có thể mở vest cả-5-ngày epoch đó. KHÔNG rút-ròng (vẫn ≤ trần, vẫn cần active). Cần §3.7 chốt để tránh mơ-hồ, KHÔNG phải lỗ. 🟢.

---

## GV9 🟡 — Abandon-rejoin re-D (nếu D re-grant cùng PersonDID)

**Cơ-chế thử:** GetLAMP → PHA-1 vài ngày → AbandonPhase1 (conditional về pot) → GetLAMP LẠI cùng PersonDID nhận D mới. Nếu D re-grant mỗi lần rejoin → farm-vòng: mỗi vòng nhận D, tiêu-tối-thiểu, abandon, lặp. Hoặc canh pot-cao rejoin để re-D lớn hơn.

**Chặn hiện có:** I-ACT-10 "một DID không mở nhiều vault né trần (khi did_commit per-DID sẵn sàng)". `genesis_vault_ok` kiểm `name == owner_commit` — nhưng **KHÔNG kiểm did_commit-này-đã-từng-cấp-D-chưa** (không có sổ-toàn-cục "PersonDID X đã GetLAMP"). Nếu abandon burn-vault-NFT (CloseVault) → cùng did-string đúc vault-mới-được → **re-GetLAMP được**. Chặn phụ-thuộc backend ghi-nhớ "did_commit đã-cấp" (§6.2 resolve-API, chưa có).

**Lợi-ích-ròng:** thấp một-mình (abandon = mất phần-đã-tiêu, re-D chỉ hữu-ích nếu pot-timing lời — nối GV6). Nhưng **kết-hợp GV1** (mỗi PersonDID-giả rejoin nhiều lần) → nhân-bội. 🟡.

**Vá:** backend/on-chain ghi **1-PersonDID-1-lần-GetLAMP-đời** (sổ did_commit đã-cấp, hoặc cooldown dài sau abandon). Nối §6.2.

---

## Tổng kết trung thực

**v4.1 CHẶN TỐT (không tô hồng — đây là thật):**
- Cung-LAMP-tức-thời: rào 1001 ngày + vest nhỏ-giọt, validator ép trần luỹ-kế `vest_ok`(5) — **GV4 🟢 chặn cứng on-chain**.
- Money-drain on-chain: anti-drain (`nonlamp_preserved`+`only_expected_policies`) + đích-đúng (`lamp_to_addr` pot/owner) — red-team sạch, 173/173 test.
- Epoch-timing: gap-đo forfeit ungameable — **GV8 🟢**.
- Sybil **đa-địa-chỉ**: vô-ích (D per-PersonDID) — đúng [[no-sybil-concern]] ở tầng đa-address.

**v4.1 CÒN HỞ (lỗ-kinh-tế-thật, KHÔNG tô hồng):**
- **🔴 GV1** sybil **đa-PersonDID-GIẢ** qua anchor-uniqueness — **SẬP kinh-tế** nếu deploy PersonDID trước PA2. KHÁC lo-sybil-sinh-trắc; là lỗ-mã-hoá-anchor đã-verify-live. **Chặn = PA2 hoặc entity-gate.**
- **🔴 GV2** wash qua Registry-chưa-tồn-tại — cổng-chống-wash là lời-hứa (§6.1 BLOCKER), chưa phải phanh. **Chặn = không-bật-production tới khi Registry-chuẩn + stake-đủ-răn.**
- **🟠 GV3** keeper-MVP single-point collude/grief — hở-có-chủ-đích MVP; **chặn = consume-event thay keeper + đa-keeper.**
- **🟠 GV5** hút-yield-tối-thiểu qua threshold-tự-tham-chiếu (10%-of-self) — đóng-góp-10%-hưởng-100%. **Chặn = MIN neo-tuyệt-đối / yield-tỷ-lệ-thuận.**
- **🟡 GV6/GV7/GV9** khuếch-đại/phụ-thuộc (pot-timing, Feecover-ADA-drain, abandon-rejoin) — nguy khi cộng-dồn GV1/GV2.

**Nguy-hiểm nhất = GV1 (sập-hệ) + GV2 (wash-vô-hạn).** Hai cái này **cộng-hưởng**: N-PersonDID-giả (GV1) × wash-Registry-vỏ (GV2) × hút-yield-10% (GV5) = **rút-ròng N × full-vault, chi-phí ≈ phí-tx**. Đây là kịch-bản sập-kinh-tế thực-sự, và **cả hai đều là điều-kiện-tiên-quyết-chưa-đủ** (PA2 chưa land, Registry-chuẩn chưa có) — nghĩa là **Activation KHÔNG-được-mở-GetLAMP-PersonDID-production** cho tới khi hai phanh đó land. Validator on-chain đúng; lỗ nằm ở **giả-định-tầng-trên chưa-thoả**.
