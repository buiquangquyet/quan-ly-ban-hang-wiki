# Deep dive — Phiếu Kiểm kho (Stock Take)

**Phạm vi:** Nghiệp vụ Kiểm kho (KK######) — đối chiếu tồn kho thực tế với sổ sách để phát hiện chênh lệch và cân bằng lại hệ thống.
**Entity:** E28 StockTake
**Prefix mã:** KK + 6 chữ số (VD: KK000045)
**Liên quan:** `01-tong-quan.md §C2`, `02-the-kho.md` (thẻ kho ghi nhận điều chỉnh)

---

## 1. Mục đích nghiệp vụ

Kiểm kho (còn gọi: kiểm đếm, đối soát kho) giải quyết sự thất thoát giữa **tồn sổ sách** và **tồn thực tế**:

```
Tồn sổ sách (KiotViet) ≠ Tồn thực tế (đếm tay)
           ↑
  Nguyên nhân phổ biến:
  - Hàng thất thoát (mất cắp, hư hỏng không ghi)
  - Nhập/xuất nhầm số lượng
  - Hàng giao nhầm không cập nhật
  - Serial/lô bị nhầm
  - Lỗi đồng bộ kênh online
```

Sau kiểm kho → **cân bằng kho**: điều chỉnh tồn về đúng thực tế → thẻ kho ghi nhận.

---

## 2. Schema Phiếu Kiểm kho (E28)

### Header

| Trường | Bắt buộc | Ghi chú |
|---|---|---|
| Mã phiếu | Có | KK######, auto |
| Chi nhánh | Có | Kiểm kho tại CN nào |
| Ngày tạo | Có | |
| Ngày kiểm dự kiến | Không | Lên lịch kiểm trước |
| Người tạo | Có | |
| Trạng thái | Có | Xem §3 |
| Loại kiểm | Có | Toàn bộ / Một phần (nhóm hàng, vị trí kho) |
| Ghi chú | Không | |

### Lines

| Trường | Ghi chú |
|---|---|
| Hàng hóa (FK Product) | |
| Đơn vị tính | |
| SL tồn sổ sách | Lấy từ `onHand` tại thời điểm tạo phiếu |
| SL thực tế | NV đếm và nhập vào |
| Chênh lệch | = SL thực tế − SL sổ sách (âm = thất thoát, dương = dư) |
| Giá trị chênh lệch | = Chênh lệch × Giá vốn (WAC) |
| Lô / Serial | Nếu SP quản lý lô/serial — đối chiếu chi tiết |
| Ghi chú dòng | Lý do chênh lệch |

---

## 3. State Machine

```
[Phiếu tạm] ──(bắt đầu kiểm)──► [Đang kiểm] ──(cân bằng kho)──► [Đã cân bằng]
     │                                 │
     └── (hủy) ──► [Đã hủy]            └── (hủy) ──► [Đã hủy]
```

| Trạng thái | Tồn kho | Mô tả |
|---|---|---|
| Phiếu tạm | Không đổi | Chưa bắt đầu đếm |
| Đang kiểm | **Không thay đổi** (quan trọng) | NV đang đếm, chưa cân bằng |
| Đã cân bằng | **Cập nhật theo SL thực tế** | Điều chỉnh tồn theo kết quả đếm |
| Đã hủy | Không đổi (rollback nếu cần) | |

**Lưu ý quan trọng:**
- Khi trạng thái "Đang kiểm" → hệ thống **KHÔNG khóa** bán hàng (shop vẫn hoạt động)
- SL tồn sổ sách trong phiếu = snapshot tại thời điểm tạo phiếu, KHÔNG cập nhật theo giao dịch mới
- Sau khi cân bằng → thẻ kho ghi nhận bút toán điều chỉnh (loại giao dịch KK — xem `02-the-kho.md`)

---

## 4. Luồng nghiệp vụ chi tiết

### Luồng kiểm kho chuẩn

```
1. Tạo phiếu kiểm kho
   → Chọn loại: Toàn bộ hoặc Theo nhóm hàng / Vị trí kho
   → Hệ thống auto-load danh sách SP với SL tồn sổ sách hiện tại
   → Lưu Phiếu tạm (snapshot SL)

2. Bắt đầu kiểm (chuyển sang "Đang kiểm")
   → NV nhận phiếu, ra kho đếm vật lý
   → Dùng app mobile hoặc máy in barcode scan từng SP
   → Nhập SL thực tế vào từng dòng

3. Review chênh lệch
   → Màn hình tổng hợp: tổng chênh lệch tăng / giảm / giá trị
   → Dòng có chênh lệch lớn → kiểm lại lần 2 trước khi cân bằng

4. Cân bằng kho
   → Chủ shop / QL phê duyệt
   → Hệ thống điều chỉnh onHand theo SL thực tế
   → Ghi thẻ kho: loại giao dịch "Kiểm kho cân bằng"
   → Trạng thái: Đã cân bằng
```

### Kiểm kho theo nhóm hàng (Partial Stock Take)

Cho chuỗi lớn không thể đóng cửa kiểm toàn bộ:
- Chia kho thành nhiều nhóm (theo nhóm hàng, vị trí kho, nhà cung cấp)
- Kiểm luân phiên — mỗi ngày kiểm 1 nhóm
- Cần logic tránh xung đột: SP đang bán không đồng thời nằm trong phiếu kiểm chưa cân bằng

---

## 5. Xử lý chênh lệch

| Loại chênh lệch | Giá trị | Ý nghĩa | Tác động tài chính |
|---|---|---|---|
| Thừa (+) | SL thực > SL sổ | Nhập sót, đếm sai lần trước | Tăng tồn → tăng giá vốn hàng tồn |
| Thiếu (−) | SL thực < SL sổ | Mất cắp, hư hỏng, xuất sót | Giảm tồn → ghi chi phí thất thoát |

**Phân tích nguyên nhân chênh lệch:**
- Chênh lệch nhỏ (<1%): bình thường, do sai số đếm
- Chênh lệch lớn tập trung vào 1 NV/1 ca: dấu hiệu gian lận → trigger Audit
- Chênh lệch tăng đều đặn theo tháng: lỗi quy trình nhập/xuất

---

## 6. Kiểm kho với Lô & Serial

### Kiểm kho hàng có Lô (Batch)

- Không chỉ đếm số lượng — cần xác nhận đúng lô
- Phiếu kiểm liệt kê từng Lô với SL riêng
- Phát hiện: lô đã hết hạn vẫn còn trong kho → xuất hủy ngay

### Kiểm kho hàng có Serial/IMEI

- 1:1 — kiểm từng serial
- Quét barcode / IMEI → hệ thống match với danh sách StockUnit còn trong kho
- Serial có trong kho vật lý nhưng không trong hệ thống → cần xác minh nguồn gốc
- Serial trong hệ thống nhưng không tìm thấy vật lý → báo mất, ghi thẻ kho

---

## 7. Báo cáo kiểm kho

| Báo cáo | Mô tả |
|---|---|
| Kết quả kiểm kho | Chi tiết chênh lệch từng SP trong 1 phiếu |
| Lịch sử kiểm kho | Tất cả phiếu KK theo thời gian |
| Tích lũy chênh lệch | Tổng chênh lệch theo SP theo nhiều kỳ kiểm — phát hiện SP hay "hao hụt" |
| Giá trị thất thoát | Chênh lệch âm × Giá vốn → thiệt hại tài chính |

---

## 8. Pain points

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không có app mobile để NV scan barcode khi đếm — phải ghi tay rồi nhập lại | Cao |
| P2 | Kiểm kho toàn bộ buộc phải đóng cửa — mất doanh thu nguyên ngày | Cao |
| P3 | Không có workflow phê duyệt trước khi cân bằng — NV có thể cân bằng sai | Trung bình |
| P4 | Không lên lịch kiểm kho định kỳ tự động — quên kiểm hàng tháng | Trung bình |
| P5 | Kiểm kho Serial không có màn hình scan riêng | Trung bình |
| P6 | Không có phân tích xu hướng chênh lệch theo thời gian | Thấp |

---

## 9. Cơ hội cải tiến

### I1. Mobile Stocktake App
- App mobile dùng camera scan barcode/QR từng SP
- Offline mode — NV ở kho không có WiFi vẫn đếm được, sync sau
- Hiển thị ngay "Sổ sách: 50 cái" để NV biết cần đếm đến bao nhiêu

### I2. Cycle Count (Kiểm kho luân phiên)
- Chia kho thành zone → kiểm mỗi zone 1 lần/tuần
- Không cần đóng cửa kiểm toàn bộ
- Hệ thống tự đề xuất SP cần ưu tiên kiểm (SP bán chạy, SP có chênh lệch lịch sử cao)

### I3. Phê duyệt trước khi cân bằng
- Cân bằng chênh lệch > X cái hoặc > Y giá trị → cần QL duyệt
- QL review kết quả trên điện thoại → approve 1 chạm

### I4. Anomaly Detection từ kiểm kho
- Nếu SP X luôn bị thiếu sau kiểm → cảnh báo chủ shop
- Liên kết với Audit module: NV nào hay làm ca khi SP đó thiếu?
