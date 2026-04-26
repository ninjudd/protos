# Signal

## Library

`signal-cli` — a command-line interface for Signal, used as a bridge.

## Prerequisites

- `signal-cli` installed on the host (via Homebrew, apt, or manual install)
- A phone number to register with Signal

## Configuration

`config/channels.yaml` entry:

```yaml
signal:
  phone_number: $SIGNAL_PHONE_NUMBER
  owner_id: "+15551234567"
```

- **`phone_number`** — the bot's registered Signal phone number (full E.164 format, with `+`).
- **`owner_id`** — the owner's Signal phone number (same format).

## Setup

1. Install `signal-cli` on the host
2. Register or link a phone number
3. Use `signal-cli` in daemon mode with JSON-RPC or poll for incoming messages

## Non-text content

Signal messages carry `dataMessage.attachments[]` with `contentType`, `id`, `filename`, `size`. signal-cli writes incoming attachments to its local attachment directory on receipt (default `~/.local/share/signal-cli/attachments/`).

- **Image attachments** — for each attachment whose `contentType` starts with `image/`, read the bytes from signal-cli's local file (or call its `getAttachment` JSON-RPC method), hash (sha256), copy to `runtime/blobs/{sha256}.{ext}`, and attach as an `image` on the dispatched message. The message `text` is `dataMessage.message` if present, empty string otherwise.
- **Non-image attachments** (audio, video, documents, stickers) — normalize to a placeholder string (e.g. `[voice message]`, `[document: receipt.pdf]`, `[sticker]`). Transcription and document parsing are out of scope for v1.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Notes

- signal-cli is a Java application — requires a JRE on the host
- No npm library — communicate with signal-cli via its JSON-RPC interface or by spawning the process
- Supports text, attachments, reactions, and group messages
