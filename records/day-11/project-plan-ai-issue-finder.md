# Side Project 2 — AI Issue Finder

日期：2026-03-28（景點網站完成後開始，預計 Day 15+）

---

## 專案背景

解決「怎麼找適合自己貢獻的 OSS issue」的問題。
輸入 GitHub repo URL，自動抓 open issues，用 AI 判斷哪些適合自己做。

---

## 功能

- 輸入 GitHub repo URL
- 自動抓 open issues（GitHub API）
- 用 Claude API 判斷每個 issue 的：
  - 難度（easy / medium / hard）
  - 所需技能
  - 是否已有人認領
  - 是否適合貢獻
- 輸出推薦清單，排序由最適合到最不適合

---

## 技術方向

| 層 | 選擇 |
|----|------|
| GitHub 資料 | GitHub REST API（抓 issues） |
| AI 判斷 | Claude API（Anthropic SDK） |
| 介面 | CLI 優先，之後可加簡單網頁 |

---

## 執行計畫（初步）

1. CLI 可輸入 repo URL，抓出 open issues
2. 每個 issue 送進 Claude，取得結構化評估結果
3. 輸出推薦清單（終端機顯示）
4. 之後有需要再包成網頁

---

## MVP 範圍（不做的事）

- 多 repo 批次分析
- 用戶設定（技能偏好）
- 儲存歷史記錄
- 網頁 UI（CLI 優先）

---

## 求職價值

- 展示 AI 整合能力（Claude API / Anthropic SDK）
- 是 developer tool，符合目標公司類型（dev tools、AI 產品）
- 解決真實問題，面試時 story 好說
- 對自己馬上有用（找 OSS issue 用）
