# System Design - Load Balancing

Date: 2026-04-02

---

## 什麼是負載平衡（Load Balancing）？

> 把進來的流量分散到多台 Server，避免單台過載。

```
                        ┌─────────────┐
                        │             │
User A ─────────────────►  Load       ├──► Server A
User B ─────────────────►  Balancer   ├──► Server B
User C ─────────────────►             ├──► Server C
                        └─────────────┘
```

---

## DNS 跟 Load Balancer 的關係

這兩個很容易搞混，因為**兩者都在「分流」**，但發生在完全不同的層。

### 一個請求的完整旅程

```
Step 1：你輸入 yourapp.com，按 Enter
        ↓
Step 2：DNS 查詢
        「yourapp.com 在哪裡？」
        DNS 回答：「IP 是 203.0.113.10」
        ↓
        ← DNS 的工作到這裡結束 →
        ↓
Step 3：瀏覽器連到 203.0.113.10（這個 IP 是 Load Balancer）
        LB 收到請求
        LB 決定：「送去 Server A」
        ↓
Step 4：Server A 處理請求，回傳結果
```

**DNS 回答「去哪個 IP」，LB 決定「這個 IP 後面去哪台機器」。**

---

### 職責切分

```
DNS                              Load Balancer
─────────────────────────────    ─────────────────────────────
在哪一層？  網路層（查 IP）        應用層（接 HTTP 請求）
誰在用？    作業系統 / 瀏覽器      瀏覽器連過來的 TCP 連線
做什麼？    域名 → IP             IP → 後端 Server
看得到什麼？只有域名和 IP         完整 HTTP Header、Cookie、URL
切換速度？  TTL（最快 60 秒）      毫秒
```

---

### 常見誤解：「DNS 就是 Load Balancer」

DNS Round Robin 確實能分流，但兩者本質不同：

```
DNS Round Robin：
  yourapp.com → [1.2.3.4, 5.6.7.8]  （給不同 IP）
  User A 拿到 1.2.3.4
  User B 拿到 5.6.7.8

  問題：DNS 不知道 Server 死活，不知道負載，TTL 快取讓切換很慢

真正的 Load Balancer：
  yourapp.com → 203.0.113.10（LB 的 IP，永遠這一個）
  LB 收到後再決定送 Server A 還是 Server B

  優點：毫秒切換、Health Check、Least Connections、Sticky Session
```

**現代架構的做法**：DNS 指向 LB 的 IP，LB 再分配到後端 Server。DNS 只做「導流到哪個資料中心」，LB 做「資料中心內部的細分配」。

---

### 兩者搭配的實際例子

假設你的服務部署在台灣和美國兩個地區：

```
DNS（Route 53）層：
  台灣用戶查詢 yourapp.com → 回傳 台灣 LB 的 IP（203.0.113.10）
  美國用戶查詢 yourapp.com → 回傳 美國 LB 的 IP（198.51.100.5）

台灣 LB（203.0.113.10）層：
  台灣 Server A（健康）→ 送流量
  台灣 Server B（健康）→ 送流量
  台灣 Server C（掛掉）→ 不送

美國 LB（198.51.100.5）層：
  美國 Server D → 送流量
  美國 Server E → 送流量
```

DNS 做地理分流，LB 做機房內的細分配。缺一不可。

---

## 負載平衡演算法

### 1. Round Robin（輪詢）— 最簡單

依序輪流送，每台 Server 機會均等。

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  ← 回到第一台
```

**適合**：所有 Server 規格相同、請求處理時間差不多

**缺點**：某些請求特別重（如大型運算），會造成某台 Server 累積太多

---

### 2. Weighted Round Robin（加權輪詢）

規格好的 Server 多分配流量。

```
Server A（8 core）→ 權重 4
Server B（4 core）→ 權重 2
Server C（2 core）→ 權重 1

每 7 個請求：A 拿 4 個，B 拿 2 個，C 拿 1 個
```

---

### 3. Least Connections（最少連線）

把新請求送給「目前連線數最少」的 Server。

```
Server A: 100 connections
Server B: 45 connections   ← 下一個請求送這裡
Server C: 80 connections
```

**適合**：請求處理時間差異大（如有些請求要跑 10 秒）

---

### 4. IP Hash（IP 雜湊）

根據 User 的 IP 決定送去哪台，同一個 IP 永遠送同一台 Server。

```
User IP: 192.168.1.1 → hash → Server B（固定）
User IP: 10.0.0.5   → hash → Server A（固定）
```

本質上就是 Sticky Sessions 的一種實作（但不依賴 Cookie）。

**缺點**：IP 不代表一個 User（NAT 後面可能有幾百人共用同一個 IP）

---

## 現代負載平衡架構

### Layer 4 vs Layer 7

#### 先了解 OSI 模型

網路通訊被分成 7 層，每層只做自己的事，不管上下層的細節。

```
Layer 7  應用層    HTTP、HTTPS、WebSocket、FTP
Layer 6  表示層    加密、壓縮（TLS 在這附近）
Layer 5  會話層    連線建立與管理
Layer 4  傳輸層    TCP、UDP（負責「送到哪個 Port」）
Layer 3  網路層    IP（負責「送到哪個 IP」）
Layer 2  資料連結層 MAC address（區域網路內）
Layer 1  實體層    網路線、電訊號
```

**重點**：越高層，能看到的資訊越多，但處理成本也越高。

---

#### Layer 4 LB — 傳輸層，只看 IP + Port

Layer 4 LB 在 TCP 連線建立時就做決策，**根本不解析 HTTP 內容**。

```
封包進來，Layer 4 LB 看到的：
  來源 IP:   192.168.1.100
  目的 IP:   203.0.113.10（LB 自己）
  目的 Port: 443

LB 只根據這些資訊決定：「轉給 Server A」
然後把 TCP 連線直接 proxy 過去

LB 完全不知道：
  - 這個請求是 GET /api/users 還是 POST /orders
  - Cookie 裡有什麼
  - 這個用戶是誰
```

**流程：**

```
Client ──TCP連線──► Layer 4 LB ──TCP轉發──► Server A
                   （只看IP+Port）
```

**適合：**
- 遊戲伺服器（TCP/UDP，不走 HTTP）
- 資料庫連線 proxy（MySQL port 3306）
- 需要極低延遲的場景
- AWS NLB（Network Load Balancer）就是 Layer 4

---

#### Layer 7 LB — 應用層，能看完整 HTTP

Layer 7 LB 會**把 HTTP 請求完整解析**後再做決策。

```
封包進來，Layer 7 LB 看到的：
  POST /api/orders HTTP/1.1
  Host: yourapp.com
  Cookie: session_id=abc123; lb_route=server-a
  Authorization: Bearer eyJhbG...
  Content-Type: application/json

LB 能根據這些資訊做很多事：
  - URL 路徑 /api/* → 送去 API Server
  - Cookie 有 lb_route=server-a → Sticky Session，送回 Server A
  - Header 有 Authorization → 驗 JWT
  - Host 是 admin.yourapp.com → 送去 Admin Server
```

**流程：**

```
Client ──HTTP請求──► Layer 7 LB ──新的HTTP請求──► Server A
                   （解析完整HTTP後決定）
```

注意：Layer 7 LB 是「終止再轉發」，不是單純 proxy TCP：
```
Client → LB：建立一條 TCP 連線
LB → Server：再建立另一條 TCP 連線

LB 是中間人，完全讀懂 HTTP 內容再轉發
```

**適合：**
- 一般 Web App、API（走 HTTP/HTTPS）
- 需要 URL routing、Sticky Session、Header 操作
- AWS ALB（Application Load Balancer）、Nginx 都是 Layer 7

---

#### 比較總結

|     | Layer 4 | Layer 7 |
|-----|---------|---------|
| 看什麼 | IP + Port | 完整 HTTP（URL、Header、Cookie）|
| 決策時機 | TCP 連線建立時 | HTTP 請求解析後 |
| 速度 | 快（不解析內容）| 稍慢（要解析 HTTP）|
| Sticky Session | ❌（看不到 Cookie）| ✅ |
| URL routing | ❌ | ✅ |
| SSL 終止 | ❌（不解密）| ✅ |
| 適合 | 遊戲、DB proxy、UDP | Web App、API、微服務 |
| AWS 對應 | NLB | ALB |

---

**Layer 7 範例：按路徑分流**

```
/api/*     → API Server 群組   （需要運算）
/static/*  → CDN / Static Server（直接回傳檔案）
/ws/*      → WebSocket Server  （需要長連線）
/admin/*   → Admin Server      （只有內網可連）
```

這些判斷都需要「看得到 URL」，Layer 4 做不到，必須用 Layer 7。

---

## 現代解法全景

### 架構一：單層 Load Balancer（中小型）

```
                    ┌──────────────┐
                    │              │
Internet ──────────►  Nginx / ALB  ├──► Server A
                    │              ├──► Server B
                    │              ├──► Server C
                    └──────────────┘
                           │
                           ▼
                    Redis（Session）
```

---

### 架構二：多層 Load Balancer（大型服務）

```
                ┌─────────────┐
Internet ──────►│  Global LB  │  (Cloudflare / Route53)
                └──────┬──────┘
                       │ GeoDNS（按地區分流）
          ┌────────────┴────────────┐
          ▼                         ▼
   ┌─────────────┐           ┌─────────────┐
   │  Region LB  │           │  Region LB  │
   │  (US-East)  │           │  (AP-Tokyo) │
   └──────┬──────┘           └──────┬──────┘
          │                         │
   ┌──────┴──────┐           ┌──────┴──────┐
   │  Server A   │           │  Server D   │
   │  Server B   │           │  Server E   │
   │  Server C   │           │  Server F   │
   └─────────────┘           └─────────────┘
```

---

### 架構三：Microservices + API Gateway（現代主流）

```
                    ┌───────────────────┐
                    │    API Gateway    │
Internet ──────────►│  (Kong / Nginx /  │
                    │   AWS API GW)     │
                    └─────────┬─────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                  ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  User Service│  │ Order Service│  │ Payment Svc  │
    │  (3 replicas)│  │ (5 replicas) │  │ (2 replicas) │
    └──────────────┘  └──────────────┘  └──────────────┘
            │                 │                  │
            └─────────────────┴──────────────────┘
                              │
                    ┌─────────┴──────────┐
                    │   Shared Services  │
                    │  Redis │ DB │ MQ   │
                    └────────────────────┘
```

**API Gateway 做的事**：

- 身份驗證（Auth）
- Rate Limiting（限流）
- 路由（把 `/users` 送去 User Service）
- SSL 終止

---

## Health Check — Server 掛了怎麼辦？

Load Balancer 定期對每台 Server 發送探針（Probe）：

```
LB → GET /health → Server A  → 200 OK   ✅ 繼續送流量
LB → GET /health → Server B  → timeout  ❌ 標記為 unhealthy，停止送流量
LB → GET /health → Server C  → 200 OK   ✅ 繼續送流量
```

**Server B 恢復後**，LB 重新偵測到 200 → 自動加回輪詢。

```
/health endpoint 通常檢查：
- Server 本身是否活著
- DB 連線是否正常
- Redis 連線是否正常
```

---

## 流量激增時的自動擴展

```
流量正常時：
LB ──► [A] [B]

流量激增（CPU > 70%）：
Auto Scaling 觸發 → 新開 Server C, D
LB ──► [A] [B] [C] [D]

流量降低後：
Auto Scaling 縮減 → 關掉 C, D
LB ──► [A] [B]
```

AWS 叫 **Auto Scaling Group**，GCP 叫 **Managed Instance Group**。

---

## 各工具比較


| 工具             | 類型           | 適合場景                  |
| -------------- | ------------ | --------------------- |
| **Nginx**      | Software LB  | 自架伺服器、小中型             |
| **HAProxy**    | Software LB  | 高效能、更細緻的設定            |
| **AWS ALB**    | Cloud LB（L7） | AWS 上的 HTTP/HTTPS 流量  |
| **AWS NLB**    | Cloud LB（L4） | TCP/UDP、極低延遲          |
| **Cloudflare** | Global LB    | CDN + DDoS 防護 + LB 一起 |
| **Kong**       | API Gateway  | Microservices         |


---

## Frontend Engineer 要知道的

1. **CORS 問題**：LB 後面有多台 Server，CORS Header 要統一在 LB 層設定，不能只設在某一台
2. **WebSocket**：WS 連線是長連線，LB 要支援 `Upgrade` Header（Nginx 要額外設定）
3. **Cache-Control**：靜態資源應該在 CDN / LB 層 cache，不要每次打到 App Server
4. **Rate Limiting**：通常在 LB / API Gateway 層做，不在前端做

