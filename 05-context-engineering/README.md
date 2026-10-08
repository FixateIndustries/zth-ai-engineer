# Module 5: Context Engineering

Prompt engineering is about how you ask. Context engineering is about everything else the model sees when it answers: instructions, conversation history, retrieved documents, tool results, memories, and user state, all competing for space in one finite window. As your apps grow from single calls into chatbots and agents, deciding what goes into that window, from where, and in what order becomes the job. This module gives you concrete patterns for doing it well.

**You'll be able to:**
- Design the context for a request: which sources feed it, and how to order them
- Give an app memory across turns and sessions, and compact history before it overflows
- Work with long inputs, cut cost and latency with prompt caching, and filter what you retrieve
- Share context between agents, and isolate it when sharing would hurt
- Recognize context failures and security risks, and test for them
- Know where context warehouses and data catalogs fit in

**Prerequisites:** [Module 4: Prompt Engineering](../04-prompt-engineering/README.md) and [Context windows](../01-how-llms-work/03-context-windows.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [Beyond the prompt](01-beyond-the-prompt.md) | ~12 min |
| 2 | [Memory and state](02-memory-and-state.md) | ~15 min |
| 3 | [Long context and prompt caching](03-long-context-and-caching.md) | ~15 min |
| 4 | [Context for multiple agents](04-context-for-multiple-agents.md) | ~12 min |
| 5 | [When context goes wrong](05-when-context-goes-wrong.md) | ~12 min |
| 6 | [Context tooling](06-context-tooling.md) | ~8 min |
| ✓ | [Module review](review.md) | ~10 min |

## Capstone checkpoint
Course Buddy gets a memory: conversation history that's summarized when it nears a token budget, so long chats don't overflow or get expensive. You'll also cache its system prompt so repeat requests are cheaper and faster. See the [capstone](../capstone/README.md).

---
[← Previous module](../04-prompt-engineering/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-beyond-the-prompt.md)
