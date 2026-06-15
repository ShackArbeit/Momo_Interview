# momo 前端工程師必懂：什麼是高流量（High Traffic）與高併發（High Concurrency）

## 一、什麼是高流量（High Traffic）？

高流量指的是：

> 在一段時間內，有大量使用者持續訪問系統。

以 momo 為例：

- 平常日晚上 8 點
- 雙11
- 雙12
- 618 購物節
- Apple 新機開賣
- PS6 預購
- Nintendo Switch 新機預購

這些時段都可能產生巨量流量。

---

### 高流量範例

假設：

- momo 每日活躍使用者 (DAU)：300萬
- 晚上 8~10 點為尖峰

此時可能出現：

| 指標 | 數值 |
|--------|--------|
| Page View | 5000萬/日 |
| API Request | 10億次/日 |
| 商品頁瀏覽 | 每秒數萬次 |
| APP + Web 同時在線 | 數十萬人 |

這就是典型的 High Traffic。

---

## 二、什麼是高併發（High Concurrency）？

很多人誤會：

高流量 ≠ 高併發

---

### 高併發定義

高併發指的是：

> 同一時間有大量使用者同時操作系統。

例如：

今天晚上 00:00

iPhone 18 Pro 開賣

有：

- 10 萬人同時進商品頁
- 5 萬人同時加入購物車
- 3 萬人同時結帳

此時：

系統每秒可能收到：

```text
20,000 ~ 50,000 Requests/sec
```

這叫：

```text
High Concurrency
```

---

## momo 電商案例

### 高流量

```text
100萬人
在一天內陸續進站
```

---

### 高併發

```text
10萬人
在同一秒按下購買
```

---

## 三、為什麼 momo 特別重視高流量與高併發？

momo 是台灣 B2C 電商龍頭之一

- 超過千萬會員
- 大量 APP 使用者
- 大型檔期活動頻繁

官方資料提到：

- 超過千萬會員
- 大型電商平台
- 海量資料與高流量系統場景

因此前端架構不能只考慮功能開發，而必須考慮效能與可擴展性。 :contentReference[oaicite:0]{index=0}

---

# 四、momo 前端如何面對高流量？

---

## 1. CDN

### 問題

如果所有圖片都從台北機房下載：

```text
User
 ↓
Taipei Server
```

10萬人同時下載圖片：

```text
頻寬爆掉
```

---

### 解法

使用 CDN

```text
                 CDN
              /   |   \
             /    |    \
            /     |     \
         User  User  User
              ↓
         Origin Server
```

---

### 效果

- 圖片快取
- JS 快取
- CSS 快取
- 降低 Origin 壓力

---

## 2. Code Splitting

大型電商常有：

```text
首頁
搜尋
商品頁
購物車
結帳
會員中心
```

若全部打包：

```text
main.js

10MB
```

首次載入會超慢。

---

### React / Next.js

```javascript
const CartPage = dynamic(() =>
  import('./CartPage')
);
```

使用者進購物車才下載：

```text
Cart Bundle
```

---

### 效果

首屏更快

LCP 改善

---

## 3. Lazy Loading

商品列表：

```text
1000 個商品
```

若一次載入：

```text
1000 張圖片
```

非常浪費。

---

### 實作

```html
<img
  loading="lazy"
  src="product.jpg"
/>
```

---

### 效果

只下載可視範圍圖片。

```text
Viewport
 ↓
20 張圖片

其餘不下載
```

---

## 4. Virtual List

例如搜尋結果：

```text
5000 商品
```

如果全部 Render：

```text
5000 DOM
```

React 會卡死。

---

### 解法

```text
react-window
react-virtualized
```

只 Render：

```text
目前畫面 20 筆
```

---

架構：

```text
5000 Items

 ↓

Render

20 Items
```

---

# 五、momo 前端如何面對高併發？

這才是面試官最愛問的。

---

## 場景一

iPhone 開賣

10萬人同時進商品頁

---

### 前端策略

避免：

```text
每次操作都打 API
```

---

### 錯誤做法

```text
進頁面
↓
打庫存 API

切顏色
↓
再打一次

切容量
↓
再打一次
```

造成：

```text
API Storm
```

---

### 正確做法

一次取得：

```json
{
  "stock": {
    "black-128": 100,
    "black-256": 80,
    "white-128": 60
  }
}
```

前端自行切換。

---

## 場景二

秒殺活動

---

### 問題

所有人同時刷新

```text
F5
F5
F5
F5
F5
```

API 爆掉。

---

### 解法

前端 Cache

```text
React Query
SWR
```

---

架構：

```text
User

 ↓

React Query Cache

 ↓

API
```

---

同一秒：

```text
100 次需求
```

實際：

```text
1 次 API
```

---

## 場景三

購物車數量

---

### 不良做法

```text
+1
↓
API

+1
↓
API

+1
↓
API
```

---

### Optimistic UI

```javascript
setCount(count + 1);
```

立即更新畫面。

背景同步：

```text
UI 更新
 ↓
API 同步
```

---

使用者體感：

```text
即時
```

而不是：

```text
等待 500ms
```

---

# 六、momo 的前端架構可能長什麼樣？

（面試可直接回答）

```text
                  User
                    │
                    ▼
             CloudFront CDN
                    │
                    ▼
            Next.js SSR Server
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼

 Product API   Search API   Member API

      │             │             │

      ▼             ▼             ▼

 Redis Cache   ElasticSearch   Database
```

---

## 前端在整個架構中的角色

很多初階工程師認為：

```text
前端 = 畫畫面
```

其實錯了。

在 momo 這種等級的電商：

前端是：

```text
流量第一道防線
```

---

前端負責：

### SEO

- SSR
- SSG
- Metadata
- Structured Data

---

### 效能

- CDN
- Lazy Load
- Virtual List
- Code Split

---

### 降低後端壓力

- Cache
- React Query
- SWR
- Debounce
- Throttle

---

### 提升轉換率

- 首屏速度
- 購物流程
- Checkout UX

---

# 面試一句話總結

在 momo 這類大型電商中：

高流量是指大量使用者持續訪問系統，而高併發則是大量使用者在同一時間操作系統。前端工程師的核心責任不只是開發功能，而是透過 CDN、SSR、Code Splitting、Lazy Loading、快取策略、Virtualization 與 Optimistic UI 等技術，在大型促銷活動與秒殺場景下，降低後端壓力、提升使用者體驗，並確保系統在高流量與高併發環境下仍能穩定運作。