# Agent-E

**Framework name:** Agent-E  
**Repository:** https://github.com/EmergenceAI/Agent-E  
**Snapshot date:** 2026-03-18  

---

## Architecture

Agent-E is a hierarchical multi-agent web automation framework built on top of Microsoft AutoGen. Its architecture splits responsibility into two tiers:

1. **Planner Agent** — decomposes high-level user tasks into ordered sub-tasks, manages overall workflow logic, and remains isolated from raw page noise.
2. **Browser Navigation Agent** — executes each sub-task against a real browser, selecting the appropriate DOM representation and reporting success/failure back to the Planner.

The key architectural innovation is **DOM Distillation**: rather than feeding the full HTML to the LLM, Agent-E trims the DOM to task-relevant elements and serialises it as a concise JSON snapshot. Three distillation modes are available (text-only, input-fields-only, full content), and the Navigation Agent chooses the most suitable one at runtime. After every action the agent observes the resulting DOM delta (*Change Observation* / Reflexion-style feedback), correcting course without an extra LLM call.

Skills (Python functions) are registered into AutoGen's function-calling registry, allowing the LLM to invoke them by name. The architecture is designed to be extensible: new skills can be added by registering them in the registry.

---

## Perception type

**Text-DOM (accessibility tree) + optional screenshot.** Agent-E primarily uses the Accessibility Tree rather than raw HTML — closer in semantics to what a screen reader (and an automation agent) actually needs. Screenshots are available as a secondary channel but the default interaction flow is text-DOM based.

---

## LLM compatibility

Any model that supports **function calling / structured output**. The original paper used GPT-4o; the codebase is AutoGen-based so any model wired into AutoGen works. Recommended minimum: 32k context window, function-calling support.

---

## Action interface

Skill-based via AutoGen function calling. Core skill categories:
- Navigation (`navigate`, `go_back`, `go_forward`, `refresh`)
- Element interaction (`click`, `input_text`, `send_keys`, `hover`)
- DOM sensing (`get_dom_with_content_type`, `get_focused_element`)
- Page utilities (`scroll`, `wait`, `get_url`, `save_to_memory`)
- Task control (`done`)

---

## Dependencies

- Python 3.11+
- `pyautogen` (Microsoft AutoGen)
- `playwright` (browser control)
- `openai` / compatible LLM SDK
- `pytest-httpserver` (testing)

---

## Browser control method

**Playwright** — standard Chromium-based headless or headed browser. The browser navigation agent calls Playwright commands after the LLM selects a skill. No CDP-level direct access; Playwright's high-level API is the interaction layer.

---

## Strengths

- **Hierarchical architecture** insulates planning from per-page noise, improving coherence on multi-step tasks.
- **Flexible DOM distillation** dramatically reduces token usage; the agent picks the right compression mode per task type.
- **Change observation loop** provides lightweight grounding feedback without additional LLM calls.
- **Achieved 73.2% on WebVoyager** benchmark at publication time — 20% above the prior text-only SOTA and 16% above the multimodal SOTA.
- **Extensible skill registry** makes adding domain-specific actions straightforward.
- **Research-backed** with an arXiv paper (2407.13032) synthesising 8 general agentic design principles.

---

## Weaknesses

- **Dependency on AutoGen** means the architecture inherits AutoGen's verbosity and token overhead for function-calling descriptions.
- **Shadow DOM** support is partial/in-progress; some modern SPAs can confuse distillation.
- **Google Suite / canvas elements** not yet supported (noted as a roadmap item).
- **History / context trimming** not fully implemented — long tasks can cause token bloat.
- **Development velocity lower than browser-use** — the repo is research-grade rather than product-grade.
- Change observation adds latency per step compared to purely reactive designs.

---

## Typical use cases

- Autonomous end-to-end web research (finding products, comparing prices)
- Form automation and data entry across arbitrary websites
- Multi-step enterprise workflows (e.g., CRM interaction, invoice retrieval)
- Academic baseline for evaluating hierarchical agent designs
- Building-block for research into agentic design principles

---

## Additional notes for thesis research

**Comparison axis vs. browser-use:** Agent-E is accessibility-tree-DOM-first and AutoGen-based (Python); browser-use is also DOM-first but uses CDP directly and is built from scratch with an async Python event loop. Agent-E's hierarchical split (Planner ↔ Navigator) is more explicit than browser-use's single-agent loop, which makes it easier to study task decomposition strategies.

**Reproducibility:** WebVoyager benchmark results are publicly documented and the evaluation harness is included in the repo. The benchmark uses live websites, so scores vary over time as pages change — important to note in methodology.

**Methodological note for thesis:** When cloning, remove `.git` and record snapshot date. The DOM distillation implementation (`dom_service.py`) is the most analytically interesting component for studying perception strategies.

**Relevant paper:** Abuelsaad et al., "Agent-E: From Autonomous Web Navigation to Foundational Design Principles in Agentic Systems," arXiv 2407.13032, July 2024.
