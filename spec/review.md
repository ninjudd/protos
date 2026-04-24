# Review

For the `review` command — audit `agent/` against `spec/` and produce a markdown report of any drift.

## Hard rule: read-only

`review` never writes, modifies, restarts, or runs anything. It reads `spec/` and `agent/` and produces a report. Fixes happen via `update` (for spec-driven sync) or by hand-editing `agent/` (for cases the spec doesn't cover). Auto-patching from `review` would overlap `update`'s job and create two paths into the same territory — keep `review` honest as the audit step that informs a deliberate next action.

## What to check

The spec invariants and contracts the implementation must satisfy. Same list as `spec/test.md` → "What to test" — both commands check the same things; `test` exercises them at runtime, `review` checks them by reading the code.

- **Tool return shapes** — succeed-or-fail vs. succeed-or-miss conventions; never bare `null` from a lookup tool (architecture.md → Tool return shapes).
- **Tool call / result pairing** — every assistant turn that produces a `tool_call` appends a corresponding `tool_result` event, even when the tool throws (architecture.md → Tools).
- **Skill loader** — spec scans `*.md` only; config scans both `*.md` and `*/SKILL.md`; flat wins within config; merge across roots (frontmatter per-field, body appended).
- **Memory link resolution** — shortest-path tiebreak; aliases; `add_memory` / `rename_memory` preserve resolutions on changes (architecture.md → Memory format).
- **Render filter** — last non-empty assistant text per `turn_id`; intermediate tool events skipped; `NO_REPLY` skipped (architecture.md → Cursor-based replay).
- **Channel `send()` contract** — `text.trim() === "NO_REPLY"` treated as a lifecycle marker, not a message.
- **Cron layering** — frontmatter override + body append.
- **Sub-agent runner constraints** — `delegate_task` always stripped from sub-agent's tool allowlist; missing skill returns `{ ok: false }`, doesn't throw.

Plus structural checks `test` doesn't easily catch:

- **File layout** — channels in `agent/src/channels/`, tools in `agent/src/tools/`, sub-agent runner in `agent/src/agents/runner.ts`.
- **Wrapper script** — type-checks before restart; auto-reverts a bad self-edit by reverting the last commit in `agent/`.
- **Permission guards** — always-on `spec/` write guard; conditional `agent/` write guard when `PROTOS_SELF_EDIT=false`; `self-edit` skill hidden from the prompt when disabled.
- **Bundled skills present** — every skill listed in `spec/skills/` exists; spec recipes in `spec/channels/` and `spec/tools/` are reflected in `agent/src/`.

## What NOT to flag

- **Implementation details the spec doesn't constrain** — private function signatures, internal data shapes, code style. The spec describes the *what*; coding choices that satisfy the contract aren't drift.
- **User customizations.** The owner edits `agent/` directly to extend or specialize the assistant. If a divergence is plausibly an intentional customization (a new tool with no spec recipe, an extra channel, custom logic in an existing channel), report it as "extension" rather than "drift" so the owner can confirm intent without being prompted to revert.
- **Performance, error handling shape, log verbosity** — unless the spec explicitly mandates them.

## Output

A markdown report. Each finding has:

- **Spec reference** — the doc + section the contract lives in (`architecture.md → Sub-agent runner constraints`).
- **Code location** — file path + line range in `agent/`.
- **Required vs. actual** — one sentence each.
- **Classification** — `drift` (implementation diverges from spec), `extension` (probable user customization, flag for confirmation), or `cosmetic` (matches spec semantically but reads differently).

Lead the report with a one-line verdict: "no drift detected", "N findings (M drift, K extensions, …)", etc. Then the findings, grouped by classification, drift first.

End with a short "Suggested next steps" — for each `drift` finding, point at `update` (if the gap is a recent spec change the agent didn't get yet) or a manual fix (if the spec hasn't changed and the implementation just diverged).

## Run logs

Tee the report to `runtime/reviews/{ISO-timestamp}.log` (e.g. `runtime/reviews/2026-04-23T18:02:49Z.log`) so the owner can re-read past reviews without re-running. Show the path in the chat reply.
