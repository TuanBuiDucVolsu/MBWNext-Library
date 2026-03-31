# Tổng hợp triển khai MISA MeInvoice (BA + Senior Dev)

Tài liệu này tổng hợp toàn bộ quá trình triển khai tích hợp MISA MeInvoice trong `mbwnext_einvoice`, phục vụ đào tạo lại cho đội Dev và vận hành.

---

## 1) Mục tiêu nghiệp vụ

- Tích hợp phát hành hóa đơn điện tử qua MISA MeInvoice từ ERPNext.
- Đảm bảo luồng vận hành tương tự các provider hiện có (Viettel, VNPT):
  - Cấu hình tại `Company > E-Invoice Settings`
  - Nút thao tác tại `Sales Invoice`
  - Lưu kết quả phát hành về chứng từ
  - Hỗ trợ xem PDF/link portal
- Chuẩn hóa thông báo lỗi, bản dịch và tài liệu vận hành cho người dùng nội bộ.

---

## 2) Phạm vi kỹ thuật đã triển khai

### 2.1. Kiến trúc tích hợp MISA

Đã thêm module tích hợp MISA theo pattern chuẩn của app:

- `integrations/misa/config.py`
  - Định nghĩa `MisaConfig`
  - Đọc cấu hình active từ Company (`provider = MISA`, `use_einvoice = 1`)
- `integrations/misa/client.py`
  - Quản lý auth token
  - Gọi API MISA (`/auth/token`, `/invoice`, `/invoice/templates`, `/invoice/download`, `/invoice/status`, ...)
  - Chuẩn hóa xử lý lỗi HTTP/API
- `integrations/misa/service.py`
  - Build payload `InvoiceData` từ `Sales Invoice`
  - Phát hành hóa đơn và parse `publishInvoiceResult`
  - Lấy PDF/link portal
- `mbwnext_einvoice/api/misa.py`
  - Expose endpoint whitelisted cho frontend:
    - test connection
    - issue invoice
    - get status
    - get templates
    - send email
    - get certificates

### 2.2. Tích hợp vào API tổng

- Cập nhật `api/einvoice.py`:
  - hỗ trợ provider `MISA` trong `get_einvoice_pdf`
  - trả về dữ liệu đồng nhất với frontend (`pdf_base64`, `portal_url`, `transaction_id`)

### 2.3. Tích hợp frontend

- `controllers/js/company.js`
  - thêm `Test MISA Connection`
  - thêm `Get MISA Certificates`
  - hiển thị danh sách chứng thư số HSM, bấm `Use` để đổ `Digital Certificate SN`
- `controllers/js/sales_invoice.js`
  - thêm nút `Create MISA E-Invoice`
  - gọi API phát hành và hiển thị kết quả (Transaction ID, Invoice No, CQT Code)
  - hỗ trợ xem PDF hoặc mở portal từ tab `E-Invoice Info`

### 2.4. Cấu hình Custom Field (Company)

Đã cập nhật fixture `custom_field.json` cho MISA:

- Hiển thị đúng section/field theo `depends_on` khi provider = `MISA`
- Bổ sung các field riêng:
  - `misa_app_id`
  - `misa_sign_type`
  - `misa_certificate_sn`
- Chuẩn hóa lại label/description sang tiếng Anh
- Đơn giản hóa cặp field có mã/không mã CQT:
  - giữ 1 checkbox chính `invoice_with_tax_code`
  - ẩn field đối nghịch gây nhiễu

### 2.5. Bản dịch ngôn ngữ

- Bổ sung/đồng bộ bản dịch trong:
  - `mbwnext_localization/locale/vi.po`
- Bao gồm:
  - label/description mới của MISA
  - message lỗi Python
  - message frontend (button, dialog, status, certificates, ...)

### 2.6. Tài liệu sử dụng

Đã tạo:

- `MISA_EINVOICE_GUIDE.md` (bản không dấu)
- `MISA_EINVOICE_GUIDE_VI.md` (bản tiếng Việt có dấu)

---

## 3) Các vấn đề thực tế đã gặp và cách xử lý

## 3.1. Lỗi auth trả message rỗng

**Hiện tượng**
- `MISA: Login failed. ErrorCode=. Errors=`

**Nguyên nhân**
- API có thể trả key khác chuẩn viết hoa/thường (`Success`/`success`, `Data`/`data`)
- Parser cũ chỉ đọc một định dạng.

**Xử lý**
- Cập nhật parser auth hỗ trợ cả uppercase/lowercase keys.
- Nâng chất lượng error message, parse thêm `message/descriptionErrorCode/errors`.

---

## 3.2. Lỗi `MisaIdError` / `UnAuthorize`

**Hiện tượng**
- Login fail với AppID/tài khoản sandbox.

**Nguyên nhân**
- Sai `AppID` hoặc sai `password`, hoặc mismatch bộ thông tin sandbox.

**Xử lý**
- Xác nhận lại bộ 4 thông tin: AppID + TaxCode + Username + Password cùng môi trường.
- Sau khi sửa password đúng, test connection thành công.

---

## 3.3. Lỗi phát hành `APINotSupportTypeInvoice`

**Hiện tượng**
- Gửi hóa đơn bị từ chối do type không phù hợp.

**Nguyên nhân nghiệp vụ**
- Ký hiệu hóa đơn không khớp `SignType`.
  - `...T...` (hóa đơn thường) không thể dùng `SignType=5`.

**Xử lý**
- Chuyển về mapping đúng:
  - `...T...` -> `SignType=2` + `CertificateSN`
  - `...M...` -> `SignType=5`
- Bổ sung nút lấy certificate ngay trong Company để người dùng thao tác nhanh.

---

## 3.4. Lỗi parse `publishInvoiceResult` (`'str' object has no attribute 'get'`)

**Hiện tượng**
- Traceback tại `first.get(...)`.

**Nguyên nhân**
- `publishInvoiceResult` đôi khi là JSON string (thậm chí chứa list), không phải dict trực tiếp.

**Xử lý**
- Bổ sung parser phòng thủ:
  - nếu string -> `json.loads`
  - nếu list -> lấy phần tử đầu
  - nếu format lạ -> log chi tiết + throw rõ ràng

---

## 3.5. Lỗi `TemplateIsNotUsing`

**Hiện tượng**
- MISA trả lỗi template chưa dùng.

**Nguyên nhân nghiệp vụ**
- Mẫu hóa đơn trên portal chưa ở trạng thái `Sử dụng`.

**Xử lý**
- Kích hoạt mẫu trên portal MISA.
- Retry phát hành thành công.

---

## 3.6. Không xem được PDF trực tiếp / lỗi decode `atob`

**Hiện tượng**
- UI báo lỗi decode PDF.
- Hoặc chỉ có link portal, mở ra timeout/522.

**Nguyên nhân**
- MISA response tải PDF không ổn định: có lúc trả base64, có lúc trả URL/handler.

**Xử lý**
- Backend chỉ trả `pdf_base64` khi xác định hợp lệ.
- Nếu không có base64, trả `portal_url`.
- Ưu tiên link `publishview` (ổn định hơn), fallback download URL.

---

## 3.7. Địa chỉ bắn sang MISA lệch với `Address Title`

**Hiện tượng**
- Địa chỉ trên hóa đơn MISA lẫn `Phone/Email`, khác địa chỉ kỳ vọng.

**Nguyên nhân**
- Fallback lấy từ `address_display` (chuỗi render HTML) có thể chứa thông tin phụ.

**Xử lý**
- Ưu tiên `Sales Invoice.customer_address`.
- Ưu tiên `Address Title`.
- Nếu không có title, ghép từ field địa chỉ thuần (`line1/line2/city/state/country`).
- Chỉ fallback `address_display` khi không còn nguồn khác.

---

## 4) Lưu ý nghiệp vụ quan trọng cho đội Dev

- MISA có thể tự chuẩn hóa phần năm của ký hiệu theo `InvDate`:
  - Ví dụ gửi `1C25TNP` nhưng hóa đơn năm 2026 có thể hiển thị `1C26TNP`.
- Không coi đây là bug code nếu dữ liệu portal đã đúng nghiệp vụ.
- Với sandbox, một số tính năng PDF/portal có thể không ổn định theo thời điểm.

---

## 5) Quy trình kiểm thử khuyến nghị (UAT/Regression)

- Bước 1: Cấu hình Company với provider `MISA`, test connection.
- Bước 2: Nếu `SignType=2`, dùng `Get MISA Certificates` để chọn `CertificateSN`.
- Bước 3: Phát hành hóa đơn từ Sales Invoice mẫu.
- Bước 4: Xác nhận dữ liệu lưu trên ERP:
  - `created_einvoice`
  - `einvoice_no`
  - `einvoice_uuid` (Transaction ID)
- Bước 5: Đối chiếu portal MISA:
  - số hóa đơn
  - ký hiệu
  - địa chỉ người mua
  - trạng thái
- Bước 6: Kiểm tra mở PDF:
  - xem trực tiếp (nếu có base64)
  - mở portal fallback

---

## 6) Danh sách hạng mục đã hoàn thành

- [x] Tích hợp backend MISA (config/client/service/api)
- [x] Tích hợp nút thao tác frontend ở Company + Sales Invoice
- [x] Cấu hình custom fields MISA trong Company
- [x] Chuẩn hóa message tiếng Anh trong code
- [x] Bổ sung bản dịch `vi.po`
- [x] Sửa parser response không ổn định của MISA
- [x] Sửa luồng lấy PDF/link portal
- [x] Sửa logic lấy địa chỉ đúng theo Address Title/chứng từ
- [x] Viết tài liệu hướng dẫn sử dụng

---

## 7) Đề xuất cải tiến tiếp theo

- Thêm nút `Check MISA Status` trực tiếp tại Sales Invoice để đối soát nhanh.
- Thêm cảnh báo thông minh:
  - nếu serial `...T...` nhưng `SignType=5` thì cảnh báo trước khi phát hành.
- Tạo dashboard theo dõi:
  - số hóa đơn phát hành thành công/thất bại theo ngày
  - top lỗi MISA thường gặp.

---

## 8) Ghi chú đào tạo cho team

Khi onboarding Dev mới, nên đi theo thứ tự:

1. Đọc cấu trúc chung app (`api`, `integrations`, `controllers/js`).
2. So pattern Viettel/VNPT trước, sau đó đọc module MISA.
3. Chạy test end-to-end bằng sandbox.
4. Đọc Error Log để hiểu format response thực tế từ MISA.
5. Thực hành xử lý 3 lỗi phổ biến:
   - `MisaIdError`
   - `APINotSupportTypeInvoice`
   - `TemplateIsNotUsing`

