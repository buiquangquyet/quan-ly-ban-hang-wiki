# Module 02 — Hàng hóa & Kho

Phân tích toàn bộ module Hàng hóa và Tồn kho của KiotViet, bao gồm data model, nghiệp vụ vận hành, điểm yếu hiện tại và cơ hội cải tiến.

**Test data:** 5,510 hàng hóa, 2 chi nhánh (CN 1, CN Trung tâm)

---

## Cấu trúc tài liệu

| File | Nội dung | Đọc khi |
|---|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | Sơ đồ tổ chức, danh sách 37 entities, dòng dữ liệu, điểm mạnh/yếu tổng thể | Đọc đầu tiên — nắm bức tranh toàn cảnh |
| [02-the-kho.md](./02-the-kho.md) | Thẻ kho (Stock Card) — backbone audit trail, mã chứng từ, schema, UX gaps | Hiểu cơ chế ghi nhận mọi biến động tồn |
| [03-nhap-hang.md](./03-nhap-hang.md) | Nghiệp vụ Nhập hàng (PN), giá vốn WAC, chi phí phân bổ, liên kết NCC/HĐĐT | Phân tích đầu vào tồn kho và giá vốn |
| [04-ban-hang.md](./04-ban-hang.md) | Nghiệp vụ Bán hàng — góc tác động tồn kho, reservation, âm tồn, race condition | Phân tích đầu ra tồn kho và tính chính xác kế toán |
| [05-tra-hang.md](./05-tra-hang.md) | Trả hàng bán (TH) + Trả hàng nhập (TPN) — so sánh, edge cases, fraud detection | Phân tích vòng đời ngược của hàng hóa |
| [06-lo-va-serial.md](./06-lo-va-serial.md) | Lô/Hạn sử dụng (Batch) + Serial/IMEI — data model 3 tầng, FEFO, gap analysis | Tracking nâng cao cho F&B, dược, điện máy |

---

## Entity Catalog (E01–E37)

Bảng tham chiếu nhanh toàn bộ entities, đánh số nhất quán qua tất cả tài liệu.

### Product Catalog
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E01 | Product | Hàng hóa | `01-tong-quan.md` §A1 |
| E02 | UnitOfMeasure | Đơn vị tính | `01-tong-quan.md` §A2 |
| E03 | Variant / Attribute | Biến thể / Thuộc tính | `01-tong-quan.md` §A3 |
| E04 | Category | Nhóm hàng | `01-tong-quan.md` §A4 |
| E05 | Brand | Thương hiệu | `01-tong-quan.md` §A5 |
| E06 | PriceBook | Bảng giá | `01-tong-quan.md` §A6 |
| E07 | Supplier | Nhà cung cấp | `01-tong-quan.md` §A7 |
| E08 | ProductImage | Ảnh sản phẩm | `06-lo-va-serial.md` §1 |

### Inventory Core
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E09 | Stock (per Branch) | Tồn kho theo chi nhánh | `01-tong-quan.md` §B1 |
| E10 | StockCard / StockCardLine | Thẻ kho | `02-the-kho.md` §2 |
| E11 | StockNorm | Định mức tồn (Min/Max) | `01-tong-quan.md` §B3 |
| E12 | StockoutForecast | Dự kiến hết hàng | `01-tong-quan.md` §B4 |
| E13 | DocumentCode | Mã chứng từ | `02-the-kho.md` §3 |
| E14 | TransactionType | Loại giao dịch (enum) | `02-the-kho.md` §4 |

### Purchase Operations
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E15 | PurchaseImport | Phiếu nhập hàng (PN) | `03-nhap-hang.md` §2 |
| E16 | PurchaseOrder | Đặt hàng nhập (DHN) | `03-nhap-hang.md` §1 |
| E17 | ImportCostType | Chi phí nhập hàng | `03-nhap-hang.md` §2.2 |
| E18 | InputEInvoice | Hóa đơn đầu vào | `03-nhap-hang.md` §2.3 |

### Sales Operations
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E19 | Invoice | Hóa đơn bán hàng (HD) | `04-ban-hang.md` §2 |
| E20 | CustomerOrder | Đặt hàng khách (DH) | `04-ban-hang.md` §2.2 |
| E21 | StockAggregate | Tồn tổng hợp (onHand/reserved/available) | `04-ban-hang.md` §2.3 |
| E22 | Reservation | Đặt chỗ (derived từ DH/DHN) | `04-ban-hang.md` §3 |

### Returns
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E23 | SalesReturn | Phiếu trả hàng bán (TH) | `05-tra-hang.md` §2 |
| E24 | PurchaseReturn | Phiếu trả hàng nhập (TPN) | `05-tra-hang.md` §3 |
| E25 | ReturnReason | Lý do trả (taxonomy — chưa có) | `05-tra-hang.md` §5 |
| E26 | ReturnPolicy | Chính sách trả (chưa có) | `05-tra-hang.md` §5 |

### Warehouse Operations
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E27 | StockTransfer | Phiếu chuyển hàng (TRF) | `01-tong-quan.md` §C1 |
| E28 | StockTake | Phiếu kiểm kho (KK) | `01-tong-quan.md` §C2 |
| E29 | ManufacturingVoucher | Phiếu sản xuất (SX) | `01-tong-quan.md` §C3 |
| E30 | InternalUseVoucher | Xuất dùng nội bộ (XN) | `01-tong-quan.md` §C4 |
| E31 | WriteOffVoucher | Xuất hủy (XH) | `01-tong-quan.md` §C5 |

### Batch & Serial Tracking
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E32 | Batch | Lô / Hạn sử dụng | `06-lo-va-serial.md` §3 |
| E33 | BatchStock | Junction Batch × Branch | `06-lo-va-serial.md` §3.1 |
| E34 | StockUnit | Đơn vị vật lý có Serial/IMEI | `06-lo-va-serial.md` §4.2 |
| E35 | WarrantyRecord | Bảo hành theo Serial (chưa có) | `06-lo-va-serial.md` §4.4 |

### Integration
| # | Entity | Tên VN | Tham chiếu |
|---|---|---|---|
| E36 | ChannelMapping | Liên kết kênh bán | `01-tong-quan.md` §D1 |
| E37 | AutoMarketing | Marketing tự động đẩy hàng | `01-tong-quan.md` §D2 |

---

## Dòng chảy nghiệp vụ tác động tồn kho

```
[NCC] ──PN──► [Tồn CN_X] ──HD──► [Khách]
                  │  ▲
                  │  └──TH────── [Khách trả]
                  │
                  ├──TRF──────► [Tồn CN_Y]
                  ├──KK──────── (±điều chỉnh)
                  ├──SX──────► [Thành phẩm]
                  ├──XN──────► [Nhân viên]
                  └──XH──────► [Hủy bỏ]
              [Shop]──TPN──► [NCC]
```

Tất cả nghiệp vụ đều ghi vào **Thẻ kho (E10)** — single source of truth cho audit.
