# PhoenixKey — Feecover (Feat + Math)

> Đổi tên từ Payyou → Feecover (anh chốt 2026-07-03). Bản chất không đổi: lớp trừu-tượng-hoá phí — user trả 1 loại (MAGIC), hệ thống lo phần ADA/DUST/phí dịch vụ khác.

## Trạng thái: SPEC PROPOSAL (design doc, chờ anh chốt) · 2026-07-01

**Một câu:** Feecover là **lớp trừu-tượng-hoá phí (fee-abstraction) của riêng PhoenixKey ĐẶT LÊN TRÊN cơ chế ConsumeMAGIC đã có** — KHÔNG phải một engine phí mới. Feecover tái dùng mô hình tiêu-MAGIC (số dư account trong Vault) để user luôn **định giá dịch vụ bằng MAGIC nhưng THANH TOÁN bằng CARP**, thêm vào: (1) bảng phí ổn định theo loại dịch vụ PhoenixKey, (2) cổng chỉ-DID-PhoenixKey, (3) luồng gom phí vào Vault provider theo tầng App→Platform rồi cuối epoch đối soát chuyển **CARP về Phoenix Treasury**.

---

## 0. Mô hình MAGIC ĐÚNG (anh đính chính 2026-07-01) — đọc trước

> **Mô hình: Triple Token** (LAMP · MAGIC · CARP) — KHÔNG phải Dual (2 token), KHÔNG phải one-token native (HopNhat lỗi thời). **Nguồn canonical: `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md`** (NGUỒN CHÂN LÝ đơn-vị + peg — anh chốt 2026-07-06). `MagicLamp-3Token-DacTa-Vi.md` là bản cũ, phần đơn-vị (KB·ngày) + peg (chuộc-par) ĐÃ BỊ ĐÈ bởi Whitepaper — KHÔNG dùng làm nguồn.

Bản spec trước giả định SAI rằng "MAGIC là native token, chuyển-nhượng, đốt-qua-BurnBatch" (mô hình one-token/HopNhat). **Bỏ hoàn toàn tiền đề đó.** Mô hình đúng là **ba token, ba vai đối nghịch không thể gộp** (Đèn sinh Điều ước; Thảm chở giá trị tới nơi tiêu; Điều ước tan biến sau khi phục vụ chủ):

| Token | Bản chất | Chuyển nhượng | Vai trong Feecover |
|---|---|---|---|
| **LAMP** | Tài nguyên nền, cung cố định 36 tỷ | Có | Nền backing (ngoài Feecover) |
| **MAGIC** | **Số dư account trong Vault** — con số kế toán gắn DID, KHÔNG phải token, KHÔNG có policy-id | **KHÔNG** | **Đơn vị định giá/đo phí** (unit of account) |
| **CARP** | Token ổn định nội sinh, fungible | Có | **Đơn vị thanh toán/settlement** |

**Hai nguyên tắc lõi Feecover bám theo (nguồn canonical: `Whitepaper-MagicLamp-Tokenomic-Vi.md §4` MAGIC, `§5` CARP, `§8.1` sàn-tiện-ích):**

- **Đơn vị kế toán = MAGIC; thanh toán/settlement = CARP.** Mọi chuyển động được **định giá/đo bằng MAGIC** (Điều Ước — số dư account trong Vault, non-transferable, decay) nhưng **trả bằng CARP** (Tấm Thảm — đồng-thanh-khoản chuyển-nhượng-được). Neo bất biến `1 CARP = 1 MAGIC` (sức-mua-dịch-vụ nội sinh, KHÔNG neo fiat), đơn vị nhỏ nhất `1 nanogic` (**Whitepaper §4: `1 nanogic = 1 byte·ngày` lưu-lạnh → `1 MAGIC = 1 GB·ngày`; mốc `KB·ngày` cũ ĐÃ BỎ**).
- **Tiêu MAGIC = GIẢM SỐ DƯ account trong Vault**, KHÔNG mint/burn token, KHÔNG BurnBatch. Đúng mô hình `ConsumeMAGIC/onchain/validators/consume.ak`: mỗi consume giảm balance account trong Vault, thread NFT (engage token) được bảo toàn, state (`consumed_count`, `last_epoch`) tăng đúng, value non-MAGIC bảo toàn byte-perfect (C-CM-1..5).

**Hệ quả cho Feecover:** KHÔNG dùng `collectToTreasury` cho MAGIC (đường đó đã bỏ). Thay bằng luồng gom-phí-CARP theo tầng (§6).

**Ranh giới cứng (anh chốt):**
- KHÔNG dựng lại engine consume. Tái dùng `ConsumeMAGIC/onchain/{lib,validators}` hiện có.
- Cơ chế ổn-định-giá CARP (GreenPeg/RedPeg, buyback, oracle thế-chấp) **NGOÀI PHẠM VI** — thuộc giao thức CARP cấp hệ sinh thái. Feecover chỉ định nghĩa **interface bàn-giao** phần dư về Phoenix Treasury.
- KHÔNG thiết kế oracle/buyback. KHÔNG chạm bất biến lõi CARP (Whitepaper §8, §10 Firewall): **INV-PEG-BY-DEMAND** (peg-core giữ bởi **cầu-dịch-vụ-THỰC** — utility-floored, KHÔNG backing-chuộc-ra-rổ; `P_redeem ≡ 1` oracle-free CHỈ ở tuyến-phụ CDP-LAMP — Whitepaper §8.1/§8.3), **INV-NO-LAMP-PEG-DEFENSE** (F9: tuyệt đối không đỡ-peg bằng LAMP), **`1 CARP = 1 MAGIC`** sức mua (bất biến hiến pháp) giữ nguyên.

Nguồn bám: `MAGIC-paymaster/ConsumeMAGIC/{CONTRACT.md, onchain/lib/magiclamp/consume/types.ak, onchain/validators/consume.ak}`; **`MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` (Whitepaper Tokenomic canonical — NGUỒN CHÂN LÝ LAMP·MAGIC·CARP, đơn-vị + peg)**; `MAGIC/SPEC/Carpet-CARP-DacTa-Vi.md` (chi-tiết cơ chế ổn định CARP + 3 Gen); `VeDataIO/Specs/PhoenixKey-VeData-Contract.md v0.2.0 §2.1` (resolve API point-in-time).

---

## 1. Bối cảnh + quan hệ với ConsumeMAGIC

### 1.1 Vấn đề Feecover giải

Người dùng PhoenixKey khi tiêu dịch vụ (neo DID, xác thực VC, custody, rotate khoá…) phải trả phí. Yêu cầu của anh:

1. Mỗi loại dịch vụ có **một mức phí ổn định dài hạn, định giá bằng MAGIC** — user luôn nhìn phí bằng đơn-vị-sức-mua ổn định.
2. **Thanh toán quyết toán bằng CARP** — CARP là token lưu thông, ổn định theo cùng mỏ neo (`1 CARP = 1 MAGIC`).
3. **Chỉ user PhoenixKey DID** được dùng Feecover.
4. Khi user tiêu, phí gom vào **Vault provider** theo tầng App→Platform; **cuối mỗi epoch** đối soát kế toán, chuyển **CARP về Phoenix Treasury** (KHÔNG LAMP, KHÔNG MAGIC).

Điểm (1) và (2) **ConsumeMAGIC đã làm phần lõi**: nó đã có mô hình đo-tiêu-theo-MAGIC (giá lấy từ beacon `PriceParam`, giảm số dư account trong Vault). Đây là điều quan trọng nhất phải nói thẳng: **phần lớn engine đo-tiêu đã có sẵn.** Feecover KHÔNG viết lại nó.

### 1.2 ConsumeMAGIC đã có gì (tóm lược, để không dựng lại)

- **Consume = giảm số dư account trong Vault.** `EngageDatum { owner, consumed_count, last_epoch }` neo vào continuing output của Vault UTxO qua **thread NFT** (chống replay — C-CM-4). Mỗi consume: `consumed_count += op_count`, `last_epoch = current_epoch`, thread NFT bảo toàn (`consume.ak:enforce_engagement`).
- **Giá có thẩm quyền lấy từ beacon `PriceParam`** (reference input CIP-31, NFT-authenticated) — `price(op) = base_price[op] × demand_mult / Q`. Validator KHÔNG tin amount client (C-CM-2). Đơn vị `base_price` = **nanogic** (`1 MAGIC = 1e9`), khớp `1 thread = 1 nanogic` phía CARP.
- **Bất biến C-CM-1..5:** value non-MAGIC bảo toàn byte-perfect (chống drain ADA/LAMP/CARP khỏi Vault — `non_magic_value_preserved`); chống double-satisfaction gộp N-vault-input (C-CM-3, đếm theo script hash + `#out==#in` + Σ thread NFT bảo toàn); stale-price gate (C-CM-5).
- **`did_commit` HIỆN LÀ sentinel rỗng.** `EngageDatum` hiện chỉ có `owner` (khoá), CHƯA có trường `did_commit` gắn per-DID — tương đương `did_commit = #""`. Đây là **phụ thuộc MAGIC-team** phải thêm (§8).

### 1.3 Bảng CHỒNG LẤN (chức năng | ConsumeMAGIC đã có | Feecover thêm mới)

| Chức năng | ConsumeMAGIC đã có | Feecover thêm mới |
|---|---|---|
| Tiêu MAGIC = giảm số dư account trong Vault, thread NFT bảo toàn | ✅ Trọn vẹn — `consume.ak`, C-CM-1..5 | ⟳ **Tái dùng nguyên trạng.** Feecover KHÔNG chạm luồng consume |
| Giá lấy từ beacon, không tin client | ✅ `PriceParam` + `read_price_param` (C-CM-2) | ⟳ Tái dùng cơ chế beacon. `ServiceFeeSchedule` là beacon **cùng khuôn** (NFT-authenticated, reference input) |
| Value non-MAGIC (ADA/LAMP/CARP) bảo toàn trong Vault | ✅ `non_magic_value_preserved` (C-CM-1) | ⟳ Tái dùng. Feecover KHÔNG nới bất biến value |
| Chống double-satisfaction + replay | ✅ C-CM-3/C-CM-4 (thread NFT, `#out==#in`) | ⟳ Tái dùng |
| Stale-price freshness | ✅ C-CM-5 (`max_price_stale`) | ⟳ Tái dùng mẫu cho `ServiceFeeSchedule` freshness |
| **Bảng phí THEO LOẠI DỊCH VỤ PhoenixKey** (neo, VC, custody…) | ⚠️ Có `op_type → base_price` nhưng generic, không mang ngữ nghĩa "dịch vụ PhoenixKey gì" | 🟢 **MỚI** — `ServiceFeeSchedule` ánh xạ `service_id → fee_magic`, versioned + gov-gated |
| **Cổng chỉ user PhoenixKey DID** | ❌ Chỉ biết `owner` (khoá), không ràng DID thật | 🟢 **MỚI** — `FeecoverGate` ép account gắn PhoenixKey DID qua resolve API |
| **Quy đổi định-giá-MAGIC → thanh-toán-CARP** | ❌ Consume chỉ giảm số dư; không định nghĩa CARP settlement | 🟢 **MỚI** — Feecover ép `carp_paid == fee_magic` (neo 1:1), gom CARP theo tầng |
| **Gom phí theo tầng App→Platform, cuối-epoch → Phoenix Treasury** | ❌ Không có | 🟢 **MỚI** — `FeecoverAccrual` (per-tx accrue) + `FeecoverEpochSettle` (§6) |
| **Ổn định phí dài hạn** (dựa sức mua MAGIC) | ⚠️ `demand_mult` điều tiết ngắn hạn; không policy "ổn định dài hạn" | 🟢 **MỚI** — policy re-anchor + cadence quản trị (§4) |
| Ổn-định-giá CARP, GreenPeg/RedPeg, oracle, buyback | ❌ Không (thuộc giao thức CARP) | 🚫 **NGOÀI PHẠM VI** — Feecover chỉ trỏ tới (external actor) |

**Kết luận trung thực:** 5/11 dòng là **tái dùng nguyên trạng** ConsumeMAGIC. Feecover chỉ đóng góp 5 phần chính sách mới (bảng phí dịch vụ, cổng DID, quy-đổi-CARP, gom-theo-tầng, policy ổn định) + trỏ 1 phần ra ngoài. Feecover là **lớp mỏng**, không phải engine.

---

## 2. Kiến trúc lớp (Feecover = policy/routing over ConsumeMAGIC)

```
┌──────────────────────────────────────────────────────────────┐
│  L3  Giao thức ỔN-ĐỊNH-GIÁ CARP (NGOÀI PHẠM VI)              │
│      GreenPeg/RedPeg · backing · oracle thế-chấp            │
│      ↑ deposit           ↓ request-topup                     │
├──────────────────────────────────────────────────────────────┤
│  L2  FEECOVER (lớp mới, mỏng) — chính sách + định tuyến       │
│   • ServiceFeeSchedule : service_id → fee_magic (§4)         │
│   • FeecoverGate         : chỉ PhoenixKey DID (§5)             │
│   • carp_paid == fee_magic (định-giá-MAGIC → trả-CARP)      │
│   • FeecoverAccrual (per-tx) → FeecoverEpochSettle (cuối epoch)  │
├──────────────────────────────────────────────────────────────┤
│  L1  CONSUMEMAGIC (tái dùng nguyên trạng)                    │
│   consume.ak · EngageDatum · thread NFT · PriceParam beacon │
│   C-CM-1..5 (KHÔNG sửa)                                      │
├──────────────────────────────────────────────────────────────┤
│  L0  Vault (số dư MAGIC account) · Vault provider (gom CARP) │
│   · Phoenix Treasury (nhận CARP cuối epoch)                 │
└──────────────────────────────────────────────────────────────┘
```

**Nguyên tắc kiến trúc:**

- **P-LAYER-1 — Feecover KHÔNG là input của validator `consume.ak`.** Feecover không thêm bất biến vào ConsumeMAGIC. Nó đọc `ServiceFeeSchedule` + `FeecoverGate` dạng **reference input** trong cùng tx consume, và ràng buộc **nhánh thanh toán CARP** riêng. `consume.ak` giữ nguyên C-CM-1..5.
- **P-LAYER-2 — Ép sớm, đọc thật.** `FeecoverGate` + `ServiceFeeSchedule` ép **trong cùng tx** với consume, đọc `owner`/`did_commit` (khi MAGIC-team gắn) và lượng-MAGIC-tiêu **thật** mà `consume.ak` đã tính từ beacon (không tin lại redeemer). Feecover ép `magic_consumed == fee_magic(service_id)` và `carp_paid == fee_magic` (neo 1:1).
- **P-LAYER-3 — Định tuyến, không định giá on-chain phức tạp.** Feecover chỉ **chọn** `fee_magic` (từ bảng) + gom CARP vào Vault provider đúng tầng. Quy đổi giá trị/oracle nằm ở L3, ngoài Feecover.

**Vì sao lớp mỏng:** first-principles — engine consume + kế toán state + bảo toàn value đã đúng và đã audit (C-CM-1..5). Thêm engine thứ hai = nợ kỹ thuật + bề mặt tấn công đôi. Feecover chỉ cần **ánh xạ dịch vụ → phí**, **cổng DID**, **quy-đổi-CARP**, và **gom-theo-tầng**.

---

## 3. Data model (mới, tối thiểu)

Tên field tiếng Anh; ngữ nghĩa tiếng Việt.

### 3.1 `ServiceFeeSchedule` (beacon, DAO quản trị — cùng khuôn `PriceParam`)

```
ServiceFeeSchedule {
  schedule_nft   : PolicyId × AssetName   // beacon authenticity (1/instance, như price NFT)
  version_epoch  : Int                     // epoch phát hành bảng phí này (freshness)
  entries        : List<ServiceFeeEntry>   // ánh xạ dịch vụ → phí
  gov_ref        : ScriptHash              // Governance đã duyệt bảng này
}

ServiceFeeEntry {
  service_id     : ByteArray   // định danh dịch vụ PhoenixKey (vd "did.anchor", "vc.verify")
  fee_magic      : Int         // nanogic — phí danh nghĩa (đơn-vị-kế-toán) cho op này
  enabled        : Bool
}
```

- Chỉ chỉnh qua Governance (đọc `status==Executed`, `gov_ref`). KHÔNG tự-tính ngưỡng on-chain.
- `fee_magic` là **nanogic tuyệt đối** — trùng đơn vị `base_price` của `PriceParam` (`1 MAGIC = 1e9`) và `1 thread` CARP. Không cần quy đổi tỷ giá; CARP trả = `fee_magic` thread (neo 1:1).

### 3.2 `FeecoverGate` (reference input — cổng DID)

```
FeecoverGate {
  gate_nft            : PolicyId × AssetName
  did_registry_policy : PolicyId            // policy NFT của PhoenixKey DID Registry
  resolve_ref         : ScriptHash          // điểm neo resolve API point-in-time (VeData §2.1)
  epoch_valid_until   : Int                 // freshness (như max_price_stale)
}
```

### 3.3 `FeecoverAccrual` (state UTxO — gom phí CARP per-tx trong epoch)

```
FeecoverAccrual {
  accrual_nft     : PolicyId × AssetName
  tier            : Tier                   // App | Platform (tầng gom)
  epoch           : Int                    // epoch đang gom
  carp_accrued    : Int                    // Σ CARP (thread) gom trong epoch này
  breakdown       : List<(service_id, Int)>// phân rã theo dịch vụ (audit)
}
```

- Mỗi consume Feecover **cộng CARP** vào `FeecoverAccrual` của tầng App tương ứng; đối soát App→Platform trong epoch. `carp_accrued` **chỉ tăng trong epoch**, reset khi epoch mới sau khi settle.

### 3.4 `FeecoverEpochSettle` (state UTxO — sổ đối soát cuối epoch → Phoenix Treasury + bàn-giao L3)

```
FeecoverEpochSettle {
  settle_nft        : PolicyId × AssetName
  last_settled_epoch: Int
  carp_to_treasury  : Int                  // Σ CARP đã chuyển về Phoenix Treasury (append-only)
  deposited_L3      : List<(Asset, Int)>   // đã nộp sang giao thức ổn-định CARP (tích luỹ)
  topup_requested   : List<(Asset, Int)>   // đã yêu cầu bù về (tích luỹ)
  stabilizer_ref    : ScriptHash           // địa chỉ giao thức ổn-định-giá CARP (external, L3)
}
```

---

## 4. Bảng phí ổn định theo dịch vụ (định-giá-MAGIC, thanh-toán-CARP)

### 4.1 Mô hình

- Mỗi `service_id` có một `fee_magic` (nanogic) — **giá danh nghĩa bằng đơn-vị-kế-toán MAGIC**. Consume giảm đúng lượng đó khỏi số dư account (`magic_consumed == fee_magic(service_id)`), và **thanh toán bằng CARP** với `carp_paid == fee_magic` (neo `1 CARP = 1 MAGIC`, `1 thread = 1 nanogic`).
- **User định giá bằng MAGIC, trả bằng CARP** — không có nhánh nào bắt user hiểu tỷ giá. MAGIC là thước-sức-mua ổn định; CARP là đồng lưu thông cùng mỏ neo. Đây là cam kết cốt lõi.

Ví dụ bảng (số minh hoạ, chờ anh chốt §8):

| `service_id` | Dịch vụ | `fee_magic` (nanogic) | CARP trả (thread) |
|---|---|---|---|
| `did.anchor` | Neo/cập DID lên chain | 5 × 10⁹ (= 5 MAGIC) | 5 × 10⁹ (= 5 CARP) |
| `did.rotate` | Rotate khoá thành viên | 2 × 10⁹ | 2 × 10⁹ |
| `vc.verify` | Xác thực VC / selective disclosure | 1 × 10⁹ | 1 × 10⁹ |
| `custody.op` | Thao tác custody LampNet | 3 × 10⁹ | 3 × 10⁹ |

### 4.2 "Ổn định dài hạn" đạt được thế nào (RE-DERIVE trên sức mua MAGIC)

Đây là câu hỏi bản chất. Tiền đề đúng: **1 MAGIC = một đơn vị sức mua dịch vụ nền, cố định danh nghĩa theo `base_price`** (**Whitepaper §4**: `1 nanogic = 1 byte·ngày` lưu-lạnh → `1 MAGIC = 1 GB·ngày`; neo par `P* = 1`, KHÔNG neo fiat). MAGIC là **đơn vị-kế-toán sức mua ổn định theo thiết kế** — nó không "trôi theo giá thị trường" vì MAGIC không giao dịch, chỉ là số dư account trong Vault (non-transferable). Và CARP neo cứng `1 CARP = 1 MAGIC` (bất biến hiến pháp, Whitepaper §5), nên **trả CARP theo `fee_magic` cũng ổn định-sức-mua như nhau**.

Do đó "ổn định dài hạn" của Feecover = **hai lớp**:

1. **Neo danh nghĩa (mặc định, zero-oracle):** `fee_magic` giữ NGUYÊN theo nanogic. Vì 1 nanogic ≡ 1 đơn vị sức mua nền, phí tự-ổn-định miễn là **giá dịch vụ nền** (`base_price`) không đổi. CARP trả = `fee_magic` thread, cùng sức mua. KHÔNG cần oracle, KHÔNG chạm INV-PEG-BY-DEMAND (peg-core = cầu-thực).
2. **Re-anchor bằng quản trị (khi `base_price` thực thay đổi):** khi DAO điều chỉnh `base_price` nền (Whitepaper §4: base_price khoá on-chain, đổi chỉ qua DAO vote hiến pháp), **cả MAGIC lẫn CARP mua được nhiều hơn theo cùng tỷ lệ**, tỷ giá sức mua vẫn 1:1. Nếu chi phí hạ tầng thật của một dịch vụ PhoenixKey trôi khác nhịp `base_price` chung, DAO phát hành `ServiceFeeSchedule` mới (`version_epoch` tăng), điều chỉnh `fee_magic` của service đó. Đây là **quyết định con người theo cadence**, không phải hàm giá tự động bám giá ngoài (giữ peg-by-demand, không introducing oracle).

**Vì sao lập luận ổn định KHÔNG dựa vào "MAGIC non-transfer":** tiền đề "MAGIC không chuyển-nhượng nên ổn định" là **đúng nhưng KHÔNG phải trục lập luận** — trục thật là **MAGIC neo sức-mua-dịch-vụ-nền cố định danh nghĩa**, và CARP neo cứng vào MAGIC. Ổn định đến từ **mỏ neo sức mua**, không từ tính không-chuyển-nhượng. (Tiền đề "MAGIC là native token / BurnBatch là premise" của bản trước đã chết.)

**Vì sao KHÔNG buộc phí bám giá ADA/USD real-time:** làm vậy cần oracle giá ngoài → phá zero-oracle ở chuộc-par. Việc hấp thụ biến động thị trường là nhiệm vụ của **giao thức ổn-định-giá CARP L3** (GreenPeg/RedPeg), KHÔNG phải của bảng phí Feecover. Feecover giữ phí danh nghĩa ổn định; L3 lo phần đệm giá CARP. Phân tách sạch trách nhiệm.

### 4.3 Cadence quản trị

- Điều chỉnh `fee_magic` chỉ qua Governance (`status==Executed`, `gov_ref`). Biên mượn khung `base_price` (đổi qua DAO vote hiến pháp — Whitepaper §4, `Carpet-CARP-DacTa-Vi.md`) hoặc đặt riêng cho PhoenixKey — **chờ anh chốt §8**.
- Mỗi lần đổi tăng `version_epoch`; tx consume phải đọc bảng có `version_epoch` mới nhất (freshness giống C-CM-5 `max_price_stale`).

---

## 5. Gating — chỉ user PhoenixKey DID

**Yêu cầu:** chỉ user có PhoenixKey DID mới dùng được Feecover.

**Cơ chế (tái dùng hạ tầng DID + resolve API sẵn có, thêm một vị từ):**

- **FEECOVER-GATE-1 — DID xác thực point-in-time.** Trong tx consume, account owner PHẢI gắn một PhoenixKey DID hợp lệ. Ép bằng: reference input mang credential DID (NFT DID / entry registry) với `policy == FeecoverGate.did_registry_policy`, đối chiếu qua **resolve API point-in-time** (`VeData §2.1`): `did_active(did, current_epoch)` phải trả `active`, và nếu tx ký bằng khoá thì `key_authorized(key, did, current_epoch)` (Stamp §10 key-rotation rule — verify khoá đã authorize cho DID tại thời điểm đó).
- **FEECOVER-GATE-2 — Freshness.** `current_epoch − FeecoverGate.epoch_valid_until ≤ 0` (cùng mẫu C-CM-5).
- **FEECOVER-GATE-3 — Không tự-phát-hành.** DID credential đọc qua **reference input** (CIP-31), KHÔNG tiêu, KHÔNG mint trong tx Feecover (tránh Feecover tự đúc DID giả).

**Quan hệ với `did_commit`:** hiện `EngageDatum` chỉ có `owner`, chưa gắn per-DID (`did_commit = #""` — §1.2). Khi MAGIC-team gắn trường `did_commit` (blake2b256 thật của DID) vào EngageDatum, FEECOVER-GATE-1 nâng cấp thành **ràng `did_commit` khớp DID đã resolve** — attribution per-DID chặt chẽ. Đây là **phụ thuộc chéo** (§8), Feecover tiêu thụ nó chứ không phát minh.

**Hệ quả kế toán:** khi `did_commit` gắn per-DID thật, phí gom trong `FeecoverAccrual.breakdown` truy được về từng DID → attribution phí đúng người thật, chống một owner mở nhiều account né trần.

---

## 6. Luồng gom phí theo tầng → cuối-epoch → Phoenix Treasury (CARP)

Đây là thay đổi lõi so với bản trước. **Bỏ `collectToTreasury` cho MAGIC.** Phí gom bằng **CARP** vào Vault provider theo tầng, chỉ cuối epoch mới đối soát chuyển về Phoenix Treasury.

### 6.1 Luồng per-transaction (accrue vào Vault provider)

1. User consume dịch vụ → `consume.ak` giảm **số dư MAGIC account** đúng `fee_magic` (đo bằng đơn-vị-kế-toán MAGIC). Thread NFT + value non-MAGIC bảo toàn (C-CM-1..5).
2. Cùng tx, Feecover ép **thanh toán CARP** `carp_paid == fee_magic` (thread), gom vào `FeecoverAccrual` **tầng App** của provider phục vụ op đó: `carp_accrued += carp_paid`, ghi `breakdown[service_id] += carp_paid`.
3. Đối soát **tầng Platform**: `FeecoverAccrual` tầng App tổng hợp lên tầng Platform trong epoch (App phục vụ nhiều DID; Platform gom nhiều App). Đây là kế toán **trong epoch**, chưa chạm Treasury.

### 6.2 Luồng cuối-epoch (settle → Phoenix Treasury)

- Khi sang epoch mới, `FeecoverEpochSettle` đối soát tổng `carp_accrued` của các `FeecoverAccrual` tầng Platform cho epoch vừa đóng, chuyển **CARP về Phoenix Treasury**:
  ```
  epoch_settle(closed_epoch) :
    require closed_epoch == last_settled_epoch + 1        // tuần tự, không nhảy
    total_carp = Σ FeecoverAccrual(Platform, closed_epoch).carp_accrued
    → chuyển total_carp CARP tới Phoenix Treasury (địa chỉ hiến định)
    → carp_to_treasury += total_carp   (append-only, audit)
    → last_settled_epoch = closed_epoch
    → reset carp_accrued của epoch đã settle
  ```
- **Chỉ CARP về Treasury** — KHÔNG LAMP (LAMP là backing ở tầng khác, không chảy qua Feecover fee), KHÔNG MAGIC (MAGIC là số dư account, không rời account thành token). Đây là ranh giới cứng của mô hình đúng.

### 6.3 Interface bàn-giao sang giao thức ổn-định-giá CARP (external, L3)

**Feecover KHÔNG mua/bán CARP, KHÔNG oracle, KHÔNG buyback.** Feecover chỉ định nghĩa **hai mũi tên** giữa Phoenix Treasury và giao thức ổn-định-giá CARP (GreenPeg/RedPeg, ngoài phạm vi):

```
deposit(asset, amount, meta) -> DepositReceipt        // Phoenix Treasury → L3
  asset  = CARP (đã gom cuối epoch)
  meta   : { source:"feecover", service_breakdown:List<(service_id,amount)>, epoch:Int }
  → ghi vào FeecoverEpochSettle.deposited_L3
  → release phải qua Governance gate (status==Executed)

request_topup(asset, deficit, urgency) -> TopupRequest // L3 → bù về (khi dự trữ cạn)
  → ghi vào FeecoverEpochSettle.topup_requested
  → chuyển tiếp tới stabilizer_ref (L3), L3 quyết nguồn bù
```

**Nguồn bù (external actors — chỉ trỏ tới):** RedPeg (vốn-đỏ-vô-chủ), MagicLamp Treasury, Platform Treasury. Thứ tự ưu tiên/ngưỡng do **L3 + Governance** định — KHÔNG thuộc Feecover.

### 6.4 Bất biến gom-phí + bàn-giao (Feecover-only, không chạm ConsumeMAGIC/CARP-lõi)

| ID | Bất biến |
|---|---|
| FEECOVER-PAY-1 | Mỗi consume Feecover: `carp_paid == fee_magic(service_id)` (neo 1:1 thread↔nanogic); thiếu/thừa → fail |
| FEECOVER-PAY-2 | CARP chỉ accrue vào `FeecoverAccrual` đúng tầng+epoch; `carp_accrued` chỉ tăng trong epoch |
| FEECOVER-SETTLE-1 | `epoch_settle` chỉ chạy tuần tự (`closed_epoch == last_settled_epoch + 1`), không bỏ/nhảy epoch |
| FEECOVER-SETTLE-2 | Chỉ **CARP** chuyển về Phoenix Treasury; KHÔNG LAMP/MAGIC |
| FEECOVER-SETTLE-3 | `carp_to_treasury` / `deposited_L3` / `topup_requested` chỉ tăng (append-only, audit) |
| FEECOVER-HO-1 | `deposit` chỉ chuyển CARP ĐÃ ở Phoenix Treasury; release qua Governance `status==Executed` |
| FEECOVER-HO-2 | `stabilizer_ref` chỉ đổi qua Governance |
| FEECOVER-HO-3 | Feecover KHÔNG mint/burn CARP/LAMP; KHÔNG đọc oracle giá (giữ INV-PEG-BY-DEMAND ở tầng CARP) |

---

## 7. Phản biện đối kháng (self-adversarial)

Đối chiếu với bề mặt tấn công đã biết của ConsumeMAGIC/CARP; chỉ liệt cái Feecover THÊM hoặc còn hở.

**A. Fee spam (tiêu vô nghĩa để bơm số accrual).**
Kẻ tấn công consume liên tục để inflate `carp_accrued`. → Mỗi op **giảm số dư MAGIC thật** của chính account đó + **trả CARP thật** = tự-tiêu tài nguyên. FEECOVER-GATE-1 buộc DID thật (khi `did_commit` gắn) ⇒ không mở account ảo né trần. **Còn hở:** nếu chưa gắn `did_commit`, gate chỉ ràng `owner` — một owner có thể mở nhiều account. Giảm thiểu: chờ dependency `did_commit` (§8); trước đó dựa registry-policy per-account.

**B. Account-drain qua nhánh CARP.**
Feecover thêm nhánh chuyển CARP — có drain Vault không? → C-CM-1 (`non_magic_value_preserved`) đã ép **value non-MAGIC trong Vault bảo toàn byte-perfect**, CARP-fee KHÔNG rút từ Vault account mà là **input riêng của user** trả vào `FeecoverAccrual`. Feecover KHÔNG chạm value trong Vault. Kiểm tra: CARP-fee đến từ input ngoài Vault-account-script.

**C. Treasury / accrual drain.**
`epoch_settle`/`deposit` có thể bị lạm để rút CARP? → FEECOVER-SETTLE-1 (tuần tự epoch, không double-settle), FEECOVER-HO-1 (release qua Governance). `request_topup` chỉ **phát yêu cầu**, không tự kéo tài sản — L3 quyết. Double-satisfaction ở settle chặn bằng đếm-theo-epoch (`last_settled_epoch` monotonic) + `carp_accrued` reset-sau-settle.

**D. Ổn định phí dưới drift sức mua.**
Nếu `base_price` nền trôi mà `fee_magic` giữ nguyên → phí lệch sức mua thực. → Re-anchor quản trị (§4.2 lớp 2), đồng bộ nhịp `base_price`. **Rủi ro:** cadence quá chậm. Giảm thiểu: cân "ổn định user" vs "provider không lỗ". **Không** giải bằng oracle real-time (giữ INV-PEG-BY-DEMAND). Lựa chọn có chủ đích: ổn định danh nghĩa > bám giá tức thời.

**E. Gaming quy-đổi (khai `service_id` rẻ mà tiêu nhiều).**
→ `consume.ak` đọc `magic_consumed` THẬT từ beacon `PriceParam` (C-CM-2, không tin client). Feecover thêm `magic_consumed == fee_magic(service_id)` **và** `carp_paid == fee_magic` (FEECOVER-PAY-1) ⇒ không thể lệch service. **Còn hở:** đồng bộ `PriceParam.base_price[op]` với `ServiceFeeSchedule.fee_magic[service_id]` — phải khớp; lệch hai bảng → user tiêu MAGIC ≠ trả CARP. Chốt: cả hai cùng versioned qua Governance (§8).

**F. Giả mạo DID để lọt cổng.**
→ FEECOVER-GATE-3: DID đọc qua reference input, không mint/không tiêu ⇒ không tự-đúc DID. Point-in-time `did_active`/`key_authorized` (VeData §2.1) chống dùng khoá đã rotate/DID đã revoke. Rủi ro còn lại nằm ở an toàn của DID Registry + resolve API (ngoài Feecover).

**G. Va chạm phạm vi (Feecover lỡ chạm CARP-lõi/oracle).**
→ FEECOVER-HO-3 ép Feecover không mint/burn CARP/LAMP, không đọc oracle. INV-PEG-BY-DEMAND/INV-BURN-EXCL/`1 CARP=1 MAGIC` ở tầng CARP **không có input nào từ Feecover**. Kiểm tra: grep validator Feecover không import module CARP peg/oracle.

---

## 8. Phụ thuộc chéo (dependency section)

Feecover **tiêu thụ** hai hạ tầng của đội khác, KHÔNG tự dựng:

### 8.1 Mô hình on-chain MAGIC + `did_commit` per-DID (phụ thuộc MAGIC-team)

- **Hiện trạng:** `EngageDatum { owner, consumed_count, last_epoch }` (`ConsumeMAGIC/onchain/lib/magiclamp/consume/types.ak`) chỉ gắn `owner` (khoá), **CHƯA có `did_commit`** — tương đương `did_commit = #""` (sentinel rỗng).
- **Feecover cần:** MAGIC-team thêm trường `did_commit : ByteArray` (blake2b256 của DID) vào `EngageDatum`, ràng nó bảo toàn qua consume, để Feecover **attribution phí per-DID** (§5, §6.1 breakdown). Đây cũng là tiền điều kiện chống-farm của cả hệ MAGIC (Whitepaper §7 — thưởng-Gen keyed-MAGIC đọc tiêu **cross-DID** chặn tự-burn-vòng: "did_commit = #\"\" rỗng → blake2b256 thật + PhoenixKey-liveness", thuộc PhoenixKey backend giao Long).
- **Trước khi có:** Feecover chạy được ở mức `owner`-gating (attribution thô theo account), nâng lên per-DID khi dependency sẵn sàng.

### 8.2 Resolve API point-in-time (phụ thuộc PhoenixKey-resolve-API)

- Feecover gate (FEECOVER-GATE-1) dựa **`PhoenixKey-VeData-Contract v0.2.0 §2.1`**:
  - `did_active(did, query_epoch) → {active, revoked_at, never_existed}` — verify DID còn sống tại `current_epoch`.
  - `key_authorized(key, did, query_epoch) → bool` — verify khoá ký đã authorize cho DID **tại thời điểm đó** (Stamp §10 key-rotation rule), chống dùng khoá đã rotate.
- Semantics **point-in-time** bắt buộc (query state tại `query_epoch`, không state hiện tại — §2.1). Feecover dùng `current_epoch` của tx làm `query_epoch`.

### 8.3 Đồng bộ hai bảng giá

- `PriceParam.base_price[op]` (ConsumeMAGIC) và `ServiceFeeSchedule.fee_magic[service_id]` (Feecover) phải khớp cho mỗi cặp (op ↔ service). Cả hai versioned + gov-gated cùng cadence (Phản biện E). Chốt cơ chế đồng bộ ở §9-D6.

---

## 9. Quyết định chờ anh chốt

Đã bỏ các quyết định nay được mô hình đúng giải quyết (đơn vị MAGIC vs CARP, có dùng collectToTreasury không, `==`/`≥` cho MAGIC — nay `==` bắt buộc theo FEECOVER-PAY-1 neo 1:1). Chỉ giữ quyết định **thật sự chiến lược**:

| # | Quyết định | Phương án gợi ý | Ghi chú |
|---|---|---|---|
| D1 | **Mức phí từng dịch vụ** (`fee_magic` mỗi `service_id`) | Bảng §4.1 là minh hoạ; cần số thật | Neo sức mua qua `base_price` nền |
| D2 | **Cadence + biên re-anchor phí** | Mượn `base_price` CARP (±10%/quý + siêu-đa-số cho thay đổi lớn) HAY khung riêng PhoenixKey | Cân "ổn định user" vs "provider không lỗ" (Phản biện D) |
| D3 | **Dịch vụ nào Feecover phủ** | Khởi đầu: did.anchor, did.rotate, vc.verify, custody.op | Mỗi service_id thêm 1 entry |
| D4 | **Ranh giới tầng App vs Platform** | Ai là App (provider dịch vụ), ai là Platform (gom nhiều App)? | Ảnh hưởng cấu trúc `FeecoverAccrual` |
| D5 | **`stabilizer_ref` là ai + chữ ký interface L3** | RedPeg? tổng hợp GreenPeg/RedPeg? | Feecover chỉ định interface; L3 team implement |
| D6 | **Cơ chế đồng bộ `PriceParam` ↔ `ServiceFeeSchedule`** | Cùng 1 tx governance đổi cả hai, HAY ràng khớp on-chain | Chống lệch hai bảng (Phản biện E, §8.3) |

---

## 10. Tóm tắt cho reviewer

- Feecover = **lớp chính sách mỏng trên ConsumeMAGIC**, KHÔNG engine phí mới. 5/11 chức năng tái dùng nguyên trạng (bảng §1.3).
- **Mô hình MAGIC đúng (3-token):** MAGIC = **số dư account trong Vault** (đơn-vị-kế-toán, KHÔNG token, KHÔNG policy-id); CARP = **đơn vị thanh toán**; neo `1 CARP = 1 MAGIC`. Consume = giảm số dư account (`consume.ak`), KHÔNG BurnBatch/native-token.
- **Định giá bằng MAGIC, thanh toán bằng CARP:** `carp_paid == fee_magic` (neo 1:1). Phí gom vào Vault provider theo tầng App→Platform, **cuối epoch** đối soát chuyển **CARP về Phoenix Treasury** (KHÔNG LAMP/MAGIC). Đã **bỏ `collectToTreasury`**.
- Năm phần mới: `ServiceFeeSchedule`, `FeecoverGate` (DID point-in-time), `FeecoverAccrual` (gom per-tx), `FeecoverEpochSettle` (settle cuối epoch), quy-đổi-CARP.
- Ổn định phí = neo danh nghĩa nanogic **trên sức mua MAGIC** (zero-oracle, giữ INV-PEG-BY-DEMAND) + re-anchor quản trị đồng nhịp `base_price`. KHÔNG dựa "MAGIC non-transfer", KHÔNG bám giá ngoài real-time.
- Ổn-định-giá CARP (GreenPeg/RedPeg/oracle/buyback) = **NGOÀI PHẠM VI**; Feecover chỉ định hai mũi tên `deposit`/`request_topup` sang L3.
- KHÔNG chạm `consume.ak` (C-CM-1..5) và KHÔNG chạm bất biến CARP-lõi (INV-PEG-BY-DEMAND, INV-BURN-EXCL, `1 CARP=1 MAGIC`).
- **Phụ thuộc chéo (§8):** `did_commit` per-DID (MAGIC-team, hiện `#""`) + resolve API point-in-time (VeData §2.1 `did_active`/`key_authorized`). 6 quyết định chiến lược treo cho anh (§9).
