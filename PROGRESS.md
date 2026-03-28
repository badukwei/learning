# 求職進度追蹤

更新：2026-03-25 ｜ Month 1, Week 2, Day 8

---

## 本月目標（Month 1）— 驗證方向

Month 1 只做一件事：**搞清楚自己想走哪條路**。

- [ ] 試寫 Move / DeFi code（自己選題目，感受有沒有動力）
- [ ] 試寫 1 篇技術筆記或文件貢獻（感受 DevRel 性質的工作）
- [ ] 試在 Sui 社群回答 1-2 個問題（感受社群互動）
- [ ] 月底回顧：工程師 vs DevRel vs 混合，選一條主線

> 不投遞、不建 portfolio、不追新技術。只驗證。

---

## 本週任務（Week 2，剩餘）

- [x] 看職缺、深度分析 5 個最適合職缺 → day-6/job-search.md
- [x] 初步整理學習方向 → day-6/learning-strategy.md
- [x] DevRel 職缺深度分析（5 個）→ day-7/job-search.md
- [x] DevRel 入行指南整理 → day-7/devrel-guide.md
- [ ] 找一個 Move / DeFi 的小題目，開始寫（不需要完成，感受就好）

---

## Week 1 回顧（2026-03-18 ~ 2026-03-22）

Week 1 只有 3 天工作天，主力放在學習打底。

### ✅ 完成
- Claude Code 完整課程（Day 1-5）：Context 管理、Custom Commands、MCP、GitHub Actions、Hooks 深度實作

### ⏭ 順延到 Week 2
- 職缺調查、side project 規劃、open source 目標 repo

---

## 已完成

### Day 7（2026-03-24）
- DevRel / Technical Writer 方向職缺調查：第二輪，聚焦 DevRel 職缺（Tether P2P、Tether AI、Sui Foundation Walrus、Crossmint、Mysten Labs）→ day-7/job-search.md
- 學習策略更新：Tether P2P 可立刻投遞，最大缺口是「公開內容 portfolio」而非技術 → day-7/learning-strategy.md
- 結論：DevRel 方向 Sui 候選人稀缺性更強，建議工程師 + DevRel 雙線並行
- DevRel 入行指南整理：7 篇業界經驗文章，含工作內容拆解、一天工作樣貌、30 天行動計畫 → day-7/devrel-guide.md

### Day 6（2026-03-23）
- 職缺深度分析：第一輪職缺調查，挑 5 個做深度拆解（技術缺口、面試難點、貢獻入場策略）→ day-6/job-search.md
- 初步學習方向整理：根據第一輪觀察，記錄初步技術方向，持續職缺調查後會調整 → day-6/learning-strategy.md

### Day 5（2026-03-22）
- Claude Agent SDK 學習（原名 Claude Code SDK）
  - 三種使用方式：TypeScript SDK、Python SDK、CLI（`claude -p`）
  - async iterator 輸出格式，篩 `message.type === "result"` 取最終結果
  - 權限設定：預設 read-only，需明確開放 `allowedTools`
  - `settingSources`：載入 CLAUDE.md 和 `.claude/` 設定（預設不載入）
  - System Prompt 注入：給 SDK 執行的 Claude 設定角色
  - 實際應用：Hook 裡啟動第二個 Claude 做 code review、git pre-commit 自動 review、CI/CD 自動修復測試、定期 code quality 檢查

### Day 4（2026-03-21）
- Claude Code Hooks 深度實作
  - 完整實作 .env 保護 Hook（PreToolUse + exit 2 阻擋機制）
  - 團隊共用方案：`settings.example.json`（$PWD 佔位）+ `init-claude.js` setup 腳本
  - 其他 Hook 類型：`Notification`、`Stop`、`SubagentStop`、`PreCompact`、`UserPromptSubmit`、`SessionStart`、`SessionEnd`
  - 進階：Hook + MCP 組合（寫 UI 前自動抓 Figma 設計稿）
  - Debug 技巧：用 `jq . > log.json` 記錄 stdin 格式

### Day 3（2026-03-20）
- Claude Code Hooks 學習
  - PreToolUse / PostToolUse 機制與差異（PreToolUse 可阻擋，PostToolUse 不行）
  - 設定位置：global / project shared / project local
  - 實作：TypeScript 型別檢查 Hook、.env 保護 Hook、自動格式化 Hook
  - `/hooks` 指令（互動式設定，不需手寫 JSON）

### Day 2（2026-03-19）
- Anthropic Claude Code 課程學習
  - Custom Commands：`.claude/commands/` 結構、`$ARGUMENTS` 參數、實用範例（/check、/write_tests、/review、/commit）
  - MCP Servers：MCP 概念與三種 scope（local/project/user）、Playwright 瀏覽器控制、Notion 整合
  - GitHub Integration：`/install-github-app`、`@claude` mention action、自動 PR review、Workflow 客製化（加入 MCP + allowed_tools）

### Day 1（2026-03-18）
- Anthropic Claude Code 課程學習
  - Context 管理：CLAUDE.md 三層結構、`@`、`#`、`/compact`、`/clear`
  - Making Changes：Planning Mode（Shift+Tab×2）、Thinking Mode（think/ultrathink）、截圖溝通

---

## AI 工具清單

| 工具名稱 | 狀態 | 說明 |
|----------|------|------|
| （空） | — | — |

---

## 職缺追蹤

| 公司 | 職位 | 薪資 | 符合度 | 狀態 |
|------|------|------|--------|------|
| Veda | Move (Sui) Smart Contract Engineer | 未標示 | ⭐⭐⭐⭐⭐ | 觀察中（先貢獻）|
| LI.FI | Senior Frontend Engineer (Perps) | €80K–€120K | ⭐⭐⭐⭐⭐ | 觀察中（⚠️ EMEA/LATAM 限制）|
| Sui Foundation | Senior DevRel Engineer, Walrus (APAC) | 未標示 | ⭐⭐⭐⭐ | 觀察中（Month 3 投遞）|
| Blockchain.com | Frontend Engineer (React) | $89K–$109K | ⭐⭐⭐⭐ | 觀察中 |
| MLabs | Move (Sui) Engineer | 未標示 | ⭐⭐⭐⭐ | 觀察中（⚠️ EST overlap）|
| Tether | Developer Relations P2P (100% remote) | $126K–$138K | ⭐⭐⭐⭐⭐ | 觀察中（短期投遞）|
| Tether | Developer Relations AI / QVAC (100% remote) | $115K–$117K | ⭐⭐⭐⭐ | 觀察中（補 Llama.cpp 後投遞）|
| Mysten Labs | Senior Developer Relations Engineer | 未標示 | ⭐⭐⭐⭐ | 觀察中（Month 3，先貢獻）|
| Crossmint | Senior Developer Relations Engineer | $140K–$175K | ⭐⭐⭐ | 觀察中（⚠️ US hybrid 限制）|

---

## Open Source 貢獻

| Repo | 類型 | 狀態 |
|------|------|------|
| `MystenLabs/walrus` | 文件 / SDK 範例 | 待找 issue |
| `MystenLabs/sui` examples | 範例 / 文件 | 待找 issue |
| `scallop-io/sui-scallop-sdk` | 文件 / test | 待評估 |

---

## 學習方向（持續更新，每週職缺調查後補充）

> 這裡只記錄「目前已觀察到的方向」，不是定論。每天看職缺，每週末彙整一次。

### 目前觀察到的技術需求（Week 2，第一輪）

| 技術 | 在職缺中出現頻率 | 備註 |
|------|----------------|------|
| Sui / Move | 高（Web3 職缺標配） | 已有實務 |
| React / TypeScript | 高（通用） | 已有實務 |
| viem / wagmi | 中（EVM frontend 職缺）| 缺口，待確認是否普遍 |
| Walrus Protocol | 低（特定 DevRel 職缺）| 值得觀察 |
| WebSocket / 即時資料 | 中（DEX 類職缺）| 待更多樣本確認 |

_樣本數：5 個職缺（2026-03-23）。持續調查後更新。_

### 貢獻目標 Repo（初步，待更多職缺確認）

| Repo | 方向 | 狀態 |
|------|------|------|
| `MystenLabs/walrus` | 文件 / SDK 範例 | 待找 issue |
| `MystenLabs/sui` examples | 文件 / 範例 | 待找 issue |

---

## 筆記 / 方向調整

- Week 1（只有 3 天）以學習打底為主
- Week 2：職缺調查，初步觀察到 DevRel / 合約工程師兩個方向，方向仍在驗證中
- 2026-03-25：調整計畫。Month 1 以驗證方向為主，不投遞不衝數字。每天 1h，有休息日，慢慢來。
- 傾向方向：Sui/Move 合約工程師（DeFi）為主，DevRel 為探索。月底再決定。
- 能量狀況：低能量，工作不忙但下班後能量有限。計畫設計要有彈性、有休息空間。
