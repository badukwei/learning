# Claude Code — Making Changes 學習筆記

---

## 截圖溝通

### 怎麼用
在 Claude Code 裡按 `Ctrl+V`（macOS 注意：**不是** `Cmd+V`）貼上截圖。

### 為什麼有效
文字描述 UI 問題很容易有歧義，截圖讓 Claude 直接看到你說的是哪裡，減少來回溝通。

### 使用場景
```
# 場景：UI 跑版
1. 截下跑版的畫面
2. Ctrl+V 貼進 Claude Code
3. 輸入：「這個 modal 的 padding 不對，幫我調整成跟設計稿一樣」
```

---

## Planning Mode（規劃模式）

### 啟用方式
```
Shift + Tab 兩次
（如果已在 Auto-Accept 模式，按一次就夠）
```

終端機底部會出現 `⏸ plan mode on` 的提示。

### 它在做什麼
啟用後 Claude 會：
1. 用 **read-only** 方式掃描 codebase（不會動任何檔案）
2. 建立詳細的實作計畫
3. 等待你批准後才開始執行

### 使用場景

**場景 1：新功能開發**
```
# 你想加一個 user notification 系統
# 直接叫 Claude 做 → 它可能漏掉現有的 event bus 架構

# 改用 Plan Mode：
Shift + Tab + Tab
> 幫我加一個 notification 系統，讓用戶可以收到 DeFi position 的警示

# Claude 會先掃描：
# - 現有的 event 架構在哪裡
# - 目前的 websocket 設置
# - 相關的 type 定義
# 然後列出計畫讓你確認，再動手
```

**場景 2：重構前確認影響範圍**
```
Shift + Tab + Tab
> 我想把 fetchUserPositions 這個 function 重構，
  把它改成 React Query，會影響哪些地方？

# Claude 列出所有呼叫點後，你確認沒問題再讓它執行
```

**場景 3：不確定方向**
```
# Plan Mode 下 Claude 會主動問你問題釐清需求，
# 適合需求還不完全確定的時候
```

### 什麼時候用
- 需要改動多個檔案
- 不熟悉某個 codebase 的架構
- 不想 Claude 直接動手，想先看計畫
- 需求還模糊，想讓 Claude 幫忙問問題

---

## Thinking Mode（思考模式）

### 啟用方式
在請求裡加入關鍵字，位置不限（開頭、結尾、中間都可以）：

| 等級 | 關鍵字 | Token 預算 | 適合 |
|------|--------|-----------|------|
| 基本 | `think` | ~4K | 一般問題 |
| 進階 | `think hard` / `think more` | ~10K | 中等複雜度 |
| 最強 | `think harder` / `ultrathink` | ~32K | 架構決策、複雜 bug |

> **重要：** 這些關鍵字**只在 Claude Code CLI 有效**，在 claude.ai 網頁或 API 直接呼叫裡沒有效果。

### 使用場景

**場景 1：難以重現的 bug**
```
> 這個 liquidation bot 在高 gas 的時候偶爾會失敗，
  但我找不到原因。ultrathink 幫我分析一下。
```

**場景 2：架構決策**
```
> 我們的 DeFi indexer 現在用 polling，
  想改成 event-driven。ultrathink 幫我評估
  用 WebSocket 還是 webhook 比較好，
  考慮我們目前的 Sui 節點架構。
```

**場景 3：普通任務不需要 ultrathink**
```
# ❌ 浪費 token
> ultrathink 幫我把這個 function 加上 error handling

# ✅ 用 think 就夠
> think 幫我把這個 function 加上 error handling
```

**場景 4：Planning + Thinking 組合**
```
# 最強組合：先用 Plan Mode 規劃廣度，
# 再用 ultrathink 深入分析某個特定問題

Shift + Tab + Tab
> 我要把整個 lending protocol 的利率計算邏輯重構，
  先幫我規劃一下方向

# 確認計畫方向後，針對最複雜的部分：
> ultrathink 這個 interest rate model 的邊界條件
  會不會有精度問題？
```

### 什麼時候用哪個

| 情況 | 建議 |
|------|------|
| 加 error handling、小改動 | 不需要 |
| 設計快取策略 | `think hard` |
| 複雜的 bug debug | `think harder` |
| 架構重設計 | `ultrathink` |
| legacy code 整合 | `ultrathink` |
| 完全卡住找不到方向 | `ultrathink` |

### 費用提醒
- `ultrathink` 約比普通請求多 5-8 倍 token
- 用 `/cost` 指令可以查看目前對話花了多少
- **不要每次都用 ultrathink**，只在真的需要深度分析時才用

---

## Planning vs Thinking 比較

| | Planning Mode | Thinking Mode |
|---|---|---|
| **啟用方式** | `Shift + Tab` × 2 | 在 prompt 加關鍵字 |
| **處理的是** | 廣度（跨檔案理解） | 深度（複雜推理） |
| **特點** | Read-only 探索，等你確認才執行 | 在回答前花更多時間思考 |
| **適合** | 多檔案任務、需要先看計畫 | 複雜邏輯、難以 debug 的問題 |
| **可以組合嗎** | ✅ 可以同時用 | ✅ 可以同時用 |

---

## Git 整合

Claude Code 可以直接幫你做 git 操作：

```
> 幫我 stage 這次的改動並 commit，
  commit message 要說明這次加了什麼
```

Claude 會自動：
1. `git add` 相關檔案
2. 寫一個有意義的 commit message
3. 執行 `git commit`

---

## 快速參考

```bash
# 啟用 Planning Mode
Shift + Tab + Tab

# 啟用 Thinking Mode（在 prompt 裡加）
think           # 基本
think hard      # 進階
ultrathink      # 最強

# 貼截圖
Ctrl + V        # macOS 注意不是 Cmd+V

# 查費用
/cost
```

---

## 核心原則

> **Planning 管廣度，Thinking 管深度，兩個都有 token 成本，用在刀口上。**

實際工作流程建議：
1. 複雜任務 → 先開 **Plan Mode** 讓 Claude 規劃
2. 遇到卡關或架構決策 → 加 **ultrathink**
3. 簡單任務 → 直接說就好，不需要特殊模式