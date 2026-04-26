# find_memory

Resolve a wiki-link-style memory name to every matching file, sorted by shortest path first.

## Description (shown to the model)

Look up a memory note by wiki-link-style name. Returns a list of every file whose filename (without `.md`) or `aliases:` list matches the name, sorted with the shortest path first — the first entry is what bare-name wiki-link resolution would pick. Inputs use the wiki-link form (no `memory/` prefix, no `.md` extension), matching `add_memory` and `rename_memory`. Does NOT lazy-create — if no match, `matches` is empty.

## Input

```ts
{ name: string }
```

`name` is the wiki-link-style identifier. Bare (`"coffee"`) or path-qualified (`"people/justin"`) — path qualifier narrows matches to files at that sub-path.

## Output

```ts
{ matches: Array<{ path: string, backlinks: string[] }> }
```

- `path` — workspace-relative path (e.g. `"memory/preferences/coffee.md"`), suitable for passing to `read`.
- `backlinks` — workspace-relative paths of other memory files that contain `[[...]]` references resolving to this file.

`matches` is empty on no match — `matches.length === 0` means "not found." Never `null` or `undefined`.

## Behavior

- See `architecture.md` → Memory format → Linking for the full resolution rules.
- Match on filename (without `.md`) OR `aliases:` list from frontmatter.
- Sort results by **path length ascending**, then alphabetical. The first entry is what bare `[[name]]` resolution would produce; subsequent entries are alternatives the agent can pick deliberately.
- On path-qualified input (e.g. `"people/justin"`): only match files whose path ends with the qualifier.
- Use the cached memory graph (`runtime/memory-graph.json`) for fast lookup; rebuild via mtime check if the cache is stale.

## Dependencies

Internal `agent/src/memory.ts` module (graph + cache).

## Implementation notes

- The internal resolver may deal in file paths; the tool wrapper always returns the wrapped list shape. Do NOT pass `null` through.
- This is the canonical way for the agent to look up memory; raw `read("memory/...")` works but skips name resolution and backlinks.
