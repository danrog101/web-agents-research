# open-interpreter — Consistency Test Analysis

## Framework Info

| Field | Value |
|-------|-------|
| Framework | open-interpreter v0.4.3 |
| LLM | Groq / llama-3.3-70b-versatile |
| Test site | books.toscrape.com |
| Total runs | 3 |

## Task

> Go to https://books.toscrape.com. Find the book called 'Soumission' and note its price and star rating. Then navigate to the Mystery category and find the most expensive book. Report the title, price, and star rating of the most expensive Mystery book.

## Expected Results

| Book | Price | Rating |
|------|-------|--------|
| Soumission | £50.10 | ⭐⭐⭐ (Three) |
| Boar Island (Mystery) | £59.48 | ⭐⭐⭐⭐ (Four) |

## Results Summary

| Run | Status | Duration | Soumission | Mystery Book | Correct? |
|-----|--------|----------|-----------|--------------|---------|
| 1 | ✅ SUCCESS | ~23s | £50.10, ⭐⭐⭐ | Boar Island £59.48 | ✅ |
| 2 | ✅ SUCCESS | ~28s | £50.10, ⭐⭐⭐ | Boar Island £59.48 | ✅ |
| 3 | ✅ SUCCESS | ~32s | £50.10, ⭐⭐⭐ | Boar Island £59.48 | ✅ |

**Success rate: 3/3 (100%)**

## Run Details

### Run 1 — SUCCESS
- Duration: ~23 seconds
- Soumission: £50.10, Three stars
- Mystery most expensive: Boar Island, £59.48
- Notes: Completed extremely fast using Python requests/BeautifulSoup to scrape HTML directly

### Run 2 — SUCCESS
- Duration: ~28 seconds
- Soumission: £50.10, Three stars
- Mystery most expensive: Boar Island, £59.48
- Notes: Consistent with Run 1

### Run 3 — SUCCESS
- Duration: ~32 seconds
- Soumission: £50.10, Three stars
- Mystery most expensive: Boar Island, £59.48
- Notes: Consistent with Runs 1 and 2

## Consistency Analysis

open-interpreter achieved **perfect consistency** across all three runs with 100% success rate. All runs returned identical correct results. The framework completed tasks significantly faster than browser-use (~25s vs ~9min) by writing and executing Python code to scrape the website rather than visually navigating through a browser.

**Key observations:**
- All three runs produced identical results
- Speed advantage is dramatic (~25s vs ~9min for browser-use)
- Uses code execution rather than visual browser navigation
- Does not open a visible browser window
- No rate limit issues on free Groq tier

## Strengths
- 100% success rate across all runs
- Extremely fast (~25 seconds per run)
- Very low token consumption
- No database or Docker required
- Simple setup

## Weaknesses
- Does not simulate real user browser interaction (scrapes HTML directly)
- May fail on JavaScript-heavy sites that require actual browser rendering
- Less suitable for tasks requiring visual interaction (clicking buttons, filling forms)
- Code execution approach differs fundamentally from visual web agent approach
