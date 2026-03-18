# WebVoyager

**Framework name:** WebVoyager  
**Repository:** https://github.com/MinorJerry/WebVoyager  
**Paper:** https://arxiv.org/abs/2401.13919  
**Snapshot date:** 2026-03-18  

---

## Architecture

WebVoyager is an **academic research web agent** developed by Zhejiang University, published in early 2024. It is both a framework/agent implementation and the source of the **WebVoyager benchmark** — a dataset of 643 tasks across 15 real websites that has become the de facto standard for measuring web agent performance.

The agent implementation is built around a **multimodal perception loop**:
1. Navigate to the starting URL for the task
2. Take a screenshot of the current page
3. Annotate interactive elements on the screenshot with **numbered bounding boxes** ("Set-of-Marks" — SoM) — each clickable element gets a visible number overlaid on the screenshot
4. Feed the annotated screenshot + task description to a vision LLM (GPT-4V)
5. LLM outputs the next action referencing elements by number (e.g., "CLICK [7]", "TYPE [3] search text")
6. Execute the action via Selenium
7. Repeat until the agent outputs "ANSWER: [result]"

The **Set-of-Marks approach** was the original innovation: by annotating the screenshot with numbered boxes, the agent can reference specific elements without parsing raw HTML — bridging DOM and vision approaches.

The **benchmark (WebVoyager dataset)** is separate from the agent implementation and has become far more important than the agent code itself. It contains 643 manually validated tasks on 15 websites (e.g., Google, Amazon, BBC, GitHub, ArXiv, Cambridge Dictionary) across functional types: information retrieval, navigation, filtering, form interaction.

Evaluation uses **GPT-4V as an automated judge** — it compares the agent's final answer against a reference answer and a screenshot to determine success. This semi-automated evaluation protocol has also been widely adopted.

---

## Perception type

**Multimodal — screenshot + Set-of-Marks (numbered bounding boxes).** The original approach overlays numbered boxes on screenshots so the LLM can reference elements by number rather than coordinates or DOM selectors. Later work (including Magnitude's SOTA run) showed that pure vision without SoM can outperform the original approach when using modern grounded VLMs.

Text-only variant is also available (`--text_only` flag) which uses accessibility tree input instead of screenshots — useful for comparing DOM vs. vision approaches.

---

## LLM compatibility

Originally designed for **GPT-4V** (GPT-4 with vision preview). The codebase supports:
- `gpt-4-vision-preview` (default multimodal)
- `gpt-4-1106-preview` (text-only variant)
- Any OpenAI-compatible API endpoint

The benchmark evaluation uses GPT-4V as the judge regardless of which agent model is tested.

---

## Action interface

Simple action space output from the LLM, parsed from text:
- `CLICK [element_number]`
- `TYPE [element_number] [text]`
- `SCROLL [WINDOW or element_number] [up/down]`
- `GOBACK`
- `GOOGLE`
- `ANSWER [final_answer]`

---

## Dependencies

- Python 3.10
- `selenium` (browser control — uses Selenium, not Playwright)
- Chrome + ChromeDriver
- `openai` Python SDK
- API key for GPT-4V (or compatible model)

---

## Browser control method

**Selenium** — notably different from most modern frameworks which use Playwright or CDP. Selenium was the standard when WebVoyager was developed in 2023/early 2024. The benchmark environment uses Selenium to navigate real websites and capture screenshots.

---

## Strengths

- **The benchmark is the primary contribution** — 643 tasks across 15 real websites is one of the most widely used evaluation suites in the field; virtually every web agent paper and product reports WebVoyager results
- **Set-of-Marks innovation** — the numbered bounding box approach was a meaningful contribution at the time; influenced many subsequent frameworks
- **Accessible implementation** — clean, minimal Python codebase, easy to understand and modify
- **Automated evaluation protocol** — GPT-4V as judge enabled scalable benchmark evaluation without human annotators for every run
- **Academic credibility** — peer-reviewed, widely cited, Zhejiang University origin
- **Text-only variant** — enables direct comparison of DOM vs. vision approaches on the same task set

---

## Weaknesses

- **The agent itself is outdated** — modern frameworks (browser-use, Magnitude, Stagehand) significantly outperform the original WebVoyager agent; it is primarily useful as a baseline and benchmark source, not as a production agent
- **Selenium** — slower and less capable than Playwright or CDP for modern web automation
- **Benchmark limitations** — tasks are time-sensitive (booking/flights tasks go stale), tasks use live sites that change over time, some tasks have ambiguous success criteria
- **SoM can fail on complex layouts** — numbered box annotation breaks on sites with heavily overlapping or dynamically generated elements
- **No workflow/multi-task support** — each task is independent; no chaining
- **No custom action registration** — fixed action space

---

## Typical use cases

- **Benchmark baseline** — reference implementation for evaluating new agents on the WebVoyager dataset
- **Academic research** — studying multimodal web navigation, comparing DOM vs. vision approaches
- **Teaching/learning** — clean, simple codebase for understanding how LLM-driven web agents work at a fundamental level
- **Reproducing benchmark results** — validating or comparing against published numbers

---

## Additional notes for thesis research

**Benchmark vs. framework distinction:** For your thesis, WebVoyager serves two separate roles — (1) as a benchmark to cite when comparing agent performance numbers, and (2) as an early agent design to describe in the historical context of the field. Make this distinction clear in chapter 2.5 (Benchmarks) vs. chapter 3 (Frameworks).

**Benchmark reliability:** The WebVoyager benchmark uses live websites, meaning results drift over time as pages change. Magnitude explicitly patched tasks and re-ran on a filtered 50-task subset (Skyvern's filtered version) to get more reliable results. When citing benchmark numbers in your thesis, always note which version of the benchmark was used and when the run was conducted.

**Historical importance:** WebVoyager (January 2024) was the first major work to demonstrate end-to-end web task completion on real websites using a multimodal LLM, achieving ~59% success rate. This was a landmark result at the time. Situate it in your thesis as the turning point where web agents moved from synthetic benchmarks (MiniWoB, WebArena) to real-world websites.

**The Set-of-Marks → pure vision evolution:** The progression from WebVoyager's SoM approach → browser-use's DOM indexing → Magnitude's pure vision (93.9%) is a compelling narrative arc for chapter 2.3 (DOM vs. Vision methods) and chapter 3's comparative analysis. Each step represents a design trade-off worth discussing.

**For reproducibility:** The benchmark data (`data/WebVoyager_data.jsonl`) can be cloned and used independently of the agent code. If you run your own agent on a subset of WebVoyager tasks, you have a direct comparison point to all published results.
