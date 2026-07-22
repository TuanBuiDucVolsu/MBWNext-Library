# Lưu log hoạt động in chứng từ & cảnh báo in trùng - in lặp lại

**Nguồn:** `PhuongAnSoBo.MBWD.V3[Ailinh].md`, mục 1.3.1 (🔶) và 1.4.5.3
**Khách hàng:** Ái Linh
**Trạng thái nguồn:** Phương án sơ bộ (tiền hợp đồng) — chưa qua khảo sát chi tiết
**Ngày trích:** 2026-07-22

## Đầu bài gốc (nguyên văn từ tài liệu)

> Đơn hàng bán/ Hóa đơn bán hàng: Lưu lại hoạt động in mẫu in; khi in rồi thì note lại
> trạng thái in (áp dụng in lần 1).

## Ghi chú khi trích

- Câu gốc chỉ nói tới việc **ghi log lần in đầu tiên** ("áp dụng in lần 1") — chưa rõ
  cơ chế cảnh báo khi in trùng/in lặp lại hoạt động thế nào (chặn hẳn hay chỉ cảnh
  báo rồi cho in tiếp). Tên mục trong bảng phạm vi có nhắc "cảnh báo in trùng" nhưng
  phần "Phương án chi tiết" không mô tả cơ chế cảnh báo, chỉ mô tả phần ghi log.
- Cần hỏi khách: mục đích của tính năng là kiểm soát gian lận (tránh in nhiều bản
  gốc) hay chỉ để theo dõi thao tác — hai mục đích này dẫn tới hai mức ERROR khác hẳn
  nhau (chặn hẳn vs cảnh báo).
- Áp dụng cho cả Đơn hàng bán và Hóa đơn bán hàng — hai DocType riêng, cần xác nhận
  có dùng chung một cơ chế không.

---

## Giai đoạn 1 — Phân tích (chạy thử đánh giá quy trình, 2026-07-22)

### Bước 0 — Thu thập

⚠ **Không thực hiện được đầy đủ.** Đây là phiên chạy thử nội bộ để đánh giá quy
trình, không có khách Ái Linh thật để xin hiện vật (ảnh chụp mẫu in hiện tại, hỏi
người trực tiếp in chứng từ mỗi ngày). Các câu trả lời 5W dưới đây do người phụ
trách dự án quyết định thay, đóng vai người đại diện khách cho mục đích chạy thử —
**cần khách Ái Linh xác nhận lại khi dự án chính thức bắt đầu**, không dùng thẳng
làm spec cuối.

### Bước 1 — 5W

| | Trả lời |
|---|---|
| **WHAT** | Mỗi lần bấm Print/xuất PDF trên Đơn hàng bán hoặc Hóa đơn bán hàng, hệ thống ghi một dòng log (ai in, lúc nào, in lần thứ mấy). Nếu đã từng in trước đó, hiện cảnh báo ngay trên màn hình nhưng **không chặn** — người dùng vẫn in tiếp được. |
| **WHEN** | Lúc bấm nút Print hoặc xuất PDF (xác nhận qua AskUserQuestion) |
| **WHO** | Mọi người dùng in chứng từ, không phân biệt role (xác nhận qua AskUserQuestion) |
| **WHERE** | Cảnh báo hiện ngay trên form lúc in (đề xuất `frappe.msgprint`, chưa mockup) — **chưa hỏi khách**, để cổng mockup ở Giai đoạn 2 chốt |
| **ERROR** | Cảnh báo, không chặn (xác nhận qua AskUserQuestion) — mức ERROR nhẹ nhất trong 3 mức, khớp với việc đây là tính năng theo dõi chứ không phải kiểm soát gian lận |

### Bước 2 — Chọn tầng

Trả lời theo 4 câu hỏi, dừng ở câu đúng đầu tiên:

1. Là quy định pháp luật VN? → Không.
2. Đã có ≥2 khách cần, hoặc chắc chắn khách sau cũng cần? → Không có bằng chứng —
   tài liệu gốc mô tả đặc thù cho quy trình in ấn của riêng Ái Linh.
3. Dùng lại được nhưng không phải khách nào cũng bật? → Chưa có cơ sở xếp tầng 3.
4. Còn lại: quy trình nội bộ riêng của khách này → **ĐÚNG — tầng 4.**

**Kết quả: tầng 4, app `mbwnext_ailinh`.** Khớp với lựa chọn ban đầu của người phụ
trách dự án — cây quyết định không mâu thuẫn với quyết định đã đưa ra trước khi phân
tích, tốt cho việc đánh giá độ tin cậy của quy trình.

### Kiểm tra va chạm

⚠ **Sửa lại sau khi phát hiện lỗi ở lần kiểm tra đầu (Giai đoạn 1).** Lần đầu mình
tra theo `sites/apps.txt` — đó là danh sách app **có trong bench**, không phải app
**đang cài trên site**. Hai thứ khác nhau: một app nằm trong `apps.txt` mà chưa
`bench install-app` trên site cụ thể thì hook của nó **không chạy** trên site đó.
Kiểm tra đúng phải theo `bench --site <site> list-apps`. Đây là bài học thật rút ra
từ chính lần chạy thử này.

```bash
bench --site ailinh.com list-apps
```

App thực tế cài trên site `ailinh.com`: `frappe, erpnext, print_designer,
mbwnext_advanced_selling, mbwnext_advanced_buying, mbwnext_advanced_stock,
mbwnext_advanced_accounting, mbwnext_advanced_distribution_map,
mbwnext_localization, mbwnext_ailinh`.

**`pos_next` KHÔNG có trong danh sách này** — dù có trong `apps.txt` của bench. Nghĩa
là cảnh báo về `override_doctype_class` cho `Sales Invoice` ở bản nháp trước **không
áp dụng cho site này**. Rút gọn lại:

| App đang hook (site ailinh.com) | doctype_js | doc_events |
|---|---|---|
| `mbwnext_advanced_accounting` | ✓ (cả 2) | ✓ (cả 2) |
| `mbwnext_advanced_selling` | ✓ (cả 2) | ✓ (cả 2) |
| `mbwnext_advanced_stock` | ✓ (cả 2) | ✓ (cả 2) |
| `mbwnext_localization` | ✓ (Sales Order) | ✓ (Sales Order) |

**Ghi cùng trường?** Không — tính năng chỉ *đọc* hai DocType này để biết đang in cái
gì, *ghi* vào DocType log riêng, không ghi đè trường nào của chúng. Va chạm vô hại.

### Cơ chế kỹ thuật — đã xác định, không còn "chưa rõ"

Bản nháp Giai đoạn 1 từng ghi "Frappe không có `doc_event` chuẩn cho hành động in,
cần khảo sát thêm". Đào sâu hơn ở bước Dev tìm ra câu trả lời cụ thể, đơn giản hơn dự
đoán ban đầu:

Mọi lượt in/xuất PDF đều đi qua đúng 2 hàm **đã whitelisted sẵn** trong Frappe core:

| Hàm gốc | Khi nào gọi | File |
|---|---|---|
| `frappe.utils.print_format.download_pdf` | Bấm "Download PDF" | `frappe/utils/print_format.py:236` |
| `frappe.www.printview.get_html_and_style` | Mở Print Preview; tham số `trigger_print=1` khi thật sự bấm nút in trình duyệt | `frappe/www/printview.py:305` |

**Không có app nào trong site đang override 2 hàm này** (đã `grep` toàn bộ app đang
cài — không có kết quả). Nghĩa là dùng **cách 4 — `override_whitelisted_methods`**
(wrap-and-delegate) là đủ, **không cần `doctype_js`** như dự đoán ban đầu ở Giai đoạn
1. Đây đúng là phát hiện mà việc chọn tính năng này để chạy thử muốn kiểm tra: quy
trình có buộc phải đào sâu kỹ thuật trước khi commit vào một cơ chế nặng hơn cần
thiết không — và nó có.

Với `get_html_and_style`, chỉ ghi log khi `trigger_print` là `true` — mở preview để
xem không tính là một lần in, tránh đếm nhầm.

### Áp dụng chung cho 2 DocType

Đơn hàng bán (Sales Order) và Hóa đơn bán hàng (Sales Invoice) dùng **chung một cơ
chế và một DocType log**, không tách riêng — cả hai đều là "in chứng từ bán hàng",
không có lý do nghiệp vụ để xử lý khác nhau. Log nên tham chiếu qua cặp
`reference_doctype` / `reference_name` (mẫu chuẩn của Frappe), không tạo 2 bảng log
riêng cho 2 DocType.

---

## Giai đoạn 2 — Dev (chạy thử đánh giá quy trình, 2026-07-22)

⚠ **Lỗi quy trình thật đã xảy ra khi chạy thử: làm bước ④ (viết logic) trước bước ③
(mockup).** UI của tính năng này chỉ là một hộp cảnh báo (`msgprint`), cảm giác "nhỏ
quá không cần mockup" — và đó chính xác là kiểu lý do quy trình đặt cổng chặn để
ngăn. Mockup được dựng bù lại sau (`docs/mockups/log-in-chung-tu-canh-bao-in-trung.html`),
đúng nội dung UI đã code chứ không phải ngược lại. Ghi nhận đây là phát hiện thật của
lần chạy thử, không sửa lại lịch sử.

Chi tiết duyệt: xem mục "Duyệt mockup" phía dưới, sau phần test.

### Bước ② — DocType

Kiểm tra Frappe core trước — **chưa có sẵn cơ chế theo dõi in ấn** (`grep` core
`print_count`/`PrintLog` không ra kết quả, DocType `Sales Invoice` không có field liên
quan). Xác nhận cần dựng mới, không phải làm lại thứ đã có.

Tạo DocType **`Document Print Log`** (module `MBWNext Ailinh`, không submittable):

| Field | Kiểu | Bắt buộc |
|---|---|---|
| `reference_doctype` | Link → DocType | ✓ |
| `reference_name` | Dynamic Link (theo `reference_doctype`) | ✓ |
| `printed_by` | Link → User, mặc định user hiện tại | ✓ |
| `printed_on` | Datetime, mặc định `now` | ✓ |
| `via` | Select: `PDF` / `Print Preview` | ✓ |

Dùng `reference_doctype`/`reference_name` — mẫu chuẩn của Frappe — để một DocType log
áp dụng chung cho cả Sales Order và Sales Invoice, đúng quyết định ở Giai đoạn 1.

### Cơ chế kỹ thuật — xác định lại, đơn giản hơn dự đoán ban đầu

Giai đoạn 1 dự đoán cần `doctype_js` (bắt sự kiện phía client). Đào code Frappe core
tìm ra cách nhẹ hơn: mọi lượt in/PDF đều đi qua đúng 2 hàm **đã whitelisted sẵn**:

- `frappe.utils.print_format.download_pdf` — khi tải PDF
- `frappe.www.printview.get_html_and_style` — khi mở Print Preview; có tham số
  `trigger_print` phân biệt "chỉ xem" và "thật sự bấm in trên trình duyệt"

Không app nào đang cài trên site `ailinh.com` override 2 hàm này → dùng thẳng
**cách 4 — `override_whitelisted_methods`**, wrap-and-delegate, không cần `doctype_js`.

### Kiểm tra va chạm — sửa sai lần đầu

⚠ Lần kiểm tra đầu (Giai đoạn 1) tra theo `sites/apps.txt` — danh sách app **có
trong bench**, không phải app **cài trên site**. `pos_next` có trong `apps.txt`
nhưng **không có** trong `bench --site ailinh.com list-apps` → cảnh báo về
`override_doctype_class` cho `Sales Invoice` ở bản nháp trước **không áp dụng**.
Đây là bài học thật, đã sửa lại đúng ở phần "Kiểm tra va chạm" phía trên.

### Bước ④ — Logic

File `controllers/python/print_log.py` (đặt theo tính năng, vì cắt ngang 2 DocType):
2 hàm wrap-and-delegate, có `APPLY_TO = {"Sales Order", "Sales Invoice"}` để không
log nhầm mọi lượt in trong toàn hệ thống (2 hàm gốc dùng chung cho mọi DocType).

Đăng ký ở `hooks.py`:
```python
override_whitelisted_methods = {
    "frappe.utils.print_format.download_pdf": "mbwnext_ailinh.controllers.python.print_log.log_and_delegate_download_pdf",
    "frappe.www.printview.get_html_and_style": "mbwnext_ailinh.controllers.python.print_log.log_and_delegate_get_html_and_style",
}
```

### Claude tự chạy thử trên dev — kết quả thật

Site `ailinh.com` trống dữ liệu — tạo Customer/Sales Invoice tiền tố `TEST-` để test,
xoá sạch sau khi xong (đúng quy tắc dữ liệu test).

Hai lỗi môi trường gặp phải trên đường đi, **không liên quan tới code tính năng**,
đã xử lý xong trước khi kết luận kết quả:
- Site chưa `bench migrate` → thiếu cột `tabContact.is_billing_contact` → đã migrate.
- `wkhtmltopdf` báo lỗi mạng khi tạo PDF thật (môi trường không có internet) — không
  che giấu, ghi nhận là giới hạn môi trường, không phải lỗi cơ chế.

| Case | Kỳ vọng | Kết quả thật | KQ |
|---|---|---|---|
| In lần 1 qua `get_html_and_style` (`trigger_print=1`) | Không cảnh báo, ghi 1 log | Không cảnh báo giả, log = 1 | ✅ Pass |
| In lần 2 cùng chứng từ | Cảnh báo đúng nội dung, không chặn, log = 2 | `"Chứng từ này đã được in 1 lần trước đó."`, log = 2 | ✅ Pass |
| Chỉ mở Print Preview, không bấm in (`trigger_print=0`) | Không ghi log | Không ghi log | ✅ Pass |
| In một DocType ngoài phạm vi (`Customer`) | Không ghi log | Không ghi log | ✅ Pass |
| In lần 1 qua `download_pdf` | Không cảnh báo, ghi log | Ghi log đúng (`count=1`) — nhưng PDF thật lỗi do `wkhtmltopdf` không có mạng | ⚠️ Logic Pass, môi trường Fail |

**Câu hỏi thiết kế còn mở**, cần khách thật trả lời: log có nên ghi cả khi việc tạo
PDF thất bại giữa chừng không? Hiện tại code ghi log **trước** khi gọi hàm gốc, nên
dù PDF lỗi thì log vẫn được tính là "đã in" — có thể đúng ý (ý định bấm in đã xảy ra)
hoặc sai ý (khách sẽ hỏi "sao chưa in được mà đã báo tôi in rồi"). Đây đúng dạng câu
hỏi ERROR case mà thật ra nên hỏi khách ở Giai đoạn 1, nhưng chỉ lộ ra khi test thật.

### Duyệt mockup — hoàn tất (2026-07-22, sai thứ tự)

Dựng mockup dạng artifact (3 trạng thái: chưa in / in trùng / chỉ xem trước) —
`https://claude.ai/code/artifact/1504d18e-9af7-42e5-80a2-f9abfe41d2b8`. Người phụ
trách dự án xem và duyệt đúng cả 3, nhưng **sau khi code đã tồn tại**, không phải
trước — hệ quả của lỗi thứ tự đã ghi ở đầu Giai đoạn 2, không phải lỗi mới. Đối
chiếu lại: code khớp đúng nội dung đã duyệt (câu cảnh báo, điều kiện `trigger_print`,
danh sách `APPLY_TO`) — không cần viết lại.

**Duyệt khách: chưa làm được**, không có khách Ái Linh thật trong phiên chạy thử.
Đây là điều kiện còn thiếu duy nhất trước khi coi tính năng này "xong" cho dự án thật.

### Bước ⑤ — Bản dịch

**Phát hiện lỗi convention trước khi dịch:** code ban đầu hardcode tiếng Việt trực
tiếp trong `_()` (`"Chứng từ này đã được in..."`), sai quy ước đã chốt — msgid phải
là tiếng Anh, tiếng Việt nằm ở `vi.po`. Sửa lại thành
`_("This document has already been printed {0} time(s) before.")` trước khi dịch.

Dùng đúng lệnh của skill để tự động bắt hết chuỗi cần dịch, không dịch tay từng chuỗi:

```bash
bench generate-pot-file --app mbwnext_ailinh
bench update-po-files --app mbwnext_ailinh --locale vi
```

Tự động bắt đủ 9 chuỗi: tên DocType, 5 label field, 2 option của Select `via`, 1
message + 1 title. Không sót chuỗi nào phải tự nhớ.

⚠ **Phát hiện quan trọng nhất của bước này — thiếu trong tài liệu quy trình hiện
tại:** `bench update-po-files` chỉ cập nhật file `.po` (text), **không đủ để bản
dịch có hiệu lực**. Frappe đọc bản dịch runtime từ file `.mo` (nhị phân, gettext).
Test `_()` trực tiếp sau khi chỉ chạy `update-po-files` → **vẫn trả về tiếng Anh**.
Phải chạy thêm:

```bash
bench compile-po-to-mo --app mbwnext_ailinh
bench --site ailinh.com clear-cache
```

Sau đó test lại — `_()` trả đúng tiếng Việt. Nên **sửa lại skill
`erpnext-mbwnext-customer-app` và `erpnext-mbwnext-localization`**, mục "Đa ngôn
ngữ" — cả hai đang chỉ nhắc `generate-pot-file` + `update-po-files`, thiếu
`compile-po-to-mo`. Đây là bước ai cũng sẽ quên nếu không tự tay chạy thử như lần
này.

### Bước ⑥ — Tài liệu

Tạo `CLAUDE.md` và viết lại `README.md` cho `mbwnext_ailinh` (app hoàn toàn mới,
trước đó chỉ có template mặc định của `bench new-app`).

---

## Giai đoạn 3 — Test case (chạy thử đánh giá quy trình, 2026-07-22)

File đầy đủ: `docs/testcases/log-in-chung-tu-canh-bao-in-trung.md`.

11 case (6 nhóm, `TC-VALID` không áp dụng — tính năng không có màn hình nhập liệu).
**9 Pass, 2 chưa xong**: `TC-EDGE-03` (in chứng từ chưa lưu) chỉ xác nhận qua đọc
code, chưa chạy thật; `TC-PERM-01` phát hiện một câu hỏi thiết kế mới — vai trò
thấp (`Sales User`) hiện **không tự xem được** log do chính họ tạo ra (chỉ
`System Manager` có quyền đọc `Document Print Log`). Giai đoạn 1 chưa từng hỏi "ai
được XEM log" — chỉ hỏi "ai được IN" — đây là lỗ hổng 5W thật, chỉ lộ ra khi viết
test case, không lộ ra khi phân tích.

Xoá sạch dữ liệu test sau khi chạy (site `ailinh.com` không còn Sales
Order/Invoice/Customer nào có tiền tố `TEST-`), xoá script test tạm.

### Còn thiếu để "xong" thật cho dự án thật

- **Duyệt khách thật** cho mockup — điều kiện còn thiếu duy nhất từ Giai đoạn 2
- Câu hỏi thiết kế mở (Giai đoạn 2): log có nên ghi cả khi PDF tạo thất bại giữa
  chừng không?
- Câu hỏi thiết kế mở (Giai đoạn 3): ai được xem `Document Print Log` ngoài
  `System Manager`? Cả hai câu hỏi này đáng đưa vào 5W của Giai đoạn 1 cho lần sau,
  không nên để tới lúc test mới phát hiện.
