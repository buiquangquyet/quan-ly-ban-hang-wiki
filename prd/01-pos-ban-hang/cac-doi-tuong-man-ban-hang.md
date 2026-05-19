# CÁC ĐỐI TƯỢNG TRÊN MÀN HÌNH BÁN HÀNG KIOTVIET

**Phương pháp:** Bóc tách từ UI → domain entities (data model)
**Mục đích:** Dùng làm input để thiết kế lại / mở rộng tính năng / build feature mới

---

## SƠ ĐỒ QUAN HỆ TỔNG QUAN

```
                          [Chi nhánh]
                              │
                              ▼
[Nhân viên] ─bán─► [Ca làm việc] ─chứa─► [Hóa đơn] ─thuộc─► [Kênh bán hàng]
                                            │
                            ┌───────────────┼─────────────────────┐
                            ▼               ▼                     ▼
                       [Chi tiết HĐ]   [Khách hàng]       [Thanh toán]
                            │               │                     │
                            ▼               ▼                     ▼
                       [Hàng hóa]    [Địa chỉ giao]      [Phương thức TT]
                            │
                            ▼
                       [Tồn kho]

                       [Khuyến mại/Voucher/Coupon] ──áp dụng──► [Hóa đơn]
                       [Hóa đơn điện tử] ─phát hành từ─► [Hóa đơn]
                       [Đơn giao hàng] ─sinh ra từ─► [Hóa đơn]
                       [Đối tác giao hàng] ─nhận─► [Đơn giao hàng]
                       [Trả hàng] ─tham chiếu─► [Hóa đơn]
```

---

## DANH SÁCH 24 ĐỐI TƯỢNG (cập nhật sau khi kiểm tra kỹ phần địa chỉ)

> **Cập nhật:** Đã bổ sung 4 đối tượng mình bỏ sót ở lần phân tích đầu — liên quan đến địa chỉ và gói hàng (đối tượng 11b, 21, 22, 23, 24).

### NHÓM 1 — Đối tượng giao dịch (Transaction entities)

#### 1. Hóa đơn (Invoice)
- **Vai trò:** Đơn vị giao dịch trung tâm — gắn liền 1 phiên bán
- **Thuộc tính:** mã HĐ, ngày giờ tạo, trạng thái (tạm/hoàn tất/hủy), tổng tiền hàng, giảm giá, VAT, thu khác, khách cần trả, ghi chú đơn hàng
- **Hiển thị:** Tab "Hóa đơn 1" trên top bar, có thể mở nhiều HĐ song song
- **Quan hệ:** thuộc 1 Chi nhánh, 1 Nhân viên, 1 Kênh bán, có nhiều Chi tiết HĐ, có thể có 1 Khách hàng, sinh 1 Đơn giao hàng (nếu chế độ Giao hàng), sinh 1 HĐĐT

#### 2. Chi tiết hóa đơn (Invoice Line Item)
- **Vai trò:** Mỗi dòng hàng trong cart
- **Thuộc tính:** STT, mã SP, tên SP, số lượng, đơn giá, chiết khấu dòng, thành tiền, ghi chú dòng, thuộc tính (size/màu...)
- **Hiển thị:** Cart panel bên trái, mỗi dòng có icon xóa/+/⋮
- **Quan hệ:** thuộc 1 Hóa đơn, tham chiếu 1 Hàng hóa

#### 3. Thanh toán (Payment)
- **Vai trò:** Ghi nhận tiền khách trả cho 1 hóa đơn
- **Thuộc tính:** số tiền khách thanh toán, tiền thừa trả lại, phương thức (Tiền mặt / Chuyển khoản / Thẻ / Ví), trạng thái
- **Hiển thị:** Khu tổng kết bên phải (Bán nhanh), 6 nút mệnh giá nhanh
- **Quan hệ:** thuộc 1 Hóa đơn, có thể có nhiều bản ghi (chia tách thanh toán)

#### 4. Đơn giao hàng (Shipment/Delivery)
- **Vai trò:** Gói thông tin vận chuyển cho hóa đơn cần ship
- **Thuộc tính:** loại dịch vụ, phí áp dụng, mã vận đơn, thời gian giao, trạng thái giao (Chờ xử lý/Đang giao/Đã giao...), thu hộ COD, trọng lượng, kích thước (DxRxC), số kiện
- **Hiển thị:** Context panel bên phải khi ở chế độ "Bán giao hàng"
- **Quan hệ:** thuộc 1 Hóa đơn, gửi qua 1 Đối tác giao hàng, đến 1 Địa chỉ giao

#### 5. Hóa đơn điện tử (E-Invoice)
- **Vai trò:** Chứng từ thuế phát hành sau giao dịch
- **Thuộc tính:** mẫu HĐĐT (MELYA, HĐ GTGT đại lý, MTT xăng dầu...), trạng thái phát hành, ký số, mã CQT
- **Hiển thị:** Popup từ icon "Mới" trên top bar, toggle "Tự động phát hành"
- **Quan hệ:** phát hành từ 1 Hóa đơn

#### 6. Trả hàng (Return)
- **Vai trò:** Khách trả lại hàng đã mua
- **Thuộc tính:** mã phiếu trả, mã HĐ gốc, danh sách SP trả, lý do, hoàn tiền/voucher
- **Hiển thị:** Menu ☰ → "Chọn hóa đơn trả hàng"
- **Quan hệ:** tham chiếu 1 Hóa đơn gốc

#### 7. Phiếu thu (Receipt voucher)
- **Vai trò:** Thu nợ khách hàng độc lập (không qua bán)
- **Thuộc tính:** số phiếu, khách, số tiền, ngày, lý do
- **Hiển thị:** Menu ☰ → "Lập phiếu thu"
- **Quan hệ:** thuộc 1 Khách hàng

#### 8. Đặt hàng (Pre-order)
- **Vai trò:** Đơn đặt trước, chưa xuất kho/thanh toán
- **Thuộc tính:** giống Hóa đơn nhưng status = đặt
- **Hiển thị:** Tab "+" → "Thêm mới đặt hàng", icon swap cạnh tab HĐ
- **Quan hệ:** sau khi xác nhận → chuyển thành Hóa đơn (Menu ☰ → "Xử lý đặt hàng")

---

### NHÓM 2 — Đối tượng tham chiếu (Master data)

#### 9. Hàng hóa (Product)
- **Vai trò:** Catalog sản phẩm
- **Thuộc tính:** mã SP, tên, giá, ảnh, thuộc tính (size/màu/biến thể), danh mục, đơn vị, barcode
- **Hiển thị:** Lưới SP có ảnh ở chế độ "Bán thường" (1/222 trang), kết quả search F3
- **Quan hệ:** xuất hiện trong Chi tiết HĐ, có 1 Tồn kho theo từng Chi nhánh

#### 10. Khách hàng (Customer)
- **Vai trò:** Người mua, có thể là khách lẻ hoặc khách thân thiết
- **Thuộc tính:** mã KH, tên, SĐT, email, địa chỉ, hạng/điểm tích lũy, công nợ
- **Hiển thị:** Ô "Tìm khách hàng (F4)" ở chế độ Bán thường / Giao hàng, icon "+" tạo mới
- **Quan hệ:** thuộc nhiều Hóa đơn, có nhiều Địa chỉ giao

#### 11. Địa chỉ giao hàng (Delivery Address — Destination)
- **Vai trò:** Địa chỉ NHẬN hàng của khách (điểm đến)
- **Icon trên UI:** 📍 pin XANH LÁ
- **Thuộc tính:**
  - Tên người nhận (string)
  - Số điện thoại (string)
  - Địa chỉ chi tiết (free text, vd "26 ngõ sỹ liên...")
  - Tỉnh/Thành phố (dropdown, FK → Đối tượng 21)
  - Phường/Xã (dropdown phụ thuộc tỉnh, FK → Đối tượng 22)
- **Hiển thị:** Khu giữa của chế độ "Bán giao hàng", chiếm 4 dòng
- **Validation:** Phường/Xã bắt buộc khi click Thanh toán (popup "Không hỗ trợ — Vui lòng nhập phường/xã người nhận")
- **Quan hệ:** thuộc 1 Đơn giao hàng (1-1), có thể save về 1 Khách hàng nếu khách đã tạo
- **⚠ Lưu ý:** Dropdown địa chỉ chính (icon ● XANH DƯƠNG ở trên) KHÔNG phải địa chỉ giao — đó là **Địa chỉ lấy hàng** (đối tượng 21 bên dưới)

#### 11b. Địa chỉ lấy hàng (Pickup Address — Origin) ⚠ Đối tượng mình bỏ sót lần trước
- **Vai trò:** Điểm shipper đến LẤY hàng — không phải địa chỉ giao
- **Icon trên UI:** ● pin XANH DƯƠNG (dòng đầu tiên trong khu giao hàng)
- **Thuộc tính:** mã, địa chỉ đầy đủ, SĐT người gửi, có phải địa chỉ chi nhánh không (flag "Tại chi nhánh")
- **Hiển thị:** Dropdown chọn từ list lưu sẵn, có nút "+ Thêm địa chỉ lấy hàng"
- **Use case thực tế:**
  - Chi nhánh chính (mặc định "Tại chi nhánh")
  - Kho ngoài (vd kho thuê dropship)
  - Nhà riêng chủ shop (ship từ nhà ngoài giờ)
  - Studio livestream
- **Quan hệ:** thuộc 1 Chi nhánh / 1 Cửa hàng (merchant-level master data), dùng cho 1 Đơn giao hàng

#### 12. Nhân viên bán hàng (Salesperson)
- **Vai trò:** Người thực hiện giao dịch, gắn doanh số/commission
- **Thuộc tính:** mã NV, tên, role/quyền, chi nhánh được phép bán
- **Hiển thị:** Dropdown "Admin ▼" trên header hóa đơn, user "shiptest" góc phải top bar
- **Quan hệ:** tạo nhiều Hóa đơn, thuộc 1 Ca làm việc

#### 13. Chi nhánh (Branch)
- **Vai trò:** Đơn vị bán hàng vật lý
- **Thuộc tính:** mã CN, tên, địa chỉ, kho gắn liền
- **Hiển thị:** "Chi nhánh 1 ▼" góc dưới phải bottom bar
- **Quan hệ:** chứa nhiều Hóa đơn, có nhiều Tồn kho theo SP

#### 14. Kênh bán hàng (Sales Channel)
- **Vai trò:** Đánh dấu nguồn doanh thu (offline trực tiếp / TikTok Shop / Shopee / Lazada...)
- **Thuộc tính:** mã kênh, tên, logo, loại (POS/TMĐT/MXH)
- **Hiển thị:** Dropdown logo (TikTok) bên cạnh "Admin ▼" trên header hóa đơn
- **Quan hệ:** mỗi Hóa đơn thuộc 1 Kênh, dùng để chia doanh thu trong báo cáo

#### 15. Đối tác giao hàng (Delivery Partner)
- **Vai trò:** Hãng vận chuyển (Viettel Post, GHN, GHTK, J&T...) hoặc shipper tự có
- **Thuộc tính:** mã, tên, loại dịch vụ (giao thường/hỏa tốc/COD), biểu phí
- **Hiển thị:** Dropdown "Đối tác giao hàng" ở chế độ Giao hàng, 2 tab "Cổng KiotViet ↔ Tự giao hàng"
- **Quan hệ:** xử lý nhiều Đơn giao hàng

#### 16. Phương thức thanh toán (Payment Method)
- **Vai trò:** Cách khách trả tiền
- **Thuộc tính:** mã, tên (Tiền mặt / Chuyển khoản / Thẻ / Ví), cấu hình (cổng thanh toán nếu là Ví/QR)
- **Hiển thị:** 4 radio button khu tổng kết, icon `⋮` để thêm phương thức/chia tách
- **Quan hệ:** dùng trong Thanh toán

#### 17. Khuyến mại / Voucher / Coupon (Promotion)
- **Vai trò:** Giảm giá áp lên Hóa đơn hoặc Chi tiết HĐ
- **Thuộc tính:** mã, loại (% / số tiền / mua X tặng Y), điều kiện, thời hạn
- **Hiển thị:** Ô "Mã coupon" trong tổng kết, menu ☰ → "Phát hành voucher"
- **Quan hệ:** áp dụng vào 1 Hóa đơn hoặc 1 dòng Chi tiết

---

### NHÓM 3 — Đối tượng vận hành (Operational entities)

#### 18. Ca làm việc (Shift / Session)
- **Vai trò:** Khoảng thời gian 1 nhân viên đăng nhập POS — dùng để kết ca, đối soát tiền mặt
- **Thuộc tính:** mã ca, nhân viên, thời gian mở/đóng, tiền đầu ca, tiền cuối ca
- **Hiển thị:** Menu ☰ → "Xem báo cáo cuối ngày" (Z-report)
- **Quan hệ:** chứa nhiều Hóa đơn, thuộc 1 Nhân viên × 1 Chi nhánh

#### 19. Tồn kho (Stock / Inventory)
- **Vai trò:** Số lượng hàng có sẵn theo SP × Chi nhánh
- **Thuộc tính:** SP, chi nhánh, số lượng khả dụng, số lượng đang đặt, ngưỡng cảnh báo
- **Hiển thị:** **Hiện không hiển thị trực tiếp** trên màn hình bán hàng (đây là 1 gap đã nêu trong phân tích trước)
- **Quan hệ:** thuộc 1 Hàng hóa × 1 Chi nhánh; giảm khi Hóa đơn được xác nhận

#### 20. Thuế (Tax)
- **Vai trò:** Tính VAT trên hóa đơn theo cấu hình SP/cấu hình HĐ
- **Thuộc tính:** mã thuế, tên (VAT 0/5/8/10%), kiểu (cộng vào / đã bao gồm)
- **Hiển thị:** Dòng "VAT" trong tổng kết, liên thông với cấu hình HĐĐT
- **Quan hệ:** áp lên Hóa đơn hoặc từng Chi tiết HĐ

---

### NHÓM 4 — Đối tượng địa chính (Master data hành chính) ⚠ Bổ sung

#### 21. Tỉnh/Thành phố (Province)
- **Vai trò:** Đơn vị hành chính cấp 1
- **Thuộc tính:** mã (theo chuẩn TCT/BNV), tên, loại (Tỉnh / Thành phố trực thuộc TW)
- **Hiển thị:** Dropdown ở dòng "Tỉnh/Thành phố" trong form địa chỉ giao
- **⚠ Lưu ý sau sáp nhập 2025:** KiotViet đã áp dụng hệ địa chính MỚI — chỉ còn **2 cấp** (Tỉnh + Phường/Xã). Cấp Quận/Huyện đã bị bỏ.
- **Quan hệ:** chứa nhiều Phường/Xã

#### 22. Phường/Xã (Ward / Commune)
- **Vai trò:** Đơn vị hành chính cấp 2 (sau sáp nhập 2025)
- **Thuộc tính:** mã, tên, loại (Phường / Xã / Thị trấn — đã rút gọn), tỉnh trực thuộc
- **Hiển thị:** Dropdown phụ thuộc Tỉnh đã chọn, hỗ trợ search (autocomplete)
- **Bắt buộc khi:** click "Thanh toán" trên đơn giao hàng
- **Quan hệ:** thuộc 1 Tỉnh

#### 23. Gói hàng (Package)
- **Vai trò:** Mô tả vật lý kiện hàng cho hãng vận chuyển tính phí
- **Thuộc tính:**
  - Số kiện (default 1)
  - Trọng lượng + đơn vị (gram/kg)
  - Kích thước Dài × Rộng × Cao + đơn vị (cm)
- **Hiển thị:** Khu giữa của chế độ Giao hàng, dòng có icon 📦
- **Tác động:** Hãng vận chuyển dùng max(trọng lượng thực, trọng lượng quy đổi DxRxC/6000) để tính phí
- **Quan hệ:** thuộc 1 Đơn giao hàng

#### 24. Ghi chú cho bưu tá (Shipper Note)
- **Vai trò:** Hướng dẫn cho shipper (gọi trước khi giao, gặp khách ở đâu, lưu ý hàng dễ vỡ...)
- **Thuộc tính:** free text
- **Hiển thị:** Dòng cuối cùng khu giao hàng, có icon ✏
- **Quan hệ:** thuộc 1 Đơn giao hàng
- **Khác:** Đây là note riêng cho shipper, KHÁC với "Ghi chú đơn hàng" (note nội bộ shop, không gửi shipper)

---

## BẢNG TỔNG HỢP (Cheat sheet)

| # | Đối tượng | Nhóm | Lifetime | Tần suất tạo |
|---|---|---|---|---|
| 1 | Hóa đơn | Giao dịch | Vĩnh viễn | Hàng ngày, rất cao |
| 2 | Chi tiết hóa đơn | Giao dịch | Vĩnh viễn | Rất cao |
| 3 | Thanh toán | Giao dịch | Vĩnh viễn | Cao |
| 4 | Đơn giao hàng | Giao dịch | Vĩnh viễn | Trung bình (chỉ có khi ship) |
| 5 | Hóa đơn điện tử | Giao dịch | Vĩnh viễn | Cao (auto) |
| 6 | Trả hàng | Giao dịch | Vĩnh viễn | Thấp |
| 7 | Phiếu thu | Giao dịch | Vĩnh viễn | Thấp |
| 8 | Đặt hàng | Giao dịch | Đến khi xử lý | Trung bình |
| 9 | Hàng hóa | Master | Lâu dài | Thấp (setup) |
| 10 | Khách hàng | Master | Lâu dài | Trung bình |
| 11 | Địa chỉ giao | Master | Theo KH | Thấp |
| 12 | Nhân viên | Master | Lâu dài | Rất thấp |
| 13 | Chi nhánh | Master | Lâu dài | Rất thấp |
| 14 | Kênh bán hàng | Master | Lâu dài | Rất thấp |
| 15 | Đối tác giao hàng | Master | Lâu dài | Rất thấp |
| 16 | Phương thức TT | Master | Lâu dài | Rất thấp |
| 17 | Khuyến mại | Master | Có hạn | Trung bình |
| 18 | Ca làm việc | Vận hành | 1 ngày | Theo ca |
| 19 | Tồn kho | Vận hành | Realtime | Cập nhật liên tục |
| 20 | Thuế | Master | Lâu dài | Rất thấp |
| 11b | Địa chỉ lấy hàng | Master | Lâu dài | Thấp |
| 21 | Tỉnh/Thành phố | Master (chuẩn HC) | Vĩnh viễn | Không (catalog) |
| 22 | Phường/Xã | Master (chuẩn HC) | Vĩnh viễn | Không (catalog) |
| 23 | Gói hàng | Giao dịch | Vĩnh viễn | Cao (khi ship) |
| 24 | Ghi chú bưu tá | Giao dịch | Vĩnh viễn | Trung bình |

---

## GỢI Ý SỬ DỤNG DANH SÁCH NÀY

1. **Thiết kế feature mới:** đối chiếu xem feature đụng vào những object nào → ước lượng impact và edge cases.
2. **Data model review:** map từ object → bảng/collection trong DB hiện tại, xác định nơi data scattered.
3. **API design:** mỗi object nên có CRUD endpoint riêng + endpoint phục vụ workflow (vd `POST /invoices/{id}/payments`).
4. **Quyền (RBAC):** mỗi object cần ma trận quyền theo role (Cashier không thấy được giá vốn, Manager thấy được...).
5. **Audit log:** đối tượng giao dịch (nhóm 1) cần immutable log đầy đủ.
6. **Brainstorm tiếp:** mỗi object là 1 "hook" để gắn feature mới — ví dụ:
   - Khách hàng → Loyalty, CDP, AI recommendation
   - Hàng hóa → AI pricing, demand forecast, image generation
   - Đơn giao hàng → Smart routing, COD reconciliation
   - Ca làm việc → Workforce, anomaly detection
