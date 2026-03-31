# Ghi chú triển khai VNPT E-Invoice

## Mục đích tài liệu

Tài liệu này tổng hợp lại toàn bộ quá trình phân tích, thiết kế, triển khai, debug và chuẩn hóa luồng tích hợp VNPT E-Invoice trong `mbwnext_einvoice`.

Mục tiêu:

- Giúp team Dev mới hiểu nhanh bài toán.
- Ghi lại các quyết định kỹ thuật đã chốt.
- Ghi lại các lỗi đã gặp và cách xử lý.
- Làm tài liệu handover để bảo trì, mở rộng và đào tạo nội bộ.

Tài liệu này dành cho Dev, Tech Lead, BA kỹ thuật; không phải tài liệu hướng dẫn end-user.

## 1. Bài toán nghiệp vụ

Ban đầu module `mbwnext_einvoice` đã có hướng tích hợp hóa đơn điện tử, nhưng phần VNPT được xây dựng theo một bộ tài liệu cũ và một hướng API khác.

Sau khi đối chiếu tài liệu mới do VNPT cung cấp, đã xác nhận:

- VNPT đang dùng SOAP/ASMX Web Service.
- Chuẩn dữ liệu hóa đơn là TT78.
- Luồng phát hành không phải REST JSON mà là XML gửi qua SOAP 1.1.
- Có nhiều service ASMX khác nhau:
  - `publishservice.asmx`
  - `businessservice.asmx`
  - `portalservice.asmx`

Việc này dẫn đến quyết định:

- Không vá tiếp trên client REST cũ.
- Cần viết lại client VNPT theo đúng mô hình SOAP/ASMX.

## 2. Phạm vi đã thực hiện

Phạm vi thực tế đã làm cho VNPT:

1. Đọc và phân tích tài liệu VNPT, mẫu `curl`, WSDL.
2. Viết lại client VNPT từ REST sang SOAP.
3. Bổ sung cấu hình Company cho đúng mô hình xác thực VNPT.
4. Viết lại XML builder theo chuẩn TT78.
5. Tích hợp phát hành hóa đơn từ `Sales Invoice`.
6. Lấy PDF hóa đơn từ VNPT.
7. Thêm nút `Test Connection` trên Company.
8. Thêm nút tạo hóa đơn trên `Sales Invoice`.
9. Debug các lỗi HTTP và SOAP trong quá trình kết nối.
10. Chuẩn hóa hướng xử lý theo quy định mới:
   - Bỏ luồng hủy hóa đơn.
   - Chuyển hướng nghiệp vụ sang thay thế hoặc điều chỉnh.

## 3. Tài liệu đầu vào và thông tin cần xác minh

Trong quá trình làm, cần thu thập và xác minh tối thiểu các thông tin sau:

- URL CAdmin.
- URL Portal.
- URL webservice thực tế.
- Tài khoản CAdmin.
- Tài khoản webservice.
- Pattern.
- Serial.
- Mẫu XML TT78.
- Mẫu `SOAPAction`.
- Danh sách method trong WSDL.

Kinh nghiệm:

- Không được giả định VNPT chỉ dùng một kiểu auth.
- Không được giả định tên parameter theo logic cũ.
- Phải đối chiếu trực tiếp với WSDL và mẫu request.

## 4. Mô hình xác thực VNPT đã chốt

VNPT SOAP dùng 2 cặp tài khoản:

1. Tài khoản nhân viên:
   - `Account`
   - `ACpass`

2. Tài khoản webservice:
   - `username`
   - `password`

Mapping trong `Company`:

| Field Company | Ý nghĩa |
|---|---|
| `provider` | Chọn `VNPT` |
| `tax_code` | MST đơn vị |
| `api_endpoint_url` | Base URL của hệ thống VNPT |
| `account` | VNPT Account |
| `acpass` | VNPT ACpass |
| `username` | Tài khoản webservice |
| `password` | Mật khẩu webservice |
| `invoice_pattern` | Mẫu số hóa đơn |
| `invoice_serial` | Ký hiệu hóa đơn |

Lưu ý quan trọng:

- `ACpass` không được map sai sang mật khẩu portal nếu tài liệu webservice quy định khác.
- Trong bộ tài liệu và mẫu `curl` đã test, `ACpass` trùng với mật khẩu webservice.

## 5. Kiến trúc kỹ thuật đã triển khai

### 5.1 Các file chính

| File | Vai trò |
|---|---|
| `integrations/vnpt/config.py` | Nạp cấu hình VNPT từ Company |
| `integrations/vnpt/client.py` | SOAP client cho VNPT |
| `integrations/vnpt/service.py` | Nghiệp vụ phát hành, PDF, build XML |
| `mbwnext_einvoice/api/vnpt.py` | API callable từ UI/Frappe |
| `controllers/js/company.js` | Nút `Test Connection` trên Company |
| `controllers/js/sales_invoice.js` | Nút tạo hóa đơn trên Sales Invoice |
| `fixtures/custom_field.json` | Các field cấu hình Company và field hiển thị trên Sales Invoice |

### 5.2 Tách lớp rõ ràng

Thiết kế được tách thành 3 lớp:

1. `config.py`
   - Đọc dữ liệu Company
   - Giải mã password
   - Validate provider đang active

2. `client.py`
   - Tạo SOAP envelope
   - Gọi HTTP request
   - Parse SOAP response
   - Throw lỗi HTTP/SOAP có nghĩa

3. `service.py`
   - Mapping ERPNext sang XML TT78
   - Điều phối các method của client
   - Trả kết quả nghiệp vụ

Quyết định này giúp:

- Dễ test.
- Dễ debug.
- Không trộn nghiệp vụ với transport layer.

## 6. Vấn đề lớn nhất: API cũ sai hướng

Lúc đầu implementation đi theo hướng REST API, nhưng tài liệu mới của VNPT cho thấy:

- Request phải gửi dạng XML.
- XML được bọc trong SOAP 1.1.
- `SOAPAction` phải đúng tên method trong WSDL.
- Service URL là ASMX, không phải REST endpoint.

Do đó đã viết lại:

- `_build_soap_envelope()`
- `_call()`
- Các method như:
  - `import_inv_by_pattern`
  - `get_cert_info`
  - `download_new_inv_pdf_fkey`
  - `download_inv_pdf_fkey_no_pay`
  - `download_inv_pdf_fkey`
  - `publish_inv_fkey`

## 7. Xử lý tính không ổn định của hệ thống VNPT

Trong quá trình debug, đã gặp các vấn đề thực tế sau:

### 7.1 URL service nhạy cảm chữ hoa/chữ thường

Có môi trường VNPT yêu cầu:

- `publishservice.asmx`
- hoặc `Publishservice.asmx`

Đã xử lý bằng cách sinh nhiều URL candidate trong client.

### 7.2 `SOAPAction` không đồng nhất

Có môi trường nhận:

- `"http://tempuri.org/GetCertInfo"`

nhưng có môi trường lại chỉ nhận:

- `http://tempuri.org/GetCertInfo`

Đã xử lý bằng cách retry với cả 2 biến thể.

### 7.3 Cần `Accept: text/xml`

Một số request HTTP 400 được sửa sau khi thêm header:

- `Accept: text/xml`

### 7.4 XML envelope không được dư whitespace đầu file

Envelope được `.strip()` để tránh một số case IIS/ASMX parse khắt khe.

## 8. XML TT78 builder đã thay đổi như thế nào

Implementation cũ không đúng schema TT78.

Đã viết lại thành cấu trúc:

```xml
<DSHDon>
  <HDon>
    <key>...</key>
    <DLHDon>
      <TTChung>...</TTChung>
      <NDHDon>
        <NBan>...</NBan>
        <NMua>...</NMua>
        <DSHHDVu>...</DSHHDVu>
        <TToan>...</TToan>
      </NDHDon>
    </DLHDon>
  </HDon>
</DSHDon>
```

### 8.1 Mapping người bán

Người bán (`NBan`) lấy từ:

- `Company.company_name`
- `Company.tax_id` hoặc `tax_code`
- địa chỉ, điện thoại, email công ty

### 8.2 Mapping người mua

Người mua (`NMua`) được chốt theo quy tắc:

1. Ưu tiên dữ liệu trên `Sales Invoice` nếu đã có.
2. Nếu thiếu thì fallback từ `Customer`, `Address`, `Contact`.

Đã bổ sung helper:

- `_buyer_address_and_mobile()`
- `_buyer_email()`
- `_buyer_tax_id()`

Mục tiêu:

- Giữ tính chất "snapshot" của chứng từ.
- Vẫn lấy được dữ liệu master khi SI chưa chốt đầy đủ.

### 8.3 Fkey

Đã chốt quy tắc:

- `fkey = Sales Invoice.name`

Lý do:

- Dễ truy vết.
- Duy nhất trong hệ thống.
- Dễ parse kết quả publish.

## 9. Luồng phát hành hóa đơn đã chốt

Luồng nghiệp vụ hiện tại:

1. User submit `Sales Invoice`.
2. Truy cập menu `E-Invoice`.
3. Chọn tạo hóa đơn VNPT.
4. Backend build XML TT78.
5. Gọi `ImportInvByPattern`.
6. Parse kết quả.
7. Lưu:
   - `created_einvoice = 1`
   - `einvoice_no`
   - `einvoice_uuid` (thực chất là fkey hoặc transaction key nội bộ)

Lưu ý:

- Trường `einvoice_uuid` đang được dùng chung cho nhiều nhà cung cấp.
- Với VNPT, giá trị trong trường này chính là `fkey`.

## 10. Luồng PDF đã chốt

Do VNPT có nhiều method tải PDF tùy trạng thái hóa đơn, đã chốt thứ tự fallback:

1. `downloadNewInvPDFFkey`
2. `downloadInvPDFFkeyNoPay`
3. `downloadInvPDFFkey`

Nếu không có PDF base64 thì có thể fallback sang portal URL.

## 11. Các lỗi đã gặp và cách xử lý

### 11.1 Lỗi test connection

#### Triệu chứng

`GetCertInfo` trả HTTP 400.

#### Nguyên nhân có khả năng

- Sai URL service do phân biệt hoa/thường.
- `SOAPAction` sai format.
- Header chưa đủ.
- Envelope có ký tự thừa.

#### Cách xử lý

- Thêm service URL candidates.
- Retry `SOAPAction` có và không có dấu nháy.
- Thêm `Accept: text/xml`.
- `strip()` envelope.

### 11.2 Lỗi company truyền sai tham số

#### Triệu chứng

API `test_vnpt_connection` truyền sai `company`.

#### Xử lý

Sửa API gọi `vnpt_service.test_connection(company=company)` đúng nghĩa.

### 11.3 Lỗi hủy hóa đơn `ERR:2`

#### Triệu chứng

`cancelInv: Chuỗi fkey/token không đúng định dạng hoặc không tìm thấy hóa đơn`

#### Xử lý đã từng thử

- Kiểm tra lại parameter `fkey` thay vì `Fkey`.
- Test trực tiếp bằng `curl`.
- Đối chiếu WSDL.

#### Kết luận nghiệp vụ mới

Theo quy định mới, luồng hủy hóa đơn không còn được sử dụng.

Đã xóa toàn bộ:

- API hủy VNPT
- API hủy Viettel
- auto cancel hook khi cancel `Sales Invoice`
- nút hủy trên UI

Team không được mở lại luồng này nếu không có quyết định nghiệp vụ mới.

## 12. Quyết định nghiệp vụ quan trọng đã chốt

### 12.1 Không hủy hóa đơn nữa

Theo trao đổi nghiệp vụ:

- Từ 01/06, không hủy hóa đơn điện tử.
- Nếu sai phải làm thay thế hoặc điều chỉnh.

Hệ quả kỹ thuật:

- Bỏ toàn bộ luồng cancel.
- Tài liệu và UI phải đồng bộ theo hướng này.

### 12.2 Hiển thị Fkey

Đã xác định:

- `einvoice_uuid` đang lưu Fkey của VNPT.
- Trường này trước đây bị ẩn.

Khi cần làm rõ trên UI, nên:

- hiển thị rõ tên nghiệp vụ, ví dụ `VNPT Fkey`
- hoặc đổi label theo provider

Nếu team tiếp tục dùng chung `einvoice_uuid` cho nhiều provider, cần ghi rõ trong tài liệu.

## 13. Các thay đổi trên UI

### 13.1 Company

Đã thêm:

- nút `Test Connection`

Phân nhánh theo provider:

- VNPT -> gọi API test VNPT
- Viettel -> gọi API test Viettel

### 13.2 Sales Invoice

Đã thêm:

- nút tạo hóa đơn theo provider
- nút xem PDF

Đã loại bỏ:

- nút hủy hóa đơn

## 14. Danh sách field quan trọng trên Sales Invoice

| Field | Ý nghĩa |
|---|---|
| `created_einvoice` | Đã tạo hóa đơn điện tử hay chưa |
| `einvoice_no` | Số hóa đơn |
| `einvoice_uuid` | Fkey VNPT hoặc transaction UUID tùy provider |
| `einvoice_pdf_preview` | Khu vực preview PDF |

## 15. Checklist debug cho Dev mới

Khi VNPT lỗi, kiểm tra theo thứ tự:

1. `Company.provider == VNPT`
2. `use_einvoice` đã bật
3. `api_endpoint_url` đúng base URL
4. `account`, `acpass`, `username`, `password` đúng
5. `invoice_pattern`, `invoice_serial` đúng
6. `Test Connection` pass
7. XML TT78 có đủ:
   - `NBan`
   - `NMua`
   - `DSHHDVu`
   - `TToan`
8. `Sales Invoice` có `grand_total > 0`
9. Kiểm tra Error Log trong Frappe
10. Nếu cần, test SOAP bằng `curl` trước khi sửa code

## 16. Nguyên tắc khi mở rộng tiếp

Nếu team tiếp tục phát triển VNPT, cần giữ các nguyên tắc sau:

1. Không hardcode theo một môi trường duy nhất.
2. Luôn đối chiếu WSDL khi thêm method mới.
3. Tách `client` và `service`.
4. Ưu tiên log để debug được request và response logic.
5. Không đưa nghiệp vụ hủy hóa đơn quay trở lại.
6. Nếu thêm thay thế hoặc điều chỉnh, làm theo hướng:
   - business API rõ ràng
   - mapping XML rõ ràng
   - tài liệu nghiệp vụ cập nhật song song

## 17. Đề xuất backlog tiếp theo

Để hệ thống ổn định hơn, đề xuất backlog sau:

1. Hiển thị `VNPT Fkey` rõ ràng trên form `Sales Invoice`.
2. Thêm API debug để xuất `xmlInvData` cho một `Sales Invoice`.
3. Thêm chế độ preview XML TT78 trước khi phát hành.
4. Thêm tài liệu sequence diagram cho luồng issue/PDF.
5. Triển khai luồng thay thế, điều chỉnh theo quy định mới.
6. Thêm test case cho:
   - buyer MST fallback
   - parse publish result
   - config mapping

## 18. Kết luận

Giá trị lớn nhất của đợt implementation này không chỉ là "gọi API thành công", mà là:

- xác định đúng mô hình kỹ thuật của VNPT
- chuyển hệ thống từ hướng REST sai sang SOAP đúng
- chốt được cấu trúc XML TT78 đúng nghĩa
- tách lớp kiến trúc rõ ràng để bảo trì
- loại bỏ luồng nghiệp vụ không còn phù hợp (hủy hóa đơn)

Tài liệu này cần được cập nhật tiếp khi team bắt đầu làm:

- thay thế hóa đơn
- điều chỉnh hóa đơn
- đồng bộ trạng thái hóa đơn từ portal về ERPNext

