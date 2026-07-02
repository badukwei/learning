# 求職結果紀錄

更新：2026-07-01

這個資料夾現在只放 **找工作結果與驗證資料**，不再承擔計劃與策略。

## 原則

- `job-search/` 只記錄求職結果
- 2026-07 起，新的求職紀錄先分成 `overseas/` 與 `taiwan/`
- 每個批次資料夾都有一個 `README.md` 當主索引
- `tsv -> checklist -> opportunities` 是固定三層
- 目前第一目標是取得面試邀約，所以每批都要追蹤是否有 recruiter 回覆 / 面試邀約
- 每筆 lead 都要記錄來源，避免混淆不同平台的搜尋品質

## 目前分類

- [overseas/README.md](/Users/linwgpeter/dev/learning/learning-records/job-search/overseas/README.md)：海外遠端、全球職缺、合約與 AI coding evaluation
- [taiwan/README.md](/Users/linwgpeter/dev/learning/learning-records/job-search/taiwan/README.md)：台灣本土學習型正職
- [sources.md](/Users/linwgpeter/dev/learning/learning-records/job-search/sources.md)：職缺來源與抓取方式

## 新增批次命名

海外職缺：

- `job-search/overseas/YYYY-MM-DD/README.md`
- `job-search/overseas/YYYY-MM-DD/leads.tsv`
- `job-search/overseas/YYYY-MM-DD/checklist.md`
- `job-search/overseas/YYYY-MM-DD/opportunities.md`

台灣職缺：

- `job-search/taiwan/YYYY-MM-DD/README.md`
- `job-search/taiwan/YYYY-MM-DD/leads.tsv`
- `job-search/taiwan/YYYY-MM-DD/checklist.md`
- `job-search/taiwan/YYYY-MM-DD/opportunities.md`

`leads.tsv` 建議欄位：

```text
source	company	title	location	work_type	url	status	notes
```

## 歷史日期資料夾

以下是 2026-07 分流前的歷史紀錄，暫時保留原位置：

- [2026-05-07/README.md](/Users/linwgpeter/dev/learning/learning-records/job-search/2026-05-07/README.md)
- [2026-05-19/README.md](/Users/linwgpeter/dev/learning/learning-records/job-search/2026-05-19/README.md)
- [opportunities-2026-06-23.md](/Users/linwgpeter/dev/learning/learning-records/job-search/opportunities-2026-06-23.md)

## 策略與計劃的新位置

- 投遞策略：`plans/strategy/application-strategy.md`
- 每週規劃：`plans/weekly/`
