# Claude Code — Custom Commands 學習筆記

---

## 什麼是 Custom Commands

Claude Code 內建了 `/compact`、`/clear`、`/init` 等指令，但你也可以建立自己的指令。

Custom Commands 的本質是：**一個放在特定位置的 markdown 檔案，裡面寫的就是你要 Claude 執行的指示。**

你把重複做的事情寫成一個指令，之後一行搞定。

---

## 資料夾結構

```
你的專案/
└── .claude/
    └── commands/
        ├── audit.md          → /audit
        ├── write_tests.md    → /write_tests
        └── review.md         → /review
```

檔名 = 指令名稱，就這麼簡單。

---

## Step by Step：建立第一個 Custom Command

### 情境
你在 Scallop 的 DeFi 專案裡，每次新功能做完都要手動跑安全性檢查 + 跑測試 + 確認 TypeScript 沒有型別錯誤。每次都要打好幾行指令，很煩。

現在把它變成一個 `/check` 指令。

---

### Step 1：確認 `.claude` 資料夾存在

```bash
# 在專案根目錄
ls -la | grep .claude
```

如果沒有，建立它：

```bash
mkdir -p .claude/commands
```

---

### Step 2：建立指令檔案

```bash
touch .claude/commands/check.md
```

---

### Step 3：寫指令內容

打開 `check.md`，寫入：

```markdown
請依序執行以下檢查：

1. 跑 TypeScript 型別檢查：`tsc --no-emit`
   - 如果有型別錯誤，列出所有錯誤並修復，修完再繼續

2. 跑測試：`npm run test`
   - 如果有測試失敗，分析原因並修復，修完再繼續

3. 跑安全性檢查：`npm audit`
   - 如果有 high 或 critical 等級的漏洞，嘗試用 `npm audit fix` 修復

4. 完成後給我一份摘要：
   - 型別錯誤：X 個（已修復 / 未修復）
   - 測試：X 通過 / X 失敗
   - 安全漏洞：X 個（已修復 / 需手動處理）
```

---

### Step 4：重啟 Claude Code

```bash
# 完全退出再重新開啟
# 或在 Claude Code 裡按 Ctrl+C 退出，再重新執行 claude
```

> ⚠️ 這步不能省，新指令不重啟不會生效。

---

### Step 5：使用指令

```
/check
```

Claude 會依序執行所有檢查，遇到問題自動修復，最後給你摘要。

---

## Step by Step：建立帶參數的 Custom Command

### 情境
你在 Scallop 專案裡，每次寫新的 hook 或 utility function 都要補測試。但每次都要告訴 Claude「用 Vitest、放在 `__tests__` 目錄、用這個命名規範...」，很重複。

把這些規範寫進指令，之後只要說要測試哪個檔案就好。

---

### Step 1：建立指令檔案

```bash
touch .claude/commands/write_tests.md
```

---

### Step 2：寫指令內容（含 `$ARGUMENTS`）

```markdown
請為以下目標撰寫完整測試：$ARGUMENTS

## 測試規範

**框架與工具**
- 使用 Vitest + React Testing Library
- Import 路徑使用 `@/` 前綴

**檔案位置**
- 測試檔案放在與源檔案相同目錄的 `__tests__/` 資料夾
- 命名規則：`[filename].test.ts` 或 `[filename].test.tsx`

**測試涵蓋範圍**
- Happy path（正常流程）
- Edge cases（邊界條件）
- Error states（錯誤處理）
- 如果是 React component，包含使用者互動測試

**Scallop 專案特別注意**
- Mock 所有外部 API 呼叫
- DeFi 相關的數值計算要測試精度
- 涉及 Sui wallet 的操作要 mock wallet state
```

---

### Step 3：重啟 Claude Code

---

### Step 4：使用方式

```bash
# 基本用法
/write_tests hooks/use-lending-position.ts

# 更詳細的描述也可以
/write_tests components/LiquidationWarning.tsx，這個元件在 position health 低於 1.2 時會顯示警告

# 甚至可以給多個檔案
/write_tests hooks 目錄裡所有跟 position 相關的 hooks
```

`$ARGUMENTS` 就是你在 `/write_tests` 後面打的所有內容，Claude 會把它代進去。

---

## 更多實用的 Custom Commands 範例

### `/review` — Code Review

```markdown
請對以下進行 code review：$ARGUMENTS

檢查重點：
- 有沒有潛在的 security 問題
- 有沒有 performance 問題
- 命名是否清楚
- 有沒有重複的邏輯可以抽出來
- TypeScript 型別有沒有可以更嚴格的地方

輸出格式：
- 🔴 必須修：[問題描述]
- 🟡 建議修：[問題描述]  
- 🟢 做得好：[優點]
```

用法：
```
/review 今天新增的 liquidation bot 邏輯
/review @src/hooks/use-oracle-price.ts
```

---

### `/commit` — 自動產生 Commit Message

```markdown
分析目前 git diff 的改動，產生一個符合 Conventional Commits 規範的 commit message。

格式：`type(scope): description`

type 選項：feat / fix / refactor / docs / test / chore
scope：受影響的模組或功能

要求：
- description 用英文，動詞開頭
- 如果改動超過一個面向，用 bullet points 列出 body
- 不超過 72 個字元（第一行）

產生後直接執行 git commit。
```

用法：
```
/commit
```

---

## 快速參考

| 步驟 | 動作 |
|------|------|
| 1 | `mkdir -p .claude/commands` |
| 2 | 建立 `命令名稱.md` |
| 3 | 寫入指示，用 `$ARGUMENTS` 接受參數 |
| 4 | 重啟 Claude Code |
| 5 | `/命令名稱 [參數]` |

---

## 核心原則

> **把你重複說的話寫成指令，把你的規範寫進 markdown，讓 Claude 每次都按你的標準做事。**

Custom Commands 最大的價值不是省打字，而是**讓團隊的規範變成可執行的東西**——不需要靠人記，也不會因為忘記說而讓 Claude 做出不符規範的東西。