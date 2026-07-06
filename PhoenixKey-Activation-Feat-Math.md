# PhoenixKey — Activation (Feat + Math) · v2 (re-ground whitepaper)

> **Nguồn chuẩn duy nhất:** `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` (anh Aladin chốt). Bản v1 trích `MagicLamp-3Token-DacTa-Vi.md` — **đã bỏ**; mọi luận điểm neo lại whitepaper theo §-số + dòng. Khi mâu thuẫn → theo whitepaper.
> SPEC PROPOSAL (design doc, chờ anh chốt) · 2026-07-06. KHÔNG code production — chỉ spec. Dẫn `file:line`.

---

## 0. Trạng thái + nguồn bám

**Một câu:** Activation là luồng khởi-tạo-người-dùng của PhoenixKey — user tạo khoá bằng vân tay (Secure Enclave), bấm **GetLAMP** nhận **D LAMP** từ **pot User** vào **vault vesting** khoá-chặt, mở **1 LAMP/ngày** tới hết; LAMP đã mở đem **bán tự do** HOẶC đưa vào **ScheduleGen/InstantGen** (repo MAGIC) sinh **MAGIC**; MAGIC tiêu dịch vụ qua **Feecover → GreenBack backed-path → Phoenix Treasury**; **GetMAGIC** mua CARP bằng fiat để chi tiêu dần.

**Nguồn canonical (whitepaper là chuẩn; các file khác chỉ chi-tiết-code):**
- `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` — **NGUỒN CHUẨN.** §1 dòng 47 + F8 §10 dòng 357 (`INV-MAGIC-CITIZEN`); §4 dòng 92-96 (MAGIC neo-dịch-vụ, nanogic=10⁹); §7.2 dòng 227 (use-it-or-lose-it), dòng 229 ("có-back, không Terra"); §7.3 dòng 240 (counter-cyclical GreenBack); §10 dòng 350 (F1 MAGIC một-chiều), dòng 358 (F9 không-đỡ-peg-LAMP); §13 dòng 426 (đánh-đổi Gen-Terra khai-báo).
- `spec-proposals/_activation-gaming-evaluation.md` (v2) — G1–G6 đã re-ground; lỗ thật còn lại = G3.
- `spec-proposals/PhoenixKey-Feecover-Feat-Math.md` §6 (gom CARP → Treasury), §8 (phụ thuộc `did_commit` + resolve API).
- `spec-proposals/MSG-to-MAGIC-Launch-policy-mint.md` (DID-gate vault spend handler; nanogic=10⁹).
- `MAGIC/SPEC/Carpet-CARP-DacTa-Vi.md` §4.1 (`GreenBack` chi tiết `κ_eff`/`br` — chỉ để khai interface, KHÔNG re-spec).
- `MAGIC/InstantGen/MATH.md`, `MAGIC/ScheduleGen/MATH.md` (engine Gen — tái dùng).
- `LAMP/Genesis/onchain/validators/dist_treasury.ak:14-27` (pot pattern: script + authority-sig release).
- `MAGIC/ScheduleGen/onchain/lib/magiclamp/protocol/lock.ak:9-56` (lock/unlock LAMP holdings — **tái dùng**).
- `spec-proposals/VeData-Reply-did-commit-resolve.md` (`did_commit = blake2b_256(did ‖ salt)`; resolve point-in-time).

---

## 1. Mục tiêu + đổi từ model cũ

**Model cũ (BỎ):** user nạp VND → nhận LAMP + ít ADA trả phí ("Genie"). Vấn đề: (a) phát ADA từng user = chi phí thật rò rỉ; (b) buộc user hiểu tỷ giá 2 tài sản; (c) không có vòng tự-nuôi.

**Model mới (anh Aladin chốt):** **Feecover trả phí mạng thay user** (`Feecover §6`; phí gom về Phoenix Treasury). Hệ quả:

1. **KHÔNG cần cho user ADA.** ADA/DUST/phí do Feecover lo. Activation chỉ phát **LAMP** (tài nguyên nền).
2. **LAMP không phát-trực-tiếp-tự-do** mà qua **vault vesting khoá-chặt, mở 1 LAMP/ngày** — biến "airdrop một cục" thành "dòng chảy có nhịp", chống dump-tức-thì + tạo điểm-neo-quay-lại **hằng ngày**.
3. **Pot tự-nuôi, KHÔNG BAO GIỜ cạn** (§3.6). 3 nguồn nạp: (a) **phí hoạt động user trước** (thu bằng LAMP theo giá-trị — phản-chu-kỳ), (b) **anti-idle thu-hồi hằng NGÀY** (ai một ngày không dùng → 1 LAMP về pot), (c) **MagicLamp Treasury**. Người-bỏ-cuộc + phí-user-trước nuôi người-mới.
4. **Anti-idle-hằng-ngày = cơ-chế-lõi có-chủ-đích** (whitepaper §7.2 dòng 227 use-it-or-lose-it), KHÔNG phải phạt-yếu. Nó dạy user "ngày nào không dùng là mất" — LAMP phải trao cho **NGƯỜI DÙNG THẬT** (F8, §1 dòng 47). Chỉ đòi phần **locked-chưa-mở**; KHÔNG chạm `unlocked`.
5. **Dump LAMP unlocked = HỢP LỆ.** Lực-giữ-user = **tiêu-thật** (MAGIC neo-dịch-vụ, §4 dòng 92; VP C1/chiết-khấu keyed tiêu-MAGIC, F8). Anti-idle chỉ đòi phần locked-chưa-mở của người-từ-bỏ.

**Quyết định thiết kế (4 trục working-principles):**
- (a) *dài hạn:* vòng tự-nuôi pot (phản-chu-kỳ + anti-idle + treasury) → adoption bền, không cần bơm-quỹ liên tục.
- (b) *first-principles:* phát tài-nguyên-nền (LAMP) không phát đồng-chi-phí (ADA); phí mạng trừu-tượng vào tầng Feecover.
- (c) *tối ưu:* tái dùng `dist_treasury.ak` (pot), `lock.ak` (vesting-lock), engine Gen sẵn có. On-chain tối thiểu.
- (d) *user + bền vững:* user chỉ thấy 1 nút (GetLAMP), không cần hiểu tỷ giá; kẻ-bỏ-cuộc + phí-user-trước tự-nạp-pot nuôi người-mới.

---

## 2. Luồng end-to-end

### 2.1 Sơ đồ trạng thái (vault đời một user) — hai đồng-hồ đều theo NGÀY

```
[Enclave keygen]──vân tay──►[Ví Phoenix tạo]
        │
        │ bấm GetLAMP  (backend orchestration — Long)
        ▼
[pot User] ── chuyển D LAMP ──►[Vault vesting: locked=D, unlocked=0]
        │                              │
        │                              │ mỗi đêm cuối NGÀY: unlock 1 LAMP (slot/86400)
        │                              ▼
        │                    [Vault: locked=D−t, unlocked=min(t,D)]   (t = số NGÀY trôi)
        │                              │
        │            ┌─────────────────┼──────────────────┐
        │            ▼ (LAMP đã mở)    ▼                   ▼
        │       [Bán tự do]      [ScheduleGen]        [InstantGen]
        │       (transferable —  (khoá LAMP,          (tiêu ngay,
        │        HỢP LỆ, G2)      cam kết → MAGIC)     decay → MAGIC)
        │                              │                   │
        │                              └────────┬──────────┘
        │                                       ▼
        │                              [MAGIC = số dư account trong Vault]
        │                                       │ tiêu dịch vụ (ConsumeMAGIC)
        │                                       ▼
        │                          [Feecover: carp_paid==fee_magic]  (CARP user ĐÃ CÓ)
        │                                       │ gom CARP theo tầng
        │                                       ▼
        │                          [GreenBack backed-path (§7 WP) ── κ_eff/br ──►]
        │                                       ▼
        │                          [Phoenix Treasury nhận CARP-có-backing]
        │
        │ anti-idle (job NGÀY — Long): nếu 1 NGÀY KHÔNG có ≥1 tiêu-MAGIC-counterparty
        └──◄── thu 1 LAMP từ phần locked chưa-mở ─── nạp về pot User (nuôi user mới)

[pot nạp phản-chu-kỳ]: phí user-trước thu bằng LAMP theo giá-trị → giá LAMP xuống → thu nhiều LAMP → pot ↑
[GetMAGIC] ── mua CARP bằng fiat ──► tích CARP chi tiêu dần (cửa mint CARP hợp lệ — cầu-vào-thật)
```

### 2.2 Các bước (khớp §3 toán)

1. **Keygen (Core/Enclave).** Vân tay → Secure Enclave sinh khoá → ví Phoenix. Không rời máy. (Ranh giới Core — §5.)
2. **GetLAMP (backend — Long).** Đọc `pot_balance` → `D = min(1001, ⌊pot_balance/1_000_000⌋)` (§3.1) → dựng tx: (a) spend pot chuyển `D` LAMP; (b) tạo **Vault vesting** `locked=D, unlocked=0, vest_start_slot=current, did_commit` gắn DID (§3.2); (c) khoá chặt. Ký uỷ quyền pot bằng authority pot (`dist_treasury` pattern, §3.6).
3. **Vesting 1/NGÀY (on-chain gate).** Mỗi đêm cuối NGÀY đúng **1 LAMP** chuyển `locked → unlocked` (§3.2). LAMP đã mở = `LoyaltyHolding` mở khoá (`lock.ak`), dùng cho Gen hoặc bán.
4. **Sinh MAGIC (repo MAGIC — hard-depend).** LAMP đã mở → **ScheduleGen** (khoá LAMP kỳ-hạn, cam kết `M_i`/epoch) HOẶC **InstantGen** (tiêu ngay, decay). Activation-bonus **gắn ScheduleGen** (§3.4, G4).
5. **Tiêu MAGIC (ConsumeMAGIC + Feecover).** Tiêu = **giảm số dư MAGIC account** (F1, §10 dòng 350). Feecover ép `carp_paid == fee_magic`. Settlement backing đi qua **GreenBack backed-path** (§3.5) → CARP-có-backing về **Phoenix Treasury**. **KHÔNG mint CARP tự do.**
6. **Anti-idle (job NGÀY — Long).** Cuối mỗi NGÀY, nếu vault KHÔNG phát sinh ≥ `MIN_MAGIC_TX` tiêu-MAGIC **có counterparty-DID ≠ owner** (§3.3, G3) → thu **1 LAMP** từ `locked` chưa-mở → trả về **pot User** (§3.6). Grace ngắn onboarding (§3.4); tuyến tính, KHÔNG compound; dừng-thu khi tiêu-lại.
7. **GetMAGIC (backend — Long).** User mua **CARP bằng fiat** → tích chi tiêu dần. **Cửa mint CARP hợp lệ** (cầu-vào-thật), khác hẳn "mint từ tiêu MAGIC" (F1 cấm).

---

## 3. Toán

### 3.1 D — lượng LAMP phát mỗi GetLAMP

Anh chốt: "1 phần triệu số dư còn lại, không vượt 1001".

```
D = min( D_CAP , ⌊ pot_balance / SCALE ⌋ )

  D_CAP = 1001          (LAMP)   -- TRẦN; pot tự-nuôi nên D tiến về trần này (§3.6)
  SCALE = 1_000_000     (LAMP)   -- "1 phần triệu số dư còn lại"
  pot_balance           (LAMP)   -- số dư pot User tại thời điểm GetLAMP
```

- **Đơn vị on-chain = oildrop** (`InstantGen/MATH.md §1`: `oildrop = LAMP × 10⁶`). `D_oildrop = D × 10⁶`; `SCALE_oildrop = 10¹²`. Số học **integer** (floor div), không float.
- **Ngưỡng bão hoà:** `⌊pot/1e6⌋ ≥ 1001 ⟺ pot ≥ 1_001_000_000 LAMP`. Trên ngưỡng mọi user nhận đúng 1001 (trần).
- **Pot KHÔNG cạn (§3.6):** 3 nguồn nạp (phản-chu-kỳ + anti-idle + treasury) giữ pot dương và tiến về ngưỡng-bão-hoà. **OPEN A cũ (xử lý D=0 pot-cạn) BỎ** — không cần hàng-đợi/từ-chối. (Nếu trong pilot pot tạm thấp, D co theo pot là hành-vi-tự-điều-tiết tạm thời, không phải trạng-thái-cần-xử-lý-riêng.)
- **Ví dụ:** `pot=5_000_000_000` → `min(1001,5000)=1001`. `pot=300_000_000` → `D=300` (tạm, khi pilot mới khởi động; 3 nguồn nạp kéo pot lên).

### 3.2 Lịch vesting — unlock 1 LAMP/NGÀY (đêm cuối mỗi NGÀY)

```
unlocked(t) = min( t , D )              (LAMP)
locked(t)   = D − unlocked(t) = max(0, D − t)

  t = số "đêm cuối NGÀY" đã trôi qua kể từ vest_start   (t ≥ 0, integer)
```

- **Đồng-hồ-NGÀY (slot/86400):** unlock chốt theo ranh-giới NGÀY, KHÔNG theo epoch Cardano (1 epoch ≈ 5 ngày, không khớp "1/ngày"). Validator không đọc wall-clock → dùng `validity_range` (slot): `t = ⌊(tx_lower_slot − vest_start_slot) / SLOTS_PER_DAY⌋`, `SLOTS_PER_DAY = 86_400` (1 slot = 1 giây). Validator ép `unlocked_out ≤ min(t, D)` — mở KHÔNG NHANH HƠN nhịp (monotonic, không rút-trước).
- **Tổng thời gian vest:** `D` ngày (D ≤ 1001 → ≤ 1001 ngày ≈ **2,74 năm** cho user pot-đầy). **Đây là CHỦ ĐÍCH** (anh chốt OPEN B): giữ user quanh vòng dịch-vụ dài hạn. KHÔNG rút ngắn.
- **Tái dùng `lock.ak`:** `locked`/`unlocked` biểu diễn bằng `LoyaltyHolding{amount, acquired_epoch, is_locked}` (`ScheduleGen/lock.ak:8`). Mở 1/ngày = chuyển 1 LAMP `is_locked:True → False`. KHÔNG phát minh struct mới.
- **[CHỐT B]** unit unlock = **1 LAMP/NGÀY** (anh chốt). D=1001 ⇒ 1001 ngày là CHỦ ĐÍCH, không phải side-effect cần sửa.

### 3.3 Anti-idle — điều kiện đếm (chỉ counterparty-tx, đồng-hồ-NGÀY)

Trong NGÀY `n`, gọi:

```
active(profile, n) = ( M_profile(n) ≥ MIN_MAGIC_TX )

  M_profile(n)  = TỔNG MAGIC tiêu trong NGÀY n trên TOÀN PROFILE của user
                  (mọi DID cá-nhân của chính user), THOẢ counterparty_did ≠ owner_did
                  (G3 — chống self-loop/wash);
                  ✗ KHÔNG gồm MAGIC tiêu qua các DID user chỉ CÓ THẨM QUYỀN
                    (Org / DID được uỷ-quyền) — không tính vào anti-idle cá nhân.
  MIN_MAGIC_TX  = 10% × MAGIC_genable( granted_LAMP )        [TẠM — anh đặt 2026-07-06]
                  granted_LAMP = LAMP vesting ĐƯỢC PHÁT (pot → vault),
                                 KHÔNG gồm LAMP user TỰ MUA.
```

- **Đồng-hồ-NGÀY:** anti-idle đếm theo NGÀY (slot/86400), **cùng đồng-hồ với vesting** (§3.2). Mỗi NGÀY không phát sinh dùng → thu 1 LAMP (§3.4). (v1 đếm theo epoch — **đã sửa sang NGÀY** theo anh chốt.)
- **Chỉ đếm counterparty-DID ≠ owner** (G3; `VeData-Reply §4`). Tự-sinh-MAGIC rồi tự-tiêu vào dịch-vụ mình kiểm soát **KHÔNG tính**. **Hard-depend MAGIC-team:** `did_commit` per-DID (hiện `#""` sentinel) + field **counterparty** trong consume-event (kéo lên **Phase 1**). Whitepaper F8 (§10 dòng 357) hậu thuẫn nguyên-lý cross-DID ("đọc MAGIC tiêu cross-DID chặn tự-burn-vòng"). Xem §6.
- **Lực-giữ chính KHÔNG phải điều kiện này.** Điều kiện này chỉ quyết "có thu-hồi phần locked-chưa-mở không". Lực-giữ = tiêu-thật (F8). Anti-idle = **thu-hồi phần chưa-trao của người bỏ cuộc**, không phải hình phạt lên tài sản đã mở.
- **Đo trên TOÀN PROFILE, KHÔNG chỉ vault NewUser** (anh chốt 2026-07-06): user chỉ cần **tổng tiêu-MAGIC cả profile/NGÀY ≥ MIN_MAGIC_TX** là active — bất kỳ dùng-thật nào trong danh-tính cá-nhân đều giữ vesting. **Loại trừ** MAGIC tiêu qua DID user chỉ có-thẩm-quyền (Org/uỷ-quyền) — để hoạt-động-doanh-nghiệp không "gánh hộ" anti-idle cá-nhân.
- **`MIN_MAGIC_TX = 10% MAGIC gen-able từ granted_LAMP`** (TẠM — anh đặt). `granted_LAMP` = phần vesting được phát, KHÔNG gồm LAMP user tự mua → ngưỡng KHÔNG phình theo lượng user nạp thêm. **[SUB-OPEN]** cơ-sở gen-able là *daily-gen-able (từ LAMP mở trong ngày)* hay *cumulative* — chờ anh làm rõ granularity; mặc định spec dùng daily-gen-able để ngưỡng đồng-nhịp anti-idle-NGÀY.

### 3.4 Anti-idle — lượng thu-hồi (tuyến tính, không compound, grace NGÀY)

```
reclaim(vault, n) =
  if n < vest_start_day + GRACE_DAYS       → 0                    (grace onboarding, theo NGÀY)
  else if active(vault, n)                 → 0                    (tiêu-lại → dừng thu)
  else if locked(vault) ≥ RECLAIM_UNIT     → RECLAIM_UNIT         (thu tuyến tính)
  else                                      → 0                    (hết locked → không thu nữa)

  RECLAIM_UNIT = 1        (LAMP / NGÀY idle)                       (tuyến tính, KHÔNG ×2)
  GRACE_DAYS   = 3..7     (NGÀY đầu miễn thu — đề xuất, xem lý do)  [CHỐT: 7 ngày]
```

- **Grace theo NGÀY (không epoch), CHỐT 7 ngày.** Lý do: (a) đồng-hồ vesting/anti-idle đều là NGÀY → grace phải cùng đơn-vị; (b) onboarding user mới (keygen → hiểu GetLAMP → tiêu dịch-vụ-counterparty lần đầu) cần vài ngày cho hạ-tầng counterparty/Feecover thông; (c) 7 ngày = 1 tuần, ngắn gọn, đủ để không phạt user-mới-đang-học nhưng KHÔNG mâu thuẫn "ngày nào không dùng là mất" — vì grace chỉ áp **cửa-sổ-onboarding-một-lần** lúc mới nhận vault, sau đó nguyên-lý use-it-or-lose-it áp đầy đủ. (Cân nhắc bỏ grace: mâu thuẫn nhẹ với "ngày nào không dùng là mất" nhưng phạt-user-mới-chưa-kịp-học là phản-onboarding → GIỮ grace ngắn 7 ngày là cân-bằng đúng.)
- **Tuyến tính, KHÔNG compound:** mỗi NGÀY-idle thu đúng `RECLAIM_UNIT`, không luỹ tiến.
- **Chỉ thu từ `locked` (chưa-mở).** KHÔNG chạm `unlocked` (đã thuộc về user). `locked −= reclaim`.
- **LAMP đã thu KHÔNG hồi lại vault** nhưng **dừng thu ngay khi tiêu-lại** — user quay lại tiêu-thật thì phần locked còn lại tiếp tục vest.
- **Đích đến:** `reclaim → pot User` (§3.6). Vòng tự-nuôi.
- **Hệ quả:** user thật chỉ cần ≥`MIN_MAGIC_TX` giao-dịch-counterparty/NGÀY → giữ trọn vesting. User "ôm-LAMP-không-tiêu" từ-từ trả phần chưa-mở về pot. Dump-toàn-bộ-unlocked-rồi-bỏ vẫn giữ phần đã mở, chỉ mất phần chưa-mở — hợp lý (G2).

### 3.5 Settlement phí — định-tuyến qua GreenBack backed-path (KHÔNG mint CARP tự do)

Tiêu 1 dịch vụ:

```
(i)   ConsumeMAGIC:  magic_account_balance −= fee_magic(service_id)   -- giảm số dư, KHÔNG mint/burn (F1)
(ii)  Feecover:      require carp_paid == fee_magic                    -- neo 1:1 (FEECOVER-PAY-1)
                     carp_paid = CARP user ĐÃ CÓ (mua qua GetMAGIC)    -- KHÔNG đúc mới
(iii) backing:       CARP-fee ── GreenBack backed-path (§7 WP, κ_eff/br) ──► Phoenix Treasury (CARP-có-backing)
```

- **CARP KHÔNG mint khi tiêu** (F1 §10 dòng 350). User **trả CARP đã-có** (mua qua GetMAGIC = cầu-vào-thật). Tiêu MAGIC chỉ giảm số dư account.
- **Backing quyết-toán qua GreenBack backed-path** (whitepaper §7.2 dòng 229): GreenBack `B` = LAMP (định-giá-oracle + **chiết-khấu-an-toàn theo biến động**) + tài sản cứng (ADA/stable), với **3 phanh** (van-đỏ `cap=0 khi br≤br_safe` + trần-kép + cổng-thặng-dư). Đây là đường **có-back**, KHÔNG phải mint-tự-do. Whitepaper tự nhận residual-reflexivity (collateral GreenBack = LAMP nội-sinh) nhưng khử bằng chiết-khấu + 3 phanh + stress-test — đánh-đổi-đã-khai-báo (§13 dòng 426), KHÔNG phải lỗ Terra.
- **Ranh giới cứng:** PhoenixKey **CHỈ khai interface tới GreenBack**, KHÔNG re-spec GreenBack (địa hạt đội CARP). Bám F9 (§10 dòng 358): TUYỆT ĐỐI không đỡ-peg bằng LAMP; không chạm `INV-PEG-BY-DEMAND`, `1 CARP = 1 MAGIC`.

**Interface Activation ↔ GreenBack (chỉ khai báo, không implement):**

```
greenback_settle(carp_amount, meta) -> SettleReceipt        // Phoenix Treasury ↔ GreenBack (§7 WP)
  carp_amount = Σ CARP-fee gom cuối NGÀY (Feecover §6.2)
  meta        : { source:"activation-feecover", day, service_breakdown }
  → GreenBack quyết κ_eff/br (backed-path §7), trả phần CARP-có-backing về Phoenix Treasury
  → KHÔNG mint/burn tự do ở phía Activation; KHÔNG đọc oracle-giá để cầm-lái cơ-chế (F6)
```

### 3.6 Pot cân bằng — 3 nguồn nạp, KHÔNG cạn (phản-chu-kỳ)

```
pot_balance(n+1) = pot_balance(n)
                   − Σ D(user)                        [RÚT]  GetLAMP mỗi user mới
                   + Σ reclaim(vault, n)               [NẠP]  anti-idle thu-hồi hằng NGÀY (§3.4)
                   + fee_refill_lamp(n)                [NẠP]  phí user-trước thu bằng LAMP theo giá-trị (phản-chu-kỳ)
                   + treasury_topup(n)                 [NẠP]  MagicLamp Treasury bơm (khi cần)
```

- **3 nguồn nạp (anh Aladin chốt — pot KHÔNG BAO GIỜ cạn):**
  1. **Phí hoạt động của user trước — `fee_refill_lamp` (phản-chu-kỳ, CHỐT F).** Phí giao dịch **cố định theo GIÁ-TRỊ** (MAGIC định-giá ổn-định theo dịch-vụ, §4 dòng 92) nên **thu bằng LAMP**: giá LAMP càng xuống → cùng-giá-trị-phí thu được càng NHIỀU LAMP → **pot (D) tăng**. Chống-chu-kỳ: LAMP rớt (thị trường lạnh) → pot phình → phát nhiều cho user-mới đúng lúc cần. Cùng-họ với counter-cyclical GreenBack (§7.3 dòng 240).
  2. **Anti-idle thu-hồi hằng NGÀY** (§3.4) — người-bỏ-cuộc → pot → người-mới.
  3. **MagicLamp Treasury — `treasury_topup`** — bơm khi pot dưới ngưỡng-sức-khoẻ.
- **[GHI TRUNG THỰC — whitepaper-gap]** đường `fee_refill_lamp` (thu-phí-pot bằng LAMP theo giá-trị) là **directive anh Aladin**; whitepaper §7 chưa spell-out đường pot-refill-Activation này (§7.3 dòng 240 nói GreenBack mua-LAMP-đáy, KHÁC). Không mâu thuẫn — mở-rộng-cấp-Activation. Tham số tỷ-lệ-trích + cơ-chế-quy-đổi (phí thu ở đâu → LAMP về pot) chờ chốt chi tiết.
- **Tái dùng `dist_treasury.ak`** (`LAMP/Genesis/onchain/validators/dist_treasury.ak:14-27`): pot = script `dist_treasury(authority)`, release đòi `authority` ký. Bootstrap 1-pkh; bản thật thay bằng claim drip theo vesting. Pot User = instance `dist_treasury` với authority = GetLAMP-orchestrator.
- **Bất biến bảo toàn:** LAMP không tự-sinh trong vòng Activation. `Σ D + pot_cuối = pot_đầu + Σ nạp` (I-ACT-5). Nạp từ nguồn thật (reclaim/fee-LAMP/treasury), không mint-ẩn.

---

## 4. Invariant (I-ACT-x)

| ID | Bất biến | Bám |
|---|---|---|
| **I-ACT-1** | **Vault khoá đúng ban đầu:** ngay sau GetLAMP `unlocked == 0 ∧ locked == D`; toàn bộ D `is_locked:True`. | `lock.ak`; §3.2 |
| **I-ACT-2** | **Unlock đúng nhịp NGÀY (monotonic, không rút-trước):** `unlocked_out ≤ min(t, D)`, `t = ⌊(slot−vest_start_slot)/86400⌋`; `unlocked` chỉ tăng (trừ tiêu-hợp-lệ qua Gen/bán); mở KHÔNG nhanh hơn 1/NGÀY. | §3.2 |
| **I-ACT-3** | **Anti-idle chỉ đếm counterparty-tx:** `active` chỉ tính tiêu-MAGIC `counterparty_did ≠ owner_did`; self-loop KHÔNG tính. | G3; §3.3; hard-depend §6 |
| **I-ACT-4** | **Thu-hồi theo NGÀY, tuyến tính, chỉ từ locked, có grace, không compound:** `reclaim ≤ RECLAIM_UNIT` mỗi **NGÀY**-idle; chỉ trừ `locked`, KHÔNG chạm `unlocked`; `n < vest_start_day+GRACE_DAYS → reclaim=0`; `active → reclaim=0`. Cùng đồng-hồ-NGÀY với vesting (slot/86400). | G5; §3.4 |
| **I-ACT-5** | **Pot bảo toàn (conservation):** LAMP không tự-sinh; `Σ D_rút = Σ nạp + Δpot`; mọi nạp từ nguồn thật (reclaim/fee-LAMP/treasury), KHÔNG mint-ẩn. Pot tự-nuôi → không cạn. | §3.6 |
| **I-ACT-6** | **D-cap 1001:** `D ≤ D_CAP = 1001` LAMP; `D = min(1001, ⌊pot/1e6⌋)`; pot tự-nuôi → D tiến về trần. | §3.1 |
| **I-ACT-7** | **Settlement định-tuyến qua GreenBack backed-path (KHÔNG mint CARP tự do):** mỗi tiêu-dịch-vụ `carp_paid == fee_magic` từ CARP-user-đã-có; backing đi qua **GreenBack** (LAMP-haircut + tài-sản-cứng + 3-phanh, §7 WP dòng 229); Activation KHÔNG mint/burn CARP/LAMP tự do, KHÔNG đọc oracle-giá cầm-lái (F6). KHÔNG phải "mint tự do" — đường có-back. | G1; §3.5; WP §7.2 dòng 229, F1 |
| **I-ACT-8** | **Vault gắn DID (attribution):** `did_commit = blake2b_256(did ‖ salt)` bảo toàn qua vesting/anti-idle; một DID không mở nhiều vault né trần (khi `did_commit` per-DID sẵn sàng). | `VeData-Reply §2`; §6 |
| **I-ACT-9** | **Lực-giữ = tiêu-thật, không phạt:** không invariant nào phạt holding thuần; dump `unlocked` HỢP LỆ; chỉ `locked-chưa-mở` bị thu khi idle. | G2; F8 `INV-MAGIC-CITIZEN` |

---

## 5. Ranh giới triển khai

| Tầng | Việc | Đội |
|---|---|---|
| **On-chain (Aiken)** | **Validator vault-vesting** (locked/unlocked, unlock-gate slot/NGÀY I-ACT-1/2, anti-idle-reclaim theo NGÀY I-ACT-4, `did_commit` gate I-ACT-8) + **validator pot** (kế thừa `dist_treasury.ak`, release-gate authority/drip, nạp 3 nguồn I-ACT-5). Tái dùng `lock.ak`. | **Tuân** |
| **Backend (Java)** | **GetLAMP orchestration** (đọc pot → tính D §3.1 → dựng tx phát+khoá) + **anti-idle job NGÀY** (đọc consume-event counterparty §3.3 → dựng tx reclaim→pot) + **fee_refill_lamp** (thu phí user theo giá-trị bằng LAMP → pot, phản-chu-kỳ §3.6) + **GetMAGIC** (fiat→CARP). `curl` verify sau deploy. | **Long** |
| **Core / Enclave (Flutter+Rust)** | Keygen vân tay → ví Phoenix; ký tx GetLAMP/Gen/consume bằng controller-DID; UI 1-nút GetLAMP + hiển thị vesting drip + pot-health. | **Core** |
| **Phụ thuộc MAGIC-team** | (1) `did_commit` per-DID trong `EngageDatum` (hiện `#""`); (2) **counterparty field** consume-event (kéo **Phase 1**); (3) ScheduleGen/InstantGen (đã có). Activation **tiêu thụ**, không dựng lại. | MAGIC |
| **Phụ thuộc CARP-team** | **GreenBack interface** (`greenback_settle` §3.5) — thu CARP-fee theo `κ_eff`/`br` (backed-path §7), trả CARP-có-backing về Phoenix Treasury. PhoenixKey CHỈ khai interface, KHÔNG re-spec GreenBack, KHÔNG chạm peg (F5/F9). | CARP |

---

## 6. Phụ thuộc chéo (bắt buộc nêu rõ — Phase 1)

**6.1 `did_commit` + counterparty (MAGIC-team — BLOCKER cho I-ACT-3).**
- Hiện `EngageDatum` chỉ có `owner`, `did_commit = #""` sentinel (`Feecover §8.1`). Cần: (a) `did_commit : ByteArray = blake2b_256(did ‖ salt)` (`VeData-Reply §2`); (b) consume-event mang **counterparty_did**. Không có (b) → anti-idle không phân biệt self-loop (G3 hở). **Kéo counterparty lên Phase 1.** Whitepaper F8 (§10 dòng 357) đồng nguyên-lý cross-DID.

**6.2 Resolve API point-in-time (PhoenixKey-Database — Long).**
- Anti-idle + vault-DID-gate cần `did_active(did, day)` / `key_authorized(key, did, day)` point-in-time. Hiện DB mới có state-hiện-tại; point-in-time là phần Long implement.

**6.3 GreenBack interface (CARP-team — I-ACT-7).**
- `greenback_settle` (§3.5) — CARP-team định `κ_eff`/`br` waterfall backed-path (`Carpet-CARP §4.1`, whitepaper §7). PhoenixKey chỉ gửi CARP-fee-gom + nhận CARP-có-backing. KHÔNG chạm bất biến peg.

**6.4 Vesting-release kho→pot (LAMP-team — nạp pot lần đầu).**
- Pot User nạp-vốn-ban-đầu từ MagicLamp Treasury / kho `dist_treasury`. Cần LAMP cấp địa chỉ kho + `kho_nft` + authority.

**6.5 Cơ-chế thu-phí-bằng-LAMP → pot (LAMP/backend — cho phản-chu-kỳ).**
- `fee_refill_lamp` (§3.6) cần: điểm thu phí user-trước, quy-đổi giá-trị-phí → lượng LAMP theo giá-trị, đường LAMP về pot. Directive anh Aladin; tham số + cơ-chế-quy-đổi chờ chốt.

---

## 7. Tham số then chốt

| Tham số | Ký hiệu | Giá trị / khoảng | Trạng thái |
|---|---|---|---|
| Trần D | `D_CAP` | **1001 LAMP** | CHỐT |
| Chia-pot | `SCALE` | **1_000_000 LAMP** ("1 phần triệu") | CHỐT |
| Nhịp unlock | — | **1 LAMP / NGÀY**, đêm cuối (D=1001 → 1001 ngày ≈ 2,74 năm, CHỦ ĐÍCH) | **CHỐT B** |
| Slots/ngày | `SLOTS_PER_DAY` | **86_400** (Cardano 1s/slot) | CHỐT (hằng Cardano) |
| Grace onboarding | `GRACE_DAYS` | **7 NGÀY** (cửa-sổ-onboarding-một-lần) | **CHỐT E** |
| Thu-hồi/NGÀY idle | `RECLAIM_UNIT` | **1 LAMP** (tuyến tính, không compound) | CHỐT |
| Đồng-hồ anti-idle | — | **NGÀY** (slot/86400), cùng vesting | CHỐT |
| Nguồn nạp pot | — | phản-chu-kỳ (fee-LAMP) + anti-idle + treasury; pot KHÔNG cạn | **CHỐT F** |
| Đếm active | `MIN_MAGIC_TX` | **10% MAGIC-gen-able từ granted_LAMP** (không gồm LAMP tự mua); đo TỔNG tiêu-MAGIC cả PROFILE/NGÀY, trừ DID-thẩm-quyền | **TẠM (anh đặt 07-06)** · sub-open: granularity gen-able |
| Tỷ-lệ + cơ-chế thu-phí-LAMP→pot | `fee_refill_lamp` | quy-đổi giá-trị-phí → LAMP về pot | chờ chốt chi tiết (§6.5) |
| GreenBack ref | `κ_eff` | `clamp(0.6 − a·σ̂ − b·max(0,br_safe−br), 0.43, 0.6)` — **CARP-team giữ** | tham chiếu (ngoài PhoenixKey) |

**Đã BỎ so với v1:** OPEN A (pot-cạn / D=0) — pot tự-nuôi không cạn. **Đổi so với v1:** anti-idle + grace từ epoch → **NGÀY**; OPEN E/F từ mở → **CHỐT**.

---

## 8. Phụ thuộc & thứ tự triển khai

**Build được NGAY (cái có sẵn):**
1. **Validator pot** — kế thừa `dist_treasury.ak` (đã có, đã test). Cần authority + nạp-vốn-đầu (§6.4).
2. **Validator vault-vesting (D + unlock 1/NGÀY)** — tái dùng `lock.ak` holdings + slot-gate. KHÔNG cần MAGIC/CARP cho phần vesting thuần.
3. **GetLAMP backend** — đọc pot, tính D (§3.1), dựng tx phát+khoá. Độc lập MAGIC/CARP.
4. **D-cap + conservation test** (I-ACT-5/6) — test số học thuần.

**CHỜ MAGIC-team:**
5. **Anti-idle counterparty-gate** (I-ACT-3) — hard-depend `did_commit` per-DID + counterparty (§6.1). Trước khi có: **KHÔNG bật anti-idle production** (self-loop G3 hở).
6. **Sinh MAGIC** — ScheduleGen/InstantGen đã có; cần LAMP-đã-mở làm input. Activation-bonus gắn ScheduleGen (§3.4).

**CHỜ CARP-team:**
7. **GreenBack settlement** (I-ACT-7) — `greenback_settle` backed-path (§3.5/§6.3). Trước khi có: Feecover gom CARP về Treasury (`Feecover §6.2`) nhưng bước backing-qua-GreenBack treo interface.

**CHỜ LAMP/backend (phản-chu-kỳ):**
8. **`fee_refill_lamp`** (§6.5) — thu-phí-user bằng LAMP → pot. Trước khi có: pot nạp bằng anti-idle + treasury (đủ chạy pilot); phản-chu-kỳ đầy-đủ khi cơ-chế-thu-phí-LAMP sẵn.

**Thứ tự:** (1→2→3→4) build+test độc lập → (6) nối Gen → (5) bật anti-idle khi counterparty sẵn → (7) nối GreenBack → (8) nối phản-chu-kỳ. GetMAGIC (fiat→CARP) song song.

---

## 9. Phản biện đối kháng (self-adversarial)

**A. Sybil-farm nhiều ví lấy D×N.** — NGOÀI PHẠM VI (sinh trắc SE đủ). `did_commit` per-DID chặn 1-DID-nhiều-vault né trần (I-ACT-8), nhưng uniqueness-DID không phải việc Activation.

**B. Wash-generation né anti-idle (G3 — lỗ thật).** — I-ACT-3 chỉ đếm counterparty-DID≠owner. Self-loop KHÔNG tính. Hở nếu MAGIC-team chưa gắn counterparty (§6.1) → **không bật anti-idle production tới khi có** (§8.5). Đây là lỗ thật còn lại cần vá.

**C. Rút-trước vesting (mở nhanh hơn 1/NGÀY).** — I-ACT-2: `unlocked_out ≤ min(t,D)`, `t` từ slot thật (`validity_range`). Validator từ chối mở vượt nhịp.

**D. Thu-hồi tịch-thu nhầm tài sản đã trao.** — I-ACT-4: reclaim CHỈ từ `locked` (chưa-mở). `unlocked` (đã bán/đã Gen) không đụng. User dump-hết-unlocked-rồi-bỏ chỉ mất phần chưa-mở — đúng thiết kế (G2 hợp lệ).

**E. Máy-in-tiền qua mint CARP (nghi Terra).** — I-ACT-7: tiêu MAGIC KHÔNG mint CARP; user trả CARP-đã-có (fiat qua GetMAGIC); backing qua GreenBack backed-path (LAMP-haircut + tài-sản-cứng + 3-phanh, §7 WP dòng 229). Whitepaper tự-nhận residual-reflexivity nhưng khử bằng haircut+3-phanh+stress-test (đánh-đổi khai-báo §13 dòng 426) — KHÔNG phải Terra. Tx-rác chỉ tự-đốt CARP user → tự-phạt.

**F. Pot drain / phát D vượt trần / pot cạn.** — I-ACT-5 (conservation) + I-ACT-6 (D-cap 1001) + pot release-gate authority. **Pot KHÔNG cạn**: 3 nguồn nạp (phản-chu-kỳ + anti-idle + treasury) giữ pot dương. Phản-chu-kỳ đặc-biệt: LAMP rớt → pot phình → onboarding chạy đúng lúc thị trường lạnh.

**G. Vesting-drift + anti-idle-drift (đồng-hồ).** — §3.2/§3.4: **cả vesting LẪN anti-idle cùng đồng-hồ-NGÀY** (slot/86400) — nhất quán, không lệch nhịp. (v1 để anti-idle theo epoch — đã sửa sang NGÀY theo anh chốt.)

**H. Anti-idle phạt user-nhàn-rỗi-thật (nghi phản-onboarding).** — anti-idle chỉ đòi `locked-chưa-mở` (chưa thuộc user), có grace 7 ngày onboarding, dừng-thu khi tiêu-lại. Đây là **cơ-chế-lõi có-chủ-đích** (use-it-or-lose-it, §7.2 dòng 227): dạy "ngày nào không dùng là mất", redistribute LAMP cho user-thật (F8). KHÔNG phải phạt tài-sản-đã-trao.

---

## 10. Tóm tắt cho reviewer

- Activation = **GetLAMP → vault vesting khoá-chặt → mở 1 LAMP/NGÀY → Gen MAGIC → tiêu qua Feecover/GreenBack backed-path → Treasury**; **GetMAGIC** = fiat→CARP (cửa mint hợp lệ). Bỏ model cũ nạp-VND-lấy-LAMP+ADA.
- `D = min(1001, ⌊pot/1_000_000⌋)` LAMP — trần 1001; **pot tự-nuôi KHÔNG cạn** (3 nguồn: phản-chu-kỳ + anti-idle + treasury).
- Vesting **1 LAMP/NGÀY** (slot/86400, CHỐT B); D=1001 → 1001 ngày ≈ 2,74 năm là CHỦ ĐÍCH. Tái dùng `lock.ak`; monotonic không-rút-trước (I-ACT-2).
- **Anti-idle + vesting CÙNG đồng-hồ-NGÀY** (slot/86400). Anti-idle = cơ-chế-lõi có-chủ-đích (use-it-or-lose-it, WP §7.2 dòng 227): mỗi NGÀY không dùng → thu 1 LAMP từ `locked-chưa-mở` → pot. Grace 7 ngày, tuyến tính, không compound, dừng-thu khi tiêu-lại. **Dump unlocked HỢP LỆ** (I-ACT-9).
- **Settlement (I-ACT-7):** KHÔNG mint CARP tự do — user trả CARP-đã-có, backing qua **GreenBack backed-path** (WP §7.2 dòng 229, LAMP-haircut + tài-sản-cứng + 3-phanh). PhoenixKey chỉ khai interface, không re-spec GreenBack, không chạm peg (F5/F9).
- **Phản-chu-kỳ (CHỐT F):** phí thu bằng LAMP theo giá-trị → giá LAMP xuống → thu nhiều LAMP → pot ↑. Điểm mạnh chống-chu-kỳ.
- **Lỗ thật còn lại = G3** (wash/self-loop): anti-idle CHỈ đếm counterparty-DID≠owner — hard-depend MAGIC-team `did_commit`+counterparty (Phase 1).
- 9 invariant I-ACT-1..9; on-chain (Tuân) + backend (Long) + Core/Enclave + phụ thuộc MAGIC/CARP/LAMP-team.

---

## Phụ lục — Quyết định (chốt vs OPEN)

| # | Mục | Trạng thái |
|---|---|---|
| A | ~~Hành vi khi D=0 (pot cạn)~~ | **BỎ** — pot tự-nuôi không cạn |
| B | Unit unlock = 1/NGÀY (D=1001 → 1001 ngày) | **CHỐT** (chủ đích, giữ user dài hạn) |
| D | `MIN_MAGIC_TX` ngưỡng active | **TẠM CHỐT (07-06):** 10% MAGIC-gen-able từ granted_LAMP (không gồm LAMP tự mua); đo TỔNG tiêu-MAGIC cả PROFILE/NGÀY, trừ DID-thẩm-quyền. Sub-open: granularity daily-vs-cumulative gen-able |
| E | `GRACE_DAYS` | **CHỐT 7 NGÀY** (cửa-sổ-onboarding-một-lần) |
| F | Nguồn nạp pot (phản-chu-kỳ + anti-idle + treasury) | **CHỐT** (cơ-chế; tham-số fee-LAMP chờ chi tiết §6.5) |
