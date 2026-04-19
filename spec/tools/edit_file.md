# edit_file

Surgical find-and-replace inside an existing file.

## Description (shown to the model)

Edit a file by replacing one unique substring. The `old_string` must appear exactly once — if it's missing or appears more than once, the call fails. Add surrounding context to `old_string` to make it unique.

## Input

```ts
{
  path: string,
  old_string: string,
  new_string: string,
}
```

## Output

Plain string — `Edited {path} ({delta:+N} chars)`.

On error: throws.

## Behavior

- **Reject path escapes** (same helper as `read_file` / `write_file`).
- **Self-edit guard:** when `LOGOS_SELF_EDIT=false` and the resolved path is under `agent/`, throw the same self-edit-disabled error.
- **Read the file**, count occurrences of `old_string` in its content.
  - **Zero occurrences:** throw `old_string not found in {path}`.
  - **Two or more occurrences:** throw `old_string is not unique in {path} ({N} matches); add more surrounding context`.
  - **Exactly one occurrence:** replace it with `new_string` and write the file.
- **Memory graph cache:** invalidate if the path is under `memory/` (same as `write_file`).

## Dependencies

Node standard library.

## Implementation notes

- This is the safer alternative to `write_file` with `mode: "replace"` for small targeted edits — no risk of accidentally clobbering the rest of the file.
- The uniqueness requirement forces the agent to add context, which makes diffs reviewable and prevents accidental wrong-edits.
