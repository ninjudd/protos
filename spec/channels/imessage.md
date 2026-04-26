# iMessage

## Library

No npm library needed. Uses the [BlueBubbles](https://bluebubbles.app) server's REST API.

## Prerequisites

- A Mac with Messages.app signed into your Apple ID
- BlueBubbles server installed and running on that Mac (can be the same machine running Protos)

## Configuration

`config/channels.yaml` entry:

```yaml
imessage:
  bluebubbles_url: $BLUEBUBBLES_URL
  bluebubbles_password: $BLUEBUBBLES_PASSWORD
  owner_id: "+15551234567"
```

- **`bluebubbles_url`** — BlueBubbles server URL (e.g. `http://localhost:1234`).
- **`bluebubbles_password`** — server password for API authentication.
- **`owner_id`** — the owner's iMessage handle (phone number in E.164 format, or Apple ID email).

## Setup

1. Install and configure BlueBubbles server on a Mac
2. Use the REST API to send messages and webhooks to receive them

## Non-text content

BlueBubbles webhook payloads include `attachments[]` with `mimeType`, `transferName`, `guid`.

- **Image attachments** — for each attachment whose `mimeType` starts with `image/`, fetch the bytes from `${bluebubbles_url}/api/v1/attachment/${guid}/download` (using the configured `bluebubbles_password`), hash (sha256), write to `runtime/blobs/{sha256}.{ext}`, and attach as an `image` on the dispatched message. The message `text` is the message body if present, empty string otherwise. iPhone photos often arrive as `image/heic` — current frontier LLMs accept HEIC; transcode to JPEG only if the active profile rejects it.
- **Non-image attachments** (audio, video, documents) — normalize to a placeholder string (e.g. `[voice message]`, `[document: receipt.pdf]`). Transcription and document parsing are out of scope for v1.
- **Reactions, typing indicators, read receipts** — don't forward to the agent.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Notes

- BlueBubbles exposes iMessage via REST API and webhooks
- Receiving messages requires a webhook endpoint — this channel needs a local HTTP server
- Supports text, attachments, reactions, and read receipts
- The Private API option in BlueBubbles enables typing indicators and richer features
