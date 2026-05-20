# ĐÀO SÂU: MODULE ĐẶT HÀNG / ĐƠN HÀNG — KIOTVIET

**Phạm vi:** Đặt hàng (Customer Order, prefix `DH`) là **phiếu cam kết bán** từ khách hàng — KH đã chốt mua nhưng **chưa xuất hóa đơn**. Đây là buffer giữa "ý định mua" và "doanh thu đã ghi nhận".

**Vị trí UI:** `/man/#/Orders` (menu top-bar "Đơn hàng" — note: URL `Orders` nhưng UI label "Đặt hàng")
**Mã prefix:** `DH######` (6 digits sequential per merchant)
**Test data:** 4 đơn DH001689 → DH001808 (5 tháng span, status đa dạng)

---

## 1. KHÁI NIỆM & PHÂN BIỆT VỚI MODULE LIÊN QUAN

```
┌──── Customer Journey ────────────────────────────────────────────────┐
│                                                                       │
│  KH inquire  →  KH chốt mua  →  Chuẩn bị giao  →  Khách nhận  →  Done│
│      ─              │              │                  │               │
│      ─        [Đặt hàng DH]  ►  [Hóa đơn HD]  ►  Đã giao             │
│                     │              │                                  │
│                  reserved      −onHand, +totalSale                    │
│                  +Đặt KH       +doanh thu thật                        │
└───────────────────────────────────────────────────────────────────────┘
```

### 1.1 DH vs HD — ranh giới quan trọng

| Khía cạnh | Đặt hàng (DH) | Hóa đơn (HD) |
|---|---|---|
| Khi nào tạo | KH chốt mua, chưa xuất kho | KH nhận hàng / cash transaction |
| Tác động tồn kho | **KHÔNG giảm tồn** — chỉ +`reservedByCustomer` | **GIẢM tồn** thật (`onHand`) |
| Doanh thu | Chưa ghi vào P&L | Ghi vào `Customer.totalSale` + sổ kế toán |
| Thanh toán | Có thể cọc 1 phần (Phiếu thu TTDH) | Đầy đủ (Phiếu thu TTHD) |
| Có thể hủy không tác động doanh thu | ✅ Hủy thẳng | ❌ Cần Trả hàng (TH) |
| HĐĐT phát hành | Chưa | Có thể phát hành sau khi tạo HD |

→ **DH là "soft commitment"**, HD là "hard transaction". Đây là pattern chuẩn của e-commerce (Order vs Invoice) — KiotViet implement tốt.

### 1.2 DH vs DHN (Đặt hàng nhập)

**Đừng nhầm:** `DH` (Customer Order) vs `DHN` (Purchase Order, đặt hàng từ shop tới NCC, xem `nhap-hang-deep-dive.md` § 1). Cùng pattern "phiếu cam kết" nhưng đối tác ngược chiều.

---

## 2. CẤU TRÚC DỮ LIỆU (SCHEMA)

### 2.1 Entity chính — `CustomerOrder`

```
CustomerOrder
├── Header
│   ├── code               string   DH###### (sequential per merchant)
│   ├── customerId         FK       Bắt buộc (không có "Khách lẻ" cho DH)
│   ├── branchId           FK       CN xử lý
│   ├── orderedAt          datetime Ngày đặt
│   ├── orderedById        FK User  Người tạo (Người nhận đặt)
│   ├── salesChannelId     FK       Kênh bán (Bán trực tiếp / Shopee / TikTok / ...)
│   ├── priceBookId        FK       Bảng giá áp dụng (default = Bảng giá chung)
│   ├── status             enum     Phiếu_tạm | Đang_giao_hàng | Hoàn_thành | Đã_hủy | (1 khác)
│   ├── note               string   Ghi chú free text
│   ├── createdAt / updatedAt / canceledAt
│   ├── canceledById       FK User?
│   │
│   ├── Pricing (đầy đủ trước+sau thuế)
│   │   ├── subtotal            decimal   Tổng tiền hàng (sum line totals)
│   │   ├── discountOrder       decimal   Giảm giá tổng phiếu
│   │   ├── customSurcharges    decimal   "Thu khác" tự định nghĩa (vd "tets" = 5)
│   │   ├── vat                 decimal   Tổng VAT (10% ở sample)
│   │   ├── shippingFee         decimal   Phí ship "Thu phí ship"
│   │   └── total               decimal   Tổng cộng (khách cần trả)
│   │
│   ├── Payment
│   │   ├── paidAmount          decimal   Khách đã trả (cọc)
│   │   └── balance             decimal   total − paidAmount (còn nợ)
│   │
│   └── Shipping (optional — bật/tắt section "Tự giao đến")
│       ├── deliveryAddress    string
│       ├── deliveryProvince   string
│       ├── deliveryWard       string
│       ├── pickupAddress      string    Địa chỉ lấy hàng (warehouse)
│       ├── recipientName      string
│       ├── recipientPhone     string
│       ├── trackingCode       string    Mã vận đơn (do hãng VC cấp)
│       ├── weight             number    g
│       ├── dimensions         {x,y,z}   cm
│       ├── declaredValue      bool      Khai giá
│       ├── service            enum      "Siêu tốc ưu tiên" / ...
│       ├── carrier            enum      "Grab" / "Viettel Post" / "GHTK" / ...
│       ├── carrierIntegration enum      "KiotViet" (1-click sync) | manual
│       ├── isCOD              bool      Thu hộ tiền
│       ├── shippingFeePayer   enum      Khách / Shop trả ĐTGH
│       └── expectedDeliveryAt datetime
│
└── Lines (1..N) — sản phẩm trên đơn
    ├── productId         FK
    ├── variantId         FK?
    ├── unitId            FK UoM
    ├── quantity          decimal  ⭐ Hiển thị "1/0" = đã đặt 1 / đã xuất 0 (fulfillment)
    ├── fulfilledQty      decimal  Số đã xuất sang HĐ
    ├── unitPrice         decimal  Đơn giá (trước thuế)
    ├── unitPriceAfterTax decimal  Đơn giá sau thuế
    ├── discount          decimal  Giảm giá dòng (trước thuế)
    ├── discountAfterTax  decimal
    ├── salePrice         decimal  Giá bán (sau giảm, trước thuế)
    ├── salePriceAfterTax decimal
    ├── lineTotal         decimal  Thành tiền (trước thuế)
    └── lineTotalAfterTax decimal  Thành tiền sau thuế
```

### 2.2 Schema line đặc biệt phong phú — 11 cột giá

KiotViet tách bạch **trước thuế / sau thuế** cho 4 trường giá: Đơn giá / Giảm giá / Giá bán / Thành tiền → 8 cột giá + 3 cột (Mã hàng, Tên, Số lượng) = **11 cột** trên line item.

→ Match yêu cầu **HĐĐT VAT** (NĐ123/2020 + TT78/2021) — kế toán có thể bóc tách rõ giá net vs gross.

### 2.3 Liên kết với module khác

```
CustomerOrder (DH######)
    │
    ├── customerId ──► Customer
    │                  - reservedByCustomer += Σ Line.quantity (sau quy đổi UoM)
    │                  - lastOrderAt update
    │
    ├── (khi paidAmount > 0) ──► CashVoucher (TTDH###### — Phiếu thu cọc)
    │                              hạch toán KQKD = false (chưa phải doanh thu)
    │
    ├── (khi "Xử lý đơn hàng") ──► Invoice (HD######)
    │                              Lines copy → HD, fulfilledQty += quantity
    │                              status = Đang_giao_hàng hoặc Hoàn_thành
    │
    ├── (nếu có shipping) ──► ShippingOrder (vận đơn)
    │                          - carrier API (Grab/Viettel/...)
    │                          - trackingCode sync 2-way
    │
    └── (nếu hủy) ──► canceledAt, reservedByCustomer giảm lại
```

---

## 3. STATE MACHINE — VÒNG ĐỜI ĐẶT HÀNG

```
                     [Tạo DH]
                         │
                         ▼
                  ┌─── Phiếu tạm ──┐
                  │                 │ [Sửa]
                  │ [Xử lý đơn hàng]│ (giữ ở phiếu tạm)
                  │ → POS Bán hàng  │
                  ▼                 │
              Đang giao hàng        │
              (đã tạo HD,           │
               chờ KH nhận)          │
                  │                 │
                  ├─ [Đã giao] ───► Hoàn thành ◄─────────┘
                  │                                       │
                  └─ [Chuyển hoàn] ─► (Auto TH) ──────►  │
                                                          │
                                       ┌──────[Hủy]──────┘
                                       ▼
                                   Đã hủy
                                   (immutable, hiện trong list khi filter)
```

### 3.1 4 trạng thái tường minh (+1 ẩn?)

Trên filter UI có **5 trạng thái** ("Phiếu tạm", "Đang giao hàng", "Hoàn thành" + "+1 khác"):

| Trạng thái | Hành vi | Tác động |
|---|---|---|
| **Phiếu tạm** | Mới tạo, chưa xử lý — có thể sửa lines, sửa khách, sửa shipping | reservedByCustomer += qty |
| **Đang giao hàng** | Đã "Xử lý đơn hàng" → tạo HD + ship qua carrier | HD trừ tồn, vận đơn pending |
| **Hoàn thành** | KH đã nhận, transaction closed | HD ở status Hoàn thành |
| **Đã hủy** | Hủy phiếu | reservedByCustomer −= qty, các phiếu liên quan hủy hoặc giữ tùy lựa chọn |
| **+1 khác** | Có thể là "Tạm dừng" / "Chờ xử lý" — chưa confirm | — |

### 3.2 Tác động tồn theo trạng thái

| Trạng thái | onHand | reservedByCustomer | Doanh thu |
|---|---|---|---|
| Phiếu tạm | — | ✅ +qty | — |
| Đang giao hàng | ✅ −qty (qua HD) | −qty (chuyển sang HD) | ✅ +totalSale (HD) |
| Hoàn thành | — | — | — (đã ghi ở HD) |
| Đã hủy | (rollback HD nếu có) | −qty | (rollback nếu có HD) |

→ **DH chỉ là reservation soft**, không trừ tồn thật. Tồn chỉ giảm khi state machine chuyển sang HD.

---

## 4. UI/UX MÀN ĐẶT HÀNG

### 4.1 Layout list

```
┌──────────────────────────────────────────────────────────────────────────┐
│ Đặt hàng                              🔄¹ [+Đặt hàng] [Gộp đơn] [...] ⚙ │
├──────────────────────────────────────────────────────────────────────────┤
│ Search "Theo mã phiếu đặt"                                                │
├──────────────────┬───────────────────────────────────────────────────────┤
│ FILTER SIDEBAR   │  Tổng     280,000   312,020      0                    │
│                  │ ┌──────────────────────────────────────────────────┐ │
│ Chi nhánh: [...] │ │ Mã đặt│ TG │ Mã KH │ Khách │ Tiền hàng │ Cần trả │ Đã trả │ TT │
│ Thời gian: ...   │ │ DH001808│09/12│KH...723│binhdt│ 10,000│12,005│ 0│Phiếu tạm│
│ Trạng thái:      │ │ DH001807│09/12│KH...723│binhdt│ 10,000│12,005│ 0│Phiếu tạm│
│  ☑ Phiếu tạm    │ │ DH001785│21/07│ABC126 │Thuy Pham│220,000│243,005│0│Phiếu tạm│
│  ☑ Đang giao    │ │ DH001689│10/02│AQ2204 │anh Quyết│ 40,000│ 45,005│0│Hoàn thành│
│  ☑ Hoàn thành   │ └──────────────────────────────────────────────────┘ │
│  +1 khác        │                                                       │
│ Đối tác giao:    │                                                       │
│ Thời gian giao:  │                                                       │
│ Khu vực giao:    │                                                       │
│ Phương thức TT:  │                                                       │
│ VAT (%):         │                                                       │
│ Người tạo:       │                                                       │
│ Người nhận đặt:  │                                                       │
└──────────────────┴───────────────────────────────────────────────────────┘
```

### 4.2 Filter sidebar (10 tiêu chí)

| Filter | Kiểu | Use case |
|---|---|---|
| Chi nhánh xử lý | multi-select | Tách CN |
| Thời gian | popup picker (Hôm nay/qua, 7/30 ngày qua, Tuần, Tháng, Quý, Năm + âm lịch) — **không có "Toàn thời gian"** | Tạo đơn |
| **Trạng thái** | multi-select (5 trạng thái) | Workflow filter |
| **Đối tác giao hàng** | dropdown carrier | Theo hãng VC |
| **Thời gian giao hàng** | popup picker — có **Ngày mai, Tuần sau, Tháng tới, Quý sau, Toàn thời gian** | **Future-dated** — đơn giao tương lai |
| Khu vực giao hàng | Tỉnh/TP → Quận/Huyện | Geo planning ship |
| Phương thức thanh toán | multi-select | Filter cash/card/QR |
| **VAT (%)** | dropdown | Đối soát thuế |
| Người tạo | dropdown user | Audit |
| Người nhận đặt | dropdown user | KAM portfolio |

⭐ **Filter "Thời gian giao hàng" có Ngày mai / Tuần sau / Tháng tới / Quý sau** — feature đặc trưng của module Order (vs các module khác chỉ có past) → đáp ứng use case "đơn giao tương lai" (pre-order, đặt trước event).

### 4.3 Top actions

| Button | Function | Ghi chú |
|---|---|---|
| 🔄¹ (badge 1) | **Đồng bộ đơn từ kênh online** | Pull đơn từ Shopee/TikTok/Lazada về |
| **+ Đặt hàng** | Tạo DH mới → mở màn POS dạng đơn đặt | |
| **Gộp đơn** | ⭐ Merge nhiều DH cùng KH thành 1 | Use case: KH order 3 lần liên tiếp → gộp ship chung |
| ... | Bulk actions (xuất file, xử lý, hủy hàng loạt) | |
| ⚙️ | Cài đặt cột hiển thị | |

### 4.4 Cột list (8 cột mặc định)

| Cột | Ghi chú |
|---|---|
| Mã đặt hàng | DH###### |
| Thời gian | Ngày đặt |
| Mã KH | clickable → mở Customer |
| Khách hàng | Tên |
| Tổng tiền hàng sa(u thuế?) | Truncated cột name "Tổng tiền hàng sa..." |
| Khách cần trả | Total sau VAT + ship |
| Khách đã trả | Cọc đã thu |
| Trạng thái | Badge màu (Phiếu tạm cam, Hoàn thành xanh, Đang giao xanh dương, Đã hủy đỏ) |

Footer row: Aggregate tổng từng cột numeric.

---

## 5. CHI TIẾT 1 ĐƠN HÀNG — INLINE PANEL

Click row → expand panel ngay dưới (giống Customer module).

### 5.1 Header section

```
[Mở phiếu↗]  Thuy Pham      DH001785                              Chi nhánh 1
Người tạo: Admin   Người nhận đặt: [Admin▼]   Ngày đặt: 21/07/2025 13:47 📅
Kênh bán: [Bán trực tiếp▼]   Bảng giá: Bảng giá chung
Chi nhánh xử lý: Chi nhánh 1   Trạng thái: [Phiếu tạm▼]
```

**Field editable trên detail panel:** Người nhận đặt, Ngày đặt, Kênh bán, Trạng thái (dropdown), Bảng giá. → Cập nhật nhanh không cần mở form riêng.

### 5.2 Section "Tự giao đến" (collapsible)

Section có checkbox để bật/tắt — bật thì hiện full shipping form:

```
☑ Tự giao đến: 87 Dương Khuê, Phường Mai Dịch, Cầu Giấy, Hà Nội
🏠 Địa chỉ lấy hàng: 1A Yết Kiêu, Hoàn Kiếm, Hà Nội  -  +84 991 456 788

Người nhận:    Hoa Lan           Mã vận đơn:  (—)              Người giao:  [Grab • KiotViet▼]
Điện thoại:    0965481273        Trọng lượng: [20] [g▼]         Thu hộ tiền: ☑ (COD)
Địa chỉ:       87 Dương Khuê...  Kích thước:  [10][11][12] cm  Phí trả ĐTGH ⓘ: 45,000
Khu vực:       HN — Cầu Giấy     Khai giá:    ☐                 Thời gian giao: 📅 📍
Phường/Xã:     Mai Dịch          Dịch vụ:     Siêu tốc ưu tiên

💡 Địa chỉ trên đã thay đổi sau khi sáp nhập ngày 01/07/2025.
   Bạn có muốn đổi địa chỉ mới: Phường Nghĩa Đô - Thành phố Hà Nội?  [Đổi địa chỉ]
```

**Quan sát quan trọng:**
- **Carrier integrated** — dropdown "Người giao" có Grab/Viettel/GHN/... với badge "KiotViet" cho hãng đã tích hợp API
- **COD** flag tách bạch — "Phí trả ĐTGH" (Đối Tác Giao Hàng) là phí ship, có thể khác Phí ship trên tổng (do KH trả vs shop trả)
- **Khai giá** — bảo hiểm hàng (declared value) cho carrier
- **Banner địa chỉ hành chính mới** giống Customer module — proactive UX
- **Phí trả ĐTGH có tooltip ⓘ** — giải thích cách tính

### 5.3 Section Lines

```
| Mã hàng  | Tên hàng | Số lượng | Đơn giá  | Đơn giá ST | Giảm giá | Giảm giá ST | Giá bán  | Giá bán ST | Thành tiền | Thành tiền ST |
| SP000413 | a test   | 1/0      | 220,000  | ...        | 0        | ...         | 220,000  | ...        | 220,000    | ...           |
```

→ **"1/0"** trong cột Số lượng = "đã đặt 1 / đã xuất 0" — **fulfillment tracking**! Khi đơn được xử lý qua HD, số này thành "1/1".

### 5.4 Bottom totals

```
Tổng tiền hàng (1)         220,000
Giảm giá phiếu đặt              0
tets *                         5    ← Thu khác custom (user-defined fee)
VAT                       22,000    ← 10%
Thu phí ship               1,000
Tổng cộng                243,005    ← Khách cần trả
Khách đã trả                   0
```

**(1)** sau "Tổng tiền hàng" = số dòng line items.
**`tets`** là loại "Thu khác" mà user tự tạo (giống Sổ quỹ § 4) — gắn vào đơn hàng làm phụ phí.

### 5.5 Action buttons (footer panel)

| Button | Function |
|---|---|
| 🗑 **Hủy** | Hủy phiếu (popup hỏi giữ/hủy phiếu thu cọc liên quan) |
| ⧉ **Sao chép** | Tạo DH mới với data copy (lines + KH + shipping) |
| 📥 **Xuất file** | Xuất Excel chi tiết phiếu |
| ✅ **Xử lý đơn hàng** (primary) | ⭐ Mở màn POS Bán hàng để chốt → tạo HD |
| 💾 **Lưu** | Lưu thay đổi trong inline edit |
| 🏁 **Kết thúc** | Đóng panel |
| 🖨 **In** | In phiếu |

⭐ **"Xử lý đơn hàng"** là transition core: bấm → tab mới `/sale/#/` mở với cart pre-fill từ DH. NV thu ngân finalize → tạo HD.

---

## 6. TÍNH NĂNG NÂNG CAO

### 6.1 Đồng bộ đơn online (🔄 badge)

Top-right có icon refresh với badge số. Click → pull đơn từ marketplace đã kết nối (Shopee, TikTok Shop, Lazada, Tiki, FB Shop, Zalo Shop).

→ Đơn từ marketplace tạo DH tự động với:
- `salesChannelId` = kênh nguồn
- Shipping info lấy từ marketplace API
- Lines map qua mapping SP đã setup (xem `02-hang-hoa-kho/tong-quan` § D1 — Liên kết kênh bán)

### 6.2 Gộp đơn (Merge Orders)

Button **"Gộp đơn"** ở top — chọn nhiều DH cùng KH → merge thành 1.

Use case: KH order 3 lần liên tiếp trong 30 phút → NV gộp để ship 1 vận đơn, tiết kiệm phí ship 2/3.

→ Gộp khả năng tạo 1 DH mới hoặc update DH gốc — chưa rõ behavior, cần test thêm.

### 6.3 Carrier Integration (KiotViet badge)

Dropdown "Người giao" có 2 nhóm:
- **Có KiotViet badge** = tích hợp API 2-way (Grab, GHTK, Viettel Post, GHN, J&T, AhaMove...)
  - Auto-create vận đơn
  - Auto-sync trạng thái (Đang giao → Đã giao → Chuyển hoàn)
  - Phí ship pull realtime
- **Không có badge** = ghi nhận thủ công (manual tracking)

→ Đây là moat lớn của KiotViet — tích hợp 6-8 carrier VN ngay trong DH form.

### 6.4 Auto Trả hàng khi chuyển hoàn

Nếu carrier báo "Đã chuyển hoàn" qua API → KiotViet tự tạo TH (Trả hàng bán, xem `02-hang-hoa-kho/tra-hang-deep-dive` § 2.4 case #5). DH gốc chuyển sang Hoàn thành (trừ hàng đã trả).

→ Phiếu TH **auto này không cho phép hủy** (sync chặt với carrier).

---

## 7. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN

### 7.1 Workflow & State Machine

| # | Pain | Đề xuất |
|---|---|---|
| 1 | "+1 khác" trong filter trạng thái không rõ là gì (có thể "Tạm dừng" / "Chờ xác nhận") — UX confusing | Hiển thị đầy đủ tất cả trạng thái |
| 2 | Không có **"Đã xác nhận"** giữa Phiếu tạm và Đang giao — KAM chưa kịp confirm KH đã chốt | Thêm state "Confirmed" |
| 3 | Không có **partial fulfillment** thực sự — line "1/0" hiển thị nhưng không cho split (đặt 10 cái, ship 7 trước 3 sau) | Multi-shipment per DH |
| 4 | Hủy DH không có **reason taxonomy** — chỉ free note | Dropdown: KH đổi ý / Hết hàng / Giá sai / Khác |
| 5 | Không có **deadline / promised delivery** mà tự alert khi sắp trễ | SLA tracker + alert |

### 7.2 Pricing & Tax

| # | Pain | Đề xuất |
|---|---|---|
| 6 | 11 cột giá phong phú nhưng UI **rộng quá** trên mobile/POS | Toggle nhanh "Hide tax columns" |
| 7 | "Thu khác" custom (vd `tets`) không có **rule auto-apply** | Rule engine: "đơn > 200k thì auto +1k tip" |
| 8 | Giảm giá phiếu đặt không có **multi-tier promo stacking** (KH VIP + voucher code + giảm thêm 5%) | Promo engine với priority |
| 9 | VAT chỉ 1 mức / phiếu — đơn có lẫn SP VAT 0%/5%/10% phải tính tay hoặc nhiều dòng | Per-line VAT rate (đã có ở schema nhưng UI chưa rõ) |

### 7.3 Shipping & Carrier

| # | Pain | Đề xuất |
|---|---|---|
| 10 | Chỉ 1 vận đơn / DH — đơn lớn nhiều thùng phải tách | Multi-package (kiotviet đã có "Đơn giao hàng nhiều kiện" — cần verify integrated cho DH) |
| 11 | "Phí trả ĐTGH" tách với "Thu phí ship" — confuse | Diagram giải thích trực quan |
| 12 | Không có **rule auto-chọn carrier** theo region/weight/cost | Smart routing engine (đã đề xuất ở brainstorm B2) |
| 13 | Không có **bulk vận đơn create** trên multi DH chọn | Bulk action với carrier |
| 14 | Không có **cost compare** giữa carriers cho 1 đơn | Realtime quote multi-carrier |

### 7.4 Đồng bộ kênh online

| # | Pain | Đề xuất |
|---|---|---|
| 15 | Refresh manual qua icon 🔄 — không realtime push webhook | Push-based sync via marketplace webhook |
| 16 | Mapping SP đôi khi miss → DH tạo nhưng line product mismatch | Auto-fuzzy match + alert manual review |
| 17 | Không có **conflict resolution** khi 1 đơn TMĐT update sau khi đã edit ở KV | Workflow merge/override |

### 7.5 Gộp đơn

| # | Pain | Đề xuất |
|---|---|---|
| 18 | Gộp đơn không có **preview** trước khi commit | Show "Đơn mới sẽ thế này" trước save |
| 19 | Không thấy có rule "gộp tự động" (vd cùng KH + cùng địa chỉ + trong 24h) | Auto-merge rule |

---

## 8. CƠ HỘI ĐỘT PHÁ — TOP 5 CHO ĐẶT HÀNG

| # | Tính năng | Pain | Effort | Impact |
|---|---|---|---|---|
| 1 | **Smart Carrier Routing** — auto-pick carrier rẻ/nhanh nhất per đơn | Pain #12, #14 | Cao | Rất cao |
| 2 | **Multi-shipment Fulfillment** — split DH thành nhiều ship batch | Pain #3, #10 | Trung | Cao (B2B) |
| 3 | **Order Confirmation Workflow** — state "Confirmed" + auto ZNS xác nhận KH | Pain #2, #5 | Thấp | Cao |
| 4 | **Promo Stacking Engine** — multi-tier discount với priority rule | Pain #8 | Cao | Cao |
| 5 | **Webhook-based Channel Sync** — realtime push thay refresh thủ công | Pain #15 | Trung | Rất cao (seller online) |

---

## 9. ENTITY MODEL BỔ SUNG

Bổ sung vào danh sách entities (đã có #46 CustomerOrder + #47 Reservation từ `ban-hang-deep-dive` § 8):

| # | Đối tượng | Vai trò |
|---|---|---|
| 69 | **CustomerOrder Header** | Prefix DH, link tới Customer, Pricing block, Payment block, Shipping block |
| 70 | **OrderLine (11 trường giá)** | Trước thuế + Sau thuế cho 4 trường price |
| 71 | **OrderShipping** | Embedded shipping block — carrier, COD, dimension, declared value |
| 72 | **OrderCustomCharge** | "Thu khác" custom user-defined (`tets`...) |
| 73 | **OrderFulfillment** | Tracking quantity ordered vs fulfilled per line |
| 74 | **CarrierIntegration** | Hãng VC + tích hợp API badge "KiotViet" |
| 75 | **OrderMergeOperation** | Merge nhiều DH thành 1 — audit log |

---

## 10. SO SÁNH VỚI CHUẨN E-COMMERCE

| Khía cạnh | KiotViet DH | Shopify Order | WooCommerce Order |
|---|---|---|---|
| Order ≠ Invoice | ✅ | ✅ | Partial (1 đơn = 1 transaction) |
| Multi-status workflow | ⚠️ 4-5 states cơ bản | ✅ Rich states (pending → confirmed → fulfilled → completed) | ✅ |
| Fulfillment tracking | ⚠️ "1/0" UI nhưng chưa multi-batch | ✅ Multi-fulfillment | ✅ |
| Carrier integration | ✅ 6-8 carrier VN | ✅ via app | ✅ via plugin |
| Channel sync (marketplace) | ✅ Manual refresh + auto map | ✅ Shopify Markets | ✅ via plugin |
| Multi-currency | ❌ | ✅ | ✅ |
| Subscription/recurring | ❌ | ✅ via app | ✅ via plugin |
| Backorder | ❌ (cho phép âm tồn thôi) | ✅ Pre-order | ✅ |

→ KiotViet **mạnh ở local market** (carrier VN, HĐĐT, địa danh hành chính mới) nhưng **yếu ở enterprise e-commerce features** (subscription, multi-currency, advanced fulfillment).

---

## 11. TÓM LƯỢC

**Đặt hàng KiotViet:**
- Là **soft commitment layer** tách bạch với HD — KH chốt mua nhưng chưa giảm tồn thật, chỉ +reservedByCustomer
- Schema rất chi tiết: **11 cột giá** per line (4 trường × {trước, sau} thuế + 3 base), shipping block đầy đủ với carrier integration, fulfillment tracking "1/0"
- State machine 4-5 trạng thái (Phiếu tạm → Đang giao → Hoàn thành; nhánh hủy & chuyển hoàn auto)
- **Mạnh:** Tích hợp 6-8 carrier VN với badge "KiotViet" (Grab/GHN/Viettel/J&T/AhaMove...), filter "Thời gian giao hàng" có **Ngày mai/Tuần sau/Quý sau** (future-dated unique), đồng bộ kênh online (Shopee/TikTok/Lazada/Tiki/FB/Zalo), Gộp đơn, auto-trả khi chuyển hoàn, banner địa danh hành chính mới sau 01/07/2025
- **Yếu:** Không có order confirmation state, không có multi-shipment split, không có promo stacking, không có smart carrier routing, sync marketplace là pull manual chứ không phải push webhook
- **Cơ hội đột phá:** Smart Carrier Routing, Multi-shipment Fulfillment, Order Confirmation Workflow + Auto ZNS, Promo Stacking Engine, Webhook-based realtime sync

Đây là module **đã có nền tảng vững và moat tích hợp carrier VN, còn nhiều dư địa cho workflow nâng cao (multi-shipment, confirmation, promo) và realtime sync — match đúng segment seller online đa kênh + chuỗi có giao hàng nhiều của KiotViet**.
