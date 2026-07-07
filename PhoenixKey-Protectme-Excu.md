# PhoenixKey — Protectme — ĐẶC TẢ ĐIỀU HÀNH (Excu)

> Bản tóm cho lãnh đạo. Chi tiết kỹ thuật + Math: `PhoenixKey-Protectme-Feat-Math.md`.
> Bản này KHÔNG khoe sẵn sàng. Nó nêu rõ 11 quyết định + 3 blocker phải chốt TRƯỚC production.

---

## 1. Tóm tắt một trang

**Protectme là gì.** Lớp bồi hoàn cuối cùng cho tài sản BỊ ĐÁNH CẮP — chỉ phủ **phần dư**
còn lại sau khi ba lớp on-chain đã tự cứu: ví-theo-DID (rotate/recover kéo tài sản về),
anti-drain (chặn rút sạch + Freeze), guardian (khôi phục khi mất seed). Nó KHÔNG phải bảo
hiểm mất giá thị trường, chuyển nhầm, hay quên khoá.

**Vì sao chỉ phủ phần dư.** Phủ toàn bộ = rủi ro đạo đức trực tiếp (user hết cẩn thận vì
"được đền") + kỳ vọng chi trả lớn → pot sập. Phủ đúng phần on-chain KHÔNG tự cứu được →
kỳ vọng chi trả nhỏ → premium khả thi → pot bền.

**Hai nguyên nhân, xử KHÁC nhau.**
- **SYSTEM_FLAW** (lỗi protocol/app/validator): trách nhiệm hệ thống, phủ MẠNH (tới 100%,
  không co-pay), chi từ bucket `protectme_pot_system`. Không rủi ro đạo đức (user không tự
  tạo bug) — nhưng cái GÁN NHÃN SYS lại là van pot-wide nguy hiểm nhất (xem §3, C.7).
- **USER_NEGLIGENCE** (phishing, cẩu thả khoá): trách nhiệm user, phủ HẠN CHẾ (trần thấp +
  co-pay 20–50% + deductible + soi kỹ), chi từ bucket `protectme_pot_user`.

**Tiền chảy thế nào.** Premium **định giá bằng MAGIC, thanh toán bằng CARP** qua Feecover
(cùng đường phí hiện có, KHÔNG `collectToTreasury`). Cuối epoch `FeecoverEpochSettle` chuyển
CARP về đúng bucket Treasury. Payout trả bằng CARP, cùng đơn vị — bỏ đứt gãy thứ nguyên.
Hai bucket tách vật lý, KHÔNG vay chéo (một loại claim không ăn cạn quỹ loại kia).

**Kiểm soát chi.** Mọi CARP ra khỏi pot có một claim-record on-chain đối ứng (bảo toàn giá
trị Treasury), qua **cổng release DAO** + **beacon one-shot** chống chi trùng. Lượng mất neo
vào **bằng chứng on-chain** (anti-drain log = oracle-of-truth cho "mất bao nhiêu").

**Giới hạn thành thật (không giấu).** On-chain trả lời được "mất bao nhiêu, đi đâu" nhưng
KHÔNG trả lời được "có phải TRỘM không" — chữ ký chủ và chữ ký kẻ trộm (khoá đã lộ) trông y
hệt. Self-theft (tự dàn cảnh) là ranh giới **không có oracle mật mã**; chỉ phòng vệ được bằng
kinh tế (challenge + phạt) + panel người + heuristic. Pot HỮU HẠN: sự cố cực lớn phủ pro-rata,
KHÔNG hứa 100% mọi lúc.

---

## 2. Bảng quyết định (đã chốt trong spec) — 4 trục

| Quyết định | Lý do (a dài hạn · b first-principles · c tối ưu · d user+bền) | Đánh đổi |
|---|---|---|
| Chỉ phủ **phần dư** (sau ví-theo-DID + anti-drain + guardian) | a niềm tin "tài sản được bảo vệ" là điểm bán LAMP · b phủ đúng phần on-chain không tự cứu · c tái dùng 3 lớp sẵn có · d kỳ vọng chi nhỏ → pot bền | Không phủ toàn bộ → user phải hiểu "phần đã tự an toàn không cần đền" |
| Tách **SYSTEM_FLAW vs USER_NEGLIGENCE** (2 bucket) | a giữ niềm tin protocol · b hai nguyên nhân khích lệ ngược nhau · c hai bucket = tách vật lý không vay chéo · d co-pay giữ "da trong cuộc chơi" | Ca GREY xử bảo thủ về USER → có thể oan nhẹ, user kháng lên SYS được |
| **Beacon one-shot** chống double-payout | a chi trả kiểm soát được · b một claim = một lần chi, không tái · c on-chain enforce · d không cho lời gấp đôi | Cần validator hỗ trợ (Validator #16, đang chặn-merge) |
| Premium **risk-based** (không phẳng) | a pool bền · b định giá theo rủi ro thật · c giảm phí khi bật phòng ngự · d chống adverse selection | Định giá phức tạp hơn phẳng; cần dữ liệu tỷ lệ chi trả để hiệu chỉnh |
| Premium **MAGIC-định-giá / CARP-trả** qua Feecover | a neo sức mua · b MAGIC = đơn vị kế toán, CARP = dàn xếp · c dùng nguyên đường Feecover · d thu tự động qua Vault | Phụ thuộc MAGIC-model + CARP policy-id đã chốt (BLOCKER §4) |
| Chiết khấu phí CHỈ cho factor **độc lập với seed** | a không bán ảo tưởng nhiều lớp · b I-CURVE-5: second-factor phải khác seed · c biến khẩu hiệu thành tham số phí · d không rút ngầm pot qua phòng-vệ-ảo | Solo một-thiết-bị mất chiết khấu → phí cao hơn (đúng rủi ro) |

---

## 3. Ma trận rủi ro (nhấn VAN POT-WIDE)

| Rủi ro | Mức | Bản chất | Phòng vệ | Còn hở |
|---|---|---|---|---|
| 🔴 **Gán nhãn SYS lỏng = van pot-wide** (C.7) | **NGHIÊM TRỌNG** | Một phán quyết SYS sai được KHUẾCH ĐẠI toàn-pot: batch tự-phủ mọi DID "dính". Hợp lưu self-theft × thông đồng committee × drain | Evidence bar cứng (reproducer độc lập + postmortem-ký + phạm vi khớp chain) · vẫn soi từng-user trong batch · per-incident cap + pro-rata · challenge riêng cho nhãn | **CHỜ CHỐT [PROT-10]** — chưa có ngưỡng số. KHÔNG bật SYS-batch tới khi chốt |
| 🔴 **Cohort single-factor** (C.8) | CAO | Solo một-thiết-bị: guardian/secondary cùng seed sập ĐỒNG THỜI khi lộ seed → chiết khấu cho phòng-vệ-ảo → rút ngầm pot_user | Định giá theo độc-lập không theo hiện-diện · bắt buộc anti-drain làm điều kiện phủ · trần thấp + loading · đẩy về cam kết bất-khả-huỷ | **CHỜ CHỐT [PROT-11]** — tiêu chí chứng thực độc lập chưa chốt |
| **Self-theft / claim gian lận** (C.2, C.6) | CAO | On-chain không phân biệt "trộm thật" vs "ví thứ hai của mình" | Challenge window + phạt nặng · cluster heuristic · đối chứng tín hiệu khôi phục · panel người | Chấp nhận gian lận LỌT ở mức trong PROTECT-SOLVENCY, không diệt tuyệt |
| **Rủi ro đạo đức** (C.1) | TRUNG | User cẩu thả vì "có bảo hiểm" | Co-pay + deductible + trần thấp USER + premium tăng khi tắt phòng ngự + no-claim bonus | Chỉ giảm biên, không diệt — lý do USER phủ hạn chế |
| **Rút cạn pot** (C.4) | TRUNG | Sự cố lớn / làn sóng claim vượt pot | POOL_CAP_PER_WINDOW circuit-breaker · no-cross-bucket · pro-rata · SYS batch biết exposure sớm | Pot hữu hạn — sự cố cực lớn phủ pro-rata, phải nói rõ user |
| **Adverse selection** (C.5) | TRUNG | Chỉ user rủi ro cao mua → xoáy tử thần | Premium risk-based · T_wait_activate · bundle mặc định mức nền · no-claim bonus | Cần pool đủ rộng — nghiêng về bundle mặc định |
| **Thông đồng committee/panel** (C.3) | TRUNG | Người phân xử là bên hưởng lợi | Chọn ngẫu nhiên/luân phiên · vote on-chain VP-PersonDID không token-weighted · recuse · challenge | Rủi ro governance, không kỹ thuật — giảm không diệt |
| **Provenance escrow** (gap chưa chốt) | MỞ | Ranh giới escrow bằng chứng vs Feecover B.6 chưa chốt | — | **GAP** — cần chốt cùng [PROT-6] |

---

## 4. ĐIỀU KIỆN TIÊN QUYẾT / BLOCKER CHẶN PRODUCTION

> ⚠ **KHÔNG bật production tới khi TẤT CẢ dưới đây được chốt.** Đây là bản đề xuất; mọi mục
> [N] chỉ thành normative sau khi anh duyệt. Trạng thái hiện tại: spec merged, CODE CHƯA XONG.

### 4a. 11 quyết định PHẦN D chờ anh chốt (3 🔴 là van pot-wide)

| # | Quyết định | Ưu tiên |
|---|---|---|
| **PROT-10** | 🔴 **Evidence bar gán-SYS + per-incident cap** — ngưỡng bằng chứng cứng cho nhãn SYSTEM_FLAW + bắt buộc soi từng-user trong batch + `CAP_INCIDENT_SYS` + recuse/challenge riêng cho nhãn. **VAN POT-WIDE — chốt kỹ nhất** | 🔴 |
| **PROT-11** | 🔴 **Cohort single-factor** — tiêu chí chứng thực độc-lập guardian/secondary + `CAP_USER_SINGLE` + `single_factor_loading` + có bắt buộc anti-drain làm điều kiện phủ không (đề xuất: CÓ) + đọc ranh giới committed-vs-liquid | 🔴 |
| **PROT-4** | 🔴 **Ngưỡng SYSTEM_FLAW vs USER_NEGLIGENCE** — tiêu chí gán SYS + xử GREY + ai nâng GREY→SYS (gắn chặt PROT-10) | 🔴 |
| PROT-1 | Trần phủ `CAP_SYS` / `CAP_USER` (đơn vị CARP) + tỷ lệ SYS:USER theo loại DID | |
| PROT-2 | Ai phân xử — Security Committee + Adjudication Panel: chọn thế nào, nhiệm kỳ, ngưỡng vote, recuse/challenge | |
| PROT-3 | Tỷ lệ phí — `base_rate` (0.5–2%/năm?), `θ_treasury`, `risk_mult`, `copay_rate` (20–50%?), `deductible` | |
| PROT-5 | Thời gian — `T_report` (30d?), `T_wait_activate` (14–30d?), `T_wait_payout` + challenge (7–14d?), challenge riêng gán-SYS | |
| PROT-6 | Circuit-breaker — `POOL_CAP_PER_WINDOW`, `CAP_INCIDENT_SYS`, chính sách pro-rata, điều kiện vay chéo bucket, ranh giới escrow-vs-Feecover-B.6 | |
| PROT-7 | Opt-in vs bundle mặc định (đề xuất: nền mặc định + cap cao opt-in-có-waiting) | |
| PROT-8 | Subrogation — cơ chế truy hồi + chia phần thu hồi (toàn về pot hay chia user?) | |
| PROT-9 | Ranh giới pháp lý/KYC — claim lớn USER có cần KYC không (cân với không-token-weighted + riêng tư) | |

### 4b. 3 blocker hạ tầng

| Blocker | Trạng thái | Chặn gì |
|---|---|---|
| **MAGIC-model** | Chưa xong (Account-trong-Vault, non-transferable) | Premium định-giá không chạy tới khi Vault BurnBatch + Feecover accrual ổn |
| **CARP policy-id** | Chưa chốt | Payout + pot đơn vị CARP không settle được |
| **Beacon one-shot** (Validator #16, Tuân) | Chặn-merge | Chống double-payout chưa có on-chain enforce |

### 4c. Gap code

- **Beacon one-shot chưa có** — Validator #16 đang chặn-merge. Không có nó, một claim có thể
  chi nhiều lần → phải xong trước production.
- **Ranh giới escrow-vs-Feecover B.6 chưa chốt** — cần chốt cùng [PROT-6].
- CODE Protectme (Long: hai bucket + Feecover wire + resolver claim; UI Core/SuperApp) CHƯA XONG.

---

## 5. Trạng thái + lộ trình

**Trạng thái hiện tại:**
- Spec Feat+Math: **MERGED (#15).**
- Validator payout (beacon one-shot): **Validator #16 — Tuân, đang chặn-merge.**
- CODE: **CHƯA XONG** (backend hai bucket + Feecover + resolver; UI).
- 11 quyết định PHẦN D: **CHỜ ANH** (3 🔴).

**Lộ trình đề xuất:**
1. **Anh chốt 3 🔴 trước** (PROT-10, PROT-11, PROT-4) — chúng khoá bề mặt tấn công van pot-wide.
2. Chốt nốt 8 quyết định còn lại + 3 blocker hạ tầng (MAGIC-model, CARP policy-id, beacon).
3. Long wire hai bucket + Feecover premium + resolver claim (backend PhoenixKey = ranh giới Long).
4. Tuân hoàn tất beacon one-shot (Validator #16) → merge.
5. Core/SuperApp làm UI (bật/tắt, cap, breakdown phí, claim, timeline) — ngôn ngữ KHÔNG hứa 100%.
6. Chỉ bật production khi §4 xanh toàn bộ.

---

## 6. Câu hỏi cần lãnh đạo quyết

1. **Van pot-wide (PROT-10):** anh chấp nhận rủi ro "một nhãn SYS sai" tới mức nào? Đặt
   `CAP_INCIDENT_SYS` và pro-rata bao nhiêu để một nhãn sai KHÔNG thành thảm hoạ pot?
2. **Single-factor (PROT-11):** có bắt buộc anti-drain làm điều kiện phủ USER cho DID
   single-factor không? (Đề xuất: CÓ — time-buffer là lớp thật duy nhất còn lại.)
3. **Phủ USER cao hay thấp (PROT-1/3):** phủ cao hơn = premium cao hơn nhiều + soi gắt hơn.
   Anh muốn nghiêng về niềm-tin-mạnh (đắt, rủi ro đạo đức cao) hay bền-pot (phủ hạn chế)?
4. **Bundle vs opt-in (PROT-7):** bật mức nền mặc định cho MỌI user (chống xoáy tử thần)
   hay opt-in thuần? Ảnh hưởng trực tiếp sống-chết pot.
5. **KYC claim lớn (PROT-9):** đánh đổi giữa chống-gian-lận và không-token-weighted + riêng tư.
6. **Truyền thông:** chấp nhận nói thẳng với user "phủ tới hạn pot, sự cố cực lớn chia
   pro-rata, không 100% mọi lúc"? (Hứa 100% là dối trá — C.4.)

---

*Nguồn: `PhoenixKey-Protectme-Feat-Math.md` (Feat+Math + PHẦN C phản biện + PHẦN D quyết định).
Bản Excu này chỉ tóm cho điều hành — mọi tham số thành normative sau khi anh duyệt.*
