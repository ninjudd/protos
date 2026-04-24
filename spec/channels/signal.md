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

## Notes

- signal-cli is a Java application — requires a JRE on the host
- No npm library — communicate with signal-cli via its JSON-RPC interface or by spawning the process
- Supports text, attachments, reactions, and group messages
