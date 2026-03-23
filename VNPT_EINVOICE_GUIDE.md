# Hướng dẫn sử dụng VNPT E-Invoice (MBWNext)

Tài liệu này hướng dẫn cấu hình và vận hành tích hợp hóa đơn điện tử VNPT trong app `mbwnext_einvoice`.

---

## 1) Phạm vi hỗ trợ hiện tại

- Kiểm tra kết nối VNPT từ `Company`.
- Phát hành hóa đơn từ `Sales Invoice`.
- Lấy PDF hóa đơn VNPT.
- Hủy hóa đơn VNPT theo `fkey`.

Tích hợp đang dùng SOAP API theo bộ service:

- `publishservice.asmx`
- `portalservice.asmx`
- `businessservice.asmx`

---

## 2) Chuẩn bị thông tin từ VNPT

Cần có đủ 3 nhóm thông tin:

1. **CAdmin URL**
   - Ví dụ: `https://2222222222-008-tt78democadmin.vnpt-invoice.com.vn`

2. **Portal URL**
   - Ví dụ: `https://2222222222-008-tt78demo.vnpt-invoice.com.vn`

3. **Tài khoản**
   - **Account/ACpass** (tài khoản nhân viên, dùng cho nghiệp vụ phát hành/hủy).
   - **username/password webservice** (ServiceRole).

> Lưu ý: Có môi trường VNPT cấp cùng 1 mật khẩu cho nhiều vai trò, cần nhập đúng theo tài liệu VNPT gửi.

---

## 3) Cấu hình tại Company -> E-Invoice Settings

### 3.1 General

- `Use E-Invoice`: bật.
- `E-Invoice Provider`: `VNPT`.
- `Tax Code (MST)`: MST đơn vị phát hành.

### 3.2 API Base URL

- `API URL`: nhập **CAdmin URL** (không thêm đường dẫn method).
  - Đúng: `https://...vnpt-invoice.com.vn`
  - Không nhập: `/publishservice.asmx`, `/portalservice.asmx`...

### 3.3 API Credentials

- `Username`: webservice username (ServiceRole).
- `Password`: webservice password.

### 3.4 VNPT Settings

- `VNPT Account`: Account.
- `VNPT ACpass`: ACpass.
- `Invoice Lookup URL`: Portal URL.
- `Invoice Pattern`: mẫu số (ví dụ `1/001`).
- `Invoice Serial`: ký hiệu (ví dụ `C26TTA`).

---

## 4) Kiểm tra kết nối

Trên form `Company`, chọn provider `VNPT`, sau đó bấm:

- `E-Invoice` -> `Test VNPT Connection`

Khi thành công hệ thống hiển thị:

- Provider
- Base URL
- Certificate Serial
- Certificate Valid To

Nếu lỗi, xem mục **10) Xử lý lỗi thường gặp**.

---

## 5) Luồng phát hành hóa đơn

1. Tạo và Submit `Sales Invoice`.
2. Tại `Sales Invoice`:
   - vào nhóm nút `E-Invoice`
   - bấm `Create VNPT E-Invoice`
3. Hệ thống gọi SOAP `ImportInvByPattern`.
4. Thành công sẽ cập nhật:
   - `created_einvoice = 1`
   - `einvoice_no` (số hóa đơn)
   - `einvoice_uuid` (fkey)

---

## 6) Mapping dữ liệu chính (Sales Invoice -> XML TT78)

Tích hợp phát hành theo cấu trúc XML TT78:

- `DSHDon/HDon/key`: lấy từ `Sales Invoice.name` (fkey nội bộ).
- `NDHDon/NMua/Ten`: `customer_name`.
- `NDHDon/NMua/MST`: `tax_id` (nếu có).
- `NDHDon/NMua/DChi`: địa chỉ khách hàng (ưu tiên billing address).
- `DSHHDVu/HHDVu/*`: map từ từng dòng item.
- `TToan/TgTCThue`: `net_total`.
- `TToan/TgTThue`: `total_taxes_and_charges`.
- `TToan/TgTTTBSo`: `grand_total`.
- `TToan/TgTTTBChu`: `in_words`.

---

## 7) Xem và tải PDF hóa đơn

Trên `Sales Invoice` đã có e-invoice:

- Nút `View -> E-Invoice` để mở PDF.
- Tab preview PDF hiển thị trực tiếp (nếu lấy được base64).

Thứ tự gọi API lấy PDF:

1. `downloadNewInvPDFFkey`
2. `downloadInvPDFFkeyNoPay`
3. `downloadInvPDFFkey`

Nếu chưa có PDF base64, hệ thống fallback sang link portal (`Invoice Lookup URL` + `fkey`).

---

## 8) Hủy hóa đơn

API đã có endpoint hủy theo `Sales Invoice`:

- `mbwnext_einvoice.mbwnext_einvoice.api.vnpt.cancel_invoice_for_sales_invoice`

Khi hủy thành công:

- reset `created_einvoice = 0`
- xóa `einvoice_no`, `einvoice_uuid`

> Nên kiểm tra nghiệp vụ nội bộ trước khi hủy để tránh lệch dữ liệu kế toán.

---

## 9) Nút test connection theo provider

`Company` đã hỗ trợ nút test cho:

- `VNPT` -> gọi `test_vnpt_connection`
- `Viettel` -> gọi `test_viettel_connection`

Điều kiện hiện nút:

- `Use E-Invoice` bật
- provider là `VNPT` hoặc `Viettel`

---

## 10) Xử lý lỗi thường gặp

### Lỗi HTTP 400 khi test VNPT

Kiểm tra lần lượt:

1. `API URL` đúng CAdmin URL chưa.
2. `Username/Password` webservice đúng chưa.
3. `VNPT Account/ACpass` đúng chưa.
4. `Invoice Pattern/Serial` đúng dải đã cấp chưa.
5. Chạy lại:
   - `bench --site <site> clear-cache`
   - `bench restart`
   - refresh trình duyệt.

### Lỗi `ERR:1`

- Sai tài khoản/mật khẩu hoặc không có quyền.

### Lỗi `ERR:20`

- Sai pattern/serial hoặc tài khoản không có quyền dải số.

### Lỗi `ERR:29`

- Chứng thư số hết hạn.

---

## 11) Vận hành sau khi nâng cấp code

Sau mỗi lần pull code mới:

1. `bench --site <site> migrate`
2. `bench --site <site> clear-cache`
3. `bench restart`

---

## 12) Ghi chú bảo mật

- Không chia sẻ `VNPT ACpass` và `Password` webservice.
- Chỉ cấp quyền cấu hình `Company` cho người phụ trách.
- Nên đổi mật khẩu định kỳ theo chính sách nội bộ.

