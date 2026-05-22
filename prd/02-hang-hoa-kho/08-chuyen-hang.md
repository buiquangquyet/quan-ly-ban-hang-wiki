# Deep dive — Phiếu Chuyển hàng (Stock Transfer)

**Phạm vi:** Nghiệp vụ Chuyển hàng giữa các chi nhánh (TRF######) — điều phối tồn kho trong hệ thống đa chi nhánh.
**Entity:** E27 StockTransfer
**Prefix mã:** TRF + 6 chữ số (VD: TRF000123)
**Liên quan:** `01-tong-quan.md §C1`, Module 09 (Giao vận — nếu dùng shipper nội bộ)

---

## 1. Vai trò trong hệ thống

Chuyển hàng giải quyết **bài toán tồn kho không đều** trong chuỗi đa chi nhánh:

```
[CN Trung tâm: tồn 100 cái] ──TRF──► [CN Quận 1: tồn 5 cái → sắp hết]
         tồn giảm 20                          tồn tăng 20

[Thẻ kho CN Trung tâm] ← ghi TRF000123 / xuất 20 cái
[Thẻ kho CN Quận 1]    ← ghi TRF000123 / nhận 20 cái
```

Chuyển hàng **không tác động doanh thu** — chỉ thay đổi vị trí tồn kho.

---

## 2. Schema Phiếu Chuyển hàng (E27)

### Header

| Trường | Bắt buộc | Ghi chú |
|---|---|---|
| Mã phiếu | Có | TRF######, auto |
| Chi nhánh gửi | Có | CN nguồn |
| Chi nhánh nhận | Có | CN đích; phải khác CN gửi |
| Ngày tạo | Có | Auto timestamp |
| Ngày chuyển dự kiến | Không | Kế hoạch giao hàng |
| Người tạo | Có | Nhân viên tạo phiếu |
| Người phụ trách giao | Không | NV hoặc shipper giao hàng |
| Ghi chú | Không | |
| Trạng thái | Có | Xem §3 |

### Lines

| Trường | Ghi chú |
|---|---|
| Hàng hóa (FK Product) | |
| Đơn vị tính | Theo UoM đang dùng |
| Số lượng chuyển | SL kế hoạch gửi đi |
| Số lượng thực nhận | SL thực tế nhận được (điền khi xác nhận nhận hàng) |
| Chênh lệch | = SL chuyển − SL thực nhận |
| Lô / Serial | Nếu SP quản lý lô/serial thì phải chỉ định |
| Ghi chú dòng | VD: "hàng bị móp 2 cái" |

---

## 3. State Machine

```
[Phiếu tạm] ──(xác nhận gửi)──► [Đang chuyển] ──(xác nhận nhận)──► [Hoàn thành]
     │                                  │
     └── (hủy)──► [Đã hủy]              └── (hủy giữa chừng) ──► [Đã hủy]
```

| Trạng thái | Tồn CN gửi | Tồn CN nhận | Mô tả |
|---|---|---|---|
| Phiếu tạm | Không đổi | Không đổi | Chưa thực hiện |
| Đang chuyển | Giảm (trừ ngay khi xác nhận gửi) | Chưa tăng | Hàng đang trên đường |
| Hoàn thành | Đã giảm | Tăng theo SL thực nhận | Giao dịch hoàn tất |
| Đã hủy | Hoàn tồn lại (nếu đã trừ) | Không đổi | |

**Quan trọng: Trạng thái "Đang chuyển"**
- Tồn CN gửi đã giảm nhưng tồn CN nhận chưa tăng
- POS CN nhận KHÔNG được bán số hàng này
- Hiển thị cột "Đang nhận" (in-transit) trong màn tồn kho

---

## 4. Luồng nghiệp vụ chi tiết

### Luồng chuẩn (Confirmed Transfer)

```
Chủ shop / QL CN xem báo cáo tồn → thấy CN Quận 1 sắp hết hàng A
→ Tạo TRF: gửi từ CN TT → CN Quận 1 / 20 cái hàng A
→ Lưu Phiếu tạm
→ Xác nhận gửi → tồn CN TT giảm 20, thẻ kho TT ghi TRF000123/xuất
→ NV kho đóng gói, giao cho shipper
→ Khi hàng đến CN Quận 1 → NV nhận kiểm đếm → Nhập SL thực nhận
→ Xác nhận nhận → tồn CN Quận 1 tăng, thẻ kho Q1 ghi TRF000123/nhập
→ Trạng thái: Hoàn thành
```

### Luồng có chênh lệch (Partial / Damaged)

```
Gửi 20 cái → Thực nhận 18 cái (2 cái vỡ trên đường)
→ CN nhận nhập SL thực nhận = 18, ghi chú "2 cái vỡ"
→ Hoàn thành: CN TT mất 20, CN Quận 1 nhận 18
→ Chênh lệch 2 cái → cần xử lý thủ công: Xuất hủy tại CN TT hoặc tạo Phiếu điều chỉnh
→ Cảnh báo: phiếu có chênh lệch cần xử lý trong N ngày
```

---

## 5. Phân quyền chuyển hàng

| Action | Ai được làm |
|---|---|
| Tạo phiếu chuyển | NV kho / QL CN |
| Xác nhận gửi (CN nguồn) | NV kho CN nguồn / QL CN nguồn |
| Xác nhận nhận (CN đích) | NV kho CN đích / QL CN đích |
| Hủy phiếu đã gửi | Chủ shop / QL cấp cao |

---

## 6. Filter & Search trên list Chuyển hàng

| Filter | Options |
|---|---|
| Loại | Chuyển đi / Nhận về |
| Trạng thái | Phiếu tạm / Đang chuyển / Hoàn thành / Đã hủy |
| CN gửi / CN nhận | Dropdown |
| Ngày tạo | Khoảng thời gian |
| Có chênh lệch | Toggle — lọc nhanh các phiếu SL thực nhận ≠ SL gửi |

**Column đặc trưng "Khớp / Không khớp":**
- Khớp: SL thực nhận = SL gửi
- Không khớp: có chênh lệch → cần xử lý

---

## 7. Tích hợp với Giao vận

Khi chuỗi dùng shipper nội bộ hoặc thuê đơn vị giao vận chuyển hàng giữa kho:
- TRF liên kết với Vận đơn (Module 09)
- Track trạng thái giao hàng qua mã vận đơn
- Khi vận đơn "Đã giao" → tự động chuyển TRF sang bước "Xác nhận nhận"

---

## 8. Pain points

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không có cảnh báo khi tồn CN thấp để đề xuất chuyển hàng | Cao |
| P2 | Chênh lệch hàng hóa khi chuyển không có workflow xử lý rõ ràng | Cao |
| P3 | Không track được hàng đang trên đường (in-transit) trên màn hình tồn tổng | Trung bình |
| P4 | Không thể chuyển hàng theo lô/serial được chỉ định cụ thể | Trung bình |
| P5 | Không có phê duyệt 2 bước: QL CN gửi phê duyệt trước khi xuất hàng | Trung bình |

---

## 9. Cơ hội cải tiến

### I1. Smart Transfer Suggestion
- Hệ thống tự động đề xuất điều chuyển: "CN Quận 1 dự kiến hết hàng A trong 3 ngày, CN TT có 150 cái — đề xuất chuyển 50 cái"
- Kết hợp với AI Demand Forecasting (brainstorm A2)

### I2. In-transit visibility
- Cột "Đang chuyển đến" rõ ràng trên màn tồn kho mỗi CN
- NV CN nhận biết được hàng sắp về mà không cần hỏi CN gửi

### I3. Approval workflow
- Cấu hình: chuyển hàng > X cái / > Y giá trị → cần QL hoặc chủ shop duyệt
- Thông báo approval qua Zalo/SMS

### I4. Chênh lệch workflow
- Khi nhận ít hơn gửi → hệ thống tự đề xuất: tạo Xuất hủy tại CN gửi cho SL chênh
- Ảnh chụp hàng hư hỏng đính kèm phiếu làm bằng chứng
