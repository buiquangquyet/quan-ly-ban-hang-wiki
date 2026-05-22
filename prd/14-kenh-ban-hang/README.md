# Module 14 — Kênh bán hàng online & Đa kênh

Quản lý kết nối và vận hành bán hàng trên TMĐT, mạng xã hội và chat đa kênh.

---

## Cấu trúc tài liệu

| File | Nội dung | Đọc khi |
|---|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | Channel Account, Inventory Sync, Order Pull, Chat đa kênh, Đối soát sàn | Đọc đầu tiên — bức tranh toàn cảnh |

---

## Entity Catalog (KBH01–KBH06)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| KBH01 | ChannelAccount | Gian hàng kết nối | Có |
| KBH02 | ChannelMapping (E36) | Liên kết SP ↔ Listing sàn | Có |
| KBH03 | ChannelOrder | Đơn từ sàn | Có |
| KBH04 | InventorySyncLog | Log đồng bộ tồn kho | Cần bổ sung |
| KBH05 | ChatThread | Thread hội thoại đa kênh | Có một phần |
| KBH06 | AutoMarketingSchedule (E37) | Lịch đẩy hàng tự động | Có một phần |

---

## Kênh hỗ trợ

| Loại | Kênh |
|---|---|
| TMĐT | Shopee, TikTok Shop, Lazada, Tiki |
| Social | Facebook Shop, Instagram, Zalo OA |
| Chat | Facebook Messenger, Zalo Chat, Shopee Chat, TikTok DM |
