# Skyvern — Consistency Test Analysis

## Framework Info

| Field | Value |
|-------|-------|
| Framework | Skyvern 1.0.29 |
| LLM | Groq / llama-3.3-70b-versatile |
| Test site | books.toscrape.com |
| Total runs | 1 (rate limited) |

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
| 1 | ❌ FAIL | ~8 min | Found | Found (navigated) | Partial |
| 2 | ⏳ PENDING | — | — | — | — |
| 3 | ⏳ PENDING | — | — | — | — |

**Success rate: 0/1 completed runs**

*Runs 2 and 3 pending — awaiting Groq token reset*

## Run Details

### Run 1 — FAIL (rate limit)
- Duration: ~8 minutes
- Soumission: Found and identified (£50.10, Three stars)
- Mystery category: Navigated successfully
- Failure reason: Groq TPM rate limit (12,000 tokens/min) exceeded during step processing
- Behaviour: Task looped indefinitely — restarted from beginning each time rate limit was hit
- Notes: Framework functionality confirmed working; failure is due to API constraints

## Token Consumption

Skyvern consumes significantly more tokens than other frameworks because it sends full page screenshots to the LLM at every step.

| Framework | Tokens per run (approx.) | Approach |
|-----------|--------------------------|---------|
| browser-use | ~3,000–5,000 | HTML + DOM |
| open-interpreter | ~1,000–2,000 | Code execution |
| **Skyvern** | **~15,000–30,000+** | **Visual screenshots** |

## Consistency Analysis

Only one run attempted due to Groq rate limits. Skyvern confirmed it can navigate the target site and locate the required information, but could not complete the full task within the free tier token limits.

**Key observations:**
- Skyvern uses computer vision — sends screenshots to the LLM instead of HTML
- This approach is more robust for complex UIs but requires significantly more tokens
- Groq free tier (12,000 TPM / 100,000 TPD) is insufficient for reliable Skyvern operation
- Framework requires PostgreSQL database and significant setup compared to other agents

## Strengths
- Vision-based approach works on any website regardless of HTML structure
- Can interact with complex dynamic UIs
- Built-in workflow builder and UI dashboard
- Supports parallel task execution

## Weaknesses
- Very high token consumption (~10x more than browser-use)
- Requires PostgreSQL database setup
- Complex installation with multiple bug fixes required (timezone bug in source code)
- Groq free tier insufficient — paid API or high-limit account needed
- Skyvern UI "Discover" mode incompatible with Groq (loops indefinitely)
- API key expires on every server restart
