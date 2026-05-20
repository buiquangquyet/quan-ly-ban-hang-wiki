# Nhóm khách hàng & Segmentation — Deep Dive

**Bối cảnh:** File này tập trung vào **CustomerGroup + Rule Engine** — tính năng quan trọng nhất của module Khách hàng nhưng hay bị đánh giá thấp. KiotViet CÓ rule engine khá đầy đủ cho dynamic segmentation, không chỉ là "tag thủ công" như nhiều người giả định.

**Entities:** C04 (CustomerGroup), C05 (CustomerGroupRule)
**Liên quan:** [README.md](./README.md) | [01-tong-quan.md](./01-tong-quan.md)

---

## 1. Nhóm khách hàng (CustomerGroup — C04)

### 1.1 Schema

```
CustomerGroup (C04)
├── id, name, description
├── color          hex   — màu tag hiển thị trên list
├── discountValue  decimal   — giảm giá tự động khi KH nhóm này mua
├── discountType   enum      — % hoặc VND
├── assignedToId   FK User   — người phụ trách mặc định cho nhóm
└── note           string
```

**Điểm quan trọng:** Nhóm KH có thể gắn **giảm giá tự động** (`discountValue`). Khi KH thuộc nhóm này mua hàng, hệ thống tự áp giảm mà không cần NV nhớ. Đây là cơ chế pricing đơn giản nhưng hiệu quả cho seller B2B thường xuyên.

### 1.2 Quan hệ

```
Customer (C01) ──── many-to-many ──── CustomerGroup (C04)
    │                                        │
    └─ groupIds: FK[]                         └─ rule (C05) → auto add/remove KH
```

1 KH có thể thuộc **nhiều nhóm** cùng lúc (multi-group). Giảm giá áp dụng theo nhóm nào cao nhất hoặc stack — cần verify thêm.

---

## 2. Rule engine — CustomerGroupRule (C05)

### 2.1 Giao diện modal "Thêm nhóm khách hàng"

Modal có **2 tab:**

**Tab "Thông tin":** Tên nhóm, Giảm giá, Người phụ trách, Ghi chú.

**Tab "Thiết lập nâng cao":** Rule builder

```
Thiết lập điều kiện thêm khách hàng vào nhóm:
  [Tổng bán (trừ trả hàng) ▼]  [> ▼]  [0_______]   [🗑]
  + Thêm điều kiện

Chế độ:
  ◉ Thêm khách hàng vào nhóm theo điều kiện
  ○ Cập nhật lại danh sách theo điều kiện
  ○ Không cập nhật danh sách khách hàng

☐ Hệ thống thực hiện tự động  ⓘ
```

### 2.2 12 trường điều kiện có sẵn

| value | Tên hiển thị | Ánh xạ RFM | Dùng cho |
|---|---|---|---|
| `TotalRevenue` | Tổng bán (trừ trả hàng) | **M** (net) | Monetary segment |
| `TotalInvoiced` | Tổng bán | **M** (gross) | Monetary segment (gross) |
| `RewardPoint` | Điểm hiện tại | — | Loyalty tier |
| `TotalPoint` | Tổng điểm tích lũy | — | Loyalty lifetime value |
| `PurchaseDate` | Thời gian giao dịch cuối | **R** | Recency segment |
| `PurchaseNumber` | Số lần mua hàng | **F** | Frequency segment |
| `Debt` | Công nợ hiện tại | — | Credit risk |
| `BirthDay` | Tháng sinh | — | Birthday targeting |
| `Age` | Tuổi | — | Demographic |
| `Gender` | Giới tính | — | Demographic |
| `Location` | Khu vực | — | Geo segment |
| `Type` | Loại khách (Cá nhân/Công ty) | — | B2B vs B2C |

**Nhận xét:** 3 trường RFM (`PurchaseDate` = R, `PurchaseNumber` = F, `TotalRevenue` = M) đủ để build segmentation RFM cơ bản trong KiotViet.

### 2.3 Operators (suy luận từ UI)

Mỗi điều kiện hỗ trợ operator: `>`, `<`, `>=`, `<=`, `=`, `≠`, `in range`, `is`, `contains` (tùy theo kiểu trường).

### 2.4 3 chế độ áp dụng rule

| Chế độ | Hành vi | Use case |
|---|---|---|
| **Thêm KH theo điều kiện** | Chạy 1 lần: add KH thỏa rule. KH mới về sau KHÔNG auto-add trừ khi bật "tự động" | Campaign one-time (nhóm KH mua nhiều tháng này) |
| **Cập nhật lại danh sách theo điều kiện** | Re-evaluate toàn bộ: kéo KH ra khỏi nhóm nếu không còn thỏa rule | **Dynamic segment** — nhóm luôn live theo dữ liệu mới nhất |
| **Không cập nhật danh sách KH** | Tag thuần thủ công — manual add/remove | Nhóm tĩnh do NV quyết định |

### 2.5 Tự động hóa

Checkbox **"Hệ thống thực hiện tự động"**: Khi bật, rule re-run theo schedule (đoán là daily hoặc trigger sau mỗi giao dịch). Đây là **dynamic segment** thực sự — segment luôn cập nhật khi dữ liệu KH thay đổi.

---

## 3. Build RFM với rule engine hiện tại

RFM (Recency × Frequency × Monetary) là chuẩn vàng phân khúc KH retail. Với 12 điều kiện hiện có, có thể dựng 5 nhóm RFM cơ bản:

### 3.1 Mapping điều kiện → RFM segment

```
Nhóm 1: "VIP / Champion"
  Điều kiện: TotalRevenue > 5,000,000
          AND PurchaseNumber > 5
          AND PurchaseDate < 30 ngày (giao dịch trong 30 ngày gần nhất)
  Chế độ: Cập nhật lại danh sách + Tự động
  Benefit: Discount 5% (tự động)

Nhóm 2: "Loyal"
  Điều kiện: PurchaseNumber > 3
          AND PurchaseDate < 60 ngày
  Chế độ: Cập nhật lại + Tự động

Nhóm 3: "At Risk" (trước đây mua nhiều, gần đây ngừng)
  Điều kiện: TotalRevenue > 2,000,000
          AND PurchaseDate > 90 ngày (lần cuối mua cách đây hơn 90 ngày)
  Chế độ: Cập nhật lại + Tự động
  → Trigger campaign re-engage

Nhóm 4: "New Customer"
  Điều kiện: PurchaseNumber = 1
          AND PurchaseDate < 30 ngày
  Chế độ: Cập nhật lại + Tự động
  → Nurture flow: cảm ơn + upsell

Nhóm 5: "Debt Risk"
  Điều kiện: Debt > 1,000,000
  Chế độ: Cập nhật lại + Tự động
  → Auto reminder ZNS
```

### 3.2 Hạn chế hiện tại khi build RFM

| Hạn chế | Impact | Workaround hiện có |
|---|---|---|
| Chỉ AND giữa các điều kiện — không có OR | Không thể diễn đạt "KH mua nhiều HOẶC tích nhiều điểm" | Tạo 2 nhóm riêng |
| `PurchaseDate` không rõ là "trong N ngày gần nhất" hay "cách đây hơn N ngày" | Recency rule không chính xác | Cần verify thêm qua test |
| `PurchaseNumber` là tổng toàn thời gian — không có time window | Không thể build "mua ≥ 3 lần trong 90 ngày" | Không có workaround |
| Không có preview số KH thỏa điều kiện trước khi save | Không biết segment có bao nhiêu KH | Phải save và đếm sau |
| Tần suất re-run "tự động" không cấu hình được | Không biết segment lag bao lâu so với realtime | — |

---

## 4. So sánh với chuẩn ngành

| Tiêu chí | KiotViet hiện tại | HubSpot / Mailchimp | Mục tiêu đề xuất |
|---|---|---|---|
| Số điều kiện | 12 | 50+ | 20–30 |
| Logic | AND only | AND / OR / NOT / nested | AND / OR / group |
| Time window | Không | Có ("in last 90 days") | Có |
| Preview live | Không | Có (realtime count) | Có |
| RFM preset template | Không | Không (tự build) | **Có — 1-click** |
| Dynamic segment | Có (opt-in schedule) | Có (realtime) | Có (near-realtime) |
| Exclude rules | Không | Có | Có |
| Segment overlap view | Không | Có | Tùy |

→ KiotViet ở khoảng **60–65% so với chuẩn ngành** cho rule engine. Đủ cho SMB 500–2,000 KH, chưa đủ cho chuỗi có 10k+ KH cần phân tích hành vi phức tạp.

---

## 5. Điểm đau & cơ hội cải tiến

### 5.1 Rule engine

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Chỉ AND giữa các điều kiện | Boolean tree builder với AND/OR/NOT + group nesting |
| 2 | Không có time window cho condition | Thêm time window: "trong 30 ngày qua", "kể từ ngày X" |
| 3 | Không có preview live | "Match X customers" count — cập nhật khi user thay đổi điều kiện |
| 4 | Tần suất re-run "tự động" không cấu hình | Schedule config (hourly/daily/weekly) + timestamp last-run |
| 5 | Không có RFM preset template | 1-click apply model với Champion/Loyal/At Risk/Hibernating/Lost |
| 6 | Không có segment overlap view | Venn diagram: KH thuộc nhiều nhóm chiếm bao nhiêu % |
| 7 | Không có exclude rule | "KH trong nhóm VIP NHƯNG không có công nợ" |
| 8 | Không rõ priority khi 1 KH thuộc nhiều nhóm có discount khác nhau | Cần document behavior: take max / stack / first-match |

### 5.2 Segmentation nâng cao

| # | Pain | Đề xuất |
|---|---|---|
| 9 | Không có **Customer Lifetime Value** prediction | ML model LTV từ cohort + purchase velocity |
| 10 | Không có **churn risk score** per KH | Score 0–100 realtime, threshold → alert + campaign |
| 11 | Không có **product affinity** trong điều kiện | "Đã mua sản phẩm nhóm X" → gợi ý cross-sell |
| 12 | Không có **geo-based segment** chi tiết | Segment theo Tỉnh/TP + radius từ CN |
| 13 | Không có **behavioral trigger** (vào nhóm khi KH thực hiện hành động cụ thể) | Event-based trigger: "thêm vào nhóm khi mua SP X lần 2" |

### 5.3 Loyalty tier engine

| # | Pain | Đề xuất |
|---|---|---|
| 14 | CustomerTier (C07) có data nhưng không có rule auto-upgrade/downgrade | Rule: "Tổng bán > 5tr/quý → lên Gold, < 1tr/quý → xuống Silver" |
| 15 | Tất cả KH hạng Gold/Silver nhận benefit giống nhau | Tier benefit engine: mỗi hạng có set discount, delivery fee, service riêng |
| 16 | Không có **expiry** cho điểm tích | Flag điểm hết hạn + notification 30 ngày trước |
| 17 | Không có **referral tracking** | Mã giới thiệu per KH + commission history |

---

## 6. Cơ hội đột phá — Top 3 cho Segmentation

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Advanced Rule Engine** — OR/NOT/nested + time window + live preview + RFM preset | Pain #1-5 — phần lớn shop 1k+ KH cần phân khúc tốt hơn | Cao | Rất cao |
| 2 | **AI Churn Prediction + Auto Re-engagement** — score per KH + trigger campaign khi score vượt ngưỡng | Pain #10 — biến "ngày GD cuối" thành proactive alert | Cao | Rất cao |
| 3 | **Loyalty Tier Engine** — rule auto-upgrade/downgrade + benefit per tier + expiry | Pain #14-16 — chuỗi cần differentiate VIP treatment | Trung | Cao |

---

## 7. Tóm lược

**Nhóm KH + Rule Engine KiotViet:**
- **Có rule engine thực sự** — 12 điều kiện, 3 mode (one-shot/dynamic/manual), auto-execute theo schedule
- **Đủ build RFM cơ bản** với `PurchaseDate` (R), `PurchaseNumber` (F), `TotalRevenue` (M)
- **Giảm giá tự động per nhóm** — cơ chế pricing đơn giản, hiệu quả cho seller B2B
- **Điểm yếu:** Chỉ AND, không có time window, không có preview live, không có RFM template — dừng ở mức 60-65% so với chuẩn ngành
- **Cơ hội lớn nhất:** Nâng rule engine lên boolean tree + time window + live preview + RFM 1-click preset — có thể unlock segment hoàn chỉnh cho 10k+ KH mà không cần CRM bên ngoài
