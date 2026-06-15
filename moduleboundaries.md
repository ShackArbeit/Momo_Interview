# momo 電商如何實踐 Monorepo / Multi-App、Module Boundaries 與 Shared Modules

## 為什麼 momo 需要這種架構？

momo 作為台灣最大的 B2C 電商之一，服務超過千萬會員，系統涵蓋：

- momo 購物網
- 手機 App
- 商城系統
- 供應商後台 (SCM)
- 行銷活動頁
- 廣告系統
- 會員系統
- 訂單系統
- 金流系統
- 客服系統

且必須面對：

- 雙11
- 618
- 黑色星期五
- 品牌日
- 限時搶購

等高流量、高併發場景。

因此如果所有前端都放在同一個 React 專案內，最後一定會變成：

- 難維護
- 難部署
- 難擴充
- Build 時間超長

所以大型電商通常會採用：

> Monorepo + Multi-App + Shared Modules

架構。這也是目前大型前端團隊常見做法。:contentReference[oaicite:0]{index=0}

---

# 一、什麼是 Monorepo？

Monorepo（Monolithic Repository）

意思：

整個公司的前端程式碼放在同一個 Git Repository 中管理。

---

## 傳統 Multi Repo

```text
Frontend-Web
Frontend-App
Frontend-SCM
Frontend-Admin
Frontend-Marketing

各自一個 Repo
```

問題：

- UI 重複開發
- 元件版本難同步
- 共用函式大量 Copy Paste
- CI/CD 維護成本高

---

## Monorepo

```text
momo-frontend/

├─ apps/
├─ packages/
├─ tools/
├─ docs/
└─ nx.json
```

所有前端程式碼統一管理。

---

# momo Monorepo 架構範例

```text
momo-frontend

├─ apps
│
├─ web-store
│
├─ mobile-web
│
├─ scm-center
│
├─ seller-center
│
├─ member-center
│
├─ campaign-platform
│
└─ ads-platform

├─ packages
│
├─ ui
├─ auth
├─ cart
├─ product
├─ analytics
├─ design-system
├─ utils
└─ api-sdk
```

---

# 二、什麼是 Multi-App？

Multi-App：

一個 Monorepo 裡面有很多獨立產品。

---

## momo 的實際情境

### App 1

購物首頁

```text
www.momoshop.com.tw
```

---

### App 2

會員中心

```text
member.momoshop.com.tw
```

---

### App 3

供應商中心

```text
scm.momoshop.com.tw
```

SCM 系統目前即為 momo 對外供應商平台之一。:contentReference[oaicite:1]{index=1}

---

### App 4

行銷活動系統

```text
event.momoshop.com.tw
```

---

### App 5

廣告平台

```text
ads.momoshop.com.tw
```

---

# Multi-App 的好處

## 獨立部署

雙11活動掛掉：

```text
campaign-platform
```

不影響：

```text
member-center
```

---

## 團隊可平行開發

```text
購物團隊
會員團隊
SCM團隊
廣告團隊
```

互不干擾。

---

# 三、什麼是 Module Boundaries？

這是面試非常愛問的題目。

---

Module Boundary

意思：

> 規定哪些模組可以依賴哪些模組。

避免系統變成義大利麵架構。

---

# 錯誤範例

```text
Cart

直接引用

Member
Order
Campaign
Search
```

久而久之：

```text
A -> B
B -> C
C -> A
```

形成循環依賴。

---

# momo 可能的 Boundary 設計

```text
apps
 │
 ▼

features
 │
 ▼

domain
 │
 ▼

shared
```

只能往下依賴。

---

# 架構圖

```text
                Apps
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼

    Store      SCM       Member

       │           │          │
       ▼           ▼          ▼

   Features    Features   Features

       │           │          │
       ▼           ▼          ▼

     Domain      Domain     Domain

       │           │          │
       └───────┬────┴─────┬───┘
               ▼
            Shared
```

---

# Domain Layer

例如：

```text
Product
Cart
Order
Member
Coupon
```

每個 Domain 獨立。

---

# Feature Layer

例如：

```text
AddToCart

Checkout

ProductReview

CouponApply
```

---

# Shared Layer

例如：

```text
Button
Modal
Table
Axios
Logger
```

所有 App 都能使用。

---

# 四、什麼是 Shared Modules？

Shared Modules

就是：

> 全公司共用的程式碼

---

# momo 典型 Shared Modules

## 1. Design System

```text
packages/design-system
```

包含：

```text
Button
Input
Modal
Drawer
Toast
Loading
```

---

使用方式

```tsx
import { Button } from "@momo/design-system";
```

---

# 2. API SDK

```text
packages/api-sdk
```

統一呼叫 API

```tsx
getProduct()
getOrder()
getMember()
```

---

# 3. Auth Module

```text
packages/auth
```

管理：

- JWT
- Refresh Token
- Login
- Logout

---

# 4. Analytics Module

```text
packages/analytics
```

統一送 GA

```tsx
trackClick()
trackPurchase()
trackImpression()
```

---

# 5. Cart Module

```text
packages/cart
```

共用：

```text
加入購物車
購物車數量
購物車同步
```

---

# 五、Nx Module Boundaries 實踐

很多大型 React Monorepo 會使用：

```text
Nx
```

管理 Boundary。

---

設定 Tag

```json
{
  "tags": ["scope:cart"]
}
```

---

限制規則

```text
cart

只能依賴

shared
```

---

禁止：

```text
cart -> order
order -> cart
```

循環引用。

---

# 六、momo 實際可能長這樣

```text
momo-frontend

├─ apps
│
├─ web-store
├─ mobile-web
├─ member-center
├─ scm-center
├─ campaign-platform
└─ ads-platform

├─ packages
│
├─ design-system
├─ auth
├─ analytics
├─ api-sdk
├─ product-domain
├─ cart-domain
├─ order-domain
├─ coupon-domain
└─ shared-utils
```

---

# 雙11時的運作

```text
Campaign App
       │
       ▼

Shared Analytics

       │
       ▼

Product Domain

       │
       ▼

Cart Domain
```

全部使用同一套：

- Button
- API SDK
- Login
- Tracking

但又能獨立部署。

---

# 面試總結（momo 版）

如果我是 momo 的前端工程師，我會這樣回答：

> momo 屬於大型電商平台，除了主購物網站外，還包含會員中心、供應商平台、活動平台與廣告平台等多個產品線，因此前端架構適合採用 Monorepo + Multi-App 模式。Monorepo 能集中管理程式碼與 CI/CD，而 Multi-App 則能讓不同產品獨立開發與部署。在此基礎上，再透過 Module Boundaries 控制依賴方向，避免跨 Domain 耦合；最後利用 Shared Modules 建立 Design System、Auth、API SDK、Analytics 等共用能力，降低重複開發成本並提升團隊協作效率。此架構特別適合 momo 在雙11、618 等高流量場景下快速迭代與穩定擴展。:contentReference[oaicite:2]{index=2}