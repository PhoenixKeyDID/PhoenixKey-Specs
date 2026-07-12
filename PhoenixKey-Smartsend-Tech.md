# PhoenixKey — Smartsend — Kỹ thuật (Tech)

> **Module:** Smartsend (gửi có bảo vệ). **Loại doc:** Kỹ thuật cho implementer (đội on-chain / đội backend / Core rust_core / VeData-Glint). **Ngày:** 2026-07-09.
> **Đối tượng đọc:** kỹ sư triển khai. HOW: kiến trúc, datum/redeemer CBOR, điều kiện tx, luồng e2e, ranh giới giao việc, thứ tự deploy, test.
>
> **Ranh giới (MECE):** module CHỈ đặc tả hòm ký quỹ `smartsend_escrow` (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout). **Cổng chi `did_payment`, guardian-recovery, anti-drain `limit_meter`** thuộc module Rebirthme — chỉ dẫn chiếu. Xem [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) cho bất biến; [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md) cho ví/guardian/anti-drain; [PhoenixKey-Math.md](./PhoenixKey-Math.md) §10/§11 cho TAAD.

---

## 1. Kiến trúc + sơ đồ thành phần

```
┌──────────────────── PhoenixKey-Core (Flutter + rust_core) ────────────────────┐
│  UI: màn Gửi (công-tắc Smartsend) · Huỷ · Đồng-ý (Accept) · trạng-thái hòm     │
│  rust_core (FFI): smartsend builder (Open/Cancel/Accept/Finalize) · glint proof │
└───────────────┬───────────────────────────────────────────┬───────────────────┘
                │ FFI                                        │ ZK proof
                ▼                                            ▼
┌──────────────────────────────┐              ┌────────────────────────────┐
│ PhoenixKey-Validator (Aiken) │              │  VeData / Glint (ZK)       │
│  smartsend_escrow.ak           │◄── verify ───┤  Spectra + Glint context   │
│  did_payment.ak (nạp nguồn)   │  ref-input   │  proof (FaceMatch/Geo/...)  │
│  taad_logic.ak (guardian)     │◄─────────────┤                            │
└──────────────┬───────────────┘   anchor     └────────────────────────────┘
               │ resolver / index
               ▼
┌──────────────────────────────┐
│ PhoenixKey-Database (Java)   │   theo-dõi hòm mở, cửa-sổ-veto, trạng-thái consent
└──────────────────────────────┘
```

Nền: hòm Smartsend = **địa chỉ script Plutus V3 stateful** (UTxO mang `SmartSendDatum`). Nguồn nạp = chi từ Ví Phượng hoàng (`did_payment`, dẫn chiếu Rebirthme). Guardian Freeze + factor Cancel dựa anchor TAAD (thuộc Core).

---

## 2. Bất biến kiến trúc (load-bearing)

1. **Escrow-UTxO đơn**: mỗi khoản Smartsend = một UTxO tại `smartsend_escrow`; mọi đường thoát ép `count(inputs ∈ Script(smartsend)) == 1` (SS-7′) → không gộp hai hòm thoả mãn một output.
2. **Cửa sổ-veto rời nhau**: Cancel chỉ `now < veto_deadline` (ép cận trên validity-range), Finalize chỉ `now ≥ veto_deadline` (ép cận dưới) (SS-2) — đọc từ validity-interval, không tin cờ datum.
3. **Byte-perfect payout**: Cancel/ReclaimTimeout/ResolveFreeze-timeout ép `Σ→sender == amount + min_ada`, Finalize ép `Σ→receiver == amount + min_ada` (SS-1/SS-5′/SS-12) — không đổi hướng, không rút bớt, `fee_covered` KHÔNG vào biểu thức output.
4. **Factor neo anchor-enroll**: tập factor Cancel + cờ `independent_of_seed` neo trong anchor lúc enroll, KHÔNG tin datum lúc Open (SSR-4) → chống kẻ Open tự đặt factor dễ; mỗi factor phải **distinct FactorKind** + verify thật (SS-3/SSR-13).
5. **Authority nguồn + guardian đọc động qua anchor ref-input** (CIP-31): nạp hòm và Freeze quy về `anchor_controller_ok` / guardian-sig — dẫn chiếu `auth_logic.ak`, KHÔNG sao chép.
6. **Freeze có lối thoát bắt buộc**: Freeze chỉ dùng trong cửa sổ-veto; giải-Freeze qua guardian-quorum (m-of-n) hoặc `freeze_deadline` auto-hoàn sender — không kẹt vĩnh viễn (SS-8/SS-8′).
7. **`window` có sàn cứng**: `veto_deadline - open_slot ≥ min_window_floor` bất kể "2-bên thoả"; `open_slot` phải nằm trong validity-range của chính tx Open (SS-10).

---

## 3. Datum / Redeemer — khuôn CBOR

### 3.1 `SmartSendDatum`
```
SmartSendDatum {
  sender_commit, receiver: Address, receiver_commit, asset: (PolicyId,ByteArray),
  amount: Int, min_ada: Int,                                                  -- SSR-2/SS-12
  open_slot: Int, veto_deadline: Int, reclaim_deadline: Int,                  -- SSR-7
  freeze_deadline: Int, frozen: Bool,                                        -- SSR-6/SS-8′
  unlock_policy: UnlockPolicy, large_threshold: Int, receiver_consent: Bool,
  fee_covered: Int }                                                         -- audit-only, SS-12
Redeemer = Cancel{proofs} | Accept{receiver_sig} | Finalize
         | Freeze{guardian_sig} | ResolveFreeze{guardian_quorum_proofs}      -- SS-8′
         | ReclaimTimeout
```
`unlock_policy` chứa `factors_required` + tập factor + `independent_of_seed` (neo anchor-enroll, SSR-4). Hòm gắn asset đã chỉ định trong `asset`; `min_ada` là lovelace ký quỹ ghi lúc Open, tách khỏi `amount`. `fee_covered` là số audit — validator KHÔNG đọc field này để tính output (SS-12).

### 3.2 `TAADDatum` (dẫn chiếu Core Anchorme — field Smartsend đọc)
Smartsend ĐỌC (không sửa): `controller_pkh` (verify nguồn nạp + `receiver_commit` khi Accept), `guardians` (factor Guardian + SS-8/SS-8′ Freeze/ResolveFreeze quorum), enroll-set factor (SSR-4). Thứ tự field khớp `types.ak` — thuộc Core/Validator, KHÔNG sửa ở đây.

---

## 4. Từng thao tác — điều kiện + shape tx + ai ký

| Thao tác | Validator | Điều kiện | Ai ký |
|---|---|---|---|
| **Smartsend Open** | `smartsend_escrow` | tiêu UTxO `did_payment` → escrow; datum đủ (`open_slot`/`veto_deadline`/`reclaim_deadline`/`freeze_deadline`); `open_slot` trong validity-range tx Open, `window ≥ min_window_floor` (SS-10) | sender (controller) |
| **Smartsend Cancel** | `smartsend_escrow` | `now<veto_deadline` (cận trên); ≥`factors_required` distinct-kind (đa yếu tố neo anchor SSR-4, khác gốc seed SS-6); `Σ→sender==amount+min_ada` | sender + factor |
| **Smartsend Accept** | `smartsend_escrow` | verify controller-sig `receiver_commit` qua anchor; chỉ set `receiver_consent=true`, mọi field khác + value bất biến (SS-9′/SSR-14) | receiver (controller) |
| **Smartsend Finalize** | `smartsend_escrow` | `now≥veto_deadline` (cận dưới); consent nếu `amount≥large_threshold` (trừ đích ngoài-Phoenix); `Σ→receiver==amount+min_ada` (SS-5′) | (permissionless / Bob) |
| **Smartsend Freeze** | `smartsend_escrow` | `now<veto_deadline`; guardian-sig hợp lệ (neo anchor) → `frozen:=true`, chặn Finalize (SS-8) | guardian |
| **Smartsend ResolveFreeze** | `smartsend_escrow` | `frozen==true` ∧ (guardian-quorum m-of-n anchor.guardians ∨ `now≥freeze_deadline`) → `Σ→sender==amount+min_ada` (SS-8′) | guardian-quorum / permissionless sau `freeze_deadline` |
| **Smartsend ReclaimTimeout** | `smartsend_escrow` | `now≥reclaim_deadline ∧ receiver_consent==false`; `Σ→sender==amount+min_ada` (SS-11) | sender (permissionless sau hạn) |
| **Nạp nguồn (chi ví)** | `did_payment` | anchor Active + controller ký (dẫn chiếu Rebirthme) | controller |

**Shape Open:** inputs = {vault UTxO(s) `did_payment`} ; outputs = {escrow UTxO(`SmartSendDatum`), [trả lại ví]} ; ref-inputs = {anchor} ; signer = controller ; validity `lo` = tip slot → `open_slot`.

**Shape Finalize:** inputs = {escrow UTxO} ; outputs = {→receiver == amount + min_ada, byte-perfect} ; ref-inputs = {anchor nếu cần consent} ; validity `lo ≥ veto_deadline`.

---

## 5. Luồng end-to-end

### 5.1 Gửi có bảo vệ (Open → Finalize) — SS-10
`build Open` → tiêu UTxO `did_payment` (controller ký) → escrow UTxO với `open_slot=tip` (ép nằm trong validity-range tx Open, SSR-11), `veto_deadline=open_slot+window` (`window ≥ min_window_floor`, SSR-10), `reclaim_deadline`, `freeze_deadline` → submit. Hết cửa sổ, bất kỳ ai `build Finalize` (đích cố định →receiver, byte-perfect SS-5′) → tiền về người nhận.

### 5.2 Đổi ý (Cancel trong cửa sổ)
`build Cancel` với `proofs` đủ `factors_required`, distinct FactorKind (guardian ký / Glint ZK-proof bối cảnh) → validator verify factor khớp anchor-enroll (SSR-4) + `now<veto_deadline` (cận trên) → `Σ→sender==amount+min_ada` → tiền hoàn người gửi.

### 5.3 Khoản lớn (Accept trước Finalize)
Người nhận thấy hòm → `build Accept` (controller `receiver_commit` ký) → set `receiver_consent=true`, mọi field khác byte-perfect bất biến (SSR-14). Hết cửa sổ → Finalize hợp lệ (SS-4 thoả). Không consent tới `reclaim_deadline` → `ReclaimTimeout` hoàn sender (SS-11). Đích ngoài-Phoenix (`receiver_commit==#""`): `large_threshold` không enforce on-chain được — xem giới hạn ở [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md) §7.

### 5.4 Nghi trộm (Freeze → ResolveFreeze)
Guardian `build Freeze` trong cửa sổ-veto → escrow `frozen:=true` → chặn Finalize. Giải-Freeze qua `ResolveFreeze`: guardian-quorum (m-of-n anchor.guardians) xử lý, HOẶC tới `freeze_deadline` không resolve → auto-hoàn sender permissionless (SS-8′, chống grief kẹt vĩnh viễn).

### 5.5 ZK-context bind escrow (Phase 2, VeData/Glint) — SSR-12
Public-input proof Glint (FaceMatch/SecretSelfie/DeviceGeo) PHẢI gồm `blake2b_256(own_ref ‖ escrow_datum_hash)` — proof gắn cứng vào escrow-UTxO đang spend, validator ép public-input khớp escrow đó. Spectra off-chain đảm bảo liveness + anti-replay (nonce theo escrow) — chống dùng lại 1 proof cho nhiều lệnh Cancel.

---

## 6. API backend (tham chiếu — prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`)

| Endpoint | Việc |
|---|---|
| `GET /smartsend/{did}/pending` | liệt kê hòm mở (gửi đi / chờ nhận) + `veto_deadline`/`reclaim_deadline`/`freeze_deadline` |
| `GET /smartsend/escrow/{utxo}` | trạng thái một hòm (Open/consent/frozen) + đích |
| `POST /smartsend/notify-accept` | thông báo người nhận có khoản chờ Accept |

Backend chỉ index + thông báo; KHÔNG giữ khoá, KHÔNG ký thay.

---

## 7. Ranh giới giao việc

| Team | Việc |
|---|---|
| **đội on-chain** | `smartsend_escrow.ak` (SS-1..12, SSR-4) — 7 đường (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout). Dùng lại helper `anchor_controller_ok` + guardian-sig từ `auth_logic`/`taad_logic` (KHÔNG sao chép). **KHÔNG sửa `did_payment.ak` mode-1.** |
| **đội backend** | index hòm mở + cửa sổ-veto/reclaim/freeze + trạng thái consent; thông báo Accept; resolve đích. |
| **Core (rust_core/Flutter)** | builder Open/Cancel/Accept/Finalize/ResolveFreeze/ReclaimTimeout; công tắc Smartsend ở màn Gửi; màn Huỷ + Đồng-ý; hiển thị cửa sổ. Enforce factor Cancel khác gốc seed (I-CURVE-5, dùng chung Rebirthme). |
| **VeData / Glint (ZK)** | factor bối cảnh cho Cancel (FaceMatch/SecretSelfie/DeviceGeo) — Spectra phân tích + Glint ZK-proof bind escrow-ref (§5.5, SSR-12) + verifier Aiken cắm vào `unlock_policy`. |
| **Rebirthme** | cổng chi `did_payment` (nạp nguồn), guardian (factor + Freeze/ResolveFreeze quorum), anti-drain `limit_meter` (nền chống trộm) — module này CHỈ dẫn chiếu, KHÔNG dựng lại. |

---

## 8. Thứ tự deploy + phụ thuộc chặn

**Trình tự onchain:**
1. **`smartsend_escrow.ak`** — dựng hòm + 7 đường (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout) theo bất biến SS-1..12 + SSR-4 đã hợp nhất ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §4.
2. **Verifier Glint** cho factor bối cảnh (VeData) cắm vào `unlock_policy`, public-input bind escrow-ref (§5.5).

**Phụ thuộc ngoài:**
- **Anti-drain `limit_meter.ak`** (Rebirthme) — chống trộm SS-6/T-SS-3 phụ thuộc; khuyến nghị land trước hoặc song song.
- **Guardian factor + Freeze/ResolveFreeze** — dựa guardian-recovery + quorum m-of-n từ anchor.guardians.
- **Enroll-set factor trong anchor** (SSR-4) — schema thuộc Core Anchorme/Validator.
- **Spectra/Glint** (VeData) cho factor bối cảnh ZK.

---

## 9. Test / evidence

**Cần khi land (evidence output thật, đối kháng):**
- **veto-race**: Cancel tại `veto_deadline` reject; Finalize trước `veto_deadline` reject; đúng cận (Cancel=cận trên `tx_hi`, Finalize=cận dưới `tx_lo`) (SS-2/SSR-5).
- **double-satisfaction**: 2-escrow-1-output reject (SS-7′).
- **redirect / output-phụ / minADA lệch**: Finalize đổi địa chỉ nhận reject; output thiếu `min_ada` reject (SS-5′/SS-12/SSR-2).
- **consent forge**: Accept giả chữ ký `receiver_commit` reject; Accept chạm field khác `receiver_consent` reject (SS-9′/SSR-3/SSR-14).
- **deadlock khoản lớn**: người nhận không consent → ReclaimTimeout hoàn sender PASS (SS-11).
- **factor**: Cancel với factor không khớp anchor-enroll reject; factor cùng gốc seed reject; factor trùng-kind/rỗng reject (SSR-4/SS-3/SS-6/SSR-13).
- **Freeze/ResolveFreeze**: Freeze ngoài cửa sổ-veto reject; Freeze treo → Finalize reject; ResolveFreeze trước quorum/`freeze_deadline` reject; sau `freeze_deadline` auto-hoàn sender PASS (SS-8/SS-8′/SSR-6).
- **window floor**: Open với `window < min_window_floor` reject dù "2-bên thoả"; `open_slot` ngoài validity-range tx Open reject (SS-10/SSR-10/SSR-11).
- **fee_covered**: set `fee_covered` bất kỳ không đổi output thực tế (SS-12/SSR-9).

---

## 10. Ghi chú giới hạn thiết kế

- **Chống trộm phụ thuộc anti-drain**: SS-6/T-SS-3 dựa `limit_meter.ak` (Rebirthme) làm nền — Smartsend không phải lá chắn chống trộm đủ một mình khi thiếu lớp đó; xem tiền đề (c) ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §6.
- **Factor bối cảnh ZK** phụ thuộc verifier Glint + Spectra (VeData) — bind escrow-ref theo §5.5 trước khi đưa vào `unlock_policy` production.
- **I-CURVE-5** (factor khác gốc seed) phải enforce ở builder — ràng buộc dùng chung với Rebirthme.
- **Blocker ngoài**: enroll-set factor trong `TAADDatum` (Core Anchorme/Validator), guardian ResolveFreeze quorum (nhánh mới trên guardian-recovery hiện có).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#smartsend)

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai).
Hạ tầng nền (dẫn chiếu): [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md) (ví/guardian/anti-drain), `auth_logic.ak`/`taad_logic.ak`/`did_payment.ak`.
Tài liệu cùng bộ: [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md), [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md), [PhoenixKey-Smartsend-Exec.md](./PhoenixKey-Smartsend-Exec.md).
Code: `PhoenixKey-Validator/validators/smartsend_escrow.ak`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
