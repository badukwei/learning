# Rate Limiting（限流）

> 問題：單一用戶或 IP 發送過多請求 → 拖垮服務
> 核心：Redis INCR + EXPIRE，計數超過門檻就擋

---

## 什麼是 Rate Limiting

```
正常：
  User → 每秒 1-2 個請求 → 正常處理

過多：
  攻擊者 → 每秒 1000 個請求 → DB / API 過載
  爬蟲   → 不停抓資料 → 伺服器資源耗盡
  用戶   → 不小心 retry loop → 把自己的帳號打爆
```

---

## Case 1：API 濫用（Fixed Window）

### 情境

```
公開 API，每個用戶每分鐘最多 100 次
某個 API key 開始狂打，每分鐘 10,000 次
→ 沒有限流的話，DB / 下游服務直接爆
```

### 解法：Fixed Window Counter

```
key = "rate:{userId}:{當前分鐘時間戳}"
每次請求 INCR，超過 100 就回 429
EXPIRE 設 60 秒，時間窗口自動重置
```

```typescript
async function checkRateLimit(userId: string): Promise<boolean> {
  const window = Math.floor(Date.now() / 1000 / 60)  // 當前分鐘
  const key = `rate:${userId}:${window}`

  const count = await redis.incr(key)

  if (count === 1) {
    // 第一次請求，設 TTL（確保 key 會被清掉）
    await redis.expire(key, 60)
  }

  return count <= 100  // true = 允許，false = 擋掉
}

// Middleware
app.use(async (req, res, next) => {
  const userId = req.headers['x-api-key'] as string
  const allowed = await checkRateLimit(userId)

  if (!allowed) {
    return res.status(429).json({ error: 'Too Many Requests' })
  }

  next()
})
```

### 架構流程

```
請求進來：
  → 取當前分鐘時間戳（e.g. 27453870）
  → INCR rate:{userId}:27453870
  → count = 1   → 設 EXPIRE 60s → 放行
  → count = 50  → 放行
  → count = 100 → 放行（剛好到）
  → count = 101 → 回 429

下一分鐘（時間戳 27453871）：
  → 新 key，重新從 1 開始計
  → 舊 key 60 秒後自動消失
```

### Fixed Window 的問題

```
分鐘邊界攻擊：
  第 59 秒：打 100 次（用完這分鐘的額度）
  第 61 秒：打 100 次（新的一分鐘，額度重置）

→ 2 秒內實際發了 200 個請求，但都通過了
```

---

## Case 2：登入暴力破解（Sliding Window）

### 情境

```
攻擊者對同一個帳號不停試密碼
  POST /login { email: "victim@gmail.com", password: "password123" }
  POST /login { email: "victim@gmail.com", password: "qwerty" }
  ... 每秒幾百次

沒有限流 → 幾分鐘內試完常見密碼字典
```

### 解法：Sliding Window Log

用 Redis Sorted Set 記錄每次請求的時間戳，只保留最近 N 秒的記錄。

```typescript
async function checkLoginRateLimit(email: string): Promise<boolean> {
  const key = `login:${email}`
  const now = Date.now()
  const windowMs = 60 * 1000  // 1 分鐘
  const limit = 5             // 最多 5 次登入嘗試

  // 清掉 1 分鐘前的記錄
  await redis.zremrangebyscore(key, 0, now - windowMs)
  // 加入這次請求（score = 時間戳）
  await redis.zadd(key, now, `${now}`)
  // 算目前窗口內的總數
  const count = await redis.zcard(key)
  // TTL 保底
  await redis.expire(key, 60)

  return count <= limit
}

app.post('/login', async (req, res) => {
  const { email, password } = req.body
  const allowed = await checkLoginRateLimit(email)

  if (!allowed) {
    return res.status(429).json({
      error: '登入嘗試次數過多，請 1 分鐘後再試'
    })
  }

  // 正常登入邏輯...
})
```

### 架構流程

```
t=0s  嘗試登入 → ZSet: [0]           → count=1 → 放行
t=10s 嘗試登入 → ZSet: [0, 10]       → count=2 → 放行
t=20s 嘗試登入 → ZSet: [0, 10, 20]   → count=3 → 放行
t=30s 嘗試登入 → count=4 → 放行
t=40s 嘗試登入 → count=5 → 放行
t=50s 嘗試登入 → count=6 → 429 擋掉

t=61s 嘗試登入 → 清掉 t=0 的記錄 → count=5 → 放行（窗口滑動了）
t=71s 嘗試登入 → 清掉 t=10 的記錄 → count=5 → 放行
```

Sliding Window 解決了邊界攻擊：任意 1 分鐘內都最多 5 次，不管從哪個時間點算起。

---

## Case 3：發文防洗版（Token Bucket）

### 情境

```
社群平台，限制用戶不能連續狂發文
但允許偶爾爆發（例如活動時一次發幾篇）
Fixed Window 太死板：每小時 10 篇，整點前後集中發就被擋
Token Bucket 比較自然：有累積的「點數」可以花
```

### 概念

```
每個用戶有一個桶，容量 10 個 token
每分鐘自動補 1 個 token（最多補到 10）
發一篇文消耗 1 個 token
token 用完就得等
```

```typescript
async function consumeToken(userId: string): Promise<boolean> {
  const key = `token_bucket:${userId}`
  const capacity = 10    // 桶的最大容量
  const refillRate = 1   // 每分鐘補 1 個
  const now = Date.now()

  const data = await redis.hgetall(key)

  let tokens = data.tokens ? parseFloat(data.tokens) : capacity
  let lastRefill = data.lastRefill ? parseInt(data.lastRefill) : now

  // 計算應該補多少
  const elapsed = (now - lastRefill) / 1000 / 60  // 過了幾分鐘
  tokens = Math.min(capacity, tokens + elapsed * refillRate)

  if (tokens < 1) {
    return false  // 沒 token，擋掉
  }

  // 消耗 1 個 token
  tokens -= 1
  await redis.hset(key, { tokens: tokens.toString(), lastRefill: now.toString() })
  await redis.expire(key, 3600)

  return true
}
```

---

## 三種算法比較

| 算法 | 特性 | 適合場景 |
|------|------|---------|
| Fixed Window | 簡單，有邊界攻擊漏洞 | 一般 API 限流 |
| Sliding Window | 精確，記憶體稍多（存每筆時間戳）| 登入保護、安全場景 |
| Token Bucket | 允許短暫爆發，平滑限流 | 發文、上傳、使用者行為限制 |

---

## 完整架構圖

```
請求進來
  ↓
Rate Limiter Middleware
  │
  ├─ 取 key（by IP / userId / API key）
  ├─ Redis INCR / ZADD / HSET
  ├─ 超過門檻 → 429 Too Many Requests
  │              + Retry-After header（告訴用戶幾秒後再試）
  └─ 未超過 → next()
  ↓
業務邏輯
```

**回傳 Header 讓前端知道剩餘額度**

```typescript
res.set({
  'X-RateLimit-Limit': '100',
  'X-RateLimit-Remaining': String(100 - count),
  'X-RateLimit-Reset': String(Math.ceil(Date.now() / 60000) * 60),
  'Retry-After': '60'  // 429 時才加這個
})
```
