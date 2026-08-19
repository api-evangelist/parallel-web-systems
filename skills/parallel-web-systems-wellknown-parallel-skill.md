---
name: Parallel
description: Use when building AI agents that need to search the web, extract page content, conduct deep research, enrich structured data, discover entities, or monitor the web for changes. Parallel provides APIs for real-time web access, research synthesis, and data enrichment with citations.
metadata:
    mintlify-proj: parallel
    version: "1.0"
---

# Parallel API Skill

## Product summary

Parallel is a web API platform for AI agents that need live web access, research synthesis, and data enrichment. It provides six core APIs: **Search** (natural-language web search with LLM-optimized excerpts), **Extract** (URL-to-markdown conversion), **Task** (multi-hop research with citations), **FindAll** (entity discovery from natural language), **Entity Search** (fast people/company lookup), and **Monitor** (continuous web tracking). Use the Python SDK (`parallel-web`), TypeScript SDK (`parallel-web`), CLI (`parallel-cli`), or MCP servers. Authenticate with `PARALLEL_API_KEY` environment variable or OAuth. Primary docs: https://docs.parallel.ai

## When to use

Reach for Parallel when:
- An agent needs **current facts or web data** to ground a response (use Search)
- You need to **extract content from specific URLs** after narrowing via search (use Extract)
- A task requires **multi-hop research** with synthesis and citations (use Task API)
- You're **building a list of entities** from scratch based on natural language criteria (use FindAll)
- You need **fast people/company lookup** in latency-sensitive workflows (use Entity Search)
- You need **continuous monitoring** of the web for changes on a schedule (use Monitor)
- You're integrating with **chat assistants** (use MCP servers) or **coding agents** (use CLI or SDK)

Do not use Parallel for: private data access (Parallel only accesses public web), authentication-gated content, or tasks that don't require live web data.

## Quick reference

### API Selection Table

| API | Input | Output | Latency | Use when |
|-----|-------|--------|---------|----------|
| **Search** | Objective + 2-3 keyword queries | LLM-optimized excerpts | 200ms–3s | Model needs current facts or entities |
| **Extract** | URLs + optional objective | Markdown (excerpts or full) | 1–20s | Pulling specific page content |
| **Task** | Plain text or JSON input | Structured output + citations | 10s–2hr | Deep research, enrichment, synthesis |
| **FindAll** | Natural language query | Matched entities + enrichments | 10s–2hr | Building lists from scratch |
| **Entity Search** | NL people/company query | Set of matching results | 1–3s | Fast latency-sensitive lookups |
| **Monitor** | NL query + frequency | Webhook events on change | Ambient | Continuous tracking |

### SDK Setup

```bash
# Python
pip install parallel-web
export PARALLEL_API_KEY="your-api-key"

# TypeScript
npm install parallel-web
export PARALLEL_API_KEY="your-api-key"
```

### Core Workflow (Task API Example)

```python
from parallel import Parallel

client = Parallel()  # reads PARALLEL_API_KEY from env

# 1. Create a task run
task_run = client.task_run.create(
    input="Stripe",
    task_spec={"output_schema": "Founding year and total funding raised"},
    processor="base"
)

# 2. Wait for completion
result = client.task_run.result(task_run.run_id, api_timeout=3600)

# 3. Access output and citations
print(result.output.content)
for field in result.output.basis:
    print(f"{field.field}: {len(field.citations)} citations")
```

### Processor Tiers (Task API)

| Tier | Latency | Best for | Max fields |
|------|---------|----------|------------|
| `lite` / `lite-fast` | 10–60s / 10–20s | Basic metadata | ~2 |
| `base` / `base-fast` | 15–100s / 15–50s | Standard enrichments | ~5 |
| `core` / `core-fast` | 60s–5min / 15–100s | Cross-referenced outputs | ~10 |
| `pro` / `pro-fast` | 2–10min / 30s–5min | Exploratory research | ~20 |
| `ultra` / `ultra-fast` | 5–25min / 1–10min | Deep multi-source research | ~20 |
| `ultra8x` | 5min–2hr | Most difficult research | ~25 |

Append `-fast` for 2–5x speed; trade-off is data freshness (still very fresh, optimized for speed).

### Integration Methods

| Method | Best for | Auth | Docs |
|--------|----------|------|------|
| **CLI** | Coding agents (Cursor, Cline, Claude Code) | API key or OAuth | `/integrations/cli` |
| **MCP** | Chat assistants (Claude, ChatGPT, Gemini) | OAuth or API key | `/integrations/mcp/quickstart` |
| **SDK** | Custom apps, full control | API key | Python/TypeScript SDKs |

## Decision guidance

### When to use Search vs Extract vs Task

| Scenario | Use |
|----------|-----|
| Model needs current facts in one round-trip | Search |
| You already know which URL(s) to read | Extract |
| Need synthesis across multiple sources + citations | Task |
| Building a list of entities from scratch | FindAll |
| Need fast people/company lookup (seconds) | Entity Search |
| Need continuous monitoring on a schedule | Monitor |

### When to use standard vs fast processors

| Scenario | Use |
|----------|-----|
| Real-time data critical (stock prices, breaking news) | Standard processor |
| Interactive app, user waiting for results | Fast processor |
| Background batch job, accuracy paramount | Standard processor |
| Testing agents, rapid iteration | Fast processor |

### When to use Task API vs Search API

| Aspect | Search | Task |
|--------|--------|------|
| **Latency** | 200ms–3s | 10s–2hr |
| **Reasoning** | None; returns excerpts | Full synthesis + citations |
| **Complexity** | Single search pass | Multi-hop research |
| **Output** | Raw excerpts | Structured, cited answer |
| **Cost** | Lower | Higher |

Use Search for quick facts; Task for deep research.

## Workflow

### Typical Task API enrichment workflow

1. **Define the task spec**: Write clear input/output schemas. Use field-level `description` to control output quality (this is your "prompt" for each field).
2. **Choose a processor**: Match complexity (lite/base for simple, core for ~10 fields, pro/ultra for deep research) and latency needs (append `-fast` for speed).
3. **Create the task run**: Call `client.task_run.create()` with input, task_spec, and processor.
4. **Wait for completion**: Use `client.task_run.result(run_id, api_timeout=3600)` for blocking (good for `pro` tier, <10min). For `ultra` tier (up to 2hr), use webhooks instead.
5. **Access results and citations**: Read `result.output.content` for the answer; `result.output.basis` for per-field citations and reasoning.
6. **Handle errors**: Check `result.status` for `failed`; inspect `result.errors` for details.

### Typical Search API workflow

1. **Craft the objective**: Natural-language research goal (full sentences OK).
2. **Write 2–3 keyword queries**: Diverse, 3–6 words each; vary entities and angles.
3. **Call search**: `client.search(objective=..., search_queries=[...])`.
4. **Iterate results**: Loop over `search.results`; each has `title`, `url`, `excerpts`.
5. **Extract if needed**: If excerpts aren't enough, call Extract on the full URL.

### Typical FindAll workflow

1. **Ingest**: Convert NL query to schema with `client.beta.findall.ingest(objective=...)`.
2. **Review and edit**: Adjust `match_conditions` if ingest generated unexpected criteria.
3. **Create run**: Call `client.beta.findall.create()` with edited schema and `generator` tier.
4. **Poll status**: Check `client.beta.findall.retrieve(findall_id)` until `status == "completed"`.
5. **Fetch results**: Call `client.beta.findall.result(findall_id)` to get matched candidates.
6. **Enrich (optional)**: Add enrichments via `client.beta.findall.enrich()` to extract additional fields.

## Common gotchas

- **Task spec descriptions are critical**: The `description` field on each output field is your primary control lever. Write detailed descriptions with format requirements, fallback behavior, and data sources. Vague descriptions produce vague outputs.
- **Keep schemas flat**: Avoid deeply nested JSON structures in input/output schemas. Flat structures are easier for the processor to reason about.
- **Don't block on ultra-tier tasks**: Tasks with `processor="ultra"` or higher can run up to 2 hours. Don't use `api_timeout=3600` blocking; use webhooks instead.
- **Search queries must be keywords, not sentences**: `search_queries` must be 2–3 diverse, 3–6-word keyword phrases. Don't write full sentences or use `site:` operators.
- **Objective vs search_queries are distinct**: `objective` is natural-language context (full sentences OK); `search_queries` are keyword queries. Both are required for Search API.
- **Don't cache outputs across users**: Task API, FindAll, and Monitor outputs are contextually tied to inputs. If inputs include private data, don't reuse outputs across customers. Search and Extract are safe to cache (they return public web content).
- **FindAll match conditions can be too strict**: If a run returns 0 matches, relax the `match_conditions` descriptions or use a stronger `generator` tier (preview → base → core → pro).
- **Rate limits apply to creates, not gets**: Creating a task counts against rate limits; polling status does not. GET requests are free.
- **402 Payment Required means insufficient credits**: Add funds to your account at platform.parallel.ai.
- **Processor field limits are approximate**: Max fields depend on complexity. Simple fields (dates, booleans) use less capacity than complex analytical fields. If near the limit and seeing quality issues, try a higher tier.

## Verification checklist

Before submitting work with Parallel:

- [ ] **API key is set**: `PARALLEL_API_KEY` is in environment or passed explicitly.
- [ ] **Task spec is clear**: Output schema has detailed field-level `description` fields with format requirements and fallback behavior.
- [ ] **Schema is flat**: No deeply nested structures in input/output schemas.
- [ ] **Processor tier matches task**: lite/base for simple, core for ~10 fields, pro/ultra for deep research.
- [ ] **Latency expectations are met**: Using `-fast` processors for interactive apps; standard for background jobs.
- [ ] **Polling strategy is correct**: Using `api_timeout` for `pro` tier; webhooks for `ultra` tier.
- [ ] **Error handling is in place**: Checking `result.status` and `result.errors` for failures.
- [ ] **Citations are accessible**: For Task API, `result.output.basis` contains per-field citations and reasoning.
- [ ] **Rate limits are respected**: Not creating tasks faster than quota allows; polling status doesn't count.
- [ ] **No private data in cached outputs**: If inputs include sensitive data, not reusing outputs across users.

## Resources

- **Full documentation index**: https://docs.parallel.ai/llms.txt (comprehensive page-by-page navigation for agents)
- **API Reference**: https://docs.parallel.ai/api-reference/tasks/create-task-run (Task API endpoint specs)
- **Task API Best Practices**: https://docs.parallel.ai/task-api/best-practices (task spec design, field descriptions, schema patterns)
- **Processor Selection Guide**: https://docs.parallel.ai/task-api/guides/choose-a-processor (latency/cost trade-offs, when to use each tier)

---

> For additional documentation and navigation, see: https://docs.parallel.ai/llms.txt