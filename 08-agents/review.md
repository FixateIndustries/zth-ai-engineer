# Module 8 review: AI Agents

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Which of these is an agent?
a) A script that retrieves 5 chunks and asks the model to answer
b) A model that calls `get_weather` once and then answers
c) A model that can search, read files, and run tests, and keeps choosing its next action until it decides the bug is fixed
d) A prompt chain: summarize, then translate, then format

<details><summary>Answer</summary>

**c.** An agent uses tools in a loop and chooses its own next step. (a) and (d) are workflows with fixed steps. (b) is a single function call. See [What makes an agent](01-what-makes-an-agent.md).

</details>

### 2. Name the four questions to ask before building an agent.

<details><summary>Answer</summary>

Is the task hard to specify in advance? Is the result worth the cost and latency? Can the model actually do it? Can mistakes be caught and recovered from? A "no" to any of them means a single call or a workflow is the better choice. See [lesson 1](01-what-makes-an-agent.md#when-not-to-use-an-agent).

</details>

### 3. The model returns two `tool_use` blocks in one response. What exactly do you send back?

<details><summary>Answer</summary>

First append the full assistant `content` (both `tool_use` blocks included). Then send **one** user message containing two `tool_result` blocks, each with the matching `tool_use_id`. See [Tools and the ReAct loop](02-tools-and-the-react-loop.md#one-round-trip).

</details>

### 4. What's wrong with this loop?

```python
while True:
    response = client.messages.create(model=MODEL, max_tokens=16000,
                                      tools=TOOLS, messages=messages)
    if response.stop_reason == "end_turn":
        break
    for block in response.content:
        if block.type == "tool_use":
            result = HANDLERS[block.name](**block.input)
            messages.append({"role": "user", "content": [
                {"type": "tool_result", "tool_use_id": block.id, "content": result}]})
```

<details><summary>Answer</summary>

Several things:
1. It never appends the assistant turn, so the `tool_result` blocks have no matching `tool_use`. The next request fails.
2. Each result goes in its own user message instead of all results in one.
3. There's no iteration limit, so it can loop forever.
4. An unknown tool name or a tool exception crashes the program instead of returning an `is_error` result.
5. `max_tokens` or `refusal` stop reasons fall through to the tool branch silently.

Compare with [the full agent](03-build-an-agent-from-scratch.md#the-code).

</details>

### 5. A tool raises `FileNotFoundError`. What should the agent loop do?

<details><summary>Answer</summary>

Catch it and return a `tool_result` with `"is_error": true` and an actionable message (for example, which paths are valid). The model can then retry with a corrected argument or explain the problem. Crashing wastes the run, and silently dropping the result breaks the conversation. See [lesson 2](02-tools-and-the-react-loop.md).

</details>

### 6. You want Claude to always call `lookup_order` first. On `claude-sonnet-5-5`, `tool_choice={"type": "tool", "name": "lookup_order"}` returns a 400. What do you do instead?

<details><summary>Answer</summary>

Use `tool_choice` `auto` (the default) and instruct the model in the system prompt to call `lookup_order` first. Check the response for a `tool_use` block. If you only forced the tool to get JSON back, use structured output instead. See [Steering tool use](02-tools-and-the-react-loop.md#steering-tool-use).

</details>

### 7. How does ReAct relate to native tool calling?

<details><summary>Answer</summary>

Native tool calling is ReAct with a real protocol. "Thought" is the model's reasoning, "Action" is a `tool_use` block, "Observation" is a `tool_result`, and "Final Answer" is an `end_turn` response. You no longer have to parse "Action:" lines with regex. See [lesson 2](02-tools-and-the-react-loop.md#react-the-idea-underneath).

</details>

### 8. Your team wants an agent that edits files in a repo and runs the test suite. Which SDK is the most natural fit, and why?

<details><summary>Answer</summary>

The Claude Agent SDK. It ships Claude Code's built-in `Read`, `Edit`, `Bash`, `Glob`, and `Grep` tools, plus permissions and hooks, so you don't have to build and secure file and shell tools yourself. See [Agent SDKs](04-agent-sdks.md#claude-agent-sdk).

</details>

### 9. True or false: "Vertex AI Agent Engine" and Google ADK are competing frameworks.

<details><summary>Answer</summary>

**False.** ADK is the framework you build agents with. Agent Engine (now **Agent Runtime** on the Gemini Enterprise Agent Platform) is managed infrastructure you deploy them to. They're designed to be used together. See [lesson 4](04-agent-sdks.md#google-clouds-managed-agent-platform-formerly-vertex-ai-agent-builder).

</details>

### 10. What are the two main real benefits of a multi-agent design, and the biggest cost?

<details><summary>Answer</summary>

Benefits: **context isolation** (each worker has a clean, focused context and returns a compact result) and **parallelism**. Cost: tokens. Anthropic measured multi-agent runs at roughly 15 times the tokens of a chat, plus extra coordination failure modes. See [Multi-agent systems](05-multi-agent-systems.md).

</details>

### 11. Scenario: your agent sometimes runs 30+ iterations, calling `search_course` with nearly identical queries. List three things you'd change.

<details><summary>Answer</summary>

Any three of these:
- Lower `MAX_ITERATIONS` and add a token budget.
- Improve the search tool's output so a single search is useful (more context per hit, file paths the model can act on).
- Make "no results" messages actionable ("Try broader keywords, or use list_modules").
- Tell the model in the system prompt to read a lesson once it has a promising hit, rather than searching again.
- Log trajectories and look for the pattern before changing anything.

See [lesson 3](03-build-an-agent-from-scratch.md#what-each-guard-is-for).

</details>

## Hands-on challenge

Add a **human approval gate** to your lesson 3 agent. Give Course Buddy a fourth tool, `save_note(title, text)`, that writes a Markdown file into a `notes/` folder. Before running it, your loop must print the proposed note and ask `Save this note? [y/N]`. If the user declines, return a `tool_result` saying the user declined, so the model can respond gracefully. Read-only tools still run without asking.

<details><summary>Hint</summary>

Keep a set `NEEDS_APPROVAL = {"save_note"}`. In the tool loop, before calling `run_tool`, check `block.name in NEEDS_APPROVAL` and call `input()`. On "no," append `{"type": "tool_result", "tool_use_id": block.id, "content": "User declined to save this note.", "is_error": True}`. Validate `title` in `save_note` so it can't contain `/` or `..`. The model chose the file name, so treat it as untrusted.

</details>

## Capstone checkpoint
- [ ] Course Buddy runs as an agent loop with `search_course`, `read_lesson`, and `list_modules` tools
- [ ] `search_course` uses your Chroma collection from Module 6 (not just keyword matching)
- [ ] The loop handles `tool_use`, `end_turn`, and unexpected stop reasons, and has an iteration limit
- [ ] Tool errors come back to the model as `is_error` results, and `read_lesson` refuses paths outside the course
- [ ] Answers cite the lesson files the agent actually read

---
[← Last lesson](05-multi-agent-systems.md) · [Course home](../README.md) · [Next module →](../09-mcp/README.md)
