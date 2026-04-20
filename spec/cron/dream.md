---
schedule: "0 23 * * *"
history: none
---

# Dream

Daily deep consolidation. Distill recent thread activity, the day's journal, and the inbox into long-term memory; keep the memory graph tidy.

## Approach

Do this work in a sub-agent via `delegate_task`. Reading every active thread plus the journal and inbox is a lot of context — keeping it out of your main thread is the whole point. Pass the sub-agent a single instruction along the lines of: "Read the consolidation cursors and unconsolidated tail of every thread under `runtime/threads/`, plus today's `memory/journal/{date}.md` and any files under `memory/new/`. Update memory accordingly, advance cursors, run the orphan check, and return a brief summary." The sub-agent has the full file-edit toolset.

When the sub-agent returns, post its summary as your reply (or `NO_REPLY` if there was nothing notable). The summary goes to the user's primary channel.

## What the sub-agent should do

### 1. Threads — cross-thread consolidation

For each thread under `runtime/threads/{channelId}/{conversationId}.jsonl`:

- Read the sidecar cursor file at `runtime/threads/{channelId}/{conversationId}.cursor` (a single integer; missing means 0).
- Read messages from index `cursor` to end of file. Those are the unconsolidated messages.
- After processing, write the new cursor (= total message count) back to the sidecar.

Look for patterns *across* threads as well as within them — the same person mentioned in two channels, a decision in one thread that affects work tracked in another, recurring themes worth a top-level note. Cross-thread correlation is the reason `dream` reads everything at once.

### 2. Journal — promote anything worth keeping

Read `memory/journal/{today}.md` (the day's scratch pad). Promote anything that should outlive the day into structured memory files. Routine exchanges and one-off context can stay in the journal (or be discarded — the journal is the scratch pad, not a permanent record).

### 3. Inbox — sort and merge

Files under `memory/new/` are notes the agent (or a human in Obsidian) wrote without a clear home. Move each to a sensible folder, or merge into an existing file and delete the original. The inbox should be empty (or close to it) when you're done.

### 4. Hygiene — keep the graph navigable

The system prompt only sees **root-level** files (`memory/*.md`), so anything important must be reachable from a root file via `[[wiki-links]]`. After updating memory:

- **Orphan check.** Walk the link graph from each root file. Any non-root file not transitively reachable is an orphan. Either link it in (add a `[[wiki-link]]` from a relevant root file) or move/delete it. Decide based on whether the content still matters.
- **Promote hot content to root.** If a subfolder file is being read or referenced often, consider promoting it to root level (move the file, update incoming links) so it surfaces in the manifest directly.
- **Demote cold content.** Root files that haven't earned their spot — narrow scope, rarely referenced — can be moved into a subfolder and replaced with a `[[wiki-link]]` from a broader root file.
- **Update or remove contradictions.** When new information contradicts older content, fix it in place rather than letting both coexist.
- **Keep descriptions current.** Each file's frontmatter `description:` is what shows up in the manifest. Make sure it accurately reflects the file's contents.

## Style

The shape of `memory/` is yours to design. Markdown files, `[[wiki-links]]` to connect related notes, folders for whatever taxonomy makes sense. There's no rigid schema. Trust your judgment about what to write down, where it belongs, when to split or merge. Build the graph the way you'd want to read it.
