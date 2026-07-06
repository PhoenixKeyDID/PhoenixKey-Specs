# PhoenixKey — Activation API (model GetLAMP mới) cho SuperApp

> **Đối tượng:** agent SuperApp (Thư) dựng UI Activation. **Không code** — chỉ đặc tả API.
> **Nguồn chuẩn:** `spec-proposals/PhoenixKey-Activation-Feat-Math.md` (v2, 2026-07-06) — luồng GetLAMP / vault vesting / anti-idle / GetMAGIC.
> **Convention backend bám thật:** `PhoenixKey-Database/.../controller/` (ActivationController, WalletController), `common/DataResponse.java`, `security/AuthenticatedUser.java`, `exception/ErrorCode.java`.
> SPEC PROPOSAL — chờ anh Aladin + Long chốt. Cập nhật 2026-07-06.

---

## 0. Điều SuperApp cần biết trước

**Model cũ ĐÃ BỎ.** Các endpoint `/activation/initiate`, `/{id}/confirm-payment`, `/{id}/build-tx`, `/{id}/submit-tx`, `/{id}/cancel` (luồng "200k VND → 1001 LAMP + 10 ADA qua Genie") còn nằm trong code hiện tại (`ActivationController.java:37-99`) nhưng **thuộc model VND-Genie đã huỷ** (Feat-Math §1 dòng 27). **KHÔNG dựng UI theo nhóm đó.** Model mới = **GetLAMP** (phát LAMP qua vault vesting, không phát ADA, không nạp VND).

**Trạng thái tổng:** gần như TẤT CẢ endpoint dưới đây là ⚪ **spec/chờ Long build**. SuperApp **dựng UI trước với mock**, nối API sau. Cột "Trạng thái" mỗi endpoint ghi rõ.

**Convention chung (bám code thật):**
- Prefix: `/api/v1`. Path controller Activation: `/activation`. Wallet: `/wallet`.
- Body JSON **snake_case**. Response luôn bọc `DataResponse<T>` = `{ "code": int, "message": string, "result": T }` (`DataResponse.java:17-22`). `code=1000` = OK; `message`/`result` có thể null (JsonInclude NON_NULL).
- **Auth:** `Authorization: Bearer <session_token>` (JWT type "session", TTL 24h). Controller inject `AuthenticatedUser{userDid, tokenType}` (`AuthenticatedUser.java:10`). Endpoint "auth" dưới đây = cần Bearer; endpoint "public" = query bằng DID không cần token (mẫu `/wallet/{did}/balance`, `WalletController.java:33-39`).
- **Đơn vị:** LAMP hiển thị số nguyên LAMP; on-chain `oildrop = LAMP × 10⁶` (Feat-Math §3.1). API trả **cả hai** khi hữu ích (`amount_lamp` cho UI, `amount_oildrop` cho ký/đối chiếu). "NGÀY" = ranh giới `slot/86400`, `SLOTS_PER_DAY=86_400`.
- **Ranh giới ký:** phần chuyển giá trị on-chain (GetLAMP, ClaimUnlocked, Gen-entry) theo mẫu **2 bước build-unsigned → Enclave witness → submit** (giống `ActivationController.java:67-90`). Backend build tx chưa ký; Core/Enclave ký bằng controller-DID; submit lại. Endpoint đọc trạng thái (vault status, pot health, lịch vesting) là 1 bước GET thuần.

---

## 1. Bảng tổng endpoint

| # | Method | Path | Mục đích | Auth | Trạng thái |
|---|---|---|---|---|---|
| 1a | POST | `/activation/getlamp/build` | GetLAMP bước 1 — backend đọc pot, tính D, build unsigned tx phát+khoá vault | auth | ⚪ chờ Long |
| 1b | POST | `/activation/getlamp/submit` | GetLAMP bước 2 — submit tx đã Enclave ký → tạo vault | auth | ⚪ chờ Long |
| 2 | GET | `/activation/vault/{did}` | Trạng thái vault vesting của user (D, locked, unlocked, ngày trôi, cảnh báo anti-idle) | public | ⚪ chờ Long |
| 3a | POST | `/activation/claim/build` | ClaimUnlocked bước 1 — build unsigned tx rút phần LAMP đã mở | auth | ⚪ chờ Long |
| 3b | POST | `/activation/claim/submit` | ClaimUnlocked bước 2 — submit tx đã ký | auth | ⚪ chờ Long |
| 4a | POST | `/activation/getmagic/quote` | GetMAGIC — báo giá mua CARP bằng fiat | auth | ⚪ chờ Long + GreenBack |
| 4b | POST | `/activation/getmagic/checkout` | GetMAGIC — khởi tạo phiên thanh toán fiat → CARP | auth | ⚪ chờ Long + GreenBack |
| 4c | GET | `/activation/getmagic/{order_id}` | GetMAGIC — trạng thái đơn (poll) | auth | ⚪ chờ Long + GreenBack |
| 5 | POST | `/activation/gen/build` | Gen-entry — đưa unlocked LAMP vào ScheduleGen/InstantGen (**ranh giới: trỏ SDK MAGIC**, xem §5) | auth | ⚪ chờ MAGIC-SDK |
| 6a | GET | `/activation/pot` | Sức khoẻ pot User — pot_balance + D hiện tại | public | ⚪ chờ Long |
| 6b | GET | `/activation/vault/{did}/schedule` | Lịch vesting đầy đủ (mốc ngày → unlocked tích luỹ) | public | ⚪ chờ Long |

> Không endpoint nào đang live. Cột trạng thái là sự thật để SuperApp planning: **dựng UI + mock trước**, gọi thật khi Long ship.

---

## 2. Chi tiết từng endpoint

### 1a. POST `/activation/getlamp/build` — GetLAMP bước 1 (build unsigned tx)

Backend đọc `pot_balance` → tính `D = min(1001, ⌊pot/1_000_000⌋)` (Feat-Math §3.1) → build unsigned tx: (a) spend pot chuyển D LAMP; (b) tạo vault vesting `locked=D, unlocked=0, vest_start_slot=current, did_commit`. Trả unsigned CBOR cho Enclave ký.

- **Auth:** Bearer session_token. `auth.userDid()` = chủ vault.
- **State precondition:** user CHƯA có vault đang hoạt động (1 DID = 1 vault, I-ACT-8). Nếu đã có → lỗi `VAULT_ALREADY_EXISTS`.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `wallet_address` | string | Bech32 Shelley (`^addr_(test1|1)[a-z0-9]+$`), địa chỉ nhận vault. Bắt buộc. |
  | `did_commit` | string (hex) | tuỳ chọn; nếu Core tự tính `blake2b_256(did‖salt)` thì gửi lên, không thì backend sinh. **Chờ MAGIC-team** (§4). |

- **Response `result` (`DataResponse`):**

  ```json
  {
    "unsigned_tx_cbor": "<hex>",
    "required_signer_key_hash": "<hex blake2b-224>",
    "vault_address": "<addr...>",
    "user_address": "<addr...>",
    "d_lamp": 1001,
    "d_oildrop": 1001000000,
    "pot_balance_lamp": 5000000000,
    "vest_start_slot": 0,
    "ttl_slot": 0
  }
  ```

  Chú thích UI: `d_lamp` = số LAMP user nhận lần này (đã trừ trần 1001). SuperApp hiển thị "Bạn sẽ nhận D LAMP, mở dần 1 LAMP/ngày trong D ngày".
- **Error codes:** `VAULT_ALREADY_EXISTS` (14xx mới), `WALLET_NOT_REGISTERED` (1320), `POT_UNAVAILABLE` (pot chưa deploy / D=0 pilot — 5xxx mới), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long. Phụ thuộc: pot deploy (§6.4 Feat-Math), validator vault-vesting (Tuân).

### 1b. POST `/activation/getlamp/submit` — GetLAMP bước 2 (submit)

- **Auth:** Bearer. **State:** tx từ bước 1a, đã Enclave ký.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `signed_tx_cbor` | string (hex) | signed Cardano tx CBOR. Bắt buộc. |

- **Response `result`:** `{ "cardano_tx_hash": "<hash>", "vault_address": "<addr>", "status": "SUBMITTED" }`.
- **Error codes:** `CARDANO_TX_FAILED` (5101), `SIGNATURE_INVALID` (1403), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long.

### 2. GET `/activation/vault/{did}` — trạng thái vault

Màn dashboard chính. Mẫu public như `/wallet/{did}/balance` (`WalletController.java:33`).

- **Auth:** public (query bằng DID).
- **State precondition:** vault tồn tại. Nếu chưa GetLAMP → trả `result: null` + `code` báo chưa có vault (SuperApp hiển thị nút GetLAMP).
- **Response `result`:**

  ```json
  {
    "did": "did:phoenix:...",
    "vault_address": "<addr>",
    "total_d_lamp": 1001,
    "locked_lamp": 900,
    "unlocked_lamp": 101,
    "unlocked_claimable_lamp": 101,
    "vest_start_slot": 0,
    "vest_start_day": 0,
    "current_day": 101,
    "days_elapsed": 101,
    "days_remaining": 900,
    "reclaimed_total_lamp": 0,
    "anti_idle": {
      "used_today": false,
      "grace_active": false,
      "grace_days_left": 0,
      "min_magic_tx": null,
      "at_risk_lamp": 1,
      "warning": "Hôm nay chưa dùng dịch vụ — 1 LAMP chưa-mở sẽ về pot cuối ngày"
    }
  }
  ```

  Chú thích:
  - `unlocked(t)=min(t,D)`, `locked(t)=max(0,D−t)` với `t=days_elapsed` (Feat-Math §3.2).
  - `unlocked_claimable_lamp` = phần đã mở còn nằm trong vault, chưa Claim ra ví ngoài.
  - `anti_idle.used_today` = đã có ≥`MIN_MAGIC_TX` tiêu-MAGIC counterparty hôm nay chưa (§3.3). `grace_active` = còn trong 7 ngày onboarding (`GRACE_DAYS=7`, §3.4). `min_magic_tx` = **null cho tới khi anh Aladin chốt** (OPEN D). `at_risk_lamp` = lượng locked sẽ bị thu nếu hết ngày vẫn idle (=`RECLAIM_UNIT=1`, hoặc 0 nếu grace/đã dùng/hết locked).
- **Error codes:** `USER_DID_NOT_FOUND` (2002).
- **Trạng thái:** ⚪ chờ Long. `anti_idle` phụ thuộc counterparty-field (MAGIC-team, §4) — trước khi có, backend trả `used_today: null` + `warning: null` (anti-idle chưa bật production, Feat-Math §8.5).

### 3a. POST `/activation/claim/build` — ClaimUnlocked bước 1

Rút phần `unlocked` từ vault ra ví ngoài (để bán tự do hoặc đưa vào Gen). CHỈ chạm `unlocked`, không đụng `locked` (I-ACT-9).

- **Auth:** Bearer. **State:** `unlocked_claimable_lamp > 0`.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `amount_lamp` | long | 1 ≤ amount ≤ `unlocked_claimable_lamp`. Bắt buộc. |
  | `destination_address` | string | Bech32; tuỳ chọn, mặc định ví chủ vault. |

- **Response `result`:** `{ "unsigned_tx_cbor": "<hex>", "required_signer_key_hash": "<hex>", "amount_lamp": 101, "amount_oildrop": 101000000, "ttl_slot": 0 }`.
- **Error codes:** `CLAIM_AMOUNT_EXCEEDS_UNLOCKED` (14xx mới), `VAULT_NOT_FOUND` (14xx mới), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long.

### 3b. POST `/activation/claim/submit` — ClaimUnlocked bước 2

- **Request body:** `{ "signed_tx_cbor": "<hex>" }`.
- **Response `result`:** `{ "cardano_tx_hash": "<hash>", "amount_lamp": 101, "status": "SUBMITTED" }`.
- **Error codes:** `CARDANO_TX_FAILED` (5101), `SIGNATURE_INVALID` (1403).
- **Trạng thái:** ⚪ chờ Long.

### 4a. POST `/activation/getmagic/quote` — GetMAGIC báo giá

Mua **CARP bằng fiat** (cửa mint CARP hợp lệ = cầu-vào-thật, Feat-Math §2.2 dòng 82; KHÁC "mint từ tiêu MAGIC" bị cấm F1). Quote trước checkout.

- **Auth:** Bearer.
- **Request body:**

  | field | type | validation |
  |---|---|---|
  | `fiat_currency` | string | ISO-4217 (`VND`/`USD`...). Bắt buộc. |
  | `fiat_amount` | long | > 0, đơn vị nhỏ nhất (VND đồng). Hoặc dùng `carp_amount` (một trong hai). |
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

  Chú thích: CARP neo `1 CARP = 1 MAGIC` (Feat-Math I-ACT-7); FX buffer theo lớp-đệm-FX (memory `phoenixkey-anchor-economics`). Tỷ giá + cơ chế backing thuộc **CARP-team/GreenBack** — PhoenixKey chỉ khai interface (§3.5 Feat-Math).
- **Error codes:** `GETMAGIC_UNSUPPORTED_CURRENCY` (14xx mới), `UNAUTHORIZED` (1304).
- **Trạng thái:** ⚪ chờ Long + GreenBack/CARP-team.

### 4b. POST `/activation/getmagic/checkout` — GetMAGIC khởi tạo thanh toán

- **Auth:** Bearer.
- **Request body:** `{ "quote_id": "<uuid>", "payment_method": "vietqr|card|..." }`.
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

  Sau khi gateway confirm → CARP về ví/account user (poll qua 4c). **SSE lifecycle** có thể thêm theo mẫu `ActivationController.java:115` (`GET /activation/getmagic/{order_id}/events`) — chờ Long quyết.
- **Error codes:** `GETMAGIC_QUOTE_EXPIRED`, `GETMAGIC_PAYMENT_INIT_FAILED` (14xx mới).
- **Trạng thái:** ⚪ chờ Long + gateway fiat.

### 4c. GET `/activation/getmagic/{order_id}` — trạng thái đơn GetMAGIC

- **Auth:** Bearer (chủ đơn).
- **Response `result`:** `{ "order_id": "<uuid>", "status": "PENDING_PAYMENT|PAID|CARP_DELIVERED|FAILED|EXPIRED", "carp_amount": 100, "cardano_tx_hash": "<hash|null>", "fail_reason": "<string|null>" }`.
- **Error codes:** `GETMAGIC_ORDER_NOT_FOUND` (14xx mới).
- **Trạng thái:** ⚪ chờ Long.

### 5. POST `/activation/gen/build` — Gen-entry (RANH GIỚI: trỏ SDK MAGIC)

Đưa `unlocked` LAMP vào **ScheduleGen** (khoá LAMP kỳ-hạn, cam kết → MAGIC) hoặc **InstantGen** (tiêu ngay, decay → MAGIC). Feat-Math §2.2 dòng 90.

- **RANH GIỚI RÕ (đọc kỹ):** engine Gen sống ở **repo MAGIC** (`MAGIC/ScheduleGen`, `MAGIC/InstantGen`), KHÔNG thuộc PhoenixKey-Database. Có 2 lựa chọn kiến trúc, **chờ anh + MAGIC-team chốt**:
  - **(A) SuperApp gọi thẳng MAGIC-SDK** để build Gen tx (LAMP-đã-Claim làm input). PhoenixKey KHÔNG có endpoint này — dòng 5 trong bảng chỉ là **placeholder** để SuperApp biết luồng. → Nếu chọn A, **bỏ endpoint `/activation/gen/*`**, SuperApp tích hợp MAGIC-SDK.
  - **(B) PhoenixKey backend làm proxy mỏng** dựng Gen tx thay user (tiện cho UI 1 chỗ). Nếu chọn B thì endpoint dưới đây áp dụng.
- **(Nếu B) Request body:**

  | field | type | validation |
  |---|---|---|
  | `gen_type` | string | `SCHEDULE` \| `INSTANT`. Bắt buộc. |
  | `amount_lamp` | long | > 0, ≤ LAMP-đã-Claim khả dụng. |
  | `commit_epochs` | int | chỉ với SCHEDULE — số epoch cam kết. |

- **(Nếu B) Response `result`:** `{ "unsigned_tx_cbor": "<hex>", "gen_type": "SCHEDULE", "amount_lamp": 50, "expected_magic": 0, "ttl_slot": 0 }` (`expected_magic` do engine MAGIC tính).
- **Trạng thái:** ⚪ chờ quyết A/B + MAGIC-SDK. **Khuyến nghị:** SuperApp dựng UI Gen tách khỏi PhoenixKey (giả định A), coi Gen là bước "chuyển sang màn MAGIC".

### 6a. GET `/activation/pot` — sức khoẻ pot

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

  `current_d_lamp = min(1001, ⌊pot/1e6⌋)` = D một user mới sẽ nhận nếu GetLAMP ngay bây giờ. `saturated` = pot ≥ 1_001_000_000 (mọi user nhận trần). Dùng cho banner "GetLAMP hôm nay: D LAMP".
- **Trạng thái:** ⚪ chờ Long + pot deploy.

### 6b. GET `/activation/vault/{did}/schedule` — lịch vesting

- **Auth:** public.
- **Response `result`:**

  ```json
  {
    "did": "did:phoenix:...",
    "total_d_lamp": 1001,
    "vest_start_slot": 0,
    "slots_per_day": 86400,
    "unlocked_now_lamp": 101,
    "full_unlock_day": 1001,
    "next_unlock_slot": 0,
    "milestones": [
      { "day": 30, "unlocked_lamp": 30 },
      { "day": 100, "unlocked_lamp": 100 },
      { "day": 1001, "unlocked_lamp": 1001 }
    ]
  }
  ```

  Chú thích: đường thẳng 1 LAMP/ngày, `full_unlock_day = D` (D=1001 → ≈2,74 năm, chủ đích Feat-Math §3.2). SuperApp vẽ progress bar + "mở tiếp sau X giờ".
- **Trạng thái:** ⚪ chờ Long. (Client tự tính được từ endpoint 2 nếu backend chưa ship — chỉ cần `vest_start_slot` + `total_d` + đồng hồ.)

---

## 3. Luồng UI đề xuất (SuperApp hình dung)

```
[Màn Onboarding]
  Enclave keygen (vân tay) → ví Phoenix tạo   (Core, ngoài API này)
        │
        ▼
[Màn GetLAMP]  ← GET /activation/pot (banner "Nhận D LAMP hôm nay")
  Nút "Nhận LAMP" →
    1. POST /activation/getlamp/build   → unsigned_tx_cbor
    2. Enclave ký (Core)
    3. POST /activation/getlamp/submit  → cardano_tx_hash
        │
        ▼
[Vault Dashboard]  ← GET /activation/vault/{did}  (màn chính, quay lại HẰNG NGÀY)
  ┌─ Progress: unlocked / total_d  (+ GET .../schedule vẽ đường vesting)
  ├─ Ô "Đã mở, khả dụng": unlocked_claimable_lamp
  │     └─ nút [Claim]  → POST /activation/claim/build → ký → /claim/submit
  │     └─ nút [Sinh MAGIC] → màn Gen (SDK MAGIC, §5 ranh giới A)
  ├─ Cảnh báo anti-idle: nếu used_today=false & !grace
  │     "Hôm nay chưa dùng dịch vụ — 1 LAMP chưa-mở sẽ về pot" (không hiển thị tới khi MAGIC-team gắn counterparty)
  └─ Nút [GetMAGIC] → màn mua CARP
        │
        ▼
[Màn GetMAGIC]
    POST /activation/getmagic/quote → hiển thị tỷ giá + FX buffer
    POST /activation/getmagic/checkout → payment_url (VietQR/redirect)
    poll GET /activation/getmagic/{order_id} tới CARP_DELIVERED
```

**Nguyên tắc UX bám spec:**
- Chỉ **1 nút GetLAMP**, user không cần hiểu tỷ giá (Feat-Math §1 (d)). ADA/phí do Feecover lo — KHÔNG hiển thị "phí ADA".
- **Dump unlocked là HỢP LỆ** (I-ACT-9): nút Claim + bán tự do, không cảnh báo phạt.
- Anti-idle chỉ đòi **locked-chưa-mở**, có grace 7 ngày. Ngôn ngữ UI: "khuyến khích dùng hằng ngày", KHÔNG "phạt" — tránh phản-onboarding (§9.H).
- Vesting dài (tới ~2,74 năm) là **chủ đích** — UI khung "dòng chảy hằng ngày", không phải "chờ mãi mới xong".

---

## 4. Chờ backend / phụ thuộc chặn (cái gì chặn cái gì)

| Phụ thuộc | Chặn endpoint | Đội | Ghi chú |
|---|---|---|---|
| **Pot deploy** (nạp vốn ban đầu từ Treasury/`dist_treasury`, §6.4 Feat-Math) | 1a GetLAMP, 6a pot | LAMP-team + Long | Không có pot → không tính được D → không GetLAMP. Pilot: D co theo pot (tự điều tiết, không lỗi). |
| **Validator vault-vesting** (locked/unlocked, unlock-gate slot/NGÀY, `did_commit` gate) | 1a/1b, 2, 3a/3b, 6b | Tuân (Aiken) | Tái dùng `lock.ak`. Build được ngay, độc lập MAGIC/CARP (Feat-Math §8.2). |
| **`did_commit` per-DID + counterparty field** trong consume-event | field `anti_idle` của endpoint 2 | **MAGIC-team (BLOCKER)** | Hiện `EngageDatum.did_commit=#""` sentinel. Không có counterparty → anti-idle không phân biệt self-loop (G3) → **KHÔNG bật anti-idle production** (Feat-Math §6.1, §8.5). Tới lúc đó endpoint 2 trả `anti_idle.used_today=null`. |
| **Resolve API point-in-time** (`did_active(did,day)`, `key_authorized`) | anti-idle job + vault-DID-gate | Long | DB hiện chỉ có state-hiện-tại (Feat-Math §6.2; Catalog B1). |
| **GreenBack interface** (`greenback_settle`, backed-path §7 WP) | 4a/4b/4c GetMAGIC (backing quyết-toán) | CARP-team | PhoenixKey CHỈ khai interface, không re-spec, không chạm peg (F5/F9). GetMAGIC (fiat→CARP cầu-vào) chạy được trước; backing về Treasury treo tới khi có interface. |
| **`fee_refill_lamp`** (thu phí user bằng LAMP → pot, phản-chu-kỳ) | (nạp pot, không chặn UI trực tiếp) | LAMP + Long | Tham số + cơ-chế-quy-đổi chờ chốt (§6.5). Pilot: pot nạp bằng anti-idle + treasury là đủ. |
| **MAGIC-SDK ScheduleGen/InstantGen** | 5 Gen-entry | MAGIC-SDK | Engine đã có; cần quyết A (SuperApp gọi SDK thẳng) vs B (PhoenixKey proxy). Khuyến nghị A. |
| **`MIN_MAGIC_TX` (OPEN D)** | ngưỡng `used_today` của anti-idle | anh Aladin chốt | Chưa chốt → `min_magic_tx=null`, chưa đánh giá active. |

**Thứ tự SuperApp nên dựng UI (dựa Feat-Math §8):**
1. **Ngay (mock đủ dùng):** màn GetLAMP + Vault Dashboard + lịch vesting (1a/1b/2/6a/6b) — logic thuần vesting, không chờ MAGIC/CARP.
2. **Sau (nối khi Long ship):** ClaimUnlocked (3a/3b).
3. **Chờ MAGIC-team:** hiển thị cảnh báo `anti_idle` (field trả null tới lúc đó) + màn Gen.
4. **Chờ CARP/GreenBack + gateway fiat:** màn GetMAGIC (4a/4b/4c).

---

## 5. Mã lỗi mới đề xuất (Long thêm vào `ErrorCode.java`)

Bám quy tắc đánh số hiện có (131x = Activation, 132x = Wallet/MAGIC). Đề xuất nhánh **135x — Activation GetLAMP/Vault** để tách khỏi 131x (model cũ, sẽ deprecate):

| Đề xuất code | Tên | HTTP | Dùng ở |
|---|---|---|---|
| 1350 | `VAULT_ALREADY_EXISTS` | 409 | 1a |
| 1351 | `VAULT_NOT_FOUND` | 404 | 2, 3a |
| 1352 | `POT_UNAVAILABLE` | 503 | 1a, 6a |
| 1353 | `CLAIM_AMOUNT_EXCEEDS_UNLOCKED` | 400 | 3a |
| 1360 | `GETMAGIC_UNSUPPORTED_CURRENCY` | 400 | 4a |
| 1361 | `GETMAGIC_QUOTE_EXPIRED` | 410 | 4b |
| 1362 | `GETMAGIC_ORDER_NOT_FOUND` | 404 | 4c |
| 1363 | `GETMAGIC_PAYMENT_INIT_FAILED` | 502 | 4b |

Tái dùng mã sẵn có: `UNAUTHORIZED` (1304), `WALLET_NOT_REGISTERED` (1320), `USER_DID_NOT_FOUND` (2002), `SIGNATURE_INVALID` (1403), `CARDANO_TX_FAILED` (5101).

---

## 6. Ghi trung thực cho reviewer

- **KHÔNG endpoint nào live.** Toàn bộ là spec — SuperApp dựng UI + mock, nối API khi Long ship. Model cũ VND-Genie trong `ActivationController.java` sẽ bị thay/deprecate, đừng bám.
- **Tên path là đề xuất**, bám convention `/activation/*` + snake_case + `DataResponse`. Long có thể đổi tên khi implement — chốt với Long trước khi hard-code path vào SuperApp.
- **Ranh giới Gen (§5) chưa quyết A/B** — khuyến nghị SuperApp coi Gen là "sang màn MAGIC-SDK", không giả định PhoenixKey có endpoint Gen.
- **anti-idle field trả null** cho tới khi MAGIC-team gắn counterparty — SuperApp dựng UI cảnh báo nhưng ẩn tới lúc đó.
- **GetMAGIC backing** phụ thuộc GreenBack (CARP-team) — luồng fiat→CARP UI dựng được, phần quyết-toán về Treasury là việc backend, không lộ ra UI.
