---
summary: "Implement Anthropic Claude quick connect in the OpenClaw iOS app (API key, OAuth, Claude app deep link)"
read_when:
  - Adding Anthropic/Claude authentication to the iOS app
  - Implementing setup-token or API-key flows for Claude
title: "Anthropic — iOS Quick Connect"
---

# Anthropic (Claude) — iOS Quick Connect

Anthropic provides two credential types that OpenClaw supports: a plain **API key**
(`sk-ant-api03-...`) and an OAuth **setup-token** minted by the Claude Code CLI. For a
managed iOS app with a cloud-hosted OpenClaw container, an API key is the most reliable path.

## Option A: API key paste

The user creates a key at [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)
and pastes it into the app.

### iOS implementation

```swift
struct AnthropicKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your Anthropic API key")
                .font(.headline)
            SecureField("sk-ant-api03-...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect Claude") {
                Task { await connect() }
            }
            .disabled(apiKey.isEmpty || !apiKey.hasPrefix("sk-ant-"))
            Link("Get an API key →",
                 destination: URL(string: "https://console.anthropic.com/settings/keys")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(provider: "anthropic", key: apiKey)
    }
}
```

**Key validation**: Anthropic API keys start with `sk-ant-`. Add a prefix check before
sending to avoid unnecessary API calls.

**Storage**: Call the file-api `PUT /config` endpoint with the user's Supabase JWT.
Do not cache the key locally — the container holds it.

### Config written to the container

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-api03-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

---

## Option B: OAuth PKCE browser flow (Claude subscription)

Anthropic's subscription OAuth is issued through the **Claude Code CLI** (`claude setup-token`),
not a standard web OAuth server. There is no public authorization endpoint that a third-party
app can redirect users to directly.

If Anthropic publishes an official iOS SDK or a public OAuth authorization endpoint in the
future, the implementation would follow the standard PKCE pattern using
`ASWebAuthenticationSession` (see the [Quick Connect overview](/ios)).

**Current recommendation**: Use Option A (API key) or Option D (setup-token relay) for
subscription users until Anthropic provides a first-party mobile OAuth flow.

---

## Option C: Claude iOS app deep link

The **Claude iOS app** (bundle ID `com.anthropic.claudeai`) does not currently expose a
documented OAuth deep link for third-party token delegation. You can still open it for user
reference:

```swift
let claudeURL = URL(string: "claude://")!
if UIApplication.shared.canOpenURL(claudeURL) {
    UIApplication.shared.open(claudeURL)
} else {
    // Open App Store listing
    UIApplication.shared.open(
        URL(string: "https://apps.apple.com/app/claude-by-anthropic/id6473753684")!
    )
}
```

Add `claude` to `LSApplicationQueriesSchemes` in `Info.plist` to enable the `canOpenURL` check.

Monitor the [Anthropic developer changelog](https://docs.anthropic.com/en/release-notes/overview)
for any first-party mobile auth delegation feature.

---

## Option D: OpenClaw URL scheme / setup-token relay

For power users who generated a setup-token on a desktop via `claude setup-token`, the
OpenClaw gateway can expose a one-time token transfer endpoint. The iOS app receives the
token via a universal link:

```
https://app.openclaw.ai/connect?provider=anthropic&token=<base64-encoded-setup-token>
```

### Receiving the link in SwiftUI

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.host == "app.openclaw.ai",
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "anthropic",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let token = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "anthropic", key: token) }
}
```

**Security**: Universal Links require an associated domain entitlement and a hosted
`apple-app-site-association` file at `https://app.openclaw.ai/.well-known/apple-app-site-association`.
This prevents URL scheme hijacking.

---

## Credential flow to the container

```
User pastes key / token arrives via deep link
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "anthropic", "apiKey": "sk-ant-..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  ANTHROPIC_API_KEY available
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` from Anthropic | Wrong or expired API key | Re-enter a fresh key from the Anthropic Console |
| Key accepted but models return errors | Wrong region, rate limit, or free-tier exhausted | Check Anthropic Console usage; upgrade plan if needed |
| Setup-token stops working | Anthropic revoked subscription token outside Claude Code | Switch to an API key |

## Related docs

- [Anthropic provider (OpenClaw)](/providers/anthropic)
- [iOS Quick Connect overview](/ios)
