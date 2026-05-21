# MODULE HÓA ĐƠN (INVOICES)

**Phạm vi:** Hóa đơn (`HD######`) là chứng từ thương mại cốt lõi — ghi nhận giao dịch bán đã chốt, trừ tồn thật, ghi doanh thu, và có thể phát hành hóa đơn điện tử (HĐĐT) theo NĐ123/2020 + TT78/2021. Đây là phiếu trung tâm kết nối hầu hết các module trong hệ thống.

---

## 1. Tích hợp liên module

Hóa đơn là điểm hội tụ của 8 luồng nghiệp vụ:

| Luồng | Quan hệ với hóa đơn |
|---|---|
| Bán nhanh tại POS | Tạo hóa đơn trực tiếp, không qua đặt hàng |
| Đặt hàng (DH) | Xử lý đơn hàng → tạo hóa đơn từ đặt hàng |
| Trả hàng (TH) | Phiếu trả hàng bắt buộc tham chiếu hóa đơn gốc (trừ "Trả nhanh") |
| Sổ quỹ | Phát sinh phiếu thu (`TTHD###`) khi khách hàng thanh toán |
| Thẻ kho | Mỗi dòng hàng tạo ra 1 dòng xuất kho |
| Khách hàng | Cập nhật tổng mua, công nợ, điểm tích lũy |
| HĐĐT | Hóa đơn có thể phát hành thành hóa đơn điện tử chứng từ thuế |
| Vận chuyển | Hóa đơn giao hàng phát sinh vận đơn carrier (KShip, GHN, Viettel...) |

---

## 2. Phân loại hóa đơn

| Loại | Đặc điểm | Trường hợp sử dụng |
|---|---|---|
| **Không giao hàng** | Bán ngay tại quầy, không cần ship | POS truyền thống, F&B, walk-in |
| **Giao hàng** | Có địa chỉ giao, liên kết vận đơn, hỗ trợ COD | Đơn hàng online, giao tận nơi |

Cả 2 loại dùng chung prefix `HD######`.

---

## 3. Thông tin hóa đơn

### 3.1 Thông tin chung

Mỗi hóa đơn ghi nhận:
- Khách hàng (hoặc "Khách lẻ" nếu không xác định danh tính)
- Chi nhánh, kênh bán (POS, Shopee, An95 Store...), bảng giá
- Người bán, người tạo, ngày bán
- Ghi chú

### 3.2 Dòng hàng

Mỗi dòng hàng ghi nhận:
- Sản phẩm, đơn vị tính, số lượng
- Đơn giá, giảm giá, giá bán — hiển thị cả **trước và sau thuế** (toggle được)
- **Thuế suất riêng theo từng dòng hàng** — hỗ trợ đa thuế suất (0%, 5%, 8%, 10%) trên cùng 1 hóa đơn
- Serial/IMEI nếu sản phẩm yêu cầu quản lý serial
- Lô hàng/hạn sử dụng nếu có

### 3.3 Tổng thanh toán

Phần tổng hóa đơn hiển thị đầy đủ:
- Tổng tiền hàng
- Giảm giá hóa đơn
- **VAT phân tách theo từng mức thuế suất** — đáp ứng yêu cầu TT78/2021
- Phụ phí tùy chỉnh ("Thu khác" do người dùng đặt tên)
- Phí vận chuyển (nếu là hóa đơn giao hàng)
- Khách cần trả / Khách đã trả / Còn nợ

### 3.4 Thanh toán

- Hỗ trợ đa phương thức: Tiền mặt, Chuyển khoản, Thẻ ngân hàng, Ví điện tử, Voucher, Điểm tích lũy
- Hỗ trợ thanh toán từng phần (ghi nợ khách hàng)
- Một hóa đơn có thể liên kết nhiều phiếu thu

---

## 4. Vòng đời & Trạng thái

### 4.1 Trạng thái hóa đơn

| Trạng thái | Mô tả | Ghi chú |
|---|---|---|
| **Đang xử lý** | Đã tạo nhưng chưa hoàn tất — chưa thu đủ hoặc chưa giao xong | Tồn đã trừ, doanh thu đã ghi |
| **Hoàn thành** | Đã thu đủ tiền + đã giao hàng thành công (nếu có giao hàng) | Đóng quy trình |
| **Không giao được** | Carrier báo giao thất bại (KH không nhận, sai địa chỉ...) | Chờ xử lý tiếp: giao lại / hoàn hàng / hủy |
| **Đã hủy** | Hủy toàn bộ hóa đơn | Khôi phục tồn kho + hủy/giữ phiếu thu (tùy chọn) |

### 4.2 Trạng thái giao hàng (chỉ hóa đơn Giao hàng)

| Trạng thái | Mô tả |
|---|---|
| Chờ xử lý | Chờ nhân viên đóng gói và đăng vận đơn |
| Chờ lấy hàng | Đã đăng vận đơn, chờ carrier đến lấy |
| Lấy hàng thành công | Carrier đã lấy hàng |
| Lấy hàng thất bại | Carrier không lấy được hàng |
| Đang giao hàng | Hàng đang trên đường vận chuyển |
| Giao hàng thành công | Hàng đã đến tay người nhận → hóa đơn chuyển sang Hoàn thành |
| Giao hàng thất bại | Không giao được → hóa đơn chuyển sang Không giao được |
| Đang chuyển hoàn | Carrier đang trả hàng về shop |
| Đã chuyển hoàn | Hàng đã về → tự động tạo phiếu trả hàng |

### 4.3 Trạng thái HĐĐT

**Luồng phát hành:**

| Trạng thái | Mô tả |
|---|---|
| Chưa phát hành | Hóa đơn chưa được gửi lên hệ thống thuế |
| Đang xử lý | Đang gửi lên Cơ quan thuế (CQT) |
| Đã gửi CQT | Đã gửi, chờ CQT phản hồi |
| Đã phát hành | CQT chấp nhận |
| Đã chuyển | Đã gửi HĐĐT đến khách hàng |
| Phát hành lỗi | CQT từ chối — cần sửa và gửi lại |

**Luồng điều chỉnh/thay thế (sau khi đã phát hành):**

Hỗ trợ đầy đủ các trạng thái cho quy trình Thông báo sai sót, HĐ điều chỉnh, HĐ thay thế theo TT78/2021: Chưa phát hành, Người mua yêu cầu, Đã gửi, Hợp lệ, Không hợp lệ, Bị từ chối, Đã hủy, Đã xác nhận, Đã xóa, Lỗi.

---

## 5. Chức năng chính

### 5.1 Tạo và chỉnh sửa

- **Tạo hóa đơn:** Từ POS bán nhanh, từ đặt hàng, hoặc tạo trực tiếp trên màn hóa đơn
- **Sao chép hóa đơn:** Tạo hóa đơn mới từ hóa đơn cũ — tiện cho khách hàng mua định kỳ hoặc phục hồi hóa đơn đã hủy
- **Import Excel:** Nhập hàng loạt từ file, tự động liên kết khách hàng cũ qua mã/SĐT, tự động tạo phiếu thu nếu có thông tin thanh toán
- **Xuất file:** Xuất tổng quan hoặc chi tiết hóa đơn

**Quy tắc chỉnh sửa:**

| Trạng thái hóa đơn | Quy tắc |
|---|---|
| Đang xử lý | Có thể sửa: hàng hóa, khách hàng, bảng giá, người bán, thời gian, thanh toán |
| Hoàn thành | Phải hủy hóa đơn cũ và tạo mới (không sửa trực tiếp) |
| Có Serial/Lô/HSD | Không được sửa thời gian hóa đơn |
| Đã phát hành HĐĐT | Phải điều chỉnh qua quy trình HĐĐT, không sửa trực tiếp hóa đơn |

### 5.2 Trả hàng

- Nút "Trả hàng" ngay trên màn chi tiết hóa đơn → tạo phiếu trả hàng liên kết với hóa đơn gốc
- Tiện cho thu ngân khi khách mang hóa đơn đến trả hàng trực tiếp tại quầy

### 5.3 Thu nợ qua mã QR

- Tạo mã QR thanh toán từ hóa đơn → gửi cho khách hàng quét
- Nếu shop đã đăng ký nhận thông báo QR: tự động xác nhận và tạo phiếu thu khi tiền về tài khoản
- Mẫu hóa đơn in có thể nhúng QR sẵn — khách hàng quét QR trực tiếp trên phiếu in

### 5.4 Phát hành HĐĐT

1. Tạo hóa đơn trên KiotViet bình thường
2. Click "Phát hành HĐĐT" → KiotViet gửi sang KV eInvoice
3. KV eInvoice gửi lên Cơ quan thuế qua API
4. CQT phản hồi → cập nhật trạng thái
5. Nếu sai sót: lập Thông báo sai sót / HĐ điều chỉnh / HĐ thay thế

Hỗ trợ bảng kê tổng hợp/điều chỉnh/thay thế phục vụ kiểm toán thuế.

### 5.5 Giao hàng & KShip

- Tích hợp nhiều carrier: KShip, GHN, Viettel Post, Grab, J&T...
- **KShip** — carrier first-party của KiotViet: xác thực OTP, thanh toán cước qua ví KiotViet, đối soát nhanh
- Lịch sử giao hàng: ghi nhận đầy đủ mọi sự kiện trạng thái vận đơn theo thời gian

### 5.6 Tìm kiếm & Lọc

Hỗ trợ lọc theo nhiều tiêu chí kết hợp:

| Tiêu chí | Loại |
|---|---|
| Chi nhánh | Chọn nhiều |
| Thời gian tạo hóa đơn | Khoảng ngày |
| Loại hóa đơn | Không giao hàng / Giao hàng |
| Trạng thái hóa đơn | 4 trạng thái |
| Trạng thái HĐĐT | 10+ trạng thái |
| Trạng thái giao hàng | 7+ trạng thái |
| Đối tác giao hàng | Theo carrier |
| Thời gian giao hàng | Khoảng ngày (hỗ trợ đặt trước) |
| Khu vực giao hàng | Tỉnh/TP → Quận/Huyện |

---

## 6. So sánh với phiếu liên quan

| Khía cạnh | Đặt hàng (DH) | Hóa đơn (HD) | Trả hàng (TH) |
|---|---|---|---|
| Trừ tồn thật | Chỉ đặt giữ | Có | Hoàn lại |
| Ghi doanh thu | Không | Có | Giảm doanh thu |
| Tạo phiếu thu | Thu cọc (TTDH) | Thu đầy đủ (TTHD) | Chi (TTTH) |
| Phát hành HĐĐT | Không | Có | Cần HĐ điều chỉnh |
| Tích hợp giao hàng | Có | Có | Tự động từ chuyển hoàn |
| Tích lũy điểm | Không | Cộng điểm | Trừ điểm |
| Công nợ khách hàng | Không | Cộng nợ nếu chưa đủ | Trừ nợ |
| Đa thuế suất VAT | Có | Có | Theo hóa đơn gốc |
| Chỉnh sửa sau tạo | Linh hoạt | Hạn chế (chỉ Đang xử lý) | Hạn chế |

---

## 7. Điểm đau & Cơ hội cải tiến

### 7.1 Quy trình & Trạng thái

| # | Vấn đề | Đề xuất |
|---|---|---|
| 1 | Sửa hóa đơn Hoàn thành = Hủy + Tạo mới → mất liên kết lịch sử kiểm toán | Versioning hóa đơn (HD v2, v3 liên kết phiếu gốc) |
| 2 | Sửa hóa đơn không kiểm tra lại tồn kho — dễ gây âm tồn | Kiểm tra tồn khi thay đổi số lượng/sản phẩm |
| 3 | Trạng thái "Không giao được" không có luồng xử lý tiếp rõ ràng (giao lại / hoàn hàng / hủy) | Thêm hướng dẫn decision tree |
| 4 | Không có tính năng phát hành HĐĐT hàng loạt từ màn danh sách | Chọn nhiều hóa đơn → phát hành 1 lần |
| 5 | Không cảnh báo khi hóa đơn "Đang xử lý" tồn quá lâu (vd > 7 ngày) | Báo cáo tồn đọng + cảnh báo SLA |

### 7.2 Giá & Thuế

| # | Vấn đề | Đề xuất |
|---|---|---|
| 6 | Bảng giá nhiều cột — không có chế độ hiển thị gọn | Chế độ Compact / Comfortable |
| 7 | "Thu khác" tùy chỉnh không có mô tả rõ ràng hiển thị cho khách hàng | Thêm trường mô tả cho từng loại phụ phí |
| 8 | Nhiều thuế suất trên 1 hóa đơn nhưng UI chưa phân biệt rõ từng mức | Hiển thị tách biệt: VAT 0%, VAT 5%, VAT 8%, VAT 10% |
| 9 | Lãi gộp không hiển thị trên chi tiết hóa đơn | Toggle "Xem lãi gộp" dành cho quản lý |

### 7.3 HĐĐT

| # | Vấn đề | Đề xuất |
|---|---|---|
| 10 | Trạng thái HĐĐT phức tạp (10+ trạng thái) — khó theo dõi | Thanh tiến trình trực quan: Nháp → Gửi → CQT kiểm tra → Phát hành |
| 11 | Khi CQT từ chối không có hướng dẫn sửa lỗi | Gợi ý sửa lỗi thường gặp + gửi lại tự động |
| 12 | Không có preview HĐĐT trước khi phát hành ngay trên màn hóa đơn | Nút "Xem trước" inline |
| 13 | Quy trình sửa sai sót HĐĐT phải vào module KV eInvoice riêng | Tích hợp inline trên màn hóa đơn |

### 7.4 Giao hàng

| # | Vấn đề | Đề xuất |
|---|---|---|
| 14 | Lịch sử giao hàng chỉ liệt kê thô, không có timeline trực quan | Timeline dọc với icon theo trạng thái |
| 15 | Không so sánh phí/thời gian giữa các carrier ngay trên hóa đơn | Báo giá realtime đa carrier |
| 16 | Không lưu bằng chứng giao hàng (chữ ký KH, ảnh giao) | Kéo dữ liệu bằng chứng giao hàng từ carrier API |

### 7.5 Khách hàng & Tích lũy

| # | Vấn đề | Đề xuất |
|---|---|---|
| 17 | Tạo hóa đơn với "Khách lẻ" làm mất cơ hội tích điểm | Tự động nhắc nhập SĐT trước khi lưu hóa đơn |
| 18 | Không có gợi ý upsell cuối hóa đơn | AI gợi ý sản phẩm khách hàng thường mua kèm |

### 7.6 In ấn & Mẫu

| # | Vấn đề | Đề xuất |
|---|---|---|
| 19 | Không preview mẫu in ngay khi tạo hóa đơn | Nút "In thử với mẫu hiện tại" inline |
| 20 | Mã QR in luôn hiển thị dù khách đã thanh toán đủ | Logic template: chỉ hiện QR khi còn nợ > 0 |

---

## 8. Cơ hội đột phá — Top 5

| # | Tính năng | Vấn đề giải quyết | Độ khó | Tác động |
|---|---|---|---|---|
| 1 | **Versioning hóa đơn** — thay vì hủy+tạo mới khi sửa hóa đơn Hoàn thành | #1 — audit trail và compliance | Cao | Rất cao |
| 2 | **Quy trình HĐĐT tích hợp inline** — preview + phát hành + sửa sai sót ngay trên màn hóa đơn | #10, #13 | Cao | Rất cao (compliance) |
| 3 | **So sánh carrier + bằng chứng giao hàng** — báo giá realtime, lưu chữ ký + ảnh giao | #15, #16 | Cao | Cao |
| 4 | **Phát hành HĐĐT hàng loạt + tự động sửa lỗi CQT** | #4, #11 | Trung | Rất cao |
| 5 | **Lãi gộp & Gợi ý upsell AI** — hiển thị lãi gộp per hóa đơn + gợi ý sản phẩm kèm | #9, #18 | Trung | Cao |

---

**Tóm lược:**
Hóa đơn KiotViet là module kết nối 8 luồng nghiệp vụ, hỗ trợ đa thuế suất VAT trên cùng 1 phiếu, tích hợp HĐĐT đầy đủ theo TT78/2021, và có carrier integration với KShip first-party. Cơ hội cải tiến tập trung vào versioning audit trail, inline HĐĐT workflow, và intelligence (carrier comparison, AI upsell).
