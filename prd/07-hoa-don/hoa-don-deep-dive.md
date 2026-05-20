# ĐÀO SÂU: MODULE HÓA ĐƠN (INVOICES) — KIOTVIET

**Phạm vi:** Hóa đơn (`HD######`) là **chứng từ thương mại cốt lõi** — ghi nhận giao dịch bán đã chốt, trừ tồn thật, ghi doanh thu vào sổ kế toán, có thể phát hành HĐĐT theo NĐ123/2020 + TT78/2021. Đây là phiếu **trung tâm** mà gần như mọi module khác liên kết tới.

**Vị trí UI:** `/man/#/Invoices` (menu top-bar **Đơn hàng → Hóa đơn**)
**Mã prefix:** `HD######` (6 digits sequential per merchant)
**Test data:** 2 HĐ trong tháng 5/2026 — HD017065 (DUONG, 29,605đ), HD017064 (anh Quyết, 103,000+đ)

---

## 1. KHÁI NIỆM & VỊ TRÍ TRONG HỆ SINH THÁI

```
┌─ Customer Journey ──────────────────────────────────────────────────────┐
│                                                                          │
│   POS bán nhanh ────┐                                                    │
│                     ├──► [Hóa đơn HD] ──► Doanh thu + Trừ tồn + Phiếu thu│
│   Đặt hàng DH ──────┘                          │                         │
│                                                ├──► HĐĐT (e-invoice)     │
│                                                ├──► Vận đơn (carrier)    │
│                                                ├──► Loyalty point        │
│                                                └──► Customer aggregates  │
│                                                                          │
│   Trả hàng TH  ◄────── (refund từ HĐ) ───────  HĐ                       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 1.1 HD là điểm hội tụ của 7 luồng

| Luồng | Quan hệ với HD |
|---|---|
| Bán nhanh tại POS | Tạo HD trực tiếp, không qua DH |
| Đặt hàng (DH) | "Xử lý đơn hàng" → tạo HD từ DH (xem `don-hang-deep-dive.md`) |
| Trả hàng (TH) | TH **bắt buộc tham chiếu HD** gốc (trừ "Trả nhanh") |
| Sổ quỹ | Sinh phiếu thu TTHD### khi KH thanh toán |
| Thẻ kho | Mỗi line HD = 1 dòng StockCard `HD######` |
| Customer | Cập nhật `totalSale`, `balance`, `lastTransactionAt`, points |
| HĐĐT (e-invoice) | HD có thể được phát hành thành HĐĐT chứng từ thuế |
| Vận chuyển | HD "Giao hàng" sinh vận đơn carrier (KShip, GHN, Viettel...) |

### 1.2 2 loại HĐ chính

Filter "Loại hóa đơn" có 2 checkbox:

| Loại | Đặc điểm | Use case |
|---|---|---|
| **Không giao hàng** | Bán ngay tại quầy, không cần ship | POS truyền thống, F&B, walk-in |
| **Giao hàng** | Có shipping address, link vận đơn, COD | Online order, delivery |

Cả 2 loại đều dùng prefix `HD###` chung (không tách prefix theo loại).

---

## 2. CẤU TRÚC DỮ LIỆU (SCHEMA)

### 2.1 Entity chính — `Invoice`

```
Invoice
├── Header
│   ├── code                  string   HD######
│   ├── customerId            FK?      Nullable (có "Khách lẻ" walk-in)
│   ├── customerNameSnapshot  string   Cache tên KH (KH đổi tên không phá HĐ)
│   ├── branchId              FK
│   ├── soldAt                datetime Ngày bán
│   ├── soldById              FK User  Người bán
│   ├── createdById           FK User  Người tạo (≠ soldBy khi tạo từ HD list)
│   ├── salesChannelId        FK       Kênh bán: Bán trực tiếp | An95 Store | Shopee | ...
│   ├── priceBookId           FK
│   ├── orderId               FK?      Link DH gốc (nếu có)
│   ├── status                enum     Đang_xử_lý | Hoàn_thành | Không_giao_được | Đã_hủy
│   ├── shippingStatus        enum     7 states (xem §3.3)
│   ├── eInvoiceStatus        enum     10+ states (xem §3.4)
│   ├── isShippingInvoice     bool     true = Giao hàng, false = Không giao hàng
│   ├── invoiceType           enum     HD bán hàng | HD GTGT | HD máy tính tiền
│   │
│   ├── Pricing (đa thuế suất)
│   │   ├── subtotal              decimal   Σ lineTotals
│   │   ├── orderDiscount         decimal   Giảm giá hóa đơn
│   │   ├── customSurcharges      [...]     Thu khác list (vd `tets = 5`)
│   │   ├── vatBreakdown          [{rate, base, vat}]  ⭐ Multi-rate
│   │   │                                       VAT 0%: 0 / VAT 10%: 2,600 / ...
│   │   ├── shippingFee           decimal   Thu phí ship
│   │   └── total                 decimal   Khách cần trả
│   │
│   ├── Payment
│   │   ├── paidAmount            decimal   Khách đã trả
│   │   ├── balance               decimal   total − paidAmount → còn nợ
│   │   ├── paymentMethods        [enum]    Tiền mặt / CK / Thẻ / Ví / Voucher / Điểm
│   │   └── paidVouchers          [FK]      Phiếu thu liên quan (TTHD###)
│   │
│   ├── Shipping (chỉ khi isShippingInvoice = true)
│   │   ├── deliveryAddress       string
│   │   ├── recipientName         string
│   │   ├── recipientPhone        string
│   │   ├── carrierId             FK       Grab/GHN/Viettel/J&T/KShip/...
│   │   ├── trackingCode          string
│   │   ├── shippingHistory       [...]    Lịch sử trạng thái (audit log)
│   │   ├── isCOD                 bool
│   │   ├── codAmount             decimal
│   │   ├── declaredValue         decimal?
│   │   └── expectedDeliveryAt    datetime?
│   │
│   ├── E-Invoice (HĐĐT) — section riêng
│   │   ├── eInvoiceNumber        string?  Số HĐĐT khi đã phát hành
│   │   ├── eInvoiceTemplate      string   Mẫu HĐ (ký hiệu)
│   │   ├── publishedAt           datetime?
│   │   ├── taxAuthorityRefCode   string?  Mã CQT trả về
│   │   └── adjustmentChain       [...]    Chuỗi HĐ điều chỉnh/thay thế
│   │
│   ├── canceledAt / canceledById / cancelReason
│   └── note                      string
│
└── Lines (1..N) — 11 cột giá đầy đủ
    ├── productId / variantId / unitId
    ├── unitConversion        decimal
    ├── quantity              decimal
    ├── stockQuantity         decimal   = quantity × unitConversion
    ├── unitPrice             decimal   Đơn giá (trước thuế)
    ├── unitPriceAfterTax     decimal   Đơn giá sau thuế
    ├── discount              decimal
    ├── discountAfterTax      decimal
    ├── salePrice             decimal
    ├── salePriceAfterTax     decimal
    ├── lineTotal             decimal
    ├── lineTotalAfterTax     decimal
    ├── vatRate               decimal   ⭐ Per-line VAT (0/5/8/10%)
    ├── costPriceAtSale       decimal   SNAPSHOT WAC tại t=soldAt (cho lãi gộp)
    ├── serialNumbers         string[]  Serial/IMEI nếu có
    ├── batchLotId            FK?       Lô/HSD nếu có
    └── note                  string
```

### 2.2 Multi-rate VAT trên 1 HĐ ⭐

Sample test: HĐ HD017065 có cả **VAT 0%: 0** và **VAT 10%: 2,600** trên cùng 1 phiếu.

→ KiotViet hỗ trợ **đa thuế suất** trên cùng HĐ — mỗi line có `vatRate` riêng. Tổng bottom hiển thị breakdown từng mức:

```
Tổng tiền hàng (2)    26,000
Giảm giá hóa đơn           0
VAT 0%                     0     ← line SP000006 áp 0%
tets                       5
VAT                    2,600     ← line SP000004 áp 10%
Thu phí ship           1,000
Khách cần trả         29,605
```

→ Match yêu cầu HĐĐT khi shop bán SP thuộc nhiều ngành (TT78/2021 yêu cầu bóc tách VAT theo từng nhóm hàng).

### 2.3 Liên kết với module khác

```
Invoice (HD######)
    │
    ├── customerId ──► Customer
    │                  - totalSale += total
    │                  - totalSaleNet += total − returns
    │                  - balance += (total − paidAmount)
    │                  - points += earned (theo rule loyalty)
    │
    ├── (mỗi line) ──► StockCard `HD######` (qty âm)
    │                  - Stock.onHand −= stockQuantity
    │                  - costPriceAtSale snapshot
    │
    ├── (khi paid) ──► CashVoucher `TTHD######` (Phiếu thu)
    │
    ├── (mỗi line nếu Serial) ──► StockUnit.status = "sold", soldInvoiceId = HD
    │
    ├── (mỗi line nếu Lô/HSD) ──► BatchStock.qty −= quantity
    │
    ├── (nếu là Giao hàng) ──► ShippingOrder (vận đơn) + ShippingHistory
    │
    ├── (nếu phát hành HĐĐT) ──► EInvoice (KV eInvoice) + TaxAuthorityLog
    │
    └── (nếu hoàn về) ──► AutoSalesReturn (TH######)
```

---

## 3. STATE MACHINES

### 3.1 Vòng đời Hóa đơn (`status`)

```
        [Tạo HD]
            │
            ▼
       Đang xử lý ◄───────────┐
        │  │                  │
        │  │ ─[Hủy]──► Đã hủy │
        │  │           (immutable, rollback tồn + phiếu thu) 
        │  ▼                  │
        │  Không giao được    │
        │  (Giao hàng fail —   │ 
        │   tách khỏi Hoàn thành)
        │
        ▼
    Hoàn thành ◄────────────► (đã thu đủ, đã giao xong)
```

### 3.2 4 trạng thái tường minh

| Trạng thái | Mô tả | Tác động |
|---|---|---|
| **Đang xử lý** | HD đã tạo nhưng chưa hoàn tất (chưa thu đủ / chưa giao xong) | Tồn đã trừ, doanh thu đã ghi |
| **Hoàn thành** | Đã thu đủ + (nếu Giao hàng) đã giao thành công | Đóng workflow |
| **Không giao được** | Carrier báo fail (KH không nhận, sai địa chỉ...) | Pending xử lý: re-ship / hoàn / hủy |
| **Đã hủy** | Hủy phiếu hoàn toàn | Rollback tồn + phiếu thu (tùy chọn giữ/hủy) |

### 3.3 Trạng thái giao hàng (`shippingStatus`) — chỉ HD Giao hàng

Quan sát từ filter (7 trạng thái + 3 sub-tree):

```
        [Tạo HD Giao hàng]
                │
                ▼
            Chờ xử lý ◄───── (chờ NV pack & call carrier)
                │
                ▼
           ┌── Lấy hàng ──┐  (carrier đến lấy)
           │ ├─ Chờ lấy   │
           │ ├─ Lấy thành công │
           │ └─ Lấy thất bại    │
           ▼
        ┌── Giao hàng ──┐  (đang vận chuyển)
        │ ├─ Đang giao  │
        │ ├─ Giao thành công ──► Status: Hoàn thành
        │ └─ Giao thất bại     │
        ▼                       │
     ┌── Chuyển hoàn ──┐         │
     │ ├─ Đang chuyển hoàn       │
     │ └─ Đã chuyển hoàn ──► Auto-tạo TH (xem `tra-hang-deep-dive.md`)
     ▼                          │
    Đã hủy ◄───────────────────┘
```

→ 7 trạng thái shipping mapping 1:1 với carrier API → KiotViet pull realtime qua webhook hoặc poll.

### 3.4 Trạng thái HĐĐT (`eInvoiceStatus`) ⭐ Compliance VN

Quan sát từ dropdown filter — **2 enum riêng** cho 2 luồng:

**Enum A — Workflow phát hành HĐĐT:**
```
Chưa phát hành 
    → Đang xử lý 
    → Đã gửi CQT (Cơ quan thuế)
    → ┌─ CQT chấp nhận ──► Đã phát hành ──► Đã chuyển (gửi KH)
      └─ CQT kiểm tra không hợp lệ ──► Phát hành lỗi (cần sửa & gửi lại)
```

**Enum B — Workflow điều chỉnh/thay thế (sau khi đã phát hành):**
```
Chưa phát hành | Người mua yêu cầu | Đã gửi | Hợp lệ | Không hợp lệ | 
Bị từ chối | Đã hủy | Đã xác nhận | Đã xóa | Lỗi
```

→ Phù hợp **Thông báo sai sót, HĐ điều chỉnh, HĐ thay thế** (TT78/2021). Đây là moat compliance lớn — đối thủ ít làm đủ.

---

## 4. UI/UX MÀN HÓA ĐƠN

### 4.1 Layout

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Hóa đơn                              [+Tạo mới▼] [Import file] [Xuất file▼] [⚙]│
├──────────────────────────────────────────────────────────────────────────┤
│ Search "Theo mã hóa đơn"                                                  │
├──────────────────┬───────────────────────────────────────────────────────┤
│ FILTER SIDEBAR   │  Tổng     129,000     0      129...                   │
│                  │ ┌────────────────────────────────────────────────┐ │
│ Chi nhánh: [...] │ │ Mã HĐ│TG│ Mã trả│ Mã KH│Khách│TT hàng│Giảm│Sau│
│ Thời gian: ...   │ │ HD017065│19/05│ │ DUONG │DUONG │26,000│ 0│ 29,605│
│ Loại HĐ:         │ │ HD017064│19/05│ │ AQ2204│anh Quyết│103,000│0│ ...│
│  ☑ Không giao    │ └────────────────────────────────────────────────┘ │
│  ☑ Giao hàng     │                                                       │
│ Trạng thái:      │                                                       │
│  ☑ Đang xử lý    │                                                       │
│  ☑ Hoàn thành    │                                                       │
│  ☐ Không giao    │                                                       │
│  ☐ Đã hủy        │                                                       │
│ Trạng thái HĐĐT  │                                                       │
│ Trạng thái giao  │                                                       │
│ Đối tác giao:    │                                                       │
│ Thời gian giao:  │                                                       │
│ Khu vực giao:    │                                                       │
└──────────────────┴───────────────────────────────────────────────────────┘
```

### 4.2 Filter sidebar (≥10 tiêu chí)

| Filter | Kiểu | Use case |
|---|---|---|
| Chi nhánh | multi-select | Phân CN |
| Thời gian (tạo HĐ) | popup picker | History range |
| **Loại hóa đơn** | 2 checkbox (Không giao hàng / Giao hàng) | Tách POS vs Delivery |
| **Trạng thái hóa đơn** | 4 checkbox | Workflow filter |
| **Trạng thái HĐĐT** | dropdown — 10+ states | Compliance ops |
| **Trạng thái giao hàng** | dropdown — 7 states + sub | Shipping ops |
| Đối tác giao hàng | dropdown carrier | Per-carrier filter |
| Thời gian giao hàng | popup picker (có future-dated) | Đơn pre-order |
| Khu vực giao hàng | Tỉnh/TP → Quận/Huyện | Geo report |
| (scroll thêm) | ... | |

### 4.3 Top actions

| Button | Function |
|---|---|
| **Tạo mới ▼** | Dropdown 2 option: **Vận đơn KShip** \| **Hóa đơn** |
| **Import file** | Bulk import HD từ Excel |
| **Xuất file ▼** | Dropdown: File tổng quan / File chi tiết |
| ... | Bulk actions |
| ⚙️ | Cài đặt cột hiển thị |

### 4.4 Cột list (8+ cột mặc định)

| Cột | Ghi chú |
|---|---|
| Mã hóa đơn | HD###### — clickable expand |
| Thời gian | soldAt |
| **Mã trả hàng** | Link tới TH nếu có (1 HĐ có thể có N TH) |
| Mã KH | clickable Customer |
| Khách hàng | Tên |
| Tổng tiền hàng sa(u thuế) | Subtotal |
| Giảm giá | orderDiscount |
| Tổng sau gi(ảm) | Final total |

→ Có thêm cột tùy chọn (Trạng thái, HĐĐT, Vận đơn, Người bán, Kênh bán...) — toggle qua ⚙️.

---

## 5. CHI TIẾT 1 HÓA ĐƠN — INLINE PANEL

### 5.1 Header section

```
DUONG    HD017065 ↗    [An95 Store logo]    [Đang xử lý]              Chi nhánh 1
Người tạo: Admin       Người bán: [Admin▼]      Ngày bán: 19/05/2026 09:59 📅
Kênh bán: An95 Store   Bảng giá: Bảng giá chung
```

**Quan sát:**
- **An95 Store badge** — kênh bán có icon riêng (HĐĐT template? hoặc gian hàng MTĐ riêng)
- **Status badge "Đang xử lý"** màu cam — UI rõ status
- **↗ link** cạnh mã HĐ — "Mở phiếu" ra view riêng (tab mới hoặc full screen)

### 5.2 Section "Tự giao đến" (collapsible)

Khi HĐ Giao hàng — section này show address + shipping config (giống DH § 5.2). Carrier dropdown có badge KiotViet cho integration.

### 5.3 2 tab

| Tab | Nội dung |
|---|---|
| **Thông tin** | Lines + bottom totals + actions |
| **Lịch sử giao hàng** | Audit log shipping: Thời gian \| Mã vận đơn \| Đối tác giao hàng \| Người tạo \| Trạng thái |

→ Tab **Lịch sử giao hàng** là **shipping ledger** — track mọi sự kiện trạng thái vận đơn theo timeline.

### 5.4 Lines

Toggle hiển thị **before/after tax columns** — sample HD017065 hiển thị 7 cột "sau thuế":
```
| Mã hàng  | Tên hàng              | SL | Đơn giá ST | Giảm giá ST | Giá bán ST | Thành tiền ST |
| SP000006 | TH true milk dâu 110ml | 1  | 5,000      | 0           | 5,000      | 5,000         |
| SP000004 | Thuốc Mart đỏ          | 1  | 21,000     | 0           | 21,000     | 21,000        |
```

→ Schema thực ra có 11 cột (xem § 2.1) nhưng UI ẩn 4 cột "trước thuế" để gọn — toggle được.

### 5.5 Bottom totals (multi-rate VAT)

```
Tổng tiền hàng (2)        26,000
Giảm giá hóa đơn               0
VAT 0%                         0     ← Sữa (VAT 0%)
tets                           5     ← Thu khác custom
VAT                        2,600     ← Thuốc (VAT 10%)
Thu phí ship               1,000
Khách cần trả             29,605
Khách đã trả                   0
```

### 5.6 Action buttons (footer panel — 8 actions)

| Action | Function | Editable status |
|---|---|---|
| 🗑 **Hủy** | Hủy HĐ — popup hỏi giữ/hủy phiếu thu/HĐĐT | Mọi status (trừ Đã hủy) |
| ⧉ **Sao chép** | Tạo HĐ mới copy data | Mọi |
| 📥 **Xuất file** | Xuất Excel chi tiết | Mọi |
| ✏️ **Chỉnh sửa** (primary) | Mở màn Bán hàng để edit | Chỉ Đang xử lý — sửa HD đã hoàn thành phải hủy + tạo mới |
| 💾 **Lưu** | Save inline edit | Khi có change |
| 🔄 **Trả hàng** | Tạo phiếu TH từ HĐ này (link sourceInvoiceId) | Sau khi hoàn thành |
| 🖨 **In** | In HĐ với token templates ({Ma_QR}, {Ma_QR_Co_Thong_Tin_Tai_Khoan}) | Mọi |
| 📱 **Tạo QR** | Generate VietQR thu nợ (giống Customer tab Nợ) | Còn dư nợ |

⭐ Action **Trả hàng** ngay trên panel HĐ — UX rất tiện cho nhân viên thu ngân khi KH đem HĐ trả.

⭐ Action **Tạo QR** — sinh QR thanh toán phần dư nợ với template token (HĐ in ra có QR sẵn cho KH quét).

---

## 6. TÍNH NĂNG NÂNG CAO

### 6.1 Vận đơn KShip 

**KShip** = dịch vụ vận chuyển riêng của KiotViet (white-label hoặc partnership).
- Tạo trực tiếp từ menu "Tạo mới → Vận đơn KShip"
- Có **OTP authentication** (Help mục Giao vận → Thiết lập Xác thực OTP KShip)
- **Cấn trừ cước vận chuyển** với KiotViet wallet
- Đối soát đơn nhanh

### 6.2 Sao chép (Clone Invoice)

Use case:
- KH mua hàng định kỳ (vd shop văn phòng phẩm cho công ty) — clone HĐ tháng trước, sửa nhẹ
- Phục hồi HĐ đã hủy bằng cách clone

### 6.3 Import file Excel

Per KiotViet help:
- Mã HĐ prefix `HDIP` (HD Import) — phân biệt với HD bình thường
- Bắt buộc: Mã HĐ + Mã hàng + SL + Đơn giá
- Tự động link Customer cũ qua mã KH/SĐT
- Tự tạo phiếu thu nếu có thông tin thanh toán
- **KHÔNG re-validate tồn** — nguy hiểm, dễ làm âm tồn

### 6.4 Phát hành HĐĐT trên KV eInvoice

Workflow tách bạch (xem KV eInvoice menu HĐĐT - Thuế - Kế toán):
1. Tạo HĐ trên KV bình thường
2. Click "Phát hành HĐĐT" → KV gửi sang KV eInvoice
3. KV eInvoice gửi CQT (Cơ quan thuế qua API)
4. CQT phản hồi → status update
5. (Sai sót) Lập **Thông báo sai sót / HĐ điều chỉnh / HĐ thay thế**

→ Có **Bảng kê tổng hợp / điều chỉnh / thay thế** cho audit thuế.

### 6.5 Sửa HĐ — workflow đặc biệt

Theo help:
- Sửa HĐ ở status Đang xử lý → cho phép thay đổi hàng hóa, KH, bảng giá, người bán, thời gian, thanh toán
- Sửa HĐ Hoàn thành = **Hủy HĐ cũ + Tạo HĐ mới** (audit trail giữ)
- HD chứa Serial/Lô/HSD → KHÔNG sửa được thời gian
- HD đã phát hành HĐĐT → cần điều chỉnh qua KV eInvoice (không sửa trực tiếp)
- **Không re-validate tồn khi sửa** — pain lớn

### 6.6 Thu nợ qua QR

Theo help mục "Thu nợ hóa đơn qua mã QR":
- Tài khoản có quyền "Thanh toán khách hàng → Tạo Phiếu thu chi"
- Click "Tạo QR" trên chi tiết HĐ → chọn TK NH → nhập số tiền (gợi ý = Khách cần trả) → tải/copy QR gửi KH
- Nếu shop **đã đăng ký nhận thông báo QR** → auto-confirm + tạo phiếu thu ngay khi tiền vào
- Token in QR: `{Ma_QR}` và `{Ma_QR_Co_Thong_Tin_Tai_Khoan}` cho mẫu HĐ
- KH quét QR trực tiếp trên HĐ in → same workflow

---

## 7. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN

### 7.1 Workflow & State

| # | Pain | Đề xuất |
|---|---|---|
| 1 | "Sửa HĐ Hoàn thành" = Hủy + Tạo mới → audit chain gãy | True versioning (HD v2, v3 với link parent) |
| 2 | "Sửa HĐ không re-validate tồn" — dễ làm âm tồn | Re-check tồn khi đổi SL/SP |
| 3 | Status "Không giao được" tách Hoàn thành nhưng không có sub-workflow rõ ràng (re-ship vs refund vs cancel) | Decision tree UI |
| 4 | Bulk actions không có **bulk publish HĐĐT** trên màn list | Tích chọn nhiều HĐ → 1-click publish |
| 5 | Không có **alert SLA** khi HĐ "Đang xử lý" tồn quá lâu (vd > 7 ngày) | Aging report + alert |

### 7.2 Pricing & Tax

| # | Pain | Đề xuất |
|---|---|---|
| 6 | UI 11 cột giá rất rộng — không có **density toggle** | Compact/Comfortable view |
| 7 | "Thu khác" custom (`tets`) không hiện rõ là phí gì cho KH — chỉ name của user-defined | Thêm tooltip mô tả per loại |
| 8 | Đa thuế suất hỗ trợ nhưng UI chỉ show "VAT 0%" + "VAT" — confused khi có 3+ rates | Show rõ "VAT 0%, VAT 5%, VAT 8%, VAT 10%" |
| 9 | Không có **promo stacking** rõ trên HĐ — voucher + coupon + discount thủ công lẫn lộn | Breakdown discounts |
| 10 | Lãi gộp HĐ không hiển thị trên panel (cần module Phân tích) | Toggle "Show profit per HĐ" cho manager |

### 7.3 HĐĐT

| # | Pain | Đề xuất |
|---|---|---|
| 11 | 2 enum trạng thái HĐĐT (10+ states) confusing — không có **simple workflow visual** | Stepper UI: Draft → Sent → CQT Check → Issued |
| 12 | Khi CQT từ chối, không có **auto-retry với fix** | AI tự fix common error + retry |
| 13 | Mẫu HĐĐT không có **preview before publish** trên màn HĐ | Inline preview button |
| 14 | "Thông báo sai sót" workflow tách bạch — phải vào KV eInvoice riêng | Unified inline workflow |
| 15 | Multi-rate VAT trên 1 HĐ không có **breakdown summary** rõ ràng | Per-rate row trong totals |

### 7.4 Shipping

| # | Pain | Đề xuất |
|---|---|---|
| 16 | "Lịch sử giao hàng" tab chỉ liệt kê thô, không có **timeline visual** | Vertical timeline UI với icon trạng thái |
| 17 | Không có **carrier comparison** trên panel HĐ — phải đoán hãng nào rẻ/nhanh | Inline quote multi-carrier |
| 18 | KShip badge nhưng không thấy **cost saving estimate** | "Tiết kiệm 5k so với hãng khác" |
| 19 | Không có **proof of delivery** lưu (chữ ký KH, ảnh giao hàng) | Carrier API pull POD |

### 7.5 Customer & Loyalty

| # | Pain | Đề xuất |
|---|---|---|
| 20 | HĐ có thể tạo với "Khách lẻ" — mất chance loyalty | Auto-prompt nhập SĐT (đã đề cập ở customer module) |
| 21 | Sửa "Khách lẻ → có mã" sau khi tạo HĐ thì cần re-calculate loyalty | Background recompute |
| 22 | Không có **upsell suggestion** cuối HĐ ("KH thường mua thêm X") | AI recommendation engine |

### 7.6 In ấn & Mẫu

| # | Pain | Đề xuất |
|---|---|---|
| 23 | Mẫu in cấu hình ở module riêng — không preview ngay khi tạo HĐ | Inline preview "In thử với mẫu hiện tại" |
| 24 | Token `{Ma_QR}` static — không có conditional template | Logic templates (vd: hiện QR chỉ khi nợ > 0) |

---

## 8. CƠ HỘI ĐỘT PHÁ — TOP 5

| # | Tính năng | Pain | Effort | Impact |
|---|---|---|---|---|
| 1 | **HĐ Versioning + Audit Chain** (thay vì hủy+tạo mới khi sửa Hoàn thành) | Pain #1 — compliance + audit lớn | Cao | Rất cao |
| 2 | **Inline HĐĐT Workflow** (preview + publish + sửa sai sót inline, không phải vào KV eInvoice riêng) | Pain #11, #14 | Cao | Rất cao (compliance) |
| 3 | **Smart Carrier Quote + POD** (compare cost realtime, lưu chữ ký + ảnh giao) | Pain #17, #19 | Cao | Cao (seller online) |
| 4 | **Bulk HĐĐT Publish + Auto-fix CQT errors** | Pain #4, #12 | Trung | Rất cao |
| 5 | **HĐ Profit Insight** (lãi gộp per HĐ + upsell AI suggestion) | Pain #10, #22 | Trung | Cao (manager) |

---

## 9. ENTITY MODEL BỔ SUNG

Bổ sung vào danh sách entities (đã có #45 Invoice từ `ban-hang-deep-dive`):

| # | Đối tượng | Vai trò |
|---|---|---|
| 76 | **Invoice Header (HD######)** | Identity + status pipeline + multi-status (status/shipping/eInvoice) |
| 77 | **Invoice Line (11 cột giá)** | Per-line VAT rate + cost snapshot |
| 78 | **VAT Breakdown** | Aggregate VAT theo rate trên 1 HĐ (multi-rate support) |
| 79 | **Shipping History** | Audit log timeline cho mỗi shipping status transition |
| 80 | **E-Invoice (HĐĐT)** | Số HĐĐT + chuỗi điều chỉnh/thay thế + CQT response log |
| 81 | **KShip Voucher** | Vận đơn KShip riêng (carrier first-party của KV) |
| 82 | **Invoice Custom Surcharge** | Phí "Thu khác" user-defined gắn vào HĐ (`tets`...) |

---

## 10. SO SÁNH 3 PHIẾU LIÊN QUAN

| Khía cạnh | Đặt hàng (DH) | Hóa đơn (HD) | Trả hàng (TH) |
|---|---|---|---|
| Trừ tồn thật | ❌ (chỉ reserved) | ✅ | ✅ Hoàn lại |
| Ghi doanh thu | ❌ | ✅ | ❌ Giảm doanh thu |
| Tạo phiếu thu | Thu cọc (TTDH) | Thu đầy đủ (TTHD) | Chi (TTTH) |
| HĐĐT | ❌ | ✅ Có thể phát hành | Cần HĐ điều chỉnh |
| Shipping integration | ✅ | ✅ | (Auto từ chuyển hoàn) |
| Loyalty | ❌ | ✅ +points | ✅ −points |
| Customer balance | ❌ | += dư nợ | −= dư nợ |
| Multi-rate VAT | ✅ | ✅ | Per HĐ gốc |
| Editable sau tạo | ✅ (rich) | Limited (chỉ Đang xử lý) | Limited |
| Auto-cancel sau N ngày | (đề xuất) | ❌ | N/A |

---

## 11. TÓM LƯỢC

**Hóa đơn KiotViet:**
- Là **trung tâm** của 7 luồng dữ liệu (Customer, Sổ quỹ, Thẻ kho, HĐĐT, Vận chuyển, Loyalty, AR)
- **Schema cực phong phú:** 11 cột giá per line với multi-rate VAT (0/5/8/10%), 4 status enum tách bạch (HD status, shipping status, eInvoice status A, eInvoice status B), shipping block đầy đủ với carrier integration + KShip
- **State machine 3 lớp đồng thời:**
  - HĐ status: Đang xử lý / Hoàn thành / Không giao được / Đã hủy
  - Shipping status: 7 states + sub-states (Chờ → Lấy → Giao → Chuyển hoàn → Đã chuyển hoàn / Đã hủy)
  - HĐĐT status: 2 enum riêng (Phát hành workflow + Điều chỉnh workflow)
- **Mạnh:** Multi-rate VAT trên 1 HĐ, HĐĐT compliance đầy đủ (TT78/2021), KShip first-party carrier, Tạo QR thu nợ với token in HĐ, action **Trả hàng** + **Tạo QR** ngay trên panel, 2 tab (Thông tin + Lịch sử giao hàng), lịch sử shipping audit log, kênh bán có badge logo (An95 Store)
- **Yếu:** Sửa HĐ Hoàn thành = Hủy + Tạo mới (gãy audit chain), không re-validate tồn khi sửa, status "Không giao được" thiếu sub-workflow, HĐĐT workflow tách bạch khỏi HD (phải vào KV eInvoice), không bulk publish HĐĐT, không SLA alert, không carrier comparison inline
- **Cơ hội đột phá:** HĐ Versioning, Inline HĐĐT workflow, Smart Carrier Quote + POD, Bulk HĐĐT publish + auto-fix CQT errors, Profit Insight + AI upsell

Đây là module **cực kỳ phức tạp đã được KiotViet đầu tư rất sâu (đặc biệt HĐĐT compliance), còn dư địa cho UX simplification (audit chain, inline workflow), intelligence (smart quote, upsell), và operations efficiency (bulk publish, SLA alert) — match đúng với segment chuyên nghiệp đang là moat lớn của KiotViet**.
