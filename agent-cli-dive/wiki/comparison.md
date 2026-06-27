# Claude Code vs Codex CLI

**Codex CLI locks down the OS first and trusts Claude later; Claude Code trusts the developer first and lets them configure downward — making them architecturally inverted on the safety-vs-friction spectrum.**

## Security Architecture

The two tools enforce safety at different layers. Codex CLI uses OS-level sandboxing (macOS Seatbelt / Linux bubblewrap) with network disabled by default — the hard boundary is the kernel, not the app. Claude Code enforces security at the application layer via permission prompts and rule evaluation, with an optional sandbox that is off by default.

This inversion matters in practice: Codex CLI's constraints are harder to escape but also harder to override for legitimate use. Claude Code's constraints are more expressive — and more bypassable.

## Claude Code Permission Model

Claude Code has [six permission modes](https://code.claude.com/docs/en/permission-modes), ordered from most to least restrictive:

- **default** — prompts on all writes and executes
- **plan** — reads only, no edits allowed
- **acceptEdits** — auto-approves file edits and directory operations inside the working directory
- **auto** — classifier-backed approval with background safety checks; broad wildcard rules are dropped on entry
- **dontAsk** — auto-denies anything not in explicit allow rules
- **bypassPermissions** — skips all prompts; `--dangerously-skip-permissions` is an alias

`bypassPermissions` cannot be entered mid-session — it must be set at launch. It also refuses to start as root or under sudo outside a recognized container. Two circuit-breakers survive in v2.1.126+: explicit `ask` rules in `settings.json` still force a prompt, and `rm -rf /` and `rm -rf ~` still prompt.

### Rule syntax

Rules use the form `Tool` or `Tool(specifier)`. Evaluation order is: **deny → ask → allow**, first match wins regardless of specificity. A broad deny beats a narrow allow:

```
Bash(aws *)   # deny — blocks Bash(aws s3 ls) even if aws s3 ls has an allow rule
```

Rule arrays merge across all five settings scopes (managed > CLI args > local project > shared project > user). A deny at any scope beats an allow from any other. Scalar settings use highest-precedence-wins instead.

Key Bash glob details:
- Space before `*` enforces a word boundary: `Bash(ls *)` matches `ls -la` but not `lsof`
- `Bash(npm:*)` is equivalent to `Bash(npm *)`
- Compound commands (`&&`, `||`, `;`, `|`) are split — each subcommand must independently match a rule
- Process wrappers stripped before matching: `timeout`, `nice`, `nohup`, `stdbuf`, bare `xargs`
- Dev runners like `npx`, `docker exec`, `mise exec` are **not** stripped — rules must include the runner

A bare tool name as a deny rule (e.g., `Bash`) removes the tool entirely from Claude's context. A scoped deny like `Bash(rm *)` leaves the tool visible and blocks only matching calls.

### Protected paths and auto-mode restrictions

In default/acceptEdits/plan/auto modes, writes to these are never auto-approved: `.git`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.yarn`, `.devcontainer`, `.mvn`. `allow` rules in `settings.json` do not override the protected-path check — it runs first.

In `auto` mode, the classifier additionally blocks: force push, push to main, `git reset --hard`, `git checkout -- .`, `git restore .`, `git clean -fd`, `curl | bash`, and production deploys/IAM changes. These are classifier defaults, not hardcoded — enterprise admins can loosen them.

`auto` mode ignores `defaultMode: 'auto'` in checked-in project settings since v2.1.142. It must be set in `~/.claude/settings.json` at the user scope.

### WebFetch rules

Domain rules use a `domain:` prefix: `WebFetch(domain:example.com)`. A leading `*.` wildcard matches exactly one subdomain level. `*` in other positions only matches between dots, preventing `domain:example.*` from matching `example.evil.com`.

The docs explicitly warn: `Bash(curl http://github.com/ *)` is not reliable for constraining network tools. Deny `curl`/`wget` and use `WebFetch` with domain rules instead.

### Hooks

[Five hook event categories](https://code.claude.com/docs/en/hooks) exist: session-level, per-turn, tool execution, file/directory, and config. Handler types: `command`, `http`, `mcp_tool`, `prompt`, `agent`.

Evaluation ordering: hook block check → deny rule check → ask rule check → allow rule check. A hook returning `allow` does not override a matching deny rule — deny always wins. A blocking hook (exit 2) stops execution before rule evaluation runs.

## Known Gaps

GitHub issues document real bypass cases that persist into 2026:

- `git push` executing without approval when chained with `&&` in a single Bash call ([#13009](https://github.com/anthropics/claude-code/issues/13009), [#29076](https://github.com/anthropics/claude-code/issues/29076), [#16180](https://github.com/anthropics/claude-code/issues/16180))
- Force-push destroying repo history without a permission prompt ([#33402](https://github.com/anthropics/claude-code/issues/33402))
- Regular `git push` to main being blocked even with an explicit allow rule and explicit user approval ([#22636](https://github.com/anthropics/claude-code/issues/22636))

The compound-command bypass was reportedly fixed; git/push edge cases persist.

## Which to Reach For

| | Claude Code | Codex CLI |
|---|---|---|
| Safety boundary | Application layer (configurable) | OS kernel (hard) |
| Sandbox default | Off | On |
| Network | Allowed (configurable) | Disabled by default |
| Permission granularity | Fine-grained rule syntax | Coarser, mode-based |
| Enterprise controls | Managed settings, MDM | Less mature |
| Friction at default | Lower | Higher |

Codex CLI is the right default when you want hard guarantees without configuration work. Claude Code is the right default when you need fine-grained control over what the agent can and cannot do — and you're willing to configure it.

---
*See also: [Shell Execution Model](shell-execution-model.md), [File System Access](file-system-access.md), [MCP Integration](mcp-integration.md)*
