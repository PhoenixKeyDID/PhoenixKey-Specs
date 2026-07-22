# PhoenixKey — Kiến trúc khoá 2-yếu-tố · Recovery đa-yếu-tố · AdaptiveAuth

> **Trạng thái**: thiết kế qua 4 hội đồng opus (mathematician · adversary · game-theorist · optimizer, 2026-07-22). Anh Aladin chốt: **(A)** recovery = seed ∧ yếu-tố-2 do user tự giữ (RMP mạnh), KHÔNG bắt guardian; Face/Knowme là tuỳ chọn thêm khi hoàn thiện. **Non-oracle**. **AdaptiveAuth** = danh-mục+trần on-chain / tính-toán off-chain. Bản này chờ anh review + tự merge.
> **Thay thế/đồng bộ**: `PhoenixKey-SeedDistribution-SelfCustody-Feat.md` (§3 dùng lại cơ chế RMP + client-Shamir của nó, thêm envelope-before-split — KHÔNG viết lại), `PhoenixKey-Anchorme-*` (thêm §5 datum device_pkh), `PhoenixKey-Rebirthme-*` (Rail-2 guardian), `PhoenixKey-Knowme-*` (yếu-tố tuỳ chọn §4.3), `limit_meter.ak` (§8 mở rộng — KHÔNG validator mới).

## 0. Mục tiêu (anh chốt) + sự thật nền

1. Seed KHÔNG còn là **đặc quyền duy nhất** kiểm soát DID. Kẻ có seed, **chỉ trên CLI**, KHÔNG chi được ví Phoenix/Easteregg (cưỡng chế 2-of-2 on-chain — §5).
2. Recovery = **seed ∧ ≥1 yếu-tố user tự giữ**; user tự khôi phục KHÔNG cần guardian; hacker-chỉ-seed thất bại (§4).
3. Seed **mã hoá trước khi phân tán**; gom hết mảnh công khai → không giải được (§3, chứng minh + test).
4. Thiết bị LAN = chỉ lưu mảnh (durability), không guardian/đồng thuận; chạy 1/2/3+ máy (§3).
5. **AdaptiveAuth**: cấp độ xác thực tăng theo giá trị/bối cảnh; sàn on-chain cố định, ma-sát thêm ở app (§8).

**Sự thật nền (mathematician — phải hiểu để không hứa sai):** một yếu-tố mà user tái tạo được trên **máy mới** chỉ từ dữ-liệu-công-khai + đầu-vào-người là yếu-tố **entropy thấp** (khuôn mặt ~20-30 bit, câu hỏi bảo mật <40 bit và OSINT-được). Hacker biết DID cũng tái tạo được. ⟹ **sinh trắc và Knowme KHÔNG THỂ là yếu-tố khôi phục ĐƠN LẺ** — chỉ là lớp tiện lợi AND lên trên. Yếu-tố-2 vừa mang-đi-được vừa đủ mạnh chỉ có: **(a) passphrase RMP mạnh sinh-máy (~70+ bit)**, **(b) khoá guardian**. Hướng A = chọn (a) làm mặc định không-guardian.

## 1. Hai định lý bất khả (nền)

**1.1 Tam-nan Rail-1.** Không thể đồng thời: (P1) mọi mảnh∪mọi máy LAN không đủ phục hồi Seed nếu thiếu `X`; (P2) `X` = khoá enclave-bound chết-theo-máy; (P3) máy mới phục hồi chỉ từ mảnh+sinh-trắc-máy-mới; (P4) user không giữ/nhớ gì. *Hy sinh P4 có kiểm soát:* `F_cold` = **RMP mạnh user tự cất** (mặc định hướng A) ∨ guardian (tuỳ chọn).

**1.2 Bất-khả yếu-tố-tự-khôi-phục (mắt-xích-yếu).** Recovery = seed ∧ (F₁ ∨ … ∨ Fₘ). Lợi thế hacker-có-seed = `1 − ∏(1−2^{−sᵢ}) ≥ max 2^{−sᵢ}` → **an toàn = yếu-tố YẾU NHẤT trong nhánh OR**. ⟹ **mọi Fᵢ trong nhánh OR phải độc lập đạt ≥λ (128 bit)**; yếu-tố yếu (face/Knowme) chỉ được **AND** vào một nhánh mạnh, không đứng riêng. (Spend path §5 = seed ∧ device — AND một yếu-tố 256-bit, KHÔNG dính lỗi này.)

## 2. Sơ đồ khoá thống nhất

| Khoá | Thuật toán | Sinh từ | Lưu ở | Ai dùng được |
|---|---|---|---|---|
| **Seed** (=Master_KEK) | 32B = BIP39-24từ | random | không trần; nóng=`AEAD(Device_KEK,Seed)`; lạnh=`EnvSeed` rải LampNet | đường-nóng (sinh trắc gate) / đường-lạnh (F_cold) |
| **Controller** (on-chain #1) | Ed25519 | `Ed25519.FromSeed(HKDF(Seed,"taad-controller-v1",H(DID)))` | pkh trong anchor | ai có Seed |
| **DeviceKey** (on-chain #2, **KHÔNG seed-derived**) | Ed25519/secp256k1 | **random trên máy** | priv=`AEAD(K_bio,·)`; `device_pkh` trong datum | chỉ máy đó **+ sinh trắc sống** |
| **K_bio** | P-256 non-extractable | `SecureEnclave.Generate(biometryCurrentSet)` | Secure Enclave/StrongBox | sinh trắc sống trên chính máy (gate cục bộ, KHÔNG verify on-chain) |
| **K_env** (envelope phân tán) | 32B random/DID | random khi enrol | không trần; tái tạo **chỉ từ `F_cold`** | ai có `F_cold` (off máy-LAN) |
| **F_cold** (yếu-tố-2 lạnh) | 32B | **MẶC ĐỊNH (A): `Argon2id(RMP mạnh sinh-máy)`**; TUỲ CHỌN: Shamir M-of-N guardian `ECIES(guardianDID)`; **AND thêm** (khi có): face-fuzzy / Knowme | RMP trong tay user / quorum guardian | ai có RMP / quorum |
| **Guardian** | Ed25519 (DID guardian) | thuộc DID guardian | ví guardian | chính guardian (danh tính) |

Then chốt: **chi ⟺ sig(Controller) ∧ sig(DeviceKey)**; **recovery ⟺ Seed ∧ F_cold** (F_cold ≥λ bởi RMP-mạnh hoặc guardian).

## 3. Phân tán "gom hết vô dụng" — ENVELOPE-BEFORE-SPLIT

Dùng lại nguyên cơ chế client-Shamir + LocatorID-từ-DID + **RMP** của `SeedDistribution-SelfCustody-Feat.md` (KHÔNG viết lại); thêm 1 lớp envelope:
```
K_env ←_R {0,1}^256                              # ngẫu nhiên, KHÔNG suy từ Seed
EnvSeed = AEAD(K_env, Seed ‖ H(DID))             # I-ENVELOPE-BEFORE-SPLIT
shards  = ErasureSplit(EnvSeed, k, n)            # share của CIPHERTEXT, KHÔNG của Seed
∀ shard: hybrid X25519 + ML-KEM-768 wrap          # I-PQ-WRAP (làm ngay)
upload(shards) → node LampNet công khai           # KHÔNG phụ thuộc số máy LAN
K_env bọc dưới F_cold:  wrap(K_env) = AEAD(HKDF(F_cold,"kenv"), K_env)
```
- **R1 (gom hết mảnh→vô dụng):** dựng lại `EnvSeed`; thiếu `K_env` thì phân biệt với ngẫu nhiên = phá IND-CCA của AEAD (negligible). `K_env` không nằm trong mảnh. ∎
- **R2 (gom hết MÁY→vẫn cần yếu-tố-2):** `K_env` chỉ tái tạo từ `F_cold` (RMP mạnh / guardian) — KHÔNG ở máy LAN nào. ∎
- **⚠ Điều kiện chặt (adversary):** nhánh `F_cold = RMP` thì "gom hết vô dụng" **chỉ đúng khi RMP mạnh** — shard công khai + brute offline ⟹ **RMP phải SINH-MÁY ≥70 bit (diceware), Argon2id tham số nặng, KHÔNG cho user tự nghĩ**. Rate-limit vô dụng (tấn công offline). Ghi rõ trong UX.
- **Sửa lỗi cũ** `lampnet.rs derive_x25519_static_from_kek(Master_KEK)` (khoá giải = seed, circular) → khoá giải = `K_env` off-device.
- **1/2/3+ máy:** mảnh chỉ mang durability (LT-fountain), KHÔNG secret-share → 1 máy đủ; thêm/bớt máy = re-encode + bump epoch (`K_env` bất biến); KHÔNG cần PSS.

## 4. Recovery đa-yếu-tố — Hướng A (không bắt guardian)

**4.1 Rail-1 (lấy lại SEED, mặc định A):** máy mới → user cấp `F_cold`:
- **Mặc định: RMP mạnh** (user nhập lại passphrase sinh-máy đã cất) → `K_env` → tải+giải `EnvSeed` shards → Seed → re-wrap dưới `Device_KEK` máy mới + sinh `DeviceKey` mới, cài `device_pkh` mới (§5). *Hacker-chỉ-seed:* thiếu RMP → fail.
- **Tuỳ chọn mạnh hơn: guardian M-of-N** (cho người không muốn tự giữ RMP) — Rail-2.
- **Lớp tiện lợi AND (khi Face/Knowme hoàn thiện — anh chốt "phải áp dụng thêm"):** `F_cold = Argon2id( RMP ‖ R_face ‖ H(Knowme_answers) )`, với `R_face` từ **reusable fuzzy extractor** (§4.3). AND làm ma-sát nhẹ hơn (user chỉ cần quét mặt + trả lời thay vì gõ RMP dài) NHƯNG bảo mật vẫn neo ở RMP (nhánh ≥λ). KHÔNG bao giờ để face/Knowme thành nhánh OR đứng riêng (định lý 1.2).

**4.2 Rail-2 (lấy lại QUYỀN-DID qua guardian, KHÔNG đụng seed):** guardian đồng ký `InitRecovery` cài `pending_controller_pkh`+`pending_device_pkh` tươi → timelock → `FinalizeRecovery`. `status=Recovering` **đóng băng chi**; owner máy cũ `CancelRecovery`; pending-keys công khai lúc Init. **Must-fix TRƯỚC:** vá guardian-dup (cấm phần tử trùng quorum) + timelock-floor >0 (PoC drain thật — MEMORY redteam-guardian-dup). `min_guardian_sigs` ≥2 guardian PHÂN BIỆT; thông báo bắt buộc tới owner khi Init (chống griefing đóng băng).

**4.3 Face/Knowme (tuỳ chọn, bổ trợ — chuẩn bị interface, build khi hoàn thiện):**
- **Face:** embedding client-side (KHÔNG qua enclave) → **reusable fuzzy extractor** (Canetti et al. 2016 — bắt buộc, secure sketch cổ điển rò rỉ chéo khi enrol lại) → `R_face` ~20-30 bit + helper-data `P` công khai (an toàn để lộ VÌ luôn AND với RMP ≥λ). Chống presentation/replay: liveness ở **MobileCore Lens** (§9); template KHÔNG rời thiết bị.
- **Knowme:** dùng lại module Knowme (câu hỏi bảo mật / SD-VC) — `H(answers)` là 1 hạng trong Argon2id, KHÔNG tính vào thanh entropy an toàn (OSINT-được).

## 5. Cưỡng chế 2-of-2 on-chain (spend — VỮNG chắc)

- `types.ak TAADDatum`: thêm `device_pkh: VerificationKeyHash` (f14) + `version:Int` (f15). Giữ `hw_key_pubkey` (P-256) cho attestation off-chain (Cardano KHÔNG verify P-256 — CIP-49/Builtins.hs).
- `auth_logic.anchor_controller_ok`: `is_active(status) ∧ has(sigs, controller_pkh) ∧ has(sigs, device_pkh)`. did_payment + did_stake cùng helper.
- CLI-chỉ-seed: có `sig(controller)` nhưng thiếu `DeviceKey_priv` (enclave, đòi sinh trắc) ⇒ thiếu `sig(device_pkh)` ⇒ **fail**.
- `Rotate/InitRecovery/Finalize` + **mọi op controller-uỷ-quyền** (UpdateGuardians/Deactivate/CancelRecovery/loosen-meter) đòi device co-sign (chống seed-alone chiếm ví qua UpdateGuardians→self-recovery).
- **Cổng THỜI (clean-slate):** Preprod xoá hết → `device_pkh` bắt buộc từ genesis mọi anchor mới, KHÔNG grandfather.

## 6. Bất biến + test

I-ENVELOPE-BEFORE-SPLIT · I-KENV-OFFDEVICE · I-BIO-GATE · I-DEVICEKEY-NONSEED · I-SPEND-2OF2 · I-SHARDS-PASSIVE · I-GUARDIAN-IDENTITY · I-RECOVER-TWO-RAILS · I-FREEZE-ON-RECOVERING · I-HASH-AGILITY · I-PQ-WRAP · **I-RMP-MACHINE-GEN** (RMP sinh-máy ≥70bit, Argon2id nặng) · **I-FACTOR-WEAKEST-LINK** (mọi nhánh OR recovery ≥λ; face/Knowme chỉ AND) · **I-REUSABLE-FE** (fuzzy extractor tái dùng được) · **I-FLOOR-2OF2-FIXED** (§8) · **I-RESET-BOUNDED** (§8).

Test: `test_gather_all_shards_yields_only_ciphertext` · `test_all_machines_no_fcold_cannot_recover` · `test_kenv_requires_fcold` · `test_spend_fails_only_controller` · `test_spend_fails_only_device` · `test_spend_ok_both` · `test_cli_seed_only_cannot_spend` · `test_rmp_weak_rejected` · `test_face_alone_cannot_recover` (nhánh OR face đơn bị từ chối) · `test_new_device_rail1_seed_plus_rmp` · `test_recovering_freezes_spend` · `test_reshare_epoch_bump_same_kenv` · `test_ceiling_exceed_requires_extra_factor` · `test_reset_ceiling_infinity_rejected`.

## 7. Đánh đổi / điểm yếu còn lại (thành thật)

1. **Hướng A đổi UX lấy an toàn:** user phải giữ 1 RMP mạnh (= "seed thứ 2"). Ai không muốn → chọn guardian (Rail-2). KHÔNG có đường "chỉ vân tay là xong" ở mức an toàn cao (định lý 1.2).
2. **Face/Knowme = bổ trợ**, không hạ được thanh an toàn; helper-data face lộ vô hại VÌ luôn AND RMP.
3. **Enclave bị bypass** (jailbreak/coldboot) làm sập neo sinh-trắc cục bộ — cần hardening thiết bị.
4. **Mất CẢ RMP LẪN guardian LẪN mọi máy** = mất hẳn (không backdoor — đúng chủ ý).
5. **AdaptiveAuth không cưỡng chế "tổng tài sản" on-chain** (đòi oracle — đã bỏ). On-chain chỉ ép per-lớp (§8); phần tổng là advisory off-chain.
6. Guardian-dup + timelock-floor phải vá TRƯỚC khi Rail-2 đáng tin.

## 8. AdaptiveAuth — danh-mục + trần on-chain / tính-toán off-chain (non-oracle)

**Nguyên tắc (hội đồng):** validator eUTXO cục-bộ KHÔNG biết tổng-tài-sản/tier/ngành/môi-trường (off-chain toàn cục). Nên:
- **I-FLOOR-2OF2-FIXED:** sàn on-chain = 2-of-2 (§5) cho MỌI tier, MỌI lúc, từ genesis. AdaptiveAuth **chỉ THÊM** yếu-tố, TUYỆT ĐỐI không hạ sàn. "Account mới = 1-2 factor, cho thử-sai" chỉ ở **app-UX**, không đục sàn on-chain.
- **Enforce = trần PER-LỚP theo đơn-vị-gốc** (KHÔNG oracle). Tái dùng `limit_meter.ak` (bucket/refill/loosen-delay/singleton) — mở rộng: `LimitMeterDatum` scalar → `List<ClassRule{class_id, ceiling, required_factors, bucket_Q, last_refill_slot}>` + `default_rule`; `LargeWithdraw_ok` nhị-phân → **đếm ≥ required_factors khoá PHÂN BIỆT** (tái dùng `protectme committee_ok`). Bể factor từ anchor đã load: controller/device/guardians/secondary → **0 ref-input thêm**.
- **Catalog** = 1 UTxO governance (`CatalogDatum` + catalog-NFT), đọc **chỉ ở cold-path SetConfig** (lúc user đặt/đổi trần), KHÔNG mỗi spend. GĐ1: CARP, LAMP, ADA. GĐ2: +FARM/WORK/TRACE/TIGER + user tự khai → sửa **1 UTxO, 0 re-anchor**.
- **Token ngoài catalog / không giá** → trần theo **SỐ LƯỢNG** (không quy CARP); không có `ClassRule` → rơi `default_rule` = **tier CAO** (bảo thủ: token lạ/airdrop = rủi ro cao, không tự hạ bảo vệ). Diệt trò giấu-tài-sản-để-né.
- **Ranh giới off/on:** app tính tier + gợi ý trần + chọn factor gom vào tx (chỉ để DỰNG tx); validator tự tính net-value từ `tx.outputs` (ledger-verified), đọc `ceiling` từ meter-datum của chính nó, đếm sig thật — **KHÔNG tin con số nào của app**.
- **I-RESET-BOUNDED (đảo-ngược-đặc-quyền):** user tự đặt lại trần khi ≥3 factor — NHƯNG **không lên ∞**: (a) reset đòi bộ factor ≥ bộ đang nới; (b) **ma-sát bất đối xứng** — nâng bảo mật = tức thì, **hạ bảo mật = cooling-off 24-72h + timelock (tái dùng I-LIMIT-LOOSEN-DELAY)**; (c) thông báo mọi recovery-factor (reset do bị chiếm quyền sẽ lộ); (d) **sàn cứng KHÔNG-tắt** cho thao tác thảm hoạ/bất-khả-hồi. structuring (chia nhỏ) → chặn bởi bucket cửa-sổ-thời-gian của limit_meter.
- **Ngưỡng "tài sản lớn"** = max(tuyệt-đối, tỷ-lệ-trên-net-worth); hiệu chỉnh theo dữ liệu số dư thật `[NEEDS-EVIDENCE — A/B]`, không hằng số cứng. Behavior/môi-trường chỉ **step-up** (đòi thêm 1 factor user đáp được) + luôn có fallback user-held, KHÔNG hard-lock (chống khoá-nhầm + phân biệt đối xử).

## 9. Face-bio: phối MobileCore + SuperApp (interface đã có)

- **PhoenixKey** (em): định nghĩa `knowme.face2fa.{enroll,verify,policy}` + thang LoA + reusable fuzzy extractor + neo device_pkh. Trả lời câu hỏi chặn MobileCore #6(d): **khoá enclave P-256 = wrapper gate sinh-trắc; khoá ký on-chain = Ed25519 `device_pkh` riêng** (P-256 không verify on-chain).
- **MobileCore Lens** (Long/native): `FaceCapture` = camera + liveness + chất lượng khung. Ship contract `FaceCapture` → mở khoá phần còn lại.
- **SuperApp** (Tùng): màn chụp mặt + step-up UX qua broker `capability:biometric`; KHÔNG chạm template/khoá.
- Luồng: MobileCore(capture)→FaceCapture→broker→Knowme(verify)→attestation→AND vào F_cold.

## 10. Phương án thay thế (ghi để anh cân nhắc)

**FROST group key** (= Trục B `SeedDistribution-Tech-Math`): controller = group key, device là participant bắt buộc; đóng lỗ tại gốc nhưng BỎ khôi-phục-bằng-24-từ. Anh đã chọn hướng giữ seed → để mở nếu sau muốn triệt để tối đa.

---
_Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO GS-PHOENIXKEY-01: App. No. 64/031,291._
