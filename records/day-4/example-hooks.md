# Claude Code — Hooks 學習筆記

---

## 什麼是 Hooks

Hooks 讓你在 Claude 執行工具的前後，插入自己的指令。

**正常流程：**
```
你的問題 → Claude 決定用哪個工具 → 執行工具 → 回傳結果
```

**加了 Hooks：**
```
你的問題 → Claude 決定用哪個工具
  → PreToolUse Hook 執行（可阻擋）
  → 執行工具
  → PostToolUse Hook 執行（不能阻擋，但可回饋）
  → 回傳結果
```

---

## 兩種 Hook 類型

| 類型 | 時機 | 能阻擋？ | 用途 |
|------|------|---------|------|
| **PreToolUse** | 工具執行前 | ✅ 可以 | 存取控制、安全防護 |
| **PostToolUse** | 工具執行後 | ❌ 不行 | 格式化、測試、型別檢查 |

---

## 設定位置

| 位置 | 檔案 | 說明 |
|------|------|------|
| 全域 | `~/.claude/settings.json` | 對所有專案生效 |
| 專案（共用） | `.claude/settings.json` | 整個團隊共用，commit 進去 |
| 專案（個人） | `.claude/settings.local.json` | 不提交，個人設定 |

---

## 設定格式

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "工具名稱",
        "hooks": [
          {
            "type": "command",
            "command": "要執行的指令"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "工具名稱",
        "hooks": [
          {
            "type": "command",
            "command": "要執行的指令"
          }
        ]
      }
    ]
  }
}
```

**Matcher 格式：**
- 單一工具：`"Read"`
- 多個工具：`"Write|Edit|MultiEdit"`
- 所有工具：`".*"`

**常用工具名稱：**
- `Read` — 讀取檔案
- `Write` — 寫入檔案
- `Edit` — 編輯檔案
- `MultiEdit` — 批次編輯
- `Bash` — 執行 shell 指令
- `WebFetch` — 抓取網頁

---

## Hook 腳本如何運作

Claude Code 會把工具呼叫的詳細資訊透過 stdin 以 JSON 格式傳給你的腳本：

```json
{
  "session_id": "abc123",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/path/to/file.ts"
  }
}
```

你的腳本根據內容決定：
- **Exit 0** → 允許操作繼續
- **Exit 2** → 阻擋操作（只有 PreToolUse 有效），stderr 的內容會回傳給 Claude

---

## Step by Step：建立 TypeScript 型別檢查 Hook

### 情境
你在寫 TypeScript，Claude 改完 function signature 但忘記更新 call site，導致型別錯誤。想讓 Claude 每次改完檔案就自動跑型別檢查，錯誤直接回饋給它自己修。

---

### Step 1：建立 hooks 資料夾

```bash
mkdir -p .claude/hooks
```

---

### Step 2：建立 hook 腳本

```bash
touch .claude/hooks/typecheck.sh
chmod +x .claude/hooks/typecheck.sh
```

寫入內容：

```bash
#!/bin/bash
# 讀取 stdin 的工具呼叫資訊
input=$(cat)

# 抓出被編輯的檔案路徑
file_path=$(echo "$input" | node -e "
  const data = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));
  console.log(data.tool_input?.file_path || data.tool_input?.path || '');
")

# 只處理 TypeScript 檔案
if [[ "$file_path" == *.ts || "$file_path" == *.tsx ]]; then
  # 跑型別檢查
  result=$(npx tsc --no-emit 2>&1)
  
  if [ $? -ne 0 ]; then
    # 有型別錯誤，回傳給 Claude
    echo "TypeScript type errors found:" >&2
    echo "$result" >&2
    exit 2  # 讓 Claude 知道有問題
  fi
fi

exit 0
```

---

### Step 3：加入設定檔

編輯 `.claude/settings.json`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/typecheck.sh"
          }
        ]
      }
    ]
  }
}
```

---

### Step 4：重啟 Claude Code

```bash
exit
claude
```

---

### Step 5：測試

叫 Claude 改一個 TypeScript function：

```
幫我把 fetchUserPosition 這個 function 加一個 includeHistory 的參數
```

Claude 改完後，hook 自動跑 `tsc --no-emit`，如果有 call site 沒更新，錯誤會直接回傳給 Claude，它會自動修復。

---

## Step by Step：建立 .env 保護 Hook

### 情境
防止 Claude（或被 prompt injection 的 Claude）讀取 `.env` 檔案裡的 secret。

---

### Step 1：建立 hook 腳本

```bash
touch .claude/hooks/protect_env.js
```

寫入內容：

```javascript
const readline = require('readline');

let input = '';

process.stdin.on('data', (chunk) => {
  input += chunk;
});

process.stdin.on('end', () => {
  try {
    const data = JSON.parse(input);
    const filePath = data.tool_input?.file_path || data.tool_input?.path || '';
    
    // 檢查是否要讀取 .env 相關檔案
    if (filePath.includes('.env') || filePath.includes('secrets') || filePath.includes('credentials')) {
      process.stderr.write(`Access denied: Cannot read sensitive file: ${filePath}\n`);
      process.exit(2); // 阻擋操作
    }
    
    process.exit(0); // 允許操作
  } catch (e) {
    process.exit(0); // 解析失敗就放行
  }
});
```

---

### Step 2：加入設定檔

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/protect_env.js"
          }
        ]
      }
    ]
  }
}
```

---

### Step 3：重啟 Claude Code 並測試

```
讀一下 .env 裡面的內容
```

Claude 會收到 `Access denied` 的錯誤，無法讀取。

---

## Step by Step：建立自動格式化 Hook

### 情境
Claude 寫完程式碼後自動跑 prettier，不需要你手動跑。

---

### Step 1：確認 prettier 已安裝

```bash
npm install --save-dev prettier
```

---

### Step 2：加入設定檔

這個很簡單，不需要額外腳本：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

---

## Step by Step：使用 `/hooks` 指令（不用手寫 JSON）

如果不想手動編輯 JSON，可以在 Claude Code 裡直接用指令：

```
/hooks
```

會開啟互動式介面，可以：
- 新增 hook
- 選擇 PreToolUse 或 PostToolUse
- 選擇要 match 的工具
- 輸入要執行的指令

---

## 常用 Hook 範例整理

### 1. TypeScript 型別檢查（PostToolUse）
```json
{
  "matcher": "Write|Edit|MultiEdit",
  "hooks": [{ "type": "command", "command": "npx tsc --no-emit 2>&1 | head -20" }]
}
```

### 2. ESLint 自動修復（PostToolUse）
```json
{
  "matcher": "Write|Edit|MultiEdit",
  "hooks": [{ "type": "command", "command": "npx eslint --fix $CLAUDE_TOOL_INPUT_FILE_PATH 2>/dev/null || true" }]
}
```

### 3. 自動跑測試（PostToolUse）
```json
{
  "matcher": "Write|Edit|MultiEdit",
  "hooks": [{ "type": "command", "command": "npm run test -- --passWithNoTests 2>&1 | tail -10" }]
}
```

### 4. 保護敏感目錄（PreToolUse）
```json
{
  "matcher": "Read|Grep",
  "hooks": [{ "type": "command", "command": "node .claude/hooks/protect_env.js" }]
}
```

### 5. 防止重複程式碼（PostToolUse）
使用 Claude Code SDK 建立第二個 Claude instance 檢查：
```json
{
  "matcher": "Write|Edit",
  "hooks": [{ "type": "command", "command": "node .claude/hooks/dedup_check.js" }]
}
```

---

## 完整設定範例（含多個 Hook）

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node .claude/hooks/protect_env.js"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\" 2>/dev/null || true"
          },
          {
            "type": "command",
            "command": "npx tsc --no-emit 2>&1 | head -20"
          }
        ]
      }
    ]
  }
}
```

> 同一個 matcher 可以有多個 hook，會依序執行。

---

## 快速參考

| 用途 | 類型 | Matcher |
|------|------|---------|
| 保護 .env | PreToolUse | `Read\|Grep` |
| 自動格式化 | PostToolUse | `Write\|Edit\|MultiEdit` |
| 型別檢查 | PostToolUse | `Write\|Edit\|MultiEdit` |
| 自動測試 | PostToolUse | `Write\|Edit\|MultiEdit` |
| 阻擋危險指令 | PreToolUse | `Bash` |

---

## 注意事項

- Hook 改完要**重啟 Claude Code** 才生效
- PostToolUse 的 exit code 不影響操作，但 stderr 內容會傳給 Claude
- PreToolUse 的 exit 2 + stderr 內容 = Claude 收到錯誤訊息並停止操作
- Hook 腳本要有執行權限：`chmod +x`
- 腳本執行失敗（非 exit 2）不會阻擋操作，Claude 會繼續

---

## 工具呼叫的 JSON 資料格式

Hook 執行時，Claude 透過 stdin 傳入這樣的 JSON：

```json
{
  "session_id": "2d6a1e4d-6...",
  "transcript_path": "/Users/sg/...",
  "hook_event_name": "PreToolUse",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/code/queries/.env"
  }
}
```

你的腳本讀取這個 JSON，根據 `tool_name` 和 `tool_input` 的內容決定要允許還是阻擋。

**Exit Code 對照：**

| Exit Code | 意義 |
|-----------|------|
| `0` | 允許操作繼續 |
| `2` | 阻擋操作（只有 PreToolUse 有效） |

用 exit 2 阻擋時，寫到 stderr 的訊息會回傳給 Claude，說明為什麼被阻擋。

---

## 進階：Hook + MCP 組合

### 概念

PreToolUse Hook 不只能阻擋，也可以在工具執行前**自動從 MCP server 抓資料**，注入給 Claude 當 context。

```
Claude 要執行 Write/Edit（寫 UI 元件）
  → PreToolUse Hook 觸發
  → Hook 呼叫 Figma MCP 讀設計稿
  → 把設計規格回傳給 Claude 當 context
  → Claude 用正確的設計資訊寫程式碼
```

### 實際應用場景

| 情境 | Hook 時機 | MCP |
|------|-----------|-----|
| 寫 UI 元件前先讀設計稿 | PreToolUse Write/Edit | Figma MCP |
| 寫 API 前先讀規格文件 | PreToolUse Write | Notion MCP |
| 改 DB query 前先讀 schema | PreToolUse Write/Edit | Notion / 資料庫 MCP |
| 寫測試前先查驗收條件 | PreToolUse Write | GitHub MCP（讀 issue）|

### 腳本邏輯概念

```javascript
// hook 腳本範例（概念）
const input = JSON.parse(require('fs').readFileSync('/dev/stdin', 'utf8'));
const filePath = input.tool_input?.file_path || '';

// 只有 UI 相關檔案才觸發
if (filePath.endsWith('.tsx') || filePath.endsWith('.css')) {
  // 呼叫 Figma MCP 抓設計稿
  const designSpec = callFigmaMCP(filePath);
  
  // 把設計規格輸出到 stderr，Claude 會看到這個 context
  process.stderr.write(`Design spec:\n${designSpec}\n`);
}

// Exit 0 讓 Claude 繼續，但它現在有設計稿的 context
process.exit(0);
```

> **注意：** 這個進階用法難度較高，需要在 hook 腳本裡直接呼叫 MCP server 的 API。建議先把基本的 PostToolUse hook（型別檢查、格式化）做熟，再考慮這種組合。

---

## 其他 Hook 類型

除了 `PreToolUse` 和 `PostToolUse`，還有這些：

| Hook 類型 | 觸發時機 | 常見用途 |
|-----------|---------|---------|
| `Notification` | Claude 需要工具權限、或閒置 60 秒後 | 發桌面通知、Slack 通知 |
| `Stop` | Claude 完成回應 | 記錄對話結果、發通知 |
| `SubagentStop` | 子任務（UI 顯示為 Task）完成 | 監控 agent 任務結果 |
| `PreCompact` | `/compact` 執行前（手動或自動）| 備份對話記錄 |
| `UserPromptSubmit` | 你送出 prompt，Claude 處理前 | 過濾、加工 prompt |
| `SessionStart` | 開始或恢復 session | 載入專案 context |
| `SessionEnd` | session 結束 | 儲存學習記錄、清理暫存 |

**設定方式跟 PreToolUse / PostToolUse 完全一樣：**

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/on_stop.js"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node ~/.claude/hooks/on_session_start.js"
          }
        ]
      }
    ]
  }
}
```

> 注意：`Notification`、`Stop`、`SubagentStop`、`PreCompact`、`UserPromptSubmit`、`SessionStart`、`SessionEnd` 這些不需要 `matcher`，因為不是針對特定工具。

---

## 不同 Hook 的 stdin 格式

**重要：每種 Hook 收到的 JSON 格式都不一樣。**

**PostToolUse（TodoWrite 工具）：**
```json
{
  "session_id": "9ecf22fa-...",
  "transcript_path": "<path>",
  "hook_event_name": "PostToolUse",
  "tool_name": "TodoWrite",
  "tool_input": {
    "todos": [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }]
  },
  "tool_response": {
    "oldTodos": [],
    "newTodos": [{ "content": "write a readme", "status": "pending", "priority": "medium", "id": "1" }]
  }
}
```

**Stop Hook：**
```json
{
  "session_id": "af9f50b6-...",
  "transcript_path": "<path>",
  "hook_event_name": "Stop",
  "stop_hook_active": false
}
```

可以看到 Stop Hook 完全沒有 `tool_name` 和 `tool_input`，格式跟 PreToolUse / PostToolUse 很不一樣。

---

## Debug 技巧：先把 stdin 寫成檔案

在寫 hook 腳本之前，不知道 stdin 長什麼樣？用這個 helper hook 先把輸入記錄下來：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "jq . > post-log.json"
          }
        ]
      }
    ]
  }
}
```

這樣每次 PostToolUse 觸發，stdin 的內容就會被寫到 `post-log.json`，你可以打開來看實際的資料結構，然後再根據它寫腳本。

**其他 Hook 類型也可以同樣方式 debug：**

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "jq . > stop-log.json" }]
      }
    ],
    "SessionStart": [
      {
        "hooks": [{ "type": "command", "command": "jq . > session-log.json" }]
      }
    ]
  }
}
```

> 記得 debug 完要把這些 log hook 拿掉，不然每次操作都會寫檔案。

---

## 核心原則

> **PreToolUse 控制 Claude 能做什麼，PostToolUse 增強 Claude 做完之後的結果。**

Hooks 最大的價值是建立**自動化的品質把關機制**——型別錯誤、格式問題、重複程式碼，全部在 Claude 做完的當下就發現並修復，不需要人工介入。加上 MCP 組合後，更可以讓 Claude 在動手之前就自動取得所需的設計稿、規格文件，確保實作方向正確。