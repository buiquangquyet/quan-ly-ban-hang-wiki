# ĐÀO SÂU: NGHIỆP VỤ BÁN HÀNG — GÓC TÁC ĐỘNG TỒN KHO

**Phạm vi:** File này KHÔNG đi sâu vào UX màn POS (xem `01-pos-ban-hang/phan-tich-man-hinh-ban-hang.md`), mà tập trung **toàn bộ vào tác động lên tồn kho** của nghiệp vụ Bán hàng:

- Bán hàng giảm tồn như thế nào (FIFO/WAC, validation âm tồn, multi-unit, multi-branch)
- Đặt hàng (Order) reserve tồn ra sao
- Hủy hóa đơn rollback tồn như thế nào
- Hàng Serial/IMEI và Lô/Hạn sử dụng tác động khác hàng thường ra sao

**Tham chiếu:**
- Help chính thức: `kiotviet.vn/huong-dan-su-dung-kiotviet/retail-ban-hang/ban-hang/`
- Help đặt hàng: `kiotviet.vn/huong-dan-su-dung-kiotviet/retail-dat-hang/dat-hang/`

---

## 1. CÁC LOẠI PHIẾU BÁN HÀNG TÁC ĐỘNG TỒN

KiotViet có 3 loại artifact từ luồng bán hàng, tác động tồn khác nhau:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Đặt hàng (DH)  │───►│  Hóa đơn (HD)   │───►│ Trả hàng (TH)   │
│  Customer Order │    │  Sales Invoice  │    │ Customer Return │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ KHÔNG giảm tồn  │    │ ✅ Giảm tồn     │    │ ✅ Tăng tồn lại │
│ Chỉ "KH đặt"    │    │ Ghi Thẻ kho HD  │    │ Ghi Thẻ kho TH  │
│ (reservation)   │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

| Phiếu | Prefix | Tác động Stock (qty) | Tác động "KH đặt" (reserved) |
|---|---|---|---|
| Đặt hàng | DH | Không | +qty |
| Hóa đơn | HD | −qty | Trừ ra nếu phát sinh từ DH |
| Trả hàng | TH | +qty (xem `tra-hang-deep-dive.md`) | — |

---

## 2. CẤU TRÚC DỮ LIỆU LIÊN QUAN ĐẾN TỒN

### 2.1 Entity `Invoice` (Hóa đơn) — chỉ phần liên quan tồn

```
Invoice  (HD######)
├── id, branchId, customerId, soldAt, soldBy, channelId, priceBookId
├── status                  enum   Hoàn_thành | Một_phần | Đã_hủy
├── shippingStatus          enum   Không_giao | Đang_giao | Đã_giao | Đã_chuyển_hoàn
├── eInvoiceStatus          enum   Chưa_phát_hành | Đã_phát_hành | Đã_điều_chỉnh
├── orderId                 FK?    Link Đặt hàng nếu có
├── canceledAt / canceledBy
│
└── InvoiceLines
    ├── productId / variantId
    ├── unitId               FK UoM     ← QUAN TRỌNG: nếu bán "thùng", phải quy đổi sang "cái" trước khi −tồn
    ├── unitConversion       decimal    Hệ số quy đổi (1 thùng = 24 chai)
    ├── quantity             decimal    Theo đơn vị unitId
    ├── stockQuantity        decimal    quantity × unitConversion (theo đơn vị cơ bản)
    ├── salePrice            decimal
    ├── costPriceAtSale      decimal    SNAPSHOT giá vốn tại thời điểm bán (cho lãi gộp)
    ├── serialNumbers        string[]   Nếu là hàng Serial/IMEI
    ├── batchLotId           FK?        Nếu là hàng Lô/Hạn sử dụng
    ├── reservedFromOrderId  FK?        Trừ ra từ "KH đặt" nếu HD phát sinh từ DH
    └── stockCardId          FK         Ghi vào Thẻ kho (HD######)
```

### 2.2 Entity `CustomerOrder` (Đặt hàng) — reservation

```
CustomerOrder  (DH######)
├── id, branchId, customerId, orderedAt
├── status            enum   Phiếu_tạm | Đã_đặt | Đã_xuất_HD | Đã_hủy
├── expectedDate      date   Ngày giao dự kiến (hiển thị "KH đặt N ngày tới")
└── OrderLines
    ├── productId / variantId / unitId
    ├── quantity                    (qty theo unit)
    ├── reservedQuantity            (qty quy đổi về đơn vị cơ bản, dùng cho "KH đặt")
    └── fulfilledQuantity           (đã chuyển sang HD bao nhiêu)
```

### 2.3 Entity `Stock` (Tồn ma trận SP × CN)

```
Stock  (productId, branchId)  — denormalized aggregate
├── onHand              decimal   Tồn vật lý hiện tại
├── reservedByCustomer  decimal   ∑ quantity từ DH chưa xuất HD ("KH đặt")
├── reservedFromSupplier decimal  ∑ qty từ DHN chưa nhập ("Đặt NCC")
└── available           decimal   onHand − reservedByCustomer  (số khả dụng để bán mới)
```

**Quy tắc:** `available` là số hiển thị trên màn POS dưới dạng "Còn lại" để nhân viên không bán quá.

---

## 3. STATE MACHINE — LUỒNG TỪ BÁN ĐẾN GIẢM TỒN

```
                ┌───────────────────────────────────────┐
                │ Bán nhanh / Bán thường (không qua DH) │
                │  → tạo HD trực tiếp                   │
                └─────────────────┬─────────────────────┘
                                  │
[Đặt hàng — DH]──────►┌─Hóa đơn─────────────────┐
   ├─ Phiếu tạm        │  HD######               │
   ├─ Đã đặt          │  Hoàn thành             │
   │  (+reserved)      │  (−onHand)              │
   └─ Đã xuất HD       │                         │
                       └─┬───────────────────────┘
                         │
                         ├──► (giao hàng) ─► Đang giao ─► Đã giao
                         │                         │
                         │                         └─► Đã chuyển hoàn ─► Auto TH ─► +tồn
                         │
                         ├──► (Hủy) ─► Đã hủy ─► +tồn (rollback)
                         │
                         └──► (Trả 1 phần) ─► TH ─► +tồn phần trả
```

### 3.1 Khi nào "trừ tồn" thực sự xảy ra

| Action | Trigger | Tác động |
|---|---|---|
| Tạo DH | Hoàn thành đặt hàng | `reservedByCustomer += q`, KHÔNG đổi onHand |
| Tạo HD từ DH | Chuyển DH → HD | `reservedByCustomer −= q`, `onHand −= q` |
| Tạo HD trực tiếp (không qua DH) | Hoàn tất tại màn POS | `onHand −= q` |
| Sửa HD đã tạo (thay đổi SL hàng hóa) | Trên màn Quản lý | **Hủy HD cũ + tạo HD mới** (per help). KiotViet **không validate lại tồn** khi sửa |
| Hủy HD | Trên Quản lý → Hủy bỏ | `onHand += q`, tạo dòng Thẻ kho đảo |
| Hủy DH chưa xuất HD | Trên list DH | `reservedByCustomer −= q` |

### 3.2 Validation tồn khi bán

| Check | Behavior KiotViet |
|---|---|
| Bán SL > onHand | Cho phép (tồn âm) — match với phát hiện trong `the-kho-deep-dive.md` § 1.3 |
| Bán SL > available (đã có DH giữ chỗ) | Cho phép — nhưng cảnh báo trên popup |
| Bán SP không kinh doanh tại CN này (per-branch `disabled`) | Block |
| Bán SP có `Tag bán trực tiếp = false` | Không hiển thị trên POS, nhưng vẫn cho thêm bằng search |
| Bán hàng Serial mà serial đã bán rồi | Block (serial uniqueness) |
| Bán hàng Lô mà lô hết hạn | Cảnh báo, có thể bypass tùy setting |

---

## 4. GIÁ VỐN TẠI THỜI ĐIỂM BÁN (COST AT SALE)

### 4.1 Snapshot vs Recompute

KiotViet **snapshot** `costPriceAtSale` tại thời điểm tạo HD:
```
costPriceAtSale = product.costPrice  (WAC merchant level tại t=soldAt)
```

→ Lãi gộp dòng = `(salePrice − costPriceAtSale) × stockQuantity`

### 4.2 Hệ quả

| Tình huống | Hành vi |
|---|---|
| Nhập hàng sau khi đã bán (giá nhập cao hơn) | HD cũ không update — lãi gộp lịch sử giữ nguyên |
| Hủy HD và bán lại | Cost recompute từ WAC mới |
| Bán âm tồn (chưa từng nhập) | `costPriceAtSale = 0` → lãi gộp = giá bán → con số méo |
| Backdate HD vào quá khứ | Cost lấy theo WAC **hiện tại**, KHÔNG phải WAC tại thời điểm backdate — pain |

---

## 5. CÁC NGHIỆP VỤ ĐẶC BIỆT TÁC ĐỘNG KHÁC THƯỜNG

### 5.1 Hàng Combo / Đóng gói

```
Combo: "Set quà sinh nhật" = 1 áo + 1 mũ + 1 khăn

Khi bán 1 combo:
  ├─ Thẻ kho ghi: 1 dòng cho "Combo" hoặc 3 dòng cho 3 SP con?
  └─ Tồn giảm:    Theo BOM combo (−1 áo, −1 mũ, −1 khăn)
                  Combo "ảo" không có stock độc lập
```

→ Combo **không có stock**, chỉ giảm tồn các thành phần. UI hiển thị combo qty = `min(stock_component / qty_in_combo)`.

### 5.2 Hàng Dịch vụ

→ KHÔNG tác động tồn. Có flag `productType = service`. Vẫn ghi vào HD line nhưng skip toàn bộ stock logic.

### 5.3 Hàng Sản xuất / BOM

→ Khi bán thành phẩm có BOM "tự động": có thể tự xuất NVL ngay (sản xuất ngầm). Hiện tại KiotViet làm bằng phiếu Sản xuất riêng (xem § C3 tổng quan), **không auto** khi bán.

### 5.4 Hàng Serial/IMEI

```
Serial pool per (productId × branchId):
  ├─ available_serials = [SN001, SN002, SN003, ...]
  └─ sold_serials      = [SN_X, SN_Y]   (immutable history)

Khi bán: phải chọn cụ thể serial nào → −1 từ available, +1 vào sold
Khi trả: serial phải khớp với HD → +1 lại vào available
```

### 5.5 Hàng Lô/Hạn sử dụng

```
Lot pool per (productId × branchId × lotCode):
  ├─ qty_per_lot:    {Lô_A: 50, Lô_B: 100, Lô_C: 30}
  └─ expiry_per_lot: {Lô_A: 2026-12-31, Lô_B: 2027-06-30, Lô_C: 2026-08-15}

Khi bán: 
  ├─ Option 1: Chọn lô thủ công
  └─ Option 2 (recommended): FEFO auto (First-Expired-First-Out)
```

(Xem `lo-han-su-dung-deep-dive.md` để hiểu sâu)

### 5.6 Bán nhiều đơn vị (Multi-UoM)

```
SP "Bia Sài Gòn": cái=10k, lốc=55k (1 lốc=6 cái), thùng=200k (1 thùng=24 cái)

Bán 1 thùng:
  ├─ InvoiceLine: unitId=thùng, quantity=1, unitConversion=24
  └─ Stock: onHand −= 24  (luôn quy về đơn vị cơ bản "cái")
```

---

## 6. ĐIỂM ĐAU & CƠ HỘI CẢI TIẾN — GÓC TỒN KHO

### 6.1 Validation & độ chính xác

| # | Pain | Đề xuất |
|---|---|---|
| 1 | **Cho phép bán âm tồn không cảnh báo cứng** — nhân viên có thể bán nhầm SP shop không có, đến lúc nhập lại quên → khó audit | Setting "block âm tồn" per category hoặc per branch; default false cho compatibility |
| 2 | Sửa HD ở Quản lý KHÔNG validate lại tồn (per help: "hệ thống sẽ không kiểm tra lại") | Re-validate khi sửa SL line, hoặc cảnh báo rõ |
| 3 | Backdate HD lấy WAC hiện tại — sai theo kế toán | Cost engine có versioning, query "WAC at time X" |
| 4 | `costPriceAtSale = 0` khi chưa nhập → báo cáo lãi méo | UI flag "chưa có giá vốn" thay vì hiển thị lãi = giá bán |
| 5 | "KH đặt" (reserved) chỉ trừ vào "Available" hiển thị, KHÔNG cứng — nhân viên vẫn bán được phần đã đặt | Setting block bán phần reserved cho hộ chuyên nghiệp |

### 6.2 Concurrency & race conditions

| # | Pain | Đề xuất |
|---|---|---|
| 6 | 2 nhân viên ở 2 POS bán cùng 1 SP có SL=1 — race condition có thể tạo 2 HD trùng tồn cuối cùng | Optimistic locking với version trên Stock record |
| 7 | App offline khi bán xong sync về thì có thể conflict | Conflict resolution: cho phép âm tồn + flag "cần kiểm" |
| 8 | Mass import HD từ Excel không re-validate tồn | Per help: "cần đảm bảo tồn đáp ứng, hệ thống không kiểm" — UI warning + dry-run mode |

### 6.3 Multi-branch & multi-warehouse

| # | Pain | Đề xuất |
|---|---|---|
| 9 | Bán hàng online (channel TMĐT) phải config CN nguồn cố định — không có smart routing theo tồn | Auto-allocate đơn TMĐT cho CN có hàng + gần KH nhất (đã đề xuất ở brainstorm B2) |
| 10 | Không có "transfer in-flight" — đang chuyển hàng CN1→CN2, CN2 có thể commit bán trước khi nhận | Reserved-in-transit flag tạm thời |
| 11 | Tồn an toàn (safety stock) không tác động bán hàng — vẫn cho bán xuống dưới ngưỡng | Setting block bán dưới safety stock |

### 6.4 Performance khi shop quy mô

| # | Pain | Đề xuất |
|---|---|---|
| 12 | Khi sản phẩm có >10k dòng Thẻ kho, mở tab Thẻ kho chậm | Index + pagination (xem `the-kho-deep-dive.md` § 5.3) |
| 13 | Mỗi HD line ghi 1 dòng StockCard + update Stock aggregate + có thể tạo phiếu chi sổ quỹ — transaction nặng | Bulk write, eventual consistency cho aggregate |

### 6.5 Liên thông HĐĐT

| # | Pain | Đề xuất |
|---|---|---|
| 14 | HĐĐT phát hành sau khi tồn đã trừ — nếu phát hành lỗi và muốn hủy HĐ thì tồn đã giảm | Trạng thái phụ "Đã giao hàng, chờ HĐĐT" — không trừ tồn cứng cho đến khi HĐĐT thành công |
| 15 | Sửa HD đã có HĐĐT thì hủy + tạo mới — gãy chuỗi audit | Workflow điều chỉnh + HĐĐT thay thế đồng bộ |

---

## 7. CƠ HỘI ĐỘT PHÁ — TOP 5 GẮN VỚI TỒN KHO

| # | Sản phẩm/Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Smart Stock Validation Engine** — block/cảnh báo bán âm tồn theo policy SP × CN, có override với lý do | Pain #1, #5, #11 | Trung | Rất cao |
| 2 | **Real-time Stock Engine** — pessimistic lock cho SP cuối cùng, eventual consistency cho phần còn lại | Pain #6, #7, race condition omni-channel | Cao | Cao |
| 3 | **Cost-Versioning at Sale** — snapshot WAC tại t=soldAt, query historic cost cho backdate | Pain #3, #4 — báo cáo kế toán chính xác | Cao | Cao (kế toán) |
| 4 | **Smart Order Routing** — chọn CN xuất hàng dựa tồn + khoảng cách + lead time | Pain #9 — match với chuỗi multi-branch | Cao | Rất cao |
| 5 | **HĐĐT-aware Inventory** — tồn chỉ trừ cứng khi HĐĐT phát hành thành công | Pain #14, #15 — compliance + zero loss case | Trung | Cao (legal) |

---

## 8. ENTITY MODEL BỔ SUNG

Bổ sung 4 đối tượng gắn với góc tồn kho của Bán hàng:

| # | Đối tượng | Vai trò |
|---|---|---|
| 45 | **Phiếu Bán hàng / Hóa đơn (Invoice)** | Prefix HD, header + InvoiceLines với snapshot cost |
| 46 | **Đặt hàng / Customer Order** | Prefix DH, sinh `reservedByCustomer` tác động "Available" |
| 47 | **Reservation (Đặt chỗ)** | Số liệu derived từ DH/DHN tác động "Available" cho POS |
| 48 | **Stock Aggregate** (onHand / reserved / available) | Bảng denormalized đọc nhanh cho UI, sync từ Thẻ kho |

---

## 9. TÓM LƯỢC

**Bán hàng KiotViet — Góc tồn kho:**
- **3 phiếu** từ luồng bán hàng tác động khác nhau: DH (chỉ reservation), HD (trừ tồn thật), TH (hoàn lại)
- Hỗ trợ **multi-UoM** với quy đổi tự động về đơn vị cơ bản — chính xác về mặt số học
- Snapshot `costPriceAtSale` tại thời điểm bán cho phép tính lãi gộp ổn định nhưng **không recompute khi backdate**
- **Cho phép âm tồn** ở mọi nơi — phù hợp với SMB pre-order nhưng nguy hiểm cho enterprise
- Hàng đặc biệt (Serial/IMEI, Lô/HSD, Combo, Sản xuất) có logic giảm tồn riêng — đã handle tương đối tốt
- **Yếu ở:** Validation âm tồn không cấu hình được, race condition multi-POS, không recompute cost lịch sử, HĐĐT/tồn không atomic, smart routing đơn online chưa có
- **Cơ hội đột phá:** Smart Stock Validation policy, Real-time Stock Engine với locking thông minh, Cost Versioning, Smart Order Routing, HĐĐT-aware Inventory

Bán hàng **là điểm cuối của data tồn kho** — sai ở đây thì toàn bộ báo cáo lãi/lỗ và sổ kế toán bị méo. Đây là module **đã ổn cho operation hàng ngày của SMB nhưng còn nhiều dư địa cho compliance kế toán và chuỗi đa kênh**.
