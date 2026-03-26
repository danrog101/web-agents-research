# Browser-Use Agent — Attempt 3

## 📝 Task Description
The objective was to use an autonomous browser agent to navigate `books.toscrape.com` and:
1. Find the book **'Soumission'** and note its price and star rating.
2. Navigate to the **Mystery** category.
3. Find the **most expensive book** in that category.
4. Report the title, price, and rating.

## 🤖 Agent Configuration
- **Framework**: `browser-use` (v0.12.5)
- **LLM Provider**: Groq
- **Model**: `meta-llama/llama-4-scout-17b-16e-instruct`
- **Max Steps**: 20
- **Terminal**: Same terminal as Attempt 1 and 2 (no restart)
- **Initial URL**: `https://books.toscrape.com`

## ⏱️ Timing
| Field | Value |
|-------|-------|
| **Start** | 18:47:38 |
| **End** | 18:51:00 |
| **Duration** | **202.2 seconds (~3.4 min)** |
| **Steps** | 15 / 20 |
| **Result** | ❌ FAIL — Rate limit exceeded |

## 📋 Detailed Execution Log

| Step | Action | Tool Result | Agent Evaluation |
|------|--------|-------------|-----------------|
| **1** | `navigate` | 🔗 Navigated to `https://books.toscrape.com` | ✅ Page loaded. Found 'Soumission' £50.10, 3-star rating. |
| **2** | `click` (Soumission) | 🖱️ Clicked "Soumission" | ✅ Confirmed price £50.10, rating 3 stars. |
| **3** | `click` (Fiction) | 🖱️ Clicked "Fiction" | ⚠️ Wrong click — Fiction instead of Mystery (same as Attempt 1). |
| **4** | `click` (Mystery) | 🖱️ Clicked "Mystery" | ✅ Navigated to Mystery category. |
| **5** | `scroll` | 🔍 Scrolled down 736px | ✅ Found 'The Past Never Ends' £56.50 (most expensive so far). |
| **6** | `scroll` | 🔍 Scrolled down 736px | ✅ Found 'Boar Island' £59.48 (new most expensive). |
| **7** | `scroll` | 🔍 Scrolled down 736px | ✅ Confirmed Boar Island £59.48 leader. |
| **8** | `click` (next) | 🖱️ Clicked "next" | 🔄 Page 2. Found 'The No. 1 Ladies' Detective Agency' £57.70. |
| **9** | `click` (Books to Scrape) | 🖱️ Clicked logo | ⚠️ Accidental navigation to homepage. |
| **10** | `navigate` | 🔗 Tried wrong URL `/category/mystery/` | ⚠️ Wrong URL — page timeout. |
| **11–15** | `rate limit errors x6` | ❌ Token limit reached | ❌ `Rate limit reached: Used 491,934 / 500,000 tokens per day` |

## ✅ Partial Results (before rate limit)

| Target Book | Price | Star Rating | Location |
|-------------|-------|-------------|----------|
| **Soumission** | £50.10 | ⭐⭐⭐ (3/5) | Main Page |
| **Boar Island (Anna Pigeon #19)** | **£59.48** | *(Most Expensive — found before failure)* | Mystery Category |

### Agent's Final Statement
> Task not completed — agent stopped due to rate limit error at step 10.
> Last known result: 'Boar Island (Anna Pigeon #19)' £59.48 as the most expensive book in Mystery.

## 🔍 Differences Compared to Attempt 1 and 2

| Category | Attempt 1 | Attempt 2 | Attempt 3 |
|----------|-----------|-----------|-----------|
| **Duration** | ~2 min | 576.8s | **202.2s** |
| **Steps** | 19 / 20 | 17 / 20 | **15 / 20** |
| **Fiction mistake** | ✅ Yes | ❌ No | ✅ Yes |
| **Soumission rating** | ⭐⭐⭐ reported | ⚠️ Not reported | ⭐⭐⭐ reported |
| **Final answer** | ✅ Complete | ✅ Partial | ❌ Not completed |
| **Result** | ✅ SUCCESS | ✅ SUCCESS | ❌ FAIL |
| **Failure cause** | — | — | Rate limit (500k TPD) |
| **Terminal** | New terminal | Same terminal | Same terminal |

## ⚠️ Technical Notes
- **Success Flag**: `False`
- **Step Usage**: 15/20 (75%)
- **Failure cause**: `rate_limit_exceeded` — Groq free tier daily limit: 500,000 tokens/day. Attempt 2 consumed most of the quota (576s of heavy inference), leaving insufficient tokens for Attempt 3.
- **Key observation**: Attempt 3 was faster (202s) because the agent quickly hit errors and stopped.
- **Recommendation**: For future consistency tests, use a new day or a separate API key for each attempt to avoid token exhaustion.
