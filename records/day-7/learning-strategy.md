# 學習策略更新 — 2026-03-24

> Day 6 分析聚焦工程師職缺（Move / Fullstack）。
> Day 7 深入 DevRel 職缺後，策略有重要更新。
> 本文記錄調整後的學習方向與優先順序。

---

## 策略調整摘要

Day 6 → Day 7 的最大認知更新：

| 面向 | Day 6 認知 | Day 7 更新 |
|------|-----------|-----------|
| DevRel 職缺投遞時機 | Month 3（Sui Foundation 要 7+ 年） | **立刻可投 Tether P2P**（只需 3+ 年） |
| DevRel 競爭激烈程度 | 需要 7+ 年，門檻高 | Tether / ChainGPT 門檻低，但全球遠端競爭也廣 |
| 最大共同缺口 | 各職缺有不同技術缺口 | **所有 DevRel 職缺共同缺口是「公開內容 portfolio」** |
| 新增學習項目 | — | Llama.cpp / 本地 AI（Tether AI 職缺需求） |
| 職缺路線數量 | 3 條（DevRel / Fullstack / 合約） | 調整為 **4 條**，DevRel 拆分為短期 + 長期 |

---

## 更新後的方向地圖

| 路線 | 代表職缺 | 投遞時機 | 優先度 |
|------|---------|---------|--------|
| **A1. DevRel 短期**（Tether） | Tether P2P、Tether AI | 立刻～4 週 | ⭐⭐⭐ 主力 |
| **A2. DevRel 長期**（Sui 生態） | Sui Foundation Walrus、Mysten Labs | Month 3 | ⭐⭐⭐ 並行 |
| **B. Fullstack/Frontend** | LI.FI Perps、Blockchain.com | Month 3 | ⭐⭐ 並行 |
| **C. Smart Contract Engineer** | Veda、MLabs | Month 2–3 | ⭐ 輕量貢獻 |

---

## 路線 A1：DevRel 短期（Tether，立刻可行）

### 為什麼 Tether 是現在最好的切入點

1. **年資門檻低**：只要 3+ 年，Wei-Chun 直接符合
2. **技術棧對口**：TypeScript / Node.js / Web3，全部有實務
3. **薪資透明且高**：$115k–$138k，遠超目標 $70k+
4. **全球遠端**：地點不限，台灣完全可投
5. **兩個方向**：P2P（Holepunch）和 AI（QVAC），Wei-Chun 都有對應背景

### 唯一缺口：公開 DevRel 作品集

Tether 明確要求：
- 有寫過 tutorial / blog / demo app
- 有在社群回答問題的記錄
- 有展示 API / SDK 整合能力的 demo

**現在要做的不是學新技術，是把已有的技術「公開化」。**

### ⚠️ 投遞時機調整：Month 1 先不投

**Month 1（前 4 週）目標：學習打底 + 貢獻 + 現職內部成就，不投遞。**

理由：
- 現在投太早，手上沒有可展示的公開作品
- DevRel 職缺一定會看 content sample，現在投是浪費機會
- 先把 demo / 文章 / 貢獻建立好，Month 2+ 投更有說服力

### Tether P2P 準備計畫（Month 1 執行，Month 2 投遞）

```
Week 2–3（技術探索）:
  - 讀 Holepunch 文件（Hypercore、Autobase）
  - 用 Holepunch SDK 做一個 P2P demo（例如：去中心化 note sync）
  - 推上 GitHub，寫清楚 README

Week 3–4（內容化）:
  - 整理 demo，寫英文 tutorial（500–800 字）
  - 發布到 dev.to / Medium
  - 同步在 Twitter/X 分享

Month 2: 準備 cover letter → 投遞
```

### Tether AI（QVAC）準備計畫（Month 2 執行，Month 2+ 投遞）

```
Month 2 Week 1（Llama.cpp 上手）:
  - 安裝 Llama.cpp，跑第一個 GGUF 模型
  - 了解 quantization（Q4_K_M 是什麼？）
  - 用 llama.cpp server + TypeScript 做一個 local chat demo

Month 2 Week 2: 內容化 → 投遞
```

---

## 路線 A2：DevRel 長期（Sui 生態，Month 3）

Day 6 已有詳細分析，這裡只記錄更新的認知：

### 更新：Mysten Labs vs Sui Foundation 的差異

| 面向 | Sui Foundation DevRel | Mysten Labs DevRel |
|------|----------------------|-------------------|
| 重心 | 生態建立、APAC 社群 | 技術深度、開發者工具 |
| 適合 Wei-Chun 的理由 | APAC + Sui/Move 稀缺性 | Move 技術深度 + TypeScript SDK 實務 |
| 現在就能做的事 | Walrus docs 貢獻 | `MystenLabs/sui` examples PR |

### 兩個都要做，行動一樣

不論最後投哪個，現在的行動都相同：
1. 在 `MystenLabs/walrus-docs` 或 `MystenLabs/sui` 提 PR
2. 開始在 Sui Discord `#developer-support` 回答問題
3. 寫第一篇 Sui 技術文章（英文）

---

## 路線 B：Fullstack/Frontend（維持 Day 6 計畫）

沒有大更新。維持 Day 6 的學習路線：
- viem / wagmi（EVM frontend）
- TanStack Query
- WebSocket / 即時 orderbook UI

**Month 2 開始補，Month 3 投遞 LI.FI / Blockchain.com。**

---

## 最重要的認知轉變：「公開化」比「學習」更緊迫

Day 7 分析後，最清楚的結論是：

**Wei-Chun 現在缺的不是技術，是「讓人看見技術的管道」。**

| 現在有的 | 現在沒有的 |
|---------|----------|
| ✅ Sui/Move 實務 | ❌ 沒有公開的 GitHub demo |
| ✅ DeFi protocol 設計經驗 | ❌ 沒有英文技術文章 |
| ✅ TypeScript / Node.js | ❌ 沒有 Twitter/X 技術貼文 |
| ✅ AI 工具重度使用 | ❌ 沒有社群回答問題的記錄 |
| ✅ 中英文雙語 | ❌ 沒有公開演講 / 工作坊紀錄 |

**結論：Month 1 的重點任務應該從「學習」轉向「公開化」。**

---

## 調整後的每日時間分配

Month 1 Week 2–4：

| 優先 | 活動 | 時間 | 說明 |
|------|------|------|------|
| 🥇 | 公開化（寫文章 / 做 demo） | 1.5h | 把已有技術轉化為可見產出 |
| 📚 | 新技術學習（Holepunch / Llama.cpp / Walrus 擇一） | 1h | 針對當週目標職缺 |
| 🐙 | GitHub 貢獻 | 0.5h | 找 issue → 送 PR |
| 👀 | 社群（Discord 回答 / Twitter/X 發文） | 0.5h | 建立可見度 |
| 🔍 | 職缺觀察 | 0.5h | 被動看職缺，記錄新發現 |

---

## 內容 Portfolio 建立計畫

DevRel 職缺都需要「content sample」，這是最需要主動建立的東西。

### 內容清單（Month 1-2 目標）

| 優先 | 主題 | 職缺關聯 | 目標日期 |
|------|------|---------|---------|
| 🔥 | Holepunch SDK 入門 tutorial | Tether P2P | 2 週內 |
| 🔥 | Walrus 第一個 blob 存取 demo | Sui Foundation / Mysten | 3 週內 |
| 📌 | Sui Move Object Model 解析（英文）| Mysten Labs | Month 2 |
| 📌 | 本地 AI + 區塊鏈應用場景（英文）| Tether AI | Month 2 |
| 📌 | DeFi lending 設計回顧（英文）| 所有職缺 DevRel 加分 | Month 2 |

### 發布平台策略

| 平台 | 用途 |
|------|------|
| GitHub | demo repo、PR 記錄（最重要，面試必看） |
| dev.to | 英文技術文章主平台（SEO 好，開發者看） |
| Twitter/X | 短文 + 文章轉發，建立社群存在感 |
| Sui Developer Forum | Sui 生態的回答記錄（直接被 Mysten 看到） |

---

## 三個月預期結果（更新版）

> Month 1 = 學習 + 貢獻 + 內部成就，**不投遞**。Month 2 開始投。

| 時間 | DevRel 短期（Tether） | DevRel 長期（Sui）| Fullstack |
|------|----------------------|-----------------|----------|
| Month 1 末 | 1 個 demo repo + 1 篇文章完成，準備好投遞 | 1 個 PR 送出，Walrus demo 完成 | viem demo 開始 |
| Month 2 | 投遞 Tether P2P，補 Llama.cpp，投遞 Tether AI | 2 篇文章，Discord 開始活躍 | viem/wagmi demo 完成 |
| Month 3 | 根據 Tether 結果決定下一步 | 可以投 Sui Foundation / Mysten Labs | 可以投 LI.FI / Blockchain.com |

---

*生成日期：2026-03-24 | 下次更新：Tether P2P 投遞後，根據 JD 反饋調整方向*
