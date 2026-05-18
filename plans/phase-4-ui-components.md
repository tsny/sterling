# Phase 4: UI Components

## Goal
Build all React components from the bottom up — shared types and utilities first, then atomic components, then the composite `StockCard` that orchestrates data fetching.

## Pre-conditions
- Phase 1 complete: `src/lib/`, `src/components/` directories exist
- Phase 3 complete: API routes available at `/api/ticker`, `/api/macro-scout`, `/api/synthesize`
- `recharts`, `lucide-react`, `@google/generative-ai` installed

---

## Files to Implement

### `src/lib/types.ts`
```ts
export interface PricePoint {
  date: string;   // "YYYY-MM-DD"
  close: number;
}

export type EventType = "earnings" | "dividend" | "macro_industry" | "macro_fred";

export interface TimelineEvent {
  date: string;   // "YYYY-MM-DD"
  title: string;
  description: string;
  type: EventType;
}

export interface TickerData {
  ticker: string;
  name: string;
  sector: string;
  industry: string;
  currentPrice: number;
  priceChange: number;
  priceChangePct: number;
  high52w: number;
  low52w: number;
  prices: PricePoint[];
  nextEarnings: string | null;
  dividendEvents: TimelineEvent[];
}

export interface MacroScoutResponse {
  events: TimelineEvent[];
  queriesUsed?: string[];
}
```

### `src/lib/utils.ts`
```ts
export function daysUntil(dateStr: string): number  // positive = future, negative = past
export function formatCurrency(n: number): string    // "$123.45"
export function formatPct(n: number): string         // "+1.23%" with sign
export function formatDate(dateStr: string): string  // "Jul 30, 2025"
export function eventTypeColor(type: EventType): { bg: string; text: string; border: string }
  // earnings  → violet
  // dividend  → emerald
  // macro_industry → amber
  // macro_fred     → sky
```

---

### `src/components/EarningsBadge.tsx`
Props: `{ nextEarnings: string | null }`

- If null: grey "No date set" pill
- Compute `days = daysUntil(nextEarnings)`
- If days < 0: show "Earnings passed" in slate
- If 0–29: rose/red urgency — large number, "DAYS TO EARNINGS" label
- If 30–59: amber warning
- If 60+: slate/neutral

Renders a large prominent badge, not just text.

---

### `src/components/PriceChart.tsx`
Props: `{ prices: PricePoint[]; ticker: string }`

- Uses Recharts `AreaChart` (responsive, height ~160px)
- Computes whether 6mo return is positive or negative → green or red gradient fill
- X-axis: show ~6 evenly spaced month labels (abbreviated "Jan", "Feb", etc.)
- Y-axis: hidden labels, just the domain
- Custom tooltip: shows date + formatted price
- `ResponsiveContainer` width="100%"

---

### `src/components/TimelineEvent.tsx`
Props: `{ event: TimelineEvent }`

Single row in the timeline:
- Left: colored vertical bar (matches event type color)
- Date chip: `formatDate(event.date)`, small, muted
- Type badge: pill with event type label (e.g. "EARNINGS", "MACRO", "DIVIDEND")
- Title: semibold
- Description: small muted text below title

---

### `src/components/MacroTimeline.tsx`
Props: `{ events: TimelineEvent[] }`

- Sorts events by date ascending
- Groups by calendar month
- Renders sticky month header ("July 2025") then `TimelineEvent` rows for that month
- Vertical connecting line via a left border on the container
- If no events: "No events found" empty state
- Max height with overflow-y-auto for scrollable timeline in card

---

### `src/components/AISummary.tsx`
Props: `{ ticker: string; name: string; events: TimelineEvent[] }`

- On mount (and when events change): POST to `/api/synthesize` with fetch
- Reads the streaming response with `response.body.getReader()`
- Progressively appends decoded text chunks to state
- Loading state: 3 animated skeleton lines (pulse)
- Error state: soft error message with retry button
- Display: gradient-bordered box (`from-violet-500 via-indigo-500 to-sky-500` border trick using a wrapper div + inner div)
- Small "AI" chip label in top-left corner of the box
- Text in `text-sm text-slate-200`
- Only fires if `events.length > 0`

---

### `src/components/StockCard.tsx`
Props: `{ ticker: string; onRemove: () => void }`

This is the main orchestrator component. Uses `"use client"`.

**State:**
```ts
const [tickerData, setTickerData] = useState<TickerData | null>(null)
const [macroEvents, setMacroEvents] = useState<TimelineEvent[]>([])
const [loading, setLoading] = useState(true)
const [error, setError] = useState<string | null>(null)
```

**Data fetching (on mount):**
1. `GET /api/ticker?ticker={ticker}` → set `tickerData`
2. With returned `industry` + `sector`, `POST /api/macro-scout` → set `macroEvents`
3. Both fetches run concurrently with `Promise.all` where possible

**Merged timeline**: combine `tickerData.dividendEvents` + earnings event (synthesize from `nextEarnings`) + `macroEvents`, sort by date.

**Layout (two columns, responsive)**:
```
┌─────────────────────────────────────────────────┐
│ [TICKER]  [Company Name]              [× Remove] │
├──────────────────────┬──────────────────────────┤
│ $price  +X.XX%       │  ┌─ AI Summary ────────┐ │
│                      │  │ (streaming text)    │ │
│ [PriceChart]         │  └─────────────────────┘ │
│                      │                          │
│ [EarningsBadge]      │  [MacroTimeline]         │
│                      │  (scrollable)            │
│ 52w H: $X  L: $X     │                          │
└──────────────────────┴──────────────────────────┘
```

Loading state: skeleton placeholders for each section.
Error state: centered error message with retry.

---

### `src/components/WatchlistInput.tsx`
Props: `{ onAdd: (ticker: string) => void }`

- Controlled input, uppercase transform on display
- Enter key or "Add" button submits
- Validation: trim, uppercase, non-empty, max 5 chars
- Clears input after successful add
- Lucide `Search` icon inside input on left
- Lucide `Plus` icon on button

---

## Visual Design Reference
| Element | Class |
|---------|-------|
| Card background | `bg-slate-900 border border-slate-800 rounded-xl` |
| Earnings badge (urgent) | `bg-rose-500/20 text-rose-300 border border-rose-500/40` |
| Earnings badge (warn) | `bg-amber-500/20 text-amber-300 border border-amber-500/40` |
| Earnings badge (normal) | `bg-violet-500/20 text-violet-300 border border-violet-500/40` |
| Timeline event: earnings | left border `border-violet-500` |
| Timeline event: dividend | left border `border-emerald-500` |
| Timeline event: macro_industry | left border `border-amber-500` |
| Timeline event: macro_fred | left border `border-sky-500` |
| AI summary border | wrapping div with gradient bg + 1px inner padding |
| Positive price | `text-emerald-400` |
| Negative price | `text-rose-400` |

---

## Files Created
| File | Action |
|------|--------|
| `src/lib/types.ts` | Created |
| `src/lib/utils.ts` | Created |
| `src/components/EarningsBadge.tsx` | Created |
| `src/components/PriceChart.tsx` | Created |
| `src/components/TimelineEvent.tsx` | Created |
| `src/components/MacroTimeline.tsx` | Created |
| `src/components/AISummary.tsx` | Created |
| `src/components/StockCard.tsx` | Created |
| `src/components/WatchlistInput.tsx` | Created |

---

## Verification
- `npx tsc --noEmit` passes with no type errors
- Visual check: render `StockCard` for "AAPL" — both columns populate, AI summary streams in
- Edge cases: ticker with no earnings date shows "No date set", timeline with no events shows empty state

## Commit
```
feat(components): full UI component library — chart, timeline, earnings badge, AI summary, stock card
```
