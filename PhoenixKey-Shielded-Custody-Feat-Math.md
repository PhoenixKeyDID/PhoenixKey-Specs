# PhoenixKey — Shielded Phoenix Custody (Feat + Math)

> Doc-type: Feature Spec (canonical, build-ready) · Status: **CHỐT — Phase 1 build ngay**
> Người build + test: **Long** · Người viết spec: Claude (KHÔNG sửa backend/DID-Document)
> Nguồn phán quyết: `spec-proposals/_compete/JUDGE-verdict.md` §3–§5 (Hybrid theo pha).
> Nền đã đối chiếu byte-level:
> - `PhoenixKey-Core/Enclave/rust_core/src/phoenix_address.rs` — Ví Phượng hoàng per-DID cố định (`derive_phoenix_address`, `did_payment` applied).
> - `LampNetCloud/lampnet-hivemind/lampnet-merkle-anchor` — Strata thật: `sumtree.rs` (Merkle-Sum-Tree hash+sum+count, `verify_sum_range`), `mmr.rs` (MMR append-only, dup-leaf guard, commit `n`), `hash.rs` (`H_dom` domain-separated).
> - `MAGIC/SPEC/MagicLamp-3Token-DacTa-Vi.md §1.3 / §5` — CARP = native token thật, **policy-id riêng**, chuyển nhượng được.
> - `PhoenixKey-Specs/PhoenixKey-Math.md §6/§8` — Master_KEK (256-bit Secure Enclave) → HKDF domain-separated branches.
> - `VeDataIO/Specs/Query-Feat-Spec.md §F1/§F4/§F4.proof_only/§F5/§F8` — gated read fail-closed, D9 `proof_only` (Groth16), `D9_REVOCATION` pubsub, audit-log MMR.
> - `VeDataIO/Specs/PhoenixKey-VeData-Contract.md §2.1/§2.4` — `did_active`/`key_authorized` point-in-time cho revocation.

---

## §0. Banner + vấn đề + phán quyết

**Triple Token (KHÔNG dual, KHÔNG single-native):** `LAMP` (Đèn — tài sản nền) · `MAGIC` (Điều ước — Account-trong-Vault, không chuyển nhượng, KHÔNG policy-id) · `CARP` (Thảm — **native token thật, policy-id riêng, chuyển nhượng được**). Custody này giữ **CARP** (và ADA/LAMP đi kèm). MAGIC KHÔNG nằm trong pool — nó là số-dư-quyền-tiêu trong vault datum, không phải value on-chain.

**Vấn đề 1 câu:** Doanh nghiệp nhận CARP vào một địa chỉ công khai cố định (lộ danh tính là CHẤP NHẬN), nhưng **explorer KHÔNG được đọc tồn dư** của doanh nghiệp, và khi cần thì **mở cho ngân hàng/kiểm toán xem theo yêu cầu**.

**Phán quyết (từ JUDGE §3):**
- **Phase 1 — canonical, build NGAY, test được trên Preview tuần này:** Approach 2 (Commitment-Pool trên Cardano L1, Merkle-Sum-Tree). Ẩn **tồn dư** bằng bằng-chứng-mật-mã (không phải obscurity). Reveal-on-demand qua viewing-key gated read. Đây là toàn bộ deliverable §9.
- **Phase 2 — chương tương lai (§8):** Approach 1 (Midnight overlay) bọc lên pool để ẩn **cả dòng tiền** — khi cầu Cardano↔Midnight chín. Đánh dấu rõ là tương lai, trung thực về bài toán relayer/cầu bán-tin-cậy.
- **Bất biến interface xuyên 2 pha:** `Ví Phượng hoàng (per-DID, cố định) → sweep → Pool UTxO`; cùng gốc **Master_KEK**.

---

## §1. Yêu cầu (6 tiêu chí của anh) + đạt/không

| # | Yêu cầu | Mức | Phase 1 đạt? | Cơ chế |
|---|---|---|---|---|
| R1 | Địa chỉ nhận công khai, cố định, neo-DID (Phoenix Custody). | GIỮ CỨNG | ✅ | `derive_phoenix_address` — `did_payment` applied per-DID, KHÔNG xoay đời DID. |
| R2 | Ẩn số dư hiện tại. Explorer KHÔNG thấy tồn dư từng DID. | BẮT BUỘC | ✅ **mật mã** | Merkle-Sum root + salt-per-leaf; on-chain chỉ có `pool_total` gộp + `mst_root`. Sum-invariant chống operator in-tiền-ảo. |
| R3 | Ẩn lịch sử / dòng tiền. | RẤT MUỐN | 🟡 một phần | Batching sweep trộn `Δpool_total`; nhưng sweep-in timing/amount vẫn lộ (§7). Ẩn-flow-THẬT hoãn Phase 2 (§8). |
| R4 | Reveal-on-demand cho ngân hàng/kiểm toán. | Bắt buộc | ✅ | Viewing-key gated read (mẫu Query §F1/§F4, fail-closed) + proof-only ≥ngưỡng (Query §F4.proof_only, đặc tả sẵn — mạch ZK defer). |
| R5 | Build + test trên Preview càng sớm càng tốt. | Nặng ký | ✅ | Thuần Cardano; tái dùng `phoenix_address`, Strata MST/MMR (đã test), CARP native. Không bridge, không chain thứ hai. |
| R6 | Trust-minimization. | Ưu tiên | 🟡 **safety** | Operator KHÔNG rút được tiền (withdraw cần chữ ký authority DID), KHÔNG sửa số dư mà không bị bắt (fail-closed proof-mismatch). NHƯNG operator biết plaintext số dư (privacy vs explorer, KHÔNG vs operator — §7). |

**Chốt vạch:** R1+R2+R4+R5 đạt chắc; R6 đạt về-safety; R3 đạt một phần và hoãn phần lõi sang Phase 2. Đây là ranh giới vật lý của L1 công khai, khai báo thẳng ở §7.

---

## §2. Kiến trúc Phase 1 + sơ đồ

```
                  CÔNG KHAI (explorer đọc được)                    │  RIÊNG TƯ (off-chain)
 ┌───────────────────────────────────────────────────────────────┐│
 │ Ví Phượng hoàng — DoanhNghiệp A                                ││   ┌──────────────────────┐
 │  addr = derive_phoenix_address(did_A, taad_policy, network)    ││   │ Indexer/Accountant   │
 │  = did_payment applied per-DID (KHÔNG xoay — phoenix_address.rs)││   │ (LampNet-hosted)     │
 │  khách/đối tác trả CARP vào đây; ai cũng biết là của A → OK(R1)││   │ giữ cây MST thật:    │
 └───────────────┬───────────────────────────────────────────────┘│   │  leaves[DID→balance] │
        (1) nhận CARP x  (public tx: sender+amount+slot lộ)         │   │  + salt mỗi lá       │
                 │                                                  │   │  + signed event log  │
        (2) SWEEP crank (permissionless; batching theo cửa-sổ):     │   └──────────┬───────────┘
            tiêu UTxO ở addr A → KHOÁ value vào Pool (không rút ra) │              │ gated read
                 ▼                                                  │  (viewing-key hợp lệ)
 ┌───────────────────────────────────────────────────────────────┐│              │
 │ POOL UTxO  (shared custody script `did_pool`, 1 UTxO / shard)  ││◀───anchor────┤
 │  value = Σ mọi DID (CARP + min-ADA)  — CÔNG KHAI               ││   (3) mỗi update: validator
 │  datum = PoolDatum {                                           ││       ÉP mst_root' đúng theo
 │    pool_total : Value  // = Σ leaves  (gộp, KHÔNG chia DID)    ││       Merkle-Sum update
 │    mst_root   : Hash   // Merkle-Sum root {DID→balance}        ││              │
 │    epoch      : Int    // monotonic, chống rollback           ││   (4) A cấp viewing-key
 │    log_root   : Hash   // MMR nhật ký sự kiện (anti-rollback)  ││       cho ngân hàng →
 │  }                                                            ││       thấy balance_A +
 └───────────────────────────────────────────────────────────────┘│       proof của RIÊNG A
      Explorer thấy: addr A ≈ 0 (đã sweep); Pool tổng to, KHÔNG    │       (không lộ lá khác)
      tách theo DID.                                               │
                 ▲                                                  │
      (5) RÚT: A ký (authority DID hiện tại) + nộp MST proof lá     │
          mình → validator kiểm proof ∧ root ∧ sig ∧ sum → chi x'  │
          ra withdraw_target; root cập nhật (trừ lá A), epoch++     │
```

**Ba tầng khoá bám hạ tầng đã có:**
- **Địa chỉ nhận** = `did_payment` script per-DID (`phoenix_address.rs`). Quyền chi đi theo authority HIỆN TẠI của DID, đọc động qua reference input anchor TAAD — sống sót rotate/recovery.
- **Cây số dư** = Merkle-Sum-Tree (Strata `sumtree.rs`): mỗi node `(hash, sum, count)`; `verify_sum_range` cho phép kiểm một lá + tổng mà không lộ lá khác (field-proof).
- **Nhật ký + neo** = MMR append-only (Strata `mmr.rs`): dup-leaf guard (CVE-2012-2459), commit số lá `n` vào root ⇒ chống rollback.

---

## §3. Data schema

### §3.1 PoolDatum (on-chain, per shard)

```
PoolDatum {
  pool_total : Value,   // tổng value khoá; MUST == mst_root.sum (SHIELD-1)
  mst_root   : Hash32,  // Merkle-Sum root của cây {DID→balance}
  epoch      : Int,     // monotonic tăng mỗi tx (SHIELD-2)
  log_root   : Hash32,  // MMR root nhật ký sự kiện (SHIELD-3)
  leaf_count : Int,     // = mst_root.count (số DID trong shard) — commit để chống mở rộng cây lén
}
```
> Ghi chú: on-chain KHÔNG bao giờ có field per-DID. Chỉ 3 hash/số + `pool_total`. PII/plaintext KHÔNG lên chain.

### §3.2 Leaf commitment (off-chain, cây MST)

```
leaf_bytes(DID, balance, salt) = DID_bytes ‖ salt          // 32-byte salt ngẫu nhiên mỗi update
leaf_value                     = balance                    // u128 (lovelace/CARP-unit)
leaf_node = SumTree::leaf_node(leaf_bytes, leaf_value)
          = Node { hash = H_dom(TAG_LEAF, LEAF_PREFIX ‖ balance_be ‖ leaf_bytes),
                   sum  = balance, count = 1 }              // đúng sumtree.rs
```
- **salt** ⇒ explorer/operator-đối-thủ không brute-force ra `balance` kể cả khi đoán dải giá trị; đổi salt mỗi update ⇒ commitment không tái-nhận-diện được.
- Cây build bằng `SumTree::<Blake3Hasher>::build(&[(leaf_bytes, balance)…])`; `root()` = `mst_root`; `root_node().sum` = `pool_total`; `root_node().count` = `leaf_count`.

### §3.3 Redeemer (`did_pool` validator)

```
Redeemer =
  | Sweep    { deposits: [(DID, amount, new_salt)], updates: [MstUpdateProof] }
  | Withdraw { DID, amount, sig, old_proof: MstProof, new_salt, withdraw_target }
  | Reanchor { new_log_root }        // chỉ append nhật ký, value + mst_root BẤT BIẾN
```
- `MstUpdateProof` / `MstProof` = đường-lá Merkle-Sum (mẫu `SumRangeProof` — Pruned/Internal) đủ để tái dựng root cũ và root mới.
- `sig` = Ed25519 chữ ký của authority DID hiện tại trên `(DID ‖ amount ‖ new_salt ‖ withdraw_target ‖ epoch)`.

### §3.4 ViewingKeyGrant record (off-chain, indexer ACL)

```
ViewingKeyGrant {
  did          : DID,          // chủ sở hữu lá
  grantee_hash : Hash32,       // = H(grantee_did) — ai được xem
  viewing_key  : Hash32,       // = HKDF(VaultKEK_A, info="pool-view-v1", salt=grantee_hash)
  scope        : Enum { balance_only, balance_and_history },
  valid_until  : Epoch,        // point-in-time revoke (Contract §2.1)
  granted_at   : Epoch,
  revoked      : Bool,         // set khi rotate salt / hết hạn → D9_REVOCATION event vào log_root
}
```

---

## §4. Validator `did_pool` — bất biến (SHIELD-*)

Aiken **v1.1.21** (khớp `did_payment`, xem `phoenix_address.rs` header). Applied per-shard nếu `K>1`. Validator KHÔNG giữ cây đầy đủ — chỉ kiểm đường-lá do người gọi cung cấp; ExUnit ~ `O(log N)` theo độ sâu, KHÔNG theo số DID.

```
validate(datum: PoolDatum, redeemer, ctx):

  case Sweep(deposits, updates):
    // SHIELD-4 value-conservation vào: value vào pool khớp Σ deposits, input từ đúng addr DID
    require Σ ctx.inputs(from did_payment(DID_i) addr) == Σ amount_i
    root' = fold deposits qua MST_update(datum.mst_root, updates)   // balance_i += amount_i, salt := new_salt
    require new_datum.mst_root   == root'.hash
    require new_datum.pool_total == datum.pool_total + Σ amount_i == root'.sum   // SHIELD-1
    require new_datum.leaf_count == root'.count
    require new_datum.epoch      == datum.epoch + 1                              // SHIELD-2
    require new_datum.log_root   == MMR_append(datum.log_root, event(Sweep, deposits))  // SHIELD-3

  case Withdraw(DID, amount, sig, old_proof, new_salt, withdraw_target):
    // SHIELD-5: chủ DID ký — authority HIỆN TẠI, đọc anchor TAAD như did_payment
    require verify_did_authority(DID, sig, ctx.reference_inputs.anchor(DID))
    // SHIELD-6: proof lá cũ hợp lệ với root hiện tại (field-proof, verify_sum_range)
    require MST_verify(datum.mst_root, old_proof, leaf(DID, old_bal, old_salt))
    require old_bal >= amount                                                   // SHIELD-7 không âm
    root' = MST_update_leaf(old_proof, leaf(DID, old_bal - amount, new_salt))
    require new_datum.mst_root   == root'.hash
    require new_datum.pool_total == datum.pool_total - amount == root'.sum       // SHIELD-1
    require Σ ctx.outputs(to withdraw_target) == amount                          // SHIELD-8 chi đúng
    require new_datum.epoch      == datum.epoch + 1                              // SHIELD-2
    require new_datum.log_root   == MMR_append(datum.log_root, event(Withdraw, DID, amount))

  case Reanchor(new_log_root):
    require new_datum.mst_root   == datum.mst_root       // value + phân bổ BẤT BIẾN
    require new_datum.pool_total == datum.pool_total
    require ctx.value_in == ctx.value_out                // SHIELD-9 không rút được gì
    require new_datum.log_root == new_log_root
    require new_datum.epoch    == datum.epoch + 1
```

**Bảng bất biến (IDs — Long test từng cái):**

| ID | Bất biến | Ý nghĩa an ninh |
|---|---|---|
| **SHIELD-1** | `pool_total == mst_root.sum` mọi tx | Sum-invariant: operator KHÔNG in-tiền-ảo, KHÔNG giấu-bớt-tổng. `pool_total` cao phải có value thật khoá. |
| **SHIELD-2** | `epoch` monotonic `+1` mỗi tx | Chống replay + rollback về state cũ có lợi. |
| **SHIELD-3** | `log_root == MMR_append(...)` append-only | Nhật ký không xoá được sự kiện đã neo (dup-leaf guard + commit `n` — `mmr.rs`). Anti-rollback log. |
| **SHIELD-4** | Value vào pool == Σ deposits, input từ addr DID | Sweep không bơm khống, không trộn value lạ. |
| **SHIELD-5** | Withdraw cần chữ ký authority DID hiện tại | **Operator KHÔNG rút được tiền.** Đọc anchor TAAD như `did_payment` → sống sót rotate. |
| **SHIELD-6** | MST field-proof lá cũ khớp root | Không rút lá không tồn tại / số dư giả. |
| **SHIELD-7** | `old_bal >= amount` | Không rút âm / vượt số dư. |
| **SHIELD-8** | Output tới `withdraw_target` == amount | Chi đúng số, không rò thêm. |
| **SHIELD-9** | Reanchor: `value_in == value_out`, mst_root bất biến | Cập nhật nhật ký KHÔNG chạm tài sản. |

**Golden vector cần Long dựng:** MST build/verify khớp `sumtree.rs::verify_sum_range` + red-team "sửa-lá-off-chain ⇒ proof-fail" (fail-closed). MMR append khớp `mmr.rs::verify`.

---

## §5. Off-chain (Rust — tái dùng Strata + rust_core)

### §5.1 Ba thành phần

- **Indexer/Accountant** (LampNet-hosted, in-process Strata): giữ cây MST đầy đủ `{DID → (balance, salt)}` + signed event log (MMR). Sau mỗi tx: tái dựng `mst_root`/`pool_total`/`leaf_count`, neo vào PoolDatum. API gated-read trả lá + `SumRangeProof` theo viewing-key hợp lệ (mẫu Query §F1/§F4, **fail-closed**: không có key hợp lệ ⇒ DENY + audit-log, KHÔNG trả content).
- **Sweep crank** (bot permissionless): quét ví Phượng hoàng có balance > 0 → build sweep tx **batching theo cửa-sổ** (~1 khối, gom nhiều DID cùng shard) → submit Preview. Batching trộn `Δpool_total` (giảm rò §7).
- **Withdraw builder** (app A): build tx Withdraw = MST proof lá + chữ ký authority DID (đường ký `TAAD_Key` của `rust_core`) + `new_salt` + `withdraw_target`. **Địa-chỉ-rút-một-lần** (mượn kỹ thuật từ Approach 3, chỉ kỹ thuật KHÔNG kiến trúc): `withdraw_target` là địa chỉ dùng-một-lần / đổi salt để giảm de-anonymize đích rút.

### §5.2 Tamper-evidence 3 lớp

1. **On-chain anchor:** `mst_root`/`pool_total`/`epoch`/`log_root` lên chain mỗi tx. Không ai đổi số dư mà không để lại tx.
2. **Signed event log (MMR):** mỗi deposit/withdraw = event ký bởi operator **VÀ** chữ ký chủ DID; `MMR_append` vào `log_root` on-chain. Append-only (mẫu Query audit-log §F2).
3. **Client self-accounting:** app A tự giữ event của riêng A + tự tính `balance_A_expected`. Operator KHÔNG phải nguồn-sự-thật duy nhất.

### §5.3 Operator nói dối thì sao (kịch bản xấu nhất)

| Kiểu gian lận | Chặn? | Cơ chế |
|---|---|---|
| Sửa số dư 1 DID không qua tx | ✅ | `mst_root` chỉ đổi qua validator; sửa lá off-chain ⇒ proof-fail cho chính DID đó (fail-closed, chủ tự phát hiện). |
| In-tiền-ảo (`pool_total` khống) | ✅ cứng | SHIELD-1: `pool_total == mst_root.sum` **VÀ** value UTxO thật phải khớp. |
| Giấu-bớt số dư A | 🟡 phát hiện, không ngăn | A cầm signed receipts + tx on-chain ⇒ tố cáo tamper-evident. Zero-sum: "cho A ít" ⇒ "cho B nhiều", B phản đối. **Value KHÔNG mất** (còn trong pool). |
| Operator offline (DoS) | 🟡 liveness | Cây tái dựng từ signed event log + tx history (neo qua `log_root`). Multi-indexer đối chiếu root. |
| Rollback state cũ | ✅ | SHIELD-2 + SHIELD-3 (`epoch` monotonic + MMR commit `n`). |

**Tệ nhất = tranh chấp phân bổ, KHÔNG mất tiền** (withdraw cần chữ ký chủ — SHIELD-5). "Trustless-về-safety, trusted-về-liveness."

---

## §6. Selective disclosure + revocation

### §6.1 Hai chế độ reveal (mẫu Query D9)

```
Chế độ 1 — VIEW-KEY GRANT (số dư sống + timeline của RIÊNG A):
  viewing_key = HKDF(VaultKEK_A, info="pool-view-v1", salt=H(grantee_did))
  → gửi ngân hàng qua kênh an toàn. Indexer gate (Query §F1/§F4, fail-closed):
    chỉ trả lá + SumRangeProof của A cho ai xuất trình viewing_key hợp lệ.
  → ngân hàng thấy balance_A + deposit/withdraw của A + proof mỗi mốc (kiểm với
    mst_root on-chain) — KHÔNG thấy lá DID khác.

Chế độ 2 — PROOF-ONLY ≥ngưỡng (Query §F4.proof_only, mạch ZK DEFER Phase 2):
  A xuất ZK proof: ∃ (balance_A, salt_A):
      MST_verify(mst_root_onchain, path_A, leaf(DID_A, balance_A, salt_A)) = ⊤
      ∧ balance_A ≥ threshold
  → auditor verify với mst_root on-chain (public input), KHÔNG học balance chính xác.
    Groth16 (192B, ~3ms verify) mẫu Query §F4.proof_only. Đặc tả sẵn; mạch defer §9-P2.
```

### §6.2 Revocation (salt-rotate + point-in-time)

Viewing-key là khoá đối xứng đã trao ⇒ **không thu hồi được về mật mã**. Revocation THẬT = cắt tầm nhìn TƯƠNG LAI + chặn query ở lớp ACL:

1. **Salt-rotate:** A làm một Withdraw+Redeposit 0-amount đổi `salt_A` → key cũ chỉ còn chứng minh được ảnh-quá-khứ (root cũ), KHÔNG đọc trạng thái mới.
2. **Point-in-time (Contract §2.1):** grant có `valid_until`; indexer gọi `did_active(DID, query_epoch)` / `key_authorized` — sau `valid_until`, gate DENY. Đây là điều phối mốc-thời-gian, khớp `PhoenixKey-VeData-Contract v0.2.0 §2.1/§2.4`.
3. **D9_REVOCATION event:** ghi vào log MMR (`log_root`) + pubsub (Query §F8) ⇒ tamper-evident, TTL cache siết ≤ 5s cho record private.

> Trung thực: bên đã nhận key thấy quá-khứ mãi mãi. "Revoke" = cắt tương-lai (rotate) + chặn query hợp lệ (ACL). Khớp giới hạn Query §F4.

---

## §7. Giới hạn (khai báo thẳng — đừng giấu anh)

1. **Ẩn STOCK, KHÔNG ẩn FLOW hoàn toàn.** Merkle-Sum giấu **tồn dư** bằng mật mã (R2 chắc). NHƯNG mỗi sweep tiêu input từ addr công khai của A ⇒ **timing + độ lớn mỗi khoản NHẬN của A lộ** trước khi sweep xoá. "A vừa nhận 500 CARP lúc slot T" là công khai. Đây là ranh giới vật lý L1, KHÔNG phải lỗi cài đặt.
2. **Bảo mật flow phụ thuộc mật độ.** Batching chỉ trộn khi nhiều DID nạp cùng cửa-sổ. Preview ít DID ⇒ anonymity set ≈ 1 ⇒ `Δpool_total` suy ngược trực tiếp = khoản của một DID. Che flow ≈ 0 khi thưa.
3. **Operator biết plaintext số dư.** Indexer giữ mọi lá `{DID→balance}` plaintext để dựng cây. **Privacy với explorer, KHÔNG privacy với operator.** Hack/subpoena indexer ⇒ lộ số dư mọi doanh nghiệp. Giảm nhẹ (Phase 2): mã-hoá-lá per-DID + multi-indexer đối chiếu root — làm proof phức tạp hơn.
4. **Pool 1-UTxO = contention.** eUTXO thù địch "một sổ chung nhiều người ghi". Sharding `shard = blake3(DID) mod K` giải được nhưng đánh đổi anonymity set (shard nhỏ = ít người trộn). Preview demo `K=1` là đủ chứng minh cơ chế.
5. **Withdraw target de-anonymize.** Rút ra một địa chỉ truy được về A ⇒ nối lại. Giảm nhẹ = địa-chỉ-rút-một-lần (§5.1), không vá triệt để.
6. **Proof-only ZK (chế độ 2) chưa có mạch.** Chế độ 1 (gated read) build ngay; chế độ 2 cần circuit Groth16 — defer Phase 2 (mẫu Query §F4.proof_only).

**Một câu:** Phase 1 = "che-kế-toán mật-mã trên L1 công khai" (ẩn số dư có bằng chứng, safety trust-min), KHÔNG phải "shielded ledger đúng nghĩa" (ẩn cả flow). Vế "ideally ẩn tx history" chỉ đạt một phần — phần lõi ở Phase 2.

---

## §8. Chương Phase 2 — Midnight overlay (TƯƠNG LAI, trung thực về cầu)

> **Trạng thái: KHÔNG build Phase 1.** Đây là đường nâng cấp mở sẵn để mua R3-thật khi cầu/tooling chín. Không đập Phase 1 làm lại — cộng dồn.

**Ý tưởng:** thêm nhánh tuỳ chọn cho DID muốn ẩn CẢ flow: `Pool → escrow (Cardano) → mint wCARP shielded (Midnight)`. Số dư đã ẩn (Phase 1 MST) **và** flow tiếp theo ẩn (ZSwap Pedersen commitment giấu amount).

**Key-derivation (cùng gốc Master_KEK, KHÔNG secret mới):**
```
Master_KEK
  ├─ HKDF("wallet-v1")          → ví Cardano CIP-1852 (Phase 1 — hiện có)
  └─ HKDF("midnight-zswap-v1")  → seed 32B → wallet-sdk-hd role Zswap
        ├─ ZswapCoinSecretKey       (spending)
        └─ ZswapEncryptionSecretKey (viewing — cấp auditor)
```

**Điểm nối với Phase 1:** `log_root` (MMR) của PoolDatum làm điểm neo binding-record khi bắc sang Midnight; viewing-key model 2 tầng tái dùng.

**Bất biến giữ:**
- **P2-INV-1:** "Midnight chết ⇒ mất RIÊNG-TƯ, KHÔNG mất TÀI SẢN." Timelock redeem: không redeem sau T ⇒ chủ DID ký Master_KEK rút thẳng CARP về Cardano từ escrow.
- **P2-INV-2:** viewing ≠ spending (grant view không trao quyền tiêu).

**Trung thực về TỬ HUYỆT (JUDGE §1 A1):**
- **KHÔNG có cầu Cardano↔Midnight gốc cho token tuỳ ý hôm nay.** Phải tự dựng lock-and-mint qua escrow + relayer.
- **Relayer thấy toàn bộ mapping `lock ↔ mint`** = thấy đúng dòng tiền đang giấu. Đây là mâu thuẫn nội tại: cơ chế bảo mật tạo một điểm-biết-hết mới. Chỉ **dịch** niềm tin (explorer → relayer), chưa **xoá**. Light-client trust-minimized để bỏ relayer **CHƯA TỒN TẠI**.
- **wCARP là IOU**, giá trị = niềm tin escrow còn CARP + relayer honor redeem. KHÔNG phải CARP thật khi trong vùng shielded.
- **[GIẢ-ĐỊNH — Long verify khi tới Phase 2]:** export `mn_shield-esk`/`mn_shield-addr` từ seed 32B tuỳ ý vào `wallet-sdk-hd` role Zswap (chưa rõ trên forum Midnight). Blocker mức trung bình.

**Cửa quyết định Phase 2:** chỉ mở khi có cầu trust-minimized (hoặc relayer-set t-of-n chấp nhận được) + tooling Midnight ổn định. Trước đó, Phase 1 là canonical.

---

## §9. Việc Long build + test plan trên Preview

### §9.1 Phase 1 — build (tái dùng tối đa)

**On-chain (Aiken, PhoenixKey-Validator):**
- `did_pool` validator: redeemer `Sweep | Withdraw | Reanchor`; datum `PoolDatum` (§3.1). Kiểm đủ SHIELD-1..9 (§4). Aiken **v1.1.21** (khớp `did_payment`). Reference-input đọc anchor TAAD của DID cho `verify_did_authority` — KHỚP cách `did_payment` đọc anchor.
- Applied per-shard nếu `K>1`; Preview demo `K=1`.

**Off-chain (Rust, tái dùng Strata `lampnet-merkle-anchor` + `rust_core`):**
- Indexer/Accountant: cây MST (`SumTree::<Blake3Hasher>`) + signed event log (`Mmr`); API gated-read fail-closed (mẫu Query §F1/§F4).
- Sweep crank: quét addr Phượng hoàng balance>0 → sweep tx batching → submit Preview.
- Withdraw builder: `SumRangeProof` lá + chữ ký `TAAD_Key` (rust_core) + new_salt + withdraw_target một-lần.
- Viewing-key: `HKDF(VaultKEK_A, info="pool-view-v1", salt=H(grantee))`; ACL + `D9_REVOCATION` event.

### §9.2 Test plan trên Preview (bước cụ thể — evidence tx-hash THẬT)

1. **Deploy:** `aiken build` + `aiken blueprint apply` `did_pool` (K=1) → deploy script tới **Preview**. Ghi script hash + address.
2. **Mint + nhận:** mint CARP test (policy-id CARP native) → gửi vào 2–3 ví Phượng hoàng (DID khác nhau) qua `derive_phoenix_address(did, taad_policy, network=2)`. **Kỳ vọng:** explorer thấy tx nhận + amount ở mỗi addr.
3. **Sweep:** chạy sweep crank gom vào 1 Pool UTxO. **Kỳ vọng:** explorer thấy addr Phượng hoàng ≈ 0, Pool `value` tổng to, datum chỉ có `pool_total`/`mst_root`/`epoch`/`log_root` — **KHÔNG có field per-DID** (verify R2 bằng mắt trên explorer).
4. **Prove qua viewing-key:** ngân hàng-giả cầm `viewing_key_bank` → indexer trả `balance_A` + `SumRangeProof` → verify `verify_sum_range(mst_root_onchain, …)` == ⊤. **Kỳ vọng:** khớp root on-chain; ngân hàng KHÔNG đọc được lá DID khác.
5. **Reveal-on-demand tới mock auditor:** cấp grant `scope=balance_and_history`, `valid_until` → auditor thấy timeline A; sau `valid_until` → gate DENY (point-in-time §6.2).
6. **Withdraw:** A build Withdraw (proof + sig authority + new_salt) → validator kiểm SHIELD-5/6/7/8 → chi CARP ra withdraw_target một-lần; root cập nhật, `epoch++`. **Kỳ vọng:** tx pass, `pool_total` giảm đúng amount.
7. **Red-team fail-closed (bắt buộc có evidence):**
   - Sửa lá off-chain rồi thử prove ⇒ **proof-fail** (SHIELD-6).
   - Thử Withdraw KHÔNG chữ ký authority ⇒ **validator reject** (SHIELD-5). Chứng minh operator KHÔNG rút được.
   - Thử set `pool_total` > `mst_root.sum` ⇒ **reject** (SHIELD-1).
   - Thử submit datum `epoch` cũ ⇒ **reject** (SHIELD-2).

**Điều kiện PASS Phase 1:** cả 7 bước có output thật (tx-hash Preview + explorer screenshot cho bước 3); red-team bước 7 fail-closed đủ 4 mục.

### §9.3 Phase 2 gate (sau)
Sharding `K>1`; proof-only ZK (Groth16); multi-indexer đối chiếu root; nhánh Midnight escrow (chỉ khi cầu chín — §8).

---

## §10. Ranh giới (ai làm gì)

- **Claude:** chỉ viết spec này. KHÔNG sửa backend / DID-Document / validator.
- **Long (backend PhoenixKey — ngoài ranh giới Claude):**
  - File schema on-chain PoolDatum + validator `did_pool` update path.
  - Resolver trả `mst_root`/`pool_total`/`epoch`/`log_root` cho client kiểm.
  - Neo `log_root` (MMR) mỗi tx — tamper-evident audit.
  - **Thêm field vào DID-Document** (nếu cần: pointer tới shard/pool, viewing-key ACL endpoint) — việc Long.
- **PhoenixKey team:** key-derivation `HKDF(VaultKEK_A, "pool-view-v1", …)` từ Master_KEK; viewing-key.
- **VeData team:** binding-record + Query gated-view (§F1/§F4) + `did_active`/`key_authorized` point-in-time (Contract §2.1) + `D9_REVOCATION` pubsub.

Phát hiện lỗi trong backend/validator ⇒ Claude báo anh / tạo Issue giao Long, KHÔNG tự sửa.

---

*Hết spec canonical. Phase 1 (Approach 2 Pool MST) = build + test NGAY trên Preview: ẩn số dư mật-mã (R2), reveal-on-demand (R4), safety trust-min (R6-safety). Phase 2 (Midnight overlay) = chương tương lai mở sẵn cho R3-thật, trung thực về cầu bán-tin-cậy. Bất biến xuyên 2 pha: Ví Phượng hoàng per-DID cố định → sweep → Pool UTxO; cùng gốc Master_KEK.*
