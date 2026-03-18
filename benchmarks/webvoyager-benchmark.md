
# WebVoyager Benchmark Report

## Executive Summary

WebVoyager (He et al., Zhejiang University, January 2024) introduced the **643-task benchmark on 15 live real-world websites** that became the de facto standard for comparing production web agents. It also introduced the **Set-of-Marks (SoM)** annotation method — overlaying numbered bounding boxes on screenshots to give the LLM stable element references — and demonstrated end-to-end task completion on real websites with GPT-4V. The benchmark is the primary ranking system for the Steel Leaderboard and is cited in virtually every web agent paper published after 2024.

*Note: This report covers the WebVoyager BENCHMARK specifically. For the WebVoyager agent implementation, see the separate `11-webvoyager-report.md`.*

---

## 1. Background & Motivation

### What WebVoyager Added

| Feature | Prior benchmarks | WebVoyager |
|---------|-----------------|------------|
| Websites | Simulated (MiniWoB) or self-hosted (WebArena) | 15 live real websites |
| Tasks | 104–812 | 643 |
| Drift | None (controlled) | Yes (live sites change) |
| Setup | Docker required | None — just internet |
| Cost | Free | ~$0.05-0.50/task (LLM) |
| Reproducibility | High | Medium (sites change) |

**Key innovation:** You can run WebVoyager against real Amazon, GitHub, Google Maps, Wikipedia — no Docker setup, no infrastructure. This made it practical for teams without DevOps resources.

### Paper

> He, H., Yao, W., Ma, K., Yu, W., Dai, Y., Zhang, H., Lan, Z., & Yu, D.  
> **WebVoyager: Building an End-to-End Web Agent with Large Multimodal Models**  
> arXiv:2401.13919, January 2024

---

## 2. Dataset Structure

### 643 Tasks Across 15 Websites

```json
// data/WebVoyager_data.jsonl — task format
{
    "task_id": "Amazon_23",
    "web": "Amazon",
    "web_name": "Amazon",
    "ques": "Search for the book 'Atomic Habits' by James Clear and find its current price",
    "relevant_url": "https://www.amazon.com",
    "answer": null    // not provided publicly to prevent cheating
}
```

**Website distribution (~40-45 tasks each):**

| Website | Task Types |
|---------|-----------|
| Amazon | Product search, price, reviews, bestsellers |
| eBay | Listings, bids, seller ratings |
| Google | Search results, featured snippets |
| Google Maps | Navigation, business lookup, hours |
| Google Search | News, knowledge panel |
| Wikipedia | Article content, cross-references |
| Reddit | Posts, scores, community info |
| GitHub | Repository stats, code, issues |
| ArXiv | Papers, authors, citations |
| Cambridge Dictionary | Definitions, pronunciation, examples |
| Booking.com | Hotel availability, prices (time-sensitive!) |
| Allrecipes | Recipes, ingredients, ratings |
| Apple | Product specs, support articles |
| ESPN | Sports scores, standings, player stats |
| Coursera | Course info, ratings, duration |
| Wolfram Alpha | Computations, factual lookups |

---

## 3. Set-of-Marks (SoM) Annotation

The core technical contribution of WebVoyager the agent:

```python
from PIL import Image, ImageDraw, ImageFont

def add_set_of_marks(
    screenshot: bytes,
    interactive_elements: list[Element]
) -> bytes:
    """
    Overlay numbered bounding boxes on all interactive elements.
    
    Before: plain screenshot
    After:  screenshot with colored numbered rectangles
            "[7]" appears visually on "Add to Cart" button
    
    LLM can now reference: "CLICK [7]" without CSS selectors
    """
    image = Image.open(io.BytesIO(screenshot))
    draw = ImageDraw.Draw(image)
    font = ImageFont.truetype("arial.ttf", 14)
    
    for i, element in enumerate(interactive_elements):
        mark_id = i + 1
        
        # Draw colored bounding box
        draw.rectangle(
            [element.bbox.left, element.bbox.top,
             element.bbox.right, element.bbox.bottom],
            outline="red" if element.is_button else "blue",
            width=2
        )
        
        # Draw number label
        draw.rectangle(
            [element.bbox.left, element.bbox.top,
             element.bbox.left + 20, element.bbox.top + 18],
            fill="red" if element.is_button else "blue"
        )
        draw.text(
            (element.bbox.left + 2, element.bbox.top + 1),
            str(mark_id), fill="white", font=font
        )
    
    output = io.BytesIO()
    image.save(output, format="PNG")
    return output.getvalue()
```

---

## 4. Evaluation Method

### Automated GPT-4V Evaluation

WebVoyager introduced **LLM-as-judge** for web agent evaluation — the first benchmark to use a vision model to assess whether the final page state matched the task intent:

```python
def evaluate_task(
    task: str,
    agent_answer: str,
    reference_answer: str,
    final_screenshot: bytes,
) -> bool:
    """
    GPT-4V evaluates success by comparing:
    - Agent's final answer text
    - Final screenshot of the page
    - Reference answer from annotators
    
    Returns: True (success) / False (failure)
    """
    response = openai.chat.completions.create(
        model="gpt-4-vision-preview",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": EVALUATION_PROMPT.format(
                    task=task,
                    agent_answer=agent_answer,
                    reference=reference_answer,
                )},
                {"type": "image_url", "image_url": {
                    "url": f"data:image/png;base64,{to_base64(final_screenshot)}"
                }}
            ]
        }]
    )
    verdict = response.choices[0].message.content
    return "success" in verdict.lower()
```

### Score Calculation

```
Score = (tasks judged as completed) / (total tasks) × 100

Example: browser-use score = 89.1%
  → 573 of 643 tasks judged as completed by GPT-4V
  (or equivalent on a filtered subset)
```

---

## 5. Steel Leaderboard Rankings

The Steel Leaderboard aggregates WebVoyager scores for all web agent frameworks:

| Rank | Agent | Score | Notes |
|------|-------|-------|-------|
| 1 | Surfer 2 (H Company) | 97.1% | Hosted product |
| 2 | Magnitude | 93.9% | Skyvern 50-task subset, Claude Sonnet 4 |
| 3 | AIME Browser-Use | 92.34% | Browser-use + AIME modifications |
| 4 | Browserable | 90.4% | Self-reported |
| 5 | Browser Use | 89.1% | Standard, gpt-4o |
| 6 | Operator (OpenAI) | 87% | Closed-source |
| 7 | Skyvern 2.0 | 85.85% | — |
| 8 | Project Mariner (Google) | 83.5% | Closed-source |
| 11 | Agent-E | 73.1% | DOM-only |
| 13 | Runner H 0.1 | 67% | Open weights |
| 14 | WebVoyager (original) | 59.1% | GPT-4V, baseline |

⚠️ **Comparability warning:** Not all scores use the same dataset variant or evaluation protocol. See Section 6.

---

## 6. Score Comparability Issues

This is critically important for thesis analysis:

```
Factor 1: Dataset variant
  - Full 643 tasks — used by most (WebVoyager, browser-use, Agent-E)
  - Skyvern filtered 50 tasks — removes tasks that became unsolvable
    due to website changes; Magnitude and some others use this
  - Custom subsets — not comparable

Factor 2: Evaluator
  - GPT-4V judge (standard per original paper)
  - GPT-4o judge (newer, may be more lenient)
  - Human evaluation (gold standard, expensive)
  - Custom evaluator (varies by team)

Factor 3: Verification
  - Third-party reproduced (most reliable)
  - Self-reported (common)
  - No methodology disclosed (treat with caution)

Factor 4: Time of evaluation
  - Live sites change — a score from 2024 may not match 2026
    because Booking.com or ESPN changed their UI
  - Higher volatility for time-sensitive tasks (hotel prices, scores)

Conclusion:
  Only direct same-condition comparisons are scientifically valid.
  The Steel Leaderboard scores provide directional guidance,
  not precise capability deltas.
```

---

## 7. Repository Structure

```
WebVoyager/                         (GitHub: MinorJerry/WebVoyager)
├── data/
│   ├── WebVoyager_data.jsonl       # 643 task definitions
│   ├── tasks_test.jsonl            # Test subset
│   └── reference_answers.json     # Reference answers for eval
├── evaluation/
│   └── evaluate.py                # GPT-4V based evaluator
├── run.py                         # Main execution script
├── agent.py                       # WebVoyager agent (Selenium + GPT-4V + SoM)
├── prompts/                       # System prompts
└── requirements.txt
```

---

## 8. Using the Benchmark (Evaluation Only)

```bash
git clone https://github.com/MinorJerry/WebVoyager
pip install -r requirements.txt

# Your agent produces results in this format:
# results/agent_name/task_id/
#   ├─ answer.txt       # Agent's final answer
#   └─ screenshot.png   # Final page screenshot

# Run evaluation
python evaluation/evaluate.py \
    --results_dir results/my_agent \
    --data_file data/WebVoyager_data.jsonl \
    --references_file data/reference_answers.json \
    --model gpt-4-vision-preview
    
# Output:
# Task Amazon_23: SUCCESS (agent answered "14.99", reference "14.99")
# Task Google_07: FAIL (agent answered "2024", reference "2023")
# ...
# Overall: 574/643 = 89.3%
```

---

## 9. Significance in the Field

WebVoyager's key contributions:

1. **Live website benchmark** — the first to test on real (not simulated or self-hosted) sites
2. **Set-of-Marks** — the numbered annotation technique that improved element referencing
3. **LLM-as-judge** — pioneered automated evaluation using vision models
4. **De facto standard** — the Steel Leaderboard is WebVoyager-based; almost all papers report it
5. **Accessibility** — no infrastructure required (vs WebArena's Docker setup)

**Limitations:**
- Live site drift reduces reproducibility over time
- LLM judge can disagree with humans (~10-15% disagreement rate)
- No difficulty stratification
- Time-sensitive tasks (Booking.com) become invalid quickly

**Paper:** arXiv:2401.13919, January 2024  
**License:** MIT  
**Repository:** https://github.com/MinorJerry/WebVoyager  
**Steel Leaderboard:** https://github.com/steel-dev/awesome-web-agents  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: arXiv:2401.13919, GitHub source, Steel leaderboard methodology notes, and Magnitude's WebVoyager reproducibility discussion.*
