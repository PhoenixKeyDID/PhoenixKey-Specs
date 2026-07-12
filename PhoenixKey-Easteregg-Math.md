# PhoenixKey — Easteregg · Đặc tả TOÁN hình thức (Math, cho AUDITOR)

> Module: **Easteregg** (các mức riêng tư của ví Phoenix) · Loại doc: Math (toán hình thức, cho auditor) · Ngày: 2026-07-09
> Đối tượng đọc: auditor. Định nghĩa hình thức 4 mức riêng tư, bất biến neo dòng, chứng minh
> value-conservation (không in tiền ảo / không drain) + unlinkability (phát biểu chính xác + giới hạn).
> Tài liệu cùng bộ: `-Vi-Feat.md` · `-Tech.md` · `-Exec.md`. Math canonical toàn hệ: `PhoenixKey-Math.md`.
>
> **Phân loại (quyết định maintainer 2026-07-09):** ví CHỈ 2 LOẠI — `Phoenix`/`Standard`
> (`PhoenixKey-Rebirthme-Math.md §2.2`). Easteregg KHÔNG phải loại ví thứ ba — là các **mức
> riêng tư của ví Phoenix** (mô hình 2 tầng: loại ví × mức riêng tư). Tên gọi "Tầng 0/1/2/3" bên
> dưới là tên **mức riêng tư**, không phải tầng của một loại ví riêng. "Tầng 0" (địa chỉ riêng,
> `did_subaddr`) là CÙNG một validator với **L3** trong `PhoenixKey-Rebirthme-Math.md §2.2` —
> dùng "L3" làm tên canonical cho tầng địa chỉ; "Tầng 0" giữ lại làm tên nội bộ ngữ cảnh Easteregg,
> luôn cross-reference L3. `did_pool` (Tầng 1) KHÔNG phải một loại địa chỉ — là pool kế toán dùng
> chung (Merkle-Sum-Tree), nhận tiền qua Sweep từ địa chỉ `did_payment`/`did_subaddr`, KHÔNG nhận
> trực tiếp.
>
> **⚠ QUY ƯỚC NEO PSEUDOCODE.** Mọi "neo file:dòng" dưới đây trỏ **PSEUDOCODE trong spec** —
> auditor PHẢI đối chiếu lại với Aiken thật khi validator được build (xem tiến độ ở STATUS). Các
> số đo ZK (2.842B…) là đo THẬT từ bộ Easteregg-ZK (VeData), độc lập với code PhoenixKey. Mọi
> định lý dưới đây thành lập **với điều kiện** validator được cài đúng bất biến mô tả ở §4.
>
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## 1. Ký hiệu (notation)

### 1.1 Tầng 1 — pool custody (Merkle-Sum-Tree)

| Ký hiệu | Định nghĩa | Nguồn |
|---|---|---|
| `DID` | định danh phi tập trung của chủ lá | Shielded-Custody §3.2 |
| `bal(DID)` | số dư lá của DID trong cây MST | `:102` |
| `salt(DID)` | 32-byte ngẫu nhiên mỗi update (che brute-force) | `:102,108` |
| `leaf(DID,bal,salt)` | `Node{hash=H_dom(TAG_LEAF, LEAF_PREFIX‖bal_be‖DID‖salt), sum=bal, count=1}` | `:104-106` |
| `mst_root` | Merkle-Sum root của cây `{DID→bal}`; mang `(hash,sum,count)` | `:63,90` |
| `pool_total` | tổng value khoá trong Pool UTxO | `:62,89` |
| `epoch` | bộ đếm monotonic, +1 mỗi tx | `:64,91` |
| `log_root` | MMR root nhật ký sự kiện (append-only) | `:66,93` |
| `leaf_count` | `= mst_root.count` (số DID trong shard) | `:94` |
| `PoolDatum` | `{pool_total, mst_root, epoch, log_root, leaf_count}` | `:89-95` |
| `shard(DID)` | `blake3(DID) mod K` (K=1 ở Preview) | `:259` |

### 1.2 Tầng 0 — địa chỉ riêng (`did_subaddr`) = địa chỉ L3 của ví Phoenix

> `did_subaddr` là MỘT validator duy nhất. Rebirthme gọi tầng địa chỉ này **L3** (tên canonical,
> nhất quán L1/L2/L3 — xem `PhoenixKey-Rebirthme-Math.md §2.2`); Easteregg gọi nó "Tầng 0" trong
> ngữ cảnh mức riêng tư nội bộ module này. KHÔNG phải hai validator/hai cơ chế khác nhau.

| Ký hiệu | Định nghĩa | Nguồn |
|---|---|---|
| `commit_i` | `blake2b_256(blake2b_256(DID) ‖ tag_i)` | Feat-Math §1 `:55` |
| `pay_sub(org,i)` | `Script(hash(did_subaddr(anchor_policy, commit_i)))` | `:55` |
| `tag_i` | sinh deterministic từ `Master_KEK` qua HKDF (không secret mới) | `:56` |

### 1.3 Tầng 2 — ZK overlay (opt-in; nguồn VeData Easteregg-ZK)

| Ký hiệu | Định nghĩa | Nguồn |
|---|---|---|
| `commitment` | Pedersen commitment (`ρ` public tại deposit) | ZK-Math §C.1 |
| `ν` (ZK-nullifier) | `PRF_sk(ρ)`, domain-tag `"egg-nullifier"` | §C.2 |
| `A_d` | anonymity-set; `k = \|A_d\|` (đếm), `k' = 2^{H∞}` (hiệu dụng) | §A.3–A.5 |
| `K_min` | ngưỡng anonymity-set enforce on-chain (=32 sơ bộ) | §A.4 |
| `eADA` | số kế toán TRONG pot ZK (no-policy, non-transferable, tiêu là mất) | Feat-Math §3 `:111` |

---

## 2. Định nghĩa hình thức các hàm/tập

**Cây Merkle-Sum (Strata `sumtree.rs`).** Mỗi node mang bộ ba `(hash, sum, count)`. Lá cho DID:
`leaf(DID,bal,salt)` như §1.1. `mst_root.sum = Σ_DID bal(DID)` theo định nghĩa Merkle-Sum;
`verify_sum_range(mst_root, proof, leaf)` cho phép kiểm MỘT lá + tổng mà không lộ lá khác
(field-proof). `salt` 32-byte ⇒ commitment lá không brute-force ngược ra `bal`, đổi mỗi update ⇒
không tái nhận diện.

**Nhật ký MMR (Strata `mmr.rs`).** `log_root = MMR_append(log_root, event)` append-only; dup-leaf
guard (CVE-2012-2459) + commit số lá `n` chống rollback.

**3 redeemer Tầng 1 (hiện có, `did_pool` pseudocode Shielded-Custody §3.3, §4):**
```
Redeemer =
  | Sweep    { deposits:[(DID,amount,new_salt)], updates:[MstUpdateProof] }   // bot crank permissionless
  | Withdraw { DID, amount, sig, old_proof:MstProof, new_salt, withdraw_target }  // chủ DID ký
  | Reanchor { new_log_root }        // chỉ append log; value + mst_root BẤT BIẾN
```
`sig` = Ed25519 authority DID hiện tại trên `(DID ‖ amount ‖ new_salt ‖ withdraw_target ‖ epoch)`;
`verify_did_authority` đọc anchor TAAD qua reference input — KHỚP cách `did_payment` đọc anchor.

**Tầng 2 (ZK, nguồn Easteregg-ZK-Math).** Pedersen commitment (ρ public tại deposit, an toàn vì
nullifier chỉ dựa `sk`); ZK-nullifier `ν = PRF_sk(ρ)`; commitment-tree (historical root chấp nhận)
+ nullifier-SMT toàn cục MONOTONE chỉ-insert (current-tip BẮT BUỘC, không reset/archive);
denomination set khử amount-fingerprint.

---

## 3. Bất biến sổ sách LÕI (value-conservation)

**Conservation Tầng 1:** trong mọi tx hợp lệ, `pool_total = Σ_DID bal(DID)` VÀ value UTxO thật khớp
`pool_total`. Không tx nào tạo/huỷ value ngoài phép cộng (Sweep) / trừ (Withdraw) đúng lá của
người ký. Cơ chế cưỡng chế = SHIELD-1 (§4). Chứng minh: §7 Định lý E1.

**Conservation Tầng 2:** `Σ eADA rút ≤ Σ eADA nạp` mọi lúc (I-EGG-16). Deposit per-commitment
binding 1-1 (CẤM aggregate-then-check). Chứng minh: ZK-Math §D.3, §D.6.

---

## 4. Bảng bất biến `I-EGG-n` (mã hoá + mô tả + neo)

> Mã chuẩn hoá cho module: `I-EGG-<n>`. Các bất biến Tầng 1 giữ tên lịch sử **SHIELD-1..9** (=
> `I-EGG-1..9`, alias song hành để khớp `Shielded-Custody §4`). Neo `Shielded-Custody :NNN` là
> PSEUDOCODE. Tái dùng primitive từ `PhoenixKey-Math.md`: MST/MMR (Strata), Master_KEK §6, TAAD §10.

### 4.1 Tầng 1 — SHIELD-1..9 (nguồn Shielded-Custody §4 `:180-188`)

| ID (= alias) | Bất biến | Neo (pseudocode) | Ý nghĩa an ninh |
|---|---|---|---|
| **I-EGG-1** = SHIELD-1 | `pool_total == mst_root.sum` mọi tx; **VÀ** value UTxO thật khớp | `:150,163,180` | Sum-invariant: operator KHÔNG in tiền ảo/giấu bớt tổng |
| **I-EGG-2** = SHIELD-2 | `epoch` monotonic `+1` mỗi tx | `:152,165,173,181` | Chống replay + rollback |
| **I-EGG-3** = SHIELD-3 | `log_root == MMR_append(...)` append-only | `:153,166,182` | Nhật ký không xoá được (dup-leaf guard `mmr.rs`) |
| **I-EGG-4** = SHIELD-4 | value vào pool `== Σ deposits`, input từ addr DID | `:147,183` | Sweep không bơm khống |
| **I-EGG-5** = SHIELD-5 | Withdraw cần chữ ký authority DID hiện tại (đọc anchor TAAD) | `:157,184` | **Operator KHÔNG rút được tiền**; sống qua rotate |
| **I-EGG-6** = SHIELD-6 | MST field-proof lá cũ khớp root (`verify_sum_range`) | `:159,185` | Không rút lá không tồn tại / số dư giả |
| **I-EGG-7** = SHIELD-7 | `old_bal ≥ amount` | `:160,186` | Không rút âm/vượt |
| **I-EGG-8** = SHIELD-8 | `Σ outputs(→ withdraw_target) == amount` | `:164,187` | Chi đúng số |
| **I-EGG-9** = SHIELD-9 | Reanchor: `value_in == value_out`, `mst_root` bất biến | `:171,188` | Cập nhật log KHÔNG chạm tài sản |

### 4.2 Tầng 0 — địa chỉ (= L3 `did_subaddr`; nguồn MultiAddress §M.8, chưa có validator)

| ID | Bất biến | Ý nghĩa |
|---|---|---|
| **I-EGG-10** (I-ADDR-CTRL) | chi mọi sub qua authority anchor DID gốc + controller ký | chủ DID chi được tất cả sub |
| **I-EGG-11** (I-ADDR-UNLINK) | mỗi sub một payment credential khác → on-chain KHÔNG nhóm khi NHẬN | unlinkable phía nhận |
| **I-EGG-12** (I-ADDR-ORGALL) | authority quy về cùng anchor DID gốc | sống qua rotate/recovery |
| **I-EGG-13** (I-ADDR-PRIV) | TUYỆT ĐỐI KHÔNG publish `commit_i`/addr vào `service[]`/resolver | ánh xạ chỉ ở kho riêng chủ DID |

### 4.3 Tầng 2 + Tầng 3 + xuyên tầng (nguồn ZK-Math + Feat-Math §5)

| ID | Bất biến | Neo |
|---|---|---|
| **I-EGG-14** (ONCHAIN-CREDIT) | credit trên chain chỉ từ deposit hợp lệ (per-commitment binding 1-1, CẤM aggregate-then-check) | ZK-Math §C.6, §D.6 |
| **I-EGG-15** (KMIN) | `k ≥ K_min` fail-closed on-chain | §A.4 |
| **I-EGG-16** (SOLVENCY) | `Σ eADA rút ≤ Σ eADA nạp` mọi lúc | §J, §D.3 |
| **I-EGG-17** (NULLIFIER) | `ν ∉ nullifier-SMT@tip` (current-tip BẮT BUỘC) chống double-withdraw | §C.2, §D.4' |
| **I-EGG-18** (CIRCUIT-FREEZE) | circuit `egg-zk:withdraw` đóng băng sớm; ceremony PPoT+phase-2 MPC ≥2 bên RIÊNG mỗi circuit | §L.1 |
| **I-EGG-19** (UPGRADE) | đổi circuit nằm trong upgrade-path có time-lock; SRS mới = ceremony mới, KHÔNG bypass nullifier/solvency/binding | §D.8, §I |
| **I-EGG-20** (VIEW⊥SPEND) | viewing-key CHỈ đọc, KHÔNG tiêu; compromise = privacy-leak, KHÔNG drain | §C.7 |
| **I-EGG-21** (DISCLOSURE-THRESHOLD) | audit-lane mở phải k≥2 threshold (không single-authority de-anon) | §K.3 |
| **I-EGG-22** (RECOVERY-SYMMETRY) | Tầng 1 tài sản PHẢI phục hồi thuần bằng cơ chế DID (không secret ngoài Master_KEK); Tầng 2 PHẢI hiển thị trade-off recovery trước opt-in | Feat-Math §5 `:137` |
| **I-EGG-23** (LAYER-BOUNDARY) | Easteregg consume MST/ZK/gated-read — KHÔNG tự thiết kế lại; VeData/Glint KHÔNG giữ custody | Feat-Math §5 `:138` |
| **I-EGG-24** (MODE-SAFETY) | ở MỌI mode M0–M3, không bên nào ngoài chủ khoá (hoặc quorum recovery hợp lệ) rút được tài sản; mode chỉ đổi *mức ẩn*, không đổi *quyền sở hữu* | PrivacyModes §B |
| **I-EGG-25** (EXT-QUARANTINE) | bên thứ-3 (adapter) KHÔNG chạm immutable-core; nối qua escrow timelock-reclaim: bên thứ-3 chết ⇒ chủ DID rút thẳng L1 sau T. Mất adapter = mất tiện ích/riêng tư, KHÔNG mất tài sản | PrivacyModes §D INV-EXT-1 |
| **I-EGG-26** (FEECOVER-BOUNDARY) | M2-withdraw CẤM Feecover-gate (resolve DID point-in-time ⇒ lộ DID, phá unlinkability Groth16 vừa mua); phí Tầng 2 qua (a) user tự trả hoặc (b) relayer-lane fee rời rạc bind-in-public-input, KHÔNG-DID-gate | PrivacyModes §C, Tech §7.3 |

---

## 5. Mệnh đề ép từng redeemer (đối chiếu pseudocode — guard load-bearing)

**Sweep** (I-EGG-4 + I-EGG-1 + I-EGG-2 + I-EGG-3):
```
require Σ inputs(from did_payment(DID_i) addr) == Σ amount_i           // I-EGG-4
root' = fold deposits qua MST_update(mst_root, updates)                 // bal_i += amount_i
require new.pool_total == pool_total + Σ amount_i == root'.sum          // I-EGG-1
require new.epoch == epoch + 1 ; new.log_root == MMR_append(...)        // I-EGG-2/3
```
> **⚠ LỖ G3 (🔴):** pseudocode chỉ ràng TỔNG, CHƯA ràng per-pair `(DID_i ↔ amount_i)`. Vá bắt buộc
> khi cài (I-EGG-4'): `∀i: Σ inputs(from did_payment(DID_i).addr) == amount_i ∧ updates[i]→leaf(DID_i)`.
> Crank permissionless ghi lệch nếu chỉ check tổng — xem §9 [CẦN CHỐT G3], §8 giả định.

**Withdraw** (I-EGG-5/6/7/8/1/2):
```
require verify_did_authority(DID, sig, anchor(DID))                     // I-EGG-5 operator không rút được
require MST_verify(mst_root, old_proof, leaf(DID, old_bal, old_salt))   // I-EGG-6
require old_bal ≥ amount                                                 // I-EGG-7
require new.pool_total == pool_total − amount == root'.sum              // I-EGG-1
require Σ outputs(to withdraw_target) == amount                         // I-EGG-8
```
> **⚠ LỖ G1 (🔴):** không tách phí. Vá (fee-split): `Σ out(→target) == amount − cover_value(fee_asset,
> fee_amount)` ∧ `Σ out(→fee_recipient) == fee_amount` ∧ `cover_value ≥ FEE_FLOOR` (sàn cứng chống
> lách phí). Phần phí NẰM TRONG `amount` giảm khỏi lá.

**Reanchor** (I-EGG-9): `new.mst_root == mst_root` ∧ `pool_total` bất biến ∧ `value_in == value_out`.

**Withdraw ZK Tầng 2** (I-EGG-14/16/17/15): `Verify(vk,I,π) ∧ ν ∉ SMT@tip ∧ k ≥ K_min (fail-closed)
∧ fee rời rạc ∧ bind receiveAddr+fee` (ZK-Math §C.6, §H — bind-in-public-input chống front-run).

---

## 6. Không gian trạng thái + đồ thị chuyển

Trạng thái Tầng 1 = `PoolDatum{pool_total, mst_root, epoch, log_root, leaf_count}`. Chuyển:
```
  ──Sweep──▶  (pool_total↑, mst_root', epoch+1, log_root', leaf_count↑)
  ──Withdraw▶ (pool_total↓, mst_root', epoch+1, log_root')
  ──Reanchor▶ (pool_total=, mst_root=, epoch+1, log_root')
  ──Transfer▶ (pool_total=, 2 lá đổi, epoch+1, log_root')     // G2 đề xuất — value bất biến
```
`epoch` strict-tăng ⇒ đồ thị không có chu trình có lợi (I-EGG-2). Mode-transition M0↔M1↔M2 CHỈ qua
deposit/withdraw có chủ đích của chủ khoá (I-MODE-SWITCH-1) — không đường "chuyển ngầm".

---

## 7. Định lý an toàn + chứng minh phác thảo

Giả định primitive Strata (`sumtree.rs::verify_sum_range`, `mmr.rs`) đúng (đã test ở LampNet) và bộ
ZK (VeData) đúng như đã audit. Các định lý CHỈ chứng minh phần Easteregg ghép lại — **với điều kiện
validator được cài đúng I-EGG-1..9** (chưa xảy ra).

### Định lý E1 (Value-conservation Tầng 1 — không in tiền ảo, không drain)
**Mệnh đề.** `pool_total = Σ_DID bal(DID)` mọi tx, không tx nào rút vượt số dư lá của người ký.
**Chứng minh.** (1) I-EGG-1 ép `pool_total == mst_root.sum` VÀ value thật khớp; vì `mst_root.sum =
Σ bal(DID)` (Merkle-Sum) ⇒ `pool_total = Σ bal(DID)`. (2) Sweep: I-EGG-4 ép value vào `== Σ amount_i`;
I-EGG-1 sau update giữ đẳng thức. (3) Withdraw: I-EGG-6 ép proof lá; I-EGG-7 `old_bal ≥ amount`;
I-EGG-8 chi đúng; I-EGG-1 `pool_total' = pool_total − amount`. (4) Reanchor: I-EGG-9 value bất biến.
∴ Không nhánh nào tạo/huỷ value ngoài cộng/trừ đúng lá. ∎
**Hệ quả (anti-drain operator).** Withdraw cần I-EGG-5 (chữ ký authority DID). Operator không có chữ
ký ⇒ không rút được lá nào; tệ nhất gây tranh chấp phân bổ (G3/E4), value KHÔNG rời pool.

### Định lý E2 (Anti-rollback / anti-replay)
I-EGG-2 (`epoch` +1) + I-EGG-3 (`log_root` MMR append-only, commit `leaf_count`) ⇒ mọi state có
`epoch` tăng nghiêm, log không xoá; submit datum `epoch` cũ ⇒ reject. ∎

### Định lý E3 (Unlinkability — phát biểu CHÍNH XÁC + giới hạn)
**Đạt được.** *Tầng 1:* explorer chỉ đọc `PoolDatum` (không field per-DID); `bal(DID)` ẩn dưới salt
⇒ **ẩn STOCK** với explorer. *Tầng 0:* mỗi sub payment credential khác ⇒ không nhóm khi NHẬN
(I-EGG-11). *Tầng 2:* Groth16 + nullifier ⇒ không ghép cặp nạp↔rút; operator-blind (ZK-Math §A.1).
**KHÔNG đạt (giới hạn, KHÔNG giấu).** *Tầng 1* không ẩn FLOW: timing + amount mỗi khoản NHẬN của addr
công khai lộ trước sweep; batching chỉ trộn khi mật độ đủ (thưa ⇒ anonymity ≈ 1); không ẩn với
operator (indexer giữ lá plaintext). *Tầng 0* lộ khi CHI (mở một `tag_i` + tham chiếu anchor DID).
*Tầng 2* LỘ ai nạp/ai nhận/amount(denomination)/tổng-pot; `K_min` chặn `k` (đếm) KHÔNG bảo đảm `k'`
(hiệu dụng) — self-fill là residual kinh-tế (OQ-EGG-5). ∎ (phát biểu — cấm marketing "vô hình")

### Định lý E4 (Recovery-symmetry — điều kiện + LỖ HỔNG G5)
**Mệnh đề.** Tầng 1 tài sản phục hồi thuần bằng cơ chế DID nếu (i) quyền chi đọc động authority hiện
tại qua anchor TAAD (I-EGG-5), và (ii) witness-data (salt) tái dựng được.
**Trạng thái.** (i) đạt theo thiết kế (giống `did_payment`). **(ii) CHƯA đạt** — G5: salt lưu off-chain
(`leaves.json`), mất → kẹt tiền (ĐÃ xảy ra trong PoC). Vá: `salt_leaf(DID,epoch) = HKDF(Master_KEK,
"easteregg/salt"‖DID‖epoch)` → tái sinh từ Master_KEK. **Định lý CHỈ thành khi G5 vá.** ∎ (điều kiện)

### Định lý ZK Tầng 2 (nguồn ZK-Math §D — VeData, tham chiếu, không chép lại)
- **D.1 Withdraw soundness:** không rút được nếu không sở hữu note.
- **D.2 Completeness (non-custodial):** owner LUÔN rút được; user luôn tự-submit được (chống censor).
- **D.3 Solvency:** pot luôn đủ backing (I-EGG-16).
- **D.4 / D.4' No-double-withdraw:** nullifier global-SMT current-tip (I-EGG-17); vá cross-epoch $0.
- **D.5 Withdraw circuit statement (soundness anchor).**
- **D.6 Deposit-binding [CRIT]:** nạp rút đối xứng mức verify (I-EGG-14).
- **D.7 Liveness-through-time:** completeness không suy giảm theo thời gian.
- **D.8 Invariant-preservation qua upgrade:** immutable-core (I-EGG-19).
- **G.3 (recovery):** PhoenixKey-một mình-derive `sk` = backdoor drain → **CẤM**; user chọn tier-1
  self-custody hoặc tier-2 k-of-n (k≥2, time-lock ≥7–14 ngày + veto).

---

## 8. Giả định tin cậy (ngoài phạm vi chứng minh)

- Primitive MST/MMR Strata đúng (kế thừa LampNet/Strata).
- Bộ ZK VeData (circuit + prover + verifier) đúng như đã audit — verifier Aiken phía PhoenixKey là
  thành phần độc lập cần tự-audit riêng; mọi định lý Tầng 2 là điều kiện trên đó.
- **Validator `did_pool`/`did_subaddr` phải cài ĐÚNG I-EGG-1..13.** Mọi định lý §7 là điều kiện
  trên giả định này — auditor đối chiếu khi Aiken thật xuất hiện.
- G3 per-pair binding được cài NGAY từ bản `did_pool.ak` đầu tiên (không "sửa sau").
- gated-read fail-closed + point-in-time revoke thuộc VeData Query §F1/§F4/§F8 (cross-repo, consume).

### Phân biệt crypto CHẶN vs residual kinh-tế/governance (nguồn gaming-eval)

| Vector | Bản chất | Crypto giải được? |
|---|---|---|
| in tiền ảo / giấu tổng (VG-E5) | I-EGG-1 sum-invariant + I-EGG-16 solvency | ✅ khi cài validator |
| double-withdraw (VG-E6) | global nullifier-SMT current-tip + Groth16 soundness | ✅ khi verifier audit + ceremony sạch |
| backdoor-drain sk (VG-E8) | k≥2 key-derivation (G.3 cấm single-party) | ✅ thiết kế cấm |
| drain qua viewing-key (VG-E9) | viewing⊥spending (I-EGG-20) | ✅ |
| sweep ghi lệch (VG-E1/G3) | per-pair binding I-EGG-4' | ✅ nguyên tắc (CHƯA cài) |
| lách phí (VG-E2/G1) | FEE_FLOOR sàn cứng | ✅ nguyên tắc (CHƯA cài) |
| K_min self-fill k≠k' (VG-E3) | — | ❌ residual kinh-tế (fee α + seed) / governance |
| ẩn STOCK không ẩn FLOW (VG-E7) | — | ❌ Tầng 1 (giới hạn L1); một phần Tầng 2 |
| privacy quá khứ sau revoke (VG-E9) | — | ❌ khoá đối xứng đã trao — chỉ cắt tương lai |
| anchor-giả PersonDID (VG-E10) | UniquenessThread (PA2) | 🟡 chờ đội khác |

---

## 9. [CẦN CHỐT] còn treo (blocker / dependency)

- **Điều kiện xác minh.** Auditor verify được I-EGG-1..9 khi có Aiken thật + golden vector khớp
  `sumtree.rs::verify_sum_range` + `mmr.rs::verify` (đội backend dựng). Mọi neo:dòng ở spec này
  trỏ pseudocode cho tới lúc đó.
- **[CẦN CHỐT G3]** per-pair binding Sweep (I-EGG-4') — HỎI đội on-chain trước, **chặn-merge** nếu hở.
- **[CẦN CHỐT G1]** fee-split + `fee_asset` + oracle (nếu LAMP) — nối Feecover.
- **[CẦN CHỐT G5]** MST-proof có đòi salt-ngẫu nhiên thật không (nếu không → HKDF-from-KEK thoả).
- **[ĐIỀU KIỆN Tầng 2]** verifier Aiken TỰ VIẾT + AUDIT (KHÔNG dùng thẳng scratch `groth16.ak`
  `Modulo-P ak-381`); ceremony PPoT + phase-2 MPC ≥2 bên RIÊNG mỗi circuit (KHÔNG share Glint
  universal-SRS); uplc-Conway-confirm 2.842B.
- **[cross-repo]** gated-read fail-closed + point-in-time revoke thuộc VeData Query — consume, không cài.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

### 9.1 Tham số + [CẦN CHỐT]

| Tham số | Ý nghĩa | Nguồn / [CẦN CHỐT] |
|---|---|---|
| `K` (shard) | số shard pool | Preview K=1; K>1 đánh đổi anonymity |
| `salt` | 32-byte per-leaf | **[CẦN CHỐT G5]** ngẫu nhiên thật vs HKDF-from-KEK |
| `fee_asset` | tài sản cấn phí Withdraw | **[CẦN CHỐT D6]** LAMP (cần oracle) vs CARP (ổn hơn) |
| `FEE_FLOOR` | sàn cứng coverFee | **[CẦN CHỐT]** > minADA(output) + txFee_floor |
| `K_min` | anonymity-set tối thiểu (Tầng 2) | =32 sơ bộ `[NEEDS-EVIDENCE]` (OQ-EGG-5) |
| denomination | tập mệnh giá (Tầng 2) | `{10,10²,10³,10⁴}` sơ bộ |
| `N_batch_max` | số proof/tx (Tầng 2) | =3 (đo thật); chờ OQ-EGG-8 |
| ExUnit/rút ZK | verify Groth16 | 2.652B lõi / **2.842B deploy** / 2.524B marginal < 10B; `[NEEDS uplc-Conway-confirm, <5%]` |

### 9.2 Số đo Groth16 (nguồn ZK-Math §L — ĐO THẬT, cost-model Plutus V3)

| Kịch bản | CPU (đo) | vs trần 10B/tx |
|---|---:|---|
| verify LÕI (MSM-3 + 4 miller + 3 mul + final) | 2,652,189,198 | thừa ~7.35B |
| verify DEPLOY (vk hard-code) — **số neo** | **2,842,485,975** | thừa ~7.16B |
| marginal same-vk (proof thứ 2+) | 2,524,282,553 | — |

`N_batch_max = 3`: `2.652B + (N−1)·2.524B + overhead ≤ 10B`. Halo2 LOẠI (ước 6–15B, vượt trần).
Groth16 chọn vì: verify rẻ + Frozen-Heart-immune (pairing thuần, không Fiat-Shamir in-verifier) +
malleability-safe (ν dẫn từ `sk`, độc lập proof-bytes) + proof ~200B. Đánh đổi: SRS circuit-specific
⇒ đổi circuit = ceremony phase-2 lại (I-EGG-18). BLS12-381 không PQ (privacy giữ tới CRQC).
Nguồn hàn lâm: Groth, J. (2016). *"On the Size of Pairing-Based Non-interactive Arguments"*,
EUROCRYPT 2016 — https://eprint.iacr.org/2016/260 [GROTH16] (thêm 2026-07-12; xem `PhoenixKey-Math.md`
Appendix G để đối chiếu tag).

---

## 10. Checklist cho auditor

1. `did_pool.ak` cài đủ I-EGG-1..9? Đối chiếu từng require với §5.
2. Sweep bind **per-pair** (I-EGG-4', G3) — KHÔNG chỉ tổng? Nếu hở → chặn-merge.
3. Withdraw có fee-split + `FEE_FLOOR` (G1)?
4. `verify_did_authority` đọc anchor TAAD qua reference-input (như `did_payment`)? (I-EGG-5)
5. Golden vector MST khớp `sumtree.rs::verify_sum_range`; red-team "sửa lá ⇒ proof-fail" (fail-closed).
6. Double-satisfaction: batch nhiều pool-input có ép đếm input thuộc `Script(own_policy)`?
7. salt tái dựng từ Master_KEK (G5) — MST-proof không đòi salt-ngẫu nhiên thật?
8. Tầng 2: verifier Aiken tự-audit? ceremony phase-2 ≥2 bên? nullifier-SMT current-tip (I-EGG-17)?
9. Audit-lane (nếu bật): k≥2 threshold (I-EGG-21), không single-authority?

---

## Phụ lục — bảng đối chiếu invariant ↔ dòng code

| Invariant | Neo pseudocode | Đối chiếu code Aiken |
|---|---|---|
| I-EGG-1..9 (SHIELD) | `Shielded-Custody:143-188` | `did_pool.ak` |
| I-EGG-10..13 (ADDR) | MultiAddress §M.8 | `did_subaddr.ak` |
| I-EGG-14..21 (ZK) | ZK-Math §C.6/§D/§K | verifier Aiken Tầng 2 (số 2.842B = đo VeData, độc lập) |
| I-EGG-22..26 (xuyên tầng) | Feat-Math §5, PrivacyModes §B/§C/§D | bất biến thiết kế + UX; I-EGG-26 cưỡng chế ở relayer-lane Tầng 2 |
| Primitive MST/MMR | `sumtree.rs`/`mmr.rs` | LampNet/Strata (kế thừa) |

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#easteregg)

---

## Nguồn

- Gộp từ nguồn thiết kế nội bộ (không công khai) + `-Feat-Math.md` (§5 bất biến),
  `PhoenixKey-Shielded-Custody-Feat-Math.md §3–§7` (Tầng 1 SHIELD-1..9 + conservation),
  `Easteregg/Easteregg-ZK-Math.md` (§A privacy, §C primitive, §D định lý D.1–D.8, §G recovery,
  §K viewing-key, §L số đo Groth16), `_easteregg-gaming-evaluation.md` (crypto vs residual).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
