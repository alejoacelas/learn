# Multi-Agent and Worktrees

**Claude Code offers four parallelization primitives — subagents, agent view, agent teams, and dynamic workflows — with worktree isolation providing git-level filesystem separation so parallel agents don't collide on disk.**

## Parallelization Primitives

**[Subagents](https://code.claude.com/docs/en/sub-agents)** are the base unit. Each starts with a fresh context window — no parent history, no parent system prompt. The only input is the prompt string passed by the parent. Subagents inherit MCP tools and the session's tool allowlist, but cannot use `AskUserQuestion`, `EnterPlanMode`, or `ScheduleWakeup`. As of v2.1.172, nesting is allowed up to five levels deep.

**Agent teams** (experimental, disabled by default) run multiple full sessions that share a task list and can message each other. Intermediate results land in each teammate's context window. Teams provide no worktree isolation — teammates must partition work by file ownership to avoid conflicts.

**[Dynamic workflows](https://code.claude.com/docs/en/workflows)** are JavaScript scripts Claude writes and a background runtime executes. This is the most important primitive for throughput: orchestration logic lives in the script, not in Claude's context window, so intermediate results don't accumulate there. Four core script primitives:

- `agent(prompt, opts?)` — spawns one subagent, returns its final text or validated JSON
- `parallel(thunks)` — runs tasks concurrently as a barrier; failed thunks resolve to `null` rather than rejecting the whole call
- `pipeline(items, ...stages)` — streams items through stages with no barrier; item N+1 can enter stage 2 while item N is still in stage 1
- `workflow(nameOrRef, args?)` — invokes a saved workflow as a sub-step (one nesting level maximum)

Default to `pipeline()` for throughput; use `parallel()` only when a downstream step needs every result before it can start.

**Caps:** 16 concurrent agents per run (roughly your core count; excess queues), and a hard lifetime cap of 1,000 agents per run. Both are per-run, not global.

## Worktree Isolation

Setting `isolation: 'worktree'` on an `agent()` call causes Claude Code to run `git worktree add` under `.claude/worktrees/<name>/`, branching from `origin/HEAD` by default. Override with `worktree.baseRef: 'head'` to branch from local HEAD instead — useful when subagents need to operate on in-progress work.

Gitignored files (like `.env`) are not copied into worktrees automatically. Add a [`.worktreeinclude`](https://code.claude.com/docs/en/worktrees) file at the project root using `.gitignore` syntax to copy specific gitignored files. For large monorepos, `worktree.sparsePaths` uses git sparse-checkout to write only listed directories, reducing initialization time.

**Known bug ([#55708](https://github.com/anthropics/claude-code/issues/55708)):** isolation is filesystem-level only. Git commands like `git switch -c` inside a worktree can modify the parent repository's HEAD rather than the worktree's HEAD, silently corrupting the parent's branch state. This is a live race condition risk when running parallel subagents.

Worktrees are removed automatically when the agent finishes, provided there are no uncommitted changes, untracked files, or new commits.

---
*See also: [Permission and Trust Model](permissions.md), [MCP Integration](mcp-integration.md)*
