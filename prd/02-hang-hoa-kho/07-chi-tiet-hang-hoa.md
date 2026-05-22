@# Chi tiết Hàng hóa — Deep Dive

**Bối cảnh:** Màn hình Chi tiết Hàng hóa (Product Detail) là **trung tâm quản lý toàn bộ vòng đời của một SKU** — từ thông tin danh mục, định giá, biến thể, tồn kho từng chi nhánh, lịch sử giao dịch đến kết nối kênh bán online. Đây là màn hình được truy cập nhiều nhất bởi quản lý hàng hóa và thủ kho.

**Entities:** E01 (Product), E02 (UoM), E03 (Variant), E06 (PriceBook), E08 (ProductImage), E09 (Stock), E10 (StockCard), E11 (StockNorm), E12 (StockoutForecast), E36 (ChannelMapping)
**Liên quan:** [README.md](./README.md) | [01-tong-quan.md](./01-tong-quan.md) | [02-the-kho.md](./02-the-kho.md)

---

## 1. Vị trí trong hệ sinh thái

### 1.1 Điểm truy cập

```
Danh sách Hàng hóa (/man/#/Products)
    │
    ├── Click tên SP → Drawer/Panel Chi tiết (slide in từ phải)
    ├── Click mã chứng từ trên Thẻ kho → Quick view
    ├── Click SP trên Phiếu nhập / Hóa đơn / Đặt hàng → Quick view
    └── URL trực tiếp: /products/{id}

POS màn hình bán hàng
    └── Click SP trong giỏ → Mini product detail (read-only)
```

### 1.2 Vai trò trung tâm

```
                    ┌──────────────────────────────────┐
                    │      CHI TIẾT HÀNG HÓA           │
                    │         (E01)                    │
                    └──────┬────────────────┬──────────┘
                           │                │
          ┌────────────────┼────────────────┼────────────────┐
          ▼                ▼                ▼                ▼
    [Thẻ kho]         [Tồn kho]       [Bảng giá]     [Kênh bán]
      (E10)             (E09)           (E06)           (E36)
   audit trail       per branch     multi pricebook   marketplace
          │                │
          ▼                ▼
    [Phiếu nguồn]   [Định mức tồn]
   HD/PN/TRF/KK       (E11 Min/Max)
```

---

## 2. Layout tổng thể

### 2.1 Cấu trúc trang

```
┌─────────────────────────────────────────────────────────────────────┐
│ [← Danh sách]  Giay Adidas Ultraboost 22 — SP001234       [Sửa]    │
│                                            [In tem] [...]            │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  [Ảnh chính]   Tên: Giay Adidas Ultraboost 22               │   │
│  │  (multi-img)   Mã: SP001234  |  Mã vạch: 8936123456789      │   │
│  │                Nhóm: Giay the thao / Adidas                  │   │
│  │                Giá bán: 2.350.000  |  Giá vốn: 1.620.000    │   │
│  │                Trạng thái: ● Đang kinh doanh                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  [Thông tin] [Mô tả, ghi chú] [Thẻ kho] [Tồn kho] [Liên kết kênh] │
│  ─────────────────────────────────────────────────────────────────  │
│  {Nội dung tab hiện tại}                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Action bar

| Action | Điều kiện | Mô tả |
|---|---|---|
| **Sửa** | Có quyền edit | Chuyển sang edit mode toàn trang |
| **In tem mã** | Always | In barcode label, chọn số lượng |
| **Sao chép** | Always | Clone SP mới với prefix "Copy of" |
| **Ngừng kinh doanh** | Đang kinh doanh | Toggle status per branch hoặc tất cả |
| **Xóa** | Tồn kho = 0, không có giao dịch | Soft delete |
| **Xuất file** | Always | Export thông tin SP ra Excel |

> **Business rule BR-01:** Không được xóa SP khi `onHand > 0` hoặc tồn tại bất kỳ phiếu HD/PN/TH/TRF nào liên quan còn ở trạng thái Đang xử lý.

---

## 3. Tab Thông tin — Schema đầy đủ

### 3.1 Thông tin cơ bản

| Trường | Kiểu | Ghi chú | Sửa được? |
|---|---|---|---|
| Mã hàng | string | Auto SP######, cho phép custom | Có (1 lần) |
| Tên hàng | string | Tên hiển thị, tìm kiếm full-text | Có |
| Tên tiếng Anh | string | Optional, dùng cho kênh quốc tế | Có |
| Ảnh sản phẩm | E08[] | Multi-image, ảnh chính + gallery | Có |
| Mã vạch | string | Barcode / QR code, scan hoặc tự tạo | Có |
| Nhóm hàng | FK → E04 | Hierarchy phân loại | Có |
| Thương hiệu | FK → E05 | Optional | Có |
| Vị trí kho | string | Free text, vd "Kệ A3-Tầng 2", phục vụ picker | Có |
| Trọng lượng | decimal | Gram hoặc kg, phục vụ tính phí ship | Có |
| Ghi chú | text | Nội bộ, không hiển thị khách | Có |

### 3.2 Loại sản phẩm (Product Type — immutable sau khi tạo)

| Loại | Đặc điểm | Use case |
|---|---|---|
| **Hàng hóa thường** | 1 SKU, track tồn | Quần áo, thực phẩm, đồ gia dụng |
| **Hàng hóa có biến thể** | 1 SP cha + N SP con | Giày (size, màu), áo (size, màu) |
| **Dịch vụ** | Không track tồn | Sửa chữa, giặt là, cắt tóc |
| **Combo / Set** | Gói N SP con | Combo bữa sáng, gift set |

> **Business rule BR-02:** Loại sản phẩm không được thay đổi sau khi đã có giao dịch. Lý do: thay đổi type ảnh hưởng tới cách tính tồn, giá vốn và báo cáo.

### 3.3 Định giá

```
Giá bán
├── Giá trước thuế
├── Giá sau thuế (hiển thị mặc định)
├── VAT bán: 0% / 5% / 8% / 10%  — phải khớp loại hàng trên HĐĐT
└── Giá vốn (Cost)
    ├── Tự động cập nhật theo WAC khi nhập hàng (E15)
    └── Cho phép nhập tay (khi chưa có phiếu nhập)

Bảng giá (E06) — Section riêng
├── Bảng giá chung: {giá bán hiện tại}
├── Bảng giá sỉ: {giá}
├── Bảng giá VIP: {giá}
└── [+ Thêm bảng giá]
```

> **Business rule BR-03:** Thay đổi giá bán không ảnh hưởng các hóa đơn đã tạo — giá chốt trên phiếu là immutable.

### 3.4 Đơn vị tính (UoM — E02)

Mỗi SP có thể có **1..N đơn vị**, mỗi đơn vị có giá riêng và hệ số quy đổi về đơn vị cơ bản:

```
Đơn vị cơ bản: cái (qty_base = 1)
├── hộp (12 cái, giá: 216.000 → tương đương 18.000/cái)
└── thùng (144 cái, giá: 2.160.000 → tương đương 15.000/cái)
```

UI: Bảng inline với cột: Tên đơn vị | Hệ số quy đổi | Giá bán | Giá vốn | Barcode riêng

> **Business rule BR-04:** Thay đổi hệ số quy đổi đơn vị sau khi đã có giao dịch cần cảnh báo — có thể làm inconsistent dữ liệu thẻ kho cũ.

### 3.5 Biến thể (Variant — E03)

Chỉ hiển thị khi Product Type = "Hàng hóa có biến thể":

```
SP cha: Giày Nike Air Max 270 (SP000010)
    │
    ├── Thuộc tính: SIZE (37, 38, 39, 40, 41, 42)
    │              MÀU  (Đen, Trắng, Xanh)
    │
    └── Biến thể (N = 6 size × 3 màu = 18 SKU con):
        ├── SP000011 — Size 37, Màu Đen  — Giá: 3.100.000 — Tồn: 5
        ├── SP000012 — Size 37, Màu Trắng — Giá: 3.100.000 — Tồn: 0
        ├── SP000013 — Size 38, Màu Đen  — Giá: 3.100.000 — Tồn: 8
        └── ...
```

Bảng biến thể trên Product Detail:
- Cột: Thuộc tính (Size/Màu) | Mã riêng | Barcode | Giá | Tồn CN hiện tại | Trạng thái
- Có thể sửa giá, barcode, trạng thái per variant
- Click vào variant → mở Product Detail của variant con

### 3.6 Cài đặt theo Chi nhánh

Section hiển thị **matrix SP × CN** — mỗi CN có cấu hình riêng:

| Chi nhánh | Trạng thái KD | Định mức Min | Định mức Max | Vị trí kho |
|---|---|---|---|---|
| CN Quận 1 | Đang KD | 5 | 50 | Kệ A2 |
| CN Trung tâm | Đang KD | 3 | 30 | Kệ B1 |
| CN Gò Vấp | Ngừng KD | — | — | — |

> **Business rule BR-05:** Trạng thái "Ngừng kinh doanh" tại CN X khiến SP không hiển thị trên POS tại CN X, nhưng vẫn có thể nhập/xuất qua phiếu kho.

---

## 4. Tab Tồn kho

### 4.1 Bảng tồn theo Chi nhánh

```
┌────────────────────────────────────────────────────────────────────┐
│ Chi nhánh     │ Tồn thực │ KH đặt │ Đặt NCC │ Khả dụng │ Dự kiến  │
│───────────────┼──────────┼────────┼─────────┼──────────┼──────────│
│ CN Quận 1     │    15    │   3    │    0    │    12    │  18 ngày │
│ CN Trung tâm  │     8    │   1    │    5    │     7    │  22 ngày │
│ CN Gò Vấp     │     0    │   0    │    0    │     0    │  ---     │
│───────────────┼──────────┼────────┼─────────┼──────────┼──────────│
│ TỔNG          │    23    │   4    │    5    │    19    │          │
└────────────────────────────────────────────────────────────────────┘

  Tồn thực = onHand (vật lý)
  Khả dụng = onHand − KH đặt  (số POS có thể bán)
  Dự kiến  = onHand / tốc độ bán bình quân (ngày)
```

### 4.2 Tồn theo Lô/Serial (khi bật tracking)

Khi SP có `batchTracking = true` hoặc `serialTracking = true`:

```
Tab con: Lô hàng
  ├── Lô 2024-12-01 | HSD: 31/12/2025 | Tồn: 10 | CN Quận 1
  └── Lô 2025-01-15 | HSD: 15/01/2026 | Tồn: 5  | CN Trung tâm

Tab con: Serial
  ├── SN0001234 | Trạng thái: Có hàng | CN Quận 1 | Nhập: 01/12/2024
  └── SN0001235 | Trạng thái: Đã bán  | HD016341  | Bán: 09/12/2025
```

> Xem chi tiết: [06-lo-va-serial.md](./06-lo-va-serial.md)

---

## 5. Tab Thẻ kho

Tab này nhúng toàn bộ StockCard (E10) của SP này tại CN đang xem.

**Filter nhanh:**
- Tất cả CN | CN hiện tại
- Bộ lọc thời gian: Hôm nay / 7 ngày / 30 ngày / Tùy chỉnh
- Loại GD: Bán hàng / Nhập hàng / Trả hàng / Chuyển hàng / Kiểm kho...

> Xem chi tiết: [02-the-kho.md](./02-the-kho.md)

---

## 6. Tab Mô tả, ghi chú

| Trường | Kiểu | Mục đích |
|---|---|---|
| Mô tả sản phẩm | Rich text (HTML) | Hiển thị trên kênh online, website |
| Ghi chú nội bộ | Plain text | Nhân viên đọc, không hiển thị khách |
| Tags / Nhãn | string[] | Tìm kiếm nâng cao, gom nhóm campaign |

---

## 7. Tab Liên kết kênh bán (E36)

```
┌─────────────────────────────────────────────────────────────────┐
│ Kênh        │ Gian hàng          │ SKU kênh       │ Trạng thái │
│─────────────┼────────────────────┼────────────────┼────────────│
│ 🛍 Shopee   │ Shop Giày ABC       │ SHOPEE_SP001234│ ✓ Đang bán │
│ 🎵 TikTok   │ TikTok Shop Giày   │ TT_SP001234    │ ✓ Đang bán │
│ 🛒 Lazada   │ Lazada Official    │ LAZ_001234     │ ✗ Ngừng    │
│─────────────┼────────────────────┼────────────────┼────────────│
│ [+ Liên kết kênh mới]                                           │
└─────────────────────────────────────────────────────────────────┘

Tính năng per mapping:
├── Đồng bộ tồn kho (2 chiều)
├── Đẩy ảnh / tên / mô tả lên kênh
├── Giá kênh riêng (có thể khác giá nội bộ)
└── Lịch marketing tự động (E37)
```

---

## 8. State machine — Vòng đời trạng thái hàng hóa

```
        [Tạo mới]
             │
             ▼
    ┌── Nháp (Phiếu tạm) ──┐
    │   chưa xác nhận       │
    │                       │[Xác nhận]
    └───────────►            ▼
                    Đang kinh doanh ◄──────────── Mặc định sau tạo
                          │  ▲
                [Ngừng]   │  │ [Kinh doanh lại]
                          ▼  │
                    Ngừng kinh doanh
                   (per Branch hoặc toàn bộ)
                          │
                [Xóa — khi đủ điều kiện BR-01]
                          ▼
                       Đã xóa (soft delete)
```

**Trạng thái per Branch** — 1 SP có thể:
- `Đang kinh doanh` tại CN A (hiển thị POS, bán được)
- `Ngừng kinh doanh` tại CN B (ẩn trên POS, vẫn xuất nhập qua phiếu kho)
- `Đã xóa` khi tất cả CN đều ngừng và không còn tồn/giao dịch

---

## 9. Business Rules

| ID | Rule | Ghi chú |
|---|---|---|
| BR-01 | Không xóa SP khi `onHand > 0` hoặc còn phiếu đang xử lý | Prevent orphan stock records |
| BR-02 | Loại SP (type) immutable sau khi có giao dịch đầu tiên | Thay type → vỡ kế toán |
| BR-03 | Giá bán thay đổi không ảnh hưởng HĐ đã tạo | Giá trên phiếu là final |
| BR-04 | Hệ số quy đổi UoM: cảnh báo khi sửa nếu đã có giao dịch | Inconsistent thẻ kho cũ |
| BR-05 | Ngừng KD per CN: ẩn POS, vẫn cho nhập/xuất qua phiếu | Operational flexibility |
| BR-06 | Mã hàng: unique per merchant, cho phép sửa 1 lần trước khi có giao dịch | Sau khi có GD → immutable (mã tham chiếu trên phiếu) |
| BR-07 | Combo: giá combo ≤ tổng giá lẻ các thành phần (cảnh báo nếu không) | Prevent margin âm do setup nhầm |
| BR-08 | VAT bán/nhập phải nằm trong {0, 5, 8, 10}% — theo TT78/2021 | Compliance HĐĐT |
| BR-09 | SP dịch vụ: không tạo dòng thẻ kho, không có định mức tồn | Vô hình với inventory engine |
| BR-10 | Giá vốn SP biến thể con độc lập với SP cha | WAC tính riêng per variant |

---

## 10. Luồng UX — Tạo mới sản phẩm (Happy Path)

```
1. Danh sách SP → [+ Thêm hàng hóa]
2. Chọn loại SP (Thường / Biến thể / Dịch vụ / Combo)
3. Nhập: Tên, Nhóm hàng, Thương hiệu, Mã vạch
4. Nhập giá bán, giá vốn, VAT
5. Upload ảnh
6. (Nếu có biến thể) Chọn thuộc tính → generate tổ hợp
7. Cài đặt per branch (Min/Max, vị trí kho)
8. [Lưu] → Tạo E01 + E09 (tồn = 0 per CN) + E11 (nếu có Min/Max)
9. Sau đó: Tạo Phiếu nhập (PN) để nhập hàng ban đầu
```

**Luồng nhanh (quick add từ POS / Phiếu nhập):**
- Điền Tên + Giá bán + Nhóm hàng → [Lưu nhanh]
- Bổ sung thông tin chi tiết sau

---

## 11. Điểm đau & cơ hội cải tiến

### 11.1 Quản lý thông tin SP

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có AI generate mô tả sản phẩm — seller phải tự viết từ đầu cho 5,510 SP | AI Product Copywriter: Upload ảnh → AI viết tên, mô tả, bullet points, từ khóa SEO theo ngành |
| 2 | Không tự động gợi ý nhóm hàng khi nhập tên — phải chọn tay | AI auto-categorize: "Giày Nike" → suggest "Giày thể thao / Nike" với confidence score |
| 3 | Hình ảnh nhỏ, không có crop/resize trực tiếp — seller phải edit ảnh ngoài rồi upload | Built-in image editor: crop, resize, background removal (AI) |
| 4 | Mã vạch chỉ 1 barcode per SP — không hỗ trợ SP có nhiều mã EAN khác nhau | Multi-barcode per SKU (VD: mã NCC vs mã nội bộ vs mã kênh) |
| 5 | Không có lịch sử thay đổi thông tin SP (ai sửa giá khi nào) | Change log per field — đặc biệt cho giá bán, giá vốn, nhóm hàng |

### 11.2 Định giá

| # | Pain | Đề xuất |
|---|---|---|
| 6 | Bảng giá không có rule tự động — phải gán thủ công per SP | Rule engine: "Khách hạng B + kênh Shopee → áp bảng giá sỉ" |
| 7 | Không có pricing history — không biết giá tháng trước là bao nhiêu | Price history timeline: giá bán theo thời gian, kèm sự kiện (sale, tăng giá) |
| 8 | Không có so sánh giá đối thủ — seller không biết mình đang cao/thấp thị trường | Competitor price monitor: scrape từ Shopee/Lazada, hiển thị so sánh |
| 9 | Thay giá hàng loạt chỉ qua import Excel — không có bulk edit trên UI | Bulk price edit: filter nhóm → chọn nhiều → sửa giá / áp % discount |
| 10 | Không có margin calculator realtime: seller nhập giá bán không thấy ngay margin % | Inline margin display: "Giá bán 350k − Giá vốn 220k = Margin 37%" |

### 11.3 Biến thể & UoM

| # | Pain | Đề xuất |
|---|---|---|
| 11 | Tạo biến thể thủ công — 6 size × 3 màu = 18 dòng phải nhập tay | Matrix generator: chọn SIZE {37–42} × MÀU {Đen, Trắng, Xanh} → auto-generate 18 variants |
| 12 | Không có bulk edit giá biến thể — 18 variant phải sửa giá từng cái | Bulk edit biến thể: fill đồng loạt hoặc theo pattern |
| 13 | Đơn vị tính phải cấu hình tay per SP — không có template UoM phổ biến | UoM template per ngành: F&B (chai/lốc/thùng), dược (viên/vỉ/hộp)... |
| 14 | Không hiển thị tồn theo đơn vị trong Product Detail — chỉ thấy số cơ bản | Hiển thị "23 cái = 1 hộp + 11 cái dư" |

### 11.4 Tồn kho & dự báo

| # | Pain | Đề xuất |
|---|---|---|
| 15 | Min/Max phải nhập tay per (SP × CN) — 5,510 SP × 5 CN = 27,550 dòng | AI auto-suggest: phân tích velocity 90 ngày → gợi ý Min/Max + 1-click apply |
| 16 | "Dự kiến hết hàng" là trung bình thô — không xét trend, mùa vụ, sự kiện | ML forecast: tích hợp seasonal + event calendar (sale 11/11, Tết...) |
| 17 | Không có "safety stock" riêng biệt — Min đang kiêm luôn reorder point | 3 ngưỡng: Safety / Reorder Point / Max — hiển thị visual gauge |
| 18 | Tab Tồn kho không có quick action — phải thoát ra tạo phiếu chuyển/nhập | Quick actions: [Đặt hàng NCC] [Chuyển hàng] [Điều chỉnh tồn] ngay trong tab |

### 11.5 UX tổng thể

| # | Pain | Đề xuất |
|---|---|---|
| 19 | Không có "related products" — không biết SP này thường được bán cùng gì | "Thường mua cùng" từ HĐ thực tế (collaborative filtering) |
| 20 | Không có performance summary trên Product Detail — phải qua báo cáo riêng | Mini KPI panel: Doanh thu 30 ngày / Số lần bán / Tỷ lệ trả hàng / Xu hướng |
| 21 | Ảnh sản phẩm chỉ lưu 1 set — kênh Shopee/TikTok thường yêu cầu ảnh riêng theo chuẩn | Per-channel image set: ảnh nội bộ / ảnh Shopee (1:1) / ảnh TikTok (9:16) |
| 22 | Không có QR code cho vị trí kho (location) — picker vẫn dò thủ công | Location QR: scan vị trí kho → list SP tại đó |

---

## 12. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **AI Product Studio** — Upload ảnh → auto generate tên/mô tả/bullets/SEO, auto-categorize, suggest giá theo thị trường | Pain #1, #2 — tiết kiệm 30–60 phút/SP × 5,510 SP | Trung | Rất cao |
| 2 | **Variant Matrix Generator** — chọn thuộc tính → auto-generate + bulk edit price/barcode | Pain #11, #12 — SP thời trang có 18–50 variants | Thấp | Cao |
| 3 | **Smart Pricing Engine** — Rule-based + competitor monitor + margin calculator realtime | Pain #6, #8, #10 — seller không biết margin đang ở đâu | Cao | Rất cao |
| 4 | **AI Min/Max Optimizer** — phân tích toàn bộ velocity, lead time NCC, suggest Min/Max per (SP × CN) | Pain #15 — 27,550 cấu hình không thể làm tay | Trung | Rất cao |
| 5 | **Product Performance Dashboard** — ngay trong Product Detail: doanh thu, margin, trả hàng, thị phần kênh | Pain #20 — phải ra báo cáo riêng mới biết SP đang chạy thế nào | Thấp | Cao |

---

## 13. Tóm lược

**Chi tiết Hàng hóa KiotViet:**
- **Trung tâm thông tin** của 1 SKU — 5 tabs bao phủ thông tin danh mục → định giá → thẻ kho → tồn multi-branch → kênh bán
- **Mạnh:** Multi-UoM per SKU, biến thể phân cấp, trạng thái kinh doanh per branch, liên kết trực tiếp thẻ kho, channel mapping inline
- **Yếu:** Thiếu change log giá, không có AI generate nội dung, không có margin calculator, Min/Max phải cấu hình tay, biến thể tạo thủ công
- **Business rules quan trọng:** Loại SP immutable sau GD đầu tiên, giá bán không retroactive, xóa SP cần tồn = 0

3 hướng cải tiến ưu tiên:
1. **AI-assisted content** — Auto generate mô tả, auto-categorize, smart pricing
2. **Bulk operations** — Variant matrix, bulk price edit, AI Min/Max optimizer
3. **Intelligence layer** — Performance dashboard ngay trong Product Detail, competitor price monitor, sales trend mini-chart
