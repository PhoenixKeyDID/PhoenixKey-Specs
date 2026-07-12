# PhoenixKey — Rebirthme — Kỹ thuật (Tech)

> **Module:** Rebirthme (slug `Rebirthme`). **Loại doc:** Kỹ thuật cho implementer (đội on-chain / đội backend / Core rust_core). **Ngày:** 2026-07-09.
> **Đối tượng đọc:** kỹ sư triển khai. HOW: kiến trúc, datum/redeemer CBOR, điều kiện tx, luồng e2e, API, ranh giới giao việc, thứ tự deploy, test.
>
> **Ranh giới (MECE):** module CHI + địa chỉ + anti-drain + kho bí mật/phả hệ + stake theo-DID + connector + di cư + pool-ops. **Smartsend** (gửi có bảo vệ) nay là module độc lập — xem [PhoenixKey-Smartsend-Tech.md](./PhoenixKey-Smartsend-Tech.md); module này chỉ cung cấp hạ tầng nạp nguồn/guardian/anti-drain mà Smartsend tái dùng. **TAAD state-machine (genesis/rotate/recovery-mechanics)** thuộc Core Anchorme — chỉ dẫn chiếu. Xem [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) cho bất biến, `PhoenixKey-Math.md` §10/§11 cho TAAD.
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 1. Kiến trúc + sơ đồ thành phần

```
┌──────────────────────── PhoenixKey-Core (Flutter + rust_core) ─────────────────────┐
│  UI: Ví Phoenix/Standard · Guardian · Khôi-phục · Kho bí-mật · Hạn-mức · Smartsend   │
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
│  smartsend_escrow.ak          │
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

Nền tài sản: Ví Phượng hoàng = **địa chỉ script Plutus V3 stateless** (UTxO không datum). Anti-drain thêm lớp **meter-UTxO stateful** (khi bật). Khôi phục + guardian = **UTxO anchor TAAD** (thuộc Core). Backup = **CID neo trong `TAADDatum`** trỏ ciphertext trên LampNet.

---

## 2. Bất biến kiến trúc (load-bearing)

1. **Authority đọc động qua anchor ref-input** (CIP-31): mọi cổng chi (did_payment/did_stake/did_subaddr/limit_meter) quy về `anchor_controller_ok` — đọc controller HIỆN TẠI, không hardcode. → tài sản sống qua rotate (I-WALLET-1/2).
2. **Singleton-anchor**: đúng 1 ref-input mang anchor NFT; ≠1 → reject (I-WALLET-4, `auth_logic.ak:59-70`).
3. **Đóng băng theo status**: status ≠ Active → mọi chi reject (I-WALLET-3).
4. **Helper canonical dùng chung**: `auth_logic.anchor_controller_ok` (mode-1) / `auth_satisfied` (mode-2 registry, chờ tách) — did_payment/did_stake/did_subaddr/pool KHÔNG sao chép logic anchor.
5. **Meter binding**: khi bật hạn mức, kho ÉP tiêu thụ + tái tạo meter-NFT UTxO (I-LIMIT-BIND) → không đường rút "ngoài đồng hồ".
6. **Client-side crypto**: mọi bí mật mã hoá TRÊN MÁY trước khi rời (I-VAULT-1/3); node chỉ thấy ciphertext.
7. **Tách miền khoá**: P-256 (auth/login, off-chain verify) ≠ Ed25519 (controller on-chain); lỗi nonce miền auth không chạm tài sản (I-SIGN-DOMAIN-SEP).

---

## 2.A Bất biến guardian/recovery (S4 — chiếm quyền qua guardian) [CHỜ BUILD]

> Đích thiết kế cho `taad_logic.ak` (Core Anchorme sở hữu code, module này dẫn chiếu + tiêu thụ qua §4/§5.2). Nguồn PoC on-chain thật (Preview, tx `b80b394…`) đã rút được vault DID qua guardian-dup `[G,G]` + `timelock=0`. 6 tầng dưới đây là giải pháp đóng lỗ, CHƯA code — theo dõi tiến độ ở [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme).

**Tầng 1 — Đếm khoá duy nhất** (đóng lỗ guardian-dup `[G,G]`):
- `I-GUARD-UNIQUE-WRITE`: mọi điểm ghi `guardians` (Genesis-mint, `UpdateGuardians:186-198`, `Transfer.new_guardians:202-219`) assert `list.unique(new_guardians) == new_guardians`.
- `I-GUARD-COUNT-DISTINCT`: `count_signed_guardians:265-277` dedup signer (phòng thủ lớp 2, không tin riêng write-time).

**Tầng 2 — Sàn cứng tham số** (đóng lỗ `timelock=0`/`min_sigs=1`):
- `I-PARAM-FLOOR`: đầu `validate_spend` assert `min_guardian_sigs >= 2`, `recovery_timelock_slots >= FLOOR_TIMELOCK` (~7 ngày slot cho PersonDID), `recovery_collateral_lovelace >= FLOOR_COLLATERAL` — sàn là HẰNG compile-time trong validator, KHÔNG tin `ValidatorParams` apply-param (§2.4 Anchorme-Tech).
- `I-GUARD-THRESHOLD-BOUND`: tại mọi write, `min_guardian_sigs <= len(distinct(new_guardians))` VÀ (entity cần-recovery) `len(guardians) >= min_guardian_sigs >= 2` — cấm empty-guardians brick khi `min_sigs>0`.

**Tầng 3 — Guardian có anchor danh tính độc lập controller + veto** (khép I-CURVE-5):
- `I-GUARD-ROLEDID`: mỗi `guardian_pkh` là RoleDID/anchor riêng, khoá KHÔNG dẫn xuất từ Master_KEK/seed của controller (parent-sig kiểu Org/Service). Verify anchor sống tại `InitRecovery`.
- `I-GUARD-VETO`: `GuardianConfig{weight, role}` thêm vào `TAADDatum` (`schema_version` bump — [CẦN CHỐT-W9], giao Tuân). Role `ATTESTER_ALIVE` có quyền veto. Luật Finalize: `Σweight(APPROVER-signed) >= threshold ∧ không ATTESTER_ALIVE veto ∧ timelock trôi ∧ chủ chưa cancel`.
- `I-GUARD-CAP`: không guardian đơn có `weight >= threshold` (chống một guardian tự khôi phục).

**Tầng 4 — Đổi guardian-set không tức thời** (đóng lỗ đổi toàn bộ-guardian-rồi-InitRecovery-ngay):
- `I-GUARD-CHANGE-DELAY`: `UpdateGuardians` khi HẠ an toàn (bớt guardian/đổi khoá/hạ ngưỡng) đi qua trạng thái pending có timelock + cửa veto cho guardian CŨ hoặc kênh thứ hai — mở rộng phạm vi `I-LIMIT-LOOSEN-DELAY` (§4 bảng thao tác, hàng SetConfig nới) sang cả `UpdateGuardians`.

**Tầng 5 — Kinh tế + liveness cửa recovery:**
- `I-RECOVERY-SLASH`: collateral kẻ khởi `InitRecovery` bị TỊCH THU (chuyển chủ/Treasury) khi `CancelRecovery` thành công — KHÔNG hoàn pending. Chống censor-miễn phí.
- `I-RECOVERY-COOLDOWN`: cooldown on-chain giữa hai `InitRecovery` liên tiếp (chống grief-loop giam DID vĩnh viễn).
- `I-CANCEL-INDEPENDENT`: kênh Cancel/đóng băng KHÔNG chỉ dùng `controller_pkh` cũ (ca mất máy chính khoá đó không ký được) — thêm `ATTESTER_ALIVE` veto + kênh out-of-band (VeData-Glint attestation). Vào `Recovering` PHẢI phát cảnh báo out-of-band.

**Tầng 6 — `recovery_anchor` = metadata KHÔNG tin** (giữ nguyên carry-only on-chain đúng thiết kế; ràng buộc ở tầng tooling):
- Ràng buộc off-chain (không phải on-chain mới): recovery-tooling TUYỆT ĐỐI không suy khoá từ blob tại `recovery_anchor` mà không xác thực chữ ký chủ độc lập + versioning chống rollback (đối chiếu I-VAULT-4 §9). LampNet Strata verify phả hệ CID chống entry giả.

**Cơ chế 6 tầng — tái dùng primitive VeData/LampNet, KHÔNG làm lại accumulator:**
- Guardian-anchor-uniqueness (Tầng 3, `I-GUARD-ROLEDID`) → Mosaic SMT/Merkle-root (PA2).
- Phả hệ khoá (chống entry giả, Tầng 6) → Strata MMR.
- Backup blob (`recovery_anchor`) → Mirage (LT-erasure + BLAKE3).
- "Name chưa live" (namespace guardian-anchor) → Stamp SMT-non-membership.
- Ranh giới on-chain → `blake2b_256` (builtin Plutus V3).
- Phân tán khoá → FROST-Ed25519 (xem dưới).

**Khép residual seed-lộ (R1/R2, ngoài phạm vi 6 tầng — wave riêng):** phân tán khoá FROST-Ed25519 (`controller_pkh = hash(group-key)`, xuất một chữ ký chuẩn, KHÔNG đổi validator/địa chỉ) làm seed-lộ hết nghĩa với preset t-of-n; preset self-custody 1-of-1 vẫn cần Tầng 3 veto làm van. Anti-drain `limit_meter` (I-CURVE-4, §9) là van cuối cho rút lớn sau recovery.

## 2.B Khôi phục riêng tư qua Midnight — vai bổ sung [DRAFT, chưa duyệt, KHÔNG chặn Tầng 1-6]

> Nguồn: `spec-proposals/PhoenixKey-Midnight-Privacy-Feat-DRAFT.md` — **DRAFT đề xuất, chưa duyệt**, mọi chi tiết Midnight đánh dấu `[GIẢ-ĐỊNH]` chờ Phase 1 PoC kiểm chứng. Ghi ở đây để không lạc trôi khỏi module Rebirthme (guardian recovery là trọng tâm); KHÔNG phải cam kết kỹ thuật, KHÔNG đổi thiết kế Tầng 1-6 ở trên.

**Vấn đề:** hiện `TAADDatum.guardians` (pkh guardian) ghi **PUBLIC** trên metadata-6789 (Cardano). Lộ guardian-set = lộ mạng thân nhân, tạo bề mặt tấn công có chủ đích lên đúng người đó (đe doạ/hối lộ guardian). Rủi ro hiện tại THẤP (MVP chưa gắn PII khác), đây là kế hoạch phòng ngừa.

**Đề xuất (Phase 2, sau Phase 1 PoC xác nhận khả thi):** Midnight (sidechain riêng tư của Cardano, zk-SNARK, ngôn ngữ Compact) cho một **nhánh proof bổ sung** trong `InitRecovery` — chứng minh "đủ t-of-n guardian đã uỷ quyền" **mà KHÔNG lộ guardian là ai**, bằng cách guardian-set chuyển từ pkh public sang **commitment shielded**:

```
G_c = MerkleRoot({commit(pkhᵢ, saltᵢ)})        -- commit(v,salt) = H(salt ∥ v), salt ≥128-bit
prove_recovery(sigs, G_c, t) → π_rec           -- ∃ ≥t chữ ký guardian hợp lệ trên G_c, KHÔNG lộ ai
```

**Bất biến (đề xuất, DRAFT — không ghi đè I-GUARD-* Tầng 1-6 đã chốt):**
- `I-PRIV-3` (đường lui): `recover(did)` KHÔNG bao giờ phụ thuộc tuyệt đối Midnight sống — guardian-threshold public (Tầng 1-6) LUÔN là đường nền song song, mặc định bật. Midnight chết ⇒ mất tính năng riêng tư nâng cao, KHÔNG mất quyền khôi phục/tài sản.
- `I-PRIV-4` (không lộ guardian-set khi chủ chọn shield): migration guardian-set public→shielded là **tuỳ chọn user**, không ép; DID đã ghi guardian public từ trước KHÔNG undo được phần đã public — chỉ shield từ thời điểm chuyển trở đi.
- Tài sản ở Ví Phượng hoàng vẫn chi được **chỉ cần Cardano** (script address gated theo controller — không phụ thuộc Midnight, giữ nguyên §2 mục 1).

**Điều CHƯA chắc (chặn bởi Phase 1 Midnight-Feasibility, KHÔNG code trước khi có báo cáo):** verify chữ ký (P-256 Secure Enclave hay Ed25519) trong circuit Compact có khả thi/rẻ; cơ chế neo chéo Cardano↔Midnight (bridge/relay-proof/on-chain-verify SNARK — ảnh hưởng ExUnit); SLA/độ chín Midnight (mainnet mới). Owner: Tuân + đối tác Midnight, theo dõi ở [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme).

**Ranh giới:** đây KHÔNG thay Tầng 3 (`I-GUARD-ROLEDID`/`I-GUARD-VETO`/`I-GUARD-CAP`) hay FROST (PR-5) — Midnight-shielded là lớp riêng tư PHỦ LÊN guardian-set đã có, không thay cơ chế xác thực/ngưỡng.

---

**Trình tự PR (Tầng 1+2 độc lập, chạy ngay không chờ FROST):**
| PR | Nội dung | Tầng | Phụ thuộc |
|---|---|---|---|
| PR-1 | guardian-floor: unique-write + count-distinct + PARAM-FLOOR + threshold-bound + non-empty-guardians; test fail-case on-chain (`[G,G]+min_sigs=2` reject, `timelock=0` reject, `min_sigs=1` reject, empty-guardians-với-`min>0` reject, re-verify `b80b394` không tái lập) | 1+2 | Không — đóng PoC ngay |
| PR-2 | `GuardianConfig` schema_version bump + luật Finalize theo weight + `I-GUARD-ROLEDID` | 3 | Schema (Tuân) |
| PR-3 | `UpdateGuardians`-delay (pending + timelock + veto) | 4 | Sau PR-2 |
| PR-4 | Kinh-tế: SLASH + COOLDOWN + CANCEL-INDEPENDENT + cảnh báo out-of-band | 5 | Sau PR-2 |
| PR-5 | FROST-Ed25519 vào `rust_core` (keygen/sign/rotate group-key); self-custody 1-of-1 trước, LampNet t-of-n sau | R1 | Wave riêng |
| PR-6 | anti-drain `limit_meter` LOAD-BEARING cho rút lớn sau recovery | R2 | Wave riêng, đối chiếu §9 |

Mọi thay đổi validator: grep toàn bộ callers+tests+`rust_core encode_taad_datum`, update đồng thời; test pass FULL với evidence output thật trước khi báo sẵn sàng.

---

## 2.C Multi-address: 3 loại địa chỉ Ví Phượng hoàng (L1/L2/L3) [DEP-1 `did_stake` / DEP-2 `did_subaddr`]

> Nguồn: `spec-proposals/PhoenixKey-MultiAddress-Wallet-Feat.md`. Ba loại địa chỉ anh Aladin chốt — tái dùng `did_payment` cho cả ba, chỉ khác (a) stake credential đính kèm, (b) chính sách publish + cách sinh/nhóm địa chỉ. `did_stake`/`did_subaddr` đã xuất hiện ở sơ đồ §1 và bảng ranh giới §7/§8 — mục này bổ sung construction/invariant chi tiết còn thiếu (I-ADDR-*).

**Bảng M.1 — canonical:**

| # | Tên | payment cred | stake cred | Publish `service[]` | Linkable-tới-DID on-chain | Construction |
|---|---|---|---|---|---|---|
| L1 | Base công khai | `pay_did(did)` | `stake_did(did)` | CÓ | CÓ (cố-ý — danh tính) | tái dùng `did_payment` + `did_stake` |
| L2 | Enterprise công khai | `pay_did(org)` | `stake_did(org)` | CÓ | CÓ (cố-ý) | = L1, `entity_type=Org` |
| L3 | Enterprise riêng tư ×N | `pay_sub(org,i)` (per-sub) | tuỳ chọn, mặc định không stake | KHÔNG | KHÔNG (unlinkable) | `did_subaddr` mới — dưới |

```
pay_did(did)   = Script(hash(did_payment(anchor_policy, blake2b_256(did))))    -- did_payment.ak:40, LIVE
stake_did(did) = Script(hash(did_stake(anchor_policy, blake2b_256(did), 0)))   -- DEP-1, chưa code
pay_sub(org,i) = Script(hash(did_subaddr(anchor_policy, commit_i)))            -- DEP-2, chưa code
   commit_i    = blake2b_256(blake2b_256(org) ‖ tag_i)
```

L1/L2 dùng chung construction — khác duy nhất `entity_type` trong anchor datum + nhãn `service[]`; `auth_logic.anchor_controller_ok` cố-ý KHÔNG ép `entity_type` (`auth_logic.ak:16-18`) nên `did_payment` phục vụ cả Person lẫn Org bằng một script logic (khác biệt Org m-of-n nằm ở mode-2 registry, không ở payment construction).

**Mâu thuẫn cốt lõi L3** (first-principles): "OrgDID chi được TẤT CẢ" và "N địa chỉ KHÔNG lộ chung chủ" kéo ngược nhau — dùng thẳng `did_payment` cho N sub ⇒ cùng script hash ở N địa chỉ ⇒ linkable. Ba phương án cân nhắc (4 trục dài hạn / first-principles / tối ưu eUTXO-ExUnit-đơn giản / lợi ích user):

| Phương án | Ý-tưởng | Kết quả |
|---|---|---|
| PA — apply-param sub-index thẳng vào `did_payment` | `pay_sub = Script(hash(did_payment(policy, H(org‖i))))` | ❌ loại — anchor NFT thật có `name = blake2b_256(org)`, không khớp `H(org‖i)` ⇒ `anchor_controller_ok` không tìm thấy anchor, chi FAIL; muốn chạy phải mint N anchor NFT phụ (N min-ADA, N reference-script) — phình phí, mỗi anchor phụ lại là điểm lộ nếu `H(org‖i)` bị truy ra |
| PB — SHARED `did_payment`, chỉ đổi stake/diversifier | `pay_sub = pay_did(org)`, `stake = Stealth(org,i)` | ⚠️ payment part vẫn CÙNG một script hash ⇒ KHÔNG đạt unlinkable-on-chain thật (lọc theo payment credential vẫn gom được); chỉ đạt "ẩn chủ khỏi resolver công khai" (không publish) — dùng làm MVP riêng tư mức-resolver, **build NGAY** bằng `did_payment` hiện có |
| PC — validator `did_subaddr` mới: payment cred diễn dịch per-sub, authority quy về-anchor OrgDID | xem dưới | ✅ chọn cho unlinkable thật — dependency onchain MỚI (DEP-2) |

**`did_subaddr` (DEP-2, validator MỚI, purpose `spend`):**
```
did_subaddr_valid(tx, org, commit_i) ⟺
    anchor_controller_ok(tx, anchor_policy, blake2b_256(org))     -- authority quy về anchor OrgDID GỐC (1 anchor dùng chung)
  ∧ commit_i "mở" đúng: redeemer mang tag_i, blake2b_256(blake2b_256(org) ‖ tag_i) == commit_i
```
- `commit_i` khác nhau mỗi `i` → N script hash khác nhau trên chain → **NHẬN tiền hoàn toàn ẩn**, không nhóm được N địa chỉ về một chủ.
- CHI đọc reference-input đúng anchor OrgDID gốc + controller Org ký (tái dùng `anchor_controller_ok` — cùng helper canonical §2 mục 4) → OrgDID luôn chi được TẤT CẢ, sống sót rotate/recovery như payment thường.
- Trade-off eUTXO (ghi lại theo yêu cầu, không phải lỗ hổng): lúc CHI, redeemer lộ `tag_i` + tx tham chiếu anchor OrgDID → liên kết địa chỉ này↔OrgDID lộ TẠI THỜI ĐIỂM CHI đó (các sub khác không lộ theo). Không có cách để một chữ ký công khai gom N sub-address mà KHÔNG lộ rằng chúng cùng chủ — bản chất authority đọc động của eUTXO. Ẩn cả lúc chi cần Midnight/ZK (đối chiếu §2.B, ngoài phạm vi).
- Chi phí: **1** anchor OrgDID dùng chung cho mọi sub (không nở N anchor như PA); N reference-script `did_subaddr` (CIP-33 — script giống hệt nhau, chỉ khác param `commit_i`) publish khi lần đầu dùng mỗi sub, reclaim khi đóng.
- `sub_key(org,i)`/`tag_i` sinh off-chain deterministic từ Master_KEK OrgDID (mẫu HKDF, không secret mới); builder chi sub-addr mở `tag_i` + đính anchor OrgDID ref + `script_ref` (thuộc Core, §5.7 kiểu mẫu).

**Publish:** `service[]` của DID Document CHỈ chứa `public_addr` L1/L2 (payment+stake). Mọi `addr_sub`/`pay_sub` (L3) **TUYỆT ĐỐI KHÔNG** vào `service[]`/resolver/DID Document — ánh xạ `i ↔ khách-hàng ↔ addr_sub(tag_i, commit_i)` chỉ nằm ở kho riêng backend OrgDID (không endpoint công khai nào tra ngược `addr_sub → org`).

**I-ADDR-* (bổ sung — không đè I-WALLET-*/I-DIDSTAKE-*/I-CURVE-*/I-GUARD-* đã có ở §2):**
- **I-ADDR-CTRL**: mọi địa chỉ (L1/L2/L3) chi được CHỈ khi anchor Active + controller HIỆN TẠI ký — cùng cổng `anchor_controller_ok`; không có địa chỉ Phoenix nào chi ngoài cổng này.
- **I-ADDR-PUBLIC**: CHỈ L1+L2 vào `service[]` (lộ chủ là cố-ý — danh tính công khai).
- **I-ADDR-PRIV**: mọi địa chỉ L3 KHÔNG BAO GIỜ publish (service[]/resolver/DID Document); backend chặn cứng — sub-addr lọt `service[]` = vi phạm.
- **I-ADDR-UNLINK** (chỉ PC/`did_subaddr`): hai sub `i≠j` cùng OrgDID KHÔNG chia sẻ payment credential on-chain (`commit_i ≠ commit_j` → script hash khác) → không nhóm được lúc NHẬN; liên kết chỉ lộ tối thiểu (một `tag_i`) lúc CHI.
- **I-ADDR-ORGALL**: dù N sub unlinkable, OrgDID vẫn chi được TẤT CẢ — mọi `did_subaddr` quy authority về CÙNG anchor OrgDID + controller Org ký.
- **I-ADDR-STAKE-NOSPEND**: stake part (`did_stake`) KHÔNG có quyền chi vốn (purpose chỉ `Publish`/`Withdraw`) → giúp-delegate-hộ không chạm principal, trần thiệt hại nếu lạm dụng stake-authority = reward một chu kỳ, không phải principal.

**Build ngay vs chờ onchain:**

| Năng lực | Trạng thái |
|---|---|
| payment part L1/L2 | ✅ `did_payment` mode-1 LIVE (`did_payment.ak:40-55`) |
| stake part L1/L2 theo-DID | chờ **DEP-1** `did_stake.ak`; MVP tạm `stake = KeyHash(stake_key)` — delegate được, chưa theo-DID |
| riêng tư mức-resolver L3 (PB) | ✅ build NGAY — `pay_did(org)`, chỉ không đưa vào `service[]` |
| riêng tư unlinkable thật L3 (PC) | chờ **DEP-2** `did_subaddr.ak` — **maintainer phải chốt trước khi đội on-chain code** |

---

## 3. Datum / Redeemer — khuôn CBOR

### 3.1 `did_payment` (stateless)
```
validator did_payment(anchor_nft_policy: PolicyId, anchor_nft_name: AssetName)
  spend(_datum: Option<Data>, _redeemer: DidPaymentRedeemer, _own_ref, self)
DidPaymentRedeemer = Spend           -- redeemer trống; authority neo vào anchor ref-input
```
UTxO ví KHÔNG cần datum. Params apply-per-DID: `(anchor_policy, blake2b_256(did))`. Nguồn: `did_payment.ak:36-55`.

### 3.2 `TAADDatum` (dẫn chiếu Core Anchorme — các field liên quan module này)
Thứ tự field khớp `types.ak` (KHÔNG sửa — thuộc Core/Validator). Field module này ĐỌC: `did`, `entity_type`, `controller_pkh` (Ed25519, auth chi), `hw_key_pubkey` (P-256, carry-by-equality), `status`, `guardians` (≤5), `recovery_anchor` (CID). **Đề xuất thêm** (schema thuộc đội backend/Validator — [CẦN CHỐT-W3]): `vault_index_anchor : Option<CID>`, `pool_anchor : Option<CID>`, `GuardianConfig` (trọng số/vai + `schema_version`).

### 3.3 `LimitMeterDatum` (Anti-Drain §B.2)
```
LimitMeterState = Open | Frozen
LimitMeterDatum {
  did: ByteArray, limit_lovelace: Int, refill_window: Int, bucket_Q: Int,
  last_refill_slot: Int, state: LimitMeterState, safe_address_hash: ByteArray,
  large_delay: Int, secondary_pkh: Option<VKeyHash>, schema_version: Int }
```
Q = 10^9. Meter gắn meter-NFT singleton per-DID.

### 3.4 `SmartSendDatum` — nay module riêng
> `SmartSendDatum` + redeemer (Cancel/Accept/Finalize/Freeze/ReclaimTimeout) chuyển sang `PhoenixKey-Smartsend-Tech.md §3.1`.

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

## 4. Từng thao tác — điều kiện + shape tx + ai ký

| Thao tác | Validator | Điều kiện | Ai ký |
|---|---|---|---|
| **Chi Ví Phoenix** | `did_payment` | anchor Active + controller ký + ∃!anchor ref-input | controller (Ed25519) |
| **Rotate/Recovery** (Core) | `taad` | xem `taad_logic.ak`; trong Recovering ví đóng băng | controller / guardian / pending |
| **Rút nhỏ hạn mức** | `did_payment`+`limit_meter` | `W(tx)×Q ≤ available_Q`; tiêu+tái tạo meter | controller |
| **Rút lớn L1** | `limit_meter` | OpenLarge (W=0) → chờ `large_delay` → FinalizeLarge; CancelLarge bất cứ lúc nào | controller |
| **Rút lớn L2** | `limit_meter` | `W×Q > available`; +secondary_pkh HOẶC guardian ký; đốt bucket | controller + secondary/guardian |
| **Freeze/Unfreeze** | `limit_meter` | Freeze: controller∨guardian; Unfreeze: chỉ controller | controller/guardian |
| **SetConfig nới** | `limit_meter` | 🔴 chịu `large_delay` (I-LIMIT-LOOSEN-DELAY); siết áp ngay | controller |
| **Smartsend** (Open/Cancel/Accept/Finalize/Freeze/ReclaimTimeout) | `smartsend_escrow` | nay module riêng — xem [PhoenixKey-Smartsend-Tech.md §4](./PhoenixKey-Smartsend-Tech.md) | sender/receiver/guardian |
| **Stake delegate** | `did_stake` | anchor Active + controller ký; VoteDelegation Abstain kèm (Plomin) | controller |
| **Di cư ví cũ** | (không validator mới) | 1 tx gộp 4 cert Conway; tự cấp phí từ reward | ví ngoài (CIP-30) |
| **Claim LAMP** | `airdrop_pool` (LAMP) | Merkle permissionless; leaf khớp; burn slot | controller HOẶC Lace (fee) |
| **Pool create / rotate KES** | (off-chain build + TAAD auth) | `auth_satisfied(did, pool_*)`; counter monotonic | controller (biometric) |

**Shape rút nhỏ hạn mức:** inputs = {vault UTxO(s), meter UTxO} ; outputs = {đích, [trả lại kho], meter UTxO tái tạo với bucket'} ; ref-inputs = {anchor} ; signer = controller ; validity `lo` = tip slot.

---

## 5. Luồng end-to-end

### 5.1 Chi thường
`build tx` → thêm anchor ref-input + controller required-signer → ExUnits tĩnh (evaluate-then-patch) → collateral thuần-ADA → sign (Enclave) → submit.

### 5.2 Khôi phục mất máy (guardian)
Máy mới sinh controller mới → `buildInitRecovery` (đủ t-guardian ký, collateral 50 ADA, mở timelock 3600 slot) → chờ → `buildFinalizeRecovery` (controller mới ký) → tài sản ở Ví Phoenix tự về (địa chỉ bất biến). Nghi lạm dụng trong cửa sổ → `buildCancelRecovery`.

### 5.3 Backup kho bí mật
`SecretBlob` → `vault_frame` (pad+decoy) → `ECIES_Encrypt(pk_vault)` → `LampNet.put` → CID → cập nhật `VaultIndex` → neo `vault_index_anchor` (controller ký). Khôi phục: 24 từ → Master_KEK → VaultKEK → đọc `vault_index_anchor` → giải từng blob (fail-closed).

### 5.4 Di cư ví cũ
PhoenixKey enable() Lace (watch-only) → `getRewardAddresses/getUtxos` → `taad_build_legacy_migration_tx` (gộp VoteDelegation→Abstain + Withdrawal + Move→phoenix_addr + StakeDeregistration) → **UNSIGNED** → Lace `signTx` → `assemble_witness` → `submitTx`. Reward < fee → builder trả "" + báo "cần nạp X ADA". Chi tiết connector + builder + ai ký → §5.7.

### 5.5 Xoay KES pool
Nhắc lịch (75%/90% tuổi KES) → 1 chạm biometric → sinh KES period-now → ký op-cert mới (counter+1) → re-distribute LampNet TRƯỚC → xuất artefact node (KES+VRF+op-cert, KHÔNG cold).

### 5.6 LampNet — mạng lưu trữ phân tán (network layer) [PROPOSAL, chờ Long/anh Aladin duyệt]

> Nội dung mục này lấp khoảng trống "LampNet Network Specification" (Out-of-scope,
> `PhoenixKey-Math.md` dòng 48). §5.1–5.3 ở trên mô tả *client thấy gì* (ECIES →
> CID → TAADDatum); mục này mô tả *bytes đó sống ở đâu trong mạng node* và *node
> gia nhập/duy trì tư cách ra sao*. Là **đề xuất khởi điểm (giá trị số CHƯA chốt
> normative)**, ánh xạ trực tiếp code đã có ở `lampnet-join/` — không phải thiết
> kế mới từ đầu.

**Không gian định địa chỉ.** Vật lưu được định địa chỉ bằng **CID** (content-
addressed, BLAKE3 trên ciphertext). CID chiếu vào không gian 256-bit cùng
`NodeId`; một CID `x` được nhân bản lên K node có `NodeId` gần `x` nhất theo
khoảng cách XOR (Kademlia-style). **LocatorID** (§7.2 Math) là index tất định
client-side để tra cứu/nhóm — KHÔNG phải địa chỉ fetch; CID mới là địa chỉ fetch
thật, và vì CID không tất định (chứa ephemeral pubkey + GCM nonce ngẫu nhiên)
nên phải được persist bền — chính là field `recovery_anchor`/`vault_index_anchor`
trong `TAADDatum` (§3.2).

**Tham số routing đề xuất** (mỗi object có `data_class` riêng, xem bảng dưới):

```
K  ≜ replication factor — số node độc-lập giữ ≥1 mảnh của object ngay sau ghi.
β  ≜ refresh rate/epoch — tỷ lệ replica được rà-soát/re-replicate mỗi epoch,
     bù churn giữ bất-biến I-LN-K: |{node giữ ≥1 mảnh}| ≥ K.
c  ≜ over-replication coeff — số bản THỰC ghi = ⌈c·K⌉ (dư ra để chịu rớt ngay
     sau ghi).
δ  ≜ durability target — P[object không decode được trong cửa sổ T] ≤ δ.
```

| data_class | Ví dụ | K | c | δ (T=1 năm) |
|---|---|---|---|---|
| `recovery` | `recovery_anchor` (khoá controller mới, wrapped) | 8 | 1.25 | 10⁻⁶ |
| `bulk` | DID Document off-chain, `vault_index_anchor` droplet | 4 | 1.25 | 10⁻⁴ |
| `ephemeral` | cache, dữ liệu tái tạo được | 2 | 1.0 | — |

Epoch tham chiếu ≈ 432 000 slot (~5 ngày, khớp Cardano epoch). `K=8` cho
`recovery`: với xác suất một node độc lập biến mất/epoch `p≈0.2` (giả định
bi quan mạng non-trẻ), `p^K ≈ 2.56·10⁻⁶` — đủ biên cho dữ liệu sống còn.

> ⚠ **[OPEN — cần Long xác nhận trước khi chốt]:** wire-format hiện tại của
> node LampNet ghi `"redundancy": "2.5"` khi upload — chưa rõ đây là *c=2.5*
> (over-write hệ số) hay tỷ lệ `n/k` của lớp erasure-coding phía node
> (LT-codes `k=50, n=1000`, Math §7.2). Hai lớp — *replication-K ở tầng
> object* (mục này) và *erasure-coding ở tầng node* (Math §7.2, ngoài phạm vi
> client) — trực giao nhau, KHÔNG tự gộp cho tới khi backend LampNet xác nhận
> semantics field trên.

**Node admission** (ánh xạ pipeline `lampnet-join`, không viết mới): thứ tự bắt
buộc là `challenge → rate-limit (cheap-gate trước PoUW đắt) → binding
device_pubkey==attest_pubkey → attestation (chữ ký+range+PoUW-match+mode-tier
floor+TTL; Hardware→tier≥Edge, Software→cap≤Mobile) → fingerprint dedup (SAU
verify) → score (`admission_score = 0.30·s_hw + 0.25·s_net + 0.25·s_avail +
0.20·s_energy`, Σ=1.0) → quyết định theo ngưỡng động `θ(load)` → committee
commit-reveal 2-pha PBFT (quorum 2f+1, n≥7; genesis k=5,q=3) cấp
`MembershipCertificate``. Node liveness nuôi `reputation` bằng EMA bất đối xứng
(rise-slow fall-fast: α_success=0.01, α_failure=0.05); rep < 0.20 → Suspend,
loại khỏi tập nhận ghi tới khi phục hồi.

**Liên kết DeviceDID (Math §15).** Một node LampNet đã admission PHẢI có
DeviceDID on-chain: `device_class = LampNetNode`, `hw_cert ∈ {TPM2_Quote,
SEP_Cert, FIDO2_Assertion}` (nguồn cho `AttestationMode::Hardware`),
`MAX_CERT_AGE_SLOTS[LampNetNode] = 8640` slot (~12h) → node phải re-attest
định kỳ. `NodeProfile.person_did` là chỗ bind node → PersonDID chủ sở hữu, cho
phép MagicLamp trả reward đúng DID và thu hồi node qua thu hồi DeviceDID.
> ⚠ **[OPEN]**: TTL cert on-chain (8640 slot ~12h) khác bậc độ lớn với TTL
> PoUW-attestation trong `lampnet-join` (~432 000 slot ~5 ngày) — cần chốt cái
> nào ràng buộc quyền phục vụ của node, hay dùng cả hai (cert ngắn cho quyền
> ký, PoUW dài cho điểm năng lực).

**Ranh giới kinh tế (interface tới MagicLamp).** LampNet KHÔNG thiết kế
tokenomics/reward — chỉ phát ra bằng chứng đo được mỗi epoch, ký bởi committee
(chống node khai khống):

```
NodeContribution { node_did, epoch, bytes_served, replicas_held, uptime_ratio,
                    reputation }
MagicLamp.computeReward( List<NodeContribution> ) → List<(node_did, reward)>
```

`I-LN-ECON-1`: LampNet KHÔNG mint/burn/transfer token — mọi giá trị kinh-tế do
MagicLamp quyết (đối chiếu: LAMP cố định 36 tỷ, KHÔNG burn). Đổi tham số
thưởng KHÔNG kéo theo sửa phần network layer ở trên (phân lớp trách nhiệm).

---

### 5.7 CIP-30 Lace connector + builder Delegator→LAMP-claim — chi tiết CORE dev-ready (giao Tuân)

> Fold từ `spec-proposals/PhoenixKey-Delegator-Core-Connector-Builders-Feat.md` (PR CORE, `rust_core`).
> Tầng: chỉ dựng + ký tx trong `rust_core`, expose FFI `taad_*`; KHÔNG đụng backend endpoint (§6.1) hay
> Merkle-root/ISPO (LAMP). §5.4 ở trên là tóm tắt luồng; mục này là interface CBOR/FFI + bảng ai ký cụ thể.

**A. CIP-30 connector — kiến trúc 3 tầng.** `enable()` chỉ tồn tại trong JS context WebView (không phải
Rust/Dart) → cầu 3 tầng: **JS bridge** (`window.cardano.lace.enable()` → `getUsedAddresses/
getRewardAddresses/getUtxos/getChangeAddress/getNetworkId/signTx/signData/submitTx`) → **Dart facade**
`LaceConnector`/`LaceApi` (điều phối) → **rust_core FFI** `taad_*` (builder + crypto, KHÔNG gọi được
`window.cardano`). Mobile không có `window.cardano` → dùng CIP-45 pairing (Phase-2, SDK Long); API surface
Dart giữ nguyên, chỉ đổi transport.

Bảng method (đường ranh watch-only vs trigger-popup):

| Method CIP-30 | Loại | Lace bật popup? | Trả về |
|---|---|---|---|
| `enable()` | cấp quyền | CÓ (1 lần/phiên) | api object |
| `getUsedAddresses/getRewardAddresses/getUtxos/getChangeAddress/getNetworkId` | watch-only | KHÔNG | addr/UTxO CBOR hex, networkId |
| `signTx(cbor, partial)` | request-sign | CÓ | `witness_set_cbor_hex` |
| `signData(addr, payload)` | request-sign | CÓ | `{signature, key}` COSE |
| `submitTx(cbor)` | broadcast | KHÔNG (có thể xin xác nhận) | `tx_hash` |

`signData` Phase-1 = **passthrough** (Dart gọi thẳng `api.signData` của Lace, Lace tự dựng COSE_Sign1
Ed25519, KHÔNG P-256 — P-256 chỉ dành login-native PhoenixKey, luồng riêng). Builder COSE
`taad_cose_sign1_build` (controller Ed25519 ký) CHỈ cần cho Hướng-2 (PhoenixKey là bên ký cho dApp ngoài) —
Phase-2, không chặn Delegator claim.

FFI hỗ trợ (rust_core, module `connector.rs`):
```rust
taad_assemble_witness(unsigned_tx_cbor_hex, lace_witness_set_cbor_hex, merge: u8) -> tx CBOR hex | ""
  // merge=0: set (thay); merge=1: union vkeys (multi-witness — fee-sponsorship/claim mode-lace)
taad_cip30_utxos_to_builder_json(cip30_utxo_cbor_hex_json) -> JSON [{tx_hash,index,lovelace,assets}]
```

Bất biến: **CORE-CONN-1** watch-only KHÔNG bao giờ bật popup/trả bí mật. **CORE-CONN-2** chỉ
`signTx`/`signData` khiến Lace ký — PhoenixKey không giữ khoá Lace, không có auto-sign vô hạn (đối chiếu
I-CONN-1). **CORE-CONN-3** Rust không gọi `window.cardano`, chỉ nhận CBOR làm tham số. **CORE-CONN-4**
`signData` Phase-1 = passthrough Ed25519, builder COSE = Phase-2.

**B. `taad_build_legacy_migration_tx` — di cư 4-trong-1.** Một tx Lace ký MỘT lần, gộp (1) VoteDelegation
(`old_stake_cred`→DRep, mặc định `abstain`, bỏ nếu `already_vote_delegated`) ĐẶT TRƯỚC (2) Withdrawal
(reward account cũ → R thành input ngầm, tự cấp phí), (3) Move phần dư (R−fee−deposit) → đích
`derive_phoenix_address(did, anchor_policy_hex, network)` qua `add_change_if_needed(dest)` — **KHÔNG**
add_output amount cứng, (4) StakeDeregistration tuỳ chọn (hoàn 2 ADA). Builder **KHÔNG** `finalize_and_sign`
(không có seed ví cũ) — trả **UNSIGNED** tx CBOR (witness_set rỗng) cho Lace `signTx` ký, rồi
`taad_assemble_witness(merge=0)`. `old_utxos` rỗng VÀ `reward=0` → `""`. `build()` FAIL khi
`R + Σin < fee + deposit` → trả `""` (KHÔNG đẩy tx chắc-FAIL cho Lace ký), Dart báo "cần nạp thêm X ADA".

FFI: `taad_build_legacy_migration_tx(old_reward_addr_hex, old_stake_cred_hex, old_utxos_json,
reward_lovelace, already_vote_delegated: u8, drep_id, include_deregister: u8, dest_kind:"phoenix"|"franken",
did, anchor_policy_hex, old_change_addr_hex, protocol_params_json, network) -> unsigned tx CBOR hex | null`.

Bất biến: **CORE-MIGR-1** trả UNSIGNED, không tự ký (I-MIGR-ONESIG). **CORE-MIGR-2** VoteDelegation trước
Withdrawal trong tx (Plomin). **CORE-MIGR-3** Move = change về đích DID, không output cứng. **CORE-MIGR-4**
reward<fee+deposit → `""` (I-MIGR-SELFUND). **CORE-MIGR-5** LC1 (`phoenix_addr`) dùng
`derive_phoenix_address` khớp vector aiken; LC2 (`franken_addr`) cảnh báo giam vốn ở UI.

**B.1 Hai đích `dest_kind` + điều kiện tiên quyết + chặn cứng (mở rộng B, dev-ready).**

`dest_kind="phoenix"` (LC1, full-migrate) — `derive_phoenix_address(did, anchor_policy_hex, network)` =
`Address{ payment=Script(hash(did_payment(anchor_policy, blake2b_256(did)))), stake=Script(hash(did_stake(anchor_policy,
blake2b_256(did)))) }`. PhoenixKey kiểm soát CẢ vốn+staking+voting; **không cần ví cũ/seed về sau**. Yêu cầu
`did_payment` đã deploy (Phase 2) — builder PHẢI kiểm DID đích `status=Active` trước khi set output, DID chưa
Active → `""`.

`dest_kind="franken"` (LC2, giữ vốn) — `derive_franken_address(did, k, old_payment_cred_hex, anchor_policy_hex,
network)` = `Address{ payment=<payment cred VÍ CŨ, giữ nguyên>, stake=Script(hash(did_stake(anchor_policy,
blake2b_256(did), k))) }` (DID-Stake B.1 `stake_cred(did,k)`). PhoenixKey chỉ kiểm soát staking/voting; **vốn vẫn
dưới payment cũ** → chi/exit vốn về sau VẪN cần extension cũ ký. Vì ca gốc của module này là mất-seed, LC2 chỉ
hợp lý khi user cố-ý giữ extension cũ làm két — UI PHẢI cảnh báo rõ: nếu sau này mất luôn password/extension cũ,
vốn ở `franken_addr` bị giam vĩnh viễn (không đường always-exit vì seed đã mất từ đầu). Không phát StakeRegistration
mới nếu `franken_addr` đã register sẵn (I-DIDSTAKE-NOREGDUP) — tránh deposit phát sinh ngoài dự tính fee.

**Chặn cứng khi build (đối chiếu I-MIGR-DEAD/A.6 nguồn):**
- Extension cũ mất hẳn (quên password/gỡ extension/mất máy) → không còn đường ký payment cũ → **bất khả tuyệt đối**,
  Core không có cơ chế bù (không seed, không xprv). UI phải cảnh báo SỚM "di cư ngay khi còn ký được", không phải
  lỗi builder trả về.
- Ví cũ không expose `signTx` cho tx có certificate/withdrawal (chỉ ký tx chi thường) → builder chính KHÔNG dùng
  được. Fallback theo thứ tự: (a) `signTx(partial=true)` nếu ví hỗ trợ partial witness, assemble như bình thường;
  (b) nếu ví hoàn toàn không ký được cert/withdraw → **fallback 2 bước thủ công**: hướng dẫn user tự rút
  reward + vote-delegate trong UI ví cũ trước (đưa reward về ADA ở địa chỉ cũ), rồi Core dựng **tx move thường**
  (chỉ chi UTxO, không cert/withdrawal — mọi ví CIP-30 đều hỗ trợ) cho ví ký. Không có builder gộp riêng cho
  nhánh này — dùng builder chi thường hiện có, `dest` vẫn qua `derive_phoenix_address`/`derive_franken_address`.
- `stake_cred` cũ chưa vote-delegate VÀ ví không ký được VoteDelegation cert → không có đường vòng (Plomin là luật
  ledger) — bắt buộc rơi vào fallback (b) ở trên (vote-delegate qua UI ví cũ trước).

**C. `taad_build_delegator_lamp_claim_tx` — spend Merkle claim ETD → route LAMP về ví Phoenix.** Validator
`airdrop_pool` (LAMP, `ledger.ak`+`merkle.ak`) — Core đóng gói, KHÔNG tự verify proof:

```
Claim redeemer = Constr 0 [ claimer: Address, amount: Int, proof: List<ProofStep> ]
ProofStep       = Constr 0 [ is_left: Bool, hash: ByteArray(32) ]
leaf = blake2b_256( 0x00 ++ cbor.serialise(claimer_address) ++ amount_be_8 )   -- amount_be_8 = big-endian 8B
node = blake2b_256( 0x01 ++ left ++ right )
```

Tx PHẢI: (1) spend POOL UTxO (POOL NFT) redeemer `Claim{claimer,amount,proof}`; (2) spend **claim-slot
UTxO** tên = `leaf(claimer,amount)` + **burn** NFT slot đó (`AirdropNftRedeemer::BurnSlot`, `mint == -1`) —
chống double-claim (spend-once); (3) output POOL: `value = pool_in − amount LAMP`, datum giữ nguyên trừ
`claimed_count += 1`; (4) output claimer = `amount` LAMP về `claimer = derive_phoenix_address(did,
anchor_policy_hex, network)` — **CORE-CLAIM-1**: LAMP route đúng ví Phoenix, không địa chỉ khác. Core PHẢI
tự tính `leaf` (để chọn/burn đúng slot UTxO) dù không tự verify proof.

Ai ký (`sign_mode ∈ {controller, lace}`): claim tx tự nó permissionless (validator không đòi chữ ký
claimer) nhưng input trả phí+collateral cần chữ ký chủ nó — **Trường hợp A (khuyến nghị mặc định):** phí
từ ví Phoenix (đã migrate) → controller ký (Master_KEK-derived) + reference-input anchor (mirror
`mint_lamp.rs`) → Core trả **SIGNED**. **Trường hợp B:** phí từ Lace (chưa migrate) → Lace ký phần đó
(`signTx partial=true`) → Core trả **UNSIGNED**, `assemble_witness(merge=1)`.

FFI: `taad_build_delegator_lamp_claim_tx(did, anchor_policy_hex, amount_lamp, proof_json, pool_utxo_json,
slot_utxo_json, airdrop_pool_script_hex, airdrop_nft_policy_script_hex, lamp_policy_hex, lamp_name_hex,
fee_utxos_json, sign_mode:"controller"|"lace", master_kek_hex, anchor_utxo_json, protocol_params_json,
network) -> tx CBOR hex (signed nếu controller / unsigned nếu lace) | null`.

Bất biến: **CORE-CLAIM-1** claimer output = `derive_phoenix_address` (khớp vector aiken). **CORE-CLAIM-2**
redeemer + `ProofStep` + leaf/node hash khớp `merkle.ak` byte-for-byte (prefix `0x00`/`0x01`); Core không tự
verify proof. **CORE-CLAIM-3** burn đúng slot NFT `(airdrop_nft_policy, leaf)` qty −1. **CORE-CLAIM-4** POOL
output bảo toàn datum trừ `claimed_count+1`, value = `pool_in − amount LAMP`. **CORE-CLAIM-5** ExUnits/fee
tĩnh (evaluate-then-patch), collateral thuần-ADA. **CORE-CLAIM-6** mode controller → reference-input anchor
+ `add_required_signer(controller_pkh)`; mode lace → unsigned + merge. Shape cuối
`AirdropNftRedeemer::BurnSlot`/marker redeemer là interface-contract — LAMP xác nhận trước khi Core freeze
builder.

**D. Bảng signing model tổng hợp:**

| Builder | Ai ký input-spend | Ai ký cert/withdraw | Reference-input? | Core trả |
|---|---|---|---|---|
| `taad_build_legacy_migration_tx` | Lace (payment cũ) | Lace (stake cũ) | không | UNSIGNED |
| `taad_build_delegator_lamp_claim_tx` — mode controller | controller (Core) | — (permissionless) | CÓ (anchor) | SIGNED |
| `taad_build_delegator_lamp_claim_tx` — mode lace | Lace (phí) | — | không | UNSIGNED |
| staking.rs (5 builder hiện có) | seed (Core) | seed (Core) | không | SIGNED |

`CORE-SIGN-1`: input từ ví Lace/seed-cũ ⇒ Lace ký, Core UNSIGNED (I-CONN-1, Core không có khoá đó).
`CORE-SIGN-2`: input từ ví Phoenix (`did_payment` script) ⇒ controller ký + reference-input anchor, Core
SIGNED. `CORE-SIGN-3`: input base-addr seed DID (staking.rs) ⇒ seed ký, SIGNED. `CORE-SIGN-4`: tx đa nguồn
(fee-sponsorship migration / claim mode-lace) ⇒ multi-witness UNSIGNED, mỗi bên ký phần mình,
`assemble_witness(merge=1)` hợp nhất.

**Ranh giới:** CORE (Tuân) chỉ build/ký/FFI trên. KHÔNG thuộc: backend `stake-state`/Grant/`merkle-proof`/
`claim/submit` (Long, §6.1); Merkle-root + luật ETD/ISPO + shape cuối `BurnSlot`/marker (LAMP). Test bắt buộc
(unit + Preview e2e) — xem §9.

---

## 6. API backend (tham chiếu — prefix `/api/v1`, JSON snake_case, `DataResponse<T>{code,message,result}`)

| Endpoint | Việc | Nguồn |
|---|---|---|
| `GET /identifiers/{did}` | resolver DID Document (payment/stake, recovery_anchor, vault_index_anchor) | live + mở rộng service[] |
| `POST /wallet/standard/register` · `GET /wallet/standard/{did}` | đăng ký + đọc ví Standard (CIP-1852 fixed/active/stake) | Wallet-API-v2 §2 |
| `GET /wallet/{did}/all` | gộp mọi ví + magic{source:vault} | Wallet-API-v2 §3 |
| `PUT/GET/POST /consent/*` | Grant store (route_stake_reward, cip30 session); point-in-time controller | Delegator-Offchain §1 |
| `GET /stake-state/{stake_cred}` | registered/vote_delegated/reward/pool (Blockfrost) | Delegator-Offchain §2 |
| `POST /claim/submit` · `GET /claim/{id}/status` | submit claim (KHÔNG ký) + theo dõi | Delegator-Offchain §3 |
| `GET /airdrop-claim/{addr}/{epoch}/merkle-proof` | serve proof (khớp merkle.ak byte) | Delegator-Offchain §4 |
| `POST /connect/mandate` · `/connect/challenge` · `/connect/epoch-proof` | on-ramp chữ ký | Connect-Onramp §7 |
| `GET /v1/telemetry/map/:cid` | bản đồ shard cho UI kho/khôi phục | UI-Spec S8 |

Lưu-ý: `/wallet/magic/claim` → **410 Gone** (MAGIC = account-trong-Vault, không native). `BalanceResponse` field MAGIC = 0 (deprecated).
→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

### 6.1 Chi tiết Delegator-Claim offchain (Grant/stake-state/claim/merkle) — dev-ready cho Long

> Nguồn đầy đủ: `spec-proposals/PhoenixKey-Delegator-Claim-Offchain-Feat.md`. Mục này fold phần
> request/response schema + ErrorCode để Long code thẳng, không phải mở file rời. Base path
> `/api/v1`; field JSON snake_case; envelope `DataResponse<T>{code,message,result}` (code 1000
> = ok); backend KHÔNG giữ khoá, KHÔNG ký thay (bất biến OFF-KEY-1 — chỉ điều phối + submit CBOR
> đã ký).

**B1 — Consent/Grant store (bước 3).** Schema `DelegationToken` (JSON, snake_case, lưu nguyên
văn khi `PUT`):

```json
{
  "grantor_did": "did:phoenix:...",      // chủ thẩm quyền (ký Grant)
  "grantee":     "did:phoenix:... | addr1... | agent:tiger",
  "action_tag":  "route_stake_reward",   // enum khoá backend — CHỈ tag này cho luồng này
  "resource":    "stake1u... | pool1... | *",
  "nonce":       "b64u-128bit",          // chống replay, unique/grantor
  "valid_from":  1893456000,             // unix sec
  "valid_until": 1896048000,             // unix sec
  "proof": { "alg": "Ed25519", "kid": "did:phoenix:...#controller", "sig": "hex(64B)" }
}
```

`proof.sig` ký trên canonical JSON của Grant TRỪ `proof` (keys sorted, no whitespace, UTF-8, số
nguyên không thập phân) với prefix domain-separation bắt buộc `"PHOENIXKEY_GRANT:" +
canonical_json_bytes` — tái dùng util canonical-form của `SignRequestController.approve`.

```
PUT  /consent/{grantor_did}                              [auth: grantor]
  body: DelegationToken (đã ký)
  → DataResponse<{ nonce, stored_at_slot }>
  400 INVALID_ACTION_TAG | 400 GRANT_EXPIRED (valid_until ≤ now)
  403 GRANT_SIG_INVALID  | 409 GRANT_NONCE_REUSED

GET  /consent/{grantor_did}?grantee=&action=&resource=   [auth: grantor]
  → DataResponse<{ grants: DelegationToken[] }>   // CHỈ Grant SỐNG (chưa hết hạn + chưa thu hồi)

POST /consent/verify                                     [public]
  body: { grant: DelegationToken }   (hoặc { grantor_did, nonce } để verify Grant đã lưu)
  → DataResponse<{ valid: bool, reason: string }>
     reason ∈ { ok, sig_invalid, expired, revoked, not_yet_valid, unknown_grantor,
                controller_mismatch }

POST /consent/revoke                                     [auth: grantor]
  body: { grantor_did, nonce, sig }  // sig trên "PHOENIXKEY_REVOKE:"+grantor_did+":"+nonce
  → DataResponse<{ revoked_at_slot }>
  403 REVOKE_SIG_INVALID | 404 GRANT_NOT_FOUND
```

Logic verify (dùng chung `PUT` + `/verify`), thứ tự bắt buộc:
1. Resolve controller **HIỆN TẠI** của `grantor_did` qua indexer. Nếu Grant gắn `valid_from` cụ
   thể và controller đã rotate → dùng controller **point-in-time** tại `valid_from` (backend giữ
   lịch sử rotate slot→controller_pkh); `kid` ≠ controller đúng mốc → `reason=controller_mismatch`.
2. Check `proof.sig` trên canonical bytes bằng controller pubkey đúng mốc → sai → `sig_invalid`.
3. Check hạn `valid_from ≤ now ≤ valid_until` → ngoài → `not_yet_valid`/`expired`.
4. Check bảng revoke theo `(grantor_did, nonce)` → có → `revoked`. Trạng thái luôn SỐNG (revoke
   tức thì, không cache stale).

Bất biến: **GRANT-STORE-1** node chỉ lưu Grant đã ký + cờ thu hồi, không lưu khoá/tài nguyên.
**GRANT-VERIFY-2** verify PHẢI point-in-time controller — rotate không vô hiệu Grant ký hợp lệ
trước rotate, và khoá cũ không ký được Grant mới (mốc = `valid_from`). **GRANT-REVOKE-3** revoke
tức thì, phải do controller HIỆN TẠI ký. **GRANT-NONCE-4** nonce unique/grantor, trùng → `409`.
**GRANT-TAG-5** chỉ nhận `action_tag="route_stake_reward"`, tag khác → `400`.

**B2 — Stake-state indexer (bước 2).**

```
GET /stake-state/{stake_cred}?epoch=                     [public]
  stake_cred = stake1... hoặc keyhash hex 28B; epoch mặc định = hiện tại
  → DataResponse<{
      registered: bool, vote_delegated: bool, reward_lovelace: long,
      stake_deposit: long, pool_id: string, as_of_slot: long }>
  404 STAKE_CRED_NOT_FOUND

GET /identity/{did}/stake-status                         [auth: owner]
  → DataResponse<{ did, phoenix_address, stake_states: [...], total_reward_lovelace: long,
      as_of_slot: long }>
```

Populate qua Blockfrost `GET /accounts/{stake_addr}`: `registered`←`active`, `pool_id`←`pool_id`,
`stake_deposit`←`controlled_amount` (2 ADA nếu registered, else 0), `reward_lovelace`←
`withdrawable_amount`, `vote_delegated`←`drep_id≠null` (Conway); `as_of_slot`←`GET
/blocks/latest`. MVP: proxy trực tiếp Blockfrost (không indexer worker riêng); nếu cache theo
epoch thì `reward_lovelace` PHẢI fresh (builder tự cấp phí sai → tx FAIL).

Bất biến: **STAKE-1** `reward_lovelace` = `withdrawable_amount` (không phải tổng số dư), builder
migration dùng làm withdraw amount. **STAKE-2** `vote_delegated` phản ánh Conway DRep delegation
(Plomin), Core dùng để bỏ/giữ VoteDelegation cert. **STAKE-3** `stake_deposit=2 ADA` chỉ khi
`registered=true`.

**B3 — Claim orchestration (bước 7).** Mirror `ActivationController.submitTx` + `/status` + SSE.
Backend nhận CBOR **đã ký hoàn chỉnh**, submit qua Blockfrost, track tới confirmed.

```
POST /claim/submit                                        [auth: owner]
  body: { did, tx_kind: "claim"|"migration", signed_tx_cbor, grant_nonce? }
  → DataResponse<{ claim_id, tx_hash, status: "SUBMITTED" }>
  400 INVALID_TX_CBOR | 403 GRANT_INVALID (grant_nonce fail verify §B1)
  409 ALREADY_SUBMITTED (idempotent theo tx_hash)
  502 NODE_SUBMIT_REJECT{reason}

GET  /claim/{claim_id}/status                              [public — claim_id opaque UUID 128-bit]
  → DataResponse<{ claim_id, tx_hash, status, confirmations, lamp_amount?, reason? }>
     status ∈ { SUBMITTED, ON_CHAIN, CONFIRMED, FAILED }

GET  /claim/{claim_id}/events   (text/event-stream)        [public]
  SSE: { type, payload }   type ∈ { submitted, on_chain, confirmed, failed }
```

Vòng đời: `SUBMITTED` → (thấy mempool/block) `ON_CHAIN` → (≥N conf, mặc định 1 preprod)
`CONFIRMED` (phát SSE `confirmed` + đọc `lamp_amount` tại ví Phoenix) | submit reject/rollback →
`FAILED{reason}`.

Bất biến: **CLAIM-SUBMIT-1** backend KHÔNG ký/KHÔNG chỉnh CBOR, submit `signed_tx_cbor` y nguyên
(OFF-KEY-1). **CLAIM-SUBMIT-2** idempotent theo `tx_hash` — resubmit → `409` + trả lại `claim_id`
cũ. **CLAIM-SUBMIT-3** nếu có `grant_nonce` → verify Grant TRƯỚC submit, fail → `403`, không
submit. **CLAIM-SUBMIT-4** error model chuẩn: `retryable` cho `NODE_SUBMIT_REJECT`/`NODE_BEHIND`
(app rebuild-and-retry) vs fail-cứng `INVALID_TX_CBOR`.

**B4 — Merkle-proof serving (bước 4).** Backend SERVE proof từ cây LAMP đã publish; KHÔNG dựng
root.

```
GET /airdrop-claim/{address}/{epoch}/merkle-proof          [public]
  address = ví Phoenix (addr1...), PHẢI khớp `address` trong leaf snapshot; epoch = đợt airdrop
  → DataResponse<{
      leaf: "hex32",                                   // blake2b_256(0x00 ++ serialise(addr) ++ amount_be_8)
      proof: [ { is_left: bool, hash: "hex32" } ],      // ProofStep[] — khớp merkle.ak byte-for-byte
      amount_lamp: long, merkle_root: "hex32",
      pool_ref: { policy, name } }>
  404 NOT_IN_TREE
  409 ALREADY_CLAIMED (slot đã burn — spend-once, tránh tx chắc-FAIL)
```

Leaf/proof PHẢI khớp `merkle.ak` byte-for-byte (`is_left`, `hash` 32B, `amount_be_8`) — backend
không tự hash lại theo cách khác. `merkle_root` đọc từ cây publish PHẢI đối chiếu với root ở
datum POOL on-chain (indexer) — lệch → cảnh báo cây stale. Dependency ngoài: LAMP team publish
format cây + kênh phân phối + xác nhận leaf `address` = ví Phoenix (không phải địa chỉ Lace cũ);
chưa có → test dùng snapshot 1-leaf tay (mock, khớp Core §5.2 gate).

Bất biến: **MERKLE-SERVE-1** backend chỉ SERVE, không dựng/sửa root (chủ quyền LAMP; root datum
POOL là chân lý). **MERKLE-SERVE-2** leaf/proof/amount_lamp khớp byte `merkle.ak` — sai 1 byte →
validator reject. **MERKLE-SERVE-3** leaf `address` = ví Phoenix; backend từ chối serve nếu
`address` truyền vào ≠ địa chỉ trong leaf.

**Grant gate claim — 2 lớp độc lập (§5.2 nguồn).** Backend-check (`grant_nonce` tại
`/claim/submit`) là **soft gate**: chặn sớm + audit, KHÔNG thay validator. On-chain check
(validator `airdrop_pool`, permissionless Merkle) là **hard gate thật**: validator không đọc
Grant PhoenixKey, chỉ verify proof đúng + output đúng `claimer`. An toàn tài sản nằm ở validator
+ spend-once, không phụ thuộc backend-check (**OFF-AUTH-1**).

Auth tổng hợp theo endpoint: `PUT /consent/{did}` + `POST /consent/revoke` → grantor (Bearer
session, sub=grantor_did) + Grant/revoke tự ký; `GET /consent/{did}` → grantor (riêng tư);
`POST /consent/verify` → public; `GET /stake-state/{cred}` → public (dữ liệu on-chain công
khai); `GET /identity/{did}/stake-status` → owner; `POST /claim/submit` → owner; `GET
/claim/{id}/status`+`/events` → public (claim_id opaque); `GET /airdrop-claim/.../merkle-proof`
→ public.

---

## 7. Ranh giới giao việc

| Team | Việc |
|---|---|
| **đội on-chain** | `limit_meter.ak` (I-LIMIT-*) + sửa `did_payment.ak` thêm nhánh binding meter (I-LIMIT-OPTIN); `did_stake.ak` (I-DIDSTAKE-*); `did_subaddr.ak` (I-ADDR-UNLINK, **chờ maintainer chốt DEP-2**); helper `W(tx)`. **KHÔNG sửa `did_payment.ak` mode-1** ngoài nhánh opt-in. Builder Delegator (migration/claim) + 2 FFI on-ramp (`taad_cose_sign1_verify`, `taad_addr_matches_pubkey`). Pool KES (rust_core, **[VERIFY] crate KES/VRF**). (`smartsend_escrow.ak` (SS-*) nay ở module Smartsend — [PhoenixKey-Smartsend-Tech.md §7](./PhoenixKey-Smartsend-Tech.md).) |
| **đội backend** | resolver mở rộng `service[]` (L1/L2, chặn cứng L3 — I-ADDR-PRIV); schema `vault_index_anchor`/`recovery_anchor`/`pool_anchor` + update-path (controller ký); wallet API v2; consent/Grant store (point-in-time controller); stake-state indexer; claim orchestration; merkle-serve; metering nanogic; telemetry passthrough; giám sát `r` lặp (I-SIGN-NO-REUSE — low-s I-SIGN-LOWS ép ở `crypto.rs:339-349`). |
| **Core (rust_core/Flutter)** | builder chi có-meter + freeze/SetConfig; `vault_frame`/`vault_kek`/VaultIndex + FFI kho; export re-key (`exportRevokesSeed=true` mặc định — I-WALLET-6); màn Guardian/Khôi phục/Kho/Hạn mức/Phả hệ; CIP-30 connector; pool KES builder. Enforce I-CURVE-5 (guardian/secondary khác gốc seed). (Màn + builder Smartsend nay ở module Smartsend — [PhoenixKey-Smartsend-Tech.md §7](./PhoenixKey-Smartsend-Tech.md).) |
| **VeData / Glint (ZK)** | VC-Glint recovery. (Factor bối cảnh Smartsend nay ở module Smartsend — [PhoenixKey-Smartsend-Tech.md §7](./PhoenixKey-Smartsend-Tech.md).) |
| **LampNet** | Strata engine + `GET /audit`; Mirage `EncryptedDistributed` (durability I-VAULT-8); adapter neo. |
| **MAGIC / CARP** | tokenomics nanogic, giá/nguồn trả phí kho (ngoài phạm vi module). |
| **Core Anchorme** | TAAD state-machine, genesis/rotate/recovery-mechanics, GuardianConfig schema, PA2/PA5 — module này CHỈ dẫn chiếu. |

---

## 8. Thứ tự deploy + phụ thuộc chặn

**Build được không chờ onchain mới:**
1. Mở rộng resolver `service[]` (L1/L2 payment part) + I-ADDR-PRIV chặn sub-addr.
2. Wallet API v2 (Standard register, `/all`, deprecate MAGIC).
3. Kho bí mật blob đơn (nền `lampnet.rs`) — Phase 1.

**Chờ dependency ONCHAIN MỚI (thứ tự khuyến nghị, mỗi bước độc lập giá trị):**
1. **`limit_meter.ak`** (anti-drain) — **ưu tiên CAO NHẤT** (I-CURVE-4 load-bearing). Cần sửa `did_payment.ak` thêm nhánh binding (opt-in).
2. **`did_stake.ak`** → stake part theo-DID + đa-ISPO [DEP-1].
3. **`did_subaddr.ak`** → L3 unlinkable [DEP-2] — **chờ maintainer chốt trước khi đội on-chain code**. Cùng validator mà Easteregg gọi "Tầng 0" (địa chỉ riêng) — xem `PhoenixKey-Easteregg-Tech.md §1`.
4. Registry-lib mode-2 (Org m-of-n) [DEP-3] — không chặn MVP single-controller.

(`smartsend_escrow.ak` nay ở module Smartsend — thứ tự deploy + phụ thuộc anti-drain (1) khai ở [PhoenixKey-Smartsend-Tech.md §8](./PhoenixKey-Smartsend-Tech.md).)

**Phụ thuộc chặn ngoài:** CARP policy-id (CARP team) cho balance; stake-state (đội backend) cho di cư/claim; cây Merkle LAMP (LAMP team) cho claim; `vault_index_anchor` schema (đội backend) cho khôi phục kho; crate KES/VRF Rust cho pool.

---

## 9. Test / evidence bắt buộc khi land

- `did_payment.ak`: 8 test spend (a-e + rotate-survival + 2-anchor + datum-ignored) đối chiếu I-WALLET-1..5, T-WALLET-1/2.
- `taad_logic.ak`: guardian Init/Cancel/Finalize/UpdateGuardians/Transfer + status-gate.
- `lampnet.rs`: round-trip / tamper / wrong-key fail-closed đối chiếu I-VAULT-4.
- `sign.rs`: determinism Ed25519 + sign/verify đối chiếu I-SIGN-ED25519-LIB (kèm test-vector RFC 8032).
- `crypto.rs`: P-256 verify low-s — reject high-s malleable + accept canonical low-s đối chiếu I-SIGN-LOWS.
- `limit_meter.ak`: rút>available reject; chia-100-output né NET reject; tx-2-cùng-block tranh meter (double-spend) reject; meter-NFT giả reject; chi kho không-meter reject; SetConfig limit=∞ rồi rút reject (chịu delay); Frozen→đích≠safe reject; refill-dùng-`hi` reject. PASS: rút nhỏ trừ bucket đúng; L1 sau delay; L2 ký kép; Cancel pending; Freeze→safe→Unfreeze.
- `did_stake.ak`: register-lặp, withdraw-no-vote (Plomin), anchor giả, Revoked đóng băng.
- Pool: op-cert verify, counter tụt→reject, recover→khớp, sai KEK→fail-closed, KES period sai→reject.
- Connector/builder (§5.7, rust_core unit — mirror `staking.rs #[cfg(test)]`): `assemble_witness_merges_lace_witness`, `assemble_witness_union`, `cip30_utxos_to_builder_json_roundtrip`; `migration_builds_unsigned_with_vote_withdraw_move`, `migration_skips_vote_when_already_delegated`, `migration_dereg_refunds_deposit`, `migration_insufficient_reward_returns_empty`, `migration_dest_phoenix_matches_vector`,
`migration_dest_franken_matches_vector`, `migration_dest_did_not_active_returns_empty`; `claim_leaf_matches_merkle_ak`, `claim_redeemer_shape`, `claim_output_routes_to_phoenix`, `claim_burns_slot_nft`, `claim_pool_output_conserves`, `claim_controller_mode_has_reference_input`. Preview e2e bắt buộc: migration (Lace test → assert reward=0 + UTxO ở `phoenix_addr` + Plomin gỡ, ghi txid) + claim (assert LAMP tại `phoenix_addr` + `claimed_count+1` + slot burned + claim-lần-2-cùng-leaf FAIL, ghi txid).
- Backend Delegator-Claim endpoints (§6.1 B1-B4, Preview real — cần DID PhoenixKey Active + ví
  Phoenix, 1 stake cred preprod thật có reward + đã register, Blockfrost preprod key
  `phoenixkey.cardano.*`): (1) ký `DelegationToken{action_tag:"route_stake_reward"}` qua
  `/sign/request` (mobile) → `PUT /consent/{grantor_did}` → `200{nonce}`; `POST /consent/verify`
  → `{valid:true,reason:"ok"}`; sửa 1 byte `proof.sig` → `{valid:false,reason:"sig_invalid"}`.
  (2) `GET /stake-state/{stake1...}` → đối chiếu `reward_lovelace` trực tiếp với Blockfrost
  `withdrawable_amount`, `registered`/`vote_delegated` đúng thực tế; `GET
  /identity/{did}/stake-status` gộp đúng. (3) Core builder ký claim (mode controller) → `POST
  /claim/submit{signed_tx_cbor}` → `{claim_id,tx_hash,status:SUBMITTED}` → poll `GET
  /claim/{id}/status` tới `CONFIRMED` + SSE phát `confirmed`; assert `amount` LAMP tại
  `phoenix_address`; resubmit cùng tx → `409 ALREADY_SUBMITTED`. (4) `POST /consent/revoke` →
  `200`; `/verify` cùng Grant → `{valid:false,reason:"revoked"}`; `GET /consent/{grantor_did}`
  không còn Grant đó (chỉ Grant sống). Evidence bắt buộc mỗi bước: curl request/response +
  txid confirmed + LAMP balance trước/sau. Nếu cây Merkle LAMP (§6.1 B4) chưa publish → dùng
  snapshot 1-leaf tay cho bước (3), ghi rõ mock (khớp Core §5.2 gate).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 10. Khung màn UI (Core Flutter) + bảng bridge-method — dẫn chiếu UI-Spec

> Nguồn: `spec-proposals/PhoenixKey-UI-Spec-Backup-Recovery-Lifecycle.md` (23 mục đầy đủ cho
> team UI `PhoenixKey-Core`). §7 bảng ranh giới ở trên tóm 1 dòng "màn Guardian/Khôi phục/Kho/
> Hạn mức/Phả hệ" — mục này fold khung màn + bảng bridge-method để implementer biết UI gọi
> builder nào, không cần mở file rời. Nguyên tắc thiết kế UI nguồn (không lặp lại chi tiết ở
> đây, chỉ dẫn chiếu): KHÔNG bắt user ghi/xuất seed; khôi phục = uỷ quyền rotate KHÔNG dựng lại
> Master_KEK; xuất seed = tuỳ chọn, seed hiện ra PHẢI đã bị vô hiệu với DID; Master_KEK + sinh trắc
> không bao giờ rời thiết bị — khớp I-WALLET-6/I-CURVE-5 đã có ở §2/§7.

**10.1 Bảng màn → bridge method (Dart, rust_core FFI):**

| Màn | Nội dung | Bridge method (✅ đã có trong `rust_core`/`enclave_bridge.dart`) |
|---|---|---|
| S1 thiết lập guardian | thêm/xoá guardian, đặt ngưỡng `t`, trao đổi pkh qua QR (§10.3) | `buildUpdateGuardians` |
| S2 cấu hình phương thức khôi phục | bật LampNet `recovery_anchor` / ngưỡng guardian / VC-Glint / Midnight | (đọc/ghi cấu hình; VC-Glint/Midnight tích hợp sau) |
| S3 khôi phục (mất máy) | InitRecovery (đủ t-guardian ký) → chờ timelock → FinalizeRecovery; nghi lạm dụng → CancelRecovery | `buildInitRecovery`, `buildCancelRecovery`, `buildFinalizeRecovery` |
| S4 xuất seed (re-key) | Export mặc định chạy RotateSeed (`generateMasterKek` + rotate controller) + MigrateAsset trước khi hiện seed CŨ (đã vô hiệu với DID); cờ `exportRevokesSeed=true` mặc định | `unlockMasterKek`, `generateMasterKek`, build rotate, `buildSignedTransfer` |
| S5 nhắc thiết lập sao lưu | grace 30 ngày (I-RECOVERY-4) khi genesis chưa có phương thức khôi phục; quá hạn chặn mềm thao tác giá trị cao (enforce phía client/backend, KHÔNG phải on-chain — xem gap Tầng-5 §2.A) | (đọc trạng thái, dẫn S1→S2) |
| S6 vòng đời DID | rotate/deactivate/guardian/recovery gộp một màn | `buildUpdateGuardians`, `buildDeactivate`, `buildInitRecovery`, `buildCancelRecovery`, `buildFinalizeRecovery` |
| S9 thao tác ví (Gửi/Nhận/Staking/Bỏ phiếu) | tách khỏi danh tính (S6) và thẩm quyền (Authority Dashboard) | `buildSignedTransfer` (Gửi); staking/voting builder CHƯA có |

Chung mọi tx: ExUnits TĨNH → evaluate-then-patch trước submit; collateral thuần-ADA (khớp §5.1).

**10.2 Mô hình ví — tên gọi thống nhất Core/backend:** ba cấp phân biệt **Wallet ⊃ Account ⊃
Address**. **Phoenix Wallet** (Ví Phượng hoàng) = ví theo danh tính, payment credential là script
hash `did_payment` (§2.C, §3.1) — rotate/recovery đổi khoá vẫn chi được vì authority đọc động qua
anchor ref-input. **Standard Wallet** (Ví tiêu chuẩn) = ví theo cụm seed (BIP39 + CIP-1852), bên
trong có **Rotation Account** (account cố định giữ controller key của DID — cái được "rotate" là
khoá controller BÊN TRONG account này qua TAAD rotate on-chain, không phải chỉ số dẫn xuất) và
**Imported Account** (khoá import từ ví/CLI ngoài, độc lập dẫn xuất Rotation). Khớp danh xưng
Wallet-API-v2 (§6: `wallet/standard/register` dùng CIP-1852 fixed/active/stake).

**10.3 Chuẩn QR/deep-link handshake giữa hai app PhoenixKey** (dùng cho `guardian-invite`,
device-link, permission-request, login — S1/S6 và Liên kết thiết bị): URI
`phoenixkey://<action>?<params>` + QR mã hoá cùng payload. Actions: `guardian-invite` (pkh, did,
label, role, nonce, exp) · `device-link` (did, ephemeral_pub, nonce, exp) · `permission-request`
(Grant chưa ký, khớp schema §6.1 B1) · `login` (SignRequest). Handshake: bên A hiện QR/deep-link →
bên B quét → xem rõ nội dung → ký/đồng-ý → trả về A qua deep-link callback (mobile) hoặc relay
backend (poll). Mọi payload có `nonce` + `exp` chống phát lại; **KHÔNG truyền bí mật trong QR**
(chỉ pubkey/pkh/challenge). Định dạng: JSON gọn → base64url trong QR; chữ ký P-256 (login/assert)
hoặc Ed25519 (grant/tx) tuỳ action — khớp tách miền khoá I-SIGN-DOMAIN-SEP (§2 mục 7).

**10.4 Error-states chuẩn cho mọi màn ký** (khớp error-model backend §2.8 nguồn, đối chiếu
`CLAIM-SUBMIT-4` §6.1):

| Lỗi | Khi nào | Loại |
|---|---|---|
| `WrongPin` | AES-GCM tag fail | fail-cứng — không tự thử lại |
| `SeGateUnavailable` | Secure Enclave bận/huỷ sinh trắc | retryable — "Thử lại", KHÔNG rơi về khoá sai |
| `InsufficientAda` / `CollateralMissing` | thiếu ADA / không có UTxO thuần-ADA | fail-cứng — hướng dẫn nạp ADA / tách UTxO |
| `ExUnitsTooLow` / `ScriptReject` | evaluate-then-patch lệch / validator từ chối | rebuild+retry, hoặc báo "thao tác không hợp lệ cho DID này" |
| `NetworkTimeout` / `NodeBehind` | Blockfrost/submit | retryable — backoff, giữ tx đã ký để gửi lại |
| `NonceUsed` | phát lại | tạo nonce mới |

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai).
Code: `PhoenixKey-Validator/validators/did_payment.ak`, `lib/phoenixkey/{auth_logic,taad_logic,types}.ak`; `rust_core/src/{lampnet,crypto,sign}.rs`.
Tài liệu cùng bộ: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Rebirthme-Exec.md](./PhoenixKey-Rebirthme-Exec.md).

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
