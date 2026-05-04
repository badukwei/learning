# Claude Code — .env 保護 Hook 實作指南

---

## 目標

建立一個 PreToolUse Hook，當 Claude 嘗試讀取 `.env` 檔案時自動阻擋，防止敏感資料外洩。

---

## 運作原理

```
Claude 要執行 Read 或 Grep
  → PreToolUse Hook 攔截
  → 檢查目標檔案路徑有沒有 .env
  → 有的話 → exit 2 → 阻擋，Claude 收到錯誤訊息
  → 沒有的話 → exit 0 → 正常繼續
```

---

## Step by Step

### Step 1：建立 hooks 資料夾

```bash
mkdir -p .claude/hooks
```

---

### Step 2：建立 hook 腳本

```bash
touch .claude/hooks/read_hook.js
```

寫入以下內容：

```javascript
async function main() {
  // 從 stdin 讀取 Claude 傳來的工具呼叫資訊
  const chunks = [];
  for await (const chunk of process.stdin) {
    chunks.push(chunk);
  }

  const toolArgs = JSON.parse(Buffer.concat(chunks).toString());

  // 抓出 Claude 要讀取的檔案路徑
  const readPath =
    toolArgs.tool_input?.file_path || toolArgs.tool_input?.path || "";

  // 檢查是否要讀取 .env 檔案
  if (readPath.includes(".env")) {
    console.error("You cannot read the .env file");
    process.exit(2); // 阻擋操作，錯誤訊息回傳給 Claude
  }
}

main().catch(() => process.exit(0));
```

---

### Step 3：設定 Hook 設定檔

打開 `.claude/settings.local.json`，加入 hooks 設定。

> ⚠️ **重要：一定要用絕對路徑，不要用相對路徑。**
> 相對路徑有路徑攔截和惡意程式植入的安全風險。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node /Users/linwgpeter/dev/your-project/.claude/hooks/read_hook.js"
          }
        ]
      }
    ]
  }
}
```

**重點說明：**
- `matcher: "Read|Grep"` — `|` 是 OR，同時監控 Read 和 Grep 兩種工具
- `command` — 用絕對路徑指向腳本，確保安全

---

### 什麼是 `$PWD`？

`$PWD`（Print Working Directory）是 shell 的環境變數，代表你**目前所在的目錄的絕對路徑**。

```bash
# 在 ~/dev/my-project 目錄下
echo $PWD
# 輸出：/Users/linwgpeter/dev/my-project
```

所以 `$PWD/.claude/hooks/read_hook.js` 就等於 `/Users/linwgpeter/dev/my-project/.claude/hooks/read_hook.js`。

---

### 團隊共用設定的問題與解法

**問題：** 絕對路徑因人而異，`settings.local.json` 沒辦法直接 commit 給團隊用，因為每個人的路徑不一樣。

**解法：** 提供一個 `settings.example.json` 範本，裡面用 `$PWD` 作為佔位符：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read|Grep",
        "hooks": [
          {
            "type": "command",
            "command": "node $PWD/.claude/hooks/read_hook.js"
          }
        ]
      }
    ]
  }
}
```

然後在 `package.json` 的 `setup` script 裡，執行一個初始化腳本把 `$PWD` 換成實際路徑：

```javascript
// scripts/init-claude.js
const fs = require('fs');
const path = require('path');

const projectRoot = process.cwd();
const examplePath = path.join(projectRoot, '.claude', 'settings.example.json');
const outputPath = path.join(projectRoot, '.claude', 'settings.local.json');

// 讀取範本
let content = fs.readFileSync(examplePath, 'utf8');

// 把 $PWD 替換成實際的絕對路徑
content = content.replace(/\$PWD/g, projectRoot);

// 輸出為 settings.local.json
fs.writeFileSync(outputPath, content);
console.log(`Created settings.local.json with path: ${projectRoot}`);
```

```json
// package.json
{
  "scripts": {
    "setup": "npm install && node scripts/init-claude.js"
  }
}
```

**這樣就能兩全其美：**
- `settings.example.json` → commit 進 git，團隊共用
- `settings.local.json` → 加進 `.gitignore`，每個人在自己機器上產生，包含各自的絕對路徑

---

### 拉了遠端 repo 之後要做什麼

當你或隊友 clone 這個 repo 之後，需要做以下步驟：

**Step 1：Clone repo**
```bash
git clone https://github.com/xxx/your-project.git
cd your-project
```

**Step 2：跑 setup 產生 settings.local.json**
```bash
npm run setup
```

這會自動把 `settings.example.json` 裡的 `$PWD` 替換成你機器上的實際路徑，產出 `settings.local.json`。

**Step 3：確認有沒有產生成功**
```bash
cat .claude/settings.local.json
```

應該看到裡面的路徑已經是絕對路徑，不是 `$PWD`。

**Step 4：重啟 Claude Code**
```bash
exit
claude
```

> ⚠️ **每次 clone 新的 repo 都要重新跑 `npm run setup`**，因為 `settings.local.json` 在 `.gitignore` 裡，不會跟著 clone 下來。

---

### Step 4：重啟 Claude Code

```bash
exit
claude
```

> ⚠️ Hook 改完一定要重啟才生效。

---

### Step 5：測試

在 Claude Code 裡說：

```
讀一下 .env 的內容
```

Claude 會收到這個錯誤：

```
You cannot read the .env file
```

它會告訴你操作被 hook 阻擋了，無法讀取。

**Grep 也一樣被保護：**

```
在 .env 裡搜尋 DATABASE_URL
```

同樣會被阻擋。

---

## 理解 Hook 收到的資料

Claude 執行工具時，會透過 stdin 傳入這樣的 JSON：

```json
{
  "session_id": "2d6a1e4d-6...",
  "transcript_path": "/Users/peter/...",
  "hook_event_name": "PreToolUse",
  "tool_name": "Read",
  "tool_input": {
    "file_path": "/Users/peter/project/.env"
  }
}
```

腳本從 `tool_input.file_path` 或 `tool_input.path` 拿到路徑，然後判斷要不要阻擋。

---

## 延伸：保護更多敏感檔案

把 `.env` 的判斷擴展成更完整的清單：

```javascript
const SENSITIVE_PATTERNS = [
  '.env',
  'secrets',
  'credentials',
  'private_key',
  '.pem',
  '.p12',
  '.key',
  'keystore',
  'mnemonic',
  '.npmrc',
  '.zsh_history',
  '.bash_history',
];

const isSensitive = SENSITIVE_PATTERNS.some(pattern =>
  readPath.toLowerCase().includes(pattern.toLowerCase())
);

if (isSensitive) {
  console.error(`Access denied: Cannot read sensitive file: ${readPath}`);
  process.exit(2);
}
```

---

## 專案檔案結構

```
your-project/
├── .claude/
│   ├── hooks/
│   │   └── read_hook.js          ← hook 腳本（commit 進去）
│   ├── settings.example.json     ← 範本，commit 進去，用 $PWD 佔位
│   └── settings.local.json       ← 加進 .gitignore，每人自己產生
├── scripts/
│   └── init-claude.js            ← setup 腳本（commit 進去）
└── package.json
```

---

## 快速參考

| 項目 | 內容 |
|------|------|
| Hook 類型 | PreToolUse（工具執行前攔截）|
| Matcher | `Read\|Grep` |
| 腳本位置 | `.claude/hooks/read_hook.js` |
| 設定範本 | `.claude/settings.example.json`（commit）|
| 實際設定 | `.claude/settings.local.json`（gitignore）|
| 阻擋方式 | `process.exit(2)` + `console.error()` |
| 允許方式 | 什麼都不做，腳本自然結束（exit 0）|
| Clone 後要做 | `npm run setup` 產生 `settings.local.json` |

---

## 核心原則

> **exit 2 + stderr = 阻擋操作並告訴 Claude 原因**
> **exit 0 = 允許操作繼續**

> **settings.example.json（$PWD 佔位）→ commit；settings.local.json（絕對路徑）→ gitignore**