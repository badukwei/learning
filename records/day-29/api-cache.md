# API 快取設計

> 核心問題：同樣的資料一直重複查 DB，浪費資源、拖慢速度
> 解法：把結果存起來，下次直接回傳

---

## 架構全貌

```
Frontend（瀏覽器）
  → 記憶體快取（SWR / React Query）
  → HTTP 快取（Cache-Control、ETag）
        ↓ 沒命中
Backend（App Server）
  → Redis 快取
        ↓ 沒命中
  → Database
```

每一層都能擋住請求，越早擋越快、越省資源。

---

## 一、後端快取（Redis）

### Key 設計原則

```
格式：{資源類型}:{id}:{子資源}

user:123:profile          → 用戶個人資料
user:123:posts            → 用戶的文章列表
product:456:detail        → 商品詳情
feed:user:123:page:1      → 用戶動態第 1 頁
search:keyword:iphone:page:1  → 搜尋結果

原則：
  - 有階層，用冒號分隔
  - 包含所有影響結果的變數（user id、page、keyword）
  - 不同查詢條件 → 不同 key，不能混用
```

### TTL 策略

```
更新頻率高 → TTL 短   更新頻率低 → TTL 長

用戶個人資料     → 5 分鐘（偶爾改）
商品詳情        → 1 小時（很少改）
搜尋結果        → 10 分鐘
用戶動態 Feed   → 1-2 分鐘（常更新）
排行榜         → 1 分鐘
系統設定        → 1 天

通用原則：
  TTL = 你能接受的「最舊資料」時間
  加隨機抖動防止同時過期：TTL = base + Math.random() * 60
```

### Cache-Aside 實作（Node.js + ioredis）

```typescript
// lib/cache.ts
import redis from './redis'

export async function withCache<T>(
  key: string,
  ttl: number,
  fetcher: () => Promise<T>
): Promise<T> {
  // 1. 查 Redis
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)

  // 2. 查 DB
  const data = await fetcher()

  // 3. 寫回 Redis
  await redis.setex(key, ttl, JSON.stringify(data))

  return data
}

export async function invalidateCache(key: string) {
  await redis.del(key)
}

// 刪除一個 pattern 的所有 key（e.g. 某用戶的所有快取）
export async function invalidatePattern(pattern: string) {
  const keys = await redis.keys(pattern)  // 注意：生產環境改用 SCAN
  if (keys.length > 0) await redis.del(...keys)
}
```

**使用**

```typescript
// routes/users.ts
import { withCache, invalidateCache } from '@/lib/cache'

// GET /users/:id → 查用戶資料（帶快取）
app.get('/users/:id', async (req, res) => {
  const { id } = req.params
  const key = `user:${id}:profile`

  const user = await withCache(key, 300, () =>
    db.user.findUnique({ where: { id } })
  )

  res.json(user)
})

// PUT /users/:id → 更新用戶資料（更新後刪快取）
app.put('/users/:id', async (req, res) => {
  const { id } = req.params

  const user = await db.user.update({
    where: { id },
    data: req.body,
  })

  // 刪快取，讓下次讀重建
  await invalidateCache(`user:${id}:profile`)

  res.json(user)
})
```

### Next.js API Route 範例

```typescript
// app/api/users/[id]/route.ts
import redis from '@/lib/redis'
import { db } from '@/lib/db'

export async function GET(
  req: Request,
  { params }: { params: { id: string } }
) {
  const key = `user:${params.id}:profile`

  const cached = await redis.get(key)
  if (cached) {
    return Response.json(JSON.parse(cached as string), {
      headers: { 'X-Cache': 'HIT' },  // 方便 debug
    })
  }

  const user = await db.user.findUnique({ where: { id: params.id } })
  if (!user) return Response.json({ error: 'Not found' }, { status: 404 })

  await redis.setex(key, 300, JSON.stringify(user))

  return Response.json(user, {
    headers: { 'X-Cache': 'MISS' },
  })
}
```

---

## 二、HTTP 快取（後端控制，瀏覽器執行）

### Cache-Control Header

後端在 Response header 告訴瀏覽器怎麼快取。

```
Cache-Control: max-age=300          → 快取 5 分鐘，期間不重新請求
Cache-Control: no-cache             → 每次都問伺服器（可用 ETag 驗證）
Cache-Control: no-store             → 完全不快取（敏感資料用）
Cache-Control: public, max-age=3600 → CDN 也可以快取
Cache-Control: private, max-age=300 → 只有瀏覽器快取，CDN 不快取（含個人資料）
Cache-Control: stale-while-revalidate=60 → 過期後先回傳舊的，背景重新驗證
```

**設定方式（Express）**

```typescript
// 商品資料：公開、快取 1 小時、CDN 也快取
app.get('/products/:id', (req, res) => {
  res.set('Cache-Control', 'public, max-age=3600')
  res.json(product)
})

// 用戶資料：私人、快取 5 分鐘
app.get('/users/:id/profile', (req, res) => {
  res.set('Cache-Control', 'private, max-age=300')
  res.json(user)
})

// 即時資料：不快取
app.get('/notifications', (req, res) => {
  res.set('Cache-Control', 'no-store')
  res.json(notifications)
})
```

**設定方式（Next.js）**

```typescript
export async function GET() {
  return Response.json(data, {
    headers: {
      'Cache-Control': 'public, max-age=3600, stale-while-revalidate=60',
    },
  })
}
```

### ETag（條件式請求）

資料沒變就不重傳，省頻寬。

```
第一次請求：
  Client → GET /products/123
  Server → 200 OK + ETag: "abc123" + 資料

第二次請求：
  Client → GET /products/123 + If-None-Match: "abc123"
  Server → 資料沒變 → 304 Not Modified（不回傳 body，省頻寬）
           資料有變 → 200 OK + ETag: "def456" + 新資料
```

```typescript
import crypto from 'crypto'

app.get('/products/:id', async (req, res) => {
  const product = await db.product.findUnique({ where: { id: req.params.id } })
  const etag = crypto
    .createHash('md5')
    .update(JSON.stringify(product))
    .digest('hex')

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end()  // 資料沒變，不用回傳
  }

  res.set('ETag', etag)
  res.set('Cache-Control', 'public, max-age=60')
  res.json(product)
})
```

---

## 三、前端快取（React）

### SWR（Next.js 生態系首選）

```bash
npm install swr
```

```typescript
import useSWR from 'swr'

const fetcher = (url: string) => fetch(url).then(res => res.json())

function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading, mutate } = useSWR(
    `/api/users/${userId}`,
    fetcher,
    {
      revalidateOnFocus: true,      // 切回視窗自動重新驗證
      revalidateOnReconnect: true,  // 重新連線自動重新驗證
      dedupingInterval: 2000,       // 2 秒內的相同請求只發一次
      refreshInterval: 0,           // 不自動輪詢（設秒數就變輪詢）
    }
  )

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Error</div>

  return <div>{data.name}</div>
}
```

**更新後讓快取失效**

```typescript
// 方式 1：直接更新快取內容（樂觀更新，不等 API）
const updateUser = async (newData) => {
  await fetch(`/api/users/${userId}`, { method: 'PUT', body: JSON.stringify(newData) })
  mutate()  // 重新 fetch
}

// 方式 2：樂觀更新（先改 UI，失敗再還原）
const updateUser = async (newData) => {
  mutate(
    async () => {
      await fetch(`/api/users/${userId}`, { method: 'PUT', body: JSON.stringify(newData) })
      return newData
    },
    { optimisticData: newData, rollbackOnError: true }
  )
}
```

### TanStack Query（React Query）

```bash
npm install @tanstack/react-query
```

```typescript
// app/providers.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,  // 5 分鐘內不重新 fetch
      gcTime: 10 * 60 * 1000,    // 10 分鐘後從記憶體移除
    },
  },
})

export function Providers({ children }) {
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}
```

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading } = useQuery({
    queryKey: ['user', userId],          // key 跟 Redis 的設計邏輯一樣
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000,           // 5 分鐘內視為新鮮，不重新 fetch
  })

  return <div>{data?.name}</div>
}

function UpdateUserButton({ userId }) {
  const queryClient = useQueryClient()

  const mutation = useMutation({
    mutationFn: (data) =>
      fetch(`/api/users/${userId}`, { method: 'PUT', body: JSON.stringify(data) }),
    onSuccess: () => {
      // 更新成功後，讓這個 key 的快取失效
      queryClient.invalidateQueries({ queryKey: ['user', userId] })
    },
  })

  return <button onClick={() => mutation.mutate({ name: 'New Name' })}>更新</button>
}
```

### Next.js fetch 內建快取

```typescript
// app/page.tsx（Server Component）

// 預設：永久快取（build time fetch）
const data = await fetch('https://api.example.com/products')

// 不快取（每次請求都 fetch）
const data = await fetch('https://api.example.com/products', {
  cache: 'no-store'
})

// 定時重新驗證
const data = await fetch('https://api.example.com/products', {
  next: { revalidate: 3600 }  // 1 小時後重新 fetch
})

// 標籤式快取（可手動讓特定資料失效）
const data = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] }
})
```

```typescript
// 在 Server Action 手動讓快取失效
import { revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: any) {
  await db.product.update({ where: { id }, data })
  revalidateTag('products')  // 讓所有帶 'products' tag 的快取失效
}
```

---

## 四、CDN Cache vs Redis Cache

### 根本差異

```
CDN Cache：
  快取的是 HTTP Response（整個回應）
  在網路層發生，不用寫任何程式碼
  CDN 自動讀 Cache-Control header 決定要不要存

Redis Cache：
  快取的是資料（JSON、字串、數字）
  在應用層發生，自己寫 redis.get / redis.set
  完全自己控制邏輯
```

### 請求路徑

```
用戶（台灣）
  ↓
CDN Edge Node（台灣機房）← Cache-Control: public → CDN 在這裡擋住
  ↓ Miss 才繼續
App Server → Redis     ← 個人化資料在這裡擋住
  ↓ Miss 才繼續
DB
```

CDN 命中 → Server 完全不知道這個請求存在。

### CDN 要自己設定嗎

**CDN 服務本身** → 用現有平台，不用自己架

```
Vercel      → deploy 上去就有，靜態資源自動上 CDN
Cloudflare  → DNS 指過去就有，免費方案夠用
CloudFront  → AWS 生態系，需要自己設定 Distribution
```

**快取行為** → 設好 Response header，CDN 自動照做

```typescript
// 後端這樣設就夠了
res.set('Cache-Control', 'public, max-age=3600')
// CDN 讀到 public → 自動快取 1 小時
// 不需要碰 CDN Dashboard
```

進階才需要進 Dashboard（手動清除快取、設路徑例外規則）。

### 各自適合快取什麼

```
CDN Cache（公開、所有用戶一樣）：
  靜態資源：JS、CSS、圖片 → max-age 很長（帶 hash，改了換檔名）
  公開 API：商品列表、文章 → max-age 幾分鐘到幾小時

Redis Cache（個人化、需要身份判斷）：
  用戶資料、購物車、權限 → 需要知道是誰才能返回
  運算結果、DB 查詢 → 省去重複計算

不能進 CDN 的：
  private 或 no-store header → CDN 直接放行
  POST / PUT / DELETE → CDN 不快取這些 method
```

---

## 五、各層快取對比

| | Redis（後端）| HTTP Cache-Control | SWR / React Query |
|--|------------|-------------------|------------------|
| 存在哪 | 後端 Server 記憶體 | 瀏覽器 / CDN | 瀏覽器記憶體 |
| 所有用戶共用 | 是 | 看設定（public/private）| 否（各自的） |
| 失效方式 | 主動 DEL | TTL 到期 / ETag | invalidate / mutate |
| 適合 | DB 查詢結果 | 靜態或半靜態資源 | 前端 UI 狀態 |

---

## 五、快取失效（最難的部分）

```
快取有兩個難題：
  1. 命名（key 怎麼設計）
  2. 失效（什麼時候要清掉）
```

**常見失效策略**

```
TTL 自動過期：
  最簡單，接受資料短暫不一致
  適合：商品資料、公開內容

事件驅動失效：
  資料更新時主動刪 key
  適合：用戶資料、需要即時更新的場景

  流程：
    PUT /users/123 → 更新 DB → redis.del('user:123:profile')
    → 下次 GET 重新從 DB 讀取並寫回 Redis

版本化 key：
  key 帶版本號，更新時改版本號
  user:123:profile:v2（舊的自然過期）
  適合：大規模更新，不想一個個刪

Write-Through：
  更新 DB 的同時更新 Redis（不刪，直接覆蓋）
  優：Cache 永遠是最新的
  缺：寫入多一步驟，要保證兩個都成功（考慮 transaction）
```

**哪個更新要清哪個 key**

```
用戶改了名字：
  → del user:123:profile
  → del feed:*（如果 feed 裡有顯示名字）← 這就是連鎖失效的複雜度

商品改了價格：
  → del product:456:detail
  → del search:*（搜尋結果裡可能有這個商品）← 更難處理

解法：
  接受短暫不一致（TTL 到期自然更新），不追求即時
  → 大部分場景這樣就夠了
```

---

## 六、Redis Cache 什麼時候去 DB 拿資料

Redis 完全被動，不知道 DB 是什麼，不會自己做任何檢查。
**是 App 決定什麼時候去 DB，不是 Redis。**

### 三種策略

**1. 純 TTL（最簡單）**

```
user → redis.get(key)
  有 → 直接回傳（不管 DB 有沒有更新）
  沒有（過期或不存在）→ 查 DB → 寫回 Redis

特性：TTL 內拿到的可能是舊資料，接受最終一致就夠用
```

**2. TTL + 寫入時主動刪（業界最普遍）**

```
讀：user → Redis 有就回傳
寫：更新 DB → 立刻 redis.del(key)
  → 下次讀 Cache Miss → 重建最新資料

TTL 同時設著：
  即使忘了刪，TTL 到期也會清掉
  → 雙重保險，錯誤不會永久存在
```

```typescript
// 寫入時主動刪
async function updateUser(id: string, data: any) {
  await db.user.update({ where: { id }, data })
  await redis.del(`user:${id}:profile`)  // 主動清快取
}
```

**3. 版本判斷（資料量大時的優化）**

```
Redis 存資料時帶 version：
  { data: {...}, version: 42 }

讀取流程：
  1. 拿 Redis 的 version
  2. 查 DB 的 version（只撈一個欄位，很快）
  3. 版本一樣 → 用 Redis 的資料，不用傳完整資料
  4. 版本不一樣 → 查 DB 完整資料 → 更新 Redis

注意：每次讀還是要打一次 DB（查 version）
→ 省了傳大量資料，但沒省掉 DB 連線
→ 資料量很大、更新頻率低時才划算，不是標配
```

### 怎麼選

```
資料可以短暫舊           → 純 TTL，最簡單
資料要相對即時           → TTL + 寫入時主動刪（90% 場景用這個）
資料量很大、更新頻率低   → 版本判斷
絕對不能舊（金融）       → 不快取，直接打 DB
```

---

## 七、快取沒更新到怎麼辦

### TTL 是什麼

```
TTL（Time To Live）= 快取的有效期限

TTL 到期：
  Redis 把這個 key 自動刪掉
  → Redis 不會主動去查 DB，它不知道 DB 是什麼

下次有請求進來：
  redis.get(key) → null（找不到）
  → App 才去查 DB → 寫回 Redis（重建快取）

重點：是請求觸發重建，不是 TTL 觸發
```

### TTL 當保底的意思

```
主動刪除 + TTL 同時設：

正常情況：
  更新 DB → 主動 redis.del(key) → 下次讀到最新資料

刪除失敗時（Redis 掛、程式 bug、忘記寫）：
  有設 TTL → 最多舊 N 分鐘 → TTL 到期自動消失 → 下次重建
  沒設 TTL → 快取永遠不消失 → 用戶永遠看到舊資料

→ TTL 是你忘記或失敗時的最後防線，確保錯誤不會永久存在
→ 所以即使有主動刪除邏輯，TTL 還是要設，兩個機制互補
```

### 快取沒更新的三種根本原因

```
1. 忘記刪快取
   → 靠 TTL 保底，最多舊 N 分鐘

2. Race Condition（刪了又被蓋回去）
   Thread A：讀 DB（舊值）→ ...
   Thread B：寫 DB → 刪 Redis
   Thread A：... → 把舊值寫回 Redis  ← 蓋掉新資料

   → 改用 Write-Through（寫 DB 時直接覆蓋 Redis，不是刪掉）

3. Redis 當下掛掉，del 失敗
   → TTL 保底
   → 或把「要刪的 key」丟進 Queue，worker 確保一定執行
```

---

## 七、金融場景的快取設計

一般場景接受「最終一致性」，但金融場景不行。

### 一般 vs 金融

```
一般場景（Eventually Consistent）：
  接受短暫舊資料，換取速度
  TTL 5 分鐘 → 最壞看到舊資料 5 分鐘 → 可接受

金融場景（Strong Consistency）：
  餘額錯了會出事，絕對不接受舊資料
  → 換策略，不用 TTL 保底那套
```

### 金融場景的做法

**1. 不快取，直接打 DB**

```
餘額、交易紀錄 → 每次都查 DB，完全不過 Redis
效能差一點，但資料永遠是最新的
最簡單、最安全
```

**2. Write-Through + Transaction**

```typescript
// 更新餘額：DB 跟 Redis 在同一個 transaction 裡一起更新
await db.transaction(async (tx) => {
  await tx.account.update({
    where: { id: accountId },
    data: { balance: newBalance },
  })
  await redis.set(`balance:${accountId}`, newBalance)
  // 任一失敗 → rollback，兩個保持一致
})
```

**3. 讀的時候加分散式鎖**

```typescript
// 確保同一時間只有一個人在讀寫這筆資料
const lock = await acquireLock(`account:${accountId}`)
if (!lock) throw new Error('系統繁忙，請稍後再試')

try {
  const balance = await db.account.findUnique(...)  // 直接查 DB，不走 Cache
  return balance
} finally {
  await releaseLock(`account:${accountId}`, lock)
}
```

### Redis 在金融系統的合法用途

```
不適合快取的：
  餘額、交易紀錄、訂單狀態 → 直接打 DB

適合用 Redis 的：
  匯率資料（每分鐘更新，不是即時）→ 短 TTL Cache
  用戶基本資料（名字、設定）→ 一般 Cache
  Rate Limiting（防止重複提交）→ INCR + EXPIRE
  分散式鎖（防止重複扣款）→ SET NX EX
  歷史報表（已確認的資料）→ 長 TTL Cache
```
