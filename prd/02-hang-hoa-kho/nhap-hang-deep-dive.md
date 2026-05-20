# ĐÀO SÂU: NGHIỆP VỤ NHẬP HÀNG — KIOTVIET

**Bối cảnh:** Nhập hàng (Goods Receipt / Purchase) là nghiệp vụ **đưa hàng từ NCC vào kho** — đầu nguồn của mọi tồn kho và cơ sở để tính giá vốn. Cùng với "Đặt hàng nhập" (Purchase Order) và "Trả hàng nhập", 3 phiếu này tạo thành cụm **Mua hàng** trong KiotViet.

**Tham chiếu:**
- Help chính thức: `kiotviet.vn/huong-dan-su-dung-kiotviet/retail-giao-dich/nhap-hang/`
- Vị trí UI hiện tại: `/man/#/Imports` (menu Hàng hóa → Nhập hàng). KiotViet đang chuyển sang **Giao diện 2026**, các mục Nhập hàng / Đặt hàng nhập / Trả hàng nhập / NCC / Hóa đơn đầu vào sẽ chuyển vào menu **Mua hàng** — nghiệp vụ không đổi.

---

## 1. VỊ TRÍ TRONG HỆ SINH THÁI

```
[Đặt hàng nhập (DHN)] ──tham chiếu──►  [Nhập hàng (PN)] ──tham chiếu──►  [Trả hàng nhập (TPN)]
       optional                              core                                optional
          │                                    │
          │                                    ├──► Tăng tồn (Stock per Branch)
          │                                    ├──► Tăng công nợ phải trả NCC (nếu chưa TT đủ)
          │                                    ├──► Cập nhật giá vốn (Bình quân gia quyền)
          │                                    ├──► Ghi 1 dòng Thẻ kho (PN######)
          │                                    └──► (nếu bật) Phân bổ Chi phí nhập hàng vào giá vốn
          │
          └──► Không tác động tồn — chỉ ghi nhận "Đặt NCC" trên product detail
```

3 phiếu trong cụm Mua hàng:

| Phiếu | Prefix | Tác động tồn | Vai trò |
|---|---|---|---|
| **Đặt hàng nhập** | DHN | Không (chỉ "Đặt NCC") | PO — cam kết với NCC, deadline giao hàng |
| **Nhập hàng** | PN | ✅ Tăng tồn | Goods receipt — hàng đã về kho thực tế |
| **Trả hàng nhập** | TPN | ✅ Giảm tồn | Trả NCC khi lỗi/sai quy cách |

---

## 2. CẤU TRÚC DỮ LIỆU (SCHEMA)

### 2.1 Entity chính — `PurchaseImport` (Phiếu Nhập hàng)

```
PurchaseImport
├── Header
│   ├── code              string   PN######, sequential per merchant
│   ├── branchId          FK       CN nhập hàng
│   ├── supplierId        FK?      NCC (có thể null khi mới tạo)
│   ├── importedAt        datetime Ngày nhập (cho phép backdate)
│   ├── importedBy        FK User  Người nhập (mặc định = người tạo)
│   ├── status            enum     Phiếu_tạm | Đã_nhập_hàng | Đã_hủy
│   ├── discountAmount    decimal  Giảm trên tổng phiếu
│   ├── totalCost         decimal  ∑ lines (sau giảm)
│   ├── paidAmount        decimal  Tiền đã trả NCC lần này
│   ├── debtAmount        decimal  totalCost - paidAmount
│   ├── note              string
│   ├── poId              FK?      Link tới Đặt hàng nhập (DHN) nếu có
│   ├── eInvoiceInputId   FK?      Link tới Hóa đơn đầu vào (VAT chứng từ)
│   └── createdAt / updatedAt / canceledAt
│
├── Lines  (1..N)
│   ├── productId         FK
│   ├── variantId         FK?      Biến thể nếu có
│   ├── unitId            FK UoM   cái/hộp/thùng
│   ├── unitConversion    decimal  hệ số quy về đơn vị cơ bản
│   ├── quantity          decimal
│   ├── importPrice       decimal  Giá nhập (per unit)
│   ├── discount          decimal  Giảm trên dòng
│   ├── lineTotal         decimal  qty × price - discount
│   ├── allocatedCost     decimal  Chi phí nhập phân bổ vào dòng (xem 2.2)
│   ├── costAfterAlloc    decimal  Giá vốn ghi vào product cost
│   └── note              string
│
└── Costs (0..N) — Chi phí nhập hàng (xem 2.2)
    ├── costTypeId        FK ImportCostType
    ├── value             decimal
    ├── paidToSupplier    bool     true=trả NCC | false=trả bên thứ 3
    └── allocationMethod  enum     Theo giá trị | Theo SL | Theo trọng lượng
```

### 2.2 Entity hỗ trợ — `ImportCostType` (Chi phí nhập hàng)

Bật tại **Thiết lập → Hàng hóa → Quản lý chi phí nhập hàng**. Mỗi loại:

| Trường | Ghi chú |
|---|---|
| name | Vận chuyển, Bốc dỡ, Bảo hiểm, Phí kiểm định... |
| scope | Áp dụng cho 1 phiếu | tất cả | theo NCC |
| allocationMethod | Theo giá trị dòng / theo SL / theo trọng lượng |
| paidToSupplier | Có cộng vào số nợ NCC không |

**Tác động tới giá vốn:**
- Tổng chi phí được phân bổ vào từng line theo `allocationMethod`
- `costAfterAlloc = (lineTotal + allocatedCost) / quantity`
- Cập nhật `product.costPrice` theo **Bình quân gia quyền** (xem § 4)

### 2.3 Liên kết Phiếu Nhập ↔ Thẻ kho ↔ Sổ quỹ ↔ Công nợ

```
Phiếu Nhập (PN000199)
    │
    ├──► N dòng Thẻ kho (StockCard)        [tác động tồn]
    │     mỗi line × 1 dòng (qty +, branchId, costPrice)
    │
    ├──► 1 phiếu Thu/Chi (Sổ quỹ)         [nếu paidAmount > 0]
    │     loại: Chi cho NCC
    │
    ├──► 1 dòng Công nợ NCC                [nếu debtAmount > 0]
    │     Supplier.balance -= debtAmount
    │
    └──► Có thể link Hóa đơn đầu vào (HĐĐT) [optional, cho khấu trừ VAT]
```

---

## 3. STATE MACHINE — VÒNG ĐỜI PHIẾU NHẬP

```
                  [Tạo mới]
                       │
                       ▼
              ┌──── Phiếu tạm ────┐
              │   (Lưu tạm)        │
              │                    │
        [Sửa]│                    │[Hoàn thành]
              │                    │
              └───────►            ▼
                              Đã nhập hàng ◄──── (mặc định trạng thái cuối)
                                   │
                                   │ [Hủy phiếu]
                                   │ (popup hỏi: hủy phiếu thu/chi liên quan không?)
                                   ▼
                                Đã hủy
                                (immutable, vẫn lưu trong thẻ kho dưới dạng reverse entry)
```

### 3.1 Tác động theo trạng thái

| Trạng thái | Tồn kho | Công nợ NCC | Sổ quỹ | Thẻ kho |
|---|---|---|---|---|
| Phiếu tạm | Không | Không | Không | Không |
| Đã nhập hàng | ✅ Tăng | ✅ Tăng (nếu nợ) | ✅ Tạo phiếu chi (nếu trả) | ✅ Ghi dòng + |
| Đã hủy | ❌ Hoàn lại (giảm) | ❌ Trừ lại | ❌ Hủy phiếu (tùy chọn) | ✅ Ghi dòng đảo (-) |

### 3.2 Quy tắc cập nhật ở trạng thái "Đã nhập hàng"

Theo help KiotViet:
- Thông tin chung (Người nhập, Ngày nhập, Ghi chú) → sửa được trực tiếp
- **NCC** → chỉ sửa được khi phiếu **chưa có NCC** (đã có thì khóa)
- Sửa toàn bộ phiếu → nhấn **"Mở phiếu"** → quay về trạng thái phiếu tạm để chỉnh

→ Hệ thống **không cho sửa NCC khi đã ghi nợ** để tránh phá vỡ tham chiếu công nợ.

---

## 4. ẢNH HƯỞNG LÊN GIÁ VỐN (COST PRICE)

KiotViet dùng **Bình quân gia quyền (Weighted Average Cost — WAC)** ở cấp **toàn merchant** (không phải per branch).

### 4.1 Công thức

```
Khi nhập SL=q với giá nhập=p (đã cộng chi phí phân bổ):

cost_new = (cost_old × qty_on_hand_old + p × q) / (qty_on_hand_old + q)
qty_on_hand_new = qty_on_hand_old + q
```

### 4.2 Edge cases

| Tình huống | Behavior |
|---|---|
| `qty_on_hand_old ≤ 0` (đang âm tồn) | KiotViet vẫn cho nhập, `cost_new = p` (override) |
| Nhập backdate vào quá khứ | Cost không recalculate retroactively — đây là pain (§ 6.4) |
| Hủy phiếu nhập đã có bán hàng sau đó | Cost rollback bằng reverse entry, nhưng các HĐ bán đã issue không update |
| Nhập với giá 0 (hàng tặng/mẫu) | Cho phép, ảnh hưởng kéo giá vốn xuống — pain |

### 4.3 So với chuẩn kế toán

- **FIFO** (First-In-First-Out) thuần — KiotViet **không hỗ trợ** trừ trường hợp đặc biệt có Lô/Hạn sử dụng (xem `lo-han-su-dung-deep-dive.md`)
- **LIFO** — không hỗ trợ (luật Việt Nam cũng không cho LIFO)
- **WAC** — đây là phương pháp KiotViet đang dùng, hợp pháp ở VN (TT200/2014)

---

## 5. LUỒNG NGHIỆP VỤ CHI TIẾT (theo Help)

### 5.1 Tạo phiếu nhập (happy path)

```
1. Quản lý → Hàng hóa → Nhập hàng → [+ Nhập hàng]
2. Thêm hàng hóa vào lines bằng 1 trong 4 cách:
   ├─ Tìm theo tên/mã hoặc quét barcode (F6 toggle nhập_nhanh / nhập_thường)
   ├─ Click icon nhóm hàng → add tất cả SP trong nhóm
   ├─ Click + → tạo SP mới ngay trên màn nhập (cần quyền)
   └─ Import Excel (Mã hàng, SL, Giá nhập, ...)
3. Điều chỉnh dòng:
   ├─ Nhiều mức giá: + trên dòng → ghi nhận khuyến mãi "mua 10 tặng 1" (line giá 0)
   ├─ Sửa giá bán ngay trên phiếu (icon thiết lập giá)
   └─ Nhập VAT nhập (nếu bật)
4. Chọn NCC (F4 search hoặc + tạo mới)
5. Điền:
   ├─ Giảm giá tổng phiếu
   ├─ Tiền trả NCC lần này
   └─ Công nợ = totalCost - paidAmount (auto)
6. [Hoàn thành] hoặc [Lưu tạm]
```

### 5.2 Đối tượng "Tùy chọn hiển thị" trên màn nhập

UI có icon "con mắt" cho phép bật/tắt cột:
- Ảnh hàng hóa | Tồn kho | Chế độ lọc | Sắp xếp | Giá vốn | Giá bán
- Thêm dòng | Thiết lập giá | Chỉnh sửa thành tiền | Chọn nhiều hàng hóa
- Giá nhập là giá vốn | Xem hàng hóa cùng loại | Giảm giá VND/% | Mặc định trả NCC | Tự động điền SL theo phiếu đặt

→ Hệ thống ưu tiên **personalize cho từng user** thay vì 1 layout cứng. Đây là điểm mạnh UX.

### 5.3 Các thao tác phụ trên list

- **Sao chép phiếu** — copy header + lines, không copy thanh toán/NCC ghi nhận
- **Trả hàng nhập** từ chính phiếu nhập (xem `tra-hang-deep-dive.md` § 2)
- **In tem mã** ngay sau khi nhập — số tem = số SL trong phiếu
- **Gửi Email** thông tin phiếu cho NCC (đối soát)
- **In hàng loạt** / **Xuất file tổng quan** / **Xuất file chi tiết**

---

## 6. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN

### 6.1 Nhập liệu — tốc độ & sai sót

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Phải gõ giá nhập từng dòng — khi NCC đã có price list cố định, vẫn phải nhập tay | **Catalog giá NCC**: lưu giá nhập gần nhất theo (NCC × SP × UoM), auto-fill khi chọn NCC |
| 2 | Không phát hiện được "giá nhập tăng/giảm bất thường" — nhập 100k khi gần đây toàn 30k vẫn pass | **Cảnh báo deviation** ±20% so với giá trung vị 3 lần gần nhất |
| 3 | Không có **OCR hóa đơn giấy/PDF của NCC** để bắn vào phiếu | Upload ảnh phiếu NCC → AI parse → pre-fill lines (giải pain lớn của hộ kinh doanh) |
| 4 | Quét barcode chỉ thêm 1 SP/lần — không có "scan kit/pallet" nhanh | Hỗ trợ **GS1 pallet barcode** → 1 scan = N SP đã định nghĩa |
| 5 | Không có chế độ "blind receiving" (nhận hàng mà không nhìn số PO) — pain với kho lớn cần kiểm độc lập | Blind mode: cân/đếm trước, đối chiếu PO sau |

### 6.2 Giá vốn & chi phí

| # | Pain | Đề xuất |
|---|---|---|
| 6 | WAC ở merchant level, KHÔNG per branch — CN ở vùng giá cao bị "trung bình hóa" với CN giá thấp | Option: WAC per branch hoặc per location |
| 7 | Chi phí nhập hàng chỉ phân bổ tại thời điểm nhập — chi phí đến **sau** (vd hóa đơn ship về sau 1 tuần) không phân bổ lại | "Landed cost adjustment" — phiếu phụ phân bổ chi phí trễ |
| 8 | Backdate phiếu nhập không recompute cost cho các HĐ bán sau đó | Cost engine cần versioning + recompute optional với cảnh báo |
| 9 | Nhập giá 0 (hàng tặng) kéo cost xuống | Flag "hàng tặng" — không tham gia WAC, chỉ +SL |
| 10 | Không thấy **lịch sử giá nhập** của SP theo NCC để so sánh xu hướng | Mini-chart "giá nhập 12 tháng qua" trên product detail |

### 6.3 Liên thông NCC & PO

| # | Pain | Đề xuất |
|---|---|---|
| 11 | Phiếu Nhập có thể tạo mà không link DHN — không bắt buộc PO-first → khó audit | Setting "bắt buộc PO" cho merchant chuyên nghiệp |
| 12 | Không có **3-way match** (PO ↔ Goods Receipt ↔ Invoice) tự động | Auto-match 3-way + cảnh báo discrepancy — must-have cho chuỗi |
| 13 | Một phiếu DHN không thể nhập **partial multiple times** một cách rõ ràng — khó theo dõi PO còn lại | "Partial fulfillment tracker" trên DHN — show đã nhập, còn lại |
| 14 | Không có **NCC scorecard** (on-time %, fill-rate %, defect %, lead time) | Vendor performance dashboard (đã đề xuất ở tổng quan #20) |
| 15 | Liên kết HĐĐT đầu vào vẫn thủ công — không tự pull từ hệ thống thuế | Auto-pull HĐĐT đầu vào từ Tổng cục thuế → match phiếu nhập |

### 6.4 Mobile & POS

| # | Pain | Đề xuất |
|---|---|---|
| 16 | App di động hỗ trợ tạo nhập hàng nhưng không có **scan barcode liên tục** với hands-free | Bluetooth ring scanner mode + voice confirmation |
| 17 | Không có offline mode khi nhập hàng ở kho không có wifi | Offline draft + sync khi online |

### 6.5 Phân quyền & kiểm soát

| # | Pain | Đề xuất |
|---|---|---|
| 18 | Phân quyền nhập hàng chỉ ở mức module — không có **giới hạn giá trị** (vd nhân viên chỉ được nhập phiếu ≤ 10tr) | Approval workflow theo ngưỡng giá trị |
| 19 | Không có **trace ai đã sửa giá nhập** sau khi tạo phiếu — chỉ có createdBy, không có change log | Audit log per field (đặc biệt giá nhập, NCC) |

---

## 7. CƠ HỘI ĐỘT PHÁ — TOP 5 CHO MODULE NHẬP HÀNG

| # | Sản phẩm/Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **OCR Nhập hàng** — chụp ảnh hóa đơn NCC → auto tạo phiếu | Nhập tay chậm, sai sót | Trung | Rất cao |
| 2 | **3-way Match Auto** (PO ↔ GR ↔ Invoice) + Vendor scorecard | Chuỗi cần kiểm soát NCC, hiện làm Excel | Cao | Cao (enterprise) |
| 3 | **Smart Reorder** — AI gợi ý đặt hàng từ NCC nào, SL bao nhiêu, dựa trên velocity + lead time | Đặt hàng cảm tính, out-of-stock hoặc tồn dư | Cao | Rất cao |
| 4 | **Landed Cost Engine** — phân bổ chi phí trễ, multi-currency, customs duty | F&B nhập khẩu, mỹ phẩm xách tay | Trung | Trung |
| 5 | **NCC Self-service Portal** — NCC tự tạo phiếu nhập đề xuất (như EDI nhẹ), shop duyệt | Giảm 50% thời gian nhập của shop chuỗi | Cao | Cao |

---

## 8. ENTITY MODEL BỔ SUNG

Bổ sung 4 đối tượng gắn với nghiệp vụ Nhập hàng vào danh sách entities:

| # | Đối tượng | Vai trò |
|---|---|---|
| 37 | **Phiếu Nhập hàng (PurchaseImport)** | Header + Lines, prefix `PN` |
| 38 | **Đặt hàng nhập (PurchaseOrder)** | PO prefix `DHN`, không tác động tồn |
| 39 | **Chi phí nhập hàng (ImportCost / ImportCostType)** | Phân bổ vào giá vốn |
| 40 | **Hóa đơn đầu vào (Input eInvoice)** | Chứng từ VAT, link tới PN cho khấu trừ thuế |

---

## 9. TÓM LƯỢC

**Nhập hàng KiotViet:**
- Là **mắt xích đầu** của data tồn kho — quyết định giá vốn và công nợ NCC
- Schema khá đầy đủ: hỗ trợ multi-line, multi-UoM, chi phí phân bổ, link DHN/HĐĐT
- State machine 3 trạng thái: Phiếu tạm → Đã nhập → Đã hủy. Tác động lan ra 4 đối tượng: Tồn kho, Công nợ NCC, Sổ quỹ, Thẻ kho
- Giá vốn dùng **WAC merchant-level** — đơn giản nhưng không tối ưu cho chuỗi đa vùng
- **Mạnh:** Tùy chọn hiển thị linh hoạt, import Excel, in tem mã ngay sau nhập, link PO/HĐĐT, app mobile có
- **Yếu:** Không OCR, không 3-way match, không vendor scorecard, không cảnh báo giá bất thường, backdate không recompute, không landed cost trễ
- **Cơ hội đột phá:** OCR phiếu NCC, Smart Reorder AI, NCC self-service portal — cả 3 đều match đúng định hướng AI-Operations + Enterprise của KiotViet

Module này **đã hoàn thiện cho SMB nhỏ nhưng còn rất nhiều dư địa cho chuỗi và seller có volume lớn**.
