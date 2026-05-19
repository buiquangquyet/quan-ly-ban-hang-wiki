# PHÂN TÍCH MÀN HÌNH BÁN HÀNG KIOTVIET (/sale/)

**URL:** https://testzone17.kiotviet.com/sale/#/
**Ngày phân tích:** 18/05/2026
**User test:** shiptest (role Admin), Chi nhánh 1

---

## 1. CẤU TRÚC TỔNG QUAN

Màn hình chia thành 5 khu vực lớn:

```
┌─────────────────────────────────────────────────────────────────┐
│  ① TOP BAR — Tìm hàng | Tab hóa đơn | Tab "+" | Action icons   │
├──────────────────┬─────────────────────┬────────────────────────┤
│                  │                     │                        │
│  ② CART PANEL    │  ③ CUSTOMER /       │  ④ CONTEXT PANEL       │
│  (Danh sách      │     PRODUCT GRID    │  (đổi nội dung theo    │
│   hàng đang      │     (đổi theo chế   │   chế độ bán)          │
│   bán)           │     độ bán)         │                        │
│                  │                     │                        │
├──────────────────┴─────────────────────┴────────────────────────┤
│  ⑤ BOTTOM BAR — Mode switcher | Chi nhánh | Hỗ trợ | Settings  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. CHI TIẾT TỪNG ĐỐI TƯỢNG

### ① TOP BAR

| Đối tượng | Vai trò |
|---|---|
| **Ô tìm hàng hóa (F3)** | Search nhanh sản phẩm theo tên/mã/barcode. Phím tắt F3. |
| **Icon cân/scale** (bên phải ô tìm) | Nhập trọng lượng từ cân điện tử kết nối POS. |
| **Icon barcode quét** | Quét mã vạch (USB scanner hoặc camera). |
| **Tab "Hóa đơn 1"** | Hóa đơn đang active. Click `×` để đóng. Có thể mở nhiều hóa đơn song song để phục vụ nhiều khách cùng lúc (đặc trưng quán nhậu/F&B). |
| **Icon swap/đổi** (cạnh tên tab) | Chuyển nhanh giữa Hóa đơn ↔ Đặt hàng. |
| **Tab "+"** | Mở dropdown: "Thêm mới đặt hàng" — tạo invoice mới hoặc tạo đơn đặt hàng (preorder). |
| **Icon hóa đơn điện tử** (badge "Mới") | Cấu hình mẫu HĐĐT (MELYA, HĐ GTGT MTT xăng dầu, đại lý...) — auto phát hành sau thanh toán. **Hot feature do quy định 2026.** |
| **Icon khóa** | Khóa màn hình tạm thời (đổi ca, đi vệ sinh) — bảo mật khi cashier vắng mặt. |
| **Icon undo (↶)** | Hoàn tác thao tác gần nhất. |
| **Icon refresh (↻)** | Đồng bộ dữ liệu với server (nếu offline-online). |
| **Icon máy in** | In hóa đơn / phiếu tạm. |
| **Tên user "shiptest"** | Đang đăng nhập. Hover/click → menu user. |
| **Hamburger menu (☰)** | Menu chức năng phụ — xem chi tiết ở mục ⑥. |

---

### ② CART PANEL — Danh sách hàng đang bán

Mỗi dòng sản phẩm có:

| Phần tử | Vai trò |
|---|---|
| **Số thứ tự** (1, 2…) | Đánh số dòng. Ở chế độ "Bán giao hàng" có thể ẩn. |
| **Icon xóa (🗑)** | Xóa dòng khỏi cart. |
| **Mã SP** (SP000015) | Mã hàng nội bộ. |
| **Tên hàng** | Tên hiển thị. |
| **Số lượng** (ô input) | Edit trực tiếp. Phím +/− trên bàn phím để tăng giảm. |
| **Đơn giá** | Có thể click để chỉnh giá (nếu nhân viên có quyền). |
| **Thành tiền** | SL × đơn giá. Tự động tính. |
| **Icon "+"** (mỗi dòng) | Mở popover chi tiết: chiết khấu dòng, ghi chú, thuộc tính... |
| **Icon "⋮"** (3 chấm) | Menu thao tác dòng: Sao chép, Áp giá khác, Tách dòng, Áp mã giảm... |
| **Ô "Ghi chú đơn hàng"** | Note chung cho cả hóa đơn (vd "khách lấy gấp"). |

---

### ③ KHU GIỮA — Đổi theo chế độ bán

#### Chế độ "Bán nhanh" (Quick Sale)
Không có panel khách hàng riêng — tối ưu cho POS bán lẻ truyền thống, scan-pay-go.

#### Chế độ "Bán thường" (Standard Sale)
- **Ô "Tìm khách hàng (F4)"** + icon "+" để tạo KH mới.
- **Lưới sản phẩm có ảnh** — kéo lên cart bằng click. Phân trang `1/222`.
- **3 icon view** (góc phải trên lưới): danh sách | filter | xem ảnh.
- **Nút phân trang ‹ ›**
- Phù hợp shop thời trang / mỹ phẩm cần xem ảnh khi tư vấn.

#### Chế độ "Bán giao hàng" (Delivery Sale)
- **Khu khách hàng:** Tìm KH (F4), tự động fill địa chỉ.
- **Khu địa chỉ giao:**
  - Địa chỉ chính (có thể chọn từ dropdown các địa chỉ đã lưu)
  - Tên người nhận / SĐT
  - Địa chỉ chi tiết / Phường-Xã / Thành phố
- **Khu thông tin gói hàng:**
  - Số kiện (mặc định 1)
  - Trọng lượng (gram) + kích thước (DxRxC cm)
  - Ghi chú cho bưu tá
- Phù hợp seller online ship qua đối tác vận chuyển.

---

### ④ CONTEXT PANEL (Phải) — Đổi theo chế độ

#### "Bán nhanh"
**Tổng kết hóa đơn + Thanh toán nhanh:**
- Tổng tiền hàng / Giảm giá / Mã coupon / VAT / Thu khác / **Khách cần trả**
- Khách thanh toán (mặc định = "Khách cần trả")
- 4 radio: **Tiền mặt | Chuyển khoản | Thẻ | Ví**
- 6 nút mệnh giá nhanh (115k, 116k, 120k, 150k, 200k, 500k) — auto round-up từ "Khách cần trả"
- Tiền thừa trả khách
- Icon `⋮` cạnh radio → thêm phương thức/chia tách thanh toán

#### "Bán thường"
- Trên: ô tìm KH (F4) + add KH mới
- Dưới: lưới sản phẩm

#### "Bán giao hàng" — Có 2 tab
1. **Cổng KiotViet** — đẩy đơn qua hệ thống vận chuyển KiotViet ký kết (Viettel Post, GHN, GHTK, J&T…)
2. **Tự giao hàng** — chủ shop dùng đối tác riêng
   - Đối tác giao hàng (dropdown + icon edit/add)
   - Loại dịch vụ (Giao thường…)
   - Phí áp dụng / Mã vận đơn / Thời gian giao / Trạng thái giao

---

### ⑤ BOTTOM BAR

| Đối tượng | Vai trò |
|---|---|
| **Tab "Bán nhanh" ⚡** | Mode lightning — chỉ search + scan + pay. |
| **Tab "Bán thường" 🕐** | Mode tư vấn — có lưới sản phẩm với ảnh. |
| **Tab "Bán giao hàng" 🚚** | Mode ship — full thông tin giao hàng. |
| **Tổng kết "Tổng tiền hàng / Khách cần trả"** | Hiển thị ngắn gọn ngay bottom bar khi ở Bán thường. |
| **Nút "IN"** | In nháp / in tạm hóa đơn chưa thanh toán. |
| **Nút "THANH TOÁN" (xanh lớn)** | Hoàn tất giao dịch — bước cuối CTA chính. |
| **Chi nhánh đang đăng nhập** (📍 Chi nhánh 1) | Chuyển chi nhánh trong cùng tài khoản (dropdown). |
| **1900 6522** | Hotline KiotViet. |
| **Icon "?"** | Trợ giúp / hướng dẫn. |
| **Icon settings** | Cài đặt nhanh máy bán (máy in, két, cân, scanner). |

---

### ⑥ HAMBURGER MENU (☰) — Chức năng phụ

| Chức năng | Mục đích |
|---|---|
| **Xem báo cáo cuối ngày** | Kết ca / Z-report — tổng kết doanh thu ca làm. |
| **Xử lý đặt hàng** | Chuyển preorder thành hóa đơn xuất kho. |
| **Chọn hóa đơn trả hàng** | Khách đem hàng đổi/trả → tạo phiếu trả. |
| **Lập phiếu thu** | Thu nợ KH (không qua hóa đơn bán). |
| **Phát hành voucher** | Tạo voucher quà tặng / phiếu mua hàng. |
| **Import file** | Bulk thêm sản phẩm vào cart từ Excel. |
| **Tùy chọn hiển thị** | Bật/tắt cột, layout, font size. |
| **Phím tắt** | Cheat sheet keyboard shortcuts. |
| **Quản lý** | Mở tab `/man/` (backend quản lý). |
| **Đăng xuất** | Logout. |

---

### ⑦ FORM TỔNG KẾT HÓA ĐƠN (Áp dụng mọi chế độ)

| Trường | Ghi chú |
|---|---|
| **Tổng tiền hàng** | Tổng raw chưa giảm/thuế. |
| **Giảm giá** | Discount tay (số tiền hoặc %). |
| **Mã coupon** | Voucher KiotViet hoặc tự tạo. |
| **VAT** | Thuế GTGT. Mặc định 0 nếu không bật thuế. |
| **Thu khác** | Phụ phí (vận chuyển, dịch vụ…). Có icon `ⓘ` show breakdown. |
| **Khách cần trả** | Số tiền cuối — màu xanh dương nổi bật. |
| **Khách thanh toán** | Số khách đưa — tính tiền thừa. |
| **Thu hộ tiền (COD)** | Toggle, chỉ ở chế độ Giao hàng. Mặc định ON với COD. |

---

### ⑧ HEADER HÓA ĐƠN (Trên cart)

| Đối tượng | Vai trò |
|---|---|
| **Admin ▼** (nhân viên bán) | Chọn người bán → quy commission đúng người. |
| **Logo TikTok ▼** (kế bên Admin) | **Chọn kênh bán hàng** (Trực tiếp / TikTok Shop / Shopee / Lazada…) — quan trọng để chia doanh thu theo kênh trong báo cáo. |
| **Ngày/giờ** (18/05/2026 15:28) | Edit nếu xuất hóa đơn cho ngày khác (cần quyền). |

---

## 3. ĐÁNH GIÁ THIẾT KẾ

### Điểm mạnh
- **3 chế độ bán độc lập** rất tinh tế — UX khác nhau cho 3 persona (cashier siêu thị, sales tư vấn, online seller).
- **Switch kênh bán hàng inline** (icon TikTok) cực kỳ practical cho seller đa kênh.
- **Tab nhiều hóa đơn cùng lúc** — F&B/quán nhậu không bị nghẽn.
- **Phím tắt F3/F4** + barcode + cân integration → workflow nhanh cho cashier có kinh nghiệm.
- **Hóa đơn điện tử inline** ngay top bar — đón đầu quy định 2026.

### Điểm yếu / Cơ hội cải tiến
1. **Quá nhiều icon nhỏ trên top bar (10 icon)** — onboard cashier mới sẽ vất vả. Đề xuất: gom các icon ít dùng vào ☰.
2. **Tab hóa đơn không hiển thị tên khách / tổng tiền** — khi 5 tab cùng mở, dễ click nhầm. Đề xuất: tooltip có summary.
3. **Mode switcher ở bottom** đôi khi bị click nhầm khi cashier reach down keypad. Đề xuất: cho cài mode mặc định theo branch + ẩn 2 mode còn lại.
4. **Khu "Khách thanh toán" với 6 nút mệnh giá cứng** (115k–500k) — nên auto-suggest theo "Khách cần trả" (vd 114,305 → 115k, 120k, 200k). Tốt lắm, nhưng số 116,000 hơi lạ — có vẻ là logic round +1k.
5. **Không có quick-action "Áp khuyến mại"** ngay UI — phải đi qua menu dòng. Với chuỗi F&B chạy combo, đây là thao tác hằng ngày.
6. **Không thấy hiển thị "Stock realtime"** trên lưới sản phẩm (Bán thường) — nhân viên có thể bán item OOS. Đề xuất: badge số tồn trên thumbnail.
7. **Customer info ở Bán nhanh bị ẩn hoàn toàn** — nếu khách bất ngờ muốn tích điểm, phải đổi mode. Đề xuất: collapse-able sidebar.
8. **COD toggle** chỉ là toggle không có hint "phí thu hộ x% sẽ áp lên đâu" — confusion với người mới.
9. **Không có nút "Giữ hóa đơn / Park sale"** rõ ràng — phải dùng + để mở tab mới, hóa đơn cũ vẫn ở tab cũ. Concept "park" chuẩn POS quốc tế bị thiếu.
10. **Lưới sản phẩm 222 trang** mà không có grouping theo category trên UI — phụ thuộc search. Nên có sidebar category collapsable.

### Cơ hội product mới gắn với màn hình này

| Cơ hội | Mức độ |
|---|---|
| **Voice ordering** ("Cho 2 cốc Latte size L") để cashier rảnh tay | High delight |
| **Suggest cross-sell** ngay khi add 1 SKU vào cart ("Khách mua kem dưỡng thường mua thêm…") | High impact |
| **AI tự nhận diện khách qua camera** + auto-fill khách hàng | Wow factor |
| **Auto-apply best combo voucher** | Reduce friction, tăng AOV |
| **Drag-drop reorder hàng trong cart** cho hóa đơn dài (F&B) | UX polish |
| **Floating "tax preview"** show HĐĐT sẽ ra như nào trước khi click Thanh toán | Compliance |

---

## 4. TÓM TẮT CHO BRAINSTORM

Màn hình bán hàng KiotViet đã rất đầy đủ về tính năng (multi-mode, multi-channel, multi-payment, e-invoice, delivery integration). Hướng đột phá tiếp theo nên là **giảm cognitive load cho cashier** và **AI assist** ngay trong workflow bán hàng:

1. Bottom-up UX cleanup (gom icon, tooltip thông minh, mode auto-select theo persona)
2. AI cross-sell + voice ordering
3. Camera identify customer (cho chuỗi F&B/mỹ phẩm có loyalty)
4. Stock-aware product grid (badge tồn kho realtime)

Đây cũng là các điểm có thể đưa vào mini-PRD nếu bạn muốn đào sâu.
