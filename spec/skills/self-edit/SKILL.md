---
name: self-edit
description: Modify the agent's own source code and safely restart to apply changes.
---

# Self-Edit

You can modify your own source code in `agent/src/`. Follow this process carefully.

## Steps

1. Make your code changes using the `edit_file` or `write_file` tools (or shell if needed for moves/deletes)
2. Send a message to the conversation explaining what you changed and why
3. Run `agent/logos restart` as your **last action**

The restart will type-check your code before applying it. If the check fails, the old process keeps running and the error is logged. You will not crash yourself.

## Reference

Before making changes, read the relevant documentation:

- `spec/ARCHITECTURE.md` — system design, component responsibilities, how pieces fit together
- `spec/channels/*.md` — implementation recipes for each messaging channel
- `spec/BUILD.md` — how the engine was built (useful when adding new components)

## Rules

- Always explain what you changed before restarting
- Make small, focused changes — one thing at a time
- For complex or multi-file changes, use the **coding** skill instead
- If you're unsure about a change, describe it first and ask for confirmation
- Never modify `config/.env` or files outside the workspace
- Edit files in `agent/src/` (your implementation). Don't edit `spec/` — that's the design, shared with all Logos users; if the design needs to change, raise it with the human owner.
- Don't put instance-specific changes in `agent/` — those belong in `config/`
