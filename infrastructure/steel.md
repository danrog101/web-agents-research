
# Steel Research Report

## Executive Summary

Steel is an **open-source browser API for AI agents**, developed by steel-dev and backed by Y Combinator. It is architecturally unique among cloud browser providers: the core codebase (`steel-browser`) is fully open-source (Apache-2.0) and self-hostable via Docker, while a managed cloud version (`steel.dev`) is available for teams that don't want to manage infrastructure. Steel positions itself as the most transparent, developer-friendly browser API — you can inspect, modify, and deploy the exact same code that powers Steel Cloud.

Steel also maintains the **Steel Leaderboard** — the de facto public benchmark for comparing web agent frameworks on WebVoyager tasks.

---

## 1. Architecture Overview

### Core Philosophy

**Open-source first:** The entire browser runtime, session management, and API are open-source. This gives teams full control, full auditability, and the ability to self-host in their own VPC or on bare metal.

**AI-first token efficiency:** Steel's content extraction pipeline is designed to minimise LLM token usage — converting pages to clean Markdown reduces context by up to 80% vs raw HTML.

**Infrastructure abstraction:** Like Browserbase and Hyperbrowser, Steel abstracts browser management. But Steel additionally exposes `/scrape`, `/screenshot`, and `/pdf` endpoints for single-call operations without session management.

### Repository Structure (steel-browser)

```
steel-browser/                      (Apache-2.0, GitHub: steel-dev/steel-browser)
├── api/
│   ├── src/
│   │   ├── server.ts               # Fastify API server
│   │   ├── routes/
│   │   │   ├── sessions.ts         # POST /sessions, GET /sessions/:id
│   │   │   ├── scrape.ts           # POST /scrape
│   │   │   ├── screenshot.ts       # POST /screenshot
│   │   │   ├── pdf.ts              # POST /pdf
│   │   │   └── utils.ts            # Helper endpoints
│   │   ├── browser/
│   │   │   ├── browser-manager.ts  # Chrome process lifecycle
│   │   │   ├── session-manager.ts  # Session state + CDP proxy
│   │   │   └── stealth.ts          # Stealth plugin injection
│   │   └── proxy/
│   │       └── proxy-chain.ts      # Proxy rotation management
├── ui/
│   ├── src/
│   │   ├── App.tsx                 # Session viewer UI (React)
│   │   ├── SessionList.tsx
│   │   └── SessionDetail.tsx       # Live view, replay
├── docker-compose.yml
├── docker-compose.dev.yml
└── Dockerfile
```

---

## 2. Core Components Deep Dive

### 2.1 Sessions API

```python
# Python SDK
from steel import Steel

client = Steel(steel_api_key=os.getenv("STEEL_API_KEY"))

# Create session
session = client.sessions.create()
print(f"Session ID: {session.id}")
print(f"Watch live: {session.session_viewer_url}")
# → https://app.steel.dev/sessions/{id}

# Create with options
session = client.sessions.create(
    use_proxy=True,
    solve_captcha=True,
    session_timeout=1800,      # 30 minutes (default: 900s)
    user_agent="Mozilla/5.0...",
    concurrency_id="my-pool",  # Group related sessions
)

# Connect Playwright
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp(
        f"wss://connect.steel.dev?sessionId={session.id}&apiKey={STEEL_API_KEY}"
    )
    page = browser.contexts[0].pages[0]
    page.goto("https://example.com")
    browser.close()

# Release session when done
client.sessions.release(session.id)
```

```typescript
// Node.js SDK
import Steel from "steel-sdk";
const client = new Steel({ steelApiKey: process.env.STEEL_API_KEY! });

const session = await client.sessions.create({
    useProxy: true,
    solveCaptcha: true,
});

// Connect Puppeteer
import puppeteer from "puppeteer-core";
const browser = await puppeteer.connect({
    browserWSEndpoint: `wss://connect.steel.dev?sessionId=${session.id}&apiKey=${process.env.STEEL_API_KEY}`,
});

// Release
await client.sessions.release(session.id);
```

**Session fields:**

```typescript
interface Session {
    id: string;
    status: "live" | "released" | "failed";
    sessionViewerUrl: string;       // Live/replay viewer URL
    websocketUrl: string;           // CDP WebSocket
    createdAt: Date;
    updatedAt: Date;
    duration: number;               // seconds
    proxyBytesUsed: number;
    creditsUsed: number;
}
```

### 2.2 Single-Call Endpoints (No Session)

```python
# Scrape — returns HTML, cleaned HTML, Markdown, Readability
result = client.scrape.scrape({
    "url": "https://example.com",
    "format": ["markdown", "html", "readability"],
    "screenshot": True,
    "pdf": True,
    "delay": 2000,          # ms wait after load
    "use_proxy": True,
})
print(result.content.markdown)
print(result.metadata.title)

# Screenshot
screenshot = client.screenshot.screenshot({
    "url": "https://example.com",
    "full_page": True,
    "format": "png",       # or "jpeg"
})

# PDF
pdf_result = client.pdf.pdf({
    "url": "https://example.com",
    "format": "A4",
    "print_background": True,
})
```

**Scrape response format:**
```json
{
    "content": {
        "html": "<!DOCTYPE html>...",
        "cleaned_html": "<article>...</article>",
        "markdown": "# Page Title\n\nContent...",
        "readability": {
            "title": "Page Title",
            "content": "...",
            "textContent": "..."
        }
    },
    "metadata": {
        "title": "Page Title",
        "description": "...",
        "ogImage": "https://...",
        "statusCode": 200,
        "language": "en",
        "timestamp": "2026-03-18T10:00:00Z"
    }
}
```

### 2.3 Stealth System

Steel uses puppeteer-extra stealth plugins:

```typescript
// stealth.ts — internal implementation
import puppeteer from "puppeteer-extra";
import StealthPlugin from "puppeteer-extra-plugin-stealth";

puppeteer.use(StealthPlugin());
// Stealth plugin patches:
//   - navigator.webdriver = false
//   - navigator.plugins (mimics real browser)
//   - navigator.languages
//   - Canvas fingerprinting
//   - WebGL fingerprinting
//   - Chrome runtime presence
//   - Permission handling
//   - iframe contentWindow
```

**Anti-detection capabilities:**
- `navigator.webdriver` removal
- Realistic `navigator.plugins` array
- Canvas fingerprint randomisation
- Chrome runtime mocking
- Permissions API patching
- All puppeteer-extra-plugin-stealth patches

### 2.4 Browser Manager (`api/src/browser/browser-manager.ts`)

```typescript
class BrowserManager {
    // Launch Chrome with custom args
    async launchBrowser(options: LaunchOptions): Promise<Browser> {
        const browser = await puppeteer.launch({
            headless: true,
            args: [
                "--no-sandbox",
                "--disable-setuid-sandbox",
                "--disable-dev-shm-usage",
                "--disable-accelerated-2d-canvas",
                "--no-first-run",
                "--no-zygote",
                `--proxy-server=${proxyUrl}`,    // if proxy configured
            ],
            executablePath: process.env.CHROME_EXECUTABLE_PATH,
        });
        return browser;
    }
    
    // CDP proxy — expose browser via WebSocket
    async createCDPProxy(browser: Browser, sessionId: string): Promise<void> {
        // Proxy CDP traffic to/from the managed Chrome instance
    }
}
```

### 2.5 Session Manager (`api/src/browser/session-manager.ts`)

```typescript
class SessionManager {
    private sessions: Map<string, SessionState> = new Map();
    
    async createSession(options: SessionOptions): Promise<Session> {
        const sessionId = generateId();
        const browser = await this.browserManager.launchBrowser(options);
        
        this.sessions.set(sessionId, {
            id: sessionId,
            browser,
            createdAt: new Date(),
            timeout: options.sessionTimeout || 900, // 15min default
        });
        
        // Auto-cleanup on timeout
        setTimeout(() => {
            this.releaseSession(sessionId);
        }, options.sessionTimeout * 1000);
        
        return { id: sessionId, websocketUrl: this.getCDPUrl(sessionId) };
    }
    
    async releaseSession(sessionId: string): Promise<void> {
        const session = this.sessions.get(sessionId);
        if (session) {
            await session.browser.close();
            this.sessions.delete(sessionId);
        }
    }
}
```

### 2.6 Session Viewer UI (`ui/`)

```
React SPA served at localhost:5173 (dev) / included in Docker image
├─ SessionList — all active and completed sessions
│   └─ Status, duration, credits used, live/replay badge
└─ SessionDetail — per-session view
    ├─ Live View — embedded browser iframe (real-time)
    ├─ Session Replay — rrweb recording playback
    ├─ Request Log — all HTTP requests/responses
    └─ Console Log — JavaScript console output
```

---

## 3. Self-Hosting

```bash
# Option 1: Docker (recommended, 2 containers)
git clone https://github.com/steel-dev/steel-browser
cd steel-browser
docker compose up

# API:  http://localhost:3000
# UI:   http://localhost:5173
# CDP:  ws://localhost:9223

# Option 2: Node.js directly
export CHROME_EXECUTABLE_PATH=/usr/bin/google-chrome
npm run dev

# Option 3: Docker (API only, no UI)
docker build -t steel .
docker run -p 3000:3000 -p 9223:9223 steel

# Option 4: Railway 1-click deploy
# https://railway.app → Deploy → steel-dev/steel-browser
```

### Environment Variables

```bash
CHROME_EXECUTABLE_PATH=/path/to/chrome  # Custom Chrome location
STEEL_API_KEY=your_key                  # Auth for self-hosted API
PROXY_URL=http://user:pass@host:port    # Default proxy
PORT=3000                               # API port
CDP_PORT=9223                           # CDP debug port
```

### Cloud API

```bash
# Steel Cloud (no self-hosting)
export STEEL_API_KEY=your_steel_cloud_key

# CDP URL format for cloud
wss://connect.steel.dev?sessionId={id}&apiKey={key}

# Self-hosted CDP URL
ws://localhost:9223?sessionId={id}
```

---

## 4. REST API Reference

```
# Sessions
POST   /sessions              Create new session
GET    /sessions              List sessions
GET    /sessions/:id          Get session details
POST   /sessions/:id/release  Release (terminate) session
POST   /sessions/:id/context  Save/load browser context

# Single-call operations (no session)
POST   /scrape                Scrape URL → HTML/Markdown/Readability
POST   /screenshot            Screenshot URL → PNG/JPEG
POST   /pdf                   Render URL → PDF

# Utilities
GET    /health                Health check
GET    /documentation         Swagger UI
```

**Swagger UI:** `http://localhost:3000/documentation`

---

## 5. Architecture Diagram

```
Developer / AI Agent
        ↓
Steel SDK (Python: steel / Node.js: steel-sdk)
OR direct HTTP to REST API
        ↓
Steel API Server (Fastify, Node.js)
  ├─ SessionManager
  │   ├─ BrowserManager (Puppeteer + stealth plugins)
  │   │   └─ Chrome process (local or custom path)
  │   ├─ CDP Proxy (expose via WebSocket)
  │   └─ Auto-cleanup (timeout manager)
  ├─ ScrapeHandler (single-call)
  ├─ ScreenshotHandler
  └─ PDFHandler
        ↓
WebSocket (CDP)
wss://connect.steel.dev?sessionId=XXX  (Cloud)
ws://localhost:9223?sessionId=XXX      (Self-hosted)
        ↓
Developer connects:
  playwright.chromium.connectOverCDP(...)
  puppeteer.connect({ browserWSEndpoint: ... })
        ↓
Session Viewer UI (React, port 5173)
  ├─ Live View
  ├─ Replay
  └─ Request Logs

Steel Leaderboard:
  steel.dev/leaderboard → WebVoyager benchmark rankings
```

---

## 6. Steel Leaderboard

Steel maintains the public **WebVoyager Leaderboard** — the most cited ranking for web agent frameworks:

```
# As of 2026-03-18:
Rank 1:  Surfer 2 (H Company)   97.1%
Rank 2:  Magnitude              93.9%
Rank 3:  AIME Browser-Use       92.34%
Rank 4:  Browserable            90.4%
Rank 5:  Browser Use            89.1%
Rank 6:  Operator (OpenAI)      87%
Rank 7:  Skyvern 2.0            85.85%
Rank 8:  Project Mariner        83.5%
...
Rank 11: Agent-E                73.1%
Rank 14: WebVoyager (original)  59.1%
```

The leaderboard at `https://github.com/steel-dev/awesome-web-agents` is the canonical source for web agent capability comparisons.

---

## 7. Key Technical Decisions

**Why open-source the browser runtime?**
Transparency and trust. Enterprises deploying AI agents need to audit what the browser is doing. With Steel, they can inspect every line of code, modify stealth behaviour, and deploy in their own infrastructure.

**Why Fastify instead of Express?**
Fastify has 2-3× lower overhead per request, critical when session creation must be sub-second. The Swagger plugin also auto-generates API docs.

**Why puppeteer-extra-plugin-stealth?**
Mature, well-maintained collection of stealth patches. Steel doesn't maintain a custom Chromium fork (unlike Browserbase's Advanced Stealth), keeping the build process simpler.

**Why Markdown extraction built-in?**
AI agents consume page content as LLM context. Raw HTML is token-wasteful. Markdown extraction reduces token usage ~80% while preserving semantic structure, directly reducing LLM costs.

---

## 8. Limitations & Known Issues

- **Stealth less advanced than Browserbase** — no custom Chromium fork; Cloudflare and advanced bot detection may still trigger
- **Self-hosting requires Chrome** — must have a Chrome executable available; Chromium variants may have compatibility issues
- **No action caching** — unlike Stagehand or HyperAgent, no built-in deterministic replay
- **Session limit (default 24h max)** — long-running sessions (>24h) not supported
- **Young project** — public beta, API may change

---

## 9. Conclusion

Steel is the most developer-friendly browser infrastructure option for teams that value control over convenience. Its open-source architecture, self-hosting support, and clean REST API make it ideal for teams with strict data residency requirements, internal deployments, or those who want to extend browser behaviour. The Markdown scraping endpoints are a genuine differentiator for AI teams optimising LLM token costs.

The Steel Leaderboard is independently valuable — even for teams not using Steel infrastructure, it provides the most comprehensive web agent benchmark comparison available.

**License:** Apache-2.0 (open-source)  
**SDKs:** Python (`steel`), Node.js (`steel-sdk`)  
**GitHub:** https://github.com/steel-dev/steel-browser  
**Website:** https://steel.dev  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: steel-dev/steel-browser GitHub source, README, API reference (Swagger), docs.steel.dev, and Steel Leaderboard.*
