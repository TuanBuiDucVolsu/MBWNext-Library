# Cách hoạt động của Robot — Bảng O2


---

## 1. Nhiệm vụ Robot

Robot thi đấu **một mình, hoàn toàn tự động** trong **240 giây** (4 phút) trên một
sa bàn lớn 4m × 2m. Sau khi bấm nút khởi động là **không ai được chạm vào nữa**.

Có 2 nhiệm vụ:

- **Nhiệm vụ 1 — Phân loại & giao hàng (240 điểm):**
  Lấy **12 kiện hàng** từ kho hải quan, dùng **camera nhận diện** xem mỗi kiện thuộc
  loại nào, rồi **chở đến đúng nhà máy** tương ứng.
  - Có 4 loại hàng → 4 nhà máy: **Samsung, Foxconn, Amkor, Hana Micron**.
  - Mỗi kiện giao đúng = 20 điểm. 12 kiện = 240 điểm.

- **Nhiệm vụ 2 — Kho hàng rời (30 điểm):**
  Sau khi xong 100% nhiệm vụ 1, lấy thêm 1 kiện ở kho hàng rời, giao cho **nhà máy
  liên hợp** ở giữa sân.

**Tổng tối đa: 270 điểm.**

---

## 2. Robot gồm những bộ phận gì?

Hình dung robot như một chiếc **xe nâng mini tự lái**:

| Bộ phận | Vai trò |
|---------|------------------|
| **Máy tính nhỏ (Raspberry Pi)** |  Điều khiển mọi thứ |
| **Camera** | Nhìn màu kiện hàng để biết loại nào |
| **Dãy cảm biến dò line** | Luôn nhìn xuống vạch kẻ để bám đường |
| **Cảm biến siêu âm phía trước** | Biết còn cách kệ bao xa để dừng đúng |
| **2 càng nâng (kiểu xe nâng)** | Xúc và nâng kiện hàng, thả riêng từng bên |
| **Cảm biến trên càng** | Xxác nhận đã thật sự nâng được hàng chưa |
| **2 động cơ bánh xe** | Chạy tới/lui, rẽ trái/phải |
| **Nút khởi động** | Người chơi bấm 1 lần để robot bắt đầu |

Toàn bộ xử lý **ngay trên robot, không cần Internet**.

---

## 3. Sa bàn trông như thế nào?

Sân được chia thành lưới các **vạch line màu đen trên nền sáng**. Robot di chuyển bằng
cách **bám theo các vạch đen này**, giống tàu chạy trên đường ray.

Bố trí phần sân của đội:

- **Bên trái:** 3 giá kệ chứa 12 kiện hàng (kho hải quan).
- **Cột giữa:** 5 nhà máy xếp dọc (Samsung, Hana, Liên hợp, Amkor, Foxconn).
- **Phía dưới:** ô xuất phát của robot + 1 kệ kho hàng rời (cho nhiệm vụ 2).

Các vạch line cắt nhau tạo thành **"giao lộ"**. Robot điều hướng bằng cách **đếm số
giao lộ đã đi qua** rồi rẽ tại đúng giao lộ cần thiết — như kiểu "đi thẳng qua 2 ngã
tư rồi rẽ phải".


---

## 4. Luồng chính trận đấu

1. **Xuất phát:** Người chơi bấm nút. Robot rời ô xuất phát, tiến tới chạm vạch line
   đầu tiên và bắt đầu bám đường.

2. **Đi đến kệ:** Robot bám line, đếm giao lộ, tới đúng giá kệ đang cần lấy hàng.

3. **Tiến sát kệ:** Dùng cảm biến siêu âm, robot tiến **nhanh khi còn xa, chậm lại khi
   gần** để dừng chính xác trước kệ mà không đâm vào.

4. **Nhìn & nhận diện:** Camera chụp **2 kiện hàng** đặt cạnh nhau trên kệ, phân tích
   màu để biết mỗi kiện thuộc nhà máy nào.

5. **Nâng 2 kiện cùng lúc:** Hai càng xúc và nâng cả 2 kiện lên. Cảm biến trên càng
   **xác nhận đã nâng chắc** rồi robot mới lùi ra.

6. **Tính đường giao thông minh:** Robot tự tính **nên giao kiện nào trước** để quãng
   đường ngắn nhất (tiết kiệm thời gian).

7. **Giao hàng:** Chở đến nhà máy thứ nhất, **thả đúng 1 kiện** (đúng càng giữ kiện đó).
   Nếu 2 kiện khác nhà máy thì chạy tiếp đến nhà máy thứ hai, thả nốt kiện còn lại.
   *(Nếu 2 kiện cùng nhà máy thì thả cả 2 một lần.)*

8. **Quay về lấy lượt tiếp:** Robot về kho, lên kệ/tầng tiếp theo và **lặp lại** các
   bước trên.

9. **Lặp 6 lượt** (mỗi lượt 2 kiện) → giao đủ **12 kiện** → xong Nhiệm vụ 1.

10. **Nhiệm vụ 2:** Ghé kho hàng rời lấy 1 kiện, chở lên nhà máy liên hợp ở giữa,
    thả xuống → **hoàn thành**.

11. **Kết thúc:** Robot dừng tại chỗ. Cả quá trình luôn được canh giờ để **chắc chắn
    dừng trước khi hết 240 giây**.

---

## 5. Ba "kỹ năng" cốt lõi của robot

### a) Bám đường & điều hướng
Dãy cảm biến dưới gầm liên tục dò vạch đen. Nếu robot lệch, nó **tự bẻ lái về giữa
vạch** (nhẹ nhàng, không giật). Khi gặp giao lộ, robot biết đó là điểm để **đếm** hoặc
để **rẽ**. Nếu lỡ mất vạch, robot **quét trái–phải để tìm lại**.

### b) Nhận diện hàng bằng màu (Camera AI, không cần mô hình nặng)
Mỗi loại hàng có màu đặc trưng:

| Kiện | Màu nhận biết | Giao cho |
|------|---------------|----------|
| 01 | Xanh dương (chip) | Samsung |
| 02 | Vàng đồng (chip) | Foxconn |
| 03 | Xám bạc (khối nhôm) | Amkor |
| 04 | Đỏ (mã QR viền đỏ) | Hana Micron |

Robot **ưu tiên các màu rực (xanh/vàng/đỏ)** trước, vì nền trắng/xám dễ gây nhầm với
loại "xám" (Amkor). Nhờ vậy ít nhận sai hơn.

### c) Gắp & đặt hàng chính xác
- **Tiến 2 pha** (nhanh→chậm) để dừng đúng vị trí, không đâm kệ.
- **Càng nâng kiểu xe nâng**, nâng 2 kiện cùng lúc nhưng **thả được riêng từng càng**
  (để giao 2 nhà máy khác nhau).
- **Cảm biến xác nhận** mỗi lần nâng/thả — nếu chưa chắc thì **thử lại**, không "đoán mò".

---

## 6. Nếu robot gặp sự cố thì sao?

Robot có nhiều "lưới an toàn" xếp tầng:

- **Lỗi nhỏ** (đi lệch, nhận diện không rõ, nâng hụt): **thử lại**, vẫn không được thì
  **bỏ qua kiện đó, đi tiếp** — mất 1 kiện chứ không hỏng cả trận.
- **Kẹt một chỗ:** có giới hạn thời gian cho mỗi thao tác → không bao giờ đứng ì mãi.
- **Sắp hết giờ:** robot tự biết và **dừng an toàn trước vạch 240 giây**.
- **Lỗi nặng bất ngờ:** robot **dừng máy an toàn rồi tự khởi động lại**; khi bấm nút,
  nó **chạy tiếp phần thời gian còn lại** (không reset về 240 giây). Chỉ cần đặt robot
  về ô xuất phát và bấm nút.

---

## 7. Ba chế độ chạy

| Chế độ | Dùng khi | Đặc điểm |
|--------|----------|----------|
| **Luyện tập** | Tập hằng ngày | Xong 1 lượt **tự reset, bấm nút là chạy lại** — không cần khởi động lại máy |
| **Điều khiển tay (web)** | Kiểm tra từng bộ phận | Mở trình duyệt, bấm nút điều khiển bằng tay |
| **Thi đấu** | Ngày thi | Bật nguồn là vào thế sẵn sàng, bấm nút 1 lần là tự chạy hết trận |

---

## 8. Tóm tắt

> **Robot là một xe nâng tự lái: bám vạch đen để đi, dùng camera đọc màu để biết hàng
> thuộc nhà máy nào, dùng càng nâng để chở 2 kiện mỗi lượt và giao đúng chỗ — lặp lại
> cho đủ 12 kiện rồi làm thêm nhiệm vụ phụ, tất cả trong 4 phút và hoàn toàn tự động.**
