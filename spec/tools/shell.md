# shell

Run a bash command on the host.

## Description (shown to the model)

Run a shell command using bash. Workspace root is the working directory. Output is capped at 1 MB. Don't use this for long-running commands without telling the user first — the conversation pauses while the command runs.

## Input

```ts
{ cmd: string }
```

## Output

```ts
{
  stdout: string,
  stderr: string,
  exit_code: number,
}
```

Output is truncated to 1 MB total (combined stdout + stderr). On truncation, append `\n…[truncated at 1 MB]` to the relevant stream.

## Behavior

- Run via `child_process.exec` (or equivalent) with bash, working directory set to the workspace root, default environment inheriting from the daemon.
- **Async** — must not block the event loop. Use the promise wrapper or callback.
- 1 MB output cap on combined stdout+stderr.
- Exit code returned, never thrown — the model can interpret non-zero as needed.
- **Self-edit nudge:** when `LOGOS_SELF_EDIT=false`, the tool description (what the model sees) gets a note appended: `NOTE: self-edit is disabled. Do not modify files under agent/.` This is a convention, not enforcement — shell can technically still write anywhere the process user has permission to. For hard enforcement see BUILD.md → Sandboxing.

## Dependencies

Node standard library (`node:child_process`).

## Implementation notes

- The agent should let the user know before running long commands (the description above includes this guidance).
- For destructive operations (`rm -rf`, etc.), the agent should confirm with the user first. This is documented in `AGENTS.md`.
- Command timeout: configurable but default reasonable (e.g., 60s). Document the default.
