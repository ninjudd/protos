# find_memory

Resolve a wiki-link-style memory note name (or alias) to a path under `memory/`, plus the list of files that reference it.

## Description (shown to the model)

Look up a memory note by its name or alias (e.g. `coffee` or `people/justin`). Returns `{ found: true, path, backlinks }` on hit or `{ found: false }` on miss. **Does NOT lazy-create.** If you want a new note, pick a path and use `write_file` with `mode: "create"`.

## Input

```ts
{ name: string }
```

`name` is the wiki-link-style identifier (no `.md` extension). Optional path qualifier disambiguates collisions: `people/justin` vs `colleagues/justin`.

## Output

```ts
{ found: true, path: string, backlinks: string[] }
| { found: false }
```

- `path` is workspace-relative (e.g. `memory/people/justin.md`).
- `backlinks` is the list of other workspace-relative paths that contain `[[name]]` references to this note.

**Never returns `null`.** The discriminated union with the `found` boolean keeps the model's parsing unambiguous and leaves room for future fields (`suggestions: [...]` for fuzzy matches, etc.).

## Behavior

- See `architecture.md` → Memory format → Linking for the full resolution rules.
- Match on filename (without `.md`) OR `aliases:` list from frontmatter.
- On multiple matches: shortest path wins, then alphabetical.
- On no match: return `{ found: false }`. **Do not create any file.** Do not modify state.
- Use the cached memory graph (`runtime/memory-graph.json`) for fast lookup; rebuild via mtime check if the cache is stale.

## Dependencies

Internal `agent/src/memory.ts` module (graph + cache).

## Implementation notes

- The internal resolver may return `string | null`; the tool wrapper translates `null` → `{ found: false }`. Do NOT pass `null` through.
- This is the canonical way for the agent to look up memory; raw `read_file("memory/...")` works but skips name resolution and backlinks.
