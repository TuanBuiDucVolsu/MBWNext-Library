# Test Case — Bậc Thợ & Giờ Công Chuẩn (Phần II)

**HKLED · Manufacturing · QA**

Bộ test case tay (manual) cho PHẦN II của tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED: doctype Employee Level (Bậc Thợ), trường Thời Gian Sản Xuất trên Item, và Vai Trò/Bậc Thợ/Nguồn Lực trên Employee.

- App: `mbwnext_hkled`
- Site: `hkled.com`
- Phạm vi: Phần II
- Quyền: System Manager / HR Manager

## ℹ Bối cảnh khác với Phần I

Phần lớn Phần II **đã được xây sẵn trực tiếp trên hệ thống thật** trước khi bắt tay vào việc này (doctype Employee Level với 5 bản ghi thật `Bậc 3`–`Bậc 7`, field `custom_employee_type`, `custom_employee_level`, `custom_performance_factor_` trên Employee). Công việc thực tế là: đưa các field này vào source code (version control), sửa 1 field Item bị lỗi chính tả (giữ nguyên dữ liệu), và thêm cảnh báo mềm — **không** bật ràng buộc bắt buộc vì còn nhiều nhân sự thật chưa có dữ liệu.

Vì vậy bộ test case này có thêm các case "Hồi quy" (regression) để xác nhận dữ liệu thật không bị ảnh hưởng, bên cạnh các case chức năng thông thường.

## ⚠ Lưu ý khi test

Không sửa/xoá 5 bản ghi Employee Level thật (`Bậc 3`–`Bậc 7`) hay dữ liệu của các nhân sự thật (`Nguyễn Nam`, `HR-EMP-00007`, `Đỗ Thắng`, …). Khi cần test tạo/sửa, dùng bản ghi đặt tên rõ ràng riêng (hậu tố `Test`) và xoá lại sau khi test xong.

## Mục lục

1. [Dữ liệu tham chiếu hiện có](#dữ-liệu-tham-chiếu-hiện-có)
2. [A — Employee Level (Bậc Thợ)](#a--employee-level-bậc-thợ)
3. [B — Thời Gian Sản Xuất (Item)](#b--thời-gian-sản-xuất-item)
4. [C — Vai Trò / Bậc Thợ / Nguồn Lực (Employee)](#c--vai-trò--bậc-thợ--nguồn-lực-employee)
5. [Chưa triển khai — sẽ test ở Phần III](#chưa-triển-khai--sẽ-test-ở-phần-iii)
6. [Bảng tổng hợp kết quả](#bảng-tổng-hợp-kết-quả)

---

## Dữ liệu tham chiếu hiện có

Dùng để đối chiếu kết quả mong đợi ở các case hồi quy — không tự ý sửa các bản ghi này.

**Employee Level**

| Bậc Thợ | Lương Mỗi Phút | Nguồn Lực (%) | Tỉ Lệ Đóng Quỹ Đội (%) |
|---|---|---|---|
| `Bậc 3` | 580 | 60 | 25 |
| `Bậc 4` | 660 | 70 | 20 |
| `Bậc 5` | 760 | 80 | 15 |
| `Bậc 6` | 920 | 90 | 10 |
| `Bậc 7` | 1080 | 100 | 5 |

**Nhân sự**

| Nhân sự | Loại nhân sự | Bậc Thợ | Ghi chú |
|---|---|---|---|
| `Nguyễn Nam` | Công nhân | Bậc 6 | Đầy đủ dữ liệu |
| `HR-EMP-00007` | Công nhân | Bậc 6 | Đầy đủ dữ liệu |
| `Đỗ Thắng` | Công nhân | *(trống)* | Thiếu Bậc Thợ — dùng để test cảnh báo mềm |

**Item**

| Item | Thời Gian Sản Xuất (Phút) |
|---|---|
| `Đèn pha` | 10 |
| `Đèn pha- 200W-Tr-DN-HKLED` | 10 |

---

## A — Employee Level (Bậc Thợ)

Doctype độc lập, quản lý thang bậc thợ dùng để tính lương/giờ công (Phần III sẽ dùng đến).

### TC-A-01 — Tạo mới Bậc Thợ hợp lệ `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Vào `Employee Level` → New | — |
| 2 | Nhập đủ 4 trường | Bậc Thợ=`Bậc 8 Test`, Lương Mỗi Phút=1200, Nguồn Lực=100, Tỉ Lệ Đóng Quỹ Đội=5 |
| 3 | Save | — |

**Kết quả mong đợi:** Lưu thành công; tên bản ghi = `Bậc 8 Test` (autoname theo field).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-02 — Trùng tên Bậc Thợ `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Employee Level mới, đặt tên trùng bản ghi thật đã có | Bậc Thợ=`Bậc 7` |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi trùng (Duplicate entry), không tạo thêm bản ghi, bản ghi `Bậc 7` thật không bị ảnh hưởng.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-03 — Thiếu trường bắt buộc `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New Employee Level, chỉ nhập Bậc Thợ | `Bậc 9 Test` |
| 2 | Để trống Lương Mỗi Phút / Nguồn Lực / Tỉ Lệ Đóng Quỹ Đội, Save | — |

**Kết quả mong đợi:** Lỗi mandatory cho 3 trường còn lại, không lưu được.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-04 — Hồi quy — 5 Bậc Thợ thật còn nguyên vẹn `Hồi quy`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở danh sách `Employee Level` | — |
| 2 | Đối chiếu từng bản ghi `Bậc 3`–`Bậc 7` với bảng dữ liệu tham chiếu ở mục Data | — |

**Kết quả mong đợi:** Đủ 5 bản ghi, giá trị Lương Mỗi Phút / Nguồn Lực / Tỉ Lệ Đóng Quỹ Đội khớp chính xác với bảng tham chiếu, không có bản ghi nào bị mất hoặc đổi giá trị.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## B — Thời Gian Sản Xuất (Item)

Field `custom_time_to_manufacture` ở tab Manufacturing của Item, đơn vị phút.

### TC-B-01 — Thiết lập Thời Gian Sản Xuất cho 1 Item `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở 1 Item bất kỳ (không phải 2 item tham chiếu ở mục Data) | — |
| 2 | Vào tab `Manufacturing`, cuộn xuống cuối tab | — |
| 3 | Nhập Thời Gian Sản Xuất (Phút), Save | 15 |

**Kết quả mong đợi:** Field nằm ở cuối tab Manufacturing (sau field "Total Projected Qty"); lưu thành công với giá trị 15.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-02 — Hồi quy — dữ liệu cũ giữ nguyên sau khi đổi tên field `Hồi quy`

Bối cảnh: field gốc tên `custom_time_to_manufature` (lỗi chính tả) đã được đổi tên thành `custom_time_to_manufacture`, dữ liệu phải được giữ nguyên trong quá trình đổi tên.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở Item `Đèn pha`, tab Manufacturing | — |
| 2 | Mở Item `Đèn pha- 200W-Tr-DN-HKLED`, tab Manufacturing | — |

**Kết quả mong đợi:** Cả 2 item đều hiển thị Thời Gian Sản Xuất (Phút) = 10, không bị mất/về 0.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## C — Vai Trò / Bậc Thợ / Nguồn Lực (Employee)

3 field trên Employee: `custom_employee_type` (Loại nhân sự), `custom_employee_level` (Bậc Thợ), `custom_performance_factor_` (Nguồn Lực %, tự fetch, read-only).

### TC-C-01 — Gán Công nhân + Bậc Thợ → Nguồn Lực tự fetch `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở 1 Employee test (không phải 3 nhân sự tham chiếu) | — |
| 2 | Chọn Loại nhân sự | `Công nhân` |
| 3 | Chọn Bậc Thợ | `Bậc 6` |
| 4 | Quan sát field Nguồn Lực (%) | — |

**Kết quả mong đợi:** Nguồn Lực (%) tự động điền = `90` (đúng giá trị của Bậc 6), field hiển thị dạng read-only (không sửa tay được).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-02 — Đổi Bậc Thợ → Nguồn Lực cập nhật lại `Cao`

Điều kiện: tiếp tục từ TC-C-01.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Đổi Bậc Thợ | `Bậc 6` → `Bậc 7` |
| 2 | Quan sát Nguồn Lực (%) | — |

**Kết quả mong đợi:** Nguồn Lực (%) tự cập nhật thành `100` ngay khi đổi Bậc Thợ, không cần Save trước.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-03 — Công nhân chưa gán Bậc Thợ → cảnh báo mềm, không chặn save `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở 1 Employee test khác | — |
| 2 | Chọn Loại nhân sự, để trống Bậc Thợ | Loại nhân sự=`Công nhân` |
| 3 | Quan sát góc màn hình, sau đó Save | — |

**Kết quả mong đợi:** Xuất hiện thông báo màu cam "Nhân sự Công nhân nên được gán Bậc Thợ để tính giờ công chuẩn." (tự biến mất sau ~5 giây); Save **vẫn thành công bình thường**, không bị chặn.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-04 — Loại nhân sự khác Công nhân → không cảnh báo `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở 1 Employee test, chọn Loại nhân sự | `Bán hàng` hoặc `Kế toán` |
| 2 | Để trống Bậc Thợ, Save | — |

**Kết quả mong đợi:** Không xuất hiện cảnh báo nào; lưu bình thường; field Bậc Thợ không bắt buộc.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-05 — Hồi quy — nhân sự thật không bị ảnh hưởng `Hồi quy`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở lần lượt `Nguyễn Nam`, `HR-EMP-00007` | — |
| 2 | Mở `Đỗ Thắng` | — |

**Kết quả mong đợi:** Nguyễn Nam & HR-EMP-00007: Loại nhân sự=Công nhân, Bậc Thợ=Bậc 6, Nguồn Lực=90 — không đổi. Đỗ Thắng: Loại nhân sự=Công nhân, Bậc Thợ vẫn trống — mở form thấy cảnh báo cam (TC-C-03) nhưng form vẫn mở/lưu bình thường, không có lỗi mandatory chặn lại.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## Chưa triển khai — sẽ test ở Phần III

> **Công thức Giờ Công Chuẩn**
>
> Tài liệu nghiệp vụ mô tả công thức `Thời gian tính công = Số lượng sản phẩm × Thời gian sản xuất 1 sản phẩm`, áp dụng cho Công Nhân. Công thức này **chưa có field/màn hình riêng ở Phần II** — nó sẽ được hiện thực hoá trong bảng Nhân Công Tham Gia (Work Order Employee) và thuật toán tính thời gian hoàn thành Lệnh sản xuất ở **Phần III**. Không cần tìm kiếm tính năng này khi test Phần II.

---

## Bảng tổng hợp kết quả

| Nhóm | Số TC | Đạt | Không đạt | Ghi chú |
|---|---|---|---|---|
| A — Employee Level | 4 | | | |
| B — Thời Gian Sản Xuất | 2 | | | |
| C — Vai Trò/Bậc Thợ/Nguồn Lực | 5 | | | |
| **Tổng** | **11** | | | |

---

*Phần II — Tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED · app `mbwnext_hkled` · site `hkled.com`*
