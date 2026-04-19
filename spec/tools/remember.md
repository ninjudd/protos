# remember

Append a note to today's journal entry.

## Description (shown to the model)

Sugar for journaling. Appends a timestamped line to today's journal at `memory/journal/{YYYY-MM-DD}.md`. Use this for transient observations, things to consolidate later, working notes. For structured long-term memory, use `write_file` with the appropriate path under `memory/`.

## Input

```ts
{ text: string }
```

## Output

Plain string — `Remembered to memory/journal/{YYYY-MM-DD}.md`.

## Behavior

- Compute today's date in the local timezone as `YYYY-MM-DD`.
- Append to `memory/journal/{date}.md` with a timestamp prefix (e.g., `[HH:MM] {text}`).
- Create the file if it doesn't exist; create the `memory/journal/` directory if needed.
- Insert a `\n` separator if the existing file doesn't end with one (same as `write_file` append).
- **Memory graph cache:** invalidate.

## Dependencies

Node standard library.

## Implementation notes

- This is a thin convenience over `write_file({path: "memory/journal/{date}.md", content: text, mode: "append"})`. Kept as its own tool because journaling is the most common write pattern; the agent shouldn't have to compute today's date and assemble the path each time.
- The consolidate-memories cron job (`spec/cron/consolidate-memories.md`) reads the day's journal and promotes important items to structured files under `memory/`.
