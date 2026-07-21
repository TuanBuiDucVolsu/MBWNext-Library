# Quy trình phát triển tính năng MBWNext

Quy trình chuẩn cho các dự án MBWNext sắp tới. **Mục tiêu: làm nhanh hơn.**

---

## 1. Thời gian đang mất vào đâu

MBWNext không phải một sản phẩm, mà là **một nền tảng nhiều tầng phục vụ nhiều khách
cùng lúc**, và số app lẫn số khách đều còn tăng.

Với quy mô đó, thời gian không mất vào việc gõ code. Nó mất vào bốn chỗ sau:

**Làm lại giao diện.** Code xong mới demo, khách bảo không phải cái này. Toàn bộ phần
UI làm lại từ đầu, cộng thêm một buổi giải thích vì sao chậm.

**Chép logic sang khách khác.** Một tính năng viết cho khách A, khách B cũng cần, chép
sang. Khách C cần nữa, chép tiếp. Đến lúc phải sửa thì phải sửa ở ba chỗ — và luôn có
một chỗ bị quên.

**Lỗi cũ quay lại.** Sửa xong không có chỗ nào ghi lại, vài tháng sau lỗi đó xuất hiện
lại ở khách khác. Lần thứ hai vẫn phải dò từ đầu vì không ai nhớ lần trước sửa gì.

**Dò lại quy ước từng app.** Mỗi app một kiểu: app này khai Custom Field bằng
`export-fixtures`, app kia bằng `custom_fields.json`. Mỗi lần đụng app lâu không làm là
lại mất thời gian mở code cũ ra xem. Càng nhiều app thì chỗ này càng tốn.

Bốn chỗ này không hiện ra trong ước lượng công. Chúng nằm rải trong "sao tính năng này
lâu thế" — và cộng lại thường lớn hơn thời gian viết code.

---

## 2. Tốc độ đến từ đâu

Quy trình này nhắm thẳng vào bốn chỗ trên. Chia hai loại vì chúng đến vào thời điểm khác nhau:

### Nhanh ngay từ tính năng đầu tiên

| Nguồn | Vì sao nhanh |
|---|---|
| **Skill nạp sẵn quy ước** | Claude viết đúng chuẩn MBWNext ngay lần đầu — bỏ hẳn vòng qua lại "đặt file ở đâu", "app này khai fixtures kiểu nào" |
| **Không phải nhớ tất cả app** | quy ước riêng của từng app nằm trong skill, không nằm trong đầu người |
| **Phân loại lỗi trước khi sửa** | phần lớn "lỗi" khách báo là yêu cầu mới hoặc khách hiểu sai — cả hai đều không cần đụng code |
| **Tài liệu là sản phẩm dẫn xuất** | test case viết xong thì tài liệu gần như có sẵn, không phải làm lại từ đầu |
| **Lấy hiện vật thay vì hỏi vòng vo** | một file Excel của khách thay được nhiều lượt hỏi đáp qua lại |
| **Claude tự chạy thử trước** (cần bench/site dev đã dựng sẵn) | phần lớn lỗi rõ ràng bị bắt ngay, người test tay chỉ còn việc chốt chứ không phải dò từ đầu |

### Nhanh từ tính năng thứ hai trở đi

| Nguồn | Vì sao nhanh |
|---|---|
| **Cổng mockup** | không làm lại UI sau khi đã code |
| **Chốt tầng đúng ngay** | không phải chép logic sang khách thứ 2, thứ 3 |
| **File test case tích luỹ** | lỗi cũ không quay lại lần thứ ba |

### Nói trước: một hai tính năng đầu sẽ chậm hơn

Team phải làm quen chỗ đặt file, cổng mockup, và viết test case lần đầu. Đây là chi phí
học, không phải chi phí thường xuyên.

Nói trước để khi tuần đầu thấy chậm thì biết đó là đúng dự kiến, không phải quy trình sai.

---

## 3. Bốn giai đoạn, mỗi giai đoạn có sản phẩm cụ thể

```
Câu nói của khách
   │
 ① PHÂN TÍCH  ────────────►  docs/features/<tên>.md      (đầu bài + spec)
   │
 ② DEV        ────────────►  code + docs/mockups/<tên>.html + vi.po + CLAUDE.md
   │
 ③ TEST CASE  ────────────►  docs/testcases/<tên>.md
   │
 ④ TÀI LIỆU   ────────────►  apps/<app>/docs/huong-dan/
```

Nguyên tắc xuyên suốt: **mỗi giai đoạn phải để lại một file**. Không có file nghĩa là
giai đoạn đó chưa xong.

Bốn file dùng **chung một tên gốc**, đặt một lần ở giai đoạn ①. Nhìn tên file là biết
tính năng đang ở bước nào, thiếu file nào là biết đang kẹt ở đâu.

**Chốt vị trí, để cả team đặt file vào đúng một chỗ:**

| Thư mục | Đặt ở | Vì sao |
|---|---|---|
| `docs/features/` | **gốc bench** (`<bench>/docs/features/`) | tính năng có thể chạm nhiều app, không thuộc riêng app nào |
| `docs/mockups/` | **gốc bench** | cùng lý do — mockup có thể vẽ chung nhiều màn hình liên app |
| `docs/testcases/` | **gốc bench** | test case thường kiểm cả điểm giao giữa các app (`TC-REGR`, `TC-ISO`) |
| `docs/huong-dan/` | **trong từng app** (`apps/<app>/docs/huong-dan/`) | tài liệu đi theo app cụ thể — xem Luật 1 ở Giai đoạn ④ |

Ba thư mục đầu nằm ngoài git của mọi app (bench root không phải app), nên **không tự
động đi theo khi deploy sang bench khác** — đây là kho làm việc nội bộ, không phải
phần release.

---

## 4. Nền tảng cần hiểu trước: kiến trúc 4 tầng

Mọi quyết định trong quy trình đều xoay quanh câu hỏi *"code này thuộc tầng nào?"*

| Tầng | Là gì | Ai dùng | Ví dụ |
|---|---|---|---|
| **1** | Frappe / ERPNext | tất cả | `frappe`, `erpnext`, `hrms` |
| **2** | Lõi MBWNext | mọi khách MBWNext | `mbwnext_localization`, `pos_next`, `mbwnext_advanced_*` |
| **3** | Module dùng lại, cài tuỳ khách | vài khách | `mbwnext_einvoice`, `mbwnext_superapp`, `mbwnext_integration_ecommerce`, `mbwnext_advanced_distribution_map`… |
| **4** | App riêng của khách | một khách | `mbwnext_gosumo`, `mbwnext_hkled`, `mbwnext_vnpost`… |

Cột ví dụ chỉ là **vài app tiêu biểu**, không phải danh sách đầy đủ — tầng 3 và tầng 4
còn nhiều app khác và sẽ tiếp tục thêm.

Quy tắc phụ thuộc: **tầng trên được biết về tầng dưới, tầng dưới không được biết về
tầng trên.** App khách biết mọi app lõi; không app lõi nào biết app khách nào tồn tại.

Hai điều dễ nhầm:

- **Một khách có thể có nhiều app** (VNPost dùng cả `mbwnext_vnpost` và `vnpost_sso`).
- **Thứ tự chạy hook theo `sites/apps.txt`**, không theo số tầng. Muốn biết thứ tự thật
  thì đọc file đó, đừng suy từ tên app.

---

## 5. Giai đoạn ① — Phân tích

Đầu vào: câu nói thô của khách. Đầu ra: `docs/features/<tên>.md`.

### Bước 0: Thu thập dữ liệu đầu vào

Nguyên tắc gốc: **khách không mô tả được yêu cầu, nhưng phản biện rất tốt.** Hỏi
"anh cần gì" thì nhận lại câu mơ hồ, hoặc một giải pháp khách đã tự nghĩ sẵn ("thêm
cho tôi cái nút này"). Đưa cho họ một thứ cụ thể để chê thì moi ra được nhiều gấp
mấy lần.

Nên thu thập theo thứ tự: **hiện vật trước, lời nói sau.**

**1. Xin tài liệu** — năng suất cao nhất, tốn ít công nhất:

| Xin cái gì | Nó trả lời sẵn điều gì |
|---|---|
| File Excel đang dùng | tên trường thật, công thức, cột "ghi chú" chứa ngoại lệ |
| Chứng từ giấy / mẫu in hiện tại | bố cục, thông tin bắt buộc, dấu và chữ ký |
| Ảnh chụp phần mềm cũ | luồng thao tác khách đã quen — thứ họ sẽ đem ra so sánh |
| Vài bản ghi thật (che phần nhạy cảm) | dữ liệu thật thường khác hẳn dữ liệu khách mô tả |

Một file Excel thật cho nhiều thông tin hơn một buổi phỏng vấn, vì nó chứa cả những
ngoại lệ mà người dùng không nhớ để kể.

**2. Hỏi người làm, không chỉ hỏi người yêu cầu.** Quản lý nói *"cần báo cáo tồn kho
theo lô"*; thủ kho mới biết *"nhưng hàng trả lại thì không có lô, tụi em ghi tay."*

**3. Hỏi ngoại lệ, đừng hỏi luồng chính.** Luồng chính khách kể được ngay và thường
đúng — dự án chết ở ngoại lệ:

- *"Trường hợp nào thì **không** áp dụng quy tắc này?"*
- *"Khi nào anh chị phải **sửa tay** sau khi hệ thống làm xong?"*
- *"Tháng vừa rồi có ca nào **bất thường** không?"*
- *"Nếu số liệu sai thì ai phát hiện, phát hiện bằng cách nào?"*

**4. Đưa bản nháp sai, đừng đưa form trống.** Form trống nhận lại câu trả lời trống.
Tự điền trước theo hiểu biết của mình rồi gửi kèm *"tôi hiểu thế này, sai chỗ nào anh
sửa giúp."* Chỗ khách sửa mạnh tay nhất chính là chỗ mình hiểu sai nhất.

**5. Hỏi quy mô** — khách trả lời được ngay mà lại quyết định cách làm: bao nhiêu bản
ghi, bao nhiêu người dùng cùng lúc, dùng trên máy tính hay điện thoại, tính năng này
thay thế cái gì đang chạy.

**6. Viết lại rồi bắt xác nhận.** Gửi lại bản tóm tắt mình hiểu và yêu cầu trả lời.
**Im lặng không phải là đồng ý.**

> **Mockup cũng là công cụ thu thập**, không chỉ để duyệt. Khách nhìn màn hình cụ thể
> sẽ nói ra thứ mà mười câu hỏi không lấy được: *"ủa sao không có cột đơn vị tính?"*
> Nên **đừng đợi hiểu đủ mới dựng mockup** — hiểu chưa đủ chính là lý do phải dựng.

Đủ để sang bước 1 khi: có ít nhất một hiện vật · đã hỏi người trực tiếp thao tác ·
đã hỏi ít nhất hai câu ngoại lệ · biết quy mô dữ liệu · bản tóm tắt đã có phản hồi.

### Bước 1: Làm rõ bằng 5 câu hỏi

| | Hỏi gì |
|---|---|
| **WHAT** | Kết quả mong muốn? Dữ liệu nào thay đổi? |
| **WHEN** | Lúc nào — nhập liệu, lưu, submit, theo lịch, khi bấm nút? |
| **WHO** | Vai trò nào bị ảnh hưởng / không nên bị ảnh hưởng? |
| **WHERE** | Hiện ở đâu — form, báo cáo, app mobile, hệ thống ngoài? |
| **ERROR** | Sai thì chặn hẳn, cảnh báo, hay âm thầm ghi log? |

**Chưa trả lời được cả 5 thì quay lại bước 0**, đặc biệt là xin hiện vật và hỏi người
trực tiếp thao tác. Sai ở WHO và ERROR gây rework tốn nhất — vì chúng thường chỉ lộ ra
khi hệ thống đã chạy thật.

### Bước 2: Chọn tầng

**Trước hết: kiểm tra app lõi đã có sẵn chưa.**

Các app lõi đã tích luỹ rất nhiều DocType qua nhiều dự án. Không ai nhớ hết, và
tài liệu của từng app cũng chưa liệt kê đủ. Bỏ qua bước này là làm lại thứ đã có —
kiểu lãng phí đắt nhất, vì nó còn kéo theo hai bản logic phải bảo trì song song.

```bash
ls apps/mbwnext_advanced_*/*/*/doctype/ apps/pos_next/*/*/doctype/
grep -rn "<từ khoá nghiệp vụ>" apps/*/*/*/doctype/ --include=*.json -l
```

Thấy có sẵn → dùng hoặc mở rộng cái đang có, **đừng dựng bản song song trong app khách**.

Sau đó mới trả lời theo thứ tự, **dừng ở câu ĐÚNG đầu tiên**:

1. Là quy định pháp luật VN (VAS, TT99/133/200, hoá đơn điện tử)? → **tầng 2**, `mbwnext_localization`
2. Đã có ≥2 khách cần, hoặc chắc chắn khách sau cũng cần? → **tầng 2**, app lõi tương ứng
3. Dùng lại được nhưng không phải khách nào cũng bật? → **tầng 3**, module dùng lại
4. Còn lại — quy trình nội bộ / thương hiệu / mẫu in riêng → **tầng 4**, app khách

**Luật vàng:** phân vân thì chọn **tầng cao hơn** (gần app khách hơn). Đưa nhầm xuống
lõi thì mọi khách phải gánh; để nhầm ở app khách chỉ tốn công nâng lên sau.

**Luật chống trùng lặp:** khi cùng một logic sắp xuất hiện ở **khách thứ 3** → dừng,
mở phiếu nâng lên tầng 2 hoặc 3. Không copy sang khách thứ 4.

### Bước 3: Viết spec

Viết nối tiếp vào chính file đầu bài — **giữ nguyên câu khách nói ở phần trên**. Sau
này khi tranh luận lúc nghiệm thu, cần đối chiếu "khách nói gì" với "mình hiểu thành gì".

---

## 6. Giai đoạn ② — Dev

Sáu bước, **không nhảy bước**:

```
① Đọc & làm rõ
      ↓
② DocType + Custom Field
      ↓
③ Mockup HTML  →  duyệt nội bộ  →  duyệt khách        ⛔ CỔNG CHẶN
      ↓
④ Viết logic  →  soát lại
      ↓
⑤ Bản dịch vi.po
      ↓
⑥ Cập nhật CLAUDE.md / README
```

### Vì sao dựng DocType trước mockup

Mockup phải phản ánh **cấu trúc dữ liệu thật**, không phải giao diện tưởng tượng rồi
nắn dữ liệu theo sau. Mockup dùng sai tên trường là mockup phải làm lại.

### Cổng chặn: mockup phải được duyệt trước khi viết logic

Mockup là file HTML tự chứa, mở bằng trình duyệt là xem được, lưu ở `docs/mockups/`.
Phải thể hiện đủ ba trạng thái: **rỗng · có dữ liệu · báo lỗi**.

Hai vòng duyệt:

| Vòng | Ai duyệt | Nội dung |
|---|---|---|
| 1 | Nội bộ | Được ghi kèm chi tiết kỹ thuật. Chốt bố cục và luồng thao tác |
| 2 | Khách hàng | Bỏ hết chi tiết kỹ thuật, dùng ngôn ngữ nghiệp vụ. Khách xác nhận "đúng cái tôi cần" |

**Không viết một dòng logic nào khi mockup chưa được khách duyệt.** Đây là điểm rẻ
nhất để phát hiện hiểu sai yêu cầu — sửa ở đây tốn vài phút, sửa sau khi code xong tốn
vài ngày.

Nếu lúc code phát hiện mockup bất khả thi về kỹ thuật → **quay lại bước ③**, sửa mockup
và duyệt lại. Không tự ý đổi giao diện đã chốt.

### 5 cách can thiệp vào app lõi — chọn cách nhẹ nhất đủ dùng

| # | Cách | Mức độ |
|---|---|---|
| 1 | `doc_events` — thêm hành vi, lõi vẫn chạy nguyên vẹn | nhẹ nhất, **mặc định nên dùng** |
| 2 | `doctype_js` — thêm hành vi phía client | nhẹ |
| 3 | `override_doctype_class` — thay class controller, luôn gọi `super()` | nặng |
| 4 | `override_whitelisted_methods` — thay hàm của lõi | nặng hơn |
| 5 | Nhánh riêng theo khách trên app lõi | **lối thoát cuối** |

Cách 5 là thiết kế có chủ đích của MBWNext, không phải sai lầm — nhưng chỉ dùng khi 4
cách trên **không diễn đạt được** yêu cầu (điển hình: sửa bên trong SPA Vue của
`pos_next`, nơi không có điểm móc nào). Trước khi mở nhánh phải trả lời được: *cách 1–4
tại sao không dùng được?*

Mở nhánh nghĩa là app đó **từ nay có hai bản**: bản chung và bản riêng của khách. Hai
bản này không tự đồng bộ với nhau.

Hệ quả rất cụ thể: mỗi lần app lõi sửa được một lỗi, khách chạy nhánh riêng **không tự
nhận được bản sửa đó** — phải có người mang sang. Bỏ lâu không mang, hai bản lệch dần;
tới lúc muốn mang thì vướng hàng đống xung đột, và cách dễ nhất lúc đó là bỏ luôn.
Khách đó từ đấy đứng ngoài mọi bản vá chung.

Chiều ngược lại cũng vậy: thứ gì sửa trên nhánh khách mà thực ra khách nào cũng cần,
phải mang về bản chung — nếu không khách sau lại ngồi làm lại từ đầu.

Nên khi ước lượng công cho một yêu cầu buộc phải mở nhánh, phần tốn không nằm ở lần
viết đầu tiên, mà ở **mọi lần đồng bộ về sau**.

### Quy tắc bắt buộc khi viết code

- **Override app lõi phải wrap-and-delegate** — gọi hàm gốc rồi bổ sung, tuyệt đối
  không chép lại thân hàm. Chép là mất mọi bản vá của lõi về sau.
- **Logic server** → `controllers/python/`, đặt tên theo DocType bị tác động.
- **Client script** → `controllers/js/<doctype>.js`, khai qua `doctype_js`.
  Không để trong `public/js/`, không tạo Client Script qua giao diện — cả hai cách đó
  không đi theo app khi deploy site mới và không nằm trong git.
- **Label viết tiếng Anh**, bản dịch tiếng Việt đặt trong `locale/vi.po` của chính app đó.
- **Comment và docstring viết tiếng Việt**, tên hàm và biến giữ tiếng Anh.
- **Custom Field khai theo quy ước của từng app, không có một kiểu chung cho mọi
  app** — xem skill của app đó trước khi khai:
  - `advanced_stock`, `mbwnext_localization`, app khách → gán `module` rồi
    `bench export-fixtures`
  - `advanced_accounting`, `advanced_buying`, `advanced_selling` → khai trong
    `setup/custom_fields.json` rồi `sync_custom_fields`
  - Dùng nhầm quy ước thì Custom Field không đi theo app, biến mất ở lần deploy
    site mới.

### Soát lại trước khi sang bước ⑤

Team ít người nên đây là **tự soát**, không phải review chéo. Đọc lại `git diff` của
chính mình — đọc diff bắt được những thứ khác hẳn lúc đang viết.

Năm điểm phải tự nhìn, vì **chúng không báo lỗi lúc chạy**, chỉ lộ ra khi deploy sang
site khác hoặc khi app lõi cập nhật:

1. Handler `doc_events` có `if doc.doctype != "...": return` chưa?
2. Override app lõi có gọi hàm gốc không?
3. `override_doctype_class` có `super().validate()` không?
4. Custom Field đã gán `module` chưa?
5. Client script có nằm trong `controllers/js/` không?

---

## 7. Giai đoạn ③ — Test case

Đầu ra: `docs/testcases/<tên>.md` — bảng kịch bản kiểm thử để QA chạy tay và khách ký
nghiệm thu.

### Mục đích: người khác đọc là tự test lại được

**Đây là thước đo duy nhất để biết test case viết đủ hay chưa.** Một người chưa từng
làm tính năng này mở file ra mà không biết đăng nhập ở đâu, bằng tài khoản nào, dữ
liệu nào — thì file chưa xong, dù bảng case đã kín.

Vì vậy mục **Điều kiện chuẩn bị** phải ghi đủ ba thứ:

| Ghi gì | Ví dụ |
|---|---|
| Địa chỉ site (môi trường develop) | `http://localhost:8045` |
| Tài khoản đăng nhập | `administrator` / `<mật khẩu>`, kèm role nếu case cần role khác |
| Dữ liệu cần có — **ghi đúng mã bản ghi** | Item `SP-001` có 2 UOM; Kho `Kho Hà Nội` |

Viết *"dùng một Item bất kỳ"* là chưa đủ — người sau lấy Item khác sẽ ra kết quả khác.

⚠ **Chỉ ghi tài khoản/mật khẩu của môi trường develop.** File test case nằm trong
`docs/testcases/` ở gốc bench — nếu bench này đẩy lên remote chung (kể cả repo
riêng, kể cả private), mật khẩu đi theo lịch sử git vĩnh viễn, xoá dòng sau này
không xoá được khỏi commit cũ. Tuyệt đối không chép tài khoản site production
của khách vào đây.

Dữ liệu sẵn có đáp ứng đủ thì dùng luôn; thiếu tổ hợp cần thiết thì tạo thêm (môi
trường develop nên không nặng nề), đặt tiền tố `TEST-` để xoá lại được. Phải tạo trên
5–10 bản ghi thì viết script vào `docs/testcases/data/<tên>.py`.

### Nguồn để viết — không viết từ trí tưởng tượng

| Nguồn | Cho ra |
|---|---|
| Spec | luồng nghiệp vụ đúng |
| **Đọc code vừa viết** | mỗi `frappe.throw` → một case; mỗi `return`/`continue` sớm → một case |
| `apps/*/*/hooks.py` | app nào cùng hook DocType này |

Bước "đọc code" tìm ra nhiều case nhất. **Mỗi nhánh thoát sớm trong code là một hành vi
người dùng sẽ gặp mà spec không hề nhắc tới.**

### 6 nhóm bắt buộc

| Mã | Nhóm | Trả lời câu hỏi |
|---|---|---|
| `TC-HAPPY` | Luồng đúng | Có làm được việc khách cần không? |
| `TC-VALID` | Kiểm tra dữ liệu | Nhập sai có bị chặn, báo lỗi có đúng câu không? |
| `TC-EDGE` | Biên & lặp | Rỗng, 0, trùng, submit 2 lần, cancel, amend? |
| `TC-PERM` | Phân quyền | Role thấp làm được gì không nên làm được? |
| `TC-REGR` | **Regression app lõi** | Tính năng mới có làm hỏng cái đang chạy không? |
| `TC-ISO` | **Cách ly app khách** | Site không cài app khách có bị ảnh hưởng không? |

Nhóm nào không áp dụng thì ghi lý do, **không xoá dòng**.

Hai nhóm cuối là đặc thù kiến trúc chồng tầng — bộ test thông thường luôn bỏ sót chúng,
và đó chính là nơi lỗi đắt tiền nằm.

**Nhóm thứ 7 — `TC-PWA` — chỉ khi tính năng có màn hình mobile.** Đây là loại lỗi đến
từ trình duyệt và mạng, sáu nhóm kia không bắt được: mất mạng giữa chừng · thao tác
offline rồi có mạng lại · **deploy bản mới nhưng service worker vẫn giữ cache cũ** ·
từ chối quyền camera · chạy nền lâu rồi quay lại · cài lên màn hình chính.

Bắt buộc test trên **máy Android thật**, không dùng emulator — quyền camera, hành vi
service worker khi hệ điều hành đóng app nền, và bàn phím ảo che ô nhập chỉ sai trên
máy thật.

### Quy tắc viết

- Kết quả mong đợi phải **kiểm chứng được**. "Hệ thống báo lỗi" là sai; phải chép
  **nguyên văn** câu lỗi trong code, vì khách sẽ tra tài liệu bằng cách gõ lại câu đó.
- Bước thực hiện viết bằng **ngôn ngữ người dùng thấy trên màn hình**, không dùng tên
  trường kỹ thuật.
- Một test case = một điều được kiểm chứng.
- Phải test cả **cancel** và **amend**, không chỉ submit — hai luồng này chạy lại toàn
  bộ hook và là nơi lỗi trùng dữ liệu lộ ra.
- Cột "Kết quả thực tế" chưa điền = test case chưa chạy = chưa xong.

### Ai chạy test

**Điều kiện:** Claude tự chạy thử được **chỉ khi** có bench/site dev đã dựng sẵn,
Claude truy cập được (site đang chạy, có site trong `sites.txt`, quyền thao tác đủ).
Không có môi trường đó thì bỏ qua vòng 1, đi thẳng vào vòng người test tay — đừng
suy đoán kết quả rồi ghi Pass thay cho việc chạy thật.

**Không dùng `FrappeTestCase`, không viết `test_*.py`.** Hai vòng:

```
Claude code xong  →  Claude tự chạy thử trên dev, điền kết quả thật vào file
                  →  Người test lại bằng tay theo đúng file đó
```

Claude chạy được phần **server**: tạo/sửa/submit chứng từ, kiểm tra dữ liệu sinh ra,
bắt lỗi validate, chạy lại hook — tức phần lớn `TC-HAPPY`, `TC-VALID`, `TC-EDGE`,
`TC-PERM`, `TC-REGR`.

Claude **không** thay được người ở: thao tác trên giao diện thật, toàn bộ `TC-PWA`,
mẫu in nhìn có đúng chỗ không, và trải nghiệm (chậm, khó bấm, rối). Những case đó
ghi `Cần người test`, **không ghi Pass**.

Ba luật khi Claude điền kết quả:

1. **Chỉ ghi Pass cho case đã thực sự chạy** — không suy từ đọc code ra "chắc đúng"
2. **Fail thì ghi Fail**, kèm nguyên văn thông báo lỗi
3. **Ghi rõ kiểm chứng bằng cách nào** — gọi API server hay bấm trên giao diện; gọi
   được API chỉ chứng minh backend đúng, không chứng minh người dùng bấm được

Kết quả Claude điền là **vòng một, không phải kết luận cuối**. Người test lại tay mới
là người chốt Pass/Fail để nghiệm thu. Vì vậy các bước thao tác vẫn phải viết theo
ngôn ngữ giao diện — người test lại sẽ bấm tay, không gọi API.

### Test xong có cần tạo artifact không?

**Không, trừ khi phải đưa cho người không đọc được bench** (khách nghiệm thu, quản lý).

File `docs/testcases/<tên>.md` là bản gốc duy nhất. Publish thêm một bản nghĩa là có hai
bản không tự đồng bộ — vài tháng sau ai đó mở link cũ và đọc kết quả đã lỗi thời mà
không biết. Cần đưa ra ngoài thì xuất theo yêu cầu, mỗi lần một bản mới, và bỏ thông
tin đăng nhập ra khỏi bản gửi khách.

---

## 8. Giai đoạn ④ — Tài liệu hướng dẫn sử dụng

Chỉ viết **sau khi test case đã chạy và Pass**. Viết tài liệu cho tính năng chưa nghiệm
thu là viết hai lần.

### Hai luật tách tài liệu

**Luật 1 — tài liệu đi theo tầng của code.** Tính năng lõi → tài liệu ở app lõi, mọi
khách dùng chung. Tính năng khách → tài liệu ở app khách. **Cấm chép nội dung lõi sang
tài liệu khách, phải dẫn link** — vì lõi còn thay đổi, chép nghĩa là 7 chỗ phải sửa mỗi
lần lõi đổi, và sẽ không ai sửa.

**Luật 2 — tách hai đối tượng đọc thành hai file:**

| File | Đối tượng | Tần suất dùng |
|---|---|---|
| `<tính-năng>-cau-hinh.md` | quản trị, tư vấn triển khai | một lần lúc go-live |
| `<tính-năng>-van-hanh.md` | thu ngân, kế toán, thủ kho | mỗi ngày |

Gộp hai thứ vào một file nghĩa là thu ngân phải lội qua 6 bước tạo tài khoản kế toán
mới tới được việc của mình.

### Tài liệu là sản phẩm dẫn xuất, không phải sáng tác

- `TC-HAPPY` → các bước thao tác, chép gần như nguyên xi
- `TC-VALID` → mục "Thông báo lỗi thường gặp"
- Mockup đã duyệt → bố cục màn hình, tên nút

Nếu phải viết lại từ đầu, nghĩa là test case đã viết sai chuẩn — quay lại sửa test case.

### Mục "Thông báo lỗi" là mục khách dùng nhiều nhất

Lấy danh sách lỗi từ chính code (`grep frappe.throw`), không viết theo trí nhớ. Cột
"Cách xử lý" phải là hành động cụ thể trên màn hình:

- ❌ "Do thiếu dữ liệu cấu hình"
- ✅ "Mở tab **Warehouse List**, thêm ít nhất một kho rồi lưu lại"

---

## 9. Vòng sửa lỗi

Khách báo lỗi thì **không đi lại từ đầu**, nhưng phải qua hai câu hỏi.

### Câu hỏi 1 — Có thật là lỗi không?

Mở lại spec đã chốt rồi phân loại:

| Khách nói | Thực chất | Xử lý |
|---|---|---|
| "Bấm Submit là văng lỗi" | **Lỗi** — code sai so với spec | sửa, miễn phí, nhanh |
| "Chỗ này phải cộng cả thuế nữa chứ" | **Yêu cầu mới** đội lốt lỗi | quay lại giai đoạn ①, có báo giá |
| "Tôi tưởng nó chạy kiểu khác" | **Hiểu sai** | không sửa code — bổ sung tài liệu |

Chạy đúng spec mà khách vẫn không hài lòng → yêu cầu mới, không phải lỗi. Bước phân
loại này sinh lời nhất, vì hai loại sau chiếm phần lớn và đều không nên sửa code.

### Câu hỏi 2 — Lỗi nằm ở tầng nào?

Một khách báo lỗi, nhưng lỗi có thể nằm ở app lõi — nghĩa là **các khách còn lại cũng
đang dính mà chưa ai báo**.

Thứ tự suy luận, dừng khi đủ chắc chắn:

1. **Đọc traceback** — đường dẫn file trong stack chỉ thẳng app nào
2. **Hỏi chéo khách khác** — khách không cài app này có gặp không?
3. **Đọc code đường đi** — hàm đó có bị app khách override không?
4. Chắc chắn nhất, tốn nhất: dựng **site mới** chỉ cài app lõi rồi tái hiện

⚠ Cẩn thận: stack trace dừng ở app lõi **không** có nghĩa lỗi ở lõi. Có thể app khách
đã ghi dữ liệu sai ở một hook chạy trước, rồi lõi mới vấp phải.

### Tầng quyết định tốc độ, không phải mức độ gấp

Khách gấp không có nghĩa được đi nhanh. Lỗi ở app khách thì sửa nhanh được — sai chỉ
ảnh hưởng một khách. Lỗi ở app lõi thì **không**, vì sửa một dòng chạm mọi khách.

### Luật chống tái phát

> **Mỗi lỗi được sửa phải thành một dòng mới trong file test case.**
> Viết dòng đó **trước** khi sửa code, xác nhận nó Fail; sửa xong xác nhận nó Pass.

Chi phí 2 phút, đổi lại lỗi đó không quay lại lần thứ ba. File test case là thứ duy
nhất giữ được trí nhớ về những gì từng hỏng.

---

## 10. Bộ skill hỗ trợ

Quy trình trên không chỉ nằm trên giấy — nó đã được đóng gói thành **skill cho Claude
Code**. Skill là một tập hướng dẫn mà Claude tự nạp khi gặp đúng loại việc, để nó làm
theo cách của MBWNext thay vì cách mặc định của Frappe/ERPNext.

**10 skill MBWNext** dưới đây là **bắt buộc** — chúng chứa quy trình và quy ước riêng
của nền tảng này, không skill nào khác thay được. Cài global ở `~/.claude/skills/`,
mỗi người trong team cần có bản cài của riêng mình.

Ngoài 10 skill đó, bộ cài đặt còn có **~28 skill nền** (nhóm `erpnext-syntax-*`,
`erpnext-impl-*`, `erpnext-errors-*`, `erpnext-api-patterns`…) — đây là **skill hỗ
trợ khi viết code Frappe/ERPNext nói chung** (cú pháp Client Script, Server Script,
hooks.py, xử lý lỗi theo từng loại…), không đặc thù MBWNext. Chúng tự nạp khi cần,
không phải học riêng, và không nằm trong phạm vi tài liệu này.

### Nhóm 1 — Bốn skill quy trình

Ứng đúng bốn giai đoạn trong tài liệu này:

| Giai đoạn | Skill | Chứa gì |
|---|---|---|
| ① Phân tích | `erpnext-mbwnext-requirement-analysis` | 5W, bảng chọn tầng, định tuyến app, template spec, **vòng sửa lỗi** |
| ② Dev | `erpnext-mbwnext-customer-app` | quy trình 6 bước, cổng mockup, 5 cách can thiệp, fixtures, quy ước code |
| ③ Test case | `erpnext-mbwnext-testcase` | 6 nhóm TC + TC-PWA, cách đọc code để tìm case, chia việc Claude tự chạy / người test lại |
| ④ Tài liệu | `erpnext-mbwnext-user-guide` | tách tài liệu lõi/khách, tách cấu hình/vận hành, quy ước ảnh chụp |

### Nhóm 2 — Sáu skill theo app

Không thuộc quy trình, nhưng là thứ dùng hằng ngày. Mỗi skill mô tả cấu trúc, quy ước
và cạm bẫy riêng của một app lõi:

| Skill | Trả lời những câu như |
|---|---|
| `erpnext-mbwnext-localization` | MBWNext System Setting có gì? Chart of Accounts TT99/133/200 nạp thế nào? |
| `erpnext-mbwnext-accounting` | Luồng Purchase Voucher chạy ra sao? Báo cáo VAS đặt ở đâu? Và **danh mục đủ 12 DocType** (đối soát, ngân sách, các cặp mẫu+chứng từ…) |
| `erpnext-mbwnext-buying` | Quy trình KCS, Goods Receipt hoạt động thế nào? |
| `erpnext-mbwnext-selling` | Sell-in/Sell-out, in tem barcode, QR thanh toán Sepay, và **danh mục đủ 22 DocType** (chỉ tiêu, viếng thăm, phân loại khách…) |
| `erpnext-mbwnext-stock` | Max Stock Qty, tách hàng lỗi, hai app PWA mobile |
| `erpnext-pos-next` | Frontend Vue 3, `apiWrapper`, offline mode, Wallet & Loyalty, và **danh mục đủ 11 DocType** (ca bán hàng, ví, khuyến mãi…) |

Ba skill `selling`, `accounting`, `pos_next` mới được bổ sung **danh mục DocType đầy
đủ** — trước đây mỗi skill chỉ liệt kê vài DocType hay dùng, giờ liệt kê hết những gì
app đã có. Kiểm tra danh mục này **trước khi dựng DocType mới**, để không làm lại thứ
đã tồn tại (xem thêm ở Bước 2 của Giai đoạn ①).

Giá trị lớn nhất của nhóm này là **các khác biệt giữa app mà không ai nhớ hết**. Ví dụ:
Custom Field ở `mbwnext_advanced_stock` và `mbwnext_localization` khai bằng
`bench export-fixtures`, còn ở `advanced_accounting`, `advanced_buying`,
`advanced_selling` lại khai bằng `setup/custom_fields.json` + `sync_custom_fields`.
Làm sai thì trường không đi theo app.

### Cách dùng

**Không cần gọi tên skill.** Cứ mô tả việc bằng ngôn ngữ bình thường, skill phù hợp sẽ
tự nạp:

- *"Khách Gosumo muốn khi submit phiếu nhập thì tự gán mùa vụ xuống lô hàng"* → nạp
  `requirement-analysis`, rồi `customer-app`
- *"Đây là file tính năng cần code cho app khách"* → nạp `customer-app`, và nó sẽ **chặn
  lại ở bước mockup** thay vì code luôn
- *"Viết test case cho tính năng vừa xong"* → nạp `testcase`
- *"Khách báo lỗi khi bấm Submit"* → nạp vòng sửa lỗi trong `requirement-analysis`

Muốn gọi thẳng thì gõ `/` rồi tên skill.

### Giới hạn cần biết

- **Skill hướng dẫn Claude, không thay thế người soát.** Bước "soát lại `git diff`" ở
  giai đoạn ② vẫn là việc của người viết code.
- **Skill có thể lạc hậu.** Nó mô tả cấu trúc app tại thời điểm viết. Đổi cấu trúc app
  thì phải cập nhật skill, nếu không nó sẽ hướng dẫn theo cái đã cũ.
- **Skill không biết dữ liệu thật của khách.** Nó biết cách làm, không biết site nào
  đang cấu hình ra sao.
- **Skill nạp theo mô tả công việc**, nên mô tả càng rõ thì nạp càng đúng. Nói "sửa cái
  này" thì không đủ tín hiệu để nạp skill nào cả.

---

## 11. Ba cổng chặn cần nhớ

Nếu chỉ nhớ được ba điều từ tài liệu này:

| Cổng | Nội dung |
|---|---|
| **Trước khi code** | Mockup phải được khách duyệt |
| **Trước khi viết tài liệu** | Test case phải chạy xong và Pass |
| **Trước khi chọn app** | Phải chốt tầng, và phân vân thì chọn tầng cao hơn |

---

## 12. Câu hỏi team thường đặt

**"Tính năng nhỏ có cần làm đủ 4 bước không?"**
Có, nhưng khối lượng tự co lại theo tính năng. Thêm một Custom Field thì mockup là một
ảnh chụp có chú thích, test case ba dòng, tài liệu một đoạn — tổng cộng vài chục phút.
Cái không được bỏ là **thứ tự**, vì bỏ thứ tự mới là chỗ sinh việc làm lại.

**"Làm mockup có làm chậm không?"**
Ngược lại. So cùng một tính năng có hiểu nhầm về giao diện:

| | Cách cũ | Cách mới |
|---|---|---|
| Phát hiện hiểu nhầm | sau khi code xong, lúc demo | trước khi code, lúc duyệt mockup |
| Sửa | code lại phần UI + phần logic bám theo | sửa file HTML |
| Chi phí | vài ngày | vài chục phút |
| Phụ thêm | giải thích với khách vì sao làm lại | không có |

Mockup không thêm việc — nó **dời việc phát hiện sai** lên chỗ rẻ nhất.

**"Chốt tầng có làm chậm không?"**
Tốn thêm vài phút trả lời 4 câu hỏi, đổi lại không phải chép logic sang khách thứ hai.
Một lần chép sang khách khác — kể cả copy-paste — vẫn tốn hơn nhiều lần chốt tầng đúng,
vì từ đó trở đi mọi lần sửa đều phải sửa hai chỗ.

**"Khách giục gấp thì có được bỏ bước không?"**
Bước được rút gọn là mockup (dùng ảnh thay HTML) và tài liệu (viết sau). Bước **không**
được bỏ là chốt tầng và test case — đó là hai chỗ mà sai lầm lan sang khách khác.

**"Claude đã tự test rồi, sao còn phải test lại tay?"**
Vì Claude chỉ chạy được phần server. Nó không bấm được nút, không thấy được cái người
dùng thấy, không cảm nhận được chậm hay khó dùng. Kết quả Claude điền là vòng một để
loại sớm lỗi rõ ràng; vòng tay mới là vòng chốt nghiệm thu.

**"Vậy có viết test tự động không?"**
Không. Không dùng `FrappeTestCase`, không thêm `test_*.py`. Thứ giữ trí nhớ về những
gì từng hỏng là file test case — mỗi lỗi sửa xong thành một dòng mới trong đó.

**"Comment tiếng Việt có kỳ không?"**
Người bảo trì các app này là team Việt Nam. Comment để người sau đọc hiểu, nên viết bằng
ngôn ngữ người sau đọc nhanh nhất. Tên hàm và biến vẫn tiếng Anh vì chúng là code.

**"Tại sao label tiếng Anh mà comment tiếng Việt?"**
Label là dữ liệu hệ thống, đi qua cơ chế dịch của Frappe — viết tiếng Anh rồi dịch trong
`vi.po` thì cùng một hệ thống phục vụ được cả khách dùng giao diện tiếng Anh. Comment
không đi qua cơ chế nào cả, nó chỉ để người đọc.

---

## 13. Tóm tắt một trang

| Giai đoạn | Làm gì | Để lại file gì | Cổng chặn |
|---|---|---|---|
| ① Phân tích | thu thập hiện vật → 5W → kiểm tra app lõi có sẵn → chốt tầng → viết spec | `docs/features/<tên>.md` | chưa có hiện vật + chưa đủ 5W thì chưa đi tiếp |
| ② Dev | DocType → mockup → logic → soát lại → dịch → tài liệu app | `docs/mockups/<tên>.html` + code + `locale/vi.po` + `CLAUDE.md`/`README.md` (nếu đổi cấu trúc) | mockup chưa duyệt, không code |
| ③ Test case | 6 nhóm + TC-PWA → Claude tự chạy thử, điền kết quả → người test lại tay | `docs/testcases/<tên>.md` | chưa điền kết quả thật, chưa xong |
| ④ Tài liệu | tách cấu hình / vận hành | `apps/<app>/docs/huong-dan/` | test chưa Pass, chưa viết |

**Ba luật vàng**

1. Phân vân chọn tầng → chọn tầng cao hơn (gần app khách hơn)
2. Logic lặp tới khách thứ 3 → dừng, nâng lên tầng lõi
3. Mỗi lỗi đã sửa → một dòng test case mới
