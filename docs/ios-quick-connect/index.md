---
title: "iOS Quick Connect"
summary: "Per-integration guide for connecting messaging channels to OpenClaw from the managed iOS app"
read_when:
  - Building the iOS app's channel-connection UI
  - Adding a new integration to the iOS app
  - Understanding the credential-flow between the iOS app and a user's container
---

# iOS Quick Connect

This section covers how to implement a **quick connect** for each messaging integration when users run OpenClaw through the managed iOS app. Each guide describes the credential model, the recommended in-app UX, and how those credentials are applied to the user's dedicated OpenClaw container.

## Architecture overview

```
iOS App (SwiftUI)
      │
      │  HTTPS + Supabase JWT
      ▼
files.spark.ooo  (file-api pod)
      │
      │  Azure Files mount (shared with container)
      ▼
~/.openclaw/config.json  inside user's AKS pod
      │
      ▼
openclaw gateway  (picks up config change and reconnects)
```

The iOS app writes channel credentials through the file API. The container's OpenClaw process watches for config changes and reconnects automatically.

### Config-patch endpoint

All quick-connect flows described here assume a `/config/channels` PATCH endpoint exposed by the file-api pod:

```http
PATCH /config/channels
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{
  "channel": "discord",
  "config": { "botToken": "...", "enabled": true }
}
```

The file-api pod merges this payload into `~/.openclaw/config.json` under `channels.<channel>` and sends a `SIGHUP` (or equivalent reload signal) to the running gateway process.

> If the config-patch endpoint does not yet exist, you can achieve the same result by writing the full config file via `PUT /files/config.json` and restarting the container.

## Auth method glossary

| Method | What the user does in the iOS app |
|--------|-----------------------------------|
| **Token paste** | User copies a token from a web UI and pastes it into a secure text field. |
| **OAuth in-app** | iOS app opens a `ASWebAuthenticationSession` to the service's OAuth URL; the token is captured automatically on redirect. |
| **QR scan** | iOS app displays a QR code fetched from the container; user scans it with the target messaging app on another device. |
| **Guided form** | Multi-step form collects several credentials (server URL, bot ID, secret, etc.). |
| **Phone registration** | User provides a phone number; the container sends/receives an OTP via the service's API. |

## Integration index

| Integration | Auth method | Complexity |
|-------------|-------------|------------|
| [Discord](discord) | Token paste | Low |
| [Telegram](telegram) | Token paste | Low |
| [Slack](slack) | OAuth in-app or token paste | Medium |
| [WhatsApp](whatsapp) | QR scan | Medium |
| [Signal](signal) | Phone registration or QR link | High |
| [iMessage (BlueBubbles)](bluebubbles) | Guided form (server URL + password) | Medium |
| [Microsoft Teams](msteams) | Guided form (Azure credentials) | High |
| [Matrix](matrix) | Token paste or OAuth/SSO | Medium |
| [Feishu / Lark](feishu) | Token paste | Low |
| [Nostr](nostr) | Key paste or in-app keygen | Low |
| [Twitch](twitch) | OAuth in-app | Low |
| [Zalo](zalo) | Token paste | Low |
| [Mattermost](mattermost) | Token paste + server URL | Low |
| [IRC](irc) | Guided form (server + nick) | Low |
| [LINE](line) | Token paste | Low |
| [Tlon / Urbit](tlon) | Guided form | Medium |
