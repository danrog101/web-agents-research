# Skyvern

**Framework name:** Skyvern  
**Repository:** https://github.com/Skyvern-AI/skyvern  
**Snapshot date:** 2026-03-18  

---

## Architecture

Skyvern is an open-source browser automation platform that combines **computer vision and LLMs** to understand and interact with web pages without relying on pre-defined XPath or CSS selectors. It is available as a self-hosted open-source tool and as Skyvern Cloud.

The core architecture is a **multi-agent swarm**:
- **Navigator Agent** — decides which element to interact with next based on visual page analysis
- **Data Extraction Agent** — reads structured data from pages (tables, forms, text)
- **Validation Agent** — confirms whether an action succeeded or the task is complete
- **Form Filling Agent** — handles multi-step forms including password fields and 2FA

Execution flow: user provides a `url` + `prompt` → Skyvern takes a screenshot → LLM with vision analyses the screenshot + page structure → plans next action → executes via Playwright → screenshots again → repeats.

Beyond single tasks, Skyvern supports **Workflows** — chaining multiple tasks into a pipeline (e.g., navigate → filter → extract list → iterate over each item → download). Workflows can include branching logic and loops.

Skyvern recently introduced an **"explore → replay" pattern**: the first run uses full LLM inference to discover the workflow; subsequent runs use cached Playwright code and only fall back to LLM when the site changes. This makes repeated runs 2.7x cheaper and 2.3x faster.

The SDK is Playwright-compatible — developers can mix standard Playwright calls with Skyvern's AI-powered methods:
```python
await page.act("Click the login button")          # AI-powered
await page.goto("https://example.com")            # standard Playwright
result = await page.extract("Get product price")  # AI-powered
```

---

## Perception type

**Vision + LLM (Computer Vision primary).** Skyvern takes browser screenshots and analyses them with a vision-capable LLM. It does not rely primarily on DOM parsing or element selectors — the agent sees the page as an image and reasons about what to click based on visual understanding. This makes it robust to layout changes and suitable for sites where the DOM is unhelpful (heavy JS rendering, iframes, canvas elements).

---

## LLM compatibility

Supports multiple vision-capable LLM providers. Recommended as of 2025-2026:
- **Gemini 2.5 Pro / Flash** (recommended defaults)
- **Claude Sonnet / Opus** (Anthropic)
- **GPT-4o** (OpenAI)
- **Ollama** local vision models (e.g., qwen3-vl, llava) — set `OLLAMA_SUPPORTS_VISION=true`
- AWS Bedrock models

A vision-capable model is required. Text-only models will not work correctly.

---

## Action interface

High-level natural language task/workflow descriptions plus a Playwright-compatible SDK:
- `page.act(instruction)` — execute a natural language action
- `page.extract(prompt, schema)` — extract structured data with optional JSON schema
- `page.validate(assertion)` — check a page condition, returns bool
- `page.prompt(message)` — send an arbitrary prompt to the LLM about the page

Tasks and Workflows are also configurable via a web UI (localhost:8080).

---

## Dependencies

- Python (uv recommended for environment management)
- `playwright` (browser control layer)
- Vision LLM provider API key
- PostgreSQL (for task/workflow persistence)
- Redis (optional, for task queue)
- Docker Compose (recommended for full local setup)

---

## Browser control method

**Playwright** — Skyvern uses Playwright as the underlying browser engine. The AI layer decides what actions to take; Playwright executes them. The explore→replay pattern generates and caches actual Playwright code for deterministic re-runs.

---

## Strengths

- **Works on unseen websites** — no custom code needed per site; vision + LLM generalises across any layout
- **Resistant to UI changes** — when a site redesigns, Skyvern adapts without script updates
- **Strong on WRITE tasks** — forms, logins, downloads; best performer on RPA-adjacent workflows
- **Workflow builder** — chains multiple tasks, supports branching and loops
- **Explore → replay pattern** — dramatic cost/speed reduction on repeated runs
- **Multi-LLM support** — swap providers without code changes
- **Active development** — YC-backed, rapid release velocity
- **Playwright-compatible SDK** — developers can mix AI and manual code

---

## Weaknesses

- **Vision model cost** — every step requires a screenshot + vision model inference; more expensive per step than DOM-based approaches
- **Slower per step** — screenshot round-trips add latency (2-3 seconds per action typical)
- **PostgreSQL + Redis setup** — full local deployment is more complex than browser-use's `pip install`
- **Accuracy on READ tasks** — extraction of complex structured data sometimes requires careful prompt engineering
- **~64% WebVoyager** on the Steel leaderboard (one measurement point; results vary by test set and LLM)
- **Vision coordinate precision** — can occasionally miss-click on small or closely spaced elements

---

## Typical use cases

- Invoice downloading across hundreds of vendor portals
- Job application automation across multiple sites
- Insurance quote generation on forms-heavy sites
- Government form automation (multi-step, layout changes often)
- Competitor price monitoring
- Any RPA workflow where the target site changes regularly

---

## Additional notes for thesis research

**Key differentiator from browser-use:** Skyvern is vision-first and production-RPA-focused; browser-use is DOM-first and developer/framework-focused. For your thesis, this is the central DOM vs. Vision comparison — Skyvern is the clearest representative of the vision camp among open-source tools.

**Explore → replay pattern** is architecturally interesting and worth a paragraph in your thesis: it combines the reliability of LLM-driven exploration with the speed/cost of deterministic replay, addressing one of the core weaknesses of vision agents (cost and latency).

**Benchmark note:** Skyvern claims to be "best performing on WRITE tasks" in their own benchmark comparisons. Their filtered WebVoyager dataset (50 tasks) is what Magnitude used to achieve 93.9%. Verify which dataset is used when comparing numbers.

**For your controlled environment:** Skyvern would work well on your application, but the PostgreSQL + Redis setup adds complexity. For a thesis implementation, browser-use is simpler to set up and sufficient to demonstrate the concepts.
