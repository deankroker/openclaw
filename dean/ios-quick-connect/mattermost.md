---
title: "Mattermost – iOS Quick Connect"
summary: "How to connect a Mattermost workspace to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Mattermost quick-connect screen in the iOS app
---

# Mattermost – iOS Quick Connect

**Auth method:** Token paste + server URL  
**Complexity:** Low

Mattermost is a self-hosted team messaging platform. OpenClaw connects to it using a **bot account access token**. The user provides the Mattermost server URL and the bot token, and the container connects via the Mattermost WebSocket API.

---

## User flow

```
1. User opens Settings → Channels → Mattermost
2. iOS app shows:
   - Text field: "Mattermost Server URL" (e.g., https://chat.example.com)
   - SecureField: "Bot Access Token"
   - (Optional) Text field: "Team Name" (to scope the bot to a specific team)
3. User creates a bot account in Mattermost and copies its access token
4. Pastes credentials and taps "Connect"
5. App verifies the token against the Mattermost API
6. Container config is patched; gateway connects via WebSocket
7. Success screen shows the bot's username and server URL
```

---

## iOS implementation

### 1. UI (SwiftUI sketch)

```swift
struct MattermostConnectView: View {
    @State private var serverURL = ""
    @State private var accessToken = ""
    @State private var teamName = ""
    @State private var status: ConnectionStatus = .idle
    @State private var verifiedUsername: String? = nil

    var body: some View {
        Form {
            Section(header: Text("Mattermost Server URL")) {
                TextField("https://chat.example.com", text: $serverURL)
                    .textContentType(.URL)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }

            Section(header: Text("Bot Access Token")) {
                SecureField("xxxxxxxxxxxxxxxxxxxxxxxxxx", text: $accessToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
                if let username = verifiedUsername {
                    Label("@\(username)", systemImage: "checkmark.circle.fill")
                        .foregroundStyle(.green)
                        .font(.footnote)
                }
            }

            Section(header: Text("Team (optional)")) {
                TextField("my-team", text: $teamName)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }

            Section {
                Button("Verify Token") { Task { await verify() } }
                Button("Connect") { Task { await connect() } }
                    .disabled(serverURL.isEmpty || accessToken.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect Mattermost")
    }

    private func verify() async {
        guard let url = URL(string: serverURL.appending("/api/v4/users/me")) else { return }
        var request = URLRequest(url: url)
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")
        guard let (data, _) = try? await URLSession.shared.data(for: request) else {
            verifiedUsername = nil
            return
        }
        struct MmUser: Decodable { let username: String }
        verifiedUsername = try? JSONDecoder().decode(MmUser.self, from: data).username
    }

    private func connect() async {
        status = .connecting
        var config: [String: Any] = [
            "enabled": true,
            "serverUrl": serverURL,
            "accessToken": accessToken,
            "dmPolicy": "pairing",
        ]
        if !teamName.isEmpty { config["teamName"] = teamName }
        do {
            try await ChannelConfigService.shared.patch(channel: "mattermost", config: config)
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

### 2. Bot account creation guide (in-app)

The iOS app should include a brief guide for users who have not yet created a bot:

1. Log into your Mattermost server as an admin.
2. Go to **Main Menu → Integrations → Bot Accounts** → **Add Bot Account**.
3. Set a username (e.g., `openclaw-bot`), role, and description.
4. Click **Create Bot Account** and copy the access token.
5. If you are not a server admin, ask your admin to create a bot account for you or generate a Personal Access Token (if enabled): **Account Settings → Security → Personal Access Tokens**.

---

## Container config applied

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      serverUrl: "https://chat.example.com",
      accessToken: "<BOT_ACCESS_TOKEN>",
      teamName: "my-team",         // optional
      dmPolicy: "pairing",
      allowFrom: [],
    },
  },
}
```

---

## Plugin requirement

Mattermost ships as a plugin (`@openclaw/mattermost`) and must be installed in the container. The iOS app should check plugin status and offer installation:

```http
POST /gateway/plugins/install
Authorization: Bearer <supabase-jwt>
Content-Type: application/json

{ "package": "@openclaw/mattermost" }
```

---

## Security notes

- Mattermost bot access tokens are equivalent to bot account credentials. Scope them to a bot account (not an admin user) to limit the blast radius if a token is leaked.
- Personal Access Tokens from admin users should be avoided; use a dedicated bot account instead.
- Store the token only on the container side.
- Verify that the Mattermost server's SSL certificate is valid before connecting; reject self-signed certificates in production or require the user to explicitly trust them.
