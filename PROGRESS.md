# 求職進度追蹤

更新：2026-03-23 ｜ Month 1, Week 2, Day 6

---

## 本月目標（Month 1）

- [ ] 2 個 AI 工具上線並在使用中
- [ ] 瀏覽 20+ 個職缺，了解市場需求
- [ ] GitHub profile 整理完
- [ ] 履歷草稿完成
- [ ] 確定 side project 方向並開始動工

---

## 本週任務（Week 2）

- [x] 看職缺、深度分析 5 個最適合職缺，記錄技術需求與薪資範圍 → day-6/job-search.md
- [x] 初步整理學習方向（持續更新，不是定論）→ day-6/learning-strategy.md
- [ ] 根據職缺需求規劃 side project 方向（1 個具體題目）
- [ ] 列 3 個目標 open source repo，找可貢獻的 issue
- [ ] 評估哪些日常任務可以用 AI 自動化（列清單）
- [ ] 開始第 1 個 AI 應用 side project（能對應職缺需求的）

---

## Week 1 回顧（2026-03-18 ~ 2026-03-22）

Week 1 只有 3 天工作天，主力放在學習打底。

### ✅ 完成
- Claude Code 完整課程（Day 1-5）：Context 管理、Custom Commands、MCP、GitHub Actions、Hooks 深度實作

### ⏭ 順延到 Week 2
- 職缺調查、side project 規劃、open source 目標 repo

---

## 已完成

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

- Week 1（只有 3 天）以學習打底為主，Week 2 起開始往實作和求職調查走
- AI 學習方向：應用到實際 side project，不再純讀課程
- Month 1-2 以學習 + side project 為主，Month 3 才正式投遞
- 每天 4 小時：學習/實作 1.5h + AI side project 1.5h + 職缺/貢獻調查 0.5h + 市場觀察 0.5h
- 2026-03-23：第一輪職缺調查，初步觀察到 DevRel / Fullstack / 合約工程師三個方向，詳見 day-6/learning-strategy.md。持續調查中，方向未定論。
