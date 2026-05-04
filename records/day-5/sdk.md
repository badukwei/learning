# Claude Code SDK 學習筆記

---

## 什麼是 Claude Code SDK

> **注意：** 原本叫 `Claude Code SDK`，現在已改名為 `Claude Agent SDK`。如果你看到舊的文章寫 `@anthropic-ai/claude-code`，就是同一個東西。

Claude Agent SDK 讓你**在自己的程式或腳本裡程式化地執行 Claude Code**，跟你在 terminal 裡用的是完全一樣的 Claude Code，有相同的工具、相同的能力，只是改成用程式碼來驅動。

**三種使用方式：**

| 方式 | 適合 |
|------|------|
| **TypeScript SDK** | Node.js 應用、hooks 腳本、web 後端 |
| **Python SDK** | 自動化腳本、資料處理、CI/CD |
| **CLI（`claude -p`）** | shell script、git hooks、簡單自動化 |

---

## 安裝

**TypeScript：**
```bash
npm install @anthropic-ai/claude-code
```

**Python：**
```bash
pip install claude-agent-sdk
```

---

## 基本用法

### TypeScript

```typescript
import { query } from "@anthropic-ai/claude-code";

const prompt = "查看 ./src/queries 目錄裡有沒有重複的 query";

for await (const message of query({ prompt })) {
  console.log(JSON.stringify(message, null, 2));
}
```

### Python

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage

async def main():
  async for message in query(
    prompt="查看 ./src/queries 目錄裡有沒有重複的 query"
  ):
    if isinstance(message, AssistantMessage):
      for block in message.content:
        if hasattr(block, 'text'):
          print(block.text)

asyncio.run(main())
```

### CLI（`claude -p`）

```bash
# 最簡單的用法
claude -p "幫我找出 src/ 裡所有的 TODO 註解"

# 指定輸出格式
claude -p "分析這個 codebase 的架構" --output-format json

# 允許特定工具
claude -p "跑測試並修復失敗的項目" \
  --allowedTools "Bash(npm run test *),Edit,Read"
```

---

## 理解輸出格式

SDK 回傳的是一個 async iterator，每次迭代是 Claude 的一條訊息，包含：

- Claude 的思考過程
- 工具呼叫（讀了哪個檔案、跑了什麼指令）
- 工具回傳的結果
- 最後的完整回應

**最後一條訊息才是最終結果：**

```typescript
let finalResponse = "";

for await (const message of query({ prompt })) {
  // 篩選出最終回應
  if (message.type === "result") {
    finalResponse = message.result;
  }
}

console.log(finalResponse);
```

---

## 權限設定

**預設：只有讀取權限**（Read、Grep、LS）

**開放寫入權限：**

```typescript
for await (const message of query({
  prompt,
  options: {
    allowedTools: ["Read", "Edit", "Write", "Bash"]
  }
})) {
  console.log(message);
}
```

**Python：**

```python
async for message in query(
  prompt=prompt,
  options=ClaudeAgentOptions(
    allowed_tools=["Read", "Edit", "Write", "Bash"],
    permission_mode="acceptEdits"  # 自動接受檔案修改
  )
):
  ...
```

**常用工具清單：**

| 工具 | 說明 |
|------|------|
| `Read` | 讀取檔案 |
| `Write` | 寫入新檔案 |
| `Edit` | 編輯現有檔案 |
| `Bash` | 執行 shell 指令 |
| `Grep` | 搜尋檔案內容 |
| `Glob` | 搜尋符合 pattern 的檔案 |
| `WebSearch` | 網路搜尋 |
| `WebFetch` | 抓取網頁內容 |

---

## 載入 CLAUDE.md 和專案設定

預設 SDK 不會載入你的 `CLAUDE.md` 和 `.claude/settings.json`。加上 `settingSources` 才會：

```typescript
for await (const message of query({
  prompt,
  options: {
    settingSources: ["user", "project"],  // 載入 ~/.claude/ 和 ./.claude/
    cwd: "/path/to/your/project"          // 指定專案目錄
  }
})) {
  console.log(message);
}
```

```python
options=ClaudeAgentOptions(
  setting_sources=["user", "project"],
  cwd="/path/to/your/project"
)
```

**`settingSources` 的選項：**
- `"user"` → 載入 `~/.claude/`（全域設定、CLAUDE.md）
- `"project"` → 載入 `./.claude/`（專案設定、CLAUDE.md）
- `"local"` → 載入 `./.claude/settings.local.json`

---

## 加入 System Prompt

讓 SDK 執行的 Claude 有特定的角色或規範：

```typescript
for await (const message of query({
  prompt: "Review utils.py for bugs",
  options: {
    systemPrompt: "你是一個 TypeScript 專家，遵循 Scallop 的 coding 規範。",
    allowedTools: ["Read", "Grep"]
  }
})) {
  console.log(message);
}
```

---

## 實際應用場景

### 1. 在 Hook 裡啟動第二個 Claude 做 Code Review

這是最常見的 SDK 用途之一（就是前面 Hooks 筆記裡的 dedup_check.js）：

```typescript
import { query } from "@anthropic-ai/claude-code";

// 在 PostToolUse hook 裡，用 SDK 啟動另一個 Claude 做 review
const response = query({
  prompt: `檢查這段新程式碼有沒有跟現有程式碼重複：\n${newCode}`,
  options: {
    maxTurns: 1,  // 只需要一次回應
    allowedTools: ["Read"]
  }
});

for await (const message of response) {
  // 處理回應...
}
```

---

### 2. Git Pre-commit Hook：自動 Review 變更

```bash
# .git/hooks/pre-commit
#!/bin/bash

# 取得 staged 的改動
DIFF=$(git diff --cached)

# 用 claude -p 做快速 review
REVIEW=$(echo "$DIFF" | claude -p "Review 這些改動，如果有明顯問題回傳 BLOCK，否則回傳 OK" \
  --allowedTools "Read" \
  --output-format text)

if echo "$REVIEW" | grep -q "BLOCK"; then
  echo "❌ Claude 發現問題，commit 被阻擋"
  echo "$REVIEW"
  exit 1
fi

echo "✅ Code review 通過"
exit 0
```

---

### 3. CI/CD Pipeline：自動修復測試

```typescript
import { query } from "@anthropic-ai/claude-code";

async function fixFailingTests() {
  for await (const message of query({
    prompt: "跑測試，找出失敗的項目並修復",
    options: {
      allowedTools: ["Bash", "Read", "Edit"],
      permissionMode: "acceptEdits"
    }
  })) {
    if (message.type === "result") {
      console.log("修復完成：", message.result);
    }
  }
}
```

---

### 4. 自動產生文件

```bash
# 掃描整個 src/ 目錄，為每個 function 補上 JSDoc 註解
claude -p "幫 src/ 目錄裡所有沒有 JSDoc 的 function 補上文件" \
  --allowedTools "Read,Edit,Glob" \
  --output-format text
```

---

### 5. 定期自動 Code Quality 檢查

```typescript
// scripts/daily-check.ts
import { query } from "@anthropic-ai/claude-code";

const checks = [
  "找出所有 any 型別並建議修復方式",
  "找出可能的 memory leak",
  "找出沒有 error handling 的 async function"
];

for (const check of checks) {
  console.log(`\n=== ${check} ===`);
  
  for await (const message of query({
    prompt: check,
    options: {
      allowedTools: ["Read", "Grep", "Glob"],
      cwd: "./src"
    }
  })) {
    if (message.type === "result") {
      console.log(message.result);
    }
  }
}
```

---

## CLI 進階用法

```bash
# 從 stdin 傳入內容
cat my_file.ts | claude -p "幫這個檔案加上型別"

# JSON 格式輸出（程式化處理用）
claude -p "列出所有 TODO 的位置" --output-format json

# 限制工具權限（最小原則）
claude -p "分析架構" --allowedTools "Read,Grep,Glob"

# 指定工作目錄
claude -p "Review 這個模組" --cwd "./src/lending"
```

---

## 快速參考

| 用途 | 方式 |
|------|------|
| 腳本 / CI | CLI（`claude -p`）|
| Hook 腳本 | TypeScript SDK |
| Python 自動化 | Python SDK |
| 預設權限 | Read-only |
| 開放寫入 | `allowedTools: ["Edit", "Write"]` |
| 載入專案設定 | `settingSources: ["user", "project"]` |
| 只要最終結果 | 篩選 `message.type === "result"` |

---

## 注意事項

- SDK 執行的 Claude 會消耗 API token，複雜任務費用會累積
- 預設 read-only，開放寫入權限要謹慎
- 在 hook 裡用 SDK 啟動第二個 Claude instance 會讓每次操作變慢，只在必要時才用
- `settingSources` 沒設的話，CLAUDE.md 和 hooks 都不會被載入

---

## 核心原則

> **SDK 把 Claude Code 從互動工具變成可程式化的 AI 引擎——任何需要「AI 判斷 + 程式碼操作」的自動化流程，都可以用 SDK 來建立。**