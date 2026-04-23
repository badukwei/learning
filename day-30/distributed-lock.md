# 分散式鎖（Distributed Lock）

> 問題：多台 Server 同時處理同一筆資料 → Race Condition → 資料錯誤
> 解法：SET NX（原子操作），搶到鎖才能執行，其他人等待

---

## 什麼是 Race Condition

```
沒有鎖的情況：

Server A 讀庫存：剩 1 件
Server B 讀庫存：剩 1 件
Server A 扣庫存：1 - 1 = 0，寫回 DB
Server B 扣庫存：1 - 1 = 0，寫回 DB

結果：賣出去 2 件，但庫存只有 1 件 → 超賣
```

---

## Case 1：電商庫存扣減

### 情境

```
秒殺活動，最後 1 件 iPhone
1000 個用戶同時下單
→ 多台 Server 同時讀到庫存 = 1
→ 同時扣減 → 賣出幾百件
→ 實際庫存 = 0 → 出貨爆炸
```

### 解法

```typescript
async function purchaseItem(itemId: string, userId: string) {
  const lockKey = `lock:item:${itemId}`
  const lockValue = `${userId}:${Date.now()}`  // 唯一值，防止誤刪別人的鎖

  // 搶鎖（原子操作）
  const acquired = await redis.set(lockKey, lockValue, 'NX', 'EX', 30)
  // NX = 不存在才設（只有一個人能成功）
  // EX 30 = 30 秒後自動釋放（防死鎖）

  if (acquired !== 'OK') {
    throw new Error('商品處理中，請稍後再試')
  }

  try {
    // 查庫存
    const item = await db.item.findUnique({ where: { id: itemId } })
    if (!item || item.stock <= 0) {
      throw new Error('庫存不足')
    }

    // 扣庫存 + 建訂單（同一個 transaction）
    await db.$transaction([
      db.item.update({
        where: { id: itemId },
        data: { stock: { decrement: 1 } },
      }),
      db.order.create({
        data: { itemId, userId, status: 'pending' },
      }),
    ])

    return { success: true }
  } finally {
    // 只能刪自己的鎖
    const current = await redis.get(lockKey)
    if (current === lockValue) {
      await redis.del(lockKey)
    }
  }
}
```

### 架構流程

```
1000 個請求同時進來：

Request #1（最快）：
  → SET lock:item:iphone NX → OK（搶到）
  → 查庫存 = 1 → 扣庫存 → 建訂單
  → DEL lock:item:iphone

Request #2 ~ #999：
  → SET lock:item:iphone NX → nil（已有鎖）
  → 回傳「商品處理中」

Request #1000（#1 完成後）：
  → SET lock:item:iphone NX → OK（搶到）
  → 查庫存 = 0 → 回傳「庫存不足」

DB：只有 1 筆扣減，不會超賣
```

---

## Case 2：重複付款防止

### 情境

```
用戶點付款按鈕，網路慢，以為沒反應
→ 又點了一次
→ 兩個付款請求同時送出
→ 扣了兩次錢
```

### 解法：冪等鎖（Idempotency Lock）

```typescript
async function processPayment(orderId: string, userId: string) {
  const lockKey = `lock:payment:${orderId}`

  const acquired = await redis.set(lockKey, userId, 'NX', 'EX', 60)
  if (acquired !== 'OK') {
    // 已經有人在處理這筆訂單
    return { status: 'processing' }
  }

  try {
    // 檢查訂單是否已付款
    const order = await db.order.findUnique({ where: { id: orderId } })
    if (order?.status === 'paid') {
      return { status: 'already_paid' }
    }

    // 呼叫金流
    await paymentGateway.charge(order!)

    // 更新訂單狀態
    await db.order.update({
      where: { id: orderId },
      data: { status: 'paid' },
    })

    return { status: 'success' }
  } finally {
    await redis.del(lockKey)
  }
}
```

### 架構流程

```
用戶快速點兩次付款：

Request #1：
  → SET lock:payment:order456 NX → OK
  → 呼叫金流 → 扣款成功 → 更新 DB → DEL lock

Request #2（幾乎同時）：
  → SET lock:payment:order456 NX → nil
  → 回傳 { status: 'processing' }
  → 前端顯示「付款處理中」

結果：只扣一次錢
```

---

## Case 3：定時任務防重複執行

### 情境

```
每天凌晨 2:00 跑一次對帳任務
部署了 3 台 Server，每台都有 cron job
→ 三台同時觸發 → 對帳跑了 3 次 → 資料重複
```

### 解法

```typescript
async function runReconciliation() {
  const lockKey = 'lock:cron:reconciliation'
  const lockValue = process.env.SERVER_ID || 'server-1'  // 哪台 Server 搶到

  const acquired = await redis.set(lockKey, lockValue, 'NX', 'EX', 3600)
  // EX 3600 = 1 小時，足夠讓任務跑完

  if (acquired !== 'OK') {
    console.log('其他 Server 已在執行對帳，跳過')
    return
  }

  try {
    console.log(`${lockValue} 開始執行對帳`)
    await doReconciliation()
    console.log('對帳完成')
  } finally {
    await redis.del(lockKey)
  }
}

// 三台 Server 都在凌晨 2:00 跑這個
cron.schedule('0 2 * * *', runReconciliation)
```

### 架構流程

```
凌晨 2:00，三台 Server 同時觸發：

Server A：SET lock:cron:reconciliation NX → OK → 執行對帳
Server B：SET lock:cron:reconciliation NX → nil → 跳過
Server C：SET lock:cron:reconciliation NX → nil → 跳過

只有 Server A 跑，完成後 DEL lock
```

---

## 鎖的安全細節

**為什麼 lockValue 要用唯一值**

```
壞的做法：
  Server A 設 lock，value = "1"
  Server A 任務跑太久（超過 EX 時間）→ 鎖自動過期
  Server B 搶到鎖，value = "1"
  Server A 任務終於跑完 → DEL lock
  → Server A 刪掉了 Server B 的鎖！

好的做法：
  Server A：value = "serverA:1234567890"
  Server B：value = "serverB:9876543210"
  刪之前先 GET 確認 value 是自己的，才 DEL
```

**EX 時間設多少**

```
設太短 → 任務還沒跑完鎖就過期 → 其他人搶到 → 重複執行
設太長 → Server 當機後鎖卡住很久 → 其他人等很久

原則：預估任務最長時間 × 2，再加一點餘裕
```

---

## 三個 Case 比較

| Case | 保護對象 | Lock 粒度 | EX 時間 |
|------|---------|----------|--------|
| 庫存扣減 | 單一商品 | item:id | 短（30s）|
| 重複付款 | 單一訂單 | order:id | 中（60s）|
| 定時任務 | 整個任務 | cron:name | 長（任務時間）|
