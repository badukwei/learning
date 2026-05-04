# System Design - 現代 DNS 智慧路由與解法

Date: 2026-04-02

---

## 傳統 DNS 的問題回顧

傳統 DNS Round Robin 有兩個根本缺陷：
1. **快取（TTL）**：Server 掛了，用戶在快取過期前還是連失敗的機器
2. **盲目分配**：不知道 Server 負載，機械輪流送流量

現代 DNS 服務把這兩個問題都解掉了。

---

## 現代 DNS 的四大智慧能力

### 1. Health Check — DNS 層的存活偵測

Cloudflare、Route 53、NS1 都支援直接在 DNS 層對每個 IP 做健康檢查。

**運作機制：**

```
Route 53 每 10 秒探測一次：

Route 53 ──► GET /health ──► Server A (IP: 1.2.3.4)   → 200 OK  ✅
Route 53 ──► GET /health ──► Server B (IP: 5.6.7.8)   → timeout ❌
Route 53 ──► GET /health ──► Server C (IP: 9.10.11.12) → 200 OK  ✅

DNS 記錄自動變成：
yourapp.com → [1.2.3.4, 9.10.11.12]   ← Server B 被移除

用戶查詢 → 只拿到健康機器的 IP
```

**Health Check 可以檢查什麼：**
- HTTP/HTTPS endpoint（`/health`、`/ping`）
- TCP port 是否開著
- 回應內容是否包含特定字串（e.g. `"status":"ok"`）
- CloudWatch Alarm 狀態（AWS 特有，可以整合 CPU、Memory 指標）

**TTL 短化：**

健康檢查配合短 TTL 才有效。

```
傳統設定：TTL = 86400（24小時）
          Server 掛了 → 24 小時內用戶都打錯機器

現代設定：TTL = 60（60秒）
          Server 掛了 → 最多 60 秒後全球快取刷新

代價：DNS 查詢量增加（用戶更頻繁重新查）
     但 Route 53 / Cloudflare 的基礎設施撐得住
```

---

### 2. Latency-Based Routing — 實測延遲選最快節點

不靠 IP 地理資料庫猜測，而是 AWS/Cloudflare 自己長期收集各地到各節點的實際延遲資料。

**Route 53 Latency-Based Routing 流程：**

```
User 在台灣查詢 api.yourapp.com

Route 53 查內部延遲資料庫：
  ap-northeast-1（東京）→ 35ms  ✅ 最低
  us-east-1（維吉尼亞） → 185ms
  eu-west-1（愛爾蘭）   → 265ms

Route 53 回傳：東京節點的 IP

User → 連東京 → 體感速度最快
```

**和 GeoDNS 的差異：**

```
GeoDNS：  「你的 IP 登記在台灣 → 給你亞洲節點」
           問題：IP 地理資料庫有誤差，VPN 用戶會被送錯

Latency：  「根據我們測量的資料，台灣到東京最快」
           更準確，不依賴 IP 位置
```

**Cloudflare 的做法（Traffic Steering）：**

Cloudflare Load Balancer 有多種 Steering Policy：
- `Dynamic Steering`：用 RUM（Real User Monitoring）資料，根據用戶真實的連線速度選 origin
- `Geo Steering`：指定某個國家/地區的流量送哪個 origin pool
- `Random Steering`：加權隨機（類似 Weighted Round Robin）
- `Proximity Steering`：純粹按物理距離

---

### 3. Weighted Routing — 流量比例控制

用於 Canary Deploy、A/B Testing、逐步遷移。

**Canary Deploy 實戰流程：**

```
Stage 1：新版本上線，先給 5% 流量觀察
  v1-server（舊）→ 權重 95
  v2-server（新）→ 權重 5

Stage 2：新版本穩定，擴大到 50%
  v1-server → 權重 50
  v2-server → 權重 50

Stage 3：確認沒問題，全切到新版
  v1-server → 權重 0  （或直接刪掉這條記錄）
  v2-server → 權重 100
```

**整個過程不需要重啟 Server，不需要停機，純 DNS 設定調整。**

如果新版本出問題，把權重改回去，60 秒後（TTL）生效。

---

### 4. Failover Routing — 主備自動切換

設定 Primary 跟 Secondary，Health Check 一旦偵測到 Primary 掛掉，自動切換。

**詳細流程：**

```
正常狀態：
用戶 ──► DNS ──► Primary IP (Active)
                 Secondary IP (Standby，不對外)

Primary 出問題：
T+0s   Health Check 偵測到 Primary → timeout
T+30s  連續 3 次失敗（可設定閾值）→ 標記 Unhealthy
T+30s  DNS 自動把 A record 指向 Secondary IP
T+60s  TTL 到期 → 全球 DNS 快取刷新 → 用戶開始連 Secondary

Primary 恢復後：
Health Check 重新偵測到 200 OK
等連續 3 次成功（可設定）→ 標記 Healthy
DNS 自動切回 Primary
```

**Active-Active vs Active-Passive：**

```
Active-Passive（主備）：
  Primary 正常運作，Secondary 閒置待命
  優點：Secondary 完全乾淨，不會有狀態同步問題
  缺點：Secondary 的資源平時浪費了

Active-Active（雙主）：
  兩台都對外服務，都收流量
  一台掛了 → DNS 把流量全給另一台
  優點：資源利用率高
  缺點：需要確保兩台的資料是同步的（DB replication 等）
```

---

## Anycast：比 GeoDNS 更底層的解法

### GeoDNS 的問題

```
GeoDNS 靠「IP 地理資料庫」判斷用戶在哪：

用戶 IP: 1.2.3.4
DNS 查資料庫：「1.2.3.4 登記在台灣」
DNS 回傳：亞洲節點 IP

問題：
- 企業 VPN → IP 登記在美國，但人在台灣 → 被送到美國節點
- 代理伺服器 → 同理
- IP 資料庫本來就有誤差（更新不即時）
```

### Anycast 的運作原理

Anycast 根本不靠 IP 地理資料庫，而是讓網路基礎設施自己決定。

```
傳統 Unicast（一對一）：
  1.2.3.4 只有一台機器

Anycast（一對最近）：
  1.2.3.4 這個 IP 同時在全球 300 個節點宣告（BGP Announce）
  路由器自動把封包送到「BGP 路由距離最短」的那個節點
```

**BGP（Border Gateway Protocol）是什麼：**

BGP 是網際網路的「路由地圖更新協定」，全球 ISP、資料中心之間用它來互相告訴對方「我能到哪些 IP，要走幾跳」。

```
Cloudflare 在全球 300 個 PoP（Point of Presence）節點都宣告：
「我這裡有 104.16.x.x 這段 IP，來找我」

台灣用戶發請求到 104.16.x.x：
中華電信路由器查 BGP 表 → 最近的 Cloudflare 節點在台北 → 送去台北
德國用戶同樣請求到 104.16.x.x：
Deutsche Telekom 路由器 → 最近節點在法蘭克福 → 送去法蘭克福

兩個用戶打的是同一個 IP，但回應的是不同節點
```

**Anycast 的優點：**
- 不需要 IP 地理資料庫，不會誤判
- 某個節點掛掉 → BGP 自動重新路由到次近節點，幾秒內完成
- DDoS 攻擊流量被分散到全球節點，不會打垮單一機房

**常見使用 Anycast 的服務：**
- Cloudflare `1.1.1.1`、Google `8.8.8.8`（DNS 本身）
- Cloudflare CDN
- AWS Global Accelerator

---

## AWS Global Accelerator — Anycast 的實際應用

這是 AWS 的 Anycast 服務，值得單獨了解：

```
沒有 Global Accelerator：
台灣用戶 ──► 公共網路（品質不穩）──► us-east-1 Load Balancer

有 Global Accelerator：
台灣用戶 ──► 最近的 AWS Edge（東京）──► AWS 私有骨幹網路 ──► us-east-1 LB

差異：
- 公共網路：路由不穩定，延遲高，封包可能繞路
- AWS 私有骨幹：專線品質，延遲低且穩定
```

Global Accelerator 給你兩個靜態 Anycast IP，全球流量都走 AWS 的骨幹網路，不走公共網際網路。

---

## 主流 DNS 服務詳細比較

| 功能 | Route 53 | Cloudflare | NS1 | Google Cloud DNS |
|------|:--------:|:----------:|:---:|:----------------:|
| Health Check | ✅ 10s interval | ✅ | ✅ | ✅ |
| Latency Routing | ✅ 按 AWS region | ✅ Dynamic Steering | ✅ | ✅ |
| Geo Routing | ✅ 國家/洲 | ✅ 國家精度 | ✅ 可到城市 | ✅ |
| Weighted | ✅ | ✅ | ✅ | ✅ |
| Failover | ✅ | ✅ | ✅ | ✅ |
| Anycast | ❌（要用 Global Accelerator）| ✅ 內建 | ❌ | ❌ |
| 自訂路由邏輯 | ❌ | ⚠️ 有限 | ✅ Filter Chain | ❌ |
| 免費方案 | ❌ 按查詢量收費 | ✅ 基本功能免費 | ❌ | ❌ |
| 最適合 | AWS 用戶 | 所有人 | 大型複雜場景 | GCP 用戶 |

**NS1 的 Filter Chain（最進階）：**
```
可以組合多個 routing filter：
1. 先過濾掉 unhealthy 的 server
2. 再按 geo 分流
3. 再按 weighted 分配
4. 最後看 capacity（剩餘容量）

完全可程式化，支援 Data Source（傳自己的 metrics 進去讓 NS1 決策）
```

---

## 現代完整架構（從請求進來到 Server）

```
User 輸入 yourapp.com 按 Enter
           │
           ▼
┌──────────────────────────┐
│   智慧 DNS               │  Route 53 / Cloudflare DNS
│   ・Health Check         │  → 只給健康節點 IP
│   ・Latency Routing      │  → 選延遲最低的資料中心
│   ・Failover             │  → 主掛自動切備
│   TTL: 60s               │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   Anycast Edge           │  Cloudflare PoP / AWS Edge
│   ・BGP 最近節點接收      │  → 用戶流量在最近節點進入
│   ・DDoS 吸收            │  → 攻擊流量被分散
│   ・SSL 終止             │  → TLS handshake 在這層完成
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│   CDN Cache 層           │  Cloudflare / CloudFront
│   ・靜態資源直接回傳      │  → HTML/CSS/JS/圖片不打後端
│   ・Cache-Control 判斷   │  → 只有 Cache Miss 往下
└────────────┬─────────────┘
             │ Cache Miss 或動態請求
             ▼
┌──────────────────────────┐
│   Load Balancer（L7）    │  Nginx / AWS ALB / Kong
│   ・URL-based routing    │  /api → App Server
│   ・Least Connections    │  /ws  → WebSocket Server
│   ・毫秒級 Health Check  │  Server 掛 → 立刻移除
│   ・Sticky Session       │  需要時用 Cookie
└───────┬──────────────────┘
        │
   ┌────┴────┐
   ▼         ▼
Server A   Server B      ← Auto Scaling Group
(healthy)  (healthy)       流量大 → 自動加機器
                           流量小 → 自動縮減
           ↓
    ┌──────────────┐
    │ Redis / DB   │  ← Session、資料持久化
    └──────────────┘
```

---

## 各層職責總整理

| 層 | 工具 | 職責 | 切換速度 | 主要解決的問題 |
|----|------|------|----------|---------------|
| DNS | Route 53 / CF | 大方向路由、主備切換 | 60s（TTL）| 地理分流、節點故障 |
| Anycast Edge | Cloudflare PoP | 就近接收流量、DDoS | 幾秒（BGP）| 延遲、攻擊流量 |
| CDN | Cloudflare / CloudFront | 靜態資源 Cache | 即時 | Origin 流量減少 |
| LB | Nginx / ALB | 機房內細分配 | 毫秒 | 單台故障、負載均衡 |
| Auto Scaling | AWS ASG | 動態調整機器數量 | 1-3 分鐘 | 流量突增/降 |

---

## 為什麼 DNS 再強，還是需要 LB？

即使有 Route 53 Health Check + Anycast + 短 TTL，LB 仍然不可缺少：

```
DNS 的極限：
・最快切換 = TTL（60秒）    LB = 毫秒
・看不到 HTTP Header        LB 可以看 Cookie / URL / Header
・無法做 Sticky Session     LB 可以用 Cookie 黏住 User
・無法感知連線數            LB 可以做 Least Connections
・無法做 URL-based routing  LB 可以把 /api 跟 /static 分開送
```

DNS 管「哪個資料中心」，LB 管「資料中心裡的哪台機器」，職責不重疊。

---

## 一句話記憶法

```
DNS        → 「你飛去哪個城市」        （大方向，秒級）
Anycast    → 「你在哪個城市就地服務你」 （就近，BGP 自動）
CDN        → 「常見的東西我幫你存著」   （Cache，即時）
LB         → 「城市裡去哪棟樓」        （細分，毫秒）
Auto Scale → 「人多了開新門，人少了關」 （彈性，分鐘）
```
