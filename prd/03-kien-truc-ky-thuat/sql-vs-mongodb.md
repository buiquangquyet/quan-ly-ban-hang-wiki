# SQL vs MONGODB — QUYẾT ĐỊNH KIẾN TRÚC

**Quyết định:** Mọi nghiệp vụ chính dùng **SQL (Postgres)**. MongoDB chỉ dùng cho **log & audit log**.

---

## 1. NGUYÊN TẮC

### SQL (Postgres) — Tất cả nghiệp vụ
- ACID transactions bắt buộc (bán hàng → giảm tồn → ghi sổ — all-or-nothing)
- Financial accuracy — DECIMAL chính xác
- Relational integrity mạnh (FK giữa product, invoice, customer)
- Complex aggregations — báo cáo, JOIN nhiều bảng
- Strong consistency — không tolerate eventual

### MongoDB — Chỉ log & audit log
- Schema linh hoạt — mỗi loại event có payload khác nhau
- Write-heavy, read-recent — không cần query phức tạp
- TTL built-in — tự xóa log cũ không cần cron job
- Không ảnh hưởng đến tính toàn vẹn dữ liệu nghiệp vụ nếu mất 1 dòng

---

## 2. PHÂN LOẠI MODULE

### Postgres — Toàn bộ nghiệp vụ chính

| Module | Bảng chính |
|---|---|
| Tenancy & Access | merchant, user, role, permission |
| Org Structure | branch, pickup_address |
| Product Catalog | product, product_image, product_unit, product_variant, category, brand, supplier |
| Inventory | stock, stock_norm, batch, batch_stock, stock_unit |
| Pricing | price_book, price_book_item, tax_rate |
| Partners | customer, customer_address |
| Sales Transactions | invoice, invoice_line, payment, return_invoice, preorder, e_invoice |
| Inventory Transactions | purchase_order, stock_transfer, stock_take, manufacturing, internal_use, write_off, **stock_card_line** |
| Delivery | shipment, delivery_partner, delivery_service |
| Promotion & Loyalty | promotion, voucher, coupon, loyalty_points |
| Administrative | province, ward |
| Operations | shift |

**stock_card_line là general ledger — tuyệt đối không được mất 1 dòng → Postgres.**

### MongoDB — Chỉ log & audit log

| Collection | Vai trò | TTL |
|---|---|---|
| `activity_log` | Hành vi user: login, sửa giá, đổi setting, xóa SP | 1 năm |
| `security_log` | Đăng nhập bất thường, IP lạ, brute-force | 90 ngày |
| `api_request_log` | Request/response từ marketplace (Shopee/TikTok/Lazada raw payload) | 90 ngày |
| `notification_log` | Lịch sử push notification, Zalo/SMS gửi đi | 90 ngày |
| `webhook_event_log` | Webhook inbound từ hãng vận chuyển, marketplace | 90 ngày |

```
mongo.activity_log {
  _id: ObjectId,
  merchant_id: 1,
  user_id: 12,
  action: "PRICE_CHANGE | PRODUCT_DELETE | SETTING_TOGGLE | ...",
  entity_type: "PRODUCT | CUSTOMER | BRANCH | ...",
  entity_id: 444,
  changes: { before: {...}, after: {...} },
  ip_address: "...",
  user_agent: "...",
  at: ISODate   // TTL index trên field này
}
```

---

## 3. CÁC STORAGE KHÁC

| Storage | Dùng cho |
|---|---|
| **Redis** | Session, cache (stock TTL 30s, price_book), rate limit, pub/sub realtime |
| **S3 / MinIO** | Ảnh sản phẩm, file HĐĐT XML đã ký số, PDF hóa đơn in |
| **Elasticsearch** | Full-text search sản phẩm (5,500+ SP), search khách hàng, lọc đơn hàng phức tạp |

---

## 4. KIẾN TRÚC TỔNG QUAN

```
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────┐    ┌──────────────────────────┐  │
│  │   POSTGRES (Primary) │    │   MONGODB                │  │
│  │                      │    │                          │  │
│  │  Toàn bộ nghiệp vụ:  │    │  Chỉ log:                │  │
│  │  • Merchant, User    │    │  • activity_log          │  │
│  │  • Product           │    │  • security_log          │  │
│  │  • Customer          │    │  • api_request_log       │  │
│  │  • Invoice + Lines   │    │  • notification_log      │  │
│  │  • Payment           │    │  • webhook_event_log     │  │
│  │  • Stock + StockCard │    │                          │  │
│  │  • Batch + Serial    │    │  TTL tự xóa cũ           │  │
│  │  • Inv vouchers      │    │  Schema flexible per     │  │
│  │  • Delivery          │    │  event type              │  │
│  │  • Promotion         │    │                          │  │
│  │  • Loyalty           │    │                          │  │
│  │                      │    │                          │  │
│  │  Source of truth     │    │  Append-only, TTL        │  │
│  └──────────────────────┘    └──────────────────────────┘  │
│                                                             │
│  ┌──────────────────────┐    ┌──────────────────────────┐  │
│  │   REDIS              │    │   S3 / MinIO             │  │
│  │  • Session / Auth    │    │  • Ảnh sản phẩm          │  │
│  │  • Cache stock 30s   │    │  • HĐĐT XML signed       │  │
│  │  • Cache price_book  │    │  • PDF hóa đơn in        │  │
│  │  • Rate limit        │    │                          │  │
│  └──────────────────────┘    └──────────────────────────┘  │
│                                                             │
│  ┌──────────────────────┐                                  │
│  │   ELASTICSEARCH      │                                  │
│  │  • Product search    │                                  │
│  │  • Customer search   │                                  │
│  │  • Order filter      │                                  │
│  └──────────────────────┘                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. LƯU Ý TRIỂN KHAI

- Log ghi vào MongoDB **sau khi** Postgres commit xong — không ảnh hưởng đến transaction chính
- Nếu MongoDB down, nghiệp vụ vẫn chạy bình thường — log có thể ghi buffer vào Redis queue rồi flush sau
- **Không** dùng MongoDB cho bất kỳ dữ liệu nào cần JOIN với bảng Postgres (tránh distributed query)
- `stock_card_line` là audit trail kế toán — **khác** với `activity_log` là audit hành vi user. Cái trước ở Postgres, cái sau ở MongoDB
