# Stagehand

**Framework name:** Stagehand  
**Repository:** https://github.com/browserbase/stagehand  
**Snapshot date:** 2026-03-18  

---

## Architecture

Stagehand is an **AI browser automation framework** built and maintained by the Browserbase team. Its core philosophy is bridging the gap between fully manual Playwright code (precise but brittle) and fully autonomous agents (flexible but unpredictable) — letting developers choose, step by step, how much AI involvement each action needs.

The framework is built around **three atomic primitives**:
- `act(instruction)` — execute a single natural language action on the page
- `extract(instruction, zodSchema)` — extract structured data with Zod schema validation
- `observe()` — get a list of suggested possible actions on the current page (useful for grounding planning prompts)

For multi-step autonomous tasks, Stagehand v3 adds:
- `agent().execute(goal)` — run a computer-use model (OpenAI CUA or Anthropic) to complete a multi-step goal autonomously

**v3 architectural changes (2025):** Stagehand v3 removed the Playwright dependency and rewrote the browser layer to communicate directly via **Chrome DevTools Protocol (CDP)**. This makes it ~44% faster on average, enables Puppeteer compatibility, works in Bun environments, and provides deeper control over iframes and shadow DOMs. The framework is now described as "automation-first" rather than "testing-first."

**Self-healing and caching:** Stagehand automatically caches discovered elements and actions. Once an action succeeds, it is replayed without LLM inference on subsequent runs. If the page changes and the cache breaks, AI is invoked to re-discover the element. This "write once, run forever" approach dramatically reduces LLM token costs.

**Context builder:** v3 includes a new context builder that feeds models only the essential DOM/accessibility data, reducing token waste.

Stagehand is built to work with **Browserbase** cloud browser infrastructure, but also supports local execution.

---

## Perception type

**Hybrid (DOM/accessibility tree + optional vision).** Stagehand primarily uses the accessibility tree and DOM snapshots to build context for the LLM. Vision (screenshots) is available but not the default. The context builder selectively includes only the most relevant DOM elements, avoiding the token bloat of sending full page HTML. For the `agent()` mode with computer-use models, vision is used.

---

## LLM compatibility

Multi-provider, developer-configurable:
- **OpenAI** — GPT-4o (default), `computer-use-preview` (for agent mode)
- **Anthropic** — Claude Sonnet, Claude (for agent mode with computer use)
- Any OpenAI-compatible API endpoint

The `agent()` method specifically supports OpenAI's and Anthropic's computer-use models for pixel-level interaction.

---

## Action interface

Three primary methods plus agent mode:

```typescript
// Single atomic actions
await page.act("click on the login button");
await page.act("type 'hello@example.com' into the email field");

// Structured data extraction with Zod schema
const { title, price } = await page.extract({
  instruction: "extract the product title and price",
  schema: z.object({ title: z.string(), price: z.number() })
});

// Observe available actions
const actions = await page.observe("what can I do on this page?");

// Multi-step autonomous agent
const agent = stagehand.agent({ provider: "openai", model: "computer-use-preview" });
await agent.execute("complete the checkout process");
```

---

## Dependencies

- Node.js / TypeScript (primary SDK)
- Python SDK also available (`stagehand-python`, actively maintained)
- SDKs also available in Go, Rust, Java, Kotlin, C#, PHP, Ruby (alpha/beta)
- `zod` (TypeScript schema validation)
- Browserbase account (recommended; local mode available)
- LLM provider API key

---

## Browser control method

**CDP (Chrome DevTools Protocol) directly** — Stagehand v3 removed the Playwright dependency. It now speaks directly to Chromium via CDP, with optional Puppeteer driver support. For cloud execution, it uses Browserbase's headless browser infrastructure over CDP.

---

## Strengths

- **Hybrid code + AI design** — use Playwright-style code for known steps, AI for unknown pages; best of both worlds
- **Self-healing + caching** — repeated runs are cheap and fast; AI only re-engages when needed
- **Strong TypeScript developer experience** — clean API, Zod schema validation, excellent type safety
- **v3 CDP architecture** — 44% faster, handles iframes/shadow DOM better, Puppeteer compatible
- **Multi-language SDKs** — Python, Go, Rust, Java etc. make it accessible beyond the TypeScript ecosystem
- **Computer-use model integration** — one-line integration with OpenAI CUA and Anthropic computer use
- **Half a million weekly npm downloads** (as of early 2025) — significant adoption
- **Active development** — daily commits, responsive team, major v3 release in 2025

---

## Weaknesses

- **Browserbase dependency for full features** — self-hosting gives basic functionality but cloud features (anti-detection, stealth, session management) require a paid Browserbase account
- **TypeScript-primary** — Python SDK is newer and less battle-tested than the TS version
- **No built-in task orchestration** — Stagehand is a primitive/tool layer; complex multi-step workflows require the developer to write the orchestration logic (or use `agent()` mode)
- **Smaller community than browser-use** — GitHub stars lower (~21k vs browser-use's ~50k+)
- **Agent mode reliability** — the `agent()` mode using computer-use models can still be unpredictable for complex multi-page tasks
- **Cost** — Browserbase cloud is a paid service; local mode has limitations

---

## Typical use cases

- Production web automation where reliability > cost (e-commerce, form automation)
- Developer tools and copilots that need browser interaction
- Web scraping with structured output validation
- CI/CD pipeline integration for browser-based testing
- Any TypeScript/Node.js application needing browser control with AI fallback
- Rapid prototyping of browser automation workflows

---

## Additional notes for thesis research

**Philosophical positioning:** Stagehand's tagline is "the natural choice between low-level Playwright and unpredictable agents." This is the key thesis comparison: browser-use automates the *entire* decision loop with an LLM; Stagehand lets the developer decide which steps to automate and which to hardcode. For your thesis, this is a "controllability vs. autonomy" trade-off worth discussing.

**v3 CDP migration** is architecturally significant — the same direction browser-use took (away from Playwright's test layer, toward direct CDP). This convergence suggests CDP is becoming the standard substrate for production web agents.

**Comparison to browser-use:** browser-use = fully autonomous agent framework with LLM driving everything; Stagehand = semi-autonomous with developer-controlled automation level. Stagehand is arguably more production-ready for known workflows; browser-use is more research-friendly for exploring autonomous capabilities.

**For your implementation:** Stagehand's Python SDK could be used alongside browser-use if you want to compare the two approaches on your defined tasks — a natural experiment for chapter 7.

**Steel leaderboard note:** Stagehand is not separately listed on the Steel leaderboard (it's a framework, not an agent). Its performance depends entirely on which LLM is configured and what actions are automated vs. hardcoded.
