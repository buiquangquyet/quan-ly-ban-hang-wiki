# Lô / Hạn sử dụng & Serial / IMEI — Deep Dive

**Bối cảnh:** KiotViet có 2 chế độ tracking nâng cao — riêng biệt, có thể bật song song:

| Chế độ | Đối tượng | Ngành áp dụng |
|---|---|---|
| **Lô + Hạn sử dụng (Batch)** | Nhóm units cùng đợt SX/HSD | Thực phẩm, dược, mỹ phẩm |
| **Serial / IMEI** | Từng unit vật lý cá biệt | Điện thoại, điện máy, xe, đồng hồ |

**Entities:** E32 (Batch), E33 (BatchStock), E34 (StockUnit/Serial), E35 (WarrantyRecord), E08 (ProductImage)
**Liên quan:** [README.md](./README.md) | [03-nhap-hang.md](./03-nhap-hang.md) | [04-ban-hang.md](./04-ban-hang.md)

---

## 1. Ảnh sản phẩm — Lưu ở Hàng hóa (E08)

### 1.1 Quan sát từ testzone

Trong form Tạo/Sửa hàng hóa:
- Khu vực ảnh nằm ở **Thông tin chung của SP** (tab "Thông tin")
- Tối đa nhiều ảnh (placeholder 4-5 ảnh) + "Thêm ảnh"
- Mỗi ảnh không quá **2 MB**

Trong form Chi nhánh kinh doanh:
- Chỉ có "Toàn hệ thống" / "Chi nhánh cụ thể"
- **Không có ảnh khác theo từng chi nhánh**

→ **Kết luận: Ảnh là attribute của Product, áp dụng đồng nhất cho mọi chi nhánh.**

### 1.2 Tại sao đặt ở Product chứ không phải Kho?

| Tiêu chí | Product (Hàng hóa) | Inventory/Stock (Kho) |
|---|---|---|
| Bản chất | Mô tả **LOẠI** hàng | Mô tả **SỐ LƯỢNG** & vị trí vật lý |
| Vòng đời | Lâu dài (suốt đời SP) | Thay đổi mỗi giao dịch |
| Owner | Marketing/Catalog team | Warehouse team |

Ảnh là **catalog data** — không phụ thuộc hàng đang ở kho nào. Đặt ở Kho → duplicate, inconsistency.

### 1.3 Schema ProductImage (E08)

```
Product (E01)
└── images: ProductImage[]
    ├── id
    ├── productId  (FK → E01)
    ├── url
    ├── displayOrder
    ├── isPrimary
    └── fileSize   (≤ 2MB)
```

Biến thể nâng cao (KiotViet có thể đã implement):
- **Ảnh theo Variant** — mỗi variant (size/màu) có ảnh riêng → vẫn ở Product/Variant level, không phải Kho

---

## 2. Lô + Hạn sử dụng (Batch — E32)

### 2.1 Bật tính năng

**Đường dẫn:** Thiết lập → Hàng hóa → "Giá vốn, tồn kho" → Toggle **"Quản lý tồn kho theo Lô, hạn sử dụng"**

> Theo dõi các hàng hóa như thực phẩm hoặc thuốc theo Lô, hạn sử dụng trong mọi giao dịch.

Mặc định: **OFF**. Khi bật → áp dụng toàn tài khoản, nhưng từng SP vẫn opt-in riêng.

### 2.2 Bật cho từng SP (product-level)

Khi toggle account-level ON, form Tạo/Sửa SP xuất hiện dropdown **"Quản lý theo lô, hạn sử dụng"**: Có / Không.

**Behavior thay đổi:**

| Trước khi bật Lô | Sau khi bật Lô |
|---|---|
| Field "Tồn kho" hiển thị, nhập tay được | Field "Tồn kho" **bị ẩn** |
| Tồn = 1 số đơn lẻ | Tồn = SUM(BatchStock.qty) qua các Lô |

→ Khi bật Lô, tồn kho **không còn là attribute của SP** mà là attribute của **từng Lô**. SP chỉ là parent.

### 2.3 Workflow đầy đủ

```
TẠO SP với "Quản lý theo lô = Có"
  → KHÔNG nhập tồn ban đầu

NHẬP HÀNG (PN — E15)
  Khi thêm SP có Lô vào phiếu nhập:
  → "Thêm mới lô, hạn sử dụng"
  → Form: [Số lô] [Hạn sử dụng] [Số lượng]
  → 1 phiếu PN có thể chứa NHIỀU lô của cùng 1 SP
  → Hoàn thành PN → tăng tồn theo từng lô

BÁN HÀNG (HD — E19)
  Khi thêm SP có Lô vào hóa đơn:
  → Auto-suggest FEFO (First Expired First Out)
  → User có thể chọn lô khác từ dropdown
  → Cảnh báo màu sắc:
      VÀNG: lô sắp hết hạn
      ĐỎ:   lô đã hết hạn
  → Giảm tồn của lô được chọn

TRẢ HÀNG (TH — E23)
  → Chọn đúng lô đã bán → tăng tồn lô đó

CHUYỂN HÀNG (TRF — E27)
  → Auto suggest FEFO khi chuyển đi
  → CN nhận: hàng giữ thông tin Lô gốc

KIỂM KHO (KK — E28)
  → Phải nhập SL thực tế CHO TỪNG LÔ
  → Không thể kiểm tổng

XUẤT HỦY (XH — E31)
  → Chọn chính xác lô cần hủy
  → Use case chính: hủy lô đã hết hạn
```

### 2.4 Báo cáo HSD

**Vị trí:** Phân tích → Hàng hóa → Mối quan tâm: **Hạn sử dụng**

Báo cáo liệt kê: lô sắp hết hạn (chưa hết) và lô đã hết hạn → lên kế hoạch khuyến mại, xả hàng, chủ động hủy lô.

---

## 3. Data model của Lô

### 3.1 Entities

```
Product (E01)
   │ Flag: trackByBatch = true
   │ 1:N
   ▼
Batch (E32)
├── id (PK)
├── productId       FK → E01
├── batchCode       string   số lô do user nhập
├── manufactureDate nullable
├── expireDate      date     hạn sử dụng
├── createdAt
└── createdViaDocumentCode   PN... — phiếu nhập tạo lô

BatchStock (E33 — junction Batch × Branch)
├── id (PK)
├── batchId     FK → E32
├── branchId    FK → Branch
├── quantity    tồn của lô này tại CN này
└── lastMovementAt
```

**Lưu ý:** `Batch` KHÔNG có `branchId` — vì 1 lô có thể trải qua nhiều CN sau khi chuyển hàng. `BatchStock` mới là nơi lưu tồn theo CN.

### 3.2 Thẻ kho khi bật Lô

Mỗi dòng Thẻ kho liên quan tới SP có Lô bật → thêm `batchId`:

```
StockCardLine (E10)
├── ...các trường cũ...
└── batchId (nullable — chỉ có khi SP track theo Lô)
```

Hệ quả: 1 phiếu nhập (PN) với 1 SP nhập 3 lô khác nhau → sinh **3 dòng Thẻ kho** (mỗi lô 1 dòng).

### 3.3 Aggregate query tồn

```sql
Tồn của SP X tại CN Y
= SUM(BatchStock.quantity
      WHERE Batch.productId = X
        AND BatchStock.branchId = Y)
```

Khi UI hiển thị "Tồn kho: 150" cho SP có Lô → là tổng. Click vào thấy breakdown:
- Lô L001 (HSD 30/06/2026): 50
- Lô L002 (HSD 15/08/2026): 70
- Lô L003 (HSD 20/10/2026): 30
- Tổng: 150

### 3.4 Logic FEFO (giả định)

```pseudo
function suggestBatchToSell(productId, branchId, qty):
    batches = BatchStock
        .where(productId, branchId)
        .where(quantity > 0)
        .orderBy('expireDate', ASC)   // hết hạn trước → xuất trước

    result = []
    remaining = qty
    for batch in batches:
        if remaining <= 0: break
        take = min(batch.quantity, remaining)
        result.append({batch, take})
        remaining -= take

    if remaining > 0: warn("Không đủ tồn")
    return result
```

User có thể click dropdown → chọn lô khác, bypass FEFO.

---

## 4. Serial / IMEI Tracking (E34)

### 4.1 Trạng thái trong KiotViet

KiotViet có Serial tracking — có thể bật trong cài đặt (tương tự Lô). Test account 17 chưa enable, nhưng tính năng có trong tài liệu chính thức.

### 4.2 Tại sao Serial KHÔNG đặt ở Product hay Kho aggregate?

**Không được đặt ở Product:**
- 1 SP "iPhone 15 Pro Max 256GB" có 1000 cái bán ra → 1000 IMEI khác nhau
- 1 product = N units — không thể lưu serial ở Product level

**Không được đặt ở Stock aggregate (E21):**
- `Stock(productId=X, branchId=A)` = tổng số → mất thông tin từng unit cụ thể

**Đúng: Đặt ở entity riêng StockUnit / ProductInstance (E34)**

### 4.3 Schema StockUnit (E34)

```
Product (E01)
   │ 1:N (nếu bật Serial)
   ▼
StockUnit / ProductInstance (E34)
├── id (PK)
├── productId        FK → E01
├── serialNumber     string   UNIQUE — mã serial/IMEI
├── currentBranchId  FK       vị trí hiện tại
├── status           enum     in_stock | sold | returned | lost | damaged | transferred
├── inboundAt        datetime ngày nhập
├── inboundDocumentCode       PN... — phiếu nhập
├── outboundAt       datetime nullable
├── outboundDocumentCode      HD... — hóa đơn bán
├── currentOwnerCustomerId    nullable FK → Customer (nếu đã bán)
├── batchId          nullable FK → E32 (nếu cùng lô SX)
└── warrantyExpireAt          hạn bảo hành (E35)
```

**Quan hệ với Thẻ kho:**
- Khi xuất hóa đơn, cashier **chọn cụ thể serial nào** xuất → ghi nhận vào thẻ kho
- Bán 1 chiếc iPhone không phải "giảm 1 cái khỏi tổng tồn" — là **đánh dấu unit cụ thể là 'sold'**

### 4.4 Sơ đồ data model 3 tầng

```
               ┌─────────────────┐
               │   Product (E01) │  ← Catalog data (ảnh, mô tả, giá)
               │                 │     chung cho mọi unit
               └────────┬────────┘
                        │ 1:N (nếu bật Lô)
                        ▼
               ┌─────────────────┐
               │  Batch (E32)    │  ← Nhóm units cùng đợt SX/HSD
               │  - Số lô        │
               │  - Ngày SX      │
               │  - Hạn SD       │
               └────────┬────────┘
                        │ 1:N (nếu bật Serial)
                        ▼
               ┌─────────────────┐
               │ StockUnit (E34) │  ← Đơn vị vật lý cá biệt
               │ - Serial/IMEI   │     (mỗi cái 1 record)
               │ - Vị trí hiện tại│
               │ - Trạng thái    │
               └─────────────────┘
```

**Ví dụ áp dụng:**
- **Mỳ omì bò hầm** → 50 lô SX khác ngày → StockUnit không cần thiết (chỉ track tới Lô)
- **iPhone 15 Pro Max** → không có lô → 1000 cái mỗi cái 1 IMEI (StockUnit cần thiết)
- **Thuốc Paracetamol** → 20 lô khác HSD → mỗi lô 5000 vỉ (chỉ track tới Lô, không tới vỉ)

---

## 5. So sánh Lô vs Serial

| Tiêu chí | Lô (E32) | Serial / IMEI (E34) |
|---|---|---|
| Đặc điểm | Nhóm N units cùng SX/HSD | 1 unit cá biệt |
| Cardinality | 1 SP → N Lô (mỗi Lô M units) | 1 SP → N Serial (mỗi cái 1 record) |
| Use case | Thực phẩm, dược, mỹ phẩm | Điện thoại, điện máy, xe, đồng hồ |
| Định danh | Số lô + HSD | Mã Serial/IMEI unique toàn cầu |
| Logic xuất ưu tiên | **FEFO** (hết hạn trước xuất trước) | User chọn cụ thể |
| Cảnh báo | Sắp hết hạn / Đã hết hạn (vàng/đỏ) | Bảo hành gần hết |
| Bảo hành | Theo lô | **Theo từng serial cá nhân** |
| Chống tráo hàng | Yếu (nhiều unit chung lô) | Mạnh (1-1 mapping) |
| Volume nhập | Hàng trăm/đợt/lô | Quét từng cái khi nhập |
| Bật song song? | Có thể (1 SP đặc biệt có cả 2) | Hiếm nhưng không cấm |

---

## 6. UI/UX — Lô

### 6.1 Cảnh báo trực quan trên màn POS

| Trạng thái lô | Màu sắc | Hành động |
|---|---|---|
| Bình thường | Mặc định | Bán bình thường |
| Sắp hết hạn | **VÀNG** | Ưu tiên xuất, có thể giảm giá |
| Đã hết hạn | **ĐỎ** | Không nên bán — cảnh báo cashier |

### 6.2 Hỗ trợ đa platform

| Platform | Workflow đầy đủ |
|---|---|
| Web (Quản lý + Bán hàng) | Đầy đủ tất cả nghiệp vụ |
| Mobile app (KiotViet) | Thêm SP, Nhập, Bán |
| Máy POS Android | Pop-up chọn lô khi thêm SP vào HĐ, FEFO suggestion |

---

## 7. Điểm mạnh của tính năng Lô

1. **Track tới level Lô** — đủ cho 90% use case ngành thực phẩm/dược
2. **FEFO tự động** — giảm thất thoát do hết hạn
3. **Cảnh báo màu sắc realtime** — cashier không cần nhớ lô nào sắp hết
4. **Opt-in per SP** — không ép buộc tất cả SP phải có lô
5. **Báo cáo HSD có sẵn** — proactive planning
6. **Hỗ trợ đa platform** — web/mobile/POS
7. **Tích hợp đầy đủ nghiệp vụ** — trả, chuyển, kiểm, hủy đều respect lô

---

## 8. Điểm yếu & cơ hội cải tiến

### 8.1 Lô / HSD

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Chỉ track HSD — không có ngày sản xuất để tính BB (Best Before) khác HSD | Thêm field `manufactureDate` + `bestBeforeDate` riêng |
| 2 | Không có Số lô NCC tách bạch với mã lô nội bộ | Thêm field `supplierBatchCode` |
| 3 | Không có Country of Origin / Country of Manufacturing per lô | Field `origin` — truy xuất nguồn gốc |
| 4 | Không có Recall workflow — NCC thông báo lỗi lô, không có cách bulk recall | Module Lot Recall |
| 5 | FEFO cứng — không cho rule customizable (VD: VIP customer luôn lô mới nhất) | Pluggable allocation strategy |
| 6 | Cảnh báo HSD chỉ trên màn POS — không có push notification cho chủ shop | Notification daily/weekly digest |
| 7 | Không có auto markdown pricing khi lô sắp hết hạn | Auto promotion rule by expiry |
| 8 | Threshold cảnh báo không cấu hình được (mặc định bao nhiêu ngày là "sắp"?) | Cấu hình ngày cảnh báo per SP / per category |
| 9 | Không có truy xuất ngược lô → bán cho ai | Customer batch tracking (quan trọng cho food safety) |

### 8.2 Serial / IMEI

| # | Pain | Đề xuất |
|---|---|---|
| 10 | Seller điện máy, đồng hồ đang dùng Excel + KiotViet riêng biệt | Toggle "Quản lý theo Serial" — tương tự Lô |
| 11 | Không có bảo hành tracking per serial (E35 chưa có) | `warrantyExpireAt` per StockUnit + cảnh báo gần hết bảo hành |
| 12 | Không có scan flow tối ưu khi nhập nhiều serial | Batch scan mode: quét liên tục → tự thêm vào list |
| 13 | Không có view "All Serial of Product X" | Màn hình: biết unit nào còn, đã bán cho ai, ngày nào |

---

## 9. Cơ hội module mới

### M1. Smart Recall Module
- **Pain:** NCC thông báo lỗi lô X → shop phải tìm thủ công ai mua lô đó
- **Solution:** 1-click recall — list tất cả KH đã mua lô X → gửi SMS/Zalo tự động
- **Impact:** High cho dược, thực phẩm — compliance feature

### M2. Auto Markdown Engine
- **Pain:** Lô sắp hết hạn → phải giảm giá tay
- **Solution:** Rule "Khi còn ≤ 7 ngày → giảm 20%, ≤ 3 ngày → giảm 50%" → auto áp khi cashier scan
- **Impact:** Reduce wastage 30–50%

### M3. Expiry Heatmap Dashboard
- View toàn bộ lô của shop trên 1 heatmap theo thời gian
- Trục X: các tháng tới | Trục Y: nhóm hàng / chi nhánh
- Cell color: tổng giá trị lô đáo hạn trong khoảng
- Chủ shop nhìn 1 lần biết "tháng tới đáo hạn 50 triệu hàng tươi tại CN A"

### M4. Compliance Audit Pack
- Cho ngành dược, thực phẩm có thanh tra nhà nước
- Export 1 file đầy đủ: lô vào, lô ra, ai bán, ai mua, hủy lúc nào — chuẩn TT200, GMP, HACCP
- Đính kèm hóa đơn điện tử

### M5. Serial / IMEI Tracking Module
- **Pain:** Seller điện máy, đồng hồ, mỹ phẩm chính hãng, IT đang dùng Excel + KiotViet riêng
- **Solution:** Toggle "Quản lý theo Serial":
  - Khi nhập (PN): scan/nhập từng IMEI vào phiếu nhập
  - Khi bán (HD): scan IMEI khi xuất, in lên hóa đơn để bảo hành
  - Khi trả (TH): scan IMEI verify đúng máy đã bán
  - View "All Serial of Product X" — biết unit nào còn, đã bán cho ai, ngày nào
  - Bảo hành tracking — auto cảnh báo khi unit gần hết hạn BH
- **Effort:** Trung bình (3–5 tháng) — phức tạp nhất là UX scan flow + đối soát bảo hành
- **Impact:** High — unlock segment điện máy, IT, hàng hiệu, mỹ phẩm chính hãng

---

## 10. Tóm lược

**Lô + HSD — đã hoàn thiện mức 7–8/10:**
- Đủ workflow cho nghiệp vụ thường gặp
- FEFO + cảnh báo màu là điểm sáng UX
- Tích hợp toàn diện 6 nghiệp vụ kho + báo cáo

**Serial / IMEI — có tính năng, chưa test được:**
- Entity riêng (E34 — StockUnit) là đúng kiến trúc
- Nằm ở giao điểm Product + Kho — không thuộc bên duy nhất nào
- Cần unlock cho segment điện máy/hàng hiệu

**Vai trò trong data model:**
- Lô là entity **trung gian** giữa Product và physical unit
- Khi bật Lô cho 1 SP: Tồn không còn là attribute của Product mà là **SUM(BatchStock)** qua các Lô
- Thẻ kho thêm `batchId` để audit trail chính xác tới lô

**Đề xuất ưu tiên:** 4 module mới (Smart Recall, Auto Markdown, Heatmap, Serial Tracking) gắn với theme "Tracking nâng cao" + "Compliance" — match segment nhà thuốc, F&B mid-market, điện máy.
