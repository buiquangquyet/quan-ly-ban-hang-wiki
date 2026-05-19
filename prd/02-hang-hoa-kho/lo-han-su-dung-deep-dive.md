# ĐÀO SÂU: HÀNG HÓA LÔ - HẠN SỬ DỤNG (BATCH & EXPIRY)

**Nguồn:** Quan sát testzone17 + tài liệu chính thức tại kiotviet.vn/huong-dan-su-dung-kiotviet
**Phạm vi áp dụng:** Tạp hóa-Siêu thị, Mỹ phẩm, Mẹ & Bé, Nhà thuốc, Nông sản & Thực phẩm

---

## 1. CẬP NHẬT QUAN TRỌNG VỀ KIOTVIET

**KiotViet thực ra có 2 chế độ tracking nâng cao** (riêng biệt, có thể bật song song):

| Chế độ | Đối tượng | Ngành áp dụng |
|---|---|---|
| **Lô + Hạn sử dụng** | Nhóm units cùng đợt SX/HSD | Thực phẩm, dược, mỹ phẩm |
| **Serial / IMEI** | Từng unit cá biệt | Điện thoại, điện máy, xe |

→ Phân tích trước của mình về "KiotViet không có Serial" là **sai một phần** — có, nhưng chưa enable trong test account 17. Mình đã verify qua tài liệu chính thức (file riêng phía sau).

---

## 2. LÔ + HẠN SỬ DỤNG — CHI TIẾT VẬN HÀNH

### 2.1 Bật tính năng (account-level)

**Đường dẫn:** Thiết lập → Hàng hóa → Section "Giá vốn, tồn kho" → Toggle **"Quản lý tồn kho theo Lô, hạn sử dụng"**

Mô tả setting:
> "Theo dõi các hàng hóa như thực phẩm hoặc thuốc theo Lô, hạn sử dụng trong mọi giao dịch."

Mặc định: **OFF**. Sau khi bật → áp dụng cho toàn tài khoản, nhưng từng SP vẫn opt-in riêng.

### 2.2 Bật cho từng SP (product-level)

Khi toggle account-level đang ON, form Tạo/Sửa SP xuất hiện thêm **dropdown "Quản lý theo lô, hạn sử dụng"** trong section "Tồn kho", với 2 giá trị: **Có / Không**.

**Behavior change quan trọng (đã verify):**

| Trước khi bật Lô (Không) | Sau khi bật Lô (Có) |
|---|---|
| Field "Tồn kho" hiển thị, nhập tay được | Field "Tồn kho" **bị ẩn** |
| Tồn = 1 số đơn lẻ | Tồn = tổng các Lô |
| Min/Max chỉ là số | Min/Max vẫn ở SP level |

→ **Insight data model:** Khi bật Lô, tồn kho **không còn là attribute của SP** mà là attribute của **từng Lô**. SP chỉ là parent, tổng tồn = SUM(Lô.qty còn).

### 2.3 Workflow đầy đủ

```
┌─────────────────────────────────────────────────────────────┐
│  TẠO SP với "Quản lý theo lô = Có"                          │
│  → KHÔNG nhập tồn ban đầu                                   │
├─────────────────────────────────────────────────────────────┤
│  NHẬP HÀNG (PN)                                             │
│  Khi thêm SP có Lô vào phiếu nhập:                          │
│  → Nút "Thêm mới lô, hạn sử dụng"                           │
│  → Form nhập: [Số lô] [Hạn sử dụng] [Số lượng]             │
│  → 1 phiếu PN có thể chứa NHIỀU lô của cùng 1 SP           │
│  → Hoàn thành PN → tăng tồn theo từng lô                    │
├─────────────────────────────────────────────────────────────┤
│  BÁN HÀNG (HD)                                              │
│  Khi thêm SP có Lô vào hóa đơn:                             │
│  → Hệ thống auto-suggest **FEFO** (First Expired First Out)│
│  → User có thể chọn lô khác từ dropdown                     │
│  → Cảnh báo màu sắc:                                        │
│      • VÀNG: lô sắp hết hạn                                 │
│      • ĐỎ: lô đã hết hạn                                    │
│  → Giảm tồn của lô được chọn                                │
├─────────────────────────────────────────────────────────────┤
│  TRẢ HÀNG (TH)                                              │
│  → Khi khách trả → chọn ĐÚNG lô đã bán → tăng tồn lô đó     │
├─────────────────────────────────────────────────────────────┤
│  CHUYỂN HÀNG (TRF)                                          │
│  → Auto suggest FEFO khi chuyển đi                          │
│  → CN nhận: hàng vẫn giữ thông tin Lô gốc                   │
├─────────────────────────────────────────────────────────────┤
│  KIỂM KHO (KK)                                              │
│  → Phải nhập SL thực tế CHO TỪNG LÔ                         │
│  → Không thể kiểm tổng — chi tiết tới từng lô               │
├─────────────────────────────────────────────────────────────┤
│  XUẤT HỦY (XH)                                              │
│  → Chọn chính xác lô cần hủy                                │
│  → Use case chính: hủy lô đã hết hạn                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.4 Báo cáo HSD

**Vị trí:** Phân tích → Hàng hóa → Mối quan tâm: **Hạn sử dụng**

Báo cáo liệt kê:
- Các lô **sắp hết hạn** (chưa hết)
- Các lô **đã hết hạn**

→ Dùng để lên kế hoạch khuyến mại, xả hàng, hoặc chủ động hủy lô.

---

## 3. DATA MODEL CỦA LÔ

### 3.1 Entities phát sinh khi bật Lô

```
Product (Hàng hóa, 1)
   │
   │ Flag: trackByBatch = true
   │
   │ 1:N
   ▼
Batch (Lô, N)
├── id (PK)
├── productId (FK → Product)
├── batchCode (string) — số lô do user nhập
├── manufactureDate (nullable)
├── expireDate (date) — hạn sử dụng
├── createdAt
└── createdViaDocumentCode (PN... — phiếu nhập tạo lô)

BatchStock (junction — tồn lô tại từng CN)
├── id (PK)
├── batchId (FK → Batch)
├── branchId (FK → Branch)
├── quantity — tồn của lô này tại CN này
└── lastMovementAt
```

### 3.2 Thẻ kho khi bật Lô — thêm cột Batch

Mỗi dòng Thẻ kho khi liên quan tới SP có Lô bật → phải tham chiếu `batchId`:

```
StockCardLine
├── ...các trường cũ...
├── productId
├── branchId
└── batchId (nullable — chỉ có nếu SP track theo Lô)
```

Hệ quả: 1 phiếu nhập (PN) với 1 SP nhập 3 lô khác nhau → sinh **3 dòng Thẻ kho** (mỗi lô 1 dòng), không phải 1 dòng tổng.

### 3.3 Tồn kho hiện tại — Aggregate query

```
Tồn của SP X tại CN Y = SUM(BatchStock.quantity 
                            WHERE Batch.productId = X 
                              AND BatchStock.branchId = Y)
```

Khi UI hiển thị "Tồn kho: 150" cho SP có Lô → là tổng. Click vào sẽ thấy breakdown:
- Lô L001 (HSD 30/06/2026): 50
- Lô L002 (HSD 15/08/2026): 70
- Lô L003 (HSD 20/10/2026): 30
- Tổng: 150

### 3.4 Logic FEFO trong code (giả định)

```pseudo
function suggestBatchToSell(productId, branchId, requestedQty):
    batches = BatchStock
        .where(productId, branchId)
        .where(quantity > 0)
        .orderBy('expireDate', ASC)  // hết hạn trước → xuất trước
    
    chosen_batches = []
    remaining = requestedQty
    for batch in batches:
        if remaining <= 0: break
        take = min(batch.quantity, remaining)
        chosen_batches.append({batch, take})
        remaining -= take
    
    if remaining > 0:
        warn "Không đủ tồn"
    
    return chosen_batches
```

**Override:** User có thể click dropdown để chọn lô khác → bypass FEFO (vd khách chỉ muốn hàng mới sản xuất, không muốn lô cũ).

---

## 4. UI/UX QUAN SÁT

### 4.1 Cảnh báo trực quan trên màn POS

| Trạng thái lô | Màu sắc | Hành động đề xuất |
|---|---|---|
| Bình thường (còn lâu mới hết hạn) | Mặc định | Bán bình thường |
| Sắp hết hạn | **VÀNG** | Ưu tiên xuất, có thể giảm giá |
| Đã hết hạn | **ĐỎ** | Không nên bán — cảnh báo cashier |

### 4.2 Hỗ trợ đa platform

| Platform | Workflow đầy đủ |
|---|---|
| **Web (Quản lý + Bán hàng)** | Đầy đủ tất cả nghiệp vụ |
| **Mobile app (KiotViet)** | Thêm SP, Nhập, Bán |
| **Máy POS Android** | Pop-up chọn lô khi thêm SP vào HĐ, FEFO suggestion |

---

## 5. ĐIỂM MẠNH

1. **Track tới level Lô** — đủ cho 90% use case ngành thực phẩm/dược
2. **FEFO tự động** — giảm thất thoát do hết hạn (pain # 1 ngành)
3. **Cảnh báo màu sắc realtime** — cashier không cần nhớ lô nào sắp hết
4. **Opt-in per SP** — không ép buộc tất cả SP phải có lô (giữ UX nhanh cho SP thường)
5. **Báo cáo HSD có sẵn** — proactive planning
6. **Hỗ trợ đa platform** — web/mobile/POS đều có
7. **Tích hợp đầy đủ nghiệp vụ** — trả, chuyển, kiểm, hủy đều respect lô

---

## 6. ĐIỂM YẾU & CƠ HỘI

### 6.1 Tính năng còn thiếu (so với best practice ngành)

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Chỉ track HSD — không có **ngày sản xuất** dùng để tính BB (Best Before) khác với HSD | Thêm field manufactureDate + bestBeforeDate riêng |
| 2 | Không có **Số lô NCC** (lot của nhà sản xuất) tách bạch với mã lô nội bộ | Thêm field supplierBatchCode |
| 3 | Không có **Country of Origin / Country of Manufacturing** per lô | Field origin (truy xuất nguồn gốc) |
| 4 | Không có **Recall workflow** — nếu NCC thông báo lỗi lô, không có cách bulk recall | Module Lot Recall |
| 5 | FEFO cứng — không cho **rule customizable** (vd "VIP customer luôn lô mới nhất") | Pluggable allocation strategy |
| 6 | Cảnh báo HSD chỉ trên màn POS — **không có push notification** cho chủ shop | Notification daily/weekly digest |
| 7 | Không có **auto markdown pricing** khi lô sắp hết hạn (giảm 20% khi còn 7 ngày) | Auto promotion rule by expiry |
| 8 | Cảnh báo bằng màu — không có **threshold cấu hình** (mặc định bao nhiêu ngày là "sắp"?) | Cấu hình ngày cảnh báo per SP/per category |
| 9 | Không thấy có **HSD per UoM** — nếu SP có 2 đơn vị (chai/lốc), HSD trên mỗi đơn vị giống nhau | Cần verify thêm |
| 10 | **Truy xuất ngược** từ lô → bán cho ai — chưa thấy có | Customer batch tracking (quan trọng cho food safety) |

### 6.2 Cơ hội module mới gắn với Lô

**M1. Smart Recall Module**
- Pain: NCC thông báo lỗi lô X → shop phải tìm thủ công ai mua phải lô đó
- Solution: 1-click recall — system list tất cả KH đã mua lô X → gửi SMS/Zalo tự động
- Impact cao cho dược, thực phẩm — compliance feature

**M2. Auto Markdown Engine**
- Pain: Lô sắp hết hạn → phải giảm giá tay
- Solution: Rule "Khi còn ≤ 7 ngày → giảm 20%, ≤ 3 ngày → giảm 50%" → auto áp ngay khi cashier scan
- Reduce wastage 30-50%

**M3. Expiry Heatmap Dashboard**
- View toàn bộ lô của shop trên 1 heatmap theo thời gian
- Trục X: tháng tới
- Trục Y: nhóm hàng / chi nhánh
- Cell color: tổng giá trị lô đáo hạn trong khoảng thời gian
- → Chủ shop nhìn 1 lần biết "tháng tới đáo hạn 50 triệu hàng tươi tại CN A"

**M4. Compliance Audit Pack**
- Cho ngành dược, thực phẩm có thanh tra nhà nước
- Export 1 file đầy đủ: lô vào, lô ra, ai bán, ai mua, hủy lúc nào — chuẩn TT200, GMP, HACCP
- Đính kèm hóa đơn điện tử

---

## 7. CẬP NHẬT DANH SÁCH ĐỐI TƯỢNG

Bổ sung 3 đối tượng mới vào data model:

| # | Đối tượng | Vai trò |
|---|---|---|
| 41 | **Lô (Batch)** | Nhóm units cùng đợt SX/HSD — entity cấp giữa Product và physical unit |
| 42 | **BatchStock** (junction) | Tồn của 1 lô tại 1 CN — many-to-many resolver |
| 43 | **batchId trong StockCardLine** | Cột mới trên Thẻ kho khi SP track theo Lô |

(Đối tượng 38 "Lô / Batch" đã đề cập trước — file này chi tiết hóa)

---

## 8. SO SÁNH NGẮN: LÔ vs SERIAL

| Tiêu chí | Lô (Batch) | Serial / IMEI |
|---|---|---|
| Đặc điểm | Nhóm N units cùng SX/HSD | 1 unit cá biệt |
| Cardinality SP → child | 1 SP → N Lô (mỗi Lô có M units) | 1 SP → N Serial (mỗi cái 1 record) |
| Use case | Thực phẩm, dược, mỹ phẩm | Điện thoại, điện máy, xe, đồng hồ |
| Định danh | Số lô + HSD | Mã Serial/IMEI unique toàn cầu |
| Logic xuất ưu tiên | **FEFO** (hết hạn trước xuất trước) | Bất kỳ — user chọn cụ thể |
| Cảnh báo | Sắp hết hạn / Đã hết hạn (màu vàng/đỏ) | (Không liên quan hạn — quan tâm bảo hành) |
| Bảo hành | Theo lô | **Theo từng serial cá nhân** |
| Chống tráo hàng | Yếu (nhiều unit chung lô) | **Mạnh** (1-1 mapping) |
| Volume nhập | Hàng trăm/SP/lô | Quét từng cái khi nhập |
| Có thể bật song song? | Có thể (1 SP đặc biệt có cả 2) | Hiếm, nhưng tài liệu KiotViet không cấm |

---

## 9. TÓM LƯỢC

**Lô + HSD trong KiotViet là tính năng đã hoàn thiện ở mức 7-8/10:**
- Đủ workflow cho nghiệp vụ thường gặp
- FEFO + cảnh báo màu là điểm sáng UX
- Tích hợp toàn diện 6 nghiệp vụ kho + báo cáo

**Còn thiếu (cơ hội cải tiến/bán thêm):**
- Truy xuất ngược lô → khách (food safety)
- Smart recall (compliance dược/thực phẩm)
- Auto markdown engine (giảm wastage)
- Heatmap đáo hạn (proactive planning)
- Threshold cảnh báo cấu hình (hiện cứng)

**Vai trò trong data model:**
- Lô là **entity trung gian** giữa Product và physical unit
- Khi bật Lô cho 1 SP, Tồn kho không còn là attribute của Product mà là **SUM của BatchStock** qua các Lô
- Thẻ kho thêm cột `batchId` để audit trail chính xác tới lô

**Đặt vào danh sách brainstorm:** 4 module mới đề xuất (Smart Recall, Auto Markdown, Heatmap, Compliance Audit Pack) — gắn với theme "Tracking nâng cao" và "Compliance" — match với hướng đột phá vào segment nhà thuốc và F&B mid-market.
