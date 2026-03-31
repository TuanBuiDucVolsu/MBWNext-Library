# Tài Liệu Kỹ Thuật — Tích Hợp Hóa Đơn Điện Tử (E-Invoice)

> **Phiên bản:** 1.0 — Tháng 3/2026
> **Đối tượng:** Lập trình viên (Dev) trong đội phát triển ERP
> **Phạm vi:** Module `mbwnext_einvoice` — Viettel SInvoice & VNPT

---

## Mục Lục

1. [Tổng Quan Kiến Trúc](#1-tổng-quan-kiến-trúc)
2. [Cấu Trúc Thư Mục](#2-cấu-trúc-thư-mục)
3. [Cấu Hình Hệ Thống](#3-cấu-hình-hệ-thống)
4. [Custom Fields trên Sales Invoice](#4-custom-fields-trên-sales-invoice)
5. [Luồng Phát Hành Hóa Đơn — Viettel](#5-luồng-phát-hành-hóa-đơn--viettel)
6. [Luồng Phát Hành Hóa Đơn — VNPT](#6-luồng-phát-hành-hóa-đơn--vnpt)
7. [Luồng Hủy Hóa Đơn](#7-luồng-hủy-hóa-đơn)
8. [Luồng Xem PDF Hóa Đơn](#8-luồng-xem-pdf-hóa-đơn)
9. [Giao Diện Người Dùng (UI)](#9-giao-diện-người-dùng-ui)
10. [Xử Lý Dữ Liệu — Bên Bán & Bên Mua](#10-xử-lý-dữ-liệu--bên-bán--bên-mua)
11. [Cơ Chế Tự Động (Hooks)](#11-cơ-chế-tự-động-hooks)
12. [Xử Lý Lỗi & Debugging](#12-xử-lý-lỗi--debugging)
13. [Bản Dịch (i18n)](#13-bản-dịch-i18n)
14. [Lưu Ý Quan Trọng & Best Practice](#14-lưu-ý-quan-trọng--best-practice)

---

## 1. Tổng Quan Kiến Trúc

```
ERPNext (Sales Invoice)
       │
       ▼
   sales_invoice.js          ← Giao diện nút bấm, preview PDF
       │
       ▼
   api/viettel.py             ← Whitelist API entry-point Viettel
   api/vnpt.py                ← Whitelist API entry-point VNPT
   api/einvoice.py            ← API chung: lấy PDF (dispatch Viettel/VNPT)
       │
       ▼
   integrations/viettel/      ← Logic nghiệp vụ Viettel
     ├── config.py            ← Đọc cấu hình từ Company
     ├── client.py            ← HTTP client REST API SInvoice
     └── service.py           ← Build payload, gọi client, parse kết quả
       │
   integrations/vnpt/         ← Logic nghiệp vụ VNPT
     ├── config.py            ← Đọc cấu hình từ Company
     ├── client.py            ← SOAP client ASMX
     └── service.py           ← Build XML TT78, gọi client, parse kết quả
       │
       ▼
   controllers/python/
     └── sales_invoice.py     ← on_cancel hook: tự hủy e-invoice khi hủy SI
```

**Nguyên tắc thiết kế:**

- Cấu hình **không** dùng DocType riêng (`E-Invoice Settings`), thay vào đó dùng **custom fields trên Company** (tab `E-Invoice Settings`).
- Mỗi `Sales Invoice` tự biết nhà cung cấp qua `Company.provider`.
- Mọi cập nhật trạng thái về SI dùng `frappe.db.set_value(...)` + `frappe.db.commit()` — **không** dùng `si.save()` để tránh trigger lại validate.

---

## 2. Cấu Trúc Thư Mục

```
apps/mbwnext_einvoice/mbwnext_einvoice/
├── hooks.py                        ← Khai báo doctype_js, doc_events, fixtures
├── fixtures/
│   └── custom_field.json           ← Custom fields cho Sales Invoice
├── controllers/
│   ├── js/
│   │   ├── sales_invoice.js        ← JS giao diện hóa đơn điện tử
│   │   └── company.js              ← JS nút Test Connection trên Company
│   └── python/
│       └── sales_invoice.py        ← on_cancel hook (Python)
├── integrations/
│   ├── viettel/
│   │   ├── config.py
│   │   ├── client.py
│   │   └── service.py
│   └── vnpt/
│       ├── config.py
│       ├── client.py
│       └── service.py
└── mbwnext_einvoice/
    └── api/
        ├── einvoice.py             ← get_einvoice_pdf (dispatch)
        ├── viettel.py              ← issue, cancel, get_by_uuid (whitelist)
        └── vnpt.py                 ← issue, cancel, test (whitelist)
```

---

## 3. Cấu Hình Hệ Thống

### 3.1 Nơi Lưu Cấu Hình

Cấu hình E-Invoice **lưu trên DocType `Company`**, trong custom tab `E-Invoice Settings`.

| Field Name            | Loại      | Mô Tả                                              |
|-----------------------|-----------|----------------------------------------------------|
| `use_einvoice`        | Check     | Bật/tắt tính năng hóa đơn điện tử                 |
| `provider`            | Select    | `Viettel` / `VNPT` / `MISA`                        |
| `tax_code`            | Data      | MST công ty (bắt buộc)                             |
| `username`            | Data      | Tên đăng nhập API                                  |
| `password`            | Password  | Mật khẩu API (mã hóa Frappe)                       |
| `api_endpoint_url`    | Data      | URL gốc của SInvoice / VNPT                        |
| `invoice_pattern`     | Data      | Mẫu số hóa đơn (`templateCode` / `KHMSHDon`)      |
| `invoice_serial`      | Data      | Ký hiệu hóa đơn (`invoiceSeries` / `KHHDon`)      |
| `viettel_auth_method` | Select    | `Token (POST /auth/login)` hoặc `Basic Auth`       |
| `account`             | Data      | VNPT: Tài khoản nhân viên (Account)                |
| `acpass`              | Password  | VNPT: Mật khẩu nhân viên (ACpass)                  |
| `invoice_lookup_url`  | Data      | URL cổng tra cứu PDF (VNPT fallback)               |

### 3.2 Cách Đọc Cấu Hình Trong Code

**Viettel:**
```python
from mbwnext_einvoice.integrations.viettel.config import get_active_viettel_config

config = get_active_viettel_config(company="Tên Công Ty", throw=True)
# config.tax_code, config.username, config.api_endpoint_url, ...
```

**VNPT:**
```python
from mbwnext_einvoice.integrations.vnpt.config import get_active_vnpt_config

config = get_active_vnpt_config(company="Tên Công Ty", throw=True)
```

- Nếu `throw=True`, ném `ViettelConfigNotFoundError` / `VNPTConfigNotFoundError` khi không tìm thấy cấu hình hợp lệ.
- Mật khẩu được đọc qua `frappe.utils.password.get_decrypted_password(...)`.

---

## 4. Custom Fields trên Sales Invoice

Định nghĩa trong `fixtures/custom_field.json`, module `MBWNext-Einvoice`.

| Field Name              | Loại         | Mô Tả                                           |
|-------------------------|--------------|-------------------------------------------------|
| `custom_tab_9`          | Tab Break    | Tab "E-Invoice Info"                            |
| `created_einvoice`      | Check        | Đã tạo hóa đơn điện tử hay chưa                |
| `einvoice_no`           | Data         | Số hóa đơn (do NCC cấp)                        |
| `einvoice_uuid`         | Data         | Mã định danh (Viettel: `transactionUuid`; VNPT: `fkey`) |
| `einvoice_pdf_preview`  | HTML         | Khung hiển thị PDF inline (iframe)              |

> **Quan trọng:** Sau khi thay đổi `custom_field.json`, chạy:
> ```bash
> bench --site <site> import-fixtures --app mbwnext_einvoice
> ```

---

## 5. Luồng Phát Hành Hóa Đơn — Viettel

### 5.1 Sơ Đồ Luồng

```
[Người dùng bấm "Create Viettel E-Invoice"]
       │
       ▼
sales_invoice.js → _send_viettel(frm)
       │
       ▼
frappe.call → api/viettel.py → issue_invoice_for_sales_invoice(sales_invoice)
       │
       ▼
viettel_service.issue_invoice_from_sales_invoice(sales_invoice_name)
       │
       ├── 1. Gọi get_active_viettel_config(company) → ViettelConfig
       ├── 2. Gọi build_invoice_payload(si_name) → (payload_dict, transaction_uuid)
       ├── 3. Gọi client.create_invoice(tax_code, payload)
       │       ├── Nếu lỗi MST người mua → thử lại với skip_buyer_tax_code=True
       │       └── Nếu lỗi khác → ném ViettelHTTPError
       └── 4. Parse kết quả: invoiceNo, transactionID, reservationCode
       │
       ▼
api/viettel.py → frappe.db.set_value(SI, {
    created_einvoice: 1,
    einvoice_no: invoice_no,
    einvoice_uuid: transaction_uuid,
})
       │
       ▼
[Thông báo thành công + reload form]
```

### 5.2 Xác Thực Viettel (Authentication)

Viettel hỗ trợ 2 phương thức, chọn trong trường `viettel_auth_method`:

**Token (mặc định):**
```
POST {base_url_gốc}/auth/login
Body: {"username": "...", "password": "..."}
→ access_token
→ Gửi mọi request tiếp theo với Cookie: access_token=<token>
```

**Basic Auth:**
```
Authorization: Basic base64(username:password)
```

> **Lưu ý URL Auth:** Endpoint `/auth/login` nằm ở **root domain**, không nằm trong đường dẫn API chính. Code tự tách `base_url` và lấy `scheme://netloc/auth/login`.

### 5.3 Cấu Trúc Payload Gửi Viettel

```json
{
  "generalInvoiceInfo": {
    "transactionUuid": "<uuid4>",
    "invoiceType": "1",
    "templateCode": "1/00189",
    "invoiceSeries": "K25TBA",
    "invoiceIssuedDate": 1711929600000,
    "currencyCode": "VND",
    "exchangeRate": 1.0,
    "adjustmentType": "1",
    "paymentStatus": true,
    "cusGetInvoiceRight": true
  },
  "sellerInfo": {
    "sellerTaxCode": "0123456789",
    "sellerLegalName": "Công Ty ABC",
    "sellerAddressLine": "123 Đường XYZ, Quận 1, TP.HCM",
    "sellerPhoneNumber": "0901234567",
    "sellerBankName": "Vietcombank",
    "sellerBankAccount": "1234567890"
  },
  "buyerInfo": {
    "buyerName": "Nguyễn Văn A",
    "buyerLegalName": "Nguyễn Văn A",
    "buyerTaxCode": "0987654321",
    "buyerAddressLine": "456 Đường ABC",
    "buyerPhoneNumber": "0909090909",
    "buyerEmail": "a@example.com"
  },
  "payments": [{ "paymentMethodName": "CK" }],
  "itemInfo": [...],
  "summarizeInfo": {
    "totalAmountWithoutTax": 1000000,
    "totalTaxAmount": 100000,
    "totalAmountWithTax": 1100000,
    "totalAmountWithTaxInWords": "Một triệu một trăm nghìn đồng"
  },
  "taxBreakdowns": [{ "taxPercentage": 10, "taxableAmount": 1000000, "taxAmount": 100000 }]
}
```

### 5.4 Xử Lý Trường Hợp MST Người Mua Không Hợp Lệ

Viettel có thể từ chối MST người mua (`INVALID_BUYER_TAX_CODE`). Code xử lý tự động:

```python
try:
    response = client.create_invoice(config.tax_code, payload)
except ViettelHTTPError as exc:
    if "INVALID_BUYER_TAX_CODE" in str(exc):
        # Thử lại không gửi buyerTaxCode
        payload, transaction_uuid = build_invoice_payload(si_name, skip_buyer_tax_code=True)
        response = client.create_invoice(config.tax_code, payload)
```

Kết quả trả về có `buyer_tax_code_omitted: True` để UI hiển thị cảnh báo.

### 5.5 Xử Lý Hóa Đơn Bất Đồng Bộ (Async)

Viettel đôi khi trả về `invoiceNo` rỗng (xử lý async). Khi đó:
- Log cảnh báo vào `Error Log`.
- Lưu `einvoice_uuid = transactionUuid` để tra cứu sau.
- UI gợi ý người dùng "Check status now?" → gọi `get_invoice_by_transaction_uuid`.

```python
# api/viettel.py
def get_invoice_by_transaction_uuid(transaction_uuid, company=None):
    # Gọi client.get_invoice_by_transaction_uuid(transactionUuid)
    # Endpoint: GET /InvoiceAPI/InvoiceWS/searchInvoiceByTransactionUuid
```

---

## 6. Luồng Phát Hành Hóa Đơn — VNPT

### 6.1 Giao Thức

VNPT sử dụng **SOAP/ASMX** với 3 service:
- `publishservice.asmx` — phát hành hóa đơn
- `businessservice.asmx` — hủy hóa đơn
- `portalservice.asmx` — tải PDF, tra cứu

VNPT dùng **2 bộ tài khoản** khác nhau:
- `Account / ACpass` — tài khoản nhân viên (dùng trong các thao tác hủy, điều chỉnh)
- `username / password` — tài khoản ServiceRole (dùng trong mọi lệnh gọi)

### 6.2 Chuẩn XML TT78

Hóa đơn VNPT xây theo chuẩn Thông tư 78/2021:

```xml
<DSHDon>
  <HDon>
    <key>{fkey = tên Sales Invoice}</key>
    <DLHDon>
      <TTChung>...</TTChung>
      <NDHDon>
        <NBan>Bên bán</NBan>
        <NMua>Bên mua</NMua>
        <DSHHDVu>
          <HHDVu>...</HHDVu>
        </DSHHDVu>
        <TToan>Tổng hợp thanh toán</TToan>
      </NDHDon>
    </DLHDon>
  </HDon>
</DSHDon>
```

`fkey` = tên `Sales Invoice` (dùng làm khóa duy nhất trên VNPT).

### 6.3 Lưu Kết Quả

Sau phát hành, VNPT trả về chuỗi dạng:
```
OK:1/001;C26TTA-ACC-2024-001_1234,ACC-2024-002_1235
```

Parse thành: `fkey → einvoice_uuid`, `num → einvoice_no`.

---

## 7. Luồng Hủy Hóa Đơn

### 7.1 Hủy Thủ Công (Từ Form Sales Invoice)

```
[Nút "Cancel Viettel E-Invoice" / "Cancel VNPT E-Invoice"]
       │
       ▼
Prompt nhập Lý do hủy (Cancellation Reason)
       │
       ▼
Confirm dialog
       │
       ▼
frappe.call → api/viettel.cancel_invoice_for_sales_invoice(si, reason)
           hoặc api/vnpt.cancel_invoice_for_sales_invoice(si)
       │
       ▼
service.cancel_invoice(invoice_no, reason, company)  [Viettel]
service.cancel_invoice_by_fkey(fkey, company)        [VNPT]
       │
       ▼
frappe.db.set_value(SI, {
    created_einvoice: 0,
    einvoice_no: "",
    einvoice_uuid: "",
})
       │
       ▼
[Thông báo thành công + reload form]
```

**Điều kiện để hủy được:**
- `created_einvoice = 1`
- `einvoice_no` (Viettel) hoặc `einvoice_uuid` (VNPT) không rỗng

### 7.2 Hủy Tự Động Khi Cancel Sales Invoice (Auto-Cancel Hook)

Khi người dùng bấm **Cancel** trên Sales Invoice trong ERPNext, hệ thống tự động hủy hóa đơn điện tử tương ứng.

**Cơ chế:**

`hooks.py`:
```python
doc_events = {
    "Sales Invoice": {
        "on_cancel": "mbwnext_einvoice.controllers.python.sales_invoice.auto_cancel_einvoice_on_sales_invoice_cancel",
    }
}
```

`controllers/python/sales_invoice.py`:
```python
def auto_cancel_einvoice_on_sales_invoice_cancel(doc, method=None):
    if not doc.created_einvoice:
        return  # Chưa phát hành → bỏ qua

    use_einvoice, provider = frappe.db.get_value("Company", doc.company, ["use_einvoice", "provider"])
    if not use_einvoice or not provider:
        return

    if provider == "VNPT":
        fkey = doc.einvoice_uuid
        vnpt_service.cancel_invoice_by_fkey(fkey=fkey, company=doc.company, fallback_fkeys=[doc.name])
    elif provider == "Viettel":
        invoice_no = doc.einvoice_no
        reason = f"Auto cancel from ERPNext Sales Invoice {doc.name}"
        viettel_service.cancel_invoice(invoice_no=invoice_no, reason=reason, company=doc.company)
```

> **Hành vi khi lỗi:** Nếu hủy bên nhà cung cấp thất bại → `frappe.throw(...)` → ERPNext **KHÔNG** hoàn tất việc hủy SI, giúp đảm bảo đồng bộ dữ liệu.

### 7.3 Endpoint Hủy Viettel

```
POST /api/InvoiceAPI/InvoiceWS/cancelTransactionInvoice/{taxCode}
Body: { "invoiceNo": "...", "additionalReferenceDesc": "Lý do hủy" }
```

### 7.4 Endpoint Hủy VNPT

```
SOAP businessservice.asmx → cancelInv(Account, ACpass, Fkey, userName, userPass)
```

Có fallback `cancel_inv_no_pay` nếu cần.

---

## 8. Luồng Xem PDF Hóa Đơn

### 8.1 Preview Inline (Iframe)

Hiển thị ngay trên tab `E-Invoice Info` của Sales Invoice:

```
[refresh form, docstatus=1, created_einvoice=1]
       │
       ▼
_load_einvoice_pdf_preview(frm)
       │
       ▼
frappe.call → api/einvoice.get_einvoice_pdf(sales_invoice)
       │
       ├── Lấy company → provider từ Company
       ├── Nếu Viettel → _get_viettel_pdf(si_doc) → client.get_invoice_representation_file()
       └── Nếu VNPT → _get_vnpt_pdf(si_doc) → client.download_new_inv_pdf_fkey() + fallback
       │
       ▼
Trả về { pdf_base64: "..." } hoặc { portal_url: "..." }
       │
       ▼
JS: atob(pdf_base64) → Uint8Array → Blob → URL.createObjectURL → iframe.src
```

**Caching:** Dùng `frm._einvoice_preview_cache_key` (kết hợp `name|modified|einvoice_no|einvoice_uuid`) để tránh gọi API lại khi form chưa thay đổi.

### 8.2 Mở PDF Tab Mới

Nút `View > E-Invoice` → `_view_einvoice_pdf(frm)` → Blob URL → `window.open(url, "_blank")`.

### 8.3 VNPT: Thứ Tự Thử Tải PDF

```python
# Thử theo thứ tự:
1. client.download_new_inv_pdf_fkey(fkey)     # Hóa đơn mới nhất
2. client.download_inv_pdf_fkey_no_pay(fkey)  # Không kiểm tra thanh toán
3. client.download_inv_pdf_fkey(fkey)         # Đã thanh toán
```

---

## 9. Giao Diện Người Dùng (UI)

### 9.1 Nút Động Trên Sales Invoice

Logic trong `sales_invoice.js` → `refresh(frm)`:

```
Điều kiện: docstatus === 1 && status !== "Cancelled"
       │
       ├── Lấy provider + use_einvoice từ Company
       │
       ├── created_einvoice = 0:
       │     ├── VNPT   → [E-Invoice > Create VNPT E-Invoice]
       │     ├── Viettel → [E-Invoice > Create Viettel E-Invoice]
       │     └── MISA   → [E-Invoice > Create MISA E-Invoice]
       │
       └── created_einvoice = 1:
             ├── [View > E-Invoice]               ← Mở PDF tab mới
             ├── VNPT    → [E-Invoice > Cancel VNPT E-Invoice]
             └── Viettel → [E-Invoice > Cancel Viettel E-Invoice]
```

### 9.2 Hàm Hủy Có Lý Do (Viettel)

Sử dụng `frappe.prompt(...)` để nhập lý do trước khi confirm:

```javascript
frappe.prompt([{ fieldname: "reason", label: "Cancellation Reason", fieldtype: "Small Text", reqd: 1 }],
    (values) => {
        frappe.confirm(`Cancel invoice ${frm.doc.einvoice_no}?`, () => {
            frappe.call({ method: "...", args: { sales_invoice, reason } })
        })
    }
)
```

### 9.3 Hiển Thị PDF Inline

```javascript
// Decode Base64 → Blob → URL
function _einvoice_pdf_base64_to_blob_url(b64) {
    const raw = b64.replace(/\s/g, "").replace(/-/g, "+").replace(/_/g, "/");
    const blob = new Blob([Uint8Array.from(atob(raw), c => c.charCodeAt(0))], { type: "application/pdf" });
    return URL.createObjectURL(blob);
}
```

> **Quan trọng:** Dùng `fld.set_value(html)` thay vì `fld.html(html)` để Frappe's `ControlHTML` không ghi đè khi form refresh.

---

## 10. Xử Lý Dữ Liệu — Bên Bán & Bên Mua

### 10.1 Địa Chỉ Đơn Vị Bán (Seller)

Hàm `_seller_address_from_company(company_name)` — ưu tiên theo thứ tự:

1. Address gắn với `Company` có cờ `is_billing_address = 1` (Preferred Billing Address)
2. Address gắn với `Company` có cờ `is_primary_address = 1`
3. Address có `is_your_company_address = 1`
4. Fallback: `city + country` trên Company

### 10.2 Ngân Hàng Đơn Vị Bán (Seller Bank)

Hàm `_seller_bank_from_si(si)` — ưu tiên:

1. Custom field trên SI: `custom_bank` (tên ngân hàng) + `custom_bank_account_no` (số tài khoản)
2. Fallback: `Bank Account` đầu tiên của `Company` có `is_company_account = 1`

### 10.3 Địa Chỉ & Điện Thoại Người Mua (Buyer)

Hàm `_buyer_address_and_mobile(si)` — ưu tiên:

| Trường       | Nguồn ưu tiên                                                                       |
|--------------|-------------------------------------------------------------------------------------|
| Địa chỉ      | Address gắn `Customer` có `is_billing_address = 1` → `is_primary_address` → fallback SI |
| Điện thoại   | `customer_primary_contact.mobile_no` → Contact gắn Customer có `mobile_no` → fallback SI |

### 10.4 Tên Địa Chỉ (Address Title)

Hàm `_address_text_with_title(address_name)`:

```python
title = frappe.db.get_value("Address", address_name, "address_title")
if title:
    return title.strip()  # Ưu tiên Address Title
# Fallback: get_address_display → strip HTML
return _strip_html(get_address_display(address_name))
```

### 10.5 MST Người Mua (Buyer Tax Code)

Hàm `_normalize_buyer_tax_code(value)` — chỉ chấp nhận:
- `10 chữ số`: `0123456789`
- `10 chữ số + "-" + 3 chữ số`: `0123456789-001`

Nếu sai định dạng → trả về `""` → không gửi `buyerTaxCode`.

### 10.6 Thuế Suất Dòng Hàng

Đọc từ `item_tax_rate` (JSON) theo từng dòng. Nếu không có → fallback về thuế suất cấp hóa đơn (`si.taxes[0].rate`).

---

## 11. Cơ Chế Tự Động (Hooks)

### 11.1 `doc_events` — Auto Cancel

```python
# hooks.py
doc_events = {
    "Sales Invoice": {
        "on_cancel": "mbwnext_einvoice.controllers.python.sales_invoice.auto_cancel_einvoice_on_sales_invoice_cancel",
    }
}
```

### 11.2 `doctype_js` — Gắn JS

```python
doctype_js = {
    "Sales Invoice": "controllers/js/sales_invoice.js",
    "Company": "controllers/js/company.js",
}
```

### 11.3 `fixtures` — Đồng Bộ Custom Fields

```python
fixtures = [
    {
        "doctype": "Custom Field",
        "filters": [["module", "in", ("MBWNext-Einvoice")]]
    }
]
```

---

## 12. Xử Lý Lỗi & Debugging

### 12.1 Phân Cấp Lỗi

| Exception               | Ý Nghĩa                                               |
|-------------------------|-------------------------------------------------------|
| `ViettelHTTPError`      | HTTP error (4xx/5xx) từ SInvoice REST API             |
| `ViettelAPIError`       | Lỗi nghiệp vụ (errorCode ≠ 0/200) trong body JSON    |
| `ViettelConfigNotFoundError` | Chưa cấu hình hoặc chưa bật E-Invoice cho Company |
| `VNPTSOAPError`         | VNPT trả về `ERR:x` trong chuỗi kết quả SOAP         |
| `VNPTHTTPError`         | HTTP error từ ASMX service                            |

### 12.2 Mã Lỗi VNPT Thường Gặp

| Mã      | Ý Nghĩa                                                  |
|---------|----------------------------------------------------------|
| `ERR:1` | Tài khoản sai hoặc không có quyền                        |
| `ERR:2` | fkey/token không đúng hoặc không tìm thấy hóa đơn       |
| `ERR:5` | Lỗi hệ thống (DB rollback)                               |
| `ERR:8` | Hóa đơn đã bị điều chỉnh/hủy/thay thế                   |
| `ERR:11`| Hóa đơn chưa thanh toán                                  |
| `ERR:13`| Trùng fkey — hóa đơn đã xử lý trước đó                  |

### 12.3 Lỗi Xác Thực Viettel 401

Hàm `_friendly_viettel_error(exc)` chuyển lỗi kỹ thuật thành thông báo thân thiện:

```python
if "401" in message and ("invalid_grant" in message or "Invalid User Name" in message):
    return "Viettel authentication failed: Invalid username/password. ..."
```

### 12.4 Kiểm Tra Kết Nối

- **Viettel:** Nút `Test Viettel Connection` trên Company → gọi `client.authenticate()` (Token) hoặc trả về thông báo (Basic Auth).
- **VNPT:** Nút `Test VNPT Connection` → gọi `client.get_cert_info()` → trả về thông tin chứng thư số.

### 12.5 Nơi Xem Log Lỗi

Truy cập: **ERPNext → Settings → Error Log**

Các title log thường gặp:
- `Viettel: issue invoice failed`
- `Viettel: cancel invoice for sales invoice failed`
- `VNPT: issue invoice failed`
- `VNPT: cancel invoice failed`
- `Auto cancel e-invoice on Sales Invoice cancel failed`
- `Viettel SInvoice: async invoice (no invoiceNo yet)`

---

## 13. Bản Dịch (i18n)

File bản dịch tiếng Việt:
```
apps/mbwnext_localization/mbwnext_localization/locale/vi.po
```

Tất cả chuỗi liên quan E-Invoice được thêm ở cuối file theo chuẩn `.po`:
```po
msgid "Cancellation Reason"
msgstr "Lý do hủy"

msgid "Viettel e-invoice cancelled successfully."
msgstr "Hủy hóa đơn điện tử Viettel thành công."
```

Sau khi sửa file `.po`, cần **build bản dịch**:
```bash
bench --site <site> build --force
```
hoặc restart Frappe để áp dụng.

---

## 14. Lưu Ý Quan Trọng & Best Practice

### ⚠️ Không dùng `si.save()` để cập nhật trạng thái

```python
# SAI — sẽ trigger validate lại, có thể lỗi với POS Opening Shift đã xóa
si.created_einvoice = 1
si.save()

# ĐÚNG — cập nhật thẳng vào DB, không trigger validate
frappe.db.set_value("Sales Invoice", si_name, {"created_einvoice": 1, ...}, update_modified=False)
frappe.db.commit()
```

### ⚠️ fkey VNPT = tên Sales Invoice

`fkey` được dùng làm khóa định danh hóa đơn trên hệ thống VNPT. Giá trị này chính là tên (name) của Sales Invoice trong ERPNext. Nếu `einvoice_uuid` trên SI bị mất, có thể thử dùng `si.name` làm fallback.

### ⚠️ transactionUuid Viettel là UUID mới mỗi lần

Mỗi lần gọi `build_invoice_payload(...)` sẽ sinh `uuid4()` mới. Nếu phát hành lại cùng SI (sau khi reset), Viettel sẽ nhận đây là một giao dịch hoàn toàn mới.

### ⚠️ Hủy SI sẽ tự hủy E-Invoice

Khi Cancel Sales Invoice, hook `on_cancel` tự động gọi hủy bên VNPT/Viettel. Nếu hủy bên NCC thất bại → ERPNext sẽ **không cho hủy SI** (throw lỗi). Hãy đảm bảo thông tin kết nối NCC còn hoạt động trước khi Cancel SI.

### ✅ Thứ tự ưu tiên khi lấy thông tin địa chỉ

```
Preferred Billing Address → Primary Address → Fallback khác
```

Luôn đọc `address_title` trước, nếu có thì dùng luôn (không cần gọi `get_address_display`).

### ✅ Luôn log lỗi kỹ thuật trước khi throw

```python
frappe.log_error(message=frappe.as_json({...}), title="Context: action failed")
frappe.throw(msg=str(exc), exc=type(exc))
```

### ✅ Kiểm tra fixtures sau khi thêm custom field mới

```bash
bench --site <site> import-fixtures --app mbwnext_einvoice
bench --site <site> clear-cache
```

---

## Phụ Lục — Bảng Đối Chiếu Field

### Viettel SInvoice vs ERPNext

| Viettel Field               | Nguồn ERPNext                                |
|-----------------------------|----------------------------------------------|
| `transactionUuid`           | `uuid.uuid4()` mới mỗi lần                   |
| `templateCode`              | `Company.invoice_pattern`                    |
| `invoiceSeries`             | `Company.invoice_serial`                     |
| `sellerTaxCode`             | `Company.tax_code`                           |
| `sellerLegalName`           | `Company.company_name`                       |
| `sellerAddressLine`         | Address gắn Company (billing → primary)      |
| `sellerBankName`            | `SI.custom_bank` → `Bank Account.bank`       |
| `sellerBankAccount`         | `SI.custom_bank_account_no` → `Bank Account.bank_account_no` |
| `buyerName`                 | `SI.customer_name`                           |
| `buyerTaxCode`              | `SI.tax_id` → `Customer.tax_id`              |
| `buyerAddressLine`          | Address gắn Customer (billing → primary)     |
| `buyerPhoneNumber`          | `customer_primary_contact.mobile_no`         |
| `invoiceIssuedDate`         | `SI.posting_date` → millisecond UTC          |
| `totalAmountWithoutTax`     | `SI.net_total`                               |
| `totalTaxAmount`            | `SI.total_taxes_and_charges`                 |
| `totalAmountWithTax`        | `SI.grand_total`                             |
| `totalAmountWithTaxInWords` | `SI.in_words`                                |

### VNPT TT78 vs ERPNext

| VNPT Field   | Nguồn ERPNext                                |
|--------------|----------------------------------------------|
| `<key>`      | `SI.name` (tên Sales Invoice)                |
| `DVTTe`      | `SI.currency`                                |
| `HTTToan`    | `SI.mode_of_payment` (map CK/TM/TM-CK)      |
| `MST` (NBan) | `Company.tax_code`                           |
| `DChi` (NBan)| Address gắn Company                          |
| `MST` (NMua) | `SI.tax_id`                                  |
| `DChi` (NMua)| Address gắn Customer                         |
| `TgTCThue`   | `SI.net_total`                               |
| `TgTThue`    | `SI.total_taxes_and_charges`                 |
| `TgTTTBSo`   | `SI.grand_total`                             |
| `TgTTTBChu`  | `SI.in_words`                                |

---

*Tài liệu được tổng hợp từ codebase thực tế của module `mbwnext_einvoice`. Cập nhật lần cuối: Tháng 3/2026.*
