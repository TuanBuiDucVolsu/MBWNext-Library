# MBWNext Super App — cấu trúc & tóm tắt công việc (cho team Dev)

Tài liệu mô tả **cấu trúc** app Frappe `mbwnext_superapp` và **các hướng chỉnh đã làm** để SPA Vue chạy ổn trên site, mobile/desktop, và chuẩn hóa dev (lint, typecheck, auth store). Dùng khi onboard hoặc họp kỹ thuật.

---

## 1. Tổng quan

| Thành phần | Vai trò |
|------------|---------|
| **`mbwnext_superapp/`** (Python) | App Frappe: API `@frappe.whitelist`, trang Website `/superapp`, hooks route. |
| **`SUPERAPP/`** (Vue 3 + Vite + TS) | SPA PWA-style: build ra `mbwnext_superapp/public/superapp/`. |

URL production: **`/superapp/...`** (Vue Router `base` = `/superapp/`). API Frappe: **`/api/method/...`** trên cùng site (ưu tiên cùng origin để cookie session).

---

## 2. Cấu trúc thư mục (rút gọn)

```
apps/mbwnext_superapp/
├── .github/workflows/
│   └── superapp-frontend.yml    # CI: npm ci → lint → typecheck (khi đổi SUPERAPP/**)
├── .gitignore
├── pyproject.toml
├── README.md
│
├── SUPERAPP/                      # Frontend — Vue 3 + Vite + TypeScript + Pinia
│   ├── eslint.config.js
│   ├── tsconfig.json
│   ├── vite.config.js             # build → ../mbwnext_superapp/public/superapp
│   ├── tailwind.config.js
│   ├── postcss.config.js
│   ├── index.html
│   ├── public/
│   │   └── superapp.png           # logo / favicon (dev)
│   └── src/
│       ├── app/
│       │   ├── main.ts            # Pinia + router + hydrate auth store
│       │   └── App.vue
│       ├── core/
│       │   ├── router.ts          # history: prod /superapp/, dev /
│       │   └── layouts/
│       │       └── MobileShell.vue # full viewport mobile; mockup “máy” từ md↑
│       ├── stores/
│       │   └── auth.ts            # useAuthStore: authInfo + currentSiteUrl
│       ├── shared/
│       │   ├── api/
│       │   │   └── frappeUrl.ts   # URL RPC; relative /api/... khi cùng origin
│       │   └── auth/
│       │       └── auth.ts        # kiểu AuthInfo, helper login/profile
│       ├── styles/
│       │   └── tailwind.css
│       ├── features/
│       │   ├── auth/              # LoginSite, LoginAuth, ForgotPassword
│       │   ├── home/              # Home + subAppRoutes
│       │   ├── leadership/      # Executive Overview
│       │   ├── account/
│       │   ├── notifications/
│       │   ├── ess|stock|sell|delivery|crm/  # placeholder mini-app
│       │   └── _shared/
│       └── env.d.ts
│
└── mbwnext_superapp/              # Package Python Frappe
    ├── hooks.py                   # website_route_rules: /superapp → trang superapp
    ├── api/
    │   ├── superapp.py            # superapp_ping (guest), v.v.
    │   ├── leadership.py          # Leadership / Executive Summary
    │   ├── ess.py, stock.py, …    # ping / stub theo team
    │   └── __init__.py
    ├── www/
    │   ├── superapp.html          # extends web.html; block navbar/footer/breadcrumbs rỗng
    │   └── superapp.py            # get_context: inject CSS/JS từ index.html build
    └── public/
        └── superapp/              # Kết quả npm run build (index.html, assets/, superapp.png)
```

---

## 3. Backend Frappe — điểm chính

- **`hooks.py` — `website_route_rules`**: mọi đường dẫn `/superapp` và `/superapp/<path>` trỏ về page `superapp` (một shell HTML cho SPA).
- **`www/superapp.py` + `www/superapp.html`**: đọc `public/superapp/index.html` sau build, inject stylesheet + script `type="module"`; template **không** render navbar/footer/breadcrumb website (tránh thanh “Home” Frappe).
- **`api/superapp.py`**: `superapp_ping` (guest) để SPA kiểm tra site đã cài app.
- **`api/leadership.py`**: API Leadership (ví dụ `get_executive_summary`) — tách khỏi `superapp.py` theo domain.
- Các file **`ess.py`, `stock.py`, …**: thường là ping/stub cho mini-app tương ứng.

---

## 4. Frontend — routing & build

- **Vue Router**: production `createWebHistory("/superapp/")`, dev `createWebHistory(BASE_URL)`.
- **Build**: `npm run build` với `--base=/assets/mbwnext_superapp/superapp/` → asset path đúng khi Frappe serve qua `/assets/...`.
- **`MobileShell.vue`**: trên viewport hẹp full màn hình; từ breakpoint `md` trở lên giữ layout mockup desktop (viền “máy”) nếu cần demo.

---

## 5. Auth & session (Pinia)

- **`src/stores/auth.ts` — `useAuthStore`**:
  - `authInfo`: snapshot sau login (đồng bộ `localStorage` key `mbwnext_superapp_auth_info`).
  - `currentSiteUrl`: site ERPNext đã chọn (key `mbwnext_superapp_current_site`).
  - `hydrate()` gọi khi boot app; `setAuth` / `clearAuth` / `setCurrentSite`.
- Helper **`shared/auth/auth.ts`**: type `AuthInfo`, `loadAuthInfo()` (đọc storage — dùng trong hydrate), format chữ cái đại diện, v.v.
- **Danh sách site gần đây** vẫn có thể lưu riêng trong `LoginSite` (localStorage) — không bắt buộc nằm trong store.

---

## 6. Công cụ dev & CI

| Lệnh | Ý nghĩa |
|------|---------|
| `npm run dev` | Vite dev server (thường port 8080, `strictPort`). |
| `npm run build` | Bundle vào `mbwnext_superapp/public/superapp/`. |
| `npm run lint` | ESLint (flat config) trên `src`. |
| `npm run typecheck` | `vue-tsc --noEmit`. |

- **`.github/workflows/superapp-frontend.yml`**: khi thay đổi dưới `apps/mbwnext_superapp/SUPERAPP/**`, chạy `npm ci`, `lint`, `typecheck`.

---

## 7. Các việc đã thực hiện (tóm tắt theo chủ đề)

Dưới đây là **hướng xử lý đã áp dụng** trong codebase (để team nắm ngữ cảnh, không phải changelog từng commit):

### Website / Frappe shell

- Route **`/superapp/*`** → một trang SPA; **`superapp.html`** ghi đè block **`navbar` / `footer` / `breadcrumbs`** để không còn thanh “Home” mặc định của website.
- `get_context`: `no_header`, `full_width`, tắt sidebar trên layout web.

### UX mobile / desktop

- **`MobileShell`**: màn hình nhỏ = full viewport; màn rộng = khung mockup (nếu vẫn bật ở breakpoint `md+`).

### HTTPS / tunnel / site khác

- **`frappeUrl.ts`**: `isMixedContentBlocked` — trang HTTPS không cho gọi API `http://` (Mixed Content).
- **LoginSite**: sau khi bỏ copy hướng dẫn dài, logic kiểm tra mixed content vẫn có thể chạy khi bấm **Tiếp tục** (tùy phiên bản file).

### API Leadership

- Endpoint Leadership nằm trong **`api/leadership.py`** (không gom hết vào `superapp.py`); FE gọi method dotted path tương ứng (ví dụ `mbwnext_superapp.api.leadership.get_executive_summary` — kiểm tra đúng tên trong `Leadership.vue` và backend).

### Logo

- **`SUPERAPP/public/superapp.png`** và bản copy trong **`public/superapp/`** sau build; `index.html` + màn login trỏ `/superapp.png`.

### Tooling

- **TypeScript**: `tsconfig.json`, `env.d.ts`.
- **ESLint 9** + **vue-tsc** + script **`lint`** / **`typecheck`**.
- **Pinia store** auth như mục 5; sửa lỗi type trên một số màn (Account, Leadership, Notifications) khi bật `vue-tsc`.

### `.gitignore` (SUPERAPP)

- Bổ sung pattern file tạm Vite (`SUPERAPP/vite.config.*.timestamp-*`), `*.tsbuildinfo`.

---

## 8. Gợi ý khi họp team

- **Deploy**: sau `npm run build`, `bench build` / restart nếu cần; đảm bảo site serve đúng `/superapp` và assets.
- **Cùng origin**: mở SPA và API trên cùng hostname HTTPS để tránh CORS/cookie phức tạp.
- **Roadmap**: tách thêm composables/components khi số màn tăng; mở rộng API theo file trong `api/<domain>.py`.

---

*Tài liệu được tạo để thảo luận nội bộ; cập nhật khi cấu trúc hoặc convention thay đổi.*
