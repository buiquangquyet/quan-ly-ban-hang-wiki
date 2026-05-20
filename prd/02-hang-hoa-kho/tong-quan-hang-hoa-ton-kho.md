# PHÂN TÍCH HÀNG HÓA & TỒN KHO — KIOTVIET

**Phạm vi:** Module `/man/#/Products` và toàn bộ `Kho hàng` (Chuyển hàng, Kiểm kho, Sản xuất, Xuất dùng nội bộ, Xuất hủy, Bảng giá)
**Test data:** 5,510 hàng hóa, 2 chi nhánh (CN 1, CN trung tâm)

---

## 1. SƠ ĐỒ TỔ CHỨC

```
Hàng hóa (Product)
├── Thông tin chính (mã, tên, ảnh, mã vạch, thương hiệu, trọng lượng, vị trí)
├── Loại sản phẩm (Hàng hóa thường / Dịch vụ / Combo / Có biến thể)
├── Phân nhóm
│   ├── Nhóm hàng (Category)
│   ├── Thuộc tính (Size / Kiểu / ABC / Mở rộng)
│   └── Thương hiệu (Brand)
├── Giá
│   ├── Giá vốn (Cost)
│   ├── Giá bán trước/sau thuế + VAT bán
│   ├── VAT nhập
│   └── Bảng giá (nhiều bảng — sỉ, lẻ, VIP, theo kênh...)
├── Đơn vị tính (multi: cái, hộp, thùng — mỗi đơn vị có giá riêng)
├── Tag hành vi (Bán trực tiếp / Tích điểm / Liên kết kênh bán)
└── Cài đặt theo Chi nhánh
    ├── Tồn kho (Stock theo CN)
    ├── Định mức tồn (Min/Max theo CN)
    ├── Trạng thái kinh doanh (Đang kinh doanh / Ngừng theo CN)
    └── Dự kiến hết hàng

Tồn kho (Inventory)
├── Tồn kho hiện tại (Số lượng khả dụng)
├── Đặt NCC (Đã đặt từ NCC, chưa về)
├── KH đặt (Khách đã đặt, chưa xuất)
├── Dự kiến hết hàng (số ngày)
└── Thẻ kho (Stock card — lịch sử mọi biến động)

Nghiệp vụ tác động tồn:
├── Nhập kho (Mua hàng)         → tăng tồn
├── Bán hàng (Hóa đơn)          → giảm tồn
├── Trả hàng                    → tăng tồn (về lại)
├── Chuyển hàng                 → CN gửi giảm, CN nhận tăng
├── Kiểm kho                    → cân bằng (lệch tăng/lệch giảm)
├── Sản xuất                    → giảm NVL, tăng thành phẩm
├── Xuất dùng nội bộ            → giảm tồn (không xuất hóa đơn)
└── Xuất hủy                    → giảm tồn (hàng hỏng, hết hạn)
```

---

## 2. CÁC ĐỐI TƯỢNG TRÊN MODULE HÀNG HÓA & TỒN KHO

### NHÓM A — Đối tượng "Hàng hóa" (Product entities)

#### A1. Hàng hóa (Product) — Đối tượng trung tâm
**Tab thông tin có 5 sub-tab:** Thông tin | Mô tả, ghi chú | Thẻ kho | Tồn kho | Liên kết kênh bán

| Trường | Ghi chú |
|---|---|
| Mã hàng | Auto-generated SP000XXX, cho phép custom |
| Tên hàng | Hiển thị, đa ngôn ngữ implicit qua test data |
| Ảnh sản phẩm | Multi image |
| Mã vạch | Barcode, optional |
| Nhóm hàng | FK → Nhóm hàng (A4) |
| Thương hiệu | FK → Thương hiệu (A5) |
| Vị trí | Vị trí cất trong kho (free text), giúp picker tìm |
| Trọng lượng | Phục vụ tính phí ship |
| Giá vốn | Cost — auto từ giá nhập / nhập tay |
| Giá bán trước/sau thuế | Phải khớp với VAT |
| VAT bán/nhập (%) | 0/5/8/10% — match với HĐĐT |
| Định mức tồn | Min - Max, dùng cho cảnh báo tồn |
| Tag loại | Hàng hóa thường / Dịch vụ / Combo |
| Tag tích điểm | Có tính điểm loyalty không |
| Tag bán trực tiếp | Có hiển thị trên màn POS không |

#### A2. Đơn vị tính (Unit of Measure)
- **Đặc trưng quan trọng:** 1 sản phẩm có thể có **nhiều đơn vị**, mỗi đơn vị có giá riêng
- VD: "áo sm" có 2 đơn vị: cái (650,000), hộp (216,450)
- Quan hệ quy đổi: 1 hộp = ? cái (factor)
- Hiển thị: dropdown `- cái ▼` cạnh tên SP, hover hiện danh sách

#### A3. Thuộc tính / Biến thể (Attribute / Variant)
- Pre-defined attributes: **SIZE**, **KIỂU**, **ABC** + Mở rộng (custom)
- Sản phẩm cha + N sản phẩm con (mỗi tổ hợp size×màu = 1 variant với mã riêng)
- Vd test data: "bộ hơi 4100QB-A/YN4100QB" có nhiều variant theo Lỗ nổ nhọn, Chuanlu, XM dầu, khí, gờ...

#### A4. Nhóm hàng (Category / Product Group)
- Hierarchy (cha-con), nested
- Filter chính trên list
- Vd: Quần áo, Mỹ phẩm, Đồ ăn

#### A5. Thương hiệu (Brand)
- Master data riêng, optional
- Filter và phân loại

#### A6. Bảng giá (Price Book / Price List)
- Mặc định: "Bảng giá chung"
- Có thể tạo nhiều bảng (sỉ, lẻ, VIP, theo kênh TikTok/Shopee/Lazada, theo khách)
- Mỗi SP có giá khác nhau theo từng bảng
- Filter: theo điều kiện và so sánh

#### A7. Nhà cung cấp (Supplier)
- Master data NCC
- Liên kết với hàng hóa (1 SP có thể có nhiều NCC, ưu tiên)
- Dùng cho Mua hàng và "Đặt NCC"

---

### NHÓM B — Đối tượng "Tồn kho" (Inventory entities)

#### B1. Tồn kho theo Chi nhánh (Stock per Branch) ⭐ Quan trọng
- **Đặc trưng:** Tồn kho KHÔNG phải số đơn lẻ — là **ma trận SP × Chi nhánh**
- Mỗi cặp (SP, CN) có:
  - Tồn kho hiện tại
  - Đặt NCC (số lượng đã đặt từ NCC, chưa nhập)
  - KH đặt (số lượng khách đã đặt, chưa giao)
  - Định mức tồn (min/max) — có thể khác nhau giữa các CN
  - Dự kiến hết hàng (số ngày dựa trên tốc độ bán)
  - Trạng thái kinh doanh (Đang kinh doanh / Ngừng kinh doanh) **per branch**

#### B2. Thẻ kho (Stock Card / Inventory Ledger)
- Lịch sử mọi biến động tồn của 1 SP
- **Columns:** Chứng từ | Thời gian | Loại giao dịch | Đối tác | Giá GD | Giá vốn | Số lượng | Tồn cuối
- Là audit trail bất biến — quan trọng cho kế toán

#### B3. Định mức tồn (Stock Norm / Reorder Point)
- Min — ngưỡng cảnh báo "sắp hết"
- Max — ngưỡng cảnh báo "tồn quá"
- Có thể cấu hình per branch

#### B4. Dự kiến hết hàng (Stockout Forecast)
- Tính tự động: Tồn kho / Tốc độ bán bình quân
- Hiển thị dạng "X ngày" hoặc "---" (không bán)
- Filter trên list (Toàn thời gian / Tùy chỉnh)

---

### NHÓM C — Đối tượng "Nghiệp vụ kho" (Inventory Operations)

Mỗi nghiệp vụ là 1 loại **phiếu (Voucher/Document)** với cấu trúc tương tự: Header (mã, ngày, CN, người tạo, trạng thái, ghi chú) + Lines (SP, SL, giá, vị trí).

> **Deep dive cho 3 nghiệp vụ chính:**
> - Nhập hàng → [`nhap-hang-deep-dive.md`](./nhap-hang-deep-dive.md)
> - Trả hàng (bán + nhập) → [`tra-hang-deep-dive.md`](./tra-hang-deep-dive.md)
> - Bán hàng (góc tồn kho) → [`ban-hang-deep-dive.md`](./ban-hang-deep-dive.md)

#### C1. Phiếu Chuyển hàng (Stock Transfer)
- **Vai trò:** Chuyển hàng giữa các Chi nhánh
- **Trạng thái:** Phiếu tạm → Đang chuyển → Đã nhận
- **Filter:** Chuyển đi (CN gửi), Nhận về (CN nhận), Khớp/Không khớp (SL thực tế nhận vs SL gửi)
- **Tác động tồn:** CN gửi giảm khi xuất, CN nhận tăng khi nhận

#### C2. Phiếu Kiểm kho (Stock Take / Stock Count)
- **Vai trò:** Đối chiếu tồn thực tế vs sổ sách → cân bằng
- **Trạng thái:** Phiếu tạm → Đã cân bằng kho → Đã hủy
- **Columns đặc trưng:** SL thực tế | Tổng thực tế | **Tổng chênh lệch** | SL lệch tăng | SL lệch giảm
- **Tác động:** Khi cân bằng → sinh bút toán điều chỉnh tồn

#### C3. Phiếu Sản xuất (Manufacturing / Assembly)
- **Vai trò:** Tạo thành phẩm từ NVL theo công thức (BOM — Bill of Materials)
- **Trạng thái:** Phiếu tạm → Hoàn thành → Đã hủy
- **Tác động:** Giảm tồn NVL, tăng tồn thành phẩm
- **Use case:** F&B (pha chế combo), gia công, lắp ráp

#### C4. Phiếu Xuất dùng nội bộ (Internal Use) — **Module "Mới"**
- **Vai trò:** Cấp hàng cho nhân viên/phòng ban dùng (đồng phục, dụng cụ, mẫu test), không qua bán hàng
- **Trạng thái:** Phiếu tạm → Hoàn thành → Đã hủy
- **Trường đặc trưng:** Loại xuất, Người xuất dùng nội bộ, Người nhận
- **Tác động:** Giảm tồn nhưng không tạo doanh thu

#### C5. Phiếu Xuất hủy (Stock Write-off / Damage)
- **Vai trò:** Loại bỏ hàng hỏng, hết hạn, hư hao
- **Tác động:** Giảm tồn, ghi vào chi phí

#### C6. Phiếu Mua hàng / Nhập kho (Purchase Order — module riêng "Mua hàng")
- **Tác động:** Tăng tồn theo NCC, cập nhật giá vốn (FIFO/Bình quân)
- Liên thông với "Đặt NCC" trên A1

---

### NHÓM D — Đối tượng tích hợp (Integration)

#### D1. Liên kết kênh bán (Channel Mapping)
- Tab riêng trong product detail
- Map 1 SP nội bộ ↔ N listing trên Shopee/TikTok Shop/Lazada/Tiki
- Cho phép: đồng bộ tồn 2 chiều, đẩy ảnh/giá, đổi giá theo kênh

#### D2. Marketing tự động đẩy hàng hóa
- Lịch tự động đẩy listing lên top trên marketplace
- Setup theo từng gian hàng (đã thấy ở phân tích phần Bán online)

---

## 3. SƠ ĐỒ DÒNG DỮ LIỆU TỒN KHO

```
[NCC] ──Mua hàng──► [Tồn CN_X] ──Bán hàng──► [Khách]
                       │  ▲
                       │  └─Trả hàng─── [Khách]
                       │
                       ├─Chuyển hàng───► [Tồn CN_Y]
                       ├─Kiểm kho─── (điều chỉnh ±)
                       ├─Sản xuất───► [Thành phẩm trong CN_X]
                       ├─Xuất dùng nội bộ ─► [Nhân viên]
                       └─Xuất hủy───► [Bỏ]
```

Tất cả nghiệp vụ đều ghi vào **Thẻ kho (B2)** — 1 audit trail duy nhất.

---

## 4. ĐIỂM MẠNH

1. **Mô hình tồn kho ma trận SP × CN đầy đủ** — đáp ứng được chuỗi 3-20 CN
2. **Multi-unit per SKU** — quan trọng cho ngành nước/đồ uống (chai, lốc, thùng), thuốc (viên, vỉ, hộp)
3. **Bảng giá multi** — đáp ứng được sỉ-lẻ, đa kênh, VIP
4. **Định mức tồn (min/max) cấu hình per branch** — flexibility cao
5. **Đầy đủ 6 nghiệp vụ kho** (Mua/Bán/Trả/Chuyển/Kiểm/Hủy) + 2 bổ trợ (Sản xuất, Xuất nội bộ)
6. **Thẻ kho audit trail** — không thiếu nghiệp vụ
7. **Trạng thái kinh doanh per CN** — 1 SP có thể "Ngừng kinh doanh" tại CN A nhưng "Đang kinh doanh" tại CN B
8. **Liên kết kênh bán** ngay trong tab product — UX khá tiện cho omni-channel
9. **Filter "Dự kiến hết hàng"** — proactive operations

---

## 5. ĐIỂM YẾU & CƠ HỘI CẢI TIẾN

### 5.1 Tồn kho

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Định mức tồn (min/max) phải nhập tay cho từng SP × CN. Với shop 5,510 SP × 5 CN = 27,550 cấu hình → không khả thi | **AI auto-suggest Min/Max** dựa trên velocity bán + lead time NCC |
| 2 | "Dự kiến hết hàng" tính trung bình, không xét seasonality | Forecast ML có xét sự kiện, ngày lễ, sale campaign |
| 3 | Không có **gợi ý điều chuyển hàng tự động** khi CN A thừa CN B thiếu | "Suggested transfer" widget — 1-click duyệt |
| 4 | Không hiển thị "**tồn an toàn**" (Safety Stock) tách bạch với Min | Khái niệm 3 ngưỡng: Safety / Reorder / Max |
| 5 | Không có **lot/batch tracking** (số lô, hạn sử dụng) — pain với ngành thực phẩm/dược | Module Batch & Expiry (LIFO, FEFO) |
| 6 | Không có **serial number tracking** — pain với điện máy, IT, mỹ phẩm chính hãng | Module Serial Tracking |
| 7 | Không thấy **multi-location trong cùng 1 CN** (kho A1, A2, kệ X...) — trường "Vị trí" chỉ là text | Hierarchical location (Warehouse → Zone → Bin) |

### 5.2 Hàng hóa

| # | Pain | Đề xuất |
|---|---|---|
| 8 | **5,510 SP** test data — search còn chậm. List dài, phân trang truyền thống | Virtualized scroll, search semantic ("kem dưỡng" → auto fuzzy match) |
| 9 | Không có **auto-categorization** khi tạo SP mới | AI suggest nhóm hàng từ tên + ảnh |
| 10 | Không có **AI generate product description** | Photo → AI tự viết mô tả, bullet, từ khóa SEO |
| 11 | Pricing chỉ là số cứng — không có **dynamic pricing** theo competitor | Module Competitive Pricing (B3 trong brainstorm trước) |
| 12 | "Bảng giá" không có **logic tự động áp** theo điều kiện (khách hạng VIP → bảng VIP) | Rule engine: nếu khách hạng X, kênh Y, thời gian Z → áp bảng N |
| 13 | Không có **A/B test giá** | Test 2 giá trên 2 CN tương đương → đo elasticity |
| 14 | Mã hàng auto SP000XXX, không có **pattern theo nhóm** (vd MP-001, FB-001) | Cấu hình prefix theo nhóm hàng |
| 15 | Mô tả SP dạng rich text, không có **multi-language tự động** | Auto translate sang EN/Khmer cho seller bán xuyên biên giới |

### 5.3 Nghiệp vụ

| # | Pain | Đề xuất |
|---|---|---|
| 16 | Kiểm kho **bắt buộc làm toàn bộ kho** — không thể kiểm "rolling" theo nhóm hàng hằng tuần | Cycle counting workflow |
| 17 | Chuyển hàng không có **gợi ý route tối ưu** (CN nào gửi gần nhất + có hàng đủ) | Smart routing |
| 18 | Xuất hủy không **track lý do chuẩn hóa** | Dropdown: Hỏng / Hết hạn / Rơi vỡ / Trộm cắp / Khác → analytics |
| 19 | Sản xuất chỉ là phiếu — không có **BOM nested** (NVL → bán thành phẩm → thành phẩm) | Multi-level BOM cho gia công 2-3 bước |
| 20 | Không có **vendor performance scoring** (NCC này on-time delivery %, fill rate %, defect %) | Supplier scorecard |

---

## 6. CÁC ĐỐI TƯỢNG MỚI BỔ SUNG VÀO DATA MODEL

Cập nhật danh sách 24 đối tượng (trong file "Cac-Doi-Tuong-Man-Ban-Hang.md") với 9 đối tượng mới gắn với Hàng hóa/Tồn kho:

| # | Đối tượng | Vai trò |
|---|---|---|
| 25 | **Đơn vị tính (UoM)** | Multi-unit per SKU, có quy đổi |
| 26 | **Thuộc tính / Biến thể** (Attribute/Variant) | SIZE/KIỂU/Custom, sinh ra SKU con |
| 27 | **Nhóm hàng** (Category) | Phân loại có cha-con |
| 28 | **Thương hiệu** (Brand) | Master |
| 29 | **Bảng giá** (Price Book) | Multi price list |
| 30 | **Nhà cung cấp** (Supplier) | Master + Vendor performance |
| 31 | **Định mức tồn** (Stock Norm) | Min/Max per (SP × CN) |
| 32 | **Thẻ kho** (Stock Card / Ledger) | Audit trail bất biến |
| 33 | **Phiếu** vận hành kho (Inventory Voucher) | Abstract — có 6 loại con: Chuyển, Kiểm, Sản xuất, Xuất nội bộ, Xuất hủy, Nhập |

---

## 7. CƠ HỘI BUILD MODULE/SẢN PHẨM MỚI (Top 5 từ phân tích này)

| # | Sản phẩm | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Inventory AI Copilot** — Auto Min/Max + Smart Transfer + Forecast | 5,510 SP × N CN không cấu hình tay được | Cao | Rất cao |
| 2 | **Batch & Expiry tracking** | Pain # 1 của F&B, dược, mỹ phẩm | Trung | Cao |
| 3 | **Multi-location warehouse** (Zone/Bin) | Chuỗi có kho lớn | Trung | Trung |
| 4 | **AI Product Studio** (auto tạo mô tả/ảnh/SEO từ 1 ảnh upload) | Tiết kiệm 100-300k/SKU | Trung | Cao |
| 5 | **Dynamic Pricing Rule Engine** | Áp bảng giá tự động theo điều kiện | Trung | Trung |

(Đã tổng hợp đầy đủ trong file Brainstorm-Modules-Moi.md trước đó)

---

## 8. TÓM LƯỢC

KiotViet đã có nền tảng Hàng hóa & Tồn kho **rất đầy đủ** cho POS truyền thống: multi-branch, multi-unit, multi-price-book, đủ 6 nghiệp vụ kho, audit trail thông qua thẻ kho.

3 hướng nâng cấp lớn:
1. **Từ thủ công sang AI-assisted** — auto định mức, smart transfer, forecast, auto categorization
2. **Tracking nâng cao** — batch/expiry, serial, multi-location trong CN
3. **Pricing thông minh** — dynamic, rule-based, competitor-aware

Đây cũng chính là những hướng đột phá để KiotViet bứt phá khỏi mass market POS sang **enterprise/chuỗi** và **seller chuyên nghiệp đa kênh** — match đúng segment bạn quan tâm.
