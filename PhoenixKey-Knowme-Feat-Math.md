# PhoenixKey — Knowme (Feat + Math)

**Trạng thái**: SPEC PROPOSAL (design doc, chưa code)
**Ngày**: 2026-07-01
**Phạm vi**: module `Knowme` trong SuperApp — hồ sơ giấy tờ/hình ảnh gắn với DID, cập nhật theo thời gian, giữ selective disclosure.
**Nền tảng tái dùng (KHÔNG phát minh lại)**:
- SD-VC lib `PhoenixKey-Frontend/src/lib/sdvc/*` — `ClaimSet`, `SDCredential`, `Disclosure`, `Presentation`, `Envelope`, `LampNetStore`, crypto (BLAKE2b-256 commitment, ECIES X25519+HKDF+XChaCha20Poly1305, BLAKE3 CID, Merkle root).
- Strata (`/Users/ductiger/Projects/Strata`) — hồ sơ tiến hóa: version hash-link + MMR + `state_root` field-Merkle + anchor on-chain.
- VeData tích hợp qua ĐÚNG intake của VeData: Spectra (tán media để verify), Glint (`AuthenticityProof`), **Stamp** (đóng dấu record + phát `anchor_request` sang Mosaic), Query (read gateway + D9 privacy + DP). Knowme KHÔNG gọi thẳng Mosaic; KHÔNG bịa data-plane. Xem §12 (ranh giới hợp đồng VeData) trước khi tin bất kỳ tích hợp nào.

> Ranh giới: Knowme là **module frontend/SDK trên SD-VC**, KHÔNG viết crypto mới, KHÔNG sửa backend PhoenixKey. Nó mở rộng `ClaimSet`/`SDCredential` thêm một loại claim `Document` và định nghĩa luồng seal→store→version→disclose cho tài liệu.
>
> **Ranh giới hợp đồng (đọc trước):** hợp đồng-của-record giữa PhoenixKey và VeData (`VeDataIO/Specs/PhoenixKey-VeData-Contract.md` v0.2.0) chỉ cấp **API resolve DID** (`did_active`, `key_authorized`, `revocation_check`, pubsub) — KHÔNG cấp data-plane cấp-ứng-dụng cho Strata/Mosaic/Query. Nghĩa là: mọi thứ cần đẩy **dữ liệu tài liệu** vào VeData PHẢI đi qua intake của chính VeData (Stamp), hoặc chờ VeData team mở rộng hợp đồng. §12 liệt kê rõ cái nào làm được hôm nay, cái nào "blocked-on-contract".

---

## §1. Bối cảnh

### 1.1 Vấn đề

SD-VC hiện tại (lib `sdvc/`) chỉ cầm được **giá trị vô hướng** (`ClaimValue = string | number | boolean`). Người dùng khai "họ tên", "số CCCD", "mã số thuế" dạng text — đủ cho KYC/KYB tối giản. Nhưng identity thật ngoài đời **luôn kèm giấy tờ**: ảnh mặt trước/sau CCCD, scan hộ chiếu, bằng lái, học bạ, bằng cấp; với OrgDID là giấy phép kinh doanh, giấy phép đặc thù, hồ sơ thuế, bảo hiểm; với Device/Product DID là chứng nhận nguồn gốc, tiêu chuẩn kỹ thuật, ngày sản xuất.

Schema SD-VC hôm nay KHÔNG chứa được ảnh/tệp đính kèm — đây chính là khoảng trống Knowme lấp.

### 1.2 Knowme làm gì

Knowme cho chủ DID:
1. **Tải lên** tài liệu/hình ảnh gắn với DID của mình, theo lớp DID.
2. **Cập nhật theo thời gian** — mỗi lần đổi = một phiên bản Strata mới, bản cũ giữ nguyên, proof từng tài liệu.
3. **Chọn lọc tiết lộ** — một người nhận chỉ thấy đúng tài liệu/trường chủ DID cho phép, phần còn lại vẫn ẩn.
4. **Không bao giờ để plaintext lên LampNet** — mọi tài liệu đóng gói ECIES trước khi rời máy.

Theo lớp DID (chỉ là danh mục schema, KHÔNG phải luật cứng):

| DID class | Ví dụ tài liệu |
|---|---|
| PersonDID (`0x00`) | CCCD/căn cước, hộ chiếu, bằng lái, học bạ, bằng cấp |
| OrgDID (`0x01`) | giấy phép kinh doanh, giấy phép đặc thù, hồ sơ nghĩa vụ thuế, bảo hiểm |
| Device/Product DID | nguồn gốc xuất xứ, tiêu chuẩn kỹ thuật, ngày sản xuất |

### 1.3 KHÔNG thuộc phạm vi (ranh giới cứng)

- 🔴 **KHÔNG** dedup / uniqueness / sybil-detection trên PersonDID. Knowme nói về **tài liệu**, không nói về "một người là duy nhất". Sinh trắc Secure Enclave đã đủ; mọi cơ chế đối chiếu-trùng-người nằm ngoài module này (và trái nguyên tắc dự án).
- **KHÔNG** phát minh crypto. Mọi hàm băm/ký/mã hóa lấy từ `sdvc/crypto.ts`.
- **KHÔNG** tự chấm "giấy tờ thật hay giả bằng con người" — self-declared là mặc định; Glint chỉ trả *nguồn gốc media* (thật/AI/lai), không phải *pháp lý đúng/sai* (xem §9, §10).

---

## §2. Kiến trúc

Knowme là một **lớp mỏng trên SD-VC**, cắm vào ba hạ tầng đã có:

```
             ┌──────────────────────────── SuperApp (Knowme UI) ────────────────────────────┐
             │  chọn tài liệu → xem phiên bản → chia sẻ chọn-lọc → thu hồi                    │
             └───────────────┬───────────────────────────────────────────────┬──────────────┘
                             │                                                 │
                     Knowme SDK (mới, thin)                             SD-VC lib (có sẵn)
        seal doc → Envelope → CID ; build DocumentClaim ;         commit/disclose/presentation
        version qua Strata ; disclose doc + opener                (BLAKE2b, ECIES, Merkle)
                             │
        ┌────────────────────┼────────────────────────────────────────────────┐
        ▼                    ▼                        ▼                          ▼
  LampNet (Mirage)      Strata chain           VeData qua Stamp             Cardano L1
  ECIES ciphertext,     version + state_root    Spectra/Glint (verify),     Stamp phát
  content CID           field-Merkle proof      Query (read gateway),       anchor_request
                                                 Stamp (đóng dấu→anchor)     → Mosaic anchor
                                                                              (32-byte root)
```

Chú ý luồng anchor: Knowme **không** gửi anchor request thẳng cho Mosaic. Mosaic intake CHỈ nhận `anchor_request` từ **Stamp** (Mosaic-Feat-Spec §0.5, §6 F1: "nhận `anchor_request` từ Stamp module", "`anchor_request` schema + `anchor_priority` 4-enum SSoT" là của Stamp). Đường đúng: Knowme phát một **StampRecord** → pipeline Stamp (Step 11 EMIT, Stamp-Feat §F2.12.8) tự phát `anchor_request` sang Mosaic. Xem §7.3.

Nguyên tắc first-principles:
1. **Tài liệu là claim, không phải kiểu dữ liệu mới.** Ta không nhét bytes vào `ClaimValue`; ta commit **BLAKE2b hash của tài liệu** như một claim, còn bytes đã mã hóa nằm off-chain trên LampNet theo CID. Selective disclosure tự động phủ tài liệu vì tài liệu chỉ là một leaf trong `sd[]`.
2. **On-chain chỉ giữ một root 32 byte.** Không đẩy metadata tài liệu lên chain (đắt + lộ). Chain giữ `state_root`/`mmr_root` của Strata (Mosaic anchor). eUTXO-thân-thiện: 1 datum ≤ 640 byte bất biến theo độ sâu lịch sử.
3. **Một seal = một CID.** Mỗi tài liệu là một `Envelope` độc lập → CID độc lập → có thể chia sẻ/thu hồi từng cái, không phải cả hồ sơ.
4. **Versioning tách khỏi crypto.** Strata lo lịch sử bất biến + proof; SD-VC lo commitment + disclosure. Hai tầng ghép qua `docHash`/`content_cid`, không trộn trách nhiệm.

---

## §3. Data schema (types)

Mở rộng `sdvc/types.ts`. Tất cả tên tiếng Anh, khớp phong cách file gốc.

### 3.1 Loại claim tài liệu

```ts
/** Đối chiếu: hôm nay ClaimValue = string | number | boolean (scalar only).
 *  Knowme thêm loại claim thứ hai — Document — dưới dạng một COMMITMENT tới
 *  tài liệu off-chain, KHÔNG phải bytes inline. Bytes đã mã hóa nằm trên LampNet. */

/** Metadata mô tả một tài liệu (vô hướng, an toàn để đưa vào commitment). */
export interface DocumentMeta {
  docType: string;        // "VN-CCCD-front" | "VN-Passport" | "OrgLicense" | "ProductOrigin" ...
  mime: string;           // "image/jpeg" | "application/pdf"
  bytesLen: number;       // độ dài plaintext (để hiển thị; KHÔNG lộ nội dung)
  capturedAt?: number;    // epoch seconds — thời điểm chụp/cấp, caller-supplied
  label?: string;         // nhãn hiển thị (i18n key hoặc literal)
  page?: number;          // trang/mặt (mặt trước=1, mặt sau=2)
}

/**
 * Một claim TÀI LIỆU. Đây là leaf được commit trong SD-VC `sd[]`, y hệt một
 * field vô hướng — nên selective disclosure tự động phủ tài liệu.
 *  - `docHash`  = BLAKE2b-256(plaintext bytes) — RÀNG BUỘC nội dung. base64url.
 *  - `cid`      = CID của Envelope (ciphertext) trên LampNet. KHÔNG phải hash plaintext.
 *  - `meta`     = mô tả vô hướng, đi vào commitment cùng docHash.
 * KHÔNG chứa bytes. KHÔNG chứa khóa. Chủ DID giữ opener (§6) để tiết lộ.
 */
export interface DocumentClaim {
  kind: "document";
  path: string;           // dotted path, vd "person.cccd.front", "org.license"
  docHash: string;        // base64url BLAKE2b-256(plaintext)
  cid: CID;               // LampNet content id của Envelope
  meta: DocumentMeta;
}

/** Mở rộng: một claim có thể là vô hướng (như cũ) HOẶC một DocumentClaim. */
export type KnowmeClaim = ClaimValue | DocumentClaim;
```

### 3.2 Mở rộng ClaimSet

```ts
/** Bản mở rộng của ClaimSet: claims giờ có thể chứa DocumentClaim.
 *  Vẫn keyed theo dotted path. `docs` liệt kê các path là tài liệu (tiện UI). */
export interface KnowmeClaimSet {
  did: string;
  kind: SubjectKind;              // person | org (+ Device/Product qua schema mới)
  schema: string;                 // "phoenix-knowme-person/1" | "phoenix-knowme-org/1" ...
  claims: Record<string, KnowmeClaim>;
  docPaths: string[];             // các path trong claims có kind="document"
}
```

### 3.3 Commitment của tài liệu (khớp CHÍNH XÁC wire format `commit.ts`)

**Đối chiếu wire format thật** (`sdvc/commit.ts`, đọc trực tiếp):

```ts
export function digestOf(d: Disclosure): Digest {
  return toB64url(blake2b256(canonicalBytes([d.salt, d.path, d.value])));
}
```

`digestOf` băm một **mảng 3 phần tử** `[salt, path, value]` với `value` là **vô hướng** (`ClaimValue = string | number | boolean`). Đây là bất biến-dây (wire invariant): một verifier độc lập chỉ cần lặp lại đúng `blake2b256(canonicalBytes([salt, path, value]))` là set-membership-check được. Knowme KHÔNG được đổi arity hay hình dạng mảng này — nếu đổi, verifier theo `commit.ts` sẽ tính ra digest khác và membership vỡ.

Vậy tài liệu — vốn là bộ `(docHash, cid, meta)` — phải **gập vào đúng một `value` vô hướng** trước khi vào mảng. Quy tắc gập tường minh (RULE-K-CANON):

```
value_doc = canonicalJson({ cid, docHash, meta })
```

trong đó `canonicalJson` là chính hàm `sdvc/canonical.ts` (sort key lexicographic mọi tầng, không whitespace thừa). Kết quả `value_doc` là **một chuỗi** (string) → hợp lệ như một `ClaimValue`. Commitment tài liệu do đó bằng:

```
commit_doc = toB64url( blake2b256( canonicalBytes([ salt, path, value_doc ]) ) )
           = digestOf({ salt, path, value: value_doc })
```

→ Tài liệu đi qua **đúng đường `digestOf` gốc**, không cần hàm hash riêng. Cùng không gian `Digest`, cùng arity mảng, cùng cách sort → trộn chung `sd[]`, sort chung, membership-check chung với scalar claim. Verifier theo `commit.ts` chỉ cần biết một điều bổ sung: với path là tài liệu, `value` là `canonicalJson({cid, docHash, meta})` chứ không phải giá trị người-đọc-được.

```ts
/** Opener riêng-tư của một tài liệu — analog của Disclosure cho scalar.
 *  KHÔNG băm trực tiếp interface này; luôn gập về `value_doc` (RULE-K-CANON)
 *  rồi gọi digestOf({salt, path, value: value_doc}) để ra ĐÚNG một commitment. */
export interface DocumentDisclosure {
  salt: string;                   // base64url 16 bytes (như Disclosure)
  path: string;                   // dotted path
  docHash: string;                // base64url BLAKE2b-256(plaintext)  ← ràng buộc nội dung
  cid: CID;                       // Envelope CID trên LampNet
  meta: DocumentMeta;
}

/** Gập DocumentDisclosure → ClaimValue vô hướng khớp wire format commit.ts. */
export function docValue(d: DocumentDisclosure): string {
  // canonicalJson: sort key mọi tầng, khớp sdvc/canonical.ts. meta là object —
  // canonicalJson sort field của meta để verifier tái tạo bit-hệt.
  return canonicalJson({ cid: d.cid, docHash: d.docHash, meta: d.meta });
}

/** Commitment tài liệu = digestOf({salt, path, value: docValue(d)}). base64url.
 *  Cùng arity mảng [salt,path,value] + cùng không gian Digest với scalar. */
export function docDigestOf(d: DocumentDisclosure): Digest {
  return digestOf({ salt: d.salt, path: d.path, value: docValue(d) });
}
```

**INVARIANT-K1 (ràng buộc nội dung).** `docHash` đi vào commitment ⇒ đổi 1 byte plaintext ⇒ đổi `docHash` ⇒ đổi commitment ⇒ hỏng membership. Người nhận tự băm bytes giải mã được và so với `docHash` đã tiết lộ. `cid` chỉ là con trỏ lưu trữ, KHÔNG được tin thay cho `docHash` (INV-E5 Strata: CID không lộ loại, và không ràng buộc nội dung một mình).

### 3.4 Bản ghi phiên bản Knowme (cầu nối Strata)

```ts
/** Một phiên bản hồ sơ Knowme = một StrataVersion + con trỏ SD-VC.
 *  Strata giữ lịch sử; SD-VC giữ commitment. Ghép qua state_root ↔ sd[]. */
export interface KnowmeVersionRef {
  refId: string;          // Strata ref_id "lnref1…" — ổn định xuyên phiên bản, gắn DID
  seq: number;            // Strata version seq (0,1,2,…)
  versionHash: string;    // hex Strata version_hash (không gồm sig)
  stateRoot: string;      // hex Strata state_root (field-Merkle trên các doc leaf)
  sdRoot: string;         // base64url SD-VC Merkle root (khớp SDCredential.sdRoot)
  createdAt: number;      // epoch seconds
}
```

---

## §4. Luồng upload + seal + store

Mục tiêu: bytes plaintext KHÔNG rời máy dưới dạng rõ. Chỉ commitment + ciphertext CID mới ra ngoài.

```
Chủ DID chọn tài liệu (ảnh/PDF)
   │
   ▼ (1) đọc bytes plaintext P
   │
   ▼ (2) docHash = blake2b256(P)                         [crypto.ts]
   │
   ▼ (3) seal: Envelope = eciesSeal(P, ownerEncPub, ownerDid, randomBytes(32), randomBytes(24))
   │      → tài liệu mã hóa CHO CHÍNH CHỦ DID (X25519 enc key của mình)     [crypto.ts]
   │
   ▼ (4) cid = LampNetStore.put(Envelope)  → "b"+base32(BLAKE3(envelope))   [lampnet.ts]
   │
   ▼ (5) DocumentDisclosure { salt=randomBytes(16), path, docHash, cid, meta }
   │      commit = docDigestOf(disclosure)                                  [§3.3 RULE-K-CANON]
   │             = digestOf({salt, path, value: canonicalJson({cid,docHash,meta})})  [commit.ts]
   │
   ▼ (6) thêm commit vào sd[] ; rebuild sdRoot = merkleRoot(sd)             [credential.ts]
   │
   ▼ (7) mở phiên bản Strata (§5): state_root gồm leaf của mọi doc
   │
   ▼ (8) (tùy chọn) verify Spectra/Glint (§7.1-7.2, on-device hoặc qua Stamp) ; anchor root QUA STAMP → Mosaic (§7.3)
```

**Điểm cốt lõi:**
- Bước (3) mã hóa **cho chính chủ DID** (recipient = owner enc pubkey). Không có "server key". Chỉ chủ DID mở lại được kho của mình. Khi chia sẻ cho người khác → **re-seal cho người nhận** (§6), không lộ khóa gốc.
- Bước (4): CID là hash của **ciphertext**, nên LampNet không index theo nội dung/loại (INV-E5, `sdvc/lampnet.ts`).
- Bước (5)-(6): tài liệu trở thành một leaf trong `sd[]` — không cần cơ chế disclosure riêng; tái dùng nguyên `credential.ts`.
- Không có plaintext nào được ghi log/gửi mạng. `bytesLen` trong meta chỉ là số, không phải nội dung.

**Chi phí (eUTXO/băng thông):** upload là O(1) ghi LampNet + O(n) rebuild sdRoot cục bộ (n = số tài liệu, thường < 20). On-chain: 0 cho tới khi anchor một root (§7.3, qua Stamp). Không đẻ UTXO mỗi tài liệu.

---

## §5. Luồng versioning (Strata)

Mỗi lần chủ DID thêm/thay/xóa một tài liệu = **một `append_version` Strata**. Bản cũ bất biến; proof từng tài liệu từ `state_root`.

Ánh xạ Knowme → Strata (dùng đúng API `Strata-API.md §2`):

| Hành động Knowme | Strata |
|---|---|
| Tạo hồ sơ lần đầu | `StrataChain::genesis(ref_id, v0, &policy)` — `ref_id = gen_ref_id(ownerDidHash, nonce)` |
| Thêm/đổi/xóa tài liệu | `chain.append_version(v, &policy)` — seq+1, prev_hash link, ts đơn điệu |
| Đọc hồ sơ mới nhất | `chain.head()` |
| Đọc "hồ sơ tại thời điểm t" | `chain.version_at(t)` |
| Chứng minh 1 tài liệu ở version seq | `state::prove_field(fields, key=path)` → `FieldProof` |
| Neo on-chain | Knowme phát StampRecord tham chiếu head → Stamp EMIT `anchor_request` → Mosaic (§7.3). KHÔNG gọi Mosaic trực tiếp. |

**State của một version** (loại #4 "Hồ sơ cấu trúc" trong Strata `_CONTRACT.md`):
- `fields[i] = (field_key = path, field_value_bytes = docHash-bytes)`.
- Theo CHỐT-4 Strata: `value_cid` phải là **content_cid thuần** (không class byte) để không lộ loại qua field-proof. Ta dùng **docHash (32 byte)** làm `field_value_bytes` — vừa ràng buộc nội dung vừa không lộ loại. (CID lưu trữ đi kèm ở lớp Knowme, không vào state leaf.)
- `state_root` = Merkle field-level trên các leaf đã sort theo `path` (khớp mã hóa state leaf trong `_CONTRACT.md`).

**Bất biến kế thừa Strata (không redefine):**
- INV-E1 hash-link, INV-E2 seq đơn điệu, INV-E3 append-only (proof cũ vẫn đúng dưới root mới).
- INV-E6 field-privacy: proof một tài liệu KHÔNG lộ tài liệu khác — chính là selective disclosure ở tầng versioning.
- INV-E7 chống rollback: anchor đơn điệu theo seq — không thể neo lại phiên bản cũ để "khôi phục" giấy tờ đã bị thay.

**Song song với SD-VC:** mỗi `KnowmeVersionRef` giữ CẢ `stateRoot` (Strata) VÀ `sdRoot` (SD-VC). Hai root cùng phủ một tập tài liệu nhưng phục vụ hai verifier khác nhau (Strata = lịch sử/anchor; SD-VC = disclosure cho người nhận). Bất biến ghép: tập `{path}` trong state = tập `{path}` của các DocumentClaim trong sd[] cùng seq.

**Chống-đẻ-version cao tần** (kế thừa Strata): nếu người dùng sửa nhiều lần liên tục, gộp lô theo epoch (sub-MMR) — nhưng với Knowme (giấy tờ đổi vài lần/năm) mặc định 1 sửa = 1 version là đủ.

---

## §6. Selective disclosure của tài liệu

Chủ DID chia sẻ **một số** tài liệu cho **một** người nhận, phần còn lại ẩn. Tái dùng nguyên mô hình `disclose.ts` (`buildPresentation`/`verifyPresentation`), chỉ khác payload disclosed chứa `DocumentDisclosure` + một **re-sealed Envelope cho người nhận**.

### 6.1 Luồng chia sẻ

```
Người nhận R gửi (audDid=R, nonce)
   │
Chủ DID chọn paths = ["person.cccd.front", "person.degree"]     ← chỉ 2 trong n tài liệu
   │
   ▼ (1) với mỗi path:  mở Envelope của mình → plaintext P       [eciesOpen, khóa chủ DID]
   ▼ (2) re-seal cho R:  EnvelopeR = eciesSeal(P, R.encPub, R.did, eph, nonce2)  [crypto.ts]
   ▼ (3) cidR = LampNetStore.put(EnvelopeR)   (bản mã hóa RIÊNG cho R)
   │
   ▼ (4) build Presentation:
   │        disclosed = [ DocumentDisclosure{salt,path,docHash,cid:cidR,meta} , … ]  ← chỉ paths đã chọn
   │        aud=R.did, nonce, purpose?, exp?    → ký Ed25519 (holder)   [disclose.ts]
   │
   ▼ (5) gửi Presentation cho R (qua Query gateway §7.4, hoặc kênh trực tiếp)
```

### 6.2 Người nhận verify

```
R nhận Presentation:
  (a) verify chữ ký holder + credential (như SD-VC hiện có)         [disclose.ts]
  (b) với mỗi DocumentDisclosure: commit' = docDigestOf(disclosure)
      = digestOf({salt, path, value: canonicalJson({cid,docHash,meta})})   [§3.3 RULE-K-CANON]
      kiểm tra commit' ∈ credential.sd[]   → tài liệu thuộc hồ sơ đã ký (đường digestOf gốc)
  (c) fetch EnvelopeR theo cid → eciesOpen bằng khóa R → plaintext P
  (d) kiểm tra blake2b256(P) == docHash        ← INVARIANT-K1: bytes khớp commitment
  (e) hiển thị. Các path KHÔNG trong disclosed → R không có commit, không có bytes → ẩn hoàn toàn.
```

**Tính chất:**
- **Least disclosure**: R chỉ nhận Envelope của đúng tài liệu đã chọn. Không có cách suy ra tài liệu khác (chúng chỉ là commitment ẩn trong `sd[]`, R không có salt/opener).
- **Ràng buộc người nhận + chống replay**: `aud`/`nonce` ký cùng presentation — R không chuyển tiếp cho bên thứ ba (kế thừa `disclose.ts`).
- **Giới hạn mục đích + hết hạn**: `purpose`/`exp` ký kèm (VeData-aligned) — R không dùng ngoài mục đích, hết hạn thì void.
- **Thu hồi (giới hạn nghiêm trọng — đọc §9.0):** chủ DID gọi `LampNetStore.revoke(cidR, R.did)` (ACL) và/hoặc set bit trong `StatusListDocument` (supersession) — R mất quyền đọc bản re-seal *lần sau*. NHƯNG bước (d) buộc R giải mã P để verify → R đã thấy plaintext; "thu hồi" chỉ chặn đọc-lại, KHÔNG lấy lại được ảnh R đã xem/lưu. Với ảnh tùy thân đây là tấn công CHÍNH (§9.0), không phải rìa.

**Chi phí:** re-seal là O(kích thước tài liệu) một lần cho mỗi người nhận. Không đụng chain. Không lộ khóa gốc của chủ DID (mỗi lần một ephemeral key).

---

## §7. Tích hợp VeData — qua đúng intake, từng module một

**KHÔNG có "tích hợp ngầm" như một data-plane cắm thẳng.** Contract-của-record (§12) chỉ cấp DID-resolution API + Stamp intake + Query gateway công khai. Vì vậy §7 phân hai loại rõ: (i) tích hợp **thoả mãn hôm nay** qua Stamp (đẩy) + Query (đọc) — anchor (§7.3), read (§7.4); (ii) tích hợp **blocked-on-contract** cần VeData team hoặc chạy on-device — Spectra (§7.1), Glint chủ-động (§7.2). Bảng phân loại đầy đủ ở **§12**. "Sau hậu trường" ở đây nghĩa là KHÔNG chặn UX upload; kết quả về sau dưới dạng huy hiệu/cờ. Knowme không gửi plaintext ra ngoài trừ khi chủ DID đồng ý (verify cần plaintext → chạy on-device hoặc TEE, xem §10, §12).

### 7.1 Spectra — tán media để verify  🟡 blocked-on-contract (§12)

**Trạng thái:** không có contract cho app-ngoài gửi `content_cid` thẳng Spectra (§12). Đường hợp lệ hôm nay: **chạy on-device** (ảnh không rời máy), hoặc media đi kèm StampRecord để Spectra chạy trong pipeline VeData, hoặc chờ VeData team mở hợp đồng.
**Gửi đi (nếu/khi có contract, hoặc on-device):** `content_cid` của tài liệu (ảnh/scan) + gợi ý loại (`docType`). Với ảnh nhạy cảm, phần tán chạy trên thiết bị hoặc Splash-in-TEE; Knowme chỉ gửi CID ra ngoài nếu chủ DID bật "cho phép kiểm tra chất lượng" VÀ contract cho phép.
**Nhận về (INV-SP4 — Spectra chỉ giữ `content_cid`, không giữ bytes):**
```ts
export interface SpectraDecomposition {
  contentCid: CID;
  regions: Array<{ bbox: [number,number,number,number]; kind: string }>; // vd "mrz", "photo", "text-block"
  qualityHint: number;    // độ rõ nét (blur/glare) — để nhắc chụp lại
}
```
**Dùng để:** cắt vùng MRZ hộ chiếu / vùng ảnh chân dung / vùng chữ để hiển thị và để Glint chấm đúng vùng; nhắc người dùng "ảnh mờ, chụp lại". KHÔNG quyết định pháp lý.

### 7.2 Glint — xác thực nguồn gốc media → `AuthenticityProof`  🟡 chủ-động blocked-on-contract (§12)

**Trạng thái:** Glint chạy TỰ NHIÊN trong pipeline Stamp (media kèm StampRecord → A10 AI_INFERENCE) — đường này ✅ có sẵn khi anchor qua Stamp (§7.3). Nhưng nếu Knowme muốn *chủ động* gọi Glint cho một `content_cid` rời (không qua Stamp), thì **cần VeData team mở hợp đồng** (§12) — không có contract app-ngoài gọi Glint trực tiếp.
**Gửi đi (trong pipeline Stamp, hoặc khi có contract):** `content_cid` (media thô, CIDv1 raw codec `0x55`).
**Nhận về** (đúng contract Glint-Feat-Spec §F6, KHÔNG redefine):
```ts
export interface AuthenticityProof {
  content_cid: CID;
  origin_class: "R" | "S0" | "S1" | "H0" | "H1"; // thật / synth-phi-AI / synth-AI / lai-AI / lai-CGI
  confidence_tier: "high" | "medium" | "low" | "uncertain"; // 4-value SSoT Glint-Feat §F1.3 — KHÔNG gộp "low"
  zk_proof_cid?: CID;    // Halo2 proof, có thể verify on-chain (Phase 2)
  model_id: string;
  ood_flag: boolean;
}
```
**`confidence_tier` là enum 4-value SSoT** (Glint-Feat §F1.3, khớp Glint-Math v0.2.0 §4): `high` (s≥0.85) / `medium` (0.65≤s<0.85) / `low` (0.45≤s<0.65) / `uncertain` (s<0.45). Bản Knowme cũ bỏ `low` — SAI: `low` cần cho dispute weighting (belief mass δ(low)=0) và KHÔNG được gộp vào `medium`/`uncertain`. Knowme hiển thị `low` như một tầng tín hiệu riêng ("độ tin thấp"), `uncertain` → Glint đẩy human review (F8), Knowme chỉ hiện cờ.
**Dùng để:** gắn cờ "ảnh giấy tờ này là ảnh chụp thật (R)" vs "ảnh bị dựng AI (S1)" — chống nộp giấy tờ **deepfake/photoshop**. Đây là lằn phòng thủ chính của self-declared: người ta tự khai, nhưng Glint phát hiện media tổng hợp.
**Ranh giới quan trọng:** `origin_class ⫫` tính hợp lệ pháp lý. Glint nói "media thật hay AI", KHÔNG nói "CCCD này có giá trị pháp lý". Một CCCD thật chụp lại (R) vẫn có thể là CCCD của người khác — đó là việc của bên xác minh danh tính, không phải Glint. Knowme chỉ hiển thị `origin_class` như một tín hiệu, không kết luận thay người dùng.
**Khi mâu thuẫn:** nếu chủ DID khai "ảnh gốc" nhưng Glint trả S1 → Glint trigger A12 `authenticity_dispute` (contract của Glint); Knowme hiển thị cờ cảnh báo, KHÔNG tự khóa.

### 7.3 Anchor on-chain — QUA STAMP, không gọi thẳng Mosaic

**🔴 Sửa lỗi bịa-interface (cross-review).** Mosaic intake CHỈ nhận `anchor_request` từ **Stamp module**; `anchor_request` schema + `anchor_priority` 4-enum là SSoT của Stamp (Mosaic-Feat-Spec §0.5 "INHERIT UPSTREAM — Stamp module: `anchor_request` schema + `anchor_priority` 4-enum SSoT"; §6 F1 "anchor_request intake API — HTTP/queue endpoint nhận request **từ Stamp**"). Không có đường cho một app bên-thứ-ba gửi `anchor_request` thẳng cho Mosaic. Bản Knowme cũ định nghĩa một `KnowmeAnchorRequest` gửi trực tiếp Mosaic — đó là **interface bịa**, đã bỏ.

**Đường đúng: Knowme phát một StampRecord, Stamp đóng dấu rồi tự phát `anchor_request` sang Mosaic.**

```
Chủ DID bấm "công chứng hồ sơ"
   │
   ▼ (1) Knowme dựng StampRecord (envelope Stamp) tham chiếu Strata head:
   │        content_cid  = CID của một notarization-record (không phải plaintext tài liệu)
   │        payload      = { ref_id, head_version_hash, stateRoot(32B), seq }
   │        producer_did = did:phoenix:* của chủ DID (ký Ed25519)
   │        d9_privacy   = "private"   (mặc định — §7.4)
   │        d10          (compliance dimension) tùy loại giấy tờ
   │        anchor_priority gợi ý (Knowme KHÔNG chốt — Stamp tính lại theo Quality+Security rule)
   │
   ▼ (2) POST StampRecord → Stamp intake. Stamp chạy pipeline 11 bước
   │        (Stamp-Feat §F2.12): validate schema/DID/chữ ký → COMPUTE anchor_priority
   │        (Stamp là bên gán, KHÔNG phải Knowme) → decay_hint.
   │
   ▼ (3) Stamp Step 11 EMIT (F2.12.8):
   │        ScorePackage → Score
   │        anchor_request → Mosaic      ← ĐÂY là nơi anchor_request sinh ra, từ Stamp
   │        stamp_record_stored → LampNet
   │
   ▼ (4) Mosaic route theo strategy A/B/C/D, viết 32-byte stateRoot vào datum Entity NFT,
   │        push event xuống downstream (Score, Query, và Stamp callback anchor_confirmed).
   │
   ▼ (5) Knowme quan sát trạng thái anchor QUA QUERY (§7.4 — event ANCHOR_CONFIRMED
          của `stamp_id`), KHÔNG subscribe Mosaic trực tiếp.
```

**Ràng buộc dùng đúng contract (không redefine):**
- **anchor_priority do Stamp gán**, không do Knowme. Stamp có "Quality + Security Override rule" (Stamp-Feat §F2.7, Stamp-Math §5). Knowme chỉ *gợi ý* qua `legal_tier_hint`/dispute flag (Stamp P9.21.h pattern) — "công chứng gấp" → Stamp có thể chọn `immediate`; nháp → `no_anchor` (Stamp RULE Anchor-D: Mosaic MUST NOT tx, content TTL 60 epoch).
- **32-byte root vào datum**: Knowme đưa `stateRoot`/`mmr_root` Strata head làm nội dung neo; Mosaic giữ 32-byte trong datum Entity NFT (CIP-68, 91–640 byte bất biến theo độ sâu lịch sử). Không đẩy metadata tài liệu lên chain.
- **seq chống rollback**: `seq` version đi trong StampRecord payload; tính đơn điệu là bất biến Strata (INV-E7) + Mosaic `last_nonce` monotonic (Mosaic F2). Knowme không tự enforce on-chain.

**Nhận về:** không có kênh Mosaic-trực-tiếp cho Knowme. Trạng thái anchor về qua **Query** `subscribe` event `ANCHOR_CONFIRMED` theo `stamp_id` (Query-Feat §F8), hoặc qua `resolve(stamp_id)` (§7.4) — có sẵn privacy gate + audit.

**Dùng để:** bằng chứng vĩnh viễn "hồ sơ tài liệu của DID X có root R tại slot S" cho pháp lý/compliance, không lộ nội dung (datum 32 byte). eUTXO: nhiều tài liệu → một Strata root → một StampRecord → một datum; không đẻ UTXO mỗi tài liệu.

**Trạng thái phụ thuộc:** đường này chỉ chạy được khi Knowme phát được một **StampRecord hợp lệ**. Đây là intake của chính VeData (Stamp) → thoả mãn được HÔM NAY mà KHÔNG cần mở rộng hợp đồng PhoenixKey↔VeData (§12 hàng "Anchor qua Stamp"). Điều kiện: có schema StampRecord cho "notarization của Strata head" (cần đối chiếu 21 archetype Stamp — xem §12 open item).

### 7.4 Query — read gateway (D9 privacy + DP)

**Gửi đi:** khi một bên xác minh muốn đọc một record đã được chia sẻ, họ gọi Query `resolve(stamp_id)` thay vì gọi thẳng LampNet + Mosaic + Score. **Keying là `stamp_id`, KHÔNG phải `ref_id`** — Query là read-gateway cho record của Stamp (đã đóng dấu tại §7.3), không phải cho Strata ref trực tiếp. Bản Knowme cũ dùng `resolve(ref_id)` với result-shape tự bịa → đã sửa về đúng chữ ký Query-Feat §F1.

**Chữ ký thật (Query-Feat §F1, SSoT):** `resolve(stamp_id) → { content, merkle_proof, v_r, privacy_verdict, audit_id }` trong < 1 s p95 cached; fan-out song song LampNet + Mosaic + Score.

**Query enforce (KHÔNG redefine — Query-Feat-Spec §F4):**
- **D9 privacy predicate**: `access(record, consumer_did)` — fail-closed. Hồ sơ Knowme mặc định `private`; chỉ path đã disclose qua Presentation mới `role_based`/`proof_only`. Query chặn TRƯỚC khi expose content.
- **Differential privacy** cho truy vấn **tổng hợp** (vd "bao nhiêu OrgDID có giấy phép loại X") — Gaussian mechanism, `ε_max = 2.0`/consumer/5-min-window, lifetime cap `ε_lifetime = 10.0`/producer. Ngăn suy ngược cá nhân từ thống kê.
- **Audit log MMR** tamper-evident: mọi lần đọc được ghi, root neo Mosaic mỗi 5 phút — chủ DID thấy ai đã xem giấy tờ mình.
- **Constant-size padding** (`B_pad = 4096`) + latency normalization — chống side-channel đoán loại/kích thước tài liệu.

```ts
// Knowme chỉ CONSUME chữ ký Query-Feat §F1 — KHÔNG cài lại, KHÔNG đổi field:
// resolve(stamp_id) → QueryResolveResult
export interface QueryResolveResult {
  content: unknown;              // record bytes đã qua D9 gate (⊥ nếu access=deny)
  merkle_proof: unknown;        // Mosaic proof (verify read-only, độc lập operator)
  v_r: number;                  // V(r) của Score cho record
  privacy_verdict: "allow" | "deny";   // kết quả access(record, consumer_did)
  audit_id: string;             // trace vào audit-log MMR (đọc được là ai xem gì)
}
```
Ánh xạ Knowme: `stamp_id` là của record đã đóng dấu ở §7.3 (notarization của Strata head). Bên xác minh cầm `stamp_id` (nhận qua Presentation/kênh chia sẻ) → `resolve(stamp_id)` → có `content` (chỉ khi được phép), `merkle_proof` để verify anchor độc lập, `v_r`, `privacy_verdict`, `audit_id`. Tham số DP ở trên (`B_pad = 4096`, `ε_max = 2.0`, `ε_lifetime = 10.0`) đúng SSoT Query-Math v0.2.8 — giữ nguyên.

**Dùng để:** một endpoint duy nhất cho bên xác minh, có privacy gate + audit + DP — thay vì mỗi app tự glue nhiều round-trip (dễ rò rỉ).

---

## §8. UI SuperApp

Module **Knowme** nằm trong hồ sơ cá nhân của SuperApp (tab "Giấy tờ của tôi").

**Màn hình chính — Kho tài liệu:**
- Lưới thẻ tài liệu theo nhóm (Giấy tờ tùy thân / Bằng cấp / Doanh nghiệp / Thiết bị-sản phẩm tùy DID class).
- Mỗi thẻ: nhãn + `docType` + huy hiệu trạng thái: 🟢 "media thật (R)" / 🟡 "đang kiểm tra" / 🔴 "nghi dựng (S1)" (từ Glint §7.2), + "đã công chứng slot S" nếu anchored (§7.3).
- Chưa chia sẻ = ổ khóa; mọi tài liệu ở trạng thái mã hóa (chỉ chủ DID mở).

**Thao tác:**
- **Thêm tài liệu**: chọn ảnh/PDF → chọn `docType` → seal+store+version (§4-5) chạy nền, hiện tiến trình.
- **Cập nhật**: chọn tài liệu → thay ảnh → tạo version mới; timeline hiện các bản cũ (Strata `version_at`), có thể xem bản cũ (read-only).
- **Chia sẻ chọn-lọc**: chọn tài liệu → quét QR/nhập DID người nhận → chọn `purpose` + `exp` → tạo Presentation (§6). Hiện rõ "Người nhận sẽ thấy: [2 tài liệu này]. KHÔNG thấy: [phần còn lại]".
- **Nhật ký truy cập**: ai đã xem tài liệu nào, khi nào (từ Query audit log §7.4).
- **Thu hồi**: nút thu hồi từng chia sẻ (revoke ACL + status bit §6.2), kèm cảnh báo "bản đã tải về không thu về được".

**Nguyên tắc UX:**
- Không bao giờ hiện "server đang giữ ảnh của bạn" — vì server chỉ giữ ciphertext.
- Mọi tiết lộ đều là hành động chủ động, có màn hình xác nhận liệt kê chính xác cái gì lộ.
- Huy hiệu Glint là **tín hiệu**, không phải phán quyết — không tự động khóa/từ chối; để người xác minh quyết định.

---

## §9. Phản biện đối kháng (self-adversarial)

### 9.0 🔴 TOP-LINE: bán lại plaintext sau khi verify (verify-time plaintext resale)

**Đây là tấn công CHÍNH cho giấy tờ tùy thân, không phải rìa.** Cơ chế: người nhận R muốn kiểm tra tài liệu thật → R BUỘC phải giải mã plaintext P (ảnh CCCD/hộ chiếu) để tính `blake2b256(P) == docHash` (§6.2 bước d). Một khi R đã có P trong tay (đã hiển thị lên màn hình cho người xem), **"thu hồi" chỉ còn tính mỹ phẩm**: R có thể chụp màn hình, lưu, và **bán lại/tái sử dụng ảnh** vô thời hạn. Revoke ACL (§6.2) chỉ chặn *lần đọc sau*, không lấy lại được cái R đã thấy.

- **Vì sao không vá triệt để được:** ràng buộc nội dung (INV-K1) yêu cầu R tự băm P để tin tài liệu → verify **bắt buộc lộ P cho R**. Đây là mâu thuẫn cố hữu của mọi tài liệu **người-xem-được**: nếu mắt người phải đọc được ảnh để tin, thì phần mềm không thể ngăn mắt/máy ảnh sao chép. Không có phép mã hóa nào cứu được — bài toán về cơ bản **không giải được** cho tài liệu human-viewable. Nói thẳng: Knowme KHÔNG hứa chống được việc này.
- **Giảm thiểu (thành thật, hạn chế):**
  - **Ràng-buộc-người-nhận có thể chứng minh (đối kháng bán lại):** re-seal cho R (§6.1) + đóng **watermark/steganographic** buộc-danh-tính-R vào bản R nhận (vd nhúng `R.did + nonce + timestamp` mờ). Không ngăn được sao chép, nhưng nếu ảnh rò rỉ, watermark **truy được R là nguồn** → răn đe pháp lý, KHÔNG phải chặn kỹ thuật. Đánh đổi: watermark bền vs. chất lượng ảnh; attacker có thể crop/tái-nén để phá.
  - **Least disclosure + purpose/exp (§6):** giảm *bề mặt* — R chỉ thấy đúng tài liệu cần, hết hạn thì lần-đọc-sau bị chặn. Không lấy lại bản đã tải.
  - **Ưu tiên proof thay vì ảnh:** khi bên xác minh chỉ cần một *thuộc tính* (vd "trên 18 tuổi", "quốc tịch VN"), dùng **issued credential + selective disclosure trường vô hướng** (§9.2 tầng 2) hoặc `proof_only` (Query D9) để KHÔNG bao giờ lộ ảnh gốc. Đây là đường thoát thực sự: **đừng gửi ảnh nếu có thể gửi proof.** Với tài liệu tùy thân, mặc định nên nghiêng về proof-of-attribute, chỉ lộ ảnh khi luật buộc.
  - **Pháp lý/hợp đồng:** `purpose` ký kèm presentation là ràng buộc dùng-đúng-mục-đích có thể viện dẫn; không phải cơ chế kỹ thuật.
- **Chốt thiết kế:** với ảnh giấy tờ, coi mọi lần chia sẻ-ảnh là **lộ một-chiều không thu hồi được**. UI PHẢI nói thẳng điều này ("Người nhận sẽ thấy ảnh thật và có thể lưu lại; bạn không thu hồi được bản họ đã xem"). Đường an toàn duy nhất là **không gửi ảnh** — gửi proof thuộc tính. Watermark = truy vết, không phải phòng ngừa.

### 9.1 Rò rỉ riêng tư (privacy leak)

- **Metadata trong commitment.** `DocumentMeta` (docType, mime, bytesLen) đi vào commitment. Commitment là hash → không lộ giá trị; NHƯNG nếu attacker đoán được `(salt, path, docHash, meta)` thì xác nhận được. `salt` 16 byte ngẫu nhiên chặn brute-force meta ít-entropy. **Chốt:** `docHash` (32 byte high-entropy) trong preimage đã đủ; salt là lớp đệm.
- **CID lộ loại?** Không — CID = BLAKE3(ciphertext), INV-E5. Nhưng **kích thước ciphertext** vẫn lộ (ảnh CCCD ~vài trăm KB vs PDF bằng cấp ~vài MB). Giảm thiểu: padding kích thước Envelope tới bậc chuẩn (khối 256KB) trước khi seal — chi phí lưu trữ đổi lấy chống suy loại. **[Chờ anh chốt: có bật padding kích thước không?]**
- **Query đọc lộ pattern.** Đã có `B_pad=4096` + latency normalization + DP cho aggregate (§7.4). Đủ cho read gateway.

### 9.2 Giấy tờ giả / khai sai (self-declared)

- **Bản chất self-declared**: người dùng tự khai, KHÔNG có cơ quan ký mặc định. Rủi ro: nộp ảnh giấy tờ của người khác, hoặc ảnh dựng.
- **Phòng thủ tầng 1 — Glint** (§7.2): phát hiện media **tổng hợp/AI** (S1/H1). Chặn được deepfake/photoshop, KHÔNG chặn được "ảnh thật của người khác".
- **Phòng thủ tầng 2 — issued credential** (đã có trong `credential.ts`, `buildCredential` với `issuerDid`): khi cần độ tin cao, một cơ quan (vd Công an cấp CCCD) ký một `DocumentClaim` → chuyển từ self-declared sang issued. Knowme tái dùng nguyên đường issued của SD-VC; chỉ khác value là tài liệu. Trust chain qua `TrustRecord`/`TrustListDocument` (đã có).
- **KHÔNG dùng dedup** để phát hiện "một người nộp nhiều DID" — trái nguyên tắc dự án (🔴 §1.3). Đây là chủ ý: Knowme là kho tài liệu, không phải cảnh sát danh tính.
- **Chốt thiết kế:** Knowme không giả vờ chống được mọi gian lận. Nó cung cấp *tín hiệu* (Glint) + *đường nâng cấp* (issued). Kết luận cuối thuộc bên xác minh.

### 9.3 Giả mạo phiên bản (version tampering)

- **Sửa lịch sử**: chặn bởi Strata INV-E1 (hash-link) + INV-E2 (seq đơn điệu). Không thể chèn/sửa version giữa chuỗi mà không vỡ hash-link.
- **Rollback** ("khôi phục" giấy tờ đã thay bằng bản cũ có lợi): chặn bởi INV-E7 — anchor đơn điệu theo seq; Mosaic datum `seq` chỉ tăng. Neo lại root cũ bị từ chối.
- **Fork chuỗi** (hai head song song): chặn bởi `prev_hash == head_version_hash` (`HashLinkBroken`).
- **Malleability chữ ký**: Strata yêu cầu Ed25519 canonical low-S (`verify_strict`) — không tráo được sig.
- **Đổi tài liệu nhưng giữ commitment cũ**: bất khả — `docHash` trong commitment ràng buộc bytes (INV-K1). Đổi bytes ⇒ đổi docHash ⇒ đổi commitment ⇒ vỡ membership.

### 9.4 Tương quan metadata (metadata correlation)

- **Nối nhiều tài liệu về cùng một người qua timing/CID.** LampNet không index theo chủ, nhưng nếu attacker quan sát được nhiều `put` từ cùng một phiên → suy ra chùm. Giảm thiểu: đẩy Envelope qua LampNet với độ trễ ngẫu nhiên / batch; không gắn owner_did vào request lưu trữ (chỉ CID).
- **Nối qua `ref_id`.** `ref_id = gen_ref_id(ownerDidHash, nonce)` — opaque, KHÔNG lộ DID (INV-E5 Strata). Nhưng nếu chủ DID công khai `ref_id` cho nhiều bên, các bên đối chiếu được "cùng một hồ sơ". Giảm thiểu: **per-recipient ref alias** — không tiết lộ `ref_id` gốc; chia sẻ qua Presentation + Query resolve, không qua ref_id trần. **[Chờ anh chốt: có cần per-context ref alias giống external-nullifier không?]**
- **Nối qua anchor slot.** Anchor `immediate` để lại dấu thời gian on-chain có thể tương quan. Giảm thiểu: mặc định `batch_daily` (gộp nhiều DID trong một batch root → k-anonymity theo lô). `immediate` chỉ khi chủ DID chủ động.
- **Nối qua kích thước** — xem §9.1 (padding).

---

## §10. Quyết định chờ anh chốt

1. **Padding kích thước Envelope** (§9.1): có bật padding tới khối chuẩn (256KB) để chống suy-loại-qua-kích-thước không? Đánh đổi: tốn dung lượng LampNet. Đề xuất: bật cho tài liệu nhạy cảm (docType tùy thân), tắt cho phần còn lại.

2. **Per-context ref alias** (§9.4): có làm `ref_id` alias theo ngữ cảnh (giống external-nullifier per-context) để hai bên nhận không đối chiếu được "cùng hồ sơ" không? Tăng độ riêng tư, tăng độ phức tạp SDK.

3. **Ngưỡng anchor mặc định** (§7.3): `batch_daily` cho mọi hồ sơ, hay để người dùng chọn? Đề xuất: mặc định `batch_daily`; nút "công chứng gấp" = `immediate`; nháp = `no_anchor`.

4. **Glint on/off theo loại tài liệu** (§7.2): chạy Glint cho MỌI tài liệu, hay chỉ ảnh tùy thân? Chạy Glint tốn compute (Splash) + có thể cần gửi CID media ra dịch vụ. Đề xuất: mặc định BẬT cho ảnh/scan, TẮT cho PDF text-only.

5. **Nơi chạy Spectra/Glint cho ảnh nhạy cảm** (§7.1, §7.2, §9.0, §12): một số verify cần plaintext (PRNU, MRZ). Chạy on-device hay Splash-in-TEE? On-device an toàn hơn nhưng nặng máy; TEE cần hạ tầng — VÀ đây là hai mục **blocked-on-contract** (§12): gọi Spectra/Glint chủ-động cần VeData team mở hợp đồng. Đề xuất: on-device cho ảnh tùy thân (không phụ thuộc contract). **Cần anh chốt kiến trúc verify để không vô tình gửi plaintext ra ngoài.**

6. **Đường issued cho DocumentClaim** (§9.2): có mở API để cơ quan (Công an, Sở KHĐT) ký `DocumentClaim` ở phase này, hay để phase sau? Ảnh hưởng lộ trình tích hợp cơ quan.

7. **Loại DID class Device/Product**: hiện `SubjectKind` chỉ có person/org. Thêm byte DID type cho Device/Product (`0x02`?) là việc của backend/Math — Knowme cần schema `phoenix-knowme-device/1`. **Báo anh giao Long mở DID type byte, Knowme không tự sửa backend.**

---

## §11. Tóm tắt bất biến Knowme

- **INV-K1** (ràng buộc nội dung): `docHash = BLAKE2b-256(plaintext)` nằm trong commitment; đổi bytes ⇒ vỡ membership.
- **INV-K2** (không plaintext ra ngoài): chỉ `Envelope` (ciphertext ECIES) và CID rời máy; server chỉ giữ ciphertext.
- **INV-K3** (tài liệu là claim): DocumentClaim là một leaf trong `sd[]` → selective disclosure phủ tài liệu miễn phí, không cần cơ chế riêng.
- **INV-K4** (lịch sử bất biến): mọi cập nhật = version Strata mới; bản cũ giữ nguyên; proof field-level (kế thừa các bất biến Strata `_CONTRACT.md` công bố: INV-E1, E2, E3, E5, E6, E7).
- **INV-K5** (least disclosure): người nhận chỉ có Envelope + opener của đúng tài liệu đã chọn; phần còn lại là commitment ẩn.
- **INV-K6** (không dedup): Knowme KHÔNG chạy uniqueness/sybil trên PersonDID. 🔴 Ranh giới cứng.

---

## §12. Ranh giới hợp đồng VeData — cái gì làm được HÔM NAY, cái gì blocked-on-contract

**Vấn đề cross-review (🔴):** hợp đồng-của-record `PhoenixKey-VeData-Contract.md` **v0.2.0** (SSoT) chỉ cấp cho VeData một tập **API resolve DID** từ PhoenixKey — KHÔNG cấp một data-plane cấp-ứng-dụng để Knowme "tích hợp ngầm" đẩy dữ liệu tài liệu vào Strata/Mosaic/Query. Cụ thể hợp đồng §2.1 + §2.6 chỉ liệt kê:

- `did_active(did, epoch)`, `key_authorized(key, did, epoch)`, `revocation_check(did, epoch)`, `get_dependency_chain`, `get_persondid_lineage`, `get_ownership_graph_snapshot`, `are_independent`, `shared_ancestor`.
- Pubsub 4 topic: `did_created`, `did_lineage_updated`, `did_revoked`, `device_did_bound_to_person`.

Đây đều là **query danh tính**, chiều PhoenixKey→VeData. KHÔNG có API để một app đẩy content/anchor/query record. Nghĩa là mọi tích hợp §7 phải phân loại theo một trong hai đường:

**(a) Thoả mãn HÔM NAY qua intake của chính VeData (Stamp).** VeData đã có cửa nhận dữ liệu: Stamp (đóng dấu record → phát ScorePackage/anchor_request/stored). Bất cứ gì Knowme cần đẩy vào VeData mà đi qua một **StampRecord hợp lệ** thì KHÔNG cần mở rộng hợp đồng — nó là public intake của Stamp. Verify độ thật của tích hợp: đọc contract Stamp/Mosaic/Query/Glint, không dựa "tích hợp ngầm".

**(b) Blocked-on-contract — cần VeData team mở rộng hợp đồng.** Bất cứ gì cần một API Knowme-riêng bên VeData (không phải Stamp intake, không phải Query resolve công khai) đều CHƯA có contract → phải gửi yêu-cầu-mở-rộng-hợp-đồng cho VeData team, KHÔNG được giả định tồn tại.

### Bảng phân loại từng tích hợp §7

| Tích hợp | Đường | Trạng thái | Ghi chú |
|---|---|---|---|
| **Anchor on-chain** (§7.3) | Knowme phát StampRecord → Stamp EMIT `anchor_request` → Mosaic | ✅ **Thoả mãn qua Stamp** (không cần mở hợp đồng) | Điều kiện: có schema StampRecord cho "notarization của Strata head" trong 21 archetype Stamp. Đây là **open item** (chọn archetype phù hợp, vd ENTITY_RECORD/VERIFICATION_RESULT) — công việc schema, KHÔNG phải mở data-plane mới. |
| **Query read** (§7.4) | Bên xác minh gọi `resolve(stamp_id)` | ✅ **Thoả mãn qua Query công khai** (Query-Feat §F1) | Query là gateway công khai cho consumer; keying `stamp_id`. Không cần API Knowme-riêng. Chỉ đọc được record ĐÃ qua Stamp (§7.3). |
| **Glint** authenticity (§7.2) | Gửi `content_cid` media → nhận `AuthenticityProof` | 🟡 **Blocked-on-contract, cần VeData team** | Glint intake `content_cid` là từ LampNet/Rada→Stamp pipeline nội bộ VeData; không có contract cho app-ngoài gửi CID thẳng Glint. Đường hợp lệ hôm nay: media đi kèm StampRecord (§7.3) → Glint chạy trong pipeline Stamp (A10). Muốn Knowme *chủ động* gọi Glint cho một CID rời → **cần hợp đồng mới**. |
| **Spectra** decompose (§7.1) | Gửi `content_cid` → nhận `SpectraDecomposition` | 🟡 **Blocked-on-contract, cần VeData team** | Tương tự Glint: không có contract app-ngoài. Đường an toàn: chạy on-device (không rời máy) hoặc chờ contract. Với ảnh tùy thân, on-device là đường ưu tiên bất kể (§9.1, §10). |
| **Query subscribe** ANCHOR_CONFIRMED (§7.3, §7.4) | `subscribe` theo `stamp_id` | ✅ **Thoả mãn qua Query công khai** (Query-Feat §F8) | Nhận trạng thái anchor qua đây thay vì subscribe Mosaic trực tiếp. |
| **DID validity** cho record (nền) | `did_active` / `revocation_check` / `key_authorized` | ✅ **Có trong hợp đồng §2.1** | Đây đúng là thứ hợp đồng v0.2.0 cấp. Stamp dùng nó ở Step 3 để validate producer_did của StampRecord Knowme phát. |

### Chốt §12

1. **KHÔNG giả vờ có data-plane.** Bỏ ngôn ngữ "tích hợp ngầm" như thể Knowme cắm thẳng vào Mosaic/Glint/Spectra. Đường duy nhất KHÔNG cần mở hợp đồng là **qua Stamp (đẩy) + Query (đọc)**.
2. **Hai mục 🟡 (Glint chủ-động, Spectra chủ-động)** là **blocked-on-contract**: cần gửi yêu-cầu-mở-rộng cho VeData team, hoặc giữ verify **on-device** để không phụ thuộc contract. Đề xuất: on-device cho ảnh tùy thân (trùng khuyến nghị §10 mục 5), gửi VeData chỉ khi chủ DID đồng ý và contract có.
3. **Open item cho anchor qua Stamp:** cần chọn/định nghĩa archetype StampRecord cho "notarization Strata head". Báo anh giao đối chiếu 21 archetype Stamp (Stamp-Feat §F2.2) — Knowme KHÔNG tự thêm archetype (đó là SSoT của Stamp).
