# System Design - Sticky Sessions & Modern Session Management

Date: 2026-04-02

---

## 問題背景：水平擴展（Horizontal Scaling）遇到的麻煩

傳統 PHP/語言預設把 Session 存在**本機磁碟**，例如 `/tmp/sess_abc123`。

單機沒問題。但加了 Load Balancer 後：

```
User → Load Balancer → Server A (session 存這裡)
User → Load Balancer → Server B (沒有 session！)  ← 被登出了
```

---

## 方案一：Sticky Sessions（黏性會話）

> 讓 Load Balancer 記住「這個 User 要送去哪台 Server」

### 運作方式

Load Balancer 在 Cookie 裡塞一個識別碼，之後每次請求都送回同一台 Server。

```
瀏覽器 Cookie: SERVERID=server-a-42

Load Balancer 看到 → 送去 Server A
```

實際上 LB 不會直接存 IP（有安全疑慮），而是存一個隨機大數字，內部維護對照表：

```
Cookie: lb_route=x7f3k9 → 對應 Server A
```

### 優點
- 不需要改後端程式碼
- 不需要建共享存儲
- 快速部署

### 缺點
- **負載不均**：重量級用戶黏在 Server A，Server B 閒置
- **依賴 Cookie**：用戶停用 Cookie 就壞了
- **Server 掛掉 = Session 消失**：被黏在那台的用戶全部被登出

---

## 方案二：現代解法 — 中央化 Session 儲存

把 Session 從本機磁碟移到**所有 Server 都看得到的地方**。

### 2a. Redis / Memcached（最常見）

```
Server A ──┐
Server B ──┼──→ Redis (Session Store)
Server C ──┘

User 不管被送去哪台，Session 都從 Redis 讀
```

**範例（Node.js + Express + Redis）：**
```js
import session from 'express-session'
import RedisStore from 'connect-redis'
import { createClient } from 'redis'

const redisClient = createClient({ url: 'redis://localhost:6379' })
await redisClient.connect()

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'your-secret',
  resave: false,
  saveUninitialized: false,
}))
```

任何 Server 拿到請求，都去 Redis 撈 Session，完全不需要 Sticky Sessions。

### 2b. JWT（JSON Web Token）— 無狀態解法

> Server 不存 Session，Token 直接帶在 Header 裡

```
登入成功 → Server 簽發 JWT → 存在瀏覽器 localStorage / Cookie

之後每次請求：
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...

任何 Server 收到 → 驗 Token 簽名 → 知道是誰 → 不需要查資料庫
```

**優點**：真正無狀態，Scale 到 100 台 Server 也沒問題
**缺點**：Token 一旦簽發就無法即時撤銷（登出只是刪前端的 Token，Server 端 Token 在過期前還是有效）

### 2c. 資料庫存 Session（MySQL / PostgreSQL）

存在 DB 的 sessions table，所有 Server 共享。

比 Redis 慢，但資料比較持久，適合不需要極高速讀取的場景。

---

## 各方案比較

| 方案 | 適合場景 | 缺點 |
|------|----------|------|
| Sticky Sessions | 快速部署、流量不大 | 負載不均、Server 掛掉 Session 消失 |
| Redis Session | 大多數 Web App | 多一個 Redis 要維護 |
| JWT | API、微服務、SPA | 無法即時撤銷 Token |
| DB Session | 需要持久化、低頻讀取 | 速度慢，高流量會 DB 瓶頸 |

---

## 不同語言的實作差異

### PHP（舊式預設 = Sticky Sessions 的起源）
```php
// 預設存在 /tmp，水平擴展直接壞掉
session_start();
$_SESSION['user_id'] = 42;

// 改用 Redis
ini_set('session.save_handler', 'redis');
ini_set('session.save_path', 'tcp://localhost:6379');
session_start();
```

### Node.js / Express
```js
// express-session + connect-redis（見上方範例）
```

### Python / Django
```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'default'  # 指向 Redis

CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}
```

### Go
```go
// 常見用 gorilla/sessions + redistore
store, _ := redistore.NewRediStore(10, "tcp", ":6379", "", []byte("secret"))
```

---

## 實際案例：電商購物車

**問題**：用戶加了 3 件商品到購物車（存在 Server A），結帳時被 LB 導到 Server B → 購物車空了。

**解法演進**：
1. 初期小流量 → Sticky Sessions（快速解決）
2. 成長後 → 購物車資料搬到 Redis，TTL 設 30 分鐘
3. 規模再大 → 購物車直接存 DB，User 登入後永久保留

---

## 重點總結

- Sticky Sessions 是「繞過問題」，不是「解決問題」
- 現代架構幾乎都用 **Redis** 或 **JWT**
- Frontend Engineer 要知道：JWT 存哪裡有安全考量（localStorage vs HttpOnly Cookie）
  - localStorage → 方便但有 XSS 風險
  - HttpOnly Cookie → 較安全，但要注意 CSRF
