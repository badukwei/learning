# 求職進度追蹤

更新：2026-04-16 ｜ Month 2，Day 26

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
- **多投多面試**：平日每天 1 投，月底前累積 20+ 投遞紀錄
- **面試準備並行**：算法 + 系統設計 + 前端題同步進行，不等準備好才投
- **Side project 強化履歷**：有作品比等完美更重要

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
| 🥈 | 投履歷（1 個全遠端，週一到週五）| 30min |

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

## 本週任務（Week 3，Day 14-17）

- [ ] 景點清單：初始化 + Supabase 設定 + 首頁（Day 14-15）
- [ ] 景點清單：清單頁功能（Day 16）
- [ ] 景點清單：部署 Vercel（Day 17）
- [ ] LeetCode：每天 1 題 Easy（建立節奏）
- [ ] 前端面試題：每天 1 個主題（e.g. Virtual DOM, useEffect, closure）
- [ ] 投履歷：每個工作天 1 個（本週 3 天 = 3 個，加上已投 2 個 = 共 5 個）

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
- 目標：Month 3 前掌握基礎（DB 索引、Cache、API 設計、Load Balancer）
- 目前：Load Balancer 完成（基礎概念 + 架構 + Case Study）
- 下一步：
  1. 繼續閱讀其他 LB case study（Uber、Dropbox Bandaid 細節等）
  2. 資料庫（DB 索引、Sharding、Replication、ACID）
- 資源：System Design Primer（GitHub）

### 前端面試題
- 目標：覆蓋 React 核心、TypeScript、瀏覽器原理、效能優化
- 目前：未系統整理
- 節奏：每天 1 個主題

---

## 自媒體計畫

- 平台：TikTok + Instagram（影片）、LinkedIn + X + Threads（文字）
- 頻率：一週兩次，同一內容跨發，不特別維護
- 內容：學習日記、side project 進度、求職觀察
- 啟動時間：2026-04 開始

---

## 筆記 / 方向調整

- Month 1（3/18-3/31）：打底。Claude Code 課程 + 職缺調查三輪
- 2026-03-25：調整。Month 1 以驗證方向為主，不衝數字
- 2026-03-28：調整。不限 Web3，side project 主力
- 2026-03-31：重大調整。工作 2 個月後離職（預計 5/31）。新路線：算法 + 系統設計 + 前端題 + side project + 每日投遞並行。方向：前端遠端為主，之後開放全端/Product Engineer。
