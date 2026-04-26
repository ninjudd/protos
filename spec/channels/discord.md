# Discord

## Library

`discord.js` — the standard Discord bot library for Node.js.

## Configuration

`config/channels.yaml` entry:

```yaml
discord:
  bot_token: $DISCORD_BOT_TOKEN
  owner_id: "123456789012345678"
```

- **`bot_token`** — from the [Discord Developer Portal](https://discord.com/developers/applications)
- **`owner_id`** — your Discord user ID. Enable Developer Mode in Discord (Settings → Advanced), then right-click your name and "Copy User ID".

## Setup

1. Create an application and bot in the Discord Developer Portal
2. Enable the Message Content intent (required to read message text)
3. Invite the bot to your server with appropriate permissions
4. Install `discord.js`

## Non-text content

Discord messages can carry attachments (`message.attachments`), embeds, stickers, and reactions.

- **Image attachments** — for each attachment whose `contentType` starts with `image/`, fetch the bytes from `attachment.url` (Discord CDN URLs don't require auth), hash (sha256), write to `runtime/blobs/{sha256}.{ext}` (extension from `contentType`), and attach as an `image` on the dispatched message. The message `text` is `message.content` (often empty for image-only posts).
- **Non-image attachments** (pdf, audio, video, …) and **stickers** — normalize to a placeholder string (e.g. `[document: report.pdf]`, `[sticker: WumpusWave]`). Transcription and document parsing are out of scope for v1.
- **Embeds** — the auto-generated link-preview cards. Ignore for v1; the link itself is in `message.content`.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Notes

- discord.js handles the gateway WebSocket natively — no HTTP server needed
- In servers, the bot receives all messages in channels it has access to — use mention gating to avoid responding to everything
- Supports text, embeds, attachments, threads, and reactions
