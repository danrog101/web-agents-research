# Browserbase Research Report

## Executive Summary

Browserbase is a **cloud-native, serverless browser infrastructure platform** purpose-built for AI agents and web automation. It provides managed Chromium instances accessible via CDP WebSocket, eliminating the need to run and maintain local browser fleets. Founded in 2023, it has become the default cloud browser backend for Stagehand (its own open-source framework) and is widely used by AI agent teams at scale. Core differentiators are its two-tier stealth system, session recording/replay, and the Contexts API for persistent authentication state.

---

## 1. Architecture Overview

### Core Philosophy

**Browser-as-a-Service for AI agents:** Browserbase is not an agent framework — it is browser infrastructure. It solves the "last mile" problem for AI agents: managing Chrome at scale, handling anti-bot detection, rotating IPs, solving CAPTCHAs, and recording sessions for debugging. The developer's code (Playwright, Puppeteer, Stagehand) remains unchanged; only the browser endpoint changes.

**Serverless scaling:** Sessions are created on demand via API call. No pre-warming, no capacity planning. The platform spins up Chromium instances in milliseconds, powered by 4 vCPUs per session.

**AI-first integrations:** Unlike Browserless (which is developer-first), Browserbase explicitly targets AI agent workflows — native support for Stagehand, CrewAI, Browser-Use, and any framework that accepts a CDP URL.

### Service Architecture

```
Developer / AI Agent
        ↓
  SDK Call (Node.js or Python)
  bb.sessions.create({ ... })
        ↓
Browserbase API (REST)
  POST /v1/sessions
        ↓
Session Provisioner
  ├─ Allocates 4-vCPU Chromium instance
  ├─ Applies stealth configuration
  ├─ Attaches proxy
  └─ Returns CDP WebSocket URL
        ↓
wss://connect.browserbase.com?sessionId=XXX&apiKey=YYY
        ↓
Developer connects:
  playwright.chromium.connectOverCDP(wss://...)
  puppeteer.connect({ browserWSEndpoint: wss://... })
        ↓
Full browser control via CDP
```

---

## 2. Core Components Deep Dive

### 2.1 Sessions API

The primary interface. Creates, lists, manages, and terminates browser sessions.

```python
# Python SDK
from browserbase import Browserbase

bb = Browserbase(api_key=os.environ["BROWSERBASE_API_KEY"])

# Basic session
session = bb.sessions.create(project_id="PROJ_ID")
print(session.id)
print(session.connect_url)  # wss://connect.browserbase.com?sessionId=...

# Full stealth session
session = bb.sessions.create(
    project_id="PROJ_ID",
    proxies=True,
    browser_settings={
        "advanced_stealth": True,
        "viewport": {"width": 1920, "height": 1080},
        "fingerprint": {
            "operating_systems": ["macos"],
            "devices": ["desktop"],
            "locales": ["en-US"],
        },
    },
)
```

```typescript
// Node.js SDK
import Browserbase from "@browserbasehq/sdk";

const bb = new Browserbase({ apiKey: process.env.BROWSERBASE_API_KEY! });

const session = await bb.sessions.create({
    projectId: "PROJ_ID",
    proxies: true,
    browserSettings: {
        advancedStealth: true,
        viewport: { width: 1920, height: 1080 },
    },
});
console.log(session.connectUrl);
```

**Session fields:**

```typescript
interface Session {
    id: string;
    projectId: string;
    status: "RUNNING" | "COMPLETED" | "ERROR" | "TIMED_OUT";
    connectUrl: string;        // wss:// CDP WebSocket
    sessionViewerUrl: string;  // https://browserbase.com/sessions/{id}
    createdAt: string;
    endedAt: string | null;
    duration: number;          // seconds
    proxyBytes: number;
}
```

### 2.2 Stealth System

Two-tier stealth with distinct technical approaches:

**Basic Stealth Mode (all plans):**
- Automatic CAPTCHA detection and solving (hCaptcha, reCAPTCHA, Cloudflare Turnstile)
- Randomised browser fingerprints per session:
  - `User-Agent`, `Accept-Language`, screen resolution, colour depth
  - Canvas fingerprint randomisation
  - WebGL renderer/vendor spoofing
  - AudioContext API randomisation
- Realistic viewport and window sizes

**Advanced Stealth Mode (Scale plan):**
```python
session = bb.sessions.create(
    browser_settings={"advanced_stealth": True},
    proxies=True,
)
# Uses a custom Chromium build maintained by Browserbase's Stealth Team
# Randomises low-level browser signals:
#   - Mouse movement patterns
#   - Keyboard timing
#   - WebRTC IP handling
#   - JS execution timing
#   - Network request timing
```

**Browserbase Identity (beta, Scale):**
- Official Cloudflare bypass via cryptographic authentication
- Part of Cloudflare's "Signed Agents" programme
- Provably legitimate traffic at the protocol level

### 2.3 Contexts API (Persistent Authentication)

Contexts persist browser state (cookies, localStorage, IndexedDB) across multiple sessions:

```python
# Create a persistent context
context = bb.contexts.create(project_id="PROJ_ID")
context_id = context.id

# Session 1 — log in
session1 = bb.sessions.create(
    project_id="PROJ_ID",
    browser_settings={"context": {"id": context_id, "persist": True}},
)
# ... agent logs into Gmail, state is saved to context

# Session 2 — already authenticated
session2 = bb.sessions.create(
    project_id="PROJ_ID",
    browser_settings={"context": {"id": context_id, "persist": True}},
)
# ... agent picks up from authenticated state, no re-login needed
```

This is critical for AI agents that need to access services behind authentication without storing credentials in prompts.

### 2.4 Extensions API

Load custom Chrome extensions into sessions:

```python
# Upload extension once
extension = bb.extensions.create(file=open("extension.zip", "rb"))

# Use in session
session = bb.sessions.create(
    browser_settings={
        "extension_id": extension.id,
    }
)
```

Use cases: custom ad blockers, request interceptors, cookie managers, DOM manipulation helpers.

### 2.5 Live View (Embedded Session)

```typescript
// Embed live session view in your application
const session = await bb.sessions.create({ projectId: "PROJ_ID" });

// Embed URL in iframe
const liveViewUrl = `https://www.browserbase.com/devtools-fullscreen/inspector.html?sessionId=${session.id}&apiKey=${API_KEY}`;

// React component
<iframe src={liveViewUrl} width="1280" height="720" />
```

Live View provides real-time visual feedback of the agent's actions — essential for human-in-the-loop workflows and debugging.

### 2.6 Downloads and File Handling

```python
# Files downloaded during session are automatically stored
session = bb.sessions.create(project_id="PROJ_ID")
# ... agent downloads a PDF invoice ...
session_id = session.id

# Retrieve downloads after session ends
downloads = bb.sessions.downloads.list(session_id)
for download in downloads:
    content = bb.sessions.downloads.retrieve(session_id, download.id)
    open(f"{download.filename}", "wb").write(content)
```

### 2.7 Playwright / Puppeteer Integration

```python
# Playwright (Python)
from playwright.sync_api import sync_playwright
from browserbase import Browserbase

bb = Browserbase(api_key=os.environ["BROWSERBASE_API_KEY"])
session = bb.sessions.create(project_id="PROJ_ID")

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(session.connect_url)
    context = browser.contexts[0]
    page = context.pages[0]
    
    page.goto("https://example.com")
    # All your existing Playwright code works unchanged
    
    browser.close()
```

```typescript
// Puppeteer (TypeScript)
import puppeteer from "puppeteer-core";
import Browserbase from "@browserbasehq/sdk";

const session = await bb.sessions.create({ projectId: "PROJ_ID" });
const browser = await puppeteer.connect({
    browserWSEndpoint: session.connectUrl,
});
const page = await browser.newPage();
await page.goto("https://example.com");
```

### 2.8 Stagehand Integration (Native)

Stagehand is Browserbase's own browser automation framework:

```typescript
import { Stagehand } from "@browserbasehq/stagehand";

const stagehand = new Stagehand({
    env: "BROWSERBASE",
    browserbaseApiKey: process.env.BROWSERBASE_API_KEY,
    browserbaseProjectId: process.env.BROWSERBASE_PROJECT_ID,
});
await stagehand.init();
// Stagehand automatically creates a Browserbase session
```

---

## 3. API Endpoints

```
POST   /v1/sessions                    Create new session
GET    /v1/sessions                    List sessions
GET    /v1/sessions/{id}               Get session details
DELETE /v1/sessions/{id}               Terminate session
GET    /v1/sessions/{id}/recording     Get session recording
GET    /v1/sessions/{id}/logs          Get request logs
GET    /v1/sessions/{id}/downloads     List downloaded files
GET    /v1/sessions/{id}/downloads/{file_id}

POST   /v1/contexts                    Create context
GET    /v1/contexts/{id}               Get context
DELETE /v1/contexts/{id}               Delete context

POST   /v1/extensions                  Upload extension
DELETE /v1/extensions/{id}             Delete extension

POST   /v1/scrape                      Scrape URL (single-call, no session needed)
POST   /v1/fetch                       Fetch URL (single-call)
```

**Scrape endpoint (no session required):**
```bash
curl https://api.browserbase.com/v1/scrape \
  -H "X-BB-API-Key: $BB_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "url": "https://example.com", "format": ["markdown", "html"], "screenshot": true }'
```

---

## 4. Debugging Tools

### Session Recording

Every session is recorded by default. Replay accessible at:
```
https://www.browserbase.com/sessions/{sessionId}
```

Recording captures:
- Full screen video (rrweb-based DOM recording)
- CDP network requests/responses
- Console logs
- JavaScript errors
- Action timeline

### Session Inspector

```typescript
// Print session replay URL after run
const session = await bb.sessions.create({ projectId: "PROJ_ID" });
console.log(`Debug at: https://www.browserbase.com/sessions/${session.id}`);
```

---

## 5. Architecture Diagram

```
AI Agent / Developer Code
  (Playwright / Puppeteer / Stagehand / Selenium)
        ↓
Browserbase SDK
  (Node.js: @browserbasehq/sdk)
  (Python: browserbase)
        ↓
Browserbase REST API
  POST /v1/sessions { advancedStealth, proxies, context }
        ↓
Session Provisioner (serverless)
  ├─ Chrome 4-vCPU instance
  ├─ Stealth Chromium (custom build)
  ├─ Residential Proxy Network
  ├─ CAPTCHA Solver (background)
  └─ CDP endpoint exposed
        ↓
wss://connect.browserbase.com?sessionId=XXX
        ↓
Developer connects via CDP
        ↓
Supporting services:
  ├─ Contexts API (persistent auth state)
  ├─ Extensions API (custom Chrome extensions)
  ├─ Live View (iframe embedding)
  ├─ Session Recording (rrweb replay)
  └─ Downloads Storage
```

---

## 6. Pricing

| Plan | Price | Browser Hours | Concurrency | Stealth |
|------|-------|---------------|-------------|---------|
| Free | $0 | Limited | 3 | Basic |
| Developer | ~$20/mo | 200h included | 3 | Basic |
| Startup | $99/mo | 500h + $0.10/h | 25-100 | Basic |
| Scale | Custom | Volume discount | 100+ | Advanced |

Proxy: $10-12/GB. Session minimum billing: 1 minute.

---

## 7. Key Technical Decisions

**Why serverless (not persistent pools)?**
AI agent workloads are bursty — one run needs 1 browser, next needs 50. Serverless eliminates over-provisioning and the need for a browser pool manager.

**Why CDp (not Playwright protocol)?**
CDP is the universal browser automation protocol. Any tool that speaks CDP (Playwright, Puppeteer, Selenium, Stagehand, custom code) works with Browserbase without modification.

**Why 4 vCPUs per session?**
Page loading is CPU-bound (JS execution, rendering). 4 vCPUs ensures modern, JavaScript-heavy pages load at normal speed and don't time out.

**Why record all sessions?**
AI agents are non-deterministic. Without session recordings, debugging failures is nearly impossible — you can't reproduce the exact state. Recording is the only reliable post-hoc debugging tool.

---

## 8. Limitations & Known Issues

- **1-minute minimum billing** — short tasks under 60 seconds still bill for 1 minute
- **Session duration limit** — max 60 minutes per session (Scale may be higher)
- **Advanced Stealth = Scale plan** — not available on cheaper plans
- **Cloudflare Identity = beta** — limited availability
- **Cloud-only** — no self-hosting option (unlike Steel)
- **Pricing can accumulate quickly** at high concurrency

---

## 9. Conclusion

Browserbase is the most polished cloud browser platform in the AI agent space. Its session recording, Contexts API, and two-tier stealth system solve the three hardest infrastructure problems for production web agents. Its tight integration with Stagehand (its own framework) makes it a natural default for TypeScript-based agent stacks. The main downside vs. Steel is cost and lack of self-hosting.

**License:** Commercial (closed-source cloud service)  
**SDKs:** Node.js (`@browserbasehq/sdk`), Python (`browserbase`)  
**Website:** https://www.browserbase.com  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: official documentation, API reference, stealth-mode docs, pricing page, and GitHub SDK repositories.*
