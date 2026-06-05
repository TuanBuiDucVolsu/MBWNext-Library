# Tài liệu đào tạo: Luồng Purchase Voucher (PV)

> **Phạm vi:** Module `mbwnext_advanced_accounting` — ERPNext/Frappe  
> **Cập nhật:** 2026-06-05  
> **Mục đích:** Đào tạo nhân viên hiểu toàn bộ luồng mua hàng từ Purchase Order → Purchase Voucher → Purchase Invoice → Purchase Receipt → Landed Cost Voucher

---

## Mục lục

1. [Tổng quan ba loại mua hàng](#1-tổng-quan-ba-loại-mua-hàng)
2. [Sơ đồ luồng tổng thể](#2-sơ-đồ-luồng-tổng-thể)
3. [Bước 1 — Purchase Order (PO)](#3-bước-1--purchase-order-po)
4. [Bước 2 — Purchase Voucher (PV)](#4-bước-2--purchase-voucher-pv)
5. [Bước 3 — Purchase Invoice (PI)](#5-bước-3--purchase-invoice-pi)
6. [Bước 4 — Purchase Receipt (PR)](#6-bước-4--purchase-receipt-pr)
7. [Bước 5 — Landed Cost Voucher (LCV)](#7-bước-5--landed-cost-voucher-lcv)
8. [Bước 6 — Payment Entry (PE)](#8-bước-6--payment-entry-pe)
9. [Tracking trạng thái xuyên suốt](#9-tracking-trạng-thái-xuyên-suốt)
10. [Kiểm soát tính hợp lệ](#10-kiểm-soát-tính-hợp-lệ)
11. [Tra cứu nhanh — Hooks đã đăng ký](#11-tra-cứu-nhanh--hooks-đã-đăng-ký)

---

## 1. Tổng quan ba loại mua hàng

Trường `purchase_invoice_type` trên Purchase Order và Purchase Voucher xác định toàn bộ luồng xử lý. Có ba loại chính:

| Loại | Bảng items | Có kho? | Có thuế NK? | Tạo PR? | Tạo LCV? |
|---|---|---|---|---|---|
| **Mua Dịch Vụ** | `service_pv_items` | Không | Không | Không | Không |
| **Mua Hàng Trong Nước Nhập Kho** | `items` | Có | Không | Có | Có (thủ công) |
| **Mua Hàng Nhập Khẩu Nhập Kho** | `items` + `taxes` | Có | Có | Có | Có (tự động) |

> **Lưu ý:** Mua Dịch Vụ thường tạo trước để làm nguồn chi phí (cước vận chuyển, phí hải quan...) cho các PV Hàng.

---

## 2. Sơ đồ luồng tổng thể

```
┌──────────────────────────────────────────────────────────────────────┐
│                    LUỒNG MUA HÀNG NHẬP KHẨU NHẬP KHO                │
│                      (luồng đầy đủ nhất)                             │
└──────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐   Tạo PV     ┌──────────────────────────────┐
  │ Purchase    │ ──────────── │     Purchase Voucher (PV)    │
  │ Order (PO)  │   nút "Create│  ┌─────────────────────────┐ │
  └─────────────┘   > Purchase │  │ items (hàng hóa)        │ │
         │          Voucher"   │  │ taxes (thuế NK)         │ │
         │                     │  │ pre_cost (CP trước CQ)  │ │
         │                     │  │ pv_purchase_cost (CP MH)│ │
         │                     │  └─────────────────────────┘ │
         │                     └──────────────────────────────┘
         │                                    │ Submit
         │                                    ▼
         │                     ┌──────────────────────────────┐
         │                     │   Purchase Invoice (PI)      │
         │                     │   Tự động tạo + submit       │
         │                     │   6 dòng thuế nhập khẩu      │
         │                     └──────────────────────────────┘
         │                                    │
         │                          Nút "Create > Purchase Receipt"
         │                                    ▼
         │                     ┌──────────────────────────────┐
         │                     │   Purchase Receipt (PR)      │
         │                     │   Nhập kho hàng hóa          │
         │                     └──────────────────────────────┘
         │                                    │ Submit
         │                                    ▼
         │              ┌────────────────────────────────────────┐
         │              │   Landed Cost Voucher (LCV) — TỰ ĐỘNG  │
         │              │   1 LCV / loại thuế có giá trị > 0:    │
         │              │   • LCV cho Import Tax Duty             │
         │              │   • LCV cho Special Consumption Tax     │
         │              │   • LCV cho Antidumping Tax             │
         │              │   • LCV cho Environmental Protection Tax│
         │              └────────────────────────────────────────┘
         │
         │              Người dùng cũng có thể tạo thêm LCV thủ công
         │              từ nút "Create > Landed Cost Voucher" trên PV
         │              (dùng cho Pre Customs Cost và Purchase Cost)
         │
         └─ [Sau khi PI submit] ──►  Payment Entry (PE)
                                     Khi tạo PE từ PI, tự động gán
                                     purchase_voucher vào PE
```

---

## 3. Bước 1 — Purchase Order (PO)

### Điều kiện để tạo PV từ PO

Nút **"Create > Purchase Voucher"** xuất hiện trên PO khi:
- `docstatus = 1` (đã submit)
- `purchase_invoice_type` đã chọn
- `per_voucher_created < 100` (chưa tạo PV đủ 100% số lượng)

### Dữ liệu được map sang PV

Khi nhấn nút, hàm `make_purchase_voucher` chạy và map:

| Trường PO | → Trường PV | Ghi chú |
|---|---|---|
| `purchase_invoice_type` | `purchase_invoice_type` | Xác định loại PV |
| `company` | `company` | |
| `supplier` | `supplier` | |
| `buying_price_list` | `price_list` | |
| `currency` | `currency` | |
| `conversion_rate` | `exchange_rate` | |
| `set_warehouse` | `warehouse` | |
| `apply_discount_on` | `apply_additional_discount_on` | |
| `additional_discount_percentage` | `additional_discount_percentage` | |
| `discount_amount` | `additional_discount_amount` | |
| `name` (PO) | `purchase_order` | Liên kết ngược |
| Items của PO | `items` hoặc `service_pv_items` | Tùy loại |

**Với từng dòng item được map:**
- `qty_po = PO Item.qty − PO Item.qty_pv` (số lượng còn lại chưa tạo PV)
- `qty = qty_po` (gán mặc định bằng số còn lại)
- `payable_account` ← `Company.default_payable_account`
- `vat_account` ← Tài khoản `1331` của company
- `inventory_account` ← `Company.default_inventory_account`

**Riêng loại Nhập Khẩu**, mỗi item còn tạo thêm 1 dòng trong bảng `taxes`:
- `import_tax_account` ← `Company.import_tax_account`
- `special_consumption_tax_account` ← `Company.special_consumption_tax_account`
- `antidumping_tax_account` ← `Company.antidumping_tax_account`
- `vat_account` ← `Company.value_added_tax_account`
- `vat_corresponding_account` ← `Company.value_added_tax_correspondent_account`
- `environmental_protection_tax_account` ← `Company.environmental_protection_tax_account`
- `customs_exchange_rate` = `conversion_rate` của PO
- `fob_rate = item.amount − distributed_discount_amount`
- `import_tax_base = fob_rate × conversion_rate`

---

## 4. Bước 2 — Purchase Voucher (PV)

### 4.1 Cấu trúc dữ liệu

```
Purchase Voucher
├── [Header] Thông tin chung
├── service_pv_items  ← Dùng khi Mua Dịch Vụ
├── items             ← Dùng khi Mua Hàng (Trong Nước / Nhập Khẩu)
├── taxes             ← Chỉ dùng khi Mua Hàng Nhập Khẩu Nhập Kho
├── pre_cost          ← Chi phí trước thông quan (từ PV Dịch Vụ)
└── pv_purchase_cost  ← Chi phí mua hàng trong nước (từ PV Dịch Vụ)
```

### 4.2 Trường Header

#### Thông tin cơ bản (nhập tay hoặc tự điền)

| Trường | Cách lấy giá trị | Ràng buộc |
|---|---|---|
| `supplier` | Người dùng chọn | Tự điền `address` từ supplier primary address |
| `currency` | Người dùng / khi chọn Price List | |
| `exchange_rate` | Tự động từ `get_exchange_rate(currency → VND)` | |
| `price_list` | Mặc định "Standard Buying" khi refresh | |
| `payment_term` | Người dùng | Khi chọn → tự tính `credit_days` và `due_date` |
| `due_date` | `posting_date + credit_days` | Bắt buộc `≥ posting_date` |
| `voucher_date` | Người dùng | Bắt buộc `≤ posting_date` |
| `warehouse` | Người dùng (header) | Khi đổi → đồng bộ xuống tất cả dòng items |

#### Trường tổng (tính tự động phía frontend)

> Trigger: thay đổi bất kỳ trường nào trên dòng item (qty, rate, discount, vat)

```
total       = Σ(qty × rate)          trên tất cả dòng item active
net_total   = total − additional_discount_amount
vat         = Σ(vat_amount)          từng dòng item  [Dịch Vụ / Trong Nước]
grand_total = net_total + vat        [Dịch Vụ / Trong Nước]
grand_total = net_total              [Nhập Khẩu — VAT tính riêng trong bảng taxes]

base_*      = * × exchange_rate      (quy đổi sang VND)
```

#### Chiết khấu bổ sung

| Kịch bản | Cách tính |
|---|---|
| Nhập `additional_discount_percentage` | `additional_discount_amount = base × percentage / 100` |
| Nhập `additional_discount_amount` | `additional_discount_percentage = amount / base × 100` |
| `apply_additional_discount_on = "Grand Total"` | `base = total + vat` |
| `apply_additional_discount_on = "Net Total"` | `base = total` |

Khi áp chiết khấu theo **Net Total**, chiết khấu được phân bổ xuống từng dòng item:
- `discount_allocation_type = "Quantity"`: theo tỷ lệ `row.qty / tổng_qty`
- `discount_allocation_type = "Amount"`: theo tỷ lệ `row.amount / tổng_amount`
- Kết quả lưu vào `distributed_discount_amount` từng dòng
- `fob_rate = amount − distributed_discount_amount` (căn cứ tính thuế nhập khẩu)

#### Trường tracking tiến độ

| Trường | Ý nghĩa | Khi nào cập nhật |
|---|---|---|
| `total_qty` | Tổng SL item | `before_save` |
| `per_receipted` | % số lượng đã nhận kho | Khi PR submit/cancel |
| `per_allocated` | % chi phí dịch vụ đã phân bổ qua LCV | Khi LCV submit/cancel |
| `has_been_used` | PV Dịch Vụ đã được phân bổ hết | Khi PV Hàng submit/cancel |
| `has_been_created_lcv` | Đã được đưa vào LCV | Khi LCV insert/submit/cancel |
| `accumulated_allocation` | Tổng tiền đã phân bổ sang PV Hàng | Khi PV Hàng submit/cancel |
| `translated_accumulated_allocation` | `accumulated_allocation × exchange_rate` | Cùng lúc với trên |

#### Giá trị tồn kho đầu vào (`inventory_in_value`)

```
[Mua Trong Nước Nhập Kho]
inventory_in_value (base) = net_total + purchase_cost

[Mua Nhập Khẩu Nhập Kho]
inventory_in_value (base) = base_grand_total
                          + Σ base_total_cost (pre_cost)
                          + Σ total_cost (pv_purchase_cost)
                          + import_tax_duty
                          + special_consumption_tax
                          + env_protection_tax
                          + antidumping_tax
```

### 4.3 Bảng `service_pv_items` / `items`

#### Khi thêm dòng mới
Tự động điền tài khoản từ Company:
- `payable_account` ← `Company.default_payable_account`
- `inventory_account` ← `Company.default_inventory_account`
- `cost_account` ← `Company.temporary_account` (chỉ Dịch Vụ)
- `vat_account` ← Tài khoản số `1331`

#### Tính toán trên dòng item

| Trường thay đổi | Kết quả tính |
|---|---|
| `qty` hoặc `rate` | `amount = qty × rate` → kéo `base_amount`, `vat_amount`, `totals` |
| `discount_percent` | `discount_amount = original_rate × pct / 100`; `rate = original_rate − discount_amount` |
| `discount_amount` | `discount_percent = amount / original_rate × 100`; `rate = original_rate − amount` |
| `vat_template` | `vat_percent` lấy từ template; `vat_amount = amount × vat_percent / 100` |
| `vat_amount` | `vat_percent = vat_amount / amount × 100` |
| `fob_rate` | `base_fob_rate = fob_rate × exchange_rate` |
| `warehouse` | `inventory_account` ← `Warehouse.account` hoặc fallback Company |

#### Riêng loại Nhập Khẩu: khi thêm/đổi `item_code`
- Tạo dòng thuế tương ứng trong bảng `taxes`
- Nếu đổi item_code cũ → xóa dòng thuế cũ, tạo dòng thuế mới
- Khi xóa item → xóa dòng thuế tương ứng

### 4.4 Bảng `taxes` (Purchase Voucher Tax) — chỉ Nhập Khẩu

Mỗi item hàng có **một dòng thuế riêng**. Chuỗi tính toán theo thứ tự bắt buộc:

```
━━━ BƯỚC 1: Xác định căn cứ tính thuế nhập khẩu ━━━

import_tax_base_origin_currency
  = fob_rate + pre_customs_cost_in_foreign_currency        (ngoại tệ)
  = pre_customs_cost_in_foreign_currency + fob_rate / customs_exchange_rate  (VND)

import_tax_base
  = import_tax_base_origin_currency × customs_exchange_rate
  + pre_customs_cost_in_accounting_currency

━━━ BƯỚC 2: Thuế nhập khẩu ━━━

import_tax_duty   = import_tax_base × import_tax_rate / 100
import_tax_payable = import_tax_duty

━━━ BƯỚC 3: Thuế chống bán phá giá ━━━

antidumping_tax_amount = import_tax_base × antidumping_tax_rate / 100

━━━ BƯỚC 4: Thuế bảo vệ môi trường ━━━

total_tax_env    = import_tax_base + import_tax_duty + antidumping_tax_amount
env_tax_amount   = total_tax_env × env_protection_tax_rate / 100

━━━ BƯỚC 5: Thuế tiêu thụ đặc biệt ━━━

totalTaxTTDB                   = import_tax_base + import_tax_duty + env_tax_amount
special_consumption_tax_amount = totalTaxTTDB × special_consumption_tax_rate / 100
special_consumption_tax_payable = special_consumption_tax_amount

━━━ BƯỚC 6: VAT ━━━

totalVAT  = import_tax_base + import_tax_duty + env_tax_amount
          + special_consumption_tax_amount
vat_amount = totalVAT × vat / 100
vat_payable = vat_amount
```

> **Tất cả các bước tính tự động** khi người dùng thay đổi bất kỳ rate hay base nào.  
> Nhập thẳng amount cũng được — hệ thống tính ngược ra rate.

Sau khi tính từng dòng, `calculate_total_taxes` tổng hợp lên header:

```
base_import_tax              = Σ import_tax_duty
base_special_consumption_tax = Σ special_consumption_tax_amount
base_environmental_protection_tax = Σ env_tax_amount
base_antidumping_tax         = Σ antidumping_tax_amount
base_vat                     = Σ vat_amount

(import_tax, special_consumption_tax, ...) = base_* / customs_exchange_rate
```

### 4.5 Bảng `pre_cost` (Chi phí trước thông quan)

Dùng ghi nhận chi phí **trước khi hàng qua hải quan** (cước vận tải quốc tế, phí môi giới hải quan, phí lưu kho ngoại quan...) từ các PV Dịch Vụ đã submit.

**Cách thêm dòng:** Nhấn **"Choose Cost Voucher"** trong grid `pre_cost` → dialog hiện danh sách PV Dịch Vụ có `is_cost=1`, `has_been_used=0`.

**Trường được tính khi load (hàm `make_precustom_cost`):**

```
total_cost          = grand_total của PV dịch vụ         (ngoại tệ)
base_total_cost     = base_grand_total của PV dịch vụ    (VND)
amount_allocated    = total_cost − accumulated_allocation    (nếu tiền ngoại tệ)
                    = base_total_cost − accumulated_allocation (nếu VND)
base_amount_allocated = amount_allocated × exchange_rate
```

**Khi người dùng sửa `amount_allocated`:**
- Kiểm tra không vượt quá phần còn lại: `total_cost − PV.accumulated_allocation`
- `base_amount_allocated = amount_allocated × exchange_rate`

**Sau khi xóa dòng hoặc sau khi phân bổ (Allocate Cost):**
`calculate_total_pre_cost` cập nhật header và tính lại bảng `taxes`:
```
base_pre_customs_cost = Σ total_cost_added
pre_customs_cost = base_pre_customs_cost / exchange_rate

→ Cập nhật pre_customs_cost_in_accounting_currency và _foreign_currency vào từng dòng taxes
→ Tính lại import_tax_base → kéo theo toàn bộ chuỗi thuế
```

**Nút "Allocate Cost"** (phân bổ chi phí trước thông quan xuống từng item):

| Chọn loại phân bổ | Công thức |
|---|---|
| Percentage By Quantity | `allocation_percent = row.qty / tổng_qty × 100` |
| Percentage By Amount | `allocation_percent = row.amount / tổng_amount × 100` |

Với mỗi item: 
```
pre_customs_cost (ngoại tệ) = foreign_original × allocation_percent / 100
translated_pre_customs_cost  = pre_customs_cost × customs_exchange_rate
base_pre_customs_cost (VND)  = accounting × allocation_percent / 100
total_cost                   = translated + base
```
Sau đó cập nhật lại bảng `taxes`: `import_tax_base_origin_currency` và `import_tax_base` → toàn bộ thuế tính lại.

### 4.6 Bảng `pv_purchase_cost` (Chi phí mua hàng trong nước)

Dùng ghi nhận chi phí **sau khi hàng qua hải quan về kho** (phí vận chuyển nội địa, bốc xếp...) từ các PV Dịch Vụ.

**Cách thêm dòng:** Nhấn **"Choose Cost Voucher"** trong grid `pv_purchase_cost`.

**Trường được tính (hàm `make_cost_voucher`):**
```
total_cost         = grand_total của PV dịch vụ (VND)
base_purchase_cost = Σ total_cost
purchase_cost      = base_purchase_cost / exchange_rate
inventory_in_value = base_grand_total + base_purchase_cost
```

### 4.7 Lifecycle — PV trước khi submit

```
on_refresh:
  - Set price_list = "Standard Buying" nếu chưa có
  - Thêm nút "Choose Cost Voucher" vào grid pre_cost và pv_purchase_cost
  - Thêm nút "Allocate Cost" vào grid pre_cost (chỉ khi draft)
  - Nếu đã submit + loại nhập kho + per_receipted < 100 → nút "Create > Purchase Receipt"
  - Nếu đã submit + có PR submitted + per_allocated < 100 → nút "Create > Landed Cost Voucher"

before_save:
  1. validate_qty_against_po():
     - Mỗi dòng item: qty ≤ qty_po
  2. Với loại nhập kho: kiểm tra mỗi dòng phải có warehouse
  3. total_qty = Σ qty từ bảng items active

validate:
  - qty > 0 trên tất cả dòng items
```

### 4.8 Lifecycle — PV khi Submit

```
on_submit:
  ┌─ 1. update_po_qty_pv() ────────────────────────────────────────────┐
  │  Mỗi dòng item:                                                     │
  │    SQL: PO Item.qty_pv += row.qty                                   │
  │  Tính tỷ lệ:                                                        │
  │    per_voucher_created_qty = (PV.total_qty / PO.total_qty) × 100   │
  │    PO.per_voucher_created += per_voucher_created_qty                │
  └────────────────────────────────────────────────────────────────────┘
  ┌─ 2. create_pi() ───────────────────────────────────────────────────┐
  │  Tạo Purchase Invoice mới:                                          │
  │  - supplier, currency, exchange_rate, company từ PV                │
  │  - PI.purchase_voucher = PV.name                                    │
  │  - Mỗi item → 1 dòng PI Item (có purchase_voucher_item)            │
  │                                                                     │
  │  Taxes theo loại:                                                   │
  │  [Dịch Vụ / Trong Nước]                                            │
  │    Mỗi item → 1 dòng tax: Actual / vat_account / vat_amount / Add │
  │                                                                     │
  │  [Nhập Khẩu] — 6 dòng cố định (tổng hợp toàn bộ bảng taxes):     │
  │    1. import_tax_account     → Add  / Valuation                    │
  │    2. special_cons_tax_acc   → Add  / Valuation                    │
  │    3. env_protection_tax_acc → Add  / Valuation                    │
  │    4. vat_corresponding_acc  → Add  / Total                        │
  │    5. antidumping_tax_acc    → Add  / Valuation                    │
  │    6. vat_account            → Deduct / Total                      │
  │    (tất cả amount ÷ customs_exchange_rate để về VND)               │
  │                                                                     │
  │  → pi.insert() + pi.submit() tự động                               │
  │  → Hiển thị thông báo "Purchase Invoice {name} created"            │
  └────────────────────────────────────────────────────────────────────┘
  ┌─ 3. update_pv_mdv() ───────────────────────────────────────────────┐
  │  [Chỉ chạy khi KHÔNG phải Mua Dịch Vụ]                            │
  │                                                                     │
  │  Với bảng pv_purchase_cost:                                         │
  │    PV dịch vụ (voucher_no).has_been_used = 1                       │
  │                                                                     │
  │  Với bảng pre_cost:                                                 │
  │    accumulated_allocation += row.amount_allocated                   │
  │    translated_accumulated_allocation = acc × exchange_rate          │
  │    Nếu accumulated_allocation ≥ PV.grand_total:                    │
  │      → has_been_used = 1                                           │
  └────────────────────────────────────────────────────────────────────┘
```

### 4.9 Lifecycle — PV khi Cancel

```
on_cancel:
  1. cancel_purchase_invoice():
     - Tìm PI có purchase_voucher = PV.name và docstatus = 1
     - pi.cancel() → PI bị hủy
     - Thông báo "Purchase Invoice {name} cancelled"

  2. Trừ qty_pv trên PO:
     - Mỗi dòng item: PO Item.qty_pv -= row.qty
     - PO.per_voucher_created -= (PV.total_qty / PO.total_qty) × 100

  3. update_pv_mdv_on_cancel():
     [Chỉ chạy khi KHÔNG phải Mua Dịch Vụ]
     
     Với bảng pv_purchase_cost:
       Kiểm tra xem có PV submitted nào khác đang dùng cùng PV dịch vụ không
       → Nếu không có → has_been_used = 0 (mới reset, tránh reset nhầm)
     
     Với bảng pre_cost:
       accumulated_allocation -= row.amount_allocated
       translated_accumulated_allocation = acc × exchange_rate
       has_been_used = 0
```

---

## 5. Bước 3 — Purchase Invoice (PI)

PI được **tạo và submit tự động** bởi PV khi PV submit. Người dùng không tạo thủ công.

### Dữ liệu PI nhận từ PV

| Trường PI | Nguồn | Ghi chú |
|---|---|---|
| `supplier` | PV.supplier | |
| `currency` | PV.currency | |
| `conversion_rate` | PV.exchange_rate | |
| `company` | PV.company | |
| `buying_price_list` | PV.price_list | |
| `purchase_voucher` | PV.name | **Trường liên kết quan trọng** |
| `credit_to` | `row.payable_account` (dòng cuối xử lý) | Tài khoản phải trả |
| `apply_discount_on` | PV.apply_additional_discount_on | |
| `discount_amount` | PV.additional_discount_amount | (nếu pct ≤ 0) |
| `additional_discount_percentage` | PV.additional_discount_percentage | (nếu pct > 0) |

### PI Items

Mỗi dòng item PV → 1 dòng PI Item:
- `purchase_voucher_item = PV Item.name` — **trường liên kết** dùng để tìm lại item
- `expense_account = row.cost_account` (chỉ Dịch Vụ)

### PI Taxes (xem chi tiết ở mục 4.8 trên)

### Sau khi PI submit

PI ảnh hưởng đến:
- **General Ledger**: PI tạo bút toán kế toán (công nợ phải trả, chi phí/tài sản)
- **Payment Entry**: khi tạo PE từ PI, hook `add_pv_to_pe` tự gán `PE.purchase_voucher = PI.purchase_voucher`

### Xem Accounting Ledger từ PV

Nút **"View > Accounting Ledger"** trên PV (khi đã submit): lấy PI liên kết và mở General Ledger Report theo `voucher_no = PI.name`.

---

## 6. Bước 4 — Purchase Receipt (PR)

### Điều kiện tạo PR

Nút **"Create > Purchase Receipt"** xuất hiện trên PV khi:
- PV đã submit (`docstatus = 1`)
- Loại là "Mua Hàng Trong Nước Nhập Kho" hoặc "Mua Hàng Nhập Khẩu Nhập Kho"
- `per_receipted < 100`

### Dữ liệu được map sang PR (hàm `make_purchase_receipt`)

| Trường PV | → Trường PR |
|---|---|
| `name` | `purchase_voucher` |
| `supplier` | `supplier` |
| `company` | `company` |
| `currency` | `currency` |
| `exchange_rate` | `conversion_rate` |
| `price_list` | `buying_price_list` |
| `apply_additional_discount_on` | `apply_discount_on` |

> **Lưu ý Exchange Rate:** Nếu exchange_rate của PR khác PV, một nút "Cập nhật Exchange Rate từ Purchase Voucher" hiện ra để người dùng đồng bộ.

### PR Items

Mỗi dòng PV Item → 1 dòng PR Item:

| Trường PR Item | Nguồn | Ghi chú |
|---|---|---|
| `purchase_voucher_item` | PV Item.name | Liên kết ngược |
| `purchase_invoice` | PI.name (tìm theo purchase_voucher) | |
| `purchase_invoice_item` | PI Item.name (tìm theo purchase_voucher_item) | |
| `qty_pv` | `PV Item.qty − PV Item.qty_pr` | Số lượng còn được nhận |
| `qty` | `qty_pv` (gán mặc định) | |
| `expense_account` | PI Item.expense_account (tìm theo purchase_voucher_item) | |

### PR Taxes

**[Mua Trong Nước]:** Copy `vat_account + vat_amount` từng item PV sang PR taxes.

**[Mua Nhập Khẩu]:** Chỉ map 2 dòng VAT (không map thuế nhập khẩu — các thuế đó sẽ vào LCV):
```
1. vat_corresponding_account  → Add  / Total  (VAT đầu vào được khấu trừ)
2. vat_account                → Deduct / Total (VAT phải nộp)
Amount = tổng vat_amount / customs_exchange_rate
```

### Validate trước khi save PR (hook `validate_qty_pr`)

```
before_save PR:
  Mỗi dòng item:
    qty trong PR ≤ qty_pv (số lượng còn lại từ PV)
    → Nếu vượt: throw lỗi
```

### Khi Submit PR (hooks `update_qty_pv_item` + `create_lcv`)

```
on_submit PR:
  ┌─ Hook 1: update_qty_pv_item ───────────────────────────────────────┐
  │  Nếu PR có purchase_voucher:                                        │
  │  Batch SQL:                                                         │
  │    UPDATE PV Item SET qty_pr += PR Item.qty                        │
  │    (cho tất cả dòng có purchase_voucher_item, một lần duy nhất)    │
  │                                                                     │
  │  Cập nhật PV.per_receipted:                                        │
  │    delta = (PR.total_qty / PV.total_qty) × 100                    │
  │    SQL: PV.per_receipted += delta                                  │
  └────────────────────────────────────────────────────────────────────┘
  ┌─ Hook 2: create_lcv ───────────────────────────────────────────────┐
  │  [Chỉ chạy nếu PV là "Mua Hàng Nhập Khẩu Nhập Kho" và có taxes]  │
  │                                                                     │
  │  1. Nhóm thuế theo item_code từ PV.taxes                          │
  │  2. Tính tổng từng loại thuế: import, special, antidumping, env   │
  │  3. Tạo riêng 1 LCV cho mỗi loại thuế > 0:                        │
  │     - distribute_charges_based_on = "Distribute Manually"          │
  │     - Thêm PR vào purchase_receipts                                │
  │     - Mỗi item PR → 1 dòng LCV items                              │
  │       applicable_charges = thuế tương ứng của item_code đó        │
  │     - 1 dòng taxes: expense_account + amount                       │
  │     - lcv.insert() + lcv.submit() tự động                         │
  │  4. Thông báo các LCV đã tạo                                       │
  └────────────────────────────────────────────────────────────────────┘
```

### Khi Cancel PR

```
on_cancel PR:
  Hook update_qty_pv_on_cancel:
    Batch SQL: PV Item.qty_pr -= PR Item.qty
    SQL: PV.per_receipted -= (PR.total_qty / PV.total_qty) × 100
```

---

## 7. Bước 5 — Landed Cost Voucher (LCV)

LCV phân bổ các chi phí phụ (thuế, cước...) vào giá vốn từng mặt hàng trong kho.

### Hai nguồn tạo LCV

#### Nguồn A — Tự động khi submit PR (chỉ Nhập Khẩu)

Xem chi tiết ở mục 6 Hook 2. Tạo tự động, người dùng không cần làm gì.

#### Nguồn B — Thủ công từ nút "Create > Landed Cost Voucher" trên PV

Nút xuất hiện khi có ít nhất 1 PR đã submit và `per_allocated < 100`.

**Dialog chọn chi phí:**
- Chọn loại: "Pre Customs Cost" hoặc "Purchase Cost"
- Hiện danh sách các dòng `pre_cost` / `pv_purchase_cost` có `is_allocated = 0`
- Người dùng chọn 1 hoặc nhiều dòng → nhấn "Create LCV"

**Hàm `make_land_cost_voucher` tạo LCV:**
1. Tìm tất cả PR đã submit liên kết với PV này
2. Tạo LCV mới, thêm tất cả PR vào `purchase_receipts`
3. Thêm tất cả items từ các PR vào LCV items (applicable_charges = 0 ban đầu)
4. Thêm các chi phí đã chọn vào `taxes` của LCV (`service_pv` ghi lại PV dịch vụ nguồn)
5. Tính `total_taxes_and_charges = Σ base_amount`
6. Gọi `set_applicable_charges_on_item()` phân bổ tự động
7. `is_for_pv = 1` — đánh dấu LCV này thuộc hệ thống PV

**Phân bổ `distribute_charges_based_on`:**
- Pre Customs Cost: theo `allocation_type` của dòng pre_cost (Qty hoặc Amount)
- Purchase Cost: theo lựa chọn của người dùng trong dialog (Qty hoặc Amount)

### Hooks trên LCV

```
after_insert LCV:
  update_service_pv_in_landed_cost_voucher:
    Nếu is_for_pv = 1:
      Mỗi dòng taxes có service_pv:
        Nếu PV dịch vụ đó has_been_used = 1:
          → PV dịch vụ.has_been_created_lcv = 1

on_submit LCV:
  ┌─ Hook 1: update_service_pv_in_landed_cost_voucher ─────────────────┐
  │  (như after_insert, đảm bảo has_been_created_lcv = 1)              │
  └────────────────────────────────────────────────────────────────────┘
  ┌─ Hook 2: update_allocated_percent ─────────────────────────────────┐
  │  Nếu is_for_pv = 1:                                                │
  │  Lấy PV hàng từ: items[0].purchase_voucher                        │
  │  total_taxes = LCV.total_taxes_and_charges                         │
  │                                                                     │
  │  Đánh dấu các dòng được sử dụng:                                  │
  │    pre_cost row.is_allocated = 1 (nếu voucher_number có trong LCV) │
  │    pv_purchase_cost row.is_allocated = 1 (nếu voucher_no có trong) │
  │                                                                     │
  │  Tính tổng chi phí:                                                │
  │    total_cost = Σ pre_cost.base_amount_allocated                   │
  │               + Σ pv_purchase_cost.total_cost                      │
  │                                                                     │
  │  Cập nhật PV:                                                      │
  │    per_allocated_lcv = total_taxes / total_cost × 100              │
  │    PV.per_allocated += per_allocated_lcv                           │
  └────────────────────────────────────────────────────────────────────┘

on_cancel LCV:
  ┌─ Hook 1: clear_service_pv_in_landed_cost_voucher ──────────────────┐
  │  PV dịch vụ.has_been_created_lcv = 0                              │
  └────────────────────────────────────────────────────────────────────┘
  ┌─ Hook 2: update_allocated_percent_on_cancel ────────────────────────┐
  │  Đặt lại is_allocated = 0 cho các dòng pre_cost / pv_purchase_cost │
  │  PV.per_allocated -= per_allocated_lcv                             │
  └────────────────────────────────────────────────────────────────────┘

on_trash LCV:
  clear_service_pv_in_landed_cost_voucher (như on_cancel)
```

---

## 8. Bước 6 — Payment Entry (PE)

PE được tạo theo luồng chuẩn ERPNext từ PI. Customization duy nhất:

```
before_insert PE:
  Hook add_pv_to_pe:
    Duyệt qua doc.references:
      Nếu reference_doctype = "Purchase Invoice":
        purchase_voucher = PI.purchase_voucher
        → PE.purchase_voucher = purchase_voucher
      Nếu reference_doctype = "Sales Invoice":
        sales_voucher = SI.sales_voucher
        → PE.sales_voucher = sales_voucher
```

Mục đích: cho phép tra cứu Payment Entry từ Purchase Voucher.

---

## 9. Tracking trạng thái xuyên suốt

### Sơ đồ liên kết trạng thái

```
Purchase Order
  └── per_voucher_created  (0 → 100%)
        Cộng khi: PV submit → update_po_qty_pv()
        Trừ khi:  PV cancel → on_cancel()

Purchase Voucher (Hàng)
  ├── per_receipted        (0 → 100%)
  │     Cộng khi: PR submit → update_qty_pv_item()
  │     Trừ khi:  PR cancel → update_qty_pv_on_cancel()
  │
  ├── per_allocated        (0 → 100%)
  │     Cộng khi: LCV submit → update_allocated_percent()
  │     Trừ khi:  LCV cancel → update_allocated_percent_on_cancel()
  │
  └── [liên kết PV Dịch Vụ qua pre_cost.voucher_number / pv_purchase_cost.voucher_no]
        pre_cost.is_allocated        = 1 khi LCV submit
        pv_purchase_cost.is_allocated = 1 khi LCV submit

Purchase Voucher (Dịch Vụ)
  ├── has_been_used = 1
  │     Set khi: PV Hàng submit (update_pv_mdv)
  │              → accumulated_allocation ≥ grand_total
  │     Reset khi: PV Hàng cancel (update_pv_mdv_on_cancel)
  │                → chỉ reset nếu không còn PV khác dùng nó
  │
  ├── accumulated_allocation
  │     Cộng khi: PV Hàng submit → tích lũy amount_allocated từ pre_cost
  │     Trừ khi:  PV Hàng cancel
  │
  └── has_been_created_lcv = 1
        Set khi: LCV after_insert / on_submit (có service_pv = PV này)
        Reset khi: LCV on_cancel / on_trash

Purchase Voucher Item
  ├── qty_pv = Σ qty đã tạo PV (từ PO)
  │     Cộng khi: PV submit → update_po_qty_pv()
  │     Trừ khi:  PV cancel
  └── qty_pr = Σ qty đã nhận kho (từ PR)
        Cộng khi: PR submit → update_qty_pv_item()
        Trừ khi:  PR cancel → update_qty_pv_on_cancel()
```

---

## 10. Kiểm soát tính hợp lệ

| Thời điểm | Điều kiện kiểm tra | Hành động khi sai |
|---|---|---|
| PV `validate` | `qty > 0` mỗi dòng item | throw lỗi |
| PV `before_save` | `qty ≤ qty_po` (so sánh với số còn lại từ PO) | throw lỗi |
| PV `before_save` | Loại nhập kho: mỗi item phải có `warehouse` | throw lỗi |
| PR `before_save` | `qty ≤ qty_pv` (so sánh với số còn lại từ PV) | throw lỗi |
| pre_cost `amount_allocated` | Không vượt quá `total_cost − accumulated_allocation` | msgprint + reset về 0 |
| Nút tạo PR | `per_receipted < 100` | Ẩn nút |
| Nút tạo LCV | Có PR submitted + `per_allocated < 100` | Ẩn nút |
| `make_land_cost_voucher` | Phải có ít nhất 1 PR submitted | throw lỗi |
| `make_land_cost_voucher` | Phải chọn ít nhất 1 expense | throw lỗi |

---

## 11. Tra cứu nhanh — Hooks đã đăng ký

### Purchase Receipt

| Event | Hook |
|---|---|
| `before_save` | `validate_qty_pr` — kiểm tra qty PR ≤ qty PV |
| `on_submit` | `update_qty_pv_item` — cộng qty_pr và per_receipted trên PV |
| `on_submit` | `create_lcv` — tự động tạo LCV thuế nhập khẩu |
| `on_cancel` | `update_qty_pv_on_cancel` — trừ qty_pr và per_receipted |

### Landed Cost Voucher

| Event | Hook |
|---|---|
| `after_insert` | `update_service_pv_in_landed_cost_voucher` |
| `on_submit` | `update_service_pv_in_landed_cost_voucher` |
| `on_submit` | `update_allocated_percent` — cộng per_allocated trên PV |
| `on_cancel` | `clear_service_pv_in_landed_cost_voucher` |
| `on_cancel` | `update_allocated_percent_on_cancel` — trừ per_allocated |
| `on_trash` | `clear_service_pv_in_landed_cost_voucher` |

### Payment Entry

| Event | Hook |
|---|---|
| `before_insert` | `add_pv_to_pe` — gán purchase_voucher từ PI vào PE |

### Purchase Voucher (Class Methods, không phải hooks)

| Method | Khi nào | Mô tả |
|---|---|---|
| `validate` | Mỗi lần save | Kiểm tra qty > 0 |
| `before_save` | Mỗi lần save | Validate PO, warehouse, tính total_qty |
| `on_submit` | Khi submit | update_po_qty_pv + create_pi + update_pv_mdv |
| `on_cancel` | Khi cancel | cancel_pi + trừ PO qty + update_pv_mdv_on_cancel |

---

## Phụ lục: Bộ lọc tài khoản

Các trường tài khoản có bộ lọc riêng để chỉ hiện tài khoản phù hợp:

| Trường | Bộ lọc tài khoản |
|---|---|
| `cost_account` | Đầu 1, 2, hoặc 6 |
| `payable_account` | Đầu 1, 3, hoặc 7 |
| `inventory_account` | Đầu 1 hoặc 6 |
| `import_tax_account`, `special_consumption_tax_account`, `vat_account`, `environmental_protection_tax_account` trong bảng taxes | Đầu 3 |
| `vat_corresponding_account` | Là một trong: 1331, 1332, 1388, 3388 |
| `items.cost_account` (loại Nhập Khẩu) | Đầu 1 |
