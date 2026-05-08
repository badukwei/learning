# Day 37 - 金融系統監控程式碼範例

Date: 2026-05-05

## 這份筆記在回答什麼

前一份筆記講的是：

- 交易系統整體架構
- failover
- sync / async replication
- 為什麼核心交易會偏保守

這份筆記改講：

**實際監控程式碼大概長怎樣？**

重點不是某個特定框架，而是理解金融系統常見的監控程式碼模式。

通常會包含：

1. `health check`
2. `replication lag` 監控
3. `event sequence` 監控
4. `ledger / balance` 對帳檢查
5. `不健康時自動降級或停寫`

---

## 一個總體心法

監控程式碼通常是在做三件事：

1. **收集狀態**
- DB 活著嗎
- Replica lag 多大
- 事件序號有沒有斷

2. **驗證規則**
- lag 是否超門檻
- quorum 是否足夠
- ledger 是否一致

3. **觸發保護動作**
- alert
- pause writes
- failover
- block trading

---

## 1. 基本 health check

這層只能回答：

**服務還活著嗎？**

它不能回答：

**資料是不是可信。**

```ts
type HealthStatus = "ok" | "degraded" | "down";

interface DependencyHealth {
  name: string;
  status: HealthStatus;
  detail?: string;
}

export async function healthCheck(): Promise<DependencyHealth[]> {
  const results: DependencyHealth[] = [];

  try {
    await db.query("SELECT 1");
    results.push({ name: "primary-db", status: "ok" });
  } catch (err) {
    results.push({ name: "primary-db", status: "down", detail: String(err) });
  }

  try {
    await redis.ping();
    results.push({ name: "redis", status: "ok" });
  } catch (err) {
    results.push({ name: "redis", status: "down", detail: String(err) });
  }

  return results;
}
```

### 這段程式碼代表什麼

- DB 能不能查
- Redis 能不能 ping
- 只是在做最基礎的依賴健康檢查

如果這裡失敗，通常會：

- 回報監控告警
- 將服務標記成 degraded 或 down

---

## 2. Replication Lag 監控

這層開始比較接近資料一致性問題。

會看：

- Replica 是否還在追 log
- lag 多大
- 是否仍符合核心寫入要求

```ts
interface ReplicaStatus {
  replicaName: string;
  lagMs: number;
  isHealthy: boolean;
}

const MAX_REPLICA_LAG_MS = 500;

export async function checkReplicaLag(): Promise<ReplicaStatus[]> {
  const rows = await db.query(`
    SELECT replica_name, lag_ms
    FROM replica_monitor_view
  `);

  return rows.map((row: any) => ({
    replicaName: row.replica_name,
    lagMs: row.lag_ms,
    isHealthy: row.lag_ms <= MAX_REPLICA_LAG_MS,
  }));
}
```

### 重點不是 SQL 細節

不同 DB 查法不同：

- PostgreSQL 會查 replication / WAL replay 狀態
- MySQL 會看 replica/binlog 狀態

但邏輯都差不多：

- 把「目前追到哪裡」轉成可監控指標

### 這段程式碼要回答什麼

**副本還跟得上嗎？**

不是只問「有沒有活著」。

---

## 3. 不健康時禁止核心寫入

金融系統常見的保護邏輯不是「整站關掉」，而是：

**當安全副本不足時，暫停核心寫入。**

```ts
export async function canAcceptCriticalWrites(): Promise<boolean> {
  const replicas = await checkReplicaLag();
  const healthySyncReplicas = replicas.filter(r => r.isHealthy);

  return healthySyncReplicas.length >= 1;
}

export async function placeOrder(input: PlaceOrderInput) {
  const writable = await canAcceptCriticalWrites();

  if (!writable) {
    throw new Error("Trading temporarily paused: no healthy sync replica");
  }

  return await orderService.placeOrder(input);
}
```

### 這段代表什麼

這裡的意思不是：

- 任何一台 replica 壞掉就停

而是：

- 如果連「可接受的安全副本」都不夠了
- 那就先停止核心交易寫入

這很接近前面講的：

**fail closed**

---

## 4. Event Sequence 監控

交易系統很怕事件漏掉、斷號、亂序。

例如：

- OrderAccepted 有了
- 但 TradeExecuted 不見
- 或同一個市場的成交事件順序錯亂

```ts
let lastSequence = 0;

export function onTradeEvent(event: { sequence: number; orderId: string }) {
  if (event.sequence !== lastSequence + 1) {
    alertOps("trade-sequence-gap", {
      expected: lastSequence + 1,
      actual: event.sequence,
      orderId: event.orderId,
    });

    tradingGuard.pauseMarket("sequence gap detected");
    return;
  }

  lastSequence = event.sequence;
  processTradeEvent(event);
}
```

### 這段回答的是什麼

**事件流還可信嗎？**

這跟 health check 很不一樣。

因為服務可能活著，但事件流已經壞了。

### 為什麼 sequence gap 這麼嚴重

因為如果成交事件漏一筆：

- order status 可能錯
- ledger 可能錯
- balance 可能錯
- audit trail 可能缺資料

這時很多交易系統會選：

- 暫停某個 market
- 暫停某條 pipeline

---

## 5. Ledger 對帳檢查

金融系統不是只看 API 200 OK。

更重要的是：

**帳到底對不對。**

```ts
export async function reconcileAccount(accountId: string) {
  const summary = await db.queryOne(
    `SELECT balance FROM account_balance WHERE account_id = ?`,
    [accountId]
  );

  const ledgerSum = await db.queryOne(
    `SELECT COALESCE(SUM(amount), 0) AS total FROM ledger_entries WHERE account_id = ?`,
    [accountId]
  );

  if (summary.balance !== ledgerSum.total) {
    alertOps("ledger-mismatch", {
      accountId,
      balance: summary.balance,
      ledgerTotal: ledgerSum.total,
    });

    return { ok: false };
  }

  return { ok: true };
}
```

### 這段邏輯是什麼

它在比對：

- `summary table` 的餘額
- `ledger entries` 加總後的結果

如果兩邊不同，就代表：

- 資料壞了
- 某個事件漏了
- 某次更新沒成功
- 或 failover 後狀態不一致

### 這層的重要性

有些錯誤不是服務掛了，而是：

**帳對不起來。**

這也是金融系統和一般系統很大的差別。

---

## 6. 寫後讀強制走 Primary

這個不是金融專用，但在一致性敏感場景很常見。

```ts
export async function getUserProfile(userId: string, opts?: { forcePrimary?: boolean }) {
  const dbClient = opts?.forcePrimary ? primaryDb : replicaDb;
  return dbClient.queryOne(`SELECT * FROM users WHERE id = ?`, [userId]);
}

export async function updateDisplayName(userId: string, name: string) {
  await primaryDb.query(`UPDATE users SET display_name = ? WHERE id = ?`, [name, userId]);

  return getUserProfile(userId, { forcePrimary: true });
}
```

### 這段解的是什麼

避免：

- 剛更新完
- 下一秒讀到 Replica
- 結果看到舊資料

這就是：

- `read-after-write consistency`
- `read-your-own-writes`

---

## 7. 監控器定期執行

實際系統通常會有一個 loop 或 background job 持續跑。

```ts
export async function monitoringLoop() {
  const health = await healthCheck();
  const replicas = await checkReplicaLag();

  const hasDownDependency = health.some(x => x.status === "down");
  const noHealthyReplica = replicas.every(x => !x.isHealthy);

  if (hasDownDependency || noHealthyReplica) {
    tradingGuard.pauseCriticalWrites("dependency failure or replica lag");
  } else {
    tradingGuard.resumeCriticalWrites();
  }
}
```

### 這段在做什麼

它把：

- dependency health
- replication status

變成實際決策：

- 允許交易
- 暫停核心寫入

這就是監控真正有價值的地方：

**不是只收 metrics，而是讓系統能自我保護。**

---

## 8. 真實金融系統還會多看什麼

上面只是簡化版。

真實系統通常還會監控：

- queue backlog
- consumer lag
- order accepted vs trade executed 數量差
- ledger posting delay
- failed settlement jobs
- unmatched execution reports
- reconciliation mismatch rate
- heartbeat / quorum / leader election 狀態

### 簡化理解

#### 一般系統常看

- CPU
- memory
- error rate
- latency
- health check

#### 金融系統還要看

- 事件順序
- journal 是否連續
- ledger 是否平衡
- 對帳是否成功
- 核心交易路徑是否仍可信

---

## 9. 為什麼這些監控很重要

因為很多最危險的錯誤不是：

- process crash
- 服務 500

而是：

- 事件少一筆
- failover 後少同步一段
- 帳務和 summary 對不起來
- order book 和交易回報不同步

所以金融系統監控的核心不是只有：

**服務還活著嗎？**

而是：

**資料還可信嗎？**

---

## 面試版回答

金融系統的狀態監控通常不只看 heartbeat 或 process health，而是分成幾層：第一層是基礎 health check，例如 DB 和 Redis 是否可用；第二層是 replication 與 quorum 狀態，例如 replica lag、WAL replay、同步副本是否足夠；第三層是交易事件流監控，例如 sequence gap、queue backlog、event consumer lag；第四層是業務一致性檢查，例如 ledger 與 balance summary 是否一致。程式碼上通常會把這些檢查轉成 guard logic，例如當安全副本不足時暫停核心寫入、當事件序號斷裂時暫停某個市場、當 ledger mismatch 時觸發對帳與保護模式。

---

## 一句話總結

金融系統監控程式碼的本質是：

**收集狀態 -> 驗證規則 -> 觸發保護動作**
