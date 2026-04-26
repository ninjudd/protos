# WhatsApp

## Library

`@whiskeysockets/baileys` — unofficial WhatsApp Web API. Used by both OpenClaw and NanoClaw.

## Configuration

`config/channels.yaml` entry:

```yaml
whatsapp:
  owner_id: "15551234567"
```

- **`owner_id`** — your WhatsApp JID (typically your phone number with country code, no `+`). Baileys authenticates by linking as a companion device (QR code or pairing code) on first run — no credentials in the YAML.

## Setup

1. Install `@whiskeysockets/baileys` and `qrcode-terminal`
2. On first run, scan the QR code with WhatsApp on your phone to link the device

## Non-text content

WhatsApp messages carry typed payloads: `imageMessage`, `audioMessage`, `videoMessage`, `documentMessage`, `stickerMessage`, etc.

- **Images** (`imageMessage`) — download via Baileys' `downloadMediaMessage(msg, "buffer", {})`, hash (sha256), write to `runtime/blobs/{sha256}.{ext}` (extension from `imageMessage.mimetype`), and attach as an `image` on the dispatched message. The message `text` is `imageMessage.caption` if present, empty string otherwise.
- **Audio (`audioMessage`), documents (`documentMessage`), video (`videoMessage`), stickers (`stickerMessage`)** — normalize to a placeholder string (e.g. `[voice message]`, `[document: receipt.pdf]`, `[video]`, `[sticker]`). Transcription and document parsing are out of scope for v1.

See `architecture.md` → Storage → Attachments for the blob layout and event schema.

## Notes

- Auth state (keys and session) should be persisted to disk so you don't need to re-scan on restart
- Baileys connects via WebSocket — no HTTP server needed
- Supports text, images, audio, video, and documents
- Unofficial library — WhatsApp could break compatibility at any time
