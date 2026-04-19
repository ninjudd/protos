# write_file

Write a file to the workspace.

## Description (shown to the model)

Write to a file. Modes: `create` (fail if file exists), `append` (add to the end with a newline separator if needed), `replace` (overwrite the entire file).

## Input

```ts
{
  path: string,
  content: string,
  mode: "create" | "append" | "replace",
}
```

## Output

Plain string — a one-line confirmation:

- `create`: `Created {path} ({N} chars)`
- `append`: `Appended to {path} ({N} chars)`
- `replace`: `Replaced {path} ({N} chars)`

On error: throws.

## Behavior

- **Reject path escapes** (same helper as `read_file`).
- **Self-edit guard:** when `LOGOS_SELF_EDIT=false` and the resolved path is under `agent/`, throw `self-edit is disabled; refusing to write under agent/`.
- **`create` mode:** use `fs.writeFile` with `{ flag: "wx" }`. Throw `File already exists: {path}` on `EEXIST`.
- **`append` mode:** read existing content; if it has any content and doesn't end with `\n`, prepend `\n` before the new content so appends naturally separate. Then `fs.appendFile`. Tolerate `ENOENT` (treat as empty).
- **`replace` mode:** plain `fs.writeFile`. Overwrites without warning.
- **Create parent directories** as needed (`fs.mkdir` with `recursive: true`).
- **Memory graph cache:** if the path is under `memory/`, invalidate `runtime/memory-graph.json` (delete or touch the cache file) so the next `find_memory` call rebuilds.

## Dependencies

Node standard library.

## Implementation notes

- Share path-safety + self-edit-guard helpers with `edit_file` and `read_file`.
- The `\n` separator on append matters — without it, two appends concatenate on the same line.
