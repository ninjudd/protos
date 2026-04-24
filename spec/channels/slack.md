# Slack

## Library

`@slack/bolt` — official Slack app framework.

## Configuration

`config/channels.yaml` entry:

```yaml
slack:
  bot_token: $SLACK_BOT_TOKEN
  app_token: $SLACK_APP_TOKEN
  owner_id: U123ABC456
```

- **`bot_token`** — Bot User OAuth Token (starts with `xoxb-`)
- **`app_token`** — App-Level Token for Socket Mode (starts with `xapp-`)
- **`owner_id`** — your Slack member ID. Open your profile in Slack, click the "…" menu, and "Copy member ID".

## Setup

1. Create a Slack app at [api.slack.com](https://api.slack.com/apps)
2. Enable Socket Mode (avoids needing a public URL)
3. Add bot token scopes: `chat:write`, `channels:history`, `groups:history`, `im:history`, `app_mentions:read`
4. Subscribe to events: `message.channels`, `message.groups`, `message.im`, `app_mention`
5. Install the app to your workspace
6. Install `@slack/bolt`

## Markdown conversion

The assistant writes standard markdown; Slack uses "mrkdwn", a similar but incompatible format. Convert messages before posting with a library like `slackify-markdown`. Key differences:

- `**bold**` → `*bold*`
- `[text](url)` → `<url|text>`
- Headings (`#`, `##`, …) → bold line (Slack has no heading syntax)

Code spans, fenced code blocks, blockquotes, and bullet lists are already compatible.

## Notes

- Socket Mode connects via WebSocket — no HTTP server or public URL needed
- In channels, respond only to mentions or DMs to avoid noise
