# Context and Memory

**Neither Claude Code nor Codex CLI persists memory across sessions by default — every invocation starts with a blank slate, and the only durable state is what lives in files on disk.**

## What "context" means here

In both tools, the agent's context window is its working memory: everything the model can reason over at once. It holds the conversation history, tool call results, file contents read during the session, and any instructions injected at startup. When the session ends, it's gone.

This has a concrete implication: if you want the agent to remember your project conventions, you must externalize that knowledge into a file and ensure it gets loaded at the start of every session.

## How each tool handles startup context

**Claude Code** uses a layered instruction file system called CLAUDE.md:

- `~/.claude/CLAUDE.md` — user-global instructions loaded in every project
- `<project-root>/CLAUDE.md` — project-level instructions, checked into the repo
- Subdirectory `CLAUDE.md` files — scoped to that subtree, merged in when the agent descends into it
- `.claude/settings.json` carries behavioral config (permissions, tool rules) separately from instructions

The merge order is global → project → subdirectory, with more-local files able to extend or override outer ones. This is the primary persistence mechanism: you write durable context into CLAUDE.md files and the tool re-injects them every session.

**Codex CLI** uses a `AGENTS.md` convention (same idea, different filename) and also supports a `--instructions` CLI flag for session-scoped injection. There is no global merge stack as elaborate as Claude Code's; a single project-level file is the norm.

## In-session memory

Within a session, both tools maintain the full conversation as context. Claude Code tracks every tool call result, every file read, and every shell output — all of it burns context tokens. On long tasks this matters:

- Reading large files repeatedly is expensive; the agent re-reads unless instructed otherwise
- Multi-agent subagents in Claude Code each get their own fresh context window, which is why the Workflow tool routes intermediate results back through files rather than through the parent's context
- There is no cross-session memory store; "memory" is always a file or a database the agent writes explicitly

## What you actually do for persistence

For both tools, the practical pattern is the same:

- Put standing conventions, architecture decisions, and code style in CLAUDE.md / AGENTS.md
- Have the agent write findings, decisions, and task state to files (a `notes/` dir, a `TODO.md`, a structured JSON log)
- On resumption, point the agent at those files explicitly or ensure they're in the startup context
- For project-level facts that change (current sprint, live API keys, environment quirks), keep a `context.md` that the agent is instructed to read at session start

There is no magic memory layer. The persistence model is: files are memory, and the agent is responsible for writing them.

---
*See also: [Shell Execution Model](shell-execution-model.md), [Claude Code vs Codex CLI](claude-code-vs-codex-cli.md), [MCP Integration](mcp-integration.md)*
