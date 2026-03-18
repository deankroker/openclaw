---
title: "Discord – iOS Quick Connect"
summary: "How to connect a Discord bot to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Discord quick-connect screen in the iOS app
---

# Discord – iOS Quick Connect

**Auth method:** Token paste  
**Complexity:** Low

Discord uses a long-lived **bot token** that the user copies from the Discord Developer Portal and pastes into the iOS app. No OAuth round-trip or QR scan is required.

---

## User flow

```
1. User opens Settings → Channels → Discord
2. iOS app shows a "Connect Discord" screen with:
   - Link to discord.com/developers/applications (opens in SFSafariViewController)
   - Secure text field for "Bot Token"
   - Optional: text field for "Guild ID" (to scope the bot to one server)
3. User taps "Connect" → token is validated then saved to the container
4. Success screen shows bot username and connection status
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct DiscordConnectView: View {
    @State private var botToken = ""
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section(header: Text("Bot Token")) {
                SecureField("xxxxxxxxxxxxxxxxxxx.xxxxxx.xxxxxxxxxxxxxxxxxxxxxxxxxxxx", text: $botToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }
            Section {
                Button("Connect") {
                    Task { await connect() }
                }
                .disabled(botToken.isEmpty || status == .connecting)
            }
            if case .error(let msg) = status {
                Text(msg).foregroundStyle(.red)
            }
        }
        .navigationTitle("Connect Discord")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "discord",
                config: ["botToken": botToken, "enabled": true]
            )
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

### 2. Token validation (optional pre-check)

Before patching the container config, the iOS app can call the Discord API directly to verify the token and display the bot's username:

```swift
func validateDiscordToken(_ token: String) async throws -> String {
    var request = URLRequest(url: URL(string: "https://discord.com/api/v10/users/@me")!)
    request.setValue("Bot \(token)", forHTTPHeaderField: "Authorization")
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else {
        throw ChannelError.invalidToken
    }
    let user = try JSONDecoder().decode(DiscordUser.self, from: data)
    return user.username
}
```

---

## Container config applied

```json5
{
  channels: {
    discord: {
      enabled: true,
      botToken: "<BOT_TOKEN>",
      // Optional: scope to a specific guild
      guilds: {
        "<GUILD_ID>": { allowAllChannels: true }
      }
    }
  }
}
```

Sent via the file-api config-patch endpoint:

```http
PATCH /config/channels
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{
  "channel": "discord",
  "config": {
    "enabled": true,
    "botToken": "<BOT_TOKEN>"
  }
}
```

---

## Prerequisites the user must complete first

| Step | Where |
|------|-------|
| Create a Discord application | [discord.com/developers/applications](https://discord.com/developers/applications) → **New Application** |
| Add a bot to the application | Application → **Bot** → **Add Bot** |
| Enable Message Content Intent | Bot page → **Privileged Gateway Intents** → enable **Message Content Intent** |
| Copy bot token | Bot page → **Reset Token** → copy |
| Invite the bot to a server | OAuth2 → URL Generator → scopes: `bot`, `applications.commands`; permissions: View Channels, Send Messages, Read Message History |

The iOS app can deep-link to each step using `SFSafariViewController` or include a mini-guide with screenshots.

---

## Quick auth: "Connect with Discord" OAuth2 (alternative)

If you want a zero-paste experience, you can implement an OAuth2 flow where the user authorizes your app to create a bot on their behalf. This requires you to host an OAuth2 callback endpoint.

```
iOS App  →  ASWebAuthenticationSession
         →  https://discord.com/oauth2/authorize
              ?client_id=<YOUR_APP_CLIENT_ID>
              &scope=bot+applications.commands
              &permissions=<FLAGS>
              &redirect_uri=<YOUR_CALLBACK>
         →  User grants access
         →  Callback receives guild_id + bot_token (if using bot authorization flow)
         →  App patches container config
```

> This approach requires your OAuth2 redirect handler to exchange the code for a bot token and forward it to the container. It is more complex but provides a smoother user experience for non-technical users.

---

## Security notes

- Store the bot token only on the container (never in the iOS app's local storage or Keychain after the initial POST).
- The file-api endpoint must validate the Supabase JWT before writing to config.
- Tokens displayed in the settings screen should be masked (show only last 4 characters).
