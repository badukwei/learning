# Day 37 - 金融系統總整理

Date: 2026-05-05

## 這份筆記整理什麼

這份是以下三份筆記的總整理：

- `financial-trading-architecture.md`
- `failover-data-loss.md`
- `financial-monitoring-code.md`

目的是把它們串成一條完整脈絡，讓之後複習時先看這份，再往細節展開。

---

## 一句話先講完

金融 / 證券交易系統的核心不是「永遠不停機」，而是：

**已確認的交易事件不能消失，帳務不能錯，事件順序不能亂。**

所以它們常常會選擇：

- 核心路徑高一致性
- 外圍查詢最終一致
- 監控資料可信度
- 必要時降級或暫停核心交易

---

## 一筆交易的完整路徑

可以先把交易生命週期記成：

```text
Client
  -> Order Entry
  -> Pre-Trade Risk Check
  -> Order Journal / Event Log
  -> Matching Engine
  -> Trade Event
  -> Ledger / Balance / Position
  -> Audit / Reconciliation
```

### 各段在做什麼

#### 1. Order Entry

- 收訂單 request
- 做認證、授權、基本驗證
- 處理 request id / idempotency

#### 2. Pre-Trade Risk Check

- 檢查額度
- 檢查權限
- 檢查保證金 / buying power
- 檢查是否超量、超價

#### 3. Order Journal / Event Log

- 先把訂單事件安全記下來
- 讓後續能 replay、audit、reconcile

#### 4. Matching Engine

- 根據撮合規則處理訂單
- 可能掛單，也可能立即成交
- 產生成交事件

#### 5. Ledger / Balance / Position

- 更新帳務
- 更新持倉
- 更新可用資金
- 保留 append-only 的交易明細

#### 6. Audit / Reconciliation

- 檢查系統內外資料是否一致
- 讓事後追查和補償有依據

---

## 為什麼交易系統很重視 Event Log / Journal

一般產品常常只關心：

- 現在狀態是什麼

交易系統更關心：

- 到底發生過什麼事件
- 發生順序是什麼
- 這些事件能不能重播與對帳

所以核心事件常先寫成：

- journal
- append-only log
- immutable event

好處是：

- 可追蹤
- 可稽核
- 可恢復
- 可對帳

---

## 為什麼 failover 會有資料遺失風險

問題點通常不是有沒有 `Replica`，而是：

**系統什麼時候對外回 success。**

### 非同步複製（async replication）

流程：

```text
1. Primary 寫成功
2. Primary 先回 success
3. Replica 晚一點追 log
```

風險：

```text
Primary 已回 success
但 Replica 還沒追上
Primary 就掛掉
=> failover 後最後幾筆資料可能消失
```

### 同步複製 / Quorum Commit

流程：

```text
1. Primary 寫本機 log
2. 同步副本也寫好
3. 收到 ACK
4. 才回 success
```

好處：

```text
只要回 success
通常代表不只一台機器有這筆資料
=> failover 後資料遺失風險低很多
```

代價：

- 延遲更高
- 寫入吞吐較差
- 對同步副本健康度更敏感

---

## 為什麼金融系統常常「寧可停，不硬撐」

因為它們不能接受：

- 客戶看到成功，但系統找不到交易
- 餘額錯
- 持倉錯
- 成交順序錯

所以在以下情況，很多系統會進入保守模式：

- 安全副本不足
- 無法確認新主節點資料完整
- event sequence 斷裂
- ledger 對不起來
- failover 後狀態不可信

這種思路就是：

**fail closed**

也就是：

- 暫停新下單
- 暫停核心寫入
- 等確認狀態一致後再恢復

---

## 不是所有功能都要同樣嚴格

這是很重要的取捨。

### 核心高一致性路徑

通常要更保守：

- 下單接受
- 成交事件
- 帳務
- 餘額
- 持倉

### 外圍可最終一致路徑

通常可以放鬆：

- 報表
- 分析
- 歷史查詢
- 統計頁
- 非核心通知

所以交易系統不是所有東西都同步到底，而是：

**核心強一致，外圍最終一致。**

---

## 監控不是只看 heartbeat

一般系統常常只看：

- process 活著嗎
- DB 連得到嗎
- API 200 OK 嗎

但金融系統還要多看三件事：

### 1. Replication 狀態

- replica lag
- WAL / binlog replay 進度
- sync replica 是否仍健康
- quorum 是否足夠

### 2. Event Flow 狀態

- sequence number 有沒有斷
- queue / topic 有沒有 backlog
- event consumer 是否落後

### 3. 業務一致性

- ledger 和 balance summary 對不對得起來
- order journal 和 order status 是否一致
- trade event 是否完整

所以監控真正要回答的是：

**資料還可信嗎？**

不只是：

**服務還活著嗎？**

---

## 監控程式碼的本質

可以總結成三步：

### 1. 收集狀態

- health check
- replica lag
- event sequence
- ledger consistency

### 2. 驗證規則

- lag 是否超門檻
- quorum 是否足夠
- 事件是否斷號
- 帳務是否 mismatch

### 3. 觸發保護動作

- alert
- pause critical writes
- pause market
- failover
- 進入保護模式

一句話：

**收集狀態 -> 驗證規則 -> 觸發保護動作**

---

## 你現在最該記住的幾句話

### 1. 交易系統重視的是事件，不只是最終狀態

- 先記事件
- 再驅動撮合、帳務、通知、稽核

### 2. failover 風險來自「success 回得太早」

- async：可能只代表 Primary 成功
- sync/quorum：通常代表至少有安全副本確認

### 3. 核心資料和外圍資料的一致性要求不同

- balance / position / trade：高一致性
- reports / analytics / history：可最終一致

### 4. 金融系統監控的是資料可信度

不是只有 process 活著沒。

---

## 面試版總回答

金融或證券交易系統通常不是單純做 Primary/Replica 複製，而是以交易事件為核心來設計整個流程：訂單先經過 pre-trade risk 檢查，再寫入 journal 或 event log，之後進入 matching engine，產生成交事件，最後驅動 ledger、balance、position 與 audit/reconciliation。這種系統最怕的是 failover 時已確認交易遺失，所以核心寫入路徑通常會偏向同步保護或 quorum commit，也就是至少一個安全副本確認後才回成功；如果系統無法保證資料一致，常見做法是 fail closed，暫停核心交易而不是硬撐。監控上也不只看 heartbeat，而會同時看 replication lag、event sequence、ledger consistency，因為金融系統真正要確保的是資料可信、事件有序、帳務正確。

---

## 對應細節筆記

如果之後要展開看：

- 架構流程：`financial-trading-architecture.md`
- failover / data loss：`failover-data-loss.md`
- 監控程式碼：`financial-monitoring-code.md`
