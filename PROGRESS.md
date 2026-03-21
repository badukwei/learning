# 求職進度追蹤

更新：2026-03-21 ｜ Month 1, Week 1, Day 4

---

## 本月目標（Month 1）

- [ ] 2 個 AI 工具上線並在使用中
- [ ] 瀏覽 20+ 個職缺，了解市場
- [ ] GitHub profile 整理完
- [ ] 履歷草稿完成

---

## 本週任務（Week 1）

- [ ] 建第 1 個 AI 工具 prototype（建議：JD 分析工具）
- [ ] 開職缺追蹤表，看 3-5 個職缺並記錄
- [ ] 列 3 個目標 open source repo
- [x] 讀 1 篇技術文章並留 3 行筆記

---

## 已完成

### Day 4（2026-03-21）
- Claude Code Hooks 深度實作
  - 完整實作 .env 保護 Hook（PreToolUse + exit 2 阻擋機制）
  - 團隊共用方案：`settings.example.json`（$PWD 佔位）+ `init-claude.js` setup 腳本
  - 其他 Hook 類型：`Notification`、`Stop`、`SubagentStop`、`PreCompact`、`UserPromptSubmit`、`SessionStart`、`SessionEnd`
  - 進階：Hook + MCP 組合（寫 UI 前自動抓 Figma 設計稿）
  - Debug 技巧：用 `jq . > log.json` 記錄 stdin 格式
  - 不同 Hook 的 stdin JSON 格式差異

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
| （空） | — | — | — | — |

---

## Open Source 貢獻

| Repo | 類型 | 狀態 |
|------|------|------|
| （空） | — | — |

---

## 筆記 / 方向調整

- Month 1-2 以學習為主，Month 3 才正式投遞
- 每天 4 小時，學習 1.5h + AI 工具 1.5h + GitHub 0.5h + 市場觀察 0.5h