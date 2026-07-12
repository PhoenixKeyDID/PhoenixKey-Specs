# PhoenixKey — Rebirthme — Toán hình thức (Math)

> **Module:** Rebirthme (slug `Rebirthme`). **Loại doc:** Toán hình thức cho auditor. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** auditor / kiểm toán mật mã + kinh-tế. Định nghĩa hình thức, bất biến, mệnh đề ép từng thao tác, đối chiếu dòng code thật.
>
> **Ranh giới (MECE):** module này đặc tả **cổng CHI của Ví Phượng hoàng** (`did_payment`), **địa chỉ đa tầng**, **hạn mức anti-drain**, **kho bí mật / phả hệ seed**, **vệ sinh nonce ký**, **stake theo-DID**. **Smartsend** (gửi có bảo vệ) nay là module độc lập — xem [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md); module này CHỈ cung cấp hạ tầng tái dùng (cổng chi, guardian, anti-drain). Cơ chế **state-machine TAAD** (Genesis/Rotate/Init·Cancel·Finalize·Recovery/UpdateGuardians/Transfer/Deactivate) **thuộc Core Anchorme** — ở đây CHỈ dẫn chiếu, không định nghĩa lại. Xem `PhoenixKey-Anchorme-Math.md` + `PhoenixKey-Math.md` §10 (TAAD), §11 (recovery), §6 (key hierarchy), §7 (LampNet), §33 (emergency vault).
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md#rebirthme)
>
> **Mã bất biến:** MOD = `WALLET`. Bất biến mới của module = `I-WALLET-n`. Các bất biến gốc từ nguồn giữ NGUYÊN mã (I-LIMIT-*, I-LIN-*, I-VAULT-*, I-CURVE-4/5, I-ADDR-*, I-SIGN-*, I-DIDSTAKE-*, I-GUARD-*) và ghi rõ nguồn. (**SS-*** của Smartsend nay ở module riêng `PhoenixKey-Smartsend-Math.md`.) Bất biến tái dùng từ `PhoenixKey-Math.md` ghi rõ §.

---

## 1. Ký hiệu (notation)

| Ký hiệu | Nghĩa |
|---|---|
| `did` | định danh phi tập trung, chuỗi `did:phoenix:...` |
| `anchor_policy` | script hash của validator `taad` Design-2, **hằng toàn hệ** (một policy) — `taad.ak` |
| `anchor_name(did) = blake2b_256(did)` | asset-name của anchor NFT per-DID |
| `pay_did(did) = Script(hash(did_payment(anchor_policy, anchor_name(did))))` | payment credential Ví Phượng hoàng |
| `stake_did(did) = Script(hash(did_stake(anchor_policy, anchor_name(did), salt)))` | stake credential theo-DID (chờ on-chain) |
| `pay_sub(org,i)` | payment credential địa chỉ con riêng tư (chờ `did_subaddr`) |
| `vault_addr(did)` | địa chỉ kho `did_payment` |
| `meter_addr(did)` | địa chỉ đồng hồ hạn mức `limit_meter` (chờ on-chain) |
| `controller_pkh` | khoá điều khiển DID, **Ed25519** (28 byte), lưu trong `TAADDatum` |
| `hw_key_pubkey` | khoá phần cứng P-256 (SEC1), lưu carry-by-equality, KHÔNG verify on-chain |
| `Master_KEK` | khoá gốc dẫn xuất từ seed BIP39; KHÔNG rời máy |
| `VaultKEK = HKDF(Master_KEK, "secret-vault-v1", salt=H(did‖"vault-salt-v1"))` | khoá kho bí mật, cách ly |
| `W(tx)` | net giá trị rời kho trong tx (vào − ra lại kho) |
| `Q = 10^9` | Q-format cho token-bucket |
| `lo`, `hi` | biên dưới / trên hữu hạn của validity-interval tx |
| `oildrop = LAMP×10^6`, `nanogic = MAGIC×10^9` | đơn vị (nhất quán brief §4) |

---

## 2. Định nghĩa hình thức các hàm/tập

### 2.1 Cổng chi Ví Phượng hoàng (`did_payment` mode-1)

```
anchor_controller_ok(tx, anchor_policy, anchor_name) ⟺
    ∃! r ∈ tx.reference_inputs : quantity_of(r.value, anchor_policy, anchor_name) == 1
  ∧ read_taad_datum(r) = Some(d)
  ∧ d.status == Active
  ∧ d.controller_pkh ∈ tx.extra_signatories
```
Nguồn: `auth_logic.ak:37-52`. `∃!` (singleton) ở `find_anchor_datum` `auth_logic.ak:59-70`: 0 hoặc >1 carrier → `None` → chi FAIL. `is_active` `auth_logic.ak:82-87`. Cổng KHÔNG ép `entity_type` (`auth_logic.ak:16-18`) → phục vụ mọi loại DID.

```
VaultSpend_mode1(tx, did) ⟺ anchor_controller_ok(tx, anchor_policy, anchor_name(did))
```
Nguồn: `did_payment.ak:40-50`.

### 2.2 Bảng loại địa chỉ (canonical — từ MultiAddress §M.1)

| Loại | payment cred | stake cred | Publish `service[]` | Linkable | Construction |
|---|---|---|---|---|---|
| L1 Base công khai | `pay_did(did)` | `stake_did(did)` | CÓ | CÓ (cố ý) | `did_payment` + `did_stake` |
| L2 Enterprise công khai | `pay_did(org)` | `stake_did(org)` | CÓ | CÓ (cố ý) | = L1, `entity_type=Org` |
| L3 Enterprise riêng tư ×N | `pay_sub(org,i)` | *(mặc định không stake)* | KHÔNG | KHÔNG (unlinkable) | **`did_subaddr`** (L3 unlinkable) |

> **L3 = "Tầng 0" của Easteregg.** `did_subaddr` là MỘT validator duy nhất; Rebirthme gọi nó **L3** (nhất quán hệ L1/L2/L3 của ví Phoenix — tên canonical), Easteregg gọi nó "Tầng 0" trong ngữ cảnh nội bộ của module đó. KHÔNG phải hai validator khác nhau — xem `PhoenixKey-Easteregg-Math.md §1.2`.

### 2.3 Đồng hồ hạn mức (token-bucket — từ Anti-Drain §B)

```
LimitMeterState ≜ Open | Frozen
REFILL_RATE_Q(d) ≜ d.limit_lovelace × Q // d.refill_window
bucket_full_Q(d) ≜ d.limit_lovelace × Q
available_Q(d, lo) ≜ min( bucket_full_Q(d),
                          d.bucket_Q + max(0, lo − d.last_refill_slot) × REFILL_RATE_Q(d) )
W(tx) ≜ Σ_{i∈inputs, addr(i)=vault_addr} value(i).lovelace
      − Σ_{o∈outputs, addr(o)=vault_addr} value(o).lovelace
```
Dùng `lo` (không `hi`) làm "now" — chống khai-man thời gian tương lai. Nguồn: Anti-Drain §B.2–B.5.

### 2.4 Khoá kho bí mật + đường ống (từ Secret-Vault §B)

```
(sk_vault, pk_vault) ≜ derive_x25519_static_from_kek(VaultKEK)          -- lampnet.rs:120
upload(sb, decoy) ⟹ blob_ct = ECIES_Encrypt(pk_vault, pad_to_bucket(serialize(sb) ‖ decoy))
                    cid = LampNet.put(blob_ct); RETURN cid
recover() ⟹ Master_KEK ← BIP39_ToEntropy(24 từ); vidx ← TAADDatum.vault_index_anchor;
             ∀ e ∈ VaultIndex: sb = deserialize(unpad(ECIES_Decrypt(sk_vault, GET(e.cid))))
```
Fail-closed: sai KEK / tamper → GCM tag fail → lỗi, không trả plaintext sai (`lampnet.rs:497-520`).

### 2.5 Bộ khoá pool & vòng đời KES (từ Pool-Ops §B.1–B.4, phục vụ I-POOL-*/I-KES-* §4.M)

```
ColdKey  = (cold_sk : Ed25519Priv, cold_vk : Ed25519Pub)        -- node operator key
VrfKey   = (vrf_sk  : VrfPriv,      vrf_vk  : VrfPub)            -- Praos leader election
KesKey_p = (kes_sk_p: KesPriv@p,    kes_vk  : KesPub)            -- p = KES period gốc của khoá

pool_id  = blake2b_224( cold_vk )                                -- định danh pool on-chain

kes_period(slot) = ⌊ slot / SLOTS_PER_KES_PERIOD ⌋               -- period hiện hành từ SLOT CHAIN
                                                                   -- SLOTS_PER_KES_PERIOD, MAX_KES_EVOL đọc từ genesis, KHÔNG hardcode

OpCert = { kes_vk : KesPub, kes_period_start : ℕ, counter : ℕ, sigma : Ed25519Sig }

make_opcert(cold_sk, kes_vk, period, counter) :=
  OpCert{ kes_vk, kes_period_start := period, counter,
          sigma := Ed25519.Sign( cold_sk, kes_vk ‖ counter ‖ period ) }

valid_opcert(oc, cold_vk) ⟺
      Ed25519.Verify( cold_vk, oc.kes_vk ‖ oc.counter ‖ oc.kes_period_start, oc.sigma )
    ∧ oc.kes_period_start = kes_period(now_slot)       -- I-KES-PERIOD
    ∧ oc.counter = last_counter(pool_id) + 1            -- I-KES-COUNTER, monotonic +1
```
`last_counter(pool_id)` đọc bền từ `PoolAnchor` (dưới), KHÔNG từ bộ nhớ phiên.

```
PoolAnchor = {
  pool_id, cold_vk, vrf_vk, kes_vk_current,
  op_counter     : ℕ,        -- counter cao nhất đã cấp — nguồn chân lý monotonic
  kes_period     : ℕ,
  cid_cold, cid_vrf,          -- LampNet CID, cố định suốt đời pool
  cid_kes,                    -- CẬP NHẬT mỗi rotation
  updated_slot   : SlotNo
}
```

State machine (vòng đời khoá pool):
```
∅ ──create_pool──▶ Registered ──▶ KesActive(p, counter) ──evolve/expire──▶ KesStale
                                       │ rotate_kes (counter+1, re-distribute LampNet TRƯỚC kích hoạt) │
                                       └──────────────────────────◀──────────────────────────────────┘
                                  ──▶ Retiring ──▶ Retired
```
`rotate_kes` sinh **KES key hoàn toàn mới** (không `evolve()` khoá cũ) — mỗi rotation gắn `kes_vk` mới + `counter` mới, đơn giản hoá custody. Phân tán (cold/VRF/KES skey) dùng CHUNG pipeline §2.4 (ECIES qua khoá dẫn xuất từ `Master_KEK`, `data_class="recovery"`); artefact xuất cho node KHÔNG BAO GIỜ gồm `cold_sk` (I-POOL-NODE-BOUNDARY).

Nguồn: `PhoenixKey-Pool-Operations-and-KES-Rotation-Feat-Math.md §B.1–B.6`. Thẩm quyền tạo/xoay tái dùng `auth_satisfied(did, action_tag)` chung (§2.1), `action_tag ∈ {pool_create, pool_rotate_kes, pool_update, pool_retire}` — không sao chép predicate riêng.

---

## 3. Bất biến sổ sách LÕI (conservation)

Ví Phượng hoàng `did_payment` là **stateless** (UTxO không datum) → không giữ sổ token nội bộ; bảo toàn giá trị do luật ledger Cardano lo. Hai chỗ có bảo toàn sổ sách trong module này:

- **Anti-drain (I-LIMIT-NET):** hạn mức tính trên `W(tx)` = NET rời kho, không trên tổng output → chia-output không lách.

(Value-conservation của **Smartsend** escrow (SS-1) nay ở `PhoenixKey-Smartsend-Math.md §3`.)

Module KHÔNG đặc tả tokenomics LAMP/MAGIC/CARP (thuộc MagicLamp) — chỉ khẳng định địa chỉ chứa được token và chi qua đúng cổng `did_payment`.

---

## 4. Bảng bất biến `I-WALLET-n` + kế thừa

### 4.A Bất biến MỚI của module (`I-WALLET-n`)

| Mã | Mô tả | Neo dòng code / nguồn |
|---|---|---|
| **I-WALLET-1** | Ví Phượng hoàng chi hợp lệ ⟺ `anchor_controller_ok` (DID Active + controller hiện tại ký, đọc động qua ref-input); không có đường chi nào khác. | `did_payment.ak:40-50`, `auth_logic.ak:37-52` |
| **I-WALLET-2** | Tài sản đi theo DANH TÍNH: rotate/recovery đổi `controller_pkh` KHÔNG mất tiền, địa chỉ bất biến (đọc controller động). | test `spend_ok_after_controller_rotated` `did_payment.ak:280-290` |
| **I-WALLET-3** | Đóng băng theo trạng thái: `status ∈ {Recovering, Migrated, Revoked}` ⇒ MỌI chi Ví Phượng hoàng bị từ chối (chống chiếm phiên khôi phục). | `auth_logic.ak:13-15,82-87`; test `did_payment.ak:166-208` |
| **I-WALLET-4** | Singleton-anchor: >1 ref-input mang anchor NFT ⇒ mơ hồ authority ⇒ từ chối cả hai. | `auth_logic.ak:59-70`; test `spend_fails_when_two_anchor_refs` `did_payment.ak:296-309` |
| **I-WALLET-5** | Datum-ví bị bỏ qua: UTxO ví không cần datum; datum độc/rác KHÔNG đổi luồng (authority chỉ đọc từ anchor ref-input). | test `wallet_datum_ignored_on_valid_path` `did_payment.ak:340-363` |
| **I-WALLET-6** | Xuất seed vô hiệu trước khi hiện: Export mặc định chạy RotateSeed + MigrateAsset TRƯỚC khi hiện seed → seed hiện ra đã revoked với DID. | UI-Spec §4 (`exportRevokesSeed=true` mặc định) — UI; builder (`generateMasterKek`, rotate, `buildSignedTransfer`) |
| **I-WALLET-7** | Chặn export nếu chưa có đường khôi phục thay thế (guardian < 2 ∧ chưa VC-Glint) — không tự đẩy user vào ngõ cụt. | UI-Spec §4/§5 (I-RECOVERY-4 grace 30 ngày) |
| **I-WALLET-8** | Khôi phục = uỷ quyền rotate, KHÔNG dựng lại Master_KEK; guardian KHÔNG giữ mảnh (đã bỏ Shamir). Đủ MỘT phương thức (guardian / VC-Glint / Midnight). | UI-Spec §0.2, §2, §6 — guardian-threshold , VC/Midnight |

### 4.B Anti-drain (kế thừa NGUYÊN từ Withdrawal-Limit §B.9 — `limit_meter.ak`)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-LIMIT-CAP** | Net rời kho / cửa sổ ≤ `limit_lovelace`, TRỪ điều kiện nâng cao (L1 delay-finalize hoặc L2 khoá phụ/guardian); chỉ-controller-ký không vượt `available_Q/Q`. | (spec §B.9) |
| **I-LIMIT-NOBURST** | Refill liên tục theo `elapsed × REFILL_RATE_Q`, cap ở `limit×Q`, KHÔNG reset mép cửa sổ; dùng `lo` làm now. | — |
| **I-LIMIT-BIND** | Mọi chi từ kho (khi bật) PHẢI tiêu thụ 1 meter-NFT input + tái tạo 1 meter-NFT output; không meter ⇒ kho reject. | — |
| **I-LIMIT-NET** | Hạn mức trên `W(tx)` = NET (vào − ra lại kho), không trên output. | — |
| **I-LIMIT-SERIAL** | Một meter UTxO chi đúng một lần (luật eUTXO) → không rút song song vượt trần trong một block. | — |
| **I-LIMIT-SINGLETON** | Datum meter chỉ tin khi UTxO mang đúng 1 meter-NFT(did); NFT giả/policy khác ⇒ từ chối. | — |
| **I-LIMIT-LOOSEN-DELAY** | 🔴 Nới cấu hình (tăng limit / giảm delay / đổi secondary / đổi safe-address) PHẢI chịu `large_delay` + chủ huỷ được; siết áp ngay. Chống SetConfig limit=∞. | — |
| **I-LIMIT-FREEZE** | Khi `Frozen`, chi từ kho CHỈ tới `safe_address_hash`; đích khác bị từ chối (kể cả rút nhỏ). Chỉ controller gỡ băng. | — |
| **I-LIMIT-CANCEL** | Controller huỷ `PendingLargeWithdrawal` bất cứ lúc nào trước finalize (đối xứng I-GUARD-CANCEL). | — |
| **I-LIMIT-OPTIN** | DID không bật hạn mức ⇒ `did_payment` chạy y Feat §2 (không meter). Bật là thao tác admin có kiểm soát. | — |
| **I-LIMIT-FREEZE-OVER-ACTIVE** | Hạn mức + freeze chạy TRƯỚC anchor vào Recovering/Revoked; khi anchor ≠ Active, `anchor_controller_ok` đã từ chối → hai lớp xếp chồng. | — |

### 4.C Curve-routing bất đối xứng (kế thừa Curve-Routing §5)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-CURVE-4** | 🔴 Anti-drain `limit_meter` là **LOAD-BEARING** cho mọi DID giữ giá trị đáng kể — KHÔNG phải feature trang trí. Vì P-256 enclave không verify on-chain, value chỉ bảo vệ ở mức **SEED** (Ed25519 controller). | (load-bearing, hở HIGH) |
| **I-CURVE-5** | 🔴 ≥1 second-factor rút lớn PHẢI **không dẫn xuất từ** Master_KEK/seed. Nếu `secondary_pkh`/guardian cùng gốc seed → một lần lộ seed sập cả hai. Yêu cầu: phần cứng độc lập hoặc kênh khác (EmailOracle / VeData-Glint). | (chưa enforce ở builder) |

### 4.D Kho bí mật (kế thừa Secret-Vault §B.6 — I-VAULT-*)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-VAULT-1** | Không plaintext rời máy: chỉ `blob_ct` rời máy; khoá giải mã không rời máy. | nền code (lampnet.rs), pipeline |
| **I-VAULT-2** | Decoy bắt buộc: mọi blob `pad_to_bucket` → on-wire chỉ lộ bucket. | — |
| **I-VAULT-3** | Mã hoá client-side bắt buộc: LampNet chỉ nhận `blob_ct`. | nền code |
| **I-VAULT-4** | Fail-closed: sai KEK / tamper → GCM fail → lỗi, không trả plaintext sai. | test `lampnet.rs:497` |
| **I-VAULT-5** | Phí gồm nhiễu: metering trên `size_wire` đã gồm decoy; UI hiện rõ. | — |
| **I-VAULT-6** | CID phải persist (không tất định) — SecretBlob → VaultIndex → `vault_index_anchor`; controller key → `recovery_anchor`. | (schema on-chain) |
| **I-VAULT-7** | Cách ly khoá kho: `VaultKEK` info riêng, isolated khỏi Master_KEK/TAAD_Key/HW_Key (Theorem 28.1 §33). | — |
| **I-VAULT-8** | Mất máy vẫn khôi phục: chỉ 24 từ + đọc chain công khai là dựng lại toàn kho; không phụ thuộc backend uptime. | — |
| **I-VAULT-9** | Không tự ký mạng khác: `raw_key:<chain≠cardano>`/document/note chỉ cất; dùng thì user mở từng lần, zeroize sau. | — |
| **I-VAULT-10** | Seed nhập ≠ khoá DID: `bip39_seed` nhập tạo Imported Account độc lập, KHÔNG đụng Rotation Account/controller. | — |
| **I-VAULT-11** | Fingerprint không bí mật: dedup dùng H(pubkey) không H(payload) — tránh oracle xác nhận bí mật đoán trước. | — |

### 4.E Phả hệ seed (kế thừa Seed-Lineage §F — I-LIN-*)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-LIN-1** | Append-only: seed/khoá KHÔNG bao giờ xoá khỏi lineage; rotate = đánh dấu `rotated` + thêm entry. | (offchain index/UI + Strata LampNet) |
| **I-LIN-2** | Plaintext seed KHÔNG lên Strata: chỉ `content_cid` (ciphertext) + `state_root` (commit trường không bí mật). | — |
| **I-LIN-3** | Khoá giải mã không rời máy: `sk_vault` phái sinh tại chỗ từ VaultKEK; 24 từ là gốc duy nhất. | — |
| **I-LIN-4** | Lineage lộ ra chỉ metadata mã hoá: `ref_id` opaque, `content_cid` hash thuần, `state` commitment. | — |
| **I-LIN-5** | Tamper-evident: MMR append-only + anchor đơn điệu → không xoá/sửa lén. | — |
| **I-LIN-6** | Fingerprint không bí mật: `seed_id = H(pubkey‖salt)` không H(seed) (kế thừa I-VAULT-11). | — |
| **I-LIN-7** | Status chỉ tiến: `active → rotated → compromised` (hoặc active→compromised); từ chối lùi. | — |

### 4.F Địa chỉ đa tầng (kế thừa MultiAddress §M.8 — I-ADDR-*)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-ADDR-CTRL** | Mọi địa chỉ (L1/L2/L3) chi ⟺ anchor Active + controller hiện tại ký (`auth_logic.ak:46-51`). | L1/L2 , L3 |
| **I-ADDR-IDENTITY** | Địa chỉ theo-DID → rotate/recovery không mất tiền, địa chỉ bất biến (= I-WALLET-2). | — |
| **I-ADDR-FREEZE** | `status ∈ {Recovering, Migrated, Revoked}` ⇒ mọi địa chỉ đóng băng (= I-WALLET-3). | — |
| **I-ADDR-PUBLIC** | CHỈ L1 + L2 vào `service[]` (được phép lộ chủ — cố ý). | backend |
| **I-ADDR-PRIV** | 🔴 Mọi địa chỉ L3 KHÔNG BAO GIỜ publish (service[]/resolver/DID Document); backend chặn cứng. | — |
| **I-ADDR-UNLINK** | (chỉ `did_subaddr`) hai sub `i≠j` cùng OrgDID không chia sẻ payment credential (`commit_i≠commit_j`) → không nhóm khi NHẬN; lộ tối thiểu một `tag_i` khi CHI. | (blocker `did_subaddr.ak`) |
| **I-ADDR-ORGALL** | Dù N sub unlinkable, OrgDID vẫn chi được TẤT CẢ (mọi `did_subaddr` quy authority về cùng anchor OrgDID). | — |
| **I-ADDR-STAKE-NOSPEND** | Stake part (`did_stake`) KHÔNG có quyền chi vốn (purpose chỉ Publish/Withdraw) → giúp-delegate-hộ không chạm principal; trần thiệt hại = reward một chu kỳ. | — |
| **I-ADDR-PLOMIN** | Địa chỉ công khai delegate PHẢI kèm VoteDelegation (Abstain mặc định) — kế thừa I-DIDSTAKE-PLOMIN, không rơi bẫy "rút FAIL". | — |
| **I-ADDR-SEED-VALUE** | Quyền CHI value quy về Ed25519 controller (mức seed) → anti-drain load-bearing (I-CURVE-4), second-factor khác gốc seed (I-CURVE-5). Áp cho L1/L2/L3. | — |

### 4.G Smartsend — nay là module riêng

> **Smartsend** (gửi có bảo vệ: escrow + cửa sổ-veto + huỷ đa yếu tố) nay là module độc lập — bất biến **SS-1…SS-11′ + SSR-4** và định lý **T-SS-1..3** chuyển sang `PhoenixKey-Smartsend-Math.md §4.A + §7`. Smartsend TÁI DÙNG guardian (§4.I) + anti-drain (§4.B) + cổng chi `did_payment` (§2.1) của module này — dependency khai ở `PhoenixKey-Smartsend-Math.md §4.B`.

### 4.H Vệ sinh nonce ký (kế thừa Secure-Signing §C — I-SIGN-*)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-SIGN-ED25519-LIB** | Chữ ký Ed25519 (TAAD_Key) PHẢI dùng thư viện kiểm định RFC 8032 deterministic (`ed25519-dalek`); cấm impl tự chế, cấm trộn RNG vào `r`. | `sign.rs:63-65`; ép bằng test-vector RFC 8032 |
| **I-SIGN-ECDSA-NONCE** | `k` của ECDSA P-256 PHẢI từ RFC 6979 HOẶC TRNG/CSPRNG đã thẩm định trong Secure Enclave/StrongBox; cấm sinh `k` ở tầng app bằng RNG không thẩm định. | phần cứng (Secure Enclave/StrongBox) |
| **I-SIGN-NO-REUSE** | Không tái dùng `k` giữa hai message khác nhau; `r` trùng trên message khác ⇒ báo động + quay khoá. | (giám sát server) |
| **I-SIGN-NO-LEAK-K** | `k`/scalar `r`/prefix SHA-512 KHÔNG log/in/trả FFI/ghi storage. | (CI grep) |
| **I-SIGN-LOWS** | Chữ ký P-256 PHẢI low-s (`s ≤ n/2`) tại điểm verify; high-s bị từ chối (chống malleability). | `crypto.rs:339-349` ép `normalize_s.is_some ⇒ reject`; test `test_p256_verify_rejects_high_s_malleable` + `..._accepts_canonical_low_s` (`crypto.rs:641-660`) |
| **I-SIGN-DOMAIN-SEP** | P-256 (auth) và Ed25519 (controller Cardano) tách miền; lỗi nonce miền auth KHÔNG lan tới quyền chi on-chain. | (kiến trúc) |
| **I-SIGN-HW-CUSTODY** | HW_Key non-extractable, sinh + ký trong Secure Enclave/StrongBox; ưu tiên cao hơn tự kiểm soát nonce. | — |

### 4.I Guardian governance (kế thừa Feat §1 — I-GUARD-*)

Mô hình: guardian có **trọng số** (weight) + **vai** (APPROVER / ATTESTER_ALIVE / DELAYER / INITIATOR), khoá "ngủ đông". Luật khôi phục: Σ weight(APPROVER) ≥ threshold ∧ KHÔNG veto-còn sống ∧ vượt timelock ∧ chủ chưa Cancel. Nguồn: `PhoenixKey-Feat.md §1`.

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-GUARD-WEIGHT** | Quyết theo TỔNG trọng số, không số đếm (Feat §1 dòng 62). | GuardianConfig schema; threshold-đếm hiện có ở `taad_logic.ak:87-91` |
| **I-GUARD-VETO** | Một veto ATTESTER_ALIVE ⇒ chặn khôi phục tới hết hạn (chống thông đồng). | — |
| **I-GUARD-CAP** | 🔴 KHÔNG guardian nào có weight ≥ threshold (chống tự khôi phục đơn phương; ngưỡng khôi phục ≥2, khớp `min_guardian_sigs=2` ở `taad_logic.ak:384`). Căn cứ: đây là ngưỡng M-of-N tối thiểu để loại trừ single-point-of-failure — với `threshold=1`, một guardian duy nhất bị chiếm/thông đồng là đủ khôi phục trái phép; `threshold=2` là số nhỏ nhất buộc kẻ tấn công phải phá ≥2 điểm độc lập. Đây là placeholder hợp lý theo nguyên tắc M-of-N Byzantine threshold chung (không phải một con số suy ra từ mô hình đe doạ định lượng cụ thể của PhoenixKey — `[NEEDS-EVIDENCE]` cho việc 2 thay vì 3 là đủ với hồ sơ rủi ro thật). Khung tham chiếu liên quan: NIST SP 800-63B §6.1.2 (account-recovery thresholds) `[NIST-63B]`, đã có trong `PhoenixKey-Math.md` Appendix G. | — |
| **I-GUARD-CANCEL** | Chủ còn khoá ⇒ Cancel được BẤT CỨ lúc nào trong timelock. | `taad_logic.ak:116-131` (CancelRecovery, ub < deadline) |

**Neo code guardian recovery (thuộc Core Anchorme — dẫn chiếu):** `taad_logic.ak:77-108` (InitRecovery: min_guardian_sigs, collateral A-5), `:116-131` (CancelRecovery A-6), `:135-157` (FinalizeRecovery pending-ký), `:186-198` (UpdateGuardians ≤5), params `:383-388` (timelock 3600 slot, min_guardian_sigs 2, collateral 50 ADA). Module này TÁI DÙNG guardian cho Smartsend factor Guardian + anti-drain L2 + Freeze.

### 4.J Stake theo-DID (kế thừa DID-Stake §B.5 — I-DIDSTAKE-*, `did_stake.ak`)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-DIDSTAKE-PRINCIPAL** | Vốn KHÔNG dưới `did_stake`; trần thiệt hại nếu lạm dụng stake-authority = reward một chu kỳ, không principal. | — |
| **I-DIDSTAKE-EXIT** | User always-exit về `exit_addr` (payment seed cũ + stake cũ) KHÔNG cần permission. | — |
| **I-DIDSTAKE-PLOMIN** | Delegation BẮT BUỘC kèm VoteDelegation (Abstain mặc định) → không bẫy "reward tích nhưng rút FAIL". | — |
| **I-DIDSTAKE-ACTIVE** | `did_stake` chi ⟺ anchor status=Active; Recovering/Revoked → đóng băng. | — |
| **I-DIDSTAKE-NOREGDUP** | Nhánh register gộp StakeReg+Delegation+VoteDeleg; delegate-only KHÔNG lặp StakeReg. | — |
| **I-DIDSTAKE-ROUTE** | Withdraw định tuyến qua Grant `route_stake_reward` sống (revocable). | — |

**Bối cảnh (DID-Stake Feat §A, tóm tắt cho auditor).** Vấn đề gốc: mô hình ví
seed thường ghép cứng payment + stake vào cùng khoá → muốn PhoenixKey giúp-delegate
buộc phải đưa quyền chạm khoá đang giữ vốn. Ledger Plomin/Conway còn ép thêm: stake
credential PHẢI đã vote-delegate mới được withdraw reward — delegate "kiểu cũ" (chỉ
stake-delegate) thấy reward tích nhưng rút FAIL on-chain (bẫy thật). DID-Stake dùng
**địa chỉ ghép (Frankenstein)**: `payment` vẫn là khoá seed cũ của user (PhoenixKey
không bao giờ chạm vốn — stake credential trong Cardano không có quyền chi), còn
`stake` là `Script(did_stake)` gated theo controller DID hiện tại. Một stake
credential chỉ delegate được một pool tại một thời điểm → đa-ISPO cần **N stake
credential** (`salt`/`k` phân biệt), PhoenixKey multiplex N luồng cho user nhưng
mỗi luồng vẫn always-exit độc lập. Claim/redeem chương trình (vd Early TIGER
Delegation) chỉ thuộc phần KÝ của PhoenixKey; luật phân phối/token = ngoài phạm vi
(team LAMP).

**Cấu trúc địa chỉ ghép (DID-Stake §B.1):**
```
stake_cred(did, k)  = Script( hash( did_stake(anchor_policy, blake2b_256(did), k) ) )   -- k = 0..N-1, N = số luồng đa-ISPO

franken_addr(did, k) = Address{ payment = <payment cred SEED CŨ user>,   -- KHÔNG do PhoenixKey kiểm-soát
                                 stake   = stake_cred(did, k) }

exit_addr(user)      = Address{ payment = seed cũ, stake = stake cũ của user }   -- đích always-exit
```
`payment` của địa chỉ ghép là khoá seed cũ → mọi UTxO ở đó chỉ seed cũ chi được;
`stake_cred(did,k)` chỉ điều khiển staking, KHÔNG điều khiển chi.

**Predicate `did_stake` (DID-Stake §B.2 — chạy ở hai `ScriptPurpose`, KHÔNG có `Spend`):**
```
AnchorOK(tx, did) ⟺ ∃! ref ∈ tx.reference_inputs : ref mang anchor NFT của did ∧ ref.datum.status = Active

AuthSatisfied(tx, did) ⟺ auth_satisfied(tx, did, "did_stake")   -- helper chung (= §2.1 anchor_controller_ok, action_tag khác)
      Single:   controller_pkh(did) ∈ tx.extra_signatories
    | Registry: entry(action_tag="did_stake", grantor=did) trong DID Authorization Registry (ref-input thứ hai)
                thoả Authorization (SinglePkh | MultiSig{pkhs,threshold}); signer tương-ứng ∈ tx.extra_signatories
                                                                       -- Revoked ⇒ fail

did_stake_valid(tx, did) ⟺ AnchorOK(tx, did) ∧ AuthSatisfied(tx, did)
```
`status ≠ Active` ⇒ `AnchorOK = false` ⇒ từ chối MỌI thao tác staking (đóng băng —
chống chiếm phiên khôi phục để đổi pool/hút reward); tương đương I-DIDSTAKE-ACTIVE.

**Ràng buộc cert (DID-Stake §B.3 — thực thi I-DIDSTAKE-PLOMIN/NOREGDUP):**
```
-- Nhánh register (lần đầu cho stake_cred(did,k)):
certs ⊇ { StakeRegistration(stake_cred(did,k)),
          StakeDelegation(stake_cred(did,k), target_pool),
          VoteDelegation(stake_cred(did,k), drep := Abstain) }        -- Abstain = mặc-định an-toàn

-- Nhánh delegate-only (đã register — top-up / đổi pool):
certs ⊇ { StakeDelegation(stake_cred(did,k), target_pool')
        ∧ ( VoteDelegation(stake_cred(did,k), drep) NẾU chưa từng vote-delegate ) }   -- KHÔNG StakeRegistration lặp
```
Mỗi cert `Publish` đều qua `did_stake_valid(tx, did)`. `drep` mặc định `Abstain`,
user đổi được sang DRep cụ thể / `NoConfidence` qua thao tác vote-delegate riêng
(cũng gated bởi `did_stake_valid`).

**Withdraw — reward routing (DID-Stake §B.4, thực thi I-DIDSTAKE-ROUTE):**
```
Withdraw(reward_account = stake_cred(did,k)) hợp-lệ ⟺
    did_stake_valid(tx, did)
  ∧ ∃ Grant g : g.action = "route_stake_reward" ∧ g.grantor_did = did
              ∧ g.revocable ∧ g sống (chưa thu-hồi, còn hạn)
              ∧ output reward của tx ĐÚNG đích g.resource (= địa-chỉ ISPO HOẶC địa-chỉ user)
```
Tiền điều kiện ledger (Plomin): withdraw chỉ thành công nếu `stake_cred(did,k)` ĐÃ
vote-delegate (B.3) — nếu chưa, ledger từ chối TRƯỚC khi validator chạy; builder
phải bảo đảm vote-delegation tồn tại trước lần withdraw đầu. Nguồn/loại token trả
(ADA vs LAMP; LAMP từ đâu) = ngoài phạm vi PhoenixKey.

**Luận điểm an toàn vốn cho auditor (DID-Stake Feat §A.4 — vì sao pre-audit dễ).**
Bề mặt rủi ro-audit của VỐN = 0: trong Cardano, một stake credential script KHÔNG
BAO GIỜ được hỏi ý-kiến khi **chi** một UTxO (việc chi do payment credential quyết
hoàn toàn); `did_stake` chỉ chạy ở `Publish`/`Withdraw`, KHÔNG có purpose `Spend`
(= I-ADDR-STAKE-NOSPEND). Dù `did_stake` có lỗi logic gì đi nữa, kẻ tấn công KHÔNG
THỂ rút vốn qua nó — chi vốn đòi chữ ký của `payment = seed cũ`, khoá này không bao
giờ nằm trong tay PhoenixKey. → Auditor chỉ cần xác minh MỘT mệnh đề hẹp: "`did_stake`
không cho phép chi vốn" — đúng theo **cấu trúc ledger**, không phụ thuộc độ phức tạp
của validator. Trần thiệt hại tối đa nếu staking-authority bị lạm dụng = phần
**reward tích luỹ một chu kỳ**, KHÔNG phải toàn bộ principal — khác định tính với
mô hình "ví giữ vốn bằng-script" (ở đó một lỗi validator = mất vốn); ở đây lỗi
validator tệ nhất chỉ là sai định tuyến reward một chu kỳ, và always-exit
(I-DIDSTAKE-EXIT) gỡ được ngay, không phụ thuộc PhoenixKey còn hoạt động hay không.

> **[DEP-1]** `did_stake.ak` chưa build (M3 lộ trình) — bảng I-DIDSTAKE-* trên và
> predicate B.1–B.4 là đặc tả chờ hiện thực hoá on-chain, KHÔNG có neo dòng code
> thật; khi build xong cần bổ sung neo test như mẫu `did_payment.ak` (§5.1).

### 4.K Di cư ví cũ (kế thừa Legacy-Migration §A.7 — I-MIGR-*)

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-MIGR-ONESIG** | Một chữ ký payment cũ qua CIP-30; PhoenixKey KHÔNG ký, KHÔNG thấy seed. | (builder) |
| **I-MIGR-SELFUND** | tx tự trả fee/deposit từ reward; thiếu → builder trả "" (không đẩy tx chắc-FAIL). | — |
| **I-MIGR-PLOMIN** | Gộp VoteDelegation→Abstain TRƯỚC Withdrawal (gỡ bẫy Plomin). | — |
| **I-MIGR-TARGET** | LC1 (full-migrate → phoenix_addr) bỏ ví cũ; LC2 (franken, giữ vốn seed cũ) VẪN cần ví cũ → cảnh báo giam vốn. | — |
| **I-MIGR-DEAD** | Mất extension hẳn ⇒ bất khả tuyệt đối; KHÔNG hứa khôi phục. | — |

### 4.L Connector CIP-30 + on-ramp chữ ký (kế thừa Connector §B.7 + Connect-Onramp — I-CONN-*, CONNECT-*, CORE-*)

> Bảng dưới CHỈ giữ bất biến tổng hợp (Connector §B.7). Cơ chế đầy đủ — hai hướng
> dApp (PhoenixKey là dApp vs PhoenixKey expose CIP-30), sequence diagram `enable()`,
> công thức CKDpub derivation watch-only (`acct_xvk` → payment/stake/DRep pub, M2),
> bảng ánh xạ method→khoá/Grant (§B.3), pseudocode `authorize(session,op,tx)` (§B.4)
> và `verify_session`/`verify_login` (§B.5), CIP-45 pairing (§A.4) — ở
> `PhoenixKey-Connector-CIP30-Feat-Math.md` (MECE: file này không định nghĩa lại).

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-CONN-1** | Ví ngoài = watch-only; PhoenixKey KHÔNG giữ khoá ví ngoài. | Feat , code |
| **I-CONN-2** | Chỉ nhận PUBLIC key `acct_xvk` (CKDpub soft-derive), KHÔNG seed/xprv. | — |
| **I-CONN-3** | Hướng-2 (expose CIP-30): mỗi connection = Grant revocable + hạn hữu hạn (không connect-toàn quyền vô hạn). | (consent-store) |
| **I-CONN-4** | Cấp phạm vi tối thiểu (action/resource). | — |
| **I-CONN-5** | Phiên dApp sống liệt kê được + thu hồi tức thì. | — |
| **I-CONN-6** | `signTx` → Ed25519 controller; `signData` → Ed25519 COSE_Sign1 (CIP-8), KHÔNG P-256; login-native P-256 riêng. | — |
| **I-CONN-7** | nonce+exp, QR không mang bí mật, origin xác thực. | — |
| **CONNECT-M1..M4** | On-ramp mandate: KHÔNG cấp chi on-chain; người ký = khoá ví ngoài COSE, không controller; verify addr↔pubkey↔sig; mọi mandate có domain/nonce/hạn, revocable. | cần 2 FFI Core mới |
| **CONNECT-E1..E3** | Ghi có LAMP: KHÔNG chi UTxO ví cũ; LAMP luôn route về `phoenix_addr`; eligibility theo program. | — |
| **CONNECT-S1..S3** | An ninh: không đường mandate→chi UTxO ví cũ; khoá lộ ⇒ tối đa ghi có LAMP về chính phoenix_addr; thu hồi tức thì. | — |
| **CORE-MIGR-1..5 / CORE-CLAIM-1..6 / CORE-SIGN-1..4** | Builder Delegator (unsigned migration Lace-ký; claim controller/Lace mode; multi-witness merge). | đội on-chain |

**Định nghĩa hình thức `AuthorizationMandate` + `verify_mandate` (Connect-Onramp §1, phục vụ CONNECT-M1..M4):**

```
AuthorizationMandate ≜ Grant {
  wallet_address : Bech32Addr,   -- payment addr ví ngoài (Lace/Yoroi) đã ký
  phoenix_did    : DID,          -- lightweight DID (§2.1 dưới) — grantor
  scope          : "lamp_program",
  program_ids    : List<String>, -- [] = mọi program của issuer
  nonce          : Bytes32,      -- server cấp, chống phát-lại
  valid_from     : UnixTime,
  valid_until    : UnixTime,     -- BẮT BUỘC hữu-hạn
  revocable      : true,
  domain         : Origin,       -- neo server-side khi phát nonce, KHÔNG client tự khai
  cose_sign1     : Bytes,        -- COSE_Sign1 (CIP-8, Ed25519) ví ngoài trả
}

mandate_payload ≜ canonical_json({
  t:"phoenixkey.mandate.v1", addr:wallet_address, did:phoenix_did, scope:"lamp_program",
  progs:program_ids, nonce:nonce_hex, iat:valid_from, exp:valid_until, dom:domain
})

verify_mandate(m):
  1. (payload, sig, pubkey) = cose_sign1_parse(m.cose_sign1)
  2. require payload == canonical(m.mandate_payload)                -- không tráo nội-dung
  3. require cose_verify(pubkey, payload, sig) == OK                -- Ed25519 COSE_Sign1 (I-CONN-6)
  4. require addr_matches_pubkey(m.wallet_address, pubkey)          -- payment_cred(addr)==blake2b_224(pubkey)
  5. require m.domain ∈ ALLOWED_ORIGINS                             -- domain-binding
  6. require valid_from ≤ now ≤ valid_until
  7. require lookup_nonce(m.nonce).status == Unused                 -- chống phát-lại
  8. bind: store Grant{grantor=m.phoenix_did, wallet_address=m.wallet_address, ...}
```

Bước 4 = "bind" mấu chốt: khoá ký = khoá chi của `wallet_address` → gắn cứng `wallet_address ↔ phoenix_did` do chính chủ khoá ngoài tự khẳng định có kiểm chứng (KHÔNG suy `phoenix_did` từ nguồn khác).

**Lightweight DID neo địa chỉ (Connect-Onramp §2.1, MỚI — cho user ví ngoài không có Secure Enclave):**
```
phoenix_did_ext(wallet_address) = "did:phoenix:ext:" ‖ blake2b_224(payment_cred(wallet_address))
```
Tất định từ `wallet_address` (idempotent, cùng ví → cùng DID); controller ban đầu = khoá ví ngoài (verify qua COSE_Sign1, cùng `taad_cose_sign1_verify`). Backend KHÔNG giữ private key. Nâng cấp lên PersonDID đầy đủ = rotate/merge controller khi user tốt nghiệp (§5 Connect-Onramp).

**Ghi có LAMP — hai chế độ xác nhận kiểm soát địa chỉ mỗi epoch (CONNECT-E1..E3):**
- **Chế độ A (mandate-đủ):** mandate còn hạn + `program_id ∈ program_ids` + chưa thu hồi ⇒ đủ tính tham gia epoch, không cần ký lại.
- **Chế độ B (thách thức tươi):** program đòi re-prove mỗi epoch; server cấp `challenge` (nonce+epoch), user ký `signData` nhẹ (`"phoenixkey.epoch.proof.v1"`), verify như trên. Chọn A/B là điều khoản program (LAMP team), không hard-code ở Connect.

LAMP ghi có luôn route `claim_address = phoenix_addr(phoenix_did)` (KHÔNG bao giờ về `wallet_address`) — CONNECT-E2; leaf Snapshot-Merkle `(claim_address, amount, epoch)` theo `LAMP-DISTRIBUTION-SPEC §2/§6`.

**FFI Core MỚI (verify §7.1 Connect-Onramp, giao cho W11):**
```rust
pub fn cose_sign1_verify(cose_sign1_hex: &str) -> Option<CoseVerified>;
// CoseVerified { payload_hex: String, pubkey_hex: String } — Ed25519 CIP-8, KHÔNG P-256 (I-CONN-6)
pub fn addr_matches_pubkey(addr_bech32: &str, pubkey_hex: &str) -> bool;
// payment_cred(addr) == blake2b_224(pubkey); hỗ trợ base/enterprise/reward addr

#[no_mangle] pub unsafe extern "C" fn taad_cose_sign1_verify(cose_sign1_hex: *const c_char) -> *mut c_char;
// JSON {"ok":bool,"payload_hex":..,"pubkey_hex":..}; null nếu lỗi (KHÔNG panic qua FFI)
#[no_mangle] pub unsafe extern "C" fn taad_addr_matches_pubkey(addr_bech32: *const c_char, pubkey_hex: *const c_char) -> u8;
```
Khác `taad_cose_sign1_build` (Delegator-Core §1.5 Phase-2, PhoenixKey LÀ bên ký) — đây là chiều VERIFY (PhoenixKey nhận chữ ký ví ngoài). Dùng crate COSE (`coset`) + `ed25519-dalek` (đã có `sign.rs`) + `Blake2b224` (`phoenix_address.rs:39`, đã có).

**Ranh giới an ninh (CONNECT-S1..S3, đối chiếu §6 Connect-Onramp):** không tồn tại đường mandate→chi UTxO ví ngoài (Cardano không có account-abstraction); mandate KHÔNG phải bearer-token (verify luôn resolve trạng thái sống+hạn+nonce); khoá ví ngoài lộ ⇒ kẻ tấn công tối đa ký mandate mới ghi có LAMP về CHÍNH `phoenix_addr` của DID đó (route cố định CONNECT-E2, không đổi hướng được) + rủi ro chi ví ngoài chưa-migrate (vốn có, không do Connect mở rộng bề mặt tấn công).

### 4.M Vận hành pool + xoay KES (kế thừa Pool-Ops §B.8 — I-POOL-*, I-KES-*)

Định nghĩa hình thức (bộ khoá, `OpCert`, `PoolAnchor`, state machine) ở §2.5.

| Mã | Mô tả | Neo / ghi chú |
|---|---|---|
| **I-POOL-COLD-SECRET** | Cold key plaintext chỉ trong RAM, biometric-gate, zeroize sau dùng. | (PoC) |
| **I-POOL-WRAP** | Key ra máy CHỈ dạng ECIES ciphertext (không plaintext). | — |
| **I-POOL-RECOVER** | Khôi phục từ 24 từ + `pool_anchor`; fail-closed. | — |
| **I-POOL-ID** | `pool_id = blake2b_224(cold_vk)` bất biến; đổi cold = pool khác. | — |
| **I-KES-COUNTER** | Op-cert counter MONOTONIC, không tụt/trùng/reset (kể cả qua recovery). | — |
| **I-KES-PERIOD** | Op-cert `kes_period_start = kes_period(now_slot)`; KES hết hạn sau MAX_KES_EVOL. | — |
| **I-KES-REDIST** | Re-distribute KES mới LampNet TRƯỚC khi coi xoay hoàn tất. | — |
| **I-POOL-NODE-BOUNDARY** | Artefact node = KES+VRF+op-cert, KHÔNG cold; node bị hack lộ KES (xoay được) KHÔNG lộ cold. | — |
| **I-POOL-FREEZE** | `pool_*` chi ⟺ anchor Active; Recovering/Revoked → đóng băng. | — |
| **I-POOL-AUTH** | `auth_satisfied(did, action_tag ∈ {pool_create, pool_rotate_kes, pool_update, pool_retire})` + thu hồi quyền → chặn tức thì. | — |

---

## 5. Mệnh đề ép từng thao tác (đối chiếu code load-bearing)

### 5.1 Chi Ví Phượng hoàng
```
did_payment.spend(anchor_policy, anchor_name, _datum, Spend, _own_ref, self)
  = auth_logic.anchor_controller_ok(self, anchor_policy, anchor_name)          -- did_payment.ak:49
```
Guard load-bearing:
- `expect Some(anchor) = find_anchor_datum(...)` — `∃!` singleton (`auth_logic.ak:43-44,59-70`). Guard I-WALLET-4.
- `is_active(anchor.status)` — guard I-WALLET-3 / I-ADDR-FREEZE (`auth_logic.ak:47,82-87`).
- `list.has(tx.extra_signatories, anchor.controller_pkh)` — guard I-WALLET-1, đọc động (`auth_logic.ak:50`).

Mệnh đề (ép bởi test):
- (a) Active + controller ký → PASS (`did_payment.ak:140-149`).
- (b) thiếu chữ ký → FAIL (`:153-162`).
- (c) Recovering/Revoked/Migrated → FAIL (`:166-208`).
- (d) anchor NFT giả (policy/name sai) → FAIL (`:212-236`).
- (e) không có ref-input anchor → FAIL (`:240-274`).
- (+) rotate rồi controller mới ký → PASS (`:280-290`) — **chứng minh I-WALLET-2**.
- (+) 2 ref anchor → FAIL (`:296-309`) — chứng minh I-WALLET-4.

### 5.2 Guardian recovery (cơ chế thuộc Core, dẫn chiếu)
InitRecovery ép: `signed_guardians ≥ min_guardian_sigs`, `collateral ≥ recovery_collateral_lovelace` nạp vào UTxO tiếp (A-5), `sequence+1`, `status→Recovering{deadline = lb + timelock}` (`taad_logic.ak:90-108`). CancelRecovery ép `ub < deadline_slot` + controller ký (A-6, `:119-121`). FinalizeRecovery ép `lb > deadline_slot` + `pending_controller_pkh` ký (`:140-149`). → hệ quả cho ví: trong Recovering, I-WALLET-3 đóng mọi chi.

### 5.3 Anti-drain (mệnh đề spec)
```
MeterSpend_ok(tx, d, lo) ⟺
    (d.state=Open ∧ (SmallWithdraw_ok(d,lo,W(tx)) ∨ LargeWithdraw_ok(tx,d,lo)))
  ∨ (d.state=Frozen ∧ FrozenSpend_ok(tx,d))
  ∨ AdminMeter_ok(tx,d)
SmallWithdraw_ok ⟺ d.state=Open ∧ W(tx)×Q ≤ available_Q(d,lo)
                    ∧ meter_next.bucket_Q = available_Q − W(tx)×Q ∧ meter_next.last_refill_slot = lo
```
Guard cốt tử SetConfig (I-LIMIT-LOOSEN-DELAY): nới → chịu `large_delay`. Nguồn Anti-Drain §B.3–B.7.

### 5.4 Smartsend — mệnh đề chuyển sang module riêng
Ba (bốn) đường thoát escrow-UTxO + mệnh đề ép SS-* nay ở `PhoenixKey-Smartsend-Math.md §5`. Module này chỉ cung cấp cổng chi nguồn (§5.1) + guardian (§5.2) mà Smartsend tái dùng.

---

## 6. Không gian trạng thái + đồ thị chuyển

**Ví Phượng hoàng** (stateless — không có state riêng; trạng thái đọc từ anchor TAAD):
```
[anchor.status] : Active ──(chi được)──►
                  Recovering / Migrated / Revoked ──(mọi chi FAIL, I-WALLET-3)
```

**Đồng hồ hạn mức** (khi bật):
```
Open ──SetConfig(siết)──► Open        Open ──Freeze──► Frozen ──Unfreeze(controller)──► Open
Open ──SmallWithdraw/LargeWithdraw──► Open (bucket cập-nhật)
Open ──OpenLarge──► [PendingLargeWithdrawal] ──(FinalizeLarge sau delay | CancelLarge)──► Open
```

**Smartsend escrow** (nay module riêng): đồ thị chuyển ở `PhoenixKey-Smartsend-Math.md §2.2`.

**Phả hệ khoá:** `active → rotated → compromised` (hoặc active→compromised), chỉ tiến (I-LIN-7).

---

## 7. Định lý an toàn + chứng minh phác thảo

**T-WALLET-1 (Tài sản sống qua rotate/recovery).** Với mọi tx `t`, `did_payment.spend` PASS ⟺ `anchor_controller_ok` (đọc `controller_pkh` HIỆN TẠI từ anchor ref-input). Rotate/Finalize chỉ đổi `controller_pkh` trong datum anchor, KHÔNG đổi `pay_did(did)` (hàm của `anchor_name(did)`, bất biến). ⇒ địa chỉ bất biến, khoá mới chi được. ∎ (test `did_payment.ak:280-290`).

**T-WALLET-2 (Đóng băng chống chiếm phiên khôi phục).** Khi `status ≠ Active`, `is_active` = False ⇒ `anchor_controller_ok` = False ⇒ mọi chi FAIL. Trong Recovering, kẻ chiếm controller cũ KHÔNG rút được; hệ quả xếp chồng với anti-drain (I-LIMIT-FREEZE-OVER-ACTIVE). ∎ (test `:166-208`).

**T-WALLET-3 (Trần thiệt hại khi lộ seed — CÓ ĐIỀU KIỆN).** NẾU anti-drain bật, thiệt hại tối đa khi lộ controller/seed = `limit` mỗi `refill_window` (I-LIMIT-CAP), phần lớn cần delay (chủ huỷ được) hoặc second-factor khác gốc-seed (I-CURVE-5). **Điều kiện của định lý:** anti-drain (`limit_meter.ak`, I-CURVE-4) PHẢI bật cho DID giữ giá trị đáng kể — không bật thì trần này không áp dụng và thiệt hại tối đa = toàn bộ kho nếu seed lộ. ∎ (có điều kiện; §9).

**T-SS-1..3 (Smartsend).** No-loss / Veto-safety / Anti-theft — nay ở `PhoenixKey-Smartsend-Math.md §7` (với 3 tiền đề §6 của file đó).

---

## 8. Giả định tin cậy (ngoài phạm vi chứng minh)

- **Secure Enclave / StrongBox** sinh nonce ECDSA P-256 chất lượng (không đọc/chứng minh được — Secure-Signing §B.2.1). Bù bằng low-s + giám sát `r` + tách miền.
- **Ledger Cardano** thực thi đúng luật eUTXO (double-spend từ chối → I-LIMIT-SERIAL).
- **LampNet node** giữ ciphertext trung thực + erasure đủ mảnh để dựng lại (durability I-VAULT-8 phụ thuộc Mirage `EncryptedDistributed`).
- **Guardian** không thông đồng vượt ngưỡng trong cửa sổ timelock (bù bằng vai veto + Cancel).
(Giả định Spectra/Glint cho factor bối cảnh ZK nay ở `PhoenixKey-Smartsend-Math.md §8`.)

---

## 9. [CẦN CHỐT] còn treo

- **[CẦN CHỐT-W1]** Meter-NFT policy: dùng `taad` Design-2 hay policy meter chuyên dụng (Anti-Drain §B.1 "chốt khi code").
- **[CẦN CHỐT-W2]** `PendingLargeWithdrawal`: biến thể datum meter hay UTxO pending riêng (Anti-Drain §B.7).
- **[CẦN CHỐT-W3]** `vault_index_anchor` + `recovery_anchor` vào `TAADDatum` (schema thuộc Specs/Validator — đội backend). [OPEN-V2 Secret-Vault].
- **[CẦN CHỐT-W4]** Smartsend (nay module riêng): 5 vá 🔴 + phụ thuộc anti-drain — treo ở `PhoenixKey-Smartsend-Math.md §9`.
- **[CẦN CHỐT-W5]** `did_subaddr.ak` (L3 unlinkable) — dependency onchain MỚI [DEP-2], chờ maintainer chốt.
- **[CẦN CHỐT-W6]** R1 vs R2 khoá kho: chốt R2 (VaultKEK cách ly) canonical cho kho, R1 cho `recovery_anchor` [OPEN-V1].
- **[CẦN CHỐT-W7]** Bước bucket B + sàn FLOOR_BYTES (đề xuất 4 KiB / 64 KiB) [OPEN-V4].
- **[CẦN CHỐT-W8]** Enforce I-CURVE-5 ở builder: guardian/secondary Ed25519 phải khác gốc seed (chưa thấy enforce).
- **[CẦN CHỐT-W9]** GuardianConfig (trọng số/vai) vào `TAADDatum` cần `schema_version` + headroom (Feat §1) — thuộc Core Anchorme/Validator; module này dẫn chiếu.
- **[CẦN CHỐT-W10]** Pool-Ops: **[VERIFY]** thư viện KES/VRF Rust khả dụng (rủi ro chính, cần PoC sớm — Pool §C.7 OPEN-P3); `pool_anchor` on-chain schema [OPEN-P1].
- **[CẦN CHỐT-W11]** 2 FFI Core mới cho on-ramp: `taad_cose_sign1_verify`, `taad_addr_matches_pubkey` (Connect-Onramp §7).
- **[CẦN CHỐT-W12]** Đa-pool một DID: một người vận hành N pool — counter store + indexer phải phân biệt theo `pool_id` (schema `PoolAnchor` §2.5 đã hỗ trợ per-pool, cần xác nhận indexer backend) (Pool §C.7 OPEN-P4).

---

## 10. Checklist cho auditor

- [ ] Ví Phượng hoàng: chi ⟺ `anchor_controller_ok`; không đường bypass (I-WALLET-1). Kiểm 8 test `did_payment.ak`.
- [ ] Rotate-survival: `spend_ok_after_controller_rotated` PASS (I-WALLET-2/T-WALLET-1).
- [ ] Đóng băng: 3 status ≠ Active đều FAIL (I-WALLET-3/T-WALLET-2).
- [ ] Singleton-anchor: 2-ref FAIL (I-WALLET-4).
- [ ] Datum-ví bỏ qua: datum độc/rác không đổi luồng (I-WALLET-5).
- [ ] Guardian recovery: Init≥threshold, collateral nạp, timelock, Cancel<deadline, Finalize pending-ký (`taad_logic.ak:77-157`).
- [ ] Anti-drain: verify `limit_meter.ak` ép đúng I-LIMIT-CAP/NET/SERIAL/SINGLETON/LOOSEN-DELAY/FREEZE trước khi coi DID giá trị lớn an toàn (I-CURVE-4 load-bearing).
- [ ] Second-factor khác gốc-seed (I-CURVE-5) — enforce ở builder.
- [ ] Kho bí mật: fail-closed (`lampnet.rs:497`), decoy bắt buộc, VaultKEK cách ly, không tự ký mạng khác.
- [ ] Ký: Ed25519 dalek deterministic; P-256 low-s ép tại verify (`crypto.rs:339-349` + test `:641-660`); giám sát `r` lặp server-side (I-SIGN-NO-REUSE).
- [ ] Smartsend (nay module riêng): checklist ở `PhoenixKey-Smartsend-Math.md §10`; nền dẫn chiếu (guardian/anti-drain/`did_payment`) kiểm ở đây.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md#rebirthme)

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai).
Math canonical (dẫn chiếu, KHÔNG sửa): `PhoenixKey-Math.md` §6, §7, §9, §10, §11, §33;
`PhoenixKey-Connector-CIP30-Feat-Math.md` (§4.L — cơ chế CIP-30/CIP-45 đầy đủ).
Code: `PhoenixKey-Validator/validators/did_payment.ak`, `lib/phoenixkey/{auth_logic,taad_logic}.ak`; `rust_core/src/{lampnet,crypto,sign}.rs`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
