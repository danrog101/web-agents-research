# Nanobrowser

**Framework name:** Nanobrowser  
**Repository:** https://github.com/nanobrowser/nanobrowser  
**Snapshot date:** 2026-03-18  

---

## Architecture

Nanobrowser is an **open-source Chrome Extension** that provides AI-powered web automation running entirely inside the user's local browser. It is architecturally distinct from server-side or Python-based frameworks: all execution happens within the browser process itself, with no external backend required.

The extension implements a **multi-agent system** with three specialised roles:
1. **Planner Agent** — receives the user's natural language goal, decomposes it into a sequence of sub-tasks, and coordinates the overall strategy. Performs self-correction when obstacles are encountered.
2. **Navigator Agent** — executes individual navigation and interaction steps on the live page (click, type, scroll, extract).
3. **Validator Agent** — verifies whether sub-tasks and the overall goal have been completed successfully.

Users interact through a **side panel chat interface** embedded in Chrome. Tasks are submitted conversationally; the extension shows real-time status updates and allows follow-up questions.

Each agent role can be assigned a different LLM model — e.g., a cheaper model for Navigator, a more capable model for Planner — allowing cost/performance trade-offs.

The extension uses **Chrome Extension APIs** (not CDP/WebDriver) for browser access. This has privacy and detection-avoidance implications: the automation is executed in the same browser session as the user, with direct DOM access through content scripts.

---

## Perception type

**DOM / page content via Chrome Extension content scripts.** The agents read page content through the browser's built-in extension APIs. No raw screenshot-based vision is used in the default pipeline; perception is text/DOM-based.

---

## LLM compatibility

Flexible, user-configured. Supported providers (as of v0.2.x):
- OpenAI (GPT-4o, GPT-4o-mini, etc.)
- Anthropic (Claude)
- Google (Gemini)
- Groq
- Cerebras
- More being added

Users bring their own API keys. Different models can be assigned to Planner vs. Navigator vs. Validator for cost optimisation.

---

## Action interface

The user submits natural language instructions via the side panel chat. Internally, the Navigator Agent executes atomic browser actions: click, type, scroll, navigate, extract. No programmatic action API is exposed — the interface is conversational only.

---

## Dependencies

- Chrome or Edge browser (Chromium-based; Firefox/Safari not supported)
- LLM provider API keys (user-supplied)
- No Python, no Node.js backend required — runs purely as a browser extension
- Extension installed from Chrome Web Store or manually loaded from ZIP

---

## Browser control method

**Chrome Extension APIs** — content scripts, `chrome.tabs`, `chrome.scripting`, `chrome.storage`, etc. This is fundamentally different from CDP-based approaches: there is no debugging WebSocket, no external process, and no detectable automation flag. The automation runs within the normal browser profile, inheriting all cookies, sessions, and login state.

---

## Strengths

- **Privacy-first, local execution** — data never leaves the user's machine; no cloud backend.
- **Free to use** — no subscription fees; pay only for LLM API usage.
- **Uses normal browser session** — accesses authenticated sites, cookies, and sessions without additional auth configuration.
- **Not detectable as automation** — uses Chrome Extension APIs instead of CDP, avoiding WebDriver fingerprinting.
- **Multi-agent role separation** — cleaner task management with explicit Planner/Navigator/Validator split.
- **Per-agent model assignment** — cost/performance optimisation by using cheaper models for simpler roles.
- **Conversational follow-up** — users can ask contextual follow-up questions about completed tasks.
- **No infrastructure setup** — install and go; no Docker, no Python environment.

---

## Weaknesses

- **Chrome/Edge only** — no cross-browser support.
- **No programmatic API** — cannot be integrated into automated pipelines or CI/CD; interaction is always manual/conversational.
- **Limited to the user's active browser** — not suitable for server-side, headless, or large-scale automation.
- **No Python/JS SDK** — developers cannot build workflows programmatically on top of Nanobrowser.
- **Shadow DOM support** in progress; some modern SPAs may behave unexpectedly.
- **Session replay / workflow recording** not yet available (on roadmap).
- **Community smaller than browser-use** — fewer integrations and less battle-testing.

---

## Typical use cases

- Personal productivity automation (research, form filling, data gathering)
- Privacy-sensitive automation requiring access to authenticated sessions
- Non-technical users wanting AI help with browser tasks without coding
- Quick one-off automation tasks without infrastructure investment
- Exploratory automation in environments where CDP-based tools are blocked

---

## Additional notes for thesis research

**Architectural classification:** Nanobrowser is the only framework in this comparison set that is implemented as a browser extension rather than a Python library, TypeScript library, or standalone server. This makes it uniquely suited for studying the *consumer/end-user* segment of the web agent ecosystem, as opposed to developer-facing frameworks.

**Detection avoidance:** The Chrome Extension API approach is architecturally interesting from a security/ethics perspective — it is indistinguishable from normal human browsing at the protocol level. This is worth discussing in a section on anti-bot measures and ethical considerations.

**Multi-agent specialisation:** The explicit Planner/Navigator/Validator role decomposition is simpler than browser-use's unified Agent but provides cleaner separation of concerns. Compare this to Agent-E's Planner/Navigator split — both converge on hierarchical decomposition as the right approach for complex tasks.

**Reproducibility note:** Because Nanobrowser runs in the user's live browser session, exact reproducibility requires the same browser profile, login state, and page state — making it harder to reproduce results in a controlled academic setting. Note this limitation in your methodology.
