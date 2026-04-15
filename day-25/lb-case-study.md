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

**背景**：Slack 同時在線用戶數千萬，每個用戶保持一條 WebSocket 長連線。

**問題**：
HAProxy 每次更新設定需要 reload，reload 時會起一個新 process 接新連線，舊 process 繼續服務現有連線直到結束。

HTTP 短連線幾秒就結束，沒問題。但 WebSocket 長連線可能幾小時都不斷。結果：
- 舊 process 越堆越多（「殭屍 process」）
- 每個 process 都占記憶體，記憶體使用量持續增長
- 平均每次設定更新要等數小時才能完全清乾淨

Slack 頻繁更新 LB 設定（後端節點上下線、路由規則調整），這個問題每天都在發生。

**決定**：
換 Envoy。Envoy 用 xDS API 動態更新設定，不需要 reload，不會產生殭屍 process。額外收穫：
- Zone-aware routing 內建（優先把流量打到同 AZ 的後端，降低跨 AZ 費用和延遲）
- Outlier Detection 自動偵測並踢掉回應異常的節點（被動 health check）
- Panic routing：當健康節點比例太低時，暫時忽略 health check 把流量分散出去，避免流量全堆在少數健康節點上

**代價**：
- 兩套系統並行跑了 6 個月，AWS 費用暫時翻倍
- HAProxy 有些沒文件的隱藏行為，下游 service 依賴它，要一個一個找出來並在 Envoy 復刻
- 用 DNS-based 流量切換（逐步把 DNS 指向 Envoy），確保零用戶感知

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

為什麼選 Lua：
- Nginx 本身處理流量，Lua 腳本在 Nginx 內執行，效能幾乎沒有損耗
- 不用動 Rails app
- 小團隊的 production engineer 兩週就能上線
- 峰值最高撐到 3400 萬 req/min

**關鍵邏輯**：
速度 > 架構完美。架構理想解是隔離 shard、加 queue，但要幾個月。Lua 兩週止血，先讓商家不受牽連，之後再做根本解。

---

## Dropbox — 自建動態 LB（Robinhood）

**背景**：Dropbox 有大量 stateless 微服務，跑在不同規格的機器上。

**問題**：
Round-robin 分流假設每台機器處理能力相同，但現實是機器規格不一（老機器 / 新機器混跑），同樣數量的 request 在不同機器上消耗的 CPU 差很多。結果：新機器 CPU 10%，老機器 CPU 60%，整體平均低但老機器快撐不住。

工程師看到 CPU 平均低，不敢加機器（「資源還夠啊」），實際上老機器已經是瓶頸，稍微流量高峰就炸。

**嘗試過的方案（Bandaid）**：
被動讀取 server 負載。做法：server 在 HTTP response header 帶上自己的 CPU 使用率，LB 讀取後調整分流權重，讓較閒的 server 接更多流量。

效果有，但有兩個問題：
1. **時間差**：負載資訊是從上一個 response 來的，有延遲，高峰期資訊太舊
2. **只有 response 時才更新**：閒置的 server 沒有 response，LB 對它的認知一直停在舊資料

用 inverted sigmoid curve 讓舊資料慢慢衰減來緩解，但根本問題沒解。

**最終決定（Robinhood）**：
自建動態 LB service，用 PID controller 根據即時負載持續調整每台 server 的權重。PID controller 是控制系統的標準算法（想成「油門自動調節」），能對誤差快速反應又不過度震盪。

**結果**：最大服務的 fleet 縮小 25%——同樣流量用更少機器扛，直接省錢。

**關鍵邏輯**：
市面上沒有現成解法能處理「異質硬體 + 動態負載感知」的問題。Bandaid 已是業界少見的做法，還不夠好，只能自造。

---

## Twitter — Deterministic Aperture

**背景**：Twitter 有數千台後端 server，client（proxy）需要決定連哪些 server。

**問題**：
隨機 P2C（Power of Two Choices）：每次隨機選兩台 server，選負載較低那台。理論上分布應該均勻，但實際上：
- 小規模時隨機誤差不大
- 大規模時（數千台），隨機選擇的統計誤差積累，造成一台收 60% 流量、另一台收 10%
- 更大的問題：每個 client 跟所有 server 都建連線，數千個 client × 數千台 server = 連線數爆炸

**決定**：
Deterministic Aperture。把 client 和 server 都排在環形上，每個 client 只連環形上自己「正對面」那一段的 server（aperture = 開口）。用確定性算法決定每個 client 的 aperture，不再隨機。

效果：
- 每個 client 只連一部分 server，連線數大幅下降
- 分布是確定的，不受隨機誤差影響
- Client 數量增加時，aperture 自動調整，保持均勻覆蓋

**結果**：連線數 -91%、CPU -25%、GC overhead -50%（Java GC 要掃 heap，連線物件少了，GC 壓力直接降）。

**關鍵邏輯**：
規模夠大之後，「隨機」不再等於「均勻」。確定性算法才能在超大規模下保證分布。

---

## Lyft — 發明 Envoy

**背景**：Lyft 2015 年快速從 monolith 切換到微服務，用多種語言（Python、Go、Java）寫不同服務。

**問題**：
服務間通訊靠 library 處理（每種語言各一套）。問題：
- Retry、timeout、circuit breaker 每種語言要各自實作，行為不一致
- 出問題時不知道是哪一層：是 Python service 的 bug、Go service 的 bug，還是網路問題？
- 新語言加入就要再寫一套 library

**決定**：
做 sidecar proxy（Envoy）。每個 service 旁邊跑一個 Envoy process，所有 outbound / inbound 流量都過 Envoy：
- 重試、timeout、circuit breaker 統一在 Envoy 處理，語言無關
- 所有流量有統一的 metrics 和 tracing，出問題馬上知道是哪段
- xDS API 動態更新路由規則，不需要重啟

後來 Google 看到 Envoy，一起把這套模式標準化成 **Istio**（Service Mesh），成為 K8s 微服務的業界標準。

**關鍵邏輯**：
Polyglot 環境下，library 方案的維護成本是 O(語言數量)。Sidecar 讓網路層跟 app 完全解耦，維護成本降到 O(1)。

---

## 決策模式總結

| 公司 | 核心問題 | 決策邏輯 |
|------|---------|---------|
| GitHub | Git 連線不能中斷 | 連線穩定 > 一切，市面無解，自造 stateless LB |
| Slack | 長連線 + 頻繁更新設定 | HAProxy reload 架構限制，換支援動態設定的 Envoy |
| Shopify | 每週 flash sale 打掛 DB | 速度優先，2 週 Nginx Lua 止血，不等幾個月重構 |
| Dropbox | 異質硬體導致分布不均 | Bandaid 不夠好，自造 PID controller 動態 LB |
| Twitter | 超大規模隨機分布誤差 | 確定性算法取代隨機，連線數 -91% |
| Lyft | Polyglot 微服務網路層 | Library 維護成本 O(n)，Sidecar 降到 O(1)，發明 Envoy |

---

## 進階細節

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
