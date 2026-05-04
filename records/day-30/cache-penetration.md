# Cache Penetration（快取穿透）

> 問題：查詢不存在的資料 → Cache 永遠 Miss → 每次都打 DB
> 兩種解法：空值快取 / Bloom Filter

---

## 什麼是 Cache Penetration

```
正常流程：
  User → Redis（Hit）→ 回傳
  User → Redis（Miss）→ DB（有資料）→ 寫回 Redis → 回傳

穿透：
  User → Redis（Miss）→ DB（也沒有）→ 寫不回 Redis
  → 下次同樣查詢：Redis Miss → DB → 沒有 → ...
  → DB 被反覆打，且毫無意義
```

Cache Stampede 是「同時 Miss」，Cache Penetration 是「永遠 Miss」。

---

## Case 1：惡意爬蟲攻擊

### 情境

```
攻擊者寫腳本，不斷查詢不存在的用戶 ID
  GET /users/99999999
  GET /users/88888888
  GET /users/77777777
  ... × 每秒 10,000 個不同 ID

Redis：全部 Miss（這些 ID 根本不存在）
DB：每次都查，每次都空手而回
→ DB 連線數爆炸 → 服務掛掉
```

### 用戶流程

```
攻擊者 → GET /users/99999999 → Redis Miss → DB（找不到）→ 回 404
攻擊者 → GET /users/88888888 → Redis Miss → DB（找不到）→ 回 404
攻擊者 → GET /users/77777777 → Redis Miss → DB（找不到）→ 回 404
... 每個都不同，Cache 完全沒用上

DB：每秒 10,000 個查詢，全是無效的
```

### 解法 1：空值快取

DB 查不到，也把「沒有」這件事寫進 Redis，TTL 設短一點。

```typescript
async function getUser(userId: string) {
  const cacheKey = `user:${userId}`

  // 1. 查 Redis
  const cached = await redis.get(cacheKey)
  if (cached !== null) {
    if (cached === 'NULL') return null  // 快取的空值
    return JSON.parse(cached)
  }

  // 2. 查 DB
  const user = await db.user.findUnique({ where: { id: userId } })

  if (user) {
    // 有資料 → 正常快取
    await redis.setex(cacheKey, 3600, JSON.stringify(user))
  } else {
    // 沒資料 → 也快取，但 TTL 很短（避免正常新增的用戶被擋住）
    await redis.setex(cacheKey, 60, 'NULL')
  }

  return user
}
```

### 解法 2：Bloom Filter

查 Redis / DB 之前，先判斷這個 ID 可不可能存在。

```typescript
import { BloomFilter } from 'bloom-filters'

// 啟動時把所有合法 userId 放入 Bloom Filter
const bloom = new BloomFilter(100000, 0.01)  // 容量 10 萬，誤判率 1%

async function initBloomFilter() {
  const allUsers = await db.user.findMany({ select: { id: true } })
  allUsers.forEach(u => bloom.add(u.id))
  console.log(`Bloom Filter 初始化：${allUsers.length} 個用戶`)
}

async function getUserWithBloom(userId: string) {
  // 先問 Bloom Filter
  if (!bloom.has(userId)) {
    // Bloom Filter 說不存在 → 一定不存在，直接回 404
    // （Bloom Filter 說存在 → 可能存在，才往下查）
    return null
  }

  // Bloom Filter 放行 → 正常走 Cache-Aside
  const cached = await redis.get(`user:${userId}`)
  if (cached) return JSON.parse(cached)

  const user = await db.user.findUnique({ where: { id: userId } })
  if (user) await redis.setex(`user:${userId}`, 3600, JSON.stringify(user))
  return user
}
```

**Bloom Filter 特性**

```
誤判方向只有一種：
  說「存在」→ 可能是誤判（False Positive，約 1%）
  說「不存在」→ 一定不存在（No False Negative）

所以：
  「不存在」→ 直接擋掉，0 次 DB 查詢
  「可能存在」→ 往下查（偶爾多查一次不存在的，代價低）
```

### 架構流程

```
攻擊者查詢 /users/99999999：

方案一（空值快取）：
  第 1 次：Redis Miss → DB Miss → 存 "NULL"（TTL=60）→ 回 404
  第 2～N 次：Redis Hit（"NULL"）→ 直接回 404（DB 完全不碰）

方案二（Bloom Filter）：
  任何次：Bloom Filter Miss → 直接回 404（Redis + DB 都不碰）
```

---

## Case 2：刪除後的快取問題

### 情境

```
用戶 #1234 刪除帳號
系統刪了 DB 的資料，但忘記清 Redis

→ 用戶本人再查 → Redis HIT → 拿到舊資料（帳號已刪但還顯示）
→ 過了 TTL 後 → Redis Miss → DB 查不到 → 回 404（才對）
→ 在 TTL 內，資料不一致
```

這個是 Cache 資料一致性問題，但也會導致穿透：

```
系統清了 Redis 快取，但沒存空值
→ 刪除後有人查 → Redis Miss → DB 也沒了 → 每次都打 DB
```

### 解法：刪除時同步更新 Cache

```typescript
async function deleteUser(userId: string) {
  // 1. 刪 DB
  await db.user.delete({ where: { id: userId } })

  // 2. 設空值（不是 DEL，是設短 TTL 的 NULL）
  await redis.setex(`user:${userId}`, 300, 'NULL')
  // 5 分鐘後自動消失，之後的查詢才回「不存在」

  // 如果用 Bloom Filter，也要從裡面移除（Bloom Filter 不支援刪除）
  // → 需要用 Counting Bloom Filter 或重建 Bloom Filter
}
```

---

## 兩種解法比較

| 解法 | 適用場景 | 複雜度 | 效果 |
|------|---------|--------|------|
| 空值快取 | 通用，容易實作 | 低 | 第 2 次起擋住，但多佔 Redis 空間 |
| Bloom Filter | 大規模攻擊、ID 空間大 | 中 | 第 1 次就擋，幾乎 0 DB 壓力 |

**實際組合**

```
一般服務：空值快取就夠
  → 實作簡單，維護容易

高流量 / 安全要求高：空值快取 + Bloom Filter
  → Bloom Filter 擋非法 ID，空值快取處理邊緣情況（合法但已刪除）

新用戶註冊時要更新 Bloom Filter：
  bloom.add(newUser.id)
```

---

## 三種 Cache 問題對比

| 問題 | 觸發條件 | 特徵 | 解法 |
|------|---------|------|------|
| Cache Stampede（雪崩）| 大量 key 同時過期 | 瞬間湧入，短暫 | TTL 抖動 / Mutex Lock / Warming |
| Cache Penetration（穿透）| 查詢不存在的 key | 持續穿透，不會自癒 | 空值快取 / Bloom Filter |
| Hot Key（熱點）| 單一 key 被瘋狂讀取 | Redis 單節點頻寬打滿 | 本地快取 / Key 分片 |
