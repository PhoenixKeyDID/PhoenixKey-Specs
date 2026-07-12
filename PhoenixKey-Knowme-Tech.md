# PhoenixKey — Knowme (Đặc tả KỸ THUẬT)

> **Đối tượng:** KỸ SƯ triển khai (Frontend/SDK, đội on-chain/ZK, đội backend, VeData ZK, MAGIC-CARP không liên quan module này).
> **Mục đích:** kiến trúc + schema + luồng + API + ranh giới giao việc + thứ tự deploy — đủ để code, không cần đọc lại Feat-Math.
>
> **Nguồn đối chiếu:**
> - `PhoenixKey-Frontend/src/lib/sdvc/` (Mức 1+2) — `credential.ts`, `disclose.ts`, `commit.ts`, `canonical.ts`, `didDoc.ts`, `didResolver.ts`, `statusList.ts`, `trust.ts`, `trustList.ts`, `crypto.ts`, `lampnet.ts`, `fingerprint.ts`, `dossier.ts`, `consent.ts`, `schema.ts`, `schemaRegistry.ts`, `anchor.ts`, `did.ts`, `types.ts`, `index.ts`.
> - `PhoenixKey-Frontend/src/app/verify/page.tsx` + `src/components/vc/{ui,DeclareForm}.tsx` — luồng `/vc`.
> - Nguồn thiết kế nội bộ (không công khai): Knowme-Feat-Math (lớp tài liệu — **[SPEC]**); KYC-KYB-ZK-Feat-Math (Mức 3 BBS+ [BBS-CRYPT] — chuẩn W3C, tại thời điểm viết là Editor's Draft chưa Recommendation — **[SPEC]**).
>
> **Ranh giới code/spec:** Mức 1+2 nằm trong `src/lib/sdvc/`. Mức 3 ZK (`src/lib/sdvc/zk/`, `ZkCredential`/`BBS`/`ProveDerived`) là **[SPEC]** — thiết kế riêng, chưa gộp vào cây thư mục lõi.
>
> → Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)
>
> **Lưu-ý lớp tài liệu:** một lớp tài liệu SƠ KHAI nằm trong `dossier.ts` (`DocumentRecord`/`Dossier`/`buildDocumentRecord`/`dossierServiceEntry`), `fingerprint.ts` (`documentFingerprint` one-hash-per-doc + `UniquenessRegistry`), lưu ảnh mã hoá lên LampNet qua `crypto.ts:eciesSeal` + `lampnet.ts:cidOf` (BLAKE3), neo dossier vào DID Document qua service-entry `PhoenixDossier`. **KHÁC** với thiết kế `DocumentClaim`/`docDigestOf` trong Feat-Math §3 (gập `docHash` vào `sd[]` để tiết lộ chọn lọc chung membership). Nghĩa là: (a) đính ảnh mã hoá + fingerprint-tài liệu = khung sơ khai trong `dossier.ts`/`fingerprint.ts`; (b) tiết lộ chọn lọc tài liệu qua `sd[]` (RULE-K-CANON) + re-seal per-recipient + versioning Strata = **[SPEC]**. Doc này giữ ký hiệu spec (`DocumentClaim`/`docDigestOf`) cho phần (b); phần (a) ghi rõ khi có.

---

## 1. Kiến trúc

### 1.1 Sơ đồ thành phần

```
┌────────────────────── SuperApp (Knowme UI, tab "Giấy tờ của tôi") ──────────────────────┐
│  khai trường → (đính tài-liệu) → chọn trường/tài-liệu chia-sẻ → kiểm/thu-hồi            │
│  màn /vc: khai → chọn trường → disclose+seal → verify → revoke                          │
└───────────────┬──────────────────────────────────────────────────────┬──────────────────┘
                │                                                        │
        Knowme SDK (thin, cho lớp tài-liệu — [SPEC])              SD-VC lib (src/lib/sdvc/)
   seal doc→Envelope→cid ; docDigest ; version(Strata) ;     buildCredential / buildPresentation /
   re-seal cho người nhận                                     verifyPresentation / StatusList / TrustRegistry
                │                                              (BLAKE2b commit, ECIES, Merkle, DID resolve)
     ┌──────────┼───────────────────────────────────────────────────────┐
     ▼          ▼                         ▼                              ▼
  LampNet    Strata chain [SPEC]     VeData qua Stamp/Query [SPEC]    Cardano L1 (metadata-6789)
  ECIES      version + state_root     Glint/Spectra (verify),         anchor credential / ZkAnchor
  ciphertext field-Merkle proof       Query (read gateway D9+DP)      (chỉ HASH, không PII)
  content CID                         Stamp → anchor_request → Mosaic
```

**Phân định ranh giới cốt lõi (MECE):**
- **PhoenixKey (module Knowme) LÀM:** đóng gói credential (envelope), phân giải khoá (resolve DID/issuer), **tiết lộ chọn lọc** (selective disclosure), lớp tài liệu (commit + seal + version + re-seal), neo hash on-chain.
- **VeData LÀM:** catalog các loại giấy tờ (VC types) + issuer thật, cùng các module Stamp/Query/Glint/Spectra. Knowme **dẫn chiếu**, không tự dựng, không gọi thẳng Mosaic (chỉ qua Stamp intake).

### 1.2 Bất biến kiến trúc (load-bearing)

- **Một đường commitment duy nhất** (I-KNOW-1): mọi trường — vô hướng hay tài liệu — băm qua `digestOf([salt,path,value])`. Tài liệu gập `value = canon({cid,docHash,meta})` trước. KHÔNG hàm hash riêng cho tài liệu. Neo `commit.ts:digestOf`.
- **Khoá người trình lấy từ DID Document, KHÔNG từ presentation** (I-KNOW-4, chống mạo danh). Neo `disclose.ts:143` + `didResolver.ts`.
- **`aud`+`nonce` trong preimage chữ ký holder** (I-KNOW-5, anti-replay). Neo `disclose.ts:130-134`.
- **Canonical JSON đồng nhất TS ↔ Java** (sort key mọi tầng): membership cross-language phụ thuộc điều này. Neo `canonical.ts` (khớp Jackson `ORDER_MAP_ENTRIES_BY_KEYS`).
- **On-chain chỉ hash** (I-KYC-NO-PII-CHAIN): anchor credential = `H(sub‖sdRoot‖iat)`; ZkAnchor = `H(...)`; StatusList bit; DID Document (khoá công khai). KHÔNG value/PII.

---

## 2. Schema / Datum — khuôn dữ liệu

### 2.1 Trường vô hướng (`types.ts`)
```ts
type ClaimValue = string | number | boolean;      // scalar only
interface Disclosure { salt: string; path: string; value: ClaimValue; }
// digest = toB64url(blake2b256(canonicalBytes([salt, path, value])))   -- commit.ts:digestOf
```
`SDCredential`: `{ sub, iss, iat, sd: Digest[], sdRoot, proof }`. `HeldCredential` = credential + `disclosures` (opener holder giữ).

### 2.2 Tài liệu **[SPEC]** — nguồn Knowme-Feat-Math §3
```ts
interface DocumentMeta { docType: string; mime: string; bytesLen: number;
                         capturedAt?: number; label?: string; page?: number; }
interface DocumentClaim { kind: "document"; path: string;
                          docHash: string;  // b64url BLAKE2b-256(plaintext) — ràng-buộc nội-dung
                          cid: CID;          // BLAKE3(ciphertext Envelope) — con-trỏ LampNet
                          meta: DocumentMeta; }
interface DocumentDisclosure { salt: string; path: string; docHash: string; cid: CID; meta: DocumentMeta; }

// RULE-K-CANON — gập tài-liệu về value vô-hướng ĐÚNG wire format commit.ts:
function docValue(d): string { return canonicalJson({ cid: d.cid, docHash: d.docHash, meta: d.meta }); }
function docDigestOf(d): Digest { return digestOf({ salt: d.salt, path: d.path, value: docValue(d) }); }
```
**Bất biến dây (CẤM đổi):** giữ arity mảng `[salt, path, value]`; `value` là **string** `canonicalJson({cid,docHash,meta})`. Đổi → verifier theo `commit.ts` tính digest khác → membership vỡ.

### 2.3 Phiên bản (Strata) **[SPEC]** — nguồn §3.4
```ts
interface KnowmeVersionRef { refId: string; seq: number; versionHash: string;
                             stateRoot: string;  // Strata field-Merkle trên doc leaf
                             sdRoot: string;      // SD-VC Merkle root (khớp SDCredential.sdRoot)
                             createdAt: number; }
```
Bất biến ghép: tập `{path}` trong Strata state = tập `{path}` các DocumentClaim trong `sd[]` cùng `seq`.

### 2.4 Neo on-chain (metadata-6789)
- **Mức 1+2:** `anchorDigest(cred) = H(canon([sub, sdRoot, iat]))` — `credential.ts:138`. Publish cạnh DID Document.
- **Mức 3 [SPEC]:** `ZkAnchor = H(canon([sub, H(bbsIssuerKey), zkAlg, epoch]))` — KYC-KYB-ZK §B.5. KHÔNG value/DOB/fingerprint/proof.
- **Tài liệu [SPEC]:** Strata `state_root`/`mmr_root` (32 byte) neo **QUA Stamp** → Mosaic; KHÔNG gọi thẳng Mosaic (§6.4).

---

## 3. Từng thao tác — điều kiện + shape + ai ký

### 3.1 Cấp credential — `buildCredential`
- **Tự khai:** `issuerDid` bỏ trống → `iss = sub`; `signer` = khoá subject.
- **Issued:** `issuerDid` = DID cơ quan; `signer` = khoá cơ quan. Subject vẫn hold + present.
- Ra: `HeldCredential` (credential ký + openers). `sdRoot = merkleRoot(sd[])`.

### 3.2 Trình tiết lộ chọn lọc — `buildPresentation` / `verifyPresentation`
- **Build:** input `{ held, holderDid, signer, aud, nonce, paths, purpose?, exp? }`. Chỉ đưa disclosure của `paths` đã chọn; ký Ed25519 phủ `[aud, nonce, disclosed]`.
- **Verify:** kiểm (1) `aud=expectedAud`; (2) `nonce=expectedNonce`; (3) `vm=assertionKeyId(holder)` + chữ ký holder (khoá từ `holderPublicKey` = `Resolve(holder)`); (4) mỗi disclosed digest ∈ `sd[]`; (5) `¬revoked`. Ra `VerifyResult{valid, claims, reason?}`.
- **Ai ký:** holder (subject) ký presentation. Verifier KHÔNG cần khoá issuer online (self-declared) hoặc lấy issuer key từ TrustList (issued).
- **Niêm phong kênh:** `sealPresentation`/`openPresentation` (`index.ts:170`) bọc presentation ECIES cho `aud`.

### 3.3 Thu hồi — `RevocationRegistry` / `StatusList`
- **Holder-side:** `buildRevocation` theo CID → verifier từ chối presentation đó.
- **Issuer-side:** `StatusList` (W3C Bitstring, append-only) set bit; `isRetired(status,doc)` → credential retired → mọi presentation phái sinh fail.

### 3.4 Lớp tài liệu **[SPEC]** — luồng seal→store→version
```
plaintext P → docHash = blake2b256(P)                              [crypto.ts]
           → Envelope = eciesSeal(P, ownerEncPub, ownerDid, ...)    [crypto.ts]  (mã-hoá CHO CHÍNH CHỦ)
           → cid = LampNetStore.put(Envelope)                       [lampnet.ts]
           → DocumentDisclosure{salt, path, docHash, cid, meta}; docDigestOf → thêm vào sd[]; rebuild sdRoot
           → (tùy) version Strata: state_root gồm leaf mọi doc
           → (tùy) anchor state_root QUA STAMP → Mosaic (§6.4)
```
**Chia sẻ tài liệu (re-seal):** mở Envelope của mình → `eciesSeal(P, R.encPub, R.did, eph, ...)` (ephemeral key, KHÔNG lộ khoá gốc) → `cidR` → Presentation chứa `DocumentDisclosure{cid: cidR}`. Người nhận: `docDigestOf ∈ sd[]` **∧** `blake2b256(decrypt) = docHash`.

### 3.5 Mức 3 (ZK) **[SPEC]** — `ZkProve`/`ZkVerify`
- `ZkCredential = SDCredential ⊕ { zkAlg:"BBS-2023", bbsIssuer, msgOrder, bbsProof }`. `sd[]`/`sdRoot` Ed25519 GIỮ để tương thích Mức 2.
- `ZkProve(opener, predicate, aud, nonce)` → π (chỉ `result` + proof, không value). `ZkVerify` offline (không gọi issuer). Chi tiết KYC-KYB-ZK §B.2/B.3.

---

## 4. Luồng end-to-end (`/vc`)

```
1. DeclareForm: user khai trường  →  buildCredential (iss=sub)  →  HeldCredential + sdRoot
2. Chọn trường tiết-lộ  →  buildPresentation(paths, aud, nonce)  →  sealPresentation cho verifier
3. Verifier: openPresentation  →  verifyPresentation(expectedAud, expectedNonce, holderPublicKey)
             → resolve holderPublicKey qua BackendDidResolver/DidRegistry (didResolver.ts)
             → accept/reject + claims
4. Thu-hồi: RevocationRegistry.revoke(cid)  |  issuer StatusList.setBit  →  verify sau đó reject
```
Neo UI: `app/verify/page.tsx` (32KB, 5 bước), `components/vc/{DeclareForm,ui}.tsx`.

---

## 5. API backend (tham chiếu)

> Prefix `/api/v1`, JSON snake_case, bọc `DataResponse<T>{code,message,result}`. Phần lớn Knowme là **client-side/SDK**; backend chỉ cần cho resolve + (tùy) lampnet gateway + trustlist neo.

| Nhóm | Endpoint (đề xuất) | Vai |
|---|---|---|
| DID resolve | `GET /identity/{did}/document\|pubkey\|status` | verifier phân giải khoá holder/issuer (`BackendDidResolver`); point-in-time là việc đội backend (VeData-Reply-did-commit-resolve) |
| TrustList | `GET /trust/list` (+ Merkle root on-chain) | gate issuer (issued credential) |
| StatusList | `GET /credential/status/{listId}` | verifier kiểm revoke |
| LampNet gateway **[SPEC]** | `PUT/GET /lampnet/{cid}` (ciphertext) | lưu/đọc Envelope tài liệu (Backend handoff SD-VC §8) |

**Lưu ý:** resolve dùng CHUNG kênh Contract VeData v0.2.0 §2.1 (`did_active`/`key_authorized`/`revocation_check`) — cùng API MAGIC/Rada dùng (VeData-Reply-did-commit-resolve). Fail-closed: PhoenixKey down → 503 → verifier reject.

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## 6. Ranh giới giao việc

### 6.1 Frontend/SDK
- Mức 1+2: lớp lõi (`src/lib/sdvc/`). Không đụng.
- Lớp tài liệu **sơ khai:** `dossier.ts` + `fingerprint.ts` + `crypto.ts:eciesSeal` + `lampnet.ts` — đính ảnh mã hoá lên LampNet, fingerprint one-hash-per-doc, neo dossier vào DID Document.
- Lớp tài liệu **tiết lộ chọn lọc [SPEC]:** thêm `DocumentClaim`/`docDigestOf`/`docValue` (RULE-K-CANON — gập `docHash` vào `sd[]`), luồng version(Strata)/re-seal per-recipient, UI tab "Giấy tờ của tôi". Tái dùng `crypto.ts`/`lampnet.ts`/`dossier.ts`.
- Mức 3 UI **[SPEC]:** màn "Chứng minh thuộc tính" (đủ 18 / quốc tịch / DN hợp lệ) — nâng từ UI `/vc`.

### 6.2 Đội on-chain / VeData ZK
- **BBS prover/verifier lib [SPEC]:** bọc lib BBS (BLS12-381) đã kiểm định (MATTR/Digital Bazaar) — **KHÔNG tự viết mạch**. Thêm `src/lib/sdvc/zk/{bbs,predicate,zkProve,zkVerify}.ts`. Framework-free để lift vào PhoenixKey-SDK.
- Predicate spec + eval (`age_over`/`nat_in`/`org_active`) map sang BBS derived-proof.
- OID4VP request/response (`presentation_definition` → `vp_token`).

### 6.3 Đội backend
- Resolve API point-in-time (Contract v0.2.0 §2.4) — cùng kênh MAGIC/Rada.
- LampNet gateway wiring (cấp JWT, chọn node/erasure, ACL share/revoke).
- Khoá BBS issuer trong `TrustRecord`/`TrustListDocument` (neo Merkle root).

### 6.4 On-chain (đội on-chain) + VeData
- Neo `anchorDigest`/`ZkAnchor` vào metadata-6789/datum cạnh DID Document.
- **Anchor tài liệu:** Knowme phát **StampRecord** (tham chiếu Strata head) → Stamp EMIT `anchor_request` → Mosaic. **KHÔNG** gọi thẳng Mosaic (interface đó là bịa — Knowme-Feat-Math §7.3). Trạng thái anchor về qua Query `subscribe ANCHOR_CONFIRMED`.
- **Đọc:** verifier gọi Query `resolve(stamp_id)` (D9 privacy fail-closed + DP + audit-log). Keying `stamp_id`, KHÔNG `ref_id`.

---

## 7. Thứ tự deploy + phụ thuộc chặn

| Mốc | Nội dung | Phụ thuộc |
|---|---|---|
| **M1** | Mức 1+2 (khai/tiết lộ/thu hồi) + `/vc` | — |
| **M2** | Resolve point-in-time + TrustList/StatusList backend | đội backend |
| **M3** | Lớp tài liệu SDK (DocumentClaim + seal + re-seal) + UI | LampNet gateway (đội backend) |
| **M4** | Versioning Strata (nối Knowme↔Strata) | lib Strata ngoài |
| **M5** | Anchor tài liệu qua Stamp | archetype StampRecord (K-3) |
| **M6** | Mức 3 ZK (BBS lib + predicate + ZkAnchor + UI) | lib BBS + khoá BBS issuer |
| **M7** | Query read gateway (D9+DP+audit) | VeData Query |

**Phụ thuộc chặn quan trọng:** M5/M7 phụ thuộc hợp đồng liên-module — Contract PhoenixKey↔VeData v0.2.0 chỉ cấp resolve DID; anchor-qua-Stamp + Query-read là intake **công khai** của VeData (thoả hôm nay); nhưng Glint/Spectra **chủ động** cần VeData mở hợp đồng hoặc chạy on-device (Knowme-Feat-Math §12).

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## 8. Test / evidence

- **Mức 1+2:** bộ test trong `src/lib/sdvc/__tests__/` (`credential.test.ts`, `disclose.test.ts`, `scenarios.test.ts`, `statusList.test.ts`, `trust.test.ts`, `trustList.test.ts`, `didResolver.test.ts`, `commit.test.ts`, `canonical.test.ts`, `crypto.test.ts`, `fingerprint.test.ts`, `anchor.test.ts`, `hardening.test.ts`, `integration.test.ts`, `smoke.test.ts`, `acceptance.test.ts`, `adminUnit.test.ts`, `did.test.ts`, `lampnet.test.ts`, `schema.test.ts`). Demo chạy tại `/vc`. Chạy: `npx vitest run src/lib/sdvc/`.
- **Lớp tài liệu + Mức 3:** bộ test tương ứng nằm cạnh code khi triển khai từng phần **[SPEC]**.

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## 9. Ranh giới + giới hạn cố hữu

- **Lớp tài liệu:** `DocumentClaim`/`docDigestOf` là thiết kế trong Knowme-Feat-Math, giao Frontend/SDK triển khai (§6.1).
- **Mức 3 ZK:** phụ thuộc `zk/`, `BBS`, `ProveDerived`, `ZkCredential` — toàn bộ là spec KYC-KYB-ZK, giao đội on-chain/Frontend/VeData.
- **Query gateway:** phụ thuộc VeData Query (một phần phụ thuộc hợp đồng liên-module).
- **Giới hạn cố hữu (verify-time plaintext resale):** với ảnh mắt người xem được, thu hồi KHÔNG lấy lại bản đã xem — **không vá triệt để được** (Knowme-Feat-Math §9.0). Giảm thiểu: watermark truy vết + ưu tiên proof-thay ảnh. UI PHẢI nói thẳng.
- **KHÔNG dedup/uniqueness PersonDID** trong Knowme (chủ trương dự án; INV-K6). Catalog VC + issuer thuộc VeData.

→ Trạng thái & tiến độ: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Knowme-Specs/blob/main/PhoenixKey-STATUS.md)

---

## Nguồn

- Code: `PhoenixKey-Frontend/src/lib/sdvc/*` + `src/app/verify/page.tsx` + `src/components/vc/*`.
- Thiết kế nội bộ (không công khai): Knowme-Feat-Math (§2 kiến trúc, §3 schema, §4-6 luồng, §7 VeData, §12 hợp đồng); KYC-KYB-ZK-Feat-Math (§B Math, §C bàn giao); VeData-Reply-did-commit-resolve (kênh resolve chung).
- `PhoenixKey-Knowme-Math.md`, `PhoenixKey-Knowme-Vi-Feat.md`, `PhoenixKey-Knowme-Exec.md`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
