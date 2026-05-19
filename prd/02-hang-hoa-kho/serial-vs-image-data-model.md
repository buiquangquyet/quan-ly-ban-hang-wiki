# SERIAL NUMBER & ẢNH — LƯU Ở "HÀNG HÓA" HAY "KHO"?

**Câu hỏi:** Với mặt hàng có serial number / ảnh — dữ liệu này được ghi nhận ở entity Hàng hóa (Product) hay ở Kho (Inventory)?

**TL;DR:**

| Dữ liệu | KiotViet hiện tại | Best practice data model |
|---|---|---|
| **Ảnh sản phẩm** | ✅ Ở **Hàng hóa** | ✅ Ở **Hàng hóa** (Product) |
| **Serial / IMEI** | ❌ **Không có** trong KiotViet test zone | Phải ở entity riêng **StockUnit** (gắn cả Hàng hóa và Kho × CN) |
| **Lô (Batch)** | ✅ Có (toggle "Quản lý tồn kho theo Lô, hạn sử dụng") | Entity **Lot/Batch** ở giữa Product và unit |

---

## 1. ẢNH SẢN PHẨM — Lưu ở HÀNG HÓA (Product)

### 1.1 Quan sát từ testzone

Trong form `Tạo/Sửa hàng hóa`:
- Khu vực ảnh nằm ở **Thông tin chung của SP** (tab "Thông tin")
- Tối đa nhiều ảnh (placeholder 4-5 ảnh) + "Thêm ảnh"
- Quy định: **Mỗi ảnh không quá 2 MB**

Trong form `Chi nhánh kinh doanh`:
- Chỉ có 2 lựa chọn: "Toàn hệ thống" / "Chi nhánh cụ thể"
- **KHÔNG có ảnh khác theo từng chi nhánh**

→ **Kết luận: Ảnh là attribute của Product, áp dụng đồng nhất cho mọi chi nhánh, mọi đơn vị vật lý.**

### 1.2 Tại sao đặt ở Product chứ không phải Kho?

| Tiêu chí | Product (Hàng hóa) | Inventory/Stock (Kho) |
|---|---|---|
| Bản chất dữ liệu | Mô tả **LOẠI** hàng | Mô tả **SỐ LƯỢNG** & **VỊ TRÍ** vật lý |
| Vòng đời | Lâu dài (suốt đời SP) | Thay đổi mỗi giao dịch |
| Lặp lại | 1 ảnh cho mọi unit | Mỗi unit có vị trí riêng |
| Owner | Marketing/Catalog team | Warehouse team |

Ảnh là **catalog data** — gắn với "thông tin loại hàng này trông như nào", không phụ thuộc đang nằm ở kho nào. Đặt ở Kho sẽ duplicate và inconsistency rủi ro cao.

### 1.3 Schema giả định trong KiotViet

```
Product (Hàng hóa)
├── id, code, name, brand, category, ...
└── images: ProductImage[] ─┐
                              ├── id
                              ├── productId (FK)
                              ├── url
                              ├── displayOrder
                              ├── isPrimary
                              └── fileSize (≤ 2MB)
```

Một số biến thể nâng cao (KiotViet có thể đã implement):
- **Ảnh theo Variant** — nếu SP có biến thể (size/màu), mỗi variant có ảnh riêng → vẫn ở Product/Variant level, không phải Kho
- **Ảnh thumbnail vs full** — tự generate sau khi upload

---

## 2. SERIAL NUMBER / IMEI — Lưu ở đâu?

### 2.1 Quan sát từ testzone

Search trong **Thiết lập → Hàng hóa** với từ khóa:
- `"imei"` → **Không có kết quả phù hợp**
- `"seri"` → **Không có kết quả phù hợp**

Search trên form tạo/sửa SP:
- Không có trường Serial / IMEI
- Không có toggle "Quản lý theo Serial"

→ **Kết luận: KiotViet testzone17 hiện KHÔNG có native serial tracking.** Đây là **gap đáng chú ý** cho ngành điện máy, mỹ phẩm chính hãng, IT, đồng hồ, túi xách hàng hiệu...

### 2.2 Nếu KiotViet implement serial — phải đặt ở ĐÂU?

**KHÔNG được đặt ở Product:**
- 1 SP "iPhone 15 Pro Max 256GB Titanium" có 1000 cái bán ra → 1000 IMEI khác nhau
- Nếu lưu serial ở Product → conflict với khái niệm "1 product = N units"

**KHÔNG được đặt ở Stock aggregate (tổng tồn):**
- Stock(productId=X, branchId=A) = số lượng — nhưng số đó gồm những unit nào (serial nào)?
- Aggregate level mất thông tin individual

**ĐÚNG ĐẮN — Đặt ở entity riêng "StockUnit" / "ProductInstance":**

```
Product (Hàng hóa, 1)
       │
       │ 1:N
       ▼
StockUnit / ProductInstance (mỗi unit vật lý 1 record)
├── id (PK)
├── productId (FK → Product)
├── serialNumber / imei (string, UNIQUE)
├── currentBranchId (FK → Branch) — vị trí HIỆN TẠI
├── status (enum: in_stock | sold | returned | lost | damaged | transferred)
├── inboundAt (datetime) — ngày nhập
├── inboundDocumentCode (PN... — phiếu nhập)
├── outboundAt (datetime, nullable)
├── outboundDocumentCode (HD... — hóa đơn bán)
├── currentOwnerCustomerId (nullable, FK → Customer) — nếu đã bán
├── batchId (nullable, FK → Batch) — nếu cùng lô SX
└── warrantyExpireAt — Hạn bảo hành
```

**Quan hệ với Thẻ kho:**
- Mỗi dòng Thẻ kho **thuộc về 1+ StockUnit** (vì có thể bán nhiều unit cùng 1 phiếu)
- Khi xuất hóa đơn, cashier **chọn cụ thể serial nào** xuất → ghi nhận vào thẻ kho
- Bán 1 chiếc iPhone không phải "giảm 1 cái khỏi tổng tồn" — là **đánh dấu unit cụ thể là 'sold'**

### 2.3 Sơ đồ data model 3 tầng

```
                        ┌─────────────────┐
                        │ Product (1)     │  ← Hàng hóa
                        │ - ảnh           │     (catalog data, chung)
                        │ - mô tả         │
                        │ - giá           │
                        └────────┬────────┘
                                 │ 1:N (nếu bật Lô)
                                 ▼
                        ┌─────────────────┐
                        │ Batch / Lô (N)  │  ← Lô sản xuất
                        │ - Số lô         │     (chung của 1 đợt)
                        │ - Ngày SX       │
                        │ - Hạn SD        │
                        └────────┬────────┘
                                 │ 1:N (nếu bật Serial)
                                 ▼
                        ┌─────────────────┐
                        │ StockUnit (N×M) │  ← Đơn vị vật lý
                        │ - Serial/IMEI   │     (mỗi cái 1 record)
                        │ - Vị trí hiện tại│
                        │ - Trạng thái    │
                        └─────────────────┘
```

- **Mỳ omni bò hầm** (1 product) → 50 lô SX khác ngày → mỗi lô 1000 gói (StockUnit không cần thiết)
- **iPhone 15 Pro Max** (1 product) → KHÔNG có lô → 1000 cái mỗi cái 1 IMEI (StockUnit cần thiết)
- **Thuốc Paracetamol** (1 product) → 20 lô khác HSD → mỗi lô 5000 vỉ (chỉ track tới Lô, không track tới vỉ)

### 2.4 Tại sao serial phải có cả "Product link" và "Branch link"?

**Product link:** để biết unit này là LOẠI gì (lấy ảnh, giá, mô tả)
**Branch link:** để biết unit này đang ở ĐÂU vật lý

→ Serial nằm ở **giao điểm** của Product và Kho — không thuộc 1 bên duy nhất.

---

## 3. LÔ + HẠN SỬ DỤNG (BATCH) — KiotViet ĐÃ CÓ

### 3.1 Settings observable

Tại **Thiết lập → Hàng hóa → Giá vốn, tồn kho**:

> **Quản lý tồn kho theo Lô, hạn sử dụng** [toggle OFF mặc định]
> Theo dõi các hàng hóa như thực phẩm hoặc thuốc theo Lô, hạn sử dụng trong mọi giao dịch.

Đây là toggle **ở cấp tài khoản** (account-level). Khi bật:
- Mỗi SP có flag "track theo Lô" hay không
- Khi nhập hàng (PN) → khai báo số lô + HSD
- Khi xuất hàng (HD/TRF) → chọn lô nào xuất, hệ thống auto-suggest FEFO (First Expired First Out)
- Thẻ kho ghi nhận cả số lô

### 3.2 Lô lưu ở ĐÂU trong data model?

Lô là entity **ở GIỮA** Product và individual unit — không lưu ở Product, không lưu ở Kho aggregate:

```
Product (1)
   │
   │ 1:N
   ▼
Batch / Lô (N)
├── id
├── productId (FK)
├── batchCode (string) — số lô
├── manufactureDate
├── expireDate
├── supplierId (FK) — NCC cung cấp lô này
├── inboundDocumentCode (PN...)
└── (lưu ý: KHÔNG có branchId — vì 1 lô có thể trải nhiều CN sau khi chuyển hàng)

BatchStock (junction)
├── batchId (FK)
├── branchId (FK)
└── remainingQty — số lượng còn của lô này tại CN này
```

**Mỗi dòng Thẻ kho** khi bật Lô phải mang theo `batchId` để biết xuất từ lô nào.

---

## 4. TÓM LƯỢC — TRẢ LỜI CÂU HỎI GỐC

### 4.1 Ảnh — ✅ Ở HÀNG HÓA
- KiotViet: Lưu trong Product, max 2MB/ảnh, áp dụng cho mọi CN
- Lý do: là catalog data, không phụ thuộc kho

### 4.2 Serial number — KiotViet HIỆN KHÔNG CÓ
- Test "imei" và "seri" trong Settings đều ra empty
- Nếu implement: PHẢI ở entity riêng **StockUnit** — không thuộc Product cũng không thuộc Kho, mà là **kết hợp** (Product link + Branch link + Status)
- Đây là **gap rõ ràng** cho phân khúc seller điện máy, đồng hồ, hàng hiệu — opportunity cho module mới

### 4.3 Lô + HSD — KiotViet CÓ (opt-in)
- Toggle ở Settings → Hàng hóa
- Entity riêng **Batch**, ở giữa Product và unit
- Mỗi Thẻ kho có thể tham chiếu batchId khi tính năng được bật

---

## 5. BỔ SUNG ĐỐI TƯỢNG DATA MODEL

| # | Đối tượng | Trạng thái KiotViet | Vai trò |
|---|---|---|---|
| 37 | **Ảnh sản phẩm (ProductImage)** | ✅ Có | Catalog data, gắn Product |
| 38 | **Lô / Batch** | ✅ Có (opt-in) | Nhóm units cùng đợt SX/HSD |
| 39 | **BatchStock** (junction Batch × Branch) | Implicit (theo Lô bật) | Tồn lô tại từng CN |
| 40 | **StockUnit / ProductInstance** | ❌ Chưa có | Đơn vị vật lý có serial — **module mới cần build** |

---

## 6. CƠ HỘI MODULE MỚI: KiotViet Serial Tracking

Sau phân tích này, đề xuất bổ sung vào danh sách brainstorm trước:

**Module #N+1: Serial / IMEI Tracking** (mở rộng module hiện có)
- **Pain:** Seller điện máy, đồng hồ, mỹ phẩm chính hãng, IT đang phải dùng Excel + KiotViet riêng
- **Solution:** Toggle tương tự "Quản lý theo Lô" — bật "Quản lý theo Serial":
  - Khi nhập (PN): scan/nhập từng IMEI vào phiếu nhập
  - Khi bán (HD): scan IMEI khi xuất, in lên hóa đơn để bảo hành
  - Khi trả (TH): scan IMEI để verify đúng máy đã bán
  - View "All Serial of Product X" — biết unit nào còn, đã bán cho ai, ngày nào
  - Bảo hành tracking — auto cảnh báo khi unit gần hết hạn BH
- **Effort:** Trung bình (3-5 tháng) — phức tạp nhất là UX scan flow + đối soát công nợ bảo hành
- **Impact:** High — unlock 1 segment market đáng kể (điện máy, IT, hàng hiệu, mỹ phẩm chính hãng)
- **Bundle với:** Lô tracking sẵn có để upsell tier "Pro" hoặc "Enterprise"

→ Đây là module #N+1 mà mình sẽ thêm vào file brainstorm trước, gắn với theme "Tracking nâng cao" (đã list ở phân tích hàng hóa-tồn kho).
