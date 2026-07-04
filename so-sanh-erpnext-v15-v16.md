# So sánh ứng dụng ERPNext v15.91.3 → v16.26.2

> Phạm vi: diff trực tiếp giữa tag chính thức `v15.91.3` (khớp đúng `__version__` trong `erpnext/__init__.py` của bench production `vnpost`) và `v16.26.2` (bench test `testv16`), lấy từ repo gốc `frappe/erpnext` — cách nhau **11.549 commit**. Đây là báo cáo song song với [so-sanh-frappe-v15-v16.md](so-sanh-frappe-v15-v16.md) (đã so sánh phần framework Frappe); file này chỉ tập trung vào ứng dụng **ERPNext**.

**Lưu ý quan trọng về nguồn diff:** app `erpnext` trong bench production `vnpost` là **fork tuỳ biến của MOBIWORK** (`MOBIWORK/MBW_MarketPlace_ErpNext`, nhánh `version-15-mbwnext`) và lịch sử git đã bị squash về **1 commit duy nhất** — không thể diff trực tiếp fork này với v16 mà tách bạch được đâu là thay đổi chính thức của Frappe/ERPNext, đâu là tuỳ biến riêng của MBW. Do đó báo cáo dưới đây diff giữa **2 bản vanilla chính chủ** (`v15.91.3` ↔ `v16.26.2`) để có kết quả sạch, đáng tin cậy. Điều này có 2 hệ quả:
1. Toàn bộ tuỳ biến MBW (VAT hoá đơn điện tử, tồn kho nâng cao, bán hàng nâng cao...) **chưa được đối chiếu** — cần tự rà soát riêng khi port các app `mbwnext_*` lên nhánh v16.
2. Bench test `testv16` hiện chỉ có `frappe + erpnext + hrms` bản vanilla, chưa cài app custom nào — giống hệt tình trạng đã nêu ở báo cáo Frappe.

---

## 1. Breaking changes chính thức (đánh dấu `!`)

1. `feat!: Item Wise Tax Details Table (#48692)` + `refactor!: store item wise tax details as a more flexible dict` — cách lưu **chi tiết thuế theo từng dòng item** đổi hẳn cấu trúc dữ liệu (doctype con mới `Item Wise Tax Detail` thay cho field JSON thô trước đây). Report/tích hợp custom nào đọc trực tiếp field `item_wise_tax_detail` dạng JSON string sẽ hỏng.
2. `refactor!: remove redundant title field từ sales transactions (#45115)` — bỏ field `title` dư thừa trên các chứng từ bán hàng.
3. `refactor!: remove whitelisted method make_bank_account (#49001)` — API custom nào gọi `frappe.call` tới method này sẽ lỗi 404.
4. `refactor!: switch to creation sort (#40699)` — đổi thứ tự sắp xếp mặc định của danh sách sang theo `creation` thay vì field khác; ảnh hưởng nếu có report/view custom giả định thứ tự cũ.
5. `fix!(portal): remove Homepage` — xoá hẳn tính năng **Homepage/Portal Homepage Section** (xem mục 3).
6. `fix!: don't round unduly` — thay đổi logic làm tròn số ở một số nơi tính toán.
7. `perf!: Avoid updating sales data on every transaction (#51151)` — đổi cơ chế cập nhật số liệu bán hàng (best-seller/last-sold-price...), không còn realtime trên từng transaction.
8. `feat!: configure which rate is used to auto-update price list` — đổi mặc định/hành vi field nào được dùng để tự cập nhật Price List.
9. `chore!: remove unused utils` — dọn dẹp hàm util không dùng, app custom `import` trực tiếp cần kiểm tra lại.

Không có commit nào chứa footer `BREAKING CHANGE:` — danh sách trên (đánh dấu `!` trong tiêu đề) là đầy đủ.

---

## 2. Thay đổi dependency / cấu trúc thư mục

### `pyproject.toml`
- `requires-python`: `>=3.10` → **`>=314`** (đồng bộ với yêu cầu Python 3.14 của Frappe framework).
- Ràng buộc `frappe` dependency: `>=15.40.4,<16.0.0` → `>=16.0.0,<17.0.0`.
- Nâng version: `Unidecode 1.3.6→1.4.0`, `rapidfuzz 3.12.2→3.14.3`, `holidays 0.28→0.87`, `googlemaps` (ghim version `~=4.10.0`), `python-youtube 0.8.0→0.9.8`.
- **Dependency mới:** `mt-940` (parser định dạng sao kê ngân hàng chuẩn MT940 châu Âu), `pdfplumber` (trích bảng dữ liệu từ sao kê PDF, kéo theo `pdfminer.six`, `Pillow`, `pypdfium2`) — phục vụ tính năng **Bank Statement Import** mở rộng (xem mục 3).
- Thêm khối `[tool.bench.assets]` với `build_dir = "./banking"` — khai báo một **sub-project frontend riêng** được build và output vào `erpnext/public/banking` + `erpnext/www/banking.html`.

### `package.json` (root) và thư mục `banking/` mới
- Thêm scripts `postinstall`/`dev`/`build` trỏ vào thư mục con `banking` (chạy `yarn` riêng).
- **`banking/` là một SPA React + TypeScript + Vite hoàn toàn mới** (dùng Tailwind v4, `@tanstack/react-table`, `@tanstack/react-virtual`, `@dnd-kit/*` cho kéo-thả) — lần đầu tiên ERPNext có một module frontend viết bằng React thay vì Vue/vanilla JS như toàn bộ desk còn lại. Các trang chính: `BankReconciliation.tsx`, `BankStatementImporter.tsx`, `BankStatementImporterContainer.tsx`, `ViewBankStatementImportLog.tsx`. Đây thực chất là **giao diện Đối chiếu Ngân hàng (Bank Reconciliation) được viết lại hoàn toàn**, phục vụ qua route mới `/banking/...` (khai báo trong `hooks.py`).

### Cấu trúc thư mục cấp cao nhất `erpnext/`
- **Thêm mới:** `deprecation_dumpster.py` (nơi tập trung các hàm/method deprecated kèm cơ chế cảnh báo theo version, giống cơ chế tương tự bên Frappe core), `desktop_icon`, `gettext`, `locale` (đồng bộ hệ i18n gettext `.po` mới của Frappe v16), `report_center` (xem mục 3), `workspace_sidebar`.
- **Bị xoá:** `translations` (file dịch CSV cũ — đồng bộ với Frappe core).

---

## 3. Doctype thêm / xoá / đổi tên (theo module)

### Kế toán (`accounts`)
- **Mới:** `Account Category` (phân loại tài khoản kế toán mới, có patch migrate dữ liệu tài khoản cũ), `Bank Account Balance`, `Bank Statement Import Log` (+ `Column Map`), `Bank Transaction Rule` (+ `Rule Accounts`, `Rule Description Conditions` — **quy tắc tự động khớp/phân loại giao dịch ngân hàng theo điều kiện mô tả**, chạy định kỳ qua scheduler mới `scheduler_run_rule_evaluation`), `Budget Distribution` (phân bổ ngân sách theo tháng — kèm patch `migrate_budget_records_to_new_structure`), `Financial Report Template` + `Financial Report Row` (**trình xây dựng báo cáo tài chính tuỳ biến** — P&L/Bảng cân đối, hỗ trợ style XLSX, có view tăng trưởng/biên lợi nhuận), `Item Wise Tax Detail` (thay cấu trúc JSON cũ — xem mục 1), `Payment Reference`, `Sales Invoice Reference`, `Tax Withholding Entry` + `Tax Withholding Group` (**làm lại hoàn toàn nghiệp vụ khấu trừ thuế** — TDS/thuế nhà thầu, có patch `migrate_tax_withholding_data`).
- **Xoá:** `Advance Tax` (gộp vào cơ chế Tax Withholding Entry mới), `Repost Accounting Ledger Settings` (gộp vào Accounts Settings — patch `merge_repost_settings_to_accounts_settings`), `Tax Withheld Vouchers`.

### Sản xuất (`manufacturing`)
- **Mới:** `Master Production Schedule` (+ item con) — **kế hoạch sản xuất tổng thể (MPS)** theo warehouse cha/con, dự báo số lượng; `Sales Forecast` (+ item con) — **dự báo bán hàng** làm đầu vào cho MPS; `Workstation Cost` + `Workstation Operating Component` (+ `Account`) — **tính giá thành theo từng công đoạn/trạm làm việc** chi tiết hơn (ràng buộc không âm cho chi phí); `Work Order Additional Item`; `BOM Secondary Item`, `Job Card Secondary Item` (đổi tên/mở rộng khái niệm từ *Scrap Item*, gồm cả Co-Product/By-Product/Additional Finished Good, cho phép số lượng = 0).
- **Xoá:** `BOM Scrap Item`, `Job Card Scrap Item` (thay bằng "Secondary Item" ở trên, phạm vi rộng hơn — không chỉ phế phẩm).
- **Tính năng lớn không có doctype riêng nhưng đáng chú ý:** `feat: Disassembly Order (#42655)` — nghiệp vụ **tháo dỡ thành phẩm ngược lại nguyên vật liệu** (ngược chiều Work Order), có patch `add_disassembly_order_stock_entry_type` thêm loại Stock Entry mới.

### Gia công ngoài (`subcontracting`) — tính năng lớn nhất về mặt nghiệp vụ mới
- **Mới hoàn toàn:** `Subcontracting Inward Order` (+ `Item`, `Received Item`, `Secondary Item`, `Service Item`) — `feat: subcontracting inward (#47728)`. Đây là **chiều ngược của gia công outsourcing truyền thống**: thay vì mình gửi nguyên liệu cho nhà thầu phụ gia công, tính năng này dùng khi **khách hàng/đối tác gửi nguyên vật liệu để mình gia công hộ** (nhận nguyên liệu vào, xuất thành phẩm ra cho khách, tính giá vốn bình quân gia quyền riêng cho nguyên liệu khách cung cấp).

### Bán hàng / Mua hàng (`selling`, `buying`)
- **Mới:** `Delivery Schedule Item`, `Supplier Number At Customer`, `Customer Number At Supplier` (mã số chéo giữa khách hàng và nhà cung cấp — phục vụ đối chiếu B2B).

### Tồn kho (`stock`)
- **Mới:** `Item Lead Time`, `Landed Cost Vendor Invoice`, `Stock Closing Balance` + `Stock Closing Entry` — `refactor: stock closing balance -> stock closing entry (#44489)`, tách "Closing Stock Balance" cũ thành 2 doctype rõ vai trò hơn (entry hành động vs. balance kết quả).
- **Xoá:** `Closing Stock Balance` (đổi tên/tách như trên).

### Portal / Website
- **Xoá hẳn:** `Homepage`, `Homepage Section`, `Homepage Section Card` (`fix!(portal): remove Homepage`) — tính năng trang chủ portal khách hàng tự cấu hình qua ERPNext bị loại bỏ hoàn toàn, có patch `add_portal_redirects` để tự động redirect link cũ.
- **Xoá:** `Print Heading` (setup), `SMS Log` (utilities — dời hẳn về core Frappe, đã xác nhận ở báo cáo Frappe: `frappe/core/doctype/sms_log` là doctype core mới).

---

## 4. Thay đổi cấu trúc `hooks.py` (diff ~300 dòng)

- `app_home = "/desk"` mới, `add_to_apps_screen` sửa lại dùng biến `app_name`/`app_title`/`app_home` thay vì hard-code `/app/home` — đồng bộ đổi URL scheme `/app` → `/desk` của Frappe v16.
- **`override_doctype_class` đổi tên thành `extend_doctype_class`** — nếu bất kỳ app custom nào (kể cả các app `mbwnext_*`) dùng hook key `override_doctype_class` trong `hooks.py` riêng để override class doctype của ERPNext (ví dụ Address), **phải đổi tên key** khi lên v16, nếu không hook sẽ im lặng không có tác dụng.
- Bỏ `web_include_js = "erpnext-web.bundle.js"`; thêm `app_include_icons`/`web_include_icons` (icon POS riêng), thêm `page_js = {"print": "public/js/print.js"}`.
- **Scheduler event tái cấu trúc toàn diện**, khớp với thay đổi tầng Frappe: các bucket `hourly_long`, `daily`, `daily_long` cũ bị dồn/đổi tên thành `hourly_maintenance`, `daily_maintenance` (thêm cả `run_parallel_reposting`, đánh giá `Bank Transaction Rule` định kỳ). App custom có job scheduler đăng ký vào các bucket cũ theo tên cứng cần rà lại.
- `advance_payment_doctypes` (1 biến gộp) → tách thành **`advance_payment_receivable_doctypes`** và **`advance_payment_payable_doctypes`** — code custom đọc biến cũ `advance_payment_doctypes` sẽ lỗi `ImportError`.
- **`erpnext.regional.create_transaction_log` bị xoá khỏi hook `on_submit` của Sales Invoice và Payment Entry** — khớp với việc doctype `Transaction Log` (audit hash-chain) bị xoá hoàn toàn ở tầng Frappe core.
- Hook `default_roles` (tự gán role Customer/Supplier cho Contact) **bị xoá hoàn toàn** — cần kiểm tra nếu vnpost có phụ thuộc cơ chế tự động này.
- `before_install = ["erpnext.setup.install.check_frappe_version"]` bị xoá — không còn tự kiểm tra tương thích version Frappe lúc cài đặt qua hook này (ràng buộc version giờ nằm ở `pyproject.toml`/dependency resolver của bench).
- `setup_wizard_complete`, `setup_wizard_test`, `before_tests` bị xoá (liên quan demo/test data, thêm `standard_navbar_items` mới cho action "Delete Demo Data").
- `naming_series_variables`: từ chỉ có `FY` mở rộng thành 9 biến (`FY, TFY, ABBR, MM, DD, YY, YYYY, JJJ, WW`) — naming series linh hoạt hơn nhiều, có thể ảnh hưởng nếu vnpost tuỳ biến naming series dựa vào hành vi cũ chỉ hỗ trợ `FY`.
- Thêm `repost_allowed_doctypes` (mở rộng thêm `Purchase Receipt`), `subscription_doctypes`, `ignore_links_on_delete` cho `Tax Withholding Entry`.

---

## 5. Thay đổi tính năng / kiến trúc lớn khác

- **Cấu hình đa công ty (multi-company) chuyển từ singleton toàn hệ thống sang theo từng Company**, xác nhận qua các patch migrate dữ liệu tự động:
  - `set_valuation_method_on_companies`: field **Valuation Method** dời từ *Stock Settings* (1 giá trị chung cho cả hệ thống) sang field riêng trên từng **Company**.
  - `set_company_wise_warehouses`: kho WIP/Thành phẩm/Phế phẩm mặc định dời từ *Manufacturing Settings* sang field riêng trên từng Company.
  - `migrate_account_freezing_settings_to_company`: field khoá sổ kế toán (`acc_frozen_upto`, `frozen_accounts_modifier`) dời từ *Accounts Settings* sang Company.
  - → **Đây là thay đổi kiến trúc quan trọng nhất về mặt vận hành** nếu vnpost chạy nhiều Company: cấu hình cần rà soát lại theo từng công ty thay vì 1 chỗ chung, nhưng có patch tự động migrate nên dữ liệu hiện tại sẽ được giữ nguyên giá trị khi lên v16.
- **Đồng tiền báo cáo hợp nhất (Reporting Currency)**: patch `set_reporting_currency` tính và lưu số tiền quy đổi sang đồng tiền báo cáo của công ty mẹ ngay trên GL Entry/Account Balance Closing — hỗ trợ hợp nhất báo cáo tài chính nhóm công ty đa tiền tệ.
- **Report Center** (module mới, doctype `Report Center`): trang tổng hợp report chuẩn theo từng nhóm nghiệp vụ (vd "Accounting" trỏ tới Sales Register...), tích hợp với hệ Workspace Sidebar mới của Frappe v16.
- **Bank Reconciliation làm lại bằng React SPA riêng** (mục 2) + hỗ trợ import sao kê **định dạng MT940** và **trích bảng dữ liệu trực tiếp từ file PDF** (`pdfplumber`) — trước đây chỉ hỗ trợ CSV/XLSX.
- **Financial Report Template**: cho phép tự dựng mẫu báo cáo tài chính (P&L, Bảng cân đối) với style xuất Excel, thêm view tăng trưởng (growth) và biên lợi nhuận (margin).
- **Tax Withholding làm lại**: thêm khái niệm "Tax Withholding Group", doctype `Tax Withholding Entry` riêng để theo dõi từng khoản khấu trừ (trước đây tính toán ẩn trong report), có patch migrate dữ liệu cũ.
- **Currency Exchange Settings**: nhà cung cấp tỷ giá miễn phí đổi domain `frankfurter.app` → `frankfurter.dev` (tự động patch cập nhật, không cần thao tác thủ công).
- **Disassembly Order** và **Subcontracting Inward Order**: 2 nghiệp vụ hoàn toàn mới, đã nêu ở mục 3.
- `patches.txt` app tăng từ 429 → 488 dòng patch — tổng khối lượng migrate dữ liệu khi chạy `bench migrate` từ v15 lên v16 là rất lớn, nên **backup đầy đủ trước khi migrate** và thử trên bench test với dữ liệu thật (đã ẩn danh) trước.

---

## Tổng hợp việc cần làm cụ thể cho vnpost (bổ sung cho báo cáo Frappe)

1. **Rà soát toàn bộ report/tích hợp custom đọc trực tiếp field JSON `item_wise_tax_detail`** trên các chứng từ bán/mua — cấu trúc lưu trữ đã đổi sang child table `Item Wise Tax Detail`.
2. **Kiểm tra các app `mbwnext_*` có dùng hook `override_doctype_class` trong `hooks.py` riêng không** — phải đổi thành `extend_doctype_class`, nếu không hook sẽ mất tác dụng mà không báo lỗi rõ ràng.
3. **Rà soát code custom đọc biến `erpnext.hooks.advance_payment_doctypes`** (đã tách thành 2 biến `_receivable`/`_payable`), và bất kỳ job scheduler custom nào đăng ký cứng vào bucket `hourly_long`/`daily`/`daily_long` cũ.
4. **Nếu vnpost chạy nhiều Company**, kiểm tra lại cấu hình Valuation Method, kho WIP/Thành phẩm/Phế phẩm mặc định, và mốc khoá sổ kế toán sau migrate — các giá trị này chuyển từ cấu hình chung sang theo từng Company (có patch tự động nhưng nên đối chiếu lại thủ công).
5. **Nếu có dùng tính năng Homepage/Portal Homepage Section cho khách hàng**, cần chuyển sang giải pháp khác (Homepage đã bị xoá hoàn toàn) — kiểm tra patch `add_portal_redirects` để biết link cũ có được redirect không.
6. **Nếu có dùng "Advance Tax" hoặc report thuế khấu trừ (TDS) cũ**, cần kiểm tra kỹ dữ liệu sau khi chạy patch `migrate_tax_withholding_data` — đây là nghiệp vụ làm lại gần như hoàn toàn.
7. **Nếu app `print_designer`/`mbwnext_einvoice` có tuỳ biến giao diện đối chiếu ngân hàng (Bank Reconciliation)**, cần biết giao diện này đã chuyển hẳn sang một SPA React riêng (`banking/`) — tuỳ biến JS/CSS kiểu cũ trên desk sẽ không áp dụng được cho giao diện mới này nữa.
8. **Test lại toàn bộ luồng BOM/Job Card có "Scrap Item"** — đã đổi tên/mở rộng thành "Secondary Item" (bao gồm cả Co-Product/By-Product), kiểm tra field nào bị đổi tên trong quá trình đó.
9. **Nếu có báo cáo/tích hợp phụ thuộc doctype `Closing Stock Balance`**, cần cập nhật sang cặp doctype mới `Stock Closing Entry`/`Stock Closing Balance`.
10. Việc port 17 app custom (`mbwnext_*`, `pos_next`, `print_designer`, `super_admin`, `vnpost_sso`, `builder`) lên nhánh tương thích v16 — như đã nêu ở báo cáo Frappe — là bước bắt buộc để có thể đối chiếu đầy đủ với các thay đổi ở trên, đặc biệt phần kế toán/thuế Việt Nam (VAT, hoá đơn điện tử) rất có thể đụng trực tiếp vào cấu trúc `Item Wise Tax Detail` và nghiệp vụ Tax Withholding vừa bị làm lại.
