# PhoenixKey — Delegator→LAMP-claim: CORE layer (CIP-30 Lace connector + builder còn thiếu) — DEV-READY [N]

> **PHÂN LOẠI:** CORE (`rust_core` / SDK — Enclave, lớp crypto/tx-builder). PR giao
> **Tuân** (Core/crypto). Nếu có Core dev riêng cho tầng Cardano tx-builder →
> reassign cho người đó; Tuân giữ review interface FFI + crypto.
>
> **Mục tiêu spec này:** biến hai mảnh 🔴 của luồng Delegator→LAMP-claim thành
> **build-ready**: (1) CIP-30 Lace connector Hướng-1 (watch-only + request-sign),
> (2) hai builder còn thiếu `taad_build_legacy_migration_tx` +
> `taad_build_delegator_lamp_claim_tx`. KHÔNG phát minh cơ chế mới — hợp nhất
> `PhoenixKey-Connector-CIP30-Feat-Math.md` (Math xong, code thiếu) thành FFI ký
> hiệu + API surface cụ thể, mirror mẫu có sẵn trong `staking.rs` (#19) và
> `mint_lamp.rs`.
>
> **Đọc kèm (nguồn sự thật, KHÔNG chép lại — chỉ neo):**
> - `PhoenixKey-Connector-CIP30-Feat-Math.md` — bất biến I-CONN-1…7, ánh xạ method
>   ↔ khoá (B.3), `acct_xvk` watch-only (B.1). Spec này HOÀN THIỆN phần C.1 của nó.
> - `PhoenixKey-Legacy-Wallet-Migration-Feat.md` — A.2 tx bàn-giao 4-trong-1, A.4
>   tự-cấp-phí + Plomin, A.5 hai đích (LC1 phoenix / LC2 franken), A.7 I-MIGR-*.
> - `staking.rs` — `finalize_and_sign`, `funding_ctx`, `build_unspent_outputs`,
>   `derive_stake_xprv_account`, `WithdrawalsBuilder`, `CertificatesBuilder`,
>   `VoteDelegation`, `StakeDeregistration` (TÁI DÙNG, không viết lại).
> - `mint_lamp.rs` — mẫu reference-input (CIP-31) + `add_required_signer` +
>   redeemer PlutusData + `PlutusWitness` cho spend script (mirror cho §3).
> - `phoenix_address.rs` — `derive_phoenix_address(did, anchor_policy_hex, network)`
>   = đích Move (LC1) + đích nhận LAMP (§3).
> - `SuperApp-Reply-delegator-lace-claim-lamp.md` — bảng 7 bước, ai build cái gì.
> - `LAMP/Airdrop/onchain/lib/magiclamp/airdrop/ledger.ak` + `merkle.ak` — redeemer
>   `Claim{claimer, amount, proof}`, `ProofStep{is_left, hash}`, leaf/node hash,
>   spend-once slot burn. Đây là interface contract §3 phải khớp byte.

---

## §0. Phân loại + vị trí trong luồng 7 bước

**Tầng:** CORE — chỉ dựng + ký tx trong `rust_core`, expose FFI `taad_*`. KHÔNG đụng
backend endpoint (Long) + Merkle-root/ISPO (LAMP). Xem §6 ranh giới.

Luồng Delegator→LAMP-claim (từ `SuperApp-Reply-delegator-lace-claim-lamp.md`),
**tô đậm phần spec này chịu trách nhiệm:**

| Bước | Việc | Trạng thái | Chủ | Spec này? |
|---|---|---|---|---|
| 0 | Tạo DID + ví Phoenix (đích claim) | ✅ live | — | — |
| 1 | **Kết nối Lace, đọc stake/reward (CIP-30 watch-only)** | 🔴 code thiếu | **Core** | **§1** |
| 2 | Đọc stake-state của credential | 🔴 blocker [OPEN-4] | Long | tiêu thụ (input §2/§3) |
| 3 | Uỷ quyền thẩm quyền (Grant/Consent) | 🔴 thiếu | Long | tiêu thụ (không build) |
| 4 | Lấy Merkle-proof để claim LAMP | 🔴 thiếu | Long + LAMP | tiêu thụ (input §3) |
| 5 | **Dựng tx (migrate / claim)** | 🔴 2 builder thiếu | **Core** | **§2 + §3** |
| 6 | **Ký** — (a) CIP-30 Lace / (b) `/sign/request` controller | 🔴 (a) / ✅ (b) | **Core** / Long | **§1 + §4** |
| 7 | Submit + theo dõi | 🔴 thiếu | Long | tiêu thụ |

**Spec này giao 3 khối code:**
- **K1** — CIP-30 Lace connector (§1): `enable` + 6 method watch-only/sign, FFI +
  Dart/JS surface. Giải quyết [OPEN-1] signData (§1.5).
- **K2** — builder `taad_build_legacy_migration_tx` (§2): 4-trong-1 tự-cấp-phí.
- **K3** — builder `taad_build_delegator_lamp_claim_tx` (§3): spend Merkle claim
  validator → route LAMP về ví Phoenix.

---

## §1. CIP-30 Lace connector — PhoenixKey LÀ dApp (Hướng 1)

### §1.0 Ranh giới bản chất (từ SuperApp-Reply)

Cardano KHÔNG có account-abstraction → **Lace không trao được "chữ ký lâu dài"**.
Connector chỉ làm hai việc: **đọc công khai** (watch-only) và **xin ký từng lần**
(request-sign → Lace bật popup). Bí mật KHÔNG rời Lace (I-CONN-1/2). Mọi thứ đi qua
connector là dữ liệu công khai: address, UTxO CBOR, tx-chưa-ký CBOR, witness CBOR.

### §1.1 Kiến trúc 3 tầng — ai làm gì

CIP-30 `enable()` chỉ tồn tại trong **JS context của WebView** (không phải Rust,
không phải Dart native). Vì vậy connector là **cầu 3 tầng**, KHÔNG phải một hàm FFI
đơn:

```
┌─ Flutter UI (Dart) ─────────────────────────────────────────────┐
│  LaceConnector (Dart facade)  — API surface §1.3                 │
│    ├─ gọi JS bridge (InAppWebView) cho enable/get*/sign*         │
│    └─ gọi rust_core FFI cho: dựng tx (§2/§3), COSE build (§1.5), │
│         assemble witness (§1.4)                                  │
├─ JS bridge (chạy trong WebView, chèn 1 lần) ────────────────────┤
│  window.cardano.lace.enable() → api                             │
│  api.getUsedAddresses / getRewardAddresses / getUtxos           │
│  api.signTx(cbor, partial) / signData(addr, payload)            │
│  api.submitTx(cbor)                                             │
├─ rust_core (Rust, FFI `taad_*`) ────────────────────────────────┤
│  taad_build_legacy_migration_tx (§2)                            │
│  taad_build_delegator_lamp_claim_tx (§3)                        │
│  taad_assemble_witness (§1.4)  taad_cose_sign1_build (§1.5)      │
└─────────────────────────────────────────────────────────────────┘
```

**Vì sao chia thế:** Rust KHÔNG gọi được `window.cardano` (không có JS runtime
trong Enclave). Rust CHỈ nhận **kết quả** JS (address list, UTxO CBOR, witness
CBOR) làm tham số, và trả tx-chưa-ký CBOR để JS đẩy vào `signTx`. Đây là biên giới
sạch: **JS = kênh Lace, Rust = builder + crypto, Dart = điều phối.**

> Mobile (không có `window.cardano`): dùng CIP-45 pairing (Connector A.4). API
> surface §1.3 GIỮ NGUYÊN — chỉ đổi transport JS-bridge → relay/deep-link. Spec
> này đặc tả Phase-1 web/WebView; CIP-45 relay = SDK (Long, §6).

### §1.2 Watch-only vs trigger-Lace-popup (đường ranh chính xác)

| Method CIP-30 | Loại | Lace bật popup? | Ai xử lý | Trả về |
|---|---|---|---|---|
| `enable()` | cấp quyền | **CÓ** (1 lần duyệt phiên) | JS | api object |
| `getUsedAddresses()` | watch-only | KHÔNG | JS | `[addr_hex]` |
| `getRewardAddresses()` | watch-only | KHÔNG | JS | `[reward_addr_hex]` |
| `getUtxos(amount?)` | watch-only | KHÔNG | JS | `[utxo_cbor_hex]` |
| `getBalance()` | watch-only | KHÔNG | JS | `value_cbor_hex` |
| `getChangeAddress()` | watch-only | KHÔNG | JS | `addr_hex` |
| `getNetworkId()` | watch-only | KHÔNG | JS | `0|1` |
| `signTx(cbor, partial)` | **request-sign** | **CÓ** | JS→Lace | `witness_set_cbor_hex` |
| `signData(addr, payload)` | **request-sign** | **CÓ** | JS→Lace | `{signature, key}` (COSE) |
| `submitTx(cbor)` | broadcast | KHÔNG (có thể xin xác nhận) | JS→Lace | `tx_hash` |

**Bất biến K1-WATCH:** mọi `get*` KHÔNG bao giờ bật popup + KHÔNG trả bí mật. Chỉ
`signTx`/`signData` bật Lace (user nhập password/sinh-trắc CỦA LACE). Đây là toàn
bộ "uỷ quyền" khả dĩ — không có chế độ "Lace ký tự động mãi mãi" (I-CONN-1).

### §1.3 API surface — Dart facade `LaceConnector`

Ký hiệu Dart (facade UI gọi; JS bridge + FFI ẩn dưới):

```dart
class LaceConnector {
  /// enable(): user duyệt phiên trong Lace (1 popup). Trả API handle hoặc ném.
  /// walletKey = "lace" (hoặc "eternl", "nami" — cùng surface CIP-30).
  Future<LaceApi> enable(String walletKey);
}

class LaceApi {
  // ── watch-only (KHÔNG popup) ───────────────────────────────────
  Future<List<String>> getUsedAddresses();      // addr hex (CBOR-decoded ở Dart)
  Future<List<String>> getRewardAddresses();     // reward addr hex
  Future<List<String>> getUtxos({String? amountCborHex});  // UTxO CBOR hex list
  Future<String>       getChangeAddress();       // addr hex
  Future<int>          getNetworkId();           // 0 preprod/preview, 1 mainnet

  // ── request-sign (popup Lace) ──────────────────────────────────
  /// signTx: Lace ký tx-chưa-ký (CBOR đầy đủ tx, KHÔNG chỉ body).
  /// partial=true → Lace chỉ thêm witness của NÓ, không đòi ký hết (migration
  /// multi-witness). Trả witness_set CBOR hex.
  Future<String> signTx(String unsignedTxCborHex, {bool partial = false});

  /// signData (CIP-8/COSE_Sign1): ký payload theo addr. Trả COSE {signature,key}.
  Future<CoseSign1> signData(String addrHex, String payloadHex);

  // ── broadcast ──────────────────────────────────────────────────
  Future<String> submitTx(String signedTxCborHex);  // tx_hash
}

class CoseSign1 { final String signatureHex; final String keyHex; }
```

**Quy ước dữ liệu:** mọi address/UTxO/tx trao đổi với Lace là **CBOR hex** (chuẩn
CIP-30). Dart decode address CBOR→bech32 khi cần hiển thị; UTxO CBOR chuyển thẳng
xuống rust_core builder (§2/§3 nhận `utxos_json` — Dart chuyển CIP-30 UTxO CBOR →
JSON `{tx_hash,index,lovelace,assets}` mà builder đang ăn, cùng shape `StakeUtxo`).

### §1.4 FFI — hàm rust_core connector cần (assemble + convert)

Builder trả tx-chưa-ký CBOR; Lace trả witness_set CBOR; cần rust_core **ghép** lại
(giống `finalize_and_sign` nhưng witness đến TỪ NGOÀI thay vì tự ký):

```rust
// rust_core/src/connector.rs  (module mới)

/// Ghép tx-chưa-ký (body + placeholder witness rỗng) với witness_set CBOR mà
/// Lace trả qua signTx → tx đã-ký hoàn chỉnh, sẵn submitTx. Mirror bước
/// `Transaction::new(&body, &witnesses, aux)` của finalize_and_sign, nhưng
/// witnesses parse từ CBOR ngoài. Với partial=true: MERGE witness Lace vào
/// witness set đã có (controller/khác) thay vì thay thế.
pub fn assemble_witness(
    unsigned_tx_cbor_hex: &str,
    lace_witness_set_cbor_hex: &str,
    merge: bool,           // true = union vkeys (multi-witness); false = set
) -> String;               // signed tx CBOR hex, hoặc "" nếu lỗi

/// Chuyển UTxO CIP-30 (CBOR hex list, JSON array) → `utxos_json` shape mà
/// staking/migration builder ăn (`[{tx_hash,index,lovelace,assets:[{policy,name,
/// quantity}]}]`). Tách ở Rust để 1 nguồn parse (CSL TransactionUnspentOutput).
pub fn cip30_utxos_to_builder_json(cip30_utxo_cbor_hex_json: &str) -> String;
```

FFI:

```rust
#[no_mangle] pub unsafe extern "C" fn taad_assemble_witness(
    unsigned_tx_cbor_hex: *const c_char,
    lace_witness_set_cbor_hex: *const c_char,
    merge: u8,                       // 0 = set, 1 = merge/union
) -> *mut c_char;

#[no_mangle] pub unsafe extern "C" fn taad_cip30_utxos_to_builder_json(
    cip30_utxo_cbor_hex_json: *const c_char,
) -> *mut c_char;
```

Quy ước lỗi = staking.rs: thất bại → trả `null`/`""` (KHÔNG panic qua FFI).

### §1.5 signData [OPEN-1] — GỠ, không để mở

Build-readiness đánh dấu `signData` là gate [OPEN-1] (chưa có builder COSE). **Gỡ
dứt điểm ở đây theo I-CONN-6:**

- **CIP-30 `signData` = COSE_Sign1 (CIP-8), khoá Ed25519** — KHÔNG P-256. P-256 chỉ
  cho login-native PhoenixKey (luồng RIÊNG, Permission §7), KHÔNG phải method này.
  Dùng P-256 ở đây = phá interop với mọi ví/dApp Cardano.
- **Trong luồng Delegator→LAMP-claim, `signData` KHÔNG nằm trên đường tới-hạn.**
  Claim + migration đều đi qua `signTx` (Lace ký tx on-chain). `signData` chỉ cần
  khi PhoenixKey EXPOSE CIP-30 cho dApp ngoài (Hướng 2 — Phase 2, §6). → **Scope:**
  connector Phase-1 (spec này) triển khai `signData` ở tầng **passthrough**: Dart
  gọi thẳng `api.signData` của Lace (Lace tự dựng COSE_Sign1) và trả `CoseSign1`
  cho caller. **KHÔNG cần builder COSE trong rust_core cho Phase-1** — vì Lace là
  bên ký, Lace dựng COSE.
- Builder COSE trong rust_core (`taad_cose_sign1_build`, ký bằng **controller
  Ed25519**) CHỈ cần cho Hướng 2 (PhoenixKey là bên ký, dApp ngoài xin `signData`
  vào ví Phoenix). Đó là Phase-2, giao kèm consent-store (Long). **[OPEN-1] đóng
  cho luồng này**; mở lại chỉ khi làm Hướng 2.

**Chốt §1.5:** connector Phase-1 = `enable + 5 get* + signTx + signData(passthrough)
+ submitTx`. Builder COSE controller = Phase-2, không chặn Delegator claim.

### §1.6 Bất biến CIP-30 connector

- **CORE-CONN-1:** watch-only method KHÔNG bao giờ bật popup / trả bí mật (§1.2).
- **CORE-CONN-2:** chỉ `signTx`/`signData` khiến Lace ký; PhoenixKey không giữ khoá
  Lace (I-CONN-1). Không có "auto-sign vô hạn".
- **CORE-CONN-3:** Rust không gọi `window.cardano`; nhận address/UTxO/witness CBOR
  làm tham số. JS = kênh, Rust = builder+crypto, Dart = điều phối (§1.1).
- **CORE-CONN-4:** `signData` Phase-1 = passthrough Lace COSE_Sign1 (Ed25519).
  Builder COSE controller = Phase-2 (§1.5), KHÔNG dùng P-256 (I-CONN-6).

---

## §2. Builder `taad_build_legacy_migration_tx` — tx bàn-giao 4-trong-1

### §2.1 Việc làm (từ Legacy-Migration A.2)

MỘT tx, Lace ký MỘT lần (I-MIGR-ONESIG), gộp tối đa 4 việc theo Conway:

1. **VoteDelegation**(stake_cred cũ → DRep) — gỡ bẫy Plomin (I-MIGR-PLOMIN). Mặc
   định `abstain`; bỏ được nếu indexer báo stake_cred cũ ĐÃ vote-delegate.
2. **Withdrawal**(reward account cũ, R) — R thành input ngầm → **tự trả phí**
   (I-MIGR-SELFUND).
3. **Move** phần còn lại (R − fee − deposit) → **đích DID**:
   - LC1 `phoenix_addr(did)` = `derive_phoenix_address` (mặc định, khuyến nghị).
   - LC2 `franken_addr(did,k)` = payment cũ + stake did — CẢNH BÁO giam vốn.
4. **StakeDeregistration**(stake_cred cũ, tuỳ chọn) — hoàn 2 ADA deposit → thêm
   nhiên liệu.

### §2.2 Ai ký

**Lace (CIP-30 `signTx`)** — payment key + stake key CŨ đều trong Lace, một chữ ký
Lace phủ cả input-spend + withdrawal + cert. **KHÁC staking.rs**: staking.rs tự ký
bằng seed (PhoenixKey giữ khoá DID); migration KHÔNG có seed cũ (I-CONN-1) → builder
CHỈ dựng tx-chưa-ký, trả CBOR cho Lace ký. → **Builder trả UNSIGNED tx CBOR**, không
gọi `finalize_and_sign` (không có xprv để ký). Đây là điểm mấu chốt khác 5 builder
staking hiện có.

> Fee-sponsorship (reward < fee): tx multi-witness, một địa-chỉ-dịch-vụ thêm input
> ADA + ký phần của nó (`partial=true` assemble). CƠ CHẾ ở đây; CHÍNH SÁCH ai trả =
> ngoài phạm vi (LAMP/Treasury). Builder chỉ cần hỗ trợ output multi-witness.

### §2.3 Thuật toán

```
build_legacy_migration_tx(
    old_reward_addr_hex,     // reward addr ví cũ (từ getRewardAddresses)
    old_stake_cred_hex,      // stake credential ví cũ (28B keyhash hex)
    old_utxos_json,          // UTxO ví cũ (từ getUtxos → cip30_utxos_to_builder_json)
    reward_lovelace,         // R = reward chờ rút (từ stake-state §2, Long)
    already_vote_delegated,  // bool (từ stake-state) → bỏ VoteDelegation nếu true
    drep_id,                 // "abstain" mặc định (nếu cần vote-deleg)
    include_deregister,      // bool → có StakeDeregistration không
    dest_kind,               // "phoenix" (LC1) | "franken" (LC2)
    did,                     // DID đích
    anchor_policy_hex,       // taad policy (cho derive_phoenix_address)
    old_change_addr_hex,     // đích LC2 franken payment (nếu dest_kind=franken)
    protocol_params_json,
    network,
) -> unsigned_tx_cbor_hex (hoặc "" nếu bất khả):

 1. Parse + validate. Nếu old_utxos rỗng VÀ reward=0 → "" (không nhiên liệu).
 2. tb = build_tx_builder(params)                     // tái dùng taad_did.rs
 3. certs = CertificatesBuilder::new()
 4. NẾU !already_vote_delegated:                      // Plomin, ĐẶT TRƯỚC withdrawal
       certs.add(VoteDelegation(Credential::from_keyhash(old_stake_cred), parse_drep(drep_id)))
 5. NẾU include_deregister:
       certs.add(StakeDeregistration(old_stake_cred))  // hoàn 2 ADA (CSL implicit input)
 6. tb.set_certs_builder(certs)
 7. wb = WithdrawalsBuilder::new()
    wb.add(RewardAddress(old_reward_addr), reward_lovelace)   // R → implicit input
    tb.set_withdrawals_builder(wb)
 8. dest = dest_kind == "phoenix"
             ? derive_phoenix_address(did, anchor_policy_hex, network)   // LC1
             : franken_addr(old_change_addr payment_cred, stake_cred(did))// LC2
    // KHÔNG add_output cố định amount cho Move — để add_change_if_needed dồn
    // (R − fee − deposit_mới + deposit_hoàn) về dest. dest = change addr.
 9. tb.add_inputs_from(old_utxos, LargestFirst)       // gom UTxO cũ (nếu có)
10. tb.add_change_if_needed(&dest)                    // MOVE = change về đích DID
11. body = tb.build()
    // Tự-cấp-phí check (I-MIGR-SELFUND): build() FAIL nếu R + Σin < fee + deposit.
    // → trả "" + (Dart hiển thị "cần nạp thêm X ADA", A.6).
12. // KHÔNG ký. Dựng Transaction với witness_set RỖNG (placeholder) → CBOR hex.
    tx = Transaction::new(&body, &empty_witness_set, None)
    return hex(tx.to_bytes())
```

**Điểm khác biệt so với staking.rs cần chú ý khi code:**
- **KHÔNG `finalize_and_sign`** — không có seed cũ. Trả unsigned (bước 12).
- **change addr = đích DID** (khác staking: change về funding_addr). Đây là cách
  Move: dồn phần dư về ví Phoenix qua `add_change_if_needed`.
- **VoteDelegation dùng `old_stake_cred` truyền vào** (từ Lace/indexer), KHÔNG
  derive từ seed (không có seed). Ký sẽ do Lace lo qua signTx.
- **Reward addr từ Lace** (`getRewardAddresses`), không derive.

### §2.4 FFI

```rust
#[no_mangle] #[allow(clippy::too_many_arguments)]
pub unsafe extern "C" fn taad_build_legacy_migration_tx(
    old_reward_addr_hex: *const c_char,
    old_stake_cred_hex: *const c_char,
    old_utxos_json: *const c_char,
    reward_lovelace: u64,
    already_vote_delegated: u8,      // 0/1
    drep_id: *const c_char,          // "abstain" mặc định
    include_deregister: u8,          // 0/1
    dest_kind: *const c_char,        // "phoenix" | "franken"
    did: *const c_char,
    anchor_policy_hex: *const c_char,
    old_change_addr_hex: *const c_char,  // rỗng nếu dest_kind=phoenix
    protocol_params_json: *const c_char,
    network: u8,
) -> *mut c_char;                    // UNSIGNED tx CBOR hex; null nếu bất khả
```

Dart điều phối: `taad_build_legacy_migration_tx` → `api.signTx(cbor, partial=false)`
→ `taad_assemble_witness(cbor, witness, merge=0)` → `api.submitTx(signed)`.

### §2.5 Bất biến

- **CORE-MIGR-1:** builder trả **UNSIGNED** tx (Lace ký), KHÔNG bao giờ tự ký (không
  có seed cũ — I-CONN-1/I-MIGR-ONESIG).
- **CORE-MIGR-2:** VoteDelegation ĐẶT TRƯỚC Withdrawal trong cert/tx order (Plomin);
  bỏ chỉ khi `already_vote_delegated=1`.
- **CORE-MIGR-3:** Move = `add_change_if_needed(dest)` — dồn dư về đích DID; KHÔNG
  add_output amount cứng (fee/deposit do CSL cân).
- **CORE-MIGR-4:** reward < fee+deposit → `build()` FAIL → trả "" (KHÔNG đẩy tx
  chắc-FAIL cho Lace ký — I-MIGR-SELFUND).
- **CORE-MIGR-5:** LC1 dùng `derive_phoenix_address` (khớp vector aiken CLI); LC2
  cảnh báo giam vốn ở tầng UI (I-MIGR-TARGET).

---

## §3. Builder `taad_build_delegator_lamp_claim_tx` — spend Merkle claim → route LAMP về ví Phoenix

### §3.1 Cơ chế (từ LAMP-DISTRIBUTION-SPEC §6.1 ETD + airdrop_pool.ak)

ETD (Early TIGER Delegated) = claim **permissionless Merkle**. Validator
`airdrop_pool` với redeemer `Claim{claimer, amount, proof}`. Một claim tx PHẢI:

1. **Spend POOL UTxO** (mang POOL NFT) với redeemer `Claim` (address, amount, proof).
2. **Merkle-gated:** validator verify `verify_proof(merkle_root, claimer, amount,
   proof)`. Root ở datum POOL; proof + amount TỪ backend (§4 Long, ngoài Core).
3. **Spend-once (chống double-claim):** consume ĐÚNG claim-slot UTxO mang
   `(airdrop_nft_policy, name=leaf(claimer,amount))` ở địa chỉ `marker_dest`, VÀ
   **burn** NFT slot đó (`mint == -1`). Slot tiêu rồi → claim lần 2 không có input.
4. **POOL output:** trả 1 POOL UTxO mang POOL NFT, `value = pool_in − amount LAMP`,
   datum bảo toàn trừ `claimed_count += 1`.
5. **Claimer output:** `amount` LAMP về **`claimer` = ví Phoenix của DID**.

→ `claimer` PHẢI = `derive_phoenix_address(did, anchor_policy_hex, network)`. Đây là
điểm "route LAMP về ví Phoenix": leaf snapshot phải đã dùng đúng địa chỉ Phoenix làm
`address` (backend/LAMP dựng snapshot theo địa chỉ Phoenix của delegator — input,
không phải việc Core; Core CHỈ đảm bảo output khớp `claimer` trong redeemer).

### §3.2 Interface contract với validator (khớp byte — KHÔNG tự chế)

Từ `ledger.ak` + `merkle.ak` (LAMP):

```
Claim redeemer   = Constr 0 [ claimer: Address, amount: Int, proof: List<ProofStep> ]
ProofStep        = Constr 0 [ is_left: Bool, hash: ByteArray(32) ]
leaf   = blake2b_256( 0x00 ++ cbor.serialise(claimer_address) ++ amount_be_8 )
node   = blake2b_256( 0x01 ++ left ++ right )      // domain-separation prefix
```

- **amount encoding:** leaf dùng `amount_be_8` (big-endian 8-byte). Redeemer field
  `amount` là `Int` thường. Backend cấp proof đã tính theo leaf này; Core KHÔNG tự
  hash leaf (validator lo verify) — Core chỉ **đóng gói** `{claimer, amount, proof}`
  đúng PlutusData shape trên. **Nhưng** slot NFT name = `leaf` → Core PHẢI tính leaf
  để chọn/burn đúng slot UTxO (§3.3 bước 3). Dùng cùng `blake2b_256(0x00 ++
  serialise(addr) ++ amount_be_8)` — mirror merkle.ak `leaf_hash`.
- **AirdropNftRedeemer::BurnSlot** (name cũ MintClaim, ngữ nghĩa burn): `mint ==
  -1`, có POOL input, không đụng POOL NFT. Core dựng mint entry `-1` cho unit
  `(airdrop_nft_policy, leaf)`.

### §3.3 Thuật toán (mirror mint_lamp.rs cho phần script-spend + reference/mint)

```
build_delegator_lamp_claim_tx(
    did, anchor_policy_hex,          // → claimer = phoenix addr
    amount_lamp,                     // từ backend §4
    proof_json,                      // [{is_left:bool, hash:hex32}] từ backend §4
    pool_utxo_json,                  // POOL UTxO (spend): {tx_hash,index,value,datum}
    slot_utxo_json,                  // claim-slot UTxO {tx_hash,index} (name=leaf)
    airdrop_pool_script_hex,         // validator CBOR (spend witness)
    airdrop_nft_policy_script_hex,   // mint policy CBOR (burn witness)
    lamp_policy_hex, lamp_name_hex,  // LAMP asset unit (đếm amount)
    fee_utxos_json,                  // UTxO trả phí + collateral (ví Phoenix/Lace)
    fee_signer_seed_hex_OR_lace,     // xem §3.4 ai ký
    protocol_params_json, network,
) -> tx_cbor_hex (signed nếu controller ký; unsigned nếu Lace ký):

 1. claimer = derive_phoenix_address(did, anchor_policy_hex, network)   // §phoenix
 2. leaf = blake2b_256(0x00 ++ serialise(Address(claimer)) ++ amount_be_8(amount_lamp))
 3. tb = build_tx_builder(params)
 4. // SPEND POOL UTxO với redeemer Claim (PlutusWitness — mirror mint_lamp §5/§6)
    claim_redeemer = PlutusData::Constr(0, [ Address(claimer), Int(amount), proof ])
    tb.add_plutus_script_input( pool_utxo, airdrop_pool_script, claim_redeemer )
 5. // SPEND slot UTxO (name=leaf) — cần cho spend-once; validator marker cho tiêu
    tb.add_input(slot_utxo)  // (redeemer marker nếu marker validator gắn, xem note)
 6. // BURN slot NFT: mint -1 cho (airdrop_nft_policy, leaf), redeemer BurnSlot
    mint_builder.add( airdrop_nft_policy_script, name=leaf, qty=-1, redeemer=BurnSlot )
 7. // POOL output: value = pool_in − amount LAMP, datum claimed_count+1
    pool_out = TransactionOutput(pool_addr, pool_value − amount_lamp_of_LAMP)
    pool_out.set_datum(InlineDatum( AirdropPool{..same.., claimed_count+1} ))
    tb.add_output(pool_out)
 8. // Claimer output: amount LAMP về ví Phoenix (≥ amount cho phép gộp min-ADA)
    claimer_out = TransactionOutput(claimer, Value{min_ada, LAMP=amount_lamp})
    tb.add_output(claimer_out)
 9. tb.add_inputs_from(fee_utxos, LargestFirst)   // trả phí + min-ADA
10. tb.set_collateral(...)                         // thuần-ADA (script tx cần collateral)
11. tb.calc_script_data_hash(cost_models)          // Plutus tx cần script_data_hash
12. tb.add_change_if_needed(&fee_change_addr)
    // ExUnits TĨNH → caller evaluate-then-patch (giống mint_lamp note §fee)
13. finalize_and_sign HOẶC trả unsigned — xem §3.4
```

> **Note spend-once:** `airdrop_pool.ak` ép consume slot ở `marker_dest` + burn.
> `marker`/`nft` validator (LAMP) quy định redeemer chính xác cho slot spend + burn.
> Core PHẢI khớp `AirdropNftRedeemer::BurnSlot` (mint −1). Nếu slot ở địa chỉ script
> `airdrop_marker` cần redeemer riêng để tiêu → lấy shape từ `airdrop_marker.ak`
> (interface LAMP; nếu LAMP đổi → cập nhật 1 chỗ như `lamp_mint_redeemer` const của
> mint_lamp.rs). **Đây là interface contract với LAMP — §6 giao LAMP xác nhận shape
> cuối trước khi freeze builder.**

### §3.4 Ai ký

Claim tx spend POOL (permissionless — validator KHÔNG đòi chữ ký claimer, chỉ đòi
proof đúng + output đúng claimer). Nhưng tx cần **input trả phí + collateral** →
input đó cần chữ ký của CHỦ nó:

- **Trường hợp A — phí từ ví Phoenix (đã migrate):** input phí là UTxO ở
  `phoenix_addr` (script did_payment) → cần **controller ký** (`/sign/request`,
  Long) + reference-input anchor (mirror mint_lamp reference-input). → **Core
  `finalize_and_sign` với controller key** (giống mint_lamp) HOẶC trả unsigned cho
  `/sign/request`. Vì controller key = Master_KEK-derived (PhoenixKey giữ), Core ký
  được → trả **signed**.
- **Trường hợp B — phí từ Lace (chưa migrate, delegator còn dùng ví cũ):** input
  phí là UTxO ví Lace → **Lace ký** phần đó (`signTx partial=true`), Core trả
  **unsigned**, Dart cho Lace ký, `assemble_witness(merge=1)`.

→ Builder nhận cờ `sign_mode ∈ {controller, lace}`. `controller` → ký trong Core
(cần `master_kek_hex` + `anchor_utxo_json` reference-input như mint_lamp).
`lace` → trả unsigned, merge witness sau.

**Khuyến nghị mặc định:** phí từ ví Phoenix + controller ký (Trường hợp A) — sạch,
không cần Lace mỗi lần claim; hợp mục tiêu "sau migrate không cần ví cũ".

### §3.5 FFI

```rust
#[no_mangle] #[allow(clippy::too_many_arguments)]
pub unsafe extern "C" fn taad_build_delegator_lamp_claim_tx(
    did: *const c_char,
    anchor_policy_hex: *const c_char,
    amount_lamp: u64,
    proof_json: *const c_char,               // [{"is_left":true,"hash":"..32B hex.."}]
    pool_utxo_json: *const c_char,           // {tx_hash,index,value,datum_cbor,address}
    slot_utxo_json: *const c_char,           // {tx_hash,index}
    airdrop_pool_script_hex: *const c_char,
    airdrop_nft_policy_script_hex: *const c_char,
    lamp_policy_hex: *const c_char,
    lamp_name_hex: *const c_char,
    fee_utxos_json: *const c_char,
    sign_mode: *const c_char,                // "controller" | "lace"
    master_kek_hex: *const c_char,           // dùng nếu controller; rỗng nếu lace
    anchor_utxo_json: *const c_char,         // reference-input anchor (controller mode)
    protocol_params_json: *const c_char,
    network: u8,
) -> *mut c_char;   // signed (controller) hoặc unsigned (lace) tx CBOR hex; null lỗi
```

### §3.6 Bất biến

- **CORE-CLAIM-1:** claimer output = `derive_phoenix_address(did,...)` (byte-khớp
  vector aiken); LAMP route về đúng ví Phoenix (không địa chỉ khác).
- **CORE-CLAIM-2:** redeemer `Claim{claimer,amount,proof}` + `ProofStep{is_left,
  hash}` + leaf/node hash khớp `merkle.ak` (0x00 leaf / 0x01 node prefix). Core
  đóng gói, KHÔNG tự verify proof (validator lo).
- **CORE-CLAIM-3:** burn đúng slot NFT `(airdrop_nft_policy, leaf)` qty −1
  (spend-once); thiếu → double-claim, validator fail.
- **CORE-CLAIM-4:** POOL output bảo toàn datum trừ `claimed_count+1`, value =
  `pool_in − amount LAMP`.
- **CORE-CLAIM-5:** ExUnits/fee TĨNH → evaluate-then-patch (mirror mint_lamp);
  collateral thuần-ADA.
- **CORE-CLAIM-6:** controller mode → reference-input anchor + `add_required_signer
  (controller_pkh)` (mirror mint_lamp); Core ký. lace mode → unsigned + merge.

---

## §4. Signing model — tổng hợp per builder

| Builder | Ai ký input-spend | Ai ký cert/withdraw | Reference-input? | Core trả |
|---|---|---|---|---|
| `taad_build_legacy_migration_tx` (§2) | **Lace** (payment cũ) | **Lace** (stake cũ) | không | **UNSIGNED** |
| `taad_build_delegator_lamp_claim_tx` A (§3, controller) | **controller** (Core) | — (permissionless) | **CÓ** (anchor) | **SIGNED** |
| `taad_build_delegator_lamp_claim_tx` B (§3, lace) | **Lace** (phí) | — | không | **UNSIGNED** |
| staking.rs (5 builder, #19) | seed (Core) | seed (Core) | không | SIGNED |

**Nguyên tắc phân biệt (CORE-SIGN-*):**
- **CORE-SIGN-1:** input từ **ví Lace/seed-cũ** ⇒ Lace ký (CIP-30 signTx), Core trả
  UNSIGNED (I-CONN-1 — Core không có khoá đó).
- **CORE-SIGN-2:** input từ **ví Phoenix (did_payment script)** ⇒ controller ký
  (Master_KEK-derived, Core ký được) + reference-input anchor (đọc authority động,
  mirror mint_lamp). Core trả SIGNED.
- **CORE-SIGN-3:** input từ **base-addr seed DID** (staking.rs) ⇒ seed ký trong
  Core, trả SIGNED.
- **CORE-SIGN-4:** tx đa-nguồn (migration fee-sponsorship / claim mode-B) ⇒
  multi-witness: Core dựng UNSIGNED, mỗi bên ký phần mình, `assemble_witness
  (merge=1)` hợp nhất (§1.4).

---

## §5. Test plan

### §5.1 Rust unit (offline, không cần node — mirror staking.rs `#[cfg(test)]`)

**Connector §1.4:**
- `assemble_witness_merges_lace_witness` — dựng unsigned tx (body + empty ws),
  witness_set giả (1 vkey), assemble merge=0 → tx decode, `vkeys().len()==1`.
- `assemble_witness_union` — 2 witness set (controller + lace) merge=1 → union vkeys.
- `cip30_utxos_to_builder_json_roundtrip` — UTxO CBOR CIP-30 → JSON builder →
  `StakeUtxo` parse lại khớp (tx_hash/index/lovelace/assets).

**Migration §2:**
- `migration_builds_unsigned_with_vote_withdraw_move` — dựng tx: assert có 1
  VoteDelegation + 1 withdrawal + change output tại `phoenix_addr(did)`; witness set
  RỖNG (chưa ký — CORE-MIGR-1).
- `migration_skips_vote_when_already_delegated` — `already_vote_delegated=1` → 0
  VoteDelegation cert.
- `migration_dereg_refunds_deposit` — `include_deregister=1` → 1 StakeDeregistration;
  value conservation: Σout+fee = R+Σin+2ADA.
- `migration_insufficient_reward_returns_empty` — R < fee → "" (CORE-MIGR-4).
- `migration_dest_phoenix_matches_vector` — dest addr == `derive_phoenix_address`
  vector (`did:phoenix:person:alice`, policy a1..a1) preprod.

**Claim §3:**
- `claim_leaf_matches_merkle_ak` — `leaf(claimer, amount)` == vector từ merkle.ak
  test (0x00 prefix, amount_be_8). Đối chiếu 1 vector aiken.
- `claim_redeemer_shape` — decode redeemer PlutusData: Constr 0 [Address, Int,
  List[Constr 0 [Bool, Bytes32]]] khớp `Claim`.
- `claim_output_routes_to_phoenix` — claimer output address == phoenix addr, LAMP
  quantity == amount.
- `claim_burns_slot_nft` — mint entry `(airdrop_nft_policy, leaf) == -1`.
- `claim_pool_output_conserves` — POOL out value = pool_in − amount LAMP;
  claimed_count+1.
- `claim_controller_mode_has_reference_input` — mode controller → 1 reference input
  == anchor outpoint + required_signer == controller_pkh (mirror mint_lamp test).

### §5.2 Preview e2e (field test — evidence output thật, mandate anh Aladin)

Cần: 1 ví Lace test (preprod/preview) có reward tích + đã delegate; DID PhoenixKey
Active; POOL UTxO ETD live Preview + Merkle-proof từ backend (hoặc snapshot tay).

1. **Migration:** `enable()` Lace test → `getRewardAddresses/getUtxos` → build
   migration (dest=phoenix) → Lace `signTx` → `assemble` → `submitTx`. **Assert
   on-chain:** reward account = 0 sau tx; UTxO xuất hiện ở `phoenix_addr(did)`;
   stake_cred cũ vote-delegated (Plomin gỡ). Ghi txid.
2. **Claim:** lấy Merkle-proof cho `phoenix_addr` từ backend §4 → build claim
   (mode=controller, phí ví Phoenix) → Core ký → submit. **Assert on-chain:**
   `amount` LAMP tại `phoenix_addr`; POOL `claimed_count+1`; slot NFT burned;
   claim lần 2 cùng leaf → FAIL (spend-once). Ghi txid + LAMP balance.

**Gate PASS:** cả 2 txid confirmed + assert khớp (KHÔNG chỉ compile). Nếu backend
§4 (proof) chưa sẵn → dựng snapshot 1-leaf tay để chạy claim e2e, ghi rõ là mock.

---

## §6. Ranh giới — cái gì THUỘC spec này, cái gì KHÔNG

**THUỘC CORE (spec này, PR Tuân):**
- CIP-30 Lace connector Phase-1: `enable` + get* watch-only + signTx +
  signData(passthrough) + submitTx; `assemble_witness`, `cip30_utxos_to_builder_json`
  (§1).
- Builder `taad_build_legacy_migration_tx` (§2), `taad_build_delegator_lamp_claim_tx`
  (§3) + FFI + Rust unit test.

**KHÔNG thuộc (chỉ tiêu thụ input / tham chiếu):**
- **Backend (Long, spec riêng):** `GET /stake-state/{cred}` (§2 blocker [OPEN-4]),
  Consent/Grant 4 endpoint (§3 bước 3), `GET /airdrop-claim/.../merkle-proof` (§4),
  `POST /claim/submit` + monitor (§7), CIP-45 relay cho mobile, fee-sponsorship
  endpoint (nếu bật). Core NHẬN proof/stake-state/params làm tham số.
- **LAMP (đội LAMP):** Merkle-root snapshot + luật ETD/ISPO/airdrop; **shape cuối
  của `AirdropNftRedeemer::BurnSlot` + marker redeemer** (§3.3 note) — LAMP xác nhận
  trước khi Core freeze builder. Merkle snapshot PHẢI dùng `phoenix_addr` làm
  `address` trong leaf (để LAMP route về ví Phoenix).
- **Hướng 2 CIP-30 expose (Phase-2):** builder COSE controller `taad_cose_sign1_build`
  + predicate engine + consent-store — KHÔNG chặn Delegator claim ([OPEN-1] đóng cho
  luồng này, §1.5).

**4 trục quyết định (mandate):**
- *Dài hạn:* cùng FFI `taad_*` shape mọi team Cardano lái được; connector là cầu
  giao thức mở, không cạnh tranh ví (Connector A.1).
- *First-principles:* "uỷ quyền" = Lace-ký-từng-lần (CIP-30) HOẶC migrate-rồi-
  controller-ký; KHÔNG có trao-khoá-lâu-dài (Cardano không có AA).
- *Tối ưu:* migration gộp 4 việc/1 tx/1 fee/1 chữ ký (Conway); claim self-fund từ ví
  Phoenix (controller ký, không cần Lace mỗi lần). Reference-input = read-only oracle
  (mint_lamp), không lock lại anchor.
- *User + bền vững:* bất khả (reward<fee / ví cũ mất) → trả "" + báo rõ, KHÔNG đẩy
  tx chắc-FAIL đốt phí; bí mật không rời thiết bị (I-CONN-1).
