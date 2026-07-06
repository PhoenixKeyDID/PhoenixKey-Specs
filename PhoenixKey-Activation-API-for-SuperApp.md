# PhoenixKey — Activation API (model 2-PHA: có-điều-kiện → vesting-sở-hữu) cho SuperApp

> **Đối tượng:** agent SuperApp (Thư) dựng UI Activation. **Không code** — chỉ đặc tả API.
> **Nguồn chuẩn:** `spec-proposals/PhoenixKey-Activation-Feat-Math.md` (**v4, 2026-07-06** — model 2-PHA).
> **Convention backend bám thật:** `PhoenixKey-Database/.../controller/` (ActivationController, WalletController), `common/DataResponse.java`, `security/AuthenticatedUser.java`, `exception/ErrorCode.java`.
> SPEC PROPOSAL — chờ anh Aladin + Long chốt. **v4 2026-07-06.**
>
> **⚠ v4 THAY v3.** v3 model borrow-only (GetLAMP=MƯỢN, LAMP mãi thuộc pot, user KHÔNG-bao-giờ sở-hữu, CloseBorrow trả-toàn-bộ-về-pot) — **BỎ.** Model đúng = **2 PHA**: GetLAMP chuyển D LAMP vào **VAULT CỦA USER** (khoá có-điều-kiện). **PHA-1 (ngày 1→1001):** sinh MAGIC (Gen ĐỌC-số-dư), anti-idle rút LAMP-chưa-kiếm về pot, user hưởng MAGIC (chưa sở-hữu LAMP). **PHA-2 (từ ngày 1002):** anti-idle DỪNG, phần LAMP sống sót **vest 1 LAMP/ngày thành SỞ HỮU user** → **rút/bán/tự-Gen** (endpoint mới `/vault/{did}/claim-vested`). Dashboard = **pha hiện tại + số ngày + conditional_lamp vs vested_unlocked + MAGIC + cảnh-báo-anti-idle (chỉ PHA-1)**.

---

## 0. Điều SuperApp cần biết trước

**Hai model cũ ĐÃ BỎ:**
1. **VND-Genie** (`/activation/initiate`, `/{id}/confirm-payment`... "200k VND → 1001 LAMP + 10 ADA") — còn trong `ActivationController.java:37-99` nhưng thuộc model huỷ. KHÔNG dựng UI theo nhóm đó.
2. **Borrow-only v3** ("MƯỢN, LAMP thuộc pot mãi, CloseBorrow trả-toàn-bộ") — **BỎ.** LAMP KHÔNG "thuộc pot mãi"; user **CÓ** đường sở-hữu (PHA-2). Không còn "CloseBorrow trả-toàn-bộ-về-pot".

**Model mới = 2 PHA:**
- **GetLAMP → vault CỦA USER** (khoá có-điều-kiện). "Mượn" chỉ là **ngữ-nghĩa-UX** (khoản CÓ ĐIỀU KIỆN tới khi kiếm được).
- **PHA-1 (ngày 1→1001):** `conditional_lamp` sinh **MAGIC** (Gen chỉ ĐỌC số dư, không đụng LAMP). User tiêu MAGIC dịch vụ. Idle → 1 LAMP về pot/ngày. User **CHƯA sở-hữu/không-rút** LAMP.
- **PHA-2 (từ ngày 1002):** anti-idle DỪNG. Phần LAMP sống sót **vest 1 LAMP/ngày** `conditional_lamp → vested_unlocked` = user **SỞ HỮU thật** → **rút/bán/tự-Gen** (`claim-vested`).

**Trạng thái tổng:** gần như TẤT CẢ endpoint là ⚪ **spec/chờ Long build**. SuperApp **dựng UI trước với mock**, nối API sau.

**Convention chung (bám code thật):**
- Prefix `/api/v1`. Path `/activation`, `/wallet`. Body JSON **snake_case**. Response bọc `DataResponse<T>` = `{code, message, result}` (`DataResponse.java:17-22`); `code=1000` = OK.
- **Auth:** `Authorization: Bearer <session_token>` (JWT "session", TTL 24h). Controller inject `AuthenticatedUser{userDid, tokenType}` (`AuthenticatedUser.java:10`). "public" = query bằng DID không cần token (mẫu `/wallet/{did}/balance`, `WalletController.java:33-39`).
- **Đơn vị:** LAMP số nguyên; on-chain `oildrop = LAMP × 10⁶`. MAGIC: `nanogic = MAGIC × 10⁹`. API trả cả hai khi hữu ích. "NGÀY" = `slot/86400`, `SLOTS_PER_DAY=86_400`, gốc = `vest_start`.
- **Ranh giới ký:** chuyển-giá-trị on-chain (GetLAMP, ClaimVested rút-vested) theo mẫu **build-unsigned → Enclave witness → submit** (giống `ActivationController.java:67-90`). Đọc-trạng-thái (vault status, pot, MAGIC) là GET thuần.

---

## 1. Bảng tổng endpoint

| # | Method | Path | Mục đích | Auth | Trạng thái |
|---|---|---|---|---|---|
| 1a | POST | `/activation/getlamp/build` | GetLAMP bước 1 — đọc pot, tính D, build unsigned tx vào-vault-user+khoá | auth | ⚪ chờ Long |
| 1b | POST | `/activation/getlamp/submit` | GetLAMP bước 2 — submit tx đã ký → tạo vault | auth | ⚪ chờ Long |
| 2 | GET | `/activation/vault/{did}` | Trạng thái vault: **pha + ngày + conditional vs vested + MAGIC + anti-idle** | public | ⚪ chờ Long |
| 3 | GET | `/activation/vault/{did}/magic` | MAGIC-hàng-ngày (yield ĐỌC-số-dư + lịch drip) | public | ⚪ chờ Long + MAGIC |
| 4a | POST | `/activation/claim-vested/build` | **PHA-2:** build tx rút LAMP đã-sở-hữu (`vested_unlocked`) ra ví/tự-Gen | auth | ⚪ chờ Long |
| 4b | POST | `/activation/claim-vested/submit` | **PHA-2:** submit tx rút vested đã ký | auth | ⚪ chờ Long |
| 4c | POST | `/activation/abandon-phase1/build` | **PHA-1 (tuỳ chọn):** từ-bỏ — trả `conditional_lamp` còn lại về pot | auth | ⚪ chờ Long |
| 5a | POST | `/activation/getmagic/quote` | GetMAGIC — báo giá mua CARP bằng fiat | auth | ⚪ chờ Long + GreenBack |
| 5b | POST | `/activation/getmagic/checkout` | GetMAGIC — khởi tạo thanh toán fiat → CARP | auth | ⚪ chờ Long + GreenBack |
| 5c | GET | `/activation/getmagic/{order_id}` | GetMAGIC — trạng thái đơn (poll) | auth | ⚪ chờ Long + GreenBack |
| 6 | GET | `/activation/gen-entry` | **Ranh giới:** trỏ SDK MAGIC (engine Gen ĐỌC-số-dư, tự-động-nền) | — | thông tin (xem §5) |
| 7 | GET | `/activation/pot` | Sức khoẻ pot User — pot_balance + D hiện tại | public | ⚪ chờ Long |

> **BỎ so với v3:** `/activation/close/build`, `/activation/close/submit` (CloseBorrow trả-toàn-bộ-về-pot). **THÊM:** 4a/4b **`claim-vested`** (rút LAMP đã-sở-hữu PHA-2 — anh yêu cầu), 4c `abandon-phase1` (tuỳ chọn, từ-bỏ PHA-1). Endpoint 2 THÊM field `phase, conditional_lamp, vested_unlocked, days_elapsed`.
>
> Không endpoint nào đang live. SuperApp **dựng UI + mock trước**, gọi thật khi Long ship.

---

## 2. Chi tiết từng endpoint

### 1a. POST `/activation/getlamp/build` — GetLAMP bước 1

Backend đọc `pot_balance` → `D = min(1001, ⌊pot/1_000_000⌋)` (Feat-Math §3.1) → build unsigned tx: (a) spend pot chuyển D LAMP **vào vault user**; (b) tạo vault `conditional_lamp=D, vested_unlocked=0, reclaimed_to_pot=0, vest_start_slot=current, last_tick_day=0, did_commit`. Trả unsigned CBOR cho Enclave ký.

- **Auth:** Bearer. `auth.userDid()` = chủ vault.
- **Precondition:** user CHƯA có vault đang hoạt động (1 DID = 1 vault, I-ACT-10). Có rồi → `VAULT_ALREADY_EXISTS`.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `wallet_address` | string | Bech32 Shelley, địa chỉ nhận vault. Bắt buộc. |
  | `did_commit` | string (hex) | tuỳ chọn; Core tự tính `blake2b_256(did‖salt)` thì gửi, không thì backend sinh. **Chờ MAGIC-team** (§4). |

- **Response `result`:**

  ```json
  {
    "unsigned_tx_cbor": "<hex>",
    "required_signer_key_hash": "<hex blake2b-224>",
    "vault_address": "<addr...>",
    "d_lamp": 1001,
    "d_oildrop": 1001000000,
    "pot_balance_lamp": 5000000000,
    "vest_start_slot": 0,
    "phase1_days": 1001,
    "ttl_slot": 0
  }
  ```

  Chú thích UI: `d_lamp` = số LAMP vào **vault user** (khoá có-điều-kiện, trần 1001). `phase1_days=1001` = số ngày cam-kết PHA-1. SuperApp hiển thị **"Nhận D LAMP vào vault của bạn — dùng dịch vụ mỗi ngày để giữ trọn. Sau 1001 ngày, phần còn lại thành CỦA BẠN (rút/bán được)."** KHÔNG hiển thị "nhận LAMP để bán ngay".
- **Error codes:** `VAULT_ALREADY_EXISTS`, `WALLET_NOT_REGISTERED` (1320), `POT_UNAVAILABLE`, `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long. Phụ thuộc: pot deploy (§6.4 Feat-Math), validator vault 2-pha (Tuân).

### 1b. POST `/activation/getlamp/submit` — GetLAMP bước 2

- **Auth:** Bearer. **State:** tx từ 1a, đã Enclave ký.
- **Request body:** `{ "signed_tx_cbor": "<hex>" }`.
- **Response `result`:** `{ "cardano_tx_hash": "<hash>", "vault_address": "<addr>", "status": "SUBMITTED" }`.
- **Error codes:** `CARDANO_TX_FAILED` (5101), `SIGNATURE_INVALID` (1403), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long.

### 2. GET `/activation/vault/{did}` — trạng thái vault 2-pha (màn dashboard chính)

Mẫu public như `/wallet/{did}/balance` (`WalletController.java:33`). Hiển thị **pha hiện tại + số ngày + conditional vs vested + MAGIC + cảnh-báo-anti-idle (chỉ PHA-1)**.

- **Auth:** public. **Precondition:** vault tồn tại; chưa GetLAMP → `result: null` (SuperApp hiện nút GetLAMP).
- **Response `result` (ví dụ PHA-1):**

  ```json
  {
    "did": "did:phoenix:...",
    "vault_address": "<addr>",
    "phase": 1,
    "days_elapsed": 5,
    "phase1_days_total": 1001,
    "days_to_phase2": 996,
    "initial_d_lamp": 1001,
    "conditional_lamp": 1000,
    "vested_unlocked": 0,
    "reclaimed_to_pot_lamp": 1,
    "vest_start_slot": 0,
    "magic_generated_total": "12.50",
    "magic_balance_current": "4.20",
    "anti_idle": {
      "used_today": false,
      "grace_active": false,
      "grace_days_left": 0,
      "min_magic_tx": null,
      "at_risk_lamp": 1,
      "warning": "Hôm nay chưa dùng dịch vụ — cuối ngày 1 LAMP-điều-kiện sẽ về pot (số dư còn 999, MAGIC hàng ngày giảm theo). Đây là LAMP chưa-kiếm-được, không phải tài sản của bạn."
    }
  }
  ```

- **Response `result` (ví dụ PHA-2):**

  ```json
  {
    "did": "did:phoenix:...",
    "phase": 2,
    "days_elapsed": 1050,
    "phase1_days_total": 1001,
    "days_to_phase2": 0,
    "initial_d_lamp": 1001,
    "conditional_lamp": 951,
    "vested_unlocked": 48,
    "vested_claimed": 0,
    "reclaimed_to_pot_lamp": 2,
    "vest_per_day": 1,
    "vest_remaining_days": 951,
    "magic_generated_total": "1240.00",
    "magic_balance_current": "9.10",
    "anti_idle": null
  }
  ```

  Chú thích:
  - `phase` = **1** (có-điều-kiện, ngày ≤1001) hoặc **2** (vesting-sở-hữu, ngày ≥1002). `days_elapsed` = `⌊(now − vest_start)/86400⌋`. `days_to_phase2` = `max(0, 1001 − days_elapsed)`.
  - **`conditional_lamp`** = LAMP-điều-kiện **hiện tại** (khoá, sinh-MAGIC, PHA-1-anti-idle-rút; PHA-2-vest-dần-đi). **User KHÔNG rút được.**
  - **`vested_unlocked`** = LAMP **đã-sở-hữu** chưa-rút (chỉ PHA-2, tăng 1/ngày). **User RÚT ĐƯỢC** qua `claim-vested`. `vested_claimed` = đã rút ra ví.
  - `reclaimed_to_pot_lamp` = tổng anti-idle PHA-1 đã rút về pot.
  - `magic_generated_total` / `magic_balance_current` = MAGIC engine sinh (ĐỌC-số-dư) / MAGIC còn chưa tiêu (đơn vị MAGIC, chuỗi thập phân từ nanogic).
  - `anti_idle` = **null trong PHA-2** (anti-idle DỪNG). Trong PHA-1: `used_today` (đã có ≥`MIN_MAGIC_TX` tiêu counterparty chưa), `grace_active` (còn 7 ngày onboarding), `min_magic_tx` (**null tới khi MAGIC-team gắn counterparty** §4), `at_risk_lamp` (=1 hoặc 0 nếu grace/đã-dùng/hết-số-dư).
- **Error codes:** `USER_DID_NOT_FOUND` (2002).
- **Trạng thái:** ⚪ chờ Long. `anti_idle` + `magic_*` phụ thuộc MAGIC-team (§4) — trước khi có: `magic_*` từ engine khi nối được; `anti_idle.used_today: null` (anti-idle chưa bật). `phase`/`conditional_lamp`/`vested_unlocked` tính được ngay từ validator (không chờ Gen).

### 3. GET `/activation/vault/{did}/magic` — MAGIC-hàng-ngày (yield ĐỌC-số-dư)

Màn phụ: user xem dòng MAGIC. **Gen = yield ĐỌC-số-dư** — cả Instant + Schedule đọc `(conditional_lamp + vested_unlocked)` để tính MAGIC, **KHÔNG đụng LAMP** (Feat-Math §3.3, WP §7.2 dòng 205).

- **Auth:** public.
- **Response `result`:**

  ```json
  {
    "did": "did:phoenix:...",
    "phase": 1,
    "gen_basis_lamp": 1000,
    "magic_per_epoch_est": "2.50",
    "magic_generated_total": "12.50",
    "last_fire_epoch": 0,
    "next_fire_epoch": 0,
    "note_readonly": "Gen CHỈ ĐỌC số dư LAMP để tính MAGIC — KHÔNG burn/không-tiêu LAMP (Feat-Math §3.3, /CARP).",
    "note_granularity": "MAGIC nền drip theo EPOCH (~5 ngày); tick anti-idle/vest theo NGÀY (Feat-Math §3.7, [cần chốt])"
  }
  ```

  Chú thích: `gen_basis_lamp` = **cơ-sở-đọc** = `conditional_lamp + vested_unlocked_chưa_rút` (số-dư Gen đọc để tính suất). Anti-idle (PHA-1) / rút-vested (PHA-2) làm cơ-sở giảm → MAGIC-yield giảm. `magic_per_epoch_est` từ engine (/CARP). **[CẦN CHỐT]** daily-vs-epoch (Feat-Math §3.7) — chốt drip-daily thì đổi field sang `magic_per_day`.
- **Trạng thái:** ⚪ chờ Long + MAGIC-team (engine Gen ĐỌC-số-dư + hoà §3.7).

### 4a. POST `/activation/claim-vested/build` — PHA-2: rút LAMP đã-sở-hữu bước 1

**Chỉ PHA-2 (ngày ≥ 1002).** User rút `vested_unlocked` (một phần/toàn bộ) ra ví ngoài (bán/chuyển) HOẶC để trong vault tự-Gen. LAMP đã-vest = user **sở hữu thật** (Feat-Math §3.5, I-ACT-8).

- **Auth:** Bearer (chủ vault — ký `owner_commit`). **State:** vault ở PHA-2, `vested_unlocked > 0`.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `amount_lamp` | int | > 0, ≤ `vested_unlocked` hiện tại. Rút toàn bộ = gửi `vested_unlocked`. |
  | `destination` | string | `"wallet"` (rút ra `wallet_address`) hoặc `"keep_in_vault"` (giữ tự-Gen). Mặc định `"wallet"`. |

- **Response `result`:** `{ "unsigned_tx_cbor": "<hex>", "required_signer_key_hash": "<hex>", "claim_amount_lamp": 48, "vested_remaining_after": 0, "ttl_slot": 0 }`.
- **Error codes:** `VAULT_NOT_FOUND`, `NOT_IN_PHASE2` (PHA-1 chưa vest), `CLAIM_AMOUNT_EXCEEDS_VESTED`, `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long + validator vault 2-pha (Tuân). **Không** phụ thuộc Gen/GreenBack — chỉ cần validator vault.

### 4b. POST `/activation/claim-vested/submit` — PHA-2: rút vested bước 2

- **Request body:** `{ "signed_tx_cbor": "<hex>" }`.
- **Response `result`:** `{ "cardano_tx_hash": "<hash>", "claimed_lamp": 48, "status": "CLAIMED" }`.
- **Error codes:** `CARDANO_TX_FAILED` (5101), `SIGNATURE_INVALID` (1403).
- **Trạng thái:** ⚪ chờ Long.

### 4c. POST `/activation/abandon-phase1/build` — PHA-1 (tuỳ chọn): từ-bỏ

**Chỉ PHA-1.** User chủ động từ-bỏ trước khi kiếm được → trả **toàn bộ `conditional_lamp` còn lại** về pot (chưa-sở-hữu nên KHÔNG rút ra ví). Mặc định user không cần dùng — cứ để anti-idle tự thu-hồi (Feat-Math §3.5).

- **Auth:** Bearer (chủ vault). **State:** vault PHA-1.
- **Request body:** `{ "confirm": true }`.
- **Response `result`:** `{ "unsigned_tx_cbor": "<hex>", "returned_to_pot_lamp": 998, "ttl_slot": 0 }`.
- **Error codes:** `VAULT_NOT_FOUND`, `ALREADY_IN_PHASE2` (PHA-2 dùng claim-vested thay), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long. **[CẦN CHỐT]** tương-tác `gen_schedules` đang chạy nếu engine Gen có state (Feat-Math §3.7).

### 5a. POST `/activation/getmagic/quote` — GetMAGIC báo giá

Mua **CARP bằng fiat** (cửa mint CARP hợp lệ = cầu-vào-thật, Feat-Math §2.2; KHÁC "mint từ tiêu MAGIC" bị cấm F1). Quote trước checkout.

- **Auth:** Bearer.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `fiat_currency` | string | ISO-4217 (`VND`/`USD`...). Bắt buộc. |
  | `fiat_amount` | long | > 0, đơn vị nhỏ nhất. Hoặc `carp_amount` (một trong hai). |
  | `carp_amount` | long | > 0, lượng CARP muốn mua. Một trong hai. |

- **Response `result`:**

  ```json
  {
    "quote_id": "<uuid>",
    "fiat_currency": "VND",
    "fiat_amount": 250000,
    "carp_amount": 100,
    "rate": "2500 VND / CARP",
    "fee_breakdown": { "fx_buffer": 5000, "network": 0 },
    "expires_at": 0
  }
  ```

  Chú thích: CARP neo `1 CARP = 1 MAGIC` (Feat-Math I-ACT-9); FX buffer theo lớp-đệm-FX (memory `phoenixkey-anchor-economics`). Tỷ giá + backing thuộc **CARP-team/GreenBack**.
- **Error codes:** `GETMAGIC_UNSUPPORTED_CURRENCY`, `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long + GreenBack/CARP-team.

### 5b. POST `/activation/getmagic/checkout` — khởi tạo thanh toán

- **Auth:** Bearer. **Request body:** `{ "quote_id": "<uuid>", "payment_method": "vietqr|card|..." }`.
- **Response `result`:**

  ```json
  {
    "order_id": "<uuid>",
    "payment_url": "https://... (VietQR / gateway redirect)",
    "carp_amount": 100,
    "status": "PENDING_PAYMENT",
    "expires_at": 0
  }
  ```

  Sau gateway confirm → CARP về account user (poll 5c). **SSE lifecycle** có thể thêm theo mẫu `ActivationController.java:115`.
- **Error codes:** `GETMAGIC_QUOTE_EXPIRED`, `GETMAGIC_PAYMENT_INIT_FAILED`.
- **Trạng thái:** ⚪ chờ Long + gateway fiat.

### 5c. GET `/activation/getmagic/{order_id}` — trạng thái đơn

- **Auth:** Bearer (chủ đơn).
- **Response `result`:** `{ "order_id": "<uuid>", "status": "PENDING_PAYMENT|PAID|CARP_DELIVERED|FAILED|EXPIRED", "carp_amount": 100, "cardano_tx_hash": "<hash|null>", "fail_reason": "<string|null>" }`.
- **Error codes:** `GETMAGIC_ORDER_NOT_FOUND`.
- **Trạng thái:** ⚪ chờ Long.

### 6. GET `/activation/gen-entry` — RANH GIỚI: trỏ SDK MAGIC

Engine Gen **ĐỌC-số-dù** `(conditional_lamp + vested_unlocked)` → sinh MAGIC tự động, **KHÔNG cần user thao-tác "đưa LAMP vào Gen"** và **KHÔNG đụng LAMP** (Feat-Math §3.3). Ở model 2-pha, **Gen là tự-động-nền** trên số-dư-vault.

- **RANH GIỚI RÕ:** engine Gen sống ở **repo MAGIC/CARP**, KHÔNG thuộc PhoenixKey-Database. Việc dựng schedule/đọc-số-dư do **backend PhoenixKey + MAGIC/CARP-team phối** (Feat-Math §3.3), KHÔNG phải user bấm từng lần.
- **Endpoint này chỉ THÔNG TIN:** trả trạng-thái Gen của vault (cơ-sở-đọc số-dư, MAGIC đã fire) + link SDK. **KHÔNG có endpoint "user đẩy LAMP vào Gen".** PHA-2, nếu user muốn tự-Gen `vested_unlocked` sau khi sở-hữu → dùng SDK MAGIC/CARP trực-tiếp (ngoài Activation API).
- **[CẦN CHỐT §3.7 Feat-Math]:** spell-out engine Gen ĐỌC-số-dư on-chain (thay code cũ fire→Treasury); xác nhận cả Instant+Schedule /CARP đều đọc-số-dư; daily-vs-epoch. SuperApp **coi Gen là tự-động-nền**, chỉ hiển thị MAGIC-yield (endpoint 3).
- **Trạng thái:** thông tin — dùng endpoint 3 cho MAGIC-yield.

### 7. GET `/activation/pot` — sức khoẻ pot

- **Auth:** public.
- **Response `result`:**

  ```json
  {
    "pot_balance_lamp": 5000000000,
    "current_d_lamp": 1001,
    "d_cap": 1001,
    "scale": 1000000,
    "saturated": true
  }
  ```

  `current_d_lamp = min(1001, ⌊pot/1e6⌋)` = D một user mới sẽ nhận vào vault nếu GetLAMP ngay. `saturated` = pot ≥ 1_001_000_000. Banner "GetLAMP hôm nay: D LAMP vào vault của bạn". **Lưu ý pot:** PHA-2 LAMP-vest **rời-hệ** sang user → pot cần bù dòng-vest-ra (Feat-Math §3.6, §6.5).
- **Trạng thái:** ⚪ chờ Long + pot deploy.

---

## 3. Luồng UI đề xuất (SuperApp hình dung)

```
[Màn Onboarding]
  Enclave keygen (vân tay) → ví Phoenix tạo   (Core, ngoài API này)
        │
        ▼
[Màn GetLAMP]  ← GET /activation/pot (banner "Nhận D LAMP vào vault — dùng dịch vụ, sau 1001 ngày phần còn lại thành của bạn")
  Nút "Nhận LAMP" →
    1. POST /activation/getlamp/build   → unsigned_tx_cbor
    2. Enclave ký (Core)
    3. POST /activation/getlamp/submit  → cardano_tx_hash
        │
        ▼
[Vault Dashboard]  ← GET /activation/vault/{did}  (màn chính, quay lại HẰNG NGÀY)
  ┌─ CHỈ-BÁO PHA: phase=1 "Đang kiếm (còn N ngày)" | phase=2 "Đang sở-hữu"
  ├─ "LAMP điều-kiện (khoá, đang sinh MAGIC)": conditional_lamp   (KHÔNG có nút rút — PHA-1)
  ├─ "LAMP đã-của-bạn (rút/bán được)": vested_unlocked            (PHA-2, nút [Rút LAMP])
  ├─ "MAGIC hàng ngày trên số dư": magic_generated_total + magic_balance_current
  │     └─ GET .../magic  vẽ dòng MAGIC-yield (Gen ĐỌC-số-dư)
  ├─ [PHA-1] Cảnh báo anti-idle: nếu used_today=false & !grace
  │     "Hôm nay chưa dùng — cuối ngày mất 1 LAMP-điều-kiện về pot (chưa-kiếm-được)"
  │     (ẩn tới khi MAGIC-team gắn counterparty; ẩn hẳn trong PHA-2)
  ├─ [PHA-1] "Đã trả về pot X LAMP do idle": reclaimed_to_pot_lamp
  ├─ [PHA-2] "Mỗi ngày +1 LAMP thành của bạn": vest_per_day, vest_remaining_days
  ├─ Nút [GetMAGIC] → màn mua CARP
  └─ [PHA-2] Nút [Rút LAMP đã-của-bạn] → POST /activation/claim-vested/build → ký → /claim-vested/submit
  └─ [PHA-1] (tuỳ chọn) [Từ-bỏ] → POST /activation/abandon-phase1/build (trả conditional về pot)
        │
        ▼
[Màn GetMAGIC]
    POST /activation/getmagic/quote → tỷ giá + FX buffer
    POST /activation/getmagic/checkout → payment_url (VietQR/redirect)
    poll GET /activation/getmagic/{order_id} tới CARP_DELIVERED
```

**Nguyên tắc UX bám spec:**
- Chỉ **1 nút GetLAMP**, user không cần hiểu tỷ giá (Feat-Math §1 (d)). ADA/phí do Feecover lo — KHÔNG hiển thị "phí ADA".
- **Phân biệt rõ 2 loại LAMP trên UI:** `conditional_lamp` (khoá, PHA-1 chưa-của-user, sinh-MAGIC) vs `vested_unlocked` (PHA-2, **của user, rút-được**). ĐỪNG gộp một con số.
- **PHA-1:** KHÔNG có nút rút LAMP (chưa sở-hữu). Nhấn "MAGIC hàng ngày" là thứ user hưởng. Anti-idle: ngôn ngữ "dùng hằng ngày để giữ" + "LAMP-điều-kiện chưa-kiếm-được", KHÔNG "phạt tài sản".
- **PHA-2:** hiện nút **[Rút LAMP]** (`claim-vested`) — user sở-hữu thật, rút/bán/tự-Gen. Vest **nhỏ-giọt 1/ngày** (không mở hết cùng lúc).
- Chuyển-pha (ngày 1001→1002): UI đổi banner từ "Đang kiếm" → "Đã hoàn thành cam-kết — LAMP đang thành của bạn 1/ngày".

---

## 4. Chờ backend / phụ thuộc chặn

| Phụ thuộc | Chặn endpoint | Đội | Ghi chú |
|---|---|---|---|
| **Pot deploy** (nạp vốn ban đầu + bù dòng-vest-ra, §6.4/§6.5 Feat-Math) | 1a GetLAMP, 7 pot | LAMP-team + Long | Không pot → không tính D. PHA-2 LAMP rời-hệ → pot cần bù (sub-open). |
| **Validator vault 2-pha** (`conditional_lamp`/`vested_unlocked`, ranh-giới-pha n=1001, anti-idle→pot PHA-1, **vest 1/ngày PHA-2**, **ClaimVested** ký-owner, Gen-đọc-số-dư-KHÔNG-spend-LAMP, `did_commit` gate) | 1a/1b, 2, 4a/4b/4c | Tuân (Aiken) | Tái dùng `LoyaltyHolding`. Khoá+vest+claim thuần build được ngay (không chờ Gen). |
| **`did_commit` per-DID + counterparty field** | field `anti_idle` (endpoint 2) | **MAGIC-team (BLOCKER)** | `EngageDatum.did_commit=#""`. Không counterparty → G3 hở → KHÔNG bật anti-idle production. Tới đó `anti_idle.used_today=null`. |
| **Engine Gen ĐỌC-số-dư + hoà §3.7** | `magic_*` (endpoint 2/3), 6 gen-entry | **MAGIC/CARP-team (BLOCKER kiến trúc)** | /CARP chốt nguyên-lý "Gen đọc-số-dư không-đụng-LAMP" nhưng **chưa spell-out on-chain**; code cũ fire→Treasury lỗi-thời. Cần engine đọc-VaultDatum→drip→không-spend-LAMP (Feat-Math §3.7/§6.6). |
| **Resolve API point-in-time** | anti-idle job | Long | DB hiện chỉ state-hiện-tại (Feat-Math §6.2). |
| **GreenBack interface** | 5a/5b/5c GetMAGIC (backing) | CARP-team | CHỈ khai interface, không chạm peg. Fiat→CARP chạy trước; backing treo tới khi có. + shadow-price nếu PHA-2 bán năng-lực. |
| **`fee_refill_lamp` + cân-đối vest-ra** | (nạp pot, không chặn UI) | LAMP + Long | Tham số chờ chốt (§6.5). Pilot: anti-idle + treasury đủ; PHA-2-vest-ra cần theo dõi. |
| **`MIN_MAGIC_TX`** | ngưỡng `used_today` | anh Aladin (TẠM 07-06: 10% gen-able từ `conditional_lamp`) | Sub-open granularity gen-able. |

**Thứ tự SuperApp nên dựng UI:**
1. **Ngay (mock đủ):** GetLAMP + Vault Dashboard 2-pha (1a/1b/2/7) — logic pha/conditional/vested tính từ validator, không chờ MAGIC/CARP.
2. **Sau (Long ship):** ClaimVested PHA-2 (4a/4b) + abandon PHA-1 (4c).
3. **Chờ MAGIC/CARP-team:** MAGIC-yield (endpoint 3) + cảnh-báo anti-idle (null tới đó).
4. **Chờ CARP/GreenBack + gateway:** GetMAGIC (5a/5b/5c).

---

## 5. Mã lỗi mới đề xuất (Long thêm `ErrorCode.java`)

Bám quy tắc hiện có (131x = Activation cũ, 132x = Wallet). Nhánh **135x — Activation Vault 2-pha**:

| Đề xuất code | Tên | HTTP | Dùng ở |
|---|---|---|---|
| 1350 | `VAULT_ALREADY_EXISTS` | 409 | 1a |
| 1351 | `VAULT_NOT_FOUND` | 404 | 2, 4a, 4c |
| 1352 | `POT_UNAVAILABLE` | 503 | 1a, 7 |
| 1353 | `NOT_IN_PHASE2` | 409 | 4a (rút-vested khi còn PHA-1) |
| 1354 | `CLAIM_AMOUNT_EXCEEDS_VESTED` | 400 | 4a |
| 1355 | `ALREADY_IN_PHASE2` | 409 | 4c (abandon khi đã PHA-2) |
| 1360 | `GETMAGIC_UNSUPPORTED_CURRENCY` | 400 | 5a |
| 1361 | `GETMAGIC_QUOTE_EXPIRED` | 410 | 5b |
| 1362 | `GETMAGIC_ORDER_NOT_FOUND` | 404 | 5c |
| 1363 | `GETMAGIC_PAYMENT_INIT_FAILED` | 502 | 5b |

**BỎ so với v3:** không còn `CLOSE`/`CloseBorrow` codes. Tái dùng: `UNAUTHORIZED` (1304), `WALLET_NOT_REGISTERED` (1320), `USER_DID_NOT_FOUND` (2002), `SIGNATURE_INVALID` (1403), `CARDANO_TX_FAILED` (5101).

---

## 6. Ghi trung thực cho reviewer

- **KHÔNG endpoint nào live.** Toàn bộ spec — SuperApp dựng UI + mock, nối khi Long ship.
- **Model = 2 PHA.** PHA-1 user hưởng MAGIC, KHÔNG sở-hữu/rút LAMP. PHA-2 phần sống sót vest thành **SỞ HỮU**, user **rút/bán/tự-Gen** (`claim-vested`). Dashboard nhấn **pha + conditional vs vested + MAGIC-hàng-ngày + anti-idle(PHA-1)**.
- **Gen = ĐỌC-số-dư, KHÔNG đụng LAMP** (endpoint 3, WP §7.2 dòng 205, /CARP). Tự-động-nền, không UI "chọn Gen".
- **Tên path là đề xuất** — chốt với Long trước khi hard-code.
- **anti-idle + MAGIC field trả null/treo** cho tới khi MAGIC-team gắn counterparty + spell-out engine đọc-số-dư §3.7 (Feat-Math).
- **[CẦN CHỐT]:** engine Gen đọc-số-dư on-chain (endpoint 3/6, Feat-Math §3.7); daily-vs-epoch MAGIC; pot bù dòng-vest-ra PHA-2 (§6.5); PHA-2-vest-sở-hữu khớp WP §7.2.
