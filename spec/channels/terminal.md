# Terminal

A client-server chat channel. The daemon binds a Unix socket; `agent/logos chat` connects a client. Multiple clients can connect simultaneously, each tracking its own cursor into the conversation's JSONL file.

## Library

Node standard library only — `node:net`, `node:fs/promises`, `node:fs`, `node:readline`. No external dependencies.

## Environment variables

- `LOGOS_TERMINAL` — `true` (default) or `false`. When false, the channel skips registering and the socket isn't created. No other env vars required.
- `LOGOS_TERMINAL_SOCKET` — optional override for the socket path. Defaults to `runtime/logos.sock`.

## Channel identity

- `channelId`: `terminal`
- `conversationId`: `cli` (single persistent thread). All client sessions share this conversation by default.

## Setup

Zero setup. The channel is always generated during bootstrap. Start the daemon with `agent/logos start`, then connect with `agent/logos chat` from another terminal (or the same one, after backgrounding).

## Client-server protocol

Unix socket at `runtime/logos.sock`. Messages are newline-delimited JSON.

**Client → server:**

```
{"from": <int>}                               # first line: where to start replay
{"type": "message", "text": "..."}            # user message
```

**Server → client:**

```
{"type": "replay", "role": "user|assistant", "index": N, "text": "..."}
{"type": "live-start"}                         # caught up; now streaming live
{"type": "thinking"}                           # optional: agent is working
{"type": "message", "role": "assistant", "index": N, "text": "..."}
```

## Cursor-based replay

The conversation lives at `runtime/threads/terminal/cli.jsonl` like any other channel. Each line is a message with an implicit index (its line number).

When a client connects:

1. Client sends `{"from": N}` — the last index it has seen locally.
2. Server reads lines `N..end` from the JSONL file and streams them as `replay` messages.
3. Server emits `{"type": "live-start"}` to signal the client is caught up.
4. Server subscribes to the JSONL file via `fs.watch` and streams new appends as `message` events.

The client stores its cursor at `runtime/clients/{session}.cursor` (default session: `chat`). The default replay window on connect is computed as `from = min(stored_cursor, total - 20)` — show **at least the last 20 messages** for context, plus everything queued since the last disconnect if that span is longer. This means:

- A fresh `agent/logos chat` in an existing conversation always shows recent context (not an empty screen).
- Messages queued while disconnected (e.g., cron firing overnight) are never skipped.
- Whichever span is longer — "last 20" or "since cursor" — is the one you see.

The flags `--from-start`, `--last N`, and `--new` override this default.

## Owner filtering

None needed. The socket lives in the workspace filesystem; only processes with filesystem access to `runtime/logos.sock` can connect. Document that the socket should not be exposed over the network.

## Implementation notes

- The channel's `register()` is called once at daemon startup. It creates the socket server, registers the accept handler, and returns `{ channelId, ownerConversationId: "cli", send }`.
- The `send` function is called by the router with agent replies. It appends to the JSONL file (which the router already does via `threads.appendMessage`) — no direct socket write needed. All connected clients pick it up via their `fs.watch` subscription.
- Wait: the router ALREADY calls `appendMessage`. So the channel's `send` doesn't need to write to JSONL. But it DOES need to watch the JSONL file so it can push new lines to connected clients. Separate concerns: the channel maintains one `fs.watch` per connected client (or one watch shared across clients — implementation choice).
- On daemon shutdown, close all client sockets and remove the socket file.

## Client (`agent/logos chat`)

Lives at `agent/src/cli/chat.ts` — isolated from the rest of the codebase. Only imports `node:` standard library and the shared protocol types from `agent/src/channels/terminal-protocol.ts`. Does not import the router, agent, memory, or any other engine code. May additionally import thin npm packages for rendering (see below).

Flags:

- `agent/logos chat` — show at least the last 20 messages on connect (see [Cursor-based replay](#cursor-based-replay))
- `agent/logos chat --from-start` — replay the entire conversation
- `agent/logos chat --last N` — replay last N messages
- `agent/logos chat --new` — advance cursor to end without replay (show only new messages from this point on)

Exit with `Ctrl+D`, `/quit`, or `/exit`. Client disconnect doesn't affect the daemon.

If the daemon isn't running, `chat` fails fast with: `error: daemon not running. Start it with "agent/logos start".` Don't auto-start.

### Rendering

The client formats assistant replies for terminal readability:

- **Word-boundary wrapping.** Wrap output at the terminal width (`process.stdout.columns`, defaulting to 80 if unavailable). Preserve leading whitespace so indented content (list items, code blocks) lines up correctly on continuation lines. ANSI escape codes must not count toward visible width. Use a small well-maintained package such as [`wrap-ansi`](https://www.npmjs.com/package/wrap-ansi) — don't hand-roll the wrapping logic.
- **Inline markdown → ANSI.** The agent replies with markdown; render the common inline styles:
  - `**bold**` / `__bold__` → ANSI bold (`\x1b[1m` … `\x1b[22m`)
  - `*italic*` / `_italic_` → ANSI italic (`\x1b[3m` … `\x1b[23m`)
  - `` `code` `` → ANSI inverse or a dim color
  - Fenced code blocks (```` ``` ````) → indented, colored if you like
  - Headings (`# Title`) → bold, optionally with a blank line before
  - Links `[text](url)` → show as `text (url)` or with underline on `text`
- **User input and control messages** (your own lines, `> ` prompt, `[thinking]` indicators) are plain — no markdown parsing applied.
- Keep it simple: small regex-based renderer, not a full markdown parser. Block-level features beyond code fences and headings can be passed through as-is.

Re-render the *entire* assistant message after the wrapping pass — otherwise partial ANSI codes from one line can leak into the next.

### Chat flow and echo suppression

The loop should feel like a normal chat: the user types a line, sees the assistant reply, types the next line. Specifically:

- **Prompt:** `> ` (just the prompt character — no `you>` prefix). This is what the user sees before typing.
- **After Enter:** the user's typed line is already on screen (readline drew it). The client sends the message and waits.
- **Echo suppression:** the server's `fs.watch` broadcasts EVERY new line, including the user's own message. The client **MUST NOT re-render** the user's own just-sent message — the user already saw it on the prompt line. Suppress by matching recently-sent outgoing text within a short window (e.g. last 10 seconds, FIFO queue of pending sends; when a live `role: "user"` message matches the front of the queue, skip rendering and pop). This is pragmatic for a single-client setup; cross-client visibility can be improved later by tagging broadcasts with an origin.
- **During replay:** render all roles (user AND assistant). Replay exists to show the user their history; suppression only applies to *live* `role: "user"` messages.
- **After assistant reply:** redraw the `> ` prompt for the next turn.
- **Async output collision:** readline may have the `> ` prompt drawn when an assistant message arrives. Before printing, clear the current line (`readline.cursorTo(out, 0); readline.clearLine(out, 0);`), print the message, then `rl.prompt()` again. Otherwise the prompt and message smash onto the same line (`> logos: ...`).

Assistant messages are rendered with a single-line prefix like `logos:` followed by the body (wrapped, markdown applied). No `you:` or `you>` prefix appears in the rendered output anywhere — only in replay of historical user messages, and even then, plain text (no bold label) is fine.
