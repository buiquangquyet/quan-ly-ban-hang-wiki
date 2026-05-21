# Tổng quan nghiệp vụ — Sổ quỹ

---

## 1. Định nghĩa & phạm vi

Sổ quỹ ghi nhận **tất cả dòng tiền thực tế** vào và ra khỏi cửa hàng dưới dạng phiếu thu hoặc phiếu chi. Mọi giao dịch phát sinh tiền — dù từ bán hàng, mua hàng, trả hàng, hay chi phí vận hành — đều bắt buộc đi qua sổ quỹ. Nhờ vậy, sổ quỹ là nguồn dữ liệu duy nhất phản ánh vị thế tiền mặt (cash position) của cửa hàng tại bất kỳ thời điểm nào.

Phạm vi của sổ quỹ là **lớp vận hành** (operational layer): phản ánh tiền đã thực sự trao tay. Sổ quỹ không thay thế kế toán kép; dữ liệu từ sổ quỹ là đầu vào cho module Thuế & Kế toán xử lý bút toán Nợ/Có.

---

## 2. Vị trí trong hệ sinh thái

Sổ quỹ là điểm hội tụ của sáu module có phát sinh tiền:

| Nghiệp vụ | Module nguồn | Loại chứng từ phát sinh |
|---|---|---|
| Khách hàng thanh toán hóa đơn | Bán hàng (HD) | Phiếu thu — TTHD |
| Khách hàng đặt cọc đơn hàng | Đặt hàng (DH) | Phiếu thu — TTDH |
| Hoàn tiền khi khách trả hàng | Trả hàng (TH) | Phiếu chi — TTTH |
| Thanh toán tiền hàng cho nhà cung cấp | Nhập hàng (PN) | Phiếu chi |
| Nhà cung cấp hoàn tiền khi trả hàng nhập | Trả hàng nhập (TPN) | Phiếu thu |
| Chi phí vận hành, lương, thu khác | Thủ công | Phiếu thu / Phiếu chi |

Ngoài ra, sổ quỹ cập nhật đồng thời công nợ của đối tác liên quan: khi ghi nhận phiếu thu từ khách hàng, dư nợ của khách hàng đó giảm; khi ghi nhận phiếu chi trả nhà cung cấp, dư nợ của nhà cung cấp đó giảm tương ứng.

---

## 3. Cấu trúc nghiệp vụ

### 3.1 Các loại quỹ

Cửa hàng quản lý tiền theo bốn hình thức:

| Loại quỹ | Mô tả | Ví dụ sử dụng |
|---|---|---|
| Tiền mặt | Tiền vật lý tại quầy thu ngân | Thu tiền mặt khi bán hàng tại cửa hàng |
| Ngân hàng | Tài khoản thanh toán tại ngân hàng | Nhận chuyển khoản, thanh toán QR VietQR |
| Ví điện tử | Ví MoMo, ZaloPay, VNPay và tương đương | Nhận thanh toán qua ứng dụng ví |
| Tổng quỹ | Tổng hợp của cả ba loại trên | Xem tổng quan toàn bộ dòng tiền |

Một cửa hàng có thể có nhiều tài khoản ngân hàng và nhiều ví điện tử. Mỗi tài khoản hoặc ví được quản lý như một tài khoản quỹ riêng. View "Tổng quỹ" tổng hợp số dư của tất cả tài khoản quỹ.

### 3.2 Chứng từ thu chi (Phiếu thu / Phiếu chi)

Mỗi dòng tiền được ghi nhận bằng một chứng từ — gọi là phiếu thu (tiền vào) hoặc phiếu chi (tiền ra). Một phiếu thu/chi mang các thông tin nghiệp vụ sau:

- **Loại chứng từ:** phiếu thu hoặc phiếu chi
- **Mã chứng từ:** định danh duy nhất theo hệ thống prefix (xem mục 3.3)
- **Loại thu chi:** danh mục phân loại giao dịch (xem mục 4)
- **Loại quỹ & tài khoản quỹ:** tiền thuộc quỹ nào, tài khoản cụ thể nào
- **Ngày ghi sổ:** thời điểm giao dịch được xác nhận
- **Giá trị:** số tiền (luôn dương; chiều vào/ra xác định bởi loại chứng từ)
- **Đối tác:** khách hàng, nhà cung cấp, nhân viên, hoặc đối tác khác liên quan
- **Chứng từ nguồn:** hóa đơn, đơn hàng, phiếu nhập... gốc phát sinh giao dịch này (nếu là auto-gen)
- **Flag "Hạch toán KQKD":** giao dịch có ảnh hưởng lãi/lỗ hay không (xem mục 4.4)
- **Ghi chú:** thông tin bổ sung
- **Trạng thái:** Đã thanh toán hoặc Đã hủy

### 3.3 Hệ thống mã chứng từ

Mã chứng từ theo quy tắc: **[PREFIX] + 6 chữ số tuần tự**.

| Prefix | Tên nghiệp vụ | Chiều | Phát sinh từ |
|---|---|---|---|
| TTHD | Thu tiền hóa đơn | Thu | Khách hàng thanh toán hóa đơn bán hàng |
| TTDH | Thu tiền đặt hàng | Thu | Khách hàng đặt cọc đơn hàng |
| TTTH | Chi tiền trả khách | Chi | Hoàn tiền khi khách trả hàng |
| CNH | Chuyển ngân hàng / quỹ | Chi | Chuyển tiền nội bộ giữa các quỹ |
| TTD_CNH | Đối ứng chuyển quỹ | Thu | Phiếu nhận tiền đối ứng với CNH |

**Lưu ý quan trọng:** Số thứ tự trong mã TTHD dùng chung dãy với mã hóa đơn — ví dụ TTHD016157 tương ứng với HD016157. Điều này giúp tra cứu nhanh phiếu thu từ hóa đơn gốc mà không cần tìm kiếm.

Phiếu chuyển quỹ (CNH) luôn xuất hiện thành **cặp đối ứng**: một phiếu chi CNH và một phiếu thu TTD_CNH có cùng thời điểm và cùng giá trị — net effect bằng 0 với tổng quỹ.

---

## 4. Phân loại giao dịch thu chi

### 4.1 Giao dịch thu (phiếu thu)

| Loại thu | Nguồn phát sinh | Ghi chú |
|---|---|---|
| Thu tiền khách trả hóa đơn | Tự động từ module Bán hàng | Phát sinh khi khách thanh toán HĐ |
| Thu cọc đặt hàng | Tự động từ module Đặt hàng | Cọc chưa phải doanh thu đã hoàn thành |
| Thu hoàn tiền từ nhà cung cấp | Tự động từ module Trả hàng nhập | NCC trả lại tiền khi cửa hàng trả hàng nhập |
| Thu khác | Thủ công | Thu nhập ngoài nghiệp vụ chính |

### 4.2 Giao dịch chi (phiếu chi)

| Loại chi | Nguồn phát sinh | Ghi chú |
|---|---|---|
| Chi hoàn tiền khách | Tự động từ module Trả hàng | Phát sinh khi khách trả hàng và nhận lại tiền |
| Chi trả nhà cung cấp | Tự động từ module Nhập hàng | Thanh toán tiền hàng cho NCC |
| Chi lương | Thủ công | Trả lương, tạm ứng nhân viên |
| Chi vận hành | Thủ công | Điện, nước, thuê mặt bằng, quảng cáo, ship... |
| Chi khác | Thủ công | Chi phí ngoài các danh mục trên |

### 4.3 Giao dịch chuyển quỹ nội bộ (CNH)

Chuyển quỹ là việc di chuyển tiền giữa các quỹ trong cùng cửa hàng — ví dụ chuyển từ quỹ tiền mặt sang tài khoản ngân hàng, hoặc rút từ ngân hàng về tiền mặt. Giao dịch này:

- Tạo hai phiếu đối ứng đồng thời (một phiếu chi tại quỹ nguồn, một phiếu thu tại quỹ đích)
- **Không làm thay đổi tổng quỹ** — chỉ thay đổi cơ cấu giữa các loại quỹ
- **Không hạch toán KQKD** — không ảnh hưởng lãi/lỗ

### 4.4 Flag "Hạch toán KQKD" — tách cash flow khỏi P&L

Đây là một thiết kế nghiệp vụ đáng chú ý: mỗi phiếu thu/chi được đánh dấu xem có ảnh hưởng đến kết quả kinh doanh hay không.

| Giá trị | Ý nghĩa | Ví dụ |
|---|---|---|
| Có | Giao dịch ảnh hưởng lãi/lỗ (doanh thu hoặc chi phí thực) | Thu từ bán hàng, chi lương, chi vận hành, chi hoàn tiền khách |
| Không | Giao dịch chỉ là dòng tiền, không tính vào lãi/lỗ | Chuyển quỹ nội bộ, chủ vay/rút vốn, ứng/hoàn ứng nhân viên |

Nhờ flag này, cùng một màn hình sổ quỹ, người dùng có thể lọc để xem:
- Toàn bộ dòng tiền thực tế (cash flow view)
- Chỉ các giao dịch ảnh hưởng kinh doanh (P&L view sơ bộ)

---

## 5. Vòng đời chứng từ

### 5.1 Hai trạng thái

Phiếu thu/chi chỉ có hai trạng thái:

- **Đã thanh toán** — trạng thái ngay khi tạo, giao dịch đã được ghi vào sổ quỹ
- **Đã hủy** — khi cần hủy bỏ chứng từ; phiếu bị hủy vẫn hiển thị trong danh sách khi lọc

### 5.2 Không có trạng thái "Phiếu tạm"

Khác với phiếu nhập hàng (cần kiểm đếm hàng vật lý trước khi xác nhận), sổ quỹ **không có bước nháp**. Phiếu thu/chi được xác nhận ngay khi tạo vì bản chất của giao dịch tiền là đã trao tay — không có trạng thái "đang trao".

### 5.3 Hiệu ứng tức thì khi tạo phiếu

Khi một phiếu thu/chi được tạo, hệ thống cập nhật đồng thời:

1. **Số dư quỹ** — tồn quỹ tăng hoặc giảm theo giá trị phiếu
2. **Dư nợ đối tác** — công nợ của khách hàng hoặc nhà cung cấp liên quan được cập nhật
3. **Số tiền đã thanh toán trên chứng từ nguồn** — ví dụ hóa đơn gốc ghi nhận thêm phần đã thu

Khi một phiếu bị hủy, các cập nhật trên được đảo ngược.

---

## 6. Quan hệ với chuẩn kế toán

Sổ quỹ KiotViet tương đương về mục đích với **Sổ quỹ tiền mặt (Mẫu S07-DN)** và **Sổ tiền gửi ngân hàng (Mẫu S08-DN)** theo Thông tư 200/2014/TT-BTC — nhưng đơn giản hóa để phù hợp với cửa hàng bán lẻ vừa và nhỏ.

| Khía cạnh | Sổ quỹ KiotViet | Kế toán chuẩn TT200 |
|---|---|---|
| Phương pháp ghi sổ | Ghi đơn (chỉ ghi một chiều vào/ra) | Ghi kép (Nợ/Có trên hai tài khoản đối ứng) |
| Tài khoản kế toán | Không có (dùng flag KQKD thay thế) | Đầy đủ TK 111, 112, 131, 331... |
| Audit trail | Có — mã chứng từ không đổi sau khi tạo | Có |
| Đối chiếu sao kê ngân hàng | Chưa có tính năng tự động | Bắt buộc định kỳ |
| Đa ngoại tệ | Chưa hỗ trợ | Có |

Sổ quỹ là **lớp vận hành** cung cấp dữ liệu đầu vào cho module Thuế & Kế toán xử lý bút toán đầy đủ — hai module này bổ sung cho nhau, không thay thế nhau.

---

## 7. Điểm mạnh của thiết kế hiện tại

- **Tách bạch cash flow và P&L trên cùng một màn hình** nhờ flag "Hạch toán KQKD" — người dùng không cần hai module riêng biệt để hiểu dòng tiền và lãi lỗ
- **Liên thông tự động với 5 module** — phiếu thu chi được tạo tự động khi có giao dịch từ Bán hàng, Đặt hàng, Trả hàng, Nhập hàng, Trả hàng nhập; không phải nhập tay
- **Hỗ trợ cả dương lịch và âm lịch** trong bộ lọc thời gian — phù hợp với cửa hàng Việt theo mùa vụ lễ tết
- **Quản lý đa quỹ** — tiền mặt, nhiều tài khoản ngân hàng, nhiều ví điện tử trong cùng một giao diện
- **Nhận thanh toán QR realtime** qua VietQR — phiếu thu tự động ghi nhận khi khách quét mã

---

## 8. Hạn chế & cơ hội cải tiến

### 8.1 Nhập liệu & trải nghiệm vận hành

| # | Hạn chế hiện tại | Hướng cải tiến |
|---|---|---|
| 1 | Tạo phiếu chi nhỏ lẻ thủ công nhiều lần mỗi ngày (F&B, salon: 10–20 phiếu/ngày) | Templates 1-click cho các loại chi thường gặp |
| 2 | Không thể nhập loạt phiếu từ file Excel | Luồng import tương tự module Hóa đơn |
| 3 | Không thể chụp ảnh hóa đơn giấy để tự động tạo phiếu chi | Nhận dạng hóa đơn bằng OCR rồi tự điền phiếu chi |
| 4 | Ứng dụng di động chưa có lối tắt vào Sổ quỹ | Widget "Hôm nay thu/chi bao nhiêu?" trên màn hình chính app |

### 8.2 Liên thông & đối soát

| # | Hạn chế hiện tại | Hướng cải tiến |
|---|---|---|
| 5 | Không có đối chiếu tự động với sao kê ngân hàng (chỉ nhận QR một chiều) | Kéo sao kê ngân hàng và tự ghép với phiếu thu/chi |
| 6 | Thanh toán NCC chia nhiều lần sinh nhiều phiếu rời rạc, không có lịch trả tổng hợp | Lịch thanh toán nhà cung cấp theo lô |
| 7 | Thanh toán qua MoMo/ZaloPay chưa tự động vào đúng quỹ "Ví điện tử" tương ứng | Định tuyến tự động theo kênh thanh toán |
| 8 | Chuyển quỹ tạo hai phiếu đối ứng nhưng giao diện không hiển thị liên kết giữa hai phiếu | Hiển thị "phiếu đối ứng" trực tiếp trên trang chi tiết |

### 8.3 Báo cáo & phân tích

| # | Hạn chế hiện tại | Hướng cải tiến |
|---|---|---|
| 9 | Không có biểu đồ dòng tiền theo ngày/tuần/tháng | Biểu đồ cash flow tích hợp ngay trên màn hình sổ quỹ |
| 10 | Không có báo cáo lãi/lỗ từ các phiếu có flag "Hạch toán KQKD = Có" | Dashboard bóc tách doanh thu và chi phí theo danh mục |
| 11 | Không có dự báo dòng tiền — chủ shop không biết tuần/tháng tới sẽ thiếu/thừa tiền | Dự báo cash flow 7–30 ngày từ đơn hàng chưa thu và lịch trả NCC |
| 12 | Không tính được "runway" — số ngày cửa hàng tồn tại được nếu doanh thu bằng 0 | KPI runway = tồn quỹ hiện tại / chi phí bình quân ngày |
| 13 | Không có so sánh kỳ này với kỳ trước | Toggle so sánh tháng/quý/năm cùng kỳ |

### 8.4 Phân quyền & kiểm soát nội bộ

| # | Hạn chế hiện tại | Hướng cải tiến |
|---|---|---|
| 14 | Bất kỳ nhân viên có quyền đều tạo được phiếu chi không giới hạn giá trị | Luồng duyệt: phiếu chi vượt ngưỡng cần quản lý phê duyệt |
| 15 | Không có lịch sử thay đổi chi tiết theo từng trường — không biết ai sửa, sửa gì | Lịch sử thay đổi đầy đủ theo từng trường dữ liệu |
| 16 | Khi hủy phiếu không bắt buộc ghi lý do | Danh mục lý do hủy: Nhập sai / Giao dịch lỗi / Hủy theo yêu cầu KH / Khác |
| 17 | Không có quy trình quỹ tạm ứng cho nhân viên (petty cash) | Module quỹ tạm ứng với hạn mức và duyệt hoàn ứng |
| 18 | Không có cảnh báo khi tồn quỹ tiền mặt vượt ngưỡng an toàn | Cài đặt ngưỡng tiền mặt tối đa và nhận thông báo khi vượt |

---

## 9. Cơ hội ưu tiên cao

| # | Tính năng | Giải quyết vấn đề | Độ phức tạp | Tác động |
|---|---|---|---|---|
| 1 | Đối chiếu sao kê ngân hàng tự động | Hạn chế #5 — đau nhất với shop nhiều TK NH | Cao | Rất cao |
| 2 | Dashboard lãi/lỗ theo danh mục | Hạn chế #10 — thông tin chủ shop cần hàng ngày | Trung | Rất cao |
| 3 | Nhận dạng hóa đơn giấy bằng OCR | Hạn chế #3 — giảm 50% thời gian nhập phiếu chi nhỏ | Trung | Cao |
| 4 | Dự báo dòng tiền 7–30 ngày | Hạn chế #11 — quản lý tiền chủ động thay vì bị động | Cao | Cao |
| 5 | Luồng duyệt phiếu chi theo ngưỡng | Hạn chế #14 — bắt buộc cho chuỗi nhiều chi nhánh | Trung | Cao (phân khúc chuỗi) |

---

## 10. Tóm lược

Sổ quỹ KiotViet là một operational cash ledger hoàn thiện ở lớp cơ bản: liên thông tự động với 5 module nghiệp vụ, quản lý đa quỹ, và có thiết kế tinh tế khi tách bạch cash flow khỏi P&L ngay trên cùng một màn hình thông qua flag "Hạch toán KQKD".

Điểm yếu tập trung ở ba nhóm: thiếu đối chiếu với ngân hàng (reconciliation), thiếu công cụ phân tích và dự báo (intelligence), và thiếu kiểm soát nội bộ (approval, audit). Đây cũng là ba hướng phát triển chính để nâng sổ quỹ từ công cụ ghi chép thành công cụ quản lý tài chính thực sự cho chủ shop.
