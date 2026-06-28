# Sources

Primary references used to research AI agent CLI tool internals, permission systems, sandboxing, and orchestration mechanics.

## Official Docs

- [Configure the sandboxed Bash tool — Claude Code](https://code.claude.com/docs/en/sandboxing) — Mechanism names (Seatbelt/sandbox-exec on macOS, bubblewrap+seccomp on Linux), filesystem/network isolation details, env var behavior, and subprocess scope.
- [Bash tool — Anthropic API Docs](https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool) — Explicitly states no interactive commands, no TTY, no streaming, session statefulness, and the 245-token overhead per invocation.
- [Choose a permission mode — Claude Code Docs](https://code.claude.com/docs/en/permission-modes) — All six permission modes, bypassPermissions circuit-breakers, protected paths list, auto mode classifier defaults, and what is/isn't blocked in each mode.
- [Configure permissions — Claude Code Docs](https://code.claude.com/docs/en/permissions.md) — Rule syntax (Tool(specifier)), glob patterns, deny/ask/allow semantics, Bash rule word-boundary behavior, WebFetch domain rules, and compound command handling.
- [Connect Claude Code to tools via MCP — Claude Code Docs](https://code.claude.com/docs/en/mcp) — Transports, scopes, OAuth auth, deferred loading, subagent mcpServers field, headersHelper, channels, resources, and prompts.
- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents) — Built-in agent types, MCP tool inheritance, tool restrictions, nested subagent depth limits, and the isolation:worktree frontmatter option.
- [Orchestrate subagents at scale with dynamic workflows — Claude Code Docs](https://code.claude.com/docs/en/workflows) — All workflow mechanics: agent(), parallel(), pipeline(), concurrency caps (16 concurrent / 1000 total), token budget, StructuredOutput, and resume/pause behavior.
- [Configure Server-Managed Settings — Official Docs](https://code.claude.com/docs/en/server-managed-settings) — Fetch/caching behavior, security considerations table covering tamper scenarios, forceRemoteSettingsRefresh, and bypass via third-party providers.

## GitHub

- [GHSA-5cwg-9f6j-9jvx: Insecure System-Wide Config Loading (CVE-2026-35603)](https://github.com/anthropics/claude-code/security/advisories/GHSA-5cwg-9f6j-9jvx) — Confirms Claude Code loaded managed-settings.json without ownership validation; the v2.1.75 fix moved the Windows path from ProgramData to Program Files.
- [Bash tool timeout kills Claude Code process — GitHub Issue #45717](https://github.com/anthropics/claude-code/issues/45717) — Detailed analysis of the shared process group bug causing timeouts to SIGTERM the parent process; exit code 143 and default 120s timeout confirmed.
- [Background task cleanup kills long-running processes — GitHub Issue #25188](https://github.com/anthropics/claude-code/issues/25188) — Confirms all tracked background processes are SIGTERMed on session end/context compaction; no persistent/exempt flag exists.

## Blog Posts / Writeups

- [Making Claude Code more secure and autonomous with sandboxing — Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-sandboxing) — Anthropic's official explanation of the sandbox architecture: bubblewrap/Seatbelt, filesystem policy, the network proxy approach, and the 84% permission prompt reduction stat.
- [How we built Claude Code auto mode — Anthropic Engineering](https://www.anthropic.com/engineering/claude-code-auto-mode) — Two-layer classifier architecture, false positive rates, tier system for permission decisions, and a comparison to the sandboxing approach.
- [Inside the Codex Sandbox: Platform-Specific Implementation — Daniel Vaughan](https://codex.danielvaughan.com/2026/04/08/codex-sandbox-platform-implementation/) — Most technically detailed secondary source on Codex CLI: exact SBPL file names, seccomp syscall list, Landlock kernel version requirement, and Windows sandbox user names.
