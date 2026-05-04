# Redis 深度筆記

> Redis = Remote Dictionary Server
> 本質：in-memory key-value store，可同時扮演 Cache、Database、Message Broker

---

## 一、Redis 是什麼

```
傳統架構：
  App → DB（磁碟）→ 回傳結果
  問題：磁碟 I/O 慢，高並發下 DB 成瓶頸

加了 Redis：
  App → Redis（RAM）→ 命中 → 直接回傳
             ↓ 沒命中
            DB → 寫回 Redis → 回傳
  優點：熱資料全走記憶體，DB 壓力大幅下降
```

**速度對比**

| 儲存層 | 讀取延遲 |
|--------|---------|
| 磁碟 HDD | ~10ms |
| 磁碟 SSD | ~0.1ms |
| Redis（RAM）| ~0.01ms（10 微秒）|
| CPU L1 Cache | ~1 奈秒 |

Redis 比 SSD 快 10 倍，這就是為什麼 Cache 命中能大幅降延遲。

---

## 二、內部運作原理

### 單執行緒 Event Loop

```
Redis 主要操作跑在單一執行緒

請求 1 ─┐
請求 2 ─┤→ Event Loop → 依序執行 → 回傳
請求 3 ─┘
```

**為什麼單執行緒反而快？**

```
多執行緒的成本：
  - Thread 切換（Context Switch）有開銷
  - 共享資料需要鎖（Lock），等鎖也是成本
  - Deadlock 問題

Redis 單執行緒：
  - 沒有鎖的開銷
  - 沒有 Context Switch
  - 操作都是 O(1) 或 O(log n)，單次執行極快
  - 瓶頸通常是網路 I/O，不是 CPU
```

注意：Redis 6.0 之後，網路 I/O 改成多執行緒處理，但資料操作仍是單執行緒。

**單執行緒的代價**

```
慢操作會阻塞所有請求：

危險指令（生產環境禁用）：
  KEYS *          → 掃所有 key，O(N)，資料多就卡死
  SMEMBERS（大Set）→ 回傳幾百萬筆，記憶體爆炸
  FLUSHALL        → 同步清除所有資料，期間 block

安全替代：
  KEYS * → SCAN 0 MATCH * COUNT 100（分批掃，不 block）
```

### 記憶體管理

```
Redis 預設把所有資料放 RAM
當記憶體快滿時，根據 maxmemory-policy 決定怎麼處理：

allkeys-lru    → 淘汰最久沒用的 key（最常用）
volatile-lru   → 只淘汰有設 TTL 的 key 中最久沒用的
allkeys-random → 隨機淘汰
noeviction     → 拒絕寫入，回傳錯誤（當 DB 用時設這個）
```

---

## 三、資料結構與使用場景

### String

```
最基本的類型，值可以是字串或數字。

SET user:123:name "Alice"
GET user:123:name          → "Alice"
INCR page:views            → 原子 +1，不怕並發
INCRBY score:user:123 10   → 原子 +10
SETEX session:abc123 3600 "{user_id: 1}"  → 設值 + TTL

場景：計數器、API Response 快取、Session
```

### Hash

```
一個 key 對應多個 field-value，像是一個物件。

HSET user:123 name "Alice" age 30 email "alice@example.com"
HGET user:123 name          → "Alice"
HGETALL user:123            → 全部欄位
HINCRBY user:123 loginCount 1

場景：儲存物件（不用把整個 JSON 序列化）
優點：可以只更新單一欄位，不用讀整個物件再改
```

### List

```
有序列表，支援從兩端 push/pop（雙向 Queue）。

LPUSH queue:email "{payload1}"   → 從左推入
RPUSH queue:email "{payload2}"   → 從右推入
RPOP queue:email                 → 從右取出
BRPOP queue:email 0              → Blocking pop（等到有資料才回傳）
LRANGE queue:email 0 9           → 取前 10 筆

場景：消息隊列、最新動態列表（取最近 N 筆）
```

### Set

```
不重複的集合，O(1) 新增/刪除/查詢。

SADD online:users "user:123"
SADD online:users "user:456"
SISMEMBER online:users "user:123"  → 1（存在）
SMEMBERS online:users              → 全部（小 Set 才用）
SCARD online:users                 → 計算人數

集合運算：
SUNION set1 set2    → 聯集
SINTER set1 set2    → 交集（共同好友）
SDIFF set1 set2     → 差集

場景：誰在線上、去重（已讀 / 已按讚）、共同好友
```

### Sorted Set（ZSet）

```
每個元素帶分數（score），自動依分數排序。

ZADD leaderboard 9500 "user:123"
ZADD leaderboard 8200 "user:456"
ZADD leaderboard 9800 "user:789"

ZREVRANGE leaderboard 0 9 WITHSCORES   → 取前 10 名（高到低）
ZRANK leaderboard "user:123"           → 我的排名
ZINCRBY leaderboard 100 "user:123"     → 加分

場景：排行榜、延遲隊列（score 設成時間戳，定時取到期任務）
```

### Stream（Redis 5.0+）

```
持久化的消息流，類似 Kafka 的輕量版。

XADD events * action "purchase" user_id "123"
XREAD COUNT 10 STREAMS events 0   → 讀取

場景：事件日誌、需要消費確認的消息隊列
比 List 強：支援 Consumer Group，多個 Consumer 分擔處理
```

---

## 四、作為 Cache 使用

### Cache-Aside（最常見）

```
讀：
  1. 查 Redis
  2. 命中（Hit）→ 直接回傳
  3. 沒中（Miss）→ 查 DB → 寫回 Redis（設 TTL）→ 回傳

寫：
  1. 寫 DB
  2. 刪 Redis（讓下次讀重建，保持一致）
     ← 為什麼是刪而不是更新？
        更新有競態條件：兩個 Thread 同時更新，後寫的可能是舊值
        刪除後讓讀重建，永遠拿 DB 最新值
```

Python 範例：

```python
def get_user(user_id):
    key = f"user:{user_id}"
    cached = redis.get(key)
    if cached:
        return json.loads(cached)

    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(key, 300, json.dumps(user))  # TTL 5 分鐘
    return user
```

### Write-Through

```
寫：同時寫 Cache + DB（同步）

優：Cache 永遠是最新的
缺：每次寫都多一步，寫入變慢
適用：讀多寫少，且讀到舊資料代價高
```

### Write-Behind（Write-Back）

```
寫：先寫 Cache，非同步批次寫 DB

優：寫入極快
缺：Cache 掛掉可能丟資料
適用：高寫入量、允許少量丟失（e.g. 計數、日誌）
```

### TTL 設計

```
設太短：Cache 命中率低，DB 壓力大
設太長：資料不新鮮

原則：
  - 靜態資料（商品描述）→ 長 TTL（1 小時 ~ 1 天）
  - 動態資料（購物車）→ 短 TTL（5 ~ 30 分鐘）
  - 用戶 Session → TTL = 登入有效期（e.g. 7 天）
  - 加隨機抖動：TTL = base + random(0, 60)
    → 防止大量 key 同時過期（Cache Stampede）
```

---

## 五、常見問題與解法

### Cache Stampede（快取雪崩）

```
問題：
  大量 key 同時過期（或 Redis 重啟）
  → 所有請求同時打 DB
  → DB 過載，甚至崩潰

解法：
  1. TTL 加隨機抖動（最簡單）
  2. 互斥鎖（Mutex）：
     Cache miss 時，只讓第一個請求去打 DB
     其他請求等待或回傳舊值（stale-while-revalidate）
  3. 預熱：系統啟動時主動把熱資料寫入 Cache
```

### Cache Penetration（快取穿透）

```
問題：
  查詢不存在的資料（e.g. user_id = 99999999）
  → Cache miss → 打 DB → DB 也沒有
  → 每次都穿透，DB 被打爛
  → 惡意攻擊：故意查不存在的 key

解法：
  1. 空值快取：DB 查不到也寫 Redis（值 = null，TTL 短）
  2. Bloom Filter：
     預先把所有合法 ID 放入 Bloom Filter
     查詢前先判斷是否可能存在，不存在直接回 404
     （Bloom Filter 可能誤判存在，但不會漏判不存在）
```

### Hot Key（熱點問題）

```
問題：
  某個 key 被瘋狂讀取（e.g. 熱門商品、明星動態）
  → 單一 Redis node 成瓶頸（網路頻寬打滿）

解法：
  1. 本地快取（App Server 記憶體）：
     最熱的 key 在 App 記憶體層再 Cache 一層
     → 完全不打 Redis
     缺：多台 App Server 各自快取，一致性問題

  2. Key 分散：
     "product:hot:1" → "product:hot:1:shard0"、":shard1"...
     讀取時隨機選一個 shard
     → 分散到多個 Redis node

  3. 升級 Redis Cluster，增加 node
```

---

## 六、作為 Database 使用

當 Redis 是主要儲存（不只是 Cache），需要開 **Persistence**。

### RDB（快照，Redis Database）

```
每隔固定時間，把記憶體狀態 dump 成 .rdb 二進位檔案

設定（redis.conf）：
  save 900 1    → 900 秒內有 1 次寫入就存
  save 300 10   → 300 秒內有 10 次寫入就存
  save 60 10000 → 60 秒內有 10000 次寫入就存

優：
  - 檔案小（二進位壓縮）
  - 重啟速度快（直接載入快照）
  - 適合備份

缺：
  - 快照間隔內 crash → 丟失最後 N 分鐘資料
```

### AOF（Append-Only File）

```
每個寫入指令都 append 到 log 檔

SET user:123 "Alice"   → 寫一行
INCR counter           → 寫一行
...

fsync 策略：
  always    → 每個指令都 fsync，最安全，最慢
  everysec  → 每秒 fsync（預設），最多丟 1 秒資料
  no        → 讓 OS 決定，最快，可能丟多

AOF Rewrite：
  檔案越來越大 → 定期重寫（只保留最終狀態）
  BGREWRITEAOF 觸發，背景執行，不影響服務

優：幾乎不丟資料
缺：檔案大，重啟慢
```

### 實際生產設定

```
兩個都開：
  AOF → 日常保護（最多丟 1 秒）
  RDB → 定期備份（方便還原到某個時間點）

重啟恢復優先順序：AOF > RDB（AOF 更完整）
```

### 適合當主 DB 的場景

```
Session 儲存     → TTL 自動清過期，KV 完美匹配
排行榜           → Sorted Set 原生支援，SQL 做同樣事很慢
Rate Limiting    → INCR + EXPIRE 原子，SQL 做需要 Transaction
分散式鎖         → SET NX EX 原子，無法用普通 DB 安全實現
即時狀態         → 誰在線上、誰在打字
短連結映射       → short_code → original_url（讀多）
```

### 不適合當主 DB 的場景

```
需要 JOIN / 複雜查詢  → 用 PostgreSQL / MySQL
資料量超過記憶體      → RAM 貴，TB 級資料不適合
需要強 ACID           → Redis Transaction 相對弱
關聯式資料            → 用 RDB
```

---

## 七、高可用架構

### Sentinel（主從 + 自動切換）

```
Master（寫）
   ↓ 複製
Replica 1（讀）
Replica 2（讀）

Sentinel 1 ─┐
Sentinel 2 ─┤→ 監控 Master 心跳
Sentinel 3 ─┘   Master 掛了 → 自動選出新 Master（Failover）

適用：中小規模，資料量單機放得下
```

### Redis Cluster（分散式）

```
Cluster 把資料分成 16384 個 slot
每個 node 負責部分 slot

Node 1（slot 0-5460）     → Master + Replica
Node 2（slot 5461-10922） → Master + Replica
Node 3（slot 10923-16383）→ Master + Replica

Key → hash(key) % 16384 → 路由到對應 node
Client 會快取 slot 對應表，直接打正確的 node

適用：超大規模，資料量超出單機記憶體
```

---

## 八、Redis vs 其他方案

| | Redis | Memcached | MySQL（InnoDB）| PostgreSQL |
|--|-------|-----------|---------------|-----------|
| 速度 | 極快（RAM）| 極快（RAM）| 慢（磁碟）| 慢（磁碟）|
| 資料結構 | 豐富 | 只有 String | Table | Table |
| 持久化 | 有（RDB/AOF）| 無 | 有 | 有 |
| 複雜查詢 | 無 | 無 | 強 | 最強 |
| 適合 | Cache + 簡單 DB | 純 Cache | OLTP | OLTP + 複雜查詢 |

---

## 九、實際使用範例

### Rate Limiting

```python
import time

def is_rate_limited(user_id: str, limit: int = 100) -> bool:
    key = f"rate:{user_id}:{int(time.time() // 60)}"  # 每分鐘一個 key
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, 60)  # 第一次設 TTL
    return count > limit
```

### 分散式鎖

```python
import uuid

def acquire_lock(resource: str, ttl: int = 30) -> str | None:
    lock_id = str(uuid.uuid4())
    key = f"lock:{resource}"
    # NX = 不存在才設（原子），EX = 自動過期（防止忘記釋放）
    acquired = redis.set(key, lock_id, nx=True, ex=ttl)
    return lock_id if acquired else None

def release_lock(resource: str, lock_id: str):
    key = f"lock:{resource}"
    current = redis.get(key)
    # 確認是自己的鎖才刪（防止誤刪別人的鎖）
    if current == lock_id:
        redis.delete(key)
    # 注意：嚴格原子性需用 Lua Script（redis.execute_script），
    # 確保 GET 和 DEL 之間不被搶走
```

### 排行榜

```python
# 加分
redis.zincrby("leaderboard:weekly", 100, f"user:{user_id}")

# 取前 10 名
top10 = redis.zrevrange("leaderboard:weekly", 0, 9, withscores=True)

# 查自己排名
rank = redis.zrevrank("leaderboard:weekly", f"user:{user_id}")
```

---

## 十、連線設定實戰

### 環境變數（所有環境共用）

```bash
# .env
REDIS_URL=redis://localhost:6379           # 本地
REDIS_URL=redis://:password@host:6379      # 有密碼
REDIS_URL=rediss://host:6380               # TLS（rediss://，多一個 s）
```

---

### Node.js — ioredis（業界最常用）

```bash
npm install ioredis
```

**基本連線**

```typescript
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)

// 測試連線
await redis.ping()  // → "PONG"
```

**建議寫法：單例模式（整個 App 共用一個連線）**

```typescript
// lib/redis.ts
import Redis from 'ioredis'

let redis: Redis

if (process.env.NODE_ENV === 'production') {
  redis = new Redis(process.env.REDIS_URL!)
} else {
  // 開發環境：避免 hot reload 每次建新連線
  if (!(global as any).__redis) {
    (global as any).__redis = new Redis(process.env.REDIS_URL!)
  }
  redis = (global as any).__redis
}

export default redis
```

**使用**

```typescript
import redis from '@/lib/redis'

// 寫
await redis.set('key', 'value')
await redis.setex('key', 300, 'value')   // 設 TTL 300 秒
await redis.set('key', 'value', 'EX', 300)  // 同上，另一種寫法

// 讀
const value = await redis.get('key')     // string | null

// 刪
await redis.del('key')

// 計數
await redis.incr('counter')
await redis.incrby('counter', 5)

// Hash
await redis.hset('user:123', 'name', 'Alice', 'age', '30')
await redis.hget('user:123', 'name')     // "Alice"
await redis.hgetall('user:123')          // { name: 'Alice', age: '30' }

// JSON（ioredis 沒有原生 JSON，要自己序列化）
await redis.set('user:123', JSON.stringify({ name: 'Alice' }))
const raw = await redis.get('user:123')
const user = raw ? JSON.parse(raw) : null
```

**連線設定選項**

```typescript
const redis = new Redis({
  host: 'localhost',
  port: 6379,
  password: 'yourpassword',
  db: 0,                    // Redis 有 0-15 個 DB，預設 0
  maxRetriesPerRequest: 3,  // 失敗重試次數
  retryStrategy(times) {
    return Math.min(times * 100, 3000)  // 指數退避，最多等 3 秒
  },
  lazyConnect: true,        // 不立刻連，第一次操作才連
})
```

---

### Next.js + Upstash（Serverless 首選）

Vercel 部署時用 Upstash，原因：
- 傳統 Redis 是長連線，Serverless function 每次冷啟動都建新連線 → 連線數爆炸
- Upstash 用 HTTP 協議，無狀態，天生適合 Serverless

```bash
npm install @upstash/redis
```

```typescript
// lib/redis.ts
import { Redis } from '@upstash/redis'

const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
})

export default redis
```

```bash
# .env
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=AxxxXXX
```

**使用方式跟 ioredis 幾乎一樣**

```typescript
await redis.set('key', 'value', { ex: 300 })  // 注意 Upstash 用物件設 TTL
const value = await redis.get('key')

// Upstash 原生支援 JSON，不用手動序列化
await redis.set('user:123', { name: 'Alice', age: 30 })
const user = await redis.get<{ name: string; age: number }>('user:123')
```

---

### Cache-Aside 完整範例（Next.js API Route）

```typescript
// app/api/users/[id]/route.ts
import redis from '@/lib/redis'
import db from '@/lib/db'

export async function GET(req: Request, { params }: { params: { id: string } }) {
  const { id } = params
  const cacheKey = `user:${id}`

  // 1. 查 Redis
  const cached = await redis.get(cacheKey)
  if (cached) {
    return Response.json(JSON.parse(cached as string))
  }

  // 2. 查 DB
  const user = await db.user.findUnique({ where: { id } })
  if (!user) {
    return Response.json({ error: 'Not found' }, { status: 404 })
  }

  // 3. 寫回 Redis，TTL 5 分鐘
  await redis.setex(cacheKey, 300, JSON.stringify(user))

  return Response.json(user)
}

export async function PUT(req: Request, { params }: { params: { id: string } }) {
  const { id } = params
  const body = await req.json()

  // 1. 更新 DB
  const user = await db.user.update({ where: { id }, data: body })

  // 2. 刪 Redis（讓下次讀重建）
  await redis.del(`user:${id}`)

  return Response.json(user)
}
```

---

### Python — redis-py

```bash
pip install redis
```

```python
import redis
import json
import os

# 連線
r = redis.from_url(os.environ['REDIS_URL'], decode_responses=True)
# decode_responses=True → 自動把 bytes 轉成 str，省得每次 .decode()

# 基本操作
r.set('key', 'value', ex=300)     # 設值 + TTL
value = r.get('key')               # str | None

# JSON
r.set('user:123', json.dumps({'name': 'Alice'}), ex=300)
user = json.loads(r.get('user:123'))

# Pipeline（批次操作，一次送）
pipe = r.pipeline()
pipe.set('a', 1)
pipe.set('b', 2)
pipe.incr('counter')
pipe.execute()   # 一次送出三個指令
```

---

### 本地開發（Docker）

```bash
# 啟動 Redis
docker run -d --name redis -p 6379:6379 redis:7-alpine

# 進 CLI 測試
docker exec -it redis redis-cli
> ping        → PONG
> set foo bar
> get foo     → "bar"
```

```yaml
# docker-compose.yml（搭配整個 App 用）
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass yourpassword  # 設密碼
```

---

## 十一、部署架構與備援

### Redis 永遠獨立部署

```
錯誤（Redis 跟 App 混在一起）：
  App Server 1 → Redis A（各自的）
  App Server 2 → Redis B（各自的）
  → 資料不共用，Cache 失去意義

正確：
  App Server 1 ─┐
  App Server 2 ─┤→ Redis（獨立機器）→ PostgreSQL
  App Server 3 ─┘
```

### 生產環境備援

**Sentinel（中小規模）**

```
Master（讀寫）
  ↓ 非同步複製
Replica 1、Replica 2

Sentinel × 3 → 監控 Master 心跳 → Master 掛了自動投票選新 Master

App 連 Sentinel，不直接連 Master
→ Master 換了，App 不用改設定，Sentinel 自動通知
→ Failover 約 10-30 秒，期間短暫不可寫
```

**Cluster（大規模）**

```
Node 1 Master + Replica → 負責 slot 0-5460
Node 2 Master + Replica → 負責 slot 5461-10922
Node 3 Master + Replica → 負責 slot 10923-16383

某個 Master 掛掉 → 自己的 Replica 自動接手
App 不感知，ioredis 自動路由
```

### 實際生產架構

```
Internet → Load Balancer（AWS ALB）
                ↓
         App Server × N（ECS / K8s）
           │                  │
           ↓                  ↓
     Redis Cluster        PostgreSQL Aurora
    （AWS ElastiCache）    （AWS RDS）
      Master × 3            Primary + Replica
      Replica × 3           跨 3 AZ
      跨 3 AZ
```

每一層都跨多個 AZ，任一 AZ 掛掉服務不中斷。

### 託管 vs 自建

| | 自建（K8s + Helm）| 託管（ElastiCache / Upstash）|
|--|------------------|------------------------------|
| 備援設定 | 自己設 Sentinel/Cluster | 平台自動處理 |
| Failover | 自己監控 | 自動，Dashboard + Alert 內建 |
| 成本 | 低（只付機器）| 高一點 |
| 維運成本 | 高 | 幾乎沒有 |
| 適合 | 大公司、有 SRE 團隊 | 中小公司首選 |

---

## 十二、微服務中的 Redis 設計

多個服務怎麼用 Redis，四種做法：

**Option 1：各服務獨立 Redis（最乾淨）**

```
User Service    → Redis A
Order Service   → Redis B
Payment Service → Redis C
```

完全隔離，一個服務的 Hot Key 不影響其他服務，可獨立擴展。
大公司（Netflix、Uber）的做法，成本較高。

**Option 2：共用 Redis，DB number 隔開**

```typescript
const userRedis  = new Redis({ url: REDIS_URL, db: 0 })
const orderRedis = new Redis({ url: REDIS_URL, db: 1 })
```

省錢，一台搞定。缺點：Cluster 模式不支援多 DB，沒辦法獨立擴展。

**Option 3：共用 Redis，key prefix 隔開（最常見折衷）**

```
user:cache:123、user:session:abc
order:cache:456、order:lock:789
```

最簡單，沒有真正隔離，適合服務少、流量不大的情況。

**Option 4：同一服務連多個 Redis（不同用途）**

```typescript
export const cacheRedis   = new Redis(process.env.REDIS_CACHE_URL!)
export const queueRedis   = new Redis(process.env.REDIS_QUEUE_URL!)
export const sessionRedis = new Redis(process.env.REDIS_SESSION_URL!)
```

原因：不同用途需要不同 eviction policy，同一台只能設一種：
- Cache → `allkeys-lru`（滿了淘汰最舊的）
- Queue / Session → `noeviction`（滿了報錯，不能亂刪）

**選擇原則**

```
服務少、流量小   → Option 3（key prefix）
服務中等        → Option 2（DB number）
服務多、高流量  → Option 1（各自獨立）
需要不同 policy → Option 4（多個 Redis URL）
```

---

## 總結

```
Redis 的核心思想：
  把「常讀、少改、算得出來」的資料放進記憶體
  讓 DB 只處理「需要持久化、需要查詢」的資料

三個角色：
  Cache  → 加速 DB，TTL 自動過期，資料可重建
  DB     → 開 Persistence，適合 KV 天然場景（Session、排行榜、鎖）
  Broker → List / Stream，輕量消息隊列

最大限制：
  RAM 很貴，不適合存大量冷資料
  不適合複雜關聯查詢
```
