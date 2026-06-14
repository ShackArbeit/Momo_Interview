# momo 大型電商中的「高流量」與「高併發」

## 一、先理解：高流量與高併發不是同一件事

在 momo 這類大型電商平台中，「高流量」與「高併發」經常同時發生，但兩者描述的是不同問題。

| 概念 | 高流量 High Traffic | 高併發 High Concurrency |
|---|---|---|
| 核心問題 | 一段時間內總共有多少流量進入系統 | 同一時間有多少請求正在被系統處理 |
| 常見指標 | PV、UV、Requests per Day、流量頻寬 | Concurrent Users、RPS、QPS、同時連線數 |
| 時間特性 | 可以分散在一天內 | 集中發生在幾秒或幾分鐘內 |
| 系統壓力 | 網路頻寬、CDN、圖片與靜態資源傳輸 | API、伺服器執行緒、連線池、資料庫與庫存鎖定 |
| momo 情境 | 一天有大量使用者瀏覽首頁、分類頁與商品頁 | 雙 11 零點時，大量使用者同時搶購同一商品 |

---

# 二、什麼是高流量？

## 2.1 定義

高流量是指：

> 在特定時間範圍內，有大量使用者、頁面瀏覽、API 請求及靜態資源下載進入平台。

例如，假設 momo 在雙 11 當天發生：

- 500 萬名使用者進站
- 每位使用者平均瀏覽 20 個頁面
- 每個頁面載入 30 個圖片、JavaScript、CSS 或 API 資源

那麼平台需要處理的不只是：

```text
500 萬次使用者造訪
```

而可能是：

```text
500 萬 × 20 頁 × 30 個資源
= 30 億次資源請求
```

這就是高流量問題。

## 2.2 高流量主要消耗什麼？

高流量主要消耗：

1. CDN 傳輸量
2. 網路頻寬
3. 圖片與影片傳輸量
4. Web Server 處理能力
5. API Gateway 吞吐量
6. Log、Analytics、Tracking 處理能力

## 2.3 momo 的高流量情境

```text
平常日
使用者 ────────────────> momo

雙 11、618、母親節、年中慶
使用者使用者使用者使用者 ──> momo
廣告導流使用者──────────> momo
推播導流使用者──────────> momo
直播導流使用者──────────> momo
搜尋引擎使用者──────────> momo
```

大型促銷期間，流量可能來自：

- Google 搜尋
- App Push
- LINE 官方帳號
- Facebook、Instagram 廣告
- momo 直播
- EDM
- 聯盟行銷
- 使用者直接進站

因此，前端不應讓每一個請求都直接打到 momo 的應用伺服器。

---

# 三、什麼是高併發？

## 3.1 定義

高併發是指：

> 在同一個時間區間內，有大量請求同時進入系統，而且這些請求尚未處理完成。

例如：

```text
10 萬名使用者在 10 分鐘內陸續瀏覽商品
```

這是高流量，但不一定是極端高併發。

相反地：

```text
5 萬名使用者在雙 11 的 00:00:00
同時按下「立即購買」
```

這就是高併發。

## 3.2 併發量的簡化理解

可以用下面的概念估算：

```text
同時進行中的請求數
≈ 每秒請求數 RPS × 平均回應時間
```

假設 momo API 每秒收到 10,000 個請求，平均每個請求需要 0.5 秒：

```text
Concurrency
≈ 10,000 × 0.5
≈ 5,000 個同時處理中的請求
```

如果 API 回應時間惡化到 3 秒：

```text
Concurrency
≈ 10,000 × 3
≈ 30,000 個同時處理中的請求
```

這表示：

> API 越慢，同一時間堆積的請求越多，系統越容易發生雪崩。

## 3.3 momo 常見的高併發操作

不是每個頁面操作都有相同風險。

| 使用者行為 | 併發風險 | 原因 |
|---|---:|---|
| 讀取首頁 Banner | 低 | 可以使用 CDN 快取 |
| 瀏覽分類頁 | 中 | 搜尋、篩選可能呼叫 API |
| 查看商品頁 | 中 | 價格、庫存可能需要動態查詢 |
| 商品搜尋 | 高 | 搜尋服務需要處理大量條件 |
| 加入購物車 | 高 | 需要寫入會員購物車資料 |
| 領取優惠券 | 高 | 優惠券有數量與資格限制 |
| 搶購限量商品 | 非常高 | 大量使用者競爭相同庫存 |
| 建立訂單 | 非常高 | 涉及庫存、價格、優惠與會員 |
| 支付 | 非常高 | 涉及金流、安全與交易一致性 |

---

# 四、高流量與高併發的差異範例

## 情境 A：高流量，但併發程度較低

一天內有 300 萬名使用者瀏覽 momo 商品頁，但流量平均分散在 24 小時。

```text
300 萬使用者
     │
     ├── 上午進站
     ├── 下午進站
     ├── 晚上進站
     └── 深夜進站
```

此時主要問題是：

- CDN 成本
- 圖片流量
- JavaScript Bundle 大小
- 首頁與商品頁載入速度
- 搜尋引擎爬蟲流量

## 情境 B：高流量且高併發

雙 11 零點有 10 萬名使用者同時重新整理頁面、領券、加入購物車及結帳。

```text
00:00:00
10 萬名使用者
    │
    ├── 重新整理活動頁
    ├── 查詢價格
    ├── 查詢庫存
    ├── 領取優惠券
    ├── 加入購物車
    └── 建立訂單
```

此時問題不只是流量大，而是所有人同時操作系統中最昂貴的功能。

---

# 五、momo 前端面對高流量的核心原則

前端在高流量架構中的目標，不只是「畫面載入快」，而是：

> 盡可能讓請求停留在瀏覽器與 CDN，減少請求穿透到後端核心服務。

整體架構可設計為：

```text
                    ┌──────────────────┐
                    │   momo 使用者     │
                    │ Web / App WebView│
                    └────────┬─────────┘
                             │
                    DNS / Traffic Routing
                             │
                 ┌───────────▼───────────┐
                 │ CDN / Edge / WAF      │
                 │                       │
                 │ HTML Cache            │
                 │ JS / CSS Cache        │
                 │ Image Optimization    │
                 │ Bot Protection        │
                 │ Rate Limiting         │
                 └───────────┬───────────┘
                             │
              CDN 無法直接回應時才進入系統
                             │
                 ┌───────────▼───────────┐
                 │ Next.js Web Layer     │
                 │                       │
                 │ SSG / ISR             │
                 │ SSR / Streaming       │
                 │ Server Components     │
                 │ Client Components     │
                 └───────────┬───────────┘
                             │
                      ┌──────▼──────┐
                      │ momo BFF    │
                      │ API 聚合    │
                      │ 權限與裁切   │
                      └──────┬──────┘
                             │
       ┌──────────────┬──────┼───────┬───────────────┐
       │              │      │       │               │
┌──────▼─────┐ ┌──────▼───┐ │ ┌─────▼──────┐ ┌─────▼─────┐
│商品服務     │ │搜尋服務   │ │ │促銷服務     │ │會員服務    │
└────────────┘ └──────────┘ │ └────────────┘ └───────────┘
                     ┌──────▼──────┐
                     │庫存／訂單服務│
                     └─────────────┘
```

---

# 六、momo 前端如何實踐高流量架構？

## 6.1 CDN：讓靜態資源不要進入應用伺服器

適合放入 CDN 的內容：

- JavaScript
- CSS
- 商品圖片
- 活動 Banner
- 字型
- Icon
- 不常變動的活動頁 HTML
- 商品頁的部分公開內容

```text
沒有 CDN：

Browser
   │
   └──── 每次都向 momo Origin Server 要圖片
                  │
                  └── Origin Server 負載持續增加


使用 CDN：

Browser
   │
   └──── 台灣邊緣 CDN 節點
              │
              ├── 命中快取：直接回傳
              │
              └── 未命中：才向 Origin 取一次
```

例如商品圖片可使用內容雜湊：

```text
/product/iphone-17.abc123.webp
```

並設定：

```http
Cache-Control: public, max-age=31536000, immutable
```

因為檔名內容改變時，Hash 也會改變，所以可以安全地長時間快取。

---

## 6.2 根據頁面特性選擇 Rendering Strategy

momo 不應把所有頁面都設計成 SSR，也不能把所有內容都做成 CSR。

| momo 頁面 | 建議策略 | 原因 |
|---|---|---|
| 品牌故事、購物說明 | SSG | 內容很少改變 |
| 活動 Landing Page | SSG／ISR | 高流量、更新頻率可控 |
| 商品分類頁 | ISR＋動態 API | SEO 內容可快取，篩選結果動態載入 |
| 商品頁 | ISR／Partial Prerendering | 商品描述可快取，價格庫存需動態 |
| 搜尋結果頁 | SSR 或 CSR | 搜尋條件高度動態 |
| 會員中心 | CSR／動態 SSR | 個人資料不能使用公共快取 |
| 購物車 | Client Component＋API | 高度個人化 |
| 結帳頁 | CSR／動態 Server Rendering | 即時價格、庫存與交易狀態 |

### 商品頁拆分範例

```text
商品頁
│
├── 可快取區域
│   ├── 商品名稱
│   ├── 商品描述
│   ├── 商品圖片
│   ├── 規格資訊
│   └── SEO Metadata
│
└── 動態區域
    ├── 即時價格
    ├── 即時庫存
    ├── 個人優惠券
    ├── 會員等級價格
    └── 購物車狀態
```

這樣使用者不需要等待所有個人化資料完成後，才看到商品頁。

---

## 6.3 使用 ISR，避免每次請求都重新產生頁面

假設 momo 有 100 萬個商品頁。

如果所有商品頁每次進站都 SSR：

```text
使用者請求
   │
   ├── 執行 Next.js Server
   ├── 呼叫商品 API
   ├── 呼叫價格 API
   ├── 產生 HTML
   └── 回傳頁面
```

高流量時，這會產生大量重複計算。

改成 ISR 後：

```text
第一位使用者
   │
   └── 產生商品 HTML → 儲存快取

後續大量使用者
   │
   └── 直接取得快取 HTML

資料到期
   │
   └── 背景重新產生新版 HTML
```

Next.js 概念範例：

```tsx
async function getProduct(productId: string) {
  const response = await fetch(
    `https://api.momo.example/products/${productId}`,
    {
      next: {
        revalidate: 300,
        tags: [`product:${productId}`],
      },
    },
  );

  if (!response.ok) {
    throw new Error("Failed to fetch product");
  }

  return response.json();
}
```

表示商品基礎資訊可以快取 300 秒，而不是每個使用者都重新呼叫商品服務。

---

## 6.4 使用 stale-while-revalidate

促銷活動頁不一定要在快取過期的一瞬間阻塞使用者。

可以設定：

```http
Cache-Control:
public,
s-maxage=60,
stale-while-revalidate=300
```

概念如下：

```text
0～60 秒
└── 使用新鮮快取

61～360 秒
├── 先把舊內容快速回傳給使用者
└── 背景更新快取

超過 360 秒
└── 重新向來源伺服器取得內容
```

這比「快取一過期，所有人同時回源」更安全。

否則可能發生 Cache Stampede：

```text
快取到期
   │
   ├── 使用者 A 回源
   ├── 使用者 B 回源
   ├── 使用者 C 回源
   ├── 使用者 D 回源
   └── 數萬名使用者一起回源
```

---

## 6.5 圖片最佳化

大型電商頁面最大的流量來源之一通常是圖片。

momo 前端可以採取：

- WebP／AVIF
- Responsive Images
- Lazy Loading
- CDN Image Resize
- 不同裝置使用不同圖片尺寸
- 首屏主圖 preload
- 非首屏圖片延遲載入
- 圖片固定寬高，避免版面位移

```tsx
<Image
  src={product.image}
  alt={product.name}
  width={640}
  height={640}
  sizes="
    (max-width: 768px) 50vw,
    (max-width: 1200px) 33vw,
    25vw
  "
/>
```

錯誤做法：

```text
手機只需要 320px 圖片
卻下載一張 3000px、4MB 的原始圖
```

正確做法：

```text
手機     → 320px WebP
平板     → 640px WebP
桌機     → 960px WebP
高解析度 → 依 DPR 提供適當尺寸
```

---

## 6.6 JavaScript Bundle 拆分

momo 首頁可能包含：

- Header
- 搜尋框
- 商品推薦
- 廣告輪播
- 直播
- 瀏覽紀錄
- 優惠券
- 會員資訊
- 推薦演算法模組

這些功能不應全部放在第一個 JavaScript Bundle。

```text
初始載入
│
├── Header
├── Search
├── 首屏 Banner
└── 首屏商品

使用者往下捲動後
│
├── 載入直播模組
├── 載入推薦模組
└── 載入瀏覽紀錄
```

Next.js 範例：

```tsx
import dynamic from "next/dynamic";

const LiveShopping = dynamic(
  () => import("./LiveShopping"),
  {
    loading: () => <LiveShoppingSkeleton />,
    ssr: false,
  },
);
```

但不能把所有元件都設成 `ssr: false`，否則可能傷害：

- SEO
- 首屏速度
- 弱網路體驗
- 低階手機效能

---

# 七、momo 前端如何實踐高併發架構？

## 7.1 防止重複請求

使用者快速連點「加入購物車」時，前端應避免送出十次請求。

```tsx
function AddToCartButton({ productId }: { productId: string }) {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const addToCart = async () => {
    if (isSubmitting) return;

    setIsSubmitting(true);

    try {
      await fetch("/api/cart/items", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Idempotency-Key": crypto.randomUUID(),
        },
        body: JSON.stringify({
          productId,
          quantity: 1,
        }),
      });
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <button
      disabled={isSubmitting}
      onClick={addToCart}
    >
      {isSubmitting ? "加入中..." : "加入購物車"}
    </button>
  );
}
```

這裡包含兩層防護：

1. 前端按鈕送出後暫時 Disable
2. 使用 `Idempotency-Key`，讓後端辨識重複交易

但必須注意：

> 前端 Disable 只能改善 UX，真正的重複交易防護仍必須由後端保證。

---

## 7.2 Debounce：降低搜尋請求數量

假設使用者輸入：

```text
i → ip → iph → ipho → iphone
```

若每輸入一個字就呼叫搜尋 API，會產生五次請求。

使用 Debounce 後：

```text
使用者輸入 iphone
        │
        ├── 等待 300ms
        └── 使用者停止輸入後才送出一次請求
```

React 範例：

```tsx
useEffect(() => {
  const timer = window.setTimeout(() => {
    if (keyword.trim().length >= 2) {
      searchProducts(keyword);
    }
  }, 300);

  return () => {
    window.clearTimeout(timer);
  };
}, [keyword]);
```

---

## 7.3 AbortController：取消過期請求

使用者先搜尋：

```text
iphone
```

接著立刻改成：

```text
iphone case
```

舊請求可能比新請求更晚回來，導致畫面顯示錯誤結果。

```tsx
useEffect(() => {
  const controller = new AbortController();

  async function search() {
    const response = await fetch(
      `/api/search?q=${encodeURIComponent(keyword)}`,
      {
        signal: controller.signal,
      },
    );

    const data = await response.json();
    setProducts(data.items);
  }

  search().catch((error) => {
    if (error.name !== "AbortError") {
      throw error;
    }
  });

  return () => controller.abort();
}, [keyword]);
```

這不只能防止畫面競態，也能降低不必要的 API 工作。

---

## 7.4 Request Deduplication

同一個商品頁中的多個元件可能都需要商品資料：

```text
ProductTitle
ProductGallery
ProductSpecification
Recommendation
SEO Metadata
```

錯誤設計：

```text
五個元件各自呼叫一次 Product API
= 一次瀏覽產生五個相同請求
```

較好的設計：

```text
Server Component／BFF 呼叫一次
              │
              ├── ProductTitle
              ├── ProductGallery
              ├── ProductSpecification
              └── SEO Metadata
```

也可以使用 React Query／SWR 的 cache key 共用結果：

```tsx
const { data } = useQuery({
  queryKey: ["product", productId],
  queryFn: () => fetchProduct(productId),
  staleTime: 30_000,
});
```

---

## 7.5 BFF 聚合 API

沒有 BFF 時，商品頁可能由 Browser 呼叫：

```text
Browser
├── Product API
├── Pricing API
├── Inventory API
├── Promotion API
├── Review API
├── Member API
└── Recommendation API
```

如果一個頁面要呼叫 7 個 API，10 萬人同時瀏覽，就可能形成大量對外請求。

使用 BFF 後：

```text
Browser
   │
   └── GET /api/product-page/123
              │
              └── momo BFF
                   ├── Product Service
                   ├── Pricing Service
                   ├── Inventory Service
                   ├── Promotion Service
                   ├── Review Service
                   └── Recommendation Service
```

BFF 可以：

- 聚合 API
- 裁切欄位
- 統一錯誤格式
- 統一 Timeout
- 進行快取
- 隱藏內部服務位址
- 降低 Browser 與後端服務的耦合

但要注意：

> BFF 不是把所有資料都塞成一支巨大 API，而是依頁面與使用情境合理聚合。

---

## 7.6 Progressive Loading 與 Streaming

商品頁不應等待所有服務成功後才顯示。

```text
第一階段
├── 商品名稱
├── 商品圖片
└── 商品描述

第二階段
├── 價格
├── 庫存
└── 促銷資訊

第三階段
├── 推薦商品
├── 評論
└── 瀏覽紀錄
```

React Suspense 範例：

```tsx
export default function ProductPage() {
  return (
    <>
      <ProductBasicInfo />

      <Suspense fallback={<PriceSkeleton />}>
        <ProductPrice />
      </Suspense>

      <Suspense fallback={<PromotionSkeleton />}>
        <PromotionPanel />
      </Suspense>

      <Suspense fallback={<RecommendationSkeleton />}>
        <RecommendationList />
      </Suspense>
    </>
  );
}
```

即使推薦服務暫時變慢，也不應阻塞：

- 商品資訊
- 加入購物車
- 結帳流程

---

# 八、雙 11 搶購場景的完整架構

假設 momo 推出：

```text
iPhone 限量 1,000 台
雙 11 00:00 開賣
同時有 10 萬人搶購
```

## 8.1 架構流程

```text
                    10 萬名使用者
                          │
              ┌───────────▼───────────┐
              │ CDN / WAF / Edge      │
              │                       │
              │ 快取活動頁             │
              │ Bot Detection         │
              │ IP Rate Limit         │
              │ Device Fingerprint    │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ Virtual Waiting Room  │
              │ 排隊／Token 驗證       │
              └───────────┬───────────┘
                          │
                   合法流量逐步放行
                          │
              ┌───────────▼───────────┐
              │ Next.js / momo BFF    │
              │                       │
              │ 防重複提交             │
              │ Idempotency Key       │
              │ Timeout               │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ Purchase API          │
              │                       │
              │ MQ / Queue            │
              │ Atomic Inventory      │
              │ Distributed Lock      │
              └───────────┬───────────┘
                          │
              ┌───────────▼───────────┐
              │ Order / Payment       │
              └───────────────────────┘
```

## 8.2 前端在搶購中的責任

前端可以負責：

- 顯示倒數時間
- 避免重複點擊
- 傳送排隊 Token
- 顯示等待狀態
- 處理 Timeout
- 處理庫存售罄
- 處理重試提示
- 保留使用者操作狀態
- 避免整頁無限重新整理

前端不能負責：

- 最終庫存扣減
- 判斷誰真正買到商品
- 保證不超賣
- 保證優惠券不超發
- 保證訂單唯一性
- 保證付款一致性

這些必須由後端與資料庫保證。

## 8.3 前端狀態機

搶購按鈕不應只有：

```text
可點擊 / 不可點擊
```

而應設計成完整狀態：

```text
IDLE
  │
  ▼
WAITING_ROOM
  │
  ▼
SUBMITTING
  │
  ├── SUCCESS
  ├── SOLD_OUT
  ├── RATE_LIMITED
  ├── TOKEN_EXPIRED
  ├── TIMEOUT
  └── FAILED
```

例如：

```ts
type PurchaseState =
  | "idle"
  | "waiting-room"
  | "submitting"
  | "success"
  | "sold-out"
  | "rate-limited"
  | "token-expired"
  | "timeout"
  | "failed";
```

這樣才能針對不同錯誤提供正確 UX，而不是一律顯示：

```text
系統錯誤，請稍後再試
```

---

# 九、前端必須實作的降級策略

高併發期間，目標不一定是讓所有功能都正常，而是：

> 保住「搜尋商品 → 查看商品 → 加入購物車 → 結帳」這條核心交易路徑。

## 9.1 功能優先級

```text
P0：絕對不能壞
├── 商品頁
├── 價格
├── 庫存
├── 購物車
├── 訂單
└── 支付

P1：可以稍慢
├── 搜尋
├── 優惠券
└── 評論

P2：必要時可關閉
├── 個人化推薦
├── 直播
├── 動態動畫
├── 瀏覽紀錄
└── 非核心追蹤工具
```

## 9.2 Feature Flag

```tsx
{featureFlags.enableRecommendation ? (
  <RecommendationList />
) : null}
```

促銷高峰時若推薦服務壓力過大，可透過 Remote Config 關閉，而不必重新部署整個網站。

## 9.3 錯誤隔離

```tsx
<ProductErrorBoundary>
  <ProductInfo />
</ProductErrorBoundary>

<RecommendationErrorBoundary fallback={null}>
  <RecommendationList />
</RecommendationErrorBoundary>
```

推薦模組壞掉，不應造成整個商品頁白畫面。

## 9.4 API Fallback

```text
即時推薦 API 失敗
       │
       ├── 顯示快取推薦
       │
       ├── 顯示熱門商品
       │
       └── 隱藏推薦區塊
```

---

# 十、前端效能指標與監控

高流量系統不能只看 Server CPU，也必須觀察真實使用者體驗。

## 10.1 Core Web Vitals

| 指標 | 說明 | momo 範例 |
|---|---|---|
| LCP | 最大內容出現時間 | 商品主圖或首頁 Banner 何時顯示 |
| INP | 使用者操作回應速度 | 點擊加入購物車後多久有反應 |
| CLS | 畫面穩定性 | 優惠 Banner 是否突然插入造成位移 |

## 10.2 技術指標

前端應監控：

- JavaScript Error Rate
- API Error Rate
- API P95／P99 Latency
- CDN Cache Hit Ratio
- SSR Response Time
- Hydration Error
- Chunk Load Error
- 白畫面比例
- 搜尋成功率
- 加入購物車成功率
- 結帳成功率

## 10.3 商業指標

```text
頁面速度
   ↓
商品瀏覽率
   ↓
加入購物車率
   ↓
結帳完成率
   ↓
GMV
```

因此不能只說：

```text
LCP 從 3 秒改善到 2 秒
```

還應分析：

```text
LCP 改善後
├── 跳出率是否下降？
├── 商品點擊率是否提高？
├── 加入購物車率是否提高？
└── 成交率是否提高？
```

---

# 十一、常見錯誤設計

## 錯誤一：所有頁面都使用 SSR

```text
每個使用者
   └── 每次都重新產生 HTML
```

問題：

- Server 壓力大
- API 重複查詢
- 流量高峰容易塞車
- TTFB 變長

應依頁面特性混用：

```text
SSG + ISR + SSR + CSR + Streaming
```

## 錯誤二：所有資料都放在 Client Fetch

問題：

- SEO 不佳
- 首屏需要等待 JavaScript
- API Request 數量增加
- 容易產生 Loading Waterfall

```text
HTML
  ↓
下載 JavaScript
  ↓
Hydration
  ↓
呼叫 API
  ↓
顯示內容
```

## 錯誤三：所有 API 都設定 no-cache

價格與庫存需要即時，不代表：

- 商品描述
- 商品圖片
- 品牌資訊
- 規格資料
- 分類資訊

都不能快取。

應針對資料特性分級：

| 資料 | 快取策略 |
|---|---|
| JS、CSS、Hash 圖片 | 一年或長效快取 |
| 商品描述 | 分鐘到小時 |
| 分類資料 | 分鐘級 |
| 活動頁 | 秒到分鐘級 |
| 搜尋結果 | 短時間快取 |
| 價格 | 極短快取或即時查詢 |
| 庫存 | 即時或近即時 |
| 會員資料 | Private Cache／不使用共享快取 |
| 購物車 | 不使用公共快取 |

## 錯誤四：前端顯示「剩餘一件」就認為一定買得到

前端看到的庫存只是某個時間點的快照。

```text
使用者 A：看到剩餘 1 件
使用者 B：看到剩餘 1 件
使用者 C：看到剩餘 1 件
```

最後只能由後端原子操作決定誰成功。

## 錯誤五：API 失敗後無限制自動重試

```text
後端過載
   │
   ├── 前端第一次失敗
   ├── 立即重試
   ├── 再次失敗
   ├── 再次重試
   └── 形成 Retry Storm
```

應採用：

- 限制重試次數
- Exponential Backoff
- Random Jitter
- 非必要 API 不重試
- 交易 API 不可盲目重試

---

# 十二、面試時可以怎麼回答？

## 精簡回答版本

高流量與高併發並不完全相同。

高流量代表一段時間內有大量使用者、頁面瀏覽與資源請求；高併發則代表同一時間有大量尚未完成的請求。以 momo 為例，一整天有數百萬人瀏覽商品屬於高流量，而雙 11 零點大量使用者同時領券、搶購及結帳，則是高併發。

前端處理高流量的核心，是利用 CDN、HTTP Cache、圖片最佳化、Code Splitting，以及 Next.js 的 SSG、ISR、SSR 與 Streaming，讓大部分內容在瀏覽器或 Edge 層完成，不要讓每個請求都穿透到後端。

面對高併發，前端需要使用 Debounce、AbortController、Request Deduplication、防重複提交、Idempotency Key、等待室、錯誤隔離與功能降級，減少無效請求並保護核心交易流程。

但庫存扣減、訂單唯一性與防止超賣，不能依靠前端，仍然必須由後端、訊息佇列、資料庫交易或原子操作保證。

## 一句話總結

> 高流量處理的是「很多請求如何有效傳輸」，高併發處理的是「大量請求同時發生時，系統如何不被壓垮」；momo 前端的責任，是透過快取、分層渲染、請求控制與降級機制，減少後端壓力並維持核心購物體驗。