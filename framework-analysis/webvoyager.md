# WebVoyager Research Report

## Executive Summary

WebVoyager is both an **academic research agent** and the **de facto standard benchmark** for evaluating web agents. Developed by Zhejiang University (published January 2024), it introduced the Set-of-Marks (SoM) approach: overlaying numbered bounding boxes on screenshots so the LLM can reference elements by number. The **benchmark dataset** (643 tasks across 15 real websites) is far more important than the agent implementation, which has been superseded by modern frameworks. Almost every web agent in the field reports WebVoyager scores, making it essential to understand.

---

## 1. Architecture Overview

### Core Philosophy

**Multimodal web navigation:** WebVoyager was the first major work to demonstrate end-to-end task completion on real (live) websites using a multimodal LLM (GPT-4V). Prior work used synthetic environments (MiniWoB, WebArena).

**Set-of-Marks (SoM):** Rather than parsing raw HTML or using vision-only pixel coordinates, WebVoyager overlays numbered labels on every interactive element in the screenshot. The LLM then references elements by number (`CLICK [7]`). This bridges DOM and vision approaches.

### Repository Structure

```
WebVoyager/
├── run.py                    # Main execution script
├── agent.py                  # WebVoyager agent implementation
├── data/
│   ├── WebVoyager_data.jsonl  # 643 task definitions
│   ├── tasks_test.jsonl       # Test subset
│   └── reference_answer.json  # Reference answers for evaluation
├── evaluation/
│   └── evaluate.py           # GPT-4V based automated evaluator
├── requirements.txt
└── README.md
```

---

## 2. Core Components Deep Dive

### 2.1 Agent Implementation

```python
# The original WebVoyager agent uses Selenium + GPT-4V

class WebVoyagerAgent:
    def __init__(self, api_key: str, max_iter: int = 15):
        self.llm = OpenAI(api_key=api_key)
        self.model = "gpt-4-vision-preview"
        self.driver = webdriver.Chrome()  # Selenium
        self.max_iter = max_iter
    
    def run(self, task: str, url: str) -> str:
        self.driver.get(url)
        
        for iteration in range(self.max_iter):
            # 1. Take screenshot
            screenshot = self.driver.get_screenshot_as_png()
            
            # 2. Annotate with numbered bounding boxes (SoM)
            annotated_screenshot = self.add_set_of_marks(screenshot)
            
            # 3. Call GPT-4V with annotated screenshot + task
            response = self.llm.chat.completions.create(
                model=self.model,
                messages=[{
                    "role": "user",
                    "content": [
                        {"type": "text", "text": f"Task: {task}"},
                        {"type": "image_url", "image_url": {"url": to_base64(annotated_screenshot)}}
                    ]
                }]
            )
            
            # 4. Parse action from response
            action = parse_action(response.choices[0].message.content)
            
            # 5. Execute action
            if action.type == "ANSWER":
                return action.text
            elif action.type == "CLICK":
                element = self.get_element_by_mark(action.mark_id)
                element.click()
            elif action.type == "TYPE":
                element = self.get_element_by_mark(action.mark_id)
                element.send_keys(action.text)
            elif action.type == "SCROLL":
                self.scroll(action.direction)
        
        return "Task not completed within max iterations"
```

### 2.2 Action Space

The LLM outputs one of these action strings:

```
CLICK [7]               → Click the element marked [7]
TYPE [3] search text    → Type "search text" into element [3]
SCROLL [WINDOW] down    → Scroll the page down
SCROLL [7] down         → Scroll element [7] down
GOBACK                  → Browser back
GOOGLE                  → Go to google.com
ANSWER: [final answer]  → Task complete, return answer
```

### 2.3 Set-of-Marks (SoM) Annotation

```python
def add_set_of_marks(screenshot: bytes, elements: list) -> bytes:
    """
    Overlay numbered bounding boxes on all interactable elements.
    
    Input:  raw screenshot
    Output: screenshot with colored numbered rectangles on each element
    
    Result: "[7]" appears visually on the "Add to Cart" button
    The LLM can then say "CLICK [7]" without needing CSS selectors
    """
    image = Image.open(io.BytesIO(screenshot))
    draw = ImageDraw.Draw(image)
    
    for i, element in enumerate(elements):
        # Draw colored bounding box
        draw.rectangle(element.bbox, outline="red", width=2)
        # Add number label
        draw.text((element.bbox[0], element.bbox[1]), str(i+1), fill="red")
    
    return image_to_bytes(image)
```

### 2.4 Benchmark Dataset (`data/WebVoyager_data.jsonl`)

```json
{
  "task_id": "Amazon_23",
  "web": "Amazon",
  "ques": "Search for the book 'Atomic Habits' by James Clear and find its current price",
  "web_name": "Amazon",
  "relevant_url": "https://www.amazon.com"
}
```

**643 tasks across 15 websites:**

| Website | Task count | Task types |
|---------|-----------|------------|
| Amazon | 40+ | Product search, price, reviews |
| eBay | 40+ | Listings, pricing |
| Google | 40+ | Search, Maps queries |
| Google Maps | 40+ | Navigation, places |
| Wikipedia | 40+ | Information retrieval |
| Reddit | 40+ | Posts, communities |
| GitHub | 40+ | Repos, issues, code |
| ArXiv | 40+ | Papers, authors |
| Cambridge Dictionary | 40+ | Definitions, pronunciation |
| Booking.com | 40+ | Hotels, prices (time-sensitive!) |
| Allrecipes | 40+ | Recipes, ingredients |
| Apple | 40+ | Products, support |
| ESPN | 40+ | Sports scores, teams |
| Coursera | 40+ | Courses, certificates |
| Wolfram Alpha | 40+ | Computations |

### 2.5 Automated Evaluation

```python
def evaluate_task(
    task: str,
    agent_answer: str,
    reference_answer: str,
    final_screenshot: bytes,
) -> bool:
    """
    GPT-4V evaluates whether the agent's answer is correct.
    
    Compares: agent answer + final screenshot vs reference answer
    Returns: True (correct) / False (incorrect)
    """
    response = openai_client.chat.completions.create(
        model="gpt-4-vision-preview",
        messages=[{
            "role": "user",
            "content": [
                {"type": "text", "text": EVALUATION_PROMPT.format(
                    task=task,
                    agent_answer=agent_answer,
                    reference=reference_answer,
                )},
                {"type": "image_url", "image_url": {"url": to_base64(final_screenshot)}}
            ]
        }]
    )
    return parse_evaluation(response)
```

---

## 3. Benchmark Score Calculation

```
Score = (tasks completed correctly / total tasks) × 100

Example: Magnitude = 93.9%
  → Completed 604/643 tasks correctly (on full dataset)
  → Or on filtered subset: e.g., 47/50 tasks

⚠️ IMPORTANT: Not all scores use the same dataset!
  - Full 643-task suite → lower scores (live sites change)
  - Skyvern filtered 50-task subset → higher scores
  - Always check methodology when comparing
```

---

## 4. Why WebVoyager Scores Vary

From the Steel leaderboard methodology note:

```
Factors affecting comparability:

1. Dataset variant
   - Full 643 tasks (harder, more drift over time)
   - Skyvern filtered 50 tasks (removed impossible tasks)
   - Custom subsets (hand-picked favourable tasks)

2. Evaluator
   - GPT-4V judge (standard)
   - Custom evaluator (may be more/less strict)
   - Human evaluation (most accurate, expensive)

3. Verification
   - Third-party reproduced
   - Self-reported
   - No methodology provided

Rule: Scores from different conditions are NOT directly comparable.
```

---

## 5. Architecture Diagram

```
User Task (natural language)
    ↓
Selenium browser → navigate to start URL
    ↓
┌─── WebVoyager Loop (max 15 iterations) ────┐
│                                              │
│  1. Screenshot (Selenium)                    │
│  2. Add Set-of-Marks (PIL — numbered boxes)  │
│  3. GPT-4V (annotated screenshot + task)     │
│  4. Parse action: CLICK[7] / TYPE[3] / etc.  │
│  5. Execute via Selenium                     │
│  6. Check for ANSWER: → done                 │
│                                              │
└──────────────────────────────────────────────┘
    ↓
GPT-4V evaluation (agent answer vs reference)
```

---

## 6. Historical Significance

| Date | Event |
|------|-------|
| 2017 | World of Bits — first synthetic web benchmark |
| 2018 | MiniWoB++ — 104 mini browser tasks |
| 2023 | WebArena — self-hosted realistic web |
| Jan 2024 | **WebVoyager** — first real (live) website benchmark |
| Jul 2024 | Agent-E — 73.2% (20% improvement at publication) |
| Late 2024 | browser-use — 89.1% |
| 2025 | Magnitude — 93.9% |
| Feb 2026 | Surfer 2 — 97.1% SOTA |

---

## 7. Limitations & Known Issues

- **Live website drift** — tasks written in 2023 may reference outdated page structures
- **Time-sensitive tasks** — Booking.com / Google Flights tasks use specific dates that become outdated
- **Ambiguous success criteria** — some tasks have multiple valid answers
- **Selenium-based** — dated compared to modern CDP/Playwright approaches
- **No difficulty stratification** — tasks range from trivial to very hard without labels

---

## 8. Conclusion

WebVoyager the **benchmark** is essential for understanding the web agent field — every paper and product reports these scores. WebVoyager the **agent** is outdated (Selenium + GPT-4V SoM, 59.1% score), useful only as a historical baseline and as an understanding of the SoM perception approach.

**WebVoyager (agent) score:** 59.1% (Steel leaderboard)  
**License:** MIT  
**Language:** Python 3.10  
**Repository:** https://github.com/MinorJerry/WebVoyager  
**Paper:** He et al., arXiv 2401.13919, January 2024  
**Snapshot date:** 2026-03-18

*Generated: 2026-03-18. Based on: GitHub source analysis, arXiv 2401.13919, Steel leaderboard methodology notes, and emergentmind.com benchmark overview.*
