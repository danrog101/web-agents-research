# Browserable

**Framework name:** Browserable  
**Repository:** https://github.com/browserable/browserable  
**Snapshot date:** 2026-03-18  

---

## Architecture

Browserable is an **open-source, self-hostable browser automation library** designed for AI agents. It exposes a task-oriented API where agents submit natural language instructions, and Browserable handles the full execution loop — browser lifecycle, page interaction, and result extraction.

The system is deployable via `npx browserable` and exposes:
- A **web admin dashboard** (localhost:2001) for configuring LLM and remote browser API keys.
- A **JavaScript/TypeScript SDK** (`browserable-js`) for programmatic task submission.
- A **REST API** for async task management.

Tasks are submitted with a natural language `task` string and an `agent` type (e.g., `BROWSER_AGENT`). Browserable handles planning, navigation, and extraction internally, returning structured results.

The architecture separates browser infrastructure (integrating with remote browser providers like **Hyperbrowser** or **Steel**) from the AI execution layer, making it possible to swap providers.

---

## Perception type

**DOM + LLM reasoning.** Browserable processes page content through DOM analysis fed to the configured LLM. No explicit details on accessibility-tree-only vs. full DOM are documented in the public README; the system appears to use a standard HTML/DOM extraction pipeline.

---

## LLM compatibility

Provider-agnostic via configurable API keys. Supports:
- **Gemini** (Google)
- **OpenAI** (GPT-4o, etc.)
- **Anthropic** (Claude)
- Other providers can be configured via compatible endpoints.

---

## Action interface

High-level natural language task interface. The user submits a task description; Browserable internally plans and executes actions (navigate, click, type, extract). The SDK exposes:
- `createTask({ task, agent })` — create and start a task
- `waitForRun(taskId)` — poll until completion
- `getTaskStatus(taskId)` — check progress

No fine-grained action control is exposed to the caller in the default API.

---

## Dependencies

- Node.js / npm for the `npx` CLI and JavaScript SDK
- Remote browser provider account (Hyperbrowser or Steel — free tier available)
- LLM provider API key (Gemini, OpenAI, or Claude)
- Self-hosted server (Docker or Node environment)

---

## Browser control method

**Remote browser via CDP** — Browserable integrates with third-party headless browser providers (Hyperbrowser, Steel) over their CDP or API interfaces. It does not manage its own Chrome instances by default, offloading browser infrastructure to the cloud provider. Local browser support appears limited in the current version.

---

## Strengths

- **90.4% reported score on WebVoyager** — highest of any framework in this comparison set (as of snapshot date).
- **Self-hostable** — full control over deployment, privacy, and data.
- **Simple API** — task submission is a single function call; no need to write step-by-step logic.
- **Multi-LLM support** out of the box.
- **Admin dashboard** for non-technical configuration.
- **Open source** (MIT/Apache license) with full transparency.

---

## Weaknesses

- **Remote browser dependency** — requires signing up for a third-party browser provider (Hyperbrowser or Steel) to get started; local-first operation is not the default.
- **Limited documentation** — the README covers basic setup but architectural internals are not well documented, making it harder to study or extend.
- **JavaScript/TypeScript only** for the SDK — Python users have no native SDK.
- **Early-stage project** — fewer community contributions, less battle-tested than browser-use.
- **No fine-grained action API** — limited to natural language task descriptions; unsuitable for deterministic step-by-step automation.
- The 90.4% WebVoyager figure is self-reported and should be independently verified.

---

## Typical use cases

- Rapid prototyping of browser automation workflows without infrastructure setup
- Building lightweight AI agents for data extraction and web research
- Integration into Node.js/TypeScript backend services
- Startups needing a deployable browser agent without managing Playwright infrastructure

---

## Additional notes for thesis research

**Benchmark claims:** Browserable's 90.4% WebVoyager score is notable but needs scrutiny — it is self-reported, and the exact test conditions (which tasks, which LLM, what timeout) are not detailed in the repo. This is methodologically important for your thesis on AI agent consistency and reproducibility.

**Reproducibility concern:** The reliance on remote browser providers means exact reproducibility requires access to the same provider infrastructure — a limitation worth noting in your methodology chapter.

**Comparison axis:** Browserable occupies a different layer of the stack from browser-use or Stagehand. It is closer to a "managed task runner" than a framework — analogous to the difference between a library and a platform. This distinction is relevant for your framework classification taxonomy.
