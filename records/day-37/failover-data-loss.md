# Day 37 - Failover 與資料遺失

Date: 2026-05-05

## 這份筆記要解決什麼問題

問題是：

> 如果 `Primary DB` 已經收到寫入，但在同步到 `Replica` 的過程中掛掉，資料會不會遺失？

這個問題要分三種情況看：

1. 一般系統的 `非同步複製（async replication）`
2. 一般系統的 `同步複製 / quorum commit`
3. 證券 / 交易系統這種高一致性場景

---

## 先記最核心的一句話

### 非同步複製

`Primary` 可以先回成功，`Replica` 晚一點再追。

所以：

**Primary 掛掉時，最後幾筆資料可能還沒到 Replica，會有遺失風險。**

### 同步複製

`Primary` 要等至少一個同步副本確認，才回成功。

所以：

**只要對外回成功，通常代表至少不只一台機器有這筆資料。**

### 證券 / 交易系統

通常不接受「已經回成功的交易結果還會丟掉」。

所以它們常見做法是：

**寧可暫停交易，也不要讓已確認交易遺失。**

---

## 情境一：非同步複製（最常見）

### 架構

```text
App
  |
  v
Primary DB  ----async replicate---->  Replica DB
```

### 寫入成功的流程

```text
Step 1. User 發 request
        「我要扣款 100」

Step 2. App 寫到 Primary

Step 3. Primary 寫入成功並 commit

Step 4. Primary 對 App 回 success

Step 5. Primary 之後才把 WAL / binlog 傳給 Replica

Step 6. Replica 晚一點 replay
```

### 問題發生的時機

如果在這個時候：

```text
Primary 已經回 success
但 Replica 還沒追到最新 log
Primary 就掛掉了
```

會變成：

```text
User 看到：成功
Replica 裡：沒有這筆資料
Primary：已經死掉
```

### 圖解

```text
T0  User -> App -> Primary : UPDATE balance = balance - 100

T1  Primary : commit 成功

T2  Primary -> App : success

T3  Primary 原本要把 WAL 傳給 Replica

T4  Primary 掛掉 ❌

T5  系統 failover 到 Replica

結果：
Replica 沒有這筆扣款資料
=> 這筆資料可能遺失
```

### 為什麼會這樣

因為在 `async replication` 裡：

- `Primary` 不需要等 `Replica`
- 只要自己 commit 成功，就可以先回應 client

所以你拿到的成功，只代表：

**Primary 本機成功**

不代表：

**副本也已經同步完成**

### 優點

- 快
- 吞吐量高
- 寫入延遲低

### 缺點

- failover 時最後幾筆資料可能丟

---

## 情境二：同步複製 / Quorum Commit

### 架構

```text
App
  |
  v
Primary DB  ----sync replicate---->  Replica A
            ----async replicate--->  Replica B
```

常見做法不是等所有 Replica，而是：

- 至少等 `1` 台同步副本確認
- 或等一個 `quorum`

### 寫入成功的流程

```text
Step 1. User 發 request

Step 2. App 寫到 Primary

Step 3. Primary 先寫本機 WAL

Step 4. Primary 把 WAL 傳給同步副本

Step 5. 同步副本確認「我也寫好了」

Step 6. Primary 才對 App 回 success
```

### 圖解

```text
T0  User -> App -> Primary : UPDATE order status = paid

T1  Primary : WAL 落盤

T2  Primary -> Replica A : 傳送 WAL

T3  Replica A : WAL 落盤，回 ACK

T4  Primary -> App : success
```

### 這樣有什麼好處

如果 `T4` 之後 Primary 掛掉：

```text
Primary 掛掉 ❌
Replica A 還活著 ✅
```

因為 `Replica A` 已經確認過這筆資料，所以 failover 後通常還能保住這筆寫入。

也就是說：

**對外回 success 前，至少兩個地方已經有資料。**

### 缺點

- 寫入變慢
- 任何同步副本變慢，都可能拖慢 Primary
- 如果同步副本掛掉，寫入可能卡住或失敗

### Quorum Commit 是什麼

不是等所有副本都確認，而是等「足夠多」的副本確認。

例如：

```text
Primary + 3 replicas

規則：
至少 2 個節點確認，才算成功
```

這樣可以比「全同步」更平衡：

- 一致性較高
- 可用性比全同步好一點

---

## 情境三：Failover 真正發生時

### 沒有自動 failover

```text
Primary 掛掉
=> 沒人能寫入
=> 讀可能還能靠 Replica
=> 要人工把某台 Replica 升格成新 Primary
```

### 有自動 failover

通常流程是：

```text
Step 1. 系統偵測 Primary 沒心跳

Step 2. 其他節點判斷這是不是短暫網路問題

Step 3. 選出最適合升格的 Replica
        通常選資料最新的那台

Step 4. 把這台 Replica promote 成新 Primary

Step 5. App / Proxy / 連線端切到新 Primary

Step 6. 其他 Replica 開始跟新 Primary 複製
```

### 圖解

```text
原本：

          write
App --------------> Primary
                      |
                      +--> Replica A
                      |
                      +--> Replica B

Primary 掛掉後：

          write
App --------------> Replica A (promoted as new Primary)
                      |
                      +--> Replica B
```

---

## 問題核心：資料遺失是發生在哪一刻

最關鍵是這一段：

```text
Primary 已回 success
但同步還沒完成
Primary 就掛掉
```

如果這段存在，就有可能資料遺失。

所以真正差別不是：

- 有沒有 Replica

而是：

- `回 success` 的時候，是否已有足夠多副本確認

---

## 證券 / 交易系統通常怎麼做

這裡先講原則，不假設每家交易所完全一樣。

### 它們最在意的不是「永遠不停機」

而是：

1. 已確認交易不能亂丟
2. 帳務 / 持倉 / 餘額不能錯
3. 事件順序必須可靠
4. 要能完整稽核與對帳

所以它們常見會做：

### 做法 1：核心交易路徑偏向同步保護

核心交易資料常用：

- synchronous replication
- quorum commit
- sync standby

目的：

**對外回成功前，至少不只一台機器有資料**

### 做法 2：Fail closed，不隨便繼續交易

如果系統無法確定資料完全一致，常見策略不是硬撐，而是：

- 暫停下單
- 暫停寫入
- 進入保護模式
- 等確認新主節點狀態後再恢復

也就是：

**寧可短暫不可用，也不要讓帳務錯亂。**

### 做法 3：事件日誌 / 不可變更交易紀錄

交易系統通常不只靠最終資料表，還會有：

- append-only event log
- audit trail
- message journal

因為事後一定要能回答：

- 這筆單到底有沒有收進來？
- 有沒有成交？
- 在哪個時間點 commit？
- failover 前後各節點看到什麼？

### 做法 4：對帳與補償流程

即使做很多保護，系統恢復後仍常需要：

- reconciliation 對帳
- 補單 / 補事件
- 人工確認異常 case

### 做法 5：主交易與報表查詢分流

交易核心資料可能要求高一致性，但：

- 報表
- 歷史查詢
- 統計頁

這些可以晚一點同步，不一定要卡在同步主路徑上。

---

## 用最白話的方式理解證券系統

### 一般產品

```text
可以接受：
少數最後幾筆資料在災難時遺失

換到的是：
更高效能、更低延遲
```

### 證券 / 金流 / 交易系統

```text
不能接受：
已回成功的交易事後消失

所以會選：
較高一致性
較保守 failover
必要時暫停服務
```

---

## 你要記住的判斷框架

問自己三個問題：

### 1. 成功回應代表什麼

是代表：

- 只有 Primary 成功？

還是：

- 至少一個同步副本也成功？

### 2. 這筆資料能不能丟

- like count 可以比較寬鬆
- balance / order / trade 不行

### 3. 系統比較重視什麼

- 低延遲、高吞吐
- 還是強一致、不丟資料

---

## 一句話總結

### 非同步複製

`success` 可能只代表 Primary 成功，所以 failover 時可能丟最後幾筆資料。

### 同步複製

`success` 代表至少有同步副本確認，所以資料遺失風險低很多，但延遲更高。

### 證券 / 交易系統

通常不接受已確認交易遺失，因此核心路徑更偏向同步保護與保守 failover。
