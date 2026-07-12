# PhoenixKey — Protectme — Đặc tả KỸ THUẬT (cho kỹ sư)

> **Module:** Protectme (Bảo hiểm phủ tài sản) · **Loại doc:** Kỹ thuật (implementer) ·
> **Ngày:** 2026-07-09.
> **Đối tượng đọc:** KỸ SƯ triển khai — đội on-chain, đội backend (backend + Treasury), Core/
> SuperApp (UI), MAGIC/CARP/Feecover-team.
> **Mục đích:** kiến trúc + datum/redeemer CBOR + luồng tx + ai ký + ranh giới đội + thứ tự
> deploy + phụ thuộc CHẶN. Đây là **HOW**; lý do kinh-tế đọc [PhoenixKey-Protectme-Vi-Feat.md](./PhoenixKey-Protectme-Vi-Feat.md)/[PhoenixKey-Protectme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Protectme-Specs/blob/main/PhoenixKey-Protectme-Exec.md), toán
> đọc [PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md).
>
> **Ranh giới mã (branch `feat/protectme-payout`):**
> - `PhoenixKey-Validator/lib/phoenixkey/protectme_types.ak`
> - `PhoenixKey-Validator/lib/phoenixkey/protectme_logic.ak`
> - `PhoenixKey-Validator/validators/protectme_payout.ak`
>
> Ba trạng thái phân biệt xuyên suốt bản này:
> - **ĐÃ CÓ** — đã hiện thực trong branch trên (neo file:hàm cụ thể).
> - **THÊM** — cần viết mới để chặn-merge (chủ yếu: **beacon one-shot per claim_id**).
> - **CHỜ CHỐT** — lệch spec/chưa quyết, KHÔNG code tới khi maintainer + đội backend chốt hướng.
>
> **Đơn vị:** mọi giá trị chi/thu = **CARP** (đơn vị dàn xếp). MAGIC = nhãn giá premium
> (`P*=1`), tiêu qua Vault `BurnBatch`, KHÔNG token/mint. Xem [PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md) §2 + §9.
>
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md#protectme)

---

# 1. Kiến trúc + sơ đồ thành phần

Protectme chia làm **bốn khối hợp tác**, ranh giới đội rõ (§7):

```
                         ┌──────────────────────────────────────────┐
   [Feecover]            │            PHOENIX TREASURY               │
   premium CARP  ──────► │  bucket protectme_pot_user   (CARP)       │  (đội backend)
   θ×fee   CARP  ──────► │  bucket protectme_pot_system (CARP)       │
                         └───────────┬──────────────────────────────┘
                                     │ committee-gated release (B.6)
                                     │ + MINT beacon (THÊM) khi nạp escrow
                                     ▼
                         ┌──────────────────────────────────────────┐
   claim_id ──beacon────►│   ESCROW UTxO @ protectme_payout          │  (đội on-chain)
   (state NFT, THÊM)     │   value = amount CARP + beacon NFT + ADA  │
                         │   datum = ClaimDatum (đã DUYỆT)           │
                         └───────┬───────────────────┬──────────────┘
                          Payout │                   │ Reject{reason}
                          (perm. │                   │ (committee M-of-N ký)
                          -after │                   ▼
                          window)│           hoàn TOÀN BỘ escrow
                                 ▼           về bucket(cause)  +  BURN beacon
                    ┌────────────┴───────────┐
        payee_cred (Ví Phượng hoàng)   record_cred (ClaimRecord bất biến, T2)
             nhận amount CARP+ADA        + BURN beacon (THÊM)
```

**Bốn khối:**

1. **Cổng CHI TRẢ (validator payout)** — `validators/protectme_payout.ak`. Gác UTxO
   escrow của một claim ĐÃ DUYỆT. Ép bất biến **phía chi** (NO-DOUBLE, CHALLENGE,
   SINGLE-FACTOR, POOL/INCIDENT-CAP, T2, CAUSE-SPLIT/NO-CROSS-BUCKET). **ĐÃ CÓ.**
2. **Escrow ↔ bucket** — nạp escrow từ đúng bucket khi committee duyệt (release-gate
   Treasury B.6). **CHỜ CHỐT** (escrow-tự giữ hiện tại vs release-from-bucket B.6, §8.2).
3. **Committee/panel** — authority phân xử M-of-N Ed25519, cấu hình trong
   `ProtectmeConfig`. Ép Reject cần ≥`committee_threshold` khoá **khác nhau**. **ĐÃ CÓ.**
4. **Beacon one-shot per claim_id** — state NFT singleton `asset_name = claim_id`, policy
   ≡ own-hash. Mint khi nạp escrow (provenance Feecover→bucket→escrow), burn khi
   Payout/Reject. Đóng #6 (escrow-giả) + F3 (double-payout). **THÊM — chặn-merge.**

## 1.1 Bất biến kiến trúc (load-bearing)

- **Validator payout KHÔNG tin lời khai** — nó ép công thức `expected_amount` trên
  `loss_eligible` + `policy` đã committee-chứng thực off-chain (không có oracle on-chain
  cho "có phải trộm không", [PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md) §0/§9, C.6). Validator **CHỈ** gác đường CHI/HOÀN
  của escrow đã nạp.
- **Escrow chi đúng một lần** — eUTXO đảm bảo trong-tx + liên-tx-cùng-UTxO; liên-CLAIM
  (cùng `claim_id`, khác UTxO) phụ thuộc **beacon** (THÊM, khối 4). Đây là bất biến
  I-PROT-BEACON-ONESHOT ([PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md) §5.2) — **chưa đóng on-chain**.
- **Hai bucket tách vật lý** — `pot_system` và `pot_user` không vay chéo. Reject hoàn về
  ĐÚNG bucket (I-PROT-NO-CROSS-BUCKET, đã ép).

---

# 2. Validator payout — ĐÃ CÓ (đối chiếu mã branch)

## 2.1 Entry point

`validators/protectme_payout.ak` — `validator protectme_payout(config: ProtectmeConfig)`:

```
spend(datum_opt, redeemer, own_ref, self):
  expect Some(d) = datum_opt
  when redeemer is {
    Payout        -> protectme_logic.payout_ok(config, d, own_ref, self)
    Reject { .. } -> protectme_logic.reject_ok(config, d, own_ref, self)
  }
else(_) { fail }
```

`config: ProtectmeConfig` là **apply-param** (mẫu `ValidatorParams`) → nới/siết rule =
DAO đổi config, deploy instance mới; policy cũ grandfather theo `schema_version`
(I-PROT-CFG).

## 2.2 Bất biến payout (`protectme_logic.payout_ok`) — ĐÃ ép

Neo `lib/phoenixkey/protectme_logic.ak:payout_ok`. Mọi mệnh đề dưới đã có mã + test:

| # | Bất biến | Mệnh đề trong mã | Chống |
|---|---|---|---|
| P1 | miền tham số | `policy_domain_ok(loss, policy)` + `amount > 0` | payout âm / tạo giá trị (F1/F4) |
| P2 | I-PROT-NO-DOUBLE | `d.amount == expected_amount(cause, loss, policy)` | trả sai công thức |
| P3 | I-PROT-INCIDENT-CAP | `d.amount <= cfg.incident_cap` | vượt trần/incident |
| P4 | ràng buộc giá trị escrow | `escrow_carp == d.amount` | escrow không khớp amount |
| P5 | double-satisfaction | `count_inputs_at(tx, own_cred) == 1` | gộp nhiều escrow một tx (CRIT1) |
| P6 | cred phân biệt | `payee ≠ record ≠ own_cred` | một output đóng hai vai (CRIT2) |
| P7 | I-PROT-CHALLENGE | `lo >= approved_slot + t_wait_payout + challenge_window` | chi sớm (F5, đọc `lower_bound`) |
| P8 | I-PROT-SINGLE-FACTOR | `single_factor_ok(cause, policy, require_ad)` | phủ USER_NEG single-factor không anti-drain (C.8) |
| P9 | bảo toàn giá trị | `sole_output_value_at(outputs, payee_cred) == Some(escrow_val)` | skim ADA/token, chia đôi (HIGH3) |
| P10 | I-PROT-T2 | `record_ok(outputs, record_cred, claim_id, did, amount, cause)` | thiếu claim-record bất biến |

Công thức chi (`expected_amount`): SYS = `min(loss, cap_sys)` (không co-pay); USER_NEG =
`max(0, min(loss,cap_user) − deductible) × (10000 − copay_bps)/10000`.

## 2.3 Bất biến reject (`protectme_logic.reject_ok`) — ĐÃ ép

| # | Bất biến | Mệnh đề | Chống |
|---|---|---|---|
| R1 | committee M-of-N | `committee_ok(tx, cfg)` — `list.unique` rồi đếm ≥ threshold | một khoá grief / khoá trùng (F7) |
| R2 | double-satisfaction | `count_inputs_at(tx, own_cred) == 1` | gộp escrow |
| R3 | NO-CROSS-BUCKET + bảo toàn | `sole_output_value_at(outputs, bucket_cred(cause)) == Some(escrow_val)` | hoàn sai bucket / skim (HIGH4) |

## 2.4 Ai ký (ĐÃ CÓ)

- **Payout:** **permissionless-after-window.** `payee_cred` + `amount` đã CỐ ĐỊNH trong
  datum → ai kích hoạt cũng an toàn (tiền chỉ chảy về claimant). Không đòi committee ký
  lúc chi (họ đã duyệt lúc nạp escrow + set datum).
- **Reject:** committee ký, ≥ `committee_threshold` khoá **khác nhau** trong
  `extra_signatories` (dedup qua `list.unique`).

## 2.5 🟡 Siết hẹp P9 — stake_credential + payee VerificationKey-cred (CHỜ LÀM, rẻ, trước-merge)

P9 (`sole_output_value_at(outputs, payee_cred) == Some(escrow_val)`) hiện chỉ khớp
**`payment_credential`**, bỏ qua `stake_credential`; và `payee_cred` giả định output đích
luôn mang **`Script`**-credential. Hai chỗ hẹp cần đóng trước merge (rẻ, không đổi kiến trúc):

- **(a) `stake_credential` bị bỏ:** hiện chỉ ép đúng payment-cred tại output payee, không ép
  stake-cred. Cần siết thêm (khớp cả cặp payment+stake) HOẶC ghi rõ trong `protectme_types.ak`
  rằng bỏ qua-stake là **cố ý** (ví dụ nếu payee luôn none-stake theo thiết kế ví Phượng hoàng).
  Chưa quyết — đội on-chain xác nhận rồi chốt.
- **(b) `payee_cred` giả định luôn `Script`-cred:** ví Phượng hoàng ở **L1 công khai**
  ([multiaddress-custody-model] — L1 base công khai payment+stake) có thể là
  **VerificationKey-credential**, không phải Script. `sole_output_value_at` cần hỗ trợ cả hai
  dạng credential cho payee, không chỉ Script.

**Vị trí:** `protectme_logic.ak:payout_ok` (P9) + `protectme_types.ak` (định nghĩa
`payee_cred`). **Không chặn-merge** như beacon (§3/§8.1) nhưng nên làm trước-merge vì rẻ.

---

# 3. Beacon one-shot per claim_id — THÊM (chặn-merge)

> **Đây là hạng mục chặn-merge từ feedback đội on-chain.** Không có nó, hai lỗ còn nguyên
> (gaming VG-P1):
> - **#6 escrow-giả:** validator giả định escrow "đã nạp hợp lệ" (chỉ kiểm `escrow_carp
>   == amount`), KHÔNG kiểm CARP đó **đến từ đúng bucket**. Ai gửi CARP vào địa chỉ
>   escrow với một `ClaimDatum` bịa có thể tạo escrow-giả.
> - **F3 double-payout:** header `protectme_types.ak` ghi rõ — "nếu Treasury nạp HAI
>   escrow cùng claim_id, cả hai trả được. Validator này KHÔNG ép uniqueness claim_id
>   on-chain." Trách nhiệm hiện đẩy sang đường nạp (đội backend) — beacon đóng on-chain.

## 3.1 Mẫu ĐÚNG: `state_nft` (KHÔNG copy `lamp_policy`/`magic_policy`)

**Chốt kỹ thuật:** beacon PHẢI theo mẫu **singleton state NFT** của
`lib/phoenixkey/state_nft_logic.ak`, KHÔNG theo mẫu `lamp_policy.ak`/`magic_policy.ak`
(so sánh KIẾN TRÚC minting-policy — `lamp_policy.ak` còn sống trong Validator; `magic_policy.ak`
đã bị xoá 2026-07-01 theo quyết định MAGIC=account-trong-vault, giữ tên ở đây làm ví dụ lịch sử
cho mẫu authority-sig, KHÔNG phải file đang tồn tại).

- `state_nft_logic` = **one-shot thật**: mint đúng +1 mỗi tx, `asset_name` khoá mật mã vào
  danh tính (`asset_name == blake2b_256(did)`), chống bundle (`has_non_unit → None`), một
  carrier output duy nhất. Đây là uniqueness on-chain.
- `lamp_policy`/`magic_policy` = **authority-sig** (kiểm chữ ký có thẩm quyền mint), KHÔNG
  ép singleton-per-key. Copy nó KHÔNG đóng được double-payout — cùng claim_id vẫn mint được
  nhiều lần. Sai mẫu.

## 3.2 Đặc tả beacon (viết mới — `protectme_beacon.ak` + logic thuần)

Bám sát `state_nft_logic.validate_mint` + `find_single_minted_did`:

```
policy protectme_beacon(config: ProtectmeConfig)   -- own_policy ≡ own script-hash

redeemer BeaconRedeemer {
  MintClaim   -- nạp escrow: đúng +1, asset_name = claim_id, carrier = escrow UTxO
  BurnClaim   -- Payout|Reject: mọi movement của policy này < 0 (pure burn)
}

validate_mint(redeemer, own_policy, self):
  MintClaim ->
    expect Some((name, escrow_out)) = find_single_minted_beacon(self, own_policy)
      -- đúng +1 dưới own_policy, KHÔNG bundle (+2/burn kèm ⇒ None), carrier duy nhất
    expect InlineDatum(d): ClaimDatum = escrow_out.datum
    and {
      name == d.claim_id,                          -- asset_name khoá vào claim_id
      escrow_out.address.payment_credential == Script(payout_own_cred),  -- carrier = escrow
      quantity_of(escrow_out.value, cfg.carp_policy, cfg.carp_name) == d.amount,
      -- PROVENANCE (đóng #6): CARP nạp escrow đến TỪ bucket(cause), không tự-gửi.
      -- Kiểm có input chi tại bucket_cred(d.cause) trong CÙNG tx (release-gate B.6):
      spends_from_bucket(self, bucket_cred(d.cause, cfg)),
      -- committee duyệt lúc nạp (họ set datum + release bucket):
      committee_ok(self, cfg),
    }
  BurnClaim -> all_movements_negative(self, own_policy)
```

**Ghép với payout/reject (THÊM 1 mệnh đề mỗi bên):**
- `payout_ok` + `reject_ok`: THÊM yêu cầu **beacon `(own_policy, d.claim_id)` bị BURN
  (−1) trong cùng tx** (`quantity_of(tx.mint, own_policy, claim_id) == -1`). Escrow chỉ chi
  được nếu đốt beacon → mỗi claim_id chi **đúng một lần** (đóng F3).
- Escrow chỉ **tồn tại** nếu có beacon (mint gate ép carrier=escrow + provenance bucket) →
  không tạo được escrow-giả (đóng #6).

## 3.3 Vòng đời beacon

```
[nạp escrow]  MintClaim: +1 beacon(claim_id) ─ carrier = escrow UTxO
              provenance: input từ bucket(cause) + committee ký  (đóng #6)
      │
      ▼
[escrow sống]  UTxO @ protectme_payout giữ {amount CARP, beacon NFT, ADA}
      │
      ├── Payout ─► BurnClaim: −1 beacon ; amount→payee ; record→record_cred  (đóng F3)
      │
      └── Reject ─► BurnClaim: −1 beacon ; escrow→bucket(cause)               (đóng F3)
```

> **Ghi chú 1-hash:** hiện `ProtectmeConfig` là apply-param cho payout validator. Beacon
> cần biết `payout_own_cred` (carrier check) → sinh phụ thuộc vòng (giống state_nft↔taad).
> Giải như state_nft: beacon là **policy độc lập** (own script-hash riêng), KHÔNG hard-bind
> địa chỉ payout lúc compile; thay vào đó `payout_own_cred` đưa vào qua `ProtectmeConfig` +
> payout validator kiểm beacon-policy-id từ config. **Refactor 1-hash + config-reference-input
> làm SAU — KHÔNG chặn nếu beacon đã đóng #2/#6/F3.**

---

# 4. Schema datum / redeemer (ĐÃ CÓ + THÊM) — khuôn CBOR

Thứ tự field CỐ ĐỊNH, khớp `aiken` ↔ `rust_core`.

## 4.1 `ClaimDatum` (escrow, ĐÃ CÓ — `protectme_types.ak`)

```
ClaimDatum {
  claim_id      : ByteArray,   -- ID claim; = asset_name beacon (THÊM ràng buộc)
  did           : ByteArray,
  loss_eligible : Int,         -- net rời ví-theo-DID, committee chứng thực (A.6/B.5)
  cause         : Cause,       -- SystemFlaw | UserNegligence
  approved_slot : Int,         -- mốc committee chốt Approved (tính challenge window)
  amount        : Int,         -- CARP sẽ chi; == expected_amount(...) + == escrow CARP
  policy        : PolicySnapshot,
  payee_cred    : ByteArray,   -- Ví Phượng hoàng claimant (payout chảy về)
  record_cred   : ByteArray,   -- nơi ghi ClaimRecord bất biến (T2)
}

PolicySnapshot {
  cap_sys, cap_user      : Int,   -- trần theo cause (CARP)
  copay_bps              : Int,   -- 0..10000
  deductible             : Int,
  single_factor          : Bool,  -- C.8: không có second-factor độc lập với seed
  anti_drain_enrolled    : Bool,  -- C.8: điều kiện phủ USER_NEG cho single-factor
}
```

## 4.2 `ClaimRecord` (T2, ĐÃ CÓ)

```
ClaimRecord { claim_id, did, amount, cause }   -- output bất biến tại record_cred
```

## 4.3 Redeemer (ĐÃ CÓ)

```
ProtectmeRedeemer { Payout | Reject { reason: Int } }
```

## 4.4 `ProtectmeConfig` (apply-param, ĐÃ CÓ + THÊM cho beacon)

```
ProtectmeConfig {
  committee_vkhs        : List<VerificationKeyHash>,  -- M-of-N phân xử
  committee_threshold   : Int,
  carp_policy           : PolicyId,
  carp_name             : AssetName,
  pot_system_cred       : ByteArray,   -- bucket SYSTEM_FLAW
  pot_user_cred         : ByteArray,   -- bucket USER_NEGLIGENCE
  t_wait_payout         : Int,
  challenge_window      : Int,
  incident_cap          : Int,
  require_antidrain_for_single_factor : Bool,   -- I-PROT-CFG (C.8, CHỐT maintainer = True)
  -- THÊM (beacon):
  -- beacon_policy       : PolicyId,   -- policy-id beacon one-shot (đóng #6/F3)
}
```

> **Ghi trung thực:** `beacon_policy` hiện là **comment** (`protectme_types.ak` khu THÊM),
> chưa phải field thật. `require_antidrain_for_single_factor` thì ĐÃ có (khớp I-PROT-CFG).

## 4.5 Kiểu THÊM cho beacon

```
BeaconRedeemer { MintClaim | BurnClaim }   -- THÊM, §3.2
```

---

# 5. Luồng end-to-end (nạp → claim → phân xử → payout → burn)

Off-chain (backend, đội backend) dàn dựng; validator ép trên chain. Năm tx:

## Tx-A — nạp escrow (release bucket → escrow, MINT beacon)

- **Input:** UTxO bucket(cause) tại `pot_system_cred`|`pot_user_cred` (đủ `amount` CARP).
- **Mint:** `+1` beacon `(beacon_policy, claim_id)` — redeemer `MintClaim`.
- **Output:** escrow UTxO @ `protectme_payout` = `{ amount CARP, beacon NFT, min-ADA }`,
  inline `ClaimDatum` (đã DUYỆT, `approved_slot` = now).
- **Ký:** committee ≥ threshold (họ duyệt + set datum + release bucket).
- **Ép:** beacon mint gate (§3.2) — provenance bucket + carrier=escrow + `name==claim_id`
  + `escrow CARP == amount`; release-gate Treasury (đội backend).
- **CHỜ CHỐT:** input đến từ bucket (B.6) VS escrow-tự giữ (§8.2).

## Tx-B — mở claim (off-chain, KHÔNG on-chain)

- Backend đóng gói `ClaimPacket` (B.1), đọc anti-drain log + vault UTxO scan,
  tính `loss_eligible` (A.6/B.5). Triage cause. **Không** validator — chỉ resolver/indexer.

## Tx-C — phân xử (off-chain quyết, on-chain ở Tx-A/Tx-D)

- Committee/panel chốt `cause` + `amount = expected_amount(...)`. Kết quả **vào datum**
  của escrow (Tx-A). Panel USER_NEG + Security Committee SYS (B.4). Không tx riêng
  — quyết định thể hiện qua việc committee ký Tx-A (nạp) hoặc Tx-E (reject).

## Tx-D — PAYOUT (chi + burn beacon)

- **Input:** escrow UTxO (redeemer `Payout`); **Mint:** `−1` beacon (redeemer `BurnClaim`).
- **Output:** (1) `payee_cred` ← TOÀN BỘ `escrow_val` trừ beacon (amount CARP + ADA);
  (2) `record_cred` ← `ClaimRecord` bất biến (T2).
- **Ký:** permissionless-after-window (payee+amount cố định).
- **Ép:** P1–P10 (§2.2) + burn beacon (§3.2). `validity_range.lower_bound ≥ approved_slot
  + t_wait_payout + challenge_window` (P7).

## Tx-E — REJECT (hoàn bucket + burn beacon)

- **Input:** escrow (redeemer `Reject{reason}`); **Mint:** `−1` beacon.
- **Output:** `bucket_cred(cause)` ← TOÀN BỘ `escrow_val` trừ beacon (NO-CROSS-BUCKET).
- **Ký:** committee ≥ threshold. **Ép:** R1–R3 (§2.3) + burn beacon.

## Tx-F — subrogation (sau chi, off-chain→Treasury)

- Truy hồi tài sản kẻ trộm → nạp về đúng bucket qua đường thu Treasury (đội backend). Không
  validator payout — thuộc Treasury release/deposit (I-PROT-SUBROGATION).

## Bảng ai ký (tổng hợp)

| Tx | Hành động | Ai ký | Ép bởi |
|---|---|---|---|
| A | nạp escrow + mint beacon | committee ≥ threshold | beacon mint gate + Treasury release |
| D | payout + burn beacon | permissionless-after-window | `payout_ok` (P1–P10) + burn |
| E | reject + burn beacon | committee ≥ threshold | `reject_ok` (R1–R3) + burn |
| F | subrogation | đường thu Treasury (đội backend) | Treasury deposit |

---

# 6. API backend (tham chiếu)

> Prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`. Đây là **đề
> xuất** — backend PhoenixKey = ranh giới đội backend, tài liệu này KHÔNG code. Endpoint dưới đi
> cùng resolver + hai bucket (CHỜ CHỐT, xem §8).

| Method | Path | Vai trò |
|---|---|---|
| POST | `/api/v1/protectme/enroll` | bật bảo vệ, chọn `coverage_cap`, tính premium breakdown (Feecover wire) |
| GET | `/api/v1/protectme/quote?did=&cap=` | báo giá premium (risk_mult breakdown) |
| POST | `/api/v1/protectme/claim` | mở claim; resolver auto-điền `loss_eligible` từ anti-drain log |
| GET | `/api/v1/protectme/claim/{claim_id}` | trạng thái claim (triage → adjudicate → challenge → paid) |

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md#protectme)

---

# 7. Ranh giới đội

| Khối | Chủ | Phạm vi | File / neo |
|---|---|---|---|
| **Validator payout** | **đội on-chain** | escrow gate: `payout_ok`/`reject_ok`, single-factor tunable, T2, cred-collision, double-satisfaction | `protectme_payout.ak`, `protectme_logic.ak`, `protectme_types.ak` |
| **Beacon one-shot (THÊM)** | **đội on-chain** | `protectme_beacon` policy + logic, mint/burn gate, provenance bucket | `protectme_beacon.ak` (viết mới) |
| **Escrow ↔ bucket** | **đội backend** | release bucket→escrow lúc duyệt (B.6), committee-gate Treasury, ghi claim-record deposit | `LAMP/Treasury` |
| **Hai bucket** | **đội backend** | `protectme_pot_system` + `protectme_pot_user`, DAO-gated release (T1 Model A), KHÔNG sổ phụ | `LAMP/Treasury/CONTRACT.md` |
| **Feecover-integration** | **đội backend** | wire `service_id="protectme.*"`, `FeecoverAccrual`→`FeecoverEpochSettle`→bucket, `app_id="feecover"`, grace/`last_paid_epoch`; KHÔNG `collectToTreasury` | `PhoenixKey-Feecover` |
| **Resolver/indexer claim** | **đội backend** | trạng thái ClaimPacket, đọc anti-drain log + vault UTxO scan, tính `loss_eligible` | backend PhoenixKey |
| **UI** | **Core/SuperApp** | màn Protectme (bật/cap/premium breakdown), mở claim (auto-điền từ anti-drain log), trạng thái claim | app |
| **Governance committee/panel** | **DAO/maintainer** | thành phần Security Committee + Adjudication Panel, ngưỡng vote, recuse/challenge | LAMP/Governance |

> Ranh giới code: **backend PhoenixKey = đội backend, tài liệu này KHÔNG sửa.** Validator = đội on-chain,
> không tự merge; phát hiện lỗi → báo maintainer / Issue giao đội backend/on-chain.

---

# 8. Thứ tự deploy + phụ thuộc CHẶN

## 8.1 🔴 CHẶN-MERGE — beacon one-shot per claim_id (THÊM)

- **Chặn:** không đóng được **#6 (escrow-giả)** + **F3 (double-payout)** nếu thiếu beacon.
- **Yêu cầu:** mẫu `state_nft` (singleton, `asset_name=claim_id`, policy≡own-hash),
  KHÔNG `lamp_policy`/`magic_policy` (authority-sig, không one-shot). §3.
- **Hành động:** đội on-chain viết `protectme_beacon.ak` + ghép burn vào `payout_ok`/`reject_ok` +
  thêm `beacon_policy` vào `ProtectmeConfig`. Sau đó merge.

## 8.2 🟡 CHỜ CHỐT — escrow-tự giữ VS release-from-bucket (B.6)

- **Lệch:** spec **B.6** = release-from-bucket qua Treasury (mọi CARP ra bucket có
  claim-record đối ứng, T2). **Mã hiện tại** = escrow-tự giữ (validator giả định escrow "đã
  nạp hợp lệ", chỉ kiểm `escrow_carp == amount`, KHÔNG kiểm nguồn bucket).
- **Chờ:** **đội backend + maintainer** chốt hướng nạp. Nếu chốt B.6 (release-from-bucket) → beacon
  provenance (§3.2 `spends_from_bucket`) trở thành ràng buộc bắt buộc; nếu escrow-tự giữ
  đứng → provenance nới nhưng #6 chỉ đóng một phần.
- **KHÔNG code** đường nạp tới khi chốt — chỉ chuẩn bị beacon để cả hai hướng đều dùng được.

## 8.3 🟡 CHỜ CHỐT — 1-hash + config-reference-input

- **Refactor SAU:** gộp payout+beacon về 1-hash + đọc config qua reference-input thay
  apply-param. **KHÔNG chặn** nếu beacon đã đóng #2/#6/F3. Beacon làm policy độc lập trước
  (§3.3 ghi chú), gộp sau.

## 8.4 🟢 Rẻ, trước-merge — siết P9 (stake_credential + payee VerificationKey-cred)

- **Không chặn-merge** như beacon, nhưng nên làm **trước-merge** vì rẻ (không đổi kiến trúc).
- Chi tiết: §2.5.

## 8.5 Phụ thuộc MAGIC/CARP ([PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md) §2)

- **CARP** = đơn vị chi/thu; `carp_policy`/`carp_name` trong `ProtectmeConfig`. Phụ thuộc
  CARP là `accepted_asset` Phoenix Treasury (đội backend xác nhận policy-id thật lúc deploy).
- **MAGIC** = nhãn giá premium, tiêu Vault `BurnBatch` (KHÔNG token/mint) — **ngoài**
  validator payout; thuộc Feecover premium wiring (đội backend). Validator payout KHÔNG chạm MAGIC.
- **Premium** qua Feecover (`service_id="protectme.*"`, `app_id="feecover"`) → bucket
  `protectme_pot_user` qua `FeecoverEpochSettle`, KHÔNG `collectToTreasury`.

## 8.6 Phụ thuộc bằng chứng (đọc, không đổi)

- **Anti-drain log** (Withdrawal-Limit) + **vault UTxO scan** = oracle `loss_eligible`
  (A.6/B.5). Resolver đội backend **đọc**, KHÔNG đổi validator custody. `loss_eligible`/`policy`
  vào `ClaimDatum` do committee chứng thực off-chain (không oracle on-chain, C.6/F6).

---

# 9. Ranh giới kiểm chứng + Ghi trung thực (ĐÃ CÓ vs THÊM vs CHỜ CHỐT)

Hạng mục chặn-merge kỹ thuật thuần DUY NHẤT là **beacon one-shot** (§3); còn lại là CHỜ CHỐT
chính sách (maintainer + đội backend + DAO). Neo kiểm chứng theo hạng mục:

| Hạng mục | Neo |
|---|---|
| `payout_ok` (P1–P10) | `protectme_logic.ak` |
| `reject_ok` (R1–R3) | `protectme_logic.ak` |
| `expected_amount` / `single_factor_ok` / `policy_domain_ok` | `protectme_logic.ak` |
| `ClaimDatum`/`ClaimRecord`/`ProtectmeConfig`/redeemer | `protectme_types.ak` |
| committee M-of-N dedup (F7) | `protectme_logic.ak:committee_ok` |
| double-satisfaction / cred-collision / bảo toàn value | `count_inputs_at`, `sole_output_value_at` |
| provenance escrow (CARP từ bucket) | `protectme_logic.ak` (F3 header) — cần beacon |
| uniqueness claim_id on-chain (#6/F3) | `protectme_beacon.ak` (viết mới, §3) |
| beacon burn ghép vào payout/reject | §3.2 |
| siết P9 (stake_credential + payee VerificationKey-cred) | `protectme_logic.ak:payout_ok` (§2.5, §8.4) |
| escrow ← bucket (B.6) | đội backend + maintainer (§8.2) |
| hai bucket Treasury + release-gate | đội backend (§7) |
| premium/Feecover wiring | đội backend (§8.4) |
| resolver claim (`loss_eligible`) | đội backend (§8.5) |
| 1-hash + config-reference-input | §8.3 |
| tham số kinh tế (caps, rates, thời gian) | [PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md) §11 [PROT-1..11] |

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](https://github.com/PhoenixKeyDID/PhoenixKey-Anchorme-Specs/blob/main/PhoenixKey-STATUS.md#protectme)

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai).
- Code: `PhoenixKey-Validator` nhánh `feat/protectme-payout` — `protectme_{types,logic}.ak`
  + `validators/protectme_payout.ak` + mẫu `state_nft_logic.ak`.
- Tài liệu cùng bộ: [PhoenixKey-Protectme-Vi-Feat.md](./PhoenixKey-Protectme-Vi-Feat.md), [PhoenixKey-Protectme-Math.md](./PhoenixKey-Protectme-Math.md), [PhoenixKey-Protectme-Exec.md](https://github.com/PhoenixKeyDID/PhoenixKey-Protectme-Specs/blob/main/PhoenixKey-Protectme-Exec.md).

*Bản kỹ thuật này bám branch `feat/protectme-payout`. Không normative tới khi maintainer
duyệt + hợp thức vào PhoenixKey-Specs.*

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
