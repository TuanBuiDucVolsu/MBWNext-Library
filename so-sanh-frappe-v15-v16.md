# So sánh Frappe Framework v15.107.5 → v16.25.0

> Phạm vi: so sánh dựa trên diff trực tiếp giữa tag `v15.107.5` (bench production `vnpost`) và `v16.25.0` (bench test `testv16`), cách nhau 2.487 commit. Frappe không có file `CHANGELOG.md`/`RELEASE.md` chính thức trong repo (chỉ có doctype `Changelog Feed` lấy dữ liệu online lúc chạy), nên toàn bộ nội dung dưới đây được rút ra từ việc diff commit, file cấu hình, và cấu trúc thư mục thực tế — không phải suy đoán.

---

## 0. Rào cản môi trường cần xử lý trước (chặn nâng cấp)

| Hạng mục | v15.107.5 (production: vnpost) | v16.25.0 (test bench) |
|---|---|---|
| Python | `>=3.10,<3.15` (production đang chạy **3.12.3**) | **`>=3.14,<3.15` — bắt buộc Python 3.14** |
| Node.js | `>=18` (bench đã chạy 24.12.0 qua nvm) | `>=24` (đã đáp ứng sẵn) |
| Backend DB | MariaDB, Postgres | MariaDB, Postgres, **+ SQLite (mới)** |
| PDF engine | wkhtmltopdf / WeasyPrint | wkhtmltopdf (mặc định) **+ engine mới dựa trên Chrome/CDP** (`frappe/utils/pdf_generator/`, hook mới `pdf_generator`). Bench test đã có sẵn thư mục `/home/pc/testv16/chromium/` — cần binary Chromium. |

**Việc cần làm #1 (bắt buộc):** server production phải nâng lên Python 3.14.x trước khi có thể cài v16 — đây không phải tuỳ chọn (`requires-python = ">=3.14,<3.15"`). Đây là bước nhảy lớn hơn nhiều so với v14→v15, cần lên kế hoạch riêng ở tầng OS (glibc, package hệ thống, wheel C-extension cho mysqlclient/psycopg2/cryptography... phải có sẵn cho Python 3.14).

**Việc cần làm #2 (bắt buộc để đánh giá đầy đủ):** bench test (`/home/pc/testv16/apps`) hiện chỉ có `frappe + erpnext + hrms`. Production (`/home/pc/vnpost/apps`) có thêm 17 app custom: `builder, mbwnext_advanced_stock, mbwnext_advanced_distribution_map, pos_next, mbwnext_localization, mbwnext_reconciliation, mbwnext_dashboard, mbwnext_einvoice, mbwnext_integration_dms, mbwnext_advanced_accounting, mbwnext_econtract_service, mbwnext_advanced_buying, print_designer, mbwnext_vnpost, super_admin, vnpost_sso, mbwnext_advanced_selling`. Chưa app nào được đưa lên bench test — việc đánh giá v16 hiện chưa đầy đủ nếu thiếu bước port toàn bộ app custom lên nhánh tương thích version-16 (hoặc patch lại).

---

## 1. Breaking changes chính thức (đánh dấu `!` trong conventional commit)

Đây là những thay đổi mà chính Frappe tự gắn cờ là breaking:

1. `refactor!: Drop posthog integration` — bỏ hoàn toàn tích hợp phân tích PostHog và các hook liên quan.
2. `refactor!: Remove all Gravatar integration from framework (server and client)` — không còn fallback avatar qua Gravatar.
3. `refactor!: Remove UUID Utils library` — bỏ thư viện wrapper UUID, chuyển sang dùng `uuid` chuẩn của Python (Python 3.14 đã có UUIDv7 native). App custom nào import `frappe.utils.uuid_utils` (hoặc tương đương) phải chuyển sang `uuid` stdlib.
4. `fix!: Restrict allowed HTML in msgprints (#37399)` — `frappe.msgprint`/`frappe.throw` giờ sanitize HTML chặt hơn (`frappe/utils/messages.py`). App custom render HTML phức tạp trong msgprint có thể bị strip nội dung.
5. `fix!: use secret for auth between servers (#36778)` — auth token giữa các server (socket.io/webhook) giờ dùng secret lưu trong Redis theo bench thay vì theo site. Ảnh hưởng nếu có tích hợp custom gọi trực tiếp endpoint auth server-to-server cũ.
6. `fix!: Only query single for single doctypes (#36519)` — `frappe.db.get_value(doctype, None, ...)` không còn tự động fallback query Single doctype khi truyền `None` làm tên. Code nào dựa vào fallback ngầm này sẽ hỏng.
7. `fix!: Remove weird fallback to tabSingles (#36517)` — dọn dẹp liên quan đến cùng cơ chế fallback Single doctype ở trên.
8. `feat!: faster generation and formatting utils for excel exports (#36323)` — đổi signature/behavior của utility export Excel; code nào gọi trực tiếp `frappe.utils.xlsxutils` cần test lại.

Không tìm thấy footer `BREAKING CHANGE:` nào trong nội dung commit — danh sách đánh dấu `!` ở trên là đầy đủ những gì upstream tự nhận là breaking trong khoảng version này.

---

## 2. Thay đổi dependency / cấu trúc

### Python (`pyproject.toml`)
- `requires-python`: `>=3.10,<3.15` → **`>=3.14,<3.15`** (xem rào cản ở mục 0).
- Nâng version lớn: **RQ (Redis Queue) 1.15.1 → 2.6.1** (major — `pyproject.toml` có cảnh báo rõ: *"We depend on internal attributes of RQ... Audit the code changes w.r.t. `background_jobs.py` before updating"*), `redis 4.5.5 → 7.1.0`, `hiredis 2.2.3 → 3.3.0`, `Click 8.2 → 8.3.1`, `pypdf 6.10.2 → 6.13.3`, `PyMySQL 1.1.1 → 1.1.2`, `psutil 5.9.5 → 7.0.0`, `phonenumbers 8.13.55 → ~9.0.21`, `requests-oauthlib 1.3.1 → 2.0.0`, `tenacity 8.2.2 → 9.1.2`.
- Dependency mới thêm: `mysqlclient`, `nh3`, `orjson`, `xlsxwriter`, `pycountry`, `websockets` (dùng cho PDF generator Chrome mới), `distro`.
- Dependency bị bỏ: `boto3`, `dropbox`, `posthog`, `maxminddb-geolite2`, `cairocffi`, `tinycss2` — tương ứng trực tiếp với các tính năng bị xoá bên dưới (backup S3/Dropbox, telemetry PostHog).
- `background_jobs.py` diff lớn (256 dòng thay đổi) xác nhận có tái cấu trúc nội bộ thật sự gắn với việc nâng RQ 2.x — cẩn thận nếu app custom nào can thiệp vào internal của RQ (ví dụ script monitoring custom).
- `frappe/utils/scheduler.py` cũng thay đổi đáng kể (46 dòng).
- Có thêm `[project.optional-dependencies]` (`dev`, `test`) và cấu hình `[tool.mypy]`/basedmypy — chỉ ảnh hưởng tooling dev, không ảnh hưởng runtime.

### Node/JS (`package.json`)
- `engines.node`: `>=18` → `>=24`.
- Thêm mới: các package `@fullcalendar/*`, `photoswipe`, nâng `frappe-datatable 1.19.0 → 1.20.5`, nâng `@editorjs/editorjs`.
- Bỏ: `superagent`.
- Vue vẫn giữ `^3.3.0` — **không cần migrate Vue major version**, nhưng đừng chủ quan: phần desk frontend có thêm nhiều bề mặt SPA mới (xem mục 3), nên vẫn có thể cần nhiều công sức chỉnh JS custom dù Vue không đổi major.

### Thư mục cấp cao nhất trong module `frappe/`
- **Bị xoá:** `frappe/config`, `frappe/social` (tính năng gamification Energy Points — xoá hoàn toàn, không còn Energy Point Rule/Log/Settings), `frappe/translations` (file dịch CSV cũ).
- **Thêm mới:** `frappe/desktop_icon`, `frappe/workspace_sidebar` (mô hình điều hướng "Apps" màn hình chính mới, cấu hình dạng JSON), `frappe/testing` (internal của test framework), `frappe/locale` (**file gettext `.po` thay thế CSV dịch theo ngôn ngữ cũ** — hệ thống i18n viết lại dùng gettext; app custom hiện đang có file dịch CSV riêng — cần kiểm tra cơ chế mới có còn đọc được không, hoặc phải migrate sang `.po`/gettext extraction, xem module mới `frappe/gettext` với `extractors/` và `translate.py`).
- **Tính năng/doctype bị xoá hẳn (xác nhận qua việc thư mục biến mất):** `frappe/website/doctype/blog_post` (Blog Post), `frappe/email/doctype/newsletter` (Newsletter), `frappe/social/*` (Energy Points). Các hook tương ứng cũng bị xoá: route website `/blog/<category>`, `/newsletters`; job scheduler hourly `send_scheduled_email`; mục global search cho Blog Post/Newsletter.
- **Doctype core mới** (`frappe/core/doctype`): `api_request_log`, `permission_log`, `permission_type`, `role_replication`, `sms_log`, `user_role_profile`, `user_session_display`.
- **Doctype core bị xoá:** `transaction_log` (doctype audit-trail dạng hash-chain của v15 đã biến mất hoàn toàn — không còn tham chiếu nào trong source v16). Nếu report/tích hợp nào đọc doctype `Transaction Log` sẽ hỏng.
- Thêm mới `frappe/model/qb_query.py` (355 dòng mới) và `frappe/model/trace.py` (172 dòng mới) — cho thấy có tái cấu trúc ORM thật sự: việc build query đang dịch chuyển nhiều hơn về hướng Query Builder (`frappe.qb`), cộng thêm hạ tầng trace query mới. `frappe/model/document.py` (+895 dòng) và `frappe/model/base_document.py` (+719 dòng) thay đổi đáng kể — liên quan tới cơ chế "dehydrate"/sandbox mới mô tả bên dưới.
- **Tích hợp backup bị xoá khỏi hooks/core:** job backup định kỳ Dropbox, S3, và Google Drive (`daily_maintenance`, `weekly_long`, `monthly_long`) đều bị xoá. Nếu vnpost đang dựa vào backup tự động qua Dropbox/S3/Google Drive built-in của Frappe, **phải thay thế** (ví dụ cron ngoài + `bench backup` + rclone, hoặc kiểm tra xem có chuyển sang app riêng không).

---

## 3. Thay đổi tính năng / hành vi lớn

- **PDF generator mới dựa trên Chrome/CDP** (`frappe/utils/pdf_generator/{browser.py, chrome_pdf_generator.py, pdf_merge.py}`, hook mới `pdf_generator = "frappe.utils.pdf.get_chrome_pdf"`). Bật theo từng print format, mặc định vẫn là wkhtmltopdf. Vì wkhtmltopdf đã ngừng phát triển upstream, nên cân nhắc test/áp dụng engine mới — cần binary Chromium trên server (đã có sẵn trong bench test tại `/home/pc/testv16/chromium/`).
- **Mô hình object "dehydrated"/sandbox mới cho Server Script** — thêm `safer_exec`, `get_safer_globals`, `exec_safe_globals`, `render_safe_globals`, cùng các wrapper "dehydrate" bọc quanh object trả về từ `frappe.get_doc`, `new_doc`, `get_list`/`get_all`, `get_meta`, `get_cached_doc`, `get_last_doc`, `copy_doc`, `get_mapped_doc`. **Đây là thay đổi quan trọng nhưng không được ghi nhận là breaking chính thức** — nếu Server Script/Client Script custom của vnpost dựa vào việc truy cập đầy đủ attribute/method của Document object trả về từ các hàm này trong môi trường sandbox, hành vi có thể khác. Nên test riêng từng Server Script production với v16.
- **Hệ thống giới hạn concurrency mới**: `implement concurrency limiting decorator`, `derive concurrency limit from gunicorn master's cmdline`, `get_stats` cho concurrency limit, cache key mới dạng `concurrency:*`. Cơ chế này giới hạn request dựa theo số gunicorn worker — cần lưu ý nếu vnpost có endpoint custom chạy lâu hoặc export report nặng; có thể cần điều chỉnh `gunicorn_workers` (hiện tại là 17) hoặc opt-out cho endpoint cụ thể.
- **Màn hình chính "Apps" mới**: `app_home = "/app/build"`, hook mới `add_to_apps_screen` (app phải tự đăng ký với launcher Apps mới), hook `after_app_install`/`after_app_uninstall` (tự sinh/xoá desktop icon + sidebar entry), doctype mới `Desktop Icon` và cấu hình `Workspace Sidebar` dạng JSON theo từng module/role. **Cả 17 app custom của vnpost nên tự khai báo `desktop_icon`/sidebar config, hoặc chấp nhận mặc định của framework** — nếu không có thể không hiển thị đúng trong màn hình Apps mới.
- **Đổi URL scheme**: desk app chuyển từ `/app/...` sang `/desk/...`. `website_route_rules` và `website_redirects` được đảo ngược tương ứng (v15 redirect `/desk → /app`; v16 redirect `/app → /desk`, và `/apps`, `/app` → `/desk`). **Nên rà soát mọi link cứng `/app/...`** (JS app custom, bookmark, URL iframe nhúng, tích hợp hệ thống ngoài trỏ vào desk, link sâu trong email template) — link cũ vẫn redirect được nhưng link mới do framework sinh ra sẽ dùng `/desk`.
- **Tính năng Impersonation**: admin có thể "đóng vai" user khác, có gửi email thông báo cho user bị impersonate, và safeguard bỏ qua gửi mail khi ở developer mode.
- **Undo Send** cho email gửi đi (delay 10 giây + cảnh báo undo + API backend).
- Hỗ trợ **`security.txt`** (RFC 9116) tích hợp sẵn.
- **Assignment Rule**: thêm chiến lược phân phối round-robin có trọng số.
- **Query Builder**: thêm hàm `JSON` và `YEAR` native (bao gồm cả Postgres), helper `build_match_conditions`/`build_filter_conditions`, và toàn bộ module mới `qb_query.py` (đường query-builder đang được ưu tiên hơn so với build raw SQL string — liên quan nếu app custom build SQL thô đụng vào internal của db_query).
- Cải thiện **import CSV** (tự nhận diện delimiter, hỗ trợ format ngày như `DD-MMM-YYYY`).
- **Tái cấu trúc telemetry**: client telemetry "pulse" nhẹ hơn, gắn dimension `team`/`fc_team` — nên kiểm tra `site_config` để chắc chắn tắt telemetry nếu vnpost không muốn báo cáo phân tích về Frappe Cloud.
- Hỗ trợ separator cho **Group Concat** trên MariaDB trong filter query custom.
- Thêm option cấu hình site **`max_writes_per_transaction`** để canh chừng transaction chạy dài.
- **Global Search** giờ có thể cấu hình trực tiếp qua "Global Search Settings" thay vì chỉ qua hook `global_search_doctypes` trong hooks.py.
- Tần suất job full-text-search index của SQLite đổi từ mỗi 15 phút sang mỗi 3 tiếng (`build_index_if_not_exists`).

## 4. Tính năng bị xoá / deprecated đáng chú ý khác

- **Energy Points / hệ thống gamification** — xoá hoàn toàn (`frappe/social`), bao gồm các job scheduler liên quan (`allocate_review_points`, tổng kết theo tuần/tháng) và hook `on_change` gắn trên File nuôi dữ liệu cho tính năng này.
- **Blog Post** và **Newsletter** doctype/module — xoá khỏi frappe core (có thể chuyển sang app riêng, hoặc xoá hẳn — cần kiểm tra xem vnpost có dùng cái nào không).
- Hook **`leaderboards`** (`leaderboards = "frappe.desk.leaderboard.get_leaderboards"`) bị xoá khỏi hooks.py.
- Hook **`standard_navbar_items`** trong hooks.py bị xoá hoàn toàn — thay bằng hệ thống Apps/Workspace Sidebar mới; app nào tuỳ biến navbar qua hook này cần cách tiếp cận mới.
- **Tích hợp backup tự động Dropbox/S3/Google Drive built-in** bị xoá khỏi wiring scheduler mặc định (xem mục 2).
- Doctype **Transaction Log** (audit trail hash-chain) bị xoá.
- **File dịch dạng CSV** (`frappe/translations/*.csv`) thay bằng gettext `.po` (`frappe/locale/*.po`) — cần kiểm tra file dịch của app custom với cơ chế mới.
- Bỏ tích hợp **PostHog** và **Gravatar** (đã nêu trong mục breaking changes).

## 5. Tóm tắt thay đổi cấu trúc `hooks.py`

Các hook key mới mà app custom nên xem xét áp dụng/rà soát:
- `app_home` — route đích khi launcher "Apps" mở app của bạn.
- `after_app_install` / `after_app_uninstall` — hook lifecycle tự sinh icon/sidebar.
- `add_to_apps_screen` — danh sách dict đăng ký app vào launcher Apps mới (tên/logo/tiêu đề/route/permission check).
- `web_include_icons` — tách riêng bộ icon cho website với desk (`app_include_icons` cũng có thêm bộ icon `lucide` và `desktop_icons/alphabets.svg`).
- `pdf_generator` — điểm đăng ký cho PDF generator Chrome mới.
- `default_log_clearing_doctypes` thêm `OAuth Bearer Token`, `API Request Log`, `Email Queue Recipient`.
- `persistent_cache_keys` thêm `concurrency:*`.
- `ignore_links_on_delete` thêm `Permission Log`, `Desktop Icon`.
- Hook bị xoá: `leaderboards`, `standard_navbar_items`.

Lưu ý: file tài liệu `hooks.md` **không được cập nhật** giữa v15 và v16 — vẫn chỉ mô tả bộ hook cũ, nên các hook mới này chưa được ghi tài liệu chính thức, phải đọc trực tiếp `frappe/hooks.py`. App custom nên diff `hooks.py` của mình với `hooks.py` framework để xem hook key nào liên quan.

## 6. Diff file cấu hình

- **Procfile / patches.txt**: không có khác biệt đáng kể giữa vnpost (production, v15) và testv16 (bench test) — đây là các file ở tầng bench-tool, không phụ thuộc version framework. Không cần thay đổi gì ở đây cho việc nâng cấp framework.
- **`common_site_config.json`**: schema giống hệt nhau giữa 2 bench — khác biệt duy nhất là port (do chạy 2 bench riêng trên cùng máy). Không thấy key bắt buộc mới ở tầng cấu hình bench.
- **`site_config.json`**: site test v16 (`mbw.com`) có thêm 2 key mới/thay đổi so với site production v15 (`vnp.com`): `db_user` (giờ lưu tách riêng khỏi `db_name`) và `user_type_doctype_limit: {}` (rỗng theo mặc định — cơ chế giới hạn license mới, có thể liên quan Frappe Cloud nhưng vẫn xuất hiện ở cấu hình self-hosted). Cũng xác nhận `db_type: sqlite` là option hợp lệ ở cấp site (site `test.com` đang dùng) — hỗ trợ backend SQLite mới ở tầng site, nhưng nhiều khả năng dành cho dev/test hơn là migrate dữ liệu MariaDB production.

---

## Tổng hợp việc cần làm cụ thể cho việc nâng cấp vnpost

1. **Chuẩn bị Python 3.14** trên server đích (hoặc server mới) — đây là yêu cầu bắt buộc, không phải khuyến nghị. Xác nhận mọi dependency hệ thống/C-extension (mysqlclient, cryptography, psycopg2, WeasyPrint/pydyf, RestrictedPython) đã có wheel chạy được trên 3.14 trước khi chốt ngày nâng cấp.
2. **Port toàn bộ 17 app custom + `builder`/`print_designer`/`super_admin`/`vnpost_sso`** lên nhánh tương thích version-16 trong bench test trước khi coi việc đánh giá là hoàn tất; hiện bench test chưa có app nào trong số này.
3. **Rà soát Server Script/Client Script và mọi code custom nào dùng kết quả trả về từ `frappe.get_doc/get_list/get_all/get_meta/new_doc/copy_doc`** trong môi trường sandbox — lớp "dehydration"/`safer_exec` mới có thể đổi những attribute/method còn truy cập được.
4. **Rà soát code nào đụng trực tiếp internal của RQ/Redis** (monitoring custom, kiểm tra background job) do RQ nâng major 1.x→2.x — chính framework cảnh báo rủi ro này trong `pyproject.toml`.
5. **Nếu đang dùng backup built-in Dropbox/S3/Google Drive**, cần lên kế hoạch thay thế — các hook này đã bị xoá trong v16.
6. **Nếu đang dùng Energy Points, Blog Post, hoặc Newsletter doctype**, cần lên kế hoạch migrate dữ liệu/thay thế tính năng — đã bị xoá hoàn toàn.
7. **Kiểm tra xem có sử dụng doctype `Transaction Log`** trong report/tích hợp nào không — đã bị xoá.
8. **Kiểm tra link cứng dạng `/app/...`** trong code custom, email, tích hợp hệ thống ngoài — desk đã chuyển sang `/desk/...` (link cũ vẫn redirect nhưng không nên phụ thuộc lâu dài).
9. **Đăng ký mỗi app custom vào launcher Apps mới** qua `add_to_apps_screen`/`desktop_icon`/`workspace_sidebar` JSON, hoặc xác nhận mặc định của framework là chấp nhận được.
10. **Kiểm tra file dịch (translation) CSV của app custom** so với hệ thống i18n gettext/`.po` mới.
11. **Test lại các chỗ gọi `frappe.msgprint`/`throw` với nội dung HTML phức tạp** (do sanitize chặt hơn), và code nào dựa vào hành vi ngầm của `frappe.db.get_value(doctype, None, ...)` với Single doctype.
12. **Nếu có tuỳ biến print/PDF (đặc biệt là app `print_designer`)**, cần test riêng đường PDF generator Chrome mới — bắt buộc có binary Chromium trên server.
13. Chạy lại toàn bộ test suite app custom với bản nâng redis client 7.1.0/hiredis 3.3.0 và toolchain build Node `>=24`.
