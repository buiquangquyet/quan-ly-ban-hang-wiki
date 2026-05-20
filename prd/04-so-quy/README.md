# Module 04 — Sổ quỹ

Phân tích module Sổ quỹ của KiotViet: operational cash ledger ghi nhận mọi dòng tiền vào/ra shop, phân biệt với accounting ledger (module Thuế & Kế toán) và inventory ledger (Thẻ kho).

**Test data:** 46 phiếu thu chi | Tổng thu 9,045,753đ | Tổng chi −729,095đ | Tồn quỹ 8,316,658đ

---

## Cấu trúc tài liệu

| File | Nội dung |
|---|---|
| [01-tong-quan.md](./01-tong-quan.md) | Vai trò trong hệ sinh thái, entity schema, mã chứng từ, loại thu chi, state machine, UI/UX, đối chiếu kế toán, pain points & cơ hội |

---

## Entity Catalog (SQ01–SQ08)

| # | Entity | Tên VN | Trạng thái | Tham chiếu |
|---|---|---|---|---|
| SQ01 | CashVoucher | Phiếu thu / Phiếu chi | Có | `01-tong-quan.md` §2 |
| SQ02 | FundType | Loại quỹ (Tiền mặt / NH / Ví / Tổng) | Có | `01-tong-quan.md` §2.1 |
| SQ03 | FundAccount | Tài khoản quỹ (TK NH cụ thể, ví cụ thể) | Có | `01-tong-quan.md` §2.1 |
| SQ04 | CashTransactionType | Loại thu chi (enum cấu hình được) | Có | `01-tong-quan.md` §4 |
| SQ05 | AccountingFlag | Hạch toán KQKD (bool tách cash flow vs P&L) | Có | `01-tong-quan.md` §4.3 |
| SQ06 | CashTransfer | Chuyển/Rút quỹ (CNH — cặp đối ứng) | Có | `01-tong-quan.md` §3 |
| SQ07 | CashForecast | Dự báo dòng tiền 7/14/30 ngày | **Chưa có — đề xuất** | `01-tong-quan.md` §7 |
| SQ08 | ReconciliationRecord | Đối chiếu sao kê ngân hàng | **Chưa có — đề xuất** | `01-tong-quan.md` §7 |

---

## Sổ quỹ là điểm hội tụ dòng tiền

```
[Bán hàng HD]      ──KH trả tiền──►  [Phiếu thu TTHD — SQ01]  ─► Sổ quỹ → Customer.balance ↓
[Đặt hàng DH]      ──KH cọc────────► [Phiếu thu TTDH — SQ01]  ─► Sổ quỹ
[Trả hàng TH]      ──refund────────► [Phiếu chi TTTH — SQ01]  ─► Sổ quỹ → Customer.balance ↑
[Nhập hàng PN]     ──trả NCC───────► [Phiếu chi      — SQ01]  ─► Sổ quỹ → Supplier.balance ↓
[Trả hàng nhập TPN]──NCC hoàn tiền► [Phiếu thu      — SQ01]  ─► Sổ quỹ → Supplier.balance ↑
[Chi phí khác]     ──thủ công──────► [Phiếu thu/chi  — SQ01]  ─► Sổ quỹ
[Chuyển quỹ]       ──internal──────► [CNH cặp đối ứng — SQ06] ─► Sổ quỹ (±0 net)
```

Mọi tiền vào/ra shop đều bắt buộc ghi qua đây → single source of truth cho cash position.
