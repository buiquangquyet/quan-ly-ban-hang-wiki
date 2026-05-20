# NGHIỆP VỤ ĐẶT HÀNG NHẬP — KIOTVIET

**Phạm vi:** Mô tả **thuần nghiệp vụ** của module Đặt hàng nhập (Purchase Order). Không đi vào schema/code.

**Vị trí UI:** Menu top-bar **Mua hàng → Đặt hàng nhập** (URL `/man/#/OrderSupplier`)
**Mã chứng từ:** Tiền tố **DHN** (Đặt Hàng Nhập)

---

## 1. ĐẶT HÀNG NHẬP LÀ GÌ

**Đặt hàng nhập** (Purchase Order — PO) là **phiếu cam kết mua** từ shop gửi tới Nhà cung cấp (NCC). Đây là khâu trung gian giữa "có nhu cầu nhập" và "hàng đã về kho", giúp shop:

- **Khóa giá nhập** với NCC tại thời điểm đặt (chống biến động giá)
- **Lên kế hoạch dòng tiền** (biết trước bao nhiêu tiền sẽ chi cho NCC)
- **Lên kế hoạch kho** (biết trước hàng về khi nào, chuẩn bị chỗ)
- **Phối hợp với NCC** (NCC biết shop sẽ lấy gì, bao nhiêu — chuẩn bị trước)
- **Audit trail** (mọi cam kết với NCC đều có chứng từ)

→ Đặt hàng nhập là **hành động cam kết**, Nhập hàng là **hành động ghi nhận hàng thực về**.

### Phân biệt với module Đặt hàng (DH) đã có

| Khía cạnh | Đặt hàng (DH) | Đặt hàng nhập (DHN) |
|---|---|---|
| Đối tác | Khách hàng | Nhà cung cấp |
| Chiều hàng | Sẽ xuất kho (khi tạo HĐ) | Sẽ nhập kho (khi NCC giao) |
| Chiều tiền | Sẽ thu | Sẽ chi |
| Phiếu kết thúc workflow | Hóa đơn (HD) | Phiếu Nhập hàng (PN) |
| Có shipping integration | ✅ (đến KH) | ❌ (NCC tự ship đến shop) |

Pattern giống nhau (phiếu cam kết, partial fulfillment, có thể hủy), nhưng đối tượng và dòng tiền ngược chiều.

---

## 2. VAI TRÒ TRONG HỆ SINH THÁI

```
   ┌── shop nhận thấy "sắp hết hàng" ──────────────────────────┐
   │   (cảnh báo định mức / dự kiến hết hàng / kế hoạch sale)  │
   ▼                                                            │
[Đặt hàng nhập DHN######]                                       │
   │                                                            │
   │   ─ Gửi NCC (qua Email / điện thoại / hệ thống)            │
   │   ─ NCC xác nhận chốt giá + ngày giao                      │
   │                                                            │
   │   tác động sổ sách: "Đặt NCC" trên product detail tăng    │
   │   (KHÔNG tác động tồn kho thực)                            │
   ▼                                                            │
NCC giao hàng tới shop                                          │
   │                                                            │
   ▼                                                            │
[Phiếu Nhập hàng PN######]  ───► Tồn kho thực tăng              │
   │                          ─►  Công nợ phải trả NCC tăng     │
   │                          ─►  Giá vốn cập nhật WAC           │
   │                                                            │
   │   PN có thể nhập 1 phần — DHN chuyển sang "Nhập một phần"  │
   │                                                            │
   ▼                                                            │
Đặt NCC còn lại trên DHN cập nhật                               │
   (nếu nhập đủ → DHN chuyển "Hoàn thành")                      │
   (nếu chưa → giữ "Nhập một phần", chờ PN tiếp theo)          │
```

DHN liên thông với 5 đối tượng:
- **NCC (Supplier)** — nguồn hàng, đối tác cam kết
- **Sản phẩm** — cập nhật "Đặt NCC" trên product detail panel
- **Phiếu Nhập (PN)** — phiếu kết thúc workflow
- **Tồn kho** — chỉ qua PN (DHN không tác động trực tiếp)
- **Sổ quỹ** — không trực tiếp, nhưng giúp forecast cash out

---

## 3. CÁC TRẠNG THÁI WORKFLOW

Quan sát từ filter sidebar, Đặt hàng nhập có **4 trạng thái**:

| # | Trạng thái | Ý nghĩa nghiệp vụ | Hành động kế tiếp |
|---|---|---|---|
| 1 | **Phiếu tạm** | Mới tạo, đang dự thảo, chưa gửi NCC | Sửa, gửi NCC, hủy |
| 2 | **Đã xác nhận NCC** | NCC đã chốt giá + ngày giao | Chờ NCC giao, có thể tạo PN từ đây |
| 3 | **Nhập một phần** | Đã nhập 1 phần SL qua PN, còn dư chưa nhập đủ | Tiếp tục tạo PN nhập số còn lại, hoặc đóng phiếu nếu NCC không giao hết |
| 4 | **Hoàn thành** | Đã nhập đủ qua PN | Đóng phiếu |
| (5) | **Đã hủy** | Hủy bỏ phiếu | Immutable, vẫn lưu trong list |

### Workflow chuẩn (happy path)

```
Tạo → Phiếu tạm → (gửi NCC, NCC OK) → Đã xác nhận NCC
   → (NCC giao hàng đợt 1) → Nhập một phần
   → (NCC giao hàng đợt 2 đủ) → Hoàn thành
```

### Workflow rút gọn

```
Tạo → Phiếu tạm → (NCC giao luôn 1 lần đủ) → Hoàn thành
```

(Nếu shop tin NCC giao đúng, có thể skip "Đã xác nhận NCC")

### Workflow hủy

```
Bất kỳ trạng thái nào (trừ Hoàn thành) → [Hủy] → Đã hủy
   Hệ thống hỏi: hủy luôn các PN liên quan hay giữ?
```

---

## 4. CÁC FIELD NGHIỆP VỤ QUAN TRỌNG

### 4.1 Trên list (8 cột)

| Cột | Vai trò nghiệp vụ |
|---|---|
| Mã đặt hàng nhập | DHN###### — đối chiếu khi NCC giao hàng / kế toán |
| Thời gian | Ngày tạo DHN |
| **Nhà cung cấp** | Đối tác cam kết |
| **Ngày nhập dự kiến** | ⭐ Ngày NCC hứa giao hàng → key cho **kế hoạch kho + cash flow** |
| **Số ngày chờ** | ⭐ Tính từ ngày tạo → ngày nhập dự kiến (hoặc đến hiện tại nếu chưa nhập). Cảnh báo NCC chậm |
| **VAT nhập hàng** | Mức thuế suất áp dụng (0%/5%/8%/10%) — chuẩn HĐĐT đầu vào |
| **Cần trả NCC** | Tổng giá trị shop sẽ phải thanh toán NCC |

→ Đây là module duy nhất có cột **"Số ngày chờ"** — KiotViet đã proactive cảnh báo NCC chậm.

### 4.2 Trong phiếu

Header chính:
- Mã DHN###### (auto)
- Nhà cung cấp (chọn từ master)
- **Người nhận đặt** (NV phụ trách phiếu này)
- **Ngày đặt** + **Ngày nhập dự kiến** (key cho forecast)
- Chi nhánh nhập
- VAT (mức thuế)
- Trạng thái

Lines (hàng hóa):
- Sản phẩm + biến thể
- Đơn vị tính + quy đổi
- **Số lượng đặt** (cam kết) — vs **Số lượng đã nhập** thực tế (track qua PN)
- Đơn giá nhập (đã thỏa thuận)
- Giảm giá (NCC ưu đãi nếu có)
- Thành tiền
- VAT per line

Tổng:
- Tổng tiền hàng
- Giảm giá phiếu
- VAT tổng
- Tổng cộng (cần trả NCC)

### 4.3 Form tạo phiếu — luồng nhập liệu

Layout 2 cột chia rõ vùng:

**Vùng trái (chính):** danh sách hàng hóa
- Search bar "Tìm hàng hóa theo mã hoặc tên" — phím tắt **F3**
- Cách thêm hàng:
  1. Gõ search hoặc quét mã vạch (NCC gửi catalog có barcode)
  2. **Import từ file Excel** — "Chọn file dữ liệu" + link "Tải về file mẫu: File đặt hàng nhập"
  3. Click icon **+** thêm SP mới ngay trong phiếu (nếu shop chưa từng nhập SP này)
- Bảng lines mặc định: STT | Mã hàng | Tên hàng | ĐVT | Số lượng | Đơn giá | Giảm giá | VAT (%)

**Vùng phải (sidebar):** các thông tin phiếu
- **Tìm NCC** + button **+** tạo NCC mới ngay trong form (không cần thoát ra master)
- Mã DHN tự sinh (sửa được)
- Trạng thái (dropdown — default Phiếu tạm)
- Tổng tiền hàng ⓘ (auto)
- Giảm giá
- **VAT nhập hàng** — có **toggle bật/tắt** + nhập số
- **Chi phí nhập hàng** → giá trị + mũi tên (phân bổ vào giá vốn)
- **Cần trả nhà cung cấp** (computed = Tổng - giảm + VAT + chi phí trả NCC)
- **Tiền trả NCC (F8)** + icon ngân hàng — phím tắt F8 jump
- **Tiền mặt** (chọn quỹ — Tiền mặt / NH / Ví)
- **Tiền NCC trả lại** (âm số — nếu shop trả thừa)
- **Chi phí nhập khác** → phân bổ vào giá vốn nhưng KHÔNG tính vào nợ NCC
- **Dự kiến ngày nhập hàng** (date picker)
- Ghi chú (textarea)

**2 button bottom:**
- ⭐ **Đặt và gửi email** — tạo phiếu + gửi NCC email tự động (đỡ phải xuất file → mở mail → đính kèm → gửi)
- **Đặt hàng nhập** (primary) — chỉ tạo phiếu nội bộ

### 4.4 Phân biệt 2 loại chi phí

| Chi phí | Trả cho | Cộng vào nợ NCC? | Phân bổ giá vốn? |
|---|---|---|---|
| **Chi phí nhập hàng** | NCC (vd phí giao hàng NCC charge) | ✅ Có | ✅ Có |
| **Chi phí nhập khác** | Bên thứ 3 (vd thuê xe ngoài, bốc xếp) | ❌ Không | ✅ Có |

→ Cả 2 đều đẩy giá vốn lên (landed cost), nhưng chỉ "Chi phí nhập hàng" tính vào số phải trả NCC.

### 4.5 Tùy chọn hiển thị (icon mắt 👁) — personalize per user

Popup có 2 tab:

**Tab "Hiển thị" — cột trên bảng lines:**
- Ảnh hàng hóa (toggle)
- Thương hiệu (toggle)
- Tồn Kho (toggle — xem tồn hiện tại trong khi đặt → tránh đặt thừa)
- Sắp xếp thứ tự hàng hóa ↑↓

**Tab "Khác" — hành vi nhập liệu:**
- Thêm dòng (cho phép thêm dòng giá khác cho cùng 1 SP — khuyến mãi NCC "mua 10 tặng 1")
- Chế độ lọc (filter inline)
- Chọn nhiều hàng hóa (tích nhiều SP cùng lúc)
- ⭐ **Giá nhập là giá vốn** ⓘ — quyết định **có dùng đơn giá nhập trên phiếu này làm giá vốn mới (WAC update) hay giữ giá vốn cũ**
- Giảm giá: VND / % toggle mặc định

→ Setting **"Giá nhập là giá vốn"** là quyết định nghiệp vụ quan trọng:
- **Bật:** Giá nhập NCC trên phiếu → auto cập nhật WAC khi hàng về (qua PN). Đây là chế độ chuẩn cho hầu hết shop.
- **Tắt:** Giữ giá vốn cũ. Use case: NCC giao giá thấp đột biến (sale, khuyến mãi 1 lần) — không muốn kéo giá vốn xuống làm méo báo cáo lãi.

### 4.6 Sắp xếp + xuất nhanh

Top icon row trong form có:
- 👁 Tùy chọn hiển thị (đã mô tả §4.5)
- 🖨 In phiếu — in để gửi NCC giấy
- Icon "..." cho actions phụ (Xuất file Excel, Sao chép từ phiếu cũ...)

---

## 5. CÁC NGHIỆP VỤ NÂNG CAO

### 5.1 Nhập một phần (Partial Fulfillment)

Đây là **điểm mạnh** của DHN. Use case:
- Shop đặt 100 thùng nước. NCC giao đợt 1 = 60 thùng, đợt 2 = 40 thùng
- DHN → tạo PN001 ghi nhận 60 thùng → DHN chuyển "Nhập một phần"
- Tuần sau tạo PN002 ghi nhận 40 thùng → DHN chuyển "Hoàn thành"

Hệ thống tự tính SL còn lại của DHN = SL đặt − Σ SL đã nhập qua PN.

### 5.2 Nhập từ nhiều DHN trong 1 PN

Ngược lại: 1 phiếu PN có thể nhập gộp từ **nhiều DHN** cùng NCC. Use case: NCC giao 1 chuyến chở hàng của 3 đơn DHN khác nhau → shop tạo 1 PN gộp.

### 5.3 Chi phí nhập trả NCC

Filter sidebar có "Chi phí nhập trả NCC" — gắn chi phí (vận chuyển, bốc dỡ, bảo hiểm...) với DHN. Khi tạo PN từ DHN, chi phí này phân bổ vào giá vốn từng dòng (xem `02-hang-hoa-kho/nhap-hang-deep-dive.md` § 2.2).

### 5.4 Người nhận đặt (Buyer)

Trường này gắn DHN với 1 NV cụ thể của shop (vd: nhân viên thu mua / kế toán mua hàng). Filter theo NV → KAM thu mua xem portfolio mình quản lý.

### 5.5 Xuất file (Excel)

Top button "Xuất file ▼" có 2 option (suy luận từ pattern các module khác):
- **File tổng quan** — list các DHN với header
- **File chi tiết** — bóc tách từng line hàng

Use case: gửi NCC để đối soát, gửi kế toán để hạch toán.

### 5.6 Gửi email NCC trực tiếp ⭐

Button **"Đặt và gửi email"** bottom form — 1-click:
1. Tạo phiếu DHN
2. Render template email với nội dung phiếu
3. Gửi đến email của NCC (lấy từ master NCC)

→ Tiết kiệm 5-10 phút mỗi đơn so với workflow "Tạo → Xuất Excel → Mở Gmail → Đính kèm → Soạn email → Gửi".

### 5.7 Tạo NCC mới ngay trong form

Form DHN có icon **+** cạnh ô "Tìm NCC" — mở mini-modal tạo NCC mới mà không cần thoát ra menu Mua hàng → Nhà cung cấp.

Use case: lần đầu nhập từ NCC mới, đỡ phải chuyển màn hình.

### 5.8 Phím tắt thao tác nhanh

| Phím | Hành động |
|---|---|
| **F3** | Focus ô tìm hàng hóa |
| **F8** | Focus ô "Tiền trả NCC" để nhập thanh toán |

→ Phù hợp NV thu mua quen workflow (gõ phím nhanh hơn click chuột).

### 5.9 Thêm dòng giá khác cho cùng 1 SP

Bật setting "Thêm dòng" trong Tùy chọn hiển thị → cho phép add nhiều dòng cho cùng 1 SP với giá khác nhau. Use case:
- NCC "mua 10 tặng 1" → 1 dòng giá thật, 1 dòng giá 0
- Mua theo lô có giá tier (10 SP đầu giá 100k, 10 SP sau giá 90k) → 2 dòng

### 5.10 Quy tắc cập nhật giá vốn

Setting **"Giá nhập là giá vốn"** (Tùy chọn hiển thị → Khác):
- **Bật (default):** Khi DHN này được nhập qua PN, đơn giá nhập trên line → cập nhật vào WAC (Bình quân gia quyền) của SP
- **Tắt:** Giá vốn giữ nguyên cost cũ, không update từ phiếu này

Đây là setting **per-phiếu** — không phải global setting của shop. Mỗi DHN có thể bật/tắt độc lập theo ngữ cảnh nghiệp vụ.

---

## 6. SO SÁNH 4 PHIẾU LIÊN QUAN CHUỖI MUA HÀNG

| Khía cạnh | DHN | PN | TPN | HĐ đầu vào |
|---|---|---|---|---|
| Vai trò | Cam kết mua | Ghi nhận hàng về | Trả hàng cho NCC | HĐĐT thuế nhập |
| Mã prefix | DHN | PN | TPN | (theo CQT) |
| Tác động tồn | ❌ (chỉ "Đặt NCC") | ✅ Tăng | ✅ Giảm | ❌ |
| Tác động công nợ NCC | ❌ | ✅ Tăng | ✅ Giảm | ❌ (chỉ chứng từ thuế) |
| Tác động giá vốn (WAC) | ❌ | ✅ Update | ❌ (đã cố định) | ❌ |
| Bắt buộc tham chiếu phiếu trước | — | Optional (có thể tạo độc lập) | **Bắt buộc PN gốc** | Optional |
| Có "Nhập một phần" | ✅ | — | — | — |
| Có "Số ngày chờ" cảnh báo | ✅ | — | — | — |
| Trên menu | Mua hàng → Đặt hàng nhập | Mua hàng → Nhập hàng | Mua hàng → Trả hàng nhập | Mua hàng → HĐ đầu vào (badge "Mới") |

---

## 7. ĐIỂM MẠNH NGHIỆP VỤ HIỆN TẠI

1. **Workflow đầy đủ 4 trạng thái** rõ ràng — không có "ghost state" gây confused
2. **Cột "Số ngày chờ" độc nhất** — proactive cảnh báo NCC chậm (ít POS khác có)
3. **Cột "Ngày nhập dự kiến"** — gắn vào kế hoạch kho + cash flow
4. **Partial fulfillment thực sự** — 1 DHN có thể được nhập qua nhiều PN, system tự track SL còn lại
5. **Liên thông 2 chiều** với PN — nhập từ DHN hoặc gộp nhiều DHN
6. **VAT per line** — chuẩn HĐĐT đầu vào (TT78/2021)
7. **Phân quyền "Người nhận đặt"** — KAM thu mua có portfolio riêng
8. **Tách bạch tồn vật lý vs cam kết** — DHN không trừ tồn, không gây nhiễu báo cáo
9. **"Đặt và gửi email" 1-click** ⭐ — gửi NCC ngay khi tạo phiếu, tiết kiệm 5-10 phút/đơn
10. **Tạo NCC mới inline** trong form — không cần thoát menu master
11. **Phím tắt F3 (search) + F8 (thanh toán)** — phù hợp NV thu mua thao tác nhanh
12. **Tách 2 loại chi phí**: "Chi phí nhập hàng" (trả NCC, cộng vào nợ) vs "Chi phí nhập khác" (trả bên thứ 3, chỉ phân bổ giá vốn) — landed cost đúng nghiệp vụ
13. **Setting "Giá nhập là giá vốn" per-phiếu** — quyết định có update WAC từ phiếu này hay không, control giá vốn tinh tế
14. **Tùy chọn hiển thị personalize** — bật/tắt cột Ảnh / Thương hiệu / Tồn kho và hành vi nhập liệu (Thêm dòng giá, Chọn nhiều SP, Filter inline)

---

## 8. ĐIỂM ĐAU NGHIỆP VỤ

### 8.1 Workflow & Lifecycle

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có **"Đang chờ NCC xác nhận"** giữa Phiếu tạm và Đã xác nhận NCC — shop không biết NCC đã trả lời chưa | Thêm state "Đã gửi NCC, chờ phản hồi" |
| 2 | Không có **deadline tự động đóng DHN** khi NCC không giao (vd 30 ngày sau "Ngày nhập dự kiến") | Auto-close rule + alert |
| 3 | Cảnh báo "Số ngày chờ" hiển thị nhưng không có **alert proactive** (email/notif) khi vượt ngưỡng | Threshold setting + push alert |
| 4 | Không có **lý do hủy** chuẩn hóa (NCC hết hàng / shop đổi ý / giá tăng / khác) | Dropdown reason → analytics |

### 8.2 Liên thông NCC

| # | Pain | Đề xuất |
|---|---|---|
| 5 | "Gửi NCC" vẫn manual (email/zalo/sđt) — không có **portal NCC** để NCC xem + confirm online | NCC self-service portal |
| 6 | Không có **EDI light** — NCC tự update "đã giao N đợt" lên DHN của shop | Vendor sync API |
| 7 | Không có **NCC scorecard auto** (on-time delivery %, fill-rate %, defect %) từ data DHN/PN/TPN | Vendor performance dashboard (đã đề xuất ở `nhap-hang-deep-dive` § 7.5) |
| 8 | Không có **giá tham chiếu** từ lịch sử DHN trước với cùng NCC (gợi ý "giá lần trước = X, lần này NCC báo Y có hợp lý?") | Price benchmark inline |

### 8.3 Forecast & Planning

| # | Pain | Đề xuất |
|---|---|---|
| 9 | Không có **Smart Reorder Engine** — gợi ý "SP A sắp hết, NCC X cho ngày Y, đặt N cái?" dựa trên velocity + lead time | AI reorder suggestion |
| 10 | Không có **PO suggestion từ định mức tồn** — phải tay quét tồn rồi tạo DHN | Auto-generate DHN draft khi tồn xuống định mức |
| 11 | Không có **batch creation** — đặt 50 SP từ 5 NCC khác nhau phải tạo 5 phiếu thủ công | Bulk wizard: chọn SP → group theo NCC → auto-tạo nhiều DHN |
| 12 | "Ngày nhập dự kiến" không feed vào **cash flow forecast** trên Sổ quỹ | Integrated forecast |

### 8.4 3-Way Match Compliance

| # | Pain | Đề xuất |
|---|---|---|
| 13 | Không có **3-way match** (DHN ↔ PN ↔ Hóa đơn đầu vào) tự động | Auto-match + alert discrepancy |
| 14 | Phiếu PN có thể tạo độc lập không link DHN — bypass control | Setting "bắt buộc PO-first" cho enterprise |
| 15 | Không có **threshold approval** (DHN > N triệu cần manager duyệt) | Workflow approval theo ngưỡng |
| 16 | Không có **change order** chính thức — sửa DHN trực tiếp khi NCC đã confirm là dirty audit | Change order với version + ai duyệt |

### 8.5 Khác

| # | Pain | Đề xuất |
|---|---|---|
| 17 | Không có **template DHN** — đặt định kỳ hàng tháng cùng SP phải tạo lại | Save as template + 1-click duplicate |
| 18 | Không có **OCR phiếu báo giá NCC** → auto tạo DHN | OCR import (giống nhập hàng) |
| 19 | Không thấy app mobile hỗ trợ tạo DHN nhanh (NV thu mua ngoài shop) | Mobile shortcut |

---

## 9. CƠ HỘI ĐỘT PHÁ — TOP 5 NGHIỆP VỤ

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Smart Reorder AI** — gợi ý DHN tự động dựa trên velocity + lead time + định mức tồn | Pain #9, #10 | Cao | Rất cao |
| 2 | **3-way Match Auto + Vendor Scorecard** | Pain #7, #13 | Cao | Cao (enterprise) |
| 3 | **NCC Self-service Portal** — NCC tự confirm + update giao hàng | Pain #5, #6 | Cao | Rất cao (giảm 50% công NV thu mua) |
| 4 | **PO Approval Workflow** (theo ngưỡng giá trị + change order versioning) | Pain #15, #16 | Trung | Cao (chuỗi) |
| 5 | **Cash Flow Integration** — "Ngày nhập dự kiến" + "Cần trả NCC" feed vào Sổ quỹ forecast | Pain #12 | Trung | Cao |

---

## 10. TÓM LƯỢC

**Đặt hàng nhập KiotViet (nghiệp vụ):**
- Là **cầu nối cam kết** giữa shop và NCC — không tác động tồn vật lý, chỉ cập nhật "Đặt NCC" trên product
- 4 trạng thái workflow rõ ràng: Phiếu tạm → Đã xác nhận NCC → Nhập một phần → Hoàn thành (+ Đã hủy)
- **Mạnh:** Partial fulfillment thực sự (1 DHN có thể nhập qua nhiều PN), cột "Số ngày chờ" cảnh báo NCC chậm, "Ngày nhập dự kiến" cho forecast, VAT per line chuẩn HĐĐT, tách bạch "Người nhận đặt" cho KAM thu mua, **"Đặt và gửi email" 1-click**, **tạo NCC mới inline trong form**, **2 phím tắt F3/F8**, **tách 2 loại chi phí (NCC vs bên thứ 3)**, **setting "Giá nhập là giá vốn" per-phiếu**, tùy chọn hiển thị personalize
- **Yếu:** Không có Smart Reorder AI, không có NCC self-service portal, không có 3-way match auto, không có vendor scorecard, không có cash flow integration với Sổ quỹ, không có approval workflow theo ngưỡng, không có OCR phiếu báo giá
- **Cơ hội đột phá:** Smart Reorder AI, 3-way Match + Vendor Scorecard, NCC Self-service Portal, PO Approval Workflow, Cash Flow Integration

Đây là module **đã có nền tảng nghiệp vụ chắc cho SMB nhưng còn rất nhiều dư địa cho intelligence (AI reorder, vendor scoring) và collaboration (NCC portal) — match với segment chuỗi và shop có volume mua hàng lớn**.
