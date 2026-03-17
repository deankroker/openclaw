---
summary: "Quick connect Telegram from the managed iOS app using a bot token"
read_when:
  - Implementing Telegram channel setup in the iOS app
  - Understanding the token-based connect flow
title: "Telegram — iOS Quick Connect"
sidebarTitle: "Telegram"
---

# Telegram — iOS Quick Connect

**Pattern: paste bot token**

Telegram is the easiest provider to connect from a managed iOS app. The user creates a bot
through @BotFather (one-time, takes under two minutes), copies the token, and pastes it into
the app. No OAuth, no QR code, no server-side redirect handling needed.

## User journey

```
User                iOS App              file-api               OpenClaw Pod
 │                     │                    │                        │
 │  Tap "Add Telegram" │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  Show token input  │                        │
 │◄────────────────────│                    │                        │
 │  Paste 123:abc…     │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  POST /credentials/telegram                 │
 │                     │  { token, dmPolicy, allowFrom }             │
 │                     │────────────────────►                        │
 │                     │                    │  Write telegram.json   │
 │                     │                    │───────────────────────►│
 │                     │                    │                        │  Reload config
 │                     │  POST /rpc/reload  │                        │◄──────────────
 │                     │────────────────────►                        │
 │                     │  ✓ Connected       │                        │
 │◄────────────────────│                    │                        │
```

## What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `botToken` | Bot token | Yes | Format `123456:ABCdef…` from @BotFather |
| `dmPolicy` | Who can message the bot | Yes | Default: `pairing` |
| `allowFrom` | Telegram usernames or user IDs | No | Leave blank for pairing-only |

## Implementation guide

### Step 1 — Token input screen

Present a single `SecureField` for the bot token. Add a help link that opens the @BotFather
deep link `tg://resolve?domain=BotFather` to start bot creation, or fall back to
`https://t.me/BotFather` if Telegram is not installed.

```swift
// Open BotFather to guide user
func openBotFather() {
    let tgURL = URL(string: "tg://resolve?domain=BotFather")!
    let webURL = URL(string: "https://t.me/BotFather")!
    if UIApplication.shared.canOpenURL(tgURL) {
        UIApplication.shared.open(tgURL)
    } else {
        UIApplication.shared.open(webURL)
    }
}
```

### Step 2 — Validate the token before saving

Call the Telegram Bot API to validate the token client-side before writing it to the container.
This gives users instant feedback without a round-trip to the container.

```swift
func validateTelegramToken(_ token: String) async throws -> TelegramBotInfo {
    let url = URL(string: "https://api.telegram.org/bot\(token)/getMe")!
    let (data, response) = try await URLSession.shared.data(from: url)
    guard let http = response as? HTTPURLResponse, http.statusCode == 200 else {
        throw ConnectError.invalidToken
    }
    return try JSONDecoder().decode(TelegramMeResponse.self, from: data).result
}
```

### Step 3 — Write credentials to the container

```swift
struct TelegramCredential: Encodable {
    let botToken: String
    let dmPolicy: String        // "pairing" | "allowlist" | "open"
    let allowFrom: [String]     // Telegram user IDs or usernames
}

func saveTelegramCredential(_ cred: TelegramCredential) async throws {
    var req = URLRequest(url: URL(string: "\(fileAPIURL)/credentials/telegram")!)
    req.httpMethod = "POST"
    req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
    req.setValue("application/json", forHTTPHeaderField: "Content-Type")
    req.httpBody = try JSONEncoder().encode(cred)
    let (_, resp) = try await URLSession.shared.data(for: req)
    guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.saveFailed
    }
}
```

### Step 4 — Reload the Gateway channel config

After writing the credential file, tell the container's Gateway to pick up the new config:

```swift
func reloadChannel(_ channel: String) async throws {
    var req = URLRequest(url: URL(string: "\(fileAPIURL)/rpc/reload-channel")!)
    req.httpMethod = "POST"
    req.setValue("Bearer \(supabaseJWT)", forHTTPHeaderField: "Authorization")
    req.setValue("application/json", forHTTPHeaderField: "Content-Type")
    req.httpBody = try JSONSerialization.data(withJSONObject: ["channel": channel])
    let (_, resp) = try await URLSession.shared.data(for: req)
    guard (resp as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.reloadFailed
    }
}
```

### Step 5 — Confirm connection

Poll `GET /status/channels` until `telegram.connected == true`, then show the user the bot
username (from the `getMe` response in step 2) and an instructional message:

> **Your Telegram bot @YourBotName is connected.**
> Open Telegram, find **@YourBotName**, and send `/start` to pair your account.

## Gateway config written to container

```json5
// /data/credentials/telegram.json  (written by file-api)
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123456:ABCdef...",
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

## Alternative: full e2e flow (no manual token copy)

If you want users to avoid ever seeing the raw token, build an intermediate function:

1. Your backend calls the Telegram Bot API `createBot` flow using a service-level BotFather
   account (not supported by public API today — this requires partnering with Telegram).
2. Alternatively, use a Telegram **Login Widget** (`tg-login`) to authenticate the *user's*
   identity and pre-approve a group bot. This is purely an identity check; the bot token itself
   still needs manual creation by the user once.

For most managed apps, the paste-token approach above is the right trade-off between friction
and implementation complexity.

## DM policy options

| Value | Behavior |
| --- | --- |
| `pairing` | First DM triggers a pairing code; user must confirm |
| `allowlist` | Only users in `allowFrom` can message the bot |
| `open` | Any Telegram user can message the bot |

Set `dmPolicy` to `pairing` as the default for managed deployments. Users can upgrade to
`allowlist` after they've verified their Telegram account.

## Related docs

- [Telegram channel](/channels/telegram) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/dean/ios-quickconnect)
