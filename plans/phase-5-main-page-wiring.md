# Phase 5: Main Page, Layout & Final Wiring

## Goal
Wire everything together into the main dashboard page — watchlist state management with localStorage persistence, the full-page layout with header and responsive card grid, global styling, and final polish.

## Pre-conditions
- All previous phases complete
- All components from Phase 4 exist and are type-safe
- Both services (Next.js + Python) can start successfully

---

## Files to Implement / Update

### `src/app/globals.css`
Replace the default Next.js globals with:

```css
@import "tailwindcss";

/* Custom scrollbar — thin, dark */
::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: #334155; border-radius: 3px; }
::-webkit-scrollbar-thumb:hover { background: #475569; }

/* Timeline vertical connector line */
.timeline-container { position: relative; }
.timeline-container::before {
  content: '';
  position: absolute;
  left: 0;
  top: 0;
  bottom: 0;
  width: 2px;
  background: linear-gradient(to bottom, #7c3aed, #0ea5e9);
  opacity: 0.3;
}

/* AI Summary gradient border animation */
@keyframes gradient-shift {
  0%, 100% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
}
.ai-border {
  background: linear-gradient(135deg, #7c3aed, #4f46e5, #0ea5e9);
  background-size: 200% 200%;
  animation: gradient-shift 4s ease infinite;
  padding: 1px;
  border-radius: 0.75rem;
}

/* Skeleton pulse — ensure it works without JS */
@keyframes pulse-skeleton {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.4; }
}
.skeleton { animation: pulse-skeleton 1.5s ease-in-out infinite; background: #1e293b; border-radius: 0.375rem; }
```

### `src/app/layout.tsx`
```tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";

const geist = Geist({ subsets: ["latin"], variable: "--font-geist" });
const geistMono = Geist_Mono({ subsets: ["latin"], variable: "--font-geist-mono" });

export const metadata: Metadata = {
  title: "Sterling — Financial Dashboard",
  description: "AI-powered earnings and macro event tracker",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className="dark">
      <body className={`${geist.variable} ${geistMono.variable} bg-slate-950 text-slate-100 min-h-screen antialiased`}>
        {children}
      </body>
    </html>
  );
}
```

### `src/app/page.tsx`
Main dashboard — `"use client"`.

**Watchlist state:**
```ts
const [watchlist, setWatchlist] = useState<string[]>(() => {
  if (typeof window === "undefined") return [];
  try {
    return JSON.parse(localStorage.getItem("sterling-watchlist") || "[]");
  } catch { return []; }
});

useEffect(() => {
  localStorage.setItem("sterling-watchlist", JSON.stringify(watchlist));
}, [watchlist]);
```

**Handlers:**
```ts
const handleAdd = (ticker: string) => {
  if (!watchlist.includes(ticker)) setWatchlist(prev => [...prev, ticker]);
};
const handleRemove = (ticker: string) => {
  setWatchlist(prev => prev.filter(t => t !== ticker));
};
```

**Layout:**
```tsx
<div className="min-h-screen bg-slate-950">
  {/* Sticky Header */}
  <header className="sticky top-0 z-10 bg-slate-950/80 backdrop-blur border-b border-slate-800 px-6 py-4">
    <div className="max-w-7xl mx-auto flex items-center justify-between gap-6">
      <div className="flex items-center gap-2">
        <TrendingUp className="text-violet-400" size={24} />
        <h1 className="text-xl font-semibold tracking-tight">Sterling</h1>
        <span className="text-xs text-slate-500 ml-1">financial dashboard</span>
      </div>
      <WatchlistInput onAdd={handleAdd} />
    </div>
  </header>

  {/* Main Content */}
  <main className="max-w-7xl mx-auto px-6 py-8">
    {watchlist.length === 0 ? (
      <EmptyState />
    ) : (
      <div className="grid grid-cols-1 xl:grid-cols-2 gap-6">
        {watchlist.map(ticker => (
          <StockCard key={ticker} ticker={ticker} onRemove={() => handleRemove(ticker)} />
        ))}
      </div>
    )}
  </main>
</div>
```

**`EmptyState` (inline in page.tsx)**:
```tsx
function EmptyState() {
  return (
    <div className="flex flex-col items-center justify-center py-32 text-center">
      <TrendingUp size={48} className="text-slate-700 mb-4" />
      <h2 className="text-xl font-medium text-slate-400 mb-2">Your watchlist is empty</h2>
      <p className="text-slate-600 text-sm">Add a ticker symbol above to get started — try AAPL, MSFT, or NVDA</p>
    </div>
  );
}
```

---

## `next.config.ts` — Final Version
No changes needed from default. `PYTHON_API_URL` is read server-side in route handlers, not needed in Next.js config.

---

## `.env.example` — Confirm Final State
```env
# Gemini API (https://aistudio.google.com/app/apikey — free)
GEMINI_API_KEY=

# FRED API (https://fred.stlouisfed.org/docs/api/api_key.html — free)
FRED_API_KEY=

# Python microservice base URL (default: local dev)
PYTHON_API_URL=http://localhost:8000
```

---

## Files Created / Modified
| File | Action |
|------|--------|
| `src/app/globals.css` | Modified (replace defaults with dark theme + custom utilities) |
| `src/app/layout.tsx` | Modified (dark html class, metadata) |
| `src/app/page.tsx` | Modified (full watchlist dashboard) |
| `.env.example` | Verified / finalized |

---

## End-to-End Verification

### Startup
```bash
# Terminal 1
cd /home/user/sterling/python && uvicorn main:app --reload --port 8000

# Terminal 2
cd /home/user/sterling && npm run dev
```
Open `http://localhost:3000`

### Test Checklist
1. **Empty state**: page loads, shows "Your watchlist is empty" with icon
2. **Add ticker**: type "AAPL", press Enter → card appears, loading skeletons shown
3. **Price chart**: chart renders with 6 months of price data, green gradient if positive
4. **Earnings badge**: badge shows correct day count with appropriate color
5. **Timeline**: at least 3 events appear (earnings + macro) sorted by date
6. **AI summary**: streams in 2 sentences describing highest-risk week
7. **Persistence**: refresh page → "AAPL" still in watchlist
8. **Remove**: click × button → card disappears, removed from localStorage
9. **Multiple tickers**: add "MSFT" → 2 cards in grid layout
10. **No earnings**: add a ticker with no earnings date → badge shows "No date set"

### TypeScript
```bash
npx tsc --noEmit
```
Should pass with zero errors.

## Commit
```
feat(page): main dashboard — watchlist state, localStorage persistence, layout, dark theme
```

---

## Final Commit (after all phases)
Tag the completed implementation:
```
feat: Sterling v1 — AI-powered financial dashboard complete
```
