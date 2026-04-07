# Skyvern — Setup Guide

## Requirements

- Python 3.11.9 (Skyvern is NOT compatible with Python 3.14)
- PostgreSQL 16
- Node.js v22.20.0
- Groq API key (high-limit account recommended)
- Windows 10/11

## Step 1 — Install Python 3.11

Download Python 3.11.9 from python.org and install alongside your existing Python version.

```
Installation path: C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\
```

## Step 2 — Install PostgreSQL 16

Download from postgresql.org/download/windows. Install to `D:\PostgreSQL` (or preferred path).

Set a password for the `postgres` user during installation.

## Step 3 — Install Skyvern

```cmd
C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\python.exe -m pip install skyvern
C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\python.exe -m playwright install chromium
```

## Step 4 — Create the Database

```cmd
"D:\PostgreSQL\bin\psql" -U postgres -c "CREATE DATABASE skyvern;"
```

## Step 5 — Configure .env

Create `C:\skyvern-test\.env`:

```
DATABASE_STRING=postgresql+asyncpg://postgres:<your_password>@localhost:5432/skyvern
ENABLE_OPENAI_COMPATIBLE=true
OPENAI_COMPATIBLE_MODEL_NAME=llama-3.3-70b-versatile
OPENAI_COMPATIBLE_API_KEY=<your_groq_api_key>
OPENAI_COMPATIBLE_API_BASE=https://api.groq.com/openai/v1
LLM_KEY=OPENAI_COMPATIBLE
SECONDARY_LLM_KEY=OPENAI_COMPATIBLE
```

## Step 6 — Apply Timezone Bug Fix

Skyvern 1.0.29 has a bug with timezone-aware datetime objects and PostgreSQL. Fix it:

```cmd
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\python.exe" -c "import re; path = r'C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Lib\site-packages\skyvern\forge\sdk\db\repositories\tasks.py'; f=open(path,'r',encoding='utf-8'); content=f.read(); f.close(); content=re.sub(r'datetime\.now\(timezone\.utc\)', 'datetime.now()', content); f=open(path,'w',encoding='utf-8'); f.write(content); f.close(); print('Done!')"
```

## Step 7 — Run Alembic Migrations

```cmd
cd C:\skyvern-test
C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\alembic.exe upgrade head
```

## Step 8 — Start the Server

**Terminal 1 — API server:**
```cmd
C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\skyvern.exe run server
```

**Terminal 2 — UI (optional, http://localhost:8080):**
```cmd
C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\skyvern.exe run ui
```

## Step 9 — Generate API Key

Run this every time after restarting the server:

```cmd
curl -X POST http://localhost:8000/api/v1/internal/auth/repair
```

Copy the `api_key` value from the response.

## Step 10 — Run a Task

```cmd
curl -X POST http://localhost:8000/api/v1/tasks ^
  -H "Content-Type: application/json" ^
  -H "x-api-key: <api_key>" ^
  -d "{\"url\":\"https://books.toscrape.com\",\"navigation_goal\":\"Find the book Soumission and note price and star rating. Then go to Mystery category and find most expensive book title price and rating.\"}"
```

## Cancelling Stuck Runs

If a run gets stuck, cancel it via PostgreSQL:

```cmd
"D:\PostgreSQL\bin\psql" -U postgres -d skyvern -c "UPDATE workflow_runs SET status='canceled' WHERE status='created';"
```

## Important Notes

- **Do NOT use the Discover UI** — it automatically uses "Code" mode which loops indefinitely with Groq
- **Always use the direct API curl** to run tasks
- **Generate a new API key** before each run (key expires on server restart)
- **Groq free tier** (12,000 TPM / 100,000 TPD) may be insufficient — use a high-limit account
- Skyvern sends full page screenshots to the LLM, consuming ~15,000–30,000+ tokens per run
