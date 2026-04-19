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

The client stores its cursor at `runtime/clients/{session}.cursor` (default session: `chat`). This gives each client "pick up where I left off" semantics across reconnects. Messages queued while no client was connected (e.g., cron firing when the terminal is closed) are replayed on next connect.

## Owner filtering

None needed. The socket lives in the workspace filesystem; only processes with filesystem access to `runtime/logos.sock` can connect. Document that the socket should not be exposed over the network.

## Implementation notes

- The channel's `register()` is called once at daemon startup. It creates the socket server, registers the accept handler, and returns `{ channelId, ownerConversationId: "cli", send }`.
- The `send` function is called by the router with agent replies. It appends to the JSONL file (which the router already does via `threads.appendMessage`) — no direct socket write needed. All connected clients pick it up via their `fs.watch` subscription.
- Wait: the router ALREADY calls `appendMessage`. So the channel's `send` doesn't need to write to JSONL. But it DOES need to watch the JSONL file so it can push new lines to connected clients. Separate concerns: the channel maintains one `fs.watch` per connected client (or one watch shared across clients — implementation choice).
- On daemon shutdown, close all client sockets and remove the socket file.

## Client (`agent/logos chat`)

Lives at `agent/src/cli/chat.ts` — isolated from the rest of the codebase. Only imports `node:net`, `node:fs/promises`, `node:readline`, and the shared protocol types from `agent/src/channels/terminal-protocol.ts`. Does not import the router, agent, memory, or any other engine code.

Flags:

- `agent/logos chat` — resume from stored cursor, or default to last 20 messages on first connect
- `agent/logos chat --from-start` — replay the entire conversation
- `agent/logos chat --last N` — replay last N messages
- `agent/logos chat --new` — start fresh (advance cursor to end without replay)

Exit with `Ctrl+D`, `/quit`, or `/exit`. Client disconnect doesn't affect the daemon.

If the daemon isn't running, `chat` fails fast with: `error: daemon not running. Start it with "agent/logos start".` Don't auto-start.
