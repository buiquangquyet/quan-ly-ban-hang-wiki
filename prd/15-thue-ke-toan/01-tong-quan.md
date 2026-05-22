# Tổng quan — Thuế & Kế toán

**Phạm vi:** Module quản lý thuế VAT, Hóa đơn điện tử (HĐĐT) đầu ra và đầu vào, báo cáo thuế và sổ sách kế toán. Đây là module compliance bắt buộc theo quy định pháp luật Việt Nam.
**Căn cứ pháp lý:** Nghị định 123/2020/NĐ-CP, Thông tư 78/2021/TT-BTC, Nghị định 70/2025/NĐ-CP (HĐĐT bắt buộc từ 2026)
**Liên quan:** Module 07 (Hóa đơn bán — phát HĐĐT đầu ra), Module 03 (Nhập hàng — nhận HĐĐT đầu vào), Module 04 (Sổ quỹ — phát sinh thuế)

---

## 1. Vị trí trong hệ sinh thái compliance

```
[Bán hàng HD] ──────────────────────► [HĐĐT Đầu ra] ──► [CQT xác nhận]
      │                                      │
      └── VAT đầu ra (thuế phải nộp)          └── Báo cáo thuế tháng/quý

[Nhập hàng PN] ─────────────────────► [HĐĐT Đầu vào] ──► [Khấu trừ thuế]
      │                                      │
      └── VAT đầu vào (được khấu trừ)         └── Tổng hợp thuế đầu vào

                    [VAT phải nộp] = [VAT đầu ra] − [VAT đầu vào]
                    → Nộp cho Cơ quan Thuế theo tháng / quý
```

---

## 2. Hóa đơn điện tử (HĐĐT)

### 2.1 HĐĐT Đầu ra (Output E-Invoice)

Phát sinh khi bán hàng cho khách. Đã được mô tả chi tiết trong Module 07:
- Phát hành khi tạo Hóa đơn (HD)
- Tích hợp với nhà cung cấp HĐĐT (VNPT, Viettel, FPT, MobiFone, MISA)
- State machine: Khởi tạo → Gửi CQT → CQT chấp nhận → Hợp lệ
- Trường hợp đặc biệt: Hóa đơn điều chỉnh, Hóa đơn thay thế, Hóa đơn hủy

**Thông tin bắt buộc trên HĐĐT đầu ra:**
| Trường | Quy định |
|---|---|
| Ký hiệu mẫu số hóa đơn | VD: 1C25TAA |
| Ký hiệu hóa đơn | VD: AA/25E |
| Số thứ tự | Tự động tăng dần |
| Ngày lập | Ngày tạo hóa đơn |
| Tên người bán, MST người bán | Thông tin của shop |
| Tên người mua, MST/CCCD người mua | Lấy từ entity Customer |
| Tên hàng hóa, dịch vụ | Tên sản phẩm |
| Đơn vị tính, số lượng, đơn giá | |
| Thành tiền chưa thuế | |
| Thuế suất VAT, tiền thuế | Theo từng dòng SP |
| Tổng tiền thanh toán | |
| Chữ ký số người bán | |

### 2.2 HĐĐT Đầu vào (Input E-Invoice)

Nhận từ nhà cung cấp khi nhập hàng:
- Nhận qua email hoặc cổng CQT (tra cứu trực tiếp)
- Validate: số hóa đơn có tồn tại trên cổng CQT không?
- Lưu trữ và liên kết với Phiếu nhập hàng (PN)
- Dùng để khấu trừ VAT đầu vào

**Trạng thái HĐĐT đầu vào:**
| Trạng thái | Mô tả |
|---|---|
| Chưa có | PN chưa đính kèm HĐĐT của NCC |
| Đã nhập | Nhập thủ công từ email/PDF |
| Đã xác thực | Validated qua API CQT — hóa đơn hợp lệ |
| Không hợp lệ | CQT báo sai / hóa đơn giả |

---

## 3. Thuế VAT

### 3.1 Các mức thuế suất

Việt Nam hiện có 4 mức thuế suất VAT:

| Mức | Áp dụng cho |
|---|---|
| 0% | Hàng xuất khẩu, dịch vụ xuất khẩu |
| 5% | Thực phẩm thiết yếu, phân bón, thuốc, giáo dục, nước sạch |
| 8% | Nhóm hàng được giảm thuế (theo NQ 43/2022, gia hạn nhiều lần) |
| 10% | Hàng hóa dịch vụ thông thường |

Mỗi SP trong KiotViet được gán thuế suất VAT bán + VAT nhập riêng biệt.

### 3.2 Báo cáo thuế VAT

**Bảng kê hóa đơn bán (Mẫu 01-1/KK-GTGT):**
- Liệt kê tất cả HĐĐT đầu ra trong kỳ
- Cột: Ngày lập / Ký hiệu / Số / Tên người mua / MST / Doanh thu chưa thuế / Tiền thuế

**Bảng kê hóa đơn mua (Mẫu 01-2/KK-GTGT):**
- Liệt kê tất cả HĐĐT đầu vào hợp lệ đã nhận trong kỳ
- Cột: Ngày lập / Ký hiệu / Số / Tên người bán / MST / Giá trị chưa thuế / Tiền thuế

**Tờ khai thuế GTGT (Mẫu 01/GTGT):**
- Tổng VAT đầu ra
- Tổng VAT đầu vào được khấu trừ
- VAT phải nộp = Đầu ra − Đầu vào

**Kỳ kê khai:**
- Theo tháng: Doanh thu năm trước > 50 tỷ
- Theo quý: Doanh thu năm trước ≤ 50 tỷ (thường gặp ở SMB)

### 3.3 Hộ kinh doanh (HKD) — đặc thù

Từ 2026 (Nghị định 70/2025), hộ kinh doanh bắt buộc dùng HĐĐT:
- **Phương pháp tỷ lệ:** Nộp thuế khoán theo % doanh thu (thay vì kê khai VAT đầu ra - đầu vào)
- Mức tỷ lệ: Ngành bán lẻ hàng hóa = 1% (VAT) + 0.5% (TNCN) trên doanh thu

---

## 4. Sổ sách kế toán cơ bản

### 4.1 Phạm vi kế toán trong KiotViet

KiotViet không phải phần mềm kế toán đầy đủ (như MISA, Fast) nhưng cần đủ để SMB vận hành:

| Tính năng | Có trong KiotViet | Ghi chú |
|---|---|---|
| Sổ quỹ (Cash book) | Có | Module 04 |
| Công nợ KH (AR) | Có | Derived từ HD + Phiếu thu |
| Công nợ NCC (AP) | Có | Derived từ PN + Phiếu chi |
| Doanh thu | Có | Báo cáo Module 13 |
| Giá vốn COGS (WAC) | Có | Module 02 |
| Lợi nhuận gộp | Có (báo cáo) | |
| Sổ cái kế toán | Không (chưa) | Cần cho kế toán thuế đầy đủ |
| Bút toán Nợ/Có | Không (chưa) | Future scope |
| Kết chuyển cuối kỳ | Không (chưa) | Future scope |

### 4.2 Xuất dữ liệu sang phần mềm kế toán

Do KiotViet chưa có kế toán đầy đủ, cần **export** để NV kế toán nhập vào MISA/Fast:
- Export danh sách HĐ bán theo kỳ (Excel)
- Export danh sách phiếu nhập / chi phí
- Export sổ quỹ
- Format chuẩn mapping sang bút toán kế toán

---

## 5. Quản lý nhà cung cấp HĐĐT

Mỗi shop khi dùng HĐĐT phải đăng ký với 1 hoặc nhiều nhà cung cấp dịch vụ HĐĐT (NCCHDDT):

| NCCHDDT | Ghi chú |
|---|---|
| VNPT | Phổ biến nhất ở VN |
| Viettel | |
| FPT | |
| MobiFone | |
| MISA | Tích hợp sâu với phần mềm kế toán |

**Thông tin kết nối:**
- Mã thuế người bán (MST)
- Token / API key của nhà cung cấp
- Mẫu hóa đơn đăng ký (ký hiệu mẫu)
- Phạm vi số (từ số nào đến số nào)

---

## 6. Entity Catalog (KT01–KT07)

| # | Entity | Tên VN | Tình trạng |
|---|---|---|---|
| KT01 | EInvoiceProvider | Nhà cung cấp HĐĐT | Có |
| KT02 | EInvoiceTemplate | Mẫu HĐĐT đã đăng ký | Có |
| KT03 | OutputEInvoice | HĐĐT đầu ra (liên kết HD) | Có — Module 07 |
| KT04 | InputEInvoice (E18) | HĐĐT đầu vào (liên kết PN) | Có — Module 02 |
| KT05 | TaxDeclaration | Tờ khai thuế GTGT | Cần bổ sung |
| KT06 | TaxPeriod | Kỳ kê khai thuế | Cần bổ sung |
| KT07 | AccountingExport | Package xuất kế toán | Cần bổ sung |

---

## 7. Nghiệp vụ chính

### UC-KT01: Phát hành HĐĐT đầu ra
- **Trigger:** Tạo Hóa đơn (HD) → auto hoặc thủ công phát HĐĐT
- **Flow:** Tạo HD → Hệ thống gọi API NCCHDDT → NCCHDDT gửi CQT → CQT trả mã tra cứu → Lưu trạng thái
- **Error handling:** CQT từ chối → hiển thị lý do → cho phép điều chỉnh và phát lại
- **Tham khảo chi tiết:** `07-hoa-don/hoa-don.md`

### UC-KT02: Nhận và xác thực HĐĐT đầu vào
- **Actor:** NV kế toán / NV thu mua
- **Flow:** NCC gửi email PDF → NV nhập số HĐ + ký hiệu → Hệ thống auto-validate qua CQT API → Đính kèm vào PN
- **Rule:** Chỉ HĐĐT đã xác thực mới được tính vào khấu trừ thuế
- **Edge case:** NCC chưa có HĐĐT điện tử → chấp nhận hóa đơn giấy tạm thời

### UC-KT03: Lập báo cáo thuế GTGT theo kỳ
- **Actor:** Kế toán
- **Flow:** Chọn kỳ (tháng/quý) → Hệ thống tổng hợp Đầu ra + Đầu vào → Sinh bảng kê 01-1, 01-2 → Sinh tờ khai 01/GTGT → Export Excel/XML → Nộp lên cổng thuế
- **Rule:** Chỉ tính HĐĐT đầu vào đã xác thực; chỉ tính HĐĐT đầu ra trạng thái "Hợp lệ"

---

## 8. Pain points hiện tại

| # | Pain | Mức độ |
|---|---|---|
| P1 | HĐĐT phát hành xong nhưng CQT từ chối — nhân viên không biết vì không có alert | Cao |
| P2 | Không có màn hình tổng hợp HĐĐT đầu vào / khấu trừ thuế | Cao |
| P3 | Báo cáo thuế phải làm tay trên Excel — sai sót khi có 1000+ HĐ/tháng | Cao |
| P4 | Kết nối HĐĐT lỗi không tự reconnect — phải vào cài đặt reconnect thủ công | Trung bình |
| P5 | HĐ cần điều chỉnh/thay thế — quy trình phức tạp, NV không biết làm | Cao |
| P6 | Không có cảnh báo khi phạm vi số HĐĐT sắp hết | Trung bình |
| P7 | SMB không biết kỳ kê khai của mình là tháng hay quý | Thấp |

---

## 9. Cơ hội cải tiến (Top 5)

### I1. 1-Click Tax Report
- Tự động tổng hợp bảng kê 01-1/01-2 + tờ khai 01/GTGT
- Export XML chuẩn iXML nộp thẳng lên cổng thuế HTKK
- Notification trước deadline nộp thuế (ngày 20 tháng sau / ngày 30 tháng cuối quý)

### I2. HĐĐT Đầu vào Auto-fetch
- NCC gửi HĐĐT qua email → system auto-fetch và validate qua CQT
- Không cần NV nhập tay số hóa đơn
- Auto-link với PN theo số phiếu + MST NCC

### I3. Compliance Dashboard
- Trạng thái tất cả HĐĐT trong kỳ (bao nhiêu đã hợp lệ, bao nhiêu lỗi)
- Alert khi sắp đến deadline nộp thuế
- Checklist "Tháng này cần làm gì" cho kế toán SMB

### I4. Smart Error Recovery
- Khi HĐĐT bị từ chối tự động đề xuất nguyên nhân + hướng xử lý
- Batch re-issue cho nhiều HĐ bị lỗi cùng loại

### I5. Tích hợp export kế toán sang MISA
- Export bút toán theo chuẩn MISA SME / MISA AMIS
- Mapping tài khoản kế toán (TK 511, 632, 131, 331...) tự động
