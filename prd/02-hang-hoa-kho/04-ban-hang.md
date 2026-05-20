# Nghiệp vụ Bán hàng — Góc tác động tồn kho

**Phạm vi:** File này tập trung **toàn bộ vào tác động lên tồn kho** của nghiệp vụ Bán hàng — không đi sâu vào UX màn POS (xem `01-pos-ban-hang/`).

Các khía cạnh được phân tích:
- Hóa đơn giảm tồn như thế nào (validation âm tồn, multi-unit, multi-branch)
- Đặt hàng (DH) reserve tồn ra sao
- Hủy hóa đơn rollback tồn như thế nào
- Hàng Serial/IMEI và Lô/HSD tác động khác hàng thường ra sao

**Entities:** E19 (Invoice), E20 (CustomerOrder), E21 (StockAggregate), E22 (Reservation)
**Liên quan:** [README.md](./README.md) | [02-the-kho.md](./02-the-kho.md) | [05-tra-hang.md](./05-tra-hang.md) | [06-lo-va-serial.md](./06-lo-va-serial.md)

---

## 1. Các phiếu bán hàng tác động tồn

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Đặt hàng (DH)  │───►│  Hóa đơn (HD)   │───►│ Trả hàng (TH)   │
│  Customer Order │    │  Sales Invoice  │    │ Customer Return │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│ KHÔNG giảm tồn  │    │ Giảm tồn        │    │ Tăng tồn lại    │
│ Chỉ "KH đặt"    │    │ Ghi Thẻ kho HD  │    │ Ghi Thẻ kho TH  │
│ (reservation)   │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

| Phiếu | Prefix | Tác động onHand | Tác động "KH đặt" (reserved) |
|---|---|---|---|
| Đặt hàng | DH | Không | +qty |
| Hóa đơn | HD | −qty | Trừ ra nếu phát sinh từ DH |
| Trả hàng | TH | +qty | — |

---

## 2. Cấu trúc dữ liệu liên quan đến tồn

### 2.1 Entity Invoice (Hóa đơn bán hàng — E19)

```
Invoice  (HD######)
├── id, branchId, customerId, soldAt, soldBy, channelId, priceBookId
├── status          enum   Hoàn_thành | Một_phần | Đã_hủy
├── shippingStatus  enum   Không_giao | Đang_giao | Đã_giao | Đã_chuyển_hoàn
├── eInvoiceStatus  enum   Chưa_phát_hành | Đã_phát_hành | Đã_điều_chỉnh
├── orderId         FK?    Link Đặt hàng nếu có (E20)
│
└── InvoiceLines
    ├── productId / variantId
    ├── unitId               FK → E02  ← nếu bán "thùng", quy đổi sang "cái" trước khi −tồn
    ├── unitConversion       decimal   Hệ số quy đổi (1 thùng = 24 chai)
    ├── quantity             decimal   Theo đơn vị unitId
    ├── stockQuantity        decimal   quantity × unitConversion (đơn vị cơ bản)
    ├── salePrice            decimal
    ├── costPriceAtSale      decimal   SNAPSHOT giá vốn WAC tại thời điểm bán
    ├── serialNumbers        string[]  Nếu là hàng Serial/IMEI (E34)
    ├── batchId              FK? → E32 Nếu là hàng Lô/HSD
    └── reservedFromOrderId  FK? → E20 Trừ ra từ "KH đặt" nếu HD phát sinh từ DH
```

### 2.2 Entity CustomerOrder (Đặt hàng khách — E20)

```
CustomerOrder  (DH######)
├── id, branchId, customerId, orderedAt
├── status       enum   Phiếu_tạm | Đã_đặt | Đã_xuất_HD | Đã_hủy
├── expectedDate date   Ngày giao dự kiến
└── OrderLines
    ├── productId / variantId / unitId
    ├── quantity
    ├── reservedQuantity      qty quy đổi về đơn vị cơ bản → "KH đặt"
    └── fulfilledQuantity     đã chuyển sang HD bao nhiêu
```

### 2.3 Entity StockAggregate (E21)

```
Stock  (productId × branchId)  — denormalized aggregate, cập nhật realtime
├── onHand                decimal  Tồn vật lý hiện tại
├── reservedByCustomer    decimal  ∑ qty từ DH chưa xuất HD ("KH đặt")
├── reservedFromSupplier  decimal  ∑ qty từ DHN chưa nhập ("Đặt NCC")
└── available             decimal  onHand − reservedByCustomer  (số hiển thị POS)
```

`available` là số hiển thị dưới dạng "Còn lại" trên màn POS để nhân viên không bán quá.

---

## 3. State machine — Luồng từ bán đến giảm tồn

```
        ┌───────────────────────────────────────┐
        │ Bán nhanh / Bán thường (không qua DH) │
        │  → tạo HD trực tiếp                   │
        └─────────────────┬─────────────────────┘
                          │
[Đặt hàng — DH]──────►┌──HD──────────────────────┐
   ├─ Phiếu tạm        │  HD######                │
   ├─ Đã đặt (+reserved)│  Hoàn thành (−onHand)   │
   └─ Đã xuất HD        └─┬─────────────────────── ┘
                           │
                           ├──► (giao hàng) ─► Đang giao ─► Đã giao
                           │                                    │
                           │                          Đã chuyển hoàn ─► Auto TH ─► +tồn
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
| Tạo HD trực tiếp | Hoàn tất tại POS | `onHand −= q` |
| Hủy HD | Hủy bỏ | `onHand += q`, tạo dòng Thẻ kho đảo |
| Hủy DH chưa xuất HD | Trên list DH | `reservedByCustomer −= q` |

### 3.2 Validation tồn khi bán

| Check | Behavior KiotViet |
|---|---|
| Bán SL > onHand | Cho phép (tồn âm) |
| Bán SL > available (đã có DH giữ chỗ) | Cho phép — nhưng cảnh báo popup |
| Bán SP ngừng kinh doanh tại CN | Block |
| Bán SP có `Tag bán trực tiếp = false` | Không hiển thị POS — vẫn thêm được qua search |
| Bán hàng Serial mà serial đã bán | Block (serial uniqueness) |
| Bán hàng Lô đã hết hạn | Cảnh báo, có thể bypass tùy setting |

---

## 4. Giá vốn tại thời điểm bán (Cost at Sale)

### 4.1 Snapshot vs Recompute

KiotViet **snapshot** `costPriceAtSale` tại thời điểm tạo HD:
```
costPriceAtSale = product.costPrice  (WAC merchant level tại t = soldAt)
```

Lãi gộp dòng = `(salePrice − costPriceAtSale) × stockQuantity`

### 4.2 Hệ quả của snapshot

| Tình huống | Hành vi |
|---|---|
| Nhập hàng sau khi đã bán (giá nhập cao hơn) | HD cũ không update — lãi gộp lịch sử giữ nguyên |
| Hủy HD và bán lại | Cost recompute từ WAC mới |
| Bán âm tồn (chưa từng nhập) | `costPriceAtSale = 0` → lãi gộp = giá bán → con số méo |
| Backdate HD vào quá khứ | Cost lấy theo WAC **hiện tại**, KHÔNG phải WAC tại thời điểm backdate |

---

## 5. Nghiệp vụ đặc biệt tác động khác thường

### 5.1 Hàng Combo

```
Combo: "Set quà sinh nhật" = 1 áo + 1 mũ + 1 khăn

Khi bán 1 combo:
  → Combo không có stock độc lập
  → Tồn giảm: −1 áo, −1 mũ, −1 khăn (theo BOM combo)

UI hiển thị: combo qty = min(stock_component / qty_in_combo)
```

### 5.2 Hàng Dịch vụ

→ KHÔNG tác động tồn. Flag `productType = service`. Vẫn ghi vào HD line nhưng skip toàn bộ stock logic.

### 5.3 Hàng Serial / IMEI (xem [06-lo-va-serial.md](./06-lo-va-serial.md))

```
Serial pool per (productId × branchId):
  ├─ available_serials = [SN001, SN002, SN003, ...]
  └─ sold_serials      = [SN_X, SN_Y]   (immutable history)

Khi bán: phải chọn cụ thể serial nào → −1 từ available, +1 vào sold
Khi trả: serial phải khớp với HD → +1 lại vào available
```

### 5.4 Hàng Lô / Hạn sử dụng (xem [06-lo-va-serial.md](./06-lo-va-serial.md))

```
Lot pool per (productId × branchId × lotCode):
  ├─ qty_per_lot:    {Lô_A: 50, Lô_B: 100, Lô_C: 30}
  └─ expiry_per_lot: {Lô_A: 2026-12-31, Lô_B: 2027-06-30, Lô_C: 2026-08-15}

Khi bán:
  ├─ Option 1: Chọn lô thủ công
  └─ Option 2 (mặc định): FEFO auto (First-Expired-First-Out)
```

### 5.5 Bán nhiều đơn vị (Multi-UoM)

```
SP "Bia Sài Gòn": cái = 10k, lốc = 55k (1 lốc = 6 cái), thùng = 200k (1 thùng = 24 cái)

Bán 1 thùng:
  ├─ InvoiceLine: unitId = thùng, quantity = 1, unitConversion = 24
  └─ Stock: onHand −= 24  (luôn quy về đơn vị cơ bản "cái")
```

---

## 6. Điểm đau & cơ hội cải tiến

### 6.1 Validation & độ chính xác

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Cho phép bán âm tồn không cảnh báo cứng — nhân viên bán nhầm SP không có, đến lúc nhập lại quên | Setting "block âm tồn" per category / per branch |
| 2 | Sửa HD ở Quản lý KHÔNG validate lại tồn | Re-validate khi sửa SL line, hoặc cảnh báo rõ |
| 3 | Backdate HD lấy WAC hiện tại — sai theo kế toán | Cost engine có versioning, query "WAC at time X" |
| 4 | `costPriceAtSale = 0` khi chưa nhập → báo cáo lãi méo | UI flag "chưa có giá vốn" thay vì hiển thị lãi = giá bán |
| 5 | "KH đặt" (reserved) chỉ trừ vào Available hiển thị — nhân viên vẫn bán được phần đã đặt | Setting block bán phần reserved |

### 6.2 Concurrency & race conditions

| # | Pain | Đề xuất |
|---|---|---|
| 6 | 2 nhân viên ở 2 POS bán cùng 1 SP có SL=1 — race condition | Optimistic locking với version trên Stock record |
| 7 | App offline sync về → conflict | Conflict resolution: cho phép âm tồn + flag "cần kiểm" |
| 8 | Mass import HD từ Excel không re-validate tồn | Per help: "cần đảm bảo tồn đáp ứng" — UI warning + dry-run mode |

### 6.3 Multi-branch & multi-warehouse

| # | Pain | Đề xuất |
|---|---|---|
| 9 | Bán hàng online (TMĐT) phải config CN nguồn cố định — không smart routing | Auto-allocate đơn TMĐT cho CN có hàng + gần KH nhất |
| 10 | Không có "transfer in-flight" — đang chuyển CN1→CN2, CN2 có thể bán trước khi nhận | Reserved-in-transit flag tạm thời |
| 11 | Tồn an toàn (safety stock) không tác động bán hàng — vẫn cho bán xuống dưới ngưỡng | Setting block bán dưới safety stock |

### 6.4 Liên thông HĐĐT

| # | Pain | Đề xuất |
|---|---|---|
| 12 | HĐĐT phát hành sau khi tồn đã trừ — nếu phát hành lỗi và muốn hủy HĐ thì tồn đã giảm | Trạng thái phụ "Đã giao, chờ HĐĐT" — không trừ tồn cứng cho đến khi HĐĐT thành công |
| 13 | Sửa HD đã có HĐĐT → hủy + tạo mới — gãy chuỗi audit | Workflow điều chỉnh + HĐĐT thay thế đồng bộ |

---

## 7. Cơ hội đột phá — Top 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Smart Stock Validation Engine** — block/cảnh báo bán âm tồn theo policy SP × CN | Pain #1, #5, #11 | Trung | Rất cao |
| 2 | **Real-time Stock Engine** — pessimistic lock cho SP cuối cùng, eventual consistency phần còn lại | Pain #6, #7, race condition omni-channel | Cao | Cao |
| 3 | **Cost-Versioning at Sale** — snapshot WAC tại t=soldAt, query historic cost cho backdate | Pain #3, #4 — kế toán chính xác | Cao | Cao (kế toán) |
| 4 | **Smart Order Routing** — chọn CN xuất hàng dựa tồn + khoảng cách + lead time | Pain #9 — chuỗi multi-branch | Cao | Rất cao |
| 5 | **HĐĐT-aware Inventory** — tồn chỉ trừ cứng khi HĐĐT phát hành thành công | Pain #12, #13 — compliance | Trung | Cao (legal) |

---

## 8. Tóm lược

**Bán hàng KiotViet — Góc tồn kho:**
- **3 phiếu** từ luồng bán hàng tác động khác nhau: DH (chỉ reservation), HD (trừ tồn), TH (hoàn lại)
- Multi-UoM với quy đổi tự động về đơn vị cơ bản — chính xác về mặt số học
- Snapshot `costPriceAtSale` cho phép tính lãi gộp ổn định nhưng **không recompute khi backdate**
- **Cho phép âm tồn** — phù hợp SMB pre-order nhưng nguy hiểm cho enterprise
- **Yếu ở:** Validation âm tồn không cấu hình được, race condition multi-POS, không recompute cost lịch sử, HĐĐT/tồn không atomic, smart routing đơn online chưa có

Bán hàng là **điểm cuối của data tồn kho** — sai ở đây thì toàn bộ báo cáo lãi/lỗ và sổ kế toán bị méo. Đã ổn cho operation hàng ngày SMB — còn nhiều dư địa cho compliance kế toán và chuỗi đa kênh.
