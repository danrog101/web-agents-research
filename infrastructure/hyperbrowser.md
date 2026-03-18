
# Hyperbrowser Research Report

## Executive Summary

Hyperbrowser is a **cloud browser infrastructure platform backed by Y Combinator**, self-described as "the internet infrastructure for AI." It provides managed browser sessions accessible via CDP WebSocket, similar to Browserbase, but differentiates itself with an **AI-native automation layer** (HyperAgent — a Playwright-based AI agent framework), action caching for deterministic replay, and built-in AI-powered scraping/crawling endpoints. It targets teams who want both browser infrastructure AND an AI agent layer from one vendor.

---

## 1. Architecture Overview

### Core Philosophy

**AI-native from the ground up:** Hyperbrowser is not just a browser host — it builds the AI reasoning layer on top of the browser infrastructure. HyperAgent (their open-source framework) adds `page.ai()`, `page.extract()`, and `executeTask()` on top of standard Playwright pages. This creates a vertically integrated stack: cloud browser + AI agent + caching.

**Sub-second session start:** Sessions start in under 1 second (claimed), with support for 1,000+ concurrent sessions at enterprise tiers.

**Action caching:** HyperAgent can record action traces (XPath + intent metadata) and replay them without LLM calls. On cache miss (page changed), it falls back to LLM. This is architecturally similar to Stagehand's caching system.

### Service Architecture

```
Developer / AI Agent
        ↓
Hyperbrowser SDK (Node.js or Python)
client.sessions.create({ useStealth: true, ... })
        ↓
Hyperbrowser REST API
        ↓
Session Provisioner
  ├─ Managed Chromium instance
  ├─ Fingerprint randomisation
  ├─ Proxy rotation (if configured)
  └─ CAPTCHA solving (built-in)
        ↓
session.wsEndpoint (wss:// CDP WebSocket)
        ↓
Connect: playwright.chromium.connectOverCDP(wsEndpoint)
         puppeteer.connect({ browserWSEndpoint: wsEndpoint })
         pyppeteer.connect(browserWSEndpoint=wsEndpoint)
```

---

## 2. Core Components Deep Dive

### 2.1 Sessions API

```typescript
// TypeScript
import { Hyperbrowser } from "@hyperbrowser/sdk";
const client = new Hyperbrowser({ apiKey: process.env.HYPERBROWSER_API_KEY });

// Basic session
const session = await client.sessions.create();
console.log(session.wsEndpoint);  // wss://...

// Stealth session
const stealthSession = await client.sessions.create({
    useStealth: true,
    operatingSystems: ["macos"],  // "windows", "linux"
    device: ["desktop"],          // "mobile"
    locales: ["en-US"],
    screen: { width: 1920, height: 1080 },
});

// Ultra stealth (Enterprise)
const ultraSession = await client.sessions.create({
    useUltraStealth: true,  // Enterprise only
});

// Stop session
await client.sessions.stop(session.id);
```

```python
# Python
from hyperbrowser import Hyperbrowser, AsyncHyperbrowser
from hyperbrowser.models import CreateSessionParams, ScreenConfig

client = Hyperbrowser(api_key=os.getenv("HYPERBROWSER_API_KEY"))

session = client.sessions.create(
    params=CreateSessionParams(
        use_stealth=True,
        operating_systems=["macos"],
        device=["desktop"],
        locales=["en-US"],
        screen=ScreenConfig(width=1920, height=1080),
    )
)
ws_endpoint = session.ws_endpoint
```

**Session parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `useStealth` | bool | Enable fingerprint randomisation |
| `useUltraStealth` | bool | Maximum anti-detection (Enterprise) |
| `operatingSystems` | list | `["macos", "windows", "linux"]` |
| `device` | list | `["desktop", "mobile"]` |
| `locales` | list | `["en-US", "en-GB", ...]` |
| `screen` | object | `{ width, height }` |
| `solveCaptchas` | bool | Auto-solve CAPTCHAs (default: true) |
| `adblock` | bool | Block ads and trackers |
| `proxy` | object | Custom proxy config |

### 2.2 Stealth System

Three-tier stealth:

**Standard Stealth (`useStealth: true`):**
- Randomised User-Agent, screen resolution, locale
- Canvas fingerprint randomisation
- WebGL renderer/vendor spoofing
- AudioContext, WebRTC handling
- Ad blocking (optional)
- CAPTCHA solving (automatic)

**Advanced Fingerprinting:**
Hyperbrowser fingerprints browsers across:
- User-Agent and Accept-Language
- WebGL (renderer, vendor string)
- Canvas (pixel-level randomisation)
- AudioContext (frequency response randomisation)
- IP rotation with behaviour simulation

**Ultra Stealth (`useUltraStealth: true`, Enterprise):**
- Maximum bot detection bypass
- Full human behaviour simulation
- Contact: info@hyperbrowser.ai

### 2.3 HyperAgent — AI Agent Framework

HyperAgent is Hyperbrowser's open-source Playwright + AI extension (`@hyperbrowser/agent`):

```typescript
import { HyperAgent } from "@hyperbrowser/agent";

const agent = new HyperAgent({
    browserProvider: "Hyperbrowser",  // or local
});

// Simple task
const response = await agent.executeTask(
    "Go to hackernews and list the 5 most recent article titles"
);
console.log(response.output);

// Multi-page tasks
const page1 = await agent.newPage();
const page2 = await agent.newPage();

const destination = await page1.ai(
    "Go to google.com/travel/explore, set start to New York, return first destination"
);
const recommendations = await page2.ai(
    `I want to plan a trip to ${destination.output}. Recommend places to visit.`
);
```

**Page AI methods:**

```typescript
// Natural language action
await page.ai("Click the sign-in button and enter credentials");

// Extract structured data
const result = await page.extract({
    url: "https://example.com",
    prompt: "Extract the product name and price",
    schema: {
        type: "object",
        properties: {
            name: { type: "string" },
            price: { type: "string" }
        }
    }
});

// Get all active pages
const pages = await agent.getPages();
await agent.closeAgent();
```

### 2.4 Action Caching System

HyperAgent's key production feature — deterministic replay after first run:

```typescript
import { HyperAgent, ActionCacheOutput } from "@hyperbrowser/agent";
import fs from "fs";

const agent = new HyperAgent();
const page = await agent.newPage();

// First run — records XPath trace
const result = await page.ai("Navigate to amazon.com, search headphones");
const cache: ActionCacheOutput = result.actionCache;

// Save cache
fs.writeFileSync("action-cache.json", JSON.stringify(cache));

// Subsequent runs — replay from XPath (no LLM)
const savedCache: ActionCacheOutput = JSON.parse(
    fs.readFileSync("action-cache.json", "utf-8")
);
const replay = await page.runFromActionCache(savedCache, {
    maxXPathRetries: 3,     // Retry XPath 3x before LLM fallback
    debug: true,
});
```

**Cache schema (XPath-based):**
```json
{
    "taskId": "86d13abe-b9f3-4ca3-a9bb-bdeddf234cd1",
    "createdAt": "2025-12-06T05:44:52.257Z",
    "status": "completed",
    "steps": [
        {
            "stepIndex": 0,
            "instruction": "Click on the search field",
            "elementId": "0-138",
            "method": "click",
            "arguments": [],
            "frameIndex": 0,
            "xpath": "/html[1]/body[1]/div[1]/input[1]",
            "actionType": "actElement",
            "success": true
        }
    ]
}
```

### 2.5 AI Scraping Endpoints (No Session Required)

Built-in AI-powered scraping without managing sessions:

```typescript
// Scrape a single page
const scraped = await client.scrape.startAndWait({
    url: "https://example.com",
    scrapeOptions: {
        formats: ["markdown", "html"],
        includeTags: ["h1", "h2", "p", "article"],
        onlyMainContent: true,
    },
    sessionOptions: { solveCaptchas: true },
});

// Crawl entire site
const crawl = await client.crawl.startAndWait({
    url: "https://docs.example.com",
    maxPages: 50,
    scrapeOptions: { formats: ["markdown"] },
});

// Extract structured data
const extracted = await client.extract.startAndWait({
    urls: ["https://example.com/product"],
    prompt: "Extract product name, price, and availability",
    schema: ProductSchema,
});
```

### 2.6 Playwright + Puppeteer Integration

```typescript
// Playwright
import { chromium } from "playwright-core";
import { Hyperbrowser } from "@hyperbrowser/sdk";

const session = await client.sessions.create({ useStealth: true });
const browser = await chromium.connectOverCDP(session.wsEndpoint);
const context = browser.contexts()[0];
const page = await context.newPage();

await page.goto("https://example.com");
// Standard Playwright API from here

await client.sessions.stop(session.id);
```

```python
# Pyppeteer (Python)
from pyppeteer import connect
from hyperbrowser import AsyncHyperbrowser

async with AsyncHyperbrowser(api_key=os.getenv("HYPERBROWSER_API_KEY")) as client:
    session = await client.sessions.create()
    browser = await connect(
        browserWSEndpoint=session.ws_endpoint,
        defaultViewport=None,
    )
    pages = await browser.pages()
    page = pages[0]
    await page.goto("https://news.ycombinator.com/")
    title = await page.title()
```

### 2.7 LangChain Integration

```python
from langchain_hyperbrowser import (
    HyperbrowserScrapeTool,
    HyperbrowserCrawlTool,
    HyperbrowserExtractTool,
    HyperbrowserOpenAICUATool,  # Computer Use Agent
)
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_functions_agent

tools = [HyperbrowserScrapeTool(), HyperbrowserExtractTool()]
llm = ChatOpenAI(temperature=0)
agent = create_openai_functions_agent(llm, tools, verbose=True)
executor = AgentExecutor(agent=agent, tools=tools)
result = executor.invoke({"input": "Extract product info from https://example.com"})
```

---

## 3. Architecture Diagram

```
Developer / AI Agent
        ↓
Hyperbrowser SDK (Node.js / Python)
        ↓
REST API
  /sessions/create
  /sessions/stop
  /scrape, /crawl, /extract
        ↓
Session Provisioner (serverless)
  ├─ Managed Chromium
  ├─ Fingerprint randomisation
  ├─ Proxy rotation
  └─ CAPTCHA solver
        ↓
session.wsEndpoint (wss://)
        ↓
Playwright / Puppeteer / HyperAgent
        ↓
HyperAgent AI Layer (optional)
  ├─ page.ai() — natural language actions
  ├─ page.extract() — structured extraction
  ├─ executeTask() — multi-step autonomous
  └─ Action Cache — deterministic replay
        ↓
LangChain Integration (optional)
  Tools: Scrape, Crawl, Extract, CUA
```

---

## 4. Pricing

| Feature | Value |
|---------|-------|
| Session start time | <1 second |
| Max concurrent sessions | 1,000+ (enterprise) |
| CAPTCHA solving | Built-in, automatic |
| Standard stealth | Included |
| Ultra stealth | Enterprise only |
| Scrape API | Per-call pricing |
| Crawl API | Per-page pricing |

Credit-based pricing (exact rates: see hyperbrowser.ai/pricing).

---

## 5. Key Technical Decisions

**Why build HyperAgent on top of infrastructure?**
Browserbase stops at the CDP layer; the developer still needs an agent framework. Hyperbrowser collapses the stack — one SDK for both infrastructure and AI reasoning.

**Why XPath-based action caching?**
XPaths are more stable than CSS selectors and more precise than natural language. Storing the XPath of each interaction allows deterministic replay without any LLM. When XPath fails (site changed), the intent string is used for LLM-based re-identification.

**Why scrape/crawl/extract as API endpoints?**
Most scraping workflows don't need persistent sessions. Single-call endpoints amortise session startup overhead over one request, reducing cost.

---

## 6. Limitations & Known Issues

- **Credit-based pricing** — can be harder to predict costs vs. fixed per-hour models
- **Ultra Stealth = Enterprise** — strongest anti-detection not on self-serve plans
- **Newer platform** — less track record than Browserbase; some "growing pains" noted in community
- **No self-hosting** — fully managed cloud only

---

## 7. Conclusion

Hyperbrowser occupies a unique position as the only browser infrastructure provider that also ships an AI agent framework (HyperAgent) and AI-powered scraping endpoints. For teams starting fresh who want to avoid combining multiple vendors, the vertically integrated stack is compelling. For teams with existing frameworks (Playwright, browser-use, Stagehand), Hyperbrowser works as a drop-in CDP backend just like Browserbase.

**License:** Commercial (cloud service); HyperAgent is open-source (MIT)  
**SDKs:** Node.js (`@hyperbrowser/sdk`), Python (`hyperbrowser`)  
**HyperAgent repo:** https://github.com/hyperbrowserai/HyperAgent  
**Website:** https://www.hyperbrowser.ai  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: docs.hyperbrowser.ai, GitHub SDK repos, LangChain integration README, and community comparisons.*
