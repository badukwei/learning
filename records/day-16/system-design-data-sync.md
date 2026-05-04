# System Design - 多台 Server 的資料同步

Date: 2026-04-02

---

## 問題背景

水平擴展後，同一份資料可能要在多台 Server 上保持一致：

```
用戶 A 在 Server 1 更新了個人資料
用戶 B 在 Server 2 查詢同一個用戶的資料 → 要看到最新的？
```

同步策略取決於「你能接受多少延遲」和「資料一致性要多強」。

---

## 同步的對象分兩種

先分清楚：到底是誰要同步誰？

```
1. DB 同步（最根本）
   → 多台 DB 之間的資料要一致
   → Replication、Sharding

2. 應用層 Server 狀態同步
   → Session、Cache、在線人數等
   → Redis、Message Queue
```

---

## DB 層的同步

### Primary-Replica Replication（主從複製）

最常見的做法：一台 Primary 負責寫入，多台 Replica 負責讀取。

```
                  寫入
用戶 ──────────► Primary DB
                    │
                    │ 複製（Replication）
                    ├──────────────► Replica 1
                    ├──────────────► Replica 2
                    └──────────────► Replica 3

讀取
用戶 ──────────► Load Balancer ──► Replica 1, 2, 3（任一台）
```

**同步怎麼觸發的？**

不是定期同步，是**事件驅動**的。Primary 每次有寫入操作，就把這個變更記錄到 **Binary Log（binlog）**，Replica 持續監聽並重播這些 log。

```
Primary 執行：
  UPDATE users SET name='Wei-Chun' WHERE id=1

Primary 寫入 binlog：
  [timestamp] UPDATE users SET name='Wei-Chun' WHERE id=1

Replica 1 持續讀 binlog → 發現新記錄 → 重播這個 SQL → 資料同步
```

**延遲（Replication Lag）：**

Replica 跟 Primary 不是完全即時的，通常有幾毫秒到幾秒的延遲。

```
問題情境：
T+0ms  用戶改了密碼 → 寫入 Primary
T+0ms  立刻查詢 → 被導到 Replica 1
T+50ms Replica 1 還沒同步到 → 查到舊密碼 ← 問題！

解法：
  - 寫入後的查詢強制走 Primary（Read-after-write consistency）
  - 或用同步複製（犧牲效能換一致性）
```

---

### 同步複製 vs 非同步複製

```
非同步複製（預設）：
  Primary 寫入成功 → 立刻回傳 OK → 之後慢慢同步 Replica
  優點：速度快
  缺點：Primary 掛掉，還沒同步的資料會遺失

同步複製：
  Primary 寫入 → 等 Replica 也確認寫入 → 才回傳 OK
  優點：資料不會遺失
  缺點：每次寫入都要等 Replica，速度慢
  用途：金融、支付等不允許資料遺失的場景
```

---

### Multi-Primary（多主）— 每台都可以寫

```
Primary A ◄──────────────► Primary B
    │    雙向同步             │
    ▼                         ▼
Replica A1, A2           Replica B1, B2
```

**衝突問題：**

```
T+0ms  Primary A：用戶餘額 = 1000，扣 500 → 寫入 500
T+0ms  Primary B：同一個用戶，扣 800 → 寫入 200

兩台同時寫入，最終值要是多少？← 衝突！
```

解法（衝突解決策略）：
- **Last Write Wins**：時間戳較新的贏（可能丟資料）
- **Application-level merge**：應用層自己處理合併邏輯
- **CRDT**（Conflict-free Replicated Data Type）：特殊資料結構，設計成任何順序合併結果都一樣

---

## Cache 層的同步

Cache（Redis）是另一個需要同步的地方。

### 常見的 Cache 更新策略

#### Cache-Aside（最常見）

```
讀取：
  先查 Redis → 有 → 直接回傳（Cache Hit）
              沒有 → 查 DB → 寫入 Redis → 回傳（Cache Miss）

寫入：
  寫入 DB → 刪除 Redis 的對應 Cache（讓下次讀取重新載入）
```

**為什麼是「刪除」而不是「更新」Cache？**

```
更新 Cache 有 race condition：

T+1  Server A：更新 DB（name = Wei-Chun）
T+2  Server B：更新 DB（name = Peter）
T+3  Server B：更新 Cache（name = Peter）← 先到
T+4  Server A：更新 Cache（name = Wei-Chun）← 後到，覆蓋了！

最終 DB = Peter，Cache = Wei-Chun → 不一致

刪除 Cache 就不會有這個問題：
  誰先更新 DB 誰後更新 DB，都只是把 Cache 刪掉
  下次讀取時重新從 DB 撈 → 永遠拿到最新的
```

#### Write-Through

```
寫入時：同時更新 DB 和 Cache（兩個都寫）

優點：Cache 永遠和 DB 一致
缺點：每次寫入都要打兩個地方，速度稍慢
```

#### Write-Behind（Write-Back）

```
寫入時：只寫 Cache → 立刻回傳成功 → 非同步批次寫入 DB

優點：極快，減少 DB 壓力
缺點：Cache 掛掉 → 資料遺失
適合：可以接受短暫資料遺失的場景（如計數器、點擊數）
```

---

## Session 同步

Session 是最典型的「多台 Server 要共享狀態」問題。

```
解法一：Sticky Session（不是真正的同步，而是避免同步）
  → 同一個 User 永遠送同一台 Server
  → 缺點：Server 掛掉就消失

解法二：集中式 Session Store（Redis）
  → 所有 Server 都去 Redis 讀寫 Session
  → 任何 Server 都能服務任何 User

解法三：JWT（完全不存 Session）
  → Session 資訊加密後存在 Token 裡，Server 無狀態
  → 根本不需要同步
```

---

## 應用層的即時同步（WebSocket / 聊天室情境）

多台 Server 時，用戶連在不同 Server 上，怎麼廣播訊息？

```
問題：
  User A 連在 Server 1
  User B 連在 Server 2
  User A 發訊息 → 只有 Server 1 知道 → Server 2 不知道 → User B 收不到

解法：Pub/Sub（Redis 或 Kafka）

  User A 發訊息
      ↓
  Server 1 收到 → 發布到 Redis Channel "room:123"
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
                Server 1              Server 2
                訂閱了 "room:123"     訂閱了 "room:123"
                把訊息推給連在        把訊息推給連在
                自己的 User          自己的 User B ✅
```

---

## 定期同步 vs 事件觸發，誰在用？

| 機制 | 同步方式 | 原因 |
|------|----------|------|
| DB Replication | 事件觸發（binlog）| 即時性要求高，定期會有資料空窗 |
| Cache 更新 | 事件觸發（寫入後刪除）| 定期同步會有很長的不一致窗口 |
| Redis Pub/Sub | 事件觸發（有訊息就推）| 聊天室不能靠輪詢 |
| Message Queue | 事件觸發 | Kafka/SQS 本質就是事件驅動 |
| 搜尋索引同步 | 定期 + 事件混合 | ES 索引可以接受幾秒的延遲 |
| 備份（Backup）| 定期（每天/每小時）| 備份本來就不是即時同步 |
| DNS TTL 快取刷新 | 定期（TTL 到期）| DNS 協定設計如此，無法事件觸發 |

**結論：幾乎所有「需要低延遲的同步」都是事件觸發，定期同步只用在可以接受延遲的場景（備份、搜尋索引）。**

---

## 一致性的取捨（CAP 定理）

分散式系統有個根本限制：**C、A、P 三個只能同時滿足兩個**。

```
C（Consistency）   一致性：所有節點看到的資料一樣
A（Availability）  可用性：系統永遠能回應
P（Partition Tolerance）分區容錯：網路斷了系統還能運作

網路分區（P）在分散式系統中幾乎一定會發生，所以只能選：

CP：保證一致性，犧牲可用性
    → 網路斷了，寧可回傳錯誤，也不回傳舊資料
    → 適合：銀行帳戶、庫存（餘額不能出錯）

AP：保證可用性，犧牲一致性（最終一致性）
    → 網路斷了，繼續服務，但可能回傳稍舊的資料
    → 適合：社群媒體按讚數、商品瀏覽數（差幾秒沒差）
```

大多數 Web App 選 **AP + 最終一致性**：「讓用戶先看到舊資料，幾秒後自動同步到最新」，比「讓用戶看到 500 Error」好得多。

---

## 資料不一致的時候怎麼辦？

不一致分兩種情況，解法完全不同。

---

### 情況一：暫時不一致（Replication Lag）

Replica 還沒追上 Primary，這是正常現象，通常幾毫秒到幾秒就會自動補上。

```
T+0ms   用戶改密碼 → 寫入 Primary
T+0ms   立刻查詢   → 被 LB 導到 Replica → 還是舊密碼 ← 短暫不一致
T+50ms  Replica 追上 → 之後查詢都是新密碼 ✅
```

**解法：依場景選擇容忍度**

```
可以接受短暫不一致的場景（大多數都是這樣）：
  社群貼文的按讚數、商品瀏覽次數、推薦內容
  → 直接讀 Replica，接受幾秒誤差，沒差

不能接受的場景（錢、帳號安全）：
  剛付款後查訂單、剛改密碼後登入、帳戶餘額
  → 強制讀 Primary（Read-your-own-writes）
```

**Read-your-own-writes 的實作：**

```
方法一：寫入後的請求強制打 Primary
  在 Header 或 Session 裡記錄「最近有寫入」
  這段時間的讀取都走 Primary

方法二：用 replication lag 時間戳判斷
  Primary 回傳寫入的時間戳
  讀取時帶上這個時間戳
  如果 Replica 還沒追到這個時間點 → 改打 Primary

方法三：寫完直接從 Primary 讀一次，存進 Cache
  下次讀取先查 Cache → 拿到剛寫的資料
  Cache 到期後 Replica 已經追上了
```

---

### 情況二：真正的衝突（Conflict）

多台機器同時對同一筆資料寫入，無法自動決定誰對。

**例子：兩個人同時編輯同一份文件**

```
T+0ms  User A 打開文件：「今天天氣很好」
T+0ms  User B 打開文件：「今天天氣很好」

T+5s   User A 改成：「今天天氣很好，適合出門」→ 存入 Server 1
T+5s   User B 改成：「今天天氣很好，但我不想出門」→ 存入 Server 2

Server 1 和 Server 2 要同步 → 兩個版本，誰贏？
```

#### 解法一：Last Write Wins（時間戳決勝）

```
比較兩個寫入的時間戳，較新的覆蓋較舊的。

優點：簡單
缺點：可能丟資料，而且分散式系統的時間不可靠
     （不同機器的時鐘可能差幾毫秒，誰「新」誰「舊」不一定準確）

適合：社群媒體大頭貼更新（丟掉其中一個版本沒關係）
```

#### 解法二：Version Vector（版本向量）

每次寫入帶上版本號，系統可以偵測到衝突並要求人工處理。

```
User A 的寫入：{ version: 1, content: "...適合出門" }
User B 的寫入：{ version: 1, content: "...不想出門" }

都是從 version 1 衍生出來的 → 系統偵測到衝突
→ 通知雙方：「有版本衝突，請確認」

Amazon Dynamo、Git 都用這個概念
```

#### 解法三：CRDT（Conflict-free Replicated Data Type）

設計一種特殊資料結構，讓任何順序的合併結果都一樣，從根本上消除衝突。

```
計數器（Counter CRDT）：
  Server 1：+3
  Server 2：+5
  合併：不管誰先誰後，結果永遠是 +8

集合（Set CRDT）：
  Server 1：新增 "apple"
  Server 2：新增 "banana"
  合併：{"apple", "banana"}，順序不影響結果

文字編輯（如 Google Docs）：
  不存整段文字，而是存「操作」（在第5個字後插入"好"）
  兩個操作可以被正確合併，不會互相覆蓋
```

Google Docs、Figma 的多人即時協作就是用類似 CRDT 的機制。

#### 解法四：Pessimistic Locking（悲觀鎖）

「我在編輯這筆資料，別人不能動。」

```
User A 開始編輯 → 系統鎖定這筆資料
User B 想編輯   → 「目前有人在編輯，請稍後」或排隊等待
User A 存檔     → 解鎖 → User B 才能編輯
```

**優點**：完全沒有衝突
**缺點**：效能差，User A 離開頁面忘記儲存 → 資料永遠鎖住（要加 timeout 解鎖機制）
**適合**：資料庫 Transaction、不能有任何衝突的金融操作

---

### 各場景的選擇

| 場景 | 問題類型 | 解法 |
|------|----------|------|
| 剛改密碼立刻查詢 | 暫時不一致 | Read-your-own-writes，走 Primary |
| 電商庫存扣減 | 衝突（超賣）| Pessimistic Lock 或 DB Transaction |
| 社群按讚數 | 暫時不一致 | 接受幾秒誤差，讀 Replica |
| Google Docs 多人編輯 | 衝突（文字）| CRDT |
| Git 合併 | 衝突（程式碼）| Version Vector + 人工 merge |
| 用戶大頭貼更新 | 衝突（可丟資料）| Last Write Wins |
| 銀行轉帳 | 衝突（錢）| 強一致性，Pessimistic Lock + Transaction |

---

### 核心心態

**不是「怎麼讓系統永遠一致」，而是「為每個場景選擇可接受的不一致程度」。**

```
Google 搜尋結果慢幾秒更新 → 沒差，接受最終一致
銀行餘額少 1 塊錢 → 不行，要強一致

過度追求一致性 → 效能差、系統複雜
過度放鬆一致性 → 資料錯亂、用戶信任崩潰

工程師的工作是找到這條線。
```

---

## Primary DB 壞掉怎麼辦？

這是系統設計裡最重要的問題之一，有完整的一套機制處理。

### 問題有多嚴重？

```
Primary 壞掉的瞬間：
  ✅ 讀取：還能用（Replica 還活著）
  ❌ 寫入：全部失敗（沒有 Primary 可以寫）
  ❌ 部分資料：可能遺失（非同步複製的情況下，最後幾筆還沒同步的資料消失）
```

---

### 解法一：自動 Failover（現代主流）

讓系統自己偵測 Primary 掛掉，自動把某台 Replica 升級為新的 Primary。

```
正常：
  App ──寫入──► Primary
              └──複製──► Replica 1
              └──複製──► Replica 2

Primary 掛掉：
  Step 1：Replica 1、2 偵測到 Primary 沒有心跳
  Step 2：投票選出新的 Primary（通常選資料最新的那台）
  Step 3：Replica 1 升格為 Primary
  Step 4：DNS / App 自動切換到新 Primary
  Step 5：Replica 2 改向新 Primary 複製

  整個過程：通常 30 秒以內完成
```

**選舉機制（Raft / Paxos）**：

多台 Replica 要選出新 Primary，用分散式共識演算法決定，防止「腦裂」（兩台都以為自己是 Primary）。

```
腦裂問題：
  Primary 網路斷了，但還活著
  Replica 以為 Primary 掛了，選自己當 Primary
  → 兩台都在接受寫入 → 資料分裂

解法：Quorum（多數決）
  只有拿到超過半數節點同意，才能當 Primary
  3 台節點：至少 2 台同意才能升格
  → 不可能同時有兩台都拿到多數票
```

---

### 實際服務怎麼做

#### AWS RDS Multi-AZ

```
Region: ap-northeast-1（東京）

  AZ-a（可用區 A）        AZ-c（可用區 C）
  ┌─────────────┐         ┌─────────────┐
  │  Primary DB │──同步──►│  Standby DB │
  │  (活躍)     │  複製    │  (備援)     │
  └─────────────┘         └─────────────┘

  AZ-a 機房火災/斷電/Primary 掛掉：
  → AWS 自動把 Standby 升格為 Primary
  → DNS endpoint 自動切換
  → 通常 60-120 秒完成
  → App 不需要改任何程式碼（連線字串不變）
```

AZ（Availability Zone）= 同一個 Region 裡物理隔離的資料中心，不同 AZ 不共用電力和網路，一個 AZ 掛掉不影響另一個。

#### MongoDB Replica Set

```
  Primary
     │
     ├──► Secondary 1  （平時可分擔讀取）
     └──► Secondary 2  （平時可分擔讀取）

Primary 掛掉 → Secondary 1、2 投票 → 選出新 Primary
自動完成，應用層感受到的只是幾秒的寫入失敗
```

#### PlanetScale / Vitess（MySQL 雲端版）

自動處理 Failover，對開發者完全透明，Primary 掛掉重建的過程感知不到。

---

### 解法二：跨 Region 備援（更極端的情況）

整個資料中心（Region）掛掉怎麼辦？

```
正常：
  台灣用戶 ──► ap-northeast-1（東京）Primary DB

東京整個 Region 掛掉（颱風、海底電纜斷）：

  ┌─────────────────────┐     ┌──────────────────────┐
  │  ap-northeast-1     │     │  us-west-2           │
  │  東京（掛掉）❌      │     │  奧勒岡（備援）       │
  │                     │     │                      │
  │  Primary DB 💀      │────►│  Replica DB          │
  └─────────────────────┘     └──────────────────────┘
                                        │ 升格為 Primary
                                        ▼
                               台灣用戶切換過來
```

代價：跨 Region 的 Replication 延遲比同 Region 高（幾十毫秒 vs 幾毫秒），且切換時間更長（幾分鐘）。

---

### 還是會遺失資料？RPO vs RTO

即使有 Failover，還是可能遺失少量資料，這用兩個指標衡量：

```
RPO（Recovery Point Objective）— 能接受遺失多少資料？
  「Primary 掛掉的瞬間，最多遺失幾秒/幾分鐘的資料？」

  非同步複製：可能遺失最後幾秒還沒同步的資料
  同步複製：RPO = 0，完全不遺失（但效能差）

RTO（Recovery Time Objective）— 能接受停機多久？
  「從 Primary 掛掉到服務恢復，最多幾分鐘？」

  AWS RDS Multi-AZ：RTO ≈ 1-2 分鐘
  跨 Region Failover：RTO ≈ 5-30 分鐘
  手動恢復（沒有自動化）：RTO ≈ 幾小時
```

**不同業務的要求：**

```
一般 Web App：RPO = 幾秒可接受，RTO = 幾分鐘可接受
電商訂單：RPO = 0（不能遺失訂單），RTO = 1 分鐘內
銀行核心系統：RPO = 0，RTO = 秒級，且要 24/7 有人 on-call
```

---

### 最後一道防線：定期備份

Failover 是應對「Primary 掛掉」，備份是應對「資料被刪掉或損壞」。

```
情境：
  工程師手誤執行 DROP TABLE users; ← Failover 救不了你
  Replica 也會跟著複製這個操作 → 所有節點資料都沒了

這時候只能從備份還原：
  AWS RDS 自動備份：每天一次快照 + 連續 transaction log
  → 可以還原到任意時間點（Point-in-time Recovery）
  → 但還原需要時間（幾分鐘到幾小時，視資料量）
```

---

### 總結：Primary 掛掉的防護層

```
Layer 1：同 AZ 的 Primary-Replica
  → 應對：Primary 機器故障
  → 切換時間：秒級
  → RPO：幾毫秒

Layer 2：跨 AZ 的 Multi-AZ（AWS RDS）
  → 應對：整個 AZ 斷電 / 機房火災
  → 切換時間：1-2 分鐘
  → RPO：幾秒（同步複製可到 0）

Layer 3：跨 Region 備援
  → 應對：整個城市/國家的災難
  → 切換時間：幾分鐘到幾十分鐘
  → RPO：幾秒到幾分鐘

Layer 4：定期備份（S3 / Glacier）
  → 應對：人為誤刪、資料損壞
  → 還原時間：幾分鐘到幾小時
  → RPO：最後一次備份到現在的資料

越往下，成本越高，平時用不到，但沒有的時候就完蛋。
```
