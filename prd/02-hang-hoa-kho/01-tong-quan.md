# Tổng quan — Hàng hóa & Tồn kho

**Phạm vi:** Module `/man/#/Products` và toàn bộ Kho hàng (Chuyển hàng, Kiểm kho, Sản xuất, Xuất dùng nội bộ, Xuất hủy, Bảng giá)
**Test data:** 5,510 hàng hóa, 2 chi nhánh (CN 1, CN Trung tâm)
**Entity catalog:** [README.md](./README.md)

---

## 1. Sơ đồ tổ chức

```
Hàng hóa (Product — E01)
├── Thông tin chính (mã, tên, ảnh, mã vạch, thương hiệu, trọng lượng, vị trí)
├── Loại sản phẩm (Hàng hóa thường / Dịch vụ / Combo / Có biến thể)
├── Phân nhóm
│   ├── Nhóm hàng (E04)
│   ├── Thuộc tính / Biến thể (E03)
│   └── Thương hiệu (E05)
├── Giá
│   ├── Giá vốn (Cost)
│   ├── Giá bán trước/sau thuế + VAT bán
│   ├── VAT nhập
│   └── Bảng giá nhiều bảng (E06)
├── Đơn vị tính — multi (E02): cái, hộp, thùng — mỗi đơn vị có giá riêng
├── Ảnh sản phẩm (E08)
├── Tag hành vi (Bán trực tiếp / Tích điểm / Liên kết kênh bán)
└── Cài đặt theo Chi nhánh
    ├── Tồn kho (E09 — Stock per Branch)
    ├── Định mức tồn (E11 — Min/Max per Branch)
    ├── Trạng thái kinh doanh (Đang / Ngừng — per branch)
    └── Dự kiến hết hàng (E12)

Tồn kho (Inventory)
├── Tồn kho hiện tại (onHand)
├── Đặt NCC (đã đặt từ NCC, chưa về)
├── KH đặt (khách đã đặt, chưa xuất)
├── Dự kiến hết hàng (số ngày)
└── Thẻ kho (E10 — lịch sử mọi biến động)

Nghiệp vụ tác động tồn:
├── Nhập kho (PN)     → tăng tồn    [03-nhap-hang.md]
├── Bán hàng (HD)     → giảm tồn   [04-ban-hang.md]
├── Trả hàng (TH/TPN) → ±tồn       [05-tra-hang.md]
├── Chuyển hàng (TRF) → CN gửi −, CN nhận +
├── Kiểm kho (KK)     → cân bằng (±)
├── Sản xuất (SX)     → NVL −, thành phẩm +
├── Xuất nội bộ (XN)  → giảm tồn, không doanh thu
└── Xuất hủy (XH)     → giảm tồn, ghi chi phí
```

---

## 2. Các đối tượng — Nhóm A: Hàng hóa (Product Entities)

### A1. Hàng hóa (Product — E01)

Tab thông tin gồm 5 sub-tab: Thông tin | Mô tả, ghi chú | Thẻ kho | Tồn kho | Liên kết kênh bán

| Trường | Ghi chú |
|---|---|
| Mã hàng | Auto-generated SP000XXX, cho phép custom |
| Tên hàng | Tên hiển thị |
| Ảnh sản phẩm | Multi image (E08) |
| Mã vạch | Barcode, optional |
| Nhóm hàng | FK → E04 |
| Thương hiệu | FK → E05 |
| Vị trí | Vị trí cất trong kho (free text), giúp picker tìm hàng |
| Trọng lượng | Phục vụ tính phí ship |
| Giá vốn | Cost — auto từ giá nhập WAC hoặc nhập tay |
| Giá bán trước/sau thuế | Phải khớp với VAT |
| VAT bán/nhập (%) | 0 / 5 / 8 / 10% — match với HĐĐT |
| Định mức tồn | Min - Max, dùng cho cảnh báo tồn (E11) |
| Tag loại | Hàng hóa thường / Dịch vụ / Combo |
| Tag tích điểm | Có tính điểm loyalty không |
| Tag bán trực tiếp | Có hiển thị trên màn POS không |

### A2. Đơn vị tính (UoM — E02)

- 1 sản phẩm có thể có **nhiều đơn vị**, mỗi đơn vị có giá riêng
- VD: "áo sm" có 2 đơn vị: cái (650,000), hộp (216,450)
- Quan hệ quy đổi: 1 hộp = N cái (factor)
- UI: dropdown `- cái ▼` cạnh tên SP, hover hiện danh sách đơn vị

### A3. Thuộc tính / Biến thể (Variant / Attribute — E03)

- Pre-defined attributes: **SIZE**, **KIỂU**, **ABC** + Mở rộng (custom)
- Sản phẩm cha + N sản phẩm con (mỗi tổ hợp thuộc tính = 1 variant với mã riêng)
- VD: "bộ hơi 4100QB" có nhiều variant: Lỗ nổ nhọn, Chuanlu, XM dầu, XM khí, XM gờ...

### A4. Nhóm hàng (Category — E04)

- Hierarchy cha-con, nested
- Filter chính trên list sản phẩm
- VD: Quần áo → Nam → Áo sơ mi

### A5. Thương hiệu (Brand — E05)

- Master data riêng, optional
- Dùng để filter và phân loại

### A6. Bảng giá (Price Book — E06)

- Mặc định: "Bảng giá chung"
- Tạo nhiều bảng: sỉ, lẻ, VIP, theo kênh TikTok/Shopee/Lazada, theo khách
- Mỗi SP có giá khác nhau theo từng bảng
- Áp dụng: chọn bảng khi bán, tương lai cần rule engine tự động (xem §5.2 #12)

### A7. Nhà cung cấp (Supplier — E07)

- Master data NCC
- 1 SP có thể liên kết nhiều NCC, có thứ tự ưu tiên
- Dùng cho Mua hàng và theo dõi "Đặt NCC"

---

## 3. Các đối tượng — Nhóm B: Tồn kho (Inventory Entities)

### B1. Tồn kho theo Chi nhánh (Stock per Branch — E09)

Tồn kho KHÔNG phải số đơn lẻ — là **ma trận SP × Chi nhánh**.

Mỗi cặp (SP, CN) có:
- `onHand` — tồn vật lý hiện tại
- `reservedByCustomer` — KH đặt chưa xuất
- `reservedFromSupplier` — đã đặt NCC chưa về
- `available` = onHand − reservedByCustomer (số khả dụng POS)
- Định mức tồn min/max (E11) — cấu hình per branch
- Dự kiến hết hàng (E12) — tính từ velocity bán
- Trạng thái kinh doanh (Đang / Ngừng) **per branch** — 1 SP có thể ngừng tại CN A nhưng đang bán tại CN B

### B2. Thẻ kho (Stock Card — E10)

- Lịch sử mọi biến động tồn của 1 SP tại 1 CN
- Columns: Chứng từ | Thời gian | Loại GD | Đối tác | Giá GD | Giá vốn | Số lượng (có dấu) | Tồn cuối
- Audit trail bất biến — quan trọng cho kế toán
- Deep dive: [02-the-kho.md](./02-the-kho.md)

### B3. Định mức tồn (Stock Norm — E11)

- Min — ngưỡng cảnh báo "sắp hết hàng"
- Max — ngưỡng cảnh báo "tồn quá nhiều"
- Cấu hình per (SP × CN)

### B4. Dự kiến hết hàng (Stockout Forecast — E12)

- Tính tự động: `onHand / tốc_độ_bán_bình_quân`
- Hiển thị dạng "X ngày" hoặc "---" (không bán)
- Filter trên list: Toàn thời gian / Tùy chỉnh

---

## 4. Các đối tượng — Nhóm C: Nghiệp vụ kho (Inventory Operations)

Mỗi nghiệp vụ là 1 **phiếu (Voucher)** với cấu trúc: Header (mã, ngày, CN, người tạo, trạng thái, ghi chú) + Lines (SP, SL, giá, vị trí).

> Deep dive cho 5 nghiệp vụ chính:
> - Nhập hàng → [03-nhap-hang.md](./03-nhap-hang.md)
> - Bán hàng (góc tồn) → [04-ban-hang.md](./04-ban-hang.md)
> - Trả hàng → [05-tra-hang.md](./05-tra-hang.md)
> - Lô + Serial → [06-lo-va-serial.md](./06-lo-va-serial.md)

### C1. Phiếu Chuyển hàng (Stock Transfer — E27)

- **Vai trò:** Chuyển hàng giữa các chi nhánh
- **Trạng thái:** Phiếu tạm → Đang chuyển → Đã nhận
- **Tác động tồn:** CN gửi giảm khi xuất, CN nhận tăng khi nhận
- **Filter đặc trưng:** Chuyển đi / Nhận về / Khớp / Không khớp (SL thực tế vs SL gửi)

### C2. Phiếu Kiểm kho (Stock Take — E28)

- **Vai trò:** Đối chiếu tồn thực tế vs sổ sách → cân bằng
- **Trạng thái:** Phiếu tạm → Đã cân bằng kho → Đã hủy
- **Columns đặc trưng:** SL thực tế | Tổng thực tế | Tổng chênh lệch | SL lệch tăng | SL lệch giảm
- **Tác động:** Khi cân bằng → sinh bút toán điều chỉnh tồn

### C3. Phiếu Sản xuất (Manufacturing — E29)

- **Vai trò:** Tạo thành phẩm từ NVL theo công thức (BOM — Bill of Materials)
- **Trạng thái:** Phiếu tạm → Hoàn thành → Đã hủy
- **Tác động:** Giảm tồn NVL, tăng tồn thành phẩm
- **Use case:** F&B (pha chế combo), gia công, lắp ráp

### C4. Phiếu Xuất dùng nội bộ (Internal Use — E30)

- **Vai trò:** Cấp hàng cho nhân viên/phòng ban dùng (đồng phục, dụng cụ, mẫu test), không qua bán hàng
- **Trạng thái:** Phiếu tạm → Hoàn thành → Đã hủy
- **Trường đặc trưng:** Loại xuất, Người xuất dùng nội bộ, Người nhận
- **Tác động:** Giảm tồn nhưng không tạo doanh thu

### C5. Phiếu Xuất hủy (Stock Write-off — E31)

- **Vai trò:** Loại bỏ hàng hỏng, hết hạn, hư hao
- **Tác động:** Giảm tồn, ghi vào chi phí

---

## 5. Các đối tượng — Nhóm D: Tích hợp (Integration)

### D1. Liên kết kênh bán (Channel Mapping — E36)

- Tab riêng trong product detail
- Map 1 SP nội bộ ↔ N listing trên Shopee / TikTok Shop / Lazada / Tiki
- Cho phép: đồng bộ tồn 2 chiều, đẩy ảnh/giá, đổi giá theo kênh

### D2. Marketing tự động (AutoMarketing — E37)

- Lịch tự động đẩy listing lên top trên marketplace
- Setup theo từng gian hàng

---

## 6. Sơ đồ dòng dữ liệu tồn kho

```
[NCC] ──PN──► [Tồn CN_X] ──HD──► [Khách]
                  │  ▲
                  │  └──TH────── [Khách trả]
                  │
                  ├──TRF──────► [Tồn CN_Y]
                  ├──KK──────── (±điều chỉnh)
                  ├──SX──────► [Thành phẩm trong CN_X]
                  ├──XN──────► [Nhân viên]
                  └──XH──────► [Hủy bỏ]
              [Shop]──TPN──► [NCC]
```

Tất cả nghiệp vụ đều ghi vào **Thẻ kho (E10)** — single source of truth cho mọi audit.

---

## 7. Điểm mạnh

1. **Mô hình tồn kho ma trận SP × CN** — đáp ứng chuỗi 3–20 chi nhánh
2. **Multi-unit per SKU** — quan trọng cho ngành nước/đồ uống (chai, lốc, thùng), thuốc (viên, vỉ, hộp)
3. **Bảng giá multi** — đáp ứng sỉ-lẻ, đa kênh, VIP
4. **Định mức tồn (min/max) per branch** — flexibility cao
5. **6 nghiệp vụ kho** (Mua/Bán/Trả/Chuyển/Kiểm/Hủy) + 2 bổ trợ (Sản xuất, Xuất nội bộ)
6. **Thẻ kho audit trail** — không thiếu nghiệp vụ nào
7. **Trạng thái kinh doanh per branch** — 1 SP ngừng tại CN A, vẫn bán tại CN B
8. **Liên kết kênh bán** ngay trong product detail — UX tiện cho omni-channel
9. **Dự kiến hết hàng** — proactive operations

---

## 8. Điểm yếu & cơ hội cải tiến

### 8.1 Tồn kho

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Định mức tồn (min/max) nhập tay per (SP × CN). 5,510 SP × 5 CN = 27,550 cấu hình — không khả thi | AI auto-suggest Min/Max từ velocity bán + lead time NCC |
| 2 | "Dự kiến hết hàng" tính trung bình, không xét seasonality | Forecast ML có xét sự kiện, ngày lễ, sale campaign |
| 3 | Không có gợi ý điều chuyển hàng tự động khi CN A thừa, CN B thiếu | "Suggested transfer" widget — 1-click duyệt |
| 4 | Không hiển thị "tồn an toàn" (Safety Stock) tách bạch với Min | 3 ngưỡng: Safety / Reorder / Max |
| 5 | Không có lot/batch tracking (số lô, hạn sử dụng) — pain F&B/dược | Module Batch & Expiry (xem `06-lo-va-serial.md`) |
| 6 | Không có serial number tracking — pain điện máy, IT, mỹ phẩm chính hãng | Module Serial Tracking (xem `06-lo-va-serial.md`) |
| 7 | Multi-location trong 1 CN không có — "Vị trí" chỉ là text free | Hierarchical location: Warehouse → Zone → Bin |

### 8.2 Hàng hóa

| # | Pain | Đề xuất |
|---|---|---|
| 8 | 5,510 SP — search chậm, phân trang truyền thống | Virtualized scroll, semantic search ("kem dưỡng" → fuzzy match) |
| 9 | Không có auto-categorization khi tạo SP mới | AI suggest nhóm hàng từ tên + ảnh |
| 10 | Không có AI generate product description | Photo → AI tự viết mô tả, bullet, từ khóa SEO |
| 11 | Pricing là số cứng — không có dynamic pricing theo competitor | Module Competitive Pricing |
| 12 | Bảng giá không có logic tự động áp theo điều kiện | Rule engine: nếu khách hạng X, kênh Y, thời gian Z → áp bảng N |
| 13 | Mã hàng auto SP000XXX, không có pattern theo nhóm | Cấu hình prefix theo nhóm hàng (MP-001, FB-001) |

### 8.3 Nghiệp vụ kho

| # | Pain | Đề xuất |
|---|---|---|
| 14 | Kiểm kho bắt buộc toàn bộ kho — không thể kiểm rolling theo nhóm hằng tuần | Cycle counting workflow |
| 15 | Chuyển hàng không có gợi ý route tối ưu (CN nào gửi gần nhất + đủ hàng) | Smart routing |
| 16 | Xuất hủy không track lý do chuẩn hóa | Dropdown: Hỏng / Hết hạn / Rơi vỡ / Trộm cắp / Khác → analytics |
| 17 | Sản xuất chỉ là phiếu — không có BOM nested (NVL → bán thành phẩm → thành phẩm) | Multi-level BOM cho gia công 2–3 bước |
| 18 | Không có vendor performance scoring | Supplier scorecard (đã đề xuất trong `03-nhap-hang.md`) |

---

## 9. Cơ hội module mới — Top 5

| # | Module | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Inventory AI Copilot** — Auto Min/Max + Smart Transfer + Forecast | 5,510 SP × N CN không cấu hình tay được | Cao | Rất cao |
| 2 | **Batch & Expiry Tracking** nâng cao | Pain ngành F&B, dược, mỹ phẩm | Trung | Cao |
| 3 | **Serial / IMEI Tracking** | Điện máy, IT, đồng hồ, hàng hiệu | Trung | Cao |
| 4 | **Multi-location Warehouse** (Zone/Bin) | Chuỗi có kho lớn | Trung | Trung |
| 5 | **AI Product Studio** (auto mô tả/ảnh/SEO từ 1 ảnh upload) | Tiết kiệm 100–300k/SKU | Trung | Cao |

---

## 10. Tóm lược

KiotViet đã có nền tảng Hàng hóa & Tồn kho **rất đầy đủ** cho POS truyền thống: multi-branch, multi-unit, multi-price-book, đủ 8 nghiệp vụ kho, audit trail qua thẻ kho.

3 hướng nâng cấp lớn:
1. **Từ thủ công sang AI-assisted** — auto định mức, smart transfer, forecast, auto categorization
2. **Tracking nâng cao** — batch/expiry, serial, multi-location trong CN
3. **Pricing thông minh** — dynamic, rule-based, competitor-aware

Đây là hướng đột phá để bứt phá từ mass-market POS sang **enterprise/chuỗi** và **seller chuyên nghiệp đa kênh**.
