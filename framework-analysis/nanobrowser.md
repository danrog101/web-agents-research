# Nanobrowser Research Report

## Executive Summary

Nanobrowser is an **open-source Chrome Extension** providing AI-powered web automation that runs entirely in the user's browser. It is architecturally distinct from all other frameworks: instead of CDP/Playwright, it uses Chrome Extension APIs to access the browser — making it invisible to anti-bot systems. A multi-agent system of three specialised roles (Planner, Navigator, Validator) coordinates to complete tasks, with all execution happening locally in the user's live browser session.

---

## 1. Architecture Overview

### Core Philosophy

**Extension-native automation:** By using Chrome Extension APIs instead of CDP WebSocket connections, Nanobrowser cannot be detected as automation. CDP creates detectable WebSocket connections and sets `navigator.webdriver = true`; Extension APIs appear identical to normal browser behaviour.

**Privacy-first:** Everything runs locally. Credentials, cookies, and browsing data never leave the machine. No cloud account required.

**Per-agent model assignment:** Each of the three agent roles (Planner, Navigator, Validator) can use a different LLM, allowing cost/performance optimisation.

### Package Structure

```
nanobrowser/           (Chrome Extension, TypeScript)
├── src/
│   ├── agents/
│   │   ├── planner.ts          # Planner Agent implementation
│   │   ├── navigator.ts        # Navigator Agent implementation
│   │   └── validator.ts        # Validator Agent implementation
│   ├── background/
│   │   └── service_worker.ts   # Extension background service worker
│   ├── content/
│   │   └── content_script.ts   # Injected into pages for DOM access
│   ├── side_panel/
│   │   ├── App.tsx              # React chat interface
│   │   └── ChatHistory.tsx
│   ├── storage/
│   │   └── session_storage.ts  # LLM keys, settings
│   └── manifest.json           # Chrome Extension manifest v3
├── dist/                        # Built extension
└── package.json
```

---

## 2. Core Components Deep Dive

### 2.1 Three-Agent System

```typescript
// Planner Agent — receives user goal, creates execution plan
class PlannerAgent {
    async plan(goal: string, pageContext: PageContext): Promise<Plan> {
        // Uses configured Planner LLM (e.g., GPT-4o)
        const response = await this.llm.chat([
            { role: "system", content: PLANNER_SYSTEM_PROMPT },
            { role: "user", content: `Goal: ${goal}\nPage: ${pageContext}` }
        ]);
        return parsePlan(response);
    }
}

// Navigator Agent — executes individual steps on the page
class NavigatorAgent {
    async executeStep(step: PlanStep, page: ExtensionPage): Promise<StepResult> {
        // Uses configured Navigator LLM (e.g., GPT-4o-mini for cost)
        const action = await this.llm.decideAction(step, page.dom, page.screenshot);
        return await this.executeBrowserAction(action);
    }
}

// Validator Agent — verifies goal completion
class ValidatorAgent {
    async validate(goal: string, result: TaskResult): Promise<ValidationResult> {
        // Checks if the goal was actually achieved
        return await this.llm.validate(goal, result);
    }
}
```

### 2.2 Browser Access via Extension APIs

```typescript
// content_script.ts — runs in page context
// Direct DOM access without CDP

// Read DOM
const pageContent = document.documentElement.innerHTML;
const interactableElements = document.querySelectorAll(
    'button, a, input, select, textarea, [onclick], [role="button"]'
);

// Simulate user interactions
element.click();
element.focus();
element.value = "text to type";
element.dispatchEvent(new InputEvent('input', { bubbles: true }));

// background/service_worker.ts — extension service worker
// Tab management
chrome.tabs.executeScript(tabId, { code: '...' });
chrome.tabs.sendMessage(tabId, { type: 'GET_DOM' });

// Screenshot
chrome.tabs.captureVisibleTab(null, { format: 'png' }, (dataUrl) => {
    // screenshot available
});
```

### 2.3 Side Panel Chat Interface

```typescript
// side_panel/App.tsx
export function App() {
    const [messages, setMessages] = useState<Message[]>([]);
    const [isRunning, setIsRunning] = useState(false);
    
    const handleSubmit = async (goal: string) => {
        setIsRunning(true);
        // Send to background service worker
        chrome.runtime.sendMessage({
            type: 'RUN_TASK',
            goal: goal,
            config: getUserLLMConfig(),
        });
    };
    
    // Real-time status updates via message passing
    chrome.runtime.onMessage.addListener((msg) => {
        if (msg.type === 'STEP_UPDATE') {
            setMessages(prev => [...prev, msg.update]);
        }
    });
}
```

### 2.4 LLM Configuration

Users configure different models per agent in Settings:

```typescript
interface LLMConfig {
    planner: {
        provider: 'openai' | 'anthropic' | 'google' | 'groq' | 'cerebras';
        model: string;     // e.g., "gpt-4o"
        apiKey: string;
    };
    navigator: {
        provider: string;
        model: string;     // e.g., "gpt-4o-mini" for lower cost
        apiKey: string;
    };
    validator: {
        provider: string;
        model: string;
        apiKey: string;
    };
}
```

**Recommended configurations:**
```
High performance:
  Planner: GPT-4o
  Navigator: GPT-4o
  Validator: GPT-4o-mini

Cost-effective:
  Planner: GPT-4o-mini
  Navigator: Claude-3-haiku
  Validator: GPT-4o-mini
```

---

## 3. Execution Flow

```
User opens Nanobrowser side panel
Types: "Find a Bluetooth speaker on Amazon under $50"
    ↓
PlannerAgent.plan(goal, currentPageContext)
    → Plan: ["Navigate to amazon.com",
             "Search 'bluetooth speaker'",
             "Filter by price under $50",
             "Extract top 3 results"]
    ↓
For each step:
    NavigatorAgent.executeStep(step, page)
    ├─ Read DOM via content_script
    ├─ LLM decides: click element X / type text Y
    └─ Execute via Extension APIs
    ↓
ValidatorAgent.validate(goal, results)
    → "Found 3 speakers matching criteria"
    ↓
Report to user in side panel chat
```

---

## 4. Architecture Diagram

```
Chrome Extension
  ├─ Side Panel (React UI)
  │   └─ Chat interface, real-time updates
  ├─ Background Service Worker
  │   ├─ PlannerAgent → LLM API (external)
  │   ├─ NavigatorAgent → LLM API (external)
  │   └─ ValidatorAgent → LLM API (external)
  └─ Content Script (per-tab)
      ├─ DOM access (direct, no CDP)
      ├─ Element interaction (Extension APIs)
      └─ Screenshot (chrome.tabs.captureVisibleTab)
```

---

## 5. Key Technical Decisions

**Why Chrome Extension instead of CDP?**
- CDP creates detectable automation signatures (WebSocket, `navigator.webdriver = true`)
- Extension APIs are indistinguishable from normal browser behaviour
- Extension can access the user's authenticated sessions, cookies, saved passwords

**Why three separate agent roles?**
- Each role can use a different (cheaper/faster) LLM
- Planner does complex reasoning → high-capability model
- Navigator does repetitive execution → cheaper model
- Clean separation allows independent optimisation

---

## 6. Limitations

- Chrome/Edge only (no Firefox, Safari)
- No programmatic/CLI API — interaction is always via the GUI side panel
- Cannot be integrated into automated pipelines or CI/CD
- Requires user to be present at the browser

**License:** Apache-2.0  
**Language:** TypeScript  
**Repository:** https://github.com/nanobrowser/nanobrowser  
**Snapshot date:** 2026-03-18
