PHÂN LOẠI: OFFCHAIN → PR giao Long.

# PhoenixKey — Point-in-Time DID Resolve API (Feat + Math)

> Doc-type: Dev-ready spec | Status: Ready-to-implement | Date: 2026-07-02
> Owner triển khai: **Long** (PhoenixKey-Database, backend indexer + API)
> Nguồn hợp đồng: `VeDataIO/Specs/PhoenixKey-VeData-Contract.md` v0.2.0 §2.1 / §2.2 / §2.3 / §2.4 / §2.5 / §2.6
> Cross-ref nội bộ: `spec-proposals/VeData-Reply-did-commit-resolve.md` (chốt boundary), `PhoenixKey-Database` V4 (`onchain_taad_state_cache`), `IndexerController#syncTaad`
> Ranh giới: **OFFCHAIN backend**. On-chain TAAD ĐÃ phát event, **không sửa validator**.

---

## 0. Tóm tắt cho dev (đọc cái này trước)

Đây là nút thắt #1 — VeData (Stamp/Score/Query), MAGIC (`did_commit` bind), Rada (revocation), LampNet (gate) đều chờ.

Việc phải làm, ngắn gọn:

1. **Thêm 1 kho lịch sử append-only per-epoch** cho trạng thái DID (controller key, active/revoked, lineage, ownership edge). Kho hiện tại `onchain_taad_state_cache` là **overwrite 1-dòng-mỗi-DID** — KHÔNG dùng để trả point-in-time. Ta thêm bảng mới, KHÔNG đụng bảng cũ.
2. **Mở rộng `POST /internal/sync-taad`**: giữ nguyên hành vi cập-nhật-cache-hiện-tại, THÊM bước *append* một dòng lịch sử bất biến mỗi khi có thay đổi state. Không bao giờ ghi đè dòng quá khứ.
3. **Thêm nhóm endpoint `/identity/{did}/...`** point-in-time (đã có sẵn tiền tố này: `pubkey`, `status`, `document`).
4. **Thêm pubsub** 4 topic với at-least-once + ACK 30s; `did_revoked` cần 2-of-3 DAO multisig.
5. **Fail-closed**: dữ liệu thiếu / lag quá ngưỡng → `503`; consumer PHẢI từ chối, không đi tiếp.

Nguyên tắc bất biến cốt lõi (đọc kỹ §6): **không bao giờ mutate dòng lịch sử quá khứ**. Mọi thay đổi = append dòng mới với `epoch` mới. Query point-in-time = "dòng có `effective_epoch` lớn nhất mà `≤ query_epoch`".

---

## 1. Bối cảnh + ai tiêu thụ

### 1.1 Tại sao point-in-time

DID có thể xoay khóa (key rotation) và bị thu hồi (revocation) theo thời gian. Consumer cần verify một chữ ký / một record **tại đúng epoch nó được tạo**, không phải trạng thái hiện tại. Nếu chỉ có state hiện tại:

- Stamp verify sai: chấp nhận record ký bởi khóa CŨ (đã rotate) hoặc từ chối record hợp lệ trong quá khứ.
- Score whitewashing defense vô hiệu: DID revoke rồi tạo DID mới → nếu không tra được lineage lịch sử, kẻ tấn công rửa danh tiếng.

Vì vậy **mọi query MUST đánh giá theo state tại `query_epoch`** (Contract §2.4).

### 1.2 Ai tiêu thụ (subscriber matrix)

| Consumer | Dùng gì | Endpoint / topic |
|---|---|---|
| **Stamp** (VeData §10) | verify key rotation + revocation tại epoch record | `key-authorized`, `revocation`, `active`; topic `did_revoked`, `device_did_bound_to_person` |
| **Score** (VeData §D.6) | whitewashing defense — lineage + cache invalidation | `lineage`; topic `did_created`, `did_lineage_updated`, `did_revoked`, `device_did_bound_to_person` |
| **Query** (VeData D9) | access predicate runtime check | `revocation`; topic `did_revoked` |
| **MAGIC** (`did_commit` bind) | burn/bind bằng `key_authorized(key, did, epoch)` — CÙNG call Rada dùng | `key-authorized` |
| **Rada** (revocation) | device attestation lookup + revocation | `revocation`; topic `device_did_bound_to_person` |
| **LampNet** (gate) | gate quyền vào mạng theo DID active + revocation | `active`, `revocation`; topic `did_revoked` |

MAGIC `did_commit` và Rada revocation dùng **chung một call** `key_authorized(key, did, epoch)` (chốt tại `VeData-Reply-did-commit-resolve.md §1`). Không tách hai đường.

---

## 2. Data model — kho lịch sử bất biến per-epoch

### 2.1 Ý tưởng

Với mỗi loại state (controller key, revocation, lineage, ownership edge) ta lưu **dòng có hiệu lực từ một epoch** (`effective_epoch`) và không bao giờ sửa. Trả point-in-time = chọn dòng mới nhất `effective_epoch ≤ query_epoch`.

`epoch` ở đây = **Cardano epoch** (không phải block). Indexer đã có `last_synced_block` + `sequence`; ta THÊM `cardano_epoch` vào payload sync (§3.4).

### 2.2 Bảng (PostgreSQL, migration V16)

Tất cả cột snake_case. Type/field name tiếng Anh. Bảng cũ `onchain_taad_state_cache` (V4) GIỮ NGUYÊN — nó vẫn phục vụ "state hiện tại" cho App. Bảng mới đây phục vụ "lịch sử point-in-time".

#### 2.2.1 `did_state_history` — trạng thái controller + active/revoked theo epoch

```sql
CREATE TABLE did_state_history (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    did               VARCHAR(128) NOT NULL,       -- DID chủ thể (subject)
    effective_epoch   BIGINT       NOT NULL,       -- Cardano epoch dòng này bắt đầu có hiệu lực
    controller_pkh    VARCHAR(64)  NOT NULL,       -- Public Key Hash kiểm soát tại epoch này
    controller_pubkey_hex VARCHAR(128),            -- pubkey đầy đủ (nếu có), phục vụ key_authorized
    status            taad_status  NOT NULL,        -- ACTIVE | RECOVERING | MIGRATED (dùng lại enum V4)
    is_revoked        BOOLEAN      NOT NULL DEFAULT FALSE,
    revoked_epoch     BIGINT,                      -- epoch bị revoke (NULL nếu chưa)
    revocation_reason VARCHAR(256),                -- NULL nếu chưa revoke
    person_did        VARCHAR(128),                -- PersonDID gốc (cho lineage); NULL nếu chính nó là PersonDID
    -- Truy vết on-chain (audit, chống reorg)
    cardano_slot      BIGINT       NOT NULL,
    last_synced_block BIGINT       NOT NULL,
    block_hash        VARCHAR(64)  NOT NULL,
    event_nonce       BIGINT       NOT NULL,       -- unique per slot (anti-replay, §5.5 contract)
    sequence          BIGINT       NOT NULL,       -- sequence on-chain
    created_at        TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    -- APPEND-ONLY: 1 DID chỉ có 1 dòng cho mỗi effective_epoch
    CONSTRAINT uq_did_epoch UNIQUE (did, effective_epoch)
);

CREATE INDEX idx_dsh_did_epoch   ON did_state_history (did, effective_epoch DESC);
CREATE INDEX idx_dsh_person_did  ON did_state_history (person_did);
CREATE INDEX idx_dsh_slot_nonce  ON did_state_history (cardano_slot, event_nonce);
```

> Dùng `GENERATED ALWAYS AS IDENTITY` (chuẩn dự án), không dùng `SERIAL`.

Chống mutate: **cấp app-user quyền `INSERT` + `SELECT`, KHÔNG cấp `UPDATE`/`DELETE`** trên bảng này (trừ job GC có role riêng, §2.4). Đây là hàng rào ở tầng DB, không chỉ ở code.

#### 2.2.2 `did_lineage` — DID cùng PersonDID (whitewashing defense)

```sql
CREATE TABLE did_lineage (
    id                 BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    person_did         VARCHAR(128) NOT NULL,      -- gốc chung
    member_did         VARCHAR(128) NOT NULL,      -- 1 DID lịch sử thuộc person_did
    registration_epoch BIGINT       NOT NULL,
    revocation_epoch   BIGINT,                     -- NULL nếu còn active
    revocation_reason  VARCHAR(256),
    lineage_hash       VARCHAR(64)  NOT NULL,      -- H(sorted member_did list) tại thời điểm append
    created_at         TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_person_member UNIQUE (person_did, member_did)
);

CREATE INDEX idx_lineage_person ON did_lineage (person_did);
CREATE INDEX idx_lineage_member ON did_lineage (member_did);
```

`lineage_hash` = `blake2b_256` của danh sách `member_did` đã sort, snapshot mỗi lần append. Emit vào topic `did_lineage_updated` (`old_lineage_hash`, `new_lineage_hash`).

#### 2.2.3 `did_ownership_edge` — cạnh sở hữu per-epoch (ownership graph $G_{phx}$)

```sql
CREATE TABLE did_ownership_edge (
    id               BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    parent_did       VARCHAR(128) NOT NULL,        -- OrgDID / PersonDID cha
    child_did        VARCHAR(128) NOT NULL,
    edge_type        VARCHAR(32)  NOT NULL,        -- OWNS | DELEGATES | DEVICE_OF | MEMBER_OF
    effective_epoch  BIGINT       NOT NULL,        -- epoch cạnh bắt đầu có hiệu lực
    removed_epoch    BIGINT,                       -- NULL nếu chưa gỡ (append dòng mới khi gỡ)
    cardano_slot     BIGINT       NOT NULL,
    block_hash       VARCHAR(64)  NOT NULL,
    created_at       TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_edge_epoch UNIQUE (parent_did, child_did, edge_type, effective_epoch)
);

CREATE INDEX idx_edge_parent ON did_ownership_edge (parent_did, effective_epoch DESC);
CREATE INDEX idx_edge_child  ON did_ownership_edge (child_did,  effective_epoch DESC);
```

Cạnh cũng append-only: gỡ cạnh = append dòng mới có `removed_epoch` set, KHÔNG update dòng cũ. Snapshot tại epoch = cạnh có `effective_epoch ≤ epoch` AND (`removed_epoch` IS NULL OR `removed_epoch > epoch`).

#### 2.2.4 `pubsub_event_log` + `pubsub_subscription` — outbox + ACK

```sql
CREATE TABLE pubsub_event_log (
    id             BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    topic          VARCHAR(48)  NOT NULL,          -- did_created|did_lineage_updated|did_revoked|device_did_bound_to_person
    payload_json   JSONB        NOT NULL,
    cardano_slot   BIGINT       NOT NULL,
    event_nonce    BIGINT       NOT NULL,          -- unique per slot
    emitter_sig    VARCHAR(128) NOT NULL,          -- Ed25519 sign(emitter_key, H(topic ‖ payload ‖ slot))
    dao_multisig   JSONB,                          -- 2-of-3 sigs; NOT NULL bắt buộc cho did_revoked
    created_at     TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_slot_nonce UNIQUE (cardano_slot, event_nonce)
);
CREATE INDEX idx_evt_topic_slot ON pubsub_event_log (topic, cardano_slot);

CREATE TABLE pubsub_subscription (
    subscriber_id   VARCHAR(64)  NOT NULL,         -- score|stamp|query|rada|magic|lampnet
    topic           VARCHAR(48)  NOT NULL,
    last_acked_slot BIGINT       NOT NULL DEFAULT 0,
    last_acked_nonce BIGINT      NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (subscriber_id, topic)
);
```

### 2.3 Cách populate từ TAAD sync

**KHÔNG tạo indexer mới.** Mở rộng đường đã có `IndexerController#syncTaad → IndexerServiceImpl.syncTaad`. Sau khi cập nhật `onchain_taad_state_cache` như hiện tại (optimistic lock + reorg), THÊM:

```
// pseudo — append vào did_state_history
INSERT INTO did_state_history (did, effective_epoch, controller_pkh, ...,
                               cardano_slot, last_synced_block, block_hash, event_nonce, sequence)
VALUES (:did, :cardano_epoch, :controller_pkh, ...)
ON CONFLICT (did, effective_epoch) DO NOTHING;   -- idempotent: sync lại cùng epoch không tạo trùng
```

Quy tắc append:
- **Chỉ append khi state THỰC SỰ đổi** so với dòng mới nhất (controller_pkh / status / is_revoked / lineage khác). Không đổi → skip (tránh phình bảng).
- **Reorg**: dòng lịch sử ứng với `block_hash` cũ bị coi vô hiệu bằng cách append dòng mới ở cùng epoch KHÔNG được (vướng `uq_did_epoch`). → Xử lý: nếu reorg tại epoch `e`, GC-role xóa các dòng history có `cardano_slot ≥ reorg_slot` rồi re-append từ chain (đây là ngoại lệ DELETE duy nhất, chỉ role indexer-gc, log audit). Chi tiết §6 PIT-1 note reorg.
- **`did_revoked`**: append dòng có `is_revoked=true`, `revoked_epoch`, `revocation_reason`; đồng thời update `did_lineage.revocation_epoch` cho `member_did` (append-semantics: cập nhật cột revocation trên dòng lineage tương ứng — đây KHÔNG phải mutate history vì lineage là bảng quan hệ person↔member, không phải chuỗi thời gian; revocation_epoch chỉ set 1 lần, immutable sau đó — enforce bằng trigger `revocation_epoch` chỉ chuyển NULL→value).

### 2.4 Retention + GC

- `did_state_history` / `did_lineage`: giữ ≥ `T_inherit = 365` epoch sau revocation (Contract §2.2/§3). GC role riêng, có audit log.
- `did_ownership_edge` snapshot: giữ ≥ `730` epoch (Contract §2.3/§3).
- GC do 1 role DB riêng (`phx_gc`), KHÔNG phải app-user. App-user không có `DELETE`.

---

## 3. API endpoints (offchain, mở rộng PhoenixKey-Database)

Tất cả dưới controller mới `PointInTimeController` `@RequestMapping("/identity")` (cùng tiền tố `IdentityController` hiện có). Response bọc trong `DataResponse<T>` chuẩn dự án (`code`, `message`, `data`). Field trong `data` = **snake_case** (đã cấu hình Jackson dự án). Tham số `epoch` = Cardano epoch; nếu bỏ trống ⇒ epoch hiện tại (nhưng consumer nên luôn truyền tường minh).

### 3.1 GET `/identity/{did}/active`

Query: `?epoch={cardano_epoch}` (optional).
Trả trạng thái tại epoch.

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "active":        true,
    "revoked_at":    null,        // epoch bị revoke, null nếu chưa
    "never_existed": false        // true nếu không có dòng history nào có effective_epoch ≤ epoch
}}
```

Ngữ nghĩa: chọn dòng `did_state_history` mới nhất `effective_epoch ≤ epoch`. Nếu không có dòng nào ⇒ `never_existed=true`, `active=false`. Nếu `is_revoked=true` AND `revoked_epoch ≤ epoch` ⇒ `active=false`, `revoked_at=revoked_epoch`.

### 3.2 GET `/identity/{did}/key-authorized`

Query: `?key={pubkey_or_pkh}&epoch={cardano_epoch}`.

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "authorized": true            // key có kiểm soát did tại epoch không
}}
```

Ngữ nghĩa: dòng `did_state_history` mới nhất `effective_epoch ≤ epoch`; `authorized = (controller_pkh == key OR controller_pubkey_hex == key)` AND state không revoked tại epoch. So khớp chấp nhận cả PKH lẫn pubkey hex.

Đây là call MAGIC `did_commit` bind + Rada revocation dùng chung.

### 3.3 GET `/identity/{did}/revocation`

Query: `?epoch={cardano_epoch}`.

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "status": "not_revoked",      // "not_revoked" | "revoked"
    "reason": null                // revocation_reason nếu revoked, else null
}}
```

Ngữ nghĩa: dòng mới nhất `effective_epoch ≤ epoch`. `revoked` khi `is_revoked=true` AND `revoked_epoch ≤ epoch`.

### 3.4 GET `/identity/{did}/lineage`

Không epoch param — trả toàn bộ DID lịch sử cùng PersonDID với `{did}`.

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "person_did": "did:phx:person:abc...",
    "lineage": [
      { "did": "did:phx:...:v1", "registration_epoch": 410, "revocation_epoch": 455, "revocation_reason": "key_compromise" },
      { "did": "did:phx:...:v2", "registration_epoch": 456, "revocation_epoch": null, "revocation_reason": null }
    ]
}}
```

Ngữ nghĩa: tìm `person_did` của `{did}` (từ `did_state_history.person_did` hoặc `did_lineage.member_did`), trả mọi `member_did` cùng `person_did`, kèm metadata (Contract §2.2). Nếu `{did}` chính là PersonDID ⇒ trả mọi member của nó.

### 3.5 GET `/identity/{did}/dependency-chain`

Trả chuỗi phụ thuộc đi lên tới PersonDID terminal (Stamp §8 P16).

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "chain": [
      { "did": "did:phx:device:...", "edge_type": "DEVICE_OF" },
      { "did": "did:phx:person:...", "edge_type": "TERMINAL" }
    ],
    "terminal_person_did": "did:phx:person:..."
}}
```

Traverse `did_ownership_edge` từ `{did}` theo `child_did → parent_did` tới khi gặp PersonDID (terminal). Dùng cạnh còn hiệu lực ở epoch hiện tại (không param). Chống vòng lặp: giới hạn độ sâu (ví dụ 32 hop) → nếu vượt ⇒ `422` `DEPENDENCY_CHAIN_CYCLE`.

### 3.6 GET `/identity/ownership-graph-snapshot`

Query: `?epoch={cardano_epoch}` (bắt buộc).

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "epoch": 460,
    "nodes": ["did:phx:...", "..."],
    "edges": [
      { "parent_did": "did:phx:org:...", "child_did": "did:phx:...", "edge_type": "OWNS" }
    ]
}}
```

Ngữ nghĩa (Contract §2.3): mọi cạnh `effective_epoch ≤ epoch` AND (`removed_epoch` IS NULL OR `removed_epoch > epoch`). Immutable sau khi đóng epoch.

### 3.7 GET `/identity/are-independent`

Query: `?dids=did1,did2,did3&hop_threshold=2` (default 2).

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "independent": true           // false nếu tồn tại cặp share OrgDID ancestor trong ≤ hop_threshold hop
}}
```

Chạy trên snapshot epoch hiện tại. Cặp bất kỳ trong set share ancestor ≤ `hop_threshold` hop ⇒ `independent=false` (Contract §2.3, Score C.3, default hop=2).

### 3.8 GET `/identity/shared-ancestor`

Query: `?did_a=&did_b=&max_hop=2`.

```
200 OK
{ "code": 1000, "message": "ok", "data": {
    "shared_ancestor": "did:phx:org:..."   // hoặc null nếu không có trong max_hop
}}
```

### 3.9 Pubsub (topic + ACK)

4 topic theo Contract §2.6.1. Cơ chế: **transactional outbox** — indexer ghi `pubsub_event_log` trong cùng transaction với append history (bảo đảm không mất event). Broker/dispatcher đọc outbox → đẩy tới subscriber.

| Topic | Payload (snake_case) | Ghi chú auth |
|---|---|---|
| `did_created` | `{ did, subtype, created_slot, cnf_pubkey }` | Ed25519 emitter sig |
| `did_lineage_updated` | `{ did, old_lineage_hash, new_lineage_hash, updated_slot }` | Ed25519 emitter sig |
| `did_revoked` | `{ did, revocation_reason, revoked_slot }` | Ed25519 emitter sig **+ 2-of-3 DAO multisig** |
| `device_did_bound_to_person` | `{ device_did, person_did, bound_slot, attestation_cid }` | Ed25519 emitter sig |

Delivery (Contract §2.6.2):
- **At-least-once** + ACK. Subscriber ACK trong **30s**; quá hạn ⇒ replay exponential backoff `1s → 60s cap`.
- Idempotent re-sync: `POST /pubsub/replay { subscriber_id, topic, from_slot }` trả event từ `last_acked_slot`.
- ACK: `POST /pubsub/ack { subscriber_id, topic, acked_slot, acked_nonce }` → cập nhật `pubsub_subscription`.
- Anti-replay (Contract §2.6.5): subscriber reject nếu `(slot, nonce) ≤ last_processed_pair`. Server cấp `(cardano_slot, event_nonce)` đơn điệu.
- **`did_revoked` bắt buộc `dao_multisig` 2-of-3**: dispatcher KHÔNG phát nếu thiếu ≥2 chữ ký DAO hợp lệ; event thiếu ⇒ drop + log `REVOKE_MULTISIG_MISSING`.

Auth event (Contract §2.6.3): `signature = sign(emitter_key, H(topic ‖ payload ‖ slot))` (Ed25519). Subscriber verify trước khi xử lý.

---

## 4. Fail-closed

Đây là ràng buộc an toàn — sai chỗ này thì cả VeData/MAGIC/Rada bị lừa.

### 4.1 Khi nào 503

Trả **`503 Service Unavailable`** (KHÔNG 200 với dữ liệu đoán) khi:

1. **Freshness bound vi phạm**: indexer tụt hậu — snapshot Mithril mới nhất `< query_epoch − k`. Nghĩa là ta CHƯA đủ dữ liệu để khẳng định state tại `query_epoch`. `k` cấu hình (đề xuất `k = 1`, khớp SLA "≤ 1 epoch staleness" Contract §3). Health: so `latest_synced_epoch` (max `effective_epoch` đã ghi từ chain đã finalize) với `query_epoch`.
2. **Dữ liệu không có** cho DID + epoch mà đáng lẽ phải có (phân biệt với `never_existed`: `never_existed` là câu trả lời XÁC ĐỊNH khi đã sync tới epoch đó; 503 là khi CHƯA sync tới).
3. **DB / indexer down**.

Body 503:
```
{ "code": 5030, "message": "resolve_unavailable", "data": {
    "reason": "index_behind",            // index_behind | data_gap | backend_down
    "latest_synced_epoch": 458,
    "requested_epoch": 460
}}
```

### 4.2 Nghĩa vụ consumer

Nhận 503 ⇒ **REJECT, KHÔNG proceed** (fail-closed, không fail-open). Áp cho MỌI consumer (Contract §3 degrade + `VeData-Reply §3`):
- Stamp: REJECT record mới.
- MAGIC: từ chối bind/burn.
- Rada: từ chối revocation flow.
- LampNet: từ chối gate.
- Score: được phép dùng cached snapshot ≤ 5 epoch cũ; nếu cũ hơn ⇒ downweight (đây là ngoại lệ Score, không phải hard-fail — theo Contract §3).

### 4.3 Freshness từ Mithril

Indexer chỉ ghi `did_state_history` từ block đã **finalize** (đề xuất neo theo Mithril snapshot / độ sâu xác nhận k-block). Không ghi từ block chưa finalize ⇒ tránh reorg làm sai history. `latest_synced_epoch` chỉ tăng khi block finalize.

---

## 5. Math — ngữ nghĩa point-in-time (để không cãi nhau lúc implement)

Ký hiệu: với DID `d`, tập dòng history `H(d) = { (e_i, s_i) }` sắp theo `effective_epoch` `e_i` tăng.

**Chọn dòng hiệu lực tại epoch q:**
$$
\text{row}(d, q) = \arg\max_{i}\{ e_i : e_i \le q \}
$$
Nếu tập rỗng ⇒ `never_existed(d, q) = true`.

**active(d, q):**
$$
\text{active}(d, q) = \big[\, \text{row}(d,q) \text{ tồn tại} \,\big] \wedge \neg\big[\, \text{is\_revoked} \wedge \text{revoked\_epoch} \le q \,\big]
$$

**key\_authorized(k, d, q):**
$$
\text{key\_authorized}(k,d,q) = \text{active}(d,q) \wedge \big( k = \text{controller\_pkh}(\text{row}(d,q)) \vee k = \text{controller\_pubkey\_hex}(\text{row}(d,q)) \big)
$$

**Key rotation window (Contract §2.5):** với 2 khóa liên tiếp `K1 → K2` của cùng DID:
$$
K_1.\text{revocation\_epoch} + \text{MAX\_KEY\_ROTATION\_GAP} \ge K_2.\text{registration\_epoch}, \quad \text{MAX\_KEY\_ROTATION\_GAP} = 2
$$
Vi phạm ⇒ indexer set cờ `key_rotation_gap_violation` (thêm cột boolean vào `did_state_history` khi append dòng khóa mới, mặc định false) để consumer đọc qua `key-authorized` (thêm field `rotation_gap_violation` vào response 3.2 khi true).

**ownership snapshot tại q:** cạnh `x` thuộc snapshot ⟺ `x.effective_epoch ≤ q ∧ (x.removed_epoch = null ∨ x.removed_epoch > q)`.

---

## 6. Invariants (PIT-*)

| ID | Invariant | Enforce ở đâu |
|---|---|---|
| **PIT-1** | **Append-only history** — không bao giờ UPDATE/DELETE dòng quá khứ của `did_state_history` / `did_ownership_edge`. Thay đổi = INSERT dòng mới. | GRANT DB: app-user không có UPDATE/DELETE; `uq_did_epoch`. Ngoại lệ DUY NHẤT: reorg re-sync bởi role `phx_gc` với audit log. |
| **PIT-2** | **Point-in-time correctness** — `row(d,q)` luôn là dòng có `effective_epoch` lớn nhất `≤ q`. Query không được lẫn state tương lai. | Index `idx_dsh_did_epoch (did, effective_epoch DESC)` + `WHERE effective_epoch ≤ :q ORDER BY effective_epoch DESC LIMIT 1`. |
| **PIT-3** | **Monotonic epoch** — `latest_synced_epoch` chỉ tăng; không ghi history từ block chưa finalize; `effective_epoch` mới ≥ `effective_epoch` đã có cho cùng DID. | Indexer check trước INSERT; kết hợp optimistic lock `last_synced_block` sẵn có. |
| **PIT-4** | **Revocation monotonic** — `revoked_epoch` / `revocation_epoch` một khi set thì immutable (chỉ NULL→value). | Trigger DB: reject update từ non-null. |
| **PIT-5** | **Lineage immutable** — `did_lineage` entry không revisable; đổi lineage = append `member_did` mới + emit `did_lineage_updated`. | `uq_person_member` + không cấp UPDATE cho `member_did`/`registration_epoch`. |
| **PIT-6** | **Fail-closed** — thiếu dữ liệu / index behind ⇒ 503, không trả state đoán. | §4 check `latest_synced_epoch` vs `query_epoch`. |
| **PIT-7** | **Outbox atomicity** — event pubsub ghi trong CÙNG transaction với append history ⇒ không mất/không thừa event. | `@Transactional` bao cả append + INSERT `pubsub_event_log`. |
| **PIT-8** | **Anti-replay** — `(cardano_slot, event_nonce)` đơn điệu, unique. | `uq_slot_nonce`. |

---

## 7. Test plan (cụ thể, phải có evidence output thật)

Seed dữ liệu qua `/internal/sync-taad` (mở rộng payload có `cardano_epoch`), rồi assert qua API. Mọi test integration (Testcontainers Postgres, như dự án đang dùng).

### T1 — Key rotation point-in-time
1. Seed DID `d` với `controller_pkh = K1` tại `epoch=100`.
2. Seed rotation: `controller_pkh = K2` tại `epoch=105`.
3. Assert `GET /identity/d/key-authorized?key=K1&epoch=100` ⇒ `authorized=true`.
4. Assert `GET /identity/d/key-authorized?key=K1&epoch=105` ⇒ `authorized=false` (đã rotate).
5. Assert `GET /identity/d/key-authorized?key=K2&epoch=105` ⇒ `authorized=true`.
6. Assert `GET /identity/d/key-authorized?key=K2&epoch=104` ⇒ `authorized=false` (K2 chưa hiệu lực).

### T2 — Revocation point-in-time
1. Seed DID `d` ACTIVE tại `epoch=200`.
2. Seed revoke tại `epoch=210` reason `key_compromise`.
3. `GET /identity/d/active?epoch=205` ⇒ `active=true, revoked_at=null`.
4. `GET /identity/d/active?epoch=210` ⇒ `active=false, revoked_at=210`.
5. `GET /identity/d/revocation?epoch=210` ⇒ `status=revoked, reason=key_compromise`.
6. `GET /identity/d/revocation?epoch=209` ⇒ `status=not_revoked`.

### T3 — Lineage
1. Seed `person_did=P`, member `d1` reg@410 revoke@455, member `d2` reg@456.
2. `GET /identity/d2/lineage` ⇒ trả cả `d1` (revoke@455) lẫn `d2` (active) cùng `person_did=P`.
3. `GET /identity/d1/lineage` ⇒ cùng kết quả (tra từ DID đã revoke vẫn ra PersonDID — Contract §2.2).

### T4 — Fail-closed / freshness
1. Set `latest_synced_epoch=458`.
2. `GET /identity/d/active?epoch=460` ⇒ `503` body `reason=index_behind, latest_synced_epoch=458`.
3. `GET /identity/d/active?epoch=458` ⇒ `200`.

### T5 — never_existed vs 503
1. DID `d_unknown` chưa từng sync, `latest_synced_epoch=500`.
2. `GET /identity/d_unknown/active?epoch=300` ⇒ `never_existed=true` (đã sync qua 300, xác định không tồn tại).
3. `GET /identity/d_unknown/active?epoch=510` ⇒ `503` (chưa sync tới 510).

### T6 — Ownership snapshot + independence
1. Seed cạnh `OrgO OWNS d1 @ epoch=300`, `OrgO OWNS d2 @ epoch=300`.
2. `GET /identity/ownership-graph-snapshot?epoch=300` ⇒ chứa 2 cạnh.
3. `GET /identity/are-independent?dids=d1,d2&hop_threshold=2` ⇒ `independent=false` (share OrgO ≤2 hop).
4. `GET /identity/shared-ancestor?did_a=d1&did_b=d2` ⇒ `shared_ancestor=OrgO`.

### T7 — Append-only (PIT-1)
1. Seed DID `d` @100, rồi @105.
2. Assert bảng `did_state_history` có ĐÚNG 2 dòng cho `d`; dòng @100 controller_pkh KHÔNG đổi.
3. Sync lại cùng `epoch=105` (idempotent) ⇒ vẫn 2 dòng (`ON CONFLICT DO NOTHING`).

### T8 — Pubsub ACK + replay + did_revoked multisig
1. Emit `did_created` ⇒ subscriber nhận, ACK trong 30s ⇒ `last_acked_slot` cập nhật.
2. Không ACK ⇒ replay (kiểm tra backoff schedule).
3. `POST /pubsub/replay from_slot` ⇒ trả event từ `last_acked_slot`.
4. Emit `did_revoked` thiếu DAO multisig ⇒ dispatcher DROP + log `REVOKE_MULTISIG_MISSING`, subscriber KHÔNG nhận.
5. Emit `did_revoked` đủ 2-of-3 ⇒ phát bình thường.

---

## 8. Ranh giới

| Phần | Ai làm | Ghi chú |
|---|---|---|
| **OFFCHAIN backend** (bảng history, mở rộng `/internal/sync-taad`, 8 endpoint point-in-time, pubsub outbox + ACK, fail-closed) | **Long** — PhoenixKey-Database | Đây là toàn bộ scope PR này. |
| **On-chain TAAD** | KHÔNG sửa | TAAD ĐÃ phát event (created/revoked/rotate). Không đụng validator. Indexer chỉ ĐỌC + index. |
| **MAGIC `did_commit` consumer** | MAGIC team (ngoài scope) | Chỉ CALL `key-authorized`. Ta cung cấp endpoint; MAGIC bỏ sentinel `#""` phía họ. `did_commit = blake2b_256(did ‖ enrolled_slot)` (`VeData-Reply §2`). |
| **Rada revocation consumer** | Rada team (ngoài scope) | Dùng CHUNG `key-authorized` + `revocation` + topic `device_did_bound_to_person`. |
| **Emitter key / DAO multisig quản trị** | Governance (ngoài code) | Ta verify chữ ký; quản lý khóa DAO là quy trình vận hành. |

---

## 9. Checklist bàn giao (Long tick trước khi báo PASS)

- [ ] Migration V16: 5 bảng + index + GRANT (app-user không UPDATE/DELETE trên history) + trigger PIT-4.
- [ ] `SyncTaadRequest` thêm `cardanoEpoch` (không phá tương thích: nếu null, derive từ slot).
- [ ] `IndexerServiceImpl.syncTaad` append history (chỉ khi state đổi) + ghi outbox trong cùng `@Transactional`.
- [ ] `PointInTimeController` — 8 GET endpoint §3.1–3.8, response snake_case bọc `DataResponse`.
- [ ] Pubsub: outbox dispatcher, `/pubsub/ack`, `/pubsub/replay`, ACK 30s + backoff, `did_revoked` 2-of-3 gate.
- [ ] Fail-closed: `latest_synced_epoch` health + 503 body §4.1.
- [ ] Test T1–T8 xanh với evidence output thật (Testcontainers).
- [ ] Swagger `@Operation` cho từng endpoint (dự án đang dùng springdoc).
- [ ] KHÔNG Co-Authored-By trong commit/PR.
