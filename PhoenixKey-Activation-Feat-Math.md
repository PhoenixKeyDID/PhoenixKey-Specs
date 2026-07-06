# PhoenixKey — Activation (Feat + Math) · v4 (model 2-PHA: có-điều-kiện → vesting-sở-hữu)

> **Nguồn chuẩn duy nhất:** `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` (anh Aladin chốt) + **cơ chế Gen MỚI ở `/Users/ductiger/Projects/CARP`** (`SESSION-STATE-carp-2026-06-26.md`, `CARP-ConsumptionMarket-Vi.md`, `CARP-Math-Vi.md`). Mọi luận điểm neo lại nguồn theo §-số + dòng. Khi mâu thuẫn → theo whitepaper/CARP (và ghi trung thực mâu thuẫn).
> SPEC PROPOSAL (design doc, chờ anh chốt) · **v4 2026-07-06**. KHÔNG code production — chỉ spec. Dẫn `file:line`.
>
> **⚠ v4 THAY HOÀN TOÀN v3.** v3 sai bản chất: dựng "GetLAMP = MƯỢN, LAMP thuộc pot, user KHÔNG sở hữu, CloseBorrow trả về pot, TOÀN BỘ khoản mượn sinh MAGIC mãi mãi". Anh Aladin xác nhận 2026-07-06: **GetLAMP chuyển D LAMP vào VAULT CỦA USER; "mượn" chỉ là NGỮ NGHĨA UX (khoản CÓ ĐIỀU KIỆN tới khi kiếm được) — bản chất LAMP ở trong vault user.** Có **2 PHA rõ rệt**: PHA-1 (ngày 1→1001) LAMP có-điều-kiện, sinh MAGIC (đọc-số-dư), anti-idle rút về pot; **PHA-2 (từ ngày 1002)** phần LAMP sống sót **vest thành SỞ HỮU của user 1 LAMP/ngày** — user rút/bán/tự-Gen tuỳ ý. Khái niệm "LAMP mãi thuộc pot / không-bao-giờ-sở-hữu / CloseBorrow-trả-pot-toàn-bộ" **BỎ HẲN**.

---

## 0. Trạng thái + nguồn bám

**Một câu:** Activation là luồng khởi-tạo-người-dùng của PhoenixKey — user tạo khoá bằng vân tay (Secure Enclave), bấm **GetLAMP** để nhận **D LAMP vào vault CỦA MÌNH** (khoá có-điều-kiện); **số dư LAMP sinh MAGIC** (Gen MỚI ở /CARP — **CHỈ ĐỌC số dư để tính MAGIC, KHÔNG đụng/không burn LAMP**); user hưởng = **MAGIC** tiêu dịch vụ qua **Feecover → GreenBack backed-path → Phoenix Treasury**; **PHA-1 (≤ngày 1001)** anti-idle rút LAMP-chưa-kiếm về pot; **PHA-2 (≥ngày 1002)** phần sống sót **vest 1 LAMP/ngày thành sở-hữu user**; **GetMAGIC** mua CARP bằng fiat để chi tiêu dần.

**Nguồn canonical:**
- `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` — **NGUỒN CHUẨN.** §1 dòng 47 + F8 §10 dòng 357 (`INV-MAGIC-CITIZEN`); §4 (MAGIC neo-dịch-vụ, nanogic=10⁹); **§7.2 dòng 203-227 (InstantGen — "nắm LAMP chỉ MỞ TƯ-CÁCH (gate)... biến quyết định là lượng-MAGIC-tiêu, không phải lượng-LAMP-giữ"; use-it-or-lose-it dòng 227)** — **neo trực-tiếp nguyên lý Gen-đọc-số-dư (không đụng LAMP)**; **§7.3 dòng 235 (ScheduleGen: "LAMP vẫn nằm trong ví họ (không bị mang đi), được trả lại nguyên vẹn khi hết hợp đồng")** — neo LAMP-khoá-nằm-nguyên; §7 dòng 187 (LAMP không có đường ra thụ-động); §10 (F1 MAGIC một-chiều, F9 không-đỡ-peg-LAMP); §13 (đánh-đổi Gen-Terra khai-báo).
- **`/CARP/SESSION-STATE-carp-2026-06-26.md` dòng 86 (Gen MỚI):** "3 Gen MAGIC: PrepaidGen / **ScheduleGen (nắm LAMP + tiêu định kỳ)** / **InstantGen (gộp Snapshot+Instant cũ, tiêu ngay)**. **Gen≠Mint.** MAGIC = quyền-tiêu-1-dịch-vụ-cụ-thể." + dòng 106 (MAGIC thuần Consumable, "nắm LAMP→SnapshotMint; tiêu-ổn-định→ScheduleMint"). **Nguyên lý anh chốt: cả 2 Gen CHỈ ĐỌC số dư LAMP để tính lượng MAGIC — KHÔNG BAO GIỜ burn/tiêu/chuyển LAMP** (yield-trên-số-dư). Whitepaper §7.2 dòng 205 hậu thuẫn trực tiếp (LAMP là GATE, biến quyết định = MAGIC-tiêu).
- `/CARP/CARP-ConsumptionMarket-Vi.md` §1 dòng 15-17 (MAGIC = số kế toán gắn DID, non-transferable, tự-burn-khi-tiêu), §5 dòng 63-72 (shadow-price = chi-phí-thay-thế, dùng khi user PHA-2 tự-Gen bán năng-lực-tiêu-thụ).
- **⚠ Code MAGIC-repo cũ (`MAGIC/ScheduleGen/{FEAT,MATH}.md`) LỖI THỜI cho nguyên lý này:** `FEAT §3.2 dòng 48-51` fire chuyển `lamp_transfer = fires_in_tx × λ` **từ vault → Treasury script** (TIÊU-THỤ LAMP); `FEAT §5 dòng 98` T10 Cancel/refund KHÔNG tồn tại ("LAMP lock đến hết hoặc chuyển dần vào Treasury"). **MÂU THUẪN nguyên lý Gen-đọc-số-dư của /CARP.** Bỏ ground vào code này; xem §3.7 + [cần chốt].
- `MAGIC/ScheduleGen/onchain/lib/magiclamp/protocol/types.ak:31-125` (`LoyaltyHolding{amount, acquired_epoch, is_locked}`, `VaultDatum`) — **tái dùng struct, KHÔNG tái dùng cơ-chế-fire-Treasury.**
- `spec-proposals/_activation-gaming-evaluation.md` (v4) — G1–G8 re-ground model 2-pha; lỗ thật còn lại = G3 (wash).
- `spec-proposals/PhoenixKey-Feecover-Feat-Math.md` §6 (gom CARP → Treasury), §8 (`did_commit` + resolve API).
- `LAMP/Genesis/onchain/validators/dist_treasury.ak:14-27` (pot pattern: script + authority-sig release).
- `spec-proposals/VeData-Reply-did-commit-resolve.md` (`did_commit = blake2b_256(did ‖ salt)`; resolve point-in-time).

---

## 1. Mục tiêu + đổi từ model cũ

**Model VND-Genie (BỎ):** user nạp VND → nhận LAMP + ít ADA. Vấn đề: phát ADA từng user rò-chi-phí; buộc user hiểu tỷ giá; không vòng tự-nuôi.

**Model borrow-only v3 (BỎ ở v4):** "GetLAMP = MƯỢN, LAMP mãi thuộc pot, user KHÔNG sở hữu, CloseBorrow trả về pot". Sai vì (a) không có đường user **kiếm được** LAMP — mãi mãi chỉ hưởng MAGIC-dòng-chảy, không có phần-thưởng-sở-hữu cho người cam-kết đủ 1001 ngày; (b) "TOÀN BỘ khoản mượn sinh MAGIC mãi mãi" không có điểm-dừng-cam-kết; (c) CloseBorrow-trả-toàn-bộ triệt tiêu động-lực trung-thành dài hạn.

**Model 2-PHA (anh Aladin xác nhận 2026-07-06 — v4):**

1. **GetLAMP → vault CỦA USER.** `D = min(1001, ⌊pot/1e6⌋)` LAMP chuyển từ pot **vào vault của user**, khoá **có-điều-kiện**. "Mượn" chỉ là **ngữ nghĩa UX** — user biết khoản này **CÓ ĐIỀU KIỆN** (chưa sở-hữu) tới khi **kiếm được** qua cam-kết. Bản chất: LAMP ở trong vault user. **Đồng hồ: slot mỗi 24h kể từ lúc nhận LAMP về vault** (`vest_start`).
2. **PHA-1 — có-điều-kiện (ngày 1 → 1001, ĐÚNG 1001 ngày):**
   - LAMP nằm trong vault (khoá có-điều-kiện). **CẢ InstantGen + ScheduleGen** (Gen MỚI /CARP) sinh MAGIC — **CHỈ ĐỌC số dư LAMP `conditional_lamp` để tính lượng MAGIC tại thời điểm; KHÔNG đụng/không burn/không chuyển LAMP** (yield-trên-số-dư, WP §7.2 dòng 205).
   - User hưởng = **MAGIC** (tiêu dịch vụ), **KHÔNG có quyền chi-tiêu/rút LAMP** trong pha này.
   - **Anti-idle:** NGÀY nào tiêu-MAGIC `< MIN_MAGIC_TX` → rút **1 LAMP về pot**. `conditional_lamp` giảm; MAGIC-yield tính trên số-dư-hiện-tại.
3. **PHA-2 — vesting sở-hữu (từ ngày 1002):**
   - **Anti-idle DỪNG** (đã kiếm được — qua đủ cam-kết 1001 ngày).
   - Phần LAMP **sống sót** (`conditional_lamp` còn lại) **vest thành SỞ HỮU của user 1 LAMP/ngày** (nhỏ giọt, KHÔNG mở hết). Từ ngày 1002 đủ `(1001 − reclaimed_to_pot)` ngày nữa để vest hết.
   - LAMP đã vest (`vested_unlocked`) = user **thật sự sở hữu** → **rút/bán/tự-Gen tuỳ ý** (rời vault sang ví user; hoặc tự Gen bán năng-lực-tiêu-thụ qua CARP-ConsumptionMarket §5).
4. **Vòng đời:** PHA-1 quay-vòng LAMP-chưa-kiếm về pot (anti-idle) cho user-mới; PHA-2 vest phần-sống-sót thành sở-hữu — **phần-thưởng-trung-thành** cho người cam-kết đủ.
5. **Pot tự-nuôi** (§3.6): nạp từ (a) phí user-trước thu bằng LAMP theo giá-trị (phản-chu-kỳ), (b) anti-idle PHA-1 thu-hồi, (c) MagicLamp Treasury. Lưu ý: PHA-2 LAMP-vest **rời-hệ sang user** (khác v3) → pot conservation phải tính khoản-vest-ra (§3.6).

**Quyết định thiết kế (4 trục working-principles):**
- (a) *dài hạn:* 2-pha tạo **hợp-đồng-trung-thành**: cam-kết 1001 ngày dùng-thật → sở-hữu phần sống sót. Người bỏ-cuộc trả-mượn PHA-1 nuôi người-mới; người kiên-trì được thưởng LAMP-thật. Bền hơn borrow-only (không điểm-thưởng) LẪN vesting-bán-ngay-v2 (cung-chực-bán tức thì, WP §7.2 dòng 227).
- (b) *first-principles:* tách **quyền-tiêu-dịch-vụ (MAGIC, cả 2 pha)** khỏi **sở-hữu-LAMP (chỉ PHA-2 sau cam-kết)**. Gen chỉ ĐỌC số dư → LAMP không cạn vì Gen (nguyên lý /CARP + WP §7.2). Cung-chực-bán bị **hoãn 1001 ngày** (qua rào cam-kết) → không phải cung-tức-thời.
- (c) *tối ưu:* tái dùng `dist_treasury.ak` (pot), `LoyaltyHolding` (struct khoá). Gen = **đọc-số-dư** (on-chain đọc datum `conditional_lamp`, KHÔNG chuyển LAMP) → tx Gen nhẹ, không đụng UTXO-LAMP. On-chain tối thiểu.
- (d) *user + bền vững:* user thấy 1 nút (GetLAMP) → nhận dòng MAGIC hằng ngày (cả 2 pha); ai kiên-trì đủ 1001 ngày → **được sở-hữu LAMP thật** (thưởng rõ ràng); kẻ-bỏ-cuộc trả phần-chưa-kiếm nuôi người-mới.

---

## 2. Luồng end-to-end

### 2.1 Sơ đồ trạng thái (vault đời một user) — 2 PHA, đồng-hồ 24h-slot từ vest_start

```
[Enclave keygen]──vân tay──►[Ví Phoenix tạo]
        │
        │ bấm GetLAMP  (backend orchestration — Long)
        ▼
[pot User] ── chuyển D LAMP vào VAULT USER ──►[Vault: conditional_lamp = D, vest_start = now]
   ▲                                    │
   │              ┌─────────────────────┴──────────────── vest_start + 24h·n  (đồng hồ NGÀY) ───────────────┐
   │              │                                                                                          │
   │   ╔══════════▼══ PHA-1 (ngày 1 → 1001, có-điều-kiện) ══════════════╗    ╔══ PHA-2 (từ ngày 1002, sở-hữu) ══╗
   │   ║  Gen MỚI (Instant + Schedule) ĐỌC conditional_lamp → drip MAGIC ║    ║  anti-idle DỪNG                   ║
   │   ║  ── KHÔNG đụng/không burn LAMP (WP §7.2 d205) ──                ║    ║  vest 1 LAMP/ngày:                 ║
   │   ║        │                                                        ║    ║  conditional_lamp → vested_unlocked║
   │   ║        │ user tiêu MAGIC (ConsumeMAGIC)                         ║    ║        │                           ║
   │   ║        ▼                                                        ║    ║        ▼ user SỞ HỮU vested_unlocked║
   │   ║  [Feecover: carp_paid==fee_magic] → GreenBack → Treasury        ║    ║  ── rút/bán/tự-Gen tuỳ ý ──────────►[Ví user]
   │   ║        │                                                        ║    ║  (Gen vẫn ĐỌC conditional_lamp còn lại)║
   │   ║  anti-idle NGÀY: idle → rút 1 LAMP → pot                        ║    ╚════════════════════════════════════╝
   │   ╚════════│═════════════════════════════════════════════════════════╝
   └──◄─────────┘ (chỉ PHA-1) reclaim 1 LAMP/ngày-idle về pot

[pot nạp phản-chu-kỳ]: phí user-trước thu bằng LAMP theo giá-trị → giá LAMP xuống → thu nhiều LAMP → pot ↑
[GetMAGIC] ── mua CARP bằng fiat ──► tích CARP chi tiêu dần (cửa mint CARP hợp lệ — cầu-vào-thật)
```

**Điểm cốt lõi (khác v3):** (1) LAMP vào **vault USER** (không "thuộc pot mãi"); (2) Gen **ĐỌC số dư**, KHÔNG chuyển LAMP đi đâu; (3) có **PHA-2**: LAMP sống sót **vest thành sở-hữu**, chảy `vault → ví user`; (4) anti-idle CHỈ PHA-1.

### 2.2 Các bước (khớp §3 toán)

1. **Keygen (Core/Enclave).** Vân tay → Secure Enclave sinh khoá → ví Phoenix. (Ranh giới Core — §5.)
2. **GetLAMP (backend — Long).** Đọc `pot_balance` → `D = min(1001, ⌊pot_balance/1_000_000⌋)` (§3.1) → dựng tx: (a) spend pot chuyển `D` LAMP vào **vault user**; (b) tạo **Vault** `conditional_lamp = D`, `vested_unlocked = 0`, `reclaimed_to_pot = 0`, `vest_start_slot = now`, `last_tick_day = 0`, `did_commit` (§3.2). Ký uỷ quyền pot (`dist_treasury` pattern, §3.6).
3. **Sinh MAGIC — Gen ĐỌC số dư (repo MAGIC/CARP — hard-depend).** Instant + Schedule Gen **đọc `conditional_lamp` (+ `vested_unlocked` PHA-2 nếu user chưa rút)** → drip MAGIC vào `magic_batches`. **KHÔNG chuyển LAMP** — Gen chỉ đọc datum-số-dư (§3.3). (⚠ code ScheduleGen cũ fire-LAMP→Treasury; §3.7 + [cần chốt].)
4. **Tiêu MAGIC (ConsumeMAGIC + Feecover).** Tiêu = **giảm số dư MAGIC account** (F1). Feecover ép `carp_paid == fee_magic`. Settlement backing qua **GreenBack backed-path** → CARP-có-backing về **Phoenix Treasury**. KHÔNG mint CARP tự do.
5. **Anti-idle — CHỈ PHA-1 (job NGÀY — Long).** Cuối mỗi NGÀY `n ≤ 1001`, nếu vault KHÔNG phát sinh ≥ `MIN_MAGIC_TX` tiêu-MAGIC **có counterparty-DID ≠ owner** (§3.3, G3) → rút **1 LAMP** khỏi `conditional_lamp` → trả về **pot** (§3.4). Grace ngắn onboarding; tuyến tính; dừng-rút khi tiêu-lại. **Ngày ≥ 1002: anti-idle DỪNG hẳn.**
6. **Vesting sở-hữu — CHỈ PHA-2 (từ ngày 1002 — validator/backend).** Mỗi NGÀY chuyển **1 LAMP** từ `conditional_lamp` → `vested_unlocked` (§3.5). `vested_unlocked` = user **sở hữu thật**.
7. **ClaimVested (user rút — PHA-2, tuỳ chọn).** User rút `vested_unlocked` (một phần/toàn bộ) ra ví ngoài, hoặc tự-Gen/bán (§3.5). LAMP rời-hệ sang user.
8. **GetMAGIC (backend — Long).** User mua **CARP bằng fiat** → tích chi tiêu dần. **Cửa mint CARP hợp lệ** (cầu-vào-thật), khác "mint từ tiêu MAGIC" (F1 cấm).

---

## 3. Toán

### 3.1 D — lượng LAMP mỗi GetLAMP

Anh chốt: "1 phần triệu số dư còn lại, không vượt 1001".

```
D = min( D_CAP , ⌊ pot_balance / SCALE ⌋ )

  D_CAP = 1001          (LAMP)   -- TRẦN; = số NGÀY cam-kết PHA-1 (đối-xứng 1001 ngày ↔ 1001 LAMP)
  SCALE = 1_000_000     (LAMP)   -- "1 phần triệu số dư còn lại"
  pot_balance           (LAMP)   -- số dư pot User tại thời điểm GetLAMP
```

- **Đơn vị on-chain = oildrop** (`InstantGen/MATH.md §1`: `oildrop = LAMP × 10⁶`). `D_oildrop = D × 10⁶`; `SCALE_oildrop = 10¹²`. Số học **integer** (floor div), không float.
- **Ngưỡng bão hoà:** `⌊pot/1e6⌋ ≥ 1001 ⟺ pot ≥ 1_001_000_000 LAMP`. Trên ngưỡng mọi user nhận đúng 1001.
- **Đối-xứng 1001:** `D_CAP = 1001 LAMP` khớp `1001 ngày PHA-1`. Nếu user KHÔNG bị anti-idle rút ngày nào (active cả 1001 ngày) → PHA-2 vest đủ 1001 LAMP thành sở-hữu (1 LAMP/ngày × 1001 ngày). Anti-idle rút `r` LAMP → PHA-2 chỉ còn `1001 − r` để vest.
- **Ví dụ:** `pot=5_000_000_000` → `D=1001`. `pot=300_000_000` → `D=300` (pilot mới; 3 nguồn nạp kéo pot lên; PHA-1 = 300 ngày? → **KHÔNG**: PHA-1 luôn 1001 ngày; D thấp chỉ nghĩa vault ít LAMP hơn để vest, ranh-giới-pha vẫn tại ngày 1001).

### 3.2 Datum vault + số-dư 2 pha (KHÔNG có "thuộc pot mãi")

**Datum MỚI (phản ánh 2 pha):**

```
VaultDatum {
  owner_commit      : hash        -- ràng vault vào owner (ký ClaimVested)
  did_commit        : hash        -- blake2b_256(did ‖ salt); attribution + 1-DID-1-vault
  vest_start_slot   : slot        -- lúc nhận LAMP về vault; gốc đồng-hồ 24h
  conditional_lamp  : int (oildrop) -- số dư PHA-1 CHƯA-vest: khoá, sinh MAGIC (đọc), anti-idle rút
  vested_unlocked   : int (oildrop) -- đã-sở-hữu (PHA-2): user rút được (ClaimVested)
  reclaimed_to_pot  : int (oildrop) -- tổng anti-idle đã rút về pot (PHA-1)
  last_tick_day     : int          -- NGÀY cuối đã xử-lý (anti-idle PHA-1 / vest PHA-2) — chống double-tick
}
```

**Phân biệt hai loại LAMP (LÕI v4):**

| Loại | Field | Pha | Khoá? | Sinh MAGIC? | Anti-idle? | User rút? |
|---|---|---|---|---|---|---|
| **LAMP-điều-kiện** | `conditional_lamp` | cả 2 (giảm dần PHA-2) | có (có-điều-kiện) | **CÓ (Gen đọc)** | **CÓ (chỉ PHA-1)** | **KHÔNG** |
| **LAMP-đã-vest** | `vested_unlocked` | chỉ PHA-2 | không (sở-hữu) | có (Gen đọc, nếu chưa rút) | KHÔNG | **CÓ (ClaimVested)** |

Gọi `n` = số NGÀY kể từ `vest_start` = `⌊(tx_lower_slot − vest_start_slot) / 86_400⌋`. `SLOTS_PER_DAY = 86_400` (1 slot = 1 giây).

```
-- PHA-1: 1 ≤ n ≤ 1001
conditional_lamp(n) = D − reclaimed_to_pot(n)                (khoá có-điều-kiện; chỉ giảm qua anti-idle)
vested_unlocked(n)  = 0

-- PHA-2: n ≥ 1002
vest_progress(n)    = min( conditional_lamp(1001), n − 1001 )  (số LAMP đã vest, ≤ phần-sống-sót)
vested_unlocked(n)  = vest_progress(n) − vested_claimed(n)     (đã-vest chưa-rút, còn trong vault)
conditional_lamp(n) = conditional_lamp(1001) − vest_progress(n) (còn-lại chưa-vest, vẫn khoá)

  conditional_lamp(1001) = D − reclaimed_to_pot(1001)  = phần-SỐNG-SÓT sau PHA-1
  vest hết sau  (1001 + conditional_lamp(1001))  ngày kể từ vest_start
```

- **Đơn-điệu:** `conditional_lamp` đơn-điệu-GIẢM (PHA-1 anti-idle; PHA-2 vest). `reclaimed_to_pot` + `vest_progress` đơn-điệu-TĂNG.
- **Tái dùng `LoyaltyHolding`** (`types.ak:31`): `conditional_lamp` = holdings `is_locked:True`; `vested_unlocked` = holdings `is_locked:False`. Vest = flip 1 unit `is_locked True→False`. KHÔNG phát minh struct mới.
- **`last_tick_day`** chống double-tick (anti-idle rút 2 lần/ngày, hoặc vest 2 lần/ngày).

### 3.3 Sinh MAGIC — Gen MỚI ĐỌC số dư, KHÔNG đụng LAMP

**Nguyên lý (anh chốt + /CARP):** Gen là **yield-đọc-số-dư**. **CẢ InstantGen + ScheduleGen** đọc `(conditional_lamp + vested_unlocked_chưa_rút)` → tính lượng MAGIC drip vào `magic_batches`. **KHÔNG burn, KHÔNG chuyển, KHÔNG tiêu LAMP** — LAMP đứng yên trong vault.

- **Neo /CARP + WP:** `SESSION-STATE-carp-2026-06-26.md` dòng 86 ("Gen≠Mint; MAGIC = quyền-tiêu-1-dịch-vụ") + WP §7.2 dòng 205 ("nắm LAMP chỉ MỞ TƯ-CÁCH (gate)... biến quyết định là lượng-MAGIC-tiêu, không phải lượng-LAMP-giữ"). LAMP = **cơ-sở-đọc** để tính suất MAGIC, không phải **nhiên-liệu-đốt**.
- **MAGIC-yield trên số-dư-HIỆN-TẠI:** `M(n) = f_gen( conditional_lamp(n) + vested_unlocked_chưa_rút(n) )`. Anti-idle rút (PHA-1) hoặc user rút vested (PHA-2) → cơ-sở-đọc giảm → MAGIC-yield giảm theo. Ai idle mất dần cả LAMP-điều-kiện LẪN MAGIC-yield → phạt-tự-nhiên use-it-or-lose-it (WP §7.2 dòng 227).
- **use-it-or-lose-it:** `M` là **trần-suất mỗi kỳ**, không tiêu thì mất suất kỳ đó (WP §7.2 dòng 227) — KHÔNG cộng-dồn bể-quyền-chực-bán.
- **[GHI TRUNG THỰC — nguyên-lý-vs-code]** /CARP chốt nguyên-lý "Gen đọc-số-dư không-đụng-LAMP" (WP §7.2 rõ; SESSION-STATE dòng 86) NHƯNG **/CARP chưa spell-out cơ-chế-on-chain đọc-datum-số-dư** (validator/công-thức cụ-thể tính MAGIC từ số-dư đọc-được). Code MAGIC-repo cũ thì fire-LAMP→Treasury (TIÊU-THỤ, mâu thuẫn). → Cần MAGIC/CARP-team **spell-out engine đọc-số-dư** (§3.7, §6.6). **[CẦN CHỐT].**

### 3.4 Anti-idle — CHỈ PHA-1 (ngày ≤ 1001), đồng-hồ-NGÀY, tuyến tính, grace

Trong NGÀY `n` **với `n ≤ 1001`** (PHA-1), gọi:

```
active(profile, n) = ( M_profile(n) ≥ MIN_MAGIC_TX )

  M_profile(n)  = TỔNG MAGIC tiêu trong NGÀY n trên TOÀN PROFILE của user
                  (mọi DID cá-nhân của chính user), THOẢ counterparty_did ≠ owner_did
                  (G3 — chống self-loop/wash);
                  ✗ KHÔNG gồm MAGIC tiêu qua DID user chỉ CÓ THẨM QUYỀN (Org/uỷ-quyền).
  MIN_MAGIC_TX  = 10% × MAGIC_genable( granted_LAMP )        [TẠM — anh đặt 2026-07-06]
                  granted_LAMP = conditional_lamp(n) (LAMP-điều-kiện còn trong vault),
                                 KHÔNG gồm LAMP user TỰ MUA, KHÔNG gồm vested_unlocked.

reclaim(vault, n) =                                          -- CHỈ áp dụng khi n ≤ 1001
  if n > 1001                              → 0                    (PHA-2: anti-idle DỪNG)
  else if n < vest_start_day + GRACE_DAYS  → 0                    (grace onboarding, theo NGÀY)
  else if active(vault, n)                 → 0                    (tiêu-lại → dừng rút)
  else if conditional_lamp(vault) ≥ RECLAIM_UNIT → RECLAIM_UNIT   (rút tuyến tính về pot)
  else                                      → 0                    (hết số dư → không rút nữa)

  RECLAIM_UNIT = 1        (LAMP / NGÀY idle)                       (tuyến tính, KHÔNG ×2)
  GRACE_DAYS   = 7        (NGÀY đầu miễn rút — cửa-sổ-onboarding-một-lần)
```

- **CHỈ PHA-1 (n ≤ 1001).** Ngày ≥ 1002 anti-idle DỪNG HẲN — user đã kiếm được (qua cam-kết đủ 1001 ngày).
- **Đồng-hồ-NGÀY** (slot/86400). Mỗi NGÀY-idle rút đúng `RECLAIM_UNIT` từ `conditional_lamp` → **pot** (§3.6). Tuyến tính, không compound.
- **Rút từ `conditional_lamp`** (LAMP-điều-kiện, chưa-vest). KHÔNG đụng `vested_unlocked` (=0 trong PHA-1). Không "tịch-thu tài-sản-user" vì PHA-1 user **chưa sở-hữu** LAMP — chỉ **thu-hồi-phần-chưa-kiếm**.
- **Chỉ đếm counterparty-DID ≠ owner** (G3; `VeData-Reply §4`). Self-loop KHÔNG tính. **Hard-depend MAGIC-team:** `did_commit` per-DID + field `counterparty` trong consume-event.
- **Đo TỔNG tiêu-MAGIC cả PROFILE/NGÀY**, trừ DID-thẩm-quyền (anh chốt).
- **`MIN_MAGIC_TX = 10% MAGIC-gen-able từ granted_LAMP=conditional_lamp`** (TẠM). **[SUB-OPEN]** cơ-sở gen-able = *daily-gen-able* hay *cumulative* — buộc với granularity §3.7; mặc định spec dùng daily-gen-able.
- **Lực-giữ chính KHÔNG phải điều kiện này** mà là **tiêu-MAGIC-thật** (F8). Anti-idle = thu-hồi-phần-chưa-kiếm của người-bỏ-cuộc trong PHA-1.

### 3.5 Vesting sở-hữu (PHA-2, n ≥ 1002) + ClaimVested + (bỏ CloseBorrow)

```
-- Vesting (mỗi NGÀY n ≥ 1002, tự-động validator/backend, 1 lần/ngày qua last_tick_day):
vest_tick(vault, n) :
  if n ≤ 1001                          → 0                        (chưa tới PHA-2)
  else if conditional_lamp(vault) ≥ VEST_UNIT :
        conditional_lamp −= VEST_UNIT
        vested_unlocked  += VEST_UNIT                              (flip is_locked True→False)
  else                                  → 0                        (đã vest hết phần-sống-sót)

  VEST_UNIT = 1   (LAMP / NGÀY)   -- nhỏ giọt, KHÔNG mở hết cùng lúc

-- ClaimVested (user rút, PHA-2, tuỳ chọn):
claim_vested(vault, amount) :
  expect signed_by(owner_commit)
  expect amount ≤ vested_unlocked(vault)
  vested_unlocked −= amount
  → chuyển `amount` LAMP ra ví user (rời-hệ) HOẶC giữ trong vault để tự-Gen (đọc-số-dư tiếp)
```

- **Vesting nhỏ-giọt 1 LAMP/ngày** (KHÔNG mở hết) — chống cung-đổ-một-lúc dù đã sang PHA-2. Từ ngày 1002, đủ `conditional_lamp(1001)` ngày để vest hết phần-sống-sót.
- **`vested_unlocked` = user SỞ HỮU thật** → rút ra ví (bán/chuyển) hoặc **giữ trong vault tự-Gen** (Gen vẫn đọc-số-dư `vested_unlocked` → tiếp tục MAGIC-yield). Nếu bán năng-lực-tiêu-thụ → qua CARP-ConsumptionMarket §5 (shadow-price = chi-phí-thay-thế).
- **CloseBorrow BỎ (v3 có, v4 xét lại):** vì LAMP ở vault USER (không "thuộc pot"), không cần "trả toàn-bộ về pot". User thoát **PHA-1** = **từ-bỏ**: `conditional_lamp` còn lại về pot (redeemer `AbandonPhase1`, chỉ PHA-1, tự-nguyện — vì chưa-sở-hữu). User thoát **PHA-2** = **ClaimVested** phần đã-sở-hữu + phần `conditional_lamp` chưa-vest **vẫn về pot khi từ-bỏ** (chưa kiếm). Chi tiết `AbandonPhase1` tuỳ chọn — mặc định user cứ để anti-idle tự thu-hồi.
- **[GHI TRUNG THỰC]** whitepaper/Gen KHÔNG spell-out "vest LAMP-điều-kiện thành sở-hữu sau 1001 ngày" — là **directive Activation-layer anh Aladin**. WP §7.2 dòng 205 (LAMP = gate, không đường-ra-thụ-động) áp cho **giai-đoạn-cam-kết**; PHA-2 sở-hữu là **phần-thưởng-đã-kiếm** (đã qua tiêu-thật đủ 1001 ngày → đúng F8 "giá-trị từ TIÊU-DỊCH-VỤ"). Không mâu thuẫn: cung-bán bị **hoãn qua rào cam-kết 1001 ngày**, không phải cung-tức-thời (WP §7.2 dòng 227 lo cung-tồn-đọng-chực-bán = cung KHÔNG-qua-cam-kết). [cần chốt] xác nhận đọc này.

### 3.6 Pot cân bằng — nạp/rút, phản-chu-kỳ, tính khoản-vest-ra

```
pot_balance(n+1) = pot_balance(n)
                   − Σ D(user_mới)                    [RÚT]  GetLAMP mỗi user mới (vào vault user)
                   + Σ reclaim(vault, n)               [NẠP]  anti-idle PHA-1 thu-hồi hằng NGÀY (§3.4)
                   + Σ abandon_phase1(vault)           [NẠP]  user từ-bỏ PHA-1 → conditional_lamp về pot (§3.5)
                   + fee_refill_lamp(n)                [NẠP]  phí user-trước thu bằng LAMP theo giá-trị (phản-chu-kỳ)
                   + treasury_topup(n)                 [NẠP]  MagicLamp Treasury bơm (khi cần)

  -- LƯU Ý conservation (khác v3): PHA-2 vest → vested_unlocked → ClaimVested RỜI-HỆ sang user.
  -- LAMP-vest KHÔNG quay về pot. Pot chỉ thu-hồi phần CHƯA-kiếm (anti-idle + abandon PHA-1).
```

- **3 nguồn nạp (anh chốt):**
  1. **Phí user-trước — `fee_refill_lamp` (phản-chu-kỳ).** Phí cố-định theo GIÁ-TRỊ (§4) thu bằng LAMP: giá LAMP xuống → thu nhiều LAMP → pot ↑. Cùng-họ GreenBack counter-cyclical (WP §7.3 dòng 240).
  2. **Anti-idle + abandon PHA-1** (§3.4/§3.5) — người-bỏ-cuộc-PHA-1 trả phần-chưa-kiếm → pot.
  3. **MagicLamp Treasury — `treasury_topup`.**
- **Bù dòng-ra PHA-2:** vì LAMP-vest rời-hệ, pot cần nguồn nạp bù đủ để duy trì onboarding. Nguồn 1 (phản-chu-kỳ) + nguồn 3 (treasury) là bù chính. **[SUB-OPEN §6.5]** cân-đối tốc-độ-vest-ra vs tốc-độ-nạp để pot không cạn khi nhiều user qua PHA-2.
- **[GHI TRUNG THỰC — whitepaper-gap]** `fee_refill_lamp` là **directive anh Aladin**; WP §7 chưa spell-out. Không mâu thuẫn — mở-rộng-cấp-Activation.
- **Tái dùng `dist_treasury.ak`** (`LAMP/Genesis/onchain/validators/dist_treasury.ak:14-27`): pot = script + authority-sig release.
- **Bất biến bảo toàn:** LAMP không tự-sinh; `Σ D_phát = Σ nạp-pot + Σ vested_claimed-ra-user + Δ(LAMP trong vault còn sống)` (I-ACT-5). LAMP-điều-kiện quay-vòng nội-bộ (pot↔vault); LAMP-vest rời-hệ hợp-lệ (đã-kiếm).

### 3.7 Cơ-chế Gen đọc-số-dư + granularity — [CẦN CHỐT]

**Nguyên-lý anh chốt (v4):** Gen **CHỈ ĐỌC số dư LAMP để tính MAGIC, KHÔNG BAO GIỜ burn/tiêu/chuyển LAMP.** /CARP hậu thuẫn nguyên-lý (WP §7.2 dòng 205, SESSION-STATE-carp dòng 86). **Ba điểm cần MAGIC/CARP-team spell-out on-chain:**

1. **Engine đọc-số-dư (thay fire→Treasury cũ).** /CARP chốt nguyên-lý nhưng **chưa có validator/công-thức on-chain đọc-datum-số-dư → tính MAGIC drip**. Code MAGIC-repo cũ (`ScheduleGen/FEAT.md §3.2 dòng 48-51`) fire chuyển LAMP→Treasury (TIÊU-THỤ) — **LỖI THỜI, mâu thuẫn nguyên-lý.** **[CẦN CHỐT]:** MAGIC/CARP-team spell-out engine mới: Gen tx **đọc `conditional_lamp + vested_unlocked` từ VaultDatum (reference input)** → drip `M_i` MAGIC → **KHÔNG spend/không-chuyển UTXO-LAMP** (LAMP đứng yên; chỉ magic_batches thay đổi). Đây là cơ-chế đúng nguyên-lý; cần code hoá.

2. **Cả 2 Gen (Instant + Schedule) đọc-số-dư — đối-xử đồng-nhất.** v3 phân biệt "ScheduleGen-lock hợp, InstantGen tiêu-thụ không dùng" — **KHÔNG còn đúng ở v4.** Nguyên-lý mới: **cả 2 CHỈ ĐỌC** → cả 2 dùng được cho `conditional_lamp`. Khác biệt chỉ là **nhịp** (Instant = tiêu-ngay theo trần-kép; Schedule = drip-định-kỳ) — KHÔNG phải "một-tiêu-LAMP một-không". **[CẦN CHỐT]** xác nhận cả 2 Gen phiên-bản-/CARP đều đọc-số-dư (không có nhánh nào còn burn LAMP).

3. **Daily (anh) vs epoch (Gen).** Anh muốn "MAGIC hàng NGÀY" + anti-idle/vest theo NGÀY. Nếu engine drip theo EPOCH (~5 ngày) → cách hoà: **anti-idle + vest + đọc-số-dư theo NGÀY (slot/86400); MAGIC nền drip theo epoch** (nhịp Gen). Nếu muốn cảm-giác-MAGIC-daily thật → drip-daily-buffer backend (không sửa on-chain). **[CẦN CHỐT]** anh muốn daily thật hay daily-tick + epoch-drip (spec mặc định).

---

## 4. Invariant (I-ACT-x) — viết lại cho model 2-pha

| ID | Bất biến | Bám |
|---|---|---|
| **I-ACT-1** | **Vault thuộc USER, khoá có-điều-kiện khi tạo:** ngay sau GetLAMP `conditional_lamp == D`, `vested_unlocked == 0`; LAMP ở vault user (không "thuộc pot mãi"). | §3.2; §1(1) |
| **I-ACT-2 (2-pha ranh giới)** | **Anti-idle CHỈ PHA-1 (n ≤ 1001); vest CHỈ PHA-2 (n ≥ 1002).** Ngày 1002 trở đi: `reclaim ≡ 0`; vest bắt-đầu. Không chồng-lấn hai cơ-chế. | §3.4; §3.5 |
| **I-ACT-3** | **Anti-idle chỉ đếm counterparty-tx:** `active` chỉ tính tiêu-MAGIC `counterparty_did ≠ owner_did`; self-loop KHÔNG tính. | G3; §3.4; hard-depend §6 |
| **I-ACT-4** | **Thu-hồi PHA-1 theo NGÀY, tuyến tính, từ `conditional_lamp`, có grace, không compound:** `reclaim ≤ RECLAIM_UNIT` mỗi NGÀY-idle → **pot**; grace/active/hết-số-dư → 0. Đồng-hồ-NGÀY (slot/86400). | §3.4 |
| **I-ACT-5 (conservation)** | **Bảo toàn:** LAMP không tự-sinh; `Σ D_phát = Σ nạp-pot + Σ vested_claimed-ra-user + Δ(LAMP-vault-sống)`. LAMP-điều-kiện quay-vòng (pot↔vault); **LAMP-vest RỜI-HỆ hợp-lệ** (đã-kiếm, PHA-2). | §3.6 |
| **I-ACT-6** | **D-cap 1001, đồng-hồ-pha:** `D ≤ 1001`; ranh-giới PHA-1/PHA-2 tại NGÀY 1001 **bất kể D** (D thấp = ít LAMP-vest, không đổi ngày-1001). `conditional_lamp` đơn-điệu-giảm. | §3.1; §3.2 |
| **I-ACT-7 (Gen đọc-số-dư)** | **Gen KHÔNG đụng LAMP:** Instant + Schedule **ĐỌC** `conditional_lamp + vested_unlocked` để tính MAGIC → drip magic_batches; **KHÔNG burn/không-chuyển/không-spend UTXO-LAMP.** LAMP đứng yên trong vault. | §3.3; §3.7; WP §7.2 dòng 205; /CARP SESSION dòng 86 |
| **I-ACT-8 (vest-sở-hữu)** | **PHA-2 vest 1 LAMP/ngày `conditional_lamp → vested_unlocked`; `vested_unlocked` user RÚT được (ClaimVested, ký owner); `conditional_lamp` user KHÔNG rút.** Vest ≤ phần-sống-sót `conditional_lamp(1001)`. | §3.5 |
| **I-ACT-9 (settlement)** | **MAGIC drip đúng + settlement qua GreenBack backed-path:** tiêu-MAGIC `carp_paid == fee_magic` từ CARP-user-đã-có; backing qua GreenBack (LAMP-haircut + tài-sản-cứng + 3-phanh); Activation KHÔNG mint/burn CARP/LAMP tự do, KHÔNG đọc oracle-giá cầm-lái (F6). | G1; §3.4; WP §7.2, F1 |
| **I-ACT-10** | **Vault gắn DID (attribution):** `did_commit = blake2b_256(did ‖ salt)` bảo toàn qua đời vault; một DID không mở nhiều vault né trần (khi `did_commit` per-DID sẵn sàng). | `VeData-Reply §2`; §6 |
| **I-ACT-11** | **Lực-giữ = tiêu-MAGIC-thật; PHA-1 không phạt tài-sản-user:** PHA-1 user CHƯA sở-hữu LAMP → anti-idle thu-hồi-phần-chưa-kiếm, KHÔNG tịch-thu; PHA-2 sở-hữu là phần-thưởng-đã-kiếm (F8). | G2; F8 `INV-MAGIC-CITIZEN` |

**Đã BỎ so với v3:** "LAMP-mượn KHÔNG rời-vault-sang-user bao giờ" (v4 PHA-2 rời-hệ hợp-lệ); "CloseBorrow trả toàn-bộ về pot" (thay AbandonPhase1 chỉ PHA-1 + ClaimVested PHA-2); "chỉ ScheduleGen-lock, InstantGen tiêu-thụ không dùng" (v4 cả-2-đọc-số-dư).

---

## 5. Ranh giới triển khai

| Tầng | Việc | Đội |
|---|---|---|
| **On-chain (Aiken)** | **Validator vault 2-pha** (`conditional_lamp`/`vested_unlocked` I-ACT-1, ranh-giới-pha n=1001 I-ACT-2, anti-idle-reclaim→pot PHA-1 I-ACT-4, **vest 1/ngày PHA-2** I-ACT-8, **ClaimVested** ký-owner, Gen-đọc-số-dư-KHÔNG-spend-LAMP I-ACT-7, `did_commit` gate I-ACT-10) + **validator pot** (kế thừa `dist_treasury.ak`, nạp 3 nguồn + bù dòng-vest-ra I-ACT-5). Tái dùng `LoyaltyHolding`. **Phối MAGIC/CARP-team** engine Gen đọc-số-dư (§3.7). | **Tuân** |
| **Backend (Java)** | **GetLAMP orchestration** (đọc pot → D §3.1 → dựng tx vào-vault-user+khoá) + **anti-idle job NGÀY PHA-1** (đọc consume-event counterparty §3.4 → tx reclaim→pot) + **vest job NGÀY PHA-2** (§3.5) + **ClaimVested** (§3.5) + **fee_refill_lamp** (phản-chu-kỳ §3.6) + **GetMAGIC** (fiat→CARP). `curl` verify sau deploy. | **Long** |
| **Core / Enclave (Flutter+Rust)** | Keygen vân tay → ví Phoenix; ký tx GetLAMP/consume/ClaimVested bằng controller-DID; UI 1-nút GetLAMP + **chỉ-báo-PHA (PHA-1: đang-kiếm / PHA-2: đang-sở-hữu)** + MAGIC-hàng-ngày + cảnh-báo-anti-idle (PHA-1) + "đã kiếm X LAMP (vested)". | **Core** |
| **Phụ thuộc MAGIC/CARP-team** | (1) `did_commit` per-DID trong `EngageDatum`; (2) **counterparty field** consume-event (Phase 1); (3) **engine Gen ĐỌC-số-dư** (đọc VaultDatum reference-input → drip MAGIC → KHÔNG-spend-LAMP) — spell-out on-chain (§3.7). Activation tiêu-thụ + phối, không dựng lại. | MAGIC/CARP |
| **Phụ thuộc CARP-team** | **GreenBack interface** (`greenback_settle` §3.4) — thu CARP-fee `κ_eff`/`br` (backed-path §7), trả CARP-có-backing về Treasury. + **shadow-price** (CARP-ConsumptionMarket §5) nếu user PHA-2 bán năng-lực-tiêu-thụ. CHỈ khai interface, không chạm peg (F5/F9). | CARP |

---

## 6. Phụ thuộc chéo (bắt buộc nêu rõ — Phase 1)

**6.1 `did_commit` + counterparty (MAGIC-team — BLOCKER cho I-ACT-3).** `EngageDatum` hiện `did_commit = #""` sentinel. Cần `did_commit = blake2b_256(did‖salt)` + consume-event mang `counterparty_did`. Không có → anti-idle không phân biệt self-loop (G3 hở). Kéo Phase 1.

**6.2 Resolve API point-in-time (PhoenixKey-Database — Long).** Anti-idle + vault-DID-gate cần `did_active(did, day)` / `key_authorized(key, did, day)`. DB hiện chỉ state-hiện-tại.

**6.3 GreenBack interface (CARP-team — I-ACT-9).** `greenback_settle` (§3.4) — CARP-team định `κ_eff`/`br` backed-path. PhoenixKey chỉ gửi CARP-fee-gom + nhận CARP-có-backing.

**6.4 Vesting-release kho→pot (LAMP-team — nạp pot lần đầu).** Pot User nạp-vốn-ban-đầu từ Treasury/`dist_treasury`. Cần địa chỉ kho + `kho_nft` + authority.

**6.5 Cơ-chế thu-phí-bằng-LAMP → pot + cân-đối dòng-vest-ra (LAMP/backend — phản-chu-kỳ).** `fee_refill_lamp` (§3.6) cần: điểm thu phí, quy-đổi giá-trị→LAMP, đường về pot. **+ Cân-đối tốc-độ-vest-ra (PHA-2) vs nạp** để pot không cạn. Directive anh Aladin; tham số chờ chốt.

**6.6 Engine Gen ĐỌC-số-dư (MAGIC/CARP-team — BLOCKER kiến trúc, §3.7).** /CARP chốt nguyên-lý "Gen đọc-số-dư không-đụng-LAMP" (WP §7.2, SESSION dòng 86) nhưng **chưa spell-out validator/công-thức on-chain**. Code MAGIC-repo cũ fire-LAMP→Treasury (TIÊU-THỤ, lỗi thời). Cần: engine đọc VaultDatum-số-dư → drip MAGIC → KHÔNG-spend-UTXO-LAMP; xác nhận cả Instant+Schedule đều đọc-số-dư; daily-vs-epoch.

---

## 7. Tham số then chốt

| Tham số | Ký hiệu | Giá trị / khoảng | Trạng thái |
|---|---|---|---|
| Trần D | `D_CAP` | **1001 LAMP** (= 1001 ngày PHA-1, đối-xứng) | CHỐT |
| Chia-pot | `SCALE` | **1_000_000 LAMP** ("1 phần triệu") | CHỐT |
| Ranh-giới pha | — | **PHA-1: ngày 1→1001 (đúng 1001 ngày); PHA-2: từ ngày 1002** | **CHỐT (2-pha)** |
| Slots/ngày | `SLOTS_PER_DAY` | **86_400** (Cardano 1s/slot); đồng-hồ từ `vest_start` | CHỐT |
| Grace onboarding | `GRACE_DAYS` | **7 NGÀY** (PHA-1) | CHỐT |
| Thu-hồi/NGÀY idle (PHA-1) | `RECLAIM_UNIT` | **1 LAMP** (tuyến tính) → pot; CHỈ n ≤ 1001 | CHỐT |
| Vest/NGÀY (PHA-2) | `VEST_UNIT` | **1 LAMP** (nhỏ giọt) `conditional→vested_unlocked`; CHỈ n ≥ 1002 | **CHỐT (2-pha)** |
| Đếm active | `MIN_MAGIC_TX` | **10% MAGIC-gen-able từ granted_LAMP** (= `conditional_lamp`, không gồm tự-mua/vested); TỔNG profile/NGÀY, trừ DID-thẩm-quyền | **TẠM (07-06)** · sub-open granularity |
| Engine Gen | — | **Instant + Schedule ĐỌC-số-dư** (không burn LAMP) — cả 2 dùng được | **[CẦN CHỐT §3.7 — spell-out on-chain]** |
| Granularity MAGIC | — | tick NGÀY (anti-idle/vest) + drip epoch (mặc định) HAY drip-daily | **[CẦN CHỐT §3.7]** |
| Thoát PHA-1 | `AbandonPhase1` | tự-nguyện: `conditional_lamp` còn lại → pot (chưa-kiếm) | tuỳ chọn (mặc định để anti-idle) |
| Rút PHA-2 | `ClaimVested` | rút `vested_unlocked` ra ví/tự-Gen; ký owner | **CHỐT (2-pha)** |
| Nguồn nạp pot | — | phản-chu-kỳ + anti-idle/abandon-PHA-1 + treasury; bù dòng-vest-ra | CHỐT (tham-số §6.5) |

---

## 8. Phụ thuộc & thứ tự triển khai

**Build được NGAY:** (1) Validator pot (kế thừa `dist_treasury.ak`); (2) Validator vault 2-pha (`conditional_lamp`/`vested_unlocked` + I-ACT-1/2/4/8, phần khoá+vest thuần KHÔNG cần Gen); (3) GetLAMP backend (đọc pot → D → tx vào-vault-user); (4) D-cap + conservation test (I-ACT-5/6); (5) ClaimVested (I-ACT-8, chỉ cần validator vault).

**CHỜ MAGIC/CARP-team (BLOCKER kiến trúc):** (6) Anti-idle counterparty-gate (I-ACT-3) — `did_commit` + counterparty. (7) **Engine Gen ĐỌC-số-dư §3.7** — spell-out on-chain (đọc-datum → drip → không-spend-LAMP) TRƯỚC khi nối Gen production.

**CHỜ CARP-team:** (8) GreenBack settlement (I-ACT-9) + shadow-price (PHA-2 bán năng-lực).

**CHỜ LAMP/backend:** (9) `fee_refill_lamp` phản-chu-kỳ + cân-đối dòng-vest-ra (§6.5).

**Thứ tự:** (1→2→3→4→5) build+test độc lập (khoá+vest thuần chạy được không cần Gen) → **spell-out engine Gen đọc-số-dư §3.7 với MAGIC/CARP-team** → (7) nối Gen → (6) bật anti-idle → (8) GreenBack → (9) phản-chu-kỳ.

---

## 9. Phản biện đối kháng (self-adversarial)

**A. Sybil-farm nhiều ví lấy D×N.** — NGOÀI PHẠM VI (sinh trắc SE đủ). `did_commit` per-DID chặn 1-DID-nhiều-vault né trần (I-ACT-10).

**B. Wash-generation né anti-idle (G3 — lỗ thật).** — I-ACT-3 chỉ đếm counterparty≠owner. Hở nếu MAGIC-team chưa gắn counterparty (§6.1) → không bật anti-idle production tới khi có.

**C. Bán cung LAMP ngay từ đầu (nghi cung-chực-bán, WP §7.2 dòng 227).** — **CHẶN bằng rào 2-pha:** PHA-1 (1001 ngày) user KHÔNG rút/bán LAMP (chưa sở-hữu, `conditional_lamp` khoá — I-ACT-8). Chỉ PHA-2 sau đủ cam-kết mới sở-hữu + vest **nhỏ-giọt 1/ngày**. → cung-bán **hoãn ≥ 1001 ngày + nhỏ-giọt**, KHÔNG phải cung-tồn-đọng-tức-thời. Người bán PHA-2 đã **tiêu-thật đủ 1001 ngày** (đúng F8). [cần chốt] xác nhận đọc-này khớp WP.

**D. Gen rút LAMP khỏi vault (code cũ fire→Treasury).** — **Nguyên-lý v4 GỠ hẳn:** Gen CHỈ ĐỌC số dư, KHÔNG chuyển LAMP (I-ACT-7, WP §7.2 dòng 205, /CARP dòng 86). Code MAGIC-repo cũ lỗi-thời → cần spell-out engine đọc-số-dư (§3.7). Không phải lỗ-thiết-kế mà là **gap-code-hoá nguyên-lý-đã-chốt**.

**E. Máy-in-tiền qua mint CARP (nghi Terra).** — I-ACT-9: tiêu MAGIC KHÔNG mint CARP; user trả CARP-đã-có; backing qua GreenBack backed-path. WP tự-nhận residual-reflexivity, khử bằng haircut+3-phanh (§13) — KHÔNG Terra.

**F. Pot cạn khi nhiều user qua PHA-2 (LAMP-vest rời-hệ).** — **Rủi ro thật mới ở v4** (khác v3 nơi LAMP không rời-hệ). Pot phải bù dòng-vest-ra bằng phản-chu-kỳ + treasury (§3.6, §6.5 sub-open). Anti-idle PHA-1 vẫn thu-hồi người-bỏ-cuộc. **[SUB-OPEN]** cân-đối tốc-độ.

**G. Anti-idle phạt user-nhàn-rỗi.** — chỉ PHA-1, đòi phần-chưa-kiếm, grace 7 ngày, dừng-rút khi tiêu-lại. use-it-or-lose-it (WP §7.2 dòng 227). PHA-2 anti-idle DỪNG (đã kiếm).

**H. Vest quá nhanh → đổ cung.** — vest **nhỏ-giọt 1 LAMP/ngày** (I-ACT-8), KHÔNG mở hết. Phần-sống-sót lớn nhất = 1001 LAMP → vest hết mất tối-đa 1001 ngày nữa. Cung rải đều, không sốc.

---

## 10. Tóm tắt cho reviewer

- Activation = **GetLAMP D LAMP vào VAULT USER (khoá có-điều-kiện) → 2 PHA**. **PHA-1 (ngày 1→1001):** Gen ĐỌC-số-dư sinh MAGIC (KHÔNG đụng LAMP), user hưởng MAGIC, anti-idle idle→1 LAMP về pot. **PHA-2 (từ ngày 1002):** anti-idle DỪNG, phần LAMP sống sót vest **1 LAMP/ngày thành SỞ HỮU user** (rút/bán/tự-Gen). **GetMAGIC** = fiat→CARP. Bỏ hẳn borrow-only v3.
- `D = min(1001, ⌊pot/1e6⌋)` LAMP; **pot tự-nuôi** (phản-chu-kỳ + anti-idle/abandon-PHA-1 + treasury), **bù dòng-vest-ra PHA-2**.
- **Gen = yield-đọc-số-dư:** cả Instant + Schedule CHỈ ĐỌC `conditional_lamp + vested_unlocked` → drip MAGIC, **KHÔNG burn/không-chuyển LAMP** (I-ACT-7; WP §7.2 dòng 205; /CARP SESSION dòng 86).
- **Datum MỚI:** `{owner_commit, did_commit, vest_start_slot, conditional_lamp, vested_unlocked, reclaimed_to_pot, last_tick_day}`. Phân biệt LAMP-điều-kiện (khoá, sinh-MAGIC, anti-idle PHA-1) vs LAMP-đã-vest (sở-hữu, rút-được PHA-2).
- **Redeemer MỚI:** `GenDrip` (MAGIC yield, LAMP-preserved), `Reclaim` (anti-idle PHA-1, n≤1001), `VestToOwner` (PHA-2, n≥1002, 1 LAMP conditional→vested), `ClaimVested` (rút vested ra user), `AbandonPhase1` (tuỳ chọn, từ-bỏ PHA-1→pot). CloseBorrow-trả-toàn-bộ-về-pot **BỎ**.
- **11 invariant I-ACT-1..11** (2-pha): anti-idle CHỈ n≤1001; vest CHỈ n≥1002; conditional-LAMP không rời-sang-user; vested-LAMP user rút được; Gen không-đụng-LAMP; conservation (LAMP-vest rời-hệ hợp-lệ).
- **Settlement (I-ACT-9):** user trả CARP-đã-có, backing qua GreenBack backed-path. PhoenixKey chỉ khai interface, không chạm peg.
- **[CẦN CHỐT — §3.7]:** (1) spell-out **engine Gen đọc-số-dư on-chain** (đọc VaultDatum reference → drip → KHÔNG-spend-LAMP; thay code cũ fire→Treasury); (2) xác nhận cả Instant+Schedule /CARP đều đọc-số-dư (không nhánh nào burn LAMP); (3) daily(anh) vs epoch(Gen); (4) xác nhận PHA-2-vest-sở-hữu khớp WP §7.2 (cung-hoãn-qua-cam-kết ≠ cung-chực-bán).
- **Ghi trung thực:** "GetLAMP/pot/2-pha/vest-sở-hữu-sau-1001-ngày" KHÔNG có trong whitepaper — directive Activation-layer anh Aladin. /CARP + WP §7.2/§7.3 hậu thuẫn **nguyên-lý Gen-đọc-số-dù** rõ ràng, nhưng **/CARP CHƯA spell-out cơ-chế-on-chain đọc-datum-số-dư** (mới có nguyên-lý, chưa có validator). Lỗ thật còn lại = G3 (wash) + F (pot cạn dòng-vest, sub-open).
- On-chain (Tuân) + backend (Long) + Core/Enclave + phụ thuộc MAGIC/CARP/LAMP-team.

---

## Phụ lục — Quyết định (chốt vs CẦN CHỐT)

| # | Mục | Trạng thái |
|---|---|---|
| — | Model = 2-PHA (có-điều-kiện → vesting-sở-hữu); thay borrow-only v3 | **CHỐT (anh xác nhận 07-06)** |
| — | LAMP vào VAULT USER; "mượn" = ngữ-nghĩa-UX; PHA-2 vest thành SỞ HỮU | **CHỐT** |
| — | Gen CHỈ ĐỌC số dư, KHÔNG burn/tiêu/chuyển LAMP (cả Instant + Schedule) | **CHỐT (nguyên-lý)** · [spell-out on-chain §3.7] |
| A | D-cap 1001, SCALE 1e6, ranh-giới-pha ngày 1001 | **CHỐT** |
| B | Anti-idle PHA-1 (n≤1001), RECLAIM_UNIT=1→pot, GRACE=7 | **CHỐT** |
| C | Vest PHA-2 (n≥1002), VEST_UNIT=1/ngày, ClaimVested | **CHỐT (2-pha)** |
| D | `MIN_MAGIC_TX` = 10% gen-able từ `conditional_lamp`, TỔNG profile/NGÀY, trừ DID-thẩm-quyền | **TẠM (07-06)** · sub-open granularity |
| F | Nguồn nạp pot + **bù dòng-vest-ra PHA-2** | **CHỐT** (tham-số + cân-đối §6.5, sub-open) |
| **§3.7-1** | Spell-out engine Gen đọc-số-dư on-chain (thay fire→Treasury cũ) | **[CẦN CHỐT — MAGIC/CARP-team]** |
| **§3.7-2** | Xác nhận cả Instant+Schedule /CARP đều đọc-số-dư (không burn LAMP) | **[CẦN CHỐT — xác nhận /CARP]** |
| **§3.7-3** | Daily(anh) vs epoch(Gen) — tick-daily + drip-epoch mặc định | **[CẦN CHỐT — anh]** |
| **§3.5** | PHA-2-vest-sở-hữu khớp WP §7.2 (cung-hoãn-qua-cam-kết) | **[CẦN CHỐT — xác nhận đọc WP]** |
