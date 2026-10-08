# Module 12 review: Observability

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. What's the difference between a trace and a span?

<details><summary>Answer</summary>

A trace is one request end to end. A span is one timed unit of work inside it (a retrieval, a model call, a tool call), with a name, attributes, and a parent. Spans nest to form the trace's tree. See [Tracing and logging](01-tracing-and-logging.md).

</details>

### 2. A response has `input_tokens=1000`, `cache_read_input_tokens=4000`, `cache_creation_input_tokens=0`, `output_tokens=500` on `claude-sonnet-5-5` ($2 in, $10 out, $0.20 cache read per million). What did it cost, and how many input tokens did the model see?

<details><summary>Answer</summary>

(1000 × 2 + 500 × 10 + 4000 × 0.20) / 1,000,000 = (2000 + 5000 + 800) / 1,000,000 = **$0.0078**. The model saw 5,000 input tokens: Anthropic's `input_tokens` counts only uncached input, so you add the cache fields. See [Cost and latency](02-cost-and-latency.md).

</details>

### 3. You export your spans to an OpenTelemetry backend and set `gen_ai.usage.input_tokens` to Anthropic's `usage.input_tokens`. What goes wrong?

<details><summary>Answer</summary>

The GenAI convention expects `gen_ai.usage.input_tokens` to include cache reads and writes. Anthropic's field excludes them. Your backend will under-report input tokens on every cached request, and any cost or cache-hit numbers it derives will be wrong. Sum the three input fields. See [Tracing and logging](01-tracing-and-logging.md).

</details>

### 4. Why report p95 latency rather than the average?

<details><summary>Answer</summary>

Averages hide the slow tail. A few very slow requests barely move the mean but are exactly what users notice and complain about. p50 shows the typical experience and p95/p99 show the worst common one. See [Cost and latency](02-cost-and-latency.md).

</details>

### 5. Users say answers "just stop mid-sentence", but your error rate is 0%. Which signal should you check, and what's the likely fix?

<details><summary>Answer</summary>

The `stop_reason` distribution. A rise in `max_tokens` means responses are being truncated. Raise `max_tokens`, ask for shorter answers, or check whether more thinking or longer context is eating the budget. A `max_tokens` stop rate makes a good alert. See [Production monitoring](03-production-monitoring.md).

</details>

### 6. What's an online eval, and why can't it use the same graders as your golden-set evals?

<details><summary>Answer</summary>

It's a grader run on a sample of real production traces. There's no reference answer for real traffic, so it has to use reference-free checks: groundedness against the retrieved context, relevance to the question, format, being on-topic. See [Production monitoring](03-production-monitoring.md).

</details>

### 7. Which of these is a good alert?
a) Any single request takes longer than 10 seconds
b) Thumbs-down rate above 0 today
c) 429 rate-limit errors above 2% of requests for 10 minutes
d) A new trace was recorded

<details><summary>Answer</summary>

c) It's a rate over a window, and it has a clear action (back off, raise limits, shed load). a) and b) fire on single events that are normal, and d) is just activity. See [Production monitoring](03-production-monitoring.md).

</details>

### 8. Your team can't send prompt or response data to third-party services. Which tools from this module are still options?

<details><summary>Answer</summary>

Self-hosted ones: Langfuse, Helicone, or Arize Phoenix running in your own infrastructure, or your own tracer exporting to an internal OpenTelemetry backend. LangSmith's self-hosting is an enterprise offering. See [Observability tools](04-observability-tools.md).

</details>

### 9. What's the main limitation of a proxy-based tool like Helicone for debugging a RAG agent?

<details><summary>Answer</summary>

It only sees model API calls. Retrieval (queries, scores, dropped chunks), tool execution, and your own logic don't go through the proxy, so they're not in its traces. Pair it with span-level instrumentation if you need those. See [Observability tools](04-observability-tools.md).

</details>

### 10. Daily cost jumped 40% with no change in traffic. Name two likely causes you could confirm from traces.

<details><summary>Answer</summary>

Any two of: prompt cache misses (cache reads drop to zero, uncached input rises), a config change to a more expensive model, longer outputs or more thinking after a model or prompt change, retrieval returning more or longer chunks, or an agent taking more steps per request. See [Cost and latency](02-cost-and-latency.md).

</details>

## Hands-on challenge

Build a `report` command for Course Buddy that reads `traces.jsonl` and prints, for the last 24 hours: request count, error rate, p50 and p95 latency, total and per-request cost, cache hit rate (cache-read tokens as a share of all input tokens), the `stop_reason` breakdown, and the five most expensive traces with their questions. Then use it to find and fix one real inefficiency.

<details><summary>Hint</summary>

Group spans by `trace_id`, take root spans (no `parent_id`) for latency and status, and sum `cost_usd` across all spans in a trace. Filter by the root span's `start` timestamp. The most expensive traces usually point straight at the fix: an oversized context, a missed cache, or an agent looping.

</details>

## Capstone checkpoint
- [ ] Every Course Buddy request produces a trace with a root span and child spans for retrieval, model calls, and tool calls
- [ ] Each model span records model, stop reason, input/output/cache tokens, latency, and cost in USD
- [ ] Streamed calls also record time to first token
- [ ] A report script prints request count, errors, p50/p95 latency, cost, and cache hit rate
- [ ] User feedback (even just a `/good` and `/bad` command in the CLI) is recorded with the trace ID
- [ ] Optional: the same traces viewable in Langfuse, LangSmith, Helicone, or Phoenix

---
[← Last lesson](04-observability-tools.md) · [Course home](../README.md) · [Next module →](../13-safety-and-ethics/README.md)
