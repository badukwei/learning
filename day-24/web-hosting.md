# Day 24 - Web Hosting & Scalability

Date: 2026-04-14

---

## 基本功能評估標準

### 1. SFTP（必要）

- FTP：明碼傳輸，username/password 在網路上裸奔
- SFTP：加密傳輸，保護憑證安全
- 現代 PaaS 直接 git push，SFTP 已不需要

### 2. 「無限方案」的陷阱

- 無限流量 / 無限儲存 / 無限 RAM → 通常是共用主機（Shared Host）
- 實質：數百個客戶擠在同一台實體機，資源互搶
- 正式業務不應依賴 Shared Host

### 3. 存取權限與環境獨立性

| 類型 | 說明 | 隱私程度 |
|------|------|----------|
| Shared Host | 多人共用一台機器 | 低 |
| VPS | 透過 Hypervisor 切割，有獨立 OS（Ubuntu / Fedora）| 中 |
| 實體伺服器 | 完全獨立，代管商無法存取 | 高 |

> VPS 注意：代管商系統管理員若有實體存取權，技術上可透過單用戶模式進入你的機器。

### 4. 彈性擴展（Elastic Scaling）

- 網站爆紅 → 自動生成更多 VM（Spawn）
- 流量退去 → 自動關機
- 按分鐘計費，不用為尖峰流量長期付費

### 5. 網路可達性

- 部分代管商 IP 段在某些國家 / 公司網路被封鎖
- 上線前要測試目標用戶能否連到你的服務

---

## 現代服務商比較

### 重要功能對照表

| 功能 | Vercel | Railway/Render | Supabase | DigitalOcean | AWS | Cloudflare |
|------|--------|----------------|----------|--------------|-----|------------|
| HTTPS 自動 | ✅ | ✅ | ✅ | App Platform ✅ / Droplet 要設 | ✅ via ALB | ✅ |
| 加密傳輸（取代 SFTP）| git push | git push | SDK | SSH | SSH | — |
| 資源獨立性 | 共用（Serverless）| 容器隔離 | 共用 | VPS ✅ | EC2 ✅ | Edge |
| 彈性擴展 | 自動 ✅ | 有限 | 有限 | 手動 / K8s | 自動 ✅ | 自動 ✅ |
| DDoS 防護 | 基本 | 基本 | 基本 | 基本 | AWS Shield | 強 ✅ |
| 環境變數管理 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| DB 加密 at rest | — | — | ✅ | 要自設 | RDS ✅ | — |
| SQL Injection 防護 | — | — | SDK ✅ / RLS ✅ | — | — | — |
| Rate Limiting | 需自加 | 需自加 | 有限 | 需自加 | 要設定 | ✅ |
| WAF | — | — | — | — | AWS WAF（付費）| 付費 ✅ |

---

### 平台自動幫你處理的

```
HTTPS / TLS 加密   → 部署就有（Vercel、Railway、Render、Supabase）
DDoS 基本防護      → Cloudflare 掛上去就有
DB 加密 at rest    → Supabase、AWS RDS 預設有
SQL Injection      → 用 ORM / SDK 正常呼叫就有（Prisma、Supabase SDK）
```

### 需要自己設定的

```
環境變數 / Secret  → 平台有介面，但要自己填
CORS              → 自己寫
Rate Limiting     → 自己加或靠 Cloudflare
```

### 平台完全不管的

```
程式碼安全         → XSS、自己拼 SQL → 你的責任
JWT secret 強度   → 你自己設
GDPR 合規         → 你自己處理
```

---

## SQL Injection 防護原則

ORM / SDK 預設有保護，但前提是用正確方式：

```js
// ✅ 安全：Prisma 自動 parameterized query
const user = await prisma.user.findUnique({ where: { id: userId } })

// ❌ 危險：自己拼字串，沒有保護
const query = `SELECT * FROM users WHERE id = ${userId}`
await db.query(query)
```

---

## 選擇代管的優先順序

1. 加密通訊（HTTPS 自動）
2. 資源分配的真實性（不買「無限」空話）
3. 系統獨立性（VPS 起步，或用容器隔離的 PaaS）
4. 彈性擴展能力（能因應突發流量）
5. 安全工具（DDoS、WAF、Rate Limiting）

---

## 實際選擇建議

| 場景 | 推薦組合 |
|------|----------|
| Side project / 個人專案 | Vercel + Supabase |
| 小型新創後端 | Railway + Supabase |
| 需要完整控制 | DigitalOcean VPS |
| 大流量 / 企業 | AWS + Cloudflare |
