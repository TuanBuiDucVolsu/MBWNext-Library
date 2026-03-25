# Huong dan su dung MISA eInvoice (ERPNext)

Tai lieu nay huong dan cau hinh va su dung MISA MeInvoice trong app `mbwnext_einvoice`.

## 1) Dieu kien truoc khi cau hinh

- Da cai app `mbwnext_einvoice`.
- Da co tai khoan MISA MeInvoice (sandbox hoac production).
- Da duoc MISA cap bo thong tin API:
  - `AppID`
  - `Tax Code`
  - `Username`
  - `Password`
- Da tao va kich hoat mau hoa don tren portal MISA (trang thai `Su dung`).

## 2) Cau hinh Company > E-Invoice Settings

Vao `Company` > tab `E-Invoice Settings` va cau hinh:

- `Use E-Invoice`: bat.
- `E-Invoice Provider`: chon `MISA`.
- `Tax Code (MST)`: ma so thue don vi.
- `API URL`:
  - Sandbox: `https://testapi.meinvoice.vn/api/integration`
  - Production: `https://api.meinvoice.vn/api/integration`
- `Username`, `Password`: thong tin dang nhap MISA.
- `Invoice Serial`: vi du `1C26TNP` hoac `1C26MTM`.
- `MISA AppID`: AppID duoc MISA cap.
- `Signing Method (SignType)`:
  - `2`: HSM co chu ky so hien thi (can Certificate SN)
  - `5`: khong ky (thuong dung cho hoa don MTT/POS)
- `Digital Certificate SN`:
  - chi can khi `SignType = 2` hoac `3`
  - bam nut `Get MISA Certificates` de lay va dien nhanh.
- `Invoice With Tax Authority Code`: tick neu la hoa don co ma CQT (ky tu thu 2 cua ky hieu la `C`).

## 3) Test ket noi MISA

Tai `Company`:

- Bam `E-Invoice` > `Test MISA Connection`.
- Neu thanh cong, he thong hien:
  - Provider
  - Base URL
  - Tax Code
  - So luong template tim thay

Neu loi `UnAuthorize` / `MisaIdError`:
- Kiem tra lai `AppID`, `Tax Code`, `Username`, `Password` co dung cung 1 moi truong khong.

## 4) Phat hanh hoa don tu Sales Invoice

Tai `Sales Invoice` da submit:

- Bam `E-Invoice` > `Create MISA E-Invoice`.
- Neu thanh cong:
  - `Created Einvoice` = checked
  - Luu `E-Invoice No`
  - Luu `Transaction ID` vao `einvoice_uuid`

## 5) Xem PDF hoa don

Tai tab `E-Invoice Info` tren `Sales Invoice`:

- Neu MISA tra ve du lieu PDF base64: xem truc tiep trong ERPNext.
- Neu MISA chi tra ve link: he thong hien nut `Open E-Invoice PDF on portal`.

Luu y: mot so thoi diem MISA co the tra ve link portal/download thay vi base64, day la hanh vi binh thuong.

## 6) Quy tac ky hieu hoa don va SignType

Theo MISA:

- Ky tu thu 2 cua ky hieu:
  - `C`: hoa don co ma CQT
  - `K`: hoa don khong ma
- Ky tu thu 5 cua ky hieu:
  - `T`: hoa don thuong
  - `M`: hoa don tu may tinh tien (MTT/POS)

Mapping khuyen nghi:

- `...T...` (hoa don thuong) -> `SignType = 2` + `Certificate SN`
- `...M...` (hoa don MTT) -> `SignType = 5`

## 7) Luu y ve nam trong ky hieu

MISA co the tu dong chuan hoa nam trong ky hieu theo ngay hoa don (`InvDate`).

Vi du:
- Gui `1C25TNP` cho hoa don nam 2026
- MISA co the luu thanh `1C26TNP`

Day la hanh vi dung theo nghiep vu MISA.

## 8) Loi thuong gap va cach xu ly

### `APINotSupportTypeInvoice`
- Nguyen nhan: `SignType` khong phu hop loai ky hieu.
- Cach xu ly:
  - Ky hieu `...T...` -> dung `SignType = 2`
  - Ky hieu `...M...` -> dung `SignType = 5`

### `TemplateIsNotUsing`
- Nguyen nhan: mau hoa don tren MISA chua o trang thai `Su dung`.
- Cach xu ly: vao portal MISA, kich hoat mau hoa don.

### `InvalidAppID` / `MisaIdError`
- Nguyen nhan: AppID sai hoac khong khop bo thong tin sandbox/product.
- Cach xu ly: lien he MISA lay lai bo thong tin API dung.

### Khong xem duoc PDF tren portal (timeout)
- Thu mo lai sau it phut.
- Kiem tra dung moi truong (sandbox/product).
- Dung tra cuu tren trang quan ly MISA bang `Invoice No` hoac `Transaction ID`.

## 9) Checklist cau hinh nhanh

- [ ] Provider = `MISA`
- [ ] API URL dung moi truong
- [ ] AppID/TaxCode/Username/Password dung
- [ ] Invoice Serial dung loai (`T` hoac `M`)
- [ ] SignType phu hop voi ky hieu
- [ ] Mau hoa don tren MISA dang `Su dung`
- [ ] (Neu SignType=2) da chon `Digital Certificate SN`

---

Neu can, co the bo sung them SOP van hanh (quy trinh doi serial theo nam, retry khi loi, va doi soat MISA-ERP cuoi ngay).
