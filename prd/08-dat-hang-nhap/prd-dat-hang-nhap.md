# PRD — DAT HANG NHAP (PURCHASE ORDER)

**Pham vi:** Mo ta thuan nghiep vu cua module Dat hang nhap (Purchase Order). Khong di vao schema/code.

**Vi tri:** Menu top-bar **Mua hang → Dat hang nhap**
**Ma chung tu:** Tien to **DHN** (Dat Hang Nhap)

---

## 1. DAT HANG NHAP LA GI

**Dat hang nhap** (Purchase Order — PO) la **phieu cam ket mua** tu shop gui toi Nha cung cap (NCC). Day la kham trung gian giua "co nhu cau nhap" va "hang da ve kho", giup shop:

- **Khoa gia nhap** voi NCC tai thoi diem dat (chong bien dong gia)
- **Len ke hoach dong tien** (biet truoc bao nhieu tien se chi cho NCC)
- **Len ke hoach kho** (biet truoc hang ve khi nao, chuan bi cho)
- **Phoi hop voi NCC** (NCC biet shop se lay gi, bao nhieu — chuan bi truoc)
- **Audit trail** (moi cam ket voi NCC deu co chung tu)

> Dat hang nhap la **hanh dong cam ket**. Nhap hang la **hanh dong ghi nhan hang thuc ve**.

### Phan biet voi module Dat hang (DH) da co

| Khia canh | Dat hang (DH) | Dat hang nhap (DHN) |
|---|---|---|
| Doi tac | Khach hang | Nha cung cap |
| Chieu hang | Se xuat kho (khi tao HD) | Se nhap kho (khi NCC giao) |
| Chieu tien | Se thu | Se chi |
| Phieu ket thuc workflow | Hoa don (HD) | Phieu Nhap hang (PN) |
| Co shipping integration | Co (den KH) | Khong (NCC tu ship den shop) |

Pattern giong nhau (phieu cam ket, partial fulfillment, co the huy), nhung doi tuong va dong tien nguoc chieu.

---

## 2. VAI TRO TRONG HE SINH THAI

```
[Shop nhan thay sap het hang]
      |
      v
[Dat hang nhap DHN######]
      |
      +-- Gui NCC (xac nhan chot gia + ngay giao)
      |
      |   Tac dong so sach: "Dat NCC" tren product detail tang
      |   (KHONG tac dong ton kho thuc)
      |
      v
[NCC giao hang toi shop]
      |
      v
[Phieu Nhap hang PN######]
      |
      +-- Ton kho thuc tang
      +-- Cong no phai tra NCC tang
      +-- Gia von cap nhat WAC
      |
      v
DHN cap nhat SL con lai
  (nhap du -> DHN "Hoan thanh")
  (chua du -> "Nhap mot phan", cho PN tiep theo)
```

DHN lien thong voi 5 doi tuong:
- **NCC (Supplier)** — nguon hang, doi tac cam ket
- **San pham** — cap nhat "Dat NCC" tren product detail panel
- **Phieu Nhap (PN)** — phieu ket thuc workflow
- **Ton kho** — chi qua PN (DHN khong tac dong truc tiep)
- **So quy** — khong truc tiep, nhung giup forecast cash out

---

## 3. CAC TRANG THAI WORKFLOW

| # | Trang thai | Y nghia nghiep vu | Hanh dong ke tiep |
|---|---|---|---|
| 1 | **Phieu tam** | Moi tao, dang du thao, chua gui NCC | Sua, gui NCC, huy |
| 2 | **Da xac nhan NCC** | NCC da chot gia + ngay giao | Cho NCC giao, co the tao PN tu day |
| 3 | **Nhap mot phan** | Da nhap 1 phan SL qua PN, con du chua nhap du | Tiep tuc tao PN nhap so con lai, hoac dong phieu |
| 4 | **Hoan thanh** | Da nhap du qua PN | Dong phieu |
| (5) | **Da huy** | Huy bo phieu | Immutable, van luu trong list |

### Workflow chuan (happy path)

```
Tao → Phieu tam → (gui NCC, NCC OK) → Da xac nhan NCC
   → (NCC giao hang dot 1) → Nhap mot phan
   → (NCC giao hang dot 2 du) → Hoan thanh
```

### Workflow rut gon

```
Tao → Phieu tam → (NCC giao luon 1 lan du) → Hoan thanh
```

(Neu shop tin NCC giao dung, co the skip "Da xac nhan NCC")

### Workflow huy

```
Bat ky trang thai nao (tru Hoan thanh) → [Huy] → Da huy
   He thong hoi: huy luon cac PN lien quan hay giu?
```

---

## 4. CAC TRUONG NGHIEP VU QUAN TRONG

### 4.1 Tren danh sach

| Cot | Vai tro nghiep vu |
|---|---|
| Ma dat hang nhap | DHN###### — doi chieu khi NCC giao hang / ke toan |
| Thoi gian | Ngay tao DHN |
| **Nha cung cap** | Doi tac cam ket |
| **Ngay nhap du kien** | Ngay NCC hua giao hang → key cho **ke hoach kho + cash flow** |
| **So ngay cho** | Tinh tu ngay tao → ngay nhap du kien (hoac den hien tai neu chua nhap). Canh bao NCC cham |
| **VAT nhap hang** | Muc thue suat ap dung (0%/5%/8%/10%) — chuan HDDT dau vao |
| **Can tra NCC** | Tong gia tri shop se phai thanh toan NCC |

> Module duy nhat co cot **"So ngay cho"** — KiotViet proactive canh bao NCC cham.

### 4.2 Thong tin tren phieu

**Header:**
- Ma DHN###### (tu sinh, sua duoc)
- Nha cung cap
- **Nguoi nhan dat** — NV phu trach phieu (KAM thu mua)
- **Ngay dat** + **Ngay nhap du kien** (key cho forecast)
- Chi nhanh nhap
- VAT (muc thue)
- Trang thai

**Dong hang (Lines):**
- San pham + bien the
- Don vi tinh + quy doi
- **So luong dat** (cam ket) — theo doi so voi **So luong da nhap** thuc te (track qua PN)
- Don gia nhap (da thoa thuan voi NCC)
- Giam gia (NCC uu dai neu co)
- Thanh tien
- VAT per line

**Tong:**
- Tong tien hang
- Giam gia phieu
- VAT tong
- Tong cong (can tra NCC)

### 4.3 Thong tin can nhap khi tao phieu

Cac thong tin bat buoc/quan trong khi tao mot DHN moi:

- Tim/chon NCC (co the tao NCC moi truc tiep trong form neu chua co trong he thong)
- Them san pham: tim theo ma/ten, quet ma vach, hoac **import tu file Excel** (co file mau)
- Co the them san pham moi vao he thong ngay trong form (neu SP chua ton tai)
- Nhap so luong dat, don gia nhap, giam gia per line
- Cai dat **VAT nhap hang** (bat/tat + nhap muc)
- Nhap **chi phi nhap hang** (tra NCC, phan bo vao gia von)
- Nhap **chi phi nhap khac** (tra ben thu 3, phan bo gia von nhung khong cong vao no NCC)
- Cai dat **Ngay nhap du kien**
- Ghi chu
- Tuy chon gui email NCC ngay khi tao phieu (xem §5.6)

**Phim tat ho tro:**
- **F3** — focus o tim hang hoa
- **F8** — focus o "Tien tra NCC" de nhap thanh toan

### 4.4 Phan biet 2 loai chi phi

| Chi phi | Tra cho | Cong vao no NCC? | Phan bo gia von? |
|---|---|---|---|
| **Chi phi nhap hang** | NCC (vd phi giao hang NCC charge) | Co | Co |
| **Chi phi nhap khac** | Ben thu 3 (vd thue xe ngoai, boc xep) | Khong | Co |

> Ca 2 deu day gia von len (landed cost), nhung chi "Chi phi nhap hang" tinh vao so phai tra NCC.

### 4.5 Cai dat ca nhan hoa (per user)

Moi nguoi dung co the cau hinh rieng:

**Hien thi tren bang lines:**
- Anh hang hoa (bat/tat)
- Thuong hieu (bat/tat)
- Ton Kho hien tai (bat/tat — xem ton trong khi dat tranh dat thua)
- Thu tu hien thi dong hang

**Hanh vi nhap lieu:**
- Them dong gia khac cho cung 1 SP (vd khuyen mai NCC "mua 10 tang 1")
- Che do loc inline
- Chon nhieu hang hoa cung luc
- **Gia nhap la gia von** — quyet dinh co dung don gia nhap tren phieu nay lam gia von moi (WAC update) hay giu gia von cu (xem §5.10)
- Giam gia: mac dinh VND hay %

### 4.6 Xuat du lieu

- In phieu — in de gui NCC giay
- Xuat file Excel: **File tong quan** (list cac DHN voi header) hoac **File chi tiet** (boc tach tung line hang)

Use case: gui NCC de doi soat, gui ke toan de hach toan.

---

## 5. CAC NGHIEP VU NANG CAO

### 5.1 Nhap mot phan (Partial Fulfillment)

Day la **diem manh** cua DHN. Use case:
- Shop dat 100 thung nuoc. NCC giao dot 1 = 60 thung, dot 2 = 40 thung
- DHN → tao PN001 ghi nhan 60 thung → DHN chuyen "Nhap mot phan"
- Tuan sau tao PN002 ghi nhan 40 thung → DHN chuyen "Hoan thanh"

He thong tu tinh SL con lai cua DHN = SL dat − Tong SL da nhap qua PN.

### 5.2 Nhap tu nhieu DHN trong 1 PN

Nguoc lai: 1 phieu PN co the nhap gop tu **nhieu DHN** cung NCC. Use case: NCC giao 1 chuyen cho hang cua 3 don DHN khac nhau → shop tao 1 PN gop.

### 5.3 Chi phi nhap tra NCC

Gan chi phi (van chuyen, boc do, bao hiem...) vao DHN. Khi tao PN tu DHN, chi phi nay phan bo vao gia von tung dong (xem `02-hang-hoa-kho/prd-nhap-hang.md` § 2.2).

### 5.4 Nguoi nhan dat (Buyer)

Truong nay gan DHN voi 1 NV cu the cua shop (vd: nhan vien thu mua / ke toan mua hang). Filter theo NV → KAM thu mua xem portfolio minh quan ly.

### 5.5 Import tu file Excel

Shop co the import danh sach hang hoa vao phieu DHN tu file Excel (co file mau). Use case: NCC gui catalog co san, shop download → dien so luong + gia → import thang vao phieu.

### 5.6 Gui email NCC truc tiep

Khi tao phieu, shop co the chon **"Dat va gui email"** de:
1. Tao phieu DHN
2. Render template email voi noi dung phieu
3. Gui den email cua NCC (lay tu master NCC)

> Tiet kiem 5-10 phut moi don so voi workflow "Tao → Xuat Excel → Mo Gmail → Dinh kem → Soan email → Gui".

### 5.7 Tao NCC moi trong form

Khi tao DHN, neu NCC chua ton tai trong he thong, shop co the tao NCC moi ngay trong form ma khong can thoat ra menu quan ly NCC.

Use case: lan dau nhap tu NCC moi.

### 5.8 Them dong gia khac cho cung 1 SP

Khi bat setting "Them dong", cho phep add nhieu dong cho cung 1 SP voi gia khac nhau. Use case:
- NCC "mua 10 tang 1" → 1 dong gia that, 1 dong gia 0
- Mua theo lo co gia tier (10 SP dau gia 100k, 10 SP sau gia 90k) → 2 dong

### 5.9 Sao chep phieu cu

Tao DHN moi bang cach sao chep tu phieu cu. Use case: dat hang dinh ky cung NCC, cung SP.

### 5.10 Quy tac cap nhat gia von (WAC)

Setting **"Gia nhap la gia von"** — cai dat **per-phieu** (khong phai global):

| Tuy chon | Y nghia |
|---|---|
| **Bat (default)** | Khi DHN nay duoc nhap qua PN, don gia nhap tren line → cap nhat vao WAC (Binh quan gia quyen) cua SP |
| **Tat** | Gia von giu nguyen cost cu, khong update tu phieu nay |

Use case tat: NCC giao gia thap dot bien (sale, khuyen mai 1 lan) — khong muon keo gia von xuong lam meo bao cao lai.

---

## 6. SO SANH 4 PHIEU LIEN QUAN CHUOI MUA HANG

| Khia canh | DHN | PN | TPN | HD dau vao |
|---|---|---|---|---|
| Vai tro | Cam ket mua | Ghi nhan hang ve | Tra hang cho NCC | HDDT thue nhap |
| Ma prefix | DHN | PN | TPN | (theo CQT) |
| Tac dong ton | Khong (chi "Dat NCC") | Tang | Giam | Khong |
| Tac dong cong no NCC | Khong | Tang | Giam | Khong (chi chung tu thue) |
| Tac dong gia von (WAC) | Khong | Cap nhat | Khong (da co dinh) | Khong |
| Bat buoc tham chieu phieu truoc | — | Tuy chon (co the tao doc lap) | **Bat buoc PN goc** | Tuy chon |
| Co "Nhap mot phan" | Co | — | — | — |
| Co "So ngay cho" canh bao | Co | — | — | — |

---

## 7. DIEM MANH NGHIEP VU HIEN TAI

1. **Workflow day du 4 trang thai** ro rang — khong co "ghost state" gay confused
2. **Cot "So ngay cho" doc nhat** — proactive canh bao NCC cham (it POS khac co)
3. **Cot "Ngay nhap du kien"** — gan vao ke hoach kho + cash flow
4. **Partial fulfillment thuc su** — 1 DHN co the duoc nhap qua nhieu PN, system tu track SL con lai
5. **Lien thong 2 chieu** voi PN — nhap tu DHN hoac gop nhieu DHN
6. **VAT per line** — chuan HDDT dau vao (TT78/2021)
7. **Phan quyen "Nguoi nhan dat"** — KAM thu mua co portfolio rieng
8. **Tach bach ton vat ly vs cam ket** — DHN khong tru ton, khong gay nhieu bao cao
9. **"Dat va gui email" 1-click** — gui NCC ngay khi tao phieu, tiet kiem 5-10 phut/don
10. **Tao NCC moi inline** trong form — khong can thoat menu master
11. **Phim tat F3 (search) + F8 (thanh toan)** — phu hop NV thu mua thao tac nhanh
12. **Tach 2 loai chi phi**: "Chi phi nhap hang" (tra NCC, cong vao no) vs "Chi phi nhap khac" (tra ben thu 3, chi phan bo gia von) — landed cost dung nghiep vu
13. **Setting "Gia nhap la gia von" per-phieu** — quyet dinh co update WAC tu phieu nay hay khong, control gia von tinh te
14. **Cai dat ca nhan hoa** — bat/tat cot Anh / Thuong hieu / Ton kho va hanh vi nhap lieu (Them dong gia, Chon nhieu SP, Filter inline)

---

## 8. DIEM DAU NGHIEP VU

### 8.1 Workflow & Lifecycle

| # | Pain | De xuat |
|---|---|---|
| 1 | Khong co **"Dang cho NCC xac nhan"** giua Phieu tam va Da xac nhan NCC — shop khong biet NCC da tra loi chua | Them state "Da gui NCC, cho phan hoi" |
| 2 | Khong co **deadline tu dong dong DHN** khi NCC khong giao (vd 30 ngay sau "Ngay nhap du kien") | Auto-close rule + alert |
| 3 | Canh bao "So ngay cho" hien thi nhung khong co **alert proactive** (email/notif) khi vuot nguong | Threshold setting + push alert |
| 4 | Khong co **ly do huy** chuan hoa (NCC het hang / shop doi y / gia tang / khac) | Dropdown reason → analytics |

### 8.2 Lien thong NCC

| # | Pain | De xuat |
|---|---|---|
| 5 | "Gui NCC" van manual (email/zalo/sdt) — khong co **portal NCC** de NCC xem + confirm online | NCC self-service portal |
| 6 | Khong co **EDI light** — NCC tu update "da giao N dot" len DHN cua shop | Vendor sync API |
| 7 | Khong co **NCC scorecard auto** (on-time delivery %, fill-rate %, defect %) tu data DHN/PN/TPN | Vendor performance dashboard |
| 8 | Khong co **gia tham chieu** tu lich su DHN truoc voi cung NCC (goi y "gia lan truoc = X, lan nay NCC bao Y co hop ly?") | Price benchmark inline |

### 8.3 Forecast & Planning

| # | Pain | De xuat |
|---|---|---|
| 9 | Khong co **Smart Reorder Engine** — goi y "SP A sap het, NCC X cho ngay Y, dat N cai?" dua tren velocity + lead time | AI reorder suggestion |
| 10 | Khong co **PO suggestion tu dinh muc ton** — phai tay quet ton roi tao DHN | Auto-generate DHN draft khi ton xuong dinh muc |
| 11 | Khong co **batch creation** — dat 50 SP tu 5 NCC khac nhau phai tao 5 phieu thu cong | Bulk wizard: chon SP → group theo NCC → auto-tao nhieu DHN |
| 12 | "Ngay nhap du kien" khong feed vao **cash flow forecast** tren So quy | Integrated forecast |

### 8.4 3-Way Match Compliance

| # | Pain | De xuat |
|---|---|---|
| 13 | Khong co **3-way match** (DHN ↔ PN ↔ Hoa don dau vao) tu dong | Auto-match + alert discrepancy |
| 14 | Phieu PN co the tao doc lap khong link DHN — bypass control | Setting "bat buoc PO-first" cho enterprise |
| 15 | Khong co **threshold approval** (DHN > N trieu can manager duyet) | Workflow approval theo nguong |
| 16 | Khong co **change order** chinh thuc — sua DHN truc tiep khi NCC da confirm la dirty audit | Change order voi version + ai duyet |

### 8.5 Khac

| # | Pain | De xuat |
|---|---|---|
| 17 | Khong co **template DHN** — dat dinh ky hang thang cung SP phai tao lai | Save as template + 1-click duplicate |
| 18 | Khong co **OCR phieu bao gia NCC** → auto tao DHN | OCR import |
| 19 | Khong thay app mobile ho tro tao DHN nhanh (NV thu mua ngoai shop) | Mobile shortcut |

---

## 9. CO HOI DOT PHA — TOP 5 NGHIEP VU

| # | Tinh nang | Pain giai quyet | Effort | Impact |
|---|---|---|---|---|
| 1 | **Smart Reorder AI** — goi y DHN tu dong dua tren velocity + lead time + dinh muc ton | Pain #9, #10 | Cao | Rat cao |
| 2 | **3-way Match Auto + Vendor Scorecard** | Pain #7, #13 | Cao | Cao (enterprise) |
| 3 | **NCC Self-service Portal** — NCC tu confirm + update giao hang | Pain #5, #6 | Cao | Rat cao (giam 50% cong NV thu mua) |
| 4 | **PO Approval Workflow** (theo nguong gia tri + change order versioning) | Pain #15, #16 | Trung | Cao (chuoi) |
| 5 | **Cash Flow Integration** — "Ngay nhap du kien" + "Can tra NCC" feed vao So quy forecast | Pain #12 | Trung | Cao |

---

## 10. TOM LUOC

**Dat hang nhap KiotViet (nghiep vu):**
- La **cau noi cam ket** giua shop va NCC — khong tac dong ton vat ly, chi cap nhat "Dat NCC" tren product
- 4 trang thai workflow ro rang: Phieu tam → Da xac nhan NCC → Nhap mot phan → Hoan thanh (+ Da huy)
- **Manh:** Partial fulfillment thuc su, cot "So ngay cho" canh bao NCC cham, "Ngay nhap du kien" cho forecast, VAT per line chuan HDDT, tach bach "Nguoi nhan dat" cho KAM thu mua, "Dat va gui email" 1-click, tao NCC moi inline, 2 phim tat F3/F8, tach 2 loai chi phi (NCC vs ben thu 3), setting "Gia nhap la gia von" per-phieu, cai dat ca nhan hoa
- **Yeu:** Khong co Smart Reorder AI, khong co NCC self-service portal, khong co 3-way match auto, khong co vendor scorecard, khong co cash flow integration voi So quy, khong co approval workflow theo nguong, khong co OCR phieu bao gia
- **Co hoi dot pha:** Smart Reorder AI, 3-way Match + Vendor Scorecard, NCC Self-service Portal, PO Approval Workflow, Cash Flow Integration

Module **da co nen tang nghiep vu chac cho SMB nhung con rat nhieu du dia cho intelligence (AI reorder, vendor scoring) va collaboration (NCC portal) — match voi segment chuoi va shop co volume mua hang lon**.
