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

## Notes

- BlueBubbles exposes iMessage via REST API and webhooks
- Receiving messages requires a webhook endpoint — this channel needs a local HTTP server
- Supports text, attachments, reactions, and read receipts
- The Private API option in BlueBubbles enables typing indicators and richer features
