# ĐÀO SÂU: NGHIỆP VỤ TRẢ HÀNG — KIOTVIET

**Bối cảnh:** "Trả hàng" trong KiotViet gồm **2 nghiệp vụ độc lập** chia theo chiều dòng tiền và đối tác:

1. **Trả hàng bán** (Customer Return) — KH trả lại hàng đã mua, mã prefix `TH`, tăng tồn
2. **Trả hàng nhập** (Supplier Return) — Shop trả lại hàng cho NCC, mã prefix `TPN`, giảm tồn

Hai nghiệp vụ này có data flow ngược chiều, nhưng cùng chia sẻ pattern phiếu (header + lines) và đều ghi vào Thẻ kho.

**Tham chiếu:**
- Trả hàng bán: `kiotviet.vn/huong-dan-su-dung-kiotviet/retail-tra-hang/tra-hang/`
- Trả hàng nhập: `kiotviet.vn/huong-dan-su-dung-kiotviet/retail-giao-dich/tra-hang-nhap/`
- UI hiện tại: Trả hàng bán nằm trong **Đơn hàng → Trả hàng**; Trả hàng nhập nằm trong **Hàng hóa → Nhập hàng → [Trả hàng nhập]** (Giao diện 2026 sẽ chuyển vào menu **Mua hàng**)

---

## 1. PHÂN BIỆT 2 NGHIỆP VỤ TRẢ HÀNG

```
        KHÁCH HÀNG                          SHOP                        NHÀ CUNG CẤP
            │                                │                                │
            │  Bán hàng (HD) ──giảm tồn──►   │                                │
            │  Trả hàng bán (TH) ◄─tăng tồn  │                                │
            │                                │                                │
            │                                │  ◄──tăng tồn── Nhập hàng (PN) │
            │                                │  ──giảm tồn──► Trả hàng nhập (TPN) │
            │                                │                                │
```

| Tiêu chí | Trả hàng bán (TH) | Trả hàng nhập (TPN) |
|---|---|---|
| Đối tác | Khách hàng (Customer) | Nhà cung cấp (Supplier) |
| Chiều hàng | Vào kho (tăng) | Ra khỏi kho (giảm) |
| Chiều tiền | Trả tiền cho KH / Ghi giảm công nợ KH | Nhận tiền lại từ NCC / Giảm công nợ phải trả NCC |
| Phiếu nguồn | Hóa đơn HD (optional — có "Trả nhanh") | Phiếu Nhập PN (bắt buộc — phải từ phiếu nhập gốc) |
| Auto-trigger | Khi đơn giao hàng chuyển hoàn (`Đã chuyển hoàn`) | Không có auto-trigger |
| Tác động giá vốn | Không đổi giá vốn (chỉ +SL) | Không recompute WAC, chỉ −SL |

---

## 2. TRẢ HÀNG BÁN (CUSTOMER RETURN)

### 2.1 Cấu trúc dữ liệu

```
SalesReturn (Phiếu Trả hàng bán)
├── Header
│   ├── code              string   TH######
│   ├── branchId          FK       CN tiếp nhận trả
│   ├── customerId        FK?      Có thể "Khách lẻ"
│   ├── returnedAt        datetime
│   ├── receivedBy        FK User  Người nhận trả
│   ├── sourceInvoiceId   FK?      HĐ gốc — NULL nếu là "Trả nhanh"
│   ├── status            enum     Đã_trả | Đã_hủy
│   ├── totalReturnValue  decimal  ∑ lines
│   ├── refundAmount      decimal  Tiền trả lại khách
│   ├── exchangeAmount    decimal  Tổng tiền hàng đổi (nếu có)
│   ├── netAmount         decimal  refund − exchange (có thể âm = thu thêm)
│   ├── returnFee         decimal  Phí trả hàng (nếu có)
│   ├── isAutoFromReturn  bool     true nếu đơn giao bị hoàn
│   └── shippingReturnId  FK?      Link tới vận đơn hoàn (nếu có)
│
├── ReturnLines  (hàng trả)
│   ├── productId / variantId / unitId
│   ├── quantity        decimal
│   ├── returnPrice     decimal   Có thể khác giá bán gốc
│   ├── originalLineId  FK?       Link tới line HĐ gốc (nếu có)
│   └── note
│
└── ExchangeLines  (hàng đổi — optional)
    ├── productId / variantId / unitId
    ├── quantity, price, discount
    └── (giống line HĐ bán)
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
                           │ Hủy   │ (auto từ giao hàng
                           ▼       │  hoàn → KHÔNG hủy được)
                       Đã hủy ─────┘
```

### 2.3 3 mode chính trên UI

| Mode | Trigger | Use case |
|---|---|---|
| **Trả theo hóa đơn** | Tìm HĐ gốc → load lines → chọn SL trả | KH còn hóa đơn / nhân viên tra được |
| **Trả nhanh** | Bỏ qua HĐ gốc, tạo phiếu trả độc lập | KH mất HĐ, hàng đổi qua nhiều người, gift-card return |
| **Đổi trả hàng** | Bất kỳ mode nào + thêm hàng "khách mua" | Đổi size, đổi mẫu, upgrade SP |

### 2.4 Edge cases quan trọng

| # | Tình huống | Behavior KiotViet |
|---|---|---|
| 1 | KH trả nhiều lần trên cùng 1 HĐ | Hỗ trợ — mỗi lần tạo 1 TH riêng, SL còn lại tính = `qty_HD − ∑qty_TH` |
| 2 | Trả với giá khác giá bán gốc | Cho phép, ghi nhận trên `returnPrice`; không sửa HĐ gốc |
| 3 | Trả hàng có Serial/IMEI | Phải nhập đúng serial đã bán — không cho trả serial khác |
| 4 | Trả hàng có Lô/Hạn sử dụng | Trả vào đúng lô gốc; nếu lô hết hạn → không cho trả về sale lại |
| 5 | Đơn giao hàng `Đã chuyển hoàn` | **Auto tạo TH**, không cho hủy phiếu này (vì sync với hãng vận chuyển) |
| 6 | HĐ gốc đã in HĐĐT | Phải lập **HĐ điều chỉnh giảm** trên KiotViet eInvoice — không tự động |
| 7 | Khách lẻ trả → refund cash | Tạo phiếu chi từ sổ quỹ |
| 8 | Khách có công nợ → refund | Giảm công nợ KH thay vì cash |
| 9 | Đổi sang SP đắt hơn → thu thêm | Pop-up nhập phương thức TT bổ sung |
| 10 | Đổi sang SP rẻ hơn → trả thêm tiền | Tạo phiếu chi tự động |

### 2.5 Tác động hệ thống

```
TH######  (Đã trả)
    │
    ├──► Thẻ kho: +SL (quantity dương)
    ├──► Stock per Branch: tăng
    ├──► Customer.totalReturn += value (loyalty cập nhật)
    ├──► Loyalty Point: trừ điểm tích từ HĐ gốc (nếu có)
    ├──► Sổ quỹ: phiếu chi (nếu refund cash) hoặc phiếu thu (nếu thu thêm)
    ├──► HĐ gốc: cập nhật trạng thái "Hoàn thành" (nếu trả toàn bộ) hoặc "Một phần"
    └── (NẾU có exchange lines):
         ├──► Hóa đơn bán hàng mới sinh ra song song (tự động hoặc gắn vào TH)
         └──► Thẻ kho ghi −SL cho hàng đổi
```

---

## 3. TRẢ HÀNG NHẬP (SUPPLIER RETURN)

### 3.1 Cấu trúc dữ liệu

```
PurchaseReturn (Phiếu Trả hàng nhập)
├── Header
│   ├── code              string   TPN######
│   ├── branchId          FK       CN trả hàng đi
│   ├── supplierId        FK       Bắt buộc (luôn có NCC)
│   ├── returnedAt        datetime
│   ├── returnedBy        FK User
│   ├── sourceImportId    FK       PN gốc — BẮT BUỘC (khác Trả hàng bán)
│   ├── status            enum     Đã_trả | Đã_hủy
│   ├── totalReturnValue  decimal
│   ├── refundFromSupplier decimal Tiền nhận lại từ NCC
│   ├── reduceDebt        decimal  Số ghi giảm công nợ phải trả NCC
│   ├── reason            enum     Hàng_lỗi | Sai_quy_cách | Quá_hạn | Hỏng | Khác
│   └── note              string
│
└── Lines  (1..N)
    ├── productId / variantId / unitId
    ├── quantity        decimal
    ├── returnPrice     decimal  Mặc định = giá nhập gốc, sửa được
    ├── originalLineId  FK       Bắt buộc tham chiếu PN line
    └── note
```

### 3.2 State machine

```
   [Phiếu Nhập gốc PN######]
            │
            │ (chỉ phiếu PN ở trạng thái "Đã nhập hàng")
            ▼
     [Trả hàng nhập] ────► [Chọn lines + SL]
            │
            ▼
   [Cấu hình refund:           
    - Nhận tiền/Ghi giảm nợ]
            │
            ▼
        [Hoàn thành]
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
| Phải từ PN gốc | Không có "Trả nhanh" như TH — luôn cần tham chiếu PN |
| SL trả ≤ SL nhập trừ đi đã trả trước | Hệ thống chặn over-return |
| Giá trả mặc định = giá nhập gốc | Cho phép sửa nếu thỏa thuận khác với NCC |
| Tồn hiện tại < SL trả | Cảnh báo nhưng vẫn cho (cho phép âm tồn) |
| Hàng đã bán đi rồi mới trả NCC | Cảnh báo, vẫn cho — sẽ kéo tồn xuống âm |
| Có Serial/IMEI | Phải chọn serial cụ thể từ PN gốc |
| Có Lô/Hạn sử dụng | Phải chọn đúng lô đã nhập |

### 3.4 Tác động hệ thống

```
TPN######  (Đã trả)
    │
    ├──► Thẻ kho: −SL (quantity âm)
    ├──► Stock per Branch: giảm
    ├──► Supplier.balance:
    │      ├─ nếu reduceDebt > 0: giảm nợ phải trả NCC
    │      └─ nếu refundFromSupplier > 0: tạo phiếu thu từ NCC (sổ quỹ)
    ├──► PN gốc: cập nhật "Đã trả N/M items" (không hủy PN)
    └──► Giá vốn: KHÔNG recompute WAC (vẫn dùng cost cũ)
```

**Note quan trọng:** TPN **không thay đổi WAC** vì WAC chỉ tính khi nhập (xem `nhap-hang-deep-dive.md` § 4.2). Đây là behavior có thể tranh cãi — về mặt kế toán nghiêm, trả NCC cũng nên revert phần đóng góp vào WAC.

---

## 4. SO SÁNH 2 NGHIỆP VỤ — MA TRẬN

| Khía cạnh | Trả hàng bán (TH) | Trả hàng nhập (TPN) |
|---|---|---|
| Prefix mã | TH | TPN (hoặc TN) |
| Bắt buộc tham chiếu phiếu gốc | Không (có "Trả nhanh") | **Bắt buộc** PN gốc |
| Đối tác phiếu | Customer (optional, có thể "Khách lẻ") | Supplier (bắt buộc) |
| Auto trigger | ✅ Khi giao hàng chuyển hoàn | ❌ |
| Hủy được | ✅ (trừ phiếu auto từ chuyển hoàn) | ✅ |
| Có "Đổi hàng" inline | ✅ Mode đổi trả | ❌ |
| Tác động loyalty | ✅ Trừ điểm | ❌ |
| Tác động giá vốn | Không đổi | Không đổi (debatable) |
| Tác động HĐĐT | Cần lập HĐ điều chỉnh giảm | Cần điều chỉnh hóa đơn đầu vào |
| Tác động Sổ quỹ | Phiếu chi (refund) hoặc thu (đổi đắt) | Phiếu thu (refund) |
| App mobile | ✅ Đầy đủ 3 mode | ✅ Có (từ phiếu nhập) |
| App POS Android | ✅ 2 mode | ❌ Chỉ tạo từ Web |

---

## 5. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN

### 5.1 Trả hàng bán

| # | Pain | Đề xuất |
|---|---|---|
| 1 | "Trả nhanh" không có anti-fraud — KH có thể trả hàng không mua ở shop để lấy tiền | Yêu cầu **mã HĐ tối thiểu** hoặc xác thực bằng SĐT KH + lịch sử mua |
| 2 | Không có **chính sách trả hàng có time window** (vd chỉ trả trong 7 ngày) | Setting per category: max return window, hết hạn → block |
| 3 | Không có **lý do trả chuẩn hóa** (chỉ free-text note) — không phân tích được | Dropdown: Lỗi sản phẩm / Sai size / Đổi ý / Không vừa / Khác → analytics |
| 4 | Trả hàng có Serial nhưng không kiểm tra **tình trạng vật lý** SP — có thể nhận về hàng vỡ rồi bán lại | Workflow QC: phiếu "Hàng đợi kiểm" → kiểm xong mới về tồn bán được |
| 5 | Không có **restocking fee** tự động (phí 10% khi đổi ý) | Trường `restocking_fee_pct` per category, tự tính |
| 6 | HĐĐT điều chỉnh giảm vẫn làm tay — không tự động khi tạo TH | Workflow: tạo TH → trigger auto HĐĐT điều chỉnh + gửi cơ quan thuế |
| 7 | Phiếu TH auto-từ-vận-chuyển không hủy được — nếu hãng VC báo nhầm "chuyển hoàn" thì kẹt | Cơ chế "đảo TH" có audit chứ không phải hủy thuần |
| 8 | Không có **return abuse detection** (KH trả >X lần trong N ngày) | AI flag KH có pattern lạm dụng |
| 9 | Loyalty point bị trừ ngay khi tạo TH — KH có thể đã consume điểm rồi | Reserve điểm khi consume + clawback workflow |
| 10 | Đổi hàng đắt hơn nhưng KH không thanh toán đủ → vẫn cho hoàn thành | Validation: blocking thay vì warning |

### 5.2 Trả hàng nhập

| # | Pain | Đề xuất |
|---|---|---|
| 11 | Phải trả từ **đúng PN gốc** — nếu nhập 2 lần cùng SP từ 1 NCC, phải nhớ phiếu nào để trả | Cho phép trả "theo SP × NCC × Lô" thay vì theo PN |
| 12 | Không recompute giá vốn — về mặt kế toán không đúng nếu trả ngay sau nhập | Option: revert cost contribution khi TPN |
| 13 | Không có **return shipping label tích hợp** — phải tự liên hệ NCC | Tích hợp shipping cho TPN giống cho HĐ giao hàng |
| 14 | NCC không có view "tôi đã bị trả bao nhiêu" — không feedback loop | Supplier portal hiển thị returns |
| 15 | Không có **defect rate** tự động trên NCC scorecard | Compute `defect_rate = ∑TPN / ∑PN` per NCC, hiển thị supplier card |
| 16 | Trả hàng nhập không gắn với HĐĐT đầu vào điều chỉnh — phải làm thủ công | Auto-link và auto-tạo bản điều chỉnh |
| 17 | Không có **mass return** (trả 50 SP cùng lúc cho 1 NCC từ nhiều PN) — phải tạo nhiều TPN | Bulk TPN từ filter NCC |

### 5.3 Chung cho 2 loại

| # | Pain | Đề xuất |
|---|---|---|
| 18 | Không có **photo evidence** đính kèm phiếu trả (ảnh hàng lỗi) | Upload ảnh trên line + so sánh khi audit |
| 19 | Không có **return reason taxonomy** chuẩn hóa giữa 2 module | Shared taxonomy: lỗi sản xuất / hỏng vận chuyển / sai mô tả / khác |
| 20 | Báo cáo trả hàng chỉ aggregate, không có **drill-down theo lý do/SP/NCC/KH** | Report module nâng cấp với pivot |

---

## 6. CƠ HỘI ĐỘT PHÁ — TOP 5

| # | Sản phẩm/Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Return Workflow Engine** — chính sách trả theo SP/Category (time window, restocking fee, QC required) | Pain #2, #4, #5 — chuỗi cần policy nhất quán | Trung | Cao |
| 2 | **Return Fraud Detection AI** — flag KH lạm dụng, NV thông đồng, pattern bất thường | Pain #8, đặc biệt với marketplace high-volume | Cao | Cao |
| 3 | **NCC Defect Dashboard** — tự động compute defect rate + alert | Pain #15, gắn vendor scorecard | Trung | Cao (B2B) |
| 4 | **Auto HĐĐT điều chỉnh** cho cả TH và TPN | Pain #6, #16 — đang là pain compliance lớn | Trung | Rất cao (legal) |
| 5 | **Return Photo + AI Damage Assessment** — chụp ảnh hàng trả → AI đánh giá có bán lại được không | Pain #4, #18 | Cao | Trung |

---

## 7. ENTITY MODEL BỔ SUNG

Bổ sung 4 đối tượng gắn với nghiệp vụ Trả hàng vào danh sách entities:

| # | Đối tượng | Vai trò |
|---|---|---|
| 41 | **Phiếu Trả hàng bán (SalesReturn)** | Prefix TH, header + ReturnLines + ExchangeLines |
| 42 | **Phiếu Trả hàng nhập (PurchaseReturn)** | Prefix TPN, bắt buộc tham chiếu PN |
| 43 | **Return Reason** (Lý do trả) | Taxonomy chuẩn hóa, hiện thiếu |
| 44 | **Return Policy** (Chính sách trả) | Time window, restocking fee, QC required — per category — hiện chưa có |

---

## 8. TÓM LƯỢC

**Trả hàng KiotViet:**
- Là **2 nghiệp vụ ngược chiều**: TH (KH trả vào kho) và TPN (shop trả ra NCC) — cùng pattern phiếu nhưng đối tác và dòng tiền ngược nhau
- TH **mạnh ở UX**: 3 mode (theo HĐ / nhanh / đổi trả), auto-trigger từ vận chuyển hoàn, có trên cả Web/Mobile/POS
- TPN **chặt chẽ hơn**: bắt buộc tham chiếu PN gốc, không có "Trả nhanh", không có trên POS
- **Cả 2 đều yếu** ở: lý do chuẩn hóa, photo evidence, fraud detection, HĐĐT điều chỉnh tự động, return policy engine
- **Auto-tạo TH từ vận chuyển hoàn** là feature hiếm có ở các phần mềm tương tự — điểm cộng
- **Cơ hội đột phá:** Return Workflow Engine (policy theo SP), Fraud Detection AI, NCC Defect Dashboard, Auto HĐĐT điều chỉnh

Đây là module **đã giải quyết được happy path nhưng còn nhiều gap cho enterprise compliance (HĐĐT) và anti-abuse — match với segment cao cấp hơn mà KiotViet đang nhắm tới**.
