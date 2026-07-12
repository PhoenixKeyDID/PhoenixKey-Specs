# PhoenixKey — Protectme — ĐẶC TẢ TOÁN HÌNH THỨC (cho AUDITOR)

> **Module:** Protectme (Bảo hiểm phủ tài sản) · **Loại doc:** Toán hình thức (auditor) ·
> **Ngày:** 2026-07-09.
> **Đối tượng đọc:** auditor smart-contract + nhà kiểm toán tokenomic. Đây là bản rút gọn
> về **phần TOÁN kiểm được**: ký hiệu, miền, công thức, bảng bất biến (kèm ánh xạ tới dòng
> validator), không gian trạng thái, và định lý có chứng minh phác. Nó KHÔNG lặp lại phần
> FEAT/điều hành — đọc tài liệu cùng bộ [PhoenixKey-Protectme-Vi-Feat.md](./PhoenixKey-Protectme-Vi-Feat.md) (WHAT/WHY) và [PhoenixKey-Protectme-Exec.md](./PhoenixKey-Protectme-Exec.md) (quyết định/rủi ro).
>
> Mọi mục **[N]** / **[PROT-n]** = đề xuất; chỉ normative sau khi maintainer duyệt.
>
> **Nguồn:**
> - Tài liệu cùng bộ: [PhoenixKey-Protectme-Vi-Feat.md](./PhoenixKey-Protectme-Vi-Feat.md), [PhoenixKey-Protectme-Tech.md](./PhoenixKey-Protectme-Tech.md), [PhoenixKey-Protectme-Exec.md](./PhoenixKey-Protectme-Exec.md). Nguồn thiết kế nội bộ (không công khai).
> - Code đối chiếu: `PhoenixKey-Validator` nhánh `feat/protectme-payout`:
>   `lib/phoenixkey/protectme_logic.ak`, `protectme_types.ak`, `validators/protectme_payout.ak`.
> - Gaming: Nguồn thiết kế nội bộ (không công khai) (VG-P1..P10).
> - Whitepaper §4 (nanogic = byte·ngày), §5 (P\*=1: 1 CARP = 1 MAGIC sức mua, KHÔNG neo fiat).
> - Math canonical: [PhoenixKey-Math.md](./PhoenixKey-Math.md) (đơn vị CARP/MAGIC). Curve I-CURVE-4/5: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) + addendum Curve-Routing (CHƯA gộp vào Math.md).
>
> → Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#protectme)
>
> **CONTRACT đơn vị (bất di):** premium **định giá MAGIC** (nanogic), **THANH TOÁN CARP**
> (P\*=1, qua Feecover, KHÔNG `collectToTreasury`). `coverage_cap`, `loss_eligible`,
> `amount` (payout) **cùng đơn vị CARP**. MAGIC chỉ là nhãn giá. Đây là điều kiện tiên
> quyết để bất biến solvency KHÔNG đứt gãy thứ nguyên (bỏ nanogic-vs-lovelace bản cũ).

---

## 0. Trục kiểm của auditor (đọc trước)

Protectme có **hai mặt phẳng** với mức bảo đảm KHÁC hẳn nhau — auditor phải soi riêng:

| Mặt phẳng | Bảo đảm | Ai gác |
|---|---|---|
| **Phía CHI** (payout/reject escrow) | **On-chain, mật mã** — validator ép công thức + trần + challenge + T2 + bảo toàn giá trị | `protectme_payout.ak` |
| **Phía DUYỆT** (loss_eligible, cause, single_factor flags) | **Off-chain, phán người** (committee/panel) — KHÔNG có oracle mật mã (C.6) | committee off-chain |
| **Phía NẠP** (escrow = đúng bucket, một lần-per-claim) | **Treasury release gate** (đội backend) — validator này KHÔNG ép uniqueness | Treasury (ngoài PR) |

**Kết luận toán học chỉ mạnh tới đâu input được chứng thực.** Validator ép `amount ==
expected_amount(loss, policy, cause)` một cách chắc chắn; nhưng `loss` và `policy` là số
committee ĐẶT vào datum. Định lý §7 đều có dạng "GIẢ SỬ input datum trung thực ⇒ …".
Ranh giới tin cậy F3/F6 là chỗ giả thiết đó có thể vỡ — auditor tập trung ở đó.

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#protectme)

---

## 1. Ký hiệu

### 1.1 Miền cơ sở

| Ký hiệu | Miền | Nghĩa |
|---|---|---|
| $s$ | $\mathbb{N}$ (SlotNo) | slot Cardano |
| $e$ | $\mathbb{N}$ (EpochNo) | epoch |
| CARP | $\mathbb{N}$ | đơn vị dàn xếp (một `accepted_asset` Phoenix Treasury); mọi cap/loss/payout |
| MAGIC | $\mathbb{N}$ (nanogic $=10^9$) | đơn vị kế toán/sức mua; nhãn giá premium; $P^\*=1$ nên $x$ MAGIC $\equiv x$ CARP |
| $\mathrm{did}$ | ByteArray | định danh DID |

### 1.2 Nguyên nhân (Cause)

$$\mathcal{C} \triangleq \{\mathsf{SYS},\ \mathsf{USER},\ \mathsf{GREY}\}$$

- $\mathsf{SYS}$ = SYSTEM_FLAW (lỗi hệ thống, trách nhiệm protocol).
- $\mathsf{USER}$ = USER_NEGLIGENCE (sơ suất user).
- $\mathsf{GREY}$ = chưa gán ⇒ **xử như $\mathsf{USER}$** (bảo thủ cho pot).

> Code (`protectme_types.ak`) chỉ mã hoá `Cause = SystemFlaw | UserNegligence` — $\mathsf{GREY}$
> giải quyết ở tầng triage off-chain (B.4) rồi hạ về `UserNegligence` trước khi tạo escrow.
> Auditor lưu: on-chain KHÔNG có $\mathsf{GREY}$; nó không phải trạng thái chi trả.

### 1.3 Policy (ảnh chụp lúc duyệt) — `PolicySnapshot`

$$
\pi \triangleq \big(\ \mathrm{cap}_{\mathsf{sys}},\ \mathrm{cap}_{\mathsf{user}},\ b\ (\text{copay bps}),\ D\ (\text{deductible}),\ \sigma\ (\text{single\_factor}),\ \alpha\ (\text{anti\_drain\_enrolled})\ \big)
$$

với $\mathrm{cap}_{\mathsf{sys}}, \mathrm{cap}_{\mathsf{user}}, D \in \mathbb{N}$ (CARP), $b \in [0, 10000] \cap \mathbb{Z}$ (basis points), $\sigma, \alpha \in \{\bot, \top\}$.

### 1.4 Claim (escrow datum) — `ClaimDatum`

$$
d \triangleq \big(\ \mathrm{claim\_id},\ \mathrm{did},\ L\ (\text{loss\_eligible}),\ c \in \mathcal{C},\ s_{\mathrm{app}}\ (\text{approved\_slot}),\ a\ (\text{amount}),\ \pi,\ \mathrm{payee},\ \mathrm{rec}\ \big)
$$

$L, a \in \mathbb{Z}$ (CARP; miền hợp lệ ép ở I-PROT-DOM); $\mathrm{payee}, \mathrm{rec}$ = script credential.

### 1.5 Config governance-tunable — `ProtectmeConfig` (apply-param)

$$
\Gamma \triangleq \big(\ K\ (\text{committee\_vkhs}),\ t\ (\text{threshold}),\ \mathrm{carp\_pol},\ \mathrm{carp\_name},\ \mathrm{pot}_{\mathsf{sys}},\ \mathrm{pot}_{\mathsf{user}},\ T_{\mathrm{wp}},\ W_{\mathrm{ch}},\ \mathrm{CAP}_{\mathrm{inc}},\ \rho\ (\text{require\_ad})\ \big)
$$

$T_{\mathrm{wp}}$ = `t_wait_payout`, $W_{\mathrm{ch}}$ = `challenge_window`, $\mathrm{CAP}_{\mathrm{inc}}$ = `incident_cap`, $\rho \in \{\bot,\top\}$ = `require_antidrain_for_single_factor` (I-PROT-CFG, **[CHỜ ANH CHỐT PROT-11]** = $\top$).

### 1.6 Ký hiệu giao dịch

- $V_{\mathrm{esc}}$ : `Value` của UTxO escrow (own-input). $\mathrm{carp}(V) \triangleq$ `quantity_of(V, carp_pol, carp_name)`.
- $\mathrm{lo}(\mathrm{tx})$ : cận dưới hữu hạn của `validity_range` (nếu $\ne$ Finite ⇒ tx fail). Ledger ép $\mathrm{lo} \le s_{\text{on-chain}}$ (F5).
- $\#\mathrm{in}(\mathrm{cred})$ : số input đang chi tại một script credential.
- $\mathrm{sole}(\mathrm{cred})$ : $\mathrm{Some}(V)$ nếu ĐÚNG một output tại `cred` (trả `Value` của nó); $\mathrm{None}$ nếu $\ne 1$.

---

## 2. Công thức Premium (định giá MAGIC, thu CARP)

Phía THU **không** ở validator payout (thu qua Feecover, đối soát cuối epoch); ghi ở đây
làm phần định giá + bất biến solvency mà auditor kinh-tế soi.

### 2.1 Hệ số rủi ro

$$
\mathrm{rmult}(\pi) = 1.0 \cdot g_{\mathrm{ad}} \cdot g_{\mathrm{gu}} \cdot g_{\mathrm{fr}} \cdot \mathrm{ncb}(\text{streak}) \cdot \mathrm{pen}_{\mathrm{bk}}
$$

trong đó (🔴 **I-PROT-FACTOR-INDEP**, C.8 — chiết khấu CHỈ khi factor độc lập với-seed):

$$
g_{\mathrm{gu}} = \begin{cases} 0.8 & \text{guardian } \wedge\ \mathrm{indep}(\text{guardian})\\ 1.0 & \text{ngược lại}\end{cases}
\qquad
g_{\mathrm{fr}} = \begin{cases} 0.9 & \text{freeze-2nd } \wedge\ \mathrm{indep}(\text{2nd})\\ 1.0 & \text{ngược lại}\end{cases}
$$

$$
g_{\mathrm{ad}} = \begin{cases} 0.7 & \text{anti-drain bật} + \text{limit chặt}\\ 1.0 & \text{ngược lại}\end{cases}
$$

$$
\mathrm{indep}(f) \Leftrightarrow (\text{PersonDID trên enclave KHÁC lineage/attestation})\ \vee\ (\text{kênh không-seed: VeData-Glint / EmailOracle})
$$

**Rigor note (2026-07-12):** khác các vị từ khác trong tài liệu này (vd $\mathrm{expected}(c,L,\pi)$
ở §7, định nghĩa đầy đủ trên kiểu dữ liệu đã khai ở §1), $\mathrm{indep}(f)$ ở trên **KHÔNG** phải
một vị từ hình thức trên kiểu dữ liệu đã định nghĩa — "lineage/attestation" chưa có ký hiệu/miền
trong §1, và vế phải chỉ là mô tả văn xuôi. Đây là **tham số kinh-tế** (định giá premium theo
mức độc lập của yếu tố thứ hai), không phải bất biến an toàn on-chain — độ ưu tiên hình thức hoá
THẤP so với các định lý §7. Muốn hình thức hoá đầy đủ, cần trước hết định nghĩa `lineage`/
`attestation` như hai trường dữ liệu cụ thể (nguồn: TAAD datum / enclave attestation record —
xem `PhoenixKey-Anchorme-Math.md`) rồi viết lại $\mathrm{indep}$ như một hàm thuần trên hai
trường đó thay vì mô tả bằng lời. Không chứng thực được $\mathrm{indep}$ ⇒ **mặc định $\bot$** (bảo thủ). Đây là chỗ Protectme
định giá TRỰC TIẾP I-CURVE-5 ("second-factor phải khác seed"; nguồn canonical: Rebirthme-Math §, addendum Curve-Routing — CHƯA gộp vào `PhoenixKey-Math.md`) thành
tham số phí.

### 2.2 Premium

$$
\boxed{\ \mathrm{premium}(\pi) = \mathrm{base\_rate} \cdot (\mathrm{cap}_{\mathsf{sys}} + \mathrm{cap}_{\mathsf{user}}) \cdot \mathrm{rmult}(\pi) \cdot \ell_{\mathrm{sf}}\ }
$$

$\ell_{\mathrm{sf}} \ge 1$ = `single_factor_loading` (tải rủi ro cohort không có factor độc lập, C.8).
Đơn vị **nhãn giá = nanogic MAGIC/kỳ**; **THU = CARP** (P\*=1) qua `FeecoverEpochSettle`
$\Rightarrow \mathrm{pot}_{\mathsf{user}} \mathrel{+}= \sum \mathrm{premium\_CARP}$.

> **[CHỜ ANH CHỐT PROT-3]** `base_rate` (0.5–2%/năm?), $\theta_{\mathrm{treasury}}$, các hệ số
> $g_\bullet$, $b$ (copay 20–50%?), $D$. **[PROT-11]** $\ell_{\mathrm{sf}}$, tiêu chí $\mathrm{indep}$.

---

## 3. Công thức Coverage (loss_eligible) — chống trùng ví theo-DID

$$
\boxed{\ L(\mathrm{did},\mathrm{inc}) = \underbrace{\sum_{\mathrm{tx}\in\text{theft}(\mathrm{inc})} W(\mathrm{tx})}_{\text{net rời ví theo-DID}} - R_{\mathrm{cancel}} - R_{\mathrm{freeze}} - R_{\mathrm{vault}}\ }
$$

- $W(\mathrm{tx})$ = **net-rời kho** — TÁI DÙNG ĐÚNG định nghĩa Withdrawal-Limit (đọc chain, không cãi).
- $R_{\mathrm{cancel}}$ = phần `PendingLargeWithdrawal` chủ đã Cancel (I-LIMIT-CANCEL).
- $R_{\mathrm{freeze}}$ = phần bị Freeze chặn kịp (Frozen→safe-address).
- $R_{\mathrm{vault}}$ = UTxO còn tại `vault_addr(did)` sau rotate/recover (= KHÔNG mất).

Nếu $c = \mathsf{USER}$: trừ thêm $D$ (deductible) ở bước amount (§4), KHÔNG ở đây.

> **Auditor lưu (F3/F6):** $L$ là số **committee chứng thực** đưa vào datum. Validator
> KHÔNG tự tính $L$ từ chain (nó không đọc anti-drain log). Nó chỉ ép $L \ge 0$ (I-PROT-DOM)
> + $a = \mathrm{expected}(L,\pi,c)$. Tính đúng $L$ = trách nhiệm off-chain (A.6/B.5).

---

## 4. Công thức Payout — `expected_amount`

Định nghĩa hàm chi (code: `protectme_logic.expected_amount`):

$$
\mathrm{gross}(c, L, \pi) = \min\big(L,\ \mathrm{cap}(c,\pi)\big),\qquad
\mathrm{cap}(c,\pi) = \begin{cases}\mathrm{cap}_{\mathsf{sys}} & c=\mathsf{SYS}\\ \mathrm{cap}_{\mathsf{user}} & c=\mathsf{USER}\end{cases}
$$

$$
\boxed{\ \mathrm{expected}(c, L, \pi) = \begin{cases}
\mathrm{gross}(c,L,\pi) & c = \mathsf{SYS}\quad(\text{không co-pay, không deductible})\\[4pt]
\big\lfloor\ \max(0,\ \mathrm{gross} - D)\ \cdot\ \dfrac{10000 - b}{10000}\ \big\rfloor & c = \mathsf{USER}
\end{cases}\ }
$$

Chia nguyên (`* (10000 - copay_bps) / 10000` trong Aiken = floor). Ví dụ đối chiếu test:
$\mathrm{expected}(\mathsf{USER}, 1000, \pi)$ với $\mathrm{cap}_{\mathsf{user}}{=}500, D{=}100, b{=}2000$:
$(500-100)\times 0.8 = 320$ ✓ (`amount_user_copay_deductible`).

---

## 5. Bảng bất biến (kèm ánh xạ code)

> Namespace **I-PROT-\*** (viết đầy đủ; tương đương `PROT-*` trong code comment — engine
> MAGIC-Paymaster đã chiếm `PM-*` nên KHÔNG dùng `I-PM-*`). Cột "Code" trỏ predicate/dòng
> trong `protectme_logic.ak` (nhánh `feat/protectme-payout`). Mã bất biến theo brief §2:
> tiền tố module = **PROT**.

### 5.1 Bất biến ĐÃ ép on-chain (validator gác được)

| ID | Phát biểu hình thức | Code (ánh xạ) |
|---|---|---|
| **I-PROT-DOM** | $L \ge 0 \wedge \mathrm{cap}_{\mathsf{sys}}\!\ge\!0 \wedge \mathrm{cap}_{\mathsf{user}}\!\ge\!0 \wedge D\!\ge\!0 \wedge 0\le b\le 10000$ | `policy_domain_ok` |
| **I-PROT-NO-DOUBLE** | $a = \mathrm{expected}(c, L, \pi)\ \wedge\ a>0\ \wedge\ \mathrm{carp}(V_{\mathrm{esc}}) = a$ | `payout_ok`: `d.amount == expected`, `d.amount > 0`, `escrow_carp == d.amount` |
| **I-PROT-INCIDENT-CAP** | $a \le \mathrm{CAP}_{\mathrm{inc}}$ | `payout_ok`: `d.amount <= cfg.incident_cap` |
| **I-PROT-VALUE-CONSERVE** (HIGH3) | $\mathrm{sole}(\mathrm{payee}) = \mathrm{Some}(V_{\mathrm{esc}})$ — **TOÀN BỘ** value (ADA+CARP+token) về đúng 1 payee, không skim | `payout_ok`: `sole_output_value_at(tx.outputs, d.payee_cred) == Some(escrow_val)` |
| **I-PROT-NO-DBL-SAT** (CRIT1) | $\#\mathrm{in}(\mathrm{own\_cred}) = 1$ — đúng một escrow input | `count_inputs_at(tx, own_cred) == 1` (cả payout + reject) |
| **I-PROT-CRED-DISTINCT** (CRIT2) | $\mathrm{payee}\ne\mathrm{rec} \wedge \mathrm{payee}\ne\mathrm{own} \wedge \mathrm{rec}\ne\mathrm{own}$ | `payout_ok`: 3 bất đẳng thức `d.payee_cred != …` |
| **I-PROT-CHALLENGE** | $\mathrm{lo}(\mathrm{tx}) \ge s_{\mathrm{app}} + T_{\mathrm{wp}} + W_{\mathrm{ch}}$ | `payout_ok`: `lo >= d.approved_slot + cfg.t_wait_payout + cfg.challenge_window` |
| **I-PROT-T2** | $\exists!$ output tại $\mathrm{rec}$ mang `ClaimRecord{claim_id, did, a, c}` khớp | `record_ok` |
| **I-PROT-SINGLE-FACTOR** | $c=\mathsf{USER} \wedge \rho \wedge \sigma \Rightarrow \alpha$ (SYS luôn qua) | `single_factor_ok` |
| **I-PROT-NO-CROSS-BUCKET** | Reject: $\mathrm{sole}(\mathrm{bucket}(c)) = \mathrm{Some}(V_{\mathrm{esc}})$; $\mathrm{bucket}(\mathsf{SYS})=\mathrm{pot}_{\mathsf{sys}}, \mathrm{bucket}(\mathsf{USER})=\mathrm{pot}_{\mathsf{user}}$ | `reject_ok`: `sole_output_value_at(…, bucket_cred(d.cause,cfg))` |
| **I-PROT-COMMITTEE** (F7) | Reject: $\big|\{k\in\mathrm{unique}(K): k\in\mathrm{sig}(\mathrm{tx})\}\big| \ge t$ | `committee_ok` (dedup qua `list.unique`) |

### 5.2 Bất biến CHỜ enforce on-chain — trust-boundary (auditor SOI KỸ)

| ID | Phát biểu | Trạng thái | Rủi ro |
|---|---|---|---|
| **I-PROT-BEACON-ONESHOT** 🔴 | Mỗi $\mathrm{claim\_id}$ được nạp escrow + chi ĐÚNG MỘT LẦN (chống double-payout) | **F3: KHÔNG ép on-chain.** Validator không kiểm uniqueness `claim_id` | Treasury nạp HAI escrow cùng `claim_id` ⇒ **cả hai trả được**. Cần **beacon một lần theo `claim_id` (state NFT)** — `protectme_beacon.ak` chưa tồn tại (chặn-merge, Tech §3). Hiện dựa `dao_release_ok` nạp không trùng (đội backend) |
| **I-PROT-EVIDENCE-ONCHAIN** | $L$ = net rời ví (anti-drain log) đọc từ chain, off-chain chỉ bổ trợ | **F6: committee ĐẶT $L$** vào datum; validator không đọc log | Committee đặt $L$ sai ⇒ $a$ sai theo. Không có oracle mật mã cho "trộm hay không" (C.6) |
| **I-PROT-FACTOR-INDEP** | $\mathrm{indep}(f)$ chứng thực trước khi cho chiết khấu | **F6: cờ $\sigma,\alpha$ committee đặt** — tripwire, không chống committee cố sai | Sức mạnh = độ trung thực committee |
| **I-PROT-SOLVENCY** | (kinh tế, §8) | off-chain đo lăn + DAO hiệu chỉnh | Không phải bất biến on-chain |

### 5.3 Bất biến ở tầng luồng (không ở validator escrow này)

I-PROT-CAUSE-SPLIT, I-PROT-WAIT-ACTIVATE, I-PROT-REPORT-WINDOW, I-PROT-POOL-CAP (tổng/cửa sổ),
I-PROT-SUBROGATION, I-PROT-LAPSE, I-PROT-TRIAGE-BOUNDARY, I-PROT-PREMIUM-INCENTIVE — ép ở
adjudicate/DAO-gate/Feecover/Treasury (B.4/B.6/A.3), KHÔNG ở `protectme_payout`. Auditor
soi ở đường nạp (đội backend) + governance gate, không ở validator escrow.

---

## 6. Không gian trạng thái + đồ thị chuyển

Một claim đi qua các trạng thái sau (trạng thái NẠP/DUYỆT off-chain, chỉ 3 trạng thái sau
là on-chain: escrow-sống, đã chi, đã hoàn):

```
   (off-chain)          (off-chain)         ┌── Payout ──►  PAID     (burn beacon; amount→payee, record→rec)
  OPENED ──triage──► ADJUDICATED ──nạp──► ESCROW-LIVE ─┤
   (mở claim)         (chốt cause+amount)   (UTxO on-chain) └── Reject ──►  REJECTED  (burn beacon; escrow→bucket(cause))
```

- **OPENED → ADJUDICATED:** committee/panel chốt $c$ và $a=\mathrm{expected}(c,L,\pi)$ (B.4). Không tx.
- **ADJUDICATED → ESCROW-LIVE (Tx-A):** release bucket$(c)$ → escrow UTxO @ `protectme_payout`,
  mint beacon (khi có). Ràng: $\mathrm{carp}(V_{\mathrm{esc}})=a$; committee ký $\ge t$.
- **ESCROW-LIVE → PAID (Tx-D):** cổng `payout_ok` (I-PROT-DOM..T2), sau challenge-window.
  Permissionless-after-window.
- **ESCROW-LIVE → REJECTED (Tx-E):** cổng `reject_ok` (I-PROT-NO-CROSS-BUCKET + COMMITTEE),
  hoàn escrow về ĐÚNG bucket$(c)$. Committee ký $\ge t$.

**Bất biến chuyển:** ESCROW-LIVE là trạng thái hấp thụ theo eUTXO — UTxO escrow chi đúng
MỘT lần (PAID **xor** REJECTED), không quay lại. Nhánh hở duy nhất: hai escrow ĐỘC LẬP cùng
`claim_id` (I-PROT-BEACON-ONESHOT chưa đóng) → hai lần vào ESCROW-LIVE cho cùng một claim.

---

## 7. Định lý (có chứng minh phác)

> Ba định lý CONTRACT yêu cầu. Mỗi cái có dạng **có điều kiện** trên input datum trung
> thực — nêu rõ giả thiết vì đó là ranh giới an toàn thật (§0, §5.2).

### Định lý 1 (Payout ≤ pot) — solvency mỗi giao dịch

**Phát biểu.** Với mọi tx `Payout` qua validator, số CARP chi $\le$ CARP escrow, và
escrow đã được Treasury nạp từ đúng bucket:
$$
a \le \mathrm{carp}(V_{\mathrm{esc}}) \quad\text{và thực tế}\quad a = \mathrm{carp}(V_{\mathrm{esc}}).
$$

**Chứng minh.** I-PROT-NO-DOUBLE ép `escrow_carp == d.amount`, tức $\mathrm{carp}(V_{\mathrm{esc}}) = a$
(đẳng thức, không chỉ bất đẳng thức). Escrow chỉ tồn tại nếu Treasury đã `release(bucket(c), a)`
từ bucket tương ứng (B.6, ngoài validator). Do đó tổng chi qua các claim $\sum a_i =
\sum \mathrm{carp}(V_{\mathrm{esc},i}) \le \mathrm{pot}(c)$ **miễn là** mỗi escrow trừ pot đúng một lần
lúc nạp. $\square$

**Điều kiện then chốt:** đẳng thức chỉ chặn "chi vượt escrow"; chặn "chi vượt POT" phụ thuộc
**I-PROT-BEACON-ONESHOT** (mỗi claim nạp một escrow). Nếu F3 hở (nạp trùng), một $\mathrm{claim\_id}$
rút được $2a$ trong khi pot chỉ trừ $a$ ⇒ **định lý VỠ**. → Đây là lỗ auditor phải chốt đóng
(§5.2, blocker beacon; gaming VG-P1).

### Định lý 2 (Không chi hai lần) — no-double-payout

**Phát biểu.** Không tồn tại hai tx `Payout` hợp lệ cùng tiêu một escrow UTxO; và trong
một tx, không escrow nào bị thoả mãn kép (double-satisfaction).

**Chứng minh.**
1. *Trong-tx:* I-PROT-NO-DBL-SAT ép $\#\mathrm{in}(\mathrm{own\_cred}) = 1$. Nếu tx gộp hai escrow
   để dùng chung một output payee (kiểu double-satisfaction), $\#\mathrm{in} = 2 \ne 1$ ⇒ fail
   (test `payout_fails_double_satisfaction`). I-PROT-VALUE-CONSERVE thêm: $\mathrm{sole}(\mathrm{payee})
   = \mathrm{Some}(V_{\mathrm{esc}})$ đòi đúng một output payee mang trọn value một escrow.
2. *Liên-tx (cùng UTxO):* eUTXO ledger — một UTxO chi được đúng một lần (đã tiêu ⇒ biến mất).
   Hai tx cùng own_ref không thể cùng vào chain.
3. *Liên-claim (cùng claim_id, khác UTxO):* **KHÔNG chặn on-chain** — xem I-PROT-BEACON-ONESHOT.
$\square$

**Điều kiện:** (1)(2) mật mã chắc; (3) hở (F3). "Không chi hai lần cho cùng CLAIM" ⇔ đóng
beacon. "Không chi hai lần cùng UTxO" ⇔ đã đảm bảo.

### Định lý 3 (Premium–payout đồng đơn vị)

**Phát biểu.** Premium (thu) và payout (chi) cùng đơn vị CARP; solvency so sánh được không
đứt gãy thứ nguyên.

**Chứng minh.** Premium định giá nanogic MAGIC nhưng THU bằng CARP với $P^\*=1$ (Whitepaper §5),
nên $\mathrm{premium\_CARP} = \mathrm{premium\_MAGIC}$ về trị số, đơn vị CARP. Nạp pot:
$\mathrm{pot}_{\mathsf{user}} \mathrel{+}= \sum \mathrm{premium\_CARP}$ (CARP). Payout: `expected_amount`
tính trên $L, \mathrm{cap} \in$ CARP; validator ép `escrow_carp == amount` (CARP). Vậy cả hai
vế của I-PROT-SOLVENCY (§8) cùng đơn vị CARP. $\square$

> Đây là điểm SỬA bản cũ (2026-07-01): trước premium coi như MAGIC chi thẳng
> `collectToTreasury(MAGIC)` ⇒ đứt gãy nanogic-vs-lovelace. Nay mọi số một đơn vị CARP.

---

## 8. Bất biến kinh tế — I-PROT-SOLVENCY (off-chain, DAO hiệu chỉnh)

Đo lăn theo cửa sổ $W$, **từng bucket riêng, cùng đơn vị CARP**:

$$
\sum_{W} \mathrm{premium\_CARP} \ \ge\ \mathbb{E}[\mathrm{payout}_{\mathsf{USER},W}] + \mathrm{opex}_{\mathsf{user},W}
\qquad(\text{bucket } \mathrm{pot}_{\mathsf{user}})
$$

$$
\sum_{W} \theta_{\mathrm{treasury}}\!\cdot\!\mathrm{fee\_CARP} \ \ge\ \mathbb{E}[\mathrm{payout}_{\mathsf{SYS},W}] + \mathrm{opex}_{\mathsf{sys},W}
\qquad(\text{bucket } \mathrm{pot}_{\mathsf{sys}})
$$

Vi phạm ⇒ DAO PHẢI hiệu chỉnh ($\uparrow$base_rate / $\downarrow$cap / $\uparrow$copay / $\uparrow$soi)
TRƯỚC khi bucket cạn. **KHÔNG bảo lãnh Treasury vô hạn.** Cùng circuit-breaker DÒNG RA:
I-PROT-POOL-CAP ($\sum a$/cửa sổ $\le \mathrm{POOL\_CAP\_PER\_WINDOW}$) + I-PROT-INCIDENT-CAP
(on-chain, đã ép) + pro-rata khi sự cố vượt pot (C.4).

---

## 9. Giả định tin cậy (ngoài phạm vi chứng minh)

Các định lý §7 chỉ đứng vững DƯỚI các giả thiết sau — auditor phải soi chúng riêng, KHÔNG
coi là đã chứng minh:

1. **Committee trung thực khi đặt $L, c, \sigma, \alpha$ vào datum** (F6). Validator không có
   oracle mật mã cho các số này (§0, §3, C.6). Committee lỏng/đồng lõa ⇒ input phồng.
2. **Treasury nạp escrow từ đúng bucket, mỗi claim đúng một escrow** (F3). Cho tới khi
   I-PROT-BEACON-ONESHOT đóng on-chain, đây là giả định về đường nạp (đội backend), không chứng minh.
3. **CARP `carp_policy`/`carp_name` trỏ đúng asset canonical** của Phoenix Treasury (test dùng
   hằng giả `0xcaca…`). Sai policy-id ⇒ `escrow_carp` đếm sai asset.
4. **Ledger eUTXO đúng** (một UTxO chi một lần; `validity_range.lower_bound ≤ slot`, F5).
5. **Self-theft là ranh giới nền tảng** (C.6): on-chain KHÔNG phân biệt "trộm thật" vs "ví
   thứ hai của mình" (chữ ký y hệt). Phòng vệ = kinh-tế (challenge + phạt) + panel người,
   KHÔNG mật mã. Gian lận LỌT ở mức trong I-PROT-SOLVENCY được chấp nhận có chủ đích.

---

## 10. Checklist cho auditor

- [ ] `policy_domain_ok` chặn $L<0$, $b>10000$, $D<0$ (test F1a/F1b). → I-PROT-DOM.
- [ ] `payout_ok` ép `d.amount == expected_amount(cause,L,π) ∧ d.amount>0 ∧ escrow_carp==d.amount`. → I-PROT-NO-DOUBLE.
- [ ] `d.amount <= cfg.incident_cap` (test over-cap). → I-PROT-INCIDENT-CAP.
- [ ] `sole_output_value_at(payee) == Some(escrow_val)` (test `payout_fails_ada_leak`). → I-PROT-VALUE-CONSERVE.
- [ ] `count_inputs_at(own_cred)==1` payout+reject (test `..double_satisfaction`). → I-PROT-NO-DBL-SAT.
- [ ] `payee ≠ record ≠ own` (test `..cred_collision`). → I-PROT-CRED-DISTINCT.
- [ ] `lo >= approved_slot + t_wait_payout + challenge_window` (test `..before_challenge_window`). → I-PROT-CHALLENGE.
- [ ] `record_ok` ghi `ClaimRecord` khớp (test `..no_record`). → I-PROT-T2.
- [ ] `single_factor_ok`: USER ∧ require_ad ∧ σ ⇒ α (test P8/A4/A4b). → I-PROT-SINGLE-FACTOR.
- [ ] `reject_ok`: NO-CROSS-BUCKET (test `..cross_bucket`) + `committee_ok` dedup `list.unique` (test R1/R2b). → I-PROT-NO-CROSS-BUCKET, I-PROT-COMMITTEE.
- [ ] **SOI KỸ trust-boundary (§5.2):** `protectme_beacon.ak` tồn tại chưa? `beacon_policy` có trong config chưa? Beacon burn ghép vào payout+reject chưa? (nếu chưa → I-PROT-BEACON-ONESHOT HỞ, Định lý 1/2 có điều kiện).
- [ ] Xác nhận `carp_policy`/`carp_name` = CARP canonical thật, KHÔNG hằng test.

---

## 11. 🔴 CHỜ ANH CHỐT — PHẦN D (11 mục) + BLOCKER

Các mục sau **chính sách/kinh tế**, chưa normative — auditor KHÔNG được coi là đã chốt:

- **[PROT-1]** Trần $\mathrm{cap}_{\mathsf{sys}}$ (tới 100% $L$?), $\mathrm{cap}_{\mathsf{user}} \ll$; tỷ lệ SYS:USER; con số tuyệt đối/loại DID.
- **[PROT-2]** Ai phân xử: Security Committee (SYS) + Adjudication Panel (USER); chọn/nhiệm kỳ/ngưỡng vote/recuse-challenge.
- **[PROT-3]** $\mathrm{base\_rate}$, $\theta_{\mathrm{treasury}}$, $g_\bullet$, $b$ (20–50%?), $D$.
- **[PROT-4]** 🔴 Ngưỡng gán $\mathsf{SYS}$ vs $\mathsf{USER}$ (flaw-vs-negligence): tiêu chí cụ thể (reproducer? audit ký? phạm vi $\ge N$ DID?), xử $\mathsf{GREY}$, ai nâng $\mathsf{GREY}\to\mathsf{SYS}$. **Gắn chặt [PROT-10]** vì đây là bề mặt tấn công C.7.
- **[PROT-5]** $T_{\mathrm{report}}$ (30d?), $T_{\mathrm{wait\_activate}}$ (14–30d?), $T_{\mathrm{wp}}+W_{\mathrm{ch}}$ (7–14d?); challenge window RIÊNG cho gán-SYS.
- **[PROT-6]** POOL_CAP_PER_WINDOW ($\le x\%$/tháng?), CAP_INCIDENT_SYS, chính sách pro-rata, điều kiện vay chéo bucket.
- **[PROT-7]** Opt-in vs bundle mặc định (chống adverse selection C.5).
- **[PROT-8]** Subrogation: cơ chế truy hồi + chia phần thu hồi (toàn bộ về pot hay chia với user?).
- **[PROT-9]** Ranh giới KYC/pháp lý cho USER claim lớn (đối trọng "không token-weighted" + riêng tư).
- **[PROT-10]** 🔴 **Evidence-bar gán-SYS + cap-per-incident = VAN POT-WIDE.** Ngưỡng bằng chứng cứng cho nhãn `SYSTEM_FLAW` (reproducer độc lập + postmortem-ký trỏ ĐÚNG lớp + phạm vi khớp chain); BẮT BUỘC soi sơ suất từng-user KỂ CẢ trong batch SYS; $\mathrm{CAP\_INCIDENT\_SYS}$; recuse/challenge RIÊNG cho quyết định gán nhãn. **Một** nhãn SYS sai + batch tự phủ + không-co-pay = rút cạn $\mathrm{pot}_{\mathsf{sys}}$ một nhịp (hợp lưu C.2×C.3×C.4). I-PROT-INCIDENT-CAP on-chain là hàng rào cuối, nhưng ngưỡng gán nhãn nằm off-chain — cần chốt kỹ.
- **[PROT-11]** 🔴 **Cohort single-factor (C.8).** Tiêu chí chứng thực $\mathrm{indep}$ (enclave khác / kênh không-seed); $\mathrm{CAP\_USER\_SINGLE}$; $\ell_{\mathrm{sf}}$; có bắt buộc anti-drain làm điều kiện phủ USER cho DID single-factor không (đề xuất: có; $\rho=\top$ đã cắm code, tunable I-PROT-CFG); cơ chế ĐỌC ranh giới committed-vs-liquid (ScheduleGen/InstantGen) để định exposure theo slice thanh khoản.

**BLOCKER (ngoài quyền quyết của Protectme — chặn normative):**

1. **I-PROT-BEACON-ONESHOT chưa đóng on-chain (F3).** Chống double-payout per `claim_id` hiện
   dựa Treasury nạp không trùng (đội backend). Cần **beacon một lần theo claim_id (state NFT)** để ép
   trên chain — follow-up (`protectme_beacon.ak`, Tech §3). Đây là lỗ auditor phải chốt trước
   khi tuyên "no-double-payout" mật mã.
2. **MAGIC-model:** premium định giá MAGIC phụ thuộc `MAGIC/ConsumeMAGIC` + Feecover
   `service_id="protectme.*"` + `FeecoverEpochSettle` chuyển CARP về bucket. Chưa wire (đội backend).
3. **CARP policy-id:** `carp_policy`/`carp_name` (config) phải trỏ CARP canonical thật của Phoenix
   Treasury. Test dùng hằng giả (`0xcaca…`). Chốt policy-id thật trước deploy.

---

## Phụ lục A — bảng đối chiếu invariant ↔ dòng code

| Invariant | Predicate | File | Test |
|---|---|---|---|
| I-PROT-DOM | `policy_domain_ok` | `protectme_logic.ak` | F1a/F1b |
| I-PROT-NO-DOUBLE | `payout_ok` (amount==expected, >0, escrow==amount) | `protectme_logic.ak` | A2/A6/R-F4 |
| I-PROT-INCIDENT-CAP | `payout_ok` (≤ incident_cap) | `protectme_logic.ak` | over-cap |
| I-PROT-VALUE-CONSERVE | `sole_output_value_at(payee)` | `protectme_logic.ak` | `payout_fails_ada_leak` |
| I-PROT-NO-DBL-SAT | `count_inputs_at==1` | `protectme_logic.ak` | `..double_satisfaction` |
| I-PROT-CRED-DISTINCT | 3 bất đẳng thức cred | `protectme_logic.ak` | `..cred_collision` |
| I-PROT-CHALLENGE | `lo >= approved_slot + t_wait_payout + challenge_window` | `protectme_logic.ak` | `..before_challenge_window` |
| I-PROT-T2 | `record_ok` | `protectme_logic.ak` | `..no_record` |
| I-PROT-SINGLE-FACTOR | `single_factor_ok` | `protectme_logic.ak` | P8/A4/A4b |
| I-PROT-NO-CROSS-BUCKET | `reject_ok` (`sole_output_value_at(bucket_cred)`) | `protectme_logic.ak` | `..cross_bucket` |
| I-PROT-COMMITTEE | `committee_ok` (`list.unique`) | `protectme_logic.ak` | R1/R2b |
| I-PROT-BEACON-ONESHOT 🔴 | (chưa có) `protectme_beacon.ak` | — | — (chặn-merge) |

**Bảng tham số (auditor):**

| Ký hiệu | Code | Tầng ép | Giá trị đề xuất | Trạng thái |
|---|---|---|---|---|
| $\mathrm{cap}_{\mathsf{sys}}$ | `cap_sys` | datum + on-chain min | tới 100% $L$, cap tuyệt đối/DID | **[PROT-1]** |
| $\mathrm{cap}_{\mathsf{user}}$ | `cap_user` | datum + on-chain min | $\ll \mathrm{cap}_{\mathsf{sys}}$ | **[PROT-1]** |
| $b$ (copay) | `copay_bps` | on-chain $[0,10^4]$ | 2000–5000 (20–50%) | **[PROT-3]** |
| $D$ (deductible) | `deductible` | on-chain $\ge 0$ | — | **[PROT-3]** |
| $\mathrm{base\_rate}, \theta_{\mathrm{treasury}}$ | — | Feecover/DAO | 0.5–2%/năm | **[PROT-3]** |
| $T_{\mathrm{wp}}, W_{\mathrm{ch}}$ | `t_wait_payout`, `challenge_window` | on-chain (I-PROT-CHALLENGE) | 7–14 ngày | **[PROT-5]** |
| $T_{\mathrm{report}}, T_{\mathrm{wait\_activate}}$ | — | adjudicate (B.4) | 30 / 14–30 ngày | **[PROT-5]** |
| $\mathrm{CAP}_{\mathrm{inc}}$ | `incident_cap` | on-chain (I-PROT-INCIDENT-CAP) | — | **[PROT-6]** |
| POOL_CAP_PER_WINDOW, CAP_INCIDENT_SYS | — | DAO gate | $\le x\%$ pot/tháng | **[PROT-6/10]** |
| $\rho$ (require_ad) | `require_antidrain_for_single_factor` | on-chain (I-PROT-SINGLE-FACTOR) | $\top$ (CHỐT 2026-07-03) | **[PROT-11]** tunable |
| $\ell_{\mathrm{sf}}, \mathrm{CAP\_USER\_SINGLE}$ | — | premium/policy | $\ge 1$ / $< \mathrm{cap}_{\mathsf{user}}$ | **[PROT-11]** |
| $t$ (committee threshold), $K$ | `committee_threshold`, `committee_vkhs` | on-chain (I-PROT-COMMITTEE) | M-of-N | **[PROT-2]** |

---

## Nguồn

- Nguồn thiết kế nội bộ (không công khai).
- Code: `PhoenixKey-Validator` nhánh `feat/protectme-payout` — `lib/phoenixkey/protectme_logic.ak`,
  `protectme_types.ak`, `validators/protectme_payout.ak`.
- Math canonical đơn vị: [PhoenixKey-Math.md](./PhoenixKey-Math.md). Curve I-CURVE-4/5: [PhoenixKey-Rebirthme-Math.md](./PhoenixKey-Rebirthme-Math.md) + addendum Curve-Routing (CHƯA gộp Math.md).
- Tài liệu cùng bộ: [PhoenixKey-Protectme-Vi-Feat.md](./PhoenixKey-Protectme-Vi-Feat.md), [PhoenixKey-Protectme-Tech.md](./PhoenixKey-Protectme-Tech.md), [PhoenixKey-Protectme-Exec.md](./PhoenixKey-Protectme-Exec.md).

→ Trạng thái & tiến độ hiện tại: [PhoenixKey-STATUS.md](./PhoenixKey-STATUS.md#protectme)

*Hết bản Math. Mọi [N]/[PROT-\*] là đề xuất, normative sau khi maintainer duyệt. Ba định lý §7
đều CÓ ĐIỀU KIỆN trên input datum trung thực (§0, §5.2, §9) — auditor soi ranh giới F3/F6 trước.*

---
_Tài liệu này đã được bảo vệ. Bản quyền © GreenSun Tech Inc. Sáng chế tạm thời USPTO — GS-PHOENIXKEY-01: Application No. 64/031,291._
