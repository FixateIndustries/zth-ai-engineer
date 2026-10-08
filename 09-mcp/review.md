# Module 9 review: Model Context Protocol (MCP)

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. What problem does MCP solve, in one sentence?

<details><summary>Answer</summary>

It replaces N × M custom integrations between AI applications and tools or data sources with one standard protocol: each source is wrapped once as a server, and any MCP-capable host can use it. See [Why MCP](01-why-mcp.md).

</details>

### 2. Match each primitive to who controls it: tools, resources, prompts.
a) The user picks it  b) The model decides to call it  c) The application decides to attach it

<details><summary>Answer</summary>

Tools: **b** (model-controlled). Resources: **c** (application-controlled). Prompts: **a** (user-controlled, often as slash commands). See [What a server can offer](01-why-mcp.md#what-a-server-can-offer).

</details>

### 3. Claude Code is connected to a local filesystem server and a remote Sentry server. Identify the host, the clients, and the servers.

<details><summary>Answer</summary>

Host: Claude Code. Clients: two, both inside Claude Code, one per server. Servers: the filesystem server (local, stdio) and the Sentry server (remote, Streamable HTTP). See [Three participants](02-mcp-architecture.md#three-participants).

</details>

### 4. Which layer defines `tools/call`, and which defines how the bytes travel?

<details><summary>Answer</summary>

The **data layer** (JSON-RPC 2.0 messages and primitives) defines `tools/call`. The **transport layer** (stdio or Streamable HTTP) defines how messages are framed and delivered, plus connection and auth. See [Two layers](02-mcp-architecture.md#two-layers).

</details>

### 5. You're building a server that reads a team's shared Postgres database and should be usable by 20 engineers' Claude Code sessions. stdio or Streamable HTTP?

<details><summary>Answer</summary>

Streamable HTTP. One shared, authenticated server serves many clients and keeps the database credentials in one place. With stdio, every engineer would run their own copy with their own credentials, which works but is harder to manage. See [the transport table](02-mcp-architecture.md#the-transport-layer-how-it-travels).

</details>

### 6. What's wrong with this stdio server?

```python
from mcp.server import MCPServer

mcp = MCPServer("notes")

@mcp.tool()
def read_note(name: str) -> str:
    print(f"reading {name}")
    return open(f"notes/{name}").read()

if __name__ == "__main__":
    mcp.run()
```

<details><summary>Answer</summary>

Two problems:
1. `print()` writes to stdout, which is the protocol channel in stdio. It corrupts the stream. Log to stderr.
2. `name` isn't validated, so `../../.ssh/id_rsa` reads files outside `notes/`. Resolve the path, check it stays inside the folder, and raise `ToolError` with a helpful message otherwise.

There's also a missing docstring, so the model gets no description of the tool. See [Build an MCP server](03-build-an-mcp-server.md).

</details>

### 7. Write the command to register a local server launched with `uv run --directory /srv/cb server.py`, named `course-buddy`, with the env var `COURSE_DIR=/srv/course`, in Claude Code.

<details><summary>Answer</summary>

```bash
claude mcp add course-buddy -e COURSE_DIR=/srv/course -- uv run --directory /srv/cb server.py
```

Everything after `--` is the server's launch command. See [Register it with Claude Code](03-build-an-mcp-server.md#register-it-with-claude-code).

</details>

### 8. How do you make a server available to everyone who clones your repo?

<details><summary>Answer</summary>

Add it with `--scope project`. Claude Code writes it to `.mcp.json` at the repo root, which you commit. (Keep secrets out of that file. Use env vars or per-user config for credentials.) See [lesson 3](03-build-an-mcp-server.md#register-it-with-claude-code).

</details>

### 9. In your own Python host, what's the difference between `Client(StdioServerParameters(command="uv", args=["run", "server.py"]))` and `Client("https://mcp.example.com/mcp")`?

<details><summary>Answer</summary>

The first launches the server as a local subprocess and talks to it over stdio. Your app owns its lifecycle. The second connects to an already-running remote server over Streamable HTTP. See [Build an MCP client](04-build-an-mcp-client.md).

</details>

### 10. To feed MCP tools to Claude, what do you map, and to what?

<details><summary>Answer</summary>

Each MCP tool's `name`, `description`, and `input_schema` become a Claude tool definition with the same fields. Each `tool_use` block from Claude becomes `client.call_tool(name, input)`, and the result's text content and `is_error` become a `tool_result`. See [Make Course Buddy an MCP host](04-build-an-mcp-client.md#make-course-buddy-an-mcp-host).

</details>

### 11. Scenario: after connecting a popular third-party MCP server, your agent starts emailing summaries to an unknown address. What likely happened, and what do you change?

<details><summary>Answer</summary>

Prompt injection: text returned by the server (or by data it fetched) contained instructions, and the model followed them with another tool it had access to. Fixes: only connect trusted servers, remove or gate write tools (like sending email) behind human confirmation, limit which tools are available together, and log tool calls so you notice. See the warnings in [lesson 4](04-build-an-mcp-client.md) and [Module 13](../13-safety-and-ethics/02-prompt-injection.md).

</details>

## Hands-on challenge

Make your `host.py` from lesson 4 connect to **two** servers at once: your Course Buddy server over stdio, and a second server of your choice (for example `uvx mcp-server-fetch` over stdio, so Course Buddy can also fetch the official docs linked in "Go deeper"). Merge both tool lists for Claude, and route each `tool_use` to the right client. Ask a question that needs both: *"Summarize what lesson 2 of the MCP module says about transports, then check the official spec page and tell me if anything is missing."*

<details><summary>Hint</summary>

Use `contextlib.AsyncExitStack` to enter several `Client(...)` contexts in one function. Build `routes: dict[str, Client] = {}` while listing tools from each client, and look up `routes[block.name]` when a `tool_use` arrives. If two servers expose the same tool name, rename one when you build the Claude tool list (for example `fetch__fetch`) and keep the original name for `call_tool`.

</details>

## Capstone checkpoint
- [ ] Course Buddy's `search_course`, `read_lesson`, and `list_modules` run in an MCP server (`server.py`) built with the `mcp` SDK
- [ ] The server validates paths, raises `ToolError` for bad input, and never prints to stdout
- [ ] It's registered in an MCP host (Claude Code via `claude mcp add`, or Claude Desktop) and answers course questions there
- [ ] Optional: the server also runs over Streamable HTTP, and `host.py` uses it as an MCP client from your own agent loop

---
[← Last lesson](04-build-an-mcp-client.md) · [Course home](../README.md) · [Next module →](../10-multimodal/README.md)
