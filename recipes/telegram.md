# Telegram

## Library

`grammy` — modern Telegram bot framework for Node.js.

## Environment variables

- `TELEGRAM_BOT_TOKEN` — get this from [@BotFather](https://t.me/BotFather)
- `TELEGRAM_OWNER_CHAT_ID` — the chat ID of the owner's private conversation with the bot. Send a message to the bot, then check the logs or use [@userinfobot](https://t.me/userinfobot) to find it.

## Setup

1. Create a bot via BotFather and copy the token
2. Install `grammy`
3. Implement the channel in `src/channels/telegram.ts`

## Owner filtering

Only forward messages where the chat ID matches `TELEGRAM_OWNER_CHAT_ID`. Silently ignore all other messages.

## Notes

- grammY handles long polling natively — no webhook or HTTP server needed
- `bot.start()` runs long polling and does not resolve until the bot stops — don't await it during registration or it will block startup. Fire and forget it.
- Supports text, photos, documents, voice messages, and stickers. For non-text content, normalize to a placeholder string (e.g. `[photo]`, `[voice message]`) since the agent only handles text for now.
