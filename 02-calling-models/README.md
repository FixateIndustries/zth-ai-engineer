# Module 2: Calling Models

Everything you build as an AI engineer starts with one HTTP request: you send a model some messages, it sends text back. This module makes that request second nature. You'll call Claude, OpenAI and Gemini side by side, reach dozens of other models through the OpenAI-compatible format, stream tokens as they're generated, get JSON you can trust, and let a model ask your code to run a function.

**You'll be able to:**
- Make single-turn and multi-turn calls with the Claude Messages API, the OpenAI Responses API and the Gemini API
- Point one OpenAI-style client at OpenRouter, Ollama or Hugging Face by changing two lines
- Stream a response to the terminal and know what you give up when you do
- Get schema-valid JSON back and validate it with Pydantic
- Run a single round of function calling: define a tool, execute the call, return the result

**Prerequisites:** [Module 0: Orientation](../00-orientation/README.md) (a working `uv` project with an API key) and [Module 1: How LLMs Work](../01-how-llms-work/README.md) (tokens and context windows).

| # | Lesson | Time |
|---|--------|------|
| 1 | [Your first API call](01-your-first-api-call.md) | ~15 min |
| 2 | [OpenAI-compatible APIs](02-openai-compatible-apis.md) | ~15 min |
| 3 | [Streaming responses](03-streaming-responses.md) | ~10 min |
| 4 | [Structured output](04-structured-output.md) | ~15 min |
| 5 | [Function calling](05-function-calling.md) | ~15 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Course Buddy becomes a real terminal chat: a loop that keeps the conversation, streams each answer as it's generated, and has a `--json` mode that returns `{answer, confidence}` validated by Pydantic. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../01-how-llms-work/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-your-first-api-call.md)
