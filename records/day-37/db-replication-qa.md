# Day 37 - DB Replication Q&A

Date: 2026-05-05

## 題目 1

假設你的系統有一台 `Primary DB` 跟兩台 `Replica DB`。

使用者剛在 `Primary` 改完密碼，結果下一秒刷新頁面，卻還看到舊資料。

請回答：

1. 這種情況為什麼會發生？
2. 背後是哪個機制造成的？
3. 你會怎麼解？

## 當下回答（原始版本）

- 更新密碼時，auth session 沒有同步更新到
- 快取也沒有同步更新到
- 前端頁面的快取設定可能也還沒觸發 TTL
- 解法是更改用戶資料時刪除快取，重新登入時從 DB 更新資料到快取

## 校正重點

這個回答有碰到「舊資料」的直覺，但把幾個不同層的問題混在一起了：

- `DB replication lag`
- `session 狀態`
- `cache invalidation`
- `前端快取`

如果題目已經明確指定是 `Primary DB + Replica DB`，第一優先要想到的是：

**這是 DB 複製延遲（replication lag），不是先怪 session 或前端快取。**

## 標準答案

### 1. 為什麼會發生

因為資料是先寫到 `Primary`，但後續讀取請求可能被導到 `Replica`。

如果 `Replica` 還沒同步到最新資料，使用者就會讀到舊值。

## 2. 背後機制

這通常是 `Primary -> Replica` 的**非同步複製**造成的。

流程大致是：

1. 使用者修改密碼
2. 寫入成功落在 `Primary`
3. `Primary` 把變更記錄到 `binlog` 或 `WAL`
4. `Replica` 非同步讀取這些 log 並重播
5. 在重播完成前，如果讀請求被送到 `Replica`，就會看到舊資料

這個時間差就叫：

**replication lag**

### 3. 怎麼解

常見解法：

#### 解法一：寫入後短時間強制讀 Primary

對剛做完寫操作的使用者，在接下來一小段時間內，讀請求直接走 `Primary`。

這叫：

**read-after-write consistency** 或 **read-your-own-writes**

適合：

- 改密碼
- 剛付款後查訂單
- 剛更新個人資料後立刻刷新

#### 解法二：對敏感場景套用 read-your-own-writes

不是所有查詢都要強一致。

例如：

- 社群按讚數可以接受延遲
- 帳號安全、付款、訂單不行

所以通常只對敏感流程強制讀 `Primary`，其他照樣讀 `Replica`。

#### 解法三：同步複製

讓 `Primary` 寫入後，等 `Replica` 也確認成功，才回傳成功。

優點：

- 幾乎沒有 replication lag
- 一致性更強

缺點：

- 寫入延遲變高
- 吞吐量下降

適合：

- 金融
- 支付
- 高一致性要求的核心資料

#### 解法四：如果系統還有 Cache，再搭配 cache invalidation

如果除了 DB 複製延遲，系統還有 Redis / application cache，那寫入後也要處理 cache invalidation。

但這是第二層問題，不是這題最優先的答案。

## 面試版回答

如果使用者剛寫入資料到 `Primary`，下一秒讀請求卻被導到 `Replica`，而 `Replica` 還沒完成同步，就會看到舊資料。這通常是 `Primary-Replica` 非同步複製造成的 `replication lag`。常見解法是對敏感操作使用 `read-your-own-writes`，也就是寫入後一小段時間內強制讀 `Primary`；如果一致性要求更高，可以改成同步複製，但代價是寫入延遲上升。

## 關鍵詞

- `Primary / Replica`
- `replication lag`
- `binlog`
- `WAL`
- `read-after-write consistency`
- `read-your-own-writes`
- `asynchronous replication`
- `synchronous replication`

## 下一題

`Primary-Replica replication` 通常不是定期每 5 分鐘同步一次。

請回答：

1. 它通常是怎麼同步的？
2. `binlog` 或 `WAL` 在這裡扮演什麼角色？
3. 為什麼這種方式比定期全量同步更合理？

---

## 題目 2

假設有一台 `Primary`、兩台 `Replica`。

請回答：

1. 使用者送出 `UPDATE users SET password = 'new' WHERE id = 1` 之後，這筆資料從 `Primary` 到 `Replica` 的同步流程是什麼？
2. `binlog` 或 `WAL` 是先寫還是資料頁先寫？
3. 為什麼系統不直接每 5 分鐘把整份資料庫複製一次就好？

## 當下回答（原始版本）

1. 用戶 call API 到 server 寫入 primary，然後 primary 發 event，接著同步到 replica
2. `binlog` 或 `WAL` 先寫還是資料頁先寫，沒印象
3. 如果資料量多，整份複製會出現大問題，會堵塞，也會吃很多效能

## 校正重點

### 第 1 小題

方向是對的，但描述還不夠精準。

重點不是：

- app server 自己發 event 給 replica

而是：

- `Primary` 先把變更寫進 transaction log
- `Replica` 去持續追這份 log
- 再把變更重播到自己身上

所以更貼近實際的說法是：

**log-based replication**

### 第 2 小題

這題是目前的知識缺口。

要先記住：

**WAL = Write-Ahead Log**

意思就是：

**先寫 log，再寫資料頁**

這樣如果中途 crash，系統還能靠 log replay 回復資料。

### 第 3 小題

這題方向正確，核心就是：

- 成本太高
- 延遲太大
- 大部分資料其實沒變
- 全量同步不適合高頻寫入場景

## 標準答案

### 1. Primary 到 Replica 的同步流程

流程可以這樣理解：

1. 使用者發 request 到 app server
2. app server 對 `Primary DB` 發出寫入操作
3. `Primary` 在處理這筆寫入時，先把變更記錄到 `binlog` 或 `WAL`
4. `Replica` 持續讀取這些 log
5. `Replica` 把相同變更重播到自己身上
6. 同步完成後，`Replica` 才會看到新資料

所以不是每次同步一整份 DB，而是：

**只同步「有變更的部分」**

### 2. binlog / WAL 先寫還是資料頁先寫

先寫的是：

**log**

後寫的是：

**資料頁**

也就是：

**Write-Ahead Log**

核心理由：

- 如果先寫資料頁，寫到一半 crash，可能資料只更新一半
- 如果 log 已經先寫好，系統重啟後還可以根據 log replay，恢復成正確狀態

### 3. 為什麼不用每 5 分鐘整份複製一次

因為這樣會有幾個問題：

1. 延遲太高
- 這 5 分鐘內 `Replica` 都可能是舊資料

2. 成本太高
- 每次都要搬整份資料庫，I/O、CPU、網路都很重

3. 浪費
- 很多資料根本沒變，只改 1 筆卻要重傳全部

4. 同步過程更複雜
- 全量同步還沒做完，又有新的寫入進來，處理起來更麻煩

所以現代資料庫通常採用：

- `incremental replication`
- `event-driven replication`
- `log-based replication`

## 面試版回答

Primary-Replica 同步通常不是定期全量複製，而是 log-based replication。當 app server 對 Primary 寫入資料時，Primary 會先把變更記錄到 binlog 或 WAL，Replica 持續追這些 log，然後把變更重播到自己身上。這比每幾分鐘整份複製一次更合理，因為延遲更低、成本更小，而且只同步真正有變化的資料。

## 補充觀念

- `binlog`：常拿來做 replication
- `WAL`：核心精神是先寫 log，再寫資料頁
- `replay`：系統根據 log 重播操作
- `incremental`：只同步變更，不同步整份

## 下一題

`WAL` 為什麼一定要先寫？

如果系統是：

1. 先寫資料頁
2. 還沒寫 log
3. 中途 crash

會發生什麼問題？

---

## 題目 3

`WAL` 為什麼一定要先寫？

如果系統是：

1. 先寫資料頁
2. 還沒寫 log
3. 中途 crash

會發生什麼問題？

## 當下回答（原始版本）

這比更動就遺失了，造成 primary 跟 replica 的資料不同步，必須要逐一比對或是整個搬移，導致維護成本提高，或是需要暫停服務。

## 校正重點

這個回答抓到了「資料可能遺失 / 不一致」，但重點還是偏到：

- Primary / Replica 不同步
- 後續維護成本

這題第一層核心其實不是 replication，而是：

**crash recovery**

也就是：

`WAL` 先寫，是為了讓資料庫在 crash 後還有依據可以恢復。

## 標準答案

### 為什麼 WAL 一定要先寫

`WAL` 的全名是：

**Write-Ahead Log**

意思就是：

**先寫 log，再寫資料頁。**

原因是：

- log 先記錄這次變更要做什麼
- 之後資料頁才真的被修改
- 如果中途 crash，系統重啟後還可以根據 log replay 或 recovery

### 如果先寫資料頁、還沒寫 log 就 crash，會怎樣

可能出現的問題：

1. 資料頁只寫了一半
- 例如 transaction 涉及多個資料頁，有些改了，有些還沒改

2. 系統沒有 log 可以回放
- 重啟後不知道這筆變更到底做到哪裡

3. 無法可靠 recovery
- 不知道該補做、回滾，還是保持現狀

4. 資料庫可能進入半套狀態
- 有部分變更已落到資料頁
- 但沒有完整紀錄能證明 transaction 的最終狀態

所以最危險的不是單純「資料不同步」，而是：

**資料庫自己都無法可靠判斷自己的狀態。**

## 面試版回答

WAL 要先寫，是因為資料庫必須先把變更安全記錄下來，之後才去更新真正的資料頁。這樣如果中途 crash，系統重啟後可以根據 WAL 做 recovery 或 replay。若先寫資料頁、還沒寫 log 就 crash，可能留下只更新一半的資料，而且沒有 log 可以判斷這筆 transaction 是否完成，會讓資料庫狀態不一致，甚至無法可靠恢復。

## 一句話記法

- `先寫 log = 先留下可恢復的證據`
- `沒先寫 log 就 crash = DB 可能變半套，而且無法可靠恢復`

## 延伸釐清

### 常見誤解：log 是不是主資料庫更新完才有？

不是。

不是：

- 主資料庫先完整更新
- 然後才順便留一份 log

而是：

1. 先把這次變更記進 `WAL`
2. 確保 log 安全寫下來
3. 再去更新真正的資料頁

這就是 `Write-Ahead` 的意思。

### WAL 是不是一個暫存資料庫？

不是。

`WAL` 是資料庫自己的**持久化交易日誌**，不是：

- 快取
- replica
- 給 app 直接查詢的另一個 DB

它通常存在資料庫所在機器的磁碟上，例如 PostgreSQL 的資料目錄裡會有專門放 WAL 的區域。

## 下一題

既然 `Replica` 可能會有 lag，為什麼很多系統還是要用 `Replica`，不乾脆全部都讀 `Primary`？

---

## 題目 4

既然 `Replica` 可能會有 lag，為什麼很多系統還是要用 `Replica`，不乾脆全部都讀 `Primary`？

## 當下回答（原始版本）

- 避免單點故障
- 減少 primary 的負擔
- 功能分離

## 校正重點

這題方向正確，三點都成立。

其中更精確的說法是：

- `減少 Primary 負擔` → 分攤大量讀流量，保護 Primary 的寫入能力
- `功能分離` → 更準確可以說成 `workload isolation`

## 標準答案

很多系統即使知道 `Replica` 會有 replication lag，還是會使用它，主要原因有四個：

### 1. 分攤讀取流量

大部分系統都是讀多寫少。

如果所有查詢都打到 `Primary`，`Primary` 很容易變成瓶頸。

把讀請求分散到 `Replica`，可以提升整體吞吐量。

### 2. 保護 Primary，讓它專心處理寫入

`Primary` 最重要的工作是：

- transaction
- commit
- 寫入一致性

如果它還要同時扛大量讀流量，寫入延遲容易變差。

### 3. 提供高可用與備援

當 `Primary` 掛掉時，可以把某台 `Replica` 升格成新的 `Primary`。

這樣系統不會因為單一資料庫故障就整體停擺。

### 4. 工作負載分流（Workload Isolation）

有些查詢很重，例如：

- 報表
- 後台分析
- 統計查詢

這些可以導到 `Replica`，避免影響線上交易流量。

## 面試版回答

雖然 Replica 可能有 replication lag，但系統還是會使用它，因為它能分攤大量讀取流量、降低 Primary 的負擔，讓 Primary 專心處理寫入，同時也提供高可用與 failover 能力。另外，一些較重的查詢或報表工作也可以放到 Replica 上執行，避免影響主交易流量。

## 延伸釐清：剛改密碼後立刻查，是 Replica 問題還是 Cache 問題？

答案是：

**兩種都有可能，要看題目上下文。**

### 情況 1：Replica lag

流程：

1. 密碼寫入 `Primary`
2. 下一次讀請求被導到 `Replica`
3. `Replica` 還沒追上
4. 讀到舊資料

這是：

- `replication lag`
- `read-after-write consistency` 問題

解法：

- 寫入後短時間強制讀 `Primary`
- 對敏感流程做 `read-your-own-writes`

### 情況 2：Cache stale

流程：

1. DB 裡其實已經是新資料
2. 但 Redis / application cache / frontend cache 還是舊值
3. 讀請求先命中 cache
4. 所以看到舊資料

這是：

- `cache invalidation` 問題

解法：

- 刪除 cache
- 更新 cache
- 控制 TTL

## 一句話記法

如果題目明確在講 `Primary + Replica`，第一反應先答：

**replication lag**

如果題目明確在講 `Redis cache` 或前端快取，第一反應才答：

**cache stale**

---

## 題目 5

有個使用者更新了自己的 `display name`，API 回傳成功。  
但他立刻刷新頁面，還看到舊名字。

系統架構如下：

- 寫入走 `Primary DB`
- 讀取大多走 `Replica DB`
- 前面還有一層 `Redis cache`
- 前端也有 `SWR` 快取

請回答：

1. 有哪幾種可能？
2. 你會怎麼一步一步排查？
3. 如果要快速止血，你會先改哪裡？

## 當下回答（原始版本）

1. 前端快取還在使用舊資料，但不確定重整之後通常會重新呼叫 api，所以這個可能性比較小
2. 快取跟 session 在更新之後沒有同步到 db，或是後端忘記 await 快取更新就回傳成功
3. replica db 還沒寫入新資料，api 就回傳成功，query 的時候找到舊資料

排查方式：

- 先看前端是否有快取機制並且看 log
- 接著看後端程式碼是否在更改敏感資料時，有完整等待所有寫入成功才回傳
- 最後看 db logs 去看同步是否有問題

## 校正重點

這題比前幾題進步，因為已經開始分層思考了。

但有兩個地方要修正：

1. `session` 不是這題的第一優先嫌疑
2. 通常不是「快取同步到 DB」，而是 `DB` 作為 source of truth，cache 跟著 invalidation 或更新

## 標準答案

### 1. 有哪幾種可能

最可能的三層：

#### A. Replica lag

- 寫入已經到 `Primary`
- 但讀請求打到 `Replica`
- `Replica` 還沒追上
- 所以讀到舊名字

#### B. Redis cache stale

- DB 裡其實已經是新資料
- 但 Redis 還留著舊值
- 讀請求先 hit cache
- 所以看到舊名字

#### C. 前端 SWR / React Query stale state

- API 已經回新資料
- 但前端先顯示本地 cache
- 還沒 revalidate 或 invalidate

### 2. 怎麼一步一步排查

#### 第一步：先看 API response 本身是新還是舊

用 DevTools / Network 看刷新後的 API response。

- 如果 response 已經是新名字：
  - 問題偏前端 state / cache
- 如果 response 還是舊名字：
  - 問題偏後端、Redis、或 Replica

#### 第二步：如果 API response 是舊的，先繞過 Redis

- 暫時 bypass cache
- 或直接查 DB

判斷：

- 如果 DB 是新的、API 還回舊的：
  - 很可能是 Redis stale
- 如果 DB 也是舊的：
  - 再往 Primary / Replica lag 看

#### 第三步：確認讀請求打的是 Primary 還是 Replica

如果更新後立刻讀，卻是從 `Replica` 讀，很可能就是 replication lag。

這時要檢查：

- 有沒有 read-after-write 保護
- 有沒有針對敏感流程強制讀 `Primary`

#### 第四步：最後才檢查前端 cache 行為

看：

- SWR / React Query 有沒有 invalidate
- refresh 後是不是還存在 persisted cache
- 本地 state 有沒有覆蓋掉 server response

### 3. 如果要快速止血，先改哪裡

優先做法通常是兩個：

#### A. 更新後短時間強制讀 Primary

這可以直接避開 replication lag。

適合：

- user profile
- security-related settings
- 剛更新完會立刻刷新的資料

#### B. 寫入成功後立即 invalidation Redis cache

例如：

- `user:{id}:profile`

寫成功後先刪掉對應 cache key，下次讀才會從 DB 重建。

如果只能先做一個，通常先做：

**更新後短時間強制讀 Primary**

## 面試版回答

這種舊資料可能來自三層：Replica lag、Redis stale cache，或前端 SWR/React Query 的 stale state。我會先看刷新後 API response 本身是新還是舊。如果 response 已經是新的，問題多半在前端 state 或 cache；如果 response 還是舊的，我會先確認是不是 hit 到 Redis，再確認讀請求是不是走 Replica 而非 Primary。快速止血的做法通常是更新後短時間內強制讀 Primary，並在寫入成功後立即 invalidation 對應的 Redis cache。

## 一句話記法

這類題目的核心不是一開始就猜哪層壞掉，而是：

**先定位舊資料是從哪一層回來的。**

## 下一題

如果更新的是：

- `display name`
- `balance`

哪一種比較可以接受讀 `Replica`，哪一種應該優先讀 `Primary`？為什麼？

---

## 一致性判斷小表

這幾種資料可以先這樣判斷：

| 資料類型 | 比較適合讀哪裡 | 原因 |
|------|------|------|
| `display name` | `Replica` 可接受 | 短暫不一致通常不會造成實際損失 |
| `balance` | 優先讀 `Primary` | 餘額屬於高敏感資料，不能接受舊值 |
| 剛下單後的 `order history` | 優先讀 `Primary` | 使用者剛完成寫入，預期立即看得到 |
| 一般歷史 `order history` | `Replica` 可接受 | 非即時敏感資料，可接受短暫延遲 |
| `like count` | `Replica` 或 `Cache` | 高讀取量，可接受 eventual consistency |

---

## 題目 6

實際情境：

用戶寫入一筆資料，`Primary` 已經寫成功，但在準備同步到 `Replica` 的時候，`Replica` 掛掉了。

這時候會發生什麼？

## 標準答案

### 先講結論

如果是**非同步複製**，通常不會影響這次寫入在 `Primary` 上成功。

會受影響的是：

- 這筆資料暫時不會出現在那台 `Replica`
- `Replica` 會落後
- 系統的備援能力下降

### 流程推演

1. User 發送寫入 request
2. App server 寫入 `Primary`
3. `Primary commit` 成功
4. `Primary` 把變更記錄到 `WAL / binlog`
5. `Replica` 原本要追這段 log
6. 但追到一半 `Replica` 掛掉

### 這時會怎樣

#### 1. Primary 上的資料還在

因為這次寫入已經在 `Primary` 成功 commit，所以 source of truth 沒丟。

#### 2. 那台 Replica 會落後

因為它還沒追完 WAL / binlog，所以它看到的資料會比 `Primary` 舊。

#### 3. Replica 恢復後，通常會補追 log

如果 `Primary` 還保留著這段 log，`Replica` 重啟後通常會從上次追到的位置繼續追，最後重新追上 `Primary`。

所以很多情況下不是整個重搬，而是：

**從斷掉的位置繼續 catch up**

## 什麼時候事情會變麻煩

### 情況 1：Replica 掛太久

如果 `Primary` 已經把舊的 WAL / binlog 清掉，`Replica` 回來後就無法補追缺的那一段。

這時候可能需要：

- 重新做 base backup
- 或重新做一次全量同步

### 情況 2：唯一的 Replica 掛掉

寫入還是可以進 `Primary`，但系統暫時失去備援。

如果這時 `Primary` 再掛，風險就很高。

### 情況 3：系統使用同步複製

如果寫入成功必須等 `Replica` ack，那 `Replica` 掛掉就可能影響：

- 寫入延遲
- 寫入成功率
- 甚至直接卡住寫入

## 同步複製 vs 非同步複製

### 非同步複製

- `Primary` 先成功
- `Replica` 晚點追
- `Replica` 掛掉：主要影響備援與一致性時間差

### 同步複製

- `Primary` 要等 `Replica` ack`
- `Replica` 掛掉：可能直接影響寫入可用性

## 面試版回答

如果 Primary 已經寫成功，但 Replica 在同步途中掛掉，在非同步複製架構下，通常不會影響這次寫入本身，因為 source of truth 已經落在 Primary。影響的是那台 Replica 會暫時落後，系統備援能力下降。Replica 恢復後，如果對應的 WAL 或 binlog 還在，通常可以從中斷位置繼續追；如果掛太久、log 已被清掉，才需要重新做 base backup 或全量同步。若是同步複製，Replica 掛掉就可能直接影響寫入延遲或成功與否。
