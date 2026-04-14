# Skyvern — Setup Guide

## Requirements
- Python 3.11.9 (Skyvern is NOT compatible with Python 3.14)
- PostgreSQL 16
- Node.js v22.20.0
- Groq API key (high-limit account recommended)
- Windows 10/11

---

## Step 1 — Install Python 3.11
Download Python 3.11.9 from python.org and install alongside existing Python version.

```
py -3.11 --version
→ Python 3.11.9
Path: C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\
```

## Step 2 — Install PostgreSQL 16
Download from postgresql.org/download/windows. Install to `D:\PostgreSQL`.
Set password for `postgres` user during installation (used: `skyvern123`).

## Step 3 — Install Skyvern and Playwright

```cmd
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\python.exe" -m pip install skyvern
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\playwright.exe" install chromium
```

## Step 4 — Clone Repository (required for Alembic migrations)

```cmd
mkdir C:\skyvern-test
cd C:\skyvern-test
git clone https://github.com/Skyvern-AI/skyvern .
```

## Step 5 — Create the Database

```cmd
"D:\PostgreSQL\bin\psql" -U postgres -c "CREATE DATABASE skyvern;"
```

## Step 6 — Configure .env
Create `C:\skyvern-test\.env`:

```
DATABASE_STRING=postgresql+asyncpg://postgres:skyvern123@localhost:5432/skyvern
ENABLE_OPENAI_COMPATIBLE=true
OPENAI_COMPATIBLE_MODEL_NAME=meta-llama/llama-4-scout-17b-16e-instruct
OPENAI_COMPATIBLE_API_KEY=<your_groq_api_key>
OPENAI_COMPATIBLE_API_BASE=https://api.groq.com/openai/v1
LLM_KEY=OPENAI_COMPATIBLE
SECONDARY_LLM_KEY=OPENAI_COMPATIBLE
```

## Step 7 — Apply Timezone Bug Fix
Skyvern 1.0.29-1.0.30 has a bug with timezone-aware datetime objects and PostgreSQL. Fix:

```cmd
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\python.exe" -c "import re; path = r'C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Lib\site-packages\skyvern\forge\sdk\db\repositories\tasks.py'; f=open(path,'r',encoding='utf-8'); content=f.read(); f.close(); content=re.sub(r'datetime\.now\(timezone\.utc\)', 'datetime.now()', content); f=open(path,'w',encoding='utf-8'); f.write(content); f.close(); print('Done!')"
```

## Step 8 — Run Alembic Migrations
Must be run from the cloned repository folder (not the pip package):

```cmd
cd C:\skyvern-test
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\alembic.exe" upgrade head
```

Also copy .env to skyvern-test folder before running alembic:

```cmd
copy C:\Users\<user>\.env C:\skyvern-test\.env
```

## Step 9 — Start the Server

```cmd
cd C:\skyvern-test
"C:\Users\<user>\AppData\Local\Python\pythoncore-3.11-64\Scripts\skyvern.exe" run server
```

## Step 10 — Generate API Key
Run after every server restart:

```cmd
curl -X POST http://localhost:8000/api/v1/internal/auth/repair
```

Copy the `api_key` from the response. Then immediately open `.env` and add Groq config below the SKYVERN_API_KEY line (repair overwrites the file).

## Step 11 — Run a Task

```cmd
curl -X POST http://localhost:8000/api/v1/tasks ^
  -H "Content-Type: application/json" ^
  -H "x-api-key: <api_key>" ^
  -d "{\"url\":\"https://books.toscrape.com\",\"navigation_goal\":\"...\",\"extracted_information_schema\":{...},\"max_steps_per_run\":25}"
```

---

## Important Notes

- **Do NOT use the Discover UI** — it automatically uses Code mode which loops indefinitely with Groq
- **Always use direct API curl** to run tasks
- **Generate a new API key** before each run (key expires on server restart)
- **repair overwrites .env** — always re-add Groq config after repair
- **Groq free tier**: `meta-llama/llama-4-scout-17b-16e-instruct` has 500K tokens/day and 30K TPM — better than llama-3.3-70b-versatile (100K/day, 12K TPM)
- **Skyvern 2.0** (Code generation mode) was not tested — requires paid OpenAI API

## Known Issues

| Issue | Description | Solution |
|-------|-------------|----------|
| Timezone bug | `datetime.now(timezone.utc)` incompatible with PostgreSQL | Apply Step 7 fix |
| repair overwrites .env | SKYVERN_API_KEY replaces all config | Re-add Groq config after repair |
| No memory between steps | Agent re-finds same elements repeatedly | Known limitation of Skyvern 1.0 |
| extracted_information empty | Schema not populated even on completed tasks | Known limitation with Groq models |
| alembic config not found | pip package lacks alembic.ini | Must clone repo (Step 4) |
| Docker required for quickstart | `skyvern quickstart` requires Docker | Use `skyvern run server` directly |
