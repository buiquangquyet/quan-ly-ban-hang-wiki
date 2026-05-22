# Module 15 — Thuế & Kế toán

Quản lý compliance thuế VAT, Hóa đơn điện tử (HĐĐT) và sổ sách kế toán theo quy định pháp luật Việt Nam.

**Căn cứ pháp lý:** Nghị định 123/2020/NĐ-CP, Thông tư 78/2021/TT-BTC, Nghị định 70/2025/NĐ-CP

---

## Cấu trúc tài liệu

| File | Nội dung | Đọc khi |
|---|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | HĐĐT đầu ra/vào, thuế VAT 4 mức, báo cáo thuế GTGT, kế toán cơ bản, nhà cung cấp HĐĐT | Đọc đầu tiên — bức tranh toàn cảnh |

---

## Entity Catalog (KT01–KT07)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| KT01 | EInvoiceProvider | Nhà cung cấp HĐĐT (VNPT/Viettel/FPT/MISA) | Có |
| KT02 | EInvoiceTemplate | Mẫu HĐĐT đã đăng ký | Có |
| KT03 | OutputEInvoice | HĐĐT đầu ra (liên kết Hóa đơn bán) | Có — xem Module 07 |
| KT04 | InputEInvoice (E18) | HĐĐT đầu vào (liên kết Phiếu nhập) | Có — xem Module 02 |
| KT05 | TaxDeclaration | Tờ khai thuế GTGT | Cần bổ sung |
| KT06 | TaxPeriod | Kỳ kê khai thuế (tháng/quý) | Cần bổ sung |
| KT07 | AccountingExport | Package xuất kế toán sang MISA | Cần bổ sung |
