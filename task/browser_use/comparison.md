# Browser-Use — Consistency Test Analysis

## 📝 Task
Navigate `books.toscrape.com`, find 'Soumission' (price + rating), go to Mystery category, find most expensive book, report results.

## 🤖 Configuration
- **Framework**: `browser-use` (v0.12.5)
- **LLM**: Groq / `meta-llama/llama-4-scout-17b-16e-instruct`
- **Approach**: DOM-based browser automation — opens real Chrome browser

---

## ⏱️ Results Summary

| | Attempt 1 | Attempt 2 | Attempt 3 |
|--|-----------|-----------|-----------|
| **Terminal** | New terminal | Same terminal | Same terminal |
| **Start** | Not measured | 18:36:36 | 18:47:38 |
| **End** | Not measured | 18:46:13 | 18:51:00 |
| **Duration** | ~9 min (estimated) | 576.8s (~9.6 min) | 202.2s (~3.4 min) |
| **Steps** | 19 / 20 | 17 / 20 | 15 / 20 |
| **Errors** | None | Element not available (step 17) | Rate limit exceeded (step 10+) |
| **Result** | ✅ SUCCESS | ✅ SUCCESS | ❌ FAIL |

**Average duration**: ~9 min (Attempts 1 & 2 only — Attempt 3 failed early)  
**Success rate**: 2/3 (67%)

---

## ✅ Output Consistency

| Field | Attempt 1 | Attempt 2 | Attempt 3 | Consistent? |
|-------|-----------|-----------|-----------|-------------|
| **Soumission price** | £50.10 | £50.10 | £50.10 (partial) | ✅ Yes |
| **Soumission rating** | ⭐⭐⭐ (3/5) | ⚠️ Not reported | ⭐⭐⭐ (3/5) | ❌ Inconsistent |
| **Most expensive book** | Boar Island £59.48 | Boar Island £59.48 | Not completed | ⚠️ Partial |
| **Boar Island rating** | Not reported | Not reported | Not completed | ❌ Not reported |

**Output was partially consistent** — correct book and price every time, but Soumission rating missing in Attempt 2 and task not completed in Attempt 3.

---

## 🔍 Key Observations

### 1. Fiction category mistake — Attempts 1 and 3
In both Attempt 1 and 3, the agent accidentally clicked "Fiction" before "Mystery". In Attempt 2 it went directly to Mystery. This shows **non-deterministic navigation behaviour** — the same task produced different action sequences.

### 2. Loop detection warnings
All 3 attempts triggered multiple "Loop detection nudge" warnings (8+ times per run), indicating the agent repeatedly visited the same pages without making progress. This is a known limitation of LLM-based agents on multi-page browsing tasks.

### 3. Attempt 3 failed due to rate limit
Running Attempts 1 and 2 in the same session consumed most of Groq's free tier daily token limit (500,000 tokens/day). Attempt 3 hit the limit at step 10 and could not continue. This demonstrates a **practical infrastructure limitation** of using free API tiers for consistency testing.

### 4. Soumission rating inconsistency
Attempt 2 did not report Soumission's star rating in its final answer, even though the agent noted it internally at Step 1 ("Price: £50.10, Rating: 3 stars"). This suggests **output formatting inconsistency** — the agent found the data but chose not to include it in the final report.

### 5. Duration difference — same vs new terminal
No significant speed difference was observed between same and new terminal. The main duration difference in Attempt 3 was caused by the agent stopping early due to rate limits, not by terminal context.

---

## 📊 Comparison Across Attempts

| Metric | Attempt 1 | Attempt 2 | Attempt 3 |
|--------|-----------|-----------|-----------|
| Fiction mistake | ✅ Yes | ❌ No | ✅ Yes |
| Loop detection warnings | 8 | 8 | 4 (stopped early) |
| Soumission rating reported | ✅ Yes | ❌ No | ✅ Yes (partial) |
| Final answer complete | ✅ Yes | ⚠️ Partial | ❌ No |
| Rate limit hit | ❌ No | ❌ No | ✅ Yes |

---

## 📊 Comparison with open-interpreter

| Metric | browser-use | open-interpreter |
|--------|-------------|-----------------|
| **Approach** | DOM-based browser control | Code generation (requests) |
| **Browser opened** | ✅ Yes (Chrome) | ❌ No |
| **Avg duration** | ~9 minutes | ~26 seconds |
| **Success rate** | 2/3 (67%) | 3/3 (100%) |
| **Output consistency** | Partial | Full |
| **Loop detection** | 8+ warnings per run | Not applicable |
| **Rate limit hit** | ✅ Yes (Attempt 3) | ❌ No |
| **Self-healing** | Loop nudge only | ValueError self-correction |
| **Token usage** | High (DOM + screenshots) | Low (text only) |

---

## ✅ Conclusion

Browser-use successfully completed the task in 2 out of 3 attempts, with consistent identification of the most expensive book (Boar Island £59.48). However, it showed notable inconsistencies in output formatting (Soumission rating missing in Attempt 2), non-deterministic navigation (Fiction mistake in 2/3 runs), and failed entirely on Attempt 3 due to API rate limits.

The framework is well-suited for tasks that require **real browser interaction** — clicking, scrolling, navigating JavaScript-heavy pages, handling authentication. For simple data extraction from static HTML, faster and cheaper alternatives exist.

**Verdict**: ⚠️ Moderately consistent. Reliable for browser automation tasks but sensitive to token limits and non-deterministic in navigation.

**Recommended for**: Tasks requiring real browser interaction, login flows, JavaScript-rendered pages.  
**Not recommended for**: Simple data extraction where code-based scraping suffices.
