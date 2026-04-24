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

## Notes

- matrix-js-sdk handles sync via long polling — no HTTP server needed
- Supports text, formatted messages (HTML), files, and end-to-end encryption (with additional setup)
- Self-hosted homeservers give you full control over data
