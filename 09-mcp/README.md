# Module 9: Model Context Protocol (MCP)

In Module 8 you hard-wired three tools into one agent. The Model Context Protocol (MCP) is an open standard that lets you write those tools once, as a **server**, and plug them into any AI application that speaks MCP: Claude Code, Claude Desktop, IDEs, your own agents. In this module you'll learn how the protocol is put together, build a server, and write clients that connect to local and remote servers.

**You'll be able to:**
- Explain the integration problem MCP solves and when it's worth using
- Describe hosts, clients, and servers, plus the data layer and transport layer
- Tell tools, resources, and prompts apart
- Build an MCP server in Python and register it with Claude Code
- Write a client that connects to a local stdio server and a remote Streamable HTTP server, and feed its tools to Claude

**Prerequisites:** [Module 8: AI Agents](../08-agents/README.md), especially [Build an agent from scratch](../08-agents/03-build-an-agent-from-scratch.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [Why MCP](01-why-mcp.md) | ~10 min |
| 2 | [MCP architecture](02-mcp-architecture.md) | ~15 min |
| 3 | [Build an MCP server](03-build-an-mcp-server.md) | ~20 min |
| 4 | [Build an MCP client](04-build-an-mcp-client.md) | ~20 min |
| ✓ | [Module review](review.md) | ~10 min |

## Capstone checkpoint

You'll move Course Buddy's tools (`search_course`, `read_lesson`, `list_modules`) into an MCP server, register it with an MCP host such as Claude Code or Claude Desktop, and ask questions about the course from there. See the [capstone milestones](../capstone/README.md).

---
[← Previous module](../08-agents/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-why-mcp.md)
