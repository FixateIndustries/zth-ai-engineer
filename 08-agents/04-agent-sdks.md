# Agent SDKs

> Module 8 · Lesson 4 · ~15 min

**Goals**
- Know what an agent SDK adds on top of the loop you wrote
- Recognize the main options: OpenAI Agents SDK (and AgentKit), Claude Agent SDK, Google ADK, and Google Cloud's managed agent platform
- Choose between hand-rolled, SDK, and managed hosting for a given project

## What a framework actually buys you

Your [from-scratch agent](03-build-an-agent-from-scratch.md) already does the core job. SDKs earn their place when you need the parts around the loop:

- **Tool plumbing.** Schemas generated from type hints, input validation, async and parallel execution.
- **Multi-agent wiring.** Handoffs, sub-agents, agents as tools.
- **Guardrails and approvals.** Checks on inputs and outputs, human approval before risky tools.
- **Sessions and memory.** Conversation history stored and resumed for you.
- **Tracing.** Every model call and tool call recorded for debugging ([Module 12](../12-observability/01-tracing-and-logging.md)).
- **Built-in tools.** File editing, shell, web search, code execution.

The cost is another abstraction between you and the API. When a run misbehaves you need to know what prompt the framework actually sent. Write one agent by hand first (you just did), then reach for an SDK when you need what it adds.

## OpenAI Agents SDK

A lightweight Python (and JS/TS) library built around a few primitives: **agents** (an LLM with instructions and tools), **handoffs** and **agents as tools** for delegation, **guardrails**, **sessions**, and built-in **tracing**. It's provider-agnostic: it defaults to OpenAI models but can drive others.

```bash
uv add openai-agents   # needs OPENAI_API_KEY
```

```python
from agents import Agent, Runner, function_tool

@function_tool
def list_modules() -> list[str]:
    """List the course's module folders."""
    return ["00-orientation", "01-how-llms-work", "08-agents"]

agent = Agent(
    name="Course Buddy",
    instructions="Answer questions about the course using your tools.",
    tools=[list_modules],
)

result = Runner.run_sync(agent, "Which modules are there?")
print(result.final_output)
```

`Runner` runs the same loop you wrote: call the model, execute tools, repeat until there's a final output. Delegation is one argument: `Agent(name="Triage", handoffs=[billing_agent, refund_agent])`.

**AgentKit** is OpenAI's name for a bundle of hosted products around this SDK: **Agent Builder** (a visual canvas for multi-step workflows), **ChatKit** (an embeddable chat UI for agents), a connector registry, and evaluation tooling.

> [!WARNING]
> OpenAI has deprecated Agent Builder and scheduled it to shut down on November 30, 2026. Existing workflows can be exported as Agents SDK code. For new work, use the Agents SDK directly. ChatKit remains available as a front end.

## Claude Agent SDK

The Claude Agent SDK is **Claude Code packaged as a library** (Python and TypeScript). Instead of you defining every tool, it ships Claude Code's agent loop, context management, and built-in tools: `Read`, `Write`, `Edit`, `Bash`, `Glob`, `Grep`, `WebSearch`, `WebFetch`, and more. Add permissions, hooks, sub-agents, sessions, and MCP servers on top.

```bash
uv add claude-agent-sdk   # needs ANTHROPIC_API_KEY
```

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def main():
    async for message in query(
        prompt="Find every lesson that mentions MCP and list their file paths.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep"],  # read-only, auto-approved
            cwd="../zth-ai-engineer",
        ),
    ):
        if isinstance(message, ResultMessage):
            print(message.result)

asyncio.run(main())
```

Notice what you didn't write: no search tool, no file reader. The SDK's built-in tools did it. You add your own tools with the `@tool` decorator wrapped in an in-process MCP server (`create_sdk_mcp_server`), or by connecting any [MCP server](../09-mcp/README.md).

Reach for it when your agent works with files, code, or a shell. For a narrow agent with only your own APIs as tools, the plain Messages API loop (or the SDK's tool runner) is often simpler.

> [!NOTE]
> Don't confuse three Anthropic options. The **tool runner** in the `anthropic` SDK automates the loop for tools you define. The **Claude Agent SDK** adds Claude Code's built-in tools and harness, and you host it. **Claude Managed Agents** is a hosted API where Anthropic runs the loop and a sandbox for you.

## Google Agent Development Kit (ADK)

ADK is Google's open-source, code-first agent framework (Python, Java, Go, TypeScript). It's optimized for Gemini but model-agnostic, and it was designed for multi-agent systems: an agent can have `sub_agents`, and there are workflow agents (sequential, parallel, loop) for fixed orchestration.

```bash
uv add google-adk   # ADK reads GOOGLE_API_KEY (a Gemini API key) or Google Cloud credentials
```

```python
# course_buddy/agent.py
from google.adk.agents.llm_agent import Agent

def list_modules() -> dict:
    """Lists the course's module folders."""
    return {"modules": ["00-orientation", "01-how-llms-work", "08-agents"]}

root_agent = Agent(
    model="gemini-flash-latest",
    name="course_buddy",
    description="Answers questions about the course.",
    instruction="Answer questions about the course using your tools.",
    tools=[list_modules],
)
```

ADK finds `root_agent` by convention. Run it with `adk run course_buddy` in the terminal, or `adk web` for a local dev UI that shows every event and tool call.

## Google Cloud's managed agent platform (formerly Vertex AI Agent Builder)

Google's managed agent offering has been renamed several times, so you'll see older names in tutorials. What was **Vertex AI Agent Builder** now lives in the **Gemini Enterprise Agent Platform**, and the deployment runtime once called **Vertex AI Agent Engine** is now **Agent Runtime**.

It's not a separate framework. It's where you **deploy and operate** agents, typically ones built with ADK (other frameworks are supported too). It gives you managed hosting and scaling, plus sessions, a long-term memory bank, code execution, evaluation, and governance (agent identity, registry, policies). Choose it when your company already runs on Google Cloud and you want managed infrastructure instead of running agent servers yourself.

## Choosing

| Option | What it is | Pick it when |
|---|---|---|
| Hand-rolled loop | ~100 lines on the raw API | You want full control, few dependencies, or you're learning |
| OpenAI Agents SDK | Lightweight multi-agent framework | You want handoffs, guardrails, and tracing in plain Python |
| Claude Agent SDK | Claude Code's harness as a library | Your agent reads and edits files, runs commands, or browses |
| Google ADK | Code-first, multi-agent framework | You're on Gemini or Google Cloud, or want workflow agents built in |
| Gemini Enterprise Agent Platform / Agent Runtime | Managed hosting and ops on Google Cloud | You need managed deployment, memory, and governance at scale |

> [!TIP]
> **Try it.** Port your Course Buddy agent from [lesson 3](03-build-an-agent-from-scratch.md) to one SDK of your choice. Keep the same three tools and the same system prompt. Then compare: how many lines did you delete? What can you no longer see or control (the exact request, the iteration limit, error handling)? Find where the SDK sets its maximum number of turns.

<details><summary>Hints</summary>

- OpenAI Agents SDK: decorate each tool function with `@function_tool`, and look at the `max_turns` argument to `Runner.run_sync`.
- Claude Agent SDK: you may not need your tools at all. Try `allowed_tools=["Read", "Glob", "Grep"]` with `cwd` set to the course folder and compare the answers. `ClaudeAgentOptions` also has a `max_turns` option.
- ADK: plain Python functions with docstrings become tools. Use `adk web` to watch the trajectory.

</details>

## Key takeaways
- SDKs add tool plumbing, delegation, guardrails, sessions, and tracing around the same loop you wrote.
- The OpenAI Agents SDK is a lightweight multi-agent framework. AgentKit's visual Agent Builder is being shut down, so build on the SDK.
- The Claude Agent SDK gives you Claude Code's built-in tools and harness, which is best for agents that work with files and commands.
- Google ADK is a code-first framework. The Gemini Enterprise Agent Platform (formerly Vertex AI Agent Builder) with Agent Runtime (formerly Agent Engine) is where you deploy and operate agents on Google Cloud.
- Learn the raw loop first, then adopt an SDK for what it adds, not by default.

## Go deeper
- [OpenAI Agents SDK docs](https://openai.github.io/openai-agents-python/): agents, handoffs, guardrails, tracing
- [Claude Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview): capabilities, and how it compares with the client SDK and Managed Agents
- [Agent Development Kit docs](https://adk.dev/): quickstarts, multi-agent patterns, deployment
- [Gemini Enterprise Agent Platform overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/overview): Google Cloud's build, scale, and govern stack for agents

---
[← Build an agent from scratch](03-build-an-agent-from-scratch.md) · [Module home](README.md) · [Multi-agent systems →](05-multi-agent-systems.md)
