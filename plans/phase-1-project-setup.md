# Phase 1: Project Setup

## Goal
Install all dependencies, scaffold the Python microservice directory, configure environment files, and establish the baseline project structure ready for feature development.

## Pre-conditions
- Next.js 16 scaffold already exists (package.json, src/app/layout.tsx, src/app/page.tsx, tsconfig.json, tailwind v4)
- Branch: `claude/financial-dashboard-ai-ettwJ`

---

## Steps

### 1.1 — Install npm dependencies
```bash
cd /home/user/sterling
npm install recharts lucide-react
npm install --save-dev @types/recharts
```
Expected result: `node_modules/recharts` and `node_modules/lucide-react` present.

### 1.2 — Scaffold Python microservice directory
Create the following empty files (content filled in Phase 2):
```
python/
  main.py
  ticker_service.py
  macro_service.py
  gemini_service.py
  requirements.txt
  .env               # gitignored
```

**`python/requirements.txt`** (create now):
```
fastapi
uvicorn[standard]
yfinance
fredapi
duckduckgo-search
google-generativeai
python-dotenv
```

### 1.3 — Create `src/lib/` directory with placeholder files
```
src/lib/types.ts    # TypeScript interfaces (filled in Phase 4)
src/lib/utils.ts    # Helper functions (filled in Phase 4)
```

### 1.4 — Create `src/components/` directory (empty, filled in Phase 4)

### 1.5 — Create environment files

**`.env.local`** (gitignored):
```
GEMINI_API_KEY=your_gemini_api_key_here
FRED_API_KEY=your_fred_api_key_here
PYTHON_API_URL=http://localhost:8000
```

**`.env.example`** (committed):
```
GEMINI_API_KEY=
FRED_API_KEY=
PYTHON_API_URL=http://localhost:8000
```

### 1.6 — Update `.gitignore`
Ensure these are ignored:
```
.env.local
python/.env
python/__pycache__/
python/.venv/
```

### 1.7 — Update `next.config.ts`
Add the Python API URL to Next.js config so it's available at build time:
```ts
// next.config.ts
import type { NextConfig } from "next";
const nextConfig: NextConfig = {};
export default nextConfig;
```
(No changes needed — keep default for now.)

---

## Files Created / Modified
| File | Action |
|------|--------|
| `package.json` | Modified (recharts + lucide-react added) |
| `python/requirements.txt` | Created |
| `python/main.py` | Created (stub) |
| `python/ticker_service.py` | Created (stub) |
| `python/macro_service.py` | Created (stub) |
| `python/gemini_service.py` | Created (stub) |
| `src/lib/types.ts` | Created (stub) |
| `src/lib/utils.ts` | Created (stub) |
| `.env.example` | Created |
| `.env.local` | Created (gitignored) |
| `.gitignore` | Modified |

---

## Verification
```bash
node -e "require('recharts')"           # Should not throw
node -e "require('lucide-react')"       # Should not throw
ls python/requirements.txt              # Exists
cat .env.example                        # Shows template vars
```

## Commit
```
feat: scaffold project — install deps, python microservice structure, env files
```
