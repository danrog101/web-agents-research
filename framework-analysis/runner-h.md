# Runner H Research Report

## Executive Summary

Runner H is an AI web agent developed by **H Company** (Paris, based on $220M raised), positioned as the most accurate web agent on the Steel leaderboard with its newer **Surfer 2** model at 97.1%. The framework is unique because H Company trained **in-house foundation models** (Hollow One family) specifically for GUI automation — a 3B VLM and 2B LLM that outperform much larger general-purpose models for browser tasks. Core model weights are open-sourced under Apache-2.0; the hosted product is commercial.

---

## 1. Architecture Overview

### Core Philosophy

**Specialised small models beat large general models:** H Company's thesis is that a 3B parameter VLM trained specifically on GUI interaction data outperforms GPT-4o for web agent tasks at a fraction of the cost. This validates the "task-specific training" approach over prompt engineering of general models.

**Reinforcement Learning training:** The Hollow One models were trained with RL, not just supervised fine-tuning, enabling the models to develop strategies that work across diverse web interfaces.

### Hollow One Model Family

```
hcompany/
├── hollow-one-7b-policy     (HuggingFace, Apache-2.0)
│   # Plans steps to achieve a goal
│   # Input: goal string
│   # Output: list of step descriptions
│
├── hollow-one-7b-localizer  (HuggingFace, Apache-2.0)
│   # Maps natural language targets to pixel coordinates
│   # Input: screenshot + target description
│   # Output: (x, y) coordinates
│
└── hollow-one-7b-validator  (HuggingFace, Apache-2.0)
    # Verifies task completion
    # Input: screenshot + goal + action history
    # Output: success/failure + explanation
```

**H-VLM (3B parameter):** Vision-Language Model trained to extract information from and interact with GUIs and images.  
**H-LLM (2B parameter):** Language Model for task planning and reasoning.

---

## 2. Core Components Deep Dive

### 2.1 Hollow One Policy (`hcompany/hollow-one-7b-policy`)

```python
from hollow_one import Policy

policy = Policy.from_pretrained("hcompany/hollow-one-7b-policy")

# Plan a multi-step task
steps = policy.plan(
    goal="Extract 10 top AI tools listings and add to a Google Sheet"
)
# Returns list of step strings:
# ["Navigate to ProductHunt", "Search for AI tools",
#  "Extract top 10 listings", "Open Google Sheets", ...]
print("\n".join(steps))
```

### 2.2 Hollow One Localizer (`hcompany/hollow-one-7b-localizer`)

```python
from hollow_one import Localizer
import PIL.Image as Image

loc = Localizer.from_pretrained("hcompany/hollow-one-7b-localizer")

# Find element by natural language description in screenshot
xy = loc.find(
    screenshot=Image.open("page_screenshot.png"),
    target="Search button"
)
# Returns: (340, 220) — pixel coordinates

# Execute click via browser automation
browser.click(*xy)
```

### 2.3 Surfer H SDK

```python
from surfer_h import SurferAgent

agent = SurferAgent(
    policy="hollow-one-7b-policy",
    localizer="hollow-one-7b-localizer",
    validator="hollow-one-7b-validator"
)

result = agent.run(
    task="Book a Sarova Stanley hotel Aug 3-6 and export confirmation to Notion",
    budget_usd=0.25  # Cost limit
)
```

### 2.4 WebClick Dataset

H Company open-sourced WebClick — 30,000 episodes of click-level annotations:
```
WebClick Dataset (Apache-2.0)
├── 30,000 click episodes
├── Screenshots + target descriptions + click coordinates
└── Used to train the Localizer model
```

---

## 3. Execution Flow

```
Goal: "Book hotel and export to Notion"
    ↓
Policy.plan(goal)
    → ["1. Navigate to booking.com",
       "2. Search for hotel name",
       "3. Select dates Aug 3-6",
       ...]
    ↓
For each step:
    Screenshot captured
    ↓
    Localizer.find(screenshot, step_target)
    → (x, y) coordinates
    ↓
    Browser.click(x, y) or keyboard.type(...)
    ↓
    Validator.verify(screenshot, step_goal)
    → success? continue : retry/recover
    ↓
All steps complete → task done
```

---

## 4. Architecture Diagram

```
Runner H / Surfer H
  ├─ Hollow One Policy (7B)
  │   └─ Goal → Step list
  ├─ Hollow One Localizer (7B)
  │   └─ Screenshot + Description → (x, y)
  ├─ Hollow One Validator (7B)
  │   └─ Screenshot + Goal → Success/Fail
  └─ Browser (Playwright or local)
      └─ Execute mouse/keyboard actions
```

---

## 5. Key Technical Decisions

**Why train models instead of prompting GPT-4?**
Smaller specialised models (3B VLM) can match or exceed large general models (GPT-4V) at browser tasks while running at a fraction of the cost. H Company has published benchmarks showing this.

**Why separate Policy/Localizer/Validator?**
Decomposition into three single-purpose models improves accuracy — each model is optimised for its specific subtask. Contrast with end-to-end models that must handle all three simultaneously.

---

## 6. Limitations & Known Issues

- GPU required for local inference (7B models need ~16GB VRAM)
- Full product is commercial; open weights are research access
- Limited documentation for `surfer_h` and `hollow_one` packages
- No WebVoyager score for open-source weights (only hosted Runner H)

**WebVoyager score:** Surfer 2 = 97.1% SOTA (hosted product), Runner H 0.1 = 67%  
**License:** Apache-2.0 (model weights), Commercial (hosted product)  
**Repository:** https://github.com/hcompany/hollow-one (model weights on HuggingFace)  
**Snapshot date:** 2026-03-18
