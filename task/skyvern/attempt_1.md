# Skyvern — Attempt 1

## 📝 Task Description
The objective was to use Skyvern to navigate `books.toscrape.com` and:
1. Find the book **'Soumission'** and note its price and star rating.
2. Navigate to the **Mystery** category.
3. Find the **most expensive book** in that category.
4. Report the title, price, and rating.

## 🤖 Agent Configuration
- **Framework**: Skyvern 1.0.30
- **LLM Provider**: Groq (OpenAI-Compatible)
- **Model**: `meta-llama/llama-4-scout-17b-16e-instruct`
- **Max Steps**: default
- **Terminal**: Same terminal as Run 2 and Run 3
- **Engine**: Skyvern 1.0 (via direct API curl)

## ⏱️ Timing
| Field | Value |
|-------|-------|
| **Start** | 14:23:08 |
| **End** | 14:27:21 |
| **Duration** | ~4 minutes 13 seconds |
| **Steps** | 14 |
| **Input Tokens** | 134,689 |
| **Result** | ❌ FAIL |

## 📋 Execution Summary
- Agent opened books.toscrape.com
- Found Soumission visually but failed to report it in final output
- Found Boar Island £59.48 visually but failed to report it in final output
- Attempted to use text input on a link element (InvalidElementForTextInput error)
- Reached max steps and exited with `Task has no webhook callback url`

## ✅ Final Results
| Field | Value |
|-------|-------|
| **DB Status** | `failed` |
| **extracted_information** | empty |
| **Soumission price** | ⚠️ Found visually, not recorded |
| **Most expensive book** | ⚠️ Found visually (Boar Island £59.48), not recorded |

## ⚠️ Technical Notes
- **Root cause**: Skyvern 1.0 has no persistent memory between steps — each step receives only the current screenshot + original goal, causing the agent to "forget" previously found data
- **Loop behavior**: Agent found Soumission multiple times but re-searched each step
- **Error**: `InvalidElementForTextInput` — agent tried to type into a link element instead of an input field
- **No extracted_information_schema** was provided in this run
