# Hướng dẫn sử dụng Hóa đơn điện tử Viettel

Tài liệu này hướng dẫn cấu hình và sử dụng tích hợp HĐĐT Viettel trong hệ thống.

## 1) Phạm vi áp dụng

- Áp dụng cho Sales Invoice đã `Submit`.
- Cấu hình tích hợp được đặt trong tab **E-Invoice Settings** của từng **Company**.
- Nhà cung cấp đang dùng: **Viettel**.

## 2) Chuẩn bị dữ liệu trước khi dùng

### 2.1 Cấu hình Company

Vào `Company` -> tab `E-Invoice Settings`, điền tối thiểu:

- `Use E-Invoice`: bật.
- `E-Invoice Provider`: chọn `Viettel`.
- `Tax Code (MST)`.
- `Username`, `Password` (tài khoản API Viettel).
- `API URL` (endpoint Viettel theo môi trường test/prod).
- `Invoice Pattern` (template code).
- `Invoice Serial`.

### 2.2 Địa chỉ công ty (bên bán)

Hệ thống lấy địa chỉ bên bán theo Address gắn với Company:

1. Ưu tiên Address có cờ **Preferred Billing Address**.
2. Fallback **Primary Address**.
3. Fallback địa chỉ mặc định trên Company.

Lưu ý: nếu có `Address Title`, hệ thống ưu tiên dùng đúng giá trị này để đưa lên hóa đơn.

### 2.3 Ngân hàng công ty (bên bán)

- Nếu SI có trường ngân hàng riêng thì ưu tiên dữ liệu trên SI.
- Nếu SI không có, hệ thống lấy từ `Bank Account` đầu tiên của công ty (`is_company_account = 1`):
  - `Bank` -> Ngân hàng
  - `Bank Account No` -> Số tài khoản

### 2.4 Dữ liệu khách hàng (bên mua)

- **Địa chỉ**: lấy từ Address của Customer, ưu tiên **Preferred Billing Address**.
- **Điện thoại**: lấy từ `Mobile No` của contact chính.
- Nếu thiếu, hệ thống fallback về thông tin trên SI.

## 3) Quy trình phát hành HĐĐT từ Sales Invoice

1. Mở `Sales Invoice` đã Submit.
2. Nếu chưa có HĐĐT, vào menu:
   - `E-Invoice` -> `Create Viettel E-Invoice`
3. Hệ thống gọi Viettel API để phát hành.
4. Sau khi thành công:
   - SI được cập nhật `created_einvoice`, `einvoice_no`, `einvoice_uuid`.
   - Tab `Einvoice Info` hiển thị preview PDF inline (nếu Viettel trả file).

## 4) Xem hóa đơn đã phát hành

- Nút:
  - `View` -> `E-Invoice` mở PDF tab mới.
- Hoặc xem trực tiếp preview tại tab `Einvoice Info`.
- Nếu API chưa trả PDF base64, hệ thống sẽ fallback link portal (nếu có).

## 5) Ý nghĩa các trường trên SI

- `Created Einvoice`: đã phát hành HĐĐT hay chưa.
- `E-Invoice No`: số hóa đơn điện tử.
- `E-Invoice UUID`: mã giao dịch/định danh để tra cứu.

## 6) Lỗi thường gặp & cách xử lý

### 6.1 `401 Unauthorized / invalid_grant`

Nguyên nhân: sai `Username`/`Password` hoặc sai môi trường endpoint.

Cách xử lý:

- Kiểm tra lại `Username`, `Password` trong Company.
- Kiểm tra đúng `API URL` test/prod.
- Kiểm tra phương thức auth đang dùng.

### 6.2 Không lên địa chỉ bên bán / bên mua

- Đảm bảo Address đã link đúng với Company/Customer.
- Tick đúng cờ **Preferred Billing Address**.
- Kiểm tra trường `Address Title` nếu muốn hiển thị chính xác theo tiêu đề.

### 6.3 Không có số hóa đơn ngay sau khi phát hành

Một số trường hợp Viettel xử lý bất đồng bộ:

- SI có thể có `einvoice_uuid` trước, `einvoice_no` cập nhật sau.
- Chờ thêm và thử tra cứu lại.

## 7) Khuyến nghị vận hành

- Cấu hình theo từng Company, tránh dùng chung credential giữa các pháp nhân.
- Chuẩn hóa Customer Address + Contact trước khi xuất hàng loạt.
- Test phát hành trên môi trường test trước khi chạy production.

## 8) Checklist nhanh

- [ ] Company bật `Use E-Invoice`
- [ ] Provider = `Viettel`
- [ ] Có `Tax Code`, `Username`, `Password`, `API URL`
- [ ] Có `Invoice Pattern`, `Invoice Serial`
- [ ] Company có Preferred Billing Address
- [ ] Customer có Preferred Billing Address + Mobile No
- [ ] SI đã Submit

