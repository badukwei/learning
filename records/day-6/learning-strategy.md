# 學習策略整理 — 2026-03-23

> 根據職缺深度分析後，確立短中期學習方向與優先順序。

---

## 方向確認

從 5 個職缺分析中，提取三條可行路線：

| 路線 | 代表職缺 | 短期可行度 | 優先度 |
|------|---------|-----------|--------|
| **A. DevRel** | Sui Foundation Walrus DevRel | 🟢 高（已有實務背景） | ⭐⭐⭐ 主力 |
| **B. Fullstack/Frontend** | LI.FI Perps、Blockchain.com | 🟡 中（有技術缺口要補） | ⭐⭐ 並行 |
| **C. Smart Contract Engineer** | Veda、MLabs | 🔴 低（短期難突破）| 輕量貢獻即可 |

---

## 路線 A：DevRel（主力方向）

### 為什麼不需要重新建作品集

DevRel 評估的是「你能不能幫開發者上手這個協議」，不是看你有沒有新的 side project。

Wei-Chun 已具備：
- ✅ DeFi lending protocol 實務（Scallop）— 能講清楚 protocol 設計邏輯
- ✅ Sui/Move 實際開發經驗 — 能回答開發者問題
- ✅ 現職本來就在做 DevRel 性質工作（community、技術溝通）

**核心缺口不是技術，是「公開可見度」。**

---

### DevRel 職缺實際在看什麼

| 面向 | 說明 | Wei-Chun 現狀 |
|------|------|--------------|
| 技術深度 | 能不能回答開發者問的問題 | ✅ 有（Sui/Move/DeFi） |
| 內容產出 | Blog、tutorial、demo repo | ❌ 無公開內容 |
| 社群存在感 | Twitter/X、Discord 知名度 | ❌ 無公開帳號活動 |
| 公開演講 / Demo | 能不能在 hackathon 做 workshop | 🟡 有能力但無記錄 |
| 協議理解廣度 | 不只懂自己做的，還懂生態全貌 | 🟡 Scallop 深、Walrus 需補 |

---

### DevRel 強化行動（不需要大量學習，主要是「讓人看到你」）

**第 1 步：Walrus 協議研究（1 週）**

Sui Foundation Walrus 職缺要求 APAC DevRel，核心是 Walrus 分散式儲存協議。
需要快速弄懂 Walrus 的：
- 架構概念（epoch、shard、blob storage）
- 如何用 Walrus SDK 存取資料
- 對開發者的使用場景（NFT metadata 存儲、鏈上 app 靜態資源）

資源：
- https://docs.walrus.site/
- Walrus GitHub: mysten-labs/walrus

**第 2 步：寫 1-2 篇技術內容（2-3 週）**

不需要長篇，500–1000 字就夠，重點是「有公開記錄」。

選題建議（從現有經驗出發）：
- 「如何在 Sui 上建一個 DeFi lending protocol」（已做過，整理一下就有）
- 「Walrus SDK 快速入門：5 分鐘存第一個 blob」（新學完可以寫）
- 「Sui Move vs Solidity：lending 合約設計的差異」（跨生態比較，DevRel 很喜歡）

發布平台：Mirror.xyz（Web3）或 Dev.to，然後轉發到 Twitter/X。

**第 3 步：Twitter/X 存在感（持續）**

不需要每天發，一週 2-3 則就夠，重點是技術內容：
- Sui/Move 使用心得
- DeFi 協議觀察
- 轉發 Sui Foundation 重要公告並加上自己的觀點

目標：1 個月後，搜尋「Sui developer」能看到你的 profile。

**第 4 步：Sui Discord / Telegram 社群活躍（持續）**

在 Sui 官方 Discord 的 `#developer-support` 回答新手問題。
每週 2-3 則有品質的回答，比很多 PR 更有 DevRel 說服力。

---

### DevRel 面試最難的部分

1. **Live demo**：要求你現場用 Walrus SDK 或 Sui SDK 做一個示範，邊做邊講解給開發者聽
2. **Content sample**：要求你提供過去寫的文章或教學（現在還沒有 → 近期補）
3. **Community 提問模擬**：給你一個開發者的問題（如「為什麼我的 Move object 沒辦法 transfer？」），要求你解釋清楚
4. **Protocol 廣度**：問你 Walrus 的分散式儲存機制，或 Sui 生態其他協議

---

## 路線 B：Fullstack/Frontend（並行補強）

針對 LI.FI（Perps DEX frontend）、Blockchain.com（React）類型職缺。

### 技術缺口

| 缺口 | 說明 | 學習難度 | 優先度 |
|------|------|----------|--------|
| viem / wagmi | EVM 鏈上互動的現代標準 | 🟢 低（概念與 Sui SDK 類似） | 高 |
| TanStack Query | 非同步資料管理 | 🟢 低（有 React 基礎） | 高 |
| WebSocket 訂閱 | 即時 orderbook 資料 | 🟡 中 | 中 |
| Perpetual DEX 機制 | funding rate、mark price、leverage | 🟡 中（概念需補） | 中 |

### 學習路線（3-4 週）

```
Week 1: viem + wagmi 基礎
  - 官方 docs 快速過（1-2 天）
  - 用 wagmi 建一個連接 MetaMask、讀取 ERC-20 餘額的 demo
  - 推上 GitHub

Week 2: TanStack Query + 狀態管理
  - 用 TanStack Query 重構 demo 的資料讀取邏輯
  - 了解 cache / staleTime / refetchInterval 設定

Week 3: WebSocket + 即時資料
  - 訂閱一個 DEX 的 public orderbook WebSocket（Binance 或 Hyperliquid）
  - 建簡單的即時 orderbook 顯示 UI

Week 4: 整合 side project
  - 做一個 EVM DeFi dashboard：連接錢包、顯示 positions、即時價格
  - 直接放在履歷上
```

---

## 路線 C：Smart Contract Engineer（輕量貢獻，不調整主路線）

合約工程師短期難成功的原因：
- GitHub 上沒有公開 Sui Move 合約
- 面試需要 live coding 寫 Move，且會考安全性
- 這個缺口需要 1-2 個月紮實學習才有競爭力

**現在的做法：貢獻先行，讓工程師認識你**

不用自己從頭寫複雜合約，先找簡單切入點：
- 補 test（現有合約加 unit test）
- 文件（README、inline comment）
- 小 bug fix 或 gas optimization

目標：讓 Veda/MLabs 工程師在 GitHub 上「認識」你，建立人脈。

---

## 貢獻策略（Month 1-2 執行）

### 目標 Repo 清單

| Repo | 關聯職缺 | 貢獻類型 | 難度 |
|------|---------|---------|------|
| `MystenLabs/walrus` | Sui Foundation DevRel | 文件、SDK 範例 | 🟢 低 |
| `MystenLabs/sui` 官方 examples | 所有 Sui 職缺 | 範例補充、文件修正 | 🟢 低 |
| `scallop-io/sui-scallop-sdk` | Veda、MLabs（生態相關）| 文件、test | 🟢 低（現職有優勢） |
| Veda 的 public repo（待確認） | Veda | test、文件 | 🟡 中 |

### 貢獻節奏

```
Week 2-3（現在）: 找到 good first issue
  → 在 MystenLabs/walrus 或 MystenLabs/sui examples 找文件缺漏
  → 送第 1 個 PR

Week 4-6: 送 2-3 個 PR
  → 往 test 或小功能升級
  → 在 Sui Discord 開始留言

Month 2: 開始在 Veda/MLabs repo 出現
  → 有前面的貢獻紀錄，cold DM 效果更好
```

---

## 每日時間分配（調整版）

| 時段 | 活動 | 時間 |
|------|------|------|
| 主學習 | Walrus 研究 或 viem/wagmi（交替） | 1.5h |
| 貢獻 | 找 issue → 送 PR | 0.5h |
| 社群 | Twitter/X 發文 或 Discord 回答 | 0.5h |
| 職缺觀察 | 被動看職缺，記錄新發現 | 0.5h |
| 緩衝 | 實作 side project demo | 1h |

---

## 三個月預期結果

| 時間 | DevRel 路線 | Fullstack 路線 |
|------|------------|---------------|
| Month 1 末 | Walrus 研究完、1 篇文章、Twitter 開始活躍 | 第 1 個 PR 送出 |
| Month 2 末 | 2 篇公開文章、Discord 有露出 | viem/wagmi demo 完成 |
| Month 3 | 可以投 Sui Foundation DevRel，有具體佐證 | 可以投 LI.FI / Blockchain.com |
