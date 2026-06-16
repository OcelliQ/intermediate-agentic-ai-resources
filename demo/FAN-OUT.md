# Live demo: fan-out subagents (parallel usage lookup)

**This file is a prompt for Claude.** During the talk the presenter will say something like
"run `demo/FAN-OUT.md`". When that happens, fan out one child subagent per provider IN
PARALLEL, let each do the wide reading (web lookup of model usage on OpenRouter), then - as
the parent - synthesize their short results into one comparison table.

The teaching point: **the parent stays small. Each child burns its own context on messy web
reading and hands back a few clean numbers. They run in parallel, so wall-clock is the
slowest child, not the sum - and the parent never sees the raw pages.**

---

## How to run it

0. **Announce the plan first (for teaching).** Before spawning anything, print one short line
   on screen saying what is about to happen, e.g.: "Fanning out 3 subagents in parallel - one
   each for OpenAI, Anthropic, and Alibaba - to look up flagship model price + weekly token
   volume on OpenRouter, then I'll merge their results into one table." Then spawn.
1. Spawn THREE child subagents **in a single message** (so they run concurrently), one per
   provider: **OpenAI**, **Anthropic**, **Alibaba (Qwen)**. Use `subagent_type:
   "general-purpose"` so each has web tools (WebSearch / WebFetch). Model: inherit (or pin
   `haiku` to make the "cheap children" point - mention which you chose).
2. Give each child the **Child prompt** below, filled in for its provider. Each child must
   return ONLY a compact result (2-3 flagship models, each with its price and weekly token
   volume, the date, and the source URL) - not the pages it read.
3. As the parent, do the **fan-in**: merge the three short results into one table. Do NOT
   re-fetch anything yourself - synthesize only what the children returned. Flag any provider
   a child couldn't resolve.
4. Present using the **Output format** below.

> Source convention (repo): usage and model data come from OpenRouter - `openrouter.ai/models`.
> Point the children there first; web search is the fallback. Always show the date + source:
> live usage drifts, and "as of <date>, per OpenRouter" is the honest framing.
>
> Reliability note: live web lookups vary and can fail. If a child returns nothing or looks
> stale, say so on screen (a real fan-out has to handle a flaky worker) and either re-run
> that one child or present the other two. This honesty is itself a teaching moment.

---

## The task

For each of three providers, look up its flagship models on OpenRouter and report each
model's **price and usage (weekly token volume)**. Do it in parallel, then compare in one
table.

- Child 1 -> **OpenAI** (e.g. GPT-5.x / GPT-4.1 tier)
- Child 2 -> **Anthropic** (Claude Opus / Sonnet / Haiku)
- Child 3 -> **Alibaba (Qwen)** (Qwen flagship tier)

---

## Child prompt (fill in {PROVIDER} and {EXAMPLE_MODELS})

> You are a research worker. On OpenRouter, find up to 3 flagship {PROVIDER} models
> ({EXAMPLE_MODELS}) and report, for each, its **price** and its recent usage (**weekly token
> volume**).
>
> Source: <https://openrouter.ai/models> (filter by {PROVIDER}); the listing and each model
> page show pricing and weekly token throughput. Use web search only as a fallback.
>
> Return ONLY a compact result - do not include the pages you read or your reasoning:
>
> - For each of up to 3 flagship models: model name, input price and output price (USD per 1M
>   tokens), and weekly token volume (e.g. "2.5T tokens/week").
> - The date the data is valid as of.
> - The source URL.
>
> Keep it to a few lines. If you cannot find price or usage for a model, say so explicitly.

Fill-ins:

- OpenAI -> {EXAMPLE_MODELS} = "GPT-5 / GPT-4.1 family"
- Anthropic -> {EXAMPLE_MODELS} = "Claude Opus, Sonnet, Haiku"
- Alibaba -> {EXAMPLE_MODELS} = "Qwen flagship (e.g. Qwen-Max / Qwen3)"

---

## Output format (what to print on screen)

```text
TASK: flagship models - price + weekly volume - OpenAI vs Anthropic vs Alibaba   (fan-out, 3 children, parallel)

(spawned 3 children at once - each did its own web reading)

| Provider  | Model            | $/1M in | $/1M out | Weekly tokens |
|-----------|------------------|---------|----------|---------------|
| OpenAI    | ...              | ...     | ...      | ...           |
| Anthropic | ...              | ...     | ...      | ...           |
| Alibaba   | ...              | ...     | ...      | ...           |

Source: OpenRouter, as of <date>.  (note anything a child couldn't resolve)
```

Then one punchline line, e.g.:
> Three workers read the web in parallel; the parent only ever saw nine numbers. That is the
> fan-out: wide reading at the edges, a small clean context in the middle.

---

## Optional contrast (if there is time)

Mention what the *single-agent* version would have looked like: one agent fetching all three
providers sequentially, its context filling with three sets of raw usage pages - slower, and
noisier to debug. Same answer, worse shape.
