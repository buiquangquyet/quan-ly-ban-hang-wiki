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

## 8. Màn hình Chi tiết Thẻ kho — Screen Spec

### 8.1 Hai màn hình cần phân biệt

| Màn hình | Điểm truy cập | Scope | Hiện có |
|---|---|---|---|
| **A. Tab Thẻ kho trong SP** | SP Detail → Tab "Thẻ kho" | 1 SP × CN filter | ✅ Đã có |
| **B. Cross-product Ledger** | Kho hàng → Tab "Thẻ kho" | Multi-SP, multi-CN | ❌ Chưa có |

Màn hình B là **cơ hội đột phá** — kế toán/quản lý cần xem dòng chảy tồn kho toàn kho mà không phải vào từng SP.

---

### 8.2 Layout (ASCII)

**Màn hình A — Tab Thẻ kho trong SP:**

```
┌──────────────────────────────────────────────────────────────────────┐
│ [SP selector: "Giay Adidas Ultraboost 22 — SP001234" ▼]              │
│ [CN ▼]  [30 ngay qua ▼]  [Loai GD ▼]  [Bien the ▼]  [Xuat file]    │
├──────────────────────────────────────────────────────────────────────┤
│ ■ SP001234 — Giay Adidas Ultraboost 22 | WAC: 1.620.000d             │
│   Ton CN Q1: 15  |  Ton CN TT: 8  |  Tong: 23  |  Kha dung: 23     │
├──────────────────────────────────────────────────────────────────────┤
│ Chung tu  │ Thoi gian │ Loai GD     │ Chi nhanh │ Doi tac   │ Gia GD │ Gia von │  SL │ Ton cuoi │
│ ──────────────────────────────────────────────────────────────────── │
│ HD016580  │ 21/05 ... │ Ban hang    │ CN Q1     │ Nguyen... │ 2.35M  │ 1.62M   │  -1 │    15   │ ← row bg: #fef2f2
│ TRF000088 │ 28/11 ... │ Chuyen hang │ CN Q1     │ → CN TT   │   —    │ 1.62M   │  -3 │    16   │ ← row bg: #fffbeb
│ XH000008  │ 05/05 ... │ Xuat huy    │ CN Q1     │ Hang hong │   —    │ 1.62M   │  -2 │    19   │ ← row bg: #fef2f2
│ PN000312  │ 01/12 ... │ Nhap hang   │ CN Q1     │ Adidas VN │ 1.62M  │ 1.62M   │ +20 │    21   │ ← row bg: #f0fdf4
│ KK000045  │ 01/05 ... │ Kiem kho    │ CN Q1     │ Dinh ky   │   —    │ 1.62M   │  +2 │     3   │ ← row bg: #eff6ff
│ HD015906  │ 10/09 ... │ Ban hang    │ CN Q1     │ Tran...   │ 2.35M  │ 1.58M   │  -1 │     1   │ ← row bg: #fef2f2
├──────────────────────────────────────────────────────────────────────┤
│ Ton dau ky (01/05): 1   Tong nhap: +20   Tong xuat ban: -5          │
│ Tong chuyen di: -3      Tong huy: -2     Can bang KK: +2             │
│ Ton cuoi: 15            Gia tri ton kho: 24.300.000d                 │
│                                              Hien thi 6 / 38 GD     │
└──────────────────────────────────────────────────────────────────────┘
```

**Màn hình B — Cross-product Ledger (Kho hang → Tab The kho):**

```
┌──────────────────────────────────────────────────────────────────────┐
│ [Theo san pham ●]  [Toan bo GD]                                      │
├──────────────────────────────────────────────────────────────────────┤
│ [CN ▼]  [30 ngay qua ▼]  [Loai GD ▼]  [Nguoi tao ▼]  [Xuat file]  │
├──────────────────────────────────────────────────────────────────────┤
│ Hang hoa         │ Chung tu  │ Loai GD     │ Chi nhanh │ SL │ Ton cuoi │
│ Giay Adidas...   │ HD016580  │ Ban hang    │ CN Q1     │ -1 │   15    │
│ Sua Vinamilk...  │ HD016579  │ Ban hang    │ CN TT     │ -6 │  244    │
│ Giay Adidas...   │ TRF000088 │ Chuyen hang │ CN Q1→TT  │ -3 │   16   │
│ ...                                                                  │
└──────────────────────────────────────────────────────────────────────┘
```

---

### 8.3 Use Cases

#### UC-01: Xem thẻ kho một sản phẩm

| | |
|---|---|
| **Trigger** | Người dùng chọn SP trong màn hình A hoặc vào SP Detail → Tab "Thẻ kho" |
| **Main flow** | 1. Chọn SP → hệ thống load filter bar + product banner + ledger table<br>2. Banner hiển thị tồn từng CN, WAC hiện tại, tồn tổng, khả dụng<br>3. Ledger load 50 dòng đầu, DESC theo `transactionAt`<br>4. Mỗi dòng color-coded theo loại GD<br>5. Footer tổng hợp hiển thị: tồn đầu kỳ, tổng nhập/xuất, tồn cuối, giá trị tồn kho |
| **AC-01.1** | Khi SP được chọn, banner hiển thị đúng tồn từng CN và WAC hiện tại |
| **AC-01.2** | Ledger sắp xếp DESC theo `transactionAt` mặc định |
| **AC-01.3** | Dòng nhập hàng (PN) có background xanh lá; dòng bán hàng (HD/TH) có background đỏ nhạt; dòng chuyển hàng (TRF) có background vàng; dòng kiểm kho (KK) có background xanh dương; dòng xuất hủy/nội bộ (XH/XN) có background đỏ đậm nhạt |
| **AC-01.4** | Footer tính đúng: Tổng nhập = Σ quantity dương, Tổng xuất = |Σ quantity âm|, Giá trị tồn kho = Tồn cuối × WAC |

#### UC-02: Filter & search trong thẻ kho

| | |
|---|---|
| **Trigger** | Người dùng thay đổi filter (CN, khoảng thời gian, loại GD, biến thể) |
| **Main flow** | 1. User chọn filter → table reload ngay lập tức (không cần submit)<br>2. Filter CN → chỉ hiện dòng của CN đó; `runningBalance` tính trong scope CN<br>3. Filter loại GD → ẩn dòng không khớp nhưng `runningBalance` không bị ảnh hưởng<br>4. Filter thời gian → chỉ hiện dòng trong khoảng; footer tính lại tồn đầu kỳ |
| **AC-02.1** | Filter CN → `runningBalance` cột "Tồn cuối" chỉ reflect tồn của CN đó |
| **AC-02.2** | Filter loại GD không làm sai `runningBalance` — chỉ ẩn hiển thị |
| **AC-02.3** | Filter thời gian → tồn đầu kỳ trong footer tính đúng từ dòng trước khoảng thời gian |
| **AC-02.4** | Search theo mã chứng từ (VD: "HD016") → highlight tất cả dòng khớp |

#### UC-03: Click chứng từ → popup phiếu nguồn

| | |
|---|---|
| **Trigger** | Người dùng click vào mã chứng từ (VD: HD016580) trên thẻ kho |
| **Main flow** | 1. Mở popup modal chi tiết phiếu nguồn tương ứng<br>2. Popup hiển thị đầy đủ thông tin phiếu (giống màn hình chi tiết HD/PN/TRF...)<br>3. Có nút "Xem đầy đủ" → navigate sang màn hình chi tiết phiếu nguồn |
| **AC-03.1** | Popup mở trong ≤ 300ms |
| **AC-03.2** | Popup đúng loại phiếu — HD016580 mở chi tiết Hóa đơn, PN000312 mở chi tiết Phiếu nhập |
| **AC-03.3** | Phiếu đã hủy → popup hiển thị trạng thái "Đã hủy" với banner cảnh báo |

#### UC-04: Cross-product Stock Ledger

| | |
|---|---|
| **Trigger** | Người dùng vào Kho hàng → Tab "Thẻ kho" → sub-tab "Toan bo GD" |
| **Main flow** | 1. Hiển thị tất cả giao dịch tồn kho của tất cả SP, sắp xếp DESC theo thời gian<br>2. Filter: CN, thời gian, loại GD, người tạo — không có filter SP<br>3. Có thể lọc theo người tạo để audit nhân viên |
| **AC-04.1** | Hiển thị cột "Hàng hóa" + SKU trước cột chứng từ |
| **AC-04.2** | Filter "Người tạo" hoạt động, thu hẹp về GD của 1 nhân viên |
| **AC-04.3** | Tổng số GD hiển thị ở footer |

#### UC-05: Xuất file thẻ kho

| | |
|---|---|
| **Trigger** | Người dùng click "Xuất file" |
| **Main flow** | 1. Hiển thị dialog chọn định dạng (CSV/Excel)<br>2. Xuất toàn bộ dòng khớp filter hiện tại (không giới hạn 50 dòng)<br>3. File có đầy đủ cột bao gồm cả `runningBalance` |
| **AC-05.1** | File CSV/Excel chứa đúng dữ liệu theo filter đang áp dụng |
| **AC-05.2** | Xuất ≤ 10,000 dòng trong ≤ 5 giây |
| **AC-05.3** | Tên file: `the-kho_{SP_code}_{date_range}.csv` |

---

### 8.4 Business Rules

| # | Rule | Chi tiết |
|---|---|---|
| **BR-01** | Chỉ confirmed documents ghi vào thẻ kho | DH/DHN (draft order) không tác động tồn → không xuất hiện trên thẻ kho |
| **BR-02** | `runningBalance` nhất quán | Tổng tất cả `quantity` từ dòng đầu đến dòng hiện tại = `runningBalance` của dòng hiện tại. Bất kỳ vi phạm nào = data integrity bug |
| **BR-03** | Thứ tự mặc định DESC | Sắp xếp theo `transactionAt` DESC — dòng mới nhất trên cùng |
| **BR-04** | `costPrice = 0` → hiển thị "—" | Không hiển thị "0" để tránh nhầm lẫn với "giá vốn thật là 0đ" |
| **BR-05** | Âm tồn được phép | `runningBalance < 0` hiển thị màu đỏ nhưng không block giao dịch |
| **BR-06** | Filter CN scopes `runningBalance` | Khi filter CN Q1, cột "Tồn cuối" phản ánh tồn CN Q1, không phải tồn tổng |
| **BR-07** | Filter loại GD chỉ ảnh hưởng hiển thị | `runningBalance` không bị recalculate khi ẩn dòng — đây là giá trị lịch sử |
| **BR-08** | Backdate document → recompute | Khi nhập phiếu backdate, hệ thống phải recompute `runningBalance` cho tất cả dòng sau `transactionAt` của phiếu đó trong cùng SP × CN |

---

### 8.5 Non-Functional Requirements

| # | NFR | Target |
|---|---|---|
| **NFR-01** | Initial load | Màn hình thẻ kho render ≤ 1s cho SP có ≤ 500 dòng; pagination/virtual scroll cho > 500 dòng |
| **NFR-02** | Popup phiếu nguồn | Mở trong ≤ 300ms |
| **NFR-03** | Xuất file | CSV/Excel ≤ 10,000 dòng trong ≤ 5s |
| **NFR-04** | Concurrent access | Không race condition khi 2 phiếu cùng thời điểm tác động SP × CN — dùng optimistic locking hoặc serialized queue |

---

## 9. Tóm lược

**Thẻ kho KiotViet:**
- Single source of truth cho mọi biến động tồn — đủ chi tiết để audit
- Mã chứng từ: **PREFIX + 6 digits**, prefix theo loại nghiệp vụ
- Hỗ trợ **âm tồn** — tốt cho seller online pre-order
- Click mã → mở phiếu nguồn — UX tốt

**Gaps cần fix (theo priority):**
1. Filter trong tab (CN, thời gian, loại GD) — thiếu cơ bản nhất
2. Cột Chi nhánh — hiện không có
3. Summary footer (tồn đầu kỳ, tổng nhập/xuất, giá trị tồn kho)
4. Color-code dòng theo loại GD — visual recognition
5. Cross-product ledger view (màn hình B) — cho kế toán audit

**Cơ hội lớn:**
- AI Anomaly Detection trên thẻ kho (phát hiện pattern khả nghi)
- Time-travel query "Tồn tại thời điểm X"
- Real-time Stock Card Stream API

Module đã hoàn thiện về core — nhiều room cho UX layer và intelligence layer.
