
# Skyvern — Attempt 3

## 📝 Task Description
Same task as Attempts 1 and 2, but with added `extracted_information_schema` and increased `max_steps_per_run`.

## 🤖 Agent Configuration
- **Framework**: Skyvern 1.0.30
- **LLM Provider**: Groq (OpenAI-Compatible)
- **Model**: `meta-llama/llama-4-scout-17b-16e-instruct`
- **Max Steps**: 25
- **Terminal**: Same terminal as Run 1 and Run 2
- **Engine**: Skyvern 1.0 (via direct API curl)
- **extracted_information_schema**: Added (soumission_price, soumission_rating, most_expensive_title, most_expensive_price, most_expensive_rating)

## ⏱️ Timing
| Field | Value |
|-------|-------|
| **Start** | 14:37:23 |
| **End** | ~14:39:40 |
| **Duration** | ~2 minutes 17 seconds |
| **Steps** | 9 |
| **Input Tokens** | 87,701 |
| **Result** | ✅ COMPLETED |

## 📋 Step Details
| Order | Status | Retry | Input Tokens | Output Tokens |
|-------|--------|-------|-------------|---------------|
| 0 | completed | 1 | 9,227 | 458 |
| 0 | failed | 0 | 8,062 | 428 |
| 1 | completed | 0 | 13,598 | 392 |
| 2 | completed | 0 | 12,132 | 435 |
| 3 | completed | 0 | 8,396 | 531 |
| 4 | completed | 0 | 10,067 | 497 |
| 5 | completed | 1 | 12,062 | 540 |
| 5 | failed | 0 | 7,895 | 209 |
| 6 | canceled | 0 | 6,262 | 318 |

## ✅ Final Results
| Field | Value |
|-------|-------|
| **DB Status** | `completed` |
| **extracted_information** | empty (schema not filled) |
| **Soumission price** | ⚠️ Found visually, not recorded in schema |
| **Most expensive book** | ⚠️ Found visually (Boar Island £59.48), not recorded in schema |

## 🔍 Differences Compared to Attempts 1 and 2
| Category | Attempt 1 | Attempt 2 | Attempt 3 |
|----------|-----------|-----------|-----------|
| **Duration** | ~4:13 | ~3:49 | **~2:17** |
| **Steps** | 14 | 12 | **9** |
| **Input Tokens** | 134,689 | 115,141 | **87,701** |
| **DB Status** | failed | failed | **completed** |
| **Max Steps** | default | default | **25** |
| **Schema** | none | none | **added** |

## ⚠️ Technical Notes
- Only run to achieve `completed` status
- Despite completed status, extracted_information was still empty — agent completed the task visually but did not populate the schema
- Most efficient run: fewest steps, fewest tokens, fastest completion
- Error `KeyError: 'text'` observed in step 5 — agent tried to parse an INPUT_TEXT action without a text field
