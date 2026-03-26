# MBWNext Modiser - Tóm Tắt Tính Năng và Bộ Test Cases

Tài liệu này tóm tắt các tính năng mới được thêm vào và cung cấp test case cho QA.

## 1) Phạm Vi Tính Năng Mới

### 1.1 Tự động điền trên Sales Order theo quy tắc
- Khi người dùng chọn `Customer` trên `Sales Order`, hệ thống tự động khớp và điền:
  - `custom_bank_account` (Tài khoản ngân hàng)
  - `set_warehouse` (Kho xuất trên header)
  - `warehouse` từng dòng item (tất cả items)
  - `custom_van_chuyen` (Đơn vị vận chuyển)

### 1.2 Matching API (gọi một lần duy nhất)
- API: `mbwnext_modiser.api.rule_matching.get_matching_rules`
- Đầu vào: `customer`
- Đầu ra:
  - `bank_account`
  - `warehouse`
  - `carrier`

### 1.3 Các Doctype quy tắc được thêm mới
- `Bank Account Selection Rule` (Quy tắc chọn ngân hàng)
- `Warehouse Selection Rule` (Quy tắc chọn kho xuất hàng)
- `Carrier Selection Rule` (Quy tắc chọn đơn vị vận chuyển)

### 1.4 Các Doctype hỗ trợ được thêm mới
- `VN Province` (Tỉnh/Thành phố)
- `VN District` (Quận/Huyện)

### 1.5 Custom Fields được thêm/sử dụng
- `Customer`
  - `custom_tinh_cu` (Tỉnh cũ)
  - `custom_huyen_cu` (Quận/Huyện cũ)
- `Sales Order`
  - `custom_van_chuyen` (Đơn vị vận chuyển)
  - `custom_bank_account` được dùng trong logic (tạo bởi app khác)
- `Delivery Note`
  - `custom_van_chuyen` (Đơn vị vận chuyển)

### 1.6 Kế thừa đơn vị vận chuyển sang Delivery Note
- Khi validate `Delivery Note`, nếu DN có đúng một SO nguồn và trường carrier của DN đang trống:
  - tự động sao chép `custom_van_chuyen` từ `Sales Order` nguồn.

## 2) Hành Vi Matching Theo Quy Tắc (Kỳ vọng)

### 2.1 Nguyên tắc chung
- Trường điều kiện để trống trong quy tắc = khớp mọi giá trị (wildcard).
- Trường điều kiện có giá trị trong quy tắc = phải khớp chính xác.
- Độ đặc hiệu cao hơn sẽ thắng (nhiều điều kiện không rỗng được khớp hơn).
- Customer Group được cộng điểm ưu tiên khi matching kho và vận chuyển.
- Tiebreak: sắp xếp theo tên quy tắc tăng dần.

### 2.2 Ưu tiên đặc biệt cho ngân hàng
- Nếu `Customer.default_company_bank_account` đã được thiết lập:
  - dùng giá trị này trực tiếp,
  - bỏ qua toàn bộ quy tắc ngân hàng.

## 3) Điều Kiện Tiên Quyết Cho QA

1. Site đã migrate với code mới nhất.
2. App đã cài đặt và đã xóa cache.
3. Dữ liệu master đã chuẩn bị:
   - Bank Accounts (Tài khoản ngân hàng)
   - Warehouses (Kho hàng)
   - Customer Groups (Nhóm khách hàng)
   - VN Provinces và VN Districts (Tỉnh/Thành phố và Quận/Huyện)
4. Khách hàng test đã được tạo với các giá trị đa dạng:
   - `customer_group`
   - `custom_channel`
   - `custom_tinh_cu`
   - `custom_huyen_cu`

## 4) Bộ Test Cases

### TC-01: Validate Quy Tắc Ngân Hàng – thiếu điều kiện
- **Thiết lập:** Tạo `Bank Account Selection Rule` với cả `channel` và `customer_group` đều để trống.
- **Các bước:** Lưu.
- **Kỳ vọng:** Lưu bị chặn với thông báo lỗi validation.

### TC-02: Validate Quy Tắc Ngân Hàng – trùng điều kiện đang active
- **Thiết lập:** Đã tồn tại một quy tắc active có cùng `channel` + `customer_group`.
- **Các bước:** Tạo quy tắc active thứ hai với cùng cặp điều kiện đó.
- **Kỳ vọng:** Lưu bị chặn (lỗi trùng quy tắc).

### TC-03: Validate Quy Tắc Kho – không có điều kiện nào
- **Thiết lập:** Tạo `Warehouse Selection Rule` với tất cả các trường điều kiện để trống.
- **Các bước:** Lưu.
- **Kỳ vọng:** Lưu bị chặn với thông báo lỗi validation.

### TC-04: Validate Quy Tắc Kho – chọn huyện nhưng chưa chọn tỉnh
- **Thiết lập:** Điền `huyen_cu`, để trống `tinh_cu`.
- **Các bước:** Lưu.
- **Kỳ vọng:** Lưu bị chặn (phải chọn tỉnh trước).

### TC-05: Validate Quy Tắc Kho – trùng customer_group
- **Thiết lập:** Đã tồn tại quy tắc kho active dùng customer group X.
- **Các bước:** Tạo thêm một quy tắc kho active với cùng customer group X.
- **Kỳ vọng:** Lưu bị chặn.

### TC-06: Validate Quy Tắc Kho – trùng toàn bộ điều kiện
- **Thiết lập:** Đã tồn tại quy tắc active với cùng `customer_group` + `tinh_cu` + `huyen_cu`.
- **Các bước:** Tạo quy tắc khác với cùng tổ hợp đó.
- **Kỳ vọng:** Lưu bị chặn.

### TC-07: Validate Quy Tắc Vận Chuyển – không có điều kiện nào
- **Thiết lập:** Tạo `Carrier Selection Rule` với cả `customer_group` và `tinh_cu` đều để trống.
- **Các bước:** Lưu.
- **Kỳ vọng:** Lưu bị chặn.

### TC-08: Validate Quy Tắc Vận Chuyển – trùng customer_group
- **Thiết lập:** Đã tồn tại quy tắc vận chuyển active dùng customer group X.
- **Các bước:** Tạo thêm quy tắc vận chuyển active với cùng customer group X.
- **Kỳ vọng:** Lưu bị chặn.

### TC-09: Validate Quy Tắc Vận Chuyển – trùng toàn bộ điều kiện
- **Thiết lập:** Đã tồn tại quy tắc vận chuyển active với cùng `customer_group` + `tinh_cu`.
- **Các bước:** Tạo quy tắc khác với cùng tổ hợp đó.
- **Kỳ vọng:** Lưu bị chặn.

### TC-10: Thay đổi Customer trên SO kích hoạt tự động điền
- **Thiết lập:** Tạo các quy tắc khớp duy nhất với một khách hàng test.
- **Các bước:** Tạo SO mới → chọn khách hàng.
- **Kỳ vọng:**
  - Tài khoản ngân hàng được điền,
  - `set_warehouse` trên SO được điền,
  - Kho của tất cả dòng item được điền,
  - Đơn vị vận chuyển được điền.

### TC-11: Ngân hàng bị ghi đè bởi tài khoản mặc định của khách hàng
- **Thiết lập:** Khách hàng có `default_company_bank_account`; đồng thời tồn tại quy tắc ngân hàng có kết quả khác.
- **Các bước:** Tạo SO và chọn khách hàng.
- **Kỳ vọng:** Tài khoản ngân hàng trên SO bằng giá trị mặc định của khách hàng, không phải kết quả từ quy tắc.

### TC-12: Nhiều quy tắc khớp – độ đặc hiệu cao hơn thắng
- **Thiết lập:** Thêm 2+ quy tắc có thể khớp; một quy tắc có nhiều điều kiện không rỗng hơn.
- **Các bước:** Chọn khách hàng trên SO.
- **Kỳ vọng:** Quy tắc có độ đặc hiệu cao nhất được áp dụng.

### TC-13: Nhiều quy tắc khớp – ưu tiên customer group thắng
- **Thiết lập:** Đối với kho/vận chuyển, tạo một quy tắc chỉ theo tỉnh và một quy tắc theo customer group.
- **Các bước:** Chọn khách hàng khớp với cả hai.
- **Kỳ vọng:** Quy tắc có customer group thắng.

### TC-14: Tiebreak theo tên quy tắc
- **Thiết lập:** Hai quy tắc có cùng điểm và cùng khớp.
- **Các bước:** Chọn khách hàng.
- **Kỳ vọng:** Quy tắc có tên nhỏ hơn theo thứ tự từ điển sẽ thắng.

### TC-15: Không có quy tắc nào khớp
- **Thiết lập:** Khách hàng không khớp bất kỳ quy tắc active nào.
- **Các bước:** Chọn khách hàng trên SO.
- **Kỳ vọng:** Trường đích liên quan vẫn để trống; không hiển thị lỗi ngoại lệ.

### TC-16: SO mới với khách hàng đã được điền sẵn (trigger khi refresh)
- **Thiết lập:** Mở SO mới đã có sẵn customer (ví dụ từ luồng khác).
- **Các bước:** Load form.
- **Kỳ vọng:** Matching vẫn thực thi và các trường được tự động điền.

### TC-17: DN kế thừa đơn vị vận chuyển từ SO
- **Thiết lập:** SO có `custom_van_chuyen`.
- **Các bước:** Tạo DN từ SO đó và lưu DN.
- **Kỳ vọng:** `custom_van_chuyen` trên DN được tự động sao chép từ SO.

### TC-18: DN từ nhiều SO không nên tự động sao chép
- **Thiết lập:** DN chứa items từ nhiều SO.
- **Các bước:** Lưu DN.
- **Kỳ vọng:** Tự động sao chép không chạy (trường giữ nguyên trống nếu đang trống).

## 5) Kiểm Tra Hồi Quy

### RC-01: Các nút tùy chỉnh trên Sales Order hiện có vẫn hoạt động
- Các nút in ấn phải vẫn hoạt động bình thường.

### RC-02: Tính năng Bulk Upload PO PDFs vẫn hoạt động
- Luồng upload và mapping phải vẫn vận hành bình thường.

## 6) Các Giả Định Đã Biết

1. `custom_channel` tồn tại trên Customer và được dùng cho matching channel trong quy tắc ngân hàng.
2. `custom_bank_account` tồn tại trên Sales Order từ app khác (`mbwnext_advanced_selling`).
3. `VN Province` và `VN District` được dùng làm dữ liệu master địa chỉ cho mapping tỉnh/huyện cũ.

## 7) Thứ Tự Thực Thi QA Được Đề Xuất

1. Chạy toàn bộ test case validation trước (TC-01 đến TC-09).
2. Chạy các test case logic tự động điền trên SO (TC-10 đến TC-16).
3. Chạy các test case kế thừa DN (TC-17, TC-18).
4. Chạy kiểm tra hồi quy (RC-01, RC-02).
