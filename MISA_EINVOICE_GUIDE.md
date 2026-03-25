# Hướng dẫn sử dụng MISA eInvoice (ERPNext)

Tài liệu này hướng dẫn cấu hình và sử dụng MISA MeInvoice trong app `mbwnext_einvoice`.

## 1) Điều kiện trước khi cấu hình

- Đã cài app `mbwnext_einvoice`.
- Đã có tài khoản MISA MeInvoice (sandbox hoặc production).
- Đã được MISA cấp bộ thông tin API:
  - `AppID`
  - `Tax Code`
  - `Username`
  - `Password`
- Đã tạo và kích hoạt mẫu hóa đơn trên portal MISA (trạng thái `Sử dụng`).

## 2) Cấu hình `Company` > `E-Invoice Settings`

Vào `Company` > tab `E-Invoice Settings` và cấu hình:

- `Use E-Invoice`: bật.
- `E-Invoice Provider`: chọn `MISA`.
- `Tax Code (MST)`: mã số thuế đơn vị.
- `API URL`:
  - Sandbox: `https://testapi.meinvoice.vn/api/integration`
  - Production: `https://api.meinvoice.vn/api/integration`
- `Username`, `Password`: thông tin đăng nhập MISA.
- `Invoice Serial`: ví dụ `1C26TNP` hoặc `1C26MTM`.
- `MISA AppID`: AppID do MISA cấp.
- `Signing Method (SignType)`:
  - `2`: ký số HSM có hiển thị CKS (cần `Certificate SN`)
  - `5`: không ký số (thường dùng cho hóa đơn MTT/POS)
- `Digital Certificate SN`:
  - chỉ cần khi `SignType = 2` hoặc `3`
  - bấm nút `Get MISA Certificates` để lấy và điền nhanh.
- `Invoice With Tax Authority Code`: tích nếu là hóa đơn có mã CQT (ký tự thứ 2 của ký hiệu là `C`).

## 3) Test kết nối MISA

Tại `Company`:

- Bấm `E-Invoice` > `Test MISA Connection`.
- Nếu thành công, hệ thống hiển thị:
  - Provider
  - Base URL
  - Tax Code
  - Số lượng template tìm thấy

Nếu gặp lỗi `UnAuthorize` / `MisaIdError`:

- Kiểm tra lại `AppID`, `Tax Code`, `Username`, `Password` có đúng và cùng một môi trường (sandbox hoặc production) không.

## 4) Phát hành hóa đơn từ `Sales Invoice`

Tại `Sales Invoice` đã submit:

- Bấm `E-Invoice` > `Create MISA E-Invoice`.
- Nếu thành công:
  - `Created Einvoice` được đánh dấu
  - Lưu `E-Invoice No`
  - Lưu `Transaction ID` vào `einvoice_uuid`

## 5) Xem PDF hóa đơn

Trên tab `E-Invoice Info` của `Sales Invoice`:

- Nếu MISA trả `pdf_base64`: xem trực tiếp trong ERPNext.
- Nếu MISA chỉ trả link: hệ thống hiển thị nút `Open E-Invoice PDF on portal`.

Lưu ý: có thể MISA trả link portal/download thay vì base64 PDF. Đây là hành vi bình thường.

## 6) Quy tắc ký hiệu hóa đơn và SignType

Theo MISA:

- Ký tự thứ 2 của ký hiệu:
  - `C`: hóa đơn có mã CQT
  - `K`: hóa đơn không mã
- Ký tự thứ 5 của ký hiệu:
  - `T`: hóa đơn thường
  - `M`: hóa đơn từ máy tính tiền (MTT/POS)

Mapping khuyến nghị:

- `...T...` (hóa đơn thường) -> `SignType = 2` + nhập `Certificate SN`
- `...M...` (hóa đơn MTT) -> `SignType = 5` (không cần Certificate SN)

## 7) Lưu ý về năm trong ký hiệu

MISA có thể tự động chuẩn hóa phần năm trong ký hiệu theo ngày hóa đơn (`InvDate`).

Ví dụ:

- Gửi `1C25TNP` cho hóa đơn năm 2026
- MISA có thể lưu/chuẩn hóa thành `1C26TNP`

Đây là hành vi đúng theo nghiệp vụ MISA.

## 8) Lỗi thường gặp và cách xử lý

### `APINotSupportTypeInvoice`

- Nguyên nhân: `SignType` không phù hợp loại ký hiệu.
- Cách xử lý:
  - Ký hiệu `...T...` -> dùng `SignType = 2`
  - Ký hiệu `...M...` -> dùng `SignType = 5`

### `TemplateIsNotUsing`

- Nguyên nhân: mẫu hóa đơn trên MISA chưa ở trạng thái `Sử dụng`.
- Cách xử lý: vào portal MISA và kích hoạt (bắt đầu sử dụng) mẫu hóa đơn.

### `InvalidAppID` / `MisaIdError`

- Nguyên nhân: AppID sai hoặc không khớp bộ thông tin sandbox/production.
- Cách xử lý: liên hệ MISA để lấy lại đúng bộ thông tin API và cập nhật vào `Company`.

### Không xem được PDF trên portal (timeout/Cloudflare)

- Thử mở lại sau vài phút.
- Kiểm tra đúng môi trường (sandbox/production).
- Dùng tra cứu trên trang quản lý MISA bằng `Invoice No` hoặc `Transaction ID`.

## 9) Checklist cấu hình nhanh

- [ ] Provider = `MISA`
- [ ] API URL đúng môi trường
- [ ] AppID/TaxCode/Username/Password đúng
- [ ] `Invoice Serial` đúng loại (`T` hoặc `M`)
- [ ] `SignType` phù hợp với ký hiệu
- [ ] Mẫu hóa đơn trên MISA đang `Sử dụng`
- [ ] (Nếu `SignType = 2`) đã chọn `Digital Certificate SN`

---

Nếu cần, có thể bổ sung SOP vận hành (đổi serial theo năm, retry khi lỗi, và đối soát MISA-ERP cuối ngày).

