# Test Case — BOM Template & Tạo BOM Tự Động (Phần I)

**HKLED · Manufacturing · QA**

Bộ test case tay (manual) cho PHẦN I của tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED: BOM Component, BOM Component Table, BOM Rule, BOM Template và engine "Tạo BOM Tự Động" trên Kế hoạch sản xuất.

- App: `mbwnext_hkled`
- Site: `hkled.com`
- Phạm vi: Phần I
- Quyền: System Manager / Manufacturing Manager

## ⚠ Bắt buộc đọc trước khi test

**Không bao giờ dùng mã Item / mã BOM Template trùng với dữ liệu đang tồn tại trong hệ thống.** Nút "Tạo BOM Tự Động" sẽ tạo BOM mới và gán làm `Default` ngay lập tức — nếu Item đó đã có BOM Default đang được Work Order thật sử dụng, thao tác test có thể ghi đè lên dữ liệu sản xuất thật.

Luôn dùng đúng bộ dữ liệu mẫu cô lập ở mục "Dữ liệu mẫu" bên dưới (tiền tố `RM-`, item family `Den pha` / `Module LED Test` / `Chip LED Module Test`). Nếu cần tạo thêm dữ liệu test mới, đặt tên rõ ràng không trùng với item thật, và kiểm tra bằng `bench --site hkled.com console` trước khi bấm nút tạo BOM.

## Mục lục

1. [Dữ liệu mẫu đã chuẩn bị](#dữ-liệu-mẫu-đã-chuẩn-bị-sẵn)
2. [A — BOM Component](#a--bom-component)
3. [B — BOM Component Table](#b--bom-component-table)
4. [C — BOM Rule](#c--bom-rule)
5. [D — BOM Template](#d--bom-template)
6. [E — Popup "Tạo Rule"](#e--popup-tạo-rule)
7. [F — Engine Tạo BOM Tự Động](#f--engine-tạo-bom-tự-động)
8. [Bảng tổng hợp kết quả](#bảng-tổng-hợp-kết-quả)

---

## Dữ liệu mẫu đã chuẩn bị sẵn

Bộ dữ liệu demo cô lập, mô phỏng đúng ví dụ trong tài liệu nghiệp vụ (Đèn pha → Module LED → Chip LED Module) nhưng dùng item riêng để không đụng dữ liệu sản xuất thật.

```
Den pha-200W-Tr-DN-HKLED (BOM Template: BT-DEN-PHA)
├── RM-NGUON-50W          × 4  — theo Rule (Công Suất=200W)
├── RM-VO-DEN-200W        × 1  — cố định
├── RM-OC-VIT             × 3  — cố định
├── RM-DAY-DIEN           × 32 — cố định
└── Module LED Test-Tr-Led Philips × 1 — theo Rule (BOM Template: BT-MODULE-LED-TEST)
    ├── Chip LED Module Test-Tr-Led Philips × 1 — theo Rule (BOM Template: BT-CHIP-LED-MODULE-TEST)
    │   ├── RM-CHIP-NGUYEN-LIEU  × 20 — cố định
    │   └── RM-PCB               × 1  — cố định
    ├── RM-TAN-NHIET       × 1 — cố định
    ├── RM-LENS            × 1 — cố định
    └── RM-GIOANG          × 1 — cố định
```

**Item Template & đặc tính**

| Item Template | Đặc tính | Giá trị | Biến thể mẫu |
|---|---|---|---|
| `Den pha` | Cong Suat HKLED | 50W, 200W | `Den pha-200W-Tr-DN-HKLED`<br>200W · Trang · Done nho |
| | Anh Sang HKLED | Trang, Vang | |
| | Loai Nguon HKLED | Done nho | |
| `Module LED Test` | Anh Sang HKLED / Loai Chip HKLED | Trang / Led Philips | `Module LED Test-Tr-Led Philips` |
| `Chip LED Module Test` | Anh Sang HKLED / Loai Chip HKLED | Trang / Led Philips | `Chip LED Module Test-Tr-Led Philips` |

**BOM Component**

| BOM Component | Dùng trong Template | Kiểu |
|---|---|---|
| `Nguon` | BT-DEN-PHA | Theo Rule |
| `Module LED` | BT-DEN-PHA | Theo Rule |
| `Vo den pha`, `Oc vit`, `Day dien` | BT-DEN-PHA | Cố Định |
| `Chip LED Module` | BT-MODULE-LED-TEST | Theo Rule |
| `Tan nhiet module`, `Lens`, `Gioang cao su` | BT-MODULE-LED-TEST | Cố Định |
| `Chip nguyen lieu`, `PCB` | BT-CHIP-LED-MODULE-TEST | Cố Định |

> **Lưu ý cho case tổ hợp nhiều biến thể (TC-E-04):** Cần bổ sung sẵn item biến thể `Den pha-200W-Vang-DN-HKLED` (Công Suất=200W, Ánh Sáng=Vàng) trong dữ liệu mẫu, song song với `Den pha-200W-Tr-DN-HKLED` đã có, để test đúng kịch bản "chọn nhiều giá trị đặc tính → sinh nhiều dòng".

---

## A — BOM Component

Danh mục gốc đặt tên thành phần BOM, không phụ thuộc Item hay Template.

### TC-A-01 — Tạo mới BOM Component hợp lệ `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Vào `BOM Component` → New | — |
| 2 | Nhập Tên Thành Phần | `Vo den pha test` |
| 3 | Save | — |

**Kết quả mong đợi:** Lưu thành công; tên bản ghi (docname) chính là giá trị vừa nhập (autoname theo field).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-02 — Trùng tên → chặn unique `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo BOM Component mới, nhập đúng tên đã có ở TC-A-01 | `Vo den pha test` |
| 2 | Save | — |

**Kết quả mong đợi:** Báo lỗi trùng (Duplicate entry), không tạo thêm bản ghi mới.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## B — BOM Component Table

Bảng thành phần bên trong BOM Template. Test trực tiếp trên tab "Thông Tin Chính" của một BOM Template (có thể dùng bản nháp chưa lưu).

### TC-B-01 — Kiểu "Cố Định" thiếu Mặt Hàng / Số Lượng `Cao`

Điều kiện: đang mở BOM Template nháp, có 1 dòng trong Bảng Thành Phần BOM.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Chọn Thành Phần BOM bất kỳ, ví dụ | `Vo den pha` |
| 2 | Chọn Kiểu Thành Phần | `Cố Định` |
| 3 | Để trống Mặt Hàng, Số Lượng để trống/0, bấm Save | — |

**Kết quả mong đợi:** Lỗi rõ ràng: "Mặt Hàng bắt buộc khi Kiểu Thành Phần là Cố Định" hoặc "Số Lượng phải lớn hơn 0…"; không lưu được.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-02 — Kiểu "Theo Rule" tự ẩn và xoá Mặt Hàng / Số Lượng `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Trên dòng vừa nhập ở TC-B-01, đổi Kiểu Thành Phần sang | `Theo Rule` |
| 2 | Quan sát giao diện dòng đó | — |
| 3 | Save Template | — |

**Kết quả mong đợi:** Cột Số Lượng & Mặt Hàng biến mất khỏi dòng, nút `Tạo Rule` xuất hiện; sau khi Save, 2 field này được server tự xoá về rỗng dù trước đó có dữ liệu.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-03 — Trùng Thành Phần BOM trong cùng bảng `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm dòng thứ 2 trong Bảng Thành Phần BOM, chọn lại đúng | `Vo den pha` (đã có ở dòng 1) |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi "Thành Phần BOM … bị trùng trong Bảng Thành Phần BOM", kèm số dòng (Row #).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-04 — Filter Mặt Hàng không cho chọn item Template `Trung bình`

Điều kiện: dòng Thành Phần BOM đang ở Kiểu `Cố Định`. Bổ sung vì tài liệu mục 3 nêu rõ "Filter không cho chọn những mặt hàng template" nhưng chưa có test case tương ứng.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở cột Mặt Hàng, mở dropdown, gõ tên 1 item Template | ví dụ `Den pha` (Has Variants=1) |

**Kết quả mong đợi:** Item Template không xuất hiện trong danh sách gợi ý (link filter `has_variants=0`); chỉ các item thường (nguyên vật liệu) mới được chọn.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-05 — Bỏ trống Kiểu Thành Phần `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm dòng mới trong Bảng Thành Phần BOM, chọn Thành Phần BOM nhưng để trống Kiểu Thành Phần | — |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi bắt buộc nhập (Mandatory field) cho Kiểu Thành Phần; không lưu được.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## C — BOM Rule

Test trong tab "Công Thức Thành Phần" của BOM Template, bảng Công Thức BOM.

### TC-C-01 — Mặt Hàng Sản Xuất chỉ được chọn item biến thể `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm dòng mới trong bảng Công Thức BOM | — |
| 2 | Mở dropdown Mặt Hàng Sản Xuất, gõ tên 1 item thường (không phải biến thể) | ví dụ item bất kỳ có `Has Variants=0` và `Variant Of` trống |

**Kết quả mong đợi:** Item đó không xuất hiện trong danh sách gợi ý (link filter `variant_of is set`).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-02 — Nguyên Vật Liệu không được là item Template `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở cột Nguyên Vật Liệu, mở dropdown, gõ tên 1 item Template | ví dụ `Den pha` (Has Variants=1) |

**Kết quả mong đợi:** Item Template không xuất hiện trong gợi ý (link filter `has_variants=0`); nếu ép set bằng API, Save sẽ báo "không được là mặt hàng Template".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-03 — Trùng cặp (Mặt Hàng Sản Xuất, Thành Phần BOM) `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm 2 dòng Công Thức BOM cùng cặp | `Den pha-200W-Tr-DN-HKLED` + `Nguon` |
| 2 | Điền NVL/SL khác nhau ở 2 dòng, Save | — |

**Kết quả mong đợi:** Lỗi "Công Thức BOM bị trùng cho Mặt Hàng Sản Xuất … và Thành Phần BOM …".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-04 — Bỏ trống Số Lượng trong Công Thức BOM `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm dòng Công Thức BOM đầy đủ Mặt Hàng Sản Xuất, Thành Phần BOM, Nguyên Vật Liệu nhưng để trống Số Lượng | — |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi bắt buộc nhập (Mandatory field) cho Số Lượng; không lưu được.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## D — BOM Template

Ràng buộc ở cấp document chính.

### TC-D-01 — Mặt Hàng Cha phải là item Template (Has Variants) `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New BOM Template | — |
| 2 | Mở dropdown Mặt Hàng Cha, gõ tên 1 item thường | item bất kỳ có `Has Variants=0` |

**Kết quả mong đợi:** Item không xuất hiện trong gợi ý (link filter); nếu ép set qua API, Save báo "phải là mặt hàng Template (Has Variants)".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-02 — Chỉ 1 BOM Template hoạt động / item cha `Cao`

Điều kiện: **BT-DEN-PHA** đang `Hoạt Động = 1` cho item cha `Den pha`.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo BOM Template mới | Mã: `BT-DEN-PHA-2` |
| 2 | Chọn Mặt Hàng Cha | `Den pha` |
| 3 | Tích Hoạt Động, thêm 1 dòng thành phần bất kỳ, Save | — |

**Kết quả mong đợi:** Lỗi "Mặt hàng cha Den pha đã có BOM Template BT-DEN-PHA đang hoạt động". Bỏ tích Hoạt Động thì lưu được bình thường (2 template có thể tồn tại song song miễn chỉ 1 cái active).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-03 — Trùng Mã BOM Template → chặn unique `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo BOM Template mới, nhập đúng mã đã tồn tại | `BT-DEN-PHA` |
| 2 | Điền các field bắt buộc còn lại, Save | — |

**Kết quả mong đợi:** Báo lỗi trùng (Duplicate entry) trên Mã BOM Template; không tạo thêm bản ghi mới.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## E — Popup "Tạo Rule"

Luồng UI 2 bước để tự sinh dòng Công Thức BOM từ tổ hợp đặc tính. Test trên `BT-DEN-PHA`.

### TC-E-01 — Popup hiển thị đủ đặc tính của item cha `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở `BT-DEN-PHA`, tab Thông Tin Chính | — |
| 2 | Ở dòng thành phần `Module LED` (Theo Rule), bấm nút | `Tạo Rule` |

**Kết quả mong đợi:** Popup "Chọn Điều Kiện" hiện đủ 3 đặc tính của `Den pha`: `Cong Suat HKLED` (50W, 200W), `Anh Sang HKLED` (Trang, Vang), `Loai Nguon HKLED` (Done nho) — mỗi đặc tính là 1 nhóm multi-check.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-E-02 — Chọn tổ hợp → sinh đúng dòng Công Thức BOM `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở popup Chọn Điều Kiện, tick | Cong Suat HKLED = 200W, Anh Sang HKLED = Trang |
| 2 | Bấm | `Tiếp Theo` |
| 3 | Popup Chọn Nguyên Vật Liệu: nhập | Item = `RM-NGUON-50W`, Số Lượng = 4 |
| 4 | Bấm | `Xác Nhận` |

**Kết quả mong đợi:** Alert xanh "Đã thêm 1 dòng…"; bảng Công Thức BOM có thêm đúng 1 dòng: Mặt Hàng Sản Xuất=`Den pha-200W-Tr-DN-HKLED`, Thành Phần BOM=`Nguon`, NVL=`RM-NGUON-50W`, SL=4 (đúng 1 dòng vì chỉ 1 biến thể khớp tổ hợp 200W+Trắng trong dữ liệu mẫu).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-E-03 — Chạy lại cùng tổ hợp → cập nhật, không tạo dòng trùng `Trung bình`

Điều kiện: đã thực hiện xong TC-E-02.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Lặp lại TC-E-02 với cùng tổ hợp (200W, Trang) | — |
| 2 | Ở bước chọn NVL, đổi Số Lượng | Số Lượng = 5 (giữ nguyên Item) |
| 3 | Xác Nhận | — |

**Kết quả mong đợi:** Alert "cập nhật 1 dòng"; dòng cũ trong bảng Công Thức BOM được cập nhật SL=5 tại chỗ, **không** sinh thêm dòng mới; Save Template không báo lỗi trùng.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-E-04 — Tick nhiều giá trị cùng 1 đặc tính → sinh nhiều dòng cho nhiều biến thể `Cao`

Đúng ví dụ minh hoạ trong tài liệu nghiệp vụ (mục 5): "chọn công suất 200W, ánh sáng Trắng, Vàng → tự động thêm dòng cho công suất 200W VÀ ánh sáng Trắng, công suất 200W VÀ ánh sáng Vàng". Cần dữ liệu mẫu `Den pha-200W-Vang-DN-HKLED` (xem lưu ý ở mục Dữ liệu mẫu).

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở `BT-DEN-PHA`, ở dòng `Nguon` (Theo Rule), bấm `Tạo Rule` | — |
| 2 | Ở popup Chọn Điều Kiện, tick | Cong Suat HKLED = 200W; Anh Sang HKLED = Trang **và** Vang (2 giá trị cùng lúc) |
| 3 | Bấm `Tiếp Theo` | — |
| 4 | Popup Chọn Nguyên Vật Liệu: nhập | Item = `RM-NGUON-50W`, Số Lượng = 4 |
| 5 | Bấm `Xác Nhận` | — |

**Kết quả mong đợi:** Hệ thống tính tổ hợp (cartesian product): {200W} × {Trắng, Vàng} → 2 combo. Alert phải báo thêm/cập nhật **2 dòng** (không phải 1); mỗi dòng có `item_to_manufacture` khác nhau (biến thể Trắng và biến thể Vàng) nhưng cùng NVL=`RM-NGUON-50W`, SL=4. Đây là case khác biệt cốt lõi so với TC-E-02 (chỉ 1 combo → 1 dòng).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-E-05 — Tổ hợp không khớp biến thể nào `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở popup Chọn Điều Kiện, tick tổ hợp không tồn tại biến thể nào khớp | ví dụ Cong Suat HKLED = 50W, Anh Sang HKLED = Vang (nếu dữ liệu mẫu không có biến thể này) |
| 2 | Tiếp Theo → nhập NVL/SL bất kỳ → Xác Nhận | — |

**Kết quả mong đợi:** Không có dòng nào được thêm/cập nhật vào bảng Công Thức BOM; hệ thống nên hiển thị thông báo rõ ràng kiểu "Không tìm thấy biến thể khớp điều kiện" thay vì im lặng hoặc lỗi khó hiểu.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## F — Engine Tạo BOM Tự Động

Test trên màn hình **Kế hoạch sản xuất** (Production Plan) → bảng "Select Items to Manufacture" → nút `Tạo BOM Tự Động` ở mỗi dòng.

### TC-F-01 — Item không phải biến thể `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Production Plan mới, thêm dòng với 1 item không phải biến thể | item bất kỳ, `Variant Of` trống |
| 2 | Bấm nút `Tạo BOM Tự Động` trên dòng đó | — |

**Kết quả mong đợi:** Lỗi "Không xác định được Item cha cho …".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-F-02 — Item cha chưa có BOM Template active `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạm bỏ tích Hoạt Động trên `BT-DEN-PHA`, Save | — |
| 2 | Ở Production Plan, thêm dòng item `Den pha-200W-Tr-DN-HKLED`, bấm `Tạo BOM Tự Động` | — |

**Kết quả mong đợi:** Lỗi "Item cha Den pha chưa được thiết lập BOM Template".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: tích lại Hoạt Động cho `BT-DEN-PHA` trước khi qua test case tiếp theo.

### TC-F-03 — Thiếu BOM Rule cho thành phần Theo Rule `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở `BT-DEN-PHA`, xoá tạm dòng Công Thức BOM của `Nguon`, Save | — |
| 2 | Bấm `Tạo BOM Tự Động` trên `Den pha-200W-Tr-DN-HKLED` | — |

**Kết quả mong đợi:** Lỗi "Không tìm thấy BOM Rule cho Mặt Hàng Den pha-200W-Tr-DN-HKLED, Thành Phần BOM Nguon (BOM Template BT-DEN-PHA)".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: phục hồi lại dòng Rule `Nguon → RM-NGUON-50W × 4` trước khi qua test case tiếp theo.

### TC-F-04 — Lần đầu chạy — tạo đúng cây BOM 3 cấp, đúng thứ tự con → cha `Cao`

Điều kiện: `Den pha-200W-Tr-DN-HKLED` **chưa có** BOM Default nào (item mới, hoặc đã xoá BOM cũ trước đó).

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở Production Plan, dòng item `Den pha-200W-Tr-DN-HKLED`, bấm `Tạo BOM Tự Động` | — |
| 2 | Đọc thông báo trả về | — |
| 3 | Mở danh sách BOM, lọc theo 3 item trong cây | — |

**Kết quả mong đợi:** Thông báo xanh lá "Đã tạo mới 3 BOM…"; field `BOM No` trên dòng Production Plan Item được set tự động. Có đúng 3 BOM mới, trạng thái `Submitted`, `Is Default = 1`:
- BOM của `Chip LED Module Test-Tr-Led Philips`: gồm RM-CHIP-NGUYEN-LIEU ×20, RM-PCB ×1
- BOM của `Module LED Test-Tr-Led Philips`: gồm Chip LED Module Test-Tr-Led Philips ×1 (cột BOM No trỏ đúng BOM Chip vừa tạo), RM-TAN-NHIET ×1, RM-LENS ×1, RM-GIOANG ×1
- BOM của `Den pha-200W-Tr-DN-HKLED`: gồm RM-NGUON-50W ×4, RM-VO-DEN-200W ×1, RM-OC-VIT ×3, RM-DAY-DIEN ×32, Module LED Test-Tr-Led Philips ×1 (cột BOM No trỏ đúng BOM Module vừa tạo)

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-F-05 — Chạy lại lần 2, không đổi cấu hình → hợp lệ, không tạo mới `Cao`

Điều kiện: đã hoàn thành TC-F-04.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ghi lại số lượng BOM hiện có của 3 item trong cây (Count) | — |
| 2 | Bấm lại `Tạo BOM Tự Động` trên cùng dòng, không sửa gì Template/Rule | — |

**Kết quả mong đợi:** Thông báo (màu xanh dương) "BOM hiện tại hợp lệ"; số lượng BOM của cả 3 item **không đổi** — không có bản ghi mới nào được tạo.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-F-06 — Đổi BOM Rule rồi chạy lại → phát hiện không hợp lệ, tạo lại cả cây, đổi Default `Cao`

Điều kiện: đã hoàn thành TC-F-04/05.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở `BT-DEN-PHA`, sửa dòng Rule `Nguon`: đổi Số Lượng | 4 → 5 |
| 2 | Save Template | — |
| 3 | Quay lại Production Plan, bấm lại `Tạo BOM Tự Động` | — |

**Kết quả mong đợi:** Thông báo "Đã tạo mới 3 BOM…" — **cả 3 cấp đều được tạo lại** (không chỉ cấp Đèn pha), vì khi bất kỳ cấp nào không khớp Template/Rule hiện hành, toàn bộ cây được tái tạo và gán Default mới theo đúng mục 7.3 tài liệu nghiệp vụ. BOM cũ của cả 3 item chuyển `Is Default = 0` nhưng vẫn giữ nguyên trạng thái Submitted (không bị huỷ/xoá). BOM mới của Đèn pha có dòng RM-NGUON-50W với SL=5.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: đổi lại SL `Nguon` về 4 nếu muốn dùng lại bộ dữ liệu mẫu ban đầu cho các lần test sau.

### TC-F-07 — Liên kết BOM No giữa cha và bán thành phẩm con `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở BOM Default mới nhất của `Den pha-200W-Tr-DN-HKLED` | — |
| 2 | Xem dòng nguyên liệu `Module LED Test-Tr-Led Philips`, cột `BOM No` | — |

**Kết quả mong đợi:** Cột BOM No trỏ đúng đến BOM Default mới nhất của Module LED Test-Tr-Led Philips vừa được engine tạo (không để trống, không trỏ về BOM cũ đã hết hiệu lực).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-F-08 — Đổi Rule ở cấp lồng bên trong → toàn cây vẫn bị phát hiện không hợp lệ và tái tạo `Cao`

Điều kiện: đã hoàn thành TC-F-04/05 (cây BOM 3 cấp đang hợp lệ). Khác TC-F-06 ở chỗ sửa Rule tại **cấp con** (Chip LED Module Test), không phải cấp gốc (Đèn pha) — xác minh đúng mục 7.3/7.5 "Toàn bộ cây BOM phải được kiểm tra" ở mọi cấp, không chỉ cấp trên cùng.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở `BT-CHIP-LED-MODULE-TEST`, sửa dòng Cố Định `Chip nguyen lieu`: đổi Số Lượng | 20 → 25 |
| 2 | Save Template | — |
| 3 | Quay lại Production Plan, dòng `Den pha-200W-Tr-DN-HKLED`, bấm `Tạo BOM Tự Động` | — |

**Kết quả mong đợi:** Thông báo "Đã tạo mới 3 BOM…" — mặc dù thay đổi chỉ nằm ở cấp thấp nhất (Chip LED Module Test), hệ thống vẫn phải phát hiện toàn bộ cây không hợp lệ và tạo lại **cả 3 cấp** theo đúng thứ tự con trước cha, gán Default mới cho cả 3; BOM Chip mới có dòng RM-CHIP-NGUYEN-LIEU với SL=25; BOM Module và BOM Đèn pha cũng có bản ghi mới (BOM No mới) dù các NVL khác không đổi số lượng.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: đổi lại SL `Chip nguyen lieu` về 20 nếu muốn dùng lại bộ dữ liệu mẫu ban đầu.

### TC-F-09 — User không có quyền tạo BOM bị chặn `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Đăng nhập bằng user không có quyền Create trên doctype BOM (không thuộc role System Manager/Manufacturing Manager có quyền tạo BOM) | — |
| 2 | Ở Production Plan, bấm `Tạo BOM Tự Động` trên 1 dòng item biến thể hợp lệ | — |

**Kết quả mong đợi:** Lỗi từ chối quyền (PermissionError / "Not permitted"); không có BOM nào được tạo.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-F-10 — [Bonus] Vòng lặp trong cây BOM bị chặn `Trung bình`

Ca kiểm thử bổ sung ngoài tài liệu nghiệp vụ — kiểm tra cơ chế an toàn có trong code (`build_bom_tree`) nhưng không được đặc tả rõ trong tài liệu gốc.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Trong dữ liệu test cô lập, cố tình thiết lập BOM Rule để A tham chiếu B, B tham chiếu ngược lại A (tạo vòng lặp) | ví dụ sửa tạm Rule của Chip LED Module Test trỏ ngược về chính Module LED Test-Tr-Led Philips |
| 2 | Bấm `Tạo BOM Tự Động` trên item gốc của cây | — |

**Kết quả mong đợi:** Lỗi rõ ràng "Phát hiện vòng lặp trong cây BOM…"; không tạo BOM nào, không rơi vào đệ quy vô hạn/treo hệ thống.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: phục hồi lại Rule gốc sau khi test xong.

---

## Bảng tổng hợp kết quả

| Nhóm | Số TC | Đạt | Không đạt | Ghi chú |
|---|---|---|---|---|
| A — BOM Component | 2 | | | |
| B — BOM Component Table | 5 | | | |
| C — BOM Rule | 4 | | | |
| D — BOM Template | 3 | | | |
| E — Popup Tạo Rule | 5 | | | |
| F — Engine Tạo BOM Tự Động | 10 | | | |
| **Tổng** | **29** | | | |

---

*Phần I — Tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED · app `mbwnext_hkled` · site `hkled.com`*
