# Momo 電商：Next.js Component 對應列表

## 1. Next.js Web／BFF Layer 與各服務的 Component 對應

> Product Service、Search Service 等本身是後端服務，不是 React Component。  
> 下表表示在 Next.js 前端中，通常由哪一種 Component 負責取得資料或提供互動。

| Service | 中文名稱 | 主要對應 Component | 實際用途 |
|---|---|---|---|
| Product Service | 商品服務 | Server Component 為主，Client Component 輔助 | Server Component 取得商品名稱、描述、規格與 SEO 資料；Client Component 處理 SKU、顏色、尺寸等互動 |
| Search Service | 搜尋服務 | Server Component＋Client Component | Server Component 取得初始搜尋結果；Client Component 處理搜尋框、Autocomplete、篩選與排序互動 |
| Pricing Service | 價格服務 | Server Component 為主，Client Component 輔助 | Server Component 取得初始與最終可信價格；Client Component 顯示規格切換後的價格變化 |
| Inventory Service | 庫存服務 | Server Component＋Client Component | Server Component 取得初始庫存；Client Component 在切換 SKU 時更新即時庫存狀態 |
| Promotion Service | 促銷服務 | Server Component 為主，Client Component 輔助 | Server Component 取得並計算促銷結果；Client Component 處理優惠券、贈品與活動選擇 |
| Cart Service | 購物車服務 | Server Component＋Client Component | Server Component 取得真實購物車內容；Client Component 處理數量、勾選、刪除與 Optimistic UI |
| Order Service | 訂單服務 | Server Component 為主，Client Component 輔助 | Server Component 取得訂單列表與詳情；Client Component 處理取消、退貨與查詢條件 |
| Member Service | 會員服務 | Server Component＋Client Component | Server Component 取得會員、點數、地址與權限資料；Client Component 處理會員資料編輯與表單互動 |
| Payment Service | 支付服務 | Server-side 為主，Client Component 輔助 | Server 處理交易、金額與付款驗證；Client Component 只處理付款方式選擇、支付 SDK 與導頁 |
| Recommendation Service | 推薦服務 | Server Component＋Client Component | Server Component 取得初始推薦結果；Client Component 處理輪播、載入更多、曝光與點擊追蹤 |

## 2. Server Component、Client Component、SSR、CSR 比較

| 項目 | Server Component | Client Component | SSR | CSR |
|---|---|---|---|---|
| 本質 | Component 類型 | Component 類型 | Rendering 策略 | Rendering 策略 |
| 主要執行地點 | Server | Server 預渲染＋Browser 執行 | Server | Browser |
| 是否送 JS 到 Browser | 元件本身不送 | 會 | 視 Component 而定 | 會 |
| 是否能用 `useState` | 不行 | 可以 | 不一定 | 可以 |
| 是否能用 `onClick` | 不行 | 可以 | 不一定 | 可以 |
| 是否可直接存取 DB | 可以，但大型系統通常經服務層 | 不可 | Server 可處理 | 不應直接處理 |
| 是否需要 hydration | 不需要 | 需要互動時需要 | 不一定 | 通常需要 |
| SEO | 很適合 | 可搭配 SSR | 通常較好 | 純 CSR 較不理想 |
| 典型用途 | 資料取得、內容、權限 | 互動、狀態、Browser API | 首頁、商品頁、動態內容 | 個人化、互動式內容 |
