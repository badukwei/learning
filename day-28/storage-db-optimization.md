# Storage & DB Optimization

> 來源：舊課程（~2012 Harvard CS75）+ 現代對比整理
> 核心問題：應用程式 I/O 成為瓶頸時，怎麼優化？

---

## 總覽架構

```
請求 → App Server → Cache（記憶體）→ DB（磁碟）
                         ↑ 能擋住就不落磁碟
```

優化順序（永遠從最便宜的開始）：
1. 加 Cache，減少打 DB 的次數
2. 優化 DB 本身（Index、Engine、Query）
3. 讀寫分離（Replica）
4. 水平擴展（Sharding）
5. 換架構（NewSQL、Column Store 等）

---

## 一、硬體層

### 舊觀念（2012）

| 類型 | 特性 |
|------|------|
| SATA HDD 7200 RPM | 標準，便宜，慢 |
| SAS HDD 15k RPM | 快一倍，貴，企業用 |
| SSD | 無機械移動，快，但貴且容量小 |

**RAID**

| 類型 | 原理 | 用途 |
|------|------|------|
| RAID 0（Striping）| 資料切片分散寫 | 速度快，沒備援 |
| RAID 1（Mirroring）| 寫兩份鏡像 | 備援，空間浪費 50% |
| RAID 5/6 | 分散 + 奇偶校驗 | 平衡效能/備援 |

### 現代（2024+）

- **NVMe SSD 已是標準**：比 SATA SSD 快 5-10 倍，比 HDD 快 100x+，價格已大幅下降
- **雲端環境**：AWS EBS、GCP Persistent Disk 在雲層做備援，硬體 RAID 通常不需要自己管
- **磁碟 I/O 已不是首要瓶頸**：NVMe 很快，現在瓶頸更多在 CPU、記憶體、網路
- RAID 在自建機房（on-prem）仍然相關，雲端環境基本由平台處理

---

## 二、快取策略（Caching）

### 舊觀念（2012）

1. **靜態化（Static HTML）**：把動態頁面預先渲染成 HTML 存磁碟，Apache 直接回傳，不跑 PHP + DB
2. **MySQL Query Cache**：DB 記住相同 SQL 的結果，下次直接回傳
3. **Memcached**：RAM Key-Value 存資料，極快，Facebook 大量使用

### 現代（2024+）

**MySQL Query Cache → 已死**
- MySQL 5.7.20 deprecated，**MySQL 8.0 完全移除**
- 原因：鎖競爭嚴重，Table 任何一筆寫入就讓整個 Cache 失效，反而成瓶頸

**Memcached → 被 Redis 取代（大多數場景）**

| 特性 | Memcached | Redis |
|------|-----------|-------|
| 資料結構 | 只有 String | String / Hash / List / Set / Sorted Set / Stream |
| 持久化 | 無 | 有（AOF / RDB） |
| Cluster | 有，但功能少 | Redis Cluster / Sentinel |
| Pub/Sub | 無 | 有 |
| 適用場景 | 純快取，大量簡單 KV | 快取 + Session + 排行榜 + 消息隊列 |

**Redis 實際使用場景**

```
1. API Response 快取
   key = "user:123:profile"
   value = JSON string（設 TTL 5 分鐘）

2. Session 儲存
   key = "session:{token}"
   value = user_id + 權限（設 TTL = 登入有效期）
   優點：水平擴展時多台 App Server 共享 Session

3. Rate Limiting（限流）
   key = "rate:{user_id}:{分鐘時間戳}"
   INCR 計數 → 超過 100 就擋（配合 EXPIRE）

4. 排行榜
   Sorted Set：ZADD leaderboard 9500 "user:123"
   ZREVRANGE leaderboard 0 9 → 取前 10 名

5. 分散式鎖（防止 Race Condition）
   SET lock:order:456 "process1" NX EX 30
   NX = 不存在才設（atomic），EX = 30秒自動釋放

6. 消息隊列（輕量）
   LPUSH queue:email {payload}
   BRPOP queue:email 0（blocking pop）
   → 重度場景改用 Kafka
```

**快取常見問題**

```
Cache Stampede（快取雪崩）：
  問題：大量 key 同時過期 → 全部打到 DB → DB 過載
  解法：
    1. TTL 加隨機抖動（TTL = base + random(0, 300)）
    2. 非同步更新：快取快到期時背景重建，不等使用者觸發

Cache Penetration（快取穿透）：
  問題：查詢不存在的資料 → Cache miss → 每次都打 DB
  解法：
    1. DB 也查不到 → Redis 存空值（TTL 短）
    2. Bloom Filter：先判斷 key 是否可能存在

Hot Key（熱點問題）：
  問題：某個 key 被瘋狂讀取 → 單一 Redis node 成瓶頸
  解法：
    1. 本地快取（App Server 記憶體）+ Redis 雙層
    2. key 加 shard 後綴（"user:123:1"、"user:123:2"）分散
```

**靜態化 → CDN**

```
舊：Nginx 直接回傳 /static/xxx.html
現代：CloudFront / Cloudflare 在全球 Edge 節點快取，使用者就近取用
```

**現代快取模式**

```
Cache-Aside（最常見）：
  讀：先查 Redis → 沒中才查 DB → 寫回 Redis
  寫：先寫 DB → 刪 Redis（讓下次讀重建）

Write-Through：
  寫：同時寫 Cache + DB（強一致）

Write-Behind（Write-Back）：
  寫：先寫 Cache，非同步批次寫 DB（高寫入量用）
```

---

## 三、資料庫擴展

### 舊觀念（2012）

**主從複製（Master-Slave）**
```
Write → Master → 複製 → Slave 1
                      → Slave 2（讀）
```

**主主複製（Master-Master）**
```
Write ←→ Master A ←→ Master B
```
兩台互備，任一掛掉寫入不中斷

**Sharding（資料分割）**
- 按姓氏 A-M → DB 1，N-Z → DB 2
- 按機構分（Harvard → DB 1，MIT → DB 2）
- 應用層自己決定打哪個 DB

### 現代（2024+）

**命名更新**：Master/Slave → **Primary/Replica**（業界去除歧視性詞彙）

**讀寫分離標配化**

```
App → ProxySQL / PgBouncer / RDS Proxy
           ↓ 寫入           ↓ 讀取
       Primary DB       Replica 1/2/3
```

- 應用層不用直接管理哪個 DB，Proxy 自動路由
- AWS RDS / Aurora 原生支援 Read Replica，一鍵建立

**Aurora（AWS）**

```
舊：Primary 複製 WAL log 到 Replica（網路開銷大）

Aurora：
  Primary ─────┐
  Replica 1 ──→│→ 共用 Storage Layer（6 份，跨 3 AZ）
  Replica 2 ──→│
               └─ S3 備份

複製是在 Storage 層發生，不走 Primary，Replica lag 極低（<10ms）
```

**Sharding 策略細節**

```
Range Sharding（範圍）：
  user_id 1-1M → Shard 1
  user_id 1M-2M → Shard 2
  優：範圍查詢快
  缺：Hot Shard（新用戶都在同一台）

Hash Sharding（雜湊）：
  shard = hash(user_id) % N
  優：均勻分散
  缺：範圍查詢要打所有 Shard，擴容要 rehash

Consistent Hashing（一致性雜湊）：
  用環狀結構，新增節點只影響相鄰 key
  優：擴容影響最小，適合動態節點數

Directory-based Sharding（查表）：
  另開一張表記 user_id → shard_id 的對應
  優：最靈活，可手動調整
  缺：這張對應表本身可能成瓶頸
```

**跨 Shard 查詢問題**

```
問題：SELECT * FROM orders JOIN users → users 在 Shard 1，orders 在 Shard 2
解法：
  1. 資料設計盡量讓同一個 user 的資料在同 Shard（Locality）
  2. 接受在應用層做 Join（查兩次，程式合）
  3. 改用 Vitess 或 NewSQL，由框架處理
```

**Sharding 現代解法**

| 工具 | 用途 |
|------|------|
| Vitess | YouTube / PlanetScale 的 MySQL Sharding 中間層，應用層不感知 |
| Citus | PostgreSQL 分散式擴展，SQL 相容 |
| PlanetScale | 基於 Vitess 的 Serverless MySQL |
| Neon | Serverless PostgreSQL，Branch 功能（類似 Git branch DB）|

**NewSQL（舊 SQL 解不了的問題）**

| 特性 | 舊 MySQL | NewSQL（CockroachDB / TiDB / Spanner）|
|------|----------|--------------------------------------|
| 水平擴展 | 需手動 Sharding | 自動分散式，透明擴展 |
| 強一致性 | 只有單機 | 分散式 ACID（使用 Raft / Paxos）|
| SQL 相容 | 完整 | 大部分相容（PostgreSQL wire protocol）|
| 跨區 | 複雜 | 原生支援 Multi-Region |

---

## 四、儲存引擎

### 舊觀念（2012）

| Engine | 特性 |
|--------|------|
| InnoDB | 支援 Transaction，現代預設 |
| Memory | 存 RAM，快，重啟消失 |
| Archive | 壓縮存放，慢查詢，省空間，存 Log |
| MyISAM | 無 Transaction，老舊 |

### 現代（2024+）

- **MySQL 8.0**：MyISAM、Archive 等引擎已 legacy，不建議用
- **InnoDB 幾乎是唯一選擇**（OLTP 場景）
- Memory 引擎已被 Redis 取代，不再手動用

**不同場景用不同 DB（多模架構）**

```
OLTP（交易型）→ MySQL / PostgreSQL（InnoDB）
OLAP（分析型）→ ClickHouse / BigQuery / Redshift（Column Store）
時序資料       → TimescaleDB / InfluxDB
全文搜尋       → Elasticsearch / OpenSearch
文件型         → MongoDB
圖資料         → Neo4j
```

---

## 五、新增（舊課程沒提的重要概念）

### Connection Pooling

```
問題：每次 DB 連線開銷大（TCP handshake + auth）
解法：維持固定連線 Pool，App 從 Pool 借用

工具：PgBouncer（PostgreSQL）、ProxySQL（MySQL）
```

### WAL（Write-Ahead Log）

```
每次寫入 DB，先寫 WAL Log → 再寫實際資料頁
優點：crash 後可 replay WAL 恢復，不怕中途斷電
這也是 Replication 的底層機制
```

### MVCC（Multi-Version Concurrency Control）

```
讀不鎖寫，寫不鎖讀
每筆資料有多個版本（含時間戳），讀取看到「快照」
InnoDB / PostgreSQL 都用這個機制，是現代高並發的基礎
```

**Transaction Isolation Levels（交易隔離層級）**

從低到高，安全性越高、並發越低：

| 層級 | Dirty Read | Non-repeatable Read | Phantom Read | 說明 |
|------|-----------|---------------------|-------------|------|
| Read Uncommitted | 會 | 會 | 會 | 幾乎不用，太危險 |
| Read Committed | 不會 | 會 | 會 | PostgreSQL 預設 |
| Repeatable Read | 不會 | 不會 | 會（MySQL 用 MVCC 防住了）| MySQL InnoDB 預設 |
| Serializable | 不會 | 不會 | 不會 | 最安全，效能最差 |

```
Dirty Read：讀到另一個 Transaction 還沒 commit 的資料
  → 對方 rollback，你讀到的是假資料

Non-repeatable Read：同一 Transaction 讀同一筆，兩次結果不同
  → 中間被別人改了

Phantom Read：同一 Transaction 查同一條件，兩次行數不同
  → 中間被別人新增/刪除了
```

**實際使用場景**

```
電商下單：
BEGIN;
  SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- 鎖這筆
  UPDATE products SET stock = stock - 1 WHERE id = 1;
  INSERT INTO orders ...;
COMMIT;

FOR UPDATE = 悲觀鎖，確保同時間只有一個 Transaction 能修改
樂觀鎖（Optimistic Locking）：加 version 欄位，更新時 WHERE version = old_version
  → 高並發 + 衝突少時效能更好
```

### ACID vs BASE

```
ACID（傳統 RDB）：
  Atomicity    全成功或全失敗（Transaction 要嘛全 commit，要嘛全 rollback）
  Consistency  資料永遠符合約束（外鍵、唯一值等）
  Isolation    多個 Transaction 互不干擾
  Durability   Commit 後就算斷電也不消失（靠 WAL）

BASE（分散式 NoSQL）：
  Basically Available  系統大部分時間可用（允許部分節點失敗）
  Soft State           資料狀態可能短暫不一致
  Eventually Consistent 最終會一致，但不保證即時

選擇原則：
  金融、庫存、訂單 → 必須 ACID（PostgreSQL / MySQL）
  社群動態、推薦、計數器 → BASE 可接受（MongoDB / Cassandra / DynamoDB）
```

### Index 策略（現代重點）

```
B-Tree Index：範圍查詢 / 排序，大多數場景（預設）
Hash Index：只適合等值查詢（= 號），不支援範圍
Composite Index：多欄位，注意 Leftmost Prefix 規則
  (a, b, c) → 可走 a、a+b、a+b+c，不能只走 b 或 c
Covering Index：查詢所需欄位全在 Index 裡，不回表（最快）
  SELECT name, email FROM users WHERE email = ?
  → 建 INDEX(email, name) → 直接在 Index 拿完，不碰主表
Partial Index：只對部分資料建 Index（PostgreSQL 支援）
  CREATE INDEX ON orders(user_id) WHERE status = 'pending';
  → 只索引未完成訂單，Index 小、快
```

**什麼時候不該加 Index**

```
1. 低基數（Low Cardinality）欄位：性別（M/F）、布林值
   → 反而比全表掃描慢（Index 效益低，還多一層查找）
2. 寫入頻繁的表：每次 INSERT/UPDATE 都要更新 Index
3. 小表（< 數千行）：全表掃描比走 Index 還快
```

**EXPLAIN 分析 Query**

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;
```

```
看幾個關鍵欄位：
  type: ALL = 全表掃描（壞），ref/range/eq_ref = 有走 Index（好）
  rows: 預估掃描行數，越少越好
  Extra: Using filesort（排序沒走 Index，壞）、Using index（Covering Index，好）
```

### Query Optimization（常見問題）

**N+1 Problem**

```
壞：
  users = SELECT * FROM users;          -- 1 次查詢
  for user in users:
    orders = SELECT * FROM orders       -- N 次查詢
              WHERE user_id = user.id

好：
  SELECT users.*, orders.*
  FROM users LEFT JOIN orders
  ON users.id = orders.user_id          -- 1 次查詢搞定

或用 ORM 的 eager loading（Rails: includes, Django: select_related）
```

**Slow Query Log**

```sql
-- MySQL 開啟慢查詢 log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;  -- 超過 1 秒才記錄

-- PostgreSQL
SELECT * FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;
```

找出最慢的 Query → 加 Index 或改寫 Query

---

## 新舊對比總結

| 主題 | 2012 做法 | 2024 做法 |
|------|-----------|-----------|
| 硬體 | SATA/SAS RAID | NVMe SSD / 雲端 Storage，平台管備援 |
| 快取 | MySQL Query Cache + Memcached | Redis（多資料結構）+ CDN |
| 讀寫分離 | 手動 Master-Slave | RDS / Aurora Read Replica + Proxy 自動路由 |
| Sharding | 應用層手動 | Vitess / PlanetScale / NewSQL |
| 儲存引擎 | 多選（Memory/Archive） | InnoDB 統一，分析改用 ClickHouse 等 |
| 連線管理 | 直連 | Connection Pool（PgBouncer/ProxySQL）|
| 備援/複製 | WAL 複製 Slave | Aurora 共用 Storage / Spanner 分散式 |
| 一致性 | ACID 單機 | ACID + BASE 依場景選，NewSQL 給分散式 ACID |
| Sharding 策略 | 按欄位值硬切 | Range / Hash / Consistent Hashing / Vitess |
| Query 分析 | 靠經驗 | EXPLAIN ANALYZE + Slow Query Log + pg_stat_statements |
| 快取問題 | 無 | Cache Stampede / Penetration / Hot Key 有對應解法 |

**核心思維沒變的**：
- 快取 > DB（永遠）
- 讀寫分離減壓力
- Sharding 是最後手段（複雜度高，先確定真的需要）
- 硬體可以買，架構要先設計好

**決策框架（面試常問）**

```
流量小（< 1M DAU）：
  單一 PostgreSQL + Redis Cache + CDN → 夠用

讀多（> 80% 讀）：
  加 Read Replica + Redis Cache

寫多：
  Connection Pool + Write-Behind Cache + 評估 Sharding

超大規模（> 10M DAU）：
  Sharding（Vitess）或 NewSQL（TiDB / Spanner）
  Column Store for analytics（ClickHouse）
  多模架構（不同資料用不同 DB）
```
