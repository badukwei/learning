# agent.md

這份文件描述 agent 在這個 repo 內的使用方式，目的是讓後續進來協作的 agent 可以快速理解資料結構、更新流程與紀錄規則。

## Repo Purpose

這是一個個人學習與求職追蹤 repo，主軸有兩條：

- 學習紀錄：每天的技術學習與筆記
- 求職規劃：投遞策略、每週計劃、LinkedIn、coffee chat、進度回顧

目標：

- 2026-08 前拿到遠端海外 offer
- 目標年薪 `70k+ USD`

## Directory Overview

- `README.md`：repo 最簡短總覽
- `CLAUDE.md`：給 Claude Code 的精簡操作提示
- `PROGRESS.md`：最高優先級的進度追蹤檔
- `day-N/`：每日學習筆記
- `job-search/`：求職策略、週計劃、後續回顧

## Source Of Truth

如果是規劃、追蹤、回報進度，優先讀：

1. `PROGRESS.md`
2. `job-search/` 相關文件

如果只是要了解某天具體學了什麼，再去讀對應的 `day-N/`。

不要在沒有必要時把所有 `day-N/` 全部讀過一遍。

## What Goes Where

### `PROGRESS.md`

用途：

- 記錄高層級進度
- 記錄最近完成事項
- 記錄求職方向、策略與階段目標

適合寫入：

- 今天/本週完成了什麼
- 求職方向調整
- 新的週重點

不適合寫入：

- 太細的貼文草稿
- 冗長的學習內容
- 大量 outreach 明細

### `day-N/`

用途：

- 每天的學習筆記
- 技術主題拆解
- 題目、系統設計、工具學習紀錄

命名習慣：

- 一天一個資料夾，例如 `day-30/`
- 檔名直接反映主題，例如 `redis.md`、`auth-flow.md`

### `job-search/`

用途：

- 記錄求職策略與規劃
- 管理每週目標
- 後續可擴充 LinkedIn 與 coffee chat 素材

目前已有：

- `README.md`
- `application-strategy.md`
- `week-2026-05-04.md`

後續若新增，也應優先放在這裡，例如：

- `linkedin-content-plan.md`
- `coffee-chat-outreach.md`
- `application-review.md`

## Agent Workflow

### 當使用者要問本週計劃 / 今日任務

- 先讀 `PROGRESS.md`
- 再讀 `job-search/` 中最近的週計劃
- 根據目前日期給出具體可打勾的任務

### 當使用者回報做了什麼

- 直接更新 `PROGRESS.md`
- 如果屬於求職策略或當週規劃，也同步更新 `job-search/` 對應文件

### 當使用者要調整求職方向

- 先以 `PROGRESS.md` 的方向為基準
- 若方向有變，更新 `PROGRESS.md`
- 若是策略細化，寫進 `job-search/`

### 當使用者只是做技術學習

- 新增或更新對應的 `day-N/` 筆記
- 只有在這件事影響整體節奏時，才補到 `PROGRESS.md`

## Writing Rules

- 優先寫具體、可執行、可追蹤的內容
- 任務要能打勾，避免空泛描述
- 同一件事不要同時寫在太多地方
- 高層進度進 `PROGRESS.md`
- 細節內容進 `day-N/` 或 `job-search/`

## Practical Rule

如果不確定該寫哪裡，先用這個判斷：

- 這是整體方向或進度嗎？寫 `PROGRESS.md`
- 這是技術學習細節嗎？寫 `day-N/`
- 這是求職策略、投遞、LinkedIn、coffee chat 嗎？寫 `job-search/`
