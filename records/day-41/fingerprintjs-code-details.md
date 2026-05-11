# Day 41 - FingerprintJS Code Details

Date: 2026-05-11

這份筆記只看程式碼細節，聚焦兩件事：

1. 它怎麼抓這些指紋
2. 它怎麼 hash 成 `visitorId`

對照的主要檔案：

- `src/agent.ts`
- `src/sources/index.ts`
- `src/utils/entropy_source.ts`
- `src/utils/hashing.ts`
- `src/sources/fonts.ts`
- `src/sources/audio.ts`
- `src/sources/webgl.ts`
- `src/sources/canvas.ts`

---

## 一、它怎麼抓這些指紋

## 先看整體流程

入口在 `src/agent.ts`：

```ts
export async function load(options: Readonly<LoadOptions> = {}): Promise<Agent> {
  const { delayFallback, debug, monitoring = true } = options
  if (monitoring) {
    monitor()
  }
  await prepareForSources(delayFallback)
  const getComponents = loadBuiltinSources({ cache: {}, debug })
  return makeAgent(getComponents, debug)
}
```

這段做了三件事：

1. 先做 `prepareForSources()`
2. 建立 `loadBuiltinSources(...)`
3. 回傳一個 agent，等你呼叫 `get()` 時才真正收集 components

所以不是 `load()` 當下就把所有指紋全抓完，而是：

```text
load()
  -> 準備 sources
  -> 建立 getter
get()
  -> 真正收集 components
```

---

## 為什麼要先 `prepareForSources()`

`prepareForSources()` 本質上是在等一小段 idle 時間：

```ts
export function prepareForSources(delayFallback = 50): Promise<void> {
  return requestIdleCallbackIfAvailable(delayFallback, delayFallback * 2)
}
```

原因不是效能而已，重點是穩定性。

有些 browser signal 在頁面剛載入時還不穩，例如：

- layout 還沒定
- 字型載入狀態還在變
- 某些 rendering / browser API 結果可能飄

對 fingerprint 來說：

**晚一點抓到穩定值，比早一點抓到不穩定值重要。**

---

## `src/sources/index.ts`：所有 entropy source 的總表

真正有哪些指紋，是在 `src/sources/index.ts` 這個 object 定義的：

```ts
export const sources = {
  userAgentData: getUserAgentData,
  fonts: getFonts,
  domBlockers: getDomBlockers,
  fontPreferences: getFontPreferences,
  audio: getAudioFingerprint,
  screenFrame: getScreenFrame,
  canvas: getCanvasFingerprint,
  osCpu: getOsCpu,
  languages: getLanguages,
  colorDepth: getColorDepth,
  deviceMemory: getDeviceMemory,
  screenResolution: getScreenResolution,
  hardwareConcurrency: getHardwareConcurrency,
  timezone: getTimezone,
  sessionStorage: getSessionStorage,
  localStorage: getLocalStorage,
  indexedDB: getIndexedDB,
  openDatabase: getOpenDatabase,
  cpuClass: getCpuClass,
  platform: getPlatform,
  plugins: getPlugins,
  touchSupport: getTouchSupport,
  vendor: getVendor,
  vendorFlavors: getVendorFlavors,
  cookiesEnabled: areCookiesEnabled,
  ...
  webGlBasics: getWebGlBasics,
  webGlExtensions: getWebGlExtensions,
}
```

這代表 FingerprintJS 的做法不是：

- 讀一個超神奇欄位

而是：

- 同時收集很多中小型訊號
- 最後再把它們組起來

---

## `loadBuiltinSources()` 沒有直接抓值，而是先包裝 sources

在 `src/sources/index.ts` 最後：

```ts
export default function loadBuiltinSources(options: BuiltinSourceOptions): () => Promise<BuiltinComponents> {
  return loadSources(sources, options, [])
}
```

關鍵在 `loadSources()`，它在 `src/utils/entropy_source.ts`。

這層 abstraction 很重要，因為它把每個 source 都統一包成：

```ts
type Component<T> =
  | { value: T, duration: number }
  | { error: unknown, duration: number }
```

也就是每個 source 最後都會變成：

- 成功值 `value`
- 或失敗 `error`
- 再加上花多少時間 `duration`

這樣設計的好處：

- 某個 source 壞掉，不會拖垮整體 fingerprint
- 同步 / 非同步 source 都能用同一套流程
- 可以保留 debug 與 timing 資訊

---

## `loadSource()` 支援兩段式 source

`entropy_source.ts` 最核心的地方，是 source 不一定一次完成。

source 的型別是：

```ts
type Source<TOptions, TValue> =
  (options: TOptions) => MaybePromise<TValue | (() => MaybePromise<TValue>)>
```

意思是 source 可以回傳兩種東西：

1. 直接回傳結果
2. 先回傳一個 function，之後再真正取值

這對 `audio` 這種需要先 setup、再 finish rendering 的 source 很重要。

---

## 具體看幾個 source

## 1. `languages.ts`

這是最簡單的一類，直接讀 browser API：

```ts
const language = n.language || n.userLanguage || n.browserLanguage || n.systemLanguage
...
if (Array.isArray(n.languages)) {
  result.push(n.languages)
}
```

它抓的是：

- `navigator.language`
- `navigator.languages`

這類 source 的特色是：

- 簡單
- 便宜
- entropy 不高
- 但組起來有價值

---

## 2. `user_agent_data.ts`

這個 source 讀的是 Chromium 系的 `navigator.userAgentData`：

```ts
const uaData = (navigator as unknown as { userAgentData?: NavigatorUAData }).userAgentData
...
const highEntropy = await uaData.getHighEntropyValues(['architecture', 'bitness', 'model', 'platformVersion'])
```

這裡抓的包含：

- brands
- mobile
- platform
- architecture
- bitness
- model
- platformVersion

有一個細節值得記：

它會過濾 `GREASE` brand，也就是 browser 故意加的假 brand。

也就是這段：

```ts
function isGreaseBrand(brand: string): boolean {
  return /not/i.test(brand)
}
```

原因是如果把故意加的隨機干擾一起算進去，指紋反而會更不穩。

---

## 3. `fonts.ts`

字型偵測不是去讀「系統字型清單」，
因為 browser 本來就不會直接把完整字型名單公開給你。

它的做法是：

1. 先準備三個 base font
   - `monospace`
   - `sans-serif`
   - `serif`
2. 建立一批 `span`
3. 對每個待測 font，套用：
   - `'字型名', baseFont`
4. 比較 `offsetWidth` / `offsetHeight` 是否和 base font 一樣

核心判斷：

```ts
fontSpans[baseFontIndex].offsetWidth !== defaultWidth[baseFont] ||
fontSpans[baseFontIndex].offsetHeight !== defaultHeight[baseFont]
```

意思是：

- 如果指定字型不存在，browser 會 fallback 到 base font
- 尺寸就會跟 base font 一樣
- 如果存在，render 結果通常會不同

所以它不是「讀出字型名單」，
而是：

**用渲染差異反推出哪些字型存在。**

另外它特別把整件事包在 iframe 裡跑：

```ts
return withIframe(async (_, { document }) => { ... })
```

這是為了避免被頁面自己的 CSS 汙染，也避免影響原本頁面外觀。

---

## 4. `canvas.ts`

Canvas fingerprint 的做法是：

1. 建一個 canvas
2. 用固定字型、固定顏色、固定圖形去畫
3. `toDataURL()` 轉成字串
4. 把這個字串當成 fingerprint component

它分成兩種圖：

- text image
- geometry image

而且它會做穩定性檢查：

```ts
const textImage1 = canvasToString(canvas)
const textImage2 = canvasToString(canvas)

if (textImage1 !== textImage2) {
  return [ImageStatus.Unstable, ImageStatus.Unstable]
}
```

原因是有些 browser 會故意對 canvas 加 noise。
如果兩次 encode 都不同，就不能拿它當穩定指紋。

---

## 5. `audio.ts`

Audio fingerprint 不是抓麥克風，而是用 `OfflineAudioContext` 在本地生成一段音訊，
再看結果長什麼樣。

核心流程：

1. 建立 `OfflineAudioContext`
2. 建 `Oscillator`
3. 建 `DynamicsCompressor`
4. 把 oscillator 接到 compressor，再接到 destination
5. `startRendering()`
6. 取出 buffer 的一小段資料
7. 算出一個數值

程式碼片段：

```ts
const context = new AudioContext(1, hashToIndex, 44100)
const oscillator = context.createOscillator()
const compressor = context.createDynamicsCompressor()
...
renderPromise.then((buffer) => getHash(buffer.getChannelData(0).subarray(hashFromIndex)))
```

最後的 `getHash()` 很簡單：

```ts
function getHash(signal: ArrayLike<number>): number {
  let hash = 0
  for (let i = 0; i < signal.length; ++i) {
    hash += Math.abs(signal[i])
  }
  return hash
}
```

這個 source 的重點不是加密，而是：

- 相同環境下通常會有相近輸出
- 不同環境下輸出可能不同

另外它會避開已知 anti-fingerprinting 環境，例如某些 Safari / Samsung Internet 版本。

---

## 6. `webgl.ts`

WebGL 是高 entropy source。

它會先拿到 WebGL context：

```ts
const canvas = document.createElement('canvas')
context = canvas.getContext('webgl') as CanvasContext
```

然後抓很多 GPU / driver / WebGL capability 資訊，例如：

- `VERSION`
- `VENDOR`
- `RENDERER`
- `SHADING_LANGUAGE_VERSION`
- supported extensions
- shader precision
- context attributes
- 各種 `getParameter(...)` 值

像這段：

```ts
version: gl.getParameter(gl.VERSION)?.toString() || '',
vendor: gl.getParameter(gl.VENDOR)?.toString() || '',
renderer: gl.getParameter(gl.RENDERER)?.toString() || '',
```

這類資訊對辨識很有幫助，因為：

- GPU
- driver
- browser WebGL stack

都會反映在結果上。

---

## 所以「抓指紋」的本質是什麼

程式碼層面上，它做的事情可以總結成三種：

### 1. 直接讀 browser / navigator 欄位

例如：

- language
- platform
- memory
- hardwareConcurrency

### 2. 讀 browser capability

例如：

- storage 是否可用
- cookies 是否可用
- WebGL extension 有哪些
- PDF viewer 是否可用

### 3. 主動執行一個 deterministic 操作，再觀察輸出

例如：

- canvas render
- audio render
- font render

這第三類通常 entropy 較高，也是更難純靠改 `navigator` 造假的部分。

---

## 二、它怎麼 hash 成 `visitorId`

這段主流程在 `src/agent.ts`。

## 1. 先把所有 components canonicalize

關鍵函式：

```ts
function componentsToCanonicalString(components: UnknownComponents) {
  let result = ''
  for (const componentKey of Object.keys(components).sort()) {
    const component = components[componentKey]
    const value = 'error' in component ? 'error' : JSON.stringify(component.value)
    result += `${result ? '|' : ''}${componentKey.replace(/([:|\\])/g, '\\$1')}:${value}`
  }
  return result
}
```

這段很重要，因為 fingerprint 不能直接把 JS object 拿去 hash。

必須先變成一個穩定字串。

它做了三件事：

### a. `Object.keys(...).sort()`

先把 key 排序，避免 object 順序不同導致 hash 不同。

### b. value 用 `JSON.stringify`

例如：

- array
- object
- string
- number

都統一轉成穩定字串。

### c. 特殊字元 escape

它會把 key 裡的 `:`, `|`, `\` escape 掉：

```ts
componentKey.replace(/([:|\\])/g, '\\$1')
```

因為最後整包字串是用：

- `|`
- `:`

拼起來的，如果不 escape，邊界就可能被破壞。

---

## 2. 字串長什麼樣

概念上會變成像這樣：

```text
audio:123.45|canvas:{"winding":true,"geometry":"...","text":"..."}|languages:[["en-US"],["en-US","zh-TW"]]|platform:"MacIntel"
```

這就是 canonical string。

接著才會拿去 hash。

---

## 3. `hashComponents()`

`agent.ts` 裡很直接：

```ts
export function hashComponents(components: UnknownComponents): string {
  return x64hash128(componentsToCanonicalString(components))
}
```

也就是：

```text
components
  -> canonical string
  -> x64hash128(...)
  -> visitorId
```

---

## 4. `x64hash128` 用的是什麼

`src/utils/hashing.ts` 用的是 MurmurHash3 x64 128-bit 版本。

不是 crypto hash，例如：

- 不是 SHA-256
- 不是 bcrypt
- 不是 HMAC

它比較像：

- 快速
- 穩定
- 非加密用途
- 適合大量資料做 fingerprint / lookup key

---

## 5. `x64hash128` 內部在做什麼

大方向是標準 MurmurHash3 流程：

1. 把 input string 轉成 UTF-8 bytes
2. 每 16 bytes 一組做 block mixing
3. 不滿 16 bytes 的尾巴做 tail handling
4. 把長度混進去
5. 做 final mix (`fmix`)
6. 輸出 128-bit hex string

對應程式碼：

```ts
const key = getUTF8Bytes(input)
const remainder = length[1] % 16
const bytes = length[1] - remainder
const h1 = [0, seed]
const h2 = [0, seed]
```

中間每個 block 會做：

- multiply
- rotate left
- xor
- add

也就是你在檔案裡看到的：

- `x64Multiply`
- `x64Rotl`
- `x64Xor`
- `x64Add`

最後尾端：

```ts
x64Xor(h1, length)
x64Xor(h2, length)
x64Add(h1, h2)
x64Add(h2, h1)
x64Fmix(h1)
x64Fmix(h2)
x64Add(h1, h2)
x64Add(h2, h1)
```

最後輸出 32 hex chars x 4 段：

```ts
return (
  ('00000000' + (h1[0] >>> 0).toString(16)).slice(-8) +
  ('00000000' + (h1[1] >>> 0).toString(16)).slice(-8) +
  ('00000000' + (h2[0] >>> 0).toString(16)).slice(-8) +
  ('00000000' + (h2[1] >>> 0).toString(16)).slice(-8)
)
```

這就是最後的 `visitorId` 字串。

---

## 6. 為什麼不用加密 hash

因為這裡的需求不是密碼學安全，而是：

- 同樣輸入要穩定得到同樣輸出
- 要夠快
- 要能當 lookup key
- 要固定長度

所以 MurmurHash 這種 non-cryptographic hash 比較適合。

這也再次說明：

**`visitorId` 的目的不是防止逆向，而是把一整包 fingerprint components 壓成一個穩定識別碼。**

---

## 總結

如果只從程式碼角度看，FingerprintJS 做的事可以縮成兩句：

### 怎麼抓指紋

它透過很多 entropy source 去：

- 讀 browser / navigator 資訊
- 讀 capability
- 主動做 render / execution，再觀察輸出差異

### 怎麼 hash

它先把所有 component key 排序、value stringify、組成 canonical string，
再用 MurmurHash3 x64 128-bit 算成固定長度的 `visitorId`。

所以本質上是：

```text
many signals
  -> normalized components
  -> canonical string
  -> murmur hash
  -> visitorId
```
