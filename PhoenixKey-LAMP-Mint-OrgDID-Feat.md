# PhoenixKey — LAMP Mint gated bằng OrgDID (OFFCHAIN Feat)

> DEV-READY spec, giao **Long** (offchain). On-chain policy + Rust builder ĐÃ SẴN — không xây lại.
> Ngày: 2026-07-02. Register: terminal. Type/field: tiếng Anh, JSON snake_case.

---

## ⚠ v2-REGISTRY UPDATE (2026-07-07) — ĐÈ phần authorization/policy/recipient của thân bài

> **Điều tra 07-07 (đối chiếu code thật):** thân bài dưới viết theo **v1-anchor** (LAMP đọc anchor TAAD → `controller_pkh` + status Active) + **apply-param per-OrgDID** + **mint ra ví**. **CODE THẬT đã chuyển sang v2-registry** — 3 điểm sau ĐÈ thân bài. Nguồn: `LAMP/Genesis/onchain/validators/{lamp_mint.ak, registry.ak}` (branch `feat/lamp-mint-compose-anchor-cap`, HEAD `6fbb6db`). Header `registry.ak`: *"THAY mô hình v1 (đọc controller_pkh trực tiếp từ anchor TAAD). LAMP KHÔNG còn đọc anchor; nó đọc REGISTRY NFT"*.

**(V2-1) Authorization: Registry NFT ref-input, KHÔNG đọc anchor TAAD.**
LAMP mint gate bằng cách đọc **Registry NFT** (`RegistryDatum{ token_tag → Authority }`) qua reference input, gọi `registry.validate_mint(policy_id, token_name, registry_nft_policy, registry_nft_name, token_tag, tx)`. `Authority` = `SinglePkh{pkh}` | `MultiSig{pkhs, threshold}` | `Revoked` (→ fail). ĐÈ §4 (a)(b)(c) [anchor + controller_pkh + Active]. → LAMP không còn đọc anchor; ràng buộc "OrgDID Active" dời sang **write-side validator của Registry** (ai được tạo/sửa entry). Rotate khoá = **sửa entry registry** (không redeploy policy).

**(V2-2) 1 POLICY-ID LAMP CHUNG — KHÔNG apply-param per-OrgDID.** ĐÈ §0 dòng 16-18 + §5 toàn bộ.
`lamp_mint` apply-param theo **token** (`token_name`/`dist_cap`/`token_tag`) → policy-id khác nhau **giữa các token** (tLAMP/LAMP/FARM), NHƯNG **KHÔNG khác giữa các OrgDID**. 12 param của `lamp_mint` KHÔNG có `anchor_nft_name`/`anchor_nft_policy` (tiền đề §5 sai từ gốc) — có `registry_nft_policy` + `registry_nft_name` + `token_tag`. Phân biệt org xảy ra **runtime** qua Registry NFT `name = blake2b_256(governing_did)`. → 1 policy-id LAMP tĩnh lưu ở backend (bỏ cột `org_dids.lamp_mint_policy_cbor` per-org §5); mỗi OrgDID phải có **entry `token_tag:"LAMP"`** trong Registry của mình TRƯỚC khi mint (Core dựng qua `registry_mint` deploy+update, controller-DID-ký). Khớp memory hệ: 1-policy-chung, per-org qua ref-input runtime.

**(V2-3) Mint → KHO `dist_treasury`, KHÔNG ra ví.** ĐÈ §1 dòng 140 + §6 (recipient). `lamp_mint.ak:199` ép **A-DEST: toàn bộ Δ rót vào KHO** (`dist_treasury`); ví user nhận LAMP ở **bước vesting-release riêng** (không thuộc tx mint này). → bỏ `recipient_address` khỏi endpoint mint; T1/T2 kỳ vọng LAMP xuất hiện ở **địa chỉ KHO**, không phải ví recipient.

**(V2-4) m-of-n THÁO được (v2 giải blocker v1).** ĐÈ §5 dependency (dòng 280) + LMINT-8. Threshold OrgDID → entry `MultiSig{pkhs, M}` trong Registry; gom M chữ ký member (mỗi member 1 pkh trong `MultiSig.pkhs`). KHÔNG cần native-multisig/aggregated-key/m-anchor như §5 lo. m-of-n offchain (§3 gom chữ ký) giữ nguyên; on-chain map thẳng vào `MultiSig`.

**GIỮ NGUYÊN (thân bài đúng):** kiến trúc offchain (mobile build+ký Master_KEK-on-device, backend relay+submit), intent `LAMP_MINT` (§2), gom m-of-n offchain (§3), SupplyState cap 36B compose (§1.5 — cap ĐÚNG, chỉ redeemer/route theo branch), fail-fast (§4 backend gate rẻ).

**🔴 #8 CHỈ ĐÓNG ĐƯỢC SAU KHI LAMP team (blocker liên-repo — xem message `_msg-to-LAMP-registry-orgdid-mint.md`):**
1. **Merge branch `feat/lamp-mint-compose-anchor-cap` → main** (hiện `lamp_mint.ak` v2 CHƯA có trên main).
2. **Cấp write-side Registry validator** (tạo/sửa Registry UTxO) + CBOR + hash — LAMP repo hiện chỉ có `registry.ak` read-side.
3. **Cấp KHO `dist_treasury` + `kho_nft`** trên Preview (địa chỉ, policy+name, authority) + deploy 4 script theo runbook.
4. Chốt tên `token_tag` (code) vs `action_tag` (`DID-Authorization-Registry-DRAFT`) trước khi Core dựng builder.

---

## §0 — PHÂN LOẠI + quyết định + vấn đề

**PHÂN LOẠI:** OFFCHAIN → PR giao **Long** (`PhoenixKey-Database`, backend Java).
Ranh giới: Claude/Agent KHÔNG tự sửa backend — spec này là bản giao việc để Long implement + PR.

**Vấn đề (1 dòng):** hôm nay **không có endpoint offchain nào mint LAMP ký bằng OrgDID** — catalog đã xác nhận (`PhoenixKey-API-Catalog.md` liệt kê `/identity/org/create` nhưng không có `mint-lamp`); `/activation` chỉ phát 1001 LAMP ban đầu qua **faucet** (`lamp_policy`), KHÔNG phải mint-gated-OrgDID. On-chain đã đủ (`build_mint_lamp_via_did` có test, `lamp_mint.ak` là policy) — chỉ thiếu lớp offchain nối chúng lại.

**QUYẾT ĐỊNH ĐÃ CHỐT (anh Aladin, 2026-07-02):** ⚠ *"per-OrgDID" + "applied per-OrgDID" ĐÃ ĐÈ bởi V2-2 (1 policy-id chung, gate qua Registry NFT). Đọc phần v2-UPDATE ở đầu.*
- **`lamp_mint` là policy mint LAMP CANONICAL** (~~OrgDID-gated per-OrgDID~~ → **1 policy chung, gate qua Registry NFT ref-input**, V2-1/V2-2). LAMP cố định 36 tỷ, KHÔNG burn — mọi mint đi qua đường này.
- **`lamp_policy` (faucet) chỉ là interim/testnet**, dùng cho `/activation` để phát LAMP kích hoạt ban đầu. KHÔNG canonical.
- ~~Config `LAMP_POLICY_CBOR_HEX` = applied `lamp_mint` per-OrgDID~~ → **1 policy-id LAMP tĩnh** (apply-param theo token, không theo OrgDID); org phân biệt runtime qua Registry NFT (V2-2, ĐÈ §5).

**Đã có vs còn thiếu:**

| Phần | Trạng thái | Chủ |
|---|---|---|
| Policy on-chain `lamp_mint` (12 tham số, per-OrgDID) — `LAMP/Genesis/onchain/validators/lamp_mint.ak` | ✅ có | LAMP repo |
| Builder Rust `build_mint_lamp_via_did()` — `PhoenixKey-Core/Enclave/rust_core/src/mint_lamp.rs`, có test | ✅ có | Core |
| Intent `LAMP_MINT` + 2 endpoint `/mint-lamp` + `/mint-lamp/submit-tx` | ❌ thiếu | **Long** |
| Mở rộng `/sign/request` gom **m-of-n** chữ ký | ❌ thiếu | **Long** |
| Config applied-`lamp_mint`-per-OrgDID + cách lưu | ❌ thiếu | **Long** (+ dependency, xem §5) |

---

## §1 — Kiến trúc

**Nguyên tắc gốc — Master_KEK KHÔNG rời thiết bị.** Builder Rust `build_mint_lamp_via_did` cần `master_kek_hex` để derive controller key. Master_KEK sống trong Secure Enclave của **mobile controller**, KHÔNG có trên backend. Vì vậy — đúng mẫu `/activation/submit-tx` đã chạy — **tx được build + ký ở mobile Enclave**, backend chỉ **relay intent** và **submit CBOR đã ký**. Backend KHÔNG gọi builder Rust server-side (nó không có Master_KEK, và không được phép có).

**Tái dùng `/sign/request`** với **intent type mới `LAMP_MINT`** (đúng mẫu `SEED_EXPORT` đang chạy trong `SignRequestServiceImpl`) — KHÔNG dựng flow relay mới toanh. Hai endpoint mới, đặt trong `OrgController` (`/identity/org`):

### Endpoint 1 — khởi tạo yêu cầu mint

```
POST /identity/org/{orgDid}/mint-lamp
Authorization: Bearer <session_token>      # session của OrgDID controller/member (§4)
Content-Type: application/json
```

Request body:
```json
{
  "amount": 1000000,
  "recipient_address": "addr_test1qz...",   // optional; rỗng/absent → self-custodial (ví controller)
  "session_id": "sess_web_abc"              // SSE channel để nhận "signed"
}
```

Xử lý:
1. Verify `session_token` → resolve caller PersonDID; check caller là member/controller của `orgDid` (§4).
2. Load OrgDID (`org_dids` + nếu threshold thì `org_authority_members`) → xác định `authority_model`, `threshold`, member set.
3. Dựng `SignIntent{ type:"LAMP_MINT", body:{org_did,amount,recipient}, ... }` (§2).
4. Gọi nội bộ `SignRequestService.create(...)` → tạo request trong Redis + push notification mobile.
5. Nếu `authority_model=threshold` → khởi tạo bộ đếm chữ ký (§3).

Response (`DataResponse<T>`):
```json
{
  "code": 1000,
  "message": "LAMP mint sign request created",
  "result": {
    "request_id": "0192...uuidv7",
    "expires_at": 1751462400,
    "authority_model": "single",
    "threshold": 1,
    "signatures_required": 1
  }
}
```
Web sau đó lắng nghe SSE event **`signed`** (đã có sẵn trong relay) trên `session_id`.

### Endpoint 2 — submit tx đã ký

```
POST /identity/org/{orgDid}/mint-lamp/submit-tx
Authorization: Bearer <session_token>
Content-Type: application/json
```

Request body:
```json
{
  "request_id": "0192...uuidv7",
  "signed_tx_cbor": "84a500..."     // mobile Enclave build+ký qua build_mint_lamp_via_did
}
```

Xử lý:
1. Load sign request theo `request_id` → phải `status=approved` (single) hoặc đã đạt `threshold` (m-of-n, §3).
2. Verify request thuộc `orgDid` này (intent.body.org_did khớp path).
3. Verify `signed_tx_cbor` khớp intent đã duyệt (§4 — backend check gì / validator check gì).
4. (Phòng thủ, §1.5) Đối chiếu tx **compose SupplyState**: tx PHẢI spend SupplyState NFT UTxO (redeemer `Advance`) + recreate SupplyState' + `Σ minted ≤ remaining_cap`. Nếu tx không đụng SupplyState → reject `LAMP_MINT_NO_SUPPLY_STATE` (không tốn phí submit một tx chắc-chắn-reject). Đây là fail-fast; validator SupplyState là cửa chốt cap (LMINT-8).
5. Submit CBOR qua `BlockfrostHttpClient.submitSignedTx(...)` (đúng như `/activation/submit-tx`).
6. Ghi `activity_log` (`flow=lamp_mint`, org_did, amount, route, tx_hash).

Response:
```json
{
  "code": 1000,
  "message": "LAMP mint tx submitted",
  "result": { "tx_hash": "abc123..." }
}
```

**Về anchor UTxO + fetch:** trong bản gốc §1 của brief có ghi backend "fetch anchor UTxO → gọi build_mint_lamp_via_did". Đã điều chỉnh theo code thật: **mobile fetch anchor UTxO + SupplyState UTxO + wallet UTxO + protocol params và build+ký** (nó có Master_KEK; đúng như mobile build tx activation hôm nay). Backend chỉ submit. Nếu tương lai muốn backend build hộ (server-signed variant) thì cần một controller key ký uỷ quyền tách khỏi Master_KEK — ngoài phạm vi PR này, ghi làm open item (§7).

Backend **có thể** fetch anchor UTxO + SupplyState UTxO để **đối chiếu** (verify tx tham chiếu đúng anchor của orgDid + spend đúng SupplyState) — dùng `orgDid → TAAD script address` + SupplyState script address rồi Blockfrost query. Đây là check phòng-thủ, KHÔNG phải để build (§4, §1.5).

---

## §1.5 — Compose SupplyState — ép trần 36 tỷ (no-burn)

**Bối cảnh (refinement LAMP team, đã đối chiếu code thật):** `lamp_mint` (canonical, neo OrgDID qua anchor NFT CIP-31) CHỈ gate **ai được mint + tên token**, KHÔNG tự ép trần. **Trần 36 tỷ no-burn nằm ở tầng SupplyState** — một tx mint hợp lệ PHẢI **compose** hai luật: mint qua `lamp_mint` + **đồng thời spend SupplyState NFT UTxO**. Thiếu SupplyState = không có gì chặn cap → **hở** (đây là lỗ brief gốc bỏ sót).

**Sự thật chính xác từ code (`LAMP/Genesis/offchain/src/mintBuilder.ts` = CONTRACT §5, `supplyState.ts`, `types.ts`, `constants.ts`) — KHÔNG viết từ trí nhớ:**

| Thành phần | Giá trị đúng | Ghi chú (đính chính refinement) |
|---|---|---|
| SupplyState **spend** redeemer | **`Advance`** = Constr(0,[]) | KHÔNG phải `DistributionVest`. Refinement nhầm chỗ. |
| `lamp_mint` **mint route** redeemer | **`DistributionVest`** = Constr(0,[]) **hoặc** `ReserveDraw` = Constr(1,[]) | Đây là redeemer của **policy mint** (chọn quota), KHÔNG phải redeemer spend SupplyState. |
| Cấu trúc cap | 2 quota: `dist_cap = 26,370 tỷ`, `reserve_cap = 9,630 tỷ` (oil) | Tổng = `TOTAL_CAP_OIL = 36,000,000,000,000,000` (36 tỷ LAMP × 10⁶). BẤT BIẾN. |
| SupplyState datum | `{dist_minted, reserve_minted, dist_cap, reserve_cap}` = Constr(0,[int×4]) | Bộ đếm đơn điệu; `minted_total = dist_minted + reserve_minted`. |
| oil | `OIL_PER_LAMP = 10⁶` | 1 LAMP = 10⁶ oil. `amount` trong intent tính bằng **oil**. |
| Token name (param) | testnet `TLAMP_NAME=744c414d50`, mainnet `LAMP_NAME=4c414d50` | Là PARAM apply-param của `lamp_mint` → policyId khác nhau. |
| thread NFT SupplyState | `SUPPLY_NAME=535550504c59` ("SUPPLY") | 1 NFT ghim UTxO SupplyState. |

**Đường mint OrgDID-gated đi qua `DistributionVest`.** Mint neo OrgDID = phát hành theo quota Distribution → route `DistributionVest`, ép `dist_minted + Δ ≤ dist_cap`. Đường `ReserveDraw` (rót Reserve, permissionless, spend reserve_thread NFT) do builder **riêng** `Reserve/offchain/drawBuilder.ts` dựng — **KHÔNG** dùng đường mint-by-OrgDID này (mintBuilder.ts §PHẠM VI ghi rõ).

**Shape tx mint đúng (khớp `buildMintTx` trong mintBuilder.ts):**
1. **Spend** SupplyState UTxO (mang thread NFT) — redeemer `Advance`, attach supply_state spend validator.
2. **Mint** Δ LAMP qua policy `lamp_mint` — redeemer mint route `DistributionVest`.
3. **Reference input** anchor TAAD của OrgDID (CIP-31, `lamp_mint` đọc controller_pkh + status Active).
4. **Output** recreate SupplyState' tại **cùng** script address, mang lại thread NFT, datum cập nhật `dist_minted += Δ`.
5. **Output** Δ LAMP trả `recipient` (rỗng → ví controller self-custodial).
6. **Signer** controller_pkh (must_be_signed_by của `lamp_mint`).

**Wiring chuẩn = `LAMP/Genesis/offchain/src/mintBuilder.ts` (CONTRACT §5).** Builder `build_mint_lamp_via_did()` (mint_lamp.rs) + flow submit-tx PHẢI khớp mẫu này — hiện `mint_lamp.rs` chưa compose SupplyState (nó chỉ mint + ref-input anchor). Đây là **gap builder** cần Core bổ sung trước khi mint chạy đúng cap on-chain (open item §7).

🔴 **KHÔNG dùng `did_token_mint`** cho LAMP — đó là bản tổng quát đa-token, sai policy. LAMP đi đúng cặp `lamp_mint` + `supply_state`.

**Fail-fast offchain (mirror `applyMint` trong supplyState.ts):** trước khi submit, backend đọc SupplyState datum hiện tại → kiểm `dist_minted + Δ ≤ dist_cap`. Vượt → reject `LAMP_MINT_CAP_EXCEEDED` với `remaining = dist_cap − dist_minted` (không tốn phí một tx chắc-chắn-reject). Validator SupplyState là cửa chốt (LMINT-8).

---

## §2 — Intent `LAMP_MINT`

Thêm hằng `INTENT_LAMP_MINT = "LAMP_MINT"` cạnh `INTENT_SEED_EXPORT` trong `SignRequestServiceImpl`. `SignIntent` là record mở (`type`, `body: Map<String,Object>`, `domain`, `appId`, `nonce`, `timestamp`, `displayText`) — không đổi shape, chỉ thêm giá trị `type`.

**`intent.body` schema (snake_case, keys sorted):**
```json
{
  "amount": 1000000,
  "org_did": "did:phoenix:org:...",
  "recipient": "addr_test1qz..."      // rỗng "" nếu self-custodial
}
```

**Canonical JSON để ký** — GIỐNG HỆT cơ chế `SEED_EXPORT`/`TRANSFER` đang chạy: `canonicalMapper.writeValueAsBytes(intent)` với `ORDER_MAP_ENTRIES_BY_KEYS=true`, no whitespace, no indent. Mobile PHẢI dùng cùng canonicalization (đã đồng bộ cho các intent khác — không phát minh lại). Message ký = canonical bytes của **toàn bộ `SignIntent`** (không chỉ `body`), đúng như `canonicalIntentBytes(payload.intent())` trong `SignRequestServiceImpl.approve`.

**`display_text` (hiển thị mobile chống blind-sign):**
```
Mint 1,000,000 LAMP cho <recipient ngắn gọn | "ví của bạn">
thay mặt Org "<org name>"
```
Ví dụ: `Mint 1,000,000 LAMP → ví của bạn — thay mặt Org "Green Sun Co."`. Backend dựng `display_text` từ `org_dids.name` + amount có phân tách hàng nghìn. `recipient` rỗng → "ví của bạn".

**Nonce + timestamp:** như mọi intent — `nonce` 32-byte hex, `timestamp` epoch giây, server kiểm lệch ±60s (`TIMESTAMP_SKEW_SEC`) + consume nonce khi approve (chống replay). Với m-of-n: mỗi founder ký **cùng một** intent (cùng nonce) — xem §3 về dedup + consume-nonce một lần.

---

## §3 — Gom chữ ký m-of-n

Hôm nay `SignRequestServiceImpl.approve` nhận **1** chữ ký → set `status=approved` → emit `signed`. Với OrgDID `authority_model=threshold` cần **m** chữ ký từ m member khác nhau. Mở rộng relay để **tích luỹ** và chỉ emit `signed` khi đủ ngưỡng.

### Nguồn ngưỡng + member set
- `authority_model` + `threshold`: đọc từ `org_dids` (`OrgDidRepository.findById(orgDid)`).
- Member set: `org_authority_members` theo `orgDid` (`OrgAuthorityMemberRepository`, `findByOrgDid`).
- `authority_model=single` → `threshold=1`, member set = `{owner_did}` → hành vi y hệt hôm nay (1 chữ ký).

### Thiết kế tích luỹ (extend `SignRequestPayload` state trong Redis)
Thêm khối `mint_authorization` vào payload lưu Redis cho intent `LAMP_MINT` (chỉ khi threshold):
```json
{
  "threshold": 3,
  "org_did": "did:phoenix:org:...",
  "member_dids": ["did:...a","did:...b","did:...c","did:...d"],
  "collected": [
    { "member_did": "did:...a", "public_key_hex": "04ab..", "signature": "30..", "at": 1751461000 }
  ]
}
```
`SignApproveRequest` hiện chỉ có `{public_key_hex, signature}` → với LAMP_MINT threshold, member ký từ thiết bị của **member đó**; backend suy ra `member_did` từ `public_key_hex` (tra `authorized_keys` → user_did) rồi kiểm user_did ∈ member set.

### Luồng approve mở rộng (mỗi lần 1 founder gọi `/sign/{id}/approve`)
1. Load payload; nếu `type != LAMP_MINT` hoặc `authority_model=single` → nhánh cũ (1 chữ ký, emit ngay). **[LMINT-1]**
2. Kiểm timestamp skew ±60s (như cũ, làm trước để fail-fast).
3. Verify `public_key_hex` là **active key** của một `member_did ∈ member_dids` (`authorized_keys` join). Nếu không thuộc member set → reject `SIGNATURE_INVALID`. **[LMINT-2]**
4. Verify chữ ký ECDSA trên canonical intent bytes (cùng message cho mọi founder). **[LMINT-3]**
5. **Dedup:** nếu `member_did` đã có trong `collected` → reject `SIGN_REQUEST_DUPLICATE_SIGNER` (không cho 1 người ký 2 lần lấp ngưỡng). **[LMINT-4]**
6. Append `{member_did, public_key_hex, signature, at}` vào `collected`; lưu lại Redis với TTL còn lại.
7. Nếu `size(collected) < threshold` → **KHÔNG** emit `signed`; trả 200 với `{signatures_collected, signatures_required}`. Optionally emit SSE event phụ `signature_progress` để web hiện "2/3 đã ký". **[LMINT-5]**
8. Khi `size(collected) == threshold` → set `status=approved`, **consume nonce đúng MỘT lần** (tại thời điểm đủ ngưỡng, không consume mỗi chữ ký — nếu không founder thứ 2 sẽ vấp nonce đã dùng), emit `signed` kèm **toàn bộ** `collected` (mobile cần đủ m chữ ký để lắp vào tx witness set). **[LMINT-6]**

### Bất biến (invariant)
| ID | Bất biến |
|---|---|
| **LMINT-1** | `authority_model=single` ⇒ đúng 1 chữ ký, hành vi không đổi so với hôm nay. |
| **LMINT-2** | Mỗi chữ ký chấp nhận PHẢI thuộc một active key của một `member_did` trong member set on-record của orgDid. |
| **LMINT-3** | Chữ ký verify trên **cùng** canonical intent bytes (cùng nonce, cùng timestamp) cho mọi founder. |
| **LMINT-4** | Không đếm trùng: mỗi `member_did` đóng góp tối đa 1 chữ ký vào ngưỡng (dedup theo member_did, không theo public_key_hex). |
| **LMINT-5** | `signed` CHỈ emit khi `collected == threshold`. Dưới ngưỡng → pending, không side-effect. |
| **LMINT-6** | Nonce consume đúng 1 lần, tại thời điểm đủ ngưỡng. Hết TTL trước khi đủ ngưỡng → request expire, mọi chữ ký đã gom bị bỏ (phải khởi tạo lại). |
| **LMINT-7** | Ngưỡng theo `authority_model` (Math §13, patch Specs#12): **single** → `SoleOwner` 1 chữ ký (threshold=⊥); **threshold** → `threshold ≥ 2` ∧ `≤ size(member_dids)`. 🟡 Thực tế: `V16` (đã merge #40) hiện CHECK `threshold >= 1` — chờ follow-up sửa `>= 2`. Vi phạm ngưỡng = lỗi cấu hình OrgDID (enforce ở founding), không phải lỗi mint. |
| **LMINT-8** | 🔴 **Redeemer + cap CHƯA CHỐT — chờ LAMP hợp nhất design lên main.** Đính chính: redeemer `Advance` là BỊA (không tồn tại). Code thật: **main** (`Tokenomics/supply_state.ak`) = SupplyState `CountMint`, cap đơn `total' ≤ CAP`, `lamp_mint` = `Mint` (rỗng); **branch** `feat/lamp-mint-compose-anchor-cap` (chưa merge) = route `DistributionVest`/`ReserveDraw`, `dist_cap`/`reserve_cap`, mint rót vào **kho Distribution (không ra ví)**. Builder Core freeze shape redeemer + `build_mint_lamp_via_did` compose SupplyState SAU khi LAMP chốt canonical + merge main (xem message → LAMP). Cap 36 tỷ enforce ở validator SupplyState (nguồn chân lý); backend fail-fast (`LAMP_MINT_CAP_EXCEEDED`/`LAMP_MINT_NO_SUPPLY_STATE`). |

**TTL:** dùng `phoenixkey.sign-request.ttl-seconds` (mặc định 120s) — có thể quá ngắn để m người ký. Thêm config riêng `phoenixkey.sign-request.mint-ttl-seconds` (đề xuất 600s = 10 phút, khớp `ORG_NONCE_TTL`) cho intent LAMP_MINT threshold. **[open item nếu muốn UX chờ lâu hơn — §7]**

---

## §4 — Authorization: backend check gì vs validator check gì

**KHÔNG nhân đôi lòng tin.** Backend làm gate rẻ + UX; validator on-chain là nguồn chân lý cuối.

### Backend check (offchain — rẻ, fail-fast, UX)
- `session_token` hợp lệ (`TYPE_SESSION`) → resolve caller PersonDID.
- Caller PersonDID là **member/controller** của `orgDid`:
  - `single`: caller == `org_dids.owner_did`.
  - `threshold`: caller ∈ `org_authority_members(orgDid)`.
- Mỗi chữ ký gom được thuộc active key của một member (LMINT-2).
- Đủ ngưỡng m trước khi cho submit (LMINT-5/6).
- `amount > 0`; `recipient_address` (nếu có) là bech32 hợp lệ (mobile builder cũng check, backend check sớm để phản hồi 400 rõ ràng).
- **Cap fail-fast (§1.5):** đọc SupplyState datum → `dist_minted + Δ ≤ dist_cap`. Vượt → `LAMP_MINT_CAP_EXCEEDED`. Tx `signed_tx_cbor` phải spend SupplyState (redeemer `Advance`) → thiếu = `LAMP_MINT_NO_SUPPLY_STATE`.
- (Phòng thủ, optional) fetch anchor UTxO qua `orgDid → TAAD script addr` → xác nhận tx `signed_tx_cbor` tham chiếu đúng anchor đó, chưa bị revoke off-chain.

### Validator/policy on-chain check (nguồn chân lý — KHÔNG lặp lại ở backend)
⚠ **ĐÈ bởi V2-1 (đầu file):** code thật đọc **Registry NFT**, KHÔNG đọc anchor TAAD. (a)(b)(c) dưới là mô-tả v1 lỗi-thời — giữ để đối chiếu.
`lamp_mint` policy (~~đọc anchor TAAD~~ → **đọc Registry NFT ref-input**, `registry.validate_mint`) tự enforce:
- **(a)** ~~tx mang reference input giữ anchor NFT của OrgDID~~ → tx mang **Registry NFT ref-input** (`name = blake2b_256(org_did)`) chứa entry `token_tag:"LAMP"`.
- **(b)** `authority_satisfied` theo entry: `SinglePkh{pkh}` (1 chữ ký) | `MultiSig{pkhs, threshold}` (m chữ ký ∈ pkhs) — thay cho `controller_pkh` đơn.
- **(c)** entry KHÔNG `Revoked` (thay cho "anchor status Active"); ràng-buộc-Active dời sang write-side Registry validator.

**Validator SupplyState (spend, redeemer `Advance`) — cửa chốt CAP, §1.5:**
- **(d)** tx spend SupplyState NFT + recreate SupplyState' cùng script address với thread NFT.
- **(e)** `dist_minted + Δ ≤ dist_cap` (⇒ `minted_total ≤ TOTAL_CAP_OIL = 36 tỷ`), monotonic, cap bất biến.

⇒ Backend **KHÔNG** cần tự kiểm "OrgDID còn Active không" hay "còn cap không" bằng cách tin DB — nếu OrgDID revoked on-chain, validator `lamp_mint` từ chối (điều kiện c); nếu vượt cap, validator SupplyState từ chối (điều kiện e). Backend chỉ làm UX/gate rẻ (fail-fast cap qua đọc SupplyState datum) để không tốn phí submit một tx chắc-chắn-fail. Với m-of-n: backend verify m chữ ký **app-level** (chống forge trước khi trả phí), validator enforce `controller_pkh` — với threshold, controller_pkh là key gộp/đại diện đúng theo cách `lamp_mint` được tham số hoá (xem §5 dependency).

---

## §5 — ~~Config: `LAMP_POLICY_CBOR_HEX` = applied `lamp_mint` per-OrgDID~~ (LỖI-THỜI — ĐÈ bởi V2-2)

> ⚠ **TOÀN BỘ §5 LỖI-THỜI (đầu file V2-2).** Tiền đề "`lamp_mint` apply-param per-OrgDID theo `anchor_nft_name`" SAI: code không có param anchor. Đúng: **1 policy-id LAMP tĩnh** (apply-param theo token), org phân biệt runtime qua **Registry NFT** `name=blake2b_256(org_did)`. Thay cột `org_dids.lamp_mint_policy_cbor` per-org bằng: (a) 1 config `lamp_mint_policy_id` tĩnh toàn hệ; (b) mỗi OrgDID có **entry Registry `token_tag:"LAMP"`** (Core dựng qua `registry_mint`, controller-DID-ký) TRƯỚC khi mint. m-of-n → `MultiSig{pkhs,M}` (V2-4, tháo dependency dưới). Phần dưới giữ để đối chiếu lịch-sử.

### Vấn đề tham số hoá (v1 — lỗi-thời)
`lamp_mint.ak` có **12 tham số**, trong đó có `anchor_nft_policy` + `anchor_nft_name` — mà `anchor_nft_name` **derive từ DID** (mỗi OrgDID một anchor NFT khác nhau). ⇒ script `lamp_mint` **applied** ra **policy id khác nhau cho mỗi OrgDID**. Không có một `LAMP_POLICY_CBOR_HEX` toàn cục dùng chung cho mọi org.

### Quyết định (4 trục — dài hạn / first-principles / tối ưu / user)
**Áp tham số tại thời điểm tạo/kích hoạt OrgDID và LƯU per-OrgDID**, KHÔNG áp lại on-demand mỗi lần mint.
- **Lý do (first-principles):** `applied_lamp_mint_cbor` là hàm thuần của (base blueprint + 12 tham số của OrgDID). Các tham số đó bất biến sau khi OrgDID sinh ra (anchor policy/name cố định theo DID). Áp một lần → xác định (deterministic) → cache. Áp lại mỗi mint = lặp công vô ích + rủi ro lệch nếu blueprint đổi giữa chừng.
- **Lý do (tối ưu):** mỗi mint chỉ đọc 1 hàng DB thay vì chạy applicator.
- **Lý do (user + dài hạn):** policy id ổn định theo OrgDID → ví/explorer hiển thị nhất quán, audit dễ.

### Lưu ở đâu
Thêm cột vào `org_dids` (hoặc bảng phụ `org_lamp_policy` nếu muốn tách):
```
org_dids.lamp_mint_policy_cbor   TEXT     -- applied lamp_mint script CBOR hex (per-OrgDID)
org_dids.lamp_mint_policy_id     CHAR(56) -- blake2b_224 hash = policy id (tiện query/hiển thị)
```
Điền tại **thời điểm OrgDID được kích-hoạt-để-mint** (khi anchor TAAD UTxO đã on-chain + biết `anchor_nft_name`). Với single-owner mint hiện dùng metadata-6789 (transitional, xem `OrgServiceImpl` note "swaps to TAAD UTxO redeemer when validator ships") — nên field này điền khi **validator TAAD deploy + anchor mint xong**, không phải ngay ở `/identity/org/create` hôm nay.

`build_mint_lamp_via_did` nhận `lamp_policy_cbor_hex` = giá trị `org_dids.lamp_mint_policy_cbor` này (mobile đọc qua một endpoint đọc org, hoặc backend nhét vào intent.body — chọn: **trả trong response endpoint 1** để mobile khỏi round-trip thêm).

### DEPENDENCY (chặn) — cần chốt với Core/LAMP team
> **`lamp_mint` được tham số hoá per-OrgDID BẰNG CÁCH NÀO, và ai chạy applicator?**
> - Applicator (apply-params) chạy ở đâu: Rust core? backend? script deploy?
> - 12 tham số lấy từ đâu (anchor policy = validator hash; anchor name = derive từ DID — công thức nào?)
> - Với `authority_model=threshold`: `controller_pkh` trong anchor datum là 1 key hay multisig-native? `lamp_mint` enforce `must_be_signed_by(controller_pkh)` đơn — vậy m-of-n ánh xạ ra sao (native script multisig? aggregated key? m anchor?).
>
> Đây là **open item chặn** — ghi rõ ở §7. Nếu chưa có applicator per-OrgDID, endpoint mint chạy được cho **single-owner** trước (1 controller key), m-of-n chờ chốt cơ chế.

Biến hiện có: `phoenixkey.cardano.lamp-policy-id` (đã có trong `application-testnet.yml`) — là **faucet** `lamp_policy` id, DÙNG cho `/activation`. KHÔNG nhầm với `lamp_mint`. Đề xuất đổi tên rõ ràng: `phoenixkey.cardano.faucet-lamp-policy-id` (faucet) vs `org_dids.lamp_mint_policy_id` (canonical per-OrgDID). **[open item naming — §7]**

---

## §6 — Test plan trên Preview (thật, không mock)

Điều kiện: `lamp_mint` deploy Preview + ít nhất 1 OrgDID có anchor TAAD on-chain + `lamp_mint_policy_cbor` điền sẵn. Blockfrost Preview key cấu hình (Lợi, §B.0).

| # | Kịch bản | Bước | Kỳ vọng |
|---|---|---|---|
| T1 | **Single-owner mint (1 chữ ký)** | tạo OrgDID single → `POST /mint-lamp {amount:1_000_000}` → mobile controller ký → `submit-tx` | 200 `{tx_hash}`; sau ~1 block, LAMP xuất hiện ở recipient (Blockfrost `/addresses/{addr}` thấy asset under `lamp_mint_policy_id`, quantity 1_000_000). |
| T2 | **Threshold mint (m chữ ký gom đủ)** | OrgDID threshold m=2 (3 member) → `POST /mint-lamp` → member A `approve` (progress 1/2) → member B `approve` (đủ 2/2, emit `signed`) → mobile lắp 2 witness → `submit-tx` | Sau A: 200 `{signatures_collected:1,required:2}`, KHÔNG emit `signed`. Sau B: emit `signed`, `submit-tx` → `{tx_hash}`, LAMP xuất hiện. |
| T3 | **Reject < m chữ ký** | như T2 nhưng chỉ A ký rồi gọi `submit-tx` | `submit-tx` reject (request chưa `approved`) — `SIGN_REQUEST_NOT_APPROVED`/`409`. |
| T4 | **Reject signer không phải member** | threshold org → một PersonDID **ngoài** member set gọi `approve` | reject `SIGNATURE_INVALID` (LMINT-2); `collected` không tăng. |
| T5 | **Reject OrgDID revoked** | revoke anchor on-chain (status ≠ Active) → gom đủ chữ ký → `submit-tx` | Backend submit, **validator từ chối** (điều kiện c); `submit-tx` trả `CARDANO_TX_FAILED`, không có LAMP mint. (Chứng minh backend không nhân đôi trust — validator là cửa chốt.) |
| T6 | **Dedup 1 member ký 2 lần** | threshold m=2 → member A `approve` 2 lần | Lần 2 reject `SIGN_REQUEST_DUPLICATE_SIGNER` (LMINT-4); `collected` vẫn = 1. |
| T7 | **Reject vượt remaining cap** | mint `Δ` sao cho `dist_minted + Δ > dist_cap` → gom đủ chữ ký → `submit-tx` | Backend fail-fast `LAMP_MINT_CAP_EXCEEDED` (mirror `applyMint` GMINT-010) HOẶC nếu lọt tới submit thì **validator SupplyState từ chối** → `CARDANO_TX_FAILED`; không LAMP mint. Chứng minh trần 36 tỷ được ép (LMINT-8). |
| T8 | **Reject tx thiếu SupplyState** | dựng tx chỉ mint qua `lamp_mint`, KHÔNG spend SupplyState → `submit-tx` | Backend reject `LAMP_MINT_NO_SUPPLY_STATE` (§1.5 fail-fast); chứng minh compose SupplyState bắt buộc. |

**Evidence bắt buộc trước khi tuyên bố PASS:** với mỗi T1/T2 dán `tx_hash` + output Blockfrost `/addresses/{recipient}` thấy LAMP quantity đúng. Với T3–T6 dán response body lỗi cụ thể. (Nguyên tắc verify-behavior: compile pass ≠ đúng.)

---

## §7 — Ranh giới + open items

**Giao Long (OFFCHAIN — PR này):**
- 2 endpoint `POST /identity/org/{orgDid}/mint-lamp` + `/mint-lamp/submit-tx` (trong `OrgController` + `OrgService`).
- Intent type `LAMP_MINT` + canonical body + `display_text` (§2).
- Mở rộng `SignRequestServiceImpl.approve` gom m-of-n (§3, LMINT-1..7) + `signature_progress` SSE (optional).
- Migration `org_dids.lamp_mint_policy_cbor` + `lamp_mint_policy_id` (§5) + đường điền giá trị.
- Config `mint-ttl-seconds`; đổi tên faucet policy id cho rõ (§5).
- Fail-fast cap + đối chiếu compose SupplyState ở `submit-tx` (§1.5, §4): đọc SupplyState datum, kiểm `dist_minted + Δ ≤ dist_cap`, kiểm tx spend SupplyState — `LAMP_MINT_CAP_EXCEEDED` / `LAMP_MINT_NO_SUPPLY_STATE`.
- Test Preview T1–T8 với evidence thật (§6).

**ĐÃ XONG (không làm lại):**
- On-chain policy `lamp_mint.ak` + validator SupplyState `supply_state` (LAMP repo).
- Wiring tham chiếu `LAMP/Genesis/offchain/src/mintBuilder.ts` (CONTRACT §5) — mẫu compose lamp_mint + supply_state.

**⚠️ GAP builder (cần Core bổ sung — chặn mint đúng cap on-chain):**
- `build_mint_lamp_via_did()` (mint_lamp.rs) hiện **CHỈ** mint + ref-input anchor, **CHƯA compose SupplyState** (chưa spend SupplyState NFT với redeemer `Advance`, chưa recreate SupplyState', chưa mint route `DistributionVest`). Phải khớp `mintBuilder.ts` (CONTRACT §5) thì cap 36 tỷ mới được ép. Đây là **on-chain/Core**, KHÔNG phải Long — nhưng chặn test T1/T2 tới lúc sửa xong. Ghi open item #6.

**Ghi chú phí MAGIC (Feecover):** tx mint này **về sau có thể chịu phí dịch vụ MAGIC** (mô hình Feecover — tính-trên-MAGIC, trả bằng CARP về provider-vault). PR này KHÔNG gộp phí — mint chạy trần trước. Khi Feecover wire vào, thêm 1 output/relay phí ở tầng build (mobile) + tính phí ở backend. Ghi làm hạng mục kế tiếp, không chặn PR này.

### Open items cần anh / Tuân / Core chốt
1. **[CHẶN m-of-n] Cơ chế tham số hoá `lamp_mint` per-OrgDID + ánh xạ threshold → controller_pkh** (§5 dependency). `lamp_mint` enforce `must_be_signed_by(controller_pkh)` đơn — với OrgDID threshold, `controller_pkh` là native-multisig? aggregated key? Cần Core/LAMP chốt trước khi T2 chạy được. **Single-owner (T1) không bị chặn** — có thể ship trước.
2. **Applicator apply-params chạy ở đâu** (Rust core / backend / deploy script) và công thức `anchor_nft_name` derive từ DID — cần để điền `lamp_mint_policy_cbor`.
3. **Server-signed variant?** Hiện thiết kế = mobile build+ký (Master_KEK on-device). Nếu muốn backend build hộ (headless mint, không cần mobile online) → cần controller key uỷ quyền tách khỏi Master_KEK. Anh quyết có làm không (ngoài phạm vi PR này).
4. **`mint-ttl-seconds`** cho threshold: 10 phút đủ để m người ký? (UX) — anh/Tuân xác nhận con số.
5. **Naming config** faucet vs canonical policy id (§5) — xác nhận đổi `phoenixkey.cardano.lamp-policy-id` → `...faucet-lamp-policy-id`.
6. **[CHẶN mint on-chain] `build_mint_lamp_via_did()` chưa compose SupplyState** (§1.5, §7 GAP). Cần Core sửa builder Rust khớp `mintBuilder.ts` CONTRACT §5: spend SupplyState (redeemer `Advance`) + mint route `DistributionVest` + recreate SupplyState'. Không sửa → tx mint hở cap, không đi qua trần 36 tỷ. Đây là on-chain/Core, không phải Long, nhưng T1/T2 chờ nó.
7. **Datum SupplyState + địa chỉ script SupplyState trên Preview** — mobile builder cần biết SupplyState UTxO ở đâu (script address + thread NFT policy `SUPPLY_NAME=535550504c59`) để fetch + spend. Ai cấp địa chỉ/policy này cho PhoenixKey (LAMP deploy Preview)? Cần trước T1.
