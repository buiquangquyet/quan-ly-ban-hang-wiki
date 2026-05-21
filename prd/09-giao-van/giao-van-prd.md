# PRD — NGHIỆP VỤ GIAO VẬN

**Phạm vi:** 2 module song hành tạo thành **lớp giao vận**:
- **Đối tác giao hàng** — master data + đối soát tài chính với hãng vận chuyển
- **Vận đơn** — từng đơn vận chuyển cụ thể, theo dõi trạng thái thời gian thực

**Vị trí trong hệ thống:** Menu **Đơn hàng → Đối tác giao hàng** / **Vận đơn**

---

## 1. KHÁI NIỆM & MỐI QUAN HỆ

```
Hóa đơn "Giao hàng" hoặc Đặt hàng "Tự giao đến"
                    │
                    ▼
        Chọn Đối tác giao hàng + cấu hình vận chuyển
                    │
                    ▼
           Tạo Vận đơn (gửi yêu cầu đến hãng)
           → Hãng sinh mã vận đơn
                    │
                    ▼
        Hãng đồng bộ trạng thái 2 chiều
        (Chờ xử lý → Lấy hàng → Giao hàng
         → Giao thành công / Chuyển hoàn / Đã hủy)
                    │
        (cuối kỳ)   ▼
           Đối soát với Đối tác → Lịch sử đối soát


   [Đối tác giao hàng]               [Tổng hợp tài chính]
   - Tích hợp (15+ hãng)          - Cần thu hộ (COD)
   - Khác (tự quản lý)            - Còn cần thu (COD)
                                  - Tổng phí giao hàng
                                  - Còn cần trả đối tác
```

**Đối tác giao hàng** = master data + tổng hợp tài chính + đối soát kỳ.
**Vận đơn** = chi tiết từng lần vận chuyển.

---

## 2. MODULE "ĐỐI TÁC GIAO HÀNG"

### 2.1 Phân loại Đối tác

Hệ thống phân 2 nhóm đối tác:

| Nhóm | Đặc điểm | Trường hợp sử dụng |
|---|---|---|
| **Tích hợp** | 15+ hãng tích hợp sẵn với hệ thống. Đồng bộ trạng thái 2 chiều, đối soát tự động | Chuỗi / shop chuyên nghiệp, khối lượng cao |
| **Khác** | Đối tác shop tự thêm — tài xế quen, đối tác ad-hoc, shipper riêng | Shop nhỏ, giao nội thành riêng |

### 2.2 Danh sách hãng tích hợp

| Hãng | Ghi chú |
|---|---|
| Be, Be Delivery | App gọi xe + giao hàng |
| BEST Inc, BEST FW | Express; FW = đối soát đơn chuyển hoàn |
| EMS, EMS FW | Bưu chính nhà nước |
| GHN, GHN FW | Giao hàng nhanh — phổ biến nhất |
| GHTK | Giao Hàng Tiết Kiệm |
| Grab | Giao cùng ngày nội thành |
| Green SM | Vingroup (xe điện) |
| J&T Express, J&T FW | Phổ biến trong TMĐT |
| NinjaVan, NinjaVan FW | Khu vực Đông Nam Á |
| AhaMove | Giao cùng ngày nội thành |
| Viettel Post | Bưu chính lớn |
| KShip | Hãng vận chuyển riêng của hệ thống |

> **Quy ước suffix "FW" (Forward):** Mỗi hãng lớn có thêm dòng "FW" để tách bạch đối soát đơn chuyển hoàn khỏi đơn giao thành công — đảm bảo hạch toán chính xác.

### 2.3 Cấu trúc tab tính năng

| Tab | Vai trò nghiệp vụ |
|---|---|
| **Thông tin** | Hồ sơ đối tác + 5 KPI tài chính tổng hợp |
| **Lịch sử giao hàng** | Toàn bộ vận đơn từng đối tác — tra cứu từ đối tác xuống chi tiết |
| **Lịch sử đối soát** | Kỳ đối soát với hãng — bảng đối chiếu, chấp nhận/từ chối từng dòng |

### 2.4 5 KPI tài chính per đối tác (tab Thông tin)

| Trường | Ý nghĩa nghiệp vụ |
|---|---|
| **Tổng đơn hàng** | Tổng số vận đơn đã gửi qua đối tác |
| **Cần thu hộ (COD)** | Tổng tiền đối tác phải thu hộ shop từ khách hàng |
| **Còn cần thu (COD)** | COD đối tác đang giữ, chưa chuyển về shop |
| **Tổng phí giao hàng** | Tổng cước phí shop nợ đối tác |
| **Còn cần trả đối tác** | Phần cước phí shop chưa thanh toán |

> Đây là **hệ thống AR/AP mini với từng hãng vận chuyển** — shop kiểm soát được tiền KH chưa chuyển về và cước chưa thanh toán theo từng đối tác.

### 2.5 Bộ lọc — Đối tác nhóm "Khác"

**Tiêu chí lọc:**
- Nhóm đối tác giao hàng (có thể tạo mới)
- Tổng phí giao hàng (khoảng từ – tới)
- Thời gian (toàn thời gian / tùy chỉnh)
- Nợ hiện tại (khoảng từ – tới)
- Trạng thái (Tất cả / Đang hoạt động / Ngưng hoạt động)

**Thao tác trên danh sách:**
- Tìm kiếm theo mã / tên / số điện thoại
- Thêm đối tác mới
- Nhập file hàng loạt
- Xuất file

**Cột hiển thị:** Mã | Tên | Điện thoại | Tổng đơn hàng | Nợ cần trả hiện tại | Tổng phí giao hàng

> Đối tác nhóm "Khác" có trường **Số điện thoại** để liên hệ tài xế/shipper riêng.

---

## 3. MODULE "VẬN ĐƠN"

### 3.1 Khái niệm

Mỗi vận đơn = **1 lần vận chuyển cụ thể** gắn với 1 hóa đơn "Giao hàng" hoặc đơn hàng "Tự giao đến". Khi hãng tiếp nhận, hệ thống nhận về mã vận đơn do hãng cấp (vd `26944ZAE-1`). Khi chưa kết nối hãng, mã vận đơn để trống.

### 3.2 Bộ lọc danh sách

| Tiêu chí | Mô tả |
|---|---|
| Chi nhánh | Lọc theo chi nhánh xử lý giao hàng |
| Trạng thái giao hàng | 7 trạng thái (xem mục 3.3) |
| Đối tác giao hàng | Lọc theo từng hãng |
| Thời gian tạo | Khi vận đơn được tạo |
| Thời gian hoàn thành | Khi giao thành công |
| Khu vực giao hàng | Tỉnh/Thành phố → Quận/Huyện |
| Thu hộ tiền (COD) | Tất cả / Có / Không |

**Cột hiển thị:** Mã vận đơn | Thời gian tạo | Mã hóa đơn (liên kết) | Mã KH / Khách hàng | Đối tác giao hàng | Trạng thái giao | Thời gian giao hàng

### 3.3 7 Trạng thái vận đơn

| # | Trạng thái | Diễn giải |
|---|---|---|
| 1 | **Chờ xử lý** | Vừa tạo, chưa gửi đến hãng |
| 2 | **Lấy hàng** | Hãng đến shop lấy (sub: Chờ lấy / Đã lấy / Lấy thất bại) |
| 3 | **Giao hàng** | Đang vận chuyển (sub: Đang giao / Giao thất bại) |
| 4 | **Giao thành công** | Khách hàng đã nhận hàng |
| 5 | **Chuyển hoàn** | Hãng đang đem hàng về shop (sub: Đang chuyển hoàn) |
| 6 | **Đã chuyển hoàn** | Hàng về shop — hệ thống tự tạo phiếu Trả hàng |
| 7 | **Đã hủy** | Vận đơn bị hủy (hiển thị lý do qua tooltip) |

> Trạng thái đồng bộ 2 chiều với hãng vận chuyển. Mỗi thay đổi trạng thái tạo 1 bản ghi trong Lịch sử giao hàng.

### 3.4 Chi tiết vận đơn — 3 tab

#### Tab 1 — Thông tin

**Thông tin người nhận:**
- Họ tên / Số điện thoại / Địa chỉ đầy đủ
- Tỉnh/Thành phố – Quận/Huyện – Phường/Xã (theo chuẩn hành chính)

**Thông tin hóa đơn:**
- Mã hóa đơn (liên kết) / Chi nhánh / Khách hàng / Số lượng hàng / Giá trị / Ghi chú

**Thông tin vận chuyển:**

| Trường | Mô tả |
|---|---|
| Mã vận đơn | Do hãng sinh, có nút sao chép |
| Trạng thái giao | Kèm lý do nếu hủy |
| Thời gian tạo / Người tạo | Lịch sử tạo vận đơn |
| Dịch vụ | Giao thường / Siêu tốc ưu tiên / Giao tiết kiệm — tùy hãng |
| Mã đối soát | Mã hãng cấp khi đối soát cuối kỳ |
| Số kiện | Số kiện hàng trong vận đơn (đơn nhiều kiện) |
| Trọng lượng quy đổi | Volumetric weight — dùng tính cước cho hàng cồng kềnh nhẹ |
| Dự kiến giao | SLA hãng cam kết |
| Khai giá | Bảo hiểm hàng (giá trị khai báo) |
| Tự mang ra bưu cục | Shop tự mang đến, không cần hãng đến lấy |
| Bên trả phí | Người gửi hoặc Người nhận |
| Lưu ý khi phát | Ghi chú đặc biệt cho hãng (vd: "Gọi trước khi giao") |
| Mã giảm giá | Voucher hãng áp dụng |

**Thông tin tài chính:**

| Trường | Mô tả |
|---|---|
| Đối tác giao hàng | Hãng vận chuyển |
| Thu hộ tiền (COD) | Số tiền hãng thu hộ từ khách hàng |
| Còn cần thu (COD) | Phần hãng chưa chuyển về shop |
| Tổng cước phí | Cước hãng tính |
| Phí trả đối tác | Phí shop nợ hãng |
| Còn cần trả đối tác | Phần shop chưa thanh toán hãng |
| Người nhận trả phí | Trường hợp khách hàng chịu phí vận chuyển |

#### Tab 2 — Lịch sử giao hàng

Timeline theo dõi vận đơn: mỗi thay đổi trạng thái ghi lại thời điểm và nguồn cập nhật (thủ công hoặc hãng đẩy về).

#### Tab 3 — Yêu cầu hỗ trợ

Luồng xử lý sự cố trực tiếp trên vận đơn:
- Shop tạo yêu cầu hỗ trợ khi gặp sự cố (mất hàng / hỏng / sai địa chỉ / ...)
- Hãng nhận và phản hồi trong tab này
- Theo dõi trạng thái xử lý

> Tính năng này gắn quy trình xử lý sự cố vận chuyển trực tiếp vào vận đơn, không cần dùng kênh hỗ trợ ngoài. Đây là điểm khác biệt so với các phần mềm POS khác.

---

## 4. CÁC NGHIỆP VỤ QUAN TRỌNG

### 4.1 Tạo vận đơn

Vận đơn được tạo theo 3 cách:
1. **Từ hóa đơn "Giao hàng"** trên màn Bán hàng / Hóa đơn — chuẩn nhất, 1 thao tác
2. **Từ đơn hàng "Tự giao đến"** khi xử lý đơn
3. **Tạo trực tiếp** từ menu Vận đơn, chọn hóa đơn liên kết

### 4.2 Đối soát với hãng vận chuyển

Tab "Lịch sử đối soát" trong Đối tác giao hàng:
- Cuối kỳ (tuần / tháng), hãng gửi báo cáo
- Hệ thống đối chiếu từng vận đơn với báo cáo
- Tách 2 chiều:
  - **COD chuyển về shop:** hãng trả tiền thu hộ cho shop
  - **Cước shop trả hãng:** shop thanh toán cước vận chuyển

**Cấn trừ:** nếu COD hãng đang giữ lớn hơn cước nợ → hãng chuyển phần chênh lệch về shop. Ngược lại, shop trả thêm phần thiếu.

### 4.3 Phân tách đối soát đơn chuyển hoàn (suffix FW)

Mỗi hãng lớn có 2 dòng trong danh sách đối tác:
- `J&T Express` — đơn giao thành công
- `J&T FW` — đơn chuyển hoàn (hàng bị trả về shop)

Mục đích: tách bạch hạch toán giữa 2 loại. Đơn chuyển hoàn vẫn phát sinh cước 2 chiều (shop trả cước giao đi và cước hoàn về).

### 4.4 KShip — Hãng vận chuyển riêng

KShip là dịch vụ vận chuyển riêng của hệ thống, có thêm tính năng:
- Xác thực OTP cho thao tác nhạy cảm (đối soát, hủy đơn)
- Cấn trừ tự động cước với ví hệ thống
- Chu kỳ đối soát nhanh hơn
- Tạo đơn giao nhiều điểm trong cùng ngày

### 4.5 Đơn nhiều kiện (Multi-package)

Trường "Số kiện" cho phép 1 vận đơn chia thành nhiều thùng/kiện.
Ví dụ: đơn hàng lớn gồm 8 thùng → 1 vận đơn, 8 kiện. Hãng nhận toàn bộ và theo dõi theo mã vận đơn duy nhất.

### 4.6 Tự mang ra bưu cục

Khi bật tùy chọn này, hãng không cần đến lấy hàng tại shop. Phù hợp khi:
- Khu vực hãng không đến lấy tận nơi
- Shop muốn tiết kiệm phí lấy hàng
- Đơn nhỏ, shop tiện đường mang ra

---

## 5. SO SÁNH CẠNH TRANH

| Khía cạnh | Hệ thống | Sapo | Misa | Haravan |
|---|---|---|---|---|
| Số hãng tích hợp | **15–18** | 8–10 | 5–7 | 10–12 |
| Đối soát tự động | Có | Một phần | Thủ công | Có |
| Tách đối soát chuyển hoàn (FW) | Có | Không | Không | Một phần |
| Hãng vận chuyển riêng (first-party) | Có (KShip) | Không | Không | Không |
| Vận đơn nhiều kiện | Có | Một phần | Không | Có |
| Yêu cầu hỗ trợ inline trên vận đơn | Có | Không | Không | Không |
| Đối tác tùy chỉnh | Có | Có | Có | Có |
| So sánh giá / thời gian thực nhiều hãng | Không | Không | Không | Một phần |

---

## 6. ĐIỂM ĐAU NGHIỆP VỤ

### 6.1 Chọn hãng vận chuyển

| # | Vấn đề | Đề xuất |
|---|---|---|
| 1 | Không có bảng so sánh khi chọn hãng — shop phải tự nhớ ưu/nhược từng hãng | Báo giá thời gian thực nhiều hãng (cước + thời gian giao dự kiến) |
| 2 | Không có quy tắc tự động chọn hãng (vd: nội tỉnh → GHN; liên tỉnh → J&T) | Bộ quy tắc routing theo vùng / trọng lượng / SLA |
| 3 | Không có fallback khi hãng tích hợp gặp sự cố — shop phải chọn lại thủ công | Chuỗi fallback tự động |

### 6.2 Tài chính & Đối soát

| # | Vấn đề | Đề xuất |
|---|---|---|
| 4 | Tỷ lệ COD hãng chưa chuyển về shop cao, chu kỳ thanh toán dài — shop bị kẹt vốn | Đẩy COD thời gian thực, không chờ đối soát kỳ |
| 5 | Shop chậm trả cước có thể bị hãng ngừng dịch vụ — không có cảnh báo | Tự động thanh toán khi đến hạn, cảnh báo trước kỳ |
| 6 | Lịch sử đối soát không có thông báo chủ động — shop phải vào xem thủ công | Thông báo khi có kỳ đối soát mới |
| 7 | Không xử lý đa tiền tệ — shop bán xuyên biên giới không quản lý được | Hỗ trợ đa tiền tệ cho cước |
| 8 | Cấn trừ 2 chiều không có màn hình trực quan | Visualize "Bạn đang nợ X / Hãng đang nợ bạn Y" |

### 6.3 Vận đơn — quy trình

| # | Vấn đề | Đề xuất |
|---|---|---|
| 9 | Tab "Yêu cầu hỗ trợ" không hiển thị SLA phản hồi của hãng | Hiển thị "Hãng sẽ phản hồi trong X giờ" |
| 10 | Lý do hủy vận đơn không có danh mục chuẩn — mỗi hãng tự định nghĩa | Chuẩn hóa danh mục: KH không nghe máy / Sai địa chỉ / KH hủy / Hãng hủy |
| 11 | Không lưu bằng chứng giao hàng (chữ ký, ảnh) | Nhận dữ liệu bằng chứng từ hãng |
| 12 | Không có đánh giá chất lượng giao hàng từ khách hàng | Gửi khảo sát sau giao hàng |
| 13 | Không có link theo dõi vận đơn để chia sẻ cho khách hàng | Tự tạo link tracking gửi khách qua ZNS/SMS |

### 6.4 Phân tích & Nhiều kiện

| # | Vấn đề | Đề xuất |
|---|---|---|
| 14 | Vận đơn nhiều kiện không theo dõi trạng thái từng kiện riêng biệt | Theo dõi chi tiết theo kiện |
| 15 | Không có báo cáo hiệu suất từng hãng (đúng hạn, tỷ lệ lỗi, tỷ lệ hoàn) | Scorecard hãng tự động từ dữ liệu hệ thống |
| 16 | Không có báo cáo chi phí trung bình cước / đơn để so sánh giữa các hãng | Phân tích chi phí vận chuyển |

### 6.5 Đối tác tùy chỉnh (nhóm "Khác")

| # | Vấn đề | Đề xuất |
|---|---|---|
| 17 | Đối tác tùy chỉnh không đồng bộ trạng thái — shop phải gọi tài xế rồi cập nhật thủ công | Ứng dụng mobile cho shipper riêng |
| 18 | Không có đánh giá shipper tùy chỉnh | Đánh giá sao + lịch sử |
| 19 | Nhóm đối tác giao hàng có nhưng chưa gắn quy tắc routing theo nhóm | Routing engine theo nhóm đối tác |

---

## 7. CƠ HỘI ĐỘT PHÁ — TOP 5

| # | Tính năng | Giải quyết vấn đề | Mức độ ưu tiên |
|---|---|---|---|
| 1 | **Báo giá & Routing tự động** — so sánh cước + thời gian nhiều hãng, quy tắc tự động chọn | #1, #2, #3 | Cao |
| 2 | **Scorecard hãng vận chuyển + Phân tích chi phí** | #15, #16 | Cao |
| 3 | **COD thời gian thực + Tự động thanh toán cước** | #4, #5 | Cao (cash flow) |
| 4 | **Link theo dõi + Tự động thông báo khách hàng** qua ZNS/SMS | #13 | Trung |
| 5 | **Bằng chứng giao hàng + Đánh giá chất lượng** | #11, #12 | Trung |

---

## 8. TÓM LƯỢC

**Lớp giao vận gồm 2 module song hành:**
- **Đối tác giao hàng** = master data + 5 KPI tài chính tổng hợp + đối soát kỳ
- **Vận đơn** = chi tiết từng lần giao + 7 trạng thái + tab Yêu cầu hỗ trợ

**Điểm mạnh:**
- 15–18 hãng vận chuyển Việt Nam tích hợp 2 chiều — lợi thế cạnh tranh lớn nhất
- Tách đối soát đơn chuyển hoàn (suffix FW) — hạch toán chính xác
- Tab Yêu cầu hỗ trợ gắn trực tiếp vào vận đơn — độc nhất trên thị trường
- KShip first-party với cấn trừ tự động và chu kỳ đối soát nhanh
- Vận đơn nhiều kiện, bảo hiểm hàng (khai giá), tùy chọn tự mang ra bưu cục

**Khoảng trống chính:** Thiếu lớp thông minh — không có so sánh giá, routing tự động, scorecard hãng, link tracking cho khách hàng, bằng chứng giao hàng.

**Định hướng phát triển:** Từ nền tảng tích hợp rộng hiện có, đầu tư vào intelligence layer (routing, analytics, real-time COD, customer notification) để phục vụ seller chuyên nghiệp đa kênh.
