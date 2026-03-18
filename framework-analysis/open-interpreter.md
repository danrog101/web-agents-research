# Open Interpreter

**Framework name:** Open Interpreter  
**Repository:** https://github.com/openinterpreter/open-interpreter  
**Snapshot date:** 2026-03-18  

---

## Architecture

Open Interpreter is an **open-source natural language interface for computers** — not a web-only agent framework. It allows LLMs to execute code (Python, JavaScript, Shell, and more) locally on the user's machine, and includes browser control as one capability among many.

The architecture is centred around a **code execution loop**:
1. User provides a natural language instruction.
2. LLM generates code (Python, JS, Shell, etc.) to fulfil the instruction.
3. Open Interpreter executes the code in a local sandboxed environment.
4. Output/errors are fed back to the LLM, which continues or corrects.
5. Loop until the task is complete or the user intervenes.

Key components:
- **`interpreter` Python module** — the main interface, usable as a Python library or CLI (`interpreter` command).
- **LiteLLM integration** — universal LLM provider abstraction supporting 100+ models.
- **Code interpreter runtimes** — Python, JavaScript/Node, Bash, R, and more.
- **Browser computer use** — the agent can launch and control browsers as part of multi-step tasks (via Selenium or Playwright, depending on version).
- **GUI / vision mode** — optional vision capability for interpreting screenshots.
- **Computer use protocol** — later versions support controlling the GUI (mouse, keyboard) beyond browser automation.

Open Interpreter is positioned as a local, open-source alternative to OpenAI's Code Interpreter / ChatGPT Operator, running on the user's machine with full internet access and no runtime limits.

---

## Perception type

**Code-execution-centric + optional vision.** The primary perception modality is through code: the agent writes Python or JS to read files, query APIs, or manipulate browser DOM programmatically. Vision (screenshot analysis) is available as an optional module. Unlike other frameworks in this comparison, Open Interpreter does not have a specialised DOM serialisation or element-indexing layer — it accesses the web through code it writes itself.

---

## LLM compatibility

Virtually any LLM via **LiteLLM**:
- OpenAI (GPT-4o, GPT-4-turbo, etc.)
- Anthropic (Claude family)
- Google (Gemini)
- Local models via Ollama, LM Studio, or any OpenAI-compatible API
- Azure OpenAI
- AWS Bedrock
- Hugging Face hosted models

100k+ models via LiteLLM routing. Local mode available with offline operation.

---

## Action interface

The LLM generates executable code as its "action". There is no fixed set of browser actions — the agent writes whatever code it deems appropriate. For browser automation this typically means writing Selenium, Playwright, or `requests` code. In GUI control mode, the agent can also issue mouse and keyboard events.

---

## Dependencies

- Python 3.10 or 3.11
- `litellm` (LLM routing)
- Various optional runtimes: Node.js (JavaScript), R, etc.
- Selenium or Playwright (for browser tasks, installed separately)
- LLM provider API key (or local model server)

---

## Browser control method

**Code-generated automation** — Open Interpreter writes Python/Selenium/Playwright code to interact with the web, rather than providing a built-in browser automation layer. In computer-use mode, it can also control the GUI at the OS level (mouse, keyboard). There is no dedicated CDP integration or DOM serialisation pipeline.

---

## Strengths

- **Universal computer control** — not limited to web browsers; can run code, manipulate files, call APIs, control GUIs.
- **Full internet access** — no restrictions on packages or network calls (unlike hosted sandboxes).
- **Massive LLM compatibility** via LiteLLM (100+ models).
- **Local privacy** — runs entirely on the user's machine; no data sent to a third-party automation server.
- **Highly flexible** — the agent can write any code, making it adaptable to any task.
- **50k+ GitHub stars** — one of the most popular open-source AI agents.
- **Active community** with 100+ contributors.
- **Local model support** — can run fully offline with Ollama.

---

## Weaknesses

- **Not a specialised web agent** — browser automation is one capability among many, not the primary focus. Purpose-built web agent frameworks (browser-use, Skyvern) will outperform it on web-specific benchmarks.
- **No DOM distillation or element indexing** — the LLM must write its own element selection code, which can be brittle.
- **Security risk** — executing LLM-generated code locally is inherently risky; code can delete files, exfiltrate data, etc.
- **Inconsistency** — code generation is non-deterministic; the same prompt may produce different code on different runs, making reproducibility challenging.
- **Requires user confirmation flow** for destructive actions (configurable), which adds friction.
- **No built-in benchmarking** on web-specific tasks (WebVoyager, WebArena).

---

## Typical use cases

- General-purpose computer automation (not just web)
- Data analysis and visualisation via code generation
- File management, PDF creation, media processing
- Multi-modal tasks combining web browsing with local computation
- Power users wanting a local, open-source alternative to ChatGPT's code interpreter
- Research into LLM code-generation capabilities and safety

---

## Additional notes for thesis research

**Classification note:** Open Interpreter is architecturally different from all other frameworks in this comparison. It is a *general-purpose computer agent* that happens to support browser automation, not a *web-first agent framework*. In your thesis taxonomy, it may belong in a separate category or serve as a reference point for the "general compute" end of the spectrum.

**Reproducibility:** Because the agent generates code non-deterministically, exact step-by-step reproduction of a task execution is impossible without logging the generated code. This is a significant limitation for academic methodology sections.

**Security angle:** The risk of LLM-generated code execution is a legitimate research concern — relevant if your thesis discusses safety and trust in agentic systems.

**Comparison to browser-use:** Browser-use provides a structured action space (indexed DOM elements); Open Interpreter provides an unconstrained code generation interface. The trade-off is expressiveness vs. reliability — a key theme in the broader web agent literature.

**Relevant context:** Open Interpreter was one of the first open-source projects to demonstrate that LLMs could autonomously drive a computer, predating most frameworks in this comparison. It has significant historical importance for the field.
