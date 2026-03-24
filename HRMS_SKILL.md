---
title: Frappe HRMS App Knowledge
description: Kiến thức đầy đủ về Frappe HRMS v15+ — doctypes, payroll, leave, attendance, API, hooks, patterns, edge cases và examples
version: 15.0.0+
dependencies: frappe>=15, erpnext>=15
app: hrms
---

# Kiến thức về Frappe HRMS App

Bạn là chuyên gia về Frappe HRMS — hệ thống quản lý nhân sự và lương built trên Frappe Framework v15+, kết hợp với ERPNext. Dưới đây là toàn bộ kiến thức bạn cần nắm để hỗ trợ phát triển, debug, và mở rộng app này.

---

## Tổng quan hệ thống

- **App name**: `hrms`
- **Framework**: Frappe v15+ / ERPNext v15+
- **Module chính**: `hrms.hr` (nhân sự) và `hrms.payroll` (lương)
- **Tổng doctypes**: 157 (60 HR + 41 Payroll + 56 supporting)
- **Frontend**: Vue 3 + Ionic Framework (mobile PWA)
- **Database**: MySQL/MariaDB

Cấu trúc thư mục chính:
- `hrms/hr/` — toàn bộ tính năng HR: phép, chấm công, tuyển dụng, đánh giá
- `hrms/payroll/` — lương, thuế, phúc lợi
- `hrms/api/` — REST API cho mobile/frontend
- `hrms/overrides/` — override các doctype ERPNext gốc
- `hrms/controllers/` — base controllers dùng chung
- `hrms/mixins/` — reusable mixins
- `hrms/regional/` — tùy chỉnh theo quốc gia (India, UAE)
- `hrms/frontend/src/` — Vue components và views

---

## Module HR — Các Doctype Quan Trọng

### Quản lý Phép (Leave)

**Leave Type** — định nghĩa các loại phép (Casual, Sick, Privilege, LWP...). Field quan trọng: `is_lwp` (Leave Without Pay), `max_continuous_days_allowed`, `allow_encashment`.

**Leave Policy** — quy định số phép cấp theo designation/tenure. Gán cho nhân viên qua **Leave Policy Assignment**.

**Leave Application** — đơn xin nghỉ. Workflow: `Open → Approved/Rejected`. Khi submit tự động tạo Leave Ledger Entry và cập nhật Attendance. Khi cancel thì đảo ngược.

Methods chính của `LeaveApplication`:
- `validate()` — kiểm tra active employee, ngày hợp lệ, số dư đủ, không trùng đơn, không bị block, lương chưa xử lý
- `on_submit()` — tạo ledger entry, cập nhật attendance, gửi thông báo PWA
- `on_cancel()` — đảo ngược ledger, xóa attendance record
- `validate_balance_leaves()` — kiểm tra số dư phép còn đủ
- `validate_leave_overlap()` — tránh trùng với đơn khác trong cùng khoảng thời gian
- `validate_max_days()` — kiểm tra số ngày liên tục tối đa

**Leave Ledger Entry** — sổ cái theo dõi số dư phép, không edit trực tiếp mà tạo qua Leave Application.

**Leave Encashment** — quy đổi phép còn lại thành tiền, liên kết với Salary Slip.

**Leave Block List** — chặn các ngày không cho phép xin nghỉ (vd: ngày kết sổ).

Hàm tiện ích về Leave:
```python
from hrms.hr.doctype.leave_application.leave_application import get_leave_balance_on
balance = get_leave_balance_on(employee, date, leave_type)
```

---

### Chấm Công & Ca Làm (Attendance & Shifts)

**Attendance** — bản ghi chấm công hàng ngày. Status: `Present`, `Absent`, `Half Day`, `Work From Home`. Tự động tạo từ Employee Checkin qua scheduler.

**Employee Checkin** — log check-in/check-out theo thời gian thực. Nguồn: thiết bị chấm công hoặc mobile app.

**Shift Type** — định nghĩa ca làm: giờ bắt đầu/kết thúc, buffer time, late entry/early exit settings. Có scheduler job `process_auto_attendance()` chạy mỗi giờ để tạo Attendance từ checkin logs.

**Shift Assignment** — gán ca cho nhân viên, có thể theo ngày hoặc theo lịch dài hạn.

**Shift Request** — nhân viên yêu cầu đổi ca, cần approval.

**Attendance Request** — yêu cầu chấm công bổ sung/thủ công cho ngày đã qua.

**Auto-attendance flow**: `Employee Checkin records` → (hourly scheduler) → `Shift Type.process_auto_attendance()` → tạo `Attendance` với status tính từ giờ check-in.

---

### Tuyển Dụng & Onboarding

**Job Opening** → **Job Applicant** → **Interview** → **Job Offer** → **Employee Onboarding** → **Employee**

Khi tạo Employee từ onboarding: `EmployeeOnboarding.make_employee()` — copy thông tin từ Job Applicant, link `job_applicant` field trên Employee.

**Employee Onboarding** dùng Project + Tasks để theo dõi tiến độ. Base class là `EmployeeBoardingController` tại `hrms/controllers/employee_boarding_controller.py`.

**Employee Separation** — quy trình offboarding tương tự, cũng dùng `EmployeeBoardingController`.

---

### Đánh Giá Hiệu Suất (Appraisal)

**Appraisal Cycle** → **Appraisal** (per employee) với `Appraisal Template` (tiêu chí KRA + Goals).

**Appraisal KRA** — Key Result Areas, có trọng số.

**Employee Performance Feedback** — 360-degree feedback từ đồng nghiệp.

Mixin: `AppraisalMixin` tại `hrms/mixins/`.

---

### Chi Phí & Tạm Ứng

**Expense Claim** — workflow: `Draft → Submitted (pending approval) → Approved/Rejected → Paid`. GL entries tạo khi submit. Thanh toán qua Payment Entry.

Methods:
- `validate_sanctioned_amount()` — kiểm tra giới hạn duyệt theo chức danh
- `calculate_taxes()` — áp thuế theo Expense Claim Type
- `get_gl_entries()` — tạo journal entries khi submit

**Employee Advance** — tạm ứng tiền mặt, theo dõi số dư, trừ vào lương hoặc hoàn trả.

---

### Employee Master (Override)

`Employee` được override bởi `hrms.overrides.employee_master.EmployeeMaster`. Thêm các field:
- `employment_type`, `grade`, `default_shift`
- `health_insurance_provider`
- Approvers: `leave_approver`, `expense_approver`, `shift_request_approver`
- `payroll_cost_center`
- `job_applicant` (link từ Recruitment flow)

Naming options: Series, Employee Number, hoặc Full Name.

---

## Module Payroll — Các Doctype Quan Trọng

### Flow tạo lương

```
Salary Component (công thức)
    ↓
Salary Structure (tập hợp components)
    ↓
Salary Structure Assignment (gán cho employee)
    ↓
Payroll Entry (tạo hàng loạt, theo period)
    ↓
Salary Slip (per employee, per month)
    ↓
Submit → GL Entries → Journal Entry
```

**Salary Component** — một thành phần lương (Basic, HRA, PF, Income Tax...). Type: `Earning` hoặc `Deduction`. Công thức Python expression, có thể dùng: `base`, `gross`, `total_earnings`, `payment_days`, tên component khác.

**Salary Structure** — tập hợp components. Định nghĩa mode of payment, payroll frequency.

**Salary Slip** — bảng lương của 1 nhân viên 1 kỳ. Tự tính: leave deductions, loan repayments, tax. Methods quan trọng:
- `get_emp_and_working_day_details()` — lấy thông tin nhân viên và ngày công
- `calculate_components()` — đánh giá công thức, tính từng component
- `calculate_net_pay()` — Gross - Deductions = Net Pay
- `deduct_loan_repayment()` — trừ kỳ trả nợ vay
- `create_gl_entries()` — hạch toán kế toán
- `on_submit()` — tạo journal entries, link Payroll Entry

**Payroll Entry** — tạo hàng loạt Salary Slip cho 1 kỳ lương. Lọc theo: Branch, Department, Designation, Employee Type.

**Payroll Period** — định nghĩa chu kỳ lương (start/end date).

---

### Thuế & Phúc Lợi

**Income Tax Slab** — bảng thuế lũy tiến. Liên kết với Payroll Period và Employee.

**Employee Tax Exemption Declaration** — nhân viên khai báo miễn giảm thuế (80C, 80D theo India). Cần submit trước khi tính lương.

**Employee Benefit Application** — đăng ký hưởng phúc lợi (flexible benefit plan).

**Additional Salary** — thêm khoản thưởng/phạt ngoài salary structure (bonus, deduction đặc biệt). Tự động include vào Salary Slip của kỳ tương ứng.

---

### Gratuity (Trợ cấp thôi việc)

**Gratuity Rule** → **Gratuity Rule Slab** (theo số năm kinh nghiệm) → **Gratuity** (per employee khi nghỉ).

Methods:
- `calculate_work_experience_and_amount()` — tính theo slab dựa trên tenure
- `create_gl_entries()` — hạch toán gratuity payable
- `create_additional_salary()` — link vào Salary Slip kỳ cuối

**Full and Final Statement** — quyết toán tổng khi nhân viên nghỉ: lương còn lại, gratuity, encashment, khoản trừ.

---

## API Endpoints

File: `hrms/api/__init__.py` — tất cả dùng `@frappe.whitelist()`.

```python
# Thông tin nhân viên
get_current_user_info()
get_current_employee_info()
get_all_employees(fields=None)

# Cấu hình
get_hr_settings()                          # mobile checkin, geolocation, ...
get_payroll_settings()

# Thông báo
get_unread_notifications_count()
mark_all_notifications_as_read()
are_push_notifications_enabled()

# Chấm công
get_attendance_calendar_events(month, year)

# Ca làm
get_shift_requests(employee)
get_shift_request_approvers(employee, department)

# Phép
get_leave_applications(employee)

# Chi phí
get_expense_claims(employee)
get_employee_advance_balance(employee)
```

File: `hrms/api/roster.py` — API riêng cho mobile roster, bulk attendance sync.

---

## Hooks (`hooks.py`)

### Document Events quan trọng

```python
doc_events = {
    "Employee": ["validate", "on_update", "after_insert", "on_trash"],
    "User": ["validate"],           # quản lý Employee Self Service role
    "Company": ["validate", "on_update", "on_trash"],
    "Holiday List": ["on_update", "on_trash"],   # cache invalidation
    "Timesheet": ["validate"],      # kiểm tra active employee
    "Payment Entry": ["on_submit", "on_cancel", "on_update_after_submit"],  # expense claim payment
    "Journal Entry": ["validate", "on_submit", "on_cancel"],
    "Loan": ["validate"],           # kiểm tra salary repayment link
    "Project": ["on_update"],       # boarding controller
    "Task": ["on_update"],          # boarding controller
}
```

### Scheduled Jobs

| Tần suất | Job |
|----------|-----|
| Hourly | `daily_work_summary_group.trigger_emails()` |
| Hourly_long | `shift_type.process_auto_attendance()`, `shift_schedule_assignment.process_auto_shift()` |
| Daily | Birthday/anniversary reminders, interview feedback reminder, đóng Job Opening hết hạn |
| Daily_long | Xử lý phép hết hạn, tạo leave encashment, cấp phát earned leaves |
| Weekly | Reminder trước 7 ngày |
| Monthly | Reminder trước 30 ngày |

### Override Doctype Classes

```python
override_doctype_class = {
    "Employee":      "hrms.overrides.employee_master.EmployeeMaster",
    "Timesheet":     "hrms.overrides.employee_timesheet.EmployeeTimesheet",
    "Payment Entry": "hrms.overrides.employee_payment_entry.EmployeePaymentEntry",
    "Project":       "hrms.overrides.employee_project.EmployeeProject",
}
```

---

## Controllers & Mixins

### PWANotificationsMixin (`hrms/mixins/pwa_notifications.py`)
Dùng cho các doctype cần gửi thông báo khi approve/reject:
```python
class MyDoc(PWANotificationsMixin, Document):
    def on_update(self):
        if self.status in ("Approved", "Rejected"):
            self.notify_approval_status(self.doctype, self.name, self.status)
```
Đang dùng trong: `LeaveApplication`, `ExpenseClaim`, `ShiftRequest`.

### EmployeeBoardingController (`hrms/controllers/employee_boarding_controller.py`)
Base class cho `Employee Onboarding` và `Employee Separation`. Tạo Project + Tasks từ template, theo dõi tiến độ.

### EmployeeRemindersController (`hrms/controllers/employee_reminders.py`)
Gửi reminder email sinh nhật, anniversary, phép sắp hết hạn, appraisal sắp đến.

---

## HR Utils (`hrms/hr/utils.py`)

Các hàm hay dùng nhất:

```python
from hrms.hr.utils import (
    validate_active_employee,
    validate_dates,
    validate_overlap,
    get_holiday_dates_for_employee,
    get_holidays_for_employee,
    get_leave_period,
    get_holiday_list_for_employee,
    share_doc_with_approver,
    get_salary_assignments,
    allocate_earned_leaves,        # gọi bởi scheduler
    generate_leave_encashment,     # gọi bởi scheduler
)

validate_active_employee(employee)                    # raise nếu nhân viên đã nghỉ
validate_dates(doc, start_date, end_date)             # kiểm tra ngày hợp lệ
validate_overlap(doc, start_date, end_date)           # tránh trùng với record khác
get_holiday_list_for_employee(employee)               # lấy holiday list áp dụng
get_holiday_dates_for_employee(emp, start, end)       # danh sách ngày lễ trong khoảng
share_doc_with_approver(doc, approver_user)           # chia sẻ quyền xem cho approver
```

---

## Frontend (Vue 3 + Ionic)

**Thư mục**: `hrms/frontend/src/`

**Views**:
- `Home.vue` — dashboard tổng hợp
- `attendance/` — portal chấm công, lịch tháng
- `leave/` — xin nghỉ, xem số dư phép
- `expense_claim/` — tạo và theo dõi hoàn tiền
- `employee_advance/` — tạm ứng tiền
- `salary_slip/` — xem bảng lương
- `Profile.vue` — hồ sơ cá nhân
- `Notifications.vue` — thông báo approval

**Components chính**:
- `AttendanceCalendar.vue` — lịch chấm công với màu trạng thái
- `CheckInPanel.vue` — giao diện check-in có geolocation
- `LeaveBalance.vue` — hiển thị số dư phép
- `FormView.vue` — dynamic form rendering (18KB, xử lý mọi loại field)
- `FileUploaderView.vue` — upload ảnh/file đính kèm

**Refresh realtime**:
```javascript
hrms.refetch_resource('Leave Balance')   // trigger re-fetch resource
```

**Dev setup**:
```bash
cd hrms/frontend && yarn install && yarn dev
```

---

## Payroll Utils (`hrms/payroll/utils.py`)

```python
from hrms.payroll.utils import sanitize_expression, get_payroll_settings_for_payment_days

sanitize_expression(expr)                  # làm sạch công thức lương trước khi eval
get_payroll_settings_for_payment_days()    # lấy config từ cache (frappe.cache)
```

---

## Regional — India

Tại `hrms/regional/india/utils.py`:
```python
calculate_annual_eligible_hra_exemption(employee, payroll_period)
calculate_hra_exemption_for_period(...)
calculate_tax_with_marginal_relief(...)    # giảm thuế biên theo luật India
```

---

## Cấu hình thêm trên Company & Department

**Company** (custom fields từ HRMS):
- `default_expense_claim_payable_account`
- `default_employee_advance_account`
- `default_payroll_payable_account`

**Department** (custom fields):
- `payroll_cost_center`
- `leave_block_list`
- `department_approvers` (child table: Shift Approver, Leave Approver, Expense Approver)

---

## User Permissions — Employee Self Service

Nhân viên thường dùng User Type "Employee Self Service", chỉ có quyền:
- **Xem**: Salary Slip của bản thân
- **Tạo**: Leave Application, Expense Claim, Attendance Request, Shift Request, Timesheet, Training Feedback, Employee Grievance
- **Không xem** được dữ liệu của nhân viên khác

---

## Reports

**HR**: Employee Leave Balance Summary, Shift Attendance, Recruitment Analytics, Department Analytics, Employee Birthday, Employees Working on a Holiday.

**Payroll**: Bank Remittance (file chuyển tiền ECS/NEFT), Income Tax Deductions, Income Tax Computation, Salary Payments by Payment Mode.

---

## Patterns Chuẩn Khi Phát Triển

**1. Kiểm tra nhân viên còn làm việc**
```python
from hrms.hr.utils import validate_active_employee
validate_active_employee(self.employee)
```

**2. Chia sẻ doc với approver**
```python
from hrms.hr.utils import share_doc_with_approver
share_doc_with_approver(self, self.leave_approver)
```

**3. Dùng Query Builder thay raw SQL**
```python
from frappe.query_builder import DocType
LA = DocType("Leave Application")
results = frappe.qb.from_(LA).select(LA.name, LA.status).where(
    (LA.employee == employee) & (LA.status == "Approved")
).run(as_dict=True)
```

**4. Gửi thông báo PWA khi approve/reject**
```python
from hrms.mixins.pwa_notifications import PWANotificationsMixin

class MyDoctype(PWANotificationsMixin, Document):
    def on_update(self):
        if self.status in ("Approved", "Rejected"):
            self.notify_approval_status(self.doctype, self.name, self.status)
```

**5. Background job cho tác vụ nặng**
```python
frappe.enqueue(
    "hrms.hr.utils.allocate_earned_leaves",
    queue="long",
    timeout=3600
)
```

**6. Cache payroll settings**
```python
from hrms.payroll.utils import get_payroll_settings_for_payment_days
settings = get_payroll_settings_for_payment_days()  # dùng frappe.cache, không query lại
```

---

## Quick Reference — Tạo Salary Slip

```python
import frappe

# Tạo thủ công 1 salary slip
ss = frappe.new_doc("Salary Slip")
ss.employee = "HR-EMP-00001"
ss.start_date = "2026-03-01"
ss.end_date = "2026-03-31"
ss.get_emp_and_working_day_details()
ss.calculate_net_pay()
ss.save()

# Kiểm tra số dư phép
from hrms.hr.doctype.leave_application.leave_application import get_leave_balance_on
balance = get_leave_balance_on(
    employee="HR-EMP-00001",
    date=frappe.utils.today(),
    leave_type="Casual Leave"
)
```

---

## ✅ Verification Checklist

Dùng để kiểm tra sau khi cài đặt hoặc sau migrate:

- [ ] **Leave Application submit** → tạo `Leave Ledger Entry` với transaction_type = "Leave Application", giảm số dư phép
- [ ] **Attendance tự tạo từ Employee Checkin** → sau khi có checkin log, chạy `Shift Type.process_auto_attendance()` hoặc đợi hourly scheduler, kiểm tra record `Attendance` được tạo với đúng status
- [ ] **Salary Slip tính đúng** → `total_earnings - total_deduction = net_pay`, kiểm tra field `gross_pay`, `total_deduction`, `net_pay` trên slip sau khi `calculate_net_pay()`
- [ ] **Expense Claim tạo GL Entries khi submit** → sau submit, vào GL Entry kiểm tra có 2 entries: debit `Expense Account`, credit `Expense Claim Payable Account`

---

## ⚠️ Edge Cases

### Leave Application
- **Trùng ngày**: Nếu submit 2 Leave Application cùng employee, cùng khoảng ngày → `validate_leave_overlap()` raise `OverlapError`. Phải cancel cái cũ trước.
- **Ngày là holiday**: Leave Application trên ngày holiday vẫn cho phép tạo nhưng không tính vào số ngày phép trừ (hệ thống tự bỏ qua holiday).
- **Leave type có `is_lwp=True`**: Không cần kiểm tra số dư, nhưng tự động trừ lương khi tính Salary Slip.
- **Salary đã xử lý**: Nếu Salary Slip của kỳ đó đã submit → không cho phép submit Leave Application trùng kỳ đó.

### Attendance & Checkin
- **Checkin không thuộc ca nào**: Nếu Employee không có Shift Assignment và Shift Type không set `determine_check_in_and_check_out` → checkin bị bỏ qua, không tạo Attendance.
- **Check-in thiếu check-out**: Nếu chỉ có check-in mà không có check-out trước khi `process_auto_attendance()` chạy → Attendance có thể tạo với status `Absent` hoặc bị bỏ qua tùy cấu hình `working_hours_threshold_for_half_day`.
- **Duplicate Attendance**: Nếu đã có Attendance thủ công cho ngày đó → `validate_duplicate_record()` raise lỗi khi auto-attendance cố tạo thêm.

### Salary Slip
- **Đã submit không sửa được**: Salary Slip sau khi submit là locked. Phải cancel → amend → submit lại. Amend tạo bản mới với amended_from link.
- **Component formula lỗi**: Nếu công thức Salary Component có lỗi Python syntax → `calculate_components()` raise exception, toàn bộ slip không tính được. Kiểm tra bằng cách test formula trực tiếp trong Salary Component.
- **Không có Salary Structure Assignment**: Nếu employee chưa có assignment hợp lệ cho kỳ lương đó → Payroll Entry bỏ qua employee này, không báo lỗi rõ ràng.
- **Additional Salary trùng kỳ**: Nếu có nhiều Additional Salary cho cùng employee, cùng component, cùng tháng → tất cả đều được cộng vào, không deduplicate.

### Expense Claim
- **Approval status khác submitted status**: `approval_status` (Approved/Rejected) độc lập với `docstatus` (0/1/2). Phải set `approval_status = "Approved"` trước khi submit mới tạo được GL entries.
- **Tài khoản chưa cấu hình**: Nếu Company chưa set `default_expense_claim_payable_account` → GL entry creation lỗi khi submit.

---

## 📝 Examples

### Tạo Leave Application

```python
import frappe

# Tạo và submit leave application
doc = frappe.get_doc({
    "doctype": "Leave Application",
    "employee": "HR-EMP-001",
    "leave_type": "Casual Leave",
    "from_date": "2026-03-25",
    "to_date": "2026-03-26",
    "half_day": 0,
    "description": "Personal work",
    "status": "Open",
})
doc.insert()
doc.submit()
# → tự động tạo Leave Ledger Entry, cập nhật Attendance
```

### Tạo Expense Claim

```python
doc = frappe.get_doc({
    "doctype": "Expense Claim",
    "employee": "HR-EMP-001",
    "company": "My Company",
    "expense_approver": "approver@example.com",
    "expenses": [
        {
            "expense_date": "2026-03-20",
            "expense_claim_type": "Travel",
            "description": "Taxi to client",
            "amount": 150000,
            "sanctioned_amount": 150000,
        }
    ],
    "approval_status": "Approved",
})
doc.insert()
doc.submit()
# → tạo GL Entries: debit Travel Expense, credit Expense Payable
```

### Tạo Salary Slip thủ công

```python
ss = frappe.new_doc("Salary Slip")
ss.employee = "HR-EMP-001"
ss.start_date = "2026-03-01"
ss.end_date = "2026-03-31"
ss.get_emp_and_working_day_details()   # load salary structure, working days
ss.calculate_net_pay()                 # tính tất cả components
ss.save()
# Kiểm tra: ss.gross_pay, ss.total_deduction, ss.net_pay
```

### Kiểm tra số dư phép

```python
from hrms.hr.doctype.leave_application.leave_application import get_leave_balance_on

balance = get_leave_balance_on(
    employee="HR-EMP-001",
    date="2026-03-25",
    leave_type="Casual Leave",
)
print(f"Còn lại: {balance} ngày")
```

### Tạo Attendance Request thủ công

```python
doc = frappe.get_doc({
    "doctype": "Attendance Request",
    "employee": "HR-EMP-001",
    "from_date": "2026-03-20",
    "to_date": "2026-03-20",
    "reason": "Forgot to check in",
    "half_day": 0,
})
doc.insert()
doc.submit()
```

### Query Leave Ledger Entry

```python
from frappe.query_builder import DocType

LLE = DocType("Leave Ledger Entry")
entries = (
    frappe.qb.from_(LLE)
    .select(LLE.transaction_date, LLE.leaves, LLE.transaction_type)
    .where(
        (LLE.employee == "HR-EMP-001")
        & (LLE.leave_type == "Casual Leave")
        & (LLE.is_expired == 0)
    )
    .orderby(LLE.transaction_date)
    .run(as_dict=True)
)
```
