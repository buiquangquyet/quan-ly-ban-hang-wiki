# Nghiệp vụ Trả hàng — Deep Dive

**Bối cảnh:** "Trả hàng" trong KiotViet gồm **2 nghiệp vụ độc lập** theo chiều dòng tiền và đối tác:

1. **Trả hàng bán (TH)** — Khách trả lại hàng đã mua, mã prefix `TH`, tăng tồn
2. **Trả hàng nhập (TPN)** — Shop trả lại hàng cho NCC, mã prefix `TPN`, giảm tồn

**Entities:** E23 (SalesReturn), E24 (PurchaseReturn), E25 (ReturnReason), E26 (ReturnPolicy)
**Liên quan:** [README.md](./README.md) | [03-nhap-hang.md](./03-nhap-hang.md) | [04-ban-hang.md](./04-ban-hang.md)

---

## 1. Phân biệt 2 nghiệp vụ trả hàng

```
        KHÁCH HÀNG                          SHOP                        NHÀ CUNG CẤP
            │                                │                                │
            │  Bán hàng (HD) ──giảm tồn──►   │                                │
            │  Trả hàng bán (TH) ◄─tăng tồn  │                                │
            │                                │                                │
            │                                │  ◄──tăng tồn── Nhập hàng (PN) │
            │                                │  ──giảm tồn──► Trả hàng nhập (TPN) │
```

| Tiêu chí | Trả hàng bán (TH — E23) | Trả hàng nhập (TPN — E24) |
|---|---|---|
| Đối tác | Khách hàng | Nhà cung cấp |
| Chiều hàng | Vào kho (tăng tồn) | Ra khỏi kho (giảm tồn) |
| Chiều tiền | Trả tiền KH / Ghi giảm công nợ KH | Nhận tiền từ NCC / Giảm công nợ phải trả NCC |
| Phiếu nguồn | Hóa đơn HD (optional — có "Trả nhanh") | Phiếu Nhập PN (bắt buộc) |
| Auto-trigger | Khi đơn giao chuyển hoàn | Không có |
| Tác động giá vốn | Không đổi (chỉ +SL) | Không đổi (chỉ −SL) |

---

## 2. Trả hàng bán (Customer Return — E23)

### 2.1 Cấu trúc dữ liệu

```
SalesReturn  (TH######)
├── Header
│   ├── code              string   TH######
│   ├── branchId          FK       CN tiếp nhận trả
│   ├── customerId        FK?      Có thể "Khách lẻ"
│   ├── returnedAt        datetime
│   ├── receivedBy        FK User  Người nhận trả
│   ├── sourceInvoiceId   FK?      HĐ gốc — NULL nếu "Trả nhanh"
│   ├── status            enum     Đã_trả | Đã_hủy
│   ├── totalReturnValue  decimal  ∑ lines
│   ├── refundAmount      decimal  Tiền trả lại khách
│   ├── exchangeAmount    decimal  Tiền hàng đổi (nếu có)
│   ├── netAmount         decimal  refund − exchange (âm = thu thêm từ KH)
│   ├── returnFee         decimal  Phí trả hàng (nếu có)
│   ├── isAutoFromReturn  bool     true nếu đơn giao bị hoàn
│   └── shippingReturnId  FK?      Link vận đơn hoàn
│
├── ReturnLines  (hàng trả)
│   ├── productId / variantId / unitId
│   ├── quantity
│   ├── returnPrice       Có thể khác giá bán gốc
│   ├── originalLineId    FK? → InvoiceLine gốc (nếu có)
│   └── note
│
└── ExchangeLines  (hàng đổi — optional)
    ├── productId / variantId / unitId
    ├── quantity, price, discount
    └── (tương tự line HĐ bán)
```

### 2.2 State machine

```
  [Trả theo HĐ]                    [Trả nhanh]
        │                                │
        ▼                                ▼
   [Tìm HĐ gốc]            [Tạo phiếu trả độc lập]
        │                                │
        └────────────┬───────────────────┘
                     ▼
            [Nhập SL trả, giá trả]
                     │
        ┌────────────┴──────────┐
        ▼                       ▼
  (chỉ trả)              (trả + đổi hàng)
        │                       │
        │                  [Thêm Exchange Lines]
        │                  [Hệ thống đối trừ tự động]
        └────────────┬──────────┘
                     ▼
            [F9 — Trả hàng / Thanh toán]
                     │
                     ▼
                 Đã trả ◄────┐
                     │       │
                     │ Hủy   │ (auto từ giao hàng hoàn → KHÔNG hủy được)
                     ▼       │
                 Đã hủy ─────┘
```

### 2.3 3 mode chính trên UI

| Mode | Trigger | Use case |
|---|---|---|
| **Trả theo hóa đơn** | Tìm HĐ gốc → load lines → chọn SL trả | KH còn hóa đơn |
| **Trả nhanh** | Bỏ qua HĐ gốc, tạo phiếu trả độc lập | KH mất HĐ, hàng đổi qua nhiều người |
| **Đổi trả hàng** | Bất kỳ mode + thêm hàng "khách mua" | Đổi size, đổi mẫu, upgrade SP |

### 2.4 Edge cases quan trọng

| # | Tình huống | Behavior KiotViet |
|---|---|---|
| 1 | KH trả nhiều lần trên cùng 1 HĐ | Hỗ trợ — mỗi lần 1 TH riêng, SL còn = `qty_HD − ∑qty_TH` |
| 2 | Trả với giá khác giá bán gốc | Cho phép, ghi nhận `returnPrice`; không sửa HĐ gốc |
| 3 | Trả hàng có Serial/IMEI | Phải nhập đúng serial đã bán — không cho trả serial khác |
| 4 | Trả hàng có Lô/HSD | Trả vào đúng lô gốc; nếu lô hết hạn → không cho bán lại |
| 5 | Đơn giao "Đã chuyển hoàn" | **Auto tạo TH**, không cho hủy phiếu này |
| 6 | HĐ gốc đã in HĐĐT | Phải lập HĐ điều chỉnh giảm trên eInvoice — không tự động |
| 7 | Khách lẻ trả → refund cash | Tạo phiếu chi từ sổ quỹ |
| 8 | Đổi sang SP đắt hơn → thu thêm | Popup nhập phương thức TT bổ sung |
| 9 | Đổi sang SP rẻ hơn → trả thêm tiền | Tạo phiếu chi tự động |

### 2.5 Tác động hệ thống

```
TH######  (Đã trả)
    │
    ├──► Thẻ kho (E10): +SL
    ├──► Stock per Branch (E09): tăng
    ├──► Customer.totalReturn += value (loyalty cập nhật)
    ├──► Loyalty Point: trừ điểm từ HĐ gốc (nếu có)
    ├──► Sổ quỹ: phiếu chi (refund) hoặc thu (đổi đắt)
    ├──► HĐ gốc: cập nhật trạng thái "Hoàn thành" hoặc "Một phần"
    └── (NẾU có exchange lines):
         ├──► Hóa đơn bán hàng mới sinh ra song song
         └──► Thẻ kho ghi −SL cho hàng đổi
```

---

## 3. Trả hàng nhập (Supplier Return — E24)

### 3.1 Cấu trúc dữ liệu

```
PurchaseReturn  (TPN######)
├── Header
│   ├── code              string   TPN######
│   ├── branchId          FK       CN trả hàng đi
│   ├── supplierId        FK       Bắt buộc
│   ├── returnedAt        datetime
│   ├── returnedBy        FK User
│   ├── sourceImportId    FK → E15 PN gốc — BẮT BUỘC
│   ├── status            enum     Đã_trả | Đã_hủy
│   ├── totalReturnValue  decimal
│   ├── refundFromSupplier decimal Tiền nhận lại từ NCC
│   ├── reduceDebt        decimal  Ghi giảm công nợ phải trả NCC
│   ├── reason            enum     Hàng_lỗi | Sai_quy_cách | Quá_hạn | Hỏng | Khác
│   └── note              string
│
└── Lines  (1..N)
    ├── productId / variantId / unitId
    ├── quantity
    ├── returnPrice     Mặc định = giá nhập gốc, sửa được
    ├── originalLineId  FK → PN line (bắt buộc)
    └── note
```

### 3.2 State machine

```
[Phiếu Nhập gốc PN######]
         │
         │ (chỉ PN ở trạng thái "Đã nhập hàng")
         ▼
  [Trả hàng nhập] ──► [Chọn lines + SL]
         │
         ▼
  [Cấu hình refund: Nhận tiền / Ghi giảm nợ]
         │
         ▼
     Đã trả ◄────┐
         │       │
         │ Hủy   │
         ▼       │
      Đã hủy ────┘
```

### 3.3 Quy tắc nghiệp vụ

| Quy tắc | Behavior |
|---|---|
| Phải từ PN gốc | Không có "Trả nhanh" — luôn cần tham chiếu PN |
| SL trả ≤ SL nhập trừ đã trả trước | Hệ thống chặn over-return |
| Giá trả mặc định = giá nhập gốc | Cho phép sửa nếu thỏa thuận khác |
| Tồn hiện tại < SL trả | Cảnh báo nhưng vẫn cho (cho phép âm tồn) |
| Serial/IMEI | Phải chọn serial cụ thể từ PN gốc |
| Lô/HSD | Phải chọn đúng lô đã nhập |

### 3.4 Tác động hệ thống

```
TPN######  (Đã trả)
    │
    ├──► Thẻ kho (E10): −SL
    ├──► Stock per Branch (E09): giảm
    ├──► Supplier.balance:
    │      ├─ reduceDebt > 0: giảm nợ phải trả NCC
    │      └─ refundFromSupplier > 0: tạo phiếu thu từ NCC
    ├──► PN gốc: cập nhật "Đã trả N/M items"
    └──► Giá vốn WAC: KHÔNG recompute (chỉ −SL)
```

**Lưu ý:** TPN không recompute WAC — về mặt kế toán nghiêm, trả NCC cũng nên revert phần đóng góp vào WAC (xem §5.2 #12).

---

## 4. So sánh 2 nghiệp vụ

| Khía cạnh | Trả hàng bán (TH) | Trả hàng nhập (TPN) |
|---|---|---|
| Prefix mã | TH | TPN |
| Bắt buộc tham chiếu phiếu gốc | Không (có "Trả nhanh") | **Bắt buộc** PN gốc |
| Đối tác | Customer (có thể "Khách lẻ") | Supplier (bắt buộc) |
| Auto trigger | Khi giao hàng chuyển hoàn | Không |
| Hủy được | Trừ phiếu auto từ chuyển hoàn | Có |
| Có "Đổi hàng" inline | Có (mode đổi trả) | Không |
| Tác động loyalty | Trừ điểm | Không |
| Tác động giá vốn | Không đổi | Không đổi (debatable) |
| Tác động HĐĐT | Cần lập HĐ điều chỉnh giảm | Cần điều chỉnh hóa đơn đầu vào |
| App mobile | Đầy đủ 3 mode | Có (từ phiếu nhập) |
| App POS Android | 2 mode | Chỉ tạo từ Web |

---

## 5. Điểm đau & cơ hội cải tiến

### 5.1 Trả hàng bán

| # | Pain | Đề xuất |
|---|---|---|
| 1 | "Trả nhanh" không có anti-fraud — KH có thể trả hàng không mua ở shop để lấy tiền | Yêu cầu mã HĐ tối thiểu hoặc xác thực SĐT KH + lịch sử mua |
| 2 | Không có chính sách trả hàng time window (VD: chỉ trả trong 7 ngày) | Setting per category: max return window, hết hạn → block |
| 3 | Không có lý do trả chuẩn hóa (chỉ free-text note) — không phân tích được | Dropdown: Lỗi sản phẩm / Sai size / Đổi ý / Không vừa / Khác → analytics |
| 4 | Trả hàng có Serial nhưng không kiểm tra tình trạng vật lý — có thể nhận về hàng vỡ rồi bán lại | Workflow QC: phiếu "Hàng đợi kiểm" → kiểm xong mới về tồn bán được |
| 5 | Không có restocking fee tự động (phí 10% khi đổi ý) | Trường `restocking_fee_pct` per category, tự tính |
| 6 | HĐĐT điều chỉnh giảm vẫn làm tay khi tạo TH | Tạo TH → auto trigger HĐĐT điều chỉnh + gửi cơ quan thuế |
| 7 | Phiếu TH auto từ vận chuyển không hủy được — nếu hãng VC báo nhầm "chuyển hoàn" thì kẹt | Cơ chế "đảo TH" có audit, không phải hủy thuần |
| 8 | Không có return abuse detection (KH trả >X lần trong N ngày) | AI flag KH có pattern lạm dụng |
| 9 | Loyalty point trừ ngay khi tạo TH — KH có thể đã consume điểm rồi | Reserve điểm khi consume + clawback workflow |
| 10 | Đổi hàng đắt hơn nhưng KH không thanh toán đủ → vẫn cho hoàn thành | Validation blocking thay vì warning |

### 5.2 Trả hàng nhập

| # | Pain | Đề xuất |
|---|---|---|
| 11 | Phải trả từ đúng PN gốc — nhập 2 lần cùng SP từ 1 NCC phải nhớ phiếu nào | Cho phép trả "theo SP × NCC × Lô" thay vì theo PN |
| 12 | Không recompute WAC khi TPN — về kế toán không chính xác nếu trả ngay sau nhập | Option: revert cost contribution khi TPN |
| 13 | Không có return shipping label tích hợp — phải tự liên hệ NCC | Tích hợp shipping cho TPN |
| 14 | Không có defect rate tự động trên NCC scorecard | Compute `defect_rate = ∑TPN / ∑PN` per NCC, hiển thị supplier card |
| 15 | Không có mass return (trả 50 SP từ nhiều PN cho 1 NCC) — phải tạo nhiều TPN | Bulk TPN từ filter NCC |

### 5.3 Chung cho 2 loại

| # | Pain | Đề xuất |
|---|---|---|
| 16 | Không có photo evidence đính kèm phiếu trả (ảnh hàng lỗi) | Upload ảnh trên line + so sánh khi audit |
| 17 | Không có return reason taxonomy chuẩn hóa chung 2 module | Shared taxonomy: lỗi sản xuất / hỏng vận chuyển / sai mô tả / khác |
| 18 | Báo cáo trả hàng chỉ aggregate — không drill-down theo lý do/SP/NCC/KH | Report module nâng cấp với pivot |

---

## 6. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Return Workflow Engine** — chính sách trả theo SP/Category (time window, restocking fee, QC required) | Pain #2, #4, #5 | Trung | Cao |
| 2 | **Return Fraud Detection AI** — flag KH lạm dụng, NV thông đồng | Pain #8 — marketplace high-volume | Cao | Cao |
| 3 | **NCC Defect Dashboard** — tự động compute defect rate + alert | Pain #14, gắn vendor scorecard | Trung | Cao (B2B) |
| 4 | **Auto HĐĐT điều chỉnh** cho cả TH và TPN | Pain #6 — compliance pain lớn | Trung | Rất cao (legal) |
| 5 | **Return Photo + AI Damage Assessment** — chụp ảnh hàng trả → AI đánh giá có bán lại được không | Pain #4, #16 | Cao | Trung |

---

## 7. Tóm lược

**Trả hàng KiotViet:**
- **2 nghiệp vụ ngược chiều**: TH (KH trả vào kho) và TPN (shop trả ra NCC) — cùng pattern phiếu nhưng đối tác và dòng tiền ngược nhau
- TH **mạnh ở UX**: 3 mode (theo HĐ / nhanh / đổi trả), auto-trigger từ vận chuyển hoàn, có trên Web/Mobile/POS
- TPN **chặt chẽ hơn**: bắt buộc tham chiếu PN gốc, không có "Trả nhanh", không có trên POS
- **Auto-tạo TH từ vận chuyển hoàn** là feature hiếm có — điểm cộng
- **Cả 2 đều yếu** ở: lý do chuẩn hóa, photo evidence, fraud detection, HĐĐT điều chỉnh tự động, return policy engine

Đã giải quyết happy path — còn nhiều gap cho enterprise compliance (HĐĐT) và anti-abuse.
