# Shell Execution Model

**Both Claude Code and Codex CLI execute shell commands as non-interactive subprocesses with no real TTY — a fundamental constraint that cascades into environment inheritance, stdin behavior, signal handling, and sandbox architecture.**

## No TTY, No Stdin

Claude Code's Bash tool runs commands via `child_process.spawn()` with `stdio: pipe`. The [official docs state explicitly](https://github.com/anthropics/claude-code/issues/26353): "the Bash tool doesn't have a TTY or interactive input." Commands that break as a result:

- Text editors: `vim`, `nano`
- Interactive git: `git rebase -i`
- TUI tools: `htop`, `lazygit`, `tig`
- Install wizards: `npm init`, interactive package installers
- REPLs: `python`, `node`, `irb`
- Privilege prompts: `sudo` (no TTY means no password prompt; sudo's per-TTY credential cache is also invisible to the subprocess)
- Any script calling `process.stdin.setRawMode(true)`

There is [no mid-execution stdin passthrough](https://github.com/anthropics/claude-code/issues/30555). Once a Bash tool call starts, user keystrokes cannot reach it. The `!` prefix reaches the host shell with a real TTY but returns no tool result to the model, breaking agentic continuity.

Gemini CLI shipped PTY support in v0.9.0 via `node-pty`. Claude Code has no PTY infrastructure; issue [#9881](https://github.com/anthropics/claude-code/issues/9881) tracking this has been open since October 2025 with no Anthropic response, and a related opt-in proposal was [explicitly closed as not planned](https://github.com/anthropics/claude-code/issues/37523).

**Codex CLI differs structurally**: it uses `openpty()` and the `portable-pty` crate, giving it PTY infrastructure from the start.

## Environment Inheritance

Claude Code inherits the full parent shell environment when launched from a terminal — including secrets like `AWS_SECRET_ACCESS_KEY` and `GITHUB_TOKEN`. No scrubbing happens by default.

`CLAUDE_CODE_SUBPROCESS_ENV_SCRUB=1` (shipped v2.1.83) removes a fixed whitelist of credentials from Bash tool, hook, and MCP stdio subprocess environments. It is [best-effort by design](https://github.com/anthropics/claude-code-action/blob/main/docs/security.md) — notably, `GITHUB_TOKEN`, `AWS_ACCESS_KEY_ID`, and `OPENAI_API_KEY` are **not** on the scrub list. Before v2.1.128, the Read tool could bypass the scrub entirely by reading `/proc/self/environ`.

**Codex CLI is more aggressive**: `shell_environment_policy` defaults to `all` but supports `core` (PATH, HOME, USER, SHELL, TERM, LANG, LC_*) and `none`. Regardless of mode, any variable whose name contains `KEY`, `SECRET`, or `TOKEN` is [stripped by default](https://codex.danielvaughan.com/2026/04/28/codex-cli-shell-environment-policy-subprocess-secrets-defence/). Codex also strips `DYLD_*` / `LD_*` at startup to block library-injection attacks.

One gotcha: Claude Code Desktop launched from the Dock [does not inherit the user's shell PATH](https://github.com/anthropics/claude-code/issues/44649). Homebrew tools, `pyenv`/`nvm` shims, and similar binaries are absent. Launch from a terminal to get the correct environment.

## Signal Handling and Background Processes

Claude Code's default Bash tool timeout is 120 seconds. On timeout, it sends `SIGTERM` to the process group. Because Claude Code and its child shell [share the same process group](https://github.com/anthropics/claude-code/issues/45717), the signal kills Claude Code itself (exit code 143). This issue was closed as "not planned" with no fix shipped through v2.1.187.

Background processes launched with `nohup ... &` are tracked by Claude Code and receive `SIGTERM` when the session ends or context compacts — including intentional long-running daemons. To truly detach a process:

```
nohup <cmd> </dev/null >/dev/null 2>&1 &
```

Without redirecting stdin from `/dev/null`, the background process holds Claude Code's stream-json pipe open, [causing an indefinite hang](https://github.com/anthropics/claude-code/issues/43123) (became fatal in v2.1.87).

**Codex CLI handles this correctly**: every shell tool child calls `setsid()` (or `setpgid(0,0)` on `EPERM`) before exec, putting children in their own session. On Linux, `prctl(PR_SET_PDEATHSIG, SIGTERM)` ensures children die when Codex dies, even on `SIGKILL`. On timeout, Codex resolves the child's PGID and sends `SIGKILL` to that group — not its own group.

Codex has no background process registry; processes started with `cmd &` are not tracked and not terminated on session end. A [documented model-policy bug](https://github.com/openai/codex/issues/13141) causes the Codex agent to emit `nohup`/`setsid`/`&` launch patterns by default, which defeats its own cleanup mechanisms.

## Sandbox Architecture

Neither tool enables sandboxing by default. When enabled, both use the same underlying OS primitives:

| | Claude Code | Codex CLI |
|---|---|---|
| macOS | `sandbox-exec` (Seatbelt), dynamic SBPL profile | `sandbox-exec`, static SBPL files assembled at runtime |
| Linux | `bubblewrap` + optional seccomp via `@anthropic-ai/sandbox-runtime` | `bubblewrap` + Landlock + seccomp via `codex-linux-sandbox` binary |
| Windows | — | Restricted process tokens via `codex-windows-sandbox` |
| Default stance | Closed-by-default (`deny default`) | Closed-by-default (`deny default`) |
| Network | Outbound via localhost proxy only | Outbound via localhost proxy only |

Both sandbox profiles allow `pseudo-tty` conditionally. Both restrict `mach-lookup` to an explicit allowlist, which produces real-world failures: `os.cpus()` returns an empty array in both tools (breaking `yarn`, `npm`, `node-gyp`) because `mach-host*` is missing from the allowlist. iOS Simulator tests [fail in Claude Code](https://github.com/anthropics/claude-code/issues/36611) for the same reason.

Claude Code's sandbox is opt-in via `/sandbox`. Codex's sandbox is on by default with `--sandbox danger-full-access` to disable.

---
*See also: [File System Access](file-system-access.md), [Claude Code vs Codex CLI](claude-code-vs-codex.md), [Claude Code Permission and Trust Model](permission-trust-model.md)*
