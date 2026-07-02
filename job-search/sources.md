# 職缺來源與抓取方式

更新：2026-07-01

目前第一目標是取得面試邀約，所以來源選擇以「可找到足夠可投職缺」與「能追蹤回覆率」為主。

## 來源分層

| 來源 | 主要用途 | 分類 | 建議方式 |
|---|---|---|---|
| 104 | 台灣本土正職 | `taiwan` | 手動搜尋優先；可評估用公開搜尋頁輔助整理 |
| Cake | 台灣 / 亞洲 / 遠端職缺 | `taiwan` / `overseas` | 手動搜尋優先；適合找新創與英文職缺 |
| LinkedIn | 海外與台灣補充來源、networking | `taiwan` / `overseas` | 手動搜尋與 saved search；不把爬蟲當主線 |
| 公司官網 | 高品質精投 | `taiwan` / `overseas` | 手動確認，優先官方 JD |
| Remote job boards | 海外遠端補量 | `overseas` | 手動搜尋或公開頁面整理 |
| Third-party APIs | 快速產生候選 lead | `taiwan` / `overseas` | 小量測試，需標記 provider 與風險 |

## LinkedIn 使用原則

LinkedIn 適合當作：

- 搜尋職缺與建立 saved search。
- 看公司、recruiter、hiring manager。
- 補 LinkedIn outreach / direct contact 訊號。
- 驗證某職缺是否仍開放。

LinkedIn 不適合當作：

- 自動大量爬職缺的主要來源。
- 用 Playwright 登入後批次抓取搜尋結果。
- 用非官方 API 抓職缺資料。

原因：

- LinkedIn 官方 API 產品有 Talent / Job Posting 相關能力，但主要面向招聘方、ATS、合作夥伴，不是一般求職者可直接使用的公開職缺搜尋 API。
- LinkedIn User Agreement 明確限制使用 crawlers、browser plugins、bots 或其他自動化方式 scrape / copy LinkedIn 服務內容。

因此 LinkedIn 的策略是「手動搜尋 + saved search + outreach」，不是自動化抓取。

## 第三方 API / Scraper 工具

可以評估第三方 API，但要把它們當成「lead discovery 工具」，不是官方資料來源。

常見類型：

- `Apify actors`：有 LinkedIn Jobs Scraper、Google Jobs Scraper、Indeed Scraper 等現成 actor，可以用 API 匯出 JSON / CSV。
- `SERP APIs`：例如 Google Jobs 搜尋結果 API，用搜尋引擎結果建立候選清單。
- `Job search APIs`：例如 RapidAPI 上的 job search 類 API，常整合多個來源。
- `Data providers`：例如 Bright Data 這類資料供應商，可能提供 Web Scraper API 或預建資料集，但通常成本較高。

可評估工具：

| 工具 | 類型 | 適合用途 | 初步判斷 |
|---|---|---|---|
| Apify | Actor / scraper marketplace | LinkedIn / Google Jobs / Indeed 候選 lead 匯出 | 已使用過，可優先延續 |
| Bright Data | Web data provider / scraper API / datasets | 大量資料或預建資料集 | B 開頭最可能是這個；偏貴，先不作主力 |
| Browse AI | No-code scraper / monitor | 監控特定公開頁面變化 | 可小量測試，不適合當主資料源 |
| Browserless | Browser automation API | 自己寫 Playwright / Puppeteer 流程 | 需要自己維護流程，LinkedIn 不建議 |
| RapidAPI job APIs | API marketplace | 快速測資料品質 | 品質差異大，只適合小量測試 |

使用規則：

- 只用來產生候選 lead，投遞前仍要打開原始 JD 或公司官網人工確認。
- 每筆 lead 的 `source` 寫原始來源，例如 `linkedin`，並在 `notes` 補 provider，例如 `provider=apify`。
- 不使用需要提供 LinkedIn 帳密、cookie、繞過驗證碼或大量模擬帳號行為的工具。
- 不自動投遞、不自動傳訊息、不自動加人。
- 小量測試即可，目標是節省整理時間，不是建立大型資料庫。
- 若 provider 的資料品質差、職缺過期率高、或成本過高，就停用。

第三方 API 評估欄位：

```text
provider	target_source	cost	auth_required	output_fields	risk	notes
```

`risk` 建議值：

- `low`：公開搜尋結果或官方 API，無帳號自動化。
- `medium`：第三方 scraper，但不需要個人帳號 / cookie。
- `high`：需要登入 cookie、帳密、繞驗證或模擬帳號行為；原則上不用。

## Playwright 使用原則

Playwright 可以用在：

- 有公開搜尋頁、沒有登入需求、沒有明確禁止自動化的來源。
- 幫忙開頁面、擷取自己手動可見的職缺資訊。
- 做半自動整理，例如把搜尋結果頁的公司、職稱、URL 整成候選清單後再人工確認。

Playwright 不應用在：

- 繞過登入、驗證碼、rate limit、地區限制。
- 模擬帳號行為大量抓 LinkedIn。
- 自動送出申請或自動傳訊息。

## 每筆 lead 必填來源

`leads.tsv` 至少包含：

```text
source	company	title	location	work_type	url	status	notes
```

`source` 建議值：

- `104`
- `cake`
- `linkedin`
- `company-site`
- `remoteok`
- `wellfound`
- `himalayas`
- `third-party-api`
- `other`

第三方工具的 `notes` 建議格式：

```text
provider=apify
provider=bright-data
provider=browse-ai
provider=browserless
provider=rapidapi
```

`status` 建議值：

- `new`
- `shortlisted`
- `applied`
- `replied`
- `interview`
- `rejected`
- `closed`

## 目前優先順序

台灣：

1. `104`
2. `Cake`
3. `LinkedIn`
4. 公司官網

海外：

1. 公司官網
2. LinkedIn 手動搜尋 / saved search
3. Cake 遠端 / 英文職缺
4. Remote job boards
