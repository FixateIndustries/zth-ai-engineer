# Module 8: AI Agents

So far your code has decided what happens next: retrieve, then prompt, then answer. An agent hands some of that control to the model. It picks a tool, looks at the result, and decides what to do next, in a loop, until the job is done. In this module you'll learn when that's worth it, build an agent from scratch with nothing but the Messages API, compare the main agent SDKs, and see how several agents can work together.

**You'll be able to:**
- Explain the agent loop and tell an agent apart from a fixed workflow
- Decide when an agent is the wrong tool for the job
- Design clear tools and run the ReAct-style loop with native tool calling
- Build a working agent with `tool_use` / `tool_result` blocks, `stop_reason` handling, and an iteration guard
- Pick between the OpenAI Agents SDK, Claude Agent SDK, Google ADK, and Google Cloud's managed agent platform
- Sketch a multi-agent design and say what it costs you

**Prerequisites:** [Module 2: Calling models](../02-calling-models/README.md) (especially [function calling](../02-calling-models/05-function-calling.md)), [Module 4: Prompt engineering](../04-prompt-engineering/README.md), and [Module 7: RAG](../07-rag/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [What makes an agent](01-what-makes-an-agent.md) | ~10 min |
| 2 | [Tools and the ReAct loop](02-tools-and-the-react-loop.md) | ~15 min |
| 3 | [Build an agent from scratch](03-build-an-agent-from-scratch.md) | ~25 min |
| 4 | [Agent SDKs](04-agent-sdks.md) | ~15 min |
| 5 | [Multi-agent systems](05-multi-agent-systems.md) | ~15 min |
| ✓ | [Module review](review.md) | ~10 min |

## Capstone checkpoint

Course Buddy stops being a fixed RAG pipeline and becomes an agent. You give it three tools (`search_course`, `read_lesson`, `list_modules`) and let it decide which ones to call, and how often, before it answers. See the [capstone milestones](../capstone/README.md).

---
[← Previous module](../07-rag/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-what-makes-an-agent.md)
