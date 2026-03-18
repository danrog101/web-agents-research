# Magnitude Research Report

## Executive Summary

Magnitude is an open-source, **vision-first browser automation framework** that achieved **93.9% on WebVoyager** — the highest documented open-source score. Unlike DOM-based frameworks, Magnitude sends raw screenshots to a visually grounded VLM (vision-language model) and interacts via pixel coordinates rather than DOM element indices. This approach generalises to visually complex pages where DOM methods fail: custom sliders, canvas elements, drag-and-drop interfaces, and any page where the HTML structure doesn't reflect the visual layout.

---

## 1. Architecture Overview

### Core Philosophy

**Vision-first, not DOM-first:** Magnitude's thesis is that modern VLMs (Claude Sonnet 4, Qwen-2.5VL) are now good enough to understand arbitrary GUIs from screenshots alone. Avoiding DOM parsing eliminates an entire category of failures (shadow DOM, dynamically generated selectors, canvas elements).

**Controllable automation:** Most browser agents follow "high-level prompt + tools = work until done." Magnitude allows both high-level (`act("Create a task")`) and low-level (`act("Drag to top of column")`) instructions — the developer controls the abstraction level per step.

**Planner → Executor split (test mode):** For the testing use case, Magnitude builds an action plan on the first run and caches it. Subsequent runs use only the Executor against the cached plan — fast, deterministic, and cheap.

### Package Structure

```
magnitude/
├── packages/
│   └── magnitude-core/
│       ├── src/
│       │   ├── browser/
│       │   │   ├── BrowserAgent.ts        # Main agent class
│       │   │   ├── BrowserSession.ts      # Playwright session management
│       │   │   └── startBrowserAgent.ts   # Public entry point
│       │   ├── vision/
│       │   │   ├── VisionModel.ts         # VLM abstraction
│       │   │   ├── ScreenshotCapture.ts   # Screenshot pipeline
│       │   │   └── ActionParser.ts        # Parse VLM action output
│       │   ├── actions/
│       │   │   ├── ActionExecutor.ts      # CDP/Playwright action execution
│       │   │   ├── MouseActions.ts        # Click, drag, hover
│       │   │   └── KeyboardActions.ts     # Type, key press, shortcut
│       │   ├── agent/
│       │   │   ├── AgentLoop.ts           # Main reasoning loop
│       │   │   └── ReasoningChain.ts      # CoT injection
│       │   ├── extract/
│       │   │   └── ExtractService.ts      # Structured data extraction
│       │   └── utils/
│       │       ├── ContextManager.ts      # Reasoning history window
│       │       └── PatchrightManager.ts   # Anti-bot browser setup
├── packages/
│   └── magnitude-test/
│       ├── src/
│       │   ├── TestRunner.ts              # Built-in test runner
│       │   ├── Planner.ts                 # Build action plan (first run)
│       │   ├── Executor.ts                # Replay cached plan
│       │   └── Assertions.ts             # Visual assertions
│       └── bin/
│           └── magnitude.ts              # CLI entry point
└── apps/
    └── magnitude-demo-repo/              # Example projects
```

---

## 2. Core Components Deep Dive

### 2.1 BrowserAgent (`packages/magnitude-core/src/browser/BrowserAgent.ts`)

The central class that users interact with:

```typescript
import { startBrowserAgent } from "magnitude-core";
import z from 'zod';

const agent = await startBrowserAgent({
    url: 'https://app.example.com',
    provider: {
        id: 'anthropic',
        options: {
            apiKey: process.env.ANTHROPIC_API_KEY,
            model: 'claude-sonnet-4-20250514',  // Required: grounded VLM
        }
    },
    // Optional
    headless: false,
    viewport: { width: 1280, height: 720 },
});
```

**Core API — three methods:**

```typescript
// High-level act — VLM interprets screenshot and finds how to do it
await agent.act('Create a task', {
    data: {
        title: 'Use Magnitude',
        description: 'Run npx create-magnitude-app',
    },
});

// Low-level act — precise drag/drop, pixel-level operations
await agent.act('Drag "Use Magnitude" to the top of the in-progress column');

// Extract — structured data with Zod schema
const tasks = await agent.extract(
    'List all in-progress tasks',
    z.array(z.object({
        title: z.string(),
        description: z.string(),
        difficulty: z.number().describe('Rate difficulty 1-5'),
    }))
);

// Verify — visual assertion for testing
await agent.verify('The task appears in the Done column');
```

### 2.2 Vision Model (`packages/magnitude-core/src/vision/VisionModel.ts`)

The VLM abstraction layer. Magnitude requires a **grounded VLM** — a model that can output precise pixel coordinates, not just text descriptions:

```typescript
interface VisionModel {
    // Takes screenshot → returns action with coordinates
    decide(
        screenshot: Buffer,
        instruction: string,
        reasoningHistory: ReasoningStep[],
    ): Promise<VisionAction>;
}

interface VisionAction {
    reasoning: string;          // Chain-of-thought
    action: {
        type: 'click' | 'drag' | 'type' | 'scroll' | 'hotkey' | 'done';
        x?: number;             // Pixel coordinates
        y?: number;
        text?: string;
        fromX?: number;         // For drag
        fromY?: number;
        toX?: number;
        toY?: number;
    };
}
```

**Supported VLMs (grounded models required):**
- `anthropic/claude-sonnet-4-20250514` — **recommended, best performance**
- `Qwen/Qwen2.5-VL-72B` — open-source alternative
- Any OpenAI-compatible endpoint with vision + coordinate grounding

### 2.3 Agent Loop (`packages/magnitude-core/src/agent/AgentLoop.ts`)

```typescript
class AgentLoop {
    async run(instruction: string): Promise<void> {
        while (true) {
            // 1. Capture screenshot
            const screenshot = await this.session.screenshot();
            
            // 2. Build reasoning context (last 20 turns only)
            const context = this.contextManager.buildContext(
                instruction,
                this.reasoningHistory.slice(-20),  // Context window management
                this.screenshotHistory.slice(-3),   // Last few screenshots
            );
            
            // 3. Call VLM
            const action = await this.visionModel.decide(
                screenshot,
                instruction,
                context,
            );
            
            // 4. Append to reasoning chain
            this.reasoningHistory.push({
                reasoning: action.reasoning,
                action: action.action,
                screenshot: screenshot,
            });
            
            // 5. Execute action
            if (action.action.type === 'done') break;
            await this.actionExecutor.execute(action.action);
        }
    }
}
```

**Reasoning Chain (CoT injection):**

From Magnitude's WebVoyager paper:
- Simple CoT injection before each action: ask for reasoning + next actions
- Reasoning chains are included in future turns — the model can build on previous reasoning
- Context is limited to last 20 turns to prevent context overflow
- Last 2-3 screenshots included for visual context

### 2.4 Action Executor (`packages/magnitude-core/src/actions/ActionExecutor.ts`)

Translates VLM output coordinates into browser actions via Playwright/Patchright:

```typescript
class ActionExecutor {
    async execute(action: VisionAction): Promise<void> {
        switch (action.type) {
            case 'click':
                await this.page.mouse.click(action.x, action.y);
                break;
            case 'drag':
                await this.page.mouse.move(action.fromX, action.fromY);
                await this.page.mouse.down();
                await this.page.mouse.move(action.toX, action.toY);
                await this.page.mouse.up();
                break;
            case 'type':
                await this.page.keyboard.type(action.text);
                break;
            case 'scroll':
                await this.page.mouse.wheel(0, action.direction === 'down' ? 300 : -300);
                break;
            case 'hotkey':
                await this.page.keyboard.press(action.keys);
                break;
        }
    }
}
```

### 2.5 Context Manager (`packages/magnitude-core/src/utils/ContextManager.ts`)

Manages the reasoning history window:

```typescript
class ContextManager {
    private readonly MAX_TURNS = 20;
    private readonly MAX_SCREENSHOTS = 3;
    
    buildContext(
        instruction: string,
        history: ReasoningStep[],
        screenshots: Buffer[],
    ): VLMContext {
        return {
            task: instruction,
            reasoning_history: history.slice(-this.MAX_TURNS),
            recent_screenshots: screenshots.slice(-this.MAX_SCREENSHOTS),
        };
    }
}
```

This is a key design decision: **do NOT accumulate all screenshots**. Keeping only the last 3 screenshots prevents context overflow while giving the model enough visual continuity.

### 2.6 Test Runner (`packages/magnitude-test/src/TestRunner.ts`)

For AI-native E2E testing:

```typescript
// magnitude.config.ts
export default {
    provider: {
        id: 'anthropic',
        options: { model: 'claude-sonnet-4-20250514' }
    }
};

// tests/magnitude/example.test.ts  
test('User can create and complete a task', { url: 'http://localhost:3000' },
    async ({ ai }) => {
        await ai.act('Create a task called "Write tests"');
        await ai.act('Drag "Write tests" to the Done column');
        await ai.verify('"Write tests" appears in the Done column');
    }
);
```

**Planner → Executor caching:**
```
First run:
  Planner builds action plan → saves to .magnitude/cache/
  Executor runs plan against live app
  
Subsequent runs:
  Executor only → uses cached plan
  Planner invoked only if plan execution fails
  → Much faster CI/CD cycles
```

---

## 3. Agent Execution Flow

### Automation Flow

```
agent.act("Add a task called 'Deploy to production'")
    ↓
BrowserSession.screenshot()
    → 1280×720 PNG buffer
    ↓
VisionModel.decide(screenshot, instruction, history)
    → Grounded VLM (Claude Sonnet 4)
    → Reasoning: "I see a task input field at the top. 
                   I should click it and type the task name."
    → Action: { type: "click", x: 640, y: 120 }
    ↓
ActionExecutor.execute({ type: "click", x: 640, y: 120 })
    → Playwright/Patchright page.mouse.click(640, 120)
    ↓
Next iteration:
    → New screenshot
    → Action: { type: "type", text: "Deploy to production" }
    ↓
Next iteration:
    → Action: { type: "hotkey", keys: "Enter" }
    ↓
Action: { type: "done" }
    → Loop ends
```

### Data Flow

```
Instruction (string)
    ↓
Screenshot (PNG buffer)
    ↓
VisionModel (grounded VLM)
    Input: screenshot + instruction + last 20 reasoning steps
    Output: reasoning string + action with pixel coordinates
    ↓
ActionExecutor
    → page.mouse.click(x, y) OR
    → page.keyboard.type(text) OR
    → page.mouse.drag(from, to) ...
    ↓
Loop until action.type === "done"
```

---

## 4. Configuration

### Setup

```bash
npm create magnitude-app@latest
# or add to existing project:
npm install magnitude-core magnitude-test
```

```typescript
// .env
ANTHROPIC_API_KEY=sk-ant-...

// Basic usage
import { startBrowserAgent } from "magnitude-core";

const agent = await startBrowserAgent({
    url: 'https://myapp.com',
    provider: {
        id: 'anthropic',
        options: {
            apiKey: process.env.ANTHROPIC_API_KEY,
            model: 'claude-sonnet-4-20250514',
        }
    }
});
```

### Supported Providers

```typescript
// Anthropic (recommended)
{ id: 'anthropic', options: { model: 'claude-sonnet-4-20250514' } }

// OpenAI-compatible with vision + grounding
{ id: 'openai-compatible', options: { baseURL: '...', model: '...' } }

// Qwen (open-source)
{ id: 'openai-compatible', options: {
    baseURL: 'https://dashscope.aliyuncs.com/...',
    model: 'qwen2.5-vl-72b'
}}
```

---

## 5. Architecture Diagrams

### Component Hierarchy

```
Magnitude (magnitude-core)
  ├─ BrowserAgent
  │   ├─ AgentLoop
  │   │   ├─ ContextManager (last 20 turns, last 3 screenshots)
  │   │   └─ ReasoningChain (CoT history)
  │   ├─ VisionModel (grounded VLM)
  │   │   ├─ Claude Sonnet 4 (recommended)
  │   │   └─ Qwen-2.5VL 72B (open-source)
  │   ├─ ActionExecutor
  │   │   ├─ MouseActions (click, drag, hover, scroll)
  │   │   └─ KeyboardActions (type, hotkey)
  │   └─ BrowserSession (Playwright/Patchright)
  │       └─ Chromium (with anti-bot patches)
  └─ ExtractService (Zod schema extraction)

magnitude-test
  ├─ TestRunner (AI-native E2E tests)
  ├─ Planner (first run → build plan)
  └─ Executor (subsequent runs → replay plan)
```

### Vision Loop

```
Screenshot (PNG)
    ↓
Grounded VLM
    → "I see [button at 340, 220]. Reasoning: [...]"
    → Action: { click, x:340, y:220 }
    ↓
Playwright/Patchright
    → page.mouse.click(340, 220)
    ↓
New screenshot
    ↓
Loop
```

### DOM-Based vs Vision-Based (Magnitude's argument)

```
DOM-Based (browser-use, Agent-E):
  HTML → element index → LLM selects [7] → click index 7
  ✅ Fast, cheap
  ❌ Fails on: price sliders, canvas, custom widgets, drag-drop

Vision-Based (Magnitude):
  Screenshot → VLM → pixel coordinates → execute
  ✅ Works on everything a human can see
  ✅ Generalisable across desktop, mobile, VMs
  ❌ Slower, more expensive per step
```

---

## 6. Key Technical Decisions

**Why pure vision instead of DOM + vision hybrid?**
From Magnitude's WebVoyager paper: "The pure-vision approach proves stronger than techniques that rely on DOM for interaction (like browser-use), while our agentic architecture is more robust than lightweight tool wrappers like Operator."
Concrete examples where DOM fails: Amazon price sliders (CSS range input), Cambridge Dictionary mini-games (canvas), Glassdoor star ratings (SVG overlay).

**Why Patchright instead of vanilla Playwright?**
- Patchright patches Chromium to avoid bot detection fingerprints
- Without it, many sites (Cloudflare, Google reCAPTCHA) block the browser
- Essential for real-world benchmark runs

**Why limit context to last 20 turns?**
- Full history causes context overflow on multi-step tasks
- Models don't effectively use information from early turns anyway
- 20 turns provides enough temporal context for recovery

**Why CoT reasoning chains?**
- Allows planning over time in a flexible way
- Disadvantage: can get stuck in "reasoning loops" (acknowledged in paper)
- Advantage: model builds on its own previous analysis

**Why Planner → Executor split for tests?**
- Pure planner on every run = expensive LLM inference
- Cached plan = deterministic, near-zero cost on repeat runs
- Supports CI/CD pipelines where cost and speed matter

---

## 7. Limitations & Known Issues

- **VLM required** — text-only models don't work; Claude Sonnet 4 or equivalent needed
- **Cost per step** — screenshot + VLM inference is significantly more expensive than DOM approaches (~$0.05-0.15/step vs ~$0.005-0.01)
- **"Reasoning loop" failure mode** — agent can get stuck reasoning without making progress; acknowledged in WebVoyager paper
- **TypeScript only** — no Python SDK
- **Slower per step** — 1-3 seconds per VLM call vs milliseconds for cached DOM operations
- **Small community** — ~4k GitHub stars vs browser-use's ~50k

---

## 8. Conclusion

Magnitude represents the current frontier of open-source vision-based browser automation. Its 93.9% WebVoyager score demonstrates that pure-vision approaches now outperform DOM-based methods when a capable grounded VLM is available. The framework is particularly valuable for testing and automation of visually complex applications where DOM methods systematically fail.

**WebVoyager score:** 93.9% (Steel leaderboard, Skyvern filtered 50-task dataset, Claude Sonnet 4)  
**License:** Apache-2.0  
**Language:** TypeScript  
**Repository:** https://github.com/magnitudedev/browser-agent  
**Snapshot date:** 2026-03-18  

*Generated: 2026-03-18. Based on: GitHub source analysis, magnitudedev/webvoyager benchmark repo, README.md, and HN discussion.*
