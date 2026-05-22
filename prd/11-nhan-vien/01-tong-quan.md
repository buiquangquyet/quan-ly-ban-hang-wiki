# Tổng quan — Nhân viên

**Phạm vi:** Module Nhân viên — quản lý thông tin nhân sự, ca làm việc, hoa hồng và hiệu suất bán hàng của nhân viên trong hệ thống POS đa chi nhánh.
**Liên quan:** Module 10 (Phân quyền), Module 04 (Sổ quỹ — chi lương), Module 06/07 (Đơn hàng/Hóa đơn — tính hoa hồng)

---

## 1. Vị trí trong hệ sinh thái

Nhân viên là **master data vận hành** — mọi giao dịch trên POS đều gắn nhân viên thực hiện:

```
[Nhân viên] ──────────────────────────────────────────────┐
     │                                                      │
     ├── Bán hàng (HD) → gán "Người tạo" + "Nhân viên bán" │
     ├── Đặt hàng (DH) → gán "Nhân viên phụ trách"         │
     ├── Nhập hàng (PN) → gán "Người nhận"                  │
     ├── Kiểm kho (KK) → gán "Người kiểm"                  │
     ├── Sổ quỹ (PT/PC) → gán "Người tạo"                  │
     └── Phân quyền (RBAC) → 1 NV có 1..N vai trò          │
                                                            ▼
                                                   [Hoa hồng, Báo cáo]
```

---

## 2. Cấu trúc nghiệp vụ

### 2.1 Thông tin nhân viên (Employee — NV01)

Mỗi nhân viên là một tài khoản người dùng với thêm lớp thông tin nhân sự:

| Trường | Bắt buộc | Ghi chú |
|---|---|---|
| Mã nhân viên | Có | Auto NV000XXX, cho phép custom |
| Tên nhân viên | Có | Tên hiển thị trên POS |
| Điện thoại | Có | Duy nhất trong hệ thống |
| Email | Không | Dùng để login / nhận thông báo |
| Địa chỉ | Không | |
| Ngày sinh | Không | Dùng gửi chúc mừng sinh nhật |
| Giới tính | Không | |
| Ngày vào làm | Có | Dùng tính thâm niên |
| Chức vụ | Có | Enum: Quản lý / Thu ngân / NV kho / NV bán hàng / Giao hàng |
| Chi nhánh | Có | 1..N chi nhánh (NV có thể làm nhiều CN) |
| Vai trò hệ thống | Không | Link sang module 10 — RBAC |
| Lương cơ bản | Không | Phục vụ báo cáo nhân sự |
| Trạng thái | Có | Đang làm việc / Nghỉ phép / Đã nghỉ việc |

**Mã tài khoản vs Mã nhân viên:**
- Tài khoản hệ thống (module 10) dùng để đăng nhập
- Mã nhân viên là định danh nghiệp vụ, hiển thị trên chứng từ

### 2.2 Ca làm việc (Shift — NV02)

Quản lý lịch làm việc của nhân viên theo ca:

| Trường | Ghi chú |
|---|---|
| Tên ca | VD: Ca sáng 8h-12h, Ca chiều 13h-17h |
| Giờ bắt đầu / kết thúc | Định nghĩa khung giờ |
| Ngày trong tuần | Thứ 2–7, Chủ nhật |
| Chi nhánh áp dụng | |
| Nhân viên trong ca | Danh sách NV được phân công |

**Lịch phân ca (Schedule):**
- Chủ shop tạo lịch phân ca theo tuần/tháng
- Kéo thả NV vào ca
- Cảnh báo khi NV trùng ca / làm việc quá N giờ/ngày
- Xuất lịch dạng bảng (Google Sheet style) để in hoặc gửi Zalo

### 2.3 Hoa hồng (Commission — NV03)

Cơ chế tính thưởng cho nhân viên dựa trên doanh số:

**Cấu trúc rule hoa hồng:**

| Loại rule | Mô tả | Ví dụ |
|---|---|---|
| % doanh thu | Tỷ lệ % trên doanh thu bán | 2% tổng doanh thu HĐ |
| % lợi nhuận | % trên lợi nhuận gộp | 5% (Giá bán − Giá vốn) |
| Theo sản phẩm | Cố định per SKU hoặc nhóm hàng | 10,000đ/sản phẩm A |
| Theo bậc doanh số | Bậc thang tăng dần | 0–10tr: 1%, 10–20tr: 1.5%, >20tr: 2% |
| Kết hợp | Nhiều rule áp đồng thời | |

**Scope áp dụng:**
- Per nhân viên cụ thể
- Per vai trò (tất cả thu ngân)
- Per chi nhánh

**Tính toán hoa hồng:**
- Kỳ tính: theo ngày / tuần / tháng (cấu hình)
- Trạng thái HĐ: chỉ tính HĐ đã hoàn thành (không tính hủy, trả hàng)
- Khấu trừ: Trả hàng trừ ngược hoa hồng tương ứng
- Phê duyệt: Hoa hồng cần được chủ shop duyệt trước khi chi

---

## 3. Sơ đồ quan hệ entity

```
Employee (NV01)
├── 1..N Shift Assignment (NV02) — ca làm việc
├── 1..N CommissionRule (NV03)   — rule hoa hồng
├── 1..N CommissionPeriod        — bảng tổng hợp hoa hồng theo kỳ
├── 1..N Invoice (HD)            — các HĐ đã tạo
├── 1..N UserAccount             — tài khoản đăng nhập (module 10)
└── 1..N BranchAccess            — chi nhánh được phép truy cập
```

---

## 4. Nghiệp vụ chính

### UC-NV01: Tạo / Sửa nhân viên
- **Actor:** Chủ shop / Quản lý chi nhánh
- **Flow:** Nhập thông tin → Gán chi nhánh → Tạo tài khoản login (tùy chọn) → Gán vai trò RBAC → Lưu
- **Rule:** Mã NV + SĐT phải duy nhất trong tenant
- **Edge case:** NV làm nhiều CN → access control độc lập theo CN

### UC-NV02: Phân ca làm việc
- **Actor:** Quản lý chi nhánh
- **Flow:** Chọn tuần → Kéo thả NV vào ô (Ca × Ngày) → Lưu → Thông báo NV qua Zalo/SMS
- **Rule:** NV không thể trùng 2 ca cùng lúc tại cùng CN; cảnh báo nếu tổng giờ/tuần > 48h
- **Edge case:** NV xin đổi ca → cần giao diện để NV request, quản lý approve

### UC-NV03: Tính và duyệt hoa hồng
- **Actor:** Chủ shop (tính), Kế toán (chi)
- **Flow:** Hệ thống tự tính cuối kỳ → Chủ shop review → Duyệt → Tạo Phiếu chi Sổ quỹ tự động → Chi lương
- **Rule:** Hoa hồng chỉ tính HĐ trạng thái "Hoàn thành"; TH trả hàng khấu trừ ngược
- **Business rule:** 1 HĐ có thể gắn nhiều NV (người tư vấn + người chốt) → split hoa hồng theo tỷ lệ cấu hình

### UC-NV04: Xem hiệu suất nhân viên
- **Actor:** Chủ shop / Quản lý
- **View:** Doanh thu / Số HĐ / Giá trị TB / Hoa hồng theo kỳ per NV
- **So sánh:** Xếp hạng NV trong cùng chi nhánh

---

## 5. Bảng giá tích hợp nhân viên

Nhân viên gắn với **giá bán** theo 2 cách:
1. **Người bán được quyền sửa giá** → kiểm soát qua RBAC permission "Xem-sửa giá bán"
2. **NV có bảng giá riêng** → VD: NV sỉ dùng bảng giá sỉ khi tạo HĐ (future)

---

## 6. Pain points hiện tại

| # | Pain | Mức độ |
|---|---|---|
| P1 | Không có module ca làm việc — chủ shop dùng Excel/Zalo | Cao |
| P2 | Hoa hồng tính tay cuối tháng — sai sót, tranh cãi | Cao |
| P3 | Không track được NV nào bán hóa đơn nào — chỉ có "người tạo" | Trung bình |
| P4 | Không thể gán 2 nhân viên vào 1 HĐ (tư vấn + chốt đơn) | Trung bình |
| P5 | NV nghỉ việc vẫn xuất hiện trong dropdown → confusion | Thấp |
| P6 | Không có cảnh báo khi NV làm việc > định mức giờ | Thấp |
| P7 | Không có history thay đổi nhân sự (ai được thăng chức, đổi CN) | Trung bình |

---

## 7. Cơ hội cải tiến (Top 5)

### I1. Ca làm việc thông minh (Smart Shift)
- Tự động đề xuất phân ca dựa trên lịch sử đông khách (từ báo cáo giờ vàng)
- Cân bằng workload: tránh NV nào làm quá nhiều ca liên tiếp

### I2. Hoa hồng minh bạch real-time
- NV xem hoa hồng tích lũy của mình ngay trên app mobile
- Chủ shop duyệt hoa hồng 1 click → tự động sinh Phiếu chi

### I3. Hiệu suất NV dashboard
- Bảng xếp hạng doanh số NV cross-branch
- Alert khi NV có hành vi bất thường (hoàn hàng nhiều, hủy HĐ nhiều) → liên kết module Audit

### I4. NV multi-branch với access độc lập
- NV làm tại nhiều CN với vai trò khác nhau (Thu ngân CN A, Quản lý CN B)
- Time-based access: chỉ được đăng nhập trong giờ ca

### I5. Tích hợp chấm công GPS/WiFi
- Check-in/check-out qua app mobile với xác nhận vị trí GPS hoặc WiFi nội bộ
- Auto-generate bảng chấm công tháng cho payroll
