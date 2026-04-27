---
name: debug
description: Investigate when the user reports something seems off — read your own logs, diagnose, surface findings with evidence. Never auto-fix.
---

# Debug

When the user says something seems off ("the cron didn't fire," "you forgot a memory," "Telegram has been weird"), this skill walks you through diagnosing your own behavior using the runtime artifacts you have direct access to. Diagnosis only — never auto-fix.

This is the **runtime** debug skill. The matching coding-agent workflow lives in `spec/debug.md`; the user invokes that one from a terminal (Claude Code, Codex) when the diagnosis needs deeper code review or runtime restart.

## When to use this

- The user reports specific misbehavior — a job didn't run, a memory got dropped, a channel went quiet, a tool returned something weird, you replied something the user thinks is wrong.
- You yourself notice something off in a recent turn (a tool call that errored, a continuation that broke, a sub-agent that returned `ok: false`).
- The user asks "what happened with X?" and X is something you should be able to reconstruct from your own logs.

Don't reach for this skill for general "are you working OK?" questions — those don't need an investigation, they need a quick health check or a restart.

## Evidence sources (read first, ask later)

Try to answer the question from your own artifacts before asking the user clarifying questions. Most things are reconstructable.

- **Recent conversation history** — `runtime/threads/{channelId}/{conversationId}/{profile}.jsonl`. Each line is a typed event (user message, assistant text, tool call, tool result). Use `read` for a specific file or `glob runtime/threads/**/*.jsonl` to enumerate. Cross-profile activity in the same conversation lives in sibling JSONL files.
- **Cron run logs** — `runtime/logs/cron/{jobname}/{ISO-timestamp}.jsonl`. One file per run. Includes the cron body that was injected, what tools the agent called, and the final reply (or `NO_REPLY`). `glob runtime/logs/cron/{jobname}/*.jsonl` for the run history.
- **Sub-agent logs** — `runtime/logs/sub-agents/{channelId}/{conversationId}/{turn_id}-{call_id}.jsonl`, or nested under cron logs for cron-triggered sub-agents. The parent's `tool_result` event has a `sub_agent_log` field pointing at the file.
- **Daemon log** — `runtime/logs/agent.log`. Process-level stdout/stderr including dispatch errors, scheduler warnings, channel-startup failures. `tail` it via `bash` for recency: `bash 'tail -200 runtime/logs/agent.log'`.
- **Daemon liveness** — `runtime/agent.pid` exists and the PID is alive (`bash 'cat runtime/agent.pid'`, then `bash 'ps -p $PID'`). If the daemon is dead but the user is talking to you, you might be looking at a *different* running process — check.
- **Config** — `config/models.yaml`, `config/channels.yaml`, `config/.env` (don't quote secrets back in chat, but you can `read` it to confirm a key is set). `config/SOUL.md` for identity drift.
- **Spec contracts** — `spec/architecture.md`, `spec/build.md`, the relevant `spec/tools/*.md` and `spec/channels/*.md` recipes. Useful when the question is "is this the expected behavior" — the spec is the canonical answer.
- **Implementation source** — `agent/src/`. Read-only; diagnosis only. If a fix needs to land in code, hand off to a coding agent (see below).
- **Skill manifest** — call `list_skills` if you suspect skills changed during the conversation and your system prompt's manifest may be stale.

## Diagnosis pattern

1. **State the symptom precisely.** Quote the user back in your own words. If their description is ambiguous, ask one clarifying question — but only after you've read what's already obvious from the artifacts.
2. **Locate the relevant evidence.** Pick the artifacts above that match the symptom. A "cron didn't fire" question goes to `runtime/logs/cron/{jobname}/`. A "you forgot a memory" question goes to the conversation JSONL plus the relevant memory file. A "Telegram is weird" question goes to `runtime/logs/agent.log` plus recent thread events for that channel.
3. **Reconstruct what happened.** Read enough of the evidence to reconstruct the timeline. Cite specific events with their timestamps.
4. **Form a hypothesis.** State what you think went wrong, including the contracted-vs-actual delta. ("Spec says cron jobs without `model:` should use the `default` profile; the log shows it ran with `subagent` — possibly a config override I'm not aware of, or a regression.")
5. **Surface findings to the user.** Be specific. Point at the file paths and timestamps. Don't summarize without evidence.
6. **Recommend the next step** — see "Handoffs" below.

## Handoffs

When diagnosis exceeds what you can do in chat, point the user at the right tool. Don't try to do it yourself.

- **Suspected spec/agent drift** (the implementation doesn't match the spec contract) → suggest the user run `review` from a coding agent. Quote the suspected drift point.
- **Suspected regression that needs test coverage** → suggest the user run `test` from a coding agent, ideally with a focused test name reflecting the bug.
- **Need a code fix** → suggest the user run `debug "<description>"` from a coding agent. The coding-agent workflow has more capability (full repo access, can spawn sub-agents, can restart the daemon). Phrase the description concretely so the coding agent has a head start.
- **Need a config change** (e.g. wrong model profile, missing env var) → tell the user what to change, where, and what restart command to run after.

## Hard rules

- **Never write to `agent/src/`** as part of debugging. That's a coding-agent operation. If the diagnosis says code needs to change, hand off.
- **Never restart the daemon** as part of debugging. If a restart is needed, tell the user explicitly and let them run it.
- **Never quote secrets back to the user** even if `config/.env` is part of the evidence. Confirm "key is set" / "key is unset" — never the value.
- **Never disable a skill, cron job, or channel** as part of debugging. Surface the recommendation; let the user apply it.
- **Be honest about uncertainty.** "I don't see this in the logs" is a finding. "I think this is what happened, but I can't tell from the artifacts" is a finding. Don't guess past the evidence.
