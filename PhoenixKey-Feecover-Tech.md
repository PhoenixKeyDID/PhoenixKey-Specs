# PhoenixKey — Feecover (TECH) — đặc-tả kỹ-thuật cho kỹ sư

> Tài liệu KỸ-THUẬT: kiến trúc, datum/beacon schema, luồng tx, API endpoint, ranh-giới giao-việc, phụ-thuộc-CHẶN, test/evidence. Bám sát code THẬT của ConsumeMAGIC/Paymaster (không suy diễn từ trí nhớ).
>
> **Nguồn Feat/Math (đã chốt design):** `spec-proposals/PhoenixKey-Feecover-Feat-Math.md`.
> **Nguồn engine tái dùng:** `MAGIC/ConsumeMAGIC/{CONTRACT.md, onchain/lib/magiclamp/consume/types.ak, onchain/validators/consume.ak, onchain/lib/magiclamp/consume/pricing.ak}`; `MAGIC/Paymaster/onchain/lib/magiclamp/paymaster/types.ak`.
> **Nguồn resolve API:** `spec-proposals/PhoenixKey-PointInTime-Resolve-API-Feat-Math.md §3`; `spec-proposals/PhoenixKey-DID-Registry-Resolver-Feat.md`.
> **Nguồn canonical đơn-vị/peg:** `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` (§4 MAGIC, §5 CARP). `1 nanogic = 1 byte·ngày`; `1 MAGIC = 1e9 nanogic = 1 GB·ngày`; `1 CARP = 1 MAGIC`.

## Trạng thái: TECH SPEC PROPOSAL · chờ anh chốt §9 (Feat-Math) + 2 BLOCKER (§7)

---

## 0. Kết luận cho kỹ sư (đọc trước)

- Feecover = **overlay chính-sách+định-tuyến ĐẶT LÊN TRÊN `consume.ak`**. KHÔNG dựng lại engine tiêu-MAGIC. `consume.ak` + C-CM-1..5 GIỮ NGUYÊN, KHÔNG sửa 1 dòng.
- Feecover THÊM 4 on-chain object mới (`ServiceFeeSchedule` beacon, `FeecoverGate` beacon, `FeecoverAccrual` state, `FeecoverEpochSettle` state) + 1 validator mới (`feecover.ak`) + tx-builder offchain + endpoint backend.
- **Đơn vị:** tất cả `fee_magic` = **nanogic** (Int), trùng đơn vị `base_price` của `PriceParam`. CARP trả = `fee_magic` (neo 1:1, `1 thread CARP = 1 nanogic MAGIC`).
- **2 phụ-thuộc CHẶN (BLOCKER, §7):** (1) MAGIC-team enforce **NỘI DUNG** `did_commit` per-DID (field đã CÓ trong `EngageDatum` nhưng validator chưa ràng nội dung); (2) CARP-team chốt **`policy_id` + `asset_name` + decimals** của CARP trên preprod/mainnet. Thiếu (2) → không định tuyến được nhánh CARP. Thiếu (1) → attribution chỉ ở mức `owner`-key, không per-DID.
- **ĐÍNH CHÍNH nguồn Feat-Math (§1.2, §8.1):** Feat-Math viết "EngageDatum CHƯA có `did_commit`, cần MAGIC-team THÊM trường". **SAI theo code THẬT.** `EngageDatum` đã có `did_commit : ByteArray` là field thứ 4 (append-only, `types.ak:46-51`), MVP = `#""`, validator ĐÃ ép immutable (`consume.ak:409` `od.did_commit == did_commit`). Cái CÒN THIẾU là **enforce NỘI DUNG** (bind về DID sinh trắc + verify signer khớp DID), KHÔNG phải thêm field. §7.1 dưới ghi lại đúng blocker.

---

## 1. Kiến trúc — Feecover overlay ConsumeMAGIC

```
┌──────────────────────────────────────────────────────────────────────┐
│  L3  Giao thức ỔN-ĐỊNH-GIÁ CARP (NGOÀI PHẠM VI Feecover)             │
│      GreenPeg/RedPeg · backing · oracle. Feecover chỉ có 2 interface: │
│      deposit(CARP) ↑   ·   request_topup(deficit) ↓                    │
├──────────────────────────────────────────────────────────────────────┤
│  L2  FEECOVER (MỚI — validator feecover.ak + 4 object)                 │
│                                                                        │
│   reference inputs (CIP-31, KHÔNG tiêu):                              │
│     • ServiceFeeSchedule beacon  → service_id → fee_magic  (§3.1)      │
│     • FeecoverGate beacon        → cổng DID (resolve API)   (§3.2)     │
│     • DID credential (registry NFT)                                    │
│                                                                        │
│   spend (per-tx):                                                      │
│     • FeecoverAccrual state UTxO  → accrue CARP theo tầng   (§3.3)     │
│                                                                        │
│   ràng buộc chéo với L1 CÙNG TX:                                       │
│     magic_consumed(consume.ak) == fee_magic(service_id)                │
│     carp_paid == fee_magic          (neo 1:1)                          │
│                                                                        │
│   epoch boundary (spend):                                              │
│     • FeecoverEpochSettle state    → App→Platform→Treasury  (§3.4,§5)  │
├──────────────────────────────────────────────────────────────────────┤
│  L1  CONSUMEMAGIC (TÁI DÙNG NGUYÊN TRẠNG — KHÔNG SỬA)                 │
│   consume.ak · EngageDatum{owner,consumed_count,last_epoch,did_commit} │
│   PriceParam beacon · thread NFT · C-CM-1..5                          │
├──────────────────────────────────────────────────────────────────────┤
│  L0  Generator Vault (giảm số dư MAGIC qua BurnBatch — nơi DUY NHẤT)   │
│      · Vault provider (giữ CARP accrue) · Phoenix Treasury (nhận CARP) │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.1 Nguyên tắc kiến trúc (bám Feat-Math §2)

- **P-LAYER-1 — Feecover KHÔNG là input của `consume.ak`.** Không thêm bất biến vào `consume.ak`. `feecover.ak` đọc `ServiceFeeSchedule` + `FeecoverGate` + DID credential dạng **reference input**, và spend `FeecoverAccrual` của chính nó, TRONG CÙNG tx với consume. Hai validator (`consume.ak`, `feecover.ak`) chạy song song trong 1 tx, đọc CÙNG `PriceParam` beacon.
- **P-LAYER-2 — đọc THẬT, ép SỚM.** `feecover.ak` KHÔNG tin redeemer amount. Nó đọc lại `magic_consumed` THẬT (Σ burns) từ cùng cơ chế `consume.ak` dùng (`read_vault_burns` qua `tx.redeemers`), và ép `magic_consumed == fee_magic(service_id)` + `carp_paid == fee_magic`.
- **P-LAYER-3 — định tuyến, không định giá phức tạp on-chain.** Feecover chỉ CHỌN `fee_magic` từ bảng + gom CARP vào Accrual đúng tầng/epoch. Oracle/quy-đổi-giá-trị ở L3.

### 1.2 Vì sao 2 validator co-spend, không gộp vào consume.ak

First-principles (bám CLAUDE.md build-mode trục b/c): `consume.ak` là engine đã audit (C-CM-1..5, gồm cả vá double-satisfaction AGGREGATE + validity-range gaming). Nhét logic phí PhoenixKey vào đó = (1) sửa file của MAGIC-team → phá ranh giới sở hữu, (2) tăng bề mặt tấn công của engine dùng-chung. Feecover là validator RIÊNG, co-spend cùng tx — MAGIC-team giữ nguyên `consume.ak`, PhoenixKey sở hữu `feecover.ak`.

---

## 2. Bảng phân định: ĐÃ-có (ConsumeMAGIC) vs THÊM-mới (Feecover) vs BLOCKER

| # | Thành phần | Trạng thái | File / vị trí |
|---|---|---|---|
| 1 | Engine tiêu MAGIC = giảm số dư vault, thread NFT bảo toàn | ✅ ĐÃ-có, tái dùng nguyên trạng | `consume.ak`, C-CM-1..5 |
| 2 | Giá có thẩm quyền từ beacon, không tin client | ✅ ĐÃ-có | `PriceParam` + `read_price_param` (`consume.ak:130`) |
| 3 | Value non-MAGIC bảo toàn byte-perfect @engage | ✅ ĐÃ-có (C-CM-1) | `engage_value_preserved` (`consume.ak:317`) |
| 4 | Chống double-satisfaction + replay + stale price | ✅ ĐÃ-có (C-CM-2/3/4/5) | `consume.ak` aggregate invariants |
| 5 | `EngageDatum.did_commit` field (append-only) | ✅ ĐÃ-có (immutable enforced) | `types.ak:46`, `consume.ak:409` |
| 6 | Mẫu paymaster đọc `magic_consumed` từ vault redeemer | ✅ ĐÃ-có (tham chiếu) | `Paymaster/.../types.ak`, `read_vault_burns` |
| 7 | `ServiceFeeSchedule` beacon (service_id → fee_magic) | 🟢 THÊM-mới | `feecover/onchain/lib/.../types.ak` (mới) |
| 8 | `FeecoverGate` beacon (cổng DID point-in-time) | 🟢 THÊM-mới | như trên |
| 9 | `FeecoverAccrual` state (accrue CARP per-tx) | 🟢 THÊM-mới | như trên |
| 10 | `FeecoverEpochSettle` state (settle cuối epoch) | 🟢 THÊM-mới | như trên |
| 11 | `feecover.ak` validator (ép `carp_paid==fee_magic`, gate, accrue) | 🟢 THÊM-mới | `feecover/onchain/validators/feecover.ak` (mới) |
| 12 | tx-builder offchain (co-spend consume + feecover) | 🟢 THÊM-mới | `feecover/offchain/` (mới) |
| 13 | Backend endpoint quote/gate/settle | 🟢 THÊM-mới | PhoenixKey backend (Long) |
| 14 | **Enforce NỘI DUNG `did_commit`** per-DID + signer bind | 🔴 BLOCKER | MAGIC-team (`consume.ak`/vault) — §7.1 |
| 15 | **CARP `policy_id`/`asset_name`/decimals** preprod+mainnet | 🔴 BLOCKER | CARP-team — §7.2 |
| 16 | Resolve API point-in-time (`/active`, `/key-authorized`) | 🟡 phụ-thuộc (spec có, Long implement) | §7.3 |
| 17 | Ổn-định-giá CARP (GreenPeg/RedPeg/oracle/buyback) | 🚫 NGOÀI PHẠM VI | CARP L3 |

**Tỷ lệ:** 6/17 tái dùng nguyên trạng, 7/17 build mới (5 on-chain object/validator + builder + backend), 2 BLOCKER, 1 phụ-thuộc mềm, 1 ngoài phạm vi.

---

## 3. Datum / Beacon schema (Aiken — mới)

> Constr index = THỨ TỰ KHAI BÁO field (Plutus Data). Phải khớp byte-perfect với offchain codec. Field mới ở CUỐI (append-only), theo đúng kỷ luật `types.ak` của ConsumeMAGIC/Paymaster.
> Đơn vị số: mọi field tiền tệ = **nanogic** (Int); CARP amount = **thread** (Int), neo `1 thread = 1 nanogic`.

### 3.1 `ServiceFeeSchedule` — beacon (reference input, NFT-auth, gov-gated)

```aiken
/// Beacon bảng phí dịch vụ PhoenixKey. Cùng khuôn PriceParam:
/// reference input CIP-31, NFT one-shot xác thực, versioned, gov-gated.
pub type ServiceFeeEntry {
  service_id : ByteArray,   // "did.anchor", "did.rotate", "vc.verify", "custody.op"
  op_type    : Int,         // ÁNH XẠ sang OpPrice.op_type của PriceParam (đồng bộ §7.4)
  fee_magic  : Int,         // nanogic — phí danh nghĩa; PHẢI == price(op_type) từ PriceParam
  enabled    : Bool,
}

pub type ServiceFeeSchedule {
  entries       : List<ServiceFeeEntry>,
  version_epoch : Int,        // epoch phát hành (freshness giống PriceParam.epoch)
  gov_ref       : ByteArray,  // ScriptHash Governance đã duyệt (status==Executed)
}
```

Validator ép (khi đọc beacon): NFT one-shot qty==1 (mẫu `read_price_param` `consume.ak:138`); `0 ≤ current_epoch − version_epoch ≤ max_schedule_stale` (mẫu C-CM-5); entry có `enabled==true`.

> **Ghi chú `op_type` bridge:** Feat-Math §8.3 nêu phải đồng bộ `PriceParam.base_price[op]` ↔ `ServiceFeeSchedule.fee_magic[service_id]`. Cách nối cụ thể: mỗi `ServiceFeeEntry` mang `op_type` trỏ đúng dòng `OpPrice` trong `PriceParam`; validator đọc `required = price(op_type)` từ `PriceParam` (như `consume.ak` làm) và ép `fee_magic == required`. Xem §7.4 (quyết định D6 — Feat-Math §9).

### 3.2 `FeecoverGate` — beacon (reference input, cổng DID)

```aiken
pub type FeecoverGate {
  did_registry_policy : ByteArray,  // PolicyId NFT của PhoenixKey DID Registry
  resolve_ref         : ByteArray,  // neo điểm resolve API point-in-time (off-chain check)
  epoch_valid_until   : Int,        // freshness (mẫu max_price_stale)
}
```

- On-chain ép: DID credential đọc qua **reference input** (KHÔNG tiêu, KHÔNG mint — FEECOVER-GATE-3), có NFT `policy == did_registry_policy`; `current_epoch ≤ epoch_valid_until` (FEECOVER-GATE-2).
- Off-chain (tx-builder + backend) ép point-in-time: `did_active`/`key_authorized` qua resolve API (§4.2, §7.3). On-chain KHÔNG gọi HTTP; nó chỉ verify credential-presence + (khi blocker §7.1 xong) `did_commit == blake2b256(did‖salt)` khớp DID mà builder đã resolve.

### 3.3 `FeecoverAccrual` — state UTxO (accrue CARP per-tx trong epoch)

```aiken
pub type Tier { App }  // hoặc Platform — xem D4 (§7.4)
pub type Tier { Platform }

pub type FeecoverAccrual {
  tier         : Tier,                    // App | Platform
  epoch        : Int,                     // epoch đang gom
  carp_accrued : Int,                     // Σ CARP (thread) gom trong epoch
  breakdown    : List<(ByteArray, Int)>,  // (service_id, Σ) — audit + attribution
  provider_id  : ByteArray,               // định danh provider (App) sở hữu accrual này
}
```

- Neo bằng `accrual_nft` (thread NFT one-shot, giống engage NFT) — 1 UTxO / (provider, tier, epoch). `carp_accrued` CHỈ tăng trong epoch; reset sau settle (§5).
- CARP thật GIỮ trong value của UTxO này (native token, qty == `carp_accrued`). Datum `carp_accrued` PHẢI khớp value CARP thật (ép on-chain: `quantity_of(out.value, carp_policy, carp_name) == out.datum.carp_accrued`).

### 3.4 `FeecoverEpochSettle` — state UTxO (settle cuối epoch → Treasury + bàn-giao L3)

```aiken
pub type FeecoverEpochSettle {
  last_settled_epoch : Int,
  carp_to_treasury   : Int,                   // Σ CARP đã về Phoenix Treasury (append-only)
  deposited_l3       : List<(ByteArray, Int)>, // (asset, Σ) đã nộp L3 (append-only)
  topup_requested    : List<(ByteArray, Int)>, // (asset, Σ) đã yêu cầu bù (append-only)
  stabilizer_ref     : ByteArray,             // ScriptHash giao thức ổn-định CARP (L3, đổi qua Gov)
}
```

---

## 4. Luồng transaction

### 4.1 Tx CONSUME + FEECOVER (per-transaction — user tiêu dịch vụ)

**1 tx co-spend 3 validator (vault BurnBatch + consume + feecover):**

```
INPUTS (spend):
  [V] vault UTxO        redeemer BurnBatch{burns}      → giảm số dư MAGIC  (L0, MAGIC-team validator)
  [E] engage UTxO       redeemer Consume{op_type,op_count,price_ref,vault_ref}  (L1, consume.ak)
  [A] FeecoverAccrual   redeemer Accrue{service_id,carp_amount,gate_ref,schedule_ref}  (L2, feecover.ak)
  [C] user CARP UTxO    (input thường mang CARP để trả phí — KHÔNG phải từ vault)

REFERENCE INPUTS (CIP-31, không tiêu):
  [P] PriceParam beacon        (đọc bởi consume.ak VÀ feecover.ak — CÙNG price_ref)
  [S] ServiceFeeSchedule beacon
  [G] FeecoverGate beacon
  [D] DID credential (registry NFT)

OUTPUTS:
  [E'] engage UTxO   datum.consumed_count += op_count, last_epoch=current, did_commit immutable, thread NFT giữ
  [A'] FeecoverAccrual  datum.carp_accrued += carp_paid, breakdown[service_id]+=carp_paid; value CARP += carp_paid
  [V'] vault continuing (do vault validator quản)
  (change về user)
```

**feecover.ak ÉP (bất biến FEECOVER-*, bám Feat-Math §6.4):**

| ID | Bất biến on-chain | Cơ chế |
|---|---|---|
| FC-PAY-1 | `magic_consumed == fee_magic(service_id)` | đọc Σ burns THẬT qua `read_vault_burns` (mẫu `consume.ak:148`); đọc `fee_magic` từ `ServiceFeeSchedule` entry khớp `service_id` |
| FC-PAY-2 | `carp_paid == fee_magic` (neo 1:1) | `carp_paid = Δ CARP value giữa [A'] và [A]`; ép `== fee_magic` |
| FC-PAY-3 | CARP-fee đến từ input NGOÀI vault-account-script | verify không rút CARP từ `[V]` (C-CM-1 đã ép value non-MAGIC @vault bảo toàn; feecover ép thêm nguồn CARP == user input) |
| FC-ACC-1 | `[A'].carp_accrued == [A].carp_accrued + carp_paid`; `[A'].epoch == current_epoch` (nếu cùng epoch) | accrue đúng tầng+epoch, chỉ tăng |
| FC-ACC-2 | `quantity_of([A'].value, carp_policy, carp_name) == [A'].carp_accrued` | datum khớp value CARP thật (chống datum-lệch-value) |
| FC-GATE-1 | DID credential present (ref input, `policy==did_registry_policy`) | đọc `[D]`, KHÔNG tiêu/mint (FEECOVER-GATE-3) |
| FC-GATE-2 | `current_epoch ≤ FeecoverGate.epoch_valid_until` | freshness |
| FC-GATE-3 | (khi §7.1 xong) `EngageDatum.did_commit == blake2b256(did‖salt)` khớp DID resolved | bind per-DID |
| FC-SCHED-1 | `ServiceFeeSchedule` freshness + NFT-auth + `entry.enabled` | mẫu C-CM-5 + `read_price_param` |
| FC-SCHED-2 | `fee_magic == price(entry.op_type)` từ `PriceParam` (chống lệch 2 bảng) | đọc cùng `price_ref` như consume.ak |

> **Điểm nối load-bearing:** FC-PAY-1 + FC-SCHED-2 cùng đọc `PriceParam` qua `price_ref` mà `consume.ak` dùng. Vì `consume.ak` ĐÃ ép `total_burned == total_required` (C-CM-2) với `required` lấy từ CÙNG beacon, và feecover ép `fee_magic == price(op_type)` + `carp_paid == fee_magic`, chuỗi ràng đóng kín: `Σburns == required == fee_magic == carp_paid`. Không có kẽ để khai service rẻ mà tiêu nhiều (Feat-Math Phản biện E).

### 4.2 Off-chain gate (tx-builder + backend, TRƯỚC khi submit tx)

Builder KHÔNG dựng tx nếu gate fail (fail-closed):

```
1. resolve DID của account:
   GET /identity/{did}/active?epoch={current_epoch}
       → data.active == true, never_existed == false   (nếu không → REJECT)
2. nếu tx ký bằng khoá:
   GET /identity/{did}/key-authorized?key={signer_pkh}&epoch={current_epoch}
       → data.authorized == true                        (nếu không → REJECT)
3. 503 từ resolve API (indexer lag) → REJECT, KHÔNG submit (fail-closed, Contract §3)
4. tính did_commit = blake2b256(did ‖ salt), salt = enrolled_slot/epoch (khi §7.1 xong)
5. dựng tx §4.1, đính [D]=DID credential ref input, submit
```

`query_epoch = current_epoch` của tx (point-in-time semantics — Resolve-API §3, §5).

### 4.3 Tx SETTLE (epoch boundary — App→Platform→Treasury)

```
INPUTS (spend):
  [Acc*] các FeecoverAccrual(Platform, closed_epoch)   redeemer SettleEpoch{closed_epoch}
  [Set]  FeecoverEpochSettle                            redeemer SettleEpoch{closed_epoch}

OUTPUTS:
  [Tre]  Phoenix Treasury UTxO  += Σ carp_accrued (CARP thật chuyển tới địa chỉ hiến định)
  [Set'] FeecoverEpochSettle datum:
           last_settled_epoch = closed_epoch
           carp_to_treasury  += total_carp
  [Acc'] FeecoverAccrual reset carp_accrued=0 cho epoch mới (thread NFT giữ)
```

**feecover.ak ÉP (settle branch):**

| ID | Bất biến | Ghi chú |
|---|---|---|
| FC-SET-1 | `closed_epoch == last_settled_epoch + 1` (tuần tự, không nhảy/bỏ) | monotonic, chống double-settle |
| FC-SET-2 | `total_carp = Σ [Acc*].carp_accrued`; **CHỈ CARP** chuyển về Treasury | KHÔNG LAMP/MAGIC (Feat-Math §6.2) |
| FC-SET-3 | `[Tre]` nhận đúng `total_carp` CARP tới địa chỉ Treasury hiến định | verify address == treasury_hardcoded |
| FC-SET-4 | `carp_to_treasury` / `deposited_l3` / `topup_requested` chỉ tăng (append-only) | audit |

### 4.4 Interface bàn-giao L3 (deposit / request_topup — NGOÀI PHẠM VI engine)

Feecover KHÔNG mua/bán/mint CARP, KHÔNG oracle (FEECOVER-HO-3). Chỉ 2 mũi tên, release qua Governance:

```
deposit(asset=CARP, amount, meta{source:"feecover", service_breakdown, epoch})
   → ghi FeecoverEpochSettle.deposited_l3; release qua Gov (status==Executed)
request_topup(asset, deficit, urgency)
   → ghi FeecoverEpochSettle.topup_requested; chuyển tiếp stabilizer_ref; L3 quyết nguồn bù
```

---

## 5. Kế toán epoch (đối soát)

```
per-tx (§4.1):   FeecoverAccrual(App, e).carp_accrued  += fee_magic
App→Platform:    Σ FeecoverAccrual(App, e) → FeecoverAccrual(Platform, e)   (trong epoch e)
epoch e đóng:    settle(e): Σ FeecoverAccrual(Platform, e).carp_accrued → Phoenix Treasury (CARP)
                 last_settled_epoch = e ; reset accrual
```

Bất biến kế toán tổng: `Σ carp_to_treasury (append) == Σ fee_magic của mọi consume Feecover đã settle`. Kiểm chứng qua `breakdown` per-service.

---

## 6. API endpoint (backend PhoenixKey — Long)

> Feecover backend KHÔNG tự dựng resolve API — nó GỌI resolve API point-in-time đã có (§7.3). Endpoint mới của Feecover:

| Method + path | Vào | Ra | Ghi chú |
|---|---|---|---|
| `GET /v1/feecover/quote?service_id={id}&epoch={e}` | service_id, epoch | `{ fee_magic, carp_amount, op_type, schedule_version }` | đọc `ServiceFeeSchedule` beacon; `carp_amount == fee_magic` |
| `GET /v1/feecover/gate?did={did}&key={pkh}&epoch={e}` | did, key, epoch | `{ allowed: bool, reason }` | proxy resolve API (§4.2 bước 1-3); 503 upstream → `allowed:false, reason:"resolve_unavailable"` (fail-closed) |
| `POST /v1/feecover/build-consume` | `{ did, service_id, op_count, signer, utxos }` | tx CBOR chưa ký (co-spend §4.1) | chỉ dựng khi gate pass; đính ref inputs [P][S][G][D] |
| `POST /v1/feecover/settle` | `{ closed_epoch }` | tx CBOR settle (§4.3) | epoch boundary; ép `closed_epoch == last_settled_epoch+1` |
| `GET /v1/feecover/accrual?tier={t}&epoch={e}&provider={id}` | tier, epoch, provider | `{ carp_accrued, breakdown }` | đọc `FeecoverAccrual` UTxO (audit/dashboard) |
| `GET /v1/feecover/settle-ledger` | — | `{ last_settled_epoch, carp_to_treasury, deposited_l3, topup_requested }` | đọc `FeecoverEpochSettle` |

Resolve API Feecover phụ thuộc (KHÔNG tự dựng — Resolve-API §3):
- `GET /identity/{did}/active?epoch={e}` → `{ active, revoked_at, never_existed }`
- `GET /identity/{did}/key-authorized?key={k}&epoch={e}` → `{ authorized }`
- 503 body `{code:5030, message:"resolve_unavailable"}` → consumer REJECT.

---

## 7. Phụ-thuộc + thứ-tự triển-khai (BLOCKER neo rõ)

### 7.1 🔴 BLOCKER-1 — enforce NỘI DUNG `did_commit` (MAGIC-team)

- **Hiện trạng CODE (đã verify):** `EngageDatum.did_commit : ByteArray` ĐÃ TỒN TẠI (`types.ak:46-51`, field 4 append-only). MVP = `#""`. `consume.ak:409` ĐÃ ép **immutable** (`od.did_commit == in.did_commit`). ĐÍNH CHÍNH Feat-Math §8.1 "cần thêm trường" — trường ĐÃ CÓ.
- **Cái CÒN THIẾU (blocker thật):** validator/vault CHƯA ép **NỘI DUNG** — `did_commit` rỗng-được, không bind về DID nào, không verify signer khớp DID. Cần MAGIC-team thêm invariant: `did_commit = blake2b256(did ‖ salt)` (salt = `enrolled_slot`/epoch, chống rainbow) + burn hợp lệ CHỈ khi signer/delegate khớp DID cam kết (qua resolve API `key_authorized`).
- **Feecover chịu tác động:** FC-GATE-3 (bind per-DID) chỉ bật khi blocker này xong. Trước đó Feecover chạy ở mức `owner`-key gating (attribution thô per-account, `breakdown` không truy được về DID thật).
- **Việc đã giao:** `_msg-to-MAGIC-reconcile.md §1` (draft, chờ anh duyệt gửi). Kèm blocker-2 dưới: hoà giải model MAGIC account-in-vault vs greenpeg native — nếu MAGIC lật về native, Feecover viết lại toàn bộ nhánh accrue.

### 7.2 🔴 BLOCKER-2 — CARP policy-id / asset-name / decimals (CARP-team)

- **Cần:** `policy_id` + `asset_name` (hex) canonical trên **preprod** (test) và **mainnet**; **decimals** (LAMP/MAGIC dùng nanogic=1e9; CARP theo gì?); xác nhận CARP là native token chuyển-nhượng-được (`FeecoverAccrual` giữ CARP trong value, Treasury nhận CARP).
- **Feecover chịu tác động:** không có policy-id → không lọc/đếm CARP trong value UTxO (FC-ACC-2), không định tuyến nhánh CARP (FC-PAY-2). Đây là dependency dùng-chung 4 module (Ví, Feecover, Protectme, Shielded custody).
- **Việc đã giao:** `_msg-to-CARP-policy-id.md` (draft, chờ anh duyệt gửi).

### 7.3 🟡 Resolve API point-in-time (PhoenixKey backend — Long)

- Spec ĐÃ CÓ: `PhoenixKey-PointInTime-Resolve-API-Feat-Math.md §3` (`/active`, `/key-authorized`, fail-closed 503). Long implement (chưa live). Feecover gate (§4.2) tiêu thụ, KHÔNG tự dựng.
- Point-in-time bắt buộc (query state tại `query_epoch`, không state hiện tại). Feecover dùng `current_epoch` của tx.

### 7.4 🟡 Đồng bộ 2 bảng giá (D6, Feat-Math §9)

- `PriceParam.base_price[op_type]` (ConsumeMAGIC) ↔ `ServiceFeeSchedule.fee_magic[service_id]` phải khớp cho mỗi cặp. Cách nối (§3.1): entry mang `op_type`, on-chain ép `fee_magic == price(op_type)` (FC-SCHED-2). Cadence gov: cùng 1 tx governance đổi cả hai HAY ràng khớp on-chain — chờ anh chốt (Feat-Math §9 D6).

### 7.5 Thứ-tự triển-khai (dependency-gated)

```
Bước 0 (song song, không chặn nhau):
  0a. CARP-team chốt policy-id (BLOCKER-2)          ─┐
  0b. Long implement resolve API §3 (7.3)           ─┤→ chặn Feecover on-chain build
  0c. Anh chốt §9 Feat-Math (fee_magic, tầng, D6)   ─┘

Bước 1 (sau 0a+0c): feecover.ak + 4 datum type + aiken test (mock, KHÔNG cần blocker-1)
  → chạy được ở mức owner-gating. FC-GATE-3 để placeholder.

Bước 2 (sau 0b): tx-builder offchain + backend endpoint §6 + gate off-chain §4.2

Bước 3 (sau BLOCKER-1 MAGIC-team): bật FC-GATE-3 (bind per-DID) + breakdown truy-DID

Bước 4: e2e preprod (cần 0a CARP preprod policy) — §8
```

Feecover on-chain (bước 1) KHÔNG chặn bởi blocker-1; chặn bởi 0a (CARP policy để test value CARP) + 0c (số phí). Blocker-1 chỉ nâng gating từ owner → per-DID (bước 3).

---

## 8. Test / evidence (tiêu chí PASS)

> Bám kỷ luật ConsumeMAGIC: aiken test mock Transaction, happy + negatives cho MỖI bất biến. KHÔNG tuyên bố PASS khi chỉ compile.

### 8.1 Unit (aiken test, `feecover.ak` — mock tx, không cần chain)

| Test | Kỳ vọng |
|---|---|
| `fc_pay_happy` | `magic_consumed==fee_magic`, `carp_paid==fee_magic` → PASS |
| `fc_underpay_carp_fail` | `carp_paid < fee_magic` → fail (FC-PAY-2, `==` không `≥`) |
| `fc_overpay_carp_fail` | `carp_paid > fee_magic` → fail |
| `fc_service_mismatch_fail` | khai `service_id` rẻ nhưng burns nhiều (`fee_magic != price(op_type)`) → fail (FC-SCHED-2) |
| `fc_accrual_datum_value_mismatch_fail` | `datum.carp_accrued != quantity_of(value,carp)` → fail (FC-ACC-2) |
| `fc_accrual_wrong_epoch_fail` | accrue vào epoch sai → fail (FC-ACC-1) |
| `fc_gate_no_did_fail` | thiếu DID credential ref input → fail (FC-GATE-1) |
| `fc_gate_did_minted_fail` | DID credential bị mint/tiêu trong tx → fail (FC-GATE-3 no-self-issue) |
| `fc_gate_stale_fail` | `current_epoch > epoch_valid_until` → fail (FC-GATE-2) |
| `fc_schedule_stale_fail` | `ServiceFeeSchedule` quá cũ → fail (FC-SCHED-1) |
| `fc_schedule_fake_nft_fail` | beacon NFT qty≠1 → fail |
| `fc_settle_happy` | `closed==last+1`, Σ CARP → Treasury → PASS |
| `fc_settle_skip_epoch_fail` | `closed != last+1` → fail (FC-SET-1) |
| `fc_settle_double_fail` | settle lại epoch đã settle → fail |
| `fc_settle_lamp_to_treasury_fail` | chuyển LAMP/MAGIC thay CARP → fail (FC-SET-2) |
| `fc_settle_wrong_treasury_addr_fail` | CARP tới địa chỉ ≠ treasury hiến định → fail (FC-SET-3) |
| `fc_didcommit_bind_happy` (bước 3) | `did_commit==blake2b256(did‖salt)` khớp → PASS |

### 8.2 Integration (co-spend consume.ak + feecover.ak, mock tx)

- Chuỗi đóng kín: `Σburns == required(consume.ak) == fee_magic == carp_paid`. Test 1 tx với cả 3 redeemer (BurnBatch, Consume, Accrue) → verify không kẽ under-charge (đối chiếu `consume_two_engage_share_vault_undercharge_fail`).
- **KHÔNG sửa** file `consume.ak`; import + co-spend trong test harness Feecover.

### 8.3 E2E preprod (evidence output thật — sau bước 4)

- Cần CARP preprod policy (0a). Dựng: PriceParam + ServiceFeeSchedule + FeecoverGate beacon + engage UTxO + FeecoverAccrual UTxO → 1 tx consume+feecover thật → verify on-chain: `consumed_count` +op_count, số dư MAGIC vault giảm đúng, `FeecoverAccrual.carp_accrued` += fee_magic, value CARP UTxO tăng đúng.
- Settle: qua epoch boundary → verify CARP tới Treasury address, `carp_to_treasury` tăng đúng, accrual reset.
- Evidence: tx hash preprod + datum decode trước/sau + số dư CARP Treasury trước/sau (`curl` Blockfrost). KHÔNG tuyên bố PASS thiếu output này (CLAUDE.md verify-behavior).

---

## 9. Ranh-giới giao-việc (bảng phân công)

| Việc | Chủ | Chặn bởi | File/spec neo |
|---|---|---|---|
| `feecover.ak` validator + 4 datum type + aiken test | Claude/PhoenixKey onchain | 0a, 0c | `feecover/onchain/` (mới), §3, §4, §8.1 |
| tx-builder offchain co-spend (consume+feecover) | Claude/PhoenixKey offchain | 0a, 0b | `feecover/offchain/` (mới), §4.1 |
| Backend endpoint §6 (quote/gate/build/settle/accrual/ledger) | **Long** (PhoenixKey backend) | 0b (resolve API) | §6 |
| Resolve API point-in-time `/active` `/key-authorized` | **Long** | — (spec có) | Resolve-API §3, §7.3 |
| **Enforce nội-dung `did_commit`** per-DID + signer bind | **MAGIC-team** | — | §7.1, `_msg-to-MAGIC-reconcile.md §1` |
| Hoà giải model MAGIC (account-in-vault vs greenpeg native) | **MAGIC-team** | — | `_msg-to-MAGIC-reconcile.md §2` (gốc lật model) |
| **CARP `policy_id`/`asset_name`/decimals** preprod+mainnet | **CARP-team** | — | §7.2, `_msg-to-CARP-policy-id.md` |
| Ổn-định-giá CARP (GreenPeg/RedPeg/oracle/buyback) | CARP L3 | — | 🚫 ngoài phạm vi; Feecover chỉ `deposit`/`request_topup` (§4.4) |
| Chốt `fee_magic`/tầng App-Platform/`stabilizer_ref`/D6 | **Anh** (chiến lược) | — | Feat-Math §9 (D1-D6) |

---

## 10. Ranh-giới cứng (KHÔNG vượt)

- KHÔNG sửa `consume.ak` / C-CM-1..5. Feecover co-spend, không nhét logic vào engine MAGIC-team.
- KHÔNG mint/burn CARP/LAMP trong `feecover.ak`; KHÔNG đọc oracle giá (FEECOVER-HO-3 — giữ INV-PEG-BY-DEMAND, INV-NO-LAMP-PEG-DEFENSE, `1 CARP=1 MAGIC` ở tầng CARP).
- KHÔNG `collectToTreasury` cho MAGIC (đường đã bỏ). MAGIC = số dư account, KHÔNG rời account thành token. Chỉ CARP về Treasury.
- CHỈ CARP về Phoenix Treasury (FC-SET-2). KHÔNG LAMP (backing tầng khác), KHÔNG MAGIC.
- PhoenixKey KHÔNG sửa code MAGIC-team/CARP-team — phát hiện lỗi → báo anh / tạo issue (CLAUDE.md ranh-giới).
```
