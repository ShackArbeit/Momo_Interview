# momo 大型電商的 Monorepo、Multi-App、Module Boundaries 與 Shared Modules 設計

---

## 一、先理解整體概念

以 momo 這類大型電商來說，網站並不是只有一個簡單的前端專案。

它可能同時包含：

- momo 前台購物網站
- 行動版網站
- 會員中心
- 購物車與結帳系統
- 活動與促銷頁面
- 廠商後台
- 內部營運後台
- 客服系統
- Design System
- API SDK
- 共用商業邏輯
- ESLint、TypeScript 等工程設定

因此，比較合理的做法不是把所有功能都塞進一個巨大 Next.js 專案，而是採用：

> **Monorepo 管理程式碼 + Multi-App 拆分應用程式 + Module Boundaries 控制依賴 + Shared Modules 重用共用能力**

整體關係如下：

```text
┌─────────────────────────────────────────────────────────────┐
│                    momo Frontend Monorepo                    │
│                                                             │
│  ┌────────────────────── Apps ────────────────────────────┐  │
│  │                                                       │  │
│  │  商品站   搜尋站   活動站   會員站   結帳站   管理後台 │  │
│  │                                                       │  │
│  └─────────────────────────┬─────────────────────────────┘  │
│                            │                                │
│                   透過公開 API 使用                         │
│                            ▼                                │
│  ┌────────────────── Shared Packages ────────────────────┐  │
│  │                                                       │  │
│  │  UI / Auth / Analytics / API Client / Config / Utils │  │
│  │                                                       │  │
│  └─────────────────────────┬─────────────────────────────┘  │
│                            │                                │
│                 Module Boundaries 限制依賴                  │
│                            │                                │
│              Turborepo + pnpm 管理建置與套件                │
└─────────────────────────────────────────────────────────────┘
```

---

# 二、什麼是 Monorepo？

## 2.1 定義

Monorepo 是指：

> 將多個彼此相關的應用程式、套件與工程設定，集中放在同一個 Git Repository 中管理。

Monorepo 不代表所有程式都必須一起部署，也不代表系統是一個 Monolith。

一個 Monorepo 裡面的不同 App，仍然可以：

- 獨立開發
- 獨立測試
- 獨立建置
- 獨立部署
- 使用不同發布週期
- 由不同團隊維護

---

## 2.2 momo 的 Monorepo 目錄範例

```text
momo-frontend/
├── apps/
│   ├── storefront/             # 商品首頁、分類頁、商品詳情頁
│   ├── search/                 # 搜尋結果、篩選、排序
│   ├── campaign/               # 雙 11、週年慶、品牌活動
│   ├── member/                 # 登入、會員資料、訂單查詢
│   ├── checkout/               # 購物車、結帳、付款
│   ├── vendor-console/         # 廠商後台
│   └── admin-console/          # momo 內部營運後台
│
├── packages/
│   ├── ui/                     # Design System 與基礎 UI
│   ├── layout/                 # Header、Footer、Navigation
│   ├── auth/                   # 登入狀態與權限處理
│   ├── analytics/              # GA、廣告與行為追蹤
│   ├── api-client/             # API Client 與 Request Wrapper
│   ├── product-domain/         # 商品領域共用模型與邏輯
│   ├── cart-domain/            # 購物車領域共用模型與邏輯
│   ├── promotion-domain/       # 優惠與促銷邏輯
│   ├── feature-flags/          # Feature Toggle
│   ├── i18n/                   # 多語系
│   ├── observability/          # Logging、Tracing、Monitoring
│   ├── eslint-config/          # ESLint 共用設定
│   ├── typescript-config/      # TypeScript 共用設定
│   └── test-utils/             # 測試工具
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.json
```

---

## 2.3 Monorepo 對 momo 的價值

### 1. 共用程式碼容易管理

例如商品卡片 `ProductCard` 可能會出現在：

- 首頁推薦區
- 商品搜尋結果
- 分類頁
- 活動頁
- 會員收藏頁
- 購物車推薦區

若每個 App 都自己實作一份，容易產生：

- 樣式不一致
- 價格顯示規則不同
- 優惠標籤不同
- 無障礙規範不一致
- 修正 Bug 時需要修改多個專案

Monorepo 可以將共用元件集中到：

```text
packages/ui
packages/product-domain
```

各 App 再透過 package 使用。

```tsx
import { ProductCard } from "@momo/ui";
import { formatProductPrice } from "@momo/product-domain";
```

---

### 2. 跨 App 修改可以放在同一個 Pull Request

假設商品 API 增加：

```ts
promotionLabels: string[];
```

工程師可以在同一個 PR 中同步修改：

```text
packages/api-client
packages/product-domain
packages/ui
apps/storefront
apps/search
apps/campaign
```

如此可降低 API 型別已更新，但某些 App 尚未同步的問題。

---

### 3. 統一工程規範

所有 App 可以共用：

```text
@momo/eslint-config
@momo/typescript-config
@momo/test-config
@momo/observability
```

讓不同團隊遵守相同的：

- TypeScript 規則
- ESLint 規則
- 測試標準
- Commit 規範
- Logging 格式
- 錯誤處理方式
- 效能監控方式

---

### 4. 只建置受到影響的 App

例如修改：

```text
packages/cart-domain
```

理論上只需要重新檢查：

```text
apps/checkout
apps/member
```

不一定需要重新建置：

```text
apps/vendor-console
apps/admin-console
```

依賴圖可以表示為：

```text
packages/cart-domain
        │
        ├──────────────► apps/checkout
        │
        └──────────────► apps/member

packages/vendor-domain
        │
        └──────────────► apps/vendor-console
```

Turborepo 可以依照 package dependency graph，判斷哪些任務受到變更影響，並搭配快取減少重複建置。

---

# 三、什麼是 Multi-App 架構？

## 3.1 定義

Multi-App 是指：

> 在同一個產品或 Monorepo 中，建立多個可以獨立執行與部署的 Application。

Monorepo 回答的是：

> 「程式碼放在哪裡管理？」

Multi-App 回答的是：

> 「系統要拆成哪些可獨立執行與部署的應用程式？」

因此兩者不是同一件事。

| 概念 | 解決的問題 |
|---|---|
| Monorepo | 多個專案與套件如何集中管理 |
| Multi-App | 大型產品如何拆成多個應用程式 |
| Micro Frontend | 多個前端應用如何共同組成使用者看到的產品 |
| Shared Modules | 哪些程式碼可以被不同 App 共用 |
| Module Boundaries | 哪些模組可以依賴哪些模組 |

---

## 3.2 為什麼 momo 不適合只有一個巨大 App？

假設 momo 所有功能都放在：

```text
apps/web
```

隨著系統擴大，可能出現：

```text
apps/web/
├── product/
├── search/
├── cart/
├── checkout/
├── member/
├── campaign/
├── vendor/
├── admin/
├── customer-service/
└── ...
```

這種架構容易產生：

- 建置時間越來越長
- 任一功能修改都可能影響整個網站
- 團隊之間互相阻塞
- Deployment 風險擴大
- 權限與環境變數難以隔離
- 活動站流量暴增時，必須連整個系統一起擴容
- 後台程式碼可能被錯誤打包到前台
- 模組之間逐漸形成循環依賴

因此可按照「業務領域、流量特性與發布頻率」拆成多個 App。

---

## 3.3 momo Multi-App 劃分範例

| App | 負責範圍 | 流量特性 | 部署特性 |
|---|---|---|---|
| `storefront` | 首頁、分類、商品詳情 | 高流量、SEO 重要 | 穩定發布、CDN 快取 |
| `search` | 搜尋、排序、篩選 | 高查詢量 | 可獨立擴容 |
| `campaign` | 雙 11、週年慶活動 | 短時間流量暴增 | 快速發布、快速回滾 |
| `member` | 登入、會員、訂單 | 個人化程度高 | 重視安全與 Session |
| `checkout` | 購物車、付款、結帳 | 高交易風險 | 嚴格測試、保守發布 |
| `vendor-console` | 廠商商品與訂單管理 | 低公開流量 | 內部權限導向 |
| `admin-console` | momo 營運管理 | 內部使用 | 與公開網站隔離 |

---

## 3.4 URL 與 App 的對應關係

對使用者而言，仍然可以使用同一個網域：

```text
https://www.momoshop.com.tw/
```

但內部可將不同路徑導向不同 App：

```text
/                       → storefront
/category/*             → storefront
/goods/*                → storefront
/search/*               → search
/campaign/*             → campaign
/member/*               → member
/cart/*                  → checkout
/checkout/*              → checkout
```

架構示意：

```text
                         使用者
                            │
                            ▼
               www.momoshop.com.tw
                            │
                CDN / WAF / Edge Router
                            │
        ┌───────────────────┼─────────────────────┐
        │                   │                     │
        ▼                   ▼                     ▼
   /goods/*             /search/*           /campaign/*
 storefront App          search App          campaign App
        │                   │                     │
        └───────────────────┼─────────────────────┘
                            │
                            ▼
                     BFF / API Gateway
                            │
       ┌──────────────┬─────┼──────┬──────────────┐
       ▼              ▼            ▼              ▼
 Product Service  Search Service  Pricing      Promotion
```

這類模式可使用：

- Edge Router
- Reverse Proxy
- API Gateway
- CDN Route Rules
- Next.js Multi-Zones

將同一網域下的不同路徑交給不同應用程式。

---

## 3.5 Multi-App 不等於每個頁面都拆成一個 App

拆分粒度過細也會造成問題。

例如不建議直接拆成：

```text
apps/product-card
apps/product-price
apps/add-to-cart-button
apps/product-image
```

因為這些通常只是 UI 或 Feature Module，不應該全部成為獨立部署單位。

較合理的原則是：

> App 應對應一個具有獨立業務責任、團隊責任、發布週期或流量特性的領域。

例如：

```text
商品瀏覽       → storefront App
搜尋體驗       → search App
促銷活動       → campaign App
交易流程       → checkout App
會員服務       → member App
```

---

# 四、什麼是 Module Boundaries？

## 4.1 定義

Module Boundaries 是指：

> 明確規定不同 App、Package、Domain 與 Layer 之間，哪些可以互相依賴，哪些不可以。

Monorepo 最大的風險之一，是所有程式都在同一個 Repository，導致工程師很容易直接跨目錄 import。

例如：

```tsx
// 錯誤示範
import { calculateFinalPrice } from "../../../checkout/src/internal/price";
```

這代表 `storefront` 直接使用 `checkout` App 的內部程式碼。

後果可能是：

- checkout 修改內部實作時，storefront 跟著壞掉
- App 無法獨立部署
- 形成隱藏依賴
- 測試範圍擴大
- 依賴圖逐漸失控

因此必須設計並強制執行 Module Boundaries。

---

## 4.2 momo 建議的依賴分層

```text
┌────────────────────────────────────────────┐
│                Application Layer           │
│ storefront / search / campaign / checkout │
└──────────────────────┬─────────────────────┘
                       │ 可以依賴
                       ▼
┌────────────────────────────────────────────┐
│                  Feature Layer             │
│ product-list / cart-panel / member-login   │
└──────────────────────┬─────────────────────┘
                       │ 可以依賴
                       ▼
┌────────────────────────────────────────────┐
│                   Domain Layer             │
│ product / cart / promotion / member        │
└──────────────────────┬─────────────────────┘
                       │ 可以依賴
                       ▼
┌────────────────────────────────────────────┐
│              Infrastructure Layer          │
│ api-client / analytics / auth / storage    │
└──────────────────────┬─────────────────────┘
                       │ 可以依賴
                       ▼
┌────────────────────────────────────────────┐
│                Foundation Layer            │
│ ui / utils / types / config / tokens       │
└────────────────────────────────────────────┘
```

依賴應該大致維持由上往下：

```text
App → Feature → Domain → Infrastructure / Foundation
```

不應反向依賴：

```text
UI → Checkout App                    X
Domain → Feature                     X
API Client → Storefront App          X
Search App → Checkout App Internal   X
```

---

## 4.3 建議的 Module 類型

可以替每個 package 設定兩種標籤：

### Scope：屬於哪一個業務領域

```text
scope:product
scope:search
scope:cart
scope:checkout
scope:member
scope:promotion
scope:shared
```

### Type：模組扮演什麼角色

```text
type:app
type:feature
type:domain
type:data-access
type:ui
type:util
type:config
```

例如：

```text
packages/product-domain
├── scope:product
└── type:domain

packages/product-data-access
├── scope:product
└── type:data-access

packages/ui
├── scope:shared
└── type:ui

apps/checkout
├── scope:checkout
└── type:app
```

---

## 4.4 momo 依賴規則範例

| 來源模組 | 允許依賴 |
|---|---|
| `type:app` | feature、domain、data-access、ui、util |
| `type:feature` | domain、data-access、ui、util |
| `type:domain` | domain、util |
| `type:data-access` | domain、util |
| `type:ui` | ui、util |
| `type:util` | util |
| `scope:product` | product、shared、promotion |
| `scope:cart` | cart、product、promotion、shared |
| `scope:checkout` | checkout、cart、member、promotion、shared |
| `scope:shared` | shared |

其中 `scope:shared` 不應依賴任何特定業務領域：

```text
shared → product      X
shared → checkout     X
shared → member       X
```

否則 shared 會失去真正的通用性。

---

## 4.5 合法與非法依賴範例

### 合法依賴

```text
apps/checkout
    │
    ├──► packages/cart-feature
    ├──► packages/cart-domain
    ├──► packages/payment-data-access
    ├──► packages/auth
    └──► packages/ui
```

### 非法依賴

```text
packages/ui
    └──► apps/checkout              X

packages/product-domain
    └──► packages/product-feature   X

apps/storefront
    └──► apps/checkout/src/private  X

packages/shared-utils
    └──► packages/member-domain     X
```

---

## 4.6 Domain 之間不要隨意互相依賴

錯誤示範：

```text
product-domain ─────► cart-domain
       ▲                  │
       └──────────────────┘
```

這會形成循環依賴：

```text
product → cart → product
```

較好的方式是抽出共同契約：

```text
             commerce-contracts
                ▲          ▲
                │          │
       product-domain    cart-domain
```

例如：

```ts
// packages/commerce-contracts/src/product.ts

export interface PurchasableProduct {
  productId: string;
  skuId: string;
  price: number;
  available: boolean;
}
```

```ts
// packages/product-domain

import type { PurchasableProduct } from "@momo/commerce-contracts";
```

```ts
// packages/cart-domain

import type { PurchasableProduct } from "@momo/commerce-contracts";
```

如此 `product-domain` 與 `cart-domain` 不需要直接互相依賴。

---

# 五、什麼是 Shared Modules？

## 5.1 定義

Shared Modules 是指：

> 可以被多個 App 或 Domain 重複使用，而且具有穩定公開介面的共用模組。

例如：

```text
@momo/ui
@momo/auth
@momo/analytics
@momo/api-client
@momo/i18n
@momo/observability
```

但不是看到兩段程式碼相似，就立刻抽成 shared。

Shared Module 應符合至少一部分條件：

- 被兩個以上 App 使用
- 概念與行為具有一致性
- 有清楚的責任範圍
- 有穩定的 Public API
- 不依賴某一個 App 的內部狀態
- 有明確維護團隊
- 有測試與版本變更規則

---

## 5.2 momo 的 Shared Modules 分類

```text
packages/
├── foundation/
│   ├── ui/
│   ├── design-tokens/
│   ├── icons/
│   ├── utils/
│   └── types/
│
├── platform/
│   ├── auth/
│   ├── analytics/
│   ├── observability/
│   ├── feature-flags/
│   ├── i18n/
│   └── api-client/
│
├── domains/
│   ├── product/
│   ├── cart/
│   ├── member/
│   ├── promotion/
│   └── order/
│
└── tooling/
    ├── eslint-config/
    ├── typescript-config/
    ├── test-config/
    └── tailwind-config/
```

可以分成四類：

| 類型 | 用途 | 範例 |
|---|---|---|
| Foundation | 最底層通用能力 | UI、Utils、Types |
| Platform | 跨 App 平台能力 | Auth、Analytics、API Client |
| Domain | 特定商業領域能力 | Product、Cart、Promotion |
| Tooling | 工程規範 | ESLint、TypeScript、Testing |

---

# 六、Shared UI 如何設計？

## 6.1 不要把所有元件都放進一個巨大 UI Package

以下設計雖然簡單：

```text
packages/ui/
├── Button.tsx
├── ProductCard.tsx
├── CheckoutForm.tsx
├── OrderHistory.tsx
├── CampaignBanner.tsx
└── VendorProductEditor.tsx
```

但它混合了：

- 基礎 UI
- 商品業務
- 結帳業務
- 訂單業務
- 活動業務
- 廠商後台業務

最後 `@momo/ui` 會變成另一個大型 Monolith。

---

## 6.2 建議分成基礎 UI 與 Domain UI

```text
packages/
├── ui/
│   ├── button/
│   ├── modal/
│   ├── input/
│   ├── tabs/
│   └── skeleton/
│
├── product-ui/
│   ├── product-card/
│   ├── product-gallery/
│   ├── product-price/
│   └── product-badge/
│
├── cart-ui/
│   ├── cart-item/
│   ├── cart-summary/
│   └── quantity-selector/
│
└── promotion-ui/
    ├── coupon-badge/
    ├── campaign-banner/
    └── countdown-timer/
```

依賴關係：

```text
product-ui ─────► ui
cart-ui ────────► ui
promotion-ui ───► ui

ui ─────────────► product-ui     X
```

---

## 6.3 基礎 UI 應該偏向無業務邏輯

例如：

```tsx
import type { ButtonHTMLAttributes, ReactNode } from "react";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  children: ReactNode;
  variant?: "primary" | "secondary" | "danger";
  loading?: boolean;
}

export function Button({
  children,
  variant = "primary",
  loading = false,
  disabled,
  ...props
}: ButtonProps) {
  return (
    <button
      data-variant={variant}
      disabled={disabled || loading}
      {...props}
    >
      {loading ? "處理中" : children}
    </button>
  );
}
```

`Button` 不應該知道：

- 商品 ID
- 購物車 API
- 會員等級
- momo 幣
- 優惠券
- 促銷活動

這些應該由上層 Feature 或 Domain Component 負責。

---

# 七、Shared Domain Module 如何設計？

以商品領域為例，可以拆成：

```text
packages/
├── product-domain/
│   ├── models/
│   ├── schemas/
│   ├── rules/
│   └── index.ts
│
├── product-data-access/
│   ├── product-api.ts
│   ├── product-query.ts
│   └── index.ts
│
├── product-ui/
│   ├── product-card.tsx
│   └── index.ts
│
└── product-feature/
    ├── product-detail.tsx
    └── index.ts
```

各層責任：

| Module | 責任 |
|---|---|
| `product-domain` | 商品型別、資料規則、純函式 |
| `product-data-access` | 呼叫 Product API、Query Key、資料轉換 |
| `product-ui` | 商品相關視覺元件 |
| `product-feature` | 組合 API、Domain 與 UI 完成功能 |

依賴方向：

```text
product-feature
      │
      ├──► product-data-access
      ├──► product-domain
      └──► product-ui
                  │
                  └──► ui
```

---

## 7.1 Product Domain 範例

```ts
// packages/product-domain/src/models/product.ts

export interface Product {
  id: string;
  name: string;
  listPrice: number;
  salePrice: number;
  stockStatus: "in-stock" | "low-stock" | "out-of-stock";
  promotionLabels: string[];
}
```

```ts
// packages/product-domain/src/rules/calculate-discount-rate.ts

import type { Product } from "../models/product";

export function calculateDiscountRate(product: Product): number {
  if (product.listPrice <= 0) {
    return 0;
  }

  return Math.round(
    ((product.listPrice - product.salePrice) / product.listPrice) * 100,
  );
}
```

這些 Domain 函式應保持：

- 不依賴 React
- 不直接呼叫 API
- 不讀取 Browser Storage
- 不操作 DOM
- 容易進行單元測試

---

## 7.2 Product Data Access 範例

```ts
// packages/product-data-access/src/get-product.ts

import type { Product } from "@momo/product-domain";
import { apiClient } from "@momo/api-client";

interface ProductResponse {
  goodsNo: string;
  goodsName: string;
  originalPrice: number;
  sellingPrice: number;
  stock: number;
  labels: string[];
}

export async function getProduct(productId: string): Promise<Product> {
  const response = await apiClient.get<ProductResponse>(
    `/products/${productId}`,
  );

  return {
    id: response.goodsNo,
    name: response.goodsName,
    listPrice: response.originalPrice,
    salePrice: response.sellingPrice,
    stockStatus: response.stock > 0 ? "in-stock" : "out-of-stock",
    promotionLabels: response.labels,
  };
}
```

Data Access 負責把後端資料格式轉換成前端 Domain Model。

因此 Component 不需要知道後端欄位叫做：

```text
goodsNo
goodsName
sellingPrice
```

Component 只需要使用穩定的前端模型：

```text
id
name
salePrice
```

---

# 八、Public API 設計

Shared Module 不應讓外部 App 任意存取內部檔案。

錯誤示範：

```tsx
import { ProductCard } from
  "@momo/product-ui/src/components/product-card/private/product-card";
```

正確方式：

```tsx
import { ProductCard } from "@momo/product-ui";
```

透過 `index.ts` 控制 Public API：

```ts
// packages/product-ui/src/index.ts

export { ProductCard } from "./product-card/product-card";
export type { ProductCardProps } from "./product-card/product-card.types";
```

不公開的內容：

```text
internal helpers
private hooks
test fixtures
implementation details
internal constants
```

---

## 8.1 package.json exports 範例

```json
{
  "name": "@momo/product-ui",
  "private": true,
  "exports": {
    ".": "./src/index.ts",
    "./product-card": "./src/product-card/index.ts"
  },
  "peerDependencies": {
    "react": "^19.0.0"
  }
}
```

使用者只能透過允許的入口引用：

```tsx
import { ProductCard } from "@momo/product-ui";
```

或：

```tsx
import { ProductCard } from "@momo/product-ui/product-card";
```

不能任意 deep import 私有檔案。

---

# 九、避免「Shared Everything」

Monorepo 常見錯誤是建立：

```text
packages/shared
```

然後把所有東西都放進去：

```text
packages/shared/
├── components/
├── hooks/
├── utils/
├── api/
├── constants/
├── product/
├── cart/
├── member/
└── checkout/
```

這會造成：

- package 責任不清
- 任何 App 都依賴 shared
- shared 反過來又依賴各 Domain
- 修改一個小工具導致大量 App 重新測試
- 容易形成循環依賴
- 無法判斷維護團隊
- Tree Shaking 與 Bundle 分析困難

建議將模組依照責任拆分：

```text
@momo/ui
@momo/utils
@momo/auth
@momo/api-client
@momo/product-domain
@momo/cart-domain
@momo/promotion-domain
```

不要只建立一個無限膨脹的：

```text
@momo/shared
```

---

# 十、App 之間應如何溝通？

Multi-App 架構下，App 不應直接引用另一個 App 的內部程式碼。

錯誤方式：

```text
storefront
    └── import checkout/src/cart-store
```

正確方式可分成三種。

---

## 10.1 透過 Shared Contract

```ts
// packages/cart-contracts

export interface CartSummary {
  itemCount: number;
  totalAmount: number;
}
```

各 App 使用相同契約，但不共享 App 內部實作。

---

## 10.2 透過後端 API 或 BFF

```text
storefront App
      │
      ▼
Cart BFF / Cart API
      ▲
      │
checkout App
```

例如首頁 Header 顯示購物車數量時，storefront 不需要讀取 checkout App 的 Store，而是呼叫 Cart Summary API。

---

## 10.3 透過有限的 Browser Event

同一個頁面若有跨模組溝通需求，可以透過明確定義的事件：

```ts
window.dispatchEvent(
  new CustomEvent("momo:cart-updated", {
    detail: {
      itemCount: 3,
    },
  }),
);
```

監聽：

```ts
window.addEventListener("momo:cart-updated", handleCartUpdated);
```

但 Event 必須：

- 有型別定義
- 有命名規則
- 有事件 Schema
- 有版本相容策略
- 避免成為任意傳遞資料的全域 Event Bus

---

# 十一、momo 完整依賴架構範例

```text
┌──────────────────────────────────────────────────────────────┐
│                         Apps                                 │
│                                                              │
│  storefront     search      campaign     member    checkout  │
└──────┬────────────┬────────────┬────────────┬─────────┬──────┘
       │            │            │            │         │
       ▼            ▼            ▼            ▼         ▼
┌──────────────────────────────────────────────────────────────┐
│                       Feature Packages                       │
│                                                              │
│ product-list  search-result  campaign-page  login  cart-flow │
└──────┬────────────┬────────────┬────────────┬─────────┬──────┘
       │            │            │            │         │
       ▼            ▼            ▼            ▼         ▼
┌──────────────────────────────────────────────────────────────┐
│                        Domain Packages                       │
│                                                              │
│ product-domain  search-domain  promotion  member  cart/order │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                        Platform Packages                     │
│                                                              │
│ api-client  auth  analytics  feature-flags  i18n  telemetry │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                       Foundation Packages                    │
│                                                              │
│ ui  design-tokens  icons  utils  types  config               │
└──────────────────────────────────────────────────────────────┘
```

核心規則：

```text
Apps 可以依賴 Packages
Packages 不可以依賴 Apps

Feature 可以依賴 Domain
Domain 不可以依賴 Feature

Domain 可以依賴 Foundation
Foundation 不可以依賴 Domain

App A 不可以直接 import App B 的程式碼
跨 App 共用應抽成獨立 Package 或透過 API 溝通
```

---

# 十二、pnpm Workspace 設定範例

```yaml
# pnpm-workspace.yaml

packages:
  - "apps/*"
  - "packages/*"
  - "packages/domains/*"
  - "packages/platform/*"
  - "packages/foundation/*"
  - "packages/tooling/*"
```

App 使用 Workspace Package：

```json
{
  "name": "@momo/storefront",
  "private": true,
  "dependencies": {
    "@momo/ui": "workspace:*",
    "@momo/product-domain": "workspace:*",
    "@momo/product-ui": "workspace:*",
    "@momo/api-client": "workspace:*"
  }
}
```

`workspace:*` 表示這個 dependency 必須來自目前 Workspace，而不是意外從公開 npm Registry 安裝同名套件。

---

# 十三、Turborepo 任務設計範例

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**", "!.next/cache/**"]
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "typecheck": {
      "dependsOn": ["^typecheck"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

其中：

```text
^build
```

代表先建置目前 App 所依賴的 Workspace Packages，再建置 App 本身。

例如：

```text
@momo/ui ───────────────┐
@momo/product-domain ───┼──► @momo/storefront
@momo/api-client ───────┘
```

建置順序：

```text
1. @momo/ui
2. @momo/product-domain
3. @momo/api-client
4. @momo/storefront
```

---

# 十四、實際案例：新增「雙 11 商品卡」

假設 momo 雙 11 需要在商品卡加入：

- 限時折扣
- 倒數計時
- momo 幣回饋
- 庫存警示
- 加入購物車

不建議把所有邏輯都寫進基礎 `ProductCard`：

```tsx
<ProductCard
  product={product}
  campaignId="double-11"
  coupon={coupon}
  cartApi={cartApi}
  memberLevel={memberLevel}
  momoCoin={momoCoin}
/>
```

這會讓 ProductCard 與活動、會員、購物車全部耦合。

建議拆成：

```text
@momo/ui
└── Card、Button、Badge、Skeleton

@momo/product-ui
└── ProductCard、ProductPrice、StockLabel

@momo/promotion-ui
└── CampaignBadge、CountdownTimer

@momo/cart-feature
└── AddToCartButton

@momo/campaign-feature
└── Double11ProductCard
```

組合範例：

```tsx
import { ProductCard } from "@momo/product-ui";
import { CampaignBadge, CountdownTimer } from "@momo/promotion-ui";
import { AddToCartButton } from "@momo/cart-feature";

export function Double11ProductCard({ product, campaign }) {
  return (
    <ProductCard product={product}>
      <CampaignBadge label="雙 11 限時優惠" />

      <CountdownTimer endAt={campaign.endAt} />

      <AddToCartButton
        productId={product.id}
        skuId={product.skuId}
      />
    </ProductCard>
  );
}
```

依賴關係：

```text
campaign-feature
    ├──► product-ui
    ├──► promotion-ui
    └──► cart-feature

product-ui
    └──► ui

promotion-ui
    └──► ui

cart-feature
    ├──► cart-domain
    ├──► cart-data-access
    └──► ui
```

---

# 十五、實際案例：商品價格顯示

商品價格不只是：

```tsx
<span>{price}</span>
```

大型電商可能包含：

- 原價
- 售價
- 活動價
- 會員價
- 折價券後價格
- momo 幣回饋
- 分期付款資訊

建議分層：

```text
Pricing API
    │
    ▼
pricing-data-access
    │ 將 API 格式轉成 Domain Model
    ▼
pricing-domain
    │ 計算與驗證價格顯示規則
    ▼
pricing-ui
    │ 顯示價格
    ▼
storefront / search / campaign
```

Domain Model：

```ts
export interface ProductPrice {
  listPrice: number;
  salePrice: number;
  campaignPrice?: number;
  memberPrice?: number;
  currency: "TWD";
}
```

純函式：

```ts
export function getDisplayPrice(price: ProductPrice): number {
  const candidates = [
    price.salePrice,
    price.campaignPrice,
    price.memberPrice,
  ].filter((value): value is number => value !== undefined);

  return Math.min(...candidates);
}
```

如此搜尋站、商品站與活動站可以使用一致的價格規則。

但需要注意：

> 前端價格計算主要用於顯示，真正交易金額仍必須由後端重新驗證，不能信任瀏覽器傳入的價格。

---

# 十六、Module Boundaries 的自動化檢查

只有架構文件通常不夠，因為工程師仍可能不小心違反規則。

因此應在以下階段自動檢查：

```text
開發者 IDE
    │
    ▼
ESLint
    │
    ▼
Pre-commit Hook
    │
    ▼
Pull Request CI
    │
    ▼
Architecture Validation
```

可以檢查：

- App 是否直接依賴另一個 App
- 是否出現跨 Package Deep Import
- 是否形成 Circular Dependency
- Shared Package 是否依賴特定 Domain
- Domain 是否反向依賴 Feature
- 前台是否引用後台 Package
- Server-only Module 是否進入 Client Bundle

例如：

```text
apps/storefront
    └──► apps/checkout             禁止

packages/ui
    └──► packages/cart-domain      禁止

packages/cart-domain
    └──► packages/cart-feature     禁止
```

---

# 十七、團隊 Ownership 設計

大型 Monorepo 必須搭配模組維護責任。

```text
packages/ui                     → Design System Team
packages/auth                   → Identity Team
packages/api-client             → Frontend Platform Team
packages/product-domain         → Product Experience Team
packages/cart-domain            → Cart Team
packages/checkout-feature       → Checkout Team
apps/campaign                   → Campaign Team
```

可以透過 CODEOWNERS 控制 Review：

```text
/packages/ui/                  @momo/design-system
/packages/auth/                @momo/identity-team
/packages/product-domain/      @momo/product-team
/packages/cart-domain/         @momo/cart-team
/apps/checkout/                @momo/checkout-team
/apps/campaign/                @momo/campaign-team
```

當工程師修改結帳模組時，CI 與 Git 平台可要求 Checkout Team Review。

---

# 十八、常見錯誤

## 錯誤一：Monorepo 變成所有東西互相 import

```text
App A → App B → Shared → App C → App A
```

解法：

```text
建立明確 Layer
限制跨 App Import
建立 Public API
檢查 Circular Dependency
```

---

## 錯誤二：建立超大型 shared package

```text
@momo/shared
```

包含所有 UI、API、Domain 與 Hook。

解法：

```text
@momo/ui
@momo/auth
@momo/analytics
@momo/product-domain
@momo/cart-domain
@momo/promotion-domain
```

依照責任拆分。

---

## 錯誤三：共用 App 的內部 State

例如 storefront 直接使用 checkout 的 Zustand Store：

```tsx
import { useCartStore } from
  "@momo/checkout/src/store/private-cart-store";
```

解法：

- 抽出 cart contract
- 透過 Cart API 取得狀態
- 或建立真正可共享的 `@momo/cart-client`
- 不直接引用另一個 App 的 private store

---

## 錯誤四：所有 App 必須一起部署

Monorepo 是一起管理程式碼，不代表一定要一起發布。

應透過 CI 判斷 affected projects：

```text
修改 product-ui
    │
    ├── storefront 受到影響
    ├── search 受到影響
    └── campaign 受到影響

checkout 若未引用 product-ui
    └── 不需要重新部署
```

---

## 錯誤五：過早抽象化

只有一個 App 使用、需求仍快速變動的功能，不需要立刻抽成 shared package。

較好的演進方式：

```text
第一次出現
    │
    ▼
先放在 App 內部
    │
    ▼
第二個 App 出現相同需求
    │
    ▼
確認行為與概念是否真的一致
    │
    ▼
再抽成 Shared Module
```

不是「程式碼長得像」就共用，而是「業務概念與變更原因一致」才共用。

---

# 十九、momo 建議架構總結

```text
                         momo Monorepo
                              │
            ┌─────────────────┴──────────────────┐
            │                                    │
          apps/                              packages/
            │                                    │
   可獨立建置與部署的 App               可重複使用的模組
            │                                    │
  ┌─────────┼──────────┐             ┌───────────┼───────────┐
  │         │          │             │           │           │
商品站    搜尋站     結帳站        UI/Utils    Platform    Domain
  │         │          │             │           │           │
  └─────────┴──────────┴─────────────┴───────────┴───────────┘
                              │
                     Module Boundaries
                              │
          限制依賴方向、禁止跨 App 私有引用
                              │
                        pnpm Workspace
                              │
                 管理 Workspace Packages
                              │
                         Turborepo
                              │
            任務編排、快取、Affected Build、CI
```

---

# 二十、核心結論

## Monorepo

負責將 momo 的多個 App、Shared Packages 與工程設定集中管理。

```text
一個 Repository
不等於一個 Application
也不等於一次 Deployment
```

## Multi-App

按照業務領域、團隊責任、發布頻率與流量特性，將 momo 拆成多個可獨立執行與部署的 App。

```text
storefront
search
campaign
member
checkout
vendor-console
admin-console
```

## Module Boundaries

限制依賴方向，避免 Monorepo 最後變成任意引用、循環依賴的巨大程式碼庫。

```text
App → Feature → Domain → Foundation
```

並禁止：

```text
Package → App
Domain → Feature
Shared → Specific Domain
App A → App B Private Code
```

## Shared Modules

將真正穩定、可重用的能力抽成獨立 Package。

```text
@momo/ui
@momo/auth
@momo/api-client
@momo/analytics
@momo/product-domain
@momo/cart-domain
```

但不應把所有程式都塞進一個 `@momo/shared`。

## 最終設計原則

> Monorepo 解決的是集中治理，Multi-App 解決的是系統拆分，Shared Modules 解決的是能力重用，而 Module Boundaries 則負責防止這些模組失去秩序。

對 momo 這種大型電商而言，理想架構不是單純追求「共用越多越好」，而是：

> 在團隊可獨立交付、系統可獨立部署與共用能力一致性之間取得平衡。