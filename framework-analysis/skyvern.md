# Skyvern Research Report

## Executive Summary

Skyvern automates browser-based workflows using LLMs and computer vision. It provides a Playwright-compatible SDK that adds AI capabilities on top of Playwright, plus a no-code workflow builder. Unlike DOM-based frameworks, Skyvern relies on **vision LLMs** to interpret screenshots and decide which elements to interact with — making it robust to layout changes and capable of operating on websites it has never seen before. The architecture uses a **swarm of specialised agents** each responsible for a different aspect of the task.

---

## 1. Architecture Overview

### Core Philosophy

Skyvern was inspired by the Task-Driven autonomous agent design popularised by BabyAGI and AutoGPT — with one major addition: the ability to interact with websites through Playwright. Three core principles:

**Vision-First Interaction:** Instead of parsing the DOM and looking for XPath selectors, Skyvern takes screenshots and sends them to a vision LLM. This makes it resilient to layout changes — no selectors ever break.

**Agent Swarm Architecture:** Skyvern decomposes tasks across multiple specialised agents, each with a specific responsibility (element detection, navigation planning, data extraction, password handling, 2FA, etc.).

**Explore → Replay Pattern (Skyvern 2.0):** The first run uses full LLM inference to explore the workflow. Subsequent runs use cached Playwright code, with LLM fallback only when the site changes. This makes repeated runs 2.7× cheaper and 2.3× faster.

### Package Structure

```
skyvern/
├── forge/
│   ├── agent.py              # ForgeAgent — main orchestrator
│   ├── app.py                # Application entrypoint, ForgeApp
│   ├── api_app.py            # FastAPI application, HTTP endpoints
│   ├── prompts/              # LLM prompt templates
│   │   └── prompt_engine.py  # Prompt rendering
│   ├── sdk/
│   │   ├── api/
│   │   │   ├── llm/
│   │   │   │   └── api_handler_factory.py  # LLMCaller
│   │   │   └── files.py                    # File management
│   │   ├── db/               # Database models and repositories
│   │   ├── workflow/         # Workflow engine
│   │   │   ├── models/
│   │   │   │   └── block.py  # TaskBlock, WorkflowBlock types
│   │   │   └── context_manager.py
│   │   └── schemas/          # Pydantic schemas
│   └── async_operations.py   # AgentPhase, AsyncOperationPool
├── webeye/
│   ├── actions/
│   │   ├── models.py         # AgentStepOutput, DetailedAgentStepOutput
│   │   ├── parse_actions.py  # parse_actions, parse_anthropic_actions, parse_cua_actions
│   │   └── responses.py      # ActionResult, ActionSuccess
│   ├── scraper/
│   │   └── scraper.py        # scrape_website, ElementTreeFormat, ScrapedPage
│   ├── browser_factory.py    # BrowserState, browser session management
│   └── utils/
│       └── page.py           # SkyvernFrame, page utilities
├── integrations/
│   └── mcp/                  # MCP server integration
├── cli/
│   └── commands.py           # CLI entry point (skyvern quickstart, etc.)
└── __init__.py
```

---

## 2. Core Components Deep Dive

### 2.1 ForgeAgent (`skyvern/forge/agent.py`)

The `ForgeAgent` is Skyvern's main orchestrator — it coordinates the agent swarm and manages the task execution lifecycle.

```python
class ForgeAgent:
    def __init__(self) -> None:
        if settings.ADDITIONAL_MODULES:
            for module in settings.ADDITIONAL_MODULES:
                __import__(module)
        # Initialise agent swarm, LLM caller, browser factory

class ActionLinkedNode:
    def __init__(self, action: Action) -> None:
        self.action = action
        self.next: ActionLinkedNode | None = None
```

**Agent Swarm (specialised agents):**

| Agent | Responsibility |
|-------|---------------|
| `InteractableElementAgent` | Parse HTML, extract interactable elements |
| `NavigationAgent` | Plan navigation steps (click, type, select) |
| `DataExtractionAgent` | Extract structured data from pages |
| `PasswordAgent` | Handle password forms, credential managers |
| `2FAAgent` | Handle TOTP, QR code, SMS 2FA flows |
| `DynamicAutoCompleteAgent` | Fill dynamic autocomplete fields (addresses, universities) |
| `Planner Agent` (v2.0) | Decompose complex tasks into steps |
| `Validator Agent` (v2.0) | Verify each step succeeded before proceeding |

### 2.2 Scraper (`skyvern/webeye/scraper/scraper.py`)

The scraper is Skyvern's perception layer — it processes pages into a format the LLM can reason about.

```python
async def scrape_website(
    browser_state: BrowserState,
    url: str,
    element_tree_format: ElementTreeFormat = ElementTreeFormat.HTML,
) -> ScrapedPage:
    # 1. Take screenshot
    # 2. Extract element tree (HTML/accessibility)
    # 3. Identify interactable elements
    # 4. Return ScrapedPage with screenshot + element tree
```

**ElementTreeFormat options:**
- `ElementTreeFormat.HTML` — raw HTML representation
- `ElementTreeFormat.JSON` — structured JSON element tree

**ScrapedPage fields:**
```python
class ScrapedPage:
    url: str
    html: str
    elements: list[dict]      # Parsed interactable elements
    screenshot: bytes         # Base64 screenshot for vision LLM
    element_tree: str         # Formatted element tree
    id_to_css_dict: dict      # Element ID → CSS selector mapping
```

### 2.3 Action System (`skyvern/webeye/actions/`)

Actions are parsed from the LLM's response and executed via Playwright.

**Action parsing:**
```python
# Multiple parsers for different LLM response formats
from skyvern.webeye.actions.parse_actions import (
    parse_actions,              # Standard JSON parsing
    parse_anthropic_actions,    # Anthropic-format tool calls
    parse_cua_actions,          # Computer Use Agent actions
)
```

**Available actions:**
```python
# Navigation
class NavigateAction:
    url: str

# Interaction
class ClickAction:
    element_id: str | None
    prompt: str | None        # AI-powered element finding
    x: int | None
    y: int | None

class InputTextAction:
    element_id: str | None
    prompt: str | None
    text: str

class SelectOptionAction:
    element_id: str
    option: str

# Form-specific
class UploadFileAction:
    file_path: str

class DownloadFileAction:
    file_name: str | None

# Task control
class CompleteAction:
    data_extraction_goal: str | None
    errors: list[str]

class TerminateAction:
    errors: list[str]

class WaitAction:
    wait_sec: int
```

**ActionResult:**
```python
class ActionResult:
    success: bool
    exception_type: str | None
    exception_message: str | None

class ActionSuccess(ActionResult):
    data: dict | str | list | None
    download_triggered: bool | None
    interacted_with_sibling: bool | None
```

### 2.4 Playwright SDK Integration

Skyvern 2.0 provides a Playwright-compatible SDK:

```python
from skyvern import Skyvern

# Local mode
skyvern = Skyvern.local()

# Or Skyvern Cloud
skyvern = Skyvern(api_key="your-api-key")

# Mix standard Playwright with AI
browser = await skyvern.launch_cloud_browser()
page = await browser.get_working_page()

await page.goto("https://example.com")
await page.click("#login-button")              # Standard Playwright
await page.click(prompt="Click the login button")  # AI-powered
await page.agent.login(credential_type="skyvern")  # AI login agent
await page.agent.run_task("Complete checkout")     # Full AI task
```

**AI-augmented Playwright methods:**
```python
# act — natural language action
await page.act("Click the login button and wait for dashboard")

# extract — structured data extraction
result = await page.extract("Get the product name and price")
result = await page.extract(
    prompt="Extract order details",
    schema={"order_id": "string", "total": "number"}
)

# validate — check page state
is_logged_in = await page.validate("Check if user is logged in")

# prompt — arbitrary LLM interaction
summary = await page.prompt("Summarise what's on this page")
```

### 2.5 Workflow Engine (`skyvern/forge/sdk/workflow/`)

Workflows chain multiple tasks into a pipeline:

```python
# Workflow block types
class TaskBlock:
    url: str
    prompt: str
    data_extraction_schema: dict | None
    error_codes: list[str] | None

class ConditionalBlock:
    condition: str
    true_branch: list[WorkflowBlock]
    false_branch: list[WorkflowBlock]

class LoopBlock:
    items: str            # Expression to iterate over
    loop_variable: str
    loop_blocks: list[WorkflowBlock]

class CodeBlock:
    code: str            # Python code to execute

class FileParserBlock:
    file_url: str
    file_type: str       # CSV, PDF, etc.

class SendEmailBlock:
    recipients: list[str]
    subject: str
    body: str
```

### 2.6 LLM Integration (`skyvern/forge/sdk/api/llm/`)

```python
class LLMCaller:
    # Abstracts over multiple LLM providers
    # Handles retries, cost tracking, response parsing
    
    async def call(
        self,
        prompt: str,
        screenshot: bytes | None = None,
        model: str | None = None,
    ) -> LLMResponse: ...
```

**Supported LLM providers (from settings):**
```
GEMINI_3.0_FLASH, GEMINI_2.5_PRO, GEMINI_2.5_FLASH
ANTHROPIC_CLAUDE4.5_OPUS, ANTHROPIC_CLAUDE4.5_SONNET
ANTHROPIC_CLAUDE4_OPUS, ANTHROPIC_CLAUDE4_SONNET
OPENAI_GPT4O, OPENAI_GPT4O_MINI
AWS Bedrock models
Azure OpenAI models
Ollama (local, with OLLAMA_SUPPORTS_VISION=true)
```

### 2.7 BrowserState (`skyvern/webeye/browser_factory.py`)

```python
class BrowserState:
    browser: Browser                    # Playwright browser
    browser_context: BrowserContext     # Playwright context
    page: Page                          # Active page
    # Session persistence
    # Anti-bot configuration
    # Cookie/storage management
```

---

## 3. Agent Execution Flow

### Task Execution

```
User submits Task (url + prompt + optional schema)
    ↓
ForgeAgent receives task
    ↓
┌──────────────── STEP LOOP ────────────────┐
│                                            │
│  1. scrape_website(url)                    │
│     ├─ Take screenshot                     │
│     ├─ Extract element tree                │
│     └─ Build ScrapedPage                   │
│                                            │
│  2. LLMCaller.call(prompt + screenshot)    │
│     └─ Vision LLM analyses page visually  │
│                                            │
│  3. parse_actions(llm_response)            │
│     └─ Extract list of Action objects      │
│                                            │
│  4. Execute each action via Playwright     │
│     ├─ click / input_text / select_option  │
│     ├─ navigate / upload_file              │
│     └─ complete / terminate                │
│                                            │
│  5. Record ActionResult                    │
│  6. if CompleteAction or TerminateAction   │
│     → break                                │
│                                            │
└────────────────────────────────────────────┘
    ↓
Return task result + extracted data
```

### Explore → Replay Pattern (Skyvern 2.0)

```
First run:
  LLM inference on every step → discover workflow
  Generate + cache Playwright script
  Store intent metadata for self-healing
    ↓
Subsequent runs:
  Execute cached Playwright code (deterministic, no LLM)
  If page changes → self-heal using intent metadata
  LLM invoked only when cache misses
```

### Data Flow

```
URL + Prompt
    ↓
BrowserState → navigate to URL
    ↓
scraper.scrape_website()
    → screenshot (bytes) + element_tree (HTML/JSON)
    ↓
LLMCaller(prompt + screenshot + element_tree)
    → "Click element with id=btn_submit"
    → "Input 'John Smith' into name field"
    ↓
parse_actions() → [ClickAction(element_id="btn_submit"), ...]
    ↓
Execute via Playwright → BrowserState.page.click(...)
    ↓
Next iteration
```

---

## 4. Configuration System

### Environment Variables (`.env`)

```bash
# LLM
LLM_KEY=GEMINI_2.5_PRO
GEMINI_API_KEY=your-key
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...

# Browser
BROWSER_TYPE=chromium-headful
BROWSER_LAUNCH_TIMEOUT_SECS=300

# Database
DATABASE_STRING=postgresql+psycopg://skyvern:skyvern@localhost/skyvern

# Anti-bot (cloud only)
SKYVERN_TELEMETRY=false
```

### Task Schema

```python
{
    "url": "https://example.com",
    "prompt": "Log in with username john@example.com and password secret123",
    "data_extraction_schema": {
        "confirmation_number": "string",
        "status": "string"
    },
    "error_codes": [
        "CAPTCHA_NOT_SOLVED",
        "LOGIN_FAILED"
    ],
    "webhook_callback_url": "https://myapp.com/webhook"
}
```

### SDK Setup

```python
# Minimal local setup
from skyvern import Skyvern

skyvern = Skyvern.local()
browser = await skyvern.launch_local_browser()
page = await browser.get_working_page()
await page.goto("https://example.com")
result = await page.agent.run_task("Fill out the contact form")
```

---

## 5. MCP Integration

```python
# MCP server configuration
{
  "mcpServers": {
    "Skyvern": {
      "env": {
        "SKYVERN_BASE_URL": "https://api.skyvern.com",
        "SKYVERN_API_KEY": "YOUR_API_KEY"
      },
      "command": "PATH_TO_PYTHON",
      "args": ["-m", "skyvern", "run", "mcp"]
    }
  }
}
```

```bash
# Run Skyvern MCP server locally
python -m skyvern run mcp
```

---

## 6. Testing & Evaluation

```bash
# Quickstart
pip install skyvern && skyvern quickstart

# With existing database
skyvern quickstart --database-string "postgresql+psycopg://user:pw@localhost/skyvern"

# Interactive UI
navigate to http://localhost:8080
```

Skyvern evaluation recordings at `eval.skyvern.com` — every task run includes full action + decision recordings.

---

## 7. Architecture Diagrams

### Component Hierarchy

```
Skyvern
  ├─ ForgeAgent (orchestrator)
  │   ├─ InteractableElementAgent
  │   ├─ NavigationAgent
  │   ├─ DataExtractionAgent
  │   ├─ PasswordAgent
  │   ├─ 2FAAgent
  │   ├─ DynamicAutoCompleteAgent
  │   └─ (Skyvern 2.0) PlannerAgent + ValidatorAgent
  ├─ Scraper (webeye/scraper/scraper.py)
  │   ├─ Screenshot capture
  │   └─ ElementTree extraction
  ├─ LLMCaller (forge/sdk/api/llm/)
  │   └─ Gemini / Anthropic / OpenAI / Ollama
  ├─ BrowserState (webeye/browser_factory.py)
  │   └─ Playwright Browser + Context + Page
  ├─ WorkflowEngine (forge/sdk/workflow/)
  │   ├─ TaskBlock
  │   ├─ ConditionalBlock
  │   ├─ LoopBlock
  │   └─ CodeBlock
  ├─ FastAPI App (forge/api_app.py)
  └─ PostgreSQL + Redis (persistence + queuing)
```

### Data Flow

```
Task(url, prompt)
    ↓
BrowserState.navigate(url)
    ↓
Scraper.scrape_website()
    → screenshot + element_tree
    ↓
LLMCaller(prompt + screenshot + element_tree)
    → parsed action list
    ↓
ActionExecutor → Playwright calls
    ↓
ActionResult → record → next step
    ↓
CompleteAction → return extracted_data
```

---

## 8. Key Technical Decisions

**Why vision instead of DOM selectors?**
- Selectors break when layouts change — Skyvern's target use case (invoice portals, government sites) changes frequently
- Vision understands semantic meaning — "the blue Submit button" works regardless of HTML structure
- Works on sites with unhelpful DOM (canvas-heavy, heavy JS rendering)

**Why Playwright as execution layer?**
- Mature, well-maintained browser automation
- Cross-browser support (Chromium, Firefox, WebKit)
- Network interception for download handling
- The Playwright-compatible SDK makes migration from existing scripts easy

**Why PostgreSQL + Redis?**
- Task persistence and audit trail
- Workflow state management across steps
- Redis task queue for parallel execution

**Why a swarm of agents instead of one?**
- Each agent is specialised and better at its specific domain
- Password agent can use credential managers securely
- 2FA agent handles complex authentication flows
- Separation of concerns improves reliability

**Explore → Replay rationale:**
- Pure LLM inference on every step is expensive and slow (~$0.10/step with vision models)
- Once the path is known, Playwright scripts are deterministic and free
- Self-healing via intent metadata handles site changes without full re-exploration

---

## 9. Limitations & Known Issues

- **PostgreSQL + Redis required** — much heavier setup than browser-use's `pip install`
- **Vision model cost** — every unexplored step requires a screenshot + vision LLM call (~2-3 seconds, ~$0.05-0.15/step)
- **AGPL-3.0 license** — commercial use of the open-source code requires open-sourcing modifications
- **Anti-bot measures cloud-only** — stealth, proxy rotation, CAPTCHA solving require Skyvern Cloud
- **No Python library install** — must run as a service (Docker or local server), not importable as a Python module

---

## 10. Conclusion

Skyvern represents the most complete vision-based web automation platform in this survey. Its multi-agent architecture, explore→replay pattern, Playwright-compatible SDK, and workflow builder make it suitable for enterprise RPA use cases. It excels specifically at WRITE tasks (forms, logins, downloads) where the DOM is unhelpful or constantly changing.

**WebVoyager score:** 85.85% (Skyvern 2.0, Steel leaderboard)  
**License:** AGPL-3.0  
**Language:** Python 3.11+  
**Repository:** https://github.com/Skyvern-AI/skyvern  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub source analysis, forge/agent.py, webeye/scraper/, forge/sdk/workflow/, README.md, and official documentation.*
