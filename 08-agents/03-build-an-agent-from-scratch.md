# Build an agent from scratch

> Module 8 · Lesson 3 · ~25 min

**Goals**
- Write a complete agent loop on the Claude Messages API, with no framework
- Handle every `stop_reason` you'll meet and guard against runaway loops
- Turn Course Buddy into an agent with `list_modules`, `search_course`, and `read_lesson`

## The loop, step by step

Here is everything that happens when Course Buddy answers one question. Your code is the middle column. It talks to the model on one side and to your tools on the other.

```mermaid
sequenceDiagram
    participant User
    participant Loop as Your agent loop
    participant Claude as Claude API
    participant Tools as Your tools
    User->>Loop: question
    loop until end_turn or the iteration limit
        Loop->>Claude: messages.create with system, tools, messages
        Claude-->>Loop: response with stop_reason
        alt stop_reason is tool_use
            Loop->>Tools: run each tool_use block
            Tools-->>Loop: output or error
            Loop->>Loop: append assistant turn and tool_result blocks
        else stop_reason is end_turn
            Loop-->>User: final answer
        end
    end
```

Two things to notice. The model is stateless: each request resends the whole conversation, which grows with every tool result. And the loop can only end in three ways: the model says it's done, something unexpected stops it, or your iteration guard trips.

## Setup

In your `course-buddy` project:

```bash
uv add anthropic
```

`ANTHROPIC_API_KEY` should already be in your `.env` from [Module 0](../00-orientation/03-set-up-your-workbench.md). Point `COURSE_DIR` at your local copy of this course.

## The code

Save this as `agent.py`. It's about 120 lines, and every line is doing work.

```python
import json
import sys
from pathlib import Path

import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from the environment
MODEL = "claude-sonnet-5-5"
COURSE_DIR = Path("../zth-ai-engineer")  # wherever you cloned the course
MAX_ITERATIONS = 10

SYSTEM = """You are Course Buddy, a study assistant for the Zero to Hero: AI Engineer course.
Answer only from the course files, which you reach through your tools.
Cite the lesson paths you used. If the course doesn't cover something, say so."""

TOOLS = [
    {
        "name": "list_modules",
        "description": "List every module folder and the lesson files inside it. "
        "Use this to find where a topic lives before reading.",
        "input_schema": {"type": "object", "properties": {}},
    },
    {
        "name": "search_course",
        "description": "Keyword search over all lessons. Returns up to 8 matching lines, "
        "each prefixed with its file path. Use short, specific keywords.",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string", "description": "Keywords to look for"}},
            "required": ["query"],
        },
    },
    {
        "name": "read_lesson",
        "description": "Read the full Markdown of one lesson. `path` is relative to the "
        "course root, e.g. '08-agents/01-what-makes-an-agent.md'.",
        "input_schema": {
            "type": "object",
            "properties": {"path": {"type": "string"}},
            "required": ["path"],
        },
    },
]


# ---------- tool implementations ----------

def lesson_files() -> list[Path]:
    return sorted(COURSE_DIR.glob("[0-9][0-9]-*/*.md"))


def list_modules() -> str:
    modules: dict[str, list[str]] = {}
    for f in lesson_files():
        modules.setdefault(f.parent.name, []).append(f.name)
    return json.dumps(modules, indent=1)


def search_course(query: str) -> str:
    # Simple keyword scoring. Swap in your Chroma search from Module 6 for better recall.
    words = query.lower().split()
    hits = []
    for f in lesson_files():
        for line in f.read_text().splitlines():
            score = sum(w in line.lower() for w in words)
            if score:
                hits.append((score, f"{f.relative_to(COURSE_DIR)}: {line.strip()[:200]}"))
    hits.sort(key=lambda h: -h[0])
    return "\n".join(text for _, text in hits[:8]) or "No matches."


def read_lesson(path: str) -> str:
    file = (COURSE_DIR / path).resolve()
    if not file.is_relative_to(COURSE_DIR.resolve()) or file.suffix != ".md":
        raise ValueError(f"Not a lesson file: {path}")
    return file.read_text()


HANDLERS = {"list_modules": list_modules, "search_course": search_course, "read_lesson": read_lesson}


def run_tool(name: str, args: dict) -> tuple[str, bool]:
    """Run one tool call. Returns (output, is_error). Never raises."""
    handler = HANDLERS.get(name)
    if handler is None:
        return f"Unknown tool: {name}", True
    try:
        return handler(**args), False
    except Exception as e:  # report the failure to the model instead of crashing
        return f"{type(e).__name__}: {e}", True


# ---------- the agent loop ----------

def run_agent(question: str) -> str:
    messages = [{"role": "user", "content": question}]

    for _ in range(MAX_ITERATIONS):
        response = client.messages.create(
            model=MODEL,
            max_tokens=16000,
            system=SYSTEM,
            tools=TOOLS,
            messages=messages,
        )
        # Keep the whole assistant turn, tool_use blocks included.
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason == "end_turn":
            return "".join(b.text for b in response.content if b.type == "text")
        if response.stop_reason != "tool_use":
            # max_tokens, refusal, ...: stop rather than pretend we have an answer
            raise RuntimeError(f"Agent stopped early: {response.stop_reason}")

        tool_results = []
        for block in response.content:
            if block.type == "tool_use":
                print(f"  -> {block.name}({json.dumps(block.input)})", file=sys.stderr)
                output, is_error = run_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,  # must match the tool_use block
                    "content": output,
                    "is_error": is_error,
                })
        # All results for this turn go back in ONE user message.
        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError(f"No answer after {MAX_ITERATIONS} iterations")


if __name__ == "__main__":
    question = " ".join(sys.argv[1:]) or "What is the difference between a workflow and an agent?"
    print(run_agent(question))
```

Run it:

```bash
uv run --env-file .env agent.py "Which lessons talk about prompt caching, and what do they say?"
```

You'll see the trajectory on stderr (`-> search_course(...)`, `-> read_lesson(...)`) before the answer prints.

## What each guard is for

| Situation | What the code does | Why |
|---|---|---|
| `stop_reason == "tool_use"` | Runs every `tool_use` block, sends all results back, loops | The normal working state |
| `stop_reason == "end_turn"` | Returns the text blocks | The model decided it's done |
| `max_tokens` | Raises | The answer, or a tool call, was cut off. Raise `max_tokens` or ask for shorter output |
| `refusal` | Raises | The model declined. Show a safe message, don't retry blindly |
| Unknown tool name | Returns an `is_error` result | Models occasionally invent tools. Let them correct course |
| Tool raises | Returns an `is_error` result with the message | One bad path shouldn't kill the run |
| Loop runs 10 times | Raises | Caps cost and stops an agent stuck repeating itself |

> [!NOTE]
> If you add Anthropic's server-side tools (web search, code execution), you may also see `stop_reason == "pause_turn"`, meaning the server paused a long turn. Append the assistant content and call again to let it continue. With only your own tools, as here, it won't happen.

`MAX_ITERATIONS` is the simplest budget. Real agents usually add a token or dollar budget too (sum `response.usage` across iterations) and a wall-clock timeout. Choose the limit by looking at real trajectories: if good runs take 3 to 5 steps, 10 is a sensible ceiling.

**Check yourself:** what breaks if you append only the text of the assistant turn instead of `response.content`?

<details><summary>Answer</summary>

The next request fails. Each `tool_result` must answer a `tool_use` block with the same id in the previous assistant message. Drop those blocks and the API rejects the conversation.

</details>

## Or let the SDK run the loop

Once you understand the loop, you don't always have to write it. The `anthropic` Python SDK ships a beta **tool runner** that builds the tool schemas from type hints and docstrings and runs the loop for you:

```python
from anthropic import Anthropic, beta_tool

client = Anthropic()

@beta_tool
def read_lesson(path: str) -> str:
    """Read the full Markdown of one lesson.

    Args:
        path: Path relative to the course root, e.g. '08-agents/01-what-makes-an-agent.md'.
    """
    ...

runner = client.beta.messages.tool_runner(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    tools=[read_lesson],
    messages=[{"role": "user", "content": "Summarize the first agents lesson."}],
)
for message in runner:  # one message per loop iteration
    print(message)
```

Hand-rolling still pays off when you need custom control: approval prompts, budgets, tracing each step, or swapping providers. The [agent SDKs](04-agent-sdks.md) in the next lesson go further.

> [!TIP]
> **Try it.** Run three experiments and watch the stderr trajectory each time.
> 1. Ask something the course doesn't cover ("How do I fine-tune Llama on a TPU?"). Does Course Buddy admit it, or make something up? Tighten `SYSTEM` if needed.
> 2. Set `MAX_ITERATIONS = 1` and ask a question that needs a search and a read. Confirm the guard fires.
> 3. Ask Course Buddy to "read the file `../../.env`". Check that `read_lesson` refuses and that the model gets an `is_error` result rather than your secrets.

<details><summary>What you should see</summary>

1. With a good system prompt the agent searches, finds nothing relevant, and says the course doesn't cover it. If it answers from general knowledge, add a line such as "If your tools return nothing relevant, say you couldn't find it in the course."
2. `RuntimeError: No answer after 1 iterations`. The first response is a `tool_use`, so the loop never reaches `end_turn`.
3. The trajectory shows `-> read_lesson({"path": "../../.env"})`, the tool returns `ValueError: Not a lesson file`, and the model explains it can only read lessons. This check in code, not the prompt, is what protects you.

</details>

## Key takeaways
- An agent is a `for` loop around `messages.create`: append the assistant turn, run each `tool_use`, append the `tool_result`s, repeat.
- Branch on `stop_reason`: `tool_use` continues, `end_turn` finishes, anything else is a problem to surface.
- Always cap iterations (and ideally tokens and time). Agents can loop.
- Turn tool failures into `is_error` results so the model can recover, and enforce safety checks in your tool code.
- The SDK's tool runner automates this loop once you no longer need to own it.

## Go deeper
- [How to implement tool use (Claude docs)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use): the full request and response shapes
- [Tool runner (Claude docs)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-runner): the SDK helper that runs the loop for you
- [Building effective agents (Anthropic)](https://www.anthropic.com/engineering/building-effective-agents): why simple loops with good tools beat complex frameworks

---
[← Tools and the ReAct loop](02-tools-and-the-react-loop.md) · [Module home](README.md) · [Agent SDKs →](04-agent-sdks.md)
