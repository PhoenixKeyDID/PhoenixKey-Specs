# PhoenixKey — Protectme — ĐẶC TẢ TOÁN HÌNH THỨC (cho AUDITOR)

> Bản này rút gọn về **phần TOÁN kiểm-được**: ký hiệu, miền, công thức, bảng bất
> biến (kèm ánh xạ tới dòng validator), và định lý có chứng minh phác. Nó là bản
> soi cho auditor — KHÔNG lặp lại phần FEAT/PHẢN-BIỆN (đọc `PhoenixKey-Protectme-Feat-Math.md`
> Phần A/C). Mọi mục **[N]** = đề xuất; chỉ normative sau khi anh Aladin duyệt.
>
> **Nguồn:**
> - Spec: `spec-proposals/PhoenixKey-Protectme-Feat-Math.md` (Phần B toán + bất biến + Phần C phản biện + Phần D chờ chốt).
> - Code đối chiếu: `PhoenixKey-Validator` nhánh `feat/protectme-payout`:
>   `lib/phoenixkey/protectme_logic.ak`, `protectme_types.ak`, `validators/protectme_payout.ak`.
> - Whitepaper §4 (nanogic = byte·ngày), §5 (P\*=1: 1 CARP = 1 MAGIC sức mua, KHÔNG neo fiat).
>
> **CONTRACT đơn vị (bất di):** premium **định-giá MAGIC** (nanogic), **THANH-TOÁN CARP**
> (P\*=1, qua Feecover, KHÔNG `collectToTreasury`). `coverage_cap`, `loss_eligible`,
> `amount` (payout) **cùng đơn vị CARP**. MAGIC chỉ là nhãn giá. Đây là điều kiện tiên
> quyết để bất biến solvency KHÔNG đứt gãy thứ nguyên (bỏ nanogic-vs-lovelace bản cũ).

---

## 0. Trục kiểm của auditor (đọc trước)

Protectme có **hai mặt phẳng** với mức bảo đảm KHÁC hẳn nhau — auditor phải soi riêng:

| Mặt phẳng | Bảo đảm | Ai gác | Trạng thái |
|---|---|---|---|
| **Phía CHI** (payout/reject escrow) | **On-chain, mật mã** — validator ép công thức + trần + challenge + T2 + bảo toàn giá trị | `protectme_payout.ak` | ✅ code + test pass |
| **Phía DUYỆT** (loss_eligible, cause, single_factor flags) | **Off-chain, phán người** (committee/panel) — KHÔNG có oracle mật mã (C.6) | committee off-chain | trust-boundary F3/F6 |
| **Phía NẠP** (escrow = đúng bucket, một-lần-per-claim) | **Treasury release gate** (Long) — validator này KHÔNG ép uniqueness | Treasury (ngoài PR) | 🔴 F3 hở on-chain (xem §5.2) |

**Kết luận toán học chỉ mạnh tới đâu input được chứng thực.** Validator ép `amount ==
expected_amount(loss, policy, cause)` một cách chắc chắn; nhưng `loss` và `policy` là số
committee ĐẶT vào datum. Định lý §6 đều có dạng "GIẢ SỬ input datum trung thực ⇒ …".
Ranh giới tin cậy F3/F6 là chỗ giả thiết đó có thể vỡ — auditor tập trung ở đó.

---

## 1. Ký hiệu

### 1.1 Miền cơ sở

| Ký hiệu | Miền | Nghĩa |
|---|---|---|
| $s$ | $\mathbb{N}$ (SlotNo) | slot Cardano |
| $e$ | $\mathbb{N}$ (EpochNo) | epoch |
| CARP | $\mathbb{N}$ | đơn-vị dàn xếp (một `accepted_asset` Phoenix Treasury); mọi cap/loss/payout |
| MAGIC | $\mathbb{N}$ (nanogic $=10^9$) | đơn-vị-kế-toán/sức mua; nhãn giá premium; $P^\*=1$ nên $x$ MAGIC $\equiv x$ CARP |
| $\mathrm{did}$ | ByteArray | định danh DID |

### 1.2 Nguyên nhân (Cause)

$$\mathcal{C} \triangleq \{\mathsf{SYS},\ \mathsf{USER},\ \mathsf{GREY}\}$$

- $\mathsf{SYS}$ = SYSTEM_FLAW (lỗi hệ thống, trách nhiệm protocol).
- $\mathsf{USER}$ = USER_NEGLIGENCE (sơ suất user).
- $\mathsf{GREY}$ = chưa gán ⇒ **xử như $\mathsf{USER}$** (bảo thủ cho pot).

> Code (`protectme_types.ak`) chỉ mã hoá `Cause = SystemFlaw | UserNegligence` — $\mathsf{GREY}$
> giải quyết ở tầng triage off-chain (B.4) rồi hạ về `UserNegligence` trước khi tạo escrow.
> Auditor lưu: on-chain KHÔNG có $\mathsf{GREY}$; nó không phải trạng thái chi-trả.

### 1.3 Policy (ảnh chụp lúc duyệt) — `PolicySnapshot`

$$
\pi \triangleq \big(\ \mathrm{cap}_{\mathsf{sys}},\ \mathrm{cap}_{\mathsf{user}},\ b\ (\text{copay bps}),\ D\ (\text{deductible}),\ \sigma\ (\text{single\_factor}),\ \alpha\ (\text{anti\_drain\_enrolled})\ \big)
$$

với $\mathrm{cap}_{\mathsf{sys}}, \mathrm{cap}_{\mathsf{user}}, D \in \mathbb{N}$ (CARP), $b \in [0, 10000] \cap \mathbb{Z}$ (basis points), $\sigma, \alpha \in \{\bot, \top\}$.

### 1.4 Claim (escrow datum) — `ClaimDatum`

$$
d \triangleq \big(\ \mathrm{claim\_id},\ \mathrm{did},\ L\ (\text{loss\_eligible}),\ c \in \mathcal{C},\ s_{\mathrm{app}}\ (\text{approved\_slot}),\ a\ (\text{amount}),\ \pi,\ \mathrm{payee},\ \mathrm{rec}\ \big)
$$

$L, a \in \mathbb{Z}$ (CARP; miền hợp lệ ép ở PROT-DOM); $\mathrm{payee}, \mathrm{rec}$ = script credential.

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

## 2. Công thức Premium (định-giá MAGIC, thu CARP)

Phía THU **không** ở validator payout (thu qua Feecover, đối soát cuối epoch); ghi ở đây
làm phần định-giá + bất biến solvency mà auditor kinh-tế soi.

### 2.1 Hệ số rủi ro

$$
\mathrm{rmult}(\pi) = 1.0 \cdot g_{\mathrm{ad}} \cdot g_{\mathrm{gu}} \cdot g_{\mathrm{fr}} \cdot \mathrm{ncb}(\text{streak}) \cdot \mathrm{pen}_{\mathrm{bk}}
$$

trong đó (🔴 **PROT-FACTOR-INDEPENDENCE**, C.8 — chiết khấu CHỈ khi factor độc-lập-với-seed):

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

Không chứng thực được $\mathrm{indep}$ ⇒ **mặc định $\bot$** (bảo thủ). Đây là chỗ Protectme
định-giá TRỰC TIẾP I-CURVE-5 ("second-factor phải khác seed") thành tham số phí.

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

## 3. Công thức Coverage (loss_eligible) — chống-trùng ví-theo-DID

$$
\boxed{\ L(\mathrm{did},\mathrm{inc}) = \underbrace{\sum_{\mathrm{tx}\in\text{theft}(\mathrm{inc})} W(\mathrm{tx})}_{\text{net rời ví-theo-DID}} - R_{\mathrm{cancel}} - R_{\mathrm{freeze}} - R_{\mathrm{vault}}\ }
$$

- $W(\mathrm{tx})$ = **net-rời-kho** — TÁI DÙNG ĐÚNG định nghĩa Withdrawal-Limit (đọc chain, không cãi).
- $R_{\mathrm{cancel}}$ = phần `PendingLargeWithdrawal` chủ đã Cancel (I-LIMIT-CANCEL).
- $R_{\mathrm{freeze}}$ = phần bị Freeze chặn kịp (Frozen→safe-address).
- $R_{\mathrm{vault}}$ = UTxO còn tại `vault_addr(did)` sau rotate/recover (= KHÔNG mất).

Nếu $c = \mathsf{USER}$: trừ thêm $D$ (deductible) ở bước amount (§4), KHÔNG ở đây.

> **Auditor lưu (F3/F6):** $L$ là số **committee chứng thực** đưa vào datum. Validator
> KHÔNG tự tính $L$ từ chain (nó không đọc anti-drain log). Nó chỉ ép $L \ge 0$ (PROT-DOM)
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

> Namespace **PROT-\*** (đổi từ `I-PM-*` — engine MAGIC-Paymaster đã chiếm `PM-*`). Cột
> "Code" trỏ predicate/dòng trong `protectme_logic.ak` (nhánh `feat/protectme-payout`).

### 5.1 Bất biến ĐÃ ép on-chain (validator gác được)

| ID | Phát biểu hình thức | Code (ánh xạ) |
|---|---|---|
| **PROT-DOM** | $L \ge 0 \wedge \mathrm{cap}_{\mathsf{sys}}\!\ge\!0 \wedge \mathrm{cap}_{\mathsf{user}}\!\ge\!0 \wedge D\!\ge\!0 \wedge 0\le b\le 10000$ | `policy_domain_ok` |
| **PROT-NO-DOUBLE** | $a = \mathrm{expected}(c, L, \pi)\ \wedge\ a>0\ \wedge\ \mathrm{carp}(V_{\mathrm{esc}}) = a$ | `payout_ok`: `d.amount == expected`, `d.amount > 0`, `escrow_carp == d.amount` |
| **PROT-INCIDENT-CAP** | $a \le \mathrm{CAP}_{\mathrm{inc}}$ | `payout_ok`: `d.amount <= cfg.incident_cap` |
| **PROT-VALUE-CONSERVE** (HIGH3) | $\mathrm{sole}(\mathrm{payee}) = \mathrm{Some}(V_{\mathrm{esc}})$ — **TOÀN BỘ** value (ADA+CARP+token) về đúng 1 payee, không skim | `payout_ok`: `sole_output_value_at(tx.outputs, d.payee_cred) == Some(escrow_val)` |
| **PROT-NO-DBL-SAT** (CRIT1) | $\#\mathrm{in}(\mathrm{own\_cred}) = 1$ — đúng một escrow input | `count_inputs_at(tx, own_cred) == 1` (cả payout + reject) |
| **PROT-CRED-DISTINCT** (CRIT2) | $\mathrm{payee}\ne\mathrm{rec} \wedge \mathrm{payee}\ne\mathrm{own} \wedge \mathrm{rec}\ne\mathrm{own}$ | `payout_ok`: 3 bất đẳng thức `d.payee_cred != …` |
| **PROT-CHALLENGE** | $\mathrm{lo}(\mathrm{tx}) \ge s_{\mathrm{app}} + T_{\mathrm{wp}} + W_{\mathrm{ch}}$ | `payout_ok`: `lo >= d.approved_slot + cfg.t_wait_payout + cfg.challenge_window` |
| **PROT-T2** | $\exists!$ output tại $\mathrm{rec}$ mang `ClaimRecord{claim_id, did, a, c}` khớp | `record_ok` |
| **PROT-SINGLE-FACTOR** | $c=\mathsf{USER} \wedge \rho \wedge \sigma \Rightarrow \alpha$ (SYS luôn qua) | `single_factor_ok` |
| **PROT-NO-CROSS-BUCKET** | Reject: $\mathrm{sole}(\mathrm{bucket}(c)) = \mathrm{Some}(V_{\mathrm{esc}})$; $\mathrm{bucket}(\mathsf{SYS})=\mathrm{pot}_{\mathsf{sys}}, \mathrm{bucket}(\mathsf{USER})=\mathrm{pot}_{\mathsf{user}}$ | `reject_ok`: `sole_output_value_at(…, bucket_cred(d.cause,cfg))` |
| **PROT-COMMITTEE** (F7) | Reject: $\big|\{k\in\mathrm{unique}(K): k\in\mathrm{sig}(\mathrm{tx})\}\big| \ge t$ | `committee_ok` (dedup qua `list.unique`) |

### 5.2 Bất biến CHỜ enforce on-chain — trust-boundary (auditor SOI KỸ)

| ID | Phát biểu | Trạng thái | Rủi ro |
|---|---|---|---|
| **PROT-BEACON-ONESHOT** 🔴 | Mỗi $\mathrm{claim\_id}$ được nạp escrow + chi ĐÚNG MỘT LẦN (chống double-payout) | **F3: KHÔNG ép on-chain.** Validator không kiểm uniqueness `claim_id` | Treasury nạp HAI escrow cùng `claim_id` ⇒ **cả hai trả được**. Cần **beacon một-lần theo `claim_id` (state NFT)** — follow-up ngoài PR. Hiện dựa `dao_release_ok` nạp-không-trùng (Long) |
| **PROT-EVIDENCE-ONCHAIN** | $L$ = net rời ví (anti-drain log) đọc từ chain, off-chain chỉ bổ trợ | **F6: committee ĐẶT $L$** vào datum; validator không đọc log | Committee đặt $L$ sai ⇒ $a$ sai theo. Không có oracle mật mã cho "trộm hay không" (C.6) |
| **PROT-FACTOR-INDEP** | $\mathrm{indep}(f)$ chứng thực trước khi cho chiết khấu | **F6: cờ $\sigma,\alpha$ committee đặt** — tripwire, không chống committee cố sai | Sức mạnh = độ trung thực committee |
| **PROT-SOLVENCY** | (kinh tế, §7) | off-chain đo lăn + DAO hiệu chỉnh | Không phải bất biến on-chain |

### 5.3 Bất biến ở tầng luồng (không ở validator escrow này)

PROT-CAUSE-SPLIT, PROT-WAIT-ACTIVATE, PROT-REPORT-WINDOW, PROT-POOL-CAP (tổng/cửa-sổ),
PROT-SUBROGATION, PROT-LAPSE, PROT-TRIAGE-BOUNDARY, PROT-PREMIUM-INCENTIVE — ép ở
adjudicate/DAO-gate/Feecover/Treasury (B.4/B.6/A.3), KHÔNG ở `protectme_payout`. Auditor
soi ở đường nạp (Long) + governance gate, không ở validator escrow.

---

## 6. Định lý (có chứng minh phác)

> Ba định lý CONTRACT yêu cầu. Mỗi cái có dạng **có điều kiện** trên input datum trung
> thực — nêu rõ giả thiết vì đó là ranh giới an toàn thật (§0, §5.2).

### Định lý 1 (Payout ≤ pot) — solvency mỗi giao dịch

**Phát biểu.** Với mọi tx `Payout` qua validator, số CARP chi $\le$ CARP escrow, và
escrow đã được Treasury nạp từ đúng bucket:
$$
a \le \mathrm{carp}(V_{\mathrm{esc}}) \quad\text{và thực tế}\quad a = \mathrm{carp}(V_{\mathrm{esc}}).
$$

**Chứng minh.** PROT-NO-DOUBLE ép `escrow_carp == d.amount`, tức $\mathrm{carp}(V_{\mathrm{esc}}) = a$
(đẳng thức, không chỉ bất đẳng thức). Escrow chỉ tồn tại nếu Treasury đã `release(bucket(c), a)`
từ bucket tương ứng (B.6, ngoài validator). Do đó tổng chi qua các claim $\sum a_i =
\sum \mathrm{carp}(V_{\mathrm{esc},i}) \le \mathrm{pot}(c)$ **miễn là** mỗi escrow trừ pot đúng một lần
lúc nạp. $\square$

**Điều kiện then chốt:** đẳng thức chỉ chặn "chi vượt escrow"; chặn "chi vượt POT" phụ thuộc
**PROT-BEACON-ONESHOT** (mỗi claim nạp một escrow). Nếu F3 hở (nạp trùng), một $\mathrm{claim\_id}$
rút được $2a$ trong khi pot chỉ trừ $a$ ⇒ **định lý VỠ**. → Đây là lỗ auditor phải chốt đóng
(§5.2, blocker beacon).

### Định lý 2 (Không chi hai lần) — no-double-payout

**Phát biểu.** Không tồn tại hai tx `Payout` hợp lệ cùng tiêu một escrow UTxO; và trong
một tx, không escrow nào bị thoả-mãn-kép (double-satisfaction).

**Chứng minh.**
1. *Trong-tx:* PROT-NO-DBL-SAT ép $\#\mathrm{in}(\mathrm{own\_cred}) = 1$. Nếu tx gộp hai escrow
   để dùng chung một output payee (kiểu double-satisfaction), $\#\mathrm{in} = 2 \ne 1$ ⇒ fail
   (test `payout_fails_double_satisfaction`). PROT-VALUE-CONSERVE thêm: $\mathrm{sole}(\mathrm{payee})
   = \mathrm{Some}(V_{\mathrm{esc}})$ đòi đúng một output payee mang trọn value một escrow.
2. *Liên-tx (cùng UTxO):* eUTXO ledger — một UTxO chi được đúng một lần (đã tiêu ⇒ biến mất).
   Hai tx cùng own_ref không thể cùng vào chain.
3. *Liên-claim (cùng claim_id, khác UTxO):* **KHÔNG chặn on-chain** — xem PROT-BEACON-ONESHOT.
$\square$

**Điều kiện:** (1)(2) mật mã chắc; (3) hở (F3). "Không chi hai lần cho cùng CLAIM" ⇔ đóng
beacon. "Không chi hai lần cùng UTxO" ⇔ đã đảm bảo.

### Định lý 3 (Premium–payout đồng đơn vị)

**Phát biểu.** Premium (thu) và payout (chi) cùng đơn vị CARP; solvency so-sánh-được không
đứt gãy thứ nguyên.

**Chứng minh.** Premium định-giá nanogic MAGIC nhưng THU bằng CARP với $P^\*=1$ (Whitepaper §5),
nên $\mathrm{premium\_CARP} = \mathrm{premium\_MAGIC}$ về trị số, đơn vị CARP. Nạp pot:
$\mathrm{pot}_{\mathsf{user}} \mathrel{+}= \sum \mathrm{premium\_CARP}$ (CARP). Payout: `expected_amount`
tính trên $L, \mathrm{cap} \in$ CARP; validator ép `escrow_carp == amount` (CARP). Vậy cả hai
vế của PROT-SOLVENCY (§7) cùng đơn vị CARP. $\square$

> Đây là điểm SỬA bản cũ (2026-07-01): trước premium coi như MAGIC chi thẳng
> `collectToTreasury(MAGIC)` ⇒ đứt gãy nanogic-vs-lovelace. Nay mọi số một đơn vị CARP.

---

## 7. Bất biến kinh tế — PROT-SOLVENCY (off-chain, DAO hiệu chỉnh)

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
PROT-POOL-CAP ($\sum a$/cửa-sổ $\le \mathrm{POOL\_CAP\_PER\_WINDOW}$) + PROT-INCIDENT-CAP
(on-chain, đã ép) + pro-rata khi sự cố vượt pot (C.4).

---

## 8. Tham số (bảng auditor)

| Ký hiệu | Code | Tầng ép | Giá trị đề xuất | Trạng thái |
|---|---|---|---|---|
| $\mathrm{cap}_{\mathsf{sys}}$ | `cap_sys` | datum + on-chain min | tới 100% $L$, cap tuyệt đối/DID | **[PROT-1]** |
| $\mathrm{cap}_{\mathsf{user}}$ | `cap_user` | datum + on-chain min | $\ll \mathrm{cap}_{\mathsf{sys}}$ | **[PROT-1]** |
| $b$ (copay) | `copay_bps` | on-chain $[0,10^4]$ | 2000–5000 (20–50%) | **[PROT-3]** |
| $D$ (deductible) | `deductible` | on-chain $\ge 0$ | — | **[PROT-3]** |
| $\mathrm{base\_rate}, \theta_{\mathrm{treasury}}$ | — | Feecover/DAO | 0.5–2%/năm | **[PROT-3]** |
| $T_{\mathrm{wp}}, W_{\mathrm{ch}}$ | `t_wait_payout`, `challenge_window` | on-chain (PROT-CHALLENGE) | 7–14 ngày | **[PROT-5]** |
| $T_{\mathrm{report}}, T_{\mathrm{wait\_activate}}$ | — | adjudicate (B.4) | 30 / 14–30 ngày | **[PROT-5]** |
| $\mathrm{CAP}_{\mathrm{inc}}$ | `incident_cap` | on-chain (PROT-INCIDENT-CAP) | — | **[PROT-6]** |
| POOL_CAP_PER_WINDOW, CAP_INCIDENT_SYS | — | DAO gate | $\le x\%$ pot/tháng | **[PROT-6/10]** |
| $\rho$ (require_ad) | `require_antidrain_for_single_factor` | on-chain (PROT-SINGLE-FACTOR) | $\top$ (CHỐT 2026-07-03) | **[PROT-11]** tunable |
| $\ell_{\mathrm{sf}}, \mathrm{CAP\_USER\_SINGLE}$ | — | premium/policy | $\ge 1$ / $< \mathrm{cap}_{\mathsf{user}}$ | **[PROT-11]** |
| $t$ (committee threshold), $K$ | `committee_threshold`, `committee_vkhs` | on-chain (PROT-COMMITTEE) | M-of-N | **[PROT-2]** |

---

## 9. 🔴 CHỜ ANH CHỐT — PHẦN D (11 mục) + BLOCKER

Các mục sau **chính sách/kinh tế**, chưa normative — auditor KHÔNG được coi là đã chốt:

- **[PROT-1]** Trần $\mathrm{cap}_{\mathsf{sys}}$ (tới 100% $L$?), $\mathrm{cap}_{\mathsf{user}} \ll$; tỷ lệ SYS:USER; con số tuyệt đối/loại DID.
- **[PROT-2]** Ai phân xử: Security Committee (SYS) + Adjudication Panel (USER); chọn/nhiệm kỳ/ngưỡng vote/recuse-challenge.
- **[PROT-3]** $\mathrm{base\_rate}$, $\theta_{\mathrm{treasury}}$, $g_\bullet$, $b$ (20–50%?), $D$.
- **[PROT-4]** 🔴 Ngưỡng gán $\mathsf{SYS}$ vs $\mathsf{USER}$ (flaw-vs-negligence): tiêu chí cụ thể (reproducer? audit ký? phạm vi $\ge N$ DID?), xử $\mathsf{GREY}$, ai nâng $\mathsf{GREY}\to\mathsf{SYS}$. **Gắn chặt [PROT-10]** vì đây là bề mặt tấn công C.7.
- **[PROT-5]** $T_{\mathrm{report}}$ (30d?), $T_{\mathrm{wait\_activate}}$ (14–30d?), $T_{\mathrm{wp}}+W_{\mathrm{ch}}$ (7–14d?); challenge window RIÊNG cho gán-SYS.
- **[PROT-6]** POOL_CAP_PER_WINDOW ($\le x\%$/tháng?), CAP_INCIDENT_SYS, chính sách pro-rata, điều kiện vay-chéo bucket.
- **[PROT-7]** Opt-in vs bundle mặc định (chống adverse selection C.5).
- **[PROT-8]** Subrogation: cơ chế truy hồi + chia phần thu hồi (toàn bộ về pot hay chia với user?).
- **[PROT-9]** Ranh giới KYC/pháp lý cho USER claim lớn (đối trọng "không token-weighted" + riêng tư).
- **[PROT-10]** 🔴 **Evidence-bar gán-SYS + cap-per-incident = VAN POT-WIDE.** Ngưỡng bằng chứng cứng cho nhãn `SYSTEM_FLAW` (reproducer độc lập + postmortem-ký trỏ ĐÚNG lớp + phạm vi khớp chain); BẮT BUỘC soi-sơ-suất-từng-user KỂ CẢ trong batch SYS; $\mathrm{CAP\_INCIDENT\_SYS}$; recuse/challenge RIÊNG cho quyết-định-gán-nhãn. **Một** nhãn SYS sai + batch tự-phủ + không-co-pay = rút cạn $\mathrm{pot}_{\mathsf{sys}}$ một nhịp (hợp lưu C.2×C.3×C.4). PROT-INCIDENT-CAP on-chain là hàng rào cuối, nhưng ngưỡng gán-nhãn nằm off-chain — cần chốt kỹ.
- **[PROT-11]** 🔴 **Cohort single-factor (C.8).** Tiêu chí chứng thực $\mathrm{indep}$ (enclave khác / kênh không-seed); $\mathrm{CAP\_USER\_SINGLE}$; $\ell_{\mathrm{sf}}$; có bắt buộc anti-drain làm điều kiện phủ USER cho DID single-factor không (đề xuất: có; $\rho=\top$ đã cắm code, tunable I-PROT-CFG); cơ chế ĐỌC ranh giới committed-vs-liquid (ScheduleGen/InstantGen) để định exposure theo slice thanh khoản.

**BLOCKER (ngoài quyền quyết của Protectme — chặn normative):**

1. **PROT-BEACON-ONESHOT chưa đóng on-chain (F3).** Chống double-payout per `claim_id` hiện
   dựa Treasury nạp-không-trùng (Long). Cần **beacon một-lần theo claim_id (state NFT)** để ép
   trên chain — follow-up. Đây là lỗ auditor phải chốt trước khi tuyên "no-double-payout" mật mã.
2. **MAGIC-model:** premium định-giá MAGIC phụ thuộc `MAGIC-paymaster/ConsumeMAGIC` + Feecover
   `service_id="protectme.*"` + `FeecoverEpochSettle` chuyển CARP về bucket. Chưa wire (Long).
3. **CARP policy-id:** `carp_policy`/`carp_name` (config) phải trỏ CARP canonical thật của Phoenix
   Treasury. Test dùng hằng giả (`0xcaca…`). Chốt policy-id thật trước deploy.

---

*Hết bản Math. Mọi [N]/[PROT-\*] là đề xuất, normative sau khi anh Aladin duyệt. Ba định lý §6
đều CÓ ĐIỀU KIỆN trên input datum trung thực (§0, §5.2) — auditor soi ranh giới F3/F6 trước.*
