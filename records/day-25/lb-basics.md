# Load Balancer — 基礎概念

## 是什麼

LB 是一個「黑盒子」，放在用戶和後端伺服器之間。

```
用戶 → [LB 公開 IP] → [Server A 私有 IP]
                    → [Server B 私有 IP]
```

後端用私有 IP 的原因：節省公網 IP，且後端不直接暴露於網際網路。

---

## Layer 4 vs Layer 7

| | Layer 4 | Layer 7 |
|--|---------|---------|
| 看什麼 | TCP/IP（IP + Port）| HTTP header、path、host |
| 速度 | 快（不解析內容）| 稍慢（要解析 HTTP）|
| 彈性 | 低 | 高（可根據路徑、hostname 路由）|
| 適合 | 原始 TCP、UDP、高效能 | HTTP API、微服務、Path-based routing |

---

## 分流算法

| 算法 | 說明 | 適用 |
|------|------|------|
| Round Robin | 輪流分 | 機器規格相同 |
| Weighted Round Robin | 按權重分 | 機器規格不同 |
| Least Connections | 送給目前連線數最少的 | 請求處理時間差異大 |
| IP Hash | 同一 IP 永遠打同一台 | 需要 Sticky Sessions |
| Random P2C | 隨機選兩台，選較閒那台 | 大規模分散式 |

---

## Sticky Sessions

當流量分散到多台 server，Session 存在本機時，用戶換到另一台 server 會被踢出登入。

**解法 A：IP Hash（LB 層處理）**
同 IP 永遠打同一台，簡單但 server 掛掉連線全斷。

**解法 B：Cookie Sticky（LB 插入 Cookie）**
LB 在 response 插入隨機 ID，下次 request 讀取 Cookie 導回同一台。

**解法 C：Redis Session / JWT（治本）**
Session 存 Redis 或改用 JWT，任何 server 都能處理，LB 不需要 Sticky。

---

## Health Check

LB 定期確認後端是否存活，掛掉的自動踢出 pool。

**不好的設計**：打 `/`，回 200 就過 → 其實 DB 掉了只是 HTTP 還在

**好的設計**：打 `/health`，endpoint 裡真的確認：
- DB 連線正常
- Redis 連線正常
- 關鍵依賴可達

```
UnhealthyThresholdCount: 3   # 連失敗 3 次才踢出
HealthCheckInterval: 10s     # 每 10 秒檢查一次
HealthyThresholdCount: 2     # 連成功 2 次才放回來
```

---

## SPOF 與 HA

單一 LB = 單點故障（SPOF）：LB 掛掉 → 全站掛掉。

### Active-Passive（常見生產環境）

兩台 LB，用戶連的是 VIP（Virtual IP）。Primary 掛掉，Standby 約 3 秒內接管 VIP。

```
用戶 → VIP 192.168.1.100
         ↓
      [LB Primary] ←VRRP心跳→ [LB Standby]
         ↓      ↓
      [S A]   [S B]
```

### Active-Active（高流量）

兩台都在服務，DNS 輪流給兩個 IP。一台掛掉流量走另一台。
需要 Session 共享（Redis 或 JWT），不能存本機。

### 雲端（內建 HA）

AWS ALB、Cloudflare 跨多 AZ，一個機房掛掉自動切換，不需要自己搞 Keepalived。

---

## Connection Draining（優雅下線）

把 server 從 pool 移除時，不直接斷掉現有連線：

```
LB 停止送新 request 給這台
→ 等現有 request 跑完
→ 超過 timeout 才強制斷

一般 API：30 秒
WebSocket：300 秒以上
```

注意：app 本身也要處理 SIGTERM，不然 LB 等它，它自己先死了。

---

## Slow Start（新節點保護）

新加進來的 server 不能馬上收滿流量，要暖機（cache warmup、JIT 編譯）：

```
slow_start 30s：
  第 1 秒：3% 流量
  第 15 秒：50% 流量
  第 30 秒：100% 流量
```

---

## TCP vs UDP vs HTTP

### 協議層次關係

```
Layer 7（應用層）  HTTP、WebSocket、gRPC、FTP、SSH、Git
                        ↑ 都建在 TCP 上面
Layer 4（傳輸層）  TCP、UDP
                        ↑ 建在 IP 上面
Layer 3（網路層）  IP
```

HTTP 是 TCP 的上層應用，不是替代品。

### TCP 的特性

保證順序、完整、流量控制。代價是每個封包要等 ACK，有延遲。

選 TCP 的場景：
```
Git push    → 少一個封包，code 就壞掉    → TCP
HTTP API    → response 少一段就解析失敗  → TCP
資料庫連線  → SQL 結果不完整比沒有更危險 → TCP
金融交易   → 少一筆訂單是大問題         → TCP
```

### UDP 的特性

不保證可靠，速度快，少幾個封包沒關係。

選 UDP 的場景：
```
線上遊戲  → 少一個位置更新封包沒差，下一個馬上來 → UDP
視訊通話  → 少幾幀可以接受，但不能卡頓          → UDP
DNS 查詢  → 一個小封包，沒回應就重問            → UDP
直播串流  → 少幾個影格比整個卡住好              → UDP
```

### 為什麼不直接用 HTTP

有些應用不需要 HTTP 格式，自己定義格式：
```
Git protocol → 自己的 binary 格式，不是 HTTP request/response
PostgreSQL   → 自己的 wire protocol
SSH          → 自己的協議格式
```

這些場景說「用 TCP」是指「直接用 TCP，不加 HTTP 這層，自己定義格式」。

```
HTTP = TCP + 約定好的格式（header、method、status code）
TCP  = 可靠傳輸，格式自己定
UDP  = 不保證可靠，格式自己定，速度快

不需要 HTTP 格式 → 直接用 TCP
需要 HTTP 格式   → 用 HTTP（底層還是 TCP）
```

---

## SPOF 實際怎麼避免

LB 掛掉有兩種情況，處理方式不同：
1. **Process 死掉**（OOM、crash）— 機器還活著
2. **機器死掉**（硬體故障、機房斷電）

### 解法一：Keepalived + VIP（自架標配）

用戶連的是 VIP（浮動 IP），不是某台機器。誰活著誰拿走它。

```
平常：Primary 持有 VIP，每秒送 VRRP 心跳給 Standby

Primary 掛掉（任何原因）：
  → VRRP 心跳停了
  → Standby 約 3 秒後偵測到
  → 用 ARP 廣播宣告自己是新的 VIP 持有者
  → Router 更新 ARP table → 流量切過來
```

切換速度約 3 秒（可調成 1 秒，但誤判率上升）。

陷阱：Standby 設定要和 Primary 完全一致，通常用 Ansible / Puppet 確保同步。

### 解法二：DNS Failover

Primary 掛掉就改 DNS，把流量指向 Standby：

```
example.com → 1.2.3.4（Primary LB）
           → 5.6.7.8（Standby LB）
```

問題：DNS TTL。改了 DNS，各地 DNS server 可能還快取舊 IP，要等 TTL 過期才生效（幾分鐘到幾小時）。通常是最後手段，不是主力 HA 方案。

### 解法三：雲端 LB（內建 HA）

AWS ALB 跨多個 AZ 運行，一個機房整個掛掉，流量自動切到其他 AZ，毫秒級，用戶完全無感。不需要自己維護心跳、VIP、Keepalived。

### 比較

| 方案 | 切換速度 | 複雜度 | 適合 |
|------|---------|--------|------|
| Keepalived + VIP | ~3 秒 | 中 | 自架生產環境 |
| DNS Failover | 幾分鐘起 | 低 | 緊急備援 |
| 雲端 LB | 毫秒級 | 極低 | 新創、雲端架構 |

---

## 面試速查

| 問題 | 答案 |
|------|------|
| LB 怎麼知道後端掛掉？ | Health Check，連不上就踢出 pool |
| 怎麼保持 Session？ | IP Hash / Cookie Sticky，或 Redis Session / JWT |
| LB 本身 HA？ | Active-Passive + Keepalived VIP 飄移，或直接用雲端 LB |
| Layer 4 vs Layer 7？ | L4 快但只看 IP/Port，L7 彈性高可看 HTTP 內容 |
| 後端為何用私有 IP？ | 節省公網 IP，不直接暴露於網際網路 |
