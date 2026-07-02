# PhoenixKey Wallet API v2 — DEV-READY (Feat)

> **PHÂN LOẠI: OFFCHAIN → PR giao Long.**
> **Dependency ngoài: CARP policy-id + asset-name (đội CARP — CHƯA có giá trị, chờ xác nhận canonical).**
>
> Refine cho Issue `PhoenixKey-Database#41`. Nguồn đối chiếu code thật:
> `WalletController.java`, `WalletDtos.java`, `BlockfrostHttpClient.java`,
> `IdentityServiceImpl.java` + `AuthorizedKeyRepository.java`, `phoenix_address.rs`,
> `PhoenixKey-Math §8` (CIP-1852), và bản đính chính SuperApp
> `spec-proposals/SuperApp-Reply-wallet-api.md`.
> PhoenixKey chốt interface; Long ước lượng thời gian + build backend.

---

## Bối cảnh: Triple Token — MAGIC KHÔNG phải số dư ví

Ba token hệ PhoenixKey·MagicLamp: **LAMP · CARP · MAGIC**. Phân biệt bản chất trước khi code:

| Token | Bản chất | Nằm ở đâu | Đọc bằng |
|---|---|---|---|
| **LAMP** | Native token (có policy-id) | Trong `value` của UTxO tại địa chỉ ví | Blockfrost `/addresses/{addr}` lọc theo `policyId+assetName` |
| **CARP** | Native token (có policy-id) | Cùng ví với LAMP (trong `value` UTxO) | Cơ chế Y hệt LAMP — **thêm policy CARP** |
| **MAGIC** | **Số kế toán trong Vault** (`VaultDatum.magic_batches`) — KHÔNG native, KHÔNG policy-id | Vault accounting (MagicLampNetwork), **KHÔNG trong ví** | Đọc từ vault, **KHÔNG từ `getBalance`** |

> 🔴 **Chốt cứng:** `balanceMagic` / `magicAccrued` trong `BalanceResponse` hiện đã deprecated.
> App KHÔNG lấy MAGIC từ endpoint số dư ví. MAGIC là dòng riêng, nguồn Vault (§3 field `magic`).

---

## Mô hình 2 ví

Một DID có thể có **cả hai** ví, phân biệt bằng field `kind`.

| | **Phoenix custody** (`kind:"phoenix"`) | **Standard** (`kind:"standard"`) |
|---|---|---|
| Bản chất | Địa chỉ **script enterprise** theo danh tính | Ví **CIP-1852 HD** từ Master_KEK |
| Dẫn từ | `did + anchor_nft_policy` (`phoenix_address.rs`, validator `did_payment`) | `m/1852'/1815'/account'/role/index` (Math §8.1) |
| Địa chỉ | **1 địa chỉ cố định/đời-DID**, bất biến, sống sót rotate/recovery | `fixed` = account 0 (cố định) + `active` = account N (xoay được) |
| Stake | **KHÔNG** (enterprise, không stake key) | **CÓ** — `stake_test1…` / `stake1…` (role 2, Math §8.1) |
| Giữ gì | LAMP, CARP (native, công khai) | LAMP, CARP, ADA |
| Đối tượng | Enterprise / custody bất biến theo DID | Ví Cardano thường (tương thích Eternl/Lace/Yoroi) |
| Nơi derive | Client (rust_core `derive_phoenix_address`) | Client (Master_KEK **KHÔNG bao giờ lên server**) |
| Trạng thái code | ✅ `phoenix_address.rs` (test đối chiếu aiken CLI xanh) | 🟡 Math §8 đủ, **CHƯA có API** — mục §2 |

> **Nguyên tắc nền:** Master_KEK sinh cả DID và ví Standard, không rời thiết bị.
> ⇒ Server **không tự derive** địa chỉ Standard được (không có Master_KEK).
> Client derive → **register** địa chỉ lên backend → backend lưu + query số dư (§2).

---

## §1 — `balanceCarp`: thêm CARP vào BlockfrostHttpClient

CARP đọc **cùng cơ chế đang chạy cho LAMP** trong `BlockfrostHttpClient.getAddressUtxos()`
(`BlockfrostHttpClient.java:88-110`): lọc `amount[]` theo `unit == policyId + assetNameHex`.

### §1.1 Config keys mới (`application.yml`)

```yaml
phoenixkey:
  cardano:
    # đã có:
    lamp-policy-id:      "<28-byte hex>"
    lamp-asset-name-hex: "4c414d50"          # "LAMP"
    # THÊM MỚI cho CARP:
    carp-policy-id:      ""                    # ⚠️ TRỐNG — chờ đội CARP (external dep)
    carp-asset-name-hex: ""                    # ⚠️ TRỐNG — chờ đội CARP
```

> **⚠️ External dependency — đội CARP:** `carp-policy-id` + `carp-asset-name-hex`
> **CHƯA có giá trị canonical.** CARP là sản phẩm repo riêng, PhoenixKey chưa track policy.
> Long build code + config sẵn key rỗng; **khi rỗng → `balanceCarp = 0`** (không throw, không query nhầm).
> Chỉ khi đội CARP xác nhận policy-id/asset-name mới điền vào → CARP xuất hiện tự động.

### §1.2 Đổi `BlockfrostHttpClient`

- Thêm 2 field `@Value` `carpPolicyId`, `carpAssetNameHex` (song song lamp/magic hiện có, dòng 31-41).
- `AddressUtxos` record thêm `long carpQuantity`:
  ```java
  public record AddressUtxos(long lovelace, long lampQuantity,
                             long carpQuantity, long magicQuantity) {}
  ```
  (giữ `magicQuantity` để không vỡ caller cũ; xem §4 để dọn sau).
- Trong vòng lặp `getAddressUtxos` (dòng 95-104), thêm nhánh:
  ```java
  String carpUnit = carpPolicyId + carpAssetNameHex;
  ...
  else if (!carpPolicyId.isEmpty() && unit.equals(carpUnit)) carp += qty;
  ```
  **Guard `!carpPolicyId.isEmpty()`** — tránh khi config rỗng thì `carpUnit` = "" khớp nhầm.

### §1.3 Đưa `balanceCarp` ra API

- `BalanceResponse` thêm `long balanceCarp` (chi tiết field §4.1).
- `BalanceService.getBalance()` map `carpQuantity → balanceCarp`.

---

## §2 — Standard wallet: register + query (client derive, server lưu)

### §2.1 Vì sao KHÔNG có endpoint "derive"

Server **không** có Master_KEK ⇒ không derive được CIP-1852 địa chỉ Standard.
Đúng thiết kế: **client derive, server đăng ký + tra số dư** — cùng mô hình `/wallet/register`
(`WalletController.java:22-30`) đang dùng cho Phoenix.

### §2.2 Luồng register (client → backend)

Client (rust_core, có Master_KEK) derive theo Math §8.1:
- `fixed`  = `address_base(account=0, index=0)`  — địa chỉ cố định (account 0)
- `active` = `address_base(account=N, index=0)`  — account đang dùng (tuỳ chọn)
- `stake`  = từ `staking_key(account)` (role 2) — base address đã kèm stake credential

Rồi POST đăng ký (đã ký AuthenticatedUser — server lấy `did` từ auth, KHÔNG nhận did trong body):

```
POST /wallet/standard/register
Authorization: <PhoenixKey signed session>
Body:
{
  "fixedAddress":  "addr_test1q...",   // account 0, base (payment+stake)
  "activeAddress": "addr_test1q...",   // account N, base — optional
  "stakeAddress":  "stake_test1u..."   // stake credential — optional nhưng nên có
}
```
- Validate mỗi địa chỉ theo regex hiện có (`WalletDtos.WalletRegisterRequest`):
  `^addr_(test1|1)[a-z0-9]+$`; stake: `^stake_(test1|1)[a-z0-9]+$`.
- Lưu vào bảng địa chỉ Standard theo `(userDid, kind='standard', slot=fixed|active, stakeAddress)`.
  Cho phép cập nhật `activeAddress` khi client xoay account (idempotent theo `fixed`).
- Trả `code 1000`.

> **Bảng lưu (gợi ý migration `V13__wallet_standard_addresses.sql`):**
> `wallet_standard_address(user_did, fixed_address, active_address NULLABLE,
> stake_address NULLABLE, updated_at)` — unique theo `user_did` (1 Standard-set/DID);
> hoặc `(user_did, fixed_address)` nếu sau này cho nhiều account. Long chốt theo schema hiện có.

### §2.3 Query endpoint

```
GET /wallet/standard/{did}
→ 200
{
  "addresses": {
    "fixed":  "addr_test1q...",
    "active": "addr_test1q...",   // vắng nếu chưa register
    "stake":  "stake_test1u..."   // vắng nếu chưa register
  },
  "balances": {
    "lovelace": 0,
    "lamp":     0,
    "carp":     0                  // 0 tới khi §1 config CARP
  }
}
```
- Public như `/wallet/{did}/balance` hiện tại (địa chỉ Cardano vốn công khai).
- Số dư = tổng UTxO tại `fixed` **+** `active` (nếu có), qua `getAddressUtxos` mỗi địa chỉ rồi cộng.
- Chưa register Standard → **404** (phân biệt rõ với "đã register, số dư 0").

---

## §3 — Endpoint gộp: `GET /wallet/{did}/all`

Một lần gọi trả **cả 2 ví + MAGIC tách riêng**. Đây là interface app nên dùng.

```
GET /wallet/{did}/all
→ 200
{
  "wallets": [
    {
      "kind": "phoenix",
      "addresses": { "fixed": "addr_test1w..." },          // 1 địa chỉ, KHÔNG active/stake
      "balances":  { "lovelace": 0, "lamp": 0, "carp": 0 }  // KHÔNG có magic
    },
    {
      "kind": "standard",
      "addresses": { "fixed": "addr...", "active": "addr...", "stake": "stake..." },
      "balances":  { "lovelace": 0, "lamp": 0, "carp": 0 }
    }
  ],
  "magic": {
    "source": "vault",     // 🔴 MAGIC từ Vault, KHÔNG phải balance ví
    "available": 0,        // nanogic (10⁹ = 1 MAGIC) — đọc từ vault accounting
    "accrued": 0
  }
}
```

Quy tắc:
- **`wallets[]` chỉ chứa ví đã có địa chỉ.** Phoenix luôn có (derive từ DID). Standard chỉ xuất hiện khi đã register (§2). Không register Standard → mảng chỉ có `phoenix`.
- `phoenix.balances` **KHÔNG** có `magic`. MAGIC nằm ở block `magic` top-level, `source:"vault"`.
- `magic.available/accrued`: đọc từ vault accounting (MagicLampNetwork SDK). Nếu Phase này chưa nối vault → trả `0` + `source:"vault"` (app hiển thị "—"), **không** trả từ `getBalance`.
- Số dư chưa có → `0`; app tự render "—". Interface chốt 1 lần.

> **Lưu ý kiến trúc (shielded custody, PhoenixKey-Specs PR#3):** ví Phoenix sắp có chế độ ẩn số dư
> — Phase sau đọc qua viewing-key/pool thay vì đọc thẳng địa chỉ. Long **tách lớp đọc số dư Phoenix**
> (1 interface `PhoenixBalanceReader`) để đổi nguồn sau không phải sửa endpoint.

---

## §4 — Deprecate MAGIC trong ví

### §4.1 `BalanceResponse` — flag + bỏ MAGIC fields

`WalletDtos.BalanceResponse` hiện có `balanceMagic`, `magicAccrued`, `magicRatePerSlot`,
`lastAccrualSlot`, `currentSlot`. Đổi:

```java
public record BalanceResponse(
        String address,
        long balanceLovelace,
        long balanceLamp,
        long balanceCarp,          // THÊM (§1.3)

        // ── DEPRECATED — MAGIC KHÔNG phải balance ví, đọc từ vault (§3.magic) ──
        @Deprecated long balanceMagic,      // luôn 0; sẽ xoá bản sau
        @Deprecated long magicAccrued,      // luôn 0
        @Deprecated String magicRatePerSlot,
        @Deprecated long lastAccrualSlot,
        @Deprecated long currentSlot
) {}
```
- **Giai đoạn 1 (bản này):** giữ field để không vỡ app, **ép giá trị = 0/""**, đánh `@Deprecated`
  + swagger `deprecated=true` mô tả "dùng `/wallet/{did}/all` → `magic`".
- **Giai đoạn 2 (bản sau, sau khi SuperApp chuyển):** xoá hẳn 5 field MAGIC.

### §4.2 `/wallet/magic/claim` — deprecate

`WalletController.claimMagic()` (`WalletController.java:41-46`) + `BalanceService.claimMagic()`:
- MAGIC là account-in-vault, **không mint gửi vào ví** (mô hình đã đảo lại). Endpoint này lỗi thời.
- **Giai đoạn 1:** đánh `@Deprecated`, swagger `deprecated=true`, trả `410 Gone` hoặc
  `code` lỗi rõ ràng ("MAGIC claim đã ngừng — MAGIC là số vault, không claim vào ví").
  KHÔNG mint token.
- **Giai đoạn 2:** xoá endpoint + service method.

### §4.3 Migration note cho SuperApp

> 1. **Ngừng đọc** `balanceMagic`/`magicAccrued` từ `/wallet/{did}/balance`.
> 2. Lấy MAGIC từ `/wallet/{did}/all` → `magic.available` / `magic.accrued` (`source:"vault"`).
> 3. **Ngừng gọi** `/wallet/magic/claim` (sẽ trả 410).
> 4. LAMP+CARP hiển thị cùng ví; MAGIC hiển thị dòng riêng, nhãn nguồn "Vault".

---

## §5 — BUG offchain (sửa cùng đợt): `/identity/{did}/pubkey` 500 cho user đã recover

**Hiện tượng:** user đã qua Mode B Recovery → `GET /identity/{did}/pubkey` trả **500**.

**Nguyên nhân gốc:** recovery chèn HW_Key owner **mới** (active) nhưng **key owner cũ có thể còn `status='active'`**
⇒ DID có **>1 owner-key active**. Endpoint `getPubkey` (`IdentityServiceImpl.java:195-199`) gọi
`findOwnerByUserDid` (`AuthorizedKeyRepository.java:135-139`) trả `Optional<AuthorizedKey>`.

- Query hiện đã có `ORDER BY created_at DESC, id DESC LIMIT 1` → *về nguyên tắc* không throw non-unique.
- Nhưng nếu deployment thực còn dùng bản **trước khi thêm `LIMIT 1`** (hoặc `getOwnerKey` khác gọi
  `findBy...` không LIMIT), Spring Data ném `IncorrectResultSizeDataAccessException` → HTTP 500.

**Việc Long (offchain, cùng PR này hoặc PR kề):**
1. **Grep toàn bộ** truy vấn owner-key: `grep -rn "key_role = 'owner'" src/` — đảm bảo MỌI query
   đọc-đơn owner-key đều có `AND status='active' … ORDER BY created_at DESC, id DESC LIMIT 1`
   và trả `Optional`, không có biến thể trả single-throw.
2. **Đúng hơn — sửa recovery để revoke owner cũ:** trong `RecoverDeviceService`, khi chèn HW_Key owner mới,
   set các owner-key active cũ của DID về `status='revoked'` **trong cùng transaction**
   (đã có `@Query UPDATE … SET status='revoked'` dòng 93/103 — dùng lại).
   ⇒ Bất biến: **tối đa 1 owner-key active/DID** → pubkey luôn xác định, hết 500.
3. Bọc `getPubkey`: `.orElseThrow(() -> new NotFoundException(...))` → **404** khi thiếu, không để 500.
4. **Test** (§Test plan) tái hiện: register → recover → `GET pubkey` phải 200 trả HW_Key mới.

> Cả 3 thay đổi 100% offchain (Java + SQL). Không đụng validator/onchain.

---

## Test plan (evidence bắt buộc trước khi báo PASS)

Chạy trên preprod (Blockfrost preprod key). Ghi output thật vào PR.

| # | Bước | Assert |
|---|---|---|
| T1 | Client derive Standard (`address_base(0,0)`, `(N,0)`, stake) → `POST /wallet/standard/register` | `code 1000`; bản ghi trong `wallet_standard_address` |
| T2 | `GET /wallet/standard/{did}` sau register | 200; `addresses.fixed/active/stake` đúng địa chỉ đã register |
| T3 | `GET /wallet/standard/{did}` DID **chưa** register | **404** (phân biệt với số dư 0) |
| T4 | `GET /wallet/{did}/all` khi DID có cả Phoenix + Standard | `wallets[]` có 2 phần tử `kind` đúng; `phoenix.addresses` chỉ `fixed`, KHÔNG stake |
| T5 | `GET /wallet/{did}/all` — kiểm MAGIC | có block `magic{source:"vault",...}`; **KHÔNG** field `magic` trong `wallets[].balances` |
| T6 | CARP config **rỗng** → `getAddressUtxos` | `balanceCarp = 0`, không throw, không query nhầm unit |
| T7 | CARP config **đã điền** policy+name (dùng địa chỉ testnet có CARP) → `/wallet/{did}/all` | `carp` > 0 **đúng 1 lần** ở đúng ví giữ CARP (không nhân đôi, không lẫn LAMP) |
| T8 | `/wallet/{did}/balance` (deprecated) | `balanceMagic=0`, `magicAccrued=0`; swagger đánh `deprecated` |
| T9 | `POST /wallet/magic/claim` | **410 Gone** (hoặc code lỗi rõ); KHÔNG mint token |
| T10 | Register → Recover (Mode B) → `GET /identity/{did}/pubkey` | **200**, trả HW_Key owner **mới**; DB chỉ còn **1** owner-key active |

> **Verify behavior, không chỉ compile:** T7 phải chạy `curl` thật với địa chỉ preprod có CARP
> sau khi điền policy — 1 lệnh curl xác nhận CARP hiện đúng 1 lần. T10 phải tái hiện recovery thật.

---

## Tóm việc + giao

| Mục | Trạng thái | Giao | Chặn bởi |
|---|---|---|---|
| §1 `balanceCarp` (config + client + response) | 🔴 thêm | Long | **CARP policy-id/asset-name (đội CARP)** |
| §2 API ví Standard (register + query) | 🔴 mới | Long | — |
| §3 Endpoint gộp `/wallet/{did}/all` | 🔴 mới | Long | vault-read cho `magic` (Phase — tạm trả 0) |
| §4 Deprecate MAGIC (`BalanceResponse` + `/magic/claim`) | 🟡 sửa | Long | phối hợp SuperApp migrate |
| §5 Bug `/identity/{did}/pubkey` 500 (recover) | 🔴 sửa | Long | — (offchain thuần) |

**Interface do PhoenixKey chốt. Long ước lượng thời gian + build. Không đụng onchain/validator.**
