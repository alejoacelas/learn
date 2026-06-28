# Codex CLI Sandbox

**Codex CLI hard-sandboxes every agent session at the OS kernel level by default — network disabled, writes confined to CWD — while Claude Code defaults to application-layer permission prompts with kernel sandboxing as an opt-in; the two tools are architecturally inverted on the safety-vs-friction axis.**

## How it works

Codex CLI's sandbox is opt-out. The moment an agent session starts, the shell process is wrapped in a kernel-enforced policy that the agent code itself cannot escape:

- **macOS** — Apple Seatbelt via `/usr/bin/sandbox-exec`, driven by two SBPL profiles: `seatbelt_base_policy.sbpl` and `seatbelt_network_policy.sbpl`. Network access is binary: fully off, or restricted to specific loopback proxy ports.
- **Linux** — [Landlock LSM](https://codex.danielvaughan.com/2026/04/08/codex-sandbox-platform-implementation/) (kernel 5.13+) for filesystem control, plus seccomp-BPF to block `connect`, `accept`, `bind`, `listen`, `sendto`, and `sendmsg` syscalls. AF_UNIX sockets are exempted so local IPC still works.
- **Windows** — Restricted tokens, ACLs, and Windows Firewall rules enforced under dedicated sandbox users (`CodexSandboxOffline` and `CodexSandboxOnline`).

Because enforcement lives in the kernel, a compromised or jailbroken model response cannot sidestep it. The seccomp layer operates at syscall granularity — it cannot enforce semantic policies like "HTTPS only to a specific host," only "no `connect` calls at all."

## Policy modes

| Mode | Writes | Network |
|---|---|---|
| `read-only` | blocked | blocked |
| `workspace-write` *(default)* | CWD only | blocked |
| `external-sandbox` | assumed by container | assumed by container |
| `danger-full-access` | unrestricted | unrestricted |

One design constraint holds across all modes and all platforms: **read access is never restricted.** Sandboxed processes can read the full disk. The security surface Codex targets is unauthorized writes and network egress, not data exfiltration — a deliberate trade-off in favor of dev workflow compatibility.

## Known limitations and bugs

The kernel approach is strong but not perfect. Two notable incidents from the changelog:

- **v0.106.0**: a zsh fork-based execution path bypassed sandbox wrappers on certain shell invocations (fixed in [PR #12800](https://codex.danielvaughan.com/2026/04/08/codex-sandbox-platform-implementation/)).
- **v0.105.0**: the sandbox required `/dev/tty` and `/dev/urandom` to be provisioned inside the sandbox namespace; without them, affected commands failed silently.

Detection of sandbox denials uses heuristic stderr/exit-code pattern matching (`is_likely_sandbox_denied`), not structured error codes — false negatives are possible.

## Comparison: Claude Code's model

Claude Code inverts the approach. Its baseline is interactive permission prompts (application-layer), with OS-level sandboxing added optionally via macOS Seatbelt or Linux bubblewrap. That sandbox follows a read-broadly/write-narrowly policy: reads are broadly permitted, writes restricted to the workspace, and outbound network goes through a unix domain socket proxy rather than being blocked at the syscall level.

Anthropic reports an [84% reduction in permission prompts](https://www.anthropic.com/engineering/claude-code-sandboxing) with the sandbox enabled. For cases where sandboxing breaks workflows that need live network or host access, Claude Code's [auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode) adds a two-stage classifier — a fast single-token yes/no filter, then chain-of-thought reasoning on flagged actions — achieving a 0.4% false positive rate on real traffic.

The hook system (30 lifecycle events including `PreToolUse` and `PostToolUse`) enables programmable governance: hooks can block tool calls via exit code 2 or rewrite arguments before execution. But hooks share a process boundary with the agent, which is theoretically weaker than kernel-level isolation.

## When the difference matters

- **Long unattended sessions on sensitive machines**: Codex's kernel sandbox is strictly stronger — no policy escape possible within the session.
- **Workflows that require outbound network mid-task** (API calls, package installs, live fetches): both tools require dropping to a less-restrictive mode; Codex's mode switch is coarser.
- **Team governance and audit requirements**: Claude Code's hook system lets you enforce business-logic policies (e.g., block writes to `infra/`) that no kernel policy can express.
- **Cloud delegation**: Codex CLI's `codex cloud exec` delegates tasks to cloud agents and returns diffs asynchronously. Claude Code has no direct equivalent — its subagents run locally and require the parent process to stay alive.

---
*See also: [Claude Code Permission and Trust Model](claude-code-permissions.md), [Shell Execution Model](shell-execution-model.md), [File System Access](file-system-access.md)*
