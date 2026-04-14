
# Skyvern — Consistency Test Analysis

## 📝 Task
Navigate `books.toscrape.com`, find 'Soumission' (price + rating), go to Mystery category, find most expensive book, report results.

## 🤖 Configuration
- **Framework**: Skyvern 1.0.30
- **LLM**: Groq / `meta-llama/llama-4-scout-17b-16e-instruct`
- **Approach**: Vision-based browser automation — screenshots sent to LLM each step
- **Note**: Skyvern 2.0 (Code mode) was not tested — requires OpenAI API (paid)

---

## ⏱️ Results Summary

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

**Success rate**: 1/4 (25%) — only Run 3 achieved `completed` status  
**Extracted data success rate**: 0/4 (0%) — no run produced structured output

---

## ✅ Output Consistency

| Field | Run 1 | Run 2 | Run 3 | Run 4 | Consistent? |
|-------|-------|-------|-------|-------|-------------|
| **Soumission price** | visual only | visual only | visual only | visual only | ⚠️ Partial |
| **Soumission rating** | visual only | visual only | visual only | visual only | ⚠️ Partial |
| **Most expensive book** | visual only | visual only | visual only | visual only | ⚠️ Partial |
| **DB extracted_information** | empty | empty | empty | empty | ✅ Consistently empty |

---

## 🔍 Key Observations

### 1. No persistent memory between steps
Skyvern 1.0 sends only the **current screenshot + original goal** to the LLM at each step. It does not include the history of previous steps. This caused the agent to repeatedly find Soumission (6+ times in some runs) without "remembering" it had already been found.

### 2. extracted_information_schema not populated
Even in Run 3 (completed status), the extracted_information field in the database was empty. Skyvern 1.0 with Groq models does not reliably populate structured output schemas. The agent completes the task visually but does not format results into the requested schema.

### 3. Run 3 was the most efficient despite fewest parameters
Run 3 had the fewest steps (9), fastest completion (~2:17), and was the only run to achieve `completed` status. This suggests that more steps/tokens does not correlate with better results for Skyvern 1.0.

### 4. New terminal did not improve results
Run 4 (new terminal, most detailed goal, most steps) still failed. This suggests the limitation is in the framework itself, not in session state or context.

### 5. High token consumption
Average input tokens per run: ~109,403. Compare to browser-use (~9 min runs) and open-interpreter (~25 seconds). Skyvern sends full page screenshots at every step, making it significantly more token-intensive.

### 6. Skyvern 1.0 vs 2.0
Only Skyvern 1.0 was tested due to budget constraints. Skyvern 2.0 (Code generation mode) requires OpenAI API and would likely have better memory management and structured output support.

---

## 📊 Comparison with Other Frameworks

| Metric | browser-use | open-interpreter | skyvern 1.0 |
|--------|-------------|-----------------|-------------|
| **Approach** | DOM-based | Code generation | Vision (screenshots) |
| **Avg duration** | ~9 min | ~26 sec | ~3:27 min |
| **Success rate** | 2/3 (67%) | 3/3 (100%) | 1/4 (25%) |
| **Structured output** | ⚠️ Partial | ✅ Yes | ❌ No |
| **Memory between steps** | ✅ Yes | ✅ Yes | ❌ No |
| **Token efficiency** | Medium | Low | High (expensive) |
| **Works with Groq** | ✅ Yes | ✅ Yes | ✅ Yes (1.0 only) |

---

## ✅ Conclusion

Skyvern 1.0 demonstrated the **lowest consistency** of all tested frameworks — only 1 out of 4 runs achieved a `completed` status, and none produced structured extracted data. The framework's reactive, stateless architecture (screenshot per step, no history) makes it poorly suited for multi-step data extraction tasks when using Groq free-tier models.

**Verdict**: ❌ Not consistent for this task type. Better suited for single-page form-filling tasks where memory between steps is less critical.

**Recommended for**: Simple, single-page automation tasks (form filling, button clicking).  
**Not recommended for**: Multi-step data extraction requiring memory across steps.

**Important caveat**: Results may differ significantly with Skyvern 2.0 + OpenAI GPT-4o, which was not tested due to budget constraints.
