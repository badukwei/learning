# Claude Code — MCP Servers 學習筆記

---

## 什麼是 MCP Server

MCP（Model Context Protocol）是 Anthropic 開源的標準協議，讓 Claude Code 能夠連接外部工具和服務。

沒有 MCP 的 Claude Code 只能讀寫你本機的檔案。有了 MCP，它可以控制瀏覽器、查資料庫、存取 Notion、操作 GitHub——幾乎任何有 API 的服務都可以接。

```
Claude Code（MCP client）
    ↕
MCP Server（翻譯層）
    ↕
外部服務（Notion / GitHub / 資料庫 / 瀏覽器...）
```

---

## MCP Server 的三種 Scope

| Scope | 說明 | 適合 |
|-------|------|------|
| `local`（預設） | 只有你，只在這個專案 | 專案特定的 DB 連線 |
| `project` | 整個團隊共用，via `.mcp.json` | 團隊都需要的工具 |
| `user` | 只有你，跨所有專案 | Playwright、Notion、GitHub |

```bash
# 預設（local）
claude mcp add playwright npx @playwright/mcp@latest

# 跨所有專案（推薦給常用工具）
claude mcp add --scope user playwright npx @playwright/mcp@latest

# 團隊共用
claude mcp add --scope project playwright npx @playwright/mcp@latest
```

---

## Step by Step：安裝 Playwright MCP

### 情境
你在開發前端，想讓 Claude 直接開瀏覽器看畫面，幫你找 UI 問題、自動調整樣式。

---

### Step 1：安裝（在終端機，不是 Claude Code 裡）

```bash
# 推薦用 user scope，所有專案都能用
claude mcp add --scope user playwright npx @playwright/mcp@latest
```

---

### Step 2：重啟 Claude Code

```bash
exit
claude
```

---

### Step 3：確認安裝成功

```
/mcp
```

看到 `playwright · ✓ connected` 就成功了。

---

### Step 4：預先核准權限（可選）

不想每次都跳確認視窗，打開 `.claude/settings.local.json`：

```json
{
  "permissions": {
    "allow": ["mcp__playwright"],
    "deny": []
  }
}
```

> 注意：是兩個底線 `mcp__playwright`

---

### Step 5：使用

```bash
# 截圖看現在長什麼樣
> 開啟 localhost:3000，截圖給我看

# 視覺調整（Claude 看畫面 + 改程式碼）
> 導航到 localhost:3000，分析目前的 UI 問題，
  然後更新 @src/styles/dashboard.css 修復它

# 自動優化 prompt
> 導航到 localhost:3000，產生一個基本元件，
  分析樣式品質，然後更新 @src/lib/prompts/generation.tsx
  讓之後產生的元件更好看
```

**為什麼這樣有效：** Claude 能看到實際的視覺輸出，不只是程式碼，所以能做出更精準的樣式改進決策，而不是猜。

---

## Step by Step：安裝 Notion MCP

### 情境
你想讓 Claude Code 直接讀寫 Notion workspace，不用再手動複製貼上文件內容。

---

### Step 1：安裝

```bash
claude mcp add --scope user --transport http notion https://mcp.notion.com/mcp
```

> 如果你的 claude.ai 帳號已經連接了 Notion，`/mcp` 裡會看到 `claude.ai Notion`，可以直接用那個，不需要手動裝。

---

### Step 2：重啟並認證

```bash
exit
claude
```

```
/mcp
```

選 `notion` 或 `claude.ai Notion` → 跑 OAuth 流程，用瀏覽器登入 Notion 帳號授權。

---

### Step 3：使用

```bash
# 讀取
> 列出我 Notion workspace 裡的頁面

# 搜尋
> 搜尋 Notion 裡跟 Claude Code 相關的內容

# 建立新頁面
> 在 Notion 建立一個新頁面，標題叫「MCP 測試」，
  內容寫今天學到的重點

# 修改指定頁面
> 在 Notion 的「學習筆記」頁面最後加一行：
  今天成功設定了 MCP Servers
```

---

## 常用 MCP Servers 清單

| 用途 | 安裝指令 |
|------|---------|
| 瀏覽器控制 | `claude mcp add --scope user playwright npx @playwright/mcp@latest` |
| Notion | `claude mcp add --scope user --transport http notion https://mcp.notion.com/mcp` |
| GitHub | `claude mcp add --scope user --transport http github https://api.githubcopilot.com/mcp/v1` |
| PostgreSQL | `claude mcp add postgres npx @modelcontextprotocol/server-postgres postgresql://localhost/mydb` |
| Filesystem | `claude mcp add filesystem npx @modelcontextprotocol/server-filesystem /path/to/dir` |

---

## 常用指令

```bash
# 列出所有已安裝的 MCP servers
/mcp

# 查看 context token 使用量（包含 MCP）
/context

# 移除 MCP server
claude mcp remove playwright
```

---

## 快速參考

| 步驟 | 動作 |
|------|------|
| 1 | `claude mcp add [--scope user] [名稱] [啟動指令]` |
| 2 | 重啟 Claude Code |
| 3 | `/mcp` 確認安裝成功 |
| 4 | 需要 OAuth 的話跑認證流程 |
| 5 | 在 `settings.local.json` 預先核准權限（可選） |
| 6 | 直接用自然語言叫 Claude 操作 |

---

## 核心原則

> **MCP 讓 Claude 從「只能改你電腦上的檔案」變成「能跟你整個工具鏈互動的開發夥伴」。**

安裝一個 MCP server 只要一行指令。優先安裝你最常切換去的工具——對工程師來說通常是：

1. **Playwright**（瀏覽器）→ 視覺調整、UI 測試
2. **Notion**（文件）→ 讀規格、寫筆記
3. **GitHub**（版控）→ 處理 issue、review PR