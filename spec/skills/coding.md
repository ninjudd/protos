---
name: coding
description: Delegate complex coding tasks to a coding agent (Claude Code or Codex).
preferred_model: reasoning
---

# Coding

For complex, multi-file, or unfamiliar coding tasks, delegate to a coding agent rather than hand-editing with the canonical `edit` / `write` tools or shell commands.

## When to use this

This skill is most load-bearing when **the running backend lacks native coding-agent tools** or when you need **fresh-process isolation**.

- **Vercel / openai backends** — their bundled `read`/`write`/`edit` are general-purpose, not coding-tuned (no LSP, no git-aware diffs, no sandboxed bash). Shell out for real code work.
- **Fresh-process isolation** — different cwd, different env, different identity. Useful when the work should happen *outside* the daemon's process (editing a vendored repo, running a long build that shouldn't share the daemon's memory, anything you want to keep off the agent's audit trail).

**If you're on `claude` or `codex` backend**, prefer `bash` + `edit` + `delegate_task` directly. Those backends already wrap a coding-agent runtime — `bash` is sandboxed, `edit` is tuned for code, `delegate_task` gives context isolation within the daemon. Shelling out to a `claude`/`codex` CLI is redundant, slower (extra process launch), more expensive (a second billable agent run), and bypasses the model profile in `config/models.yaml`.

Cases that still warrant the shell-out even on `claude`/`codex` backends:

- The work is in a separate repo with its own toolchain (e.g. `vendor/browser-harness/`).
- You need to use a CLI option `delegate_task` doesn't expose (a different working directory, a specific env var, an interactive session).
- The task is genuinely large enough that you want a dedicated process with its own context window.

## Available agents

Use whichever is installed on the host. Check with `which claude` or `which codex`.

### Claude Code

```
claude --dangerously-skip-permissions -p "your prompt here"
```

- `--dangerously-skip-permissions` lets it run without interactive approval (use only for tasks you've already confirmed with the owner)
- `-p` passes a one-shot prompt
- For tasks in a specific directory, `cd` there first

### Codex

```
codex --approval-mode full-auto "your prompt here"
```

- `--approval-mode full-auto` lets it run without interactive approval
- For tasks in a specific directory, `cd` there first

## Steps

1. Confirm the task with the owner — what to build, where it should go, any constraints
2. Craft a clear, detailed prompt describing the task
3. Run the coding agent via your `bash` tool
4. Review the output and report back to the owner what was built or changed

## Rules

- Always confirm the task before running a coding agent
- Set a working directory appropriate to the task — don't run everything from the project root
- If the coding agent fails or produces bad output, tell the owner rather than retrying blindly
