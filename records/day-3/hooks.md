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

## 核心原則

> **PreToolUse 控制 Claude 能做什麼，PostToolUse 增強 Claude 做完之後的結果。**

Hooks 最大的價值是建立**自動化的品質把關機制**——型別錯誤、格式問題、重複程式碼，全部在 Claude 做完的當下就發現並修復，不需要人工介入。