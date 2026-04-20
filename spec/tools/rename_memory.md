# rename_memory

Move a memory file and rewrite `[[wiki-links]]` to preserve every existing reference's target.

## Description (shown to the model)

Rename or move a memory note. Takes wiki-link-style names for both old and new (no `memory/` prefix, no `.md` extension — same form as `find_memory` and `add_memory`). After the move, every `[[...]]` reference is rewritten as needed so it resolves to the same file it did before the rename. Fails if the new target already exists.

## Input

```ts
{ oldName: string, newName: string }
```

- `oldName` — wiki-link-style identifier for the existing file (e.g. `"drinks/coffee"`).
- `newName` — wiki-link-style identifier for the destination (e.g. `"coffee"` for a root-level move, or `"preferences/coffee"` for a new subfolder).

## Output

```ts
{ ok: true, oldPath: string, newPath: string, updatedFiles: string[], updatedLinks: number }
```

- `oldPath` / `newPath` — workspace-relative paths (e.g. `"memory/drinks/coffee.md"` → `"memory/coffee.md"`).
- `updatedFiles` — memory files whose contents were modified to preserve link resolutions.
- `updatedLinks` — total number of `[[...]]` occurrences rewritten.

Always returns the discriminated shape — never `null`.

## Behavior

1. Resolve `oldName` to `memory/{oldName}.md` and `newName` to `memory/{newName}.md`. Refuse if either path escapes `memory/`.
2. Refuse if `oldName` doesn't exist or `newName` already exists.
3. **Pre-rename snapshot.** Build a resolution map: for every `[[...]]` reference in memory files, record its resolved target file.
4. Move the file with `fs.rename` (creating `newName`'s parent directory if missing).
5. **Post-rename snapshot.** Rebuild the resolution map.
6. For every reference whose resolved target changed, rewrite it to the minimal form that restores its original target. Two sub-cases:
   - References whose original target was the renamed file → rewrite to `[[newName]]` (or path-qualified if `newName` would be ambiguous).
   - References whose original target was an **unrelated** file that got shadowed by the rename → rewrite to a path-qualified form of the unrelated file's location.
7. Invalidate the memory graph cache.

The snapshot-and-rewrite step shares a helper with `add_memory`.

### Example

Given `memory/drinks/coffee.md` and `memory/table/coffee.md`, an existing `[[coffee]]` reference resolves to `drinks/coffee` (alphabetical tiebreak on shortest-path). Renaming `table/coffee` → `coffee` (to root) moves the table entry to the root; a bare `[[coffee]]` would now resolve to the root file (shortest path wins). To preserve the original target, `[[coffee]]` is rewritten to `[[drinks/coffee]]`.

## Dependencies

Internal `agent/src/memory.ts` module.

## Implementation notes

- Reuse the wiki-link parser the graph builder uses.
- `aliases:` frontmatter entries on other files are about names, not paths, and aren't affected by the move.
