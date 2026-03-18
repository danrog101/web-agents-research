# Stagehand Research Report

## Executive Summary

Stagehand is a browser automation framework built and maintained by Browserbase. Its core philosophy is **bridging the gap between fully manual Playwright code and fully autonomous unpredictable agents** — letting developers choose, step by step, how much AI involvement each action needs. Version 3 (2025) removed the Playwright dependency and rewrote the browser layer to communicate directly via CDP, making it 44% faster on average and enabling Puppeteer/Bun compatibility.

---

## 1. Architecture Overview

### Core Philosophy

Three guiding principles:

**Choose when to use AI vs. code:** Use `act()` with natural language when navigating unfamiliar pages; use standard CDP page calls when you know exactly what to do. Unlike browser-use (where LLM drives everything), Stagehand lets the developer decide per-step.

**Self-healing and caching:** Stagehand automatically caches discovered elements and actions. Once a workflow succeeds, subsequent runs replay from cache without any LLM inference. When the site changes and the cache breaks, AI re-engages automatically.

**Write once, run forever:** The auto-caching + self-healing combination means automations don't need maintenance when sites change — the agent handles recovery.

### Package Structure

```
stagehand/
├── packages/
│   └── core/                     # Main @browserbasehq/stagehand package
│       ├── src/
│       │   ├── Stagehand.ts      # Main class — entry point
│       │   ├── StagehandPage.ts  # V3 Page wrapper with act/extract/observe
│       │   ├── StagehandContext.ts  # Browser context management
│       │   ├── handlers/
│       │   │   ├── actHandler.ts     # act() implementation
│       │   │   ├── extractHandler.ts # extract() implementation
│       │   │   └── observeHandler.ts # observe() implementation
│       │   ├── agent/
│       │   │   ├── AgentClient.ts    # agent() — computer use model integration
│       │   │   └── BrowserAgentTool.ts
│       │   ├── cache/
│       │   │   ├── ActionCache.ts    # Action caching system
│       │   │   └── CacheManager.ts
│       │   ├── context/
│       │   │   └── ContextBuilder.ts # Token-optimised context assembly
│       │   ├── drivers/
│       │   │   ├── CDPDriver.ts      # Native CDP (v3 default)
│       │   │   └── PlaywrightDriver.ts # Optional Playwright adapter
│       │   └── llm/
│       │       ├── LLMClient.ts      # LLM provider abstraction
│       │       └── providers/        # OpenAI, Anthropic, Google, etc.
├── examples/
│   ├── example.ts                # Blank template
│   └── 2048.ts                   # Classic example
├── evals/                        # Evaluation suite
│   └── tasks/                    # Individual eval tasks
└── claude.md                     # Claude Code integration docs
```

---

## 2. Core Components Deep Dive

### 2.1 Stagehand Class (`packages/core/src/Stagehand.ts`)

The main entry point. In v3 it no longer uses Playwright as a dependency — it communicates directly with Chromium via CDP.

```typescript
import { Stagehand } from "@browserbasehq/stagehand";

const stagehand = new Stagehand({
  env: "LOCAL",           // or "BROWSERBASE"
  verbose: 1,             // 0 (silent), 1 (default), 2 (debug)
  model: "openai/gpt-4.1-mini",  // or any supported model
  localBrowserLaunchOptions: {
    cdpUrl: "ws://localhost:9222",  // Connect to existing Chrome
    headless: false,
  },
  // Browserbase options
  browserbaseApiKey: process.env.BROWSERBASE_API_KEY,
  browserbaseProjectId: process.env.BROWSERBASE_PROJECT_ID,
});

await stagehand.init();

// Access CDP context and pages
const page = stagehand.context.pages()[0];
const context = stagehand.context;
const page2 = await stagehand.context.newPage();
```

**Key methods on `Stagehand`:**

```typescript
// AI methods (act on current active page by default)
await stagehand.act("click the sign in button");
await stagehand.act("click the sign in button", { page: page2 }); // specific page

const { author, title } = await stagehand.extract(
    "extract the PR author and title",
    z.object({
        author: z.string(),
        title: z.string(),
    })
);

const [action] = await stagehand.observe("find the sign in button");
await stagehand.act(action);  // execute observed action

// Multi-step agent
const agent = stagehand.agent({ provider: "openai", model: "computer-use-preview" });
await agent.execute("Navigate to the latest PR and extract its title");

// Cleanup
await stagehand.close();
```

### 2.2 Three Primitive Methods

These are the core atomic operations:

**`act(instruction, options?)` — Execute a single browser action**

```typescript
// Natural language instruction → CDP action
await stagehand.act("click the sign in button");
await stagehand.act("type 'hello@example.com' into the email field");
await stagehand.act("scroll down to the footer");

// IMPORTANT: instructions must be ATOMIC (one action)
// ✅ "Click the sign in button"
// ✅ "Type 'hello' into the search input"
// ❌ "Order me pizza" (multi-step)
// ❌ "Type in the search bar and hit enter" (multi-step)
```

**`extract(instruction, schema?)` — Extract structured data**

```typescript
// With Zod schema (TypeScript)
const { author, title } = await stagehand.extract(
    "extract the author and title of the PR",
    z.object({
        author: z.string().describe("The username of the PR author"),
        title: z.string().describe("The title of the PR"),
    })
);

// Without schema — returns raw string
const summary = await stagehand.extract("summarise the page content");

// Python SDK
result = await client.sessions.extract(
    session_id,
    instruction="Get the product name and price",
    schema={"name": "string", "price": "number"}
)
```

**`observe(instruction, options?)` — Get suggested actions without executing**

```typescript
// Returns list of possible actions for grounding planning prompts
const actions = await stagehand.observe("select blue as the favourite color");
// → [{ description: "click the Blue radio button", selector: "#color-blue" }]

// Cache the result for deterministic replay
await stagehand.act(actions[0]);
```

### 2.3 Agent Mode (`packages/core/src/agent/`)

For multi-step autonomous tasks using computer-use models:

```typescript
// OpenAI computer-use-preview
const agent = stagehand.agent({
    provider: "openai",
    model: "computer-use-preview",
});
await agent.execute("Get to the latest PR on the stagehand repo");

// Anthropic computer-use (claude-4-5-opus, etc.)
const agent = stagehand.agent({
    provider: "anthropic",
    model: "claude-opus-4-5-20251101",
});
await agent.execute("Find all open issues and summarise them");
```

The agent runs in a loop:
1. Take screenshot
2. Send to computer-use model
3. Model returns action (click at x,y, type text, etc.)
4. Execute via CDP
5. Repeat until complete

### 2.4 Caching System (`packages/core/src/cache/`)

The caching system is one of Stagehand's defining features:

```
First run:
  act("click the sign in button")
    → LLM identifies element → executes → CACHE the selector + result
    
Subsequent runs:
  act("click the sign in button")
    → Cache hit → execute directly, NO LLM call
    
When cache breaks (site changed):
  → Self-healing: LLM re-runs to find element → updates cache
```

This is controlled via `ActionCache`:
- Caches: element selectors, XPaths, action results
- Invalidation: automatic when element not found on page
- Storage: local filesystem (configurable)

### 2.5 Context Builder (`packages/core/src/context/ContextBuilder.ts`)

In v3, a new context builder was introduced to reduce token waste:

```
Old approach: Send full DOM/accessibility tree to LLM
    → High token count, slow, expensive

New approach (v3): ContextBuilder selects only relevant elements
    → Based on instruction semantic similarity
    → Feeds model only what's essential
    → Significant token cost reduction
```

### 2.6 CDP Driver (`packages/core/src/drivers/CDPDriver.ts`)

v3 removed Playwright as a dependency and replaced it with a native CDP driver:

```typescript
// Direct CDP communication
// No Playwright test-layer overhead
// Works with any CDP-compatible browser: Chrome, Chromium, Puppeteer

// Connection modes:
// 1. Local (CDP URL)
const stagehand = new Stagehand({
    env: "LOCAL",
    localBrowserLaunchOptions: { cdpUrl: "ws://localhost:9222" }
});

// 2. Browserbase (cloud)
const stagehand = new Stagehand({
    env: "BROWSERBASE",
    browserbaseApiKey: "...",
    browserbaseProjectId: "..."
});
```

### 2.7 LLM Integration (`packages/core/src/llm/`)

Multi-provider support via unified `LLMClient`:

```typescript
// Supported providers (from releases and docs)
"openai/gpt-4o"
"openai/gpt-4.1-mini"
"openai/computer-use-preview"    // for agent() mode
"anthropic/claude-sonnet-4-6"
"anthropic/claude-opus-4-5-20251101"
"google/gemini-2.0-flash"        // default for MCP server
"google/gemini-2.5-pro"
"google/vertex/gemini-3-pro"
```

### 2.8 Python SDK (`stagehand-python`)

The Python SDK wraps the Stagehand API server:

```python
from stagehand import Stagehand, AsyncStagehand

# Synchronous
with Stagehand(
    server="remote",
    browserbase_api_key="...",
    browserbase_project_id="...",
    model_api_key="...",
) as client:
    session = client.sessions.start(
        model_name="anthropic/claude-sonnet-4-6",
        browser={"type": "browserbase"},
    )
    cdp_url = session.data.cdp_url
    # Connect Playwright to same session via CDP
    
    observe_stream = client.sessions.observe(session.id, instruction="...")
    act_stream = client.sessions.act(session.id, input="...")
    extract_stream = client.sessions.extract(session.id, instruction="...", schema={...})

# Async
client = AsyncStagehand(
    browserbase_api_key="...",
    browserbase_project_id="...",
    model_api_key="...",
)
```

---

## 3. Execution Flow

### act() Flow

```
stagehand.act("click the sign in button")
    ↓
Check ActionCache → HIT? Execute cached selector, done.
    ↓ (cache miss)
ContextBuilder.build(instruction, current_page)
    → Accessibility tree (scoped, relevant elements only)
    ↓
LLMClient.call(instruction + context)
    → { action: "click", selector: "#sign-in-btn", xpath: "//button[@id='sign-in-btn']" }
    ↓
CDPDriver.execute(action)
    → Chrome DevTools Protocol: Runtime.callFunctionOn(...)
    ↓
Cache result for future runs
    ↓
Return result
```

### observe() → act() Pattern

```typescript
// Step 1: Observe — get grounded candidates (no execution)
const actions = await stagehand.observe("select blue as the favourite color");
// → [{ description: "click Blue radio button", selector: "#blue" }]

// Step 2: Use for planning (feed to your own agent loop)
// Step 3: Execute deterministically
await stagehand.act(actions[0]);
```

### Data Flow Diagram

```
User instruction (string)
    ↓
Stagehand.act() / extract() / observe()
    ↓
Check Cache → HIT → Execute CDP → Done
    ↓ (miss)
ContextBuilder
    → Accessibility snapshot (CDP)
    → Semantic filtering (relevant elements only)
    ↓
LLMClient
    → Provider API (OpenAI / Anthropic / Google)
    → Model response (action + reasoning)
    ↓
CDPDriver
    → Chrome DevTools Protocol
    → Browser executes action
    ↓
Update cache
    ↓
Return ActionResult / ExtractResult
```

---

## 4. Configuration

### TypeScript Setup

```typescript
// .env
OPENAI_API_KEY=sk-...
// or
ANTHROPIC_API_KEY=sk-ant-...
BROWSERBASE_API_KEY=...          // optional, for cloud
BROWSERBASE_PROJECT_ID=...       // optional, for cloud

// package.json
{
  "dependencies": {
    "@browserbasehq/stagehand": "latest",
    "zod": "^3.0.0"
  }
}
```

```bash
npm install @browserbasehq/stagehand zod
npx playwright install  # only needed for local mode fallback
```

### Python Setup

```bash
pip install stagehand
# Python 3.9+
```

### MCP Server

```json
{
  "mcpServers": {
    "browserbase": {
      "command": "npx",
      "args": ["@browserbasehq/mcp-server-browserbase"],
      "env": {
        "BROWSERBASE_API_KEY": "...",
        "BROWSERBASE_PROJECT_ID": "...",
        "GEMINI_API_KEY": "..."
      }
    }
  }
}
```

---

## 5. Architecture Diagrams

### Component Hierarchy

```
Stagehand (main class)
  ├─ StagehandContext (CDP context)
  │   └─ StagehandPage (CDP pages) — multiple
  ├─ act() → actHandler
  │   ├─ ActionCache (check/store)
  │   ├─ ContextBuilder (accessibility snapshot)
  │   └─ LLMClient → CDPDriver
  ├─ extract() → extractHandler
  │   ├─ ContextBuilder
  │   └─ LLMClient (Zod schema validation)
  ├─ observe() → observeHandler
  │   └─ Returns ObserveResult[] (no execution)
  ├─ agent() → AgentClient
  │   ├─ Computer-use model (OpenAI CUA / Anthropic)
  │   └─ Screenshot → action → CDP loop
  └─ CDPDriver (v3 — no Playwright dependency)
      └─ Chrome DevTools Protocol
          └─ Chromium / Puppeteer / Browserbase
```

### Stagehand v2 vs v3

```
v2:
  Stagehand → Playwright → CDP → Chrome
  (Playwright added test-layer overhead, auto-waiting)

v3:
  Stagehand → CDPDriver → CDP → Chrome
  (Direct CDP: 44% faster, Puppeteer-compatible, Bun support)
```

---

## 6. Key Technical Decisions

**Why remove Playwright in v3?**
- Playwright is designed for testing, not automation — its actionability checks and auto-waiting add overhead
- Direct CDP gives lower latency, especially for iframes and shadow DOM
- Playwright-first design prevented Puppeteer compatibility and Bun support
- Building at the protocol level = full portability across environments

**Why cache-first architecture?**
- LLM calls are expensive (~$0.002-0.01/call) and slow (500ms-2s)
- Most automation runs the same workflow repeatedly — cache hit rate is high
- Self-healing only when needed = best of both worlds

**Why context builder (v3)?**
- Full DOM/accessibility trees waste tokens on irrelevant elements
- "Find the sign-in button" doesn't need the entire page DOM
- Semantic filtering reduces token usage 60-80% per call

**Why multi-language SDKs?**
- Stagehand API is a hosted service (Browserbase server)
- SDKs in Python, Go, Rust, Java, Kotlin, C#, PHP, Ruby call the same REST API
- Language-specific code generation from OpenAPI spec

**Why `observe()` as a separate primitive?**
- Separates perception from execution
- Enables human-in-the-loop workflows (review before executing)
- Provides deterministic action candidates for caching

---

## 7. Limitations & Known Issues

- **Browserbase dependency for full features** — local mode works but anti-detection, stealth, parallel sessions require Browserbase Cloud (paid)
- **`connectOverCDP` complexity** — passing external Playwright pages to Stagehand v3 requires careful handling (`StagehandInitError: Failed to resolve V3 Page`)
- **TypeScript primary** — Python/other SDKs wrap the API server, adding network latency vs. TypeScript SDK
- **No workflow/task chaining built-in** — Stagehand is a primitive layer; complex orchestration requires developer code (or `agent()` mode)
- **~500ms per act()** — even with caching, LLM cold paths are slower than pure Playwright

---

## 8. Conclusion

Stagehand represents the most developer-ergonomic approach to AI browser automation. Its "code where you can, AI where you must" philosophy and self-healing cache make it uniquely suitable for production workflows. Version 3's CDP-native architecture closes the performance gap with raw Playwright while maintaining natural language capabilities.

**License:** MIT  
**Language:** TypeScript (primary), Python, Go, Rust, Java, Kotlin, C#, PHP, Ruby  
**Repository:** https://github.com/browserbase/stagehand  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub source analysis, claude.md, README.md, stagehand-python README, mcp-server-browserbase README, and issue tracker.*
