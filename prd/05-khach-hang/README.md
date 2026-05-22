# Module 05 — Khách hàng

Phân tích toàn bộ module Khách hàng của KiotViet: identity, AR (công nợ), loyalty, segmentation rule engine và cơ hội cải tiến.

**Test data:** 1,624 KH | Nợ hiện tại 7,181,257,836đ | Tổng bán 8,517,104,309đ

---

## Cấu trúc tài liệu

| File | Nội dung | Đọc khi |
|---|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | Entity Customer, 5 tab detail, form tạo, list/filter, state machine, tích hợp module khác, pain points & cơ hội | Đọc đầu tiên — bức tranh toàn cảnh |
| [02-nhom-va-segmentation.md](./02-nhom-va-segmentation.md) | CustomerGroup + Rule Engine (12 điều kiện, 3 mode, auto-execute), so sánh với RFM chuẩn, pain points segmentation | Hiểu cơ chế phân nhóm tự động và cơ hội nâng cấp |
| [03-sua-thong-tin-khach-hang.md](./03-sua-thong-tin-khach-hang.md) | Screen spec: sửa thông tin KH — form 4 section, dedup phone warning, địa danh 01/07/2025, audit trail, BRs | Thiết kế màn hình sửa KH |
| [04-gui-tin-nhan.md](./04-gui-tin-nhan.md) | Screen spec: gửi tin nhắn — ZNS/SMS/Email/Zalo OA, luồng đơn lẻ vs hàng loạt, template, opt-out, BRs | Thiết kế tính năng messaging KH |

---

## Entity Catalog (C01–C11)

| # | Entity | Tên VN | Trạng thái | Tham chiếu |
|---|---|---|---|---|
| C01 | Customer | Khách hàng | Có | `01-tong-quan.md` §1 |
| C02 | ShippingAddress | Địa chỉ nhận hàng (1..N per KH) | Có | `01-tong-quan.md` §1.4 |
| C03 | InvoiceIssuingInfo | Thông tin xuất HĐĐT | Có | `01-tong-quan.md` §1.3 |
| C04 | CustomerGroup | Nhóm khách hàng | Có | `02-nhom-va-segmentation.md` §2 |
| C05 | CustomerGroupRule | Rule engine nhóm KH (12 điều kiện) | Có | `02-nhom-va-segmentation.md` §3 |
| C06 | LoyaltyTransaction | Lịch sử tích/đổi/điều chỉnh điểm | Có | `01-tong-quan.md` §4.5 |
| C07 | CustomerTier | Hạng KH (Bronze/Silver/Gold) | Có (chưa có rule engine) | `01-tong-quan.md` §1.1 |
| C08 | CustomerReceivable | Công nợ KH (view derived từ HĐ + Phiếu thu) | Derived | `01-tong-quan.md` §4.4 |
| C09 | CustomerLifecycle | Stage vòng đời KH (Active/Inactive/Lost) | **Chưa có — đề xuất** | `01-tong-quan.md` §5 |
| C10 | CreditLimit | Hạn mức công nợ per KH | **Chưa có — đề xuất** | `01-tong-quan.md` §6.3 |
| C11 | CustomerOwner | Người phụ trách (1:1 hiện tại → cần 1:N) | Có (1:1) | `01-tong-quan.md` §6.5 |

---

## Customer là hub trung tâm

```
                  ┌──────── Customer (C01) ────────┐
                  │                                  │
  ┌───────────────┼───────────────┬──────────────────┼────────────────────┐
  ▼               ▼               ▼                  ▼                    ▼
[Bán hàng HD] [Đặt hàng DH] [Trả hàng TH]   [Tích điểm C06]   [Nhóm KH C04]
  │               │               │                  │                    │
  └─ totalSale    └─ balance (pre) └─ totalReturn     └─ points            └─ discount auto-apply
  └─ balance↑                     └─ balance↓

[Sổ quỹ phiếu thu] ──────────────────────────────► Customer.balance ↓
[Bảng giá] ──────────────────────────────────────► áp giá riêng per nhóm
[Marketing CSKH] ────────────────────────────────► ZNS/SMS/Email/Zalo OA
```
