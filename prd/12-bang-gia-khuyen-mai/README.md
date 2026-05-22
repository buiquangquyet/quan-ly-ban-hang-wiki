# Module 12 — Bảng giá & Khuyến mãi

Tầng định giá động: quản lý bảng giá multi-tier, chương trình khuyến mãi, coupon, voucher và tích điểm.

---

## Cấu trúc tài liệu

| File | Nội dung | Đọc khi |
|---|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | PriceBook, Promotion Engine (7 loại KM), Coupon/Voucher, Tích điểm/đổi điểm, stacking rules | Đọc đầu tiên — bức tranh toàn cảnh |

---

## Entity Catalog (BG01–BG06)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| BG01 | PriceBook | Bảng giá | Có |
| BG02 | PriceBookLine | Dòng giá trong bảng | Có |
| BG03 | Promotion | Chương trình khuyến mãi | Có một phần |
| BG04 | PromotionCondition | Điều kiện áp dụng KM | Cần bổ sung |
| BG05 | Coupon | Mã giảm giá / voucher | Có một phần |
| BG06 | LoyaltyRule | Quy tắc tích/đổi điểm | Cần bổ sung |
