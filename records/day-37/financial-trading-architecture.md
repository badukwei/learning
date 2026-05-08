# Day 37 - 金融 / 證券交易系統架構流程

Date: 2026-05-05

## 這份筆記要回答什麼

你前面已經理解：

- `Primary / Replica`
- `async replication`
- `sync replication`
- `failover`

但金融 / 證券系統不只是資料庫問題。

它真正的問題是：

1. 下單後怎麼確認有沒有成功進系統
2. 成交結果怎麼保證不亂掉
3. 帳務 / 持倉 / 餘額怎麼保證正確
4. failover 時寧可停機還是寧可丟資料

所以這份筆記用「交易生命週期」來看整體流程。

---

## 先講最核心的原則

### 一般產品

比較常接受：

- 少量資料短暫不一致
- 少數最後幾筆在災難時可能遺失

換到的是：

- 更高吞吐
- 更低延遲
- 較簡單的架構

### 金融 / 證券 / 交易系統

通常不接受：

- 已確認成功的交易事後消失
- 餘額 / 持倉不一致
- 事件順序錯亂

所以它們更常做的事是：

**寧可暫停交易，也不要讓已確認交易遺失。**

這種思路叫：

**fail closed**

---

## 一個簡化但實用的整體架構圖

```text
Client / Trader
      |
      v
API Gateway / Order Entry
      |
      v
Pre-Trade Risk Checks
      |
      v
Order Journal / Event Log
      |
      v
Matching Engine
      |
      v
Trade Event Stream
      |
      +---------------------> Positions / Balances / Ledger
      |
      +---------------------> Order Status / Execution Reports
      |
      +---------------------> Risk Recalculation
      |
      +---------------------> Audit / Compliance / Reconciliation
```

這裡最關鍵的不是某一個 DB，而是：

**交易事件的順序、持久化、可重播、可稽核。**

---

## 拆成三段看：Pre-Trade / At-Trade / Post-Trade

### 1. Pre-Trade（交易前）

目標：

- 檢查這個人有沒有權限下單
- 檢查風險限制
- 檢查資金、持倉、額度、商品限制

### 2. At-Trade（交易中）

目標：

- 把有效訂單放進撮合邏輯
- 確保撮合順序正確
- 產生成交事件

### 3. Post-Trade（交易後）

目標：

- 更新訂單狀態
- 更新持倉 / 餘額 / 帳務
- 發送成交回報
- 做對帳、稽核、報表

---

## 一筆訂單的實際流程

假設使用者送出一筆買單：

> 買 100 股 AAPL，價格 180

### Step 1. Client 發送下單請求

```text
Client -> Order Entry API
```

這時候系統第一件事通常不是立刻寫最終交易表，而是先做：

- 認證 / 授權
- 參數驗證
- request id / idempotency key

為什麼要 request id？

因為網路重送時，不能讓同一筆訂單被處理兩次。

---

## Step 2. Pre-Trade Risk Check

這裡會檢查：

- 帳戶是否可交易
- 商品是否可交易
- 單筆下單量是否超限
- 當日風險額度是否超限
- 保證金 / 可用資金是否足夠

```text
Order Entry
   |
   +--> Account permission check
   +--> Product permission check
   +--> Buying power / margin check
   +--> Max order size check
   +--> Fat-finger / price band check
```

如果沒通過：

- 直接 reject
- 不會進到撮合核心

如果通過：

- 才能進下一步

---

## Step 3. 寫入 Order Journal / Event Log

這是關鍵。

金融系統常常不是直接「改資料列」當成第一步，而是先把這筆訂單當成一個不可變事件記下來。

例如：

```text
Event:
  type = OrderAccepted
  order_id = O123
  user_id = U456
  symbol = AAPL
  side = BUY
  qty = 100
  price = 180
  ts = 2026-05-05T10:01:01.123Z
```

為什麼這麼做？

因為這樣你會有：

- 可重播
- 可稽核
- 可對帳
- 可恢復

這跟一般 CRUD 很不一樣。

一般產品常想的是「現在狀態是什麼」。

交易系統常更重視：

**到底發生過什麼事，而且順序是什麼。**

---

## Step 4. Matching Engine（撮合）

撮合引擎負責：

- 決定這張單有沒有對手盤
- 按規則決定成交順序
- 產生成交事件

簡化流程：

```text
OrderAccepted Event
      |
      v
Matching Engine
      |
      +--> 沒成交：掛單進 order book
      |
      +--> 有成交：產生 TradeExecuted Event
```

### 為什麼撮合引擎常做得很保守

因為撮合不能亂序。

對同一個商品來說：

- 誰先進來
- 誰先成交
- 價格優先 / 時間優先

都必須非常明確。

很多交易系統會讓同一市場 / 同一商品的撮合邏輯盡量單執行緒或單 leader 順序化處理，目的就是：

**避免同時多點寫入導致撮合結果不一致。**

---

## Step 5. 成交事件往外傳

撮合結果一出來，不是只改一張表就結束。

會有很多下游系統需要知道：

- 訂單狀態服務
- 持倉服務
- 帳務 / Ledger
- 風控服務
- 客戶通知 / execution report
- 合規 / audit / 報表

```text
TradeExecuted Event
      |
      +--> Order Status Service
      +--> Position Service
      +--> Balance / Ledger Service
      +--> Risk Engine
      +--> Client Notification
      +--> Audit / Compliance
```

這也是為什麼交易系統很常用：

- event log
- event stream
- append-only journal

因為一個核心事件會驅動很多下游。

---

## Step 6. Ledger / Balance / Position 更新

這一段比「顯示名稱更新」嚴格很多。

因為這裡牽涉：

- 餘額
- 持倉
- 已成交數量
- 凍結資金 / 解凍資金

常見做法是：

- 核心帳務路徑強一致
- 寫入前先記交易事件
- Ledger 用 append-only entries

例如：

```text
TradeExecuted
  -> create ledger entry: reserve cash
  -> create ledger entry: decrease cash
  -> create ledger entry: increase position
```

為什麼金融系統喜歡 ledger / append-only？

因為不要直接覆蓋成：

```text
balance = 900
```

而是保留：

```text
-100 because trade T123 executed
```

這樣之後：

- 對帳比較容易
- 稽核比較容易
- 補償比較容易

---

## Failover 時為什麼會那麼敏感

因為交易系統最怕這種情況：

```text
Client 收到：下單成功 / 成交成功
但 failover 後系統裡找不到這筆
```

這比一般產品嚴重太多。

例如：

- 用戶以為自己買到了
- 系統恢復後卻沒有這筆單
- 客戶、券商、清算端看到的結果不同

這是不能接受的。

---

## 非同步複製下，問題怎麼發生

### 流程圖

```text
T0  Client -> Order Entry : place order

T1  Primary 寫入成功

T2  Primary 對外回 success

T3  Primary 準備同步到 Replica

T4  Primary 掛掉 ❌

T5  Failover 到 Replica

結果：
Replica 還沒有那筆訂單
=> 客戶看到成功，但新主節點找不到單
```

這就是：

**已確認事件遺失**

---

## 所以金融系統怎麼避免

### 做法 1：核心路徑偏向同步保護

對交易核心事件，系統常見要求：

- 至少一個同步副本也確認成功
- 或 quorum 已確認
- 才對 client 回成功

流程：

```text
T0  Client -> Primary : place order

T1  Primary 寫本機 journal / WAL

T2  Primary 把 journal / WAL 傳給同步副本

T3  同步副本落盤並 ACK

T4  Primary 才回成功給 Client
```

這樣如果 `T4` 之後 Primary 掛掉：

- 同步副本仍有這筆事件
- failover 後能保住結果

### 做法 2：不確定一致時，先停止交易

如果 failover 發生，但系統無法保證：

- 新主節點完整
- journal 沒缺
- ledger 沒斷

那很多交易系統寧可：

- 暫停接新單
- 暫停成交
- 暫停核心寫入

等人工或自動機制確認後再恢復

這就是：

**fail closed**

### 做法 3：使用 Journal + Replay + Reconciliation

即使做了很多保護，恢復後通常還要：

- replay event log
- 對帳
- 比對 broker / OMS / clearing records
- 補發或標記異常事件

這代表：

**不是只靠主從切換就結束，還要能證明資料一致。**

---

## 一個更完整的 failover 流程

### 正常運作

```text
Client
  |
  v
Order Entry
  |
  v
Primary Journal / DB
  |
  +--> Sync Replica
  |
  +--> Async Analytics / Reports
```

### Primary 掛掉

```text
Step 1. Health check / heartbeat 偵測 Primary 掛掉

Step 2. 自動化系統判斷：
        哪個副本是最新、可升格的？

Step 3. 只有「已同步到安全點」的副本，才允許升格

Step 4. Promote Replica -> New Primary

Step 5. 交易入口切到新 Primary

Step 6. 恢復接單

Step 7. 事後做 reconciliation / audit check
```

### 圖解

```text
原本：

Client -> Primary
            |
            +--> Sync Replica A
            +--> Async Replica B

掛掉後：

Client -> Replica A (promoted)
            |
            +--> Replica B
```

注意：

不是所有 Replica 都能安全升主。

如果某台只是 async，且明顯落後，很多系統不會直接讓它自動接主。

---

## 交易系統常見的分層取捨

### 強一致核心

這些通常要很保守：

- 下單接受
- 成交事件
- 帳務
- 餘額
- 持倉

### 可最終一致外圍

這些可以比較放鬆：

- 報表
- 歷史查詢
- 分析
- 儀表板
- 通知副本

所以交易系統不是所有東西都同步到死，而是：

**核心強一致，外圍最終一致。**

---

## 你可以這樣理解證券系統的哲學

### 一般 App

```text
重點：
服務先活著、快、好擴展
```

### 交易系統

```text
重點：
已確認交易不能亂掉
帳務不能錯
必要時先停，不硬撐
```

---

## 面試版回答

金融或證券交易系統通常不只是在做 Primary/Replica 複製，而是把整個交易流程拆成 pre-trade risk、matching、post-trade ledger 與 audit。核心交易事件通常會先寫入 journal 或 event log，再進入撮合與帳務流程。為了避免 failover 時發生「客戶已收到成功，但新主節點沒有這筆交易」的情況，核心交易路徑通常會偏向同步保護或 quorum commit，也就是至少一個安全副本確認後才回成功；如果 failover 發生但系統無法保證一致性，很多交易系統會選擇 fail closed，暫停接單或寫入，等確認新主節點狀態無誤後再恢復。外圍的報表、分析、歷史查詢則可以接受最終一致。

---

## 一句話總結

金融系統最重要的不是「永遠不停」，而是：

**已確認的交易事件必須可追、可證明、可恢復，而且不能在 failover 後消失。**
