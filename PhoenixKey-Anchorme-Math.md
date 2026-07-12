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
| `E(did)` | EntityType | 1 trong 10 loại (Person…Avatar) | `TAADDatum.entity_type`, `types.ak:21-32` |
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
                      Bot, Agent, Service, Context, Avatar}            types.ak:21-32
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
| **I-CID-6** (CanOwn) | Quan-hệ sở-hữu theo ma-trận §22.1 (Person/Org đẻ mọi thứ trừ Person; Machine→{Dev,Bot,Agent}; Agent→{Bot,Agent}; Service→{Dev,Asset,Bot,Agent,Service}; còn lại leaf) | hàm thuần `can_own` | `state_nft_logic.ak:82-125` |
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

**Chuỗi khai thác đầy-đủ CID-1 (4 bước, đã verify bằng test PASS — `redteam_mint.ak::collide_person_over_org_name`):**
```
1. Attacker gọi taad.mint redeemer GenesisPerson, output datum:
   { did = <did-nạn-nhân>, entity_type = Person, controller_pkh = <khoá attacker>,
     parent_did = None, sequence = 0, status = Active }.
   NFT name = N(did-nạn-nhân) = blake2b_256(did-nạn-nhân) — TRÙNG anchor thật.
2. validate_mint nhánh GenesisPerson (§5.1, I-CID-4) chỉ kiểm name-binding, entity=Person,
   parent=None, self-sign, fresh — TẤT CẢ thoả cho datum tự-khai của attacker → mint PASS
   → tồn-tại UTxO thứ 2 tại Script(π) mang (π, N(did)) với controller = attacker.
3. Khi chi ví nạn-nhân (did_payment/did_stake/did_subaddr), attacker đính anchor GIẢ của hắn
   làm reference-input — CHỈ 1 carrier trong tx này → qua được kiểm singleton của
   find_anchor_datum (I-CID-10, vốn chỉ chống ≥2-ref-TRONG-tx, không biết có anchor thật
   khác đang sống ở tx khác) — ký khoá attacker.
4. anchor_controller_ok đọc controller_pkh từ anchor ĐƯỢC-THAM-CHIẾU (= anchor giả), thấy
   Active + attacker ký → True → drain toàn-bộ ví neo-theo-did đó.
```

**Vì sao I-CID-1/I-CID-2 (singleton-per-spend, §3 SINGLETON-SPEND) KHÔNG cứu được (first-principles):**
I-CID-1/I-CID-2 là bất-biến **trong-phạm-vi-1-tx** (đúng 1 mint +1, đúng 1 carrier output mỗi mint; đúng 1 NFT input + 1 continuing output mỗi spend) — KHÔNG phải bất-biến **toàn UTXO-set**. Minting policy Cardano chỉ validate *delta* của `mint` field trong tx hiện-tại; không có API đọc "asset (π,name) đã tồn-tại ở UTxO nào khác trên chain". Token là **fungible class** (asset-name không unique-per-UTxO, quantity cộng-dồn) ⟹ đúc "+1 of N(did)" ở tx A (attacker) và tx B (nạn-nhân, đã có từ trước) đều PASS ĐỘC-LẬP → hai UTxO khác nhau, mỗi cái 1 đơn-vị cùng `(π, N(did))`, ledger CHO PHÉP. Đây là gốc: **tính duy-nhất-của-anchor-theo-did KHÔNG khả-thi bằng bất kỳ kiểm cục-bộ nào tại mint-time** — uniqueness toàn-cục đòi **global mutable state** (registry/thread) mà genesis phải spend+update (= PA2, §CID-3).

**Phương-án đã xét + LOẠI (auditor không cần đề-xuất lại):**

| PA | Cơ-chế | Vì sao LOẠI/không-đủ |
|---|---|---|
| PA1 | GenesisPerson tiêu-thụ 1 `OutputReference` seed one-shot | Chỉ chứng-minh "policy chạy đúng 1 lần cho seed đó", KHÔNG ràng tên=f(seed) (name buộc = `blake2b_256(did)` tất-định để resolver tra-cứu). Attacker dùng seed CỦA CHÍNH HẮN → vẫn thoả one-shot → vẫn đúc name nạn-nhân. Giải-quyết "replay cùng seed", KHÔNG giải-quyết "collision cross-seed". |
| PA3 | Gate `did` prefix kiểu `did:phoenix:person:` on-chain | **BẤT-KHẢ-THI** với format DID thật: canonical = `did:phoenix:{base32(slot)}:{hex(hash)}` (`taad_did.rs:126`), entity_type nằm TRONG preimage hash (`type_byte`), KHÔNG lộ segment `:person:`/`:org:` để parse on-chain. Chuỗi `did:phoenix:org:acme` trong tài-liệu minh-hoạ là HƯ-CẤU, không phải format thật. |
| PA4 | did_payment/stake/subaddr bake thêm ref tới UTxO-anchor-GỐC (địa-chỉ = f(outref-genesis)) | Phá tính tất-định địa-chỉ theo `did` → **breaking địa-chỉ** (mất tính "địa-chỉ tính được từ did không cần index"). Loại vì trái mục-tiêu thiết-kế địa-chỉ. |
| PA5-a | `anchor_controller_ok` nhận thêm `allowed: List<EntityType>`, ép `anchor.entity_type ∈ allowed` | Không LOẠI — ĐÃ CHỌN làm defense-in-depth ngay (= §CID-2). Đóng cross-entity, KHÔNG đóng same-entity. |
| PA2 | Global uniqueness-thread: 1 UTxO thread duy-nhất, genesis phải spend+update accumulator, chứng-minh did∉accumulator trước khi thêm | **CANONICAL dài-hạn, ĐÃ CHỌN** (= §CID-3). Cách DUY-NHẤT khép-kín uniqueness toàn-cục trên eUTXO vì ép mọi genesis chuỗi-hoá qua 1 điểm tranh-chấp. Chi-phí: contention 1 thread (giảm bằng shard K-thread theo prefix hash) + ExUnit chứng-minh non-membership (accumulator/Merkle). |

**Evidence test PA5-a (patch đã thử-nghiệm, ADDITIVE, không đổi validator/param):**
```
$ aiken check -m "anchor_entity_gate.{..}"
PASS  org_wallet_real_anchor_ok                  (baseline: anchor Org THẬT vẫn chi được — không hồi quy)
PASS  cross_entity_person_over_org_BLOCKED       (Person-giả-Org → entity∉[Org] → predicate False → KHÔNG drain)
PASS  cross_entity_leaks_without_gate            (chứng minh KHÔNG có gate thì cùng anchor giả DRAIN được)
PASS  same_entity_person_over_person_STILL_LEAKS (GIỚI HẠN: same-entity dup VẪN lọt → cần PA2)
4 tests | 4 passed | 0 failed
$ aiken check   -- toàn bộ suite, không hồi quy
Summary 177 checks, 0 errors, 10 warnings
```
File mới `lib/phoenixkey/anchor_entity_gate.ak` — diff DUY-NHẤT, không chạm bất kỳ `validators/*.ak` (compile-param) → 4 địa-chỉ `taad`/`did_payment`/`did_stake`/`did_subaddr` GIỮ NGUYÊN bytes.

### 5.7 PA2 UniquenessThread — kiến-trúc structural (chi-tiết cho §CID-3)

Thiết-kế đầy-đủ ở `spec-proposals/PhoenixKey-Anchor-UniquenessThread-PA2-Design.md` (2026-07-04, PoC 16 test PASS). Tóm-tắt structural cho đội build (đủ để wiring, KHÔNG cần đọc lại file gốc trừ khi cần chứng-minh PoC):

**Thành-phần mới (mẫu tái-dùng từ LAMP `supply_state.ak`):**
```
UniquenessThread validator (spend, mới) — giữ K UTxO "shard-thread", mỗi UTxO 1 thread-NFT
  singleton (thread_policy, shard_name(i)), shard_name(i) = "pk-uniq" ‖ i.
  Datum ShardDatum { shard: Int, entries: <accumulator> }.
thread_policy (mint, mới) — one-shot đúc đúng K thread-NFT lúc bootstrap; sau đó KHÔNG đúc thêm.
taad.mint::GenesisPerson (SỬA) — thêm: tx phải SPEND đúng shard-thread shard_of(name) + TÁI TẠO
  (continuing output cùng địa-chỉ + cùng thread-NFT) + register_ok(old_entries,new_entries,name)==True
  (name chưa live ∧ new = old ∪ {name}).
taad.mint::GenesisBurn (SỬA) — tương-tự, spend+tái-tạo shard-thread với burn_ok(old,new,name)
  (name live → tombstone), giải-phóng slot.
```
Ủy-quyền cùng mẫu LAMP: `spend shard-thread ⟺ có mint/burn anchor thuộc shard đó` (Δ anchor NFT ≠ 0). Toàn-bộ luật uniqueness ép Ở MINT (1 nguồn sự-thật); shard-thread chỉ là state-holder bị chuỗi-hoá.

**Vì sao bất-biến này ĐÓNG CID-1 mà KHÔNG cần sửa ví (đã verify dứt-khoát, PA2-Design §4):** PA2 KHÔNG đổi compile-param của `did_payment`/`did_stake`/`did_subaddr`/`taad` spend — chỉ thêm điều-kiện ở `taad.mint`. ⟹ script-hash + địa-chỉ 4 validator BẤT-BIẾN. PA2 ép **tại-mọi-thời-điểm ≤1 anchor-live per name toàn-cục**; do đó kiểm single-carrier hiện-có của `auth_logic.find_anchor_datum` (I-CID-10) — vốn chỉ chống ≥2-ref-TRONG-1-tx — trở nên ĐỦ AN-TOÀN, vì không còn tồn-tại anchor-live-giả thứ hai trên chain để attacker đính. **Điều-kiện áp-dụng bắt-buộc:** PA2 phải phủ MỌI đường tạo anchor (GenesisPerson + GenesisChild); rotate/recovery Mode B không đúc/burn anchor nên không đụng thread — giữ đúng.

**So-sánh 4 accumulator (lý-do ExUnit, PA2-Design §3.1):**

| Accumulator | Datum size | Non-membership proof | ExUnit/genesis | Kết-luận |
|---|---|---|---|---|
| Sorted-list-in-datum | O(N)·32B | neighbour (0-byte witness, datum tự-chứa) | O(N) scan | Rẻ nhất khi N nhỏ; KHÔNG scale dân-số |
| Merkle root-in-datum | O(1)·32B | Merkle path (log₂N × 32B witness) | O(log N) hash (~vài chục hash) | Scale; đắt hashing hơn; datum tí-hon |
| RSA / vector accumulator | O(1) | 1 phần-tử + modexp | Plutus KHÔNG có builtin modexp → phải mô-phỏng = CẤM (ExUnit nổ) | LOẠI |
| Sparse-Merkle (SMT) | O(1)·32B | non-membership = path tới leaf rỗng | O(256) hash (key=32B) hoặc nén | Scale, non-membership native; đắt hơn Merkle thường |

**Vòng-đời entry (sorted-list PoC, `Entry{name, live:Bool}`):**
```
        register_ok                       burn_ok                    register_ok
absent ───────────► LIVE(name) ──────────────────► TOMBSTONE(name) ─────────────► LIVE(name)
        (insert sorted)      (flip live=False, giữ slot)     (flip live=True, giữ slot)
```
Tombstone KHÔNG xoá slot (tránh delete-shift đắt + giữ sorted rẻ). Re-mint-after-burn hợp-lệ (khớp `state_nft_logic.ak:42-47`): burn ⇒ tombstone; genesis lại cùng did ⇒ `register_ok` thấy name không-live ⇒ flip lại LIVE, không đẻ slot trùng.

**Sharding:** `shard_of(name, K) = name[0] mod K` (tất-định, blake2b uniform ⟹ tải cân-bằng). 2 genesis khác shard ⟹ song-song; cùng shard cùng block ⟹ chuỗi-hoá (1 thắng/block). K chốt ở deploy-param, **256 khuyến-nghị**.

**Throughput theo K (trần lý-thuyết, ĐỘC-LẬP accumulator — sorted-list hay SMT đều bị trần này vì mỗi shard tối-đa 1 spend/block, PA2-Design §5.2):** giả-định Cardano ~1 block/20s ⟹ ~4.320 block/ngày.
```
K (shard)    Genesis song-song/block (trần)   Genesis/ngày (trần)   Ghi-chú
1            1                                 ~4.320                🔴 nghẽn — 1 UTxO toàn hệ
16           16                                ~69.000                1 nibble prefix
256          256                               ~1,1 triệu             1 byte prefix (khuyến-nghị)
4.096        4.096                             ~17,7 triệu            1,5 byte
65.536       65.536                            ~283 triệu             2 byte
```
Trần contention-per-block là lý-thuyết; thực-tế thấp hơn do 2 genesis tranh cùng-shard cùng-block chỉ 1 vào block (R2 dưới). K=256 dư sức cho mục-tiêu onboarding "hàng nghìn+/ngày".

**🔴 Evidence ExUnit đo thật (PA2-Design §7.2, load-bearing cho §CID-3) — marginal per-genesis, tách khỏi chi-phí build:**
```
N=100:  ~1.96 M mem  /  ~0.59 B cpu   (bench_verify_over_100 − bench_baseline_build_100)
N=300:  ~10.58 M mem /  ~2.51 B cpu   (bench_verify_over_300 − bench_baseline_build_300)
```
Trần tx mainnet = 14 M mem / 10 B cpu. Mem tăng **siêu-tuyến-tính** theo N (do `is_sorted` tái-tạo cons-cell: 100→300 = 3×N nhưng mem ≈5,4×) ⟹ **N_max/shard ≈ 300–400**. Ràng-buộc chéo K↔N: `K ≥ P/400` (P = tổng dân-số Person trong accumulator). P=1 triệu ⟹ K≥2.500; P=8 tỷ ⟹ K≥20 triệu (20 triệu UTxO-thread-NFT tồn-tại vĩnh-viễn × ~1,5 ADA min-ADA ≈ 30 triệu ADA khoá — KHÔNG khả-thi). Đây là lý-do cứng đẩy Person quy-mô dân-số sang Merkle-root-in-datum (giao-cắt chi-phí: N < ~50/shard sorted-list rẻ hơn; N > ~50 Merkle rẻ hơn + là đường DUY-NHẤT scale).

**Interface contract (PA2-Design §10, cho đội build):**
```
Anchor NFT: name = blake2b_256(did) — BẤT-BIẾN, did_payment/stake/subaddr tiếp-tục neo name này.
ShardDatum { shard: Int, entries } — sorted-list PoC, hoặc { shard, root: ByteArray(32) } (Merkle).
Redeemer thread: implicit (spend ⟺ có genesis/burn anchor thuộc shard — mẫu supply_state).
shard_of(name, K) = name[0] mod K. K = 256 khuyến-nghị.
Địa-chỉ ví: KHÔNG đổi. Ví KHÔNG reference thread.
```

**Rủi-ro tồn-dư cần ép khi wiring (PA2-Design §9):** R2 contention cùng-shard cùng-block: burst đăng-ký trùng prefix → tranh 1 UTxO → chỉ 1 tx vào block, còn lại tx-conflict phải rebuild+resubmit; giảm bằng tăng K (bảng trên) hoặc batch-rollup (gom nhiều genesis vào 1 tx spend 1 shard — nhưng làm datum/redeemer per-tx phình bội, chỉ hợp SMT không hợp sorted-list). R3 bootstrap phải one-shot đúng K/đúng shard-index (sai → shard mồ-côi hoặc trùng, vỡ uniqueness); R4 phải ép trần N/shard on-chain (từ-chối register khi `len ≥ N_max`, nếu không 1 genesis có thể ăn hết ExUnit-budget = DoS đăng-ký); R5 GenesisChild hiện an-toàn nhờ parent-sig (Đ-lý 4) nhưng để bất-biến "≤1 live/name" đúng TUYỆT-ĐỐI cũng nên spend shard tương-ứng (Person là ưu-tiên; Child là hoàn-thiện).

### 5.7bis PA2 — CHỐT hướng SMT/Merkle-root-in-datum (khép §CID-3, tầng Person)

`PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md` (2026-07-09) chốt hướng thay accumulator cho tầng Person, thay sorted-list PoC ở §5.7 (giữ nguyên §5.7 làm PoC/baseline đo ExUnit — phần này là quyết-định NÂNG-CẤP, không xoá):

```
ShardDatum { shard: Int, root: ByteArray(32) }        -- root = SMT-root, O(1) bất kể N
```

- **Lý-do chốt:** sorted-list-in-datum KHÔNG scale dân-số (§5.7 evidence: mem siêu-tuyến-tính, N/shard ≤ ~300-400 → K≥20 triệu ở P=8 tỷ ≈ 30 triệu ADA khoá vĩnh-viễn, KHÔNG khả-thi). SMT-root đổi datum O(N)·32B → O(1)·32B; non-membership proof `~log₂N` hash cố-định theo depth, KHÔNG theo N (N=1 triệu → ~20 hash/tx).
- **register (GenesisPerson):** witness = non-membership proof (SMT path tới leaf rỗng của `key = blake2b_256(did)`) đưa qua **redeemer**; validator verify path + tính root mới, KHÔNG lưu cây đầy-đủ on-chain.
- **K sau chốt chỉ còn để giải contention** (2 genesis cùng shard cùng block → chuỗi-hoá), KHÔNG còn để giới-hạn N. K=256 khuyến-nghị giữ nguyên (§5.7).
- **SMT thắng Merkle-neighbour (sorted-tree)** vì non-membership native (không cần "hàng-xóm sắp-xếp" trong witness) — thuật-toán tái-dùng từ VeData `Stamp` (SMT non-membership, IACR 2016/683), PhoenixKey chỉ **instantiate lại bằng `blake2b_256`** (không tái-dùng cây/instance của Stamp).

**Quy-ước băm PA2 (BẮT BUỘC, domain-separation RFC 6962 — khác Mirage/Strata dùng BLAKE3, khác Mosaic/Stamp dùng SHA3 cho cây CỦA HỌ):**
```
H_leaf(d)   = blake2b_256(0x00 ‖ d)
H_node(l,r) = blake2b_256(0x01 ‖ l ‖ r)
```
Lý-do: on-chain (Plutus) chỉ có `blake2b_256` là builtin rẻ ExUnit và đã là hàm của `N(did)` (Đ-1 §2) — indexer PA2 PHẢI dựng cây bằng đúng hàm/domain-sep này để root khớp validator; lệch (vd lỡ dùng BLAKE3 theo thói-quen Mirage) ⟹ root không khớp ⟹ mọi genesis FAIL.

**Đổi-lại (chi-phí duy-nhất):** off-chain (indexer) phải giữ cây đầy-đủ + sinh witness mỗi genesis; giảm rủi-ro bằng (a) ≥2 indexer độc-lập (không single point), (b) cây **tái-dựng-được-từ-chain** (mọi register/burn phát event đủ để dựng lại cây từ genesis — bắt-buộc, nếu không indexer thành điểm tin-cậy đơn-lẻ mới).

**Không đổi gì khác** (giữ nguyên toàn-bộ khẳng-định §5.7 "vì sao đóng CID-1 mà không sửa ví"): compile-param `did_payment/stake/subaddr/taad` không đổi, `TAADDatum`/`auth_logic` không đổi field, chỉ 1 field `ShardDatum.entries → ShardDatum.root`; validator PA2 CHƯA deploy production ⟹ không state phải migrate.

**Cập-nhật bảng §9 CID-3:** hướng "maintainer chốt trục chi-phí" nay đã có đề-xuất cụ-thể = SMT/Merkle-root-in-datum (trên); vẫn còn 2 điểm chờ maintainer xác-nhận cuối: chấp-nhận vận-hành ≥2 indexer + cây-tái-dựng-từ-chain (đánh-đổi datum-O(1) lấy off-chain-witness), và SMT vs Merkle-neighbour (đề-xuất SMT). Nguồn: `spec-proposals/PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md` Phần III, [CHỐT-P1]/[CHỐT-P2]/[CHỐT-H1].

### 5.10 Custody khoá FROST-Ed25519 — trực-giao với `controller_pkh` (thiết-kế, chờ chốt)

Thiết-kế mới (`PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md` Phần I, 2026-07-09, **THIẾT-KẾ — chờ maintainer chốt, chưa code**) mở khả-năng `ctrl` (`controller_pkh`, §1) là hash của **group public key** thay vì khoá đơn, KHÔNG đổi bất-kỳ bất-biến I-CID-* nào ở tài-liệu này:

```
controller_pkh := blake2b_224(PK_group)     -- FROST t-of-n, cùng field/độ-dài với blake2b_224(vk) đơn hiện-tại
```

- **FROST (Flexible Round-Optimized Schnorr Threshold, Ed25519)** — nguồn hàn-lâm: Komlo, C. & Goldberg, I. (2020). *"FROST: Flexible Round-Optimized Schnorr Threshold Signatures"*, SAC 2020 — https://eprint.iacr.org/2020/852 `[FROST20]` (thêm 2026-07-12): nhóm t-of-n participant cùng ký ra **MỘT chữ-ký Ed25519 chuẩn** trên `PK_group` — ledger Cardano verify y-hệt chữ-ký đơn-khoá (`extra_signatories`). Toàn-bộ phân-tán khoá xảy-ra **off-chain** (DKG + sign qua transport `rust_core`/LampNet); validator/on-chain **hoàn-toàn không đổi** — `auth_logic.ak:37-52` (I-CID-10), `taad_logic.ak:62-73` (I-CID-7 Rotate) đọc/so `ctrl ∈ S.extra_signatories` giống hệt hiện-tại, không phân-biệt được `ctrl` sinh từ khoá đơn hay từ `PK_group`.
- **Không đổi:** compile-param, script-hash, địa-chỉ ví (vẫn neo `blake2b_256(did)` qua Đ-1), `TAADDatum` (không thêm field). FROST chỉ thay-đổi **cách `ctrl` được cấu-tạo off-chain**, không thay-đổi ngữ-nghĩa on-chain của field này.
- **Trực-giao với Rotate/Recovery (I-CID-7, §5.3, §5.8):** đổi/tái-cấu-hình FROST-group (re-share/DKG mới → `PK_group'` mới) đi qua đúng redeemer `Rotate` hiện-có (`new_controller_pkh = blake2b_224(PK_group')`) — KHÔNG cần redeemer mới, KHÔNG đụng I-CID-3 (seq+1) hay Đ-lý 3 (IDENTITY-PERSISTENCE, `did` bất-biến qua Rotate).
- **Trực-giao với guardian-recovery** (Rebirthme I-WALLET-8, ĐÃ BỎ Shamir): FROST là trục **cấu-tạo khoá** (khoá đầy-đủ chưa từng tồn-tại — DKG loại "điểm nguyên-vẹn lúc chia" + "điểm nguyên-vẹn lúc dựng-lại" mà Shamir còn giữ); guardian là trục **quyền** (ai được uỷ Rotate khi mất khoá). Hai trục xếp chồng, không thay nhau; guardian **không** giữ share FROST.
- **Không đóng lỗ CID-1/T-3 (§9, §8):** FROST đổi *cách cấu-tạo* `controller_pkh`, KHÔNG kiểm did-string có phải của người thật hay không ở genesis — lỗ Person-over-Person (I-CID-9-lỗ, self-genesis + HW P-256 carry-only I-CID-11) vẫn cần đóng riêng bằng PA2 (§5.7/§5.7bis) + PA5-a (§CID-2). FROST và PA2/CID-1 là hai việc **độc-lập, không thay nhau**.
- **Trạng-thái:** THIẾT-KẾ, hai preset đề-xuất — (1) LampNet-default (`{device}` bắt-buộc `∪ {LampNet node}` t-of-n, đề-xuất 2-of-3) và (2) self-custody (`{device}` 1-of-1 + session-key uỷ-quyền revocable, KHÔNG phải controller — ship trước làm van an-toàn liveness). **CHƯA code, CHƯA đổi `rust_core`** (hiện chỉ có `ed25519-dalek` đơn-khoá, `sign.rs:63-65`). Danh-sách [CHỐT-F1..F3]/[CHỐT-K1] và rủi-ro R-FROST-1..3 → xem nguồn.

Nguồn đầy-đủ (preset, ngưỡng t/n, liveness trade-off, transport DKG, thư-viện `frost-ed25519`): `spec-proposals/PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md` Phần I–II.

### 5.8 InitRecovery / CancelRecovery / FinalizeRecovery (tóm — chi-tiết ở Rebirthme)
Ba nhánh recovery (`taad_logic.ak:76-157`) ép: A-5 collateral khóa vào continuing UTxO (`:95`), A-6 cancel chỉ trước deadline `ub < deadline_slot` (`:120`), Finalize chỉ sau deadline `lb > deadline_slot` + `pending_controller_pkh` ký (`:141,149`). **Ranh-giới MECE:** luồng recovery-guardian thuộc module **Rebirthme** — ở đây chỉ ghi 3 nhánh này TÔN-TRỌNG I-CID-2/I-CID-3 (singleton + seq+1). KHÔNG trình đầy-đủ.

### 5.9 DeviceDID — `Op_create_device` (tóm — đặc tả đầy đủ ở `spec-proposals/PhoenixKey-DeviceDID-Op-Feat-Math.md`, **on-chain blocker B3**, chưa build)

Neo `EntityType.Device` (§1, `types.ak:21-32`) — mint riêng, KHÔNG dùng `state_nft_logic.GenesisChild` (khác cây bất-biến vì mang thêm `hw_cert`/TTL). File dự kiến: `validators/device_did.ak` + mint policy `device_did_mint` (**chưa tồn tại trong repo hiện tại** — khác 4 file neo ở đầu tài-liệu này).

**Datum `DeviceDIDDatum`** (nhân bản `TAADDatum` §2.3 Math + phần mở-rộng riêng):
```
did, schema_version=1, type=Device(0x03), owner (PersonDID|OrgDID, Active tại create),
security_level=Hardware_Attestation, created_slot, seq=0, metadata_hash,
device_pubkey (Ed25519), device_class (Mobile|Laptop|Server|VPS|Sensor|Drone|Wearable|Gateway|LampNetNode),
hw_cert_hash (BLAKE2b-256), hw_cert_kind (TPM2_Quote|SEP_Cert|FIDO2_Assertion),
cert_issued_slot, device_id_hash (= anchor NFT token_name), provisioner (DID cấp phát)
```

**Redeemer**: `Create{provisioner_sig, owner_pkh, nonce}` — `provisioner_sig` ký `CHALLENGE_STR = "PHOENIXKEY_DEVICE_CREATE:" ++ hex(device_pubkey) ++ ":" ++ owner ++ ":" ++ hex(hw_cert_hash) ++ ":" ++ device_class_tag ++ ":" ++ nonce` (byte-exact, nhân bản mẫu genesis §5.1/§13.1 Math).

**Bất-biến DEV-1..DEV-8** (đặt tên độc-lập với dãy I-CID-* ở tài-liệu này — namespace riêng cho module DeviceDID, KHÔNG trộn):

| ID | Bất-biến | Ép ở |
|---|---|---|
| DEV-1 | `provisioner` ký `CHALLENGE_STR`; `provisioner` Active tại `s` | mint policy |
| DEV-2 | `owner` Active tại `s`, `type(owner) ∈ {Person, Org}` (owner đọc qua reference-input) | validator |
| DEV-3 | `hw_cert_hash ≠ 0`, khớp `hw_cert_kind` (băm ở backend, validator neo) | mint policy |
| DEV-4 | `device_id_hash` duy-nhất: mint đúng 1 NFT `token_name=device_id_hash` dưới `device_did_mint` (Cardano chặn double-mint cùng name — thay first-claim của AssetDID §17, KHÔNG phải PA2 uniqueness-thread) | mint policy |
| DEV-5 | `cert_issued_slot ≤ s < cert_issued_slot + MAX_CERT_AGE_SLOTS[device_class]` | validator (validity range) |
| DEV-6 | `did = DID_construct(Device, provisioner, s)`; `device_pubkey`, `device_id_hash` IMMUTABLE sau create | validator |
| DEV-7 | `device_class = LampNetNode ⇒ hw_cert_kind` Hardware bắt-buộc (không Software) | mint policy |
| DEV-8 | type byte = Device = `0x03`; `schema_version=1`; `seq=0`; `security_level=Hardware_Attestation` | mint policy |

`MAX_CERT_AGE_SLOTS` (TTL theo `device_class`, ép re-attest): Mobile=4320, Server/VPS=2160, Laptop/LampNetNode/Drone=8640, Gateway/Wearable=17280, Sensor=43200.

**Ranh-giới hw_cert (khác I-CID-11 HW carry-only ở §4):** validator KHÔNG parse chuỗi cert nhà-SX (TPM2/SEP/FIDO2 root-CA) — quá đắt ExUnit. Backend verify đủ 4 bước (parse → verify chữ-ký root → kiểm PCR/nonce binding → freshness) rồi băm `hw_cert_hash = BLAKE2b-256(hw_cert_canonical_bytes)`; validator chỉ neo hash + enforce cấu trúc (DEV-3/5/7). Đây là mở-rộng cụ-thể của giả-định **T-6** (§8) — không phải bất-biến on-chain mới, mà xác nhận lại: DeviceDID cũng dựa gatekeeper backend giống hw_cert nói chung.

**Trạng-thái build:** spec DEV-READY, **chưa có code** (`device_did.ak` không tồn tại trong repo tại thời điểm viết); liệt vào on-chain blocker cho `lampnet-join` (node admission), Knowme (Document Claim subject), Rada (device↔owner binding). Chi tiết endpoint offchain, test plan T-ON/T-OFF/T-E2E → xem file nguồn.

### 5.11 ServiceDID self-service — `Op_create_service` (tóm — đặc tả đầy đủ ở `spec-proposals/ServiceDID-SelfService-and-Delegation-DRAFT.md` §1, **PROPOSAL chờ duyệt**, chưa build)

Neo `EntityType.Service` (§1, `types.ak:21-32`) — ĐÃ có type on-chain, nhưng KHÔNG có operation/endpoint nào tạo ra nó (§20 Math định-nghĩa ServiceDID là record tĩnh). Gap: `/auth/token/exchange` + `ResolverController` giả-định ServiceDID ĐÃ tồn-tại on-chain (có `serviceEndpoint[]`), nhưng hiện chỉ cấp thủ-công → không scale.

**Endpoint đề-xuất `POST /identity/service/create`** (mô-hình theo `/identity/register` đã live: verify chữ-ký owner → publish on-chain → trả txHash):
```
Request:  { owner_did, service_name, redirect_uris[], capability_set[], api_hash, signature, nonce }
Challenge ký: "PHOENIXKEY_SERVICE_CREATE:" ++ owner_did ++ ":" ++
              H(service_name ∥ redirect_uris ∥ capability_set ∥ api_hash) ++ ":" ++ nonce
Response: { service_did, tx_hash, client_id }        -- client_id ≜ service_did (1 nguồn sự-thật,
                                                          KHÔNG sinh registry off-chain phụ; khớp
                                                          TokenExchangeController: "aud=ServiceDID,
                                                          không có app registry riêng")
```

**Operation hình-thức `Op_create_service(owner, service_name, redirect_uris, capability_set, api_hash, s) → ServiceDID`:**
```
Pre:  Authority(owner, Write_DID(new_service), s)      -- owner ký: Person=HW_Key, Org=m-of-n threshold
      ∧ capability_set ⊑ scope(owner, s)                -- C-SCOPE-1, quan-hệ ⊑ (§3.3 Math v4.7, xem CID-6)
      ∧ type(owner) ∈ {Person, Org, Service}             -- §20 owner; Service→Service cho phép
      ∧ |{anc ∈ Ancestors | type=Service}| ≤ MAX_SERVICE_DEPTH=3   -- I-SVC-CHAIN (§20 Math)
      ∧ nonce_not_used_on_chain(nonce, owner)
Effect: ServiceDID{type=Service, owner, service_name, version=1,
                    capability_set, api_hash, audit_contract, app_token=None, service_key};
        publish anchor (2 lựa chọn anchoring, xem dưới).
```
Owner ký = "duyệt" — KHÔNG có khâu phê-duyệt người (self-service phi-tập-trung, đúng vai-trò Layer 0 hạ-tầng của PhoenixKey, §35.1).

**2 lựa chọn anchoring on-chain + khuyến-nghị 2 giai-đoạn:**

| | (a) metadata-6789 (custodial-lite, đường PersonDID hiện-tại) | (b) TAAD anchor NFT, `entity_type=Service` |
|---|---|---|
| Trạng-thái | ✅ đã chạy thật (preprod/preview), tái-dùng ngay | ⏳ phụ-thuộc tx-builder feature-branch (chưa merge) |
| Quyền kiểm-soát | chữ-ký off-chain, backend kiểm | on-chain, validator ép — chống backend tự ý đổi |
| Đời-sống ServiceDID | khó làm state-machine sạch (rotate `service_key`, revoke) | có TAAD state-machine đúng §20 `version` monotone |

**KHUYẾN-NGHỊ (đã trong DRAFT):** (b) TAAD anchor NFT là đích canonical dài-hạn (ServiceDID cần rotate `service_key`/revoke/`version` monotone — chỉ on-chain state-machine làm sạch được, validator Design-2 ĐÃ hỗ-trợ mint theo `entity_type` nên chi-phí kỹ-thuật là tx-builder chứ không phải validator mới). Phát-hành **2 giai-đoạn**: GĐ1 dùng (a) để self-service chạy NGAY (tái-dùng đường PersonDID đã live); GĐ2 chuyển canonical sang (b) khi tx-builder merge, (a) hạ xuống vai-trò compatibility-mirror (đúng mẫu §2.5.6 Math: UTxO datum tồn-tại thì nó authoritative). `client_id ≜ service_did` bất-biến qua migration.

**Platform vs App:** KHÔNG type mới — dùng quan-hệ owner-chain sẵn-có của §20 (`Service→Service ≤ 3 hop`, I-SVC-CHAIN). Platform = ServiceDID `owner=OrgDID`, capability_set rộng; App = ServiceDID `owner=ServiceDID-của-Platform`, capability_set hẹp hơn (⊑ capability_set Platform, C-SCOPE-1). Ví-dụ chain `OriLife → SatelliteAPI → WeatherAPI` (§20).

**Trạng-thái build:** PROPOSAL, **chưa duyệt, chưa code**. `Op_create_service` chưa có trong §23 Math; endpoint `/identity/service/create` chỉ mới nhắc tên ở `PhoenixKey-Anchorme-Tech.md` (chưa có operation formal). Việc sửa `⊑` (điều-kiện Pre `capability_set ⊑ scope(owner)`) phụ-thuộc CID-6 (§9) đã fold vào Math v4.7 §3.3. Chi-tiết đầy-đủ (request/response JSON, phụ-lục việc chờ Long/anh Aladin duyệt: phí tạo, `api_hash` optional/bắt-buộc theo giai-đoạn) → xem file nguồn.

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
| T-5 | **Type-code canonical: MÃ-VĂN lệch, KHÔNG lệch giữa impl.** `DidPhoenixGenerator.java:56-65` (Person=0,Org=1,Device=2,Machine=3,Asset=4,Bot=5,Agent=6,Service=7,Context=8,Avatar=9) LÀ bảng-byte canonical, khớp thứ-tự `EntityType` Aiken (`types.ak:21-32`). Văn Math §2.2 phải bám THEO bảng-byte này (KHÔNG dùng thứ-tự liệt-kê Person,Org,Context,Device,Machine,Asset,Bot,Agent,Service,Avatar). | Nếu ai encode sai thứ-tự bảng-byte → DID phi-nhân sinh khác. Rà bản Rust/mobile bám đúng bảng-byte trước khi mint author-DID phi-nhân. | maintainer + đội Core/backend §9 |
| T-6 | **DeviceDID hw_cert verify ở BACKEND**, validator chỉ neo `hw_cert_hash` (chi tiết datum/invariant DEV-1..8 → §5.9). | Backend là gatekeeper cert thật; nếu lỏng, cert giả lọt. Validator không parse chuỗi cert nhà-SX. | đội backend + đội on-chain |

**Kết-luận phạm-vi:** trong mô-hình {T-1,T-4,T-5,T-6} + **GIẢ-ĐỊNH uniqueness-Person được vá (PA2)**, năm định-lý §7 GIỮ cho MỌI loại DID. **KHÔNG có PA2**, Định-lý 1/3/5 vẫn giữ cho Org/Service/Child (Đ-lý 4 che), nhưng **PersonDID hở** (T-2/T-3): một danh-tính Person-giả cùng name có thể tồn-tại toàn-cục và bị đính làm anchor-ref → drain ví neo-theo-did đó. Đây là ranh-giới an-toàn quan-trọng nhất của module.

→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#anchorme)

---

## 9. [CẦN CHỐT] còn treo (auditor ghi nhận — chưa đóng)

| # | Mục | Ảnh-hưởng an-toàn | Chủ |
|---|---|---|---|
| **CID-1** 🔴 | **Lỗ mã-hoá-anchor Person-over-Person (I-CID-9-lỗ / T-3).** `GenesisPerson` đúc được anchor `name = N(did-nạn-nhân)` + `controller = khoá attacker` + self-sign (vì HW P-256 không verify on-chain, I-CID-11). Attacker đính anchor-giả làm ref khi chi ví nạn-nhân → drain custody. **KHÔNG phải lỗ sinh-trắc/sybil** — sinh-trắc Enclave đủ chống trùng người; đây thuần lỗ **mã-hoá anchor**. Org/Service AN-TOÀN (parent-sig G-1, Đ-lý 4). Đóng bằng **PA2** (structural, §CID-3) + **PA5-a** (thu-hẹp bề-mặt, §CID-2). | 🔴 CAO — rút-được ví PersonDID. Gốc: mint-policy Cardano KHÔNG đọc UTXO-set → không biết name đã đúc → uniqueness toàn-cục đòi structural (thread). | đội on-chain + audit |
| **CID-2** | **PA5-a entity-gate:** `anchor_controller_ok` nhận thêm `allowed: List<EntityType>` ép `anchor.entity_type ∈ allowed`. Đóng Person-giả-**non-Person** (attacker phải khớp entity ví), KHÔNG đóng same-entity (Person↔Person). | Trung — thu-hẹp bề-mặt, defense-in-depth, KHÔNG breaking địa-chỉ (chỉ đổi chữ-ký hàm, grep-update mọi caller). | đội on-chain |
| **CID-3** | **PA2 accumulator + shard:** sorted-list-in-datum scale tới ~triệu (K=256, N/shard ≤ ~400 do trần 14M mem, `PA2-Design §7.2`); dân-số → Merkle-root-in-datum. Địa-chỉ ví GIỮ NGUYÊN (§4 PA2). **Cập-nhật:** hướng SMT/Merkle-root-in-datum đã có đề-xuất cụ-thể (§5.7bis, `ShardDatum{shard,root}`, `blake2b_256` domain-sep RFC 6962) — còn chờ maintainer xác-nhận cuối (≥2 indexer + cây-tái-dựng-từ-chain, SMT vs Merkle-neighbour). | CAO — quyết cách đóng CID-1 (crypto-cứng vs uniqueness-mềm resolver-first-wins). | maintainer chốt trục chi-phí |
| **CID-4** | **Type-code canonical: BẢNG-BYTE code, không phải thứ-tự liệt-kê văn Math §2.2.** `DidPhoenixGenerator.java:56-65` ≡ `types.ak:21-32` (Device=2…Service=7,Context=8) LÀ canonical. Văn Math §2.2 phải bám bảng-byte này (không dùng thứ-tự Context=2,Service=8). | Trung — chặn mint author-DID phi-nhân (ProofChat v2). Chưa lộ vì Person/Org/Avatar (0,1,9) trùng. Cần rà bản Rust/mobile bám đúng bảng-byte. | maintainer (Math v4.7) + đội Core/backend rà Rust/mobile |
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
- `spec-proposals/PhoenixKey-Anchor-UniquenessThread-PA2-Design.md` (2026-07-04) — PA2 sorted-list PoC + evidence ExUnit (§5.7).
- `spec-proposals/PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md` (2026-07-09) — PA2 SMT/Merkle-root chốt-hướng (§5.7bis) + FROST-Ed25519 custody (§5.10).
- `spec-proposals/ServiceDID-SelfService-and-Delegation-DRAFT.md` (2026-06-17, PROPOSAL chờ duyệt) — Op_create_service + anchoring 2 giai-đoạn (§5.11); §3 bug Full_Authority/⊑ đã fold vào Math v4.7 §3.3 (xem CID-6, §9).
- Nguồn thiết-kế nội-bộ (không công khai).
