# AGENTS.md

這個 repo 只放個人技術學習紀錄。

## Directory Overview

- `README.md`：repo 總覽
- `CLAUDE.md`：給 Claude Code 的精簡操作提示
- `PROGRESS.md`：學習進度 dashboard / index
- `records/day-N/`：每日學習筆記

## Source Of Truth

學習進度、最近完成事項、重要索引優先讀 `PROGRESS.md`。

如果要了解某天具體學了什麼，再讀對應的 `records/day-N/`。

不要在沒有必要時把所有 `records/day-N/` 全部讀過一遍。

## What Goes Where

### `PROGRESS.md`

用途：

- 記錄學習高層進度
- 記錄最近完成的學習事項
- 作為重要學習筆記索引

### `records/day-N/`

用途：

- 每天的學習筆記
- 技術主題拆解
- 題目、系統設計、工具學習紀錄


## Agent Workflow

### 當使用者要問學習進度 / 今日學習任務

- 先讀 `PROGRESS.md`
- 需要細節時再讀最近的 `records/day-N/README.md`
- 給出具體可打勾的學習任務

### 當使用者回報學了什麼

- 新增或更新對應的 `records/day-N/` 筆記
- 只有在影響整體學習節奏或值得索引時，才補到 `PROGRESS.md`
