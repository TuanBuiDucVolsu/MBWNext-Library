# Tổng hợp các chỉnh sửa cho tương thích v16 (bench `testv16`)

> Phạm vi: các thay đổi code thực tế đã áp dụng vào source trong `/home/pc/testv16/apps/` khi rà soát từng app custom của MBWNext để tương thích với Frappe v16.25.0 / ERPNext v16.26.2. Đã kiểm tra 8/12 app; mỗi app đều được xác nhận bằng `bench install-app` + `bench migrate` thật trên site test `mbw.com`, không chỉ đọc code tĩnh.

---

## 1. `mbwnext_advanced_selling`

| File | Thay đổi | Lý do |
|---|---|---|
| `hooks.py` | `override_doctype_class` → `extend_doctype_class` (mapping `Item` → `ItemOverride`) | Frappe v16 đổi tên hook key này; key cũ bị bỏ qua âm thầm (không lỗi, chỉ mất tác dụng) |
| `controllers/python/discontinued_product.py` | `_company_from_pos_profile()`: thêm bước `json.loads()` khi `pos_profile` là chuỗi JSON, trước khi lấy `name`/`pos_profile` | Đồng nhất với cách `pos_next.api.items.search_by_barcode`/`get_item_details` tự parse `pos_profile`; tránh việc logic khoá kinh doanh sản phẩm bị bỏ qua âm thầm khi `pos_profile` không phải dict thuần |
| `install/after_install.py` | `rebuild_tree("Customer Group", "parent_customer_group")` → `rebuild_tree("Customer Group")`; tương tự bỏ tham số `parent_field` trong hàm dùng chung `_create_nested_set_from_json()` | Frappe v16 đổi chữ ký `frappe.utils.nestedset.rebuild_tree()`, bỏ hẳn tham số `parent_field` (v15 vẫn nhận nhưng ghi chú "sẽ bị xoá ở v16+") — gọi thừa tham số gây `TypeError` |

## 2. `mbwnext_advanced_buying`

Không cần sửa gì — `hooks.py` không dùng `override_doctype_class`/`doc_events`/`scheduler_events`; xuất Excel vốn đã dùng `openpyxl` trực tiếp (không đụng `xlsxutils`); không có `rebuild_tree` hay tham chiếu doctype đã xoá. Cài đặt + migrate thành công không cần chỉnh sửa.

## 3. `mbwnext_advanced_stock`

| File | Thay đổi | Lý do |
|---|---|---|
| `report/summary_of_inventory_on_hand/summary_of_inventory_on_hand.py` | Viết lại `get_opening_from_closing_balance()`: query `Stock Closing Entry` (header) + `Stock Closing Balance` (child) thay vì `Closing Stock Balance`; map `actual_qty`→`bal_qty`, `stock_value`→`bal_val` | ERPNext v16 xoá doctype `Closing Stock Balance`, tách thành 2 doctype mới; `get_prepared_data()` cũ giờ đọc từ file đính kèm nén gzip nên không dùng được nữa |
| `report/summary_of_on_hand_inventory_on_multiple_warehouse/...py` | Sửa giống hệt trên | nt |
| `report/summary_of_on_hand_inventory_on_multiple_warehouse_cross_reference/...py` | Sửa đoạn code đang active giống hệt trên (các đoạn tương tự khác trong file đã bị comment sẵn) | nt |
| `utils/stock_validation.py` | Viết lại `_violations_html()`: bỏ hết `<th>`, `<small>`, thuộc tính `style=`/`class=`; chỉ dùng `<table><thead><tbody><tr><td><b>` | Frappe v16 xiết chặt sanitize HTML trong `frappe.throw()`/`msgprint()` (`nh3.clean`, chỉ giữ `div,p,br,ul,ol,li,strong,b,em,i,u,table,thead,tbody,td,tr,a`, xoá mọi attribute) — đã verify bằng `nh3.clean` thật, HTML cũ bị vỡ định dạng (mất `<th>`, mất style căn lề/tô màu) |

## 4. `mbwnext_advanced_accounting`

| File | Thay đổi | Lý do |
|---|---|---|
| `hooks.py` | `override_doctype_class` → `extend_doctype_class` (mapping `Sales Invoice` → `SalesInvoiceOverride`) | Cùng lý do mục 1 — bật lại tính năng đồng bộ status Sales Invoice ↔ Sales Voucher |
| `controllers/python_hook/company.py:140` | `rebuild_tree("Account", "parent_account")` → `rebuild_tree("Account")` | Cùng lý do đổi chữ ký `rebuild_tree()` ở mục 1 |
| `.../doctype/payment_entry_group/payment_entry_group.py` | 3 chỗ gọi `frappe.get_hooks("advance_payment_doctypes")` → `erpnext.accounts.utils.get_advance_payment_doctypes()` (đã thêm vào import) | Frappe v16 tách hook `advance_payment_doctypes` thành `advance_payment_receivable_doctypes`/`advance_payment_payable_doctypes`; gọi hook cũ trả về rỗng, làm sai lệch logic tỷ giá/GL khi payment tham chiếu Sales/Purchase Order |
| `.../doctype/payment_entry_group/payment_entry_group.py` | Xoá import chết `from erpnext.accounts.doctype.tax_withholding_category.tax_withholding_category import get_party_tax_withholding_details` | Hàm này đã bị xoá hoàn toàn khỏi ERPNext v16 (thay bằng class `PaymentTaxWithholding` mới). Hàm duy nhất dùng import này — `set_tax_withholding()` — vốn đã là dead code, bị comment sẵn ở cả bản v15 production lẫn v16, chưa từng chạy thật. Theo quyết định của người dùng: chỉ xoá import để module load được, không viết lại logic TDS đang tắt |

**Sự cố phụ đã xử lý (không phải sửa code):** 2 lần cài đặt thất bại giữa chừng để lại rác trong DB site test — bản ghi `Module Def "MBWNext Advanced Accounting"` trùng khoá và `Workspace MBWNext POS` tạo dở dang. Đã dọn bằng `frappe.delete_doc(..., force=True)` trước khi cài lại thành công.

## 5. `mbwnext_localization`

| File | Thay đổi | Lý do |
|---|---|---|
| `.../doctype/mbwnext_navbar/mbwnext_navbar.py` | `from frappe.config import get_modules_from_all_apps_for_user` → `from frappe.utils.modules import get_modules_from_all_apps_for_user` | Package `frappe/config` bị xoá hoàn toàn ở Frappe v16; hàm dời sang `frappe.utils.modules`, giữ nguyên tên và chữ ký |
| `fixtures/workspace.json` | Thêm `"type": "Workspace"` vào cả 57 bản ghi Workspace | Frappe v16 thêm field `type` bắt buộc (Select: Workspace/Link/URL, mặc định "Workspace") trên doctype `Workspace`; import fixture không tự áp default cho field thiếu nên bị `MandatoryError` |

**Sự cố phụ đã xử lý:** dọn rác `Module Def "MBWNext Localization"` và `Workspace "MBWNext POS"` tạo dở từ các lần cài lỗi trước, trước khi cài lại thành công.

## 6. `mbwnext_advanced_distribution_map`

Không cần sửa gì — app sạch nhất, không dùng bất kỳ hook/API nào bị ảnh hưởng bởi v16. Cài đặt + migrate thành công ngay từ lần đầu.

## 7. `mbwnext_econtract_service`

Không cần sửa gì — kể cả phần `page_renderer`/`website_path_resolver`/`auth_hooks` tự viết cho WOPI/Collabora Online đều khớp đúng interface Frappe v16 hiện tại. Cài đặt + migrate thành công ngay từ lần đầu.

## 8. `mbwnext_einvoice`

| File | Thay đổi | Lý do |
|---|---|---|
| `.../workspace/e_invoice/e_invoice.json` | Thêm `"type": "Workspace"` | Cùng lỗi field bắt buộc mới như mục 5, nhưng lần này nằm trong workspace bundle theo module (tự động sync khi cài/migrate) chứ không phải qua `fixtures` |

---

## Chưa kiểm tra (còn lại trong `vnpost/apps` chưa port sang `testv16`)

`builder`, `print_designer`, `super_admin`, `vnpost_sso` — cần `bench get-app` về `testv16` trước khi kiểm tra theo cùng quy trình (đọc CLAUDE.md/hooks.py → rà lỗi tĩnh theo checklist v16 → `bench install-app` + `bench migrate` thật để xác nhận).

## Checklist các lỗi v16 đã dùng để rà từng app

1. `override_doctype_class` → phải đổi thành `extend_doctype_class`.
2. `frappe.utils.nestedset.rebuild_tree(doctype, parent_field)` → bỏ tham số thứ 2.
3. `frappe.get_hooks("advance_payment_doctypes")` → dùng `erpnext.accounts.utils.get_advance_payment_doctypes()`.
4. `frappe.utils.xlsxutils.make_xlsx` → nên chuyển sang `openpyxl` trực tiếp.
5. `from frappe.config import ...` → package đã xoá, hàm dời sang `frappe.utils.modules` (hoặc nơi khác tuỳ hàm).
6. Doctype `Closing Stock Balance` → tách thành `Stock Closing Entry` + `Stock Closing Balance`.
7. Doctype/API đã xoá hẳn: `Advance Tax`, `Homepage`, `BOM/Job Card Scrap Item`, `SMS Log` (dời vào Frappe core), `Transaction Log`, `get_party_tax_withholding_details` (thay bằng `PaymentTaxWithholding`).
8. HTML trong `frappe.throw()`/`msgprint()` bị sanitize chặt hơn (`nh3.clean`) — chỉ còn `div,p,br,ul,ol,li,strong,b,em,i,u,table,thead,tbody,td,tr,a`, mất mọi attribute.
9. Doctype `Workspace` thêm field bắt buộc `type` (Workspace/Link/URL) — mọi fixture/workspace bundle từ trước v16 phải bổ sung field này.
10. Xác minh trực tiếp bằng `bench install-app` + `bench migrate` — một số lỗi (rebuild_tree, frappe.config, Workspace.type) chỉ lộ ra lúc chạy thật, không bắt được bằng grep tĩnh.
