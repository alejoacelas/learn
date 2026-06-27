# File System Access

**Both tools restrict writes to the working directory by default but leave reads wide open — including credential files — unless you explicitly configure otherwise, and the sandbox only covers shell commands, not the built-in file tools.**

## Default Behavior

Neither tool is read-restricted by default. Claude Code's [permission system](https://code.claude.com/docs/en/permissions) gates writes but not reads: the agent can read `/etc`, `~/.ssh/`, `~/.aws/credentials`, or any other path on disk without prompting. Codex CLI's workspace-write sandbox mounts the entire filesystem read-only at `/` and then overlays write permissions on cwd, `/tmp`, and `$TMPDIR` — so reads are similarly unrestricted unless you configure otherwise.

## What "Sandbox" Actually Covers

**Claude Code sandbox** (opt-in, disabled by default):

- Enabled via `/sandbox`; uses [macOS Seatbelt](https://code.claude.com/docs/en/sandboxing) (`sandbox-exec`) or Linux bubblewrap
- Covers only Bash subprocesses — the built-in Read, Edit, and Write tools [bypass the sandbox entirely](https://code.claude.com/docs/en/sandboxing) and go through the permission system
- Write defaults: cwd + session tmpdir only
- Read defaults: entire filesystem — the docs explicitly warn: "this still allows reading `~/.aws/credentials` and `~/.ssh/`; add them to `denyRead` to block them"
- The escape hatch: when a sandboxed command fails, the model can retry with `dangerouslyDisableSandbox`; disable this with `allowUnsandboxedCommands: false`

**Codex CLI sandbox** (on by default on Linux):

- Linux: bubblewrap with `--ro-bind / /` baseline; writable overlays for cwd, `/tmp`, `$TMPDIR`; `.git`, `.agents`, `.codex` are re-applied read-only inside writable roots
- macOS: Seatbelt with dynamically generated SBPL profiles; same write defaults
- Codex sandboxes its own shell commands, not itself — the main Codex process runs unsandboxed
- [CVE GHSA-w5fx-fh39-j5rw](https://github.com/openai/codex/security/advisories/GHSA-w5fx-fh39-j5rw): model-controlled `cwd` was used as the sandbox write boundary pre-v0.39.0, enabling arbitrary writes

## .env and Credential Files

Neither tool protects `.env` files automatically. Claude Code reads them through the standard Read tool like any file; `denyRead` rules on the Read tool alone are insufficient because `cat .env` via Bash bypasses them. Only a sandbox-level `denyRead` rule blocks both paths.

The permission system's deny rules have a documented history of regressions — [issue #6699](https://github.com/anthropics/claude-code/issues/6699) and [#12918](https://github.com/anthropics/claude-code/issues/12918) both describe deny rules being silently non-functional for entire versions. The most reliable protection is a `PreToolUse` hook that exits with code 2, which operates before permission evaluation.

## Symlinks

Claude Code resolves symlinks before checking permissions. Deny rules fire if either the symlink path or its target matches; allow rules require both to match. Two separate CVEs ([GHSA-66m2-gx93-v996](https://github.com/anthropics/claude-code/security/advisories/GHSA-66m2-gx93-v996), [GHSA-4q92-rfm6-2cqx](https://github.com/anthropics/claude-code/security/advisories/GHSA-4q92-rfm6-2cqx)) document the same class of symlink-bypass vulnerability recurring across versions.

Codex CLI handles worktree `.git` pointer files correctly — it resolves the `gitdir:` path and protects both the pointer and the target object store as read-only.

---
*See also: [Claude Code Permission and Trust Model](permissions.md), [Shell Execution Model](shell-execution.md), [Claude Code vs Codex CLI](tool-comparison.md)*
