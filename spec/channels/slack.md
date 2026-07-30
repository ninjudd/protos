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

## Non-text content

Slack messages can carry uploaded files (`event.files[]`).

- **Image files** — for each file whose `mimetype` starts with `image/`, fetch the bytes from `file.url_private` with `Authorization: Bearer ${bot_token}` (Slack file URLs require auth even for the bot's own workspace), hash (sha256), write to `runtime/blobs/{sha256}.{ext}`, and attach as an `image` on the dispatched message. The message `text` is `event.text` if present; if blank, fall back to `file.title` or `file.initial_comment` (Slack often puts captions there for file-only posts).
- **Non-image files** (pdf, docx, audio, …) — normalize to a placeholder string (e.g. `[document: report.pdf]`, `[voice message]`). Transcription and document parsing are out of scope for v1.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Receipt ack

Slack has no bot-visible typing indicator, so the channel posts a short placeholder message as soon as it accepts an inbound event, before dispatching to the router. This gives the user immediate feedback that the message was received and is being worked on; the agent's actual reply lands as a follow-up message (the ack stays in the channel — there's no edit-in-place or deletion).

- The placeholder is a randomly-selected playful gerund — "Thinking…", "Pondering…", "Noodling…", "Mulling…", "Brewing…", etc. — followed by `...`. The variation is deliberate (mirrors the Claude Code thinking indicator) and the verb is picked fresh on every inbound event. Keep the list playful but tasteful; one of the entries should always be the literal `Thinking`.
- Post in the same channel, on the same thread (`thread_ts` from the inbound event for DM messages; the thread root — `event.thread_ts ?? event.ts` — for `app_mention`).
- If the post fails (e.g. transient API error), log `[slack] ack failed: <msg>` and continue — the dispatch still runs. Failing the receipt must never prevent the message from reaching the router.

## Markdown conversion

The assistant writes standard markdown; Slack uses "mrkdwn", a similar but incompatible format. Convert messages before posting with a library like `slackify-markdown`. Key differences:

- `**bold**` → `*bold*`
- `[text](url)` → `<url|text>`
- Headings (`#`, `##`, …) → bold line (Slack has no heading syntax)

Code spans, fenced code blocks, blockquotes, and bullet lists are already compatible.

## Diagnostics & reconnection

Bolt's Socket Mode client auto-reconnects on transient WebSocket disruption (ping timeouts, brief network drops). What turns a transient blip into a stuck-disconnected daemon is almost always an unhandled error in the receiver chain — Bolt logs it but the connection never re-establishes. Before designing any self-healing on top, the channel must surface what's actually happening.

The channel implementation MUST:

1. **Register a global error handler.** Catches errors that bubble out of listeners; without this they go to Bolt's logger only and can be drowned out by other output.

   ```ts
   app.error(async (err) => {
     console.error(`[slack] unhandled: ${err.message}`);
   });
   ```

2. **Log Socket Mode lifecycle events.** Reach the underlying client through `app.receiver` (a `SocketModeReceiver` when `socketMode: true`) and listen for the six state events the client emits: `connecting`, `connected`, `authenticated`, `reconnecting`, `disconnecting`, `disconnected`. Log each with a timestamp and the elapsed time since the last received event. (The client uses `eventemitter3` and does not emit `error` events — error surfacing is the `app.error()` handler's job.) Without this timeline, "the bot stopped responding" is indistinguishable from "the daemon crashed silently" or "Slack rate-limited us."

3. **Track `lastEventAt` in the channel module.** Update it whenever the client emits a `slack_event` (covers every inbound Slack event including messages and app_mentions). Not consumed for recovery yet — it's the signal a future watchdog or `/health`-style probe will read.

The `log_level` field in `channels.yaml` defaults to `warn`. Bump to `info` while diagnosing — Bolt emits reconnect attempts at INFO, and their presence or absence tells us whether the client is even trying.

**Recovery is out of scope for this section.** The agent process cannot self-restart via the `agent/protos` wrapper — the wrapper runs as a child of the daemon, so killing the daemon tears down the wrapper subprocess before it reaches the start step. Recovery designs go through an external supervisor (launchd, systemd, etc.) or in-process re-registration, addressed separately once the diagnostics here have identified the actual failure mode.

## Home tab

The Slack app Home tab can serve as a personal dashboard for the owner. When the owner opens the Home tab, Slack fires an `app_home_opened` event; the channel handles it and publishes a Block Kit view via `views.publish`.

### Enabling

Go to api.slack.com → your app → Features → App Home → toggle **Home Tab** on. Also enable **Messages Tab** if you want the owner to send DMs from the app tab.

Required additional bot scopes: `users:read` (to fetch the owner's display name).

### View contents

The published view should include:

1. **Header** — a time-aware greeting using the owner's display name fetched via `users.info` (`display_name` preferred, `real_name` fallback, `"there"` if the API call fails)
2. **Context** — the app icon (fetched once at startup via `users.info` on the bot user, `image_72`) and today's date
3. **Last note** — the most recent journal line from `memory/journal/YYYY-MM-DD.md` that is not a TODO, header, or timestamp line
4. **Open TODOs** — lines from today's journal containing `TODO:`, stripped of the prefix, up to 5 items, with a count in the section heading
5. **Action buttons** — one per row, each dispatching a fixed prompt to the router as if the owner had typed it:
   - 🗓️ Start weekly review → `"start my weekly review"`
   - ☀️ Start my day → `"let's start my day"`
   - 🌙 Daily review → `"let's do a daily review"`
   - 📝 Add a note → `"I want to add a note"`
6. **Refresh button** — re-publishes the view without dispatching to the router

### Owner guard

Only publish the view when `event.user === ownerUserId`. Ignore all other users.

### Refresh

Extract the view-publishing logic into a `publishHomeView()` helper inside `createChannel()`. Both the `app_home_opened` event handler and the refresh button action call this helper, so the view is always built from the same code path.

### Action routing

Register a single `app.action(/^home_/)` handler. On click: ack immediately, then either re-publish (refresh) or post the prompt text as a chat message to the owner's DM channel and dispatch it to the router.

## Notes

- Socket Mode connects via WebSocket — no HTTP server or public URL needed
- In channels, respond only to mentions or DMs to avoid noise
