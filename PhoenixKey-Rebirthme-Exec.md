# PhoenixKey — Rebirthme — Điều hành (Exec)

> **Module:** Rebirthme (slug `Rebirthme`). **Loại doc:** Điều-hành cho lãnh-đạo. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** maintainer + lãnh-đạo. Quyết-định, rủi-ro, lộ-trình. Chi-tiết ở [-Vi-Feat](./PhoenixKey-Rebirthme-Vi-Feat.md) / [-Math](./PhoenixKey-Rebirthme-Math.md) / [-Tech](./PhoenixKey-Rebirthme-Tech.md).
> → Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)

---

## 1. Tóm tắt một trang

Module **Rebirthme** là trục "giữ tiền" của PhoenixKey: ví đi theo DANH TÍNH (không theo chìa), khôi-phục khi mất máy/seed, chống rút-sạch khi lộ khoá, backup tự-động mọi bí-mật. Đây là điểm bán khác-biệt lớn nhất so với ví seed-phrase / ví cứng: **mất seed không còn là mất trắng, và lộ khoá không còn là mất sạch tức-thì.**

**Nền-tảng:** Ví Phượng-hoàng (`did_payment`) chi khi DID Active + controller hiện-tại ký; **tài-sản sống qua rotate/recovery** (địa-chỉ bất-biến, đọc controller động qua anchor ref-input); đóng-băng theo trạng-thái (Recovering/Migrated/Revoked chặn mọi chi); guardian-recovery theo ngưỡng + timelock, đã BỎ mô-hình giữ-mảnh (Shamir), chỉ uỷ-quyền.

**Luật thiết-kế cốt-tử:** lớp **anti-drain** (trần rút + đóng-băng chủ-động) là **LOAD-BEARING** cho mọi DID giữ giá-trị đáng-kể — không phải tính-năng trang-trí. Vì khoá phần-cứng (P-256 Secure Enclave) không verify được trên chuỗi, quyền chi value quy về khoá gốc (SEED, Ed25519 controller) — anti-drain là lớp cản DUY-NHẤT giữa lộ seed và mất sạch, nên là hạng-mục ưu-tiên số 1 (xem `PhoenixKey-Rebirthme-Math.md` I-CURVE-4). **Địa-chỉ riêng-tư L3** dùng chung validator `did_subaddr` với Easteregg. (**Smartsend** nay là module độc-lập thứ 8 — xem `PhoenixKey-Smartsend-Exec.md`; nó tái-dùng anti-drain/guardian của module này.)

---

## 2. Bảng quyết-định

| Quyết-định | Lý-do (4 trục: a dài-hạn / b first-principles / c tối-ưu / d user) | Đánh-đổi |
|---|---|---|
| **Ví theo DANH TÍNH (`did_payment`), địa-chỉ bất-biến** | (a) nền cho SDK mở mọi team Cardano; (b) tài-sản thuộc danh-tính không thuộc khoá; (c) một script logic cho mọi loại DID (không validator riêng Person/Org); (d) mất máy khôi-phục xong tiêu tiếp | UTxO là script (ExUnit cao hơn ví khoá thuần) — bù bằng reference-script |
| **Guardian = uỷ-quyền, BỎ Shamir-shard** | (a) mô-hình OrgDID nhất-quán; (b) guardian không nên giữ bí-mật của ai; (c) đơn-giản hơn quản-mảnh; (d) không ai từng nắm bí-mật gốc của user | Cần backup tự-động (LampNet) làm trục durability độc-lập |
| **Không bắt xuất seed; export = rotate-before-reveal** | (a) hạ rào onboarding; (b) hé-lộ seed ≠ hé-lộ chìa tài-sản; (c) tái dùng builder rotate/migrate sẵn; (d) user phổ-thông không mất tiền vì vô-tư export | Cần UI cắm mặc-định (chưa xong) |
| **Anti-drain là bắt-buộc cho DID giá-trị (I-CURVE-4)** | (a) custody tổ-chức đòi circuit-breaker; (b) value bảo-vệ ở mức seed → cần lớp cản độc-lập; (c) opt-in, không phá bất-biến cũ; (d) lộ khoá mất tối-đa một cửa-sổ | Bắt-buộc, ưu-tiên cao |
| **Second-factor rút-lớn KHÁC gốc seed (I-CURVE-5)** | (b) cùng gốc seed = một điểm-hỏng; (d) "các cấp độ" phải thật | Cần phần-cứng/kênh độc-lập; builder chưa enforce |
| **L3 unlinkable qua `did_subaddr` (chờ chốt)** | (b) OrgDID-chi-tất-cả vs không-lộ-chung-chủ kéo ngược nhau; (c) một anchor chung, N script hash | Dependency onchain MỚI; lộ `tag_i` khi CHI |

---

## 3. Ma-trận rủi-ro (luật giảm-thiểu)

| Rủi-ro | Mức | Giảm-thiểu (luật thiết-kế) |
|---|---|---|
| **Seed lộ → rút sạch** | 🔴 CAO | Anti-drain (`limit_meter.ak`) là hàng-rào bắt-buộc số 1; không để tiền lớn tập-trung một địa-chỉ khi chưa bật; đóng-băng theo status luôn chạy khi vào Recovering |
| **Guardian thông-đồng** | 🟡 | timelock + Cancel trong cửa-sổ + vai veto "còn-sống" + cap (không guardian nào ≥ threshold, ngưỡng ≥2) |
| **Second-factor cùng gốc seed** | 🟡 | I-CURVE-5 buộc khác gốc — enforce ở builder |
| **Malleability chữ-ký / tái-dùng nonce P-256** | ⚪ THẤP | low-s ép tại verify (chống malleability); giám-sát `r` lặp server-side (I-SIGN-NO-REUSE); tách miền khoá giới-hạn thiệt-hại xuống "giả auth", không tới tài-sản |
| **Durability backup** | 🟡 | ciphertext đã mã-hoá trước khi rời máy; repair dựa Mirage `EncryptedDistributed` (LampNet) |
| **Phụ-thuộc ngoài (CARP policy, stake-state, Merkle LAMP, KES crate)** | ⚪ | phối team tương-ứng; không chặn phần lõi ví |

---

## 4. Lộ-trình mốc

- **M1:** mở resolver `service[]` L1/L2 + chặn cứng L3; wallet API v2 (Standard, `/all`, deprecate MAGIC); kho bí-mật blob-đơn.
- **M2 (ưu-tiên số 1):** `limit_meter.ak` + sửa `did_payment` nhánh binding opt-in → đóng hở anti-drain. Enforce I-CURVE-5 ở builder.
- **M3:** `did_stake.ak` (stake theo-DID + đa-ISPO); export re-key cắm mặc-định UI.
- **M4:** phả-hệ seed + Strata. (Smartsend `smartsend_escrow.ak` nay ở module riêng — lộ-trình ở `PhoenixKey-Smartsend-Exec.md §5`; phụ-thuộc anti-drain land.)
- **M5:** `did_subaddr.ak` (L3 unlinkable — sau khi maintainer chốt); registry-lib mode-2 (Org m-of-n); pool KES (sau PoC crate).

---

## 5. Câu hỏi cần LÃNH-ĐẠO chốt

1. **`did_subaddr.ak` (L3 unlinkable)** — dependency onchain MỚI [DEP-2]. Chốt build hay hoãn? (MVP tạm "riêng-tư mức-resolver" chạy được ngay.)
2. **Meter-NFT policy** — dùng `taad` Design-2 hay policy meter chuyên-dụng? [CẦN CHỐT-W1]
3. **`vault_index_anchor` + `recovery_anchor` + `pool_anchor` vào `TAADDatum`** — thay-đổi schema thuộc Specs/Validator (đội backend). Duyệt? [CẦN CHỐT-W3]
4. **Vị-trí module Smartsend** — Smartsend là module độc-lập thứ 8 (tách khỏi Rebirthme). Nó tái-dùng hạ-tầng của module này (guardian-factor, anti-drain, `did_payment`) — dependency khai rõ ở `PhoenixKey-Smartsend-{Tech,Math}.md`, KHÔNG nhân-bản. Quyết-định + rủi-ro + lộ-trình + 5 vá đỏ ở `PhoenixKey-Smartsend-Exec.md`.
5. **Chính-sách tiền lớn khi anti-drain chưa land** — có khuyến-cáo hạn-mức tiền/địa-chỉ tạm-thời không, hay chấp-nhận rủi-ro tới M2?
6. **Crate KES/VRF Rust** — cấp nguồn cho PoC sớm (rủi-ro chính của pool-ops)?

---

## Nguồn

Chi-tiết: [PhoenixKey-Rebirthme-Vi-Feat.md](./PhoenixKey-Rebirthme-Vi-Feat.md), [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md), [PhoenixKey-Rebirthme-Tech.md](./PhoenixKey-Rebirthme-Tech.md). Code: `PhoenixKey-Validator/validators/did_payment.ak`, `lib/phoenixkey/{auth_logic,taad_logic}.ak`.
→ Trạng-thái & tiến-độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#rebirthme)
