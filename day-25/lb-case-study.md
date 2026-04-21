# Load Balancer — Case Study

真實公司的 LB 選擇，重點是「為什麼」而不是「用什麼」。

---

## GitHub — 自己造輪子（GLB）

**背景**：GitHub 是全球最大的 Git hosting，每天處理數億次 push/pull。

**問題**：
HAProxy 維護機器時連線中斷。Git 協議設計上不支援重試——push 到一半斷線，整個操作失敗，用戶要從頭來。這對 GitHub 是 P0 問題。

**嘗試過的方案**：
LVS（L4）+ HAProxy（L7）的組合。LVS 做 director，把封包轉給 HAProxy，但 director node 是 stateful 的——它要記住「這條連線是哪台 HAProxy 在處理」。維護任一台 director node，連線狀態同步就會出問題，還是會斷線。

另一個嘗試：用 multicast 做 failover。但 multicast 在資料中心網路環境複雜，運維成本高，GitHub 不想依賴它。

**最終決定**：
自己用 DPDK 在 commodity hardware 上寫 stateless LB（GLB）。

核心設計三個技術：

**① Rendezvous Hashing**

問題：LB 多台 proxy，怎麼決定「這條連線給哪台」，且任一台掛掉後無縫接管？

傳統 stateful 做法：Director 查表記住「這條連線給 Proxy-2」，Proxy-2 掛掉要更新 state，複雜且有 SPOF。

Rendezvous Hashing 做法：

```
連線進來（src=用戶IP:port, dst=GitHub IP）
         ↓
每台 Proxy 都用同一個公式算分：
  score(Proxy-1) = hash(連線特徵 + "Proxy-1")
  score(Proxy-2) = hash(連線特徵 + "Proxy-2")
  score(Proxy-3) = hash(連線特徵 + "Proxy-3")
         ↓
分數最高的 Proxy 贏，負責這條連線

Proxy-2 掛掉 → 重算忽略 Proxy-2 → Proxy-3 自動接管
不需要 state，不需要通知任何人
```

任何 proxy 用同樣公式都能算出同樣答案，完全 stateless。

**實際封包流程**：

```
用戶
 ↓ TCP 連線（目標永遠是 GitHub 公開 IP，用戶不知道 proxy 存在）
Director（算 hash → 決定哪台 proxy）
 ↓ GUE 封裝，轉發給對應 proxy
Proxy（SSL 終止、健康判斷）
 ↓
Backend Server（實際 Git server）
```

**Director 掛掉怎麼辦**：

Director 本身是 stateless 的（只做 hash，不存狀態），所以可以跑多台，用 ECMP 分流：

```
用戶封包進來
 ↓
路由器（ECMP，均勻分給多台 director）
 ↙      ↓      ↘
Dir-1  Dir-2  Dir-3   ← 全部 stateless

Dir-1 收到封包 → hash → 決定給 Proxy-2
Dir-2 收到封包 → 同樣 hash → 也決定給 Proxy-2

Dir-1 掛掉 → ECMP 自動把流量給 Dir-2、Dir-3
Dir-2 算出一樣的答案 → 無縫接管，連線不斷
```

**整體 HA 設計**：

```
用戶
 ↓
ECMP 路由（多台 director，任一台掛掉自動移除）
 ↓
Director cluster（stateless，掛幾台都沒事）
 ↓ GUE 封裝
Proxy cluster（Rendezvous Hashing，掛掉只影響那台的連線）
 ↓
Backend Server cluster（health check）
```

三層都沒有單點：Director 靠 ECMP + stateless，Proxy 靠 Rendezvous Hashing，Backend 靠 health check。

---

**② GUE（Generic UDP Encapsulation）**

問題：Director 把封包轉給 Proxy，怎麼「標記」要給哪台？

方法一：IP Option（加在 IP header 的額外欄位）
問題：某些路由器看到 IP Option 就把封包丟給 software 處理，不走硬體加速。
吞吐量：hardware path → 數百萬 pps，software path → 數千 pps（差 1000 倍）。

方法二：GUE（用 UDP 包住原始封包）

```
原始封包：
┌─────────────────────────────┐
│ IP Header │ TCP │ Git data  │
└─────────────────────────────┘

Director 加一層 UDP 外殼：
┌──────────────────────────────────────────────────┐
│ 外層 IP │ UDP │ GUE Header │ 原始 IP │ TCP │ Git │
└──────────────────────────────────────────────────┘
  dst = 目標 Proxy IP            原始封包原封不動

路由器只看外層 IP → 走 hardware path → 百萬 pps
Proxy 收到後剝掉外殼，取出原始封包正常處理
```

---

**③ DPDK（Data Plane Development Kit）**

問題：kernel 網路 stack 處理封包有很多 overhead。

傳統 kernel path：

```
網卡收到封包
  → 硬體 interrupt 通知 CPU
  → Kernel 複製封包到 kernel buffer
  → Kernel 做協議處理
  → 系統呼叫複製到 user space（context switch）
  → App 才能讀到

每個封包：2 次 context switch + 2 次記憶體複製
```

DPDK（繞過 kernel）：

```
網卡收到封包
  → DMA 直接寫進 user space 記憶體
  → App 用 polling 主動去讀（不等 interrupt）
  → App 直接處理

零 context switch、零記憶體複製、零 interrupt
```

代價：CPU 一直 polling，即使沒封包也跑 100%。通常專門配一顆 CPU core 給它。適合吞吐量要求極高的場景（LB、防火牆、電信設備）。

**關鍵邏輯**：
連線穩定 > 一切。市面上沒有 stateless 的方案能同時做到這點，才選擇自造。

---

### Q&A：常見疑問

**Q：用戶連線是直接打到 proxy 嗎？**

不是。用戶以為自己在跟 GitHub 公開 IP 直連，實際封包路徑是：

```
用戶 → Director（hash，轉發） → Proxy（TCP state 在這）→ Backend
```

Director 是透明的，用戶感知不到它的存在。

**Q：Director 掛掉，用戶要重連嗎？**

不用。TCP state 存在 proxy，不在 director。Director 掛掉後：

```
ECMP 偵測到 → 把流量改送其他 director
其他 director 算出同樣的 proxy
封包繼續送到同一台 proxy → 連線繼續，用戶無感
```

**Q：Proxy 掛掉，用戶要重連嗎？**

要。TCP state 就存在 proxy 上，proxy 死了 state 消失，連線斷。

GLB 緩解方式：

- **Connection Draining**：先停送新連線，等舊連線自然結束再關
- **同時只能一台 draining**：避免大量連線同時斷
- **新連線不受影響**：health check 偵測到後，Rendezvous Hash 重算自動排除

```
Director 掛掉 → state 不在 director → 連線不斷 → 用戶無感
Proxy 掛掉   → state 在 proxy     → 連線斷   → 用戶要重連
```

**Q：Proxy 是什麼？**

代理人。分兩種方向：

```
Forward Proxy（正向代理） = 幫用戶出去（VPN、翻牆）
Reverse Proxy（反向代理） = 幫 server 擋入（LB、Nginx、Envoy）
```

GLB 裡的 Proxy 是 Reverse Proxy，負責終止 TCP 連線、SSL 終止、轉發給 backend。Director 只做 hash + 轉封包，不碰 TCP 內容。

**Q：Director 只有第一個請求會經過嗎？**

從單一請求角度：對，一個請求過 director 一次，轉給 proxy，結束。

但長連線（Git push、WebSocket）同一條連線會持續傳很多封包，每個封包都過 director。這就是為什麼需要 DPDK——不是每個請求複雜，是同時幾百萬條連線在傳資料，封包量極大，kernel overhead 吃不消。

---

## Slack — HAProxy 換 Envoy

**背景**：Slack 同時在線用戶數千萬，每個用戶保持一條 WebSocket 長連線（訊息、presence、app 整合各自有獨立 endpoint）。

**問題**：
HAProxy 每次更新設定需要 reload，reload 時會起一個新 process 接新連線，舊 process 繼續服務現有連線直到結束。

HTTP 短連線幾秒就結束，沒問題。但 WebSocket 長連線可能幾小時都不斷。結果：

- 舊 process 越堆越多（「殭屍 process」）
- 每個 process 都占記憶體，記憶體使用量持續增長
- 平均每次設定更新要等數小時才能完全清乾淨

Slack 頻繁更新 LB 設定（後端節點上下線、路由規則調整），這個問題每天都在發生。

此外 HAProxy 缺少：

- Zone-aware routing（跨 AZ 流量費用高）
- Passive health check / Outlier Detection
- Panic routing（大規模 backend 掛掉時的 fallback）

**決定**：
換 Envoy。Envoy 用 xDS API 動態更新設定，不需要 reload，不會產生殭屍 process。額外收穫：

- **Zone-aware routing**：優先打到同 AZ 的後端，降低跨 AZ 費用和延遲
- **Outlier Detection**：被動偵測回應異常的節點並自動踢掉
- **Panic routing**：健康節點比例太低時，暫時忽略 health check 把流量分散出去，避免流量全堆在少數節點上
- **Hot restart**：升級 Envoy 本身時，新舊 process 交接，連線不中斷，stats 一起轉移
- **標準化**：Envoy 本來就是 service mesh data plane，全公司統一，不用同時維護兩套 config 語言

**遷移架構**：
雙軌並行 + DNS 漸進切流：

```
用戶
  ↓
NS1 DNS（weighted routing）
  ↙                    ↘
HAProxy（舊）         Envoy cluster（新）
  ↓                         ↓
      WebSocket Service 後端
```

流量分階段調整，每個 region 獨立操作：
`10% → 25% → 50% → 75% → 100%`

Config 管理：用 Chef 生成 Envoy YAML（不手寫 template），部署前用 `envoy --mode validate` 驗證，Admin API（`/clusters`、`/listeners`）提供即時 debug 能見度。

**踩到的坑**：

- HAProxy 有沒文件的隱藏行為（timeout、header 細節），下游 service 已依賴它，要一個一個找出來並在 Envoy 復刻
- 一開始沒有自動化測試，靠手動驗，中間不小心搞壞了 **DAU 指標**（Daily Active User），後來才補 test suite
- SNI listener 重構到一半留下技術債

**代價**：

- 雙軌並行跑了 6 個月，費用暫時翻倍

**結果**：

- 6 個月完成，zero customer impact
- 完成後峰值流量超過歷史最高，沒出問題
- 後續順手把 client metrics ingestion pipeline 和 internal LB 也一起遷過去

**關鍵邏輯**：
長連線場景，「不重啟能更新設定」是硬需求，HAProxy 架構上做不到。寧願花 6 個月做乾淨的遷移，不打算繞路 hack HAProxy。

---

## Shopify — Nginx + Lua（OpenResty）

**背景**：Shopify 托管數十萬個電商，其中不乏 Kylie Cosmetics 這類有龐大粉絲的品牌。

**問題**：
Flash sale 時，大量用戶同時湧入，瞬間流量遠超正常值。Shopify 的 DB 是 sharded 的，同一個 shard 上有多個商家。Kylie Cosmetics 的 flash sale 打掛了她的 shard，連帶把同 shard 的其他商家也打掛——這些商家跟 Kylie 完全無關，卻受牽連。

問題的本質：流量峰值直接打進 DB，沒有任何緩衝。

**嘗試過的方案**：
改 Rails app 加 rate limiting。問題：Shopify 是多租戶平台，Rails app 極度複雜，「開心臟手術」要幾個月，而 flash sale 每週都在發生。

**決定**：
在 Nginx 層加 Lua 腳本（OpenResty），實作 Leaky Bucket 限流。流量超過閾值就排隊或回傳 503，不讓峰值打穿到 DB。

**Lua 是什麼**：
輕量腳本語言，特點是可以嵌進其他程式裡執行（遊戲邏輯、Nginx 自訂邏輯都常用）。Nginx 本身不能寫判斷邏輯，OpenResty 讓你在 Nginx 裡嵌 Lua，腳本跑在 Nginx process 內，幾乎零效能損耗。

**完整請求流程**：

```
用戶點開 Kylie Cosmetics 商店
        ↓
DNS 解析 → 回傳 Shopify LB 的 IP
        ↓
Load Balancer（AWS ELB）
  決定送給哪台 Nginx server
        ↓
Nginx（OpenResty）
  Lua 腳本執行：
    1. 這個請求是哪個商家的？（看 domain）
    2. 這個商家的流量桶還有空間嗎？

    有空間 → 放行
    沒空間 → 直接回 503（排隊頁面），流程結束
        ↓
Rails app（Puma workers）
  處理業務邏輯、商品庫存、購物車
        ↓
DB 路由層
  判斷這個商家在哪個 shard
  Kylie Cosmetics → Shard 1
        ↓
Shard 1 的 MySQL
  查詢、寫入訂單
        ↓
結果逆向回傳給用戶
```

**Leaky Bucket 在哪一層擋**：

```
LB → Nginx（在這裡擋）→ Rails → DB

Nginx 收到 100,000 req/s
Leaky Bucket 設定上限 1,000 req/s
→ 1,000 req 放行進 Rails
→ 99,000 req 直接回 503，Rails 跟 DB 完全看不到
```

Leaky Bucket 概念：

```
請求不斷進來（倒水進桶）
        ↓
    請求排隊  ← 桶有容量上限，超過就溢出（503）

        │ 固定速率流出
        ↓
    Rails app 處理
```

桶沒滿 → 請求排隊等處理。桶滿了 → 新請求直接 503。流出速率固定，DB 每秒收到的請求量永遠可控。

為什麼選 Lua：

- Lua 腳本跑在 Nginx process 內，效能幾乎沒有損耗
- 不用動 Rails app
- 兩週就能上線
- 峰值最高撐到 3400 萬 req/min

**關鍵邏輯**：
速度 > 架構完美。架構理想解是隔離 shard、加 queue，但要幾個月。Lua 兩週止血，先讓商家不受牽連，之後再做根本解。

---

## Dropbox — 自建動態 LB（Robinhood）

**背景**：Dropbox 2015-2016 年把 90% 資料從 AWS 搬回自建機房，一年省下幾千萬美元。代價是要自己管硬體，機房裡新舊機器混跑。

**為什麼不用雲端 LB 解決**：

- 搬回自建機房就是為了省錢，換回雲端等於推翻這個決定
- AWS ALB 用的也是 round-robin / least connections，不知道每台機器的實際 CPU 負載，換了也是一樣的問題

**問題**：
Round-robin 假設每台機器能力相同，但現實是：

```
同樣數量的請求：
新機器 CPU 10%  ← 還很閒
老機器 CPU 60%  ← 快撐不住
平均 CPU 35%
```

工程師看監控：「平均才 35%，資源夠用。」→ 不加機器。
實際上老機器已是瓶頸，流量一高就炸。

**第一個嘗試：Bandaid**

Server 在 HTTP response header 帶上自己的 CPU 使用率，LB 讀取後調整分流權重：

```
老機器回應 → Header: X-CPU-Load: 0.6
新機器回應 → Header: X-CPU-Load: 0.1
    ↓
LB：新機器權重高 → 多分流量
    老機器權重低 → 少分流量
```

問題一：時間差

```
T=0  老機器 CPU 60%，帶在 response header 回傳
T=1  LB 讀到，調整權重
T=2  老機器 CPU 已經 90%，LB 還在用 T=0 的資料
```

問題二：閒置 server 沒有 response，就沒有 header，LB 對它的認知永遠停在舊資料。

用 inverted sigmoid curve 讓舊資料慢慢衰減來緩解，但根本問題沒解。

**最終決定：Robinhood**

自建動態 LB，每台 server 主動推送即時 CPU 負載給 LB，用 PID controller 持續調整權重。

**CPU 負載怎麼讀取**

不再等 response header（被動），改成 server 主動定期回報：

```
每台 server 定期（每秒）把 CPU 使用率推給 Robinhood LB
    ↓
Robinhood 有每台 server 的即時負載數字
    ↓
不管有沒有流量進來，資料都是最新的
```

**PID Controller 怎麼計算權重**

PID（Proportional-Integral-Derivative）是控制系統的標準算法，概念是「自動油門」：

```
目標：每台機器 CPU 維持在 50%

老機器 CPU 70%（超出目標 20%）
    → PID：誤差大，大幅降低這台的權重

老機器 CPU 55%（超出目標 5%）
    → PID：快到了，小幅調整

老機器 CPU 50%
    → PID：達標，維持現狀
```

不會調過頭，也不會反應太慢，持續微調到穩定。

**完整流程**：

```
請求進來
        ↓
Robinhood LB
  讀取每台 server 即時 CPU 負載
  PID controller 計算權重：
    新機器 CPU 10% → 權重高
    老機器 CPU 60% → 權重低
  根據權重選一台
        ↓
新機器收到較多流量，老機器收到較少
        ↓
兩台 CPU 趨向均等
```

**結果**：最大服務的 fleet 縮小 25%——同樣流量用更少機器扛，直接省錢。

**關鍵邏輯**：


| 方案          | 問題                              |
| ----------- | ------------------------------- |
| Round-robin | 假設機器能力相同，異質硬體完全不適用              |
| Bandaid     | 被動讀 header，有時間差，閒置 server 資訊不更新 |
| Robinhood   | server 主動推即時負載 + PID 持續調整，根本解   |


市面上沒有現成工具能處理「異質硬體 + 即時動態負載感知」，只能自造。

---

## Twitter — Deterministic Aperture

**背景**：Twitter 有數千台後端 server，每個 service 都需要跟後端建連線才能分流。Twitter 用的是 **client-side LB**，LB 邏輯寫在每個 service 的 Finagle client 裡（自建的 RPC framework，開源），不是獨立的 LB proxy。

**Service 之間為什麼要互相呼叫**

微服務把一個大系統拆成很多小 service，每個只負責一件事。原本 monolith 裡直接呼叫 function，現在要走網路。

以用戶發一則推文為例：

```
Tweet Service（處理推文）
    ↓ 這則推文是誰發的？
User Service（拿用戶資料）
    ↓ 要推給哪些人？
Follower Service（拿追蹤者清單）
    ↓ 寫進每個追蹤者的時間軸
Timeline Service（更新時間軸）
    ↓ 通知追蹤者
Notification Service（發推播）
    ↓ 存起來
Storage Service（寫入 DB）
```

用戶按一個「發推文」，背後可能觸發幾十個 service 互相呼叫。兩種主要用途：
- **拿資料**：「給我 user_id=123 的資料」→ 回傳結果
- **觸發動作**：「幫我通知這些人」→ 不一定回傳資料

好處：每個 service 獨立部署、獨立擴容。代價：原本 function call，現在走網路，連線數爆炸。

**為什麼用 client-side LB 而不是 proxy LB**

Twitter 的核心問題是連線數爆炸。如果用獨立 LB proxy，等於把所有連線集中在 proxy 那層，問題搬位置但沒解決。Client-side LB 讓每個 service 自己管連線，才能真的減少連線數。

**原本的設計：P2C（Power of Two Choices）**

每次請求進來，隨機選兩台 server，選負載較低那台：

```
請求進來
    ↓
Finagle client 隨機選 server-42 和 server-187
server-42 負載 60%，server-187 負載 30%
    ↓
選 server-187，送過去
```

要做到「隨機選任意兩台」，前提是每個 client 跟**所有** server 都有連線：

```
1000 個 client × 1000 台 server = 1,000,000 條連線

每條連線：
  占記憶體
  定期 health check
  Java GC 要掃 heap 裡的連線物件
```

問題一：連線數爆炸，資源消耗極大。

問題二：大規模下隨機不等於均勻：

```
1000 個 client 各自獨立做隨機選擇
統計誤差積累
→ server-42 被選 60 次，server-99 被選 10 次
→ 分布嚴重不均，熱點自然形成
```

**為什麼不用獨立 LB proxy 取代 P2C**

proxy LB（HAProxy、Envoy）解決的是流量入口的分流，不解決 service 之間互相呼叫的連線數問題。Twitter 內部服務互相呼叫的量遠大於外部流量，用 proxy 擋不住這個問題。

**解法：Deterministic Aperture**

把 client 和 server 都排在虛擬環形上，每個 client 只負責環形上正對面那段的 server（aperture = 開口）：

```
server 環：  S1  S2  S3  S4  S5  S6  S7  S8
client 環：       C1          C2          C3

C1 的 aperture = S2、S3
C2 的 aperture = S5、S6
C3 的 aperture = S8、S1
```

每個 client 只跟自己 aperture 內的 server 建連線，不連其他 server。

**Deterministic（確定性）的意思**

不靠隨機，用算法直接算出「我負責哪幾台」：

```
輸入：client 數量、server 數量、自己在環上的位置
輸出：我的 aperture 是哪幾台

任何 client 用同樣公式都能算出正確結果
不需要集中協調，不需要外部服務告知
```

**Client 數量變動時自動調整**

```
原本 3 個 client，每個負責 1/3 的 server
新增第 4 個 client
    ↓
環形重新計算，每個 client 負責 1/4 的 server
所有 server 依然被完整覆蓋，沒有死角
```

**完整請求流程（從用戶到 DB）**

```
瀏覽器輸入 twitter.com
        ↓
DNS 解析 → 回傳 Twitter VIP（虛擬 IP，背後對應多台 Edge LB）
        ↓
路由器 ECMP
  根據 src IP + dst IP hash，決定送給哪台 Edge LB
  Edge LB 台1
  Edge LB 台2  ← 任一台掛掉，ECMP 自動重分
  Edge LB 台3
        ↓
Edge LB（stateless）
  決定送給哪台 Tweet Service
  Tweet Service 台1
  Tweet Service 台2  ← 多台，分散請求
  Tweet Service 台3
        ↓
Tweet Service 台1（處理推文邏輯）
  需要查用戶資料，呼叫 User Service
        ↓
Finagle client（內建在 Tweet Service 程式碼裡）
  計算自己的 aperture（確定性算出負責哪幾台 User Service）
  在 aperture 內選負載最低那台
  直接連過去，不經過獨立 LB proxy
        ↓
User Service server 2（aperture 內負載最低的那台）
        ↓
DB（primary / replica，查詢用戶資料）
        ↓
結果逆向回傳給瀏覽器
```

每一層解決不同問題：


| 層                      | 解決什麼                |
| ---------------------- | ------------------- |
| DNS + ECMP             | Edge LB 不能只有一台，流量分散 |
| Edge LB                | 用戶請求分給哪台後端 service  |
| Finagle client-side LB | service 之間呼叫，連線數不爆炸 |
| DB replication         | DB 不能只有一台，掛掉有備援     |


**為什麼這樣比 P2C 好**


|      | P2C                | Deterministic Aperture |
| ---- | ------------------ | ---------------------- |
| 連線數  | client × 所有 server | client × aperture 大小   |
| 分布   | 隨機，大規模有誤差          | 確定性，保證均勻覆蓋             |
| 協調需求 | 不需要                | 不需要（公式算出來）             |
| 規模擴展 | 連線數線性爆炸            | aperture 自動調整          |


**結果**


| 指標          | 變化                         |
| ----------- | -------------------------- |
| 連線數         | -91%                       |
| CPU         | -25%                       |
| GC overhead | -50%（連線物件少了，Java heap 小很多） |


**關鍵邏輯**：
規模夠大之後，隨機不等於均勻，連線數也會爆炸。確定性算法 + client-side LB 才能同時解決這兩個問題，proxy LB 解不了 service 間內部呼叫的連線問題。

**現在的解法：Service Mesh**

Twitter 當時沒有現成工具才自造 Finagle，現在新公司直接用 Service Mesh：

Istio + Envoy sidecar：每個 service 旁邊跑一個 Envoy，service 間呼叫全部過 sidecar，LB、retry、circuit breaker 統一處理，不用改 service 本身程式碼。

gRPC client-side LB：Google 開源的 RPC framework，內建 client-side LB，直接整合進程式碼。


|      | Twitter Finagle | Istio / gRPC |
| ---- | --------------- | ------------ |
| 時間   | 2011 年自造        | 2017 年後成熟    |
| 語言   | Scala / Java    | 任何語言         |
| 維護   | 自己維護            | 社群維護         |
| 採用門檻 | 高（要改程式碼）        | 低（Istio 不用改） |


**Service Mesh 分流原理**

```
Service A 要呼叫 Service B
        ↓
請求先到 Service A 旁邊的 Envoy sidecar
        ↓
Sidecar 從 Control Plane 拿到 Service B 所有 instance 清單
用 LB 算法選一台（least request、round-robin 等）
        ↓
轉發到 Service B 的 Envoy sidecar
        ↓
Service B sidecar 轉給 Service B 本體
```

本質還是 client-side LB，只是把 LB 邏輯從程式碼移到 sidecar，service 本身完全不知道有幾台 instance。

---

## Lyft — 發明 Envoy

**背景**：Lyft 2015 年快速從 monolith 切換到微服務，用多種語言（Python、Go、Java）寫不同服務。

**問題**：

service 之間互相呼叫，每種語言都要自己處理網路層：

```
Python service：自己寫 retry、timeout、circuit breaker
Go service：自己寫一套
Java service：再寫一套
```

三個問題：
- 行為不一致（每套實作細節不同）
- 出問題不知道是誰的錯（Python bug？Go bug？還是網路？沒有統一觀測點）
- 新語言加進來就要再寫一套，維護成本 O(語言數量)

**為什麼不同語言能協同**

Envoy 處理的是**網路流量**，不是程式碼。HTTP、gRPC 這些網路協定本來就跟語言無關：

```
Python → HTTP request → Envoy 攔截
Go     → HTTP request → Envoy 攔截  ← Envoy 看到的是一樣的東西
Java   → HTTP request → Envoy 攔截
```

Envoy 不需要知道是哪個語言發的請求，只看網路流量，所以語言無關。

**Envoy 是什麼語言寫的**

C++。原因：效能極高、記憶體控制精確、不依賴特定 runtime（不像 Java 需要 JVM），可以跑在任何環境。每個 service 旁邊都要跑一個 sidecar，sidecar 本身慢的話整個系統都慢，所以效能是第一需求。

**解法：Sidecar Proxy（Envoy）**

把網路層邏輯從程式碼抽出來，放進 sidecar：

```
之前：
Python 程式碼（含 retry、timeout、circuit breaker）
Go 程式碼（含 retry、timeout、circuit breaker）
Java 程式碼（含 retry、timeout、circuit breaker）

之後：
Python 程式碼 → Envoy sidecar（統一處理）
Go 程式碼    → Envoy sidecar（同一套）
Java 程式碼  → Envoy sidecar（同一套）
```

**完整請求流程**

```
Service A（Python）要呼叫 Service B（Go）
        ↓
Service A 程式碼發出 HTTP 請求
        ↓
被 Envoy sidecar 攔截（Service A 旁邊）
  - 查 Control Plane 拿到 Service B 所有 instance 清單
  - LB 算法選一台
  - 處理 retry 設定、timeout 設定
        ↓
發出去，到 Service B 的 Envoy sidecar
  - 記錄 metrics（延遲、錯誤率）
  - tracing（這個請求從哪來）
        ↓
轉給 Service B 程式碼（Go）處理
        ↓
回傳，逆向經過兩個 sidecar 回到 Service A
```

Service A 和 Service B 的程式碼完全不知道 LB、retry、circuit breaker 的存在，都在 sidecar 處理。

**出問題怎麼 debug**

```
Service A 呼叫 Service B 失敗
        ↓
Envoy metrics 直接顯示：
  - 是哪段連線失敗（A sidecar → B sidecar）
  - 失敗率、延遲分佈
  - 是 timeout 還是 connection refused
不需要猜是哪個語言的 bug
```

**結果**

後來 Google 看到 Envoy，跟 Lyft 一起把這套模式標準化成 **Istio**（Service Mesh），成為 K8s 微服務的業界標準。現在幾乎所有大規模微服務系統都在用：Envoy 當 data plane，Istio 當 control plane。

**關鍵邏輯**：

Polyglot 環境下，library 方案維護成本 O(語言數量)。Sidecar 讓網路層跟 app 完全解耦，維護成本降到 O(1)。不管幾種語言，只維護一個 Envoy。

---

## 決策模式總結


| 公司      | 核心問題                | 決策邏輯                                       |
| ------- | ------------------- | ------------------------------------------ |
| GitHub  | Git 連線不能中斷          | 連線穩定 > 一切，市面無解，自造 stateless LB             |
| Slack   | 長連線 + 頻繁更新設定        | HAProxy reload 架構限制，換支援動態設定的 Envoy         |
| Shopify | 每週 flash sale 打掛 DB | 速度優先，2 週 Nginx Lua 止血，不等幾個月重構              |
| Dropbox | 異質硬體導致分布不均          | Bandaid 不夠好，自造 PID controller 動態 LB        |
| Twitter | 超大規模隨機分布誤差          | 確定性算法取代隨機，連線數 -91%                         |
| Lyft    | Polyglot 微服務網路層     | Library 維護成本 O(n)，Sidecar 降到 O(1)，發明 Envoy |


---

## 進階細節

### HAProxy vs Envoy 架構

**HAProxy 內部架構**

```
Frontend（接流量）
  - 監聽 port
  - ACL 規則（根據 header、path 路由）
       ↓
Backend（轉流量）
  - Server 清單（寫在 config file）
  - LB 算法（round-robin 等）
  - Health check

設定來源：config file（靜態）
更新方式：改檔案 → reload → 新 process
```

**Envoy 內部架構**

```
Listener（監聽 port）
       ↓
Filter Chain（可插拔的處理鏈）
  - HTTP filter（retry、rate limit 等）
  - TLS filter（SSL 終止）
       ↓
Router → Cluster（後端群組）
  - Endpoint 清單（動態）
  - LB 算法
  - Outlier Detection

Admin API（/clusters、/listeners）

         ↑
         │ xDS API（動態推設定）
         │
   Control Plane（獨立服務，e.g. Istio）
```

**最大的架構差異**

HAProxy：Data plane 跟 config 綁在一起，沒有 control plane 概念。

Envoy：Data plane（Envoy）跟 Control plane（推設定的服務）分離。

```
HAProxy：
  [config file] → [HAProxy process]
  設定變了 → 整個 process 要重來

Envoy：
  [Control Plane] ──xDS──→ [Envoy process]
  設定變了 → Control Plane 推過來 → Envoy 熱更新
```

這個 data / control plane 分離，就是 Envoy 能動態更新的根本原因。

**Control Plane 是什麼**

負責追蹤所有節點狀態，把最新設定推給 Envoy：

```
節點開起來
    ↓
向 Control Plane 註冊（「我是 192.168.1.4，我活著」）
    ↓
Control Plane 更新 Endpoint 清單
    ↓
透過 EDS 推給所有 Envo現在的解法：Service Meshy
    ↓
Envoy 更新，開始把流量分給這台
```

常見實作：Istio（K8s service mesh）、Consul、自己實作 xDS API。

**為什麼分離是好事**

```
Control Plane：記錄（哪些節點活著、設定是什麼）
Data Plane（Envoy）：分流（實際處理流量）
```

- Control Plane 掛掉 → Envoy 手上還有最後一份設定，流量繼續跑，不斷
- 設定要更新 → Control Plane 推過來，Envoy 不需要重啟

HAProxy 把這兩件事混在同一個 process，改設定就要重啟整個東西，這是根本限制。

**記憶體 vs Config File 的差異**

- Config file = 抽屜裡的說明書，程式啟動時讀一次，要更新就要停下來重新讀、重新啟動
- 記憶體 = 桌上攤開的筆記本，隨時可以直接改內容，不用停下手邊的事

```
HAProxy：
啟動時讀 config file → 照著跑
config file 改了 → 停掉、重新讀、重新啟動
→ process 停了，連線斷掉

Envoy：
啟動時讀一次設定存進記憶體 → 照著跑
Control Plane 推新設定來 → 直接改記憶體
→ process 沒停，連線繼續跑
```

連線是存在 process 裡的，process 一停連線就消失。改記憶體不需要停 process，所以現有連線不受影響。

**Control Plane 推送機制**

Envoy 啟動時向 Control Plane 建立 gRPC 長連線（訂閱 xDS 更新）。有變動時 Control Plane 主動推過來，不是 Envoy 定期去問。

定期輪詢的問題：有延遲、沒變動也一直發請求浪費資源。長連線 + 主動推送：變動發生後幾乎立即同步。

---

### Connection Draining

移除 server 時，讓現有 request 跑完再關。

```
LB 標記這台 server 為 draining
→ 不送新 request 給它
→ 現有 request 繼續跑完
→ 超過 timeout 才強制斷

一般 API：30 秒
WebSocket：300 秒以上（連線可能幾小時）
```

關鍵：app 本身也要能優雅處理 SIGTERM。流程是：

```
LB 停送新 request
→ 同時送 SIGTERM 給 app
→ App 停止接新 request，跑完現有的，然後關閉
→ LB 偵測到 health check 失敗，確認下線完成
```

如果 app 收到 SIGTERM 直接 exit，LB 的 draining 等待就是白等。

### Slow Start

新節點加進 pool 時，不立刻收滿流量，逐步增加：

```
slow_start 30s：
  第 1 秒：3% 流量
  第 15 秒：50% 流量
  第 30 秒：100% 流量
```

解決的問題：

- JVM 服務需要 JIT 編譯 warm up，冷啟動時處理速度慢
- 連線池（DB、Redis）需要時間建立
- 本地 cache 是空的，第一批 request 都要打 DB

AWS ALB、Envoy 內建支援，Nginx 要手動調整 weight。

### Health Check 設計原則

**不好的設計**：

```
GET /          → 回 200 就算活著
```

HTTP server 還在，但 DB 連線斷了，這個 server 其實不能服務。

**好的設計**：

```javascript
// /health endpoint
app.get('/health', async (req, res) => {
  try {
    await db.query('SELECT 1')      // 確認 DB 連線
    await redis.ping()              // 確認 Redis 連線
    res.status(200).json({ ok: true })
  } catch (e) {
    res.status(503).json({ error: e.message })
  }
})
```

**參數調整**：

```
HealthCheckInterval: 10s       # 每 10 秒打一次
UnhealthyThresholdCount: 3     # 連失敗 3 次才踢出（避免一次網路抖動就踢掉）
HealthyThresholdCount: 2       # 連成功 2 次才放回來（避免剛恢復就被大量流量打掛）
Timeout: 5s                    # 超過 5 秒沒回應算失敗
```

