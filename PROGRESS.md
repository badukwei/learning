# 求職進度追蹤

更新：2026-04-23 ｜ Month 2，Day 30

---

## 求職方向

目標：全遠端海外工作，年薪 $70k+（底線 $50k）

### 目標職位（優先順序）
1. **Frontend Engineer**（主力，技術棧直接符合）
2. **Fullstack / Product Engineer**（熟悉 DB + 系統設計後開放）
3. Web3 / DeFi（備選，不主動優先）

### 目標公司類型
- 任何提供全遠端 + 合理薪資的公司
- AI 產品公司、SaaS 新創、Developer Tools、一般科技公司
- 不限 Web3

### 策略
- **人脈優先**：LinkedIn + 社群經營為主，冷投為輔（2-3個/週，有目標地投）
- **Coffee chat**：每週 1 個 outreach，目標公司的人 > 在國外的台灣工程師
- **面試準備並行**：算法 + 系統設計 + 前端題同步進行，不等準備好才投
- **Side project 強化履歷 + 內容**：做什麼就分享什麼，作品即內容

---

## Roadmap

| 月份 | 狀態 | 重點 |
|------|------|------|
| Month 2（4月）| 在職 | 算法 Easy→Medium + 前端面試題 + 景點清單完成 + 每天 1 投（週一到週五）|
| Month 3（5月）| 在職 → 離職（5/31）| 算法 Medium + 系統設計入門 + AI Issue Finder + 持續投遞 |
| Month 4（6月）| 離職後 | 全力求職 3-5 投/天 + 系統設計深化 + 面試衝刺 |
| Month 5（7月）| 離職後 | 面試 + offer 談判 |
| Month 6（8月）| 離職後 | 收割 |

---

## 每日優先級（5h，現階段）

| 優先 | 內容 | 時間 |
|------|------|------|
| 🥇 | LeetCode（2-3 題，Easy→Medium）| 1.5h |
| 🥇 | Side project（AI workflow / 後端知識）| 2h |
| 🥈 | 系統設計 or AI 開發課程 | 1h |
| 🥈 | LinkedIn 發文 or Coffee chat outreach（每週 1-2 篇 + 1 個 outreach）| 30min |
| 🥈 | 投履歷（2-3個/週，有目標地投）| 20min |

> 忙的時候：只留 LeetCode 1 題 + 投 1 個，其他跳過。
> 離職後（Month 4 起）投遞量升為 3-5/天，面試準備時間翻倍。

---

## Side Projects

### 1. 景點清單共享網站（本週完成目標）
朋友需求：Google Maps 沒有分享清單功能。任何人可建立主題清單、新增/刪除景點，不需登入。

**Tech stack：** Next.js + Tailwind + Supabase + Vercel

| Day | 目標 |
|-----|------|
| Day 14-15 | 初始化 + Supabase 設定 + 首頁 |
| Day 16 | 清單頁（景點列表 + 新增 + 刪除）|
| Day 17 | 收尾 + 部署 Vercel |

### 2. AI Issue Finder（Month 3 主力）
解決「怎麼找適合自己的 OSS issue」問題。輸入 GitHub repo，用 Claude API 判斷 issue 適合度，輸出推薦清單。

**Tech stack：** GitHub API + Anthropic SDK（Claude API）+ CLI

---

## 下週任務（Week 8，Day 31-35，2026-04-28 ~ 2026-05-02）

系統設計往 DB 深化走，LeetCode 繼續 Binary Search → Two Pointers。

- [ ] 系統設計：ACID 與 Transaction Isolation（Dirty Read / Phantom Read / 隔離層級實作）
- [ ] 系統設計：PostgreSQL vs MySQL 核心差異（MVCC 實作、Index 差異、JSON 支援）
- [ ] 系統設計：Index 實戰（EXPLAIN ANALYZE 解讀、Composite / Covering / Partial 實際範例）
- [ ] LeetCode：Binary Search 收尾（2-3 題 Medium）→ Two Pointers 開始
- [ ] 投履歷：每週 2-3 個（有目標地挑）

---

## 投遞紀錄

已投：10-11 個（截至 2026-04-16）
本月目標：20 個（平日每天 1 個）

---

## Week 2 回顧（2026-03-23 ~ 2026-03-31）

- 職缺調查三輪（Frontend、DevRel、前端 Worldwide Remote）
- 觀察到方向：前端遠端職缺，多投多試，不限 Web3
- 2026-03-31：工作確定兩個月後離職（預計 5/31），調整路線圖
- 新策略：算法 + 系統設計 + 前端題 + side project + 每日投遞並行
- 確認全端/Product Engineer 是之後可開放的方向（熟悉 DB + 系統設計後）

## Week 1 回顧（2026-03-18 ~ 2026-03-22）

Week 1 只有 3 天工作天，主力放在學習打底。

### ✅ 完成
- Claude Code 完整課程（Day 1-5）：Context 管理、Custom Commands、MCP、GitHub Actions、Hooks 深度實作

---

## 已完成

### Day 30（2026-04-23）
- 系統設計：Redis 實戰應用深化（6 個主題）
  - Cache Stampede：TTL 抖動、Cache Warming、Mutex Lock、Facebook Lease 機制（同時解 Thundering Herd + Stale Set）
  - Cache Penetration：空值快取、Bloom Filter、Rate Limit 與 Bloom Filter 的互補關係
  - Rate Limiting：Fixed Window（邊界攻擊問題）、Sliding Window（Sorted Set）、Token Bucket（允許爆發）
  - Session 管理：多實例共享 Session、強制登出所有裝置、Session vs JWT 比較
  - 身份驗證完整流程：JWT Access Token（15m）+ Refresh Token（7天）+ Redis，登入 / 正常請求 / 換 token / 登出四個流程
  - 分散式鎖：庫存扣減防超賣、重複付款防止（冪等鎖）、定時任務防重複執行
  - 筆記：day-30/cache-stampede.md、cache-penetration.md、rate-limiting.md、session.md、auth-flow.md、distributed-lock.md

### Day 29（2026-04-22）
- 系統設計：API 快取模式
  - Redis key 設計原則（user:123:profile、feed:user:123:page:1）
  - withCache 通用 helper（Cache-Aside 封裝）
  - HTTP Cache-Control（public / private / no-store / stale-while-revalidate）
  - ETag 條件式請求（省頻寬，不是省 DB）
  - 前端快取：SWR（revalidateOnFocus / mutate）、TanStack Query（queryKey / staleTime / invalidateQueries）、Next.js fetch cache（revalidate / revalidateTag）
  - CDN Cache vs Redis Cache：網路層自動 vs 應用層手動
  - 金融場景：餘額不快取、Write-Through + Transaction、分散式鎖讀取
  - 筆記：day-29/api-cache.md

### Day 28（2026-04-21）
- 系統設計：Storage & DB Optimization
  - 快取：MySQL Query Cache 已死（MySQL 8.0 移除）、Redis vs Memcached、CDN、Cache-Aside / Write-Through / Write-Behind
  - DB 擴展：Primary-Replica、Aurora 共用 Storage 架構、Vitess / PlanetScale Sharding、NewSQL（CockroachDB / TiDB）
  - 新概念：Connection Pooling、WAL、MVCC、ACID vs BASE、Index 策略（B-Tree / Hash / Covering / Partial）、N+1 Problem、EXPLAIN 分析
  - 新舊對比：2012 Harvard CS75 vs 現代做法、決策框架（流量規模 → 對應架構）
  - 筆記：day-28/storage-db-optimization.md
- 系統設計：Redis 深度學習
  - 原理：單執行緒 Event Loop、RAM vs 磁碟速度差異、記憶體淘汰策略（LRU/noeviction）
  - 資料結構：String / Hash / List / Set / Sorted Set / Stream 各自場景
  - Cache 模式：Cache-Aside / Write-Through / Write-Behind、TTL 設計
  - 常見問題：Cache Stampede、Cache Penetration（Bloom Filter）、Hot Key
  - 作為 DB：RDB 快照 vs AOF Append Log、兩個都開的原因
  - 高可用：Sentinel（自動 Failover）vs Cluster（Sharding，16384 slots）
  - 實際範例：Rate Limiting、分散式鎖、排行榜
  - 筆記：day-28/redis.md

### Day 26（2026-04-16）
- LeetCode 4 題（Binary Search）
- 投遞：Grafana Labs（Frontend Engineer）、Gamintec

### Day 25（2026-04-15）
- LeetCode 4 題（Stack）
- 系統設計：Load Balancer 深度學習
  - 基礎概念：Layer 4/7、分流算法、Sticky Sessions、Health Check、SPOF、HA、TCP vs UDP vs HTTP
  - 總體設計：工具比較（AWS ALB/NLB、Cloudflare、Nginx、HAProxy、Traefik、Envoy）、架構模式、程式碼
  - Case Study：GitHub GLB（Rendezvous Hashing、GUE、DPDK）、Slack（HAProxy→Envoy）、Shopify（Nginx+Lua）、Dropbox（Robinhood）、Twitter（Deterministic Aperture）、Lyft（Envoy）
- 筆記拆成三份：lb-basics.md、lb-architecture.md、lb-case-study.md

### Day 24（2026-04-14）
- 系統設計：Session 管理深度討論（Sticky Sessions vs Redis Session vs JWT vs Short-lived JWT + Refresh Token）
- 系統設計：Web Hosting & Scalability（SFTP、VPS、Elastic Scaling、現代服務商比較）
- 釐清：HTTP 無狀態 → Session 存在原因、Redis 暫存 vs DB 持久化、ORM SQL Injection 防護機制

### Day 23（2026-04-12）
- LeetCode 3 題
- 投遞：Blockaid（區塊鏈安全，符合安全監控背景）

### Day 22（2026-04-11）
- 景點清單：後端建立完成
- 研究 AI 開發 workflow

### Day 20（2026-04-09）
- LeetCode 5 題

### Day 19（2026-04-08）
- LeetCode Arrays & Hashing 補題 6 題：Contains Duplicate, Valid Anagram, Two Sum, Group Anagrams, Encode and Decode Strings, Product of Array Except Self
- HashMap 各種用法打通（HashSet、計次、存 index、排序當 key、Prefix/Postfix）
- 投遞：thingnario 慧景科技 Senior Full Stack（Hybrid，備選性質）

### Day 11（2026-03-28）
- 求職方向調整：不限 Web3，拓展到 A/B 類型公司
- 確定未來幾週節奏：side project 主力 + 職缺觀察 + 履歷整理
- 規劃兩個 side project → day-11/

### Day 7（2026-03-24）
- DevRel / Technical Writer 方向職缺調查 → day-7/job-search.md
- DevRel 入行指南整理 → day-7/devrel-guide.md

### Day 6（2026-03-23）
- 職缺深度分析：第一輪，5 個職缺 → day-6/job-search.md
- 初步學習方向整理 → day-6/learning-strategy.md

### Day 5（2026-03-22）
- Claude Agent SDK 學習

### Day 4（2026-03-21）
- Claude Code Hooks 深度實作

### Day 3（2026-03-20）
- Claude Code Hooks 學習

### Day 2（2026-03-19）
- Custom Commands、MCP、GitHub Integration

### Day 1（2026-03-18）
- Context 管理：CLAUDE.md 三層結構

---

## 職缺追蹤

| 公司 | 職位 | 薪資 | 符合度 | 狀態 |
|------|------|------|--------|------|
| Veda | Move (Sui) Smart Contract Engineer | 未標示 | ⭐⭐⭐⭐⭐ | 觀察中 |
| LI.FI | Senior Frontend Engineer (Perps) | €80K–€120K | ⭐⭐⭐⭐⭐ | 觀察中（⚠️ EMEA/LATAM 限制）|
| Blockchain.com | Frontend Engineer (React) | $89K–$109K | ⭐⭐⭐⭐ | 觀察中 |
| Tether | Developer Relations P2P (100% remote) | $126K–$138K | ⭐⭐⭐⭐⭐ | 觀察中 |
| Tether | Developer Relations AI / QVAC (100% remote) | $115K–$117K | ⭐⭐⭐⭐ | 觀察中 |
| DuckDuckGo | Senior Frontend Engineer, React/TypeScript | $178,500 | ⭐⭐⭐⭐ | 觀察中（確認台灣是否接受）|
| Phantom | Software Engineer, Frontend/Full Stack | $200K–$250K | ⭐⭐⭐ | 觀察中（⚠️ US/EU 時區限制）|
| Kraken | Frontend Engineer | $96K–$277K | ⭐⭐⭐⭐ | 觀察中（直接查 careers 頁）|
| Blockaid | （職位待補）| 未標示 | ⭐⭐⭐⭐ | 已投（區塊鏈安全，符合安全監控背景）|
| thingnario 慧景科技 | Senior Full Stack Engineer | NT$150-200萬（~$46-62k USD） | ⭐⭐ | 已投（⚠️ Hybrid，非遠端，低於薪資目標）|
| Grafana Labs | Frontend Engineer | 未標示 | ⭐⭐⭐⭐⭐ | 已投（監控系統背景直接符合，遠端）|
| Gamingtec | Frontend Developer（React/TypeScript）| 未標示 | ⭐⭐⭐⭐ | 已投（全遠端，React 18 + Redux + TS + Webpack 完全符合）|

---

## Open Source 貢獻

| Repo | 類型 | 狀態 |
|------|------|------|
| `MystenLabs/walrus` | 文件 / SDK 範例 | 待找 issue |
| `MystenLabs/sui` examples | 範例 / 文件 | 待找 issue |

> OSS 貢獻降為次要，Month 4 離職後再加強。

---

## 面試準備進度

### LeetCode
- 目標：Easy 全會 → Medium 60%+
- 目前：Arrays & Hashing 完成 + Stack 完成 + Binary Search 進行中（Day 26 做了 4 題），共約 22 題
- HashMap 各種用法已熟悉，Medium 可解
- 下一步：Binary Search 收尾 → Two Pointers
- 節奏：每天 1 題（平日）

### 系統設計
- 目標：Month 3 前掌握基礎（Cache、DB、API 設計、Load Balancer）
- 目前進度：Load Balancer ✅ → Storage & DB Optimization ✅ → Redis ✅
- 下一步：
  1. 資料庫深化（ACID Transaction Isolation、Index 實戰、PostgreSQL vs MySQL）
  2. API 設計（RESTful、Rate Limiting、Idempotency、API Gateway）
  3. 完整系統設計練習（URL Shortener → Twitter Feed → Chat 系統）
- 資源：System Design Primer（GitHub）

**學習心智圖**

```
系統設計學習路線
│
├── Load Balancer ✅（day-25）
│   ├── Layer 4 / Layer 7 差異
│   ├── 分流算法（Round Robin / Least Conn / IP Hash）
│   ├── HAProxy vs Envoy（config file vs xDS 動態更新）
│   └── Case Study：GitHub GLB / Slack / Shopify / Dropbox / Twitter / Lyft
│
├── Storage & DB Optimization ✅（day-28）
│   ├── 硬體：NVMe SSD 已是標準，雲端平台管備援
│   ├── 快取策略：Cache-Aside / Write-Through / Write-Behind
│   ├── DB 擴展：Primary-Replica → Sharding（Vitess）→ NewSQL
│   ├── 索引：B-Tree / Covering / Partial，EXPLAIN 分析
│   └── 底層：WAL / MVCC / ACID vs BASE
│
├── Redis ✅（day-28 ~ day-30）
│   ├── 原理：單執行緒 Event Loop，RAM 速度（0.01ms）
│   ├── 資料結構：String / Hash / List / Set / ZSet / Stream
│   ├── Cache 模式 + 三大問題（Stampede / Penetration / Hot Key）
│   ├── Persistence：RDB（快照）vs AOF（Append Log），生產兩個都開
│   ├── 高可用：Sentinel（Failover）vs Cluster（16384 slots）
│   ├── API 快取：withCache helper、ETag、CDN vs Redis、前端 SWR / TanStack Query
│   ├── 實戰應用：Rate Limiting（3 種算法）、Session、Auth Flow（JWT + Refresh Token）
│   └── 分散式鎖：NX EX 原子操作、庫存 / 付款 / Cron 防重複
│
├── 資料庫深化（下週）
│   ├── ACID 四特性與 Transaction Isolation Levels
│   ├── Dirty Read / Non-repeatable Read / Phantom Read
│   ├── PostgreSQL vs MySQL 核心差異
│   └── Index 實戰（EXPLAIN ANALYZE 解讀）
│
├── API 設計（Month 3）
│   ├── RESTful 設計原則（冪等性、狀態碼、版本控制）
│   ├── Rate Limiting（Token Bucket / Leaky Bucket）
│   ├── Idempotency（冪等鍵）
│   └── API Gateway（認證、限流、路由）
│
└── 完整系統設計練習（Month 3-4）
    ├── URL Shortener（入門，考 Hashing + DB 選型）
    ├── Twitter Feed（考 Fan-out、Cache、Sharding）
    ├── Chat 系統（考 WebSocket、消息隊列、已讀狀態）
    └── 面試流程：需求澄清 → 估算規模 → 架構圖 → 深挖細節
```

### 前端面試題
- 目標：覆蓋 React 核心、TypeScript、瀏覽器原理、效能優化
- 目前：未系統整理
- 節奏：每天 1 個主題

---

## 社群 / 人脈計畫

### 方向
長期經營，不急，分享生活就好。做什麼就記錄什麼。

### 平台
- **LinkedIn**（主力）：文字 + 貼文，求職人脈最直接
- **X / Threads**（次要）：輕量隨手發，不強求

### 內容方向
- 轉職過程（在職備戰、離職倒數）
- LeetCode / 系統設計學習筆記
- Side project 進度（做什麼分享什麼）
- 遠端求職觀察

### 節奏
- LinkedIn：每週 1-2 篇，沒靈感就跳過，不強迫
- Coffee chat outreach：每週 1 個，優先目標公司的人 > 在國外的台灣工程師
- 長期複利，不追短期數字

---

## 筆記 / 方向調整

- Month 1（3/18-3/31）：打底。Claude Code 課程 + 職缺調查三輪
- 2026-03-25：調整。Month 1 以驗證方向為主，不衝數字
- 2026-03-28：調整。不限 Web3，side project 主力
- 2026-03-31：重大調整。工作 2 個月後離職（預計 5/31）。新路線：算法 + 系統設計 + 前端題 + side project + 每日投遞並行。方向：前端遠端為主，之後開放全端/Product Engineer。
