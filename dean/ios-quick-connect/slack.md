---
title: "Slack – iOS Quick Connect"
summary: "How to connect a Slack workspace to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Slack quick-connect screen in the iOS app
---

# Slack – iOS Quick Connect

**Auth method:** OAuth in-app (recommended) or token paste  
**Complexity:** Medium

Slack supports two quick-connect paths from the iOS app:

1. **OAuth "Add to Slack" button** — The user taps a button, authorizes in Slack's web UI (via `ASWebAuthenticationSession`), and the app captures the tokens automatically. This is the smoothest UX and requires a hosted OAuth callback endpoint.
2. **Token paste** — The user manually creates a Slack app, copies the App Token (`xapp-`) and Bot Token (`xoxb-`), and pastes them into text fields. Suitable for power users or when you do not want to host an OAuth callback.

---

## Option A: OAuth "Add to Slack" flow (recommended)

### User flow

```
1. User opens Settings → Channels → Slack
2. Taps "Add to Slack" button
3. ASWebAuthenticationSession opens Slack's OAuth page
4. User selects their workspace and authorizes the app
5. Slack redirects back to the app's callback URL with a code
6. Backend exchanges the code for bot token + app token
7. Tokens are forwarded to the user's container
8. App shows success with workspace name
```

### iOS implementation

```swift
import AuthenticationServices

struct SlackOAuthView: View {
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Button("Add to Slack") {
            Task { await startOAuth() }
        }
        .buttonStyle(.borderedProminent)
        .navigationTitle("Connect Slack")
    }

    private func startOAuth() async {
        let clientId = AppConfig.slackClientId
        let redirectURI = "https://files.spark.ooo/oauth/slack/callback"
        let scopes = "app_mentions:read,channels:history,chat:write,groups:history,im:history,mpim:history"
        
        var components = URLComponents(string: "https://slack.com/oauth/v2/authorize")!
        components.queryItems = [
            .init(name: "client_id", value: clientId),
            .init(name: "scope", value: scopes),
            .init(name: "redirect_uri", value: redirectURI),
            .init(name: "state", value: generateStateToken()),
        ]

        let session = ASWebAuthenticationSession(
            url: components.url!,
            callbackURLScheme: "openclaw"
        ) { callbackURL, error in
            guard let url = callbackURL else { return }
            let code = URLComponents(url: url, resolvingAgainstBaseURL: false)?
                .queryItems?.first(where: { $0.name == "code" })?.value
            Task { await self.exchangeCode(code!) }
        }
        session.presentationContextProvider = self
        session.prefersEphemeralWebBrowserSession = true
        session.start()
    }

    private func exchangeCode(_ code: String) async {
        // The file-api OAuth endpoint handles the Slack code exchange
        // and patches the container config automatically.
        try? await ChannelConfigService.shared.completeOAuth(
            channel: "slack",
            code: code
        )
        status = .connected
    }
}
```

### Backend: OAuth callback endpoint (file-api)

The `files.spark.ooo` file-api pod must implement:

```
GET /oauth/slack/callback?code=<CODE>&state=<STATE>

1. Validates the state token against the Supabase session
2. POSTs to https://slack.com/api/oauth.v2.access with code + client secret
3. Extracts bot_token (xoxb-) and app_token (requires separate socket-mode token)
4. PATCHes the user's container config with the tokens
5. Redirects to the app via deep link: openclaw://oauth/slack/success
```

---

## Option B: Token paste

### Slack app setup (user does this once)

| Step | Where |
|------|-------|
| Create a Slack app | [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From Scratch** |
| Enable Socket Mode | App settings → **Socket Mode** → toggle on |
| Create App Token | App settings → **Basic Information** → App-Level Tokens → **Generate Token** with `connections:write` scope → copy `xapp-...` token |
| Install to workspace | App settings → **Install App** → **Install to Workspace** → copy `xoxb-...` Bot Token |

### UI (SwiftUI sketch)

```swift
struct SlackTokenPasteView: View {
    @State private var appToken = ""   // xapp-...
    @State private var botToken = ""   // xoxb-...
    @State private var status: ConnectionStatus = .idle

    var body: some View {
        Form {
            Section(header: Text("App Token (xapp-)")) {
                SecureField("xapp-1-...", text: $appToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }
            Section(header: Text("Bot Token (xoxb-)")) {
                SecureField("xoxb-...", text: $botToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
            }
            Section {
                Button("Connect") {
                    Task { await connect() }
                }
                .disabled(appToken.isEmpty || botToken.isEmpty)
            }
        }
        .navigationTitle("Connect Slack")
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "slack",
                config: [
                    "mode": "socket",
                    "appToken": appToken,
                    "botToken": botToken,
                    "enabled": true,
                ]
            )
            status = .connected
        } catch {
            status = .error(error.localizedDescription)
        }
    }
}
```

---

## Container config applied

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",          // Socket Mode (no public URL needed)
      appToken: "xapp-...",    // connections:write
      botToken: "xoxb-...",    // Bot User OAuth Token
      dmPolicy: "pairing",
    },
  },
}
```

---

## Security notes

- Slack bot tokens grant broad workspace access. Scope them as narrowly as possible.
- For the OAuth flow, the `state` parameter must be a cryptographically random token tied to the user's Supabase session to prevent CSRF.
- The file-api callback endpoint must verify the `state` before exchanging the code.
- Never log or display the full token in the iOS UI; mask all but the last 6 characters.
