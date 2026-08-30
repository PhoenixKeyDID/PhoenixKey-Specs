# PhoenixKey — Knowme · Đặc-tả TOÁN hình-thức cho AUDITOR

> **Đối tượng:** auditor VC / KYC-KYB / ZK selective-disclosure. Đây là đặc-tả TOÁN (ký-hiệu, định-nghĩa, bất-biến, mệnh-đề-ép, định-lý riêng-tư), KHÔNG phải doc thiết-kế. Doc thiết-kế ở `PhoenixKey-Knowme-Tech.md`; doc người-dùng ở `PhoenixKey-Knowme-Vi-Feat.md`.
>
> **Nguồn chân-lý = CODE (Mức 1+2) + SPEC (lớp tài-liệu, Mức 3):** bất-biến của lớp lõi neo trực-tiếp về `file:hàm` trong `PhoenixKey-Frontend/src/lib/sdvc/`. Bất-biến của lớp tài-liệu + Mức 3 neo về thiết-kế nội-bộ (không công khai), đánh dấu **[SPEC]**. Khi văn ≠ code → code thắng.
>
> → Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)
>
> **Phạm-vi đặc-tả này CHỨNG:** (a) tính đúng của commitment + membership tiết-lộ chọn-lọc; (b) ràng-buộc holder ↔ subject chống mạo-danh; (c) chống replay qua `aud`/`nonce`; (d) ràng-buộc nội-dung tài-liệu qua `docHash` **[SPEC]**; (e) tính riêng-tư Mức 3 (đúng-một-bit, unlinkable) **[SPEC]**; (f) **khoá** duy-nhất-người v1 — dấu giấy-tờ `fp` và các bất-biến đi kèm (Đ-7, I-KNOW-12..16). **NGOÀI phạm-vi:** **cơ-chế** sinh-trắc Enclave (§8) — doc này chỉ nêu **giới-hạn** của nó (I-KNOW-16: gác lượt đăng-ký, không gác người) chứ không chứng nó; catalog VC + issuer (VeData — §8), mạch ZK chi-tiết (circuit constraints), backend PhoenixKey.
>
> **Tái dùng bất-biến từ nguồn:** aliasing các bất-biến `INV-K1..K6` (nguồn `PhoenixKey-Knowme-Feat-Math §11`) và `I-KYC-*` (nguồn `PhoenixKey-KYC-KYB-ZK-Feat-Math §B.8`) được GHI RÕ nguồn ở cột "Neo/Nguồn". Mã canonical của module này là **`I-KNOW-n`**.

---

## 1. Ký-hiệu

| Ký-hiệu | Kiểu | Nghĩa | Neo code / nguồn |
|---|---|---|---|
| `H` | hàm băm | `BLAKE2b-256` | `crypto.ts:blake2b256` |
| `canon(x)` | ByteArray | JSON canonical: sort key mọi tầng, không whitespace thừa, giữ thứ-tự mảng | `canonical.ts:canonicalJson/canonicalBytes` |
| `salt` | ByteArray(16) | muối ngẫu-nhiên mỗi trường (hiding) | `commit.ts:makeDisclosure` |
| `path` | String | dotted path của trường, vd `"person.dob"` | `types.ts` |
| `value` | `String\|Number\|Boolean` | giá-trị vô-hướng của trường (`ClaimValue`) | `types.ts:ClaimValue` |
| `d` | Disclosure | bộ `{salt, path, value}` | `types.ts:Disclosure` |
| `digest(d)` | Digest = b64url(32B) | commitment của một trường | `commit.ts:digestOf` |
| `sd[]` | Set<Digest> | tập commitment công-bố trong credential | `credential.ts` |
| `sdRoot` | b64url(32B) | Merkle root trên `sd[]` | `credential.ts:recomputeSdRoot` |
| `sub` | DID | subject DID (chủ-thể của claim) | `credential.ts:91` |
| `iss` | DID | issuer DID (bên ký). Tự-khai ⟺ `iss = sub` | `credential.ts:92` |
| `aud` | DID | audience DID (bên nhận trình) | `disclose.ts:60` |
| `nonce` | ByteArray | thách-thức verifier phát, chống replay | `disclose.ts:61` |
| `holder` | DID | bên trình (ký presentation) | `disclose.ts:59` |
| `Resolve(did)` | DIDDoc | phân-giải khoá công-khai từ DID Document | `didResolver.ts` |
| `Envelope` | ciphertext | ECIES X25519→HKDF→AEAD của bytes tài-liệu | `crypto.ts` **[SPEC dùng cho tài-liệu]** |
| `cid` | CID | `BLAKE3(ciphertext)` — con-trỏ LampNet | `lampnet.ts` |
| `docHash` | b64url(32B) | `H(plaintext bytes)` — ràng-buộc nội-dung tài-liệu | **[SPEC]** Knowme-Feat-Math §3.3 |
| `k` | ByteArray(32) | khoá của `H_k`. Gánh **riêng-tư** (chặn dò ngược số giấy-tờ), KHÔNG gánh duy-nhất-người. Đích: chia `t`-trong-`n` (Đ-7.1) | **[SPEC]** Đ-7.1 |
| `kv` | Nat | `keyVersion` — số hiệu đợt khoá đang dùng, lưu cạnh `fp` để xoay được | `fingerprint.ts:Registration` |
| `fp(t, F)` | b64url(32B) | **dấu giấy-tờ** của loại `t` trên tập trường định-danh `F`. **Không phải vân tay sinh-trắc** — xem chú-thích Đ-7 | `FingerprintRegistryService.computeFingerprintHex` (backend, có pepper, `:209-219`); `fingerprint.ts:documentFingerprint` là bản tham-chiếu/minh-hoạ phía web, KHÔNG khoá — KHÔNG phải hàm chạy thật của registry |

**Đơn-vị wire (bất-biến-dây, load-bearing):** một trường được băm là **mảng 3 phần-tử** `[salt, path, value]` với `value` **vô-hướng**. Đây là điểm neo cross-language: verifier độc-lập chỉ cần lặp `H(canon([salt, path, value]))`. KHÔNG được đổi arity/hình-dạng mảng. Neo: `commit.ts:digestOf`.

---

## 2. Định-nghĩa hình-thức các hàm/tập

**Đ-1 · Commitment một trường** (`commit.ts:digestOf`):
```
digest(salt, path, value) = b64url( H( canon([salt, path, value]) ) )
```

**Đ-2 · Merkle root credential** (`credential.ts:recomputeSdRoot`):
```
sdRoot = merkleRoot( sort(sd[]) )      -- sd[] = { digest(dᵢ) : i ∈ trường khai }
```

**Đ-3 · Membership tiết-lộ (Mức 2)** — verifier nhận `d = {salt, path, value}`, chấp-nhận ⟺
```
digest(d.salt, d.path, d.value) ∈ sd[]
```
Neo `disclose.ts:verifyPresentation` (kiểm từng disclosed digest ∈ sd credential).

**Đ-4 · Neo anchor credential** (`credential.ts:anchorDigest`):
```
anchor(cred) = b64url( H( canon([ cred.sub, cred.sdRoot, cred.iat ]) ) )
```
Giá-trị công-bố on-chain cạnh DID Document; KHÔNG chứa value nào.

**Đ-5 · Gập tài-liệu về value vô-hướng (RULE-K-CANON)** **[SPEC]** — nguồn Knowme-Feat-Math §3.3:
```
value_doc = canon({ cid, docHash, meta })          -- một String hợp-lệ như ClaimValue
docDigest(d) = digest(d.salt, d.path, value_doc)    -- ĐÚNG đường digestOf gốc
```
Tài-liệu đi qua **cùng** `digestOf`, cùng arity mảng, cùng không-gian `Digest` với trường vô-hướng ⟹ trộn chung `sd[]`, membership-check chung. `docHash` (32B high-entropy) trong preimage đủ chống brute-force; `salt` là lớp đệm cho `meta` ít-entropy.

**Đ-6 · Predicate Mức 3** **[SPEC]** — nguồn KYC-KYB-ZK §B.1:
```
Predicate = { id, path, op ∈ {GE,LE,EQ,IN,RANGE}, arg, ctx? }
```

**Đ-7 · Dấu giấy-tờ tuỳ-thân (khoá duy-nhất-người v1)** (`FingerprintRegistryService.computeFingerprintHex`, backend, `:209-219`):
```
fp(t, F) = b64url( H_k( canon([ t, normalize(F) ]) ) )           -- H_k = BLAKE2b-256 keyed
```
- `t` = loại giấy-tờ (vd `"vn-cccd"`); `F` = tập trường **định-danh** đã chuẩn-hoá, chỉ gồm trường bất-biến theo thời-gian (số giấy-tờ, ngày-sinh, giới), **không** gồm ngày cấp / nơi cấp / ảnh.
- **Tất-định, KHÔNG salt-mỗi-lần-đăng-ký** — đây là điều kiện cần: hai bản khai của cùng một giấy-tờ phải cho cùng `fp`, nếu không thì không phát-hiện được trùng.
- `kv` lưu **rời** cạnh `fp` ⟹ xoay khoá được: cấp `kv+1`, tính lại dần, không phải khai lại giấy-tờ.

> ⚠️ **`fp` KHÔNG phải vân tay sinh-trắc.** Tên hàm trong mã (`fingerprint.ts`) dùng chữ
> "fingerprint" theo nghĩa **dấu băm của một chuỗi byte**, không phải theo nghĩa đường vân ngón tay.
> Mẫu sinh-trắc thật **không bao giờ rời Secure Enclave**, không đi qua mạng, và **không nằm trong
> bảng nào** — kể cả bảng này. Đọc nhầm hai thứ đó thành một sẽ dẫn tới đánh giá sai toàn bộ mục
> dưới đây. Trong đặc-tả này, thứ được lưu và đối-chiếu luôn là **dấu giấy-tờ** `fp`.

> **Registry THẬT chạy ở backend, không phải ở TS.** `fingerprint.ts:UniquenessRegistry`
> (`PhoenixKey-Frontend`) giữ state trong một `Map` bộ nhớ và tự khai trong chính docstring của nó
> là **"reference semantics"** — đây là bản tham-chiếu/minh-hoạ cho việc thiết-kế API, **KHÔNG phải
> registry chạy thật/production**. Cơ chế duy-nhất-người v1 của Đ-7 chạy thật ở
> `PhoenixKey-Database` (backend Java):
> - `FingerprintRegistryService.computeFingerprintHex` (`:209-219`) — đúng công-thức
>   `H(pepper ‖ canonicalize(t, F))`; `canonicalize` dạng netstring (mỗi đoạn tiền-tố bằng độ-dài,
>   đơn-ánh — tránh ambiguity khi nối chuỗi); pepper 32 byte **fail-closed khi khởi-động** ở
>   prod/staging (thiếu pepper ⟹ service không lên được, không âm-thầm chạy không-khoá).
> - Bảng `document_fingerprints` — migration `V35__document_fingerprints.sql` +
>   `V37__document_fingerprints_intent.sql`.
> - Endpoint `KnowmeFingerprintController`: `POST /identity/fingerprint/register`,
>   `GET /identity/fingerprint/status`.
>
> **⚠️ Cảnh-báo có thời-hạn — `documentFingerprint` phía web CHƯA khoá.** Cạnh
> `UniquenessRegistry` (tham-chiếu), `fingerprint.ts` còn có hàm `documentFingerprint`
> (`fingerprint.ts:77-80`) — hàm này **CÓ chạy thật** ở phía Frontend (copy-detection), và nó tính
> `blake2b256(canonicalBytes(["phoenix-docfp/1", …]))` **trần, không pepper/không salt**. Số CCCD
> Việt-Nam 12 số có cấu-trúc công-khai (3 số đầu = mã tỉnh · 1 số = giới-tính+thế-kỷ · 2 số = năm
> sinh · 6 số cuối = ngẫu-nhiên) ⟹ kẻ tấn-công biết tỉnh+năm-sinh+giới-tính của một người thì miền
> còn lại chỉ `10⁶` tổ-hợp ⟹ băm trần bị vét cạn gần như tức-thì — cùng lỗ đúng như Đ-7.1(B) mô-tả
> cho trường-hợp thiếu pepper, nhưng ở đây là hiện-trạng thật của mã, không phải giả-định. Ngày
> 2026-08-19 hàm này **chưa có nơi gọi sống nào** (route/UI) — chỉ export ở `index.ts:63` và trong
> file test — nên rủi-ro hiện chưa hiện-thực-hoá. Nhưng nó phải được khoá **TRƯỚC KHI** lớp tài-liệu
> tiếp theo của Knowme lên public, không phải sau.

**Đ-7.1 · Duy-nhất-người dựa vào phép SO-SÁNH, không dựa vào bí-mật** **[SPEC]**

Hai việc khác nhau, bản trước gộp làm một rồi cùng đặt lên một `pepper` phía máy-chủ:

| | Việc | Ai gánh | Lộ `k` hoặc lộ cả bảng `fp` thì sao |
|---|---|---|---|
| **(A)** | **Chặn cấp DID thứ hai cho cùng một giấy-tờ** | chính phép so-sánh ở `UniquenessRegistry` | **không ảnh-hưởng gì** |
| **(B)** | **Chặn dò ngược số giấy-tờ từ bảng `fp`** | khoá `k` của `H_k` | mất riêng-tư (B); (A) còn nguyên |

**(A) không cần bí-mật, vì so-sánh không cần bí-mật.** Registry chỉ hỏi một câu: `fp` này đã có
chủ chưa? Câu đó trả lời được kể cả khi bảng công-khai hoàn-toàn. Kẻ biết `k` và cầm cả bảng vẫn
không đăng-ký thêm được DID nào cho một giấy-tờ đã có chủ — không phải vì hắn thiếu bí-mật, mà vì
**registry từ chối, và lời từ chối đó không phụ-thuộc vào việc gì bị lộ**. Đây là lý do bất-biến
duy-nhất-người đứng vững mà không cần ai giữ gì cả.

Sinh-trắc Enclave đứng ở chỗ khác trong luồng: nó chứng **có người thật đang thao-tác trên máy này**
và buộc mỗi lượt đăng-ký phải qua một con người — nên nó chặn đăng-ký hàng-loạt bằng máy. Nó
**không** cho duy-nhất-người ở mức người: mẫu sinh-trắc nằm trong Enclave từng máy và không đối-chiếu
chéo giữa các máy được (T-1). Đừng ghi công cho nó việc mà nó không làm.

**(B) là chuyện riêng-tư, và thiệt-hại của nó có trần.** Số CCCD Việt-Nam 12 chữ-số **có cấu-trúc**
(3 tỉnh · 1 giới+thế-kỷ · 2 năm-sinh · 6 ngẫu-nhiên), nên biết tỉnh + năm-sinh + giới thì miền còn
`10⁶` — băm trần là vét cạn được tức thì, và làm chậm bằng hàm tốn-bộ-nhớ không cứu được vì miền
quá nhỏ. Nên vẫn giữ `k`.

Nhưng nói cho đúng cỡ: nếu `k` **và** bảng `fp` cùng lộ, thứ kẻ tấn-công thu được là **liên-kết
DID ↔ số giấy-tờ** của những người trong bảng. Đó là mất riêng-tư — **không** phải mất quyền chi,
không phải mất DID, không phải cấp thêm được DID. Tài-sản nằm sau cổng chi của Rebirthme, không nằm
sau `k`.

**Hướng đi cho `k`, và mức chấp-nhận được ở từng giai-đoạn:**
- **Đích:** `k` chia `t`-trong-`n` bằng đúng cơ-chế Tier-0 đã chốt cho bí-mật ≤32 byte (Shamir trên
  `Z_q` + cam-kết Feldman, an-toàn vô-điều-kiện dưới ngưỡng — `PhoenixKey-SeedDistribution-Tech-Math.md`),
  tính `fp` bằng **OPRF ngưỡng**: máy khách làm mù `canon([t, normalize(F)])`, `t` nút đánh giá từng
  phần, máy khách nội-suy Lagrange rồi gỡ mù ⟹ không nút nào thấy số giấy-tờ, không nút nào giữ `k`.
- **Chấp-nhận được trước khi có OPRF ngưỡng:** `k` một-mảnh do backend giữ, **với điều-kiện** ghi
  thẳng vào tài-liệu vận-hành rằng lộ nó = lộ liên-kết DID ↔ số giấy-tờ, và `kv` cho phép xoay khi
  cần. Đây **không** phải blocker của duy-nhất-người — vì (A) không dựa vào nó.
- Xoay khoá = chia lại mảnh (proactive resharing) → `kv+1`. Không ai phải khai lại giấy-tờ.

**Duy-nhất-người là ĐỒNG NHẤT, độ mạnh custody thì PHÂN TẦNG — đừng trộn hai trục.** Một người
một DID áp như nhau cho mọi người: người giữ vài nghìn đồng và người giữ kho tài-sản đều đi qua
đúng một phép so-sánh `fp`, vì bất-biến này nói về **danh-tính**, không nói về **giá-trị**. Số lớp
xác-thực thì ngược lại — chủ DID giá-trị cao bật thêm anti-drain, yếu-tố-2 thiết-bị, ngưỡng
guardian cao hơn (Rebirthme I-CURVE-4/5). Nghĩa là: **không được lấy "người quan trọng sẽ tự bật
nhiều lớp" làm lý-do nới lỏng (A)** — (A) bảo-vệ tính đúng của sổ danh-tính cho mọi người, và nó
không có nút bật/tắt theo người.

**Ảnh + video giấy-tờ.** Người dùng nộp ảnh và video giấy-tờ; bộ trích-xuất (Spectra) đọc ra `F` rồi ghi vào Knowme. Ảnh/video là **PII nguyên bản**, nên chúng đi đúng đường tài-liệu đã có: `Envelope` ciphertext + CID (I-KNOW-9), **không** lên chuỗi (I-KYC-NO-PII-CHAIN). Thứ tham-gia `fp` là `F` đã trích, không phải bytes ảnh — hai lần chụp cùng một thẻ cho hai `docHash` khác nhau nhưng **cùng** `fp`, đó là điều làm cơ-chế duy-nhất chạy được.

---

## 3. Bất-biến LÕI (tiết-lộ chọn-lọc)

**Bất-biến neo (COMMIT ↔ REVEAL):** với mọi trường được tiết-lộ hợp-lệ,
```
(COMMIT-REVEAL)   accept(d)  ⟺  digest(d.salt, d.path, d.value) ∈ sd[]  ∧  sig_holder hợp-lệ
```
Ép ở `disclose.ts:verifyPresentation`: (1) `aud = expectedAud`; (2) `nonce = expectedNonce`; (3) chữ ký holder phủ `[aud, nonce, disclosed set]`; (4) mỗi disclosed digest ∈ `sd[]`.

**Ẩn (HIDING):** với trường KHÔNG tiết-lộ, verifier chỉ thấy `digest` (ảnh của `H` có salt) ⟹ không suy ra `value`. Salt 16B ngẫu-nhiên chặn dictionary attack trên value ít-entropy (vd DOB). Nguồn: `commit.ts` docstring.

---

## 4. Bảng bất-biến `I-KNOW-1 .. I-KNOW-16`

| ID | Bất-biến (hình-thức) | Cơ-chế-ép | Neo / Nguồn |
|---|---|---|---|
| **I-KNOW-1** | **Wire-arity:** mọi commitment = `H(canon([salt,path,value]))`, `value` vô-hướng. Tài-liệu gập về `value_doc` (Đ-5) trước khi băm — KHÔNG đổi arity. | 1 đường `digestOf` duy nhất | `commit.ts:digestOf`; §3.3 (RULE-K-CANON) |
| **I-KNOW-2** | **Membership:** trình một trường được chấp-nhận ⟺ `digest ∈ sd[]` (Đ-3). Không digest ngoài `sd[]` nào lọt. | `verifyPresentation` kiểm ∈ sd | `disclose.ts` |
| **I-KNOW-3** | **Hiding:** trường không-tiết-lộ ⟹ verifier chỉ có `digest` (salt ≥128-bit) ⟹ không suy ra `value`. | salt 16B ngẫu-nhiên/trường | `commit.ts:makeDisclosure` |
| **I-KNOW-4** | **Holder ↔ Subject binding:** khoá người trình phân-giải từ **DID Document** (không từ presentation); kẻ trình credential người khác → resolve khoá không khớp → fail. `verificationMethod` proof phải khớp `assertionKeyId(holder)`. | `verifyPresentation` bind vm↔holder DID + `Resolve` | `disclose.ts:143`, `didDoc.ts`, `didResolver.ts` (alias `disclose.ts:161`) |
| **I-KNOW-5** | **Anti-replay + audience-binding:** `p.aud = expectedAud ∧ p.nonce = expectedNonce`; chữ ký holder phủ cả `aud`+`nonce`+disclosed ⟹ không replay sang verifier/phiên khác. | `verifyPresentation:130-134` | `disclose.ts` |
| **I-KNOW-6** | **Issuer-scope (issued, iss ≠ sub):** issuer phải ∈ TrustList đã neo, Active, phủ đúng docType/scope. Tự-khai (iss = sub) bỏ gate này. | `verifyIssuedCredential` | `trust.ts:650`, `trustList.ts` (alias I-KYC-ISSUER-SCOPE, §B.8) |
| **I-KNOW-7** | **Revoke:** (a) holder thu-hồi một disclosure theo CID → verifier từ-chối; (b) issuer set bit trong StatusList (append-only) → mọi proof phái-sinh fail. | `RevocationRegistry`; `StatusList`/`isRetired` | `statusList.ts:isRetired`; `disclose.ts` (revoked) (alias I-KYC-REVOKE) |
| **I-KNOW-8** | **Ràng-buộc nội-dung tài-liệu:** `docHash = H(plaintext)` nằm trong commitment ⟹ đổi 1 byte plaintext → đổi docHash → đổi commitment → vỡ membership. `cid` chỉ là con-trỏ, KHÔNG thay `docHash`. | `docDigest` (Đ-5) | **[SPEC]** Knowme-Feat-Math §11 INV-K1 |
| **I-KNOW-9** | **Không plaintext ra ngoài:** chỉ `Envelope` (ciphertext ECIES) + CID rời máy; server chỉ giữ ciphertext. | `eciesSeal` trước khi `LampNetStore.put` | `crypto.ts:eciesSeal`, `lampnet.ts:cidOf/put` — WIRING vào luồng DocumentClaim/sd[] là **[SPEC]** INV-K2 |
| **I-KNOW-10** | **Least-disclosure tài-liệu:** người nhận chỉ có Envelope + opener của đúng tài-liệu đã chọn; phần còn lại là commitment ẩn trong `sd[]`. | re-seal per-recipient + Presentation | **[SPEC]** INV-K5 |
| **I-KNOW-11** | **Lịch-sử bất-biến (versioning):** mọi cập-nhật tài-liệu = một `append_version` Strata mới; bản cũ giữ nguyên, proof field-level của một tài-liệu vẫn đúng dưới root mới; anchor đơn-điệu theo `seq` ⟹ không neo-lại phiên-bản cũ để "khôi-phục" giấy-tờ đã bị thay (chống rollback). | kế-thừa bất-biến Strata (KHÔNG redefine ở đây): hash-link, seq đơn-điệu, append-only, field-privacy, chống-rollback | **[SPEC]** Knowme-Feat-Math §5, §11 INV-K4 (alias Strata `_CONTRACT.md` INV-E1, E2, E3, E5, E6, E7) |

| **I-KNOW-12** | **Một giấy-tờ ⇒ một PersonDID (duy-nhất-người v1):** với mọi `(t, F)`, tồn tại **nhiều nhất một** PersonDID đang giữ `fp(t,F)`. Đăng-ký thứ hai cùng `fp` bị từ-chối, KHÔNG ghi đè. | Backend THẬT: `FingerprintRegistryService` + bảng `document_fingerprints` (migration V35/V37) kiểm `fp` đã có chủ chưa trước khi ghi, qua `KnowmeFingerprintController: POST /identity/fingerprint/register`. `UniquenessRegistry.register` (`fingerprint.ts`) chỉ là bản tham-chiếu/minh-hoạ cùng logic, KHÔNG chạy thật | `FingerprintRegistryService.java`; `V35__document_fingerprints.sql`, `V37__document_fingerprints_intent.sql`; `KnowmeFingerprintController.java:/identity/fingerprint/register`; (minh-hoạ) `fingerprint.ts:UniquenessRegistry` |
| **I-KNOW-13** | **Từ-chối trùng KHÔNG lộ chủ hiện-hữu:** kết-quả trả về khi trùng chỉ nói **có** chủ khác, không nói **ai**. | Endpoint `GET /identity/fingerprint/status` (backend) — verify response không mang DID chủ khác. `isHeldByAnother()`/`conflict_other_owner` (`fingerprint.ts`, đã bỏ trường `existingOwner`) là minh-hoạ cùng nguyên-tắc, KHÔNG phải server thật | `KnowmeFingerprintController.java:/identity/fingerprint/status`; (minh-hoạ) `fingerprint.ts` |
| **I-KNOW-14** | **`fp` là hàm CÓ KHOÁ, không phải băm trần:** mọi `fp` sinh qua `H_k`; `kv` lưu rời để xoay được. Đây là bất-biến **riêng-tư** — nó chặn dò ngược số giấy-tờ từ bảng — **KHÔNG** phải bất-biến duy-nhất-người. Vi-phạm nó làm lộ liên-kết DID ↔ số giấy-tờ, **không** làm cấp thêm được DID. | tham-số pepper **bắt-buộc** ở `computeFingerprintHex`, fail-closed khi thiếu ở prod/staging (backend, `:107-158`) — không có đường gọi nào bỏ qua | `FingerprintRegistryService.computeFingerprintHex:209-219` + **[SPEC]** Đ-7.1(B). ⚠️ KHÔNG nhầm với `fingerprint.ts:documentFingerprint` (web) — hàm đó KHÔNG có khoá, xem cảnh-báo sau Đ-7 |
| **I-KNOW-15** | **Duy-nhất-người KHÔNG phụ-thuộc vào bất-kỳ bí-mật nào:** phép so-sánh `fp` ở registry cho cùng kết-quả dù bảng công-khai hay không. Lộ `k`, lộ toàn bộ bảng `fp`, hay mất cả hai, đều **không** cấp thêm được DID nào cho một giấy-tờ đã có chủ. | Điều-kiện từ-chối của backend registry (`FingerprintRegistryService`) không đọc pepper, không đọc gì bí-mật — chỉ so `fp` đã tồn-tại trong `document_fingerprints` chưa | **[SPEC]** Đ-7.1(A); `FingerprintRegistryService.java`; bảng `document_fingerprints`; (minh-hoạ) `fingerprint.ts:UniquenessRegistry` |
| **I-KNOW-16** | **Sinh-trắc gác LƯỢT ĐĂNG-KÝ, không gác NGƯỜI:** cổng Enclave buộc mỗi lượt ghi `fp` phải qua một người thật trên máy đó ⟹ chặn đăng-ký hàng-loạt bằng máy. Nó **không** chứng người đó chưa đăng-ký ở máy khác — mẫu sinh-trắc nằm trong Enclave từng máy, **không rời máy**, **không có bảng nào** đối-chiếu chéo. | cổng sinh-trắc chạy trước `register`; không có kho mẫu sinh-trắc phía máy-chủ để đối-chiếu | §8; T-1 |

### 4.1 Tranh-chấp một giấy-tờ — hai mức, hệ-quả tách rời

Hai người có thể cùng khai một giấy-tờ mà **cả hai đều tin mình đúng** (mạo-nhận, nhập nhầm, giấy-tờ bị dùng lại). Quy-trình tách **nghi-ngờ** khỏi **xác-định**, vì hệ-quả khác nhau:

| Mức | Kích-hoạt | Hệ-quả |
|---|---|---|
| `suspected` | có khai-báo tranh-chấp trên `fp` | **Tạm ngưng cấp LAMP trong Wakeme** cho các bên liên-quan. Giai-đoạn `Daily` vẫn tích, chỉ **không cấp mới**. |
| `confirmed` | đã xác-định có tranh-chấp thật | LAMP Wakeme đã tích ở `Daily` **tạm ngưng phân-bổ về ví người dùng** ở giai-đoạn `Epochy`. |

**Bất-biến của quy-trình — giữ lại, KHÔNG tịch-thu.** Tạm ngưng là **hoãn**, không phải mất: khi tranh-chấp đóng (`closeDispute`), phần đã tích được phân-bổ tiếp cho bên được xác-định là chủ. Lý do: một quy-trình có thể sai, và một cơ-chế sai mà không hoàn-nguyên được thì mọi lỗi của nó đều vĩnh-viễn.

**PersonDID không bị vô-hiệu khi tranh-chấp.** DID là định-danh, không phải phán-quyết. Chỉ dòng giá-trị bị hoãn; ví, khoá, chữ ký, phiên đăng-nhập chạy bình-thường.

Neo: `fingerprint.ts:openDispute / closeDispute / disputeOf / lampHoldLevel`.

**Ngoài phạm-vi v1, để lại roadmap:** cùng-người-hai-giấy-tờ (đổi số CCCD, giấy-tờ nước khác), giấy-tờ ngoài Việt-Nam, chứng-minh duy-nhất-người không cần khai số giấy-tờ (hướng cam-kết ZK + nullifier theo ngữ-cảnh).

**Bất-biến Mức 3 (ZK) — GHI RÕ NGUỒN** (alias từ `KYC-KYB-ZK §B.8`, không tái-định-nghĩa):
- `I-KYC-PRIVATE` — proof lộ **đúng một bit** (`result`) về `predicate.path`; không lộ value/trường khác.
- `I-KYC-UNLINKABLE` — hai proof từ cùng credential cho hai verifier không tương-quan (BBS ngẫu-nhiên-hoá mỗi lần); ngoại-lệ có-chủ-ý: `predicate.ctx ≠ ⊥` dùng chung nullifier để khử-trùng "một người một lần/ngữ-cảnh".
- `I-KYC-RAW-OFFCHAIN` — value thô + giấy-tờ gốc không lên chuỗi, không rời máy dạng thường; chỉ (a) ciphertext LampNet, (b) witness cục-bộ.
- `I-KYC-NO-PII-CHAIN` — mọi thứ on-chain (ZkAnchor, TombstoneRecord, StatusList bit, DID Document) là hash/khoá công-khai, KHÔNG PII (GDPR Rec.26).
- `I-KYC-OFFLINE-VERIFY` — `ZkVerify` thành-công không cần issuer online.

---

## 5. Mệnh-đề-ép từng thao-tác (đối-chiếu code — trích guard load-bearing)

### 5.1 Trình tiết-lộ chọn-lọc (Mức 2) — `verifyPresentation` (`disclose.ts:121`)
```
p.aud = expectedAud                          -- (I-KNOW-5) audience
∧ p.nonce = expectedNonce                     -- (I-KNOW-5) anti-replay
∧ proof.verificationMethod = assertionKeyId(p.holder)   -- (I-KNOW-4) bind vm↔holder DID
∧ verify_sig(holderPublicKey, proof, [aud,nonce,disclosed])   -- holder ký THIS presentation
∧ ∀ dᵢ ∈ disclosed :  digest(dᵢ) ∈ credential.sd[]           -- (I-KNOW-2) membership
∧ ¬revoked(credential)                                       -- (I-KNOW-7)
```
**Chú-ý auditor:** `holderPublicKey` do verifier phân-giải từ DID Document (§B.4 nguồn), KHÔNG lấy từ `p`. Một kẻ trình credential người khác bằng khoá mình → `Resolve(holder).key` không khớp chữ ký → fail. Đây là tuyến chống mạo-danh (I-KNOW-4).

### 5.2 Cấp credential (Mức 1 tự-khai / issued) — `buildCredential` (`credential.ts:71`)
```
issuer = opts.issuerDid ?? claimSet.did       -- self-declared mặc-định (iss = sub)
∧ cred.sub = claimSet.did
∧ cred.iss = issuer
∧ proof.verificationMethod = assertionKeyId(issuer)   -- bind proof↔issuer key
∧ sdRoot = merkleRoot(sd[])
```
Tự-khai: `iss = sub`, chữ ký chứng "holder đã khai". Issued: `iss ≠ sub`, `signer` là khoá cơ-quan; verifier phân-giải khoá issuer từ `iss` để quyết tin (I-KNOW-6).

### 5.3 Kiểm chữ ký credential — `verifyCredentialSignature` (`credential.ts:115`)
```
proof.verificationMethod = assertionKeyId(cred.iss)    -- defense-in-depth
∧ verify_sig(issuerPublicKey, cred, proof)
```

### 5.4 Thu-hồi — `isRetired` / StatusList (`statusList.ts:262`)
```
isRetired(status, doc)  ⟺  statusOf(doc, status.index) = 1     -- bit set (append-only)
```
Verifier Mức 2 gọi kèm `verifyPresentation(..., revoked)`; credential set bit → mọi presentation phái-sinh fail (I-KNOW-7).

### 5.5 Gập + commit tài-liệu (Mức tài-liệu) **[SPEC]** — `docDigestOf` (Knowme-Feat-Math §3.3)
```
value_doc = canon({ cid, docHash, meta })
docDigest = digest(salt, path, value_doc)      -- ĐÚNG digestOf gốc
```
Người nhận kiểm: `docDigest ∈ sd[]` (thuộc hồ-sơ đã ký) **∧** `H(decrypt(EnvelopeR)) = docHash` (bytes khớp commitment — I-KNOW-8).

### 5.6 Sinh/kiểm proof Mức 3 (ZK) **[SPEC]** — `ZkProve`/`ZkVerify` (KYC-KYB-ZK §B.2/B.3)
```
ZkVerify(π, aud, nonce, s):
    π.aud = aud ∧ π.nonce = nonce
  ∧ Resolve(π.holder).bbsKey ≠ ⊥
  ∧ ∃ trusted issuer I : H(I.bbsKey) = π.issuerAnchor ∧ Active(I) ∧ BBS.VerifyDerived(...)
  ∧ π.result = true
  ∧ ( π.predicate.ctx = ⊥  ∨  ¬Seen(π.nullifier, π.predicate.ctx) )
```
Không lời gọi mạng tới issuer (I-KYC-OFFLINE-VERIFY). `w` (DOB, số giấy-tờ) chỉ tồn-tại trong máy holder; `π` không chứa `w` (I-KYC-PRIVATE).

---

## 6. Không-gian trạng-thái + đồ-thị chuyển

```
                buildCredential                buildPresentation(paths)          verifyPresentation
  [ClaimSet] ───────────────────► (SDCredential: sd[], sdRoot, sig) ─────────────► (Presentation: disclosed⊆sd, aud, nonce, sig) ──► accept/reject
                    │                          │                                            │
       (iss=sub tự-khai | iss≠sub issued)      │  seal→Envelope, aud/nonce bind             │  membership + holder-bind + revoke
                                               │                                            ▼
                                       StatusList.setBit ─────────────────────────► reject (I-KNOW-7)
       [SPEC] tài-liệu: seal→cid→docDigest ∈ sd[]  →  re-seal cho R  →  Presentation(DocumentDisclosure)  →  R: docDigest∈sd ∧ H(P)=docHash
       [SPEC] Mức 3:  ZkCredential(bbsProof) ── ZkProve(predicate) ──► π(result,proof, ctx?) ── ZkVerify ──► true/false
```

Chuyển hợp-lệ: `Declare → Sign(iss) → Disclose(subset) → Verify(accept)`; `Revoke → Verify(reject)`.
Chuyển BỊ CẤM (verifier phải REJECT): digest ∉ sd[]; sai `aud`/`nonce`; khoá người trình ≠ `Resolve(holder)` (mạo-danh); credential đã revoke; issued mà issuer ∉ TrustList/không Active.

---

## 7. Định-lý riêng-tư (+ chứng-minh phác-thảo)

### Định-lý 1 (SELECTIVE-DISCLOSURE-SOUND) — tiết-lộ đúng, ẩn phần còn lại.
> **Phát-biểu.** Verifier chấp-nhận một trường `d` ⟺ `d` thuộc credential đã-ký (membership) và presentation do đúng holder ký cho đúng phiên. Với trường không-tiết-lộ, verifier không thu được `value`.

**Chứng-minh (phác-thảo).**
1. *Soundness:* `accept(d) ⟹ digest(d) ∈ sd[]` (Đ-3) và `sd[]` được holder ký trong credential; giả-mạo `d′` với `digest(d′) ∈ sd[]` (một digest **đã tồn-tại sẵn** trong `sd[]`) đòi tìm **second-preimage** của `H` — tức tìm một `d′ ≠ d` sao cho `H(d′) = H(d)` với `H(d)` đã cho trước. Đây là thuộc-tính **chống-tiền-ảnh-thứ-hai (second-preimage resistance)** của BLAKE2b, KHÔNG phải **chống va-chạm (collision-resistance)** — hai thuộc-tính mật-mã khác nhau: collision-resistance là bài-toán tìm HAI input bất-kỳ (cả hai do kẻ tấn-công tự chọn) cùng chung một hash, còn ở đây một digest đã cố-định sẵn (từ `sd[]`), kẻ tấn-công chỉ được chọn input còn lại — đúng định-nghĩa second-preimage. *(Sửa 2026-07-12: bản trước ghi nhầm "chống va-chạm".)* ∎
2. *Binding phiên:* chữ ký holder phủ `[aud, nonce]` (I-KNOW-5) ⟹ tái dùng sang `aud′/nonce′` khác → chữ ký fail.
3. *Hiding:* trường không-tiết-lộ chỉ xuất-hiện dưới `digest = H(canon([salt,path,value]))` với salt ≥128-bit ẩn ⟹ dictionary attack trên value ít-entropy bất-khả nếu không biết salt (I-KNOW-3). ∎

### Định-lý 2 (NO-IMPERSONATION) — không trình được credential của người khác.
> **Phát-biểu.** Một tác-nhân chỉ trình được credential mà `holder = sub` của nó và tác-nhân giữ khoá phân-giải từ `Resolve(holder)`.

**Chứng-minh.** `verifyPresentation` (1) đòi `proof.verificationMethod = assertionKeyId(p.holder)`; (2) `holderPublicKey` lấy từ DID Document của `p.holder`, không từ `p`. Kẻ trình credential người khác (`sub = victim`) bằng khoá mình phải đặt `holder = victim` để membership khớp subject, nhưng khi đó `Resolve(victim).key ≠ khoá attacker` → chữ ký sai → reject. ∎ (Nguồn `disclose.ts:143`; alias I-KNOW-4.)

### Định-lý 3 (ZK-ONE-BIT) **[SPEC]** — Mức 3 lộ đúng một bit.
> **Phát-biểu.** `π` do `ZkProve` sinh tiết-lộ đúng `result ∈ {true,false}` về `predicate.path`; không lộ value cũng không lộ trường khác.

**Chứng-minh (phác-thảo, phụ-thuộc BBS).** `BBS.ProveDerived(disclosed = ∅, predicates = {(path,op,arg)})` ẩn mọi message, chỉ chứng ràng-buộc predicate trên message ẩn; π không chứa `w`. Unlinkability từ ngẫu-nhiên-hoá dẫn-xuất BBS (I-KYC-UNLINKABLE). Chứng-minh phụ-thuộc lib BBS production (T-5, §9). ∎

---

## 8. Giả-định tin-cậy (NGOÀI phạm-vi chứng-minh)

| # | Giả-định | Rủi-ro nếu vỡ | Ghi-chú |
|---|---|---|---|
| T-1 | **Duy-nhất-một-người (v1) neo vào giấy-tờ tuỳ-thân do nhà-nước cấp.** Mỗi PersonDID khai đúng một giấy-tờ tuỳ-thân; dấu giấy-tờ `fp` (§4) là khoá duy-nhất. Một giấy-tờ đã gắn PersonDID thì PersonDID thứ hai **không** đăng-ký được cùng giấy-tờ đó ⇒ "one-hash-per-tài-liệu" **chính là** cơ-chế một-người-một-DID ở v1. Sinh-trắc Secure Enclave gác **lượt đăng-ký trên một máy**, không gác **người** (I-KNOW-16) — nên nó là lớp chống-hàng-loạt, không phải nguồn của bất-biến này. | Vỡ khi: cùng người có hai giấy-tờ hợp-lệ khác nhau (đổi số, giấy-tờ nước khác), hoặc giấy-tờ bị mạo-nhận. Xử-lý: quy-trình tranh-chấp §4.1, không phải bịt bằng mật-mã. | Trong phạm-vi Knowme từ v1 — Đ-7, I-KNOW-12..16 (thay INV-K6) |
| T-2 | **Catalog VC + issuer + Trust Registry nội-dung** do **VeData** cung-cấp; PhoenixKey chỉ envelope + resolve + selective-disclosure. | Issuer giả trong TrustList → credential issued sai tin. TrustList neo Merkle root on-chain giảm-thiểu. | VeData; PhoenixKey dẫn-chiếu |
| T-3 | **Canonical JSON đồng-nhất** giữa TS (`canonical.ts`) và Java backend (`ORDER_MAP_ENTRIES_BY_KEYS`). | Lệch sort → digest khác → membership vỡ cross-language. | Căn chỉnh qua docstring `canonical.ts` |
| T-4 | **BLAKE2b/BLAKE3/Ed25519/ECIES** an-toàn mật-mã tiêu-chuẩn. | Va-chạm/giả chữ ký phá soundness. | Giả-định tiêu-chuẩn |
| T-5 | **[SPEC]** Lib BBS (BLS12-381) production (MATTR/Digital Bazaar), khoá BBS issuer quản vòng-đời (xoay → StatusList authority). | Chưa có lib BBS → I-KYC-PRIVATE/UNLINKABLE chưa chứng được bằng code (§9). | Phụ-thuộc thư-viện ngoài |
| T-6 | **[SPEC]** Với ảnh **mắt-người-xem-được**, verify buộc lộ plaintext cho người nhận → "thu-hồi" không lấy lại bản đã xem (verify-time plaintext resale, Knowme-Feat-Math §9.0). | Bán-lại ảnh vô-thời-hạn; **không vá triệt-để được**. | Giới-hạn cố-hữu; giảm-thiểu = watermark truy-vết + ưu-tiên proof-thay-ảnh |

**Kết-luận phạm-vi:** trong mô-hình {T-1..T-6}, Định-lý 1+2 (Mức 1+2) GIỮ. Định-lý 3 (Mức 3) và I-KNOW-8..10 (lớp tài-liệu) là **[SPEC]**.

---

## 9. [CẦN CHỐT] còn treo

| # | Mục | Ảnh-hưởng | Chủ |
|---|---|---|---|
| K-1 | **Padding kích-thước Envelope** (chống suy-loại-qua-kích-thước ciphertext). | Trung — privacy metadata. Đề-xuất: bật cho docType tùy-thân. | maintainer (Knowme-Feat-Math §10.1) |
| K-2 | **Per-context ref alias** (giống external-nullifier) để hai bên nhận không đối-chiếu "cùng hồ-sơ". | Trung — tăng riêng-tư, tăng phức-tạp SDK. | maintainer (§10.2) |
| K-3 | **Archetype StampRecord cho "notarization Strata head"** (neo qua Stamp). Knowme KHÔNG tự thêm archetype (SSoT của Stamp). | Chặn đường anchor qua Stamp. | đối-chiếu 21 archetype Stamp (§12) |
| K-4 | **Đường issued cho DocumentClaim** (cơ-quan ký ảnh) — phase này hay sau. | Lộ-trình tích-hợp cơ-quan. | maintainer (§10.6) |
| K-5 | **Nơi chạy Spectra/Glint cho ảnh nhạy-cảm** (on-device vs Splash-in-TEE) — phụ-thuộc hợp-đồng VeData. | Rủi-ro gửi plaintext ra ngoài. | maintainer + VeData (§10.5, §12) |

→ Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)

---

## 10. Checklist cho auditor

1. **Wire-arity (I-KNOW-1):** grep `digestOf` — chỉ MỘT đường `H(canon([salt,path,value]))`? Tài-liệu gập qua `value_doc = canon({cid,docHash,meta})` rồi mới `digestOf`? (§3.3)
2. **Membership (I-KNOW-2):** `verifyPresentation` kiểm mọi disclosed digest ∈ `sd[]`?
3. **Holder-bind (I-KNOW-4):** `holderPublicKey` lấy từ `Resolve(holder)` **chứ không** từ presentation? `verificationMethod = assertionKeyId(holder)`? (`disclose.ts:143`)
4. **Anti-replay (I-KNOW-5):** `aud`/`nonce` cùng trong preimage chữ ký holder? Test tráo `aud`/`nonce` → reject.
5. **Issuer-scope (I-KNOW-6):** issued → `verifyIssuedCredential` gate TrustList + Active + scope? Tự-khai bỏ gate?
6. **Revoke (I-KNOW-7):** `isRetired`/`StatusList` append-only; credential set bit → mọi presentation fail?
7. **Canonical cross-language (T-3):** sort-key TS ≡ Java? (chống membership-vỡ)
8. **[SPEC] docHash-bind (I-KNOW-8):** người nhận băm lại bytes giải-mã, so `docHash`? `cid` KHÔNG thay `docHash`?
9. **[SPEC] Mức 3:** π chứa `w`? (phải KHÔNG); `ZkVerify` gọi mạng issuer? (phải KHÔNG).
10. **[SPEC] Versioning (I-KNOW-11):** mỗi cập-nhật tài-liệu tạo `append_version` Strata mới (không sửa in-place)? Anchor `seq` chỉ tăng, neo-lại phiên-bản cũ bị từ-chối (chống rollback)?

---

## Phụ-lục A — Bảng đối-chiếu bất-biến ↔ dòng code / nguồn

| Bất-biến | Neo |
|---|---|
| I-KNOW-1 wire-arity | `commit.ts:digestOf`; Feat-Math §3.3 |
| I-KNOW-2 membership | `disclose.ts:verifyPresentation` |
| I-KNOW-3 hiding | `commit.ts:makeDisclosure` (salt 16B) |
| I-KNOW-4 holder-bind | `disclose.ts:143`; `didDoc.ts`; `didResolver.ts` |
| I-KNOW-5 anti-replay | `disclose.ts:130-134` |
| I-KNOW-6 issuer-scope | `trust.ts:650`; `trustList.ts` |
| I-KNOW-7 revoke | `statusList.ts:262`; `disclose.ts` |
| I-KNOW-8 docHash-bind | **[SPEC]** Feat-Math §11 INV-K1 |
| I-KNOW-9 no-plaintext | **[SPEC]** INV-K2 |
| I-KNOW-10 least-disclosure doc | **[SPEC]** INV-K5 |
| I-KNOW-11 lịch-sử bất-biến (versioning) | **[SPEC]** Feat-Math §5, §11 INV-K4 (alias Strata `_CONTRACT.md` INV-E1,E2,E3,E5,E6,E7) |
| I-KNOW-12 một-giấy-tờ-một-PersonDID | `FingerprintRegistryService.java`; `document_fingerprints` (V35/V37); `KnowmeFingerprintController.java:/identity/fingerprint/register` — backend THẬT. `fingerprint.ts:UniquenessRegistry` chỉ là tham-chiếu |
| I-KNOW-13 từ-chối trùng không lộ chủ | `KnowmeFingerprintController.java:/identity/fingerprint/status` — backend THẬT. `fingerprint.ts` chỉ là tham-chiếu |
| I-KNOW-14 fp có-khoá | `FingerprintRegistryService.computeFingerprintHex:209-219` (pepper fail-closed `:107-158`) — backend THẬT. KHÔNG phải `fingerprint.ts:documentFingerprint` (web, KHÔNG khoá) |
| I-KNOW-15 duy-nhất-người không phụ-thuộc bí-mật | **[SPEC]** Đ-7.1(A); `FingerprintRegistryService.java` + bảng `document_fingerprints` |
| I-KNOW-16 sinh-trắc gác lượt không gác người | §8; T-1 |
| I-KYC-PRIVATE/UNLINKABLE/... | **[SPEC]** KYC-KYB-ZK §B.8 |

---

## Nguồn

- Code (nguồn chân-lý Mức 1+2): `PhoenixKey-Frontend/src/lib/sdvc/{commit,canonical,credential,disclose,didDoc,didResolver,statusList,trust,trustList,crypto,lampnet,dossier,fingerprint,consent,schema,schemaRegistry,anchor,did}.ts`. Riêng phần **registry duy-nhất-người v1 (Đ-7, I-KNOW-12..15)**, nguồn chân-lý là backend `PhoenixKey-Database` (`FingerprintRegistryService.java`, `document_fingerprints`, `KnowmeFingerprintController.java`) — `fingerprint.ts:UniquenessRegistry`/`documentFingerprint` chỉ là bản tham-chiếu/minh-hoạ phía web.
- Spec (lớp tài-liệu + Mức 3): thiết-kế nội-bộ (không công khai) — Knowme-Feat-Math (§3, §6, §9, §11), KYC-KYB-ZK-Feat-Math (§B).
- `PhoenixKey-Math.md §34` (Privacy/GDPR tombstone — dẫn-chiếu, KHÔNG sửa).
- → Trạng-thái & tiến-độ: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#knowme)

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
