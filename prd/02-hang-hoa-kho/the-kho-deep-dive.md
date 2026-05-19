# ĐÀO SÂU: MODULE THẺ KHO & MÃ THẺ KHO — KIOTVIET

**Bối cảnh:** Thẻ kho (Stock Card / Inventory Ledger) là **sổ cái duy nhất ghi nhận mọi biến động tồn của từng hàng hóa** — tương đương "general ledger" trong kế toán. Đây là backbone của hệ thống inventory và là single source of truth cho audit, báo cáo, đối soát.

---

## 1. THẺ KHO LÀ GÌ TRONG KIOTVIET

### 1.1 Vị trí truy cập
- Mở 1 sản phẩm → **Tab "Thẻ kho"** trong product detail panel
- Không có view "thẻ kho cross-product" độc lập — phải vào từng SP để xem
- (Có module riêng "Báo cáo Hàng hóa" nhưng dạng aggregate, không phải ledger chi tiết)

### 1.2 Cấu trúc dữ liệu (Schema)

```
Thẻ kho (StockCard / InventoryLedger)
├── Một dòng = 1 movement của 1 SP tại 1 thời điểm
├── Cấp độ chi tiết: 1 SP × 1 phiếu × 1 line = 1 dòng thẻ kho
└── Trường:
    ├── Chứng từ (Document code) — mã phiếu nguồn (HD, PN, TRF...)
    ├── Thời gian (Timestamp) — chính xác đến giây
    ├── Loại giao dịch (Transaction type)
    ├── Đối tác (Partner) — Khách hàng / NCC / Nhân viên / CN khác
    ├── Giá GD (Transaction price) — giá trên phiếu nguồn
    ├── Giá vốn (Cost price tại thời điểm GD)
    ├── Số lượng (Quantity) — **CÓ DẤU**: âm = xuất, dương = nhập
    └── Tồn cuối (Running balance) — số tồn ngay sau GD này
```

**Sắp xếp mặc định:** DESC theo Thời gian (mới nhất trên cùng).
**Tồn cuối:** Là **running balance** — tích lũy từ đầu kỳ + tất cả movement tới thời điểm dòng đó.

### 1.3 Sample data quan sát được

Sản phẩm `AUTOinventoryCase1`, Tồn hiện tại = **-6**:

| Chứng từ | Thời gian | Loại GD | Đối tác | Giá GD | Giá vốn | SL | Tồn cuối |
|---|---|---|---|---|---|---|---|
| HD016341 | 09/12/2025 15:10 | Bán hàng | Khách lẻ | 994,856 | 0 | -1 | -6 |
| HD016193 | 07/11/2025 18:02 | Bán hàng | Phan ngân | 994,856 | 0 | -1 | -5 |
| HD016152 | 01/11/2025 14:19 | Bán hàng | Phan ngân | 994,856 | 0 | -1 | -4 |
| HD015906 | 10/09/2025 16:29 | Bán hàng | Phan ngân | 994,856 | 0 | -1 | -3 |
| HD015742 | 01/08/2025 12:08 | Bán hàng | Phan ngân | 400,000 | 0 | -1 | -2 |
| HD015266 | 23/05/2025 16:16 | Bán hàng | Phan ngân | 994,856 | 0 | -1 | -1 |

Quan sát:
- 6 lần bán liên tiếp → tồn -6 (chưa nhập)
- Giá vốn = 0 cho thấy SP chưa từng nhập kho → KiotViet vẫn cho bán (cho phép âm tồn)
- 1 lần bán giá 400k khác mặc định 994k (sale một lần)

---

## 2. MÃ CHỨNG TỪ (DOCUMENT CODE / MÃ THẺ KHO)

### 2.1 Quy tắc đặt mã (đã xác nhận từ test data)

Format: **[PREFIX] + [SEQUENTIAL 6 DIGITS]** — vd `HD016341`, `PN000199`, `TRF000015`

| Prefix | Tên đầy đủ | Nghiệp vụ | Dấu SL | Xác nhận |
|---|---|---|---|---|
| **HD** | Hóa Đơn | Bán hàng (Sales) | (-) Xuất | ✅ Confirmed |
| **PN** | Phiếu Nhập | Nhập hàng từ NCC (Goods Receipt) | (+) Nhập | ✅ Confirmed |
| **TRF** | Transfer | Chuyển hàng giữa CN | (-) ở CN gửi, (+) ở CN nhận | ✅ Confirmed |
| **TH** | Trả Hàng | Khách trả hàng | (+) Nhập | Suy luận |
| **TN** hoặc **TPN** | Trả hàng Nhập | Trả NCC | (-) Xuất | Suy luận |
| **DH** | Đặt Hàng | Order chưa xuất kho | Không tác động tồn | Suy luận |
| **DHN** | Đặt Hàng Nhập | PO chưa về | Không tác động tồn (Đặt NCC) | Suy luận |
| **KK** | Kiểm Kho | Stock take adjustment | (±) Tùy lệch | Suy luận |
| **SX** | Sản Xuất | Manufacturing/Assembly | NVL (-), Thành phẩm (+) | Suy luận |
| **XN** hoặc **XDN** | Xuất dùng Nội bộ | Internal use | (-) Xuất | Suy luận |
| **XH** | Xuất Hủy | Write-off | (-) Xuất | Suy luận |

### 2.2 Tính chất của mã

1. **Sequential per type** — mỗi loại có dải số riêng (HD016341 ≠ PN016341 — đếm độc lập)
2. **6 chữ số padded** — tới 999,999 phiếu/loại trước khi cần expand
3. **Per merchant scope** — KHÔNG global qua nhiều shop, chỉ unique trong 1 tài khoản KiotViet
4. **Per branch?** — Cần kiểm tra: thường shared toàn merchant (cross-branch)
5. **Immutable** — sau khi tạo, mã không đổi (kể cả khi hủy phiếu)
6. **Clickable link** trên Thẻ kho → mở popup chi tiết phiếu nguồn
7. **Custom prefix** — chưa thấy option cho phép merchant đổi prefix

### 2.3 Quan hệ Mã thẻ kho ↔ Phiếu nguồn

```
Thẻ kho (StockCard.id) ─tham chiếu─► Chứng từ (SourceDocument.code)
                                        │
                            ┌───────────┼─────────────┬──────────────┐
                            ▼           ▼             ▼              ▼
                         Hóa đơn    Phiếu Nhập     Chuyển hàng    Kiểm kho ...
                         (HD)       (PN)           (TRF)          (KK)
```

**Tính chất:** 1 chứng từ có thể sinh ra **N dòng thẻ kho** (mỗi line item của phiếu = 1 dòng thẻ kho cho SP tương ứng × CN tương ứng).

Vd: Phiếu chuyển hàng `TRF000015` (2 items) sinh ra:
- 2 dòng thẻ kho ở **CN gửi** (-5 bò húc, -2 bánh ngọt)
- 2 dòng thẻ kho ở **CN nhận** (+5 bò húc, +2 bánh ngọt)
→ Tổng 4 dòng thẻ kho cho 1 mã TRF000015.

---

## 3. LOẠI GIAO DỊCH (TRANSACTION TYPE)

Đây là **enum** mô tả nghiệp vụ tạo ra dòng thẻ kho. Quan sát + suy luận:

| Loại | Dấu | Tác động | Mã prefix |
|---|---|---|---|
| Bán hàng | − | Giảm tồn CN bán | HD |
| Trả hàng (KH trả) | + | Tăng tồn CN tiếp nhận | TH |
| Nhập hàng (từ NCC) | + | Tăng tồn CN nhập + cập nhật giá vốn | PN |
| Trả hàng nhập (trả NCC) | − | Giảm tồn | TN/TPN |
| Chuyển hàng đi | − | Giảm tồn CN gửi | TRF |
| Chuyển hàng đến | + | Tăng tồn CN nhận | TRF (cùng mã) |
| Kiểm kho — lệch tăng | + | Tăng tồn theo SL thực tế | KK |
| Kiểm kho — lệch giảm | − | Giảm tồn theo SL thực tế | KK |
| Sản xuất — xuất NVL | − | Giảm NVL | SX |
| Sản xuất — nhập thành phẩm | + | Tăng thành phẩm | SX (cùng mã) |
| Xuất dùng nội bộ | − | Giảm tồn, không xuất HĐ | XN |
| Xuất hủy | − | Giảm tồn, ghi chi phí | XH |
| Điều chỉnh tay (manual) | ± | Hiếm, chỉ admin có quyền | (chưa biết) |

---

## 4. CÁC ĐỐI TƯỢNG (ENTITIES) LIÊN QUAN

### 4.1 Entity chính: **StockCard / StockCardLine**

| Trường | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint, PK | Internal |
| `productId` | FK → Product | SP nào |
| `branchId` | FK → Branch | CN nào — vì Tồn = SP × CN |
| `documentCode` | string | Mã chứng từ display (HD016341) |
| `documentType` | enum | HD/PN/TRF/KK/SX/XN/XH/TH/TN |
| `documentId` | FK → SourceDocument | Link tới phiếu nguồn |
| `transactionType` | enum | Bán hàng / Nhập / Chuyển đi /... |
| `partnerType` | enum | Customer/Supplier/Branch/Employee |
| `partnerId` | FK | Đối tác |
| `partnerName` | string | Cache name (vì đối tác có thể đổi tên sau) |
| `transactionPrice` | decimal | Giá GD |
| `costPrice` | decimal | Giá vốn tại thời điểm |
| `quantity` | decimal | **CÓ DẤU** (- xuất, + nhập) |
| `runningBalance` | decimal | Tồn cuối — **denormalized** |
| `transactionAt` | datetime | Thời gian |
| `createdAt` | datetime | Audit |
| `createdBy` | FK → User | Người tạo |
| `unitId` | FK → UoM | Đơn vị tính (cái/hộp/thùng) |
| `unitConversion` | decimal | Hệ số quy đổi về đơn vị cơ bản |

### 4.2 Vấn đề về `runningBalance`

`Tồn cuối` được lưu trực tiếp vào DB (denormalized) để query nhanh, nhưng:
- **Recalculation cost cao** nếu chèn 1 dòng thẻ kho có thời gian backdate — phải recompute tất cả dòng sau đó
- **Concurrency risk** nếu 2 phiếu cùng tác động lên 1 SP × CN cùng thời điểm — cần lock
- Đây là một trong những điểm phức tạp nhất của inventory engine

### 4.3 Entity phụ: **StockCardSummary** (giả định)

Để render fast "Tồn kho hiện tại" trên UI list, hệ thống chắc chắn có 1 bảng aggregate `Stock(productId, branchId, qty, qty_ordered_from_supplier, qty_reserved_by_customer)` được update khi mỗi phiếu hoàn tất — KHÔNG phải SUM từ thẻ kho mỗi lần.

---

## 5. UI/UX TRÊN MÀN HÌNH THẺ KHO

### 5.1 Cấu trúc hiển thị

```
┌─────────────────────────────────────────────────────────────────┐
│  TAB: Thông tin | Mô tả ghi chú | Thẻ kho ✓ | Tồn kho | LK kênh │
├─────────────────────────────────────────────────────────────────┤
│  Chứng từ │ Thời gian │ Loại GD │ Đối tác │ Giá GD ⓘ │ Giá vốn │ SL │ Tồn cuối │
│  ─────────────────────────────────────────────────────────────  │
│  HD016341 │ 09/12/25  │ Bán hàng│ Khách lẻ│ 994,856  │ 0       │-1 │  -6     │
│  HD016193 │ 07/11/25  │ Bán hàng│ Phan ngân│ 994,856 │ 0       │-1 │  -5     │
│  ...                                                            │
├─────────────────────────────────────────────────────────────────┤
│  [📥 Xuất file]                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Hiện có

- ✅ Sort DESC theo thời gian
- ✅ Click mã chứng từ → mở popup chi tiết phiếu nguồn
- ✅ Xuất file (CSV/Excel)
- ✅ Tooltip "ⓘ" cạnh Giá GD (chưa hover được hiệu quả — UX cần fix)

### 5.3 Còn THIẾU

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có **filter** trên Thẻ kho (theo loại GD, theo thời gian, theo CN, theo người tạo) | Toolbar filter — quan trọng cho SP có >100 dòng thẻ kho |
| 2 | Không thấy **chi nhánh** của mỗi dòng — phải đoán | Thêm cột Chi nhánh hoặc filter Chi nhánh |
| 3 | Không có **đơn vị tính** — nếu SP có nhiều UoM, SL hiển thị theo đơn vị nào? | Thêm cột UoM, hoặc luôn quy về đơn vị cơ bản |
| 4 | Không phân biệt visual giữa các loại GD | Color-code/icon theo Loại GD |
| 5 | Không có **summary footer** (Tổng nhập, Tổng xuất, Tồn đầu, Tồn cuối kỳ) | Footer aggregate row |
| 6 | Không có **bulk action** | Đa chọn → export, hoặc copy |
| 7 | Mã chứng từ trên popup chi tiết không link tới các phiếu liên quan (HĐ ↔ Phiếu trả nếu có) | Bidirectional link |
| 8 | Không hiển thị **người thực hiện** trên dòng | Cột "Người tạo" hoặc tooltip |
| 9 | Không có **search trong thẻ kho** (theo mã chứng từ, đối tác) | Search bar trong tab |
| 10 | Giá vốn = 0 khi chưa từng nhập — không rõ là "không có dữ liệu" hay "giá 0 thật" | UI hint: "—" thay vì 0 khi missing |
| 11 | Không có view "Thẻ kho gộp cả SP cha + biến thể con" | Toggle gộp variants |
| 12 | Phân trang không thấy — list dài sẽ load chậm | Pagination hoặc virtual scroll |

---

## 6. ĐỐI CHIẾU VỚI CHUẨN KẾ TOÁN

Thẻ kho KiotViet phản ánh **mô hình kho thực tế** (physical inventory), tương đương:
- **Stock Card** trong PMS/ERP truyền thống (SAP Material Document)
- **Inventory Ledger** trong QuickBooks / Xero
- **Mẫu S12-DN** Sổ chi tiết vật tư hàng hóa theo Thông tư 200/2014/TT-BTC của Bộ Tài chính

**Khác biệt với kế toán:**
- Thẻ kho không phải sổ kế toán tổng hợp — không có cột Nợ/Có
- KiotViet có module "Thuế & Kế toán" riêng để xử lý kế toán
- Thẻ kho có thể inconsistent với sổ kế toán nếu chu kỳ chốt sổ khác nhau

---

## 7. CƠ HỘI ĐỘT PHÁ

### 7.1 Quick wins

1. **Bổ sung filter + search + summary footer** trên tab Thẻ kho (đã list ở § 5.3)
2. **Color code** theo loại GD — visual recognition nhanh
3. **Cột Chi nhánh** — hiện thiếu

### 7.2 Cải tiến lớn

4. **Cross-product Stock Ledger view** — màn hình standalone xem thẻ kho qua nhiều SP với filter, để kế toán đối soát
5. **Inventory Audit Trail Hub** — search thẻ kho qua nhiều chiều (theo người làm, theo CN, theo loại GD trong khoảng thời gian)
6. **AI Anomaly Detection** trên thẻ kho:
   - Phát hiện chuỗi xuất bất thường (nhiều lần hủy phiếu, void liên tục)
   - Phát hiện nhân viên có pattern khả nghi (xuất hủy nhiều SP đắt tiền)
   - Cảnh báo gap kiểm kho lặp lại trên cùng SKU
   → Đây là feature đã propose trong brainstorm (C3 — Audit & Anomaly Detection)
7. **Time-travel "Tồn tại thời điểm X"** — query rollback từ thẻ kho để xem tồn quá khứ, phục vụ báo cáo định kỳ
8. **Reverse stock card** — từ 1 dòng thẻ kho → trace ngược chuỗi nguyên nhân (vd: chai bia này đã đi qua những phiếu nào?)

### 7.3 Đột phá

9. **Blockchain-anchored stock card** — hash mỗi block thẻ kho lên chain, đối tác/NCC verify được tính bất biến — selling point lớn cho enterprise/franchise.
10. **Real-time stock card stream API** — cho phép tích hợp ERP/BI realtime, không phải poll.

---

## 8. ENTITY MODEL BỔ SUNG (cho danh sách đối tượng)

Bổ sung 3 đối tượng mới gắn với module Thẻ kho vào danh sách 33 đối tượng trước đó:

| # | Đối tượng | Mô tả |
|---|---|---|
| 34 | **Mã chứng từ (Document Code)** | Định danh phiếu nguồn, format `[PREFIX][6 digits]`, prefix theo loại nghiệp vụ |
| 35 | **Loại giao dịch (Transaction Type)** | Enum 13 giá trị mô tả nghiệp vụ kho |
| 36 | **Tồn cuối / Running balance** | Số tồn tích lũy sau mỗi GD — denormalized field |

(Đối tượng 32 "Thẻ kho" đã liệt kê — file này là deep dive bổ sung cho nó)

---

## 9. TÓM LƯỢC

**Thẻ kho KiotViet:**
- Là **single source of truth** cho mọi biến động tồn — đủ chi tiết để audit
- Mã chứng từ tuân quy tắc **PREFIX + 6 digits**, prefix theo loại nghiệp vụ (HD/PN/TRF/...)
- Hỗ trợ **âm tồn** (cho phép bán khi chưa nhập) — tốt cho seller online preorder
- Click mã → mở phiếu nguồn — UX khá tốt
- **THIẾU:** filter trong tab, cột chi nhánh, summary footer, cross-product view, anomaly alerts
- **CƠ HỘI:** AI audit, real-time API, time-travel query, blockchain anchoring cho enterprise

Đây là module **đã hoàn thiện về core nhưng còn nhiều room cho UX và intelligence layer** — match đúng định hướng AI-Operations đã đề xuất ở phase brainstorm trước.
