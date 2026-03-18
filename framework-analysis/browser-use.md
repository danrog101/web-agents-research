# Browser-Use

**Framework name:** Browser-Use  
**Repository:** https://github.com/browser-use/browser-use  
**Snapshot date:** 2026-03-18  

---

## Architecture

Browser-Use is an async Python (≥3.11) library implementing a **DOM-centric, event-driven AI browser automation** framework. The core architecture consists of:

- **Agent (`agent/service.py`)** — main orchestrator; runs a step loop (up to `max_steps`), calls the LLM, parses `AgentOutput`, dispatches actions, records history.
- **BrowserSession (`browser/session.py`)** — wraps a Chromium browser via the `cdp-use` library (Chrome DevTools Protocol). Manages browser lifecycle, event routing to watchdogs, and exposes both high-level and raw CDP interfaces.
- **EventBus (bubus)** — decouples components; watchdog services register handlers by method name (`on_EventTypeName`).
- **Watchdogs (`browser/watchdogs/`)** — 12 modular services: Downloads, Popups, Security, DOM, CAPTCHA, Crash, Screenshot, StorageState, Permissions, HARRecording, DefaultAction, AboutBlank.
- **DomService (`dom/service.py`)** — extracts the DOM tree, assigns integer indices to clickable elements, highlights them with coloured bounding boxes, and serialises to a compact text format for LLM consumption.
- **Tools / Registry (`tools/`)** — actions (click, type, navigate, scroll, extract, done, …) registered via decorator `@registry.action(...)`.
- **MessageManager** — builds the LLM context window (system prompt + browser state + action history + screenshots).
- **LLM abstraction (`llm/`)** — `BaseChatModel` protocol implemented for 15+ providers.
- **MCP integration (`mcp/`)** — Browser-Use can run as an MCP server *or* consume external MCP tool servers.

Step execution: CAPTCHA check → prepare context → build messages → LLM invocation → parse actions → execute actions → record history → repeat.

---

## Perception type

**DOM-first (indexed element text) + optional screenshots.** The DOM is serialised to a text representation where every interactive element is annotated with a numeric index (e.g., `[12] <button>Add to Cart</button>`). The LLM references elements by index rather than pixel coordinates. Vision (`use_vision=True`) attaches screenshots as additional context but is not required.

---

## LLM compatibility

15+ providers via a unified `BaseChatModel` protocol:

| Provider | Key models |
|---|---|
| OpenAI | GPT-4o, GPT-4o-mini, O1, O3, GPT-5 |
| Anthropic | Claude 3.5, Claude Sonnet 4 |
| Google | Gemini 2.0, 2.5 |
| Azure OpenAI | Azure-hosted GPT models |
| AWS Bedrock | Bedrock / Anthropic Bedrock |
| Groq | Fast inference |
| DeepSeek | DeepSeek models |
| Mistral | Mistral models |
| Ollama | Local models |
| OpenRouter | Multi-provider gateway |
| Cerebras | Fast inference |
| Vercel | Vercel AI SDK |
| Browser-Use Cloud | Hosted `bu-*` models optimised for automation |

Minimum requirements: function calling, structured output, ~32k+ context. Recommended: GPT-4o, Claude Sonnet 4, Gemini 2.5 Pro with 100k+ context for complex tasks.

---

## Action interface

Registered Python functions returning `ActionResult`. Categories:
- **Navigation:** `navigate`, `go_back`, `go_forward`, `refresh`, `search`
- **Element interaction:** `click`, `click_element_with_text`, `input_text`, `send_keys`
- **Dropdowns:** `get_dropdown_options`, `select_dropdown_option`
- **Tabs:** `switch_to_tab`, `close_tab`
- **Scroll:** `scroll`, `scroll_to_text`
- **Content:** `extract_content`, `screenshot`
- **Task:** `done`, `wait`

Custom actions can be added via `@tools.action(...)` decorator.

---

## Dependencies

- Python ≥3.11
- `cdp-use` (CDP client, maintained by browser-use team)
- `bubus` (event bus)
- `playwright` (underlying browser engine)
- `pydantic` v2 (validation, schema generation)
- `litellm` or provider-specific SDKs
- `uv` (recommended package manager)

---

## Browser control method

**CDP (Chrome DevTools Protocol) via `cdp-use`.** Browser-Use connects directly to Chromium at the protocol level, bypassing Playwright's test-layer abstractions. This gives lower-level control over DOM events, network, and runtime. A Playwright layer was used in earlier versions but was deprecated in favour of direct CDP in v2/v3 cycles. Supports cloud browser profiles and local Chrome.

---

## Strengths

- **Deterministic element selection** via integer indices — no fragile XPath or CSS selectors.
- **Modular watchdog architecture** makes it easy to add custom browser-state monitoring without touching core logic.
- **Richest LLM provider support** of any open-source web agent framework (15+ providers).
- **MCP dual-mode** (server + client) enables seamless ecosystem integration.
- **Prompt caching** support for Anthropic models reduces cost on long sessions.
- **89.1% WebVoyager success rate** reported over 586 tasks (browser-use team's own benchmark run).
- **Active development** — one of the most starred web-agent repos on GitHub (~50k+).
- **Custom action extensibility** — register arbitrary Python functions as agent skills.

---

## Weaknesses

- **CDP-based detection** — CDP WebSocket connections can be flagged by anti-bot systems (compared to extension-based or stealth approaches).
- **Chromium-only** — no Firefox or Safari support (CDP protocol limitation).
- **Shadow DOM / complex iframes** require explicit configuration; may fail on some modern SPAs.
- **High LLM cost per step** — each step involves a full DOM serialisation + LLM call.
- **Python ≥3.11 dependency** may block adoption in older environments.
- **Event bus complexity** — debugging inter-watchdog interactions can be non-trivial.

---

## Typical use cases

- Automated web research and data extraction across arbitrary sites
- RPA-style workflow automation (invoice download, form fill, portal login)
- AI-powered browser testing pipelines
- Multi-agent orchestration where one agent controls a browser
- Building production browser agents with custom skills
- MCP-powered browser tool servers for LLM tool use

---

## Additional notes for thesis research

**Key architectural differentiator:** The event-driven watchdog system (bubus) makes Browser-Use arguably the most *composable* framework in this comparison — each concern (CAPTCHA, security, downloads) is a self-contained, testable unit. This is worth analysing from a software architecture perspective.

**Benchmark context:** The 89.1% WebVoyager figure should be taken carefully — it was measured by the browser-use team on their own run; independent replication may yield different results as WebVoyager uses live sites.

**Research template context:** Your mentor's Browser-Use Research Report (from the gist) was generated by 7 parallel specialised agents using Claude Code, serving as an excellent example of multi-agent research methodology itself.

**For reproducibility:** Record snapshot date, pin specific commit hash. Run `uv sync` for reproducible dependency installation.
