# add_journal_entry

Append a note to today's journal entry.

## Description (shown to the model)

Sugar for journaling. Appends a timestamped line to today's journal at `memory/journal/{YYYY-MM-DD}.md`. Use this for transient observations, things to consolidate later, working notes. For structured long-term memory, use the canonical `write` (or `edit`) tool with the appropriate path under `memory/`.

## Input

```ts
{ text: string }
```

## Output

Plain string — `Wrote journal entry to memory/journal/{YYYY-MM-DD}.md`.

## Behavior

- Compute today's date in the local timezone as `YYYY-MM-DD`.
- Append to `memory/journal/{date}.md` with a timestamp prefix (e.g., `[HH:MM] {text}`).
- Create the file if it doesn't exist; create the `memory/journal/` directory if needed.
- Insert a `\n` separator if the existing file doesn't end with one.
- **Memory graph cache:** invalidate `runtime/memory-graph.json`.

## Dependencies

Node standard library.

## Implementation notes

- This is a thin convenience targeting `memory/journal/{date}.md`. Kept as its own tool because journaling is the most common write pattern; the agent shouldn't have to compute today's date and assemble the path each time, and journaling doesn't fit cleanly into the canonical `write` (overwrites) or `edit` (find-and-replace) shapes.
- The `dream` cron job (`spec/cron/dream.md`) reads the day's journal and promotes important items to structured files under `memory/`.
