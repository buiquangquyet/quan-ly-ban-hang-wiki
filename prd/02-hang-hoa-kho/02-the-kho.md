# Thẻ kho (Stock Card) — Deep Dive

**Bối cảnh:** Thẻ kho là **sổ cái duy nhất ghi nhận mọi biến động tồn của từng hàng hóa** — tương đương "general ledger" trong kế toán. Đây là backbone của inventory engine và single source of truth cho audit, báo cáo, đối soát.

**Entity:** E10 — StockCard / StockCardLine
**Liên quan:** [README.md](./README.md) | [01-tong-quan.md](./01-tong-quan.md)

---

## 1. Thẻ kho là gì trong KiotViet

### 1.1 Vị trí truy cập

- Mở 1 sản phẩm → **Tab "Thẻ kho"** trong product detail panel
- Không có view cross-product độc lập — phải vào từng SP để xem
- Module "Báo cáo Hàng hóa" có dạng aggregate, không phải ledger chi tiết

### 1.2 Cấu trúc dữ liệu (Schema)

```
StockCard / StockCardLine
├── Một dòng = 1 movement của 1 SP tại 1 CN, 1 thời điểm
├── Cấp độ chi tiết: 1 SP × 1 phiếu × 1 line = 1 dòng thẻ kho
└── Trường:
    ├── documentCode       Mã phiếu nguồn (HD, PN, TRF...) — E13
    ├── transactionAt      Timestamp chính xác đến giây
    ├── transactionType    Loại giao dịch — E14
    ├── partnerType/Id     Khách hàng / NCC / Nhân viên / CN khác
    ├── transactionPrice   Giá trên phiếu nguồn
    ├── costPrice          Giá vốn WAC tại thời điểm giao dịch
    ├── quantity           CÓ DẤU: âm = xuất, dương = nhập
    └── runningBalance     Tồn cuối — denormalized running balance
```

**Sắp xếp mặc định:** DESC theo thời gian (mới nhất trên cùng).

### 1.3 Sample data quan sát được

Sản phẩm `AUTOinventoryCase1`, tồn hiện tại = **−6**:

| Chứng từ | Thời gian | Loại GD | Đối tác | Giá GD | Giá vốn | SL | Tồn cuối |
|---|---|---|---|---|---|---|---|
| HD016341 | 09/12/2025 15:10 | Bán hàng | Khách lẻ | 994,856 | 0 | −1 | −6 |
| HD016193 | 07/11/2025 18:02 | Bán hàng | Phan ngân | 994,856 | 0 | −1 | −5 |
| HD016152 | 01/11/2025 14:19 | Bán hàng | Phan ngân | 994,856 | 0 | −1 | −4 |
| HD015906 | 10/09/2025 16:29 | Bán hàng | Phan ngân | 994,856 | 0 | −1 | −3 |
| HD015742 | 01/08/2025 12:08 | Bán hàng | Phan ngân | 400,000 | 0 | −1 | −2 |
| HD015266 | 23/05/2025 16:16 | Bán hàng | Phan ngân | 994,856 | 0 | −1 | −1 |

Quan sát:
- 6 lần bán liên tiếp → tồn −6 (chưa nhập hàng lần nào)
- `costPrice = 0` vì SP chưa từng nhập kho → KiotViet **cho phép bán âm tồn**
- 1 lần bán giá 400k khác giá mặc định 994k (sale đột xuất)

---

## 2. Entity StockCard / StockCardLine — Schema đầy đủ

| Trường | Kiểu | Ghi chú |
|---|---|---|
| `id` | bigint PK | Internal |
| `productId` | FK → E01 | SP nào |
| `branchId` | FK → Branch | CN nào — vì Tồn = SP × CN |
| `documentCode` | string | Mã chứng từ hiển thị (HD016341) — E13 |
| `documentType` | enum | HD / PN / TRF / KK / SX / XN / XH / TH / TPN |
| `documentId` | FK | Link tới phiếu nguồn |
| `transactionType` | enum | Loại giao dịch — E14 |
| `partnerType` | enum | Customer / Supplier / Branch / Employee |
| `partnerId` | FK | Đối tác |
| `partnerName` | string | Cache name (đối tác có thể đổi tên sau) |
| `transactionPrice` | decimal | Giá GD |
| `costPrice` | decimal | Giá vốn WAC tại thời điểm |
| `quantity` | decimal | CÓ DẤU (− xuất, + nhập) |
| `runningBalance` | decimal | Tồn cuối — **denormalized** |
| `transactionAt` | datetime | Thời gian giao dịch |
| `createdAt` | datetime | Audit |
| `createdBy` | FK → User | Người tạo |
| `unitId` | FK → E02 | Đơn vị tính (cái/hộp/thùng) |
| `unitConversion` | decimal | Hệ số quy đổi về đơn vị cơ bản |
| `batchId` | FK → E32 | Nullable — chỉ có khi SP track theo Lô |

### Vấn đề `runningBalance` (denormalized)

`Tồn cuối` lưu trực tiếp vào DB để query nhanh, nhưng:
- **Recalculation cost cao** nếu chèn dòng thẻ kho có thời gian backdate — phải recompute tất cả dòng sau đó
- **Concurrency risk** nếu 2 phiếu cùng tác động lên 1 SP × CN cùng thời điểm — cần lock
- Đây là điểm phức tạp nhất của inventory engine

---

## 3. Mã chứng từ (Document Code — E13)

### 3.1 Quy tắc đặt mã

Format: **[PREFIX][6 digits]** — VD: `HD016341`, `PN000199`, `TRF000015`

| Prefix | Tên đầy đủ | Nghiệp vụ | Dấu SL | Xác nhận |
|---|---|---|---|---|
| **HD** | Hóa Đơn | Bán hàng | − | Confirmed |
| **PN** | Phiếu Nhập | Nhập hàng từ NCC | + | Confirmed |
| **TRF** | Transfer | Chuyển hàng giữa CN | − ở CN gửi, + ở CN nhận | Confirmed |
| **TH** | Trả Hàng | Khách trả hàng | + | Suy luận |
| **TPN** | Trả hàng Nhập | Trả NCC | − | Suy luận |
| **DH** | Đặt Hàng | Order chưa xuất kho | Không tác động | Suy luận |
| **DHN** | Đặt Hàng Nhập | PO chưa về | Không tác động | Suy luận |
| **KK** | Kiểm Kho | Stock take adjustment | ± tùy lệch | Suy luận |
| **SX** | Sản Xuất | Manufacturing/Assembly | NVL −, TP + | Suy luận |
| **XN** | Xuất Nội bộ | Internal use | − | Suy luận |
| **XH** | Xuất Hủy | Write-off | − | Suy luận |

### 3.2 Tính chất mã

1. **Sequential per type** — mỗi loại đếm độc lập (HD016341 ≠ PN016341)
2. **6 chữ số padded** — tối đa 999,999 phiếu/loại
3. **Per merchant scope** — unique trong 1 tài khoản KiotViet, không global
4. **Immutable** — mã không đổi kể cả khi hủy phiếu
5. **Clickable link** trên Thẻ kho → mở popup chi tiết phiếu nguồn

### 3.3 Quan hệ Mã chứng từ ↔ Phiếu nguồn

```
StockCardLine.documentCode ──► SourceDocument.code
                                        │
                           ┌────────────┼──────────────┬──────────────┐
                           ▼            ▼              ▼              ▼
                        Hóa đơn    Phiếu Nhập     Chuyển hàng    Kiểm kho ...
                        (HD)       (PN)           (TRF)          (KK)
```

1 chứng từ có thể sinh **N dòng thẻ kho** (mỗi line item = 1 dòng × CN tương ứng).

VD: `TRF000015` (2 items) sinh 4 dòng:
- CN gửi: −5 bò húc, −2 bánh ngọt
- CN nhận: +5 bò húc, +2 bánh ngọt

---

## 4. Loại giao dịch (Transaction Type — E14)

Enum mô tả nghiệp vụ tạo ra dòng thẻ kho:

| Loại | Dấu | Tác động | Prefix |
|---|---|---|---|
| Bán hàng | − | Giảm tồn CN bán | HD |
| Trả hàng (KH trả) | + | Tăng tồn CN tiếp nhận | TH |
| Nhập hàng (từ NCC) | + | Tăng tồn + cập nhật giá vốn | PN |
| Trả hàng nhập (trả NCC) | − | Giảm tồn | TPN |
| Chuyển hàng đi | − | Giảm tồn CN gửi | TRF |
| Chuyển hàng đến | + | Tăng tồn CN nhận | TRF (cùng mã) |
| Kiểm kho — lệch tăng | + | Tăng tồn | KK |
| Kiểm kho — lệch giảm | − | Giảm tồn | KK |
| Sản xuất — xuất NVL | − | Giảm NVL | SX |
| Sản xuất — nhập thành phẩm | + | Tăng thành phẩm | SX (cùng mã) |
| Xuất dùng nội bộ | − | Giảm tồn, không HĐ | XN |
| Xuất hủy | − | Giảm tồn, ghi chi phí | XH |

---

## 5. UI/UX trên màn hình Thẻ kho

### 5.1 Cấu trúc hiển thị

```
┌─────────────────────────────────────────────────────────────────┐
│  TAB: Thông tin | Mô tả ghi chú | Thẻ kho ✓ | Tồn kho | LK kênh│
├─────────────────────────────────────────────────────────────────┤
│  Chứng từ │ Thời gian │ Loại GD │ Đối tác │ Giá GD ⓘ │ Giá vốn │ SL │ Tồn cuối │
│  ─────────────────────────────────────────────────────────────  │
│  HD016341 │ 09/12/25  │ Bán hàng│ Khách lẻ│ 994,856  │ 0       │-1 │  -6     │
│  HD016193 │ 07/11/25  │ Bán hàng│ Phan ngân│ 994,856 │ 0       │-1 │  -5     │
│  ...                                                            │
├─────────────────────────────────────────────────────────────────┤
│  [Xuất file]                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Hiện có

- Sort DESC theo thời gian
- Click mã chứng từ → mở popup chi tiết phiếu nguồn
- Xuất file (CSV/Excel)
- Tooltip "ⓘ" cạnh Giá GD

### 5.3 Còn thiếu

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có filter trong Thẻ kho (theo loại GD, thời gian, CN, người tạo) | Toolbar filter — quan trọng khi SP có >100 dòng |
| 2 | Không thấy chi nhánh của mỗi dòng | Thêm cột Chi nhánh hoặc filter CN |
| 3 | Không có đơn vị tính — SL hiển thị theo đơn vị nào? | Thêm cột UoM, hoặc quy về đơn vị cơ bản |
| 4 | Không phân biệt visual giữa các loại GD | Color-code / icon theo Loại GD |
| 5 | Không có summary footer (Tổng nhập, Tổng xuất, Tồn đầu, Tồn cuối kỳ) | Footer aggregate row |
| 6 | `costPrice = 0` không rõ là "không có dữ liệu" hay "giá 0 thật" | UI hint: "—" thay vì 0 khi missing |
| 7 | Không có search trong thẻ kho (theo mã chứng từ, đối tác) | Search bar trong tab |
| 8 | Không có view "Thẻ kho gộp SP cha + biến thể con" | Toggle gộp variants |
| 9 | Không hiển thị người thực hiện trên dòng | Cột "Người tạo" hoặc tooltip |
| 10 | Phân trang không thấy — list dài load chậm | Pagination hoặc virtual scroll |

---

## 6. Đối chiếu với chuẩn kế toán

Thẻ kho KiotViet tương đương:
- **Stock Card** trong ERP truyền thống (SAP Material Document)
- **Inventory Ledger** trong QuickBooks / Xero
- **Mẫu S12-DN** Sổ chi tiết vật tư hàng hóa — Thông tư 200/2014/TT-BTC

**Khác biệt với kế toán:**
- Thẻ kho không phải sổ kế toán tổng hợp — không có cột Nợ/Có
- KiotViet có module "Thuế & Kế toán" riêng
- Thẻ kho có thể inconsistent với sổ kế toán nếu chu kỳ chốt sổ khác nhau

---

## 7. Cơ hội đột phá

### 7.1 Quick wins

1. Bổ sung filter + search + summary footer (đã list §5.3)
2. Color code theo loại GD — visual recognition nhanh
3. Cột Chi nhánh — hiện thiếu

### 7.2 Cải tiến lớn

4. **Cross-product Stock Ledger view** — màn hình standalone xem thẻ kho qua nhiều SP với filter, để kế toán đối soát
5. **Inventory Audit Trail Hub** — search qua nhiều chiều: người làm, CN, loại GD, khoảng thời gian
6. **AI Anomaly Detection** trên thẻ kho:
   - Phát hiện chuỗi xuất bất thường (nhiều lần hủy phiếu, void liên tục)
   - Phát hiện nhân viên có pattern khả nghi (xuất hủy nhiều SP đắt)
   - Cảnh báo gap kiểm kho lặp lại trên cùng SKU
7. **Time-travel "Tồn tại thời điểm X"** — query rollback từ thẻ kho, phục vụ báo cáo định kỳ
8. **Real-time Stock Card Stream API** — tích hợp ERP/BI realtime, không phải poll

---

## 8. Tóm lược

**Thẻ kho KiotViet:**
- Single source of truth cho mọi biến động tồn — đủ chi tiết để audit
- Mã chứng từ: **PREFIX + 6 digits**, prefix theo loại nghiệp vụ
- Hỗ trợ **âm tồn** — tốt cho seller online pre-order
- Click mã → mở phiếu nguồn — UX tốt
- **Thiếu:** filter trong tab, cột chi nhánh, summary footer, cross-product view
- **Cơ hội:** AI audit, real-time API, time-travel query

Module đã hoàn thiện về core — còn nhiều room cho UX và intelligence layer.
