---
title: "Matrix – iOS Quick Connect"
summary: "How to connect a Matrix account to a user's OpenClaw container from the iOS app"
read_when:
  - Implementing the Matrix quick-connect screen in the iOS app
---

# Matrix – iOS Quick Connect

**Auth method:** Token paste or SSO via homeserver  
**Complexity:** Medium

Matrix is an open, federated protocol. Users need a Matrix account on any homeserver and an **access token** for that account. The iOS app can collect the token via:

1. **Token paste** — User generates an access token from their homeserver's web client or via the login API and pastes it.
2. **Matrix SSO** — iOS app opens the homeserver's SSO login page in `ASWebAuthenticationSession` and captures the access token on success.

---

## Option A: Token paste

### User flow

```
1. User opens Settings → Channels → Matrix
2. iOS app shows:
   - Text field for "Homeserver URL" (e.g., https://matrix.org)
   - SecureField for "Access Token"
3. User generates their access token:
   - Element/web client: Settings → Help & About → Access Token
   - Or via curl (see below)
4. User pastes token and taps "Connect"
5. App verifies the token against the homeserver's /whoami endpoint
6. Container config is patched and gateway reconnects
```

### Generating a token (in-app guide)

The iOS app can display a one-tap guide with a `curl` command the user can run in Terminal, or a link to their homeserver's web UI:

```
curl --request POST \
  --url https://<homeserver>/_matrix/client/v3/login \
  --header 'Content-Type: application/json' \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@bot:example.org" },
    "password": "your-password"
  }'
```

### UI (SwiftUI sketch)

```swift
struct MatrixTokenPasteView: View {
    @State private var homeserverURL = "https://matrix.org"
    @State private var accessToken = ""
    @State private var status: ConnectionStatus = .idle
    @State private var verifiedUserId: String? = nil

    var body: some View {
        Form {
            Section(header: Text("Homeserver URL")) {
                TextField("https://matrix.org", text: $homeserverURL)
                    .textContentType(.URL)
                    .autocorrectionDisabled()
                    .autocapitalization(.none)
            }
            Section(header: Text("Access Token")) {
                SecureField("syt_...", text: $accessToken)
                    .textContentType(.password)
                    .autocorrectionDisabled()
                if let userId = verifiedUserId {
                    Label(userId, systemImage: "checkmark.circle.fill")
                        .foregroundStyle(.green)
                        .font(.footnote)
                }
            }
            Section {
                Button("Verify Token") { Task { await verify() } }
                Button("Connect") { Task { await connect() } }
                    .disabled(accessToken.isEmpty || status == .connecting)
            }
        }
        .navigationTitle("Connect Matrix")
    }

    private func verify() async {
        guard let url = URL(string: homeserverURL + "/_matrix/client/v3/account/whoami") else { return }
        var request = URLRequest(url: url)
        request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")
        guard let (data, _) = try? await URLSession.shared.data(for: request) else {
            verifiedUserId = nil
            return
        }
        struct WhoAmI: Decodable { let user_id: String }
        if let me = try? JSONDecoder().decode(WhoAmI.self, from: data) {
            verifiedUserId = me.user_id
        }
    }

    private func connect() async {
        status = .connecting
        do {
            try await ChannelConfigService.shared.patch(
                channel: "matrix",
                config: [
                    "enabled": true,
                    "homeserverUrl": homeserverURL,
                    "accessToken": accessToken,
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

## Option B: Matrix SSO (OAuth-like)

Most homeservers support SSO login. The iOS app can use `ASWebAuthenticationSession` to log the bot account in and capture its access token.

```swift
func signInViaMatrixSSO(homeserver: String) async throws -> String {
    let ssoURL = "\(homeserver)/_matrix/client/v3/login/sso/redirect"
    var components = URLComponents(string: ssoURL)!
    components.queryItems = [
        .init(name: "redirectUrl", value: "openclaw://matrix/callback"),
    ]

    return try await withCheckedThrowingContinuation { continuation in
        let session = ASWebAuthenticationSession(
            url: components.url!,
            callbackURLScheme: "openclaw"
        ) { callbackURL, error in
            guard let url = callbackURL,
                  let loginToken = URLComponents(url: url, resolvingAgainstBaseURL: false)?
                    .queryItems?.first(where: { $0.name == "loginToken" })?.value else {
                continuation.resume(throwing: ChannelError.oauthFailed)
                return
            }
            // Exchange loginToken for access token via /_matrix/client/v3/login
            Task {
                do {
                    let token = try await MatrixService.exchangeLoginToken(
                        homeserver: homeserver,
                        loginToken: loginToken
                    )
                    continuation.resume(returning: token)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
        session.prefersEphemeralWebBrowserSession = true
        session.start()
    }
}
```

---

## Container config applied

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserverUrl: "https://matrix.org",
      accessToken: "<ACCESS_TOKEN>",
      dmPolicy: "pairing",
    },
  },
}
```

---

## Plugin requirement

Matrix ships as a plugin (`@openclaw/matrix`) that must be installed in the container before the channel can be enabled. The iOS app setup screen should check whether the plugin is installed and offer to install it:

```http
GET /gateway/plugins
→ [{ "name": "@openclaw/matrix", "installed": false }]

POST /gateway/plugins/install
{ "package": "@openclaw/matrix" }
```

If the plugin is not yet installed, show an "Install Matrix plugin" step before the credential form.

---

## E2EE support

Matrix End-to-End Encryption (E2EE) requires additional crypto key storage. The container handles key verification automatically when E2EE is enabled in config. The iOS app does not need to manage crypto keys directly.

---

## Security notes

- Matrix access tokens have no built-in expiry (they stay valid until revoked). Treat them like passwords.
- Use a dedicated Matrix account for the bot; do not use a personal account.
- If the homeserver supports device verification, verify the bot's device in your Matrix client to enable E2EE.
- Revoke the access token from the homeserver's web UI if the integration is disconnected.
