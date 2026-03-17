---
title: "Twitch – iOS Quick Connect"
summary: "How to connect a Twitch bot account to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Twitch quick-connect screen in the iOS app
---

# Twitch – iOS Quick Connect

**Auth method:** OAuth in-app ("Connect with Twitch")  
**Complexity:** Low

Twitch uses OAuth2 for bot authentication. The iOS app opens Twitch's OAuth authorization page in `ASWebAuthenticationSession`, and the user grants access to a bot account. The app captures the resulting OAuth token automatically.

---

## User flow

```
1. User opens Settings → Channels → Twitch
2. iOS app shows:
   - "Connect with Twitch" button
   - Text field for "Channel to join" (the Twitch streamer's channel, e.g., "vevisk")
3. User taps "Connect with Twitch"
4. ASWebAuthenticationSession opens Twitch OAuth page
5. User authorizes the bot account (chat:read, chat:write scopes)
6. App captures the OAuth token and client ID
7. Container config is patched
8. Success screen shows the bot's Twitch username and target channel
```

---

## iOS implementation

### 1. OAuth setup

Register an application at [dev.twitch.tv/console](https://dev.twitch.tv/console) with:
- OAuth Redirect URLs: `openclaw://twitch/callback`
- Category: Chat Bot

```swift
struct TwitchConnectView: View {
    @State private var channelName = ""     // Streamer's channel to join
    @State private var status: ConnectionStatus = .idle
    @State private var connectedAs: String? = nil

    var body: some View {
        Form {
            Section(header: Text("Channel to join")) {
                TextField("vevisk", text: $channelName)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }
            Section {
                Button("Connect with Twitch") {
                    Task { await startOAuth() }
                }
                .disabled(channelName.isEmpty || status == .connecting)
                .buttonStyle(.borderedProminent)

                if let username = connectedAs {
                    Label("Connected as @\(username)", systemImage: "checkmark.circle.fill")
                        .foregroundStyle(.green)
                }
            }
        }
        .navigationTitle("Connect Twitch")
    }

    private func startOAuth() async {
        let clientId = AppConfig.twitchClientId
        let redirectURI = "openclaw://twitch/callback"
        let scopes = "chat:read chat:write"

        var components = URLComponents(string: "https://id.twitch.tv/oauth2/authorize")!
        components.queryItems = [
            .init(name: "client_id", value: clientId),
            .init(name: "redirect_uri", value: redirectURI),
            .init(name: "response_type", value: "token"),   // Implicit flow for chat bots
            .init(name: "scope", value: scopes),
            .init(name: "state", value: generateStateToken()),
            .init(name: "force_verify", value: "true"),
        ]

        let session = ASWebAuthenticationSession(
            url: components.url!,
            callbackURLScheme: "openclaw"
        ) { callbackURL, error in
            guard let url = callbackURL else { return }
            // Twitch puts the token in the URL fragment for implicit flow
            let fragment = url.fragment ?? ""
            let params = parseQueryString(fragment)
            guard let accessToken = params["access_token"] else { return }
            Task { await self.finalize(accessToken: accessToken) }
        }
        session.prefersEphemeralWebBrowserSession = true
        session.start()
    }

    private func finalize(accessToken: String) async {
        // Validate and get bot username
        let username = try? await validateTwitchToken(accessToken)
        connectedAs = username

        try? await ChannelConfigService.shared.patch(
            channel: "twitch",
            config: [
                "enabled": true,
                "accessToken": "oauth:\(accessToken)",
                "clientId": AppConfig.twitchClientId,
                "channel": channelName,
                "username": username ?? "",
            ]
        )
        status = .connected
    }
}
```

### 2. Token validation

```swift
func validateTwitchToken(_ token: String) async throws -> String {
    var request = URLRequest(url: URL(string: "https://id.twitch.tv/oauth2/validate")!)
    request.setValue("OAuth \(token)", forHTTPHeaderField: "Authorization")
    let (data, _) = try await URLSession.shared.data(for: request)
    struct ValidationResponse: Decodable { let login: String; let user_id: String }
    let resp = try JSONDecoder().decode(ValidationResponse.self, from: data)
    return resp.login
}
```

---

## Container config applied

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw_bot",           // Bot's Twitch account
      accessToken: "oauth:xxxxxx...",      // OAuth Access Token
      clientId: "xxxxxxxxxxxxxxxxxx",      // App Client ID
      channel: "vevisk",                   // Streamer's channel to join
      requireMention: true,                // Only respond when @mentioned
      allowFrom: ["123456789"],            // Optional: streamer's user ID only
    },
  },
}
```

---

## Plugin requirement

Twitch ships as a plugin (`@openclaw/twitch`) and must be installed in the container before enabling the channel. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/twitch" }
```

---

## Access control

Twitch chat is public. The iOS app settings screen should include an **allowFrom** field so the user can restrict who can interact with the bot. Recommended defaults:

- `requireMention: true` — Bot only responds when explicitly mentioned.
- `allowFrom: [<streamer-user-id>]` — Limit to the channel owner.

The iOS app can look up the streamer's Twitch user ID from their username:

```swift
func getTwitchUserId(username: String, clientId: String, token: String) async throws -> String {
    var request = URLRequest(url: URL(string: "https://api.twitch.tv/helix/users?login=\(username)")!)
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    request.setValue(clientId, forHTTPHeaderField: "Client-Id")
    let (data, _) = try await URLSession.shared.data(for: request)
    struct UsersResponse: Decodable {
        struct User: Decodable { let id: String }
        let data: [User]
    }
    return try JSONDecoder().decode(UsersResponse.self, from: data).data.first.map { $0.id } ?? { throw ChannelError.custom("Twitch user '\(username)' not found") }()
}
```

---

## Security notes

- Twitch OAuth tokens for chat bots require only `chat:read` and `chat:write` scopes; do not request broader permissions.
- Always set `requireMention: true` in the config to prevent the bot from responding to every chat message.
- Store the OAuth token only on the container side; do not persist it in the iOS Keychain after the initial POST.
- Twitch OAuth tokens expire after approximately 60 days. The iOS app settings screen should surface a "Re-authorize" button that runs the OAuth flow again.
