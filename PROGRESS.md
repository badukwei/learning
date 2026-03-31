# 求職進度追蹤

更新：2026-03-28 ｜ Month 1, Week 2, Day 11

---

## 求職方向

目標：全遠端海外工作，年薪 $70k+（底線 $50k）

### 目標公司類型
- **A）Web3 / DeFi 新創**（現有背景直接適用）
- **B）一般科技新創**：SaaS、開發工具、AI 產品（技術棧通用）
- 不限 Web3，持續看市場調整

### 目標職位（優先順序）
1. Web3 / DeFi Engineer（Sui/Move 稀缺性強）
2. Fullstack / Frontend Engineer（通用）
3. DevRel / Developer Advocate（技術 + 社群混合）

---

## 未來幾週節奏（Month 1 尾 → Month 2 初）

| 週次 | 重點 |
|------|------|
| Week 2 尾（本週）| 規劃完成 |
| Week 3（Day 12–14）| 景點清單網站 MVP |
| Week 4（Day 15+）| AI Issue Finder |
| Month 2 | OSS 貢獻為主 + 開始投履歷，目標拿到面試邀約 |
| Month 2 面試邀約後 | 根據面試形式決定是否補刷題 / 系統設計 |

---

## 每日優先級（4h 上限）

| 優先 | 內容 | 時間 |
|------|------|------|
| 🥇 | Side project（主力）| 1.5h |
| 🥇 | OSS 貢獻（選 1 repo 持續推）| 1h |
| 🥈 | 職缺觀察 + 履歷整理 | 30min |
| 🥉 | 系統設計（邊做 side project 邊學，不另排）| — |
| ⏳ | 刷題（Month 3 再說，看面試要求）| — |
| ⏳ | 上課（有明確缺口再補，不先排）| — |
| 🔋 | 彈性 / 休息 | 1h |

> 系統設計不另外排時間：在 side project 的每個架構決策中練習，做完記下來，面試直接用。

---

## Side Projects

### 1. 景點清單共享網站（Day 12 開始）
朋友需求：Google Maps 沒有分享清單功能。任何人可建立主題清單、新增 / 刪除景點，不需登入。
→ 詳細計畫：`day-11/project-plan-spots.md`

**Tech stack：** Next.js + Tailwind + Supabase + Vercel

| Day | 目標 |
|-----|------|
| Day 12 | 初始化 + Supabase 設定 + 首頁 |
| Day 13 | 清單頁（景點列表 + 新增 + 刪除）|
| Day 14 | 收尾 + 部署 Vercel |

### 2. AI Issue Finder（Day 15+ 開始）
解決「怎麼找適合自己的 OSS issue」問題。輸入 GitHub repo，用 Claude API 判斷 issue 適合度，輸出推薦清單。
→ 詳細計畫：`day-11/project-plan-ai-issue-finder.md`

**Tech stack：** GitHub API + Anthropic SDK（Claude API）+ CLI

---

## 本週任務（Week 2，剩餘）

- [x] 看職缺、深度分析 5 個最適合職缺 → day-6/job-search.md
- [x] 初步整理學習方向 → day-6/learning-strategy.md
- [x] DevRel 職缺深度分析（5 個）→ day-7/job-search.md
- [x] DevRel 入行指南整理 → day-7/devrel-guide.md
- [x] 確認求職方向 + 規劃兩個 side project → day-11/
- [ ] 找一個 Move / DeFi 的小題目，開始寫（感受就好，可跳過）

---

## Week 2 回顧（2026-03-23 ~ 2026-03-28）

- 職缺調查兩輪：第一輪 5 個（Frontend / Web3），第二輪 5 個（DevRel）
- 觀察到兩個方向：Sui/Move 工程師 + DevRel，方向仍開放
- 2026-03-28：決定不限 Web3，拓展看 A/B 類型公司
- 確定 side project 路線：邊做邊驗證，比「先想清楚再動」有效

## Week 1 回顧（2026-03-18 ~ 2026-03-22）

Week 1 只有 3 天工作天，主力放在學習打底。

### ✅ 完成
- Claude Code 完整課程（Day 1-5）：Context 管理、Custom Commands、MCP、GitHub Actions、Hooks 深度實作

---

## 已完成

### Day 11（2026-03-28）
- 求職方向調整：不限 Web3，拓展到 A/B 類型公司（Web3 新創 + 一般科技新創）
- 確定未來幾週節奏：side project 主力 + 職缺觀察 + 履歷整理
- 規劃兩個 side project：景點清單網站（Day 12）、AI Issue Finder（Day 15+）
- → day-11/project-plan-spots.md
- → day-11/project-plan-ai-issue-finder.md

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

## 職缺追蹤

| 公司 | 職位 | 薪資 | 符合度 | 狀態 |
|------|------|------|--------|------|
| Veda | Move (Sui) Smart Contract Engineer | 未標示 | ⭐⭐⭐⭐⭐ | 觀察中（先貢獻）|
| LI.FI | Senior Frontend Engineer (Perps) | €80K–€120K | ⭐⭐⭐⭐⭐ | 觀察中（⚠️ EMEA/LATAM 限制）|
| Sui Foundation | Senior DevRel Engineer, Walrus (APAC) | 未標示 | ⭐⭐⭐⭐ | 觀察中（Month 3 投遞）|
| Blockchain.com | Frontend Engineer (React) | $89K–$109K | ⭐⭐⭐⭐ | 觀察中 |
| MLabs | Move (Sui) Engineer | 未標示 | ⭐⭐⭐⭐ | 觀察中（⚠️ EST overlap）|
| Tether | Developer Relations P2P (100% remote) | $126K–$138K | ⭐⭐⭐⭐⭐ | 觀察中（Month 2 投遞）|
| Tether | Developer Relations AI / QVAC (100% remote) | $115K–$117K | ⭐⭐⭐⭐ | 觀察中（Month 2 投遞）|
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

## 學習方向（持續更新）

### 目前觀察到的技術需求（Week 2）

| 技術 | 出現頻率 | 備註 |
|------|----------|------|
| Sui / Move | 高（Web3 職缺標配）| 已有實務 |
| React / TypeScript | 高（通用）| 已有實務 |
| Claude API / AI 整合 | 中（AI 產品公司）| 邊做 side project 邊學 |
| viem / wagmi | 中（EVM frontend）| 缺口，待確認是否普遍 |
| WebSocket / 即時資料 | 中（DEX 類職缺）| 待更多樣本確認 |
| Walrus Protocol | 低（特定 DevRel）| 值得觀察 |

_樣本數：10 個職缺（2026-03-23 ~ 2026-03-24）。持續調查後更新。_

---

## 筆記 / 方向調整

- Week 1（只有 3 天）以學習打底為主
- Week 2：職缺調查兩輪，觀察到 DevRel / 合約工程師兩個方向
- 2026-03-25：調整計畫。Month 1 以驗證方向為主，不投遞不衝數字。
- 2026-03-28：調整方向。不限 Web3，拓展到 A/B 類型公司。Side project 主力，邊做邊學邊驗證。每天最低目標 3h，有休息日。
- 能量狀況：低能量，工作不忙但下班後能量有限。計畫設計要有彈性、有休息空間。
