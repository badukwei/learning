# System Design - DNS Round Robin & Global Load Balancing

Date: 2026-04-02

---

## DNS 是什麼？（先搞清楚基礎）

DNS（Domain Name System）是網路的「電話簿」。

```
你輸入：google.com
DNS 回答：這個域名對應到 IP 142.250.x.x
瀏覽器才知道要連去哪裡
```

一般網站：DNS 回傳 **1 個 IP**
Google：DNS 回傳 **一組 IP**（這就是 DNS Round Robin 的起點）

---

## DNS Round Robin 運作流程

```
Step 1：你在瀏覽器輸入 google.com

Step 2：作業系統問 DNS Server
         「google.com 的 IP 是多少？」

Step 3：DNS Server 回傳多個 IP：
         [142.250.80.46, 172.217.160.78, 216.58.200.110]
         ← 每次查詢，順序可能不同（輪詢）

Step 4：你的電腦連第一個 IP
         下一個用戶查詢，DNS 回傳順序不同 → 連到不同 IP

結果：不同用戶 → 不同 Server，流量被分散了
```

### 為什麼「順序不同」就能分散流量？

因為 Client 通常連**第一個** IP。DNS 每次把不同 IP 排在第一位，就達到輪流分流的效果。

```
User A 查詢結果：[IP-1, IP-2, IP-3]  → 連 IP-1
User B 查詢結果：[IP-2, IP-3, IP-1]  → 連 IP-2
User C 查詢結果：[IP-3, IP-1, IP-2]  → 連 IP-3
```

---

## Google 的全球負載平衡

Google 的那組 IP 不是指向同一個機房，而是全球分佈的資料中心：

```
           DNS 查詢 google.com
                    │
          ┌─────────┴─────────┐
          │   DNS 根據地區     │
          │   回傳不同 IP      │
          └─────────┬─────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
┌─────────┐   ┌─────────┐   ┌─────────┐
│  US     │   │  Europe │   │  Asia   │
│ Virginia│   │ Belgium │   │ Taiwan  │
│Data Ctr │   │Data Ctr │   │Data Ctr │
└─────────┘   └─────────┘   └─────────┘
     ▲              ▲              ▲
 美國用戶        歐洲用戶        亞洲用戶
```

台灣用戶查詢 google.com → DNS 偵測你在亞洲 → 回傳亞洲資料中心的 IP → 延遲從 200ms 降到 10ms。

這叫做 **GeoDNS**（地理感知 DNS），是 DNS Round Robin 的進階版。

---

## 兩大缺點詳解

### 缺點一：DNS 快取（Cache）問題

DNS 結果會被快取，在 TTL（Time to Live）到期前不會重新查詢。

```
情境：
09:00  User A 查詢 → DNS 回傳 IP-1（TTL = 1小時）
09:00  User A 的電腦記住：google.com = IP-1

09:30  IP-1 那台 Server 掛掉了 ❌

09:31  User A 重新整理頁面
       → 電腦不去問 DNS（快取還沒過期）
       → 直接連 IP-1 ← 連不上！

10:00  TTL 到期，電腦重新查詢 DNS
       → DNS 這次回傳 IP-2（已移除故障 Server）
       → 才恢復正常 ✅
```

**問題**：30 分鐘內 User A 都連不上，即使別的 Server 好好的。

快取存在於多個層：

```
瀏覽器快取
    ↓ 沒有才問
作業系統快取（/etc/hosts 或系統 DNS cache）
    ↓ 沒有才問
ISP（台灣大哥大、中華電信）的 DNS 快取
    ↓ 沒有才問
權威 DNS Server（google 自己的）
```

每一層都可能快取，換 Server IP 後可能要等好幾個小時才全面生效。

---

### 缺點二：不具備負載感知（Load-Blind）

DNS Round Robin 是「盲目」分配，它完全不知道每台 Server 現在的狀態。

```
正常情況：
Server A: CPU 20%  ← DNS 可能送流量來
Server B: CPU 22%  ← DNS 可能送流量來
Server C: CPU 18%  ← DNS 可能送流量來

異常情況（DNS 不知道）：
Server A: CPU 99%，快掛了  ← DNS 還是繼續送！
Server B: CPU 20%
Server C: CPU 15%
```

原因：DNS 只是個「電話簿查詢服務」，它不監控 Server 健康狀態，純粹機械式輪流。

**對比 Smart Load Balancer**：

```
DNS Round Robin：
「輪到你了，給你。」← 不管你死活

Nginx / AWS ALB：
「這台 Server /health 回 200 嗎？」
「回 503？那跳過，送給下一台。」← 會感知
```

---

## DNS Round Robin vs 現代 Load Balancer 比較

```
                DNS Round Robin        現代 LB（Nginx/ALB）
────────────────────────────────────────────────────────
感知 Server 狀態    ❌ 不知道               ✅ Health Check
快速切換故障機器    ❌ 受 TTL 限制           ✅ 毫秒級切換
按負載分配          ❌ 只能輪流              ✅ Least Connections
Sticky Session     ❌ 做不到               ✅ Cookie-based
部署成本            ✅ 極低（DNS 設定就好）  ❌ 要維護 LB 機器
全球地理分流        ✅ GeoDNS 可以做        ⚠️  需要多區域部署
```

---

## 現實中的用法

Google、Cloudflare 這類巨頭實際上是**兩層疊加**：

```
Layer 1：GeoDNS（把用戶導到最近的資料中心）
                    ↓
Layer 2：資料中心內的 Smart Load Balancer（細部分配到各台 Server）
```

DNS Round Robin 負責「大方向」（哪個大陸），Smart LB 負責「細分配」（哪台機器）。

---

## 一句話總結

| | DNS Round Robin | Smart Load Balancer |
|--|----------------|-------------------|
| 比喻 | 輪流分發傳單，不管對方在不在家 | 打電話確認對方在，才送過去 |
| 速度 | 快（純 DNS）| 稍慢（要 Health Check）|
| 智慧 | 無 | 有 |
| 適合 | 全球地理分流（第一層）| 機房內細部分配（第二層）|
