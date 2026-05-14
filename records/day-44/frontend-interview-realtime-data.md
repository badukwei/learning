# Day 44 - 前端面試：高頻即時資料怎麼設計

Date: 2026-05-14

## 一句話版本

這題的核心不是：

**我會用 WebSocket。**

而是：

**我會控制 UI 更新頻率、限制狀態影響範圍、降低渲染成本，並設計好錯誤恢復機制。**

---

## 面試回答版本

如果是價格、交易狀態、live metrics 這種高頻資料，我會優先使用 WebSocket subscription，而不是高頻 polling。因為 WebSocket 可以維持長連線，讓 server 主動推送資料，避免 client 持續輪詢。

但前端不能每收到一筆資料就立刻更新整個 React state，不然很容易造成大量 re-render。所以我通常會先加一層 buffer，然後用 throttle 或固定 flush interval，例如每 500ms 或 1 秒，批次更新一次 UI。

在狀態管理上，我會把資料拆成三層：

- `server state`：用 React Query / TanStack Query 管理 API data、cache、retry、refetch
- `live state`：用 Zustand 或其他 external store 管理 WebSocket 推送資料
- `UI state`：留在 component local state，例如 filter、modal、selected tab

如果列表很大，例如幾千個交易對或 metrics，我會搭配 virtualization，只渲染畫面上看得到的 rows，例如用 TanStack Virtual 或 `react-window`。

最後我會處理 resilience：

- WebSocket reconnect
- exponential backoff
- heartbeat / ping-pong
- fallback polling
- stale UI 狀態提示
- React Profiler 檢查 re-render bottleneck

---

## 核心設計思路

可以記成四件事：

- 控制更新頻率
- 控制狀態範圍
- 控制渲染成本
- 控制錯誤恢復

這題不管怎麼延伸，基本上都繞不開這四個點。

---

## 1. 控制更新頻率

不要每一筆 socket message 都直接 `setState`。

原因：

- message 太多時，React render 會被打爆
- browser main thread 容易卡
- UI 不一定需要毫秒級更新

比較合理的做法是：

- 先把 message 放進 memory buffer
- 每隔固定時間 flush 一次
- 一次更新一批資料

例如：

```ts
socket.onmessage = (event) => {
  const update = JSON.parse(event.data)
  bufferRef.current[update.symbol] = update
}

flushTimerRef.current = window.setInterval(() => {
  const updates = Object.values(bufferRef.current)
  if (updates.length === 0) return

  bufferRef.current = {}
  applyBatchUpdates(updates)
}, 500)
```

這裡的重點是：

**資料進來的頻率，不等於 UI 更新的頻率。**

---

## 2. 控制狀態範圍

不要把所有即時資料都丟進單一 global React state，然後讓整個 component tree 一起重跑。

比較好的做法是拆成三層：

### Server state

適合：

- REST API
- polling data
- cacheable query
- retry / stale / refetch 管理

工具：

- TanStack Query / React Query

### Live state

適合：

- WebSocket streaming data
- 不斷變動的 in-memory data
- 不需要每次都重新 query server

工具：

- Zustand
- Redux Toolkit
- Jotai
- external store

### UI state

適合：

- local filter
- sort
- selected row
- drawer / modal 開關

工具：

- `useState`
- `useReducer`

這樣拆的好處是：

- server state 不會被即時資料污染
- live data 可以獨立優化
- UI state 不需要跟全域資料綁死

---

## 3. 控制渲染成本

這是 React 面試最容易被追問的地方。

你要避免的是：

- 一個 symbol 更新，整張 table 全部重 render
- 每筆資料改動都讓父層重新 render 全樹
- 幾千筆 row 一起 mount 在 DOM 上

### 做法一：每個 row 只訂閱自己需要的資料

```tsx
export const PriceRow = React.memo(({ symbol }: PriceRowProps) => {
  const price = usePriceStore((state) => state.pricesBySymbol[symbol])

  return (
    <div>
      <span>{symbol}</span>
      <span>{price?.price ?? '-'}</span>
    </div>
  )
})
```

這樣 BTC 更新時，應該只影響 BTC 那一列，不應該讓 ETH、SOL、SUI 全部一起更新。

### 做法二：large list 用 virtualization

如果有幾千筆資料，不要真的 render 幾千個 row。

```tsx
const rowVirtualizer = useVirtualizer({
  count: symbols.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 40,
})
```

virtualization 的目的不是減少資料量，而是：

**減少同時存在於 DOM 裡的元素數量。**

### 做法三：只在必要時 memoize

可以提：

- `React.memo`
- `useMemo`
- `useCallback`

但要注意講法不要太粗糙。比較好的說法是：

**memoization 不是萬能解法，我只會用在昂貴計算或明顯的 re-render bottleneck 上。**

---

## 4. 控制錯誤恢復

即時系統一定會被問穩定性。

WebSocket 不可能永遠穩，所以要有 recovery plan。

### 常見做法

- reconnect
- exponential backoff
- heartbeat / ping-pong
- stale connection detection
- fallback polling
- loading / reconnecting / stale UI 狀態

例如 reconnect：

```ts
socket.onclose = () => {
  const delay = Math.min(1000 * 2 ** retryCount, 30000)
  retryCount += 1

  reconnectTimer = window.setTimeout(() => {
    connect()
  }, delay)
}
```

如果要再講完整一點，你可以說：

如果 socket 長時間斷線，我不會讓 UI 靜默失效。我會顯示 stale / reconnecting state，必要時 fallback 到 polling，確保使用者至少拿得到比較新的資料。

---

## 實際架構

```text
src/
  features/
    market/
      MarketDashboard.tsx
      PriceTable.tsx
      priceStore.ts
      usePriceSocket.ts
      useVisiblePrices.ts
  lib/
    socket.ts
    throttle.ts
```

核心資料流：

```text
WebSocket
  -> receive raw updates
  -> buffer in memory
  -> flush every 500ms
  -> update external store
  -> only affected rows re-render
  -> virtualized list renders visible rows only
```

---

## WebSocket hook 範例

```ts
import { useEffect, useRef } from 'react'
import { usePriceStore } from './priceStore'

type PriceUpdate = {
  symbol: string
  price: number
  timestamp: number
}

export function usePriceSocket() {
  const applyBatchUpdates = usePriceStore((state) => state.applyBatchUpdates)
  const bufferRef = useRef<Record<string, PriceUpdate>>({})
  const flushTimerRef = useRef<number | null>(null)

  useEffect(() => {
    let reconnectTimer: number | null = null
    let socket: WebSocket | null = null
    let retryCount = 0

    const connect = () => {
      socket = new WebSocket('wss://example.com/prices')

      socket.onmessage = (event) => {
        const update: PriceUpdate = JSON.parse(event.data)
        bufferRef.current[update.symbol] = update
      }

      socket.onopen = () => {
        retryCount = 0
      }

      socket.onerror = () => {
        socket?.close()
      }

      socket.onclose = () => {
        const delay = Math.min(1000 * 2 ** retryCount, 30000)
        retryCount += 1

        reconnectTimer = window.setTimeout(() => {
          connect()
        }, delay)
      }
    }

    connect()

    flushTimerRef.current = window.setInterval(() => {
      const updates = Object.values(bufferRef.current)
      if (updates.length === 0) return

      bufferRef.current = {}
      applyBatchUpdates(updates)
    }, 500)

    return () => {
      socket?.close()

      if (reconnectTimer) clearTimeout(reconnectTimer)
      if (flushTimerRef.current) clearInterval(flushTimerRef.current)
    }
  }, [applyBatchUpdates])
}
```

---

## Zustand store 範例

```ts
import { create } from 'zustand'

type Price = {
  symbol: string
  price: number
  timestamp: number
}

type PriceStore = {
  pricesBySymbol: Record<string, Price>
  applyBatchUpdates: (updates: Price[]) => void
}

export const usePriceStore = create<PriceStore>((set) => ({
  pricesBySymbol: {},

  applyBatchUpdates: (updates) => {
    set((state) => {
      const next = { ...state.pricesBySymbol }

      for (const update of updates) {
        next[update.symbol] = update
      }

      return {
        pricesBySymbol: next,
      }
    })
  },
}))
```

---

## Row component 範例

```tsx
import React from 'react'
import { usePriceStore } from './priceStore'

type PriceRowProps = {
  symbol: string
}

export const PriceRow = React.memo(({ symbol }: PriceRowProps) => {
  const price = usePriceStore((state) => state.pricesBySymbol[symbol])

  return (
    <div>
      <span>{symbol}</span>
      <span>{price?.price ?? '-'}</span>
    </div>
  )
})
```

---

## Virtualized table 範例

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { useRef } from 'react'
import { PriceRow } from './PriceRow'

type PriceTableProps = {
  symbols: string[]
}

export function PriceTable({ symbols }: PriceTableProps) {
  const parentRef = useRef<HTMLDivElement>(null)

  const rowVirtualizer = useVirtualizer({
    count: symbols.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 40,
  })

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div
        style={{
          height: rowVirtualizer.getTotalSize(),
          position: 'relative',
        }}
      >
        {rowVirtualizer.getVirtualItems().map((virtualRow) => {
          const symbol = symbols[virtualRow.index]

          return (
            <div
              key={symbol}
              style={{
                position: 'absolute',
                top: 0,
                left: 0,
                width: '100%',
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <PriceRow symbol={symbol} />
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

---

## 如果不是 WebSocket，而是 polling

不是所有資料都值得用 socket。

適合 polling 的通常是：

- dashboard summary
- user profile
- historical chart
- 低頻更新的報表

可以用 React Query：

```ts
import { useQuery } from '@tanstack/react-query'

export function useMarketSummary() {
  return useQuery({
    queryKey: ['market-summary'],
    queryFn: async () => {
      const res = await fetch('/api/market-summary')
      if (!res.ok) {
        throw new Error('Failed to fetch market summary')
      }
      return res.json()
    },
    refetchInterval: 5000,
    staleTime: 3000,
    retry: 3,
  })
}
```

面試可以這樣說：

**I would not use WebSocket for every type of data. For high-frequency live data, I would use WebSocket. For lower-frequency or less latency-sensitive data, I would use polling with React Query and configure refetch interval, stale time, and retry strategy appropriately.**

---

## 可以提到的工具

### Server state / polling

- TanStack Query / React Query

### Client state

- Zustand
- Redux Toolkit
- Jotai

### WebSocket

- Native WebSocket
- Socket.IO

### 大列表效能

- TanStack Virtual
- react-window

### 表格

- TanStack Table

### 效能分析

- React Profiler
- Chrome Performance tab

### 避免重算 / 重 render

- React.memo
- useMemo
- useCallback

### 圖表高頻更新

- Lightweight Charts
- ECharts
- TradingView Charting Library

### 背景處理

- Web Worker

### 資料流控制

- throttle
- debounce
- requestAnimationFrame

---

## React 官方觀點補充

React 官方對 `useMemo` 的定位是：

**cache expensive calculation between re-renders**

所以不要把 memoization 當成萬能解法，而是用在真的昂貴的計算或會造成大量 re-render 的地方。

---

## 英文面試回答版本

**For high-frequency real-time data, I would prefer WebSocket over aggressive polling. However, I would not update React state on every single message. I would first buffer incoming updates and flush them at a controlled interval, for example every 500 milliseconds or every second.**

**For state management, I would separate server state, live data state, and local UI state. Server state can be handled by React Query, while live WebSocket data can be stored in Zustand or another external store.**

**To avoid overloading the browser, I would only update changed entities, avoid unnecessary global state updates, memoize expensive calculations when needed, and use virtualization for large tables or lists.**

**I would also add reconnect logic, exponential backoff, heartbeat checks, and fallback polling, so the UI remains resilient when the WebSocket connection is unstable.**

---

## 最後一句總結

這題最值得記的回答不是：

**I would use WebSocket.**

而是：

**I would control the frequency of UI updates and limit the rendering scope.**
