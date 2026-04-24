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

## Notes

- discord.js handles the gateway WebSocket natively — no HTTP server needed
- In servers, the bot receives all messages in channels it has access to — use mention gating to avoid responding to everything
- Supports text, embeds, attachments, threads, and reactions
