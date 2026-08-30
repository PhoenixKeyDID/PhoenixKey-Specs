# PhoenixKey — Smartsend — Kỹ thuật (Tech)

> **Module:** Smartsend (gửi-có-bảo-vệ). **Loại doc:** Kỹ-thuật cho implementer (đội on-chain / đội backend / Core rust_core / VeData-Glint / Spectra-LampNet). **Ngày:** 2026-07-09.
> **Đối tượng đọc:** kỹ-sư triển-khai. HOW: kiến-trúc, datum/redeemer CBOR, điều-kiện tx, luồng e2e, ranh-giới giao-việc, thứ-tự deploy, test.
>
> **Ranh giới (không chồng lấn, không bỏ sót):** module CHỈ đặc-tả hòm ký-quỹ `smartsend_escrow` (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout). **Cổng chi `did_payment`, guardian-recovery, anti-drain `limit_meter`** thuộc module Rebirthme — chỉ dẫn-chiếu. Xem [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) cho bất-biến; [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md) cho ví/guardian/anti-drain; [PhoenixKey-Math.md](./PhoenixKey-Math.md) §10/§11 cho TAAD.

---

## 1. Kiến-trúc + sơ-đồ thành-phần

```
┌──────────────────── PhoenixKey-Core (Flutter + rust_core) ────────────────────┐
│  UI: màn Gửi (công-tắc Smartsend) · Huỷ · Đồng-ý (Accept) · trạng-thái hòm     │
│  rust_core (FFI): smartsend builder (Open/Cancel/Accept/Finalize) · glint proof │
└───────────────┬───────────────────────────────────────────┬───────────────────┘
                │ FFI                                        │ ZK proof
                ▼                                            ▼
┌──────────────────────────────┐              ┌────────────────────────────┐
│ PhoenixKey-Validator (Aiken) │              │ Spectra (LampNet) off-chain│
│  smartsend_escrow.ak           │◄── verify ───┤ liveness / ảnh dựng      │
│  did_payment.ak (nạp nguồn)   │  ref-input   │ Glint (VeData): P1+P6     │
│  taad_logic.ak (guardian)     │◄─────────────┤ bind escrow-ref §5.5 [CHỜ]│
└──────────────┬───────────────┘   anchor     └────────────────────────────┘
               │ resolver / index
               ▼
┌──────────────────────────────┐
│ PhoenixKey-Database (Java)   │   theo-dõi hòm mở, cửa-sổ-veto, trạng-thái consent
└──────────────────────────────┘
```

Nền: hòm Smartsend = **địa-chỉ script Plutus V3 stateful** (UTxO mang `SmartSendDatum`). Nguồn nạp = chi từ Ví Phượng-hoàng (`did_payment`, dẫn-chiếu Rebirthme). Guardian Freeze + factor Cancel dựa anchor TAAD (thuộc Core).

---

## 2. Bất-biến kiến-trúc (load-bearing)

1. **Escrow-UTxO đơn**: mỗi khoản Smartsend = một UTxO tại `smartsend_escrow`; mọi đường thoát ép `count(inputs ∈ Script(smartsend)) == 1` (SS-7′) → không gộp hai hòm thoả-mãn một output.
2. **Cửa-sổ-veto rời-nhau**: Cancel chỉ `now < veto_deadline` (ép cận-trên validity-range), Finalize chỉ `now ≥ veto_deadline` (ép cận-dưới) (SS-2) — đọc từ validity-interval, không tin cờ datum.
3. **Byte-perfect payout**: Cancel/ReclaimTimeout/ResolveFreeze-timeout ép `Σ→sender == amount + min_ada`, Finalize ép `Σ→receiver == amount + min_ada` (SS-1/SS-5′/SS-12) — không đổi hướng, không rút bớt, `fee_covered` KHÔNG vào biểu-thức output.
4. **Factor neo anchor-enroll**: tập factor Cancel + cờ `independent_of_seed` neo trong anchor lúc enroll, KHÔNG tin datum lúc Open (SSR-4) → chống kẻ Open tự đặt factor dễ; mỗi factor phải **distinct FactorKind** + verify thật (SS-3/SSR-13).
5. **Authority nguồn + guardian đọc động qua anchor ref-input** (CIP-31): nạp hòm và Freeze quy về `anchor_controller_ok` / guardian-sig — dẫn-chiếu `auth_logic.ak`, KHÔNG sao-chép.
6. **Freeze có lối thoát bắt-buộc**: Freeze chỉ dùng trong cửa-sổ-veto; giải-Freeze qua guardian-quorum (Σ trọng-số ≥ threshold, I-GUARD-WEIGHT) hoặc `freeze_deadline` auto-hoàn sender — không kẹt vĩnh-viễn (SS-8/SS-8′).
7. **`window` có sàn cứng**: `veto_deadline - open_slot ≥ min_window_floor` bất-kể "2-bên-thoả"; `open_slot` phải nằm trong validity-range của chính tx Open (SS-10).

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
`unlock_policy` chứa `factors_required` + tập factor + `independent_of_seed` (neo anchor-enroll, SSR-4). Hòm gắn asset đã chỉ-định trong `asset`; `min_ada` là lovelace ký-quỹ ghi lúc Open, tách khỏi `amount`. `fee_covered` là số audit — validator KHÔNG đọc field này để tính output (SS-12).

### 3.2 `TAADDatum` (dẫn-chiếu Core Anchorme — field Smartsend đọc)
Smartsend ĐỌC (không sửa): `controller_pkh` (verify nguồn nạp + `receiver_commit` khi Accept), `guardians` (factor Guardian + SS-8/SS-8′ Freeze/ResolveFreeze quorum), enroll-set factor (SSR-4). Thứ-tự field khớp `types.ak` — thuộc Core/Validator, KHÔNG sửa ở đây.

---

## 4. Từng thao-tác — điều-kiện + shape tx + ai-ký

| Thao-tác | Validator | Điều-kiện | Ai ký |
|---|---|---|---|
| **Smartsend Open** | `smartsend_escrow` | tiêu UTxO `did_payment` → escrow; datum đủ (`open_slot`/`veto_deadline`/`reclaim_deadline`/`freeze_deadline`); `open_slot` trong validity-range tx Open, `window ≥ min_window_floor` (SS-10) | sender (controller) |
| **Smartsend Cancel** | `smartsend_escrow` | `now<veto_deadline` (cận-trên); ≥`factors_required` distinct-kind (đa-yếu-tố neo anchor SSR-4, khác gốc seed SS-6); `Σ→sender==amount+min_ada` | sender + factor |
| **Smartsend Accept** | `smartsend_escrow` | verify controller-sig `receiver_commit` qua anchor; chỉ set `receiver_consent=true`, mọi field khác + value bất-biến (SS-9′/SSR-14) | receiver (controller) |
| **Smartsend Finalize** | `smartsend_escrow` | `now≥veto_deadline` (cận-dưới); consent nếu `amount≥large_threshold` (trừ đích ngoài-Phoenix); `Σ→receiver==amount+min_ada` (SS-5′) | (permissionless / Bob) |
| **Smartsend Freeze** | `smartsend_escrow` | `now<veto_deadline`; guardian-sig hợp-lệ (neo anchor) → `frozen:=true`, chặn Finalize (SS-8) | guardian |
| **Smartsend ResolveFreeze** | `smartsend_escrow` | `frozen==true` ∧ (guardian-quorum Σ trọng-số ≥ threshold trên `anchor.guardians` ∨ `now≥freeze_deadline`) → `Σ→sender==amount+min_ada` (SS-8′) | guardian-quorum / permissionless sau `freeze_deadline` |
| **Smartsend ReclaimTimeout** | `smartsend_escrow` | `now≥reclaim_deadline ∧ receiver_consent==false`; `Σ→sender==amount+min_ada` (SS-11) | sender (permissionless sau hạn) |
| **Nạp nguồn (chi ví)** | `did_payment` | anchor Active + controller ký (dẫn-chiếu Rebirthme) | controller |

**Shape Open:** inputs = {vault UTxO(s) `did_payment`} ; outputs = {escrow UTxO(`SmartSendDatum`), [trả-lại-ví]} ; ref-inputs = {anchor} ; signer = controller ; validity `lo` = tip slot → `open_slot`.

**Shape Finalize:** inputs = {escrow UTxO} ; outputs = {→receiver == amount + min_ada, byte-perfect} ; ref-inputs = {anchor nếu cần consent} ; validity `lo ≥ veto_deadline`.

---

## 5. Luồng end-to-end

### 5.1 Gửi-có-bảo-vệ (Open → Finalize) — SS-10
`build Open` → tiêu UTxO `did_payment` (controller ký) → escrow UTxO với `open_slot=tip` (ép nằm trong validity-range tx Open, SSR-11), `veto_deadline=open_slot+window` (`window ≥ min_window_floor`, SSR-10), `reclaim_deadline`, `freeze_deadline` → submit. Hết cửa-sổ, bất-kỳ ai `build Finalize` (đích cố-định →receiver, byte-perfect SS-5′) → tiền về người-nhận.

### 5.2 Đổi ý (Cancel trong cửa-sổ)
`build Cancel` với `proofs` đủ `factors_required`, distinct FactorKind (guardian ký / `ContextZk`: proof Glint bind escrow-ref — CHƯA nối được, §5.5) → validator verify factor khớp anchor-enroll (SSR-4) + `now<veto_deadline` (cận-trên) → `Σ→sender==amount+min_ada` → tiền hoàn người gửi.

### 5.3 Khoản lớn (Accept trước Finalize)
Người-nhận thấy hòm → `build Accept` (controller `receiver_commit` ký) → set `receiver_consent=true`, mọi field khác byte-perfect bất-biến (SSR-14). Hết cửa-sổ → Finalize hợp-lệ (SS-4 thoả). Không consent tới `reclaim_deadline` → `ReclaimTimeout` hoàn sender (SS-11). Đích ngoài-Phoenix (`receiver_commit==#""`): `large_threshold` không enforce on-chain được — xem giới-hạn ở [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md) §7.

### 5.4 Nghi trộm (Freeze → ResolveFreeze)
Guardian `build Freeze` trong cửa-sổ-veto → escrow `frozen:=true` → chặn Finalize. Giải-Freeze qua `ResolveFreeze`: guardian-quorum (Σ trọng-số ≥ threshold trên `anchor.guardians`) xử-lý, HOẶC tới `freeze_deadline` không resolve → auto-hoàn sender permissionless (SS-8′, chống grief kẹt vĩnh-viễn).

### 5.5 ZK-context bind escrow (Phase 2) — SSR-12

**Hai chủ khác nhau, KHÔNG gộp làm một:**

- **Lớp phân-tích (off-chain) — chủ là Spectra (LampNet).** "Người còn sống, không phải phát-lại, không phải ảnh dựng bằng AI". Vai này **đã rời Glint** theo founder lock 2026-07-28 (`Glint-Math.md:21`); chính Glint ghi rằng tham-chiếu "Glint phát hiện media tổng hợp" ở Smartsend phải re-home sang Spectra (`Glint-Math.md:261`).
- **Lớp chứng-minh không tiết-lộ + bind escrow — chủ là Glint (VeData).** Public-input PHẢI gồm `blake2b_256(own_ref ‖ escrow_datum_hash)` — proof gắn-cứng vào escrow-UTxO đang spend, validator ép public-input khớp escrow đó. Primitive đúng tên: **P1** knowledge-of-opening (`Glint-Math.md:101`) + **P6** nullifier chống dùng lại một proof cho nhiều lệnh Cancel (`Glint-Math.md:105`).

**Tên API `FaceMatch` / `SecretSelfie` / `DeviceGeo` KHÔNG tồn tại** — đó là API của vai media-authenticity đã bị gỡ khỏi Glint. Catalog primitive Glint (`Glint-Math.md:95-207`) gồm P1 commit · P2 range · P3 membership/non-membership · P4 quan-hệ tuyến-tính · P6 nullifier, cộng P8 zkML **TREO** và bốn primitive `[CONSTRUCTION-PENDING]` (P-thr / P-venc / P-del / P-vk). **Không primitive nào phát-biểu mệnh-đề về CON NGƯỜI** (đang sống / là chính chủ / khớp khuôn mặt) — nên phần đó phải nằm ở Spectra, không nằm ở Glint.

**Hôm nay CHƯA nối được — trạng-thái đo được, không phải dự-đoán:**

1. `glint-core` (`VeDataIO/Glint/glint-core/`) tự khai **"deterministic (non-ZK)"** (`glint-core/src/lib.rs:3-12`): có content-anchor, `circuit_id` tự-chứng-thực, tách miền `DS_TAG`, phong-bì proof CBOR, khung conformance — **KHÔNG mạch, KHÔNG prover, KHÔNG verifier**. 43 test `cargo test` xanh, nhưng xanh ở phần tất-định.
2. Chưa có sổ tra `circuit_id` (VeData-Registry) để verifier consult; mọi verifier PHẢI **reject** proof mang `circuit_id` treo (`Glint-Math.md:197`).
3. **P6 chế-độ-chặn NGOÀI phạm vi quy-phạm v0.4.2** (`Glint-Math.md:167`), `domain_tag` còn `[PENDING-DECISION]` (`Glint-Math.md:164`) ⇒ phần chống-replay bằng nullifier chưa đặc-tả được.

⇒ `ContextZk` giữ nguyên ở mức interface-contract (`PhoenixKey-Smartsend-Interfaces.md` §1, hàm `glint_verify`), **KHÔNG đưa vào `unlock_policy` production** cho tới khi ba mục trên đóng. Xem hai mục chờ [CẦN CHỐT-SS-SPECTRA] / [CẦN CHỐT-SS-NULLIFIER] ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §9.

---

## 6. API backend (tham-chiếu — prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`)

| Endpoint | Việc |
|---|---|
| `GET /smartsend/{did}/pending` | liệt-kê hòm mở (gửi đi / chờ nhận) + `veto_deadline`/`reclaim_deadline`/`freeze_deadline` |
| `GET /smartsend/escrow/{utxo}` | trạng-thái một hòm (Open/consent/frozen) + đích |
| `POST /smartsend/notify-accept` | thông-báo người-nhận có khoản chờ Accept |

Backend chỉ index + thông-báo; KHÔNG giữ khoá, KHÔNG ký thay.

---

## 7. Ranh-giới giao-việc

| Team | Việc |
|---|---|
| **đội on-chain** | `smartsend_escrow.ak` (SS-1..12, SSR-4) — 7 đường (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout). Dùng lại helper `anchor_controller_ok` + guardian-sig từ `auth_logic`/`taad_logic` (KHÔNG sao-chép). **KHÔNG sửa `did_payment.ak` mode-1.** |
| **đội backend** | index hòm mở + cửa-sổ-veto/reclaim/freeze + trạng-thái consent; thông-báo Accept; resolve đích. |
| **Core (rust_core/Flutter)** | builder Open/Cancel/Accept/Finalize/ResolveFreeze/ReclaimTimeout; công-tắc Smartsend ở màn Gửi; màn Huỷ + Đồng-ý; hiển-thị cửa-sổ. Enforce factor Cancel khác gốc seed (I-CURVE-5, dùng chung Rebirthme). |
| **Spectra (LampNet)** | lớp phân-tích off-chain cho factor bối-cảnh: liveness / chống phát-lại / chống ảnh dựng bằng AI. Vai này rời Glint từ founder lock 2026-07-28 (`Glint-Math.md:21`, `:261`). **Chưa có đặc-tả Spectra nào đo được ở phía PhoenixKey** — mục chờ [CẦN CHỐT-SS-SPECTRA] ([PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §9). |
| **VeData / Glint (ZK)** | lớp chứng-minh không tiết-lộ: **P1** knowledge-of-opening + **P6** nullifier (`Glint-Math.md:101`, `:105`), public-input bind escrow-ref (§5.5, SSR-12), verifier Aiken cắm vào `unlock_policy`. **Chưa cắm được**: `glint-core` tự khai "deterministic (non-ZK)" (`glint-core/src/lib.rs:3-12`), sổ tra `circuit_id` chưa tồn tại (`Glint-Math.md:197`). |
| **Rebirthme** | cổng chi `did_payment` (nạp nguồn), guardian (factor + Freeze/ResolveFreeze quorum), anti-drain `limit_meter` (nền chống-trộm) — module này CHỈ dẫn-chiếu, KHÔNG dựng lại. |

---

## 8. Thứ-tự deploy + phụ-thuộc-chặn

**Trình-tự onchain:**
1. **`smartsend_escrow.ak`** — dựng hòm + 7 đường (Open/Cancel/Accept/Finalize/Freeze/ResolveFreeze/ReclaimTimeout) theo bất-biến SS-1..12 + SSR-4 đã hợp-nhất ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §4.
2. **Verifier Glint** cho factor bối-cảnh (VeData) cắm vào `unlock_policy`, public-input bind escrow-ref (§5.5) — **CHẶN**: chưa có mạch/prover/verifier trong `glint-core` (`glint-core/src/lib.rs:3-12`), chưa có sổ tra `circuit_id` (`Glint-Math.md:197`). Bước này KHÔNG lên lịch được cho tới khi hai thứ đó có.

**Phụ-thuộc ngoài:**
- **Anti-drain `limit_meter.ak`** (Rebirthme) — chống-trộm SS-6/T-SS-3 phụ-thuộc; khuyến-nghị land trước hoặc song-song.
- **Guardian factor + Freeze/ResolveFreeze** — dựa guardian-recovery + quorum theo TỔNG trọng-số (I-GUARD-WEIGHT) từ `anchor.guardians`.
- **Enroll-set factor trong anchor** (SSR-4) — schema thuộc Core Anchorme/Validator.
- **Spectra** (LampNet) cho lớp phân-tích liveness/ảnh dựng — **chưa có chủ đặc-tả ở phía PhoenixKey** ([CẦN CHỐT-SS-SPECTRA]).
- **Glint** (VeData) cho lớp proof P1+P6 bind escrow-ref — chờ mạch/verifier thật + sổ tra `circuit_id` ([CẦN CHỐT-SS-NULLIFIER] cho phần nullifier).

---

## 9. Test / evidence

**Cần khi land (evidence output thật, đối-kháng):**
- **veto-race**: Cancel tại `veto_deadline` reject; Finalize trước `veto_deadline` reject; đúng cận (Cancel=cận-trên `tx_hi`, Finalize=cận-dưới `tx_lo`) (SS-2/SSR-5).
- **double-satisfaction**: 2-escrow-1-output reject (SS-7′).
- **redirect / output-phụ / minADA lệch**: Finalize đổi địa chỉ nhận reject; output thiếu `min_ada` reject (SS-5′/SS-12/SSR-2).
- **consent forge**: Accept giả chữ-ký `receiver_commit` reject; Accept chạm field khác `receiver_consent` reject (SS-9′/SSR-3/SSR-14).
- **deadlock khoản-lớn**: người-nhận không consent → ReclaimTimeout hoàn sender PASS (SS-11).
- **factor**: Cancel với factor không khớp anchor-enroll reject; factor cùng gốc seed reject; factor trùng-kind/rỗng reject (SSR-4/SS-3/SS-6/SSR-13).
- **Freeze/ResolveFreeze**: Freeze ngoài cửa-sổ-veto reject; Freeze treo → Finalize reject; ResolveFreeze trước quorum/`freeze_deadline` reject; sau `freeze_deadline` auto-hoàn sender PASS (SS-8/SS-8′/SSR-6).
- **window floor**: Open với `window < min_window_floor` reject dù "2-bên-thoả"; `open_slot` ngoài validity-range tx Open reject (SS-10/SSR-10/SSR-11).
- **fee_covered**: set `fee_covered` bất-kỳ không đổi output thực-tế (SS-12/SSR-9).

---

## 10. Ghi-chú giới-hạn thiết-kế

- **Chống-trộm phụ-thuộc anti-drain**: SS-6/T-SS-3 dựa `limit_meter.ak` (Rebirthme) làm nền — Smartsend không phải lá-chắn chống-trộm đủ một mình khi thiếu lớp đó; xem tiền-đề (c) ở [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md) §6.
- **Factor bối-cảnh ZK chưa dùng được**, và lý do nằm ở HAI nhà khác nhau: lớp phân-tích (liveness / ảnh dựng) nay thuộc **Spectra** (LampNet) mà PhoenixKey chưa đo được đặc-tả nào; lớp proof (**P1**+**P6**, bind escrow-ref §5.5) thuộc **Glint** (VeData) mà `glint-core` còn tất-định, chưa có mạch/prover/verifier (`glint-core/src/lib.rs:3-12`) và chưa có sổ tra `circuit_id` (`Glint-Math.md:197`). Không đưa `ContextZk` vào `unlock_policy` production trước khi cả hai đóng.
- **I-CURVE-5** (factor khác gốc seed) phải enforce ở builder — ràng-buộc dùng chung với Rebirthme.
- **Blocker ngoài**: enroll-set factor trong `TAADDatum` (Core Anchorme/Validator), guardian ResolveFreeze quorum (nhánh mới trên guardian-recovery hiện có).

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#smartsend)

---

## Nguồn

Nguồn thiết-kế nội-bộ (không công khai).
Hạ-tầng nền (dẫn-chiếu): [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md) (ví/guardian/anti-drain), `auth_logic.ak`/`taad_logic.ak`/`did_payment.ak`.
Tài-liệu cùng bộ: [PhoenixKey-Smartsend-Math.md](./PhoenixKey-Smartsend-Math.md), [PhoenixKey-Smartsend-Vi-Feat.md](./PhoenixKey-Smartsend-Vi-Feat.md), [PhoenixKey-Smartsend-Exec.md](./PhoenixKey-Smartsend-Exec.md).
Code: `PhoenixKey-Validator/validators/smartsend_escrow.ak`.

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
