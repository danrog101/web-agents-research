# Skyvern — Run 1

## Result

**FAIL** — Groq TPM rate limit (12,000 tokens/min insufficient for Skyvern)

| Field | Value |
|-------|-------|
| Status | ❌ FAIL (rate limit) |
| Duration | ~8 minutes |
| Soumission | Found (£50.10, ⭐⭐⭐) |
| Mystery book | Found (navigated to category) |
| Outcome | Task looped indefinitely due to rate limit |

---

## What Happened

Skyvern successfully opened the browser, navigated to books.toscrape.com, found the book Soumission with its price and star rating, and navigated to the Mystery category. However, the Groq free tier has a limit of 12,000 tokens/minute. Skyvern sends page screenshots to the LLM at every step, consuming 7,000–12,000 tokens per request. When the limit was exceeded, Skyvern restarted and began again from scratch instead of reporting an error, resulting in an infinite loop.

---

## Token Consumption

Skyvern consumes **significantly more tokens** than browser-use and open-interpreter because it sends visual page screenshots to the LLM at every step instead of HTML text.

| Framework | Tokens per run (approx.) |
|-----------|--------------------------|
| browser-use | ~3,000–5,000 |
| open-interpreter | ~1,000–2,000 |
| **Skyvern** | **~15,000–30,000+** |

---

## Setup — All Commands

### 1. Prerequisites

```
Python 3.11.9 (separate installation — Skyvern is not compatible with Python 3.14)
PostgreSQL 16
Node.js v22.20.0
```

### 2. Installing Python 3.11

Skyvern requires Python 3.11. Downloaded from python.org and installed alongside Python 3.14.

```
Path: C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\
```

### 3. Installing PostgreSQL 16

Downloaded from postgresql.org/download/windows, installed to `D:\PostgreSQL`.

Creating the database:
```cmd
"D:\PostgreSQL\bin\psql" -U postgres -c "CREATE DATABASE skyvern;"
```

Running Alembic migrations:
```cmd
cd C:\skyvern-test
C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\Scripts\alembic.exe upgrade head
```

### 4. Installing Skyvern

```cmd
py -3.11 -m pip install skyvern
py -3.11 -m playwright install chromium
```

### 5. Configuring .env

Created file `C:\skyvern-test\.env`:

```
DATABASE_STRING=postgresql+asyncpg://postgres:skyvern123@localhost:5432/skyvern
ENABLE_OPENAI_COMPATIBLE=true
OPENAI_COMPATIBLE_MODEL_NAME=llama-3.3-70b-versatile
OPENAI_COMPATIBLE_API_KEY=<groq_api_key>
OPENAI_COMPATIBLE_API_BASE=https://api.groq.com/openai/v1
LLM_KEY=OPENAI_COMPATIBLE
SECONDARY_LLM_KEY=OPENAI_COMPATIBLE
```

### 6. Timezone Bug Fix

Skyvern has a bug with timezone-aware vs timezone-naive datetime objects in PostgreSQL. Fixed in:

```
C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\Lib\site-packages\skyvern\forge\sdk\db\repositories\tasks.py
```

All instances of `datetime.now(timezone.utc)` replaced with `datetime.now()`:

```cmd
"C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\python.exe" -c "import re; path = r'C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\Lib\site-packages\skyvern\forge\sdk\db\repositories\tasks.py'; f=open(path,'r',encoding='utf-8'); content=f.read(); f.close(); content=re.sub(r'datetime\.now\(timezone\.utc\)', 'datetime.now()', content); f=open(path,'w',encoding='utf-8'); f.write(content); f.close(); print('Done!')"
```

### 7. Starting the Server

**Terminal 1 — API server:**
```cmd
C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\Scripts\skyvern.exe run server
```

**Terminal 2 — UI (optional):**
```cmd
C:\Users\38591\AppData\Local\Python\pythoncore-3.11-64\Scripts\skyvern.exe run ui
```

### 8. Generating the API Key

```cmd
curl -X POST http://localhost:8000/api/v1/internal/auth/repair
```

### 9. Running the Task

```cmd
curl -X POST http://localhost:8000/api/v1/tasks ^
  -H "Content-Type: application/json" ^
  -H "x-api-key: <api_key>" ^
  -d "{\"url\":\"https://books.toscrape.com\",\"navigation_goal\":\"Find the book Soumission and note price and star rating. Then go to Mystery category and find most expensive book title price and rating.\"}"
```

### 10. Cancelling Stuck Runs

```cmd
"D:\PostgreSQL\bin\psql" -U postgres -d skyvern -c "UPDATE workflow_runs SET status='canceled' WHERE status='created';"
```

---

## Notes

- Skyvern UI (Discover) automatically selects "Code" mode which loops indefinitely with the Groq model — use the direct API curl instead
- The API key expires after each server restart — generate a new one before each run
- Skyvern consumes ~15,000–30,000+ tokens per run due to screenshot-based vision
- Groq free tier (12,000 TPM / 100,000 TPD) is at the limit of capacity for Skyvern
