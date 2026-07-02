# PhoenixKey — Delegator→LAMP-claim: OFFCHAIN backend layer — DEV-READY

> **PHÂN LOẠI:** OFFCHAIN (`PhoenixKey-Database`, Spring Boot Java). PR giao **Long**.
> Đây là tầng **backend endpoints** cho luồng Delegator→LAMP-claim: Consent/Grant
> store, stake-state indexer, claim orchestration (submit + theo dõi), Merkle-proof
> serving. KHÔNG đụng `rust_core`/CIP-30 (Core) + Merkle-root/ISPO (LAMP).
>
> **KHÔNG chép lại Core spec.** Builder + CIP-30 connector đã đặc tả ở
> `PhoenixKey-Delegator-Core-Connector-Builders-Feat.md` (Tuân/Core). Spec này CHỈ
> viết các endpoint backend mà Core spec đánh dấu "tiêu thụ input / Long" (§0 Core:
> bước 2, 3, 4, 7). Nơi nào Core spec đã định shape (proof JSON, stake-state fields,
> submit) → spec này giữ NGUYÊN shape, không đổi.
>
> **Đọc kèm (nguồn sự thật, neo file:dòng — KHÔNG chép):**
> - `PhoenixKey-Delegator-Core-Connector-Builders-Feat.md` — Core layer. §0 bảng
>   7 bước; §2.3/§2.4 builder migration ăn `reward_lovelace`/`already_vote_delegated`;
>   §3.1/§3.5 builder claim ăn `proof_json`/`amount_lamp`/`pool_utxo_json`; §4 bảng
>   signing model; §6 ranh giới ("Backend Long, spec riêng" — chính là spec này).
> - `PhoenixKey-Build-Guide-Backend-Onchain.md:148-159` (§2.6) — Consent store +
>   verify, 4 endpoint gốc (spec này chuẩn hoá + bổ sung `/verify` `/revoke`).
> - `PhoenixKey-Build-Guide-Backend-Onchain.md:76-96` (§2.1) — resolve controller
>   HIỆN TẠI qua indexer; :161-167 (§2.7) — sự kiện `controller_changed` (rotate).
>   Verify Grant PHẢI dùng controller point-in-time (§1.4 dưới).
> - `IdentityController.java` — mẫu controller: `@RequestMapping("/identity")`,
>   `DataResponse<T>` (code 1000), `getPubkey`/`getStatus`, resolve controller
>   on-record; `userDidFromBearer` (Bearer session JWT, sub=userDid).
> - `SignRequestController.java` — mẫu relay ký web↔mobile: `/sign/request`,
>   `/sign/{id}/approve`, `parseBearer`. Grant-signing đi qua CHÍNH kênh này.
> - `ActivationController.java:66-104` — mẫu `/{id}/submit-tx` (nhận `signedTxCbor`,
>   gọi `blockfrost.submitSignedTx` → `tx_hash`) + `/{id}/status` + SSE `/{id}/events`
>   (`ActivationEventBus`). §3 claim orchestration MIRROR nguyên mẫu này.
> - `BlockfrostHttpClient.java:59,75,88` — `submitSignedTx`, `getCurrentSlot`,
>   `getAddressUtxos`; `@Value phoenixkey.cardano.*`. §2 stake-state thêm account
>   endpoints vào client này.
> - `application.yml:108` — `context-path: /api/v1`. Mọi path dưới đây tương đối với
>   base đó (vd `PUT /consent/{grantor_did}` = `PUT /api/v1/consent/{grantor_did}`).
> - `DataResponse.java` — envelope `{code, message, result}`, `JsonInclude.NON_NULL`.
> - `LAMP-DISTRIBUTION-SPEC.md` (kho LAMP) — Merkle claim ETD: leaf =
>   `blake2b_256(0x00 ++ serialise(claimer_addr) ++ amount_be_8)`, `ProofStep{is_left,
>   hash}`. §4 backend SERVE proof theo cây đã publish; KHÔNG dựng root.
> - `staking.rs` — builder Cardano cert/withdrawal ĐÃ CÓ (Core dùng); backend chỉ
>   cấp `reward_lovelace`/`stake-state` làm INPUT cho builder đó.
>
> **Quy ước chung mọi endpoint spec này:**
> - Base path `/api/v1` (context-path). Field JSON = **snake_case**. Envelope
>   `DataResponse<T>` (`code:1000` khi ok). Lỗi qua `AppException`/`ErrorCode`
>   (mẫu có sẵn), body `{code, message}`.
> - Backend KHÔNG giữ Master_KEK / khoá controller. Mobile Enclave ký; Lace ký.
>   Backend **điều phối + submit CBOR đã-ký**. Đây là bất biến nền (OFF-KEY-1).

---

## §0. Phân loại + vị trí trong luồng 7 bước

**Tầng:** OFFCHAIN backend (Long). Bản chất "uỷ quyền ký" trong luồng này (từ Core
spec §0 + §6, nhắc lại để đóng khung — KHÔNG phải trao khoá):

> **"Uỷ quyền ký" ≠ bàn giao khoá.** Cardano không có account-abstraction → không có
> "Lace ký tự động mãi mãi". Uỷ quyền khả dĩ = (a) **CIP-30 `signTx` từng-lần** (Lace
> bật popup mỗi tx), HOẶC (b) **migrate xong → controller ví Phoenix ký** (Enclave
> mobile giữ khoá, không phải backend). CỘNG (c) một **Grant** (DelegationToken ký
> off-chain) khẳng định thẩm quyền route-reward — đây là phần §1 spec này. Grant KHÔNG
> phải khoá; nó là bằng chứng-uỷ-quyền để backend gate hành động + để validator/dịch
> vụ đối chiếu. Backend KHÔNG ký thay; backend orchestrate + submit CBOR pre-signed.

Luồng 7 bước (từ Core spec §0), **tô đậm phần OFFCHAIN spec này chịu trách nhiệm**:

| Bước | Việc | Chủ | Spec này? |
|---|---|---|---|
| 0 | Tạo DID + ví Phoenix (đích claim) | — (live) | — |
| 1 | Kết nối Lace, đọc stake/reward (CIP-30 watch-only) | Core | — (Core §1) |
| 2 | **Đọc stake-state của credential** (registered/vote/reward/deposit) | **Long** | **§2** |
| 3 | **Uỷ quyền thẩm quyền (Grant/Consent store + verify)** | **Long** | **§1** |
| 4 | **Lấy Merkle-proof để claim LAMP** (serve từ cây đã publish) | **Long** | **§4** |
| 5 | Dựng tx (migrate / claim) — 2 builder | Core | — (Core §2/§3) |
| 6 | Ký — (a) CIP-30 Lace / (b) controller `/sign/request` | Core/Long | Long cấp relay ký (dùng lại `/sign/*`) |
| 7 | **Submit + theo dõi** (POST claim/submit + status/SSE) | **Long** | **§3** |

**Spec này giao 4 khối endpoint (đều `/api/v1`):**
- **B1 — Consent/Grant store** (§1): `PUT/GET /consent`, `POST /consent/verify`,
  `POST /consent/revoke`. Bước 3.
- **B2 — Stake-state indexer** (§2): `GET /stake-state/{stake_cred}`,
  `GET /identity/{did}/stake-status`. Bước 2 — gỡ blocker [OPEN-4].
- **B3 — Claim orchestration** (§3): `POST /claim/submit` + `GET /claim/{id}/status`
  + SSE. Bước 7.
- **B4 — Merkle-proof serving** (§4): `GET /airdrop-claim/{address}/{epoch}/merkle-proof`.
  Bước 4.

---

## §1. Consent/Grant store — DelegationToken (bước 3)

**Nguồn:** `Backend-Onchain §2.6` (4 endpoint gốc) + Core spec §0 bước 3 ("Long,
không build ở Core"). Spec này chuẩn hoá schema `DelegationToken` + 4 endpoint đầy đủ.

### §1.1 Việc làm

Lưu/đọc **Grant ký off-chain** cấp thẩm quyền route-stake-reward, + verify nhanh cho
app/dịch vụ. Node CHỈ lưu Grant đã-ký + cờ thu hồi; KHÔNG lưu khoá, KHÔNG lưu tài sản.
Grant = `DelegationToken` do **grantor** (chủ DID) ký bằng controller HIỆN TẠI.

### §1.2 Schema `DelegationToken`

Trường JSON (snake_case), lưu nguyên văn khi `PUT`:

```json
{
  "grantor_did": "did:phoenix:aaaaaaahl4nn6:ccd1feb6...",  // chủ thẩm quyền (ký Grant)
  "grantee":     "did:phoenix:... | addr1... | agent:tiger", // bên NHẬN uỷ quyền
  "action_tag":  "route_stake_reward",                      // tên hành động (enum khoá)
  "resource":    "stake1u... | pool1... | *",               // đối tượng (stake cred / pool / *)
  "nonce":       "b64u-128bit",                             // chống replay, unique/grantor
  "valid_from":  1893456000,                                // unix sec (không dùng trước mốc)
  "valid_until": 1896048000,                                // unix sec (hết hạn)
  "proof": {                                                // chữ ký của grantor
    "alg": "Ed25519",                                       // controller TAAD_Key = Ed25519
    "kid": "did:phoenix:...#controller",                    // khoá ký (controller on-record)
    "sig": "hex(64B)"                                       // ký canonical bytes (§1.3)
  }
}
```

- **`action_tag`** enum khoá phía backend; luồng này CHỈ chấp `"route_stake_reward"`.
  Tag lạ → `400 INVALID_ACTION_TAG`. (Mở rộng tag mới = PR riêng.)
- **`grantee`** đa dạng (DID / địa chỉ / Agent id). Verify KHÔNG suy diễn quyền của
  grantee — chỉ xác thực Grant do grantor ký hợp lệ + còn hạn + chưa thu hồi.
- **`resource`** `"*"` = mọi stake cred của grantor; hoặc stake cred / pool cụ thể.

### §1.3 Canonical bytes để ký/verify

`proof.sig` ký trên **canonical JSON của Grant TRỪ `proof`**: keys sorted, no
whitespace, UTF-8, số nguyên không thập phân. (Cùng quy ước "canonical form: keys
sorted, no whitespace" của `SignRequestController.approve` — tái dùng util có sẵn.)
Prefix domain-separation bắt buộc: ký `"PHOENIXKEY_GRANT:" + canonical_json_bytes`.

### §1.4 Endpoints

```
PUT  /consent/{grantor_did}                                     [auth: grantor]
  body: DelegationToken (đã ký)
  → DataResponse<{ nonce, stored_at_slot }>
  400 INVALID_ACTION_TAG | 400 GRANT_EXPIRED (valid_until ≤ now)
  403 GRANT_SIG_INVALID  | 409 GRANT_NONCE_REUSED

GET  /consent/{grantor_did}?grantee=&action=&resource=          [auth: grantor | public-read? xem §5]
  → DataResponse<{ grants: DelegationToken[] }>   // CHỈ Grant SỐNG (chưa hết hạn + chưa thu hồi)
  filter optional; thiếu filter → mọi Grant sống của grantor

POST /consent/verify                                            [public]
  body: { grant: DelegationToken }    (hoặc { grantor_did, nonce } để verify Grant đã lưu)
  → DataResponse<{ valid: bool, reason: string }>
     reason ∈ { "ok", "sig_invalid", "expired", "revoked", "not_yet_valid",
                "unknown_grantor", "controller_mismatch" }

POST /consent/revoke                                            [auth: grantor]
  body: { grantor_did, nonce, sig }   // sig của controller trên "PHOENIXKEY_REVOKE:"+grantor_did+":"+nonce
  → DataResponse<{ revoked_at_slot }>
  403 REVOKE_SIG_INVALID | 404 GRANT_NOT_FOUND
```

### §1.5 Logic verify (dùng cho `PUT` và `/verify`)

1. **Resolve grantor controller HIỆN TẠI** qua indexer (`IdentityService`/resolver,
   mẫu `IdentityController.getPubkey`). Nếu Grant có `valid_from`/thời điểm cụ thể và
   controller đã rotate → PHẢI dùng controller **point-in-time** tại `valid_from`
   (Backend-Onchain §2.1 resolve controller + §2.7 sự kiện `controller_changed`).
   Backend giữ lịch sử rotate (slot → controller_pkh); chọn controller đúng-mốc. Nếu
   `kid` trong proof ≠ controller đúng-mốc → `reason="controller_mismatch"`.
2. **Check chữ ký** `proof.sig` trên canonical bytes (§1.3) bằng controller pubkey
   đúng-mốc. Sai → `sig_invalid`.
3. **Check hạn:** `valid_from ≤ now ≤ valid_until`. Ngoài → `not_yet_valid`/`expired`.
4. **Check chưa thu hồi:** tra bảng revoke theo `(grantor_did, nonce)`. Có → `revoked`.
   Trả trạng thái **SỐNG** — revoke tức thì (không cache stale).

### §1.6 Bất biến GRANT-*

- **GRANT-STORE-1:** node chỉ lưu Grant đã-ký + cờ thu hồi; KHÔNG lưu khoá, KHÔNG lưu
  dữ liệu tài nguyên (`resource` chỉ là con trỏ).
- **GRANT-VERIFY-2:** verify PHẢI resolve controller point-in-time (§1.5-1); rotate
  không được vô hiệu Grant ký hợp lệ trước rotate, và không được cho khoá cũ ký Grant
  mới (mốc = `valid_from`).
- **GRANT-REVOKE-3:** revoke tức thì → `GET /consent` + `/verify` phản ánh ngay
  (không TTL cache che revoke). Revoke phải do controller HIỆN TẠI ký.
- **GRANT-NONCE-4:** `nonce` unique theo grantor; `PUT` trùng nonce → `409`
  (chống replay/ghi đè âm thầm).
- **GRANT-TAG-5:** chỉ `action_tag ∈ {"route_stake_reward"}` cho luồng này; tag khác
  → `400` (không mở quyền ngoài phạm vi spec).

---

## §2. Stake-state indexer (bước 2 — gỡ blocker [OPEN-4])

**Nguồn:** Core spec §0 bước 2 (🔴 blocker [OPEN-4], "Long"); builder migration
(Core §2.3) ĂN `reward_lovelace` + `already_vote_delegated` từ đây. `BalanceServiceImpl`
+ `BlockfrostHttpClient` là mẫu tích hợp Blockfrost đã có.

### §2.1 Việc làm

Trả trạng thái stake của một credential để Core builder migration/claim dùng làm
input. Đây là **blocker**: builder migration bỏ VoteDelegation khi
`already_vote_delegated`, tự-cấp-phí từ `reward_lovelace`; thiếu số này builder không
dựng đúng tx.

### §2.2 Endpoints

```
GET /stake-state/{stake_cred}?epoch=                           [public — dữ liệu on-chain công khai]
  stake_cred = stake credential (bech32 stake1... hoặc keyhash hex 28B)
  epoch optional (mặc định epoch hiện tại)
  → DataResponse<{
      registered:      bool,     // stake key đã register chưa (ảnh hưởng deposit/dereg)
      vote_delegated:  bool,     // đã vote-delegate DRep chưa (Plomin — Core §2.3 bỏ vote nếu true)
      reward_lovelace: long,     // R = reward chờ rút (withdraw amount cho builder)
      stake_deposit:   long,     // 2_000_000 nếu registered (hoàn khi dereg), else 0
      pool_id:         string,   // pool đang delegate (null nếu chưa)
      as_of_slot:      long
    }>
  404 STAKE_CRED_NOT_FOUND (chưa xuất hiện on-chain)

GET /identity/{did}/stake-status                               [auth: owner (Bearer session) — mẫu getHealth]
  → DataResponse<{
      did, phoenix_address,          // ví Phoenix (đích claim) — derive_phoenix_address
      stake_states: [ /* stake-state như trên cho từng stake cred của DID */ ],
      total_reward_lovelace: long,
      as_of_slot: long
    }>
```

`GET /identity/{did}/stake-status` = tổng hợp: DID có thể gắn nhiều stake cred (ví
Phoenix + ví cũ đang migrate). Aggregator gọi `/stake-state` cho từng cred + resolve
`phoenix_address` (từ resolver, mẫu `IdentityController.getDocument`).

### §2.3 Cách populate (Blockfrost account endpoints)

Thêm vào `BlockfrostHttpClient` (mẫu `getAddressUtxos`/`getCurrentSlot`):

| Field | Blockfrost source |
|---|---|
| `registered`, `pool_id`, `stake_deposit` | `GET /accounts/{stake_addr}` (`active`, `pool_id`, `controlled_amount`) |
| `reward_lovelace` | `GET /accounts/{stake_addr}` `withdrawable_amount` (reward chưa rút) |
| `vote_delegated` | `GET /accounts/{stake_addr}` `drep_id` ≠ null (Conway) |
| `as_of_slot` | `GET /blocks/latest` (`getCurrentSlot` đã có) |

**Không cần indexer worker riêng cho MVP** — proxy trực tiếp Blockfrost account
endpoints (đủ real-time cho migration/claim). Nếu tải cao → cache theo epoch
(`OnchainTaadStateCache` mẫu có sẵn), nhưng `reward_lovelace` phải fresh (đừng cache
stale — builder tự-cấp-phí sai là tx FAIL).

### §2.4 Bất biến STAKE-*

- **STAKE-1:** `reward_lovelace` = `withdrawable_amount` (reward chưa rút), KHÔNG phải
  tổng số dư; builder migration coi đây là withdraw amount (Core §2.3 bước 7).
- **STAKE-2:** `vote_delegated` phản ánh Conway DRep delegation (Plomin); Core dựa
  vào để bỏ/giữ VoteDelegation cert (CORE-MIGR-2).
- **STAKE-3:** `stake_deposit = 2 ADA` chỉ khi `registered=true`; builder dùng cho
  StakeDeregistration hoàn deposit (Core §2.3 bước 5).

---

## §3. Claim orchestration — submit + theo dõi (bước 7)

**Nguồn:** MIRROR `ActivationController.submitTx` (:66-104) + `/status` + SSE
`/events` + `ActivationEventBus`. Core spec §3.4/§4: claim tx đã-ký (controller mode
= Core ký; lace mode = Lace ký, Dart `assemble_witness`). Backend nhận **CBOR đã-ký**,
submit qua Blockfrost, track vòng đời.

### §3.1 Việc làm

Nhận claim/migration tx **đã ký hoàn chỉnh** (CBOR hex) từ app, submit lên Cardano,
theo dõi đến confirmed, phát sự kiện qua SSE. Backend KHÔNG ký (OFF-KEY-1) — chỉ
submit + poll (mẫu `blockfrost.submitSignedTx` + poll confirm, Backend-Onchain §2.4).

### §3.2 Endpoints

```
POST /claim/submit                                             [auth: owner (Bearer session)]
  body: {
    did,                       // chủ claim (ví Phoenix đích)
    tx_kind,                   // "claim" | "migration"  (theo dõi + kiểm chuẩn)
    signed_tx_cbor,            // CBOR hex ĐÃ KÝ (Core builder + Lace/controller đã ký)
    grant_nonce?               // nếu claim gate bằng Grant (§5) — verify trước submit
  }
  → DataResponse<{ claim_id, tx_hash, status }>   // status = "SUBMITTED"
  400 INVALID_TX_CBOR | 403 GRANT_INVALID (nếu grant_nonce fail §1.5)
  409 ALREADY_SUBMITTED (idempotent theo tx_hash)
  502 NODE_SUBMIT_REJECT{reason}   // Blockfrost/node từ chối (mẫu error model §2.8)

GET  /claim/{claim_id}/status                                  [public — claim_id opaque UUID 128-bit]
  → DataResponse<{ claim_id, tx_hash, status, confirmations, lamp_amount?, reason? }>
     status ∈ { SUBMITTED, ON_CHAIN, CONFIRMED, FAILED }

GET  /claim/{claim_id}/events   (text/event-stream)           [public — mẫu ActivationController.events]
  SSE: { type, payload }   type ∈ { "submitted", "on_chain", "confirmed", "failed" }
```

### §3.3 Vòng đời (state machine — mẫu ActivationStatus)

`SUBMITTED` → (poll thấy trong mempool/block) `ON_CHAIN` → (≥ N conf, mặc định 1
preprod) `CONFIRMED` → phát SSE `confirmed` + đọc `lamp_amount` tại ví Phoenix (xác
minh LAMP đã về). Submit reject / rollback → `FAILED` + `reason`.

- Submit qua `blockfrost.submitSignedTx(signed_tx_cbor)` → `tx_hash` (mẫu có sẵn).
- Poll confirm: worker định kỳ `GET /txs/{tx_hash}` (block_height/confirmations);
  giống pipeline poll của Activation.
- SSE qua `ActivationEventBus` (tái dùng bus, hoặc bus riêng `ClaimEventBus` cùng mẫu).

### §3.4 Bất biến CLAIM-SUBMIT-*

- **CLAIM-SUBMIT-1:** backend KHÔNG ký / KHÔNG chỉnh CBOR; chỉ submit `signed_tx_cbor`
  y nguyên (OFF-KEY-1). Sửa CBOR = phá chữ ký.
- **CLAIM-SUBMIT-2:** idempotent theo `tx_hash` — resubmit cùng tx → `409` + trả lại
  `claim_id` cũ (không tạo bản ghi trùng, không double-submit).
- **CLAIM-SUBMIT-3:** nếu `grant_nonce` có → verify Grant (§1.5) TRƯỚC submit; fail →
  `403`, KHÔNG submit (backend-check, §5).
- **CLAIM-SUBMIT-4:** error model chuẩn (Backend-Onchain §2.8): mã lỗi rõ +
  `retryable` cho `NODE_SUBMIT_REJECT`/`NODE_BEHIND` (app rebuild-and-retry) vs
  fail-cứng (`INVALID_TX_CBOR`).

---

## §4. Merkle-proof serving (bước 4)

**Nguồn:** Core spec §3.1/§3.5 (builder claim ăn `proof_json` +`amount_lamp`);
`LAMP-DISTRIBUTION-SPEC.md` (leaf/proof shape). Core §6: "Merkle snapshot PHẢI dùng
`phoenix_addr` làm `address` trong leaf". Backend SERVE proof từ cây đã publish;
KHÔNG dựng root.

### §4.1 Việc làm

Trả Merkle-proof cho một địa chỉ (ví Phoenix) tại một epoch để Core builder claim
đóng gói `Claim{claimer, amount, proof}`. Backend đọc cây đã publish (từ đội LAMP),
tính đường proof cho leaf tương ứng.

### §4.2 Endpoint

```
GET /airdrop-claim/{address}/{epoch}/merkle-proof             [public — proof là dữ liệu công khai]
  address = ví Phoenix (bech32 addr1...) — PHẢI khớp `address` trong leaf snapshot
  epoch   = đợt airdrop/ETD
  → DataResponse<{
      leaf:        "hex32",                    // blake2b_256(0x00 ++ serialise(addr) ++ amount_be_8)
      proof:       [ { is_left: bool, hash: "hex32" } ],   // ProofStep[] — Core §3.2 khớp byte
      amount_lamp: long,                       // amount trong leaf (redeemer Int + amount_be_8)
      merkle_root: "hex32",                    // root đã publish (để app/Core đối chiếu)
      pool_ref:    { policy, name }            // POOL NFT unit (Core resolve POOL UTxO)
    }>
  404 NOT_IN_TREE (address không có trong snapshot epoch này)
  409 ALREADY_CLAIMED (slot đã burn — spend-once; backend biết qua indexer, tránh
      để user dựng tx chắc-FAIL)
```

Shape `proof[]` + `leaf` PHẢI khớp `merkle.ak`/Core §3.2 byte-for-byte (`is_left`,
`hash` 32B, `amount_be_8`). Backend KHÔNG tự hash lại theo cách khác — đọc từ cây
publish của LAMP.

### §4.3 Nguồn cây + DEPENDENCY trên LAMP

- **Merkle-root SOURCE = đội LAMP (Keeper posting root).** Root nằm ở datum POOL
  on-chain; **cây đầy đủ (leaves + nội bộ) do LAMP publish** (file/endpoint LAMP).
  Backend Long **KHÔNG dựng root, KHÔNG quyết snapshot** — chỉ:
  1. Nạp cây publish (leaves = `{address, amount}` theo `phoenix_addr` của delegator).
  2. Với `{address, epoch}` → tính đường proof + trả (§4.2).
  3. Đối chiếu `merkle_root` đọc được với root ở datum POOL (indexer) — lệch → `409`
     / cảnh báo (cây stale).
- **DEPENDENCY (message §7):** LAMP phải publish (a) format cây + kênh phân phối,
  (b) khẳng định leaf `address` = ví Phoenix của delegator (không phải địa chỉ Lace
  cũ), (c) `pool_ref` NFT unit. Chưa có → §6 test dùng snapshot 1-leaf tay (đánh dấu
  mock), khớp Core §5.2 gate.

### §4.4 Bất biến MERKLE-SERVE-*

- **MERKLE-SERVE-1:** backend chỉ SERVE proof; không dựng/không sửa root (chủ quyền
  LAMP). Root ở datum POOL là chân lý; cây publish phải hash về đúng root.
- **MERKLE-SERVE-2:** `leaf`/`proof`/`amount_lamp` khớp byte `merkle.ak` (Core §3.2)
  — sai 1 byte → validator reject claim.
- **MERKLE-SERVE-3:** leaf `address` = ví Phoenix (Core §3.1) → LAMP route đúng ví
  Phoenix. Backend từ chối serve nếu `address` truyền vào ≠ địa chỉ trong leaf.

---

## §5. Authorization — endpoint nào auth vs public, Grant gate claim

### §5.1 Phân loại auth

| Endpoint | Auth | Lý do |
|---|---|---|
| `PUT /consent/{grantor_did}` | **grantor** (Bearer session, sub=grantor_did) + Grant tự-ký | chỉ chủ DID ghi Grant của mình |
| `POST /consent/revoke` | **grantor** (controller ký revoke) | chỉ chủ thu hồi |
| `GET /consent/{grantor_did}` | **grantor** (mặc định) | Grant có thể lộ grantee/resource — riêng tư chủ DID |
| `POST /consent/verify` | **public** | verify là kiểm-tra công khai (input là Grant đã ký) |
| `GET /stake-state/{cred}` | **public** | dữ liệu on-chain công khai |
| `GET /identity/{did}/stake-status` | **owner** (Bearer, mẫu getHealth) | gộp theo DID → tránh user A soi user B |
| `POST /claim/submit` | **owner** (Bearer session) | chỉ chủ ví Phoenix submit claim của mình |
| `GET /claim/{id}/status` `/events` | **public** | `claim_id` opaque UUID 128-bit (mẫu Activation SSE) |
| `GET /airdrop-claim/.../merkle-proof` | **public** | proof công khai |

Bearer session = mẫu `userDidFromBearer` (JWT `type=session`, `sub=userDid`).
Grant-signing (grantor ký DelegationToken) đi qua kênh `/sign/request` +
`/sign/{id}/approve` có sẵn (mobile Enclave ký) — KHÔNG endpoint ký mới.

### §5.2 Grant gate claim — backend-check vs on-chain

Hai lớp, ĐỘC LẬP, cả hai đều thật:

- **Backend-check (soft gate, §3):** khi `POST /claim/submit` kèm `grant_nonce`,
  backend verify Grant (§1.5) TRƯỚC submit. Fail → `403`, không tốn phí node. Đây là
  UX-gate + audit (backend biết claim này có uỷ quyền hợp lệ). KHÔNG thay validator.
- **On-chain check (hard gate, validator):** claim tx do validator `airdrop_pool`
  gác — **permissionless Merkle** (Core §3.4): validator KHÔNG đọc Grant của
  PhoenixKey; nó chỉ verify proof đúng + output đúng `claimer` (ví Phoenix). Tức
  **an toàn tài sản nằm ở validator + spend-once**, không phụ thuộc backend-check.
- **Grant dùng để gì:** với luồng **migration route-reward** (grantee thay grantor
  route reward về ví Phoenix), Grant là bằng chứng-uỷ-quyền để dịch vụ/relayer chấp
  nhận orchestrate thay. Với **claim permissionless**, Grant KHÔNG bắt buộc on-chain;
  backend vẫn ghi Grant để audit "ai uỷ quyền route về ví nào".
- **OFF-AUTH-1:** backend-check KHÔNG được coi là đủ an toàn tài sản; validator +
  spend-once là hard gate. Backend-check chỉ chặn sớm + audit.

---

## §6. Test plan Preview (real — evidence output thật, mandate anh Aladin)

Cần: DID PhoenixKey Active + ví Phoenix (preprod/preview); 1 stake cred preprod thật
(ví Lace test có reward + đã register); Blockfrost preprod key (`phoenixkey.cardano.*`).

1. **Store + verify Grant (§1):**
   - Ký `DelegationToken{action_tag:"route_stake_reward", ...}` bằng controller (qua
     `/sign/request` mobile) → `PUT /consent/{grantor_did}` → `200 {nonce}`.
   - `POST /consent/verify {grant}` → `{valid:true, reason:"ok"}`.
   - Sửa 1 byte `proof.sig` → `verify` trả `{valid:false, reason:"sig_invalid"}`.
   - **Evidence:** curl request/response cả 3 (mẫu "curl endpoint payload thực" —
     mandate verify behavior).

2. **Query stake-state (§2) cho cred preprod thật:**
   - `GET /stake-state/{stake1...}` → assert `reward_lovelace` == Blockfrost
     `withdrawable_amount` (đối chiếu trực tiếp Blockfrost account), `registered`,
     `vote_delegated` đúng thực tế. `GET /identity/{did}/stake-status` gộp đúng.
   - **Evidence:** JSON response + đối chiếu Blockfrost raw.

3. **Submit signed claim (§3) → tx_hash + LAMP về ví Phoenix:**
   - Core builder dựng claim tx (mode controller) → ký → `POST /claim/submit
     {signed_tx_cbor}` → `{claim_id, tx_hash, status:"SUBMITTED"}`.
   - Poll `GET /claim/{id}/status` → `CONFIRMED`; SSE phát `confirmed`.
   - **Assert on-chain:** `amount` LAMP tại `phoenix_address`; resubmit cùng tx →
     `409 ALREADY_SUBMITTED`. Ghi txid + LAMP balance.

4. **Revoke Grant → reject (§1):**
   - `POST /consent/revoke {grantor_did, nonce, sig}` → `200`.
   - `POST /consent/verify` cùng Grant → `{valid:false, reason:"revoked"}`.
   - `GET /consent/{grantor_did}` KHÔNG còn Grant đó (chỉ Grant sống).
   - **Evidence:** curl trước/sau revoke.

**Gate PASS:** cả 4 khối có evidence output thật (txid confirmed + LAMP balance +
curl trước/sau). Nếu §4 (cây LAMP) chưa publish → dùng snapshot 1-leaf tay cho §3
claim, ghi rõ mock (khớp Core §5.2). Compile pass ≠ PASS.

---

## §7. Ranh giới — cái gì THUỘC spec này, cái gì KHÔNG

**THUỘC OFFCHAIN (spec này, PR Long, `PhoenixKey-Database`):**
- B1 Consent/Grant store (§1) — 4 endpoint + DelegationToken store/verify/revoke.
- B2 Stake-state indexer (§2) — `/stake-state`, `/identity/{did}/stake-status` +
  Blockfrost account endpoints trong `BlockfrostHttpClient`.
- B3 Claim orchestration (§3) — `/claim/submit` + status + SSE (mirror Activation).
- B4 Merkle-proof serving (§4) — `/airdrop-claim/.../merkle-proof` (serve từ cây
  publish).
- Auth model (§5), error model chuẩn (Backend-Onchain §2.8), test Preview (§6).

**KHÔNG thuộc (tham chiếu / tiêu thụ input — KHÔNG build ở đây):**
- **Core (Tuân/Core, spec `PhoenixKey-Delegator-Core-Connector-Builders-Feat.md`):**
  CIP-30 Lace connector, `assemble_witness`, builder `taad_build_legacy_migration_tx`
  + `taad_build_delegator_lamp_claim_tx`, ký controller/Lace. Backend NHẬN CBOR
  đã-ký + CẤP `stake-state`/`proof`/`grant` làm input; KHÔNG dựng/không ký tx.
- **LAMP (đội LAMP — MESSAGE):**
  - Merkle-root SOURCE + **publish cây đầy đủ** (leaves theo `phoenix_addr` của
    delegator) + format/kênh phân phối cây (§4.3). Backend chỉ serve proof từ cây đó.
  - Luật ETD/ISPO/airdrop + `pool_ref` NFT unit + xác nhận leaf `address` = ví
    Phoenix. Chưa có → test dùng mock 1-leaf (§6).
- **Mobile/Enclave:** ký Grant (`/sign/*` có sẵn) + ký claim controller. Backend
  KHÔNG giữ Master_KEK (OFF-KEY-1).

**4 trục quyết định (mandate):**
- *Dài hạn:* endpoint theo chuẩn `/api/v1` + `DataResponse<T>` + Blockfrost mọi team
  Cardano gọi được; Grant store là hạ tầng uỷ-quyền mở, không khoá vào 1 luồng.
- *First-principles:* backend orchestrate + submit; KHÔNG giữ khoá, KHÔNG ký thay
  (Cardano không có AA → uỷ quyền = Lace-ký-từng-lần hoặc controller-ký sau migrate;
  Grant chỉ là bằng chứng, không phải khoá).
- *Tối ưu:* stake-state proxy Blockfrost account endpoint (không dựng indexer riêng
  cho MVP); claim orchestration tái dùng nguyên pipeline Activation (submit+poll+SSE);
  Merkle serve từ cây publish (không tính lại root).
- *User + bền vững:* backend-check chặn claim sai sớm (không đốt phí node), nhưng an
  toàn tài sản là validator + spend-once (§5.2); `409 ALREADY_CLAIMED`/`ALREADY_SUBMITTED`
  tránh tx chắc-FAIL; revoke Grant tức thì (không cache che thu hồi).
