# Browserable Research Report

## Executive Summary

Browserable is an **open-source, self-hostable browser automation library** with a simple task-oriented API. Users submit natural language task descriptions; Browserable handles the full browser lifecycle, action execution, and result extraction. It reported **90.4% on WebVoyager**, placing it 4th on the Steel leaderboard. The library integrates with cloud browser providers (Hyperbrowser, Steel) for infrastructure.

---

## 1. Architecture Overview

### Core Philosophy

**Task-runner design:** Browserable is not an agent framework you import into your own code — it is a self-hosted service that exposes an API. You create tasks; it executes them and returns results.

**Remote browser integration:** Rather than managing its own Chrome instances, Browserable integrates with cloud browser providers. This offloads CAPTCHA solving, proxy management, and session persistence to the provider.

### Package Structure

```
browserable/
├── server/
│   ├── src/
│   │   ├── agent/
│   │   │   └── BrowserAgent.ts        # Core agent implementation
│   │   ├── browser/
│   │   │   ├── HyperbrowserProvider.ts # Hyperbrowser integration
│   │   │   └── SteelProvider.ts        # Steel integration
│   │   ├── llm/
│   │   │   ├── GeminiProvider.ts
│   │   │   ├── OpenAIProvider.ts
│   │   │   └── AnthropicProvider.ts
│   │   ├── api/
│   │   │   └── TaskRouter.ts           # REST API endpoints
│   │   └── task/
│   │       ├── TaskRunner.ts
│   │       └── TaskQueue.ts
│   └── package.json
├── browserable-js/                     # JavaScript SDK
│   └── src/
│       └── Browserable.ts
├── docker-compose.yml                  # Self-hosting config
└── README.md
```

---

## 2. Core Components Deep Dive

### 2.1 JavaScript/TypeScript SDK

```typescript
import { Browserable } from 'browserable-js';

// Initialize SDK with self-hosted or cloud endpoint
const browserable = new Browserable({
    apiKey: 'your-api-key',
    baseURL: 'http://localhost:2001',  // self-hosted
    // or cloud endpoint
});

// Create and run a task
async function runTask() {
    const createResult = await browserable.createTask({
        task: 'Find the top trending GitHub repos of the day.',
        agent: 'BROWSER_AGENT',
    });
    
    const taskId = createResult.data.id;
    
    // Poll until complete
    const result = await browserable.waitForRun(taskId);
    console.log('Results:', result.data);
}
```

### 2.2 Task API Endpoints

```bash
# Create task
POST /api/tasks
{
    "task": "Search arxiv for latest papers on LLM agents",
    "agent": "BROWSER_AGENT"
}

# Get task status
GET /api/tasks/{taskId}

# Get task result
GET /api/tasks/{taskId}/result
```

### 2.3 Self-Hosting Setup

```bash
# One-command setup
npx browserable

# Then configure at localhost:2001:
# 1. Set LLM provider (Gemini / OpenAI / Claude)
# 2. Set remote browser provider (Hyperbrowser or Steel)
# 3. Submit tasks via SDK or web UI

# Environment variables
GEMINI_API_KEY=...
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
HYPERBROWSER_API_KEY=...  # or STEEL_API_KEY=...
```

### 2.4 Example Tasks

```
"On amazon.com search for a yoga mat at least 6mm thick, non-slip, eco-friendly, and under $50"
"On arxiv.org locate the latest paper in 'Nonlinear Sciences - Chaotic Dynamics', summarise the abstract"
"On coursera.com find a beginner-level 3D printing course lasting 1-3 months from a renowned university"
```

---

## 3. Architecture Diagram

```
User/Application
    ↓
browserable-js SDK (HTTP)
    ↓
Browserable Server (self-hosted, Node.js)
    ├─ Task Queue
    ├─ BrowserAgent (LLM reasoning loop)
    │   ├─ Gemini / OpenAI / Anthropic
    │   └─ Action parsing + execution
    └─ Browser Provider
        ├─ Hyperbrowser (cloud Chromium)
        └─ Steel (cloud Chromium)
```

---

## 4. Key Technical Decisions

**Why remote browser providers?**
Cloud browsers handle CAPTCHA, IP rotation, and anti-bot measures that are expensive to build in-house. Browserable focuses on the AI reasoning layer and delegates infrastructure.

**Why task-runner design instead of a library?**
Lower barrier to entry — no need to write an agent loop. Submit a task string, get results.

---

## 5. Limitations

- Requires a remote browser provider account (Hyperbrowser or Steel)
- TypeScript/JavaScript only — no Python SDK
- Less documentation than browser-use or Stagehand
- Self-reported 90.4% WebVoyager — test conditions not fully disclosed

**WebVoyager score:** 90.4% (self-reported, Steel leaderboard)  
**License:** MIT/Apache  
**Language:** TypeScript  
**Repository:** https://github.com/browserable/browserable  
**Snapshot date:** 2026-03-18
