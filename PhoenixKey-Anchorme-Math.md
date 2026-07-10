# PhoenixKey — Anchorme · Đặc-tả TOÁN hình-thức cho AUDITOR

> **Module:** Anchorme (danh-tính lõi). **Đối tượng:** auditor smart-contract + nhà kiểm-toán danh-tính. Đây là đặc-tả TOÁN (định-nghĩa, bất-biến, mệnh-đề-ép, định-lý an-toàn), KHÔNG phải doc thiết-kế. Ngữ-cảnh sản-phẩm ở `PhoenixKey-Anchorme-Vi-Feat.md`; kỹ-thuật ở `-Tech.md`.
> **Loại doc:** Math. **Ngày:** 2026-07-09.
>
> **Nguồn chân-lý = CODE, không phải văn:** mọi bất-biến neo trực-tiếp về `file:hàm:dòng` trong validator. Khi văn ≠ code → **code thắng**. Auditor thấy chênh → báo lỗi CODE hoặc lỗi SPEC, không tự hoà.
>
> **Code neo (nguồn chân-lý), repo `PhoenixKey-Validator`, Aiken 1.1.x / Plutus V3:**
> - `validators/taad.ak` (63 dòng) — validator đa-mục-đích: `mint` (genesis) + `spend` (lifecycle).
> - `lib/phoenixkey/state_nft_logic.ak` (683 dòng) — mint-gate: GenesisPerson / GenesisChild / GenesisBurn + `can_own`.
> - `lib/phoenixkey/taad_logic.ak` (802 dòng) — spend state-machine: Rotate / Init/Cancel/FinalizeRecovery / Deactivate / UpdateGuardians / Transfer.
> - `lib/phoenixkey/auth_logic.ak` (87 dòng) — `anchor_controller_ok` (đường ví "đi-theo-DID" đọc anchor).
> - `lib/phoenixkey/types.ak` (144 dòng) — `TAADDatum` (10 field), `TAADStatus`, `EntityType` (10 loại), `TAADRedeemer`, `StateNftRedeemer`.
>
> **Bao-phủ test bắt-buộc:** module danh-tính (`taad_logic`, `state_nft_logic`, `attack_tests`; `auth_logic`/`types` không cần test riêng) phải phủ GenesisPerson/GenesisChild/can_own/Rotate/Cancel/Finalize/Deactivate + regression Bug#3 (NFT off-script).
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)
>
> **Aliasing từ `PhoenixKey-Math.md`:** bất-biến A-1/A-2/A-3 (§10.4), CanOwn (§22.1), C-SEQ (§26), Active/Authority (§4) là canonical trong Math.md — ở đây TÁI DÙNG và ánh-xạ sang code, GHI RÕ nguồn §. Mã mới cho tầng code này dùng tiền-tố **`I-CID-n`**.

---

## 1. Ký-hiệu (notation)

| Ký-hiệu | Kiểu | Nghĩa | Neo code |
|---|---|---|---|
| `did` | ByteArray | chuỗi `did:phoenix:<slot>:<hash>` (UTF-8) | `TAADDatum.did`, `types.ak:35` |
| `N(did)` | ByteArray(32) | asset-name của anchor NFT = `blake2b_256(did)` | `taad_logic.ak:55`, `state_nft_logic.ak:143` |
| `π` | PolicyId | own_policy ≡ script-hash của `taad` (Design-2) | `taad.ak:42`, `taad_logic.ak:52` |
| `ctrl` | {0,1}²²⁴ | `controller_pkh` — băm TAAD_Key (Ed25519) hiện-tại | `TAADDatum.controller_pkh`, `types.ak:37` |
| `hw` | ByteArray | `hw_key_pubkey` — HW_Key P-256 (SEC1, carry-by-equality) | `TAADDatum.hw_key_pubkey`, `types.ak:38` |
| `seq` | ℕ | `sequence` — bộ đếm đơn-điệu | `TAADDatum.sequence`, `types.ak:49` |
| `st` | TAADStatus | `{Active, Recovering{...}, Migrated, Revoked}` | `types.ak:9-19` |
| `E(did)` | EntityType | 1 trong 10 loại (Person…Character) | `TAADDatum.entity_type`, `types.ak:21-32` |
| `par` | Option<ByteArray> | `parent_did` — cha (None ⟺ Person) | `TAADDatum.parent_did`, `types.ak:52` |
| `S` | Transaction | tx đang validate | — |
| `sig ∈ S` | — | `list.has(S.extra_signatories, sig)` (đã ký) | — |
| `ref(π,n)` | — | reference-input mang đúng 1 NFT `(π, n)` | `auth_logic.find_anchor_datum:59` |
| primed `x′` | — | giá-trị field ở output tái-tạo (post-state) | `next_datum.*` |

**Hằng (code `types.ak` / `taad_logic.ak`):**
```
MAX_GUARDIANS      = 5           (list.length(new_guardians) <= 5)   taad_logic.ak:195,217
TAADStatus         = {Active, Recovering, Migrated, Revoked}          types.ak:9-19
EntityType (10)    = {Person, Org, Device, Machine, Asset,
                      Bot, AI, Service, Context, Character}            types.ak:21-32
```

> **Chú-ý loại:** trong code `AI` = biến-thể EntityType cho **AgentDID** (Math §19). `Migrated` là trạng-thái đã-định-nghĩa nhưng KHÔNG có redeemer nào chuyển tới nó ở bản này (chỉ Genesis→Active, Rotate/Update giữ Active, Deactivate→Revoked, Recovery↔Recovering).

---

## 2. Định-nghĩa hình-thức

**Đ-1 · anchor name (tất-định):**
```
N(did) = blake2b_256(UTF-8(did))          -- 32 byte
```
Đây CHÍNH là asset-name NFT on-chain (ép ở genesis) và là khoá tra `did_hash` off-chain (`phoenix_address.rs:52` `anchor_name_from_did`; `DID-Registry-Resolver-Feat §1.1`). KHÔNG salt, KHÔNG double-hash. Test-vector byte-exact: `blake2b_256("did:phoenix:person:alice") = 1643c0c9…c470` (`…-Feat §1.3`).

**Đ-2 · anchor sống (live) tại tip:** anchor của `did` là UTxO `u` sao cho `u.address = Script(π)` ∧ `quantity_of(u.value, π, N(did)) = 1`. State-machine ép **≤1 continuing output** mang NFT này mỗi spend (Đ-3), nhưng **không** ép ≤1 toàn-UTXO-set ở mint (đó là gốc lỗ §9, mã I-CID-9).

**Đ-3 · continuing output (spend):** `find_unique_nft_output(S.outputs, π, N(did), Script(π))` = output DUY NHẤT mang `(π, N(did))` VÀ ở `Script(π)`. `None` ⟺ 0 hoặc ≥2 → tx REJECT. Neo `taad_logic.ak:232-249`.

**Đ-4 · fresh (genesis):** `is_fresh(d) ⟺ d.seq = 0 ∧ d.status = Active ∧ d.revoked_slot = None`. Neo `state_nft_logic.ak:294-300`.

**Đ-5 · Active (đọc bởi ví):** `is_active(st) ⟺ st = Active`. Recovering/Migrated/Revoked ⟹ False. Neo `auth_logic.ak:82-87` (bản ví); `taad_logic.status_is_active:322`. Ánh-xạ `Math §4.1 Active()` (nhưng bản code KHÔNG có suspension field).

---

## 3. Bất-biến sổ-sách LÕI (structural conservation)

Module Anchorme KHÔNG giữ tiền (không conservation tài-chính); bất-biến LÕI là **tính-toàn-vẹn cấu-trúc danh-tính**:

```
(SINGLETON-SPEND)   ∀ spend hợp-lệ: input mang đúng 1 NFT (π,N(did))
                    ∧ đúng 1 continuing output mang (π,N(did)) tại Script(π).
(SEQ-MONO)          ∀ transition hợp-lệ: seq′ = seq + 1  (strict +1).
(NFT-BOUND)         NFT KHÔNG rời Script(π) qua bất kỳ spend nào (trừ GenesisBurn).
(IMMUT-CORE)        did, entity_type BẤT-BIẾN qua MỌI spend (nhận-diện không đổi).
```

Ép ở MỌI redeemer spend qua bộ-ba:
- `expect quantity_of(own_input…, π, N(did)) == 1` (`taad_logic.ak:56`),
- `expect Some(next_output) = find_unique_nft_output(…)` (`:57-58`),
- `next_datum.sequence == datum.sequence + 1` (mỗi nhánh),
- `immutable_fields_preserved(next, datum)` (`:310-320`, giữ did/entity_type/parent_did/revoked_slot) — Deactivate thay bằng kiểm từng-field (`:171-181`).

**Đơn-điệu (ánh-xạ A-2, `Math §10.4`):**
```
(MONO-seq)   seq chỉ TĂNG +1;  KHÔNG đường giữ/giảm  → chống replay (tx seq ≤ on-chain seq bị từ chối).
```

---

## 4. Bảng bất-biến I-CID-1 .. I-CID-11 (cơ-chế-ép + neo dòng)

| ID | Bất-biến (hình-thức) | Cơ-chế-ép on-chain | Neo `file:hàm:dòng` |
|---|---|---|---|
| **I-CID-1** *(=A-1 mint)* | Genesis đúc ĐÚNG 1 NFT `+1` dưới `π`; carrier output DUY NHẤT tại `Script(π)`; `name = N(did)` | `find_single_minted_did` (reject bundled ±qty) + `find_unique_output_with_nft` (ép `Script(π)`) | `state_nft_logic.ak:198-223`, `:228-253` |
| **I-CID-2** *(=A-1/A-3 spend)* | Spend: input 1 NFT + continuing output DUY NHẤT tại `Script(π)` mang NFT | `expect quantity_of == 1` + `find_unique_nft_output` | `taad_logic.ak:55-58`, `:232-249` |
| **I-CID-3** *(=A-2/C-SEQ)* | `seq′ = seq + 1` mọi transition (kể cả Deactivate) | mệnh-đề `next.sequence == datum.sequence + 1` ở 7 nhánh | `taad_logic.ak:66,96,122,150,168,190,210` |
| **I-CID-4** (genesis Person) | `E = Person ∧ par = None ∧ ctrl ∈ S ∧ fresh ∧ name = N(did)` | 5 mệnh-đề GenesisPerson | `state_nft_logic.ak:138-152` |
| **I-CID-5** (genesis Child) | Owner Active + ký (G-1); `can_own(E_owner,E_child)` (G-2); `par = Some(owner_did)` (G-3); `E_child ≠ Person` (G-4); fresh; name=N(child) | 8 mệnh-đề GenesisChild + owner qua CIP-31 ref-input | `state_nft_logic.ak:154-177` |
| **I-CID-6** (CanOwn) | Quan-hệ sở-hữu theo ma-trận §22.1 (Person/Org đẻ mọi thứ trừ Person; Machine→{Dev,Bot,AI}; AI→{Bot,AI}; Service→{Dev,Asset,Bot,AI,Service}; còn lại leaf) | hàm thuần `can_own` | `state_nft_logic.ak:82-125` |
| **I-CID-7** (Rotate) | `ctrl` cũ ký; `ctrl′=new_ctrl ∧ hw′=new_hw`; `st′=Active`; guardians giữ; core immut; `seq+1` | 7 mệnh-đề, chỉ khi `(Rotate,Active)` | `taad_logic.ak:62-73` |
| **I-CID-8** (Deactivate) | `ctrl` ký; `st′=Revoked ∧ revoked_slot′=Some(lb)` (lb = finite lower-bound); did/entity/parent/ctrl/hw/guardians giữ; `seq+1` | 9 mệnh-đề, chỉ `(Deactivate,Active)` | `taad_logic.ak:163-183` |
| **I-CID-9** 🔴 (Transfer 2-of-2) | `E = Service`; ctrl cũ **và** ctrl mới cùng ký; `new_ctrl ≠ ctrl`; `ctrl′=new_ctrl ∧ hw′=new_hw`; guardians mới (≤5) atomic; value ≥ vào; `seq+1` | 11 mệnh-đề, chỉ `(Transfer,Active)` | `taad_logic.ak:202-219` |
| **I-CID-10** (auth ví) | Ví "đi-theo-DID" chi ⟺ ĐÚNG 1 anchor ref `(π,N(did))` + `st = Active` + `ctrl ∈ S` | `anchor_controller_ok` (single-carrier + active + ký) | `auth_logic.ak:37-52`, `:59-70` |
| **I-CID-11** 🔴 (HW carry-only) | Validator KHÔNG decode/verify curve của `hw`; chỉ mang bằng `==`. HW_Key P-256 **không** là gốc-tin-cậy on-chain | không có nhánh nào verify `hw`; chỉ `==` ở Rotate/Finalize | `types.ak:46-48` (comment), dùng ở `taad_logic.ak:69,153` |

**Ghi mã 🔴:** I-CID-9 (Transfer) an-toàn NHƯNG là bề-mặt hẹp cần soi (chỉ Service); I-CID-11 là **giả-định load-bearing của lỗ gốc** (§9). Xem §5.6, §9.

---

## 5. Mệnh-đề-ép từng redeemer (đối-chiếu code — trích guard load-bearing)

### 5.1 GenesisPerson — `validate_mint(GenesisPerson,…)` (I-CID-1, I-CID-4)
```
find_single_minted_did(S, π) = Some((name, child))
∧ name == blake2b_256(child.did)              -- A-1 binding
∧ entity_eq(child.entity_type, Person)         -- gốc-tin-cậy
∧ child.parent_did == None                     -- Person không cha (§12,§22.0)
∧ list.has(S.extra_signatories, child.controller_pkh)   -- SELF-genesis proof
∧ is_fresh(child)                              -- seq=0 ∧ Active ∧ revoked_slot=None
```
Neo `state_nft_logic.ak:138-152`. **🔴 Chú-ý auditor:** `controller_pkh` do người-đúc TỰ khai; validator chỉ kiểm "khoá đó đã ký" — KHÔNG kiểm did-string là của người thật. Đây là gốc I-CID-9-lỗ (§9). `hw_key_pubkey` KHÔNG được verify (I-CID-11).

### 5.2 GenesisChild — `validate_mint(GenesisChild{owner_did},…)` (I-CID-5)
```
find_single_minted_did(S, π) = Some((name, child))
∧ find_owner_reference(S, π, owner_did) = Some(owner)     -- owner NFT ở Script(π) qua CIP-31
∧ name == blake2b_256(child.did)
∧ owner.did == owner_did                        -- ref đúng owner khai
∧ status_is_active(owner.status)                -- owner phải Active
∧ list.has(S.extra_signatories, owner.controller_pkh)    -- (G-1) owner ký
∧ can_own(owner.entity_type, child.entity_type)          -- (G-2) §22.1
∧ child.parent_did == Some(owner_did)           -- (G-3) parent edge bất-biến
∧ not_person(child.entity_type)                 -- (G-4) con không bao giờ Person
∧ is_fresh(child)
```
Neo `state_nft_logic.ak:154-177`. **An-toàn hơn Person:** owner phải Active + ký (parent-sig) ⟹ Org/Service/Child KHÔNG dính lỗ §9. `find_owner_reference` ép owner NFT ở `Script(π)` (`:257-281`) — chống forge owner-ref.

### 5.3 Rotate — `(Rotate,Active)` (I-CID-7)
```
must_be_signed_by(S, datum.controller_pkh)      -- ctrl HIỆN TẠI ký
∧ next.sequence == datum.sequence + 1
∧ immutable_fields_preserved(next, datum)        -- did/entity/parent/revoked_slot giữ
∧ next.controller_pkh == new_controller_pkh
∧ next.hw_key_pubkey == new_hw_pubkey            -- carry-only (I-CID-11)
∧ status_is_active(next.status)
∧ next.guardians == datum.guardians
```
Neo `taad_logic.ak:62-73`. `did` bất-biến ⟹ xoay chìa KHÔNG đổi danh-tính (Math §5.1 rotate).

### 5.4 Deactivate — `(Deactivate,Active)` (I-CID-8)
```
must_be_signed_by(S, datum.controller_pkh)
∧ next.sequence == datum.sequence + 1
∧ next.did == datum.did ∧ entity_eq(next.entity_type, datum.entity_type)
∧ next.parent_did == datum.parent_did
∧ next.controller_pkh == datum.controller_pkh
∧ next.hw_key_pubkey == datum.hw_key_pubkey
∧ next.guardians == datum.guardians
∧ status_is_revoked(next.status)                -- st′ = Revoked
∧ next.revoked_slot == Some(lb)                 -- lb = strict_finite_lower_bound(S)
```
Neo `taad_logic.ak:163-183`. `revoked_slot` = lower-bound HỮU-HẠN (không tuỳ-ý). Revoked là dead-end: KHÔNG nhánh spend nào nhận `(_,Revoked)` ⟹ đóng-băng chi-tiêu.

### 5.5 Transfer — `(Transfer,Active)` (I-CID-9)
```
entity_is_service(datum.entity_type)            -- CHỈ ServiceDID
∧ must_be_signed_by(S, datum.controller_pkh)     -- ctrl CŨ ký
∧ must_be_signed_by(S, new_controller_pkh)       -- ctrl MỚI ký (2-of-2 atomic)
∧ new_controller_pkh != datum.controller_pkh
∧ next.sequence == datum.sequence + 1
∧ immutable_fields_preserved(next, datum)
∧ next.controller_pkh == new_controller_pkh ∧ next.hw_key_pubkey == new_hw_pubkey
∧ status_is_active(next.status)
∧ next_value_lovelace >= own_value_lovelace      -- không rút ADA khỏi anchor
∧ next.guardians == new_guardians ∧ list.length(new_guardians) <= 5   -- guardian mới atomic
```
Neo `taad_logic.ak:202-219`. Cài guardian mới **cùng tx** ⟹ tránh cửa-sổ zero-recovery (`types.ak:105-119`).

### 5.6 anchor_controller_ok — đường ví "đi-theo-DID" (I-CID-10)
```
find_anchor_datum(S.reference_inputs, π, N(did)) = Some(anchor)   -- ĐÚNG 1 carrier
∧ is_active(anchor.status)
∧ list.has(S.extra_signatories, anchor.controller_pkh)
```
Neo `auth_logic.ak:37-52`. `find_anchor_datum` (`:59-70`): ≥2 ref cùng NFT ⟹ `None` ⟹ REJECT (chống mơ-hồ authority TRONG-tx). **🔴 Điểm yếu (gốc lỗ §9):** single-carrier chỉ chống 2-anchor-TRONG-1-tx, KHÔNG chống 2-anchor-live-TOÀN-CỤC cùng name. Nếu tồn-tại anchor-live-giả thứ hai (đúc qua §5.1), attacker đính CÁI GIẢ làm ref → drain. `auth_logic` cố-ý KHÔNG ép entity_type (`:16-18`) ⟹ attacker chỉ cần khai Person + tự ký.

### 5.7 InitRecovery / CancelRecovery / FinalizeRecovery (tóm — chi-tiết ở Rebirthme)
Ba nhánh recovery (`taad_logic.ak:76-157`) ép: A-5 collateral khóa vào continuing UTxO (`:95`), A-6 cancel chỉ trước deadline `ub < deadline_slot` (`:120`), Finalize chỉ sau deadline `lb > deadline_slot` + `pending_controller_pkh` ký (`:141,149`). **Ranh-giới MECE:** luồng recovery-guardian thuộc module **Rebirthme** — ở đây chỉ ghi 3 nhánh này TÔN-TRỌNG I-CID-2/I-CID-3 (singleton + seq+1). KHÔNG trình đầy-đủ.

---

## 6. Không-gian trạng-thái + đồ-thị chuyển (auditor coverage)

```
   GenesisPerson / GenesisChild
        │  (mint +1 NFT, fresh: seq=0, Active)
        ▼
     ┌─────────┐  Rotate (ctrl cũ ký)         ┌──────────────┐
     │ Active  │◄──────────────────────────────┤              │
     │ seq=k   │  UpdateGuardians              │  Recovering  │
     └────┬────┘  Transfer(Service,2of2)       │  (timelock)  │
          │  ────InitRecovery(guardians)──────►│              │
          │  ◄───CancelRecovery(ub<deadline)───┤              │
          │  ◄───FinalizeRecovery(lb>deadline)─┤ (pending ký) │
          │                                     └──────────────┘
          │ Deactivate (ctrl ký)
          ▼
     ┌─────────┐  GenesisBurn (mọi movement < 0)
     │ Revoked │──────────────────────────────► [NFT burned]
     └─────────┘  (dead-end: KHÔNG spend path nào nhận Revoked)
```
Chuyển hợp-lệ (đầy-đủ, mọi cái ép `seq+1` + singleton):
- `GenesisPerson/Child`: mint → `(Active, seq=0, fresh)`. LAMP-N/A.
- `Rotate`: `Active → Active`, đổi ctrl+hw, giữ did/entity/guardians.
- `UpdateGuardians`: `Active → Active`, đổi guardians (≤5).
- `Transfer`: `Active → Active`, CHỈ Service, đổi ctrl+hw+guardians, 2-of-2.
- `Deactivate`: `Active → Revoked`, set revoked_slot.
- `Init/Cancel/Finalize Recovery`: `Active ↔ Recovering` (chi-tiết Rebirthme).
- `GenesisBurn`: burn NFT (mọi movement âm).

Chuyển BỊ CẤM (auditor xác nhận REJECT — có test âm):
`(_, Revoked)` spend · Transfer non-Service · seq không +1 (replay) · NFT rời Script(π) · đổi did/entity_type · Person có parent · Person đúc name ≠ N(did) · mint 2 NFT/tx · genesis non-fresh · GenesisChild owner-Person (Person con) · owner không-ký (G-1) · CanOwn vi-phạm (G-2) · burn+mint bundled.

---

## 7. Định-lý an-toàn (+ chứng-minh phác-thảo)

### Định-lý 1 (SINGLETON-PER-SPEND / anchor không nhân-bản trong tx).
> **Phát-biểu.** Mọi spend hợp-lệ tiêu đúng 1 anchor UTxO mang `(π,N(did))` và tái-tạo đúng 1 tại `Script(π)`; NFT không rời validator; không tx nào tạo/hủy anchor giữa spend.

**Chứng-minh (phác).** `taad_logic.ak:56` ép input có `quantity_of == 1`. `:57-58` ép `find_unique_nft_output` trả DUY NHẤT output mang NFT ở `own_credential = Script(π)` (`:242`); 0 hoặc ≥2 → `None` → REJECT. Vì `own_credential` bắt-buộc `Script(_)` (`:52`, VerificationKey input reject), NFT KHÔNG thể ra địa-chỉ ví. GenesisBurn là đường duy-nhất hủy (mọi movement âm, `:186-192`). ∎

### Định-lý 2 (SEQ-MONO / chống replay).
> **Phát-biểu.** `seq` tăng đúng +1 mỗi transition; tx với `seq ≤ on-chain seq` bị từ-chối.

**Chứng-minh.** Mỗi nhánh spend ép `next.sequence == datum.sequence + 1` (7 điểm §4 I-CID-3). Genesis lập `seq=0` (Đ-4). Quy-nạp: post-state luôn `seq_prev+1` ⟹ đơn-điệu-strict. Replay dùng datum cũ (seq thấp) ⟹ output phải `seq_old+1` ≤ seq hiện-tại-on-chain ⟹ không khớp UTxO tip ⟹ trượt. Ánh-xạ `Math §10.4 A-2` + C-SEQ. ∎

### Định-lý 3 (IDENTITY-PERSISTENCE / xoay chìa không mất người).
> **Phát-biểu.** `did` và `entity_type` bất-biến qua Rotate/Transfer/Deactivate/UpdateGuardians/Recovery. Đổi khoá (ctrl,hw) KHÔNG đổi danh-tính.

**Chứng-minh.** `immutable_fields_preserved(next,datum)` (`:310-320`) ép `did′=did ∧ entity_type′=entity_type ∧ parent′=parent ∧ revoked_slot′=revoked_slot` ở Rotate/Init/Cancel/Finalize/UpdateGuardians/Transfer. Deactivate kiểm did/entity/parent riêng (`:171-173`), chỉ đổi revoked_slot (được-phép). ⟹ nhận-diện `(did, entity_type)` là bất-biến toàn vòng-đời. Rotate đổi `ctrl,hw` (`:68-69`) nhưng did giữ ⟹ quan-hệ neo-theo-did (ví did_payment, dịch-vụ con) sống-sót. ∎

### Định-lý 4 (CHILD-AUTHORITY / con phải có cha sống + đúng luật).
> **Phát-biểu.** Mọi GenesisChild hợp-lệ có owner Active, owner-controller ký, quan-hệ `can_own(E_owner,E_child)` đúng §22.1, và con không bao giờ là Person.

**Chứng-minh.** §5.2: `status_is_active(owner.status)` (`:166`) + `list.has(…, owner.controller_pkh)` (`:168`) ⟹ owner sống + ký (parent-sig G-1). `can_own(…)` (`:170`) ⟹ I-CID-6/§22.1. `not_person(child.entity_type)` (`:174`) ⟹ G-4. `find_owner_reference` ép owner NFT ở `Script(π)` (`:257-281`) ⟹ không forge owner-ref. ∎ **Hệ-quả:** Org/Service/mọi non-Person AN-TOÀN với lỗ §9 (có parent-sig chặn); chỉ Person (self-genesis, không parent) HỞ.

### Định-lý 5 (DEACTIVATE-FREEZE / danh-tính chết không lạm-dụng được).
> **Phát-biểu.** Sau Deactivate, `status = Revoked`; không spend path nào nhận `(_, Revoked)` ⟹ anchor không chi/rotate/transfer được; ví đọc anchor thấy `¬is_active` ⟹ từ-chối.

**Chứng-minh.** `when (redeemer, datum.status)` (`taad_logic.ak:60`) chỉ khớp `Active`/`Recovering`; `(_, Revoked)` rơi `_ -> False` (`:222`). `anchor_controller_ok` ép `is_active` (`:47`) ⟹ Revoked/Recovering/Migrated → ví REJECT (I-CID-10). ∎

---

## 8. Giả-định tin-cậy (NGOÀI phạm-vi chứng-minh — auditor soi riêng)

| # | Giả-định | Rủi-ro nếu vỡ | Chủ-quản |
|---|---|---|---|
| T-1 | **Secure Enclave giữ khóa gốc** (HW_Key P-256 + TAAD_Key). Sinh-trắc Enclave đủ chống trùng người. | Máy bị bẻ khóa phần cứng → lộ khóa. Ngoài phạm-vi on-chain. | Enclave (Core), tin-cậy |
| T-2 | **HW_Key P-256 KHÔNG verify on-chain (I-CID-11).** Validator chỉ carry `hw` bằng `==`. | Gốc lỗ §9 (I-CID-9-lỗ): không có ràng buộc on-chain nào buộc did-string ↔ HW thật. | maintainer §9 |
| T-3 | **Uniqueness-anchor-Person phải được ép ở mint** (đóng I-CID-9-lỗ) bằng **PA2 UniquenessThread**. Nếu chưa wiring: GenesisPerson đúc name bất-kỳ + ctrl bất-kỳ → Attacker đúc anchor-Person-giả cùng `N(did)` nạn-nhân → chiếm ví/custody. Org/Service AN-TOÀN (Đ-lý 4). | 🔴 CAO tới khi PA2 land. | đội on-chain — PA2 |
| T-4 | **Resolver/indexer off-chain trung-thực** (resolve-by-hash, point-in-time). Point-in-time fail-closed (503). | Indexer sai/tụt-hậu → verify chữ-ký-cũ sai. Fail-closed giảm-thiểu. | đội backend |
| T-5 | **Type-code canonical: MÃ-VĂN lệch, KHÔNG lệch giữa impl.** `DidPhoenixGenerator.java:56-65` (Person=0,Org=1,Device=2,Machine=3,Asset=4,Bot=5,AI=6,Service=7,Context=8,Character=9) LÀ bảng-byte canonical, khớp thứ-tự `EntityType` Aiken (`types.ak:21-32`). Văn Math §2.2 phải bám THEO bảng-byte này (KHÔNG dùng thứ-tự liệt-kê Person,Org,Context,Device,Machine,Asset,Bot,Agent,Service,Character). | Nếu ai encode sai thứ-tự bảng-byte → DID phi-nhân sinh khác. Rà bản Rust/mobile bám đúng bảng-byte trước khi mint author-DID phi-nhân. | maintainer + đội Core/backend §9 |
| T-6 | **DeviceDID hw_cert verify ở BACKEND**, validator chỉ neo `hw_cert_hash` (`DeviceDID §4`). | Backend là gatekeeper cert thật; nếu lỏng, cert giả lọt. Validator không parse chuỗi cert nhà-SX. | đội backend + đội on-chain |

**Kết-luận phạm-vi:** trong mô-hình {T-1,T-4,T-5,T-6} + **GIẢ-ĐỊNH uniqueness-Person được vá (PA2)**, năm định-lý §7 GIỮ cho MỌI loại DID. **KHÔNG có PA2**, Định-lý 1/3/5 vẫn giữ cho Org/Service/Child (Đ-lý 4 che), nhưng **PersonDID hở** (T-2/T-3): một danh-tính Person-giả cùng name có thể tồn-tại toàn-cục và bị đính làm anchor-ref → drain ví neo-theo-did đó. Đây là ranh-giới an-toàn quan-trọng nhất của module.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 9. [CẦN CHỐT] còn treo (auditor ghi nhận — chưa đóng)

| # | Mục | Ảnh-hưởng an-toàn | Chủ |
|---|---|---|---|
| **CID-1** 🔴 | **Lỗ mã-hoá-anchor Person-over-Person (I-CID-9-lỗ / T-3).** `GenesisPerson` đúc được anchor `name = N(did-nạn-nhân)` + `controller = khoá attacker` + self-sign (vì HW P-256 không verify on-chain, I-CID-11). Attacker đính anchor-giả làm ref khi chi ví nạn-nhân → drain custody. **KHÔNG phải lỗ sinh-trắc/sybil** — sinh-trắc Enclave đủ chống trùng người; đây thuần lỗ **mã-hoá anchor**. Org/Service AN-TOÀN (parent-sig G-1, Đ-lý 4). Đóng bằng **PA2** (structural, §CID-3) + **PA5-a** (thu-hẹp bề-mặt, §CID-2). | 🔴 CAO — rút-được ví PersonDID. Gốc: mint-policy Cardano KHÔNG đọc UTXO-set → không biết name đã đúc → uniqueness toàn-cục đòi structural (thread). | đội on-chain + audit |
| **CID-2** | **PA5-a entity-gate:** `anchor_controller_ok` nhận thêm `allowed: List<EntityType>` ép `anchor.entity_type ∈ allowed`. Đóng Person-giả-**non-Person** (attacker phải khớp entity ví), KHÔNG đóng same-entity (Person↔Person). | Trung — thu-hẹp bề-mặt, defense-in-depth, KHÔNG breaking địa-chỉ (chỉ đổi chữ-ký hàm, grep-update mọi caller). | đội on-chain |
| **CID-3** | **PA2 accumulator + shard:** sorted-list-in-datum scale tới ~triệu (K=256, N/shard ≤ ~400 do trần 14M mem, `PA2-Design §7.2`); dân-số → Merkle-root-in-datum. Địa-chỉ ví GIỮ NGUYÊN (§4 PA2). | CAO — quyết cách đóng CID-1 (crypto-cứng vs uniqueness-mềm resolver-first-wins). | maintainer chốt trục chi-phí |
| **CID-4** | **Type-code canonical: BẢNG-BYTE code, không phải thứ-tự liệt-kê văn Math §2.2.** `DidPhoenixGenerator.java:56-65` ≡ `types.ak:21-32` (Device=2…Service=7,Context=8) LÀ canonical. Văn Math §2.2 phải bám bảng-byte này (không dùng thứ-tự Context=2,Service=8). | Trung — chặn mint author-DID phi-nhân (ProofChat v2). Chưa lộ vì Person/Org/Character (0,1,9) trùng. Cần rà bản Rust/mobile bám đúng bảng-byte. | maintainer (Math v4.7) + đội Core/backend rà Rust/mobile |
| **CID-5** | **`Migrated` status** định-nghĩa nhưng KHÔNG có transition tới. Cần chốt ai/khi-nào chuyển Active→Migrated. | Thấp — trạng-thái chết, không ảnh-hưởng an-toàn hiện-tại. | [CẦN CHỐT] |
| **CID-6** | **Full_Authority / ⊆ bug** (`ServiceDID-…-DRAFT §3`): `scope(Person)={Full_Authority}` + set-⊆ thuần ⟹ PersonDID KHÔNG delegate cap cụ-thể được. Sửa = quan-hệ subsumption `⊑`. | Trung — chặn delegation của PersonDID (Math §4.4/§23). Thuần tầng Math, chưa vào code Core-Anchorme. | maintainer (Math v4.7) |

---

## 10. Danh-mục kiểm cho auditor (checklist rút gọn)

1. **(SINGLETON)** mọi spend: input 1 NFT + 1 continuing output tại `Script(π)`? → grep `find_unique_nft_output` + `quantity_of(...) == 1` (`taad_logic.ak:56-58`).
2. **(SEQ+1)** mọi nhánh ép `next.sequence == datum.sequence + 1`? → 7 điểm §4. Test âm: seq không tăng → REJECT (Deactivate test).
3. **(NFT-BOUND)** `own_credential` bắt-buộc `Script(_)` + carrier output ở `Script(π)`? → `:52`, `:242`. Bug#3 regression test (NFT off-script → REJECT).
4. **(GENESIS Person)** name=N(did) + Person + parent=None + self-sig + fresh? → `state_nft_logic.ak:138-152`. Test âm: name≠N(did), có parent, no-sig, non-fresh.
5. **(GENESIS Child)** owner Active+ký (G-1) + can_own (G-2) + parent=Some(owner) (G-3) + không-Person (G-4)? → `:154-177`. Test âm: Org→Person, no-owner-sig, CanOwn-vi-phạm.
6. **(CanOwn)** ma-trận khớp §22.1? → `:82-125`. Test: Person→Person=False, Person→Org=True, Machine→Device=True, leaf→any=False.
7. **(Transfer)** CHỈ Service + 2-of-2 + new≠old + guardian-atomic? → `:202-219`. Test âm: Transfer non-Service.
8. **(Deactivate-freeze)** Revoked là dead-end (`_ -> False`)? ví đọc `is_active`? → `:222`, `auth_logic.ak:47`.
9. **🔴 (LỖ CID-1)** GenesisPerson đúc name bất-kỳ + ctrl attacker PASS? → xác-nhận `redteam_mint.collide_*` PASS = lỗ THẬT (chưa vá). auth_logic single-carrier KHÔNG chống 2-anchor-toàn-cục.
10. **(HW carry-only)** không nhánh nào verify curve `hw`? → grep vắng verify P-256 on-chain (I-CID-11).

---

## Phụ-lục A — Bảng đối-chiếu bất-biến ↔ dòng code

| Bất-biến | Neo |
|---|---|
| I-CID-1 genesis singleton | `state_nft_logic.ak:198-223`, `:228-253` |
| I-CID-2 spend singleton (A-1/A-3) | `taad_logic.ak:55-58`, `:232-249` |
| I-CID-3 seq+1 (A-2) | `taad_logic.ak:66,96,122,150,168,190,210` |
| I-CID-4 GenesisPerson | `state_nft_logic.ak:138-152` |
| I-CID-5 GenesisChild | `state_nft_logic.ak:154-177` |
| I-CID-6 can_own (§22.1) | `state_nft_logic.ak:82-125` |
| I-CID-7 Rotate | `taad_logic.ak:62-73` |
| I-CID-8 Deactivate | `taad_logic.ak:163-183` |
| I-CID-9 Transfer 2-of-2 | `taad_logic.ak:202-219` |
| I-CID-10 anchor_controller_ok (ví) | `auth_logic.ak:37-52`, `:59-70` |
| I-CID-11 HW carry-only | `types.ak:46-48` |
| immutable_fields_preserved | `taad_logic.ak:310-320` |
| Recovery (Rebirthme scope) | `taad_logic.ak:76-157` |

---

## Nguồn

- `PhoenixKey-Validator`: `validators/taad.ak`, `lib/phoenixkey/{taad_logic,state_nft_logic,auth_logic,types}.ak` — module danh-tính (`taad_logic` + `state_nft_logic` + `attack_tests`).
- `PhoenixKey-Specs/PhoenixKey-Math.md` §2 (DID base/type), §3–§5 (capability/authority/lifecycle), §10.1–§10.5 (TAAD ASM + A-1..A-6), §12–§24 (type catalog + ownership/delegation/revocation), §26 (global invariants).
- Nguồn thiết-kế nội-bộ (không công khai).
