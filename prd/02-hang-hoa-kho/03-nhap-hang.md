# Nghiệp vụ Nhập hàng — Deep Dive

**Bối cảnh:** Nhập hàng (Goods Receipt / Purchase) là nghiệp vụ **đưa hàng từ NCC vào kho** — đầu nguồn của mọi tồn kho và cơ sở để tính giá vốn. Cùng với Đặt hàng nhập (DHN) và Trả hàng nhập (TPN), 3 phiếu này tạo thành cụm **Mua hàng** trong KiotViet.

**Entities:** E15 (PurchaseImport), E16 (PurchaseOrder), E17 (ImportCostType), E18 (InputEInvoice)
**Liên quan:** [README.md](./README.md) | [02-the-kho.md](./02-the-kho.md) | [05-tra-hang.md](./05-tra-hang.md)

---

## 1. Vị trí trong hệ sinh thái

```
[Đặt hàng nhập (DHN — E16)] ──tham chiếu──► [Nhập hàng (PN — E15)] ──tham chiếu──► [Trả hàng nhập (TPN — E24)]
       optional                                      core                                      optional
          │                                           │
          │                                           ├──► Tăng tồn (E09 — Stock per Branch)
          │                                           ├──► Tăng công nợ phải trả NCC
          │                                           ├──► Cập nhật giá vốn (WAC)
          │                                           ├──► Ghi dòng Thẻ kho (E10 — PN######)
          │                                           └──► (nếu bật) Phân bổ Chi phí nhập vào giá vốn
          │
          └──► Không tác động tồn — chỉ ghi nhận "Đặt NCC" trên product detail
```

| Phiếu | Prefix | Tác động tồn | Vai trò |
|---|---|---|---|
| Đặt hàng nhập | DHN | Không (chỉ "Đặt NCC") | PO — cam kết với NCC, deadline giao hàng |
| Nhập hàng | PN | Tăng tồn | Goods receipt — hàng đã về kho thực tế |
| Trả hàng nhập | TPN | Giảm tồn | Trả NCC khi lỗi/sai quy cách |

---

## 2. Cấu trúc dữ liệu

### 2.1 Entity chính — PurchaseImport (E15)

```
PurchaseImport  (PN######)
├── Header
│   ├── code              string   PN######, sequential per merchant
│   ├── branchId          FK       CN nhập hàng
│   ├── supplierId        FK?      NCC (có thể null khi mới tạo)
│   ├── importedAt        datetime Ngày nhập (cho phép backdate)
│   ├── importedBy        FK User  Người nhập
│   ├── status            enum     Phiếu_tạm | Đã_nhập_hàng | Đã_hủy
│   ├── discountAmount    decimal  Giảm trên tổng phiếu
│   ├── totalCost         decimal  ∑ lines (sau giảm)
│   ├── paidAmount        decimal  Tiền đã trả NCC lần này
│   ├── debtAmount        decimal  totalCost − paidAmount
│   ├── note              string
│   ├── poId              FK?      Link tới DHN (E16) nếu có
│   ├── eInvoiceInputId   FK?      Link tới Hóa đơn đầu vào (E18)
│   └── createdAt / updatedAt / canceledAt
│
├── Lines  (1..N)
│   ├── productId         FK → E01
│   ├── variantId         FK? → E03
│   ├── unitId            FK → E02
│   ├── unitConversion    decimal  hệ số quy về đơn vị cơ bản
│   ├── quantity          decimal
│   ├── importPrice       decimal  Giá nhập per unit
│   ├── discount          decimal  Giảm trên dòng
│   ├── lineTotal         decimal  qty × price − discount
│   ├── allocatedCost     decimal  Chi phí nhập phân bổ vào dòng
│   ├── costAfterAlloc    decimal  Giá vốn ghi vào product cost
│   └── note              string
│
└── Costs  (0..N) — Chi phí nhập hàng
    ├── costTypeId        FK → E17
    ├── value             decimal
    ├── paidToSupplier    bool     true = trả NCC | false = trả bên thứ 3
    └── allocationMethod  enum     Theo_giá_trị | Theo_SL | Theo_trọng_lượng
```

### 2.2 Chi phí nhập hàng (ImportCostType — E17)

Bật tại **Thiết lập → Hàng hóa → Quản lý chi phí nhập hàng**. Mỗi loại:

| Trường | Ghi chú |
|---|---|
| name | Vận chuyển, Bốc dỡ, Bảo hiểm, Phí kiểm định... |
| scope | Áp dụng cho 1 phiếu / tất cả / theo NCC |
| allocationMethod | Theo giá trị dòng / theo SL / theo trọng lượng |
| paidToSupplier | Có cộng vào số nợ NCC không |

**Tác động giá vốn:**
```
costAfterAlloc = (lineTotal + allocatedCost) / quantity
```
→ Cập nhật `product.costPrice` theo **Bình quân gia quyền (WAC)**

### 2.3 Liên kết hệ thống sau khi nhập hàng

```
Phiếu Nhập (PN######)
    │
    ├──► N dòng Thẻ kho (E10)          — tác động tồn (+qty, branchId, costPrice)
    │
    ├──► 1 phiếu Chi sổ quỹ            — nếu paidAmount > 0
    │
    ├──► 1 dòng Công nợ NCC            — nếu debtAmount > 0
    │
    └──► Hóa đơn đầu vào (E18)         — optional, cho khấu trừ VAT
```

---

## 3. State machine — Vòng đời phiếu nhập

```
            [Tạo mới]
                 │
                 ▼
        ┌── Phiếu tạm ──┐
        │   (Lưu tạm)    │
        │                │
  [Sửa]│                │[Hoàn thành]
        │                │
        └───────►         ▼
                    Đã nhập hàng
                          │
                          │ [Hủy phiếu]
                          ▼
                       Đã hủy
                    (immutable — vẫn lưu trong thẻ kho dưới dạng reverse entry)
```

### 3.1 Tác động theo trạng thái

| Trạng thái | Tồn kho | Công nợ NCC | Sổ quỹ | Thẻ kho |
|---|---|---|---|---|
| Phiếu tạm | Không | Không | Không | Không |
| Đã nhập hàng | Tăng | Tăng (nếu nợ) | Tạo phiếu chi (nếu trả) | Ghi dòng + |
| Đã hủy | Hoàn lại (giảm) | Trừ lại | Hủy phiếu (tùy chọn) | Ghi dòng đảo − |

### 3.2 Quy tắc sửa phiếu ở trạng thái "Đã nhập hàng"

- Thông tin chung (Người nhập, Ngày nhập, Ghi chú) → sửa trực tiếp
- **NCC** → chỉ sửa được khi phiếu **chưa có NCC** (đã có NCC → khóa, vì không được phá vỡ tham chiếu công nợ)
- Sửa toàn bộ phiếu → nhấn **"Mở phiếu"** → quay về Phiếu tạm để chỉnh

---

## 4. Giá vốn (Cost Price) — Bình quân gia quyền (WAC)

KiotViet dùng **Weighted Average Cost (WAC)** ở cấp **toàn merchant** (không per branch).

### 4.1 Công thức

```
Khi nhập quantity = q với giá = p (đã cộng chi phí phân bổ):

cost_new = (cost_old × qty_old + p × q) / (qty_old + q)
qty_new  = qty_old + q
```

### 4.2 Edge cases

| Tình huống | Behavior |
|---|---|
| `qty_old ≤ 0` (đang âm tồn) | Vẫn cho nhập — `cost_new = p` (override) |
| Nhập backdate vào quá khứ | Cost **không recompute** retroactively — pain (xem §6.2 #8) |
| Hủy phiếu nhập đã có bán sau đó | Cost rollback bằng reverse entry — nhưng HĐ bán đã issue không update |
| Nhập giá 0 (hàng tặng/mẫu) | Kéo giá vốn xuống — nên dùng flag "hàng tặng" |

### 4.3 So với chuẩn kế toán

- **FIFO** — KiotViet không hỗ trợ thuần (ngoại trừ khi kết hợp Lô/HSD)
- **LIFO** — không hỗ trợ (luật VN không cho)
- **WAC** — phương pháp đang dùng, hợp pháp theo TT200/2014

---

## 5. Luồng nghiệp vụ chi tiết

### 5.1 Tạo phiếu nhập (happy path)

```
1. Quản lý → Hàng hóa → Nhập hàng → [+ Nhập hàng]
2. Thêm hàng hóa vào lines (4 cách):
   ├─ Tìm theo tên/mã hoặc quét barcode (F6 toggle chế độ nhập)
   ├─ Click nhóm hàng → add tất cả SP trong nhóm
   ├─ Click + → tạo SP mới ngay trên màn nhập
   └─ Import Excel (Mã hàng, SL, Giá nhập...)
3. Điều chỉnh dòng:
   ├─ Nhiều mức giá: ghi nhận khuyến mãi "mua 10 tặng 1" (line giá 0)
   ├─ Sửa giá bán ngay trên phiếu (icon thiết lập giá)
   └─ Nhập VAT nhập (nếu bật)
4. Chọn NCC (F4 search hoặc + tạo mới)
5. Điền: Giảm giá tổng, Tiền trả NCC, Công nợ (auto)
6. [Hoàn thành] hoặc [Lưu tạm]
```

### 5.2 Tùy chỉnh hiển thị

UI có icon "con mắt" cho phép bật/tắt cột và chế độ:
- Ảnh hàng hóa | Tồn kho | Giá vốn | Giá bán
- Giá nhập là giá vốn | Xem hàng hóa cùng loại | Giảm giá VND/%
- Mặc định trả NCC | Tự động điền SL theo phiếu đặt

### 5.3 Thao tác phụ trên list

- **Sao chép phiếu** — copy header + lines, không copy thanh toán
- **Trả hàng nhập** từ chính phiếu nhập (xem [05-tra-hang.md](./05-tra-hang.md) §3)
- **In tem mã** ngay sau nhập — số tem = SL trong phiếu
- **Gửi Email** thông tin phiếu cho NCC đối soát
- **Xuất file** tổng quan hoặc chi tiết

---

## 6. Điểm đau & cơ hội cải tiến

### 6.1 Nhập liệu — tốc độ & sai sót

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Phải gõ giá nhập từng dòng — NCC có price list cố định vẫn phải nhập tay | **Catalog giá NCC**: lưu giá nhập gần nhất per (NCC × SP × UoM), auto-fill |
| 2 | Không phát hiện "giá nhập tăng bất thường" — nhập 100k khi gần đây toàn 30k vẫn pass | **Cảnh báo deviation** ±20% so với giá trung vị 3 lần gần nhất |
| 3 | Không có OCR hóa đơn giấy/PDF của NCC để bắn vào phiếu | Upload ảnh phiếu NCC → AI parse → pre-fill lines |
| 4 | Quét barcode chỉ thêm 1 SP/lần — không có "scan pallet" nhanh | Hỗ trợ GS1 pallet barcode → 1 scan = N SP |
| 5 | Không có chế độ "blind receiving" (nhận hàng không nhìn số PO) | Blind mode: đếm/cân trước, đối chiếu PO sau |

### 6.2 Giá vốn & chi phí

| # | Pain | Đề xuất |
|---|---|---|
| 6 | WAC ở merchant level — CN vùng giá cao bị "trung bình hóa" với CN giá thấp | Option: WAC per branch hoặc per location |
| 7 | Chi phí nhập hàng chỉ phân bổ lúc nhập — chi phí đến sau (VD: hóa đơn ship về sau 1 tuần) không phân bổ lại | "Landed cost adjustment" — phiếu phụ phân bổ chi phí trễ |
| 8 | Backdate phiếu nhập không recompute cost cho HĐ bán sau đó | Cost engine cần versioning + recompute optional với cảnh báo |
| 9 | Nhập giá 0 (hàng tặng) kéo cost xuống | Flag "hàng tặng" — không tham gia WAC, chỉ +SL |
| 10 | Không có lịch sử giá nhập của SP theo NCC để so sánh xu hướng | Mini-chart "giá nhập 12 tháng qua" trên product detail |

### 6.3 Liên thông NCC & PO

| # | Pain | Đề xuất |
|---|---|---|
| 11 | Phiếu Nhập có thể tạo không link DHN — không bắt buộc PO-first → khó audit | Setting "bắt buộc PO" cho merchant chuyên nghiệp |
| 12 | Không có 3-way match (PO ↔ Goods Receipt ↔ Invoice) tự động | Auto-match 3-way + cảnh báo discrepancy |
| 13 | 1 DHN không thể nhập partial rõ ràng — khó theo dõi PO còn lại | "Partial fulfillment tracker" trên DHN |
| 14 | Không có NCC scorecard (on-time %, fill-rate %, defect %, lead time) | Vendor performance dashboard |
| 15 | Liên kết HĐĐT đầu vào vẫn thủ công | Auto-pull HĐĐT đầu vào từ Tổng cục thuế → match phiếu nhập |

### 6.4 Mobile & phân quyền

| # | Pain | Đề xuất |
|---|---|---|
| 16 | App mobile không có scan barcode liên tục hands-free | Bluetooth ring scanner mode + voice confirmation |
| 17 | Không có offline mode khi nhập hàng tại kho không có wifi | Offline draft + sync khi online |
| 18 | Phân quyền nhập hàng chỉ ở mức module — không có giới hạn giá trị | Approval workflow theo ngưỡng giá trị |
| 19 | Không có trace ai đã sửa giá nhập — chỉ có createdBy, không có change log | Audit log per field (đặc biệt giá nhập, NCC) |

---

## 7. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **OCR Nhập hàng** — chụp ảnh hóa đơn NCC → auto tạo phiếu | Nhập tay chậm, sai sót (pain #3) | Trung | Rất cao |
| 2 | **3-way Match Auto** (PO ↔ GR ↔ Invoice) + Vendor scorecard | Chuỗi cần kiểm soát NCC (pain #12, #14) | Cao | Cao (enterprise) |
| 3 | **Smart Reorder** — AI gợi ý đặt hàng từ NCC nào, SL bao nhiêu | Đặt hàng cảm tính → out-of-stock hoặc tồn dư | Cao | Rất cao |
| 4 | **Landed Cost Engine** — phân bổ chi phí trễ, multi-currency, customs duty | F&B nhập khẩu, mỹ phẩm xách tay (pain #7) | Trung | Trung |
| 5 | **NCC Self-service Portal** — NCC tự tạo phiếu đề xuất, shop duyệt | Giảm 50% thời gian nhập liệu cho chuỗi | Cao | Cao |

---

## 8. Tóm lược

**Nhập hàng KiotViet:**
- **Mắt xích đầu** của data tồn kho — quyết định giá vốn và công nợ NCC
- Schema đầy đủ: multi-line, multi-UoM, chi phí phân bổ, link DHN/HĐĐT
- State machine: Phiếu tạm → Đã nhập → Đã hủy — tác động lan ra 4 đối tượng
- Giá vốn dùng **WAC merchant-level** — đơn giản nhưng không tối ưu cho chuỗi đa vùng
- **Mạnh:** Tùy chỉnh hiển thị linh hoạt, import Excel, in tem mã ngay sau nhập, app mobile
- **Yếu:** Không OCR, không 3-way match, không vendor scorecard, không cảnh báo giá bất thường, backdate không recompute
- **Cơ hội đột phá:** OCR phiếu NCC, Smart Reorder AI, NCC self-service portal

Đã hoàn thiện cho SMB nhỏ — còn nhiều dư địa cho chuỗi và seller volume lớn.
