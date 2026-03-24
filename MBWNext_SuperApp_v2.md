# MBWNext Super App — Kiến trúc & Lộ trình triển khai
> **Tài liệu kỹ thuật — Phiên bản 2.0 | Tháng 3/2026**
> Cập nhật: Frappe Cloud SaaS, ưu tiên Standalone PWA

---

## Mục lục

1. [Tổng quan dự án](#1-tổng-quan-dự-án)
2. [Bối cảnh hạ tầng — Frappe Cloud SaaS](#2-bối-cảnh-hạ-tầng--frappe-cloud-saas)
3. [Kiến trúc đề xuất — Standalone PWA ưu tiên](#3-kiến-trúc-đề-xuất--standalone-pwa-ưu-tiên)
4. [Cấu trúc thư mục chi tiết](#4-cấu-trúc-thư-mục-chi-tiết)
5. [Tech Stack](#5-tech-stack)
6. [App Registry — Phân phối app con theo gói](#6-app-registry--phân-phối-app-con-theo-gói)
7. [Auth Flow — Token Auth](#7-auth-flow--token-auth)
8. [CI/CD — Deploy trên Frappe Cloud](#8-cicd--deploy-trên-frappe-cloud)
9. [Push Notification](#9-push-notification)
10. [Lộ trình triển khai](#10-lộ-trình-triển-khai)
11. [Chi tiết từng App con](#11-chi-tiết-từng-app-con)
12. [Frappe Backend](#12-frappe-backend)
13. [Quy trình vận hành](#13-quy-trình-vận-hành)
14. [Ưu điểm & Nhược điểm](#14-ưu-điểm--nhược-điểm)

---

## 1. Tổng quan dự án

### 1.1 Bối cảnh

MBWNext vận hành hệ thống ERPNext/Frappe theo mô hình **SaaS — mỗi khách hàng là một Frappe site độc lập chạy trên Frappe Cloud**. Người dùng cần truy cập nghiệp vụ trên thiết bị di động, hoạt động được kể cả khi mất kết nối mạng.

**Giải pháp:** Xây dựng một **Super App PWA Standalone** — một ứng dụng triển khai tại 1 URL duy nhất (`app.mbwnext.com`), có khả năng kết nối đến bất kỳ site Frappe Cloud nào của khách hàng, chứa nhiều app con chuyên biệt phân phối theo gói đăng ký.

### 1.2 Mục tiêu

- **1 URL duy nhất** cho tất cả khách hàng: `app.mbwnext.com`
- Trải nghiệm di động native-like, installable (PWA)
- Hoạt động offline khi mất kết nối (IndexedDB + sync queue)
- Bán và phân phối từng app con theo gói đăng ký
- Deploy 1 lần, update tự động qua CI/CD — không phụ thuộc bench của từng site

### 1.3 Danh sách App con

| App con | Đối tượng | Nghiệp vụ chính | Ưu tiên |
|---------|-----------|-----------------|---------|
| **leadership** | CEO, Director, Manager | Phê duyệt chứng từ, xem KPI, báo cáo | **P1** |
| **delivery** | Nhân viên giao hàng | Xác nhận giao hàng, chụp ảnh, thu tiền | **P1** |
| **sell_dms** | Sales rep, DMS | Đặt đơn di động, quản lý khách hàng | P2 |
| **hrm** | Nhân viên, HR | Chấm công, nghỉ phép, nhân sự | P2 |
| **pos_crm** | Thu ngân, CRM | Bán hàng tại quầy, CRM | P3 |

---

## 2. Bối cảnh hạ tầng — Frappe Cloud SaaS

### 2.1 Mô hình triển khai

```
┌─────────────────────────────────────────────────────┐
│                  Frappe Cloud                        │
│                                                     │
│  site: client-a.frappe.cloud  (khách hàng A)        │
│  site: client-b.frappe.cloud  (khách hàng B)        │
│  site: client-c.frappe.cloud  (khách hàng C)        │
│  ...                                                │
│                                                     │
│  Mỗi site: ERPNext + mbwnext_superapp installed     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│         app.mbwnext.com (Standalone PWA)             │
│         Deploy: Vercel / Netlify / VPS riêng         │
│         1 URL duy nhất cho tất cả khách hàng        │
└─────────────────────────────────────────────────────┘
```

### 2.2 Tại sao Standalone > Frappe Native trên Frappe Cloud

| Vấn đề với Frappe Native trên Cloud | Standalone giải quyết như thế nào |
|------------------------------------|------------------------------------|
| Mỗi site có URL khác nhau (client-a.frappe.cloud/superapp) | 1 URL duy nhất: app.mbwnext.com |
| Không SSH được để chạy `bench build` | Deploy độc lập qua Vercel/Netlify |
| Update app = phải update từng site trên Frappe Cloud | Push code → CI/CD → tất cả user nhận ngay |
| Frappe Cloud giới hạn background service cho push | Dùng service push riêng (FCM/OneSignal) |
| User phải nhớ nhiều URL | 1 URL, nhập site khi đăng nhập |

### 2.3 Frappe Cloud — Những gì vẫn cần

Dù PWA chạy Standalone, **vẫn cần cài `mbwnext_superapp` lên mỗi site** trên Frappe Cloud để:
- Expose các Whitelisted API methods cho PWA gọi
- Quản lý `MBW App Subscription` DocType (khách mua gói nào)
- Cấu hình CORS cho domain `app.mbwnext.com`
- Xử lý business logic phía server

```bash
# Frappe Cloud Dashboard → Apps → Install
# Hoặc qua Frappe Cloud API khi onboard khách hàng mới
frappe-cloud install-app --site client-a.frappe.cloud --app mbwnext_superapp
```

---

## 3. Kiến trúc đề xuất — Standalone PWA ưu tiên

### 3.1 Tổng quan kiến trúc

```
┌─────────────────────────────────────────────────────────┐
│            app.mbwnext.com                               │
│            Vue 3 + Vite + PWA                            │
│            Deploy: Vercel/Netlify (CDN toàn cầu)        │
├──────────────────────────────────────────────────────────┤
│                    App Shell                             │
│   Login (nhập site URL) · App Registry · Router         │
│   Token Auth · Notification Hub · Offline DB            │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│leadership│ delivery │ sell_dms │   hrm    │  pos_crm    │
│  Vue app │  Vue app │  Vue app │  Vue app │   Vue app   │
│ (lazy)   │ (lazy)   │ (lazy)   │ (lazy)   │  (lazy)     │
├──────────┴──────────┴──────────┴──────────┴─────────────┤
│              Shared Layer                                │
│  useApi() · useOfflineDb() · useNotification()          │
│  Axios (token auth + interceptors) · Dexie.js           │
└──────────────────────┬───────────────────────────────────┘
                       │ HTTPS + Token Auth
        ┌──────────────┴──────────────┐
        ▼                             ▼
client-a.frappe.cloud        client-b.frappe.cloud
(ERPNext + mbwnext_superapp) (ERPNext + mbwnext_superapp)
```

### 3.2 Luồng hoạt động

```
1. User mở app.mbwnext.com
2. Nhập site URL: client-a.frappe.cloud
3. Nhập username / password
4. PWA gọi POST /api/method/login → nhận token
5. Gọi get_enabled_apps() → biết user được dùng app nào
6. Lazy load đúng app con theo subscription + role
7. User làm việc, dữ liệu cache offline nếu cần
8. Token tự động refresh khi gần hết hạn
```

### 3.3 Frappe Cloud CORS Configuration

Mỗi site khách hàng cần thêm vào `site_config.json` trên Frappe Cloud:

```json
{
  "allow_cors": "https://app.mbwnext.com",
  "cors_allow_credentials": true
}
```

Tự động hóa khi onboard bằng Frappe Cloud API:
```python
# Script onboarding tự động set CORS
frappe_cloud_api.update_site_config(
    site="client-a.frappe.cloud",
    config={"allow_cors": "https://app.mbwnext.com"}
)
```

---

## 4. Cấu trúc thư mục chi tiết

```
mbwnext-superapp/                    ← Git repository
│
├── .github/
│   └── workflows/
│       ├── build.yml                ← CI/CD: build Vue → deploy Vercel
│       └── release.yml              ← Release Frappe app lên Frappe Cloud
│
├── SuperApp/                        ← Vue 3 Standalone PWA
│   ├── src/
│   │   ├── shell/
│   │   │   ├── App.vue
│   │   │   ├── AppLayout.vue        ← Sidebar + Topbar
│   │   │   ├── LoginView.vue        ← Nhập site URL + username/password
│   │   │   ├── router/
│   │   │   │   └── index.js         ← Vue Router (dynamic addRoute)
│   │   │   └── stores/
│   │   │       ├── auth.js          ← Token auth + refresh
│   │   │       └── appRegistry.js   ← App Registry
│   │   │
│   │   ├── shared/
│   │   │   ├── api/
│   │   │   │   └── index.js         ← Axios + interceptors + token refresh
│   │   │   ├── components/          ← UI components chung (Frappe UI)
│   │   │   ├── composables/
│   │   │   │   ├── useApi.js
│   │   │   │   ├── useOfflineDb.js
│   │   │   │   └── usePush.js       ← Push notification
│   │   │   └── db/
│   │   │       └── index.js         ← Dexie.js schema
│   │   │
│   │   └── apps/
│   │       ├── leadership/          ← P1
│   │       │   ├── App.vue
│   │       │   ├── views/
│   │       │   │   ├── KPIDashboard.vue
│   │       │   │   ├── ApprovalList.vue
│   │       │   │   └── ApprovalDetail.vue
│   │       │   ├── store/leadership.js
│   │       │   └── api/index.js
│   │       │
│   │       ├── delivery/            ← P1
│   │       │   ├── App.vue
│   │       │   ├── views/
│   │       │   │   ├── DeliveryList.vue
│   │       │   │   ├── DeliveryDetail.vue
│   │       │   │   ├── ConfirmDelivery.vue
│   │       │   │   └── CollectPayment.vue
│   │       │   ├── store/delivery.js
│   │       │   └── api/index.js
│   │       │
│   │       ├── sell_dms/            ← P2
│   │       ├── hrm/                 ← P2
│   │       └── pos_crm/             ← P3
│   │
│   ├── public/
│   │   ├── manifest.json
│   │   └── icons/
│   ├── index.html
│   ├── vite.config.js
│   ├── tailwind.config.js
│   └── package.json
│
└── mbwnext_superapp/                ← Frappe app (cài lên từng site)
    ├── api/
    │   ├── app_registry.py          ← get_enabled_apps()
    │   ├── leadership.py
    │   ├── delivery.py
    │   ├── sell_dms.py
    │   └── hrm.py
    ├── doctype/
    │   └── mbw_app_subscription/
    ├── hooks.py
    └── __init__.py
```

---

## 5. Tech Stack

| Thành phần | Công nghệ | Phiên bản | Lý do |
|-----------|-----------|-----------|-------|
| Frontend Framework | Vue.js 3 | 3.4+ | Frappe ecosystem, POSNext pattern |
| Build Tool | Vite | 5.0+ | Nhanh, HMR, env flags |
| PWA | vite-plugin-pwa | 0.19+ | Service Worker, offline cache tự động |
| State | Pinia + persist | 2.1+ | Token lưu localStorage |
| UI | Frappe UI | Latest | Consistent với ERPNext |
| Offline DB | Dexie.js | 3.x | IndexedDB wrapper |
| HTTP Client | Axios | 1.6+ | Token interceptor, retry queue |
| Router | Vue Router | 4.x | Dynamic addRoute per app |
| CSS | Tailwind CSS | 3.4+ | Utility-first |
| Deploy PWA | Vercel / Netlify | — | CDN, HTTPS tự động, free tier |
| Backend | Frappe v15+ | — | Python API, Whitelisted Methods |
| Push | Firebase FCM / OneSignal | — | Không phụ thuộc Frappe Cloud infra |
| CI/CD | GitHub Actions | — | Build → Deploy tự động |

---

## 6. App Registry — Phân phối app con theo gói

### 6.1 DocType: MBW App Subscription

```json
{
  "doctype": "MBW App Subscription",
  "fields": [
    {"fieldname": "app_id",      "fieldtype": "Select",
     "options": "leadership\ndelivery\nsell_dms\nhrm\npos_crm"},
    {"fieldname": "status",      "fieldtype": "Select",
     "options": "Active\nExpired\nTrial"},
    {"fieldname": "valid_until", "fieldtype": "Date"},
    {"fieldname": "notes",       "fieldtype": "Text"}
  ]
}
```

> **Lưu ý Frappe Cloud:** Không còn field `site` vì mỗi site đã là 1 database riêng biệt — subscription chỉ cần lưu app_id + status.

### 6.2 Whitelisted Method

```python
# mbwnext_superapp/api/app_registry.py
import frappe

@frappe.whitelist()
def get_enabled_apps():
    """
    Trả về list app con: subscription Active + user có role phù hợp.
    PWA gọi 1 lần sau khi login để lazy load đúng app.
    """
    user = frappe.session.user
    roles = frappe.get_roles(user)

    active_apps = frappe.get_all(
        "MBW App Subscription",
        filters={"status": ["in", ["Active", "Trial"]]},
        fields=["app_id", "valid_until"]
    )

    # Lọc Trial còn hạn
    import datetime
    today = datetime.date.today()
    purchased = set()
    for a in active_apps:
        if a.status == "Active":
            purchased.add(a.app_id)
        elif a.status == "Trial" and (not a.valid_until or a.valid_until >= today):
            purchased.add(a.app_id)

    ROLE_MAP = {
        "leadership": ["Management", "System Manager", "CEO", "Director"],
        "delivery":   ["Delivery User"],
        "sell_dms":   ["Sales User", "Sales Manager"],
        "hrm":        ["HR User", "HR Manager"],
        "pos_crm":    ["POS User", "CRM User"],
    }

    return [
        app_id for app_id, required_roles in ROLE_MAP.items()
        if app_id in purchased and any(r in roles for r in required_roles)
    ]
```

### 6.3 Vue App Registry Store

```javascript
// src/shell/stores/appRegistry.js
import { defineStore } from 'pinia'
import { useApi } from '@/shared/api'
import router from '@/shell/router'

export const useAppRegistryStore = defineStore('appRegistry', {
  state: () => ({ enabledApps: [], loaded: false }),

  actions: {
    async loadApps() {
      const api = useApi()
      const res = await api.get(
        '/api/method/mbwnext_superapp.api.app_registry.get_enabled_apps'
      )
      this.enabledApps = res.data.message

      // Lazy load và đăng ký dynamic routes
      for (const appId of this.enabledApps) {
        const mod = await import(`@/apps/${appId}/App.vue`)
        router.addRoute('shell', {
          path: `/${appId}`,
          name: appId,
          component: mod.default,
          meta: { appId },
        })
      }
      this.loaded = true
    },
  },
})
```

---

## 7. Auth Flow — Token Auth

### 7.1 Luồng tổng quan

```
App mở → Có token trong localStorage?
  ├── Có → GET /api/method/ping
  │         ├── 200 OK → Bootstrap → App sẵn sàng
  │         └── 401 → Silent refresh
  │                    ├── OK → Bootstrap → App sẵn sàng
  │                    └── Fail → Về màn login
  │
  └── Không → Màn login (nhập site URL + user/pass)
               ├── Validate site: GET /api/method/ping
               ├── POST /api/method/login → token + refresh_token
               ├── Lưu vào Pinia + localStorage
               └── Bootstrap (Promise.all):
                   ├── get user roles
                   ├── get company/branch
                   ├── get POS profile / territory
                   ├── get custom permissions
                   └── get_enabled_apps() → lazy load apps
```

### 7.2 Axios — Token Auth + Interceptors

```javascript
// src/shared/api/index.js
import axios from 'axios'
import { useAuthStore } from '@/shell/stores/auth'

let isRefreshing = false
let failedQueue = []

const processQueue = (error, token = null) => {
  failedQueue.forEach(({ resolve, reject }) =>
    error ? reject(error) : resolve(token)
  )
  failedQueue = []
}

export function useApi() {
  const auth = useAuthStore()

  const instance = axios.create({ timeout: 15000 })

  // Request: inject baseURL + token
  instance.interceptors.request.use(config => {
    config.baseURL = auth.siteUrl
    if (auth.token) {
      config.headers.Authorization = `token ${auth.token}`
    }
    return config
  })

  // Response: 401 → silent refresh → retry
  instance.interceptors.response.use(
    res => res,
    async err => {
      const original = err.config
      if (err.response?.status !== 401 || original._retry) {
        return Promise.reject(err)
      }

      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject })
        }).then(token => {
          original.headers.Authorization = `token ${token}`
          return instance(original)
        })
      }

      original._retry = true
      isRefreshing = true

      const ok = await auth.silentRefresh()
      isRefreshing = false

      if (ok) {
        processQueue(null, auth.token)
        original.headers.Authorization = `token ${auth.token}`
        return instance(original)
      }

      processQueue(new Error('Refresh failed'))
      auth.logout()
      return Promise.reject(err)
    }
  )

  return instance
}
```

### 7.3 Auth Store

```javascript
// src/shell/stores/auth.js
import { defineStore } from 'pinia'

export const useAuthStore = defineStore('auth', {
  state: () => ({
    siteUrl:       '',
    token:         '',
    refreshToken:  '',
    tokenExpiry:   null,
    currentUser:   '',
    fullName:      '',
    userRoles:     [],
    userInfo:      {},
    posProfile:    null,
    territory:     null,
    customPerms:   [],
    siteHistory:   [],
  }),

  persist: {
    key: 'mbwnext_v2',
    paths: ['siteUrl','token','refreshToken','tokenExpiry',
            'currentUser','fullName','siteHistory'],
  },

  getters: {
    isLoggedIn:      s => !!s.token && !!s.siteUrl,
    isTokenStale:    s => s.tokenExpiry && Date.now() > s.tokenExpiry - 5*60*1000,
    tokenMinutesLeft:s => Math.max(0, Math.floor((s.tokenExpiry - Date.now()) / 60000)),
    hasRole:         s => role => s.userRoles.includes(role),
  },

  actions: {
    async login(siteUrl, username, password) {
      const url = siteUrl.replace(/\/$/, '')

      // 1. Validate site
      await fetch(`${url}/api/method/ping`)
        .catch(() => { throw new Error('Site không tồn tại') })

      // 2. Authenticate
      const res = await fetch(`${url}/api/method/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ usr: username, pwd: password }),
      })
      if (!res.ok) throw new Error('Sai tài khoản hoặc mật khẩu')

      const data = await res.json()
      this.siteUrl      = url
      this.currentUser  = username
      this.token        = data.message.token
      this.refreshToken = data.message.refresh_token
      this.tokenExpiry  = Date.now() + data.message.expires_in * 1000

      // Lưu site history
      this.siteHistory = [url, ...this.siteHistory.filter(s => s !== url)].slice(0, 5)

      await this.bootstrap()
      this.startTokenWatcher()
    },

    async bootstrap() {
      const api = useApi()
      const [roles, info, profile, perms] = await Promise.all([
        api.get('/api/method/frappe.client.get_list', {
          params: { doctype: 'Has Role', filters: { parent: this.currentUser }, fields: ['role'] }
        }),
        api.get('/api/method/frappe.client.get', {
          params: { doctype: 'User', name: this.currentUser, fields: ['full_name','company','branch'] }
        }),
        api.get('/api/method/mbwnext_superapp.api.app_registry.get_user_profile'),
        api.get('/api/method/mbwnext_superapp.api.app_registry.get_custom_permissions'),
      ])

      this.userRoles  = roles.data.message.map(r => r.role)
      this.fullName   = info.data.message.full_name
      this.userInfo   = { company: info.data.message.company, branch: info.data.message.branch }
      this.posProfile = profile.data.message?.pos_profile
      this.territory  = profile.data.message?.territory
      this.customPerms = perms.data.message
    },

    async silentRefresh() {
      try {
        const res = await fetch(
          `${this.siteUrl}/api/method/frappe.integrations.oauth2.get_token`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              grant_type: 'refresh_token',
              refresh_token: this.refreshToken,
            }),
          }
        )
        if (!res.ok) return false
        const data = await res.json()
        this.token        = data.access_token
        this.refreshToken = data.refresh_token
        this.tokenExpiry  = Date.now() + data.expires_in * 1000
        return true
      } catch { return false }
    },

    startTokenWatcher() {
      setInterval(async () => {
        if (this.isLoggedIn && this.isTokenStale) await this.silentRefresh()
      }, 60 * 1000)
    },

    logout() {
      this.$reset()
      router.replace('/login')
    },
  },
})
```

---

## 8. CI/CD — Deploy trên Frappe Cloud

### 8.1 Vấn đề với Frappe Cloud

Frappe Cloud **không cho phép SSH vào chạy `bench build`**. Giải pháp: build Vue trong CI/CD trước khi push, commit file đã build vào repo.

### 8.2 GitHub Actions — Build & Deploy PWA

```yaml
# .github/workflows/build.yml
name: Build & Deploy PWA

on:
  push:
    branches: [main]
    paths: ['SuperApp/**']

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: SuperApp/package-lock.json

      - name: Install dependencies
        run: cd SuperApp && npm ci

      - name: Build PWA
        run: cd SuperApp && npm run build
        env:
          VITE_APP_NAME: MBWNext Super App

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: SuperApp/dist
          vercel-args: '--prod'
```

### 8.3 GitHub Actions — Update Frappe App trên Frappe Cloud

```yaml
# .github/workflows/release.yml
name: Release Frappe App

on:
  push:
    branches: [main]
    paths: ['mbwnext_superapp/**']

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Notify Frappe Cloud to update app
        run: |
          curl -X POST \
            -H "Authorization: token ${{ secrets.FC_TOKEN }}" \
            https://frappecloud.com/api/method/press.api.site.update_app \
            -d '{"app": "mbwnext_superapp"}'
```

### 8.4 vite.config.js

```javascript
// SuperApp/vite.config.js
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { VitePWA } from 'vite-plugin-pwa'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [
    vue(),
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'MBWNext Super App',
        short_name: 'MBWNext',
        display: 'standalone',        // BẮT BUỘC để iOS cho push notification
        theme_color: '#2E75B6',
        background_color: '#ffffff',
        start_url: '/',
        icons: [
          { src: '/icons/icon-192.png', sizes: '192x192', type: 'image/png' },
          { src: '/icons/icon-512.png', sizes: '512x512', type: 'image/png' },
          { src: '/icons/icon-512-maskable.png', sizes: '512x512',
            type: 'image/png', purpose: 'maskable' },
        ],
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /\/api\//,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'frappe-api',
              expiration: { maxEntries: 100, maxAgeSeconds: 300 },
              networkTimeoutSeconds: 10,
            },
          },
        ],
      },
    }),
  ],

  resolve: {
    alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) },
  },

  build: {
    outDir: 'dist',
    sourcemap: false,
  },
})
```

---

## 9. Push Notification

### 9.1 Chiến lược: Firebase FCM (không phụ thuộc Frappe Cloud infra)

Vì Frappe Cloud giới hạn background services, dùng **Firebase Cloud Messaging (FCM)** để gửi push — miễn phí, đáng tin cậy, hỗ trợ cả Android và iOS PWA.

```
Frappe (sự kiện xảy ra)
    → gọi Firebase Admin SDK
        → FCM Server
            → Service Worker trên thiết bị user
                → Notification hiển thị
```

### 9.2 Hỗ trợ theo nền tảng

| Nền tảng | Push | Điều kiện |
|---------|------|----------|
| Android (Chrome) | ✅ Đầy đủ | Chỉ cần grant permission |
| iOS 16.4+ | ✅ Hoạt động | **Phải cài PWA lên Home Screen** |
| iOS < 16.4 | ❌ Không hỗ trợ | — |
| Desktop Chrome/Edge | ✅ Đầy đủ | Chỉ cần grant permission |

> **Onboarding iOS:** Hướng dẫn user Safari → Share → Add to Home Screen ngay khi đăng nhập lần đầu.

### 9.3 Service Worker — Nhận push

```javascript
// public/sw-custom.js
self.addEventListener('push', event => {
  const data = event.data?.json() ?? {}
  event.waitUntil(
    self.registration.showNotification(data.title ?? 'MBWNext', {
      body: data.body,
      icon: '/icons/icon-192.png',
      badge: '/icons/badge-72.png',
      data: { url: data.url ?? '/' },
      actions: data.actions ?? [],
    })
  )
})

self.addEventListener('notificationclick', event => {
  event.notification.close()
  const url = event.notification.data?.url ?? '/'
  event.waitUntil(
    clients.matchAll({ type: 'window' }).then(windowClients => {
      const existing = windowClients.find(c => c.url === url)
      if (existing) return existing.focus()
      return clients.openWindow(url)
    })
  )
})
```

### 9.4 Frappe — Gửi push khi có sự kiện

```python
# mbwnext_superapp/notifications.py
import frappe
import firebase_admin
from firebase_admin import messaging

def send_push(user, title, body, url='/'):
    """Gửi push notification tới user qua FCM."""
    subscription = frappe.db.get_value(
        "MBW Push Subscription", {"user": user, "active": 1}, "fcm_token"
    )
    if not subscription:
        return

    message = messaging.Message(
        notification=messaging.Notification(title=title, body=body),
        data={"url": url},
        token=subscription,
    )
    messaging.send(message)

# Hook: gửi notification khi Sales Order cần phê duyệt
def on_sales_order_submit(doc, method):
    approvers = get_approvers_for(doc.company)
    for user in approvers:
        send_push(
            user=user,
            title="Đơn hàng cần phê duyệt",
            body=f"SO {doc.name} - {doc.customer} - {doc.grand_total:,.0f} VNĐ",
            url=f"/leadership/approval/{doc.name}"
        )
```

---

## 10. Lộ trình triển khai

### 10.1 Các giai đoạn

| Giai đoạn | Nội dung | Thời gian | Deliverable |
|-----------|---------|-----------|-------------|
| **Giai đoạn 0** | Scaffold + CI/CD + App Registry | 1–2 tuần | `app.mbwnext.com` chạy được, token auth |
| **Giai đoạn 1** | App Giao hàng (P1) | 3–4 tuần | App giao hàng production + offline |
| **Giai đoạn 2** | App Lãnh đạo (P1) + Push | 3–4 tuần | App phê duyệt + KPI + FCM push |
| **Giai đoạn 3** | App Sell/DMS (P2) | 4–5 tuần | App đặt đơn + DMS |
| **Giai đoạn 4** | App HRM (P2) | 3–4 tuần | Chấm công + nghỉ phép |
| **Giai đoạn 5** | Polish + QA + Go-live | 2–3 tuần | UAT, iOS install guide, go-live |

### 10.2 Chi tiết Giai đoạn 0 — Scaffold

**Checklist:**
- [ ] Tạo repo `mbwnext-superapp` trên GitHub
- [ ] Scaffold Vue 3 + Vite + PWA trong folder `SuperApp/`
- [ ] Pinia Auth Store với token auth flow
- [ ] App Registry Store với dynamic route
- [ ] Login screen: site URL dropdown + user/pass
- [ ] CI/CD GitHub Actions → deploy Vercel
- [ ] Frappe app `mbwnext_superapp` với `MBW App Subscription` DocType
- [ ] `get_enabled_apps()` whitelisted method
- [ ] CORS config cho `app.mbwnext.com`
- [ ] Test end-to-end với 1 staging site trên Frappe Cloud

---

## 11. Chi tiết từng App con

### 11.1 App Giao hàng — P1

| Màn hình | Chức năng | DocType | Offline |
|---------|-----------|---------|---------|
| Danh sách giao hàng | Chứng từ cần giao hôm nay | Delivery Note | Cache buổi sáng |
| Chi tiết | Địa chỉ, items, số tiền | Delivery Note | Cache cùng list |
| Xác nhận | Check items, chụp ảnh, ký | Delivery Note | Queue offline |
| Thu tiền | Nhập tiền, chọn PT | Payment Entry | Queue offline |
| Lịch sử | Chứng từ đã xử lý | Delivery Note | Cache 7 ngày |

**Offline strategy:**
```javascript
// Sáng: cache danh sách
await offlineDb.deliveries.bulkPut(todayDeliveries)

// Offline: đẩy vào queue
await offlineDb.syncQueue.add({
  action: 'confirm_delivery',
  payload: { name: 'DN-00001', status: 'Delivered', photo: base64 },
  createdAt: Date.now(),
})

// Có mạng: flush queue
await syncService.flush()
```

### 11.2 App Lãnh đạo — P1

| Màn hình | Chức năng | DocType | Offline |
|---------|-----------|---------|---------|
| KPI Dashboard | Doanh thu, đơn hàng, công nợ | Sales Invoice, SO | Cache 30 phút |
| Chờ phê duyệt | Danh sách cần duyệt | SO, PO, Leave, Expense | Không |
| Chi tiết phê duyệt | Xem, comment, Approve/Reject | Tương ứng | Không |
| Báo cáo nhanh | Top KH, SP, NV | Custom | Cache 1 giờ |

---

## 12. Frappe Backend

### 12.1 CORS — BẮT BUỘC cho Standalone PWA

```json
// site_config.json (set qua Frappe Cloud dashboard)
{
  "allow_cors": "https://app.mbwnext.com",
  "cors_allow_credentials": true
}
```

### 12.2 hooks.py

```python
# mbwnext_superapp/hooks.py
app_name = "mbwnext_superapp"
app_title = "MBWNext Super App"
app_version = "2.0.0"

# Không cần website_route_rules vì PWA là Standalone
# Chỉ cần expose API

doc_events = {
    "Sales Order": {
        "on_submit": "mbwnext_superapp.notifications.on_sales_order_submit"
    },
    "Leave Application": {
        "on_submit": "mbwnext_superapp.notifications.on_leave_submitted"
    },
    "Purchase Order": {
        "on_submit": "mbwnext_superapp.notifications.on_purchase_order_submit"
    },
}
```

### 12.3 API Delivery

```python
# mbwnext_superapp/api/delivery.py
import frappe

@frappe.whitelist()
def get_today_deliveries():
    """Lấy danh sách Delivery Note cần giao hôm nay."""
    return frappe.get_all("Delivery Note", filters={
        "docstatus": 1,
        "status": ["in", ["To Deliver", "Partially Delivered"]],
        "posting_date": frappe.utils.today(),
    }, fields=["name","customer","grand_total","shipping_address_name"])

@frappe.whitelist()
def confirm_delivery(name, status, photo=None, signature=None):
    """Xác nhận giao hàng, lưu ảnh và chữ ký."""
    doc = frappe.get_doc("Delivery Note", name)
    doc.status = status
    if photo:
        doc.custom_delivery_photo = photo
    if signature:
        doc.custom_customer_signature = signature
    doc.save()
    return {"success": True}

@frappe.whitelist()
def collect_payment(delivery_note, amount, mode_of_payment):
    """Tạo Payment Entry từ Delivery Note."""
    pe = frappe.new_doc("Payment Entry")
    pe.payment_type = "Receive"
    pe.mode_of_payment = mode_of_payment
    pe.paid_amount = float(amount)
    pe.references = [{"reference_doctype": "Delivery Note", "reference_name": delivery_note}]
    pe.insert()
    pe.submit()
    return {"payment_entry": pe.name}
```

---

## 13. Quy trình vận hành

### 13.1 Onboarding khách hàng mới

```bash
# 1. Frappe Cloud Dashboard → Install App
#    Site: client-a.frappe.cloud → Install: mbwnext_superapp

# 2. Set CORS (Frappe Cloud Site Settings)
#    allow_cors: https://app.mbwnext.com

# 3. Trong Frappe Desk → MBW App Subscription
#    Tạo records cho các app đã mua:
#    app_id: delivery,    status: Active
#    app_id: leadership,  status: Trial, valid_until: 30/04/2026

# 4. Assign roles cho user:
#    Nhân viên giao hàng → Delivery User
#    Manager → Management

# 5. Gửi link cho user: https://app.mbwnext.com
#    Hướng dẫn nhập site URL: client-a.frappe.cloud
```

### 13.2 Update PWA

```bash
# Dev push code lên main branch
git push origin main

# GitHub Actions tự động:
# 1. Build Vue → dist/
# 2. Deploy lên Vercel
# 3. Service Worker tự notify user cập nhật
# Không cần làm gì thêm
```

### 13.3 Update Frappe App

```bash
# Push changes vào repo
git push origin main

# GitHub Actions notify Frappe Cloud update app
# Frappe Cloud pull latest code → áp dụng cho tất cả site
# bench migrate tự động nếu có schema change
```

### 13.4 Bật/tắt app con

Vào Frappe Desk → MBW App Subscription → đổi `status`:
- Kích hoạt: `Active`
- Hủy: `Expired`
- Dùng thử: `Trial` + set `valid_until`

Lần tiếp theo user mở app, Shell tự gọi lại `get_enabled_apps()`.

---

## 14. Ưu điểm & Nhược điểm

### ✅ Ưu điểm

| # | Ưu điểm | Tại sao quan trọng với Frappe Cloud SaaS |
|---|---------|----------------------------------------|
| 1 | **1 URL duy nhất** `app.mbwnext.com` | User không cần nhớ URL của từng site |
| 2 | **Deploy độc lập** với Frappe Cloud | Không bị giới hạn bởi bench/infra của Cloud |
| 3 | **Update instant** qua CI/CD | Push code → tất cả user nhận ngay, không cần update từng site |
| 4 | **Token auth** rõ ràng | Kiểm soát session tốt hơn, không phụ thuộc cookie |
| 5 | **Offline support** | Nhân viên giao hàng làm việc vùng sóng yếu |
| 6 | **PWA installable** | Android + iOS (16.4+), push notification |
| 7 | **App Registry linh hoạt** | Bật/tắt app con không cần redeploy |
| 8 | **Push via FCM** | Không phụ thuộc Frappe Cloud background service |
| 9 | **Lazy loading** | Chỉ download code app con user có quyền |
| 10 | **Multi-site** | 1 PWA kết nối nhiều site, switch dễ dàng |

### ⚠️ Nhược điểm & Cách xử lý

| # | Nhược điểm | Cách xử lý |
|---|-----------|-----------|
| 1 | Cần cấu hình CORS trên mỗi site | Script tự động hóa khi onboard |
| 2 | Vẫn cần cài Frappe app per site | Frappe Cloud Marketplace + auto-install script |
| 3 | iOS push cần cài PWA lên Home Screen | Hướng dẫn rõ trong onboarding flow |
| 4 | Token auth phức tạp hơn session | Axios interceptor + silent refresh xử lý tự động |
| 5 | CORS restrict một số Frappe API | Cần whitelist đầy đủ, test kỹ cross-origin |

---

## Ghi chú cho Claude Code

Khi implement dự án này:

1. **Môi trường target là Frappe Cloud** — không có SSH, không có `bench build` trực tiếp
2. **Standalone mode là chính** — không có Frappe native mode, chỉ có token auth
3. **CORS phải cấu hình đầy đủ** — test ngay từ đầu với staging site thật trên Frappe Cloud
4. **CI/CD từ ngày 1** — build Vue qua GitHub Actions, deploy Vercel, không commit thủ công
5. **Push dùng FCM** — không dùng VAPID trực tiếp vì Frappe Cloud hạn chế background service
6. **Giai đoạn 0 trước tiên** — Shell + App Registry + token auth + CORS phải xong trước khi build app con
7. **Test offline từ sớm** — đừng để sau mới add Dexie.js, sẽ phải refactor lớn
8. **Mỗi app con hoàn toàn độc lập** — chỉ import từ `@/shared/`, không import cross-app
