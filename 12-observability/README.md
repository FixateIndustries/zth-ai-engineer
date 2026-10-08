# Module 12: Observability

Evals tell you how your app does on examples you wrote down. Observability tells you what it's actually doing for real users, right now: which requests are slow, which are expensive, which ones failed, and what the model saw when it gave a strange answer. In this module you'll build a small tracer by hand so you understand what the tools record, add cost and latency accounting, decide what to watch in production, and then see where hosted tools like Langfuse, LangSmith, Helicone, and Arize Phoenix fit.

**You'll be able to:**
- Trace a request through retrieval, model calls, and tool calls as a tree of spans
- Compute the cost and latency of every model call from the API's `usage` data
- Choose what to monitor and alert on in production, including online evals and user feedback
- Wire up a hosted LLM observability tool, and know how OpenTelemetry's GenAI conventions fit in

**Prerequisites:** [Module 2: Calling models](../02-calling-models/README.md), [Module 8: Agents](../08-agents/README.md), and [Module 11: Evals](../11-evals/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [Tracing and logging](01-tracing-and-logging.md) | ~15 min |
| 2 | [Cost and latency](02-cost-and-latency.md) | ~15 min |
| 3 | [Production monitoring](03-production-monitoring.md) | ~15 min |
| 4 | [Observability tools](04-observability-tools.md) | ~20 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Every Course Buddy request gets a trace: one root span per question, with child spans for retrieval, each model call, and each tool call. Each model span records tokens, cost, and latency, and a small report script tells you what a day of questions cost. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../11-evals/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-tracing-and-logging.md)
