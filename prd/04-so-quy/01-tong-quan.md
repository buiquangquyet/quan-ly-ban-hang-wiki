# Tổng quan — Sổ quỹ

**Phạm vi:** Module `/man/#/CashFlow` — operational cash ledger ghi nhận mọi dòng tiền vào/ra shop. Tách bạch với Thẻ kho (inventory ledger) và module Thuế & Kế toán (accounting ledger).

**Test data:** 46 phiếu thu chi | Tổng thu 9,045,753đ | Tổng chi −729,095đ | Tồn quỹ 8,316,658đ
**Entity catalog:** [README.md](./README.md)

---

## 1. Vai trò trong hệ sinh thái

```
[Bán hàng HD]      ──KH trả tiền──►  [Phiếu thu TTHD]  ─► Sổ quỹ → Customer.balance ↓
[Đặt hàng DH]      ──KH cọc────────► [Phiếu thu TTDH]  ─► Sổ quỹ
[Trả hàng TH]      ──refund────────► [Phiếu chi TTTH]  ─► Sổ quỹ → Customer.balance ↑
[Nhập hàng PN]     ──trả NCC───────► [Phiếu chi]       ─► Sổ quỹ → Supplier.balance ↓
[Trả hàng nhập TPN]──NCC hoàn tiền► [Phiếu thu]       ─► Sổ quỹ → Supplier.balance ↑
[Chi phí khác]     ──thủ công──────► [Phiếu thu/chi]   ─► Sổ quỹ
[Chuyển quỹ]       ──internal──────► [CNH cặp đối ứng] ─► Sổ quỹ (±0 net)
```

Sổ quỹ là **điểm hội tụ** của 6 module. Mọi tiền vào/ra shop đều bắt buộc ghi qua đây — single source of truth cho **cash position** tại bất kỳ thời điểm nào.

**Phân biệt 3 sổ cái:**

| Sổ cái | Module | Ghi nhận |
|---|---|---|
| **Thẻ kho** | Hàng hóa | Biến động số lượng hàng hóa |
| **Sổ quỹ** | Sổ quỹ (file này) | Biến động tiền thực tế (cash flow) |
| **Sổ kế toán** | Thuế & Kế toán | Bút toán Nợ/Có theo chuẩn TT200 |

---

## 2. Cấu trúc đối tượng

### 2.1 Loại quỹ (FundType — SQ02)

| Quỹ | Mô tả | Use case |
|---|---|---|
| **Tiền mặt** | Tiền vật lý tại quầy | POS cash drawer |
| **Ngân hàng** | TK NH liên kết | Chuyển khoản, QR thanh toán |
| **Ví điện tử** | MoMo / ZaloPay / VNPay | Thanh toán online |
| **Tổng quỹ** | Aggregate 3 loại trên | Báo cáo tổng quan |

1 shop có thể có nhiều TK NH và nhiều ví (SQ03 — FundAccount). View "Tổng quỹ" hiển thị aggregate toàn bộ.

### 2.2 Phiếu thu / Phiếu chi (CashVoucher — SQ01)

```
CashVoucher  (SQ01)
├── code            string    Prefix + 6 digits (xem §3)
├── voucherType     enum      Phiếu_thu | Phiếu_chi
├── transactionType FK → SQ04 Loại thu chi
├── fundType        enum      Tiền_mặt | Ngân_hàng | Ví_điện_tử
├── fundAccountId   FK → SQ03 TK NH/Ví cụ thể (nếu fundType ≠ tiền mặt)
├── branchId        FK
├── voucherDate     datetime  Ngày ghi sổ
├── status          enum      Đã_thanh_toán | Đã_hủy
├── partnerType     enum      Customer | Supplier | Employee | Other
├── partnerId       FK?
├── partnerName     string    Cache (đối tác có thể đổi tên sau)
├── value           decimal   Giá trị (luôn dương — dấu suy từ voucherType)
├── accountingFlag  bool      Hạch toán KQKD (SQ05)
├── sourceDocId     FK?       Link HĐ/PN/DH/TH/TPN gốc (nếu auto-gen)
├── sourceDocType   enum?     HD | PN | DH | TH | TPN | Manual
└── note            string
```

### 2.3 Liên kết phiếu nguồn

```
CashVoucher ─sourceDocId─► Invoice (HD)          — phiếu thu auto-gen khi KH thanh toán
            ─sourceDocId─► CustomerOrder (DH)    — phiếu thu cọc
            ─sourceDocId─► SalesReturn (TH)      — phiếu chi refund
            ─sourceDocId─► PurchaseImport (PN)   — phiếu chi trả NCC
            ─sourceDocId─► PurchaseReturn (TPN)  — phiếu thu hoàn từ NCC
            ─partnerId──►  Customer / Supplier   — update balance
            ─partnerId──►  Employee              — chi lương/tạm ứng
```

---

## 3. Mã chứng từ (SQ06 — CashTransfer)

### 3.1 Prefix quan sát được

| Prefix | Tên đầy đủ | Nghiệp vụ | Phiếu nguồn |
|---|---|---|---|
| **TTHD** | Thu Tiền Hóa Đơn | KH thanh toán HĐ bán hàng | → HD###### |
| **TTDH** | Thu Tiền Đơn Hàng | KH cọc cho đơn đặt hàng | → DH###### |
| **TTTH** | Chi Tiền Trả Khách | Refund khi KH trả hàng | → TH###### |
| **CNH** | Chuyển NH/quỹ | Chuyển tiền giữa các quỹ | (cặp đối ứng) |
| **TTD_CNH** | Đối ứng CNH | Phiếu chi ứng với CNH | ↔ CNH###### |

**Quy tắc:**
- Format: **[PREFIX][6 digits]** — VD: `TTHD016157`, `TTTH000168`, `CNH000021`
- `TTHD016157` → sequence chia sẻ với HD016157 (cùng số 6 chữ)
- Chuyển quỹ `CNH` luôn xuất hiện **cặp đối ứng** với `TTD_CNH` cùng timestamp và cùng giá trị nhưng ngược dấu

### 3.2 Sample data (toàn thời gian, Tổng quỹ)

| Mã phiếu | Thời gian | Loại thu chi | Người nộp/nhận | Giá trị |
|---|---|---|---|---|
| TTTH000168 | 06/04/2026 15:32 | Chi tiền trả khách | ngan | −97,995 |
| TTD_CNH000021 | 15/01/2026 17:39 | Chuyển/Rút | 001122334455 | +200,000 |
| CNH000021 | 15/01/2026 17:39 | Chuyển/Rút | (đối ứng) | −200,000 |
| TTHD016157 | 03/11/2025 17:12 | Thu tiền khách trả | Phương Thu | +10,000 |
| TTHD014517 | 10/01/2025 10:57 | Thu tiền khách trả | Ngan | +3,921,405 |
| TTDH000734 | 22/08/2024 11:51 | Thu tiền đặt hàng | abc | +45,005 |

---

## 4. Loại thu chi (CashTransactionType — SQ04)

Enum **cấu hình được** — shop có thể tạo loại mới (VD: "Chi lương", "Chi điện nước", "Thu phụ phí ship").

### 4.1 Loại thu chuẩn (built-in)

| Loại thu | Auto-gen từ | Hạch toán KQKD |
|---|---|---|
| Thu tiền khách trả (HĐ) | HD | Có (doanh thu đã hạch toán ở HĐ) |
| Thu cọc đơn đặt hàng | DH | Không (cọc chưa phải doanh thu) |
| Thu từ NCC (TPN) | TPN | Có (giảm chi phí mua hàng) |
| Thu khác | Manual | Tùy chọn |
| Chuyển/Rút quỹ | Manual | **Không** (internal) |

### 4.2 Loại chi chuẩn (built-in)

| Loại chi | Auto-gen từ | Hạch toán KQKD |
|---|---|---|
| Chi tiền trả khách (refund) | TH | Có (giảm doanh thu) |
| Chi trả NCC | PN | Có (chi phí mua hàng) |
| Chi lương | Manual | Có |
| Chi vận hành (điện/nước/quảng cáo...) | Manual | Có |
| Chi khác | Manual | Tùy chọn |
| Chuyển/Rút quỹ | Manual | **Không** |

### 4.3 Flag "Hạch toán KQKD" (AccountingFlag — SQ05)

Filter sidebar có 3 trạng thái: **Tất cả / Có / Không**

- **Có** = phiếu thu chi ảnh hưởng P&L (doanh thu/chi phí thực)
- **Không** = chuyển quỹ nội bộ, vay/cho vay chủ, ứng/hoàn ứng nhân viên — chỉ là dòng tiền, không tính lãi/lỗ

Đây là điểm KiotViet thiết kế tinh tế: **tách bạch cash flow với P&L** ngay trên 1 màn hình mà không cần 2 module riêng.

---

## 5. State machine — Vòng đời phiếu thu chi

```
         [Tạo phiếu]
               │
               ▼
         Đã thanh toán ────────[Hủy]────► Đã hủy
               │                          (immutable, vẫn hiện trong list khi filter)
               │
               ▼  (ngay lập tức, không có "Phiếu tạm")
         Sổ quỹ cập nhật:
           - Tồn quỹ ± value
           - Customer.balance / Supplier.balance (nếu có partner)
           - SourceDoc.paidAmount / debtAmount
```

**Không có trạng thái "Phiếu tạm"** — khác với Phiếu nhập hàng (cần đếm hàng vật lý). Sổ quỹ commit ngay khi tạo vì tiền đã thực sự trao tay.

---

## 6. UI/UX

### 6.1 Layout tổng quan

```
┌──────────────────────────────────────────────────────────────────┐
│ Sổ quỹ tiền mặt / Tổng quỹ          [+ Phiếu thu] [+ Phiếu chi] │
├──────────────────────────────────────────────────────────────────┤
│ Search Mã phiếu                          [Xuất file] [⚙️] [?]    │
├──────────────┬───────────────────────────────────────────────────┤
│ FILTER       │  Quỹ đầu kỳ | Tổng thu | Tổng chi | Tồn quỹ ⓘ   │
│              │                                                    │
│ Loại quỹ:   │  Mã phiếu  │ Thời gian │ Loại   │ Người │ Giá trị│
│  ○ Tiền mặt │  TTTH168   │ 06/04/26  │ Chi    │ ngan  │  −97k  │
│  ○ Ngân hàng│  CNH021    │ 15/01/26  │ Chuyển │ ...   │  +200k │
│  ○ Ví ĐT    │  ...                                               │
│  ◉ Tổng quỹ │                                                    │
│              │                                                    │
│ Chi nhánh   │                                                    │
│ Thời gian   │                                                    │
│ Loại chứng từ│                                                   │
│ Loại thu chi│                                                    │
│ Trạng thái  │                                                    │
│ Hạch toán   │                                                    │
│ Người tạo   │                                                    │
│ Nhân viên   │                                                    │
└──────────────┴───────────────────────────────────────────────────┘
```

### 6.2 Điểm mạnh

- Filter giàu: 10+ tiêu chí bao gồm **Hạch toán KQKD** (Có/Không/Tất cả)
- Dropdown thời gian hỗ trợ **dương lịch + âm lịch** + Toàn thời gian + Tùy chỉnh
- Top bar KPI: Quỹ đầu kỳ / Tổng thu / Tổng chi / Tồn quỹ với tooltip ⓘ
- Nút "+ Phiếu thu/chi" có **caret dropdown** → sub-types (Thu khác, Thu từ KH, Chuyển quỹ...)
- Xuất file + cài đặt cột hiển thị

---

## 7. Đối chiếu với chuẩn kế toán

Sổ quỹ KiotViet ≈ **Sổ quỹ tiền mặt** (Mẫu S07-DN) và **Sổ tiền gửi ngân hàng** (Mẫu S08-DN) theo Thông tư 200/2014/TT-BTC.

| Khía cạnh | KiotViet Sổ quỹ | Kế toán chuẩn |
|---|---|---|
| Ghi nhận | Single entry (phiếu thu/chi) | Double entry (Nợ/Có) |
| Tài khoản đối ứng | Flag KQKD — không có TK 111/112/131/331 | Có đầy đủ tài khoản |
| Audit trail | Có (mã immutable) | Có |
| Đối chiếu sao kê NH | Không có | Bắt buộc |
| Multi-currency | Không có | Có |

→ KiotViet xử lý kế toán đầy đủ trong module **Thuế & Kế toán** riêng. Sổ quỹ là **layer ops** (operational ledger) feed vào layer accounting — không phải thay thế.

---

## 8. Điểm đau & cơ hội cải tiến

### 8.1 Nhập liệu & UX

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Tạo phiếu chi vặt thủ công nhiều lần/ngày (F&B/Salon: 10–20 phiếu chi nhỏ) | Quick templates: "Mua bánh sáng", "Ship grab" — 1-click lưu |
| 2 | Không có bulk import phiếu từ Excel | Import flow tương tự Hóa đơn |
| 3 | Không có scan hóa đơn giấy → tự tạo phiếu chi | OCR receipt + auto-create phiếu chi |
| 4 | App mobile không có shortcut Sổ quỹ — chủ shop phải vào web | Mobile widget "Hôm nay thu/chi bao nhiêu?" |

### 8.2 Liên thông & đối soát

| # | Pain | Đề xuất |
|---|---|---|
| 5 | Không có **2-way sync với ngân hàng** (có 1-way nhận QR realtime qua VietQR) | Pull sao kê NH → auto-match phiếu thu/chi (Bank Reconciliation) |
| 6 | Phiếu chi NCC chia nhỏ thanh toán (3 lần) sinh 3 phiếu — không có "lịch trả NCC" view | Vendor payment schedule view |
| 7 | KH thanh toán Momo/ZaloPay không thấy auto-route vào quỹ "Ví điện tử" đúng | Auto-route theo payment channel |
| 8 | Chuyển quỹ tạo 2 phiếu nhưng UI không gắn liên kết visual | Link "↔ phiếu đối ứng" trên detail panel |

### 8.3 Báo cáo & phân tích

| # | Pain | Đề xuất |
|---|---|---|
| 9 | Không có **biểu đồ cash flow** theo ngày/tuần/tháng trên màn Sổ quỹ | Chart toggle trên top bar |
| 10 | Không có **P&L view** từ phiếu có flag "Hạch toán KQKD = Có" | Dashboard bóc tách doanh thu/chi phí theo category |
| 11 | Không có **cash flow forecast** 7/14/30 ngày (từ DH chưa thu + PN sắp đến hạn) | AI forecast module (SQ07) |
| 12 | Không có **runway** (số ngày shop sống được nếu doanh thu = 0) | KPI runway tính từ tồn quỹ / chi phí bình quân |
| 13 | Không có **so sánh kỳ** (kỳ này vs kỳ trước) | YoY / MoM toggle |

### 8.4 Phân quyền & kiểm soát

| # | Pain | Đề xuất |
|---|---|---|
| 14 | Phiếu thu/chi tạo bởi bất kỳ NV có quyền — không có **approval ngưỡng** | Workflow: phiếu chi > 5tr cần manager duyệt |
| 15 | Không có **audit log per field** — ai sửa/hủy phiếu | Full audit trail per field |
| 16 | Phiếu hủy không có **reason taxonomy** | Dropdown: Nhập sai / Giao dịch lỗi / Hủy bởi KH / Khác |
| 17 | Không có **petty cash workflow** (quỹ tạm ứng nhân viên với approval limit) | Module Petty Cash |
| 18 | Không có **alert** khi tồn quỹ vượt ngưỡng (VD: Tiền mặt > 50tr nên chuyển NH) | Setting "Max cash on hand" + cảnh báo |

---

## 9. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Bank Reconciliation Engine** — auto-match phiếu thu/chi với sao kê NH | Pain #5 — pain lớn nhất cho shop nhiều TK | Cao | Rất cao |
| 2 | **P&L Dashboard** — bóc tách doanh thu/chi phí theo category từ flag "Hạch toán KQKD = Có" | Pain #10 — insight quan trọng chủ shop cần hàng ngày | Trung | Rất cao |
| 3 | **Receipt OCR** — chụp ảnh hóa đơn giấy → auto tạo phiếu chi | Pain #3 — giảm 50% thời gian nhập vặt | Trung | Cao |
| 4 | **Cash Flow Forecast** — AI dự báo 7/30 ngày từ DH pending + lịch trả NCC | Pain #11 — proactive cash management | Cao | Cao |
| 5 | **Approval Workflow** — phiếu chi vượt ngưỡng cần manager duyệt | Pain #14 — bắt buộc với chuỗi nhiều CN | Trung | Cao (enterprise) |

---

## 10. Tóm lược

**Sổ quỹ KiotViet:**
- **Operational cash ledger** — tách bạch với accounting (Thuế & Kế toán) và inventory (Thẻ kho)
- 4 loại quỹ (Tiền mặt / NH / Ví / Tổng) + 2 loại chứng từ (Phiếu thu / Phiếu chi)
- Mã chứng từ: TTHD (thu từ HĐ), TTDH (thu từ DH), TTTH (chi refund), CNH (chuyển quỹ — cặp đối ứng)
- **Mạnh:** Filter giàu với flag "Hạch toán KQKD" tách cash flow vs P&L; dương + âm lịch; auto-link 6 module; QR thanh toán VietQR realtime
- **Yếu:** Không có bank reconciliation, không có P&L view, không có forecast, không có approval workflow, không có receipt OCR
- **Cơ hội đột phá:** Bank Reconciliation, P&L Dashboard, Receipt OCR, Cash Flow Forecast AI, Approval Workflow

Đã hoàn thiện về ledger cơ bản — nhiều dư địa cho **intelligence (forecast, anomaly detection) và compliance (reconciliation, approval audit)** — match với segment chuỗi và shop volume lớn.
