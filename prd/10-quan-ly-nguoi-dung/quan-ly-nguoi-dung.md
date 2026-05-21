# PHÂN TÍCH QUẢN LÝ NGƯỜI DÙNG & PHÂN QUYỀN — KIOTVIET

**Phạm vi:** Module **Thiết lập → Quản lý người dùng** trong KiotViet. Tập trung mô tả nghiệp vụ user-management + permission system. Đây là **module bảo mật cốt lõi** quyết định ai làm được gì trong toàn hệ thống.

**Vị trí UI:** `/man/#/Settings?SettingType=user-manager` (menu Settings → Cửa hàng → Quản lý người dùng)
**Test data:** 47 user thực tế, 3 vai trò built-in (Quản trị chi nhánh / Nhân viên kho / Nhân viên thu ngân)

---

## 1. KIẾN TRÚC NGHIỆP VỤ

Module gồm **2 tab chính** triển khai mô hình **RBAC (Role-Based Access Control)**:

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   Tab 1: Tài khoản người dùng    ──┐                        │
│   ─ Identity: tên, email, phone    │                        │
│   ─ Mật khẩu                       │                        │
│   ─ Chi nhánh truy cập             ├─► gán Vai trò         │
│   ─ Trạng thái Đang hoạt động      │                        │
│                                    │                        │
│   Tab 2: Quản lý vai trò ◄─────────┘                        │
│   ─ Tên vai trò (Quản trị / NV kho / NV thu ngân / Custom)  │
│   ─ Mô tả                                                   │
│   ─ Permission tree (10 module × N permissions)              │
│                                                              │
└─────────────────────────────────────────────────────────────┘

→ User N : N Vai trò (multi-role assignment)
→ Vai trò N : N Permission (granular permission per resource)
```

→ KiotViet hỗ trợ **multi-role assignment**: 1 user có thể gán nhiều vai trò (vd user "Đức" có "2 vai trò. Xem"), permissions = UNION các vai trò.

---

## 2. TAB "TÀI KHOẢN NGƯỜI DÙNG"

### 2.1 List view

**Cột list:**
| Cột | Mô tả |
|---|---|
| ☐ Checkbox | Bulk select |
| **Tên hiển thị** | Tên hiển thị trong UI (kèm badge **(Tôi)** cho user hiện tại) |
| **Tên đăng nhập** | Username dùng để login |
| **Điện thoại** | Phone đăng ký |
| **Vai trò** | Role được gán; nếu nhiều → hiển thị "N vai trò. **Xem**" |
| **Trạng thái** | Đang hoạt động / Ngưng hoạt động |

### 2.2 Sample data thực (47 user testzone17)

| Tên hiển thị | Tên đăng nhập | Vai trò |
|---|---|---|
| Admin **(Tôi)** | shiptest | (root admin, không gán vai trò) |
| Dung 2 | Dung2 | **Nhân viên kho** |
| Dung1 | Dung1 | **Quản trị chi nhánh** |
| giangtest3 | giangtest3 | Quản trị chi nhánh |
| nv1 | nv1 | Quản trị chi nhánh |
| ngan | ngantest08 | Quản trị chi nhánh |
| **Đức** | duc | **2 vai trò. Xem** ⭐ multi-role |
| quynhit2 | quynhit2 | Quản trị chi nhánh |

→ **User Admin** đặc biệt — không gán vai trò vì có quyền tuyệt đối (root permissions không cần qua role).

### 2.3 Top actions

| Button | Function |
|---|---|
| **Lọc** | Filter user theo trạng thái, vai trò, chi nhánh |
| **Tạo tài khoản** (primary) | Thêm user mới |
| **Cài đặt cột** (⚙) | Hiện/ẩn cột |

### 2.4 Sidebar hint

Hệ thống proactive gợi ý:
> "Thiết lập quyền mặc định theo vai trò để phân quyền người dùng nhanh chóng"
> Link: "Tìm hiểu cách thiết lập tài khoản người dùng và vai trò"

→ KiotViet nudge admin **dùng role pattern** thay vì set quyền per-user (best practice).

---

## 3. TAB "QUẢN LÝ VAI TRÒ"

### 3.1 3 vai trò built-in

| Vai trò | Số user gán | Use case |
|---|---|---|
| **Quản trị chi nhánh** | 55 tài khoản | Trưởng cửa hàng — quyền gần như full trong CN, không có cấp tenant |
| **Nhân viên kho** | 6 tài khoản | NV thủ kho — chỉ thao tác Hàng hóa + Mua hàng + Chuyển hàng + Kiểm kho |
| **Nhân viên thu ngân** | 23 tài khoản | NV bán hàng tại quầy — Bán hàng + Hóa đơn + Khách hàng (giới hạn) |

→ Tổng cộng **55 + 6 + 23 = 84 assignment** trên 47 user → trung bình **1.79 vai trò/user**, xác nhận multi-role.

### 3.2 Cột list vai trò

| Cột | Mô tả |
|---|---|
| Vai trò | Tên |
| Mô tả | (optional) |
| **Tài khoản** | "N tài khoản. **Xem**" — clickable drill-down |
| **Thao tác** | ✏️ Sửa permissions \| 📋 Clone (tạo role mới từ role này) |

### 3.3 Action chính

| Action | Use case |
|---|---|
| **+ Tạo vai trò** | Tạo role custom (vd "Trưởng ca", "Kế toán nội bộ") |
| ✏️ **Sửa** | Cập nhật permission tree |
| 📋 **Clone** | Copy role → chỉnh nhẹ → save tên mới. Use case: "NV thu ngân ca tối" copy từ "NV thu ngân" rồi thêm quyền duyệt hủy đơn |
| **Xóa vai trò** | Trong modal sửa — chỉ cho role custom, không xóa được built-in |

---

## 4. PERMISSION TREE — KIẾN TRÚC CHI TIẾT

### 4.1 10 module top-level

Modal "Sửa vai trò" có **anchor menu** điều hướng nhanh, mỗi mục là 1 module phân quyền:

| # | Module | Use case nghiệp vụ |
|---|---|---|
| 1 | **Tổng quan** | Xem dashboard tổng |
| 2 | **Hàng hóa** | Sản phẩm + Kho hàng |
| 3 | **Mua hàng** | NCC + Nhập + Trả nhập + HĐ đầu vào |
| 4 | **Đơn hàng** | Hóa đơn + Đặt hàng + Trả hàng + Vận đơn |
| 5 | **Khách hàng** | KH + Nhóm + Công nợ + Tích điểm |
| 6 | **Sổ quỹ** | Phiếu thu chi |
| 7 | **Bán online** | Đa kênh TMĐT |
| 8 | **Phân tích & Báo cáo** | Reports + Analytics |
| 9 | **Nhân viên** | Quản lý NV nội bộ |
| 10 | **Thiết lập** | Cấu hình hệ thống |

### 4.2 Cấu trúc phân cấp 3 tầng

```
Module (top — anchor)
   │
   ├── Sub-section (group resource)
   │      │
   │      ├── Resource (1 màn hình/đối tượng cụ thể)
   │      │      │
   │      │      ├── Action: Xem
   │      │      ├── Action: Tạo
   │      │      ├── Action: Chỉnh sửa
   │      │      ├── Action: Xóa
   │      │      └── Khác:
   │      │             ├── Import
   │      │             ├── Xuất file
   │      │             ├── ⭐ Xem và sửa giá vốn
   │      │             └── ⭐ Xem và sửa giá nhập
```

### 4.3 Ví dụ: module "Hàng hóa"

**2 sub-section:**

#### Sub-section "Hàng hóa" (Hàng hóa, Thiết lập giá, Bảo hành & Bảo trì)
- ☑ Danh sách hàng hóa ▼
- ☑ Thiết lập giá ▼

#### Sub-section "Kho hàng" (Chuyển hàng, Sản xuất, Kiểm kho, Xuất hủy, Xuất dùng nội bộ)
- ☑ Chuyển hàng ▼
- ☑ Sản xuất ▼
- ☑ Kiểm kho ▼
- ☑ Xuất dùng nội bộ ▼
- ☑ Xuất hủy ▼

#### Khi expand "Danh sách hàng hóa" — 8 permission:

| Nhóm | Permission |
|---|---|
| **CRUD chuẩn** | ☑ Xem - Hàng hóa |
| | ☑ Tạo - Hàng hóa |
| | ☑ Chỉnh sửa - Hàng hóa |
| | ☑ Xóa - Hàng hóa |
| **Khác** | ☑ Import |
| | ☑ Xuất file |
| | ⭐ ☑ **Xem và sửa giá vốn** |
| | ⭐ ☑ **Xem và sửa giá nhập** |

→ Tách bạch quyền nhạy cảm:
- **Giá vốn** = WAC, biên lợi nhuận → chỉ chủ shop / kế toán nội bộ
- **Giá nhập** = giá thỏa thuận với NCC → bí mật thương mại, tránh NV bán hàng lộ ra
- Import / Xuất file → tách riêng vì có thể bulk gây hậu quả lớn

### 4.4 Sub-section "Mua hàng" (quan sát từ scroll)

- Nhà cung cấp
- Công nợ nhà cung cấp
- Đặt hàng nhập
- Nhập hàng
- Trả hàng nhập

→ Pattern lặp lại: mỗi resource có CRUD + Import + Export + có thể có quyền nhạy cảm riêng.

### 4.5 Tính năng tìm kiếm phân quyền

Top modal có hint: **"Ctrl+F để tìm phân quyền"**

→ Vì cấu trúc có thể có **hàng trăm permission** (10 module × N resource × 8 actions), search là cần thiết. Admin gõ "giá vốn" → highlight tất cả permission liên quan để bật/tắt nhanh.

---

## 5. WORKFLOW NGHIỆP VỤ THƯỜNG GẶP

### 5.1 Onboard NV mới

1. Settings → Quản lý người dùng → **+ Tạo tài khoản**
2. Nhập identity: Tên hiển thị, Tên đăng nhập, mật khẩu khởi tạo, SĐT, email
3. Gán **Chi nhánh** mà user được truy cập (limit theo CN)
4. Gán **1 hoặc nhiều Vai trò** từ danh sách có sẵn
5. Active = Đang hoạt động
6. Gửi credentials cho NV qua kênh secure

### 5.2 Tạo role custom

1. Tab "Quản lý vai trò" → **+ Tạo vai trò**
2. Nhập tên (vd "Trưởng ca", "Kế toán cộng tác viên")
3. **Tích check permission tree** — chọn module/resource/action
4. Lưu

### 5.3 Clone role (workflow phổ biến)

1. Vai trò gốc → ✏️/📋 **Clone**
2. Hệ thống copy full permission tree
3. Đổi tên + chỉnh nhẹ
4. Lưu

→ Use case: shop có 5 trưởng ca với quyền hơi khác nhau → clone từ role base "Trưởng ca" rồi customize per ca.

### 5.4 User multi-role

User "Đức" có "2 vai trò" — vd vừa là **Nhân viên kho** (sáng) vừa **Nhân viên thu ngân** (chiều) → cần truy cập cả 2 module. Permission = UNION (KH nào ngon nhất).

→ Pattern này KHÁC với nhiều phần mềm POS competitor (chỉ cho 1 role/user).

### 5.5 Off-board NV nghỉ việc

1. Tab Tài khoản → tìm user → ✏️ Sửa
2. Đổi trạng thái → **Ngưng hoạt động**
3. Không xóa (giữ audit log cho lịch sử thao tác)
4. Reassign các phiếu/HĐ NV đó tạo (nếu cần)

### 5.6 Đổi mật khẩu / Reset

User self-service hoặc admin reset:
- Admin gắn mật khẩu mới khi tạo
- User vào tài khoản tự đổi
- Nếu quên → admin reset (tabAdmin → Sửa → đổi mật khẩu)

---

## 6. CÁC ĐẶC ĐIỂM ĐẶC BIỆT

### 6.1 Phân quyền theo chi nhánh (per-branch scoping)

Mỗi user gán cụ thể **N chi nhánh** được truy cập. Vai trò nói "được làm gì", chi nhánh nói "ở đâu".

→ Vd: "Quản trị chi nhánh" + scope = "CN Quận 1" → user thấy + edit data CN1 thôi, không thấy CN2/CN3.

### 6.2 Phân quyền theo Nhóm hàng (advanced)

Theo help KiotViet, có menu phụ "Phân quyền người dùng theo nhóm hàng" — cho phép user chỉ thao tác được trên 1 nhóm hàng cụ thể (vd NV mỹ phẩm chỉ xem được nhóm "Mỹ phẩm", không xem "Thực phẩm").

→ **Phân quyền 3 chiều**: User × Chi nhánh × Nhóm hàng.

### 6.3 Quyền nhạy cảm tách riêng

KiotViet tách bạch quyền liên quan đến thông tin nhạy cảm:
- **Giá vốn** — hiện biên lợi nhuận
- **Giá nhập** — bí mật với NCC
- **Công nợ** — số tiền KH nợ
- **Xóa** — irreversible action

→ Default mở Xem nhưng tắt các quyền nhạy cảm cho NV cấp thấp.

### 6.4 Audit Log (Lịch sử thao tác)

Trong Settings sidebar có mục **"Lịch sử thao tác"** — ghi nhận mọi hành động của user (tạo HĐ, sửa giá, hủy phiếu, đăng nhập bất thường...).

→ Đây là **layer thứ 2** sau permission — kể cả khi user có quyền, mọi hành động vẫn được audit log.

### 6.5 Bảo mật (Settings sidebar)

Mục **"Bảo mật"** riêng trong Settings — config:
- IP whitelist (chỉ login từ IP công ty)
- 2FA / OTP
- Session timeout
- Phát hiện đăng nhập bất thường (banner cảnh báo trên dashboard khi có 17 hoạt động đăng nhập khác thường — đã thấy ở dashboard)

---

## 7. ĐIỂM MẠNH NGHIỆP VỤ

1. **RBAC chuẩn** — User → Role → Permission, không cần set quyền per-user
2. **Multi-role assignment** — 1 user N role, permissions = UNION → flexible cho NV đa nhiệm
3. **Permission tree 3 tầng** (Module → Sub-section → Resource → Actions) — granular nhưng có hierarchy dễ navigate
4. **Anchor menu 10 module** với Ctrl+F search → admin tìm quyền nhanh trong hàng trăm permission
5. **3 role built-in chuẩn ngành** (Quản trị / NV kho / NV thu ngân) — out-of-box dùng được ngay
6. **Clone role** — không cần build từ đầu, copy + chỉnh
7. **8 permission per resource** chuẩn (CRUD + Import + Xuất file + 2 nhạy cảm) — rich enough cho enterprise
8. **Quyền nhạy cảm tách riêng** — Giá vốn, Giá nhập có permission riêng → bảo vệ thông tin chiến lược
9. **Per-branch scoping** — user thấy data CN nào tùy cấu hình
10. **Per-product-group scoping** (advanced) — phân quyền 3 chiều cực kỳ chi tiết
11. **Multi-tenant** — Admin (Tôi) badge phân biệt user hiện tại
12. **Trạng thái Đang/Ngưng** — off-board mà không mất audit history
13. **Lịch sử thao tác (Audit log)** — layer thứ 2 sau permission
14. **Bảo mật riêng** — IP whitelist, 2FA, session, anomaly detection
15. **Sidebar hint** gợi ý best practice "dùng vai trò thay vì per-user"
16. **Drill-down "N tài khoản. Xem"** — từ Role → list user gán role đó, dễ audit

---

## 8. ĐIỂM ĐAU NGHIỆP VỤ

### 8.1 Onboard & Lifecycle

| # | Pain | Đề xuất |
|---|---|---|
| 1 | Không có **bulk create user** từ Excel | Import file (đã có ở các module khác — apply ở đây) |
| 2 | Không có **invite via email** — admin phải set pass cứng và gửi tay | Magic link / OTP invite |
| 3 | Không có **expire account** tự động (NV thời vụ làm 3 tháng) | Auto-expire date |
| 4 | Không thấy **bulk reassign** khi NV nghỉ (chuyển portfolio sang NV khác) | Bulk reassign wizard |

### 8.2 Role Design

| # | Pain | Đề xuất |
|---|---|---|
| 5 | Permission tree có hàng trăm checkbox — admin **dễ miss / nhầm** | Permission matrix view (Role × Resource × Action grid) |
| 6 | Không có **role template library** (Trưởng ca / KAM thu mua / Kế toán nội bộ / Marketer) ngoài 3 built-in | Marketplace role templates |
| 7 | Không thấy **diff between roles** — khó so sánh "NV thu ngân" khác "NV thu ngân ca tối" chỗ nào | Compare 2 roles side-by-side |
| 8 | Không có **role versioning** — sửa role ảnh hưởng ngay tất cả user gán role đó | Version + draft mode + apply later |
| 9 | Không có **conditional permission** (vd: NV chỉ được hủy HĐ của chính mình tạo, không hủy của người khác) | ABAC (Attribute-Based) bổ sung |

### 8.3 Compliance & Audit

| # | Pain | Đề xuất |
|---|---|---|
| 10 | Lịch sử thao tác đặt riêng module — không inline trên user detail | Audit log tab inline trên user |
| 11 | Không có **separation of duties** alert (vd cùng 1 user vừa tạo PN vừa duyệt → conflict) | SoD rule + alert |
| 12 | Không thấy **session management** — không revoke session đang active từ admin | Active session list + force logout |
| 13 | Không thấy **approval workflow** trên các action nhạy cảm (vd xóa data, sửa giá vốn) | 4-eye approval cho action critical |

### 8.4 UX

| # | Pain | Đề xuất |
|---|---|---|
| 14 | "Ctrl+F để tìm phân quyền" — chỉ search browser, không phải search trong modal | In-modal search box |
| 15 | Không có **visual diff** khi sửa role — admin không thấy permission nào đã đổi | Diff highlight |
| 16 | User detail không có **trang riêng** — click row chỉ thấy summary | Full user profile page |
| 17 | Không thấy **last login time** + **last action time** per user trên list | Add cột Last seen |
| 18 | "2 vai trò. Xem" — Xem là link nhưng tooltip không hiện vai trò gì | Hover preview |

### 8.5 Multi-tenant / Enterprise

| # | Pain | Đề xuất |
|---|---|---|
| 19 | Không có **SSO integration** (Google Workspace / Microsoft AD) cho chuỗi lớn | SAML / OIDC support |
| 20 | Không có **SCIM provisioning** | SCIM endpoint |
| 21 | Không thấy **organizational hierarchy** (CEO → CTO → Manager → Staff) | Org tree + reports-to |

---

## 9. CƠ HỘI ĐỘT PHÁ — TOP 5

| # | Tính năng | Pain giải quyết | Effort | Impact |
|---|---|---|---|---|
| 1 | **Role Template Marketplace** + **Diff between roles** + **Versioning + Draft mode** | Pain #5, #6, #7, #8 | Cao | Rất cao |
| 2 | **Conditional Permission (ABAC)** + **Separation of Duties alert** | Pain #9, #11 — compliance lớn cho chuỗi | Cao | Cao |
| 3 | **Magic Link Invite** + **Auto-expire** + **Bulk reassign** | Pain #1, #2, #3, #4 | Trung | Cao |
| 4 | **SSO + SCIM** (Google / MS AD) | Pain #19, #20 | Cao | Rất cao (enterprise) |
| 5 | **Active Session Manager** + **4-eye Approval** cho action nhạy cảm | Pain #12, #13 | Trung | Cao (security) |

---

## 10. SO SÁNH VỚI ĐỐI THỦ & CHUẨN INDUSTRY

| Khía cạnh | KiotViet | Sapo | Misa | Shopify | Industry standard |
|---|---|---|---|---|---|
| RBAC model | ✅ User → Role → Permission | ✅ | Partial | ✅ | ✅ |
| Multi-role per user | ✅ ⭐ | ❌ (1 role) | ❌ | Partial | Best practice |
| Permission per-resource granularity | ✅ 8 actions | 5-6 | 4 | 6-8 | 8+ |
| Sensitive field separation (giá vốn) | ✅ ⭐ | ❌ | Partial | ❌ | Best practice |
| Per-branch scoping | ✅ | ✅ | ✅ | ❌ (location ok) | Must-have |
| Per-product-group scoping | ✅ ⭐ | ❌ | ❌ | ❌ | Advanced |
| Role clone | ✅ | ❌ | ❌ | ✅ | Standard |
| Search permissions (Ctrl+F) | Partial (browser) | ❌ | ❌ | ✅ | Standard |
| Audit log | ✅ riêng module | ✅ | Partial | ✅ | Must-have |
| 2FA | ✅ | ✅ | ✅ | ✅ | Must-have |
| SSO | ❌ | ❌ | Partial | ✅ | Enterprise |
| Conditional permission (ABAC) | ❌ | ❌ | ❌ | Partial | Advanced |
| Separation of duties | ❌ | ❌ | ❌ | ❌ | Compliance |

→ KiotViet **dẫn đầu thị trường VN** về: multi-role, sensitive field separation, per-product-group scoping. Còn dư địa: SSO, ABAC, SoD.

---

## 11. TÓM LƯỢC

**Quản lý người dùng & Phân quyền KiotViet:**

- Triển khai **RBAC chuẩn** với 2 tab tách bạch: Tài khoản người dùng và Quản lý vai trò
- **47 user** thực tế (test), **3 vai trò built-in** (Quản trị chi nhánh / Nhân viên kho / Nhân viên thu ngân) — out-of-box dùng được ngay
- **Multi-role assignment** ⭐ — 1 user có thể gán N vai trò, permissions = UNION (đặc biệt mạnh so với competitor VN)
- **Permission tree 3 tầng** (Module → Sub-section → Resource → 8 actions) với **10 module top-level**, anchor menu + Ctrl+F search
- **8 permission chuẩn per resource**: Xem / Tạo / Chỉnh sửa / Xóa + Import / Xuất file + **Xem-sửa giá vốn** + **Xem-sửa giá nhập** (2 quyền nhạy cảm tách riêng — moat lớn cho compliance)
- **Phân quyền 3 chiều**: User × Chi nhánh × Nhóm hàng — granular chưa từng thấy ở phần mềm POS VN khác
- **Mạnh:** RBAC chuẩn, multi-role, permission nhạy cảm tách riêng, per-branch + per-product-group scoping, clone role, audit log riêng, bảo mật riêng module, badge "(Tôi)" cho user hiện tại
- **Yếu:** Không có role template marketplace, không có diff/versioning, không có ABAC/SoD, không có magic link invite, không có SSO/SCIM, không có in-modal search (chỉ Ctrl+F browser), không có active session manager
- **Cơ hội đột phá:** Role Template + Diff + Versioning, Conditional Permission (ABAC) + SoD, Magic Link Invite + Auto-expire + Bulk reassign, SSO + SCIM (Google/MS AD), Active Session + 4-eye Approval

Đây là module **mạnh nhất thị trường VN về granularity và multi-role, còn nhiều dư địa cho enterprise compliance (SSO, ABAC, SoD) và UX improvements (visual diff, in-modal search) — match đúng segment chuỗi và doanh nghiệp vừa đang là moat lớn của KiotViet**.
