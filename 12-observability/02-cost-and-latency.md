# Cost and latency

> Module 12 · Lesson 2 · ~15 min

**Goals**
- Turn the `usage` block of every response into a dollar cost
- Measure total latency and time to first token
- Summarize a day of traces with percentiles, not averages
- Know the main levers for making a feature cheaper or faster

## Where the money goes

Model APIs bill per token, with different prices for input, output, and cached input. Every Claude response tells you exactly what it used:

```python
response = client.messages.create(model="claude-sonnet-5-5", max_tokens=1024, messages=messages)
print(response.usage)
# input_tokens=2950 output_tokens=412 cache_read_input_tokens=2100 cache_creation_input_tokens=0 ...
```

| Field | Meaning |
|---|---|
| `input_tokens` | Input tokens billed at the full rate (not served from or written to the cache) |
| `cache_read_input_tokens` | Input tokens served from the prompt cache, at a steep discount |
| `cache_creation_input_tokens` | Input tokens written to the cache, at a premium |
| `output_tokens` | Generated tokens, including any thinking tokens. These are the expensive ones |

The total input the model saw is the sum of the three input fields.

## A cost function

Keep prices in one table. These are list prices in USD per million tokens at the time of writing. Check the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing) and update them, because they change.

```python
# course_buddy/pricing.py
PRICES = {  # USD per 1M tokens: input, output, cache read, cache write (5-minute TTL)
    "claude-opus-5-5":           {"in": 4.00, "out": 20.00, "cache_read": 0.20, "cache_write": 5.00},
    "claude-sonnet-5-5":         {"in": 2.00, "out": 10.00, "cache_read": 0.10, "cache_write": 2.50},
    "claude-haiku-5-5":          {"in": 0.10, "out": 0.50,  "cache_read": 0.01, "cache_write": 0.125},
}  # Haiku 5.5 prices are for prompts up to 100K tokens; longer prompts cost 5x

def cost_usd(model: str, usage) -> float:
    p = PRICES[model]
    return (
        usage.input_tokens * p["in"]
        + usage.output_tokens * p["out"]
        + (usage.cache_read_input_tokens or 0) * p["cache_read"]
        + (usage.cache_creation_input_tokens or 0) * p["cache_write"]
    ) / 1_000_000
```

Add it to `traced_create` from [lesson 1](01-tracing-and-logging.md):

```python
s["attributes"]["cost_usd"] = round(cost_usd(model, response.usage), 6)
```

For the usage line above on Sonnet 5.5: 2,950 × $2 + 412 × $10 + 2,100 × $0.20, all divided by a million, is about **$0.0104**. Without the cache hit, those 2,100 tokens would have cost $0.0042 instead of $0.0004.

> [!NOTE]
> Use the `usage` numbers the API returns, not your own token estimate. Tokenizers differ between models, and thinking tokens, tool definitions, and system prompts all count. The provider's number is the one on your bill.

## Measuring latency

Users feel two kinds of latency:

- **Time to first token (TTFT).** How long until something appears. Streaming makes this the number that shapes perceived speed.
- **Total time.** How long until the answer is complete. Matters for anything that waits on the full output, like tool calls or JSON parsing.

A streaming version of the traced call records both:

```python
import time

def traced_stream(client, **kwargs):
    model = kwargs["model"]
    with span(f"chat {model}", kind="llm", model=model, stream=True) as s:
        t0 = time.perf_counter()
        ttft_ms = None
        with client.messages.stream(**kwargs) as stream:
            for text in stream.text_stream:
                if ttft_ms is None:
                    ttft_ms = round((time.perf_counter() - t0) * 1000, 1)
                print(text, end="", flush=True)
            final = stream.get_final_message()
        s["attributes"].update(
            ttft_ms=ttft_ms,
            output_tokens=final.usage.output_tokens,
            input_tokens=final.usage.input_tokens,
            cost_usd=round(cost_usd(model, final.usage), 6),
        )
        return final
```

Output tokens drive total latency much more than input tokens. A 2,000-token answer takes several times longer than a 200-token one on the same model, so output length is the first thing to look at when a request is slow.

## Report with percentiles

Averages hide the slow requests your users complain about. Report the median (p50) and the tail (p95, p99):

```python
# scripts/report.py
import json
from collections import defaultdict
from statistics import quantiles

spans = [json.loads(line) for line in open("traces.jsonl")]
roots = [s for s in spans if s["parent_id"] is None]
cost_by_trace = defaultdict(float)
for s in spans:
    cost_by_trace[s["trace_id"]] += s["attributes"].get("cost_usd", 0)

latencies = sorted(r["duration_ms"] for r in roots)
p = quantiles(latencies, n=100)            # needs at least 2 requests
print(f"requests: {len(roots)}   errors: {sum(r['status'] == 'error' for r in roots)}")
print(f"latency p50 {p[49]:.0f} ms   p95 {p[94]:.0f} ms")
print(f"cost total ${sum(cost_by_trace.values()):.4f}   per request ${sum(cost_by_trace.values()) / len(roots):.5f}")
```

Break these down by the things you can act on: model, prompt version, endpoint, and user tier. "p95 doubled" is an alarm. "p95 doubled for agent requests since prompt v7" is a fix.

## Levers for cheaper and faster

| Lever | Saves | Trade-off |
|---|---|---|
| Prompt caching for stable prefixes (system prompt, tools, docs) | Input cost and TTFT | Prefix must stay byte-identical ([Module 5](../05-context-engineering/03-long-context-and-caching.md)) |
| Smaller model for easy requests (routing) | Cost and latency | Quality drop on hard cases. Prove it with evals ([Module 11](../11-evals/README.md)) |
| Lower effort / less thinking | Output tokens, latency | Weaker reasoning on hard tasks |
| Fewer, better retrieved chunks | Input cost | Recall drops if you cut too far |
| Shorter answers (prompt for brevity, sensible `max_tokens`) | Output cost, total latency | Too low a `max_tokens` truncates answers |
| Batch API for offline jobs | Cost | Results arrive asynchronously, not in real time |
| Streaming | Perceived latency (TTFT) | No cost change |

<details><summary>Check yourself: your cost per request jumped 40% overnight, but traffic and prompts didn't change. What do you check first?</summary>

The token breakdown in your traces. A common cause is the cache silently missing: `cache_read_input_tokens` drops to zero and `input_tokens` rises, often because something volatile (a timestamp, a request ID) slipped into the cached prefix. Other suspects: a model ID change in config, longer outputs after a model update, or retrieval returning more or longer chunks.

</details>

> [!TIP]
> **Try it:** Add `cost_usd` to your traced calls and run 10 questions through Course Buddy. Run `report.py`. Then ask the same 10 questions again and compare: did prompt caching kick in for your system prompt on the second pass? Look at `cache_read_tokens` in the spans. If it's zero, find out why.

## Key takeaways
- Compute cost from the API's `usage` fields with a price table you keep up to date. Cached reads and writes are priced differently from plain input.
- Measure both time to first token and total time. Output length drives total time the most.
- Report p50 and p95, broken down by model, prompt version, and route.
- Caching, routing to smaller models, trimming context and output, and batching are the main levers. Check every one with evals.

## Go deeper
- [Claude pricing](https://platform.claude.com/docs/en/about-claude/pricing) — current per-token prices, cache multipliers, and batch discounts.
- [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — how cache reads and writes are billed and how to get hits.
- [Messages API reference](https://platform.claude.com/docs/en/api/messages) — the `usage` object in full.

---
[← Tracing and logging](01-tracing-and-logging.md) · [Module home](README.md) · [Production monitoring →](03-production-monitoring.md)
