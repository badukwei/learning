# 身份驗證完整流程

> JWT（Access Token + Refresh Token）+ Redis
> 無狀態驗證 + 可撤銷登入的標準做法

---

## 角色說明

```
Access Token：短效（15 分鐘），每個請求帶著，後端直接解碼
Refresh Token：長效（7 天），存在 HttpOnly Cookie，只用來換新 Access Token
Redis：存 Refresh Token，讓登出可以即時生效
```

---

## 流程一：登入

```
User 輸入帳號密碼
  ↓
POST /auth/login { email, password }
  ↓
後端驗證密碼（bcrypt compare）
  ↓
產生兩個 token：
  accessToken  = JWT({ userId, role }, exp: 15m)
  refreshToken = crypto.randomUUID()（隨機字串，不是 JWT）
  ↓
Redis：SET refresh:{refreshToken} {userId}  EX 7*24*3600
  ↓
回傳：
  Body：{ accessToken }         ← 前端存在記憶體
  Cookie：refreshToken=xxx      ← HttpOnly，JS 拿不到
```

```typescript
app.post('/auth/login', async (req, res) => {
  const { email, password } = req.body
  const user = await db.user.findUnique({ where: { email } })

  if (!user || !await bcrypt.compare(password, user.passwordHash)) {
    return res.status(401).json({ error: '帳號或密碼錯誤' })
  }

  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  )

  const refreshToken = crypto.randomUUID()
  await redis.setex(`refresh:${refreshToken}`, 7 * 24 * 3600, user.id)

  res.cookie('refreshToken', refreshToken, {
    httpOnly: true,   // JS 無法讀取
    secure: true,     // 只走 HTTPS
    sameSite: 'strict',
    maxAge: 7 * 24 * 60 * 60 * 1000,
  })

  res.json({ accessToken })
})
```

---

## 流程二：正常請求

```
前端每個 Request：
  Authorization: Bearer {accessToken}
  ↓
後端 Middleware 直接解碼 JWT
  ↓
驗簽名 + 檢查 exp
  ├─ 有效 → 拿出 { userId, role } → 往下處理
  └─ 過期 → 401（前端去換新 token）

不查 Redis，純解碼，最快
```

```typescript
function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const token = req.headers.authorization?.replace('Bearer ', '')
  if (!token) return res.status(401).json({ error: '未登入' })

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!) as JwtPayload
    req.userId = payload.userId
    req.role = payload.role
    next()
  } catch {
    res.status(401).json({ error: 'Token 無效或已過期' })
  }
}
```

---

## 流程三：換新 Access Token

```
Access Token 過期（15 分鐘）
  ↓
前端自動發：POST /auth/refresh
  Cookie 自動帶 refreshToken（HttpOnly）
  ↓
後端：
  1. 從 Cookie 拿 refreshToken
  2. 查 Redis：GET refresh:{refreshToken} → userId
  3. Redis 有 → 發新 accessToken
  4. Redis 沒有 → 401（已登出或過期）
```

```typescript
app.post('/auth/refresh', async (req, res) => {
  const refreshToken = req.cookies.refreshToken
  if (!refreshToken) return res.status(401).json({ error: '未登入' })

  const userId = await redis.get(`refresh:${refreshToken}`)
  if (!userId) return res.status(401).json({ error: '請重新登入' })

  const user = await db.user.findUnique({ where: { id: userId } })
  const newAccessToken = jwt.sign(
    { userId: user!.id, role: user!.role },
    process.env.JWT_SECRET!,
    { expiresIn: '15m' }
  )

  res.json({ accessToken: newAccessToken })
})
```

---

## 流程四：登出

```
POST /auth/logout
  ↓
DEL refresh:{refreshToken}（Redis 刪掉）
  ↓
清 Cookie
  ↓
前端清掉記憶體裡的 accessToken

結果：
  refreshToken → Redis 查不到 → 換不了新 token
  accessToken  → 最多再撐 15 分鐘自然過期
  （如果需要即時撤銷，把 accessToken 加黑名單）
```

```typescript
app.post('/auth/logout', async (req, res) => {
  const refreshToken = req.cookies.refreshToken
  if (refreshToken) {
    await redis.del(`refresh:${refreshToken}`)
  }

  res.clearCookie('refreshToken')
  res.json({ ok: true })
})
```

---

## 完整時序圖

```
前端                  後端                    Redis         DB
 │                     │                       │             │
 │── POST /login ──────>│                       │             │
 │                      │── 驗密碼 ─────────────────────────>│
 │                      │<─ user ───────────────────────────│
 │                      │── SET refresh:xxx ───>│             │
 │<─ accessToken ───────│                       │             │
 │  (Cookie: refresh)   │                       │             │
 │                      │                       │             │
 │── GET /api/data ─────>│                       │             │
 │  (Bearer accessToken) │                       │             │
 │                      │── jwt.verify() ✓      │             │
 │                      │── 查資料 ──────────────────────────>│
 │<─ data ──────────────│                       │             │
 │                      │                       │             │
 │  （15 分鐘後）         │                       │             │
 │── POST /auth/refresh ─>│                       │             │
 │  (Cookie: refresh)    │── GET refresh:xxx ───>│             │
 │                       │<─ userId ────────────│             │
 │<─ newAccessToken ─────│                       │             │
 │                      │                       │             │
 │── POST /auth/logout ──>│                       │             │
 │                      │── DEL refresh:xxx ───>│             │
 │<─ ok ────────────────│                       │             │
```

---

## 安全細節

```
HttpOnly Cookie：
  refreshToken 存在 HttpOnly Cookie
  → XSS 攻擊拿不到（JS 讀不到 Cookie）
  → CSRF 用 sameSite: strict 防

accessToken 存記憶體（不存 localStorage）：
  → XSS 攻擊拿不到（不在 DOM 上）
  → 頁面重整就消失 → 靠 /auth/refresh 自動補回來

HTTPS only：
  → 傳輸過程無法被截取
```

---

## 微服務架構

```
前端
  ↓（帶 accessToken）
API Gateway
  → jwt.verify()（不查 Redis，純解碼）
  → 注入 X-User-Id: 123 header
  ↓
各 Service 直接用 X-User-Id，不碰 Redis

Auth Service（獨立）：
  → 負責 /login、/logout、/refresh
  → 擁有 Refresh Token Redis 的讀寫權
  → 其他 Service 不直接碰這個 Redis
```