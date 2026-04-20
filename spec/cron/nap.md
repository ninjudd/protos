---
schedule: "0 * * * *"
history: none
---

# Nap

Quick, opportunistic per-thread consolidation. Runs hourly. Threshold-gated — does nothing unless a thread has accumulated enough new messages to be worth processing.

## Approach

Do this work in a sub-agent via `delegate_task`. Each consolidation pass reads a thread tail and writes to memory, which uses too much context to do inline. Pass the sub-agent an instruction along the lines of: "Walk `runtime/threads/`. For each thread, compare its current message count to the cursor in the sidecar `*.cursor` file. For any thread where unconsolidated messages ≥ 50, read those messages, write anything worth remembering into `memory/` (creating files, updating existing ones, using `[[wiki-links]]`), then advance the cursor. Skip threads under the threshold. Return a brief summary." The sub-agent has the full file-edit toolset.

When the sub-agent returns, post its summary as your reply (or `NO_REPLY` if no thread crossed the threshold). The summary goes to the user's primary channel.

## Scope

`nap` does **only** per-thread consolidation. It deliberately skips:

- Journal sweep (`dream`'s job)
- Inbox sweep (`dream`'s job)
- Cross-thread correlations (`dream`'s job — `nap` looks at one thread at a time)
- Orphan check / hierarchy hygiene (`dream`'s job)

The split exists because per-thread consolidation needs to react quickly to bursts of activity (a long conversation in the morning shouldn't have to wait until 23:00), while full hygiene only needs to run once a day. Both jobs share the same cursor scheme, so neither re-consolidates content the other has already processed.

## Threshold

50 messages of unconsolidated tail is a reasonable starting point. The threshold is what prevents `nap` from doing meaningless work every hour on quiet threads — adjust if it's firing too often or letting threads grow too big between consolidations.
