# Build an MCP server

> Module 9 · Lesson 3 · ~20 min

**Goals**
- Build a Python MCP server with the official `mcp` SDK, exposing a tool, a resource, and a prompt
- Test it in the MCP Inspector
- Register it with Claude Code over stdio, and serve it over Streamable HTTP

## The SDK in one minute

The official Python SDK is the `mcp` package. Its high-level server class, `MCPServer`, turns ordinary type-hinted functions into MCP tools, resources, and prompts. It generates the JSON Schema from your type hints and the description from your docstring, and it handles the protocol and transports for you.

> [!NOTE]
> In version 1 of the SDK this class was called **FastMCP** (`from mcp.server.fastmcp import FastMCP`), and most tutorials online still use that name. Version 2 renamed it to `MCPServer` (`from mcp.server import MCPServer`). The decorators work the same way. This course uses v2.

In your `course-buddy` project:

```bash
uv add "mcp[cli]"
```

The `[cli]` extra adds the `mcp` command, which you'll use to launch the Inspector.

## The Course Buddy server

You already wrote the tool logic in [Module 8](../08-agents/03-build-an-agent-from-scratch.md). Now you'll wrap it in a server instead of hard-wiring it into one agent. Save this as `server.py`:

```python
import json
import os
from pathlib import Path

from mcp.server import MCPServer
from mcp.server.mcpserver.exceptions import ToolError

COURSE_DIR = Path(os.environ.get("COURSE_DIR", "../zth-ai-engineer")).resolve()

mcp = MCPServer("course-buddy", instructions="Tools for reading the Zero to Hero: AI Engineer course.")


def lesson_files() -> list[Path]:
    return sorted(COURSE_DIR.glob("[0-9][0-9]-*/*.md"))


@mcp.tool()
def list_modules() -> str:
    """List every module folder and the lesson files inside it."""
    modules: dict[str, list[str]] = {}
    for f in lesson_files():
        modules.setdefault(f.parent.name, []).append(f.name)
    return json.dumps(modules, indent=1)


@mcp.tool()
def search_course(query: str) -> str:
    """Keyword search over all lessons. Returns up to 8 matching lines with file paths."""
    words = query.lower().split()
    hits = []
    for f in lesson_files():
        for line in f.read_text().splitlines():
            score = sum(w in line.lower() for w in words)
            if score:
                hits.append((score, f"{f.relative_to(COURSE_DIR)}: {line.strip()[:200]}"))
    hits.sort(key=lambda h: -h[0])
    return "\n".join(text for _, text in hits[:8]) or "No matches."


@mcp.tool()
def read_lesson(path: str) -> str:
    """Read one lesson. `path` is relative to the course root, e.g. '09-mcp/01-why-mcp.md'."""
    file = (COURSE_DIR / path).resolve()
    if not file.is_relative_to(COURSE_DIR) or file.suffix != ".md" or not file.exists():
        raise ToolError(f"Not a lesson file: {path}. Use list_modules to see valid paths.")
    return file.read_text()


@mcp.resource("course://syllabus")
def syllabus() -> str:
    """The course home page: the module list and how the course works."""
    return (COURSE_DIR / "README.md").read_text()


@mcp.prompt()
def quiz_me(module: str) -> str:
    """Quiz me on one module of the course."""
    return (
        f"Read the lessons in the '{module}' module with the course tools, then ask me "
        "3 questions about them, one at a time. Wait for my answer before the next one."
    )


if __name__ == "__main__":
    mcp.run()  # stdio by default
```

A few details worth noticing:

- **Each decorator maps to a primitive.** `@mcp.tool()` becomes something the model can call, `@mcp.resource(uri)` becomes readable data, and `@mcp.prompt()` becomes a user-selectable template.
- **The docstring is the tool description.** Everything you learned about [tool design](../08-agents/02-tools-and-the-react-loop.md#designing-tools-a-model-can-use) applies here.
- **`ToolError` is for expected failures.** The SDK turns it into a result with `isError: true` and your message, so the model can recover. Any other exception also becomes an error result, but with a generic message, and the details stay in the server log.
- **Config comes from the environment.** The host launches the server, so `COURSE_DIR` is passed as an env var rather than hard-coded.
- **No `print()`.** Over stdio, stdout is the protocol channel. If you need logs, write to stderr.

## Test it in the Inspector

Before you plug a server into a host, poke at it directly:

```bash
COURSE_DIR=/path/to/zth-ai-engineer uv run mcp dev server.py
```

This launches the MCP Inspector in your browser, connected to your server over stdio. List the tools, call `search_course` with `"vector database"`, read the `course://syllabus` resource, and render the `quiz_me` prompt. Then call `read_lesson` with `../../.env` and confirm you get an error result, not a file.

## Register it with Claude Code

Claude Code is an MCP host. Add your server with `claude mcp add`. Everything after `--` is the command Claude Code runs to launch it:

```bash
claude mcp add course-buddy \
  -e COURSE_DIR=/path/to/zth-ai-engineer \
  -- uv run --directory /path/to/course-buddy server.py
```

Use absolute paths, because Claude Code may start the server from a different working directory. Then:

```bash
claude mcp list        # shows course-buddy and whether it connected
```

Start `claude`, type `/mcp` to see the server and its tools, and ask: *"Using the course-buddy tools, which lessons explain the MCP transport layer?"* Claude Code will ask permission the first time it calls each tool. Prompts show up as slash commands: type `/` and look for `course-buddy:quiz_me`, or run `/mcp__course-buddy__quiz_me 08-agents` directly.

By default the server is registered for you alone, in the current project (`--scope local`). Add `--scope project` to write it to a `.mcp.json` file in the repo that your team can commit, or `--scope user` to make it available in all your projects:

```json
{
  "mcpServers": {
    "course-buddy": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/course-buddy", "server.py"],
      "env": {"COURSE_DIR": "/path/to/zth-ai-engineer"}
    }
  }
}
```

Claude Desktop uses the same `mcpServers` shape in its own config file (Settings, then Developer, then Edit Config).

## Serve it remotely over Streamable HTTP

To share one server across machines or teammates, run it over HTTP instead of stdio. Change the last line:

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http", port=8000)  # serves http://127.0.0.1:8000/mcp
```

Start it with `uv run server.py`, then register it by URL:

```bash
claude mcp add --transport http course-buddy-http http://127.0.0.1:8000/mcp
```

> [!WARNING]
> An HTTP server is reachable by anything that can reach its port. Before you expose one beyond `localhost`, add authentication. The spec recommends OAuth, and the SDK supports it. Hosts pass headers with `--header "Authorization: Bearer ..."`. Also remember that every tool is an action an attacker can trigger via prompt injection, so keep remote servers read-only unless you really need writes.

**Check yourself:** your server works in the Inspector but Claude Code shows it as failed to connect. You recently added `print("loaded", len(lesson_files()), "lessons")` at startup. What's wrong?

<details><summary>Answer</summary>

Over stdio, that `print` writes non-JSON text to stdout, which is the protocol channel, and the client can't parse it. Send it to stderr instead: `print(..., file=sys.stderr)`. (A relative `COURSE_DIR` or a wrong `--directory` path are the other usual suspects. Use `claude mcp get course-buddy` to check the exact command.)

</details>

> [!TIP]
> **Try it.** Add a fourth tool, `lesson_outline(path)`, that returns only the Markdown headings (lines starting with `#`) of a lesson, so the model can skim before reading the whole thing. Restart the server in the Inspector to check it. Then, in Claude Code, ask a question and see whether Claude uses the outline tool before `read_lesson`. If it doesn't, improve the docstring.

<details><summary>Solution</summary>

```python
@mcp.tool()
def lesson_outline(path: str) -> str:
    """Return just the headings of a lesson. Cheaper than read_lesson. Use it to
    check whether a lesson is relevant before reading it in full."""
    text = read_lesson(path)  # reuses the path checks
    return "\n".join(line for line in text.splitlines() if line.startswith("#"))
```

Claude Code picks up new tools when the server restarts. Run `/mcp` to reconnect, or restart the session.

</details>

## Key takeaways
- The `mcp` SDK's `MCPServer` (FastMCP in v1) turns type-hinted, documented functions into tools, resources, and prompts.
- Raise `ToolError` for failures the model should see, validate inputs in code, and never print to stdout in a stdio server.
- Test with `mcp dev` (the Inspector) before connecting a host.
- `claude mcp add <name> -- <command>` registers a local stdio server. `claude mcp add --transport http <name> <url>` registers a remote one. Scopes control who gets it.
- Remote HTTP servers need authentication, and should stay read-only unless writes are essential.

## Go deeper
- [MCP Python SDK docs](https://py.sdk.modelcontextprotocol.io/): servers, context, auth, and transports in v2
- [Build an MCP server (MCP docs)](https://modelcontextprotocol.io/docs/develop/build-server): the official walkthrough, in several languages
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp): every `claude mcp` option, scopes, and auth
- [MCP security best practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices): what to watch for when you expose a server

---
[← MCP architecture](02-mcp-architecture.md) · [Module home](README.md) · [Build an MCP client →](04-build-an-mcp-client.md)
