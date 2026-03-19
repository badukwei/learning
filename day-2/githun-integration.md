# Claude Code — GitHub Integration 學習筆記

---

## 什麼是 GitHub Integration

讓 Claude Code 跑在 GitHub Actions 裡，變成一個自動化的團隊成員：
- 有人在 issue 或 PR 裡 `@claude` → Claude 自動處理任務
- 有人開 PR → Claude 自動 review

---

## Step by Step：初次設定

### 前置條件

```bash
# 1. 安裝 GitHub CLI
brew install gh

# 2. 登入 GitHub
gh auth login
# 選 GitHub.com → HTTPS → Login with a web browser
# 複製 one-time code → 貼到瀏覽器 → 完成
```

---

### Step 1：在 Claude Code 裡執行安裝指令

```
/install-github-app
```

---

### Step 2：選擇要安裝的 repo

```
# 格式
badukwei/learning

# 或貼完整網址
https://github.com/badukwei/learning
```

---

### Step 3：選擇 Workflow

兩個都勾：

```
✔ @Claude Code - Tag @claude in issues and PR comments
✔ Claude Code Review - Automated code review on new PRs
```

---

### Step 4：選擇 API Key 方式

```
> Create a long-lived token with your Claude subscription
```

選這個，用你現有的 Claude 訂閱，不需要另外付 API 費用。

---

### Step 5：安裝 Claude GitHub App

去 [github.com/apps/claude](https://github.com/apps/claude)，選你的 repo 安裝。
（如果看到 `Configure` 代表已裝好）

---

### Step 6：Merge 自動產生的 PR

去 `github.com/[你的repo]/pulls`，找 Claude 自動開的 PR，直接 Merge。

Merge 後 `.github/workflows/` 裡會多出：
- `claude.yml` → 處理 `@claude` 呼叫
- `claude-code-review.yml` → 自動 PR review

---

### Step 7：測試

去 repo 開一個 issue，在留言打：

```
@claude 你好，請問你能看到這個 issue 嗎？
```

等約 30-60 秒，看 Actions tab 有沒有跑起來，Claude 有沒有在 issue 裡回應。

> ⚠️ 注意：在 Claude Code terminal 裡打 `@claude` 沒有用，要在 **GitHub 網頁**的 issue 或 PR 留言裡打。

---

## 兩個預設 Workflow 用法

### 1. Mention Action（`@claude` 呼叫）

在 GitHub 的 issue 或 PR 留言裡 tag `@claude`，它會：
1. 讀取整個 issue / PR 的內容和 codebase
2. 建立任務計畫
3. 執行任務
4. 直接在 issue / PR 裡回報結果

**使用範例：**

```
# 在 issue 裡請它實作功能
@claude 根據上面的需求，在 src/hooks 裡建立 use-notification.ts

# 在 PR 裡請它找問題
@claude 這個 PR 有沒有遺漏什麼 edge case？

# 在 PR 裡請它重構
@claude 幫我把 fetchUserData 這個 function 重構得更乾淨

# 請它寫測試
@claude 幫這個 PR 裡新增的 function 補上測試
```

---

### 2. PR Review Action（自動 Review）

每次開 PR，不需要任何操作，Claude 自動：
- Review 所有改動
- 分析影響範圍和潛在問題
- 在 PR 上留下詳細報告

---

## Step by Step：客製化 Workflow（加入 MCP）

### 情境
你想讓 GitHub Actions 裡的 Claude 也能控制瀏覽器，在 PR 被建立時自動開啟 app 測試 UI。

---

### Step 1：找到 workflow 檔案

```bash
# 在你的專案根目錄
ls .github/workflows/
# claude.yml
# claude-code-review.yml
```

---

### Step 2：編輯 `claude.yml`

打開 `.github/workflows/claude.yml`，找到 `claude-code-action` 的步驟，加入以下設定：

**加入環境準備（在 Claude 跑之前先啟動 server）：**

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  # 加在這裡：Claude 跑之前先準備環境
  - name: Project Setup
    run: |
      npm install
      npm run build
      npm run dev:daemon   # 背景跑 dev server

  - name: Run Claude
    uses: anthropics/claude-code-action@beta
    with:
      # ... 其他設定
```

**加入 Custom Instructions（告訴 Claude 環境狀況）：**

```yaml
    with:
      custom_instructions: |
        The project is already set up with all dependencies installed.
        The server is already running at localhost:3000.
        Logs are being written to logs.txt.
        Use mcp__playwright tools to launch a browser and interact with the app.
```

**加入 MCP Server（讓 Claude 能控制瀏覽器）：**

```yaml
    with:
      mcp_config: |
        {
          "mcpServers": {
            "playwright": {
              "command": "npx",
              "args": [
                "@playwright/mcp@latest",
                "--allowed-origins",
                "localhost:3000;cdn.tailwindcss.com;esm.sh"
              ]
            }
          }
        }
```

**加入 Tool 權限（必填，每個工具都要列）：**

```yaml
    with:
      allowed_tools: >-
        Bash(npm:*),
        Bash(node:*),
        mcp__playwright__browser_navigate,
        mcp__playwright__browser_snapshot,
        mcp__playwright__browser_click,
        mcp__playwright__browser_type,
        mcp__playwright__browser_screenshot
```

> ⚠️ GitHub Actions 裡**沒有**本地那種一鍵核准，每個工具都要一個一個列，漏了就不能用。

---

### Step 3：Commit 並測試

```bash
git add .github/workflows/claude.yml
git commit -m "chore: add playwright mcp to claude workflow"
git push
```

然後開一個 PR 或在 issue 裡 `@claude`，看 Actions tab 裡 Claude 有沒有成功用到 Playwright。

---

## 完整 Workflow 範例（含 MCP）

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  claude:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Project Setup
        run: |
          npm install
          npm run dev:daemon

      - name: Run Claude
        uses: anthropics/claude-code-action@beta
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          custom_instructions: |
            Server is running at localhost:3000.
            Use mcp__playwright tools to interact with the browser.
          mcp_config: |
            {
              "mcpServers": {
                "playwright": {
                  "command": "npx",
                  "args": ["@playwright/mcp@latest", "--allowed-origins", "localhost:3000"]
                }
              }
            }
          allowed_tools: >-
            Bash(npm:*),
            mcp__playwright__browser_navigate,
            mcp__playwright__browser_snapshot,
            mcp__playwright__browser_click,
            mcp__playwright__browser_screenshot
```

---

## 實際應用情境

### 情境 1：自動抓安全問題

PR Review Action 自動分析：
- 有沒有敏感資料外洩（API key、用戶 email 進 log）
- TypeScript 型別錯誤
- DeFi 相關的邏輯漏洞

### 情境 2：`@claude` 處理 issue 並開 PR

```
# 在 issue 裡
@claude 根據這個需求實作 use-liquidation-alert.ts，
監控 position health 低於 1.2 時發出警告，
完成後開一個 PR
```

Claude 會：讀 issue → 看 codebase → 寫程式碼 → 開 PR → 在 issue 回報完成

### 情境 3：PR 建立時自動跑 UI 測試

加了 Playwright 之後，`@claude` 可以：

```
@claude 測試這個 PR 的前端改動，
打開 localhost:3000 截圖，
確認 lending dashboard 顯示正常，
把結果報告回這個 PR
```

---

## 快速參考

| 動作 | 方式 |
|------|------|
| 初次安裝 | Claude Code 裡 `/install-github-app` |
| 呼叫 Claude | GitHub issue/PR 留言裡 `@claude [任務]` |
| 自動 PR review | 開 PR 就自動觸發 |
| 加 MCP | 編輯 `.github/workflows/claude.yml` |
| 查看執行狀況 | GitHub repo → Actions tab |

---

## 注意事項

- `@claude` 要在 **GitHub 網頁**裡打，不是在 Claude Code terminal
- GitHub Actions 裡每個工具都要在 `allowed_tools` 明確列出
- 用 Claude 訂閱的 token 不需要額外付費，但複雜任務有用量限制
- 先用簡單任務測試，確認 workflow 正常再上複雜設定

---

## 核心原則

> **GitHub Integration 讓 Claude 從開發助手變成自動化團隊成員——在 issue 裡交辦任務、PR 自動 review、甚至能開瀏覽器跑 UI 測試，全部不需要你手動介入。**