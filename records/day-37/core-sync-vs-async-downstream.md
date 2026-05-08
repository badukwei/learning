# Day 37 - 核心同步與下游延遲同步

Date: 2026-05-05

## 這份筆記要回答什麼

這份整理的是你剛剛一路追問的核心問題：

1. 交易所 / 金融系統既然要求高一致性，為什麼還能低延遲？
2. 安全副本同步到底同步了什麼？
3. 安全副本同步完，後面不是還有很多資料庫要同步嗎？
4. 這些延遲同步的資料，最後不會累積成卡頓或影響用戶體驗嗎？

---

## 先講最核心的一句話

交易系統不是把所有事情都同步做完才回成功。

它做的是：

**只把「核心交易事件」放進很短的同步保護路徑，其他衍生資料放到下游 async 更新。**

所以要分成兩段看：

1. `回 success 前的核心同步`
2. `回 success 後的下游同步`

---

## 一、回 success 前的核心同步

### 最短同步路徑

```text
下單
-> 風控檢查
-> 撮合 / 排序
-> 更新記憶體核心狀態
-> append journal / event log
-> 至少一個安全副本 ACK
-> 回 success
```

### 這條路徑保證什麼

它保證的是：

- 這筆 order / trade 已被接受
- 順序已確定
- 核心事件已持久化
- 至少一個安全副本也知道這件事

它**不保證**：

- 報表已更新
- 歷史查詢已更新
- cache 已刷新
- analytics 已同步

### 為什麼這樣還能快

因為這條同步路徑被壓得很短：

- 核心狀態很多在 RAM
- 同步的是小型 append-only event / log
- 同步副本距離很近
- 不等一堆周邊系統

所以它慢的只是：

**多等一個很短的安全副本確認**

不是：

**把所有資料庫與所有服務都等完**

---

## 二、安全副本同步到底同步什麼

### 不是整份資料庫同步

安全副本通常同步的不是：

- 全部 tables
- 全部 read model
- 全部報表 DB

而是很小的核心事件，例如：

- `OrderAccepted`
- `TradeExecuted`
- 一段 `WAL / binlog`
- 一段 append-only journal

### 可以這樣理解

```text
Primary:
  我已經接受了這筆訂單
  這是 event #89123
  內容如下

Sync Replica:
  我也安全記下來了
  ACK
```

只要這段成立，Primary 就敢回成功。

---

## 三、回 success 後的下游同步

### 這時還有哪些東西沒更新

回成功之後，通常還會有很多下游要更新：

```text
TradeExecuted event
  -> order history DB
  -> balances view DB
  -> positions view DB
  -> reporting DB
  -> analytics DB
  -> cache
  -> notification system
```

這些確實都還要同步。

但差別是：

**它們不在用戶這次請求的同步等待路徑上。**

也就是說：

- 會影響資料新鮮度
- 不一定影響這次寫入 latency

### 核心資料 vs 衍生資料

這裡一定要分清楚：

#### 核心資料（source of truth）

- order journal
- trade journal
- ledger
- 核心撮合狀態

#### 衍生資料（read models / views）

- 訂單歷史頁
- 帳戶總覽頁
- 報表
- analytics
- cache

只要核心事件沒丟，衍生資料晚一點通常都能補。

---

## 四、既然下游 async，為什麼不會立刻影響用戶體驗

### 原因 1：不是所有資料都同等重要

#### 高敏感、要很快追上的

- order status
- execution report
- available balance
- position

#### 可接受延遲的

- 報表
- BI
- analytics
- 歷史統計

也就是說：

**async 世界裡也有優先級，不是全部一視同仁。**

### 原因 2：用戶看到的是階段性狀態，不一定要立刻全完成

例如訂單狀態可以是：

- `ACCEPTED`
- `MATCHING`
- `PARTIALLY_FILLED`
- `FILLED`
- `SETTLING`

所以系統不需要假裝所有事情瞬間全部完成。

而是：

**讓使用者知道目前處於哪個可靠階段。**

### 原因 3：重要查詢不一定讀最慢的資料庫

例如：

- 剛下單後的訂單頁，可能讀比較新的 order store
- 餘額頁，可能讀比較可靠的 account service
- 報表頁，才讀較慢的 reporting DB

所以不是每個頁面都接最延遲的 read model。

---

## 五、那 async 下游會不會最後失控

### 會，如果沒有管控

如果只是把事情丟到 async 然後不管，最後真的會出問題：

- queue backlog 越積越多
- consumer lag 越來越大
- order history 不更新
- balance view 落後
- cache 一直 stale
- 用戶開始覺得系統卡住

所以正確答案不是：

**async 不會有問題**

而是：

**async 的問題要被監控、分級、補償、必要時降級。**

---

## 六、金融系統怎麼避免 async 累積失控

### 1. 監控 backlog / lag

會看：

- queue backlog
- consumer lag
- event processing latency
- dead letter queue
- 某個 read model 落後多久

如果超標，就：

- alert
- 擴容 consumer
- 重啟 pipeline
- 降級部分功能

### 2. 重要下游優先處理

不是每個 consumer 同等優先。

高優先：

- order status
- balance
- positions
- execution report

低優先：

- analytics
- BI
- reports

### 3. Replay / Idempotency

如果某個 consumer 掛掉，不是資料就永遠錯掉。

通常會有：

- sequence / offset
- replay 機制
- idempotent handler
- retry
- reconciliation

所以它可以補追。

### 4. Reconciliation / 對帳

金融系統最後一定會比：

- ledger vs balance summary
- journal vs order history
- trade events vs positions

如果對不起來，就要進保護模式或補償。

### 5. 必要時降級

如果 backlog 太大、重要 read model 太落後，系統可能會：

- 暫停某些查詢功能
- 暫停部分市場
- 限制新單
- 進入保護模式

因為繼續塞更多事件只會讓狀態更亂。

---

## 七、你可以怎麼理解整個取捨

### 核心同步解的是

- 交易不能丟
- 順序不能亂
- 成功不能回太早

### 下游 async 解的是

- 多資料庫、多 view、多報表不必卡住交易主路徑
- 讓整體系統保有低延遲與擴展性

### 監控與降級解的是

- 避免 async 無限累積
- 避免 read model 長時間失真
- 避免用戶開始感覺系統卡住或不可信

---

## 一張總圖

```text
                [核心同步路徑]

Client
  |
  v
Order Entry
  |
  v
Risk Check
  |
  v
Matching / Sequencing
  |
  v
In-Memory Core State
  |
  v
Journal / Event Log
  |
  v
Sync Replica ACK
  |
  v
Return Success


                [下游 async 路徑]

Trade Event
  |
  +--> Order Status Read Model
  +--> Balance View
  +--> Position View
  +--> Order History DB
  +--> Reporting DB
  +--> Analytics
  +--> Cache
  +--> Notifications
```

---

## 一句話總結

**安全副本同步後，後面確實還有很多資料庫要同步；但那些多半是下游衍生資料，它們不阻塞這次請求，所以主要影響的是資料新鮮度，而不是當下寫入延遲。真正避免它們失控的方法，是優先級分層、lag/backlog 監控、replay/reconciliation，以及必要時降級。**

