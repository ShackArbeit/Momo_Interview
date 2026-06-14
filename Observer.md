# 以台灣 momo 為例：Observability、Monitoring 與 Error Tracking

> 情境假設：
>
> momo 是一個大型電商平台，系統包含：
>
> - Next.js 電商網站
> - App／行動版網站
> - BFF（Backend for Frontend）
> - 商品、搜尋、價格、庫存、促銷、購物車、訂單、支付、會員等微服務
> - Redis、Kafka、資料庫、CDN、Kubernetes
>
> 在雙 11、雙 12 或大型直播促銷活動期間，系統可能瞬間湧入大量流量。
>
> 因此，momo 不只需要知道「伺服器有沒有壞掉」，還需要回答：
>
> - 使用者為什麼無法加入購物車？
> - 為什麼商品頁突然變慢？
> - 哪一個版本發布後錯誤率上升？
> - 是前端、BFF、Redis、資料庫，還是庫存服務出問題？
> - 問題影響多少使用者、商品、裝置與地區？
> - 雙 11 系統是否即將超出負載上限？

---

# 一、三個概念的核心差異

| 概念 | 核心問題 | 主要用途 | 常見資料 |
|---|---|---|---|
| Monitoring | 系統現在是否正常？ | 發現已知問題、觸發告警 | CPU、記憶體、流量、錯誤率、延遲 |
| Error Tracking | 程式發生了什麼錯誤？ | 收集、分類與定位程式例外 | Stack trace、版本、瀏覽器、使用者操作 |
| Observability | 為什麼系統會出現這個狀況？ | 從系統輸出推導內部狀態 | Logs、Metrics、Traces、Events、Profiles |

可以用一句話理解：

```text
Monitoring 告訴你：系統出問題了。

Error Tracking 告訴你：哪一段程式碼發生錯誤。

Observability 幫助你回答：為什麼會出問題，以及問題如何跨系統傳播。
```

它們不是互相取代，而是不同層次的能力：

```text
Monitoring
    │
    │ 發現異常
    ▼
Error Tracking
    │
    │ 找到程式錯誤
    ▼
Observability
    │
    │ 串聯前端、後端、微服務與基礎設施
    ▼
Root Cause Analysis
    │
    ▼
修復、回滾與持續改善
```

---

# 二、什麼是 Monitoring？

## 2.1 定義

Monitoring 是持續蒐集系統指標，並根據預先設定的規則判斷系統是否正常。

例如 momo 可以設定：

```text
如果付款 API 的 5xx 錯誤率連續 5 分鐘高於 2%
→ 發送告警給支付服務團隊

如果商品頁 P95 回應時間超過 2 秒
→ 發送效能告警

如果 Kubernetes Pod CPU 使用率超過 85%
→ 通知平台團隊，或自動擴充 Pod
```

Monitoring 特別擅長回答「已知問題」。

例如團隊已經知道：

- CPU 太高可能造成服務變慢
- Redis connection pool 用盡會造成 API timeout
- 付款錯誤率太高代表結帳流程異常
- CDN cache hit rate 太低會增加 Origin Server 壓力

因此可以事先建立 Dashboard 與 Alert Rule。

---

## 2.2 momo 應監控哪些指標？

Google SRE 常使用四個 Golden Signals：

| 指標 | 說明 | momo 範例 |
|---|---|---|
| Latency | 請求需要多少時間 | 商品 API P95 延遲 800ms |
| Traffic | 系統處理多少流量 | 每秒 50,000 次商品查詢 |
| Errors | 請求失敗比例 | 結帳 API 錯誤率 1.5% |
| Saturation | 資源接近滿載程度 | CPU 90%、DB connection 95% |

## momo 電商監控指標

### 前端指標

```text
Core Web Vitals
├── LCP：主要內容載入速度
├── INP：互動反應速度
└── CLS：頁面版面跳動程度

其他前端指標
├── JavaScript Error Rate
├── API Failure Rate
├── 白畫面比例
├── 商品圖片載入失敗率
├── Checkout Abandonment Rate
└── 不同瀏覽器與裝置的錯誤率
```

### BFF 與微服務指標

```text
商品服務
├── request count
├── error rate
├── P50／P95／P99 latency
└── cache hit rate

庫存服務
├── 查詢延遲
├── 庫存鎖定失敗率
├── 庫存超賣次數
└── DB connection 使用率

支付服務
├── 付款成功率
├── 第三方支付 timeout rate
├── 重複付款事件
└── callback 處理延遲

Kafka
├── consumer lag
├── message throughput
├── failed message count
└── dead-letter queue count
```

---

## 2.3 momo Monitoring 架構範例

```text
┌───────────────────────────────────────────────────────┐
│                    momo 使用者                         │
│          Web／Mobile Web／App／跨境站點                 │
└─────────────────────────┬─────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────┐
│              CDN／WAF／Load Balancer                   │
│                                                       │
│  監控：流量、Cache Hit Rate、4xx、5xx、攻擊流量         │
└─────────────────────────┬─────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────┐
│                Next.js Web／BFF                        │
│                                                       │
│  監控：SSR latency、API latency、錯誤率、RPS            │
└─────────────────────────┬─────────────────────────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
       商品服務        購物車服務       訂單服務
             │            │            │
             └────────────┼────────────┘
                          ▼
              Redis／Kafka／Database
                          │
                          ▼
┌───────────────────────────────────────────────────────┐
│              Metrics Collection                       │
│                                                       │
│       Prometheus／Cloud Monitoring／APM Agent          │
└─────────────────────────┬─────────────────────────────┘
                          ▼
┌───────────────────────────────────────────────────────┐
│              Dashboard & Alerting                      │
│                                                       │
│       Grafana／Alertmanager／PagerDuty／Slack           │
└───────────────────────────────────────────────────────┘
```

---

# 三、什麼是 Error Tracking？

## 3.1 定義

Error Tracking 專門收集應用程式執行期間發生的錯誤，例如：

```javascript
TypeError: Cannot read properties of undefined
```

單純看到這個錯誤還不夠。

現代化 Error Tracking 通常還會記錄：

```text
錯誤類型
├── TypeError
├── ReferenceError
├── APIError
└── ChunkLoadError

錯誤環境
├── production
├── staging
└── development

版本資訊
├── Release version
├── Git commit SHA
└── Source map

使用者環境
├── 瀏覽器
├── 作業系統
├── 裝置
├── 國家／地區
└── 頁面 URL

操作上下文
├── 使用者前一個操作
├── Breadcrumb
├── API request
├── Feature flag
└── Session replay
```

---

## 3.2 momo 前端錯誤範例

假設雙 11 活動上線後，部分使用者按下「加入購物車」沒有反應。

前端出現：

```javascript
TypeError: Cannot read properties of undefined
  at addToCart (ProductDetail.tsx:138)
```

Error Tracking 平台可能顯示：

```text
Issue：ADD_TO_CART_PRODUCT_UNDEFINED

發生次數：12,540 次
受影響使用者：8,300 人
開始時間：2026-11-11 00:03
Release：web-2026.11.11.1
Route：/goods/GoodsDetail.jsp
Browser：Safari 18
Device：iPhone
Feature Flag：new-cart-api=true
```

團隊便可能發現：

```text
只有開啟 new-cart-api Feature Flag
且使用 Safari 的使用者發生錯誤。
```

這比工程師只看 `console.log()` 有效得多。

---

## 3.3 Next.js Error Tracking 範例

```typescript
import * as Sentry from "@sentry/nextjs";

type AddToCartInput = {
  productId: string;
  quantity: number;
};

export async function addToCart(input: AddToCartInput) {
  try {
    const response = await fetch("/api/cart/items", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(input),
    });

    if (!response.ok) {
      throw new Error(`Add to cart failed: ${response.status}`);
    }

    return await response.json();
  } catch (error) {
    Sentry.captureException(error, {
      tags: {
        domain: "cart",
        action: "add-to-cart",
      },
      extra: {
        productId: input.productId,
        quantity: input.quantity,
      },
    });

    throw error;
  }
}
```

Error Tracking 的價值不只是「把錯誤存起來」，而是讓錯誤可以依照以下維度分類：

```text
domain = cart
action = add-to-cart
release = web-2026.11.11.1
browser = Safari
country = TW
```

如此可以快速判斷錯誤影響範圍。

---

## 3.4 Source Map 的重要性

前端 production bundle 通常會被壓縮：

```javascript
function a(b){return b.c.d}
```

沒有 Source Map 時，錯誤可能只顯示：

```text
TypeError at app-83df8.js:1:38291
```

上傳 Source Map 後，Error Tracking 才能還原為：

```text
TypeError
at addToCart
src/features/cart/services/addToCart.ts:42
```

因此 momo 的 CI/CD Pipeline 應在發布時：

```text
Build Next.js
    │
    ▼
產生 production bundle
    │
    ▼
產生 Source Map
    │
    ▼
將 Source Map 上傳至 Error Tracking 平台
    │
    ▼
部署 production artifact
    │
    ▼
將 Release Version 與 Git Commit 綁定
```

Source Map 不一定要公開給瀏覽器下載，可以只上傳到受控的 Error Tracking 平台。

---

# 四、什麼是 Observability？

## 4.1 定義

Observability，中文常翻譯為「可觀測性」。

它是指：

> 透過系統輸出的資料，理解系統內部正在發生什麼事情。

Observability 的重點不是某一套工具，而是一種系統能力。

一個系統即使安裝了 Grafana、Sentry 或 ELK，也不代表它一定具備良好的 Observability。

真正的可觀測系統必須讓工程師能夠：

```text
從使用者問題
→ 找到前端請求
→ 找到 BFF request
→ 找到跨微服務呼叫
→ 找到資料庫或外部服務
→ 找到真正 Root Cause
```

---

## 4.2 Observability 的主要訊號

現代化 Observability 通常包含：

```text
Observability
├── Metrics
├── Logs
├── Traces
├── Errors
├── Events
├── Profiles
└── Real User Monitoring
```

最常見的是三大核心訊號：

```text
Metrics + Logs + Traces
```

---

# 五、Metrics、Logs 與 Traces

## 5.1 Metrics：系統整體趨勢

Metrics 是可以隨時間統計與聚合的數值。

例如：

```text
http_request_total = 25,000,000
checkout_error_rate = 2.3%
product_api_p95_latency = 780ms
redis_cache_hit_rate = 94%
kafka_consumer_lag = 12,000
```

Metrics 適合回答：

```text
系統是否異常？
異常從什麼時候開始？
影響範圍有多大？
錯誤率是否持續上升？
```

### momo Metrics 範例

```text
雙 11 00:00
商品 API RPS：80,000
P95 latency：1.8 秒
5xx error rate：3.2%
CPU：92%
DB connection pool：98%
```

從 Metrics 可以知道系統異常，但不一定知道是哪一筆請求造成。

---

## 5.2 Logs：特定事件的詳細紀錄

Logs 是系統執行過程產生的事件紀錄。

不建議只記錄：

```text
付款失敗
```

應使用 Structured Logging：

```json
{
  "timestamp": "2026-11-11T00:03:12.123+08:00",
  "level": "error",
  "service": "payment-service",
  "environment": "production",
  "traceId": "7ddfd893a124",
  "orderId": "ORD-928381",
  "paymentProvider": "LINE_PAY",
  "errorCode": "PROVIDER_TIMEOUT",
  "durationMs": 5032,
  "message": "Payment provider request timeout"
}
```

Structured Log 可以依欄位搜尋：

```text
service = payment-service
paymentProvider = LINE_PAY
errorCode = PROVIDER_TIMEOUT
```

但應注意：

```text
不要直接記錄
├── 信用卡號
├── CVV
├── 密碼
├── Access Token
├── 完整身分證號
├── 個人地址
└── 未遮罩的個人資料
```

---

## 5.3 Traces：追蹤一次完整請求

Trace 用來追蹤一個使用者請求如何經過不同系統。

假設使用者按下「送出訂單」：

```text
Browser
  │
  ▼
Next.js BFF
  │
  ├── Member Service
  ├── Cart Service
  ├── Pricing Service
  ├── Promotion Service
  ├── Inventory Service
  ├── Order Service
  └── Payment Service
```

一次完整請求稱為 Trace，每個處理階段稱為 Span。

```text
Trace：submit-order
│
├── Span：POST /api/checkout                  3,200ms
│
├── Span：member-service.validate               80ms
│
├── Span：cart-service.get-cart                 90ms
│
├── Span：pricing-service.calculate            130ms
│
├── Span：promotion-service.apply-coupon       240ms
│
├── Span：inventory-service.reserve-stock      420ms
│
├── Span：order-service.create-order           180ms
│
└── Span：payment-service.authorize          2,000ms
```

工程師可以立刻看出：

```text
整體結帳花費 3.2 秒，
其中支付服務就花了 2 秒。
```

---

# 六、momo 完整 Observability 架構

```text
┌─────────────────────────────────────────────────────────┐
│                       使用者端                           │
│                                                         │
│  momo Web／App／Mobile Web                              │
│                                                         │
│  蒐集：                                                  │
│  ├── Core Web Vitals                                    │
│  ├── JavaScript Error                                   │
│  ├── API latency                                        │
│  ├── User action                                        │
│  └── Session／Release／Feature Flag                      │
└──────────────────────────┬──────────────────────────────┘
                           │
                           │ traceparent / request-id
                           ▼
┌─────────────────────────────────────────────────────────┐
│                    CDN／WAF／Gateway                     │
│                                                         │
│  Metrics：request、bandwidth、cache hit、4xx、5xx        │
│  Logs：access log、security event                        │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                 Next.js Web／BFF Layer                   │
│                                                         │
│  Metrics：SSR latency、API latency、error rate           │
│  Logs：request log、business event                       │
│  Traces：前端 → BFF → 微服務                             │
│  Errors：SSR／Server Component／Route Handler error      │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     Domain Services                      │
│                                                         │
│ 商品 │ 搜尋 │ 價格 │ 庫存 │ 促銷 │ 購物車                │
│ 訂單 │ 會員 │ 支付 │ 推薦 │ 配送                         │
│                                                         │
│ 每個服務都輸出：                                         │
│ ├── Metrics                                              │
│ ├── Structured Logs                                     │
│ ├── Distributed Traces                                  │
│ └── Error Events                                        │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  Infrastructure Layer                    │
│                                                         │
│ Kubernetes／Redis／Kafka／Database／Object Storage       │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│              OpenTelemetry Collector                    │
│                                                         │
│      Receive → Process → Sample → Enrich → Export        │
└───────────────┬──────────────────┬──────────────────────┘
                │                  │
        ┌───────▼───────┐  ┌──────▼────────┐
        │ Metrics       │  │ Logs／Traces  │
        │ Prometheus    │  │ Loki／Elastic │
        └───────┬───────┘  │ Tempo／Jaeger │
                │          └──────┬────────┘
                └──────────┬──────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│             Grafana／APM／Error Tracking                 │
│                                                         │
│ Dashboard／Alert／Trace Analysis／Error Investigation    │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│             Slack／Teams／PagerDuty／On-call             │
└─────────────────────────────────────────────────────────┘
```

---

# 七、三者如何一起運作？

假設 momo 雙 11 當晚出現「結帳失敗」。

## 第一階段：Monitoring 發現問題

Dashboard 顯示：

```text
checkout_success_rate
99.5% → 91.2%

payment_api_p95_latency
500ms → 4,800ms

payment_api_error_rate
0.3% → 8.5%
```

告警條件觸發：

```yaml
alert: PaymentHighErrorRate

condition:
  error_rate > 5%
  for: 5 minutes

severity: critical
team: payment
```

Monitoring 回答：

```text
付款流程現在不正常。
```

---

## 第二階段：Error Tracking 找到應用程式錯誤

Error Tracking 顯示：

```text
PaymentTimeoutError

Release：
payment-service-2026.11.11.2

Affected Orders：
18,500

Provider：
某第三方支付服務

First Seen：
00:04

Last Seen：
持續發生
```

Error Tracking 回答：

```text
Payment Service 發生大量 timeout，
而且集中在剛發布的新版本。
```

---

## 第三階段：Trace 找到延遲路徑

```text
POST /checkout                       5,200ms
│
├── inventory.reserve                 150ms
├── order.create                      180ms
└── payment.authorize               4,700ms
    │
    └── external-provider.request   4,650ms
```

Tracing 回答：

```text
主要延遲來自第三方支付供應商。
```

---

## 第四階段：Logs 找到具體原因

```json
{
  "service": "payment-service",
  "traceId": "trace-928381",
  "provider": "external-payment-A",
  "errorCode": "CONNECTION_POOL_EXHAUSTED",
  "activeConnections": 200,
  "maxConnections": 200
}
```

Logs 回答：

```text
支付服務的 HTTP connection pool 已經耗盡。
```

---

## 第五階段：採取措施

```text
1. 暫時擴大 connection pool
2. 增加 Payment Service Pod
3. 對第三方支付加入 timeout 與 circuit breaker
4. 回滾造成連線未釋放的版本
5. 對使用者提供替代付款方式
6. 事後建立 SLO、Runbook 與壓力測試
```

完整流程：

```text
Monitoring
發現付款錯誤率上升
        │
        ▼
Error Tracking
找到 PaymentTimeoutError
        │
        ▼
Distributed Trace
定位外部支付請求花費 4.6 秒
        │
        ▼
Structured Logs
發現 connection pool exhausted
        │
        ▼
Root Cause
新版本未正確釋放 HTTP connection
        │
        ▼
Rollback + Scale Out + Circuit Breaker
```

---

# 八、前端 Observability 不只是追蹤 JavaScript Error

momo 的前端 Observability 至少應包含四個層次。

## 8.1 Technical Monitoring

監控技術層錯誤：

```text
JavaScript Error
ChunkLoadError
Hydration Error
API 4xx／5xx
圖片載入失敗
SSR Error
Server Component Error
```

## 8.2 Performance Monitoring

監控真實使用者效能：

```text
LCP
INP
CLS
TTFB
Route transition duration
API request latency
React component render duration
```

## 8.3 Business Monitoring

監控商業流程：

```text
商品曝光
商品點擊
加入購物車
套用折價券
開始結帳
付款成功
訂單成立
```

例如：

```text
商品頁流量正常
加入購物車 API 正常
但「加入購物車轉換率」下降 40%
```

這可能表示：

- 按鈕被版面遮住
- 新 UI 讓使用者找不到按鈕
- 某些瀏覽器 click handler 無效
- Feature Flag 只影響部分使用者
- 商品規格選擇流程出現 UX 問題

如果只監控伺服器 CPU，無法發現這類問題。

## 8.4 User Journey Monitoring

```text
首頁
  ↓
搜尋「iPhone」
  ↓
商品列表
  ↓
商品詳情
  ↓
選擇規格
  ↓
加入購物車
  ↓
套用優惠券
  ↓
結帳
  ↓
付款
  ↓
訂單成功
```

每一個階段都應有成功率、延遲與錯誤率。

---

# 九、Trace ID：串聯所有系統的關鍵

每個請求應建立唯一 Trace ID：

```text
traceId = 7ddfd893a124
```

這個 ID 應沿著請求傳遞：

```text
Browser
  │ traceId=7ddfd893a124
  ▼
Next.js BFF
  │ traceId=7ddfd893a124
  ▼
Cart Service
  │ traceId=7ddfd893a124
  ▼
Pricing Service
  │ traceId=7ddfd893a124
  ▼
Inventory Service
  │ traceId=7ddfd893a124
  ▼
Order Service
```

如此工程師才能：

```text
從前端 Error
→ 找到 BFF Log
→ 找到微服務 Trace
→ 找到 Database Query
→ 找到問題根因
```

若每個系統各自使用不同 ID，就會出現：

```text
前端知道出錯，
後端卻找不到對應請求。
```

---

# 十、Logging 的錯誤做法與正確做法

## 錯誤做法

```typescript
console.log("發生錯誤");
```

問題是：

- 不知道哪個服務
- 不知道哪個使用者操作
- 不知道是哪個訂單
- 不知道錯誤版本
- 無法依欄位搜尋
- 無法和 Trace 串聯

## 建議做法

```typescript
logger.error({
  event: "checkout_failed",
  service: "checkout-bff",
  traceId,
  orderId,
  paymentMethod: "credit-card",
  errorCode: "PAYMENT_TIMEOUT",
  durationMs: 5032,
  release: process.env.APP_VERSION,
});
```

但個資應遮罩：

```typescript
const safeUserId = hash(userId);
const maskedEmail = maskEmail(email);
```

不要直接記錄：

```typescript
logger.info({
  creditCardNumber,
  password,
  accessToken,
  fullAddress,
});
```

---

# 十一、Alerting 不等於「所有異常都通知」

告警太多會造成 Alert Fatigue。

工程師最後可能忽略真正嚴重的通知。

## 不佳的告警

```text
任一 API 發生一次 500
→ 立即通知所有工程師
```

一次錯誤可能只是短暫網路問題，不值得半夜叫醒 On-call。

## 較佳的告警

```text
當結帳成功率低於 98%
且持續 5 分鐘
且受影響訂單超過 500 筆
→ Critical Alert
```

告警應優先反映使用者受到的影響，而不是所有底層小波動。

## momo 告警分級

| 等級 | 情境 | 處理方式 |
|---|---|---|
| P1 | 全站無法結帳、付款服務中斷 | 立即通知 On-call 與 Incident Commander |
| P2 | 部分付款方式失敗、特定地區異常 | 服務團隊立即處理 |
| P3 | 單一功能錯誤率上升但有替代方案 | 工作時間內處理 |
| P4 | 非關鍵錯誤、效能改善項目 | 建立 Ticket 追蹤 |

---

# 十二、SLA、SLO 與 SLI

Observability 最終不應只看 CPU，而應和服務可靠度連結。

## SLI

實際量測指標：

```text
Checkout Success Rate
Payment API Availability
Product Page P95 Latency
```

## SLO

團隊內部設定的服務目標：

```text
每月結帳成功率 ≥ 99.9%
商品頁 P95 TTFB < 800ms
付款 API 可用率 ≥ 99.95%
```

## SLA

對外承諾，未達標可能涉及補償或合約責任：

```text
平台對合作廠商承諾每月服務可用率 99.9%
```

關係如下：

```text
Telemetry Data
      │
      ▼
SLI：目前實際表現
      │
      ▼
SLO：內部可靠度目標
      │
      ▼
SLA：外部服務承諾
```

---

# 十三、momo Monorepo 中的 Observability Shared Modules

如果 momo 採用 Monorepo，可以建立統一的 Observability 套件：

```text
momo-platform/
├── apps/
│   ├── main-web/
│   ├── checkout-web/
│   ├── seller-center/
│   └── campaign-web/
│
└── packages/
    ├── observability-browser/
    ├── observability-server/
    ├── logger/
    ├── metrics/
    ├── tracing/
    └── error-tracking/
```

例如：

```typescript
import {
  captureException,
  trackEvent,
  startSpan,
} from "@momo/observability-browser";
```

Server 端：

```typescript
import {
  logger,
  metrics,
  withTracing,
} from "@momo/observability-server";
```

優點：

- 統一錯誤格式
- 統一 Trace Context
- 統一 Release Tag
- 統一敏感資料遮罩
- 統一 Sampling 策略
- 統一 Dashboard Label
- 避免每個團隊自行定義不相容格式

---

# 十四、Shared Observability Module 範例

```typescript
type ErrorContext = {
  domain: "product" | "cart" | "checkout" | "payment";
  action: string;
  traceId?: string;
  release?: string;
  metadata?: Record<string, unknown>;
};

export function captureAppError(
  error: unknown,
  context: ErrorContext,
): void {
  const normalizedError =
    error instanceof Error ? error : new Error(String(error));

  console.error({
    event: "application_error",
    message: normalizedError.message,
    stack: normalizedError.stack,
    domain: context.domain,
    action: context.action,
    traceId: context.traceId,
    release: context.release,
    metadata: sanitize(context.metadata),
  });
}
```

使用方式：

```typescript
try {
  await submitOrder(orderPayload);
} catch (error) {
  captureAppError(error, {
    domain: "checkout",
    action: "submit-order",
    traceId,
    release: process.env.NEXT_PUBLIC_APP_VERSION,
    metadata: {
      paymentMethod,
      itemCount: orderPayload.items.length,
    },
  });

  throw error;
}
```

---

# 十五、大型促銷活動的 Observability 設計

雙 11 活動前：

```text
1. 建立活動專屬 Dashboard
2. 驗證所有服務都有 Trace
3. 設定商品、購物車、結帳、支付 SLO
4. 進行 Load Test 與 Stress Test
5. 檢查 Alert Rule
6. 建立 Runbook
7. 確認 On-call 排班
8. 設定 Feature Flag 與快速回滾能力
```

活動期間：

```text
即時監控
├── 線上人數
├── RPS
├── CDN Cache Hit Rate
├── 商品頁 latency
├── 加入購物車成功率
├── 結帳成功率
├── 付款成功率
├── Kafka consumer lag
├── DB connection usage
└── Kubernetes saturation
```

活動後：

```text
1. 分析錯誤與延遲
2. 檢查 SLO 是否達標
3. 分析 Error Budget
4. 進行 Incident Review
5. 更新 Runbook
6. 補充自動化測試
7. 調整容量預估模型
```

---

# 十六、常見工具如何分工？

| 類型 | 工具範例 | 主要用途 |
|---|---|---|
| Metrics | Prometheus | 蒐集與查詢時間序列指標 |
| Dashboard | Grafana | 建立監控圖表與告警 |
| Logs | ELK、OpenSearch、Loki | 集中儲存與搜尋 Log |
| Tracing | Jaeger、Tempo | 分散式請求追蹤 |
| Error Tracking | Sentry | 錯誤分組、Stack Trace、Release Tracking |
| Telemetry Standard | OpenTelemetry | 統一產生、收集與輸出遙測資料 |
| Alert Routing | Alertmanager | 告警分組、抑制、去重與通知 |
| Incident Management | PagerDuty、Opsgenie | On-call 與事故處理 |
| RUM | Sentry、Datadog 等 | 蒐集真實使用者效能與錯誤 |

實際採用哪一套產品不是最重要的。

最重要的是資料能否：

```text
統一格式
＋
跨系統關聯
＋
快速查詢
＋
對應服務負責團隊
＋
支援問題定位
```

---

# 十七、Observability 與傳統監控的根本差異

## 傳統監控思維

```text
CPU 有沒有超過 80%？
Server 是否還活著？
硬碟空間是否不足？
```

## 現代 Observability 思維

```text
為什麼只有 Safari 使用者無法加入購物車？

為什麼高雄地區的商品頁比台北慢？

哪一個版本造成結帳成功率下降？

哪一個 Feature Flag 影響付款流程？

這次延遲來自 Next.js SSR、BFF、Redis，
還是後端商品服務？

一筆訂單經過十個微服務時，
究竟在哪一個 Span 發生 timeout？
```

Monitoring 主要基於「已知的未知」：

```text
我們知道 CPU 過高可能有問題，
所以事先監控 CPU。
```

Observability 還需要處理「未知的未知」：

```text
我們事先不知道問題會以什麼形式出現，
但仍能透過 telemetry 交叉查詢找到原因。
```

---

# 十八、面試時可以怎麼回答？

## 精簡版回答

> Monitoring 是針對預先定義的系統指標進行持續監控與告警，例如 momo 在大型促銷期間監控流量、P95 latency、錯誤率與資源飽和度。
>
> Error Tracking 則專注在應用程式錯誤，包含 Stack Trace、Release、瀏覽器、使用者操作與影響範圍，例如追蹤 Next.js 加入購物車功能在特定版本發生的 JavaScript Error。
>
> Observability 的範圍更廣，它透過 Metrics、Logs 與 Distributed Traces，讓工程師能從前端一路追蹤到 BFF、微服務、Redis、Kafka 或資料庫，進一步理解系統為什麼出現異常。
>
> 對 momo 這類大型電商而言，真正重要的是把技術指標和商品瀏覽、加入購物車、結帳、付款等商業流程串聯，並透過 Trace ID、結構化 Log、SLO 與告警分級，在高流量期間快速定位問題及降低事故影響。

---

# 十九、最終總結

```text
Monitoring
＝ 系統有沒有異常？

Error Tracking
＝ 哪個錯誤、哪段程式、哪個版本出問題？

Observability
＝ 問題為什麼發生，並且如何跨越前端、BFF、
   微服務、資料庫與外部服務傳播？
```

momo 的現代化開發架構不應只做到：

```text
伺服器掛掉才通知工程師
```

而應做到：

```text
使用者體驗開始惡化
        │
        ▼
系統自動偵測
        │
        ▼
告警指出受影響的商業流程
        │
        ▼
Error Tracking 顯示錯誤版本與使用者環境
        │
        ▼
Trace 串聯前端、BFF 與微服務
        │
        ▼
Logs 提供詳細 Root Cause
        │
        ▼
團隊依 Runbook 回滾、擴容或降級
```

因此，Observability 的真正目的不是「收集更多資料」，而是：

> 在大型、分散式且高流量的電商系統中，縮短問題發現與修復時間，並將技術異常與真實使用者、訂單及營收影響連結起來。