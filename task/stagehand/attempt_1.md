# Stagehand — Consistency Test

## ❌ Framework Not Testable with Groq

### Setup
- **Framework**: `@browserbasehq/stagehand` (v3)
- **Installation**: `npm install @browserbasehq/stagehand`
- **LLM Attempted**: Groq / `llama-3.3-70b-versatile`

### Problem
Stagehand V3 API (`extract`, `act`) internally uses the OpenAI SDK regardless of the configured model provider. When using Groq, the framework still requires an `OPENAI_API_KEY` environment variable.

**Error received:**
```
LoadAPIKeyError: OpenAI API key is missing. Pass it using the 'apiKey' parameter or the OPENAI_API_KEY environment variable.
```

### Additional Issues Encountered
- `stagehand.page` is not directly accessible — requires `[...stagehand.ctx.pagesByTarget.values()][0]`
- `extract()` returns raw `pageText` (accessibility tree) instead of structured data when using non-OpenAI providers
- Browser launches successfully but LLM calls fail

### Conclusion
Stagehand requires an **OpenAI API key** to function. Not compatible with Groq free tier.

**Verdict**: ❌ Skipped — requires paid OpenAI API.
