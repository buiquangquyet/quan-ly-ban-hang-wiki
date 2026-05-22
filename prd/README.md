# PRD — Quản Lý Bán Hàng (KiotViet-like)

**Cập nhật:** 19/05/2026
**Phạm vi:** Phân tích, thiết kế sản phẩm và kiến trúc hệ thống POS đa chi nhánh, đa kênh

---

## Cấu trúc tài liệu

### 00 — Tổng quan sản phẩm

| File | Mô tả |
|------|-------|
| [brainstorm-modules-moi.md](./00-tong-quan-san-pham/brainstorm-modules-moi.md) | 24 ý tưởng module mới phân theo 7 themes (AI, Multi-channel, Workforce, CRM, FinTech, B2B, Analytics). Ma trận ưu tiên Impact × Effort và Top 5 khuyến nghị. |

---

### 01 — POS & Màn hình bán hàng

| File | Mô tả |
|------|-------|
| [phan-tich-man-hinh-ban-hang.md](./01-pos-ban-hang/phan-tich-man-hinh-ban-hang.md) | Phân tích chi tiết 8 khu vực UI màn hình `/sale/`: Top bar, Cart panel, 3 chế độ bán (Nhanh / Thường / Giao hàng), Context panel, Bottom bar, Hamburger menu. Điểm mạnh, điểm yếu, cơ hội cải tiến. |
| [cac-doi-tuong-man-ban-hang.md](./01-pos-ban-hang/cac-doi-tuong-man-ban-hang.md) | 24 domain entities bóc tách từ UI POS: 4 nhóm (Giao dịch, Master data, Vận hành, Địa chính). Sơ đồ quan hệ, bảng tổng hợp lifetime và tần suất tạo. |

---

### 02 — Hàng hóa & Kho

| File | Mô tả |
|------|-------|
| [tong-quan-hang-hoa-ton-kho.md](./02-hang-hoa-kho/tong-quan-hang-hoa-ton-kho.md) | Toàn cảnh module Hàng hóa & Tồn kho: 4 nhóm entity (Product, Inventory, Nghiệp vụ kho, Integration). 6 nghiệp vụ kho, multi-unit, bảng giá, điểm mạnh/yếu, top 5 cơ hội module mới. |
| [the-kho-deep-dive.md](./02-hang-hoa-kho/the-kho-deep-dive.md) | Deep dive Thẻ kho (Stock Card): cấu trúc dữ liệu, quy tắc mã chứng từ (PREFIX + 6 digits), 13 loại giao dịch, schema `StockCardLine`, vấn đề `runningBalance`, 12 điểm UI thiếu, cơ hội đột phá (AI Anomaly, time-travel, real-time API). |
| [lo-han-su-dung-deep-dive.md](./02-hang-hoa-kho/lo-han-su-dung-deep-dive.md) | Deep dive Lô + Hạn sử dụng (Batch & Expiry): workflow đầy đủ qua 6 nghiệp vụ kho, data model `Batch` / `BatchStock`, logic FEFO, cảnh báo màu sắc, so sánh Lô vs Serial. 4 module mới đề xuất (Smart Recall, Auto Markdown, Expiry Heatmap, Compliance Audit Pack). |
| [serial-vs-image-data-model.md](./02-hang-hoa-kho/serial-vs-image-data-model.md) | Phân tích nơi lưu Ảnh sản phẩm và Serial/IMEI: ảnh ở Product (catalog data), serial ở entity riêng `StockUnit` (kết hợp Product link + Branch link). Data model 3 tầng: Product → Batch → StockUnit. |
| [nhap-hang-deep-dive.md](./02-hang-hoa-kho/nhap-hang-deep-dive.md) | Deep dive Nhập hàng (Goods Receipt): schema `PurchaseImport`, state machine 3 trạng thái, WAC merchant-level, phân bổ chi phí nhập (landed cost), liên thông Đặt hàng nhập + HĐĐT đầu vào. 19 pain points + Top 5 đột phá (OCR phiếu NCC, 3-way match, Smart Reorder, NCC self-service portal). |
| [tra-hang-deep-dive.md](./02-hang-hoa-kho/tra-hang-deep-dive.md) | Deep dive 2 nghiệp vụ trả hàng: Trả hàng bán (TH — KH trả, 3 mode: theo HĐ / nhanh / đổi trả, auto-trigger từ vận chuyển hoàn) và Trả hàng nhập (TPN — trả NCC). State machine, edge cases Serial/Lô, tác động loyalty + công nợ, 20 pain + Top 5 đột phá (Return Policy Engine, Fraud Detection AI, NCC Defect Dashboard, Auto HĐĐT điều chỉnh). |
| [ban-hang-deep-dive.md](./02-hang-hoa-kho/ban-hang-deep-dive.md) | Deep dive Bán hàng từ góc tồn kho (KHÔNG đi vào UX POS): DH/HD/TH ảnh hưởng tồn khác nhau, validation âm tồn, snapshot `costPriceAtSale`, race condition multi-POS, multi-UoM, Serial/Lô/Combo/BOM. 15 pain + Top 5 đột phá (Smart Stock Validation, Real-time Stock Engine, Cost Versioning, Smart Order Routing, HĐĐT-aware Inventory). |

---

### 03 — Kiến trúc kỹ thuật

| File | Mô tả |
|------|-------|
| [database-design.md](./03-kien-truc-ky-thuat/database-design.md) | Schema đầy đủ 45 bảng cho hệ thống POS đa chi nhánh: 12 module (Tenancy, Org, Catalog, Inventory, Pricing, Partners, Sales, Inv Transactions, Delivery, Promotion, Admin, Operations). ER diagram, key design decisions (multi-tenancy, denormalized stock, running balance, immutable ledger), indexes, scaling. |
| [sql-vs-mongodb.md](./03-kien-truc-ky-thuat/sql-vs-mongodb.md) | Phân tích 12 module: module nào dùng Postgres (tuyệt đối — tồn kho, hóa đơn, thanh toán), module nào hybrid Mongo (chat đa kênh, CDP, tracking, AI features), kiến trúc hybrid đề xuất theo 4 giai đoạn scale. |

---

### 04 — Sổ quỹ (Cash Flow)

| File | Mô tả |
|------|-------|
| [tong-quan-so-quy.md](./04-so-quy/tong-quan-so-quy.md) | Tổng quan module Sổ quỹ: 4 loại quỹ (Tiền mặt/NH/Ví/Tổng), 2 loại chứng từ (Phiếu thu/Phiếu chi), prefix mã (TTHD/TTDH/TTTH/CNH), state machine, flag "Hạch toán KQKD" tách dòng tiền với P&L, liên thông 6 module khác. 14 pain + Top 5 đột phá (Bank Reconciliation, Receipt OCR, Cash Flow Forecast, P&L Dashboard, Approval Workflow). |

---

### 05 — Khách hàng (Customer)

| File | Mô tả |
|------|-------|
| [tong-quan-khach-hang.md](./05-khach-hang/tong-quan-khach-hang.md) | Tổng quan module Khách hàng: schema Customer với identity + contact + sales aggregates + HĐĐT info (MST/CCCD/Hộ chiếu/Bank), 5 tab chi tiết (Thông tin / Địa chỉ nhận / Lịch sử bán-trả / Nợ cần thu / Tích điểm), filter "Ngày giao dịch cuối" cho churn detection, bulk message 4 kênh (ZNS/SMS/Email/Zalo OA). 23 pain + Top 5 đột phá (RFM Auto-Group, Customer 360 Unified, Churn AI, B2B Account Mode, Credit Limit + Auto Reminder). |
| [khach-hang-deep-dive.md](./05-khach-hang/khach-hang-deep-dive.md) | Deep dive THUẦN Customer (loại trừ Coupon/Voucher): schema 4 layer (Identity + Contact + Sales aggregates + HĐĐT), 5 tab inline + AR toolbox 5 actions, **rule engine Nhóm KH 12 trường điều kiện** với 3 mode + auto-execute (đủ build RFM), lifecycle derived từ lastTransactionAt. 29 pain chia 7 nhóm + Top 5 đột phá thuần Customer (Dedup phone, AR Aging + Credit Limit, Advanced Rule Engine OR/RFM preset, Lifecycle Stage, B2B Account multi-contact). |

---

### 06 — Đặt hàng / Đơn hàng (Customer Order)

| File | Mô tả |
|------|-------|
| [don-hang-deep-dive.md](06-don-hang/don-hang.md) | Deep dive Đặt hàng (DH######): soft commitment layer giữa "ý định mua" và "hóa đơn". Schema 11 cột giá per line (4 trường × trước/sau thuế), shipping block với carrier integration (Grab/GHN/Viettel/J&T với badge KiotViet), fulfillment tracking "1/0". State machine 4-5 trạng thái, filter "Thời gian giao hàng" có **Ngày mai/Tuần sau/Quý sau** (future-dated unique), đồng bộ kênh online (Shopee/TikTok/Lazada/Tiki/FB/Zalo), Gộp đơn, auto-Trả hàng khi chuyển hoàn. 19 pain + Top 5 đột phá (Smart Carrier Routing, Multi-shipment Fulfillment, Order Confirmation + Auto ZNS, Promo Stacking Engine, Webhook realtime sync). |

---

### 07 — Hóa đơn (Invoices)

| File | Mô tả |
|------|-------|
| [hoa-don-deep-dive.md](./07-hoa-don/hoa-don-deep-dive.md) | Deep dive Hóa đơn (HD######) — trung tâm 7 luồng dữ liệu. Schema multi-rate VAT (0/5/8/10%) trên 1 HĐ, **3 state machine song song** (HD status 4 trạng thái + Shipping status 7 trạng thái + HĐĐT status 2 enum × 10 states), 2 tab (Thông tin + Lịch sử giao hàng audit log), 8 actions inline (Hủy/Sao chép/Sửa/Trả hàng/In/Tạo QR thu nợ). KShip first-party carrier với OTP + cấn trừ cước. Workflow HĐĐT đầy đủ (TT78/2021): phát hành → CQT → điều chỉnh/thay thế. 24 pain + Top 5 đột phá (HĐ Versioning, Inline HĐĐT workflow, Smart Carrier Quote + POD, Bulk HĐĐT publish + auto-fix, Profit Insight + AI upsell). |

---

### 08 — Đặt hàng nhập (Purchase Order)

| File | Mô tả |
|------|-------|
| [dat-hang-nhap-deep-dive.md](./08-dat-hang-nhap/dat-hang-nhap-deep-dive.md) | Deep dive Đặt hàng nhập (DHN######) — **thuần nghiệp vụ**, không code/schema. Cầu nối cam kết shop ↔ NCC, không tác động tồn vật lý. 4 trạng thái workflow (Phiếu tạm → Đã xác nhận NCC → Nhập một phần → Hoàn thành). Mạnh: partial fulfillment (1 DHN qua N PN), cột "Số ngày chờ" cảnh báo NCC chậm, "Ngày nhập dự kiến" cho forecast, VAT per line, "Người nhận đặt" cho KAM thu mua, **"Đặt và gửi email" 1-click**, **tạo NCC inline**, **phím tắt F3/F8**, **tách 2 loại chi phí (NCC vs bên thứ 3)**, **setting "Giá nhập là giá vốn" per-phiếu**. 19 pain + Top 5 đột phá (Smart Reorder AI, 3-way Match + Vendor Scorecard, NCC Self-service Portal, PO Approval Workflow, Cash Flow Integration). |

---

### 09 — Giao vận (Đối tác giao hàng + Vận đơn)

| File | Mô tả |
|------|-------|
| [giao-van-deep-dive.md](./09-giao-van/giao-van-deep-dive.md) | Deep dive 2 module giao vận song hành — **moat lớn nhất của KiotViet** ở thị trường VN. **Partner Delivery** master + tài chính tổng + đối soát kỳ với 15-18 carrier tích hợp 2-way (GHN/GHTK/J&T/Grab/AhaMove/NinjaVan/BE/BEST/EMS/Viettel/GreenSM/KShip + variants FW); **Vận đơn** chi tiết per shipment với 7 trạng thái sync 2-way + 3 tab (Thông tin / Lịch sử / **Yêu cầu hỗ trợ** độc nhất). 6 KPI tài chính per partner (mini AR/AP với COD + cước), suffix **FW** tách đối soát forward, KShip first-party với OTP + cấn trừ wallet, Multi-package, Khai giá, Tự mang ra bưu cục. 20 pain + Top 5 đột phá (Smart Carrier Quote + Routing, Carrier Scorecard, Real-time COD Push + Auto-pay, Tracking Link + Auto-notify, POD + Delivery Rating). |

---

### 10 — Quản lý người dùng & Phân quyền

| File | Mô tả |
|------|-------|
| [quan-ly-nguoi-dung-deep-dive.md](10-quan-ly-nguoi-dung/quan-ly-nguoi-dung.md) | Deep dive User Manager + RBAC. **47 user** test data, 2 tab (Tài khoản / Vai trò), 3 role built-in (Quản trị chi nhánh / NV kho / NV thu ngân). **Multi-role assignment** ⭐ (1 user N role, permissions = UNION). Permission tree 3 tầng (Module → Sub-section → Resource → 8 actions) với **10 module top-level** + anchor menu + Ctrl+F search. **8 permission chuẩn per resource**: CRUD + Import + Xuất file + **Xem-sửa giá vốn** + **Xem-sửa giá nhập** (2 quyền nhạy cảm tách riêng — moat). **Phân quyền 3 chiều**: User × Chi nhánh × Nhóm hàng. Clone role, audit log riêng, bảo mật riêng module. 21 pain + Top 5 đột phá (Role Template + Diff + Versioning, ABAC + Separation of Duties, Magic Link Invite + Auto-expire, SSO + SCIM, Active Session + 4-eye Approval). |

---

### 11 — Nhân viên

| File | Mô tả |
|------|-------|
| [01-tong-quan.md](./11-nhan-vien/01-tong-quan.md) | Tổng quan module Nhân viên: entity Employee với thông tin nhân sự + tài khoản hệ thống, Ca làm việc (Shift scheduling), Hoa hồng theo doanh số (CommissionRule — % doanh thu / % lợi nhuận / theo SKU / bậc thang). Tích hợp Sổ quỹ khi chi hoa hồng. 7 pain + Top 5 đột phá (Smart Shift dự báo đông khách, Hoa hồng realtime trên app, Hiệu suất NV dashboard, Time-based access, Chấm công GPS/WiFi). |

---

### 12 — Bảng giá & Khuyến mãi

| File | Mô tả |
|------|-------|
| [01-tong-quan.md](./12-bang-gia-khuyen-mai/01-tong-quan.md) | Tổng quan module Bảng giá & Khuyến mãi: PriceBook multi-tier (lẻ/sỉ/VIP/kênh), quy tắc ưu tiên áp bảng, Promotion Engine (7 loại KM — % / số tiền / mua X tặng Y / combo / tặng hàng / ship / tích điểm nhân đôi), Coupon code + Voucher per KH, cơ chế stacking. Entity catalog BG01–BG06. 8 pain + Top 5 đột phá (Promotion Engine đầy đủ, Smart Stacking conflict detection, Flash Sale Scheduler, Voucher campaign tự động, Giá sàn enforcement nghiêm). |

---

### 13 — Báo cáo & Phân tích kinh doanh

| File | Mô tả |
|------|-------|
| [01-tong-quan.md](./13-bao-cao/01-tong-quan.md) | Tổng quan hệ thống báo cáo: 7 nhóm (Bán hàng / Lợi nhuận / Hàng hóa & Kho / Khách hàng / Tài chính & Sổ quỹ / Nhân viên / Nhà cung cấp), Dashboard 5 khu vực (KPI hôm nay / Biểu đồ / Cảnh báo tồn / Top hàng / Công nợ), dimension lọc chung (thời gian / CN / NV / kênh), export Excel/PDF + scheduled email. Kiến trúc materialized view cho báo cáo phức tạp. 8 pain + Top 5 đột phá (P&L Dashboard thực sự, AI Insight ngôn ngữ tự nhiên, Custom Dashboard, Cohort Retention, Smart Alert push). |

---

### 14 — Kênh bán hàng online & Đa kênh

| File | Mô tả |
|------|-------|
| [01-tong-quan.md](./14-kenh-ban-hang/01-tong-quan.md) | Tổng quan module Kênh bán hàng: Channel Account (Shopee / TikTok Shop / Lazada / Tiki / FB / Zalo OA / Instagram), Inventory Sync 2 chiều với buffer tồn tránh oversell, Order Pull workflow (webhook → DH nội bộ → fulfill → push tracking), Đối soát tiền sàn, Chat đa kênh hợp nhất (thread per conversation + lịch sử KH). Entity KBH01–KBH06. 8 pain + Top 5 đột phá (Real-time sync <5s, Smart Buffer, Unified Customer Identity Inbox 2.0, Cross-channel Reconciliation, Catalog Publisher 1-click). |

---

### 15 — Thuế & Kế toán

| File | Mô tả |
|------|-------|
| [01-tong-quan.md](./15-thue-ke-toan/01-tong-quan.md) | Tổng quan module Thuế & Kế toán: HĐĐT đầu ra (liên kết HD — TT78/2021), HĐĐT đầu vào (validate qua CQT, liên kết PN), các mức VAT (0/5/8/10%), báo cáo thuế GTGT (bảng kê 01-1/01-2 + tờ khai 01/GTGT), kỳ kê khai tháng/quý, đặc thù Hộ kinh doanh (Nghị định 70/2025 bắt buộc HĐĐT từ 2026), nhà cung cấp HĐĐT (VNPT/Viettel/FPT/MISA). Entity KT01–KT07. 7 pain + Top 5 đột phá (1-Click Tax Report + nộp XML HTKK, HĐĐT đầu vào auto-fetch, Compliance Dashboard, Smart Error Recovery, Export MISA). |

---

## Map nhanh: pain → tài liệu

| Bạn muốn tìm hiểu về... | Xem tài liệu |
|---|---|
| Ý tưởng module mới, roadmap | `00-tong-quan-san-pham/brainstorm-modules-moi.md` |
| UI/UX màn hình bán hàng | `01-pos-ban-hang/phan-tich-man-hinh-ban-hang.md` |
| Domain entities của POS | `01-pos-ban-hang/cac-doi-tuong-man-ban-hang.md` |
| Tổng quan module hàng hóa & kho | `02-hang-hoa-kho/tong-quan-hang-hoa-ton-kho.md` |
| Thẻ kho, mã chứng từ, audit trail | `02-hang-hoa-kho/the-kho-deep-dive.md` |
| Quản lý lô, hạn sử dụng, FEFO | `02-hang-hoa-kho/lo-han-su-dung-deep-dive.md` |
| Serial/IMEI tracking, ảnh sản phẩm | `02-hang-hoa-kho/serial-vs-image-data-model.md` |
| Nhập hàng từ NCC, giá vốn WAC, landed cost | `02-hang-hoa-kho/nhap-hang-deep-dive.md` |
| Trả hàng KH, trả NCC, đổi hàng, auto chuyển hoàn | `02-hang-hoa-kho/tra-hang-deep-dive.md` |
| Bán hàng tác động tồn kho, validation, cost snapshot | `02-hang-hoa-kho/ban-hang-deep-dive.md` |
| Database schema 45 bảng | `03-kien-truc-ky-thuat/database-design.md` |
| Khi nào dùng Postgres vs MongoDB | `03-kien-truc-ky-thuat/sql-vs-mongodb.md` |
| Sổ quỹ, phiếu thu/chi, dòng tiền, P&L flag | `04-so-quy/tong-quan-so-quy.md` |
| Khách hàng, công nợ AR, loyalty, segmentation | `05-khach-hang/tong-quan-khach-hang.md` |
| Đặt hàng, fulfillment, carrier integration, gộp đơn | `06-don-hang/don-hang-deep-dive.md` |
| Hóa đơn, HĐĐT compliance, multi-rate VAT, KShip, QR thu nợ | `07-hoa-don/hoa-don-deep-dive.md` |
| Đặt hàng nhập, PO workflow, NCC, partial fulfillment, lead time | `08-dat-hang-nhap/dat-hang-nhap-deep-dive.md` |
| Giao vận, đối tác giao hàng, vận đơn, COD, đối soát, KShip | `09-giao-van/giao-van-deep-dive.md` |
| User Manager, RBAC, phân quyền, role, multi-role, audit log | `10-quan-ly-nguoi-dung/quan-ly-nguoi-dung-deep-dive.md` |
| Nhân viên, ca làm việc, hoa hồng, hiệu suất bán hàng | `11-nhan-vien/01-tong-quan.md` |
| Bảng giá, khuyến mãi, coupon, voucher, tích điểm đổi điểm | `12-bang-gia-khuyen-mai/01-tong-quan.md` |
| Báo cáo bán hàng, lợi nhuận, kho, khách hàng, dashboard | `13-bao-cao/01-tong-quan.md` |
| Kênh bán hàng online, TMĐT sync, chat đa kênh, đối soát sàn | `14-kenh-ban-hang/01-tong-quan.md` |
| Thuế VAT, HĐĐT đầu ra/vào, báo cáo thuế, kế toán | `15-thue-ke-toan/01-tong-quan.md` |
| Chuyển hàng giữa chi nhánh, in-transit, chênh lệch | `02-hang-hoa-kho/08-chuyen-hang.md` |
| Kiểm kho, cycle count, đối chiếu thực tế vs sổ sách | `02-hang-hoa-kho/09-kiem-kho.md` |
| Sản xuất, BOM, pha chế, giá vốn thành phẩm, traceability NVL | `02-hang-hoa-kho/10-san-xuat.md` |
