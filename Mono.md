# Turborepo 為什麼建議搭配 pnpm？以及 momo 等大型電商如何設計 Monorepo + Micro Frontend + 高流量架構

---

# 一、為什麼 Turborepo 建議搭配 pnpm？

很多人誤以為：

> Turborepo = Monorepo

其實並不是。

---

## Turborepo 負責什麼？

Turborepo 本質上是：

- Build Orchestration Tool
- Task Runner
- Cache System

主要工作：

```text
build
test
lint
type-check
deploy
```

以及決定：

```text
誰先執行
誰後執行
誰可以平行執行
哪些結果可以 Cache
```

例如：

```text
UI Library
   ↓
Product Site
   ↓
Checkout Site
```

Turbo 會知道：

```text
先 build UI
再 build Product
最後 build Checkout
```

---

## pnpm 負責什麼？

pnpm 負責：

```text
Dependency Management
Workspace Management
Package Linking
```

例如：

```yaml
packages:
  - apps/*
  - packages/*
```

pnpm 會建立：

```text
Dependency Graph
```

例如：

```text
Product App
    ↓
UI Package

Checkout App
    ↓
UI Package

Member App
    ↓
UI Package
```

Turbo 就是讀取這個 Graph。

---

# 為什麼不用 npm？

大型 Monorepo 會遇到：

```text
200+
300+
500+
Packages
```

若使用 npm：

```text
node_modules 巨大

重複安裝

CI 慢

Disk 空間浪費
```

---

## pnpm 的優勢 1

### Content Addressable Store

假設：

```text
React 19
```

有 100 個 Package 使用。

npm：

```text
React
React
React
React
React
...
100 次
```

pnpm：

```text
只存一份

透過 Hard Link
共享
```

節省大量空間。 :contentReference[oaicite:0]{index=0}

---

## pnpm 的優勢 2

### 安裝速度快

pnpm：

```text
Install Faster

CI Faster
```

非常適合：

```text
大型電商
大型企業
Monorepo
```

:contentReference[oaicite:1]{index=1}

---

## pnpm 的優勢 3

### Workspace Support

Turbo 本身就是建立在 Workspace 概念上。

```text
Turbo
     ↑
 Workspace
     ↑
 pnpm
```

Turbo 官方文件甚至直接說：

```text
Turbo 建立於 Workspaces 之上
```

:contentReference[oaicite:2]{index=2}

---

## pnpm 的優勢 4

### 避免 Phantom Dependency

很多大型專案常發生：

```js
import axios
```

但 package.json 根本沒裝。

npm 有時候仍能運作。

到了 CI：

```text
爆炸
```

pnpm 會直接報錯。

因此大型團隊更喜歡 pnpm。 :contentReference[oaicite:3]{index=3}

---

# 二、momo 大型電商 Monorepo 應該怎麼設計？

假設 momo 已全面 Next.js 化。

---

## 業務系統

```text
momo.com.tw
```

其實不是一個網站。

而是：

```text
首頁
商品頁
搜尋頁
購物車
結帳
會員中心
客服中心
直播商城
跨境商城
活動頁
品牌館
```

---

## Monorepo 架構

```text
momo-monorepo

apps/
│
├─ home
├─ product
├─ search
├─ cart
├─ checkout
├─ member
├─ live
├─ campaign
└─ global

packages/
│
├─ ui
├─ design-system
├─ api-client
├─ auth
├─ analytics
├─ seo
├─ i18n
├─ monitoring
├─ utils
└─ feature-flags
```

---

# Shared Module

例如：

```text
Button
Modal
Input
```

放在：

```text
packages/ui
```

---

搜尋頁：

```js
import { Button } from "@momo/ui";
```

商品頁：

```js
import { Button } from "@momo/ui";
```

結帳頁：

```js
import { Button } from "@momo/ui";
```

---

優點：

```text
一致性高

維護成本低

Design System 統一
```

---

# 三、Monorepo 如何支援高流量與高併發？

---

## momo 的流量特徵

平常：

```text
5,000 ~ 20,000 concurrent users
```

雙11：

```text
100,000+
```

618：

```text
50,000+
```

大型演唱會票券：

```text
瞬間流量暴增
```

---

# 問題

若所有功能都放在一個前端：

```text
Mega Frontend
```

結果：

```text
Bundle 超大

Build 超慢

Deploy 風險高
```

---

# Monorepo 解法

讓每個業務獨立：

```text
Product Team

Checkout Team

Search Team

Member Team
```

只部署自己的 App。

---

例如：

```text
apps/search
```

修改後：

```bash
turbo run build --filter=search
```

只 Build Search。

---

不會：

```text
重新 Build 全站
```

這就是：

```text
Incremental Build
```

:contentReference[oaicite:4]{index=4}

---

# 四、Monorepo + Micro Frontend

大型 momo 更可能演進成：

```text
Monorepo
+
Micro Frontend
```

---

# 架構

```text
                    CDN

                     │

                     ▼

            Shell Application
                    │
 ┌──────────┬─────────┬───────────┐
 ▼          ▼         ▼           ▼

Product   Search   Cart     Member

Remote    Remote   Remote   Remote
```

---

## Shell

只負責：

```text
Header

Footer

Layout

Routing
```

---

## Product Team

負責：

```text
商品頁
```

---

## Search Team

負責：

```text
搜尋頁
```

---

## Cart Team

負責：

```text
購物車
```

---

# 技術

通常：

```text
Webpack Module Federation

or

Next.js Module Federation

or

Runtime Composition
```

---

# 五、Monorepo + Micro Frontend + 高流量最終架構

```text
                    CloudFront
                         │
                         ▼
                     CDN Edge
                         │
        ┌─────────────────────────┐
        ▼                         ▼

    SSR Cluster             Static Assets

        │
        ▼

   Shell App

        │
 ┌──────┼───────┬───────┬───────┐
 ▼      ▼       ▼       ▼       ▼

Search Product Cart Member Campaign

Remote Remote Remote Remote Remote
```

---

# 六、momo 實務上最可能的演進路線

Phase 1

```text
Next.js
+
Monorepo
+
Shared Components
```

---

Phase 2

```text
Monorepo
+
Design System
+
Turbo Cache
+
pnpm Workspace
```

---

Phase 3

```text
Monorepo
+
Micro Frontend
+
Independent Deployment
```

---

Phase 4

```text
Multi Region

TW
HK
SG
MY
JP
```

```text
Shared Packages

+
Regional Apps
```

例如：

apps/

```text
tw-store
hk-store
jp-store
sg-store
```

共用：

```text
packages/ui
packages/auth
packages/payment
packages/analytics
```

---

# 面試 momo 時的高分回答

如果被問：

「為什麼 Turborepo 要搭配 pnpm？」

可以回答：

> Turborepo 本身只負責 Build Orchestration 與 Cache，不負責 Workspace 管理；pnpm 提供 Workspace、Dependency Graph、快速安裝與嚴格依賴管理。Turbo 會直接利用 pnpm Workspace 建立出的 Dependency Graph 做增量建置與平行化執行，因此在大型 Monorepo 中，pnpm + Turborepo 幾乎是目前 React/Next.js 生態系最主流的組合。

如果被問：

「momo 的前端架構你會怎麼設計？」

可以回答：

> 我會採用 Monorepo + Turborepo + pnpm Workspace，將 Product、Search、Cart、Checkout、Member 等業務拆分成獨立 App，共享 Design System、API Client 與 Analytics Module；當團隊規模持續成長後，再逐步導入 Micro Frontend，讓各業務團隊可以獨立部署，同時透過 CDN、SSR、Edge Cache 與 Incremental Build 支撐雙11等高流量高併發場景。