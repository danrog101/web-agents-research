# Skyvern — Attempt 4

## 📝 Task Description
Same task with improved step-by-step navigation goal, extracted_information_schema, and increased max_steps_per_run. This was the first run in a **new terminal** (fresh start).

## 🤖 Agent Configuration
- **Framework**: Skyvern 1.0.30
- **LLM Provider**: Groq (OpenAI-Compatible)
- **Model**: `meta-llama/llama-4-scout-17b-16e-instruct`
- **Max Steps**: 30
- **Terminal**: New terminal (fresh start — server restarted before this run)
- **Engine**: Skyvern 1.0 (via direct API curl)
- **extracted_information_schema**: Added (soumission_price, soumission_rating, most_expensive_mystery_title, most_expensive_mystery_price, most_expensive_mystery_rating)

## Navigation Goal (improved)
```
Step 1: Go to https://books.toscrape.com and find the book called Soumission.
Extract its exact price and star rating.
Step 2: Navigate to the Mystery category using the left sidebar.
Step 3: Browse all books in Mystery category and find the one with the highest price.
Extract its title, price and star rating.
Step 4: Report all extracted information.
```

## ⏱️ Timing
| Field | Value |
|-------|-------|
| **Start** | ~14:41:00 |
| **End** | ~14:44:30 |
| **Duration** | ~3 minutes 30 seconds |
| **Steps** | 11 |
| **Input Tokens** | 100,083 |
| **Result** | ❌ FAIL |

## ✅ Final Results
| Field | Value |
|-------|-------|
| **DB Status** | `failed` |
| **extracted_information** | empty |
| **Soumission price** | ⚠️ Found visually, not recorded |
| **Most expensive book** | ⚠️ Found visually (Boar Island £59.48), not recorded |

## 🔍 Differences Compared to Previous Runs
| Category | Run 1 | Run 2 | Run 3 | Run 4 |
|----------|-------|-------|-------|-------|
| **Duration** | ~4:13 | ~3:49 | ~2:17 | ~3:30 |
| **Steps** | 14 | 12 | 9 | 11 |
| **Input Tokens** | 134,689 | 115,141 | 87,701 | 100,083 |
| **DB Status** | failed | failed | completed | failed |
| **Max Steps** | default | default | 25 | 30 |
| **Terminal** | same | same | same | **new** |
| **Schema** | none | none | basic | **detailed** |

## ⚠️ Technical Notes
- Despite more detailed step-by-step goal and increased max_steps, result was still failed
- New terminal did not improve results compared to same terminal runs
- Run 3 (completed) was actually more efficient despite less detailed goal
- Suggests that Skyvern 1.0 with Groq model has inherent limitations regardless of prompt engineering
