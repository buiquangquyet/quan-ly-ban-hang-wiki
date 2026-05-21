# MODULE ĐẶT HÀNG (ĐƠN HÀNG) — NGHIỆP VỤ CHI TIẾT

**Phạm vi:** Đặt hàng (Customer Order, mã prefix `DH`) là **phiếu cam kết bán** — khách hàng đã chốt mua nhưng chưa xuất hóa đơn. Đây là lớp đệm giữa "ý định mua" và "doanh thu đã ghi nhận".

---

## 1. KHÁI NIỆM & VỊ TRÍ TRONG QUY TRÌNH BÁN HÀNG

Luồng khách hàng chuẩn diễn ra theo thứ tự:

> **Khách hỏi** → **Khách chốt mua** → **Chuẩn bị & giao hàng** → **Khách nhận** → **Hoàn tất**

Trong đó:
- Bước "Khách chốt mua" tạo ra **Phiếu đặt hàng (DH)** — tồn kho được **đặt giữ** (reserved) cho khách, nhưng chưa thực sự xuất kho.
- Bước "Khách nhận" tạo ra **Hóa đơn (HD)** — tồn kho thực sự giảm, doanh thu được ghi nhận.

### 1.1 Phân biệt Đặt hàng (DH) và Hóa đơn (HD)

| Khía cạnh | Đặt hàng (DH) | Hóa đơn (HD) |
|---|---|---|
| Thời điểm tạo | Khách chốt mua, chưa giao hàng | Khách nhận hàng hoặc thanh toán xong |
| Tác động tồn kho | Không giảm tồn thật — chỉ đặt giữ số lượng cho khách | Giảm tồn thật |
| Doanh thu | Chưa ghi nhận vào sổ | Ghi nhận vào doanh thu và sổ kế toán |
| Thanh toán | Có thể thu cọc một phần (Phiếu thu cọc) | Thanh toán đầy đủ (Phiếu thu hóa đơn) |
| Hủy không ảnh hưởng doanh thu | Có — hủy thẳng | Không — cần lập Phiếu trả hàng |
| Hóa đơn điện tử | Chưa phát hành | Có thể phát hành sau khi tạo HD |

→ **DH là cam kết mềm (soft commitment)**, HD là giao dịch cứng (hard transaction). Đây là chuẩn của thương mại điện tử (Order vs Invoice).

### 1.2 Phân biệt DH (Đặt hàng bán) và DHN (Đặt hàng nhập)

Không nhầm lẫn hai loại phiếu này:
- **DH** = Khách hàng đặt mua từ cửa hàng (Customer Order).
- **DHN** = Cửa hàng đặt mua từ nhà cung cấp (Purchase Order — xem tài liệu nhập hàng).

Cùng dạng "phiếu cam kết" nhưng chiều giao dịch ngược nhau.

---

## 2. CÁC THÔNG TIN TRÊN PHIẾU ĐẶT HÀNG

### 2.1 Thông tin đầu phiếu

| Thông tin | Mô tả |
|---|---|
| Mã phiếu | Định dạng DH + 6 chữ số, tăng dần theo thứ tự của từng merchant |
| Khách hàng | Bắt buộc chọn — không hỗ trợ "Khách lẻ" không tên như POS |
| Chi nhánh xử lý | Chi nhánh chịu trách nhiệm đơn hàng này |
| Ngày đặt | Thời điểm tạo phiếu |
| Người tạo | Nhân viên tạo phiếu |
| Người nhận đặt | Nhân viên chịu trách nhiệm chăm sóc đơn (có thể khác người tạo) |
| Kênh bán | Nguồn đơn: Bán trực tiếp, Shopee, TikTok Shop, Lazada, Facebook Shop, Zalo Shop... |
| Bảng giá | Bảng giá áp dụng cho đơn (mặc định là Bảng giá chung) |
| Ghi chú | Ghi chú tự do |

### 2.2 Thông tin dòng hàng (Line Items)

Mỗi dòng hàng trên phiếu đặt gồm:
- Sản phẩm (mã hàng, tên hàng, phiên bản nếu có)
- Đơn vị tính
- Số lượng đặt / Số lượng đã xuất — theo dõi mức độ hoàn thành từng dòng. Ví dụ "1/0" nghĩa là đặt 1, chưa xuất cái nào.
- Đơn giá (trước thuế và sau thuế)
- Chiết khấu dòng (trước thuế và sau thuế)
- Giá bán thực tế (sau chiết khấu, trước và sau thuế)
- Thành tiền (trước thuế và sau thuế)

KiotViet tách biệt rõ ràng **giá trước thuế** và **giá sau thuế** trên 4 trường — đáp ứng yêu cầu hóa đơn điện tử VAT theo nghị định hiện hành, cho phép kế toán đối chiếu giá net và gross một cách rõ ràng.

### 2.3 Tổng tiền phiếu

| Khoản mục | Mô tả |
|---|---|
| Tổng tiền hàng | Cộng thành tiền tất cả dòng hàng |
| Giảm giá phiếu | Chiết khấu áp lên toàn bộ phiếu (ngoài chiết khấu từng dòng) |
| Thu khác (tùy chỉnh) | Các loại phụ phí tự định nghĩa theo nghiệp vụ (ví dụ: phí đóng gói, phụ phí...) |
| VAT | Thuế giá trị gia tăng (tỷ lệ áp dụng: 10% hoặc theo cấu hình) |
| Phí vận chuyển | Phí giao hàng thu từ khách |
| Tổng cộng | Số tiền khách cần trả |
| Khách đã trả | Số tiền cọc đã thu |
| Còn lại | Tổng cộng trừ đã trả |

### 2.4 Thông tin giao hàng (tùy chọn)

Phần giao hàng có thể bật hoặc tắt tùy đơn. Khi bật, ghi nhận đầy đủ:

| Nhóm thông tin | Chi tiết |
|---|---|
| Địa chỉ giao | Địa chỉ, phường/xã, quận/huyện, tỉnh/thành phố |
| Địa chỉ lấy hàng | Kho/điểm xuất phát |
| Người nhận | Họ tên, số điện thoại |
| Đối tác giao hàng | Hãng vận chuyển (Grab, GHN, Viettel Post, GHTK, J&T, AhaMove...) |
| Mã vận đơn | Do hãng vận chuyển cấp, đồng bộ 2 chiều nếu tích hợp API |
| Thông số kiện hàng | Trọng lượng (gram), kích thước (cm) |
| Khai giá | Bật/tắt khai báo giá trị hàng để mua bảo hiểm vận chuyển |
| Dịch vụ vận chuyển | Loại dịch vụ (Siêu tốc, Tiêu chuẩn...) |
| Thu hộ tiền (COD) | Bật/tắt — nếu bật, hãng vận chuyển thu tiền hộ khi giao |
| Phí trả đối tác giao hàng | Phí KiotViet trả cho hãng VC (có thể khác phí thu khách) |
| Thời gian giao dự kiến | Ngày/giờ dự kiến giao |

**Lưu ý nghiệp vụ phí ship:** "Phí vận chuyển" trên tổng phiếu là số tiền thu từ khách hàng. "Phí trả đối tác giao hàng" là số tiền cửa hàng trả cho hãng VC. Hai con số này có thể khác nhau (ví dụ: thu khách 30.000đ nhưng thực tế trả hãng 45.000đ — cửa hàng bù phần chênh).

---

## 3. VÒNG ĐỜI ĐẶT HÀNG (TRẠNG THÁI)

### 3.1 Các trạng thái

| Trạng thái | Ý nghĩa | Hành động tiếp theo |
|---|---|---|
| **Phiếu tạm** | Mới tạo, chưa xử lý. Có thể chỉnh sửa tự do (hàng hóa, khách, shipping) | Xử lý đơn hàng → chuyển sang Đang giao; hoặc Hủy |
| **Đang giao hàng** | Đã xử lý → đã tạo Hóa đơn + vận đơn gửi hãng VC | Đã giao → Hoàn thành; hoặc Chuyển hoàn → tạo Trả hàng tự động |
| **Hoàn thành** | Khách đã nhận hàng, giao dịch khép lại | Không thay đổi |
| **Đã hủy** | Phiếu bị hủy | Bất biến, hiển thị trong danh sách khi lọc |
| **(+1 trạng thái khác)** | Hiển thị trên bộ lọc nhưng chưa xác nhận rõ tên — có thể là "Tạm dừng" hoặc "Chờ xác nhận" | Cần làm rõ |

### 3.2 Luồng chuyển trạng thái

- **Tạo mới:** Phiếu tạm
- **Xử lý đơn hàng** (từ Phiếu tạm): Mở màn bán hàng, nhân viên thu ngân xác nhận → tạo Hóa đơn → chuyển sang Đang giao hàng
- **Đã giao** (từ Đang giao hàng): Xác nhận khách nhận hàng → Hoàn thành
- **Chuyển hoàn** (từ Đang giao hàng): Hãng VC báo hoàn → hệ thống tự tạo Phiếu trả hàng → Hoàn thành (trừ hàng đã trả)
- **Hủy:** Từ bất kỳ trạng thái chưa hoàn thành — hỏi nghiệp vụ giữ/hủy phiếu thu cọc liên quan

### 3.3 Tác động tồn kho theo trạng thái

| Trạng thái | Tồn thực tế (xuất kho) | Tồn đặt giữ cho khách | Doanh thu |
|---|---|---|---|
| Phiếu tạm | Không thay đổi | Tăng theo số lượng đặt | Chưa ghi nhận |
| Đang giao hàng | Giảm (qua Hóa đơn) | Chuyển sang Hóa đơn | Ghi nhận vào Hóa đơn |
| Hoàn thành | Không thay đổi thêm | Không thay đổi thêm | Đã ghi nhận từ bước trên |
| Đã hủy | Hoàn lại nếu đã tạo HD (qua TH) | Giảm về 0 | Hoàn lại nếu có |

→ **Đặt hàng chỉ là đặt giữ mềm.** Tồn kho thực tế chỉ giảm khi chuyển sang Hóa đơn.

---

## 4. DANH SÁCH ĐẶT HÀNG

### 4.1 Bộ lọc (10 tiêu chí)

| Bộ lọc | Kiểu | Ghi chú |
|---|---|---|
| Chi nhánh xử lý | Chọn nhiều | Tách riêng theo chi nhánh |
| Thời gian tạo đơn | Chọn khoảng thời gian | Hôm nay, Hôm qua, 7/30 ngày, Tuần, Tháng, Quý, Năm — kể cả âm lịch. **Không có "Toàn thời gian"** |
| Trạng thái | Chọn nhiều | 5 trạng thái |
| Đối tác giao hàng | Dropdown | Lọc theo hãng VC |
| **Thời gian giao hàng** | Chọn khoảng thời gian | Đặc biệt có **Ngày mai, Tuần sau, Tháng tới, Quý sau, Toàn thời gian** — phục vụ lập kế hoạch giao đơn tương lai (đặt trước sự kiện, pre-order) |
| Khu vực giao hàng | Tỉnh/TP → Quận/Huyện | Lập kế hoạch giao theo vùng |
| Phương thức thanh toán | Chọn nhiều | Tiền mặt / Thẻ / QR |
| VAT (%) | Dropdown | Đối soát thuế |
| Người tạo | Dropdown nhân viên | Kiểm tra, audit |
| Người nhận đặt | Dropdown nhân viên | Xem danh mục theo nhân viên phụ trách |

**Điểm đặc trưng:** Bộ lọc "Thời gian giao hàng" hỗ trợ khoảng thời gian **tương lai** (Ngày mai, Tuần sau...) — tính năng hiếm có so với các module khác — phục vụ nghiệp vụ lập kế hoạch giao đơn trước.

### 4.2 Các hành động trên danh sách

| Hành động | Mô tả |
|---|---|
| Tạo đặt hàng mới | Mở màn POS dạng đặt hàng để nhập đơn |
| Đồng bộ đơn online | Kéo đơn từ sàn thương mại điện tử (Shopee, TikTok Shop, Lazada...) về. Hiển thị badge số đơn chờ đồng bộ. |
| Gộp đơn | Chọn nhiều phiếu của cùng một khách → gộp thành 1 phiếu để giao 1 chuyến, tiết kiệm phí vận chuyển |
| Xuất file | Xuất Excel danh sách đã lọc |
| Xử lý hàng loạt | Xử lý / hủy nhiều đơn cùng lúc |
| Tùy chỉnh cột | Chọn cột hiển thị trên bảng danh sách |

### 4.3 Cột mặc định trong bảng danh sách

| Cột | Ghi chú |
|---|---|
| Mã đặt hàng | DH + 6 chữ số |
| Thời gian | Ngày đặt |
| Mã khách hàng | Nhấp vào để mở hồ sơ khách |
| Tên khách hàng | |
| Tổng tiền hàng | Tổng trước khi tính phí và VAT |
| Khách cần trả | Tổng sau VAT, phí ship, thu khác |
| Khách đã trả | Số cọc đã thu |
| Trạng thái | Hiển thị nhãn màu: Phiếu tạm (cam), Đang giao (xanh dương), Hoàn thành (xanh lá), Đã hủy (đỏ) |

Dòng cuối bảng hiển thị tổng cộng các cột số.

---

## 5. CHI TIẾT MỘT PHIẾU ĐẶT HÀNG

### 5.1 Xem nhanh (Inline Panel)

Nhấp vào một hàng trong danh sách → mở panel chi tiết ngay bên dưới, không cần chuyển trang. Cho phép chỉnh sửa nhanh các trường: Người nhận đặt, Ngày đặt, Kênh bán, Bảng giá, Trạng thái.

### 5.2 Thông báo địa chỉ hành chính mới

Nếu địa chỉ giao hàng thuộc khu vực đã sáp nhập/thay đổi sau ngày 01/07/2025, hệ thống hiển thị banner chủ động thông báo và đề xuất cập nhật địa chỉ mới — giống tính năng trên module Khách hàng.

### 5.3 Theo dõi hoàn thành từng dòng hàng

Cột "Số lượng" trên mỗi dòng hàng hiển thị theo dạng **đã đặt / đã xuất**. Ví dụ:
- "1/0" = đặt 1, chưa xuất cái nào
- "1/1" = đặt 1, đã xuất đủ qua Hóa đơn

Khi phiếu được xử lý sang Hóa đơn, số đã xuất tự động cập nhật.

### 5.4 Hành động trên chi tiết phiếu

| Hành động | Mô tả |
|---|---|
| Xử lý đơn hàng | Hành động chính — mở màn bán hàng với giỏ hàng điền sẵn từ đơn đặt. Nhân viên thu ngân xác nhận → tạo Hóa đơn |
| Hủy phiếu | Hủy đơn — hỏi có giữ hay hủy phiếu thu cọc liên quan |
| Sao chép | Tạo phiếu mới với toàn bộ dữ liệu copy (hàng hóa, khách, thông tin giao hàng) |
| Lưu | Lưu chỉnh sửa trong panel |
| Xuất file | Xuất Excel chi tiết phiếu |
| In | In phiếu đặt hàng |

---

## 6. NGHIỆP VỤ ĐẶC BIỆT

### 6.1 Đồng bộ đơn từ sàn thương mại điện tử

Khi kết nối kênh bán online (Shopee, TikTok Shop, Lazada, Tiki, Facebook Shop, Zalo Shop), đơn đặt hàng từ các sàn được kéo về và tự động tạo phiếu DH trong KiotViet với:
- Kênh bán = sàn nguồn
- Thông tin giao hàng lấy từ sàn
- Sản phẩm map theo cấu hình liên kết hàng hóa đã thiết lập trước

Hiện tại đồng bộ theo cơ chế **kéo thủ công** (nhân viên bấm nút), không tự động đẩy realtime.

### 6.2 Gộp đơn (Merge Orders)

Cho phép chọn nhiều phiếu đặt hàng của cùng một khách rồi gộp thành 1 phiếu duy nhất.

**Nghiệp vụ điển hình:** Khách đặt 3 lần trong vòng 30 phút → nhân viên gộp lại để giao 1 chuyến, tránh mất phí ship cho 3 vận đơn riêng lẻ.

### 6.3 Tích hợp hãng vận chuyển

Dropdown chọn hãng vận chuyển chia làm 2 nhóm:

**Tích hợp API (có nhãn "KiotViet"):** Grab, GHN, Viettel Post, GHTK, J&T Express, AhaMove...
- Tự động tạo vận đơn trên hệ thống hãng VC
- Đồng bộ trạng thái 2 chiều: Đang giao → Đã giao → Chuyển hoàn
- Lấy phí ship thực tế theo thời gian thực

**Ghi nhận thủ công (không có nhãn):** Nhân viên tự điền mã vận đơn và cập nhật trạng thái.

→ Đây là lợi thế cạnh tranh lớn của KiotViet — tích hợp sẵn 6-8 hãng vận chuyển Việt Nam trực tiếp trong form đặt hàng.

### 6.4 Tự động tạo Trả hàng khi chuyển hoàn

Khi hãng vận chuyển tích hợp API báo trạng thái "Đã chuyển hoàn":
- Hệ thống **tự động tạo Phiếu trả hàng** cho các mặt hàng bị hoàn
- Phiếu DH gốc chuyển sang trạng thái Hoàn thành (phần còn lại nếu có)
- Phiếu trả hàng tự động này **không cho phép hủy** do đồng bộ chặt với hãng VC

---

## 7. ĐIỂM CẦN CẢI TIẾN

### 7.1 Quy trình & Trạng thái

| # | Vấn đề | Đề xuất |
|---|---|---|
| 1 | Trạng thái "+1 khác" trong bộ lọc không rõ tên — gây nhầm lẫn cho người dùng | Hiển thị đầy đủ tên tất cả trạng thái |
| 2 | Không có trạng thái **"Đã xác nhận"** giữa Phiếu tạm và Đang giao — nhân viên chưa kịp xác nhận khách đã chắc chắn mua | Thêm trạng thái "Xác nhận" |
| 3 | Không hỗ trợ **giao một phần** thực sự — đặt 10 cái, giao 7 trước, 3 sau không được tách | Cho phép tách đơn thành nhiều đợt giao |
| 4 | Hủy đơn không có **lý do cụ thể** — chỉ ghi chú tự do | Thêm danh mục lý do: KH đổi ý / Hết hàng / Giá sai / Khác |
| 5 | Không có **cảnh báo sắp trễ hẹn giao** | Theo dõi SLA + cảnh báo tự động |

### 7.2 Giá & Thuế

| # | Vấn đề | Đề xuất |
|---|---|---|
| 6 | Nhiều cột giá (trước/sau thuế) làm bảng rất rộng, khó dùng trên màn nhỏ | Nút ẩn nhanh cột thuế khi không cần |
| 7 | "Thu khác" tự định nghĩa không có quy tắc tự động áp dụng | Cấu hình quy tắc tự áp (ví dụ: đơn > 200k thì tự cộng 1k phí đóng gói) |
| 8 | Giảm giá phiếu không hỗ trợ **cộng nhiều ưu đãi** (VIP + voucher + giảm thêm) | Phân cấp ưu đãi với độ ưu tiên |
| 9 | Thuế chỉ áp 1 mức/phiếu — đơn có lẫn hàng VAT 0%/5%/10% phải xử lý thủ công | Thuế theo từng dòng hàng |

### 7.3 Giao hàng & Vận chuyển

| # | Vấn đề | Đề xuất |
|---|---|---|
| 10 | Chỉ 1 vận đơn / đơn đặt — đơn lớn nhiều thùng phải tách đơn tay | Hỗ trợ nhiều kiện hàng / đơn |
| 11 | Phí thu khách và phí trả hãng VC dễ nhầm lẫn | Biểu đồ giải thích trực quan |
| 12 | Không có **tự động chọn hãng VC** theo khu vực / trọng lượng / giá | Gợi ý hãng tối ưu chi phí-tốc độ |
| 13 | Không tạo vận đơn hàng loạt cho nhiều đơn cùng lúc | Tạo vận đơn bulk từ danh sách |
| 14 | Không so sánh phí giữa nhiều hãng cho cùng một đơn | Hiển thị báo giá realtime nhiều hãng |

### 7.4 Đồng bộ kênh online

| # | Vấn đề | Đề xuất |
|---|---|---|
| 15 | Đồng bộ thủ công qua nút bấm — không realtime | Nhận đẩy (webhook) từ sàn thay vì kéo thủ công |
| 16 | Map sản phẩm có thể sai → đơn tạo ra nhưng sản phẩm không khớp | Gợi ý map mờ + cảnh báo review thủ công |
| 17 | Không có xử lý xung đột khi đơn trên sàn bị cập nhật sau khi đã sửa trên KV | Quy trình merge/ghi đè |

### 7.5 Gộp đơn

| # | Vấn đề | Đề xuất |
|---|---|---|
| 18 | Gộp đơn không có bước xem trước kết quả | Hiển thị preview "đơn mới sẽ thế này" trước khi xác nhận |
| 19 | Không có quy tắc **tự động gộp** (cùng khách + cùng địa chỉ + trong vòng X giờ) | Cấu hình auto-merge |

---

## 8. CƠ HỘI ĐỘT PHÁ — TOP 5

| # | Tính năng | Giải quyết vấn đề | Nỗ lực | Tác động |
|---|---|---|---|---|
| 1 | **Tự động chọn hãng VC tối ưu** — so sánh và chọn hãng rẻ/nhanh nhất cho từng đơn | #12, #14 | Cao | Rất cao |
| 2 | **Giao hàng nhiều đợt** — tách 1 đơn thành nhiều lô giao | #3, #10 | Trung bình | Cao (B2B, đơn lớn) |
| 3 | **Xác nhận đơn + Gửi ZNS tự động** — thêm bước xác nhận trước khi giao, tự động nhắn khách qua ZNS | #2, #5 | Thấp | Cao |
| 4 | **Cộng nhiều ưu đãi** — nhiều tầng giảm giá với độ ưu tiên | #8 | Cao | Cao |
| 5 | **Đồng bộ realtime từ sàn** — nhận webhook thay vì kéo thủ công | #15 | Trung bình | Rất cao (seller đa kênh) |

---

## 9. CÁC ĐỐI TƯỢNG NGHIỆP VỤ (ENTITIES)

| # | Đối tượng | Vai trò |
|---|---|---|
| 69 | Phiếu đặt hàng | Đầu phiếu: liên kết khách hàng, thông tin giá, thanh toán, giao hàng |
| 70 | Dòng hàng | Mỗi sản phẩm trên phiếu: số lượng đặt/xuất, giá trước và sau thuế |
| 71 | Thông tin giao hàng | Địa chỉ, hãng VC, COD, kích thước kiện, khai giá |
| 72 | Phụ phí tùy chỉnh | Loại "Thu khác" do người dùng tự định nghĩa |
| 73 | Theo dõi hoàn thành | Số lượng đặt vs số lượng đã xuất trên từng dòng |
| 74 | Tích hợp vận chuyển | Hãng VC + trạng thái kết nối API |
| 75 | Lịch sử gộp đơn | Ghi nhận thao tác gộp nhiều DH thành 1 để audit |

---

## 10. SO SÁNH VỚI CHUẨN THƯƠNG MẠI ĐIỆN TỬ

| Tính năng | KiotViet DH | Shopify Order | WooCommerce Order |
|---|---|---|---|
| Tách biệt Order và Invoice | Có | Có | Một phần |
| Nhiều trạng thái workflow | Cơ bản (4-5 trạng thái) | Đầy đủ (pending → confirmed → fulfilled → completed) | Đầy đủ |
| Theo dõi hoàn thành từng dòng | Có UI nhưng chưa hỗ trợ nhiều lô giao | Có, hỗ trợ multi-fulfillment | Có |
| Tích hợp hãng vận chuyển | Mạnh — 6-8 hãng VN tích hợp sẵn | Qua ứng dụng | Qua plugin |
| Đồng bộ sàn TMĐT | Có — kéo thủ công + map sản phẩm | Có — Shopify Markets | Qua plugin |
| Đa tiền tệ | Không | Có | Có |
| Đặt hàng định kỳ / tự động | Không | Qua ứng dụng | Qua plugin |
| Đặt trước (backorder) | Không (chỉ cho phép âm tồn) | Có — Pre-order | Có |

→ KiotViet **mạnh ở thị trường trong nước** (hãng vận chuyển VN, hóa đơn điện tử, địa danh hành chính sau 01/07/2025) nhưng **còn hạn chế ở tính năng thương mại điện tử nâng cao** (đặt hàng định kỳ, đa tiền tệ, giao hàng nhiều lô).

---

## 11. TÓM LƯỢC

**Đặt hàng KiotViet** là lớp cam kết mềm giữa ý định mua và giao dịch thực tế:

- **Tách biệt rõ ràng** Đặt hàng (DH) và Hóa đơn (HD) — đúng chuẩn Order vs Invoice của thương mại điện tử
- **Thông tin đầy đủ:** Giá trước/sau thuế trên từng dòng hàng, giao hàng với tích hợp hãng VC, theo dõi số lượng đặt/xuất
- **Vòng đời 4-5 trạng thái:** Phiếu tạm → Đang giao → Hoàn thành; nhánh Hủy và tự động Chuyển hoàn
- **Điểm mạnh:** Tích hợp 6-8 hãng vận chuyển Việt Nam, bộ lọc thời gian giao **tương lai** (Ngày mai/Tuần sau/Quý sau), đồng bộ 6 sàn TMĐT, Gộp đơn, Tự động trả hàng khi chuyển hoàn, Cảnh báo địa danh hành chính mới
- **Điểm yếu:** Không có trạng thái xác nhận đơn, không hỗ trợ giao nhiều đợt, không có cộng nhiều ưu đãi, không tự động chọn hãng VC tối ưu, đồng bộ sàn là kéo thủ công chứ không phải push realtime
- **Cơ hội:** Tự động chọn hãng VC, Giao nhiều lô, Xác nhận đơn + ZNS tự động, Cộng ưu đãi nhiều tầng, Đồng bộ realtime từ sàn
