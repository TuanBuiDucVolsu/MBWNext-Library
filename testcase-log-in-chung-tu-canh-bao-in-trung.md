# Test case — Log in chứng từ & cảnh báo in trùng

| | |
|---|---|
| App | `mbwnext_ailinh` |
| DocType liên quan | Sales Order, Sales Invoice, Document Print Log |
| Spec | `docs/features/log-in-chung-tu-canh-bao-in-trung.md` |
| Mockup | `https://claude.ai/code/artifact/1504d18e-9af7-42e5-80a2-f9abfe41d2b8` |
| Người viết / Ngày | Claude (chạy thử đánh giá quy trình) / 2026-07-22 |

## Điều kiện chuẩn bị

- **Môi trường:** develop — site `ailinh.com` (bench `mbwnext_project`)
- **Tài khoản:** `Administrator` (mọi case chạy dưới quyền System Manager, trừ
  TC-PERM-01 cần thêm một user role `Sales User` để đối chiếu)
- **Dữ liệu cần có:** 1 Company (`Ái Linh`), 1 Warehouse (`Kho thành phẩm - AL`),
  1 Item bất kỳ có `is_stock_item=1` (item mẫu dùng: `Giảm thuế XK` — tồn kho sẵn
  trên site, không đại diện đúng nghiệp vụ thật, chỉ dùng để test kỹ thuật)
- **Dữ liệu tự tạo cho test:** Customer `TEST-KHAL01`; Sales Invoice/Sales Order
  tạo mới mỗi lần chạy (autoname), **đã xoá sạch sau khi test xong** — không còn
  tồn dư trên site `ailinh.com`

⚠ Tài khoản trên là tài khoản môi trường **develop**, không phải site production
của khách.

## Bảng test case

| Mã | Mục tiêu | Bước thực hiện | Dữ liệu vào | Kết quả mong đợi | KQ thực tế | P/F |
|---|---|---|---|---|---|---|
| TC-HAPPY-01 | In lần đầu một Hóa đơn bán hàng | Mở Hóa đơn bán hàng → bấm **In / Xuất PDF** lần đầu tiên | Hóa đơn chưa từng in | Không có cảnh báo nào hiện ra; hệ thống in/xuất PDF bình thường | Claude tự chạy qua `download_pdf` (Giai đoạn 2): không cảnh báo, `Document Print Log` = 1 dòng | ✅ P |
| TC-HAPPY-02 | In lần đầu một Đơn hàng bán | Mở Đơn hàng bán → bấm **In / Xuất PDF** lần đầu tiên | Đơn hàng chưa từng in | Không cảnh báo; `Document Print Log` ghi đúng `reference_doctype = Sales Order` | Claude tự chạy: `Document Print Log` = 1 dòng, không cảnh báo | ✅ P |
| TC-VALID | — | — | — | — | **Không áp dụng** — tính năng không có màn hình nhập liệu, chỉ phản ứng theo hành động bấm nút Print | — |
| TC-EDGE-01 | Chỉ xem trước bản in, không bấm in | Mở Print Preview của một chứng từ, **không** bấm nút in của trình duyệt | `trigger_print` không được gửi | Không ghi log, không cảnh báo | Claude tự chạy (Giai đoạn 2): log không tăng | ✅ P |
| TC-EDGE-02 | In một chứng từ ngoài phạm vi | Mở một DocType khác (ví dụ Khách hàng) → in / xuất PDF | DocType = Customer | Không ghi vào `Document Print Log` dù dùng chung 2 hàm gốc với Sales Order/Sales Invoice | Claude tự chạy (Giai đoạn 2): log không tăng | ✅ P |
| TC-EDGE-03 | In chứng từ chưa lưu (chưa có số) | Bấm in ngay trên form mới tạo, chưa bấm Lưu | Document chưa có `name` (còn là "New Sales Invoice 1") | Không ghi log — không có số chứng từ hợp lệ để ghi | **Chưa chạy trực tiếp** — xác nhận qua đọc code: điều kiện `isinstance(name, str)` trong `log_and_delegate_get_html_and_style` chặn đúng trường hợp này | ⚠️ Chưa test thật |
| TC-EDGE-04 | In liên tiếp nhiều lần, số đếm phải đúng | In cùng một Hóa đơn 3 lần liên tiếp | 1 Hóa đơn, in 3 lần | Log tăng 1→2→3; cảnh báo lần 2 nói "đã in 1 lần", lần 3 nói "đã in 2 lần" | Claude tự chạy: log = 3; lần 1 không cảnh báo; lần 2 = "đã in 1 lần"; lần 3 = "đã in 2 lần" | ✅ P |
| TC-EDGE-05 | In chứng từ ở trạng thái Nháp (chưa Submit) | Tạo Hóa đơn, **không** submit, bấm in ngay | `docstatus = 0` | Vẫn ghi log bình thường — tính năng không phân biệt trạng thái submit | Claude tự chạy: `docstatus=0` xác nhận, log = 1 sau khi in | ✅ P |
| TC-PERM-01 | Vai trò thấp có xem lại được log mình vừa tạo không | Đăng nhập bằng role `Sales User` (không phải System Manager) → mở danh sách `Document Print Log` | Role: Sales User | **Chưa có quyết định từ khách** — hiện chỉ System Manager đọc được, `Sales User` không thấy log dù chính họ tạo ra nó | Claude đọc quyền đã khai: chỉ `System Manager` có `read=1` | ⚠️ Cần khách quyết định — không phải lỗi, là câu hỏi thiết kế còn mở |
| TC-REGR-01 | Override không phá luồng render in bình thường | Gọi lại đúng hàm gốc `get_html_and_style` sau khi đã override, không bật `trigger_print` | Bất kỳ Hóa đơn | Vẫn trả về HTML/style bình thường — không lỗi, không rỗng | Claude tự chạy: trả về HTML hợp lệ (`html` không rỗng) | ✅ P |
| TC-REGR-02 | Không va chạm với app lõi đang hook Sales Order/Sales Invoice | Đối chiếu `hooks.py` của `advanced_selling`, `advanced_stock`, `advanced_accounting`, `localization` | 4 app đang cài trên site `ailinh.com` | Không app nào override `download_pdf`/`get_html_and_style` — tính năng này ở một lớp hoàn toàn khác (whitelisted method) so với `doc_events`/`doctype_js` mà 4 app kia dùng | Đã `grep` toàn bộ hooks.py của 4 app (Giai đoạn 2) — không có kết quả trùng | ✅ P |
| TC-ISO-01 | Site không cài `mbwnext_ailinh` không bị ảnh hưởng | Trên site `tamdaimoc.com` (không cài app này): kiểm tra hook và DocType | Site khác trong cùng bench | `override_whitelisted_methods` không đăng ký; DocType `Document Print Log` không tồn tại | Claude tự chạy trên `tamdaimoc.com`: cả hai đều `None` | ✅ P |

## Kết luận

- Tổng: 11 case (không tính TC-VALID không áp dụng) — Pass: 9 — Cần xử lý thêm: 2
  (TC-EDGE-03 chưa chạy thật, TC-PERM-01 chờ khách quyết định)
- Lỗi còn lại: không có lỗi thật — 2 case "chưa xong" đều là thiếu quyết định/thiếu
  điều kiện chạy, không phải code sai
- Đủ điều kiện nghiệm thu: **Chưa** — thiếu duyệt khách thật cho mockup (đã ghi ở
  Giai đoạn 2) và câu trả lời cho TC-PERM-01

## Hai điểm trong checklist chuẩn không áp dụng đúng nghĩa — ghi rõ, không bỏ qua

- **"Đã test cancel và amend"** — quy tắc này viết cho tính năng hook vào
  `doc_events` (submit/cancel/amend chạy lại toàn bộ hook). Tính năng này **không
  hook `doc_events`** — nó phản ứng theo hành động in, xảy ra độc lập với
  `docstatus`. Đã test Draft (TC-EDGE-05); không thấy lý do kỹ thuật để hành vi
  khác đi ở trạng thái Cancelled, vì code không đọc `docstatus` ở đâu cả. Không
  thêm case riêng cho Cancelled vì sẽ chỉ lặp lại đúng TC-EDGE-05.
- **"Custom Field có gán đúng module"** — không áp dụng, tính năng không thêm
  Custom Field nào, toàn bộ dữ liệu mới nằm trong DocType riêng `Document Print Log`.
