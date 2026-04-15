# Load Balancer — 總體設計

## 工具選擇

### 選擇框架

| 維度 | 問題 |
|------|------|
| 協議 | HTTP only？還是需要 TCP/UDP/WebSocket/gRPC？ |
| 動態設定 | 新增後端節點要不要重啟 LB？ |
| 連線特性 | 短連線（API）還是長連線（WebSocket）？ |
| 架構 | 單體？微服務？K8s？ |
| 團隊規模 | 有沒有人力維護自架？ |
| 成本 | 雲端按流量 vs 硬體買斷 |
| 合規 | 資料能不能出境？ |

---

### 雲端 LB

#### AWS ALB
- **收費**：$0.008/LCU-hour，約 $16–50/月（中小流量）
- **優點**：HTTP/HTTPS/WebSocket、內建 SSL、path/header/host routing、整合 Auto Scaling
- **缺點**：AWS 生態鎖定，TB 級流量費用高

#### Cloudflare Load Balancing
- **收費**：$5/月起
- **優點**：附帶 CDN + DDoS + WAF，Geo-based routing，設定簡單
- **缺點**：WebSocket 要 Pro 以上，細粒度控制較少

#### GCP Cloud Load Balancing
- **收費**：約 $18/月起
- **優點**：全球單一 Anycast IP，與 GKE 深度整合
- **缺點**：GCP 生態鎖定，設定複雜

---

### 自架 LB

#### Nginx
- **收費**：免費（Nginx Plus $2,500/年）
- **優點**：最泛用，同時做 Web Server + Reverse Proxy + LB，社群豐富
- **缺點**：要管機器，動態新增節點要 reload

#### HAProxy
- **收費**：免費
- **優點**：純 LB 專用，效能最高，即時 Stats 頁面，支援 L4 + L7
- **缺點**：不能做 Web Server，設定語法複雜

#### Traefik
- **收費**：免費
- **優點**：容器原生，自動偵測 Docker/K8s 服務，內建 Let's Encrypt、Dashboard
- **缺點**：非容器環境反而複雜，大流量效能不如 Nginx/HAProxy

#### Envoy
- **收費**：免費
- **優點**：動態設定（xDS API）不需重啟，支援 gRPC/HTTP2，Circuit Breaker 內建
- **缺點**：學習曲線陡，YAML 設定極深，適合 K8s 微服務

---

### 選擇速查

```
新創 / 小團隊          → AWS ALB 或 Cloudflare，不要自己管機器
一般自架               → Nginx（最泛用）
高流量 / 純 LB 需求    → HAProxy
Docker / K8s           → Traefik 或 Envoy
微服務 / Service Mesh  → Envoy（Istio）
純靜態前端             → 根本不需要 LB，Vercel + Cloudflare 就夠
```

---

## 架構模式

### 單一 LB（有 SPOF，只適合開發環境）

```
用戶 → [LB 公開 IP] → [Server A] [Server B]
```

### Active-Passive HA（生產環境標配）

```
用戶 → VIP → [LB Primary] ←VRRP→ [LB Standby]
                ↓      ↓
             [S A]   [S B]
```

### Active-Active HA（高流量）

```
DNS Round Robin
 ↙           ↘
[LB 1]      [LB 2]
 ↓   ↓       ↓   ↓
[S A][S B] [S C][S D]
```

### 現代雲端（主流新創）

```
用戶
 ↓
Cloudflare（DDoS + CDN + WAF）
 ↓
AWS ALB（HTTP routing）
 ↙          ↓            ↘
Next.js    API Servers   WebSocket Servers
              ↓
           Redis（Cache + Pub/Sub）
              ↓
           PostgreSQL
```

### K8s Ingress

```
用戶
 ↓
Ingress Controller（Nginx / Traefik）
 ↓            ↓
[Service A]  [Service B]
 ↓    ↓       ↓    ↓
[Pod][Pod]  [Pod][Pod]
```

---

## 程式碼

### Nginx

```nginx
http {
    upstream backend {
        # Round Robin（預設）
        server 192.168.1.10:3000;
        server 192.168.1.11:3000;

        # least_conn;          # Least Connections
        # ip_hash;             # Sticky Sessions
        # server ... weight=3; # 加權

        server 192.168.1.10:3000 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /api/ { proxy_pass http://api_servers; }
        location /images/ { proxy_pass http://image_servers; }
    }
}
```

### HAProxy

```
frontend http_front
    bind *:80
    acl is_api hdr(host) -i api.example.com
    use_backend api_servers if is_api
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    server s1 192.168.1.10:3000 check
    server s2 192.168.1.11:3000 check

backend api_servers
    balance leastconn
    server api1 192.168.1.20:4000 check
```

### Traefik（Docker）

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports: ["80:80"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  app_1:
    image: my-app
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`example.com`)"
  app_2:
    image: my-app  # 自動納入 LB pool
```

### Keepalived（VIP 飄移）

```
# Primary
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    virtual_ipaddress { 192.168.1.100 }
}

# Standby：priority 改 90，state 改 BACKUP
```

---

## AWS LB 三種類型與協議選擇

### 三種 LB

| | ALB | NLB | GLB |
|--|-----|-----|-----|
| 層級 | Layer 7 | Layer 4 | Layer 3 |
| 協議 | HTTP/HTTPS/WebSocket/gRPC | TCP/UDP/TLS | IP |
| 速度 | 稍慢（解析 HTTP）| 極快（pass-through）| - |
| 路由 | path/header/host | IP + Port | - |
| 靜態 IP | 否 | 是 | - |
| 費用 | 稍高 | 約便宜 25% | - |
| 適合 | API、微服務、WebSocket | 遊戲、DB proxy、靜態 IP | 防火牆 appliance |

### 各協議對應

| 協議 | 選擇 | 說明 |
|------|------|------|
| HTTP / HTTPS | ALB | 最常見，支援 SSL 終止、WAF、path routing |
| WebSocket | ALB | 原生支援，升級握手自動處理 |
| gRPC | ALB | 要在 Target Group 指定 protocol version = gRPC |
| TCP / UDP（自訂協議）| NLB | ALB 看不懂非 HTTP 內容 |
| 需要靜態 IP | NLB | ALB IP 會變動，B2B / 防火牆白名單場景必用 |
| 極低延遲 | NLB | pass-through 不解析封包，延遲更低 |

### 什麼時候需要 TCP LB（NLB）

**1. 非 HTTP 協議**
遊戲 server（自訂 binary protocol）、MQTT（IoT）、FTP、SSH proxy。ALB 看不懂這些。

**2. 資料庫 proxy**
PostgreSQL cluster、Redis cluster 前面加 LB，DB 走 TCP，不是 HTTP，只能 NLB 或 HAProxy L4。

**3. 需要靜態 IP**
客戶要把你的 IP 加進防火牆白名單（金融、企業 B2B 常見），ALB IP 會動態變動，必須用 NLB 的 Elastic IP。

**4. 極低延遲**
高頻交易、即時競價（RTB），NLB pass-through 延遲遠低於 ALB。

**5. 超高連線數**
百萬條長連線（大型遊戲、即時通訊），NLB connection handling 比 ALB 更有效率。

### 判斷規則

```
HTTP / HTTPS / WebSocket / gRPC？ → ALB
非 HTTP、需要靜態 IP、延遲極敏感、超高連線數？ → NLB
```

一般 Web 應用 99% 用 ALB，NLB 出現在基礎設施層（DB proxy、遊戲後端、B2B 整合）。

---

## ALB + NLB 同時使用的場景

### 場景一：遊戲平台

```
用戶
 ↓
ALB → /api/*  → REST API（帳號、道具、排行榜）
    → /ws/*   → WebSocket（即時對戰）

NLB → :7777  → Game Server（自訂 UDP，低延遲射擊遊戲）
```

遊戲實際連線需要 UDP + 最低延遲，ALB 解析 HTTP 的 overhead 不能接受。

### 場景二：金融交易平台

```
散戶（Web / App）
 ↓
ALB → /api/*  → 下單、查詢（HTTP）
    → /ws/*   → 即時報價（WebSocket，ALB 原生支援）

機構客戶（程式交易，需要靜態 IP 白名單）
 ↓
NLB（Elastic IP）→ FIX Protocol Server（金融業標準 TCP 協議）
```

散戶走 ALB，機構走 NLB。FIX Protocol 不是 HTTP，且對方防火牆需要固定 IP。

### 場景三：NLB 前接 ALB（AWS 特殊組合）

```
用戶 → NLB（提供靜態 IP）→ ALB（做 HTTP routing、WAF）→ 後端
```

ALB 沒有靜態 IP，但客戶要白名單。NLB 在前提供固定 IP，pass-through 給 ALB 做 L7 routing。AWS 官方支援此架構。

### DeFi 前端的判斷

```
ALB → /api/*  → Next.js SSR + API
    → /ws/*   → WebSocket（即時價格、倉位更新）← ALB 就夠

需要 NLB 的條件：
  - 機構客戶要固定 IP 白名單
  - 有自訂 TCP 協議（幾乎不會）
一般 DeFi 前端不需要 NLB。
```

---

## WebSocket 的 LB 問題

WebSocket 是長連線，用戶換到不同 server 會斷線。

```
解法 A：IP Hash → 簡單，但 server 掛掉所有連線斷
解法 B：Redis Pub/Sub（推薦）
  → 任何 server 收到更新，publish 給 Redis
  → 所有 server 訂閱，推給自己連著的用戶
  → server 掛掉只影響那台連著的用戶，其他不受影響
```
