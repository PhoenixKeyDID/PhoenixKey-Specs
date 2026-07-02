# PhoenixKey — Phả hệ seed/khoá lưu trên LampNet Strata — Feat + Math [N]

> **PHÂN LOẠI: OFFCHAIN (PhoenixKey index+UI) → Long · Strata engine → LampNet
> (message). onchain strata_ref → Tuân nếu cần schema.**

> Đặc tả tính năng (WHAT/WHY) + hình thức hoá (Math) cho **phả hệ seed/khoá** của
> PhoenixKey. Hướng mới anh Aladin nêu 2026-07-02: PhoenixKey **KHÔNG chỉ giữ bộ
> seed đang dùng** mà giữ **MỌI bộ seed/khoá đã xoay, không xoá**. Mỗi seed/khoá
> sinh ra → tạo **một Strata** (Hồ sơ Tiến hóa của LampNet) mang: lịch sử truy cập,
> vị trí (bản đồ shard), mục đích, ghi chú. PhoenixKey **phối hợp Strata của
> LampNet** để hiển thị **phả hệ khoá/seed lên UI**.
>
> Quy trình: **Feat+Math này → anh Aladin duyệt → Long (backend index + UI wiring)
> + LampNet (Strata engine, message riêng) + Tuân (schema strata_ref nếu cần).**
>
> **Đọc kèm (đã đối chiếu source thật):**
> - `PhoenixKey-Secret-Vault-and-LampNet-Custody-Feat-Math.md` — ECIES client-side,
>   `VaultKEK` (HKDF info `"secret-vault-v1"`), Master_KEK KHÔNG rời máy,
>   `recovery_anchor`/`vault_index_anchor` neo on-chain, VaultIndex, fail-closed.
> - `PhoenixKey-UI-Spec-Backup-Recovery-Lifecycle.md` — mô hình rotate: Rotation
>   Account cố định, KHOÁ controller xoay được (§9); tài sản đi theo danh tính.
> - **Strata API thật** — `LampNetCloud/Strata/spec/Strata-API.md` +
>   `_CONTRACT.md`; code `lampnet-strata/src/{version,chain,state,refid,audit}.rs`
>   (`StrataChain::genesis`, `append_version`, `AuditLog::append_access`,
>   `prove_field`, `StrataAnchor` 104B; INV-E1..E9; CHỐT-4 content_cid thuần).
>
> **Nuance cốt tử:** PhoenixKey mã hoá **CLIENT-SIDE**. LampNet/Strata chỉ lưu
> **CIPHERTEXT + metadata không-bí-mật** (content_cid trỏ blob mã hoá, state_root
> trên field commitment). **Plaintext seed KHÔNG BAO GIỜ lên Strata.**

---

# PHẦN A — FEAT

## A.1 Bối cảnh — vì sao giữ TOÀN BỘ lineage, không xoá

Ngày nay app ví chỉ giữ **bộ seed đang hoạt động**; khi user xoay khoá (rotate),
seed cũ bị quên/xoá. Với PhoenixKey điều đó **sai về bản chất tài sản**:

1. **Địa chỉ cũ vẫn nhận tiền tương lai.** Một địa chỉ Cardano dẫn xuất từ seed cũ
   **không bao giờ "hết hạn"** — người khác có thể gửi ADA/token/UTxO tới nó bất
   cứ lúc nào (airdrop, người trả nợ dùng địa chỉ cũ, hợp đồng trả về địa chỉ đã
   khai báo). Nếu app quên seed cũ, user **mất khả năng quét + rút** số tiền đó.
   → Phải **giữ seed/khoá cũ để tiếp tục theo dõi + quét + chi** các địa chỉ đó.

2. **Seed cũ lộ ≠ mất quyền kiểm soát.** Ngay cả khi một seed cũ bị lộ, hacker chỉ
   trở thành **đồng-sở-hữu** các địa chỉ dẫn xuất từ seed đó — họ KHÔNG chiếm được
   danh tính DID (controller đã rotate sang khoá mới, on-chain, validator ép). Địa
   chỉ vẫn nhận tiền được; ai quét trước rút trước. → Giữ lineage để user **giám
   sát chủ động** các địa chỉ rủi ro (đánh dấu `compromised`, cảnh báo khi có tiền
   vào, quét-rút khẩn cấp).

3. **Kiểm toán + minh bạch vòng đời khoá.** Giữ toàn bộ phả hệ (khi nào sinh, dùng
   làm gì, đã xoay chưa, ai/lúc-nào truy cập) cho user một **bản ghi tamper-evident**
   về mọi khoá họ từng có — đúng nhu cầu một hệ danh tính chủ-quyền.

**Nguyên tắc cứng (I-LIN-1):** seed/khoá **KHÔNG bao giờ bị xoá khỏi lineage**.
Rotate = **đánh dấu `rotated`**, thêm entry mới; không ghi đè, không xoá. Đây là
mở rộng tự nhiên của mô hình rotate hiện có (UI-Spec §9): Rotation Account cố định,
khoá controller xoay được — nay ta **giữ luôn dấu vết mọi khoá đã xoay**.

## A.2 Vì sao Strata (không chỉ VaultIndex)

Kho bí mật (Custody spec) đã lưu **bản sao mã hoá** của seed để khôi phục. Nhưng
một bản sao tĩnh **không mang lịch sử tiến hoá**: khi nào truy cập, ai truy cập, vị
trí shard đổi thế nào, mục đích/ghi chú cập nhật ra sao. Đó chính là **loại dữ liệu
Strata** — "Hồ sơ Tiến hóa" (Evolving Content Record):

- **Nội dung tiến hoá theo phiên bản** (loại #4 Structured + #2 Append-only trong
  Strata `_CONTRACT.md §"4 loại dữ liệu"`): mỗi lần đổi trạng thái/ghi chú →
  `append_version`, lịch sử cũ bất biến (INV-E3).
- **Lịch sử truy cập tamper-evident** qua `AuditLog` (MMR append-only): mỗi lần mở
  khoá/quét địa chỉ → một `AuditEntry` (INV-E3, đơn điệu thời gian).
- **Field-proof riêng-tư** (INV-E6): chứng minh một trường (vd `status`) mà không
  lộ trường khác.

→ **Mỗi seed/khoá = một Strata** (một `ref_id` ổn định `lnref1…`). Lineage của một
DID = **tập các Strata**, hiển thị dạng timeline trên UI.

**Quan hệ với VaultIndex:** VaultIndex (Custody spec) vẫn là **nơi khôi phục
ciphertext seed** (24 từ → VaultIndex → blob). Strata **KHÔNG thay** VaultIndex; nó
là **lớp metadata tiến-hoá song song**, trỏ về cùng ciphertext qua `content_cid`.
`SeedLineageEntry.strata_ref` neo trong **TAAD vault index** cạnh `cid` ciphertext
(A.6). [Quyết định 4-trục ở C.1.]

## A.3 Ba đối tượng UI phải có

1. **Màn phả hệ khoá/seed** (S-LINEAGE): timeline mọi seed/khoá của DID — trạng
   thái, mục đích, ghi chú, lịch sử truy cập, bản đồ shard (A.7).
2. **Cảnh báo địa chỉ cũ nhận tiền** (monitor A.5.4): banner khi một địa chỉ thuộc
   entry `rotated`/`compromised` có UTxO mới → nút "Quét & rút".
3. **Sửa mục đích/ghi chú** một entry → `append_version` vào Strata tương ứng.

---

# PHẦN B — MÔ HÌNH DỮ LIỆU

## B.1 `SeedLineageEntry` — một mắt phả hệ

```
SeedLineageEntry ≜ {
  seed_id      : Hash32,          -- định danh nội bộ ổn định của seed/khoá (KHÔNG-bí-mật;
                                  --   = H(pubkey gốc ∥ salt_did); KHÔNG H(seed))
  derived_at   : SlotNo,          -- slot sinh seed/khoá
  addresses    : [Bech32Addr],    -- (các) địa chỉ dẫn xuất cần theo dõi/quét
  status        : active | rotated | compromised,
  purpose      : ByteArray,       -- mục đích người-đọc ("ví chính", "ví lạnh 2026")
  notes        : ByteArray,       -- ghi chú người-đọc
  strata_ref   : Bech32RefId,     -- "lnref1…" — Strata của entry này (A.6)
  cid_ct       : Cid              -- content_cid của CIPHERTEXT seed (từ Custody VaultIndex)
}

SeedLineage(did) ≜ [ SeedLineageEntry ]     -- FULL phả hệ, append-only, KHÔNG xoá (I-LIN-1)
```

- `seed_id`, `derived_at`, `addresses`, `strata_ref` là **không-bí-mật** → sống
  trong **TAAD vault index** (index dẫn xuất của PhoenixKey, off-chain) để UI render
  nhanh không cần giải mã. `purpose`/`notes` cũng lưu **bản mã hoá** trong Strata
  content (không lộ node LampNet); UI hiện chúng sau khi giải mã client-side.
- `status ∈ {active, rotated, compromised}` — bất biến chỉ **thêm chuyển tiếp**,
  không lùi (active→rotated→compromised; hoặc active→compromised).

## B.2 Ánh xạ `SeedLineageEntry → Strata` (một-một)

Mỗi entry → **một `StrataChain`** (một `ref_id`). Dùng đúng kiểu code thật của
Strata (`Strata-API.md §1`), KHÔNG bịa struct mới:

```
-- ref_id: sinh TẤT ĐỊNH từ (author_did ∥ genesis_nonce)  [refid.rs]
strata_ref = gen_ref_id(author_did = H(did), genesis_nonce = seed_id)   -- "lnref1…"

-- content_cid MỖI version: trỏ blob CIPHERTEXT metadata (đã ECIES client-side).
content_cid = gen_content_cid( ECIES_Encrypt( pk_vault, serialize(LineageMeta) ) )
   -- CHỐT-4: content_cid là hash THUẦN BLAKE3, KHÔNG class byte → không leak loại (INV-E5).

-- state_root MỖI version: field-level commit trên các trường KHÔNG-bí-mật để field-proof.
state_fields = [
   ("status",      status_byte),                -- 1 byte enum; proof status không lộ notes
   ("derived_at",  u64_be(derived_at)),
   ("seed_fp",     seed_id),                     -- fingerprint không-bí-mật
   ("meta_cid",    content_cid)                  -- trỏ ciphertext metadata (giá trị = content_cid thuần)
]
state_root = build_state_root(state_fields)      -- state.rs, INV-E6

-- author_did = H(did) [u8;32]; policy = { allowed: { H(did) → controller_pubkey } } (V1 mức-chain)
```

**`LineageMeta`** = phần **mã hoá** (KHÔNG lên state, chỉ nằm trong `content_cid`
blob) — mang dữ liệu nhạy-cảm-mềm mà user muốn riêng tư:

```
LineageMeta ≜ {
  purpose        : ByteArray,     -- mã hoá
  notes          : ByteArray,     -- mã hoá
  addresses      : [Bech32Addr],  -- mã hoá (địa chỉ = quan hệ, giữ riêng tư)
  access_history : [AccessRec],   -- lịch sử truy cập chi tiết (mã hoá; xem B.3)
  shard_map      : [ShardRef]     -- vị trí shard hiện tại (từ telemetry Mirage)
}
AccessRec ≜ { ts, actor : self|guardian|scan, action, note }
ShardRef  ≜ { node_id, cid_chunk }         -- bản đồ vị trí (UI-Spec S8)
```

## B.3 Hai đường ghi lịch sử truy cập (chọn theo tần suất)

Strata API cho **hai** cách (`Strata-API.md §2.6`) — PhoenixKey dùng CẢ HAI:

1. **Sự kiện giá-trị-cao / thưa** (rotate, đổi status, sửa purpose) →
   **`append_version`**: một version mới với `state_root` mới (INV-E1/E2). Lịch sử
   version = phả hệ trạng thái của khoá.

2. **Truy cập tần-suất-cao / audit** (mỗi lần mở khoá, mỗi lần quét địa chỉ) →
   **`AuditLog::append_access`**: một `AuditEntry` leaf MMR riêng, KHÔNG đẻ version
   mỗi lần (tránh phình chain). Đây là **lịch sử truy cập tamper-evident** hiển thị
   trên UI:

```
AuditEntry ≜ { created_ts, actor_did:[u8;32], action:AuditAction,
               signed_hash:Hash32, location:Hash32 }        -- audit.rs (kiểu thật)
AuditAction ∈ { Create, Read, Sign, ShareProof, Update }
```

> **Ánh xạ hành động PhoenixKey → `AuditAction`:** mở-để-xem = `Read`; ký tx bằng
> khoá này = `Sign`; quét-địa-chỉ-nhận-tiền = `Read` (actor = `scan`); rotate/đổi
> status = `Update`; xuất proof phả hệ = `ShareProof`. `location` = H(shard_map
> hiện tại) hoặc H(context truy cập) — KHÔNG-bí-mật.

## B.4 Bảng: trường nào ở đâu (chống rò rỉ)

| Trường | TAAD vault index (off-chain, plaintext) | Strata `state` (commit công khai) | Strata `content` (ciphertext) |
|---|---|---|---|
| `seed_id`/fingerprint | ✅ | ✅ (`seed_fp`) | — |
| `derived_at` | ✅ | ✅ | — |
| `status` | ✅ | ✅ (field-proof được) | — |
| `strata_ref` | ✅ | — (là chính ref_id) | — |
| `addresses` | ✅ (để quét nhanh)¹ | — | ✅ |
| `purpose`/`notes` | — | — | ✅ |
| `access_history` chi tiết | — | — (chỉ MMR audit-log root công khai) | ✅ |
| plaintext seed | 🔴 KHÔNG BAO GIỜ | 🔴 KHÔNG BAO GIỜ | 🔴 KHÔNG BAO GIỜ (chỉ ciphertext ở VaultIndex) |

¹ Địa chỉ trong TAAD vault index là index **cục bộ trên máy** của user (để quét
UTxO nhanh); nếu chính sách riêng-tư yêu cầu, có thể chỉ giữ ở `content` mã hoá và
quét sau khi giải mã. [OPEN-L4]

---

# PHẦN C — LUỒNG

## C.1 Sinh seed/khoá → tạo entry + Strata

```
(1) User sinh seed/khoá mới (genesis DID, import ví, hoặc dẫn xuất khoá mới).
(2) Client tính seed_id = H(pubkey_gốc ∥ salt_did)  (KHÔNG-bí-mật, I-LIN-6).
(3) Client mã hoá metadata + (nếu là seed cất kho) ciphertext seed:
       blob_meta = ECIES_Encrypt( pk_vault, serialize(LineageMeta) )   [client-side]
       content_cid = LampNet.put(blob_meta)         -- Mirage lưu ciphertext
(4) Client dựng version genesis Strata (ký bằng controller key):
       v0 = StrataVersion::unsigned(seq=0, prev=0^32, content_cid,
                 build_state_root(state_fields), author_did=H(did), policy_hash, ts)
       v0.sign(controller_sk)                        -- Ed25519 low-S over version_hash
       POST /v1/strata/create → { ref_id = "lnref1…" }     [LampNet daemon]
(5) Client lưu SeedLineageEntry{ seed_id, derived_at, addresses, status=active,
       purpose, notes, strata_ref=ref_id, cid_ct } vào TAAD vault index.
(6) strata_ref được neo cạnh cid_ct trong vault_index_anchor (A.6 / Custody spec).
```

**Quyết định (4 trục) — vì sao Strata bọc ngoài, ciphertext ở Mirage:** (a) *dài
hạn* — một mô hình Strata thống nhất cho mọi đối tượng tiến-hoá của hệ sinh thái
(ProofChat/OriLife/PhoenixKey), tái dùng engine LampNet; (b) *first-principles* —
plaintext seed chỉ tồn tại trong máy; Strata chỉ giữ **commitment + ciphertext**;
(c) *tối ưu* — proof `O(log n)`, anchor 104B, không nhồi datum; (d) *user* — phả hệ
kiểm-toán-được, khôi phục tự-chủ từ 24 từ. **Chốt: strata_ref neo trong TAAD vault
index (off-chain-anchored), KHÔNG thêm field on-chain riêng mỗi seed** (datum nhỏ).

## C.2 Rotate → cũ `rotated` (KHÔNG xoá), thêm entry mới

```
(1) User rotate controller (UI-Spec §9: build rotate tx → submit on-chain).
(2) Entry CŨ: append_version vào Strata cũ, đổi state field "status" → rotated.
       KHÔNG xoá entry, KHÔNG xoá Strata, KHÔNG xoá ciphertext (I-LIN-1).
       AuditLog: append_access(action=Update, actor=self).
(3) Entry MỚI cho khoá mới: chạy nguyên C.1 (seed_id mới → Strata mới, status=active).
(4) TAAD vault index: cả hai entry cùng tồn tại; UI timeline hiện cả hai, khoá cũ
       gắn nhãn "đã xoay" + vẫn còn nút "quét địa chỉ" (C.4).
```

Địa chỉ của khoá cũ **vẫn được theo dõi** — đây chính là lý do không xoá (A.1).

## C.3 Mỗi truy cập → append vào access-log

```
Bất kỳ thao tác chạm seed/khoá (mở xem, ký tx, quét địa chỉ, xuất proof) ⟹
   POST /v1/strata/:strata_ref/event  { kind:"audit", actor_did, action, ts, sig, ... }
   → AuditLog.append_access(AuditEntry{...})   -- MMR append-only, ts đơn điệu (INV-E3)
UI đọc access-log qua field/inclusion-proof để hiện timeline "ai đọc gì khi nào".
```

Tamper-evident: sửa/xoá một entry cũ → `mmr_root` đổi → lệch với anchor đã neo →
phát hiện được (INV-E3/E7).

## C.4 Monitor — quét địa chỉ cũ vẫn nhận tiền

```
Định kỳ (hoặc khi user mở màn phả hệ):
  ∀ entry ∈ SeedLineage(did) với status ∈ {active, rotated, compromised}:
     ∀ addr ∈ entry.addresses:
        utxos = QueryChain(addr)                 -- Blockfrost/node (Long backend)
        nếu utxos ≠ ∅:
           UI cảnh báo "Địa chỉ [nhãn] có tiền mới" + nút "Quét & rút"
           AuditLog.append_access(action=Read, actor=scan, location=H(addr))
```

- Với `rotated`/`compromised`: user vẫn **chi được** (còn giữ khoá cũ trong kho) —
  đúng A.1. Với `compromised`: UI khuyến nghị **quét-rút NGAY** (đua với hacker
  đồng-sở-hữu). Đây là giá trị cốt lõi của việc giữ lineage.
- Quét là **client/backend PhoenixKey** (Long), KHÔNG thuộc Strata.

---

# PHẦN D — UI (màn phả hệ khoá/seed, S-LINEAGE)

Timeline dọc, mỗi node = một `SeedLineageEntry`, sắp theo `derived_at`:

```
┌─ Phả hệ khoá của bạn ───────────────────────────────┐
│ ● Khoá hiện tại        [active]      sinh 2026-06-01 │
│   Mục đích: Danh tính chính                          │
│   Truy cập gần nhất: hôm nay 09:12 (ký giao dịch)    │
│   Shard: 12 node ✓        [Xem lịch sử] [Ghi chú]    │
│ │                                                     │
│ ○ Khoá #2              [rotated]     sinh 2026-03-10 │
│   Mục đích: Khoá cũ trước khi xoay                   │
│   ⚠ Địa chỉ này vừa nhận 40 ADA   [Quét & rút]       │
│   Shard: 12 node ✓        [Xem lịch sử] [Ghi chú]    │
│ │                                                     │
│ ○ Khoá #1              [compromised] sinh 2026-01-02 │
│   🔴 Đã đánh dấu lộ — theo dõi để rút khẩn cấp        │
│   [Xem lịch sử truy cập] [Ghi chú]                   │
└─────────────────────────────────────────────────────┘
```

Thành phần mỗi node (đọc từ Strata sau khi giải mã client-side):
- **Trạng thái** — `active`/`rotated`/`compromised` (từ state field `status`;
  field-proof được, không lộ ghi chú).
- **Mục đích + ghi chú** — từ `LineageMeta` (ciphertext → giải mã trên máy).
- **Lịch sử truy cập** — bung `AuditLog`: danh sách `{ts, actor, action}`
  (tamper-evident, kèm nút "kiểm chứng" = verify inclusion-proof về anchor).
- **Vị trí shard** — `shard_map` (telemetry Mirage, UI-Spec S8): "N node ✓".
- **Cảnh báo nhận tiền** — banner + "Quét & rút" (C.4).

Chuỗi hiển thị qua khoá i18n (UI-Spec §12): "Phả hệ khoá", "Đã xoay", "Đã lộ",
"Quét & rút", "Lịch sử truy cập", "Vị trí lưu".

---

# PHẦN E — INTERFACE CẦN TỪ LAMPNET STRATA  🔗 **DEPENDENCY — cần chốt với LampNet**

> **Đây là phần DEPENDENCY sang LampNet.** PhoenixKey KHÔNG build Strata engine.
> Danh sách dưới là **chính xác cái PhoenixKey cần** — để anh soạn message cho
> LampNet. Mọi signature/route đối chiếu `Strata-API.md` (code thật), đánh dấu rõ
> cái đã-có-code (✅) vs cái là [SPEC-TODO] bên LampNet.

### E.1 API create Strata (một seed → một Strata)
```
POST /v1/strata/create        ✅ có code (StrataChain::genesis)
req : { author_did:H(did), genesis_nonce=seed_id, content_cid=<ciphertext_meta CID>,
        state_fields:[{status},{derived_at},{seed_fp},{meta_cid}],
        policy_hash, ts, sig }
res : { ref_id:"lnref1…", head_seq:0, head_version_hash, mmr_root }
```
**PhoenixKey cần LampNet xác nhận:** `genesis_nonce` nhận `seed_id` 32B tuỳ ý;
`policy` V1 mức-chain với đúng một author = controller key của DID.

### E.2 API append-version (đổi status / mục đích / ghi chú)
```
POST /v1/strata/:ref/version  ✅ có code (chain.append_version)
req : { prev_seq, content_cid=<new ciphertext_meta CID>, state_fields:[...new...],
        author_did, policy_hash, ts, sig }
res : { seq, version_hash, mmr_root, prev_hash }
```
**Cần:** append giữ **lịch sử cũ bất biến** (INV-E3) — đúng "không xoá" của A.1.

### E.3 API read-with-access-log (đọc head + lịch sử truy cập)
```
GET  /v1/strata/:ref/head                 ✅  → { head_seq, head_version_hash, mmr_root, content_cid }
POST /v1/strata/:ref/event {kind:"audit"} ✅  → { index, log_root }   (append AuditEntry)
GET  /v1/strata/:ref/version?at=<ts>      ✅  → version + inclusion-proof tại thời điểm t
```
**PhoenixKey cần LampNet cung cấp thêm (❓ hỏi):** một endpoint **liệt kê
access-log** cho UI — hiện `Strata-API.md §2.6` chỉ có `append_access` +
`log.prove(idx)` ở lớp Rust; **chưa thấy HTTP `GET /v1/strata/:ref/audit?from=..`
để BUNG danh sách `AuditEntry`**. UI phả hệ cần list này. → **[YÊU CẦU-1 cho
LampNet]:** thêm `GET /v1/strata/:ref/audit` trả `[{created_ts, actor_did, action,
location, index}]` + `log_root` để client verify.

### E.4 API field-proof (chứng minh `status` không lộ ghi chú)
```
GET  /v1/strata/:ref/proof/field/status   ✅  (state::prove_field)
res : { key:"status", value, fvh, siblings, state_root, version_seq }
```
**Cần:** field-proof `status` KHÔNG lộ `notes`/`purpose` (INV-E6) — dùng khi user
xuất bằng chứng "khoá này đã rotated" cho bên thứ ba mà giữ ghi chú riêng tư.

### E.5 Mapping seed → Strata (PhoenixKey giữ, LampNet cần biết quy ước)
```
strata_ref = gen_ref_id( H(did), seed_id )      -- ổn định, một-một với seed_id
```
**Cần LampNet xác nhận:** `gen_ref_id` tất định từ `(author_did, genesis_nonce)`
→ PhoenixKey tự tính lại `ref_id` khi khôi phục (không cần LampNet lưu mapping).

### E.6 Access-log schema — PhoenixKey đề xuất, cần LampNet chốt
```
AuditEntry { created_ts, actor_did:[u8;32], action:AuditAction, signed_hash, location }
AuditAction cần đủ: Read (mở/quét), Sign (ký tx), Update (rotate/đổi status),
                    ShareProof (xuất proof), Create (genesis).
```
**[YÊU CẦU-2 cho LampNet]:** xác nhận 5 biến thể `AuditAction` phủ đủ nhu cầu
PhoenixKey (đối chiếu audit.rs: đúng 5 biến thể — khớp ✅, chỉ cần chốt ánh xạ).

### E.7 Anchor (neo commitment lineage on-chain)
```
POST /v1/strata/:ref/anchor  → StrataAnchor{ ref_id, head_version_hash, mmr_root, seq } 104B
```
**Ranh giới:** adapter neo (`AnchorSink`, Settlement vs Mosaic) là **[SPEC-TODO]
bên LampNet + quyết định liên-nền-tảng của anh** (Strata-API.md §4). PhoenixKey chỉ
cần biết: neo là **tuỳ chọn theo tầng giá trị**; lineage giá-trị-cao → `immediate`,
thường ngày → `batch_daily`. **KHÔNG chặn Phase 1** (Strata sống tầng (a)/(b) không
neo vẫn dùng được cho UI).

### E.8 Tóm tắt yêu cầu gửi LampNet (checklist message)
| # | PhoenixKey cần | Trạng thái Strata | Hành động |
|---|---|---|---|
| 1 | `POST /create`, `/version`, `/event`, `/head`, `/version?at=`, `/proof/field` | ✅ có code | Wire HTTP (LampNet daemon [SPEC-TODO] §3) |
| 2 | `GET /v1/strata/:ref/audit` (bung access-log cho UI) | ❌ chưa có HTTP | **[YÊU CẦU-1]** thêm route |
| 3 | Xác nhận `gen_ref_id` tất định (seed→ref) | ✅ | Xác nhận |
| 4 | Chốt ánh xạ 5 `AuditAction` | ✅ | Xác nhận **[YÊU CẦU-2]** |
| 5 | Mirage mode `EncryptedDistributed` (mã hoá ∧ repair) | [SPEC-TODO] | Cần cho ciphertext bền (INV-E9) |
| 6 | Adapter neo (backend mặc định) | [SPEC-TODO] | Quyết định liên-nền-tảng (anh) — KHÔNG chặn P1 |

---

# PHẦN F — BẢO MẬT

- **I-LIN-2 (plaintext seed KHÔNG lên Strata):** Strata chỉ nhận `content_cid` (trỏ
  **ciphertext** đã ECIES client-side) + `state_root` (commit trên trường KHÔNG-bí-mật).
  Plaintext seed KHÔNG BAO GIỜ rời máy; ciphertext seed sống ở VaultIndex/Mirage,
  Strata chỉ trỏ tới. Node LampNet thấy **ciphertext + hash**, không thấy seed/ghi chú.
- **I-LIN-3 (khoá giải mã không rời máy):** `pk_vault` (public) an toàn để rời máy;
  `sk_vault` phái sinh tại-chỗ từ `VaultKEK = HKDF(Master_KEK, "secret-vault-v1", …)`
  (Custody B.2), KHÔNG lưu/gửi. 24 từ DID là gốc giải mã duy nhất.
- **I-LIN-4 (lineage lộ ra ngoài chỉ là metadata mã hoá):** thứ node/observer thấy
  là `ref_id` opaque (INV-E5, không lộ loại), `content_cid` hash thuần, `state`
  commitment (status enum + slot + fingerprint — không-bí-mật), `mmr_root`. Địa
  chỉ/mục đích/ghi chú/lịch-sử-chi-tiết đều **ciphertext**.
- **I-LIN-5 (tamper-evident):** append-only MMR (INV-E3) + anchor đơn điệu (INV-E7)
  → không thể xoá/sửa lén một mắt phả hệ hay một dòng truy cập. Đúng "không xoá".
- **I-LIN-6 (fingerprint không-bí-mật):** `seed_id`/`seed_fp` = H(pubkey ∥ salt),
  KHÔNG là hàm của seed bí mật → không tạo oracle xác nhận seed đoán-trước
  (kế thừa I-VAULT-11 Custody).
- **I-LIN-7 (status chỉ tiến, không lùi):** `active → rotated → compromised` (hoặc
  active→compromised); append_version từ chối chuyển lùi (kiểm ở client + có thể ép
  bằng field-policy V2 nếu LampNet bật).

---

# PHẦN G — RANH GIỚI (chia theo chủ sở hữu)

| Phần | Chủ | Nội dung |
|---|---|---|
| **OFFCHAIN — index + UI** | **Long** (backend) + Core/SuperApp (UI) | TAAD vault index thêm `SeedLineageEntry[]`; wiring gọi Strata HTTP (create/append/audit/field-proof); monitor quét địa chỉ cũ (C.4); màn S-LINEAGE (Phần D); i18n. Custody spec đã có ECIES/VaultKEK/upload. |
| **Strata engine** | **LampNet** (message riêng, Phần E) | Toàn bộ Strata core + daemon HTTP; **[YÊU CẦU-1]** endpoint `/audit`; Mirage `EncryptedDistributed`; adapter neo. PhoenixKey KHÔNG build. |
| **Onchain — strata_ref** | **Tuân** (nếu cần schema) | Mặc định strata_ref neo GIÁN TIẾP qua `vault_index_anchor` (đã đề xuất ở Custody, một CID). Chỉ cần Tuân **nếu** chốt thêm field schema on-chain cho lineage (không khuyến nghị — datum nhỏ). |

**Ranh giới cứng:** PhoenixKey backend (Specs/Validator/Database) = Long/Tuân, Claude
KHÔNG sửa (memory ranh giới). Strata engine = LampNet. Claude viết spec + wiring
đề xuất; code thật do các chủ tương ứng.

---

# PHẦN H — TEST PLAN

**H.1 Core (client, rust_core) — round-trip mã hoá/phả hệ:**
- Sinh seed → `seed_id` tất định, KHÔNG = H(seed) (I-LIN-6). Evidence: hai seed
  khác → seed_id khác; đổi seed nội dung nhưng cùng pubkey → seed_id giống.
- `ECIES_Encrypt(pk_vault, LineageMeta)` → put → get → `ECIES_Decrypt(sk_vault)` =
  LineageMeta gốc; sai VaultKEK → GCM tag fail (fail-closed, I-LIN-2).
- `content_cid` là hash thuần, KHÔNG lộ loại (INV-E5): hai entry khác loại →
  content_cid không suy ra loại.

**H.2 Strata integration (gọi API thật LampNet daemon):**
- create → append_version(status=active→rotated) → head phản ánh rotated; version
  cũ vẫn đọc được qua `?at=<ts cũ>` (INV-E3, "không xoá").
- `append_access` 100 lần → `GET /audit` trả đủ 100 entry theo thứ tự ts đơn điệu;
  sửa một entry → `mmr_root` lệch, verify fail (I-LIN-5).
- field-proof `status` verify OK mà KHÔNG lộ `meta_cid` khác (INV-E6): kiểm siblings
  chỉ là hash.
- `gen_ref_id(H(did), seed_id)` gọi hai lần → cùng `ref_id` (E.5, khôi phục tự-tính).

**H.3 Rotate (end-to-end):**
- rotate controller → entry cũ status=rotated (append_version, KHÔNG xoá Strata) +
  entry mới status=active. TAAD vault index có ĐỦ hai entry. UI timeline hiện cả hai.

**H.4 Monitor địa chỉ cũ:**
- Nạp UTxO giả vào một địa chỉ của entry `rotated` (preprod) → C.4 phát cảnh báo +
  "Quét & rút" build được tx chi từ khoá cũ. Evidence: tx submit thành công preprod.
- entry `compromised` → UI hiện khuyến nghị quét-rút khẩn cấp.

**H.5 Khôi phục (máy mới, chỉ 24 từ):**
- 24 từ → Master_KEK → VaultKEK → sk_vault. Với mỗi entry: tự-tính `ref_id` từ
  seed_id → `GET /head` → giải mã LineageMeta → dựng lại timeline. Evidence: phả hệ
  đầy đủ hiện lại KHÔNG cần máy cũ, KHÔNG cần backend PhoenixKey uptime (chỉ cần
  LampNet đọc + đọc chain). Sai 24 từ → fail-closed.

**H.6 Bảo mật (negative):**
- Node LampNet (giả lập) chỉ thấy ciphertext + commitment: assert KHÔNG có đường
  nào trả plaintext seed/purpose/notes/addresses (I-LIN-2/4).
- status lùi (rotated→active) bị từ chối (I-LIN-7).

---

# PHẦN I — CÂU HỎI MỞ (cần anh / Long / LampNet quyết)

- **[OPEN-L1]** strata_ref neo GIÁN TIẾP qua `vault_index_anchor` (một CID trỏ index
  chứa cả entry) — chốt KHÔNG thêm field on-chain riêng mỗi seed? Đề xuất: có (datum nhỏ).
- **[OPEN-L2]** **[YÊU CẦU-1]** LampNet thêm `GET /v1/strata/:ref/audit` bung
  access-log cho UI (hiện chỉ có `prove(idx)` lớp Rust). Chặn màn "lịch sử truy cập".
- **[OPEN-L3]** Backend neo mặc định (Settlement metadata vs Mosaic UTxO) cho lineage
  giá-trị-cao — liên-nền-tảng, anh + GreenSun (Strata-API §4). KHÔNG chặn Phase 1.
- **[OPEN-L4]** `addresses` giữ ở TAAD vault index (quét nhanh) hay chỉ ở content
  mã hoá (riêng tư hơn, quét sau giải mã)? Cân tốc-độ vs riêng-tư.
- **[OPEN-L5]** Cadence quét monitor (C.4): định kỳ nền vs chỉ khi mở màn — chi phí
  query chain (Blockfrost) vs kịp thời cảnh báo địa chỉ compromised.
- **[OPEN-L6]** Mirage `EncryptedDistributed` (mã hoá ∧ repair) chưa có (Strata-API
  §7.3): tới khi có, ciphertext lineage CHƯA đạt INV-E9 đầy đủ (mã hoá có, repair
  chưa) — việc của LampNet/Mirage.

---

*Hết bản đề xuất. Mọi mục [N] là đề xuất, chỉ thành normative sau khi anh Aladin /
Long / LampNet duyệt và hợp thức hoá. Signature/route Strata trích từ code thật
(`Strata-API.md` + `_CONTRACT.md`); phần [YÊU CẦU-*] là dependency cần LampNet bổ sung.*
