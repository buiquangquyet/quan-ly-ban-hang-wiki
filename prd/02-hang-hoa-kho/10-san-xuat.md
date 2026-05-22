# Deep dive — Phiếu Sản xuất & BOM (Manufacturing)

**Phạm vi:** Nghiệp vụ Sản xuất (SX######) — tạo thành phẩm từ nguyên vật liệu (NVL) theo công thức định mức (BOM — Bill of Materials). Phù hợp cho F&B pha chế, gia công, lắp ráp đơn giản.
**Entity:** E29 ManufacturingVoucher
**Prefix mã:** SX + 6 chữ số (VD: SX000012)
**Liên quan:** `01-tong-quan.md §C3`, `03-nhap-hang.md` (NVL nhập về), `02-the-kho.md` (thẻ kho ghi nhận)

---

## 1. Bài toán Sản xuất trong POS SMB

Đây KHÔNG phải module MRP/ERP chuyên nghiệp. Phạm vi là **sản xuất đơn giản** phổ biến ở:

| Ngành | Ví dụ sản xuất |
|---|---|
| F&B | Pha chế đồ uống từ nguyên liệu; làm bánh từ bột, trứng, đường |
| Thời trang | Gia công: nhận vải + phụ liệu → áo thành phẩm |
| Điện tử | Lắp ráp: mua linh kiện riêng → lắp thành bộ |
| Làm đẹp | Pha trộn: mua nguyên liệu → đóng gói sản phẩm nhãn riêng |

---

## 2. BOM — Bill of Materials (Công thức sản xuất)

BOM là **công thức** định nghĩa: tạo ra 1 đơn vị thành phẩm cần bao nhiêu NVL.

### 2.1 Cấu trúc BOM

```
Thành phẩm: Cà phê sữa đá (1 ly)
├── NVL 1: Cà phê rang xay           20g
├── NVL 2: Sữa đặc có đường          30ml
├── NVL 3: Đá viên                   150g
└── NVL 4: Cốc + ống hút (hao phí)   1 bộ
```

| Trường BOM | Ghi chú |
|---|---|
| Thành phẩm (FK Product) | SP đầu ra |
| Số lượng thành phẩm tạo ra | 1 lần sản xuất tạo bao nhiêu (VD: 1 ly, 10 bánh) |
| NVL (FK Product, là NVL) | Nguyên liệu đầu vào |
| Số lượng NVL per thành phẩm | Định mức tiêu hao |
| Đơn vị tính NVL | |
| Hao phí (%) | Tỷ lệ hao hụt thực tế (VD: bột hao 5%) |

### 2.2 Multi-level BOM (tương lai)

Hiện tại: BOM phẳng 1 cấp (NVL → thành phẩm).
Tương lai: BOM nhiều cấp (bán thành phẩm → thành phẩm).

---

## 3. Schema Phiếu Sản xuất (E29)

### Header

| Trường | Bắt buộc | Ghi chú |
|---|---|---|
| Mã phiếu | Có | SX######, auto |
| Chi nhánh | Có | Sản xuất tại CN nào |
| Ngày sản xuất | Có | |
| Người thực hiện | Có | NV sản xuất |
| Trạng thái | Có | Xem §4 |
| Ghi chú | Không | |

### Lines — Thành phẩm (Output)

| Trường | Ghi chú |
|---|---|
| Thành phẩm | SP đầu ra |
| BOM áp dụng | Công thức dùng |
| Số lượng sản xuất | Bao nhiêu đơn vị thành phẩm |
| Số lượng thực tế | Thực tế tạo ra được (có thể < kế hoạch do hỏng) |
| Lô thành phẩm | Gắn lô nếu thành phẩm quản lý lô |
| Ngày sản xuất / Hạn sử dụng | Tự động tính từ ngày SX + thời hạn bảo quản |

### Lines — NVL tiêu hao (Input)

Auto-tính từ BOM × số lượng sản xuất, nhưng cho phép override:

| Trường | Ghi chú |
|---|---|
| NVL | |
| SL định mức | = BOM quantity × SL sản xuất |
| SL thực tế tiêu hao | Cho phép nhập tay khi hao phí thực ≠ định mức |
| Chênh lệch tiêu hao | = Thực tế − Định mức — phân tích hiệu quả sản xuất |
| Lô NVL | Nếu NVL quản lý lô → chỉ định lô nào dùng (FEFO) |

---

## 4. State Machine

```
[Phiếu tạm] ──(xác nhận)──► [Đang sản xuất] ──(hoàn thành)──► [Hoàn thành]
     │                              │
     └── (hủy) ──► [Đã hủy]         └── (hủy) ──► [Đã hủy]
```

| Trạng thái | Tồn NVL | Tồn thành phẩm | Mô tả |
|---|---|---|---|
| Phiếu tạm | Không đổi | Không đổi | Chưa thực hiện |
| Đang sản xuất | Giảm theo SL định mức | Chưa tăng | NVL đã xuất vào sản xuất |
| Hoàn thành | Đã giảm (SL thực tế) | Tăng theo SL thực tế | Hoàn tất |
| Đã hủy | Hoàn tồn NVL | Không đổi | |

---

## 5. Tính giá vốn thành phẩm

Giá vốn thành phẩm = **tổng chi phí NVL đã tiêu hao** (theo giá vốn WAC của từng NVL tại thời điểm sản xuất):

```
Giá vốn thành phẩm (1 đơn vị)
= Σ (SL NVL tiêu hao × Giá vốn WAC của NVL) / SL thành phẩm thực tế
```

**Ví dụ:**
- Cà phê rang xay: 20g × 200đ/g = 4,000đ
- Sữa đặc: 30ml × 50đ/ml = 1,500đ
- Đá viên: 150g × 10đ/g = 1,500đ
- Cốc + ống hút: 1 bộ × 2,000đ = 2,000đ
→ Giá vốn 1 ly cà phê sữa đá = **9,000đ**

---

## 6. Luồng nghiệp vụ chi tiết

### Luồng chuẩn

```
Chủ shop tạo Phiếu SX: 50 ly cà phê sữa đá
→ Hệ thống auto-tính NVL cần: 1kg cà phê, 1.5L sữa đặc, 7.5kg đá...
→ Kiểm tra: NVL có đủ tồn không?
   - Đủ → cho phép xác nhận
   - Thiếu → cảnh báo, highlight dòng NVL thiếu
→ Xác nhận: tồn NVL giảm, trạng thái "Đang sản xuất"
→ NV sản xuất thực hiện, nhập SL thực tế
→ Hoàn thành: tồn thành phẩm tăng 50, tính giá vốn
→ Thẻ kho:
   - Cà phê rang xay: −1kg / loại giao dịch: SX
   - 1 ly cà phê sữa đá: +50 / loại giao dịch: SX
```

### Trường hợp NVL thiếu

- Nếu NVL chưa đủ → không chặn hard (vẫn cho tạo phiếu tạm)
- Cảnh báo rõ: "Thiếu 200g cà phê — cần nhập thêm hoặc giảm SL sản xuất"
- Tự động đề xuất: tạo Đặt hàng nhập NVL từ NCC

---

## 7. Sản phẩm Combo vs Sản phẩm Sản xuất

| | Combo | Sản xuất |
|---|---|---|
| Bản chất | Gói bán nhiều SP cùng nhau | Tạo ra SP mới từ NVL |
| Tồn kho | Không có tồn riêng của combo — trừ tồn từng SP con khi bán | Có tồn riêng của thành phẩm |
| Giá vốn | Tổng giá vốn các SP con | Tính lại theo công thức |
| Tạo khi nào | Cấu hình trong Product | Sản xuất trước khi bán |
| Use case | Bộ quà tặng, combo upsell | Pha chế, lắp ráp, gia công |

---

## 8. Pain points

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không nhắc khi NVL sắp hết — phát hiện khi đã thiếu giữa sản xuất | Cao |
| P2 | BOM chỉ 1 cấp — F&B chuyên nghiệp cần BOM multi-level | Trung bình |
| P3 | Chênh lệch tiêu hao NVL không có báo cáo riêng để cải thiện định mức | Trung bình |
| P4 | Không tính được chi phí lao động và overhead vào giá thành | Trung bình |
| P5 | Không có kế hoạch sản xuất (production planning) — chỉ phiếu thực hiện | Thấp |
| P6 | Sản xuất theo lô — không track được lô NVL nào đã vào lô thành phẩm nào | Trung bình |

---

## 9. Cơ hội cải tiến

### I1. Smart Reorder khi NVL sắp thiếu
- Tính toán ngược: với BOM hiện tại + kế hoạch bán → cần bao nhiêu NVL mỗi ngày
- Cảnh báo: "NVL cà phê còn đủ sản xuất 3 ngày — đặt hàng ngay"
- Tự tạo Đặt hàng nhập NVL 1 click

### I2. Traceability Lô đầu vào → đầu ra
- Track: lô NVL X đã vào sản xuất phiếu nào → tạo ra lô thành phẩm nào
- Khi có vấn đề (lô NVL bị lỗi) → tìm ngay các lô thành phẩm bị ảnh hưởng
- Liên kết với Smart Recall (brainstorm từ `06-lo-va-serial.md`)

### I3. Production Planning
- Lên kế hoạch sản xuất tuần / tháng
- Tự tính nhu cầu NVL tổng hợp → tạo Đặt hàng nhập gộp

### I4. Hiệu quả sản xuất
- Báo cáo: Định mức vs Thực tế tiêu hao theo từng SP, từng NV sản xuất
- Alert khi hao phí vượt ngưỡng X% → điều tra nguyên nhân
