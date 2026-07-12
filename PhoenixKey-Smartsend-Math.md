# PhoenixKey — Smartsend — Toán hình thức (Math)

> **Module:** Smartsend (gửi có bảo vệ). **Loại doc:** Toán hình thức cho auditor. **Ngày:** 2026-07-09.
> **Đối tượng đọc:** auditor / kiểm toán mật mã. Định nghĩa hình thức, bất biến, mệnh đề ép từng thao tác, đối chiếu dòng code thật (khi land).
>
> **Ranh giới (MECE):** module này đặc tả **hòm ký quỹ gửi có bảo vệ** (`smartsend_escrow`): escrow + cửa sổ-veto + huỷ đa yếu tố + đồng-ý người nhận + hoàn tất byte-perfect + Freeze + ReclaimTimeout. Cơ chế **ví theo-DID** (`did_payment`), **guardian-recovery**, **anti-drain** (`limit_meter`) **thuộc module Rebirthme** — ở đây CHỈ dẫn chiếu, không định nghĩa lại. Xem [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) (§2.1 cổng chi, §4.B anti-drain, §4.I guardian) + [PhoenixKey-Math.md](./PhoenixKey-Math.md) §10/§11.
>
> **Mã bất biến:** bất biến Smartsend giữ NGUYÊN tiền tố `SS-*` / `SSR-*` (đã độc lập từ nguồn). Bất biến tái dùng từ Rebirthme ghi rõ mã gốc + nguồn (I-WALLET-*, I-LIMIT-*, I-GUARD-*, I-CURVE-*). Bất biến từ `PhoenixKey-Math.md` ghi rõ §.

---

## 1. Ký hiệu (notation)

| Ký hiệu | Nghĩa |
|---|---|
| `did` | định danh phi tập trung, chuỗi `did:phoenix:...` |
| `anchor_policy`, `anchor_name(did)` | policy + asset-name anchor NFT (dẫn chiếu Rebirthme §1) |
| `escrow_addr` | địa chỉ script `smartsend_escrow` |
| `sender_commit`, `receiver_commit` | commitment danh tính người gửi / người nhận (neo controller qua anchor) |
| `amount` | số tài sản khoá trong hòm (đơn vị asset đã chỉ định) |
| `open_slot` | lower-bound hữu hạn của tx Open (mốc mở hòm) |
| `veto_deadline` | mốc hết cửa sổ-veto = `open_slot + window` |
| `reclaim_deadline` | mốc quá hạn cho ReclaimTimeout (SSR-7) |
| `freeze_deadline` | mốc quá hạn thoát-Freeze (SSR-6) — sau mốc này không resolve → auto-hoàn sender |
| `window` | độ dài cửa sổ-veto, `∈ {24,48,72}h` hoặc 2-bên thoả, ép sàn `min_window` (SSR-10) |
| `large_threshold` | ngưỡng "khoản lớn" đòi `receiver_consent` |
| `unlock_policy` | chính sách huỷ: `factors_required` + tập factor + verifier |
| `min_ada` | lovelace ký quỹ tối thiểu ghi lúc Open, tách khỏi `amount` (SSR-2) |
| `now` | thời điểm tx đọc từ validity-interval (`lo`/`hi` tuỳ chiều) |
| `W(tx)` | net giá trị rời một địa chỉ (dẫn chiếu Rebirthme §2.3) |

---

## 2. Định nghĩa hình thức các hàm/tập

### 2.1 Cổng chi nguồn (dẫn chiếu — KHÔNG định nghĩa lại)
Nguồn nạp vào hòm đến từ Ví Phượng hoàng: chi qua `did_payment` mode-1 ⟺ `anchor_controller_ok` (DID Active + controller hiện tại ký). Định nghĩa đầy đủ: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) §2.1, `auth_logic.ak:37-52`, `did_payment.ak:40-50`. Module này giả định cổng đó đúng và chỉ đặc tả logic hòm.

### 2.2 Ba đường thoát một escrow-UTxO
```
SmartSendSpend_ok(tx, d, now) ⟺
    Cancel_ok(tx, d, now)            -- về sender
  ∨ Finalize_ok(tx, d, now)         -- về receiver
  ∨ Freeze_ok(tx, d)                -- treo (guardian)
  ∨ ReclaimTimeout_ok(tx, d, now)   -- về sender (chống kẹt)

Cancel_ok(tx,d,now)   ⟺ now < d.veto_deadline
                       ∧ satisfies(d.unlock_policy, tx)             -- SS-3, ≥ factors_required
                       ∧ Σ outputs(→sender, d.asset) == d.amount ∧ Σ outputs(→sender, lovelace) == d.min_ada  -- SS-1/SS-12 (byte-perfect, min_ada tách riêng)
Finalize_ok(tx,d,now) ⟺ now ≥ d.veto_deadline
                       ∧ (d.amount < d.large_threshold ∨ d.receiver_consent == true)  -- SS-4
                       ∧ Σ outputs(→receiver, d.asset) == d.amount ∧ Σ outputs(→receiver, lovelace) == d.min_ada  -- SS-5′/SS-12 (byte-perfect, không đổi hướng, min_ada đi cùng receiver)
Freeze_ok(tx,d,now)   ⟺ now < d.veto_deadline                       -- SS-8 (Freeze chỉ trong cửa-sổ, sau đó vô-nghĩa)
                       ∧ guardian_sig hợp-lệ (neo anchor) ∧ escrow → treo (frozen := true)
ResolveFreeze_ok(tx,d,now) ⟺ d.frozen == true
                       ∧ ( guardian_quorum_ok(tx, anchor)                              -- m-of-n anchor.guardians, KHÔNG 1 guardian
                         ∨ now ≥ d.freeze_deadline )                                   -- SSR-6 auto-hoàn timeout
                       ∧ Σ outputs(→sender, d.asset) == d.amount ∧ Σ outputs(→sender, lovelace) == d.min_ada
ReclaimTimeout_ok(tx,d,now) ⟺ now ≥ d.reclaim_deadline
                       ∧ d.receiver_consent == false
                       ∧ Σ outputs(→sender, d.asset) == d.amount ∧ Σ outputs(→sender, lovelace) == d.min_ada  -- SS-11/SS-12
```
Ràng buộc chung mọi đường: `list.count(inputs ∈ Script(smartsend)) == 1` (SS-7′, mẫu `activation_vault.ak:106-110`) → escrow-UTxO chi đúng một lần. `fee_covered` là số audit tham chiếu Feecover-vault ngoài — **KHÔNG bao giờ xuất hiện trong biểu thức output ở trên** (SS-12).

### 2.3 Đồng-ý người nhận (Accept)
```
Accept_ok(tx, d) ⟺ verify_controller_sig(d.receiver_commit, tx via anchor)   -- SS-9′ (vá SSR-3)
                  ∧ d_next == d với receiver_consent := true, phần value BẤT-BIẾN
```
`Accept` chỉ chuyển cờ `receiver_consent`, KHÔNG chạm value (không rút, không đổi đích).

### 2.4 Tập factor huỷ neo enroll-time
```
factors_valid(d, tx) ⟺ ∀ f ∈ proofs(tx) : f ∈ enroll_set(anchor)            -- SSR-4
                       ∧ verify(f) == OK
                       ∧ independent_of_seed(f) khi d yêu-cầu               -- SS-6
```
Tập factor Cancel neo trong ANCHOR lúc enroll, KHÔNG tin cờ datum lúc Open (chống kẻ Open đặt factor dễ cho mình).

---

## 3. Bất biến sổ sách LÕI (conservation)

Escrow-UTxO là **stateful** (mang `SmartSendDatum`) nhưng KHÔNG tự in/tự huỷ token:

- **Value-conservation (SS-1/SS-12):** hòm khoá đúng `amount + min_ada`; mọi đường thoát trả **byte-perfect** — Cancel/ReclaimTimeout/ResolveFreeze-timeout → sender, Finalize → receiver; `fee_covered` không bao giờ vào biểu thức output; không tạo, không mất, không redirect (T-SS-1). Nguồn conservation này trước ghi ở [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) §3, nay là bất biến lõi của module Smartsend.
- **Serial (eUTXO):** escrow-UTxO spent đúng một lần (SS-7′) → luật ledger Cardano chặn double-spend, không có hai đường thoát cùng chạy.

Module KHÔNG đặc tả tokenomics LAMP/MAGIC/CARP (thuộc MagicLamp) — chỉ khẳng định hòm giữ được asset bất kỳ và thoát qua đúng ba đường.

---

## 4. Bảng bất biến

### 4.A Smartsend core (`SS-*` / `SSR-*`)

| Mã | Mô tả |
|---|---|
| **SS-1** | Value-conservation: escrow khoá đúng `amount + min_ada`; Cancel→sender / Finalize→receiver byte-perfect trên cả `asset` lẫn `lovelace:min_ada`. |
| **SS-2** | Cửa sổ-veto: `Cancel` chỉ khi `now < veto_deadline` (ép cận trên validity-range); `Finalize` chỉ khi `now ≥ veto_deadline` (ép cận dưới) — chặn off-by-one biên (SSR-5). |
| **SS-3** | `Cancel` thoả `unlock_policy` (≥ `factors_required`, mỗi proof verify OK, **distinct FactorKind** — chặn proof trùng lặp/rỗng, SSR-13). |
| **SS-4** | `amount ≥ large_threshold` ⇒ `Finalize` đòi `receiver_consent == true`. Với `receiver_commit == #""` (đích ngoài-Phoenix), `large_threshold` KHÔNG áp được on-chain — xem SS-9′ (SSR-3). |
| **SS-5′** | `Finalize` ép `Σ outputs(→receiver, asset) == amount ∧ Σ outputs(→receiver, lovelace) == min_ada` BẰNG ĐÚNG (vá SSR-1/SSR-2), không đổi hướng, không dư output. |
| **SS-6** | `independent_of_seed` ⇒ mọi factor Cancel KHÁC gốc khoá gửi (guardian/ZK-context/thiết bị-2). Chống trộm. |
| **SS-7′** | Escrow-UTxO spent đúng 1 lần; `list.count(inputs ∈ Script(smartsend)) == 1` (vá SSR-1, mẫu `activation_vault.ak:106-110`) — MVP không gộp batch nhiều escrow trong 1 tx. |
| **SS-8** | `Freeze` (guardian-sig neo anchor) → `frozen := true`, chặn `Finalize`; chỉ dùng được **trong cửa sổ** (`now < veto_deadline`) — sau đó Freeze vô nghĩa vì đã tới hạn Finalize (SSR-6). |
| **SS-8′** | State-machine Freeze: `Open → {Cancelled, Finalized, Frozen}`; `Frozen → {Cancelled(guardian-quorum m-of-n anchor.guardians), Cancelled(now≥freeze_deadline, auto-hoàn sender)}` — KHÔNG `Finalized` từ `Frozen` (SSR-6). Guardian-quorum, KHÔNG 1 guardian đơn lẻ, để tránh grief. |
| **SS-9′** | `Accept` chỉ set `receiver_consent` (`false→true`); **mọi field khác datum byte-perfect bất biến** (chặn đổi lén `unlock_policy`/`large_threshold`/`veto_deadline`, SSR-14); verify controller-sig của `receiver_commit` qua anchor (vá SSR-3), không chạm value. |
| **SS-10** | `open_slot` hữu hạn (lower-bound), ép `tx_lo(Open) ≤ open_slot ≤ tx_hi(Open)` (SSR-11, chặn khai-man quá khứ); `veto_deadline = open_slot + window`; `window ≥ min_window_floor` **sàn cứng on-chain** kể cả khi "2-bên thoả" (SSR-10) — nới rộng được, KHÔNG rút ngắn dưới sàn. |
| **SS-11** | (vá SSR-7) `now ≥ reclaim_deadline ∧ receiver_consent == false` → `ReclaimTimeout` hoàn về sender byte-perfect (chống kẹt vĩnh viễn, đóng deadlock khoản lớn không-Accept). |
| **SS-12** | (vá SSR-9) `fee_covered` là số **audit tham chiếu Feecover-vault ngoài** — validator KHÔNG đọc `fee_covered` để tính output; mọi đường thoát luôn chi trọn `amount + min_ada`. Nếu cấn phí từ escrow: tách rõ `min_ada` về sender/receiver theo SS-1/SS-5′, `txFee` do Feecover-vault nguồn riêng — `fee_covered` không được xuất hiện trong biểu thức output. |
| **SSR-4** | Factor `independent_of_seed` + tập factor Cancel neo enroll-time trong ANCHOR (không tin cờ datum lúc Open); Cancel verify factor khớp anchor-enroll. |

### 4.B Bất biến kế thừa (dẫn chiếu Rebirthme — KHÔNG nhân bản)

| Mã | Vai trò trong Smartsend | Nguồn định nghĩa |
|---|---|---|
| **I-WALLET-1** | Cổng chi nguồn nạp hòm: `anchor_controller_ok`. | `Rebirthme-Math §4.A` |
| **I-GUARD-\*** | Guardian dùng cho factor Cancel (Guardian) + SS-8 Freeze. | `Rebirthme-Math §4.I` |
| **I-LIMIT-\*** | Anti-drain là nền cho chống trộm SS-6 (giới hạn thiệt hại khi lộ khoá). | `Rebirthme-Math §4.B` |
| **I-CURVE-5** | Second-factor (kể cả factor Cancel) PHẢI khác gốc seed. | `Rebirthme-Math §4.C` |

---

## 5. Mệnh đề ép từng thao tác

Năm đường thoát 1 escrow-UTxO (SS-7′ đếm input==1):
- **Cancel** — SS-2 (`now<deadline`, cận trên) ∧ SS-3/SSR-4 (đa yếu tố neo anchor-enroll, distinct FactorKind, SS-6 khác gốc) ∧ SS-1/SS-12 (→sender byte-perfect, `amount+min_ada`).
- **Finalize** — SS-2 (`now≥deadline`, cận dưới) ∧ SS-4 (consent nếu ≥ large_threshold, trừ đích ngoài-Phoenix) ∧ SS-5′/SS-12 (→receiver byte-perfect, `amount+min_ada`).
- **Freeze** — SS-8 (`now<deadline` ∧ guardian-sig neo anchor) → `frozen:=true`.
- **ResolveFreeze** — SS-8′ (guardian-quorum m-of-n HOẶC `now≥freeze_deadline`) → hoàn sender byte-perfect.
- **ReclaimTimeout** — SS-11 (`now≥reclaim_deadline ∧ ¬consent`) → sender byte-perfect.
- **Accept** — SS-9′ (verify controller-sig `receiver_commit` qua anchor, chỉ set cờ, mọi field khác bất biến).

Định lý T-SS-1..3 (§7) với 3 tiền đề (§6).

**Mảng lỗ đối kháng đã đóng (nguồn: red-team `_smartsend-redteam-2026-07-08.md`, SSR-1..14):**

| Lỗ | Cách vá trong bản này |
|---|---|
| SSR-1 (double-sat batch) | SS-7′ ép `count(inputs∈Script(smartsend))==1`; SS-5′ bỏ `>=`, ép `==` |
| SSR-2 (minADA/output-phụ) | `min_ada` tách field riêng, SS-1/SS-5′/SS-11 ép byte-perfect trên cả `asset` lẫn `lovelace:min_ada` |
| SSR-3 (consent forge / đích ngoài-Phoenix) | SS-9′ verify controller-sig qua anchor, bỏ `receiver_sig` tự do; SS-4 loại trừ đích `receiver_commit==#""` khỏi enforce on-chain |
| SSR-4 (factor tự khai) | Factor Cancel neo `anchor.enroll_set`, không tin cờ datum Open (§2.4) |
| SSR-5 (veto-race biên) | SS-2 chỉ rõ Cancel dùng cận trên (`tx_hi`), Finalize dùng cận dưới (`tx_lo`) |
| SSR-6 (Freeze grief/kẹt) | SS-8 giới hạn Freeze trong cửa sổ; SS-8′ state-machine + `freeze_deadline` auto-hoàn |
| SSR-7 (deadlock khoản lớn) | SS-11 + `reclaim_deadline` + redeemer `ReclaimTimeout` |
| SSR-8 (Finalize permissionless grief) | Chấp nhận: an toàn (không mất tiền), chỉ mất quyền huỷ phút chót — giới hạn ghi ở §9 |
| SSR-9 (fee_covered phá conservation) | SS-12: `fee_covered` chỉ audit, không vào biểu thức output |
| SSR-10 (window=0 vô hiệu) | SS-10: `window ≥ min_window_floor` sàn cứng, "2-bên thoả" chỉ nới rộng |
| SSR-11 (open_slot khai-man) | SS-10: ép `tx_lo(Open) ≤ open_slot ≤ tx_hi(Open)` |
| SSR-12 (ZK-proof replay, Phase 2) | Public-input ZK phải gồm `blake2b_256(own_ref ‖ escrow_datum_hash)` — ghi ở Tech §5.1 |
| SSR-13 (factor trùng/rỗng) | SS-3: distinct FactorKind + verify thật mỗi proof |
| SSR-14 (Accept đổi lén field) | SS-9′: mọi field khác `receiver_consent` byte-perfect bất biến |

---

## 6. Tiền đề (điều kiện để định lý Smartsend đứng)

Ba định lý Smartsend chỉ đứng khi ba tiền đề thoả (nếu không, tuyên bố an toàn KHÔNG áp):

- **(a) Bật trước khi mất khoá.** Người gửi bật Smartsend + cài factor Cancel TRƯỚC thời điểm khoá gửi bị lộ. Bật sau khi đã lộ không cứu được khoản đã đi.
- **(b) Factor neo anchor-enroll (SSR-4).** Tập factor Cancel neo trong anchor lúc enroll; không tin cờ datum lúc Open → kẻ Open không tự đặt factor dễ.
- **(c) Guardian / anti-drain sẵn sàng.** Chống trộm SS-6 dựa guardian (Freeze/factor) + anti-drain (`limit_meter`, `Rebirthme §4.B`).

---

## 7. Định lý an toàn + chứng minh phác thảo

**T-SS-1 (No-loss / Bảo toàn giá trị).** Với mọi đường thoát, tổng value ra khỏi hòm = `amount + min_ada` và chỉ tới sender (Cancel/ReclaimTimeout/ResolveFreeze-timeout) hoặc receiver (Finalize), byte-perfect (SS-1 + SS-5′ + SS-12). Không đường nào tạo/mất/redirect tiền; SS-7′ đảm bảo không hai đường cùng chạy. ⇒ tiền không bao giờ "bốc hơi" khỏi hòm. ∎

**T-SS-2 (Veto-safety).** Cửa sổ-veto rời nhau: `Cancel` chỉ `now < veto_deadline` (cận trên), `Finalize` chỉ `now ≥ veto_deadline` (cận dưới) (SS-2). ⇒ không có khoảng thời gian nào vừa huỷ được vừa chốt được; người gửi luôn có toàn bộ cửa sổ để huỷ trước khi tiền tới người nhận. ∎

**T-SS-3 (Anti-theft).** NẾU ba tiền đề §6 thoả: kẻ chỉ có khoá gửi KHÔNG huỷ được (Cancel đòi factor khác gốc seed, SS-6/SSR-4) và KHÔNG chuyển hướng chốt được (Finalize ép →receiver, SS-5′); guardian còn Freeze được (SS-8/SS-8′), không kẹt vĩnh viễn (freeze_deadline). ⇒ lộ khoá gửi không đủ để rút khoản đang trong hòm. Định lý CÓ ĐIỀU KIỆN — hiệu lực phụ thuộc tiền đề (c) (guardian/anti-drain sẵn sàng); trạng thái tiền đề (c) → xem STATUS §9. ∎ (có điều kiện).

---

## 8. Giả định tin cậy (ngoài phạm vi chứng minh)

- **Ledger Cardano** thực thi đúng luật eUTXO (double-spend từ chối → SS-7′).
- **Anchor + guardian** (thuộc Rebirthme) đúng như đặc tả nguồn — cổng nạp và Freeze dựa vào đó.
- **Spectra / Glint** (factor bối cảnh ZK) có liveness/anti-replay chống ảnh AI-generated.
- **Enroll-set trong anchor** trung thực phản ánh factor người dùng cài (SSR-4).

---

## 9. [CẦN CHỐT] còn treo

- **[CẦN CHỐT-SS2]** `reclaim_deadline` đặt tương đối `veto_deadline` bao lâu (chống kẹt vs tránh reclaim-quá sớm).
- **[CẦN CHỐT-SS3]** `window` mặc định + `min_window_floor` (SS-10) + có cho 2-bên thoả thuận ngoài {24,48,72}h không.
- **[CẦN CHỐT-SS4]** Enforce I-CURVE-5 (factor Cancel khác gốc seed) ở builder — dependency chung với Rebirthme [CẦN CHỐT-W8].
- **[CẦN CHỐT-SS-FREEZE]** `freeze_deadline` đặt tương đối lúc Freeze bao lâu (SS-8′, vd 30 ngày) — cân giữa an toàn vs kẹt tiền.

---

## 10. Checklist cho auditor

- [ ] Value-conservation: Cancel/ReclaimTimeout/ResolveFreeze-timeout→sender, Finalize→receiver byte-perfect trên cả `asset` lẫn `min_ada` (SS-1/SS-5′/SS-11/SS-12).
- [ ] Cửa sổ-veto rời nhau (SS-2): veto-race test — Cancel tại `deadline` FAIL, Finalize trước `deadline` FAIL; đúng cận (Cancel=cận trên, Finalize=cận dưới).
- [ ] Double-satisfaction: 2-escrow-1-output reject (SS-7′).
- [ ] Redirect output / output-phụ / minADA lệch khi Finalize reject (SS-5′/SS-12).
- [ ] Consent forge reject; Accept chỉ set cờ, mọi field khác bất biến (SS-9′/SSR-14).
- [ ] Deadlock khoản lớn: người nhận không consent → ReclaimTimeout hoàn sender (SS-11).
- [ ] Factor Cancel neo anchor-enroll, distinct FactorKind, khác gốc seed (SSR-4/SS-3/SS-6/I-CURVE-5).
- [ ] Freeze treo được trong cửa sổ, chặn Finalize; guardian-quorum hoặc `freeze_deadline` mới thoát Freeze (SS-8/SS-8′).
- [ ] `fee_covered` không xuất hiện trong biểu thức output (SS-12).
- [ ] `open_slot` ràng trong validity-range tx Open (SS-10/SSR-11); `window ≥ min_window_floor` (SSR-10).
- [ ] Nền dẫn chiếu: `anchor_controller_ok` + guardian + anti-drain (Rebirthme).

---

## Nguồn

Nguồn thiết kế nội bộ (không công khai).
Math canonical (dẫn chiếu, KHÔNG sửa): `PhoenixKey-Math.md` §10, §11.
Hạ tầng nền (dẫn chiếu): [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) §2.1, §4.B, §4.C, §4.I.
Code: `PhoenixKey-Validator/validators/smartsend_escrow.ak`.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#smartsend)

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
