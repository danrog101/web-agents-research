# Browser-Use Research Report

## Executive Summary

Browser-Use is an async Python (≥3.11) library that implements AI browser automation using LLMs + CDP (Chrome DevTools Protocol). It enables AI agents to autonomously navigate web pages, interact with elements, and complete complex tasks through an event-driven architecture. The library follows a **DOM-centric approach** where LLMs reason about elements by their integer index rather than pixel coordinates — making it fundamentally different from vision-first approaches like Magnitude or Skyvern.

---

## 1. Architecture Overview

### Core Philosophy

Browser-Use solves browser automation through three interlocking design principles:

**DOM-First Architecture:** Instead of screenshots and pixel coordinates, browser-use processes the DOM tree, assigns sequential indices to every clickable/interactive element, and has the LLM select actions by referencing these indices (e.g., `CLICK [7]`). This provides deterministic element selection independent of visual layout.

**Event-Driven Browser Management:** Uses the `bubus` event bus to coordinate multiple watchdog services that handle different aspects of browser state — downloads, popups, security, DOM snapshots, CAPTCHA, crashes.

**Async Python Design:** Built from the ground up with `async/await`, requiring Python ≥3.11 for modern typing features (`str | None` instead of `Optional[str]`).

### Package Structure

```
browser_use/
├── agent/
│   ├── service.py           # Main Agent class — orchestration loop
│   ├── views.py             # AgentOutput, AgentHistory, AgentSettings, ActionResult
│   ├── message_manager/     # LLM context building
│   └── system_prompt*.md    # System prompts in markdown
├── browser/
│   ├── session.py           # BrowserSession — CDP controller + event bus
│   ├── profile.py           # BrowserProfile — launch args, display, extensions
│   ├── events.py            # All event type definitions (Pydantic models)
│   └── watchdogs/           # 12 modular watchdog services
│       ├── downloads.py
│       ├── popups.py
│       ├── security.py
│       ├── dom.py
│       ├── captcha.py
│       ├── crash.py
│       └── ...
├── dom/
│   ├── service.py           # DomService — DOM extraction, serialisation
│   └── serializer/          # DOM-to-text pipeline
├── llm/
│   ├── base.py              # BaseChatModel protocol
│   ├── openai/chat.py       # ChatOpenAI
│   ├── anthropic/chat.py    # ChatAnthropic
│   ├── google/chat.py       # ChatGoogle
│   └── [provider]/          # 13+ more providers
├── tools/
│   ├── service.py           # Tools class + action registry
│   └── registry/            # @registry.action decorator system
├── mcp/
│   ├── server.py            # Browser-use as MCP server
│   └── client.py            # Connect to external MCP servers
├── filesystem/
│   └── file_system.py       # FileSystemState
└── tokens/
    └── views.py             # UsageSummary — token tracking
```

---

## 2. Core Components Deep Dive

### 2.1 Agent System (`browser_use/agent/service.py`)

The `Agent` class is the main orchestrator for LLM-driven browser automation. It is generic over `Context` and `AgentStructuredOutput`.

**Key Responsibilities:**
- LLM integration and model management
- Browser session lifecycle management
- Action execution and validation
- History tracking and memory management
- Telemetry emission

**Configuration (`AgentSettings` from `views.py`):**

```python
class AgentSettings(BaseModel):
    use_vision: bool | Literal['auto'] = True
    vision_detail_level: Literal['auto', 'low', 'high'] = 'auto'
    save_conversation_path: str | Path | None = None
    max_failures: int = 3
    generate_gif: bool | str = False
    override_system_message: str | None = None
    extend_system_message: str | None = None
    max_actions_per_step: int = 3
    page_extraction_llm: BaseChatModel | None = None
    # sensitive_data masking, allowed_domains, etc.
```

**Constructor signature (simplified):**

```python
class Agent(Generic[Context, AgentStructuredOutput]):
    def __init__(
        self,
        task: str,
        llm: BaseChatModel | None = None,
        browser: BrowserSession | None = None,
        browser_profile: BrowserProfile | None = None,
        tools: Tools[Context] | None = None,
        sensitive_data: dict[str, str | dict[str, str]] | None = None,
        initial_actions: list[dict[str, dict[str, Any]]] | None = None,
        output_model_schema: type[AgentStructuredOutput] | None = None,
        register_new_step_callback: Callable | None = None,
        register_done_callback: Callable | None = None,
    )
```

**Key Methods:**
- `run(max_steps=100)` — Execute the full task loop
- `step()` — Execute one single step
- `save_trajectory(path)` — Persist execution history to JSON
- `rerun(history, task)` — Replay from saved trajectory with variable substitution

**AgentOutput structure:**

```python
class AgentOutput(BaseModel):
    thinking: str | None = None
    evaluation_previous_goal: str | None = None
    memory: str | None = None
    next_goal: str | None = None
    action: list[ActionModel]  # At least one required
```

### 2.2 Action System (`browser_use/tools/service.py`)

Actions are atomic units of behaviour, registered via a decorator pattern:

```python
@registry.action(
    'Click element by index or coordinates',
    param_model=ClickElementAction,
)
async def click(params: ClickElementAction, browser_session: BrowserSession):
    # implementation
    return ActionResult(extracted_content="Clicked element [7]")
```

**Available Action Categories:**

| Category | Actions |
|----------|---------|
| Navigation | `navigate`, `go_back`, `go_forward`, `refresh`, `search` |
| Element interaction | `click`, `click_element_with_text`, `input_text`, `send_keys` |
| Dropdowns | `get_dropdown_options`, `select_dropdown_option` |
| Tabs | `switch_to_tab`, `close_tab`, `open_tab` |
| Scroll | `scroll`, `scroll_to_text` |
| Content | `extract_content`, `screenshot`, `get_page_source` |
| Files | `upload_file`, `write_file`, `read_file` |
| Task | `done`, `wait` |

**ActionResult structure:**

```python
class ActionResult(BaseModel):
    is_done: bool | None = False
    success: bool | None = None
    error: str | None = None
    extracted_content: str | None = None
    long_term_memory: str | None = None
    include_extracted_content_only_once: bool = False
```

**Custom actions** (developer extension point):

```python
from browser_use import Tools

tools = Tools()

@tools.action(description='Send a Slack notification')
async def send_slack(message: str) -> str:
    # your logic here
    return f"Sent: {message}"

agent = Agent(task="...", llm=llm, tools=tools)
```

### 2.3 BrowserSession (`browser_use/browser/session.py`)

BrowserSession is the event-driven browser controller with CDP integration via the `cdp-use` thin wrapper.

**Architecture:** 2-layer design
- **High level:** event bus dispatch to watchdogs
- **Low level:** direct CDP/cdp-use calls

```python
class BrowserSession(BaseModel):
    def __init__(
        self,
        # Cloud browser
        cloud_profile_id: UUID | str | None = None,
        # Local browser
        cdp_url: str | None = None,
        headless: bool = False,
        user_data_dir: str | None = None,
        # Security
        allowed_domains: list[str] | None = None,
        prohibited_domains: list[str] | None = None,
        # Display
        viewport_width: int = 1280,
        viewport_height: int = 720,
    )
```

**Key Methods:**
- `start()` / `stop()` — Lifecycle
- `dispatch_event(event)` — Emit to event bus
- `wait_for_event(event_type, timeout)` — Block until event received
- `get_current_state()` — Full browser state snapshot

**CDP integration via `cdp-use`:**

```python
# Type-safe CDP calls
cdp_client.send.DOMSnapshot.enable(session_id=session_id)
cdp_client.send.Target.attachToTarget(
    params=ActivateTargetParameters(targetId=target_id, flatten=True)
)
# Event registration
cdp_client.register.Browser.downloadWillBegin(callback_func)
```

### 2.4 Event System (`browser_use/browser/events.py`)

All events are Pydantic models for type safety. Categories:

| Category | Examples |
|----------|---------|
| Browser lifecycle | `BrowserStartEvent`, `BrowserStopEvent`, `BrowserLaunchEvent` |
| Navigation | `NavigateToUrlEvent`, `NavigationCompleteEvent`, `GoBackEvent` |
| Element interaction | `ClickElementEvent`, `TypeTextEvent`, `ScrollEvent` |
| Tabs | `SwitchTabEvent`, `TabCreatedEvent`, `TabClosedEvent` |
| Downloads | `DownloadStartedEvent`, `FileDownloadedEvent` |
| Dialogs | `DialogOpenedEvent` |
| CAPTCHA | `CaptchaSolverStartedEvent`, `CaptchaSolverFinishedEvent` |

```python
class ClickElementEvent(ElementSelectedEvent[dict[str, Any] | None]):
    """Click an element."""
    node: 'EnhancedDOMTreeNode'
    button: Literal['left', 'right', 'middle'] = 'left'
    num_clicks: int = 1
```

### 2.5 Watchdog Services (`browser_use/browser/watchdogs/`)

Watchdogs are modular services that subscribe to events and handle specific browser concerns. They register handlers by method naming convention: `on_EventTypeName`.

```python
class BaseWatchdog(BaseModel):
    LISTENS_TO: ClassVar[list[type[BaseEvent[Any]]]] = []
    EMITS: ClassVar[list[type[BaseEvent[Any]]]] = []

class DownloadsWatchdog(BaseWatchdog):
    LISTENS_TO = [DownloadStartedEvent, FileDownloadedEvent]
    
    async def on_DownloadStartedEvent(self, event: DownloadStartedEvent):
        # handle download
        pass
```

| Watchdog | Purpose |
|----------|---------|
| `DownloadsWatchdog` | PDF auto-download, file management |
| `PopupsWatchdog` | JavaScript dialogs (alert, confirm, prompt) |
| `SecurityWatchdog` | Domain restrictions, allowed/prohibited lists |
| `DOMWatchdog` | DOM snapshots, element highlighting |
| `CaptchaWatchdog` | CAPTCHA detection and solving |
| `CrashWatchdog` | Browser crash recovery, automatic reconnect |
| `ScreenshotWatchdog` | Screenshot capture management |
| `StorageStateWatchdog` | Cookie/localStorage persistence |
| `PermissionsWatchdog` | Browser permission handling |
| `HARRecordingWatchdog` | HAR file generation for debugging |
| `AboutBlankWatchdog` | about:blank page handling |
| `DefaultActionWatchdog` | Fallback default action execution |

### 2.6 DOM Service (`browser_use/dom/service.py`)

DomService handles DOM tree extraction, element indexing, and serialisation for LLM consumption.

```python
class DomService:
    def __init__(
        self,
        browser_session: 'BrowserSession',
        cross_origin_iframes: bool = False,
        paint_order_filtering: bool = True,
        max_iframes: int = 100,
        max_iframe_depth: int = 5,
        viewport_threshold: int | None = 1000,
    )
```

**DOM Serialisation Pipeline:**

```
Raw HTML/DOM
    ↓
Accessibility Tree extraction (a11y API)
    ↓
Clickable Element Detection (buttons, links, inputs)
    ↓
Viewport filtering (only visible elements)
    ↓
Paint order filtering (z-index awareness)
    ↓
Integer Index Assignment [1], [2], [3]...
    ↓
Bounding Box Calculation + Highlight (colored borders)
    ↓
Text Serialisation → "[7] <button>Add to Cart</button>"
    ↓
LLM Context Ready
```

**Key features:**
- Accessibility tree > raw HTML (closer to what automation needs)
- Cross-origin iframe support (opt-in)
- Paint-order filtering prevents clicking on hidden elements
- Coloured bounding box highlighting for visual debugging

### 2.7 LLM Integration (`browser_use/llm/`)

Unified `BaseChatModel` protocol all providers implement:

```python
class BaseChatModel(Protocol):
    model: str
    
    @property
    def provider(self) -> str: ...
    
    async def ainvoke(
        self,
        messages: list[BaseMessage],
        output_format: type[T] | None = None
    ) -> ChatInvokeCompletion[T] | ChatInvokeCompletion[str]: ...
```

**Supported Providers (from `__init__.py`):**

```python
from browser_use import (
    ChatOpenAI,      # GPT-4o, o1, o3, GPT-5
    ChatAnthropic,   # Claude 3.5, Claude Sonnet 4
    ChatGoogle,      # Gemini 2.0, 2.5
    ChatGroq,        # Fast inference
    ChatOllama,      # Local models
    ChatAzureOpenAI, # Azure-hosted
    # + DeepSeek, Mistral, Cerebras, Vercel, OpenRouter, Bedrock
    ChatBrowserUse,  # Hosted bu-* models optimised for automation
)
```

---

## 3. Agent Execution Flow

### High-Level Run Flow

```
agent.run(max_steps=100)
    ↓
Signal handlers setup (graceful Ctrl+C)
    ↓
BrowserSession.start() — launch Chromium via CDP
    ↓
┌─────────────── STEP LOOP ───────────────┐
│                                          │
│  1. Wait if CAPTCHA solving              │
│  2. _prepare_context()                   │
│     └─ BrowserSession.get_current_state()│
│        └─ DomService.get_serialized_dom()│
│  3. MessageManager.build_context()       │
│     ├─ System prompt (markdown file)     │
│     ├─ Browser state (DOM + URL + tabs)  │
│     ├─ Action history                    │
│     └─ Optional screenshot               │
│  4. llm.ainvoke(messages, AgentOutput)   │
│  5. parse_actions(response.action)       │
│  6. execute each action                  │
│     └─ dispatch_event(ActionEvent)       │
│        └─ Watchdog handles → CDP call    │
│  7. Record AgentHistory                  │
│  8. if done action → break               │
│                                          │
└──────────────────────────────────────────┘
    ↓
BrowserSession.stop()
    ↓
Return AgentHistoryList
```

### Step Execution Detail

```python
async def step(self, step_info: AgentStepInfo | None = None) -> None:
    # Phase 0: CAPTCHA check
    await self.browser_session.wait_if_captcha_solving()
    
    # Phase 1: Context preparation
    browser_state = await self._prepare_context(step_info)
    
    # Phase 2: Message building
    messages = await self.message_manager.build_context(
        browser_state, action_history
    )
    
    # Phase 3: LLM call
    response = await self.llm.ainvoke(messages, AgentOutput)
    
    # Phase 4: Action parsing
    parsed_actions = self.parse_actions(response.action)
    
    # Phase 5: Execution
    for action in parsed_actions:
        result = await self.execute_action(action)
        self.state.last_result.append(result)
    
    # Phase 6: History recording
    self.history.append(AgentHistory(
        model_output=response,
        result=self.state.last_result,
        state=browser_state,
    ))
```

---

## 4. Configuration System

### 4.1 LLM Configuration

```python
from browser_use import ChatOpenAI, ChatAnthropic, ChatGoogle

llm = ChatOpenAI(model="gpt-4o", api_key="sk-...", temperature=0.2)
llm = ChatAnthropic(model="claude-sonnet-4-20250514", temperature=0.2)
llm = ChatGoogle(model="gemini-2.5-pro", temperature=0.2)
llm = ChatBrowserUse()  # Optimised hosted model, 3-5x faster
```

### 4.2 Browser Configuration

```python
from browser_use.browser.profile import BrowserProfile

profile = BrowserProfile(
    cdp_url="http://localhost:9222",  # Connect to existing Chrome
    headless=False,
    user_data_dir="./browser_profile",
    viewport_width=1280,
    viewport_height=720,
    allowed_domains=["example.com"],
    prohibited_domains=["ads.com"],
    proxy={"server": "http://proxy:8080", "username": "u", "password": "p"},
    extensions=["/path/to/extension"],
    args=['--disable-blink-features=AutomationControlled'],
)
```

### 4.3 Agent Configuration

```python
from browser_use import Agent

agent = Agent(
    task="Go to amazon.com, search for wireless headphones, add first result to cart",
    llm=llm,
    browser_profile=profile,
    max_actions_per_step=3,
    use_vision=True,          # 'auto', True, False
    sensitive_data={           # Values masked in logs as {{key}}
        "username": "user@example.com",
        "password": "secret123"
    },
    register_new_step_callback=my_callback,
    output_model_schema=MyPydanticOutput,  # Structured output
)

result = await agent.run(max_steps=50)
```

---

## 5. MCP Integration

### MCP Server Mode

```bash
# Run browser-use as an MCP server
uvx browser-use[cli] --mcp
```

```json
// Claude Desktop mcp config
{
  "mcpServers": {
    "browser-use": {
      "command": "uvx",
      "args": ["browser-use[cli]", "--mcp"],
      "env": { "OPENAI_API_KEY": "sk-..." }
    }
  }
}
```

### MCP Client Mode

```python
from browser_use.mcp.client import MCPClient

mcp = MCPClient(
    server_name="filesystem",
    command="npx",
    args=["@modelcontextprotocol/server-filesystem"]
)
await mcp.register_to_tools(agent.tools)
# MCP tools now available as browser-use actions
await agent.run()
await mcp.disconnect()
```

---

## 6. Testing Framework

```
tests/
├── ci/                    # CI-run tests
│   ├── conftest.py        # Shared fixtures, mock LLM
│   ├── browser/           # Browser-level tests
│   ├── models/            # LLM provider tests
│   ├── interactions/      # Click/type/scroll tests
│   └── security/          # Domain restriction tests
└── conftest.py            # Root fixtures
```

```bash
uv run pytest -vxs tests/ci    # CI tests
uv run pytest -vxs tests/      # All tests
```

Key testing pattern — HTTP server for local HTML:
```python
def test_navigation(httpserver):
    httpserver.expect_request("/test").respond_with_data(
        "<html><body><button>Click me</button></body></html>"
    )
    # Agent navigates to httpserver.url_for("/test")
```

---

## 7. Architecture Diagrams

### Component Hierarchy

```
Agent
  ├─ BrowserSession
  │   ├─ EventBus (bubus)
  │   │   └─ Watchdogs (12 services)
  │   │       ├─ DownloadsWatchdog
  │   │       ├─ PopupsWatchdog
  │   │       ├─ SecurityWatchdog
  │   │       ├─ DOMWatchdog
  │   │       ├─ CaptchaWatchdog
  │   │       └─ CrashWatchdog ...
  │   └─ CDP Client (cdp-use)
  │       └─ Chrome/Chromium (local or cloud)
  ├─ Tools
  │   └─ Registry
  │       └─ Actions: click, type, navigate, scroll, extract, done...
  ├─ MessageManager
  │   └─ System Prompt + Browser State + History
  ├─ DomService
  │   └─ Accessibility Tree → Index Assignment → Text Serialisation
  ├─ LLM (BaseChatModel)
  │   └─ 15+ providers via unified protocol
  └─ FileSystem
      └─ FileSystemState
```

### Data Flow

```
User Task (string)
    ↓
Agent.run()
    ↓
BrowserSession.get_current_state()
    ↓
DomService.get_serialized_dom()
    → "[1] <a>Home</a>  [2] <input> Search  [7] <button>Add to Cart</button>"
    ↓
MessageManager.build_context()
    → System Prompt + DOM text + URL + screenshot (if vision) + history
    ↓
LLM.ainvoke() → AgentOutput
    → { thinking: "...", next_goal: "click add to cart", action: [CLICK [7]] }
    ↓
execute_action(CLICK [7])
    ↓
dispatch_event(ClickElementEvent(node=[7]))
    ↓
EventBus → DOMWatchdog → CDP call → Chrome executes click
    ↓
Record result → loop
```

### Event Flow

```
Action Execution
    ↓
dispatch_event(ClickElementEvent)
    ↓
EventBus.route()
    ↓
Watchdog.on_ClickElementEvent() → CDP call → Browser
    ↓
emit_event(NavigationCompleteEvent)
    ↓
Other watchdogs react (SecurityWatchdog checks domain, etc.)
```

---

## 8. Key Technical Decisions

**Why CDP instead of Playwright?**
- Low-level control over browser internals
- Direct access to network, DOM, runtime events
- No test-layer overhead (Playwright's actionability checks add latency)
- Better for automation than testing

**Why `cdp-use` wrapper?**
- Type-safe CDP interfaces (auto-generated from Chrome's protocol)
- Managed by browser-use team
- Async-first design

**Why `bubus` event bus?**
- Decoupled component architecture — watchdogs don't know about each other
- Type-safe event handling via Pydantic models
- Easy to add new watchdogs without touching existing code
- Testable in isolation

**Why Pydantic v2?**
- Runtime validation of LLM outputs
- JSON Schema generation for tool definitions
- Modern Python typing (`str | None` vs `Optional[str]`)

**Why `uv` instead of `pip`?**
- 10-100x faster dependency resolution
- Reproducible builds via lockfile
- Manages Python versions too

**Why async/await throughout?**
- Non-blocking CDP WebSocket communication
- Concurrent event handling
- Natural fit for Python 3.11+ patterns

---

## 9. Security & Privacy

```python
# Sensitive data masking — values never appear in logs
agent = Agent(
    task="Login to portal",
    sensitive_data={
        "username": "user@company.com",
        "password": "secret"
    }
)
# Prompt shows: {{username}}, {{password}}
# Logs show: ●●●●●●●●

# Domain restriction
profile = BrowserProfile(
    allowed_domains=["company.com"],
    prohibited_domains=["ads.com", "tracker.com"]
)
# SecurityWatchdog blocks navigation to prohibited domains
```

---

## 10. Limitations & Known Issues

**Current limitations:**
- **Chromium only** — CDP not available in Firefox/Safari
- **Shadow DOM** — complex shadow DOM requires `cross_origin_iframes=True`
- **Cross-origin iframes** — disabled by default, may miss content
- **No parallel browsing** — single browser session per Agent instance

**Model requirements (minimum):**
- Function calling / tool use support
- Structured JSON output
- 32k+ context window

**Recommended models:**
- `ChatBrowserUse()` — purpose-trained, 3-5x faster, SOTA accuracy
- GPT-4o, Claude Sonnet 4, Gemini 2.5 Pro
- Vision capability needed if `use_vision=True`
- 100k+ context for complex multi-step tasks

---

## 11. Performance Optimisations

```python
# Prompt caching (Anthropic)
llm = ChatAnthropic(model="claude-sonnet-4-20250514")
# System prompt and history are automatically cached — reduces cost ~90%

# Vision modes
agent = Agent(task="...", llm=llm, use_vision='auto')
# 'auto' — includes screenshot tool, uses only when needed
# True   — always includes screenshot
# False  — never uses screenshots, no vision model needed

# Flash mode — faster with simpler models
agent = Agent(task="...", llm=llm, use_thinking=False)
# Disables: thinking, evaluation_previous_goal, next_goal fields
```

---

## 12. Development Workflow

```bash
# Setup
uv venv --python 3.11
source .venv/bin/activate
uv sync

# Quality
uv run pyright         # Type checking
uv run ruff check --fix
uv run ruff format
uv run pre-commit run --all-files

# Tests
uv run pytest -vxs tests/ci
```

---

## 13. Conclusion

Browser-Use represents the current state-of-the-art for open-source DOM-based web agent frameworks. Its event-driven watchdog architecture is uniquely extensible, the LLM provider abstraction is the broadest in its class (15+ providers), and the CDP-first design gives lower latency than Playwright-wrapped alternatives.

**Key strengths summary:**
- DOM-first → deterministic element selection, no vision model required
- 12-watchdog event architecture → modular, testable, extensible
- 15+ LLM providers via unified BaseChatModel protocol
- MCP server + client → ecosystem connectivity
- 89.1% WebVoyager (Steel leaderboard, 2025)
- ~50k GitHub stars, actively maintained

**WebVoyager score:** 89.1% (Steel leaderboard, single-run, self-reported)  
**License:** MIT  
**Language:** Python ≥3.11  
**Repository:** https://github.com/browser-use/browser-use  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub source code analysis, CLAUDE.md, AGENTS.md, __init__.py, and official documentation.*
