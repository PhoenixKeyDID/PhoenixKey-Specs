# PhoenixKey — Easteregg · Đặc tả KỸ THUẬT (Tech, cho IMPLEMENTER)

> Module: **Easteregg** (các mức riêng tư của ví Phoenix) · Loại doc: Tech (kỹ thuật, cho implementer) · Ngày: 2026-07-09
> Đối tượng đọc: kỹ sư build — **đội on-chain** (Aiken), **đội backend** (Rust), **VeData/Glint** (ZK).
> Tài liệu cùng bộ: `-Vi-Feat.md` · `-Math.md` (bất biến I-EGG-n) · `-Exec.md`.
>
> **Phân loại (quyết định maintainer 2026-07-09):** ví CHỈ 2 LOẠI — `Phoenix`/`Standard`. Easteregg
> KHÔNG phải loại ví thứ ba, mà là các **mức riêng tư** áp lên ví Phoenix (loại ví × mức riêng tư).
> "Tầng 0" (`did_subaddr`) dưới đây = **L3** trong `PhoenixKey-Rebirthme-Tech.md` — cùng một
> validator, "L3" là tên canonical cho tầng địa chỉ, "Tầng 0" là tên nội bộ ngữ cảnh Easteregg.
> `did_pool` (Tầng 1) KHÔNG phải "một loại địa chỉ" — là pool kế toán dùng chung (Merkle-Sum-Tree),
> nhận tiền qua Sweep, KHÔNG nhận trực tiếp.
>
> **Nguồn:** `PhoenixKey-Shielded-Custody-Feat-Math.md §3–§5,§9` (Tầng 1 build-ready);
> `PhoenixKey-Easteregg-Feat-Math.md` (hợp nhất); `-Gaps-Addendum-Feat.md` (5 gap);
> `-PrivacyModes-Addendum-Feat.md` (Mode Matrix + External-Adapter);
> `Easteregg/Easteregg-ZK-{Math,Tech,Exec-Spec}.md` (Tầng 2, VeData).

---

## 0. Ranh giới thành phần + quy ước

| Thành phần | Vai trò | Ghi chú |
|---|---|---|
| `validators/did_pool.ak` | Tầng 1 lõi — pool custody MST | đội on-chain build |
| `validators/did_subaddr.ak` | Tầng 0 eAddr unlinkable (DEP-2) | đội on-chain build |
| Indexer/Accountant (Rust) | giữ cây MST + sweep/withdraw | đội backend — Strata in-process |
| Sweep crank / Withdraw builder | build+submit tx | đội backend |
| ZK circuit `egg-zk:withdraw` + verifier Aiken | Tầng 2 | VeData/Glint; verifier PhoenixKey TỰ VIẾT + AUDIT |
| Primitive MST/MMR (`sumtree.rs`/`mmr.rs`/`hash.rs`) | tái dùng | LampNet/Strata — **kế thừa**, Easteregg consume |
| `plutus.json` (Easteregg) | blueprint | sinh ra khi validator build |
| PoC Python off-chain (`PhoenixKey-PoC/easteregg-shielded-demo/`) | minh hoạ nguyên lý | ẩn số dư + gated-proof trên Preview; sweep vào ví thường (không thay validator SHIELD-5) |

> **Không bịa test count / hash.** Số ExUnit ZK (2.842B) đo THẬT thuộc bộ Easteregg-ZK (VeData),
> độc lập với code PhoenixKey. Aiken target = **v1.1.21** (khớp `did_payment`/`phoenix_address.rs`).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 1. Kiến trúc + sơ đồ thành phần (ai build gì)

```
Easteregg = mức riêng-tư của ví Phoenix (KHÔNG phải loại ví riêng)
├─ Tầng 0  did_subaddr.ak         [đội on-chain]   — eAddr unlinkable (= L3, DEP-2 MultiAddress)
├─ Tầng 1  did_pool.ak            [đội on-chain]   — MST pool custody (I-EGG-1..9)  ← BUILD TRƯỚC
│          indexer + sweep + withdraw builder  [đội backend]  — Rust, Strata in-process
├─ Tầng 2  pot ZK + nullifier-SMT validator  [đội on-chain, theo public-input contract]
│          circuit egg-zk:withdraw + prover + verifier Aiken  [VeData/Glint]
└─ Tầng 3  viewing-key grant (chế-độ-1)  [đội backend+PhoenixKey]  · proof_only (chế-độ-2)  [VeData]
```

> **Tầng 0 = L3.** `did_subaddr.ak` là validator địa chỉ riêng tư mà `PhoenixKey-Rebirthme-Tech.md`
> gọi **L3** (nhất quán L1/L2/L3). Cùng một file, cùng một cơ chế — chỉ khác tên gọi theo ngữ cảnh
> tài liệu. `did_pool` (Tầng 1) là pool kế toán chung, không phải một tầng địa chỉ.

Luồng giá trị (Tầng 1): `Ví Phượng hoàng per-DID (công khai) → Sweep crank → Pool UTxO (did_pool) →
Withdraw (chủ DID ký) → withdraw_target`. Explorer chỉ đọc `PoolDatum` (tổng + hash), không field
per-DID. Indexer off-chain giữ cây MST đầy đủ + gated-read fail-closed cho viewing-key.

**Ranh giới cứng (I-EGG-23):** PhoenixKey giữ **pool/pot validator + datum + redeemer + min-UTxO +
recovery + UX**. VeData/Glint cấp **circuit + prover + verifier + ceremony + evidence**, KHÔNG giữ
tiền. LampNet/Strata giữ **primitive MST/MMR**. Easteregg **consume**, không viết lại primitive.

---

## 2. Bất biến kiến trúc (load-bearing)

- **I-EGG-1..9 (SHIELD)** — cưỡng chế trong `did_pool.ak`. Xem `-Math.md §4.1/§5`.
- **I-EGG-4' (G3 per-pair, 🔴):** Sweep phải bind mỗi input từ ĐÚNG addr DID_i khớp ĐÚNG amount_i.
  **CHẶN-MERGE nếu hở** — cài NGAY từ bản đầu, không "sửa sau".
- **I-EGG-11/13 (Tầng 0):** mỗi sub một payment credential (unlinkable khi nhận); backend chặn
  publish `commit_i`/addr vào resolver.
- **I-EGG-24 (MODE-SAFETY):** không mode nào (M0–M3) rút được tài sản ngoài chủ khoá / quorum recovery.
- **I-EGG-25 (EXT-QUARANTINE):** adapter bên thứ-3 nối qua escrow timelock-reclaim, không chạm core.

---

## 3. Datum / Redeemer — khuôn CBOR (Tầng 1, build-ready — Shielded-Custody §3)

### 3.1 PoolDatum (on-chain, per shard — thứ tự field CỐ ĐỊNH)
```
PoolDatum {
  pool_total : Value,   // [0] MUST == mst_root.sum (I-EGG-1)
  mst_root   : Hash32,  // [1] Merkle-Sum root {DID→balance}
  epoch      : Int,     // [2] monotonic +1 mỗi tx (I-EGG-2)
  log_root   : Hash32,  // [3] MMR root nhật ký (I-EGG-3, append-only)
  leaf_count : Int,     // [4] = mst_root.count
}
```
> On-chain KHÔNG BAO GIỜ có field per-DID. Chỉ 3 hash/số + `pool_total`. PII/plaintext không lên chain.

### 3.2 Leaf commitment (off-chain, cây MST)
```
leaf_bytes(DID, balance, salt) = DID_bytes ‖ salt         // salt 32-byte, đổi mỗi update
leaf_node = SumTree::leaf_node(leaf_bytes, balance)
          = Node{ hash=H_dom(TAG_LEAF, LEAF_PREFIX‖balance_be‖leaf_bytes), sum=balance, count=1 }
```
Cây: `SumTree::<Blake3Hasher>::build(&[(leaf_bytes,balance)…])`; `root()`=`mst_root`;
`root_node().sum`=`pool_total`; `.count`=`leaf_count`. Khớp `sumtree.rs`.

### 3.3 Redeemer `did_pool` — 3 case hiện có (+ đề xuất G2/G4)
```
Redeemer =
  | Sweep    { deposits:[(DID,amount,new_salt)], updates:[MstUpdateProof] }   // bot crank permissionless
  | Withdraw { DID, amount, sig, old_proof:MstProof, new_salt, withdraw_target }  // chủ DID ký
  | Reanchor { new_log_root }        // chỉ append log; value + mst_root BẤT BIẾN
  // ── đề xuất addendum (chờ duyệt) ──
  | Transfer { DID_from, DID_to, v, sig_from, proof_from, proof_to, new_salt_from, new_salt_to }  // G2
  | ReclaimCarrier { ... }           // G4 (hoặc gộp vào Reanchor)
```
`sig` = Ed25519 authority DID hiện tại trên `(DID ‖ amount ‖ new_salt ‖ withdraw_target ‖ epoch)`.
`verify_did_authority` đọc anchor TAAD qua **reference input** — KHỚP cách `did_payment` đọc anchor.

### 3.4 ViewingKeyGrant (off-chain, indexer ACL — Shielded-Custody §3.4)
```
ViewingKeyGrant { did, grantee_hash=H(grantee), viewing_key=HKDF(VaultKEK, "pool-view-v1", grantee_hash),
                  scope∈{balance_only, balance_and_history}, valid_until, granted_at, revoked }
```

---

## 4. Từng redeemer — điều kiện + shape tx + ai ký (đội on-chain cài đủ I-EGG-1..9)

Pseudocode logic đầy đủ: `Shielded-Custody §4 :143-174`. Điểm PHẢI chú ý khi cài:

- **G3 (🔴) — Sweep per-pair:**
  ```
  require ∀i: Σ inputs(from did_payment(DID_i).addr).value == amount_i
          ∧ updates[i] ghi vào ĐÚNG leaf(DID_i)      // KHÔNG chỉ ràng tổng
  ```
  Crank permissionless ⇒ ghi lệch nếu chỉ check tổng. **CHẶN-MERGE nếu hở.**
- **G1 (🔴) — Withdraw fee-split** (mẫu `relayer_fee` Tầng 2 ZK, tỷ lệ rời rạc `{0.5%,1%,2%}` chống
  fee-fingerprint):
  ```
  require Σ outputs(→ withdraw_target) == amount − cover_value(fee_asset, fee_amount)  // I-EGG-8'
  require Σ outputs(→ fee_recipient)   == fee_amount        // Feecover/relayer ứng ADA thật L1
  require cover_value(fee_asset, fee_amount) ≥ FEE_FLOOR    // sàn cứng chống lách phí
  // sổ pool: giảm đúng `amount` khỏi leaf (phần phí NẰM TRONG amount)
  ```
- **G2 (🟡) — Transfer nội bộ eAddr→eAddr** (như Reanchor: pool_total bất biến, đổi 2 lá):
  ```
  case Transfer(DID_from, DID_to, v, sig_from, proof_from, proof_to, new_salt_from, new_salt_to):
    require verify_did_authority(DID_from, sig_from, anchor(DID_from))  // CHỈ chủ-nguồn ký
    require MST_verify(mst_root, proof_from, leaf(DID_from, bal_from, salt_from))
    require MST_verify(mst_root, proof_to,   leaf(DID_to,   bal_to,   salt_to))
    require bal_from ≥ v
    root' = MST_update_2leaf(leaf(DID_from, bal_from−v, new_salt_from), leaf(DID_to, bal_to+v, new_salt_to))
    require new.pool_total == pool_total          // I-EGG-1 (KHÔNG đổi tổng)
    require value_in == value_out                 // như I-EGG-9
    require new.epoch == epoch+1 ; new.log_root == MMR_append(...)
  ```
  Receiver KHÔNG cần ký (như nhận tiền thường). Phí L1 cấn Feecover như G1.
- **G4 (🟡) — ReclaimCarrier:** tách sổ `pool_total` (gắn lá) vs `carrier_reserve` (min-ADA vận hành,
  không gắn lá); rút phần dư về Treasury, KHÔNG chạm lá. Áp lại mẫu "ngủ đông + reclaim" của TAAD anchor.
- **Double-satisfaction:** như lỗ Wakeme (đội on-chain bắt) — nếu batch nhiều pool-input, ép đếm input
  thuộc `Script(own_policy)` để cấm 1 output thoả nhiều input. Kiểm khi cài.

---

## 5. Luồng end-to-end

### 5.1 Off-chain Tầng 1 (Rust — đội backend, tái dùng Strata + rust_core)

| Thành phần | Nội dung | Nguồn |
|---|---|---|
| Indexer/Accountant | giữ cây MST đầy đủ `{DID→(balance,salt)}` + signed event log (MMR); sau mỗi tx tái dựng `mst_root`/`pool_total`/`leaf_count`, neo PoolDatum; API gated-read fail-closed | §5.1 |
| Sweep crank (permissionless) | quét addr Phượng hoàng balance>0 → build sweep tx batching theo cửa sổ → submit | §5.1 |
| Withdraw builder | `SumRangeProof` lá + chữ ký `TAAD_Key` (rust_core) + new_salt + `withdraw_target` một lần | §5.1 |
| Viewing-key | `HKDF(VaultKEK_A, "pool-view-v1", salt=H(grantee))`; ACL + `D9_REVOCATION` | §3.4,§6 |
| **G5 salt-recovery** | `salt_leaf(DID,epoch)=HKDF(Master_KEK,"easteregg/salt"‖DID‖epoch)` — tái sinh, không cần lưu `leaves.json` riêng | Gaps G5 |

**Tamper-evidence 3 lớp** (§5.2): on-chain anchor + signed event log MMR + client self-accounting.
Operator KHÔNG phải nguồn sự thật duy nhất. Tệ nhất = tranh chấp phân bổ, value KHÔNG mất.

### 5.2 Mode-transition (chuyển mode an toàn — PrivacyModes §B)
```
M0 ↔ M1  : gửi vào / rút khỏi did_pool — tx thường, 1 chữ ký.
M1 → M1+ : tạo did_subaddr mới cho nguồn thu mới (địa chỉ cũ giữ nguyên).
M1 → M2  : NÂNG = re-deposit có chủ đích (rút Tầng 1 → nạp pool ZK theo denomination).
M2 → M1  : HẠ = withdraw ZK về địa chỉ đích — user hiểu đây là MỘT LẦN REVEAL (đích rút lộ).
```
- **I-MODE-SWITCH-1:** chuyển mode CHỈ qua deposit/withdraw có chủ đích của chủ khoá — không đường
  "chuyển ngầm" (kể cả operator/governance).
- **I-MODE-SWITCH-2:** UI nêu hệ quả riêng tư của bước chuyển TRƯỚC khi ký (nâng M2: cold-start K_min
  + recovery trade-off; hạ M2: reveal đích rút).
- **I-MODE-HONEST:** UI hiển thị đúng bảng LỘ/ẨN của mode; M2 công bố `k'` ước lượng, CẤM dùng `k` đếm;
  không mode nào quảng cáo "vô hình" trừ M3-adapter với đầy đủ cảnh báo.

---

## 6. API backend (tham chiếu; prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`)

- Resolver trả `mst_root`/`pool_total`/`epoch`/`log_root`/`leaf_count` cho client kiểm (public).
- Gated-read (viewing-key): trả `{leaf, sum_range_proof}` của MỘT DID nếu viewing-key hợp lệ; DENY +
  audit-log nếu không (fail-closed, mẫu Query §F1/§F4).
- Revocation: `salt-rotate` (Withdraw+Redeposit 0-amount) + `valid_until` point-in-time
  (`did_active`/`key_authorized`, Contract §2.1) + `D9_REVOCATION` pubsub (Query §F8, TTL ≤5s).
> Endpoint cụ thể do đội backend chốt trong `PhoenixKey-API-Catalog.md`. Chưa có endpoint Easteregg (0 backend).

---

## 7. Ranh giới giao việc

| Bên | Việc | Thứ tự ưu tiên |
|---|---|---|
| **đội on-chain** | `did_pool.ak` (I-EGG-1..9, Aiken v1.1.21) + G3 per-pair (I-EGG-4') + G1 fee-split · `did_subaddr.ak` (Tầng 0, DEP-2) · Tầng 2: pot/nullifier-SMT validator theo public-input contract ZK-Math §C.6 | did_pool: ưu tiên cao nhất · did_subaddr: sau khi chốt DEP-2 · Tầng 2: sau gate B7–B9 |
| **đội backend** | Indexer/Accountant (MST+MMR Strata in-process) · sweep crank · withdraw builder · resolver + viewing-key ACL + chặn I-EGG-13 · G5 salt-from-KEK (rust_core HKDF) · golden vector + red-team §9.2 | song song Phase 1 với đội on-chain |
| **VeData/Glint (ZK)** | circuit Groth16 `egg-zk:withdraw` (Tầng 2) + `proof_only` (Tầng 3) · prover off-chain · verifier Aiken TỰ VIẾT + AUDIT · ceremony PPoT + phase-2 MPC ≥2 bên · evidence Stamp/Mosaic · gated-read Query | theo bộ Easteregg-ZK, độc lập tiến độ với Tầng 1 |
| **LampNet/Strata** | `sumtree.rs`/`mmr.rs`/`hash.rs` — primitive MST/MMR | Easteregg consume, không viết lại |
| **Midnight** | — RỜI critical-path (Tầng 2 L1 thay vai flow-privacy) | chỉ xét lại nếu cần asset Midnight-native |

### 7.1 Tầng 0 — did_subaddr (= địa chỉ L3, đội on-chain, DEP-2)
```
commit_i   = blake2b_256( blake2b_256(DID) ‖ tag_i )      // tag_i từ HKDF(Master_KEK)
pay_sub(i) = Script( hash( did_subaddr(anchor_policy, commit_i) ) )
```
Mỗi sub một payment credential → unlinkable khi NHẬN (I-EGG-11). Backend chặn publish vào resolver
(I-EGG-13). Trước khi có `did_subaddr.ak`: MVP dùng PB "riêng tư mức-resolver" (cùng `pay_did`, không publish).

### 7.2 Tầng 2 — ZK overlay (VeData/Glint; PhoenixKey giữ pot)
- **Public-input contract (interface CỨNG)** — đội on-chain cài pot/nullifier-SMT validator theo `ZK-Math §C.6`:
  deposit per-commitment binding 1-1 (CẤM aggregate-then-check, redeemer output-index tagging); withdraw
  `Verify(vk,I,π) ∧ ν∉SMT@tip ∧ KMIN fail-closed ∧ fee rời-rạc ∧ bind receiveAddr+fee`.
- **Groth16 CHỐT:** 2.652B lõi / 2.842B deploy / 2.524B marginal < 10B; N_batch_max=3. Halo2 LOẠI
  (6–15B). Cost-model embed Aiken v1.1.21, còn `[NEEDS uplc-Conway-confirm]`.
- **Verifier Aiken: TỰ VIẾT + AUDIT trước mainnet.** `VeDataIO/Code/groth16/lib/egg/groth16.ak` chỉ là
  scratch; `Modulo-P ak-381` KHÔNG dùng thẳng. **Ceremony:** PPoT phase-1 + phase-2 MPC ≥2 bên RIÊNG
  mỗi circuit; KHÔNG share Glint universal-SRS. `I-EGG-18`: đóng băng circuit sớm.

### 7.3 Ranh giới Feecover per-mode (PrivacyModes §C — đính chính nội bộ 2026-07-06)

| Phạm vi | Ai trả phí mạng | Vì sao |
|---|---|---|
| M0 / M1 / M1+ (mọi tx) | **Feecover** (DID-gated bình thường) | danh tính vốn gắn DID — gate không rò thêm |
| M2 — deposit | **Feecover** được | phía nạp vốn CÔNG KHAI trên L1 |
| M2 — **withdraw** | **CẤM Feecover-gate** (I-EGG-26 FEECOVER-BOUNDARY) — dùng (a) user tự trả, hoặc (b) **relayer-lane** fee rời rạc `pct∈{0.5%,1%,2%}` bind-in-PI | Feecover resolve DID point-in-time ⇒ tx trả phí lộ DID, phá unlinkability vừa mua bằng Groth16 |

Relayer-lane CÓ THỂ do chính PhoenixKey/Feecover vận hành (miễn về mật mã là lane KHÔNG-DID-gate:
không resolve DID, thu fee rời rạc từ chính value withdraw, mở permissionless — user luôn tự-submit được).

### 7.4 External-Adapter (PrivacyModes §D — bên thứ-3 có trả phí, ngủ đông)

| # | Nhu cầu | Bên thứ 3 | Trạng thái |
|---|---|---|---|
| EXT-1 | Trả gas hộ | KHÔNG còn bên thứ-3 — Feecover nội bộ (§7.3) | Nội bộ hoá xong |
| EXT-2 | Attestation bởi bên uy tín | Attestor ngoài | Qua Tầng 3 (verify proof_only/viewing-grant rồi ký) — hệ không đổi |
| EXT-3 | Existence-hiding (M3) | **Midnight** (hoặc tương đương) | **ADAPTER NGỦ ĐÔNG** — ngoài critical-path; chỉ mở khi qua cửa quyết định |

4 bất biến adapter (I-EGG-25 + phái sinh): **INV-EXT-1 quarantine** (escrow timelock-reclaim, mất
adapter ≠ mất tài sản); **INV-EXT-2 user-pays** (không trợ giá chéo Treasury); **INV-EXT-3 disclosure
nội bộ** (nghĩa vụ minh bạch qua Tầng 3, không phụ thuộc disclose bên thứ-3); **INV-EXT-4 cửa quyết
định đo được** (EXT-3 chỉ kích hoạt khi có cầu trust-minimized / relayer-set t-of-n + tooling ổn định
+ cầu người dùng thật đo được). Trung thực EXT-3: mọi đường sang Midnight 2026 tạo điểm biết hết
(relayer thấy mapping lock↔mint) — chỉ dịch niềm tin, chưa xoá → M3 không vào critical-path.

---

## 8. Thứ tự deploy + phụ thuộc chặn

| Blocker | Chủ | Chặn |
|---|---|---|
| B1 `did_pool.ak` build (I-EGG-1..9) | đội on-chain | Toàn Tầng 1 |
| B2 G3 per-pair bind (I-EGG-4') | đội on-chain | Sweep (**chặn-merge**) |
| B3 G1 fee-split + `fee_asset`/oracle | đội on-chain+backend | User tự rút |
| B4 indexer/sweep/withdraw | đội backend | Cây MST/proof/gated-read |
| B5 G5 salt-from-KEK | đội backend+on-chain | Kẹt tiền khi mất witness |
| B6 `did_subaddr.ak` | đội on-chain | Tầng 0 unlinkable (DEP-2) |
| B7 verifier Aiken audit + ceremony | VeData | Tầng 2 |
| B8 uplc-Conway-confirm 2.842B | VeData | Lock N_batch Tầng 2 |
| B9 OQ-EGG-5/8/9 | VeData | ước `k'`, chốt N_batch, threshold-decrypt |

Thứ tự: **B1+B2+B3+B4+B5 (Tầng 1) → B6 (Tầng 0) → B7+B8+B9 (Tầng 2)**. OQ-EGG-1 ✅ RESOLVED (Groth16).

---

## 9. Test / evidence

**Điều kiện tiên quyết:** `did_pool.ak` build (B1) + G3 vá (B2) + off-chain (B4) + G1 fee-split (B3).

| Bước | Nội dung | Kỳ vọng (evidence) |
|---|---|---|
| 1 | `aiken build` + `aiken blueprint apply` did_pool (K=1) → deploy Preview | script hash + address |
| 2 | mint CARP test → gửi 2–3 ví Phượng hoàng (DID khác nhau) | explorer thấy tx nhận + amount |
| 3 | Sweep gom vào 1 Pool UTxO | addr Phượng hoàng ≈ 0; Pool value to; datum KHÔNG có field per-DID (screenshot) |
| 4 | Prove qua viewing-key (ngân hàng giả) | `verify_sum_range(mst_root_onchain,…)==⊤`; không đọc lá khác |
| 5 | Reveal grant `scope=balance_and_history`, `valid_until` | auditor thấy timeline; sau `valid_until` → DENY |
| 6 | Withdraw (proof + sig authority + new_salt + fee-split) | tx pass; `pool_total` giảm đúng amount; phí cấn đúng |
| 7 | **Red-team fail-closed (BẮT BUỘC)** | 4 mục dưới REJECT |

**Red-team bước 7 (đủ 4 mục có evidence):**
- Sửa lá off-chain → prove ⇒ **proof-fail** (I-EGG-6).
- Withdraw KHÔNG chữ ký authority ⇒ **reject** (I-EGG-5) — chứng minh operator không rút được.
- `pool_total > mst_root.sum` ⇒ **reject** (I-EGG-1).
- Submit datum `epoch` cũ ⇒ **reject** (I-EGG-2).
- **(thêm khi có G3)** Sweep ghi lệch amount giữa 2 DID cùng batch ⇒ **reject** (I-EGG-4' per-pair).

**PASS Phase 1:** 7 bước có output thật (tx-hash + explorer screenshot bước 3) + red-team đủ mục.
**Phase 2 gate:** sharding K>1; proof-only ZK; multi-indexer đối chiếu root; deposit commitment +
withdraw Groth16 → preprod E2E (double-withdraw REJECT, solvency giữ, K_min enforce).

### 9.1 GO / NO-GO theo tầng (nguồn gaming-eval — anti-gaming)

| Tầng | Verdict |
|---|---|
| **Tầng 0 (eAddr)** | **NO-GO production** tới khi `did_subaddr.ak` build+test. MVP "riêng tư mức-resolver" chấp nhận cho demo. |
| **Tầng 1 (MST custody)** | **NO-GO production.** GO cho build+test Preview khi B1+B2+B3+B5 land + 7-bước + red-team pass với tx-hash thật. **VG-E1/G3 chặn-merge.** |
| **Tầng 2 (ZK)** | **NO-GO** tới khi B7+B8+B9 land + ceremony phase-2 + preprod E2E. Marketing dùng `k'` KHÔNG dùng `k`. |
| **Tầng 3 (disclosure)** | **GO chế độ-1** (view-key HKDF + fail-closed ACL) cùng Tầng 1. **NO-GO chế độ-2** (proof-only) tới khi ZK land. Audit-lane nếu bật: PHẢI threshold t-of-n (I-EGG-21). |

**Chốt tổng:** NO-GO production toàn module. Chỉ GO cho build+test Preview Tầng 1 + Tầng 3 chế độ-1.

---

## 10. Ghi trung thực (blocker + gap — nguyên tắc)

- **Neo pseudocode.** Mọi neo:dòng trong bản này trỏ spec, không phải Aiken đã build — đối chiếu
  lại khi validator thật xuất hiện. Verifier Aiken Tầng 2 là hạng mục độc lập (số 2.842B = đo
  VeData, độc lập với code PhoenixKey).
- **G1🔴 + G3🔴 ưu tiên** (chặn user tự rút / chặn-merge Sweep). G2/G4/G5 (🟡) enhancement/vá.
- PoC Preview minh hoạ ẩn số dư + gated-proof (3 tx-hash) nhưng **KHÔNG thay validator SHIELD-5**
  (sweep vào ví thường) → chưa chứng minh operator-không rút. "PoC ≠ sản phẩm".
- **OQ mở:** OQ-EGG-2/3/4/5/6/7/8/9 + uplc-Conway-confirm. OQ-EGG-1 ✅ RESOLVED (Groth16).
- Phát hiện lỗi validator/backend → báo maintainer / Issue giao đội on-chain/backend, KHÔNG tự sửa (ranh giới tài liệu này).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## Nguồn

- Gộp từ nguồn thiết kế nội bộ (không công khai) + `-Feat-Math.md` (§7 phụ thuộc),
  `-Gaps-Addendum-Feat.md` (G1–G5 + đề xuất vá), `-PrivacyModes-Addendum-Feat.md`
  (§A Mode Matrix, §B transition, §C Feecover-boundary, §D External-Adapter, §E demo),
  `PhoenixKey-Shielded-Custody-Feat-Math.md §3–§5,§9` (Tầng 1 schema + test plan),
  `Easteregg/Easteregg-ZK-{Math,Tech,Exec-Spec}.md` (Tầng 2 ZK), `_easteregg-gaming-evaluation.md`
  (GO/NO-GO, VG-E1..E10).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
