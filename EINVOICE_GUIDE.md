# Hướng dẫn sử dụng Hóa đơn Điện tử (mbwnext_einvoice)

> **Cập nhật theo quy định:** Từ 01/06/2025, Nghị định 70/2025/NĐ-CP không còn cho phép hủy hóa đơn điện tử.
> Khi hóa đơn sai sót phải thực hiện **thay thế** hoặc **điều chỉnh** thay vì hủy.

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Cấu hình VNPT](#2-cấu-hình-vnpt)
3. [Cấu hình Viettel SInvoice](#3-cấu-hình-viettel-sinvoice)
4. [Kiểm tra kết nối](#4-kiểm-tra-kết-nối)
5. [Phát hành hóa đơn điện tử](#5-phát-hành-hóa-đơn-điện-tử)
6. [Xem PDF hóa đơn](#6-xem-pdf-hóa-đơn)
7. [Khi hóa đơn sai sót – Thay thế / Điều chỉnh](#7-khi-hóa-đơn-sai-sót--thay-thế--điều-chỉnh)
8. [Câu hỏi thường gặp](#8-câu-hỏi-thường-gặp)
9. [Mã lỗi VNPT thường gặp](#9-mã-lỗi-vnpt-thường-gặp)

---

## 1. Tổng quan

Module `mbwnext_einvoice` tích hợp hai nhà cung cấp hóa đơn điện tử vào ERPNext:

| Nhà cung cấp | Giao thức | Chuẩn |
|---|---|---|
| **VNPT** | SOAP / ASMX (Thông tư 78/2021) | TT78 |
| **Viettel SInvoice** | REST API (JSON) | TT78 |

Mỗi công ty trong ERPNext được cấu hình riêng một nhà cung cấp. Quy trình nghiệp vụ:

```
Tạo Sales Invoice → Submit → Phát hành HĐDT → Xem PDF
                                  ↓ (sai sót)
                          Thay thế / Điều chỉnh
```

---

## 2. Cấu hình VNPT

### 2.1 Thông tin VNPT cung cấp

Khi đăng ký với VNPT, bạn nhận được:

| Thông tin | Ghi chú |
|---|---|
| URL CAdmin | VD: `https://MSTTAXCODE-tt78admin.vnpt-invoice.com.vn` |
| Tài khoản CAdmin | VD: `MSTTAXCODE_admin_demo` / `Matkhau@2025` |
| URL webservice (Portal) | VD: `https://MSTTAXCODE-tt78.vnpt-invoice.com.vn` |
| Tài khoản webservice | VD: `tenws` / `Matkhau@2025` |
| Mẫu số hóa đơn | VD: `1/001` |
| Ký hiệu hóa đơn | VD: `C26TTA` |

### 2.2 Ánh xạ vào Company

Vào **Accounting → Company → tab E-Invoice**, điền các trường sau:

| Field trên Company | Giá trị cần điền | Nguồn thông tin |
|---|---|---|
| **Sử dụng HĐDT** | ✅ Bật | — |
| **Nhà cung cấp** | `VNPT` | — |
| **Mã số thuế** | Mã số thuế công ty | — |
| **API Endpoint URL** | URL CAdmin (không kết thúc bằng `/`) | VNPT cung cấp |
| **Username** | Tài khoản **webservice** (ServiceRole) | VNPT cung cấp |
| **Password** | Mật khẩu **webservice** | VNPT cung cấp |
| **VNPT Account** *(Tài khoản nhân viên)* | Tài khoản **CAdmin** | VNPT cung cấp |
| **VNPT ACpass** *(Mật khẩu nhân viên)* | Mật khẩu **webservice** (thường là mật khẩu ws) | VNPT cung cấp |
| **Invoice Pattern** | Mẫu số hóa đơn | VNPT cung cấp |
| **Invoice Serial** | Ký hiệu hóa đơn | VNPT cung cấp |

> **Lưu ý quan trọng về VNPT ACpass:**
> `ACpass` trong SOAP là **mật khẩu webservice** (cùng với `password`), **không phải** mật khẩu đăng nhập cổng CAdmin.
> Ví dụ từ VNPT:
> ```
> <Account>MSTTAXCODE_admin_demo</Account>   ← Tài khoản CAdmin
> <ACpass>Matkhau@2025</ACpass>              ← Mật khẩu webservice
> <username>tenws</username>                 ← Tài khoản webservice
> <password>Matkhau@2025</password>          ← Mật khẩu webservice
> ```

### 2.3 Ví dụ cấu hình môi trường test VNPT

```
API Endpoint URL : https://2222222222-008-tt78democadmin.vnpt-invoice.com.vn
Username         : namduocws
Password         : Namduoc@2025
VNPT Account     : 2222222222-008_admin_demo
VNPT ACpass      : Namduoc@2025
Invoice Pattern  : 1/001
Invoice Serial   : C26TTA
```

---

## 3. Cấu hình Viettel SInvoice

### 3.1 Thông tin Viettel cung cấp

| Thông tin | Ghi chú |
|---|---|
| API URL | VD: `https://api.sinvoice.viettel.vn` |
| Username | Tên đăng nhập SInvoice |
| Password | Mật khẩu SInvoice |
| Template Code | Mã mẫu hóa đơn (Invoice Pattern) |
| Invoice Series | Ký hiệu hóa đơn |

### 3.2 Ánh xạ vào Company

| Field trên Company | Giá trị cần điền |
|---|---|
| **Sử dụng HĐDT** | ✅ Bật |
| **Nhà cung cấp** | `Viettel` |
| **Mã số thuế** | Mã số thuế công ty |
| **API Endpoint URL** | URL API SInvoice |
| **Username** | Tên đăng nhập SInvoice |
| **Password** | Mật khẩu SInvoice |
| **Invoice Pattern** | Template Code |
| **Invoice Serial** | Invoice Series |
| **Phương thức xác thực** | `Token (POST /auth/login)` *(khuyến nghị)* |

---

## 4. Kiểm tra kết nối

Sau khi cấu hình, kiểm tra kết nối ngay trên form Company:

1. Mở **Company** vừa cấu hình
2. Menu **E-Invoice** → **Test Connection**
3. Kết quả:
   - ✅ **Thành công**: hiển thị thông tin chứng thư số (VNPT) hoặc xác nhận token (Viettel)
   - ❌ **Thất bại**: kiểm tra lại URL, tài khoản, mật khẩu

**Lỗi thường gặp khi test:**

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| HTTP 400 / ERR:1 | Sai tài khoản hoặc mật khẩu | Kiểm tra Account, ACpass, username, password |
| HTTP 401 / invalid_grant | Sai username/password Viettel | Kiểm tra credentials trong Company |
| ERR:20 | Pattern/Serial không phù hợp | Kiểm tra Invoice Pattern và Invoice Serial |
| ERR:22 | Chưa đăng ký chứng thư số | Liên hệ VNPT để cấu hình chứng thư |

---

## 5. Phát hành hóa đơn điện tử

### 5.1 Điều kiện

- Sales Invoice phải ở trạng thái **Submitted** (đã submit)
- `grand_total > 0`
- Company đã được cấu hình HĐDT và bật sử dụng

### 5.2 Các bước thực hiện

1. Mở **Sales Invoice** đã submit
2. Click menu **E-Invoice** → **Create VNPT E-Invoice** (hoặc **Create Viettel E-Invoice**)
3. Xác nhận trong hộp thoại
4. Kết quả thành công sẽ hiển thị:
   - **Số hóa đơn** (einvoice_no)
   - **Fkey** (mã tham chiếu nội bộ)
5. Form tự động reload, trường **E-Invoice No** và **Created E-Invoice** được cập nhật

### 5.3 Luồng kỹ thuật VNPT (TT78)

```
ImportInvByPattern  →  Hóa đơn "Mới tạo lập" trên VNPT Portal
       ↓
PublishInvFkey      →  Hóa đơn "Đã phát hành" (ký số tự động qua HSM)
```

> **Môi trường test:** Nếu VNPT chưa cấu hình HSM tự ký, bước PublishInvFkey sẽ không thành công và hóa đơn sẽ ở trạng thái **"Mới tạo lập"** trên VNPT Portal. Cần vào Portal → chọn hóa đơn → nhấn **Phát hành** thủ công. Môi trường production với HSM cấu hình đúng sẽ tự động ký ngay.

### 5.4 Luồng kỹ thuật Viettel

```
POST /InvoiceAPI/invoice/createInvoice  →  Hóa đơn được tạo và ký số
       ↓
Nhận transactionUuid và invoiceNo
```

---

## 6. Xem PDF hóa đơn

### 6.1 Từ Sales Invoice

- Menu **View** → **E-Invoice**: mở PDF trong tab mới
- Tab **Einvoice Info** (nếu cấu hình): xem PDF preview ngay trên form

### 6.2 Từ VNPT Portal

Truy cập URL Portal do VNPT cấp → đăng nhập → tìm theo số hóa đơn hoặc ngày.

### 6.3 Lưu ý

- PDF chỉ khả dụng sau khi hóa đơn **Đã phát hành** (ký số hoàn tất)

---

## 7. Khi hóa đơn sai sót – Thay thế / Điều chỉnh

> Theo **Nghị định 70/2025/NĐ-CP** (hiệu lực 01/06/2025), **không được phép hủy hóa đơn điện tử**. Khi phát hiện sai sót phải:

### 7.1 Phân loại sai sót

| Loại sai sót | Xử lý |
|---|---|
| Sai thông tin người mua (tên, địa chỉ, MST), sai hàng hóa, số lượng, đơn giá | **Thay thế** hoặc **Điều chỉnh** |
| Sai thuế suất, tổng tiền | **Thay thế** hoặc **Điều chỉnh** |
| Hóa đơn viết sai toàn bộ, không giao hàng | **Điều chỉnh về 0** *(không phải hủy)* |

### 7.2 Hướng dẫn xử lý trong ERPNext

**Bước 1:** Xác định hóa đơn sai cần xử lý

**Bước 2:** Tạo tài liệu điều chỉnh/thay thế trong ERPNext:
- Nếu dùng **Credit Note** → tạo Sales Invoice loại Return/Credit Note
- Nếu điều chỉnh tăng → tạo Sales Invoice bổ sung với ghi chú rõ ràng

**Bước 3:** Submit tài liệu mới → phát hành HĐDT mới (thay thế/điều chỉnh)

**Bước 4:** Trên VNPT Portal hoặc Viettel Portal:
- Vào menu **Xử lý hóa đơn** → **Điều chỉnh / Thay thế**
- Chọn hóa đơn gốc cần xử lý
- Liên kết với hóa đơn điều chỉnh/thay thế đã phát hành

> **Lưu ý:** Tính năng điều chỉnh/thay thế qua API (`AdjustInvoiceAction`, `ReplaceInvoiceAction`) đang trong lộ trình phát triển. Hiện tại thực hiện trực tiếp trên Portal của nhà cung cấp.

---

## 8. Câu hỏi thường gặp

### Q: Hóa đơn phát hành xong nhưng trên VNPT Portal vẫn hiện "Mới tạo lập"?

**A:** Môi trường test chưa cấu hình HSM tự ký. Vào VNPT Portal → Danh mục hóa đơn → chọn hóa đơn → nhấn **Phát hành**. Môi trường production sẽ tự ký ngay.

---

### Q: Nhấn "Create VNPT E-Invoice" bị lỗi ERR:3?

**A:** XML gửi lên VNPT không đúng định dạng. Kiểm tra:
- Thông tin người mua (tên công ty, địa chỉ) không chứa ký tự đặc biệt lạ
- `grand_total` > 0
- Mẫu số và ký hiệu hóa đơn đúng với đăng ký

---

### Q: Lỗi ERR:7 "Username không phù hợp hoặc không tìm thấy công ty"?

**A:** Tài khoản webservice (`username`) không có quyền với Pattern/Serial đã cấu hình. Liên hệ VNPT để gán quyền.

---

### Q: Viettel báo lỗi 401?

**A:** Sai username/password. Kiểm tra lại credentials trong tab E-Invoice của Company. Đảm bảo phương thức xác thực là `Token (POST /auth/login)`.

---

### Q: PDF không tải được?

**A (VNPT):** Hóa đơn chưa "Đã phát hành". Vào Portal phát hành thủ công rồi thử lại.
**A (Viettel):** Invoice chưa có số (đang xử lý async). Chờ vài giây rồi reload form.

---

### Q: Trường "Số hóa đơn" (einvoice_no) trống sau khi phát hành?

**A (VNPT):** Hóa đơn vào queue VNPT nhưng chưa được ký (Mới tạo lập). Sau khi phát hành thủ công trên Portal, số hóa đơn sẽ xuất hiện trên VNPT nhưng cần cập nhật thủ công vào ERPNext nếu muốn lưu trữ.
**A (Viettel):** Invoice được tạo bất đồng bộ. Chờ vài giây, nhấn **Check Status** nếu có.

---

## 9. Mã lỗi VNPT thường gặp

| Mã lỗi | Mô tả | Cách xử lý |
|---|---|---|
| `ERR:1` | Sai tài khoản/mật khẩu hoặc không có quyền | Kiểm tra Account, ACpass, username, password |
| `ERR:2` | Không tìm thấy hóa đơn theo fkey | Hóa đơn chưa phát hành hoặc fkey sai |
| `ERR:3` | XML đầu vào không đúng định dạng | Kiểm tra thông tin hóa đơn |
| `ERR:4` | Không tìm thấy Pattern/Serial | Kiểm tra Invoice Pattern và Invoice Serial |
| `ERR:5` | Lỗi hệ thống VNPT (DB rollback) | Thử lại sau hoặc liên hệ VNPT |
| `ERR:6` | Dải hóa đơn không đủ hoặc không tìm thấy | Liên hệ VNPT cấp thêm dải hóa đơn |
| `ERR:7` | Username không phù hợp | Kiểm tra quyền tài khoản webservice |
| `ERR:8` | Hóa đơn đã bị điều chỉnh/thay thế | Không thể thao tác lần nữa |
| `ERR:9` | Trạng thái hóa đơn không cho phép | Kiểm tra trạng thái trên Portal |
| `ERR:12` | Hóa đơn chưa được CQT chấp nhận | Chờ CQT xác nhận |
| `ERR:13` | Fkey bị trùng | Fkey đã được dùng cho hóa đơn khác |
| `ERR:20` | Pattern/Serial không phù hợp | Kiểm tra cấu hình trong Company |
| `ERR:22` | Chưa đăng ký chứng thư số | Liên hệ VNPT thiết lập chứng thư |
| `ERR:26` | Chứng thư số hết hạn | Gia hạn chứng thư số với VNPT |
| `ERR:30` | Ngày hóa đơn nhỏ hơn hóa đơn đã phát hành | Kiểm tra ngày trên Sales Invoice |

---

## Phụ lục – Cập nhật sau khi nâng cấp app

Sau mỗi lần cập nhật `mbwnext_einvoice`:

```bash
cd /path/to/frappe-bench
bench --site your-site migrate
bench --site your-site clear-cache
bench restart
```

Nếu có thay đổi Custom Field (fixture):

```bash
bench --site your-site import-fixtures --app mbwnext_einvoice
```

---

*Tài liệu này áp dụng cho phiên bản mbwnext_einvoice tích hợp VNPT TT78 (SOAP/ASMX) và Viettel SInvoice (REST API). Cập nhật: 03/2026.*
