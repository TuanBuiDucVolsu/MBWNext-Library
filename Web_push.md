# Web Push (VAPID) — MBWNext Super App

Tài liệu mô tả cách **thông báo đẩy Web Push** hoạt động trong app `mbwnext_superapp`: cấu hình, luồng dữ liệu và vai trò các hàm/API chính.

---

## 1. Tổng quan luồng

1. **Cấu hình site** có cặp khóa **VAPID** (public lộ cho trình duyệt, private chỉ trên server) và **contact** (`mailto:`) cho claim `sub` khi ký JWT gửi tới dịch vụ push (FCM, Mozilla, …).
2. **Người dùng** mở Super App (PWA), cấp quyền thông báo, **subscribe** `PushManager` với `applicationServerKey` = public key.
3. **Client** gọi API đăng ký → server lưu **endpoint + p256dh + auth** vào DocType **MBW Web Push Subscription** (mỗi thiết bị/trình duyệt một endpoint).
4. Khi có **Notification Log** mới (Desk), hook **`after_insert`** gọi gửi push tới user tương ứng; hoặc có thể gọi **`send_push_to_user`** từ code/Python.
5. **Service Worker** nhận sự kiện `push`, hiện notification; khi bấm, mở URL trong payload (đã xử lý origin/path cho tunnel/PWA).

---

## 2. Điều kiện triển khai

| Thành phần | Ghi chú |
|------------|---------|
| **HTTPS** | Web Push trên trình duyệt yêu cầu ngữ cảnh bảo mật (HTTPS hoặc `localhost`). |
| **Python** | `pywebpush`, `cryptography` — khai báo trong `mbwnext_superapp/requirements.txt`; cài vào env bench (`bench pip install` / rebuild). |
| **PWA / Service Worker** | Bundle Super App (`SUPERAPP`) build ra `public/superapp`, có `sw.js` (Workbox `injectManifest`). |
| **Frappe** | DocType **MBW Web Push Subscription** đã migrate; `hooks.py` đăng ký `doc_events` cho **Notification Log**. |

---

## 3. Cấu hình `site_config` (hoặc `common_site_config.json`)

Có thể đặt **chung cả bench** trong `sites/common_site_config.json` để mọi site kế thừa (ví dụ một bộ VAPID dùng chọn nhiều site).

| Khóa | Bắt buộc | Mô tả |
|------|-----------|--------|
| `mbwnext_web_push_vapid_public_key` | Có | Chuỗi **base64url** (không padding `=`), public key P-256 cho `PushManager.subscribe`. |
| `mbwnext_web_push_vapid_private_key` | Có | Private key dạng **PEM PKCS#8** (`-----BEGIN PRIVATE KEY-----`). Trong JSON có thể dùng `\n` thật hoặc chuỗi `\\n`. |
| `mbwnext_web_push_contact` | Có | Claim VAPID **`sub`**: thường `mailto:admin@company.com`. Đây là **liên hệ vận hành máy chủ push** (theo RFC), **không** phải “chỉ một user Frappe” được nhận tin. Mọi user vẫn subscribe và nhận push bình thường. |
| `mbwnext_web_push_click_origin` | Không | Khi test qua **cloudflared** / domain tạm: ví dụ `https://xxx.trycloudflare.com`. Dùng để **thay scheme+host** của URL bấm trong payload (giữ path/query/hash), tránh `get_url()` trả về LAN. Production có `host_name` đúng thì thường **không cần**. |

Sau khi sửa config: `bench clear-cache` (và restart nếu cần).

**Sinh key lần đầu (System Manager):** gọi method whitelist **`generate_mbwnext_vapid_keys`** (API `/api/method/...`), nhận public/private + gợi ý `mailto`, rồi **dán vào** `site_config` — API **không** tự ghi file.

**Dùng chung key nhiều site:** Một cặp public/private có thể dùng cho nhiều site miễn là mỗi site cấu hình **cùng** cặp và client subscribe đúng public key tương ứng. Đổi key (xoay) thì user cần **bật lại push** trên thiết bị.

---

## 4. DocType: MBW Web Push Subscription

Lưu subscription theo user:

| Trường | Ý nghĩa |
|--------|---------|
| `user` | User Frappe |
| `endpoint` | URL endpoint do trình duyệt/cửa hàng push cấp (**unique**) |
| `p256dh`, `auth` | Khóa mã hóa payload theo chuẩn Web Push |
| `user_agent`, `last_seen` | Thông tin phụ / cập nhật khi gửi thành công |

Endpoint **404/410** khi gửi → subscription bị **xóa** tự động (thiết bị hết hạn/ gỡ).

---

## 5. Hook Frappe (`hooks.py`)

```text
"Notification Log": {
    "after_insert": "mbwnext_superapp.api.web_push.send_push_for_notification_log",
}
```

Mỗi khi tạo **Notification Log** mới, server cố gắng gửi push tới `for_user` (nếu đã cấu hình VAPID và user có subscription).

---

## 6. Backend — module `mbwnext_superapp.api.web_push`

### 6.1 Hàm “nội bộ” / tiện ích

| Hàm | Công dụng |
|-----|-----------|
| `_conf(key)` | Đọc `frappe.conf` (site + common). |
| `_rewrite_push_click_url(url)` | Nếu có `mbwnext_web_push_click_origin`, đổi origin của URL mở khi bấm thông báo. |
| `_normalize_vapid_private_key_pem` / `_coerce_vapid_private_key_pkcs8_pem` | Chuẩn hóa PEM trong JSON (xuống dòng, một dòng, …) và ép PKCS#8 hợp lệ. |
| `_vapid_from_site_private_key` | Tải private key bằng **cryptography**, tạo `py_vapid.Vapid` — tránh lỗi truyền PEM thẳng vào `pywebpush` (`from_string` không parse PEM). Có **cache** theo nội dung PEM. |
| `is_web_push_configured()` | `True` khi đủ public + private + contact. |
| `_send_raw_push(subscription_info, payload, endpoint)` | Gọi `pywebpush.webpush` với payload JSON (`title`, `body`, `url`, `tag`). Xử lý lỗi 404/410. |
| `_remove_subscription_by_endpoint` | Xóa bản ghi subscription theo endpoint. |
| `_generate_vapid_keypair` | Sinh cặp key P-256 (public base64url, private PEM). |

### 6.2 API / điểm mở rộng cho client và nghiệp vụ

| Hàm | Loại | Công dụng |
|-----|------|-----------|
| `get_web_push_config` | `@frappe.whitelist` | Trả `enabled`, `public_key`, `my_subscription_count` (số endpoint đã lưu cho user hiện tại). Client gọi trước khi subscribe. |
| `register_web_push_subscription` | `@frappe.whitelist` | Lưu/cập nhật subscription; `doc.reload()` trước save để tránh `TimestampMismatchError`. |
| `unregister_web_push_subscription` | `@frappe.whitelist` | Xóa subscription theo `endpoint`. |
| `generate_mbwnext_vapid_keys` | `@frappe.whitelist` | System Manager: sinh key + contact mẫu để dán vào site config. |
| `send_push_to_user(for_user, *, title, body, url, tag)` | Python | Gửi push tới **mọi** subscription của user. `url` mặc định sau rewrite: `get_url("/superapp/")` nếu không truyền. Trả về **số** push gửi thành công. |
| `send_push_for_notification_log(doc, method)` | `doc_events` | Lấy tiêu đề từ Notification Log, URL mở: **`/superapp/notifications`** (danh sách Notifications trong Super App), gọi `send_push_to_user`. |

---

## 7. Frontend (Super App — tóm tắt)

| Thành phần | Vai trò |
|-------------|---------|
| `shared/api/webPushApi.ts` | RPC: `get_web_push_config`, `register_web_push_subscription`, `unregister_web_push_subscription`. |
| `shared/composables/useWebPush.ts` | `enable` / `disable`, `refreshHasActivePushSubscription` (dựa trên `pushManager.getSubscription`), timeout, đồng bộ subscription lên server. |
| `features/account/Account.vue` | UI Bật/Tắt push; hiển thị `my_subscription_count` khi thiết bị hiện tại chưa subscribe nhưng đã có thiết bị khác. |
| `src/sw.js` | `push`: `showNotification` với `data.url` đã qua `notificationTargetUrl` (đồng bộ path với origin PWA, tránh mở `/` gây redirect LAN). `notificationclick`: chỉ **focus/navigate** cửa sổ **cùng origin** với SW; không thì `clients.openWindow`. |
| `app/main.ts` | Đăng ký PWA service worker; sau hydrate có thể `syncSubscriptionIfNeeded`. |

**Lưu ý:** Mỗi **trình duyệt / PWA cài riêng** có subscription riêng; bật trên PWA không tự bật trên tab web khác.

---

## 8. Payload push (JSON)

Server gửi chuỗi JSON, ví dụ:

```json
{
  "title": "...",
  "body": "...",
  "url": "https://site.example.com/superapp/notifications",
  "tag": "..."
}
```

Service Worker đọc các trường này để hiển thị và mở URL khi bấm.

---

## 9. Gợi ý xử lý sự cố

- **Không gửi được / lỗi VAPID:** Kiểm tra PEM private PKCS#8, public base64url khớp cặp, `contact` có `mailto:`.
- **Tunnel vs LAN:** Dùng `mbwnext_web_push_click_origin` hoặc cập nhật `host_name`; deploy `sw.js` mới (rewrite URL + không reuse tab LAN).
- **404/410:** Subscription hết hạn — user bật lại push trên thiết bị.
- **TimestampMismatch khi đăng ký:** Đã xử lý bằng `reload()` trước `save` trong `register_web_push_subscription`.

---

## 10. File tham chiếu trong repo

- Backend: `mbwnext_superapp/api/web_push.py`
- Hook: `mbwnext_superapp/hooks.py` (`doc_events`)
- DocType: `mbwnext_super_app/doctype/mbw_web_push_subscription/`
- Phụ thuộc Python: `mbwnext_superapp/requirements.txt`
- PWA: `SUPERAPP/src/sw.js`, `SUPERAPP/src/shared/composables/useWebPush.ts`, `SUPERAPP/vite.config.js`
- Scope SW: `hooks.py` → `set_pwa_service_worker_scope` (header `Service-Worker-Allowed`)

---

*Tài liệu phản ánh hành vi code tại thời điểm viết; khi refactor API nên cập nhật bảng hàm và đường dẫn tương ứng.*
