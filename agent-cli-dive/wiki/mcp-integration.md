# MCP Integration

**MCP servers are the escape hatch from Claude Code's built-in toolset — they connect the agent to any external system, but each connection is a new trust surface that demands explicit configuration.**

## What MCP servers add

Native Claude Code tools cover the local filesystem, shell, and a handful of web fetches. MCP servers extend this to [anything with an API](https://code.claude.com/docs/en/mcp):

- **Databases** — direct PostgreSQL queries, not just file reads
- **Issue trackers** — JIRA, Linear, GitHub issues and PRs
- **SaaS actions** — Slack messages, Gmail drafts, Figma design sync
- **Monitoring** — Sentry errors, Statsig experiments
- **Push channels** — CI results, Telegram messages entering the session unprompted

Popular servers: the GitHub MCP server (most-installed), `@bytebase/dbhub` for databases, `korotovsky/slack-mcp-server` (the official Slack server was archived May 2025), Playwright/Puppeteer for browser automation.

## Tool naming and permission rules

MCP tool names follow a fixed pattern: `mcp__<servername>__<toolname>`. This pattern plugs directly into the permission rule syntax:

- `mcp__puppeteer__*` — allow all tools from one server
- `mcp__*` — deny all MCP tools entirely (removes them from context)
- `mcp__github__create_issue` — allow one specific tool

MCP tools go through [the same permission layer as native tools](https://code.claude.com/docs/en/permissions). `acceptEdits` mode does **not** auto-approve MCP tools — it covers file edits only. `bypassPermissions` does auto-approve them, alongside everything else. Prefer allow-listing specific tool patterns over reaching for `bypassPermissions` for MCP access.

## Deferred tool loading

By default (`ENABLE_TOOL_SEARCH=true`), [MCP tools are deferred](https://code.claude.com/docs/en/mcp): only tool names and server instructions load at session start. Full schemas load on demand via a `ToolSearch` call backed by the `tool_reference` API feature. This cuts token usage by ~85% when large tool libraries are connected.

To override for a specific server, set `alwaysLoad: true` in its `.mcp.json` entry, or flag individual tools with `"anthropic/alwaysLoad": true` in their `_meta` object. Note: Haiku models do not support the `tool_reference` mechanism.

## Configuration scopes

MCP servers are declared at [four scopes](https://code.claude.com/docs/en/mcp):

| Scope | File | Shared? |
|---|---|---|
| User | `~/.claude.json` | No — personal, cross-project |
| Local | `~/.claude.json` (project-keyed) | No — private to current project |
| Project | `.mcp.json` (checked in) | Yes — whole team |
| Managed | `/Library/Application Support/ClaudeCode/managed-mcp.json` | Enforced — users cannot add servers |

When a managed config is present, it has exclusive control. Project-scoped servers (`.mcp.json`) prompt for approval before first use; this dialog is skipped in headless mode (`-p` flag).

## Authentication

Remote MCP servers use [OAuth 2.0](https://code.claude.com/docs/en/mcp). Claude Code detects 401/403 responses and flags the server in `/mcp` for a browser OAuth flow. Tokens are stored in the macOS Keychain; a credentials file is used on Windows and Linux. Tokens refresh automatically. Static Bearer tokens can also be passed as a header in config.

The `headersHelper` config field runs arbitrary shell commands at connection time to generate auth headers dynamically (10-second timeout, user environment access).

## MCP resources, prompts, and channels

Beyond tools, MCP servers expose three additional capability types:

- **Resources** — data addressable via `@server:protocol://path` syntax in prompts
- **Prompts** — pre-defined templates that become `/mcp__servername__promptname` slash commands
- **Channels** — push-based event streams that inject external events into the session

## Subagents and MCP

[Subagents inherit MCP connections](https://code.claude.com/docs/en/sub-agents) in two ways:

1. **String reference** — a server name in `mcpServers` frontmatter reuses the parent's existing connection
2. **Inline definition** — a full server definition object opens a fresh connection that closes when the subagent finishes

If `mcpServers` is omitted from a subagent definition, the subagent inherits all tools available to the parent, including MCP tools. The `disallowedTools` field accepts `mcp__server__*` patterns to restrict at the subagent level.

**Plugin subagents cannot use `mcpServers`** — the field is explicitly ignored for security. Copy agent files into `.claude/agents/` or `~/.claude/agents/` if you need MCP access in plugin-provided agents.

## Security

> **Local vs. remote server risk:** Local (stdio) MCP servers are OS processes with full user privileges — direct filesystem and network access, persistent across sessions. A compromised local server can exfiltrate data and execute arbitrary code as the user. Remote servers are limited to their API surface but can aggregate OAuth tokens across multiple connected services.

Key risks:

- **Tool poisoning** — malicious servers embed hidden instructions in tool descriptions (`tools/list`), injecting prompts directly into the model's context window. The [first malicious MCP package in the wild](https://arxiv.org/html/2603.22489v1) appeared September 2025 and exfiltrated email data undetected for two weeks.
- **TrustFall (Adversa AI, 2025)** — a malicious repository ships `.mcp.json` plus `.claude/settings.json` with `enableAllProjectMcpServers` to execute arbitrary OS processes when a developer accepts the folder trust dialog. In headless/CI mode, the trust dialog is bypassed entirely — zero interaction required.
- **OAuth token exposure** — OAuth credentials land in `~/.claude.json` (plaintext). CVE-2026-21852 allowed API key exfiltration by overriding an environment variable, redirecting authenticated MCP traffic to attacker-controlled infrastructure before any consent prompt.

Mitigations: prefer allow-listing specific `mcp__server__tool` rules over broad permission modes; treat `.mcp.json` in unfamiliar repositories as executable code; use managed-mcp.json in enterprise environments to lock the server list.

---
*See also: [Permission and Trust Model](permission-trust-model.md), [Multi-Agent and Worktree Isolation](multi-agent-worktree-isolation.md), [Claude Code vs Codex CLI](claude-code-vs-codex-cli.md)*
