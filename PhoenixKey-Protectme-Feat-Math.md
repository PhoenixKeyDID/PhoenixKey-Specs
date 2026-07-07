# PhoenixKey — Protectme (bồi hoàn tài sản bị đánh cắp) — Feat + Math [N]

> Đặc tả tính năng (WHAT/WHY) + hình thức hoá (Math) cho **Protectme** — cơ chế
> **bồi hoàn** tài sản của user khi bị **đánh cắp**, phủ HAI nguyên nhân với cách xử
> KHÁC nhau:
> (a) **lỗi hệ thống** (lỗ hổng protocol/app) — trách nhiệm thuộc protocol, phủ mạnh;
> (b) **sơ suất user** (phishing, xử lý khoá cẩu thả) — phủ hạn chế, có trần, soi kỹ.
>
> Nguồn quỹ: **hai bucket Phoenix Treasury** dành riêng (`protectme_pot_system`,
> `protectme_pot_user`) + **phí bảo vệ (premium)** do user trả — **định giá bằng MAGIC,
> THANH TOÁN bằng CARP**, thu qua Feecover (cùng đường phí Feecover, `app_id="feecover"`).
>
> Quy trình: **Feat+Math này → anh Aladin duyệt → Long (Treasury/backend) + Tuân
> (validator log) + Core/SuperApp (UI claim).** Phần normative đề xuất nhập
> PhoenixKey-Math.md.
>
> **Mô hình: Triple Token** (LAMP · MAGIC · CARP) — KHÔNG phải Dual (2 token),
> KHÔNG phải one-token native (HopNhat lỗi thời). **Nguồn canonical:
> `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md`** (NGUỒN CHÂN LÝ đơn-vị + peg,
> anh chốt 2026-07-06; `MagicLamp-3Token-DacTa-Vi.md` bản cũ ĐÃ BỊ ĐÈ, không dùng).
>
> 🔴 **SỬA MÔ HÌNH MAGIC (2026-07-01) — đọc trước A.3.** Bản trước ghi premium "tính
> bằng MAGIC" như thể MAGIC là token chi trực tiếp vào pot qua `collectToTreasury(MAGIC)`.
> Điều đó SAI và làm luồng tiền premium/pot **không nối được**. Mô hình đúng (bám
> `MAGIC/SPEC/Whitepaper-MagicLamp-Tokenomic-Vi.md` + `MAGIC-paymaster/ConsumeMAGIC` +
> `PhoenixKey-Feecover-Feat-Math.md`):
> - **MAGIC = Điều Ước = số dư tài khoản trong Vault** (đơn-vị-kế-toán / thước sức mua),
>   **KHÔNG phải native token, KHÔNG policy-id, non-transferable, decay theo thời hạn.**
>   Tiêu MAGIC = `BurnBatch` giảm `current_amount` trong datum Vault, KHÔNG `tx.mint`.
> - **Thanh toán = CARP** (Tấm Thảm — đồng-thanh-khoản chuyển-nhượng-được). MAGIC chỉ
>   định GIÁ. `P*=1`: 1 CARP = 1 MAGIC sức mua (Whitepaper §5, KHÔNG neo fiat). Premium
>   **định giá bằng MAGIC, TRẢ bằng CARP**.
> - **Premium KHÔNG dùng `collectToTreasury`** (đường đó đã bỏ cho phí Feecover — cả MAGIC
>   lẫn CARP, `Feecover §6`). Nó chảy **cùng cơ chế Feecover đã định nghĩa**: user tiêu qua
>   Vault → phí CARP tích (`FeecoverAccrual`) vào **Vault nhà cung cấp** (App→Platform) →
>   cuối epoch **`FeecoverEpochSettle`** đối soát chuyển CARP về bucket Treasury Protectme.
>   Protectme là **consumer** của đường thu Feecover, KHÔNG tự chế đường mới.
> - 🔴 **Đồng nhất đơn vị:** premium VÀ payout PHẢI cùng đơn vị. `coverage_cap`,
>   `loss_eligible`, payout đều tính bằng **CARP** (đơn vị dàn xếp), MAGIC chỉ là đơn vị
>   định giá. Bỏ đứt gãy thứ nguyên nanogic-vs-lovelace trong bất biến solvency cũ.
>
> **Đọc kèm (đã đối chiếu):**
> - `PhoenixKey-Withdrawal-Limit-Anti-Drain-Feat-Math.md` — chống rút sạch qua hạn
>   mức + freeze + `PendingLargeWithdrawal`; anti-drain log là **bằng chứng on-chain**
>   cốt lõi cho phân xử (§C.3).
> - `PhoenixKey-Secret-Vault-and-LampNet-Custody-Feat-Math.md` — kho tài sản/bí mật;
>   ranh giới "chỉ giữ, không tự ký" (I-VAULT-9) định nghĩa mặt phủ.
> - `PhoenixKey-UI-Spec-Backup-Recovery-Lifecycle.md` — mô hình guardian (khoá thành
>   viên ngủ-đông uỷ quyền rotate) + **ví-theo-DID** (địa chỉ script bất biến, tài sản
>   sống sót qua rotate/recover). ← Protectme chỉ phủ **phần dư** sau khi ví-theo-DID
>   đã tự cứu.
> - `LAMP/Treasury/CONTRACT.md` — bucket = **sổ kế toán trong datum** (một bucket = MỘT
>   ô kế toán, KHÔNG có primitive sổ-phụ); release **cổng bởi DAO governance vote**
>   (T1 = Model A); T2/T3 bảo toàn giá trị. Hiện **CHƯA có** pot bồi hoàn → Protectme mở
>   **HAI bucket Treasury MỚI**. (Đường THU của Protectme KHÔNG gọi `collectToTreasury` —
>   xem Feecover; chỉ mượn mô hình bucket + release-gate cho phần CHI.)
> - `PhoenixKey-Feecover-Feat-Math.md` — premium đi **đúng đường thu Feecover**: phí CARP
>   tích `FeecoverAccrual` theo tầng App→Platform, cuối epoch **`FeecoverEpochSettle`** đối
>   soát chuyển CARP về Phoenix Treasury (`app_id="feecover"`; `collectToTreasury` đã BỎ
>   cho phí Feecover, `Feecover §6`). User luôn chỉ tiêu MAGIC (Vault `BurnBatch`), App trả
>   ADA+LAMP hộ. Protectme = một `service_id` trên đường này, KHÔNG tự chế đường thu.
> - `MAGIC-paymaster/ConsumeMAGIC` — MAGIC là số dư Vault tiêu qua giảm datum (không
>   token, không mint value tại app); giá lấy từ beacon có thẩm quyền, không tin client.
> - Memory: `magic-not-native-token` (MAGIC = đơn-vị-kế-toán/sức mua, nanogic=10⁹),
>   `phoenixkey-anchor-economics` (Treasury độc lập, phí cố-định-theo-dịch-vụ + đệm FX),
>   `no-token-weighted governance` (VP = tích nhân ≥4 tham số, cử tri = PersonDID).
>
> **Phạm vi Protectme:** quỹ + phí + luồng claim→bằng-chứng→phân-xử→chi-trả + trần/chờ
> + chống-trùng với ví-theo-DID + tích hợp anti-drain/guardian/Treasury-gate.
> **KHÔNG** đặc tả: tokenomics MAGIC/LAMP (chỉ tham chiếu), giá quy đổi FX (đệm ở
> anchor-economics), luật KYC/pháp lý từng quốc gia (ngoài kỹ thuật).
>
> **Thành thật trước:** pot kiểu-bảo-hiểm CHẾT vì rủi ro đạo đức + gian lận + rút cạn.
> §PHẢN BIỆN (Phần C) là phần QUAN TRỌNG NHẤT của bản này — không có nó, Protectme là
> một máy đốt quỹ. Đọc C trước khi tin A/B.

---

# PHẦN A — FEAT

## A.1 Vấn đề: ví-theo-DID cứu tài sản, KHÔNG cứu tài sản đã RỜI đi

Kiến trúc PhoenixKey đã có ba lớp phòng ngự tài sản, xếp chồng:

1. **Ví-theo-DID** (UI-Spec §9): địa chỉ script bất biến `blake2b(DID)`, chi khi
   controller hiện tại ký. Mất máy / lộ seed → rotate/recover → controller mới chi
   tiếp. **Tài sản Ở TRONG ví-theo-DID sống sót mất khoá.**
2. **Anti-drain** (Withdrawal-Limit): kẻ chiếm khoá controller không rút sạch tức thì;
   trần `limit`/cửa sổ + `PendingLargeWithdrawal` chủ-huỷ-được + Freeze→safe-address.
3. **Guardian recovery**: t-of-n uỷ quyền rotate khi mất seed.

Nhưng có một **khoảng dư** ba lớp trên KHÔNG bịt được:

- Kẻ tấn công **đã rút thành công** một khoản `≤ limit` mỗi cửa sổ trong khoảng "khoá
  đã lộ nhưng chủ chưa Freeze". Tài sản đó đã **rời ví-theo-DID sang địa chỉ kẻ trộm**
  → rotate/recover KHÔNG kéo về được (nó không còn ở địa chỉ script nữa).
- Tài sản **ngoài ví-theo-DID**: ví Standard (seed BIP39), khoá mạng khác trong Kho bí
  mật mà user CHỦ ĐỘNG mở để dùng (I-VAULT-9 ngoại lệ) rồi bị đánh cắp trong lúc mở.
- **Lỗ hổng hệ thống**: nếu chính validator/app có bug cho phép chi không đúng luật,
  ví-theo-DID KHÔNG bảo vệ được — bug ở đúng lớp bảo vệ.

Protectme phủ **phần dư này** — tài sản đã BỊ ĐÁNH CẮP mà các lớp on-chain không tự
kéo về. Đây KHÔNG phải bảo hiểm "mất giá thị trường", KHÔNG phải "chuyển nhầm", KHÔNG
phải "quên khoá" (cái đó guardian recovery lo). Chỉ **trộm cắp** — có kẻ thứ ba chiếm
đoạt.

> **Quyết định (4 trục):** (a) *dài hạn* — niềm tin "tài sản trong PhoenixKey được bảo
> vệ" là điểm bán cốt lõi để LAMP có giá trị; (b) *first-principles* — phủ đúng phần
> mà cơ chế on-chain KHÔNG tự cứu, không phủ trùng; (c) *tối ưu* — tái dùng Treasury
> bucket + Feecover + anti-drain log, không dựng hạ tầng mới; (d) *user + bền vững* —
> quỹ phải TỰ NUÔI (premium ≥ kỳ vọng chi trả) nếu không sẽ sập, nên trần/soi/phí phải
> đặt để pot sống lâu, không phải để marketing.

## A.2 Hai nguyên nhân — vì sao xử KHÁC nhau

Protectme phân biệt trách nhiệm theo **ai gây ra lỗ hổng dẫn tới trộm**:

| Trục | (a) Lỗi hệ thống (SYSTEM_FLAW) | (b) Sơ suất user (USER_NEGLIGENCE) |
|---|---|---|
| Định nghĩa | Trộm do lỗ hổng protocol/app/validator: bug cho chi sai luật, ký nhầm, rò khoá do app, lỗi thư viện | Trộm do hành vi user: phishing, dán seed vào web giả, cấp quyền cho dApp độc, mở khoá mạng khác rồi bị lừa ký |
| Trách nhiệm | **Thuộc protocol** — hệ thống hứa đúng, hệ thống sai | **Thuộc user** — hệ thống làm đúng, user bị lừa |
| Phủ | **Mạnh**: tới trần cao (đề xuất tới 100% giá trị mất, cap tuyệt đối/DID) | **Hạn chế**: trần thấp + đồng-chi-trả (co-pay) + soi kỹ hơn |
| Nguồn chi | Bucket **`protectme_pot_system`** (trách nhiệm chung của protocol) | Bucket **`protectme_pot_user`** (user tự bảo vệ nhau) |
| Ngưỡng bằng chứng | Cần **xác nhận lỗ hổng** (bug reproducible / postmortem / patch) — nếu đúng lỗi hệ thống thì MỌI user dính đều được phủ | Cần bằng chứng trộm + chứng minh KHÔNG cố ý dàn cảnh; soi hành vi |
| Thời gian chờ | Ngắn hơn khi lỗ hổng đã xác nhận công khai (batch) | Dài hơn (waiting period + điều tra cá nhân) |
| Rủi ro đạo đức | Thấp (user không tự tạo được bug hệ thống) | **Cao** (user có thể cẩu thả vì "có bảo hiểm" — xem C.1) |

**Vì sao tách:** hai nguyên nhân có **cấu trúc khích lệ ngược nhau**.
- Lỗi hệ thống là **trách nhiệm protocol** — phủ mạnh để giữ niềm tin, và vì user
  KHÔNG kiểm soát được bug hệ thống nên KHÔNG có rủi ro đạo đức (không ai cố tình tạo
  lỗ hổng validator để claim). Phủ rộng ở đây là **an toàn**.
- Sơ suất user là **trách nhiệm user** — nếu phủ 100% thì tạo rủi ro đạo đức trực tiếp:
  user hết cẩn thận vì "được đền". Phải có **co-pay** (user chịu một phần) + trần thấp
  + soi kỹ để giữ khích lệ tự-bảo-vệ. Đây là nguyên lý bảo hiểm cơ bản: người được bảo
  hiểm phải còn "da trong cuộc chơi".

**Ranh giới xám (quan trọng):** một số ca khó gán — vd app hiển thị địa chỉ nhận sai
do bug UI khiến user gửi nhầm cho kẻ lừa (lỗi hệ thống HAY sơ suất?). Cơ chế phân xử
(B.4) phải **gán nguyên nhân** trước khi tính phủ; ca xám mặc định về USER_NEGLIGENCE
(bảo thủ cho pot), user/committee có thể nâng cấp lên SYSTEM_FLAW nếu chứng minh được
lỗi ở lớp hệ thống.

## A.3 Quỹ: HAI Treasury bucket + premium (Feecover, định-giá-MAGIC / trả-CARP)

Hai nguồn, hai vai — và 🟡 **HAI bucket Treasury tách vật lý**, KHÔNG phải sổ-phụ trong
một bucket. Lý do: `LAMP/Treasury/CONTRACT.md §1` định nghĩa **bucket = một ô kế toán
trong datum**, KHÔNG có primitive sổ-phụ (sub-ledger). Muốn hai lane không-vay-chéo, đo
riêng, chi riêng → phải là **hai bucket** (hai `category`), không nhồi vào một.

### A.3.1 Hai Phoenix Treasury bucket

Mở **hai bucket Treasury MỚI** (`LAMP/Treasury/CONTRACT.md` — bucket tham số hoá; đơn vị
dàn xếp = **CARP**, một `accepted_asset` của Phoenix Treasury):

```
protectme_pot_system   -- nguồn chi cho SYSTEM_FLAW  (trách nhiệm protocol)
protectme_pot_user     -- nguồn chi cho USER_NEGLIGENCE (user tự bảo vệ nhau)
```

- **Nạp `protectme_pot_system`:** (i) một phần **phí dịch vụ PhoenixKey** (anchor fee,
  LampNet metering margin) trích theo tỷ lệ `θ_treasury` do DAO đặt — tích `FeecoverAccrual`
  trong epoch, cuối epoch **`FeecoverEpochSettle`** chuyển CARP về bucket
  `protectme_pot_system` (`app_id="feecover"`); (ii) **thu hồi** (subrogation) từ claim
  SYSTEM_FLAW hoàn về đúng bucket này.
- **Nạp `protectme_pot_user`:** (i) toàn bộ **premium** user trả (A.3.2) — tích
  `FeecoverAccrual` trong epoch, cuối epoch **`FeecoverEpochSettle`** chuyển CARP về bucket
  `protectme_pot_user`; (ii) thu hồi từ claim USER_NEGLIGENCE.
- **Chi ra (cả hai bucket):** chỉ qua **claim đã phân xử** (B.4) + **cổng release DAO**
  (B.6, T1 Model A). KHÔNG có đường chi trực tiếp. Bảo toàn giá trị T2/T3 của Treasury
  được giữ: mọi CARP ra khỏi bucket có một claim-record đối ứng.
- **KHÔNG vay chéo bucket:** cạn `protectme_pot_user` KHÔNG được rút từ
  `protectme_pot_system` (và ngược lại) trừ khi DAO vote riêng (chống một loại claim ăn
  cạn quỹ loại kia — C.4). Đây là hệ quả TRỰC TIẾP của việc dùng hai bucket: tách vật lý
  ⇒ không có đường tràn ngầm giữa hai lane.

> 🟡 **Vì sao KHÔNG một bucket + hai sổ-phụ (sửa bản cũ):** Treasury không có primitive
> sổ-phụ; một bucket là một số dư kế toán duy nhất. "Hai sub-ledger trong một bucket"
> (bản trước) không hiện thực được trên interface Treasury. Hai bucket là cách đúng để
> đạt tách-lane + không-vay-chéo mà vẫn dùng nguyên đường thu Feecover (`FeecoverEpochSettle`)
> + release-gate DAO.

### A.3.2 Premium — Feecover, ĐỊNH GIÁ bằng MAGIC, THANH TOÁN bằng CARP

🔴 **Sửa luồng tiền (2026-07-01).** User bật Protectme = trả **premium** định kỳ
(tháng/năm) **qua Feecover — đúng đường phí Feecover hiện có**, KHÔNG phải một đường chi
riêng. Cụ thể (bám `PhoenixKey-Feecover-Feat-Math.md §6.1`):

```
[kỳ premium]  Feecover coi Protectme như một service_id (vd "protectme.premium")
   │  fee_magic(service_id) = premium(policy)   -- ĐỊNH GIÁ bằng MAGIC (nanogic)
   ▼
[1] user tiêu MAGIC:  Vault BurnBatch giảm current_amount đúng fee_magic
   │                  (MAGIC = số dư datum, KHÔNG token, KHÔNG mint)
   ▼
[2] Feecover-App trả ADA (phí mạng) + LAMP (phí giao thức) hộ; phí CARP thu thật
   │  TÍCH (FeecoverAccrual) vào VAULT NHÀ CUNG CẤP (App→Platform), như mọi phí Feecover khác
   ▼
[3] CUỐI EPOCH — FeecoverEpochSettle đối soát (Feecover §6.2, KHÔNG collectToTreasury):
   │  total_carp = Σ FeecoverAccrual(Platform, closed_epoch) cho service protectme.*
   │  chuyển CARP về Phoenix Treasury, category="protectme_pot_user" (app_id="feecover")
   ▼
[4] protectme_pot_user += Σ premium_CARP   -- pot nạp bằng CARP, cùng đơn vị payout
```

- **KHÔNG `collectToTreasury`** (đường đó đã bỏ cho phí Feecover, `Feecover §6`) — CARP về
  Treasury qua **`FeecoverEpochSettle`**. MAGIC chỉ là **nhãn giá** (`P*=1`: 1 MAGIC sức
  mua = 1 CARP). Cái thực chảy vào pot là **CARP** — cùng đơn vị với payout (A.6/B.5),
  nên bất biến solvency KHÔNG còn đứt gãy thứ nguyên.
- 🟡 **`app_id` nhất quán:** vì premium thu qua Feecover, nó dùng **`app_id="feecover"`**
  (đúng quy ước Feecover). Không đặt `app_id="phoenixkey"` cho nhánh premium — nếu tách
  app_id thì receipt + tín dụng VP/uy tín (`Treasury CONTRACT §3.4`) bị chẻ đôi cho cùng
  một dòng phí. Phân biệt system-vs-user pot bằng **`category`** (bucket), KHÔNG bằng
  app_id.

**Mô hình phí (risk-based, KHÔNG phẳng):** premium phẳng tạo **chọn-lọc-bất-lợi**
(adverse selection — chỉ user rủi ro cao mua, C.5). Premium phải phản ánh rủi ro:

```
premium(user, kỳ) = base_rate × coverage_cap × risk_multiplier(user) × exposure_factor
   -- đơn vị: MAGIC (nanogic) làm nhãn giá; thu bằng CARP qua Feecover (P*=1).
```

- `base_rate` — tỷ lệ nền/đơn-vị-phủ/kỳ (đề xuất khởi điểm 0.5%–2%/năm giá trị phủ,
  DAO đặt, hiệu chỉnh theo tỷ lệ chi trả thực).
- `coverage_cap` — trần phủ user chọn (đơn vị **CARP**; cao hơn = premium cao hơn, tuyến
  tính).
- `risk_multiplier(user)` — **GIẢM** nếu user bật các lớp phòng ngự (khích lệ tự-bảo-vệ,
  đối trọng rủi ro đạo đức):
  - anti-drain bật + `limit` chặt → ×0.7
  - guardian ≥ 2-of-3 → ×0.8
  - Freeze safe-address đã đặt → ×0.9
  - lịch sử KHÔNG claim (no-claim bonus, tích luỹ) → giảm dần
  - Ngược lại: chưa bật backup (quá grace §5 UI-Spec), hành vi rủi ro cao → **TĂNG**.
- `exposure_factor` — theo giá trị tài sản thực trong ví-theo-DID (phủ nhiều → trả nhiều).

**Vì sao Feecover (định-giá-MAGIC/trả-CARP):** premium là **dịch vụ định kỳ neo sức mua**,
đúng vai MAGIC đơn-vị-kế-toán (anchor-economics: phí cố-định-theo-dịch-vụ + đệm FX);
CARP là phương tiện dàn xếp (`P*=1`). Feecover lo thu định kỳ tự động qua Vault burn +
đối soát; ngắt Feecover (Vault cạn MAGIC / lapse) = mất phủ ở kỳ kế (grace ngắn để chống
lapse do lỗi thanh toán).

> **Bất biến quỹ (PROTECT-SOLVENCY):** hai bucket phải **tự nuôi kỳ vọng** (đo riêng
> từng lane, cùng đơn vị CARP) — Σ premium_CARP (vào `pot_user`) ≥ E[payout USER] + opex;
> Σ θ_treasury×fee_CARP (vào `pot_system`) ≥ E[payout SYS] + opex. Nếu tỷ lệ chi trả
> vượt ngưỡng, DAO PHẢI hiệu chỉnh (tăng base_rate / giảm cap / siết soi) trước khi pot
> cạn. Không có bảo lãnh vô hạn từ Treasury (C.4).

## A.4 Luồng: claim → bằng chứng → phân xử → chi trả (tổng quan)

```
[1] TRIGGER   user phát hiện trộm → mở claim (trong cửa sổ báo cáo, A.5)
      │
[2] EVIDENCE  gom bằng chứng on-chain (anti-drain log, tx trộm, guardian signal)
      │        + off-chain (báo cáo, ảnh phishing…) → đóng gói ClaimPacket
      │
[3] TRIAGE    tự động gán nguyên nhân sơ bộ (SYSTEM_FLAW | USER_NEGLIGENCE | xám)
      │        + kiểm chống-trùng ví-theo-DID (tài sản đã sống sót ⇒ loại trừ, A.6)
      │
[4] ADJUDICATE  phân xử (B.4): committee kỹ thuật (SYSTEM_FLAW) / panel (USER_NEG)
      │           → quyết định phủ + số tiền + nguyên nhân chốt
      │
[5] WAITING     hết waiting period (A.5) + cửa sổ phản-tố (challenge window)
      │
[6] PAYOUT      DAO release gate mở pot → chi từ đúng bucket → ghi claim-record
      │           on-chain (bảo toàn T2) → cập nhật no-claim / risk_multiplier
      │
[7] SUBROGATION nếu về sau truy được tài sản kẻ trộm → thu hồi về pot
```

Chi tiết ai-phân-xử, bằng-chứng-gì, trần/chờ ở B.4–B.6 và A.5–A.6.

## A.5 Trần phủ, thời gian chờ, cửa sổ báo cáo

- **Cửa sổ báo cáo (report window):** claim phải mở trong `T_report` slot kể từ tx
  trộm (đề xuất 30 ngày). Quá hạn → không phủ (chống claim mốc, ép user cảnh giác).
- **Waiting period sau khi BẬT:** phủ chỉ hiệu lực sau `T_wait_activate` (đề xuất 14–30
  ngày) kể từ lần bật/tăng cap đầu — chống mua-bảo-hiểm-sau-khi-đã-bị-trộm và chống
  bật-cap-cao-ngay-trước-claim (C.5 adverse selection / C.2 gian lận).
- **Waiting period trước CHI:** sau khi phân xử duyệt, chờ `T_wait_payout` +
  **challenge window** (đề xuất 7–14 ngày) để ai phát hiện gian lận phản-tố trước khi
  tiền ra (C.2).
- **Trần phủ (coverage cap) — đơn vị CARP** (cùng đơn vị `loss_eligible` + payout):
  - **SYSTEM_FLAW:** trần cao — đề xuất tới `CAP_SYS` = min(giá trị thực mất, cap tuyệt
    đối/DID do DAO đặt). Vì trách nhiệm protocol, có thể tới 100% giá trị dư đã mất.
  - **USER_NEGLIGENCE:** trần thấp + **co-pay** — chi trả = `min(loss, CAP_USER) × (1 −
    copay_rate)`, `copay_rate` đề xuất 20–50% (user luôn chịu một phần → giữ khích lệ).
    `CAP_USER` ≪ `CAP_SYS`.
  - **Trần pool tổng/kỳ:** tổng chi trả toàn hệ / cửa sổ ≤ một phần pot (đề xuất ≤ x%
    pot/tháng) → một sự cố lớn KHÔNG rút cạn pot trong một nhịp (C.4, giống trần
    per-window của anti-drain — cùng triết lý circuit-breaker cho DÒNG RA của pot).
- **Đồng-chi-trả + khấu trừ (deductible):** USER_NEGLIGENCE có deductible (một mức mất
  tối thiểu user tự chịu, chống claim vặt), rồi co-pay phần trên.

## A.6 Chống trùng với ví-theo-DID (không bồi hoàn phần đã tự sống sót)

🔴 **Cốt tử.** Ví-theo-DID đã cứu phần lớn tài sản qua rotate/recover — Protectme
**chỉ phủ phần DƯ đã thực sự rời khỏi tầm kiểm soát**. Không phủ trùng.

`loss_eligible` quy về **CARP** (đơn vị dàn xếp) — giá trị on-chain rời ví (ADA/token)
quy đổi sang CARP tại thời điểm incident theo cùng bảng quy đổi Feecover/anchor-economics,
để premium, cap và payout đồng một đơn vị (bỏ đứt gãy thứ nguyên):

```
loss_eligible(DID, incident)   -- CARP
  = (giá trị ĐÃ RỜI ví-theo-DID sang địa chỉ ngoài kiểm soát của DID)
  − (giá trị đã/được thu hồi qua rotate/recover/Freeze/Cancel-large)
  − (giá trị vẫn còn trong ví-theo-DID sau incident — KHÔNG mất)
  − deductible (nếu USER_NEGLIGENCE)
```

- Tài sản **còn trong ví-theo-DID** sau incident (script address bất biến, controller
  mới chi được) = **KHÔNG mất** → 0 phủ. Kiểm bằng: quét UTxO tại `vault_addr(DID)`
  sau rotate/recover, phần còn đó trừ khỏi loss.
- Phần kẻ trộm mở `PendingLargeWithdrawal` nhưng **chủ đã Cancel** (I-LIMIT-CANCEL) =
  KHÔNG mất → 0 phủ.
- Phần bị **Freeze chặn lại** kịp = KHÔNG mất → 0 phủ.
- Chỉ phần **thực sự đã confirm rời sang địa chỉ kẻ trộm và không kéo về được** mới vào
  `loss_eligible`. Đây là điểm neo bằng chứng on-chain (anti-drain log, B.4) — không
  cãi được vì đọc thẳng chain.

→ Protectme là **lớp cuối**, phủ đúng lỗ hổng còn lại sau ba lớp on-chain. Điều này vừa
đúng nguyên tắc không-phủ-trùng, vừa **giảm mạnh kỳ vọng chi trả** (phần lớn tài sản đã
được on-chain tự cứu) → pot bền hơn.

## A.7 Vì sao thuyết phục (định tính)

- **Phủ đúng phần dư, không phủ trùng** → kỳ vọng chi trả nhỏ (ba lớp on-chain đã cứu
  phần lớn) → premium khả thi, pot bền.
- **Tách hai nguyên nhân theo trách nhiệm** → giữ khích lệ tự-bảo-vệ cho user
  (co-pay/trần thấp cho sơ suất) trong khi vẫn giữ niềm tin protocol (phủ mạnh cho lỗi
  hệ thống, nơi không có rủi ro đạo đức).
- **Premium giảm khi bật phòng ngự** → Protectme đẩy user bật anti-drain/guardian, tăng
  an toàn TOÀN HỆ, giảm claim → vòng lặp lành mạnh.
- **Cổng DAO + bằng chứng on-chain** → chi trả có kiểm soát, không chi tuỳ tiện; anti-
  drain log làm oracle-of-truth cho "đã bị đánh cắp bao nhiêu".

---

# PHẦN B — MATH [N]

## B.1 Kiểu dữ liệu Protectme

```
Cause ≜ SYSTEM_FLAW | USER_NEGLIGENCE | GREY        -- GREY = chưa gán, mặc định về USER

CoveragePolicy ≜ {                                   -- hợp đồng phủ của một DID
  did              : ByteArray,
  active_since     : SlotNo,                          -- mốc bật (tính T_wait_activate)
  coverage_cap_sys : ℕ,                               -- trần SYSTEM_FLAW (CARP)
  coverage_cap_user: ℕ,                               -- trần USER_NEGLIGENCE (CARP, ≪ sys)
  copay_rate_bps   : ℕ,                               -- đồng-chi-trả USER (basis points)
  deductible       : ℕ,                               -- khấu trừ USER (CARP)
  premium_per_epoch: ℕ,                               -- ĐỊNH GIÁ nanogic MAGIC / kỳ; TRẢ bằng CARP (P*=1, A.3.2)
  last_paid_epoch  : EpochNo,                          -- kỳ premium gần nhất đã trả (Feecover)
  no_claim_streak  : ℕ,                                -- kỳ không claim (giảm risk_mult)
  defense_flags    : { anti_drain, guardian, freeze_addr : Bool },  -- risk_multiplier
  schema_version   : Int,
}

ClaimPacket ≜ {                                       -- một yêu cầu bồi hoàn
  claim_id         : ByteArray,
  did              : ByteArray,
  incident_slot    : SlotNo,                           -- slot tx trộm sớm nhất
  opened_slot      : SlotNo,                           -- slot mở claim (≤ incident+T_report)
  claimed_loss     : ℕ,                                -- user khai (sẽ đối chiếu on-chain)
  cause_claimed    : Cause,
  onchain_evidence : { theft_txids, antidrain_log_ref, guardian_signals, vault_addr },
  offchain_evidence: Hash,                             -- CID/hash gói bằng chứng off-chain
  status           : ClaimStatus,
}

ClaimStatus ≜ Opened | Triaged | Adjudicating | Approved{amount,cause}
            | Rejected{reason} | Challenged | Paid{amount} | Subrogated{recovered}
```

## B.2 Quỹ (HAI bucket Treasury, đơn vị CARP)

🟡 Hai **bucket** Treasury tách vật lý (KHÔNG sổ-phụ — Treasury không có primitive đó,
`CONTRACT §1`). Nạp qua **`FeecoverEpochSettle`** (KHÔNG `collectToTreasury` — `Feecover §6`);
`app_id="feecover"` cho cả hai; phân biệt bằng `category`:

```
protectme_pot_system ≜ bucket (app_id="feecover", category="protectme_pot_system") : ℕ CARP
protectme_pot_user   ≜ bucket (app_id="feecover", category="protectme_pot_user")   : ℕ CARP

-- Nạp (tất cả bằng CARP, qua FeecoverEpochSettle cuối epoch — KHÔNG collectToTreasury; Feecover §6.2):
settle_service_fee(fee_CARP) ⟹ protectme_pot_system += θ_treasury × fee_CARP  -- θ∈(0,1), DAO
settle_premium(p_CARP)       ⟹ protectme_pot_user   += p_CARP                 -- p = premium quy CARP
subrogate(r_CARP, cause)      ⟹ bucket(cause) += r_CARP                        -- thu hồi hoàn đúng bucket

-- Chi (chỉ qua claim đã duyệt + DAO gate, B.6):
payout(amount, cause) ⟹ REQUIRE dao_release_ok(amount, cause)              -- amount tính CARP
                       ∧ bucket(cause) ≥ amount
                       ∧ bucket(cause) -= amount
   bucket(SYSTEM_FLAW) = protectme_pot_system ; bucket(USER_NEGLIGENCE) = protectme_pot_user
```

**Không vay chéo bucket** trừ khi `dao_cross_bucket_ok` (một vote riêng) — tách vật lý
hai bucket ⇒ không có đường tràn ngầm; một loại claim không ăn cạn quỹ loại kia (C.4).

## B.3 Premium (risk-based)

```
risk_mult(policy) ≜  Π các hệ số theo defense_flags + no_claim
   base 1.0
   × (anti_drain ? 0.7 : 1.0)
   × (guardian   ? 0.8 : 1.0)
   × (freeze_addr? 0.9 : 1.0)
   × no_claim_discount(no_claim_streak)          -- giảm dần, sàn ở một mức
   × backup_penalty(policy)                        -- >1 nếu quá grace chưa backup (§5 UI)
   -- 🔴 C.8: hệ số guardian(×0.8)/freeze-secondary(×0.9) CHỈ áp khi factor ĐỘC LẬP với seed
   --         (PROTECT-FACTOR-INDEPENDENCE). Cùng seed / cùng thiết bị ⇒ ×1.0 — nếu không sẽ
   --         trả chiết khấu cho lớp phòng vệ ẢO (I-CURVE-5). single_factor_loading ≥ 1 nếu
   --         KHÔNG có factor độc lập nào (C.8).

premium(policy) ≜ base_rate
                × (policy.coverage_cap_sys + policy.coverage_cap_user)   -- exposure (CARP)
                × risk_mult(policy)
   -- ĐỊNH GIÁ: nanogic MAGIC / kỳ ; THU: CARP qua Feecover (P*=1) + đối soát cuối epoch.

-- Bất biến solvency (đo lăn theo cửa sổ W, TỪNG bucket riêng, cùng đơn vị CARP):
PROTECT-SOLVENCY:
   Σ_{W} premium_CARP        ≥ E[payout_USER_W] + opex_user_W        -- bucket pot_user
   Σ_{W} θ_treasury×fee_CARP ≥ E[payout_SYS_W]  + opex_sys_W         -- bucket pot_system
   vi phạm ⇒ DAO PHẢI hiệu chỉnh (↑base_rate / ↓cap / ↑copay / ↑soi) trước khi bucket cạn.
```

## B.4 Phân xử — ai, bằng chứng gì

**Gán nguyên nhân (triage tự động trước, người sau):**

```
triage_cause(claim) ⟹
  IF ∃ xác nhận lỗ hổng hệ thống áp cho incident (bug reproducible / postmortem đã ký /
        patch tham chiếu đúng lớp bị khai thác)
     THEN SYSTEM_FLAW
  ELSE IF bằng chứng chỉ ra hành vi user (phishing sig, cấp quyền dApp, mở khoá tự dùng)
     THEN USER_NEGLIGENCE
  ELSE GREY  -- mặc định xử như USER_NEGLIGENCE (bảo thủ cho pot); user có thể kháng lên SYS
```

**Hai hội đồng, hai ngưỡng bằng chứng:**

- **SYSTEM_FLAW → Security Committee (kỹ thuật):** một hội đồng nhỏ kỹ thuật + audit,
  phán "có phải lỗ hổng hệ thống không". Nếu ĐÚNG → **áp CHUNG cho MỌI DID dính cùng
  lỗ hổng** (không phân xử từng cái) → nhanh, công bằng, batch. Bằng chứng: reproducer,
  diff patch, phạm vi ảnh hưởng on-chain.
- **USER_NEGLIGENCE → Adjudication Panel + DAO:** panel (có thể gồm đại diện cộng đồng
  qua VP-PersonDID, KHÔNG token-weighted — memory governance) xét từng ca. Bằng chứng
  BẮT BUỘC on-chain (không cãi được):
  - **anti-drain log** (Withdrawal-Limit B.4/B.7): meter UTxO sequence cho thấy chính
    xác `W(tx)` đã rời ví-theo-DID mỗi tx, khi nào Freeze, có Cancel không → **oracle-
    of-truth cho "mất bao nhiêu"** (C.6).
  - **guardian signals**: có InitRecovery / Freeze / Cancel không, thời điểm → xác định
    user đã phản ứng hay bỏ mặc.
  - **vault_addr UTxO scan**: phần còn trong ví-theo-DID (không mất, A.6).
  - off-chain (báo cáo phishing, ảnh) chỉ **bổ trợ**, không thay on-chain.

**Quyết định:**

```
adjudicate(claim) ⟹
  cause  = final cause (committee/panel chốt, có thể nâng GREY→SYS nếu chứng minh)
  loss   = loss_eligible(claim.did, incident)                       -- A.6, đọc on-chain
  cap    = (cause=SYSTEM_FLAW ? policy.coverage_cap_sys : policy.coverage_cap_user)
  gross  = min(loss, cap)
  amount = (cause=USER_NEGLIGENCE
              ? max(0, (gross − deductible)) × (1 − copay_rate)
              : gross)                                              -- SYS: không co-pay
  REQUIRE  opened_slot ≤ incident_slot + T_report                   -- cửa sổ báo cáo
         ∧ incident_slot ≥ policy.active_since + T_wait_activate     -- đã qua waiting bật
         ∧ premium đã trả đủ tới incident (Feecover, không lapse)
  → status = Approved{amount, cause}
```

## B.5 Chống-trùng ví-theo-DID (hình thức)

```
loss_eligible(did, inc) ≜
   Σ_{tx ∈ theft_txs(inc)} W(tx)                    -- net rời ví-theo-DID (Withdrawal-Limit B.5)
 − recovered_via_cancel(inc)                         -- PendingLargeWithdrawal đã Cancel
 − blocked_via_freeze(inc)                           -- chặn bởi Frozen→safe-address
 − still_in_vault_after(inc)                          -- UTxO còn tại vault_addr(did) sau recover
 (− deductible nếu USER_NEGLIGENCE, áp ở B.4)

PROTECT-NO-DOUBLE:  amount_paid(claim) ≤ loss_eligible − (mọi giá trị đã kéo về on-chain).
   Tài sản còn trong ví-theo-DID hoặc đã Cancel/Freeze ⇒ KHÔNG vào loss_eligible.
```

`W(tx)` tái dùng ĐÚNG định nghĩa net-rời-kho của Withdrawal-Limit (đọc chain, không
cãi) → oracle-of-truth cho lượng mất.

## B.6 Cổng chi trả (DAO release gate) + T2

```
dao_release_ok(amount, cause) ⟺
     claim.status = Approved{amount, cause}
   ∧ now ≥ approved_slot + T_wait_payout                       -- chờ trước chi
   ∧ challenge_window đã đóng ∧ không có phản-tố thắng          -- C.2
   ∧ amount + paid_in_window ≤ POOL_CAP_PER_WINDOW             -- trần dòng-ra pot (C.4)
   ∧ ( cause=SYSTEM_FLAW  → security_committee vote pass
     ∨ cause=USER_NEGLIGENCE → panel/DAO vote pass )            -- cổng governance (LAMP/Treasury)

payout ⟹ Treasury.release(bucket(cause), amount, claim_id)      -- amount tính CARP
   ∧ ghi claim-record on-chain (claim_id, amount, cause, did) — đối ứng T2 (mọi CARP
     ra có record) → bảo toàn giá trị Treasury.
   bucket(SYSTEM_FLAW)=protectme_pot_system ; bucket(USER_NEGLIGENCE)=protectme_pot_user
```

## B.7 Bất biến [N]

> 🔴 **Đổi namespace (2026-07-01):** bất biến Protectme đổi `I-PM-*` → **`PROTECT-*`**,
> quyết định `[PM-*]` → **`[PROT-*]`**. Lý do: engine **MAGIC-Paymaster** (ConsumeMAGIC —
> mà Feecover đặt lên trên) ĐÃ chiếm `PM-1..PM-12`. Giữ `I-PM-*` là **đè namespace** một hệ
> khác → nhầm bất biến khi cross-ref. Tất cả đơn vị chi/thu = **CARP**.

- **PROTECT-NO-DOUBLE:** không bồi hoàn phần tài sản đã sống sót qua ví-theo-DID / Cancel
  / Freeze; chỉ phủ `loss_eligible` = net thực rời khỏi kiểm soát DID (B.5).
- **PROTECT-CAUSE-SPLIT:** phủ + nguồn chi phụ thuộc `cause`. SYSTEM_FLAW: trần cao, không
  co-pay, chi từ **bucket `protectme_pot_system`**. USER_NEGLIGENCE: trần thấp, deductible
  + co-pay, chi từ **bucket `protectme_pot_user`**. GREY xử như USER (bảo thủ) tới khi
  chứng minh SYS.
- **PROTECT-SOLVENCY:** MỖI bucket tự-nuôi-kỳ-vọng (premium_CARP ≥ E[payout USER]+opex;
  θ×fee_CARP ≥ E[payout SYS]+opex; đo lăn); vượt ngưỡng → DAO PHẢI hiệu chỉnh trước khi
  cạn; KHÔNG bảo lãnh Treasury vô hạn.
- **PROTECT-POOL-CAP:** tổng chi trả / cửa sổ ≤ `POOL_CAP_PER_WINDOW` (circuit-breaker
  dòng ra pot, đối xứng I-LIMIT-CAP của anti-drain); một sự cố lớn không rút cạn pot một
  nhịp.
- **PROTECT-NO-CROSS-BUCKET:** `protectme_pot_system`/`protectme_pot_user` là hai bucket
  tách vật lý, không vay chéo trừ DAO vote riêng.
- **PROTECT-WAIT-ACTIVATE:** phủ chỉ hiệu lực sau `T_wait_activate` kể từ bật/tăng-cap →
  chống mua-sau-khi-bị-trộm + adverse selection phút chót.
- **PROTECT-REPORT-WINDOW:** claim phải mở trong `T_report` từ incident; quá hạn không phủ.
- **PROTECT-CHALLENGE:** chi trả chỉ sau `T_wait_payout` + challenge window đóng không có
  phản-tố thắng → có thời gian phát hiện gian lận trước khi tiền ra.
- **PROTECT-EVIDENCE-ONCHAIN:** lượng mất `loss_eligible` neo vào bằng chứng ON-CHAIN
  (anti-drain log + vault UTxO scan); off-chain chỉ bổ trợ, không thay.
- **PROTECT-PREMIUM-INCENTIVE:** premium GIẢM khi bật phòng ngự (anti-drain/guardian/
  freeze) + no-claim; TĂNG khi rủi ro cao/chưa backup → đẩy user tự-bảo-vệ (đối trọng
  rủi ro đạo đức C.1).
- **PROTECT-SUBROGATION:** tài sản truy hồi được sau chi trả hoàn về đúng bucket →
  không cho user lời gấp đôi (đền + tự lấy lại).
- **PROTECT-T2:** mọi CARP rời bucket có claim-record on-chain đối ứng (bảo toàn giá trị
  Treasury T2); không có đường chi trực tiếp ngoài claim đã duyệt + DAO gate.
- **PROTECT-LAPSE:** premium lapse (Vault cạn MAGIC / Feecover ngắt quá grace) → mất phủ ở
  kỳ kế; incident trong lúc lapse không phủ.
- **PROTECT-TRIAGE-BOUNDARY:** ranh giới phân loại SYSTEM_FLAW/USER_NEGLIGENCE là bề mặt
  tấn công (C.7); ép ngưỡng-bằng-chứng-SYS + soi-sơ-suất-từng-user-KỂ-CẢ-trong-batch-SYS
  + committee challenge/recuse + trần/incident (C.7).
- **PROTECT-FACTOR-INDEPENDENCE:** chiết khấu `risk_mult` cho guardian/secondary CHỈ áp khi
  factor được chứng thực ĐỘC LẬP với seed chính (enclave khác / kênh không-seed). Cùng seed
  / cùng thiết bị ⇒ không chiết khấu; không chứng thực được ⇒ mặc định không độc lập (C.8).
  Không có điều này thì I-CURVE-5 bị né bằng phòng-vệ-ảo và pot_user bị rút ngầm.
- **PROTECT-SINGLE-FACTOR:** DID không có second-factor độc lập chỉ được phủ USER_NEGLIGENCE
  khi anti-drain bật + `limit` chặt; không anti-drain ⇒ chỉ phủ SYSTEM_FLAW. `CAP_USER_SINGLE
  < CAP_USER` + `single_factor_loading` premium; exposure bó tự nhiên theo rate-cap anti-drain
  (siết limit = tăng bảo vệ VÀ giảm exposure cùng lúc). Phủ theo slice thanh khoản, không tổng
  số dư (phần cam kết bất-khả-huỷ tự bảo vệ, exposure 0) (C.8).
  > 🟢 **CHỐT (anh, chế-độ-thử 2026-07-03):** anti-drain là **ĐIỀU KIỆN BẮT BUỘC** để phủ
  > USER_NEGLIGENCE cho DID single-factor. **Điều chỉnh lại sau được** — quy tắc đọc từ
  > **config governance-tunable** (mẫu `ValidatorParams`: `require_antidrain_for_single_factor:
  > Bool` + ngưỡng), KHÔNG hardcode hằng số on-chain. Nới/siết = DAO update config; policy cũ
  > grandfather theo `schema_version`. Đây là lý do validator payout tham-số-hoá rule này ngay
  > từ đầu (I-PROT-CFG).

---

# PHẦN C — PHẢN BIỆN ĐỐI KHÁNG (tự rà — QUAN TRỌNG NHẤT)

> Pot kiểu-bảo-hiểm là mục tiêu tấn công tự nhiên. Nếu không giải thoả đáng các mục
> dưới, Protectme là **máy đốt quỹ**. Thành thật: một vài mục KHÔNG có lời giải kỹ
> thuật sạch — chỉ có đánh đổi + hàng rào kinh tế + phán người. Nêu rõ, không giấu.

## C.1 Rủi ro đạo đức (moral hazard) — user cẩu thả vì "có bảo hiểm"

**Mối đe doạ:** biết được đền, user hết cẩn thận — dán seed bừa, cấp quyền dApp lạ,
tắt anti-drain. Đây là **thất bại kinh điển của bảo hiểm**.

**Phòng vệ (nhiều lớp, không lớp nào đủ một mình):**
- **Co-pay + deductible** (USER_NEGLIGENCE): user LUÔN chịu một phần → còn "da trong
  cuộc chơi". `copay_rate` 20–50% là hàng rào chính.
- **Premium risk-based**: tắt phòng ngự → premium TĂNG (backup_penalty, thiếu defense
  flags) → cẩu thả bị đánh phí trước, không chỉ sau khi mất.
- **Trần thấp cho sơ suất**: `CAP_USER ≪ CAP_SYS` → không đền đủ để "yên tâm cẩu thả".
- **No-claim bonus**: claim → mất chiết khấu → premium kỳ sau tăng → chi phí dài hạn
  của cẩu thả cao hơn khoản đền một lần.

**Thành thật:** không lớp nào diệt được rủi ro đạo đức, chỉ **giảm biên**. Đây là lý do
USER_NEGLIGENCE phải phủ HẠN CHẾ — không phải keo kiệt, mà để hệ thống sống. Nếu anh
muốn phủ USER cao hơn, phải chấp nhận premium cao hơn nhiều + soi gắt hơn.

## C.2 Claim gian lận — tự dàn cảnh "bị đánh cắp"

**Mối đe doạ:** user tự chuyển tài sản sang địa chỉ thứ hai của CHÍNH mình, khai "bị
trộm", claim, rồi tiêu cả tài sản (vẫn ở địa chỉ kia) lẫn tiền đền. Đây là tấn công
NGUY HIỂM NHẤT.

**Phòng vệ:**
- **Bằng chứng on-chain là oracle** (PROTECT-EVIDENCE-ONCHAIN): "đã rời ví-theo-DID" thì đọc được,
  NHƯNG "rời sang địa chỉ kẻ trộm thật hay ví thứ hai của mình" thì **on-chain KHÔNG
  phân biệt được** — đây là lỗ hổng cốt lõi (xem C.6).
- **Challenge window + waiting**: cho thời gian điều tra trước khi tiền ra; ai chứng
  minh địa chỉ nhận thuộc claimant → bác claim + phạt (đốt no-claim, cấm phủ).
- **Cluster analysis / heuristic**: địa chỉ nhận có liên hệ lịch sử với claimant (từng
  nhận từ cùng nguồn, cùng dApp KYC…) → cờ đỏ, nâng ngưỡng soi. KHÔNG tự động bác (có
  thể oan) nhưng chuyển sang điều tra người.
- **Đối chứng khôi phục**: user thật bị trộm sẽ có tín hiệu phản ứng (Freeze, InitRecovery,
  báo cáo sớm). User dàn cảnh thường KHÔNG có phản ứng thật (vì "kẻ trộm" là chính họ)
  → thiếu guardian signal = cờ đỏ.
- **Phạt gian lận nặng**: claim gian lận bị bắt → cấm Protectme vĩnh viễn + mất mọi
  premium đã trả + (nếu KYC) truy cứu.

**Thành thật:** self-theft là ranh giới **on-chain không giải được**. Phòng vệ mạnh nhất
là kinh tế (challenge + phạt) + điều tra người + heuristic cluster — KHÔNG phải bằng
chứng mật mã. Đây là lý do USER_NEGLIGENCE cần panel người, không tự-động-chi. Chấp nhận
sẽ có gian lận LỌT; thiết kế để tỷ lệ + quy mô nằm trong PROTECT-SOLVENCY, không phải để
diệt tuyệt đối.

## C.3 Thông đồng (collusion)

**Mối đe doạ:** (i) claimant + "kẻ trộm" thông đồng chia tiền đền; (ii) claimant +
thành viên committee/panel; (iii) guardian thông đồng dàn cảnh khôi phục giả để hợp
thức "user đã phản ứng".

**Phòng vệ:**
- (i) = biến thể self-theft (C.2): cùng phòng vệ cluster + challenge + phạt.
- (ii) **committee/panel không được là bên hưởng lợi**: chọn ngẫu nhiên/luân phiên,
  công khai vote on-chain (VP-PersonDID không token-weighted → khó mua phiếu bằng tiền),
  bên bị ảnh hưởng recuse; challenge window cho cộng đồng phản-tố quyết định của panel.
- (iii) guardian thông đồng: mô hình guardian đã có **veto/threshold** + cấu hình vai
  (UI-Spec §16 — người-xác-nhận-còn-sống chống thông đồng); Protectme KHÔNG coi guardian
  signal là bằng chứng ĐỦ, chỉ là một tín hiệu trong bộ.

**Thành thật:** thông đồng với người phân xử là rủi ro governance, không phải kỹ thuật.
Giảm bằng minh bạch + phân tán + challenge, không diệt tuyệt.

## C.4 Rút cạn pot (pot drain)

**Mối đe doạ:** (i) một sự cố lớn (lỗ hổng hệ thống ảnh hưởng vạn user) đòi chi vượt
pot; (ii) làn sóng claim gian lận đồng loạt; (iii) một loại claim ăn cạn quỹ loại kia.

**Phòng vệ:**
- **PROTECT-POOL-CAP**: trần dòng-ra pot/cửa sổ (circuit-breaker, cùng triết lý anti-drain)
  → chi trả GIÃN ra nhiều cửa sổ nếu sự cố lớn, pot không cạn một nhịp; user lớn nhận
  theo lịch, không tức thì toàn bộ.
- **PROTECT-NO-CROSS-BUCKET**: hai bucket system/user tách vật lý → claim USER không rút
  cạn quỹ SYS và ngược lại.
- **PROTECT-SOLVENCY** + hiệu chỉnh DAO: theo dõi tỷ lệ chi trả từng bucket, siết trước khi cạn.
- **SYSTEM_FLAW batch**: sự cố hệ thống lớn xử CHUNG → biết tổng exposure sớm, DAO quyết
  phân bổ (pro-rata nếu vượt pot) — công bằng hơn ai-claim-trước-được-trước.

**Thành thật:** pot HỮU HẠN. Nếu lỗ hổng hệ thống thảm hoạ vượt cả pot + Treasury, phủ
sẽ **pro-rata** (chia theo tỷ lệ), KHÔNG 100%. Phải nói rõ với user từ đầu: "phủ tới
hạn pot, sự cố cực lớn có thể chia tỷ lệ". Hứa 100%-mọi-lúc là dối trá.

## C.5 Chọn-lọc-bất-lợi (adverse selection)

**Mối đe doạ:** chỉ user rủi ro cao (biết mình sắp bị/đang bị nhắm) mua Protectme →
pool toàn rủi ro cao → premium phải rất cao → user rủi ro thấp bỏ đi → xoáy tử thần.

**Phòng vệ:**
- **Premium risk-based** (B.3): định giá theo rủi ro thật, không phẳng → user rủi ro
  cao trả đúng chi phí của họ, không trợ giá bởi người thấp.
- **T_wait_activate**: không thể mua-ngay-trước-khi-claim; bật cap cao cũng chịu waiting
  → không thể "thấy sắp bị trộm mới bật".
- **Bundle mặc định**: bật Protectme mức nền cho MỌI user như default an toàn (như
  guardian template) → pool rộng, trộn rủi ro, không chỉ người-rủi-ro-cao tự chọn.
- **No-claim bonus**: thưởng user rủi ro thấp ở lại → giữ pool cân bằng.

**Thành thật:** adverse selection chỉ kiểm soát được nếu **pool đủ rộng + định giá đúng
rủi ro + waiting đủ dài**. Nếu Protectme để opt-in thuần cho người-lo-lắng, nó sẽ xoáy
tử thần. Đề xuất: mức nền BẬT MẶC ĐỊNH (bundle), cap cao là opt-in có waiting.

## C.6 Oracle-of-truth cho "đã bị đánh cắp"

**Đây là vấn đề gốc.** On-chain trả lời được **"tài sản đã rời ví-theo-DID bao nhiêu,
đi đâu, khi nào"** (anti-drain log — chắc chắn, không cãi). Nhưng KHÔNG trả lời được
**"việc rời đó có phải TRỘM không"** — chuyển hợp lệ do chính chủ ký và chuyển do kẻ
trộm ký (khoá đã lộ) **trông y hệt trên chain**. Chữ ký hợp lệ ở cả hai ca.

**Hệ quả thẳng thắn:**
- "Mất bao nhiêu" = **on-chain, chắc chắn** (PROTECT-EVIDENCE-ONCHAIN).
- "Có phải trộm không" + "nguyên nhân gì" = **KHÔNG có oracle mật mã** → phải phán
  người (committee/panel) + heuristic + bằng chứng off-chain + kinh tế (challenge/phạt).
- Đây là lý do:
  - waiting + challenge window (thời gian phát hiện),
  - co-pay + trần thấp (giảm động cơ dàn cảnh),
  - panel người cho USER_NEGLIGENCE (không tự-động-chi),
  - phủ MẠNH cho SYSTEM_FLAW (nơi CÓ oracle khách quan hơn: bug reproducible + patch,
    không phụ thuộc lời khai user).

**Chốt thẳng:** Protectme mạnh nhất ở SYSTEM_FLAW (có bằng chứng khách quan) và yếu nhất
ở USER_NEGLIGENCE self-theft (không có oracle). Thiết kế phải nghiêng nguồn-lực-phủ về
nơi bằng chứng khách quan, và **giới hạn** nơi chỉ có lời khai. Bất kỳ ai hứa "AI/oracle
tự động phân biệt trộm-hay-không" đang bán ảo tưởng.

## C.7 🔴 Ranh giới phân loại SYSTEM_FLAW vs USER_NEGLIGENCE — TỰ NÓ là bề mặt tấn công

**Đây là lỗ hổng bản A/B bỏ sót — và nó nguy hiểm ngang C.2.** Cả thiết kế xoay quanh
một tiền đề: SYSTEM_FLAW phủ MẠNH vì "không có rủi ro đạo đức — user không tự tạo được
bug hệ thống". Đúng về mặt *tạo bug*. Nhưng phủ SYSTEM_FLAW còn có bốn thuộc tính khiến
**cái GÁN nhãn SYS** trở thành chiến lợi phẩm:

- **Trần cao / tới 100%** (`CAP_SYS ≫ CAP_USER`).
- **KHÔNG co-pay, KHÔNG deductible** (SYS: `amount = gross`, B.4).
- **Batch tự-phủ MỌI DID dính** cùng lỗ hổng — không phân xử từng cái.
- **Soi từng-user YẾU** trong batch (giả định "đã là lỗi hệ thống thì mọi nạn nhân đều
  vô can").

⇒ **Nước đi của kẻ tấn công KHÔNG phải tạo bug — mà là làm cho một vụ self-theft (C.2)
được TÁI PHÂN LOẠI thành SYSTEM_FLAW.** Khi đó nó nhảy từ "trần thấp + co-pay + panel
soi kỹ" sang "trần cao + không co-pay + batch tự-duyệt". Và tệ hơn: **một phán quyết
sai/lỏng lẻo của Security Committee được KHUẾCH ĐẠI toàn-pot** — một lần gán SYS cho một
"lỗ hổng" ngụy tạo mở đường phủ batch cho *mọi* DID được dựng lên để trông như nạn nhân.
Đây là điểm hợp lưu C.2 (self-theft) × C.3 (thông đồng committee) × C.4 (drain), tấn công
vào **đúng cái van** giữ cho C.1/C.2 an toàn.

**Ba biến thể cụ thể:**
1. **Ngụy tạo lỗ hổng:** dựng một "bug reproducible" mỏng (edge case vô hại thổi thành
   "cho chi sai luật") để committee gán SYS, rồi claim self-theft dưới nhãn đó.
2. **Ăn theo lỗ hổng thật:** khi có SYS thật + batch mở, chèn các DID **thực ra bị sơ
   suất** (hoặc self-theft) vào danh sách "cũng dính" để hưởng phủ cao không-co-pay.
3. **Bắt cóc committee:** một-hai thành viên lỏng/đồng lõa đủ để nới định nghĩa "lỗ hổng
   hệ thống" — vì batch nên đòn bẩy của một phiếu là toàn-pot, không phải một claim.

**Phòng vệ chuyên biệt (KHÔNG tái dùng nguyên C.2 — vì bề mặt khác):**
- **Ngưỡng bằng chứng SYS cứng (evidence bar):** gán SYSTEM_FLAW đòi (i) reproducer chạy
  được trên môi trường độc lập, (ii) postmortem đã ký + diff patch **trỏ ĐÚNG lớp bị khai
  thác**, (iii) phạm vi ảnh hưởng đọc-từ-chain khớp danh sách DID được phủ. Thiếu bất kỳ
  → KHÔNG được nhãn SYS, rơi về GREY→USER (bảo thủ). Nhãn SYS không cấp bằng lời khai.
- **Vẫn soi sơ-suất TỪNG user, kể cả trong batch SYS:** batch chỉ xác nhận "lỗ hổng có
  thật + DID này trong phạm vi kỹ thuật". Nó KHÔNG miễn kiểm self-theft/thông-đồng cho
  từng DID. Mỗi DID trong batch vẫn qua bộ cờ C.2 (cluster địa chỉ nhận, tín hiệu phản
  ứng Freeze/InitRecovery, mối liên hệ lịch sử). DID có cờ đỏ → tách khỏi batch, xử người
  riêng — **KHÔNG tự-duyệt chỉ vì trong danh sách kỹ thuật**. Đây là sửa trực tiếp giả
  định "soi yếu" ở trên.
- **Committee challenge + recuse + phân tán:** phán quyết gán-SYS công khai on-chain,
  chọn thành viên ngẫu nhiên/luân phiên (VP-PersonDID, không token-weighted), bên liên
  quan recuse; **challenge window RIÊNG cho quyết-định-gán-SYS** (dài hơn challenge chi
  trả) để cộng đồng phản-tố *nhãn* trước khi batch mở van. Một phiếu lỏng không đủ nếu
  challenge lật được nhãn.
- **Trần theo-incident (per-incident cap) cho SYS batch:** tổng phủ một sự cố SYS ≤
  `CAP_INCIDENT_SYS` (DAO đặt) + vẫn dưới `POOL_CAP_PER_WINDOW`. Nếu exposure vượt → chi
  **pro-rata** (C.4), KHÔNG mở van vô hạn. Giới hạn thiệt hại khi một nhãn SYS sai lọt
  qua: nó không thể một-nhịp rút cạn `protectme_pot_system`.

**Thành thật:** ranh giới phân loại là một **quyết định con người có đòn bẩy pot-wide** —
không có oracle mật mã cho "đây có thật là lỗi hệ thống không" (đúng tinh thần C.6). Phòng
vệ là **nâng chi phí ngụy-tạo-nhãn** (evidence bar) + **không bỏ soi cá nhân dưới vỏ batch**
+ **giới hạn bán kính nổ** (per-incident cap + pro-rata) + **phân tán/challenge người
phán**. Chấp nhận: nhãn sai vẫn có thể lọt; thiết kế để **một** nhãn sai không thành thảm
hoạ pot. Ghim bất biến **PROTECT-TRIAGE-BOUNDARY** (B.7).

## C.8 🔴 DID 1-người, một-thiết-bị — "các cấp độ" sập về MỘT điểm (I-CURVE-5 va thực địa)

> Nêu bởi track identity-crypto (`_msg-to-Protectme-agent-scope.md`). Đây là việc Protectme
> phải trả lời TRONG lane (điều khoản phủ + định giá), KHÔNG viết lại curve/key-model.

**Giả định CHO SẴN (tiêu thụ, KHÔNG author):** value on-chain bảo vệ ở mức **SEED**, không
phải phần cứng — P-256 không verify được on-chain nên mọi quyền chi quy về Ed25519 dẫn-xuất-
seed (`Curve-Routing-Addendum` §3). **I-CURVE-5:** ≥1 second-factor cho rút lớn PHẢI
không-dẫn-xuất-từ-seed.

**Va chạm thực địa:** nông dân solo / TNHH 1 TV 1-người, MỘT thiết bị, thường KHÔNG có
second-hardware độc lập. Với cohort này:
- `secondary_pkh` (nếu dẫn từ cùng `Master_KEK`) + guardian (nếu seed-derived / cùng máy)
  **sập ĐỒNG THỜI** dưới MỘT lần lộ seed.
- ⇒ mọi "cấp độ" (trần anti-drain, khoá phụ, guardian) KHÔNG độc lập. Lộ seed hạ tất cả
  trừ MỘT thứ sống sót: **bộ đệm thời gian của anti-drain** (rate-limit + cửa sổ Freeze) —
  lớp phủ THẬT duy nhất còn lại.
- ⇒ rủi ro dồn hết vào (a) anti-drain time-buffer + (b) chi trả Protectme.

**Tấn công A — ăn chiết khấu giả (structural mispricing):** `risk_mult` (B.3) hiện cho ×0.8
vì cờ `guardian`, ×0.7 vì `anti_drain` — nhưng cờ KHÔNG kiểm factor có custody ĐỘC LẬP.
User (ngây thơ hoặc cố ý) đặt guardian **cùng seed / cùng máy** → hưởng ×0.8 cho lớp bảo vệ
KHÔNG tồn tại → trả premium thấp cho rủi ro cao. Khi seed bị trộm, cả ví lẫn guardian cùng
đổ; claim rơi vào `pot_user` ở mức phí đã chiết khấu cho phòng vệ ảo → **rút ngầm pot_user**
(biến thể C.5 × C.1, cấu trúc chứ không hành vi). Tệ hơn ở self-theft (C.2): solo giữ mọi
khoá nên "tín hiệu khôi phục" guardian (heuristic C.2/C.3) là GIẢ → panel yên tâm sai.

**Trả lời phủ (đúng lane Protectme):**

1. **Định giá theo ĐỘC-LẬP, không theo hiện-diện.** Cờ guardian/secondary chỉ hưởng chiết
   khấu nếu chứng thực custody KHÔNG dẫn-xuất-từ-seed chính:
```
independent(factor) ⟺ guardian là PersonDID trên enclave KHÁC (controller_pkh khác lineage /
                        device-attestation khác) ∨ kênh không-seed (VeData-Glint / EmailOracle)
   -- secondary_pkh cùng Master_KEK / guardian cùng thiết bị ⇒ KHÔNG độc lập ⇒ KHÔNG chiết khấu
   -- không chứng thực được độc lập ⇒ mặc định KHÔNG độc lập (bảo thủ cho pot)
risk_mult: ×0.8 (guardian) / ×0.9 (freeze-secondary) CHỈ khi independent(factor); ngược lại ×1.0
```
   Đây là chỗ Protectme định giá TRỰC TIẾP I-CURVE-5 — biến "second-factor phải khác seed"
   thành một tham số phí, không phải khẩu hiệu.

2. **Cohort single-factor (không thể có second-factor độc lập):**
   - **[PROTECT-SINGLE-FACTOR] Bắt buộc anti-drain để được phủ.** Time-buffer là lớp thật
     DUY NHẤT còn lại → phủ cho DID single-factor ĐÒI anti-drain bật + `limit` chặt.
     Single-factor + KHÔNG anti-drain ⇒ **không phủ USER_NEGLIGENCE** (không tín hiệu độc lập
     nào chống self-theft) — chỉ còn phủ **SYSTEM_FLAW** (nơi bằng chứng khách quan, C.6).
   - **Trần thấp + phí cao:** `CAP_USER_SINGLE < CAP_USER`; premium `× single_factor_loading`
     (mất chiết khấu guardian + tải rủi ro tập trung). Không phải phạt — định giá đúng phần
     rủi ro dồn vào pot.

3. **Bù trừ đẹp — anti-drain vừa là phủ THẬT vừa là trần EXPOSURE.** Với single-factor,
   `loss_eligible` bị chặn tự nhiên bởi (`limit` × số cửa sổ trước Freeze). **Siết anti-drain
   ⇒ đồng thời tăng bảo vệ thật VÀ giảm exposure pot** — cùng một tay quay. Nên với cohort
   này: phủ hào phóng PER-WINDOW nhưng tổng exposure nhỏ vì rate-cap đã bó. Khích lệ đã có
   sẵn trong `risk_mult` ("anti-drain limit chặt" → phí thấp).

4. **Đẩy về cam kết bất-khả-huỷ (tiêu thụ ranh giới, KHÔNG author).** Phần tài sản nằm ở
   lớp cam kết bất-khả-huỷ (vd khoá-lịch kiểu ScheduleGen của module commitment/anti-drain)
   **tự bảo vệ bằng tính không-thể-rút** — sống sót lộ seed BẤT KỂ second-factor → KHÔNG cần
   phủ Protectme. Chỉ **slice thanh khoản** (kiểu InstantGen) mới ở diện rủi ro. ⇒ Protectme
   định `coverage_cap`/premium theo **slice thanh khoản**, không theo tổng số dư. (ScheduleGen/
   InstantGen thuộc module khác — Protectme chỉ ĐỌC ranh giới committed-vs-liquid để định
   exposure, KHÔNG thiết kế nó.)

> **Ví dụ nông dân 100k LAMP:** 20k đã ScheduleGen (OriLife định-danh cây) + 30k dự-ScheduleGen
> (hợp đồng công nhân) = **cam kết bất-khả-huỷ → tự bảo vệ, exposure Protectme = 0.** Chỉ
> ~50k InstantGen là drainable. Với solo single-factor: siết `limit` anti-drain trên slice
> InstantGen → thiệt hại tối đa/cửa-sổ nhỏ, và đó cũng là trần exposure pot. Protectme chỉ
> định phủ trên ~50k liquid, không trên 100k.

**Thành thật:** với solo một-thiết-bị, "nhiều cấp độ" là marketing nếu mọi cấp cùng gốc
seed. Protectme KHÔNG sửa được điều đó (thuộc key-model) — nó chỉ (i) từ chối trả chiết khấu
cho phòng vệ ảo, (ii) buộc anti-drain làm ĐIỀU KIỆN phủ, (iii) định giá phần rủi ro tập trung
đúng, (iv) đẩy phần lớn giá trị về cam kết-bất-khả-huỷ. Ai hứa "solo một-thiết-bị vẫn an toàn
tuyệt đối nhiều lớp" đang bán ảo tưởng — đúng tinh thần C.6.

---

# PHẦN D — QUYẾT ĐỊNH CHỜ ANH CHỐT

Các tham số + lựa chọn cấu trúc dưới đây có tính **chính sách/kinh tế**, không thuần kỹ
thuật — cần anh (và DAO) chốt trước khi thành normative:

- **[PROT-1] Trần phủ:** `CAP_SYS` (đề xuất tới 100% loss_eligible, cap tuyệt đối/DID) và
  `CAP_USER` (≪ SYS), đơn vị CARP. Con số tuyệt đối theo loại DID? Tỷ lệ SYS:USER?
- **[PROT-2] Ai phân xử:** Security Committee (SYSTEM_FLAW) = ai, chọn thế nào, nhiệm kỳ?
  Adjudication Panel (USER_NEGLIGENCE) = VP-PersonDID cộng đồng hay hội đồng cử? Ngưỡng
  vote? Cơ chế recuse/challenge?
- **[PROT-3] Tỷ lệ phí:** `base_rate` khởi điểm (đề xuất 0.5–2%/năm), `θ_treasury` (phần
  phí dịch vụ vào pot_system), các hệ số `risk_mult`, `copay_rate` (20–50%?), `deductible`.
- **[PROT-4] Ngưỡng lỗi-hệ-thống vs sơ-suất:** tiêu chí cụ thể gán SYSTEM_FLAW (reproducer?
  audit ký? phạm vi ≥ N DID?) và xử ca GREY. Ai có quyền nâng GREY→SYS? — LƯU Ý gắn với
  [PROT-10] (evidence bar) vì đây là bề mặt tấn công C.7.
- **[PROT-5] Thời gian:** `T_report` (30 ngày?), `T_wait_activate` (14–30 ngày?),
  `T_wait_payout` + challenge window (7–14 ngày?); challenge window RIÊNG cho gán-SYS (C.7).
- **[PROT-6] Circuit-breaker pot:** `POOL_CAP_PER_WINDOW` (≤ x% pot/tháng?); `CAP_INCIDENT_SYS`
  (trần theo-incident cho batch SYS, C.7); chính sách pro-rata khi sự cố vượt pot (C.4);
  điều kiện vay chéo hai bucket.
- **[PROT-7] Opt-in vs bundle mặc định:** bật mức nền cho MỌI user (chống adverse selection,
  C.5) hay opt-in thuần? Đề xuất: nền mặc định + cap cao opt-in-có-waiting.
- **[PROT-8] Subrogation:** cơ chế truy hồi tài sản kẻ trộm (KYC dApp downstream?) và chia
  phần thu hồi (toàn bộ về pot hay chia với user?).
- **[PROT-9] Ranh giới pháp lý/KYC:** USER_NEGLIGENCE claim lớn có cần KYC không? Ảnh hưởng
  tới "không token-weighted" và quyền riêng tư — cần anh cân.
- **[PROT-10] 🔴 Evidence bar gán-SYS + per-incident cap (C.7):** ngưỡng bằng chứng cứng
  cho nhãn SYSTEM_FLAW (reproducer độc lập + postmortem-ký trỏ đúng lớp + phạm vi khớp
  chain); bắt buộc soi-sơ-suất-từng-user KỂ CẢ trong batch SYS; `CAP_INCIDENT_SYS`; quy
  tắc recuse/challenge riêng cho quyết-định-gán-nhãn. Đây là van pot-wide — cần chốt kỹ.
- **[PROT-11] 🔴 Cohort single-factor (C.8):** tiêu chí chứng thực ĐỘC-LẬP của guardian/
  secondary (enclave khác / kênh không-seed); `CAP_USER_SINGLE`; `single_factor_loading`;
  có bắt buộc anti-drain làm điều kiện phủ USER_NEGLIGENCE cho DID single-factor không (đề
  xuất: có); cơ chế Protectme ĐỌC ranh giới committed-vs-liquid (ScheduleGen/InstantGen) từ
  module cam kết để định exposure theo slice thanh khoản.

---

# PHẦN E — VIỆC CÒN LẠI (sau khi anh duyệt) + RANH GIỚI

## E.1 Long (Treasury + backend)
- **Hai bucket** `protectme_pot_system` + `protectme_pot_user` trong `LAMP/Treasury`,
  release DAO-gated (T1 Model A). KHÔNG sổ-phụ (Treasury không có primitive đó). Ghi
  claim-record on-chain đối ứng (T2). **Đường THU KHÔNG gọi `collectToTreasury`** — nạp
  qua `FeecoverEpochSettle` (dưới).
- **Feecover premium**: wire Protectme như một `service_id` (`protectme.*`) trên đường
  Feecover — user tiêu MAGIC (Vault `BurnBatch`, ĐỊNH GIÁ nanogic), phí CARP tích
  `FeecoverAccrual` (App→Platform) trong epoch, **cuối epoch `FeecoverEpochSettle`** chuyển
  CARP về bucket `protectme_pot_user` (`app_id="feecover"`); grace lapse; `last_paid_epoch`.
  KHÔNG `collectToTreasury` (cả MAGIC lẫn CARP — đường đó đã bỏ cho phí Feecover).
- **Resolver/indexer claim**: trạng thái ClaimPacket, đọc anti-drain log + vault UTxO
  scan làm bằng chứng, tính `loss_eligible` (A.6/B.5).
- Backend PhoenixKey = ranh giới Long (Claude KHÔNG sửa — memory ranh-giới-code).

## E.2 Tuân (validator — nếu cần)
- Anti-drain log (Withdrawal-Limit) ĐÃ là bằng chứng; Protectme **đọc**, không đổi
  validator custody. Nếu claim-record cần on-chain enforce (T2) → nhánh release gate ở
  Treasury validator (Long/Tuân phối hợp).

## E.3 Core + SuperApp (UI)
- Màn **Protectme**: bật/tắt, chọn cap, xem premium (risk_mult breakdown — "bật
  anti-drain giảm X%"), lịch Feecover, no-claim streak.
- Màn **Mở claim**: chọn incident (auto-điền từ anti-drain log + tx trộm), gán nguyên
  nhân sơ bộ, đính bằng chứng off-chain, xem `loss_eligible` ước tính (đã trừ phần
  ví-theo-DID tự cứu — minh bạch A.6).
- Màn **Trạng thái claim**: triage → adjudicate → waiting/challenge → paid; timeline.
- **Ngôn ngữ người-thường** (i18n): "Bảo vệ tài sản", "Phần đã tự an toàn (không cần
  bồi hoàn)", "Đang xác minh", "Phần được bồi hoàn". KHÔNG hứa "đền 100% mọi lúc"
  (C.4 — nói rõ trần + pro-rata khi sự cố lớn).

## E.4 Ranh giới rõ (KHÔNG đặc tả ở đây)
- **Tokenomics MAGIC/LAMP, giá nanogic, FX** = MagicLamp/anchor-economics (chỉ tham chiếu).
- **Governance chi tiết** (cơ cấu committee/panel, luật vote) = LAMP/Governance (chỉ
  tham chiếu VP-PersonDID không token-weighted).
- **Luật KYC/pháp lý từng quốc gia** = ngoài kỹ thuật ([PROT-9] cần anh cân).
- **Mô hình khôi phục guardian, anti-drain custody, ví-theo-DID** = spec riêng (chỉ TRỎ
  tới làm nguồn bằng chứng + mặt phủ).

---

*Hết bản đề xuất. Mọi mục [N] là đề xuất, chỉ thành normative sau khi anh Aladin duyệt
và hợp thức hoá vào PhoenixKey-Specs. §PHẢN BIỆN (C) là phần cần anh đọc kỹ nhất —
Protectme chỉ bền nếu các đánh đổi ở C được chấp nhận có ý thức.*
