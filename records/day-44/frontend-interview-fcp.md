# Day 44 - 前端面試：如何優化首次載入時間（FCP）

Date: 2026-05-14

## 一句話版本

如果面試官問我怎麼優化 `First Contentful Paint`，我不會只回答 lazy load 或 code splitting。

我會先講一個核心原則：

**FCP 的本質，是讓瀏覽器更早拿到並渲染第一批「真的需要顯示」的內容。**

所以優化方向通常不是單點技巧，而是整條載入路徑一起看：

- HTML 什麼時候回來
- critical CSS 什麼時候可用
- JavaScript 會不會阻塞渲染
- 首屏資料是不是一定要等 API
- 圖片、字體、第三方 script 有沒有搶首屏資源

---

## 先講 FCP 是什麼

`FCP`（First Contentful Paint）指的是：

**瀏覽器第一次把實際內容畫到畫面上的時間。**

這個內容可能是：

- 文字
- 圖片
- SVG
- 非純背景的 canvas

它不是整頁載入完成，也不是互動完成。

它回答的是：

**使用者什麼時候第一次看到有東西出現。**

所以如果首頁一開始是白畫面 3 秒，就算後面互動很順，使用者體感還是會覺得慢。

---

## 面試回答版本

如果我要優化前端應用的首次載入時間，也就是 `FCP`，我會先從資源載入順序和首屏渲染路徑下手，而不是一開始就做局部微調。

我通常會先確認三件事：

- 首屏內容是否被 JavaScript 阻塞
- 首屏樣式是否被大包 CSS 或字體拖慢
- 首屏畫面是否必須等待 API 才能 render

具體策略上，我會優先做：

- 減少初始 JavaScript bundle
- 做 route-level 或 component-level code splitting
- 把非首屏元件 lazy load
- 內嵌或提早提供 critical CSS
- 延後不必要的第三方 script
- 對字體、圖片、API 做首屏優先級控制
- 如果是 SSR / streaming 能帶來幫助，我會考慮 server 先把首屏 HTML 回出來

我也不會只看 Lighthouse 分數。我會搭配：

- Chrome Performance panel
- Lighthouse
- Web Vitals
- network waterfall

去找真正卡住 FCP 的資源。

---

## 先有一個正確心智模型

FCP 可以粗略想成這條路：

```text
Request HTML
  -> parse HTML
  -> discover CSS / JS / fonts / images
  -> load critical assets
  -> build DOM + CSSOM
  -> first paint with actual content
```

所以任何會拖慢這條路的東西，都會影響 FCP。

常見兇手：

- HTML 回太慢
- CSS 太大而且 blocking
- JS bundle 太大
- 首屏 render 綁死在 API response
- web font 太慢導致文字不顯示
- 第三方 script 搶網路與 main thread

---

## 我的優化流程

如果是真的工作上要做，我通常會照這個順序。

### 1. 先量測，不先猜

先看：

- Lighthouse 的 FCP
- WebPageTest / Chrome DevTools waterfall
- 哪個 request 最晚回
- main thread 哪裡在執行長任務
- 哪些 CSS / JS 在首屏其實不需要

如果不先量測，很容易做了很多「感覺有優化」但對 FCP 沒影響的事。

### 2. 確認首屏真正需要什麼

這步很重要。

很多頁面一開始把整個 app shell、analytics、modal、chart、editor、A/B testing SDK、所有 tab content 全塞進首頁。

但真正首屏可能只需要：

- header
- hero text
- 一張小圖
- call-to-action

所以我會先把「首屏需要的東西」和「後面再載入的東西」拆開。

### 3. 重新安排載入優先級

這時才進入具體優化：

- 首屏需要：提早給
- 首屏不需要：延後
- 可以拆開：拆開
- 可以 server 先 render：server 先 render

---

## 具體策略 1：縮小初始 JavaScript bundle

大 bundle 很容易拖慢 FCP，因為：

- 檔案下載要時間
- parse / compile 要時間
- execute 也要時間

如果頁面是 CSR（client-side rendering），bundle 太大時，常常會變成：

```text
HTML 很快到
但 JS 還沒跑完
畫面還是白的
```

### 做法：route-level code splitting

```tsx
import { Suspense, lazy } from 'react'
import { BrowserRouter, Routes, Route } from 'react-router-dom'

const HomePage = lazy(() => import('./pages/HomePage'))
const DashboardPage = lazy(() => import('./pages/DashboardPage'))
const SettingsPage = lazy(() => import('./pages/SettingsPage'))

export function AppRouter() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<HomePage />} />
          <Route path="/dashboard" element={<DashboardPage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

### 為什麼有用

如果使用者先進首頁，就不用一開始把 dashboard、settings 的 JS 全部下載下來。

這能減少：

- initial download size
- parse / execute cost
- main thread 壓力

---

## 具體策略 2：把非首屏元件 lazy load

很多頁面的首屏裡，真正重的東西通常是：

- chart
- map
- rich text editor
- comments 區塊
- recommendations
- modal / drawer content

這些不一定需要一開始就進主 bundle。

```tsx
import { Suspense, lazy } from 'react'

const ProductChart = lazy(() => import('./ProductChart'))
const RecommendationPanel = lazy(() => import('./RecommendationPanel'))

export function ProductPage() {
  return (
    <main>
      <section>
        <h1>Product Title</h1>
        <p>Short description for first screen.</p>
      </section>

      <section style={{ minHeight: 240 }}>
        <Suspense fallback={<div>Loading chart...</div>}>
          <ProductChart />
        </Suspense>
      </section>

      <section>
        <Suspense fallback={<div>Loading recommendations...</div>}>
          <RecommendationPanel />
        </Suspense>
      </section>
    </main>
  )
}
```

### 重點

不是什麼都 lazy load。

如果把首屏最重要的 hero 內容也 lazy load，使用者還是會先看到空白或 loading placeholder，那就沒有真的改善 FCP。

所以判斷標準是：

**首屏必要內容不要延後，首屏非必要內容才延後。**

---

## 具體策略 3：減少 render-blocking CSS

CSS 會影響 FCP，因為瀏覽器通常要先拿到 CSSOM，才能正確繪製內容。

如果 CSS 很大，而且整包 blocking，就會拖慢首屏。

### 做法一：critical CSS inline

```html
<!doctype html>
<html lang="en">
  <head>
    <style>
      body {
        margin: 0;
        font-family: system-ui, sans-serif;
      }

      .hero {
        padding: 48px 24px;
      }

      .hero-title {
        font-size: 40px;
        font-weight: 700;
        line-height: 1.1;
      }
    </style>

    <link rel="stylesheet" href="/assets/app.css" />
  </head>
</html>
```

### 做法二：避免把整個 design system 首次全載

如果頁面只需要 10% 的樣式，卻一開始載 100% CSS，通常是浪費。

實務上可以搭配：

- CSS code splitting
- route-level CSS
- utility-first CSS 的 tree-shaking

---

## 具體策略 4：不要讓首屏一定卡在 API

很多前端頁面白屏很久，不是因為 CSS 或 JS，而是因為頁面寫成：

```tsx
if (!data) return null
```

這會讓首屏完全依賴 API。

比較差的寫法：

```tsx
export function HomePage() {
  const { data, isLoading } = useHeroQuery()

  if (isLoading) {
    return null
  }

  return <HeroSection data={data} />
}
```

比較好的方式是：

- 先 render app shell / text skeleton
- 或讓 server 先把首屏資料帶進來

```tsx
export function HomePage() {
  const { data, isLoading } = useHeroQuery()

  return (
    <main>
      <header>
        <h1>Welcome</h1>
      </header>

      <section>
        {isLoading ? (
          <HeroSkeleton />
        ) : (
          <HeroSection data={data} />
        )}
      </section>
    </main>
  )
}
```

### 更進一步：SSR / streaming

如果是 Next.js / Remix / RSC 類型架構，可以考慮讓 server 先回首屏 HTML。

這樣好處是：

- 使用者更早看到內容
- FCP 通常會比純 CSR 好

---

## 具體策略 5：優化字體載入

Web font 很常讓文字延後顯示，尤其如果設計要求自訂字體。

### 做法一：`font-display: swap`

```css
@font-face {
  font-family: 'InterCustom';
  src: url('/fonts/inter.woff2') format('woff2');
  font-display: swap;
}
```

這樣可以避免文字長時間 invisible。

### 做法二：preload 首屏必要字體

```html
<link
  rel="preload"
  href="/fonts/inter.woff2"
  as="font"
  type="font/woff2"
  crossorigin
/>
```

### 做法三：先用 system font

如果設計容許，system font 常常是最便宜的選擇。

因為：

- 不需要額外 request
- 不需要等待字體檔

---

## 具體策略 6：優化首屏圖片

首屏圖片很常影響 FCP，尤其 hero image。

### 錯誤情況

- 用超大原圖
- 沒有壓縮
- 沒有尺寸資訊
- 首屏圖和其他非必要圖片一起競爭頻寬

### 做法

- 使用正確尺寸
- 使用 WebP / AVIF
- preload 首屏 hero image
- lazy load 非首屏圖片

```html
<link
  rel="preload"
  as="image"
  href="/images/hero.avif"
/>
```

```tsx
export function Gallery() {
  return (
    <section>
      <img
        src="/images/hero.avif"
        width={1200}
        height={700}
        alt="Hero"
      />

      <img
        src="/images/detail-1.webp"
        loading="lazy"
        width={400}
        height={300}
        alt="Detail 1"
      />
    </section>
  )
}
```

### 為什麼尺寸重要

因為可以讓瀏覽器更早預留 layout space，也能減少後續 layout shift。

---

## 具體策略 7：延後第三方 script

第三方 script 很容易害你首屏變慢，尤其：

- analytics
- A/B testing SDK
- chat widget
- heatmap
- ads

它們的問題不只是多一個 request，而是：

- 搶主線程
- 搶網路資源
- 可能插入同步 script

### 做法

- 非必要 script 用 `defer` / `async`
- 互動後再載
- 頁面穩定後再載

```html
<script async src="https://example.com/analytics.js"></script>
```

或在前端延後：

```tsx
useEffect(() => {
  const timer = window.setTimeout(() => {
    const script = document.createElement('script')
    script.src = 'https://example.com/chat-widget.js'
    script.async = true
    document.body.appendChild(script)
  }, 3000)

  return () => clearTimeout(timer)
}, [])
```

---

## 具體策略 8：用 preconnect / preload 幫首屏資源加速

如果你知道某些關鍵資源一定會用到，可以提早告訴瀏覽器。

### `preconnect`

```html
<link rel="preconnect" href="https://cdn.example.com" crossorigin />
```

用途：

- 提早做 DNS lookup
- 提早做 TCP / TLS handshake

### `preload`

```html
<link rel="preload" href="/assets/home-hero.css" as="style" />
<link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin />
```

但要注意：

不要亂 preload 一堆東西，不然只是把網路優先級搞亂。

---

## 具體策略 9：減少 hydration / execute 成本

如果是 React app，首屏慢有時候不是下載慢，而是 hydration 太重。

尤其當首頁一開始就有很多：

- 大量 hooks
- 複雜 context
- 大量 client-only logic
- 很重的 chart / editor / animation

這時候即使 HTML 已經出來，真正 render 與 hydrate 也會慢。

### 可以做的事情

- 減少首頁 client component 數量
- 把重邏輯延後
- 把非必要互動元件 island 化
- 用 server component / SSR 減少 client work

面試時可以說：

**If the app is React-based, I would also inspect hydration cost, because sometimes the issue is not network time, but too much JavaScript execution before the first meaningful content is painted.**

---

## 一個比較完整的實戰流程

假設現在首頁 FCP 很差，我會這樣做。

### Step 1：量測

- 開 Lighthouse
- 開 Chrome Performance
- 看 network waterfall
- 看 main thread long tasks

### Step 2：定位首屏依賴

確認：

- 首屏有沒有卡在 API
- CSS 是不是 blocking
- JS bundle 是不是過大
- 字體 / 圖片 / 第三方 script 有沒有搶資源

### Step 3：做最有影響的優化

優先順序通常會是：

1. 拆 bundle
2. 拆首屏與非首屏內容
3. critical CSS
4. 首屏資料不阻塞
5. 字體 / 圖片 / 第三方 script 優先級調整

### Step 4：重新量測

重新看：

- FCP 改善多少
- Lighthouse / Web Vitals 變化
- 使用者真實監控（RUM）是否改善

如果只做優化不驗證，就很容易落入自我感覺良好。

---

## 面試時可以提到的工具

### 量測工具

- Lighthouse
- Chrome DevTools Performance
- Chrome Network panel
- WebPageTest
- Web Vitals

### 前端框架 / 技術

- React.lazy
- Suspense
- dynamic import
- Next.js SSR / SSG / streaming
- React Server Components

### 資源優化

- preload
- preconnect
- `font-display: swap`
- image compression
- code splitting
- critical CSS

### Bundler / build

- Vite bundle analyzer
- webpack-bundle-analyzer
- source-map-explorer

---

## 英文面試回答版本

**If I need to optimize First Contentful Paint, I would focus on shortening the critical rendering path. My goal would be to make the browser render meaningful above-the-fold content as early as possible, instead of waiting for the full application to load.**

**In practice, I would first measure what is blocking FCP using Lighthouse, Chrome DevTools, and the network waterfall. Then I would look for common bottlenecks such as oversized JavaScript bundles, render-blocking CSS, slow web fonts, large hero images, third-party scripts, and pages that block rendering on API responses.**

**From there, I would apply strategies such as code splitting, lazy loading non-critical components, inlining critical CSS, deferring non-essential scripts, optimizing fonts and images, and avoiding rendering flows where the entire first screen depends on data fetching. If the stack supports it, I would also consider SSR or streaming so the initial HTML can be rendered earlier.**

**The key idea is not just to reduce bytes, but to make sure the browser gets the first meaningful content earlier and with fewer blockers.**

---

## 最後一句總結

如果要把這題濃縮成一句最好記的話，我會記：

**FCP optimization is about prioritizing above-the-fold content and removing blockers from the critical rendering path.**
