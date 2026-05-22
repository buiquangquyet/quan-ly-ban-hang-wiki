# Tổng quan — Báo cáo & Phân tích kinh doanh

**Phạm vi:** Hệ thống báo cáo và dashboard phân tích — tổng hợp dữ liệu từ mọi module vận hành để chủ shop và quản lý đưa ra quyết định kinh doanh.
**Liên quan:** Tất cả module — báo cáo là tầng read-only tổng hợp từ dữ liệu vận hành.

---

## 1. Triết lý thiết kế báo cáo

Báo cáo phục vụ **3 persona chính**:

| Persona | Nhu cầu | Tần suất |
|---|---|---|
| **Chủ shop** | Tổng quan doanh thu, lợi nhuận, cash flow | Hàng ngày |
| **Quản lý chi nhánh** | Hiệu suất nhân viên, hàng bán chạy, tồn kho cảnh báo | Hàng ngày |
| **Kế toán** | Công nợ, sổ quỹ, thuế, đối soát | Hàng tuần / tháng |

---

## 2. Nhóm báo cáo

### 2.1 Báo cáo bán hàng (BC-BH)

Phân tích doanh thu và hiệu quả bán hàng:

| Báo cáo | Mô tả | Dimension chính |
|---|---|---|
| Doanh thu theo thời gian | Trend doanh thu ngày/tuần/tháng/quý/năm | Thời gian × Chi nhánh |
| Doanh thu theo nhân viên | Ranking NV theo doanh số, số HĐ, giá trị TB | NV × Kỳ |
| Doanh thu theo kênh bán | So sánh POS / Shopee / TikTok / Lazada | Kênh × Thời gian |
| Doanh thu theo nhóm hàng | Đóng góp doanh thu của từng nhóm SP | Nhóm SP × Thời gian |
| Phân tích theo giờ | Giờ vàng trong ngày, bản đồ nhiệt | Giờ × Thứ |
| So sánh cùng kỳ | So kỳ này với kỳ trước / cùng kỳ năm ngoái | YoY / WoW / MoM |

**Metrics chính:**
- Tổng doanh thu
- Tổng số hóa đơn
- Giá trị hóa đơn trung bình (ATV)
- Tổng giảm giá / tỷ lệ giảm giá
- Tổng trả hàng / tỷ lệ trả hàng
- Doanh thu thuần (sau trả hàng)

### 2.2 Báo cáo lợi nhuận (BC-LN)

| Báo cáo | Mô tả |
|---|---|
| Lợi nhuận gộp | Doanh thu − Giá vốn hàng bán (COGS) |
| Lợi nhuận theo SP | Lợi nhuận gộp per SKU, per nhóm |
| Lợi nhuận theo kênh | POS vs Online — kênh nào lãi hơn |
| Chi phí vận hành | Tổng phiếu chi Sổ quỹ không phải nhập hàng |
| P&L summary | Thu nhập − Chi phí = Lợi nhuận ròng (tháng) |

**Lưu ý:** Lợi nhuận gộp cần giá vốn WAC chính xác từ module Nhập hàng. Nếu nhập hàng sai giá vốn → báo cáo lợi nhuận sai.

### 2.3 Báo cáo hàng hóa & kho (BC-HH)

| Báo cáo | Mô tả |
|---|---|
| Hàng bán chạy | Top N SP theo số lượng / doanh thu / lợi nhuận |
| Hàng tồn kho | Tồn hiện tại × Chi nhánh, cảnh báo dưới min |
| Hàng tồn chậm | SP không bán trong X ngày |
| Nhập hàng | Tổng nhập theo NCC, theo thời gian |
| Biến động tồn | Thẻ kho tổng hợp per nhóm hàng / khoảng thời gian |
| Kiểm kho | Kết quả kiểm kho, chênh lệch tích lũy |
| Hàng sắp hết hạn | Lô hàng hết hạn trong 7/14/30 ngày |

### 2.4 Báo cáo khách hàng (BC-KH)

| Báo cáo | Mô tả |
|---|---|
| Khách hàng mới | Số KH mới theo kỳ |
| Khách hàng quay lại | Retention rate — tỷ lệ KH mua lần 2+ |
| RFM Analysis | Phân nhóm Recency × Frequency × Monetary |
| Top khách hàng | Xếp hạng theo doanh số / lợi nhuận |
| Công nợ khách hàng (AR Aging) | Phân nhóm tuổi nợ: <30 ngày, 30–60, 60–90, >90 |
| Khách hàng mất (Churn) | KH không mua trong X ngày |
| Tích điểm | Tổng điểm phát hành, đã dùng, sắp hết hạn |

### 2.5 Báo cáo tài chính & sổ quỹ (BC-TC)

| Báo cáo | Mô tả |
|---|---|
| Sổ quỹ | Dòng tiền vào/ra theo kỳ, số dư đầu/cuối kỳ |
| Thu chi theo loại | Phân tích cơ cấu thu chi (tiền mặt / ngân hàng / ví) |
| Công nợ phải thu (AR) | Tổng nợ KH, aging |
| Công nợ phải trả (AP) | Tổng nợ NCC, aging |
| Dự báo dòng tiền | Cash flow forecast 7/14/30 ngày (từ đơn hàng pending + phiếu nhập chưa thanh toán) |

### 2.6 Báo cáo nhân viên (BC-NV)

| Báo cáo | Mô tả |
|---|---|
| Doanh số theo NV | Ranking, trend |
| Hoa hồng | Tổng hoa hồng phát sinh, đã duyệt, đã chi |
| Hiệu suất theo ca | Doanh thu per ca làm việc |
| Hoạt động bất thường | Hủy HĐ nhiều, hoàn hàng nhiều, sửa giá nhiều |

### 2.7 Báo cáo nhà cung cấp (BC-NCC)

| Báo cáo | Mô tả |
|---|---|
| Lịch sử nhập hàng per NCC | Tổng giá trị, số lần nhập |
| Công nợ NCC (AP Aging) | Nợ đang ở độ tuổi nào |
| Hiệu suất NCC | Tỷ lệ giao đúng hạn, số ngày chờ trung bình |
| Trả hàng NCC | Tổng trả hàng theo NCC |

---

## 3. Dashboard tổng quan (BC-DB)

Dashboard là trang chủ sau khi đăng nhập — hiển thị KPI realtime:

### 3.1 Dashboard chủ shop / Quản lý

**Khu vực 1 — KPI hôm nay:**
- Doanh thu hôm nay (vs hôm qua, vs cùng ngày tuần trước)
- Số hóa đơn hôm nay
- Khách hàng mới hôm nay
- Tổng tiền trong quỹ (tiền mặt + ngân hàng)

**Khu vực 2 — Biểu đồ doanh thu:**
- Line chart doanh thu 7 ngày hoặc 30 ngày
- Bar chart doanh thu theo giờ trong ngày hiện tại

**Khu vực 3 — Cảnh báo tồn kho:**
- Danh sách SP dưới định mức tồn tối thiểu
- Danh sách SP sắp hết hàng (dự kiến hết trong 3 ngày)

**Khu vực 4 — Top hàng bán chạy:**
- Top 5 SP theo doanh số hôm nay
- Top 5 SP theo lượt bán trong 7 ngày

**Khu vực 5 — Công nợ cần xử lý:**
- Tổng nợ KH đến hạn
- Tổng nợ NCC cần thanh toán trong 7 ngày

### 3.2 Dashboard theo chi nhánh

Trong hệ thống đa chi nhánh:
- Dropdown chọn chi nhánh
- Tất cả KPI trên lọc theo CN đang chọn
- View "Tất cả CN": heatmap so sánh doanh thu các CN

---

## 4. Cơ chế kỹ thuật báo cáo

### 4.1 Chiều lọc (Dimension) chung

Mọi báo cáo đều hỗ trợ lọc theo:
- **Thời gian:** Hôm nay / Hôm qua / Tuần này / Tháng này / Quý này / Năm này / Tùy chỉnh
- **Chi nhánh:** Một CN / Nhiều CN / Tất cả
- **Nhân viên:** Lọc theo NV tạo giao dịch
- **Nhóm hàng / SP cụ thể**
- **Kênh bán:** POS / Shopee / TikTok / Lazada / Tất cả

### 4.2 Export

- Export Excel (.xlsx) toàn bộ dữ liệu raw
- Export PDF — báo cáo định dạng in ấn
- Scheduled email: gửi báo cáo tự động hàng ngày/tuần/tháng

### 4.3 Kiến trúc dữ liệu

Báo cáo phức tạp (lợi nhuận, RFM, aging) cần:
- **Materialized view** hoặc **data warehouse** riêng — không query thẳng OLTP
- Refresh theo chu kỳ (realtime cho dashboard KPI, batch cho báo cáo lợi nhuận)
- Tham khảo: `03-kien-truc-ky-thuat/database-design.md`

---

## 5. Pain points hiện tại

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không có báo cáo lợi nhuận thuần — chỉ có doanh thu | Cao |
| P2 | Báo cáo không lọc được đồng thời nhiều CN | Cao |
| P3 | Không có RFM / cohort analysis — chỉ có bảng dữ liệu thô | Cao |
| P4 | Không có AR Aging chi tiết — chỉ có tổng nợ | Trung bình |
| P5 | Không thể schedule email báo cáo tự động | Trung bình |
| P6 | Dashboard không cấu hình được (mỗi persona muốn KPI khác nhau) | Trung bình |
| P7 | Không có báo cáo so sánh cùng kỳ Year-over-Year | Trung bình |
| P8 | Xuất Excel bị timeout với dữ liệu lớn (> 100k dòng) | Cao |

---

## 6. Cơ hội cải tiến (Top 5)

### I1. P&L Dashboard thực sự
- Lợi nhuận gộp (Gross Profit) = Doanh thu − COGS
- Lợi nhuận ròng = GP − Chi phí vận hành (từ Sổ quỹ)
- Drill-down đến từng HĐ đóng góp lợi nhuận

### I2. AI Insight — Báo cáo bằng ngôn ngữ tự nhiên
- "Tuần này so với tuần trước như thế nào?"
- Hệ thống tự nhận ra anomaly và giải thích: "Doanh thu thứ 3 giảm 40% so với trung bình — có thể do mưa lớn và 3 NV vắng"
- Liên kết với KiotViet Copilot (brainstorm A1)

### I3. Custom Dashboard
- Chủ shop kéo thả widget lên dashboard cá nhân
- Mỗi vai trò có dashboard mặc định phù hợp

### I4. Cohort Analysis & Retention
- Phân tích nhóm KH mua tháng N có bao nhiêu % quay lại tháng N+1, N+2...
- Identify chính xác "rò rỉ" KH ở giai đoạn nào

### I5. Cảnh báo thông minh (Smart Alert)
- Push notification khi: doanh thu CN X < 70% kế hoạch lúc 17h
- Alert khi tồn kho SP hot bán nhanh — dự kiến hết trong 24h
- Alert khi NV có hành vi bán dưới giá sàn bất thường
