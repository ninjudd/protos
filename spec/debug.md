# Debug

For the `debug [issue description]` command — diagnose a specific issue the owner is having with their running agent.

## Inputs

Free-form text describing the symptom. Examples:

```
debug Telegram keeps crashing
debug nap cron isn't writing to memory
debug the assistant forgot my coffee preference
```

No flag syntax, no required structure. The description is for humans, not argument parsing. If the description isn't enough to act on, ask clarifying questions in chat — don't refuse to start.

## Hard rule: never auto-apply

`debug` never writes code, modifies the running agent, or restarts anything. Output is a diagnosis plus an explicit "here's what I'd change in `agent/`" or "here's what I'd change in `spec/`". The owner decides whether to apply (via `update` for spec-driven changes, by hand for one-off fixes, or by running another command).

## Workflow

1. **Read the available evidence first** before asking the owner anything:
   - `runtime/threads/{channelId}/{conversationId}.jsonl` — conversation context, tool calls, tool results
   - `runtime/logs/cron/{jobname}/` — per-cron-run event streams
   - `runtime/logs/sub-agents/` — sub-agent traces (paths nested under their parent's log)
   - `runtime/reviews/` — past `review` reports, if any
   - `runtime/tests/` — past `test` runs, if any
   - `agent/src/` — implementation
   - `spec/` — contracts

   On personal-assistant scale, scanning all of `runtime/` is cheap; don't make the owner specify which file to look at. Use the issue description's keywords (channel name, cron job name, behavior) to narrow once you have the contents in mind.

2. **Ask clarifying questions only if needed.** Good ones: when did this start, is it intermittent, what was the owner doing right before, do they have a recent example to point at. Don't ask if the answer is already in the logs.

3. **If the evidence isn't enough, offer to escalate.** Two opt-in paths — *ask first*, never run silently:
   - **Write a test** that captures the expected behavior, using `spec/test.md` for conventions. The test failure shows where the implementation diverges. The test stays as a regression guard after the fix. Subject to the same hard rule that governs `test` itself: only with explicit owner approval, every time.
   - **Run `review`** to look for spec drift. Useful when the symptom smells like an implementation that didn't quite match the spec (a contract being subtly violated rather than a clear bug).

4. **Report back** with:
   - **Diagnosis** — one or two sentences naming the cause.
   - **Evidence** — which log lines, file:line, or test result led to the conclusion.
   - **Suggested change** — explicit "in `agent/src/foo.ts:42`, change X to Y" or "in `spec/architecture.md` → Section, the contract is ambiguous about Z; clarifying it would let `update` regenerate correctly". Never "I went ahead and fixed it."

## What NOT to do

- **Don't auto-apply fixes.** Report; the owner runs `update` or hand-edits.
- **Don't restart the agent.** That's `agent/protos restart`, owner's call.
- **Don't write tests or run review without asking.** Both have their own opt-in rules; respect them.
- **Don't touch `config/`, `memory/`, `vendor/`.** Those are the owner's living state. Read them if needed for context; never modify.
- **Don't spawn a coding agent to investigate.** `debug` *is* the coding-agent invocation — escalate to the owner, not to another agent.

## Run logs

Tee the session output (clarifying questions, evidence reviewed, conclusions) to `runtime/debug/{ISO-timestamp}-{slug}.log`, where `{slug}` is a short slugified prefix of the issue description (e.g. `runtime/debug/2026-04-23T18:02:49Z-telegram-keeps-crashing.log`). Show the path in the chat reply so the owner can re-read or share it later.
