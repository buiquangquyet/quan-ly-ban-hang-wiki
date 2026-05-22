# Sửa thông tin khách hàng — Screen Spec

**Entity:** C01 Customer + C03 InvoiceIssuingInfo
**Liên quan:** [01-tong-quan.md](./01-tong-quan.md) §3 (form tạo) | [README.md](./README.md)

---

## 1. Bối cảnh

Form **Sửa KH** cho phép cập nhật identity, địa chỉ liên hệ, nhóm/phụ trách và thông tin xuất HĐĐT. Phân biệt với:

| Hành động | Màn hình | Scope |
|---|---|---|
| **Sửa thông tin** (file này) | Modal "Sửa thông tin" | C01 + C03 |
| **Sửa địa chỉ nhận hàng** | Tab "Địa chỉ nhận hàng" inline | C02 ShippingAddress |
| **Điều chỉnh công nợ** | Tab "Nợ cần thu" | AR override |
| **Điều chỉnh điểm** | Tab "Tích điểm" | C06 LoyaltyTransaction |

**Access points:**
1. Danh sách KH → expand row (tog) → dp-head → button **"Sửa"**
2. Bất kỳ popup/panel nào hiển thị KH → button "Sửa thông tin"

---

## 2. Form structure — 4 section accordion

Pre-filled từ Customer hiện tại. Section 1 luôn mở, các section còn lại collapsed.

| Section | Trường | Ghi chú |
|---|---|---|
| **1 — Identity** | Tên KH*, Mã KH (readonly), ĐT 1, ĐT 2, Email, Facebook, Avatar, Sinh nhật, Giới tính, Loại KH | Mã KH immutable |
| **2 — Địa chỉ** | Địa chỉ, Tỉnh/TP, Phường/Xã | Banner cảnh báo địa danh cũ nếu phát hiện |
| **3 — Nhóm & Phụ trách** | Nhóm KH (multi-select + Tạo mới), Người phụ trách, Ghi chú | Nhóm auto-rule vẫn ghi đè sau khi sync |
| **4 — Thông tin HĐ** | Tên người mua, MST [Tra cứu], CCCD, Hộ chiếu, Địa chỉ HĐ, Tỉnh, Phường, Email HĐ, SĐT HĐ, Ngân hàng, STK | Dropdown bank chuẩn VN |

---

## 3. Layout (ASCII)

```
┌────────────────────────────────────────────────────────────────┐
│  Sua thong tin khach hang — Nguyen Thi Hoa              [×]    │
├────────────────────────────────────────────────────────────────┤
│  [Section 1 — luon mo]                                          │
│  Ten KH*       [Nguyen Thi Hoa                            ]     │
│  Ma KH         KH125055745  (chi doc — khong sua duoc)          │
│  Dien thoai 1  [0965 888 222  ] ⚠ Trung SDT → KH125055312     │
│  Dien thoai 2  [               ]                                │
│  Email         [hoa.nguyen@email.com                      ]     │
│  Sinh nhat     [15/03/1990    ]  Gioi tinh  ◉ Nu  ○ Nam         │
│  Loai KH       ◉ Ca nhan  ○ Cong ty / To chuc                   │
├─ Dia chi ──────────────────────────────────────── [▼/▲] ──────┤
│  ⚠ Phat hien dia chi cu theo don vi hanh chinh truoc 01/07/2025 │
│  De xuat: Phuong Hoan Kiem, TP Ha Noi  [Ap dung de xuat]       │
│  Dia chi     [26 Ngo Sy Lien                              ]     │
│  Tinh/TP     [Ha Noi                                      ]     │
│  Phuong/Xa   [Phuong Hoan Kiem              ]                   │
├─ Nhom KH, Phu trach, Ghi chu ─────────────────── [▲] ─────────┤
│  Nhom KH  [VIP ×] [Than thiet ×]  [+ Them nhom]                 │
│  Phu trach  [Nguyen Van An                    ▼]                │
│  Ghi chu  [KH dac biet, uu tien xu ly...         ]              │
├─ Thong tin xuat hoa don ───────────────────────── [▼] ─────────┤
├────────────────────────────────────────────────────────────────┤
│  Lan cuoi sua: giangtest — 29/01/2026 09:14                     │
│                                            [Bo qua]  [Luu]     │
└────────────────────────────────────────────────────────────────┘
```

---

## 4. Use Cases

### UC-01: Sửa thông tin cơ bản

| | |
|---|---|
| **Trigger** | User click "Sửa" trên detail panel KH |
| **Main flow** | 1. Mở modal pre-filled với dữ liệu hiện tại<br>2. User chỉnh sửa field(s) cần thay đổi<br>3. Click "Lưu" → validate → save → toast "Đã cập nhật KH"<br>4. Detail panel refresh với dữ liệu mới |
| **AC-01.1** | Tất cả field có dữ liệu hiện tại được pre-fill đúng |
| **AC-01.2** | `customerCode` (Mã KH) render dạng readonly — không thể sửa |
| **AC-01.3** | Sau khi Lưu thành công, list row và detail panel cập nhật ngay (không cần reload trang) |
| **AC-01.4** | Footer hiển thị "Lần cuối sửa: {user} — {datetime}" |

### UC-02: Phát hiện trùng SĐT khi sửa

| | |
|---|---|
| **Trigger** | User thay đổi Điện thoại 1 hoặc Điện thoại 2 |
| **Main flow** | 1. Khi user rời field (onBlur) → gọi API check dedup<br>2. Nếu SĐT đã tồn tại ở KH khác → hiển thị inline warning: "⚠ SĐT đã có ở KH [Tên] — [Mã]"<br>3. User có thể click link → xem KH kia, hoặc bỏ qua warning và vẫn lưu<br>4. Nếu user lưu với SĐT trùng → log warning nhưng không block |
| **AC-02.1** | Warning xuất hiện ngay sau khi rời field (không cần submit form) |
| **AC-02.2** | Warning hiển thị tên + mã KH đang giữ SĐT đó (clickable link) |
| **AC-02.3** | Hệ thống vẫn cho lưu dù SĐT trùng — chỉ cảnh báo, không block |

### UC-03: Đề xuất cập nhật địa chỉ hành chính mới

| | |
|---|---|
| **Trigger** | Khi mở form sửa và địa chỉ hiện tại chứa đơn vị hành chính cũ (huyện/quận đã bị sáp nhập sau 01/07/2025) |
| **Main flow** | 1. Hệ thống detect địa chỉ cũ → hiển thị banner vàng trong Section 2<br>2. Banner gợi ý địa chỉ chuẩn mới + button "Áp dụng đề xuất"<br>3. User click "Áp dụng" → các field Tỉnh/Phường tự điền theo chuẩn mới<br>4. User xác nhận và Lưu |
| **AC-03.1** | Banner chỉ hiện khi địa chỉ thực sự thuộc đơn vị cũ |
| **AC-03.2** | Click "Áp dụng đề xuất" → field Tỉnh/TP + Phường/Xã cập nhật đúng theo địa danh mới |
| **AC-03.3** | User có thể bỏ qua banner và nhập địa chỉ tự do |

### UC-04: Thay đổi nhóm KH thủ công

| | |
|---|---|
| **Trigger** | User thêm/xóa nhóm trong Section 3 |
| **Main flow** | 1. User thêm/xóa nhóm → save<br>2. Nếu nhóm có "Auto-execute rule" (xem `02-nhom-va-segmentation.md`) → hệ thống tự re-evaluate và có thể override thay đổi thủ công lần tiếp theo<br>3. Hiển thị tooltip cảnh báo: "Nhóm [VIP] có rule tự động — thay đổi thủ công có thể bị ghi đè khi rule chạy lại" |
| **AC-04.1** | Có thể thêm nhiều nhóm (multi-select) |
| **AC-04.2** | Tooltip cảnh báo rule tự động hiển thị đúng khi nhóm có rule |

---

## 5. Business Rules

| # | Rule | Chi tiết |
|---|---|---|
| **BR-01** | `customerCode` immutable | Mã KH không thể sửa sau khi tạo — render readonly, không submit field này |
| **BR-02** | Tên KH bắt buộc | `name` không được để trống, whitespace-only cũng không hợp lệ |
| **BR-03** | SĐT trùng — cảnh báo không block | Hai KH có thể cùng SĐT (VD: hộ gia đình), nhưng phải warn |
| **BR-04** | Audit trail | Mỗi lần save ghi `updatedAt`, `updatedById` — hiển thị ở footer form |
| **BR-05** | Auto-rule group có thể override | Nhóm gán thủ công sẽ bị ghi đè nếu rule auto-execute chạy lại và KH không còn thỏa điều kiện |
| **BR-06** | MST Lookup | Khi click "Tra cứu MST" → call API Tổng cục Thuế → auto-fill Tên DN + Địa chỉ vào Section 4; không overwrite Section 1 |
| **BR-07** | Địa danh mới 01/07/2025 | Hệ thống lookup địa danh theo từ điển chuẩn mới; các đơn vị cũ (huyện/quận) vẫn được lưu nếu user tự nhập nhưng mark `addressOutdated = true` |
| **BR-08** | Concurrent edit | Nếu 2 user cùng mở form sửa 1 KH, user thứ 2 lưu sau nhận conflict alert: "KH vừa được cập nhật bởi [user]. Dữ liệu của bạn có thể đã cũ — tải lại?" |

---

## 6. Non-Functional Requirements

| # | NFR | Target |
|---|---|---|
| **NFR-01** | Modal open | < 200ms từ click đến form pre-filled |
| **NFR-02** | Phone dedup check | Inline warning sau onBlur ≤ 500ms |
| **NFR-03** | Save | API response ≤ 1s; UI update (row + panel) ≤ 200ms sau response |
| **NFR-04** | MST Lookup | Response ≤ 3s; show spinner; timeout graceful với message "Không tra cứu được — kiểm tra lại MST" |

---

## 7. Tóm lược

Form sửa KH là **phần cốt lõi nhất** của module KH — cập nhật hàng ngày cho mọi shop. Điểm mạnh cần giữ: 4-section accordion gọn (Section 1 luôn mở), tích hợp tra cứu MST. Điểm cần bổ sung: **dedup phone warning** (Pain #1), **audit trail** footer (ai sửa lần cuối), **cảnh báo địa danh 01/07/2025**, **cảnh báo rule auto-group**.
