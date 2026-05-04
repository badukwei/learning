# Cache Stampede（快取雪崩）

> 問題：大量 key 同時過期 → 全部打到 DB → DB 過載崩潰
> 三種解法：TTL 隨機抖動 / Mutex Lock / Cache Warming

---

## 什麼是 Cache Stampede

```
正常流程：
  User → Redis（Hit）→ 回傳
  User → Redis（Miss）→ DB → 寫回 Redis → 回傳

Stampede：
  1000 個 User 同時 → Redis（全部 Miss，key 剛好過期）
                    → 1000 個請求同時打 DB
                    → DB 過載 → 服務掛掉
```

---

## Case 1：電商秒殺活動

### 情境

```
雙 11 活動 00:00 開始
前一天 00:00 系統自動建立所有商品快取，TTL = 86400（24 小時）
→ 隔天 00:00，所有快取同時過期

00:00 整，百萬用戶湧入搶購
→ 所有熱門商品同時 Cache Miss
→ DB 被百萬請求同時打爛
```

### 用戶流程

```
00:00:00
  User A → GET /products/iphone → Redis Miss → 打 DB
  User B → GET /products/iphone → Redis Miss → 打 DB（同一時間）
  User C → GET /products/iphone → Redis Miss → 打 DB（同一時間）
  ... × 100 萬

DB：100 萬個查詢同時進來 → 連線數爆炸 → OOM → 崩潰
→ 所有用戶看到 500 Error
```

### 解法 1：TTL 加隨機抖動

讓快取不在同一時間過期。

```typescript
function getJitteredTTL(baseTTL: number, jitter: number = 600): number {
  return baseTTL + Math.floor(Math.random() * jitter)
  // baseTTL = 3600，jitter = 600
  // 實際 TTL 落在 3600 ~ 4200（分散在 10 分鐘內過期）
}

async function cacheProduct(productId: string, data: any) {
  const ttl = getJitteredTTL(3600)
  await redis.setex(`product:${productId}`, ttl, JSON.stringify(data))
}
```

### 解法 2：活動前預熱（Cache Warming）

不等用戶觸發，00:00 前主動把熱門商品寫入 Redis。

```typescript
// scripts/warm-cache.ts（活動前 30 分鐘執行）
async function warmCache() {
  const hotProducts = await db.product.findMany({
    where: { isFeatured: true },
    orderBy: { viewCount: 'desc' },
    take: 1000,
  })

  // 批次寫入 Redis
  await Promise.all(
    hotProducts.map((product) => {
      const ttl = getJitteredTTL(7200)
      return redis.setex(
        `product:${product.id}`,
        ttl,
        JSON.stringify(product)
      )
    })
  )

  console.log(`預熱完成：${hotProducts.length} 個商品`)
}

warmCache()
```

### 架構流程

```
活動前 23:30：
  預熱腳本 → 查 DB 熱門商品 → 批次寫入 Redis（TTL 各不同）

00:00 活動開始：
  User × 100 萬 → Redis（全部 Hit，預熱好了）→ 直接回傳
  DB：幾乎沒有請求

部分 key 過期時（隨機時間，不同時）：
  少數 User → Redis Miss → DB（少量）→ 寫回 Redis
  不是百萬同時，而是分批、少量
```

---

## Case 2：明星發文（Twitter / 微博）

### 情境

```
某明星發一則貼文
500 萬粉絲同時點開
這則貼文的 cache key 剛好在此時過期（TTL 到了）
→ 500 萬個請求同時 Miss → DB 掛掉
```

### 用戶流程

```
User A → GET /posts/9527 → Redis Miss → 去 DB 查 → 等待中...
User B → GET /posts/9527 → Redis Miss → 去 DB 查 → 等待中...
User C → GET /posts/9527 → Redis Miss → 去 DB 查 → 等待中...
... × 500 萬

DB 同時收到 500 萬個一樣的查詢
→ 重複計算、重複 I/O
→ CPU 100%、連線數爆炸 → 崩潰
```

### 解法：Mutex Lock（互斥鎖）

只讓第一個 Miss 的請求去 DB，其他等待或回傳舊資料。

```typescript
async function getWithLock<T>(
  key: string,
  ttl: number,
  fetcher: () => Promise<T>
): Promise<T | null> {
  // 1. 先查 Redis
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)

  // 2. Cache Miss → 搶 lock
  const lockKey = `lock:${key}`
  const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 10)
  // NX = 不存在才設（原子操作）
  // 第一個請求拿到 "OK"，其他拿到 null

  if (acquired === 'OK') {
    // 搶到 lock → 去 DB 查
    try {
      const data = await fetcher()
      await redis.setex(key, ttl, JSON.stringify(data))
      return data
    } finally {
      await redis.del(lockKey)  // 一定要釋放 lock
    }
  } else {
    // 沒搶到 → 等 100ms，等對方重建完
    await new Promise(resolve => setTimeout(resolve, 100))
    const retried = await redis.get(key)
    if (retried) return JSON.parse(retried)
    return null  // 還沒好，前端顯示 loading
  }
}
```

**使用**

```typescript
app.get('/posts/:id', async (req, res) => {
  const post = await getWithLock(
    `post:${req.params.id}`,
    300,
    () => db.post.findUnique({ where: { id: req.params.id } })
  )

  if (!post) {
    return res.status(503).json({ error: '服務暫時繁忙，請稍後再試' })
  }

  res.json(post)
})
```

### 架構流程

```
500 萬個請求同時進來：

Request #1（最快到）：
  → Redis Miss
  → SET lock:post:9527 NX → 成功
  → 查 DB（只有這一個）
  → 寫回 Redis
  → DEL lock:post:9527
  → 回傳資料

Request #2 ~ #500 萬：
  → Redis Miss
  → SET lock:post:9527 NX → 失敗（lock 已在）
  → 等 100ms
  → 再查 Redis → 有了（#1 重建完）→ 回傳

DB：只收到 1 個查詢，不是 500 萬
```

---

## Case 3：Facebook Memcached（2013）

### 情境

```
Facebook 幾千台 Memcached，某台當機重啟
→ 這台負責的所有 key 同時消失
→ 幾千萬個請求同時 Miss → MySQL 被打爛
```

### Facebook 的解法：Lease 機制

Lease 同時解決兩個問題：Thundering Herd 和 Stale Set。

**問題一：Thundering Herd（多個請求同時 Miss）**

```
Client A → GET post:123 → Miss
  Memcached 發 token:67890 給 A → A 去 MySQL

Client B → GET post:123 → Miss
  Memcached：token:67890 已發出，回傳 "wait"
  → B 等 10ms 重試

A 查完 → SET post:123 <data> token:67890
  Memcached 驗證 token ✓ → 寫入

B 重試 → GET post:123 → Hit

MySQL：只收到 1 個查詢
```

**問題二：Stale Set（更隱微的 Bug）**

沒有 Lease 會發生這個情況：

```
t=0  Client A 讀到舊資料（還在快取裡）
t=1  有人更新 post:123 → Memcached 刪 key
t=2  Client B Miss → 查 MySQL → 寫回新資料
t=3  Client A 把 t=0 拿的舊資料 SET 進去 → 把新資料蓋掉

→ 快取永遠是舊的，直到 TTL 過期
```

有 Lease：

```
t=0  Client A 讀到舊資料，同時拿到 token:111
t=1  有人更新 → Memcached 刪 key，token:111 失效
t=2  Client B Miss → 拿到新 token:222 → 查 MySQL → SET token:222 → 成功
t=3  Client A 拿 token:111 來 SET → Memcached 拒絕（token 已失效）

→ 舊資料永遠寫不進去
```

**完整流程圖**

```
Client A              Memcached             MySQL
   │                      │                   │
   │── GET post:123 ──────>│                   │
   │<─ MISS + token:67890 ─│                   │
   │                       │                   │
   │            Client B   │                   │
   │               │── GET post:123 ──────────>│
   │               │<─ MISS + "wait" ──────────│
   │                       │                   │
   │── SELECT * FROM posts ────────────────────>│
   │<─ { id:123, ... } ────────────────────────│
   │                       │                   │
   │── SET post:123        │                   │
   │   <data> token:67890 ─>│                   │
   │               驗證 token ✓                 │
   │               寫入快取                     │
   │<─ OK ─────────────────│                   │
   │                       │                   │
   │            Client B（10ms 後重試）          │
   │               │── GET post:123 ──────────>│
   │               │<─ HIT <data> ─────────────│
```

**Token 細節**

```
格式：64-bit integer，每個 key 各自一組
有效期：約 10 秒

過期的原因：
  Client A 拿了 token 但當機 → 沒人來 SET
  → Token 10 秒後自動失效
  → 下一個 Miss 的請求拿到新 token 繼續
  → 不會永遠卡住（防 deadlock）
```

**跟 Redis Mutex Lock 的差異**

```
Redis Mutex Lock（自己實作）：
  lock 存在 Redis 裡（另一個 key）
  需要自己管建立、釋放、過期
  沒有解決 Stale Set 問題

Facebook Lease：
  token 由 Memcached 內建管理
  Miss、token 發放、驗證都在 Memcached 內部完成
  同時解決 Thundering Herd + Stale Set
```

這個概念後來在 Redis 生態系由 `redlock` 套件實作。

---

## 三種解法比較

| 解法 | 適用場景 | 複雜度 | 效果 |
|------|---------|--------|------|
| TTL 隨機抖動 | 通用，預防雪崩 | 低 | 分散過期時間 |
| Cache Warming | 可預測流量高峰（活動、部署）| 中 | 根本不讓 Miss 發生 |
| Mutex Lock | 突發流量、無法預測 | 中 | Miss 時只讓一個打 DB |

**生產環境標配（組合使用）**

```
平時     → TTL 加隨機抖動（預防）
活動前   → Cache Warming（消除 Miss）
突發狀況 → Mutex Lock（限制 DB 壓力）
```

---

## 完整架構圖

```
用戶請求
  ↓
Load Balancer
  ↓
App Server
  │
  ├─ redis.get(key)
  │     ├─ HIT → 直接回傳（99% 的情況）
  │     │
  │     └─ MISS
  │           ├─ 搶 lock（SET NX）
  │           │     ├─ 成功 → 查 DB → 寫回 Redis → 釋放 lock → 回傳
  │           │     └─ 失敗 → 等 100ms → 重試 Redis → 回傳
  │           │
  │           └─ （無 lock 版）直接查 DB → 寫回 Redis → 回傳
  ↓
Redis Cluster
  ↓（Miss 才到這層）
PostgreSQL
```
