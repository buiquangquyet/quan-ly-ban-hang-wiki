# Chi tiết Hóa đơn — Deep Dive

**Bối cảnh:** Màn hình Chi tiết Hóa đơn (Invoice Detail) là **giao điểm của 8 luồng nghiệp vụ** — nơi thu ngân xác nhận thanh toán, quản lý theo dõi giao hàng, kế toán phát hành HĐĐT, và quản lý tra soát sai sót. Một màn hình phục vụ 4–5 vai trò khác nhau với nhu cầu đọc/ghi không trùng nhau.

**Entities:** Invoice (HD), InvoiceLine, Payment/Receipt (TTHD), ShippingOrder, EInvoice (HĐĐT), StockCardLine (E10), CustomerDebt, LoyaltyPoint
**Liên quan:** [hoa-don.md](./hoa-don.md) | [../06-don-hang/don-hang-deep-dive.md](../06-don-hang/) | [../04-so-quy/01-tong-quan.md](../04-so-quy/)

---

## 1. Điểm truy cập

```
Danh sách Hóa đơn (/man/#/Invoices)
    ├── Click mã HD → Drawer/Panel chi tiết (slide in)
    └── URL trực tiếp: /invoices/{id}

POS màn bán hàng → Thanh toán → In hóa đơn → Link mở chi tiết
Đặt hàng (DH) → Xử lý → Tạo HD → tự chuyển sang chi tiết HD
Thẻ kho (E10) → Click mã HD###### → Quick view chi tiết
Sổ quỹ → Click mã TTHD###### → Popup chi tiết phiếu thu → link sang HD
Khách hàng → Tab Lịch sử bán → Click mã HD → Chi tiết HD
```

---

## 2. Layout tổng thể

```
┌──────────────────────────────────────────────────────────────────────┐
│ [← Danh sách]  HD016580 — 21/05/2025 11:35                          │
│                                  [In] [Chia se] [Sao chep] [...]    │
├────────────────────────────────────┬─────────────────────────────────┤
│         PANEL TRÁI (60%)           │      PANEL PHAI (40%)           │
│                                    │                                 │
│  Thong tin chung                   │  Tong tien hang    2.805.000    │
│  Hang hoa (3 dong)                 │  Chiet khau hoa don       0    │
│  Tab: HDDT | Giao hang | Lich su   │  VAT (10%)          280.500    │
│                                    │  Phi giao hang       35.000    │
│                                    │  ─────────────────────────     │
│                                    │  TONG CONG        3.120.500    │
│                                    │                                 │
│                                    │  Khach can tra    3.120.500    │
│                                    │  Khach da tra     3.200.000    │
│                                    │  Tien thua tra         79.500  │
│                                    │                                 │
│                                    │  [Thanh toan them]             │
└────────────────────────────────────┴─────────────────────────────────┘
```

---

## 3. Panel trái — Thông tin chung

### 3.1 Header hóa đơn

```
┌────────────────────────────────────────────────────────────────┐
│  HD016580                           ● Hoan thanh               │
│  21/05/2025 11:35  |  CN Quan 1  |  Kenh: POS                  │
│  Nguoi ban: Tran Thi Lan                                        │
│  Bang gia: Bang gia chung                                       │
│  Ghi chu: Khach yeu cau goi rieng size 40                      │
│                                                [Sua ghi chu]   │
└────────────────────────────────────────────────────────────────┘
```

| Trường | Kiểu | Ghi chú |
|---|---|---|
| Mã hóa đơn | HD###### | Auto-gen, immutable |
| Ngày bán | datetime | Có thể backdate khi tạo (cảnh báo nếu > 30 ngày) |
| Chi nhánh | FK | CN phát sinh hóa đơn |
| Kênh bán | enum | POS / Shopee / TikTok / Lazada / Trực tiếp tạo HD |
| Người bán | FK User | Nhân viên bán hàng (phục vụ hoa hồng) |
| Người tạo | FK User | Có thể khác người bán nếu quản lý tạo thay |
| Bảng giá | FK E06 | Bảng giá áp dụng |
| Ghi chú | text | Sửa được ở mọi trạng thái |

### 3.2 Thông tin khách hàng

```
┌─────────────────────────────────────────────────────┐
│  [Avatar] Nguyen Van An                     [Xem KH] │
│           0912345678  |  KH125055745                  │
│           No hien tai: 0d                             │
│           Tich diem lan nay: +312 diem                │
└─────────────────────────────────────────────────────┘
```

- Nếu "Khách lẻ": hiển thị placeholder + gợi ý "Nhập SĐT để tích điểm"
- Link "Xem KH" → mở Customer Detail panel song song

---

## 4. Panel trái — Dòng hàng

### 4.1 Cấu trúc bảng dòng hàng

| Cột | Kiểu | Ghi chú |
|---|---|---|
| Hình ảnh | thumbnail | Optional |
| Tên sản phẩm + SKU | string | Click → mở Product Detail |
| Đơn vị tính | E02 | Cái / Hộp / Thùng |
| Số lượng | decimal | |
| Đơn giá | decimal | Giá trước hoặc sau thuế (toggle được) |
| Giảm giá dòng | decimal hoặc % | |
| **VAT dòng** | enum | 0% / 5% / 8% / 10% — **khác nhau từng dòng** |
| Thành tiền | decimal | = SL × (đơn giá − giảm) |
| Serial / IMEI | string | Nếu SP track serial |
| Lô / HSD | FK E32 | Nếu SP track batch |

### 4.2 Toggle giá trước/sau thuế

Nút toggle góc trên bảng:

```
[Hien gia TRUOC thue ↔ SAU thue]
```

- **Trước thuế:** giá gốc, dòng VAT tách riêng
- **Sau thuế (mặc định):** gộp VAT vào đơn giá, dòng VAT = 0 khi hiển thị

> **Business rule BR-01:** Dữ liệu lưu trữ luôn là giá **trước thuế + VAT rate**. Toggle chỉ là hiển thị. Không cho phép lưu hóa đơn khi VAT rate không hợp lệ ({0, 5, 8, 10}%).

### 4.3 Xem lãi gộp (restricted — chỉ có quyền "Xem lãi gộp")

Nút toggle ẩn dành cho quản lý:

```
[Hien lai gop]  → Them cot: Gia von | Lai gop | Margin %
```

Mỗi dòng hiển thị thêm:
- Giá vốn WAC tại thời điểm bán
- Lãi gộp = Thành tiền − (SL × Giá vốn)
- Margin % = Lãi gộp / Thành tiền

Footer bảng: Tổng lãi gộp | Margin % tổng hóa đơn

> **Business rule BR-02:** Giá vốn trên hóa đơn là snapshot WAC tại thời điểm bán — không thay đổi dù giá vốn hiện tại thay đổi sau đó.

---

## 5. Panel phải — Tổng và Thanh toán

### 5.1 Cấu trúc phần tổng

```
Tong tien hang:           2.805.000
Giam gia hoa don:               (0)
─────────────────────────────────
Tam tinh:                 2.805.000

VAT phan tach theo muc:
  VAT 10% (tren 2.805.000):   280.500
  VAT 0%  (dich vu):               0
─────────────────────────────────
Thu khac (Phi dong goi):        15.000
Phi van chuyen:                 35.000
─────────────────────────────────
TONG CONG:                3.135.500
```

> **Business rule BR-03:** VAT phải phân tách theo từng mức thuế suất — không gộp chung "VAT" — theo yêu cầu TT78/2021. Ví dụ hóa đơn có 2 dòng VAT 10% và 1 dòng VAT 0% thì tách thành 2 dòng riêng.

### 5.2 Lịch sử thanh toán

```
┌──────────────────────────────────────────────────────┐
│  THANH TOAN                                          │
│  ─────────────────────────────────────────────────  │
│  21/05 11:35  Tien mat    3.200.000   TTHD016580 [↗] │
│  ─────────────────────────────────────────────────  │
│  Khach da tra:       3.200.000                       │
│  Khach can tra:      3.135.500                       │
│  Tien thua:             64.500 ✓                     │
│                                                      │
│  [+ Ghi nhan thanh toan them]  [Tao QR thanh toan]  │
└──────────────────────────────────────────────────────┘
```

**Trường hợp thanh toán từng phần (ghi nợ):**
```
  21/05 11:35  Tien mat    2.000.000   TTHD016580
  22/05 09:00  Chuyen khoan 1.000.000  TTHD016601
  ─────────────────────────────────────────────
  Da tra:      3.000.000
  Con no:        135.500  [Thu no]
```

### 5.3 Action panel

| Action | Điều kiện | Mô tả |
|---|---|---|
| **Ghi nhan thanh toan** | Còn nợ > 0 | Thêm phương thức thanh toán, số tiền |
| **Tao QR** | Còn nợ > 0 | Sinh QR chuyển khoản gắn nội dung "HD016580" |
| **Tra hang** | Hoàn thành hoặc Đang xử lý | Mở form tạo phiếu TH liên kết HD này |
| **Huy hoa don** | Đang xử lý hoặc Hoàn thành (nếu có quyền) | Cần xác nhận + lý do hủy |
| **Sao chep HD** | Always | Clone HD mới (tiện cho khách mua định kỳ) |

---

## 6. Tab HĐĐT

### 6.1 Trạng thái machine HĐĐT

```
[Chua phat hanh]
       │
       │[Click "Phat hanh HDDT"]
       ▼
[Dang xu ly] ──► timeout/loi ──► [Phat hanh loi]
       │                               │
       │[CQT phan hoi]                 │[Sua + gui lai]
       ▼                               │
  [Da gui CQT] ◄─────────────────────┘
       │
       ├──[Chap nhan]──► [Da phat hanh]
       │                      │
       │                      │[Gui cho KH]──► [Da chuyen]
       │                      │
       │                      └──[Sai sot]──► Quy trinh dieu chinh
       │
       └──[Tu choi]──► [Phat hanh loi] (xem §6.2)
```

### 6.2 Xử lý lỗi CQT từ chối

Khi CQT từ chối HĐĐT, hiển thị:

```
┌─────────────────────────────────────────────────────────────┐
│  ⚠ HDDT bi tu choi — Ma loi: MST_INVALID                    │
│  Ly do: Ma so thue nguoi mua khong hop le                   │
│                                                             │
│  Goi y sua:                                                 │
│  → Kiem tra lai MST tai: [https://tracuunnt.gdt.gov.vn]     │
│  → Dung so: 0123456789  |  Hien tai: 012345678X            │
│                                                             │
│  [Sua thong tin MST]  [Gui lai HDDT]  [Huy HDDT nay]       │
└─────────────────────────────────────────────────────────────┘
```

**Catalog lỗi thường gặp:**

| Mã lỗi | Ý nghĩa | Gợi ý sửa |
|---|---|---|
| MST_INVALID | MST người mua không tồn tại | Tra cứu lại MST trên cổng GDT |
| TAX_RATE_MISMATCH | Thuế suất không khớp loại hàng | Kiểm tra nhóm hàng + thuế suất |
| AMOUNT_MISMATCH | Tổng tiền trên HĐDT không khớp tờ khai | Kiểm tra làm tròn |
| SERIAL_DUPLICATE | Số serial HĐDT trùng | Liên hệ KV eInvoice |
| SIGN_ERROR | Lỗi ký số | Kiểm tra token USB-Token còn hạn |

### 6.3 Quy trình sau phát hành (sai sót, điều chỉnh, thay thế)

```
HDDT da phat hanh
    │
    ├── [Khach phat hien sai so tien]
    │       └──► Lap HD dieu chinh (Adjustment Invoice)
    │            → Goi y: Dieu chinh tang/giam tung dong
    │
    ├── [Sai thong tin khach hang / hang hoa nghiem trong]
    │       └──► Lap HD thay the (Replacement Invoice)
    │            → HD goc bi huy, so serial moi
    │
    └── [Huy hoa don truoc khi KH nhan]
            └──► Thong bao sai sot → CQT chap nhan → HDDT huy
```

### 6.4 Preview HĐĐT inline

Nút "Xem truoc HDDT" → render popup preview XML/PDF:

```
┌──────────────────────────────────────────────────────┐
│  HOA DON GIA TRI GIA TANG                            │
│  Mau so: 01GTKT0/001  |  Ky hieu: C25TAA             │
│  So: 0000016580       |  Ngay: 21/05/2025            │
│                                                      │
│  Nguoi ban: Cua hang Giay ABC                        │
│  MST nguoi ban: 0123456789                           │
│  Nguoi mua: Nguyen Van An                            │
│  MST nguoi mua: ...                                  │
│  ─────────────────────────────────────────────────  │
│  [Xem XML]  [Xem PDF]  [Phat hanh ngay]             │
└──────────────────────────────────────────────────────┘
```

---

## 7. Tab Giao hàng (chỉ hóa đơn loại Giao hàng)

### 7.1 Timeline vận đơn

```
Vận đơn GHN: GHN20250521123456  |  [Copy]  [Track trên GHN]

●  21/05 14:30  Đã tạo vận đơn — Bưu cục: GHN Q1
●  21/05 16:00  Chờ lấy hàng — nhân viên: Nguyen A
●  22/05 08:30  Lấy hàng thành công
●  22/05 10:15  Đang chuyển phát — Hub Tân Bình
○  22/05 15:00  Đang giao hàng — shipper: Tran B — 0987xxx
◌  (dự kiến)   Giao hàng thành công
```

### 7.2 Thông tin giao hàng

| Trường | Nội dung |
|---|---|
| Người nhận | Nguyen Van An — 0912345678 |
| Địa chỉ | 123 Nguyen Hue, P. Ben Nghe, TP.HCM |
| Carrier | GHN — Standard |
| Phí vận chuyển | 35.000d (Người bán trả) |
| COD | 3.120.500d (thu hộ) |
| Ghi chú giao | Gọi trước 30 phút khi đến |
| Trọng lượng / Kích thước | 800g / 30×20×15cm |

### 7.3 Báo giá đa carrier (khi tạo HD)

```
┌──────────────────────────────────────────────────────────────┐
│  Chon nha van chuyen                                         │
│  ──────────────────────────────────────────────────────────  │
│  ● GHN Express     35.000d  |  1–2 ngay  |  COD: Co        │
│  ○ GHTK Standard   28.000d  |  2–3 ngay  |  COD: Co        │
│  ○ J&T             32.000d  |  2–4 ngay  |  COD: Co        │
│  ○ Viettel Post    30.000d  |  3–5 ngay  |  COD: Co        │
│  ○ KShip           25.000d  |  1–3 ngay  |  COD: Co 🌟     │
│                             (first-party, uu dai cuoc)       │
└──────────────────────────────────────────────────────────────┘
```

### 7.4 Bằng chứng giao hàng

Kéo từ carrier API (nếu carrier hỗ trợ):

```
┌─────────────────────────────────────────────────────┐
│  Bang chung giao hang — GHN20250521123456           │
│  [Anh chup khi giao]  [Chu ky nguoi nhan]           │
│  Giao luc: 22/05/2025 15:42                         │
│  Nguoi nhan: Nguyen V. An (chu ky dien tu)          │
└─────────────────────────────────────────────────────┘
```

### 7.5 Luồng xử lý khi giao thất bại

```
Giao hang that bai
    │
    ├── [Giao lai] → Tao van don moi / chinh sua dia chi
    ├── [Hoan hang] → Cho ve kho (tu dong tao phieu TH khi "Da chuyen hoan")
    └── [Huy HD]   → Huy hoa don + hoan tien KH (neu da thu COD)
```

---

## 8. Tab Lịch sử & Liên quan

### 8.1 Audit log thay đổi hóa đơn

```
21/05 11:40  Tran Thi Lan  →  Tao hoa don — Thanh toan 3.200.000 tien mat
21/05 12:05  Nguyen Quan   →  Sua ghi chu: "Khach yeu cau goi rieng"
21/05 14:00  Admin         →  Cap nhat kho: Xac nhan xuat hang
```

### 8.2 Chứng từ liên quan

```
┌────────────────────────────────────────────────────────┐
│  Phieu thu        TTHD016580  — 3.200.000d  [Xem]     │
│  The kho (3 dong) HD016580    — xuat kho    [Xem]     │
│  Dat hang goc     DH000192    — 3 san pham  [Xem]     │
│  Van don          GHN2025...  — GHN Q1      [Track]   │
└────────────────────────────────────────────────────────┘
```

---

## 9. Action bar — Đầy đủ

### 9.1 Nhóm chính

| Action | Phím tắt | Điều kiện | Ghi chú |
|---|---|---|---|
| **In hoa don** | Ctrl+P | Always | Chọn mẫu in, số bản, máy in |
| **Chia se / Gui** | — | Always | Email / Zalo / SMS / copy link |
| **Sao chep** | — | Always | Clone HD mới |
| **Xuat file** | — | Always | PDF / Excel |

### 9.2 Nhóm nghiệp vụ

| Action | Điều kiện | Tác động |
|---|---|---|
| **Ghi nhan TT** | Còn nợ > 0 | Tạo TTHD mới, cập nhật nợ KH |
| **Tra hang** | Đang xử lý / Hoàn thành | Tạo phiếu TH liên kết |
| **Phat hanh HDDT** | Chưa phát hành + HD Hoàn thành | Gửi sang KV eInvoice |
| **Huy HDDT** | Đã phát hành chưa cấp | Thông báo sai sót CQT |
| **Huy HD** | Có quyền hủy | Hoàn tồn + hủy/giữ phiếu thu |
| **Mo lai HD** | Hoàn thành → Đang xử lý | Chỉ khi chưa phát hành HĐĐT |

### 9.3 Overflow menu ("...")

- Xem log thay đổi
- Gán nhãn / Tag nội bộ
- Thêm ghi chú nội bộ (không in ra)
- Chuyển khách hàng (gắn KH vào HD "Khách lẻ")

---

## 10. State machine tổng hợp

### 10.1 3 state machines song song

```
HD Status:         Dang xu ly ──► Hoan thanh
                                ╰──► Khong giao duoc
                   (bat ky)    ──► Da huy

Giao hang:         Cho xu ly ──► Cho lay ──► Lay hang OK ──►
                   Dang giao ──► Giao OK → [HD Hoan thanh]
                              ╰─► Giao that bai → [HD K.giao duoc]
                              ╰─► Chuyen hoan ──► Da chuyen hoan
                                              ↳ Auto-tao phieu TH

HDDT:             Chua PH ──► Dang xu ly ──► Da gui CQT ──►
                  Da PH ──► Da chuyen
                         ╰─► Loi ──► Sua lai
```

### 10.2 Ràng buộc chéo state machines

| Tình huống | Ràng buộc |
|---|---|
| HD Đã hủy | Không phát hành HĐĐT được |
| HĐĐT Đã phát hành | Không sửa HD trực tiếp — phải qua quy trình HĐĐT |
| Giao hàng Đang giao | Không hủy HD (phải wait hoặc yêu cầu carrier cancel) |
| Có Serial/Lô | Không backdate ngày hóa đơn |

---

## 11. Business Rules

| ID | Rule | Lý do |
|---|---|---|
| BR-01 | VAT rate chỉ nhận {0, 5, 8, 10}% theo TT78/2021 | Compliance thuế |
| BR-02 | Giá vốn trên HD là snapshot WAC lúc bán — immutable | Kế toán không retroactive |
| BR-03 | VAT phân tách theo từng mức thuế suất trong phần tổng | Yêu cầu HĐĐT TT78/2021 |
| BR-04 | HD Hoàn thành không sửa trực tiếp — phải Hủy + Tạo mới (hoặc versioning) | Audit trail |
| BR-05 | Không hủy HD khi HĐĐT đang Đang xử lý / Đã gửi CQT | Không thể thu hồi request |
| BR-06 | Không hủy HD khi vận đơn đang Đang giao / Chờ lấy hàng | Carrier đã nhận hàng |
| BR-07 | HD có Serial/Lô: không backdate thời gian hóa đơn | Batch tracking FIFO/FEFO sẽ vỡ |
| BR-08 | Tổng tiền trên HĐĐT phải khớp tổng HD (sai lệch do làm tròn ≤ 1đ được phép) | CQT validation |
| BR-09 | "Khách lẻ" không tích điểm, không cộng nợ | Không có identity để lưu |
| BR-10 | Phiếu thu TTHD liên kết 1:N với HD — 1 HD có thể có nhiều lần thanh toán | Partial payment flow |
| BR-11 | Hủy HD: tự động sinh reverse entry trên Thẻ kho + Sổ quỹ | Tính nhất quán hệ thống |
| BR-12 | Sửa số lượng/sản phẩm khi HD "Đang xử lý": phải kiểm tra lại tồn khả dụng | Tránh âm tồn |

---

## 12. Điểm đau & Cơ hội cải tiến (mở rộng từ hoa-don.md)

### 12.1 Thông tin & Hiển thị

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có toggle hiển thị lãi gộp inline — quản lý phải sang module Báo cáo | Toggle "Xem lai gop" restricted per role, hiển thị cột Giá vốn + Margin% ngay trên bảng dòng hàng |
| 2 | VAT phân tách không rõ ràng trên UI — 1 ô "VAT" gộp hết | Tách row: "VAT 10% (trên X)" / "VAT 5% (trên Y)" — như yêu cầu HĐĐT |
| 3 | Phụ phí "Thu khác" không có mô tả in ra hóa đơn — khách không hiểu | Thêm field "Mô tả in" cho từng loại Thu khác |
| 4 | Không hiển thị tổng điểm tích lũy của khách sau hóa đơn này | Hiển thị: "+312 điểm — Tổng: 4.850 điểm" ngay phần tổng |
| 5 | Không có mini preview mẫu in ngay trên chi tiết | Side button "In thử" → preview popup trước khi in thật |

### 12.2 Thanh toán & Nợ

| # | Pain | Đề xuất |
|---|---|---|
| 6 | QR tạo thủ công — không auto-detect khi tiền vào | Webhook từ ngân hàng/Momo → auto-confirm phiếu thu khi khớp nội dung "HD016580" |
| 7 | Không có nhắc nhở khi HD "Đang xử lý" + còn nợ > 0 quá X ngày | Cảnh báo SLA: hiện badge "Nợ 5 ngày" trên danh sách |
| 8 | Thanh toán đa phương thức ghi nhận tách rời nhiều TTHD — phức tạp đối soát | Group display: hiện tất cả trong 1 panel thanh toán, nhưng vẫn tạo TTHD riêng trong sổ quỹ |
| 9 | Không có tính năng "Thu nợ hàng loạt" cho nhiều HD cùng khách | Batch collection: chọn nhiều HD của 1 KH → thu 1 lần → split TTHD |

### 12.3 HĐĐT

| # | Pain | Đề xuất |
|---|---|---|
| 10 | Preview HĐĐT trước phát hành chỉ có trên module KV eInvoice riêng | Inline preview button ngay trên Tab HĐĐT: render XML → PDF |
| 11 | Khi CQT từ chối không giải thích lỗi bằng tiếng Việt dễ hiểu | Map error code → hướng dẫn tiếng Việt + gợi ý field cần sửa |
| 12 | Không phát hành HĐĐT hàng loạt từ danh sách | Chọn nhiều HD → "Phát hành HĐĐT (50)" → queue xử lý ngầm |
| 13 | Quy trình HĐ điều chỉnh / thay thế phải vào module riêng | Nút "Điều chỉnh HĐĐT" inline → wizard 3 bước ngay trên chi tiết HD |
| 14 | Không có cảnh báo khi ngày phát hành HĐĐT cách ngày HD > 1 ngày (nghi vấn audit) | Warning badge + ghi nhận lý do delay |

### 12.4 Giao hàng

| # | Pain | Đề xuất |
|---|---|---|
| 15 | Timeline vận đơn chỉ liệt kê text — không có visual timeline | Vertical timeline với icon + color theo trạng thái (đang giao = xanh, thất bại = đỏ) |
| 16 | Báo giá carrier phải ra màn hình riêng khi tạo đơn giao hàng | Inline pricing widget: nhập địa chỉ → hiện ngay 4–5 carrier + phí + ngày dự kiến |
| 17 | Không lưu bằng chứng giao hàng (ảnh, chữ ký) — tranh chấp không có căn cứ | Kéo POD (Proof of Delivery) từ GHN/GHTK API → lưu vào HD |
| 18 | Địa chỉ giao hàng vẫn còn field "Quận/Huyện" sau cải cách 01/07/2025 | Cập nhật: Province + Ward only (bỏ District field) theo NĐ202/2023 |

### 12.5 Quy trình & Audit

| # | Pain | Đề xuất |
|---|---|---|
| 19 | Hủy HD Hoàn thành = mất liên kết lịch sử (HD cũ không còn truy vết được từ HD mới) | Invoice Versioning: HD_v2 link "Thay thế HD_v1", có thể drill-down lịch sử version |
| 20 | Audit log quá ngắn gọn — không ghi field nào thay đổi, giá trị trước/sau | Field-level change log: "Số lượng: 2 → 3 lúc 14:05 bởi Nguyen A" |
| 21 | Không có note/comment nội bộ trên HD — ghi chú bị dùng nhầm (in ra HD) | Tách: Ghi chú in + Comment nội bộ (không in, chỉ team xem) |
| 22 | Không có alert khi cùng KH mở > N HD "Đang xử lý" cùng lúc | Duplicate check: cảnh báo "KH này đang có 3 HD chưa hoàn thành" |

---

## 13. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Invoice Versioning** — HD có v1/v2/v3 liên kết nhau thay vì Hủy+Tạo | Pain #19 — audit trail, compliance HĐĐT | Cao | Rất cao |
| 2 | **Inline HĐĐT Workflow** — preview, phát hành, sửa lỗi CQT, điều chỉnh/thay thế tất cả ngay trong chi tiết HD | Pain #10, #11, #13 — workflow liền mạch | Trung | Rất cao |
| 3 | **Smart Payment Webhook** — tự động khớp chuyển khoản/QR với HD còn nợ | Pain #6 — giảm công xác nhận thủ công 80% | Trung | Cao |
| 4 | **Visual Delivery Timeline + POD** — timeline đẹp + kéo bằng chứng ảnh/chữ ký từ carrier | Pain #15, #17 — UX + giảm tranh chấp | Thấp | Cao |
| 5 | **AI Upsell trên HD** — phân tích giỏ hàng + lịch sử KH → gợi ý sản phẩm add-on | Pain từ hoa-don.md #18 — tăng giá trị đơn hàng | Trung | Cao |

---

## 14. Tóm lược

**Chi tiết Hóa đơn phục vụ 4 vai trò:**
- **Thu ngân:** ghi nhận thanh toán, in hóa đơn, tạo QR thu nợ
- **Quản lý kho:** xác nhận giao hàng, theo dõi vận đơn, xử lý chuyển hoàn
- **Kế toán:** phát hành HĐĐT, xử lý lỗi CQT, đối soát sổ quỹ
- **Quản lý shop:** xem lãi gộp, theo dõi tồn đọng, tra soát sai sót

**Mạnh:** 3 state machines song song (HD / Giao hàng / HĐĐT), đa thuế suất per dòng, đa phương thức thanh toán, carrier integration đa nhà, HĐĐT theo TT78/2021

**Yếu:** Không versioning khi sửa HD Hoàn thành, HĐĐT workflow phải sang module riêng, không visual timeline giao hàng, không auto-confirm thanh toán QR, audit log quá đơn giản

**3 cải tiến ưu tiên nhất:**
1. Invoice Versioning thay cho Hủy+Tạo mới
2. HĐĐT workflow inline — giữ user trong 1 màn hình
3. Smart QR Payment Webhook — giảm 80% xác nhận thanh toán thủ công
