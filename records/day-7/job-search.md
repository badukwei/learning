# Job Search Deep Analysis — 2026-03-24

> 本次分析聚焦 **DevRel / Developer Advocate** 方向。
> 針對 Wei-Chun Lin 的背景（3 年 Web3/DeFi fullstack，Sui/Move，AI workflow 重度使用者，中英文雙語），
> 挑選 5 個最適合的職缺做深度拆解。

---

## 評分總覽

| # | 職缺 | 公司 | 薪資 | 整體適配 | 難度 |
|---|------|------|------|----------|------|
| 1 | Developer Relations P2P（100% remote） | Tether | $126k–$138k | ⭐⭐⭐⭐⭐ | 中 |
| 2 | Developer Relations AI / QVAC（100% remote） | Tether | $115k–$117k | ⭐⭐⭐⭐ | 中高 |
| 3 | Senior DevRel Engineer, Walrus（APAC） | Sui Foundation | 未標示 | ⭐⭐⭐⭐ | 高 |
| 4 | Senior Developer Relations Engineer（US） | Crossmint | $140k–$175k | ⭐⭐⭐ | 高 |
| 5 | Senior Developer Relations Engineer | Mysten Labs | 未標示 | ⭐⭐⭐⭐ | 高 |

---

## 1. Tether — Developer Relations P2P（100% remote Worldwide）

🔗 https://web3.career/developer-relations-p2p-100-remote-worldwide-tether/146913

**公司背景：** Tether 是全球最大穩定幣（USDT）公司，P2P 方向是他們的 Holepunch/Keet 產品線——去中心化 P2P 通訊協議（基於 BitTorrent DHT）。職位是為這條產品線建立開發者社群與生態。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| JavaScript / TypeScript / Node.js（能寫 production code） | ★★★★★ |
| 技術內容創作（tutorial、blog、sample app、文件） | ★★★★★ |
| DevRel / 開發者社群建立 | ★★★★★ |
| 公開演講 / 工作坊 / webinar | ★★★★ |
| API / SDK 開發與整合 | ★★★★ |
| GitHub / Discord / 社群管理 | ★★★ |
| P2P 協議概念（DHT、BitTorrent、去中心化通訊） | ★★★ |

### 已具備的

- ✅ TypeScript / Node.js（3 年實務）
- ✅ API / SDK 整合（Scallop DeFi protocol）
- ✅ Web3 / blockchain 技術深度（超越一般 DevRel 候選人）
- ✅ 中英文雙語（APAC 市場加分）
- ✅ AI 工具重度使用（現代 DevRel 明確加分項）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| 正式 DevRel 頭銜 | 沒有 official Developer Advocate / Relations 職稱 | 🔴 需要作品集彌補 |
| P2P 協議深度（Holepunch） | Tether 的 P2P 產品用 DHT/BitTorrent，與 Sui 背景不同 | 🟡 中等（1–2 週上手） |
| 公開技術文章 | 目前無可展示的 blog / tutorial | 🔴 短期內需要建立 |
| 公開演講紀錄 | 沒有 conference talk / 工作坊主持記錄 | 🟠 中高（需要時間累積） |
| 測試 / Demo app | 沒有公開可展示的 demo project | 🔴 需要立刻開始做 |

### 面試最難的部分

1. **Content sample**：幾乎肯定要求看你寫過的 tutorial 或技術文章
2. **Live demo / coding session**：要你現場示範如何用 Holepunch SDK 建一個去中心化 app
3. **Past DevRel impact**：「你做過什麼社群建立的成果？用數字說明」

### 如何透過貢獻找到這份工作 ✅ 優先策略

Holepunch 是開源的：https://github.com/holepunchto

**路線：**
1. 讀 Holepunch SDK 文件，用它做一個小 demo（例如去中心化 chat）
2. 在 GitHub 提 PR：補文件、修錯誤範例、新增 JavaScript 範例
3. 發布一篇英文 tutorial：「Build a peer-to-peer chat app with Holepunch」到 Medium / dev.to
4. 把這篇文章主動分享給 Tether 的 DevRel 相關人員（LinkedIn / Twitter）

### 學習路線圖

```
Week 1: Holepunch 上手
  - 讀 Hypercore / Autobase / Keet 文件
  - 用 Holepunch SDK 做一個 P2P file sync demo
  - 推上 GitHub，寫清楚 README

Week 2: 建立內容
  - 寫第一篇技術文章：「P2P vs. Blockchain storage：Holepunch 的設計哲學」
  - 發布到 Medium + dev.to

Week 3–4: 投遞前準備
  - 確認有 1 個 public demo repo + 1 篇文章
  - Cover letter 強調：Sui DeFi 背景 + AI 工具 + 中文市場優勢
```

---

## 2. Tether — Developer Relations AI / QVAC（100% remote Worldwide）

🔗 https://web3.career/developer-relations-ai-100-remote-worldwide-tether/146917

**公司背景：** 同一個 Tether，但這條是 QVAC 產品線——本地端隱私 AI 平台，設計上不依賴雲端，所有 AI 運算在本機執行。類似 Ollama + 隱私保護的方向。需要懂 Llama.cpp / ggml。

**⚠️ 注意：這個職缺要求 Llama.cpp / ggml 實務經驗，比 P2P 那個更硬。但 Wei-Chun 的 Claude Code / Cursor / AI workflow 背景是加分項。**

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| JavaScript / TypeScript / Node.js | ★★★★★ |
| Llama.cpp / ggml inference engine（本地 AI 執行） | ★★★★★ |
| LLM / chatbot / AI SDK 整合 | ★★★★★ |
| 技術內容創作 | ★★★★★ |
| DevRel / 社群建立 | ★★★★ |
| AI 工具使用（Cursor / Codex / 類似工具） | ★★★★ |
| GPU 架構理解（ggml 的 quantization、model deployment） | ★★★ |

### 已具備的

- ✅ TypeScript / Node.js
- ✅ AI 工具重度使用（Claude Code、Cursor → 直接符合 preferred 條件）
- ✅ LLM 基本使用與 API 整合概念
- ✅ Web3 / DeFi 背景（在 AI DevRel 中是稀缺加分）
- ✅ 中英文雙語

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| Llama.cpp / ggml 實作 | 沒有本地 LLM deployment 經驗 | 🟠 中高（2–3 週） |
| GPU 架構 / quantization | GGUF、Q4_K_M 等量化格式，GPU backend 選擇 | 🔴 高（需要補 ML infra 知識） |
| 本地 AI app 實作 | 沒有用 Llama.cpp 建過完整 app | 🟡 中等（1–2 週可做出 demo） |
| 公開技術內容 | 同 P2P 角色 | 🔴 需要建立 |

### 面試最難的部分

1. **技術深度**：「用 ggml 部署一個 quantized Llama 3 模型，解釋 Q4_K_M 是什麼」
2. **Demo**：展示一個用 Llama.cpp 建的本地端 AI app
3. **AI 工具使用方式**：「你怎麼把 AI 工具整合到你的 DevRel 工作流程？」（這題 Wei-Chun 可以答很好）

### 如何透過貢獻找到這份工作

**路線：**
1. 安裝並執行 Llama.cpp，跑幾個模型，理解 GGUF 格式
2. 建一個 QVAC 導向的 demo：「本地端隱私 AI + 區塊鏈存證」
3. 寫文章：「為什麼 local-first AI 對 Web3 用戶更安全？」
4. 在 Tether 的 developer channels 參與討論

### 學習路線圖

```
Week 1: Llama.cpp 上手
  - 安裝 Llama.cpp，執行 Llama 3.1 8B（GGUF 格式）
  - 了解 quantization（Q4_K_M vs Q8_0 差異）
  - 用 llama.cpp server + TypeScript client 做一個 chat demo

Week 2: QVAC / 本地 AI app
  - 讀 QVAC 文件（https://qvac.ai）
  - 建一個本地端隱私 AI app（不送 API call 給雲端）
  - 強調：blockchain 身份 + 本地 AI 結合場景

Week 3: 內容
  - 寫技術文章：「Local-first AI + Blockchain：Web3 的隱私未來」
  - 發布到 Medium / dev.to，同步 Twitter/X
```

---

## 3. Sui Foundation — Senior DevRel Engineer, Walrus（APAC）

🔗 https://jobs.sui.io/companies/sui-foundation-2/jobs/48570217-senior-developer-relations-engineer-walrus-remote-apac

**（此職缺在 Day 6 追蹤清單已有，這裡做 DevRel 專項分析）**

**⚠️ 最大缺口：要求 7+ 年，Wei-Chun 約 3 年。但 APAC + Sui/Move 稀缺性是重要差異化優勢。**

**Walrus 專注：** 這個職缺針對 Walrus（Sui 上的去中心化 blob storage）。需要懂 IPFS、Arweave、Filecoin 等去中心化儲存概念。

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| Sui Move 技術深度（能解答社群問題） | ★★★★★ |
| 公開技術寫作（英文 tutorial、doc、blog） | ★★★★★ |
| 去中心化儲存（Walrus、IPFS、Arweave、blob storage） | ★★★★★ |
| 工作坊設計與演講（線上線下） | ★★★★ |
| 社群管理（Discord、Telegram、Twitter/X） | ★★★★ |
| APAC 市場語言優勢（中文、日文、韓文加分） | ★★★ |
| Move / Rust 結合的 SDK 使用 | ★★★ |

### 已具備的

- ✅ Sui Move 實務（市場極稀缺，直接碾壓大多數候選人）
- ✅ 中英文雙語（APAC 加分，可覆蓋台灣、中國、東南亞開發者社群）
- ✅ DeFi protocol 技術深度（能回答進階問題）
- ✅ AI 工具重度使用者（現代 DevRel 明確加分）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| 年資（7+ vs 3 年） | 差距大，但 Sui 稀缺性可部分彌補 | 🔴 無法硬補，靠作品說話 |
| Walrus / 去中心化儲存 | 需要從頭學 Walrus SDK 和設計哲學 | 🟡 中等（2–3 週） |
| 正式 DevRel 頭銜 | 無 official DevRel 職稱 | 🔴 靠作品集彌補 |
| 公開技術文章 / blog | 目前沒有英文技術文章 | 🔴 需要立刻開始建立 |
| 工作坊 / 演講紀錄 | 無公開演講記錄 | 🟠 需要時間累積 |

### 面試最難的部分

1. **Mock 工作坊**：要你現場對「假設的 developer 社群」介紹 Walrus 如何使用
2. **技術深度問答**：Walrus 的 RedStuff erasure coding 原理、與 IPFS 的設計差異
3. **Content sample**：幾乎必定要求展示你寫過的英文 tutorial 或文件貢獻

### 如何透過貢獻找到這份工作 ✅ 這個職缺最值得用貢獻打入

**路線（優先順序）：**
1. 在 `MystenLabs/walrus-docs` 提 PR（補錯誤、補範例、補翻譯）
2. 建一個 Walrus 上手 demo：用 Walrus 儲存 NFT metadata，附完整 README
3. 寫一篇英文技術文章：「Store your Sui dApp data on Walrus — complete tutorial」
4. 在 Sui Developer Forum 回答 Walrus 相關問題（建立知識貢獻記錄）
5. 申請 Sui Foundation Ambassador 計畫（正式管道）

### 學習路線圖

```
Week 1–2: Walrus 深度
  - 讀 Walrus 白皮書（理解 RedStuff erasure coding）
  - 用 Walrus CLI + SDK 儲存並讀取 blob
  - 對比 IPFS / Arweave / Filecoin 的設計哲學

Week 3–4: 建立公開內容
  - Demo：把 Sui NFT 的 metadata 和圖片存到 Walrus（而非 IPFS）
  - 寫英文 tutorial 發布到 dev.to / Medium
  - 在 Sui Discord 分享並主動回應社群問題

Month 2: 建立 DevRel 作品集
  - 做 1 個完整 Walrus + Sui 整合 dApp
  - 錄一支 5 分鐘 walkthrough video（YouTube）
  - 在 Twitter/X 開始 Sui 技術系列貼文
```

---

## 4. Crossmint — Senior Developer Relations Engineer（US）

🔗 https://web3.career/senior-developer-relations-engineer-us-crossmint/147250

**公司背景：** Crossmint 是企業級 stablecoin + 智能錢包基礎設施，已獲 Ribbit Capital / Franklin Templeton 等投資，$23.6M raised in 2025。2026 年獲得歐盟 MiCA 授權。40,000+ 客戶，包含 MoneyGram、WireX。

**⚠️ 地點限制：「Flexible within the US. Miami, SF, or NY preferred.」這是 hybrid 職缺，非全遠端。台灣候選人需要在 cover letter 特別說明。**

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| TypeScript / JavaScript（能 ship production code） | ★★★★★ |
| API / SDK 整合（wallet、stablecoin、cross-chain） | ★★★★★ |
| 技術文件寫作（tutorial、sample app、starter kit） | ★★★★★ |
| DevRel / 社群建立（4+ 年）| ★★★★★ |
| Web3 / blockchain 基礎（stablecoin、wallets、smart contracts） | ★★★★ |
| 工作坊 / hackathon 主持 | ★★★★ |
| 跨部門合作（product、engineering、marketing） | ★★★ |

### 已具備的

- ✅ TypeScript / JavaScript（3 年實務）
- ✅ API / SDK 整合（Scallop DeFi protocol）
- ✅ Web3 / blockchain 實務（超越一般候選人）
- ✅ AI 工具使用（Crossmint 明確鼓勵 AI workflow，這是加分項）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| 正式 DevRel 年資（4+ 年要求） | 沒有 DevRel 頭銜 | 🔴 靠作品彌補 |
| 美國地點偏好 | Hybrid，US 優先，台灣需特別說明 | 🔴 地理限制為主要障礙 |
| Stablecoin infra 深度 | Crossmint 做 USDT/USDC 基礎設施，需要了解 on/offramp、cross-chain stablecoin | 🟡 中等（1–2 週） |
| Wallet integration（EVM）| MetaMask / WalletConnect 整合（Sui 生態沒有這個） | 🟡 中等（1 週） |
| 公開 DevRel 作品集 | 無可展示的 tutorial / demo / starter kit | 🔴 需要建立 |

### 面試最難的部分

1. **Technical take-home**：幾乎確定有，可能是「用 Crossmint API 建一個 embedded wallet demo」
2. **Past DevRel impact**：「你之前如何衡量 DevRel 成功？用數字說明」
3. **Stablecoin infra 知識**：USDC mint/burn 流程、cross-chain bridge 如何影響 UX

### 如何透過貢獻找到這份工作

Crossmint 有 public SDK 和文件：https://github.com/Crossmint

**路線：**
1. 用 Crossmint Wallets API 建一個 demo：embedded wallet + stablecoin 轉帳
2. 在 GitHub 找 issues 或文件補充機會，提 PR
3. 投遞時提「我做過 Scallop DeFi，了解 wallet UX 痛點，這是我用你們 API 做的 demo」

### 學習路線圖

```
Week 1: Crossmint API 上手
  - 用 Crossmint API 建一個 embedded wallet + NFT mint demo
  - 了解 stablecoin on/offramp 流程（USDC → 法幣）

Week 2: 建立 starter kit
  - 做一個「Next.js + Crossmint + Stablecoin payment」的 starter template
  - 推上 GitHub，寫詳細 README

投遞時機：Month 3（需要先建立 DevRel 作品集）
```

---

## 5. Mysten Labs — Senior Developer Relations Engineer（Sui）

🔗 https://jobs.sui.io/companies/mysten-labs/jobs/56440714-senior-developer-relations-engineer

**公司背景：** Mysten Labs 是 Sui 區塊鏈的創建公司（前 Meta Diem 工程師創辦），也是 Walrus 協議的開發者。這是 Sui 生態的核心公司——比 Sui Foundation 更偏工程側，DevRel 需要技術能力更強。

**核心差異：Sui Foundation 的 DevRel 偏生態建立，Mysten Labs 的 DevRel 偏技術深度 + 開發者工具推廣。**

### 需要的技能

| 技能 | 重要度 |
|------|--------|
| Sui Move 深度（能 debug 社群問題、解答 object model 問題） | ★★★★★ |
| Sui SDK（TypeScript SDK、Wallet Standard） | ★★★★★ |
| 技術文件 / 範例程式碼 | ★★★★★ |
| 公開技術溝通（英文，APAC 加分） | ★★★★ |
| Walrus / blob storage 概念 | ★★★★ |
| PTB（Programmable Transaction Block）複雜使用 | ★★★★ |
| open source 貢獻紀錄 | ★★★ |

### 已具備的

- ✅ Sui Move 實務（最強的優勢，Mysten Labs 自己的核心技術）
- ✅ TypeScript / Sui SDK（Scallop 就是用這個）
- ✅ DeFi protocol 設計（lending、oracle、liquidation）
- ✅ 中英文雙語（APAC 開發者社群覆蓋）

### 缺乏的

| 缺口 | 說明 | 學習難度 |
|------|------|----------|
| Senior 年資要求 | 預計要求 5–7 年（JD 未明確但職稱是 Senior） | 🔴 靠作品彌補 |
| GitHub 公開 Sui 貢獻 | 無法展示 Sui/Move public contributions | 🔴 最關鍵缺口 |
| 正式 DevRel 頭銜 | 同上述所有 DevRel 職缺 | 🔴 靠作品彌補 |
| PTB 進階組合交易 | 基本了解但沒有公開複雜 PTB 範例 | 🟡 中等（1–2 週） |

### 面試最難的部分

1. **Live 技術問答**：解釋 Sui 的 object ownership model（shared vs owned vs immutable），現場 debug 某段 Move 錯誤
2. **Content sample**：展示你寫過的 Sui 技術文章或文件貢獻
3. **Community impact**：「你有解答過哪些 Sui 社群的技術問題？」

### 如何透過貢獻找到這份工作 ✅ 這個職缺最值得長期經營

**路線（最直接）：**
1. 在 `MystenLabs/sui` 找 good-first-issue，提 PR（範例補充、文件修正、測試補強）
2. 在 Sui Discord / Telegram 的 #developer 頻道，每週回答 3–5 個技術問題
3. 寫一系列 Sui Move 技術文章（英文）：
   - 「Sui Move：Object Model 完全解析」
   - 「PTB vs 傳統合約：如何用一個 Transaction 做多步驟 DeFi」
4. 在 Twitter/X 開始技術系列帖（可以中英混合，APAC 受眾）
5. 申請 Sui Foundation / Mysten Labs 的 DevRel 貢獻者計畫

### 學習路線圖

```
Week 1–2: 深化 Sui 技術並公開化
  - 把 Scallop 工作中學到的 Sui Move 知識系統化整理
  - 寫第一篇英文 Sui 技術文章（PTB 或 object model）
  - 推上 Medium / dev.to

Week 3–4: 開始 GitHub 貢獻
  - 在 MystenLabs/sui 找到 1 個可貢獻的 issue
  - 提 PR，哪怕只是補一個範例或修錯誤

Month 2: 建立社群存在感
  - 在 Sui Discord 的 #developer 頻道活躍
  - 累積 5–10 篇技術貼文或文章
  - Cold DM Mysten Labs 的 DevRel 負責人說明你的貢獻

投遞時機：Month 3（有了作品集再投）
```

---

## 綜合行動計畫（優先順序）

### 短期（1–2 週，現在就能做）

1. **馬上投遞 Tether P2P DevRel**
   - 技術要求最符合（JS/TS + 3 年即可），年資不是嚴格門檻
   - 先投遞，同時補作品（先投後補內容策略）
   - Cover letter 重點：DeFi P2P 背景 + 中文市場 + AI 工具使用

2. **建立第 1 個 public DevRel demo**（所有 DevRel 職缺共同需求）
   - 任選一個：Holepunch P2P demo 或 Walrus 上手 demo
   - 先做出來，先推上 GitHub

### 中期（3–4 週）

3. **投遞 Tether AI / QVAC**
   - 先補 Llama.cpp 基礎（1 週），再投遞
   - 關鍵武器：「我是 Claude Code 重度使用者，直接符合 preferred 條件」

4. **開始 Sui GitHub 貢獻**
   - 在 `MystenLabs/sui` 或 `MystenLabs/walrus-docs` 提第一個 PR

### 長期（Month 2–3）

5. **建立 DevRel 作品集後投遞 Mysten Labs / Sui Foundation**
   - 需要：英文技術文章 2–3 篇 + GitHub 貢獻 + 社群回答記錄

6. **Crossmint 列為備選**（地理限制是主要障礙，Month 3 評估是否值得投）

---

## DevRel 職缺共同技能缺口速查

| 技能 | 重要度 | 學習難度 | 建議時間 |
|------|--------|----------|---------|
| 公開技術文章（英文） | 超高（所有職缺） | 🟠 中高 | 立刻開始 |
| public GitHub demo project | 超高（所有職缺） | 🟡 中等 | 立刻開始 |
| Walrus / 去中心化儲存 | 高（Sui Foundation / Mysten） | 🟡 中等 | 2–3 週 |
| Llama.cpp / 本地 AI | 高（Tether AI） | 🟠 中高 | 2–3 週 |
| Holepunch / P2P 協議 | 高（Tether P2P） | 🟡 中等 | 1–2 週 |
| 演講 / 工作坊紀錄 | 高（所有 DevRel 職缺） | 🔴 高（需時間） | 持續累積 |
| Stablecoin / wallet infra（EVM） | 中（Crossmint） | 🟡 中等 | 1–2 週 |

---

## DevRel vs 工程師職缺：給 Wei-Chun 的選擇建議

| 面向 | 工程師職缺（Day 6 分析） | DevRel 職缺（本次分析） |
|------|--------------------------|------------------------|
| 薪資上限 | $89k–$180k+ | $115k–$175k |
| 技術缺口 | Stablecoin CDP、viem/wagmi | Llama.cpp、公開內容、演講 |
| 競爭激烈程度 | 高（全球 frontend/Move 工程師都在搶） | 相對低（Sui DevRel 候選人極少） |
| Sui/Move 優勢 | 直接加分（Move Engineer 職缺） | 極大稀缺優勢（DevRel 的 Sui 候選人市場更小） |
| 投遞時機 | Month 3 | Month 3（需要先建 content portfolio） |
| 最大行動 | 補作品集 + 貢獻 repo | 補作品集 + 寫技術文章 |

**結論：DevRel 方向的 Sui 競爭者更少，Wei-Chun 的稀缺性更強。建議工程師 + DevRel 雙線並行，不要只投其中一條。**

---

*生成日期：2026-03-24 | 下次更新建議：Tether P2P 投遞後記錄狀態到 PROGRESS.md*
