# browser-use — Consistency Test Analysis

## Framework Info

| Field | Value |
|-------|-------|
| Framework | browser-use v0.12.5 |
| LLM | Groq / meta-llama/llama-4-scout-17b-16e-instruct |
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
| 1 | ✅ SUCCESS | ~9 min | £50.10, ⭐⭐⭐ | Boar Island £59.48 | ✅ |
| 2 | ✅ SUCCESS | ~9.6 min | £50.10, ⭐⭐⭐ | Boar Island £59.48 | ✅ |
| 3 | ❌ FAIL | ~3.4 min | N/A | N/A | ❌ |

**Success rate: 2/3 (67%)**

## Run Details

### Run 1 — SUCCESS
- Duration: ~9 minutes, 19 steps
- Soumission: £50.10, Three stars
- Mystery most expensive: Boar Island, £59.48
- Notes: Agent navigated correctly on first attempt

### Run 2 — SUCCESS
- Duration: ~9.6 minutes, 17 steps
- Soumission: £50.10, Three stars
- Mystery most expensive: Boar Island, £59.48
- Notes: Consistent with Run 1

### Run 3 — FAIL
- Duration: ~3.4 minutes, 15 steps
- Reason: Groq rate limit exceeded mid-task
- Notes: Task interrupted before completion

## Consistency Analysis

browser-use produced **consistent and correct results** across the two successful runs. Both runs returned identical values for Soumission (£50.10, Three stars) and the most expensive Mystery book (Boar Island, £59.48). The third run failed due to Groq API rate limiting rather than a framework error.

**Key observations:**
- Navigation strategy was consistent across runs (direct search, then category browse)
- Step count varied slightly (17–19 steps) but results were identical
- Rate limiting is an external constraint, not a framework limitation
- Average duration for successful runs: ~9 minutes

## Strengths
- Correct results on all successful runs
- Clear step-by-step browser interaction logs
- Compatible with free Groq API tier
- Easy setup (pip install, no database required)

## Weaknesses
- Relatively slow (~9 min per run)
- Susceptible to LLM API rate limits
- No built-in retry logic for rate limit errors
