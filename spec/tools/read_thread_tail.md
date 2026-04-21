# read_thread_tail

Read the unconsolidated messages of a conversation thread.

## Description (shown to the model)

Return the messages in a thread that haven't been consolidated yet (everything from the cursor to the end of the JSONL), plus a `newCursor` value to pass to `advance_thread_cursor` after you've successfully processed them. Pair with `list_threads` and `advance_thread_cursor` to run a consolidation pass.

## Input

```ts
{ channelId: string, conversationId: string }
```

## Output

```ts
{
  // Post-render-filter shape — NOT the raw event schema. Each entry is
  // one rendered message (user text or last assistant text of a turn);
  // tool calls, tool results, and audit events have been stripped.
  messages: Array<{ role: "user" | "assistant", text: string, timestamp: string }>,
  newCursor: number  // count of raw events at read time (what to pass to advance_thread_cursor)
}
```

`messages` is empty if the cursor is already at the end of the thread. Always returns the discriminated shape — never `null`.

## Behavior

- Reads the sidecar cursor file (missing means `0`), reads the JSONL.
- **Turn-boundary backward-alignment at the cursor.** If the stored cursor `C > 0` lands mid-turn (i.e. `events[C-1].turn_id === events[C].turn_id`), decrement `C` until that's no longer true before reading. This happens when a previous `read_thread_tail` snapshot caught a turn mid-flight and `advance_thread_cursor` stored a cursor pointing into its middle; the next read re-includes the full turn so the partial consolidation gets a second pass once the turn is complete. The rule is the same one used by `readEvents` at its head (see `build.md` → step 2), so a single invariant covers both surfaces: **a slice never starts mid-turn.** The backward walk is bounded by one turn — it never grows over time.
- Applies the same render filter channels use (see `architecture.md` → Cursor-based replay → Render filter): for each `turn_id`, emit the user event and the **last** non-empty, non-`NO_REPLY` assistant text; hide intermediate assistant texts, tool calls, tool results, and audit events. Consolidation cares about the conversation, not the agent's internal reasoning or its lifecycle markers.
- `newCursor` is the count of total **events** (lines) at read time. Pass this to `advance_thread_cursor` to mark the full span consolidated. **Don't compute the new cursor yourself** — using the value returned here avoids a race where events arriving between this call and `advance_thread_cursor` would otherwise be skipped.
- Does NOT advance the cursor. The agent is responsible for advancing only after the messages have been successfully processed (e.g. summarized into memory).
- The backward-aligned re-read can cause the same turn to appear in `messages` across two successive consolidation passes (once partial, once complete). The consolidation skill must extend existing memory entries rather than duplicate — it already handles this case.

## Dependencies

Built-in, no external services. Reads `runtime/threads/`.

## Implementation notes

- Same JSONL parsing as the regular thread loader. Skip malformed lines silently (consistent with other thread readers).
- If `channelId` or `conversationId` doesn't correspond to an existing thread, return `{ messages: [], newCursor: 0 }`.
