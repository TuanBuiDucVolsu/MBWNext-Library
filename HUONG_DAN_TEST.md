# Hướng dẫn tự test robot (dành cho người mới)

Tài liệu này dành cho sinh viên **chưa từng dùng terminal/dòng lệnh**, muốn tự kiểm tra
robot hoạt động đúng chưa — không cần hiểu code, chỉ cần làm theo từng bước.

> **Điều kiện trước khi đọc tài liệu này:** Pi đã được cài đặt xong theo
> [SETUP_PI.md](SETUP_PI.md) (đã clone code, cài thư viện, đấu nối phần cứng).
> Nếu chưa, nhờ người phụ trách cài trước hoặc làm theo tài liệu đó trước.

---

## 0. Làm quen terminal (3 phút)

**Terminal** là cửa sổ gõ lệnh thay vì bấm chuột. Bạn sẽ dùng nó để "nói chuyện" với Pi
từ xa qua WiFi (gọi là **SSH**) — không cần cắm màn hình vào Pi.

### Mở terminal trên laptop

| Hệ điều hành | Cách mở |
|---|---|
| Windows | Mở **PowerShell** (bấm Start, gõ "powershell") |
| Mac | Mở **Terminal** (Spotlight, gõ "terminal") |
| Linux | Mở **Terminal** |

### Kết nối vào Pi

```bash
ssh pi@<IP-của-Pi>
# Ví dụ: ssh pi@192.168.1.50
```

Gõ mật khẩu Pi khi được hỏi (gõ xong **không hiện chữ gì** trên màn hình, đó là bình
thường — cứ gõ rồi Enter). Nếu không biết IP Pi, hỏi người phụ trách hoặc xem
[SETUP_PI.md](SETUP_PI.md) bước 4.

Thấy dòng đại loại `pi@raspberrypi:~ $` là đã **vào được Pi thành công**.

### 5 lệnh cần biết

| Lệnh | Ý nghĩa |
|---|---|
| `cd ~/Robocon-BCIT` | Di chuyển vào thư mục code robot |
| `ls` | Xem có những file/thư mục gì trong thư mục hiện tại |
| `source ~/robot_env/bin/activate` | Bật "môi trường Python" của robot (bắt buộc trước khi chạy test — dòng lệnh sẽ hiện thêm `(robot_env)` phía trước) |
| `python3 tests/test_xxx.py` | Chạy 1 bài test |
| `Ctrl + C` | **Dừng khẩn cấp** bài test đang chạy (giữ Ctrl, bấm C) |

### Trước khi chạy bất kỳ test nào, luôn gõ 3 lệnh sau

```bash
cd ~/Robocon-BCIT
source ~/robot_env/bin/activate
```

Nếu thấy `(robot_env)` xuất hiện ở đầu dòng lệnh → sẵn sàng chạy test.

---

## 1. An toàn — đọc trước khi bật động cơ

- ⚠️ **Kê robot lên đế** (bánh xe không chạm đất) khi test động cơ lần đầu hoặc test
  xoay — robot có thể lao đi bất ngờ nếu code/dây có lỗi.
- ⚠️ Khi test cơ cấu nâng (`test_lift.py`), để tay tránh xa 2 càng — càng có thể kẹp.
- ⚠️ Luôn nhớ vị trí phím `Ctrl+C` — dừng ngay nếu robot làm gì bất thường.
- ⚠️ Không chạy test phần cứng khi robot đang tự chạy thi đấu (systemd service).
  Nếu không chắc, gõ trước: `sudo systemctl stop robot` (không báo lỗi gì là ổn, kể cả
  khi service chưa từng được cài).

---

## 2. Thứ tự test khuyến nghị

Làm **theo đúng thứ tự** — mỗi bước tiếp theo giả định bước trước đã chạy được (chốt
xong số đo thì mới nên test bước sau, tránh tốn thời gian debug nhầm nguyên nhân).

```
① test_logic  →  ② calibrate line  →  ③ test_motion  →  ④ test_lift
      →  ⑤ test_vision  →  ⑥ test_smoke  →  ⑦ chạy luyện tập full
```

---

## ① Test logic (không cần lên sân, không cần phần cứng)

Kiểm tra "bộ não" của robot (tính đường đi, phân loại...) bằng máy tính giả lập —
không động vào motor/cảm biến thật.

```bash
cd ~/Robocon-BCIT
source ~/robot_env/bin/activate
python3 -m unittest tests.test_logic -v
```

**Kết quả ĐÚNG (PASS):** rất nhiều dòng `ok`, dòng cuối cùng là:
```
OK
```

**Kết quả SAI (FAIL):** dòng cuối là `FAILED (...)` — có nghĩa là *code* bị lỗi (không
phải phần cứng). Chụp màn hình báo cho người phụ trách, đừng tự sửa `main.py`/`config.py`
nếu chưa quen code.

> Các dòng `[WARNING] PinFactoryFallback` màu vàng là **bình thường** khi chạy trên máy
> không có GPIO thật — không phải lỗi.

---

## ② Chốt cảm biến dò line (bắt buộc trước khi test di chuyển)

Cảm biến dò line (QTR-8A, 6 mắt dưới gầm robot) cần "học" đâu là line đen, đâu là nền
trắng trước khi robot có thể bám line.

```bash
python3 -m tools.calibrate_line
```

Làm theo hướng dẫn trên màn hình (thường là: đặt robot lên line đen, di chuyển qua lại
vài giây theo yêu cầu). Công cụ sẽ tự in ra 2 giá trị cần copy vào `config.py` — **nhờ
người phụ trách xác nhận trước khi sửa file** nếu đây là lần đầu bạn làm.

---

## ③ Test động cơ & di chuyển — `test_motion.py`

```bash
python3 tests/test_motion.py
```

Màn hình hiện **menu số** — gõ số rồi Enter để chọn bài test, gõ xong 1 bài có thể chạy
tiếp bài khác (menu hiện lại). `Ctrl+C` để thoát hẳn.

Chạy **theo thứ tự sau** (không cần làm hết mọi option, đây là bộ tối thiểu):

| Bước | Gõ số | Test | Robot làm gì | Kết quả mong đợi |
|:---:|:---:|---|---|---|
| 1 | `5` | Calibrate QTR-8A (raw) | Không di chuyển, chỉ đọc số | Thấy 6 số thay đổi khi đưa tay/line qua từng mắt |
| 2 | `4` | Đọc cảm biến line digital | Không di chuyển | 6 số 0/1, đổi giữa 0↔1 khi đưa qua line đen |
| 3 | `1` | Tiến/Lùi 2 giây | **Robot di chuyển** — kê lên đế trước! | Tiến đúng hướng "trước" của robot rồi lùi đúng hướng ngược lại |
| 4 | `2` | Xoay trái/phải | **Robot xoay tại chỗ** | 2 bánh quay ngược chiều nhau, robot xoay không lết ngang |
| 5 | `10` | Xoay 90° (calibrate) | Robot xoay 1 góc | Dùng để chỉnh số `TURN_TIME` — xoay đúng 90° là đạt, lệch thì làm theo hướng dẫn trên màn hình để chỉnh |
| 6 | `6` | Thoát ô start | Đặt robot vào ô start, quay mặt sang trái (9h, hướng về Kệ 3) | Robot tiến thẳng và dừng đúng khi chạm line |
| 7 | `7` | Bám line 10 giây | **Đặt robot lên line thật** | Robot chạy dọc line, không lạc ra ngoài |
| 8 | `8` | Cảm biến siêu âm | Không di chuyển | Số khoảng cách (cm) thay đổi khi đưa tay/vật lại gần |

**Dấu hiệu có vấn đề** (không phải lỗi thao tác của bạn — báo người phụ trách):
- Bánh xe quay ngược chiều mong đợi ở option `1`/`2` → dùng `d` (chẩn đoán từng bánh) để
  xác định bánh nào sai, xem [DEBUG_DONG_CO.md](../tests/DEBUG_DONG_CO.md).
- Cảm biến line toàn đọc `0` hoặc toàn `1023` → xem
  [DEBUG_CAM_BIEN_LINE.md](../tests/DEBUG_CAM_BIEN_LINE.md).
- Robot lạc khỏi line liên tục ở option `7` → có thể cần calibrate lại bước ② hoặc chỉnh
  `LINE_KP`/`LINE_KD` (báo người phụ trách).

---

## ④ Test cơ cấu nâng — `test_lift.py`

```bash
python3 tests/test_lift.py
```

⚠️ Tay tránh xa 2 càng khi test.

| Bước | Gõ số | Test | Kết quả mong đợi |
|:---:|:---:|---|---|
| 1 | `a` | Scan 8 kênh MCP3008 | Che tay vào cảm biến IR → thấy đúng 1 kênh (trái) và 1 kênh (phải) thay đổi rõ rệt |
| 2 | `8` | Đọc IR real-time | Đặt/nhấc pallet ra khỏi càng → số đổi ngay lập tức, cả 2 bên |
| 3 | `1` | Nâng/Hạ cơ bản | Cả 2 càng nâng lên rồi hạ về sàn, không kẹt, không lệch bên |
| 4 | `2` | Các tầng kệ 1/2 | Nâng đúng độ cao tầng 1, tầng 2, hạ về sàn — không đụng kệ |
| 5 | `3` | Pickup/Dropoff tầng 1 | Đặt 1 pallet có kiện hàng ở tầng 1 → robot nâng, IR báo nhận đủ, rồi hạ lại |
| 6 | `5` | Drop từng càng | Kiểm tra thả 1 kiện, giữ kiện còn lại — dùng khi giao hàng cho 2 nhà máy khác nhau |

**Dấu hiệu có vấn đề:**
- Chỉ 1 bên IR đổi số dù cả 2 càng đều có pallet → kiểm tra lại dây/channel, dùng option `a`.
- Càng nâng lệch (1 bên cao hơn) → cần chỉnh bù lệch, dùng option `d` (calibrate).

---

## ⑤ Test camera & nhận diện kiện hàng — `test_vision.py`

Trước tiên, cần chụp ảnh mẫu 4 loại kiện hàng (chỉ cần làm **1 lần**, hoặc lại làm nếu
đổi camera/kiện hàng thật):

```bash
python3 -m tools.capture_templates
```

Sau đó test nhận diện:

```bash
python3 tests/test_vision.py
```

| Bước | Gõ số | Test | Kết quả mong đợi |
|:---:|:---:|---|---|
| 1 | `1` | Chụp ảnh | In `Camera hoạt động OK!`, có file ảnh lưu lại |
| 2 | `6` | Nhận diện cặp 2 kiện | Đặt 2 kiện cạnh nhau trên kệ, hướng camera vào → in `✅ Nhận diện đủ 2 kiện` kèm đúng tên (Samsung/Foxconn/Amkor/Hana Micron) |
| 3 | `5` | Đánh giá độ ổn định (10 lần) | In `→ Ổn định: chỉ nhận 1 loại duy nhất` cho mỗi kiện — nếu thấy `⚠ CẢNH BÁO: nhận nhiều loại khác nhau` là ánh sáng/màu chưa ổn, cần chỉnh HSV (option `2`) |

**Lưu ý:** ánh sáng ở sân thi đấu khác phòng tập → nên chạy lại option `5` ngay tại sân
trước khi thi.

---

## ⑥ Smoke test — chạy thử trên sa bàn thật

Ghép nhiều bước lại, gần giống thi đấu thật nhưng có dừng lại kiểm tra từng phần.

```bash
python3 tests/test_smoke.py
```

| Bước | Gõ số | Test | Kết quả mong đợi |
|:---:|:---:|---|---|
| 1 | `1` | Exit start → Kệ 3 | In `✅ exit_start_zone OK` |
| 2 | `2` | Pickup 1 lượt trọn vẹn | In lần lượt `✅ approach_shelf OK` → `✅ classify_pair OK` → `✅ pickup OK` |
| 3 | `3` | Drop từng càng | Cả 2 kiện được thả đúng, IR xác nhận |

Nếu bất kỳ dòng nào hiện `❌`, dừng lại — đó là vị trí đang lỗi, không cần chạy tiếp các
bước sau (chúng phụ thuộc bước trước thành công).

---

## ⑦ Chạy luyện tập lặp (giống thi đấu, tự động reset)

Khi mọi test riêng lẻ ở trên đều đạt, chạy full nhiệm vụ nhiều lần liên tiếp:

```bash
bash scripts/practice.sh
```

- Robot chờ nhấn nút → chạy hết 1 lượt (240s hoặc đến khi xong) → tự dừng, chờ nút tiếp.
- Sau mỗi lượt: **đặt robot về ô xuất phát (quay mặt 9h)** rồi nhấn nút để chạy lại.
- `Ctrl+C` để thoát hẳn.

---

## Bảng checklist tổng hợp (in ra dán cạnh sa bàn)

```
[ ] ① test_logic              → dòng cuối "OK"
[ ] ② calibrate_line          → đã copy số vào config.py
[ ] ③ test_motion  #5 #4      → cảm biến line đọc số hợp lý
[ ] ③ test_motion  #1 #2      → robot tiến/lùi/xoay đúng chiều (đã kê đế)
[ ] ③ test_motion  #10        → xoay 90° chính xác
[ ] ③ test_motion  #6         → thoát ô start OK
[ ] ③ test_motion  #7         → bám line không lạc
[ ] ④ test_lift    #a #8      → IR trái/phải đổi số đúng khi che
[ ] ④ test_lift    #1 #2      → nâng/hạ mượt, không lệch bên
[ ] ④ test_lift    #3         → pickup/dropoff tầng 1 OK
[ ] ⑤ test_vision  #6         → nhận diện đúng tên kiện hàng
[ ] ⑤ test_vision  #5         → ổn định, không đổi loại giữa 10 lần
[ ] ⑥ test_smoke   #1 #2 #3   → không có dòng ❌
[ ] ⑦ scripts/practice.sh     → chạy trọn 1 lượt không dừng giữa chừng
```

---

## Khi nào cần gọi người phụ trách

- Bất kỳ test nào hiện `❌` hoặc `FAILED` **2 lần liên tiếp** sau khi thử lại.
- Cần sửa `config.py`/`main.py` nhưng chưa từng đọc code robot trước đó.
- Robot có hành vi bất thường không khớp mô tả trong bảng trên (ví dụ: khói, mùi khét,
  tiếng động cơ bất thường) → **rút nguồn ngay**, không cố chạy tiếp.

## Xem thêm

- [DEBUG_CAM_BIEN_LINE.md](../tests/DEBUG_CAM_BIEN_LINE.md) — cảm biến line đọc sai/ngược
- [DEBUG_DONG_CO.md](../tests/DEBUG_DONG_CO.md) — bánh xe chạy ngược/không dừng/lệch
- [TEST_CASE.md](../tests/TEST_CASE.md) — bảng đầy đủ mọi option test (bản kỹ thuật, dành cho người đã quen code)
- [SETUP_PI.md](SETUP_PI.md) — cài đặt Pi từ đầu
