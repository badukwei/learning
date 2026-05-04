# Day 24 - System Design

Date: 2026-04-14

## 今日主題

Session 管理：Sticky Sessions vs Redis Session vs JWT vs Short-lived JWT + Refresh Token

---

## Sticky Sessions 還存在的原因

JWT 更現代，但 Sticky Sessions 仍存在因為：

1. **遺留系統**：老 PHP/Java 系統改 JWT 成本高，Sticky Sessions 不改 code 就能水平擴展
2. **JWT 無法即時撤銷**：封號後 Token 還沒過期仍可操作，要撤銷就得加 Redis Blacklist
3. **JWT Payload 肥大**：每次 Request 都帶在 Header，Session ID 只有幾 bytes
4. **WebSocket**：長連線不帶 HTTP Header，Sticky Sessions 天然適合

---

## 現代架構選擇

| 場景 | 推薦 |
|------|------|
| 新專案、API-first、微服務 | JWT |
| 傳統 Web App（有登出、封號需求）| Redis Session |
| SPA / 行動 App | Short-lived JWT + Refresh Token |
| WebSocket / 遊戲 | Sticky Sessions |
| 金融、高安全需求 | Redis Session + Short-lived JWT |

- **JWT** 和 **Redis** 通常分開用，不是一起
- 「Redis + JWT」只在需要 Blacklist 撤銷時才組合，但 JWT 無狀態優勢會削弱

---

## Short-lived JWT + Refresh Token

### 兩種 Token 分工

| | Access Token | Refresh Token |
|---|---|---|
| 形式 | JWT | 隨機字串 |
| 有效期 | 短（15 分鐘）| 長（7-30 天）|
| 用途 | 每次 API 請求驗證身份 | 只用來換新 Access Token |
| 存放 | 前端（localStorage / Cookie）| Server 端（DB 或 Redis）|
| 可撤銷 | 否（等過期）| 是 |

### 流程

```
登入 → 拿到 Access Token（15min）+ Refresh Token（30天）

正常使用：
Request → 帶 Access Token → OK

Access Token 過期：
Request → 401 → 自動用 Refresh Token 換新 Access Token → 重試

封號 / 登出：
刪掉 Server 端的 Refresh Token → 下次換 Token 時失敗 → 強制登出
```

### 為什麼這樣好

- Access Token 短命 → 被偷也只有 15 分鐘
- Refresh Token 可撤銷 → 解決 JWT 無法即時登出的問題
- 用戶體驗好 → 自動靜默換 Token，感覺一直登著

> OAuth 2.0 也是這套邏輯，目前最主流的 Auth 架構。
