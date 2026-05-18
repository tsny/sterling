# Phase 3: Next.js API Routes

## Goal
Build the three Next.js Route Handlers that sit between the React frontend and the external services (Python microservice + Gemini API). These keep API keys server-side and provide a clean internal API for client components.

## Pre-conditions
- Phase 1 complete: project structure exists
- `GEMINI_API_KEY` and `PYTHON_API_URL` set in `.env.local`
- Python microservice from Phase 2 is available at `PYTHON_API_URL`

---

## Files to Create

### `src/app/api/ticker/route.ts`
Simple GET proxy to the Python microservice.

```ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  const ticker = req.nextUrl.searchParams.get("ticker");
  if (!ticker) return NextResponse.json({ error: "ticker required" }, { status: 400 });

  const res = await fetch(
    `${process.env.PYTHON_API_URL}/ticker-data?ticker=${ticker}`,
    { next: { revalidate: 300 } }  // cache 5 min
  );
  const data = await res.json();
  if (!res.ok) return NextResponse.json(data, { status: res.status });
  return NextResponse.json(data);
}
```

### `src/app/api/macro-scout/route.ts`
POST proxy to the Python microservice.

```ts
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const body = await req.json();
  const { ticker, industry, sector } = body;
  if (!ticker || !industry || !sector) {
    return NextResponse.json({ error: "ticker, industry, sector required" }, { status: 400 });
  }

  const res = await fetch(`${process.env.PYTHON_API_URL}/macro-scout`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ ticker, industry, sector }),
    next: { revalidate: 1800 }  // cache 30 min — macro events don't change often
  });
  const data = await res.json();
  return NextResponse.json(data, { status: res.ok ? 200 : res.status });
}
```

### `src/app/api/synthesize/route.ts`
Calls Gemini directly from Next.js (keeps `GEMINI_API_KEY` server-side). Returns a **streaming text response** so the UI can progressively render the synthesis.

Caching strategy: synthesis output is stable for as long as the underlying events are (30 min). Use a module-level Map cache keyed by ticker with a 30-min TTL. On a cache hit, return the stored text immediately as a plain response. On a cache miss, stream from Gemini while collecting chunks, then store the result before closing the stream.

```ts
import { NextRequest } from "next/server";
import { GoogleGenerativeAI } from "@google/generative-ai";

const synthesisCache = new Map<string, { text: string; expires: number }>();
const SYNTHESIS_TTL_MS = 30 * 60 * 1000;

export async function POST(req: NextRequest) {
  const { ticker, name, events } = await req.json();

  const cached = synthesisCache.get(ticker);
  if (cached && Date.now() < cached.expires) {
    return new Response(cached.text, {
      headers: { "Content-Type": "text/plain; charset=utf-8" },
    });
  }

  const eventsText = events
    .map((e: { date: string; title: string; description: string }) =>
      `${e.date}: ${e.title} — ${e.description}`
    )
    .join("\n");

  const prompt = `You are a financial risk analyst. Given this chronological timeline of upcoming earnings dates and macro/industry events for ${name} (${ticker}):

${eventsText}

Identify the single highest-risk week where macro events most directly collide with or precede the earnings window. Respond in exactly 2 sentences: the first names the specific dates and events involved, the second explains the potential market impact.`;

  const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY!);
  const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

  const result = await model.generateContentStream(prompt);

  const chunks: string[] = [];
  const stream = new ReadableStream({
    async start(controller) {
      for await (const chunk of result.stream) {
        const text = chunk.text();
        if (text) {
          chunks.push(text);
          controller.enqueue(new TextEncoder().encode(text));
        }
      }
      synthesisCache.set(ticker, { text: chunks.join(""), expires: Date.now() + SYNTHESIS_TTL_MS });
      controller.close();
    },
  });

  return new Response(stream, {
    headers: { "Content-Type": "text/plain; charset=utf-8" },
  });
}
```

**npm package needed**: `npm install @google/generative-ai`

---

## Environment Variables Used
| Variable | Used by | Purpose |
|----------|---------|---------|
| `PYTHON_API_URL` | ticker route, macro-scout route | Base URL for Python FastAPI |
| `GEMINI_API_KEY` | synthesize route | Gemini API authentication |

All three are server-side only (no `NEXT_PUBLIC_` prefix) — never exposed to the browser.

---

## Files Created
| File | Action |
|------|--------|
| `src/app/api/ticker/route.ts` | Created |
| `src/app/api/macro-scout/route.ts` | Created |
| `src/app/api/synthesize/route.ts` | Created |
| `package.json` | Modified (`@google/generative-ai` added) |

---

## Verification
```bash
# Start Next.js (with Python service already running)
npm run dev

# Test ticker proxy
curl "http://localhost:3000/api/ticker?ticker=MSFT" | python3 -m json.tool | head -20

# Test macro-scout proxy
curl -X POST "http://localhost:3000/api/macro-scout" \
  -H "Content-Type: application/json" \
  -d '{"ticker":"MSFT","industry":"Software—Infrastructure","sector":"Technology"}' \
  | python3 -m json.tool | head -20

# Test synthesis streaming (output should stream 2 sentences)
curl -X POST "http://localhost:3000/api/synthesize" \
  -H "Content-Type: application/json" \
  -d '{"ticker":"MSFT","name":"Microsoft","events":[{"date":"2025-07-30","title":"Earnings","description":"Q4 FY2025 earnings release"},{"date":"2025-07-15","title":"CPI Release","description":"June inflation data due"}]}'
```

## Commit
```
feat(api): Next.js route handlers — ticker proxy, macro-scout proxy, Gemini synthesis stream with caching
```
