---
summary: "Implement OpenRouter quick connect in the OpenClaw iOS app (API key paste)"
read_when:
  - Adding OpenRouter authentication to the iOS app
  - Implementing OpenRouter API key entry on iOS
title: "OpenRouter — iOS Quick Connect"
---

# OpenRouter — iOS Quick Connect

OpenRouter provides a **unified API key** that routes requests to many underlying LLM
providers (Anthropic, OpenAI, Google, Mistral, and more). A single OpenRouter key gives
users access to the full model catalog without managing per-provider credentials. This
makes OpenRouter an excellent default for onboarding flows where simplicity matters more
than per-provider optimization.

## Option A: API key paste

The user creates a key at [openrouter.ai/keys](https://openrouter.ai/keys) and pastes
it into the app.

### iOS implementation

```swift
struct OpenRouterKeyView: View {
    @State private var apiKey = ""
    @EnvironmentObject var session: UserSession

    var body: some View {
        VStack(spacing: 16) {
            Text("Enter your OpenRouter API key")
                .font(.headline)
            SecureField("sk-or-...", text: $apiKey)
                .textContentType(.password)
                .autocorrectionDisabled()
                .textInputAutocapitalization(.never)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(8)
            Button("Connect OpenRouter") {
                Task { await connect() }
            }
            .disabled(apiKey.isEmpty || !apiKey.hasPrefix("sk-or-"))
            Link("Get an API key →",
                 destination: URL(string: "https://openrouter.ai/keys")!)
                .font(.footnote)
        }
        .padding()
    }

    private func connect() async {
        await session.saveProviderKey(provider: "openrouter", key: apiKey)
    }
}
```

**Key validation**: OpenRouter keys start with `sk-or-`.

### Config written to the container

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      // Use any model from the OpenRouter catalog:
      model: { primary: "openrouter/anthropic/claude-opus-4-6" },
    },
  },
}
```

Model refs are `openrouter/<provider>/<model>`. See the
[OpenRouter model catalog](https://openrouter.ai/models) for available options.

---

## Option B: OpenRouter OAuth PKCE browser flow

OpenRouter supports OAuth 2.0 with PKCE for third-party app sign-in, allowing users to
authorize your app with their existing OpenRouter account and credit balance.

### PKCE flow

```swift
var components = URLComponents(string: "https://openrouter.ai/auth")!
components.queryItems = [
    .init(name: "callback_url",
          value: "https://app.openclaw.ai/oauth/openrouter/callback"),
]

let session = ASWebAuthenticationSession(
    url: components.url!,
    callbackURLScheme: "https"
) { callbackURL, error in
    // Extract `code` from callbackURL and exchange via file-api relay
}
session.presentationContextProvider = self
session.prefersEphemeralWebBrowserSession = false
session.start()
```

After the user signs in, OpenRouter redirects to `callback_url?code=<code>`. Exchange the
code for an API key via the OpenRouter token endpoint server-side.

This approach means users can authorize with their existing OpenRouter credit balance
without ever seeing an API key. It is the recommended OAuth path when targeting users
who already have an OpenRouter account.

---

## Options C and D

OpenRouter does not have a standalone iOS app, so Option C (provider app deep link) does
not apply.

**Option D** (OpenClaw URL scheme) is available for users who have a key on another device:

```
https://app.openclaw.ai/connect?provider=openrouter&token=<base64-encoded-key>
```

```swift
.onOpenURL { url in
    guard
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
        components.path == "/connect",
        let provider = components.queryItems?.first(where: { $0.name == "provider" })?.value,
        provider == "openrouter",
        let rawToken = components.queryItems?.first(where: { $0.name == "token" })?.value,
        let tokenData = Data(base64Encoded: rawToken),
        let key = String(data: tokenData, encoding: .utf8)
    else { return }

    Task { await session.saveProviderKey(provider: "openrouter", key: key) }
}
```

---

## Credential flow to the container

```
User pastes API key / OAuth key arrives
        ↓
iOS app  →  PUT https://files.spark.ooo/config
            Authorization: Bearer <supabase-jwt>
            { "provider": "openrouter", "apiKey": "sk-or-..." }
        ↓
file-api writes to Azure Files share
        ↓
OpenClaw pod reloads config  →  OPENROUTER_API_KEY available
```

---

## Why OpenRouter as a default provider

For a managed iOS app onboarding flow, OpenRouter is worth considering as the **default
provider** because:

- One key grants access to 100+ models from Anthropic, OpenAI, Google, Meta, and more.
- Users can start with a free tier and add credits as needed.
- No per-provider account creation required.
- OpenRouter keys can be scoped and rate-limited per app.

Users can always add per-provider keys later for cost optimization or feature access
(e.g., Anthropic prompt caching, OpenAI batch API).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | Wrong or expired key | Regenerate at openrouter.ai/keys |
| `402` / `insufficient credits` | Balance depleted | Top up at openrouter.ai/credits |
| Model not found | Model ID format mismatch | Use `openrouter/<provider>/<model>` format |
| OAuth code exchange fails | `callback_url` mismatch | Verify URL in OpenRouter app settings |

## Related docs

- [OpenRouter provider (OpenClaw)](/providers/openrouter)
- [iOS Quick Connect overview](./index.md)
