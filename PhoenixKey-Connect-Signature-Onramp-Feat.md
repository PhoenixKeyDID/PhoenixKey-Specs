# PhoenixKey Connect — On-ramp uỷ quyền BẰNG CHỮ KÝ trình duyệt (Lace/Yoroi) — DEV-READY [N]

> **Một câu:** cho user ví ngoài (Lace/Yoroi) — kể cả người **mất seed** (còn mỗi
> extension đã mở khoá) hoặc **chưa tin PhoenixKey** — **ký MỘT mandate off-chain**
> qua `phoenix.me` hoặc site đối tác (vd `affiso.net`), **kiếm LAMP bằng chữ ký**
> (KHÔNG động tới tài sản Lace), xây niềm tin, rồi **tốt nghiệp** bằng migrate tài
> sản sang ví Phoenix. **KHÔNG trao seed. KHÔNG uỷ quyền chi on-chain.**
>
> **Đây là spec DELTA.** Cơ chế nền đã có spec riêng — spec này **CHỈ neo, KHÔNG
> chép lại**:
> - **CIP-30 Lace connector + builder (CORE, Tuân):**
>   `PhoenixKey-Delegator-Core-Connector-Builders-Feat.md` — §1 connector, §1.5
>   `signData` passthrough gỡ [OPEN-1], §2 `taad_build_legacy_migration_tx`, §3
>   `taad_build_delegator_lamp_claim_tx`, §4 signing model.
> - **Consent/Grant + claim off-chain (Long):**
>   `PhoenixKey-Delegator-Claim-Offchain-Feat.md` *(đang viết)* — consent store,
>   stake-state, merkle-proof, claim/submit/monitor. Spec này **tiêu thụ**, KHÔNG
>   build.
> - **Nguồn bất biến:** `PhoenixKey-Connector-CIP30-Feat-Math.md` (I-CONN-1…7,
>   B.3 ánh xạ method↔khoá, B.6 chống phát lại), `PhoenixKey-Permission-and-Consent-Spec.md`
>   (schema Grant §2, verify §4, revoke §5), `PhoenixKey-Build-Guide-Backend-Onchain.md`
>   §2.6 (endpoint consent), `LAMP/LAMP-DISTRIBUTION-SPEC.md` §2/§6 (Snapshot-Merkle).
>
> **Spec này ĐÓNG GÓP gì mới (delta thật):**
> 1. **Authorization Mandate** — schema + ràng buộc `wallet_address ↔ phoenix_did`,
>    domain-binding chống lừa (§1). *(mới)*
> 2. **Lightweight DID cho user ví-ngoài** — DID neo-địa-chỉ, không cần Secure
>    Enclave, để có `phoenix_did` gắn mandate + đích claim (§2). *(mới — không có
>    trong `/identity/register` hiện tại)*
> 3. **`taad_cose_sign1_verify`** — primitive Core VERIFY COSE_Sign1 server-side
>    (rust_core **CHƯA có**; §7 Core). *(mới)*
> 4. **Partner-site embed SDK/widget** (`affiso.net`) — token-exchange scoping tái
>    dùng ServiceDID/JWKS (§3). *(mới ở tầng SDK; hạ tầng đã có)*
>
> **4 trục quyết định (mandate anh Aladin):** ghi rải trong từng §; tổng hợp §9.

---

## §0. VERIFY — cái gì ĐÃ CÓ vs MỚI (đọc code/spec thật, KHÔNG nhớ)

| Mảnh | Trạng thái thật (file:vị trí) | Spec này |
|---|---|---|
| `signData` = Ed25519 **COSE_Sign1 (CIP-8)**, KHÔNG P-256 | ĐÃ CHỐT: `PhoenixKey-Connector-CIP30-Feat-Math.md` B.3:227 + I-CONN-6:309 | tiêu thụ |
| `signData` Phase-1 = **passthrough Lace** (Lace dựng COSE) | ĐÃ CHỐT: `Delegator-Core...Feat.md` §1.5:204-235 (gỡ [OPEN-1]) | tiêu thụ; §1 dùng để **thu** mandate |
| **Verify COSE_Sign1 server-side** (Ed25519, addr↔pubkey↔sig) | 🔴 **KHÔNG có** trong rust_core: `sign.rs` chỉ `sign_ed25519`/`derive_taad_public_key`; `lib.rs:265` chỉ `taad_verify_p256_signature`. KHÔNG có Ed25519-verify, KHÔNG có COSE-verify | 🟢 **MỚI §7 Core** |
| Grant/Consent schema | `Permission-and-Consent-Spec §2` + `Connector B.2:172-190` | tái dùng (mandate = Grant kiểu mới) |
| Endpoint consent (`PUT/GET/revoke/verify-grant`) | `Build-Guide §2.6:148-159` | tái dùng (Long) |
| `POST /identity/register` = mint **PersonDID** từ HW_Key (Secure Enclave) | 🟢 live: `API-Catalog A1:14`; `ServiceDID...DRAFT:32` xác nhận verify Genesis sig | KHÔNG dùng — user ví ngoài **không có Secure Enclave** |
| DID **nhẹ/custodial** cho user ví-ngoài | 🔴 **KHÔNG có** endpoint | 🟢 **MỚI §2 (Long)** |
| `/auth/token/exchange` (session→app_token EdDSA JWT theo ServiceDID) + `/.well-known/jwks.json` | 🟢 live: `API-Catalog A7:51, A11:73`; `ServiceDID...DRAFT:29-30` (aud=ServiceDID, redirect_uri ∈ serviceEndpoint) | tái dùng §3 |
| Migrate tài sản (builder 4-trong-1) | `Delegator-Core...Feat.md` §2 `taad_build_legacy_migration_tx` (🔴 code thiếu, spec xong) | tham chiếu §5 |
| LAMP credit = **Snapshot-Merkle** permissionless, leaf `(owner,amount,epoch)`, nullifier NFT | `LAMP-DISTRIBUTION-SPEC §2:28 + §6.1-6.3` | tiêu thụ §2 |
| Claim LAMP về `phoenix_addr` builder | `Delegator-Core...Feat.md` §3 `taad_build_delegator_lamp_claim_tx` | tham chiếu §2/§5 |
| Yoroi cùng surface CIP-30 (`signData`/`signTx`) | Yoroi expose `window.cardano.yoroi` CIP-30/CIP-8 như Lace | connector §1.3 `walletKey` mở rộng "yoroi" |

**Kết luận verify:** on-ramp KHẢ THI trên nền đã spec, nhưng cần **2 mảnh Core/Long
mới**: (a) `taad_cose_sign1_verify` (Core), (b) lightweight-DID + mandate-store +
program-crediting (Long). Phần còn lại là **điều phối** connector/consent/token-
exchange đã có spec.

---

## §1. Authorization Mandate — uỷ quyền THẨM QUYỀN off-chain (KHÔNG uỷ quyền chi)

### §1.1 Bản chất (first-principles — KHÔNG được nói mập mờ)

Cardano **KHÔNG có account-abstraction** → **không có "uỷ quyền khoá ký lâu dài"**.
`signData` (CIP-8/COSE_Sign1) chứng minh **user kiểm soát khoá TẠI THỜI ĐIỂM ký** —
KHÔNG cho phép PhoenixKey **chi UTxO** của họ. Mandate vì thế là **uỷ quyền THẨM
QUYỀN off-chain**: "PhoenixKey được phép ghi có LAMP cho tôi / tính tôi tham gia
program X — dựa trên bằng chứng tôi kiểm soát địa chỉ này."

- **CHỨNG MINH được (crypto):** `address ↔ pubkey ↔ signature` khớp → người ký kiểm
  soát khoá chi của `wallet_address` tại lúc ký (đủ để coi là "chủ địa chỉ").
- **KHÔNG chứng minh / KHÔNG cho phép:** chi UTxO, ký tx thay, "auto-sign mãi mãi",
  hay bất cứ hành động on-chain nào. Mandate KHÔNG phải chữ ký tx (I-CONN-1).

→ Mandate = **Grant** (tái dùng schema `Permission §2`) với `scope="lamp_program"`,
**thu bằng `signData`** (Lace/Yoroi dựng COSE_Sign1, passthrough theo Delegator-Core
§1.5). PhoenixKey **verify** COSE server-side (§7 Core mới).

### §1.2 Schema `AuthorizationMandate` (là một Grant kiểu mới)

```
AuthorizationMandate ≜ Grant {
  wallet_address : Bech32Addr,   -- địa chỉ Lace/Yoroi (payment addr đã ký)
  phoenix_did    : DID,          -- lightweight DID của user (§2) — grantor
  scope          : "lamp_program",
  program_ids    : List<String>, -- vd ["etd-2026q3","affiso-referral-01"]; [] = mọi program của issuer
  nonce          : Bytes32,      -- server cấp, chống phát lại (Connector B.6)
  valid_from     : UnixTime,
  valid_until    : UnixTime,     -- BẮT BUỘC hữu hạn (khuyến nghị ≤ 90 ngày)
  revocable      : true,         -- luôn true (I-CONN-3)
  domain         : Origin,       -- "https://phoenix.me" | "https://affiso.net" — §1.4
  -- ── phần chữ ký (COSE_Sign1, KHÔNG phải Ed25519 raw controller) ──
  cose_sign1     : Bytes,        -- COSE_Sign1 do ví ngoài trả (chứa payload + sig + key)
}
```

**Payload được ký** (nội dung `signData(addr, payload)` — CIP-8 message body):

```
mandate_payload = canonical_json({
  "t"    : "phoenixkey.mandate.v1",     -- domain-separation nhãn kiểu
  "addr" : wallet_address,
  "did"  : phoenix_did,
  "scope": "lamp_program",
  "progs": program_ids,
  "nonce": nonce_hex,
  "iat"  : valid_from,
  "exp"  : valid_until,
  "dom"  : domain                        -- origin ràng buộc — §1.4
})
```

- `signData` được gọi với `addr = wallet_address` (payment/stake addr của ví ngoài),
  `payload = utf8(mandate_payload)`. Ví ngoài (Lace/Yoroi) bọc thành **COSE_Sign1**
  (CIP-8), gắn `COSE_Key` (pubkey Ed25519). Trả `{signature, key}` (shape
  `CoseSign1{signatureHex, keyHex}` — Delegator-Core §1.3).
- **KHÔNG dùng controller Ed25519 ký mandate.** Người ký = **khoá ví ngoài**, không
  phải PhoenixKey. Đây là điểm khác Grant thường (Grant thường controller ký; mandate
  do chủ-địa-chỉ-ngoài ký).

### §1.3 Ràng buộc `wallet_address ↔ phoenix_did` (verify — §7 Core làm)

`verify_mandate(mandate)` (server, Long gọi Core `taad_cose_sign1_verify`):

```
verify_mandate(m):
  1. (payload, sig, pubkey) = cose_sign1_parse(m.cose_sign1)     -- Core §7
  2. require payload == canonical(m.mandate_payload)             -- không tráo nội dung
  3. require cose_verify(pubkey, payload, sig) == OK             -- Ed25519 COSE_Sign1
  4. require addr_matches_pubkey(m.wallet_address, pubkey)       -- addr↔pubkey (§7.2)
        -- payment_cred(addr) == blake2b_224(pubkey)  (base/enterprise addr)
  5. require m.domain ∈ ALLOWED_ORIGINS                          -- §1.4
  6. require valid_from ≤ now ≤ valid_until                      -- còn hạn
  7. require lookup_nonce(m.nonce).status == Unused              -- chống phát lại (B.6)
  8. bind:  store Grant{grantor=m.phoenix_did, wallet_address=m.wallet_address, ...}
        -- một phoenix_did có thể gắn NHIỀU wallet_address (nhiều ví); mỗi cặp 1 mandate
```

**Bước 4 là mấu chốt "bind":** nó chứng minh khoá ký = khoá chi của
`wallet_address`. Kết hợp bước 8, ta gắn cứng `wallet_address` với `phoenix_did` do
chính chủ khoá ngoài khẳng định. **KHÔNG suy `phoenix_did` từ đâu khác** — user tự
nêu trong payload và tự ký ⇒ tự-khẳng-định-có-kiểm-chứng.

### §1.4 Chống lừa/phát lại — domain-binding + nonce + hạn (bám Connector B.6)

- **Domain-binding (`dom` trong payload + `domain` trong Grant):** payload chứa
  origin của site gọi (`phoenix.me` hoặc `affiso.net`). Nếu site lừa ở origin khác
  cố dụ user ký, `domain ∉ ALLOWED_ORIGINS` → verify từ chối (bước 5). Origin **do
  server neo** khi phát nonce (§3.3), KHÔNG để client tự khai tuỳ ý.
- **Nonce một-lần (`nonce`):** server cấp, giữ tập đã-dùng tới `exp`; trùng →
  `NONCE_USED` (Connector B.6, UI §20). Chặn phát lại một mandate đã ký.
- **Hạn (`valid_until`):** bắt buộc; quá hạn → mandate vô hiệu, phải ký lại.
- **Nhãn kiểu (`t="phoenixkey.mandate.v1"`):** domain-separation — chữ ký mandate
  KHÔNG trùng dạng bất kỳ message login/tx nào (chống chữ-ký-đa-dụng).

### §1.5 Bất biến §1

- **CONNECT-M1:** mandate KHÔNG cấp quyền chi on-chain; chỉ thẩm quyền off-chain
  (ghi-có LAMP / tính-tham-gia). Vi phạm = bug.
- **CONNECT-M2:** người ký mandate = **khoá ví ngoài** (COSE_Sign1 Ed25519), KHÔNG
  phải controller PhoenixKey.
- **CONNECT-M3:** verify PHẢI khớp `addr ↔ pubkey ↔ sig` (§1.3 bước 3-4); thiếu 1
  → từ chối.
- **CONNECT-M4:** mọi mandate có `domain`+`nonce`+`valid_until`; thu-hồi-được
  (`revocable=true`), liệt-kê-được (dashboard quyền — I-CONN-5).

---

## §2. Kiếm LAMP BẰNG CHỮ KÝ — luồng ghi-có / claim (KHÔNG động tài sản Lace)

### §2.1 Lightweight DID cho user ví-ngoài (MỚI — Long)

User ví ngoài **không có Secure Enclave** → KHÔNG mint được PersonDID qua
`/identity/register` (endpoint đó verify Genesis sig từ HW_Key). Cần một
**`phoenix_did` nhẹ** để (a) gắn mandate, (b) làm đích ghi-có LAMP trước khi họ có
ví Phoenix đầy đủ.

```
POST /connect/did/lightweight
  body: { wallet_address, cose_sign1 }     -- ký "phoenixkey.did.claim.v1" theo addr
  → { phoenix_did }                        -- did:phoenix:ext:<blake2b(wallet_address)>
```

- **Neo bằng địa chỉ, không neo Secure Enclave.** `phoenix_did` dẫn xuất tất định
  từ `wallet_address` (`did:phoenix:ext:<blake2b_224(payment_cred)>`) → cùng ví →
  cùng DID (idempotent). Controller ban đầu = **chính khoá ví ngoài** (verify qua
  COSE_Sign1, cùng primitive §7).
- **Custodial-nhẹ, KHÔNG giữ khoá:** backend KHÔNG giữ private key (đúng bất biến
  non-custodial `Mainnet-Deploy-Runbook:67`). Nó chỉ ghi một bản ghi DID nhẹ +
  controller = pubkey ví ngoài. Mọi thao tác cần chữ ký → COSE_Sign1 của ví ngoài.
- **Nâng cấp về PersonDID đầy đủ:** khi user tốt nghiệp (§5), lightweight DID
  **merge/rotate** controller sang controller PhoenixKey (Master_KEK-derived) — hoặc
  user tạo PersonDID mới và mandate/LAMP re-point. Chi tiết rotate = Long (tiêu thụ
  event `controller_changed`, Build-Guide §2.7).

> **4 trục:** *dài hạn* — DID là LỚP, ai cũng vào được kể cả chưa có app; *first-
> principles* — DID nhẹ = định danh neo-địa-chỉ, không giả vờ có sinh-trắc;
> *tối ưu* — tất định từ address, idempotent, không mint on-chain cho tới khi cần;
> *user* — vào được ngay bằng ví sẵn có, không bắt cài app/xuất seed.

### §2.2 Ghi-có LAMP theo program/epoch — hai chế độ

Với mỗi program (`program_id`) và epoch, PhoenixKey xác nhận user **vẫn kiểm soát**
`wallet_address` bằng MỘT trong hai:

- **Chế độ A — mandate-đủ (stored mandate):** mandate còn hạn + program_id ∈
  `program_ids` + chưa thu hồi ⇒ đủ để tính tham gia epoch đó. Không cần user ký lại.
  Rẻ, mượt; hợp program dài hạn.
- **Chế độ B — thách-thức tươi (fresh signData challenge):** program đòi re-prove
  mỗi epoch (chống mượn-mandate-cũ). Server cấp `challenge` (nonce+epoch), user ký
  `signData` nhẹ (`"phoenixkey.epoch.proof.v1"` + epoch + program_id). Verify như §1.3.

Chọn A/B là **điều khoản của program** (LAMP team quyết — §ngoài-phạm-vi §2.4).

### §2.3 Ghi-có ra sao — bám LAMP-DISTRIBUTION Snapshot-Merkle

LAMP ghi-có = **Snapshot-Merkle permissionless** (`LAMP-DISTRIBUTION §2:28, §6`):
keeper/committee dựng cây Merkle `(owner, amount, epoch)` → post root → claim bằng
proof. **PhoenixKey Connect đóng vai:**

1. Sau mỗi epoch, PhoenixKey xuất cho LAMP snapshot **eligible participants**:
   `{ phoenix_did, claim_address, program_id, weight/points }` — trong đó
   `claim_address` = **`phoenix_addr(phoenix_did)`** (đích claim = ví Phoenix, dù
   user chưa migrate; `derive_phoenix_address`).
2. LAMP dựng leaf `(claim_address, amount, epoch)` + post root (LAMP làm — ngoài phạm
   vi Core/Long ở đây).
3. User **claim** bằng `taad_build_delegator_lamp_claim_tx` (Delegator-Core §3) →
   LAMP về `phoenix_addr`. **Tài sản Lace KHÔNG bị đụng** ở bất kỳ bước nào.

> **Điểm cốt lõi:** LAMP luôn route về **địa chỉ Phoenix**, KHÔNG về địa chỉ Lace.
> Nên user "kiếm LAMP bằng chữ ký" nhưng phần thưởng nằm trong quỹ đạo Phoenix —
> tạo lực kéo tự nhiên để tốt nghiệp (§5). Trước khi có app đầy đủ, LAMP đợi ở
> `phoenix_addr` (script did_payment) tới khi controller ký claim/redeem được.

### §2.4 Eligibility / anti-sybil = ĐIỀU KHOẢN PROGRAM (ngoài phạm vi — message LAMP)

Ai đủ điều kiện, bao nhiêu LAMP, chống nhiều-ví-một-người ra sao = **điều khoản
program LAMP**, KHÔNG phải spec này. PhoenixKey Connect chỉ cung cấp **bằng chứng
kiểm-soát-địa-chỉ + gắn phoenix_did**; luật thưởng do LAMP team.

> **Message to LAMP team:** Connect cấp snapshot `{phoenix_did, claim_address=phoenix_addr,
> program_id, proof-of-control epoch}`. Cần LAMP xác nhận: (a) đơn vị points/weight per
> program; (b) chọn chế độ A (mandate-đủ) hay B (re-prove mỗi epoch); (c) luật
> anti-sybil/eligibility (nhiều ví ngoài → một người?). Nhắc memory `no-sybil-concern`:
> **KHÔNG áp lo ngại sybil lên PersonDID sinh-trắc**; nhưng ví-ngoài lightweight
> KHÔNG có sinh-trắc → nếu program cần dedup, đó là **luật kinh tế của LAMP program**,
> không phải bất biến định-danh.

### §2.5 Bất biến §2

- **CONNECT-E1:** ghi-có LAMP KHÔNG bao giờ đọc/chi UTxO của `wallet_address` (Lace);
  chỉ đọc watch-only + verify chữ ký.
- **CONNECT-E2:** LAMP luôn route về `phoenix_addr(phoenix_did)`, KHÔNG về addr Lace.
- **CONNECT-E3:** eligibility/points/anti-sybil = program terms (LAMP), không hard-code
  ở Connect.

---

## §3. Nhúng site đối tác (`affiso.net`) — SDK/widget + token-exchange scoping

### §3.1 Việc làm

Site bên thứ ba (vd `affiso.net`) nhúng **PhoenixKey Connect widget**: nút "Kết nối &
ký nhận LAMP" mở luồng CIP-30 connect + ký mandate **scoped cho site đó**. Site nhận
**bằng chứng có phạm vi** (scoped proof / app_token) mà **KHÔNG bao giờ thấy khoá**.

### §3.2 Kiến trúc — tái dùng ServiceDID + token-exchange + JWKS (đã live)

Mỗi partner site đăng ký **ServiceDID** (vd `did:phoenix:service:affiso`) với
`serviceEndpoint[]` chứa origin `https://affiso.net` (mô hình `ServiceDID...DRAFT`).
Widget dùng hạ tầng **đã live** (`API-Catalog A7/A11`):

```
Partner site (affiso.net)                PhoenixKey
  │ 1. nhúng <script> widget                 (ALLOWED_ORIGINS ⊇ affiso.net qua ServiceDID)
  │ 2. user bấm Connect → widget mở popup phoenix.me?origin=affiso.net&sid=<ServiceDID>
  │ 3. connect Lace/Yoroi (CIP-30) + ký mandate (§1, domain="https://affiso.net")
  │ ◄── 4. session_token (phiên connect thành công) ──
  │ 5. POST /auth/token/exchange {session_token, aud=did:phoenix:service:affiso}
  │ ◄── 6. app_token (EdDSA JWT, kid=phoenixkey-ed25519-1) scoped aud=affiso ──
  │ 7. affiso.net verify app_token qua GET /.well-known/jwks.json (KHÔNG thấy khoá)
```

- **`/auth/token/exchange`** (live): validate `redirect_uri`/origin ∈
  `serviceEndpoint[]` của ServiceDID on-chain → cấp `app_token` scoped `aud=affiso`.
  Nguồn sự thật app registry = **Cardano**, không phải DB riêng (ServiceDID...DRAFT:29).
- **JWKS** (live `/.well-known/jwks.json`): affiso verify chữ ký `app_token` bằng
  public key EdDSA — **không cần bí mật** ⇒ site đối tác không bao giờ chạm khoá.
- **Domain-binding của mandate (§1.4)** khớp origin ServiceDID: mandate ký ở
  `affiso.net` chỉ dùng cho program của affiso (scope theo `sid`).

### §3.3 Cấp nonce + neo origin (server-side, chống origin giả)

```
POST /connect/challenge
  body: { origin, service_did (nếu partner), program_ids }
  → { nonce, valid_until }        -- server ràng origin vào nonce; client KHÔNG tự khai dom
```

Verify (§1.3 bước 5) đối chiếu `payload.dom == origin-đã-neo-với-nonce`. Nếu partner:
origin PHẢI ∈ `serviceEndpoint[]` của `service_did` → chống site giả mạo affiso.

### §3.4 Ranh giới widget (đừng biến affiso thành ví)

Widget CHỈ: mở CIP-30 connect + thu mandate + đổi token có phạm vi. KHÔNG dựng tx,
KHÔNG giữ khoá, KHÔNG thấy UTxO của user (partner chỉ nhận app_token + trạng thái
"đã ký mandate"). Mọi bí mật ở ví ngoài + thiết bị user (I-CONN-1/2, B.6).

### §3.5 Bất biến §3

- **CONNECT-P1:** partner site nhận app_token scoped `aud=ServiceDID`, verify qua
  JWKS; KHÔNG bao giờ thấy private key / seed / Master_KEK.
- **CONNECT-P2:** origin neo server-side vào nonce + đối chiếu `serviceEndpoint[]`
  on-chain; origin giả → từ chối (§3.3).
- **CONNECT-P3:** widget không dựng tx / không giữ khoá / không đọc UTxO user.

---

## §4. Ca "mất seed / còn mỗi extension" — HOẠT ĐỘNG, kèm cảnh báo rõ

### §4.1 Vì sao vẫn chạy

Extension Lace/Yoroi **đã mở khoá** vẫn **ký được** `signData` (mandate) và **một tx
migrate một-lần** (`signTx`) — kể cả khi user **mất 24 từ seed**. Miễn extension còn
sống + mở khoá, khoá chi còn dùng được. Nên:

- **Ký mandate (§1):** chạy — extension ký COSE_Sign1.
- **Migrate một-lần (§5):** chạy — extension ký `taad_build_legacy_migration_tx`
  (Delegator-Core §2), **một chữ ký** gộp 4 việc (vote-deleg + withdraw + move +
  dereg). Sau tx này, tài sản ở `phoenix_addr`, **không còn phụ thuộc extension**.

### §4.2 Cảnh báo UX BẮT BUỘC (Frontend — không được bỏ)

> **"Bạn đang dùng ví KHÔNG có bản sao lưu seed. Chừng nào extension này còn mở, bạn
> vẫn chuyển được tài sản sang Ví Phượng hoàng. Nếu extension bị gỡ / máy hỏng / xoá
> dữ liệu trình duyệt, khoá MẤT VĨNH VIỄN và tài sản không lấy lại được. Hãy migrate
> NGAY khi extension còn sống."**

- Hiện cảnh báo này ở màn connect **khi phát hiện** user không xác nhận có seed
  (hoặc luôn hiện cho ca extension-only). KHÔNG dùng ngôn ngữ hù doạ sai; nói đúng
  rủi ro kỹ thuật.
- **Không bắt xuất seed** (memory `no-mandatory-seed-export`): ta KHÔNG yêu cầu user
  nhập 24 từ; ta khuyên **migrate tài sản** trong khi extension còn ký được.

### §4.3 Bất biến §4

- **CONNECT-L1:** extension-only vẫn ký mandate + một-tx-migrate; sau migrate không
  còn phụ thuộc extension.
- **CONNECT-L2:** UX PHẢI cảnh báo rõ "migrate khi extension còn sống, nếu không mất
  khoá vĩnh viễn" trước khi user rời luồng.

---

## §5. Tốt nghiệp — migrate tài sản sang ví Phoenix (controller tiếp quản)

Đây là bước **một-lần về sau** (không bắt buộc để kiếm LAMP; là đích của on-ramp).

- **Builder:** `taad_build_legacy_migration_tx` (Delegator-Core **§2** — KHÔNG lặp
  lại ở đây). Gộp 4-trong-1 Conway: VoteDelegation (gỡ Plomin) → Withdrawal (tự cấp
  phí) → Move phần dư về `phoenix_addr` → StakeDeregistration (hoàn 2 ADA). Lace/Yoroi
  ký MỘT lần (`signTx`), Core trả **UNSIGNED**, `assemble_witness` ghép.
- **Sau migrate:** tài sản ở `phoenix_addr` (script `did_payment`) → **controller
  PhoenixKey ký** mọi thao tác sau (đây mới là "ký thay" thật, lâu dài —
  Delegator-Core §4 CORE-SIGN-2). LAMP đã claim/đang chờ ở `phoenix_addr` giờ
  controller redeem được.
- **Nghỉ hưu mandate:** sau khi user có PersonDID/ví Phoenix đầy đủ + controller
  tiếp quản, mandate ví-ngoài có thể **thu hồi** (`POST /consent/revoke`) — không còn
  cần thẩm-quyền-off-chain vì user đã ở trong quỹ đạo Phoenix. Lightweight DID
  rotate/merge controller (§2.1).

### §5.1 Bất biến §5

- **CONNECT-G1:** migrate = một tx Lace/Yoroi ký một lần (Delegator-Core §2 invariants
  CORE-MIGR-*); Core KHÔNG tự ký (không có seed cũ).
- **CONNECT-G2:** sau tốt nghiệp, controller PhoenixKey tiếp quản; mandate ví-ngoài
  retire-được.

---

## §6. Ranh giới TRUNG THỰC + an ninh

- **KHÔNG uỷ quyền chi on-chain âm thầm.** Cardano không có account-abstraction;
  mandate = thẩm quyền OFF-CHAIN thuần. PhoenixKey KHÔNG bao giờ chi UTxO Lace/Yoroi
  bằng mandate. Mọi chi on-chain vẫn cần chữ ký tươi của chủ khoá (Lace `signTx`)
  hoặc — sau migrate — controller Phoenix.
- **Thu hồi:** `POST /consent/revoke` (Build-Guide §2.6) với `grant_nonce` + chữ ký.
  Với mandate ví-ngoài, revoke ký bằng **khoá ví ngoài** (COSE_Sign1) hoặc — nếu đã
  tốt nghiệp — controller. Trạng thái SỐNG (revoke tức thì, không cache vượt hạn —
  Connector B.4).
- **Nếu khoá ví ngoài bị lộ về sau:** kẻ tấn công CHỈ có thể (a) ký mandate mới (=
  ghi-có LAMP về `phoenix_addr` của **chính DID đó** — không rút được đi đâu khác vì
  claim luôn route `phoenix_addr`, CONNECT-E2), và (b) chi UTxO Lace **chưa migrate**
  (đây là rủi ro của ví ngoài, KHÔNG phải do Connect tạo ra — Connect không mở rộng
  bề mặt tấn công chi tiền). → **Khuyến nghị:** migrate sớm; sau migrate, tài sản ở
  `phoenix_addr` do controller Phoenix canh (khoá lộ của ví ngoài không đụng được).
- **Mandate KHÔNG phải bearer-token:** verify luôn resolve trạng thái sống + hạn +
  nonce; một COSE_Sign1 bị đánh cắp không tái dùng được (nonce một-lần, exp).
- **Bí mật không rời thiết bị:** qua widget/connector chỉ đi dữ liệu công khai
  (address, COSE_Sign1, app_token, tx-chưa-ký). KHÔNG seed/xprv/Master_KEK (I-CONN-1/2,
  Connector B.6).

### §6.1 Bất biến §6

- **CONNECT-S1:** không có đường nào để mandate dẫn tới chi UTxO ví ngoài. Vi phạm = bug.
- **CONNECT-S2:** khoá ví-ngoài lộ ⇒ tối đa ghi-có LAMP về chính `phoenix_addr` của
  DID đó (không đổi hướng) + rủi ro chi-Lace-chưa-migrate (vốn có, không do Connect).
- **CONNECT-S3:** mandate thu-hồi-được tức thì; verify luôn đọc trạng thái sống.

---

## §7. Ranh giới build — phân loại từng mảnh (Frontend / Long / Core)

### §7.1 CORE (`rust_core`, Tuân) — **MẢNH MỚI QUAN TRỌNG NHẤT**

**`taad_cose_sign1_verify` — CHƯA CÓ trong rust_core, PHẢI viết.** (verify §0: `sign.rs`
chỉ ký; `lib.rs:265` chỉ verify P-256; không có Ed25519-verify/COSE-verify.)

```rust
// rust_core/src/connector.rs (hoặc cose.rs mới)

/// Parse + verify một COSE_Sign1 (CIP-8) do ví ngoài (Lace/Yoroi) trả.
/// Trả về payload + pubkey nếu chữ ký hợp lệ; "" nếu sai (KHÔNG panic qua FFI).
/// KHÔNG P-256 — COSE_Sign1 CIP-8 dùng Ed25519 (I-CONN-6).
pub fn cose_sign1_verify(cose_sign1_hex: &str) -> Option<CoseVerified>;
// CoseVerified { payload_hex: String, pubkey_hex: String }  (sig đã verify OK)

/// Kiểm addr (bech32) khớp pubkey: payment_cred(addr) == blake2b_224(pubkey).
/// Hỗ trợ base_addr + enterprise_addr + reward(stake) addr.
pub fn addr_matches_pubkey(addr_bech32: &str, pubkey_hex: &str) -> bool;
```

```rust
#[no_mangle] pub unsafe extern "C" fn taad_cose_sign1_verify(
    cose_sign1_hex: *const c_char,
) -> *mut c_char;   // JSON {"ok":bool,"payload_hex":..,"pubkey_hex":..} ; null nếu lỗi

#[no_mangle] pub unsafe extern "C" fn taad_addr_matches_pubkey(
    addr_bech32: *const c_char, pubkey_hex: *const c_char,
) -> u8;            // 0/1
```

- Dùng crate COSE (vd `coset`) + `ed25519-dalek` (đã có trong `sign.rs`) để verify;
  blake2b_224 đã có (`phoenix_address.rs:39`, `Blake2b224`).
- **Đây KHÁC builder COSE `taad_cose_sign1_build`** (Delegator-Core §1.5 Phase-2, để
  PhoenixKey LÀ bên ký). Connect cần **VERIFY** (PhoenixKey là bên **nhận** chữ ký ví
  ngoài), builder KHÔNG cần cho Connect.
- Tái dùng (KHÔNG viết lại): `derive_phoenix_address` (`phoenix_address.rs`),
  migration/claim builder (Delegator-Core §2/§3), `assemble_witness` (Delegator-Core §1.4).

**Bảng endpoint/FFI Core:**

| FFI | Trạng thái | Ghi chú |
|---|---|---|
| `taad_cose_sign1_verify` | 🟢 **MỚI (§7.1)** | verify mandate/challenge/lightweight-DID sig |
| `taad_addr_matches_pubkey` | 🟢 **MỚI (§7.1)** | ràng addr↔pubkey (bind §1.3) |
| `taad_build_legacy_migration_tx` | 🔴 reuse (Delegator-Core §2) | migrate §5 |
| `taad_build_delegator_lamp_claim_tx` | 🔴 reuse (Delegator-Core §3) | claim §2.3 |
| `assemble_witness` / `cip30_utxos_to_builder_json` | 🔴 reuse (Delegator-Core §1.4) | connector |
| `signData` passthrough (Lace dựng COSE) | reuse (Delegator-Core §1.5) | thu mandate |

### §7.2 LONG (backend/SDK) — mandate store + program crediting + token scoping

| Endpoint | Trạng thái | Ghi chú |
|---|---|---|
| `POST /connect/challenge` | 🟢 **MỚI (§3.3)** | cấp nonce + neo origin (partner: check `serviceEndpoint[]`) |
| `POST /connect/mandate` | 🟢 **MỚI (§1)** | nhận mandate → gọi Core `taad_cose_sign1_verify` → lưu Grant |
| `POST /connect/did/lightweight` | 🟢 **MỚI (§2.1)** | tạo `did:phoenix:ext:*` neo-địa-chỉ (KHÔNG giữ khoá) |
| `POST /connect/epoch-proof` | 🟢 **MỚI (§2.2 chế độ B)** | thu challenge tươi mỗi epoch (tuỳ program) |
| `GET /connect/snapshot/{program_id}/{epoch}` | 🟢 **MỚI (§2.3)** | xuất `{phoenix_did, phoenix_addr, points}` cho LAMP |
| `PUT /consent/{grantor_did}` · `GET /consent/...` · `POST /consent/revoke` · `GET /verify-grant` | 🔴 reuse (Build-Guide §2.6) | mandate = Grant; revoke §6 |
| `POST /auth/token/exchange` · `GET /.well-known/jwks.json` | 🟢 live reuse (§3.2) | partner scoped app_token |
| `GET /stake-state/{cred}` · `GET /airdrop-claim/.../merkle-proof` · `POST /claim/submit` | 🔴 reuse (Delegator-Claim-Offchain, Long) | migrate/claim off-chain |

> `POST /connect/mandate` **gọi Core** `taad_cose_sign1_verify` + `taad_addr_matches_pubkey`
> để verify (Long KHÔNG tự viết crypto). Long lo store + trạng-thái-sống + snapshot.

### §7.3 FRONTEND (`phoenix.me` connect UI + affiso SDK widget)

| Mảnh | Trạng thái | Ghi chú |
|---|---|---|
| Màn connect `phoenix.me` (chọn Lace/Yoroi → `enable` → ký mandate) | 🟢 **MỚI** | dùng `LaceConnector` (Delegator-Core §1.3), thêm `walletKey="yoroi"` |
| Cảnh báo extension-only (§4.2) | 🟢 **MỚI** | text bắt buộc; không hù doạ sai |
| Nút "Migrate sang Ví Phượng hoàng" (§5) | 🟢 **MỚI (điều phối)** | gọi builder Delegator-Core §2 + signTx + assemble + submit |
| Dashboard mandate (liệt kê + thu hồi — I-CONN-5) | 🟢 **MỚI** | tái dùng S11 Quyền-đã-cấp (Connector C.3) |
| **affiso SDK widget** (`<script>` nhúng → popup connect + mandate + token-exchange) | 🟢 **MỚI** | scoped theo ServiceDID; verify app_token qua JWKS |
| i18n: `pk.connect.*`, `pk.mandate.*`, `pk.warn.extension_only` | 🟢 **MỚI** | Terminology spec |

---

## §8. Test plan (Preview) — evidence output thật (mandate anh Aladin)

**Cần:** 1 ví Lace **testnet** (preprod/preview) có reward tích; 1 ví Yoroi testnet;
backend Long chạy endpoint `/connect/*`; Core build có `taad_cose_sign1_verify`.

### §8.1 Rust unit (offline, mirror `sign.rs`/`staking.rs` `#[cfg(test)]`)

- `cose_sign1_verify_accepts_valid` — dựng COSE_Sign1 hợp lệ (ký Ed25519 test vector)
  → `ok=true`, payload/pubkey khớp.
- `cose_sign1_verify_rejects_tampered` — sửa 1 byte payload → `ok=false`.
- `cose_sign1_verify_rejects_p256` — nạp COSE ES256/P-256 → từ chối (I-CONN-6).
- `addr_matches_pubkey_base_and_enterprise` — addr dựng từ pubkey X → `true`; addr
  của pubkey Y → `false`.
- `lightweight_did_deterministic` — cùng `wallet_address` → cùng `did:phoenix:ext:*`.

### §8.2 Preview e2e (field test — 2 txid confirmed + assert, KHÔNG chỉ compile)

1. **Connect + mandate:** mở `phoenix.me` → chọn Lace testnet → `enable` →
   `POST /connect/challenge` lấy nonce → ký mandate (`signData`) → `POST /connect/mandate`.
   **Assert server:** `taad_cose_sign1_verify` OK; `addr↔pubkey` khớp; Grant lưu với
   `wallet_address ↔ phoenix_did` đúng; nonce chuyển Used.
2. **Kiếm LAMP:** đóng epoch → `GET /connect/snapshot` xuất user → LAMP dựng leaf +
   root (mock 1-leaf nếu LAMP chưa sẵn, ghi rõ) → `taad_build_delegator_lamp_claim_tx`
   (controller mode) → submit. **Assert on-chain:** `amount` LAMP tại
   `phoenix_addr(phoenix_did)`; **KHÔNG** tx nào chi từ addr Lace (CONNECT-E1). Ghi
   txid + LAMP balance.
3. **Claim về ví Phoenix:** xác nhận LAMP redeemable ở `phoenix_addr` (controller ký).
4. **Revoke:** `POST /connect/revoke` (ký khoá ví ngoài) → lệnh kế dùng mandate →
   `verify-grant` trả `valid=false` (revoke tức thì, CONNECT-S3).
5. **Wrong-origin:** ký mandate với `dom="https://evil.example"` (không neo với nonce)
   → `POST /connect/mandate` từ chối `ORIGIN_MISMATCH` (CONNECT-M4/§1.4).
6. **Yoroi:** lặp bước 1 với Yoroi testnet → mandate verify OK (cùng COSE_Sign1 path).
7. **Partner (affiso mô phỏng):** đăng ký ServiceDID test với `serviceEndpoint` =
   origin test → widget flow → `token/exchange` trả app_token `aud=affiso` → verify
   qua JWKS OK; origin giả → từ chối (CONNECT-P2).

**Gate PASS:** bước 2-3 có txid confirmed + LAMP balance thật ở `phoenix_addr`; bước
4-5 reject đúng; bước 7 app_token verify đúng scope. Nếu LAMP snapshot/proof chưa sẵn
→ mock 1-leaf, ghi rõ là mock.

---

## §9. Tổng hợp 4 trục quyết định (mandate)

- **Định hướng dài hạn:** on-ramp mở — bất kỳ user Cardano (Lace/Yoroi) vào được hệ
  PhoenixKey/LAMP **không cần xuất seed, không cần cài app trước**; DID là LỚP. Hạ
  tầng (token-exchange/JWKS/ServiceDID) mở cho mọi partner (affiso và sau này). Hướng
  tới **LAMP có giá trị** = có người thật kiếm LAMP + dần tốt nghiệp vào Phoenix.
- **First-principles:** phân biệt sạch **thẩm-quyền-off-chain (mandate)** vs
  **chữ-ký-tx-on-chain**; Cardano không có AA nên KHÔNG hứa hão "uỷ quyền ký lâu dài".
  Mandate chứng minh kiểm-soát-địa-chỉ, không hơn.
- **Tối ưu:** tái dùng tối đa (consent store, token-exchange, JWKS, migration/claim
  builder đã spec); chỉ thêm **1 primitive Core** (`taad_cose_sign1_verify`) + vài
  endpoint Long mỏng. LAMP route thẳng `phoenix_addr` (không hop thừa). Lightweight DID
  tất định từ address (idempotent, không mint on-chain tới khi cần).
- **User + bền vững:** không đụng tài sản Lace (an tâm khi chưa tin PhoenixKey);
  ca mất-seed vẫn cứu được (migrate khi extension sống) + cảnh báo trung thực; phần
  thưởng nằm trong quỹ đạo Phoenix tạo lực kéo tốt nghiệp tự nhiên, không cưỡng ép.

---

## §10. Phụ thuộc ngoài (blocker thật)

- **LAMP team (điều khoản program):** points/weight per program; chế độ A (mandate-đủ)
  hay B (re-prove mỗi epoch); luật eligibility/anti-sybil cho ví-ngoài lightweight
  (§2.4). Leaf snapshot PHẢI dùng `phoenix_addr` làm `address`. → message §2.4.
- **Long (`Delegator-Claim-Offchain-Feat.md`, đang viết):** consent store, stake-state,
  merkle-proof, claim/submit/monitor — Connect tiêu thụ, không build.
- **Core (Delegator-Core...Feat.md):** migration + claim builder + connector +
  `assemble_witness` — reuse; **thêm** `taad_cose_sign1_verify`/`taad_addr_matches_pubkey`
  (§7.1) do spec này giao.

---

## §11. Chốt

PhoenixKey Connect = **on-ramp uỷ quyền BẰNG CHỮ KÝ**: ký mandate off-chain (Lace/
Yoroi), kiếm LAMP bằng chữ ký (không đụng tài sản), tốt nghiệp bằng migrate. Delta
thật so với 2 spec đang viết: **(1)** schema `AuthorizationMandate` + bind
`wallet_address↔phoenix_did` + domain-binding; **(2)** lightweight DID neo-địa-chỉ;
**(3)** `taad_cose_sign1_verify` (Core, CHƯA có); **(4)** affiso SDK widget +
token-exchange scoping. Mọi cơ chế tx/consent/claim **neo** spec đã có, KHÔNG chép.
Trung thực về giới hạn: **KHÔNG uỷ quyền chi on-chain** — Cardano không có
account-abstraction; mandate chỉ là thẩm quyền off-chain, thu-hồi-được, hữu hạn.
