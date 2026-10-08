# Module 5 review: Context Engineering

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. In one sentence, how does context engineering differ from prompt engineering?

<details><summary>Answer</summary>

Prompt engineering is about wording instructions and examples; context engineering is about deciding everything that goes into the window (instructions, history, memory, retrieved docs, tool results, state), where it comes from, and in what order, on every request. See [Beyond the prompt](01-beyond-the-prompt.md).

</details>

### 2. Put these in a good order for a single request: retrieved documents, system instructions, the user's current question, conversation history, tool definitions.

<details><summary>Answer</summary>

Tool definitions and system instructions (stable), then conversation history (grows), then retrieved documents (per request), then the current question last. Stable-first helps prefix caching; documents-before-question helps the model use them. See [Beyond the prompt](01-beyond-the-prompt.md).

</details>

### 3. Your chatbot "remembers" earlier messages. Where does that memory actually live?

a) In the model's weights, updated after each conversation
b) On the provider's servers, automatically
c) In your app, which resends the history with each request
d) In the system prompt

<details><summary>Answer</summary>

**c.** Messages APIs are stateless. Your app stores the conversation and replays it each call. See [Memory and state](02-memory-and-state.md).

</details>

### 4. After compaction kicks in, your assistant forgets the exact file path the user mentioned early on. What do you change?

<details><summary>Answer</summary>

The summarization prompt: explicitly tell it to keep specifics like names, numbers, and file paths (and give the summary call enough `max_tokens`). For details that must never be lost, also write them to long-term memory. See [Memory and state](02-memory-and-state.md).

</details>

### 5. What's wrong with this system prompt from a caching point of view?

```python
system = f"Today is {datetime.now():%Y-%m-%d %H:%M}. You are Course Buddy...\n{LONG_RULES}"
```

<details><summary>Answer</summary>

The timestamp sits at the very start of the prefix and changes every minute, so nothing after it can ever be read from cache. Move the date into the latest user message (or drop the time and keep just the date late in the context). See [Long context and prompt caching](03-long-context-and-caching.md).

</details>

### 6. You add `cache_control` to your system prompt, but `cache_read_input_tokens` is always 0. Name two likely causes.

<details><summary>Answer</summary>

Any two of: something in the prefix changes between requests (timestamp, user ID, unordered tool list); the prefix is shorter than the model's minimum cacheable length; requests are more than the cache lifetime (5 minutes by default) apart; the model changed between requests. See [Long context and prompt caching](03-long-context-and-caching.md).

</details>

### 7. Your model supports a 1M-token window. Why not put your whole 800K-token knowledge base in every request?

<details><summary>Answer</summary>

Cost and latency scale with input size on every request, and answer quality tends to drop with long, mostly irrelevant context (context rot, lost in the middle, distraction). Retrieve the relevant slice instead, and reserve long context for tasks that truly need the whole thing. See [Long context and prompt caching](03-long-context-and-caching.md).

</details>

### 8. An orchestrator delegates research to four sub-agents. What should each sub-agent return?

a) Its full message history, so nothing is lost
b) A condensed result in an agreed format, plus sources or a reference to full output
c) Only "done"
d) The raw documents it read

<details><summary>Answer</summary>

**b.** The point of isolation is to keep messy work out of the orchestrator's context. Condensed results in a shared format can be combined; full outputs can be stored as artifacts and referenced. See [Context for multiple agents](04-context-for-multiple-agents.md).

</details>

### 9. Two sub-agents each built half of a settings page in isolation, and the halves don't match. What went wrong, and what would you do differently?

<details><summary>Answer</summary>

Tightly coupled work was split across isolated contexts, so each made design decisions the other couldn't see. Either use a single agent (with compaction) for coupled work, or share a decision log and constraints in every brief. See [Context for multiple agents](04-context-for-multiple-agents.md).

</details>

### 10. Your support bot retrieved both the v1 and v2 docs and gave an answer mixing the two. Which failure mode is this, and what's the fix?

<details><summary>Answer</summary>

Context clash (contradictory information in the window). Filter retrieval by product version (metadata filter) so only the relevant docs are retrieved, or resolve conflicts before sending. See [When context goes wrong](05-when-context-goes-wrong.md).

</details>

### 11. A teammate proposes preventing cross-customer leaks by adding "Never reveal other customers' data" to the system prompt. Is that enough?

<details><summary>Answer</summary>

No. If another customer's data is in the context, assume it can come out. Enforce the boundary in code by filtering retrieval (and memory) by tenant before anything reaches the window. See [When context goes wrong](05-when-context-goes-wrong.md).

</details>

### 12. When would a data catalog like DataHub or Atlan be worth connecting to your agent?

<details><summary>Answer</summary>

When the agent must work across a large, shared data estate and failures come from picking the wrong source (wrong table, wrong definition), especially if the company already maintains a catalog for humans. Its MCP server can give agents lineage, ownership, and definitions. See [Context tooling](06-context-tooling.md).

</details>

## Hands-on challenge

Build a "context inspector" for any LLM feature you've written (Course Buddy is perfect). Before each request, log a breakdown like:

```text
system: 1,850 tokens | tools: 0 | history: 6,420 (12 msgs) | docs: 3,100 (4 chunks) | question: 22
cache: written=0 read=1,850
```

Then use it to make one measurable improvement: cut total input tokens by 30% without failing any question in your test set from Module 4.

<details><summary>Hint</summary>

Use `client.messages.count_tokens(...)` on each part separately (for example, count the system prompt alone with a one-word user message, then subtract). Usual wins: fewer or shorter retrieved chunks, compacting history earlier, and removing duplicated instructions. Read `usage.cache_read_input_tokens` from the response for the cache line.

</details>

## Capstone checkpoint
- [ ] Course Buddy keeps conversation history across turns in a chat loop
- [ ] When history passes a token budget (measured with token counting), older turns are summarized and recent turns kept verbatim
- [ ] The summary prompt explicitly preserves names, lesson paths, and open questions
- [ ] The system prompt is sent with `cache_control`, and you've confirmed `cache_read_input_tokens > 0` on repeat requests
- [ ] Nothing dynamic (timestamps, user IDs) sits in the cached prefix

---
[← Last lesson](06-context-tooling.md) · [Course home](../README.md) · [Next module →](../06-embeddings-and-vector-search/README.md)
