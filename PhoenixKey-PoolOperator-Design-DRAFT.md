# PhoenixKey — Tab Pool (vận hành stake pool trên điện thoại) — Thiết kế DRAFT

> **Loại:** Design / Math (Tech-leaning). **Ngày:** 2026-07-14. **Trạng thái:** DRAFT chờ anh + Long + Tuân duyệt.
> **Nguồn Feat:** [PhoenixKey-Wallet-Staking-Governance-Vi-Feat.md](./PhoenixKey-Wallet-Staking-Governance-Vi-Feat.md) §2.3 (điểm `[CẦN CHỐT]`).
> **Dùng cho:** SRCL đợt 1 — pool GST do GreenSun vận hành ([LAMP/SPEC/SRCL-Spec-Vi.md](../../LAMP/SPEC/SRCL-Spec-Vi.md) §5).
> Tài liệu này CHỐT ranh giới bảo mật đã bỏ ngỏ ở bản Feat, và định nghĩa phần rust_core + UI cần xây.

---

## 0. Vì sao tài liệu này tồn tại

Feat §2.3 để lại một điểm `[CẦN CHỐT]`: cho phép vận hành pool trên mobile thay cho máy air-gap riêng, **với điều kiện** bù lại bằng lớp bảo vệ tương đương. Đây là money-critical bậc cao nhất trong toàn hệ PhoenixKey: **lộ khoá cold của pool ảnh hưởng TOÀN BỘ người uỷ thác vào pool đó**, không chỉ chủ pool. Tài liệu này chốt chính xác:

1. Ranh giới **cái gì chạy trên điện thoại** vs **cái gì bắt buộc ở node/server** (không mơ hồ).
2. **Mô hình bảo mật cụ thể** thay air-gap (4 lớp, đo được).
3. **Bất biến toán học** của mỗi thao tác.
4. **Giao diện rust_core + FFI** đã xây (xem §7 — mức hoàn thành).

---

## 1. Nguyên lý gốc — tách "khoá cần bảo vệ" khỏi "khoá phải online"

Vận hành một stake pool Cardano dùng **ba** loại khoá, bản chất khác nhau:

| Khoá | Bản chất | Phải sống ở đâu | PhoenixKey |
|---|---|---|---|
| **Cold key** (operator/pool key) | Ed25519. Định danh pool (pool id = BLAKE2b-224 của cold vkey). Ký cert đăng ký/cập nhật/rút lui + ký OpCert. Dùng **hiếm** (vài lần/năm). | **Offline / được bảo vệ tối đa.** Đây là khoá "air-gap" theo thông lệ ngành. | **✅ Giữ trong Secure Enclave của điện thoại** (như Master_KEK). |
| **KES key** (hot / operational) | Sum-composition Ed25519 (khoá tiến hoá — KES). Ký **từng block** mỗi vài giây. Tự huỷ khả năng ký quá khứ khi tiến hoá. | **Online, ngay trên block-producer node** — vì nó ký block liên tục. Bản chất là khoá "nóng", được thiết kế để rotate chính vì nó phơi nhiễm. | **KHÔNG ở điện thoại.** Node sinh + giữ. Điện thoại chỉ nhận **KES *verification key*** (công khai) để ký OpCert. |
| **VRF key** | VRF (khoá xổ số quyền sản xuất block). Node đọc mỗi slot. | **Online, trên node.** | **KHÔNG ở điện thoại.** Điện thoại chỉ nhận **VRF keyhash** (công khai) để đưa vào cert đăng ký. |

**Kết luận nguyên lý:** điện thoại giữ **đúng một** khoá — cái **duy nhất** hưởng lợi từ bảo vệ kiểu air-gap (cold key). Mọi khoá phải-online (KES, VRF) **không** lên điện thoại. Đây là câu chuyện bảo mật **mạnh hơn** "nhét tất cả lên điện thoại": ta không bao giờ đưa một khoá-nóng vào Enclave rồi lại phải chuyển nó ra node (vô nghĩa và kém an toàn).

> **Ranh giới tuyệt đối:** điện thoại **KHÔNG chạy block-producer node**. Sản xuất block đòi hạ tầng server chạy 24/7, đồng bộ chain, VRF/KES nóng — không thể và không nên trên điện thoại. Điện thoại **chỉ ký certificate + phát hành OpCert**. Vai trò của điện thoại = **thiết bị ký cold key** (thay chiếc máy air-gap), không phải node.

---

## 2. Ba thao tác điện thoại ký (và một thao tác nó KHÔNG làm)

### 2.1 Pool Registration / Update certificate — LÀM trên điện thoại
Cùng **một** loại certificate (`pool_registration`) dùng cho cả **tạo mới** và **cập nhật** (margin/cost/pledge/relays/metadata). Ledger phân biệt bằng việc pool đã tồn tại hay chưa. Trường (theo cardano-ledger `pool_params`):

| Trường | Kiểu | Nguồn |
|---|---|---|
| `operator` | Ed25519KeyHash (28B) = pool id | BLAKE2b-224(cold vkey), điện thoại tự dẫn |
| `vrf_keyhash` | 32B | từ **node** (`cardano-cli node key-gen-VRF` → `vrf.vkey` → hash) |
| `pledge` | lovelace | chủ pool nhập |
| `cost` | lovelace (≥ `minPoolCost`) | chủ pool nhập |
| `margin` | UnitInterval = num/den ∈ [0,1] | chủ pool nhập |
| `reward_account` | RewardAddress | stake key của chủ pool (điện thoại dẫn) |
| `pool_owners` | tập Ed25519KeyHash (stake keyhash) | stake key của chủ pool (điện thoại dẫn) |
| `relays` | danh sách | chủ pool nhập (IP/DNS của relay node) |
| `pool_metadata` | optional {url ≤64B, hash 32B} | chủ pool nhập (BLAKE2b-256 của JSON metadata) |

Relay có 3 biến thể (CSL hỗ trợ đủ): `single_host_addr`(ipv4/ipv6 + port), `single_host_name`(DNS A/AAAA + port), `multi_host_name`(DNS SRV).

**Deposit:** đăng ký **mới** khoá 500 ADA (`poolDeposit`) do ví chủ pool trả; **cập nhật** KHÔNG khoá deposit. (Xem §6 — CSL tính deposit cho MỌI cert `pool_registration`, nên khi cập nhật ta phải truyền `pool_deposit=0`.)

**Chữ ký cần:** ví chi phí (payment) + **cold key** (operator) + mọi stake key trong `pool_owners` + stake key của `reward_account` (nếu chưa nằm trong owners).

### 2.2 Pool Retirement certificate — LÀM trên điện thoại (thao tác nhạy, có timelock)
Trường: `pool_keyhash` (pool id) + `epoch` (epoch rút lui). Ràng buộc: `e_hiện_tại < e_rút_lui ≤ e_hiện_tại + eMax` (protocol param `eMax`). Khoá deposit 500 ADA được hoàn khi retirement có hiệu lực. Chữ ký: payment + cold key.

### 2.3 Phát hành / xoay OpCert (Operational Certificate) — LÀM trên điện thoại
OpCert = giấy uỷ quyền cold key cấp cho KES vkey hiện hành, để node được quyền sản xuất block. **4 thành phần** (cardano-ledger `OCert`):

```
OpCert = ( hot_vkey   : KES verification key, 32B   ← từ node
           counter    : u64  (số thứ tự, tăng dần)  ← điện thoại giữ + tăng
           kes_period : u64  (KES period bắt đầu)   ← tính từ slot hiện tại
           sigma      : Ed25519 signature           ← COLD KEY ký )
```

**sigma** = `Ed25519.Sign( cold_sk , hot_vkey(32B) ‖ u64_big_endian(counter) ‖ u64_big_endian(kes_period) )` — thông điệp **48 byte**. (Nguồn: cardano-ledger `Cardano.Protocol.TPraos.OCert.getSignableRepresentation` — xác nhận §Nguồn.)

**OpCert KHÔNG phải certificate trong transaction body.** Khác hẳn pool_registration/retirement (nằm trong tx, submit lên chain). OpCert nằm trong **block header**; nó là file node nạp để bắt đầu ký block. → Phát hành/xoay OpCert **KHÔNG cần giao dịch on-chain, KHÔNG tốn phí** — chỉ là: điện thoại ký ra `node.opcert` (CBOR), chuyển sang node.

**Xoay KES (KES rotation):** khi KES period sắp hết hạn (mỗi KES key chỉ ký được `MaxKESEvolutions` period), chủ pool:
1. Trên **node**: `cardano-cli node key-gen-KES` → KES keypair mới → lấy `kes.vkey` (công khai).
2. Trên **điện thoại**: nhập/quét `kes.vkey` mới + `kes_period` hiện tại → Enclave mở cold key → ký OpCert mới với `counter = counter_cũ + 1`.
3. Chuyển `node.opcert` mới sang node, khởi động lại block-producer.

> KES secret **không bao giờ** lên điện thoại — chỉ KES **vkey** (công khai). "Xoay KES" từ góc điện thoại = **phát hành OpCert mới (counter+1)** cho KES vkey mới. Đây là toàn bộ vai trò cold key trong việc xoay.

### 2.4 Cái điện thoại KHÔNG làm
Sinh KES/VRF **secret**, chạy node, ký block, giữ chain state. Tất cả ở node. (rust_core **không** hiện thực sinh khoá KES — xem §7, §9 điểm chốt.)

---

## 3. Mô hình bảo mật thay air-gap — 4 lớp (đo được, không mơ hồ)

Bỏ air-gap vật lý ⇒ phải bù đủ 4 lớp dưới đây. Thiếu **bất kỳ** lớp nào thì KHÔNG được bật tab Pool cho pool thật.

### Lớp A — Cold key non-extractable trong Secure Enclave
- Cold key Ed25519 **không tồn tại ở dạng plaintext khi nghỉ**. Nó được dẫn trong RAM **chỉ** trong lúc ký, từ seed/Master_KEK mà việc giải mã bị **Secure Enclave gate bằng sinh trắc** (như Master_KEK hiện có). Ký xong → `Zeroize` (đã có sẵn trong Cargo.toml).
- Cold key **không bao giờ** rời FFI dưới dạng output (bất biến INV-1, giống seed trong `staking.rs`).
- Đường dẫn dẫn khoá: **CIP-1853** `m/1853'/1815'/0'/{pool_index}'` — chuẩn HD cho pool cold key, cô lập mật mã khỏi payment (`1852'/…/0/0`) và stake (`1852'/…/2/0`).
  - **[CẦN CHỐT — điểm 1]** Với pool giá trị cao (GST): nên cân nhắc cold key **KHÔNG** dẫn từ seed ví (để lộ seed ví ≠ lộ pool), mà là **khoá Enclave độc lập + ngưỡng guardian**. Xem §9.

### Lớp B — Đa yếu tố cho MỌI thao tác pool (khác chữ ký ví thường)
- Ví thường: 1 sinh trắc → ký. **Pool: sinh trắc + yếu tố thứ hai riêng cho pool.** Yếu tố thứ hai = **một trong**: (a) passphrase/PIN riêng cho pool-key (khác PIN ví); (b) **ngưỡng guardian** (k-trong-n người đồng ý — dùng mô hình guardian đã có, xem `guardian-recovery-model`).
- Ràng buộc thiết kế (theo memo `curve-routing-asymmetry`): yếu tố thứ hai **phải khác gốc seed** — nếu không, nó không thêm entropy độc lập. → passphrase pool KHÔNG dẫn từ seed; guardian là khoá của người khác.

### Lớp C — Jailbreak/root ⇒ CHẶN CỨNG (không cảnh báo cho qua)
- Ví thường: phát hiện jailbreak → cảnh báo, người dùng bấm "vẫn tiếp tục". **Pool: phát hiện jailbreak/root → chặn cứng**, tab Pool **từ chối** build/ký. Không có nút "vẫn tiếp tục".
- Kiểm tra thực hiện ở tầng Flutter/native TRƯỚC khi gọi FFI (jailbreak detection đã có trong nhánh `mobile-screen-jailbreak`). rust_core nhận cờ `device_integrity_ok`; nếu false → trả rỗng (không ký).

### Lớp D — Timelock + đường huỷ khẩn cho thao tác nhạy
- **Thao tác nhạy** = retirement, đổi `reward_account`, đổi `pool_owners`. Các thao tác này đi qua **cửa sổ trễ bắt buộc** (ví dụ 24–48h): điện thoại tạo tx nhưng **giữ chưa submit**; trong cửa sổ, nếu nghi lộ khoá → **huỷ** (không submit) + **xoay KES khẩn** + (nếu cần) xoay cold key.
- Thao tác thường (cập nhật margin/cost/relays, phát hành OpCert định kỳ) KHÔNG bắt buộc timelock.
- Timelock enforce ở tầng ứng dụng (giữ tx chưa submit) — KHÔNG phải on-chain script (pool cert không hỗ trợ điều kiện script). Đây là hàng rào quy trình, ghi rõ để không nhầm là bảo đảm mật mã.

---

## 4. Bất biến (Math / invariant)

- **INV-1 (cô lập cold key):** khoá cold private không bao giờ là output của FFI; chỉ tồn tại tạm trong RAM rust_core khi ký, zeroize khi drop.
- **INV-2 (counter đơn điệu):** OpCert phát hành lần n+1 có `counter = counter_n + 1`, tăng nghiêm ngặt. Ledger từ chối OpCert có `counter ≤` counter đã thấy trên chain. ⇒ điện thoại PHẢI bền hoá counter; mất counter = phải truy vấn counter cuối trên chain trước khi phát hành tiếp.
- **INV-3 (ràng buộc OpCert):** `sigma = Ed25519.Sign(cold_sk, hot_vkey ‖ u64be(counter) ‖ u64be(kes_period))`, và `Ed25519.Verify(cold_vk, msg, sigma) = true`. Node kiểm `BLAKE2b224(cold_vk) = operator` (pool id) trong cert đăng ký.
- **INV-4 (an toàn vốn):** thao tác pool chỉ chạm cold key + phí (từ ví chủ pool) + deposit 500 ADA (từ ví chủ pool, hoàn khi retire). **KHÔNG bao giờ** chạm vốn người uỷ thác — uỷ thác stake ≠ custody; ADA của delegator nằm nguyên trong ví họ.
- **INV-5 (miền tham số):** `0 ≤ margin = num/den ≤ 1`; `cost ≥ minPoolCost`; `pledge ≥ 0`; `url ≤ 64 byte`; `metadata_hash = BLAKE2b256(metadata_json)`.
- **INV-6 (miền epoch retirement):** `e_hiện_tại < e_rút_lui ≤ e_hiện_tại + eMax`.
- **INV-7 (KES period):** `kes_period = ⌊slot / K⌋`, `K = slotsPerKESPeriod` (129600 ở preprod/mainnet, lấy từ shelley genesis). OpCert hợp lệ cho các KES evolution trong `[kes_period, kes_period + MaxKESEvolutions)`; PHẢI xoay trước khi hết khoảng.

---

## 5. Luồng đầu-cuối (PreProd)

```
NODE (server GreenSun, online)              ĐIỆN THOẠI (PhoenixKey, Enclave)
────────────────────────────────           ─────────────────────────────────
cardano-cli node key-gen-VRF   →  vrf.vkey ─┐
                                            ├─→ nhập vrf_keyhash + tham số pool
                                            │   → Enclave mở cold key
                                            │   → build + ký pool_registration tx
                                            │   → submit qua Blockfrost (preprod)
cardano-cli node key-gen-KES   →  kes.vkey ─┤
                                            ├─→ nhập kes_vkey + kes_period + counter
                                            │   → Enclave mở cold key
                                            │   → ký OpCert (KHÔNG lên chain)
node.opcert ←───────────────────────────────┘   → chuyển file OpCert sang node
khởi động block-producer với node.opcert
```

Đọc trạng thái pool: điện thoại gọi Blockfrost `/pools/{pool_id}` (preprod) hiển thị pledge/margin/cost/trạng thái/relays.

---

## 6. Đánh đổi kỹ thuật đã chốt

- **Cùng một hàm cho create + update pool** (`build_pool_registration_tx`, cờ `is_update`). Khi `is_update=true` → truyền `pool_deposit=0` cho tx builder (CSL tính deposit cho MỌI cert `pool_registration` — `certificates_builder.rs:180`; nếu không zero, update sẽ dự trữ nhầm 500 ADA).
- **Owners + reward_account = khoá của chính chủ pool** (theo account index, dẫn từ seed) ở v1. Pool nhiều chủ (pledge đa bên, owner ngoài) cần lắp ráp witness ngoài — hoãn sang bản sau (ghi rõ ở §9).
- **kes_period nhận trực tiếp** làm tham số (caller tính từ Blockfrost tip + `slotsPerKESPeriod`) — giữ rust_core thuần, không phụ thuộc thời gian.
- **OpCert trả CBOR hex**; đóng gói file `.opcert` do tầng app/node lo.

---

## 7. Mức hoàn thành (2026-07-14)

- ✅ rust_core `pool.rs` (`Enclave/rust_core/src/pool.rs`): `build_pool_registration_tx` (create+update), `build_pool_retirement_tx`, `issue_op_cert` (phát hành/xoay KES), `pool_id_bech32`. **13 test PASS** + 149 test toàn crate PASS (không hồi quy).
- ✅ FFI exports (`lib.rs`): `taad_build_pool_registration_tx`, `taad_build_pool_retirement_tx`, `taad_issue_op_cert`, `taad_pool_id_bech32`.
- ✅ **OpCert 48-byte signable đã đối chiếu VERBATIM** với cardano-ledger `OCert.hs` (`getSignableRepresentation = rawSerialiseVerKeyKES vk <> word64BE counter <> word64BE kesPeriod`, size = verKeySize+8+8 = 48B cho Sum6Kes). Test `opcert_sigma_verifies_over_signable` + `opcert_signable_layout_exact` chứng minh byte-exact.
- ⛔ KES/VRF secret gen: **KHÔNG hiện thực trên điện thoại** (thuộc node) — theo nguyên lý §1 (cố ý).
- ⏳ **CHƯA làm — pha kế tiếp:** Dart FFI binding (`enclave_bridge.dart`) + Flutter UI tab Pool + **wiring 4 lớp bảo mật §3** (Enclave gate / đa yếu tố / chặn jailbreak / timelock). Phần này PHẢI tích hợp với các hệ đang dở ở nhánh khác (`mobile-screen-jailbreak`, enclave gating, guardian) — không làm rời được. rust_core cố tình KHÔNG tự enforce lớp bọc bảo mật; nếu bind Dart mà chưa có 4 lớp = hở nguy hiểm.

---

## 8. Nguồn (đã xác minh)

- OpCert 4 thành phần + thông điệp ký 48B: cardano-ledger `libs/cardano-protocol/src/Cardano/Protocol/TPraos/OCert.hs` — đã đọc VERBATIM: `getSignableRepresentation (OCertSignable vk counter period) = runByteBuilder (verKeySizeKES + 8 + 8) $ byteStringCopy (rawSerialiseVerKeyKES vk) <> word64BE counter <> word64BE (unKESPeriod period)`. Khớp byte-exact với `pool::opcert_signable`.
- pool_params (9 trường): cardano-ledger Shelley/Conway CDDL `pool_params`.
- CSL 13.2.1 API: `PoolParams::new(operator, vrf_keyhash, pledge, cost, margin, reward_account, pool_owners, relays, pool_metadata)`, `OperationalCert::new(hot_vkey, sequence_number, kes_period, sigma)`, `PoolRetirement::new(pool_keyhash, epoch)`, `Relay::new_single_host_addr/name/multi_host_name` — xác minh trực tiếp trong source crate đã vendored.
- CIP-1853 (pool cold key HD path `1853'/1815'/0'/i'`); CIP-1852 (payment/stake path).
- SRCL pool GST: `LAMP/SPEC/SRCL-Spec-Vi.md` §5.

## 9. CẦN ANH + LONG + TUÂN CHỐT

1. **Cold key: dẫn-từ-seed (CIP-1853) hay khoá Enclave độc lập + guardian?** — v1 dùng CIP-1853 (chuẩn, test được). Với GST giá trị cao, khoá độc lập + ngưỡng guardian an toàn hơn (lộ seed ví ≠ lộ pool). Chốt trước khi dùng thật.
2. **Yếu tố thứ hai:** passphrase-pool hay ngưỡng guardian? (Lớp B.)
3. **Cửa sổ timelock** cho retirement/đổi reward_account: 24h? 48h? (Lớp D.)
4. **Pool đa chủ** (pledge nhiều bên): có cần bản sau không, hay v1 single-owner đủ cho GST?
5. **Sinh KES/VRF ở node** (không trên điện thoại) — xác nhận đúng ý đồ vận hành GreenSun.

---
_DRAFT — Phoenix agent soạn cho anh Aladin + Long + Tuân duyệt. Chưa commit/push._
