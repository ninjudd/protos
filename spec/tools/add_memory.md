# add_memory

Create a new memory file and preserve existing `[[wiki-link]]` resolutions.

## Description (shown to the model)

Create a new memory note. Takes a wiki-link-style name (no `memory/` prefix, no `.md` extension — same form as `find_memory` and `rename_memory`). Before the file exists, any existing `[[...]]` reference whose resolution would be shadowed by the new file is rewritten to a path-qualified form that preserves its original target. Fails if the target already exists — use `write_file` with `mode: "replace"` or `edit_file` to update an existing file.

## Input

```ts
{ name: string, content: string }
```

- `name` — wiki-link-style identifier. Bare basename for root (`"coffee"`) or path-qualified for a subfolder (`"preferences/coffee"`). Don't include `memory/` or `.md`.
- `content` — the full file body including optional YAML frontmatter.

## Output

```ts
{ ok: true, path: string, updatedFiles: string[], updatedLinks: number }
```

- `path` — workspace-relative path of the created file (e.g. `"memory/preferences/coffee.md"`).
- `updatedFiles` — memory files whose contents were modified to preserve shadowed link resolutions.
- `updatedLinks` — total number of `[[...]]` occurrences rewritten.

Always returns the discriminated shape — never `null`.

## Behavior

1. Resolve `name` to a workspace path: `memory/{name}.md`. Refuse if the resulting path escapes `memory/` or ends with something other than `.md` after normalization.
2. Refuse if the file already exists.
3. **Pre-create snapshot.** Build a resolution map: for every `[[...]]` reference in memory files, record its resolved target file.
4. Create the file (creating parent directories as needed).
5. **Post-create snapshot.** Rebuild the resolution map.
6. For every reference whose resolution changed due to the new file's presence, rewrite it to the minimal form that restores its original target (path-qualifying the link where needed). Collect the edits in `updatedFiles` / `updatedLinks`.
7. Invalidate the memory graph cache.

The snapshot-and-rewrite step is identical to `rename_memory`'s link-preservation algorithm — both tools should share the same internal helper.

## Dependencies

Internal `agent/src/memory.ts` module (graph + cache).

## Implementation notes

- Reuse the wiki-link parser the graph builder uses — don't hand-roll regex that mishandles embeds (`![[...]]`) or aliases (`[[name|display]]`).
- The agent may also call `write_file` with `mode: "create"` under `memory/`, but that bypasses the link-preservation step. Prefer `add_memory` for any memory-file creation that could shadow existing bare-name links.
