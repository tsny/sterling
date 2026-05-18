# Phase 2: Python Microservice

## Goal
Build the FastAPI microservice that supplies all financial data to the Next.js frontend. Two endpoints: one for per-ticker data via yfinance, one for AI-discovered macro events via Gemini + DuckDuckGo + FRED.

## Pre-conditions
- Phase 1 complete: `python/` directory exists with stub files and `requirements.txt`
- `pip install -r python/requirements.txt` has been run

---

## Files to Implement

### `python/gemini_service.py`
Two functions used by the other services:

**`generate_search_queries(industry: str, sector: str) -> list[str]`**
- Calls Gemini with prompt:
  > "You are a financial research assistant. Generate exactly 3 specific Google search queries to find upcoming macro, regulatory, or industry-specific events relevant to the {sector} sector, particularly the {industry} industry, over the next 90 days. Return only a JSON array of 3 query strings, no explanation."
- Parses JSON array from Gemini response
- Returns list of 3 query strings

**`parse_events_from_text(raw_text: str, ticker: str) -> list[dict]`**
- Calls Gemini with prompt:
  > "Extract upcoming events with specific dates from this financial news text for stock {ticker}. Return a JSON array where each item has: date (YYYY-MM-DD), title (short event name), description (1 sentence), type ('macro_industry'). Only include events with a specific date in the next 180 days. If none found, return empty array."
- Parses JSON array from response
- Returns list of event dicts

**Config**: reads `GEMINI_API_KEY` from env. Uses `google.generativeai` with model `gemini-1.5-flash` (free tier).

---

### `python/ticker_service.py`
**`get_ticker_data(ticker: str) -> dict`**

Uses `yfinance.Ticker(ticker)`:
1. `t.history(period="6mo")` → convert index to `YYYY-MM-DD` strings, return list of `{date, close}` dicts (close rounded to 2 decimal places)
2. `t.calendar` → extract `Earnings Date` (first date if list), convert to `YYYY-MM-DD` string. Handle missing/None gracefully.
3. `t.dividends` → filter to next 90 days, convert to `TimelineEvent` dicts with `type: "dividend"`
4. `t.info` → extract: `longName`, `sector`, `industry`, `currentPrice`, `regularMarketPreviousClose`, `fiftyTwoWeekHigh`, `fiftyTwoWeekLow`
5. Compute `priceChange = currentPrice - regularMarketPreviousClose` and `priceChangePct`

Returns:
```python
{
  "ticker": str,
  "name": str,
  "sector": str,
  "industry": str,
  "currentPrice": float,
  "priceChange": float,
  "priceChangePct": float,
  "high52w": float,
  "low52w": float,
  "prices": [{"date": str, "close": float}],
  "nextEarnings": str | None,
  "dividendEvents": [{"date": str, "title": str, "description": str, "type": "dividend"}]
}
```

Error handling: catch all exceptions, return `{"error": str(e)}` with HTTP 400.

---

### `python/macro_service.py`
**`get_macro_events(ticker: str, industry: str, sector: str) -> dict`**

1. **Gemini queries**: call `gemini_service.generate_search_queries(industry, sector)` → get 3 search queries
2. **DuckDuckGo search**: for each query, call `DDGS().text(query, max_results=5)` → collect `body` snippets into a single text blob
3. **Event extraction**: call `gemini_service.parse_events_from_text(combined_text, ticker)` → get list of events
4. **FRED data**: using `fredapi.Fred(api_key=FRED_API_KEY)`:
   - Series `FEDFUNDS`: get last observation → format as `{"date": ..., "title": "Fed Funds Rate", "description": f"Current rate: {value}%", "type": "macro_fred"}`
   - Series `CPIAUCSL`: get last 2 observations → compute MoM change → format as `{"date": ..., "title": "CPI (Inflation)", "description": f"MoM change: {change:+.2f}%", "type": "macro_fred"}`
5. Merge all events, sort by date ascending
6. Return `{"events": [...], "queriesUsed": [...]}`

Error handling: if FRED key missing/invalid, skip FRED step silently. If DuckDuckGo fails, return empty events list with error noted.

---

### `python/main.py`
```python
from fastapi import FastAPI, HTTPException, Query
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import ticker_service, macro_service

app = FastAPI(title="Sterling Data API")

app.add_middleware(CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_methods=["*"], allow_headers=["*"])

@app.get("/ticker-data")
def ticker_data(ticker: str = Query(...)):
    result = ticker_service.get_ticker_data(ticker.upper())
    if "error" in result:
        raise HTTPException(status_code=400, detail=result["error"])
    return result

class MacroScoutRequest(BaseModel):
    ticker: str
    industry: str
    sector: str

@app.post("/macro-scout")
def macro_scout(req: MacroScoutRequest):
    return macro_service.get_macro_events(req.ticker, req.industry, req.sector)
```

Load `.env` at startup with `python-dotenv`.

---

## Verification
```bash
cd /home/user/sterling/python
pip install -r requirements.txt
uvicorn main:app --reload --port 8000 &

# Test ticker endpoint
curl "http://localhost:8000/ticker-data?ticker=AAPL" | python3 -m json.tool | head -40

# Test macro endpoint
curl -X POST "http://localhost:8000/macro-scout" \
  -H "Content-Type: application/json" \
  -d '{"ticker":"AAPL","industry":"Consumer Electronics","sector":"Technology"}' \
  | python3 -m json.tool | head -40
```

Expected: ticker endpoint returns JSON with `prices` array of 100+ items and `nextEarnings` date. Macro endpoint returns `events` array with at least 2 entries.

## Commit
```
feat(python): FastAPI microservice — ticker data, macro scout, Gemini + DuckDuckGo
```
