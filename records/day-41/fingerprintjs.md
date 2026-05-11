# Day 41 - FingerprintJS

Date: 2026-05-11

## 先講結論

`FingerprintJS` 不是直接「擋攻擊」的防護系統。

它真正做的事是：

- 從瀏覽器收集一組相對穩定、可辨識的特徵
- 把這些特徵組成一個 `visitorId`
- 讓上層風控系統拿這個 ID 去做：
  - 多帳號偵測
  - 機器人 / 濫用行為關聯
  - 繞過 cookie 後的訪客辨識
  - 裝置層級的 rate limit / risk scoring

所以它比較像：

**辨識層（identification layer）**

不是：

**決策層（block / allow engine）**

---

## 它的防護原理是什麼

核心原理是：

**同一台裝置 / 同一個瀏覽器環境，雖然 cookie 可以被清掉，但很多執行環境特徵不會同時一起改變。**

例如：

- 字型
- Canvas 渲染結果
- Audio 指紋
- WebGL 能力
- 螢幕解析度
- CPU / memory / concurrency
- 時區、語系
- platform / vendor / plugin / touch support
- storage 能力
- 廣告攔截 / content blocker 痕跡
- 顏色偏好、reduced motion、HDR、Apple Pay 等環境訊號

這些訊號單看不一定唯一，但組合起來後，會形成一個足夠有辨識度的「瀏覽器輪廓」。

接著 FingerprintJS 會把這些 component 轉成 canonical string，再做 `x64hash128`，產生 `visitorId`。

也就是：

```text
many browser/device signals
  -> normalized components
  -> canonical string
  -> hash
  -> visitorId
```

上層服務再把這個 `visitorId` 跟帳號、IP、行為事件綁在一起，做真正的風控判斷。

---

## 完整流程

## 1. `load()`

初始化時，`load()` 會先做兩件事：

- `prepareForSources()`
- `loadBuiltinSources()`

### 為什麼先 delay

在 `src/agent.ts` 裡，作者特別先等一小段 idle 時間，再開始收集 entropy source。

原因是：

- 有些瀏覽器特徵在頁面剛載入時不穩定
- 太早收集，結果可能會飄
- 指紋系統最怕「同一個人每次都算出不同結果」

所以它寧可等一下，換比較穩定的 component。

---

## 2. 載入 entropy sources

`src/sources/index.ts` 定義了所有資料來源。

大方向可分成幾類：

### 環境與平台

- `userAgentData`
- `platform`
- `vendor`
- `osCpu`
- `architecture`

### 裝置能力

- `deviceMemory`
- `hardwareConcurrency`
- `touchSupport`
- `screenResolution`
- `screenFrame`
- `colorDepth`

### 瀏覽器能力 / 功能開關

- `sessionStorage`
- `localStorage`
- `indexedDB`
- `openDatabase`
- `cookiesEnabled`
- `pdfViewerEnabled`
- `privateClickMeasurement`
- `applePay`

### 地區 / 偏好

- `languages`
- `timezone`
- `dateTimeLocale`
- `reducedMotion`
- `reducedTransparency`
- `forcedColors`
- `invertedColors`
- `contrast`
- `hdr`

### 高 entropy 訊號

- `fonts`
- `fontPreferences`
- `audio`
- `audioBaseLatency`
- `canvas`
- `webGlBasics`
- `webGlExtensions`
- `math`

### 攔截器 / 擴充套件痕跡

- `domBlockers`
- `plugins`
- `vendorFlavors`

---

## 3. 每個 source 被包成 component

`src/utils/entropy_source.ts` 做了很重要的事情：

- 每個 source 都會被包成 `{ value | error, duration }`
- 可同步、也可非同步
- 先 load，再在 `get()` 階段真正取值
- 就算某些 source 失敗，也不會整個 fingerprint 掛掉

這個設計代表：

- 指紋不是依賴單一訊號
- 少幾個 component 還是能工作
- 真實世界瀏覽器差異很大，所以容錯很重要

---

## 4. `get()` 收集所有 components

當 `fp.get()` 被呼叫時：

- 所有 source 會被執行
- 取回 component map
- 建立 `GetResult`
- 延遲計算 `visitorId`

這裡有一個小細節：

`visitorId` 是 lazy 計算的。也就是先保留 `components`，等真的讀 `visitorId` 時才 hash。

好處是：

- 初始化成本更低
- debug 時可以先看原始 components

---

## 5. hash 成 `visitorId`

`src/agent.ts` 裡的流程大致是：

```text
components
  -> sort keys
  -> stringify value
  -> escape special chars
  -> join with |
  -> x64hash128
  -> visitorId
```

這裡的重要點是：

- key 要排序，不然同樣資料可能因順序不同算出不同 hash
- 需要 canonical 化，不然序列化格式不同就不穩
- 最後才 hash，避免直接暴露完整原始特徵組合

---

## 為什麼最後要 hash

這裡的 hash 主要不是為了「安全防護」，而是為了把一大包 browser signals 變成一個適合系統使用的識別碼。

可以拆成四個原因。

### 1. 壓成固定格式

原始 fingerprint 是一整包 component map：

- 欄位很多
- 長度不固定
- 結構也不適合直接拿來當 key

hash 之後就能變成固定長度的 `visitorId`，比較適合：

- 存 DB
- 當 index
- 放到 event / log
- 跟其他事件做 join

### 2. 比對成本更低

風控系統最常做的事情其實不是分析整包 raw components，
而是快速回答：

```text
這次是不是跟上次同一個裝置 / 同一個瀏覽器環境？
```

如果每次都逐欄比較整包 component：

- 成本高
- 實作複雜
- 不容易拿來做大量查詢

hash 後就可以直接拿一個短 ID 去查：

- 這個帳號之前有沒有出現過
- 這個 visitorId 關聯過幾個帳號
- 這個 visitorId 最近有沒有大量請求

### 3. 不直接暴露原始特徵

如果直接把原始 component 回傳給上層系統當識別碼，
等於把很多環境細節原樣攤開，例如：

- 字型特徵
- WebGL 資訊
- Audio / Canvas 結果
- 語系、時區、平台資訊

hash 後至少不會把這些值直接裸露成 identifier。

要注意：

這不代表 hash 之後就完全沒有隱私問題，
只是比「直接把整包原始 fingerprint 當 ID」更合理。

### 4. 讓上層系統把它當識別碼使用

上層風控系統真正需要的通常不是 raw JSON，
而是一個可關聯的 key：

- `visitorId`
- account id
- IP
- request id

這樣才能很自然地做：

- 關聯分析
- risk scoring
- rate limit
- 黑名單 / 灰名單

所以 hash 的作用比較像是把：

```text
raw signals -> 可分析資料
hashed id   -> 可使用識別碼
```

---

## 為什麼這能拿來做防護

因為很多濫用者會先做的事是：

- 清 cookie
- 開無痕
- 重註冊帳號
- 換 session

但他不一定會：

- 連瀏覽器底層特徵一起改
- 把 Canvas / WebGL / Audio / 字型特徵一起偽裝乾淨
- 讓環境訊號之間保持一致

所以對風控系統來說，FingerprintJS 的價值是：

### 1. 補 cookie 的盲點

cookie / localStorage 很容易被清掉。

指紋不是存在 client storage，而是「重新從執行環境推導」。

所以在：

- incognito
- 清除瀏覽資料
- 不登入狀態

仍然有機會把同一個訪客串起來。

### 2. 做裝置層關聯

例如：

- 同一個 `visitorId` 短時間註冊很多帳號
- 同一個 `visitorId` 一直領新人優惠
- 同一個 `visitorId` 切很多 email / phone 但環境很像

這些都是典型風控訊號。

### 3. 提供 risk engine 額外維度

真正的防護通常不是單看指紋，而是一起看：

- `visitorId`
- IP / ASN / geo
- 帳號行為
- 滑鼠 / 輸入節奏
- 裝置歷史
- 請求頻率

也就是：

**fingerprint 提供的是 identity hint，不是單獨的 truth source。**

---

## 為什麼它要收這麼多不同種類的訊號

因為單一訊號很脆弱。

例如：

- 螢幕解析度很多人相同
- 時區很多人相同
- 語系很多人相同

但如果把：

- Canvas
- WebGL
- Audio
- Fonts
- deviceMemory
- hardwareConcurrency
- language/timezone/platform

一起看，碰撞率就會下降很多。

所以本質不是「找一個超強特徵」，而是：

**把很多中弱訊號疊成一個較穩定的識別結果。**

---

## 幾個重要 source 的意義

## 1. Canvas Fingerprint

Canvas 會畫特定文字與圖形，再把結果轉成字串。

不同環境在以下地方可能有差異：

- 字型 rasterization
- anti-aliasing
- GPU / driver
- 瀏覽器實作
- 作業系統字型與渲染管線

因此同樣的畫圖指令，不同環境常會輸出不同結果。

### 為什麼它還要檢查穩定性

`src/sources/canvas.ts` 有特別處理：

- 同一張圖 encode 兩次
- 如果結果不一樣，就標成 `unstable`

原因是：

- 有些瀏覽器會故意對 canvas 加噪音
- 如果把不穩定值直接納入指紋，反而會讓同一個使用者每次都變不同人

這是很典型的風控取捨：

**寧可少吃一點 entropy，也不要吃到不穩定 entropy。**

---

## 2. DOM Blockers

`domBlockers` 會檢查哪些 CSS selector 會被 ad blocker / content blocker 擋掉。

這背後的想法是：

- 不同使用者安裝的 blocker 與 filter list 不同
- 這些差異本身就是辨識訊號

也就是說：

不是只看「有沒有裝 ad blocker」，
而是更細地看：

- 擋了哪些 selector
- 這些 selector 對應哪些 filter list

這也是為什麼 docs 裡要維護一整包 filter / selector 對照表。

---

## 3. Audio / WebGL / Fonts / Math

這幾類都屬於高 entropy 訊號。

它們共同的價值是：

- 比純字串欄位更難偽裝
- 會反映硬體、driver、瀏覽器實作差異
- 在大量使用者中更容易拉開差異

尤其 WebGL / Canvas / Audio 這種「執行後輸出結果」的特徵，
通常比單純讀 `navigator.xxx` 更有辨識力。

---

## 為什麼這個設計合理

從工程角度看，這個 repo 有幾個很對的決策。

## 1. 多訊號組合，而不是押單點

單點訊號容易：

- 被關掉
- 被 spoof
- 被瀏覽器改版影響

多 source 組合比較抗脆弱。

## 2. 穩定性優先

他們不只是追求 entropy，也一直在處理：

- source 是否 unstable
- 某些瀏覽器的 anti-fingerprinting 行為
- 載入太早導致的不穩定

這代表作者知道：

**fingerprinting 最難的不是「拿到很多資料」，而是「同一個人下次還算得出差不多的結果」。**

## 3. 失敗可容忍

某些 source 拿不到時，整體還能工作。

這很重要，因為：

- 瀏覽器 API 支援度不同
- 權限不同
- blocker / privacy mode 會改行為

真實世界一定會有缺資料情況。

## 4. client-side 設計很容易接入

它只要前端執行就能拿到 `visitorId`，整合很輕。

對很多產品來說，這是很好的第一步：

- 先接 client fingerprint
- 再決定要不要加 server-side intelligence

---

## 這個方案的限制

repo 自己其實講得很直接：

- accuracy 比商業版低很多
- 純 client-side，所以容易被 reverse engineering 與 spoofing

這點很重要，因為它直接決定你怎麼用它。

### 1. 不能把它當絕對身份

它比較適合：

- risk signal
- duplicate detection hint
- anti-abuse feature

不適合：

- 當唯一登入依據
- 當封鎖使用者的唯一證據

### 2. 會碰到 collision

不同使用者可能會有相似環境，尤其在：

- 同款手機
- 同版瀏覽器
- 相似系統設定

所以 `visitorId` 不是 globally unique truth。

### 3. 會被 anti-fingerprinting 反制

現在瀏覽器越來越主動做隱私保護，例如：

- Safari / Firefox 對 canvas 做保護
- extension 在 incognito 的行為不一致
- 某些 API 被弱化或標準化

也就是：

**瀏覽器廠商本身就在削弱 fingerprinting 的辨識力。**

### 4. 攻擊者可以 spoof

如果對手有足夠動機，他可以：

- 改 `navigator` 欄位
- 用 anti-detect browser
- 攔截 API
- 重放或模擬 component

所以 OSS 版更適合：

- 成本較低的濫用者
- 大量自動化、低到中等強度攻擊

不是高對抗場景的最終答案。

---

## 跟真正「防護產品」差在哪

這個 repo 最值得記住的一點是：

**FingerprintJS 本體只做 client-side identification。**

真正更像防護產品的做法通常還會加上：

- server-side deduplication
- network / TLS / IP reputation signals
- replay / tamper detection
- bot / VPN / proxy detection
- event correlation
- policy engine

也就是：

```text
browser signals
  -> fingerprint / visitor id
  -> backend risk engine
  -> score / rule
  -> allow / challenge / block
```

如果只有最左邊那段，還不算完整防護。

---

## 可能的應用場景

下面這些場景用混合方式整理：

- 先講產品上實際會痛的問題
- 再補一小段技術落地方式

這樣比較接近我之後複習時會真的拿來講的版本。

重點不是「FingerprintJS 單獨解決問題」，
而是：

**它提供裝置層級的關聯訊號，幫風控系統更早把可疑事件串起來。**

---

## 1. KYC 註冊流程中的多帳號 / 羊毛黨偵測

### 產品場景

交易所或金流平台很常遇到這種情況：

- 同一批人反覆註冊新帳號
- 每次都換 email / phone
- 想重拿 onboarding bonus 或 referral 獎勵
- 想在後面做洗量、套利、詐欺或帳戶養號

如果只看帳號資料，這些帳戶彼此看起來都不一樣。
但從平台角度，真正想回答的是：

```text
這些新帳戶是不是其實來自同一批設備或同一個操作者？
```

這就是 fingerprint 的典型價值。

### 技術落地

在以下節點打點：

- 註冊頁開啟
- email / phone 驗證
- KYC start
- KYC submit
- 獎勵 claim

每個事件都帶：

- `visitorId`
- account id / applicant id
- IP / ASN / geo
- event time
- promo / referral id

後端可做的規則例如：

- 24 小時內同一 `visitorId` 建立超過 N 個新帳戶
- 同一 `visitorId` 關聯多個不同姓名 / phone / email
- 同一 `visitorId` 在完成註冊後快速切換多個帳戶進 KYC

對應動作不是直接封鎖，而是：

- 拉高 risk score
- 延後獎勵發放
- 要求額外驗證
- 丟人工審核

---

## 2. KYC 後的帳戶盜用 / 身分冒用風險

### 產品場景

帳戶盜用通常不是一登入就馬上出事。
前面常會先出現一串「不像原使用者」的操作：

- 從陌生裝置登入
- 很快打開 security settings
- 修改密碼、2FA、提現白名單
- 重新進 KYC profile 或個資頁

產品問題不是單純：

```text
這次登入成功沒？
```

而是：

```text
這次登入像不像帳戶原本的持有人？
```

如果一個老帳戶突然出現全新的 `visitorId`，而且後面緊接著敏感操作，
這就是很有價值的早期風險訊號。

### 技術落地

在登入與敏感操作事件打點：

- login success / failure
- reset password
- change 2FA
- change withdrawal settings
- open KYC profile page

每個事件保存：

- account id
- `visitorId`
- IP / geo
- first_seen / last_seen
- action type

可做的規則例如：

- 老帳戶第一次出現陌生 `visitorId`
- 陌生 `visitorId` 登入後短時間內連續修改安全設定
- 同一 `visitorId` 短時間測多個帳號密碼

後端常見處理：

- step-up auth
- 暫停敏感操作
- 提現冷卻期
- 送人工 review

---

## 3. KYC 補件流程的自動化濫用

### 產品場景

KYC 不是永遠一次過，常常會：

- 上傳證件
- 被拒
- 補件
- 重傳

這個流程很容易被自動化腳本濫用，例如：

- 大量試不同身份資料
- 測哪種文件格式最容易過
- 同一批設備重複送很多 applicant

產品真正想知道的是：

```text
這些看似不同申請人，背後是不是同一批設備在操作？
```

### 技術落地

在每次 KYC step 記錄：

- `visitorId`
- applicant id
- document submission id
- retry count
- failure reason
- OCR / face match result

可以做的檢查例如：

- 同一 `visitorId` 對很多 applicant 重複送件
- 同一 `visitorId` 在短時間出現大量 KYC failure
- 同一 `visitorId` 對不同姓名出現高度相似的提交節奏

後端用途：

- 限制重試頻率
- 調整人工審核優先級
- 觸發額外 challenge
- 保留 investigation pivot

---

## 4. 交易所 bot / 活動濫用 / API abuse

### 產品場景

交易所、防詐、成長活動場景常碰到：

- 新帳戶開好就立刻高頻打 API
- 同一批帳戶刷 onboarding bonus
- referral / coupon / airdrop 被重複 claim
- bot 批量註冊後再做 wash trading 或活動套利

單看單一帳戶，很難立刻判斷。
但如果多個可疑帳戶背後其實共享或高度相似的 `visitorId`，
風控就能更早把它們連成一個 cluster。

### 技術落地

在以下事件串上 `visitorId`：

- signup
- login
- KYC status change
- promo claim
- API key create
- sensitive API access

後端可把 `visitorId` 跟這些資料一起做關聯：

- request rate
- endpoint distribution
- account age
- KYC completion state
- reward claim history
- withdrawal destination

規則例如：

- 未完成 KYC 的新帳戶，共享相同 `visitorId` 且高頻打關鍵 endpoint
- 同一 `visitorId` 關聯多個帳戶且都在領獎勵
- 同一 `visitorId` 建很多帳戶後，又集中流向同一資金出口

動作可以是：

- challenge
- hold reward
- freeze 部分能力
- 升級調查

---

## 一個實務上的重要原則

在 KYC / ATO / anti-abuse 場景裡，
FingerprintJS 最適合扮演的是：

- `risk feature`
- `correlation key`
- `investigation pivot`

不適合扮演的是：

- 唯一封鎖依據
- 唯一身份證明
- 單點真相來源

原因很簡單：

- 會有 collision
- 會有 spoofing
- 會被 anti-fingerprinting 影響

所以比較合理的用法是：

```text
fingerprint
  + IP / ASN / geo
  + account history
  + KYC results
  + behavior events
  + security actions
  -> risk score / rule
  -> challenge / hold / review / allow
```

---

## 我目前的理解

可以把 FingerprintJS 想成：

- Session / Cookie 之外的補充身份訊號
- 裝置層級的 weak identity
- 風控系統裡的一個 feature，不是整套系統

它的價值不在於「百分之百認出一個人」，
而在於：

**讓原本完全匿名、可重設、可隨手切換的瀏覽器請求，變得比較可關聯。**

只要可關聯性上升，濫用成本就會上升。

這就是它的防護價值。

---

## 一句話總結

`FingerprintJS` 的本質是：

**把很多瀏覽器 / 裝置執行環境訊號組合成穩定指紋，讓上層風控系統能在沒有 cookie 信任前提下，仍然辨識、關聯、限制可疑訪客。**
