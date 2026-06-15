# 以台灣 momo 為例說明 Observability、Monitoring 與 Error Tracking

在大型電商公司（例如 momo）中，每天可能有數百萬次商品瀏覽、數十萬次搜尋、數萬筆訂單交易。

當雙11、618、黑色星期五等大型活動來臨時：

- 同時在線使用者暴增
- API 請求量暴增
- 商品搜尋流量暴增
- 第三方金流、物流壓力增加

此時最大的問題不是「系統會不會出錯」

而是：

> 當出錯時，你能不能在 1 分鐘內發現問題？
>
> 能不能在 5 分鐘內找到原因？
>
> 能不能在 10 分鐘內恢復服務？

這就是現代化開發中的：

1. Monitoring（監控）
2. Error Tracking（錯誤追蹤）
3. Observability（可觀測性）

誕生的原因。

---

# 一、Monitoring 是什麼？

## 定義

Monitoring（監控）

是透過預先定義好的指標持續監看系統狀態。

簡單說：

> 「告訴你系統有沒有出事」

---

## momo 範例

假設 momo 首頁：

```text
https://www.momoshop.com.tw
```

前端工程師會監控：

| 指標 | 說明 |
|--------|--------|
| Page Load Time | 首頁載入速度 |
| API Response Time | API回應時間 |
| Error Rate | 錯誤率 |
| CPU Usage | Server CPU |
| Memory Usage | Server記憶體 |
| Order Success Rate | 訂單成功率 |
| Search Success Rate | 搜尋成功率 |

```

### 架構

```text
Frontend
    │
    ▼
Collect Metrics
    │
    ▼
Prometheus
    │
    ▼
Grafana Dashboard
    │
    ▼
Alert
```

---

## 例子

正常：

```text
API Response Time

200ms
180ms
250ms
220ms
```

異常：

```text
API Response Time

200ms
300ms
800ms
5000ms
```

Grafana：

```text
🚨 Alert

Search API > 3000ms
```

工程師立刻收到通知。

---

# 二、Error Tracking 是什麼？

## 定義

Error Tracking

專門追蹤程式錯誤（Exception）。

如果 Monitoring 是：

```text
知道有人生病
```

Error Tracking 則是：

```text
知道病人得了什麼病
```

---

# momo 前端常見錯誤

例如：

```javascript
product.price.toFixed(2)
```

但 API 回傳：

```javascript
product.price = null
```

就會出現：

```javascript
TypeError:
Cannot read properties of null
```

---

## 使用 Sentry

```javascript
Sentry.captureException(error)
```

發生錯誤後：

```text
Sentry Dashboard

Error:
Cannot read properties of null

User:
123456

Browser:
Chrome 137

URL:
/product/123

Release:
v2026.06.15
```

工程師立即知道：

```text
哪個頁面
哪位使用者
哪個版本
哪段程式
```

出問題。

---

## momo Error Tracking 架構

```text
Browser
    │
    ▼
JavaScript Error
    │
    ▼
Sentry SDK
    │
    ▼
Sentry Server
    │
    ▼
Slack Notification
```

---

# 三、Observability 是什麼？

## 定義

Observability（可觀測性）

不是只知道：

```text
系統壞了
```

而是：

```text
系統為什麼壞？
```

以及：

```text
壞在哪裡？
```

```text
誰造成的？
```

```text
哪個服務造成的？
```

```text
哪個 API 慢？
```

```text
哪個資料庫查詢卡住？
```

Observability 的目標是：

> 從系統輸出的資料推斷整個系統內部狀態。 :contentReference[oaicite:0]{index=0}

---

# Observability 三大支柱

```text
Metrics
Logs
Traces
```

---

## 1. Metrics

數值型監控資料

例如：

```text
CPU
Memory
Request Count
Response Time
```

---

momo Search API

```text
Requests/sec

500
800
1200
3000
```

可以知道：

```text
流量是否暴增
```

---

## 2. Logs

系統日誌

```javascript
logger.info(
  "User add to cart",
  {
      productId:123,
      userId:888
  }
)
```

產生：

```text
2026-06-15

User 888
Add Product 123
```

可以還原事件經過。

---

## 3. Traces

最重要也是面試最常考。

Trace 可以追蹤：

```text
一個請求
經過哪些服務
```

---

# momo 下單流程

```text
User
 │
 ▼
Frontend
 │
 ▼
API Gateway
 │
 ├─ Product Service
 │
 ├─ Inventory Service
 │
 ├─ Coupon Service
 │
 ├─ Payment Service
 │
 └─ Order Service
```

---

下單請求：

```text
Order #A12345
```

Trace：

```text
Frontend
 120ms

Product Service
 50ms

Inventory Service
 30ms

Coupon Service
 40ms

Payment Service
 4200ms  ❌

Order Service
 20ms
```

馬上知道：

```text
金流服務變慢
```

而不是：

```text
整個系統都在猜
```

Distributed Tracing 的核心價值就是追蹤單一請求穿越多個服務的完整路徑。 :contentReference[oaicite:1]{index=1}

---

# 四、Monitoring、Error Tracking、Observability 差異

| 項目 | Monitoring | Error Tracking | Observability |
|--------|--------|--------|--------|
| 目的 | 發現異常 | 發現程式錯誤 | 找出根因 |
| 回答問題 | 是否有問題 | 發生什麼錯 | 為什麼發生 |
| 資料來源 | Metrics | Exceptions | Metrics + Logs + Traces |
| 使用者 | DevOps | Frontend / Backend | 全團隊 |
| 工具 | Grafana | Sentry | Datadog、New Relic、OpenTelemetry |

---

# 五、momo 現代化前端架構實踐

```text
User Browser
      │
      ▼
 Next.js Frontend
      │
      ├──── Web Vitals
      │
      ├──── Error Tracking
      │
      ├──── User Session Replay
      │
      └──── Frontend Logs
                │
                ▼
         Observability Platform
                │
      ┌─────────┼─────────┐
      ▼         ▼         ▼
   Metrics    Logs     Traces
      │         │         │
      └─────────┼─────────┘
                ▼
          Root Cause Analysis
```

---

# 六、如果你去面試 momo 前端

面試官非常可能問：

### Q1

什麼是 Monitoring？

回答：

```text
利用預先定義的 Metrics 持續監控系統健康狀況，
例如 API Response Time、CPU Usage、
Error Rate 等。
```

---

### Q2

什麼是 Observability？

回答：

```text
Observability 不只是知道系統有問題，
而是透過 Metrics、Logs、Traces
理解系統內部狀態，
並快速定位根因。
```

---

### Q3

為什麼大型電商需要 Trace？

回答：

```text
momo 的一次下單可能經過
商品、庫存、優惠券、訂單、
金流等多個微服務。

當延遲發生時，
Distributed Trace 能快速找到
是哪個服務造成問題。
```

---

### Q4

前端能做哪些 Observability？

回答：

```text
1. Web Vitals
2. Error Tracking (Sentry)
3. User Session Replay
4. API Latency Monitoring
5. Custom Business Metrics
6. OpenTelemetry Tracing
```

---

# 面試級總結

```text
Monitoring
=
知道有沒有問題

Error Tracking
=
知道哪裡出錯

Observability
=
知道為什麼出錯
並快速找到根因
```

在 momo 這種高流量電商平台中：

Metrics + Logs + Traces

已經是現代化前端與微服務架構的標準配備。

真正成熟的團隊追求的不是：

「系統不要出錯」

而是：

「系統出錯時，能在最短時間找到原因並恢復服務」