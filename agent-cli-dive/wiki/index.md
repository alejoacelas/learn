# AI Agent CLI Tools: Claude Code and Codex CLI

A precise reference for what Claude Code and Codex CLI can and cannot do — particularly useful if you are building Mac apps or running agents in CI.

## Pages

- **[Shell Execution Model](shell-execution.md)** — non-interactive subprocesses, no TTY, no stdin passthrough, SIGTERM and background process gotchas
- **[Permission and Trust Model](permission-model.md)** — Claude Code's five-tier scope hierarchy, six permission modes, rule syntax, and what stays blocked in every mode
- **[Network Access](network-access.md)** — what reaches the network and what does not by default
- **[File System Access](file-system.md)** — write restrictions, read defaults, sandbox enforcement, and unprotected .env files
- **[Mac App Development](mac-app-dev.md)** — Keychain codesign failures, notarytool credential patterns, quarantine, .pbxproj corruption, and sandbox-exec blocking Apple Events
- **[Codex CLI Sandbox](codex-sandbox.md)** — OS-level hard sandbox and network-off defaults
- **[MCP Integration](mcp-integration.md)** — extending Claude Code with external servers, trust model, OAuth, and subagent inheritance
- **[Multi-Agent and Worktrees](multi-agent.md)** — four parallelization primitives, git-level isolation, 16-agent concurrency cap, 1,000-agent lifetime cap
- **[Context and Memory](context-memory.md)** — how context is managed across sessions and agents
- **[Claude Code vs Codex CLI](comparison.md)** — application-layer vs kernel-level security, and which tool fits which developer profile
- **[Sources](sources.md)** — primary documentation and references
