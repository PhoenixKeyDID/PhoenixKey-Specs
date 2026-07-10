# PhoenixKey — Rebirthme — Kỹ thuật (Tech)

> **Module:** Rebirthme (slug `Rebirthme`). **Loại doc:** Kỹ-thuật cho implementer (đội on-chain / đội backend / Core rust_core). **Ngày:** 2026-07-09.
> **Đối tượng đọc:** kỹ-sư triển-khai. HOW: kiến-trúc, datum/redeemer CBOR, điều-kiện tx, luồng e2e, API, ranh-giới giao-việc, thứ-tự deploy, test.
>
> **Ranh giới (MECE):** module CHI + địa-chỉ + anti-drain + kho bí-mật/phả-hệ + stake theo-DID + connector + di-cư + pool-ops. **Safesend** (gửi-có-bảo-vệ) nay là module độc-lập — xem [PhoenixKey-Safesend-Tech.md](./PhoenixKey-Safesend-Tech.md); module này chỉ cung-cấp hạ-tầng nạp-nguồn/guardian/anti-drain mà Safesend tái-dùng. **TAAD state-machine (genesis/rotate/recovery-mechanics)** thuộc Core Anchorme — chỉ dẫn-chiếu. Xem [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) cho bất-biến, `PhoenixKey-Math.md` §10/§11 cho TAAD.
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 1. Kiến-trúc + sơ-đồ thành-phần

```
┌──────────────────────── PhoenixKey-Core (Flutter + rust_core) ─────────────────────┐
│  UI: Ví Phoenix/Standard · Guardian · Khôi-phục · Kho bí-mật · Hạn-mức · Safesend   │
│  rust_core (FFI): did_payment builder · lampnet ECIES · sign/crypto · migration/    │
│                   claim builder · pool KES · vault_frame                            │
└───────────────┬────────────────────────────────────────────────┬───────────────────┘
                │ FFI                                             │ CIP-30 (WebView)
                ▼                                                 ▼
┌──────────────────────────────┐                    ┌────────────────────────────┐
│ PhoenixKey-Validator (Aiken) │                    │  Ví ngoài (Lace/Eternl)    │
│  did_payment.ak               │ ref-input (CIP-31) │  watch-only + signTx        │
│  auth_logic.ak · taad_logic   │◄───────────────────┤                            │
│  limit_meter.ak              │                    └────────────────────────────┘
│  did_stake.ak · did_subaddr  │
│  safesend_escrow.ak          │
└──────────────┬───────────────┘
               │ resolver / index
               ▼
┌──────────────────────────────┐        ┌────────────────────────────┐
│ PhoenixKey-Database (Java)   │        │  LampNet (erasure + Strata)│
│  resolver /identifiers/{did} │◄──────►│  ECIES ciphertext, shard   │
│  wallet API · stake-state    │  CID   │  recovery_anchor/vault_idx │
│  consent/Grant · merkle-serve│        └────────────────────────────┘
└──────────────────────────────┘
```

Nền tài-sản: Ví Phượng-hoàng = **địa-chỉ script Plutus V3 stateless** (UTxO không datum). Anti-drain thêm lớp **meter-UTxO stateful** (khi bật). Khôi-phục + guardian = **UTxO anchor TAAD** (thuộc Core). Backup = **CID neo trong `TAADDatum`** trỏ ciphertext trên LampNet.

---

## 2. Bất-biến kiến-trúc (load-bearing)

1. **Authority đọc động qua anchor ref-input** (CIP-31): mọi cổng chi (did_payment/did_stake/did_subaddr/limit_meter) quy về `anchor_controller_ok` — đọc controller HIỆN-TẠI, không hardcode. → tài-sản sống qua rotate (I-WALLET-1/2).
2. **Singleton-anchor**: đúng 1 ref-input mang anchor NFT; ≠1 → reject (I-WALLET-4, `auth_logic.ak:59-70`).
3. **Đóng-băng theo status**: status ≠ Active → mọi chi reject (I-WALLET-3).
4. **Helper canonical dùng chung**: `auth_logic.anchor_controller_ok` (mode-1) / `auth_satisfied` (mode-2 registry, chờ tách) — did_payment/did_stake/did_subaddr/pool KHÔNG sao-chép logic anchor.
5. **Meter binding**: khi bật hạn-mức, kho ÉP tiêu-thụ + tái-tạo meter-NFT UTxO (I-LIMIT-BIND) → không đường rút "ngoài đồng-hồ".
6. **Client-side crypto**: mọi bí-mật mã-hoá TRÊN MÁY trước khi rời (I-VAULT-1/3); node chỉ thấy ciphertext.
7. **Tách miền khoá**: P-256 (auth/login, off-chain verify) ≠ Ed25519 (controller on-chain); lỗi nonce miền auth không chạm tài-sản (I-SIGN-DOMAIN-SEP).

---

## 3. Datum / Redeemer — khuôn CBOR

### 3.1 `did_payment` (stateless)
```
validator did_payment(anchor_nft_policy: PolicyId, anchor_nft_name: AssetName)
  spend(_datum: Option<Data>, _redeemer: DidPaymentRedeemer, _own_ref, self)
DidPaymentRedeemer = Spend           -- redeemer trống; authority neo vào anchor ref-input
```
UTxO ví KHÔNG cần datum. Params apply-per-DID: `(anchor_policy, blake2b_256(did))`. Nguồn: `did_payment.ak:36-55`.

### 3.2 `TAADDatum` (dẫn-chiếu Core Anchorme — các field liên-quan module này)
Thứ-tự field khớp `types.ak` (KHÔNG sửa — thuộc Core/Validator). Field module này ĐỌC: `did`, `entity_type`, `controller_pkh` (Ed25519, auth chi), `hw_key_pubkey` (P-256, carry-by-equality), `status`, `guardians` (≤5), `recovery_anchor` (CID). **Đề xuất thêm** (schema thuộc đội backend/Validator — [CẦN CHỐT-W3]): `vault_index_anchor : Option<CID>`, `pool_anchor : Option<CID>`, `GuardianConfig` (trọng-số/vai + `schema_version`).

### 3.3 `LimitMeterDatum` (Anti-Drain §B.2)
```
LimitMeterState = Open | Frozen
LimitMeterDatum {
  did: ByteArray, limit_lovelace: Int, refill_window: Int, bucket_Q: Int,
  last_refill_slot: Int, state: LimitMeterState, safe_address_hash: ByteArray,
  large_delay: Int, secondary_pkh: Option<VKeyHash>, schema_version: Int }
```
Q = 10^9. Meter gắn meter-NFT singleton per-DID.

### 3.4 `SafeSendDatum` — nay module riêng
> `SafeSendDatum` + redeemer (Cancel/Accept/Finalize/Freeze/ReclaimTimeout) chuyển sang `PhoenixKey-Safesend-Tech.md §3.1`.

### 3.5 `SecretBlob` / `VaultIndex` (Secret-Vault §B.1/B.3)
```
SecretBlob { kind: bip39_seed|raw_key<chain>|document|note, payload, metadata{label,data_class,created_slot,orig_size,fingerprint} }
frame(sb,decoy) = pad_to_bucket(serialize(sb) ‖ decoy)   -- B=4KiB, FLOOR=64KiB
VaultIndex { entries: [{label,kind,data_class,cid,created_slot,fingerprint}] }
```

### 3.6 On-ramp mandate (Connect-Onramp §1)
```
AuthorizationMandate = Grant{ wallet_address, phoenix_did, scope:"lamp_program",
  program_ids, nonce, valid_from/until, domain, cose_sign1 }
```
Ký bằng khoá ví ngoài (COSE_Sign1 Ed25519, CIP-8), KHÔNG controller.

---

## 4. Từng thao-tác — điều-kiện + shape tx + ai-ký

| Thao-tác | Validator | Điều-kiện | Ai ký |
|---|---|---|---|
| **Chi Ví Phoenix** | `did_payment` | anchor Active + controller ký + ∃!anchor ref-input | controller (Ed25519) |
| **Rotate/Recovery** (Core) | `taad` | xem `taad_logic.ak`; trong Recovering ví đóng-băng | controller / guardian / pending |
| **Rút nhỏ hạn-mức** | `did_payment`+`limit_meter` | `W(tx)×Q ≤ available_Q`; tiêu+tái-tạo meter | controller |
| **Rút lớn L1** | `limit_meter` | OpenLarge (W=0) → chờ `large_delay` → FinalizeLarge; CancelLarge bất-cứ lúc nào | controller |
| **Rút lớn L2** | `limit_meter` | `W×Q > available`; +secondary_pkh HOẶC guardian ký; đốt bucket | controller + secondary/guardian |
| **Freeze/Unfreeze** | `limit_meter` | Freeze: controller∨guardian; Unfreeze: chỉ controller | controller/guardian |
| **SetConfig nới** | `limit_meter` | 🔴 chịu `large_delay` (I-LIMIT-LOOSEN-DELAY); siết áp ngay | controller |
| **Safesend** (Open/Cancel/Accept/Finalize/Freeze/ReclaimTimeout) | `safesend_escrow` | nay module riêng — xem [PhoenixKey-Safesend-Tech.md §4](./PhoenixKey-Safesend-Tech.md) | sender/receiver/guardian |
| **Stake delegate** | `did_stake` | anchor Active + controller ký; VoteDelegation Abstain kèm (Plomin) | controller |
| **Di-cư ví cũ** | (không validator mới) | 1 tx gộp 4 cert Conway; tự-cấp-phí từ reward | ví ngoài (CIP-30) |
| **Claim LAMP** | `airdrop_pool` (LAMP) | Merkle permissionless; leaf khớp; burn slot | controller HOẶC Lace (fee) |
| **Pool create / rotate KES** | (off-chain build + TAAD auth) | `auth_satisfied(did, pool_*)`; counter monotonic | controller (biometric) |

**Shape rút-nhỏ hạn-mức:** inputs = {vault UTxO(s), meter UTxO} ; outputs = {đích, [trả-lại-kho], meter UTxO tái-tạo với bucket'} ; ref-inputs = {anchor} ; signer = controller ; validity `lo` = tip slot.

---

## 5. Luồng end-to-end

### 5.1 Chi thường
`build tx` → thêm anchor ref-input + controller required-signer → ExUnits tĩnh (evaluate-then-patch) → collateral thuần-ADA → sign (Enclave) → submit.

### 5.2 Khôi-phục mất máy (guardian)
Máy mới sinh controller mới → `buildInitRecovery` (đủ t-guardian ký, collateral 50 ADA, mở timelock 3600 slot) → chờ → `buildFinalizeRecovery` (controller mới ký) → tài-sản ở Ví Phoenix tự về (địa-chỉ bất-biến). Nghi lạm-dụng trong cửa-sổ → `buildCancelRecovery`.

### 5.3 Backup kho bí-mật
`SecretBlob` → `vault_frame` (pad+decoy) → `ECIES_Encrypt(pk_vault)` → `LampNet.put` → CID → cập-nhật `VaultIndex` → neo `vault_index_anchor` (controller ký). Khôi-phục: 24 từ → Master_KEK → VaultKEK → đọc `vault_index_anchor` → giải từng blob (fail-closed).

### 5.4 Di-cư ví cũ
PhoenixKey enable() Lace (watch-only) → `getRewardAddresses/getUtxos` → `taad_build_legacy_migration_tx` (gộp VoteDelegation→Abstain + Withdrawal + Move→phoenix_addr + StakeDeregistration) → **UNSIGNED** → Lace `signTx` → `assemble_witness` → `submitTx`. Reward < fee → builder trả "" + báo "cần nạp X ADA".

### 5.5 Xoay KES pool
Nhắc lịch (75%/90% tuổi KES) → 1 chạm biometric → sinh KES period-now → ký op-cert mới (counter+1) → re-distribute LampNet TRƯỚC → xuất artefact node (KES+VRF+op-cert, KHÔNG cold).

---

## 6. API backend (tham-chiếu — prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`)

| Endpoint | Việc | Nguồn |
|---|---|---|
| `GET /identifiers/{did}` | resolver DID Document (payment/stake, recovery_anchor, vault_index_anchor) | live + mở rộng service[] |
| `POST /wallet/standard/register` · `GET /wallet/standard/{did}` | đăng-ký + đọc ví Standard (CIP-1852 fixed/active/stake) | Wallet-API-v2 §2 |
| `GET /wallet/{did}/all` | gộp mọi ví + magic{source:vault} | Wallet-API-v2 §3 |
| `PUT/GET/POST /consent/*` | Grant store (route_stake_reward, cip30 session); point-in-time controller | Delegator-Offchain §1 |
| `GET /stake-state/{stake_cred}` | registered/vote_delegated/reward/pool (Blockfrost) | Delegator-Offchain §2 |
| `POST /claim/submit` · `GET /claim/{id}/status` | submit claim (KHÔNG ký) + theo-dõi | Delegator-Offchain §3 |
| `GET /airdrop-claim/{addr}/{epoch}/merkle-proof` | serve proof (khớp merkle.ak byte) | Delegator-Offchain §4 |
| `POST /connect/mandate` · `/connect/challenge` · `/connect/epoch-proof` | on-ramp chữ-ký | Connect-Onramp §7 |
| `GET /v1/telemetry/map/:cid` | bản-đồ shard cho UI kho/khôi-phục | UI-Spec S8 |

Lưu-ý: `/wallet/magic/claim` → **410 Gone** (MAGIC = account-trong-Vault, không native). `BalanceResponse` field MAGIC = 0 (deprecated).
→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 7. Ranh-giới giao-việc

| Team | Việc |
|---|---|
| **đội on-chain** | `limit_meter.ak` (I-LIMIT-*) + sửa `did_payment.ak` thêm nhánh binding meter (I-LIMIT-OPTIN); `did_stake.ak` (I-DIDSTAKE-*); `did_subaddr.ak` (I-ADDR-UNLINK, **chờ maintainer chốt DEP-2**); helper `W(tx)`. **KHÔNG sửa `did_payment.ak` mode-1** ngoài nhánh opt-in. Builder Delegator (migration/claim) + 2 FFI on-ramp (`taad_cose_sign1_verify`, `taad_addr_matches_pubkey`). Pool KES (rust_core, **[VERIFY] crate KES/VRF**). (`safesend_escrow.ak` (SS-*) nay ở module Safesend — [PhoenixKey-Safesend-Tech.md §7](./PhoenixKey-Safesend-Tech.md).) |
| **đội backend** | resolver mở rộng `service[]` (L1/L2, chặn cứng L3 — I-ADDR-PRIV); schema `vault_index_anchor`/`recovery_anchor`/`pool_anchor` + update-path (controller ký); wallet API v2; consent/Grant store (point-in-time controller); stake-state indexer; claim orchestration; merkle-serve; metering nanogic; telemetry passthrough; giám-sát `r` lặp (I-SIGN-NO-REUSE — low-s I-SIGN-LOWS ép ở `crypto.rs:339-349`). |
| **Core (rust_core/Flutter)** | builder chi-có-meter + freeze/SetConfig; `vault_frame`/`vault_kek`/VaultIndex + FFI kho; export re-key (`exportRevokesSeed=true` mặc-định — I-WALLET-6); màn Guardian/Khôi-phục/Kho/Hạn-mức/Phả-hệ; CIP-30 connector; pool KES builder. Enforce I-CURVE-5 (guardian/secondary khác gốc seed). (Màn + builder Safesend nay ở module Safesend — [PhoenixKey-Safesend-Tech.md §7](./PhoenixKey-Safesend-Tech.md).) |
| **VeData / Glint (ZK)** | VC-Glint recovery. (Factor bối-cảnh Safesend nay ở module Safesend — [PhoenixKey-Safesend-Tech.md §7](./PhoenixKey-Safesend-Tech.md).) |
| **LampNet** | Strata engine + `GET /audit`; Mirage `EncryptedDistributed` (durability I-VAULT-8); adapter neo. |
| **MAGIC / CARP** | tokenomics nanogic, giá/nguồn trả phí kho (ngoài phạm vi module). |
| **Core Anchorme** | TAAD state-machine, genesis/rotate/recovery-mechanics, GuardianConfig schema, PA2/PA5 — module này CHỈ dẫn-chiếu. |

---

## 8. Thứ-tự deploy + phụ-thuộc-chặn

**Build được không chờ onchain mới:**
1. Mở rộng resolver `service[]` (L1/L2 payment part) + I-ADDR-PRIV chặn sub-addr.
2. Wallet API v2 (Standard register, `/all`, deprecate MAGIC).
3. Kho bí-mật blob đơn (nền `lampnet.rs`) — Phase 1.

**Chờ dependency ONCHAIN MỚI (thứ-tự khuyến-nghị, mỗi bước độc-lập giá-trị):**
1. **`limit_meter.ak`** (anti-drain) — **ưu-tiên CAO NHẤT** (I-CURVE-4 load-bearing). Cần sửa `did_payment.ak` thêm nhánh binding (opt-in).
2. **`did_stake.ak`** → stake part theo-DID + đa-ISPO [DEP-1].
3. **`did_subaddr.ak`** → L3 unlinkable [DEP-2] — **chờ maintainer chốt trước khi đội on-chain code**. Cùng validator mà Easteregg gọi "Tầng 0" (địa chỉ riêng) — xem `PhoenixKey-Easteregg-Tech.md §1`.
4. Registry-lib mode-2 (Org m-of-n) [DEP-3] — không chặn MVP single-controller.

(`safesend_escrow.ak` nay ở module Safesend — thứ-tự deploy + phụ-thuộc anti-drain (1) khai ở [PhoenixKey-Safesend-Tech.md §8](./PhoenixKey-Safesend-Tech.md).)

**Phụ-thuộc-chặn ngoài:** CARP policy-id (CARP team) cho balance; stake-state (đội backend) cho di-cư/claim; cây Merkle LAMP (LAMP team) cho claim; `vault_index_anchor` schema (đội backend) cho khôi-phục kho; crate KES/VRF Rust cho pool.

---

## 9. Test / evidence bắt-buộc khi land

- `did_payment.ak`: 8 test spend (a-e + rotate-survival + 2-anchor + datum-ignored) đối-chiếu I-WALLET-1..5, T-WALLET-1/2.
- `taad_logic.ak`: guardian Init/Cancel/Finalize/UpdateGuardians/Transfer + status-gate.
- `lampnet.rs`: round-trip / tamper / wrong-key fail-closed đối-chiếu I-VAULT-4.
- `sign.rs`: determinism Ed25519 + sign/verify đối-chiếu I-SIGN-ED25519-LIB (kèm test-vector RFC 8032).
- `crypto.rs`: P-256 verify low-s — reject high-s malleable + accept canonical low-s đối-chiếu I-SIGN-LOWS.
- `limit_meter.ak`: rút>available reject; chia-100-output né NET reject; tx-2-cùng-block tranh meter (double-spend) reject; meter-NFT giả reject; chi-kho-không-meter reject; SetConfig limit=∞ rồi rút reject (chịu delay); Frozen→đích≠safe reject; refill-dùng-`hi` reject. PASS: rút nhỏ trừ bucket đúng; L1 sau delay; L2 ký-kép; Cancel pending; Freeze→safe→Unfreeze.
- `did_stake.ak`: register-lặp, withdraw-no-vote (Plomin), anchor giả, Revoked đóng-băng.
- Pool: op-cert verify, counter tụt→reject, recover→khớp, sai KEK→fail-closed, KES period sai→reject.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## Nguồn

Nguồn thiết-kế nội-bộ (không công khai).
Code: `PhoenixKey-Validator/validators/did_payment.ak`, `lib/phoenixkey/{auth_logic,taad_logic,types}.ak`; `rust_core/src/{lampnet,crypto,sign}.rs`.
Tài-liệu cùng bộ: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Rebirthme-Exec.md](./PhoenixKey-Rebirthme-Exec.md).
