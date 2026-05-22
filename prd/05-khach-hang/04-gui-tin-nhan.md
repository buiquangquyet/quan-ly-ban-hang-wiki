# Gửi tin nhắn cho khách hàng — Screen Spec

**Entity:** C01 Customer | Kênh: ZNS / SMS / Email / Zalo OA
**Liên quan:** [01-tong-quan.md](./01-tong-quan.md) §2.2 (top actions) | [README.md](./README.md)

---

## 1. Bối cảnh

Tính năng cho phép gửi tin nhắn tới KH qua 4 kênh. Có hai luồng chính:

| Luồng | Trigger | Scope KH | Kênh |
|---|---|---|---|
| **Đơn lẻ** | Detail panel KH → "Gửi tin nhắn" | 1 KH cụ thể | ZNS / SMS / Email / Zalo OA |
| **Hàng loạt** | List KH → chọn checkbox / filter → "Gửi tin nhắn ▼" | N KH đã chọn hoặc toàn bộ filter | ZNS / SMS / Email / Zalo OA |

---

## 2. Đặc tính các kênh gửi

| Kênh | Yêu cầu phía KH | Yêu cầu phía Shop | Giới hạn nội dung | Chi phí |
|---|---|---|---|---|
| **ZNS** | KH có Zalo, đã follow OA, đã tương tác trước | Template phải được ZCA duyệt trước | Theo template duyệt; không quảng cáo tự do | Trả phí per message (Zalo) |
| **SMS** | SĐT hợp lệ | Đăng ký Brandname (SMS có đầu số thương hiệu) | 160 ký tự/SMS; non-Unicode | Trả phí per SMS |
| **Email** | Email hợp lệ | Domain/IP không bị blacklist | Không giới hạn ký tự; HTML | Trả phí per email hoặc gói quota |
| **Zalo OA** | KH follow Zalo OA | Có tài khoản Zalo OA được xác nhận | Free text, nhưng chỉ trong 48h sau lần KH nhắn trước | Thường miễn phí |

---

## 3. Layout (ASCII)

### 3.1 Modal đơn lẻ

```
┌──────────────────────────────────────────────────────────────────┐
│  Gui tin nhan — Nguyen Thi Hoa (0965 888 222)             [×]   │
├─ Kenh gui ───────────────────────────────────────────────────────┤
│  [ZNS ●]  [SMS]  [Email]  [Zalo OA]                              │
│  Trang thai: Zalo da follow OA ✓                                 │
├─ Mau tin nhan ───────────────────────────────────────────────────┤
│  Chon mau: [Chuc mung sinh nhat ▼]   [Xem truoc]                 │
│  Danh muc: Cham soc KH | Khuyen mai | Nhac no | Thong bao DH     │
├─ Noi dung (xem truoc) ───────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────┐    │
│  │ Chao Nguyen Thi Hoa!                                     │    │
│  │ Nhan dip sinh nhat, shop tang ban voucher 50.000d.       │    │
│  │ Ma: BDAY-KH125055745. HSD: 30 ngay ke tu ngay nhan.     │    │
│  │                            — Cua hang ABC                │    │
│  └──────────────────────────────────────────────────────────┘    │
│  Bien so: {{ten_kh}} {{ma_voucher}} {{ngay_het_han}}             │
├─ Thoi gian gui ──────────────────────────────────────────────────┤
│  ◉ Gui ngay    ○ Len lich: [  /  /      ] [  :  ]                │
├──────────────────────────────────────────────────────────────────┤
│  Chi phi uoc tinh: 1 × 350d = 350d (ZNS)                        │
│                                               [Huy]  [Gui ngay] │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Modal hàng loạt

```
┌──────────────────────────────────────────────────────────────────┐
│  Gui tin nhan hang loat                                   [×]   │
├─ Doi tuong nhan ─────────────────────────────────────────────────┤
│  ◉ 3 KH da chon (Nguyen Thi Hoa, Tran Van Minh, +1)             │
│  ○ Tat ca KH theo bo loc hien tai (1.624 KH)                    │
├─ Kenh gui ───────────────────────────────────────────────────────┤
│  [ZNS ●]  [SMS]  [Email]  [Zalo OA]                              │
│  Hop le: 3/3 KH co Zalo  |  Khong hop le: 0 KH                  │
├─ Mau tin nhan ───────────────────────────────────────────────────┤
│  Chon mau: [Khuyen mai thang 5 ▼]   [Xem truoc]                  │
├─ Thoi gian gui ──────────────────────────────────────────────────┤
│  ◉ Gui ngay    ○ Len lich: [  /  /      ] [  :  ]                │
├──────────────────────────────────────────────────────────────────┤
│  So KH se nhan: 3  |  Khong gui duoc: 0  |  Chi phi: 3 × 350d = 1.050d │
│                                               [Huy]  [Gui]      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Use Cases

### UC-01: Gửi tin nhắn đơn lẻ cho 1 KH

| | |
|---|---|
| **Trigger** | User click "Gửi tin nhắn" trong detail panel KH |
| **Main flow** | 1. Mở modal với thông tin KH pre-fill (tên, SĐT, email)<br>2. Chọn kênh → hiển thị trạng thái kênh (KH có follow OA không? Email có hợp lệ?)<br>3. Chọn template → preview nội dung với biến số đã điền<br>4. Chọn "Gửi ngay" hoặc "Lên lịch"<br>5. Click "Gửi" → hệ thống enqueue → toast "Đã gửi / Đã lên lịch" |
| **AC-01.1** | Trạng thái kênh hiển thị đúng (Zalo follow / không follow; email valid/invalid) |
| **AC-01.2** | Template preview render đúng biến số từ dữ liệu KH thực |
| **AC-01.3** | Chi phí ước tính hiển thị dựa trên kênh và số lượng SMS chunk |
| **AC-01.4** | Sau khi gửi → lưu vào lịch sử gửi tin (có thể xem lại) |

### UC-02: Gửi tin nhắn hàng loạt

| | |
|---|---|
| **Trigger** | Danh sách KH → chọn ≥ 1 checkbox (hoặc không chọn = gửi all) → click "Gửi tin nhắn ▼" |
| **Main flow** | 1. Mở modal bulk với đối tượng là "N KH đã chọn" hoặc "Toàn bộ filter"<br>2. Hiển thị số KH hợp lệ per kênh (có Zalo / có email / có SĐT)<br>3. Chọn kênh, mẫu, thời gian<br>4. Confirm → hệ thống batch enqueue → toast "Đã lên hàng đợi gửi {N} tin" |
| **AC-02.1** | Hiển thị đúng số KH hợp lệ / không hợp lệ cho kênh đã chọn |
| **AC-02.2** | Option "Gửi tất cả theo filter" cho phép gửi tới toàn bộ KH trong filter hiện tại (không cần tick từng ô) |
| **AC-02.3** | Chi phí tổng ước tính hiển thị trước khi confirm |
| **AC-02.4** | Sau khi gửi → có thể theo dõi trạng thái batch trong lịch sử chiến dịch |

### UC-03: Chọn và preview template

| | |
|---|---|
| **Trigger** | User click dropdown "Chọn mẫu" |
| **Main flow** | 1. Danh sách template phân theo danh mục: Chăm sóc KH / Khuyến mãi / Nhắc nợ / Thông báo đơn hàng<br>2. Chọn template → preview panel cập nhật ngay<br>3. Biến số `{{ten_kh}}` `{{ma_kh}}` `{{so_tien_no}}` tự điền từ dữ liệu KH<br>4. Biến số chưa có data → hiển thị màu đỏ + tooltip "Thiếu dữ liệu" |
| **AC-03.1** | Template phân loại đúng danh mục |
| **AC-03.2** | Preview render ngay khi chọn template (≤ 300ms) |
| **AC-03.3** | Biến số thiếu dữ liệu được highlight đỏ, không cho phép gửi khi còn biến lỗi |

### UC-04: Lên lịch gửi

| | |
|---|---|
| **Trigger** | User chọn "Lên lịch" và nhập thời gian |
| **Main flow** | 1. Hiện datetime picker<br>2. Validate: thời gian phải > hiện tại + 5 phút<br>3. Click "Lên lịch" → toast "Đã lên lịch gửi lúc {datetime}"<br>4. Có thể hủy lịch trong vòng 5 phút trước giờ gửi |
| **AC-04.1** | Không cho phép lên lịch thời gian quá khứ |
| **AC-04.2** | Toast xác nhận hiển thị đúng thời gian đã chọn |

### UC-05: Xem lịch sử gửi tin nhắn

| | |
|---|---|
| **Trigger** | Trong detail panel KH → Tab "Thông tin" → link "Lịch sử nhắn tin" (hoặc tab riêng sau này) |
| **Scope** | Out of scope hiện tại — đây là feature "Lịch sử chiến dịch" dùng chung cho cả bulk và single |

---

## 5. Business Rules

| # | Rule | Chi tiết |
|---|---|---|
| **BR-01** | ZNS bắt buộc template đã duyệt | Chỉ cho phép chọn template ZNS đã được ZCA (Zalo Content Approval) duyệt. Template mới phải qua luồng gửi duyệt riêng (out of scope) |
| **BR-02** | ZNS bắt buộc KH follow OA | Nếu KH chưa follow Zalo OA → disable kênh ZNS cho KH đó, hiển thị "Chưa follow OA" |
| **BR-03** | Zalo OA chỉ trong session 48h | Chỉ gửi Zalo OA được nếu KH đã chủ động nhắn shop trong vòng 48h gần nhất. Nếu quá hạn → disable kênh, gợi ý chuyển sang ZNS |
| **BR-04** | SMS encoding | SMS standard = 160 ký tự (Latin). Nếu có ký tự Unicode (tiếng Việt) → 70 ký tự/SMS chunk. Hiển thị chunk count và chi phí tương ứng |
| **BR-05** | Opt-out / Blocklist | KH có `doNotContact = true` → không cho gửi bất kỳ kênh nào; hiển thị "KH đã yêu cầu không liên hệ" |
| **BR-06** | Rate limiting bulk | Batch gửi bulk không quá 1.000 message/phút per kênh để tránh bị provider block. Progress hiển thị "Đang gửi: X / N" |
| **BR-07** | Biến số bắt buộc | Nếu template có biến `{{so_tien_no}}` nhưng KH không có nợ → không cho gửi cho KH đó (highlight đỏ trong danh sách bulk) |
| **BR-08** | Lịch sử gửi | Mỗi lần gửi thành công ghi vào log: kênh, template, nội dung render, thời gian gửi, trạng thái delivery (sent/delivered/failed) |

---

## 6. Non-Functional Requirements

| # | NFR | Target |
|---|---|---|
| **NFR-01** | Template preview render | ≤ 300ms sau khi chọn template |
| **NFR-02** | Kênh status check (Zalo follow) | ≤ 500ms; nếu timeout hiển thị "Không kiểm tra được — vẫn có thể gửi" |
| **NFR-03** | Bulk enqueue | ≤ 2s để enqueue toàn bộ batch vào hàng đợi (actual gửi là async) |
| **NFR-04** | Batch throughput | 1.000 message/phút per kênh (BR-06) |
| **NFR-05** | Retry khi lỗi | Tự retry 2 lần nếu provider trả lỗi tạm thời (rate limit, timeout). Sau 3 lần fail → đánh dấu `failed` trong lịch sử |

---

## 7. Tóm lược

Gửi tin nhắn KH là tính năng **retention & engagement** cốt lõi, đặc biệt quan trọng cho shop B2C. KiotViet đã có 4 kênh (ZNS/SMS/Email/Zalo OA) và bulk select — nền tảng tốt. Cần bổ sung: **dedup opt-out check** (BR-05), **biến số validate trước gửi** (BR-07), **lịch sử gửi tin per KH** (UC-05), và **template manager** độc lập (out of scope file này). Đây là điểm mạnh cạnh tranh vs phần mềm nhỏ — cần đào sâu vào personalization và delivery analytics.
