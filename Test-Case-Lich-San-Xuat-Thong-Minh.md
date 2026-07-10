# Test Case — Lịch Sản Xuất Thông Minh (Phần III)

**HKLED · Manufacturing · QA**

Bộ test case tay (manual) cho PHẦN III của tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED: Employee Schedule, Employee Allocation, Work Order Employee, và engine tự động tính lịch sản xuất trên Work Order.

- App: `mbwnext_hkled`
- Site: `hkled.com`
- Phạm vi: Phần III
- Quyền: System Manager / Manufacturing Manager

## ℹ Bối cảnh

3 doctype (`Employee Schedule`, `Employee Allocation`, `Work Order Employee`) và các field trên Work Order đã tồn tại sẵn (dựng trên UI, nay đã đưa vào source code), nhưng **toàn bộ ràng buộc nghiệp vụ và thuật toán tính lịch đều mới viết**. Đây là phần việc trọng tâm cần test kỹ nhất trong 3 phần.

## ⚠ Lưu ý khi test

Dùng đúng bộ dữ liệu mẫu cô lập ở mục "Dữ liệu mẫu" (nhân sự tiền tố `PIII-TEST`, Work Order `MFG-WO-2026-00129`). **Không** bấm "Tính Lại Lịch" trên Work Order thật đang có Work Order Employee — thao tác này sẽ tạo/ghi đè Employee Allocation ngay lập tức.

Một số test case (nhóm D) sẽ **thay đổi trạng thái** của Work Order mẫu `MFG-WO-2026-00129`. Sau khi test xong nhóm D, làm theo bước "Khôi phục" ở cuối nhóm để trả dữ liệu mẫu về đúng ví dụ gốc cho người test sau.

## Mục lục

1. [Dữ liệu mẫu đã chuẩn bị](#dữ-liệu-mẫu-đã-chuẩn-bị-sẵn)
2. [A — Employee Schedule](#a--employee-schedule)
3. [B — Employee Allocation](#b--employee-allocation)
4. [C — Work Order Employee / Work Order](#c--work-order-employee--work-order)
5. [D — Engine Tính Lại Lịch](#d--engine-tính-lại-lịch)
6. [E — Đồng bộ khi Finish](#e--đồng-bộ-khi-work-order-finish)
7. [Bảng tổng hợp kết quả](#bảng-tổng-hợp-kết-quả)

---

## Dữ liệu mẫu đã chuẩn bị sẵn

Tái hiện đúng ví dụ tính toán trong tài liệu nghiệp vụ (mục 7.8): Item A, 100 sản phẩm, 10 phút/cái, 3 nhân sự A/B/C.

| Đối tượng | Tên trong hệ thống | Ghi chú |
|---|---|---|
| Item | `PIII-TEST-ITEM-A` | Thời Gian Sản Xuất = 10 phút |
| Nhân sự A | `HR-EMP-00010` (PIII-TEST Nhan Su A) | Bậc 7 — Nguồn Lực 100% |
| Nhân sự B | `HR-EMP-00011` (PIII-TEST Nhan Su B) | Bậc 7 — Nguồn Lực 100% |
| Nhân sự C | `HR-EMP-00012` (PIII-TEST Nhan Su C) | Bậc 6 — Nguồn Lực 90% |
| Work Order mẫu | `MFG-WO-2026-00129` | Qty 100, Thời Gian Bắt Đầu 8h00 23/06/2026 |

**Lịch làm việc & xung đột đã khai báo sẵn**

| Nhân sự | Employee Schedule đã khai báo | Employee Allocation khác (xung đột) |
|---|---|---|
| A | Sáng + Chiều 23/6 | 1 Work Order khác bắt đầu 8h00 24/6 |
| B | Chiều 23/6, Sáng + Chiều 24/6 | Không có |
| C | Sáng + Chiều 23/6 | 1 Work Order khác đến 10h00 23/6; 1 Work Order khác bắt đầu 8h00 24/6 |

**Kết quả đúng (baseline, sau khi bấm Tính Lại Lịch lần đầu trên dữ liệu sạch)**

| | |
|---|---|
| Nhân sự A | Bắt Đầu 8h00 23/6 · Giới Hạn Tham Gia 17h00 23/6 · Kết Thúc 17h00 23/6 |
| Nhân sự B | Bắt Đầu 13h15 23/6 · Giới Hạn Tham Gia *(trống)* · Kết Thúc 8h28 24/6 |
| Nhân sự C | Bắt Đầu 10h00 23/6 · Giới Hạn Tham Gia 17h00 23/6 · Kết Thúc 17h00 23/6 |
| Work Order | Thời Gian Kết Thúc Dự Kiến: **8h28 24/06/2026** · Tổng Thời Gian Dự Kiến: **478 phút** |

---

## A — Employee Schedule

Lịch làm việc hàng ngày của nhân sự (ca sáng/chiều/tăng ca).

### TC-A-01 — Tạo Lịch làm việc theo Ca hợp lệ `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Vào `Employee Schedule` → New | — |
| 2 | Chọn Tên Nhân Sự, Ngày, Ca | 1 nhân sự test bất kỳ, ngày bất kỳ, Ca=`Sáng` |
| 3 | Quan sát Bắt Đầu/Kết Thúc, Save | — |

**Kết quả mong đợi:** Bắt Đầu/Kết Thúc tự fetch theo giờ của Ca (08:00/11:45); sau khi Save, Thời Gian Bắt Đầu = Ngày+Bắt Đầu, Thời Gian Kết Thúc = Ngày+Kết Thúc, tự tính đúng, 2 field này read-only.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-02 — Tăng Ca ẩn field Ca, cho nhập giờ tự do `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New Employee Schedule, tích Tăng Ca | — |
| 2 | Quan sát field Ca | — |
| 3 | Nhập Bắt Đầu=18:00, Kết Thúc=21:00, Save | — |

**Kết quả mong đợi:** Field Ca biến mất khi tích Tăng Ca; lưu thành công với giờ tự nhập (nằm trong khung 17h01–23h59).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-03 — Tăng Ca — Bắt Đầu trước 17h01 → lỗi `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New Employee Schedule, tích Tăng Ca | — |
| 2 | Nhập Bắt Đầu=17:00, Kết Thúc=20:00, Save | — |

**Kết quả mong đợi:** Lỗi "Khi Tăng Ca, Bắt Đầu chỉ được tính từ 17h01".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-04 — Tăng Ca — Kết Thúc sau 23h59 → lỗi `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New Employee Schedule, tích Tăng Ca | — |
| 2 | Nhập Bắt Đầu=18:00, Kết Thúc=23:59:59 rồi thử vượt (dùng giờ ngày hôm sau nếu UI cho phép), Save | — |

**Kết quả mong đợi:** Lỗi "Khi Tăng Ca, Kết Thúc chỉ được tính đến 23h59" khi Kết Thúc vượt mốc này.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-A-05 — Trùng Nhân Sự + Thời Gian Bắt Đầu/Kết Thúc `Cao`

Điều kiện: nhân sự test đã có 1 Employee Schedule ca Sáng (23/6) từ TC-A-01 hoặc dữ liệu mẫu.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Employee Schedule mới, cùng Nhân Sự, cùng Ngày, cùng Ca (trùng giờ bắt đầu/kết thúc) | — |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi "Nhân sự ... đã có Lịch làm việc khác bắt đầu/kết thúc cùng thời điểm ...".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## B — Employee Allocation

Ghi nhận khoảng thời gian nhân sự tham gia 1 Work Order cụ thể.

### TC-B-01 — Tạo Employee Allocation hợp lệ `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | New Employee Allocation | 1 nhân sự Công nhân test, 1 Work Order bất kỳ, khung giờ không trùng gì |
| 2 | Save | — |

**Kết quả mong đợi:** Lưu thành công.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-B-02 — Trùng thời gian cho cùng 1 nhân sự → chặn `Cao`

Điều kiện: đã có TC-B-01, hoặc dùng nhân sự C (đã có Allocation 6h00–10h00 23/6 trong dữ liệu mẫu).

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Employee Allocation mới cho nhân sự C | Work Order khác, khung giờ 8h00–9h00 23/6/2026 (nằm trong khoảng 6h00–10h00 đã có) |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi "Nhân sự ... đã được phân công vào Lệnh sản xuất ... từ ... đến ..., trùng thời gian".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## C — Work Order Employee / Work Order

Bảng Nhân Công Tham Gia và các field lịch trên Work Order.

### TC-C-01 — Nhân Sự chỉ cho chọn Công nhân `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở Work Order mẫu, thêm dòng mới trong Nhân Công Tham Gia | — |
| 2 | Mở dropdown Nhân Sự, gõ tên 1 nhân sự Loại nhân sự=Bán hàng/Kế toán | — |

**Kết quả mong đợi:** Nhân sự đó không xuất hiện trong danh sách gợi ý (chỉ hiện Công nhân).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-02 — Bậc Thợ / Nguồn Lực tự fetch khi chọn Nhân Sự `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Thêm dòng mới, chọn Nhân Sự | `HR-EMP-00012` (nhân sự C, Bậc 6) |
| 2 | Quan sát Bậc Thợ, Nguồn Lực (%) | — |

**Kết quả mong đợi:** Bậc Thợ tự điền `Bậc 6`, Nguồn Lực tự điền `90`, cả 2 field read-only.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-03 — Thời Gian Bắt Đầu bắt buộc trên Work Order `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Work Order mới, để trống Thời Gian Bắt Đầu, điền các field khác đầy đủ | — |
| 2 | Save | — |

**Kết quả mong đợi:** Lỗi mandatory cho Thời Gian Bắt Đầu, không lưu được.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-C-04 — Trùng Nhân Sự trong cùng bảng Nhân Công Tham Gia `Trung bình`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở Work Order mẫu, thêm 1 dòng nữa với Nhân Sự trùng 1 dòng đã có (vd thêm HR-EMP-00010 lần 2) | — |
| 2 | Bấm "Tính Lại Lịch" | — |

**Kết quả mong đợi:** Lỗi "Nhân sự ... bị trùng trong Bảng Nhân Công Tham Gia".

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: xoá dòng vừa thêm trước khi qua test case tiếp theo.

---

## D — Engine Tính Lại Lịch

Phần lõi Phần III. Test trực tiếp trên Work Order mẫu `MFG-WO-2026-00129`, nút `Tính Lại Lịch`.

### TC-D-01 — Tính lịch khớp chính xác ví dụ tài liệu `Trọng tâm`

Điều kiện: Work Order mẫu đang ở trạng thái gốc (xem bảng "Kết quả đúng" ở mục Data).

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Mở Work Order `MFG-WO-2026-00129` | — |
| 2 | Bấm nút `Tính Lại Lịch` | — |
| 3 | Đọc thông báo, kiểm tra Thời Gian Kết Thúc Dự Kiến / Tổng Thời Gian Dự Kiến | — |
| 4 | Kiểm tra từng dòng trong bảng Nhân Công Tham Gia | — |

**Kết quả mong đợi:** Thời Gian Kết Thúc Dự Kiến = **8h28 24/06/2026**, Tổng Thời Gian Dự Kiến = **478 phút**. Nhân sự A: Bắt Đầu 8h00 23/6, Giới Hạn Tham Gia & Kết Thúc = 17h00 23/6. Nhân sự B: Bắt Đầu 13h15 23/6, Giới Hạn Tham Gia trống, Kết Thúc 8h28 24/6. Nhân sự C: Bắt Đầu 10h00 23/6, Giới Hạn Tham Gia & Kết Thúc = 17h00 23/6. Khớp chính xác bảng tham chiếu ở mục Data.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-02 — Employee Allocation tự tạo và liên kết đúng `Cao`

Điều kiện: đã thực hiện TC-D-01.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Trên mỗi dòng Nhân Công Tham Gia, mở link ở cột Bản Ghi Phân Công | — |

**Kết quả mong đợi:** Mỗi dòng có 1 Employee Allocation riêng, employee_name/work_order/start_time/end_time khớp đúng dữ liệu trên dòng tương ứng.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-03 — Bấm lại lần 2 không đổi dữ liệu → kết quả ổn định `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Bấm lại `Tính Lại Lịch` ngay, không sửa gì | — |

**Kết quả mong đợi:** Kết quả giống hệt TC-D-01 (478 phút, 8h28 24/6); Bản Ghi Phân Công của từng dòng vẫn là các Employee Allocation cũ (không tạo bản ghi mới, chỉ cập nhật).

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-04 — Nhân sự nghỉ giữa chừng (Khóa Kết Thúc) → tính lại đúng `Trọng tâm`

Điều kiện: đã thực hiện TC-D-01. Thao tác này sẽ thay đổi trạng thái Work Order mẫu.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở dòng Nhân sự A, sửa Kết Thúc | `10:00:00 23/06/2026` |
| 2 | Tích Khóa Kết Thúc cho dòng A, Save Work Order | — |
| 3 | Bấm `Tính Lại Lịch` | — |

**Kết quả mong đợi:** Dòng A giữ nguyên Kết Thúc 10h00 23/6 (không bị tính lại vì đã khóa). Dòng B và C được tính lại: do khối lượng công việc còn lại tăng (A đóng góp ít hơn), Thời Gian Kết Thúc Dự Kiến của WO bị đẩy muộn hơn 8h28 24/6 (dài hơn kịch bản gốc); Tổng Thời Gian Dự Kiến > 478 phút.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-05 — Thiếu lịch làm việc/nhân sự → báo lỗi rõ ràng `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Tạo Work Order test mới với 1 nhân sự Công nhân **chưa có bất kỳ Employee Schedule nào** | Item PIII-TEST-ITEM-A, qty nhỏ (vd 1), Thời Gian Bắt Đầu bất kỳ |
| 2 | Bấm `Tính Lại Lịch` | — |

**Kết quả mong đợi:** Lỗi rõ ràng "Không tìm thấy Lịch làm việc (Employee Schedule) cho nhân sự ... kể từ ...", không tạo BOM/Allocation gì, WO không bị lưu sai dữ liệu.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

### TC-D-06 — Khóa Bắt Đầu với thời điểm đang trùng lịch khác → chặn lưu `Cao`

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Ở Work Order mẫu, thêm dòng mới cho Nhân sự C | — |
| 2 | Nhập Bắt Đầu = 7h00 23/6/2026 (nằm trong khoảng 6h00–10h00 nhân sự C đã bận ở Work Order khác), tích Khóa Bắt Đầu | — |
| 3 | Bấm `Tính Lại Lịch` | — |

**Kết quả mong đợi:** Lỗi "Nhân sự ... đã có lịch ở Work Order ... vào thời điểm này", không tính toán tiếp.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

> Dọn dẹp: xoá dòng vừa thêm.

### ↺ Khôi phục dữ liệu mẫu sau nhóm D

> Sau khi test xong TC-D-04 (đã khóa Kết Thúc nhân sự A): mở lại dòng A, **bỏ tích Khóa Kết Thúc** và xoá giá trị Kết Thúc, Save, rồi bấm `Tính Lại Lịch` lại 1 lần. Kết quả sẽ trở về đúng baseline (478 phút, 8h28 24/6) cho người test sau.

---

## E — Đồng bộ khi Work Order Finish

Khi Lệnh sản xuất hoàn thành thực tế, Employee Allocation phải cập nhật theo thời điểm hoàn thành thật.

### TC-E-01 — Finish Work Order → Employee Allocation cập nhật end_time thực tế `Cao`

Điều kiện: dùng 1 Work Order test riêng (không dùng WO mẫu chính), đã có Nhân Công Tham Gia + đã Tính Lại Lịch + đã Submit + đã tạo đủ Stock Entry Manufacture cho đến khi Work Order chuyển trạng thái `Completed`.

| # | Hành động | Dữ liệu nhập |
|---|---|---|
| 1 | Hoàn tất Stock Entry Manufacture cho đủ số lượng để Work Order chuyển `Completed` | — |
| 2 | Mở từng Bản Ghi Phân Công (Employee Allocation) liên kết từ bảng Nhân Công Tham Gia | — |

**Kết quả mong đợi:** Kết Thúc (end_time) của mỗi Employee Allocation được cập nhật thành thời điểm Work Order hoàn thành thực tế (Actual End Date), khác với giá trị dự tính trước đó nếu WO hoàn thành sớm/muộn hơn kế hoạch.

**Kết quả thực tế:** _______________________ **[ ] Chưa test**

---

## Bảng tổng hợp kết quả

| Nhóm | Số TC | Đạt | Không đạt | Ghi chú |
|---|---|---|---|---|
| A — Employee Schedule | 5 | | | |
| B — Employee Allocation | 2 | | | |
| C — Work Order Employee / Work Order | 4 | | | |
| D — Engine Tính Lại Lịch | 6 | | | |
| E — Đồng bộ khi Finish | 1 | | | |
| **Tổng** | **18** | | | |

---

*Phần III — Tài liệu nghiệp vụ nâng cấp giai đoạn 1 HKLED · app `mbwnext_hkled` · site `hkled.com`*
