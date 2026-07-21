# PhoenixKey — Kiến trúc khoá 2-yếu-tố + Phân tán seed "gom hết vô dụng"

> **Trạng thái**: thiết kế đã qua 2 hội đồng (mathematician + adversary, opus, 2026-07-22) — anh Aladin chốt hướng **B+C** (giữ seed là 1 yếu-tố + cưỡng chế yếu-tố-2 on-chain + khôi phục 2-rail). Bản nháp chờ anh duyệt trước khi build đại trà.
> **Thay thế/đồng bộ**: `PhoenixKey-SeedDistribution-*` (Trục C cũ "Shamir mảnh trần" bị **thay** bởi §3 envelope-before-split), `PhoenixKey-Anchorme-*` (thêm §5 2-of-2 datum), `PhoenixKey-Rebirthme-*` (Rail-2 recovery).

## 0. Mục tiêu (anh chốt)
1. Seed KHÔNG còn là **đặc quyền duy nhất** kiểm soát DID. Kẻ có seed, **chỉ trên CLI**, KHÔNG rút được ví Phoenix/Easteregg.
2. Seed **mã hoá trước khi phân tán**. Gom đủ/hết mảnh trên LampNet công khai → **không giải mã được**.
3. Lấy được **tất cả máy trong LampNet-LAN** → **vẫn phải có vân tay của user** mới giải.
4. Tuyên bố được + **chứng minh bằng test**: "phân tán khoá thật gom hết vẫn vô dụng".
5. Thiết bị LAN = **chỉ lưu mảnh**, không guardian, không đồng thuận. Guardian = **danh tính** riêng. Chạy 1/2/3+ máy, ≥1 online, reshare động.

## 1. Định lý bất khả (nền — phải hiểu trước)
**Tam-nan Rail-1 (đường lấy-lại-SEED).** KHÔNG thể đồng thời cả bốn: (P1) {mọi mảnh}∪{mọi máy LAN} không đủ phục hồi Seed nếu thiếu yếu-tố `X`; (P2) `X`=khoá enclave-bound, đòi vân tay sống, non-extractable, **chết theo máy**; (P3) máy mới phục hồi Seed chỉ từ {mảnh}∪{vân tay máy mới}∪{không gì mang theo}; (P4) không bắt user nhớ/mang bí mật nào.
*Chứng minh:* P1 cần `X`; P2 khiến `X` không tồn tại ngoài máy cũ; P3+P4 khiến máy mới không có và không suy ra được `X` ⟹ Seed bất khả phục hồi. ∎
**Hệ quả:** phải hy sinh 1 trong 4. Thiết kế hy sinh **P4 có kiểm soát**: yếu-tố lạnh `F_cold` **ký gửi guardian** (user không phải nhớ gì; tuỳ chọn passphrase cho self-custody), VÀ tách **Rail-2** (khôi phục QUYỀN-DID qua guardian, không đụng seed) để mục tiêu "khôi phục" vẫn sống.

## 2. Sơ đồ khoá thống nhất
| Khoá | Thuật toán | Sinh từ | Lưu ở | Ai dùng được |
|---|---|---|---|---|
| **Seed** (=Master_KEK) | 32B ngẫu nhiên = BIP39-24từ | random | KHÔNG trần; nóng=`AEAD(Device_KEK,Seed)`; lạnh=`EnvSeed` rải LampNet | đường-nóng (vân tay) / đường-lạnh (F_cold) |
| **Controller** (on-chain #1) | Ed25519 | `Ed25519.FromSeed(HKDF(Seed,"taad-controller-v1",H(DID)))` | pkh công khai trong anchor | ai có Seed |
| **DeviceKey** (on-chain #2, **KHÔNG seed-derived**) | Ed25519/secp256k1 | **random trên máy** | priv=`AEAD(K_bio,·)`; `device_pkh` trong datum | chỉ máy đó **+ vân tay sống** |
| **K_bio** | P-256 non-extractable | `SecureEnclave.Generate(biometryCurrentSet)` | Secure Enclave/StrongBox | vân tay sống trên chính máy |
| **K_env** (envelope phân tán) | 32B ngẫu nhiên/DID | random khi enrol | KHÔNG trần; tái tạo từ `F_cold` | chỉ ai có `F_cold` (off-device) |
| **F_cold** (yếu-tố lạnh) | 32B | MẶC ĐỊNH Shamir M-of-N, mỗi mảnh `ECIES(guardianDID_pub,·)`; TUỲ CHỌN `argon2id(passphrase)` | ở **guardian DID** (không phải LAN) / trong đầu user | quorum guardian / user nhớ |
| **RecoveryRootKey** | 32B | `HKDF(F_cold,"recovery-root-v1",H(DID))` | không lưu | ai có `F_cold` → định vị pointer |
| **Guardian** | Ed25519 (DID guardian) | thuộc DID guardian | ví guardian | chính guardian (danh tính, không phải máy) |

Then chốt: **chi ⟺ sig(Controller) ∧ sig(DeviceKey)**; confidentiality backup-lạnh neo `K_env` (chỉ từ `F_cold`, off-máy-LAN); `K_bio` là neo vân-tay cho bản nóng.

## 3. Phân tán "gom hết vô dụng" — ENVELOPE-BEFORE-SPLIT (sửa lỗi gốc Trục C + lampnet.rs)
```
K_env ←_R {0,1}^256
EnvSeed = AEAD(K_env, Seed ‖ H(DID))          # I-ENVELOPE-BEFORE-SPLIT
shards  = ErasureSplit(EnvSeed, k=50, n=1000)  # share của CIPHERTEXT, KHÔNG của Seed
∀ shard: hybrid X25519 + ML-KEM-768 wrap        # I-PQ-WRAP (làm ngay, không chờ Cardano)
upload(shards) → node LampNet công khai          # KHÔNG phụ thuộc số máy LAN
F_cold: Shamir M-of-N → ∀ guardian_i: ECIES(guardianDID_i, share_i)
pointer: locator_id = HKDF(RecoveryRootKey, DID‖"lampns-v1")
```
- **R1** (gom hết mảnh→vô dụng): dựng lại `EnvSeed`; không `K_env` thì phân biệt với ngẫu nhiên = phá IND-CCA của AEAD (negligible). `K_env` độc lập thống kê, không nằm trong mảnh. ∎
- **R2** (gom hết MÁY→vẫn cần vân tay): bản nóng gate `gate`←`K_bio`-biometric (thiếu vân tay ⇒ không dựng `Device_KEK`); bản lạnh gate `K_env`←`F_cold` (không ở máy LAN nào). ∎
- **Sửa lỗi cũ**: `lampnet.rs:120 derive_x25519_static_from_kek(Master_KEK)` khiến khoá giải = seed (circular) → **đổi**: khoá giải phân tán = `K_env` (off-device), KHÔNG suy từ Master_KEK.
- **Hệ quả 1/2/3+ máy**: mảnh chỉ mang **durability** (LT-fountain trên node công khai), KHÔNG phải secret-share → **1 máy cũng đủ**; thêm/bớt máy LAN = cache tiện lợi, re-encode + bump epoch (K_env/Seed bất biến); **KHÔNG cần PSS** (gom-across-epoch vô hại vì không phải secret-share). Threshold-custody thật (khoá đầy đủ chưa từng tồn tại) là **mục tiêu KHÁC** (FROST/Trục B) — KHÔNG trộn vào MVP.

## 4. Hai RAIL khôi phục (giải R2 vs "khôi phục máy mới")
- **Rail-1 (lấy lại SEED)**: máy mới → user cấp `F_cold` (guardian release M-of-N share / passphrase) → `K_env,RecoveryRootKey` → định vị + tải `EnvSeed` shards → giải → Seed → re-wrap dưới `Device_KEK` máy mới. *Kẻ trộm mảnh không đi được:* thiếu `F_cold`.
- **Rail-2 (lấy lại QUYỀN-DID, KHÔNG đụng seed)**: guardian đồng ký `InitRecovery` cài `pending_controller_pkh`+`pending_device_pkh` tươi (enclave máy mới) → timelock → `FinalizeRecovery`. Tài sản sống (địa chỉ Phoenix bất biến). *Kẻ trộm không đi được:* cần chữ ký guardian-DID; `status=Recovering` **đóng băng chi**; owner giữ máy cũ `CancelRecovery`; pending-keys công khai lúc Init.

## 5. Cưỡng chế 2-of-2 on-chain (R4)
- `types.ak TAADDatum`: thêm `device_pkh: VerificationKeyHash` (+ `version` field cho di-trú). Giữ `hw_key_pubkey` (P-256) cho attestation off-chain (Cardano KHÔNG verify P-256 — đã xác minh CIP-49/Builtins.hs).
- `auth_logic.anchor_controller_ok`: `is_active(status) ∧ has(sigs, controller_pkh) ∧ has(sigs, device_pkh)`. did_payment + did_stake cùng helper.
- CLI-chỉ-seed: có `sig(controller)` nhưng KHÔNG có `DeviceKey_priv` (enclave, đòi vân tay) ⇒ thiếu `sig(device_pkh)` ⇒ **fail**.
- `Rotate/InitRecovery/Finalize` mang thêm `pending_device_pkh`. Rotate thường đòi controller cũ ∧ device cũ; mất máy ⇒ buộc Rail-2.
- **Cổng THỜI (clean-slate)**: Preprod xoá hết → `device_pkh` **bắt buộc từ genesis** mọi anchor mới, KHÔNG grandfather, KHÔNG lỗ 1-factor. Bảng quyền: đổi-device_pkh giữ bởi (controller∧device) hoặc (quorum guardian); KHÔNG tác nhân nào giữ quyền tịch-thu tài sản.

## 6. Bất biến + test chứng minh R3
I-ENVELOPE-BEFORE-SPLIT · I-KENV-OFFDEVICE · I-BIO-GATE · I-DEVICEKEY-NONSEED · I-SPEND-2OF2 · I-SHARDS-PASSIVE · I-GUARDIAN-IDENTITY · I-RECOVER-TWO-RAILS · I-FREEZE-ON-RECOVERING · I-HASH-AGILITY · I-PQ-WRAP.
Test bắt buộc (chứng minh mục tiêu 1-4): `test_gather_all_shards_yields_only_ciphertext` · `test_all_devices_no_fingerprint_no_cold_cannot_recover` · `test_kenv_requires_fcold` · `test_bio_gate_absent_unwrap_fails` · `test_spend_fails_only_controller` · `test_spend_fails_only_device` · `test_spend_ok_both` · `test_cli_seed_only_cannot_spend` · `test_new_device_rail2_no_seed_no_shards` · `test_recovering_freezes_spend` · `test_pq_layer_classical_broken_still_safe` · `test_reshare_epoch_bump_same_kenv`.

## 7. Đánh đổi / điểm yếu còn lại (thành thật)
1. **P4 hy sinh**: quorum guardian M-of-N phục hồi được Seed (bound bởi M-of-N + timelock + freeze + Cancel; guardian còn phải cài device_pkh on-chain).
2. **R2 sập nếu enclave bị bypass** (jailbreak/coldboot/DFU) — `K_bio` là toàn bộ neo vân-tay; cần hardening thiết bị (FLAG_SECURE, jailbreak-detect, không lưu gate ngoài enclave).
3. **Mất CẢ F_cold LẪN guardian LẪN mọi máy** = mất hẳn (không backdoor — đúng chủ ý).
4. **Phụ thuộc**: F_cold-guardian + Rail-2 đòi **vá guardian-dup + timelock-floor TRƯỚC**.
5. **ML-KEM = defense-in-depth** tại transport/at-rest, không thay AEAD lõi.

## 8. Phương án thay thế A (ghi để anh cân nhắc — triệt để hơn nhưng bỏ khôi-phục-bằng-24-từ)
Controller = **FROST group key**, device là participant bắt buộc; BỎ seed=controller ⟹ seed KHÔNG bao giờ tự dựng lại quyền chi (đóng lỗ tại GỐC, không cần sửa validator). Đổi lại: mất BIP39 Mode B ("gõ 24 từ là xong"), recovery = re-DKG qua guardian. **Em chọn B+C** vì khớp yêu cầu anh "khôi phục với bộ seed"; A để mở nếu anh muốn triệt để tối đa.
