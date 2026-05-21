# Module 04 — Sổ quỹ

Module Sổ quỹ là sổ cái vận hành (operational cash ledger) ghi nhận toàn bộ dòng tiền thực tế vào và ra khỏi cửa hàng. Đây là điểm hội tụ của mọi giao dịch có phát sinh tiền từ các module khác trong hệ thống.

Sổ quỹ **không phải** sổ kế toán kép (không có bút toán Nợ/Có), và cũng **không phải** thẻ kho — ba loại sổ cái này phục vụ ba mục đích riêng biệt:

| Sổ cái | Module | Ghi nhận |
|---|---|---|
| Thẻ kho | Hàng hóa | Biến động số lượng hàng tồn kho |
| Sổ quỹ | Sổ quỹ | Biến động tiền mặt và tiền gửi thực tế |
| Sổ kế toán | Thuế & Kế toán | Bút toán Nợ/Có theo Thông tư 200 |

---

## Nội dung tài liệu

| File | Nội dung |
|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | Nghiệp vụ đầy đủ: định nghĩa, cấu trúc quỹ, chứng từ, phân loại giao dịch, vòng đời, quan hệ kế toán, điểm mạnh/hạn chế, cơ hội cải tiến |

---

## Các khái niệm nghiệp vụ chính

| Khái niệm | Mô tả |
|---|---|
| Phiếu thu / Phiếu chi | Chứng từ ghi nhận một dòng tiền vào hoặc ra |
| Loại quỹ | Hình thức tiền: Tiền mặt / Ngân hàng / Ví điện tử / Tổng quỹ |
| Tài khoản quỹ | Một tài khoản ngân hàng hoặc ví cụ thể thuộc loại quỹ trên |
| Loại thu chi | Danh mục phân loại giao dịch, có thể tùy chỉnh theo cửa hàng |
| Flag "Hạch toán KQKD" | Đánh dấu giao dịch có ảnh hưởng đến lãi/lỗ kinh doanh hay không |
| Chuyển quỹ (CNH) | Giao dịch nội bộ di chuyển tiền giữa các quỹ, không ảnh hưởng tổng tồn quỹ |