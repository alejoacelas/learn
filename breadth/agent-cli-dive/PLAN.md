# Agent CLI Dive — Workflow Plan

**Goal:** Build a comprehensive wiki on what Claude Code and Codex CLI can actually do versus the naive mental model ("LLM with access to everything on your computer"). Map every capability, restriction, and security boundary — with special attention to Mac app development pain points.

---

## Mental model we're correcting

> "It's just an LLM that can run shell commands on my computer."

The real picture is layered: the model sits inside a harness with its own permission model, tool abstraction layer, network policy, and OS-level constraints — plus whatever the *underlying OS and app* impose on top of that. The workflow investigates each layer.

---

## Dimensions to investigate

### 1. Shell execution model
- What commands run in an actual shell vs. a sandboxed subprocess?
- Interactive/TTY-dependent commands: `sudo`, `vim`, `git rebase -i`, `ssh` with prompts, `brew install` (interactive Y/N)
- Long-running / background processes: can the agent truly detach?
- stdin piping: what can and can't receive interactive input?
- Signal handling: Ctrl-C, SIGTERM, process trees

### 2. Permission and trust model (Claude Code)
- The three modes: default, trusted (--dangerously-skip-permissions), and per-project `settings.json` allowlists
- What always requires confirmation regardless of mode?
- What is always blocked (e.g. `git push --force` to main)?
- Hooks: how `settings.json` hooks intercept tool calls
- How `CLAUDE.md` files affect permissions across scopes (global, project, local)

### 3. Network access and restrictions
- `WebFetch` / `WebSearch` — what domains/schemes are allowed or blocked?
- `curl` via Bash — is it unrestricted, or does Claude's sandbox intercept it?
- Codex CLI default: **network-disabled sandbox** on macOS/Linux — what exactly is blocked?
- Differences between `--full-auto` mode and default Codex behavior
- MCP servers that provide web access: how they differ from native tools

### 4. File system access
- What paths are readable/writable by default?
- `.env` and secrets: does the harness warn or block?
- Symlinks, `/etc`, system directories
- Codex's **disk-write sandbox**: what can it write, where?
- Worktree isolation in Claude Code (for multi-agent): how isolated is a worktree?

### 5. Mac app development — the hard cases
- **Code signing**: can the agent run `codesign`, manage certificates, use Keychain?
- **Entitlements**: can it modify `.entitlements` files and re-sign?
- **Notarization**: `notarytool` / `altool` — API key or Apple ID password from Keychain?
- **Gatekeeper / quarantine**: `xattr -d com.apple.quarantine` — does this work in the agent context?
- **Xcode build system**: `xcodebuild` — derived data paths, simulators, provisioning profiles
- **App sandbox**: if the built app is sandboxed, does that affect what the agent can do *to* it (install, run, inspect)?
- **Privileged helpers / SMJobBless**: any path here for an agent?
- **TestFlight / App Store submission**: API-driven vs. UI-only paths

### 6. Codex CLI sandbox model (macOS/Linux)
- Network-disabled sandbox: exact scope (does DNS still work? localhost?)
- Disk-write sandbox: which directories are writable?
- Docker isolation option: how does `--sandbox=docker` differ?
- `--full-auto` mode: what changes in the permission flow?
- Comparison: Codex sandboxing vs. Claude Code's permission prompt approach

### 7. MCP (Model Context Protocol) server integration
- How MCP servers extend the tool surface
- Security model of MCP: what trust level do MCP tool calls carry?
- Authenticated vs. unauthenticated MCP servers
- Deferred tools: when schemas aren't loaded until requested

### 8. Multi-agent / worktree isolation (Claude Code)
- How `isolation: "worktree"` works in Workflow scripts
- What the agent inside a worktree can and can't see
- Token budget and concurrency caps
- Agent types and capability differences

### 9. Context and memory limits
- Token window, compression, and what gets lost
- Persistent memory system: file-based, not the model's weights
- What happens at context overflow during long tasks

### 10. Comparison table: Claude Code vs. Codex CLI
- Default security posture
- Network policy
- File system policy
- Interactive command handling
- Extensibility (MCP vs. plugins)
- Multi-agent support
- Mac-specific behavior

---

## Workflow structure (proposed)

```
Phase 1 — Map (parallel fan-out, 10 agents)
  Each agent covers one dimension. Returns: findings, low-confidence
  items to verify, and flags for deeper investigation (with search terms).

Phase 2 — Deep Dives (parallel, driven by Phase 1 flags)
  Top ~15 flagged topics from Phase 1 get dedicated deep-dive agents.
  Each flag includes why it matters and specific search terms.
  Results feed back into the wiki pages for that dimension.

Phase 3 — Verify (parallel)
  Low-confidence findings from Phase 1 get adversarial verification.
  Each verifier tries to refute the claim; returns confirmed/refuted/nuanced.

Phase 4 — Write Wiki (parallel per page)
  One agent per wiki page; each page gets its dimension's findings +
  relevant deep dives + cross-references. Pages are ~400 words.
```

---

## Wiki style guide

- ~400 words per page — be ruthless about what to include
- First line: single declarative sentence (the lede / complete summary)
- Bullet points for discrete facts
- Paul Graham style: clear declarative sentences, no hedging, no filler
- Each sentence carries weight — cut anything that doesn't add information
- Cite sources inline with markdown links
- End each page with "See also" cross-links to related pages
- Sources page: ~200 words, top 10–15 sources grouped by type, one sentence on each

## Output format

- `wiki/` folder with one `.md` per dimension
- `wiki/mac-app-dev.md` — Mac app friction points (high priority)
- `wiki/comparison.md` — Claude Code vs. Codex side-by-side
- `wiki/sources.md` — half-page, top sources with highest information density
- `wiki/index.md` — overview with cross-links

---

## Open questions / things to validate

- Does Claude Code's Bash tool actually inherit the user's full shell environment (PATH, secrets in env)?
- When Codex says "network-disabled", does that mean `curl localhost` also fails?
- Is `sudo` categorically blocked in Claude Code, or only when no TTY is available?
- For Mac signing: does the agent's process inherit the user's Keychain unlock state?
- What's the practical difference between `--dangerously-skip-permissions` and a fully-permissive `settings.json`?

---

## Review notes (to fill in after discussion)

_Space for feedback on scope, emphasis, missing dimensions..._
