# PhoenixKey — Phân tán seed + khoá trên LampNet — Tech + Math [N]

> **Module:** hạ tầng custody (xuyên Anchorme/Rebirthme). **Đối tượng:** kỹ sư mật
> mã + auditor + đội on-chain/offchain. **Loại doc:** Tech + Math (có phần hình
> thức hoá). **Ngày:** 2026-07-12.
>
> **Mục tiêu:** đặc tả **đầy đủ + có cơ sở khoa học** cách PhoenixKey phân tán
> (a) **dữ liệu** seed (bền bỉ, durability) và (b) **khoá** ký (custody, không nơi
> nào giữ khoá đủ) trên mạng LampNet, kèm mô hình đe doạ và đánh giá đối kháng để
> chuyên gia đọc **không còn hoài nghi**.
>
> **Neo nguồn (đã đối chiếu):**
> - `PhoenixKey-Math.md` §6 (phân cấp khoá), §6.1.1 (HKDF `wallet-seed-v1`), §7
>   (LampNet LT-code), §8.1 (CIP-1852).
> - Đề xuất thiết kế: `Legacy/spec-proposals-2026-07-10/PhoenixKey-SeedDistribution-FROST-and-PA2-SMT-Design.md`,
>   `Legacy/spec-proposals-2026-07-10/PhoenixKey-Seed-Lineage-Strata-Feat-Math.md`.
> - Mã thật: `PhoenixKey-Core/Enclave/rust_core/src/{cardano.rs,sign.rs}`,
>   `derive-demo/src/main.rs`; primitive LampNet `Mirage` (LT-fountain), `Strata`
>   (MMR phả hệ).
>
> **Nuance cốt tử (không được vi phạm ở bất kỳ đâu trong doc):**
> 1. Mã hoá **CLIENT-SIDE**. LampNet chỉ lưu **ciphertext + metadata không-bí-mật**.
>    Plaintext seed/khoá **KHÔNG BAO GIỜ** rời máy dưới dạng rõ.
> 2. **Single-party-derive bị CẤM.** Không thực thể đơn lẻ nào — kể cả máy gốc, kể
>    cả toàn bộ LampNet — được phép ở vị trí tự mình dựng lại khoá ký đầy đủ (trục B).
> 3. **Không backdoor, không master key** dựng-lại-được từ một nơi.

---

## 0. TL;DR (kết luận trước)

1. **Hai trục KHÁC NHAU, không trộn.** **Trục A** = phân tán *dữ liệu* seed để
   **bền bỉ** (mất node không mất backup) — dùng mã fountain LT (Luby Transform)
   rải *ciphertext* `EncSeed`. **Trục B** = phân tán *khoá ký* để **custody** —
   dùng chữ ký ngưỡng FROST-Ed25519 (t-of-n) sao cho khoá đầy đủ **chưa từng tồn
   tại** ở một nơi.
2. **Trục A không phải custody.** LT-fountain chỉ rải bản-mã; máy gốc vẫn giữ khoá
   giải mã. Nó lo *durability của backup*, KHÔNG lo *"không nơi nào giữ khoá đủ"*.
   Muốn đạt mục tiêu 2 (§nuance) phải **thêm** trục B.
3. **FROST-Ed25519 là lựa chọn then chốt vì nó VÔ HÌNH với on-chain.** Nhóm t-of-n
   ký ra **một chữ ký Ed25519 chuẩn** trên **group public key**; ledger Cardano
   verify y hệt chữ ký đơn → `controller_pkh = blake2b_224(PK_group)` → **KHÔNG đổi
   validator, KHÔNG đổi địa chỉ ví**.
4. **Bất biến "device BẮT BUỘC" cần đúng cấu trúc.** Một FROST *phẳng* t-of-(n+1)
   **không** tự động bắt buộc device tham gia (§3.3 chứng minh). Để bảo đảm device
   luôn phải ký, dùng **cấu trúc phân tầng 2-of-2**: `device` **∧** `quorum
   t'-of-n' LampNet`. Đây là điểm **nâng cấp** so với đề xuất gốc.
5. **Hai preset.** (1) LampNet t-of-n (custody phân tán thật); (2) self-custody
   (device 1-of-1 — bước đệm giữ interface, **chưa** đạt trục B, nói thẳng).
6. **DKG loại hai điểm hở của Shamir.** Sinh khoá phân tán (DKG) → khoá đầy đủ chưa
   từng tồn tại lúc tạo; ký ngưỡng → không dựng lại khoá lúc dùng. Nhất quán với
   quyết định "đã bỏ Shamir" (Rebirthme I-WALLET-8), không đảo ngược.

---

# PHẦN I — CƠ CHẾ (Tech)

## I.1 Hai trục custody — phân biệt dứt khoát

| | **Trục A — phân tán DỮ LIỆU** | **Trục B — phân tán KHOÁ** |
|---|---|---|
| Mục tiêu | *durability* (mất node ≠ mất backup) | *custody* (không nơi nào giữ khoá đủ) |
| Cơ chế | mã fountain LT / erasure (Mirage) | chữ ký ngưỡng (FROST-Ed25519 + DKG) |
| Đối tượng rải | `EncSeed` = **bản-mã** của seed | **share** = mảnh khoá toán học (chưa ghép) |
| Máy gốc | giữ khoá giải mã (Device_KEK) | **KHÔNG** giữ khoá ký đầy đủ |
| Hiện trạng | ĐÃ có (`Math.md` §7, `lampnet.rs → /mirage/put`) | đích mới (doc này) |
| Nguồn khoa học | Luby 2002 (LT), Rabin 1989 (IDA) | Shamir 1979, Komlo–Goldberg 2020 (FROST) |

**Điểm mấu chốt:** hai trục **xếp chồng**, không thay nhau. Trục B vẫn cần trục A để
**rải backup của từng share** (durability của share). Trục A vẫn cần trục B để đạt
*custody không-single-point*.

## I.2 Trục A — `EncSeed` + mã fountain LT (Luby Transform)

Chuỗi chuẩn bị (client-side, khớp `Math.md` §6.2–§6.4):

```
-- 1. Khoá mã hoá thiết bị (không rời máy):
Device_KEK = HKDF( key  = Cloud_Secret ∥ HW_UID,
                   info = "device-kek-v1",
                   salt = H(DID ∥ "device-kek-salt-v1") )        ∈ {0,1}^256

-- 2. Mã hoá xác thực seed (AEAD — AES-256-GCM / ChaCha20-Poly1305):
EncSeed = Enc( Device_KEK, Master_KEK ∥ H(DID) )

-- 3. Mã fountain LT: sinh n droplet từ EncSeed (Math.md §7.1):
Droplets = LT_Encode( EncSeed, k = 50, n = 1000 )
           -- phân phối bậc: Robust Soliton, μ = k, δ = 0.01
Upload( Droplets, LocatorSecret ) → LampNet    -- rải các node khả dụng

-- 4. Khôi phục (Math.md §7.2):
raw       = Download( LocatorSecret, min_count = 60 )   -- xin dư k+10 cho biên
EncSeed'  = LT_Decode( raw, k = 50 )
Master_KEK = Dec( Device_KEK, EncSeed' )
```

**Tính chất trục A:**
- LampNet node **chỉ thấy droplet của `EncSeed`** — bản-mã AEAD. Không có
  `Device_KEK` (nằm trong Keychain phần cứng, `Math.md` §6.2) thì droplet **vô
  nghĩa** — không giải được, không phân biệt được với ngẫu nhiên.
- `LocatorSecret = HKDF(HW_UID ∥ DID, "lampnet-locator-v1")` **không** truyền dạng
  rõ (I-LAMP-3); tái dẫn xuất từ phần cứng + DID.
- **Đây là durability, KHÔNG phải custody:** máy gốc lúc dùng vẫn nắm `Master_KEK`
  đầy đủ trong RAM. Vì thế mới cần trục B.

## I.3 Trục B — chữ ký ngưỡng FROST-Ed25519

### I.3.1 Vì sao FROST — và vì sao "vô hình on-chain"

Ledger Cardano verify chữ ký **Ed25519 native** qua `extra_signatories`. Ví
"đi-theo-DID" chi hợp lệ ⟺ `anchor_controller_ok`: đọc `controller_pkh` từ anchor
ref-input rồi ép `controller_pkh ∈ tx.extra_signatories` (`auth_logic.ak:37-52`,
I-WALLET-1). `controller_pkh` = hash của **một** Ed25519 public key.

**FROST** (Flexible Round-Optimized Schnorr Threshold — Komlo & Goldberg, SAC
2020; chuẩn hoá IRTF [RFC 9591](https://www.rfc-editor.org/rfc/rfc9591)) trên đường
cong Ed25519 cho phép `n` participant giữ `n` share; bất kỳ **t** trong số họ cùng
ký → **một chữ ký Schnorr/Ed25519 CHUẨN** trên **group public key** `PK_group`. Chữ
ký đó **không phân biệt được** với chữ ký đơn khoá.

```
controller_pkh := blake2b_224(PK_group)          -- (mẫu pool-id: pkh = hash(vk))
```

Hệ quả:
- Ledger verify chữ ký FROST **y như** chữ ký đơn → **KHÔNG** cần multisig-script,
  **KHÔNG** đổi `did_payment`/`did_stake`/`auth_logic`.
- Địa chỉ ví = `f(compile-param validator)`. FROST **không** đổi compile-param →
  **script-hash bất biến → địa chỉ bất biến** (vẫn neo `blake2b_256(did)`).
- Toàn bộ phân tán khoá xảy ra **off-chain** (DKG + ký 2 vòng qua transport
  LampNet). Chain chỉ thấy **1 pubkey, 1 sig** → giữ **privacy** (không lộ `n`/`t`).

> **Thư viện:** đề xuất Zcash Foundation [`frost-ed25519`](https://github.com/ZcashFoundation/frost)
> (đã audit; hiện thực hoá RFC 9591). `rust_core` hiện chỉ có Ed25519 đơn
> (`ed25519-dalek`, `sign.rs`). FROST là **superset** — chữ ký ra vẫn Ed25519,
> `ed25519-dalek`-verify được → tương thích ngược. **CẤM tự hiện thực FROST.**

### I.3.2 Sinh khoá phân tán (DKG) — khoá đầy đủ chưa từng tồn tại

Trước khi ký, nhóm chạy **DKG** (Distributed Key Generation, kiểu Pedersen —
Pedersen, CRYPTO'91; gia cố Gennaro et al. 2007). Mỗi participant `i`:
1. Sinh đa thức bí mật bậc `t−1`, phát **commitment** (Feldman/Pedersen VSS) +
   gửi **share** mã hoá cho từng participant khác.
2. Kiểm chứng share nhận được khớp commitment (chống rogue-share).
3. Tổng hợp → `sk_i` (share cá nhân) + `PK_group` (công khai, ai cũng tính được).

**Bất biến DKG (INV-DKG):** khoá bí mật nhóm `sk = Σ f_i(0)` **không bao giờ được
lắp ráp** ở bất kỳ máy nào — mỗi bên chỉ giữ `sk_i`. `PK_group = sk · G` tính được
công khai **mà không** cần `sk`.

### I.3.3 Ký ngưỡng (2 vòng) — không dựng lại khoá

FROST ký 2 vòng (khớp RFC 9591):
- **Vòng 1 (commit):** mỗi ký-viên `i ∈ S` (|S| ≥ t) sinh nonce `(dᵢ, eᵢ)`, phát
  commitment `(Dᵢ, Eᵢ)`.
- **Vòng 2 (share ký):** coordinator gộp commitment → challenge `c`; mỗi `i` trả
  `zᵢ`; coordinator gộp `z = Σ λᵢ zᵢ` (λ = hệ số Lagrange) → chữ ký `(R, z)`.

**Bất biến ký (INV-SIGN):** không vòng nào lắp `sk`; chữ ký sinh **trực tiếp** từ
các `zᵢ`. So Shamir cũ (ghép ≥t share để **dựng lại** khoá rồi ký) — FROST loại
đúng "điểm nguyên vẹn" đó.

## I.4 Một khung — hai preset + bất biến "device bắt buộc"

```
                      DKG  →  PK_group  →  controller_pkh = blake2b_224(PK_group)
                               │
          ┌────────────────────┴─────────────────────┐
 PRESET (1) LampNet-default                  PRESET (2) Self-custody
 device ∧ (quorum t'-of-n' LampNet)          device (1-of-1)
 custody phân tán THẬT (trục B)              + session key uỷ (KHÔNG controller)
 recovery-không-seed                          bước đệm — CHƯA đạt trục B (nói thẳng)
```

- **INV-DEVICE (device là participant BẮT BUỘC).** Ở mọi cấu hình, **quyền ký phải
  đòi share của device**. Không thoả INV-DEVICE thì một nhóm node LampNet cấu kết
  có thể ký thay user → vi phạm mục tiêu custody.
- **INV-SESSION.** Preset (2): phần "làm việc trên LampNet" dùng **session/operating
  key uỷ quyền, thu hồi được** (mẫu Grant), **KHÔNG** phải controller → lộ session
  key **không** = lộ quyền chi.

> **Lưu ý trung thực:** preset (2) 1-of-1 — device vẫn giữ khoá đủ → **CHƯA** đạt
> trục B. Nó là *bước đệm* ổn định interface FROST-signing. **Chỉ preset (1) mới
> thực sự "không nơi nào giữ khoá đủ".** Không quảng cáo (2) là "phân tán khoá".

---

# PHẦN II — CƠ SỞ KHOA HỌC (dẫn nguồn thật)

## II.1 Mã fountain LT + Robust Soliton (trục A)

- **LT codes** — Luby, M. *"LT Codes."* Proc. 43rd IEEE FOCS, 2002, tr. 271–280.
  DOI [10.1109/SFCS.2002.1181950](https://doi.org/10.1109/SFCS.2002.1181950). Mã
  *rateless*: từ `k` symbol nguồn sinh **không giới hạn** droplet; bộ nhận cần bất
  kỳ `k(1+ε)` droplet là giải được bằng khử Gauss / belief-propagation.
- **Robust Soliton distribution** (phân phối bậc dùng ở §I.2) — do Luby (2002) đề
  xuất; phân tích chặt trong Shokrollahi, A. *"Raptor Codes."* IEEE Trans. Inf.
  Theory 52(6), 2006, DOI [10.1109/TIT.2006.874390](https://doi.org/10.1109/TIT.2006.874390).
  Bảo đảm: với `k` symbol, decode thành công **xác suất ≥ 1−δ** từ
  `k + O(√k · ln²(k/δ))` droplet; overhead trung bình nhỏ.
- **Information Dispersal (IDA)** — Rabin, M. *"Efficient Dispersal of Information
  for Security, Load Balancing, and Fault Tolerance."* JACM 36(2), 1989, tr.
  335–348. DOI [10.1145/62044.62050](https://doi.org/10.1145/62044.62050). Cơ sở lý
  thuyết cho rải `EncSeed` thành mảnh dư thừa: khôi phục từ tập con, chịu mất mát.

## II.2 Chia sẻ bí mật ngưỡng + chữ ký ngưỡng (trục B)

- **Shamir Secret Sharing** — Shamir, A. *"How to Share a Secret."* CACM 22(11),
  1979, tr. 612–613. DOI [10.1145/359168.359176](https://doi.org/10.1145/359168.359176).
  Nền tảng t-of-n; **giới hạn:** có điểm "dựng lại khoá" khi tái tạo → PhoenixKey
  **bỏ** đường reconstruct-then-sign này (Rebirthme I-WALLET-8).
- **FROST** — Komlo, C. & Goldberg, I. *"FROST: Flexible Round-Optimized Schnorr
  Threshold Signatures."* SAC 2020. IACR ePrint [2020/852](https://eprint.iacr.org/2020/852).
  Chuẩn hoá: IRTF CFRG [RFC 9591](https://www.rfc-editor.org/rfc/rfc9591) (2024).
- **Threshold Ed25519 / Schnorr** — chữ ký nhóm là Schnorr trên Ed25519
  (Ed25519: Bernstein et al.; [RFC 8032](https://www.rfc-editor.org/rfc/rfc8032)),
  verify bằng verifier Ed25519 chuẩn — đây là lý do FROST vô hình on-chain.
- **DKG (không bên nào giữ khoá đủ)** — Pedersen, T. *"Non-Interactive and
  Information-Theoretic Secure Verifiable Secret Sharing."* CRYPTO'91, LNCS 576.
  DOI [10.1007/3-540-46766-1_9](https://doi.org/10.1007/3-540-46766-1_9). Gia cố
  chống bias: Gennaro, Jarecki, Krawczyk, Rabin, *J. Cryptology* 20(1), 2007, DOI
  [10.1007/s00145-006-0347-3](https://doi.org/10.1007/s00145-006-0347-3).

## II.3 Dẫn xuất khoá + tách miền

- **HKDF** — Krawczyk & Eronen, [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869).
  Dùng cho `Device_KEK`, `wallet-seed-v1` (`Math.md` §6.1.1), `LocatorSecret`. Nhãn
  `info` khác nhau ⇒ khoá độc lập (tách miền).
- **Tách miền băm accumulator/backup** — mẫu [RFC 6962](https://www.rfc-editor.org/rfc/rfc6962)
  (`0x00` leaf / `0x01` node) để không leaf nào trùng node (chống second-preimage
  cross-tier), khi rải/hash mảnh qua Mirage/Strata.

---

# PHẦN III — ĐÁNH GIÁ ĐỐI KHÁNG (Math + phản biện)

## III.1 Mô hình đe doạ

Ký hiệu: `p` = xác suất một node LampNet **online** tại thời điểm cần; `q` = xác
suất một node **gian/cấu kết** (Byzantine). Đối thủ 𝒜 kiểm soát tập `C` participant.

| # | Đối thủ | Trục bị nhắm | Kết quả mong muốn của 𝒜 |
|---|---|---|---|
| T1 | Node **offline** hàng loạt | A (durability) / B (liveness) | chặn khôi phục / chặn ký |
| T2 | Node **gian** đọc dữ liệu rải | A (bí mật) | học seed từ droplet |
| T3 | Coalition **cấu kết** ≥ ngưỡng | B (custody) | giả mạo chữ ký, chi tài sản |
| T4 | **Máy gốc** bị chiếm | B (custody) | tự ký một mình |
| T5 | **Rogue-key / bias** trong DKG | B | thao túng `PK_group` |
| T6 | **Backdoor / master key** | B | một nơi dựng lại được khoá |

## III.2 T2 — node gian đọc droplet: rò rỉ = 0

**Mệnh đề 1.** Với AEAD an toàn (IND-CCA) và `Device_KEK` không rời phần cứng, một
đối thủ thu **toàn bộ** `n = 1000` droplet của `EncSeed` học được **thông tin
không đáng kể** về `Master_KEK`.

*Lập luận.* Droplet là tổ hợp XOR của các symbol của `EncSeed = Enc(Device_KEK, ·)`.
Thu đủ droplet chỉ khôi phục lại `EncSeed` (bản-mã), không phải plaintext. Không có
`Device_KEK` (đòi `Cloud_Secret` trong Keychain phần cứng **và** `HW_UID`), giải mã
AEAD là bài toán phá IND-CCA → ưu thế negligible. ∎

⟹ **T2 vô hại:** LampNet là kho *bản-mã*, không phải kho *bí mật*.

## III.3 T3/T4 — cấu kết & máy gốc: bất khả nếu thoả INV-DEVICE

### III.3.1 FROST phẳng KHÔNG đủ (điểm nâng cấp so với đề xuất gốc)

Xét preset "phẳng" `t`-of-`(n+1)` với participant `{device, node₁,…,node_n}`. Theo
FROST, **bất kỳ** `t` share nào cũng ký được. Nếu `n ≥ t` thì tập `{node₁,…,node_t}`
— **không có device** — đã đủ ký.

**Mệnh đề 2 (điều kiện cần).** FROST phẳng bảo đảm INV-DEVICE ⟺ `t > n` (số node
không-device < ngưỡng). Với ví dụ "2-of-3 = device + 2 node", ta có `n = 2 ≥ t = 2`
→ **VI PHẠM** INV-DEVICE: hai node cấu kết ký được không cần device.

⟹ Khẳng định của đề xuất gốc ("nhóm node cấu kết thiếu 1 share device → không ký
được") **không thành lập** cho cấu hình 2-of-3 phẳng. **Phải sửa cấu trúc.**

### III.3.2 Cấu trúc đúng — phân tầng 2-of-2 (khuyến nghị)

Đặt chữ ký ở tầng ngoài là **2-of-2** giữa hai "người ký":
```
SIG = FROST_combine₂ₒf₂(  Share_device ,  Sig_LampNet  )
      Sig_LampNet = FROST_signₜ'ₒfₙ'( {node_j} )     -- quorum LampNet t'-of-n'
```
Tức: **device** là một ký-viên **bắt buộc**, và **nhóm LampNet** góp một ký-viên
được hiện thực bằng FROST con `t'`-of-`n'`. Ledger vẫn chỉ thấy **1 `PK_group`, 1
sig** (2-of-2 và FROST-con đều gộp thành một Schnorr).

**Mệnh đề 3 (custody).** Dưới cấu trúc phân tầng 2-of-2, đối thủ 𝒜 tạo được một chữ
ký hợp lệ ⟺ 𝒜 kiểm soát **cả** (i) `Share_device` **và** (ii) ≥ `t'` share trong
`n'` node. Do đó:
- **T4 (máy gốc bị chiếm):** chiếm device có `Share_device` nhưng thiếu quorum
  LampNet (`t' ≥ 1`) → **không** ký một mình được.
- **T3 (LampNet cấu kết toàn bộ):** kiểm soát cả `n'` node nhưng thiếu
  `Share_device` → **không** ký được. *Không thực thể đơn lẻ nào — kể cả toàn bộ
  LampNet — ký thay user.*

*Lập luận.* Unforgeability của FROST (Komlo–Goldberg, định lý an toàn dưới OMDL)
cho: coalition `< t` share **không** sinh chữ ký hợp lệ với ưu thế không-negligible.
Áp cho tầng ngoài (t=2, cần cả 2) + tầng trong (cần `t'` của `n'`). ∎

### III.3.3 Xác suất & liveness (đánh đổi phải nói thẳng)

Giả định độc lập, `p` = P(node online), `d` = P(device khả dụng):
```
P(ký được)  = d · P( ≥ t' trong n' node online )
            = d · Σ_{i=t'}^{n'}  C(n', i) · p^i · (1−p)^{n'−i}
```
- Ví dụ `n'=3, t'=1, p=0.9, d≈1`: `P = 1 − (0.1)³ = 0.999`.
- `n'=3, t'=2`: `P = 3·0.9²·0.1 + 0.9³ = 0.972`.

**Van an toàn bắt buộc:** preset (1) **KHÔNG** được là đường ký duy nhất. Preset (2)
self-custody luôn là fallback (device ký không phụ thuộc LampNet online). LampNet
chết dài hạn → guardian uỷ **Rotate** sang `PK_group` mới (DKG lại). ⟹ Đổi *rủi ro
lộ-seed-mất-hết* lấy *rủi ro kẹt-tiền-tạm-thời*, và có van cho rủi ro sau.

## III.4 T1 — offline: xác suất khôi phục trục A

**Mệnh đề 4 (durability).** Với LT + Robust Soliton (`k=50`, `δ=0.01`), nếu bộ nhận
lấy được `m ≥ k + O(√k·ln²(k/δ))` droplet thì `LT_Decode` thành công xác suất
`≥ 1−δ = 0.99`. Cấu hình `min_count = 60` (biên `k+10`) chọn đúng để vượt ngưỡng
overhead này (khớp I-LAMP-2, `Math.md` §7.2).

⟹ Miễn còn ≥ ~60/1000 droplet sống (node online rải rác), seed **khôi phục được**
w.h.p. Đây là *durability*, độc lập với bí mật (T2) và custody (T3/T4).

## III.5 T5 — rogue-key / bias DKG

- **Rogue-key:** VSS (Feldman/Pedersen) trong DKG ép mỗi share khớp commitment công
  khai → share giả bị phát hiện, loại. RFC 9591 / `frost-ed25519` xử lý sẵn.
- **Bias `PK_group`:** DKG Pedersen thuần có thể bị bias bởi bên gửi cuối; dùng biến
  thể **Gennaro et al. 2007** (commit-then-reveal) loại bias. **Yêu cầu triển khai:**
  DKG PhoenixKey dùng biến thể chống-bias, không Pedersen trần.
- **Transport:** kênh DKG/ký qua LampNet phải chống replay/MITM (bind phiên, nonce
  tươi) — bề mặt tấn công mới, **cần audit riêng** (R-FROST-2).

## III.6 T6 — không backdoor, không master key

**Mệnh đề 5 (no single point).** Dưới INV-DKG + INV-SIGN + cấu trúc 2-of-2 (§III.3.2),
**không tồn tại** một thực thể (máy, node, hay "nhà vận hành") ở vị trí có thể một
mình dựng lại hoặc dùng khoá ký nhóm. Không có khoá "master" giải mã tất cả:
`Device_KEK` per-DID (tách miền HKDF), share DKG per-participant, `PK_group` công
khai. **Single-party-derive bị cấm bằng cấu trúc, không bằng chính sách.**

## III.7 So sánh với giữ-seed-tập-trung

| Trục | Giữ seed tập trung (ví thường) | PhoenixKey trục A+B |
|---|---|---|
| Nơi giữ khoá đủ | 1 (thiết bị / server) — **single point** | **0** (2-of-2: device ∧ quorum) |
| Lộ 1 nơi | mất toàn bộ tài sản | vẫn thiếu ≥1 factor → an toàn |
| Backup | giấy 24 từ (lộ được / mất được) | ciphertext rải LT (durability, T2=0) |
| Custody đa bên | không | có (device bắt buộc + quorum LampNet) |
| Địa chỉ khi đổi custody | thường phải đổi ví | **bất biến** (FROST vô hình on-chain) |
| Điểm dựng-lại-khoá | có (khi khôi phục từ seed) | **không** (DKG + sign-without-reconstruct) |

---

# PHẦN IV — GIẢ ĐỊNH + GIỚI HẠN (không che)

- **A1 — AEAD & HKDF an toàn.** Rò rỉ T2=0 dựa IND-CCA của AEAD + `Device_KEK` bí
  mật (Keychain phần cứng). Nếu `Cloud_Secret` **và** `HW_UID` cùng lộ → trục A hở.
- **A2 — DKG chống bias + VSS đúng.** Mệnh đề 3/5 cần DKG loại bias (Gennaro 2007)
  và VSS kiểm chứng; Pedersen trần **không** đủ.
- **A3 — Độc lập lỗi node.** Công thức liveness (§III.3.3) giả định node độc lập.
  Tương quan (cùng nhà cung cấp hạ tầng, cùng vùng) làm giảm `P(ký được)` thực tế.
- **A4 — Transport an toàn.** Kênh LampNet phải chống replay/MITM (R-FROST-2) —
  **chưa** trong phạm vi doc này, cần audit riêng.
- **L1 — Preset (2) CHƯA đạt trục B.** Device 1-of-1 vẫn single point; chỉ preset
  (1) đạt custody phân tán. Nói thẳng, không quảng cáo nhầm.
- **L2 — Liveness đổi lấy custody.** t/n cao hơn = an toàn hơn nhưng dễ kẹt tiền
  hơn. Đề xuất mặc định `t'`/`n'` **nhỏ** (vd `2-of-3`) + luôn có van self-custody.
- **L3 — Không vá lỗ khác.** Trục A/B **KHÔNG** vá guardian-dup / timelock-floor /
  anti-drain (các lỗ riêng, PR riêng). FROST giảm *xác suất lộ seed* nhưng **không
  thay** anti-drain; các lớp xếp chồng, đừng tạo cảm giác "custody đã an toàn".
- **L4 — DKG/FROST là VIỆC LỚN.** Cần `frost-ed25519` (ZF, audited) vào `rust_core`
  + transport DKG qua LampNet + luồng rotate = re-share/DKG mới. Không giấu chi phí.

---

# PHẦN V — INTERFACE CONTRACT + PHẠM VI (cho đội build)

- **Anchor NFT:** `name = blake2b_256(did)` — **BẤT BIẾN**. `did_payment/stake`
  tiếp tục neo name này.
- **controller_pkh:** `= blake2b_224(PK_group)` (FROST) hoặc `blake2b_224(vk)` (đơn)
  — **cùng field, cùng độ dài, on-chain không phân biệt**. Địa chỉ ví **KHÔNG đổi**.
- **FROST:** off-chain (`rust_core` + transport LampNet). Chain thấy 1 pubkey / 1
  sig. **KHÔNG** đụng validator. Thư viện: ZF `frost-ed25519` (RFC 9591). CẤM tự
  hiện thực.
- **Cấu trúc device-bắt-buộc:** phân tầng **2-of-2** `device ∧ (t'-of-n' LampNet)`
  (§III.3.2), KHÔNG dùng FROST phẳng cho preset (1) trừ khi `t' > n'` (Mệnh đề 2).
- **DKG:** biến thể chống bias (Gennaro 2007), VSS kiểm chứng; khoá đầy đủ **không
  bao giờ** lắp ráp (INV-DKG).
- **Trục A (durability):** giữ nguyên LT-fountain `k=50, n=1000`, Robust Soliton
  `δ=0.01`, `min_count=60` (`Math.md` §7). Rải **ciphertext của từng share** (backup
  durability của share), KHÔNG rải plaintext.
- **Phạm vi PhoenixKey:** FROST trong `rust_core` + transport DKG/ký + luồng
  rotate=re-share + wiring backup share qua Mirage/Strata. **KHÔNG** viết lại
  accumulator/fountain — tái dùng primitive LampNet.
- **Rotate = re-share/DKG mới** → `PK_group'` → `controller_pkh'` → redeemer
  `Rotate` (đã có). Rotate KHÔNG đúc anchor mới, địa chỉ bất biến (I-WALLET-2).

## Phản biện / rủi ro còn lại (🟡)

- **R-1 (liveness):** LampNet dưới `t'` → kẹt tiền. Van = preset (2) + guardian
  fallback rotate. **Phụ thuộc** guardian-trục sống song song (mà guardian-dup còn
  hở — PR `guardian-floor` phải land TRƯỚC khi tin guardian-fallback).
- **R-2 (transport DKG):** bề mặt tấn công mới (replay/MITM/rogue) — cần audit riêng.
- **R-3 (preset 2 chưa đạt trục B):** bước đệm — device vẫn giữ khoá đủ. Không quảng
  cáo là "phân tán khoá".
- **R-4 (tương quan lỗi node):** công thức liveness giả định độc lập; thực tế nên đo
  và chọn node đa nhà cung cấp.

---

## Tham chiếu (nguồn khoa học + mã)

**Khoa học:**
- Luby (2002) LT Codes — DOI [10.1109/SFCS.2002.1181950](https://doi.org/10.1109/SFCS.2002.1181950)
- Shokrollahi (2006) Raptor/Robust Soliton — DOI [10.1109/TIT.2006.874390](https://doi.org/10.1109/TIT.2006.874390)
- Rabin (1989) IDA — DOI [10.1145/62044.62050](https://doi.org/10.1145/62044.62050)
- Shamir (1979) Secret Sharing — DOI [10.1145/359168.359176](https://doi.org/10.1145/359168.359176)
- Komlo & Goldberg (2020) FROST — IACR [2020/852](https://eprint.iacr.org/2020/852)
- IRTF CFRG (2024) FROST — [RFC 9591](https://www.rfc-editor.org/rfc/rfc9591)
- Pedersen (1991) VSS/DKG — DOI [10.1007/3-540-46766-1_9](https://doi.org/10.1007/3-540-46766-1_9)
- Gennaro et al. (2007) DKG chống bias — DOI [10.1007/s00145-006-0347-3](https://doi.org/10.1007/s00145-006-0347-3)
- HKDF — [RFC 5869](https://www.rfc-editor.org/rfc/rfc5869) · Ed25519 — [RFC 8032](https://www.rfc-editor.org/rfc/rfc8032) · Merkle domain-sep — [RFC 6962](https://www.rfc-editor.org/rfc/rfc6962)

**Chuẩn Cardano:** [CIP-1852](https://cips.cardano.org/cip/CIP-1852) · [CIP-3](https://cips.cardano.org/cip/CIP-3)

**Thư viện:** [`frost-ed25519` (ZF)](https://github.com/ZcashFoundation/frost) · `ed25519-dalek` · [`hkdf`](https://crates.io/crates/hkdf) · [`cardano-serialization-lib`](https://crates.io/crates/cardano-serialization-lib) · [`bip39`](https://crates.io/crates/bip39) · [`blake2`](https://crates.io/crates/blake2)

**Mã PhoenixKey:** `rust_core/src/{cardano.rs,sign.rs}` · `derive-demo/src/main.rs` · `lampnet.rs → /mirage/put` (LT-fountain) · Strata (MMR phả hệ share/seed) · `Math.md` §6/§6.1.1/§7/§8.1.
