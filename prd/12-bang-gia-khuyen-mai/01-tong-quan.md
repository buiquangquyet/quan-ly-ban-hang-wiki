# Tổng quan — Bảng giá & Khuyến mãi

**Phạm vi:** Module quản lý Bảng giá (multi-tier pricing) và Chương trình khuyến mãi (discount rules, coupon, voucher) — tầng định giá động nằm giữa Product catalog và giao dịch bán hàng.
**Liên quan:** Module 01 (POS — áp giá khi bán), Module 05 (Khách hàng — tier/nhóm KH), Module 06/07 (Đơn hàng/Hóa đơn — thực thi discount)

---

## 1. Vị trí trong hệ sinh thái

```
[Product] ──giá gốc──► [Bảng giá / KM] ──giá cuối──► [POS / Đơn hàng / Hóa đơn]
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        [Bảng giá VIP]  [KM flash sale]  [Coupon GIAM50]
        (per nhóm KH)   (theo thời gian)  (per mã code)
```

Bảng giá & Khuyến mãi là tầng **pricing engine** quyết định giá thực tế khách trả. Không có module này, POS chỉ bán theo giá gốc — không thể cạnh tranh.

---

## 2. Bảng giá (PriceBook — BG01)

### 2.1 Cấu trúc

Hệ thống hỗ trợ nhiều bảng giá song song. Mỗi bảng giá là một **mapping SP → giá** riêng biệt:

| Trường | Ghi chú |
|---|---|
| Tên bảng giá | VD: Bảng giá lẻ, Bảng giá sỉ, Bảng giá VIP, TikTok Shop |
| Loại | Mặc định / Theo nhóm KH / Theo kênh bán / Tùy chỉnh |
| Trạng thái | Đang áp dụng / Tạm dừng |
| Thời gian hiệu lực | Từ ngày — đến ngày (không bắt buộc) |
| Chi nhánh áp dụng | All hoặc chọn CN cụ thể |

Bảng giá mặc định: **"Bảng giá chung"** — luôn tồn tại, không thể xóa.

### 2.2 Giá trong bảng

Mỗi dòng trong bảng giá:

| Trường | Ghi chú |
|---|---|
| Hàng hóa (FK Product) | |
| Đơn vị tính | Cho phép giá khác nhau theo UoM |
| Giá bán | Giá áp dụng khi dùng bảng này |
| Giá tối thiểu | Sàn giá — NV không được bán dưới ngưỡng này |
| Giá tối đa | Trần giá (optional) |

### 2.3 Quy tắc áp bảng giá

**Thứ tự ưu tiên áp dụng (từ cao → thấp):**
1. Bảng giá gắn trực tiếp vào giao dịch (chọn tay)
2. Bảng giá của nhóm khách hàng (auto từ KH)
3. Bảng giá theo kênh bán (Shopee / TikTok tự động)
4. Bảng giá chung (mặc định)

**Conflict resolution:** Priority cao hơn thắng; không cộng dồn bảng giá.

---

## 3. Chương trình khuyến mãi (Promotion — KM01)

### 3.1 Phân loại

| Loại KM | Mô tả | Ví dụ |
|---|---|---|
| **Giảm theo %** | Giảm X% trên đơn hàng hoặc dòng SP | Giảm 10% cho đơn > 500k |
| **Giảm số tiền** | Giảm cố định X đồng | Giảm 50,000đ cho HĐ bất kỳ |
| **Mua X tặng Y** | Mua đủ số lượng X → tặng hàng Y | Mua 2 tặng 1 |
| **Combo** | Mua nhóm SP A+B+C với giá ưu đãi | Bộ combo 3 sản phẩm = 200k |
| **Tặng hàng** | HĐ đạt ngưỡng X → tặng SP Z | Đơn > 1tr tặng túi vải |
| **Giảm vận chuyển** | Giảm/miễn phí ship | Miễn ship cho đơn > 300k |
| **Tích điểm nhân đôi** | Nhân hệ số tích điểm theo sự kiện | Ngày sinh nhật: x3 điểm |

### 3.2 Điều kiện áp dụng

| Chiều | Điều kiện | Ví dụ |
|---|---|---|
| Đơn hàng | Giá trị HĐ tổng ≥ X | Đơn từ 500,000đ |
| Hàng hóa | Theo nhóm hàng hoặc SP cụ thể | Chỉ áp nhóm "Điện thoại" |
| Khách hàng | Theo nhóm KH, hạng thành viên | Chỉ KH Gold trở lên |
| Thời gian | Từ ngày–đến ngày, giờ vàng | Flash sale 10h–12h |
| Kênh bán | POS / Online / Shopee | Chỉ TikTok Shop |
| Chi nhánh | Một hoặc nhiều CN | |
| Số lần sử dụng | Giới hạn tổng lần dùng | Chỉ 100 lần đầu tiên |
| Số lần per KH | Giới hạn mỗi KH dùng N lần | Mỗi KH 1 lần duy nhất |

### 3.3 Cộng dồn khuyến mãi (Stacking)

**Cấu hình stacking per chương trình:**
- `stackable: false` — chương trình độc quyền, không kết hợp
- `stackable: true` — có thể kết hợp với KM khác
- `priority` — khi nhiều KM eligible, dùng KM có priority cao nhất

**Rule áp dụng mặc định:**
1. Chạy qua tất cả KM eligible
2. Lọc theo `stackable`
3. Apply theo thứ tự priority; nếu bằng nhau → apply KM có lợi hơn cho KH

---

## 4. Coupon & Voucher (BG02)

### 4.1 Phân biệt

| | Coupon | Voucher |
|---|---|---|
| Phát hành | Hệ thống generate mã | Bán/tặng cho KH cụ thể |
| Nhận dạng | Mã code (GIAM50, TETK2026) | Mã riêng per KH |
| Scope | Chung (ai có mã đều dùng được) | Gắn với KH cụ thể |
| Giá trị | % hoặc số tiền cố định | Số tiền cố định hoặc SP |
| Hết hạn | Có thể có ngày hết hạn | Thường có hạn dùng |

### 4.2 Coupon code

| Trường | Ghi chú |
|---|---|
| Mã code | Tự nhập (FLASH30) hoặc auto-generate |
| Loại giảm | % hoặc số tiền |
| Giá trị | |
| Điều kiện | Đơn tối thiểu, nhóm SP, nhóm KH |
| Thời gian | Từ–đến |
| Tổng lần dùng | Giới hạn tổng |
| Lần per KH | Giới hạn mỗi KH |
| Trạng thái | Đang chạy / Tạm dừng / Hết hạn |

### 4.3 Voucher

- Tạo hàng loạt voucher cho chiến dịch (VD: 500 voucher 50k tặng KH sinh nhật)
- Gắn với KH cụ thể trong hệ thống CRM
- Gửi tự động qua ZNS/SMS/Email
- Track: đã gửi / đã dùng / hết hạn

---

## 5. Tích điểm & Đổi điểm (BG03)

Tích hợp với module Khách hàng (C06 LoyaltyTransaction):

### 5.1 Quy tắc tích điểm

| Cấu hình | Ghi chú |
|---|---|
| Tỷ lệ tích | X điểm per 1,000đ mua hàng |
| Nhóm hàng tích điểm | SP có tag "Tích điểm" mới được tính |
| Áp dụng KM thì có tích không | Cấu hình toggle |
| Ngưỡng tích tối thiểu per HĐ | VD: HĐ từ 100k mới tích |

### 5.2 Quy tắc đổi điểm

| Cấu hình | Ghi chú |
|---|---|
| Tỷ lệ đổi | 100 điểm = 10,000đ giảm |
| Đổi tối đa per HĐ | % hoặc số điểm |
| Điểm tối thiểu để đổi | VD: phải có ít nhất 200 điểm |
| SP không được đổi điểm | Exclude SP khuyến mãi |

---

## 6. Entity Catalog (BG01–BG06)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| BG01 | PriceBook | Bảng giá | Có |
| BG02 | PriceBookLine | Dòng giá trong bảng | Có |
| BG03 | Promotion | Chương trình khuyến mãi | Có một phần |
| BG04 | PromotionCondition | Điều kiện áp dụng KM | Cần bổ sung |
| BG05 | Coupon | Mã giảm giá / voucher | Có một phần |
| BG06 | LoyaltyRule | Quy tắc tích/đổi điểm | Cần bổ sung |

---

## 7. Pain points hiện tại

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không có KM "mua X tặng Y" — phải xử lý tay | Cao |
| P2 | Không stack được nhiều KM — nhiều shop cần combo discount | Cao |
| P3 | Coupon code không giới hạn được per KH — dùng nhiều lần | Cao |
| P4 | Bảng giá không gắn được thời gian hiệu lực — phải bật/tắt tay | Trung bình |
| P5 | Không có flash sale tự động bật/tắt theo lịch | Trung bình |
| P6 | Giá tối thiểu không lock được — NV hay bán dưới giá vốn | Cao |
| P7 | Không export được danh sách coupon đã dùng để audit | Trung bình |
| P8 | Đổi điểm chỉ có 1 loại quy tắc — không flexible | Thấp |

---

## 8. Cơ hội cải tiến (Top 5)

### I1. Promotion Engine đầy đủ
- Rule builder kéo thả: điều kiện AND/OR, nhiều loại action
- Tự động bật/tắt theo lịch
- Preview "KH sẽ được giảm bao nhiêu" trước khi publish

### I2. Smart Stacking với conflict detection
- Tự động phát hiện 2 KM conflict nhau khi tạo
- UI cho chủ shop cấu hình thứ tự ưu tiên

### I3. Flash Sale Scheduler
- Đặt lịch KM giờ vàng (10h–12h thứ 6)
- Tự động thông báo KH qua ZNS trước 30 phút
- Countdown timer trên POS/online store

### I4. Voucher campaign tự động
- Trigger: KH sinh nhật → auto gửi voucher 100k
- Trigger: KH không mua 30 ngày → gửi coupon win-back
- Liên kết với Marketing Automation (brainstorm D2)

### I5. Giá sàn enforcement nghiêm
- Lock cứng: POS không cho bán dưới giá sàn kể cả khi NV sửa tay
- Alert realtime khi NV cố bán dưới giá sàn → ghi log audit
