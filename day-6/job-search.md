# Job Search Deep Analysis — 2026-03-23

> 本次分析針對 Wei-Chun Lin 的背景（3 年 Web3/DeFi fullstack，Sui/Move，React/TypeScript），
> 挑選 5 個最適合的職缺做深度拆解。

---

## 評分總覽

| # | 職缺 | 公司 | 薪資 | 整體適配 | 難度 |
|---|------|------|------|----------|------|
| 1 | Move (Sui) Smart Contract Engineer | Veda | 未標示 | ⭐⭐⭐⭐⭐ | 中 |
| 2 | Senior Frontend Engineer (Perps) | LI.FI | €80K–€120K | ⭐⭐⭐⭐⭐ | 中高 |
| 3 | Senior DevRel Engineer, Walrus (APAC) | Sui Foundation | 未標示 | ⭐⭐⭐⭐ | 高 |
| 4 | Frontend Engineer (React) | Blockchain.com | $89K–$109K | ⭐⭐⭐⭐ | 中 |
| 5 | Move (Sui) Engineer | MLabs | 未標示 | ⭐⭐⭐⭐ | 中 |

---

## 1. Veda — Move (Sui) Smart Contract Engineer

🔗 https://jobs.sui.io/companies/veda-2/jobs/50781716-move-sui-smart-contract-engineer-remote-full-time

**公司背景：** Veda 做 DeFi lending / stablecoin，在 Sui 上建構 protocol。與 Scallop（Wei-Chun 現職）業務幾乎完全重疊。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| Sui Move — object model、capability pattern、shared/owned object | ★★★★★ |
| DeFi protocol 設計（lending、liquidation、oracle、interest rate model） | ★★★★★ |
| Move 安全性（reentrancy 預防、overflow、access control） | ★★★★ |
| Sui SDK / PTB（Programmable Transaction Block） | ★★★★ |
| stablecoin 機制理解（CDP、pegging mechanism、stability fee） | ★★★★ |
| indexer / event parsing（後端 on-chain data 讀取） | ★★★ |

### 我已具備的

- ✅ Sui Move 實務經驗（Scallop lending protocol）
- ✅ DeFi lending 架構：liquidation、oracle、risk control
- ✅ indexer / bot 建構
- ✅ React/TypeScript frontend 整合 smart contract

### 我缺乏的（誠實分析）

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| Stablecoin 機制設計 | CDP、over-collateralization、stability fee 細節 | 🟡 中等（2–3 週可補） |
| Move 進階安全 pattern | formal verification、MystenLabs audit 標準 | 🔴 高（需要 1–2 月深入） |
| PTB 複雜組合交易 | multi-step atomic transaction in Move | 🟡 中等（1–2 週） |
| GitHub public code | 沒有公開 Sui Move 貢獻，面試很難驗證 | 🔴 重要缺口（需建立） |

### 面試最難的部分

- **技術題**：設計一個 over-collateralized lending pool 的 Move contract，包含 liquidation threshold、oracle price feed 整合
- **安全題**：找出某段 Move 合約中的漏洞（reentrancy 等同物件在 Sui 中如何表現）
- **設計題**：如何在 Sui 上設計 stability mechanism，防止 depeg

### 如何透過貢獻找到這份工作 ✅ 強烈建議

Veda 是 Sui 上的 DeFi 協議，很可能有 public repo 或 grant 項目。

**路線：**
1. 在 GitHub 找 Veda 的 open source 合約（或 Sui Foundation grant 項目）
2. 提 PR（fix bug、優化 gas、補 test）
3. 在 Sui Discord 的 #developer 頻道討論 Move 問題並解答他人問題
4. 直接 cold DM Veda 的工程師說「我做過 Scallop lending，對你們的設計很有興趣」

### 學習路線圖

```
Week 1–2: Stablecoin 機制
  - 讀 MakerDAO/crvUSD whitepaper 理解 CDP 設計
  - 用 Sui Move 實作一個簡單的 CDP 合約
  - 推上 GitHub

Week 3–4: PTB 與進階 Move
  - 看 Sui Foundation 官方 PTB 文件
  - 實作 multi-step DeFi 流程（borrow → swap → repay）

Month 2: Move 安全
  - 讀 OtterSec / Sec3 的 Move audit reports
  - 對自己的合約做 manual security review
  - 寫一篇技術文章：「Sui Move vs Solidity 的安全模型差異」
```

---

## 2. LI.FI — Senior Frontend Engineer (Perps)

🔗 https://jobs.solana.com/companies/li-fi-2/jobs/71338589-senior-frontend-engineer-perps

**⚠️ 地點限制：EMEA 或 LATAM 優先，台灣屬 APAC。建議投遞時在 cover letter 說明可接受 EMEA timezone overlap（UTC+1 to +3，台灣 GMT+8 可接受）。**

**公司背景：** LI.FI 是知名 cross-chain bridge aggregator，目前擴展 Perps 功能（整合 Hyperliquid、Lighter 等 DEX 的永續合約）。歐洲公司，WLB 文化好，30 天 PTO、equity。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| React + TypeScript（進階，架構級） | ★★★★★ |
| viem / wagmi（Web3 前端標準函式庫） | ★★★★★ |
| 永續合約 / Perp DEX 概念（Hyperliquid、orderbook） | ★★★★★ |
| Cross-chain bridging UX 設計（bridging flow、routing） | ★★★★ |
| WebSocket / real-time data（orderbook、price feed） | ★★★★ |
| TanStack Query / Zustand（state management） | ★★★ |
| Vitest / testing frontend | ★★★ |
| Figma 到 code 的精準還原 | ★★★ |

### 已具備的

- ✅ React + TypeScript（3 年實務）
- ✅ Tailwind（stack 完全吻合）
- ✅ Node.js / API 整合
- ✅ DeFi 產品 frontend 經驗（Scallop UI）
- ✅ blockchain interaction（雖然是 Sui 非 EVM，但概念通用）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| viem / wagmi | Sui 用的是 Sui SDK，EVM 這套沒用過 | 🟡 中等（1–2 週） |
| Perp DEX 深度理解 | funding rate、mark price、orderbook mechanics | 🟡 中等（1–2 週啃文件） |
| Cross-chain bridging 實作 | 沒有 cross-chain bridge 開發經驗 | 🟠 中高（3–4 週） |
| 年資缺口 | 要求 6+ 年，目前約 3 年 | 🔴 無法硬補，靠作品彌補 |
| WebSocket real-time UI | 基本理解但沒有大規模 orderbook 實作 | 🟡 中等 |

### 面試最難的部分

1. **Take-home assignment**（LI.FI 流程明確有這關）：
   - 很可能是「實作一個 cross-chain swap UI」或「實作 Hyperliquid orderbook 顯示元件」
   - 評分重點：架構是否清晰、error state 處理、performance（virtualized list）

2. **系統設計**：如何設計 bridging flow 的 UX，處理 pending / failed / re-org 狀態

3. **Tech screen**：viem/wagmi 使用方式、React performance 優化

### 如何透過貢獻找到這份工作

LI.FI 有完整的 open source SDK 與 widget：https://github.com/lifinance

**路線：**
1. Clone `lifi-widget` 或 `sdk`，找 good-first-issue 或 bug
2. 提 PR 並在 Discord 討論
3. 投遞時主動提「我有貢獻過 lifi-widget，修了 #XXX」

### 學習路線圖

```
Week 1: viem + wagmi
  - 用 wagmi 做一個 EVM wallet connect + 轉帳 demo
  - 部署在 Vercel 並放上 GitHub

Week 2: Perp DEX 概念
  - 讀 Hyperliquid 文件，理解 funding rate、mark price、index price
  - 用 Hyperliquid API 實作一個 real-time orderbook（WebSocket）

Week 3–4: Cross-chain bridging
  - 用 LI.FI SDK 實作一個 swap + bridge UI
  - 加入 error handling：slippage、bridge failed、re-org

Month 2: 貢獻 lifi-widget
  - 找一個已知 issue 修復並提 PR
```

---

## 3. Sui Foundation — Senior DevRel Engineer, Walrus (APAC)

🔗 https://hk.linkedin.com/jobs/view/senior-developer-relations-engineer-walrus-remote-apac-at-sui-foundation-4204340765

**⚠️ 注意：這是一個對 Wei-Chun 來說「夢幻但有門檻」的職位。要求 7+ 年，但 APAC + Sui/Move 背景讓 Wei-Chun 是稀缺候選人。**

**Walrus 專注：** 這個職缺特別針對 Walrus（Sui 上的去中心化儲存協議），需要了解 blob storage、IPFS、Arweave 等。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| Sui Move 技術深度（能解答社群問題） | ★★★★★ |
| 公開技術寫作（文章、tutorial、doc） | ★★★★★ |
| 去中心化儲存協議（IPFS、Arweave、Walrus/blob storage） | ★★★★ |
| 工作坊設計與演講 | ★★★★ |
| 社群管理 / Discord / Twitter 技術互動 | ★★★ |
| 多語言溝通（中英文優勢明顯） | ★★★ |

### 已具備的

- ✅ Sui Move 實務背景（市場極稀缺）
- ✅ 中英文都流利（APAC 市場加分）
- ✅ DeFi protocol 技術深度
- ✅ AI 工具重度使用（現代 DevRel 加分）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| 正式 DevRel 經驗 | 沒有 official DevRel 頭銜 | 🔴 需要用作品彌補 |
| 年資（7+ vs 3年） | 差距很大，但 Sui/Move 稀缺性可以彌補 | 🔴 只能靠強作品集 |
| 去中心化儲存知識 | Walrus/IPFS/Arweave 需要從頭學 | 🟡 中等（2–3 週） |
| 公開技術內容 | 目前沒有 blog / Twitter 技術貼文 | 🔴 短期內需要建立 |
| 演講 / 工作坊經驗 | 沒有公開演講記錄 | 🟠 中高難度 |

### 面試最難的部分

1. **Live demo / 工作坊 mock**：要你現場對「假設的開發者社群」解釋 Walrus 如何使用
2. **技術深度問答**：Sui 的 object model、Walrus blob storage 機制
3. **內容樣本**：幾乎必要求看你寫過的 tutorial 或技術文章

### 如何透過貢獻找到這份工作 ✅ 這個職缺最適合用貢獻打入

**路線（最有效）：**
1. 在 Sui developer forum / Discord 用中文和英文回答開發者問題（每週 5 則）
2. 寫 1–2 篇 Walrus 上手 tutorial（掛在 Medium 或個人 blog）
3. 建立一個用 Walrus 儲存資料的小 dApp demo，放上 GitHub
4. 在 Twitter/X 發技術貼文：「用 Sui Move 做 DeFi 的 X 個陷阱」系列
5. 申請 Sui Foundation 的 DevRel 貢獻者 / Ambassador 計畫

### 學習路線圖

```
Week 1–2: Walrus 深度
  - 讀 Walrus 白皮書與官方文件
  - 用 Walrus SDK 儲存並讀取檔案
  - 實作：把 Sui Move 合約的 metadata 存到 Walrus

Week 3–4: 建立公開內容
  - 寫第一篇英文技術文章：「Build your first Walrus-backed dApp on Sui」
  - 發布到 Medium / dev.to
  - 在 Sui Discord 分享並回應社群

Month 2: 建立 DevRel 作品集
  - 做一個 Walrus + Sui 的完整 demo project
  - 錄一個 5 分鐘技術 walkthrough video
  - 在 Twitter/X 開始技術系列貼文
```

---

## 4. Blockchain.com — Frontend Engineer (React)

🔗 https://web3.career/front-end-engineer-react-blockchain/141795

**公司背景：** Blockchain.com 是老牌加密貨幣公司（wallet、exchange），非 DeFi protocol，但 crypto 背景。全遠端，$89K–$109K。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| React / TypeScript（中高級） | ★★★★★ |
| JavaScript（ES6+，async patterns） | ★★★★★ |
| RESTful API / GraphQL 整合 | ★★★★ |
| Testing（Jest / Cypress） | ★★★★ |
| Blockchain 基本概念（wallet、tx、UTXO 或 EVM） | ★★★ |
| CSS / Tailwind / styled-components | ★★★ |

### 已具備的

- ✅ React + TypeScript（技術棧完全對應）
- ✅ blockchain 實務背景（超越一般 frontend engineer）
- ✅ API 整合、Node.js

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| EVM / Bitcoin UTXO 基礎 | 目前只有 Sui 背景，Blockchain.com 主要做 Bitcoin/ETH | 🟡 中等（1 週） |
| Testing 文化 / coverage | 不確定 Scallop 的測試覆蓋率，需要展示 | 🟡 中等 |
| 可展示的公開作品 | GitHub 沒有公開 React 作品 | 🔴 需要補 |

### 面試最難的部分

1. **系統設計**：設計一個 multi-chain wallet dashboard
2. **React 深度**：concurrent mode、Suspense、memo/callback 優化
3. **演算法**：LeetCode medium 級別（一般大公司標配）

### 如何透過貢獻找到這份工作

Blockchain.com 有部分 open source：https://github.com/blockchain

- 找他們的 frontend packages 提 PR（小 bug fix 或文件改進）
- 這個職缺貢獻路線相對弱，主要靠作品集和履歷

### 學習路線圖

```
Week 1: 補齊 Bitcoin/EVM 基礎
  - 讀 Bitcoin UTXO model（與 Sui Object model 對比理解）
  - 用 ethers.js 實作一個 ETH wallet balance 查詢

Week 2: 作品集
  - 建一個 multi-chain portfolio dashboard（Sui + ETH + BTC）
  - 強調 React 架構、TypeScript strict mode、測試覆蓋率
  - 部署到 Vercel，放上 GitHub README

Week 3: 面試準備
  - 練習 React 優化題（何時用 memo、useCallback）
  - LeetCode medium 10 題（array、string、hashmap 為主）
```

---

## 5. MLabs — Move (Sui) Engineer

🔗 https://himalayas.app/companies/mlabs/jobs/move-sui-engineer

**公司背景：** MLabs 是 Web3 開發顧問公司，正在建構 Sui 上的 stablecoin 協議（CDP 模型）。全遠端，但需要 4 小時 EST overlap（台灣 GMT+8，EST 是 -5，最大 overlap 約晚上 9pm–1am，或早班方案）。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| Sui Move（中高級） | ★★★★★ |
| Stablecoin / CDP 設計 | ★★★★★ |
| On-chain oracle 整合 | ★★★★ |
| Smart contract security mindset | ★★★★ |
| Sui SDK / 前端整合 | ★★★ |

### 已具備的

- ✅ Sui Move 實務（高度符合）
- ✅ Oracle integration（Scallop 有做過）
- ✅ DeFi protocol 設計經驗

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| Stablecoin CDP 設計 | 與 Veda 相同缺口 | 🟡 中等 |
| EST timezone overlap | 台灣時間約晚上 9pm 開始工作 | 🟠 生活品質影響大 |
| 顧問公司節奏 | MLabs 是外包/顧問型，工作節奏與 protocol 公司不同 | ⚠️ 需要評估 WLB |

### 面試最難的部分

1. **Live coding**：用 Move 實作 collateral vault 的 deposit/withdraw/liquidate
2. **設計題**：如何設計 stability fee（借款利率）機制
3. **Timezone 測試**：可能直接用 EST 時間開會，測試你的 availability

### 如何透過貢獻找到這份工作

MLabs 有 GitHub 組織，可以找相關 Sui 專案提 PR。
- 或直接 cold email / DM：「我做過 Scallop lending，對你們的 stablecoin 設計很有興趣，可以聊聊嗎？」

### 學習路線圖

（與 Veda 相同，可合併學習）

```
與 Veda 學習計畫共用：
- CDP stablecoin 設計
- Move 安全 pattern
- Public Move 合約作品集
```

---

## 綜合行動計畫（優先順序）

### 短期（1–2 週，現在就能做）

1. **GitHub 補作品**（所有職缺共同需求）
   - 建立一個公開 Sui Move 合約 repo（CDP 簡易版或 lending 工具）
   - 建一個 React + viem/wagmi 的 EVM demo app

2. **投遞 Veda Move (Sui) Engineer**（技術最符合，不需要額外補課）
   - Cover letter 強調：Scallop lending 經驗 = Veda 的核心需求

3. **投遞 Blockchain.com Frontend**（週薪最穩，風險最低）

### 中期（3–4 週）

4. **學 viem/wagmi + Perp DEX 概念** → 投遞 LI.FI（注意地點限制）

5. **開始技術寫作** → 為 Sui Foundation DevRel 鋪路

### 長期（1–2 月）

6. **建立 DevRel 作品集** → Sui Foundation 職缺
7. **Stablecoin CDP 實作** → Veda / MLabs 後續面試強化

---

## 技能缺口速查表

| 技能 | 重要度 | 學習難度 | 建議學習時間 |
|------|--------|----------|-------------|
| Stablecoin CDP 設計 | 高（Veda/MLabs） | 🟡 中 | 2–3 週 |
| viem / wagmi | 高（LI.FI） | 🟡 中 | 1–2 週 |
| Perp DEX / Hyperliquid | 高（LI.FI） | 🟡 中 | 1–2 週 |
| Walrus / 去中心化儲存 | 高（Sui DevRel） | 🟡 中 | 2–3 週 |
| Move 進階安全 pattern | 中高（Veda/MLabs） | 🔴 高 | 1–2 月 |
| 公開技術文章 / blog | 高（DevRel 必備） | 🟠 中高 | 持續建立 |
| GitHub 公開作品集 | 超高（所有職缺） | 🟡 中 | 立刻開始 |
| LeetCode / 演算法 | 中（Blockchain.com） | 🟡 中 | 2–3 週刷題 |

---

*生成日期：2026-03-23 | 下次更新建議：投遞後記錄狀態到 PROGRESS.md*
