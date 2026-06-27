# Network Access

**Claude Code allows unrestricted outbound network access by default; Codex CLI blocks all network access by default — this single architectural difference shapes how each tool handles MCP integrations, package installs, and API calls made by agent-generated code.**

## Default posture

The two tools occupy opposite ends of the safety-vs-friction spectrum:

- **Claude Code** — no network sandbox. Agent-spawned processes can open arbitrary sockets, call external APIs, fetch packages, or exfiltrate data. Security is enforced at the application layer through permission prompts for shell commands, not at the OS level for network traffic.
- **Codex CLI** — network disabled inside its macOS Seatbelt / Linux bubblewrap sandbox, which is on by default. Outbound connections from the agent process and any subprocess it spawns are blocked at the kernel level. The developer opts in to network access by disabling the sandbox flag.

This is an architectural inversion: Codex CLI secures at the kernel; Claude Code secures at the application.

## Enabling and scoping network access

**Claude Code** has no built-in mechanism to restrict network access short of enabling the optional OS-level sandbox. With the sandbox enabled, the tool itself can still reach Anthropic's API — the sandbox carves out that one connection — but all other outbound traffic from subprocesses is blocked.

**Codex CLI** exposes a `--full-auto` flag that runs without a sandbox, restoring unrestricted network access. Teams that need package installs during agentic runs typically use this mode combined with a locked-down Docker environment as the outer boundary.

## MCP servers and network

MCP servers are the primary mechanism through which both tools reach external services — Slack, Google Drive, databases, internal APIs. Each MCP server runs as a separate process with its own network access, independent of what the agent sandbox permits.

- MCP servers authenticate via OAuth 2.0; tokens are stored in the system keychain.
- Adding an MCP server to Claude Code does not automatically grant it bypass permissions — the user still controls which servers are trusted.
- In multi-agent workflows, subagents inherit the parent's MCP connections, meaning a single compromised or malicious MCP server has access across all agents in the run.

## What this means in practice

| Scenario | Claude Code | Codex CLI |
|---|---|---|
| `npm install` in a task | Succeeds (network open) | Blocked by default |
| Agent calls external API | Succeeds | Blocked by default |
| MCP server reaches Slack | Works; governed by MCP trust | Works; governed by MCP trust |
| Sandbox mode network | Anthropic API only | Disabled entirely |

When a Claude Code project enables the sandbox, `fetch` calls and arbitrary socket connections from subprocesses fail silently unless the target host is explicitly allow-listed. There is no built-in log of blocked connections, so debugging requires inspecting subprocess exit codes and error output directly.

## Practical guidance

- If your Claude Code tasks include `npm install`, `pip install`, `curl`, or any HTTP call in generated code, expect those to break the moment you enable the sandbox. Audit tool-invoked commands before enabling.
- Codex CLI's default of network-off is the safer posture for untrusted or unfamiliar codebases. Use `--full-auto` only when you control the outer environment.
- Treat MCP server network access as a separate trust surface from the agent's own network access. An agent that cannot open a socket can still exfiltrate data through an authorized MCP server.

---
*See also: [File System Access](file-system-access.md), [Claude Code Permission and Trust Model](permission-trust-model.md), [MCP Integration](mcp-integration.md)*
