# Session 管理（Redis）

> 問題：多台 App Server 各自存 Session → 用戶被分到不同 Server 就登出
> 解法：Session 存 Redis，所有 Server 共享

---

## 什麼是 Session 問題

```
沒有 Redis 的情況（Session 存在記憶體）：

  User 登入 → Server A 建立 Session
  下一個請求 → Load Balancer 分到 Server B
  Server B：找不到 Session → 要求重新登入

  用戶：「我剛才明明登入了？」
```

---

## Case 1：電商結帳流程

### 情境

```
用戶加購物車 → 結帳 → 付款 → 完成
整個流程可能跨越多個請求，打到不同 Server

Server A 記了購物車
Server B 記了登入狀態
Server C 處理付款

沒有共享 Session → 各自為政 → 結帳失敗
```

### 解法

```
key   = "session:{token}"
value = { userId, cartId, permissions, ... }
TTL   = 7 * 24 * 3600（7 天，登入有效期）
```

```typescript
// 登入時建立 Session
async function createSession(userId: string, permissions: string[]) {
  const token = crypto.randomUUID()
  const sessionData = {
    userId,
    permissions,
    createdAt: Date.now(),
  }

  await redis.setex(
    `session:${token}`,
    7 * 24 * 3600,
    JSON.stringify(sessionData)
  )

  return token  // 回給前端，存在 Cookie
}

// 每個請求驗證 Session
async function getSession(token: string) {
  const data = await redis.get(`session:${token}`)
  if (!data) return null  // Session 不存在或過期
  return JSON.parse(data)
}

// 登出時刪除 Session
async function destroySession(token: string) {
  await redis.del(`session:${token}`)
}
```

### 架構流程

```
User 登入：
  → Server A 驗證帳號密碼
  → 建立 session:abc123 寫入 Redis
  → 回傳 token（Cookie）

User 下一個請求（被分到 Server B）：
  → Server B 從 Cookie 拿 token
  → 去 Redis 查 session:abc123
  → 拿到 userId + permissions → 正常處理

User 登出：
  → DEL session:abc123
  → 清 Cookie
  → 任何 Server 再收到這個 token → Redis 查不到 → 401
```

---

## Case 2：強制登出（踢人）

### 情境

```
用戶帳號被盜 → 管理員需要強制登出所有裝置
用戶改密碼 → 舊的 Session 要全部失效
```

### 解法：記錄所有 Session token

```typescript
// 登入時，順便把 token 記錄到用戶的 Session 集合
async function createSession(userId: string, permissions: string[]) {
  const token = crypto.randomUUID()
  const sessionData = { userId, permissions, createdAt: Date.now() }

  await redis.setex(`session:${token}`, 7 * 24 * 3600, JSON.stringify(sessionData))

  // 把這個 token 加到用戶的 session 集合
  await redis.sadd(`user:sessions:${userId}`, token)
  await redis.expire(`user:sessions:${userId}`, 7 * 24 * 3600)

  return token
}

// 強制登出該用戶所有裝置
async function destroyAllSessions(userId: string) {
  const tokens = await redis.smembers(`user:sessions:${userId}`)

  await Promise.all(tokens.map(token => redis.del(`session:${token}`)))
  await redis.del(`user:sessions:${userId}`)
}
```

### 架構流程

```
管理員踢人 / 用戶改密碼：
  → SMEMBERS user:sessions:123 → ["abc", "def", "ghi"]（三台裝置）
  → DEL session:abc, session:def, session:ghi
  → DEL user:sessions:123

三台裝置下次請求：
  → Redis 查不到 session → 全部 401 → 跳登入頁
```

---

## Case 3：Session vs JWT 選哪個

### 情境

```
新專案要設計認證系統，Session + Redis 還是 JWT？
```

### 比較

```
Session（Redis）：
  ✅ 可以立刻失效（DEL 就好）
  ✅ 可以查詢所有登入裝置
  ✅ 不暴露用戶資料（token 只是隨機字串）
  ❌ 每個請求都要查 Redis（多一次 I/O）
  ❌ Redis 掛掉 → 所有人登出

JWT：
  ✅ 無狀態，不需要查 Redis
  ✅ 可以帶 payload（userId、role 直接解出來）
  ❌ 無法立刻失效（等 exp 到期才行）
  ❌ token 被偷 → 無法撤銷
```

### 實務做法：Short-lived JWT + Redis 黑名單

```typescript
// JWT exp 設很短（15 分鐘）
// 登出時把 token 加進黑名單

async function logout(token: string) {
  const decoded = jwt.decode(token) as { exp: number }
  const ttl = decoded.exp - Math.floor(Date.now() / 1000)

  // 黑名單只需存到 JWT 本來的過期時間
  if (ttl > 0) {
    await redis.setex(`blacklist:${token}`, ttl, '1')
  }
}

async function verifyToken(token: string) {
  // 先查黑名單
  const blocked = await redis.get(`blacklist:${token}`)
  if (blocked) return null

  return jwt.verify(token, process.env.JWT_SECRET!)
}
```

```
正常情況：只驗 JWT（不查 Redis）
登出後：查黑名單（查 Redis）
15 分鐘後 JWT 自然過期：黑名單也自動清掉（TTL 到期）
```

---

## 三種方案比較

| 方案 | 適合場景 | 即時登出 | Redis 依賴 |
|------|---------|---------|-----------|
| Session + Redis | 需要管理登入狀態（後台、電商）| ✅ | 強依賴 |
| JWT only | 無狀態 API、Microservices | ❌ | 不需要 |
| JWT + 黑名單 | 兼顧無狀態 + 可撤銷 | ✅ | 弱依賴（只登出時用）|
