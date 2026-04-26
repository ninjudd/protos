# Matrix

## Library

`matrix-js-sdk` — official Matrix client SDK for JavaScript.

## Configuration

`config/channels.yaml` entry:

```yaml
matrix:
  homeserver: $MATRIX_HOMESERVER
  user_id: $MATRIX_USER_ID
  access_token: $MATRIX_ACCESS_TOKEN
  owner_id: "@justin:matrix.org"
```

- **`homeserver`** — homeserver URL (e.g. `https://matrix.org`).
- **`user_id`** — the bot's Matrix user ID (e.g. `@protos:matrix.org`).
- **`access_token`** — access token for the bot account.
- **`owner_id`** — the owner's Matrix user ID.

## Setup

1. Create a Matrix account for the bot on your homeserver
2. Generate an access token
3. Install `matrix-js-sdk`

## Non-text content

Matrix events use `m.room.message` with a `msgtype`: `m.text`, `m.image`, `m.audio`, `m.video`, `m.file`, `m.sticker`.

- **Image events (`m.image`)** — `content.url` is an `mxc://` URI; convert to the homeserver's media-download URL (`${homeserver}/_matrix/media/v3/download/${server}/${mediaId}`), fetch with `Authorization: Bearer ${access_token}`, hash (sha256), write to `runtime/blobs/{sha256}.{ext}` (extension from `content.info.mimetype`), and attach as an `image` on the dispatched message. The message `text` is `content.body` (Matrix uses `body` as the human-readable label/caption).
- **`m.audio`, `m.video`, `m.file`, `m.sticker`** — normalize to a placeholder string (e.g. `[voice message]`, `[document: report.pdf]`, `[sticker: 👋]`). Transcription and document parsing are out of scope for v1.
- **Encrypted rooms** — image events in encrypted rooms wrap the URL in an encrypted payload; decrypt before fetching. Out of scope for v1; revisit when E2EE support lands.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Notes

- matrix-js-sdk handles sync via long polling — no HTTP server needed
- Supports text, formatted messages (HTML), files, and end-to-end encryption (with additional setup)
- Self-hosted homeservers give you full control over data
