# System Design - Microservices 架構實例

Date: 2026-04-02

---

## 用「電商平台」當例子

以下用一個簡化的電商平台（像 蝦皮/Shopee）來說明 Microservices 架構。

---

## Monolith vs Microservices

先對比一下，才知道為什麼要拆。

### Monolith（單體架構）

```
┌──────────────────────────────────────────┐
│              一個大 App                   │
│                                          │
│  用戶模組  │  商品模組  │  訂單模組        │
│  購物車    │  付款模組  │  通知模組        │
│  搜尋模組  │  物流模組  │  推薦模組        │
│                                          │
│         一個 DB 存所有資料                │
└──────────────────────────────────────────┘
```

**問題**：
- 改一個模組 → 整個 App 要重新部署
- 付款模組流量暴增 → 整個 App 一起 scale，浪費資源
- 一個模組 bug → 可能把整個 App 拉掛
- 團隊大了 → 大家改同一份 code，merge conflict 地獄

---

### Microservices（微服務架構）

把每個模組拆成獨立的 Service，各自部署、各自 Scale。

```
                    ┌──────────────────┐
 Web / App          │   API Gateway    │
 ──────────────────►│  (Kong / AWS GW) │
                    └────────┬─────────┘
                             │ 依路由分發
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ User Service │   │Product Svc   │   │ Order Service│
│              │   │              │   │              │
│ /users/*     │   │ /products/*  │   │ /orders/*    │
│              │   │              │   │              │
│ PostgreSQL   │   │ PostgreSQL   │   │ PostgreSQL   │
│ (users DB)   │   │ (products DB)│   │ (orders DB)  │
└──────────────┘   └──────────────┘   └──────┬───────┘
                                             │ 下訂單後
         ┌───────────────────────────────────┘
         │  發事件（Event）到 Message Queue
         ▼
┌────────────────────────────────────────────────────┐
│              Message Queue（Kafka / SQS）           │
└──────┬────────────────┬──────────────┬─────────────┘
       │                │              │
       ▼                ▼              ▼
┌──────────┐   ┌──────────────┐  ┌──────────────┐
│ Payment  │   │Notification  │  │ Inventory    │
│ Service  │   │ Service      │  │ Service      │
│          │   │              │  │              │
│ 處理付款  │   │ 發 Email/推播 │  │ 扣庫存       │
└──────────┘   └──────────────┘  └──────────────┘
```

---

## 下一筆訂單的完整流程

```
用戶點「結帳」

Step 1: API Gateway
  POST /orders
  → 驗證 JWT Token（Auth）
  → Rate Limiting（防止惡意大量下單）
  → 轉發給 Order Service

Step 2: Order Service
  → 建立訂單（狀態：PENDING）
  → 存進自己的 DB
  → 發一個事件到 Message Queue：
    {
      event: "order.created",
      orderId: "ord-123",
      userId: "usr-456",
      items: [...],
      total: 1500
    }
  → 立刻回傳 202 Accepted 給用戶（不等付款完成）

Step 3: Message Queue 通知各 Service（非同步）

  Payment Service 收到：
    → 向金流發起扣款請求
    → 成功 → 發 "payment.success" 事件
    → 失敗 → 發 "payment.failed" 事件

  Inventory Service 收到：
    → 把商品庫存 -1
    → 如果庫存不足 → 發 "inventory.insufficient" 事件

Step 4: 各 Service 繼續處理後續事件

  Order Service 收到 "payment.success"：
    → 把訂單狀態改成 CONFIRMED

  Notification Service 收到 "payment.success"：
    → 寄確認信給用戶
    → 發推播通知

  Order Service 收到 "payment.failed"：
    → 把訂單狀態改成 CANCELLED
    → 通知 Inventory Service 還原庫存

Step 5: 用戶輪詢或 WebSocket 收到最終狀態
  訂單：CONFIRMED ✅
```

---

## 為什麼用 Message Queue，不直接 Service 呼叫 Service？

### 直接呼叫（同步）的問題

```
Order Service
  → 直接呼叫 Payment Service
      → 直接呼叫 Inventory Service
          → 直接呼叫 Notification Service

問題：
1. Notification Service 掛掉 → 整個下單流程失敗（只是發通知失敗而已）
2. Payment Service 很慢（要等金流回應）→ 用戶等很久
3. 各 Service 耦合：Order Service 要知道 Payment Service 的位置
```

### Message Queue（非同步）的優點

```
Order Service 只管：「把事件丟進 Queue，我的工作做完了」
                         ↓
其他 Service 各自從 Queue 拿事件處理

優點：
1. Notification Service 掛掉 → 訂單還是成功，等它恢復再補發通知
2. 用戶立刻拿到 202，不需要等付款完成
3. Order Service 不需要知道有哪些 Service 在訂閱事件
4. 要加新功能（如「給用戶點數」）→ 加一個新 Service 訂閱事件即可，不改 Order Service
```

---

## 每個 Service 有自己的 DB（Database per Service）

這是 Microservices 最重要的原則之一。

```
❌ 錯誤做法：共用同一個 DB
  User Svc ──┐
  Order Svc ─┼──► 同一個 PostgreSQL
  Product Svc┘

  問題：
  - DB schema 改了 → 所有 Service 要一起改
  - Order Svc 大量查詢 → 影響 User Svc 的效能
  - 一個 Service 的 bug 可能毀掉整個 DB

✅ 正確做法：各自有 DB
  User Svc    ──► PostgreSQL（users）
  Order Svc   ──► PostgreSQL（orders）
  Product Svc ──► PostgreSQL（products）
  Search Svc  ──► Elasticsearch（搜尋索引）
  Session Svc ──► Redis（快取）
  Cart Svc    ──► Redis（購物車，暫存）
```

每個 Service 可以選最適合自己的資料庫類型，這叫 **Polyglot Persistence**。

---

## 各 Service 獨立 Scale

這是 Microservices 比 Monolith 最大的優勢：

```
雙 11 購物節當天：

Search Service  → 查詢量爆炸 → Scale 到 20 個 replica
Order Service   → 下單量大   → Scale 到 15 個 replica
User Service    → 平穩       → 維持 3 個 replica
Notification Svc→ 大量發通知 → Scale 到 10 個 replica

vs Monolith：全部模組一起 Scale → 浪費資源
```

---

## 服務間溝通的兩種方式

### 同步（Synchronous）— 需要立即回應時用

```
用 HTTP/REST 或 gRPC

Order Service → GET /products/123 → Product Service
                                   ← { name: "iPhone", price: 30000 }

適合：查詢資料、需要立刻拿到結果
```

### 非同步（Asynchronous）— 不需要立即回應時用

```
用 Message Queue（Kafka / RabbitMQ / AWS SQS）

Order Service → [order.created] → Queue → Payment Service
                                        → Inventory Service
                                        → Notification Service

適合：觸發後續動作、可以慢慢處理、不在乎順序
```

---

## 實際的技術選型（現代電商）

| 角色 | 工具 | 原因 |
|------|------|------|
| API Gateway | Kong / AWS API Gateway | Auth、Rate Limit、路由一起做 |
| Service 框架 | Node.js / Go / Python | 依團隊熟悉度，各 Service 可不同 |
| 同步溝通 | gRPC（Service 間）/ REST（對外）| gRPC 更快，型別安全 |
| Message Queue | Kafka | 高吞吐、事件持久化、可重播 |
| 容器化 | Docker | 每個 Service 打成 image |
| 編排 | Kubernetes（K8s）| 管理幾十個 Service 的部署、Scale |
| Service Discovery | Consul / K8s DNS | Service 之間怎麼找到彼此 |
| 監控 | Prometheus + Grafana | 每個 Service 的指標 |
| 追蹤 | Jaeger / OpenTelemetry | 一個請求跨多個 Service 時怎麼 debug |

---

## Microservices 的缺點（不是萬能的）

```
1. 複雜度大幅提升
   Monolith：一個 process，直接 function call
   Microservices：跨網路呼叫，要處理逾時、重試、冪等性

2. 分散式事務難處理
   「下單 + 扣庫存 + 扣款」要全部成功或全部回滾
   跨多個 Service 的 DB 沒辦法用傳統 Transaction
   需要 Saga Pattern 或 2-Phase Commit

3. 本機開發變複雜
   跑一個功能要同時啟動 5 個 Service + Kafka + Redis + DB
   通常用 docker-compose 解決

4. 維運成本高
   幾十個 Service 要監控、部署、版本管理
   沒有 K8s 和 CI/CD 幾乎管不動
```

---

## 什麼時候用 Microservices？

```
適合：
  ✅ 團隊大（10人以上），需要平行開發不互相影響
  ✅ 不同模組有明顯不同的 Scale 需求
  ✅ 需要技術多樣性（搜尋用 ES，快取用 Redis）
  ✅ 業務已經穩定，邊界清楚

不適合：
  ❌ 小團隊（< 5人），複雜度 > 好處
  ❌ 早期新創，需求還在變，Service 邊界不清楚
  ❌ 沒有 DevOps / K8s 能力

建議：先從 Monolith 開始，長大了再拆。
      （這叫 Modular Monolith → 再慢慢拆成 Microservices）
```
