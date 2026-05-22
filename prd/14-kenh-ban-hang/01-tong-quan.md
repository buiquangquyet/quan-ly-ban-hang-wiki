# Tổng quan — Kênh bán hàng online & Đa kênh

**Phạm vi:** Module quản lý kết nối và vận hành bán hàng trên các kênh online — TMĐT (Shopee, TikTok Shop, Lazada, Tiki), mạng xã hội (Facebook, Zalo OA, Instagram), và chat đa kênh. Đây là moat cạnh tranh quan trọng phân biệt POS đa kênh với POS truyền thống.
**Liên quan:** Module 01 (POS), Module 02 (Tồn kho — đồng bộ tồn), Module 06 (Đơn hàng — nhận đơn online), Module 09 (Giao vận — ship đơn online)

---

## 1. Bức tranh toàn cảnh kiến trúc đa kênh

```
                          ┌─────────────────────┐
                          │   KiotViet Core      │
                          │  (Inventory + Orders)│
                          └──────────┬──────────┘
                                     │ 2-way sync
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                       ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │  TMĐT Platforms │   │  Social Commerce│   │   Chat Channels │
    │                 │   │                 │   │                 │
    │ • Shopee        │   │ • Facebook Shop │   │ • Facebook Msg  │
    │ • TikTok Shop   │   │ • Zalo OA       │   │ • Zalo Chat     │
    │ • Lazada        │   │ • Instagram     │   │ • Shopee Chat   │
    │ • Tiki          │   │ • Livestream    │   │ • TikTok DM     │
    └─────────────────┘   └─────────────────┘   └─────────────────┘
```

**3 luồng chính:**
1. **Catalog sync:** Hàng hóa, giá, ảnh → đẩy lên sàn
2. **Inventory sync:** Tồn kho thực tế → đồng bộ 2 chiều, tránh oversell
3. **Order pull:** Đơn từ sàn về KiotViet → xử lý fulfill → push tracking

---

## 2. Quản lý gian hàng (Channel Account — KBH01)

### 2.1 Kết nối gian hàng

Mỗi gian hàng là một **channel account** được kết nối qua OAuth/API key:

| Trường | Ghi chú |
|---|---|
| Tên gian hàng | VD: "Shopee — Thời Trang Huy" |
| Loại kênh | Shopee / TikTok Shop / Lazada / Tiki / Facebook / Zalo OA |
| Tên shop trên sàn | Lấy từ API sau khi kết nối |
| Chi nhánh phụ trách | Đơn từ kênh này → chuyển về kho CN nào |
| Trạng thái kết nối | Đang kết nối / Mất kết nối / Hết token |
| Lần sync cuối | Timestamp lần cuối đồng bộ thành công |
| Cấu hình đồng bộ | Tồn / Giá / Đơn hàng — bật/tắt từng loại |

**Giới hạn:**
- 1 tài khoản KiotViet có thể kết nối nhiều gian hàng (nhiều Shopee, nhiều TikTok)
- 1 SP nội bộ ↔ N listing trên N gian hàng (ChannelMapping — E36)

### 2.2 Liên kết sản phẩm (Channel Mapping — E36)

Mỗi SP nội bộ cần được **map** với listing trên sàn:

| Trạng thái mapping | Ý nghĩa |
|---|---|
| Đã liên kết | SP nội bộ ↔ SKU sàn — tồn/giá đồng bộ |
| Chưa liên kết | Listing trên sàn chưa link SP nào → đơn về không map được tồn |
| Không đồng bộ tồn | Map nhưng tắt sync tồn (VD: hàng order riêng) |

**Bulk mapping:**
- Import Excel danh sách mapping
- Auto-suggest bằng tên SP + mã barcode

---

## 3. Đồng bộ tồn kho (Inventory Sync — KBH02)

Đây là tính năng **quan trọng nhất** — tránh oversell khi bán đồng thời nhiều kênh.

### 3.1 Luồng sync tồn

```
[POS bán 1 cái] → onHand giảm → KiotViet tính available
       → Push tồn mới lên Shopee/TikTok/Lazada (async, <30s)
       → Shopee hiển thị tồn mới
```

```
[Shopee bán 1 cái] → KiotViet nhận webhook → trừ tồn internal
       → Push tồn mới lên các sàn còn lại
```

### 3.2 Cấu hình buffer tồn

**Vấn đề:** Nếu sync 100% tồn thực tế → khi 2 sàn cùng nhận đơn 1 lúc → tồn âm.

**Giải pháp: Buffer tồn:**
- `available_for_online = onHand - buffer`
- Buffer cấu hình per SP hoặc per gian hàng (VD: giữ lại 3 cái POS)
- Buffer theo % tồn (VD: giữ 20% cho POS)

### 3.3 Conflict resolution

Khi tồn âm xảy ra:
1. Hủy đơn sàn sau (theo timestamp)
2. Alert chủ shop để xử lý thủ công
3. Tự động raise trả hàng trên sàn

---

## 4. Nhận & xử lý đơn hàng online (Order Pull — KBH03)

### 4.1 Luồng đơn từ sàn về KiotViet

```
Sàn confirm đơn
    → Webhook đến KiotViet
    → Tạo Đặt hàng (DH) nội bộ
    → Map sản phẩm (via E36)
    → Gán chi nhánh fulfillment
    → NV xử lý pick & pack
    → Tạo vận đơn
    → Push tracking code lên sàn
    → Sàn tự động xác nhận giao
```

### 4.2 Trạng thái đơn hàng từ sàn

| Trạng thái sàn | Tương ứng KiotViet | Hành động |
|---|---|---|
| Chờ xác nhận | DH — Phiếu tạm | Xác nhận trong 24h hoặc auto-cancel |
| Đã xác nhận | DH — Đã xác nhận | Chuẩn bị hàng |
| Đang giao hàng | DH — Đang xử lý | Theo dõi vận đơn |
| Đã giao | HD — Hoàn thành | Hoàn tất |
| Hoàn hàng | TH — tạo tự động | Xử lý trả hàng |

### 4.3 Đối soát đơn TMĐT

Cuối kỳ (thường 15 ngày / tháng):
- Sàn trả tiền COD + thanh toán online vào tài khoản ngân hàng
- KiotViet đối soát: Tiền nhận được ↔ Tổng đơn hoàn thành − Phí sàn − Phí vận chuyển
- Báo cáo đối soát theo kênh / gian hàng / kỳ

---

## 5. Marketing đa kênh (KBH04)

### 5.1 Tự động đẩy hàng (Auto Marketing — E37)

- Lên lịch đẩy sản phẩm lên top feed trên Shopee/TikTok theo giờ
- Cấu hình per gian hàng: tần suất, nhóm SP, ngân sách

### 5.2 Đồng bộ giá theo kênh

- Bảng giá TikTok Shop riêng (link với module 12 — Bảng giá)
- Mỗi sàn có thể có giá khác nhau cho cùng 1 SKU
- Thay đổi giá nội bộ → push lên sàn tự động (có confirmation)

### 5.3 Catalog đồng bộ

- Cập nhật tên, mô tả, ảnh SP nội bộ → sync lên listing sàn
- Tạo listing mới từ KiotViet → push lên N sàn 1 lần

---

## 6. Chat đa kênh (KBH05)

Hợp nhất tin nhắn từ nhiều kênh vào 1 hộp thư:

### 6.1 Nguồn kênh chat

| Kênh | Loại |
|---|---|
| Facebook Messenger | Khách nhắn qua fanpage |
| Zalo OA | Khách nhắn qua OA |
| Shopee Chat | Khách hỏi trước mua |
| TikTok DM | Comment + DM từ TikTok |
| Instagram DM | |

### 6.2 Tính năng hộp thư hợp nhất

| Tính năng | Mô tả |
|---|---|
| Thread per conversation | Mỗi cuộc trò chuyện là 1 thread |
| Gán nhân viên | Assign cho NV phụ trách xử lý |
| Gắn nhãn | Mới / Đang xử lý / Đã xong / Spam |
| Template nhanh | Trả lời nhanh câu hỏi thường gặp |
| Tạo đơn từ chat | Tạo DH/HD trực tiếp từ cửa sổ chat |
| Xem lịch sử mua | Sidebar hiển thị lịch sử KH khi chat |

---

## 7. Entity Catalog (KBH01–KBH06)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| KBH01 | ChannelAccount | Gian hàng kết nối | Có |
| KBH02 | ChannelMapping (E36) | Liên kết SP ↔ Listing sàn | Có |
| KBH03 | ChannelOrder | Đơn từ sàn (trước khi map vào DH) | Có |
| KBH04 | InventorySyncLog | Log đồng bộ tồn kho | Cần bổ sung |
| KBH05 | ChatThread | Thread hội thoại đa kênh | Có một phần |
| KBH06 | AutoMarketingSchedule (E37) | Lịch đẩy hàng tự động | Có một phần |

---

## 8. Pain points hiện tại

| # | Pain | Mức độ |
|---|---|---|
| P1 | Tồn kho sync chậm (>5 phút) → oversell trong flash sale | Cao |
| P2 | Khi mất kết nối API sàn → không có alert — khám phá ra khi đã oversell | Cao |
| P3 | Không thể map 1 variant KiotViet ↔ nhiều SKU trên Shopee (khác màu đóng gói) | Trung bình |
| P4 | Chat đa kênh không gom được theo identity KH — cùng KH nhắn FB và Zalo → 2 thread riêng | Cao |
| P5 | Không có báo cáo hiệu quả per kênh (doanh thu, lợi nhuận sau phí sàn) | Trung bình |
| P6 | Tạo listing mới trên sàn phải làm thủ công từng sàn | Trung bình |
| P7 | Không có buffer tồn cấu hình theo sàn — nhiều shop oversell vào cuối ngày | Cao |
| P8 | Đối soát tiền từ sàn phải làm Excel thủ công | Cao |

---

## 9. Cơ hội cải tiến (Top 5)

### I1. Real-time Inventory Sync (<5s)
- Chuyển từ polling sang webhook-first
- Khi tồn = 0 → đẩy ngay, không chờ batch
- Dashboard realtime trạng thái sync từng gian hàng

### I2. Smart Buffer Management
- AI tự đề xuất buffer tối ưu dựa trên velocity bán per sàn
- Buffer tự động tăng trước flash sale / sự kiện lớn

### I3. Unified Customer Identity (Inbox 2.0)
- Gom chat theo số điện thoại / email đã xác nhận
- 1 KH = 1 thread dù nhắn từ kênh nào
- Lịch sử mua + điểm tích + nợ hiển thị ngay trong hộp thư

### I4. Cross-channel Reconciliation
- Tự động đối soát tiền sàn với KiotViet
- Dashboard: Tiền đang trên đường / Tiền đã về tài khoản / Chênh lệch
- Alert khi phí sàn bất thường

### I5. Catalog Publisher 1-click
- Tạo/cập nhật listing trên N sàn từ 1 giao diện
- AI tự viết mô tả, title SEO per sàn từ thông tin SP nội bộ
