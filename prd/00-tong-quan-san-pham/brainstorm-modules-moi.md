# KIOTVIET — BRAINSTORM SẢN PHẨM/MODULE MỚI

**Ngày:** 18/05/2026
**Target segment:** Chuỗi SMB 3–20 chi nhánh + Người bán online đa kênh (TikTok Shop, Shopee, Lazada)
**Phương pháp:** Khám phá testzone17 → Map current state → Identify whitespace → Brainstorm → Prioritize (Impact × Effort)

---

## 1. CURRENT STATE — Những gì KiotViet ĐÃ CÓ (Quan sát từ testzone17)

### Core POS & vận hành
- Tổng quan dashboard với doanh thu theo ngày/giờ/thứ, top hàng bán chạy, top khách hàng
- Hàng hóa: Danh sách, thiết lập giá, kho hàng, **Chuyển hàng giữa chi nhánh**, Kiểm kho, **Sản xuất**, Xuất dùng nội bộ, Xuất hủy
- Mua hàng, Đơn hàng, Khách hàng, Nhân viên, Sổ quỹ, Thuế & Kế toán

### Multi-channel online (đã tích hợp 9 gian hàng test)
- **Marketplace:** Shopee (5 gian), TikTok Shop, Lazada, Tiki
- **Social:** Facebook, Instagram, Zalo OA, **Shopee Chat**
- **Chat đa kênh** (gắn nhãn "Mới")
- **Livestream + Bài viết + Catalog** trên mạng xã hội
- **Marketing auto-đẩy hàng hóa** lên gian hàng
- Đối soát đơn TMĐT, Báo cáo theo kênh bán hàng

### Tài chính & dịch vụ giá trị gia tăng
- Thanh toán QR code (cài đặt miễn phí)
- Tích hợp **Vay vốn** ("Đủ vốn hôm nay, vững vàng ngày mai")
- Tích hợp vận chuyển (Viettel Post và các hãng vận chuyển khác)
- Phát hiện **hoạt động đăng nhập bất thường** (security alerting)

### Mở rộng địa lý
- Đa ngôn ngữ: Việt, Anh, **Khmer (Campuchia), Nga, Nhật, Trung** → Có dấu hiệu mở rộng SEA/quốc tế

### Quan sát điểm yếu (gap để brainstorm)
- Phân quyền & tổ chức nhân sự cho chuỗi vẫn theo mô hình phẳng (1 cấp chi nhánh, không thấy region/area manager)
- Không thấy module workforce/payroll/chấm công nhân viên cho chuỗi
- Marketing chỉ ở mức "auto-đẩy hàng" — chưa có CRM/loyalty/automation campaign sâu
- Báo cáo dạng bảng truyền thống, chưa có "ask in natural language" hay AI insight
- Chưa có module riêng cho seller TikTok Shop livestream (host script, scheduling, replay analytics)
- Chưa có B2B/wholesale mode (bảng giá theo khách, công nợ tín dụng, đặt hàng cho NCC từ chuỗi xuống)
- Chưa có "Operations command center" cho chủ chuỗi xem KPI realtime cross-branch

---

## 2. BRAINSTORM — 24 Ý TƯỞNG MODULE/SẢN PHẨM MỚI

Phân thành 7 themes. Mỗi ý tưởng có: **Problem → Solution → Why KiotViet phải làm**.

### Theme A — AI Operations (Vận hành thông minh)

**A1. KiotViet Copilot — Trợ lý AI hội thoại**
- Problem: Chủ chuỗi mất 30–60p/ngày đọc báo cáo, không biết hỏi gì khi data nằm rải rác.
- Solution: Chat box "Hỏi như hỏi nhân viên kế toán" — "Doanh thu CN Cầu Giấy tuần này so cùng kỳ", "Top 5 SKU sắp hết hàng", "Khách hàng nào im lặng 30 ngày?". Trả lời bằng số + chart + drill-down.
- Why KiotViet: Khác biệt rõ với Sapo/Misa, gắn LLM lên data sẵn có, retention 10x.

**A2. AI Demand Forecasting — Dự báo bán hàng theo SKU × chi nhánh**
- Problem: Chuỗi 5–20 CN thường ôm tồn sai chỗ, OOS chỗ này thừa chỗ kia. Chủ shop phải đoán bằng cảm tính.
- Solution: Forecast 7/14/30 ngày tới cho từng SKU × CN, kết hợp lịch sử + seasonality + sự kiện local. Đề xuất điều chuyển hàng tự động.
- Why KiotViet: Data internal đã có sẵn, đây là "must-have" của chuỗi vừa, hiện chưa ai làm tử tế ở VN cho SMB.

**A3. AI Pricing Assistant — Tối ưu giá theo kênh**
- Problem: Cùng 1 SKU bán giá khác nhau Shopee/TikTok/POS, không biết để đâu cho ngon. Sale dynamic theo đối thủ phải làm tay.
- Solution: Crawl giá đối thủ trên marketplace + phân tích elasticity giá → đề xuất giá tối ưu mỗi tuần cho mỗi kênh.
- Why KiotViet: Tận dụng sẵn data đa kênh, giải pain rất rõ của TikTok seller.

**A4. AI Product Photo & Listing Studio**
- Problem: Seller TikTok/Shopee tốn 100k–300k/SKU thuê chụp + viết title + bullet point. Listing không chuẩn → giảm hiển thị.
- Solution: Upload 1 ảnh → AI tạo nhiều phối cảnh + title SEO + 5 bullet + caption livestream. Push lên cả 4 sàn.
- Why KiotViet: Trở thành "publisher" thực sự, gắn deeper với hàng hóa.

---

### Theme B — Multi-channel & Marketplace Mastery (Bán online sâu hơn)

**B1. TikTok Live Studio — Khoang điều khiển cho livestream seller**
- Problem: Seller TikTok Shop chạy 4–8 tiếng live/ngày, hiện phải dùng OBS + Google Sheet + máy POS riêng để chốt đơn. Replay không phân tích được.
- Solution: Module riêng quản lý kịch bản live, hiển thị stock realtime trên overlay, tự động đẩy mã giảm giá theo phút, replay analytics (SKU nào lên hot lúc nào, viewer retention, đơn/giờ).
- Why KiotViet: Livestream e-commerce là sóng lớn nhất 2025–2027 ở VN, hiện chỉ TPos/UpBase đang làm nhưng không sâu vận hành như KiotViet có thể.

**B2. Multi-warehouse Smart Routing cho đơn online**
- Problem: Đơn từ Shopee về nhưng KiotViet đẩy sai kho/sai chi nhánh → ship chậm, phí cao.
- Solution: Auto chọn kho gần khách nhất + còn hàng + chi phí ship thấp nhất. Hiển thị "lý do chọn" để chủ shop tin tưởng.
- Why KiotViet: Tận dụng multi-branch của KiotViet, biến nó thành lợi thế cho seller online.

**B3. Unified Inbox 2.0 — One thread per customer cross-channel**
- Problem: 1 KH có thể nhắn Facebook, Zalo, Shopee chat, TikTok DM. Chat đa kênh hiện gom hộp thư nhưng không gom theo identity khách.
- Solution: Identity resolution (số đt + email + handle) → tất cả tin nhắn của 1 KH gộp về 1 thread + lịch sử mua hàng + ghi chú nhân viên.
- Why KiotViet: Step up từ "đa kênh" sang "đa kênh thông minh" — cản trở cạnh tranh từ tool chat chuyên (Sleekflow, Botbanhang).

**B4. TikTok Shop Affiliate Manager**
- Problem: Seller chạy affiliate trên TikTok Shop khó quản lý — affiliate xin sample, tỷ lệ commission, video upload, doanh thu theo từng KOC... đang quản lý bằng Sheet thủ công.
- Solution: Module tách riêng — duyệt affiliate, gửi mẫu, theo dõi video, attribution sale, payout tự động.
- Why KiotViet: TikTok Shop ưu tiên seller có affiliate strong → ai hỗ trợ seller làm tốt sẽ thắng.

**B5. Marketplace Compliance Bot**
- Problem: Shopee/TikTok cập nhật chính sách liên tục (cấm từ khóa, ảnh nhãn hiệu, giá so sánh...). Seller bị khóa shop oan.
- Solution: Quét tự động listing + chat content theo policy mới, cảnh báo trước khi vi phạm.
- Why KiotViet: Insurance feature — seller sợ bị khóa shop, sẵn sàng trả nhiều.

---

### Theme C — Workforce & Multi-branch Operations (Cho chuỗi)

**C1. KiotViet Workforce — Chấm công, ca làm, lương**
- Problem: Chuỗi 5–20 CN có 30–150 nhân viên — đang dùng MISA AMIS hoặc Excel, không liên kết với doanh số nhân viên trên KiotViet.
- Solution: Module chấm công GPS/wifi, xếp ca tự động dựa trên forecast đông khách, payroll tích hợp commission từ KiotViet, payslip điện tử.
- Why KiotViet: Mở rộng wallet share, nhân viên là pain # 1 của chủ chuỗi F&B/retail.

**C2. Area Manager Cockpit — Bảng điều khiển cho quản lý vùng**
- Problem: Khi chuỗi >5 CN thường có quản lý vùng nhưng KiotViet chỉ có "Chủ" và "Chi nhánh" — area manager phải xem từng CN.
- Solution: View riêng cho area manager — KPI dạng heatmap, ranking CN, alert chênh lệch bất thường, audit log nhân viên đáng nghi (gian lận tiền hàng).
- Why KiotViet: Direct path to enterprise/franchise tier — upsell pricing 3–5x.

**C3. Audit & Anomaly Detection — Chống thất thoát thông minh**
- Problem: Chuỗi mất hàng/tiền do thu ngân/QL chi nhánh "ăn" hóa đơn — chủ không phát hiện được.
- Solution: ML phát hiện pattern bất thường: hoàn hàng nhiều, void hóa đơn nhiều, gap kiểm kho lặp lại, voucher giả. Daily anomaly digest.
- Why KiotViet: Đã có sẵn "phát hiện đăng nhập bất thường" → mở rộng lên transaction-level.

**C4. Training & Onboarding Hub cho nhân viên mới**
- Problem: Tuyển thu ngân/sale mới phải train tay 3–7 ngày. Chuỗi tốn nhiều thời gian quản lý chi nhánh đào tạo.
- Solution: Mini LMS — video ngắn + quiz, mô phỏng POS, certification trước khi cấp quyền bán hàng. Có thư viện chuẩn KiotViet + custom theo shop.
- Why KiotViet: Sticky feature, giảm churn nhân viên (gián tiếp giảm churn của KiotViet).

---

### Theme D — Customer & Loyalty (CRM thật sự)

**D1. KiotViet Loyalty — App khách hàng riêng cho chuỗi**
- Problem: Khách của chuỗi F&B/thời trang không có cách nào quay lại đặt online/check điểm. Chủ shop phải dùng Mobifoodie/POS365 song song.
- Solution: White-label app cho khách (iOS/Android) — tích điểm, voucher, đặt hàng, đặt bàn, refer-a-friend. Brand hóa theo shop.
- Why KiotViet: Step lớn nhất để KiotViet thoát khỏi vai trò "POS đơn thuần" và trở thành commerce platform.

**D2. Marketing Automation 2.0 — Drip + win-back**
- Problem: Có module "thông tin khách hàng" nhưng không có flow tự động "khách 30 ngày không quay lại → gửi voucher Zalo + SMS".
- Solution: Workflow builder kéo thả, trigger từ event KiotViet (mua, không mua, sinh nhật, đạt hạng), kênh Zalo/SMS/Email/Push App.
- Why KiotViet: Khác biệt vs Misa CukCuk, kéo thêm Misa Loyalty users về.

**D3. Customer Data Platform (CDP) — Segment thông minh**
- Problem: Phân khách hàng hiện rất basic (theo tổng chi). Không biết khách nào nhạy giá, khách nào mua giờ vàng, khách VIP nào sắp churn.
- Solution: Auto-tag khách theo RFM, sản phẩm yêu thích, kênh ưa thích. Export segment để chạy ads Facebook/Zalo lookalike.
- Why KiotViet: Tiền đề cho mọi automation, cross-sell, upsell sau này.

---

### Theme E — Financial Services (Mở rộng wallet share)

**E1. KiotViet Pay — Ví thu tiền + đối soát thanh toán đa kênh**
- Problem: Đơn TikTok/Shopee/Lazada/COD về tiền nhiều ngày sau, đối soát tay rất mệt, không match được transaction ↔ đơn hàng.
- Solution: Ví trung gian thu hộ từ marketplace + COD shipper, auto-match đơn ↔ tiền về, hiển thị "tiền đang trên đường" theo timeline.
- Why KiotViet: Highest LTV — phí giao dịch + float, đối thủ là Payoo/MoMo for Business.

**E2. Tín dụng nhúng — Working capital cho seller**
- Problem: Seller TikTok 200tr/tháng vẫn khó vay vì ngân hàng đòi báo cáo tài chính. Cash flow âm do tiền về sau 7–15 ngày.
- Solution: Mở rộng "Vay vốn" sẵn có → score tự động từ doanh số KiotViet → advance 70–80% giá trị đơn TikTok/Shopee chưa về tiền, thu lãi theo ngày.
- Why KiotViet: Tận dụng data độc quyền → biên rất cao, kèm cho ngân hàng/finco.

**E3. Tax-ready e-invoice automation**
- Problem: HĐ điện tử bắt buộc với hộ kinh doanh từ 2026 — họ panic. Đối tượng SMB không hiểu nghiệp vụ thuế.
- Solution: Auto phát hóa đơn điện tử cho từng đơn POS/online + báo cáo thuế tháng/quý dạng "1-click submit". Tích hợp deeper với module Thuế & Kế toán đã có.
- Why KiotViet: Timing tuyệt đối hoàn hảo theo nghị định 2026 — first mover wins.

---

### Theme F — B2B & Wholesale (Bước vào segment mới)

**F1. KiotViet Wholesale — Mode bán sỉ cho shop bán cho shop**
- Problem: Nhiều seller chuỗi vừa đồng thời là nhà phân phối sỉ. Bán sỉ cần bảng giá theo khách, công nợ tín dụng, đặt hàng tối thiểu — không có trong KiotViet hiện tại.
- Solution: Toggle "wholesale mode" — multi-tier pricing, credit limit, đặt hàng portal cho B2B buyer, lịch sử công nợ.
- Why KiotViet: Expand TAM lên một bậc, mỗi tài khoản B2B tier giá cao hơn 3–5x retail.

**F2. Supplier Portal — Trục đặt hàng tới nhà cung cấp**
- Problem: Chủ chuỗi đặt hàng NCC bằng Zalo/điện thoại, không có audit, không có forecast tự động.
- Solution: Portal cho NCC nhận đơn từ KiotViet, xác nhận, gửi invoice. KiotViet auto đề xuất đơn nhập từ forecast (xem A2).
- Why KiotViet: Lock in supply chain — supplier cũng phải lên KiotViet.

---

### Theme G — Data Intelligence & Reporting (Vượt khỏi "báo cáo")

**G1. Competitive Benchmarking — So với chuỗi cùng ngành**
- Problem: Chủ shop F&B không biết mình "tốt" hay "tệ" so với mặt bằng.
- Solution: Anonymized benchmark: doanh số/m²/tháng, % comp food cost, conversion thực vs lý thuyết... so với median ngành cùng size cùng vùng. Opt-in chia sẻ data.
- Why KiotViet: Data network effect — chỉ KiotViet với 200k+ shop làm được.

**G2. Smart Replenishment Suggestions (kết hợp A2 + F2)**
- Problem: Chủ chuỗi mỗi sáng nhìn tồn kho → đoán đặt hàng. Vừa thiếu vừa thừa.
- Solution: Mỗi sáng đẩy "PO suggestions" trên home dashboard — auto duyệt 1 click hoặc adjust. Track accuracy theo thời gian.
- Why KiotViet: Daily-active driver, làm KiotViet trở thành workflow center thay vì chỗ tra báo cáo.

**G3. Mobile-first Owner App — KiotViet cho ông chủ**
- Problem: Web app dùng tốt trên desktop nhưng ông chủ chuỗi 80% xem qua điện thoại lúc đang di chuyển. App hiện chỉ cho thu ngân/sale.
- Solution: App native riêng cho owner — KPI cards, push alert (doanh thu CN < 70% kế hoạch), approve điều chuyển/giảm giá nhanh từ điện thoại.
- Why KiotViet: Up engagement của persona quyền lực nhất, giảm churn.

---

### Theme H — Đột phá (High-risk / High-reward bets)

**H1. KiotViet Marketplace cho seller TikTok dump hàng tồn**
- Problem: Chuỗi tồn hàng cuối mùa không biết bán cho ai, thường thanh lý lỗ. Seller TikTok cần SKU rẻ để live.
- Solution: Marketplace nội bộ KiotViet — chủ chuỗi list hàng tồn, seller TikTok mua sỉ về live.
- Why KiotViet: Tận dụng network 200k shop, asset không ai có.

**H2. KiotViet Academy — Học bán hàng đa kênh có cấp chứng chỉ**
- Problem: Chủ shop & nhân viên thiếu kiến thức livestream, ads, SEO marketplace. Không tin tưởng các course trên TikTok.
- Solution: Course online + community (Discord/Zalo group), cert hợp tác với TikTok Shop/Shopee Academy.
- Why KiotViet: Top-of-funnel marketing engine — kéo seller mới lên platform.

---

## 3. PRIORITY MATRIX

Chấm điểm theo (1–5):
- **Impact** = (revenue potential + customer pain) / 2 — Tính cả khả năng upsell tier giá cao
- **Effort** = (engineering complexity + GTM/data dependency + thời gian go-to-market)

| ID  | Ý tưởng                              | Impact | Effort | Score (I/E) | Quadrant      |
|-----|--------------------------------------|--------|--------|-------------|---------------|
| A1  | AI Copilot hội thoại                 | 5      | 3      | 1.67        | **Quick Win** |
| A2  | AI Demand Forecasting × CN           | 5      | 4      | 1.25        | **Big Bet**   |
| A3  | AI Pricing Assistant                 | 4      | 4      | 1.00        | Big Bet       |
| A4  | AI Photo & Listing Studio            | 4      | 3      | 1.33        | Quick Win     |
| B1  | TikTok Live Studio                   | 5      | 4      | 1.25        | **Big Bet**   |
| B2  | Multi-warehouse Smart Routing        | 4      | 3      | 1.33        | Quick Win     |
| B3  | Unified Inbox 2.0 (identity resolve) | 4      | 3      | 1.33        | Quick Win     |
| B4  | TikTok Affiliate Manager             | 4      | 3      | 1.33        | Quick Win     |
| B5  | Marketplace Compliance Bot           | 3      | 2      | 1.50        | Quick Win     |
| C1  | Workforce (chấm công, lương)         | 5      | 5      | 1.00        | Big Bet       |
| C2  | Area Manager Cockpit                 | 4      | 2      | 2.00        | **Quick Win** |
| C3  | Audit & Anomaly Detection            | 4      | 3      | 1.33        | Quick Win     |
| C4  | Training Hub                         | 3      | 3      | 1.00        | Maybe Later   |
| D1  | KiotViet Loyalty App (white-label)   | 5      | 5      | 1.00        | Big Bet       |
| D2  | Marketing Automation 2.0             | 4      | 3      | 1.33        | Quick Win     |
| D3  | CDP / RFM segmentation               | 4      | 2      | 2.00        | **Quick Win** |
| E1  | KiotViet Pay — đối soát đa kênh      | 5      | 4      | 1.25        | **Big Bet**   |
| E2  | Tín dụng nhúng (working capital)     | 5      | 4      | 1.25        | Big Bet       |
| E3  | Tax-ready e-invoice automation       | 5      | 2      | 2.50        | **#1 Quick Win**|
| F1  | Wholesale mode (B2B)                 | 4      | 4      | 1.00        | Big Bet       |
| F2  | Supplier Portal                      | 3      | 3      | 1.00        | Maybe Later   |
| G1  | Competitive Benchmarking             | 3      | 3      | 1.00        | Maybe Later   |
| G2  | Smart Replenishment (PO suggestions) | 4      | 3      | 1.33        | Quick Win     |
| G3  | Mobile Owner App                     | 4      | 2      | 2.00        | **Quick Win** |
| H1  | Marketplace nội bộ (dump hàng tồn)   | 3      | 5      | 0.60        | Drop / Later  |
| H2  | KiotViet Academy                     | 3      | 3      | 1.00        | Maybe Later   |

---

## 4. TOP 5 RECOMMENDATIONS

### #1. E3 — Tax-ready e-invoice automation (Quick Win, urgency 2026)
- **Why now:** Quy định HĐĐT bắt buộc cho hộ kinh doanh 2026 — KiotViet đã có Thuế & Kế toán, mở rộng thành "1-click submit" sẽ là phao cứu sinh cho 200k+ shop.
- **Effort:** Tích hợp deeper với TCT + cơ quan thuế. Reuse data đã có.
- **Mini-thesis:** Bán đính kèm tier giá Pro, conversion sẽ tăng vì FOMO compliance.

### #2. A1 — KiotViet Copilot (AI Conversational Assistant)
- **Why:** Khác biệt rõ nét với mọi đối thủ POS ở VN. Demo cực ấn tượng (sales-led growth).
- **Effort:** 3–5 tháng, cần text-to-SQL hardened cho ngôn ngữ Việt. Reuse data warehouse.
- **Mini-thesis:** Daily-active spike +30%, top-of-mind "POS có AI" cho mass market.

### #3. B1 — TikTok Live Studio
- **Why:** Sóng lớn nhất 2026–2027 ở VN. Hiện không có giải pháp deeply integrated với POS/kho. Win-the-decade bet.
- **Effort:** Cần đội build mới (livestream tech) — 6–9 tháng MVP.
- **Mini-thesis:** Capture được seller TikTok lớn → tier giá Enterprise + ecosystem revenue.

### #4. C2 — Area Manager Cockpit (Quick Win cho chuỗi)
- **Why:** Pain rất rõ của customer 5–20 CN. Effort thấp vì tận dụng query đã có.
- **Effort:** 2–3 tháng. Chủ yếu UX và phân quyền.
- **Mini-thesis:** Unlock pricing tier "Chain" 3–5x, low risk.

### #5. E1 — KiotViet Pay (Đối soát thanh toán đa kênh)
- **Why:** Pain # 1 của seller multi-channel. Mở cửa cho fintech revenue (phí giao dịch + float + lending feed dữ liệu).
- **Effort:** 6–12 tháng + giấy phép tài chính.
- **Mini-thesis:** Wallet share lớn nhất trong tất cả ý tưởng — long-term moat.

---

## 5. NHỮNG Ý TƯỞNG CẦN KIỂM CHỨNG TIẾP

Trước khi cam kết roadmap, đề xuất:
1. **Customer interview** 10 khách chuỗi 5–20 CN: pain # 1 hiện tại? Sẽ trả thêm bao nhiêu cho từng top 5?
2. **Customer interview** 10 seller TikTok Shop > 100tr/tháng: workflow live hiện tại?
3. **Competitive audit** Sapo Omnichannel, UpBase, TPos, Misa CukCuk Chain — features map.
4. **Data feasibility check** A1, A2: chất lượng tagging hàng hóa cross-merchant có đủ?

---

## 6. PROCESS DEBT — Gợi ý cho team Product

- KiotViet hiện chia module theo **chức năng** (Hàng hóa / Khách / Nhân viên). Nên test nhóm theo **persona** (Owner / Branch manager / Cashier / Online seller) để khám phá UX gaps.
- "Bán online" đang là 1 tab phụ — nếu segment online sellers tăng, nên xem xét **promote thành 1 product line riêng** với pricing độc lập.
- Thêm telemetry/event tracking để biết module nào đang ít dùng (giả thuyết: Sản xuất, Xuất hủy, Phân tích Hiệu quả có usage thấp).
