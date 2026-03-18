# Magnitude

**Framework name:** Magnitude  
**Repository:** https://github.com/magnitudedev/browser-agent (browser agent) / https://github.com/magnitudedev/magnitude (coding agent)  
**Snapshot date:** 2026-03-18  

---

## Architecture

Magnitude is a **vision-first, open-source browser automation framework** built in TypeScript. It takes a fundamentally different approach from DOM-based frameworks: rather than processing the HTML/DOM tree and assigning element indices, Magnitude sends **raw screenshots** to a visually grounded large model (VLM) that reasons about the page purely from what it sees — the same way a human would.

The core execution loop:

1. Capture a screenshot of the current browser state.
2. Feed the screenshot (optionally with a small CoT reasoning injection) to the configured VLM.
3. The model outputs reasoning + an action sequence (mouse coordinates, keyboard input, etc.).
4. Actions are executed via Playwright (mouse/keyboard at pixel coordinates).
5. Repeat until the goal is achieved.

The framework exposes three primary API methods:
- `agent.act(instruction, { data })` — execute a high-level or granular action.
- `agent.extract(prompt, zodSchema)` — extract structured data from the current page, validated against a Zod schema.
- `agent.verify(assertion)` — assert a visual or content condition (used in test mode).

Magnitude also includes a **test runner** (`magnitude-test`) for writing AI-native E2E tests, positioning it as both an automation framework and a testing tool. It uses a **planner → executor** split: the planner builds a reusable action plan; subsequent runs use only the executor against that cached plan, making repeated runs cheap and deterministic.

A separate **coding agent** product (`magnitudedev/magnitude`) exists alongside the browser agent — it is a multi-agent coding assistant and is architecturally distinct.

---

## Perception type

**Vision-first (pure screenshot / grounded VLM).** No DOM parsing, no element indices, no accessibility tree. The model interacts with the page based entirely on visual understanding of screenshots. This approach generalises better to visually complex or canvas-based pages (e.g., price sliders, custom widgets) where DOM-based methods struggle.

Key design rationale from the team: *"Most browser agents draw numbered boxes around page elements — doesn't generalise well due to complex modern sites. Future-proof architecture for desktop apps, VMs, etc."*

---

## LLM compatibility

Requires a **large visually grounded model (VLM)**. Recommended / tested:
- **Claude Sonnet 4** (recommended for best performance)
- **Qwen-2.5VL 72B** (open-source alternative)

Any OpenAI-compatible API endpoint with vision capability can be configured. Standard text-only LLMs are NOT compatible — vision is a hard requirement.

---

## Action interface

High-level natural language + low-level coordinate-based actions via `act()`. The model interprets the instruction and emits mouse/keyboard primitives:
- Mouse: click at (x, y), drag, hover
- Keyboard: type text, key press, key combo
- Navigation: go to URL
- Extraction: structured data via `extract()` with Zod schema
- Verification: visual assertion via `verify()`

No explicit numbered-element-index system.

---

## Dependencies

- Node.js / TypeScript
- `playwright` (Chromium browser control — Playwright/Patchright for anti-bot patching)
- A VLM provider (Anthropic Claude API or Qwen)
- `zod` (schema validation for extraction)
- `magnitude-test` CLI for test-runner mode

---

## Browser control method

**Playwright** (with optional **Patchright** for anti-bot patching). Actions are translated from VLM-produced pixel coordinates into Playwright mouse and keyboard calls. Patchright patches are applied to avoid CAPTCHA / Cloudflare detection in benchmark runs.

---

## Strengths

- **State-of-the-art benchmark performance** — 93.9% on WebVoyager (as of mid-2025), the highest documented score for any open-source browser agent, beating OpenAI Operator, browser-use, and others.
- **Vision-first generalisation** — handles visually complex elements (sliders, canvas, games) that stump DOM-based methods.
- **Future-proof** — the same approach extends to desktop GUI automation and virtual machine control.
- **Flexible abstraction levels** — both high-level (`act('add to cart')`) and low-level (`act('drag element to top of column')`) instructions work.
- **Built-in test runner** for AI-native E2E testing with visual assertions.
- **Caching system** — planner caches plans; executor replays them without LLM cost.
- **Transparent methodology** — WebVoyager reproduction repo published with full details.

---

## Weaknesses

- **VLM dependency** — requires a capable visual model; text-only models cannot be used. Claude Sonnet 4 or Qwen-2.5VL 72B are the only validated options at time of writing.
- **Higher cost per step** — screenshot + VLM inference is more expensive than text-DOM approaches.
- **Slower execution** — VLM inference + screenshot round-trips add latency vs. CDP-native DOM extraction.
- **"Reasoning loop" failure mode** — the CoT chain-of-thought can get stuck repeating the same reasoning without making progress.
- **TypeScript-only** — no Python SDK; may not integrate cleanly into Python-centric ML workflows.
- **Smaller community** than browser-use — fewer contributors and less documentation.

---

## Typical use cases

- AI-native end-to-end web testing (replacing Cypress/Playwright for test authoring)
- Automation of visually complex sites where DOM-based methods fail (custom sliders, canvas apps, games)
- Cross-platform automation research (web → desktop → VM)
- Benchmarking and research on vision-based browser agent performance
- Building production automations where layout changes should not break workflows

---

## Additional notes for thesis research

**Critical benchmark context:** Magnitude's 93.9% WebVoyager score is particularly significant for your thesis because it was independently reproduced and the methodology is fully published (`magnitudedev/webvoyager` repo). The team explicitly addresses methodological flaws in the original WebVoyager benchmark (outdated tasks, live site drift). This is excellent material for discussing benchmark validity in your methodology chapter.

**DOM vs. vision debate:** Magnitude directly positions itself against DOM-based approaches. Their paper on WebVoyager SOTA cites concrete failure cases where DOM indexing breaks (price sliders, mini-games) — useful examples for your thesis analysis.

**Reproducibility:** The WebVoyager benchmark repo includes the exact agent prompt, reasoning structure, and task patches used. This makes it one of the most reproducible results in this comparison set.

**Comparison axis:** Vision-first (Magnitude) vs. DOM-first (browser-use, Agent-E) is the central architectural tension in the web agent field. Magnitude's results suggest that as VLMs improve, the vision-first approach may become dominant.
