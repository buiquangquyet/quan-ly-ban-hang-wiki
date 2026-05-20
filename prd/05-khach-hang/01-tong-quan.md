# Tổng quan — Module Khách hàng

**Phạm vi:** Module `/man/#/Customers` — quản lý danh sách khách hàng, công nợ, tích điểm và lịch sử giao dịch.
**Test data:** 1,624 KH | Nợ hiện tại 7,181,257,836đ | Tổng bán 8,517,104,309đ | Tổng bán trừ trả hàng 8,468,482,779đ
**Entity catalog:** [README.md](./README.md)

---

## 1. Đối tượng trung tâm — Customer (C01)

### 1.1 Identity & Contact

```
Customer (C01)
├── code              string   KH125055XXX — auto, sequential per merchant
├── name              string   BẮT BUỘC
├── customerType      enum     Cá_nhân | Công_ty (Tổ chức/Hộ kinh doanh)
├── gender            enum     Nam | Nữ
├── birthday          date     dùng cho campaign sinh nhật + filter "Tháng sinh"
├── avatarUrl         string   ≤ 2MB
├── facebookUrl       string   facebook.com/username
│
├── phone1            string   Phone chính
├── phone2            string   Phone phụ — 2 trường (hiếm có ở phần mềm khác)
├── email             string
├── address           string   free text
├── ward              string   Phường/Xã (chuẩn địa danh hành chính mới 01/07/2025)
├── province          string   Tỉnh/Thành phố
│
├── groupIds          FK[]     Multi-group (Nhóm KH — C04)
├── tierLevelId       FK?      Hạng KH (C07 — Vàng/Bạc/Đồng)
├── assignedToId      FK User  Người phụ trách (C11)
├── createdBranchId   FK       CN tạo KH
├── createdById       FK User
├── createdAt         datetime
└── note              string
```

### 1.2 Sales Aggregates (denormalized — cập nhật bởi HD/TH/Phiếu thu)

| Trường | Công thức | Hiển thị |
|---|---|---|
| `totalSale` | Σ Invoice.total (Hoàn thành) | Cột **Tổng bán** trên list |
| `totalSaleNet` | totalSale − Σ SalesReturn.value | Cột **Tổng bán trừ trả hàng** |
| `totalReturn` | Σ SalesReturn.value | Derived |
| `balance` | Σ Invoice.debt còn lại | Cột **Nợ hiện tại** |
| `lastTransactionAt` | MAX(Invoice.soldAt) | Filter "Ngày giao dịch cuối" |
| `points` | Σ LoyaltyTransaction.pointDelta | Điểm tích lũy hiện tại |

3 cột tài chính (Nợ/Tổng bán/Tổng bán trừ trả hàng) hiển thị **agg total ở header row** — chủ shop thấy portfolio tổng nhanh.

### 1.3 Thông tin xuất HĐĐT (InvoiceIssuingInfo — C03)

```
InvoiceIssuingInfo (embedded trong Customer)
├── buyerName       default = Customer.name
├── taxCode         MST — có button "Tra cứu MST" → gọi API Tổng cục Thuế VN
├── idCard          CCCD/CMND (compliance VN)
├── passport        Số hộ chiếu (KH nước ngoài)
├── invoiceAddress  địa chỉ riêng cho HĐ (≠ địa chỉ liên hệ)
├── invoiceProvince / invoiceWard
├── invoiceEmail    gửi HĐĐT
├── invoicePhone
├── bankName        dropdown bank VN
└── bankAccount     STK cho B2B settlement
```

Đáp ứng đầy đủ **Nghị định 123/2020/NĐ-CP + TT78/2021/TT-BTC** cho phát hành HĐĐT từ máy tính tiền.

### 1.4 Địa chỉ nhận hàng (ShippingAddress — C02)

```
ShippingAddress (1..N per Customer)
├── recipientName   string   người nhận (khác chủ KH)
├── recipientPhone  string
├── address, ward, province
└── isDefault       bool
```

1 KH có thể có nhiều địa chỉ nhận (nhà / công ty / kho / ...). Có cờ default cho checkout nhanh.

### 1.5 Lịch sử tích điểm (LoyaltyTransaction — C06)

```
LoyaltyTransaction
├── customerId, voucherCode (mã phiếu nguồn)
├── transactionType    enum    Tích | Đổi | Điều_chỉnh | Hết_hạn
├── value              decimal Giá trị giao dịch gốc
├── pointDelta         integer +tích / −đổi
├── pointAfter         integer running balance
└── transactionAt      datetime
```

---

## 2. List view & Filter

### 2.1 Cột mặc định

| Cột | Ghi chú |
|---|---|
| Mã KH | clickable → expand inline detail panel |
| Tên KH | |
| Điện thoại | |
| **Nợ hiện tại** | KPI — Σ HĐ chưa thu |
| **Tổng bán** | gross revenue |
| **Tổng bán trừ trả hàng** | net revenue per KH — đo lường thực |

### 2.2 Top actions

| Button | Function |
|---|---|
| **+ Khách hàng** | Mở modal tạo |
| **Gửi tin nhắn ▼** | Bulk message: **ZNS / SMS / Email / Zalo OA** |
| **Import file** | Bulk import từ Excel |
| **...** | Bulk actions (export, gán nhóm, gán người phụ trách) |
| **⚙️** | Cài đặt cột hiển thị |

### 2.3 Sidebar filter (9 tiêu chí)

| Filter | Kiểu | Use case |
|---|---|---|
| Nhóm khách hàng | dropdown + "Tạo mới" | Segmentation |
| Chi nhánh tạo | multi-select | Phân CN quản lý |
| Ngày tạo | Toàn thời gian / Tùy chỉnh | Cohort analysis |
| Người tạo | dropdown user | Audit |
| Người phụ trách | dropdown user | KAM portfolio |
| **Loại KH** | Tất cả / Cá nhân / Công ty | B2B vs B2C |
| **Giới tính** | Tất cả / Nam / Nữ | Demographic |
| **Sinh nhật** | Toàn thời gian / Tùy chỉnh | Birthday campaign |
| **Ngày giao dịch cuối** | Toàn thời gian / Tùy chỉnh | **Churn detection** |

---

## 3. Form tạo/sửa khách hàng — 4 section accordion

```
┌────────────────────────────────────────────────────────────────┐
│  Tạo khách hàng                                             [×] │
├────────────────────────────────────────────────────────────────┤
│  [Section 1 — luôn mở]                                          │
│  Tên KH* | Mã (auto) | ĐT 1 | ĐT 2 | Sinh nhật | Giới tính    │
│  Email | Facebook | [Avatar upload ≤2MB]                        │
├─ Địa chỉ ────────────────────────────────────────────── [▼/▲] ─┤
│  Địa chỉ | Tỉnh/TP | Phường/Xã (địa danh hành chính mới)       │
├─ Nhóm KH, người phụ trách, ghi chú ──────────────────── [▼/▲] ─┤
│  Nhóm KH (multi-select + Tạo mới) | Người phụ trách | Ghi chú  │
├─ Thông tin xuất hóa đơn ─────────────────────────────── [▼/▲] ─┤
│  ◉ Cá nhân   ○ Tổ chức/Hộ kinh doanh                           │
│  Tên người mua | MST [Tra cứu MST]                              │
│  Địa chỉ HĐ | Tỉnh | Phường/Xã                                  │
│  CCCD/CMND | Số hộ chiếu                                        │
│  Email HĐ | SĐT HĐ | Ngân hàng | Số TK                         │
├────────────────────────────────────────────────────────────────┤
│                                            [Bỏ qua]  [Lưu]     │
└────────────────────────────────────────────────────────────────┘
```

**Điểm nổi bật:**
- **2 trường phone** — hiếm so với phần mềm tương đương
- Field xuất HĐ ≠ field liên hệ → KH cá nhân có thể có địa chỉ giao hàng (nhà) khác địa chỉ HĐ (công ty)
- Tool **Tra cứu MST** integrated — auto-fill tên & địa chỉ DN từ Tổng cục Thuế

---

## 4. Detail view — 5 tab inline

Khi click row KH → expand panel ngay dưới row (không navigate sang trang khác).

### 4.1 Tab "Thông tin"

```
[Avatar]  hoa  KH125055745                    Chi nhánh trung tâm
          Người tạo: giangtest | 29/01/2026 | Nhóm: Chưa có
          📊 Xem phân tích

Điện thoại  0965888222
Địa chỉ     Phường Hàng Buồm, Quận Hoàn Kiếm, Hà Nội
⚠️  Địa chỉ đã thay đổi sau sáp nhập 01/07/2025
    Bạn có muốn đổi thành: Phường Hoàn Kiếm - TP Hà Nội?  [Đổi địa chỉ]
+ Thêm thông tin xuất hóa đơn
```

**Điểm nổi bật:**
- **Banner địa danh hành chính mới** — proactive đề xuất chuẩn hóa địa chỉ sau cải cách sáp nhập 01/07/2025 (VN bỏ cấp huyện)
- Link **"Xem phân tích"** → module Phân tích với drill-down hành vi KH
- "Thêm thông tin xuất HĐ" lazy-reveal khi cần

### 4.2 Tab "Địa chỉ nhận hàng"

List multi-address với recipient name + phone riêng. Có default flag và actions (sửa/xóa/đặt mặc định).

### 4.3 Tab "Lịch sử bán/trả hàng"

```
| Mã hóa đơn | Thời gian        | Người bán | Chi nhánh | Tổng cộng | Trạng thái  |
| HD016561   | 29/01/2026 17:11 | giangtest | CN TT     | 28,505    | Đang xử lý  |
```

List flat các HD bán + TH trả. Click mã → mở chi tiết HĐ. Có "Xuất file".

### 4.4 Tab "Nợ cần thu từ khách" — AR Toolbox

```
Filter: [Tất cả giao dịch ▼]
| Mã phiếu | Thời gian        | Loại     | Giá trị | Dư nợ  |
| HD016561 | 29/01/2026 17:11 | Bán hàng | 28,505  | 28,505 |

Actions: [Xuất file công nợ] [Xuất file]
         [Thanh toán] [Điều chỉnh] [Chiết khấu thanh toán] [Tạo QR]
```

| Action | Function |
|---|---|
| **Thanh toán** | Tạo phiếu thu nhanh — chọn HĐ + nhập số tiền + phương thức |
| **Điều chỉnh** | Sửa số dư công nợ thủ công (admin override + audit) |
| **Chiết khấu thanh toán** | Giảm trừ công nợ (ưu đãi KH B2B) |
| **Tạo QR** | Generate VietQR có số tiền cố định → KH quét thanh toán |
| **Xuất file công nợ** | Excel báo cáo per KH cho kế toán |

### 4.5 Tab "Lịch sử tích điểm"

```
| Mã phiếu | Thời gian | Loại   | Giá trị | Điểm GD | Điểm sau GD |
| HD016561 | ...       | Tích   | 28,505  | +28     | 128         |
```

Loyalty ledger: cộng/trừ điểm, running balance. Action **Điều chỉnh điểm** — admin override.

*(Cấu hình chương trình tích điểm nằm ở module Bán online → Tích điểm — không scope file này.)*

---

## 5. State machine — Vòng đời khách hàng

```
[Tạo mới]
    │  (từ form Quản lý / POS bán hàng / Import Excel / tin nhắn KH đầu tiên)
    ▼
[Active]  ◄──────────────────────────────────────┐
    │                                              │
    ├─ mua hàng ──► totalSale↑, lastTransAt↑       │
    ├─ trả hàng ──► totalReturn↑                    │
    ├─ tích điểm ─► points↑                         │
    ├─ vào nhóm ──► groupIds += groupId              │
    └─ tạo HĐ nợ ─► balance↑                       │
                                                    │
[Inactive]  (derived: lastTransAt > 90 ngày — không phải status thật)
    │                                              │
    │  campaign re-engage ─────────────────────►   │
    │                                              │
[Lost]  (derived: > 1 năm không mua)               │
```

**Điểm yếu:** KiotViet KHÔNG có `status` field thật (`active/inactive/lost`). Tất cả lifecycle phải derive từ `lastTransactionAt`. Không có lifecycle stage tường minh để gán workflow tự động.

---

## 6. Tích hợp với module khác

```
                  ┌──────── Customer (C01) ────────┐
                  │                                  │
  ┌───────────────┼──────────┬───────────────────────┼──────────────────┐
  ▼               ▼          ▼                       ▼                  ▼
[Bán hàng HD] [Đặt hàng DH] [Trả hàng TH]    [Tích điểm C06]  [Người phụ trách (User)]
  └─ totalSale   └─ pre-paid  └─ totalReturn    └─ points         └─ Sales portfolio
  └─ balance↑    └─ balance   └─ balance↓

[Sổ quỹ phiếu thu TTHD/TTDH] ──────────────► Customer.balance ↓
[Sổ quỹ phiếu chi TTTH refund] ─────────────► Customer.balance ↑
[CustomerGroup rule C05] ────────────────────► auto add/remove KH khỏi nhóm
[Bảng giá] ──────────────────────────────────► áp giá riêng per nhóm KH
[Marketing CSKH] ────────────────────────────► ZNS/SMS/Email/Zalo OA
```

---

## 7. Điểm đau & cơ hội cải tiến

### 7.1 Identity & Dedup

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có **dedup khi tạo KH trùng SĐT** — có thể tạo 2 record cùng số | Auto-detect, hiển thị "đã có KH với SĐT này, gộp?" |
| 2 | 2 trường phone nhưng không có validate format / quốc tịch | E.164 validate + country detect |
| 3 | "Khách lẻ" walk-in chiếm 60–70% giao dịch — không có data, mất khả năng remarketing | Gamify nhập SĐT (giảm 5% nếu đăng ký), link tự khai báo |
| 4 | 1 KH có FB DM, Zalo, SĐT — không gom về 1 view | Identity Resolution Engine (scope module CRM 360 riêng) |
| 5 | Mã KH dùng pattern duy nhất `KH###` — không phân biệt nguồn | Cấu hình prefix theo nguồn: WALK, FB, TIKTOK, SHOPEE... |

### 7.2 Lifecycle & Status

| # | Pain | Đề xuất |
|---|---|---|
| 6 | Không có **status field** (Active/Inactive/Lost) — phải derive từ lastTransactionAt | Lifecycle Stage field (C09) + auto-transition rule |
| 7 | Không có "Customer onboarded date" ≠ "Created date" — chuỗi cần biết KH đã active chưa | Thêm `activatedAt` — lần đầu tiên tạo HĐ thành công |
| 8 | KH bị xóa → mất history HĐ liên kết | Soft delete + restore + audit log |
| 9 | Không có **VIP flag / Blacklist flag** thủ công | Quick flag VIP/Blacklist trên detail panel, không qua nhóm |

### 7.3 Công nợ (AR)

| # | Pain | Đề xuất |
|---|---|---|
| 10 | Tab "Nợ cần thu" không có **aging bucket** (0-30 / 31-60 / 61-90 / >90 ngày) | Bucket display + chart |
| 11 | Không có **credit limit** per KH (C10) — bán quá hạn mức vẫn cho | Setting credit limit + block khi vượt |
| 12 | Không có **auto reminder** SMS/ZNS khi nợ quá hạn | Workflow: T+7, T+15, T+30 trigger template |
| 13 | "Điều chỉnh công nợ" không có reason taxonomy — chỉ free note | Dropdown: Sai HĐ / Khuyến mãi sau / Bad debt / Khác |
| 14 | Không có **bad debt provision** | Flag "Khó đòi" + provision % cấu hình |
| 15 | Khi KH trả nợ partial nhiều HĐ, không có UI allocation thông minh | Cho phép chọn từng HĐ để gán payment (oldest-first / specific) |

### 7.4 Segmentation & Loyalty

*(Xem chi tiết tại [02-nhom-va-segmentation.md](./02-nhom-va-segmentation.md))*

| # | Pain | Đề xuất |
|---|---|---|
| 16 | Rule engine chỉ AND — không có OR / NOT / nested group | Boolean tree builder |
| 17 | Không có frequency rule "mua ≥ 3 lần trong 90 ngày qua" | Time window cho mỗi điều kiện |
| 18 | Không có RFM template preset | 1-click apply RFM với Champion/Loyal/At Risk/Hibernating |
| 19 | Tích điểm chỉ là counter — không có multi-program (Bronze/Silver/Gold + benefit riêng) | Loyalty tier engine với rule per tier |
| 20 | Không có **referral tracking** (KH giới thiệu KH khác) | Mã giới thiệu + commission |

### 7.5 Người phụ trách (Account Manager)

| # | Pain | Đề xuất |
|---|---|---|
| 21 | 1 KH gán cho **1 NV duy nhất** — không có team selling / collab | Multi-owner + role per owner (Sales / Support / Collection) |
| 22 | Khi NV nghỉ, KH gán họ không auto-reassign | Off-board flow → bulk transfer ownership |
| 23 | Không có **NV portfolio dashboard** (phụ trách bao nhiêu KH, doanh thu, nợ) | View "My customers" cho NV |

### 7.6 Địa chỉ & B2B

| # | Pain | Đề xuất |
|---|---|---|
| 24 | Banner "Đổi địa chỉ" sau cải cách 01/07/2025 cần bấm thủ công từng KH | Bulk migrate all addresses |
| 25 | Multi-address không có geocoding (lat/lng) | Google Maps API → tính cost ship chính xác |
| 26 | "Loại KH = Công ty" nhưng không có **multi-contact per company** (sales/kế toán/ship riêng) | Tách Account vs Contact |
| 27 | Không có **parent company / subsidiary** hierarchy | Linked accounts |
| 28 | Không có **deal pipeline** (Lead → Qualified → Quoted → Won / Lost) | Mini CRM stage tracker |

---

## 8. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **AR Aging + Credit Limit + Auto Reminder** | Pain #10, #11, #12 — compliance lớn cho shop B2B | Trung | Rất cao |
| 2 | **Advanced Rule Engine** (OR/NOT/nested + time window + RFM preset + live preview) | Pain #16-18 — segment chính xác hơn | Cao | Rất cao |
| 3 | **Identity Dedup + Phone Validate** | Pain #1, #2 — pain hàng ngày tại POS | Trung | Cao |
| 4 | **Lifecycle Stage + VIP/Blacklist Flag** | Pain #6, #9 — workflow chuẩn cho KAM | Thấp | Cao |
| 5 | **B2B Account Mode** (multi-contact, parent/subsidiary, deal pipeline) | Pain #26-28 — seller buôn 3-20 CN | Cao | Cao (B2B) |

---

## 9. Tóm lược

**Khách hàng KiotViet:**
- **Hub trung tâm** nối 5 module: Sales / AR / Loyalty / Marketing / Pricing
- **Mạnh:** 5 tab inline đầy đủ; form HĐĐT chuẩn pháp lý VN (MST/CCCD/Hộ chiếu/Bank); filter "Ngày GD cuối" cho churn detection; bulk message 4 kênh; banner địa chỉ hành chính mới proactive; multi-address per KH
- **Mạnh ngạc nhiên:** Nhóm KH có **rule engine 12 điều kiện** với dynamic sync mode + auto-execute — đủ build RFM cơ bản (xem `02-nhom-va-segmentation.md`)
- **Yếu chính:** Không có lifecycle stage tường minh, credit limit, AR aging, boolean tree rule (chỉ AND), B2B account multi-contact, dedup phone

Đã hoàn thiện CRUD + ghi nhận dòng tiền/loyalty cơ bản — nhiều dư địa cho **intelligence (segmentation, prediction) và automation (campaign, reminder, AR collection)** — đây là moat KiotViet có thể đào sâu để chống cạnh tranh từ CRM chuyên (HubSpot, Salesforce, Sleekflow).
