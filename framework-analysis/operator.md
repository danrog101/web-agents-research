# Operator (OpenAI) Research Report

## Executive Summary

OpenAI Operator is a **cloud-hosted, closed-source AI web agent** integrated into ChatGPT, launched January 2025. It uses a proprietary **Computer-Using Agent (CUA) model** (`computer-use-preview`) that receives browser screenshots and outputs pixel-level actions. Unlike open-source frameworks, there is no codebase to clone — the product is a fully managed service. The CUA model is separately accessible via OpenAI API, enabling developers to integrate it into their own tools (e.g., via Stagehand's `agent()` mode).

---

## 1. Architecture Overview

### Core Philosophy

**Vision-first GUI interaction:** The CUA model receives screenshots of the browser and outputs actions (click at x,y, type text, scroll). It does not parse DOM trees or use element selectors — it sees the page visually, exactly as a human does.

**Human-in-the-loop safety:** Operator pauses and requests user confirmation before sensitive operations (login forms, purchases, password fields). This is a deliberate UX decision, not a technical limitation.

**Operator-as-a-service:** Users interact via ChatGPT interface. There is no SDK, no Python library, no self-hosting. The service model is intentional — Operator is a consumer product, not a developer framework.

### Product Structure (as documented)

```
OpenAI Operator (closed-source)
├── ChatGPT Interface
│   └── Operator tab / side panel
├── Computer-Using Agent (CUA) model
│   ├── computer-use-preview (API accessible)
│   └── Internal production model (not API accessible)
├── Cloud Browser Infrastructure
│   └── Managed Chromium instances
├── Human-in-the-loop System
│   └── Confirmation requests for sensitive actions
└── Saved Workflows
    └── Reusable task definitions
```

---

## 2. Core Components Deep Dive

### 2.1 CUA Model (`computer-use-preview`)

The Computer-Using Agent is the core innovation — a model specifically trained (via RL) to interact with GUIs:

```python
# API access via OpenAI SDK
from openai import OpenAI
import base64

client = OpenAI(api_key="sk-...")

# The CUA model takes screenshots and returns actions
response = client.responses.create(
    model="computer-use-preview",
    tools=[{
        "type": "computer_use_preview",
        "display_width": 1024,
        "display_height": 768,
        "environment": "browser",
    }],
    input=[{
        "role": "user",
        "content": [{
            "type": "text",
            "text": "Find the cheapest flight from NYC to London"
        }, {
            "type": "image_url",
            "image_url": {"url": f"data:image/png;base64,{screenshot_b64}"}
        }]
    }]
)

# Response contains tool_use blocks with actions
for block in response.output:
    if block.type == "tool_use":
        action = block.input
        # action.type: "click", "type", "scroll", "key"
        # action.coordinate: [x, y]
        # action.text: str (for type actions)
```

**CUA Action Space:**

| Action | Parameters |
|--------|-----------|
| `click` | `coordinate: [x, y]`, `button: "left"|"right"|"middle"` |
| `double_click` | `coordinate: [x, y]` |
| `type` | `text: str` |
| `key` | `text: str` (e.g., "Return", "ctrl+a") |
| `scroll` | `coordinate: [x, y]`, `direction: "up"|"down"`, `amount: int` |
| `screenshot` | (returns current screen state) |
| `wait` | `duration: float` |

### 2.2 Integration via Third-Party Frameworks

Since Operator itself is closed-source, the CUA model is most commonly integrated through:

**Stagehand (TypeScript):**
```typescript
const agent = stagehand.agent({
    provider: "openai",
    model: "computer-use-preview",
});
await agent.execute("Complete the checkout flow");
```

**Direct API loop:**
```python
# Developer's own agent loop using CUA model
import anthropic  # or openai

def run_cua_loop(task: str, browser):
    screenshot = browser.screenshot()
    
    while True:
        response = client.responses.create(
            model="computer-use-preview",
            tools=[computer_use_tool],
            input=build_messages(task, screenshot)
        )
        
        action = extract_action(response)
        if action.type == "done":
            break
            
        execute_action(browser, action)
        screenshot = browser.screenshot()
```

### 2.3 Hosted Product Features

The ChatGPT-integrated Operator provides:
- **Natural language task input** — describe what you want in conversational language
- **Saved workflows** — store frequently-used task definitions
- **Live browser view** — watch Operator work in real-time
- **Confirmation prompts** — human approval for sensitive actions
- **Task scheduling** — run workflows on a schedule (via API)

---

## 3. Execution Flow

### User Flow (Hosted Product)

```
User types task in ChatGPT Operator
    ↓
Task sent to Operator service
    ↓
Cloud browser instance launched
    ↓
┌─────── CUA LOOP ──────────┐
│                             │
│  Take screenshot            │
│  → CUA model analyses       │
│  → Outputs next action      │
│                             │
│  Check: sensitive action?   │
│  → YES: pause, ask user     │
│  → NO: execute directly     │
│                             │
│  Execute via cloud browser  │
│  Playwright-compatible      │
│                             │
└─────────────────────────────┘
    ↓
Task complete → return result to user
```

### API Flow (Developer Access)

```
Developer calls OpenAI API with screenshot + task
    ↓
CUA model (computer-use-preview)
    → Analyses screenshot visually
    → Returns action with pixel coordinates
    ↓
Developer's code executes action
    (Playwright, Puppeteer, or Stagehand)
    ↓
New screenshot → next iteration
```

---

## 4. Configuration

### ChatGPT Product (No Config)

```
1. Navigate to chat.openai.com
2. Select Operator from tools
3. Type your task
4. Watch it work
```

### API Access

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# Requires: Plus, Pro, or API subscription with computer-use access
response = client.responses.create(
    model="computer-use-preview",
    # ...
)
```

---

## 5. Architecture Diagram

```
User (ChatGPT / API)
    ↓
OpenAI Operator Service (closed)
    ├─ Task orchestration
    ├─ Human-in-the-loop safety gates
    └─ Cloud browser pool (managed Chromium)
        ↓
CUA Model (computer-use-preview)
    Input:  screenshot + task + history
    Output: action { type, coordinate, text }
        ↓
Browser action executed
    ↓
New screenshot → loop
```

---

## 6. Key Technical Decisions

**Why vision-only (no DOM)?**
The CUA model was trained via RL on GUI interaction across diverse applications. Vision-only generalises to any application (browser, desktop, mobile) without needing application-specific DOM parsers.

**Why human-in-the-loop?**
Operator's safety philosophy: autonomous agents must not make irreversible mistakes. Payments, logins, and form submissions pause for confirmation. This is a deliberate product decision, not a technical limitation.

**Why closed-source?**
OpenAI has not published the CUA model training procedure or safety evaluation methodology. The closed-source approach reflects commercial and safety considerations.

---

## 7. Limitations & Known Issues

- **Closed-source** — no code to inspect, modify, or self-host
- **OpenAI account required** — ChatGPT Plus/Pro subscription (~$20-200/month)
- **Privacy** — all browsing activity goes through OpenAI servers
- **No customisation** — cannot add custom skills, adjust prompts, or change behaviour
- **Geographic restrictions** — not available in all countries
- **API rate limits** — `computer-use-preview` has usage limits
- **87% WebVoyager** — lower than open-source alternatives like Magnitude (93.9%)

---

## 8. Conclusion

Operator represents the commercial benchmark that open-source frameworks compete against. Its 87% WebVoyager score establishes a reference point, but multiple open-source alternatives (browser-use 89.1%, Magnitude 93.9%, Browserable 90.4%) now exceed it. Operator's value is its zero-setup consumer experience and human-in-the-loop safety — not raw technical capability.

**WebVoyager score:** 87% (Steel leaderboard)  
**License:** Closed-source / Commercial  
**Language:** N/A (no public codebase)  
**Repository:** N/A  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: OpenAI documentation, API reference, published benchmarks, and third-party integrations.*
