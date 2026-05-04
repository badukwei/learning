# Side Project 1 — 景點清單共享網站

日期：2026-03-28

---

## 專案背景

朋友反映 Google Maps 沒有「分享景點清單」的功能。
目標：做一個任何人都可以建立清單、新增 / 刪除景點的協作網站。
不需要登入，沒有權限管理，開放給所有人編輯。

範例清單：
- 台北市適合大便的廁所
- 台中市適合一個人哭的地方

---

## 技術選擇

| 層 | 選擇 | 理由 |
|----|------|------|
| Frontend | Next.js 15 (App Router) + Tailwind | 熟悉的技術棧，部署快 |
| 資料庫 | Supabase (PostgreSQL) | 免費、不需自架 API、即時同步 |
| 部署 | Vercel | 一鍵部署，免費 |
| 地圖 | 不嵌入 | 每個景點放 Google Maps 連結即可 |

---

## DB Schema

```sql
-- 清單
create table lists (
  id         uuid primary key default gen_random_uuid(),
  name       text not null,
  region     text,
  created_at timestamptz default now()
);

-- 景點
create table spots (
  id              uuid primary key default gen_random_uuid(),
  list_id         uuid references lists(id) on delete cascade,
  name            text not null,
  google_maps_url text not null,
  note            text,
  created_at      timestamptz default now()
);
```

---

## 頁面結構

```
/              首頁：所有清單 + 建立新清單
/[listId]      清單頁：景點列表 + 新增景點
```

### 首頁 `/`
- 所有清單的卡片（名稱、地區、景點數量）
- 「建立新清單」按鈕 → modal 輸入名稱 + 地區

### 清單頁 `/[listId]`
- 清單名稱 + 地區
- 景點列表：名稱、備註、「在 Google Maps 開啟」連結、刪除按鈕
- 新增景點表單：名稱 + Google Maps 連結 + 備註（選填）

---

## 專案檔案結構

```
src/
├── app/
│   ├── page.tsx               # 首頁
│   ├── [listId]/
│   │   └── page.tsx           # 清單頁
│   └── layout.tsx
├── components/
│   ├── ListCard.tsx           # 首頁清單卡片
│   ├── CreateListModal.tsx    # 建立清單 modal
│   ├── SpotItem.tsx           # 單一景點列
│   └── AddSpotForm.tsx        # 新增景點表單
└── lib/
    └── supabase.ts            # Supabase client
```

---

## 執行計畫

### Day 12 — 初始化 + 首頁
1. `npx create-next-app` 初始化專案
2. Supabase 建專案，執行上面的 SQL 建兩張 table
3. 設定 `supabase.ts`，確認可以讀寫
4. 做出首頁：列出清單 + 建立清單 modal

### Day 13 — 清單頁
1. 景點列表顯示
2. 新增景點表單
3. 刪除景點功能

### Day 14 — 收尾
1. 刪除清單功能
2. 基本 RWD（手機可用就好）
3. 部署到 Vercel

---

## MVP 範圍（不做的事）

- 搜尋功能
- 留言 / 按讚
- 標籤分類
- 圖片上傳
- 用戶帳號
- 手機 app

---

## 求職價值

- 展示 fullstack 能力（Next.js + Supabase）
- 有真實需求 + 產品感（quirky 定位好記）
- 適合放進履歷作為 side project
