# Agent-E Research Report

## Executive Summary

Agent-E is a hierarchical multi-agent web automation framework developed by Emergence AI, built on top of Microsoft AutoGen. It introduces **DOM Distillation** — a technique that compresses raw HTML to only task-relevant elements before passing it to the LLM — and **Change Observation** — a feedback loop that monitors DOM changes after each action without extra LLM calls. It achieved 73.2% on WebVoyager at publication time (July 2024), representing a 20% improvement over prior text-only SOTA. The project is also backed by an arXiv paper synthesising 8 general agentic design principles.

---

## 1. Architecture Overview

### Core Philosophy

**Hierarchical Architecture:** Two distinct LLM-powered agents with different responsibilities — the Planner stays isolated from page-level noise, the Navigator handles messy DOM reality. This separation prevents the Planner from being confused by irrelevant HTML.

**DOM Distillation:** Rather than feeding the full DOM to the LLM, Agent-E trims it to task-relevant elements in three modes: text-only (information retrieval), input-fields-only (form interaction), all-content (full task). The Navigation Agent selects the appropriate mode at runtime.

**Change Observation:** After each action, Agent-E monitors the resulting DOM delta. This Reflexion-style feedback loop lets the agent detect whether the action succeeded and correct course — without paying for an extra LLM call.

### Package Structure

```
agent_e/
├── ae/
│   ├── agent/
│   │   ├── orchestrator.py        # Main Planner + Navigator coordination
│   │   └── web_agent.py           # Browser Navigation Agent class
│   ├── skills/
│   │   ├── click_using_selector.py
│   │   ├── enter_text_using_selector.py
│   │   ├── get_dom_with_content_type.py  # DOM Distillation entry point
│   │   ├── hover_and_click.py
│   │   ├── navigate.py
│   │   ├── get_url.py
│   │   ├── go_back.py
│   │   ├── scroll.py
│   │   ├── save_to_memory.py
│   │   └── wait.py
│   ├── config/
│   │   ├── config.yml             # Main configuration
│   │   └── prompts/               # System prompts for each agent
│   └── utils/
│       ├── dom_helper.py          # Accessibility tree extraction
│       ├── change_observer.py     # Change Observation implementation
│       └── helper.py
├── browser/
│   └── playwright_manager.py     # Playwright browser lifecycle
├── tests/
│   └── tasks/                    # WebVoyager-compatible task JSONs
├── requirements.txt
└── pyproject.toml
```

---

## 2. Core Components Deep Dive

### 2.1 Planner Agent (`ae/agent/orchestrator.py`)

The Planner is the top-level orchestrator. It receives the user's task and never sees raw page content — only sub-task results reported back by the Navigator.

```python
class PlannerAgent:
    """
    Receives high-level task from user.
    Decomposes into sub-tasks.
    Delegates to BrowserNavigationAgent one sub-task at a time.
    Reports final result to user.
    """
    
    def __init__(self, llm_config: dict):
        # AutoGen-based conversation setup
        self.planner = autogen.AssistantAgent(
            name="Planner",
            system_message=PLANNER_PROMPT,
            llm_config=llm_config,
        )
        self.user_proxy = autogen.UserProxyAgent(
            name="UserProxy",
            human_input_mode="NEVER",
            code_execution_config=False,
        )
    
    async def run_task(self, task: str) -> str:
        # Initiate planner → navigator conversation
        # Each sub-task is delegated to NavigationAgent
        # Results aggregated and returned
```

**Planner responsibilities:**
- Decompose complex task into ordered sub-tasks
- Track overall progress
- Receive Navigator success/failure reports
- Decide when the full task is complete
- Remain isolated from DOM complexity

### 2.2 Browser Navigation Agent (`ae/agent/web_agent.py`)

The Navigator executes each sub-task. It sees the page, selects the DOM distillation mode, calls skills, and reports back.

```python
class BrowserNavigationAgent:
    """
    Receives one sub-task from Planner.
    Selects appropriate DOM distillation mode.
    Calls registered skills to interact with page.
    Reports success/failure + change observation result.
    """
    
    def __init__(self, llm_config: dict, playwright_manager: PlaywrightManager):
        self.navigator = autogen.AssistantAgent(
            name="Navigator",
            system_message=NAVIGATOR_PROMPT,
            llm_config=llm_config,
        )
        self.executor = autogen.UserProxyAgent(
            name="Executor",
            human_input_mode="NEVER",
            code_execution_config={"use_docker": False},
            function_map={skill.name: skill.func for skill in SKILLS},
        )
```

### 2.3 DOM Distillation (`ae/skills/get_dom_with_content_type.py`)

This is Agent-E's core innovation — the `get_dom_with_content_type` skill:

```python
async def get_dom_with_content_type(
    content_type: Literal["text", "input_fields", "all_fields"],
    page: Page,
) -> str:
    """
    content_type:
    - "text"         → Extract only text content (for information retrieval)
    - "input_fields" → Extract only input elements (for form interaction)
    - "all_fields"   → Extract all interactable elements (comprehensive)
    
    Returns: JSON-serialised, compressed DOM snapshot
    """
    accessibility_tree = await page.accessibility.snapshot()
    
    if content_type == "text":
        return extract_text_only(accessibility_tree)
    elif content_type == "input_fields":
        return extract_inputs_only(accessibility_tree)
    else:
        return extract_all_fields(accessibility_tree)
```

**Why Accessibility Tree instead of raw HTML?**
- Accessibility tree is already semantically structured — buttons are buttons, inputs are inputs
- Much smaller than raw HTML
- Closer to what automation agents need than visual pixel-based approaches
- Screen reader oriented = automation oriented

**Token savings:** On e-commerce sites, DOM distillation reduces context from ~15,000 tokens (raw HTML) to ~500-2,000 tokens (distilled). This reduces LLM costs and improves reasoning quality.

### 2.4 Skills Registry (AutoGen Function Calling)

Skills are Python functions registered into AutoGen's function-calling system:

```python
# Example skill definition
async def click_using_selector(
    css_selector: str,
    page: Page,
) -> str:
    """Click an element identified by CSS selector."""
    try:
        await page.click(css_selector)
        # Change observation happens here
        change = await observe_change(page)
        return f"Clicked {css_selector}. Page change: {change}"
    except Exception as e:
        return f"Failed to click {css_selector}: {str(e)}"
```

**Available Skills:**

| Skill | Description |
|-------|-------------|
| `get_dom_with_content_type` | DOM Distillation (core perception) |
| `click_using_selector` | Click by CSS selector |
| `enter_text_using_selector` | Type into field by selector |
| `hover_and_click` | Hover then click (for dropdowns) |
| `navigate` | Go to URL |
| `go_back` | Browser history back |
| `get_url` | Get current URL |
| `scroll` | Scroll up/down |
| `save_to_memory` | Store information for later reference |
| `wait` | Wait N seconds |
| `get_focused_element` | Get currently focused element |
| `done` | Mark task complete |

Skills return **natural language strings** — not booleans. The Navigator uses the text response to understand what happened:
```
"Clicked #submit-button. Page changed: New page loaded at /confirmation"
vs.
"Failed to click #submit-button: Element not found"
```
This conversational feedback was found to significantly improve agent recovery from errors.

### 2.5 Change Observation (`ae/utils/change_observer.py`)

After each skill execution, Agent-E checks what changed on the page:

```python
async def observe_change(
    page: Page,
    previous_dom: str,
    timeout_ms: int = 2000,
) -> str:
    """
    Monitor DOM mutations after action.
    Returns human-readable description of what changed.
    No extra LLM call required — deterministic comparison.
    """
    # Wait for DOM to settle
    await page.wait_for_load_state("networkidle", timeout=timeout_ms)
    
    current_dom = await page.content()
    
    # Compare previous vs current
    changes = diff_dom(previous_dom, current_dom)
    
    return format_changes_as_natural_language(changes)
    # → "Form submitted. New section appeared: Order Confirmation #12345"
```

This is conceptually similar to **Reflexion** (Shinn et al., 2024) — verbal feedback after each action guides the agent toward more accurate next steps.

---

## 3. Agent Execution Flow

### High-Level Flow

```
User submits task: "Find the cheapest flight from NYC to London"
    ↓
PlannerAgent receives task
    ↓
Planner decomposes:
  Sub-task 1: "Navigate to a flight search site"
  Sub-task 2: "Enter origin NYC and destination London"
  Sub-task 3: "Sort by price and extract cheapest option"
    ↓
For each sub-task, delegate to BrowserNavigationAgent
    ↓
┌──── Navigation Agent Step Loop ────┐
│                                     │
│  1. get_dom_with_content_type(...)  │
│     → Compressed DOM snapshot       │
│                                     │
│  2. LLM decides next skill          │
│     → "click_using_selector(#sort)" │
│                                     │
│  3. Execute skill                   │
│     → Playwright action             │
│                                     │
│  4. change_observer.observe_change()│
│     → "Dropdown appeared with       │
│        options: Price, Duration..." │
│                                     │
│  5. Feed change back to LLM         │
│  6. Continue until sub-task done    │
│                                     │
└─────────────────────────────────────┘
    ↓
Report result to Planner
    ↓
Planner continues to next sub-task
    ↓
All sub-tasks complete → Return final result
```

### Data Flow

```
Task (string)
    ↓
PlannerAgent (AutoGen AssistantAgent)
    ↓
Sub-task delegated to BrowserNavigationAgent
    ↓
get_dom_with_content_type("text"|"input_fields"|"all_fields")
    → Playwright page.accessibility.snapshot()
    → Compressed JSON representation
    ↓
LLM (GPT-4o, function calling)
    → Selects skill + parameters
    ↓
Skill execution (Playwright)
    ↓
change_observer.observe_change()
    → Natural language change description
    ↓
Change fed back to LLM context
    ↓
Next action or "done"
```

---

## 4. Configuration

### `config/config.yml`

```yaml
llm_config:
  model: "gpt-4o"
  api_key: "${OPENAI_API_KEY}"
  temperature: 0.0
  
browser:
  headless: false
  viewport_width: 1280
  viewport_height: 720
  
agent:
  max_steps_per_subtask: 15
  planner_max_retries: 3
```

### Setup

```bash
git clone https://github.com/EmergenceAI/Agent-E
cd Agent-E
pip install -r requirements.txt
# Set OPENAI_API_KEY in .env
playwright install chromium
python -m ae.main --task "Your task here"
```

### Testing (WebVoyager format)

```json
// tests/tasks/example_task.jsonl
{
  "task_id": "001",
  "web": "amazon.com",
  "ques": "Find the price of the Kindle Paperwhite",
  "web_name": "Amazon"
}
```

---

## 5. Architecture Diagrams

### Component Hierarchy

```
Agent-E
  ├─ PlannerAgent (AutoGen AssistantAgent)
  │   ├─ System Prompt: PLANNER_PROMPT
  │   └─ AutoGen UserProxy (task delivery)
  ├─ BrowserNavigationAgent (AutoGen AssistantAgent)
  │   ├─ System Prompt: NAVIGATOR_PROMPT
  │   ├─ AutoGen Executor (function calling)
  │   └─ Skills Registry (12+ Playwright functions)
  ├─ DOM Distillation
  │   ├─ text mode → text-only extraction
  │   ├─ input_fields mode → form-only extraction
  │   └─ all_fields mode → full interactable tree
  ├─ Change Observer
  │   └─ DOM diff → natural language feedback
  └─ PlaywrightManager
      └─ Chromium browser
```

### Two-Agent Communication

```
User Task
    ↓
PlannerAgent
    │ Sub-task 1
    ↓
NavigationAgent ──→ DOM Distillation ──→ LLM
    │                                      │
    │ ←─── Change Observation ─────────────┘
    │ Sub-task result
    ↓
PlannerAgent
    │ Sub-task 2
    ↓
NavigationAgent ...
    ↓
Final Result
```

---

## 6. Key Technical Decisions

**Why AutoGen for the multi-agent framework?**
- AutoGen provides structured conversation management between agents
- Function calling registry handles skill dispatch cleanly
- Allows easy addition of new skills by registering them
- Conversation history naturally serves as agent memory

**Why Accessibility Tree over raw HTML?**
- Semantic structure already extracted — no parsing needed
- Contains only meaningful elements for interaction
- Much smaller than raw HTML (10-50× compression typical)
- Directly maps to user-facing concepts (buttons, links, inputs)

**Why three DOM distillation modes?**
- Information retrieval (reading text) doesn't need input field structure
- Form interaction doesn't need paragraph text
- Adaptive selection reduces tokens by choosing the right mode
- Navigation Agent selects mode based on current sub-task goal

**Why natural language skill responses instead of booleans?**
- Boolean `True/False` gives the LLM no information for recovery
- "Clicked button, page navigated to /checkout" tells the agent what happened
- LLM can then decide if the result matches the intended goal
- On errors, natural language messages guide the agent toward alternative strategies

---

## 7. Limitations & Known Issues

- **AutoGen dependency** — inherits AutoGen's verbosity, function-calling overhead, and token usage for describing available skills
- **GPT-4o/4 dependency** — primarily tested with OpenAI; other models may produce worse AutoGen conversation formatting
- **Shadow DOM** — accessibility tree-based distillation misses shadow DOM elements (noted as limitation)
- **No built-in caching** — each run starts fresh; no Stagehand-style self-healing cache
- **Research-grade code** — less production-hardened than browser-use or Stagehand; fewer CI checks, fewer contributors
- **Long task degradation** — context window fills up on tasks requiring many steps; no automatic trimming

---

## 8. Conclusion

Agent-E represents a significant academic contribution to web agent design, demonstrating that hierarchical decomposition + DOM distillation + change observation can achieve state-of-the-art results. Its 8 design principles paper is one of the most cited practical frameworks for understanding web agent architecture.

**WebVoyager score:** 73.1% (Steel leaderboard, full 643-task suite, DOM-only, no vision)  
**Paper:** Abuelsaad et al., arXiv 2407.13032, July 2024  
**License:** MIT  
**Language:** Python 3.11+  
**Repository:** https://github.com/EmergenceAI/Agent-E  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub source analysis, arXiv paper 2407.13032, README.md, and skills/ directory structure.*
