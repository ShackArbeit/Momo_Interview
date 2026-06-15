# momo 電商中的 CI/CD 實踐解析

> 面試常見問題：
> 「請說明什麼是 CI/CD？」
>
> 如果是 momo 資深前端工程師面試，我會從：
>
> 1. CI/CD 基本概念
> 2. 為什麼大型電商一定要做 CI/CD
> 3. momo 前端 CI/CD 流程
> 4. 高流量環境下如何安全部署
> 5. 與 Monorepo、Micro Frontend 的結合
>
> 五個層面回答。

---

# 一、什麼是 CI/CD ?

CI/CD 是 DevOps 中最重要的實踐之一。

CI = Continuous Integration
CD = Continuous Delivery / Continuous Deployment

目標：

讓程式碼從開發到上線

從

「人工操作」

變成

「自動化流程」

---

## 傳統開發流程

```text
開發者
   ↓
Git Push
   ↓
通知測試
   ↓
手動 Build
   ↓
手動上傳 Server
   ↓
手動驗證

問題：

❌ 容易出錯

❌ 發版慢

❌ 無法頻繁部署

❌ 高風險
```

---

## CI/CD 流程

```text
Developer
    ↓
Git Push
    ↓
CI Pipeline
    ↓
Lint
    ↓
Unit Test
    ↓
Build
    ↓
Deploy
    ↓
Monitoring

全自動完成
```

---

# 二、CI 是什麼？

Continuous Integration

持續整合

每次 Commit 都自動驗證程式碼。

---

例如：

```bash
git push
```

觸發：

```bash
npm run lint

npm run test

npm run build
```

如果失敗：

```text
❌ 不允許 Merge
```

成功：

```text
✅ 允許 Merge
```

---

## momo 的 CI 可能包含

### ESLint

```bash
npm run lint
```

檢查：

```tsx
const data = any
```

是否符合規範

---

### Type Check

```bash
npm run type-check
```

檢查：

```tsx
const age: number = "18"
```

直接擋掉

---

### Unit Test

```bash
npm run test
```

Jest

Vitest

Testing Library

---

### Build Verification

```bash
npm run build
```

確保：

```text
商品頁

首頁

購物車

結帳頁
```

都能正常編譯。

---

# 三、CD 是什麼？

Continuous Delivery

或

Continuous Deployment

---

## Continuous Delivery

```text
Build 完

等待人工確認

再部署
```

---

## Continuous Deployment

```text
Build 完

直接部署
```

---

大型電商通常：

```text
Continuous Delivery
```

比較常見。

---

因為：

```text
錯誤上線

可能損失數百萬營收
```

---

# 四、momo 為什麼非常需要 CI/CD？

momo 屬於大型電商平台。:contentReference[oaicite:0]{index=0}

具有：

```text
大量商品

大量會員

大量訂單

大量促銷活動

大量流量
```

---

例如：

618

雙11

雙12

黑五

年貨節

---

流量可能暴增：

```text
10 倍

20 倍

甚至更多
```

---

如果工程師：

```text
手動上版
```

風險極高。

因此：

```text
CI/CD
+
自動化驗證
+
灰度發布
+
監控
```

幾乎是必要條件。

---

# 五、momo 前端 CI/CD 架構

```text
Developer
    ↓
GitHub / GitLab
    ↓
Pull Request
    ↓
CI Pipeline
    ├─ ESLint
    ├─ Type Check
    ├─ Unit Test
    ├─ E2E Test
    └─ Build
    ↓
Artifact
    ↓
Docker Image
    ↓
Kubernetes
    ↓
Staging
    ↓
Production
    ↓
Monitoring
```

momo 的 DevOps 職缺亦明確提到 CI/CD Pipeline、自動化部署、Docker、Kubernetes、Observability 等能力需求。:contentReference[oaicite:1]{index=1}

---

# 六、momo 前端 CI 實際範例

假設開發：

```text
商品頁優惠券功能
```

---

工程師提交 PR

```bash
git push
```

---

GitHub Actions

```yaml
name: Frontend CI

on:
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - npm install
      - npm run lint
      - npm run test
      - npm run build
```

---

若：

```text
Lint Error

Test Fail

Build Fail
```

直接拒絕 Merge。

---

# 七、momo 前端 CD 流程

PR Merge

↓

Main Branch

↓

Build

↓

Docker

↓

K8S

↓

Staging

↓

Production

---

```text
Main Branch
     ↓
Docker Build
     ↓
Docker Registry
     ↓
Kubernetes
     ↓
Staging
     ↓
Smoke Test
     ↓
Production
```

---

# 八、什麼是 Blue-Green Deployment

大型電商常用。

---

假設：

```text
Version A
```

正在服務使用者。

```text
A (Live)
```

---

新版本：

```text
Version B
```

先部署完成。

```text
B (Ready)
```

---

切換流量：

```text
A → B
```

---

架構圖：

```text
Load Balancer

      ↓

 ┌───────┐
 │  A    │
 └───────┘

      ↓

 ┌───────┐
 │  B    │
 └───────┘
```

---

若有問題：

```text
立即切回 A
```

---

# 九、什麼是 Canary Release

momo 更可能使用。

---

先給：

```text
5%
```

流量

---

確認沒問題

```text
20%
```

↓

```text
50%
```

↓

```text
100%
```

---

架構圖

```text
Users

100%

 ├── 95% → V1
 │
 └── 5% → V2
```

---

如果：

```text
錯誤率暴增
```

直接回滾。

---

# 十、CI/CD + Micro Frontend

假設 momo：

```text
首頁團隊

商品頁團隊

搜尋團隊

購物車團隊

會員團隊
```

各自獨立開發。

---

```text
apps/

├── home
├── product
├── search
├── cart
├── member
```

---

每個 App

都有自己的：

```text
CI

CD

Version
```

---

例如：

```text
Cart
```

更新

不需重建：

```text
Home

Search

Member
```

---

這就是：

```text
Independent Deployment
```

---

# 十一、CI/CD + Monorepo

假設 momo 使用：

```text
Turborepo

pnpm workspace
```

---

結構：

```text
apps/

packages/
```

---

```text
apps
 ├── home
 ├── cart
 └── member

packages
 ├── ui
 ├── api
 └── utils
```

---

Turbo Pipeline：

```bash
turbo run build
```

只 Build 有變更部分。

---

例如：

```text
只修改 Cart
```

---

Turbo：

```text
Home 不重建

Member 不重建
```

---

Build 時間：

```text
20 min
↓
3 min
```

大幅降低。

---

# 十二、momo 前端工程師實際負責的 CI/CD 工作

### 1. 撰寫 GitHub Actions

```yaml
.github/workflows
```

---

### 2. 維護測試流程

```text
Jest

Vitest

Playwright
```

---

### 3. Docker 化

```dockerfile
FROM node:22
```

---

### 4. Kubernetes 部署

```yaml
deployment.yaml
```

---

### 5. Canary Release

```text
5%
20%
50%
100%
```

---

### 6. Rollback

```bash
kubectl rollout undo
```

---

### 7. Monitoring

```text
Grafana

Prometheus

Datadog

Sentry
```

---

# 面試總結版（60秒回答）

CI/CD 是透過自動化流程，讓程式碼從 Commit、測試、Build 到 Deploy 全部標準化與自動化。

以 momo 這種高流量電商而言，前端每次上線都可能影響首頁、商品頁、購物車與結帳流程，因此會建立完整的 CI Pipeline（Lint、Type Check、Unit Test、E2E Test、Build Verification），再透過 CD Pipeline 將應用部署到 Kubernetes。

在大型促銷活動（618、雙11）期間，通常會搭配 Canary Release、Blue-Green Deployment、Rollback 機制以及 Grafana、Prometheus、Sentry 等監控系統，確保新版本可以安全上線並快速回復。若搭配 Turborepo Monorepo 與 Micro Frontend 架構，則可以做到各團隊獨立部署、快速交付與高效協作，這也是現代大型電商前端團隊常見的工程實踐。:contentReference[oaicite:2]{index=2}