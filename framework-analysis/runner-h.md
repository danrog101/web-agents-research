# Runner H

**Framework name:** Runner H  
**Developer:** H Company (The H Company, Paris, France)  
**Repository:** Partially open — model weights on Hugging Face (`hcompany/hollow-one-7b-*`), product at https://runner.hcompany.ai  
**Snapshot date:** 2026-03-18  

---

## Architecture

Runner H is a **cloud-based AI web agent** and automation platform developed by H Company, a Paris-based AI lab that raised $220M. Unlike open-source frameworks, Runner H is primarily a hosted service, though H Company has open-sourced key **foundation model weights** under Apache-2.0.

The architecture is built around **Hollow One**, a family of in-house foundation models:
- **H-VLM (3B parameter Vision-Language Model)** — trained specifically for GUI understanding: extracting information from and interacting with interfaces and images.
- **H-LLM (2B parameter Language Model)** — for task planning and reasoning.
- **Hollow One Localizer** — a specialised model that maps natural language targets to pixel coordinates (`xy = localizer.find(screenshot, target)`).
- **Hollow One Policy** — a planning model that decomposes goals into steps.
- **Hollow One Validator** — verifies task completion.

The open-sourced `hollow-one-7b-policy` and `hollow-one-7b-localizer` models can be run locally:

```python
from hollow_one import Policy, Localizer
policy = Policy.from_pretrained("hcompany/hollow-one-7b-policy")
steps = policy.plan(goal="Extract 10 Top AI tools...")
```

The hosted product provides a **Studio** for designing web automation pipelines via natural language or visual demonstration, with a self-healing mechanism that adapts to UI changes.

H Company also offers:
- **Surfer H** — an open SDK wrapping the Hollow One models for programmatic automation.
- **Tester H** — a QA testing tool using the same browser agent technology.

---

## Perception type

**Vision-first (VLM + pixel-coordinate localisation).** H-VLM processes browser screenshots to understand page state. The Localizer model specifically maps natural language descriptions to screen coordinates. No explicit DOM parsing pipeline — the system works at the visual layer.

---

## LLM compatibility

**Proprietary in-house models only** in the hosted product. However:
- The open-sourced `hollow-one-7b-*` weights can be run locally.
- `surfer_h` SDK is compatible with these open-source weights.
- The architecture does not appear to support swapping in third-party LLMs in the current product.

---

## Action interface

Natural language task descriptions for the hosted product. The `surfer_h` SDK exposes:
- `agent.run(task, budget_usd)` — run a task with a cost budget
- The Localizer can be used standalone: `localizer.find(screenshot, target)` → `(x, y)`
- The Policy can plan steps: `policy.plan(goal)` → list of steps

---

## Dependencies

- Hugging Face Transformers (for running local Hollow One models)
- `hollow_one` Python package (for using open-sourced weights)
- `surfer_h` package (SDK)
- H Company account + API key (for hosted Runner H)
- GPU recommended for local model inference (7B parameter models)

---

## Browser control method

**Proprietary cloud browser infrastructure** for the hosted product. For local use via Surfer H, browser control would use standard Playwright/Selenium with the Hollow One models providing vision-based interaction decisions. The exact local browser integration is not fully documented in public sources.

---

## Strengths

- **State-of-the-art benchmark** — ~67% on WebVoyager (Runner H), outperforming OpenAI Operator (61%) on the same benchmark.
- **Specialised compact models** — 3B VLM and 2B LLM outperform much larger general-purpose models when powering browser agents, demonstrating that task-specific training > model size for this domain.
- **Open model weights** — Hollow One models are Apache-2.0, enabling research use and local deployment.
- **Self-healing automation** — adapts to UI changes without manual script updates.
- **Strong backing** — $220M raised; investors include Amazon, Accel, Eric Schmidt.
- **Reinforcement learning training** — models trained with RL for improved task completion.
- **WebClick dataset** — 30k-episode click annotation dataset open-sourced for research.

---

## Weaknesses

- **Primarily a commercial product** — full capabilities require a paid H Company account.
- **Limited documentation** for open-source components — the `hollow_one` and `surfer_h` libraries are not as well documented as browser-use or Stagehand.
- **Closed product code** — the Studio, pipeline designer, and cloud infrastructure are proprietary.
- **GPU requirement** for local model inference — not viable for CPU-only environments.
- **No third-party LLM support** in the current product — locked to Hollow One models.
- **Early-stage open-source community** — fewer contributors than established frameworks.

---

## Typical use cases

- Enterprise web automation pipelines (e.g., e-commerce order flows, financial onboarding)
- AI-powered QA testing via Tester H
- Research into specialised small VLMs for GUI automation
- Building custom browser agents using open Hollow One model weights
- Comparative benchmarking reference for the vision-first, trained-model approach

---

## Additional notes for thesis research

**Research significance:** Runner H is architecturally important because it demonstrates that **small, task-specifically trained models can outperform large general-purpose models** for web agent tasks. The 3B VLM outperforming GPT-4 variants on specific tasks is a key finding for efficiency-focused AI research.

**Open model access:** Unlike OpenAI Operator, H Company has open-sourced the Hollow One model weights. This makes it possible to reproduce and study their approach. The `WebClick` dataset (30k episodes) is also publicly available for research.

**Benchmark context:** Runner H's ~67% WebVoyager score is from a published benchmark run. As with other frameworks, verify the exact test conditions and date against your snapshot.

**Comparison axis:** Runner H vs. OpenAI Operator: both are vision-first, hosted products with proprietary models, but Runner H uses smaller specialised models while OpenAI uses a larger general model. This is an interesting efficiency vs. generality trade-off for your thesis.

**For cloning:** The product itself cannot be cloned, but `hollow_one` and `surfer_h` packages can be installed. Document which components are open vs. closed in your methodology section.
