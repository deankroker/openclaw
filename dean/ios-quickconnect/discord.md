---
summary: "Quick connect Discord from the managed iOS app using a bot token or OAuth install flow"
read_when:
  - Implementing Discord channel setup in the iOS app
  - Understanding the bot token or OAuth install pattern for Discord
title: "Discord — iOS Quick Connect"
sidebarTitle: "Discord"
---

# Discord — iOS Quick Connect

**Pattern: paste bot token (simple) or OAuth install flow (full e2e)**

Discord supports two approaches. The **token path** lets power users paste their bot token
directly. The **OAuth install path** lets the app automate bot creation and server installation
through Discord's OAuth2 flow, removing any copy-paste.

## Option A — Paste bot token (simple)

Best for users who already have a Discord bot or who are comfortable with the developer portal.

### User journey

```
User                iOS App              file-api               OpenClaw Pod
 │                     │                    │                        │
 │  Tap "Add Discord"  │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  Show token input  │                        │
 │◄────────────────────│                    │                        │
 │  Paste bot token    │                    │                        │
 │  + guild/user ID    │                    │                        │
 │────────────────────►│                    │                        │
 │                     │  POST /credentials/discord                  │
 │                     │────────────────────►                        │
 │                     │                    │───────────────────────►│
 │                     │                    │                        │  Reload config
 │                     │  ✓ Connected       │                        │◄──────────────
 │◄────────────────────│                    │                        │
```

### What the iOS app collects

| Field | UI label | Required | Notes |
| --- | --- | --- | --- |
| `botToken` | Bot token | Yes | From Discord Developer Portal → Bot |
| `guildId` | Server (guild) ID | Yes | Right-click server name → Copy Server ID |
| `allowedUserId` | Your Discord user ID | Yes | Right-click your name → Copy User ID |

### Token validation

```swift
func validateDiscordToken(_ token: String) async throws -> DiscordBotUser {
    var req = URLRequest(url: URL(string: "https://discord.com/api/v10/users/@me")!)
    req.setValue("Bot \(token)", forHTTPHeaderField: "Authorization")
    let (data, response) = try await URLSession.shared.data(for: req)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ConnectError.invalidToken
    }
    return try JSONDecoder().decode(DiscordBotUser.self, from: data)
}
```

---

## Option B — OAuth install flow (full e2e)

Best for a polished onboarding where users should not see raw tokens. The app drives the full
Discord OAuth2 flow and stores credentials automatically.

### Pre-requisites (one-time operator setup)

1. Create a Discord application in the [Developer Portal](https://discord.com/developers/applications).
2. Under **OAuth2**, add the redirect URI:
   `https://openclaw.ai/auth/callback/discord`
3. Note your **Client ID** and **Client Secret** (kept server-side; never in the iOS app).

### User journey

```
User                iOS App              OpenClaw Backend        Discord
 │                     │                    │                       │
 │  Tap "Add Discord"  │                    │                       │
 │────────────────────►│                    │                       │
 │                     │  GET /auth/discord/start                   │
 │                     │───────────────────►│                       │
 │                     │                    │  Build OAuth URL      │
 │                     │◄───────────────────│                       │
 │  Open Safari/       │                    │                       │
 │  ASWebAuthSession   │────────────────────────────────────────────►
 │                     │                    │                       │  User logs in
 │                     │                    │                       │  and selects
 │                     │                    │                       │  server to add bot
 │                     │◄────────────────────────────────────────────
 │                     │  Universal link: openclaw.ai/auth/callback/discord?code=…
 │                     │                    │                       │
 │                     │  POST /auth/discord/exchange { code }      │
 │                     │───────────────────►│                       │
 │                     │                    │  Exchange code → tokens
 │                     │                    │  Write credentials    │
 │                     │                    │  to container volume  │
 │                     │◄───────────────────│                       │
 │                     │  POST /rpc/reload  │                       │
 │                     │  ✓ Connected       │                       │
 │◄────────────────────│                    │                       │
```

### iOS implementation

```swift
import AuthenticationServices

class DiscordConnectViewController: UIViewController, ASWebAuthenticationPresentationContextProviding {

    func startOAuthFlow() {
        // 1. Build the Discord authorization URL
        var components = URLComponents(string: "https://discord.com/oauth2/authorize")!
        components.queryItems = [
            URLQueryItem(name: "client_id", value: discordClientID),
            URLQueryItem(name: "redirect_uri", value: "https://openclaw.ai/auth/callback/discord"),
            URLQueryItem(name: "response_type", value: "code"),
            URLQueryItem(name: "scope", value: "bot applications.commands"),
            URLQueryItem(name: "permissions", value: "274878032960"),  // Read + Send
            URLQueryItem(name: "state", value: generateCSRFState()),
        ]

        // 2. Open with ASWebAuthenticationSession to handle the callback
        let session = ASWebAuthenticationSession(
            url: components.url!,
            callbackURLScheme: "https"
        ) { callbackURL, error in
            guard let url = callbackURL, error == nil else { return }
            self.handleCallback(url: url)
        }
        session.prefersEphemeralWebBrowserSession = false
        session.presentationContextProvider = self
        session.start()
    }

    func handleCallback(url: URL) {
        guard let code = URLComponents(url: url, resolvingAgainstBaseURL: false)?
            .queryItems?.first(where: { $0.name == "code" })?.value else { return }
        Task { try await exchangeCodeAndSave(code: code) }
    }

    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        return view.window!
    }
}
```

### OAuth scopes and permissions

```
Scopes:     bot   applications.commands
Bot permissions (bitmask: 274878032960):
  - View Channels
  - Send Messages
  - Read Message History
  - Embed Links
  - Attach Files
  - Add Reactions
  - Use Slash Commands
```

### Backend token exchange

Your server-side function (not in the iOS app) exchanges the authorization code and stores the
bot token in the container volume:

```typescript
// Pseudocode — runs on your backend, not in the iOS app
async function exchangeDiscordCode(code: string, userId: string) {
  const tokens = await discord.oauth.exchange(code);
  // tokens.bot_token is the credential the Gateway needs
  await fileApi.writeCredential(userId, "discord", {
    botToken: tokens.bot_token,
    guildId: tokens.guild_id,
    dmPolicy: "pairing",
    allowedUserIds: [],
  });
  await containerRPC.reloadChannel(userId, "discord");
}
```

---

## Gateway config written to container

```json5
// /data/credentials/discord.json  (written by file-api or backend)
{
  channels: {
    discord: {
      enabled: true,
      token: "Bot MTExxx...",
      guilds: {
        "1234567890": {
          enabled: true,
          dmPolicy: "pairing",
          allowedUserIds: ["9876543210"],
        },
      },
    },
  },
}
```

## Choosing between Option A and Option B

| | Option A (paste token) | Option B (OAuth) |
| --- | --- | --- |
| User effort | Medium (portal + copy) | Low (one click) |
| App complexity | Low | High |
| Token visible to user | Yes | No |
| Requires backend | No | Yes |
| Best for | Power users, self-hosted | Consumer managed app |

For the fully managed iOS app described in this architecture, **Option B** provides the
smoothest experience. Option A remains useful as a fallback for users who already have bots.

## Related docs

- [Discord channel](/channels/discord) — full channel reference
- [Pairing](/channels/pairing) — DM pairing flow
- [iOS Quick Connect overview](/dean/ios-quickconnect)
