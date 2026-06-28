# Permission and Trust Model

**Claude Code enforces permissions through a five-tier scope hierarchy, six named modes, and a deny-beats-allow rule system — but compound command parsing bugs and a file-edit blind spot in auto mode mean the system is not a hard security boundary.**

## Settings Scope Hierarchy

Rules merge across five scopes, highest to lowest precedence: managed (MDM/OS policy) > CLI args > local project (`.claude/settings.local.json`) > shared project (`.claude/settings.json`) > user (`~/.claude/settings.json`). For permission rule arrays, all scopes merge — a deny at any scope beats an allow at any other. Scalar settings use highest-precedence-wins. A project cannot grant itself `auto` mode through checked-in settings; that must be set in user scope or above.

## Six Permission Modes

From most to least restrictive:

- **default** — prompts on all writes and executes
- **plan** — reads only, no edits or shell execution
- **acceptEdits** — auto-approves file edits and safe filesystem ops (mkdir, mv, cp) inside the working directory
- **auto** — classifier-backed approval; a fixed server-side `claude-sonnet-4-6` instance evaluates each action independently of your `/model` selection
- **dontAsk** — auto-denies anything not in an explicit allow rule
- **bypassPermissions** (`--dangerously-skip-permissions`) — skips all prompts; two circuit-breakers remain in v2.1.126+: explicit `ask` rules in settings still prompt, and `rm -rf /` and `rm -rf ~` still prompt

`bypassPermissions` refuses to start as root or under sudo (outside a recognized sandbox container), and cannot be entered mid-session without an enabling flag at launch.

## Rule Syntax and Evaluation Order

Rules use `Tool` or `Tool(specifier)` syntax. Evaluation order is: hook block check → deny → ask → allow. A broad `deny Bash(aws *)` beats a narrower `allow Bash(aws s3 ls)` — specificity does not matter, deny always wins.

A bare tool name in deny (e.g., `Bash`) removes the tool from Claude's context entirely. A scoped deny (e.g., `Bash(rm *)`) leaves the tool available and blocks only matching calls.

Compound commands (`&&`, `||`, `;`, `|`) are split and each subcommand must independently match a rule. In practice this parsing has had [persistent bugs](https://github.com/anthropics/claude-code/issues/16180): chains where the first subcommand is allowed have historically executed the entire chain. A [50-subcommand cap](https://adversa.ai/blog/claude-code-security-bypass-deny-rules-disabled/) that silently bypassed all deny rules was patched in v2.1.90.

## Auto Mode Classifier

The auto mode classifier runs a two-stage pipeline: a fast single-token filter (8.5% FPR) followed by chain-of-thought reasoning (full pipeline 0.4% FPR). It is reasoning-blind by design — it sees only user messages and tool call payloads, not assistant prose or tool outputs, preventing persuasive rationalizations from influencing its decisions.

Critically, in-project file edits bypass the classifier entirely. Independent research ([AmPermBench](https://arxiv.org/html/2604.04978)) found 36.8% of state-changing actions bypassed classification via `Edit`/`Write` tool calls. On artifact cleanup tasks, the false negative rate was 92.9%.

## Hooks

[Hooks](https://code.claude.com/docs/en/hooks) intercept tool execution at five event categories. A `PreToolUse` hook exiting with code 2 blocks the call before rules run. However, hook `allow` decisions do not override settings deny rules — the ordering is hook block → deny rule → ask rule → allow rule.

## Protected Paths

A fixed set of directories (`.git`, `.vscode`, `.husky`, `.cargo`, `.claude`) and dotfiles (`.zshrc`, `.gitconfig`, `.npmrc`) are never auto-approved in default/acceptEdits/plan/auto modes. `allow` rules in settings do not override this check — it runs before rule evaluation. Only `bypassPermissions` clears it.

---
*See also: [Shell Execution Model](shell-execution-model.md), [MCP Integration](mcp-integration.md), [Claude Code vs Codex CLI](claude-code-vs-codex-cli.md)*
