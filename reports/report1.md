# Web Agent Consistency Testing — Progress Report
**Date**: April 14, 2026  
**Student**: danrog101  
**Topic**: Implementation and Evaluation of an Autonomous Web Agent in a Controlled Web Application Environment  
**Repository**: https://github.com/danrog101/web-agents-research

---

## 1. Overview

This report summarizes the progress made on the consistency testing phase (Steps 9–12 from the methodology plan). The goal was to evaluate multiple web agent frameworks by running each one three times on the same standardized task and measuring output consistency, reliability, and performance.

---

## 2. Test Task (Same for All Frameworks)

All frameworks were given the identical task:

> *"Go to https://books.toscrape.com. Find the book called 'Soumission' and note its price and star rating. Then navigate to the Mystery category and find the most expensive book. Report the title, price, and star rating of the most expensive Mystery book."*

**books.toscrape.com** was chosen as the test environment because it is a static, publicly available website specifically designed for web scraping and agent testing, with deterministic content that does not change between runs.

**Expected correct answer:**
- Soumission: £50.10, One star (1/5)
- Most expensive Mystery book: *Boar Island (Anna Pigeon #19)* — £59.48

---

## 3. Frameworks Evaluated

### 3.1 Frameworks Successfully Tested

#### browser-use (v0.12.5)
- **Approach**: DOM-based browser automation — opens real Chrome browser, navigates by reading the DOM and clicking elements
- **LLM**: Groq / `meta-llama/llama-4-scout-17b-16e-instruct`
- **Runs**: 3

| | Run 1 | Run 2 | Run 3 |
|--|-------|-------|-------|
| **Terminal** | New | Same | Same |
| **Duration** | ~9 min (no timer) | 576.8s (~9.6 min) | 202.2s (~3.4 min) |
| **Steps** | 19/20 | 17/20 | 15/20 |
| **Result** | ✅ SUCCESS | ✅ SUCCESS | ❌ FAIL (rate limit) |
| **Soumission** | £50.10, ⭐⭐⭐ | £50.10, not reported | £50.10, ⭐⭐⭐ |
| **Most expensive** | Boar Island £59.48 | Boar Island £59.48 | Not completed |

**Key observations:**
- Correct answer found in 2/3 runs
- Non-deterministic navigation: clicked Fiction instead of Mystery in 2/3 runs before self-correcting
- Run 3 failed due to Groq free tier daily token limit (500k tokens/day exhausted)
- Loop detection warnings in all runs (agent revisited same pages multiple times)
- Output inconsistency: Soumission rating was not reported in Run 2

---

#### open-interpreter (v0.4.3)
- **Approach**: Code generation — LLM writes and executes Python code (requests + BeautifulSoup), does NOT open a browser
- **LLM**: Groq / `llama-3.3-70b-versatile`
- **Runs**: 3

| | Run 1 | Run 2 | Run 3 |
|--|-------|-------|-------|
| **Terminal** | Same | Same | New |
| **Duration** | 31.9s | 23.2s | 23.8s |
| **Result** | ✅ SUCCESS | ✅ SUCCESS | ✅ SUCCESS |
| **Soumission** | £50.10, One star | £50.10, One star | £50.10, One star |
| **Most expensive** | Boar Island £59.48, Three stars | Boar Island £59.48, Three stars | Boar Island £59.48, Three stars |

**Key observations:**
- Perfect consistency: 3/3 runs, identical output every time
- Significantly faster than browser-use (~25 seconds vs ~9 minutes)
- Run 3 encountered a `ValueError` (encoding issue in new terminal) but self-corrected without human intervention
- Does NOT open a browser — uses HTTP requests instead, making it unsuitable for JavaScript-heavy pages or pages requiring authentication
- Low token consumption (text only, no screenshots)

---

#### Skyvern (v1.0.30)
- **Approach**: Vision-based browser automation — sends full page screenshots to LLM at every step
- **LLM**: Groq (OpenAI-Compatible) / `meta-llama/llama-4-scout-17b-16e-instruct`
- **Runs**: 4 (extra run added to test schema and increased steps)
- **Note**: Skyvern 1.0 was tested. Skyvern 2.0 (Code generation mode) requires paid OpenAI API and was not tested.

| | Run 1 | Run 2 | Run 3 | Run 4 |
|--|-------|-------|-------|-------|
| **Terminal** | Same | Same | Same | New |
| **Duration** | ~4:13 | ~3:49 | ~2:17 | ~3:30 |
| **Steps** | 14 | 12 | 9 | 11 |
| **Input Tokens** | 134,689 | 115,141 | 87,701 | 100,083 |
| **Max Steps** | default | default | 25 | 30 |
| **Schema** | none | none | basic | detailed |
| **DB Status** | ❌ failed | ❌ failed | ✅ completed | ❌ failed |
| **Extracted data** | ❌ empty | ❌ empty | ❌ empty | ❌ empty |

**Key observations:**
- Only 1/4 runs achieved `completed` status (Run 3)
- Despite `completed` status in Run 3, `extracted_information` field in database was always empty
- Root cause: Skyvern 1.0 sends only the current screenshot + original goal to LLM at each step — no history of previous steps. This caused the agent to find Soumission 6+ times per run without "remembering" it was already found
- High token consumption: average ~109,403 input tokens per run (vs ~25s for open-interpreter)
- Numerous bugs encountered and fixed during setup: timezone incompatibility between Python datetime.now(timezone.utc) and PostgreSQL, alembic configuration missing from pip package (required git clone), repair command overwriting .env file

---

### 3.2 Frameworks Skipped (with reasons)

| Framework | Reason Skipped |
|-----------|----------------|
| **agent-e** | Windows incompatible — `uvloop` dependency does not support Windows |
| **nanobrowser** | Groq models do not support structured output format required by Planner component |
| **stagehand** | Requires paid OpenAI API — V3 internally uses OpenAI SDK regardless of configured provider |
| **webvoyager** | Requires paid OpenAI API + GPT-4 Vision (sends screenshots as images) |
| **browserable** | Requires Docker + paid remote browser provider (Hyperbrowser or Steel) |
| **magnitude** | Requires paid Anthropic API |
| **operator** | Cloud-only — cannot access localhost environment |
| **runner-h** | Requires dedicated GPU (NVIDIA 16GB+ VRAM) — not available on test machine |

---

## 4. Comparative Analysis

### 4.1 Performance Summary

| Metric | browser-use | open-interpreter | Skyvern 1.0 |
|--------|-------------|-----------------|-------------|
| **Approach** | DOM automation | Code generation | Vision (screenshots) |
| **Opens browser** | ✅ Yes | ❌ No | ✅ Yes |
| **Success rate** | 2/3 (67%) | 3/3 (100%) | 1/4 (25%) |
| **Output consistency** | ⚠️ Partial | ✅ Full | ❌ None |
| **Avg duration** | ~9 minutes | ~26 seconds | ~3:27 minutes |
| **Memory between steps** | ✅ Yes | ✅ Yes | ❌ No |
| **Works with Groq free tier** | ✅ Yes | ✅ Yes | ✅ Yes (1.0 only) |
| **Structured output** | ⚠️ Partial | ✅ Yes | ❌ No |
| **Token efficiency** | Medium | Low (best) | High (worst) |
| **Self-healing** | Loop detection | ValueError fix | None observed |

### 4.2 Key Findings

**Finding 1: Code generation significantly outperforms browser automation for static page tasks.**
open-interpreter completed the task 100% of the time in ~25 seconds. browser-use took ~9 minutes and succeeded 67% of the time. For static HTML pages, code generation is faster, cheaper, and more reliable.

**Finding 2: Memory architecture is critical for multi-step tasks.**
Skyvern 1.0's stateless architecture (screenshot per step, no history) caused it to repeatedly find the same element without progressing. browser-use and open-interpreter both maintain action history, allowing them to avoid redundant steps.

**Finding 3: Free tier API limits are a practical constraint for browser agents.**
browser-use Run 3 failed due to Groq's 500k tokens/day limit being exhausted after two previous runs. Skyvern consumed an average of 109k tokens per run (due to screenshot-based perception). For production use, token costs are a significant factor.

**Finding 4: Framework complexity varies significantly.**
- open-interpreter: 5 minutes to set up, 1 Python script to run
- browser-use: 10 minutes to set up, 1 Python script to run
- Skyvern: ~3 hours to set up (Python 3.11, PostgreSQL, alembic, timezone bug fix, .env management issues)

---

## 5. Technical Issues Encountered

### Skyvern Setup Issues
1. **Python version incompatibility**: Skyvern requires Python 3.11. Python 3.14 (installed by default) is incompatible.
2. **Timezone bug**: `datetime.now(timezone.utc)` in Skyvern source code is incompatible with PostgreSQL's TIMESTAMP WITHOUT TIME ZONE columns. Required manual source code patching.
3. **Alembic configuration missing**: The pip package does not include `alembic.ini`. Required cloning the full repository.
4. **repair command overwrites .env**: Every call to `/api/v1/internal/auth/repair` replaces the entire `.env` file with only `SKYVERN_API_KEY`, removing all Groq configuration. Required re-adding config after every restart.
5. **Code mode (Skyvern 2.0) incompatible with Groq**: The Discover UI defaults to Code generation mode which requires OpenAI structured output. Direct API calls were required to use Skyvern 1.0.

### General Issues
- **Groq token limits**: Free tier exhaustion affected Run 3 of browser-use and limited Skyvern testing
- **Windows compatibility**: Several frameworks (agent-e, uvloop) do not support Windows natively

---

## 6. Budget Constraints

The following frameworks could not be tested due to requiring paid API access:
- stagehand, webvoyager, nanobrowser — require OpenAI API (~$5 minimum deposit)
- magnitude — requires Anthropic API
- browserable — requires Steel or Hyperbrowser (paid remote browser provider)

**Request**: If possible, a small budget (~$5–10 OpenAI credits) would allow testing of stagehand, webvoyager, and nanobrowser, providing a more complete comparison.

---

## 7. Next Steps

1. **Implement the chosen agent** in the TeamConnect web application (the controlled environment)
2. **Define evaluation tasks** specific to TeamConnect (registration, login, event creation, signup)
3. **Run the agent** on TeamConnect tasks and measure: success rate, steps required, duration, error types
4. **Write thesis chapters** 5 (application description), 6 (implementation), 7 (evaluation)

**Recommended framework for implementation**: browser-use — best balance of reliability, browser interaction capability, and documentation quality. Suitable for JavaScript-heavy applications unlike open-interpreter.

---

## 8. Repository Structure

```
web-agents-research/
├── framework-analysis/       # Research reports for 11 frameworks
├── consistency-test/
│   ├── browser-use/          # run-1.md, run-2.md, run-3.md, analysis.md, setup.md
│   ├── open-interpreter/     # run-1.md, run-2.md, run-3.md, analysis.md, setup.md
│   ├── skyvern/              # run-1.md, run-2.md, run-3.md, run-4.md, analysis.md, setup.md
│   └── skipped-frameworks/   # agent-e.md, nanobrowser.md, stagehand.md, etc.
├── task/
│   └── task.md               # Standardized test task description
└── README.md
```
