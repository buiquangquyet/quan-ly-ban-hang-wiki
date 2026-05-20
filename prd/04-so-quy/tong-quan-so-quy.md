# PHÂN TÍCH SỔ QUỸ — KIOTVIET

**Phạm vi:** Module `/man/#/CashFlow` — sổ cái dòng tiền của shop. Tách bạch với Thẻ kho (sổ cái hàng hóa), Sổ quỹ là **single source of truth cho mọi biến động tiền** đi qua shop.

**Test data quan sát (testzone17):** 46 phiếu thu chi, Quỹ đầu kỳ 0, Tổng thu 9,045,753, Tổng chi -729,095, Tồn quỹ 8,316,658 (Tổng quỹ qua mọi thời gian).

---

## 1. VAI TRÒ TRONG HỆ SINH THÁI

```
[Hóa đơn HD]      ─KH trả tiền──►  [Phiếu thu TTHD]  ─ghi──► Sổ quỹ ─update──► Customer.Nợ
[Đơn hàng DH]     ─KH cọc────────►  [Phiếu thu TTDH]  ─ghi──► Sổ quỹ
[Trả hàng TH]     ─refund─────────► [Phiếu chi TTTH]  ─ghi──► Sổ quỹ
[Phiếu nhập PN]   ─trả NCC────────► [Phiếu chi]       ─ghi──► Sổ quỹ ─update──► Supplier.Nợ
[Trả hàng nhập TPN] ─NCC hoàn tiền► [Phiếu thu]       ─ghi──► Sổ quỹ
[Chi phí khác]    ─tay──────────► [Phiếu thu/chi tự lập] ─ghi──► Sổ quỹ
[Chuyển quỹ]      ─internal──────► [Phiếu CNH cặp]    ─ghi──► Sổ quỹ
```

Sổ quỹ là **điểm hội tụ** của 6 module: Bán hàng, Đặt hàng, Trả hàng, Mua hàng, Trả hàng nhập, Customer/Supplier credit. Mọi tiền vào/ra shop đều **bắt buộc** ghi qua đây.

---

## 2. CẤU TRÚC ĐỐI TƯỢNG

### 2.1 4 loại Quỹ (Fund Type) — radio

| Quỹ | Mô tả | Use case |
|---|---|---|
| **Tiền mặt** | Tiền vật lý tại quầy | POS cash drawer |
| **Ngân hàng** | TK NH liên kết với gian hàng | Chuyển khoản, QR thanh toán |
| **Ví điện tử** | MoMo / ZaloPay / VNPay | TT online, KH ưu tiên ví |
| **Tổng quỹ** | Aggregate 3 loại trên | Báo cáo tổng quan |

Mỗi shop có thể có nhiều TK NH và nhiều ví — list lên hiển thị aggregate theo loại.

### 2.2 Phiếu thu / Phiếu chi (Receipt / Payment Voucher)

```
CashVoucher  (Phiếu thu chi)
├── Header
│   ├── code            string   Prefix + 6 digits (xem § 3)
│   ├── voucherType     enum     Phiếu_thu | Phiếu_chi
│   ├── transactionType enum     Loại thu chi (xem § 4)
│   ├── fundType        enum     Tiền_mặt | Ngân_hàng | Ví_điện_tử
│   ├── fundAccountId   FK?      TK NH/Ví cụ thể (nếu fundType ≠ tiền mặt)
│   ├── branchId        FK
│   ├── createdAt / createdBy
│   ├── voucherDate     datetime Ngày ghi sổ
│   ├── status          enum     Đã_thanh_toán | Đã_hủy
│   ├── partnerType     enum     Customer | Supplier | Employee | Other
│   ├── partnerId       FK?
│   ├── partnerName     string   Cache (đối tác có thể đổi tên)
│   ├── value           decimal  Giá trị (luôn dương, dấu suy từ voucherType)
│   ├── accountingFlag  bool     Hạch toán KQKD (có/không)
│   ├── sourceDocId     FK?      Link HĐ/PN/Đơn hàng nguồn (nếu auto-gen)
│   ├── sourceDocType   enum?    HD | PN | DH | TH | TPN | Manual
│   └── note            string
```

### 2.3 Liên kết với module khác

```
CashVoucher ─sourceDocId─► Invoice (HD)         — phiếu thu tự sinh
            ─sourceDocId─► CustomerOrder (DH)   — phiếu thu cọc
            ─sourceDocId─► SalesReturn (TH)     — phiếu chi refund
            ─sourceDocId─► PurchaseImport (PN)  — phiếu chi NCC
            ─sourceDocId─► PurchaseReturn (TPN) — phiếu thu hoàn NCC
            ─partnerId──► Customer / Supplier   — update balance
            ─partnerId──► Employee              — chi lương/tạm ứng
```

---

## 3. MÃ CHỨNG TỪ (DOCUMENT CODE)

### 3.1 Quan sát từ test data

| Prefix | Tên đầy đủ (suy luận) | Nghiệp vụ | Source doc |
|---|---|---|---|
| **TTHD** | Thu Tiền Hóa Đơn | KH thanh toán HĐ bán hàng | → HD###### |
| **TTDH** | Thu Tiền Đơn Hàng / Đặt hàng | KH cọc cho đơn đặt hàng | → DH###### |
| **TTTH** | Trả Tiền Trả Hàng / Chi Tiền Trả Khách | Refund khi khách trả hàng | → TH###### |
| **TTD** | Tổng Thu Đối ứng (?) | (chưa rõ — xuất hiện kèm CNH) | |
| **CNH** | Chuyển/rút giữa các quỹ (Cash to bank/wallet hoặc ngược lại) | Internal transfer | (cặp đối ứng) |
| **TTD_CNHxxx** | Phiếu chi tương ứng với CNH | Đối ứng của CNH | |

### 3.2 Quy tắc đặt mã

- Format: **[PREFIX] + [SEQUENTIAL 6 DIGITS]** — vd `TTHD016157`, `TTTH000168`, `CNH000021`
- Sequence **chia sẻ với HĐ gốc** ở một số trường hợp: `TTHD016157` ⟵ HD016157 (cùng số 6 chữ)
- `CNH000021` xuất hiện cặp với `TTD_CNH000021` cho chuyển/rút quỹ — 1 chuyển = 2 phiếu (1 thu CN nhận + 1 chi CN xuất)

### 3.3 Sample (toàn thời gian, Tổng quỹ — sắp xếp giảm dần thời gian)

| Mã phiếu | Thời gian | Loại thu chi | Người nộp/nhận | Giá trị |
|---|---|---|---|---|
| TTTH000168 | 06/04/2026 15:32 | Chi Tiền trả khách | ngan | -97,995 |
| TTD_CNH000021 | 15/01/2026 17:39 | Chuyển/Rút | 001122334455 | 200,000 |
| CNH000021 | 15/01/2026 17:39 | Chuyển/Rút | (đối ứng) | -200,000 |
| TTHD016157 | 03/11/2025 17:12 | Thu Tiền khách trả | Phương Thu | 10,000 |
| TTHD016094 | 20/10/2025 15:17 | Thu Tiền khách trả | Ngan | 12,005 |
| TTHD016034 | 09/10/2025 14:47 | Thu Tiền khách trả | Phan ngân | 45,005 |
| TTHD014924 | 21/03/2025 16:33 | Thu Tiền khách trả | Ngan Nga | 881,005 |
| TTHD014517 | 10/01/2025 10:57 | Thu Tiền khách trả | Ngan | 3,921,405 |
| TTDH000734 | 22/08/2024 11:51 | Thu Tiền khách trả (đặt hàng) | abc | 45,005 |
| TTHD012363 | 03/06/2024 14:55 | Thu Tiền khách trả | test01 | 445,903 |

**Quan sát:**
- Mã CNH/TTD_CNH luôn xuất hiện thành **cặp đối ứng** với cùng timestamp
- Tên "Loại thu chi" hiển thị trên UI ≠ prefix code — UI dùng label dài "Thu Tiền khách trả", "Chuyển/Rút"...
- "Người nộp/nhận" có thể là: tên KH, số TK ngân hàng (cho CNH), tên ad-hoc

---

## 4. LOẠI THU CHI (TRANSACTION TYPE)

UI có dropdown "Loại thu chi" cho phép filter. Đây là enum cấu hình được — shop có thể tạo loại mới (vd "Chi lương", "Chi điện nước", "Thu khác từ phụ phí")

### 4.1 Loại thu chuẩn (built-in)

| Loại thu | Source | Auto-create | Hạch toán KQKD |
|---|---|---|---|
| Thu Tiền khách trả (HĐ) | HD | ✅ | Có (doanh thu đã hạch toán ở HĐ) |
| Thu cọc đơn đặt hàng | DH | ✅ | Không (cọc chưa phải doanh thu) |
| Thu từ NCC (Trả hàng nhập) | TPN | ✅ | Có (giảm chi phí mua hàng) |
| Thu khác | Manual | ❌ | Tùy chọn |
| Chuyển/Rút quỹ | Manual | ❌ (cặp đối ứng) | **Không** (internal) |

### 4.2 Loại chi chuẩn (built-in)

| Loại chi | Source | Auto-create | Hạch toán KQKD |
|---|---|---|---|
| Chi Tiền trả khách (refund) | TH | ✅ | Có (giảm doanh thu) |
| Chi trả NCC | PN | ✅ | Có (chi phí mua hàng đã hạch toán ở PN) |
| Chi lương | Manual | ❌ | Có |
| Chi điện nước / vận chuyển / quảng cáo... | Manual | ❌ | Có |
| Chi khác | Manual | ❌ | Tùy chọn |
| Chuyển/Rút quỹ | Manual | ❌ | **Không** |

### 4.3 Flag "Hạch toán kết quả kinh doanh"

Filter sidebar có 3 trạng thái: **Tất cả / Có / Không**.
- **Có** = phiếu thu chi ảnh hưởng P&L (doanh thu/chi phí)
- **Không** = chuyển quỹ nội bộ, vay/cho vay chủ, ứng/hoàn ứng nhân viên... — chỉ là dòng tiền, không tính lãi/lỗ
- Đây là điểm KiotViet làm khá tinh tế — tách bạch cash flow với P&L

---

## 5. STATE MACHINE

```
              [Tạo phiếu]
                  │
                  ▼
            ┌─ Đã thanh toán ──┐
            │                  │ [Hủy]
            │                  ▼
            │              Đã hủy
            │              (immutable, vẫn hiện trong list khi filter)
            │
            ▼
       Cập nhật:
       - Sổ quỹ (Tồn quỹ ± value)
       - Customer.balance / Supplier.balance (nếu có partner)
       - Source doc.paidAmount / debtAmount
```

**Không có trạng thái "Phiếu tạm"** như Nhập hàng. Sổ quỹ tạo xong là commit ngay → khác Inventory module (cần phiếu tạm vì đếm hàng cần thời gian).

---

## 6. UI/UX TRÊN MÀN SỔ QUỸ

### 6.1 Layout

```
┌────────────────────────────────────────────────────────────────────┐
│ Sổ quỹ tiền mặt / Tổng quỹ              [+Phiếu thu][+Phiếu chi]   │
├────────────────────────────────────────────────────────────────────┤
│ Search Mã phiếu                          [Xuất file] [⚙] [?]      │
├──────────────────┬─────────────────────────────────────────────────┤
│ FILTER SIDEBAR   │  Quỹ đầu kỳ | Tổng thu | Tổng chi | Tồn quỹ ⓘ │
│                  │                                                  │
│ Quỹ tiền:        │  ┌──────────────────────────────────────────┐  │
│  ○ Tiền mặt      │  │ Mã phiếu │ Thời gian │ Loại │ Người │ GT │  │
│  ○ Ngân hàng     │  │ TTTH168  │ 06/04/26  │ Chi  │ ngan  │−97 │  │
│  ○ Ví điện tử    │  │ CNH021   │ 15/01/26  │ Chuy.│ ...  │+200│  │
│  ◉ Tổng quỹ      │  │ ...                                       │  │
│                  │  └──────────────────────────────────────────┘  │
│ Chi nhánh: [...] │                                                 │
│ Thời gian:       │                                                 │
│ Loại chứng từ:   │                                                 │
│  ☐ Phiếu thu     │                                                 │
│  ☐ Phiếu chi     │                                                 │
│ Loại thu chi: [] │                                                 │
│ Trạng thái:      │                                                 │
│  ☑ Đã thanh toán │                                                 │
│  ☐ Đã hủy        │                                                 │
│ Hạch toán KQKD:  │                                                 │
│  [Tất cả|Có|Không]│                                                │
│ Người tạo: []    │                                                 │
│ Nhân viên: []    │                                                 │
└──────────────────┴─────────────────────────────────────────────────┘
```

### 6.2 Hiện có

- ✅ Filter giàu (10+ tiêu chí)
- ✅ Dropdown thời gian có **dương lịch + âm lịch** + Toàn thời gian + Tùy chỉnh
- ✅ Top bar KPI: Quỹ đầu kỳ / Tổng thu / Tổng chi / Tồn quỹ với tooltip ⓘ
- ✅ "+ Phiếu thu" và "+ Phiếu chi" đều có **caret dropdown** → suggest có nhiều sub-types (Thu khác, Thu từ KH, Chuyển quỹ...)
- ✅ Xuất file
- ✅ Cài đặt cột hiển thị (icon ⚙️)

### 6.3 Còn THIẾU

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có **dashboard P&L view** — chỉ ledger thuần | Tab "Phân tích" với Sankey diagram cash in/out per category |
| 2 | Không có **cash flow forecast** (dự báo dòng tiền tuần tới dựa trên đơn DH chưa thu + phiếu PN sắp đến hạn) | Module forecast 7/14/30 ngày |
| 3 | Filter "Loại thu chi" là enum chuẩn, không thấy custom-create dễ dàng từ list | Allow inline create loại mới |
| 4 | Không có **reconciliation** với sao kê NH | Auto-match phiếu thu/chi với giao dịch NH (qua MB API/ACB API) |
| 5 | Không có **multi-currency** | Cho seller xuyên biên giới |
| 6 | Phiếu hủy vẫn còn trong list, không có lý do hủy | Audit reason + workflow approval |
| 7 | Chuyển quỹ tạo 2 phiếu nhưng UI không gắn liên kết visual | Link "↔ phiếu đối ứng" trên detail |
| 8 | Không thấy **petty cash workflow** (quỹ tạm ứng nhân viên) | Module Petty Cash với approval limit |
| 9 | Không có **alert** khi tồn quỹ vượt ngưỡng (vd Tiền mặt > 50tr nên chuyển NH) | Setting "Max cash on hand" + alert |
| 10 | Không có **e-signature** cho phiếu in (compliance) | Digital signature cho phiếu thu/chi |

---

## 7. ĐỐI CHIẾU VỚI CHUẨN KẾ TOÁN

Sổ quỹ KiotViet ≈ **Sổ quỹ tiền mặt** (Mẫu S07-DN) và **Sổ tiền gửi ngân hàng** (Mẫu S08-DN) theo Thông tư 200/2014/TT-BTC. Tuy nhiên:

| Khía cạnh | KiotViet | Kế toán chuẩn |
|---|---|---|
| Single entry | ✅ Phiếu thu/chi | ❌ |
| Double entry (Nợ/Có) | ❌ | ✅ |
| Tài khoản đối ứng | Có flag hạch toán KQKD nhưng không có TK 111/112/131/331... | ✅ |
| Audit trail | ✅ (mã immutable) | ✅ |
| Đối chiếu sao kê NH | ❌ | ✅ |
| Hạch toán đa tệ | ❌ | ✅ |

→ KiotViet xử lý kế toán đầy đủ trong module **Thuế & Kế toán** riêng (xem menu top bar). Sổ quỹ chỉ là **operational ledger**, không phải accounting ledger. Có thể coi là layer 1 (ops) feed vào layer 2 (kế toán).

---

## 8. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN

### 8.1 Nhập liệu & UX

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Tạo "Phiếu thu/chi khác" thủ công — nhiều shop F&B/Salon có 10-20 phiếu chi vặt/ngày | Quick templates: "Mua bánh sáng", "Ship grab", "Đổ xăng" — 1 click lưu |
| 2 | Không có **bulk import phiếu** từ Excel | Import flow giống Hóa đơn |
| 3 | Không thấy **scan hóa đơn giấy** → tự tạo phiếu chi (như nhiều ví điện tử đang có) | OCR receipt + auto-create phiếu chi |
| 4 | App mobile không thấy có shortcut cho Sổ quỹ → chủ shop phải vào web | Mobile widget "Hôm nay thu/chi bao nhiêu?" |

### 8.2 Liên thông

| # | Pain | Đề xuất |
|---|---|---|
| 5 | Không có **2-way sync với ngân hàng** (đã có 1-way nhận QR realtime cho VietQR) | Pull sao kê → match phiếu (đã trigger ở Reconciliation module §6.4) |
| 6 | Phiếu chi NCC sinh từ PN, nhưng nếu chia nhỏ thanh toán (3 lần) thì sinh 3 phiếu — không có "lịch trả NCC" view | Vendor payment schedule view |
| 7 | KH thanh toán qua ví Momo/ZaloPay tạo phiếu thu thuộc quỹ "Ví điện tử" nào? — không thấy chọn tay | Auto-route theo channel thanh toán |

### 8.3 Báo cáo & Analytics

| # | Pain | Đề xuất |
|---|---|---|
| 8 | Không có **biểu đồ cash flow theo ngày/tuần/tháng** trên màn Sổ quỹ | Chart toggle on top bar |
| 9 | Không có **breakdown theo loại thu chi** dạng pie | Pie chart filter-aware |
| 10 | Không thấy **runway** (số ngày shop sống được với mức chi hiện tại nếu doanh thu = 0) | KPI runway hữu ích cho shop nhỏ |
| 11 | Không có **comparative period** (kỳ này vs kỳ trước) | YoY/MoM toggle |

### 8.4 Phân quyền & kiểm soát

| # | Pain | Đề xuất |
|---|---|---|
| 12 | Phiếu thu/chi có thể tạo bởi bất kỳ NV có quyền — không có **approval ngưỡng** | Workflow: phiếu chi > 5tr cần manager duyệt |
| 13 | Không có **trace audit** ai sửa/hủy phiếu | Full audit log per field |
| 14 | Khi nhân viên nghỉ, các phiếu họ tạo vẫn ghi tên — không có transfer ownership | Transfer flow khi off-board |

---

## 9. CƠ HỘI ĐỘT PHÁ — TOP 5 CHO SỔ QUỸ

| # | Sản phẩm/Tính năng | Pain | Effort | Impact |
|---|---|---|---|---|
| 1 | **Bank Reconciliation Engine** — auto-match phiếu thu/chi với sao kê NH | Pain #5, #8.5 — pain cực lớn cho shop có nhiều TK | Cao | Rất cao |
| 2 | **Receipt OCR** — chụp ảnh hóa đơn chi → auto tạo phiếu chi | Pain #8.3 | Trung | Cao |
| 3 | **Cash Flow Forecast** — AI dự báo 7/30 ngày | Pain #2 | Cao | Cao |
| 4 | **P&L Dashboard** — bóc tách doanh thu/chi phí theo category với filter "Hạch toán KQKD = Có" | Pain #6.1 | Trung | Rất cao |
| 5 | **Approval Workflow** — phiếu chi vượt ngưỡng cần duyệt | Pain #8.12 | Trung | Cao (cho chuỗi) |

---

## 10. ENTITY MODEL BỔ SUNG

| # | Đối tượng | Vai trò |
|---|---|---|
| 49 | **CashVoucher (Phiếu thu/chi)** | Prefix TTHD/TTDH/TTTH/CNH, header đơn |
| 50 | **FundType (Loại quỹ)** | Tiền mặt / NH / Ví / Tổng |
| 51 | **FundAccount (Tài khoản quỹ)** | TK NH cụ thể, ví cụ thể |
| 52 | **TransactionType (Loại thu chi)** | Enum cấu hình được, gắn flag KQKD |
| 53 | **AccountingFlag (Hạch toán KQKD)** | Bool tách dòng tiền vs P&L |

---

## 11. TÓM LƯỢC

**Sổ quỹ KiotViet:**
- Là **operational cash ledger** — tách bạch với accounting (module Thuế & Kế toán) và inventory ledger (Thẻ kho)
- **4 loại quỹ** (Tiền mặt / NH / Ví / Tổng) và **2 loại chứng từ** (Phiếu thu / Phiếu chi)
- Mã chứng từ prefix-based: TTHD (thu từ HĐ), TTDH (thu từ DH), TTTH (chi refund), CNH (chuyển quỹ)
- **Mạnh:** Filter giàu (Hạch toán KQKD tách dòng tiền vs P&L, dương + âm lịch, 10+ tiêu chí), auto-link với 6 module khác, QR thanh toán VietQR realtime
- **Yếu:** Không có reconciliation NH, không có P&L view, không có forecast, không có approval workflow, không có receipt OCR
- **Cơ hội đột phá:** Bank reconciliation, OCR receipt, AI cash flow forecast, P&L dashboard, approval workflow

Đây là module **đã hoàn thiện về ledger cơ bản nhưng còn rất nhiều dư địa cho intelligence (forecast, anomaly) và compliance (reconciliation, audit) — match với segment cao cấp và chuỗi**.
